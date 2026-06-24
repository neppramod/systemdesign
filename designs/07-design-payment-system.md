# Design 07: Payment System / Digital Wallet (Stripe / PayPal-style)

> **Why this problem is different from every other one in this prep:** in a feed, a search index, a
> chat app, the worst case is *staleness* — a user sees something slightly old and nobody cares.
> Here the worst case is **a wrong number in someone's balance**, and that is a regulatory incident,
> a customer-trust event, and possibly fraud. This is a **correctness-dominated** problem, not a
> scale-dominated one. The whole interview is won by saying, early and often: "money cannot be lost,
> created, or double-charged — so I am building this around an immutable double-entry ledger,
> idempotency on every write, and reconciliation against the outside world. I will *not* reach for
> eventual consistency anywhere it touches a balance." If you remember one sentence from this doc,
> it's that one.

I'll run the standard 7-step framework, but I'm flagging up front that the deep dives (step 6) are
the ledger, idempotency, the PSP integration, and the saga — those are where this round is decided.

---

## Step 1 — Requirements (5 min)

I'll drive this and narrow scope deliberately.

### Functional

- **Create a charge / payment**: move money from a payer (customer card) to a payee (merchant) via
  an external card network. (Stripe's core verb.)
- **Wallet operations**: top up a balance, hold a balance, transfer balance between two internal
  accounts (PayPal's core verb), pay out / withdraw to a bank.
- **Refunds** (full or partial) and **chargebacks** (bank-initiated reversal).
- **Read**: balance, transaction history, charge status.
- **Idempotent retries**: the client can safely resend any money-moving request.

Scope I'll explicitly cut to protect time: I'll skip KYC/onboarding, the card-vaulting/PCI tokenization
flow (I'll assume a tokenized card reference exists), subscriptions/recurring billing, and the FX
trading desk. I'll cover multi-currency, fees, and a fraud hook at a high level. I'll say so out loud
so the interviewer can pull any of them back in.

### Non-functional — this is the whole game

- **Correctness / consistency**: **strong consistency for anything touching a balance.** No lost
  updates, no double-spend, no money created. Debits must equal credits *always*. This is the
  non-negotiable. → see `prep/05-replication-and-consistency.md` for why I'll pick a strongly
  consistent (single-leader, synchronously replicated) store, not a quorum/eventual one.
- **Exactly-once *effect*** on the books. We can't get exactly-once *delivery* over a network
  (impossible), so we get exactly-once *effect* via **client-supplied idempotency keys + dedup**. →
  `prep/10-distributed-transactions-and-idempotency.md`.
- **Durability**: a committed payment must survive disk/node/AZ loss. We accept *higher latency* to
  get *zero data loss* (RPO = 0). Synchronous replication, fsync, multi-AZ.
- **Auditability / compliance**: every state change is an immutable, append-only record. We must be
  able to answer "why is this balance this number?" months later. SOX/PCI/audit trails.
- **Availability**: high, but **CP over AP** for the write path. If we're partitioned from the
  consensus quorum, we *refuse the charge* rather than risk a double-charge. A failed payment the
  user can retry is recoverable; a double-charge is a refund + a complaint + possibly fraud.
- **Latency**: charge p99 under ~1s is fine — users tolerate a spinner at checkout. We are not
  latency-bound; we are correctness-bound. I'll trade latency for durability without hesitation.

> **The line that signals seniority:** "This is the one design where I deliberately choose
> consistency and durability over latency and availability. PACELC: under partition I pick C; even
> with no partition I pick C over L. The cost of being wrong is asymmetric — a declined payment is
> an annoyance, a double-charge is an incident."

---

## Step 2 — Estimation (3 min)

Numbers exist to justify decisions, not to impress. Let's size a mid-large processor.

- Say **10M transactions/day**. Average QPS = `10M / 86,400 ≈ 115 txn/s`. Peak (Black Friday, lunch
  rush) at 3–5× → **~500 txn/s peak**. That is *small*. A payment system is **not** a high-QPS
  problem — it's a high-*correctness* problem. Worth saying out loud, because it kills the instinct
  to over-shard.
- Each logical payment fans out into **multiple ledger entries** (double-entry: at least a debit and
  a credit, often 4–6 lines once you add fees, FX, and the payee leg). So ledger write rate ≈
  `500 × ~5 ≈ 2,500 ledger rows/s` at peak. Still modest.
- **Storage / ledger growth** — this is the number that matters, because the ledger is *append-only
  and we never delete*:
  - ~`10M txn/day × 5 entries × ~300 bytes ≈ 15 GB/day` of ledger rows.
  - → **~5.5 TB/year**, ~50+ TB over a 7-year retention window (regulatory retention is typically
    7–10 years for financial records). Indexes roughly double that.
  - So: a single Postgres node holds a year easily; over the full retention horizon we **archive
    cold partitions to cheap object storage** and keep hot data online. We shard by account only if
    write volume forces it — at 2,500 rows/s it does not, so I'll *justify staying single-leader* and
    not prematurely shard. (Sharding a ledger introduces cross-shard transfers = a distributed
    transaction; avoid it until forced — `prep/04-sharding-and-partitioning.md`.)
- **Durability requirement restated as a number**: **RPO = 0** (lose zero committed transactions),
  RTO measured in seconds-to-minutes via failover. That requirement is what forces synchronous
  replication in step 4.

The estimation conclusion I carry forward: *write volume is trivially small; the hard requirements
are durability and correctness, not throughput.* That's the opposite of a feed/Twitter problem and
I want the interviewer to hear me notice it.

---

## Step 3 — API design (3 min)

Two things to nail here: **idempotency keys are first-class** and **state is explicit** (no
fire-and-forget on money).

```
POST /v1/charges
  Idempotency-Key: <client-generated UUID>     # REQUIRED on every money-moving write
  { amountMinor: 5000, currency: "USD",
    source: "tok_card_x", destination: "acct_merchant_y",
    metadata: {...} }
  -> 200 { chargeId, status: "pending"|"succeeded"|"failed", ... }

POST /v1/refunds
  Idempotency-Key: <uuid>
  { chargeId, amountMinor?: 5000 }              # partial if amount < original
  -> { refundId, status }

POST /v1/wallets/{id}/transfers
  Idempotency-Key: <uuid>
  { toWalletId, amountMinor, currency }
  -> { transferId, status }

GET  /v1/charges/{chargeId}      -> { status, ledgerEntries[...], ... }
GET  /v1/wallets/{id}/balance    -> { available, pending, currency }
GET  /v1/wallets/{id}/ledger?cursor=  -> entries[], nextCursor   # cursor, not offset
```

Design notes I'd say out loud:

- **Amounts are integer minor units** (cents), never floats. Floating-point money is a
  correctness bug. State the currency explicitly per amount; never assume.
- **`Idempotency-Key` is mandatory and client-generated.** The client mints a UUID *before* the
  first attempt and reuses it across every retry of *that logical operation*. This is the single
  most important contract in the whole API. More in the deep dive.
- **Status is explicit and includes `pending`.** A charge is not synchronously "done" — it goes
  through the card network asynchronously. Modeling `pending` in the API (not hiding it) is what
  lets the client and us handle the "did it go through?" case honestly.
- Auth (mTLS / API keys / OAuth) and rate limiting live at the gateway — `prep/09-...` and
  `prep/17-security-and-auth.md`. I won't re-explain them downstream.

---

## Step 4 — Data model (5 min): the double-entry ledger is the heart of the design

> **This section is the single highest-leverage thing in the interview.** If you get the ledger
> right, the rest of the design is mostly plumbing. If you draw "an accounts table with a `balance`
> column that we UPDATE," you have already failed — that's a mutable shared cell with lost-update and
> no audit trail.

### The core idea: immutable, append-only, double-entry

Borrowed from 500 years of accounting, because accounting *already solved* "track money without
losing or creating it." Every money movement is recorded as a set of **ledger entries** where the
**sum of debits equals the sum of credits**. Money is never destroyed or created — it only *moves
between accounts*. If the entries don't balance to zero, the transaction is rejected. That invariant
(`Σdebits = Σcredits = 0`) is your built-in tripwire against bugs.

```
accounts
  account_id PK        # one row per "bucket of money": each user wallet, each merchant,
  type                 # plus *internal* system accounts: cash, fees_revenue, FX, payable...
  currency             # an account holds ONE currency
  -- NO balance column here. Balance is derived. (see below)

ledger_entries   (APPEND-ONLY. never UPDATE, never DELETE)
  entry_id PK
  transaction_id       # groups the entries that must balance together
  account_id FK
  direction            # DEBIT | CREDIT
  amount_minor         # positive integer, in the account's currency
  created_at
  -- immutable once written

transactions
  transaction_id PK
  idempotency_key  UNIQUE      # ← dedup happens here (deep dive)
  type                          # charge | refund | transfer | payout | chargeback ...
  status                        # pending | posted | failed | reversed
  created_at
```

A simple internal transfer of $50 from Alice to Bob is two entries under one `transaction_id`:

```
txn T1:  DEBIT  Alice  5000        # Alice's money goes down
         CREDIT Bob    5000        # Bob's money goes up
         ----------------------
         sum = 0  ✓ balances
```

A card charge with a fee is more entries, but still balances:

```
txn T2 (charge $50, $1.50 fee):
  DEBIT  customer_cash_clearing   5000     # money arriving from the network
  CREDIT merchant_payable         4850     # what we owe the merchant
  CREDIT fees_revenue              150      # our cut
  ---------------------------------------
  sum = 0  ✓
```

### Why immutable / append-only matters (say all three)

1. **Auditability**: the ledger *is* the audit log. You never ask "what was the balance and who
   changed it?" — you replay the entries. Compliance and disputes both need this.
2. **Correctness**: you cannot have a lost update on a row you never UPDATE. Concurrency reduces to
   "append a new immutable row," which is far easier to make safe than read-modify-write on a
   shared `balance` cell.
3. **Corrections are also entries.** You never edit or delete a wrong entry — you post a
   *compensating* reversing entry. The mistake stays visible in the history. (This is the same
   philosophy as a saga compensation in `prep/10-...` — you don't roll back the past, you move
   forward with a correcting action.)

### Balance is *derived*, not stored

`balance(account) = Σ credits − Σ debits` over its entries. Two ways to serve it, and I'd say which
and why:

- **Materialized balance / running total**: keep a `balances` table (or a cached current balance per
  account) that is updated **inside the same transaction that appends the ledger entries** — never
  in a separate write. That makes it a *derived cache that can always be rebuilt by replaying the
  ledger*, not a source of truth. Reads hit it directly (fast). The ledger remains the truth.
- **On-demand sum** for cold/rare accounts. For hot accounts, you'd keep periodic **balance
  snapshots** (e.g., end-of-day) so a balance query sums only entries *since the last snapshot*
  instead of all history — bounding the query cost as the ledger grows forever.

> **The tradeoff sentence:** "I store a materialized balance for read speed, but it's written in the
> *same local transaction* as the ledger entries so they can never diverge, and it's reconstructible
> from the ledger at any time. The ledger is the source of truth; the balance is a derived view."

### Store choice: strongly consistent SQL

Access pattern: small write volume, multi-row atomic transactions (all entries of a txn commit or
none do), a hard `Σ=0` invariant, point + range reads by account, and audit queries. That is a
**textbook relational/ACID workload**, not a NoSQL one. I'll pick **Postgres (or Spanner/CockroachDB
if I outgrow one leader)**: single-leader, **synchronous replication to ≥1 replica in another AZ**
(RPO=0), `SERIALIZABLE` or carefully-chosen isolation for the balance-affecting writes. This is the
deliberate "strong consistency, CP" choice from step 1 — `prep/05-replication-and-consistency.md`,
`prep/03-databases-deep-dive.md`.

---

## Step 5 — High-level design (10 min): happy path end-to-end

```
                        ┌─────────────────────────────────────────────┐
 Client (merchant SDK)  │                                             │
   │  POST /charges      │   API Gateway (authn, rate-limit, TLS)      │
   │  +Idempotency-Key   └──────────────────┬──────────────────────────┘
   ▼                                        ▼
                            ┌───────────────────────────────┐
                            │     Payment / Charge Service   │
                            │  (orchestrates the saga)       │
                            └───┬───────────┬───────────┬────┘
       1. dedup on key ────────┘           │           └──── 3. emit event
       2. write ledger (ACID)              │ 2b. call PSP        (outbox)
                            ┌──────────────▼─────┐   ┌──────────▼─────────┐
                            │  Ledger DB (SQL,    │   │  PSP Adapter       │
                            │  single-leader,     │   │  (Stripe/Adyen/    │
                            │  sync-replicated)   │   │   card networks)   │
                            │  + idempotency tbl  │   └──────────┬─────────┘
                            └─────────────────────┘              │ async
                                                                 ▼ webhook
                            ┌─────────────────────────────────────────────┐
                            │  Webhook Ingest  ──►  updates txn status     │
                            └─────────────────────────────────────────────┘

   Outbox ─► Kafka ─► [ fraud stream | notifications | analytics | data lake ]
   Reconciliation job: nightly diff(internal ledger, PSP settlement file)
```

**Walk one charge through it** (I'd narrate this):

1. Client mints an idempotency key, calls `POST /charges` through the gateway (authn + rate limit).
2. Payment service does an **idempotency check**: is there already a transaction row for this key?
   If yes → return the stored prior result, do nothing else. If no → proceed.
3. It **reserves/authorizes** with the PSP (auth hold on the card) — the charge is now `pending`.
4. On a successful authorization+capture path, it **appends balanced ledger entries** and updates
   the materialized balance **in one ACID transaction**, and writes an **outbox row** in that *same*
   transaction (so the "payment happened" event can't be lost — no dual write;
   `prep/10-...`).
5. A relay publishes the outbox event to Kafka → fraud scoring, notifications, analytics consume it
   asynchronously (`prep/07-messaging-and-streaming.md`, `prep/18-batch-and-stream-processing.md`).
6. The PSP confirms final settlement later via **webhook**; we move the txn `pending → posted`.

The key structural point: **the ledger write and the outbox write are local and atomic; everything
crossing a service or network boundary (PSP, Kafka) is made safe with idempotency + reconciliation,
never assumed to succeed.**

---

## Step 6 — Deep dives (15 min, where the round is won)

I'd propose the four hard parts myself: *"The interesting problems here are (a) idempotency end to
end, (b) the PSP being an unreliable async oracle, (c) orchestrating the multi-step payment as a
saga, and (d) reconciliation. Let me go deep on those."*

### 6.1 Idempotency end-to-end — why *every* money operation must be idempotent

The network gives us **at-least-once** at best. The client's `POST /charges` can time out *after* we
charged the card but *before* the response reached the client. The client will retry. Without
idempotency, that retry is a **double-charge**. So:

- The client mints **one idempotency key per logical operation** and reuses it on every retry. This
  is a contract, not an internal detail — it must be client-supplied because only the client knows
  "this is the same purchase, not a new one."
- Server side, the `transactions.idempotency_key` column has a **UNIQUE constraint**. The flow:
  1. Begin transaction. `INSERT` the transaction row with the key. If the unique constraint **fires**
     (key already exists) → this is a retry → **return the previously stored response**, do not
     re-charge. The DB's unique index *is* the dedup mechanism — no separate lock needed.
  2. If insert succeeds → it's the first time → do the work, store the result against the key,
     commit.
- **Concurrent** duplicates (two retries racing) are resolved by the same unique constraint: one
  wins the insert, the other gets the conflict and waits/returns the result. This is exactly the
  idempotency-key pattern from `prep/10-distributed-transactions-and-idempotency.md` — exactly-once
  *effect*, built on at-least-once delivery + dedup.
- **Idempotency is recursive**: it isn't only the public API. The Kafka consumers are at-least-once
  too, so the fraud scorer, notifier, and any internal step must also dedup (by event id /
  transaction id). Every money or money-adjacent operation, at every hop, is idempotent. That's the
  discipline.

> **Say this:** "I never make a money call I can't safely retry. Idempotency keys turn 'I don't know
> if it happened' from a disaster into a no-op retry."

### 6.2 The PSP is an unreliable async oracle — the "did the charge go through?" problem

External Payment Service Providers (Stripe/Adyen) and card networks are the hardest part because we
*don't control them* and they are **asynchronous and can fail-after-timeout**:

- A charge isn't instant. It moves `authorized → captured → settled`, and settlement can take **days**
  (T+1, T+2). So our txn status is genuinely `pending` for a while — we model that honestly rather
  than blocking.
- **The killer case**: we call the PSP to charge a card and the call **times out**. Did the charge
  go through? *We do not know.* The money may have moved on their side while we got no response. We
  **must not** blindly retry (double-charge) and **must not** blindly assume failure (lost money).
- Three mechanisms handle this:
  1. **Idempotency keys on the PSP call too.** Stripe et al. accept an idempotency key. We pass our
     own key, so a retry after a timeout is *safe at the PSP* — it returns the original result rather
     than charging twice. This is why idempotency must extend across the external boundary, not just
     our API.
  2. **Webhooks for the truth.** The PSP calls *us back* (webhook) with the authoritative final
     state (`succeeded` / `failed` / `disputed`). We treat the webhook, not our outbound call's
     return value, as the source of truth for final state. Webhooks are at-least-once and can arrive
     out of order → the webhook handler is **idempotent** (dedup by PSP event id) and
     **state-machine-guarded** (never move `posted → pending`).
  3. **Reconciliation as the backstop** (6.4) for anything that falls through the cracks — a webhook
     we never received, a timeout we never resolved.
- Until we have a definitive outcome, the txn stays `pending` and a **poller / reconciliation pass**
  asks the PSP "what happened to idempotency-key X?" rather than guessing.

> **Staff signal:** "I treat the PSP as an eventually-authoritative external system I can't trust
> synchronously. The contract is: idempotency key out, webhook in, reconciliation as the safety net.
> The one thing I never do is resolve a timeout by guessing."

### 6.3 The payment as a SAGA (reserve → charge → fulfill → compensate)

A real payment spans multiple services/steps that can't share one ACID transaction (us, the PSP,
maybe inventory/fulfillment). That's a **distributed transaction**, and the answer is a **saga**, not
2PC — `prep/10-distributed-transactions-and-idempotency.md`.

Why not 2PC: it's blocking (the coordinator holding locks across a slow external PSP would be a
disaster), and the card network won't participate in your two-phase protocol anyway. So we use a
**saga**: a sequence of local transactions, each with a **compensating action** if a later step
fails.

Orchestrated saga for a checkout-style payment:

```
1. RESERVE   — auth hold on the card via PSP (idempotency key)     compensate: void the auth hold
2. CHARGE    — capture funds; append balanced ledger entries (ACID) compensate: post a REFUND txn
3. FULFILL   — notify merchant / release goods                      compensate: claw back / cancel
   if any step fails → run compensations for completed steps, in reverse
```

Key points I'd make:

- Compensation **is not rollback** — step 2 already committed real ledger rows; we don't delete them,
  we **post a new reversing transaction** (a refund). The history stays intact (ties back to the
  append-only ledger principle in step 4).
- Each step and each compensation is **idempotent** (keyed), because the saga coordinator retries on
  crash and may re-run a step.
- The saga state is itself persisted (a `saga_state` row / log) so a crashed coordinator resumes
  where it left off — otherwise the coordinator is a SPOF that strands half-finished payments.
- I'd choose **orchestration over choreography** here: money flows want a single explicit state
  machine you can inspect and audit ("where is payment X?"), not emergent behavior across event
  handlers. (`prep/16-microservices-and-architecture.md`.)

### 6.4 Reconciliation — comparing the internal ledger vs the PSP's reality, and fixing drift

This is the deep dive that separates people who've *operated* a payment system from people who've
only drawn one. **Distributed systems drift; reconciliation is how you find and fix it before a
customer (or an auditor) does.**

- Every PSP sends a **settlement file / statement** (daily): "here is every transaction that
  actually settled, and for how much." Banks send statements too.
- A **reconciliation job** (a batch process — `prep/18-batch-and-stream-processing.md`) does a
  three-way diff: **our ledger ↔ PSP statement ↔ bank statement**. For each, it asks:
  - In our ledger as `posted` but **missing from the PSP file** → did we book money that never
    settled? Investigate / reverse.
  - In the PSP file but **not in our ledger** → a webhook we dropped → post the missing ledger
    entries.
  - Present in both but **amounts differ** (fees, FX rounding) → flag the discrepancy.
- Drift is *expected* (timeouts, dropped webhooks, FX rounding) — the job's purpose is **detection +
  automated correction for known cases, and a human-reviewed exceptions queue for the rest.** Fixes
  are always *new* compensating ledger entries, never edits.
- Reconciliation is the backstop that makes the rest of the system safe to build: it means a dropped
  webhook or an unresolved PSP timeout is *eventually caught and corrected*, not silently lost. It's
  the reason we can tolerate the PSP being unreliable.

> **Say this:** "Reconciliation is not optional bookkeeping — it's the closed-loop control that makes
> a system built on unreliable external calls actually correct over time. Internal ledger is truth
> for *us*; the recon job ensures it matches truth for the *outside world*."

### 6.5 Currencies, fees, refunds, chargebacks (high level)

- **Multi-currency**: each account holds **one** currency; never mix in one account. Cross-currency
  moves go through dedicated **FX clearing accounts** with the rate captured as its own ledger
  entries, so the conversion is auditable. Money still balances per-currency.
- **Fees**: just more ledger lines (`CREDIT fees_revenue`) inside the same balanced transaction, as
  shown in step 4. Modeling fees as ledger entries (not a side calculation) keeps revenue auditable.
- **Refunds**: a new transaction that *reverses* the original (full or partial) — never an edit of
  the original. Idempotent, keyed.
- **Chargebacks**: bank-initiated reversal that arrives via webhook days/weeks later. Same treatment:
  a new compensating transaction moving money back, plus state on the original charge
  (`posted → disputed → reversed`). The original entries stay; we add correcting ones.

### 6.6 Fraud detection hook (high level, async)

Fraud scoring **must not be on the synchronous charge path** for a real-time ML scorer (latency,
availability) — but a *cheap* rules check (velocity, blocklist) can gate synchronously. The pattern:
the charge emits an event via the **outbox → Kafka**, and a **fraud stream processor** consumes it,
scores it (features over recent history, ML model), and can asynchronously flag/hold/reverse a
transaction — which is just another compensating ledger entry. This is a clean fit for a streaming
pipeline (`prep/07-messaging-and-streaming.md`, `prep/18-batch-and-stream-processing.md`). The
async-but-can-reverse model works precisely *because* reversals are first-class ledger operations.

---

## Step 7 — Wrap-up (3 min): failure modes, SPOFs, compliance

**Failure modes I'd name before being asked:**

- **PSP call times out (unknown outcome)** → never guess. Idempotency key makes a retry safe; webhook
  + reconciliation determine truth; txn stays `pending` until resolved. (6.2)
- **Client retry after our response was lost** → idempotency key UNIQUE constraint makes it a no-op
  that returns the original result. (6.1)
- **Crash between ledger write and event publish** → avoided entirely by the **outbox pattern** (both
  in one local transaction); the relay is at-least-once and consumers dedup. (`prep/10-...`)
- **Saga coordinator crashes mid-payment** → persisted saga state + idempotent steps → resume and
  finish or compensate. (6.3)
- **Dropped/duplicate/out-of-order webhook** → idempotent, state-machine-guarded handler;
  reconciliation catches anything missed. (6.2, 6.4)
- **Ledger DB leader fails** → synchronous standby in another AZ promoted (RPO=0); we accept a brief
  write outage (CP) rather than risk divergence. (`prep/05-...`)

**SPOFs and redundancy:** the ledger DB is the most critical component — multi-AZ synchronous
replication + automated failover + point-in-time recovery from the WAL. The saga coordinator and
webhook ingest are made redundant and stateless (state in the DB). The PSP itself is a SPOF we don't
own → mitigate with **multiple PSPs** and routing/failover between them (also helps with cost and
auth success rates).

**Audit / compliance:** the append-only ledger is the audit trail by construction; we never mutate or
delete; corrections are reversing entries; PCI scope is minimized by tokenizing cards (never storing
PANs); retention 7–10 years with cold partitions archived to object storage.

**What I'd do with more time:** sharding the ledger by account *if* volume ever forces it (and the
cross-shard-transfer = distributed-transaction cost that introduces); per-currency settlement
netting; a richer fraud feature store; tiered/cold storage automation for the ledger.

