# Topic 8: Consensus, Coordination & Distributed Locks

> **Why this topic matters:** Consensus is the smallest amount of agreement a distributed system
> can buy, and it is the most *expensive* thing you can put on a request path. Staff candidates are
> separated here not by reciting Raft, but by knowing **when you need it and when you don't** — and
> by getting distributed locks *right* (fencing tokens), which 90% of candidates get wrong. The
> headline you must internalize: **push consensus to the control plane, keep it off the data path.**

---

## Part A — What consensus is, and the problems it solves

**Consensus = getting a set of nodes to agree on a single value, even when some of them fail or the
network drops/delays/reorders messages.** That's it. The "value" can be:

- **Who is the leader** (leader election).
- **The next entry in a replicated log** (ordering — and ordering is the big one; if everyone agrees on the *order* of operations, they can deterministically replay to the same state).
- **The current cluster membership / config** (who's in, who's out).
- **Whether a lock is held**, and by whom (distributed locks / leases).

Every one of those reduces to the same primitive: *agree on one value despite failures.* That's why
Raft/Paxos/ZAB all look similar — they're solving the same problem.

> **Say this in the room:** "Almost everything people call 'coordination' — leader election, config,
> membership, locks, ordering — is the same problem underneath: agree on one value despite failures.
> I'll use one consensus system for all of it rather than inventing per-feature schemes."

The properties a real consensus algorithm must guarantee:

- **Agreement / Safety** — no two nodes decide different values. *Never* violated, even under
  partition. This is the non-negotiable.
- **Validity** — the decided value was actually proposed by someone (no making up values).
- **Termination / Liveness** — eventually a decision is reached. This is the one you *give up*
  during a partition (see FLP below). You stay safe but may stall.

The asymmetry is the whole game: **consensus systems sacrifice liveness, never safety.** When in
doubt, they stop. That's a deliberate CP choice (in CAP terms).

---

## Part B — The limits: FLP, quorums, and why 2f+1

### FLP impossibility (the thing to name, not fear)

The **FLP result** (Fischer, Lynch, Paterson, 1985): in a *fully asynchronous* network (no bound on
message delay), with even **one** node that can crash, there is **no deterministic algorithm** that
guarantees consensus will *always terminate*. The problem: you can't distinguish a *dead* node from a
*slow* node. If you wait forever for the slow one, you never decide; if you give up too early, you
risk splitting the decision.

> **Say this in the room:** "FLP says you can't have a deterministic, always-terminating consensus
> algorithm in a purely asynchronous model with one faulty node. Real systems don't violate it —
> they *sidestep* it. They assume **partial synchrony** and use **timeouts** to suspect failures, plus
> **randomization** (Raft's randomized election timeouts) to break symmetry. The trade is: under bad
> network conditions you lose *liveness* — the system stalls — but you never lose *safety*."

So FLP isn't an academic curiosity; it directly explains why Raft has timeouts and randomized
backoff. Those exist *because* you can't solve FLP head-on.

### Why a majority (quorum), and why 2f+1

Consensus needs a **majority quorum** because of the **intersection property**: any two majorities of
the same set must share at least one node. That shared node carries forward the latest decision, so a
new leader (elected by a majority) is guaranteed to *see* anything a previous majority committed.
Without overlap, two disjoint groups could each "decide" — that's split-brain.

To tolerate **f failures you need 2f + 1 nodes**:

| Nodes (N) | Quorum (majority) | Failures tolerated (f) |
|-----------|-------------------|------------------------|
| 3         | 2                 | 1                      |
| 5         | 3                 | 2                      |
| 7         | 4                 | 3                      |

- **Odd numbers only.** 4 nodes tolerate the same f=1 as 3 nodes but need a bigger quorum (3 vs 2) —
  strictly worse. Go 3, 5, 7.
- **3 is the standard** for most clusters; **5** when you want to survive losing 2 (rolling deploy +
  a crash). Beyond 5–7, write latency suffers because every commit waits on more nodes — bigger
  clusters are *more* available but *slower*, not faster.

> **Say this in the room:** "Majority quorum guarantees any two quorums intersect, so a newly elected
> leader always sees the previously committed entries. That's why I need 2f+1 nodes for f failures,
> and why I run an odd count — 3 normally, 5 if I need to survive two simultaneous losses."

**Note:** this is *crash* fault tolerance (CFT) — nodes fail by stopping. **Byzantine** fault
tolerance (BFT — nodes lie/are malicious) needs **3f+1** and is what blockchains use. In normal infra
design you assume CFT; mention BFT only if the prompt involves untrusted participants.

---

## Part C — Raft, in depth (know this cold; draw it on the whiteboard)

Raft was designed (Ongaro & Ousterhout, 2014) explicitly for **understandability** — it does the same
job as Paxos but decomposes it into three clean subproblems: **leader election**, **log replication**,
and **safety**. Learn it as those three.

### The whiteboard setup

Draw 3 boxes (S1, S2, S3). Each node is in one of three states: **Follower**, **Candidate**,
**Leader**. Each holds:

- **currentTerm** — a monotonically increasing integer; a logical clock for "eras of leadership."
- **votedFor** — who it voted for in the current term.
- **log[]** — a list of entries, each `{term, command}`.
- **commitIndex** — highest log entry known committed.

**One leader per term.** All client writes go through the leader. Followers are passive — they just
accept what the leader sends and respond to votes.

### 1. Leader election (terms + votes + randomized timeouts)

- Everyone starts as a **Follower**. A follower expects periodic **heartbeats** (empty
  AppendEntries) from the leader.
- If a follower hears nothing for its **election timeout** (randomized, e.g. 150–300ms), it assumes
  the leader is dead, **increments its term**, becomes a **Candidate**, votes for itself, and sends
  **RequestVote** RPCs to everyone.
- A node grants its vote if: (a) the candidate's term ≥ its own, (b) it hasn't already voted this
  term, and (c) the candidate's log is **at least as up-to-date** as its own (this last rule is the
  safety linchpin — more below).
