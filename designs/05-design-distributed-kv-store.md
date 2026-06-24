# Design 05: Distributed Key-Value Store (Dynamo / Cassandra-style)

> **Why this one matters:** this is the *systems-heavy* design — the problem that ties together
> almost every building block at once: consistent hashing (sharding doc), leaderless replication +
> quorums + conflict resolution (replication doc), LSM storage engines + Bloom filters (databases
> doc), and gossip/failure detection (consensus doc). If you can derive this from first principles,
> you can derive almost any storage system. So I run the full 7-step framework, but I spend most of
> my budget in **Step 6 (deep dives)** — that's where this round is won.
>
> The one-line thesis I'll state up front and keep returning to: **we are building an AP system
> (Dynamo) with tunable consistency knobs.** Every decision is "stay available under partition, and
> let the operator/client dial how much consistency they pay for." Contrast that with a CP store
> (Spanner/etcd) at the end.

---

## Step 1 — Requirements (5 min) — I drive this

I'll state the buckets out loud and scope hard, because "a database" is unbounded.

**Functional**
- `get(key)` → value (or not-found). Point lookup by primary key only.
- `put(key, value)` → ack. Upsert.
- `delete(key)`.
- That's it. **Explicit non-goals I'll call out:** no SQL, no secondary indexes, no range scans across keys, no multi-key transactions, no joins. This is a *key-value* store — the access pattern is "I know the key." (If they push on range queries, I'll note that Cassandra adds an ordered clustering key *within* a partition, but cross-partition range scan is the thing consistent hashing deliberately gives up — tie to **sharding doc, Part B: hash partitioning loses range queries.**)

**Non-functional** — this is where the design is actually decided:
- **High availability** — "always writeable" is the headline Dynamo requirement. A shopping cart must accept an `add-to-cart` even during a network partition. Target 99.9%+ and **no write should be rejected because some replicas are down.**
- **Horizontal scale** — petabytes, thousands of nodes, commodity hardware. Adding a node must be cheap (no global reshuffle).
- **Partition tolerance** — non-negotiable in a multi-node distributed system; the network *will* partition. So by CAP this is a choice between C and A under partition (**replication doc, Part H**).
- **Tunable consistency** — different callers want different points on the spectrum. A cart wants AP; a config value might want stronger reads. We expose this **per-request** via quorum knobs, not as a global mode.
- **Latency** — single-digit-ms p99 for get/put. This rules out cross-region synchronous consensus on the hot path.
- **Workload shape** — write-heavy and read-heavy both supported, but the storage engine choice (LSM) leans into **write-heavy** (e.g., time-series, event logs, session/cart data).

> **The staff move in Step 1:** I'm *positioning* the system before drawing anything. "This is AP
> with tunable knobs" is the thesis that makes every later choice derivable rather than memorized.
> Per **PACELC**: under Partition we choose **A** over C; and Else (normal operation) we choose **L**
> over C — i.e., **PA/EL**. Dynamo and Cassandra are the canonical PA/EL stores.

---

## Step 2 — Estimations (3 min) — to justify, not impress

Let me pick numbers that *force* the architecture, so nothing is arbitrary.

- Say **1M write QPS + 5M read QPS** at peak (a large session/cart/feed-metadata store).
- **Data**: 100 TB of live data, replication factor **N = 3** → **300 TB** physical.
- **Per-node capacity**: commodity node holds ~2–4 TB of hot data comfortably with an LSM engine.
  300 TB / 3 TB ≈ **~100 nodes** just for capacity; more for QPS headroom → call it **a few hundred nodes** across racks/AZs.
- **Throughput per node**: 1M writes / ~200 nodes = ~5k writes/node/sec — very comfortable for an LSM engine (sequential appends), which is exactly *why* we pick LSM (**databases doc, Part B**).

> What the math *justifies*: a few hundred nodes → a single shard map in one place is a liability, and
> manual sharding is hopeless → **we need consistent hashing + automatic membership (gossip)**. The
> estimate did its job: it made sharding and decentralized membership *necessary*, not assumed.

---

## Step 3 — API design (3 min)

Deliberately tiny — the contract *is* the simplicity argument.

```
get(key, [R])            -> { value(s), context }     # context = version metadata (vector clock)
put(key, value, context) -> ack                       # context echoed back from a prior get
delete(key, context)     -> ack                       # delete = write of a tombstone

# Optional per-request consistency overrides:
get(key, consistency=R)   # R = how many replicas must respond before we answer
put(key, value, W)        # W = how many replicas must ack before we ack
```

Two contract decisions I'll flag now and justify in the deep dive:
1. **`context` (opaque version metadata) is passed back on writes.** This is the vector clock. The client does a `get`, mutates, and `put`s with the context it read. This is how we detect concurrent writes (**replication doc, Part G**).
2. **Consistency is a per-call argument (R/W), not a database-wide setting.** Same store, different durability/consistency per request. That's the "tunable knobs" thesis made concrete.

> Auth/rate-limiting/TLS terminate at the client library or a thin proxy layer — I won't re-explain
> them; see the **API-gateway doc**. The interesting surface here is the data plane.

---

## Step 4 — Data model (5 min)

- **Logical model**: opaque keys → opaque values (byte blobs, or in Cassandra's case a row keyed by a partition key with optional clustering columns *inside* the partition).
- **Access pattern**: 100% point access by key. **No requirement reads by anything but the key.** Per the **databases doc decision flow**, "known access pattern, point lookups, massive scale, eventual consistency acceptable" → **wide-column / KV NoSQL** (Cassandra/Dynamo), not SQL. The access pattern *chose* the store; I didn't reach for NoSQL by reflex.
- **What each key needs stored with it**:
  - the value,
  - a **version/causality stamp** (vector clock entries: `[(nodeId, counter), ...]`),
  - per-cell timestamps if we use LWW for sub-fields (Cassandra) ,
  - tombstone marker for deletes (with a GC grace period — you can't delete immediately or a down replica resurrects the key on repair).

> **Why I'm not normalizing anything:** there are no relations to normalize. The model is intentionally
> denormalized blobs — read speed and partition-locality over write simplicity (**framework tradeoff
> list: normalization vs denormalization**).

---

## Step 5 — High-level design (10 min) — happy path end to end

Boxes and arrows first; I'll keep it deliberately simple, then evolve under questioning.

```
                       ┌─────────────────────────────────────────────┐
   client ──get/put──▶ │  any node (acts as COORDINATOR for this key) │
   (smart client or                       │
    thin LB)                              │ hash(key) -> position on ring
                                          ▼
                          ┌──────────────────────────────────┐
                          │   the RING (consistent hashing)   │
                          │   N=3 replicas = preference list  │
                          └──────────────────────────────────┘
                              │          │           │
                            replica1   replica2    replica3   (next 3 distinct
                              │          │           │         physical nodes
                            [LSM]      [LSM]       [LSM]        clockwise)
```

**Key architectural properties of the happy path:**
- **No master.** Every node is identical and can coordinate any request. This is *leaderless replication* (**replication doc, Part A.3**). There is no leader to fail, no failover, no split-brain on the data plane — that's how we get the "always writeable" availability.
- **Any node can be the coordinator.** A smart client routes directly to a node in the key's preference list (saves a hop); a dumb client hits any node, which forwards. The coordinator drives the quorum.
- **The ring + membership is shared knowledge**, spread by **gossip** (deep dive below), not by a central config server. (Contrast: a CP store keeps the shard map in a Raft group — see Step 6's CP comparison.)

**Walking one `put` out loud:** client → coordinator → `hash(key)` → top N nodes on the ring (preference list) → coordinator writes to all N, waits for **W acks** → returns success once W respond. Reads are symmetric with **R**. Everything else (anti-entropy, hinted handoff, read-repair) is the machinery that makes this correct *despite* failures — which is the deep dive.

---

## Step 6 — Deep dives (the bulk of the round)

I'll propose the order: **(A) partitioning → (B) replication & coordinator → (C) tunable quorums → (D) availability under failure (sloppy quorum + hinted handoff) → (E) conflict resolution → (F) anti-entropy / Merkle → (G) membership / gossip → (H) the storage engine → (I) add/remove a node → (J) full read/write path → (K) CP contrast.** That's the natural dependency order: you can't talk replication before you've placed data, can't talk quorum before replication, can't talk repair before conflicts.

---

### 6A. Partitioning: consistent hashing with virtual nodes

**The problem:** a few hundred nodes, keys must map to nodes, and adding/removing a node must move *as little data as possible*. Naive `hash(key) % N` remaps almost **every** key when N changes (**sharding doc, Part C: "why naive modulo fails on resize"**) — catastrophic here because that's a full data reshuffle plus cache-coldness across the fleet.

**Consistent hashing — the ring:**
- Hash the key space onto a fixed ring, e.g. `[0, 2^128)` using a uniform hash (MD5/Murmur).
- Each node is also hashed to one or more positions on the ring.
- A key is owned by the **first node clockwise** from `hash(key)`.
- Add/remove a node → only the keys between the new node and its predecessor move. **On average only K/N keys relocate** (**sharding doc, Part C: "how many keys move"**), not all of them.

**Why naive consistent hashing isn't enough — virtual nodes:**
With one ring position per node you get two problems (**sharding doc, Part C: "virtual nodes — and why naive isn't enough"**):
1. **Uneven load** — random placement leaves big gaps; some nodes own huge arcs, others tiny. Load variance is high.
2. **Lumpy rebalancing** — when a node dies, its *entire* arc dumps onto exactly one successor (a thundering load spike on one node), and when you add a node it only steals from one neighbor.

**Fix:** give each physical node **V virtual nodes** (vnodes / tokens) — e.g. 128–256 ring positions per physical node.
- Load smooths out: each physical node owns many small scattered arcs, so by law of large numbers everyone owns ~1/M of the ring.
- **Heterogeneity**: a beefier machine just gets more vnodes → proportionally more data. Free weighting.
- **Graceful rebalancing**: when a node dies, its ~200 small arcs are spread across ~200 *different* successors, so the recovery load is shared by the whole cluster, not dumped on one neighbor. Same in reverse when you add a node: it steals a little from many nodes.

**Worked mental model (the sharding doc's worked example, applied):** 4 physical nodes, 4 vnodes each = 16 tokens on the ring. `hash("cart:42") = 0x7A...` lands at ring position 0x7A; walk clockwise to the first token; that token belongs to (say) physical node C; the next two *distinct physical* nodes clockwise are the other two replicas. Note the "distinct physical" rule — you must skip vnodes that map back to a physical node already in the preference list, or you'd put 2 of 3 replicas on the same machine and lose a replica when it dies.

> **Tradeoff stated:** vnodes cost a bigger ring/membership table and more bookkeeping, but they buy
> even load + smooth rebalancing + heterogeneity. For a few-hundred-node fleet that's an easy yes.
> This is exactly the **consistent-hashing-with-vnodes** rebalancing strategy from the **sharding doc,
> Part G**.

---

### 6B. Replication: preference list, coordinator, N replicas

- **N = replication factor** (commonly 3). Each key is stored on the **N distinct physical nodes** found by walking clockwise from the key's position — this ordered list is the **preference list**.
- **Rack/AZ awareness:** the preference list is constructed so the N replicas straddle **different failure domains** (racks, then availability zones). Otherwise a single rack power loss takes out all 3 copies. So "walk clockwise skipping nodes in the same rack until you have N distinct domains."
- **Coordinator:** the node that receives the request coordinates the replication. Typically it's the **first node in the preference list** (a smart client routes straight there); if not, it forwards. The coordinator sends the write to all N, collects acks, and applies quorum logic.
- **Leaderless, so writes go to all replicas in parallel** — no leader serialization. This is the source of both the availability *and* the conflict problem (two coordinators can accept concurrent writes to the same key — resolved in 6E).

> Cross-reference: this is **replication doc, Part A.3 (leaderless / Dynamo-style)** made concrete.
> The preference list is the leaderless analog of "which followers hold this data."

---

### 6C. Tunable consistency via quorums (N, W, R; W + R > N)

This is the knob that delivers the "tunable" half of the thesis (**replication doc, Part F**).

- **N** = replicas per key.
- **W** = replicas that must **ack a write** before the coordinator returns success.
- **R** = replicas that must **respond to a read** before the coordinator answers.

**The overlap rule:** if **W + R > N**, the write set and the read set are guaranteed to **intersect in at least one node** — so any read sees at least one replica that has the latest write. That's the formal basis for "strong-ish" reads in a leaderless system. (It is *not* full linearizability — see the caveat below.)

**Worked tunings (N = 3):**
| W | R | Property | Use it for |
|---|---|----------|-----------|
| 3 | 1 | W+R=4>3. Durable writes, fast reads, **slow/fragile writes** (all 3 must ack) | read-heavy, write-rarely config |
| 1 | 3 | Fast writes, slow reads. Write survives if *any* replica up | write-heavy ingest, logs |
| 2 | 2 | **The balanced default.** W+R=4>3, tolerates 1 node down on each path | general purpose |
| 1 | 1 | W+R=2 **< 3 → no overlap → eventual only.** Fastest, weakest | "always writeable" cart, metrics |

**The latency consequence:** the coordinator waits for the **W-th (or R-th) fastest** replica, not all N. So a single slow/GC-pausing replica doesn't stall you as long as W−1 others are quick — quorums double as a **tail-latency hedge**.

> **The honest caveat (staff-level nuance):** W+R>N gives **read-overlap**, not linearizability. Edge
> cases break strict strong consistency: a write that reaches W replicas then a *different* read
> quorum during in-flight repair, sloppy-quorum nodes (6D) that aren't in the "home" set, or a
> failed write that partially applied. Dynamo-style is honestly **eventually consistent with a high
> probability of freshness** — if you need real linearizable single-key reads, you want a CP store
> (6K). I volunteer this; pretending quorum = strong is a classic mid-level mistake.

---

### 6D. Availability under failure: sloppy quorum + hinted handoff

Strict quorum says "W of the *top-N home nodes* must ack." But if one home node is down, a strict W=2 of N=3 still works — and if two are down, the write fails. Dynamo refuses to fail the write (availability is the headline requirement). So:

- **Sloppy quorum:** if a home replica is unreachable, the coordinator walks **further down the ring** to the next healthy node and stores the write there *temporarily*, counting it toward W. The write succeeds as long as **any N healthy nodes** can be found — not necessarily the *home* N. This is what makes it "always writeable."
- **Hinted handoff:** the stand-in node stores the data with a **hint** ("this really belongs to node X, who was down"). It is not the permanent owner. When it gossips that X is back, it **hands the data off** to X and deletes its local copy.

> **Tradeoff stated:** sloppy quorum trades a *consistency guarantee* for availability — during the
> partition, a read quorum of home nodes might miss the write that's sitting on a stand-in, so you can
> read stale. We accept it because "accept the write" beats "reject the cart add." Hinted handoff
> bounds how long that staleness lasts. This is the **replication doc, Part F** mechanism that keeps
> leaderless writes available. **Caveat:** hints can be lost if the stand-in *also* dies before
> handoff — which is exactly why we *also* need anti-entropy (6F) as a backstop. Hinted handoff
> handles **temporary** failures; Merkle anti-entropy + read-repair handle the cases hints miss and
> **permanent** failures.

---

### 6E. Conflict resolution: vector clocks, read-repair, LWW tradeoff

Leaderless + sloppy quorum means **two writes to the same key can be accepted concurrently** by different coordinators with no ordering between them. We need to detect and resolve that.

**Why not just use a timestamp (last-write-wins)?**
- **LWW** = keep the value with the highest wall-clock timestamp; discard the rest. Dead simple, no metadata, what Cassandra does per-cell.
- **The cost:** **silent data loss** under concurrent writes — the "loser" write is discarded even though it was a legitimate concurrent update, and clock skew across nodes makes "highest timestamp" arbitrary. Two users adding different items to a cart at the same instant → one item vanishes (**replication doc, Part G: LWW**).
- Use LWW only when **losing a concurrent write is acceptable** (sensor readings, session refresh, "last setting wins" config). Be explicit.

**Vector clocks / version vectors (the precise tool):**
- A value carries a vector clock: `{nodeA: 3, nodeB: 1}` = "this version reflects A's 3rd update and B's 1st."
- On write, the coordinator increments its own entry. On read, the client gets the value(s) + the clock as `context`, and echoes the context back on the next `put`.
- **Comparison rule:** clock X **descends from** Y if every entry of X ≥ the corresponding entry of Y. Then X is strictly newer → keep X, drop Y. If **neither descends from the other**, they're **concurrent → a genuine conflict** (**replication doc, Part G: vector clocks**).
- **What happens to concurrent versions:** they are **both retained as siblings**. On the next `get`, the store returns **both values**, and the **client resolves the conflict** (e.g., union the two carts), then writes back a single reconciled value with a clock that descends from both.

> **Why the *client* resolves, not the store:** the store has no idea what the value *means* — it's
> opaque bytes. Only the application knows "two carts → merge by union," or "two profile edits → ask
> the user." Dynamo's shopping-cart example deliberately **unions** sibling carts: that's why a deleted
> item can reappear (a removed-then-re-added item survives the merge) — the famous Dynamo
> resurrection. Semantic reconciliation is an application concern; the store's job is only to *detect*
> the conflict and *preserve* the candidates. This is the **replication doc, Part G** point that
> conflict resolution can't be fully automated without app semantics. (CRDTs — Part G — are the
> automatable special case: a grow-only set or counter merges deterministically with no app logic. If
> the value type is a CRDT, the store *can* merge. That's the modern refinement.)

**Read-repair (resolving lazily on the read path):**
- During a read, the coordinator collects R responses. If they disagree (one replica has a stale version per the vector clock), the coordinator returns the freshest to the client **and asynchronously writes the fresh version back** to the stale replicas.
- This makes **frequently-read keys self-heal** for free on the hot path — no extra scan needed. Cold keys rely on anti-entropy instead.

---

### 6F. Anti-entropy: Merkle trees for replica sync

Read-repair only fixes keys someone reads; hinted handoff can lose hints. We need a **background process to reconcile two replicas' entire datasets** — that's anti-entropy. The naive approach (ship every key and compare) moves terabytes; unacceptable.

**Merkle tree — minimizing data transfer:**
- Each replica builds a **hash tree** over its key range: leaves = hashes of buckets of keys (or individual rows); each internal node = hash of its children; the **root** summarizes the whole range.
- Two replicas compare **roots first**. If the roots match → the datasets are identical → **transfer nothing**. Done.
- If roots differ, they exchange the **next level** of hashes and recurse **only into the subtrees that differ**. Matching subtrees are skipped entirely.
- They drill down until they reach the specific **leaf buckets that diverge**, and exchange **only those keys**.

> **Why this is the win:** divergence between two healthy replicas is usually tiny (a handful of keys
> that missed a write). Merkle trees make the comparison cost **O(log n) levels and proportional to the
> *number of differences*, not the dataset size.** If they're in sync you pay one root-hash compare.
> This is the textbook "minimize data transfer over the network for replica reconciliation" use of a
> hash tree. **Cost to name:** rebuilding the tree on range changes (node join/leave reshuffles ranges
> and forces tree rebuilds), which is why frequent membership churn hurts anti-entropy.

---

### 6G. Failure detection & membership: gossip

A few hundred nodes, no master — so "who is in the cluster and who is alive?" must be **decentralized**. A central monitor would be a single point of failure and a scaling bottleneck (and would drag us toward CP).

**Gossip protocol:**
- Each node keeps a membership table: `{nodeId → (state, heartbeat counter, version)}`.
- Every ~1s each node picks a **random peer** and exchanges (gossips) its table. Each node merges in newer entries (higher heartbeat/version wins).
- Information spreads **epidemically** — `O(log N)` rounds to reach the whole cluster. Robust: no node is special, no single point to fail.
- **Failure detection** rides on the same channel: if a node's heartbeat counter stops advancing across gossip rounds, peers mark it **suspect**, then **down** after a threshold. A **φ-accrual failure detector** (used by Cassandra) outputs a *suspicion level* tuned to observed network jitter instead of a hard timeout — fewer false positives on a flaky network.
- Membership changes (join/leave) and the ring/token assignments propagate the same way, so the **shard map is emergent gossip state**, not a config-server lookup.

> **Tradeoff stated:** gossip gives a self-healing, no-SPOF membership view, but it's **eventually
> consistent** — two nodes can briefly disagree about who owns a token mid-churn, causing transient
> misroutes (handled by forwarding / sloppy quorum). We accept eventual membership because the
> alternative (a strongly-consistent membership service via Raft/ZooKeeper) reintroduces a coordination
> dependency on the data path. **This is the deliberate "we don't need consensus here" call from the
> consensus doc, Part H** — Dynamo specifically avoids consensus for membership. (Note: some systems
> *do* put membership in a small Raft group for stronger guarantees — a legitimate hybrid; I'd mention
> it as the alternative.)

---

### 6H. The per-node storage engine: LSM-tree + commit log + memtable + SSTables + Bloom filters

Each node is itself a single-machine write-optimized store. Given our write-heavy estimate, the engine is an **LSM-tree**, not a B-tree (**databases doc, Part B**).

**Write path on a node:**
1. **Append to the commit log (WAL)** on disk — sequential, fast, **durability** (crash recovery; this is the **commit log / WAL** block from the framework toolkit). If the node crashes, replay the log.
2. **Update the memtable** — an in-memory sorted structure (red-black tree / skip list). The write returns once log + memtable are done.
3. When the memtable fills, **flush it to disk as an immutable, sorted SSTable** (Sorted String Table) and start a fresh memtable. The corresponding commit-log segment can now be discarded.

**Why this is fast for writes:** every disk write is a **sequential append** (the WAL and the SSTable flush) — no random in-place updates, no read-before-write. That's the LSM advantage and exactly why we chose it for a write-heavy KV store.

**Read path on a node:**
1. Check the **memtable** (newest data).
2. Then check **SSTables**, newest first. A key may live in several SSTables (it was updated repeatedly); the newest wins.
3. **Problem — read amplification:** a key that isn't present would require touching *every* SSTable on disk. Fix with **Bloom filters**.

**Bloom filters (avoid disk reads):**
- Each SSTable has an in-memory Bloom filter: a probabilistic set membership structure.
- "**Definitely not present**" or "**maybe present**" — **no false negatives**. So if the filter says *not present*, we **skip that SSTable entirely with zero disk I/O**. We only hit disk for the SSTables the filter says *maybe*.
- This turns "scan all SSTables for a missing/rare key" into "touch the one or two SSTables that might have it" — a huge win for the common case (**framework toolkit: Bloom filter; databases doc**).
- Plus a sparse **index + summary** per SSTable to find the key's offset without scanning.

**Compaction:** a background process merges SSTables, drops superseded versions and tombstones (after the GC grace period), and bounds read amplification.

> **Tradeoff stated (databases doc, Part B side-by-side):** LSM gives blazing sequential writes and
> good compressed storage, at the cost of **read amplification** (multiple SSTables) — mitigated by
> Bloom filters + compaction — and **compaction CPU/IO cost** (write amplification, spiky latency
> during big compactions). For a write-heavy KV store that's the right trade; for a read-heavy,
> update-in-place, range-scan workload a B-tree (Postgres/InnoDB) would be better. I'm choosing LSM
> *because* of the write-heavy non-functional requirement, not by reflex.

---

### 6I. Membership changes: node add / remove, temporary vs permanent

**Add a node (scale out):**
- New node gets **V vnodes/tokens**, claims those arcs on the ring, gossips its arrival.
- It now owns ranges previously owned by several existing nodes (vnodes spread the source). Those nodes **stream the relevant SSTable ranges** to the newcomer (bulk transfer, then catch up deltas).
- Because of vnodes, the new node draws **a little data from many nodes in parallel** → fast, even bootstrap, no single donor hot-spotted (**sharding doc, Part G**).
- Until streaming completes, the old owners keep serving + forwarding so there's no availability gap.

**Remove a node (decommission):**
- The departing node streams its ranges to the new owners (its successors on the ring), then leaves; gossip propagates the new token map.

**Temporary vs permanent failure — the key distinction:**
- **Temporary** (reboot, GC pause, brief network blip): don't rebalance! Moving terabytes for a 30-second blip is wasteful and itself destabilizing. Instead, **sloppy quorum + hinted handoff (6D)** cover the gap; when the node returns, it gets its hints + a Merkle anti-entropy pass to catch up. This is why Dynamo separates the two.
- **Permanent** (disk dead, decommission): operator (or automation after a long threshold) **removes the node from the ring**, and its ranges are re-replicated from the surviving copies via streaming + Merkle repair to restore N copies.

> **Staff nuance:** automatic permanent-failure detection is dangerous — a flapping network looks
> like a dead node, and auto-rebalancing on a false positive triggers a data-movement storm
> (correlated load that can cascade). Dynamo deliberately makes **permanent membership changes
> explicit/operator-driven** (or gated behind a long, conservative timeout), while **temporary**
> failures are handled invisibly by hinted handoff. Naming this temp-vs-permanent split is a strong
> signal.

---

### 6J. The full read/write path, end to end

**`put(key, value, context)`:**
1. Client (smart) hashes the key, picks the coordinator from the preference list; or hits any node which forwards.
2. Coordinator increments its vector-clock entry, stamps the new version.
3. Coordinator sends the write to the **top N healthy nodes** (sloppy: substitutes stand-ins with hints for any down home node).
4. Each replica **appends to its commit log + memtable** (6H) and acks.
5. Coordinator returns success once **W acks** arrive (doesn't wait for all N → tail-latency hedge).
6. Stand-in nodes later **hand off** to recovered home nodes; anti-entropy backstops.

**`get(key, R)`:**
1. Coordinator requests the value from N replicas (or the fastest); waits for **R** responses.
2. Compares the returned **vector clocks**:
   - one descends from all others → return it.
   - concurrent siblings → return **all** to the client for semantic resolution.
3. **Read-repair**: async-write the freshest version back to any stale replica that responded.
4. With **W + R > N**, the R responses include at least one node from the last write set → high probability of freshness (modulo the sloppy-quorum/in-flight caveat in 6C).

> Walking this end-to-end out loud, and pausing at each step to name *which failure it tolerates*
> (W<N → survives slow/dead replicas; sloppy quorum → survives a down home node; read-repair +
> Merkle → survives missed writes; vector clocks → survives concurrent writes), is the move that
> demonstrates you understand the system as a *whole*, not a bag of features.

---

### 6K. How this differs from a CP store (Spanner / etcd), and when to choose each

The contrast is the cleanest way to prove you understand *why* the AP design is the way it is.

| | **AP store (this design: Dynamo / Cassandra)** | **CP store (Spanner / etcd / ZooKeeper / CockroachDB)** |
|---|---|---|
| CAP under partition | stays **Available**, may serve stale / accept conflicting writes | stays **Consistent**, **rejects** writes on the minority side |
| Replication | **leaderless**, write to all, quorum acks | **leader per shard via Raft/Paxos**, leader serializes writes (**consensus doc, Part C**) |
| Reads | quorum, eventually consistent (high-prob fresh) | **linearizable** (read from leader / lease / read-index) |
| Write availability during partition | **writeable on both sides** (→ conflicts to reconcile) | only the side with a **majority quorum** can write |
| Conflict resolution | needed (vector clocks / LWW / client merge) | **none** — consensus imposes a single total order, no conflicts arise |
| Membership | **gossip**, eventually consistent | **strongly-consistent** config in the Raft group |
| Latency | low; no consensus on hot path | higher; consensus round-trip per write (Spanner adds TrueTime commit-wait) |
| Use it for | carts, sessions, feeds, metrics, "always-on" writes where stale/merge is OK | bank balances, inventory/seat-booking, locks, leader election, config, anything needing **no two conflicting writes** |

> **The decision rule I'd state:** *Can two simultaneously-accepted writes ever be reconciled
> after the fact?* If yes (merge carts, last-temperature-wins) → **AP, this design**. If a conflict is
> a correctness violation that can't be undone (double-spend, double-sell a seat) → **CP**, pay the
> consensus latency, accept that the minority partition can't write. The **consensus doc, Part H/J
> ("do I need consensus?")** is exactly this: don't pay for consensus unless a conflict is
> unacceptable. Most "always available" KV use cases genuinely don't need it — which is the whole
> reason Dynamo exists.

---

## Step 7 — Wrap-up (3 min): tradeoffs, tuning, failure modes

**The thesis held:** AP with tunable knobs. Every piece serves "stay available under partition, dial consistency per request."

**Remaining bottlenecks / failure modes to name before asked:**
- **Compaction storms** — background SSTable merges cause latency spikes; schedule/throttle them and over-provision IO headroom.
- **Hot keys** — consistent hashing spreads *keys* evenly but a single ultra-hot key still lands on N nodes; mitigate with client-side caching or key-splitting (**sharding doc, Part E: hot key**). Consistent hashing solves hot *shards*, not a hot *single key*.
- **Lost hints** — if a stand-in dies before handoff, the write survives only via the W replicas that did persist + Merkle repair; if W was 1, durability is genuinely at risk. Tune W up for durability-critical data.
- **Membership churn** thrashes Merkle trees and streaming; avoid rapid auto-rebalance (the temp-vs-permanent split).
- **Sibling explosion** — pathological clients that don't reconcile siblings can accumulate versions; cap sibling count and force resolution.

**What I'd tune for different workloads (the "tunable" payoff):**
- **Shopping cart / always-writeable:** `N=3, W=1, R=1` (or sloppy), client-side merge, accept resurrection. Availability over freshness.
- **Read-heavy config that rarely changes:** `W=3, R=1` (or `W=N`) — pay on the write, get cheap fresh reads.
- **Balanced general purpose:** `N=3, W=2, R=2` — survive one node down on each path.
- **Write-heavy ingest / metrics:** `W=1, R=1`, LWW conflict resolution, lean on LSM sequential-append throughput.
- **Strong correctness need (money, inventory):** **don't use this store** — reach for a CP store (Spanner/CockroachDB). Knowing when to walk away from your own design is the senior move.

**With more time I'd add:** cross-region replication topology (local quorum per region + async cross-region, à la Cassandra `LOCAL_QUORUM`), CDC out to a search index / analytics (**search doc**), tunable read/write consistency *per keyspace*, and backpressure/load-shedding on coordinators (**resilience doc**).

---

## What made this staff-level

- **Stated a thesis up front (AP + tunable knobs / PA-EL)** and derived every decision from it instead of listing features. Positioning > enumeration.
- **Drove requirements and let estimation *force* the architecture** (a few hundred nodes ⇒ consistent hashing + gossip became necessary, not assumed).
- **Named the precise tradeoff on every block and picked a side:** vnodes vs uneven load; LWW vs vector clocks vs CRDTs; sloppy quorum vs staleness; LSM vs B-tree; gossip-eventual-membership vs Raft-membership.
- **Volunteered the honest caveats** mid-level candidates miss: quorum overlap ≠ linearizability; Dynamo cart resurrection from sibling-union; hinted-handoff hint loss; auto-rebalance storms on false-positive failure detection.
- **Connected single-node and distributed concerns** — most candidates do *either* the ring *or* the LSM engine; tying both (and how Bloom filters + Merkle trees + compaction interlock) shows full-stack storage depth.
- **Closed with the CP contrast and a decision rule** — knowing exactly *when not to build this* (use Spanner/etcd) is the strongest seniority signal.

---

## Self-check (answer from memory before the mock)

- [ ] Why does naive `hash % N` fail and how does the ring fix it? How many keys move on add/remove?
- [ ] Two reasons plain consistent hashing isn't enough, and how vnodes fix both.
- [ ] State the quorum overlap rule and *why* W+R>N gives read-overlap — and why that's still **not** linearizable.
- [ ] Difference between **sloppy quorum** and **hinted handoff**, and which failure class each targets.
- [ ] When is **LWW** acceptable vs when do you need **vector clocks**? Why must the **client** resolve siblings? What's the Dynamo cart-resurrection bug?
- [ ] How does a **Merkle tree** minimize data transfer during anti-entropy? What's the cost on membership churn?
- [ ] Why **gossip** for membership instead of a config server — and which doc-rule ("do I need consensus?") justifies it?
- [ ] Walk the **LSM write path** (WAL → memtable → SSTable) and read path; what do **Bloom filters** save you?
- [ ] **Temporary vs permanent** failure — what handles each, and why not auto-rebalance on temporary?
- [ ] Give the AP-vs-CP decision rule in one sentence, and name a workload for each.
- [ ] Pick N/W/R for: a shopping cart, read-heavy config, write-heavy metrics.
