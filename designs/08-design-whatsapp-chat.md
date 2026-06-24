# Design 08: WhatsApp / a Large-Scale Chat System

> **Why this problem is a staff filter:** chat looks like a CRUD app — a `messages` table, a `POST
> /send`, a `GET /history` — and juniors design exactly that and never explain how a message reaches
> a phone that isn't polling. The whole problem inverts the request/response model the rest of this
> prep lives in: the *server* must push, the connection *stays open*, and statelessness — the thing
> that let you scale everything else by just adding boxes — is gone. The staff signal is that you
> stop treating "the WebSocket" as the design and start treating the **connection gateway, the
> session registry, the pub/sub backplane, and the per-user inbox** as the design; that you make
> **store-and-forward** the source of truth and the socket merely a fast path; and that you get
> **per-conversation ordering and exactly-once *effects*** right on an at-least-once substrate. This
> walkthrough runs the [Topic 1 framework](../prep/01-framework-and-building-blocks.md) end to end
> and leans hard on the [real-time/push](../prep/15-realtime-and-push.md),
> [messaging/streaming](../prep/07-messaging-and-streaming.md), and
> [blob/media](../prep/14-blob-storage-and-media.md) building blocks.

---

## 1. Requirements (5 min) — drive this, don't wait

I'll state the buckets out loud and **scope aggressively** to protect my 45 minutes.

### Functional (the verbs)
- A user **sends a 1:1 message** to a contact; the contact receives it in near-real-time if online,
  and on next connect if offline.
- **Group chat:** a message to a group reaches every member (WhatsApp groups historically ~256, now
  ~1024 members).
- **Delivery receipts** — the sent / delivered / read state machine (the one, two, blue ticks).
- **Presence + typing** — online/last-seen and "typing…" indicators.
- **Multi-device sync** — the same account on phone + laptop + tablet stays consistent.
- **Media messages** — images, video, voice notes, documents.
- **Offline delivery** — the recipient is asleep, backgrounded, or on the subway most of the time.

> **Scope out loud:** "I'll build 1:1 + group messaging, the delivery/read state machine, presence,
> multi-device sync, offline store-and-forward, and media. I'll treat **E2E encryption** at the
> design-constraint level (what it forbids the server from doing) rather than building the Signal
> ratchet, and I'll skip calls/video, status/stories, payments, and spam/abuse unless you want one
> of those as the deep dive instead. Shout if you'd rather I redirect."

### Non-functional (where staff candidates separate)
- **Durability is the top promise.** Once we ACK a message as *sent*, we must **never lose it**.
  Everything downstream (delivery, receipts) can be retried; the accept cannot be taken back.
- **Latency:** perceived send→deliver for two online users **< ~500 ms p99**. Receipts and presence
  can be looser.
- **Ordering:** messages must be consistently ordered **within a conversation** — never globally.
  This is a *much* cheaper guarantee than total order, and stating the distinction is the move.
- **Consistency — split it deliberately:**
  - Message content + the sent-tick → **durable, strongly-ordered within a conversation**.
  - Presence / typing / read-receipt-fanout → **eventual, best-effort, lossy-OK** (a dropped "typing"
    breaks nothing).
- **Availability:** the send path must be ~99.99%. A user who can't send a message thinks the app is
  broken. Degrade gracefully — accept-and-queue even when downstream delivery is impaired.
