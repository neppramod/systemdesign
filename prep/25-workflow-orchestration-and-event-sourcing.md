# Topic 25: Long-Running Workflows, Durable Execution, Event Sourcing & CQRS

> **Why this topic exists:** Most system design prep is about *serving a request* — read this, write that,
> in 200ms. This topic is about the other half of the world: **business processes that take minutes,
> hours, or weeks, span many services, and must survive crashes mid-flight.** Order fulfillment.
> Employee onboarding. A data pipeline. A loan approval. These are state machines that live longer than
> any one process, and the naïve version — a queue here, a `status` column there, a cron job to clean up
> the stuck ones — rots into an unobservable mess. The senior move is knowing the *named patterns*
> (saga, durable execution, event sourcing, CQRS), knowing they are **not the same thing**, and knowing
> that the last two are the most over-applied ideas in the field. Half of staff-level signal here is
> saying "you don't need event sourcing for this."

---

## Part A — The Problem: long-running, multi-step, failure-prone processes

Take order fulfillment. The happy path is: reserve inventory → charge payment → create shipment →
notify warehouse → send confirmation email. Five steps, five services, and **every arrow can fail**:
the payment service times out, the warehouse API 500s, your own process gets OOM-killed between step 3
and step 4. The process can also *block* — waiting on a human approval, a 3rd-party webhook, a 7-day
return window. So the requirements that make this hard:

- **Durability across crashes.** If the orchestrator dies after charging the card but before creating
  the shipment, on restart it must *know* the card was charged and resume at step 3 — not re-charge,
  not skip.
- **Long duration.** The process may live for days. You cannot hold a thread, a DB transaction, or an
  HTTP connection open that long.
- **Partial failure + compensation.** If shipment creation permanently fails, you must *undo* the
  charge (refund) and release the inventory. There is no distributed transaction across these services.
- **Retries with backoff** on transient failures, **without** re-running the steps that already succeeded.
- **Timeouts and timers** ("if no warehouse ack in 1 hour, escalate").
- **External signals** ("a human approved", "the webhook arrived").
- **Observability**: when ops asks "where is order #12345 stuck and why?", you need an answer in seconds.

### The naïve approach and why it rots

> **The thing most teams build first:** a `status` enum column on the `orders` table
> (`PENDING → PAID → SHIPPED → DONE`), a few queues between services, and a cron job that scans for
> rows stuck in a state too long and pokes them.

This works at small scale and then becomes the system everyone fears to touch. The pain, named:

| Pain | Why it happens with ad-hoc queues + DB state machines |
|---|---|
| **State is scattered** | The "real" state of a process is smeared across a status column, several queues' in-flight messages, and retry counters. No single place tells you the truth. |
| **Resumption is manual** | After a crash, *what* resumes the half-done order? A cron sweeper you hand-wrote, with bespoke logic for every state transition. Every new step = new sweeper logic. |
| **Failure handling is copy-pasted** | Retry/backoff/timeout logic is re-implemented in every consumer, slightly differently, usually buggily. |
| **Compensation is ad-hoc** | Undo logic ("refund if shipment fails") lives wherever someone remembered to add it. Easy to forget a branch. |
| **No execution history** | "Why did order #12345 end up cancelled?" requires archaeology across logs of five services. |
| **Timers are a hack** | "Do X in 7 days" becomes a cron + a `due_at` column + a scanner. Fine until you have 40 of them. |
| **The distributed monolith** | Add a step and you touch five services and their deploy order. The "decoupled" services are secretly coupled by an implicit, undocumented protocol. |

The whole topic below is a ladder of increasingly powerful answers to this. Climb only as far as the
problem forces you.

---

## Part B — Saga, recapped and escalated (ties to Topic 10)

You can't run a 2PC distributed transaction across `inventory`, `payments`, and `shipping` —
2PC blocks, doesn't scale, and these are independent services with independent databases (see Topic 10
on distributed transactions). The standard answer is a **saga**: a sequence of *local* transactions,
each with a **compensating action** that semantically undoes it. If step N fails, run the compensations
for steps N-1 … 1 in reverse.

```
Reserve inventory   ── compensate ──>  Release inventory
Charge payment      ── compensate ──>  Refund payment
Create shipment     ── compensate ──>  Cancel shipment
```

