# Design 13: A Notification System (Multi-Channel, at Scale)

> **How to read this doc:** This is a full 45-minute worked answer, run through the 7-step
> framework from `prep/01-framework-and-building-blocks.md`. The blockquotes marked
> **"Say this in the room"** are the lines a strong candidate would actually speak. The point is
> not to memorize *this* design — it's to watch the framework *derive* it, so you can derive an
> unseen one the same way. Building-block cross-references (e.g. *see messaging deep dive*) point at
> the toolkit in Part B of the framework doc and at the topic docs in `prep/`.

---

## Step 1 — Requirements (5 min)

> **Say this in the room:** "A notification system looks like a CRUD app and is actually a
> distributed-systems problem in disguise — the hard parts are fan-out, at-least-once delivery
> through flaky third parties, dedup, and keeping a 2FA code fast while a 50-million-user marketing
> blast is draining the queues. Let me pin requirements and scope before I draw anything."

### Functional (the verbs)

- **Send a notification** — an upstream service (or a campaign tool) asks us to notify *a user* or *a set of users* of an event ("your order shipped", "you have a new follower", "here's your login code").
- **Deliver across channels** — **push** (mobile/web via APNs/FCM), **email** (via SES/SendGrid), **SMS** (via Twilio), and **in-app** (a feed/inbox the client polls or streams).
- **Respect user preferences** — per-user, per-category, per-channel opt-in/opt-out; quiet hours; frequency caps. A user who muted "marketing email" must never get one.
- **Template & personalize** — senders pass a `templateId` + variables, not raw strings. We render localized, personalized content per channel.
- **Schedule** — send now, send at a future time, or send in the recipient's local morning ("send at 9am their time").
- **Track delivery** — sent / delivered / opened / clicked / failed, exposed to senders for analytics.

### Explicitly out of scope (say this — it shows seniority and protects your time)

