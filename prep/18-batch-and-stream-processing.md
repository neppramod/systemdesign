# Topic 18: Big Data — Batch & Stream Processing, Analytics & Pipelines

> **Why this topic matters in the room:** A huge class of staff interviews is really an analytics
> problem wearing a product costume — "design ad click aggregation," "design YouTube view counts,"
> "design a trending/top-K service," "design a metrics dashboard." If you reach for a normal OLTP
> database and a `COUNT(*)`, you've already lost. The signal here is: do you know that analytics is a
> *different workload* with its own storage (columnar), its own compute (batch + stream), and its own
> tricks (sketches, rollups, approximate answers)? This doc gives you the vocabulary and the
> tradeoffs to derive those designs instead of memorizing them.

---

## Part A — OLTP vs OLAP: two workloads, two worlds

The first thing to say out loud on any analytics problem: **"This is an OLAP workload, not OLTP —
I'm going to keep it off the serving database."**

| | **OLTP** (Online Transaction Processing) | **OLAP** (Online Analytical Processing) |
|---|---|---|
| Query shape | Point reads/writes by key: "get user 42", "insert order" | Aggregations over huge ranges: "sum revenue by region, last 90 days" |
| Rows touched | A few rows | Millions–billions, but few *columns* |
| Latency | Single-digit ms | Seconds–minutes acceptable |
| Writes | High-frequency, small, mutating | Bulk append, rarely mutated |
| Consistency | Strong, transactional | Eventual is fine; correctness ≈ "close enough, eventually" |
| Storage | **Row-oriented** | **Column-oriented** |
| Examples | Postgres, MySQL, DynamoDB | Snowflake, BigQuery, Redshift, ClickHouse, Druid |

### Row vs columnar storage — why columnar wins for analytics

Row storage lays out all columns of a row contiguously: `(id,name,age,country)(id,name,age,country)…`.
Great for "give me the whole row for user 42" — one seek, one read.

Columnar storage lays out *one column across all rows* contiguously: all `id`s, then all `age`s, then
all `country`s. Three reasons this dominates analytics:

1. **Scan only the columns you need.** A query like `AVG(salary) GROUP BY country` reads just two
   columns out of fifty. Row storage drags every byte of every row off disk; columnar reads ~4% of
   the data. This is the single biggest win.
2. **Compression is dramatically better.** A column holds values of one type with low cardinality
   and locality — `country` is a handful of distinct strings, `timestamp` is monotonic. Run-length
   encoding, dictionary encoding, delta encoding, and bit-packing crush these. 10× compression is
   routine; less data off disk = faster scans (compounding the win above).
3. **Vectorized execution.** Tight columnar arrays let the engine process values in CPU-friendly
   batches (SIMD), instead of pointer-chasing row by row.

The cost: writing a single row touches every column file, and point lookups are slow. That's exactly
why **you don't run analytics on your serving DB** — the storage layout that's optimal for one is
pessimal for the other.

### Why you physically separate OLTP and OLAP

> **Say this:** "I won't run analytical queries against the production OLTP database. A heavy
> `GROUP BY` scan will trash its buffer cache, hold locks, and contend for IO with latency-sensitive
> user traffic. I'll replicate data out to an analytics store via CDC or a pipeline and query there."

- **Isolation** — a runaway analyst query can't take down checkout.
- **Right tool** — columnar store answers the aggregation 100× faster anyway.
- **Independent scaling** — the warehouse scales storage/compute separately from serving.

---

## Part B — Where the data lives: warehouse vs lake vs lakehouse

| | **Data Warehouse** | **Data Lake** | **Lakehouse** |
|---|---|---|---|
| Stores | Structured, modeled tables | Raw files, any format | Files + a table/transaction layer on top |
| Schema | **Schema-on-write** (defined up front) | **Schema-on-read** (interpret at query) | Schema-on-read with managed tables |
| Storage | Proprietary columnar (often) | Object store (S3/GCS) | Object store + open formats |
| Strength | Fast BI, governance, SQL | Cheap, flexible, ML/raw data | Both: cheap storage + warehouse semantics |
| Weakness | Expensive, rigid, ingest friction | "Data swamp", no transactions/quality | Younger tooling, more moving parts |
| Examples | Snowflake, BigQuery, Redshift | S3 + Parquet, HDFS | Databricks/Delta Lake, Iceberg, Hudi |