---

## What made this staff-level

- **Led with the consistency/durability stance and never wavered**: explicitly chose CP + RPO=0,
  and justified it by the *asymmetric cost* of being wrong, instead of reflexively scaling.
- **Made the double-entry, append-only ledger the spine of the design** and derived balances from
  it, rather than UPDATE-ing a mutable balance cell (the junior trap).
- **Treated idempotency as an end-to-end, recursive contract** — client → API → PSP → Kafka
  consumers — and tied it to the DB's UNIQUE constraint as the actual dedup mechanism.
- **Was honest about the PSP being an unreliable async oracle**, and answered the "did the charge go
  through?" question with idempotency key + webhook + reconciliation instead of a guess.
- **Used a saga with compensating ledger entries (not 2PC, not deletes)** and persisted coordinator
  state so a crash doesn't strand money.
- **Named reconciliation as the closed-loop that makes a system built on unreliable parts correct
  over time** — the operator's insight.
- **Noticed the workload is correctness-bound, not QPS-bound**, and resisted premature sharding.
- Cross-referenced the building blocks throughout instead of re-deriving them:
  consistency/replication (05), messaging/streaming (07), distributed-txn/idempotency/saga/outbox
  (10), batch/stream processing (18), microservices orchestration (16), gateway/security (09, 17),
  sharding (04), databases (03).

### Self-check (answer from memory before the mock)

- [ ] Why is balance *derived* from the ledger rather than stored as the source of truth?
- [ ] State the double-entry invariant and why it's a built-in correctness tripwire.
- [ ] Where exactly does idempotency dedup happen, and why must the key be client-supplied?
- [ ] Walk the "PSP call timed out — did it go through?" resolution path.
- [ ] Why a saga and not 2PC here? What is a "compensation" in ledger terms?
- [ ] What three sources does reconciliation diff, and how are discrepancies fixed?
- [ ] Why is this a CP, RPO=0 system, and what do you give up for that?
- [ ] Why is this *not* a high-QPS problem, and what does that change about sharding?