- The **mobile/web client SDK** and how it registers device tokens (I'll assume a `device_tokens` table exists and is kept fresh).
- The **campaign authoring UI** / marketing tool — I consume its output, I don't build it.
- **Content moderation / spam classification** of notification bodies.
- The **realtime transport** for in-app (WebSocket/SSE gateway) — I'll treat it as a downstream "in-app channel" and reference the realtime design (`prep/15`, Design "realtime/push") rather than rebuild it.

> **Say this in the room:** "I'll focus on **ingest → route → fan-out → per-channel delivery →
> tracking**, with deep dives on fan-out, at-least-once reliability + dedup, preferences/throttling,
> and isolating the critical OTP path. That's where the system lives or dies."

### Non-functional (where staff candidates separate themselves)

- **Write-heavy, bursty, fan-out-shaped.** This is *not* a read-heavy app like a feed. The defining trait is **amplification**: one upstream event can become millions of channel sends, and traffic is spiky (a marketing blast, a "your team scored" sports event, an outage alert to all users).
- **Reliability — at-least-once delivery.** A "your payment failed" or OTP notification that silently vanishes is a serious bug. We accept **at-least-once** (a notification may occasionally be *attempted* twice) and lean on **dedup/idempotency** to make duplicates rare and *exactly-once-ish* at the user's eyes. Exactly-once across third-party providers is impossible (the provider may deliver and then we crash before recording it), so we design for at-least-once + dedup.
- **Latency — two very different SLAs by priority.** A 2FA/OTP code must reach the provider in **< 1–2 s p99**; a marketing email can take **minutes**. This split is *the* load-bearing non-functional requirement — it forces priority lanes (deep dive 6.6).
- **Availability over strict freshness.** 99.99% on the critical (transactional/OTP) path; marketing can tolerate brief degradation. We never let a marketing backlog starve OTP.
- **Consistency:** preferences must be read **strongly enough** that an opt-out is honored within seconds (compliance — CAN-SPAM/GDPR/TCPA). Delivery-status tracking can be **eventually consistent** (analytics).
- **Durability:** once we *accept* a send request (200 from the ingest API), we must not lose it. The request is the source of truth; everything downstream is retriable derived work.

> **Say this in the room:** "The two facts that shape everything: it's **fan-out-heavy and bursty**,
> and it has **two latency classes** — sub-second OTP vs minutes-OK marketing. Those derive priority
> lanes, durable queues, and provider-rate-limit handling before I've drawn a box."

---

## Step 2 — Estimation (3 min)

> **Say this in the room:** "I want numbers that justify three decisions: do I need durable queues
> and sharding (volume), how bad is the fan-out spike (burst), and how big is the tracking store
> (analytics). I'm not chasing precision."

Assume a large consumer platform.

### Volume and QPS

| Quantity | Assumption | Result |
|---|---|---|
| Total users | given | **500 M** |
| Notifications per user per day (all channels, avg) | mix of transactional + marketing | **4** |
| **Notifications/day** | 500M × 4 | **2 B/day** |
| **Avg send QPS** | 2B ÷ 86,400 | **~23,000/s** |
| **Peak send QPS** | 5× avg (marketing blasts cluster) | **~115,000/s** |
| Channel multiplier (a "notification" may hit 1–3 channels) | avg 1.4 sends/notification | ×1.4 |
| **Peak *channel-send* QPS** | 115k × 1.4 | **~160,000/s** |

> **Say this in the room:** "~23k/s average is healthy but not scary; the story is the **peak
> multiplier**. Marketing is bursty and clustered, so I size for ~115–160k channel-sends/sec at
> peak — well past one box and past most providers' rate limits. That alone justifies durable
> queues that *absorb the burst* and a sharded worker fleet that drains at the providers' allowed
> rate (*see messaging deep dive, prep/07*)."

### The fan-out / broadcast number (the one that drives the design)

| Quantity | Assumption | Result |
|---|---|---|
| Routine event → recipients | "your order shipped" | **1** |
| Topic event → recipients | "team you follow scored" | **10k–1M** |
| **Broadcast** → recipients | "service maintenance tonight" to all users | **500 M** |
| Time to fan out a 500M broadcast at 160k/s | 500M ÷ 160k | **~52 min** of pure drain |

> **Say this in the room:** "Here's the punchline: a single broadcast is **500 million sends** that
> take ~50 minutes to drain even at peak throughput. That asymmetry — 1 recipient vs 500,000,000 —
> is the notification version of the Twitter celebrity problem (*see Design 1 / prep/15*). It means
> (a) ingest must accept a broadcast as **one tiny request** and expand it asynchronously, never
> inline; and (b) that 50-minute marketing drain must **not** sit in the same queue as OTP. The
> estimate just told me I need fan-out expansion + priority lanes."

### Tracking storage / year

| Item | Math | Result |
|---|---|---|
| Tracking events per send (queued, sent, delivered, opened…) | ~4 | 4 |
| Tracking rows/day | 2B sends × 4 | **8 B/day** |
| Bytes/row | id + status + ts + ids | **~200 B** |
| **Tracking writes/day** | 8B × 200B | **~1.6 TB/day** |
| **Tracking store / year (hot 90d + cold)** | 1.6TB × 90 hot, rest cold/aggregated | **~150 TB hot, roll up to cold** |

> **Say this in the room:** "Tracking is a **firehose** — 8B events/day, ~1.6 TB/day. That's not an
> OLTP table; it's a **stream → columnar/OLAP** pipeline (Kafka → ClickHouse/BigQuery, *see prep/18
> stream processing*). I keep ~90 days hot for dashboards and roll the rest into aggregates. This is
> why tracking is async and eventually consistent — I will not put the delivery hot path behind an
> analytics write."

---

## Step 3 — API Design (3 min)

REST over HTTPS, JSON, for senders (internal services + campaign tool). Auth + rate-limiting terminate at the **API gateway** (*see prep/09 rate limiter / LB*); the caller is an authenticated *service* or *tenant*, and `senderId` comes from its credential, never the body.

```
POST /v1/notifications
  Authorization: Bearer <service-token>
  Idempotency-Key: <client-supplied unique key>          # REQUIRED — see reliability deep dive
  body: {
    recipients:  { userIds: [...] } | { segmentId } | { topic } | "ALL",
    templateId:  "order_shipped_v3",
    data:        { orderId: "...", eta: "..." },          # template variables
    channels:    ["push","email"]   | "auto",             # auto = use prefs/routing
    priority:    "transactional" | "marketing",           # picks the lane
    sendAt:      <iso8601> | "now" | "local:09:00",        # scheduling
    category:    "shipping"                                # for preference/opt-out matching
  }
  -> 202 { requestId }     # ACCEPTED, not "delivered" — async by design

GET  /v1/notifications/{requestId}/status
  -> 200 { requestId, accepted, perChannel: { push:{sent,delivered,failed}, ... }, perUser?: ... }

# Preferences (read/write by the user-facing app, separate service)
GET  /v1/users/{userId}/preferences
PUT  /v1/users/{userId}/preferences
  body: { channels:{email:{marketing:false, shipping:true}, sms:{...}}, quietHours:{tz, 22:00-07:00} }

# Provider webhooks (delivery receipts come back asynchronously)
POST /v1/webhooks/{provider}        # APNs/FCM/SES/Twilio call us with delivered/bounced/opened
```

