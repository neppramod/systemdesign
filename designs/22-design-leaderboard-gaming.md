# Design 22: Real-Time Leaderboard / Ranking System (Gaming)

> **Why this problem is worth a full walkthrough:** It looks tiny ("just sort scores") and that's
> the trap. The killer requirement — *give me the exact rank of one player among 100M, in real
> time, while scores update constantly* — is what separates a candidate who says "SQL `ORDER BY` +
> `COUNT`" from one who reaches for the right data structure, sees the O(log n) sorted set, and then
> knows what breaks when it outgrows a single node. The whole round lives or dies on two ideas:
> **(1) the Redis Sorted Set is the core primitive**, and **(2) global rank doesn't shard cleanly**.
> Everything else is plumbing around those two.

Cross-references: caching & Redis internals (Topic 06), sharding/partitioning (Topic 04 / framework Part B), the framework itself (Topic 01).

---

## Step 1 — Requirements (5 min)

I'll drive this and narrow scope out loud.

### Functional
- **Submit a score**: a player finishes a match / earns points → their score updates. Could be *set to value*, *increment by delta*, or *keep max*. I'll assume **cumulative score** (increment) as the default and call out where "keep best" differs.
- **Top-N leaderboard**: return the top 10 / 50 / 100 players, with score, for a board.
- **A specific player's rank**: "you are #4,213,902 of 102M." This is the hard one.
- **Neighbors around a player**: the ±5 players above and below me (the "your standing" widget). Cheap once we have rank.
- **Segmented leaderboards**: not just one global board — per *region*, per *game mode*, per *guild/friends*, per *level bracket*. Same machinery, many boards.
- **Time-windowed boards**: daily, weekly, monthly, all-time. Each is its own board with its own lifecycle (reset/expire).

### Out of scope (stated explicitly to protect time)
Matchmaking, the game servers themselves, score *generation* logic, auth (assume gateway handles it), and the social graph beyond "friends list exists." I'll keep an **anti-cheat hook** in the write path but not design the cheat-detection ML.

### Non-functional (this is where the design is actually decided)
- **Scale**: 100M registered players, ~10M DAU. Target tens of thousands of score updates/sec at peak, hundreds of thousands of leaderboard read QPS (everyone stares at the board).
- **Latency**: score update acknowledged < 100ms; top-N and "my rank" reads < 50ms p99. This is a game UI — it must feel instant.
- **Real-time**: "real-time" here means **seconds, not milliseconds of global consensus**. When I score, *my* view should reflect it immediately; other players seeing my new rank a second later is fine.
- **Read:write ratio**: heavily read-heavy for *views* (everyone reads top-N), but writes are not negligible during peak play. Call it ~10:1 to ~50:1 reads:writes depending on the title.
- **Consistency**: **eventual is acceptable** for rank display. Two players can briefly disagree on the exact ordering of positions 5,000,001 and 5,000,002. What must be *durable* is the score itself — losing a player's hard-earned points is unacceptable. So: **eventual consistency for ranking, strong durability for the score of record.** That split drives the whole architecture (serving layer vs source of truth).

> **The framing sentence I'd say:** "This is a read-heavy ranking problem where the expensive
> operation is *rank-of-one-element-in-a-sorted-set*. A relational `ORDER BY ... COUNT` is O(n) per
> rank query and collapses at 100M rows × high QPS. The entire design is: pick a structure that does
> rank in O(log n), make it durable, and figure out what happens when it doesn't fit on one box."

---

## Step 2 — Estimations (3 min)

Numbers exist to justify decisions, not to impress.

### Players & updates

| Quantity | Assumption | Result |
|---|---|---|
| Registered players | given | 100M |
| DAU | 10% of registered | 10M |
| Score updates / active player / day | ~20 matches | 200M updates/day |
| Avg update QPS | 200M ÷ 86,400 | **~2,300 writes/sec** |
| Peak update QPS | 3× avg | **~7,000 writes/sec** |

### Read QPS (leaderboard views)

| Quantity | Assumption | Result |
|---|---|---|
| Leaderboard views / DAU / day | ~30 (check board often) | 300M reads/day |
| Avg read QPS | 300M ÷ 86,400 | **~3,500 reads/sec** |
| Peak read QPS | 3× | **~10,000 reads/sec** |