- A candidate that collects votes from a **majority** becomes **Leader** and immediately sends
  heartbeats to assert authority and reset everyone's timers.

**Why randomized timeouts?** To break symmetry. If all followers timed out simultaneously, they'd all
become candidates, split the vote, no one wins, and they retry forever (a *split vote*). Randomizing
the timeout means one node almost always times out first, gets its vote out, and wins before others
even wake up. This is Raft's practical answer to FLP's liveness problem — randomization breaks the
livelock.

> **Whiteboard line:** "Term is a logical clock. Any message carrying a higher term immediately
> demotes whoever sees it back to follower and updates their term. Stale leaders can't do damage
> because their AppendEntries carry an old term and get rejected."

### 2. Log replication

- Client sends a command to the leader. Leader **appends** it to its own log (uncommitted) at the
  current term.
- Leader sends **AppendEntries** RPCs to followers with the new entry plus `prevLogIndex/prevLogTerm`
  (the entry immediately before it).
- A follower accepts only if it has a matching entry at `prevLogIndex/prevLogTerm` — this **log
  matching property** guarantees that if two logs agree at an index, they agree on *everything before
  it*. If they don't match, the follower rejects, and the leader walks its `nextIndex` for that
  follower backward until they find the agreement point, then overwrites the follower's divergent tail.
- Once a **majority** has acknowledged the entry, the leader marks it **committed**, applies it to its
  state machine, returns success to the client, and tells followers the new `commitIndex` on the next
  AppendEntries (so they apply too).

> **Say this in the room:** "Commit = replicated to a majority's *durable* log. The data is safe
> before any follower has even applied it to its state machine. Replication and application are
> separate steps."

### 3. Commit rules + the subtle safety bit

Two safety rules carry the whole proof:

1. **Election restriction (up-to-date check).** A candidate can only win if its log is at least as
   up-to-date as a majority. Combined with quorum intersection, this guarantees **a new leader already
   holds every committed entry** — committed entries are never lost.

2. **Leaders only directly commit entries from their *own* term.** This is the famous subtlety: a
   leader must **not** consider an entry from a *previous* term committed just because it's now
   replicated on a majority — a later leader could still overwrite it. The leader commits a past-term
   entry only *indirectly*, by committing a **current-term** entry on top of it (the "no-op on
   election" trick: a fresh leader appends a current-term entry to drag prior entries safely past the
   commit line). If you can articulate *this* rule, you're at staff depth.

### 4. Partitions and leader crashes (the failure walkthrough)

Walk this on the whiteboard — interviewers love it:

- **Leader crashes.** Followers stop getting heartbeats → one times out → new election → new term →
  new leader. Uncommitted entries from the dead leader are either propagated (if the new leader had
  them) or discarded. Committed entries always survive (election restriction).
- **Network partition (the split-brain test).** Split a 5-node cluster into {S1,S2} and {S3,S4,S5}.
  - The **minority side** (2 nodes): if the old leader is here, it *cannot* commit anything new — it
    can't reach a majority. It keeps trying, fails, and **serves stale data at best**. It may even
    step down. Crucially, **it cannot corrupt state** — no majority, no commit.
  - The **majority side** (3 nodes) elects a new leader in a higher term and continues normally.
  - **Heal the partition.** The old leader sees a higher term, **steps down to follower**, and its
    uncommitted divergent entries get overwritten by the new leader's log. No split-brain — at most
    one leader can ever *commit*, because committing requires a majority and there's only one majority.

> **Whiteboard punchline:** "Raft prevents split-brain structurally: two leaders might *exist*
> briefly in different terms, but only the one with a majority can *commit*. The minority leader is
> harmless — it's a leader who can't write."

**One gotcha to mention: stale reads.** A naive Raft leader serving reads from local state can return
stale data if it's been partitioned off and doesn't know it yet. Fixes: route reads through the log
(slow), or use **leader leases / ReadIndex** (confirm you're still leader by a heartbeat round before
serving the read). etcd does this.

---

## Part D — Paxos, Multi-Paxos, and why Raft exists

**Paxos** (Lamport) solves single-value consensus with two phases — **Prepare/Promise** then
**Accept/Accepted** — run by *proposers* against a majority of *acceptors*. A proposer picks a
ballot number, gets promises from a majority (learning any value already accepted), then asks them to
accept its value. Quorum intersection again guarantees safety. It's *correct* and was the academic
gold standard for two decades.

**Multi-Paxos** is the practical version: elect a stable **distinguished proposer** (a leader) so you
can skip Phase 1 for every entry and just stream Phase 2 — which makes it operate basically like
Raft's log replication. The catch: the original papers left the leader-election and log-management
details largely "as an exercise," so every implementation reinvented them differently and subtly
wrongly.

> **Say this in the room:** "Raft and Multi-Paxos are equivalent in power and roughly in performance.
> Raft exists because Paxos is famously hard to *implement correctly* — the paper leaves leader
> election and log compaction underspecified. Raft prescribes those, so I'd reach for Raft (or an
> off-the-shelf implementation like etcd) unless I had a specific reason not to."

Two more to name in one line each:
- **Zab** (ZooKeeper Atomic Broadcast) — ZooKeeper's protocol; very Raft-like (leader + ordered
  broadcast), predates Raft, optimized for high-read primary-backup.
- **EPaxos** (Egalitarian Paxos) — **leaderless**; any replica can commit commands, and only
  *conflicting* commands need ordering. Lower latency and no leader bottleneck/failover gap, at the
  cost of significant complexity. Mention it as the "no single leader" frontier.

---

## Part E — Coordination services: ZooKeeper & etcd

You almost never implement Raft yourself. You use a **coordination service** as the cluster's source
of truth, and build leader election / locks / config / discovery on top of its primitives.

### ZooKeeper (uses Zab)

- Data model: a tree of **znodes** (like a filesystem). Small data (KB), strongly consistent.
- **Ephemeral znodes** — exist only while the creating session's heartbeat is alive. Die when the
  client disconnects. *This is the magic primitive* — it's how you build locks and liveness detection
  (the lock auto-releases if the holder dies).
- **Sequential znodes** — ZK appends a monotonically increasing counter to the name. Combine with
  ephemeral → ordered queue of waiters → **fair locks and leader election without a thundering herd**.
- **Watches** — one-shot notifications when a znode changes. Clients watch instead of polling.
- Used by: Kafka (historically), HBase, Hadoop, Solr — for membership, config, leader election.

### etcd (uses Raft)

- Key-value store, **Raft-backed**, gRPC API. The coordination backbone of **Kubernetes** (stores all
  cluster state).
- **Leases** — keys with a TTL; client must renew or they expire (same role as ephemeral znodes).
- **Watches** on key prefixes; **MVCC** with revision numbers (great for "give me the version").
- **Compare-and-swap (txn)** — atomic conditional writes, the basis for correct locks.

