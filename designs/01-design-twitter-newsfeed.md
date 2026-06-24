# Design 1: Twitter / a News Feed System

> **How to read this doc:** This is a full 45-minute worked answer, run through the 7-step
> framework from `prep/01-framework-and-building-blocks.md`. The blockquotes marked
> **"Say this in the room"** are the lines a strong candidate would actually speak. The point is
> not to memorize *this* design — it's to watch the framework *derive* it, so you can derive an
> unseen one the same way. Building-block cross-references (e.g. *see caching deep dive*) point at
> the toolkit in Part B of the framework doc.

---

## Step 1 — Requirements (5 min)

> **Say this in the room:** "Before I draw anything, let me pin functional and non-functional
> requirements and scope this down. Twitter is huge — I want to nail the core feed loop and skip
> the rest unless you want it."

### Functional (the verbs)

- **Post a tweet** — a user publishes text (≤ 280 chars), optionally with a media reference.
- **Follow / unfollow** a user — this builds the social graph that drives the feed.
- **Read the home timeline** — a user opens the app and sees a feed of recent tweets *from the people they follow*, newest-ish first, paginated.
- **View a user's profile timeline** — just that one user's tweets (easy; mostly a single-key range scan, not the hard part).

### Explicitly out of scope (say this — it shows seniority and protects your time)

- DMs, notifications, search, trending/hashtags, ads, who-to-follow recommendations.
- Media *storage/transcoding* — I'll treat media as an opaque URL served from a CDN and not design the blob pipeline (*see CDN block*).
- Abuse/spam/moderation.

> **Say this in the room:** "I'll focus on **post tweet, follow, and home-timeline read**. The
> home-timeline read is the interesting part — that's where I'll spend my deep-dive time."

### Non-functional (where staff candidates separate themselves)

- **Read-heavy, hard.** Users scroll far more than they post. I'll assume a **~100:1 read:write ratio** (consistent with the framework's "most consumer apps" rule of thumb). This single number shapes the *entire* design: it justifies precomputing feeds and caching aggressively.
- **Latency:** home-timeline read **p99 < 200 ms**. This is the product. A slow feed loses users.
- **Availability over consistency:** the feed must always load. 99.99% on read path.
- **Consistency:** **eventual is fine for the feed.** If my tweet shows up in your feed 2–5 seconds late, nobody notices or cares. (I'll defend this explicitly in the deep dive — it's a load-bearing decision, not a shrug.) The *one* exception is read-your-own-writes: I should see my *own* tweet immediately.
- **Durability:** a posted tweet must never be lost once acknowledged. The tweet *itself* is the source of truth; derived feeds are disposable and rebuildable.

> **Say this in the room:** "The key non-functional facts are: it's ~100:1 read-heavy, feed reads
> need sub-200ms p99, and eventual consistency is acceptable for the feed but the source-of-truth
> tweet must be durable. Everything I design follows from those."

---

## Step 2 — Estimation (3 min)

> **Say this in the room:** "I'm not chasing precision — I want numbers that *justify* later
> decisions, especially whether I can precompute feeds and whether one DB survives."

Assume a Twitter-scale service.

### Users and traffic

| Quantity | Assumption | Result |
|---|---|---|
| DAU | given | **200 M** |
| Tweets per user per day | avg | **0.5** (most users lurk) |
| Tweets/day | 200M × 0.5 | **100 M/day** |
| **Avg write QPS** | 100M ÷ 86,400 | **~1,150 tweets/s** |
| **Peak write QPS** | 3× avg | **~3,500 tweets/s** |
| Feed opens/refreshes per user per day | avg | **~25** |
| Feed reads/day | 200M × 25 | **5 B/day** |
| **Avg read QPS** | 5B ÷ 86,400 | **~58,000 reads/s** |
| **Peak read QPS** | 3× avg | **~175,000 reads/s** |

