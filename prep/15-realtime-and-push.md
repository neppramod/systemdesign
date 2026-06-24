# Topic 15: Real-Time Systems — WebSockets, Presence, Chat & Notifications

> **Why this topic is its own beast:** Every other topic in this prep is request/response — client
> asks, server answers, connection closes. Real-time inverts that. The *server* needs to push, the
> connection *stays open*, and now you own the hard problems that statelessness used to hide:
> which box is this user on, what happens when they disconnect mid-message, how do you fan a single
> event out to 50 million sockets. The staff-level signal here is that you stop treating "the
> WebSocket" as the design and start treating the **connection registry, the pub/sub backplane, and
> the per-user inbox** as the design. The socket is the easy part.

---

## Part A — Delivery mechanisms (pick the cheapest thing that meets the requirement)

The first question on any real-time prompt is: *how does the server get data to the client?* You
have five tools. Reach for the simplest one that satisfies the latency requirement — persistent
connections are not free, and a senior candidate is the one who says "we don't actually need
WebSockets here."

| Mechanism | How it works | Latency | Server cost | Direction | Use when |
|---|---|---|---|---|---|
| **Short polling** | Client hits `GET /updates` every N seconds | Up to N sec | Wasteful — most requests return nothing | Client→Server | Updates are infrequent and seconds-stale is fine (email-style refresh) |
| **Long polling** | Client requests, server *holds* it open until data or timeout, then client re-requests | ~Near-real-time | One held request per client + reconnect churn | Client→Server (server responds) | You need pushiness but can't run WebSockets (legacy proxies, simplicity) |
| **SSE** (Server-Sent Events) | One long-lived HTTP response, server streams `text/event-stream` | Real-time | One open connection per client, but HTTP/1.1 + cheap | **Server→Client only** | Server-push only: live scores, notifications, dashboards, LLM token streaming |
| **WebSocket** | One TCP connection, HTTP `Upgrade`, then full-duplex frames | Real-time | One open connection per client, stateful server | **Bidirectional** | Chat, multiplayer, collaborative editing — anything with client→server *and* server→client |
| **Mobile push** (APNs/FCM) | You hand a payload to Apple/Google; *they* hold the connection to the device | Seconds | ~Zero for you — OS owns the socket | Server→Device | App is backgrounded/closed; you have no socket of your own |

**The decision in the room:**

- "Is it server-push only, or do I need the client to send too?" SSE if push-only, WebSocket if duplex.
- "Is the app in the foreground?" If yes, your own socket (WS/SSE). If it might be backgrounded or killed, you *must* fall back to **APNs/FCM** — your socket is dead the moment the OS suspends the app.
- "Do I even need real-time?" If "updates within 30s" is acceptable, polling is the boring correct answer and you save yourself the entire connection-management problem below.

> **Say this:** "WebSockets are the answer to *bidirectional, low-latency*. They are the wrong answer
> to *'the server occasionally pushes a thing'* — that's SSE — and the wrong answer to *'reach a
> phone that's asleep'* — that's push notifications. Real production chat apps use **all three**:
> WebSocket when the app is open, APNs/FCM when it's backgrounded."

### The cost of a persistent connection

Stateless HTTP scales by being forgettable — any box can serve any request, so you just add boxes.
Persistent connections break that:

- **Memory per connection.** Each socket has kernel buffers + your app-level state (user id, subscriptions, send buffer). Budget ~tens of KB. A million connections = tens of GB of RAM *just to hold idle sockets*.
- **File descriptors.** Each socket is an FD. Default `ulimit` is ~1024 — you'll raise it to millions and tune `nf_conntrack`, ephemeral port ranges, etc.
- **Statefulness.** The connection lives on *one specific box*. Now load balancing, deploys (every restart drops every connection), and routing a message to "wherever this user is connected" all become your problem.
- **Idle but not free.** Even a connection sending nothing costs RAM + an FD + heartbeat CPU.

