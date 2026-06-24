# Design 12: Distributed Job Scheduler / Task Queue (cron + delayed/recurring jobs at scale)

> **The one sentence to carry through this whole round:** a job scheduler is really *three* problems
> wearing one coat — (1) **finding the jobs that are due right now** without scanning the world,
> (2) **firing each due job exactly once-ish** even though schedulers crash and clocks lie, and
> (3) **running the work reliably** with retries, dead-letter, and dead-worker detection. I keep
> these three layers physically separate: a **scheduler** (decides *when*), a **durable queue**
> (buffers *what's ready*), and a **worker pool** (does the work). The moment you blur them — e.g.
> let workers poll the DB directly, or let the scheduler also execute — you get duplicate fires,
> thundering herds, and no way to scale the two axes independently. Say that split out loud early; it
> frames every later decision.

I'll run the standard 7-step framework. Flagging up front that the deep dives (step 6) are: the
**due-job scan at scale**, **exactly-once-ish firing** (scheduler HA + leasing), **retries/DLQ**,
and **recurring-job catch-up + clock skew** — those are where this round is decided.

---

## Step 1 — Requirements (5 min)

I'll drive this and narrow scope deliberately.

### Functional

- **Schedule a one-off job at a future time** ("run this at 2026-07-01T09:00Z").
- **Delayed job** ("run this 30 minutes from now") — same thing as one-off, expressed as a delay.
- **Recurring / cron job** ("every day at 09:00", "every 5 minutes") — a schedule that re-arms itself
  after each fire.
- **Submit a job** with a payload, a target (which handler/queue), a priority, and a tenant/owner.
- **Cancel / pause / update** a scheduled job before it fires.
- **Execute** the job: hand it to a worker, run a handler, record success/failure.
- **Retry** failed jobs with backoff, up to a max; **dead-letter** poison jobs.
- **Read** job status / history ("did my 2am report run? when? did it fail?").

Scope I'll explicitly cut to protect time: I'll skip the *authoring UX* (cron-expression builder),
workflow/DAG dependencies between jobs ("run B after A succeeds" — that's an orchestrator like
Airflow/Temporal, a different problem), and the actual business logic inside handlers. I'll mention
DAGs at the end as an extension. I'll say this out loud so the interviewer can pull any of it back.

### Non-functional — this is where the design is decided

- **Scale**: millions of scheduled jobs registered at rest; tens of thousands of *fires per second*
  at peak. The hard part is the **due-scan**, not raw storage. → drives the time-bucketed index in
  step 4 and the scan deep dive in 6.1.
- **Timeliness**: jobs fire **near-on-time**, not real-time. A few seconds of latency past the
  scheduled time is fine for almost all jobs; I'll target **p99 firing within a few seconds** of
  due time. This is a *soft-real-time* system, and saying so kills the instinct to over-engineer a
  microsecond timing wheel. (If a tenant needs sub-100ms precision, that's a different, smaller
  in-process scheduler — I'll note it as out of scope.)
- **Delivery semantics**: **at-least-once execution** is the honest, achievable target. Exactly-once
  *delivery* over a network is impossible; we get exactly-once *effect* by making the firing path
  leased + fenced and requiring **idempotent handlers**. → ties to
  `prep/10-distributed-transactions-and-idempotency.md`. I'll state this as the central correctness
  contract.
- **Durability**: a successfully *scheduled* job must not be lost across node/AZ failure. If we
  accept a job and ack the client, it *will* eventually fire. → durable job store, replicated.
- **Availability**: the *submit* path and the *firing* path must both survive a scheduler node dying.
  No single scheduler may be a SPOF, and — critically — **two schedulers must never fire the same job
  twice** when ownership is ambiguous (a partition during failover). → leader election / sharded
  ownership + fencing, deep dive 6.2 (`prep/08-consensus-and-coordination.md`).
- **Fairness / multi-tenancy**: one tenant scheduling a million jobs at midnight must not starve or
  delay every other tenant. → per-tenant fairness + priority, deep dive 6.5
  (`prep/16-microservices-and-architecture.md`).
- **Read vs write shape**: write-ish and *time-skewed* — a huge fraction of jobs cluster on round
  numbers (top of the hour, midnight). The **thundering herd of due jobs** at 00:00 is the
  signature failure mode of this system; I'll design against it explicitly.

> **The line that signals seniority:** "I will *not* promise exactly-once execution, because it's a
> lie over a network. I promise **at-least-once delivery + at-most-once *effect* via leasing,
> fencing, and idempotent handlers**. And I'll design for the load being *bursty and clustered on
> round timestamps*, not smooth — the 00:00 herd is the real enemy, not average QPS."

---

## Step 2 — Estimation (3 min)

Numbers exist to justify decisions, not to impress.

- Say **100M jobs scheduled at rest** (mix of pending one-offs + recurring definitions). Many are
  recurring, so the *effective* fire volume is higher than the row count.
- **Fires/sec (the number that matters).** Suppose 100M jobs fire on average once/day:
  `100M / 86,400 ≈ 1,160 fires/s` average. But fires are **clustered** — if even 10% of jobs are "top
  of the hour" and a few percent are "midnight," peak instantaneous demand at 00:00 could be
  **hundreds of thousands of jobs wanting to fire in the same second**. So:
  - average firing QPS: **~1–2k/s**
  - peak (00:00 herd): **~100k+ "due" in one tick** → must be *smeared* over seconds, not fired in a
    spike. This single observation justifies (a) sharded scanning, (b) jitter/smoothing, and (c) a
    queue that absorbs the burst so workers drain at their own rate.
- **Due-scan cost.** Naïve approach = "every second, `SELECT * FROM jobs WHERE next_run_at <= now()`"
  over 100M rows. Without the right index that's a **full table scan every second** — instantly fatal.
  With a B-tree index on `next_run_at` it's a cheap range scan of *only the due rows*; cost ∝ number
  due, not table size. That index is the entire ballgame and I'll call it out in step 4.
- **Storage.** `100M jobs × ~1 KB (payload + metadata) ≈ 100 GB`, plus execution history. History is
  the thing that grows unbounded — if we keep one history row per fire at ~300 bytes and ~100M
  fires/day, that's **~30 GB/day → ~11 TB/year**. → history goes to a cheap, time-partitioned store
  with TTL/archival, *not* in the hot job table. (`prep/03-databases-deep-dive.md`,
  `prep/04-sharding-and-partitioning.md`.)
- **Workers.** If a job runs ~1s on average and peak drain is ~10k/s, we need ~10k concurrent worker
  slots; trivially horizontal — workers are stateless and scale with the queue depth.

The estimation conclusion I carry forward: *storage is modest; the binding constraints are (a) the
cost of repeatedly finding due jobs, and (b) the bursty, clock-aligned firing load.* Everything in
steps 4–6 is built around those two facts.

---

## Step 3 — API design (3 min)

Two things to nail: **idempotent submission** (a retried submit must not create two jobs) and
**explicit schedule semantics** (cron vs one-off vs delay).

```
POST /v1/jobs
  Idempotency-Key: <client UUID>                 # dedup the SUBMIT itself
  {
    type: "ONE_OFF" | "DELAYED" | "RECURRING",
    runAt:    "2026-07-01T09:00:00Z",            # ONE_OFF
    delaySec: 1800,                               # DELAYED
    cron:     "0 9 * * *", timezone: "America/New_York",   # RECURRING
    target:   "report-generator",                # which handler/queue
    payload:  { ... },                           # opaque to the scheduler
    priority: 5,                                  # 0 = highest
    maxAttempts: 5,
    idempotencyKey: "user-supplied-exec-key"      # passed to the handler on fire (see 6.3)
  }
  -> 201 { jobId, status: "SCHEDULED", nextRunAt }

DELETE /v1/jobs/{jobId}                           # cancel
PATCH  /v1/jobs/{jobId}   { paused: true | false, cron?, payload? }
GET    /v1/jobs/{jobId}   -> { status, nextRunAt, lastRunAt, attempts, history[...] }
GET    /v1/jobs?tenant=&status=&cursor=           # cursor pagination, not offset
```

Design notes I'd say out loud:

- **Two different idempotency keys, two different jobs.** The `Idempotency-Key` *header* dedups the
  **submit** (a client retry of the create call must not register the job twice). The
  `idempotencyKey` in the body is the **execution** key handed to the worker so the *handler's* side
  effects are dedup-able (deep dive 6.3). Conflating them is a common mistake.
- **Schedule type is explicit.** DELAYED is just "compute `runAt = now + delay` at submit." RECURRING
  carries the cron expression *and a timezone* — timezone is load-bearing (DST, "9am local"), and
  forgetting it is a classic bug (deep dive 6.4).
- **`payload` is opaque** to the scheduler — it just hands bytes to the handler. Keeps the scheduler
  a generic platform, not coupled to job semantics.
- Auth, quotas, and rate-limiting live at the **gateway** (`prep/09-...`, `prep/17-security-and-auth.md`)
  — including a *per-tenant submit quota* so one tenant can't register 100M jobs and blow the index.
  I won't re-explain the gateway downstream.

---

## Step 4 — Data model (5 min): the time-indexed job store is the heart of the design

> **This is the highest-leverage section.** The whole system lives or dies on *how cheaply you can
> ask "what's due now?"* Get the index right and the scheduler is a tight loop; get it wrong and you
> full-scan 100M rows every second.

### Core entities

```
jobs                          # the durable source of truth — one row per scheduled job
  job_id        PK
  tenant_id                   # for fairness + quotas (6.5)
  type                        # ONE_OFF | DELAYED | RECURRING
  cron_expr     NULL          # set for RECURRING
  timezone      NULL          # set for RECURRING (DST-aware)
  target                      # handler/queue name
  payload       (opaque blob)
  priority
  status                      # SCHEDULED | LEASED | RUNNING | SUCCEEDED | FAILED | DLQ | PAUSED | CANCELLED
  next_run_at   ┐  INDEXED    # ← THE index. When should this fire next?
  shard_id      ┘  INDEXED    # composite: (shard_id, next_run_at) — see 6.1/6.2
  attempts
  max_attempts
  lease_owner   NULL          # which scheduler/worker holds it (6.2)
  lease_expires_at NULL       # visibility timeout / heartbeat deadline (6.2, 6.6)
  fence_token                 # monotonically increasing; fencing against zombie owners (6.2)
  exec_idempotency_key        # handed to handler on fire (6.3)
  created_at, updated_at

job_history                   # append-only, time-partitioned, TTL'd — NOT in the hot table
  history_id PK
  job_id, attempt_no
  fired_at, finished_at
  outcome                     # SUCCEEDED | FAILED | TIMED_OUT
  worker_id, error
```

### The access pattern that decides everything

The scheduler's one hot query is: **"give me jobs where `next_run_at <= now()`, ordered by time,
within my shard."** That is a **range scan on a B-tree index** of `(shard_id, next_run_at)`. Cost is
proportional to the number of *due* rows returned, **not** to table size — that's what makes it scale
to 100M rows. Without that index it's a full scan every tick (step 2). So:

- Index `(shard_id, next_run_at)`. The scheduler for shard *k* runs
  `SELECT ... WHERE shard_id = k AND next_run_at <= now() AND status='SCHEDULED' ORDER BY next_run_at LIMIT N`.
- `LIMIT N` (batch) so one tick pulls a bounded chunk even during the 00:00 herd; we loop until
  caught up rather than yanking 100k rows at once.

### Store choice

Access pattern = lots of small rows, a hot range-scan by time, point updates (lease/claim a row,
flip status), and per-tenant filtering. Two viable choices, and I'd state the tradeoff:

- **A relational store (Postgres/MySQL)** if we're at the lower end. ACID makes the **atomic
  claim** (`UPDATE ... WHERE status='SCHEDULED' RETURNING`) trivially correct, the `next_run_at`
  index is a plain B-tree, and `SELECT ... FOR UPDATE SKIP LOCKED` gives us contention-free claiming
  out of the box. Single-leader, synchronously replicated to another AZ for durability.
- **A horizontally-sharded NoSQL store (Cassandra/DynamoDB/Mongo)** if fire volume outgrows one
  leader. We lose easy cross-shard transactions, but we never *need* them — a job lives entirely in
  one shard. We shard by `shard_id` (deep dive 6.1) and keep a per-shard time-ordered index.
  (`prep/04-sharding-and-partitioning.md`, `prep/03-databases-deep-dive.md`.)

I'd **start with Postgres + `SKIP LOCKED`** for correctness and simplicity, and present the sharded
NoSQL path as the scale-out evolution — exactly the "simple first, evolve under pressure" move from
the framework. **History is separate**: time-partitioned, TTL'd, archived to object storage — it
must never bloat the hot job table or its index.

> **The tradeoff sentence:** "I index on `(shard_id, next_run_at)` so 'what's due?' is a bounded range
> scan, not a table scan. The job table stays small and hot; execution history — which grows forever
> — lives in a separate partitioned, TTL'd store so it can't poison the scan."

---

## Step 5 — High-level design (10 min): happy path end-to-end

```
                          ┌──────────────────────────────────────────────┐
  Client  POST /jobs ───► │   API Gateway (authn, per-tenant quota, RL)   │
                          └───────────────────────┬──────────────────────┘
                                                  ▼
                                       ┌────────────────────┐
                                       │   Job Service      │  idempotent submit;
                                       │   (CRUD + dedup)   │  computes next_run_at
                                       └─────────┬──────────┘
                                                 ▼
                            ┌──────────────────────────────────────────┐
                            │   Job Store (durable, replicated)         │
                            │   index: (shard_id, next_run_at)          │
                            └───────────────┬──────────────────────────┘
                                            │  due-scan (range scan, per shard)
              ┌─────────────────────────────┴──────────────────────────────┐
              ▼                              ▼                              ▼
       ┌────────────┐                 ┌────────────┐                 ┌────────────┐
       │ Scheduler  │  ... owns       │ Scheduler  │  ... owns       │ Scheduler  │
       │  shard 0   │   shard 0       │  shard 1   │   shard 1       │  shard N   │
       └─────┬──────┘                 └─────┬──────┘                 └─────┬──────┘
             │  claim due job (lease+fence), enqueue                       │
             └───────────────┬────────────────────────────────────────────┘
                             ▼
                  ┌─────────────────────────────┐    (durable queue: SQS/Kafka/Redis Streams)
                  │   Ready Queue(s)            │    per-priority / per-tenant partitions (6.5)
                  │   visibility timeout        │    prep/07-messaging-and-streaming.md
                  └──────────────┬──────────────┘
                                 ▼  pull
                  ┌─────────────────────────────┐
                  │   Worker Pool (stateless)   │  run handler; heartbeat; ack/nack
                  └──────┬───────────────┬──────┘
                         │ success/fail  │ heartbeat (extend lease)
                         ▼               ▼
              update status + history;  re-arm RECURRING (compute next_run_at);
              on failure: retry w/ backoff → DLQ after maxAttempts (6.3)

   Coordination: etcd/ZooKeeper for shard ownership + leader election + fencing (6.2)
```

**Walk one job through it** (I'd narrate this):

1. Client submits with an `Idempotency-Key`. Job Service dedups the submit, computes `next_run_at`
   (now+delay, or the next cron occurrence in the job's timezone), and writes the row
   `status=SCHEDULED`. Ack the client — durability promise made.
2. The **scheduler that owns this job's shard** runs its tight loop: every ~1s, range-scan
   `(shard_id=k, next_run_at <= now())` for a bounded batch.
3. For each due job it does an **atomic claim**: flip `SCHEDULED → LEASED`, set `lease_owner`,
   `lease_expires_at`, bump `fence_token`. With Postgres this is `SELECT ... FOR UPDATE SKIP LOCKED`
   then `UPDATE`; the claim is what prevents two schedulers double-firing (6.2).
4. It **enqueues** a message onto the durable Ready Queue (job_id, fence_token, exec key) and the job
   is now the queue's responsibility. The scheduler's job is done — *it never executes work itself.*
5. A **worker** pulls from the queue (with a **visibility timeout**), runs the handler with the
   payload + exec idempotency key, **heartbeats** to extend its lease for long jobs (6.6).
6. On **success**: worker acks; status `→ SUCCEEDED`, history row appended. If RECURRING, **re-arm**:
   compute the next cron occurrence and set `status=SCHEDULED, next_run_at=<next>`. On **failure**:
   `attempts++`, schedule a **backoff retry** (just set `next_run_at = now + backoff`, back to
   SCHEDULED) or move to **DLQ** once `attempts >= max_attempts` (6.3).

The structural point I keep hammering: **scheduler decides *when* and *claims*; queue *buffers*;
worker *executes*.** Three layers, scaled independently — the scheduler scales with shard count, the
queue absorbs bursts, workers scale with queue depth.

---

## Step 6 — Deep dives (15 min, where the round is won)

I'd propose the hard parts myself: *"The interesting problems are (a) finding due jobs cheaply at
100M scale, (b) firing exactly-once-ish across scheduler failover, (c) retries/DLQ and idempotent
execution, (d) recurring catch-up and clock skew, and (e) tenant fairness. Let me go deep."*

### 6.1 Finding "due" jobs at scale — the heart of the system

The naïve loop — `SELECT * WHERE next_run_at <= now()` over the whole table every second — is a
**repeated full scan** and dies at scale (step 2). Three techniques, layered:

**(a) Indexed range scan, not a scan.** As in step 4: a B-tree on `(shard_id, next_run_at)` turns
"what's due?" into a range scan whose cost ∝ rows *returned*, not table size. This alone takes us
from O(table) per tick to O(due) per tick. This is the single most important point.

**(b) Sharded scanning — partition the time-space.** One scheduler scanning 100M rows is a
bottleneck and a SPOF. Split the job space into **N shards** (hash of `job_id`, or hash of
`tenant_id` for tenant isolation). Each shard is **owned by exactly one scheduler** (6.2), which
scans only its slice. Now scanning is horizontal: add shards → add schedulers → linear scan
throughput. The trade is ownership coordination, which 6.2 handles.

**(c) Time-bucketing for the far future.** Most of the 100M rows are due *days or months* from now —
re-scanning them every second is waste even with an index. Bucket jobs by coarse time (e.g. per-hour
or per-minute buckets keyed `bucket = floor(next_run_at / window)`). A scheduler only **pulls the
current and next bucket** into a fine-grained in-memory structure; the distant buckets sit cold in
the store untouched. This is the classic **two-level (hierarchical) timing-wheel** idea: a coarse
on-disk wheel of buckets, and a fine in-memory wheel for the *imminent* jobs.

> **Polling vs timing-wheel — name the tradeoff.** *DB polling* (range-scan every tick) is dead
> simple, survives restarts for free (state is in the DB), and is "near-on-time" to within the poll
> interval — perfect for a soft-real-time scheduler. A *timing wheel* (in-memory bucketed array,
> O(1) insert and tick) gives millisecond precision and zero DB load per tick, but its state is
> **volatile** — a crash loses the wheel, so it must be *rehydrated from the durable store on
> startup* and is really an in-memory *cache* of the imminent slice, not the source of truth. **My
> design: DB (durable truth, time-bucketed) + a small in-memory timing wheel per scheduler for the
> next bucket-window**, rehydrated on failover. Polling for durability, wheel for precision and to
> avoid hammering the DB every millisecond.

**(d) Smear the herd.** At 00:00, ~100k jobs are due in one tick. We do **not** fire them in a spike.
The scheduler pulls a bounded `LIMIT N` batch and loops; the **queue absorbs the burst** so workers
drain at their own rate; and for jobs that don't need exact alignment we add **jitter** (spread cron
fires across a window) at schedule time. The queue + jitter + batching together defang the
thundering herd. (`prep/13-resilience-and-failure-handling.md` for backpressure.)

### 6.2 Exactly-once-*ish* firing — scheduler HA without double-fires

This is the correctness core. Two failure shapes to defeat: **(i)** a scheduler dies → its shard
must still fire (no missed jobs); **(ii)** during failover, the old owner is slow-but-alive and a new
owner takes over → **both** must not fire the same job. We solve ownership and the per-job claim
separately:

**Shard ownership — leader election / sharded ownership.** Each shard has exactly one owning
scheduler, assigned via a coordination service (**etcd / ZooKeeper**, Raft underneath —
`prep/08-consensus-and-coordination.md`). The owner holds a **lease/session**; if it dies, the
session expires and the shard is **reassigned** to a healthy scheduler. This is leader election *per
shard* — better than one global leader because it spreads load and shrinks blast radius. Add jobs?
Add shards and rebalance ownership (consistent hashing so reassignment is minimal).

**The split-brain trap.** Lease expiry is based on *time*, and a paused/GC'd/partitioned old owner
might not *know* its lease expired — it wakes up and tries to fire. Now two schedulers think they own
the shard. Two defenses:

1. **Per-job atomic claim.** Even within a shard, firing a job is a compare-and-set:
   `UPDATE jobs SET status='LEASED', lease_owner=me, fence_token=fence_token+1
   WHERE job_id=? AND status='SCHEDULED'`. Only one writer wins; the loser sees 0 rows updated and
   skips. (Postgres `SKIP LOCKED` gives the same effect contention-free.) So even a brief
   double-owner can't double-claim a *row*.
2. **Fencing tokens.** The claim bumps a **monotonic `fence_token`**. The token rides with the
   message to the worker and onto any side-effect store. A **zombie** old owner carries a *stale*
   token; downstream rejects writes with a token lower than the highest seen. This is the canonical
   fix for the "lease expired but the holder didn't notice" problem — a timeout alone is **not**
   enough; you need fencing. (`prep/08-consensus-and-coordination.md`,
   `prep/10-distributed-transactions-and-idempotency.md`.)

> **Staff signal:** "Leader election alone does *not* give safety — a lease can expire on paper while
> the old owner is still running. I pair ownership with a **per-row atomic claim** *and* a **fencing
> token**, so the worst a stale owner can do is get rejected. That's how you get at-most-once
> *firing* on top of an unreliable, time-based lease."

Net: ownership gives liveness (someone always fires the shard); atomic claim + fencing give safety
(no double-fire). Combined with idempotent handlers (6.3), the end-to-end guarantee is
**at-least-once delivery, at-most-once effect**.

### 6.3 Retries, backoff, dead-letter, and idempotent execution

Once a job is on the queue, it's a **task-queue** problem — this is where I lean on the messaging doc
(`prep/07-messaging-and-streaming.md`) and resilience doc (`prep/13-resilience-and-failure-handling.md`).

- **At-least-once delivery + visibility timeout.** Worker pulls a message; the queue hides it for a
  *visibility timeout*. If the worker acks (success) → message deleted. If it crashes or the timeout
  elapses → message reappears and another worker retries. This is exactly SQS/Kafka-style at-least-once.
- **Idempotent handlers — the contract that makes at-least-once safe.** Because delivery is
  at-least-once, a handler **will** occasionally run twice (timeout-then-redeliver, or a double-fire
  that fencing didn't catch upstream). So every handler must dedup on the **execution idempotency
  key** we carried from submit: it records "I processed key X" (a unique constraint / dedup table)
  and a second run is a no-op. This is the same exactly-once-*effect* pattern as the payment system —
  `prep/10-distributed-transactions-and-idempotency.md`. **I never promise exactly-once delivery; I
  make the effect idempotent.**
- **Retries with backoff + jitter.** On failure: `attempts++`, and reschedule with
  **exponential backoff + jitter** (`next_run_at = now + base·2^attempts ± jitter`). Jitter matters —
  without it, a batch that failed together retries together and re-creates the herd
  (`prep/13-...`). Implementation is elegant: a retry is just *re-arming the job* with a future
  `next_run_at` — it flows back through the same due-scan path. No special retry machinery.
- **Dead-letter queue (poison jobs).** After `attempts >= max_attempts`, stop retrying and move the
  job to a **DLQ** (`status=DLQ`). A poison job — one that deterministically crashes the worker — must
  not retry forever burning capacity or, worse, take down workers in a loop. The DLQ is inspected by
  humans / a separate process; it's the bound on blast radius.
- **Circuit breaker on the handler target.** If a downstream that handlers call is down, fast-fail
  and back off rather than retrying into a wall — protects the worker pool from a dependency outage
  (`prep/13-resilience-and-failure-handling.md`).

> **Say this:** "Retries, DLQ, and recurring re-arming are all the *same mechanism* — set a future
> `next_run_at` and let the due-scan pick it up. The only thing that makes at-least-once safe is the
> idempotent handler keyed on the execution key. Backoff *with jitter*, or the retry storm recreates
> the herd."

### 6.4 Recurring jobs — next-run computation, catch-up, and clock skew

Recurring (cron) jobs add three subtle problems:

- **Re-arming.** After a recurring job fires, compute its **next occurrence** from the cron
  expression *in the job's timezone* and write it back as a new `SCHEDULED` row/`next_run_at`. **When**
  you re-arm matters: re-arm *on enqueue* (not on success) so the schedule keeps ticking even if a
  single run fails — otherwise one failed 9am run delays tomorrow's. (For "skip if previous still
  running" semantics, gate on the prior run's status; that's a per-job policy.)
- **Timezone & DST.** "9am daily, America/New_York" is **not** a fixed UTC offset — DST shifts it by
  an hour twice a year, and some local times *don't exist* or *occur twice* on transition days. We
  must compute occurrences against an IANA timezone database, not a stored offset. Forgetting this is
  a classic production bug (reports fire an hour off for half the year).
- **Missed-fire / catch-up policy.** If the scheduler was down (or a shard was unowned) across a job's
  due time, what happens when it recovers and sees `next_run_at` is *in the past*? This is a **policy
  decision per job**, and naming the options is the staff move:
  - **Fire-once-immediately (coalesce):** the 2am report should run *once* late, not 6 times for the
    6 missed ticks. Coalesce all missed occurrences into a single catch-up fire. (Default for most.)
  - **Fire-all (backfill):** an hourly billing aggregation may need *every* missed window run, in
    order. Replay all missed occurrences.
  - **Skip:** a "send the daily 9am digest" job missed past noon is pointless — skip to the next
    future occurrence. (Time-sensitive jobs.)
  I'd expose this as a `missedFirePolicy` field and default to *coalesce*.
- **Clock skew.** Schedulers run on different machines with drifting clocks; if one is 30s fast it
  fires early. Mitigations: rely on **NTP-synced clocks** and treat the *store's* time / a single
  time authority as canonical where precision matters; tolerate small skew because we're
  *soft*-real-time (a few seconds early/late is within spec). The fencing + atomic claim mean skew
  causes *mistiming*, never *double-firing* — which is the property that actually matters.

### 6.5 Multi-tenancy: fairness, priority, and starvation

> One tenant scheduling a million midnight jobs must not starve everyone else. This is the
> fairness deep dive — `prep/16-microservices-and-architecture.md`.

- **Priority.** Multiple queues by priority (or a priority field the workers honor). High-priority
  jobs jump ahead — but pure priority **starves** low-priority work, so cap it with **aging** (a
  long-waiting low-priority job's effective priority rises) or reserved capacity per priority class.
- **Per-tenant fairness — the real problem.** A single FIFO queue lets one noisy tenant's 1M jobs sit
  in front of everyone's. Fixes:
  - **Per-tenant queues + weighted round-robin / fair scheduling** across them, so workers pull
    *round-robin across tenants*, not strictly FIFO. Tenant A's flood drains at its fair share while
    Tenant B's trickle still gets serviced.
  - **Per-tenant concurrency limits / quotas** (a tenant may have at most K jobs running) and
    **submit quotas at the gateway** (can't register more than Q scheduled jobs) — protects the
    index and the workers.
  - **Sharding by tenant** (6.1) gives natural isolation: a hot tenant's scan load is contained to
    its shard(s) rather than slowing the global scan.
- **Noisy-neighbor / bulkheading.** Reserve worker capacity per tenant tier so a free-tier flood
  can't consume the pool that paid tenants depend on (bulkhead — `prep/13-...`).

### 6.6 Long-running jobs, heartbeating, and dead-worker detection

A job that runs for minutes interacts badly with a fixed visibility timeout: if the timeout is
shorter than the job, the queue thinks the worker died and **redelivers while it's still running** →
duplicate execution. Two coupled mechanisms:

- **Heartbeating / lease extension.** A long-running worker periodically extends its lease /
  visibility timeout (`lease_expires_at = now + Δ`) while it's making progress. As long as it
  heartbeats, no one else picks up the job.
- **Dead-worker detection + re-queue.** If heartbeats *stop* (worker crashed, OOM'd, network died),
  the lease expires and the job is **re-queued** for another worker. This is how we get liveness
  despite worker death — but it's exactly *why* handlers must be idempotent (6.3): a worker that died
  *after* doing the side effect but *before* acking will have its job re-run. Heartbeat detects death;
  idempotency makes the re-run safe.

> **The pairing to state:** "Heartbeats let long jobs hold their lease; lease *expiry* detects dead
> workers and re-queues. The combination guarantees a job in progress is either finishing or
> re-running — never silently lost — and idempotent handlers make the occasional re-run harmless."

---

## Step 7 — Wrap-up (3 min): failure modes, SPOFs, what's left

**Failure modes I'd name before being asked:**

- **Thundering herd at 00:00** (the signature failure) → batch + `LIMIT N` scans, the queue absorbs
  the burst, jitter spreads non-aligned cron fires, workers drain at their own rate. (6.1d)
- **Scheduler dies** → its shard's lease expires; coordination service reassigns the shard to a
  healthy scheduler, which rehydrates its in-memory wheel from the durable store and resumes scanning.
  No job lost because truth lives in the store, not the wheel. (6.2)
- **Two schedulers think they own a shard (split-brain on failover)** → per-row atomic claim means
  only one wins the row; fencing token means a zombie owner's late writes are rejected downstream.
  At-most-once *firing*. (6.2)
- **Worker dies mid-job** → heartbeat stops → lease expires → job re-queued → another worker re-runs;
  idempotent handler makes the re-run a no-op for side effects. (6.6, 6.3)
- **Poison job crashes workers repeatedly** → bounded retries with backoff → DLQ after maxAttempts,
  so it can't burn the pool forever. (6.3)
- **Queue backs up** (workers can't keep up) → backpressure; the durable queue holds the backlog;
  autoscale workers on queue depth; per-tenant fairness keeps one flood from starving others.
  (6.5, `prep/13-...`)
- **Clock skew / DST** → soft-real-time tolerance + NTP + canonical time source; timezone-aware cron
  math; skew can mistime but never double-fires thanks to atomic claim + fencing. (6.4)
- **Job store leader fails** → synchronous standby in another AZ promoted; brief write pause; jobs
  durable (RPO≈0). (`prep/05-replication-and-consistency.md`)

**SPOFs and redundancy:** the **job store** is the most critical component — multi-AZ replication,
automated failover, point-in-time recovery; history offloaded so the hot store stays small. The
**scheduler** is *not* a SPOF because of per-shard ownership + reassignment; the **coordination
service** (etcd/ZK) is itself a quorum-replicated cluster (no single node SPOF). **Queue** and
**workers** are horizontally redundant and stateless.

**What I'd do with more time:** workflow/DAG dependencies (job B after A — drifts toward
Temporal/Airflow, a step machine on top of this); priority-aware admission control; a delay-queue
optimization (Redis sorted-set / SQS delay) for short delays to skip the DB entirely; tiered storage
+ automated archival of history; cross-region scheduling with regional shard ownership; an exactly-once
*sink* helper (transactional outbox in handlers) for handlers that can't easily be made idempotent.

---

## What made this staff-level

- **Separated the system into three independently-scaled layers** — scheduler (when) / queue (buffer)
  / workers (execute) — and refused to let workers poll the store directly or the scheduler execute.
- **Made the time-index the spine**: derived that "what's due?" must be an O(due) range scan on
  `(shard_id, next_run_at)`, not an O(table) scan, and kept history out of the hot table.
- **Named the real load shape** — bursty and clock-aligned, the 00:00 herd — and designed against it
  (batching, queue absorption, jitter) instead of optimizing average QPS.
- **Was honest about delivery semantics**: refused exactly-once *delivery*, delivered at-least-once +
  at-most-once *effect* via atomic claim + **fencing tokens** + **idempotent handlers** — and
  explicitly noted that leader election/leases alone are *not* safe without fencing.
- **Unified retries, DLQ, and recurring re-arming** as one mechanism (set a future `next_run_at`),
  with backoff+jitter to avoid retry storms.
- **Handled the operator-grade subtleties**: DST/timezone cron math, the missed-fire *policy* menu
  (coalesce / backfill / skip), clock skew, heartbeating + dead-worker re-queue for long jobs.
- **Addressed multi-tenant fairness head-on** — per-tenant queues, weighted fair scheduling, quotas,
  bulkheads — so one tenant can't starve the platform.
- Cross-referenced the building blocks rather than re-deriving them: messaging/queues (07), consensus
  & coordination/leader election/fencing (08), idempotency/exactly-once-effect (10), resilience/
  backoff/DLQ/circuit-breaker/bulkhead (13), sharding (04), databases (03), replication (05),
  gateway/security (09, 17), multi-tenancy (16).

### Self-check (answer from memory before the mock)

- [ ] Why is "what's due?" an O(due) range scan and not an O(table) scan — what index makes it so?
- [ ] Polling vs timing wheel: what does each buy, and why do I use *both*?
- [ ] What exactly prevents two schedulers from firing the same job — and why isn't a lease enough?
- [ ] What is a fencing token and which failure does it defeat that a timeout can't?
- [ ] What delivery guarantee do I promise, and what makes the at-least-once retries safe?
- [ ] Name the three missed-fire policies and when you'd pick each.
- [ ] How do heartbeating and lease expiry combine to handle long jobs and dead workers?
- [ ] How do I stop one tenant's million midnight jobs from starving everyone else?
- [ ] What is the signature failure mode of this system, and what three things defang it?