Two ways to coordinate a saga — this is the **orchestration vs choreography** axis:

| | **Orchestration** | **Choreography** |
|---|---|---|
| **Coordination** | A central coordinator tells each service what to do next and listens for results | Each service reacts to events and emits new events; no central brain |
| **Control flow** | Explicit, lives in one place | Implicit, emergent from who-subscribes-to-what |
| **Observability** | Easy — the orchestrator *is* the state | Hard — flow is scattered across subscriptions |
| **Coupling** | Coordinator knows all services | Services don't know each other, but couple via event contracts |
| **Adding a step** | Edit the orchestrator | Wire up a new subscription somewhere |
| **Failure mode** | Coordinator is a critical component (make it durable!) | "Distributed monolith" — hidden coupling, hard to reason about |

> **Say this in the room:** "I'll use a saga for the cross-service consistency because we can't do a
> distributed transaction. The real question is *who drives the saga*. For a complex business process
> with branching and compensation, I want **orchestration** — a single place that holds the truth and
> is debuggable. Choreography is elegant for 2-3 simple steps, but past that it becomes a distributed
> monolith you can't trace."

And then the escalation: **a hand-written orchestrator that must survive crashes is exactly the
durable state machine that rots.** So the orchestrator itself wants to be durable. That's the bridge to
the next section — workflow engines are *durable saga orchestrators with batteries included*.

---

## Part C — Durable execution / workflow engines (Temporal, Cadence, Step Functions)

This is the headline pattern of the topic. **Durable execution** means: you write your workflow as
*ordinary sequential code* — `chargePayment(); createShipment(); sleep(7 days); sendSurvey();` — and the
engine guarantees that code **runs to completion exactly as written, even across process crashes,
deploys, and machine failures, for arbitrarily long.** The thread "sleeping for 7 days" doesn't actually
hold a thread; the engine persists progress and resurrects it.

### The mental model

Two kinds of code:

- **Workflow code** — the orchestration logic. *Deterministic.* This is the "what happens in what
  order, with what branching and compensation." It must be replayable.
- **Activities** — the actual side-effecting work: call payment API, write to DB, send email. These are
  *non-deterministic* and *retried* by the engine. The engine wraps them with retry/backoff/timeout.

### How it survives crashes: the event-history + replay trick

The engine does **not** snapshot your workflow's memory. Instead it keeps an **append-only event history**
of everything that happened: "workflow started with input X", "activity `chargePayment` scheduled",
"activity `chargePayment` completed with result Y", "timer started", "timer fired", "signal received".

When a worker picks up a workflow (fresh start, or after a crash, or after a deploy), it **re-executes
the workflow code from the top** — but every time the code calls an activity or starts a timer, the
engine checks the history: *did this already happen?* If yes, it returns the recorded result instantly
(it does **not** re-run the activity). If no, it actually schedules the work. This is **deterministic
replay**: re-running the code over the recorded history fast-forwards the workflow back to exactly where
it was, then continues live.

```
Workflow code:                      Event history (the source of truth):
  id = createOrder(req)      ──>     [WorkflowStarted req]
  ok = chargePayment(id)     ──>     [ActivityScheduled createOrder]
  ship = createShipment(id)  ──>     [ActivityCompleted createOrder -> id=42]
  sleep(7d)                  ──>     [ActivityScheduled chargePayment]
  sendSurvey(id)             ──>     [ActivityCompleted chargePayment -> ok]
                                     [ActivityScheduled createShipment]   <- crash here
                              ──>     (on restart: replay re-derives id=42, ok=true
                                       WITHOUT re-charging; resumes at createShipment)
```

This is the key insight to articulate: **the workflow's "state" is the event history, and the program
counter is reconstructed by replay.** That's why it can sleep 7 days using zero resources — when the
timer fires the engine just loads the history, replays, and the code "wakes up."

### Why this beats a hand-rolled state machine

