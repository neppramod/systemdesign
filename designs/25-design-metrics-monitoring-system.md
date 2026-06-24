# Design 25: Metrics & Monitoring System (Prometheus / Datadog-style)

> **Why this one matters:** this is the canonical *time-series at scale* design, and it's the place to
> prove you understand a workload almost nothing else has — **a relentless, append-only write firehose
> where the data is born already sorted by time, and the single greatest scaling enemy isn't volume,
> it's *cardinality*.** A metric isn't a row; it's a `(metric_name, {labels}) → stream of (ts, value)`
> series, and the product of all distinct label values is the number of distinct series you must index,
> route, and keep in memory. Add one unbounded label (a `user_id`, a `request_id`, a raw URL) and your
> series count explodes from millions to billions — the system doesn't get slow, it falls over. Bounding
> cardinality *is* the design.
>
> The one-line thesis I state up front and keep returning to: **we ingest a massive append-only stream
> of timestamped samples, route each sample to a shard by hashing its series identity, store it in a
> purpose-built TSDB (in-memory head block + WAL, flushed to immutable time-partitioned blocks with
> Gorilla compression), serve dashboard and alert queries by scatter-gathering across shards and merging,
> downsample old data to trade fidelity for storage, and run the alerting engine in an independent
> failure domain because the monitoring system must survive the very outages it exists to detect.**
> Every hard choice traces to one of three pressures: *write throughput*, *cardinality*, or *the
> independent-failure-domain constraint*. I'll run all 7 steps but spend most of the budget in **Step 6**,
> because that's where this round is won. (Ties to **observability/SRE doc 27**, **databases doc 03**,
> **sharding doc 04**, **messaging doc 07**, **notification design 13**.)

---

## Step 1 — Requirements (5 min) — I drive this

"Metrics & monitoring" is unbounded, so I scope and position before drawing. The framing move is to
name the workload's defining shape in the first sentence — **mostly-write, bursty-read, cardinality-bound**
— because that fork drives the whole architecture.

**Functional**
- **Ingest** time-series metrics from a large fleet of hosts/services/containers: each sample is
  `(metric_name, labels{...}, timestamp, float64 value)`. Counters, gauges, histograms, summaries.
- **Store** them efficiently with a defined **retention** policy (e.g., 15 days raw, 13 months rolled-up).
- **Query / aggregate** for dashboards: range queries over a time window with aggregation across many
  series (`sum / avg / rate / p99 by (label)`), e.g. *"p99 latency of `api` service grouped by region,
  last 6 hours."*
- **Alert**: evaluate user-defined rules continuously (`error_rate > 0.05 for 5m`), fire/resolve, dedup,
  group, and route notifications.
- **Downsample / roll up**: collapse raw resolution into coarser aggregates as data ages.
- **Serve dashboards** (visualization) backed by the query layer.

I'll explicitly **scope out**: log aggregation and distributed tracing (separate pillars — **doc 27**;
metrics is one of the three observability signals), and the dashboard UI rendering itself. I'll mention
where logs/traces would attach.

**Non-functional** — this is where the design is decided:
- **Write scale is the headline** — this is a write-firehose: ~**10–30M samples/sec** at fleet scale
  (derived in Step 2). The entire ingest + storage path is shaped around absorbing that append stream.
- **Cardinality is the scaling enemy** — total *active distinct series* (not samples) is what we must
  index and hold in memory: **10M–100M+ active series**. Unbounded labels make this explode without
  limit. **Bounding and defending against cardinality is the central non-functional requirement.**
- **Read pattern is bursty and fan-out-heavy** — far fewer QPS than writes (dashboards refresh every
  10–30s; humans look during incidents), but a single query can touch **thousands of series across many
  shards** and must aggregate them. Reads spike hard during an incident — exactly when the system is
  most stressed.
- **Freshness** — a sample should be queryable within **seconds** of scrape (alerting on a 5-minute-old
  number is useless during an outage).
- **Query latency** — dashboard p99 **< 1s** for typical range+aggregate; bound expensive queries so one
  bad query can't OOM the system.
- **Consistency** — **eventually consistent and lossy-tolerant is acceptable.** Monitoring data is not a
  bank ledger: a dropped sample during a spike is far better than refusing writes. We trade exactness
  for availability of ingestion. (Contrast with the ad-billing design 18, where the count was money.)
- **Availability of ingestion ≫ availability of any single query.** Never block a scrape.
- **The independent-failure-domain constraint** — *the monitoring system must survive what it monitors.*
  If a region's outage also takes down the monitoring of that region, you're blind exactly when you need
  to see. This shapes deployment (separate infra, separate blast radius), and it's a constraint no other
  design in this set has. I'll flag it now and design for it in Step 6.