Read:write ≈ 58k : 1.15k ≈ **~50:1 at the QPS level** (higher per-byte because each feed read returns many tweets). Confirms: **read-heavy** → optimize the read path even at the cost of more write-side work. This is the green light for **fan-out on write** (precompute the feed).

### Fan-out volume (the number that drives the whole design)

This is the estimate that matters most.

| Quantity | Assumption | Result |
|---|---|---|
| Avg followers per user | — | **~200** |
| Fan-out writes per tweet | = followers | ~200 |
| **Avg fan-out writes/sec** | 1,150 tweets/s × 200 | **~230,000 writes/s** |
| Peak fan-out writes/sec | 3× | **~690,000 writes/s** |
| Celebrity followers | top accounts | **10M–100M+** |
| Fan-out for ONE celebrity tweet | 50M followers | **50,000,000 writes** |

> **Say this in the room:** "Here's the punchline of the estimate: average fan-out is ~230k
> timeline-inserts/sec, which a fleet of workers + Redis can handle. But a *single* celebrity tweet
> is 50 **million** inserts. That asymmetry — 200 vs 50,000,000 — is exactly why a pure
> fan-out-on-write design breaks, and why I'll land on a **hybrid**. The estimate just told me the
> answer to the hardest design question before I drew a box."

### Storage / year

| Item | Math | Result |
|---|---|---|
| Bytes per tweet (text + metadata, no media) | ~300 B text + IDs/timestamps | **~1 KB** |
| Tweets/year | 100M/day × 365 | **~36.5 B/yr** |
| **Tweet store / year** | 36.5B × 1 KB | **~37 TB/yr** |
| Media (refs only here; blobs on CDN) | — | excluded |
| **Timeline/feed store** | derived, capped (see below) | see deep dive |

> **Say this in the room:** "Tweets are ~37 TB/yr and growing — comfortably beyond one box, so the
> tweet store gets **sharded** (*see sharding block*). The derived feed store is bigger in IOPS
> than in bytes, because I'll cap each user's materialized timeline at ~800 entries — I never need
> page 50 of someone's feed."

---

## Step 3 — API Design (3 min)

REST over HTTPS, JSON. Auth + rate-limiting terminate at the **API gateway** (*see rate limiter / LB block*) so I don't re-explain them per endpoint. `userId` comes from the auth token, never the body.

```
POST /v1/tweets
  Authorization: Bearer <token>
  body: { text: string(<=280), mediaIds?: [string] }
  -> 201 { tweetId, createdAt }

GET  /v1/feed?limit=20&cursor=<opaque>
  Authorization: Bearer <token>
  -> 200 { tweets: [ {tweetId, authorId, text, mediaUrls, createdAt, stats} ],
           nextCursor: <opaque|null> }

POST   /v1/users/{targetId}/follow     -> 204
DELETE /v1/users/{targetId}/follow     -> 204

GET  /v1/users/{userId}/tweets?limit=&cursor=   # profile timeline
  -> 200 { tweets: [...], nextCursor }
```

### Key API choices (call these out)

- **Cursor-based pagination, not offset.** A feed has tweets inserted at the head constantly; `OFFSET 40` would skip or duplicate rows as the feed shifts. The cursor encodes "where I was" — typically `(timestamp, tweetId)` or a snapshotted feed position — so paging is stable. *This is the framework's explicit rule for feeds.*
- **Opaque cursor.** I return it as a base64 token, not raw offsets, so I can change the underlying pagination mechanism (DB cursor vs Redis index) without breaking clients.
- **`POST /tweets` returns fast.** It writes the source-of-truth tweet synchronously and **enqueues** fan-out asynchronously, then returns `201`. The user is not blocked on 200 fan-out writes (let alone 50M). *(Sync vs async tradeoff.)*
- **Idempotency:** `POST /tweets` accepts an optional `Idempotency-Key` header so a client retry after a timeout doesn't double-post (*see idempotency block*; relevant because the queue is at-least-once).

---