> **Say this in the room:** "I wouldn't hand-roll consensus. I'd stand up etcd or ZooKeeper as the
> control-plane source of truth and build leader election, config, service discovery, and locks on
> its primitives — ephemeral nodes / leases for liveness, sequential nodes / CAS for ordering, watches
> instead of polling. New systems → etcd (Raft, gRPC, k8s-native). Existing JVM/Kafka stack →
> ZooKeeper."

**What people actually use them for:** leader election, service discovery / membership, dynamic
config distribution, feature flags, distributed locks, and leases. Note what they're **not**: not a
database, not for high write throughput, not for large values. They're a *small, strongly-consistent
control plane*, not a data plane.

---

## Part F — Distributed locks done RIGHT (the part most people botch)

### Why distributed locks are dangerous

A distributed lock looks like a mutex but isn't. The lethal failure: **the lock holder pauses or is
slow, the lock expires, someone else grabs it, and now *two* clients think they hold it.** Causes:

- **GC pause / VM stall** — a JVM stop-the-world or a hypervisor pause can freeze a process for
  seconds. The lock's TTL expires; the process wakes up still believing it holds the lock and writes.
- **Network delay** — the lock release or the "still alive" renewal is delayed; the lock service
  expires the lease and grants it elsewhere.
- **Clock issues** — TTL math depends on time; clock skew breaks it.

So a lock with a TTL alone is **not safe for correctness**. The client that "lost" the lock doesn't
know it lost it.

### The fix: fencing tokens

The lock service hands out a **monotonically increasing token** with each grant. Every write to the
*protected resource* carries the token, and the resource **rejects any token lower than the highest it
has seen.** Now a stalled-then-resumed client writes with an old token and gets rejected — the resource
itself enforces mutual exclusion.

```
Client A acquires lock -> token 33 -> [long GC pause] ...
Lock expires. Client B acquires lock -> token 34 -> writes to storage (storage records 34)
Client A wakes up, writes with token 33 -> storage sees 33 < 34 -> REJECTED. Safe.
```

> **Say this in the room:** "A TTL lock is not enough for correctness — a GC pause can make two
> clients think they hold it. The fix is a **fencing token**: a monotonic number from the lock
> service that the protected resource validates and rejects-if-stale. The resource is the final
> arbiter, not the lock. Zab/Raft give you these tokens for free (zxid / Raft index)."

The deep point: **a lock alone can never be safe; safety requires the resource to participate** via
fencing. If the resource can't check a token, you fundamentally can't have a correctness-safe
distributed lock — you can only have an *efficiency* lock (mostly prevents duplicate work, but
occasionally won't).

### Two kinds of locks — be explicit about which you need

- **Efficiency lock** — "usually only one worker does this job; a rare double-run is fine (idempotent
  or just wasteful)." A simple Redis `SET NX PX` lock is acceptable here.
- **Correctness lock** — "a double-run corrupts data / double-charges a customer." You **need fencing
  tokens** and a resource that validates them. A TTL lock alone is not acceptable.

### The Redlock debate (present both sides honestly)

**antirez** (Redis author) proposed **Redlock**: acquire the lock on a majority of N independent Redis
masters with a TTL; you hold it if you got a majority within the validity time. Goal: a distributed
lock without a single Redis being a SPOF.

**Kleppmann's critique:**
1. Redlock relies on **bounded clocks and bounded pauses** for its TTL safety — but GC pauses and
   clock jumps violate exactly those assumptions, so Redlock can grant the same lock to two clients.
   And critically, **Redlock has no fencing token**, so the resource can't catch the overlap.
2. It depends on timing for *correctness*, which a consensus-based system (with a real log/term as the
   fence) does not.

**antirez's rebuttal:** the clock assumptions are reasonable in practice (modest, monotonic-ish
drift), fencing can be layered on top, and for many real use cases Redlock's safety is adequate.

> **The honest staff answer:** "Both are partly right. Kleppmann is correct that **no TTL-based lock
> is safe for correctness without fencing tokens** — that's a real, fundamental gap, not a tuning
> issue. antirez is correct that for **efficiency** locks (avoiding duplicate work, not preventing
> corruption) Redlock is fine and operationally simple. My rule: **need correctness → use a
> consensus system (etcd/ZooKeeper) for the lock AND fence the resource with the token. Need
> efficiency only → a single-Redis or Redlock TTL lock is acceptable.** Either way, the moment
> correctness is on the line, the *resource* must validate a fencing token — the lock service alone
> can never guarantee it."