- **Connection-bound, not QPS-bound.** Unlike every other system in this prep, the dominant cost is
  **hundreds of millions of idle-but-open sockets**, not request throughput. See [Topic 15 — "the cost
  of a persistent connection"](../prep/15-realtime-and-push.md).
- **Mobile-first reality:** unreliable networks, NAT timeouts, half-open connections, and
  backgrounded/killed apps are the *norm*, not the edge case. The design must assume the socket is
  dead more often than alive.

> **The single derived insight to state now:** "The socket is a best-effort *fast path*; the
> **durable per-user inbox is the source of truth**. I persist before I ACK, deliver out of the
> inbox, and let the connection be as flaky as mobile reality demands — losing the socket never
> loses the message. That one sentence is the whole reliable-chat design."

---

## 2. Estimation (3 min) — justify the connection-fleet and storage decisions

Numbers exist to *force* architecture, per [Topic 2](../prep/02-estimation-and-napkin-math.md).

**Assume:** ~2B registered users, **500M DAU**, each sends ~40 messages/day.

### Message throughput (the write firehose)

| Quantity | Calc | Result |
|---|---|---|
| Messages/day | 500M DAU × 40 | **20B msgs/day** |
| Avg message QPS | 20B ÷ 86,400 | **~230k msgs/sec** |
| Peak QPS (2–3×) | 230k × 2.5 | **~500k–700k msgs/sec** |
| Receipt/presence amplification | each msg → sent+delivered+read + presence | **2–4× the write QPS again** |

> The receipt + presence traffic is *larger* than the message traffic itself. That's the hidden cost
> juniors miss — and the reason receipts and presence get aggressive batching/coarsening below.

### Connections (the real bottleneck)

| Quantity | Calc | Result |
|---|---|---|
| Concurrent sockets | large fraction of 500M DAU online | **~hundreds of millions** |
| Sockets per gateway box | conservative, RAM/throughput-bound (Topic 15) | **~500k–1M** |
| **Gateway fleet size** | ~300M ÷ 500k | **~600–1,000 gateway boxes** |
| RAM just to hold idle sockets | 300M × ~tens of KB | **~tens of TB of RAM, fleet-wide** |

### Storage (write-heavy, time-ordered)

| Quantity | Calc | Result |
|---|---|---|
| Bytes per message row | text + metadata (ids, ts, conv, sender, status) | ~300 B |
| Message text/day | 20B × 300 B | **~6 TB/day** |
| Per year | 6 TB × 365 | **~2.2 PB/yr** of text alone |
| With retention + media metadata | multiply out | **petabytes** — media itself offloaded to blob store |

> **The money sentences:** "**~500k–700k msg/sec peak**, *amplified 2–4×* by receipts and presence,
> against **hundreds of millions of concurrent sockets** — so the architecture is governed by
> *connection count*, not QPS, and I split the dumb connection edge from the smart stateless core.
> And **~6 TB/day, petabytes/yr, append-mostly, range-read by conversation+time** — that's a textbook
> *wide-column* workload, not a relational one. The numbers picked Cassandra and the gateway fleet for
> me; I didn't memorize them."

---

## 3. API design (3 min)

Two surfaces: a **persistent WebSocket** for the live bidirectional path, and a thin **REST** surface
for history/media bootstrap (cold loads don't need a socket).

```
# Persistent connection
WS   /connect            (auth token in handshake)  -> upgraded socket
     ── client→server frames ──
        SEND      { client_msg_id, conversation_id, ciphertext, media_ref? }
        ACK       { type: delivered|read, up_to_message_id, conversation_id }
        TYPING    { conversation_id }                 # fire-and-forget
        SYNC      { device_id, cursors: {conv_id: last_seen_msg_id} }
     ── server→client frames ──
        MESSAGE   { message_id, conversation_id, sender, ciphertext, server_ts, seq }
        RECEIPT   { message_id, state, by_user }
        PRESENCE  { user_id, state, last_seen }

# REST (bootstrap / fallback / media)
GET  /conversations?cursor=                          -> [conv summaries], nextCursor
GET  /conversations/{id}/messages?before_seq=&limit= -> [messages], nextCursor   # cursor, not offset
POST /media/upload-url   { content_type, size }      -> { presigned_url, media_ref }   # see §6
```

- **`client_msg_id`** is a client-generated UUID. It rides the whole pipeline and is the **idempotency
  key** that makes resend-on-flaky-network safe (dedup, §"ordering & dedup"). This is load-bearing.
- **Cursor-based pagination** for history (offset breaks under constant inserts), per Topic 1.
- **Auth, rate-limit, and TLS terminate at the gateway** during the handshake — stated once here so I
  don't re-explain it. See [Topic 9](../prep/09-api-gateway-loadbalancing-ratelimiting.md).

---

## 4. Data model (5 min) — access pattern picks the store

The dominant read access pattern is the entire ballgame: **"give me messages in conversation C,
newest-first, paginated."** Write-heavy, append-mostly, range-scan by time within a partition. That
is a wide-column fit (see [Topic 3](../prep/03-databases-deep-dive.md) and §5), not relational.

**`messages` (wide-column — Cassandra/ScyllaDB)**
- **Partition key = `(conversation_id, time_bucket)`** — bucket by e.g. month so a hot group's
  partition can't grow unbounded.
- **Clustering key = `seq` (per-conversation sequence) DESC** → "latest N" is a cheap head-of-partition
  read; ordering is free.
- Columns: `message_id` (Snowflake), `sender_id`, `ciphertext`, `media_ref`, `server_ts`, `seq`.

**`inbox` / per-user delivery queue (the source of truth for delivery)**
- **Partition key = `user_id`** (or `(user_id, device_id)` for multi-device).
- Rows = undelivered/unsynced message references, ordered. This is the **store-and-forward** queue;
  see §"delivery."

**`conversations`**
- `conversation_id`, type (1:1 | group), member list (or pointer to a `members` table for big groups),
  and the **per-conversation `seq` counter** (the ordering authority).

**`device_cursors`**
- `(user_id, device_id) → last_acked_seq per conversation`. The reconnect/resume + multi-device sync
  mechanism (§"multi-device sync").

**`presence` (Redis, ephemeral)** — `user_id → {state, last_seen}` with a **TTL**; never durable.

> **SQL vs NoSQL, decided from access patterns not reflex:** messages are pure
> partition-key + time-range scans with petabyte write volume and zero need for joins or ad-hoc
> queries → **wide-column NoSQL**. The only place I'd reach for a relational store is the low-volume
> account/contacts/group-membership metadata, where I do want consistency and the occasional join.

---

## 5. High-level design (10 min) — happy path end to end

The governing structural decision, straight from [Topic 15 Part B](../prep/15-realtime-and-push.md):
**dumb edge, smart core.** Split so the connection layer scales on socket count independently of the
business logic.

```
   clients ──WS──►┌──────────────────────────────┐
   (100s of M)    │   Connection Gateway fleet     │  terminate TLS, hold sockets,
                  │   (~600–1000 boxes)            │  heartbeat, auth handshake,
                  └──────────────┬─────────────────┘  NO business logic
                       publish ▲ │ push to socket
       userId→gateway          │ ▼
   ┌──────────────┐     ┌─────────────────────────────┐
   │  Session     │◄────│   Pub/Sub Backplane           │  Redis/NATS = last-hop tickle (lossy)
   │  Registry    │     │   + Kafka (durable fan-out)   │  Kafka = durable inbox feed
   │  (Redis)     │     └──────────────┬───────────────┘
   └──────────────┘                    ▲
                            stateless  │ emit "deliver M to U"
                        ┌──────────────┴───────────────┐
                        │   Chat / Messaging service     │  assign seq, persist, fan-out
                        └───┬──────────┬──────────┬──────┘
                            ▼          ▼          ▼
                     [ messages ]  [ inbox  ]  [ presence ]
                     (Cassandra)   (queue)     (Redis)        media → [ S3 + CDN ] (§6)
```

**Walk one 1:1 message through it (both users online):**

1. **A's client** sends `SEND{client_msg_id, conv, ciphertext}` over its socket to **Gateway-7**.
2. Gateway-7 (no logic) forwards to a stateless **Chat service**.
3. Chat service: **dedup** on `client_msg_id` → **assign `seq`** for the conversation + a Snowflake
   `message_id` → **persist to the `messages` store and write a row to B's `inbox`** →
   **then ACK "sent"** back to A. *Durability before acknowledgement* — the sent tick means "we own
   this." ([Topic 15](../prep/15-realtime-and-push.md): persist before you ACK.)
4. To reach B: look up **B's gateway in the session registry** (or publish to `user:{B}` on the
   backplane). Whichever gateway holds B receives "deliver M" and **pushes down B's socket**.
5. B's client emits **`ACK delivered`** → flows back through the pipeline to A as a *delivered* receipt
   (second tick). When B opens the chat, **`ACK read`** → *read* receipt (blue ticks).
6. **B offline?** The message already sits durably in B's `inbox`. We fire an **APNs/FCM push** so the
   phone surfaces a notification with the app closed; on next connect B **syncs from the inbox** (§"delivery").

> Keep it this simple first, then evolve under questioning. The two arrows that carry the staff signal
> are **"persist → then ACK"** (durability) and **"inbox row exists regardless of socket state"**
> (store-and-forward). Everything in the deep dives hangs off those two.

---

## 6. Deep dives (15 min) — where the round is won

I'd propose the order: *"The interesting parts are (a) routing a message to the right box across 800
gateways, and (b) making delivery reliable and correctly-ordered over flaky mobile sockets. Can I go
deep on those, then touch groups, storage, sync, E2EE, presence, and media?"*

### 6a. Persistent connection management at scale

**The C10M reality (Topic 15).** Each socket costs kernel buffers + app state (user id,
subscriptions, send buffer) ≈ tens of KB, plus a file descriptor and heartbeat CPU. A modern box holds
~500k–1M *idle* sockets; the practical ceiling is **RAM and message rate, not the FD count**. We tune
FD ulimits to millions, widen ephemeral ports, size TCP buffers. Crucially, the gateway carries **no
business logic**, so it almost never deploys — *every deploy drops every connection it holds*, which is
why the smart core lives elsewhere and ships independently.

**Sticky routing.** A WebSocket lives on one gateway for its whole life, so the L4/L7 LB pins a client
to a box **for the connection lifetime only** (consistent hashing on connection, or an L7 cookie). On
*reconnect* the client may land anywhere and re-registers — we deliberately avoid pinning a *user*
permanently, which would create hot boxes.

**The session registry — the crux.** *"A on gateway-7 sends to B. Where is B, and how does the message
get there?"* I need `userId → {gatewayId, connectionId}`. Two designs, and I'd name both then pick:

- **Option 1 — Registry lookup + targeted delivery.** On connect, the gateway writes
  `user:{B} → gateway-N` to Redis (short TTL). To deliver: look up B's gateway, route there (RPC or
  backplane). *Pro:* targeted, no wasted fan-out. *Con:* a **stale entry** (B reconnected elsewhere)
  drops the push — mitigate with short TTLs and **re-lookup-on-failure**, and remember the inbox row
  still exists so nothing is *lost*, just delayed.