### Key API choices (call these out)

- **`POST` returns `202 Accepted`, not `200 delivered`.** Delivery is asynchronous through third parties; we *accept and durably enqueue*, then return. The contract is "we will attempt this at-least-once," surfaced via the status endpoint and webhooks. *(Sync vs async tradeoff — the whole system is async by necessity.)*
- **`Idempotency-Key` is required.** Because the network between sender and us is unreliable and the sender will retry, the key lets us collapse a retried `POST` into the *same* request — the first line of defense against duplicate notifications (*see prep/10 idempotency*).
- **Recipients can be `userIds`, a `segmentId`, a `topic`, or `"ALL"`.** A broadcast is a *tiny* request (`"ALL"` + a templateId), expanded server-side. The 500M-recipient blast must never be 500M items in a request body — that's the estimate (Step 2) made into an API rule.
- **`priority` picks the lane** (transactional vs marketing). The sender declares intent; we enforce isolation downstream.
- **`channels: "auto"`** delegates channel selection to the preference/routing engine rather than forcing the sender to know the user's prefs.

---

## Step 4 — Data Model (5 min)

> **Say this in the room:** "I pick stores from *access patterns*. Notifications have a few very
> different shapes — a high-churn append-only delivery log, a small strongly-read preferences store,
> a template store, and an analytics firehose — so I'll use different stores for each rather than
> one DB."

### Entities and access patterns

**`notification_requests`** — the accepted, source-of-truth send request. Written once on `POST`, read by `requestId` (point) and for retry/audit. Moderate volume (2B/day of *requests*, fewer than sends). Append-heavy, keyed by `requestId`.
→ **Sharded by `requestId`** in a write-optimized store (Cassandra/DynamoDB; *see prep/04 sharding, LSM block*). The `Idempotency-Key` is a unique secondary key so a duplicate `POST` finds the existing request.

```
notification_requests
  PK: requestId               # Snowflake-style, time-sortable
  idempotencyKey  (unique)    # dedup at ingest
  senderId, templateId, data, channels, priority, category, sendAt
  status: accepted | expanding | done
```

**`device_tokens`** — `userId → [ {channel:push, token, platform:ios|android, valid} ]`. Point lookup by `userId` at routing time. Kept fresh by the client SDK (out of scope). KV-shaped → **Cassandra / Dynamo**.

**`user_preferences`** — `userId → { per-channel × per-category opt-in, quietHours{tz}, frequencyCaps }`. **Read on every send** (must honor opt-out), low write volume, must be **strongly consistent enough** that an opt-out is honored within seconds.
→ Source of truth in a **replicated SQL/Dynamo** store, fronted by a **cache with short TTL + write-through invalidation** on preference change (*see prep/06 caching*). Compliance makes correctness here non-negotiable.

```
user_preferences
  PK: userId
  channels: { email:{marketing:false, security:true}, sms:{...}, push:{...} }
  quietHours: { tz:"America/LA", from:"22:00", to:"07:00" }
  caps: { marketing: { perDay: 3 } }
```

**`templates`** — `templateId → { perChannel bodies, localized variants, variable schema }`. Read on every send (hydrated, then cached hard — templates change rarely). Versioned (`order_shipped_v3`) so a send pins a version.
→ SQL/object store + **aggressive cache** (immutable per version).

**`dedup_keys`** — short-lived idempotency/collapse keys for *delivery* (distinct from ingest idempotency). `key → ttl`. Used to suppress "same notification to same user on same channel within window."
→ **Redis with TTL** (*see prep/06, prep/10*).

**`delivery_log` (tracking)** — the firehose. `(requestId, userId, channel) → status transitions`. Append-only, huge (8B events/day), read mostly in aggregate.
→ **Kafka → columnar/OLAP** (ClickHouse/BigQuery), *not* an OLTP table (*see prep/18*).

**`user_notifications` (in-app inbox)** — `userId → [ recent in-app notifications ]`, capped, read by the client. Redis ZSET + durable backing (the Twitter timeline pattern, *see Design 1*).

> **Say this in the room:** "The load-bearing modeling decision: **preferences are read strongly and
> cached carefully because opt-out is a compliance requirement**, while **tracking is a fire-and-
> forget stream**. Conflating those — putting analytics in the send path, or treating opt-out as
> eventually consistent — is how this design fails in either latency or in a lawsuit."