The **lakehouse** is the modern convergence: keep data cheaply in an object store in open columnar
formats, but add a metadata/transaction layer (Delta Lake, Apache Iceberg, Hudi) that gives you
ACID-ish table semantics, time travel, schema evolution, and `UPDATE`/`DELETE` over immutable files.

### ETL vs ELT

- **ETL** (Extract → **Transform** → Load): transform *before* loading into the warehouse. Classic,
  when storage/compute were expensive and the warehouse was precious. You curate before it lands.
- **ELT** (Extract → Load → **Transform**): dump raw data into cheap object storage *first*, then
  transform inside the powerful warehouse/engine (often with dbt-style SQL). The modern default —
  because object storage is cheap and engines are elastic, so "load everything raw, transform later
  and re-transform freely" wins. You keep the raw source of truth and can rebuild derived tables.

> **One-liner:** "I'd do ELT — land raw events in S3 as Parquet, then transform with the query engine.
> Cheap to store, and I can reprocess when logic changes without re-ingesting."

### The modern stack, conceptually

It's deliberately **decoupled storage and compute**:

```
Object store (S3/GCS)  +  Columnar file format (Parquet/ORC)  +  Table format (Iceberg/Delta)
        + Query/compute engine (Spark, Trino/Presto, Snowflake, Druid) on top, scaled independently
```

Parquet is the lingua franca: columnar, compressed, self-describing (embedded schema + stats per
row-group so engines can skip blocks). Decoupling means you can point five different engines at the
same files and scale compute up/down without moving data.

---

## Part C — Batch processing

Batch = process a large, **bounded** dataset, optimizing throughput over latency. Minutes to hours.

### MapReduce — the model, and why it's slow

The original Google/Hadoop model. Three phases:

1. **Map** — each worker reads an input split and emits `(key, value)` pairs. (e.g. word → 1)
2. **Shuffle** — the framework groups all values by key and sends them across the network so each key
   lands on one reducer. **This is the expensive part.**
3. **Reduce** — each reducer aggregates the values for its keys. (e.g. sum the 1s → word count)

Why it's slow:

- **The shuffle is a network + disk bottleneck.** Mappers write intermediate results to local disk,
  sort them, and reducers pull them over the network. Massive IO.
- **Materializes to disk between every stage.** A multi-step job is a *chain* of MapReduce jobs, each
  writing its full output to HDFS and reading it back. An iterative algorithm (ML, graph) pays this
  disk round-trip on every iteration.
- **Stiff programming model** — everything must be bent into map/reduce.

### Spark — RDD/DAG, in-memory, why it beats MapReduce

Spark keeps the cluster-computing idea but fixes the IO problem.

- **RDD** (Resilient Distributed Dataset) — a partitioned, immutable collection. You build new RDDs
  from old ones via **transformations** (`map`, `filter`, `join` — lazy) and trigger work with
  **actions** (`count`, `collect`). DataFrames/Datasets are the typed, optimized API on top.
- **DAG of stages** — Spark builds a directed acyclic graph of the whole computation *before*
  running, and the Catalyst optimizer plans it as a unit. It pipelines narrow transformations and
  only shuffles at wide ones — instead of N independent MapReduce jobs each hitting disk.
- **In-memory** — intermediate results stay in RAM across stages and iterations. For iterative
  workloads this is the 10–100× win. You can explicitly `cache()` a reused dataset.
- **Resilience without replication** — "Resilient" = if a partition is lost, Spark recomputes it from
  **lineage** (the recorded chain of transformations) rather than from a replicated copy.

> **In the room:** "Spark beats MapReduce because it builds a DAG and keeps intermediate data in
> memory instead of materializing to disk between every stage. MapReduce pays a disk + shuffle
> round-trip per step; Spark pipelines stages and only spills when it has to."

### When batch is the right tool