- **Option 2 — Channel-per-user pub/sub (no explicit registry).** Each gateway subscribes `user:{B}`
  for every B it holds; to deliver, publish to `user:{B}` and whichever gateway has B picks it up.
  *Pro:* dead simple, **self-healing on reconnect**. *Con:* enormous subscription counts per gateway —
  great on Redis Pub/Sub / NATS, poor where subscriptions are expensive.

> **My pick + tradeoff:** "Channel-per-user on a Redis/NATS backplane for the **last-hop tickle**,
> because self-healing beats a consistency-sensitive registry at this scale, and the durable inbox is
> my safety net anyway. I keep a Redis session-registry **only as a hint** to skip publishing to users
> who are definitely offline (fall straight to APNs/FCM)."

**Backplane choice — say the split** (Topic 15): **Redis Pub/Sub / NATS** is fire-and-forget,
ultra-low-latency, *no persistence* — perfect for the latency-critical "wake the socket" hop, where the
inbox already guarantees durability. **Kafka** is durable, ordered-per-partition, replayable — I use it
for the **fan-out work and the durable event log**, not the tickle. The production pattern is exactly
this: **Kafka for durability + fan-out, Redis for the last-hop push.**

### 6b. Message delivery: store-and-forward, and the sent/delivered/read state machine