---

## Step 5 — High-Level Design (10 min)

> **Say this in the room:** "Let me get the happy path end-to-end with the simplest thing that
> works, then evolve it under questioning. The spine is: ingest → expand fan-out → route by prefs →
> per-channel queue → per-channel worker → provider → track receipts."

```
                       ┌──────────────────────────────────────────────┐
   Internal services / │                 API Gateway                   │
   campaign tool ─────►│  TLS · authn · rate-limit · routing (L7 LB)   │
                       └───────────────────┬──────────────────────────┘
                                           │ 202 (after durable enqueue)
                                           ▼
                                 ┌──────────────────────┐
                                 │  Ingestion Service    │  write requestId + idempotencyKey
                                 │  (idempotency check)  │──► notification_requests (sharded)
                                 └──────────┬────────────┘
                                            │ publish "send.requested"
                                            ▼
                                   ┌──────────────────┐
                                   │  Kafka: requests  │  (priority-partitioned)
                                   └────────┬─────────┘
                                            ▼
                                 ┌──────────────────────┐   reads segment/topic/ALL,
                                 │  Fan-out / Expander   │   pages recipient sets,
                                 │  Workers              │   emits ONE msg per (user,channel)
                                 └──────────┬────────────┘
                                            │ per recipient:
                  reads ─────────► preferences cache  +  device_tokens  +  templates(render)
                                            │  (apply opt-out, quiet hours, freq cap, dedup)
                                            ▼
        ┌───────────────┬───────────────┬───────────────┬──────────────────┐
        ▼               ▼               ▼               ▼     (priority lanes ×N)
  ┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐
  │ push Q    │   │ email Q   │   │ sms Q     │   │ in-app Q  │   per-channel,
  │ (txn/mkt) │   │ (txn/mkt) │   │ (txn/mkt) │   │           │   per-priority
  └────┬─────┘    └────┬─────┘    └────┬─────┘    └────┬─────┘
       ▼               ▼               ▼               ▼
  ┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐
  │push worker│   │email wkr │   │ sms wkr  │   │in-app wkr│  rate-limited per provider,
  │ fleet    │    │ fleet    │   │ fleet    │   │ fleet    │  retry+backoff, DLQ
  └────┬─────┘    └────┬─────┘    └────┬─────┘    └────┬─────┘
       ▼               ▼               ▼               ▼
   APNs/FCM           SES/…          Twilio/…       WS/SSE gateway + inbox store
       │                                                  
       └──── webhooks (delivered/bounced/opened) ─────►  Tracking Service ──► Kafka ──► OLAP
```

### Happy path — accepting a send (write)

1. Sender `POST /v1/notifications` with `Idempotency-Key` → gateway authenticates, rate-limits, routes to **Ingestion Service**.
2. Ingestion checks the idempotency key: if seen, return the existing `requestId` (no double-accept). Otherwise it **durably writes** the `notification_request` (this is the ack-worthy step) and **publishes `send.requested`** to Kafka on the lane matching `priority`.
3. Return **`202 {requestId}`**. The sender is done — it is *not* blocked on expansion or delivery.

### Happy path — expansion + routing (the fan-out)

4. **Fan-out / Expander workers** consume `send.requested`. They resolve recipients: a `userIds` list is tiny; a `segmentId`/`topic`/`ALL` is **paged** from the segment store and expanded into **one message per (user, channel)** — never loaded whole. For each recipient they:
   - read **user_preferences** (cache) → drop channels the user opted out of, defer if in quiet hours, drop if over a frequency cap;
   - read **device_tokens** → resolve the concrete address per channel;
   - render the **template** (localized, personalized);
   - compute a **dedup key** and `SET NX` in Redis — if already present, skip (collapse duplicates);
   - emit the rendered, addressed message onto the **per-channel, per-priority queue**.

### Happy path — delivery (per-channel worker)

5. **Per-channel worker fleets** consume their queue, **token-bucket rate-limited to the provider's allowed rate**, and call the external provider (APNs/FCM/SES/Twilio). On success → write a `sent` tracking event. On failure → **retry with exponential backoff + jitter**; after N attempts → **dead-letter queue** (*see prep/13 resilience*).
6. Providers later call our **webhook** with `delivered`/`bounced`/`opened`; the **Tracking Service** records these to Kafka → OLAP. In-app sends additionally write to the user's inbox ZSET and notify the realtime gateway.

