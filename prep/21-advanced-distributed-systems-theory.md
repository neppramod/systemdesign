# Topic 21: Advanced Distributed Systems Theory — Clocks, CRDTs, and Coordination Avoidance

> **Why this topic separates senior from staff:** A senior engineer reaches for the right block
> (Raft here, a queue there). A staff engineer knows *when the block isn't needed at all* — when a
> monotonic design lets you skip consensus, when causal consistency is "good enough" and an order of
> magnitude cheaper, when a CRDT lets two replicas diverge and still converge without a coordinator.
> The unifying idea of this doc: **coordination is the expensive thing.** Everything here is about
> ordering events, agreeing on state, and — the prize — *avoiding* the agreement when you can prove
> you don't need it. Say in the room: "Before I add consensus, let me check whether I actually need it."

---

## Part A — Time, and Why You Can't Trust It

### A.1 Physical clocks and NTP skew
Every machine has a quartz oscillator. It drifts — typically tens of ppm, so milliseconds per minute,
and worse under temperature swings. NTP disciplines the clock against a reference, but:

- NTP corrects over a *network path* with asymmetric latency; the residual error is commonly **1–50ms**,
  sometimes hundreds of ms on a congested or misconfigured host.
- NTP can step the clock **backwards**. A timestamp taken at `t` can be *larger* than one taken later.
- Two machines can disagree by their combined error. You have **no guarantee** that node A's "now" is
  before node B's "now," even if A's event physically happened first.

> **The one sentence to say:** "Wall-clock time is fine for human-facing display and coarse TTLs, but
> I will never use it to *order* events across machines or to decide who wrote last — clock skew makes
> that incorrect, not just imprecise."

The classic failure: **last-write-wins (LWW) by wall clock.** Two nodes, skewed clocks; the write that
"happened later" gets a smaller timestamp and is silently dropped. You lose data and never know.

### A.2 Lamport logical clocks — capture *happens-before*, not real time
Lamport (1978) gave up on real time and modeled **causality**. Define the *happens-before* relation
`→`:

1. If `a` and `b` are in the same process and `a` comes first, then `a → b`.
2. If `a` is a send and `b` is the matching receive, then `a → b`.
3. Transitivity: `a → b` and `b → c` ⟹ `a → c`.

If neither `a → b` nor `b → a`, the events are **concurrent** (`a ∥ b`).

**The algorithm** (each process holds a counter `C`):
- On any local event or send: `C = C + 1`.
- On send: attach `C` to the message.
- On receive of a message carrying timestamp `t`: `C = max(C, t) + 1`.

**Guarantee:** `a → b` ⟹ `C(a) < C(b)`. The *clock condition*.

> **Critical caveat — say this:** "Lamport clocks give a one-way implication only. `a → b` implies
> `C(a) < C(b)`, but `C(a) < C(b)` does **not** imply `a → b`. So Lamport timestamps can totally-order
> events (break ties by process id) but they **cannot detect concurrency**." That last gap is exactly
> what vector clocks fix.

### A.3 Vector clocks — detect concurrency
Each of `N` processes keeps a vector `V[1..N]`. `V[i]` is process `i`'s count of events it knows about.

**The algorithm** (process `i`):
- Local event / send: `V[i] += 1`.
- Send: attach the whole vector `V`.
- Receive vector `W`: `V[k] = max(V[k], W[k])` for all `k`, then `V[i] += 1`.

**Comparison:**
- `V ≤ W` iff `V[k] ≤ W[k]` for all `k`.
- `V < W` (i.e. `V → W`, causally before) iff `V ≤ W` **and** `V ≠ W`.
- **Concurrent** (`V ∥ W`) iff neither `V < W` nor `W < V`.

This is the property Lamport clocks lack: from the vectors alone you can tell whether two events are
causally ordered *or genuinely concurrent* — which is what conflict detection needs.

#### A worked example (3 processes: A, B, C)
Start all at `[0,0,0]`. Read the vector as `[A,B,C]`.