**The mobile truth: most recipients are offline most of the time.** So **every message is durably
written to the recipient's per-user inbox first**, and delivery is a *separate* concern layered on top:

- **B online:** push via backplane → socket immediately, *and* the inbox row exists for durability.
- **B offline:** the message waits in B's inbox. On reconnect the gateway **syncs everything after B's
  last-acked cursor**; we also fire **APNs/FCM** so the phone notifies even with the app killed (our
  socket dies the instant the OS suspends the app — Topic 15 Part A).

> **The heart of reliable chat:** the socket is best-effort; the **inbox is the source of truth.**

**The state machine** — three states, each a small message flowing *back* to the sender through the
same pipeline:

```
   (A sends) ──► [SENT]      server durably accepted   (✓  one tick)
                   │ pushed to B's device
                   ▼
                [DELIVERED]  reached B's device         (✓✓ two ticks)
                   │ B opens the chat
                   ▼
                [READ]       B actually viewed it        (✓✓ blue)
```

Each transition is an idempotent, monotonic forward move keyed by `message_id` — a duplicate
`delivered` ACK is a no-op, and we never regress READ→DELIVERED. Receipts are themselves messages, so
they ride the identical store-and-forward path (a receipt for an offline sender waits in *their* inbox).

### 6c. Ordering, message IDs, and dedup