**The correct general pattern:** **lease + fencing token, validated at the resource.** Lease for
liveness (auto-release on crash), monotonic token for safety (reject stale writers).

---

## Part G — Leader election & split-brain prevention

Leader election is just consensus on "who's the leader for this term." Common patterns:

- **Via coordination service** (the default): contend to create the same **ephemeral** znode/lease;
  winner is leader, and if it dies the ephemeral node vanishes and triggers a new election. Use
  **sequential** nodes + watch-your-predecessor so you don't get a thundering herd of all candidates
  waking up at once.
- **Embedded consensus** (Raft/Multi-Paxos): the protocol elects its own leader as part of operation
  (etcd, Consul, CockroachDB ranges).

**Split-brain** = two nodes both believe they're leader and both act on it. Prevention:

1. **Majority quorum** — a leader must hold a majority lease; only one majority exists, so only one
   leader can *act*. (This is why a 2-node setup is dangerous — no majority is possible.)
2. **Fencing tokens / epochs** — even if two think they're leader, the stale one's writes are rejected
   by their monotonic epoch (same mechanism as locks).
3. **Leases with the holder backing off** — a leader that *can't* refresh its lease must **stop acting
   as leader before** the lease expires (account for clock skew), so the next leader can safely take
   over.

> **Say this in the room:** "I prevent split-brain two ways at once: a majority lease so only one
> leader can commit, and epoch/fencing tokens so a deposed leader's late writes are rejected
> downstream. And I never run a 2-node cluster — there's no majority, so any partition is fatal."

---

## Part H — When you DON'T need consensus (most of the time)