| Hand-rolled state machine + queues | Durable execution engine |
|---|---|
| State in a status column you maintain | State in engine's event history, automatic |
| You write retry/backoff/timeout per step | Declarative per-activity retry policy |
| Timers = cron + `due_at` column + scanner | `sleep(duration)` / durable timers, first-class |
| Resume after crash = bespoke sweeper logic | Automatic replay, free |
| Compensation = scattered | Try/catch + compensation in normal code |
| "Where is it stuck?" = log archaeology | Full execution history + UI per workflow instance |
| Human-in-loop / webhook = polling hack | **Signals** delivered into the running workflow |
| Branching = enum spaghetti | `if`/`for`/`while` in real code |

> **The pitch in one sentence:** "A workflow engine turns 'a crash-resilient distributed state machine'
> into 'ordinary sequential code that happens to be durable,' and gives me retries, timers, signals, and
> a per-instance audit UI for free."

### The determinism constraint (the catch you MUST mention)

Because the engine recovers state by **re-running the workflow code over recorded history**, the
workflow code must be **deterministic** — replaying it must produce the exact same sequence of
decisions. Two replays must agree. So inside workflow code you may **not**:

- Call `now()`, `random()`, `uuid()` directly → use the engine's deterministic versions (it records the
  value once in history).
- Do I/O, network calls, DB reads directly → those go in **activities**.
- Read mutable global state, depend on map iteration order, spawn raw threads.
- **Change the code in a way that diverges from in-flight histories** — this is the versioning hazard
  (see Part G).

If replay diverges from history, you get a *non-determinism error* — the engine's way of refusing to
corrupt state. Mentioning this constraint, unprompted, is a strong senior signal: it shows you
understand *why* the magic works.

### The landscape

