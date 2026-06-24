# Topic 7: Message Queues & Event Streaming Deep Dive

> **Why this topic matters:** Almost every non-trivial design has an async path. The moment you say
> "I'll enqueue that and process it in a worker," the interviewer's next five questions are about
> *delivery semantics, ordering, idempotency, and what happens when the consumer falls behind.* If you
> can only say "I'll use Kafka," you fail the deep dive. If you can say *why a log beats a queue here,
> what delivery guarantee you're buying, and how you'll dedup on the consumer*, you sound like someone
> who's run this in production. This doc is the recombination of one building block — the queue/log —
> into all the answers that follow from it.

---

## Part A — Why async at all

Synchronous request/response is the default and you should keep it until you can name a *specific*
reason to break it. Here are the reasons, each a thing to say out loud:

- **Spike absorption (load leveling).** Producers burst to 50k/s; your DB sustains 5k/s. A queue is a
  shock absorber — it accepts the burst at memory/disk speed and lets consumers drain at their own
  pace. The alternative is shedding load or toppling the DB.
- **Temporal decoupling.** Producer and consumer don't have to be up at the same time. The consumer
  can be down for an hour, come back, and drain the backlog. Sync coupling means the producer fails
  when the consumer fails.
- **Buffering / smoothing.** Convert a spiky arrival pattern into a steady processing rate. This is
  what protects downstream systems with hard rate ceilings (a payment processor, a third-party API).
- **Fan-out.** One event, many independent consumers (email service, analytics, search indexer, audit
  log) — each subscribes without the producer knowing they exist. New consumer = zero producer change.
- **Retries / durability.** The message survives a consumer crash. Sync work that dies mid-flight is
  just gone; queued work is redelivered.
- **Latency hiding.** Return `202 Accepted` to the user in 10ms; do the slow work (transcode the
  video, generate the thumbnail, send the email) off the hot path.

> **Say this in the room:** "I'll go async here specifically to absorb the write spike and decouple
> the indexer from the write path — not just because queues are 'scalable.'" Naming the *reason*
> separates you from candidates who sprinkle Kafka on everything.

### The new problems async introduces (name these before the interviewer does)

Async is not free. Every benefit above buys you a new class of bug:

| You gain | You now owe |
|---|---|
| Decoupling | Eventual consistency — the rest of the system is briefly wrong |
| Durability + retries | **At-least-once delivery → duplicates → you must be idempotent** |
| Buffering | Consumer lag, unbounded backlogs, backpressure to manage |
| Fan-out | Harder tracing/debugging; "where did my event go?" |
| Smoothing | Added end-to-end latency; harder to reason about ordering |
| A new component | Operational burden, another thing that can fail, poison messages |

If you can't tolerate eventual consistency or duplicates, either don't go async or pay for the
machinery (idempotency keys, dedup tables, transactions) that hides it. Most rounds turn on this.

---

## Part B — The core distinction: queue vs log

This is the single most important mental model in this topic. Interviewers conflate them; you should
not. Almost everything else follows from getting this right.

### Traditional message queue (SQS, RabbitMQ, ActiveMQ)

- A message is **delivered, acked, and then deleted**. The queue is a *transient* holding area.
- **Competing consumers**: N workers pull from one queue; each message goes to *one* of them. This is
  how you scale work horizontally — add workers, throughput goes up, no coordination.
- The broker tracks per-message state (in-flight, acked, visibility timeout). It's doing real
  bookkeeping per message.
- **No replay.** Once acked, it's gone. If consumer logic had a bug, the data is unrecoverable.
- Great for **task/job distribution**: "process this payment," "send this email," "resize this image."

### Distributed log (Kafka, Pulsar, Kinesis, Redpanda)

- An **append-only, ordered, immutable log** partitioned across nodes. Messages are **retained** for a
  configured time/size regardless of consumption.
- Consumers track their own **offset** (a cursor into the log). Reading does *not* delete. Multiple
  independent consumer groups read the same data at their own offsets.
- **Replayable**: reset the offset to reprocess history — bootstrap a new service, fix a bug and
  re-run, rebuild a downstream store.
- The broker does almost no per-message bookkeeping — it just appends and serves byte ranges. This is
  *why* it's so fast (Part C).
- Great for **event streaming, event sourcing, multi-consumer fan-out, and anything you might want to
  reprocess.**

### The one-paragraph version to say out loud

