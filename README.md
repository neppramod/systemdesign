# System Design Interview Prep — Staff-Level Curriculum

A complete, opinionated, tradeoff-first prep library for staff/senior system design interviews.
Every doc is written to be **derivable in the room**, not memorized — the goal is to handle
*unseen* problems by recombining building blocks, not to recall solutions.

> **Start here:** read `prep/01` (the framework + building-blocks toolkit). It's the spine —
> every other doc is "a deeper look at a block you already know how to slot in." Then read
> `prep/26` (the interview playbook) to see how it all gets used under the clock.

---

## How the library is organized

- **`prep/`** — 28 building-block deep-dives (plus a `03a` storage-engine foundations companion). The *toolkit*. Each: what problem it solves, the
  internals, the tradeoffs, and the "say this in the room" lines + a from-memory self-check.
- **`designs/`** — 26 fully worked end-to-end design walkthroughs, each run through the 7-step
  framework, cross-referencing the prep docs. The *recombination practice*.
- **`mock-interviews/`** — practice prompts to run cold.

---

## `prep/` — Building Blocks (the toolkit)

| #   | Doc                                     | What it gives you                                                                                                        |
| --- | --------------------------------------- | ------------------------------------------------------------------------------------------------------------------------ |
| 01  | Framework & Building Blocks             | The 7-step framework + the ~15-primitive toolkit. **Read first.**                                                        |
| 02  | Estimation & Napkin Math                | Latency ladder, QPS/storage/bandwidth formulas, numbers-justify-decisions.                                               |
| 03  | Databases Deep Dive                     | Choosing a store, B-tree vs LSM, NoSQL families, indexing, ACID/isolation/MVCC.                                          |
| 03a | Storage Engine Foundations              | B-tree & LSM-tree from first principles — the disk reality, read/write/delete paths, compaction. *Prereq for 03 Part B.* |
| 04  | Sharding & Partitioning                 | Strategies, consistent hashing, shard-key choice, hot shards, resharding.                                                |
| 05  | Replication & Consistency               | Topologies, lag anomalies, quorums, vector clocks, CRDTs, CAP/PACELC.                                                    |
| 06  | Caching Deep Dive                       | Patterns, invalidation, eviction, the 5 failure modes, Redis vs Memcached, CDN.                                          |
| 07  | Messaging & Streaming                   | Queue vs log, Kafka internals, delivery semantics, idempotency, outbox/CDC.                                              |
| 08  | Consensus & Coordination                | Raft (whiteboard-ready), Paxos, ZK/etcd, distributed locks + fencing, Redlock.                                           |
| 09  | Edge: API / LB / Rate Limiting          | REST/gRPC/GraphQL/WS, L4/L7 LB, the 5 rate-limit algorithms, gateways.                                                   |
| 10  | Distributed Txns & Idempotency          | Dual-write, 2PC, sagas, outbox/inbox, idempotency keys, Snowflake, ledgers.                                              |
| 11  | Search Systems                          | Inverted index, BM25, Lucene/ES internals, vector/ANN, autocomplete.                                                     |
| 12  | Geospatial Systems                      | Geohash/quadtree/S2/H3, radius queries, Yelp vs Uber, geofencing.                                                        |
| 13  | Resilience & Failure Handling           | Timeouts/retries/jitter, circuit breakers, bulkheads, DR (RTO/RPO), cascades.                                            |
| 14  | Blob Storage & Media                    | Metadata/bytes split, S3 internals, presigned/multipart, dedup, transcoding/ABR.                                         |
| 15  | Real-Time & Push                        | Polling/SSE/WS/push, C10M, presence, chat, fan-out notifications.                                                        |
| 16  | Microservices & Architecture            | Monolith vs micro, DDD, service mesh, CQRS/event sourcing, multi-tenancy.                                                |
| 17  | Security & Auth                         | Session vs JWT, OAuth/OIDC, RBAC/ABAC/ReBAC, encryption/KMS, attacks.                                                    |
| 18  | Batch & Stream Processing               | OLTP/OLAP, MapReduce/Spark, windowing/watermarks, Lambda/Kappa, HLL/CMS.                                                 |
| 19  | Networking Fundamentals                 | URL→bytes lifecycle, DNS, TCP/UDP/QUIC, TLS, HTTP/1-2-3, latency budgets.                                                |
| 20  | Data Modeling & Access Patterns         | Access-pattern-first schema, denormalization, single-table, migrations.                                                  |
| 21  | Advanced Distributed Theory             | Clocks/HLC, Spanner/TrueTime, CRDTs vs OT, CALM, FLP/BFT, coordination avoidance.                                        |
| 22  | Multi-Region & Geo-Distribution         | Replicas vs active-active vs geo-partition, conflicts, residency, failover, cells.                                       |
| 23  | ML & AI System Design                   | Recsys funnel, feature stores, ANN/vector, model serving, RAG/LLM apps.                                                  |
| 24  | Capacity Planning & Cost                | Sizing, egress economics, reserved/spot, cost as a seniority signal.                                                     |
| 25  | Workflow Orchestration & Event Sourcing | Durable execution (Temporal), event sourcing + CQRS, when they're a trap.                                                |
| 26  | Patterns, Anti-Patterns & Playbook      | Pattern/anti-pattern catalog, the tactical interview script, "hear X → reach for Y".                                     |
| 27  | Observability & SRE                     | 3 pillars, RED/USE, SLO/error budgets, burn-rate alerts, incident/on-call, tail latency.                                 |
| 28  | Infrastructure & Deployment             | Compute spectrum, Kubernetes, autoscaling, IaC/GitOps, deploy strategies.                                                |