- Latency tolerance is hours, not seconds (nightly reports, billing, training data prep).
- You need to reprocess **all historical data** (backfills, recomputing a metric with new logic).
- Correctness/completeness matters more than freshness (you have the *complete* bounded dataset).
- Heavy joins/aggregations across the full corpus.

---

## Part D — Stream processing

Stream = process an **unbounded** dataset record-by-record (or micro-batch) as it arrives,
optimizing for **latency** (sub-second to seconds).

### The core problem: event time vs processing time

- **Event time** — when the event actually happened (timestamp on the event, e.g. the click at 12:00:01).
- **Processing time** — when your system *saw* it.

These diverge because of network delays, mobile devices that were offline, retries, and partition
backlog. **Almost every hard stream problem is an event-time problem.** If you aggregate "clicks per
minute" by processing time, a phone that buffered events offline for an hour lands them all in the
wrong minute. You almost always want **event-time** semantics — which forces the next two concepts.

### Watermarks

A watermark is the engine's assertion: **"I believe I've now seen all events with event time ≤ T."**
It's a heuristic about lateness. When the watermark passes the end of a window, that window's result
is emitted. Tradeoff you must name:

- **Aggressive watermark** (assume little lateness) → low latency, but late events get dropped or
  need correction.
- **Conservative watermark** (allow lots of lateness) → accurate, but you hold windows open longer →
  higher latency and more state.
- **Allowed lateness / side outputs** — keep a window's state a bit past the watermark to fold in
  stragglers, or route too-late events to a dead-letter stream for batch correction.

### Windowing

You can't aggregate an infinite stream forever; you slice it into windows.

| Window | Definition | Example use |
|---|---|---|
| **Tumbling** | Fixed, non-overlapping, contiguous | "clicks per 1-minute bucket" |
| **Sliding** | Fixed size, overlapping, advances by a step | "avg over last 5 min, emitted every 1 min" |
| **Session** | Dynamic, closes after a gap of inactivity | "group a user's activity until 30 min idle" |

### Exactly-once via checkpointing

"Exactly-once" in streaming means **exactly-once *effect on state/output*** — not that each message is
physically delivered once. Mechanism:

- **Checkpointing** — the engine periodically snapshots all operator state plus the source offsets
  (e.g. Kafka offsets) into durable storage, using a consistent barrier (Flink's Chandy–Lamport-style
  distributed snapshot). On failure it restores the last checkpoint *and* rewinds the source to the
  matching offsets, so reprocessing resumes from a consistent point.
- **Idempotent or transactional sinks** — to make the *output* exactly-once you also need the sink to
  participate: idempotent writes (upsert by key) or a two-phase commit (Kafka transactions). Without
  this, you get effectively-once for state but at-least-once at the output.

### Flink vs Spark Streaming vs Kafka Streams

| | **Flink** | **Spark Structured Streaming** | **Kafka Streams** |
|---|---|---|---|
| Model | True per-event streaming | Micro-batch (continuous mode exists) | Per-event, embedded library |
| Latency | Lowest (ms) | Higher (batch interval) | Low |
| Deployment | Standalone cluster | On Spark cluster | **Just a library in your app** — no cluster |
| Event time / watermarks | First-class, richest | Supported | Supported |
| State | Large managed state, RocksDB-backed | Supported | Local state, backed by Kafka topics |
| Best when | Low-latency, complex event-time logic, big state | Already on Spark; unify batch + stream | Kafka-centric, lightweight per-service stream logic |

> **One-liner:** "Flink if I need true low-latency event-time processing with big state; Spark
> Structured Streaming if I'm already a Spark shop and want one engine for batch and stream; Kafka
> Streams if it's a lightweight transform inside a service that already lives on Kafka."

---

## Part E — Lambda vs Kappa architecture

Both answer: *how do I serve both fresh (low-latency) and accurate (complete) results?*

### Lambda — batch layer + speed layer

```
                ┌── Batch layer  (Spark over all history) ── batch view ──┐
ingest (Kafka) ─┤                                                          ├─ serving (merge) → query
                └── Speed layer (stream, last few min)   ── realtime view ─┘
```