> "A queue is a *to-do list* — you take an item, do it, and cross it off; once it's done it's gone, and
> many workers share one list. A log is a *ledger* — an immutable ordered history that every reader
> scans at its own position and nothing gets crossed off. If I need competing workers draining tasks,
> I want a queue. If I need replay, multiple independent consumers, or ordered event history, I want a
> log."

### When each fits

| Use a **queue** when… | Use a **log** when… |
|---|---|
| Work distribution to a worker pool | Multiple independent consumers of the same stream |
| Each message handled once, then discarded | You may need to replay / reprocess history |
| Per-message ordering doesn't matter much | You need ordering within a key |
| You want trivial competing-consumer scaling | You're doing event sourcing / CDC / analytics pipelines |
| Variable per-message visibility/delay (e.g. SQS) | Very high throughput, sequential streaming |

Note Kafka can *emulate* a queue (one consumer group, competing consumers within it) and RabbitMQ can
*emulate* fan-out (exchanges), so the lines blur. But pick based on the dominant access pattern.

---

## Part C — Kafka deep dive

Kafka is the default "log" answer and the one you'll be drilled on. Know it cold.

### The data model

- **Topic** — a named stream of records ("orders", "clicks").
- **Partition** — a topic is split into N partitions; each partition is an *independent ordered log*.
  Partitions are the unit of parallelism, ordering, and storage distribution.
- **Offset** — a monotonically increasing integer ID of a record *within a partition*. A consumer's
  position is `(topic, partition, offset)`. Offsets are committed back to Kafka (the
  `__consumer_offsets` topic) so a restarted consumer resumes where it left off.
- **Record** — key, value, timestamp, headers. The **key** decides the partition.

### Consumer groups (the scaling primitive)

- A **consumer group** is a set of consumers that collectively consume a topic. Kafka assigns each
  partition to **exactly one consumer** in the group.
- Therefore **max parallelism = number of partitions.** 10 partitions → at most 10 useful consumers in
  a group; an 11th sits idle. This is the **partition-count ceiling** and a favorite interview gotcha.
- Different consumer groups are independent — group A and group B both read every record, each tracking
  its own offsets. That's how you get fan-out (Part H).
- **Rebalancing**: when a consumer joins/leaves/dies, partitions are reassigned. Rebalances pause
  consumption (the "stop-the-world" problem); cooperative/incremental rebalancing reduces this.

### Ordering guarantee — read this twice

> **Kafka guarantees ordering ONLY within a single partition. There is NO global ordering across a
> topic.** Everything about ordering in your design reduces to: "are the records that must be ordered
> all in the same partition?"

Records with the same key hash to the same partition, so **same key → same partition → ordered**.
This is the whole game (see Part F).

### Partition key choice — the decision that makes or breaks the design

The key determines partition via `hash(key) % numPartitions`. Choosing it controls three things at
once: **ordering, even load distribution, and hot-spotting.**

- **Pick a key that groups what must stay ordered.** Order events for one `orderId` → key on `orderId`,
  so they land in one partition and stay ordered.
- **Beware hot partitions / skew.** Keying on a low-cardinality or skewed field (e.g. `country`, or a
  single celebrity `userId`) overloads one partition. A skewed key defeats the whole point of
  partitioning.
- **`null` key → round-robin** across partitions (max spread, no ordering). Use when you don't need
  ordering at all.
- Tradeoff to state: "I key on `userId` to keep per-user events ordered, accepting that a power user
  could hot-spot one partition; if that's a real risk I'd add a sub-key or split that user out."
- **Repartitioning is painful**: `hash % N` changes meaning if N changes, so adding partitions breaks
  existing key→partition mapping. Over-provision partitions up front; you can't easily shrink ordering
  guarantees later.

### Replication: leaders, followers, ISR

- Each partition has **R replicas**; one is the **leader** (handles all reads/writes), the rest are
  **followers** that pull and replicate.
- **ISR (In-Sync Replicas)** — the set of replicas caught up to the leader. A leader can only fail over
  to an ISR member, so ISR is the durability frontier.
- Producer durability is controlled by **`acks`**:
  - `acks=0` — fire and forget, can lose data (fastest).
  - `acks=1` — leader persists, acks before followers replicate; lose data if leader dies before
    followers catch up.
  - `acks=all` — leader waits for all ISR to replicate. Combined with `min.insync.replicas=2`, this is
    the durable setting. Tradeoff: higher latency.
