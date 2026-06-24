# Design 18: Ad Click Aggregator / Real-Time Analytics Pipeline

> **Why this one matters:** this is the canonical *streaming analytics* design, and it's the cleanest
> place to prove you understand the deepest split in data systems — **the same event stream feeds two
> consumers with opposite requirements.** Advertisers want a live dashboard that updates in seconds and
> is allowed to be a little wrong; the billing/finance pipeline wants a number that is *exactly right*
> and is allowed to be hours late. One stream, two truths. Reconciling that tension is the **Lambda vs
> Kappa** decision (**batch/stream doc 18**), and almost every interesting choice in this design falls
> out of it: at-least-once ingestion + dedup for idempotent counting (**idempotency doc 10**), tumbling
> windows with event-time + watermarks for late data (**doc 18**), approximate sketches (HLL, Count-Min)
> where exactness is wasteful (**doc 18**), and a columnar/OLAP serving layer for the dashboard
> (**data-modeling doc 20**, **databases doc 03**).
>
> The one-line thesis I state up front and keep returning to: **we ingest an at-least-once click
> firehose into a partitioned log, run a stream processor that emits fast, approximate, windowed
> rollups for dashboards, and run a batch layer that recomputes the exact numbers from the raw events
> for billing — the stream is the speed layer, the batch is the truth layer, and reconciliation is how
> the truth corrects the speed.** Every decision is "who is the consumer, and do they need *fast* or
> *correct*?" I'll run all 7 steps but spend most of the budget in **Step 6**, because that's where
> this round is won.

---

## Step 1 — Requirements (5 min) — I drive this

"Ad click aggregator" is unbounded, so I scope and position before drawing. The framing move is to
name the **two consumers of the same data** in the first sentence, because that fork drives the whole
architecture:

1. **Real-time dashboards** (advertisers, internal ops): "how many clicks has campaign X gotten in the
   last minute / hour?" Needs **low latency** (seconds), tolerates **approximate** answers, queries
   recent + historical aggregates by dimension.
2. **Billing & analytics** (finance, auditors): "exactly how many *valid* clicks did advertiser X get
   last month, so we charge them $Y?" Needs **exactness and auditability**, tolerates **latency**
   (minutes to hours), recomputable from source.

**Functional**
- Ingest **click and impression events** at massive scale: `(eventId, adId, campaignId, userId, ts,
  region, device, …)`.
- Serve **windowed aggregate queries**: counts/rates per dimension (ad, campaign, advertiser, region,
  device) over time buckets (per-minute, per-hour, per-day).