- **Order holds within a conversation only** — never globally. Far cheaper, and it's all users perceive.
- **`seq`: a per-conversation sequence number assigned server-side** by the chat service is the
  ordering authority. Clients **sort by `seq`, not arrival order** (the network reorders frames).
- **`message_id`: Snowflake** (timestamp + machine + counter) → globally unique, roughly time-sortable,
  no central allocator on the hot path. **Never trust client clocks** for ordering.
- **Dedup via `client_msg_id`.** Flaky mobile networks mean clients *will* resend (they didn't see the
  ACK). The chat service keeps a short-lived seen-set (Redis, or a Bloom filter — Topic 1 — to cheaply
  answer "definitely not seen") keyed by `client_msg_id` so a resend maps to the *same* `seq`/`message_id`
  instead of creating a duplicate. This makes the whole pipeline **at-least-once delivery with
  idempotent effect** — the standard messaging contract ([Topic 7](../prep/07-messaging-and-streaming.md)).

> **Why a per-conversation counter and not a global one:** a single global sequencer would be a
> throughput ceiling and a SPOF at 700k msg/sec. Per-conversation, the counter lives with the
> conversation's partition and contention is bounded by that one chat's send rate. Concurrent sends to
> the same conversation serialize at the partition (a lightweight per-conversation lock / single-writer
> per partition) — cheap because a conversation's send rate is tiny.

### 6d. 1:1 vs group fan-out — and the large-group cost

- **1:1:** one inbox write + one push. Trivial.
- **Group (fan-out on write):** assign one `seq` for the group message, then **write into every
  member's inbox + push to every online member.** For WhatsApp-sized groups (hundreds → ~1k members)
  this is fine: a few hundred inbox writes done by **batched fan-out workers** off the backplane.
- **Large broadcast channels (Slack channels, thousands+):** this is the **celebrity fan-out problem**
  in disguise — one send → thousands of inbox writes + thousands of pushes. Mitigations to name:
  **batched fan-out workers**, and for very large channels flip to **fan-out-on-read**: store the
  message once in the channel timeline and have members **pull** on read, instead of materializing
  thousands of inbox rows. This is the same push-vs-pull hybrid I'd use for celebrity followers in
  [Design 01 (Twitter feed)](01-design-twitter-newsfeed.md) — cross-reference it out loud.

> **The tradeoff stated:** fan-out-on-write = fast reads, expensive writes (explodes for huge groups);
> fan-out-on-read = cheap writes, expensive/merge-heavy reads. **Hybrid by group size** is the staff
> answer — small groups push, mega-channels pull.

### 6e. Message storage — wide-column, time-ordered

Justified by §2's numbers (petabytes, append-mostly, range-by-time): **wide-column** (Cassandra /
ScyllaDB / HBase — and famously WhatsApp on Erlang/Mnesia, Discord on Cassandra→ScyllaDB).

- **Partition key = `(conversation_id, time_bucket)`** — bucketing caps a hot group's partition size
  (avoid the unbounded-partition antipattern, [Topic 4](../prep/04-sharding-and-partitioning.md)).
- **Clustering key = `seq` DESC** → "latest N messages" is a cheap head-of-partition read; ordering is
  inherent.
- **LSM-tree under the hood** = fast sequential writes — exactly our write-heavy profile
  ([Topic 1 building blocks](../prep/01-framework-and-building-blocks.md)).
- **Sharding:** by `conversation_id` spreads load evenly and keeps a conversation's messages
  co-located for the range read. The inbox shards by `user_id`. Two different shard keys for two
  different access patterns — that's deliberate, not a contradiction.

Media **never** lives in the message row (§6h).

### 6f. Multi-device sync, reconnect + resume

Model **devices, not just users.** Deliver to **all of a user's active devices**, and track a
**per-device last-acked cursor** (`device_cursors`) so each device syncs independently from where *it*
left off:

- On reconnect, the device sends `SYNC{cursors}` → gateway/chat service streams **everything after each
  cursor**. This single mechanism *is* both reconnect-resume and multi-device sync.
- A message is removed from a device's pending-inbox view only once **that device** acks it — so adding
  a new laptop doesn't lose messages the phone already saw, and vice versa.