That last point is the whole game: with HTTP you scale on QPS; with WebSockets you scale on
**concurrent connections**, and most of them are idle.

---

## Part B — Managing millions of persistent connections

### The C10M problem

C10K ("can one box handle 10,000 concurrent connections?") was solved in the 2000s by abandoning
thread-per-connection for **event loops** (epoll/kqueue) — `nginx`, Node, Netty, Go's runtime. C10M
is the modern bar: ~1M+ connections per box. It's reachable but you're now fighting the kernel and
the GC, not your app logic. Tuning levers: raise FD limits, bigger ephemeral port range, tune TCP
buffers, pin to NUMA nodes, sometimes kernel-bypass (DPDK) for extreme cases.

**How many connections per box?** A defensible interview number: **~100k–1M idle connections** per
modern box, depending on per-connection memory and message rate. Don't quote 10M as routine. The
practical limit is usually **RAM and message throughput, not the socket count itself** — 1M sockets
each sending one message/sec is 1M msg/sec of work, which is your real ceiling.

### The architecture: dumb edge, smart core

Split the system in two so you can scale the connection layer independently of business logic:

```
                         ┌──────────────────────────────┐
  clients ──WS──► [ Connection Gateway fleet ]           │
  (millions)      (terminate TLS, hold sockets,          │
                   heartbeat, auth handshake)            │
                         │            ▲                  │
              publish    │            │  push to socket  │
                         ▼            │                  │
                  ┌─────────────────────────────┐        │
                  │   Pub/Sub Backplane          │        │
                  │   (Redis / Kafka / NATS)     │        │
                  └─────────────────────────────┘        │
                         ▲                                │
                         │ business logic emits events    │
                  [ Stateless app/chat services ]─────────┘
                         │
                  [ Storage, Inbox, etc. ]
```

- **Connection gateway / edge servers** do one job: hold sockets, do the auth handshake, heartbeat, and shuttle frames. They carry **no business logic** so they rarely deploy (deploys kill connections) and scale purely on connection count.
- **Stateless services** behind them do the real work and never touch a socket directly — they publish "send msg M to user U" to the backplane.

### Sticky routing

A WebSocket lives on one gateway box for its whole life, so the LB must keep a client pinned:

- **L4 with consistent hashing** on client IP/connection, or
- **L7 with a cookie/session token** the LB hashes on.

You only need stickiness for the *lifetime of the connection*. On reconnect the client can land on
any box (and re-register). Avoid stickiness that pins a *user* permanently — that creates hot boxes.

### The session / connection registry (the crux)

> **This is the question that separates real answers from hand-waving:** "User A on gateway-7 sends
> a message to user B. Where is B connected, and how does the message get there?"

You need a mapping `userId → {gatewayId, connectionId}`. Two ways to use it:

**Option 1 — Registry lookup + direct delivery.** On connect, gateway writes `userId → gatewayId`
to a fast store (Redis). To send to B: look up B's gateway, then deliver. Delivery box-to-box can be
a direct RPC or via the backplane. Pro: targeted, no wasted fan-out. Con: registry must be
fast/consistent; a stale entry (B reconnected elsewhere) drops messages — mitigate with short TTLs
and re-lookup on failure.

**Option 2 — Pub/sub channel per user (no explicit registry).** Each gateway subscribes the channel
`user:{B}` for every B it holds. To send to B, publish to `user:{B}`; whichever gateway holds B
gets it, everyone else ignores. Pro: dead simple, self-healing on reconnect. Con: every gateway has
huge numbers of subscriptions; works great on Redis Pub/Sub or NATS, less so if subscriptions are
expensive.

**Redis vs Kafka as backplane — say the tradeoff:**

- **Redis Pub/Sub / NATS:** fire-and-forget, ultra-low latency, **no persistence** — if no subscriber is listening, the message is gone. Perfect for the *live push* path where store-and-forward (below) handles durability separately.
- **Kafka:** durable, ordered per partition, replayable, higher latency. Use it as the **durable event log / inbox feed** and for fan-out work, not for the latency-critical "tickle the socket" path.

