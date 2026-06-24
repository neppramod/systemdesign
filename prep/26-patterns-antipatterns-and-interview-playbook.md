# Topic 26: Patterns, Anti-Patterns & The Staff-Level Interview Playbook

> **Why this is the last topic:** Every other doc taught you a block. This one is the *index and
> the meta-game*. It does two jobs. First, it gives you a single place to recognize "what shape is
> this problem, and which pattern do I reach for" — and, just as important, *when NOT to reach for
> it*, because over-application is the most common senior-to-staff failure. Second, it turns the
> 7-step framework from Topic 1 into a *tactical script* you can run under pressure on a problem
> you've never seen. Your stated weakness is unseen problems; the cure is not more memorized
> solutions, it's a derivation mindset plus a recognition vocabulary. That's this doc.

This is a reference/cheat-sheet doc. Skim it before a mock; reread the playbook section the morning of.

---

## Part A — The Pattern Catalog

Quick-reference. For each pattern: the **problem** it solves, a **one-line how**, and **when NOT to
use it**. The "when not" column is the staff-level half — anyone can name a pattern; you get hired by
knowing its cost. Cross-references point at the deep-dive doc.

### Data & read-scaling patterns

| Pattern | Problem it solves | How (one line) | When NOT to use | Doc |
|---|---|---|---|---|
| **Cache-aside** | Slow/expensive repeated reads | App reads cache; on miss, load from DB and populate | Write-heavy or strong-consistency data; tiny datasets | 06 |
| **Read replica** | Read throughput beyond one node | Replicate leader→followers; route reads to followers | When you need read-your-own-writes or strong reads | 05 |
| **CQRS** | Read and write models have different shapes/scale | Separate write path (commands) from read path (queries/projections) | Simple CRUD; one model serves both fine — it's pure overhead | 16, 20 |
| **Materialized view** | Expensive aggregations/joins on the read path | Precompute and persist a query result; refresh on change or schedule | Data changes faster than you can refresh; ad-hoc query needs | 18, 20 |
| **Sharding / partitioning** | Data + write volume beyond one node | Split data across nodes by a partition key | Data fits one node; you need cheap cross-entity transactions | 04 |
| **Consistent hashing** | Add/remove nodes without reshuffling everything | Hash nodes + keys onto a ring; only neighbors move | Fixed small node count; range queries needed | 04 |
| **Bloom filter** | "Is this *definitely not* present?" cheaply | Probabilistic set; no false negatives, some false positives | You need deletes or exact membership | 01, 06 |
| **Fan-out on write** | Fast feed/timeline reads | Push new item into each follower's precomputed feed at write time | High-fanout producers (celebrities) → write explosion | 15 |
| **Fan-out on read** | Cheap writes, bounded write cost | Assemble feed by querying followees at read time | Read latency matters and fanout-in is large | 15 |
| **Hybrid fan-out** | Both ends of the fan-out spectrum in one system | Push for normal users; pull for high-follower accounts | Uniform fanout (no celebrities) → just pick one | 15 |
| **Change data capture (CDC)** | Keep derived stores in sync with the DB | Stream the DB's commit log into search/cache/warehouse | You can own writes at the app layer cleanly; tiny scale | 07, 11, 18 |

### Write, durability & transaction patterns

| Pattern | Problem it solves | How (one line) | When NOT to use | Doc |
|---|---|---|---|---|
| **Write-ahead log (WAL)** | Crash recovery & basis of replication | Append intent to a durable log *before* applying it | You don't control storage internals (it's a given of your DB) | 03, 05 |
| **Event sourcing** | Need full history / audit / temporal replay | Store the sequence of state-changing events as source of truth | You only need current state; team can't stomach the complexity | 16, 20 |
| **Saga** | Multi-service "transaction" without 2PC | Sequence of local txns + compensating actions on failure | A single DB transaction would do; you can tolerate 2PC's coupling | 10 |
| **Outbox** | Atomically update DB *and* publish an event | Write event to an outbox table in the same txn; relay publishes it | You have a real transactional broker / single store | 10, 07 |
| **Idempotency key** | Retries must not double-apply effects | Client sends a unique key; server dedupes on it | Naturally idempotent operations (pure PUT of full state) | 10 |
| **Claim-check** | Large payloads clog the message bus | Put the blob in object store; send only a reference (the "claim") on the queue | Small messages; the indirection isn't worth it | 07, 14 |
| **Dead-letter queue (DLQ)** | Poison messages block the queue forever | After N failures, route the message aside for inspection/replay | Failures are transient and retry+backoff already handles them | 07, 13 |