- **`min.insync.replicas`**: if ISR shrinks below this, the partition rejects writes — choosing
  consistency/durability over availability (a CAP decision you can name explicitly).

### Retention and log compaction

- **Time/size retention**: keep records for 7 days, or until the log hits X GB, then delete oldest
  segments. The default model.
- **Log compaction**: keep *at least the latest value per key*, discard superseded older values.
  Produces a "latest-state-per-key" log — ideal for changelogs, CDC, and rebuilding a key-value store
  from the topic. A compacted topic is effectively a durable, replayable snapshot of current state.

### Why Kafka is fast (the "why" they want)

- **Sequential disk IO + append-only.** Writes are appends to a segment file; reads are sequential
  scans. Sequential disk throughput is close to memory-bandwidth and orders of magnitude faster than
  random IO. No per-message data structure to update.
- **Page cache, not a JVM heap cache.** Kafka writes to the OS page cache and lets the OS flush to
  disk. Reads of recent data are served from page cache (RAM) without a userspace copy.
- **Zero-copy (`sendfile`)** sends bytes straight from page cache to the network socket, skipping the
  copy into application memory. The broker barely touches the payload.
- **Batching + compression.** Producers batch records and compress the batch; the broker stores and
  ships the compressed batch as-is. Amortizes per-message overhead.
- **Dumb broker, smart clients.** Consumers track their own offsets; the broker doesn't manage
  per-consumer per-message state. Less bookkeeping = more throughput.

Net: a single well-tuned cluster does **millions of messages/sec / GB/sec**. The throughput ceiling is
usually network/disk, not the broker logic.

---

## Part D — Delivery semantics

Memorize the three and exactly what each means. This is the most-asked sub-question in the topic.

| Semantic | Meaning | How you get it | Cost / risk |
|---|---|---|---|
| **At-most-once** | 0 or 1 delivery; may lose | Ack/commit offset *before* processing; `acks=0` | Data loss on crash. Fine for metrics/telemetry |
| **At-least-once** | 1 or more; never loses, may dup | Ack/commit *after* processing; retry on failure | **Duplicates** → you must be idempotent. The common default |
| **Exactly-once** | Effectively once | Idempotency + transactions | Complex, narrower scope than people think |

### Why "exactly-once" is really "effectively once"

True exactly-once delivery over an unreliable network is impossible (the classic Two Generals
problem) — you can never be sure your ack arrived, so you must be willing to either retry (dup) or
not (loss). What systems actually deliver is **effectively-once**: messages may be *delivered* more
than once, but the *effect* happens once, achieved by **idempotency + atomic state updates.**

- **Kafka idempotent producer** (`enable.idempotence=true`): the producer tags records with a producer
  ID + sequence number; the broker dedups retried produces, so a producer retry doesn't create a
  duplicate *in the log*. Solves producer-side dups only.
- **Kafka transactions**: atomically write to multiple partitions *and* commit consumer offsets in one
  transaction → the classic **consume-process-produce** loop becomes exactly-once *within Kafka*
  (`read_committed` isolation on the consumer hides uncommitted records).
- **The catch:** Kafka EOS only covers Kafka→Kafka. The moment your consumer writes to an *external*
  system (a DB, a third-party API), you're back to needing **consumer-side idempotency.** Kafka can't
  make your Postgres write exactly-once.

> **Say this in the room:** "Exactly-once isn't a delivery guarantee you toggle on — it's
> at-least-once delivery plus idempotent consumers. I'll design for at-least-once and make the
> consumer idempotent; that's what actually holds when I write to an external store."

---

## Part E — Idempotency and dedup in consumers

Because at-least-once is the realistic default, **the consumer must produce the same result whether it
sees a message once or five times.** Three tools:

- **Naturally idempotent operations.** Prefer them. `SET balance = 100` is idempotent; `balance =
  balance + 10` is not. `UPSERT user(id, …)` is idempotent; blind `INSERT` is not. Design the write to
  be a set/upsert keyed by a stable ID and most of the problem evaporates.
- **Idempotency keys.** Producer stamps each message with a stable unique ID (e.g. `orderId`, a UUID,
  or `topic-partition-offset`). The consumer checks "have I processed this ID?" before acting.