- The cursor also drives "delivered to *which* device" — WhatsApp shows delivered once *a* device has it.

> This is why I modeled `(user_id, device_id)` early: presence, receipts, and sync all become
> per-device. Designing it as user-only and bolting on devices later is the classic rework trap.

### 6g. End-to-end encryption — the design constraints (high level)

With true E2EE (Signal protocol, WhatsApp's basis) the **server only ever sees ciphertext** — it
routes and stores opaque blobs and *cannot* read content. I'd state the **constraints it imposes** more
than the crypto:

- **No server-side search or content indexing** — the server can't read messages, so search is
  client-side only. (Big product constraint; contrast Slack, which is *not* E2EE precisely so it can
  offer server-side search/compliance.)
- **Key exchange + multi-device key management become the hard part**, not the transport. Each device
  has its own keys; a sender encrypts **per-recipient-device**, so a group message to N members with M
  devices each is N×M encryptions (fan-out cost compounds with E2EE). New device = key re-negotiation.
- **Metadata is still visible** to the server even when content isn't: who messaged whom, when, sizes,
  receipts. E2EE protects content, not the social graph.
- **The inbox stores ciphertext.** Store-and-forward, ordering, dedup, fan-out all still work — they
  operate on opaque blobs + `message_id`/`seq`/`client_msg_id`, none of which need plaintext. This is
  why the architecture is unchanged by E2EE; only the *payload* is opaque.

### 6h. Media messages (tie to blob storage)

Media goes **direct-to-blob, never through the message pipeline** — per
[Topic 14](../prep/14-blob-storage-and-media.md):

1. Client requests `POST /media/upload-url` → service returns a **presigned URL** + a `media_ref`.
2. Client **uploads bytes directly to object storage (S3)** via the presigned URL (multipart/resumable
   for large video — Topic 14 Part C), bypassing our servers entirely.
3. Client sends a normal chat message whose body is just `{media_ref, thumbnail, dims, size}` —
   **the message row holds a pointer + metadata, never the blob.**
4. Recipients fetch the media from object storage **through a CDN** with **signed URLs** for access
   control (Topic 14 Part D). Thumbnails/transcodes are produced by the async media pipeline.

> **The ordering rule (Topic 14 Part H, the dual-write trap):** upload the blob to S3 **first**, then
> write the message referencing it — never the reverse, or a recipient can receive a pointer to a blob
> that doesn't exist yet. With E2EE, media is encrypted client-side before upload, so S3 stores
> ciphertext too.

### 6i. Presence + typing indicators (tie to realtime doc)

Presence looks like a green dot and is secretly a fan-out monster (Topic 15 Part C).

- **Heartbeat + TTL.** Client pings every ~10–30s; gateway writes `SET presence:{user} online EX 45`.
  No heartbeat → key expires → implicitly offline. **TTL is how we detect *ungraceful* disconnects** —
  the half-open-connection problem (TCP won't reliably tell us the phone vanished). Self-cleaning; no
  reliable "goodbye" needed.
- **The cost is the read fan-out, not the write.** One person coming online could mean pushing to
  thousands watching them. Mitigations: **pull-on-demand** (clients query presence only for contacts
  currently *on screen* — the visible chat list), **subscribe to the visible set** (bounds fan-out by
  screen size, not contact count), **coarsen + batch** (Online/Away/Offline, debounce flapping), and
  **last-seen as a cheap DB read** instead of a live subscription.
- **Typing is presence's twin:** ephemeral, fire-and-forget, TTL ~5–10s, **never persisted**. Lose one
  and nothing breaks — so it goes over the lossy Redis backplane, not the durable inbox.
- **Shard presence by `user_id`** across a Redis cluster (consistent hashing) — small per user, huge
  and hot in aggregate.

---

## 7. Wrap-up (3 min) — bottlenecks, failure modes, SPOFs

- **A gateway dies (the headline failure mode).** Every socket on that box drops. Because gateways hold
  **no state we can't rebuild**, recovery is automatic: clients detect the dead socket (heartbeat) and
  **reconnect**, landing on any healthy box via the LB, and **resume from their device cursor** (§6f).
  Presence self-heals via TTL expiry. No message is lost because the **inbox is durable** independent of
  the socket. *This is the whole reason for dumb-edge/smart-core.*
- **Remaining bottlenecks:** (1) **connection memory** across the fleet — scale horizontally on box
  count, the cost is real (tens of TB RAM); (2) **receipt + presence amplification** dwarfing message
  traffic — handled by coarsening/batching; (3) **large-group fan-out** — handled by batched workers +
  fan-out-on-read for mega-channels.
- **Failure modes named:** *backplane (Redis) loses a push* → no loss, inbox + cursor resync covers it
  (the tickle is best-effort by design); *Kafka fan-out lag* → delayed group delivery, not lost,
  consumers catch up; *Cassandra node down* → quorum reads/writes ride it out
  ([Topic 5](../prep/05-replication-and-consistency.md)); *stale session-registry entry* →
  re-lookup-on-failure + inbox safety net.
- **SPOFs — there are none global by design.** Gateways are a fleet; the backplane and Cassandra are
  clustered; the per-conversation sequencer is *per-conversation*, not central; presence is sharded.
  The closest thing to a hot spot is a single mega-group's partition, mitigated by time-bucketing +
  fan-out-on-read.
- **With more time:** geo-distribution (route users to the nearest gateway region, replicate inbox/messages
  cross-region with conflict-free per-conversation seq); abuse/spam at the gateway; backpressure +
  rate-limiting on send ([Topic 13](../prep/13-resilience-and-failure-handling.md)); cold-storage
  tiering of old messages to cheaper blob storage.

---

## What made this staff-level

- **Made store-and-forward the source of truth and the socket a best-effort fast path** — "persist
  before ACK, deliver from the durable inbox" — instead of treating the WebSocket as the design. That
  one inversion is the entire reliability story and the thing juniors miss.
- **Designed the connection layer as the real system** — dumb-edge/smart-core, the session registry
  crux, channel-per-user vs registry-lookup *with a pick*, and the Redis-tickle / Kafka-durable
  backplane split — rather than drawing one box labeled "WebSocket server."
- **Got ordering right at the cheap altitude** — per-conversation `seq` as the authority, Snowflake for
  IDs, client-side sort, and `client_msg_id` dedup turning an at-least-once substrate into
  idempotent-effect delivery — and explained *why* not a global sequencer.
- **Treated group fan-out as the celebrity problem** and prescribed a size-based push/pull hybrid,
  cross-referencing the Twitter feed design rather than re-deriving it.
- **Let the estimation pick the stores:** connection count (not QPS) sized the gateway fleet;
  petabytes-append-mostly-range-by-time picked wide-column; the receipt/presence amplification justified
  coarsening.
- **Treated E2EE as a constraint engine** — what it forbids (server search), what it complicates
  (per-device key fan-out), and what it leaves untouched (the whole inbox/ordering/dedup pipeline runs
  on opaque blobs) — instead of hand-waving "and it's encrypted."
- **Named the gateway-death failure mode and the dual-write media trap before being asked**, and showed
  there is no global SPOF by construction.

---

### Self-check before the mock (answer these from memory)
- [ ] State the governing insight: why is the inbox the source of truth and the socket only a fast path?
- [ ] Why is this system bottlenecked on *connection count*, not QPS? Size the gateway fleet from DAU.
- [ ] Explain dumb-edge/smart-core and *why* the gateway must carry no business logic (deploys).
- [ ] The session-registry question: A on gateway-7 → B. Two designs, the stale-entry failure, your pick.
- [ ] Why Redis for the last-hop push but Kafka for fan-out/durability?
- [ ] Walk the sent→delivered→read state machine. Where does "persist before ACK" sit and why?
- [ ] Why per-conversation `seq` and not a global sequencer? How do `message_id` and `client_msg_id`
      differ in job, and how does dedup make delivery idempotent?
- [ ] 1:1 vs group fan-out, and exactly when you flip a large group from fan-out-on-write to -on-read.
- [ ] Why wide-column for messages? Give the partition + clustering keys and why bucket by time.
- [ ] How do multi-device sync and reconnect-resume share one mechanism (the per-device cursor)?
- [ ] Three things E2EE forbids/complicates, and one thing it leaves completely unchanged — and why.
- [ ] The media upload flow and the dual-write ordering rule (blob first, then message).
- [ ] Why is presence a fan-out monster, and the four mitigations? Why can typing be lossy?
- [ ] A gateway dies — trace recovery end to end. Why is no message lost and no SPOF global?