> **The staff move in Step 1:** I'm *positioning* before drawing. The thesis — "append-only TSDB, hash
> by series, scatter-gather reads, downsample for retention, independent failure domain for alerting" —
> makes every later choice derivable. The PACELC read: ingestion is **PA/EL** (under partition or
> overload, drop/shed rather than block — accept a hole in the graph, never refuse the write). The query
> side is also **EL** (serve partial/best-effort results fast rather than block for a slow shard). The
> only **PC/EC**-leaning component is *alert rule state* (we'd rather be correct about "is this alert
> firing" than fast). Naming that the *write* path and the *alert* path sit on opposite sides of PACELC
> is the senior framing.

---

## Step 2 — Estimations (3 min) — to justify, not impress

I'll pick numbers that *force* the architecture. Target: a large fleet — think a mid-size company's
entire infra, or a single big Datadog/Prometheus tenant.

**Sample (write) rate** — the firehose:

| Quantity | Formula | Result |
|---|---|---|
| Hosts/targets | given | **10,000 hosts** |
| Metrics per host | OS + app + container metrics | **~1,000 active series/host** |
| Total active series | 10,000 × 1,000 | **~10M active series** |
| Scrape interval | typical | **15 s** |
| Samples/sec | 10M series ÷ 15 s | **~670K samples/s** |
| Push to 10× scale (containers, fine-grained labels) | ×10–30 | **~7M–20M samples/s** → **provision for ~10M+/s** |

**The headline number is series count (cardinality), not sample rate.** 10M active series is the
baseline; a single careless label (`pod_id` on ephemeral pods, `customer_id`, raw `path`) multiplies it.

| Quantity | Formula | Result |
|---|---|---|
| Bytes per sample, naive | ts(8B) + value(8B) + series ref(8B) | **~24 B raw** |
| Bytes per sample, **Gorilla-compressed** | delta-of-delta ts + XOR value (see 6C) | **~1.3–2 B/sample** |
| Raw ingest volume (uncompressed) | 10M/s × 24 B | **~240 MB/s** |
| Stored volume (compressed) | 10M/s × 1.5 B | **~15 MB/s → ~1.3 TB/day** |
| Raw retention (15 days) | 1.3 TB × 15 | **~19 TB hot** |
| With downsampling (13 months @ 1h rollup) | rollup is ~100–1000× smaller | **adds ~tens of TB, not PB** |
| Index / in-memory state | ~10M series × ~few KB (labels + chunk head) | **tens of GB RAM per replica** of the index |

> What the math *justifies*:
> - **10M+ samples/s, 240 MB/s** → no single node ingests this → **horizontally sharded ingestion +
>   storage, sharded by series identity** (justified, not assumed).
> - **Gorilla compression takes 24 B → ~1.5 B (≈16×)** → this is *why* we use a purpose-built TSDB and
>   not Postgres/Cassandra rows (16× on every sample, forever, is the difference between 19 TB and
>   300 TB). I'll derive the compression in 6C.
> - **Series count (10M–100M) is the in-memory index pressure** → cardinality limits, not disk, are the
>   first thing to fall over → **bounding cardinality is a first-class design problem (6B).**
> - **Reads are few-QPS but fan-out across thousands of series and many shards** → **scatter-gather query
>   layer with cost limits** (justified), not a simple point-lookup store.
> - **Raw vs rolled-up gap is ~100–1000×** → **downsampling + tiered retention** is what makes 13-month
>   retention affordable (6F).

---

## Step 3 — API design (3 min)

Three surfaces matching the three jobs — write, read, alert. I deliberately split them so the contract
shows the workload's shape.

```
# --- WRITE: ingestion (the firehose) ---
# PUSH model (agents/StatsD-style): clients send batched samples
POST /api/v1/write        # Prometheus remote-write style, protobuf + snappy
   body: [ { labels: {__name__:"http_req_duration", service:"api", region:"us-east", le:"0.5"},
             samples: [ {ts, value}, ... ] }, ... ]   # batched, many series per request
   -> 204 No Content      # fire-and-forget; never blocks on storage

# PULL model (scrape, Prometheus-native): the system scrapes targets
GET  http://<target>:<port>/metrics      # target EXPOSES current values in text/exposition format
   -> http_req_duration{service="api",le="0.5"} 1234   # scraper pulls on an interval

# --- READ: query (dashboards) ---
GET  /api/v1/query_range?query=<PromQL>&start=&end=&step=
   query: sum(rate(http_req_duration_count{service="api"}[5m])) by (region)
   -> { series: [ {labels, values:[[ts,val],...]}, ... ] }   # range vector, stepped
GET  /api/v1/query?query=<PromQL>&time=          # instant query (single point), for alert eval
GET  /api/v1/labels , /api/v1/series             # metadata / autocomplete for dashboards

# --- ALERT: rule + notification management ---
POST /api/v1/rules        { name, expr: <PromQL>, for: "5m", labels, annotations, severity }
GET  /api/v1/alerts       -> [ {name, state: pending|firing|resolved, activeAt, labels} ]
```

- **Write is batched and returns immediately (204)** — the firehose must absorb spikes; never block a
  scrape on storage (Step 6D).
- **Query is PromQL-style** — range queries with an aggregation operator and a `by (labels)` grouping;
  this *is* the fan-out + merge contract, made explicit. I'll enforce **query cost limits** (max series
  touched, max points returned, timeout) at this layer (6G).