## Step 4 — Data Model (5 min)

> **Say this in the room:** "I pick stores from *access patterns*, not by reflex. Let me state how
> each entity is read, and the store falls out."

### Entities and access patterns

**`users`** — point lookups by `userId`. Low write volume, relational-ish. **SQL** (or any KV) — `userId → {handle, displayName, bio, ...}`.

**`tweets`** — write-once, read by `tweetId` (point) and by `authorId` (range, for profile timeline). Massive volume (37 TB/yr), no cross-entity transactions needed. → **Sharded by `tweetId`**, with a secondary access path by author.

```
tweets
  PK: tweetId            # Snowflake-style: [timestamp | shardId | seq] -> sortable, time-encoded
  authorId
  text
  mediaIds
  createdAt
  -- counters (likes/retweets) kept separately (hot, high-churn) --
```

> **Say this in the room:** "I'll generate `tweetId` as a **Snowflake-style ID**: a 64-bit,
> roughly time-sortable ID with an embedded timestamp. That gives me globally unique IDs without a
> central counter, *and* the ID itself encodes creation time, so feed merges can sort by ID. Small
> choice, big payoff downstream."

**`follows`** — the social graph. Two access patterns, so I store it **both ways** (denormalized):
- "who does X follow?" → needed at read time for fan-out-on-read (celebrities).
- "who follows X?" → needed at write time for fan-out-on-write.

```
follows_by_follower:  followerId -> [followeeId...]   # X's following list
follows_by_followee:  followeeId -> [followerId...]   # X's follower list (the fan-out target set)
```