- Serve **top-K** queries ("top 10 ads by clicks this hour") and **unique-user** counts ("unique users
  who clicked campaign X today").
- Produce an **exact, auditable** per-advertiser billable count for invoicing.
- **Filter invalid/fraudulent clicks** (bots, double-clicks, click farms) before they count.

**Non-functional** — this is where the design is decided:
- **Write scale is the headline** — this is a write-firehose: ~**1M events/sec** peak (derived in Step
  2). The whole ingest path is shaped by absorbing that.
- **Dashboard latency** — aggregates visible within **~few seconds to ~1 min** of the click (p99
  query < 200 ms once data is in the serving store).
- **Billing accuracy** — the billable number must be **exact and reproducible**; an over-count means we
  overcharge advertisers (legal/financial risk), an under-count means lost revenue. This is the one
  place we **cannot** accept "approximately right."
- **Idempotent counting** — a single real click must be counted **exactly once** even though the
  transport is **at-least-once** (retries, replays, processor restarts all create duplicates).
- **Durability** — never lose a raw event; raw events are the **source of truth** the batch layer
  recomputes from. Losing them = unbillable revenue, no audit trail.
- **Consistency split** — dashboard aggregates are **eventually consistent + approximate** (we trade
  correctness for latency); billing aggregates are **strongly correct but delayed** (we trade latency
  for correctness). Naming this split *is* the design.
- **Availability** — ingestion must stay up under spikes (a viral campaign can 5× traffic); dropping
  click events directly loses revenue. The query side can degrade (stale dashboard) without losing data.

> **The staff move in Step 1:** I'm *positioning* before drawing. The thesis — "speed layer for
> dashboards, truth layer for billing, reconciliation between them" — makes every later choice
> derivable. Per **PACELC**: the dashboard path is **PA/EL** (under partition or lag, serve a stale/
> approximate number rather than block — availability and latency win). The billing path is **PC/EC**
> (we'd rather be late than wrong — consistency wins). The point is the *same data* sits on both sides
> of PACELC depending on the consumer. That's the senior framing, and it's the seed of Lambda vs Kappa.

---

## Step 2 — Estimations (3 min) — to justify, not impress

I'll pick numbers that *force* the architecture. Target: a large ad network.

- **Scale:** ~100M DAU, each sees ~100 ads/day and clicks ~1% → but impressions dominate. Let's anchor
  on **impressions**, the bigger firehose: 100M × 100 = **10B impressions/day**. Clicks at ~1% CTR →
  **100M clicks/day**. I'll provision the pipeline for the *impression* rate since it's 100× larger.

| Quantity | Formula | Result |
|---|---|---|
| Avg event rate | 10B impressions/day ÷ 86,400 s | **~115K events/s** |
| Peak event rate | avg × ~3 (diurnal + campaign spikes) | **~350K–1M events/s** → provision for **1M/s** |
| Event size | adId+campaignId+userId+ts+region+device, JSON-ish | **~200 bytes** |
| Ingest bandwidth | 1M/s × 200 B | **~200 MB/s** (≈1.7 GB/s peak with headroom) |
| Raw events/day | 10B × 200 B | **~2 TB/day** logical raw |
| Raw retention (90 days, for recompute/audit, compressed ~5×) | 2 TB × 90 ÷ 5 | **~36 TB** in the data lake |
| Rollup size (the win) | see cardinality math below | **~GBs/day, not TBs** |

**The rollup-vs-raw insight (the headline of the estimate):** the dashboard never queries 2 TB/day of
raw events — it queries **pre-aggregated rollups**. If we roll up to per-**minute** buckets across
dimensions {campaign, region, device}, a row is `(minute, campaignId, region, device, count)`.

- Cardinality: say 100K active campaigns × ~20 regions × ~10 device types = **20M** dimension combos,
  × 1,440 minutes/day = **~29B** *potential* rows/day — but most combos are empty each minute. Realistic
  **active** combos per minute are far fewer (sparse), so real rollup volume is **~GBs/day**.
- This is the **cardinality-explosion** warning I'll flag now and bound in Step 6: rollup cost is
  `time_buckets × Π(dimension_cardinalities)` if dense — you must **cap dimensions and bucket
  granularity** or the "small" aggregate table explodes past the raw data.

> What the math *justifies*:
> - **1M events/s, 1.7 GB/s peak** → no single DB or service absorbs this → **partitioned log (Kafka) +
>   horizontally-scaled stream processors** (ingestion is justified, not assumed).
> - **2 TB/day raw, 36 TB retained** → cheap, durable, append-only → **object-store data lake**, not a
>   hot DB.
> - **Rollups are GBs/day, queried by dimension+time-range** → **columnar/OLAP store**, where the
>   dashboard actually reads (justified the serving store).
> - The 100×-to-1000× gap between raw (2 TB/day) and rollups (GBs/day) is *exactly why* we pre-aggregate
>   in the stream instead of querying raw — and why approximate sketches (Step 6F) shrink it further.

---

## Step 3 — API design (3 min)

Two surfaces, matching the two consumers — deliberately split so the contract itself shows the speed/
truth divide.

```
# Ingestion (write firehose) — fronted by the ad-serving edge, not a public API
POST /events            { eventId, type: click|impression, adId, campaignId,
                          userId, ts (event-time), region, device, signals{…} }
   -> 202 Accepted      # fire-and-forget enqueue; eventId is the CLIENT-supplied idempotency key
   # batched: the edge ships thousands of events per request; never one HTTP call per click

# Dashboard query (speed layer) — reads rollups, approximate OK
GET /metrics?campaignId=&dimension=region&from=&to=&granularity=minute|hour|day
   -> [ { bucket, region, clicks, impressions, ctr } ]     # served from OLAP, seconds-fresh
GET /metrics/topk?metric=clicks&dimension=ad&window=1h&k=10
   -> [ { adId, count } ]                                   # Count-Min/heavy-hitter backed, approximate
GET /metrics/unique?campaignId=&window=1d
   -> { uniqueUsers }                                       # HyperLogLog estimate, ±~1-2% error

# Billing (truth layer) — reads reconciled exact aggregates
GET /billing/usage?advertiserId=&month=2026-05
   -> { validClicks, billableAmount, status: provisional|final }
   # provisional = stream estimate; final = batch-reconciled exact number
```

- **`eventId` is a client-supplied idempotency key** (**doc 10**) — it's what makes at-least-once
  delivery count-once. The ad-serving SDK generates it at click time and reuses it on every retry.
- Ingestion returns **202** and never blocks on aggregation — the firehose must absorb spikes (Step 6B).
- The billing endpoint explicitly exposes **provisional vs final**, surfacing the reconciliation
  contract in the API instead of hiding it.

---

## Step 4 — Data model (5 min)

Entities + **access patterns** — the access pattern picks the store, not reflex.

**Raw events (source of truth)** — append-only, immutable, never updated.
- Lives in the **log (Kafka)** for recent data (hours) and the **data lake** (object store, e.g. S3 +
  Parquet) for retained history.
- Partition/shard key: **`campaignId`** (or `adId`) — keeps all events for a campaign on the same
  partition, which we'll need for ordered, stateful windowed aggregation per campaign. Access pattern:
  *replay a time range for recompute*; *stream forward for the processor*.

**Stream rollups (speed-layer serving table)** — pre-aggregated, written by the stream processor.
```
RollupTable  (columnar / OLAP)
  PK/sort:  (campaignId, granularity, bucket_start, region, device)
  cols:     clicks, impressions, sum/derived
  access:   GET /metrics range-scan by campaignId + time range + dimension
```
- **Columnar/OLAP store** (Druid / ClickHouse / Pinot) because the dashboard does **range scans over
  time + group-by dimension + aggregate** — columnar storage reads only the needed columns and
  compresses repetitive dimension values hugely. A row store (or worse, the raw log) would be the wrong
  shape (**doc 20**: model to the access pattern).

**Sketch state** — per (campaign, window):
- **HyperLogLog** registers for unique-user estimates.
- **Count-Min Sketch** + a small top-K heap for heavy-hitter "top ads."
- Stored alongside rollups, merged across windows on query.

**Billing aggregates (truth-layer table)** — written by the **batch** layer.
```
BillingTable  (transactional / warehouse)
  PK:    (advertiserId, billing_period)
  cols:  valid_clicks, billable_amount, status, recomputed_at, source_offset_range
```
- Strongly consistent, append-/correct-only, audited. Written once per recompute pass.

**Dedup state** — keyed by `eventId`, used to drop duplicates (Step 6C). A bounded key-value store
(RocksDB in the processor, or Redis) with TTL, since dedup only needs a recent window.

> SQL vs NoSQL falls out of the access patterns, not reflex: raw events → **append-only log + object
> lake** (immutable, replayable); dashboard rollups → **columnar OLAP** (time-range + group-by);
> billing → **transactional warehouse** (exact, audited). Three stores because there are three distinct
> access patterns — using one store for all three would be the classic mistake.

---

## Step 5 — High-level design (10 min) — happy path end-to-end

```
                 click/impression
  Ad SDK / edge ───────────────► Ingestion API (202, batched)
   (eventId =                          │
    idempotency key)                   ▼
                            ┌──────────────────────┐
                            │   Kafka (the log)     │  partitioned by campaignId
                            │  topic: raw-events    │  RF=3, retain hours hot
                            └──────────┬────────────┘
                          ┌────────────┴──────────────┐
            (tee: same data, two consumers)            │
                          ▼                            ▼
       ┌──────────────────────────────┐    ┌──────────────────────────────┐
       │  SPEED LAYER (stream)         │    │  TRUTH LAYER (batch)          │
       │  Flink/Spark Streaming        │    │  raw events ──► Data Lake     │
       │  • fraud filter               │    │     (S3 + Parquet)            │
       │  • dedup by eventId           │    │  • periodic Spark/batch job   │
       │  • tumbling windows (event-   │    │  • recompute EXACT counts     │
       │    time + watermark)          │    │    from raw, dedup globally   │
       │  • rollups + HLL + Count-Min  │    │  • write billing table        │
       └──────────────┬────────────────┘    └──────────────┬────────────────┘
                      ▼                                      ▼
            ┌───────────────────┐                  ┌───────────────────┐
            │  OLAP store        │                  │  Billing warehouse │
            │  (Druid/ClickHouse)│◄── reconcile ────│  (exact, audited)  │
            └─────────┬──────────┘   (batch corrects │  status: final     │
                      │               the stream)    └─────────┬──────────┘
                      ▼                                         ▼
            Dashboard API (seconds-fresh, approx OK)    Billing/Invoice API (exact, delayed)
```

**Walk one event through it:** A user clicks → ad SDK generates `eventId`, fires `POST /events`
(batched with thousands of others) → ingestion service validates lightly and **appends to Kafka**
(partitioned by `campaignId`), returns 202. Two independent consumers read the *same* partition:

1. **Speed layer (stream processor):** drops invalid/fraud clicks, **dedups by `eventId`**, assigns each
   event to its **event-time tumbling window** (e.g., 1-minute), and incrementally updates per-window
   rollups + HLL + Count-Min. When the window's **watermark** advances past its end, it emits the
   rollup to the OLAP store → dashboard sees it within seconds.
2. **Truth layer (batch):** the raw events are also archived to the **data lake**. A scheduled batch
   job replays the lake, applies the *same* fraud + dedup logic globally and deterministically,
   computes **exact** counts, writes the **billing** table, and **reconciles** the OLAP store
   (overwrites the approximate stream numbers for finalized windows).

This is **Lambda architecture** — two layers, one fast/approximate, one slow/exact, merged at the
serving layer. I'll defend it (and contrast Kappa) in Step 6A. Keep it this simple first; evolve under
questioning.

---

## Step 6 — Deep dives (15 min) — where the round is won

I'll propose the hard part: *"The interesting tension is that two consumers need opposite things from
the same stream. That drives Lambda-vs-Kappa, exactly-once-ish counting, and windowing under late data.
Can I go deep there?"*

### 6A. Lambda vs Kappa — the central tradeoff (**doc 18**)

The two consumers force the question. Three options:

- **Pure stream (fast only):** dashboards are great, but the stream is approximate (dedup windows are
  bounded, late data may be dropped, a processor bug corrupts counts with no way to fix). **Billing on
  an approximate number is unacceptable** — we'd over/undercharge advertisers. Rejected for billing.
- **Pure batch (correct only):** billing is perfect, but dashboards are hours stale. Advertisers
  managing live campaigns need seconds. Rejected for dashboards.
- **Lambda (both layers):** stream layer serves fast/approximate for dashboards; batch layer recomputes
  exact for billing **and corrects the stream's historical numbers**. Cost: **two codebases** that must
  produce the same answer (the classic Lambda complaint — logic duplicated in Flink and Spark drifts).

**My choice: Lambda, but minimize the duplication.** I share the fraud/dedup/aggregation *logic* as a
library used by both layers, so the only difference is the runtime (streaming vs batch). This is the
pragmatic middle.

**Kappa as the alternative I name and weigh:** Kappa says "drop the batch layer — keep *only* the
stream, and to 'recompute,' just **replay the log from offset 0** through a new version of the stream
job." It eliminates the dual codebase. It works beautifully *if* your entire source of truth fits in
the log with long-enough retention and replay is cheap. Here, billing needs **90-day** auditable
history and deterministic recompute — I'd rather replay from a cheap **immutable Parquet lake** than
keep 90 days hot in Kafka. So I land on **Lambda with a lake-backed batch layer**, and tell the
interviewer I'd move toward Kappa-style (replay-driven) recompute if retention/replay economics allowed
— naming the side *and* the condition that would flip it is the staff signal.

> **Tradeoff stated:** Lambda buys correctness-with-freshness at the cost of two pipelines; Kappa buys
> one pipeline at the cost of needing the log to be your full, replayable source of truth. I pick
> Lambda because billing's 90-day audit window makes a cheap immutable lake the right truth store.

### 6B. Ingestion at write-firehose scale (**doc 07, doc 04**)

1M events/s, 1.7 GB/s peak. The ingestion service must do *almost nothing* per event:
- **Batch at the edge.** The ad SDK / edge collectors buffer and ship thousands of events per HTTP
  request — never one call per click. Cuts request overhead by ~1000×.
- **Append to Kafka and return 202.** No synchronous aggregation, no DB write on the hot path. Kafka
  **absorbs spikes** (a viral campaign 5×s traffic) as backpressure into the log, not dropped events —
  this is *the* reason a queue sits here (**doc 07**: a log decouples producers from consumers and
  smooths bursts).
- **Partition by `campaignId`.** Keeps a campaign's events co-located and ordered on one partition, so
  the stateful per-campaign windowed aggregation is local. Watch for a **hot partition** (one mega
  campaign during the Super Bowl): mitigate by **salting** the key (`campaignId:bucket`) to spread a hot
  campaign across N sub-partitions, then **merge** the N partial rollups at the serving layer. Naming
  the hot-key problem *and* the salt+merge fix is the senior move.
- **RF=3, `acks=all`** on the raw-events topic — an acked click must survive a broker loss, because raw
  events are the billing source of truth (we can lose a *dashboard refresh*, never a *billable event*).

### 6C. Exactly-once-ish counting: at-least-once + dedup (**doc 10**)

The transport is **at-least-once** (SDK retries on network failure, Kafka redelivers on consumer
restart, the batch job re-reads the lake). **A naive counter double-counts on every retry** —
`counter += 1` on a redelivered event over-bills the advertiser. This is the bug to call out by name.

**Fix: dedup on `eventId` (the idempotency key, doc 10).**
- **Stream layer:** keep a **dedup set** of recently-seen `eventId`s in the processor's local state
  (RocksDB) with a TTL covering the allowed lateness window. On each event: if `eventId` seen → drop;
  else count it and record it. This makes the *increment* idempotent → effectively-once within the
  window. To bound memory, the seen-set can be a **Bloom filter** for "definitely new vs maybe-seen"
  (false positives drop a few real clicks → acceptable for the *approximate* dashboard, never for
  billing).
- **Batch layer:** dedup is trivial and *exact* — `SELECT count(DISTINCT eventId) …` over the immutable
  lake. No TTL window, no Bloom approximation — it sees *all* history, which is exactly why **billing
  trusts the batch number, not the stream's.**

> **Why billing uses the corrected batch number:** the stream's dedup is bounded (TTL window) and may
> use a lossy Bloom filter, so a duplicate arriving *after* the window evicts its `eventId` would
> double-count. The batch layer dedups against the *full* history deterministically. So the stream is
> "exactly-once-ish" (good enough for a dashboard ±1%); billing waits for the batch's true distinct
> count. This is the concrete payoff of the speed/truth split.

### 6D. Stream aggregation: windows, event-time, watermarks, late data (**doc 18**)

The stream computes **tumbling windows** (fixed, non-overlapping — e.g., the 12:00–12:01 bucket).
The hard part is **event-time vs processing-time**:

- **Processing-time** windows (bucket by *when the processor saw it*) are simple but wrong: a click that
  happened at 11:59 but arrives at 12:02 (mobile offline, retry, network delay) lands in the *wrong
  minute*. Dashboards and billing both want **event-time** (the click's real `ts`) so the per-minute
  number reflects reality.
- **Event-time needs watermarks.** A **watermark** is the processor's assertion "I believe I've now
  seen all events with `ts ≤ T`." When the watermark passes a window's end, the window **fires** (emits
  its rollup). The watermark is `max_event_ts_seen − allowed_lateness` (e.g., trailing by 30s–2min).
- **Late/out-of-order events** (arriving after the watermark passed their window): three policies —
  **(1) drop** (simplest, fine for the approximate dashboard), **(2) update** the already-emitted window
  (re-fire with a correction — what we want for accuracy if the OLAP store supports upserts), or **(3)
  side-output** late events to a dead-letter stream for the batch layer to absorb. I'd use **(2) for a
  bounded grace period, then (3)** — and crucially, **the batch layer catches everything the stream
  dropped**, so no billable click is ever lost to lateness. That's why the dashboard can afford to drop
  late data: the truth layer is the safety net.

> **Tradeoff stated:** a *short* watermark lag = fresher dashboards but more events arrive "late" and
> get dropped/corrected; a *long* lag = more complete windows but slower dashboard updates. This is the
> **latency-vs-completeness knob** of the speed layer, and it's tunable per the freshness SLA.

### 6E. Pre-aggregation / rollups + bounding the cardinality explosion

The stream emits rollups keyed by `(time_bucket, dimensions…)`. **Rollup-by-time-bucket-and-dimension**
is what makes the dashboard fast (query a few rows, not a billion events). But cardinality explodes
multiplicatively:

- Rows ≈ `time_buckets × Π(dimension_cardinalities)`. Add a high-cardinality dimension (e.g.,
  `userId` or raw `URL`) and the rollup table approaches the *size of the raw data* — defeating the
  purpose.
- **Bounding tactics:** (1) **whitelist low-cardinality dimensions** for rollups (campaign, region,
  device — *not* userId); (2) **roll up coarser over time** — keep per-minute for 24h, then compact to
  per-hour, then per-day (old data is queried coarsely anyway); (3) push **unique-user** (the
  high-cardinality question) into a **sketch** (HLL, next), not into rollup rows; (4) cap top-K
  dimensions with a **heavy-hitter** structure rather than materializing every value.

### 6F. Approximate structures where exactness isn't needed (**doc 18**)

For the *dashboard*, exact is wasteful — these sketches give bounded-error answers in tiny, mergeable
state:
- **HyperLogLog** for **unique users** per (campaign, window): counting distinct `userId`s exactly
  needs a set of every id (GBs); HLL estimates cardinality in **~1.5 KB** with ~1–2% error, and HLL
  registers **merge** trivially (union two windows → one). Perfect for "unique users this hour/day."
- **Count-Min Sketch** for **top-K / heavy hitters** ("top 10 ads by clicks"): tracks approximate
  frequencies in fixed memory regardless of how many distinct ads exist; pair with a small top-K heap.
  Over-estimates slightly (never under) — fine for "which ads are hot," never for billing.
- **The rule:** sketches serve the **speed/dashboard** layer (approximate OK). **Billing never uses a
  sketch** — it uses the batch layer's exact `count(DISTINCT …)`. Again the split: approximate for
  fast, exact for money.

### 6G. Storage & serving layer (**doc 20, doc 03**)

- **Raw events → data lake** (S3/object store, **Parquet** columnar files, partitioned by date/hour).
  Cheap, durable, immutable — the recompute/audit source. ~36 TB for 90 days compressed.
- **Rollups + sketches → OLAP/columnar store** (Druid / ClickHouse / Pinot). Built for time-range +
  group-by-dimension + aggregate; column compression on repetitive dimensions; sub-200ms scans. This is
  what the **dashboard** queries.
- **Billing aggregates → transactional warehouse** (Snowflake/BigQuery or a relational store), exact and
  audited.
- **Serving layer for dashboards** caches hot queries (last-hour metrics for active campaigns) in Redis
  in front of OLAP; pre-materializes the most-watched campaign dashboards.

### 6H. Reconciliation: batch corrects the stream

The reconciliation loop is the glue of Lambda:
1. Batch job runs (e.g., hourly for recent correction, daily for finalization) over the lake.
2. Recomputes exact counts per (campaign, dimension, time-bucket) — dedup global, fraud rules current,
   late events all included (they're in the lake regardless of stream watermarks).
3. **Overwrites** the stream's approximate rollups for now-finalized windows in OLAP (so the dashboard's
   *historical* numbers converge to exact), and writes the **billing** table with `status: final`.
- Dashboards thus show **provisional (stream) numbers for the last few minutes** and **reconciled
  (batch) numbers for older windows** — fresh *and* eventually-exact.
- Reconciliation must be **idempotent** (re-running a batch pass produces the same overwrite) and track
  the **source offset/file range** it processed, so a re-run or backfill is deterministic.

### 6I. Fraud / invalid-click filtering hook

Both layers apply a shared filter *before* counting:
- **Cheap inline rules (stream):** drop same-`userId`+`adId` clicks within N seconds (double-click),
  obvious bot user-agents, clicks with no preceding impression, rate spikes per IP.
- **Heavier offline detection (batch/ML):** click-farm patterns, anomalous CTR per source — too slow
  for the stream, so the **batch layer re-filters** with the better model and the reconciled billing
  number reflects it. This is *another* reason billing trusts batch: fraud detection is more thorough
  there. Invalid clicks are **side-outputted**, not deleted — kept for audit and model training.

---

## Step 7 — Wrap-up (3 min)

**Failure modes & graceful degradation**
- **Stream processor lag** (consumer falls behind the firehose): dashboards go **stale, not wrong** —
  they show older windows. Kafka retains the backlog (it's the buffer); the processor catches up.
  Autoscale processor parallelism off **consumer lag**. Billing is unaffected (it's batch). This is the
  speed/truth split paying off: the layer that can fall behind is the one that's allowed to.
- **Duplicate events** (retries/redelivery): handled by **`eventId` dedup** (6C) — stream is
  effectively-once within the window; batch is exactly-once over all history.
- **Late / out-of-order data**: stream corrects within the grace window then defers to batch (6D);
  **batch catches everything**, so no billable click is lost.
- **Backfill / reprocessing** (bug in stream logic): because raw events are immutable in the lake and
  the log is replayable, we **recompute from source** — fix the shared logic, re-run batch, reconcile
  OLAP. This is *why* we keep raw events: the stream is disposable, the raw data is not.
- **OLAP store down**: dashboards degrade (serve cached/last-known) — no data lost; rollups buffer in
  the stream/log until it recovers.
- **Ingestion API down**: the worst case (lost clicks = lost revenue) — so it's the most-replicated,
  simplest component; edge SDKs buffer-and-retry locally during a blip.

**Single points of failure & redundancy**
- **Kafka** is the spine — RF=3 + multi-broker, per-partition leader failover (**doc 05**); it's the
  durable buffer that makes everything downstream replayable.
- **The data lake is the true SPOF for billing** — but object storage is 11-nines durable and
  immutable; that's *why* truth lives there, not in a mutable DB.
- **No global SPOF by design** — the two layers are independent; the speed layer can fail entirely and
  billing still bills (from the lake), and the batch layer can fail and dashboards still update (from
  the stream). Decoupling the consumers decoupled their failure domains.

**The accuracy-vs-latency tradeoff, restated:** every choice traces to "fast and approximate
(dashboard) or slow and exact (billing)?" The stream layer is PA/EL; the batch layer is PC/EC;
reconciliation makes the fast numbers converge to the exact ones over time.

**With more time:** sessionization and attribution windows (which impression *caused* the click),
multi-region ingestion with regional aggregation then global merge (**doc 22**), exactly-once
Kafka→OLAP via transactional sinks, real-time fraud ML inline, budget-pacing feedback (stop serving an
ad when its campaign hits its daily cap — which makes the *stream* number suddenly billing-critical and
tightens its accuracy bar), and tiered OLAP (hot per-minute, cold per-day).

---

## What made this staff-level

- **Derived the entire architecture from one observation** — *two consumers of the same stream need
  opposite things* — and made that the thesis, instead of drawing a generic Kafka→Flink→DB pipe.
- **Named Lambda vs Kappa as a real decision**, picked Lambda *with the duplication-minimizing library*,
  and stated the exact condition (cheap/long-enough log replay) that would flip me to Kappa — choosing a
  side *and* its boundary.
- **Treated counting as an idempotency problem** (**doc 10**): named the at-least-once double-count bug,
  fixed the *increment* with `eventId` dedup, and explained precisely **why billing trusts the batch's
  exact distinct-count over the stream's bounded/Bloom-approximate dedup.**
- **Got windowing right** — event-time over processing-time, watermarks, and a concrete late-data policy
  (correct-then-side-output) with the **batch layer as the safety net** that lets the dashboard afford
  to drop late data.
- **Used approximate structures deliberately** — HLL for unique users, Count-Min for top-K — and drew
  the hard line that **billing never uses a sketch**, tying every approximation back to the speed/truth
  split.
- **Bounded the cardinality explosion** explicitly (`buckets × Π(dimensions)`), with whitelisting,
  time-coarsening, and pushing high-cardinality questions into sketches — instead of hand-waving
  "we roll up by dimension."
- **Picked three stores for three access patterns** (immutable lake / columnar OLAP / transactional
  billing) from the data model, not by reflex, and justified each.
- **Decoupled the failure domains** — speed can die without breaking billing and vice-versa — and showed
  the lake/log replayability is what makes backfill and reprocessing safe.

---

### Self-check before the mock (answer these from memory)
- [ ] Name the two consumers and the opposite requirement each has — and which side of PACELC each sits on.
- [ ] State Lambda vs Kappa, which you pick here, and the exact condition that would flip you to Kappa.
- [ ] What's the naive double-count bug, and how does `eventId` dedup fix the *increment*? Why does
      billing trust batch over stream dedup?
- [ ] Event-time vs processing-time: what is a watermark, and what are the three late-data policies?
      Why can the dashboard afford to drop late data?
- [ ] Give the cardinality-explosion formula and three ways to bound it.
- [ ] When do you use HyperLogLog vs Count-Min Sketch, and what is the one place you *never* use a sketch?
- [ ] Why three different stores (lake, OLAP, billing warehouse) — which access pattern picks each?
- [ ] How does reconciliation work, and why must it be idempotent + offset-tracked?
- [ ] Why partition the log by `campaignId`, what's the hot-partition risk, and how do you fix it?
- [ ] What happens when the stream processor lags, and why is that acceptable but ingestion downtime is not?