So we're at roughly **10k read QPS / 7k write QPS at peak**. That is comfortably within a single Redis node's envelope (a single node does 100k+ simple ops/sec). **Conclusion: one Redis node can serve the hot global board.** Sharding is *not* yet justified by QPS — it becomes justified by **memory** and **per-board fan-out**, which I'll show next. This is the "derive sharding, don't assume it" move.

### Memory for a sorted set

A Redis sorted set entry = member (player ID) + score, plus the skiplist + hash-table overhead. Rule of thumb ≈ **~80–100 bytes per member** for a small string member.

| Board | Members | Memory |
|---|---|---|
| All-time global | 100M | 100M × ~100B ≈ **~10 GB** |
| Daily global | ~10M (DAU) | ~1 GB |
| Per-region (×~10) | ~10M total | ~1 GB total |
| Weekly | ~30M | ~3 GB |

A single all-time global board at **~10 GB** fits on one node but is chunky, and once we add *many* segmented + windowed boards the aggregate pushes past a single node's RAM and past the blast-radius we want. **That** is what justifies sharding/cluster — memory and isolation, not raw QPS.

### Bandwidth
Top-100 response ≈ 100 × ~50B = ~5 KB. At 10k read QPS → ~50 MB/s. Trivial. The "my rank + neighbors" response is tiny. Bandwidth is not a constraint here; **CPU of rank computation and memory are.**

---

## Step 3 — API Design (3 min)

```
POST /v1/scores
  { gameId, playerId, boardId, delta | value, ts, nonce }
  -> { newScore, rank? }            # rank optional/best-effort on write

GET  /v1/leaderboards/{boardId}/top?limit=100&offset=0
  -> { entries: [{ playerId, score, rank }], asOf }

GET  /v1/leaderboards/{boardId}/players/{playerId}/rank
  -> { rank, score, percentile }

GET  /v1/leaderboards/{boardId}/players/{playerId}/neighbors?radius=5
  -> { entries: [{ playerId, score, rank }] }    # window centered on the player
```