- **Dedup table.** Persist processed keys (often with a TTL) and do the business write + the dedup-key
  insert **in one transaction** (`INSERT … ON CONFLICT DO NOTHING` on a unique key). If the key already
  exists, skip. This makes "process + record-as-done" atomic, so a redelivery after a crash is a no-op.

```
on message m:
  BEGIN
    if exists(processed where id = m.id): COMMIT; ack; return   -- already done
    apply business effect(m)
    insert processed(id = m.id)
  COMMIT
  ack
```

- Watch the **commit-vs-side-effect ordering**: if you ack before the DB commit, a crash loses the
  work (at-most-once); if you commit the side effect and crash before ack, you'll reprocess (the dedup
  table catches it). Always make the side effect + dedup atomic, and ack last.
- Dedup state isn't free — bound it with a TTL or a Bloom filter for "definitely not seen" fast paths.

---

## Part F — Ordering and its throughput cost

Ordering and parallelism are in direct tension. This is a tradeoff you must state crisply.

- **To preserve order, related records must go through a single partition** (Kafka) or a single FIFO
  group (SQS FIFO). Same key → same partition → ordered.
- **Total ordering across a topic ⇒ a single partition ⇒ a single consumer ⇒ no parallelism.** A
  globally ordered stream cannot be parallelized. That's the cost, stated plainly.
- **The right granularity is per-key ordering, not global.** You almost never need global order; you
  need "events for *this* entity in order." Key on the entity (`orderId`, `accountId`) and you get
  per-entity ordering *and* parallelism across entities. This is the staff move.
- **SQS standard is best-effort order (can reorder); SQS FIFO** preserves order within a
  `MessageGroupId` and dedups, but caps throughput (historically ~300 msg/s/group without batching).
  Same tradeoff, different knob.
- Beware in-consumer concurrency: if one consumer pulls an ordered partition but hands records to a
  thread pool, you've thrown away the ordering. Process a partition single-threaded (or hash within the
  consumer by key) to keep it.

> **The line:** "I don't need global ordering, I need per-`orderId` ordering. I'll partition by
> `orderId`, which gives me order where it matters and full parallelism across orders."

---

## Part G — Backpressure, consumer lag, and slow consumers

### Consumer lag

**Lag = latest offset (log head) − consumer's committed offset**, per partition. It's *the* health
metric for a streaming consumer. Growing lag means consumers can't keep up; the backlog (and
end-to-end latency) grows unbounded until something gives. Alert on lag and lag *rate*.

### How to handle slow consumers

- **Scale out consumers — up to the partition ceiling.** More consumers in the group, more parallel
  partitions drained. But you can't exceed partition count, so **provision partitions for your peak
  consumer count** from the start.
- **Speed up per-message work**: batch DB writes, async-IO downstream calls, drop unnecessary work.
- **Parallelize within a partition carefully** (hash by sub-key to worker threads) if strict
  per-partition order isn't required.
- **Shed or tier**: route slow/heavy messages to a separate "slow lane" topic so they don't block the
  fast path (head-of-line blocking).
- **Backpressure**: the log *is* the buffer — producers aren't blocked by slow consumers (unlike
  in-memory queues), but retention is finite. If lag exceeds retention, **the consumer misses data**
  (records expire before it reads them). That's the real failure mode: not a crash, but silent data
  loss. Size retention to survive your worst expected outage.
- In pull systems (Kafka, SQS) consumers control their own rate — natural backpressure. In push systems
  (some RabbitMQ setups) you need prefetch limits / flow control or you overwhelm the consumer.

### Scaling: queue vs log

- **Queue (SQS/RabbitMQ):** add competing consumers freely; near-linear scaling, no ceiling. This is a
  genuine advantage of queues for pure work distribution.
- **Log (Kafka):** scaling is gated by partition count. Repartitioning to add parallelism rehashes keys
  and disrupts ordering. So you trade easy scaling for ordering + replay.

---

## Part H — Dead letter queues, poison messages, retries

- **Poison message**: a message that *always* fails (malformed, references deleted data, triggers a
  bug). Naive at-least-once retry loops on it forever and **blocks the partition behind it**
  (head-of-line blocking) — a single bad message stalls the whole stream.
- **Retry with backoff**: transient failures (a downstream timeout) deserve retries with
  **exponential backoff + jitter** (jitter prevents synchronized retry storms). Cap the attempts.
