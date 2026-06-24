# Design 10: Distributed Message Queue / Event Log (Kafka-style)

> **Why this one matters:** this is the *other* systems-heavy design (the KV store being the first).
> It ties together the messaging building block (queue-vs-log, delivery semantics, idempotency —
> **messaging doc 07**), the storage internals that make a log fast (sequential IO + page cache +
> zero-copy + WAL — **databases doc 03**), partitioning a stream for parallelism (**sharding doc 04**),
> leader/follower replication with ISR (**replication doc 05**), and — the part most candidates fumble —
> *how the cluster agrees on who leads each partition* via a consensus layer (**consensus doc 08**). If
> you can derive a log from "append-only file + offsets" and defend every knob, you can derive almost
> any streaming or storage system. So I run the full 7-step framework but spend most of my budget in
> **Step 6 (deep dives)** — that's where this round is won.
>
> The one-line thesis I'll state up front and keep returning to: **we are building a replicated,
> partitioned, append-only log — a "dumb broker, smart client" system whose speed comes from doing
> almost nothing per message, and whose correctness comes from a per-partition leader chosen by a
> consensus layer.** Every decision is "keep the broker simple and sequential, push bookkeeping to the
> client, and pay for coordination only where we truly need it (leader election)." Contrast that with a
> traditional queue (RabbitMQ/SQS) at the end — that's the cleanest way to prove I know *why* the log is
> shaped the way it is.

---

## Step 1 — Requirements (5 min) — I drive this

"Design a message queue" is unbounded, so I scope hard and position the system before drawing anything.
First the **queue-vs-log fork** (**messaging doc 07, Part B**): a *queue* delivers-acks-deletes with
competing consumers and no replay; a *log* is an immutable, ordered, retained history that many
independent consumers scan at their own offset. The requirements below — replayability, multiple
independent consumers, ordering within a key, very high throughput — point squarely at a **log**
(Kafka), not a transient queue. I'll state that explicitly and build a log.

**Functional**
- `produce(topic, key, value)` → append a record to a partition, return its offset.
- `consume(topic, group, …)` → stream records from the consumer's current offset forward.
- `commitOffset(group, topic, partition, offset)` → record consumer progress so a restart resumes.
- Topics are partitioned; ordering is preserved **within a partition**.
- Records are **retained** (time/size) regardless of consumption → **replay** by resetting offset.
- **Multiple independent consumer groups** read the same topic, each at its own offset (fan-out).
- **Explicit non-goals I'll call out:** no per-message priority, no arbitrary delay/visibility-timeout
  semantics (that's SQS/RabbitMQ territory — **doc 07, Part L**), no rich routing/exchanges, no global
  total ordering across a topic, no message mutation/deletion by ID (it's an append-only log, not a
  database). If they push on those, I'll note them as a queue's strengths in the wrap-up contrast.

**Non-functional** — this is where the design is actually decided:
- **High throughput** — millions of messages/sec, GB/s aggregate. This is the headline. It's *why* we
  pick a log with sequential IO over a per-message-bookkeeping broker.
- **Durability** — an acked write must survive broker crashes and disk loss of a single node. We get
  this from **replication + WAL**, tuned via producer `acks`.
- **Ordering within a partition** — records for the same key arrive in the order produced. **No global
  ordering** — I'll defend that this is the *right* granularity, not a limitation.
- **At-least-once delivery + idempotent producer** — never lose an acked record; tolerate (and dedup)
  duplicates. Exactly-once only "effectively," inside the system, via idempotent producer + transactions.
- **Replayability** — re-read history to bootstrap a new service or fix a consumer bug.
- **Horizontal scale** — add brokers to grow storage + throughput; add partitions to grow consumer
  parallelism (with the ceiling caveat below).
- **Availability** — survive broker failures with automatic leader failover; target 99.95%+. A partition
  can choose to *reject* writes rather than lose data (a CAP knob — see ISR / `min.insync.replicas`).
- **Latency** — low single-digit-ms produce/consume at the broker, but we explicitly trade a little
  latency for throughput via **batching** — this is a throughput-first system, not a latency-first one.

> **The staff move in Step 1:** I'm *positioning* before drawing. "Replicated partitioned log, dumb
> broker / smart client, consensus only for leader election" is the thesis that makes every later choice
> derivable. Per **PACELC**: a single partition is effectively **PC/EC** for *that* partition (under
> partition, the minority side rejects writes to stay consistent via `min.insync.replicas`; in normal
> operation `acks=all` chooses Consistency/durability over Latency). But the knob is **per-producer**,
> so a metrics pipeline can dial itself to PA/EL with `acks=0`. Naming that the consistency level is a
> *knob, not a global mode* is the senior framing.

---

## Step 2 — Estimations (3 min) — to justify, not impress

I'll pick numbers that *force* the architecture, so nothing is arbitrary. Target: a large event bus —
say a clickstream + CDC + app events backbone.

- **Ingest rate:** **1M messages/sec** at peak average across all topics; peak burst ~2–3× → call it
  **2M msg/s** to provision for.
- **Message size:** ~1 KB average (a JSON event).
- **Write bandwidth:** `1M/s × 1 KB = ~1 GB/s` sustained ingest, ~2 GB/s peak.

| Quantity | Formula | Result |
|---|---|---|
| Ingest throughput | 1M msg/s × 1 KB | **~1 GB/s** (2 GB/s peak) |
| Replicated write IO | 1 GB/s × RF=3 | **~3 GB/s** physical writes across cluster |
| Daily volume | 1 GB/s × 86,400 s | **~86 TB/day** logical |
| Retention storage (7d, RF=3) | 86 TB × 7 × 3 | **~1.8 PB** physical on disk |
| Read bandwidth (fan-out ×3 consumer groups) | 1 GB/s × 3 | **~3 GB/s** read, mostly from page cache |
| Brokers (network-bound, ~1 GB/s NIC each, leave headroom) | ~6 GB/s total IO / ~1 GB/s usable per node | **~20–40 brokers** for throughput; more for storage |
| Partitions for a 1 GB/s topic (≈10 MB/s per partition sustainable) | 1 GB/s ÷ 10 MB/s | **~100 partitions** for that topic |

> What the math *justifies*:
> - **3 GB/s of replicated writes** → no single broker holds a hot topic → **partition the topic across
>   brokers** (sharding became necessary, not assumed).
> - **1.8 PB for only 7 days** → retention is expensive → I'll want **tiered storage** (offload cold
>   segments to object storage) so retention isn't capped by local disk.
> - **~100 partitions** for one topic → that's also the **consumer-parallelism ceiling** (Part 6C). The
>   estimate decided partition count, and partition count decides max consumers — I'll call that out.
> - Read ≈ write bandwidth but served mostly from **page cache** (recent data) → **zero-copy** matters
>   because the broker ships ~3 GB/s it should barely touch.