- **Batch layer** recomputes accurate views from the complete dataset (slow, periodic).
- **Speed layer** computes approximate, low-latency views over recent data only.
- **Serving layer** merges both: serve realtime for the recent gap, batch for everything older. The
  batch view eventually overwrites/corrects the approximate realtime view.
- **Cost:** you maintain **two codebases** that must produce consistent logic. This duplication and
  reconciliation is the well-known pain of Lambda.

### Kappa — stream-only, with replay

```
ingest (immutable Kafka log, long retention) ── stream job ── views → query
                                  (to "recompute", replay the log from offset 0)
```

- **One processing path.** Treat the input as an immutable, replayable log. There's no separate batch
  layer; to backfill or fix logic you **replay the log** through a new version of the stream job.
- Requires durable, long-retention, replayable storage (Kafka / object store) and a stream engine
  that can also chew through history fast (Flink).
- **Cost:** replaying very large histories can be slow/expensive, and not every aggregation is cheap
  to express purely as a stream.

| | **Lambda** | **Kappa** |
|---|---|---|
| Code paths | Two (batch + stream) | One (stream, with replay) |
| Reprocessing | Re-run batch job | Replay the log |
| Complexity | High (reconcile two views) | Lower code, needs strong log + fast stream engine |
| When | Heavy historical recompute differs fundamentally from realtime; legacy batch exists | Greenfield, log-centric, logic expressible as a stream |

> **Say this:** "Modern default is Kappa — one stream pipeline plus a replayable log — because
> maintaining two implementations of the same logic in Lambda is a constant source of skew. I'd reach
> for Lambda only if the batch computation is genuinely different from or far cheaper than streaming
> over all history."

---

## Part F — Designing analytics & metrics systems

The recurring hard parts and their canonical answers.

### Real-time counting at scale

Two problems show up constantly, and naive solutions blow up your memory/storage:

**1. Count-distinct (cardinality)** — "how many *unique* users viewed this video?" Exact counting
needs a set of every ID seen → unbounded memory per counter, and you have millions of counters.

→ **HyperLogLog (HLL).** Estimates cardinality in ~1.5 KB with ~2% error, regardless of whether the
true count is thousands or billions. Crucially, **HLL sketches are mergeable** — union two sketches to
get the distinct count of the union. That's what makes per-shard, per-time-bucket, roll-up-able
unique counts feasible. (Redis `PFADD`/`PFCOUNT` is HLL.)

**2. Top-K / heavy hitters** — "what are the trending hashtags / top 10 most-clicked ads right now?"
Exact counts of every key need a huge hash map.

→ **Count-Min Sketch** for the frequency estimates (fixed memory, overestimates only), paired with a
**min-heap of size K** to track the current top-K. Add the CMS estimate; if it beats the heap minimum,
swap it in. Approximate but bounded memory, and CMS sketches are mergeable too.

### Pre-aggregation / rollups

Don't store and scan raw events to answer "views per hour." **Pre-aggregate** into rollup tables at
multiple granularities as data arrives: per-minute → per-hour → per-day. Queries hit the coarsest
table that satisfies them.

- Trades storage (you keep aggregates) and write-time work for *enormous* read speedup.
- Layer it: keep raw for a short window, minute-rollups longer, day-rollups forever.
- This is the bread and butter of time-series and metrics systems.

### Approximate vs exact — pick deliberately

> **Say this:** "For a dashboard of trending content, 2% error on unique counts is invisible to the
> user and saves orders of magnitude of memory — I'll use sketches. For billing or financial counts I
> need exact, so I'll do exact counting in batch and accept the latency."

The staff move is matching precision to the business need: **sketches for dashboards/trending, exact
batch for money.** Often you do both — fast approximate in the speed layer, exact in batch that
corrects later (this is literally why Lambda exists).

### Time-series storage

Metrics are time-series: append-heavy, time-ordered, queried by range + aggregation, downsampled over
time. Use a TSDB (Druid, InfluxDB, Prometheus, TimescaleDB, ClickHouse). Key properties:

- Partition by time → cheap range scans and easy expiry (drop old partitions).
- Columnar + delta/RLE compression on timestamps and values.
- **Retention + downsampling** policies: keep raw for days, 1-min rollups for weeks, 1-hour forever.

---

## Part G — Designing a data pipeline end to end

```
INGESTION            PROCESSING                    SERVING
 ├ CDC from OLTP ─┐                              ┌─ OLAP store (Druid/ClickHouse)
 ├ app events ────┼─► Kafka ─► stream/batch ─────┼─ rollup tables / sketches
 └ logs ──────────┘            (Flink/Spark)     └─ cache for hot queries → dashboard/API
```

### Ingestion

- **CDC (Change Data Capture)** — stream the OLTP database's commit log (Debezium reading the WAL/binlog)
  into Kafka. The clean way to get OLTP changes into analytics *without* dual-writes or hammering the
  source DB with polling.
- **Queue/log (Kafka)** as the ingestion buffer — decouples producers from processing, absorbs spikes,
  and (long retention) becomes your replay log for Kappa/backfills.

### Idempotency and reprocessing

- Pipelines are **at-least-once** by default → **design every stage idempotent.** Give events stable
  IDs; make sink writes upserts keyed by `(entity, time_bucket)` so a replayed event overwrites rather
  than double-counts.
- This is what makes **reprocessing** safe: re-running a window produces the same result, not doubled
  metrics.

### Backfills

- Need to populate history or recompute a metric with fixed logic. With a replayable log (Kappa) or
  raw data in the lake (ELT), run the new job over the old data.
- Watch for: backfill traffic competing with live traffic (isolate resources), and **idempotent sinks
  so the backfill overwrites cleanly** instead of double-writing.

### Schema evolution

Events change shape over years; old and new must coexist.

- Use a **schema registry** with a serialization format that versions schemas (Avro, Protobuf).
- Enforce **compatibility rules**: backward-compatible (new consumer reads old data) and/or
  forward-compatible (old consumer reads new data). Practically: add optional fields with defaults;
  don't rename/retype/remove required fields.
- Parquet/Iceberg carry schema with the data and support column add/drop over time.

### Data quality

- **Validation at ingest** — reject or quarantine malformed events to a dead-letter queue.
- **Assertions on outputs** — row counts, null rates, distribution drift, "today within X% of
  yesterday" (Great Expectations / dbt tests).
- **Dead-letter queues** for poison messages so one bad record doesn't wedge the pipeline.
- **Lineage + monitoring** — freshness SLAs ("data is at most N minutes stale") and alerting.

---

## Part H — Worked mini-walkthrough: "Design ad-click aggregation / trending top-K"

This is the canonical interview. Here's the spine to walk.

**1. Requirements.** Functional: ingest click events; serve (a) per-ad click counts over time windows,
(b) the current top-K trending ads/hashtags, (c) unique users per ad. Non-functional: ~1M clicks/sec
peak, dashboard latency seconds, dashboards tolerate ~1–2% error, **but advertiser billing must be
exact.** Read-heavy on aggregates, write-heavy on ingest.

**2. Estimate.** 1M/sec × 100 bytes ≈ 100 MB/sec ingest ≈ 8.6 TB/day raw. Way past one DB → log +
stream + columnar store, not OLTP.

**3. Pipeline.**
```
clients → Kafka (partitioned by adId, long retention) → Flink (event-time, tumbling 1-min windows,
          watermarks for late mobile clicks) → { rollup tables, HLL sketches, Count-Min + heap }
          → Druid/ClickHouse + cache → dashboard API
```

**4. The aggregations — this is where you score:**

- **Counts per window:** tumbling 1-minute windows keyed by `adId`, written as **upserts keyed by
  `(adId, minute)`** → idempotent, so reprocessing/late events correct in place. Roll minutes up to
  hours/days.
- **Unique users per ad:** maintain a **HyperLogLog per `(adId, window)`**. Mergeable, so per-shard
  and per-minute sketches union into per-hour/day unique counts. ~1.5 KB each vs storing every user ID.
- **Top-K trending:** **Count-Min Sketch** for click frequency estimates + a **min-heap of size K** for
  the current leaders. Cheap memory, mergeable across shards.