- **Dead letter queue (DLQ)**: after N failed attempts, move the message to a separate queue/topic
  instead of retrying forever. The main stream keeps flowing; a human or a separate process inspects
  the DLQ. Always alert on DLQ depth — a filling DLQ is a real incident.
- **Retry topics pattern (Kafka)**: since Kafka has no native per-message redelivery, implement tiered
  retry topics (`orders.retry.5s`, `orders.retry.1m`, …) and finally `orders.DLQ`. A failed record is
  republished to the next retry tier with a delay.
- **Distinguish transient vs permanent failures.** Retry transient (network blip); DLQ-immediately on
  permanent (validation error). Retrying a permanent failure just wastes attempts.

---

## Part I — The dual-write problem, Outbox, and CDC

This is the highest-leverage thing in the doc for reliability deep dives. Learn it.

### The dual-write problem

You need to **update your DB and publish an event** ("order created" → save row + emit to Kafka). Doing
both as two separate operations is a **dual write**, and there's no way to make two independent systems
atomic without a distributed transaction:

- Commit DB, then crash before publishing → **DB updated, event lost.** Consumers never hear about it.
- Publish, then DB commit fails → **event emitted for a thing that doesn't exist.** Phantom event.
- Wrapping both in a 2PC is slow, fragile, and often unsupported by the broker.

> **Say this in the room:** "I won't dual-write to the DB and the broker — there's no atomicity across
> them. I'll use the Outbox pattern so the event and the state change commit together."

### The Outbox pattern

- In the **same local DB transaction** that writes the business row, also insert a row into an
  **`outbox` table** (the event payload).
- Because it's one transaction, the state change and the intent-to-publish are **atomic** — either both
  land or neither does. The dual-write is gone.
- A separate **relay/publisher** reads unpublished outbox rows and publishes them to the broker, marking
  them sent. Publishing is at-least-once (the relay can crash after publish, before marking) → consumers
  must be idempotent (Part E). That's fine — we already pay that.

### CDC (Change Data Capture) with Debezium

- Instead of polling the outbox table, **tail the database's write-ahead log / binlog** (Debezium reads
  Postgres WAL / MySQL binlog) and stream every committed change into Kafka.
- Point CDC at the outbox table (or at the business tables directly) → events are published *exactly as
  committed*, with no application polling and no dual write. CDC is the log-native way to do Outbox.
- Bonus: CDC turns any DB into an event source — feed search indexes, caches, data lakes, and replicas
  off the same committed changes. This is the standard way to keep Elasticsearch/Redis in sync.
- Tradeoff: another moving part (connector + Kafka Connect), schema-change handling, and you're now
  coupled to DB internals (WAL format).

---

## Part J — Fan-out patterns

| Pattern | Shape | Mechanism | Use |
|---|---|---|---|
| **Competing consumers** | 1 queue → N workers, each msg to **one** worker | SQS, RabbitMQ work queue, single Kafka consumer group | Work/task distribution, scale throughput |
| **Pub/sub (fan-out)** | 1 event → **every** subscriber gets a copy | RabbitMQ fanout exchange, SNS, multiple Kafka consumer groups | Independent consumers (email + analytics + index) |
| **Partitioned consumption** | Stream split by key → parallel ordered substreams | Kafka partitions, Kinesis shards | Per-key ordering + parallelism together |

- Kafka unifies these: **within** a consumer group = competing consumers; **across** groups = pub/sub;
  **partitions** give partitioned consumption. One system, three patterns.
- **SNS→SQS fan-out** is the canonical AWS combo: SNS topic fans an event to many SQS queues, each
  drained by its own competing-consumer pool. Pub/sub on top of queues.

---

## Part K — When NOT to use a queue

Reaching for a queue reflexively is a junior tell. Don't, when:

- **You need a synchronous response.** The caller needs the result *now* (a search query, a price
  lookup). A queue adds a round-trip and a polling/callback dance for nothing. Use RPC/HTTP.
- **Low-latency, tight SLA.** Queues add hops and buffering latency. On a sub-10ms path, that's a tax
  you can't pay.
- **Simple CRUD with no fan-out, no spikes, no slow work.** If the work is fast and synchronous and
  only one thing cares, just write to the DB in the request. Don't add a broker to operate, monitor,
  and reason about for nothing.
- **Strong read-after-write consistency is required.** Async means the change is briefly invisible. If
  the user must immediately see their own write, don't hide it behind a queue (or design a read path
  that doesn't depend on the async consumer).