---

## Step 3 — API design (3 min)

Deliberately small — the contract *is* the "dumb broker, smart client" argument. The client library
holds the cleverness (batching, partitioner, offset tracking); the broker exposes append + range-read.

```
# Producer
produce(topic, key, value, headers, acks)      -> { partition, offset }
   # key decides partition: partition = hash(key) % numPartitions  (null key -> round-robin)
   # acks in {0, 1, all}  -> durability knob (Part 6D)
   # client batches+compresses many records into one produce request

# Consumer (pull-based)
subscribe(topic, groupId)                       -> assigned partitions (after rebalance)
poll(maxBytes, maxWaitMs)                        -> [records], each (partition, offset, key, value)
commit(groupId, topic, partition, offset)        -> ack  # store progress
seek(topic, partition, offset | timestamp)       -> reset position  # this is REPLAY

# Admin
createTopic(name, numPartitions, replicationFactor, retentionMs)
```

Three contract decisions I'll flag now and justify in the deep dive:
1. **The producer sends a `key`, and the *client* computes the partition** (`hash(key) % N`). That one
   line controls ordering, load distribution, and hot-spotting all at once (**doc 07, Part C/D**).
2. **Consumers `poll` (pull), not get pushed.** Pull means consumers control their own rate → **natural
   backpressure** (**doc 07, Part G**); the broker never has to track per-consumer flow control.
3. **`seek` is a first-class operation.** Offsets are just integers into an immutable log, so "go back
   and reprocess" is a cheap cursor move — that's the entire value of a log over a queue.

> Auth/TLS/quota enforcement terminate at the broker's network layer or a thin proxy — I won't
> re-explain them (**API-gateway doc 09**). The interesting surface here is the data plane: append and
> ranged read.

---

## Step 4 — Data model (5 min): the log abstraction is the whole design