**5. Exact vs approximate, stated explicitly:** dashboards and trending run off the **stream/speed
layer** with sketches (approximate, instant). **Billing runs a nightly Spark batch** over the raw
Kafka/lake data for exact, auditable counts. That's a **Lambda** split justified by the requirement
("dashboards tolerate error, money doesn't") — or **Kappa** if I can get exact counts by replaying the
log and skip the second codebase.

**6. Event-time correctness:** clicks from offline phones arrive late → **event-time windows +
watermarks + allowed lateness**, with too-late events routed to a side output that the batch layer
reconciles.

**7. Failure modes:** Kafka retention = replay/backfill; Flink **checkpointing + Kafka offset rewind**
for exactly-once state; idempotent upsert sinks so a restart doesn't double-count; dead-letter queue
for malformed events.

> **The whole answer in one breath:** "Kafka to absorb and replay, Flink for event-time windowed
> aggregation, HyperLogLog for uniques and Count-Min + heap for top-K to keep memory bounded, rollup
> tables in a columnar store for fast dashboard reads, and a nightly exact batch job for billing
> because money can't be approximate."

---

## Part I — Probabilistic data structures recap

The staff toolkit for "exact is too expensive." All trade a small, bounded error for huge space
savings, and all are **mergeable** (you can union sketches computed in parallel/per-shard).

| Structure | Answers | Error mode | Space | Use in interviews |
|---|---|---|---|---|
| **Bloom filter** | "Is X *possibly* present / *definitely* not?" (set membership) | False positives; **never** false negatives. No deletes (use counting Bloom) | ~bits per element | "Have we seen this URL?", skip disk lookups, dedupe |
| **HyperLogLog** | "How many *distinct* elements?" (cardinality) | ~1–2% relative error | ~1.5 KB for billions | Unique visitors, distinct counts |
| **Count-Min Sketch** | "How many times has X occurred?" (frequency) | **Over**estimates only, bounded by ε | Few KB (width × depth) | Top-K / heavy hitters, rate limiting |

> **Trigger phrases → reach for these:** "unique / distinct count at scale" → HyperLogLog. "most
> frequent / trending / top-K / heavy hitters" → Count-Min Sketch + heap. "have we seen this before /
> avoid a lookup" → Bloom filter.

---

## Part J — How to use this in the room

1. **Name the workload first.** "This is OLAP — I'm keeping it off the serving DB." Instant signal.
2. **Reach for the stack:** log (Kafka) for ingest + replay, stream engine (Flink) for fresh,
   batch (Spark) for exact, columnar store for serving, sketches for bounded memory.
3. **Always declare approximate vs exact** and tie it to a requirement (dashboards vs money).
4. **Make every stage idempotent** and say *why* (at-least-once → safe reprocessing/backfills).
5. **Default to Kappa**, justify Lambda only when batch is genuinely different.
6. **Event time, not processing time** — and say "watermarks" for lateness.

> **Mental checklist for any analytics prompt:**
> Workload OLAP? → Columnar store, off the serving DB → Stream + batch split? → Approximate (sketches)
> or exact (batch)? → Event-time windows + watermarks → Pre-aggregate/rollups → Idempotent sinks for
> reprocessing → What's the replay/backfill story?

---

### Self-check before the mock (answer these from memory)
- [ ] Give three reasons columnar storage beats row storage for analytics.
- [ ] Why don't you run analytical queries against the production OLTP database?
- [ ] Warehouse vs lake vs lakehouse in one sentence each; ETL vs ELT and why ELT is the modern default.
- [ ] Why is Spark faster than MapReduce? (DAG + in-memory + lineage)
- [ ] Event time vs processing time, and what a watermark is.
- [ ] Name the three window types and one use of each.
- [ ] How does exactly-once work in a stream engine? (checkpoint + offset rewind + idempotent sink)
- [ ] Lambda vs Kappa: the tradeoff and when you'd pick each.
- [ ] Which sketch for distinct-count, which for top-K, and the error tradeoff of each.
- [ ] Walk the ad-click aggregation design end to end in under two minutes.