## `designs/` — Worked Walkthroughs (recombination practice)

| #   | Design                             | Headline challenge it teaches                                                       |
| --- | ---------------------------------- | ----------------------------------------------------------------------------------- |
| 01  | Twitter / News Feed                | Fan-out write vs read vs **hybrid** (celebrity problem).                            |
| 02  | URL Shortener                      | Code generation, read-heavy caching, the counter bottleneck.                        |
| 03  | YouTube / Netflix                  | Transcoding pipeline + **CDN/bandwidth as the binding constraint**.                 |
| 04  | Uber / Ride-Hailing                | Location firehose + geospatial matching + trip-state consistency.                   |
| 05  | Distributed KV Store (Dynamo)      | Consistent hashing + quorums + vector clocks + gossip + LSM.                        |
| 06  | Web Crawler                        | URL frontier (politeness + priority), dual dedup, distributed crawl.                |
| 07  | Payment System                     | Immutable double-entry **ledger**, idempotency, PSP timeouts, saga, reconciliation. |
| 08  | WhatsApp / Chat                    | Persistent connections, store-and-forward, ordering, group fan-out.                 |
| 09  | Instagram                          | Photo pipeline + hybrid feed + hot-post counters.                                   |
| 10  | Distributed Message Queue (Kafka)  | Partitioned log, ISR replication, consumer groups, exactly-once.                    |
| 11  | Ticketmaster / Flash Sale          | **No oversell**: locking strategies, waiting room, CQRS read/write split.           |
| 12  | Distributed Job Scheduler          | Finding due jobs at scale, leasing, HA scheduler, missed-fire policy.               |
| 13  | Notification System                | Multi-channel fan-out, dedup, priority lanes (OTP isolation), provider failures.    |
| 14  | Google Drive / Dropbox             | Chunking + dedup + delta sync, conflict resolution, ReBAC sharing.                  |
| 15  | Distributed Cache                  | Consistent hashing, cluster topology, failover, the 5 cache failure modes.          |
| 16  | Yelp / Proximity                   | Read-heavy static geo (the inverse of Uber), heavy caching + replicas.              |
| 17  | Search Autocomplete                | Trie + precomputed top-K, build/serve decoupling, keystroke latency.                |
| 18  | Ad Click Aggregator                | Lambda vs Kappa, dedup billing, windowing/watermarks, HLL/CMS.                      |
| 19  | Unique ID Generator                | Snowflake bit layout, UUID vs ticket vs ULID, clock skew.                           |
| 20  | Google Docs / Collab Editing       | **OT vs CRDT**, single-writer ordering, op-log persistence, offline sync.           |
| 21  | Stock Exchange / Matching Engine   | Deterministic single-threaded core, ordered log, sequencer HA, microsecond latency. |
| 22  | Leaderboard / Gaming               | Redis sorted sets, the 100M global-rank problem, windowed boards.                   |
| 23  | Google Maps / Navigation           | Road graph, contraction hierarchies, live-traffic probe loop, tiles.                |
| 24  | CDN                                | Edge hierarchy, anycast/GeoDNS routing, purge propagation, offload math.            |
| 25  | Metrics & Monitoring               | **Cardinality** as the scaling wall, Gorilla compression, downsampling, alerting.   |
| 26  | Distributed File System (GFS/HDFS) | Metadata master + chunkservers, replication vs erasure coding, durability.          |