Entities + access patterns. The access pattern ("append to the end; read a contiguous range from an
offset") is *why* the storage engine is an append-only segmented file and nothing fancier.

- **Topic** — a named stream ("clicks", "orders", "cdc.users"). Logical only.
- **Partition** — a topic is split into N partitions; **each partition is an independent, ordered,
  append-only log.** The partition is the unit of *ordering*, *parallelism*, and *storage distribution*.
  Everything important happens at the partition level.
- **Offset** — a monotonically increasing 64-bit integer naming a record's position **within a
  partition**. A consumer's position is `(topic, partition, offset)`. Offsets never change and are never
  reused — this is what makes replay and "resume where I left off" trivial.
- **Record** — `{ key, value, timestamp, headers }`. The **key** decides the partition; the value is an
  opaque blob to the broker (it never parses it — part of why it's fast).

**On-disk layout of one partition** (this is the access pattern made physical, ties to **databases doc 03**):
```
partition dir/
  00000000000000000000.log     <- segment: the append-only record bytes
  00000000000000000000.index   <- sparse offset -> file-position map
  00000000000000000000.timeindex<- sparse timestamp -> offset map (for seek-by-time)
  00000000000000100000.log     <- next segment, rolls at size/time
  ...
```
- The log is split into **segments** (e.g. 1 GB each). Writes always append to the **active (last)
  segment**. Old segments are immutable → safe to delete (retention), compact, or offload (tiering).
- The **sparse index** maps every Nth offset to a byte position, so a `seek(offset)` is "binary-search
  the sparse index → seek to the nearby byte position → scan forward a little." O(log) lookup, no
  per-record index bloat.

> **Why I'm not reaching for a B-tree, an LSM-tree, or any database here:** the access pattern is
> *append at the tail, read forward by offset.* There are no random updates, no point-deletes-by-id, no
> secondary indexes. So the optimal structure is the simplest one: **a sequential file + a sparse
> offset index.** This is the deliberate inversion of the KV store (design 05), which needed an LSM-tree
> because its access pattern was random point lookups. *The access pattern chose the engine* — I'm not
> picking "append-only file" by reflex, it's forced by "append + range-scan, no random mutation."

---

## Step 5 — High-level design (10 min) — happy path end to end

Boxes and arrows first; deliberately simple, then evolve under questioning.

```
   Producers                         BROKER CLUSTER                         Consumer groups
                          ┌──────────────────────────────────────┐
  ┌────────┐ produce      │   Broker 1        Broker 2   Broker 3 │   poll   ┌───────────────┐
  │client  │──(batch+     │  ┌────────┐     ┌────────┐  ┌───────┐ │◀────────│ group A: C1 C2 │
  │library │  compress)──▶│  │P0 LEAD │────▶│P0 FOLL │  │P0 FOLL│ │         │ (analytics)   │
  └────────┘              │  │P1 FOLL │◀────│P1 LEAD │─▶│P1 FOLL│ │   poll   ├───────────────┤
                          │  │P2 FOLL │     │P2 FOLL │◀─│P2 LEAD│ │◀────────│ group B: C1   │
                          │  └────────┘     └────────┘  └───────┘ │         │ (search index)│
                          │       ▲ leader/follower replication    │         └───────────────┘
                          │       │   (followers PULL from leader) │
                          │  ┌────┴───────────────────────────┐    │   __consumer_offsets topic
                          │  │ CONTROLLER (Raft/KRaft quorum)  │    │   stores each group's committed
                          │  │ owns metadata: which broker     │    │   offset per partition
                          │  │ leads each partition, ISR sets, │    │
                          │  │ topic config; does leader       │    │
                          │  │ election on broker failure      │    │
                          │  └─────────────────────────────────┘    │
                          └──────────────────────────────────────┘
```

**Key architectural properties of the happy path:**
- **A topic's partitions are spread across brokers** → aggregate throughput and storage scale with
  broker count. Each partition has **RF replicas**; exactly one is the **leader** (serves all reads +
  writes for that partition), the rest are **followers** that pull and replicate (**replication doc 05,
  leader-follower**).
- **The broker is dumb on the data path.** Produce = append bytes to the active segment; consume = serve
  a byte range from an offset (often straight from page cache via zero-copy). It does *not* track
  per-message per-consumer state. That bookkeeping lives in the **client** (offsets) — "dumb broker,
  smart client" (**doc 07, Part C**).
- **A small consensus layer (the controller — KRaft Raft quorum, historically ZooKeeper) owns the
  metadata**: the partition→leader map, ISR membership, topic configs. It does **leader election** on
  broker failure. This is the *only* place we pay for consensus, and it's off the hot data path
  (**consensus doc 08**).
- **Consumers pull** at their own offset; **multiple groups** read the same partitions independently
  (fan-out). Offsets are committed to an internal `__consumer_offsets` topic — the system stores
  consumer state *in itself*, as just another log.

**Walking one record end to end (out loud):**
producer client buffers the record into a per-partition **batch** → compresses the batch → sends one
**produce request** to the **leader** of `partition = hash(key) % N` → leader appends the batch to its
active segment (WAL is the segment itself) → **followers in the ISR pull and append** → once `acks=all`
ISR members have it, leader returns `{partition, offset}` → the record now sits in the log, retained →
**consumer group A** polls partition P, reads the byte range from `offset`, processes, and **commits
offset** → independently **group B** reads the *same* records at *its own* offset → days later someone
`seek`s group C back to offset 0 to **replay** the whole history into a new search index. Everything
else (replication failure handling, rebalancing, dedup, compaction, tiering) is the machinery that makes
this correct *despite* failures and at scale — that's the deep dive.

---

## Step 6 — Deep dives (the bulk of the round)

I'll propose the order, which follows the natural dependency chain:
**(A) the log + why sequential IO/page-cache/zero-copy is fast → (B) partitioning a topic across brokers
→ (C) consumer groups, assignment, rebalancing, the parallelism ceiling → (D) replication: leaders,
followers, ISR, `acks` → (E) leader election via the controller/consensus → (F) producer path: batching,
compression, idempotent producer → (G) consumer offsets + delivery semantics + exactly-once → (H)
retention, log compaction, tiered storage → (I) idempotency/dedup on the consumer → (J) backpressure,
consumer lag, hot partitions → (K) full produce/consume path end-to-end → (L) contrast with a
traditional queue.** You can't talk replication before you've placed partitions, can't talk leader
election before replication, can't talk delivery semantics before offsets exist.

---

### 6A. The log abstraction, and *why* it's fast (sequential IO + page cache + zero-copy)

**The problem:** sustain ~3 GB/s of replicated writes and ~3 GB/s of fan-out reads on commodity disks,
where random IO would die at a few hundred IOPS. The answer is to make *every* disk access sequential
and to barely touch the data in userspace.

**Four mechanisms, each a thing to say out loud** (**databases doc 03** + **doc 07, Part C**):

1. **Append-only, sequential disk IO.** Writes are appends to the end of the active segment file; reads
   are sequential scans from an offset. Sequential disk throughput on a modern SSD/HDD is *orders of
   magnitude* higher than random IO and approaches memory-bandwidth on NVMe. Because the log is immutable
   and append-only, there is **no read-before-write, no in-place update, no random seek** — the exact
   opposite of a B-tree's random-update pattern. *This* is why a "dumb file" beats a clever index here.
2. **Page cache, not an application heap cache.** The broker writes to the **OS page cache** and lets the
   kernel flush to disk asynchronously. Recently produced records are still in RAM, so consumers reading
   the tail (the common case — they're caught up) are served **from page cache without touching disk**.
   We deliberately *don't* maintain a JVM/heap cache: that would double-buffer, cause GC pressure, and
   waste RAM the OS already manages well. Free durability-vs-latency tuning lives in the flush interval.
3. **Zero-copy (`sendfile`).** To serve a consumer, the broker tells the kernel "send these bytes of this
   file to this socket" via `sendfile`. The data goes **page cache → NIC** without ever being copied into
   the broker's userspace and back. The broker barely touches the payload — which is also why it can't
   (and doesn't) parse or transform the value. At 3 GB/s of reads, avoiding two memory copies per byte is
   the difference between feasible and not.
4. **Batching + compression amortize per-message overhead.** Producers group many records into one
   compressed batch (Part 6F); the broker **stores and serves the batch as-is**, compressed. So per-message
   syscall/network/CRC overhead is amortized across hundreds of records, and the compressed bytes flow
   end-to-end (producer → disk → consumer) without re-compression.

> **The line:** "It's fast *because the broker does almost nothing per message* — append bytes, serve
> byte ranges, never parse the payload. Sequential IO + page cache + zero-copy mean throughput is bounded
> by the **NIC and disk, not by broker logic.** That's the whole reason a log out-throughputs a
> bookkeeping queue." This directly ties to **databases doc 03** (WAL/sequential append) and **doc 07,
> Part C** ("why Kafka is fast").

---

### 6B. Partitioning a topic across brokers (the partition key, ordering, the ceiling)

**The problem (from the estimate):** one topic needs ~1 GB/s and ~100 partitions' worth of parallelism —
no single broker or single log can hold it. So a topic is **split into N partitions**, each an
independent log, and partitions are **distributed across brokers** (**sharding doc 04**).

**How a record picks a partition:** `partition = hash(key) % numPartitions` (`null` key → round-robin).
The key choice controls **three things at once** (**doc 07, Part D**):
- **Ordering.** Same key → same partition → records for that key are strictly ordered. *Ordering is a
  per-partition property only* — there is **no global order across the topic** (6C explains why that's
  fine). So you key on the entity whose events must stay ordered: `orderId`, `accountId`, `userId`.
- **Load distribution.** A good high-cardinality key spreads load evenly across partitions/brokers.
- **Hot-spotting.** A low-cardinality or skewed key (`country`, one celebrity `userId`) sends a
  disproportionate share to one partition → a **hot partition** (6J). A skewed key defeats partitioning.

**The partition-count ceiling — say this before they ask:** the number of partitions is the **upper bound
on useful consumer parallelism** in a group (6C). And **you can't easily change it**: `hash(key) % N`
changes meaning if N changes, so *adding partitions remaps keys and breaks existing key→partition
ordering* for in-flight keys. Therefore:

> **Tradeoff stated:** I **over-provision partitions up front** (e.g. 100 for a 1 GB/s topic, even if I
> launch with 10 consumers) so I have headroom to scale consumers later *without* a repartition that
> would scramble ordering. The cost of too many partitions is more open file handles, more leader
> elections to manage, and higher end-to-end latency (more, smaller batches) + more controller metadata.
> So I size partitions to **peak future consumer count + throughput**, not today's — but not 10×, because
> partitions aren't free. This "partition count is a near-irreversible capacity decision" point is the
> staff signal (**sharding doc 04: rebalancing/repartition is painful; doc 07, Part C**).

---

### 6C. Consumer groups, partition assignment, rebalancing, and the parallelism ceiling

**The scaling primitive (the thing most candidates get fuzzy on):**
- A **consumer group** is a set of consumers that *collectively* consume a topic. Within a group, **each
  partition is assigned to exactly one consumer** — so a record is processed once *per group*.
- **Different groups are fully independent**: group A (analytics) and group B (search indexer) each read
  *every* record, each tracking its own offsets. That's how a log gives **fan-out** for free — N
  consumers, N copies, no producer change (**doc 07, Part J**).
- **Therefore max useful parallelism in a group = number of partitions.** 100 partitions → at most 100
  active consumers in a group; a 101st sits **idle**. This is the **partition-count ceiling** and the
  favorite gotcha. To scale consumption you add consumers *up to* the partition count, then you're stuck
  until you (painfully) repartition. This is *why* 6B over-provisions partitions.

**Partition assignment + rebalancing:**
- When a consumer **joins, leaves, or dies**, the group must **rebalance** — reassign partitions among
  the surviving consumers. A coordinator broker (the group coordinator) drives this.
- **The cost:** a naive (eager) rebalance is **stop-the-world** — *all* consumers stop, revoke all
  partitions, and re-acquire. Consumption pauses for the whole group during the reshuffle, which at scale
  (frequent deploys, autoscaling) is a real availability hit.
- **Fix — cooperative/incremental rebalancing:** only the partitions that actually need to move are
  revoked; the rest keep consuming. A deploy that adds one consumer reassigns a handful of partitions
  instead of freezing everyone. I'd default to this.
- **Sticky assignment** keeps a consumer on the same partitions across rebalances where possible, so
  local state/caches stay warm.

> **Tradeoff stated:** the consumer-group model gives effortless fan-out (across groups) *and* competing-
> consumer work-sharing (within a group) from one mechanism — but it **couples scaling to partition
> count** and pays a **rebalance pause** on membership change. I accept it because the alternative
> (broker tracks per-message per-consumer state, like a queue) is exactly the bookkeeping that kills
> throughput. I mitigate the pauses with cooperative rebalancing and keep rebalances rare by setting
> sane session timeouts so a brief GC pause doesn't evict a healthy consumer.

---

### 6D. Replication: leaders, followers, ISR, and the `acks` durability knob

Durability requirement → each partition is replicated **RF** times (commonly 3) across brokers in
different failure domains (racks/AZs) (**replication doc 05, leader-follower**).

- **Leader/follower per partition.** One replica is the **leader**: it handles *all* produces and
  consumes for that partition. **Followers don't serve clients** — they only **pull** record batches from
  the leader and append them (replication is itself just consuming the leader's log — elegant reuse).
  Different partitions of the same topic have leaders on *different* brokers, so write load spreads even
  though each individual partition is single-leader.
- **ISR (In-Sync Replicas)** — the set of replicas (including the leader) that are **caught up** to the
  leader within a lag threshold. ISR is the **durability frontier**: a follower that falls behind is
  *removed* from ISR; when it catches up it rejoins. **A leader can only fail over to an ISR member**
  (6E) — promoting a lagging out-of-sync replica would silently lose committed records.

**The producer `acks` knob — the core durability/latency tradeoff** (**doc 07, Part C**):

| `acks` | Leader waits for… | Durability | Latency | Use it for |
|---|---|---|---|---|
| `0` | nothing (fire-and-forget) | **can lose data** even if leader is up | lowest | metrics/telemetry where loss is fine |
| `1` | leader's own local append | lose data if leader dies **before** followers replicate | low | logs where rare loss is acceptable |
| `all` | **all ISR members** append | **no loss** as long as ≥1 ISR survives | higher | orders, payments, CDC — anything durable |

- **`min.insync.replicas`** is the companion knob: with `acks=all` and `min.insync.replicas=2`, a write
  is acked only if **≥2 replicas (leader + ≥1 follower) persist it.** If ISR shrinks below 2 (too many
  brokers down), the partition **rejects writes** rather than ack something that could be lost.

> **Tradeoff stated — and it's a CAP decision you name explicitly:** `acks=all` +
> `min.insync.replicas=2` chooses **consistency/durability over availability** for that partition: under
> a partition/failure that drops ISR below the floor, the minority side **stops accepting writes** (PC).
> The cost is higher produce latency (wait for a follower round-trip) and possible write unavailability
> during failures. I accept it for durable topics and dial down to `acks=1`/`0` for loss-tolerant ones —
> *per producer, per topic*, which is the "consistency is a knob" thesis made concrete. This is the same
> quorum-vs-availability tension as the KV store (design 05), but with a *single leader* instead of
> leaderless quorums — so there are **no write conflicts to reconcile** (the leader imposes a total order
> per partition), which is the whole reason logs don't need vector clocks.

---

### 6E. Leader election on broker failure (the controller / ZooKeeper / KRaft)

**The problem:** when a broker dies, every partition it *led* now has no leader and is unwritable/
unreadable. Something must **detect the failure and elect a new leader from the ISR** — fast, and
**without split-brain** (two brokers both thinking they lead the same partition = divergent logs =
corruption). Agreeing on "who leads partition P" across the cluster is exactly a **consensus problem**
(**consensus doc 08**).

**The controller — a small consensus group, deliberately off the data path:**
- A single **controller** owns cluster metadata: the partition→leader map, ISR sets, topic configs, broker
  liveness. It is the *only* component that needs strong agreement, so we isolate it.
- **Historically** this metadata lived in **ZooKeeper** (a separate Raft/ZAB-based ensemble), with one
  broker elected controller via ZooKeeper. **Modern Kafka (KRaft)** removes the external dependency: a
  small set of **controller nodes run a Raft quorum among themselves** and store the metadata as — fittingly
  — a **replicated metadata log**. The active controller is the Raft leader of that quorum.
- **On broker failure:** the controller (notified by the failure detector / lost heartbeat / session
  expiry) picks, for each affected partition, a **new leader from that partition's current ISR**,
  publishes the new partition→leader map through the metadata log, and brokers/clients learn the new
  leader and reroute. Because the ISR is the durability frontier, the new leader already has every acked
  record → **no acknowledged data is lost** on failover.

**Why this two-tier split is the right design** (the staff insight):
- **Consensus is expensive (a quorum round-trip per decision) and we use it only for metadata**, which
  changes rarely (leadership/ISR changes), **never per message.** The hot data path (produce/consume)
  involves **zero consensus** — the single partition leader just appends. That's how we keep millions of
  msg/s while still having strongly-consistent leadership.
- **Split-brain is prevented** because leadership comes from a single Raft-agreed source of truth with a
  **leader epoch** (a monotonically increasing term number, exactly like a Raft term — **consensus doc
  08**). A produce/replicate request carries the epoch; a stale leader that was partitioned away has an
  old epoch and gets **fenced** (rejected), so a deposed leader can't keep accepting writes.

> **Tradeoff stated:** putting metadata in a Raft quorum costs a coordination dependency and quorum
> latency *for metadata operations*, and the controller is a logical SPOF — mitigated by it being a
> **replicated quorum** (any controller node can take over via Raft leader election). I pay consensus
> **only here** because leader election genuinely needs single-value agreement (you cannot have two
> leaders). This is the exact "do I need consensus? — yes, for leader election; no, for the data path"
> call from **consensus doc 08, Part H/J.** Contrast the KV store (design 05), which *avoided* consensus
> entirely with gossip + leaderless quorums — that worked because it tolerated conflicting writes; a log
> wants a single total order per partition, so it *does* want a single elected leader, so it *does* pay
> for consensus. Same toolkit, opposite call, both justified by the consistency requirement.

---

### 6F. Producer path: batching, compression, idempotent producer, ordering

The producer client is where "smart client" lives.

- **Batching.** The client buffers records per-partition and sends them as one **batch** when the batch
  fills (`batch.size`) or a timer fires (`linger.ms`). This amortizes network/syscall/CRC overhead across
  hundreds of records — the single biggest throughput lever. **Tradeoff:** `linger.ms` trades latency for
  throughput (wait a few ms to fill a bigger batch). A throughput-first bus sets `linger.ms` > 0; a
  latency-sensitive path sets it to 0.
- **Compression.** The batch is compressed (lz4/zstd/snappy/gzip) **once on the client**, stored
  compressed on the broker, and shipped compressed to consumers (decompressed only on the consumer). So
  compression cost is paid once and the *network and disk savings flow end-to-end.* Bigger batches
  compress better → batching and compression reinforce each other.
- **Idempotent producer (`enable.idempotence=true`) — solving producer-side duplicates** (**doc 07, Part
  D**): without it, a producer that times out and **retries** a produce can write the record **twice**
  (the first attempt actually succeeded; only the ack was lost — the classic Two Generals problem). The
  fix: the producer gets a **producer ID (PID)** and stamps each record with a **monotonic sequence
  number per partition**. The broker tracks the last sequence number it accepted per (PID, partition) and
  **rejects/dedups a duplicate or out-of-order sequence** → a retry doesn't create a duplicate *in the
  log*, and ordering is preserved even with retries enabled. This makes the producer→broker hop
  **effectively-once**.
- **Ordering + in-flight requests:** to keep strict per-partition order with retries, the idempotent
  producer bounds in-flight batches and uses the sequence numbers to reorder/reject — so you get ordering
  *and* pipelining, not one or the other.

> **The line:** "Batching + compression are why one cluster does GB/s; the idempotent producer is why a
> network retry doesn't duplicate the record. Note this only fixes **producer-side** dups — duplicates
> from *consumer* reprocessing are a separate problem I solve with consumer idempotency (6I)." Calling
> out that idempotent-producer ≠ end-to-end exactly-once is the nuance mid-level candidates miss.

---

### 6G. Consumer offsets + delivery semantics + "exactly-once"

**Where offsets are stored (and why that's elegant):** a consumer group's committed offset per
`(topic, partition)` is written to an **internal compacted topic, `__consumer_offsets`** (keyed by
`group/topic/partition`, so log compaction (6H) keeps only the latest offset per key). The system stores
consumer progress **in itself, as just another log** — no separate database, no special storage engine.
A restarted/rebalanced consumer reads its last committed offset and **resumes exactly there**.

**The three delivery semantics — and the ack-ordering that produces each** (**doc 07, Part D/E**):

| Semantic | How | Risk |
|---|---|---|
| **At-most-once** | commit offset **before** processing | crash after commit, before work → **lost record** |
| **At-least-once** | commit offset **after** processing | crash after work, before commit → **reprocess (duplicate)** — the realistic default |
| **Exactly-once** | idempotent producer + transactions, or consumer-side idempotency | complex; narrower than people think |

**Why "exactly-once" is really "effectively-once":** true exactly-once *delivery* over an unreliable
network is impossible (Two Generals — you can never be sure your commit/ack landed, so you must choose to
risk a dup or a loss). What we deliver is **effectively-once**: a record may be *delivered* more than
once but its *effect* happens once.
- **Kafka transactions** let a **consume→process→produce** loop atomically write output records **and**
  commit the input offsets in one transaction. With consumers reading `read_committed`, the
  Kafka→Kafka pipeline is **exactly-once *within Kafka***.
- **The catch (say it unprompted):** the moment the consumer writes to an **external** system (Postgres,
  a payment API), Kafka's transaction can't cover it — you're back to **consumer-side idempotency** (6I).
  Kafka can't make your DB write exactly-once.

> **Say this in the room:** "I'll design for **at-least-once** — commit after processing — and make the
> consumer idempotent. Exactly-once isn't a toggle; it's at-least-once delivery plus an idempotent
> effect, and that's what actually holds when I write outside Kafka."

---

### 6H. Retention, log compaction, and tiered storage

The estimate said 7-day retention = **1.8 PB**. Storage is the binding constraint, so retention policy
and where bytes live are first-class.

- **Time/size retention (the default).** Keep records for `retentionMs` (e.g. 7d) or until the partition
  hits `retentionBytes`, then **delete whole oldest segments.** Because segments are immutable files,
  deletion is just `unlink` — no compaction of live data, no rewrite. Cheap.
- **Log compaction (a different model).** Keep **at least the latest value per key**, discard superseded
  older values. The result is a "**latest-state-per-key**" log — perfect for **changelogs / CDC /
  rebuilding a key-value store or cache from the topic** (and it's exactly how `__consumer_offsets`
  stays bounded). A compacted topic is effectively a durable, replayable **snapshot of current state**:
  replay it and you get every key's latest value, with no unbounded growth. (Tie to **databases doc 03**:
  this is the streaming analog of a compacting LSM merge that drops superseded versions.)
- **Tiered storage — what the 1.8 PB estimate forces.** Local broker disk is fast but small/expensive.
  Tiered storage keeps **recent (hot) segments on local disk** (served via page cache + zero-copy at full
  speed) and **offloads sealed (cold) segments to object storage (S3/GCS)**. Retention can then be months
  or years at object-storage cost, and a replay of old data streams from S3. **Crucially this decouples
  retention from broker disk capacity**, so I can grow retention without adding brokers — directly
  addressing the 1.8 PB problem.

> **Tradeoff stated:** compaction gives compact replayable state but **loses history** (you can't see a
> key's intermediate values) — use it for changelogs, not audit logs. Tiered storage gives cheap long
> retention but adds an **object-store dependency and higher-latency cold reads** (a replay from S3 is
> slower than from page cache) and an upload/lifecycle path that can fail. For a log that's mostly read
> at the tail, paying full speed for hot data and cheap storage for cold data is the right split.

---

### 6I. Idempotency / dedup on the consumer (because at-least-once is the default)

Since at-least-once is reality, **the consumer must produce the same result whether it sees a record once
or five times** (**doc 07, Part E**). Three tools, in order of preference:

1. **Naturally idempotent operations.** Prefer them. `SET balance = 100` is idempotent; `balance += 10`
   is not. `UPSERT order(id, …)` is idempotent; blind `INSERT` is not. Design the write as a set/upsert
   keyed by a stable ID and most of the problem evaporates.
2. **A stable idempotency key per record.** Use the business id (`orderId`), or — log-native — the
   tuple **`(topic, partition, offset)`**, which is globally unique and stable across redeliveries.
3. **Dedup table + atomic write.** Persist processed keys (with a TTL) and do the **business effect + the
   dedup-key insert in one local transaction**:
```
on record r:
  BEGIN
    if exists(processed where id = r.id): COMMIT; commit_offset; return   -- already done
    apply business effect(r)
    insert processed(id = r.id)             -- INSERT ... ON CONFLICT DO NOTHING
  COMMIT
  commit_offset(r)                          -- commit Kafka offset LAST
```
- **Ordering of the commit matters:** make the side effect + dedup-insert atomic, and **commit the Kafka
  offset last.** Crash after the DB commit but before the offset commit → the record is redelivered, the
  dedup table catches it, the effect is a no-op. That's how at-least-once delivery becomes effectively-once
  *effect* even when writing to an external store (which Kafka transactions can't cover — 6G).
- Bound dedup state with a **TTL** (you only need to dedup within the redelivery window) or a **Bloom
  filter** for a fast "definitely not seen" check (**framework toolkit: Bloom filter**).

> This is the same idempotency machinery as the payments design (07) — and it's the answer to the
> standing debt the messaging building block creates: *"a queue/log buys decoupling at the price of
> at-least-once delivery → you owe idempotency."* I name the debt and pay it explicitly.

---

### 6J. Backpressure, consumer lag, and hot partitions

**Consumer lag — the health metric.** `lag = log-head offset − consumer's committed offset`, per
partition. Growing lag = consumers can't keep up; the backlog and end-to-end latency grow. **Alert on lag
*and lag rate*** (**doc 07, Part G**).

**Backpressure — the log *is* the buffer.** Because consumers **pull** at their own rate, a slow consumer
does **not** block producers (unlike an in-memory push queue) — the broker just keeps appending and the
slow consumer falls behind. That's natural backpressure: the system degrades into *lag*, not into
*producer failure*. **But retention is finite**, so the real failure mode is insidious:
> **If lag exceeds retention, the consumer's next offset has been deleted → the consumer silently skips
> data (or resets to the earliest available).** Not a crash — **silent data loss.** The mitigation is to
> **size retention to survive your worst expected consumer outage** (e.g., 7 days of retention so a
> consumer can be down for days and still catch up), and alert long before lag approaches the retention
> horizon.

**Handling slow consumers:**
- **Scale out consumers — up to the partition ceiling** (6C). Beyond that you must repartition, so
  provision partitions for *peak* consumer count up front (6B).
- **Speed up per-record work:** batch DB writes, use async IO downstream, drop unnecessary work.
- **Slow-lane topic:** route heavy/slow messages to a separate topic so they don't cause **head-of-line
  blocking** on the fast path.

**Hot partitions (the skew problem).** Partitioning spreads *partitions* evenly, but a **skewed key**
(one celebrity `userId`, a single `country`) sends a disproportionate share to **one** partition →
that partition's leader broker is overloaded while others idle, and the single consumer assigned to it
can't keep up (so lag concentrates there). This is the streaming analog of the KV store's hot-key
problem (design 05) — partitioning fixes hot *shards*, not a hot *single key*. Mitigations:
- **Choose a higher-cardinality / composite key** (`userId#bucket`) to split the hot entity across
  partitions — *if* you can tolerate losing strict ordering for that one entity.
- **Salt the key for the worst offenders only**, keeping normal keys as-is (a hybrid, like the celebrity
  fan-out hybrid from the Twitter design).
- If you genuinely need global ordering *and* it's a single hot stream, you're fundamentally serial —
  accept the throughput ceiling or rethink the requirement.

---

### 6K. The full produce / consume path, end to end

**`produce(topic, key, value, acks=all)`:**
1. Client computes `partition = hash(key) % N`, buffers the record into that partition's **batch**.
2. On `linger.ms`/`batch.size`, client **compresses** the batch and sends it to the **partition leader**
   (it knows the leader from cached metadata; refreshes from the controller on a "not leader" error).
3. Idempotent producer stamps **(PID, sequence#)**; leader **dedups** retries by sequence (6F).
4. Leader **appends** the batch to its active segment (page cache → async flush); WAL = the segment.
5. **ISR followers pull** the batch and append; with `acks=all` + `min.insync.replicas=2`, leader waits
   for ≥2 ISR persists.
6. Leader returns `{partition, offset}`. The record is now durably in the log and retained.

**`consume` (group G):**
1. After a (cooperative) rebalance, consumer C is assigned partition P; it reads its **committed offset**
   from `__consumer_offsets` and `seek`s there.
2. `poll` fetches a byte range from offset onward — served from **page cache via zero-copy** if it's the
   tail (the common, caught-up case), else from disk (or **tiered object storage** for old offsets — 6H).
3. C processes records, doing an **idempotent + atomic** business write (6I).
4. C **commits the offset last** → at-least-once with effectively-once effect.
5. **Group H** does all of the above **independently** at its own offset (fan-out); someone can `seek`
   group J to offset 0 to **replay** the whole topic.

> Walking this out loud and pausing to name *which failure each step tolerates* — `acks=all`/ISR survives
> a broker death without data loss; controller leader-election survives the *leader's* death (6E);
> idempotent producer survives a produce retry; consumer dedup + commit-last survives a consumer crash;
> retention sized > outage survives a slow consumer — is the move that shows I understand the system as a
> *whole*, not a bag of features.

---

### 6L. How this differs from a traditional queue (RabbitMQ / SQS), and when to choose each

The contrast is the cleanest way to prove I understand *why* the log is shaped this way (**doc 07, Part B/L**).

| | **Log (this design: Kafka)** | **Queue (RabbitMQ / SQS)** |
|---|---|---|
| Message lifecycle | **Retained**; reading doesn't delete | **Delivered → acked → deleted**; transient |
| Consumer position | **Consumer-tracked offset** (cursor) | **Broker-tracked** per-message state (in-flight, visibility timeout) |
| Replay | **Yes** — `seek` to any offset | **No** — gone once acked |
| Multiple independent consumers | **Yes** — each group reads everything at its own offset | Competing consumers share one queue (one copy each); fan-out needs SNS/exchanges |
| Ordering | **Per partition** (by key) | Per queue, best-effort (FIFO queues only for guarantees) |
| Per-message features | None (no priority, no per-msg delay/TTL) | **Rich**: priority, TTL, delay, dead-letter, routing keys |
| Throughput | **Very high** (M/s, GB/s) — dumb broker | Moderate — per-message bookkeeping |
| Scaling consumers | Gated by **partition count** | **Add workers freely** — near-linear, no ceiling |
| Ops | High (self-host) / managed (Confluent, MSK) | RabbitMQ medium; **SQS zero-ops** |

> **The decision rule I'd state:** *"Do I need replay, multiple independent consumers, or ordered event
> history? → **log (Kafka).** Do I just need to distribute discrete tasks to a worker pool, with
> per-message priority/delay/TTL and effortless consumer scaling, each task done once and discarded? →
> **queue (SQS/RabbitMQ).**"* A log is a **ledger** (immutable history every reader scans at its own
> position); a queue is a **to-do list** (take an item, do it, cross it off, many workers share it).
> Our requirements — replay, fan-out to many groups, per-key ordering, GB/s throughput — are squarely the
> *log* column, which is why I built a log. If the prompt were "process payment jobs across a worker
> pool with retries and a DLQ," I'd reach for a **queue** instead — and saying *that*, knowing when **not**
> to build this, is the strongest seniority signal.

---

## Step 7 — Wrap-up (3 min): tradeoffs, failure modes, SPOFs

**The thesis held:** a replicated partitioned append-only log; dumb broker / smart client; sequential
IO + page cache + zero-copy for speed; consensus paid *only* for per-partition leader election; durability
and consistency exposed as **per-producer/per-topic knobs** (`acks`, `min.insync.replicas`,
retention/compaction), not a global mode.

**Failure modes to name before asked:**
- **Broker dies (leader of some partitions):** the controller elects a **new leader from the ISR** (6E);
  no acked data lost because ISR is the durability frontier. Brief unavailability for those partitions
  during election; clients refresh metadata and reroute. Leader **epochs** fence the old leader → no
  split-brain.
- **Broker dies (follower):** dropped from ISR; if ISR falls below `min.insync.replicas`, `acks=all`
  writes to that partition are **rejected** (durability over availability — the explicit CAP choice).
  Under-replicated partitions re-replicate to a new broker to restore RF.
- **Consumer slow / falls behind:** lag grows; if **lag > retention → silent data loss** (6J). Mitigate:
  size retention to outlast outages, alert on lag rate, scale consumers up to the partition ceiling.
- **Consumer crashes mid-process:** redelivery from last committed offset → duplicates → handled by
  **consumer idempotency + commit-offset-last** (6I).
- **Poison message** (always fails): naive at-least-once retry **blocks the partition** (head-of-line
  blocking). Use **retry topics with backoff + a DLQ** after N attempts; alert on DLQ depth (**doc 07,
  Part H**).
- **Hot partition** (skewed key): one broker/consumer saturated → re-key or salt the offender (6J).

**Single points of failure / how I add redundancy:**
- **The controller** is a logical SPOF → it's a **replicated Raft quorum (KRaft)**; any controller node
  can take over via Raft leader election. (Pre-KRaft, the ZooKeeper ensemble was the SPOF to make
  redundant.)
- **A single broker** is not a SPOF for data — every partition has RF replicas across failure domains.
- **`__consumer_offsets`** is itself a replicated, compacted topic → no special storage to fail.

**Tradeoffs vs a traditional queue** (the 6L table in one breath): I traded **easy unbounded consumer
scaling, per-message priority/delay/TTL, and zero-ops (SQS)** for **replay, multi-consumer fan-out,
per-key ordering, and GB/s throughput.** Right trade for an event backbone; wrong trade for discrete
task distribution.

**What I'd tune for different workloads:**
- **Durable orders/payments/CDC:** `acks=all`, `min.insync.replicas=2`, idempotent producer, RF=3, key
  by entity id for ordering, consumer idempotency. Durability + ordering over latency.
- **Metrics/telemetry (loss-tolerant):** `acks=0/1`, `null` key (round-robin, max spread), short
  retention. Throughput over durability.
- **Changelog / state rebuild:** **log compaction** + tiered storage; replay to rebuild a cache or KV
  store.
- **Just distributing tasks to workers with retries/DLQ/priority:** **don't build this — use SQS/RabbitMQ.**

**With more time I'd add:** cross-cluster geo-replication (MirrorMaker / cluster linking) for DR and
locality; schema registry + compatibility checks on the value bytes; quotas/rate-limits per
client to protect the cluster (**API-gateway doc 09**, **resilience doc 13**); per-topic tiered-storage
lifecycle policies; and Outbox/CDC (Debezium tailing the DB WAL) as the *correct* way producers get
events into the log without a dual-write (**doc 07, Part I**).

---

## What made this staff-level

- **Stated a thesis up front** (replicated partitioned log; dumb broker / smart client; consensus only
  for leader election) and **derived every choice from it** rather than listing Kafka features.
- **Let estimation *force* the architecture** — 3 GB/s replicated writes ⇒ partition across brokers;
  1.8 PB/7d ⇒ tiered storage; ~100 partitions ⇒ the consumer-parallelism ceiling. Nothing assumed.
- **Explained *why* it's fast from first principles** (sequential IO + page cache + zero-copy + batching),
  tying the speed directly to "the broker does almost nothing per message" — not just naming the buzzwords.
- **Got the consensus split exactly right:** consensus is paid **only** for leader election/metadata (a
  Raft quorum, KRaft), and the **hot data path uses none** — with leader **epochs** to fence split-brain.
  Then contrasted it with the KV store (design 05), which *avoided* consensus via gossip + leaderless
  quorums — **same toolkit, opposite call, each justified by the consistency requirement** (single total
  order per partition vs. reconcilable conflicts).
- **Named the precise tradeoff on every knob and picked a side:** `acks` 0/1/all + `min.insync.replicas`
  as an explicit CAP choice; partition count as a near-irreversible capacity decision; compaction vs.
  history loss; tiered storage vs. cold-read latency; cooperative vs. stop-the-world rebalance.
- **Volunteered the honest caveats** mid-level candidates miss: exactly-once is *effectively*-once and
  Kafka EOS only covers Kafka→Kafka (external writes still need consumer idempotency); idempotent
  *producer* fixes only producer-side dups; lag exceeding retention is **silent data loss**, not a crash;
  ordering is per-partition only and global ordering means no parallelism.
- **Connected storage internals to distributed concerns** — segment files + sparse index + page cache +
  zero-copy (single-node) tied to ISR replication + controller election + consumer-group rebalancing
  (distributed). Most candidates do *either* the on-disk log *or* the cluster; tying both is the depth signal.
- **Closed with the queue-vs-log contrast and a decision rule** — knowing exactly *when not to build this*
  (use SQS/RabbitMQ for task distribution) is the strongest seniority signal.

---

## Self-check (answer from memory before the mock)

- [ ] Queue vs log in one sentence each — and which our requirements (replay, multi-consumer, per-key
      ordering, GB/s) demand, and why?
- [ ] *Why* is a log fast? Name the four mechanisms (sequential IO, page cache, zero-copy, batching/
      compression) and the unifying idea ("the broker does almost nothing per message").
- [ ] How does a record pick its partition, and what three things does the key choice control at once?
- [ ] State the **partition-count ceiling** and why you can't just add partitions later.
- [ ] Define a **consumer group**: within-group vs across-group behavior, and how that yields both
      competing-consumers and fan-out from one mechanism.
- [ ] What is **ISR**, and why can a leader only fail over to an ISR member?
- [ ] `acks` 0/1/all + `min.insync.replicas` — what each buys, and which combo is the explicit CAP choice?
- [ ] Walk **leader election on broker failure**: what does the controller do, where does metadata live
      (ZooKeeper vs KRaft/Raft), and how do **leader epochs** prevent split-brain?
- [ ] *Why* does a log pay for consensus (leader election) when the KV store (design 05) avoided it? What
      requirement flips the call?
- [ ] What does the **idempotent producer** fix (PID + sequence#) — and what does it *not* fix?
- [ ] Where are **consumer offsets** stored, and what's the commit-order rule for at-least-once →
      effectively-once *effect* on an external write?
- [ ] Why is "exactly-once" really "effectively-once," and what does Kafka EOS *not* cover?
- [ ] Time/size retention vs **log compaction** vs **tiered storage** — what each is for, and the cost of
      each.
- [ ] Define **consumer lag** and the silent-data-loss failure when it exceeds retention.
- [ ] What's a **hot partition**, how does it arise, and two mitigations?
- [ ] Give the queue-vs-log decision rule in one sentence, and a workload for each.
