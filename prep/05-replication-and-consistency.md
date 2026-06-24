# Topic 5: Replication & Consistency Models Deep Dive

> **Why this topic separates staff from senior:** Almost everyone can say "use read replicas" or
> "Cassandra is eventually consistent." The staff-level move is to *derive* the right consistency
> for a feature from its requirements, name the exact anomaly your design permits, and state the
> price you're paying. "It's eventually consistent" is a non-answer. "Reads from the follower can
> violate read-your-own-writes, so I route the user's own reads to the leader for 1s after a write"
> is a staff answer. This doc gives you the vocabulary to be that precise in the room.

---

## Part A — Replication Topologies

Replication = keeping a copy of the same data on multiple nodes. You do it for three reasons:
**durability** (survive a node loss), **availability** (serve when a node is down), and **read
scaling** (spread reads across copies). The moment you have more than one copy, you have a
*consistency* problem: the copies can disagree. Everything in this doc is about managing that
disagreement.

There are exactly three topologies. Know all three cold.

### 1. Single-leader (leader-follower / primary-replica / master-slave)

One node is the **leader**. All writes go to the leader. The leader appends each write to its
replication log (WAL / binlog / oplog) and streams it to **followers**, which apply it in the same
order. Reads can go to the leader *or* the followers.

- **Writes:** client → leader only. Leader assigns an order, persists, then ships the change.
- **Reads:** leader (always fresh) or any follower (possibly stale — replication lag).
- **Failure handling:** follower dies → reads route elsewhere, no big deal. Leader dies →
  **failover** (see Part D), the hard part.
- **Conflict potential:** *none on write.* There is a single point that orders all writes, so two
  conflicting writes can never both "win" — the leader serializes them. This is the killer feature
  of single-leader and the reason it's the default.
- **When to use:** the overwhelming default. Read-heavy workloads (add followers for read scale),
  anything that wants a simple consistency story, OLTP databases. Postgres, MySQL, MongoDB
  (replica set), most relational deployments.

> **Say this in the room:** "I'll start single-leader because it gives me a serialization point for
> free — no write conflicts ever. My only consistency exposures are replication lag on follower
> reads and the failover window. I'll address both explicitly." That sentence alone signals you
> understand the topology, not just its name.

### 2. Multi-leader (master-master)

Multiple leaders, each accepting writes, each replicating to the others. Usually used across
**datacenters** (one leader per region) or for **offline-capable clients** (each device is a leader).

- **Writes:** client → nearest leader. That leader replicates async to the others.
- **Reads:** local leader (low latency).
- **Failure handling:** great — a region can keep accepting writes while others are unreachable.
- **Conflict potential:** **high and unavoidable.** Two leaders can accept conflicting writes to the
  same key concurrently. There is no single serialization point, so you *must* have a conflict
  resolution strategy (Part G). This is the price of multi-leader.
- **When to use:** multi-region writes where cross-region write latency is unacceptable; collaborative
  offline apps (calendars, note-taking — each device writes locally, syncs later). Avoid it unless
  you genuinely need geo-distributed writes; the conflict tax is real.

> **Trap:** candidates reach for multi-leader to "scale writes." It does *not* scale write throughput
> on a single key — it scales *write locality* across regions. If you need raw write throughput,
> you want sharding (Topic on partitioning), not multi-leader.

### 3. Leaderless (Dynamo-style)

No leader. The client (or a coordinator node on its behalf) writes to *many* replicas and reads from
*many* replicas, using **quorums** to get consistency. Dynamo, Cassandra, Riak, ScyllaDB.

- **Writes:** client sends the write to *all* N replicas (or a coordinator does). Waits for **W**
  acks before calling it successful.
- **Reads:** client reads from multiple replicas, waits for **R** responses, takes the newest
  (by version/timestamp). Stale replicas get repaired (**read repair** + background **anti-entropy**).
- **Failure handling:** excellent. No failover step — there's no leader to lose. A down replica just
  means W or R is satisfied by others. Optionally **sloppy quorums + hinted handoff** (Part F) keep
  writes flowing even when the "home" replicas are down.
- **Conflict potential:** **high.** Concurrent writes to different replicas → divergent versions.
  Resolved with version vectors or LWW (Part G).