- **Both push and pull write paths are first-class** — the choice between them is a real tradeoff I'll
  defend in 6A (it's the one most candidates get hand-wavy on).
- Auth/rate-limit/tenant-isolation live at the gateway (multi-tenant: every series is implicitly scoped
  by a `tenant` label and per-tenant cardinality quotas).

---

## Step 4 — Data model (5 min)

Entities + **access patterns** — the access pattern picks the store, not reflex.

**The time series — the core entity.** A series is identified by its **metric name + the full set of
label key/values**:

```
series identity = __name__="http_req_duration" + {service="api", region="us-east", le="0.5", host="h7"}
   ↳ hashed into a 64-bit SERIES ID (fingerprint of the sorted label set)
   ↳ that series is an append-only stream of (timestamp, float64 value) samples, ordered by time
```

- **Access patterns:**
  1. *Write:* append `(ts, value)` to the series identified by its label set — always the newest
     timestamp, always to the head (append-only, time-ordered). This is the firehose.
  2. *Read:* given a metric name + a **label matcher** (`{service="api", region=~"us-.*"}`) and a **time
     range**, find *all* matching series and scan their samples in the range. This is **two lookups**:
     (a) label matcher → set of series IDs (an *inverted index* problem), then (b) series ID + time range
     → samples (a *contiguous range scan* problem).
- These two access patterns want **two different indexes**, and that's the heart of the data model:

**(a) The inverted index — labels → series IDs.** `region="us-east"` → posting list of all series IDs
having that label. A query like `{service="api", region="us-east"}` **intersects** posting lists (exactly
the **inverted-index / search** primitive — **doc 11**). This is what makes "give me all series matching
this matcher" fast, and it's the structure that **cardinality blows up**: every distinct label *value*
is a posting list, and the number of *series* is the product of label cardinalities.

**(b) The chunk store — series ID + time → samples.** For each series, samples are stored as
**time-ordered, compressed chunks** (a chunk ≈ a couple hours of one series). Access is always *range
scan by series ID over a time window*, so we store chunks **contiguously per series, partitioned by
time** — and compress hard (6C).

> **SQL vs NoSQL falls out of the access pattern, not reflex:** the workload is *append-only, time-
> ordered, scanned by series+range, with a separate inverted index on labels.* That is **none** of:
> - **Relational/OLTP** — row-per-sample with B-tree indexes wastes ~10×+ space and the write
>   amplification kills you at 10M/s. Rejected.
> - **General LSM KV (Cassandra)** — closer (write-optimized, **doc 03**), and some TSDBs are built on it,
>   but it doesn't natively do delta-of-delta/XOR compression or the label inverted index, so you bolt
>   those on. Workable, not ideal.
> - **A purpose-built TSDB** — append-only head + WAL + immutable time-partitioned blocks, Gorilla
>   compression, integrated inverted index. **This is the right shape**, and I derive *why* in 6C. The
>   access pattern (not reflex) chose it.

**Alert rule state** — small, needs correctness: `(rule_id, expr, for_duration, current_state,
active_since)`. Lives in a **strongly-consistent coordination store** (etcd/Raft, **doc 08**) so alert
state survives evaluator restarts and isn't double-fired — the one PC/EC corner of this design.

---

## Step 5 — High-level design (10 min) — happy path end-to-end

```
   hosts / services / containers
   (expose /metrics  OR  run a push agent)
        │  pull (scrape)        │  push (remote_write, batched)
        ▼                       ▼
   ┌──────────────┐      ┌────────────────────┐
   │  SCRAPERS    │      │  INGESTION GATEWAY  │  auth, tenant, RATE/CARDINALITY LIMIT
   │ (service-    │─────►│  (stateless tier)   │
   │  discovery)  │      └─────────┬───────────┘
   └──────────────┘                │  hash(series labels) → shard
                                   ▼
                    ┌───────────────────────────────────────┐
                    │  (optional) Kafka buffer  topic:samples │  absorbs bursts, replay, decouple
                    │   partitioned by series-hash            │  (doc 07)
                    └───────────────────┬─────────────────────┘
                                        │  consumed by owning shard
        ┌───────────────────────────────┼───────────────────────────────┐
        ▼                                ▼                                ▼
  ┌──────────────┐               ┌──────────────┐               ┌──────────────┐
  │ TSDB SHARD 0 │               │ TSDB SHARD 1 │     ...       │ TSDB SHARD N │   (each replicated ×3)
  │  WAL + HEAD  │               │  WAL + HEAD  │               │  WAL + HEAD  │
  │  (in-mem)    │               │              │               │              │
  │  ▼ flush     │               │              │               │              │
  │ immutable    │               │              │               │              │
  │ time blocks  │──────────────────────────────────────────────► OBJECT STORE (cold blocks, downsampled)
  │ + inverted   │               │              │               │
  │   index      │               │              │               │
  └──────┬───────┘               └──────┬───────┘               └──────┬───────┘
         └───────────────┬──────────────┴──────────────┬───────────────┘
                         ▲  scatter-gather (fan-out)    │  merge
                  ┌──────┴───────────────────────────────────────┐
                  │  QUERY LAYER (PromQL engine, cost limits,     │
                  │   query cache, scatter→gather→aggregate)      │
                  └──────┬───────────────────────────┬───────────┘
                         ▼                            ▼
                 ┌───────────────┐          ┌─────────────────────────────────┐
                 │  DASHBOARDS   │          │  ALERTING ENGINE (INDEPENDENT     │
                 │  (Grafana)    │          │   FAILURE DOMAIN)                 │
                 └───────────────┘          │  • eval loop: run rule exprs      │
                                            │  • for-duration state (Raft)      │
                                            │  • dedup + group → Alertmanager   │
                                            └──────────────┬────────────────────┘
                                                           ▼
                                              NOTIFICATION FAN-OUT (design 13)
                                              PagerDuty / Slack / email / SMS
```

**Walk one sample through it (write path):** A target exposes `/metrics`; a **scraper** (which knows the
target list via **service discovery** — 6A) pulls the current values every 15s (or an agent **pushes**
them, batched). The **ingestion gateway** authenticates the tenant, **enforces the per-tenant cardinality
quota** (rejects/aggregates-away new series past the limit — 6B), computes `hash(sorted label set) →
shard`, and routes the sample (optionally through a **Kafka buffer** so bursts and shard restarts don't
drop data — 6D). The **owning TSDB shard** appends the sample to the in-memory **head block** *and* the
**WAL** (durability), and adds the series to its **inverted index** if new. Within seconds, that sample
is queryable.

**Walk one query through it (read path):** A dashboard issues `sum(rate(http_req_duration_count{
service="api"}[5m])) by (region)`. The **query layer** parses it, uses the inverted index to find which
shards hold matching series, **scatter-gathers** the raw samples from all of them (fan-out), then
**merges and aggregates** — `rate()` per series, then `sum by (region)` — and returns the stepped range
vector. **Cost limits** cap how many series/points one query may touch (6G).

**The alert path** runs the *same* query engine on a timer (the eval loop), but in an **independent
deployment** so it survives outages of everything else (6H). Keep it this simple first; evolve under
questioning.

---

## Step 6 — Deep dives (15 min) — where the round is won

I'll propose the hard part: *"The interesting tension here is that this is a write-firehose where the
real scaling limit is **cardinality**, not bytes — plus the constraint that monitoring must survive what
it monitors. Can I go deep on the TSDB storage engine, cardinality bounding, and the failure-domain
design?"*

### 6A. Collection: pull (scrape) vs push (agents) — the real tradeoff (**doc 27**)

This is the choice most candidates wave off; I'll commit and justify.

- **Pull (Prometheus scrape):** the monitoring system holds a **target list** (via **service discovery**:
  Kubernetes API, Consul, EC2 tags) and **scrapes** each target's `/metrics` endpoint on an interval.
  - *Pros:* the monitoring system controls the schedule (no client-side flooding); a target being
    *scrapable* is itself a **liveness signal** (`up==0` is a free health check); no credentials pushed
    outward; easy to run a target locally and curl it. Targets stay dumb.
  - *Cons:* needs **service discovery** to know what exists; struggles with **short-lived/batch jobs**
    (a 5-second cron job may die before the next scrape — solved with a **push gateway** that holds the
    job's last values); and **firewalls/NAT** — the monitor must reach *into* every target, which is hard
    across network boundaries and the public internet.
- **Push (StatsD / Datadog agent / remote_write):** each host runs an **agent** that pushes batched
  samples *out* to the ingestion tier.
  - *Pros:* works through **firewalls/NAT** (outbound only), handles **short-lived jobs** naturally (push
    before exit), no central service-discovery needed, scales to the open internet (SaaS monitoring).
  - *Cons:* clients can **flood** the ingestion tier (a buggy client emitting millions of series — the
    cardinality DoS); you lose the free liveness signal (a silent host is ambiguous — dead or just not
    pushing?); needs client-side auth/credentials everywhere.

> **My choice: pull for owned infra inside a trust/network boundary; push for short-lived jobs, edge,
> and cross-boundary/SaaS.** A **hybrid** is what real systems do (Prometheus scrapes the cluster; a push
> gateway covers batch jobs; Datadog's agent pushes for SaaS where it can't reach in). The deciding
> question is **"can the monitor reach the target, and does the target outlive the scrape interval?"** —
> name that question and the answer falls out. Ties directly to **doc 27**'s pull-vs-push discussion.

### 6B. The cardinality problem — the central scaling challenge

**This is the make-or-break deep dive.** Restating the core fact: the number of distinct **series** you
must index and hold in memory is the **product of the cardinalities of every label**, summed across
metrics:

```
series_count  ≈  Σ_metrics  Π_labels  cardinality(label)
```

- A metric `http_requests{service, region, status, method}` with cardinalities `50 × 20 × 40 × 5` =
  **200,000 series** — fine. Add `user_id` (1M values) and it's **200 billion** series. The system
  doesn't slow down — the **inverted index and head block exhaust RAM and the shard dies.** This is the
  #1 way real monitoring systems fall over, and it's almost always a *human* mistake (someone put a high-
  cardinality field in a label).

**Why it's so dangerous:** every distinct series needs (a) an entry in the inverted index, (b) an active
chunk in the in-memory head, and (c) routing/merge work at query time. Cost is in **series count**, and
series count is *multiplicative* — one bad label multiplies everything.

**Bounding tactics (I name several, not one):**
1. **Label hygiene / schema discipline** — labels must be **bounded-cardinality dimensions** (service,
   region, status, instance), **never unbounded identifiers** (user_id, request_id, email, full URL,
   timestamp-in-label). This is the rule; everything else is enforcement.
2. **Per-tenant cardinality quotas at the ingestion gateway** — track active series per tenant/metric;
   when a tenant exceeds its limit, **reject new series** (return an error sample, emit a meta-alert) or
   **drop the offending label** so the series collapses. Better to lose a dimension than to die.
3. **Cardinality limiting per metric** — cap distinct series per metric name; beyond the cap, fold excess
   into an `__overflow__` bucket so the metric stays usable.
4. **Relabeling / dropping at the agent** — strip or aggregate-away high-cardinality labels *before* they
   hit storage (e.g., bucket raw URLs into route templates `/users/:id`).
5. **Detection** — continuously track top-cardinality metrics/labels and **alert on cardinality growth**
   itself (a sudden spike in series count is the early warning before OOM).
6. **High-cardinality questions belong in a different system** — "how many *distinct users* hit this
   endpoint" is a *logs/events* or **HyperLogLog** question (cf. design 18), **not** a metric label.
   Pushing that question out of the label set is the architectural fix.

> **Tradeoff stated:** every label you add buys you a new query dimension at a *multiplicative* cost in
> series count, RAM, and query fan-out. The discipline is to treat the label set as a **scarce, bounded
> schema**, enforce it at ingestion, and route truly high-cardinality questions to a system designed for
> them. Naming the multiplicative-explosion formula and *four ways to bound it* is the staff signal here.

### 6C. The TSDB storage engine — in depth (**doc 03**)

Why a **purpose-built TSDB** and not a general store? Two reasons, both derivable from the access pattern:

**(1) Gorilla compression — the reason for ~16× space savings.** Time-series data has two exploitable
regularities: timestamps are nearly evenly spaced, and consecutive values change little.

- **Timestamps → delta-of-delta.** Samples arrive at ~15s intervals, so consecutive deltas are ~constant
  (15s, 15s, 15s…). Store the *delta of the delta* — which is usually **0** — in a variable-length
  encoding (0 → a single bit). A perfectly regular series stores each timestamp in ~**1 bit**.
- **Values → XOR.** Consecutive float64 values are similar, so `value[n] XOR value[n-1]` has many leading/
  trailing zero bits; store only the meaningful middle bits with a small control header. A flat or slowly-
  changing gauge compresses to **a fraction of a byte per sample**.
- Net: the naive **24 B/sample → ~1.3–2 B/sample (~16×)**. This is *the* economic reason a TSDB exists,
  and why I rejected row-per-sample in SQL in Step 4. (This is Facebook's Gorilla / the Prometheus TSDB
  encoding.)

**(2) The write path — LSM-like, append-only (ties to LSM in doc 03):**

```
   sample ──► WAL (append, fsync-batched)         ← durability: replay on crash
          ──► HEAD block (in-memory, per-series mutable chunk, recent ~2h)   ← fast appends + recent reads
                    │  every ~2h
                    ▼
              IMMUTABLE BLOCK on disk
              ( time-partitioned: e.g. [12:00–14:00] )
                • chunks: Gorilla-compressed samples, contiguous per series
                • inverted index: label→postings, for THIS block's series
                • meta: min/max time, series count
                    │  background compaction (merge adjacent blocks, drop expired, downsample)
                    ▼
              larger blocks → eventually OBJECT STORE (cold)
```

- **WAL** gives crash recovery — a sample is durable (replayable) the moment it's appended, before it's
  compressed into a block. On restart, replay the WAL to rebuild the head. (The **WAL/commit-log**
  primitive from **doc 01/03**.)
- **Head block in memory** absorbs the high-frequency appends and serves recent reads (most dashboard
  queries are "last few hours" → served entirely from RAM, fast).
- **Immutable time-partitioned blocks** — once a 2h window closes, its block is **write-once, read-many**:
  compress it hard, build its inverted index, never mutate it. Immutability makes **compaction, deletion-
  by-retention (just drop whole expired blocks), backup, and offload-to-object-store** trivial — exactly
  the LSM benefit (**doc 03**), adapted: we partition by **time** (not just key), so dropping old data is
  deleting whole blocks, not range-deleting rows.
- **This is LSM-like** (memtable=head, WAL, immutable SSTables=blocks, compaction) but **specialized for
  time**: blocks are time-bounded, compression is Gorilla, and the index is a label inverted index.

**Partitioning is two-dimensional:**
- **By time** — blocks cover fixed windows (2h hot, compacted to larger cold blocks). Retention = drop
  expired blocks. Recent data in RAM, old data in object store.
- **By series** — across the fleet, series are sharded across **TSDB nodes by `hash(series labels)`**
  (next section). So a given sample lives at `(shard = hash(labels), block = time window)`.

### 6D. Ingestion pipeline at scale: sharding, bursts, backpressure (**doc 04, doc 07**)

10M+ samples/s can't land on one node. The pipeline:

```
agents/scrapers → stateless INGESTION GATEWAY → [Kafka buffer] → owning TSDB shard (×3 replicas)
```

- **Shard by `hash(sorted label set)`** — i.e., by **series identity**. This keeps *all samples of one
  series on one shard* (and its replicas), which is essential: a series' chunks must be contiguous, and
  rate()/aggregation over a series must see all its samples in one place. Hashing gives **even spread**
  (vs range-on-series, which would hot-spot) — the consistent-hashing primitive (**doc 04**) so adding/
  removing shards reshuffles minimal series.
- **The Kafka buffer (optional but I'd include it at this scale)** decouples ingestion from storage and
  **absorbs write bursts** — a deploy that spins up 10,000 new pods spikes the series/sample rate; Kafka
  holds the surge as a log instead of dropping samples or OOMing the shard (**doc 07**). It also makes the
  shard **restartable** (replay from the last committed offset rather than losing in-flight data) and lets
  us **rebuild/migrate a shard** by replaying. Partition Kafka by the *same* series-hash so a shard
  consumes exactly its partitions.
- **Handling bursts without the buffer** (if we skip Kafka for simplicity): the gateway applies
  **backpressure + load shedding** — when shards are saturated, **shed the lowest-priority samples**
  (e.g., drop to a coarser scrape interval, or reject new series first). The cardinal rule: **never block
  the scrape and never crash the shard** — a hole in the graph beats a dead monitoring system. This is the
  PA/EL choice from Step 1 made concrete.
- **Replication (×3, leaderless or leader-follower, doc 05)** — each series' samples go to 3 shard
  replicas. Since data is lossy-tolerant, we can use **best-effort quorum** (write to all 3, ack on 2);
  on read, query any replica (or query 2 and merge to fill gaps). Compare designs that need W+R>N for
  correctness — here we relax it because a missing sample is acceptable.

> **Tradeoff stated:** the Kafka buffer buys burst-absorption + replayable shard recovery at the cost of
> added latency and operational complexity (**doc 07**'s at-least-once → we tolerate duplicate samples
> because a TSDB write is **idempotent**: writing the same `(series, ts, value)` twice is a no-op, the
> point already exists). That idempotency is *why* at-least-once delivery is fine here, unlike billing in
> design 18.

### 6E. Hot shards / hot series

- **Hot series** — one extremely high-frequency series (e.g., a 1s-scrape metric on a mega-service). It's
  pinned to one shard by its hash. Mitigate by detecting and **isolating** it (dedicated shard), or
  accept it — a single series at even 1/s is cheap; the danger is *many* series (cardinality), not one
  fast series.
- **Hot shard from skew** — if hashing is uneven or one tenant dominates. Mitigate with **consistent
  hashing + virtual nodes** (**doc 04**) for even spread, and **per-tenant isolation** (a noisy tenant
  can't starve others — bulkhead, **doc 13/resilience**).

### 6F. Downsampling + retention / rollups — fidelity vs storage

Raw 15s resolution is only valuable while data is *recent* (you debug incidents at high resolution; you
look at last quarter's trend at low resolution). So we **downsample as data ages**:

```
raw (15s) ── keep 15 days ──►  1m rollups ── keep 90 days ──►  1h rollups ── keep 13 months
   (full fidelity, hot)          (10× smaller)                   (100×+ smaller, cold/object store)
```

- A **rollup** stores, per coarser bucket, the **aggregates** needed to answer queries faithfully:
  `(min, max, sum, count, and last)` — *not* just the average, because `count`+`sum` lets you compute a
  correct average across merged buckets, and min/max preserve spikes. (Storing only avg would lose the
  ability to re-aggregate — a classic rollup bug.)
- **Compaction does the downsampling** as a background job over immutable blocks: read raw block → emit
  1m-aggregate block → later 1h-aggregate block → drop the raw block when its 15-day TTL passes (drop the
  *whole block* — cheap, because blocks are time-partitioned and immutable, 6C).
- **The query layer picks resolution by range:** a "last 1h" query reads raw; a "last 6 months" query
  reads the 1h rollup automatically (you can't and shouldn't scan 6 months of 15s data). This is what
  makes long retention affordable.

> **Tradeoff stated — the storage-vs-fidelity knob:** high resolution forever is unaffordable
> (~1.3 TB/day raw × 13 months ≈ 500 TB); downsampling trades *fidelity of old data* (you can't see a
> 15s spike from 6 months ago) for *orders-of-magnitude less storage* and *queryable long retention*.
> The bet — almost always right — is that **recent data is debugged at high resolution, old data is
> trended at low resolution.** I keep min/max in rollups precisely so old data still shows that a spike
> *happened*, just not its exact 15s shape.

### 6G. Query layer: scatter-gather, aggregation, cost limits

A query like `sum(rate(http_req_duration_count{service="api"}[5m])) by (region)`:

1. **Resolve series:** use the inverted index to intersect posting lists for the matcher
   `{service="api"}` → set of matching series IDs, and which **shards** own them.
2. **Scatter (fan-out):** ask each owning shard for the raw samples of its matching series over the
   range. This is the **read fan-out** — one user query becomes N shard sub-queries (**scatter-gather**,
   the search/MapReduce primitive).
3. **Gather + merge:** collect per-shard results; for each series compute `rate()` (per-series, so it can
   be pushed *down* to the shard — a key optimization), then **merge across shards** and apply
   `sum by (region)`.
4. **Push-down optimization:** do as much aggregation **at the shard** as possible (partial sums per
   region per shard) and merge partials at the query layer — moves less data over the network, the
   classic map-side-combine. Some aggregations (sum, count, min, max) are **associative** so they
   push down cleanly; others (quantiles over raw samples) need the raw points and can't fully push down.

**Query cost limits — non-negotiable (the OOM defense):** one query can ask for billions of points (e.g.,
`{}` matching *all* series over 13 months) and **OOM the query layer**. So:
- **Max series per query**, **max samples/points returned**, **max memory per query**, **hard timeout**.
- **Reject** or **truncate** queries that exceed them, with a clear error. A bursty incident-time read
  load must never let one expensive dashboard panel take down the query tier (bulkhead per query, **doc
  13**).
- **Query cache** in front (Redis): dashboard panels re-issue the *same* range query every 15–30s; cache
  the older, immutable portion of the range (the last step is the only changing part) — caching old time
  ranges is safe because *historical data is immutable* (great cache key behavior).

> **Tradeoff stated:** reads are cheap in QPS but expensive in fan-out; the defenses are push-down
> aggregation (less data moved), an immutable-range query cache (historical data never changes), and hard
> cost limits (one query can't OOM the tier). The fan-out is *why* the query layer is a separate, scalable
> stateless tier in front of the shards.

### 6H. Alerting subsystem — eval loop, dedup/grouping, fan-out, SLO burn-rate (**doc 27, design 13**)

Alerting is a **read-then-decide-then-notify** loop, deliberately split from the data plane:

- **The eval loop:** the alert engine holds a set of rules (`expr`, `for` duration, labels). On a fixed
  interval (e.g., every 15–30s) it **runs each rule's PromQL query** through the same query engine and
  checks the result against the condition.
- **The `for` duration / pending→firing state:** a rule like `error_rate > 0.05 for 5m` must be true
  **continuously for 5 minutes** before firing — this kills flapping on a single bad scrape. So the engine
  keeps **per-alert state** (`pending` since when, `firing`, `resolved`). That state must survive evaluator
  restarts and **must not be double-owned** (two evaluators firing the same alert) → store it in / lease
  it from a **Raft/etcd-backed coordinator** (**doc 08**); evaluators take a **lease** on rule groups
  (leader election per group). This is the one **PC/EC** component — correctness of alert state beats
  latency.
- **Dedup + grouping → the notification path (design 13):** raw alerts are noisy (a rack failure fires
  500 host-down alerts). An **Alertmanager**-style layer **deduplicates** identical alerts, **groups**
  related ones (by `cluster`/`service` label) into one notification, applies **silences/inhibition** (if
  the whole datacenter is down, suppress the 500 per-host alerts), and then does **notification fan-out**
  with retries, rate-limiting, and routing to channels (PagerDuty/Slack/email/SMS). This is exactly the
  **notification system (design 13)** — dedup, grouping, multi-channel fan-out, delivery retries — so I
  reuse that design wholesale rather than reinventing it.
- **SLO / burn-rate alerts (the SRE-grade move, doc 27):** instead of alerting on a raw threshold
  (`error_rate > 5%` — too noisy or too slow), alert on **error-budget burn rate**: "are we burning the
  monthly error budget fast enough that we'll exhaust it?" Multi-window burn-rate alerts (a fast-burn rule
  on a 1h window for pages, a slow-burn rule on a 6h window for tickets) page only when the *budget* is
  genuinely threatened — fewer false pages, faster real ones. This is the modern SRE alerting pattern and
  ties straight to **doc 27**'s SLO material.

### 6I. High availability + the "monitoring must survive what it monitors" constraint

This is the constraint unique to this design. **If the monitoring system shares a failure domain with the
thing it monitors, it goes blind exactly when you need it most.**

- **Independent infrastructure / blast radius:** run the monitoring stack on **separate clusters, separate
  power/network/availability zones**, ideally a separate account/region from the workloads it watches.
  When `us-east` workloads die, the `us-east` *monitoring* must not die with them. (Often: per-region
  monitoring + a separate global tier, so a regional outage is *observed*, not *hidden*.)
- **Redundant, independent scrape/eval paths:** run **two (or more) identical Prometheus/ingestion
  replicas scraping the same targets** in parallel (no coordination) — if one dies, the other still has
  the data. They diverge slightly (different scrape moments) but that's fine for monitoring. The **alert
  engine** runs redundantly too, with dedup downstream (Alertmanager dedups the duplicate alerts from the
  two replicas).
- **Don't depend on the monitored system's dependencies:** the alert *notification* path must not route
  through the infra it monitors (e.g., don't send pages through your own email service if you also
  monitor that email service — use an external paging provider). The **dead-man's-switch / watchdog
  alert**: a rule that is *always firing* and whose *silence* (no alert received) means "the monitoring
  pipeline itself is down" — this is how you monitor the monitor.
- **Federation / hierarchy for scale + isolation:** lower-tier Prometheus instances scrape locally; a
  global tier scrapes *aggregates* from them (federation). Keeps fan-out local and isolates failure to a
  tier.

> **The staff insight:** every other system in this set optimizes for serving its users; this one has a
> meta-requirement — **it must remain operational precisely during the failures it exists to report.**
> That forces *independent failure domains*, *redundant uncoordinated collectors*, an *externally-hosted
> notification path*, and a *dead-man's-switch*. Naming this constraint and designing for it is the
> single biggest differentiator in this round.

### 6J. Dashboards / visualization serving

- Dashboards (Grafana-style) are **read clients** of the query layer — they issue many `query_range`
  panels on a refresh timer. Two scaling notes: (1) the **query cache** (6G) absorbs the repetitive,
  mostly-immutable range re-fetches; (2) dashboards are a **bursty read amplifier during incidents** (a
  hundred engineers open the same dashboards), so the query tier autoscales and cost limits protect it.
- Pre-render/pre-aggregate the **most-watched dashboards** (recording rules — see below) so the heaviest
  panels read a single pre-computed series instead of re-aggregating thousands at view time.

### 6K. Recording rules (pre-aggregation) — the read-side cardinality fix

Mirror the downsampling idea on the *query* side: a **recording rule** continuously evaluates an
expensive aggregation (`sum(rate(http_req_count[5m])) by (service)`) and **writes the result back as a
new, low-cardinality series**. Dashboards and alerts then read the cheap pre-aggregated series instead of
re-scattering across thousands of raw series every refresh. This is **pre-aggregation / materialized view**
applied to time-series — it trades a little storage + write work for a huge read-fan-out reduction on hot
queries.

---

## Step 7 — Wrap-up (3 min)

**Failure modes & graceful degradation**
- **Cardinality blowup** (the #1 killer): a bad label multiplies series → index/head OOM → shard death.
  Defended at *ingestion* (per-tenant quotas, label limits, relabeling), *detected* (alert on series
  growth), and *contained* (reject new series, drop offending label — degrade a dimension, don't die).
  This is the failure I'd call out first and unprompted.
- **Ingestion overload / write burst** (mass deploy, traffic spike): Kafka buffer absorbs it, or the
  gateway **sheds load** (coarser interval, reject new series) — a **hole in the graph beats a dead
  monitor**. Never block the scrape. (PA/EL realized.)
- **Query OOM** (a `{}`-over-13-months query): hard **cost limits + per-query bulkhead + timeouts** so one
  query can't take down the read tier — critical because reads spike during incidents.
- **A TSDB shard dies:** ×3 replication means the data survives; the shard restarts and **replays its WAL
  / Kafka offset**. Recent in-flight samples may be lost in the worst case — acceptable (lossy-tolerant).
- **Stale/lagging shard:** queries return **partial results** (best-effort merge across available
  replicas) rather than failing — degrade, don't error.
- **Alert evaluator dies:** a redundant evaluator (with leased rule state in Raft) keeps firing;
  Alertmanager dedups. The **dead-man's-switch** catches total pipeline failure.

**Single points of failure & redundancy**
- **The inverted index / head block (RAM)** is the real soft spot — cardinality is what exhausts it; the
  whole 6B discipline exists to protect this. Not a classic SPOF, but the first thing that breaks.
- **The coordinator (Raft/etcd) for alert state** is a coordination SPOF — mitigated by Raft's own
  3/5-node quorum (**doc 08**); keep its scope tiny (just alert leases + rule config).
- **The monitoring system's own failure domain** — mitigated by the entire 6I design: separate infra,
  redundant uncoordinated collectors, externally-hosted notification path, dead-man's-switch.
- **No global data-plane SPOF by design:** shards are independent and replicated; the query and alert
  tiers are stateless and redundant; ingestion is stateless behind the gateway.

**The central tradeoff, restated:** monitoring data is **lossy-tolerant and write-dominated**, so every
choice favors *ingest availability and storage efficiency over per-sample correctness* — at-least-once +
idempotent writes, best-effort replication, load shedding, downsampling. The *only* place we flip to
correctness-first is **alert state** (Raft-backed, `for`-duration, dedup). And the meta-constraint —
**survive what you monitor** — overrides convenience everywhere it conflicts.

**With more time:** exemplars/trace-linking (jump from a latency spike to the trace — **doc 27**, the
three-pillars tie-in), multi-tenancy isolation deep-dive (per-tenant shards/quotas/billing), global query
federation across regions (**doc 22**), adaptive scrape intervals, anomaly-detection alerting (ML on the
series instead of static thresholds — **doc 23**), and object-store-native query (query cold blocks
directly from S3 with a caching gateway, à la Thanos/Cortex/Mimir).

---

## What made this staff-level

- **Identified the real scaling enemy as cardinality, not volume**, gave the multiplicative formula
  `Σ Π cardinality(label)`, and bounded it with *four+ concrete tactics* enforced at ingestion — instead
  of hand-waving "we'll shard the writes."
- **Justified a purpose-built TSDB from first principles** — derived **delta-of-delta + XOR (Gorilla)**
  compression for the ~16× win, and the **WAL + in-memory head + immutable time-partitioned blocks**
  LSM-like write path (**doc 03**), explaining *why* SQL row-per-sample and even a general LSM KV are the
  wrong shape.
- **Committed on pull-vs-push** with the deciding question ("can the monitor reach the target, and does
  the target outlive the scrape?") and landed on a justified hybrid (**doc 27**), rather than reciting
  both and picking neither.
- **Got the read path right** — scatter-gather fan-out, **push-down of associative aggregations**, an
  **immutable-range query cache**, and **hard cost limits as the OOM defense** — and explained why reads
  are low-QPS but high-fan-out and spike during incidents.
- **Treated retention as a fidelity/storage knob** — raw→1m→1h downsampling, keeping `min/max/sum/count`
  (not just avg) so old data re-aggregates correctly and still shows spikes — instead of "we set a TTL."
- **Designed alerting as a correctness-first control loop** — eval loop, `for`-duration anti-flap state in
  Raft, dedup/grouping/fan-out reusing the **notification design (13)**, and modern **SLO burn-rate**
  alerts (**doc 27**) — the one PC/EC corner of an otherwise PA/EL system.
- **Surfaced the constraint no other design has** — *the monitor must survive what it monitors* — and
  designed the independent failure domain, redundant uncoordinated collectors, externally-hosted
  notification path, and dead-man's-switch around it.
- **Placed every choice on PACELC deliberately**: write path PA/EL (shed/drop, never block), reads EL
  (partial results), alert state PC/EC (correct over fast), and used **idempotent writes** to justify
  why at-least-once delivery is fine here (unlike billing in design 18).

---

### Self-check before the mock (answer these from memory)
- [ ] What identifies a time series, and why is **cardinality** (not sample volume) the central scaling
      limit? Give the `Σ Π cardinality(label)` formula and four ways to bound it.
- [ ] Explain delta-of-delta (timestamps) and XOR (values) compression — roughly what ratio, and why it's
      the reason a purpose-built TSDB exists.
- [ ] Describe the TSDB write path (WAL, in-memory head, immutable time-partitioned blocks, compaction)
      and how it maps to LSM. How does retention become "drop whole blocks"?
- [ ] Pull vs push collection: the pros/cons of each and the one question that decides which to use.
- [ ] Why shard by `hash(series labels)`? What does the Kafka buffer buy, and why is at-least-once
      delivery fine here (what makes a TSDB write idempotent)?
- [ ] Downsampling: which aggregates do you store per rollup bucket and *why not just the average*? State
      the fidelity-vs-storage tradeoff.
- [ ] Walk a query through scatter-gather: index lookup → fan-out → merge. What pushes down, and what are
      the query cost limits defending against?
- [ ] Alerting: what is the `for` duration for, why does alert state need Raft, and what is an SLO
      burn-rate alert?
- [ ] State the "monitoring must survive what it monitors" constraint and four things you do to satisfy it
      (incl. the dead-man's-switch).
- [ ] Which single component sits on the PC/EC side of PACELC, and why is everything else PA/EL?