| Engine | Model | Notes |
|---|---|---|
| **Temporal** / **Cadence** | Code-as-workflow, deterministic replay | Most powerful; workflows are real code in Go/Java/TS/etc. Cadence is the Uber original; Temporal the popular fork. |
| **AWS Step Functions** | Workflow-as-JSON/ASL state machine | Managed, no servers; less expressive (it's a declarative state machine, not arbitrary code) but zero ops. "Express" (cheap, short) vs "Standard" (durable, long, full history). |
| **Azure Durable Functions** | Code-as-workflow on Functions | Same replay model, serverless. |
| **Netflix Conductor / Camunda / Zeebe** | DAG/BPMN-driven | Conductor = JSON DAG; Camunda/Zeebe = BPMN, popular in enterprise. |

The tradeoff across them: **expressiveness (arbitrary code, Temporal) vs operational simplicity
(declarative + managed, Step Functions).** Temporal is a stateful cluster you (or Temporal Cloud) must
run; Step Functions is just there. In an interview, name both and pick based on "do we want to run a
cluster, and do we need arbitrary branching logic."

---

## Part D — Orchestration vs choreography, revisited at scale

You met these in the saga section; here's the staff-level version. As the number of steps and services
grows, the two diverge sharply:

- **Choreography scales the *coupling* problem.** Each service subscribes to events and emits events. No
  one owns the flow. To answer "what is the full lifecycle of an order?" you must read the subscription
  graph of N services. Adding a step means quietly wiring a new subscriber, and the ordering/causality
  is implicit. This is the **distributed monolith**: services that deploy independently but are
  semantically locked together by an undocumented event protocol, with no place to see the whole flow.
- **Orchestration scales the *observability* problem away** at the cost of a central component. The
  orchestrator (especially a durable one) *is* the documented flow and the live state. Debugging is
  "open the workflow instance, see exactly where it is and its full history."

> **The rule I'd state:** "Choreography for a couple of loosely-related reactions where decoupling is the
> whole point (e.g. 'on UserSignedup, also warm a cache, also send a welcome email' — independent fire-
> and-forget consumers). Orchestration the moment there's a *business process* with ordering,
> compensation, branching, or a human asking 'where is it stuck.' At scale, observability wins, and
> orchestration is far more observable."

The mistake to avoid: choosing choreography because it *sounds* more "microservices" and "decoupled,"
then discovering you've built a system no one can trace.

---

## Part E — Event Sourcing, in depth

Now a different (and frequently confused-with-the-above) idea. **Event sourcing changes how you store
state.** Instead of storing *current state* and mutating it in place, you store an **append-only log of
the events that happened**, and derive current state by replaying them.

```
Traditional (state-oriented):           Event-sourced (event log):
  account #7: { balance: 80 }             #7: [ Opened,
  (UPDATE overwrites; history lost)              Deposited 100,
                                                 Withdrew 20 ]
                                          balance = fold(events) = 80
```

The events are **facts that already happened**, named in the past tense (`MoneyDeposited`,
`OrderShipped`, `ItemAddedToCart`) — immutable, never updated or deleted. Current state is a *derived*,
disposable view.

### How you actually run it

- **Append** new events to the log (your write path). The log is the source of truth.
- **Rebuild state** by folding events from the beginning: `state = events.reduce(apply, initial)`.
- **Snapshots for performance.** Replaying 10 million events on every read is absurd, so you periodically
  store a snapshot of derived state at event #N, and on load you start from the snapshot and replay only
  events after N. Snapshots are an optimization, never the source of truth — you can always rebuild from
  the raw log.
- **Concurrency / consistency on write** is typically optimistic: "append this event only if the stream
  is still at version V" (a compare-and-set on stream version), giving you per-aggregate serializability.

### The benefits (why anyone does this)

| Benefit | What you get |
|---|---|
| **Perfect audit log** | The log *is* the complete, immutable history. Huge for finance, compliance, anything regulated. "Who changed what when" is the data model, not an afterthought. |
| **Time travel / temporal queries** | Reconstruct state *as of any past moment* by replaying up to that point. "What did this account look like last March?" |
| **Debugging** | Reproduce a bug by replaying the exact event sequence that caused it. |
| **Rebuild read models freely** | Got a new way you want to view the data? Build a new projection by replaying the log (see CQRS). The events were never thrown away. |
| **Captures *intent*** | `ItemRemovedFromCart` then `ItemAddedBack` tells you something a final `quantity: 1` never could. |

### The costs (why it's a trap when misapplied)

| Cost | The pain |
|---|---|
| **Event schema evolution** | Events are immutable and live *forever*. Your code must replay a `MoneyDeposited` event written 4 years ago in v1 format. Schema migration is now a permanent versioning discipline (Part G). |
| **Replay cost** | Rebuilding state from millions of events is slow; you're forced into snapshots and the bookkeeping they bring. |
| **Eventual consistency** | Read models lag the event log. Read-your-own-writes needs care. |
| **Query difficulty** | You can't `SELECT * WHERE balance > 100` against an event log. You *must* build projections (which is why ES and CQRS travel together). |
| **Mental model cost** | Every engineer on the team must think in events and folds, not rows. Onboarding cost is real and forever. |
| **No easy delete** | "Delete this user's data" (GDPR) vs an immutable, append-only log is genuinely hard — you reach for crypto-shredding (delete the key, leave encrypted events unreadable). |

> **Say this in the room:** "Event sourcing is the right call when the **history *is* the product** —
> ledgers, audit trails, anything where 'how we got here' has business or regulatory value, or where I'll
> genuinely want to build new read models from the past. It is a **trap** when someone reaches for it
> because it sounds advanced: for a CRUD app where you only ever care about current state, it multiplies
> complexity for nothing. Default to storing current state; adopt event sourcing for the *specific
> aggregates* (the ledger, the order lifecycle) that earn it — not the whole system."

---

## Part F — CQRS, in depth

**CQRS = Command Query Responsibility Segregation.** Split the model that handles **writes** (commands
that change state) from the model(s) that handle **reads** (queries). They can have different schemas,
different stores, even different databases.

```
                 Command side (write)                Query side (read)
 Command ──> validate ──> write model ──> events ──> projection(s) ──> read DB(s) ──> Query
            (normalized, enforces invariants)        (denormalized, shaped per query)
```

- The **write model** is optimized for *correctness*: enforce invariants, accept commands, produce the
  authoritative state change (and, if event-sourced, append events).
- The **read models / projections** are optimized for *queries*: denormalized, possibly many of them,
  each shaped for a specific screen or query. A projection is a **materialized view** kept up to date by
  consuming the write side's change stream (events, or CDC from the write DB).

This is the natural partner of event sourcing — the event log feeds the projections — **but CQRS does
not require event sourcing.** You can do CQRS over a plain SQL write DB whose changes are streamed
(via CDC) into denormalized read stores. And you can event-source without CQRS. Conflating the two is a
common confusion; keep them separate in your head.

### The eventual-consistency gap (the headline tradeoff)

The instant you split read from write and update reads *asynchronously* from the write stream, there's a
**lag**: a command succeeds, but a query a moment later may not reflect it. This is the single most
important thing to acknowledge.

> **Mitigations to name:** (1) return the new value from the command response so the UI doesn't re-read;
> (2) read-your-own-writes by reading from the write model for that one user briefly; (3) version/ETag so
> the client can poll until the projection catches up; (4) just accept it where staleness is fine. The
> wrong move is pretending the gap isn't there.

### When CQRS pays off vs when it's over-engineering

| CQRS pays off | CQRS is over-engineering |
|---|---|
| **Asymmetric load** — reads ≫ writes (or vice versa), and you want to scale them independently | Read and write loads are similar and modest |
| **Many read shapes** — the same data needs to be served in several denormalized forms (dashboard, search, mobile, exports) | One simple read shape that maps cleanly to your write schema |
| **Complex write invariants** that you don't want polluted by read concerns | Simple CRUD; a single model serves both fine |
| **Already event-sourced** — projections are the natural read side | Forcing it onto a small app "to be ready for scale" |

> **Say this in the room:** "CQRS is justified by **asymmetry** — when reads and writes have genuinely
> different shapes, loads, or scaling needs. The cost is the eventual-consistency gap and the operational
> burden of keeping projections in sync and rebuildable. For a typical CRUD service, one model is
> simpler and I wouldn't split. I'll reach for CQRS on the specific high-read, multi-shape part of the
> system — e.g. a search/feed read model fed off the write stream — not as a blanket architecture."

---

## Part G — Combining ES + CQRS, and rebuilding projections

The classic pairing: **event-sourced write side → events → projections that are the read side.**

- Command comes in → write model validates → appends events to the log.
- Each event is published to projection consumers.
- Each projection (a "read model") applies the event to its denormalized store.
- Queries hit the read stores; they never touch the event log directly.

**The superpower this unlocks: rebuilding read models.** Because the event log is the durable source of
truth and projections are *disposable derived views*, you can:

- **Add a new read model** anytime by replaying the log from the start into a fresh projection.
- **Fix a buggy projection** by truncating its store and replaying — no data loss, the events were never
  destroyed.
- **Change a projection's schema** by building the new one in parallel from the log, then cutting over.

The mechanics: each projection tracks the **offset/position** in the event stream it has consumed up to
(its checkpoint). A rebuild resets that to zero (into a new store), replays to the head, then goes live.
This is exactly the materialized-view-from-a-log pattern (ties to Topic 07 messaging and to CDC).

---

## Part H — Idempotency, exactly-once, and ordering (ties to Topics 07 & 10)

These systems live on top of message logs and retrying engines, so **at-least-once delivery is the
reality** and "exactly-once" is something you *engineer*, not something the wire gives you.

### Idempotency (the non-negotiable)

Every activity and every event consumer can run **more than once** — the engine retries, the queue
redelivers, a crash replays. So side effects must be idempotent:

- **Idempotency keys.** Carry a unique key with the operation (e.g. the workflow run id + step). The
  payment service stores "I already processed key K → here's the result" and returns the prior result on
  retry instead of charging twice. (See Topic 10.)
- **Dedup in projections.** Track the last-applied event offset per projection; ignore any event at or
  below it. Or make the apply itself idempotent (upserts keyed by event id).
- **Natural idempotency.** Prefer operations that are inherently safe to repeat ("set status = SHIPPED"
  beats "increment shipped_count").

### "Exactly-once" — what it really means

There is no exactly-once *delivery*. What you can get is **exactly-once *effects*** = at-least-once
delivery **+** idempotent processing. Workflow engines lean into this: activities are at-least-once and
retried; *you* make them idempotent so re-execution is harmless. Say it precisely in the room:
**"exactly-once delivery is a myth; I get exactly-once effects via at-least-once + idempotency keys."**

### Ordering

- Message logs guarantee order **only within a partition** (Topic 07). To get per-entity ordering, **key
  by entity** (all events for `order:42` to the same partition) so they're processed in order.
- In event sourcing, **per-aggregate ordering is enforced by the stream version** (optimistic
  concurrency: append at version V or reject). Global ordering across all aggregates is usually neither
  available nor needed.
- Projections that must handle out-of-order or duplicate events should be **commutative/idempotent** or
  buffer-and-reorder by sequence number.

---

## Part I — Versioning events and migrating projections

This is the tax of event sourcing and the place teams get burned, so have an answer ready.

**Events are immutable and immortal**, so old-format events must remain replayable forever. Strategies:

| Strategy | How | When |
|---|---|---|
| **Weak / additive schema** | Only add optional fields; never remove or repurpose. Old events still parse (missing field → default). | The default discipline; covers most changes. |
| **Upcasting** | On read, transform old event versions to the latest shape in an "upcaster" before the code sees them. Keep `EventV1 → V2 → V3` transforms. | Structural changes (renames, splits). |
| **Versioned event types** | `OrderPlacedV2` as a new type; handlers cover both. | Breaking semantic changes. |
| **Copy-and-transform / rewrite stream** | Build a *new* event stream by transforming the old one (you don't mutate the original log). | Big migrations; rare, heavy. |

**Migrating projections** is comparatively easy *because* of the log: you rarely "migrate" a projection
in place — you **rebuild** it. Stand up the new projection schema, replay the (possibly upcasted) event
log into it, verify, cut reads over, retire the old one. This is the payoff that makes the event-
versioning tax bearable.

**For workflow engines, the analogous hazard is workflow code versioning** (the determinism constraint
from Part C): if you change workflow code while instances are mid-flight, replay of their history can
diverge → non-determinism error. Handle with the engine's **versioning API** (branch on a recorded
version marker: "instances started before vN take the old path") or by **draining** old instances before
deploying breaking changes. Mention this; it shows you've operated one of these, not just read about it.

---

## Part J — Worked mini-walkthrough: design an order-fulfillment workflow

**Prompt:** "Design the backend that fulfills an order across inventory, payment, shipping, and
notifications, surviving crashes and partial failures."

**1. Requirements.** Functional: place order → reserve stock → charge → ship → notify; support
cancellation and refunds; a 7-day return window. Non-functional: must survive process/host crashes mid-
flight; no double charges; full audit of each order's lifecycle; ops can see where any order is stuck;
process can stay open for days.

**2. Map to patterns out loud.** "Cross-service consistency with no distributed transaction → **saga**
with compensations. It must survive crashes and resume, has timers (return window), branching, and needs
to be debuggable → I'll **orchestrate it with a durable execution engine (Temporal)** rather than hand-
roll a status-column state machine."

**3. The workflow (orchestration).** Workflow code (deterministic), each step an **activity** (retried,
idempotent):

```
OrderWorkflow(order):
  try:
    reservation = ReserveInventory(order)      # activity, idempotent via order id
    payment     = ChargePayment(order)         # activity, idempotency key = order id
    shipment    = CreateShipment(order)        # activity
    Notify(order, "confirmed")                 # activity
    awaitSignal("delivered", timeout=14d)      # signal or timer
    sleep(7d)                                  # durable timer: return window
    Notify(order, "return window closed")
  catch StepFailed:
    # compensation in reverse, each idempotent:
    if shipment:    CancelShipment(shipment)
    if payment:     RefundPayment(payment)
    if reservation: ReleaseInventory(reservation)
    Notify(order, "cancelled")
```

**4. Why this satisfies the reqs.**
- *Crash survival*: the engine's event history + replay resumes exactly where it left off — no re-charge.
- *No double charge*: `ChargePayment` carries an idempotency key (order id); payment service dedupes.
- *Compensation*: plain try/catch with reverse-order undo, each activity idempotent.
- *Timers*: `sleep(7d)` and `awaitSignal(..., timeout=14d)` are durable, hold no resources.
- *Human/external events*: delivery confirmation arrives as a **signal** into the running workflow.
- *Observability*: per-order workflow instance with full history in the engine's UI — "where is #12345
  stuck" is one click.

**5. Where event sourcing / CQRS *might* enter — and where I'd stop.** "The workflow engine already
gives me a durable, auditable history of the *process*, so I would **not** event-source the whole system
by default. If the **payments ledger** needs a regulatory audit trail and balance-at-any-time, I'd
event-source *that aggregate* specifically, with snapshots. If the read side is heavy and multi-shape —
an order-status dashboard, a customer-facing tracker, an analytics export — I'd add **CQRS projections**
fed from the order events/CDC, and accept the eventual-consistency lag (mitigated by returning the new
status in the command response). I'd be explicit that this is *scoped* to the parts that earn it, not a
blanket re-architecture."

**6. Failure modes to volunteer.** Engine cluster is now a critical dependency (run it HA / use managed).
Activities to non-idempotent legacy APIs need an idempotency shim. Workflow code changes vs in-flight
instances need the versioning API. Projection lag needs a read-your-writes story.

---

## Part K — Decision guide (and the over-application warning)

Climb only as far as the problem forces you. Each rung adds power *and* cost.

| Use… | When | Cost / watch out |
|---|---|---|
| **Plain DB + status column** | Short, single-service, few states, low failure complexity | Rots if process grows long/multi-service; manual resume |
| **State machine (explicit)** | Single-service lifecycle with clear states/transitions, modest failure handling | Still single-process durability; you write retries/timers |
| **Saga (choreography)** | 2–3 loosely-coupled cross-service steps where decoupling is the point | Distributed monolith, poor observability past a few steps |
| **Saga (orchestration), hand-rolled** | Multi-step cross-service with compensation, you control coordination | The orchestrator itself must be made crash-durable → this is the thing that rots |
| **Durable execution engine** | Long-running, crash-resilient, multi-step, timers/signals, branching, must be debuggable | Operate a stateful cluster (or pay for managed); determinism + workflow-versioning discipline |
| **Event sourcing** | History/audit IS the value (ledgers, compliance); need time-travel or to rebuild read models from the past | Schema-evolution-forever, replay cost, eventual consistency, team mental cost |
| **CQRS** | Asymmetric read/write load, or many denormalized read shapes off one write model | Eventual-consistency gap, projection sync + rebuild machinery |

> **The warning to say out loud:** "Event sourcing and CQRS are the two most **over-applied** patterns
> in this space. They are powerful for ledgers, audit-heavy domains, and asymmetric read/write systems,
> and they are pure cost for ordinary CRUD. The senior move is to **scope them to the aggregates that
> earn them** and default everything else to current-state storage with one model. Likewise, reach for a
> workflow engine when the process is genuinely long-running and failure-prone — not because durable
> execution is a cool phrase. Match the rung to the problem; name the cost of the rung you pick."

Also keep the relationships straight, because interviewers probe the confusion:
- **Saga** = a *consistency* pattern (cross-service "transaction" via compensations).
- **Durable execution** = a *runtime* pattern (how to run any long process crash-safely); a great way to
  *implement* an orchestrated saga.
- **Event sourcing** = a *storage* pattern (state as an event log).
- **CQRS** = a *modeling* pattern (separate read/write models).
- They compose, but **none implies the others.** You can have a durable workflow with no event sourcing,
  CQRS with no event sourcing, event sourcing with no CQRS.

---

### Self-check before the mock (answer these from memory)
- [ ] Why do ad-hoc queues + a status column rot for long-running, multi-service processes?
- [ ] How does a durable execution engine survive a crash mid-workflow? (event history + deterministic replay)
- [ ] Why must workflow code be deterministic, and what's banned inside it?
- [ ] State the orchestration-vs-choreography tradeoff and why choreography risks a distributed monolith at scale.
- [ ] Event sourcing: how is current state derived, what are snapshots for, and name two benefits + two costs.
- [ ] What problem does CQRS solve, and what is its headline tradeoff?
- [ ] Why do ES and CQRS pair so naturally, and how do you rebuild a read model?
- [ ] "Exactly-once" — what does it actually mean here, and how do you achieve the effect?
- [ ] How do you version immutable events, and how do you migrate projections?
- [ ] Give the one-line distinction between saga, durable execution, event sourcing, and CQRS — and name a case where ES/CQRS would be over-engineering.
