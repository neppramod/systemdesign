# Topic 10: Distributed Transactions, Sagas & Idempotency — Correctness Across Services

> **Why this topic decides rounds:** the moment your design has two stores — two services, two
> shards, a DB + a queue — you have *given up the single-machine ACID transaction* whether you
> admit it or not. Staff candidates are the ones who say so out loud, name the failure window, and
> then engineer correctness back in with sagas, outboxes, and idempotency. Junior candidates draw
> `Order → Payment → Inventory` with three arrows and never mention that the middle one can fail
> after the first one committed. This doc is about the arrow that fails.

---

## Part A — Why you can't just "use a transaction"

A local ACID transaction works because **one storage engine** owns all the rows, one WAL orders
the writes, and one commit makes them all visible atomically. The instant your two writes live in
two systems, no single component can guarantee both-or-neither. There is no shared lock manager, no
shared log, no shared clock.

### The dual-write problem (the bug you will be asked to find)

The canonical anti-pattern — write to the DB, then publish an event:

```
db.save(order)            // (1) committed
queue.publish(orderEvent) // (2) ... process crashes here
```

Now the DB has the order but no event was ever published. Inventory never reserves stock, the
confirmation email never sends. Flip the order and it's worse:

```
queue.publish(orderEvent) // (1) sent
db.save(order)            // (2) fails / crashes
```

Consumers act on an order that doesn't exist. **These two writes are not atomic and cannot be made
atomic by ordering them.** Wrapping them in a try/catch doesn't help — the crash can land *between*
a successful commit and the next line. Retrying the publish doesn't help either, because you don't
know whether step (1) actually committed before the crash.

> **Say this in the room:** "DB-write-then-publish is a dual write. There's a crash window between
> the two where they diverge. I won't hand-wave it — I'll make the write and the 'intent to
> publish' commit in the *same* local transaction using the outbox pattern, and relay it
> asynchronously."

The same problem exists for **two services** (charge the card in Payments, then decrement stock in
Inventory) and for **two shards** (move money from an account on shard A to one on shard B). One
local transaction can't span them.

The three honest options, and the rest of this doc:

1. **Make it one transaction anyway** — 2PC. Strong, blocking, fragile. Rarely worth it.
2. **Sequence local transactions + compensations** — the saga. The default for business workflows.
3. **Atomic local write + async relay + idempotent consumers** — outbox/inbox + at-least-once + idempotency. The plumbing under #2.

---

## Part B — Two-Phase Commit (2PC) and why we mostly avoid it

### How it works

A **coordinator** drives all participants through two phases:

- **Phase 1 — Prepare/Vote.** Coordinator asks each participant "can you commit?" Each does the
  work, writes it durably to its log, **takes locks**, and replies *yes* (a promise it can commit
  if asked) or *no*. After voting yes, a participant is not allowed to unilaterally abort.
- **Phase 2 — Commit/Abort.** If *all* voted yes, coordinator writes a commit decision to its own
  log and tells everyone to commit. If any voted no (or timed out), it tells everyone to abort.

The atomicity comes from the coordinator's durable decision: once it logs "commit," that is the
truth, and every participant must eventually apply it.

### Why it's blocking and fragile

- **Held locks across the round trip.** Between vote-yes and the commit message, every participant
  holds its locks. That's two network round trips of lock contention on hot rows.
- **Coordinator is a single point of failure — the killer.** If the coordinator crashes *after*
  participants voted yes but *before* sending the decision, participants are **stuck**. They can't
  commit (maybe someone voted no) and can't abort (maybe everyone voted yes and the decision was
  commit). They sit holding locks until the coordinator recovers. This is the **blocking** problem,
  and it's why 2PC is a liveness hazard.
- **Participant failure** after voting yes means the coordinator waits or the participant must
  recover and ask "what was decided?" — requiring a recovery log on every node.
- It assumes a relatively reliable, low-latency network. Across services/regions it's a tax on
  every request and a correlated-failure magnet.

### 3PC and why it's rarely used