| Step | Event | Process | Vector after | Notes |
|---|---|---|---|---|
| 1 | local `a1` | A | `[1,0,0]` | A bumps its own slot |
| 2 | local `b1` | B | `[0,1,0]` | B bumps its own slot |
| 3 | A sends msg `m1` (carries `[1,0,0]`)→C | A | `[2,0,0]` | send is an event |
| 4 | C receives `m1` (`c1`) | C | `[1,0,1]` | merge `max` then bump C |
| 5 | local `c2` | C | `[1,0,2]` | |
| 6 | C sends `m2` (carries `[1,0,2]`)→B | C | `[1,0,3]` | |
| 7 | B receives `m2` (`b2`) | B | `[1,2,3]` | merge `max([0,1,0],[1,0,2])=[1,1,2]`, bump B→`[1,2,2]`… see below |

Let me fix step 7 carefully — this is the part interviewers probe. Before receive, B = `[0,1,0]`.
Incoming = `[1,0,2]`. Merge by `max`: `[max(0,1), max(1,0), max(0,2)] = [1,1,2]`. Then B bumps its
own slot: `[1,**2**,2]`. So **b2 = `[1,2,2]`**.

Now compare events:
- `a1 = [1,0,0]` vs `c2 = [1,0,2]`: `a1 ≤ c2` and `≠` ⟹ **`a1 → c2`** (causal: A's message reached C).
- `b1 = [0,1,0]` vs `c2 = [1,0,2]`: is `b1 ≤ c2`? `b1[B]=1 > c2[B]=0`. No. Is `c2 ≤ b1`? `c2[A]=1 > b1[A]=0`.
  No. ⟹ **`b1 ∥ c2`** (concurrent — B's first event and C's second event have no causal path between them).
- `b1 = [0,1,0]` vs `b2 = [1,2,2]`: `b1 ≤ b2`, `≠` ⟹ **`b1 → b2`** (same process, earlier).

> **In the room:** "The point of the vector is that I can hand you any two version vectors and *prove*
> whether one causally precedes the other or whether they're concurrent — and concurrent is precisely
> the case I must resolve with a merge function or surface as a conflict."

**Cost:** vectors grow with the number of *writers/actors*, not entries. Dynamo-style systems prune
stale entries; sequence CRDTs and version-vector systems cap actors. This `O(N)` metadata is the price
of detecting concurrency.

### A.4 Version vectors
Same machinery, different framing. A **version vector** tracks the version of a *data item* across
replicas (one slot per replica/actor). It's how Dynamo/Riak detect that two replicas hold concurrent
updates to the same key → return *siblings* to the client (or to a CRDT merge). "Vector clock" usually
means ordering events; "version vector" means versioning replicated state. Same comparison rules.

### A.5 Hybrid Logical Clocks (HLC) — the practical winner
Logical clocks give correct causality but their numbers mean nothing to humans and don't relate to
real time. Physical clocks relate to real time but lie about order. **HLC** (Kulkarni et al., 2014)
combines them:

- An HLC timestamp is a pair `(l, c)`: `l` is a physical-time component (tracks wall clock but never
  goes backwards relative to causality), `c` is a logical counter for ties.
- On a local/send event: `l = max(l_prev, pt_now)`; if `l == l_prev` then `c += 1` else `c = 0`.
- On receive `(l_m, c_m)`: `l = max(l_prev, l_m, pt_now)`; bump `c` appropriately to preserve order.

**What HLC buys you:**
- Timestamps are **within ε of physical time** (so they're meaningful for debugging, TTLs, range
  scans by approximate time) **and** respect happens-before (so they're safe for causal ordering).
- Constant size (a pair), unlike vector clocks. No `O(N)` growth.
- Used in **CockroachDB, YugabyteDB, MongoDB** (for causal consistency / cluster time).

> **The tradeoff line:** "HLC gives me causal-safe, real-time-ish, constant-size timestamps — but it
> still can't *detect concurrency* (that's vector clocks) and it can't give *external consistency*
> (that's TrueTime, which needs bounded error from hardware)."

---

## Part B — Google Spanner & TrueTime (the landmark design)

Spanner's claim is audacious: a **globally distributed, sharded, transactional SQL database** that
provides **external consistency** (a.k.a. linearizability + a real-time order across the whole system):
if transaction `T1` commits before `T2` *starts* in real time, then `T1`'s timestamp `< T2`'s
timestamp — globally, across continents. Everyone agreed this was impossible without a global clock.
Spanner's trick is to *build the clock* — and, crucially, to **expose its uncertainty**.

### B.1 TrueTime: an interval, not a point
Most clock APIs return a single instant and lie about the error. TrueTime returns an **interval**:

```
TT.now() -> [earliest, latest]   // the true time is GUARANTEED to lie in this window
```

The width of the window is `2ε` (epsilon = clock uncertainty). TrueTime is backed by **GPS receivers
and atomic clocks** in every datacenter (two independent failure-uncorrelated sources). Servers poll
time masters; between polls, ε grows with the worst-case local drift and shrinks at each sync. In
production, **ε averages ~1–7ms** (historically reported up to ~10ms at the tail).

> **Say this:** "The genius isn't a perfect clock — there's no such thing. It's *admitting* the error
> as a bound `ε` and then designing the commit protocol to wait that error out."

### B.2 Commit-wait — how external consistency falls out
To assign a commit timestamp `s` to a transaction and guarantee no one sees it out of order:

1. At commit, pick `s = TT.now().latest` (the latest possible "now").
2. **Commit-wait:** do not release locks / make the commit visible until `TT.now().earliest > s`.
   That is, **wait until the chosen timestamp is definitely in the past** for *every* clock in the
   system — wait out the uncertainty window, ~`2ε`.

Because every transaction waits until its timestamp is unambiguously past, timestamps are guaranteed
to reflect real-time commit order everywhere. That's external consistency.

| Component | Role |
|---|---|
| TrueTime interval `[earliest, latest]` | exposes bounded uncertainty `ε` |
| Paxos groups (per shard) | replicate each tablet, elect leaders, durability |
| 2PC across Paxos groups | atomic multi-shard transactions |
| Commit-wait (~2ε) | turns bounded uncertainty into global real-time order |
| TrueTime masters (GPS + atomic) | keep `ε` small, hardware-backed |

### B.3 The tradeoff
- **You pay latency:** every read-write transaction commits-waits ~`2ε` (single-digit ms). Spanner
  works to keep ε small precisely because **commit latency is proportional to ε** — shrinking the
  clock error directly buys faster commits.
- **You pay hardware:** GPS + atomic clocks in every DC, a fleet of time masters. Most companies can't
  reproduce this — which is why Cockroach/Yugabyte use HLC and offer *serializable* but not the same
  hardware-guaranteed *external* consistency without configured max-offset assumptions.
- **Read-only transactions** can be lock-free at a chosen timestamp (snapshot reads), avoiding the wait.

> **Why it's a landmark:** it's the proof-by-construction that the CAP "you can't have C and global
> scale" intuition is about *protocols*, not laws — given a tight enough clock bound, you can have
> linearizable transactions across the planet. The cost is honesty about ε plus a few ms of waiting.

---

## Part C — Consistency Models, Rigorously Ordered

Consistency models are a lattice from strongest/most-expensive to weakest/cheapest. Know the order and
*what each one forbids*.

| Model | Guarantee | Forbids | Needs coordination? | Real systems |
|---|---|---|---|---|
| **Linearizable** (strong) | Every op appears to take effect atomically at a single point between invoke and response; consistent with real-time order | Stale reads; reordering vs real time | Yes — consensus / TrueTime | Spanner, etcd, ZooKeeper, single-node RDBMS |
| **Sequential** | All processes see the *same* total order; respects per-process program order, but **not** real time | Per-process reorder; disagreement on order | Yes (weaker than linearizable) | Rare in pure form; classic shared-memory model |
| **Causal+ (causal w/ convergence)** | Causally related ops seen in causal order by everyone; concurrent ops may differ; replicas converge | Reordering of cause before effect | **No global coordination** — track dependencies | COPS, MongoDB causal sessions, many CRDT systems |
| **Eventual** | If writes stop, replicas eventually converge; no order guarantee in between | Nothing about ordering | No | Dynamo/Cassandra (default), DNS, S3 (historically) |

The **cost gradient**: linearizable requires a round trip to a leader/quorum on the critical path and
cannot survive a partition while staying available (CAP). Causal can be served from a local replica
(low latency, partition-tolerant) as long as dependencies are satisfied. Eventual is the cheapest and
most available — and the hardest to reason about for correctness.

> **The staff move:** "I default *down* the gradient. I ask: what's the weakest model that still keeps
> my invariants correct? Linearizability is the default people reach for and it's usually overkill —
> most of the system needs causal+ at most, and a lot of it is fine with eventual."

**Two reads people conflate:**
- **Read-your-writes / monotonic-reads** are *session guarantees* — weaker, often layered on eventual
  storage via sticky routing or a client-tracked version, and they're cheap. Mention them when someone
  says "but the user must see their own post immediately."

---

## Part D — Causal Consistency and How to Build It

Causal+ is the **sweet spot**: it's the strongest model that *doesn't require coordination* and still
matches human intuition (you never see a reply before the comment it replies to; you never see a photo
with permissions removed *after* you removed them — the "no anomalies" examples from the COPS paper).

**How to implement it — dependency tracking:**
1. Tag every write with metadata that captures its causal dependencies. Options:
   - **Explicit dependency list** (the ops this write causally depends on) — COPS-style.
   - **Vector clock / version vector** — one slot per replica, compare to know if deps are met.
   - **HLC / Lamport timestamp** — cheaper, gives a causal-compatible order.
2. On replication, a replica **does not apply** an incoming write until all its dependencies have
   already been applied locally. It buffers otherwise. This enforces causal order without any replica
   talking to a coordinator at write time.
3. Concurrent writes (per the vectors) are resolved by a **merge function** (LWW, app-defined, or a
   CRDT) so replicas converge → the "+" in causal+.

**Cost:** metadata size and dependency-check bookkeeping. Mitigations: track dependencies at coarse
granularity, garbage-collect satisfied deps, cap the number of tracked actors.

> **In the room:** "For a social feed or comments system I'd use causal+: a local replica serves reads
> with single-digit-ms latency, survives partitions, and I never show effect-before-cause. I pay for it
> with per-write dependency metadata, not with a coordinator on the hot path."

---

## Part E — CRDTs In Depth

A **CRDT** (Conflict-free Replicated Data Type) is a data structure that **replicas can update
independently and concurrently, with a deterministic merge that always converges** — no coordination,
no conflicts surfaced to the app. They're how you get *Strong Eventual Consistency* (SEC): replicas
that have received the same set of updates are in the same state, regardless of order or duplication.

### E.1 The two families

**State-based (CvRDT) — convergent.**
- The merge is a function over the *whole state*. The state space is a **join-semilattice**: a partial
  order with a least-upper-bound (`⊔`) for any two elements.
- Requirements: updates are **inflationary** (state only moves *up* the lattice — monotonic), and merge
  is the **join** — which is commutative, associative, idempotent (CAI).
- Because merge is CAI and updates are monotonic, replicas converge no matter the order, duplication, or
  delay of state exchange. You can gossip whole states or deltas (δ-CRDTs send only the changed part).
- **Network requirement: almost none** — just eventual delivery of states. Tolerates duplicates and
  reordering for free. Cost: shipping/merging state can be large (δ-CRDTs fix this).

**Operation-based (CmRDT) — commutative.**
- Replicas broadcast *operations*; concurrent operations must **commute** (applying in either order
  yields the same state).
- **Network requirement: reliable, exactly-once, causal-order broadcast** of operations (no
  duplication, no loss). Cheaper messages (just the op), stricter delivery guarantees.

> **The trade in one line:** "State-based asks little of the network but ships state; op-based ships
> tiny ops but demands reliable causal delivery. δ-state CRDTs are the modern middle ground."

### E.2 The catalog (know these cold)

| CRDT | What it is | Merge / convergence trick | Gotcha |
|---|---|---|---|
| **G-Counter** | Grow-only counter | Vector of per-replica counts; value = sum; merge = element-wise `max` | Can't decrement |
| **PN-Counter** | Increment + decrement | Two G-Counters (P and N); value = sum(P) − sum(N) | More metadata |
| **G-Set** | Grow-only set | Merge = set **union** | Can't remove |
| **2P-Set** | Add + remove | Add-set ∪ a tombstone remove-set; once removed, **never re-addable** | Re-add impossible; tombstones grow |
| **LWW-Register** | Single value, last write wins | Keep value with the highest timestamp (HLC!) | Concurrent writes silently drop one |
| **MV-Register** | Multi-value register | Keep all concurrent values (version vector), expose siblings | App must resolve siblings |
| **OR-Set** (observed-remove) | Add + remove, re-addable | Each add tagged with a unique id; remove only kills the **ids it observed** → a later add with a new id survives | Tag bookkeeping; this is the "correct" set most people want |
| **RGA / sequence CRDTs** | Ordered list / text | Each element gets a unique, densely-orderable id; inserts reference a position id; deletes tombstone | Tombstone accumulation; ids must be totally orderable |

**RGA / text** is the one that powers collaborative editors: every inserted character/element has a
stable unique identifier with a total order, so concurrent inserts at "the same place" deterministically
interleave the same way on every replica. Yjs, Automerge, and Logoot/LSEQ-style structures are
variations on this theme.

### E.3 Where CRDTs are used
- **Riak** (data types: counters, sets, maps), **Redis Enterprise CRDTs (Active-Active geo)**,
  **Azure Cosmos DB** (multi-region writes), **Automerge** and **Yjs** (local-first / collaborative
  apps), **Figma**'s multiplayer model is CRDT-flavored, shopping carts (the canonical Dynamo example —
  an OR-Set-like cart that never loses an add).

> **When to reach for a CRDT:** "Multi-writer, multi-region or offline-capable, and I want
> *availability under partition with automatic convergence* rather than conflicts in the user's face.
> The price is metadata growth and that the merge semantics are baked in — if the business needs a
> *custom* conflict resolution that isn't expressible as a lattice join, a CRDT fights me."

---

## Part F — OT vs CRDTs for Collaborative Editing

Both solve "many users edit the same document concurrently and everyone converges." Different machinery.

**Operational Transformation (OT)** — what **Google Docs** uses.
- Edits are operations (`insert(pos, char)`, `delete(pos)`) sent through a **central server**.
- When concurrent ops collide, the server (and clients) **transform** an op against the ones it didn't
  see — e.g. "your insert was at position 5, but a concurrent insert at position 2 shifted everything,
  so I rewrite your op to position 6." A transformation function adjusts indices so intentions are
  preserved and everyone converges.
- **Strength:** compact ops, no per-character metadata, the document stays a plain sequence.
- **Weakness:** the transformation functions are **notoriously hard to get right** (the literature is
  littered with buggy OT papers), and classic OT effectively **assumes a central server** to order ops;
  true peer-to-peer OT is very hard.

**CRDTs for text (RGA/Yjs/Automerge)** — what newer/local-first apps use.
- Every element has a stable unique id; concurrent edits merge by id ordering, **no transformation and
  no central server required**. Naturally **peer-to-peer and offline-first**.
- **Weakness:** per-element identifiers and tombstones → **metadata/memory overhead**; historically
  larger documents, though Yjs/Automerge have largely engineered this away.

| | OT (Google Docs) | CRDT (Yjs/Automerge) |
|---|---|---|
| Coordinator | Effectively needs a central server to order ops | Not required; P2P / offline friendly |
| Per-element metadata | None (plain sequence) | Unique ids + tombstones |
| Correctness difficulty | Transformation functions are hard/bug-prone | Merge is provably convergent by construction |
| Offline / local-first | Awkward | Natural |
| Why chosen | Docs predates mature text CRDTs; server-centric was fine | Local-first, multiplayer, P2P era |

> **The honest answer:** "Docs uses OT because it was built server-centric before text CRDTs were
> mature, and OT's compact ops were efficient on a server that already orders everything. New
> collaborative/local-first apps pick CRDTs because they converge with no central authority and work
> offline — at the cost of metadata. Neither is strictly 'better'; it's a coordinator-vs-metadata trade."

---

## Part G — Coordination Avoidance

This is the staff-level punchline of the whole doc: **the cheapest coordination is none.** When can you
prove you don't need it?

### G.1 The CALM theorem
**CALM = Consistency As Logical Monotonicity** (Hellerstein, Ameloot et al.).

> A program has a **consistent, coordination-free** distributed implementation **if and only if it is
> monotonic.**

- **Monotonic** = the output only *grows* as input grows; you never have to *retract* a result once
  emitted. Adding more facts never invalidates a previous conclusion.
- Monotonic operations: set union, increments of a grow-only counter, "does any record satisfy P?"
  (once true, stays true), `max`, append-only logs.
- **Non-monotonic** operations need coordination: counting/aggregation that must be *complete* before
  it's correct, deletion, "is this the *only* record?", uniqueness constraints, negation ("there does
  *not* exist…"), anything where a later fact can falsify an earlier answer.

**The practical lesson:** *design your operations to be monotonic and you eliminate consensus from the
hot path.* G-Counters, G-Sets, OR-Sets, append-only event logs are all CALM-friendly. The moment you
need a global "exactly once," "no duplicates," "the final total," or "delete and it stays deleted in a
way others must respect," you've crossed into non-monotone territory and you'll need coordination there
— so **isolate it.**

### G.2 Invariant Confluence (I-confluence)
From the Bailis et al. "Coordination Avoidance in Database Systems" work. A finer-grained, invariant-
aware test:

> A set of transactions is **I-confluent** with respect to an invariant `I` if, whenever two
> I-valid database states are merged, the result still satisfies `I`. If so, you can run those
> transactions **coordination-free** (each replica locally, merge later) and never violate `I`.

- **I-confluent (no coordination):** read-only, blind writes under LWW, incrementing a counter with a
  `≥ 0` floor that's never approached, foreign-key *insert* (adding a child whose parent exists),
  appends.
- **NOT I-confluent (coordination required):** uniqueness constraints (two replicas each assign the
  same username), "balance must stay ≥ 0" near zero (two concurrent withdrawals each valid locally,
  invalid merged), checking inventory before the last unit sells.

> **The decision frame to say out loud:** "I look at each invariant and ask: if two replicas each make
> a locally-valid change and I merge them, can the invariant break? If no — I-confluent — I run it
> coordination-free and let it converge. If yes, *that specific operation* needs coordination, and I
> scope the coordination to it instead of taxing the whole system."

---

## Part H — Impossibility & Fault Tolerance (conceptual)

You need these as *named results*, not proofs, to sound credible.

### H.1 FLP impossibility
**FLP (Fischer, Lynch, Paterson, 1985):** In an **asynchronous** network (no bound on message delay),
**no deterministic consensus protocol can guarantee termination** if even **one** node may crash. You
cannot reliably distinguish a slow node from a dead one.

- **Why it doesn't kill Raft/Paxos:** they sidestep FLP with **partial synchrony + randomization /
  timeouts**. They never sacrifice *safety* (never decide two values), only *liveness* (may stall while
  partitioned, then resume). Say: "FLP means consensus can't guarantee progress under full asynchrony;
  real protocols keep safety always and get liveness under enough synchrony."

### H.2 Consensus number / wait-free hierarchy
**Herlihy's hierarchy:** ranks synchronization primitives by the **maximum number of processes** for
which they can solve **wait-free consensus**.

| Primitive | Consensus number |
|---|---|
| Plain atomic read/write registers | **1** (can't do consensus for 2+ processes) |
| Test-and-set, fetch-and-add, queues, stacks | **2** |
| Compare-and-swap (CAS), LL/SC | **∞** (universal) |

The lesson: **CAS is universal** — with CAS you can build a wait-free implementation of *any* object.
This is *why* every lock-free data structure and every modern CPU's concurrency story is built on
compare-and-swap. Mere reads/writes can't even agree between two threads.

### H.3 Byzantine fault tolerance (BFT)
Everything above assumes **crash-stop** faults (nodes die, but don't lie). **Byzantine** faults =
nodes that behave **arbitrarily/maliciously** — send conflicting messages to different peers, forge,
collude.

- **PBFT (Castro–Liskov):** tolerates `f` Byzantine nodes with `3f + 1` total (so it needs a **2/3
  supermajority** of honest nodes), via a three-phase agreement (pre-prepare → prepare → commit) where
  nodes only act on a quorum of matching signed messages. Quadratic message complexity → small clusters.
- **Where BFT matters:** **permissionless blockchains** (Bitcoin's PoW, Ethereum's PoS, and BFT-style
  chains like Tendermint/Cosmos) — mutually distrusting parties, no central authority, real money. Also
  aerospace/avionics and some financial consortia. **Where it doesn't:** your own datacenter, where you
  trust your machines and only need to survive *crashes* — there, Raft/Paxos (crash-fault-tolerant,
  `2f+1`) is the right, far cheaper tool.

> **Say this so you don't over-engineer:** "Inside one trust domain I use crash-fault consensus
> (Raft); I only reach for BFT when participants are mutually distrusting — that's blockchains, not
> microservices."

---

## Part I — Decision Guide: Do I Actually Need Coordination?

Walk this top-to-bottom in the room.

1. **Is the operation read-only or a blind/idempotent write?** → No coordination. Serve from a replica.
2. **Are all the operations monotonic (CALM) — add-only, union, max, append, grow-counter?** → No
   coordination. Use a CRDT or an append-only log; converge.
3. **Do I have invariants? Run the I-confluence test per invariant** — "can a merge of two locally-valid
   states violate it?"
   - **No** → coordination-free; merge later.
   - **Yes** → coordination needed *for that operation only*. Scope it.
4. **What's the weakest consistency model that keeps invariants correct?** Default to **causal+** with
   dependency tracking; drop to eventual where even causality doesn't matter (likes, view counts via
   PN-Counter); climb to **linearizable** only for the operations that truly need it.
5. **For the operations that need coordination, what kind?**
   - Single-key atomicity / leader election / locks / config → **Raft/Paxos** (etcd, ZooKeeper).
   - Global real-time order across shards/regions → **Spanner/TrueTime** style (if you can afford it)
     or HLC-based serializable (Cockroach/Yugabyte).
   - Mutually distrusting parties → **BFT / blockchain**.
6. **Isolate the coordinated core.** Push uniqueness, balances-near-zero, exactly-once effects, and
   ordering-critical state into a small consensus-backed component; keep the bulk of the system
   monotone, replicated, and coordination-free around it.

> **The frame that makes you sound staff:** "Most of this system is monotone and I'll run it
> coordination-free with CRDTs/causal+ for availability and low latency. A *small* core — uniqueness on
> usernames and the wallet balance — is non-monotone and I-non-confluent, so I'll back exactly that core
> with Raft. I'm spending coordination only where an invariant forces me to."

---

### Self-check before the mock (answer these from memory)
- [ ] Why can't you order cross-machine events with wall-clock time, even with NTP?
- [ ] State the happens-before rules and the Lamport clock condition. Why can't Lamport clocks detect concurrency?
- [ ] Given two vector clocks, how do you decide `before` vs `concurrent`? Work the 3-process example.
- [ ] What does an HLC give you that a vector clock doesn't, and vice versa?
- [ ] Explain TrueTime's `ε`, commit-wait, and how they yield external consistency. What's the cost?
- [ ] Order the consistency models and name a real system at each level. Where's the cost cliff?
- [ ] How do you implement causal+ consistency without a coordinator? (dependency tracking)
- [ ] State-based vs op-based CRDT: what does each demand of the network? What's a join-semilattice?
- [ ] Why does an OR-Set allow re-adds but a 2P-Set doesn't?
- [ ] OT vs CRDT for text: why does Google Docs use OT and Yjs/Automerge use CRDTs?
- [ ] State the CALM theorem in one sentence. Give a monotonic and a non-monotonic operation.
- [ ] Run the I-confluence test on "usernames must be unique" and on "append to a log."
- [ ] What does FLP say, and why doesn't it break Raft? What's the consensus number of CAS, and why does it matter?
- [ ] When do you need BFT instead of Raft?