Notes I'd say aloud:
- `boardId` encodes the **segment + window**, e.g. `global:all-time`, `region:eu:weekly:2026-W25`, `mode:ranked:daily:2026-06-20`. This is the partition handle.
- The score POST carries a **`nonce`/idempotency key** so retries don't double-increment (writes go through a queue → at-least-once → must be idempotent). It also carries `ts` for **tie-breaking** and as an **anti-cheat** signal.
- Reads are **point or small-range** by design; no offset-deep pagination into the millions (offset 5,000,000 is an O(n) skip — I'd disallow it and force "give me a rank window" instead).
- Auth + rate-limit at the gateway; mentioned once, not re-explained.

---

## Step 4 — Data Model (5 min)

Two stores, two jobs. **Access pattern picks the store**, per the framework.

### Source of truth — durable score store (the "books")
A row per (player, board) holding the authoritative score and an audit trail of score events.

```
score_events (append-only / WAL of truth)
  event_id (PK)      -- = idempotency nonce
  board_id, player_id
  delta, ts, source, validated
  PK: event_id   (dedupe)   index: (board_id, player_id)

player_score (materialized)
  board_id, player_id  (PK composite)
  score, updated_at
```

- This lives in a durable, horizontally-scalable store. Access pattern = **point write + point read by (board, player)**, append-heavy on events → I'd pick **Cassandra / DynamoDB** (write-optimized LSM, partition by `board_id` or `(board_id, hash(player_id))`). No need for relational joins here, so SQL's main strength is unused. If the org is already on Postgres and volume is modest, Postgres is fine too — but it is *not* where ranks are computed.
- **Why not compute rank here?** "Rank" = `SELECT COUNT(*) FROM player_score WHERE board_id=? AND score > ?`. That's O(n) per query even with an index (you still scan/count the matching range), and it fights with constant writes. At 100M rows × 10k QPS it's a non-starter. Covered in the deep dive.

### Serving layer — Redis Sorted Sets (the "fast index")
One sorted set per board: `ZSET key = boardId`, member = `playerId`, score = numeric score (with tie-break encoding, below). This is the structure that makes top-N and rank O(log n). Detailed next.

> **The data-model headline:** durable store = correctness & recovery; sorted set = speed. The
> sorted set is a *derived, rebuildable index* over the source of truth. If Redis evaporates, no
> data is lost — only warm state, which we rebuild.

---

## Step 5 — High-Level Design (10 min)

Happy path, end to end.

```
                              ┌──────────────────────────┐
   Client ──score──> API GW ──> Score Ingest Service      │
   (game)            (authn,    │  - validate / anti-cheat│
                      rate)     │  - idempotency check    │
                                │  - write event (durable)│──> Durable Store
                                └──────────┬───────────────┘    (Cassandra/Dynamo)
                                           │ enqueue
                                           v
                                    ┌────────────┐
                                    │   Kafka    │  (per-board partition)
                                    └─────┬──────┘
                                          v
                                 ┌─────────────────┐
                                 │ Leaderboard     │  ZADD / ZINCRBY
                                 │ Updater (worker)│───────────────> Redis Sorted Sets
                                 └─────────────────┘                 (serving layer)
                                                                          ^
   Client ──read top-N / my rank──> API GW ──> Leaderboard Read Svc ──────┘
                                                (ZREVRANGE / ZREVRANK)
```

**Write path:**
1. Client POSTs a score. Gateway authn + rate-limits.
2. **Score Ingest** validates (anti-cheat hook), checks idempotency (`event_id` seen? drop), and **writes the event to the durable store first** — this is the commit point; once it's durable we can ack the player.
3. It enqueues an update on **Kafka**, keyed by `boardId` (so all updates for a board are ordered within a partition and land on one updater → no cross-update races for that board).
4. **Leaderboard Updater** consumes and applies `ZINCRBY board playerId delta` (or `ZADD ... GT` for keep-best) to Redis. A player belonging to multiple boards (global + region + daily + weekly) fans out to several `ZADD`s here.
5. The player's own client can be told the new score synchronously (from step 2) and the new *rank* either synchronously (one extra `ZREVRANK` after the update, accepting a tiny race) or via a follow-up read.

**Read path:**
- **Top-N**: `ZREVRANGE board 0 N-1 WITHSCORES` → O(log n + N). Cached at the edge for a second or two (everyone wants the same top-100).
- **My rank**: `ZREVRANK board playerId` → O(log n). Returns 0-based rank; +1 for display.
- **Neighbors**: from rank `r`, `ZREVRANGE board r-5 r+5 WITHSCORES` → O(log n + window).

Simple, single-node-Redis design first. Now evolve it under questioning.

---

## Step 6 — Deep Dives (15 min)

I'd propose: "The interesting parts are (a) *why the sorted set is the right primitive and how it gives rank cheaply*, (b) *exact rank of one player among 100M and why global rank won't shard*, and (c) *durability + windowing*. Let me go in that order."

### 6.1 The core data structure — Redis Sorted Sets, in depth

A Redis sorted set is a **skiplist + hash map** working together:
- The **hash map** gives O(1) `member → score` lookups (so `ZSCORE` and "does this player exist" are constant).
- The **skiplist** keeps members ordered by score, giving **O(log n)** insert, delete, and *rank* operations, plus O(log n + k) range scans.

The operations that matter and their costs:

| Operation | Redis command | Cost | Use |
|---|---|---|---|
| Update score (increment) | `ZINCRBY key delta member` | O(log n) | every score event |
| Update score (keep-best) | `ZADD key GT score member` | O(log n) | high-score boards |
| Top-N | `ZREVRANGE key 0 N-1 WITHSCORES` | O(log n + N) | leaderboard view |
| Rank of a player | `ZREVRANK key member` | **O(log n)** | "you are #X" |
| Neighbors | `ZREVRANGE key r-w r+w` | O(log n + w) | "your standing" |
| Count in score range | `ZCOUNT key min max` | O(log n) | percentile / approx rank |
| Drop a player | `ZREM key member` | O(log n) | bans, decay |

The crucial insight: **the skiplist nodes carry span counters**, so `ZREVRANK` walks down the skiplist accumulating how many elements it skipped — it computes rank **without scanning the whole set**. That is the entire trick. Rank of element #50,000,000 costs ~log₂(100M) ≈ **27 hops**, not 50M comparisons.

**Why the naive SQL approach collapses:**
`SELECT COUNT(*) FROM scores WHERE board=? AND score > :s` to get a rank is **O(n)** in the number of rows above the player — even with a B-tree index on `score`, the engine still counts every qualifying row (it can range-scan the index, but it can't read a precomputed rank). For a top player that's near-free; for a mid-pack player it counts ~50M index entries *per rank query*. Multiply by ~10k QPS and concurrent writes invalidating the index, and the DB melts. SQL has no "rank of this row" in sub-linear time without extra machinery (which is exactly what the skiplist *is*). This is the sentence that wins the round.

> Cross-ref Topic 06: the sorted set is also why Redis is the right *cache-shaped* serving layer
> here — single-threaded command execution means each `ZINCRBY`/`ZREVRANK` is atomic, no locking,
> and the hot board lives entirely in RAM.

**Tie-breaking.** Two players with score 5,000 — who's ranked higher? Usually "whoever reached it first." A float score alone can't express that. Two clean techniques:
1. **Composite encoding into the score**: pack `score` into the high bits and an inverted timestamp into the low bits of a 64-bit value: `encoded = score * 2^41 - (ts_ms)`. Earlier achiever (smaller ts) → larger encoded value → ranks higher. Redis stores scores as IEEE-754 doubles (53 bits of integer precision), so be careful with bit budget; if range exceeds 53 bits, store the score in the ZSET and break ties at read time, or use a lexicographic member trick.
2. **Lexicographic member ordering** with equal scores via `ZRANGEBYLEX` — works only when all tied members share a score; less general. I'd default to the composite-score encoding and *state the precision caveat*, which is the staff-level detail.

### 6.2 Exact rank of one player among 100M — and why global rank won't shard

On a **single** Redis node, exact rank is solved: `ZREVRANK` is O(log n). The problem appears the moment the board doesn't fit / we shard it.

**Why we'd shard at all (recap):** aggregate memory across all boards exceeds one node, and we want blast-radius isolation — a runaway region board shouldn't evict the global board. So we partition.

**Two ways to partition a leaderboard:**

1. **Partition by segment** (the easy, common case). Each *board* (region, mode, window) is an independent sorted set; route by `boardId` (consistent hashing across a Redis Cluster). Rank *within a segment* is still a single O(log n) `ZREVRANK` on one node. **No cross-shard problem** — because the question "my rank in EU-weekly" only touches the EU-weekly shard. **This handles the vast majority of real product needs**, and I'd lead with it: most "global" boards players care about are actually segmented.

2. **Partition one logical board by score range** (when a *single* board is too big for one node). Shard 0 holds scores [0, 1k), shard 1 holds [1k, 5k), etc. Now:
   - **Top-N**: only touch the top shard(s) → cheap.
   - **Global rank of player P with score s**: rank = (players above s in P's own shard) + (total count of every higher shard). So `ZREVRANK` on P's shard **plus** a `ZCARD` (cached) on each higher shard. With R range-shards that's R lookups but most are cheap cached counts → bounded and small (R is single digits). This is the **scatter-gather + merge** pattern, *bounded* because counts-per-shard are precomputable and the player only needs an exact local rank plus higher-shard totals.

   The pain: **score ranges must be rebalanced** as the distribution shifts (everyone climbs over a season), and a hot range (the modal score band) becomes a hot shard. I'd keep shard boundaries adaptive (periodically recompute percentile boundaries) — analogous to range-shard splitting in HBase.

**Why hash-partitioning a single board is the *wrong* move:** if you hash players across N shards, computing global rank means asking every shard "how many of yours are above score s?" and summing — a full **scatter-gather across all N shards on every rank query**, with no shard able to short-circuit. That's the cross-shard global-rank problem in its worst form. Range-partitioning is strictly better here because the score *is* the partition dimension you're ranking on, so most shards answer with a cached count.

**The pragmatic escape — approximate rank at the tail.** Exact rank only truly matters near the top (leaderboard prestige) and around *you*. For "you are #4,213,902 of 102M," nobody can tell 4,213,902 from 4,210,000. So:
- Keep an **exact** sorted set for the **top K** (say top 10k–100k) — small, fits one node, exact `ZREVRANK`.
- For everyone below, compute **approximate rank via score histogram / percentile buckets**: maintain a coarse `ZSET` or count-min-style histogram of `score → number of players at-or-above`. Player's approx rank = cumulative count above their score bucket. This is O(buckets) and needs no giant sorted set. Display "Top 4%" or "#~4.2M".
- This collapses the 100M-member memory and the cross-shard cost: the *exact* structure is tiny, the *approximate* structure is a few KB of bucket counts updated in aggregate.

> **The tradeoff sentence:** "Exact global rank for 100M players is expensive — O(log n) on a huge
> set, and it doesn't shard without scatter-gather. I'd serve **exact rank for the top K and for a
> player's local neighborhood**, and **approximate (percentile-bucketed) rank for the long tail**,
> because nobody perceives the difference at rank 4 million and it makes the whole thing fit and
> shard cleanly. I accept bounded inaccuracy in the tail to buy O(1)-ish tail-rank and a tiny
> memory footprint."

### 6.3 Persistence, durability, and rebuild

Redis is the **serving layer, not the source of truth.** The durable store (6.x / Step 4) holds authoritative scores.

- **Write order**: durable write **then** ZSET update (write event → ack → async apply to Redis). If we updated Redis first and the durable write failed, we'd have a phantom rank with no backing record. Durable-first means Redis can always be reconstructed.
- **Redis durability config**: enable **AOF (append-only file) with `everysec` fsync** + RDB snapshots, so a single-node restart recovers most state without a full rebuild. But AOF/RDB are an *optimization*, not the guarantee — the guarantee is the durable store.
- **Rebuild on failure**: if a Redis shard dies and replicas are gone, rebuild its board(s) by streaming `player_score` for those `board_id`s from the durable store and bulk-`ZADD`ing. At ~10M members and pipelined ZADDs (~100k–1M ZADD/sec), a board rebuilds in seconds-to-low-minutes. During rebuild, reads can be served stale from a replica or degraded to approximate-only.
- **Replication**: each Redis shard is a primary with ≥1 replica (Redis Cluster / Sentinel); replica promoted on failure, then re-warmed from durable store if it was cold.

### 6.4 Time-windowed boards

Each window is **its own sorted set** with a key that encodes the window:
- `lb:global:daily:2026-06-20`, `lb:global:weekly:2026-W25`, `lb:global:alltime`.
- On a score event, the updater does `ZINCRBY` on *every* live window the player participates in (daily + weekly + monthly + all-time = a small fan-out per event). This is the main write-amplification cost; it's bounded (handful of windows) and accepted.
- **Expiry/reset = TTL.** Set `EXPIRE` on `daily` keys to ~48h (keep yesterday briefly for "final results"), `weekly` to ~2 weeks. **No manual reset job** — the new period's key simply doesn't exist yet and is created lazily on first write. This is clean and avoids a "reset at midnight" thundering operation.
- **Rolling windows** ("last 24h", not "calendar day") are harder: a true sliding window needs per-event timestamps and eviction of aged events, which a plain ZSET (scored by points) can't do. Approaches: (a) approximate with **N fixed sub-buckets** (e.g. 24 hourly ZSETs; "last 24h" = merge the trailing 24 via `ZUNIONSTORE`, recomputed periodically), trading exactness for simplicity; or (b) keep raw events and recompute. I'd default to **calendar-aligned windows** (much cheaper) and only build rolling windows if the product genuinely requires them, stating that tradeoff.

### 6.5 Hot updates / write throughput / batching

- **Per-board ordering via Kafka partition key = boardId** means each board's updates are serialized to one consumer → no lost-update races, and we can **batch**: drain a window of events per board and apply them as one pipelined transaction (`MULTI` / pipeline of `ZINCRBY`s), cutting round-trips. For increment semantics, batching can even **pre-aggregate**: sum deltas per player in the batch, issue one `ZINCRBY` per player per flush.
- **Hot player / hot board** (a streamer everyone's watching, or the single global board): reads are absorbed by **edge/CDN caching of top-N with a 1–2s TTL** and Redis read replicas; the hot *board* under heavy writes is mitigated by batching above. A genuinely hot single key in Redis is handled by replicating that board to multiple read replicas and load-balancing reads.
- **Backpressure**: the queue absorbs write spikes (match-end bursts); the updater consumes at a steady rate. The player's score is already durable and acked before the ZSET catches up — the lag manifests only as "leaderboard reflects my score a beat later," which our consistency requirement allows.

### 6.6 Anti-cheat / score validation hook

A first-class hook in **Score Ingest**, before the durable write:
- **Stateless checks**: score delta within physically-possible bounds for the game; `ts` monotonic and recent (reject far-future/replayed timestamps — also defends idempotency); rate per player within limits.
- **Stateful / async**: ship the event to a separate anti-cheat pipeline (Kafka topic) that flags anomalies; flagged scores can be quarantined (`validated=false`, excluded from the ZSET) and reversed via `ZREM`/`ZINCRBY -delta` since we have the event log. The event log being the source of truth is what makes **reversal/clawback** possible — another reason for durable-first.

---

## Step 7 — Wrap-Up (3 min)

**What's solid:** sorted set gives O(log n) updates and rank; durable store is the source of truth so Redis is fully rebuildable; segmented + windowed boards are just keyed sorted sets with TTL; the design meets the latency/scale targets with a single hot node and shards only when memory/blast-radius force it.

**Failure modes & SPOFs:**
- **Redis shard dies** → replica promoted (Sentinel/Cluster); if cold, rebuild from durable store in minutes; serve degraded/approximate or stale-from-replica meanwhile. *Not a data-loss event* because scores live in the durable store.
- **Updater lag / Kafka backlog** → leaderboard goes stale but scores stay durable and acked; UI shows "updating…". Bounded by consumer scaling.
- **Durable store down** → we stop acking writes (fail safe: never ack a score we can't persist) but can keep *serving* reads from Redis. Availability of reads preserved, writes paused — correct tradeoff.
- **SPOFs to remove**: single Kafka broker (replicate partitions), single gateway (LB + multiple), single updater per board (consumer group with standby; the partition key guarantees a single *active* consumer per board, with failover).

**Consistency stance:** eventual consistency for displayed rank is fine and intended; strong durability for the score of record is mandatory. PACELC-wise: under partition we favor **availability of reads** (serve possibly-stale ranks) over consistency; in normal operation we favor **latency** (serve from RAM/edge cache) over perfectly-fresh ranks.

**With more time:** real sliding-window boards via sub-bucket merges; per-region Redis placement for geo-latency; a write-back path to reconcile Redis ↔ durable counts (drift detection); richer anti-cheat; "friends leaderboard" via intersecting the friend set against the global ZSET (`ZINTERSTORE` against a small per-user set, or fetch friend scores individually since friend lists are small).

---

### What made this staff-level
- **Reached for the right primitive and justified it from complexity**: Redis Sorted Set's skiplist span counters give O(log n) rank — and I explained *why SQL `ORDER BY`+`COUNT` is O(n) and collapses*, rather than just asserting "use Redis."
- **Identified the genuinely hard problem** (exact global rank among 100M doesn't shard) and gave a layered answer: exact for top-K + neighborhood, **approximate percentile buckets** for the tail, range-partition (not hash) when a single board overflows, with a *bounded* scatter-gather.
- **Separated serving layer from source of truth** and made the serving layer fully rebuildable — durable-first write ordering, which also enables cheat clawback.
- **Derived sharding from memory/blast-radius, not reflex** (showed QPS fits one node first).
- **Named the precision caveat** in tie-break encoding (53-bit double mantissa) — a detail only someone who's actually used ZSETs raises.
- Every choice carried an explicit tradeoff sentence and a stated consistency stance (PACELC).

### Self-check (answer from memory)
- [ ] Why is `ZREVRANK` O(log n) and `SELECT COUNT(*) WHERE score > s` O(n)? (skiplist span counters vs row counting)
- [ ] When does this system actually need sharding — QPS, or memory/blast-radius?
- [ ] Why range-partition rather than hash-partition a single oversized board?
- [ ] How do you serve a player's rank among 100M cheaply? (exact top-K + approximate percentile tail)
- [ ] What is the write ordering and why? (durable store first, then ZSET — rebuildability + clawback)
- [ ] How are daily/weekly boards reset without a cron job? (per-window keys + TTL)
- [ ] How do you break ties deterministically, and what's the encoding pitfall? (score+inverted-ts, 53-bit mantissa limit)