3PC inserts a **pre-commit** phase so participants learn the decision is *going* to be commit before
they actually commit, which lets a recovering node make progress without the coordinator — it's
**non-blocking under a fail-stop model**. But it adds another round trip, and it breaks under
**network partitions** (a partitioned group can wrongly decide to commit while the other aborts).
In practice nobody runs 3PC; if you need partition-tolerant agreement you reach for a **consensus
protocol (Raft/Paxos)**, which is what databases like Spanner/CockroachDB actually use under the
hood (2PC for the cross-shard atomicity, *layered on Paxos/Raft groups* so each participant and the
coordinator decision are themselves highly available — that's the trick that makes 2PC tolerable).

### When 2PC is actually OK

- **Inside a single trusted datacenter, low latency, few participants**, where strong consistency is
  non-negotiable — e.g. a distributed SQL DB doing a cross-shard write, or `XA` transactions across
  a DB and a JMS broker in a legacy enterprise stack.
- When the coordinator is itself **replicated via consensus** so it can't single-point-fail (the
  Spanner approach). That removes the blocking problem at the cost of more machinery.

> **Say this in the room:** "I'd avoid raw 2PC across microservices — coordinator failure blocks
> everyone holding locks, and it couples availability. The one place I'd accept it is a single-DC,
> consensus-backed coordinator like a distributed SQL store does internally. For a business workflow
> spanning services, I'll use a saga."

---

## Part C — The Saga pattern (the staff default)

A **saga** replaces one distributed transaction with a **sequence of local transactions**, each in
its own service/DB, plus a **compensating transaction** for each step that semantically undoes it if
a later step fails. There is no global atomicity and no global lock — only **eventual consistency**
plus the ability to roll *forward* or roll *backward*.

Key mental shift: a saga gives you **A**tomicity-of-outcome (eventually all done or all compensated)
and **D**urability, but **sacrifices Isolation** — intermediate states are visible to others. Most
of the hard saga work is managing that lost isolation.

### Orchestration vs choreography

**Orchestration** — a central **saga orchestrator** (a state machine) tells each service what to do
and listens for replies, deciding the next step or compensation.

```
Orchestrator ──cmd──▶ Payment ──reply──▶ Orchestrator ──cmd──▶ Inventory ──▶ ...
```

- Pros: explicit, centralized logic; easy to see the whole flow and where it is; easy to add steps,
  add timeouts, query status. Compensation logic lives in one place.
- Cons: the orchestrator is another service to build/run; risk of it becoming a god-object that
  holds all business logic.

**Choreography** — no central brain. Each service **emits events**; other services subscribe and
react, emitting their own events.

```
OrderCreated ─▶ Payment (charges) ─▶ PaymentCompleted ─▶ Inventory (reserves) ─▶ ...
```

- Pros: maximally decoupled, no central bottleneck.
- Cons: the workflow is **implicit and emergent** — no one place tells you the flow; hard to reason
  about, debug, or detect "stuck" sagas; cyclic event dependencies sneak in; compensation logic is
  smeared across services.

> **Say this in the room:** "For anything beyond ~3-4 steps, or anything with money and complex
> compensation, I default to **orchestration** — the explicit state machine is worth it for
> debuggability and operability. Choreography is fine for simple, loosely-coupled reactions."

### Compensating transactions

A compensation is a **new transaction that semantically reverses** a completed step — not a rollback
(the step already committed). `ChargePayment`'s compensation is `RefundPayment`, not "un-commit."

Properties compensations must have:
- **Idempotent** — they'll be retried (you may not know if the first compensation landed).
- **Commutative-enough / order-tolerant** — compensations run in reverse order, but design so a
  duplicate or reordered compensation is safe.
- **Always eventually succeed** (retried forever) — a compensation that can *fail permanently* is a
  design bug. If `RefundPayment` could be rejected, you have an unrecoverable saga.

Some steps are **not compensatable** ("send the email", "ship the package", "fire the missile").
Structure the saga so non-compensatable steps come **last** (a *pivot* transaction — once you pass
it, you only roll forward). Steps before the pivot are *compensatable*; steps after are *retriable*.

### Semantic locks (regaining a little isolation)

Because isolation is gone, you mark in-flight rows with an application-level lock — a status flag
like `PENDING` / `RESERVED` / `APPROVAL_PENDING`. Other transactions see the flag and either wait,
fail fast, or take a different path. Compensation clears or reverts the flag. This is a **semantic
lock**: not a DB lock, just a state machine field that says "don't treat this as final yet."

### Isolation anomalies in sagas, and countermeasures

Because intermediate steps are visible, you get the classic anomalies:

- **Dirty reads** — someone reads `RESERVED` inventory that later gets compensated back.
- **Lost updates** — two sagas read the same value and step on each other.
- **Fuzzy/non-repeatable reads** — a value changes mid-saga.

Countermeasures (Garcia-Molina / Richardson's saga toolkit — name these):
- **Semantic lock** — the status flag above; readers honor it.
- **Commutative updates** — design ops so order doesn't matter (e.g. `increment(-10)` instead of
  `set(balance, 90)`), so a reordered compensation still lands correctly.
- **Pessimistic view** — reorder saga steps so the risky/visible state is minimized (do the
  reversible reservation early, the irreversible action late).
- **Reread value** — before acting, re-read and verify it hasn't changed (optimistic check).
- **Version file / by value** — record operations and replay/sort them; route high-risk requests
  through stricter handling.

### Worked example: place an order

Happy path (4 local transactions, orchestrated):

| Step | Service | Local txn | Compensation |
|---|---|---|---|
| 1 | Order | create order `status=PENDING` | mark `CANCELLED` |
| 2 | Payment | charge card, `status=CHARGED` (semantic lock: funds authorized) | `RefundPayment` |
| 3 | Inventory | reserve stock, `status=RESERVED` | `ReleaseReservation` |
| 4 | Shipping | create shipment, order `status=CONFIRMED` (**pivot — non-compensatable once dispatched**) | (none — roll forward only) |

**Failure case — inventory is out of stock at step 3:**

```
Orchestrator: CreateOrder(PENDING)        ✓
Orchestrator: ChargePayment               ✓  (card charged)
Orchestrator: ReserveInventory            ✗  OUT_OF_STOCK
  → compensation phase, reverse order:
Orchestrator: RefundPayment               ✓  (compensate step 2, idempotent)
Orchestrator: CancelOrder(CANCELLED)      ✓  (compensate step 1)
  → saga ends in a consistent, fully-compensated state
```

The customer's card was charged for a few seconds and then refunded — *that brief visible state is
the price of dropping isolation*, and it's acceptable for an order flow. (For real payments you'd
**authorize then capture** so you never actually move money until inventory is reserved — see Part
F; this reorders the saga so the pivot is later.)

**Partial completion / crashes:** the orchestrator persists saga state after every step (its own
local txn — see outbox below). On restart it reads "last completed step = ChargePayment, next =
ReserveInventory" and resumes. Every command is **idempotent**, so re-sending `ReserveInventory`
after a crash is safe. A saga is never "lost" if its state is durable; it's only ever *in progress*,
*completed*, or *compensated*.

> **Say this in the room:** "Each step is a local ACID txn. Failures don't roll back — they trigger
> compensations in reverse. I keep isolation anomalies in check with semantic-lock status flags and
> commutative updates, and I order steps so the non-compensatable one is the pivot at the end."

---

## Part D — Outbox + CDC (atomic "update DB and publish event")

This is the concrete fix for the dual-write problem and the plumbing under every saga step.

### The outbox pattern

Write the business change **and** an event row into the **same database, same local transaction**:

```
BEGIN;
  INSERT INTO orders (...) VALUES (...);
  INSERT INTO outbox (id, aggregate, type, payload, created_at)
         VALUES (uuid, 'order', 'OrderCreated', '{...}', now());
COMMIT;   -- both or neither. No dual write.
```

A separate **relay** process reads unpublished outbox rows and publishes them to the broker, marking
them sent (or deleting them). Now the *only* atomicity you needed was local, which the DB gives you
for free.

Two ways to run the relay:
- **Polling publisher** — `SELECT … WHERE published=false ORDER BY id`, publish, mark. Simple,
  works anywhere; adds polling latency and DB load.
- **Change Data Capture (CDC)** — tail the DB's replication log/WAL (Debezium reading MySQL binlog /
  Postgres logical replication) and stream new outbox rows to Kafka. No polling, low latency, no
  extra query load, but more moving parts.

**Critical:** the relay is **at-least-once**. It can crash after publishing but before marking the
row sent, so it republishes. Therefore **every event is published one-or-more times** → consumers
must dedup. That's the inbox pattern.

### The inbox pattern (consumer-side dedup)

Each consumer records the IDs of messages it has already processed and **processes the side-effect
and the dedup record in one local transaction**:

```
BEGIN;
  -- abort if already seen:
  INSERT INTO inbox (message_id) VALUES (:id);   -- PK / unique → conflict if dup
  ... apply the business effect ...
COMMIT;
```

If the message is a duplicate, the unique-key insert fails, the txn aborts, and the effect is not
re-applied. The inbox table (with a TTL/cleanup job) is your **dedup store**. This is how
at-least-once delivery becomes effectively-once *processing*.

> **Say this in the room:** "Outbox makes the DB write and the event atomic via a local txn; a relay
> (polling or CDC/Debezium) ships it at-least-once; the consumer dedups with an inbox table keyed on
> message id, processed in the same local txn as the effect. That's the whole exactly-once illusion."

---

## Part E — Idempotency, in depth

**Idempotent** = applying the operation N≥1 times has the same effect as applying it once. This is
the single most important correctness primitive in distributed systems, because **everything
retries** (clients on timeout, queues on at-least-once, sagas on resume).

### Idempotency keys (the Stripe model)

The client generates a unique **idempotency key** per logical operation and sends it with the
request (and with every retry of that same request):

```
POST /charges
Idempotency-Key: 7f3c-...-client-generated-uuid
{ amount: 5000, currency: "usd", source: ... }
```

Server behavior:
1. Look up the key in a **dedup table** (`idempotency_keys`).
2. **First time:** insert the key (unique constraint → this also guards against concurrent
   duplicates), perform the operation, **store the response** against the key, return it.
3. **Replay:** key already exists → return the **stored response**, do *not* perform the operation
   again. Same charge id, same result, no double charge.

Subtleties that earn staff points:
- **Atomicity of "check + insert."** Use the DB unique constraint, not "SELECT then INSERT" — two
  concurrent retries race otherwise. The loser of the insert either waits for the winner's stored
  response or returns 409.
- **In-flight requests.** If the first request is still processing when the retry arrives, return
  `409 Conflict`/retry-later rather than running it twice.
- **Bind the key to the request fingerprint.** Stripe hashes the request body; reusing a key with a
  *different* payload is an error (someone reused a key by accident). Prevents "same key, different
  amount."
- **TTL.** Keys expire (Stripe: 24h). The dedup table is not infinite; you keep keys long enough to
  cover all realistic retries, then GC. Choose TTL > max retry window.
- **Scope.** Key is unique per account/endpoint, not globally, to avoid cross-tenant collisions.

### Making operations *naturally* idempotent

Better than a dedup table when you can manage it — design the op so duplicates are harmless:
- **Upsert by a stable business key** (`INSERT … ON CONFLICT DO NOTHING/UPDATE`) instead of blind
  insert.
- **Absolute, not relative, writes** — `SET status='SHIPPED'` is idempotent; `increment(count)` is
  not. `SET balance=90` is idempotent; `balance -= 10` is not (use a transfer-id-keyed ledger entry
  instead, see Part F).
- **Conditional updates / compare-and-set** — `UPDATE … WHERE status='PENDING'`; a replay finds the
  row already `SHIPPED` and updates zero rows.
- **Idempotent by construction with a client-supplied id** — the write *is* the dedup (PK = the
  operation id).

### Idempotent ≠ exactly-once

- **Idempotent** = safe to apply *the same* operation multiple times.
- **Exactly-once** = the *effect* happens once despite duplicate *deliveries*.

Idempotency is the **mechanism** by which you *achieve* effectively-once on top of at-least-once
delivery. They are not the same claim, and conflating them is a red flag.

---

## Part F — Exactly-once demystified

> **The line to deliver:** "True exactly-once *delivery* is impossible across an unreliable network.
> What's achievable is **effectively-once**, which is **at-least-once delivery + idempotent
> processing** (or at-most-once + reconciliation). When someone says exactly-once, they mean this."

Why true exactly-once delivery is impossible: after the sender transmits and before it learns the
outcome, the network can drop the **message** or drop the **ack**. The sender cannot distinguish
"receiver got it, ack lost" from "receiver never got it." So it must choose:
- **At-most-once** — send and don't retry on ack-loss. Never duplicates, may **lose** messages.
- **At-least-once** — retry until acked. Never loses, may **duplicate**.

You cannot have "exactly once" on the wire. The Two Generals problem is the formal statement of why.