### Resilience & traffic patterns

| Pattern | Problem it solves | How (one line) | When NOT to use | Doc |
|---|---|---|---|---|
| **Retry + backoff + jitter** | Transient failures | Retry with exponential backoff and *randomized* delay | Non-idempotent ops without an idempotency key; hard failures | 13 |
| **Circuit breaker** | Stop hammering a failing dependency | Trip open after a failure threshold; fail fast; probe to half-open | Single retryable blip; in-process pure functions | 13 |
| **Bulkhead** | One slow dependency drowns all threads | Isolate resources (thread/conn pools) per dependency | Tiny service with one dependency | 13 |
| **Backpressure** | Producers outrun consumers → OOM/collapse | Signal upstream to slow down; bound buffers; shed load | Naturally throttled, low-volume paths | 07, 13 |
| **Rate limiter** | Abuse, fairness, capacity protection | Token bucket / sliding window per key, usually at the gateway | Internal trusted low-volume calls (still consider quotas) | 09 |
| **Competing consumers** | Scale processing of a work queue | Many workers pull from one queue; each message handled once | Strict global ordering required across all messages | 07 |
| **Scatter-gather** | Aggregate results from many backends | Fan a request out to N nodes/shards in parallel, merge replies | One backend can answer; tail latency from slowest node hurts | 11, 12 |
| **Leader election** | Exactly one node owns a job/role | Use consensus (Raft) / etcd / ZooKeeper to elect a leader | Stateless work that any node can do; adds a coordination dep | 08 |

### Structural / architecture patterns

| Pattern | Problem it solves | How (one line) | When NOT to use | Doc |
|---|---|---|---|---|
| **API gateway** | One front door: authn, routing, rate-limit, TLS | Single ingress that cross-cuts concerns before fan-in to services | Single service / monolith; gateway becomes a god-box if overloaded | 09, 16 |
| **BFF (backend-for-frontend)** | Each client (web/mobile) needs a tailored API | A thin per-client aggregation layer over shared services | One client type; over-fragmentation multiplies maintenance | 09, 16 |
| **Sidecar / service mesh** | Cross-cutting infra (mTLS, retries, telemetry) out of app code | Co-deployed proxy (Envoy) handles network concerns per pod | Small fleet; the operational + latency overhead isn't justified | 16, 19 |
| **Strangler fig** | Migrate off a legacy system safely | Route slices of traffic to new services incrementally behind a façade | Greenfield; or a clean cutover is genuinely cheaper | 16 |

> **How to use this catalog in the room:** don't recite it. *Recognize the shape* of the problem,
> name the one or two patterns that fit, say the tradeoff sentence, and pick. The "when NOT to use"
> column is the part that turns a name-drop into a judgment call.

---

## Part B — The Anti-Pattern Catalog

The flip side. Each is something candidates *propose unprompted* — and proposing it tanks the
signal. For each: **why it bites** and **the fix**. If the interviewer nudges you toward one of these,
the staff move is to name the trap out loud and steer away.

