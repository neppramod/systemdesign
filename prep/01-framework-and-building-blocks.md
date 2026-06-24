# Topic 1: The System Design Framework + Building Blocks Toolkit

> **Why this is Topic 1:** Your stated weakness is *unseen problems*. That is never solved by
> memorizing more solutions. It is solved by (a) a repeatable framework that turns any prompt into
> a derivable design, and (b) a small toolkit of building blocks that you *recombine* to fit
> whatever shows up. Master this one doc and every later topic (caching, sharding, consensus…)
> becomes "a deeper look at a block you already know how to slot in."

---

## Part A — The Framework (run this on EVERY problem)

A 45-minute round, time-boxed. The numbers are a guide, not a law — but the *order* matters.

### 1. Requirements (5 min) — you drive this, don't wait
Split into two buckets and **say them out loud**:

- **Functional** — what the system *does*. Verbs. ("User posts a tweet", "followers see it in their feed.")
  - Narrow scope explicitly: "I'll focus on posting + feed read; I'll skip DMs and ads unless you want them." This shows seniority and protects your time.
- **Non-functional** — how *well* it does it. This is where staff candidates separate themselves:
  - **Scale**: how many users / QPS / data volume?
  - **Latency**: p99 target? (feed read < 200ms, payment < 500ms…)
  - **Consistency**: is stale data OK? (feed = yes; account balance = no)
  - **Availability**: 99.9% vs 99.99%? What's the cost of downtime?
  - **Read vs write ratio**: shapes the *entire* design (most consumer apps are 100:1 read-heavy).

> **The single most common failure:** jumping to a diagram before pinning non-functional reqs.
> The non-functional requirements are what *let you derive* a design you've never seen.

### 2. Estimations (3 min) — justify, don't impress
You don't need precision, you need numbers that *justify later decisions*.
- **QPS**: `DAU × actions/user/day ÷ 86,400`. Then peak ≈ 2–3× average.
- **Storage**: `writes/day × bytes/write × retention`. Multiply out to years.
- **Bandwidth**: `QPS × payload size`.
- The point: "We're at ~50k write QPS and 100TB/yr → single DB won't hold it → we'll shard." Now sharding is *justified*, not memorized.

### 3. API design (3 min)
A handful of endpoints. Pins the contract and surfaces hidden requirements.
```
POST /tweet        { userId, text }            -> tweetId
GET  /feed?userId=&cursor=                     -> [tweets], nextCursor
```
- Use **cursor-based pagination**, not offset, for feeds (offset breaks on inserts).
- Mention auth/rate-limit at the gateway so you don't re-explain it later.

### 4. Data model (5 min)
Entities + **access patterns**. The access pattern decides the store, not the other way around.
- List the main tables/collections and their keys.
- Ask: "How is this read? By what key? Range or point?" → that's your index / partition key.
- SQL vs NoSQL falls out of this (see Part B).

### 5. High-level design (10 min) — get the happy path end-to-end
Boxes and arrows. Client → LB / API gateway → service(s) → storage. Plus async path if needed (queue → worker).
- Keep it simple first. One service, one DB, a cache. *Then* evolve under questioning.
- Walk one request through it out loud: "User posts → gateway authn → write service → DB → enqueue fan-out event → …"

### 6. Deep dives (15 min) — **this is where rounds are won or lost**
The interviewer steers, but you should *propose* the hard part: "The interesting challenge here is the feed fan-out. Can I go deep on that?"
- Pick the **bottleneck, the consistency tradeoff, or the hot spot** and go deep.
- Show the tradeoff explicitly: "Fan-out on write gives fast reads but explodes for celebrities; fan-out on read is cheap to write but slow to read. I'd do a **hybrid**: fan-out on write for normal users, fan-out on read for high-follower accounts."
- Naming a tradeoff and *picking a side with justification* is the staff-level signal.

### 7. Wrap-up (3 min)
- Bottlenecks that remain, failure modes ("what if the queue dies?"), and what you'd do with more time.
- Single points of failure, and how you'd add redundancy.

---

## Part B — The Building Blocks Toolkit

~15 primitives. Every system is a recombination. For each: **what problem it solves**, and
**its tradeoff / when NOT to use it**. That second half is what makes you sound senior.

### Scaling reads
- **Caching** (Redis/Memcached). Solves: repeated reads, slow DB. Patterns: *cache-aside* (default), *write-through*, *write-back*. Tradeoff: **staleness + invalidation is hard** ("there are only two hard problems…"). Watch for: thundering herd, hot keys.
- **Read replicas**. Solves: read scaling. Tradeoff: **replication lag** → reads can be stale; don't read-your-own-writes from a replica.
- **CDN**. Solves: static/media latency, edge delivery. Tradeoff: cache invalidation, cost.