So we engineer it at the **processing** layer: deliver at-least-once, and make processing idempotent
(idempotency keys, inbox dedup, natural idempotency). The *delivery* duplicated; the *effect*
happened once. Kafka's "exactly-once semantics" is exactly this — idempotent producer (dedup by
producer-id + sequence number) + transactional consume-process-produce within Kafka — *not* magic
across your external side-effects. Your charge-the-card call still needs an idempotency key.

---

## Part G — Distributed ID generation (for idempotency keys & ordering)

You need unique ids for outbox rows, idempotency keys, ledger entries, and sortable event streams.
Options and tradeoffs:

### UUID v4 (random)
- 128-bit random. **No coordination**, generate anywhere, collision-free in practice.
- **Not sortable** (random) → terrible as a B-tree primary key: random inserts fragment the index,
  kill write locality. Big (16 bytes / 36 chars).
- Use when: you only need uniqueness, not order (dedup keys, request ids). **UUID v7** fixes
  sortability (time-ordered prefix) and is the modern default for DB keys.

### Snowflake (Twitter) — the one to draw
A **64-bit, time-sortable** id assembled from a timestamp + machine id + per-ms sequence. **Roughly
coordination-free** (each node only needs a unique machine id) and **k-sortable** (ids sort by time).

```
 0 | 41 bits timestamp (ms since custom epoch) | 10 bits machine id | 12 bits sequence
 ^   ^                                            ^                    ^
 |   |                                            |                    └ 4096 ids per machine per ms
 |   |                                            └ 1024 machines (e.g. 5 bits DC + 5 bits worker)
 |   └ ~69 years of ms before rollover (2^41 ms ≈ 69.7 yr)
 └ sign bit, kept 0 so ids are positive
```

- **41-bit ms timestamp** → `2^41 / (1000·60·60·24·365) ≈ 69.7` years from your custom epoch.
- **10-bit machine id** → 1024 nodes. Split as datacenter + worker.
- **12-bit sequence** → 4096 ids per node per millisecond → **~4.096M ids/node/sec** before you must
  wait for the next ms.
- Top sign bit stays 0 to keep the integer positive/sortable.

Tradeoffs / gotchas to mention:
- **Clock skew / NTP rollback** is the enemy — if a node's clock goes backwards you can generate
  duplicate or out-of-order ids. Real impls **refuse to generate** (or wait) until the clock catches
  up. This is the staff-level detail interviewers fish for.
- Machine-id assignment needs coordination *once* (config, or ZooKeeper/etcd lease) but not per-id.
- Leaks creation time (sometimes a privacy concern).

### ULID
- 128-bit: 48-bit ms timestamp + 80 bits randomness, Crockford base32, lexicographically sortable.
- Like a sortable UUID; no coordination, more entropy than Snowflake's sequence, larger than 64-bit.
- Use when you want sortable + decentralized + don't want to manage machine ids. (UUIDv7 is the
  near-equivalent in the standard.)

### DB ticket server / segment allocation
- A central DB hands out ranges: a node grabs a **block** of ids (e.g. 1000 at a time) under one
  txn, then serves them locally. Amortizes coordination; sortable; simple.