Consensus is **expensive**: every decision is a network round-trip to a majority (latency), and during
a partition the minority side **stops serving writes** (you've chosen C over A). If you put consensus
on your hot data path, you've capped your throughput at one Raft group's commit rate and made yourself
unavailable under partition.

So the staff instinct: **minimize consensus. Push it to the control plane; keep the data plane
consensus-free.**

You **don't** need consensus when:
- Operations are **commutative / idempotent** — order doesn't matter, so nothing to agree on.
- **Eventual consistency** is acceptable (feeds, likes, view counts, caches) — use replication +
  conflict resolution (LWW, CRDTs), not consensus.
- A **single writer / partition owner** already serializes things (sharded systems where each shard
  has one owner — the *only* consensus you need is electing that owner, in the control plane).
- You can use **quorum reads/writes (W+R>N)** for tunable consistency without full ordering (Dynamo
  style).

You **do** need consensus when:
- You must elect **exactly one** leader / lock holder (control plane).
- You need a **single agreed order** of operations (replicated state machine, distributed
  transactions, a config that everyone must see identically).
- **Membership / config** changes must be globally agreed.

> **Say this in the room:** "Consensus is a control-plane tool, not a data-plane tool. I use it to
> elect leaders and agree on config/membership — small, infrequent, critical decisions — and I keep
> it off the request path. The data plane runs on a single shard owner plus replication, so reads and
> writes don't pay a quorum round-trip and stay available under partition."

This is the most important judgment signal in the whole topic. The cost of consensus is **latency +
loss of availability under partition** — so you spend it sparingly, on the few decisions that truly
require one global answer.

---

## Part I — Ordering *without* consensus: logical clocks (and bounded-clock ordering)

Often you don't need *agreement*, you only need a consistent notion of **order / causality** — which is
much cheaper.

- **Lamport clocks** — a single counter per node, incremented on each event and bumped to
  `max(local, received)+1` on receive. Gives a **total order** consistent with causality: if A
  causally precedes B, then `L(A) < L(B)`. **Caveat:** the converse isn't true — `L(A) < L(B)` does
  *not* mean A caused B. Cheap, great for "pick a consistent tiebreak order."
- **Vector clocks** — a vector with one counter per node. Lets you **detect concurrency**: you can
  tell whether A → B, B → A, or A *concurrent with* B. This is how Dynamo/Riak detect conflicting
  writes (then resolve via LWW or app logic). **Caveat:** size grows with the number of nodes.
- These give **ordering without a quorum round-trip** — no leader, no majority, no stalling under
  partition. The trade: they tell you about causality, but they don't *decide* anything for you — you
  still need a conflict-resolution policy.

**Bounded-clock ordering (the Spanner family):** if you can *bound clock uncertainty*, you can order by
timestamp globally without per-operation consensus.

- **TrueTime (Google Spanner)** — GPS + atomic clocks give every node a time *interval* `[earliest,
  latest]` with a bounded error ε (a few ms). To guarantee external consistency, Spanner **waits out
  the uncertainty** (`commit-wait`) so timestamps never overlap across transactions. Result: globally
  consistent ordering and linearizable transactions across the planet — but it costs real money in
  special hardware and a few ms of commit latency.
- **Hybrid Logical Clocks (HLC)** — used by CockroachDB, YugabyteDB. Combine a physical timestamp with
  a logical counter, so you get *close-to-physical* timestamps that also respect causality, **without**
  special hardware. The pragmatic, commodity-hardware answer to "I want Spanner-ish ordering without a
  GPS antenna."

> **Say this in the room:** "If I only need causal *order*, not *agreement*, I reach for logical clocks
> — Lamport for a consistent total order, vector clocks to detect concurrent writes — and skip
> consensus entirely. If I need globally ordered transactions, that's Spanner's TrueTime (bounded
> clock + commit-wait) or HLCs (CockroachDB) on commodity hardware."

---

## Part J — "Do I need consensus here?" decision guide

Walk this top-to-bottom; stop at the first match:

1. **Is the operation commutative or idempotent, and is eventual consistency OK?**
   → No consensus. Replicate + resolve (LWW / CRDT). *(likes, counters, feeds, caches)*
2. **Do I only need to know the order/causality, not force one global decision?**
   → No consensus. **Logical clocks** (Lamport / vector). *(conflict detection, debugging causality)*
3. **Is there a single owner/shard that already serializes writes?**
   → Consensus only to **elect that owner** (control plane). Data path stays consensus-free.
4. **Do I need exactly one leader / lock holder, or globally agreed config/membership?**
   → **Yes, consensus.** Use etcd/ZooKeeper; don't hand-roll. Keep it small and off the hot path.
5. **Do I need a single agreed *total order* of operations (replicated state machine / distributed
   txn)?**
   → **Yes, consensus** (Raft/Multi-Paxos), or bounded-clock ordering (TrueTime/HLC) if it's
   timestamp-orderable transactions across regions.
6. **Distributed lock for correctness (double-spend, corruption)?**
   → Consensus-backed lock (etcd/ZK lease) **plus fencing token validated at the resource.** A TTL
   lock alone is not sufficient.
7. **Distributed lock for efficiency only (avoid duplicate work)?**
   → A simple TTL lock (single Redis / Redlock) is acceptable; document that a rare double-run is
   tolerated.

> **Mental shortcut:** "One global answer required? → consensus, control plane only. Just need order?
> → logical clocks. Just need it to mostly work? → leases. Locking for correctness? → lease +
> fencing token, validated at the resource."

---

### Self-check before the mock (answer these from memory)
- [ ] State what consensus guarantees (agreement/validity/termination) and which one is sacrificed under partition.
- [ ] Explain FLP in one sentence and name the two ways real systems sidestep it.
- [ ] Why 2f+1, and why odd node counts? What does quorum intersection buy you?
- [ ] Draw Raft: the three states, terms, randomized election timeouts, why randomization matters.
- [ ] Explain Raft log replication and what "committed" means (majority durable, not yet applied).
- [ ] State the two Raft safety rules — election restriction and "only commit current-term entries."
- [ ] Walk a network partition through Raft and explain why there's no split-brain.
- [ ] Why does Raft exist instead of Paxos? Name Zab and EPaxos in a line each.
- [ ] Name ZooKeeper's primitives (znodes, ephemeral, sequential, watches) and etcd's (leases, CAS, watches).
- [ ] Why is a TTL lock unsafe for correctness? Explain fencing tokens with the GC-pause example.
- [ ] Give both sides of the Redlock debate and state your rule for efficiency vs correctness locks.
- [ ] Name three ways to prevent split-brain.
- [ ] List three cases where you do NOT need consensus.
- [ ] Lamport vs vector clocks: which one detects concurrency, and what's the caveat on each?
- [ ] What does TrueTime do, and what's the HLC alternative for commodity hardware?