- **The transaction must be atomic across systems.** A queue doesn't give you ACID across services;
  it gives you eventual consistency. Use Outbox/saga, and be honest that it's eventual.

> **The mature take:** "Async buys decoupling and spike absorption at the price of eventual consistency
> and duplicate handling. If I'm not buying anything from that trade here, I keep it synchronous."

---

## Part L — Comparison table (recite the shape, not every number)

| | **Kafka** | **RabbitMQ** | **SQS** | **Pulsar** | **Redis Streams** |
|---|---|---|---|---|---|
| Model | Distributed log | Broker / queue (AMQP) | Managed queue | Log + queue (segmented) | In-memory log |
| Retention / replay | Yes (offset-based) | No (deleted on ack) | No (deleted on ack; ≤14d) | Yes (tiered storage) | Yes (capped/trimmed) |
| Ordering | Per partition | Per queue (best-effort) | FIFO queues only | Per partition | Per stream |
| Delivery | At-least / EOS (Kafka↔Kafka) | At-least-once | At-least-once (Std), FIFO exact-ish | At-least / effectively-once | At-least-once |
| Throughput | Very high (M/s) | Moderate | High (auto-scales) | Very high | High (RAM-bound) |
| Routing logic | Dumb broker | **Rich** (exchanges, routing keys) | Minimal | Flexible | Minimal |
| Scaling ceiling | Partition count | Broker/cluster | Effectively unlimited (managed) | Bookies, decoupled | Single-node-ish / cluster |
| Ops burden | High (self-host) | Medium | **None (managed)** | High | Low–Medium |
| Sweet spot | Streaming, event sourcing, replay, high throughput | Complex routing, per-message TTL/priority, RPC | Simple async on AWS, zero ops | Kafka use cases + multi-tenancy + geo-replication | Lightweight streams when you already run Redis |

How to choose, in one breath each:

- **Kafka** — high-throughput event streaming, multiple consumers, replay, CDC pipelines. The default
  "log."
- **RabbitMQ** — you need smart routing, priorities, per-message TTL, or low-latency task RPC. The
  default "rich queue."
- **SQS** — you're on AWS and want a dead-simple, zero-ops async queue; pair with SNS for fan-out.
- **Pulsar** — Kafka-class streaming but you want native multi-tenancy, tiered storage, and decoupled
  compute/storage (brokers vs BookKeeper). Niche but real.
- **Redis Streams** — you already run Redis and need lightweight streaming without standing up Kafka;
  not for durability-critical, huge-volume pipelines.

---

## How to use this in the room

1. **Justify async with a specific reason** (spike, decouple, fan-out, slow work) — never "for scale."
2. **State queue vs log explicitly** and pick based on replay + multi-consumer + ordering needs.
3. **Declare your delivery semantic** ("at-least-once") and immediately say how you'll be idempotent.
4. **Get ordering granularity right**: per-key, not global; name the throughput tradeoff.
5. **Name the failure modes unprompted**: poison messages → DLQ, lag → backpressure/retention, dual
   write → Outbox/CDC.
6. **Know when to say no**: sync request/response and simple CRUD don't need a broker.

> **Mental checklist for any async deep dive:**
> Queue or log? → What delivery semantic? → How do I dedup on the consumer? → What's the partition key
> and does it hot-spot? → What's my ordering granularity and its cost? → What happens when consumers lag
> or a message is poison? → Am I dual-writing anywhere?

---

### Self-check before the mock (answer these from memory)
- [ ] Give three concrete reasons to go async, and the new problem each one creates.
- [ ] Explain queue vs log in one sentence each, and when you'd pick each.
- [ ] Why is Kafka fast? Name at least three mechanisms.
- [ ] What is Kafka's *only* ordering guarantee, and how does the partition key relate to it?
- [ ] Define the partition-count ceiling and how it limits consumer scaling.
- [ ] State the three delivery semantics and how you achieve each.
- [ ] Why is "exactly-once" really "effectively-once"? What does Kafka EOS *not* cover?
- [ ] Show the consumer dedup transaction (side effect + dedup key + ack ordering).
- [ ] What's the throughput cost of global ordering, and what's the better granularity?
- [ ] Define consumer lag and what happens when it exceeds retention.
- [ ] Explain the dual-write problem and how Outbox + CDC solves it without 2PC.
- [ ] Name two situations where you would NOT use a queue.