- Tradeoff: the allocator is a **SPOF / hot spot**; mitigate with two servers handing out
  odd/even (Flickr's trick) or multiple ranges. Used by Instagram-style sharded id schemes too.

| Scheme | Sortable | Coordination | Size | Hot-spot risk |
|---|---|---|---|---|
| UUIDv4 | no | none | 128b | none (but bad index locality) |
| UUIDv7/ULID | yes | none | 128b | low |
| Snowflake | yes | machine-id once | 64b | clock skew |
| Ticket server | yes | central allocator | 64b | allocator SPOF |

> **Say this in the room:** "For a dedup key I just need uniqueness → UUID. For a sortable id that's
> coordination-free at scale → Snowflake: 41-bit ms timestamp, 10-bit machine id, 12-bit sequence,
> 64 bits, ~69 years, 4096 ids/node/ms. The real risk is clock skew, so I refuse to emit ids if the
> clock moves backward."

---

## Part H — Handling money: ledgers, double-entry, reconciliation

Money is the highest-stakes correctness problem and a favorite deep-dive. The rules:

### Append-only, double-entry ledger
- Model balances as a **ledger of immutable entries**, not a single mutable `balance` column. The
  balance is a **derived sum** (or a periodically-snapshotted materialized view) of entries.
- **Double-entry:** every movement writes (at least) two entries that sum to zero — a debit and a
  credit. A $10 transfer = `-10` from A, `+10` to B, tagged with one `transfer_id`. The invariant
  "all entries for a transfer sum to zero" is your built-in consistency check.

```
transfer_id  account  amount   type
   txn-abc    A        -10.00   debit
   txn-abc    B        +10.00   credit       (sums to 0 → balanced)
```

### Why you never delete or mutate ledger rows
- **Auditability / regulatory** — the ledger is the source of truth and must be reconstructable.
- **Idempotency & correctness** — append-only + unique `transfer_id` makes posting **naturally
  idempotent**: a retried transfer collides on `transfer_id` and is a no-op. Mutating a balance in
  place is *not* idempotent and races under concurrency.
- **Corrections are new entries** — to fix an error you post a **reversing entry** (and then the
  correct one), never an `UPDATE`/`DELETE`. The history shows the mistake *and* the fix.

### Reconciliation
- Independently recompute balances from entries and compare against running totals / external
  systems (the payment processor, the bank statement) on a schedule.
- Drift = a bug or a missed/duplicated event → alert and investigate. This is the **safety net that
  catches everything the happy path missed**, and the reason eventual-consistency designs are
  acceptable for money: you don't need synchronous strong consistency if you have authoritative
  asynchronous reconciliation.

> **Say this in the room:** "Balances are a derived sum over an append-only double-entry ledger.
> Entries are immutable — corrections are reversing entries. Posting is idempotent via a unique
> transfer id, and a reconciliation job recomputes balances and diffs them against the processor to
> catch drift."

---

## Part I — Decision guide: strong txn vs saga vs eventual + reconciliation

| Need | Reach for | Why |
|---|---|---|
| Multiple rows, **one** DB/shard | **Local ACID transaction** | Free, strong, simple. Don't overthink it. |
| Cross-shard write, single DC, strong consistency required | **2PC on consensus-backed nodes** (distributed SQL) | Atomic + available coordinator; accept latency. |
| Multi-step **business workflow** across services (order, booking, signup) | **Saga (orchestrated)** + outbox/inbox | No global lock; compensations; debuggable state machine. |
| Atomic "update DB + emit event" | **Outbox + CDC**, inbox dedup | Kills the dual-write window with a local txn. |
| Idempotent external side-effect (charge, send) | **Idempotency key + dedup table** | Effectively-once on at-least-once delivery. |
| High-throughput, can tolerate brief divergence, money/critical | **Eventual consistency + reconciliation** | Async authoritative re-check beats synchronous coordination. |
| Just need order/uniqueness | **Snowflake / UUIDv7 / ledger transfer-id** | Coordination-free correctness primitive. |

**The flowchart in your head:**
1. Does it fit in **one transaction boundary** (one DB/shard)? → use it, stop.
2. Does it span services but tolerate intermediate visibility? → **saga**, orchestrated, with
   compensations + semantic locks.
3. Are you publishing events off a DB write? → **outbox**.
4. Will anything be retried (it will)? → **idempotency keys / inbox / natural idempotency**.
5. Is it money or otherwise unforgiving? → **append-only ledger + reconciliation** on top of all of
   the above.

> **The one-sentence philosophy:** you don't get distributed atomicity for free, so you **trade it
> for eventual consistency plus relentless idempotency and reconciliation** — at-least-once
> everywhere, idempotent everywhere, and an independent job that catches what slipped through.

---

### Self-check before the mock (answer these from memory)
- [ ] Explain the dual-write problem and how the outbox pattern fixes it.
- [ ] Why is 2PC blocking, and what single failure causes it? When is 2PC acceptable?
- [ ] Orchestration vs choreography sagas — when do you pick each?
- [ ] Walk the order → payment → inventory → shipping saga, including the compensation path.
- [ ] Name three saga isolation anomalies and a countermeasure for each.
- [ ] How does a Stripe-style idempotency key work, including the concurrent-retry race?
- [ ] Why is true exactly-once delivery impossible, and what is "effectively once"?
- [ ] Draw the Snowflake 64-bit layout and give the year span and ids/node/ms. What breaks it?
- [ ] Why is a ledger append-only and double-entry, and how is posting made idempotent?
- [ ] Give the decision rule: local txn vs saga vs eventual + reconciliation.