> **Say this in the room:** "Notice the shape: ingest is a fast durable enqueue, expansion is async
> and paged so a broadcast can't block, and each channel has its *own* queue + worker fleet so I can
> rate-limit and fail per provider independently. The two expensive/risky things — fan-out
> amplification and flaky third-party calls — are both pushed onto async workers behind durable
> queues. That's the entire reliability strategy in one sentence."

---

## Step 6 — Deep Dives (15 min) — *where the round is won*

> **Say this in the room:** "The interesting challenges are: fan-out for broadcasts, at-least-once
> reliability with dedup through flaky providers, preferences/throttling done correctly, and keeping
> the OTP path isolated from the marketing flood. Let me go deep on those."

### 6.1 Fan-out: one event → many users (the broadcast problem)

This is the **push-amplification** problem — the notification cousin of Twitter's celebrity fan-out (*see Design 1, prep/15*).

- **Ingest must accept a broadcast as O(1).** `"ALL"` or a `segmentId` is one tiny request. The expander does the amplification, **paging** the recipient set (e.g. scan 500M users in pages of 10k) and emitting messages incrementally. If we tried to materialize 500M recipients inline, ingest would time out and we'd lose the request.
- **Parallelize expansion by partitioning the recipient set.** The expander shards the scan (by user-id range / segment shard) across many workers so a 500M broadcast fans out in parallel, not serially. Kafka **absorbs the burst** while per-channel workers drain at the providers' allowed rate (the queue is the shock absorber — *see prep/07*).
- **Batch the provider calls.** Don't make 500M individual HTTP calls. APNs/FCM accept **multicast/batch** (hundreds of tokens per request); SES has bulk send. Workers **coalesce** queue messages into provider batches → fewer round-trips, respects provider rate limits, dramatically higher throughput.
- **Don't starve transactional traffic.** The 50-minute marketing drain runs on the **marketing lane**; OTP runs on the **transactional lane** with its own workers and provider quota (deep dive 6.6).

| Recipient shape | Ingest cost | Expansion | Notes |
|---|---|---|---|
| Single user (txn) | O(1) | O(1) | fast lane, low latency |
| Topic (10k–1M) | O(1) | paged, parallel | marketing/topic lane |
| Broadcast (`ALL`, 500M) | O(1) | paged, parallel, batched | absorbed by queue; ~tens of min drain |

> **Say this in the room:** "The rule is **accept small, expand async, batch the provider calls, and
> never let the broadcast share a lane with OTP.** Same asymmetry as the Twitter celebrity — the
> tail case (500M) gets special handling so the common case (1 recipient) stays fast."

### 6.2 Reliability: at-least-once delivery + retries + DLQ