### Scaling writes / data size
- **Sharding / partitioning**. Solves: data + write volume beyond one node. Strategies: hash (even spread, no range queries), range (range queries, risk hot shards), geo. Tradeoff: **cross-shard queries + rebalancing are painful**; pick a shard key that avoids hot spots.
- **Consistent hashing**. Solves: adding/removing nodes without reshuffling everything. Used in caches, Dynamo-style DBs, LBs.
- **LSM-tree stores** (Cassandra, RocksDB). Solves: write-heavy workloads. Tradeoff: read amplification, compaction cost.

### Decoupling & spikes
- **Message queue / log** (Kafka, SQS, RabbitMQ). Solves: decoupling producers/consumers, absorbing spikes, async work, fan-out. Tradeoff: **at-least-once delivery → need idempotency**; ordering only within a partition; adds latency + operational complexity.
- **Event-driven / pub-sub**. Solves: one event, many consumers. Tradeoff: eventual consistency, harder to debug/trace.

### Finding data
- **Indexing**. Point vs range; composite indexes. Tradeoff: slows writes, uses space.
- **Inverted index** (Elasticsearch). Solves: full-text search. Tradeoff: separate system to keep in sync (often fed via CDC/queue).
- **Geospatial index** (geohash, quadtree, S2). Solves: "nearby" queries (Uber, Yelp).

### Durability & consistency
- **Replication** (leader-follower, multi-leader, leaderless). Solves: durability + availability.
- **Quorum (W + R > N)**. Solves: tunable consistency in leaderless systems. Lets you trade consistency for availability per-request.
- **WAL / commit log**. Solves: crash recovery, the basis of replication.
- **Conflict resolution**: last-write-wins (simple, loses data), vector clocks, **CRDTs** (collaborative editing).

### Coordination
- **Consensus** (Raft/Paxos), **leader election**, **etcd/ZooKeeper**. Solves: agreeing on one value/leader across nodes. Tradeoff: latency cost; use only where you truly need strong agreement (config, leader, locks).
- **Distributed locks / idempotency keys**. Solves: exactly-once effects, preventing double-spend.

### Availability & resilience
- **Load balancer** (L4/L7), health checks, failover. Solves: spread traffic, route around dead nodes.
- **Circuit breaker / rate limiter / bulkhead / backpressure**. Solves: stop cascading failure, protect from overload.
- **Bloom filter**. Solves: "is this *definitely not* present?" cheaply (avoid disk lookups, dedupe). Tradeoff: false positives, no deletes.

### The tradeoffs you must recite cold
- **CAP / PACELC** — under partition, choose Consistency or Availability; else (normal operation) choose Latency or Consistency.
- **SQL vs NoSQL** — SQL: relations, transactions, strong consistency, ad-hoc queries. NoSQL: scale, flexible schema, known access patterns, eventual consistency. *Decide from your access patterns (step 4), not by reflex.*
- **Strong vs eventual consistency** — correctness vs availability/latency.
- **Push (fan-out on write) vs pull (fan-out on read)** — fast reads vs cheap writes.
- **Sync vs async** — simplicity/immediacy vs throughput/resilience.
- **Normalization vs denormalization** — write simplicity/space vs read speed.

---

## Part C — How to use this in the room

1. **Always start with requirements + estimation.** It buys thinking time and frames everything.
2. **Map the prompt to blocks out loud.** "This is read-heavy with a 'nearby' query and spiky writes → I'm reaching for a CDN/cache, a geospatial index, and a queue."
3. **Default to the simple design, then evolve under pressure.** Don't pre-optimize.
4. **Every choice gets a tradeoff sentence.** "I'll use X because Y; the cost is Z; I accept it because of our requirement W."
5. **Name failure modes before you're asked.**

> **Mental checklist when stuck on an unseen problem:**
> Read-heavy or write-heavy? → Where's the bottleneck? → Which block scales that axis? →
> What consistency does it need? → What breaks at 10×? → Single point of failure?

---

### Self-check before the mock (answer these from memory)
- [ ] What are the 7 framework steps, in order?
- [ ] Give the QPS and storage estimation formulas.
- [ ] When do you pick NoSQL over SQL? (answer in terms of access patterns)
- [ ] Explain fan-out on write vs read and when you'd do a hybrid.
- [ ] What does a message queue buy you, and what new problem does it create?
- [ ] State PACELC in one sentence.