- **When to use:** high availability and write-availability are paramount, you can tolerate eventual
  consistency, and you want no failover-induced downtime. Shopping carts, sensor/metrics ingest,
  activity feeds.

| Topology | Write path | Conflicts? | Failover needed? | Typical use |
|---|---|---|---|---|
| Single-leader | All to leader | Never (one ordering point) | Yes — the hard part | Default OLTP, read scaling |
| Multi-leader | Each leader, async cross-replicate | Yes — must resolve | Per-leader, less critical | Multi-region writes, offline apps |
| Leaderless | To W of N replicas | Yes — quorum + versioning | No leader to lose | Max availability, Dynamo-style |

---

## Part B — Sync vs Async vs Semi-Sync Replication

This applies mostly to single-leader (and the per-link choice in multi-leader). It's the
**durability-vs-latency** dial.

- **Synchronous:** leader waits for the follower to confirm before acking the client. Guarantees the
  follower has the write → no data loss if the leader dies. **Cost:** client latency is bounded by
  the slowest follower, and if that follower hangs, *writes stall entirely*. Pure sync to all
  followers is almost never used — one slow node halts the cluster.
- **Asynchronous:** leader acks the client immediately, ships to followers in the background.
  **Cost:** if the leader dies before the write propagates, that write is **lost** — the client was
  told "success." Fast and resilient to slow followers, but weak durability.
- **Semi-synchronous:** the practical middle. Leader waits for *one* follower (or a quorum) to
  confirm, the rest are async. Guarantees the write survives on at least two nodes without being
  hostage to *every* follower. This is what most production systems actually run (MySQL semi-sync,
  Postgres `synchronous_commit` with a quorum set).

| Mode | Latency | Durability on leader crash | Failure sensitivity |
|---|---|---|---|
| Sync (all) | Worst (slowest follower) | Strong | A single slow/dead follower stalls writes |
| Async | Best | Weak — recent writes can be lost | None — writes never block |
| Semi-sync (≥1) | Moderate | Strong enough (≥2 copies) | Tolerates failures beyond the quorum |

> **The one-liner:** "Async trades durability for latency; sync trades latency (and write
> availability) for durability; semi-sync buys most of the durability for a fraction of the latency
> cost — it's my default for anything that matters."

### Replication lag and why you should fear it

With async replication, followers are behind the leader by some **lag** — usually milliseconds, but
seconds-to-minutes under load, large transactions, or network hiccups. Lag is invisible until a user
hits one of the anomalies below. **Lag is not a bug; it's the cost of async.** Your job is to know
which anomalies your design exposes and to fix the ones that matter.

---

## Part C — Replication Lag Anomalies and the Read Guarantees that Fix Them

Three classic anomalies. For each: what the user sees, and the cheap fix. This is a frequent
deep-dive because it's where "read replicas" bites people.

### 1. Read-your-own-writes (read-after-write)