This is wide-row / KV shaped (a celebrity's follower list is 50M entries — you never load it whole; you **page** it). → **Cassandra / wide-column** (*see LSM-tree block*; write-heavy, append-friendly).

**`timelines` (the materialized home feed)** — this is the derived, precomputed feed. Read by `userId`, returns a *capped* reverse-chronological list of `tweetId`s. This is the read hot path.

```
home_timeline:  userId -> [ (tweetId, score/timestamp), ... ]   # capped at ~800
```

- Stored in **Redis** as a sorted set (`ZSET`) per user, scored by timestamp/rank — `O(log n)` insert, `O(log n + k)` range read for a page (*see caching deep dive*).
- Backed by a durable copy in the wide-column store so it survives a cache flush.
- **I store tweet *IDs* (and a score), not tweet bodies**, in the timeline. The body lives once in the `tweets` store and is hydrated at read time from a tweet cache. This keeps the (heavily duplicated, 200×) timeline rows tiny and means an edited/deleted tweet has one source of truth.

> **Say this in the room:** "Critical denormalization decision: the home timeline stores tweet
> **IDs**, not tweet **content**. If I copied the full tweet into 200 followers' timelines I'd 200×
> my storage and have 200 stale copies to fix on edit/delete. IDs + a tweet-body cache gives me the
> read speed of denormalization without the duplication tax."

**`counters` (likes, retweets, replies)** — extremely hot, high write churn, approximate is fine. Kept out of the tweet row; sharded counters / Redis, reconciled async. Mentioned for completeness; not a deep dive here.

---

## Step 5 — High-Level Design (10 min)

> **Say this in the room:** "Let me get the happy path end-to-end with the simplest thing that
> works, then evolve it under your questions."

```
                         ┌────────────────────────────────────────────────┐
                         │                  API Gateway                    │
   Mobile / Web ───────► │  TLS · auth · rate-limit · routing (L7 LB)      │
                         └───────┬──────────────────────────────┬─────────┘
                                 │ writes                        │ reads
                                 ▼                               ▼
                        ┌─────────────────┐             ┌─────────────────┐
                        │  Tweet Service  │             │  Feed Service   │
                        └───┬─────────┬───┘             └───┬─────────────┘
              write tweet   │         │ publish event       │ read timeline
                            ▼         ▼                      ▼
                   ┌──────────────┐  ┌──────────────┐   ┌──────────────────┐
                   │ Tweet Store  │  │  Kafka topic │   │  Redis ZSET      │
                   │ (sharded)    │  │ "tweet.posted"│  │ home_timeline:u  │
                   └──────────────┘  └──────┬───────┘   └────────┬─────────┘
                                            │                    │ IDs
                                            ▼                    ▼ hydrate
                                   ┌──────────────────┐   ┌──────────────┐
                                   │ Fan-out Workers  │   │ Tweet Cache  │
                                   │ (consumers)      │──►│  (bodies)    │
                                   └────────┬─────────┘   └──────────────┘
                                            │ ZADD into each follower's timeline
                                            ▼
                                   ┌──────────────────┐
                                   │ follows_by_followee (wide-column) │
                                   └──────────────────┘
```

### Happy path — posting a tweet (write)

1. Client `POST /v1/tweets` → gateway authenticates, rate-limits, routes to **Tweet Service**.
2. Tweet Service mints a Snowflake `tweetId`, writes the tweet **synchronously** to the sharded **Tweet Store** (durable — this is the ack-worthy step), and writes-through to the **Tweet Cache**.
3. Tweet Service publishes a `tweet.posted {tweetId, authorId, createdAt}` event to **Kafka** and returns `201` to the client. **The user's request is now done** — total latency is one DB write, not the fan-out.
4. **Fan-out workers** consume `tweet.posted`. For a normal author they page `follows_by_followee[authorId]` and `ZADD (tweetId, score)` into each follower's `home_timeline:<followerId>` Redis ZSET, trimming to the ~800 cap. For a celebrity author, they **do nothing** (see deep dive).

### Happy path — reading the feed (read, the hot path)

1. Client `GET /v1/feed?cursor=` → gateway → **Feed Service**.
2. Feed Service `ZREVRANGE home_timeline:<userId>` for the requested page → a list of `tweetId`s + scores (the precomputed part, fast).
3. **Merge in celebrity tweets on read:** look up the small set of celebrities this user follows, pull their recent tweets, and merge them into the page by score (the pull part — see hybrid deep dive).
4. **Hydrate**: batch-fetch tweet bodies for those IDs from the **Tweet Cache** (fall back to Tweet Store on miss), attach counters, and return the page + `nextCursor`.

> **Say this in the room:** "Notice the read path is mostly a single Redis range read plus a batched
> hydrate — that's how I hit sub-200ms p99 at 175k QPS. The expensive work (fan-out) was shifted to
> write time and to async workers, which is exactly the trade a 50:1 read-heavy system wants."

---

## Step 6 — Deep Dives (15 min) — *where the round is won*

> **Say this in the room:** "The interesting challenge here is the feed fan-out and the celebrity
> problem. Can I go deep there? That's where the design lives or dies."

### 6.1 Fan-out on write vs on read vs **hybrid** — the core tradeoff

This is the **push vs pull** tradeoff from the toolkit, applied.

**Fan-out on write (push).** When a user tweets, immediately push (`ZADD`) the tweetId into every follower's materialized timeline.
- ✅ **Reads are dirt cheap** — the feed is already computed; a read is one range scan. Perfect for 50:1 read-heavy.
- ❌ **Writes amplify by follower count.** Average 200× is fine. But a celebrity at 50M followers = **50M writes for one tweet** — a write storm that (a) saturates workers and Redis, (b) delays delivery for everyone, and (c) wastes work on millions of inactive followers who'll never open the app.

**Fan-out on read (pull).** Store nothing precomputed. At read time, fetch the recent tweets of everyone the user follows and merge-sort them.
- ✅ **Writes are trivial** — just store the tweet once. No amplification, no celebrity storm.
- ❌ **Reads are expensive and slow.** A user following 1,000 people means 1,000 lookups + a merge-sort *on every feed open*, 175k times/sec. Kills the p99 budget. This is the opposite of what a read-heavy system wants.

| | Write cost | Read cost | Breaks on |
|---|---|---|---|
| **Push** (fan-out on write) | O(followers) — explodes for celebs | O(page) — cheap | celebrities, inactive followers |
| **Pull** (fan-out on read) | O(1) | O(followees) merge — slow | users who follow many; high read QPS |
| **Hybrid (chosen)** | O(followers) for normals, O(1) for celebs | cheap + small celeb merge | nothing catastrophic |

> **Say this in the room:** "Neither pure approach survives. Push dies on the celebrity write storm;
> pull dies on the read-QPS budget. The staff-level answer is a **hybrid**: fan-out on write for
> normal users, fan-out on read for the handful of celebrities each user follows. I optimize the
> common case (200 followers) with push and special-case the pathological tail (50M followers) with
> pull."

### 6.2 The celebrity / hot-user problem and the hybrid solution, in detail

Define a **celebrity** as any account above a follower threshold — say **100k+ followers** (tunable; could be the top ~10k accounts by follower count, maintained as a flag on the user row).

**On write:**
- Normal author → fan-out workers push to all followers' timelines (the precompute).
- Celebrity author → **skip fan-out entirely.** Their tweet just lives in the Tweet Store and a per-celebrity recent-tweets cache (`celeb_recent:<celebId>` — a small Redis list of their last ~50 tweets). No 50M-write storm.

**On read (Feed Service):**
1. `ZREVRANGE` the user's precomputed `home_timeline` (contains all their *normal* followees' tweets) → page of IDs.
2. Look up the **celebrities this user follows** — a small list (most people follow a handful of celebs), cached per user.
3. Fetch each followed celebrity's recent tweets from `celeb_recent:<celebId>` (already cached, tiny).
4. **Merge-sort** the precomputed page with the celebrity tweets by score, take top-`k`, hydrate, return.