| Anti-pattern | Why it bites | The fix |
|---|---|---|
| **Distributed monolith** | "Microservices" that must deploy together and call each other synchronously — all the ops cost of distribution, none of the independence. | Define service boundaries around business capabilities + data ownership; async events between them; deploy independently or stay a modular monolith (16). |
| **Shared database across services** | Two services on one DB are coupled at the schema; you can't evolve or scale either independently; nobody owns the data. | One service owns its data; others access via API/events. Use CDC or an outbox to share, not a shared table (10, 16). |
| **Chatty services** | One user action = dozens of cross-service calls → latency stacks, failure surface explodes. | Coarser APIs, batch/aggregate (BFF), or co-locate the data. Push the join down, not across the network (16, 19). |
| **Premature microservices** | You split before you understand the domain; boundaries are wrong; you pay distribution tax with no payoff. | Start as a modular monolith; extract a service only when a real axis (team, scale, deploy cadence) demands it (16). |
| **Premature optimization** | You shard/cache/precompute before estimating, optimizing a non-bottleneck and adding complexity. | Estimate first (02). Build the simple design; optimize the bottleneck the numbers reveal. |
| **Unbounded queues / retries (retry storm)** | Retries pile onto an already-failing dependency; queues grow without limit → memory blowup and a self-inflicted DDoS. | Bounded queues + backpressure; exponential backoff *with jitter*; circuit breaker; retry budgets (13). |
| **Dual writes** | Writing to DB and to a queue/cache in two steps → they diverge on partial failure; no atomicity. | Transactional **outbox** + CDC, or event sourcing. One atomic write, then derive the rest (10, 07). |
| **Cache as system of record** | Treating Redis as the source of truth → data loss on eviction/restart; silent corruption. | Cache is a *derived, disposable* copy. The DB is truth; cache must be rebuildable from it (06). |
| **Synchronous chains (latency amplification)** | A→B→C→D synchronous: p99 multiplies and any link's failure fails the whole call. | Make non-critical steps async (queue); parallelize independent calls; collapse the chain (07, 13, 19). |
| **God service** | One service owns half the domain → it's the bottleneck, the deploy chokepoint, the on-call nightmare. | Split by capability and data ownership; give each service a single responsibility (16). |
| **Missing idempotency** | At-least-once delivery + retries → double charges, duplicate posts, double-applied effects. | Idempotency keys; dedupe tables; make handlers idempotent by design (10). |
| **Ignoring backpressure** | Consumer can't keep up; lag grows unbounded; eventually the producer or broker falls over. | Measure consumer lag; bound buffers; shed/throttle/scale consumers; signal upstream (07, 13). |
| **No failure handling** | Happy-path-only design; the interviewer asks "what if the queue dies?" and you have no answer. | Name failure modes *unprompted*: timeouts, retries, DLQ, circuit breakers, replicas, SPOFs (13). |
| **Big-bang rewrite** | Replace the legacy system all at once → high risk, long no-value period, frequent rollback/abandon. | **Strangler fig**: incrementally route slices to the new system behind a façade (16). |

> **The reframe:** anti-patterns are usually a *good* pattern applied where its tradeoff doesn't
> pay. CQRS is great — until it's CRUD. Microservices are great — until the domain is one team.
> Saying "I'd normally reach for X here, but given [requirement], that buys complexity we don't
> need, so I'll do the simpler thing" is one of the strongest signals you can send.

---

## Part C — The Interview Playbook

The 7-step framework from Topic 1, rendered as a *tactical script* — with phrases to actually say.
You drive; the interviewer steers the deep dive. Time-boxes are guides.

### The 45-minute script

**1. Requirements (5 min) — you drive, don't wait.**
> "Before I draw anything, let me pin down what we're building and how well it has to work.
> Functionally: [verbs]. I'll scope to X and Y, and skip Z unless you want it. Non-functionally —
> what scale are we targeting? Any p99 latency bar? Is stale data acceptable for [read path]?
> Roughly what read:write ratio?"

Lock the read:write ratio — it shapes the whole design. State your scope cut explicitly; it shows seniority and protects your clock.