We promise **at-least-once**. Exactly-once is unachievable across a third party (we can call Twilio, succeed, then crash before recording success → we'll retry → user gets two SMS). So we make at-least-once *behave like* exactly-once via dedup (6.3), and we make "at least once" actually hold under failure:

- **Durable queues + manual ack.** A worker acks a Kafka message **only after** the provider confirms acceptance (or after writing it to the DLQ). Crash mid-flight → message is redelivered → retried. Nothing is dropped (*see prep/07, prep/13*).
- **Retry with exponential backoff + jitter.** Provider returns 5xx / times out → retry at 1s, 2s, 4s, … with jitter to avoid synchronized retry storms. Cap attempts (e.g. 5).
- **Dead-letter queue.** After max attempts, the message goes to a **DLQ** with full context, alerting + manual/automated replay. The pipeline never blocks on one poison message (*see prep/13 resilience — DLQ, bulkhead*).
- **Distinguish retryable vs terminal failures.** A `429` (rate limit) → back off and retry. A `400 invalid token` / hard bounce → **terminal**: don't retry, mark the device token invalid, emit a `failed` event. Retrying a permanent failure just wastes quota.
- **Circuit breaker per provider.** If APNs starts failing en masse, trip the breaker so we stop hammering it, shed/queue that channel, and **fail fast to a fallback channel** if the notification is critical (6.5) rather than burning the retry budget (*see prep/13 circuit breaker*).

> **Say this in the room:** "At-least-once is a *choice*: I'd rather a user occasionally sees a
> duplicate than misses a payment alert. I make it hold with manual ack + backoff + DLQ, I split
> retryable from terminal failures so I don't waste provider quota, and I put a circuit breaker per
> provider so one bad provider can't take the fleet down."

### 6.3 Deduplication + idempotency (not sent twice) + collapsing/digest

Three distinct levels — name them separately, this is a senior signal (*see prep/10 idempotency*):

1. **Ingest idempotency (sender retry).** The required `Idempotency-Key` on `POST`: a retried request collapses to the same `requestId`. Stored as a unique key on `notification_requests` with a TTL window.
2. **Delivery dedup (at-least-once side effect).** Before a worker sends, it computes a **dedup key** = `hash(userId, channel, templateId, dedupWindow)` and does `SET key NX EX <window>` in Redis. If the key exists, **skip the send** — this collapses the "we retried and the first attempt actually succeeded" case and any accidental double-expansion. This is what turns at-least-once into *looks-like-once* at the user (*see prep/06 Redis, prep/10*).
   - Caveat: dedup is best-effort across the crash-after-send gap (we may set the key *after* the provider call). To tighten it, set the dedup key **before** the call and treat its presence as "in-flight or done"; reconcile via the provider receipt. There's an irreducible tradeoff between "never duplicate" and "never drop" — we bias to never-drop for transactional, never-duplicate-aggressively for marketing.
3. **Collapsing / bundling (digest).** For high-volume low-urgency events ("5 people liked your post"), don't send 5 notifications — **collapse** them. A short **collapsing window** buffers events per (user, category); on window close, render a **digest** ("5 new likes"). This is a product feature *and* a load reducer. APNs/FCM also support a `collapse-id` so a newer push replaces an older undelivered one on the device.

> **Say this in the room:** "I separate **ingest idempotency** (sender retried the POST),
> **delivery dedup** (we retried the send), and **collapsing** (many events → one digest). They're
> often muddled into 'dedup'; calling them out separately and giving each its own mechanism — unique
> key, Redis SET NX, collapsing window — is what makes this robust."

### 6.4 User preferences, opt-out, quiet hours, channel routing, frequency capping

This is the **policy engine** that runs per recipient during expansion. Correctness here is a **compliance** requirement (CAN-SPAM/GDPR/TCPA), not a nicety.

- **Opt-out / preferences.** Per (channel × category) flags. Read from the preferences cache (short TTL, write-through invalidation on change) so an opt-out is honored within seconds. **Hard rule:** transactional/security messages (OTP, fraud alert) **bypass marketing opt-out** but *not* a hard channel-level legal opt-out — model the distinction explicitly.
- **Quiet hours.** Per-user timezone + window (e.g. 22:00–07:00 local). A notification arriving in quiet hours for a *deferrable* category is **scheduled to the window's end** rather than dropped (ties into scheduling, 6.7). Critical messages ignore quiet hours.
- **Channel routing (`channels:"auto"`).** Choose channel by preference + criticality + reachability: e.g. prefer push; if no valid device token, fall back to email; OTP may route to SMS. This is also the **fallback** mechanism for 6.5.
- **Frequency capping (per-user rate limiting).** "No more than 3 marketing notifications/day." Implemented as a **per-user token bucket / counter in Redis** keyed `(userId, category, day)` — the same primitive as edge rate limiting applied per *user* instead of per *IP* (*see prep/09 rate limiting*). Over cap → drop or roll into a digest. Critical messages are exempt.

> **Say this in the room:** "Preferences are a per-recipient policy filter in the expansion stage:
> opt-out, quiet hours, routing, frequency cap. Two senior points — (1) it's **compliance-critical**
> so I read it strongly/cache carefully, and (2) **transactional bypasses marketing caps but honors
> legal opt-out** — getting that distinction wrong either annoys users or breaks the law."

### 6.5 Handling provider failures, timeouts, and provider rate limits

The external providers are the least reliable part of the system; treat them as hostile.

- **Provider rate limits.** Each provider caps our throughput (APNs ~thousands/s/connection, Twilio per-number limits, SES per-account). Workers are **token-bucket throttled to the provider's allowed rate**, *not* the queue's arrival rate — the queue holds the backlog while we drain politely. Exceeding the limit gets us `429`s and possibly throttled/banned.
- **Timeouts + circuit breaker.** Bound every provider call with a timeout; on a spike of failures, trip a **circuit breaker** per provider to stop hammering a dead provider and fail fast (*see prep/13*).
- **Multi-provider failover.** For critical channels, configure **secondary providers** (e.g. SES *and* SendGrid; multiple SMS aggregators). If the primary's breaker is open, route to the secondary. This removes the provider as a SPOF.
- **Channel fallback for critical notifications.** If push fails/undeliverable for an OTP, **fall back to SMS or email**. The routing engine (6.4) owns this. We'd rather deliver an OTP on a worse channel than not at all.
- **Bulkhead isolation.** A separate worker pool / connection pool per provider so a slow provider can't exhaust threads shared with a healthy one (*see prep/13 bulkhead*).

> **Say this in the room:** "I throttle to the *provider's* rate, not mine, with the queue as buffer;
> I circuit-break and bulkhead per provider so one bad provider is contained; and for critical
> notifications I keep a secondary provider and a cross-channel fallback. The provider is the most
> likely thing to fail, so it gets the most resilience machinery."

### 6.6 Prioritization: OTP/2FA must be fast vs marketing batch (lane isolation)

This is the **single most important architectural decision** and follows directly from the two-SLA requirement.

- **Separate queues/lanes per priority**, end to end: a **transactional lane** (OTP, payment, fraud, security) and a **marketing/bulk lane** (campaigns, digests, topics). They do **not** share Kafka partitions, worker fleets, *or provider quota*.
- **Why physical separation, not just priority ordering:** a 500M marketing broadcast (Step 2: ~50 min of drain) sitting ahead of an OTP in a shared queue would delay the OTP by minutes — unacceptable. Even priority *ordering* within one queue risks head-of-line blocking and shared-worker contention. **Physical lane isolation** guarantees the OTP path's latency is independent of marketing volume — this is **bulkheading** applied to queues (*see prep/13*).
- **Reserve provider capacity** for the transactional lane (e.g. a dedicated APNs connection pool / Twilio number set) so marketing can't consume the provider quota OTP needs.
- **Autoscale lanes independently** on per-lane consumer lag. The transactional lane is small and over-provisioned for latency; the marketing lane scales for throughput.

> **Say this in the room:** "I physically isolate the transactional lane from the marketing lane —
> separate queues, workers, *and* provider quota — because a 50-minute broadcast must never add a
> minute of latency to a 2FA code. This is bulkheading: the blast radius of a marketing flood stops
> at its own lane. If I get only one thing right in this design, it's this."

### 6.7 Templating / personalization + localization + scheduling

- **Template service.** Senders pass `templateId + data`, never raw bodies. The service renders **per-channel** variants (push has a title+body+payload, email has HTML+text, SMS is short plaintext) from a versioned template. Rendering happens in the **expansion** stage, after preferences resolve the channel set.
- **Localization.** Template stores per-locale variants; render in the recipient's locale (from their profile). Falls back to a default locale. Keeps copy out of sender code and lets non-engineers manage content.
- **Personalization** substitutes `data` variables (name, order id) safely (escaped per channel).
- **Scheduling.** `sendAt` can be a future time or `local:09:00`. A **scheduler** (a time-bucketed store / delay queue, e.g. a sorted set keyed by fire-time, or a dedicated scheduling service) holds scheduled requests and **releases them into the ingest pipeline at fire time**. "Local 9am" expands per-recipient timezone. Quiet-hours deferral (6.4) reuses this scheduler.

> **Say this in the room:** "Templates + localization keep content out of sender code and let me
> render the right shape per channel. Scheduling is just a delay queue that releases into the same
> pipeline — and quiet-hours deferral reuses it, so I don't build that twice."

### 6.8 Tracking: sent / delivered / opened (async analytics)

- **Two sources of truth for status:** our workers emit `queued`/`sent`/`failed`; **provider webhooks** emit `delivered`/`bounced`/`opened`/`clicked` asynchronously (minutes later). Both flow into the **Tracking Service**.
- **Stream, don't OLTP it.** 8B events/day (Step 2) → publish to **Kafka**, sink to a **columnar/OLAP store** (ClickHouse/BigQuery) and a real-time aggregation pipeline for dashboards (*see prep/18 batch & stream processing*). The send hot path **never** waits on a tracking write — fire-and-forget.
- **Feedback loop.** Hard bounces / `invalid token` events feed back to **invalidate device tokens** and suppress future sends to dead addresses (protects deliverability/reputation, esp. email).
- **Eventually consistent by design** — the status endpoint reflects what's landed so far; that's acceptable for analytics and good enough for senders.

> **Say this in the room:** "Tracking is a firehose, so it's a stream → OLAP pipeline, completely
> off the delivery path and eventually consistent. The one part that loops back synchronously-ish is
> **bounce handling**: a hard bounce invalidates the token so I stop wasting sends and protect sender
> reputation."

---

## Step 7 — Wrap-Up (3 min)

> **Say this in the room:** "Let me name what still hurts, how it fails, and what I'd do with more
> time — before you ask."

### Remaining bottlenecks
- **Expander throughput on giant broadcasts** — mitigated by parallel paged expansion + provider batching; ultimately bounded by provider rate limits, which is why the queue absorbs the burst.
- **Preferences read on every send** — mitigated by a short-TTL cache with write-through invalidation; the hot path is a cache read, not a DB read.
- **Tracking firehose** — handled off-path via Kafka → OLAP; never blocks delivery.

### Failure modes (and graceful degradation)
- **A provider is down (e.g. Twilio).** Circuit breaker trips → failover to a secondary SMS provider, or cross-channel fallback for critical messages; non-critical SMS backs up in its queue and drains on recovery. **Degrade, don't fail.**
- **Queue backlog / consumer lag spikes** (big campaign). Kafka absorbs it; marketing-lane workers autoscale on lag; **the transactional lane is unaffected** because it's physically isolated. Freshness of marketing degrades by minutes — acceptable.
- **Poison message** (a malformed render). Bounded retries → DLQ with alerting + replay; the pipeline never stalls on it.
- **We crash after sending but before recording.** At-least-once → on retry the dedup key (6.3) suppresses the duplicate; worst case the user sees one extra notification.
- **Preferences store/cache down.** Fail **closed for marketing** (don't risk sending to opt-outs → compliance) and **open for critical** (still deliver OTP/security). Naming this fail-open-vs-closed split by criticality is the senior move.

### Single points of failure
- No single queue, store, or provider is a SPOF: Kafka **partitioned + replicated**; request/token/preference stores **sharded + replicated**; **multi-provider** per channel; worker fleets run N replicas, autoscaled, behind health checks; gateway/LB tier multi-AZ. The **transactional lane is isolated** so the most failure-prone component (bulk marketing volume) cannot take down the critical path.

### With more time, I'd add
- **In-app realtime delivery** detail (WebSocket/SSE fan-out gateway — *see prep/15 / realtime design*).
- **A/B testing + send-time optimization** (ML for best per-user send time) on the marketing lane.
- **Multi-region active-active** with regional queues/workers and per-region provider endpoints for latency + isolation.
- **Deliverability/reputation management** for email (warmup, DKIM/SPF, bounce/complaint thresholds).
- **Per-tenant quotas and fairness** so one noisy sender can't starve others (another bulkhead).

---

## What made this a staff-level answer

- **The estimate drove the architecture, not vibes.** The "1 vs 500,000,000 recipient" fan-out number and the "~50-minute broadcast drain" *derived* both async paged expansion and physical priority-lane isolation before any box was drawn.
- **Named the right core tradeoff and picked a side:** **at-least-once + dedup over unachievable exactly-once**, with the three dedup levels (ingest idempotency / delivery dedup / collapsing) called out separately — the precise, senior framing.
- **Isolated the critical path on purpose.** Bulkheaded the OTP lane — separate queues, workers, *and* provider quota — so marketing volume can never add latency to a 2FA code. This is the single highest-leverage decision and it was justified, not asserted.
- **Treated third-party providers as the most-likely-to-fail component** and wrapped them in the full resilience kit: per-provider rate limiting, circuit breakers, bulkheads, multi-provider failover, cross-channel fallback (*see prep/13*).
- **Made preferences a compliance-grade concern** — strong reads, fail-open-for-critical / fail-closed-for-marketing — rather than a CRUD afterthought.
- **Kept analytics off the hot path** — an 8B-event/day firehose into a stream→OLAP pipeline, eventually consistent, with a bounce feedback loop that protects deliverability.
- **Named failure modes and graceful degradation before being asked**, and every one degrades rather than fails.

> **One-line takeaway:** *A fan-out-heavy, bursty, two-SLA system → accept small + expand async +
> batch the providers, make at-least-once look like once with layered dedup, wrap every flaky
> provider in resilience machinery, and **physically isolate the OTP lane from the marketing flood**
> so the critical path's latency is independent of volume.*

---

## Self-check (answer from memory before the mock)

- [ ] Why is exactly-once impossible here, and what two mechanisms make at-least-once *behave* like once?
- [ ] Name the three distinct dedup/collapsing levels and the mechanism for each.
- [ ] Why physically isolate the OTP lane instead of just using priority ordering in one queue?
- [ ] How does a 500M-recipient broadcast stay an O(1) API request? Where does the amplification happen?
- [ ] Which way do you fail (open vs closed) when the preferences store is down — and why does it differ by message criticality?
- [ ] Why is tracking a stream→OLAP pipeline and not an OLTP table? What's the one synchronous-ish feedback loop?
- [ ] What's the notification analog of the Twitter celebrity problem, and what's the shared mitigation pattern?