A common production split: **Kafka for durability + fan-out, Redis for the last-hop socket push.**

---

## Part C — Presence (online / offline / typing)

Presence looks trivial ("show a green dot") and is secretly a fan-out monster.

### Heartbeats + TTL

- Client sends a heartbeat / ping every ~10–30s (WebSocket has ping/pong frames).
- Gateway writes presence to Redis with a **TTL** slightly longer than the interval: `SET presence:{user} online EX 45`.
- No heartbeat → key expires → user is implicitly offline. **TTL is how you detect ungraceful disconnects** (the half-open connection problem — TCP won't always tell you the client vanished).

This makes presence *self-cleaning*: you never need a reliable "I'm leaving" message.

### The fan-out cost (the real problem)

Going online is a single write. The expense is **notifying everyone who's watching you**. If you
have 5,000 friends/followers all viewing a presence indicator, one status flip = 5,000 pushes. Now
multiply by everyone toggling online/offline as phones sleep and wake. Presence write volume is
tiny; presence *read fan-out* is enormous.

Mitigations to name:

- **Pull on demand, not push.** Don't push every flip. Let clients query presence for the users currently on screen (the visible contacts in a chat list). Fan-out drops from "all friends" to "people looking right now."
- **Subscribe to the visible set.** Client subscribes to presence channels only for contacts in view; unsubscribes on scroll. Bounds fan-out by screen size, not friend count.
- **Coarsen + batch.** "Online / Away / Offline" instead of exact timestamps; debounce flapping (don't broadcast a 2-second blip); batch presence diffs every few seconds rather than per-event.
- **Last-seen instead of live.** "last seen 5m ago" is a cheap DB read, not a live subscription — much of "presence" doesn't need to be real-time at all.

### Sharding presence

Presence state is small per user but huge in aggregate and very hot. **Shard the presence store by
userId** (consistent hashing across a Redis cluster). Typing indicators are presence's twin —
ephemeral, fire-and-forget, TTL-based (~5–10s), never persisted. Lose one and nothing breaks.

---

## Part D — Designing a chat system (WhatsApp / Slack), in depth

This is the canonical real-time prompt. Drive it through the framework.

### Requirements (state them)
- **Functional:** 1:1 messaging, group chat, delivery receipts (sent/delivered/read), online/typing presence, multi-device sync, offline delivery, media.
- **Non-functional:** low latency (<500ms perceived send→deliver), **ordering within a conversation**, durability (never lose an accepted message), massive concurrent connections, mostly mobile (so unreliable networks + backgrounded apps are the norm).

### Estimation (WhatsApp-scale, defensible numbers)
- ~2B users, say **500M DAU**, each sends ~40 messages/day → **20B messages/day** ≈ **230k msg/sec average**, peak ~2–3× → **~500k–700k msg/sec**.
- Concurrent connections: a large fraction of DAU online → **hundreds of millions of sockets** → at ~500k/box, ~**hundreds to low thousands of gateway boxes**.
- Storage: 20B msgs/day × ~300 bytes (text + metadata) ≈ **6 TB/day** of message text alone, more with media offloaded to blob storage. Multiply by retention → **petabytes**. This screams *write-heavy, time-ordered* → wide-column.

### Message flow (1:1, both online)

```
A's socket ─► Gateway-7 ─► Chat service
                              ├─ assign message_id (ordered, see below)
                              ├─ persist to message store (durability FIRST)
                              ├─ write to B's inbox / publish to user:{B}
                              └─ ACK back to A ("sent")
Gateway holding B ◄─ backplane ◄─ "msg for B"
B's socket ◄─ deliver
B's client ─► ACK "delivered" ─► back to A as a receipt
B reads it  ─► ACK "read" ─────► back to A as a receipt
```

**Persist before you ACK.** The "sent" tick means *we durably own this message*, so write to storage
before acknowledging. After that, delivery can be async/retried — but you've promised not to lose it.

### Delivery + read receipts

Three states, each a separate ACK flowing back to the sender: **sent** (server accepted),
**delivered** (reached B's device), **read** (B opened it). Receipts are themselves small messages —
they go through the same pipeline. At group scale read receipts are a fan-out problem (every reader
× every member), so often summarized ("read by 12") rather than per-pair.

### Ordering

- Order only needs to hold **within a conversation**, not globally — that's a much cheaper guarantee.
- Use a **per-conversation sequence number** assigned server-side, or time-based IDs from a generator (**Snowflake**: timestamp + machine + sequence) so IDs are roughly time-sortable and unique. Don't trust client clocks for ordering.
- Clients sort by this server-assigned id, not arrival order (network reorders things).

### Online vs offline delivery — store-and-forward

The mobile reality: most recipients are offline most of the time. So **every message is written to a
durable per-user inbox / message queue first**, then delivery is a separate concern:

- **B online:** push immediately via backplane → socket, *and* the inbox row exists for durability.
- **B offline:** message sits in B's inbox. When B reconnects, the gateway **syncs everything since B's last-acked cursor**. Also fire an **APNs/FCM push** so B's phone surfaces a notification even with the app closed.

This **store-and-forward inbox per user** is the heart of reliable chat: the socket is best-effort,
the inbox is the source of truth.

### Group chat fan-out

A group message is fan-out-on-write into each member's inbox. Small groups (WhatsApp ~hundreds):
just write to every member's inbox — fine. Large broadcast groups / Slack channels with thousands:
this is the **celebrity problem** in disguise (one write → thousands of inbox writes + thousands of
pushes). Mitigate with batched fan-out workers, and for huge channels consider **fan-out-on-read**
(members pull the channel timeline) instead of writing to every inbox.

### Message storage — wide-column, time-ordered

Access pattern: "give me messages in conversation C, newest first, paginated." Write-heavy,
append-mostly, range-scan by time within a partition key. That's a textbook **wide-column** fit
(**Cassandra / HBase / DynamoDB / ScyllaDB**, and famously WhatsApp on Mnesia, Discord on Cassandra→ScyllaDB):

- **Partition key = `conversation_id`** (or a bucketed `conversation_id + time_bucket` to cap partition size).
- **Clustering key = `message_id` / timestamp**, descending → cheap "latest N" reads, natural ordering.
- LSM-tree under the hood = fast writes, which is exactly the workload.

Media never goes in the message row — store blobs in object storage (S3) and keep only a URL +
metadata in the message.

### Sync across devices

A user has phone + laptop + tablet. Model devices, not just users: deliver to **all of a user's
devices**, and track a **per-device last-acked cursor** so each device syncs independently from where
*it* left off. On reconnect: "send me everything after cursor X." This cursor is also your reconnect
+ resume mechanism (Part F).

### End-to-end encryption (note)

With true E2EE (Signal protocol — WhatsApp's basis), the **server only ever sees ciphertext**; it
routes and stores opaque blobs and cannot read content. Implications worth stating: server-side
search/indexing of content is impossible; group fan-out means encrypting per-recipient-device (each
device has its own keys); key exchange and multi-device key management become the hard part, not the
transport. Receipts/metadata may still be visible to the server even when content isn't.

---

## Part E — Designing a notification / fan-out service

A general "deliver events to many users across many channels" system. The two big axes are
**fan-out strategy** and **multi-channel delivery**.

### Fan-out on write vs read (the decision you must recite)

| | **Fan-out on write (push)** | **Fan-out on read (pull)** |
|---|---|---|
| When event happens | Write into every recipient's inbox/feed | Do nothing; record once |
| At read time | Read your own pre-built inbox (fast) | Gather + merge from all sources you follow (slow) |
| Good for | Most users, read-heavy, predictable | Celebrities / very high fan-out |
| Cost | Write amplification (1 event → N writes) | Read amplification (every read does the work) |

**The celebrity problem.** A user with 100M followers posting once = 100M writes if you fan out on
write. The classic answer is a **hybrid**: fan-out-on-write for normal accounts (fast reads for the
common case), but for celebrities **don't** fan out — their followers **pull** the celebrity's recent
posts at read time and merge them into the pushed feed. You decide per-author based on follower count.

> **Say this:** "Push gives fast reads but explodes for high-fan-out producers; pull is cheap to
> write but slow to read. I'd go hybrid — push for the long tail, pull for celebrities — and pick the
> strategy per-producer by follower count threshold."

### Dedup + idempotency

Notifications fire from at-least-once pipelines (queues redeliver), so duplicates are guaranteed
unless you defend against them:

- **Idempotency key** per logical notification (`event_id + user_id + channel`). Worker checks "already sent?" before sending; store sent-keys in Redis/dedup table with a TTL.
- **Dedup at the user level too:** collapse "5 people liked your post" into one notification rather than five.

### Rate limiting + batching

Nobody wants 50 pushes in a minute. Mitigations:

- **Per-user rate limits** on notifications (token bucket).
- **Digest / batching:** buffer low-priority notifications and send a rolled-up summary ("12 new messages") on a schedule or threshold.
- **Quiet hours / preferences** per user and per channel.

### Multi-channel: queue + worker per channel

```
event ─► Notification service ─► decides channels + applies prefs/dedup/rate-limit
                                      │
            ┌────────────┬────────────┼────────────┐
            ▼            ▼            ▼             ▼
        push queue    email queue   SMS queue    in-app queue
            │            │            │             │
        push worker   email worker  SMS worker   in-app worker
            ▼            ▼            ▼             ▼
        APNs/FCM       SES/SMTP     Twilio       WS/SSE push
```

- **A queue + dedicated worker pool per channel** so a slow/down provider (SMS gateway flaky) doesn't back up push or email — isolation/bulkhead.
- Each worker handles its provider's quirks, retries with backoff, and respects the **idempotency key** so retries don't double-send.
- **Templating + localization** in the service, channel-specific rendering in the worker.

---

## Part F — Live feed / activity feed and the push-vs-pull-vs-hybrid call

A home timeline / activity feed is the same fan-out problem with a read-merge step:

- **Pull model:** store each post once; at read time, fetch posts from everyone you follow and merge. Cheap writes, expensive reads (fan-in across thousands of followees), but always fresh.
- **Push model:** on post, write the post id into each follower's precomputed feed list (often a Redis list/sorted-set per user). Reads are a single fast lookup. Expensive on write, explodes for celebrities.
- **Hybrid (the real answer):** push for normal authors, pull for celebrities, merge the two at read time. Cache the merged feed.

Ranked feeds add a scoring/ML layer on top, but the *delivery* decision is still this push/pull/hybrid
triangle. Tie it back: chat group fan-out, notification fan-out, and feed fan-out are **the same
problem** wearing different clothes.

---

## Part G — Backpressure, buffering, reconnect & resume

Persistent connections introduce failure modes HTTP never had. Name them unprompted.

### Backpressure on slow clients

A client on bad mobile network can't drain messages as fast as you produce them. Your per-connection
send buffer grows → memory blows up → one slow client threatens the whole box. Defenses:

- **Bounded send buffers per connection.** When full, either **drop the connection** (let it reconnect and resync from cursor) or **shed/coalesce** low-value messages.
- **Coalesce stale state.** For presence/typing, only the latest matters — collapse the queue to the newest value rather than buffering every flip.
- **Don't let one slow consumer back-pressure the producer/backplane** — that's how a single phone takes down a gateway.

### Reconnect + resume (last-seen cursor)

Mobile sockets die constantly. Recovery must be cheap and lossless:

- Client tracks the **id/cursor of the last message it received**.
- On reconnect: handshake says "I'm at cursor X" → server replays everything after X from the durable inbox.
- This is why store-and-forward + monotonic message ids matter: **resume = "give me everything after X."** No cursor = either lost messages or a full re-sync.

### Ordering + dedup of real-time messages

The live push path and the resync path can both deliver the same message (you pushed it, then the
client reconnected and re-pulled it). So:

- **Every message carries a stable unique id** (Snowflake / per-conversation seq).
- **Client dedups by id** and **orders by id**, never by arrival order.
- Treat delivery as **at-least-once + idempotent consumer**, exactly like any queue. Exactly-once "delivery" is a myth; exactly-once *effect* via dedup is achievable.

---

## Part H — Worked mini-walkthrough: "Design a notification service"

A compressed run to show the shape.

1. **Requirements.** Deliver events (likes, mentions, system alerts) to users across push/email/SMS/in-app. Respect prefs + quiet hours, dedup, don't spam, never lose a high-priority alert. NFR: handle bursty fan-out (celebrity posts), at-least-once with idempotency, low end-to-end latency for in-app/push.

2. **Estimate.** 100M DAU, ~20 notifiable events touching a user/day → **2B notifications/day ≈ 23k/sec avg, ~70k/sec peak**. A celebrity post to 50M followers is a single spike of 50M fan-out writes — must be absorbed by a queue, not done inline.

3. **API.** `POST /notify {eventId, audience, payload, channels, priority}` → enqueues. `GET /notifications?userId=&cursor=` for in-app inbox.

4. **High-level.** Producer → **ingest service** (assigns `eventId`, validates) → **fan-out workers** (expand audience, apply hybrid push/pull for celebrities) → per-user **channel router** (prefs, dedup via idempotency key, rate-limit/batch) → **per-channel queues** → **per-channel workers** → APNs/FCM/SES/Twilio/WS.

5. **Deep dive — the celebrity fan-out.** 50M followers: don't fan out inline. Enqueue one fan-out job, **chunk into batches** (e.g., 10k recipients each) processed by a worker pool, each batch applying dedup + rate limits. For the very-high-fan-out case, prefer **pull** (followers' in-app feed pulls it) over writing 50M rows. Backpressure: the queue absorbs the spike; workers drain at a sustainable rate; high-priority channel (push) jumps a priority lane ahead of digest email.

6. **Reliability.** At-least-once queues + **idempotency key (`eventId+userId+channel`)** in a Redis dedup set so retries/redelivery never double-send. DLQ for poison messages. Per-channel isolation so a dead SMS provider can't stall push.

7. **Wrap-up failure modes.** Queue dies → use durable Kafka, replay from offset. Provider down → retry w/ backoff + DLQ + circuit breaker. Dedup store loses data → at worst occasional duplicate, acceptable for notifications. Thundering reconnect after an outage → jittered backoff + the resume cursor.

> The same skeleton **is** the chat system (inbox = in-app channel, gateway push = the WS worker) and
> the feed system (fan-out workers + hybrid). One mental model, three prompts.

---

### Self-check before the mock (answer these from memory)
- [ ] Give the 5 delivery mechanisms and the one-line "use when" for each. Why is push (APNs/FCM) not optional for mobile?
- [ ] What's the C10M problem, and roughly how many connections per box do you quote — and what's the *real* limiting factor?
- [ ] Draw the dumb-edge / smart-core split. Why do gateways carry no business logic?
- [ ] "User A messages user B who's on a different box." Trace it through the connection registry and backplane. Redis vs Kafka for the backplane — which for which path?
- [ ] Why is presence a fan-out problem, and name three ways to bound the fan-out.
- [ ] In chat: why persist *before* you ACK? What is the store-and-forward inbox and why is it the source of truth, not the socket?
- [ ] Which store for messages and what are the partition + clustering keys, and why?
- [ ] Fan-out on write vs read, the celebrity problem, and the hybrid — state it in one breath.
- [ ] How does reconnect-and-resume work, and why does it require message ids + a durable inbox?
- [ ] Why is exactly-once *delivery* a myth, and how do you get exactly-once *effect*?