**2. Estimation (3 min) — to justify, not to impress.**
> "Let me get order-of-magnitude numbers so my decisions are grounded. DAU × actions ÷ 86,400 →
> ~X write QPS, peak ~3×. Storage: writes/day × bytes × retention → ~Y/year. That tells me
> [single DB won't hold it / a cache is warranted / fan-out matters]."

The point is to *earn* the next decision. "We're at 50k write QPS → one DB won't hold it → we shard" (02, 04).

**3. API (3 min).** A handful of endpoints to pin the contract and surface hidden reqs.
> "Here's the contract: POST /x, GET /y with cursor-based pagination. Auth and rate-limiting live
> at the gateway so I won't re-litigate them later." (09)

**4. Data model (5 min).** Entities + **access patterns** — the access pattern picks the store, not reflex.
> "Main entities: A, B, C. A is read by key K, range scan on T. Given those access patterns I'd
> use [SQL for relations+txns / NoSQL for this known-key high-scale path]." (20, 03)

**5. High-level design (10 min).** Happy path end-to-end, *simple first*.
> "Client → gateway → service → DB, plus a cache for the hot read path and a queue for the async
> work. Let me walk one request through: user posts → authn → write service → DB → enqueue
> fan-out → workers update feeds." Then evolve under questioning — don't pre-optimize.

**6. Deep dives (15 min) — rounds are won here.** *Propose* the hard part.
> "The interesting challenge is [feed fan-out / dedup / consistency on the balance]. Can I go deep
> there?" Then: name the tradeoff, **pick a side, justify it.** "Fan-out on write = fast reads but
> explodes for celebrities; on read = cheap writes, slow reads. I'd do a hybrid: push for normal
> users, pull for high-follower accounts, because our reqs say read p99 < 200ms and celebrities
> are <0.1% of users." (Topic 01 §6 — naming a tradeoff and picking a side is *the* signal.)

**7. Wrap-up (3 min).** Remaining bottlenecks, failure modes, SPOFs, "with more time I'd…".
> "Open risks: the queue is a SPOF — I'd run it replicated with a DLQ. With more time I'd add
> [multi-region / a rate limiter on the write path / a read-repair job]." (13)

### Handling an UNSEEN problem (the derivation mindset)

This is the antidote to your stated weakness. You do not need to have seen the problem. You need a
procedure that turns *any* prompt into a derivation:

1. **Classify the shape.** Read-heavy or write-heavy? Point lookups or range/aggregations? Strong or eventual consistency? Bursty or steady?
2. **Estimate to find the bottleneck.** The numbers tell you which axis is hard (reads, writes, storage, fanout, latency).
3. **Map the bottleneck to a block.** Reads → cache/replica/CDN. Writes/size → shard/LSM. Spikes → queue. "Nearby" → geospatial index. (Use the Part E table.)
4. **State the consistency need per data type**, not globally. Balance = strong; feed = eventual.
5. **Stress it at 10×.** What breaks first? That's your next deep dive.
6. **Find the SPOF.** Everything load-bearing gets redundancy or a fallback.

> **Say this out loud when you hit something new:** "I haven't built exactly this, so let me reason
> from the access patterns. It's read-heavy with a 'nearby' query and spiky writes, so I'm reaching
> for a CDN, a geospatial index, and a queue — let me justify each." Reasoning *visibly* from
> primitives is a stronger signal than recognizing a memorized answer.

### Handling "now scale this 10×"

A standard pivot. Don't panic-redesign — find what breaks first and address *that*.

1. "Let me recompute. 10× of [50k QPS] is 500k QPS — that's the number that matters."
2. Name the *first* thing to break: usually the single DB (→ shard, 04), then the cache (→ hot keys/sharded cache, 06), then fanout (→ hybrid, 15), then a synchronous chain (→ async, 07).
3. Change *one axis at a time* and say what it costs: "Sharding fixes write volume but now cross-shard queries are scatter-gather and rebalancing is painful — I accept that because the alternative doesn't fit."
4. Watch for new failure modes scale introduces: retry storms, hot shards, thundering herd, coordinator overload.

### Recovering when you're stuck

Silence is the enemy; *visible* reasoning is recoverable. When you blank:

- **Narrate the framework step you're on.** "Let me come back to access patterns to unstick this."
- **Restate the bottleneck.** "The hard part here is X — let me enumerate the options: A, B, C, and their tradeoffs." Listing options buys time *and* shows structure.
- **Estimate.** Doing the math reliably reveals the next move and looks deliberate.
- **Simplify deliberately.** "Let me solve the single-node version first, then scale it." Reducing scope under pressure is a seniority signal, not a retreat.
- **Ask a sharpening question.** "Is read latency or write throughput the priority here?" Targeted (not flailing) questions are fine.

### Disagreeing with the interviewer well

Interviewers often push a suboptimal idea to see if you fold or defend. Do neither blindly:

- **Acknowledge, then reason.** "That works and it's simpler — the tradeoff is [X]. Given our requirement [Y], I lean the other way, but if [Z] mattered more I'd take your approach."
- **Make it about requirements, not ego.** Tie the disagreement to a stated NFR.
- **Concede fast when they're right.** "Good point — that's cleaner, I'll switch." Updating on evidence is a *strong* signal, not a weak one.
- Never dig in on a losing position, and never cave instantly on a correct one. Both read as junior.

---

## Part D — Signals & Red Flags

### What separates STAFF from SENIOR

Senior gets a correct, scalable design. Staff does that *plus* the behaviors below. Concrete, observable things — do them on purpose.

| Behavior | Senior | Staff |
|---|---|---|
| **Tradeoffs** | Lists options | Names the tradeoff *and picks a side with a justification* tied to a requirement |
| **Failure modes** | Answers when asked | Surfaces them *unprompted* ("the queue is a SPOF; here's the mitigation") |
| **Cost awareness** | Ignores $ | Mentions infra cost / dollar implications of choices ("fan-out on write is cheap to read but burns storage") |
| **Scoping** | Builds what's asked | Explicitly cuts scope and says why ("I'll skip ranking; flag if you want it") |
| **Driving** | Waits for steering | Proposes the interesting deep dive before being asked |
| **Restraint** | Applies patterns | Knows when *not* to (declines microservices/CQRS when overkill, and says why) |
| **Estimation** | Hand-waves | Uses numbers to justify the *next* decision, not to show off |
| **Evolution** | One big design | Simple design first, evolves it under load with named breakpoints |

> **The one-line test:** a staff answer sounds like *"I'll do X because Y; the cost is Z; I accept
> it because requirement W."* Every consequential decision gets that sentence.

### RED FLAGS that tank a candidate

| Red flag | Why it kills the signal | Do instead |
|---|---|---|
| **Jumping to a diagram** before requirements/estimation | You're solving an undefined problem; you can't derive | Pin functional + non-functional reqs first (Topic 01 §1) |
| **Buzzword-dropping** ("I'll use Kafka, Cassandra, Redis…") with no tradeoffs | Sounds like résumé bingo; no judgment shown | Name the tradeoff and the *because* for each choice |
| **Over-engineering** (microservices/CQRS/multi-region day one) | Complexity with no justification = poor judgment | Simplest design that meets reqs; evolve under pressure |
| **Ignoring the requirements** | You optimize the wrong axis | Tie every decision back to a stated NFR |
| **No estimation** | Every scale decision is then unjustified guessing | Always run the napkin math (02) |
| **Hand-waving the hard part** ("and then we just sync the data") | The hard part is the whole interview | Go *deeper* exactly where it's hard; that's where the points are |
| **One-way doors with no failure story** | No redundancy, no DLQ, no timeouts | Name SPOFs and mitigations unprompted (13) |
| **Not driving / waiting to be told what to do** | Reads as junior | Propose the deep dive; manage your own clock |

---

## Part E — "If you hear X, reach for Y" (rapid-fire mapping)

The recognition vocabulary. The trigger word/phrase → the primitive → the deep-dive doc. This is the
fastest path from an unseen prompt to a starting move.

| If you hear… | Reach for… | Doc |
|---|---|---|
| "nearby", "within N miles", "closest driver" | Geospatial index (geohash / quadtree / S2) + scatter-gather | 12 |
| "real-time", "live", "push updates" | WebSockets / SSE / long-poll; streaming | 15 |
| "no double-charge", "exactly once", "money" | Idempotency key + ledger (append-only) + saga for cross-service | 10 |
| "trending", "top-K", "most popular" | Count-min sketch + stream processing (heavy hitters) | 18 |
| "autocomplete", "typeahead", "prefix" | Trie (+ precomputed top-K per prefix) | 11 |
| "search", "full-text", "filter by text" | Inverted index (Elasticsearch), fed via CDC/queue | 11 |
| "feed", "timeline", "followers" | Fan-out (write/read/hybrid) + cache | 15 |
| "unique visitors", "approx count", "cardinality" | HyperLogLog | 18 |
| "is it in the set / seen before" (cheap, approx) | Bloom filter | 06 |
| "rate limit", "throttle", "quota", "abuse" | Token bucket / sliding window at the gateway | 09 |
| "spiky", "bursty", "absorb load", "decouple" | Message queue / log + competing consumers | 07 |
| "collaborative editing", "offline edits merge" | CRDTs / OT | 05 |
| "leaderboard", "ranking by score" | Sorted set (Redis ZSET) / sharded + merge | 06, 11 |
| "global", "low latency worldwide" | CDN + multi-region + edge; geo-routing | 14, 19 |
| "huge files", "video", "images", "upload" | Object store (S3) + presigned URLs + claim-check on the bus | 14 |
| "audit log", "history", "who changed what" | Event sourcing / append-only log | 16, 20 |
| "consistent config / one leader / locks" | Consensus (Raft) / etcd / ZooKeeper / leader election | 08 |
| "read-heavy", "100:1 reads" | Cache-aside + read replicas + CDN | 06, 05 |
| "write-heavy", "high ingest", "metrics/logs" | LSM-tree store (Cassandra) + sharding + queue | 03, 04, 07 |
| "keep search/cache/warehouse in sync with DB" | Change data capture (CDC) + outbox | 07, 10 |
| "two stores must update atomically" | Transactional outbox (not dual writes) | 10 |
| "must survive node/zone failure" | Replication + quorum + multi-AZ; name the SPOF | 05, 13 |
| "scheduled / cron / delayed jobs" | Delay queue / scheduler + idempotent workers | 07 |
| "session / login / who is this" | Token (JWT) at gateway, session store for revocation | 17 |
| "notifications to millions" | Queue + fan-out workers + DLQ + dedupe | 07, 15 |
| "analytics", "OLAP", "dashboards" | Columnar warehouse + batch/stream ETL + materialized views | 18, 20 |

> Use this in reverse during practice: take any past design and ask "what trigger words would have
> pointed me here?" That builds the reflex.

---

## Part F — Pre-Interview Final Checklist

The morning-of pass. If you can do all of these from memory, you're ready.

- [ ] I can run the **7 steps in order** and say the opening line for each (Part C).
- [ ] I have the **QPS / storage / bandwidth formulas** cold (02): `DAU × actions ÷ 86,400`, peak ≈ 3×; `writes/day × bytes × retention`.
- [ ] I can recite **CAP / PACELC** in one sentence and the six core tradeoffs (Topic 01 Part B).
- [ ] I can pick **SQL vs NoSQL from access patterns**, not reflex (20, 03).
- [ ] I can explain **fan-out write vs read vs hybrid** and pick one with justification (15).
- [ ] I can name **what a queue buys me and the problem it creates** (at-least-once → idempotency) (07, 10).
- [ ] For any prompt I can say the **trigger → primitive** mapping out loud (Part E).
- [ ] I can name **three failure modes and mitigations** for any design unprompted (13).
- [ ] I will **estimate before I optimize** and **scope before I diagram**.
- [ ] I have one **clean tradeoff sentence** ready: *"X because Y; cost is Z; acceptable because W."*

---

### Self-check before the mock (answer these from memory)
- [ ] Give five patterns and, for each, the *one situation where you would NOT use it*.
- [ ] Name five anti-patterns and the fix for each.
- [ ] What's the difference between a saga, an outbox, and an idempotency key — and when does each apply? (10)
- [ ] Walk the 7-step script with the phrase you'd actually say at each step.
- [ ] You're handed a problem you've never seen. State the six-step derivation procedure.
- [ ] "Scale this 10×" — what's your first move, and what usually breaks first?
- [ ] Name three behaviors that separate a staff answer from a senior one.
- [ ] List four red flags that tank a candidate.
- [ ] For each: "nearby", "no double-charge", "trending", "autocomplete", "two stores atomically" — name the primitive.