The pull cost is bounded: a user following 20 celebrities does 20 small cache reads + a merge of a few hundred items — cheap and predictable. We pulled *only* the expensive accounts; everything else was pushed.

> **Say this in the room:** "The asymmetry is the whole game. 99.9% of accounts get push because
> 200 writes is nothing. The 0.1% celebrity accounts get pull because 50M writes is catastrophic
> and most of those followers are inactive anyway. The hybrid is just 'apply the cheaper strategy
> per-account based on follower count.' I'd make the threshold a config knob and even allow
> per-account overrides."

**Edge case — crossing the threshold.** When an account grows past the celebrity line, I stop fan-out-on-write going forward and switch them to pull; existing timeline entries age out under the ~800 cap naturally. No backfill needed.

### 6.3 Timeline storage + caching: precompute vs on-read merge (*see caching deep dive*)

- **Where:** per-user **Redis sorted set** `home_timeline:<userId>`, scored by tweet timestamp (or by rank score, see 6.5). `ZADD` to insert, `ZREVRANGE start stop` to page, `ZREMRANGEBYRANK` to trim to ~800.
- **Why capped at ~800:** nobody scrolls past a few hundred items. Capping bounds memory and write cost. If a user pages past the cap (rare), fall through to a slower fan-out-on-read against their followees from the Tweet Store — graceful degradation.
- **Why IDs not bodies:** (from Step 4) the ZSET holds `(tweetId, score)`; bodies are hydrated from the **Tweet Cache** at read time. One source of truth, 200× less duplicated storage, correct on edit/delete.
- **Durability of the derived feed:** Redis is the hot copy; a periodically-snapshotted copy lives in the wide-column store. If Redis loses a user's timeline, it's **rebuildable** from `follows_by_follower` + recent tweets — the feed is *derived state*, never the source of truth. This is what lets me run the cache aggressively.
- **Inactive users:** don't materialize timelines for users who haven't opened the app in N days. Fan-out workers check a "last active" bit and skip them; their feed is built on first return (pull). Massive savings — the long tail of dormant accounts is huge.