- **Anomaly:** user updates their profile (write → leader), then immediately reloads (read → a lagging
  follower that hasn't gotten the write). They see their *old* profile and think the save failed.
- **Fix:** route a user's reads to the leader for things they may have just written. Heuristics:
  - For data the user *can* edit, read from the leader (e.g. their own profile).
  - For a short window after a write (say 1s, or until the follower's position ≥ the write's log
    position), pin that user's reads to the leader or to a sufficiently-caught-up follower.
  - Track the write timestamp/log-position client-side and require any serving replica to be at
    least that fresh.

### 2. Monotonic reads

- **Anomaly:** user reads once from a caught-up follower (sees a new comment), refreshes, hits a
  *more-lagged* follower, and the comment **disappears**. Time appears to move backward.
- **Fix:** ensure each user always reads from the *same* replica (e.g. hash userID → replica). They
  may see stale data, but never *less* fresh than what they already saw. Monotonic reads is *weaker*
  than read-your-own-writes and cheaper to provide.

### 3. Consistent prefix reads

- **Anomaly:** ordering violation across items. Observer sees an *answer* before the *question* it
  replies to, because the two writes propagated via different partitions at different speeds. The
  causal order is broken.
- **Fix:** ensure causally-related writes go to the same partition (so they're ordered together), or
  track causal dependencies explicitly. This is primarily a problem in *partitioned* systems where
  each partition replicates independently.

| Anomaly | What the user sees | Guarantee that fixes it | Cheapest mechanism |
|---|---|---|---|
| Read-your-own-writes | "My own edit vanished" | Read-after-write consistency | Pin user's reads to leader / fresh replica after a write |
| Reads moving backward | "A comment I saw is gone" | Monotonic reads | Sticky reads — same user → same replica |
| Effect before cause | "Reply shown before the message" | Consistent prefix reads | Same-partition for causally related writes |

> **Say this in the room:** "Read replicas give me read scale, but they expose three lag anomalies.
> For this feature I care about read-your-own-writes — so I'll route the user's own reads to the
> leader for a short window after they write — but I'm fine with stale reads of *other* people's data."

---

## Part D — Leader Failover, Split-Brain, and Fencing

Failover is single-leader's Achilles' heel. When the leader dies, you must promote a follower.

**The steps:**
1. **Detect** the leader is dead (timeout on heartbeats — and you can't distinguish "dead" from "slow
   network," which is the root of all the danger).
2. **Choose** a new leader (election, ideally via a consensus system like Raft/ZooKeeper, or by
   most-caught-up follower).
3. **Reconfigure** the system so writes go to the new leader and the old leader (if it comes back)
   becomes a follower.

**The dangers:**

- **Lost writes (async failover):** the old leader had acked writes that hadn't replicated. The new
  leader never saw them. If the old leader rejoins, those writes are typically **discarded** to match
  the new leader — silent data loss. (GitHub had a famous incident where this corrupted data because
  the lost writes were autoincrement IDs reused by the new leader.)
- **Split-brain (two leaders):** the old leader didn't actually die — it was network-partitioned or
  GC-paused. It comes back believing it's still leader while a new leader is also accepting writes.
  Now you have **two leaders accepting conflicting writes**, which a single-leader system has no way
  to reconcile. This is the catastrophe.

**Fencing tokens — the fix for split-brain.** Every time a new leader is elected, it gets a
monotonically increasing token (epoch number). All downstream resources (the storage, a lock service)
**reject any request carrying an older token**. So when the zombie old leader wakes up and tries to
write with its stale token, the storage refuses it. The token, not the leader's belief about itself,
is the source of truth.

```
Leader A elected with token=33, holds a lock, then GC-pauses.
Lock expires; Leader B elected with token=34, starts writing.
A wakes up, tries to write with token=33.
Storage: "I've already seen token=34. Rejecting 33." → split-brain averted.
```

> **The trap interviewers set:** "What if the old leader comes back?" The wrong answer is "we'd
> detect it." The right answer is **fencing tokens** — you make it *impossible* for the stale leader
> to do damage, rather than relying on detection. Tie failover/election to a real consensus service
> (ZooKeeper/etcd/Raft) so you're not hand-rolling leader election (which is famously easy to get
> wrong).

There's also a **timeout tuning** tradeoff: short failover timeout → fast recovery but false
positives (you fail over a leader that was just briefly slow, causing needless churn and possible
cascading failures). Long timeout → fewer false positives but longer downtime. There's no free lunch.

---

## Part E — The Consistency Spectrum

Lay this out as a ladder from strongest (most expensive) to weakest (cheapest/most available).
Stronger = easier to reason about + more coordination + worse availability/latency.

### Strong / Linearizable (the strongest single-object guarantee)

- **Guarantee:** the system behaves as if there is *one single copy* of the data and every operation
  takes effect **atomically at some instant** between its start and completion. Once a write
  completes, *every* subsequent read (by anyone) sees that value or a newer one. There is a single
  global real-time order.
- **Cost:** requires coordination on every operation → higher latency, and under a network partition
  you **must** sacrifice availability (CAP). Cross-region linearizability is brutally slow.
- **Where it shows up:** leader election & locks (ZooKeeper/etcd — they *are* linearizable stores),
  uniqueness constraints, "claim this username," anything where a stale read is a correctness bug.
  Spanner gets linearizability with TrueTime + GPS/atomic clocks.

### Sequential consistency

- **Guarantee:** all operations appear in *some* single total order, and each process's operations
  appear in that order — but the total order need **not** match real time. Slightly weaker than
  linearizable (no real-time recency requirement).
- **Where:** more of a theory/multiprocessor-memory concept; rarely the headline guarantee of a
  distributed DB, but worth naming to show you know it sits between linearizable and causal.

### Causal consistency

- **Guarantee:** operations that are **causally related** (one could have influenced the other) are
  seen by everyone in the same order. Concurrent (unrelated) operations may be seen in different
  orders by different observers. This is exactly enough to kill the "reply before message" and "effect
  before cause" anomalies.
- **Cost:** much cheaper than linearizable — *available under partition* — yet preserves the ordering
  humans actually notice. Tracked with version vectors / dependency metadata.
- **Where:** the sweet spot for many social/collaborative features; the strongest model achievable
  while staying available under partition (proven result — causal+ is the ceiling for AP systems).

### Eventual consistency

- **Guarantee:** the weakest useful one — *if writes stop, all replicas eventually converge* to the
  same value. Says **nothing** about *when*, and permits all three lag anomalies and reading stale
  data in the meantime.
- **Cost:** cheapest, most available, lowest latency. The burden moves to the application to tolerate
  staleness and resolve conflicts.
- **Where:** DNS, caches, Dynamo/Cassandra at low quorum, like-counts, view-counts — anything where
  "approximately right, eventually exact" is fine.

| Model | Guarantee | Available under partition? | Cost | Real systems |
|---|---|---|---|---|
| Linearizable | Single copy, real-time recency | No (CP) | Highest | etcd, ZooKeeper, Spanner |
| Sequential | Single total order, no real-time | No | High | Theory / memory models |
| Causal | Causal order preserved, concurrent free | Yes | Moderate | Causal+ stores, COPS |
| Eventual | Converges if writes stop | Yes (AP) | Lowest | DNS, Cassandra, Dynamo carts |

> **Say this in the room:** "I rank consistency by how much coordination it forces. Linearizable
> needs consensus on every op and dies under partition. Causal is the strongest thing I can keep
> available under partition. So the question is never 'strong or eventual' — it's *which is the
> weakest model that's still correct for this feature.*"

---

## Part F — Quorum Consistency (Leaderless Tunable Consistency)

In a leaderless system with **N** replicas per key, you choose per-operation:
- **W** = replicas that must ack a write before it's "successful."
- **R** = replicas that must respond to a read before you return.

**The overlap rule: if W + R > N, the read set and the write set must share at least one replica** —
so a read is guaranteed to touch at least one replica that has the latest write. That replica carries
the newest version, and the client picks the newest among the R responses.

```
N = 3, W = 2, R = 2  →  W + R = 4 > 3  ✓ overlap guaranteed
Write goes to replicas {1,2}. Any read of 2 replicas — {1,2},{1,3},{2,3} —
includes at least one of {1,2}. So the read sees the latest write.
```

**Tuning examples (N=3):**
- `W=3, R=1`: fast reads, slow/fragile writes (write needs all replicas up). Read-heavy.
- `W=1, R=3`: fast writes, slow reads. Write-heavy.
- `W=2, R=2`: balanced — the common default.
- `W=1, R=1`: W+R=2 ≤ 3 → **no overlap guarantee** → pure eventual consistency, max availability.

**Sloppy quorums + hinted handoff.** A *strict* quorum requires W of the key's *designated home*
replicas. If too many home replicas are down, a strict quorum can't be met and the write fails. A
**sloppy quorum** instead accepts the write on *any* W reachable nodes — even ones outside the home
set — preserving **write availability**. Those temporary nodes hold a **hint** ("this really belongs
to node X") and, via **hinted handoff**, forward the data to the rightful home node once it recovers.

> **Sloppy quorum caveat:** because the write may have landed entirely on non-home nodes, a subsequent
> read of the home replicas can *miss* it until handoff completes. So sloppy quorums boost *write
> availability* at the cost of weakening the read guarantee. Name this tradeoff — it's a classic
> follow-up.

**Why quorum is NOT linearizable.** Even with W+R>N, quorums give you "you'll read a recent value,"
**not** linearizability:
- **Concurrent reads during a write** can see different values — some replicas have the new write,
  some don't, and there's no agreement on a single instant the write "took effect."
- **Sloppy quorums** can write entirely outside the read set → the read misses it.
- A **failed write** that succeeded on some-but-not-W replicas is *not* rolled back; later reads may
  return it, then revert.
- **Last-write-wins clock skew** can silently drop a write that a strict reading of W+R>N seems to
  protect.

So: quorums give **tunable, strong-ish eventual consistency**, not linearizability. If you need
linearizability you need consensus (Raft/Paxos) or a leader, not a quorum read.

---

## Part G — Conflict Resolution

The moment you allow concurrent writes to the same key (multi-leader or leaderless), two writes can
conflict. You need a deterministic, convergent way to resolve them — *every replica must reach the
same answer* or you never converge.

### Last-write-wins (LWW)

Attach a timestamp to each write; the highest timestamp wins. **Simple and convergent, but it
silently discards data:** if two clients write concurrently, one write is thrown away even though
both were "successful." Worse, it relies on clock synchronization — **clock skew** between nodes
means the write with the "later" clock wins, which may not be the one that physically happened later.
Cassandra uses LWW by default; it's acceptable *only* when losing a concurrent write is harmless.

> **Trap:** "We'll just use timestamps." For a shopping cart or a counter, LWW *loses items*. Know
> when it's catastrophic.

### Version vectors / vector clocks (the precise tool)

Instead of a single timestamp, track a **counter per node**. This lets you detect whether two writes
are causally ordered (one happened-before the other) or genuinely **concurrent** (a real conflict),
rather than guessing with a wall clock.

A vector clock is a map `{node → counter}`. Rules:
- On a local event/write, a node increments **its own** counter.
- A message/replication carries the sender's full vector; the receiver takes the **element-wise max**,
  then increments its own.
- Compare two vectors V and W: `V ≤ W` if every component `V[i] ≤ W[i]`. If `V ≤ W` then V
  happened-before W. If neither `V ≤ W` nor `W ≤ V`, they're **concurrent → conflict**.

**Worked example.** Three replicas A, B, C; one key starts at `{A:0, B:0, C:0}`.

1. Client writes "x" at A. A → `{A:1, B:0, C:0}`.
2. That replicates to B. B merges (max) and applies → version `{A:1, B:0, C:0}`, value "x".
3. **Network partition.** A and B can't see C.
4. Client writes "y" at A (read x first, so it's a successor). A → `{A:2, B:0, C:0}`, value "y".
5. Concurrently, a *different* client writes "z" at C, having only seen the original.
   C → `{A:0, B:0, C:1}`, value "z".
6. Partition heals; A and C exchange versions. Compare `{A:2,B:0,C:0}` (y) vs `{A:0,B:0,C:1}` (z):
   - Is `{A:2,B:0,C:0} ≤ {A:0,B:0,C:1}`? No (A: 2 > 0).
   - Is `{A:0,B:0,C:1} ≤ {A:2,B:0,C:0}`? No (C: 1 > 0).
   - **Neither dominates → y and z are concurrent → genuine conflict.**

The system can't pick automatically without losing data, so it keeps **both as siblings** and returns
them to the client (Dynamo: returns both versions on read; the app/user merges — e.g. union the
shopping carts). After the merge-write, the resolving vector dominates both: `{A:2, B:0, C:1}`.

> **The point of the example:** a vector clock turned "two timestamps, pick one (lose data)" into "I
> can *prove* these were concurrent, so I'll surface both and merge intelligently." That distinction
> — detecting concurrency vs assuming order — is the whole reason version vectors exist.

(Note: a *vector clock* tracks events between processes; a *version vector* tracks replica versions of
a data item. Same mechanics; the latter is the DB term. Don't get nitpicked — mention you know the
distinction.)

### CRDTs (Conflict-free Replicated Data Types)

Data structures whose merge function is **commutative, associative, and idempotent**, so replicas
**always converge regardless of the order** writes arrive — *no coordination, no conflicts, no manual
resolution*. The structure itself guarantees convergence. This is "strong eventual consistency."

Common types:
- **G-Counter** (grow-only counter): a vector of per-node counts; value = sum; merge = element-wise
  max. Only increments. Good for like-counts/view-counts.
- **PN-Counter:** two G-Counters (one for increments P, one for decrements N); value = P − N. Now you
  can decrement too.
- **OR-Set** (observed-remove set): add/remove a set where each add carries a unique tag; remove only
  removes tags it has *observed*. Resolves the add/remove race so a concurrent add+remove behaves
  sensibly (add wins if it wasn't observed by the remove).
- (Also: LWW-Register, sequence CRDTs like RGA/Logoot for ordered text.)

**Where collaborative editing uses them:** real-time co-editing (think Figma, newer collaborative
editors, Automerge/Yjs libraries) model the document as a CRDT (often a sequence CRDT for text plus
maps/sets for structure) so every client can edit offline and merge automatically on reconnect with
guaranteed convergence and no central server arbitration.

### CRDTs vs Operational Transforms (OT)

Both solve "concurrent edits to shared state," classically in collaborative editors.
- **OT** (the Google Docs lineage): transforms each operation against concurrent ones so they apply
  consistently (insert-at-index 5 gets shifted if a concurrent insert happened earlier). Powerful and
  compact, but the transform functions are **notoriously hard to get correct**, and OT generally
  **assumes a central server** to order operations.
- **CRDTs:** push the correctness into the data type's merge rules — harder to design the type, but
  once you have it, convergence is *guaranteed* and you don't need a central coordinator (great for
  P2P/offline). Cost: metadata overhead (tombstones, tags) that can grow.

| | OT | CRDT |
|---|---|---|
| Correctness burden | Transform functions (error-prone) | Merge laws (proven once) |
| Coordination | Usually needs central server | None — fully decentralized OK |
| Overhead | Low metadata | Tombstones/tags can grow |
| Used by | Google Docs (classic) | Figma, Automerge, Yjs, Riak |

---

## Part H — CAP and PACELC, Rigorously

### CAP

When a network **partition (P)** occurs between replicas, you can preserve **Consistency** (refuse
operations that would diverge → become unavailable) **OR** **Availability** (keep serving, allow
divergence → not consistent), **not both**. CAP is *only about the partition case*. The common
mistake is treating it as "pick 2 of 3" all the time — wrong. Partitions *will* happen, so you're
really choosing **CP or AP** under partition.

- **CP systems:** prefer consistency, sacrifice availability under partition. The minority side stops
  serving. Examples: ZooKeeper, etcd, HBase, MongoDB (default), Spanner.
- **AP systems:** keep serving on both sides, reconcile later. Examples: Cassandra, Dynamo, Riak,
  CouchDB.

### PACELC (the refinement you should always volunteer)

CAP ignores the *normal* (no-partition) case, where there's still a tradeoff. PACELC:

> **If Partition (P): choose Availability (A) or Consistency (C); Else (E), choose Latency (L) or
> Consistency (C).**

This is sharper because *even with no partition*, stronger consistency costs latency (you wait for
acks/coordination). Classifications:

| System | Under partition | Normally (else) | Class |
|---|---|---|---|
| Cassandra / Dynamo | A (stay available) | L (low latency, tunable) | **PA/EL** |
| MongoDB | C (CP) | L (reads from primary, async repl) | **PC/EL** (roughly) |
| Spanner | C | C (waits on TrueTime) | **PC/EC** |
| etcd / ZooKeeper | C | C | **PC/EC** |

> **Say this in the room:** "CAP only describes the partition case; PACELC adds the question that
> actually bites in steady state — latency vs consistency *when the network is fine*. Spanner is
> PC/EC: it pays latency for consistency even with no partition. Cassandra is PA/EL: it gives up
> consistency for both availability under partition and latency normally."

---

## Part I — Linearizability vs Serializability (Different Things)

These get conflated constantly. They are **orthogonal**.

- **Linearizability** is a **recency / single-object (or per-operation)** guarantee about *reads and
  writes*: there's one copy, operations happen in real-time order, a read sees the latest write. It's
  a property of **distributed/concurrent systems** — about *when* an operation is visible.
- **Serializability** is a **transaction isolation** guarantee: the result of executing multiple
  **multi-object transactions** concurrently equals *some serial (one-at-a-time) order* of those
  transactions. It says nothing about real time — the equivalent serial order need not match wall
  clock.

So:
- Serializability is about **transactions** (multiple operations grouped, possibly many objects).
- Linearizability is about **single operations** and their **real-time** visibility.

**Strict serializability** = serializable **+** linearizable = the strongest: transactions appear in
a single serial order *that respects real time*. That's what Spanner provides.

| | Linearizability | Serializability |
|---|---|---|
| Scope | Single object / operation | Multi-object transactions |
| Guarantees | Real-time recency, one copy | Equivalent to *some* serial order |
| Concerns | Visibility timing across replicas | Isolation of concurrent transactions |
| Field | Distributed systems | Database transaction theory |

> **Say this in the room:** "Linearizability is a recency guarantee on single objects; serializability
> is an isolation guarantee on transactions. They're independent — you can have one without the other.
> Spanner gives you both, which is *strict* serializability."

---

## Part J — Decision Guide: What Consistency Does THIS Feature Need?

Don't pick a consistency model by reflex. Derive it from the cost of being wrong.

1. **Is a stale or lost write a correctness bug, or just cosmetic?**
   - Correctness bug (balance, inventory, "claim username", auth) → **linearizable / strong**, accept
     the latency and the CP availability hit.
   - Cosmetic (like count, view count, "X people are typing") → **eventual** is fine; reach for a
     CRDT counter.
2. **Does the *same user* need to see their own action immediately?**
   - Yes → at minimum **read-your-own-writes** (route their reads to leader/fresh replica).
3. **Do humans observe ordering between related items (messages, comments, replies)?**
   - Yes → **causal consistency** / consistent-prefix (same-partition for related writes).
4. **Do you need multi-region *writes* with low latency?**
   - Yes and conflicts are rare/mergeable → **multi-leader or leaderless + CRDT/version vectors**.
   - Yes but you need one truth → pay for consensus (Spanner-style) or keep a single write region.
5. **Is write-availability under failure non-negotiable (must always accept writes)?**
   - Yes → **leaderless + sloppy quorum + hinted handoff**, resolve conflicts later.
6. **Multi-object invariant (transfer money A→B, both legs or neither)?**
   - → **serializable transactions** (and if cross-region recency matters, *strict* serializability).

> **Mental checklist when handed an unseen feature:** What's the cost of a stale read here? Of a lost
> write? Does *this* user need to see *their own* write now? Does anyone observe *ordering*? Pick the
> **weakest** model that's still correct — then name the anomaly you're choosing to allow.

A worked mapping you can recite:

| Feature | Right consistency | Why |
|---|---|---|
| Account balance / payments | Linearizable + serializable | Lost/stale write = money bug |
| Username/email uniqueness | Linearizable (consensus) | Two winners = corruption |
| Social feed / timeline | Eventual + monotonic reads | Stale is fine; don't go backward |
| User's own profile edit | Read-your-own-writes | They must see their save |
| Chat / comment threads | Causal / consistent prefix | Reply must follow message |
| Like / view counters | Eventual (PN/G-counter CRDT) | Approximate is fine |
| Shopping cart | Leaderless + version vectors | Never lose an added item; merge |
| Collaborative doc editing | CRDT (or OT) | Offline edits must converge |
| Config / leader election / locks | Linearizable (etcd/ZK) | Must agree on one value |

---

### Self-check before the mock (answer these from memory)
- [ ] Describe write/read flow, failure handling, and conflict potential for single-leader, multi-leader, and leaderless.
- [ ] Why does single-leader never have write conflicts, and what does that cost you (failover)?
- [ ] Sync vs async vs semi-sync — state the durability/latency tradeoff in one sentence each.
- [ ] Name the three replication-lag anomalies and the read guarantee + cheap mechanism that fixes each.
- [ ] Explain split-brain and how a fencing token prevents it.
- [ ] State the consistency ladder strongest→weakest, with what each guarantees and a real system for each.
- [ ] Why does W+R>N guarantee overlap, and why is quorum still *not* linearizable?
- [ ] What do sloppy quorums + hinted handoff buy you, and what do they cost?
- [ ] Walk a vector-clock example that detects two concurrent writes.
- [ ] LWW vs version vectors vs CRDTs — when is each right? Name a CRDT type and where it's used.
- [ ] OT vs CRDT for collaborative editing — the core difference.
- [ ] State PACELC and classify Cassandra, MongoDB, and Spanner.
- [ ] Linearizability vs serializability — one sentence each, and what "strict serializability" is.