---

## Suggested study plan

**Phase 1 — Foundations (must be cold-recall fluent).**
`01 framework` → `02 estimation` → `26 playbook`. These three make every other doc usable.

**Phase 2 — The core building blocks (the 80%).**
`03 databases`, `04 sharding`, `05 replication/consistency`, `06 caching`, `07 messaging`,
`10 idempotency/txns`, `13 resilience`. Almost every design leans on these.

**Phase 3 — Specialized blocks (pull as needed per problem).**
`08 consensus`, `09 edge`, `11 search`, `12 geospatial`, `14 blob`, `15 real-time`,
`16 microservices`, `17 security`, `18 batch/stream`, `19 networking`, `20 data modeling`.

**Phase 4 — Staff differentiators.**
`21 advanced theory`, `22 multi-region`, `23 ML/AI`, `24 cost`, `25 workflows/ES-CQRS`,
`27 observability/SRE`, `28 infra`. These are what separate staff from senior.

**Phase 5 — Drill the designs.**
Work `designs/01–26` *cold first* (don't peek), then compare. Aim to derive, not recall.
Practice the same problem at 1×, then "now scale it 10×."

> **Your stated weakness is unseen problems.** The fix is in `01` (the derivation framework) and
> `26` (the "hear X → reach for Y" map + recovery-when-stuck playbook). Drill those two until the
> 7 steps and the trigger-word map are automatic, and an unseen prompt becomes "which blocks?"
> instead of "have I seen this?".

---

## The 7-step framework (memorize cold — full version in `prep/01`)

1. **Requirements** (functional + non-functional; scope explicitly; pin the read/write ratio).
2. **Estimation** (QPS, storage, bandwidth — numbers that *justify* later choices).
3. **API design** (a handful of endpoints; cursor pagination; auth at the gateway).
4. **Data model** (entities + **access patterns** → the access pattern picks the store).
5. **High-level design** (happy path end-to-end; simple first).
6. **Deep dives** (the bottleneck / consistency tradeoff / hot spot — *this is where rounds are won*).
7. **Wrap-up** (bottlenecks, failure modes, SPOFs, what you'd do with more time).

---

*54 docs · ~26k lines · staff-level depth. Build was incremental — extend by adding numbered
docs in `prep/` (new building blocks) or `designs/` (new worked problems) and linking them here.*

## Creating a Book

You can convert this website into a readable book using pandoc

Update metadata.yaml to what you like and run following `pandoc` command. Make sure pandoc is installed.

```bash
pandoc -o system_design_prep.epub metadata.yaml \
  README.md \
  $(ls -v prep/*.md) \
  $(ls -v designs/*.md) \
  $(ls -v mock-interviews/*.md) \
  --toc --toc-depth=2
```