> **Say this in the room:** "I precompute (push) because reads are 50× writes — I'd rather pay once
> at write time than every read. But I keep the timeline as disposable derived state in Redis, cap
> it, and skip inactive users. Precompute for the hot read path, rebuildable so I can treat the
> cache as a cache."

### 6.4 Thundering herd / hot tweet

Two distinct hot-spot problems:

**(a) Hot key on read (a viral tweet's body).** When one tweet goes viral, millions of feed hydrations hit the *same* `tweet:<id>` cache key. If that key expires, all of them stampede the Tweet Store simultaneously (*thundering herd*, called out in the caching block).
- **Mitigations:** (1) **request coalescing / single-flight** — only one miss recomputes, others wait on the result; (2) **don't expire hot keys hard** — use logical/probabilistic early expiry so one request refreshes ahead of TTL; (3) replicate the hottest keys across Redis replicas / use a small in-process LRU on the Feed Service to absorb the very hottest items at the edge.

**(b) Write storm on a celebrity post.** Already solved by 6.2 — celebrities don't fan out, so there's no 50M-write thundering herd at write time.

**(c) Backpressure on the queue.** If fan-out lag spikes (a flood of posts), Kafka **absorbs the spike** (that's the queue's job — *see message-queue block*) and workers drain at their rate. Feed freshness degrades by seconds; nothing falls over. I'd autoscale workers on consumer lag.

> **Say this in the room:** "Hot-tweet read stampede is the classic thundering herd. I coalesce
> misses so a viral tweet causes *one* DB read, not a million, and I refresh hot keys before they
> expire. The write-side storm I already designed away by not fanning out celebrities."

### 6.5 Ranking: chronological vs ranked feed (high level)

- **v1: reverse-chronological.** Score = timestamp. Simple, predictable, easy to merge across push+pull. Ship this first.
- **v2: ranked feed.** Score = a relevance model (recency × affinity × predicted engagement). Architecturally this is a **scoring layer** that runs at read time over the candidate set (the precomputed timeline + pulled celebrity tweets) before the top-`k` cut. The fan-out machinery is unchanged — ranking is just *how I compute the ZSET score / re-rank the candidate page*, not a different data flow.
- Tradeoff: ranked feeds boost engagement but add a model-serving dependency on the read hot path (latency + a new failure mode). I'd compute features async, keep scoring cheap (a lightweight model or precomputed feature vectors), and **fall back to chronological** if the ranker is slow or down — never block the feed on the model.

> **Say this in the room:** "I'd ship chronological first and treat ranking as a pluggable scoring
> step over the same candidate set, with a chronological fallback. The hard part of this system is
> the fan-out, not the ranker — I won't let ranking become a SPOF on the read path."

### 6.6 Consistency: is eventual OK for the feed? (yes — and here's why)

**Yes, eventual consistency is the right choice for the home timeline**, and naming *why* is the senior signal (*CAP/PACELC, strong-vs-eventual blocks*).

- **The feed is not a correctness-critical view.** If your tweet lands in my feed 2–5 seconds late, there's no invariant violated, no money lost, no user-visible wrongness. Contrast an account balance, where stale = double-spend. PACELC: in the *normal* case I'm choosing **L over C** — lower latency, accept slight staleness — because that's what the product needs.
- **Availability beats freshness here.** Under a partition I'd rather serve a slightly stale feed than fail the read (choose **A over C**). A feed that always loads, occasionally a few seconds behind, is the correct product behavior.
- **The async fan-out path is inherently eventual** — Kafka + workers means delivery lag, and that's fine given the above.
- **The one exception: read-your-own-writes.** A user *must* see their own tweet immediately, or it feels broken. I get this cheaply without strengthening the whole system: after posting, the client optimistically prepends the tweet locally, and/or the Feed Service merges the author's *own* very-recent tweets (from `celeb_recent`-style self-cache) on read. So *global* consistency is eventual, but *self*-consistency is immediate.

> **Say this in the room:** "Eventual consistency is not a compromise here — it's the correct
> choice. The feed is derived, non-critical, and read-heavy, so I trade freshness for latency and
> availability per PACELC. The only place I force immediacy is read-your-own-writes, and I handle
> that at the edge rather than making the whole pipeline strong."

---

## Step 7 — Wrap-Up (3 min)

> **Say this in the room:** "Let me name what still hurts, how it fails, and what I'd do with more
> time — before you have to ask."

### Remaining bottlenecks
- **Redis timeline fleet** is the read hot path. Scaled by **sharding/partitioning by `userId`** (*see sharding & consistent-hashing blocks*) across a Redis cluster; hot keys handled per 6.4. Capacity is the main cost driver.
- **Fan-out worker throughput** at peak (~690k inserts/s). Horizontally scaled, autoscaled on Kafka consumer lag.
- **Hydration fan-in** — each feed read batches many tweet-body lookups; mitigated by the Tweet Cache and multi-get.

### Failure modes (and graceful degradation)
- **Fan-out queue (Kafka) dies / lags.** Source-of-truth tweets are already durably stored, so nothing is lost. Feeds simply stop updating until it recovers; on recovery, workers drain the backlog and timelines catch up. Worst case I rebuild affected timelines from the graph + Tweet Store. **Degrade, don't fail.**
- **Redis timeline cache cold / flushed.** Feed reads miss → fall back to **fan-out-on-read** from `follows_by_follower` + Tweet Store (slower, but the feed still loads), and re-warm the ZSET. The whole reason I kept the feed as rebuildable derived state.
- **Tweet Store shard down.** Affects tweets on that shard only; replication (leader-follower) provides failover (*see replication block*). Feed reads for unaffected tweets still serve.
- **Ranker down (v2).** Fall back to chronological. Never a SPOF on the read path.

### Single points of failure
- No single DB, queue, or cache node is a SPOF: Tweet Store and follows are **sharded + replicated**, Kafka is **partitioned + replicated**, Redis is **clustered + replicated**, services run N replicas behind L7 LBs with health checks (*see LB block*). The gateway/LB tier itself is multi-AZ.

### With more time, I'd add
- **Search & trending** (inverted index / Elasticsearch fed via CDC from the tweet stream — *see inverted-index block*).
- **Ranked feed v2** with a real model-serving layer + feature store.
- **Notifications/DMs** as separate fan-out problems reusing this queue.
- **Multi-region** active-active with geo-sharded timelines and async cross-region replication; per-region read locality for the latency budget.
- **Media pipeline** (upload → transcode → CDN) behind the `mediaIds` references.

---

## What made this a staff-level answer

- **The estimate drove the architecture, not vibes.** The "200 vs 50,000,000 fan-out" number *derived* the hybrid before any box was drawn. Estimation as a decision tool, not a ritual.
- **Named the core tradeoff and picked a side with justification.** Push vs pull → hybrid, with explicit costs on each axis and a tunable threshold — the framework's stated staff-level signal.
- **Treated derived state as derived.** Timeline = disposable, capped, rebuildable Redis state; tweet = durable source of truth. That single distinction unlocked aggressive caching, cheap failure recovery, and the eventual-consistency argument.
- **Defended eventual consistency with PACELC reasoning**, and still handled read-your-own-writes — showing the consistency model was *chosen*, not defaulted.
- **Named failure modes and graceful degradation before being asked** (queue dies, cache cold, ranker down) and showed every one degrades rather than fails.
- **Scoped hard up front and protected the time budget**, spending the deep-dive minutes on the genuine bottleneck (fan-out / celebrity) instead of spreading thin.

> **One-line takeaway:** *A 50:1 read-heavy feed → precompute (push) the common case, pull the
> pathological tail (celebrities), keep the feed as rebuildable derived state, and accept eventual
> consistency everywhere except read-your-own-writes.*
