# Design 11: Ticketmaster / Flash-Sale / Limited-Inventory Booking System

> **Why this problem is its own category:** most designs in this prep are read-heavy and forgive
> staleness — a feed, a search result, a video CDN. This one has a small, sharp, unforgiving core:
> **a fixed number of seats and a stampede of users all trying to grab the same ones in the same
> second.** The entire round is decided by one sentence said early and often: "We must **never
> oversell** — selling seat 14A twice is a guaranteed refund, a furious customer, and possibly a
> legal/PR incident — so the seat-inventory mutation is **strongly consistent and serialized**, and
> everything else (browse, search, the waiting room) bends around protecting that tiny consistent
> core." If you remember one thing: **this is a concurrency + consistency problem wearing a
> high-traffic costume.** The traffic is real, but it's a shield in front of the real problem, which
> is a few thousand atomic decrements that must each happen exactly once.

I'll run the standard 7-step framework. I'm flagging up front that step 6 (the deep dives) is where
this is won: the **reservation state machine**, the **no-oversell concurrency-control choice**, the
**virtual waiting room / admission control** for the thundering herd, and the **hold → pay → confirm
saga**. Those four are the round.

---

## Step 1 — Requirements (5 min)

I'll drive this and narrow scope deliberately.

### Functional

- **Browse / search events** (artist, venue, date) and **view a seat map** for an event with
  per-seat availability.
- **Hold / reserve seats**: a user selects seats and gets a short-lived exclusive hold (a "your seats
  are reserved for 10 minutes" timer). This is the crux.
- **Checkout**: pay for held seats and **confirm** the booking, producing an order + tickets.
- **Release**: a hold that isn't paid in time expires and the seats go back on sale automatically.
- **Idempotent booking**: a user double-clicking "buy" or retrying after a flaky network must not
  produce two orders or two charges.

Scope I'll explicitly cut to protect time: I'll skip dynamic pricing, resale/secondary market, fraud
scoring (mention the hook), seat-map *rendering* UX, and refunds beyond the saga compensation. I'll
cover both **reserved seating** (specific seats, the hard case) and **general admission** (a quantity
counter, the easy case) because the concurrency control differs. I'll say all this out loud so the
interviewer can pull anything back in.

### Non-functional — this is the whole game

- **No overselling — strong consistency on the seat/inventory mutation.** This is non-negotiable. A
  seat is held or sold by **at most one** buyer. This forces a **serialized, linearizable** decrement
  on the contended row(s). → `prep/05-replication-and-consistency.md`.
- **Exactly-once *effect* on a booking.** We can't get exactly-once *delivery* over a network, so we
  get exactly-once *effect* via **client-supplied idempotency keys + a dedup store**. →
  `prep/10-distributed-transactions-and-idempotency.md`.
- **Asymmetric workload by path** — and recognizing this is the key insight:
  - **Read path (browse / seat map):** enormous, spiky, **tolerates staleness of a few seconds**.
    "47 seats left" being slightly stale is fine; the truth is checked at hold time. → cache-heavy,
    eventually consistent, served far from the database.
  - **Write path (hold / book):** small in *absolute* volume but **brutally contended** at sale start
    — thousands of writes fighting over the same hundreds of rows. Must be strongly consistent.
  - This split is **CQRS-shaped**: protect the tiny consistent write core, scale the read path
    independently.
- **Thundering herd at sale start (T=0).** 100k+ people refreshing and clicking "buy" the instant a
  Taylor Swift sale opens. The system must **admit load gradually** (a virtual waiting room) rather
  than let 100k concurrent writes hit the inventory store and melt it.
- **Latency:** browse p99 < ~200ms (cache/CDN). Hold p99 < ~1s is fine — users tolerate a brief
  spinner at the moment of truth. We are **not latency-bound; we are contention-bound.**
- **Availability:** high, but on the write path I choose **CP**. If we're partitioned from the
  inventory store's quorum, I'd rather **fail the hold (user retries)** than risk a double-sell. A
  failed hold is an annoyance; an oversell is an incident.

> **The line that signals seniority:** "I'm splitting this into two systems with opposite properties.
> The browse path is AP/eventually-consistent and I'll throw cache and CDN at it without worry. The
> seat-mutation path is CP/linearizable and I'll deliberately *serialize* it and protect it behind
> admission control. Overselling is asymmetrically catastrophic versus a declined hold, so PACELC:
> under partition I pick C, and even without partition I pick C over L on the write path."

---

## Step 2 — Estimation (3 min)

Numbers exist to justify decisions. The point of this estimation is to **prove the workload is
read-dominated in volume but write-contended at a point**, which is exactly what shapes the design.

**The headline event.** A 20,000-seat arena, a hot artist. Conventional wisdom: ~1M people *want*
those 20k seats. So the demand:supply ratio is ~**50:1**. That single number drives the whole design
— most traffic *must be turned away*, so admission control is not optional.

**Read path (browse + seat map) at sale start:**
- Say **1M users** descend in the first few minutes. If each polls the seat map / event page a few
  times: **~1M users × ~1 req/s for a couple minutes ≈ 1M+ read QPS** at the spike.
- A single DB serves maybe low-thousands of QPS. **1M read QPS → this MUST be served from cache/CDN,
  not the database.** That's the justification for the read-path cache, derived not memorized.

**Write path (holds + bookings):**
- Total *successful* holds is bounded by **inventory: 20,000 seats**, period. You cannot sell more
  than you have — the output is tiny.
- But *attempts* are not bounded by inventory. In second 0, potentially **hundreds of thousands of
  hold attempts** collide on the **same few hundred contended seat rows** (everyone wants front-row
  / center). This is the real number: **write *contention*, not write *volume*.**
- If 20k seats sell over ~10 minutes, the *committed* write rate is only `20,000 / 600 ≈ 33
  bookings/s` average — **trivially small.** The difficulty is entirely that those 33/s are buried
  under a 1000× pile of contending failed attempts on hot rows.

**Storage** — negligible by big-system standards:
- Events/venues/seat-maps: thousands of events × tens of thousands of seats × a few hundred bytes →
  **single-digit GB**, easily cached.
- Orders/tickets: even 100M tickets/year × ~500 bytes ≈ **50 GB/year**. A single relational node
  holds years of this. **We do not need to shard for size.**

> **The estimation conclusion I carry forward:** *read volume is enormous (→ cache/CDN/waiting room);
> write volume is tiny but pathologically contended on hot rows (→ serialized strong-consistency
> control + admission control to flatten the spike).* This is the **opposite** of a "shard for write
> throughput" problem, and I want the interviewer to hear me notice that the hard part is
> **contention on a hot key**, not aggregate QPS. Sharding the inventory by event actually *helps*
> here (spreads hot events across nodes) but a single hot event's seats still contend — so the fix is
> concurrency control + admission, not just sharding.

---

## Step 3 — API design (3 min)

Two things to nail: **idempotency keys are first-class** on every booking write, and the **hold is an
explicit, server-owned resource with a TTL** (no fire-and-forget).

```
# --- Read path (cacheable, eventually consistent) ---
GET  /v1/events?artist=&city=&from=&cursor=     -> [events], nextCursor   # cursor, not offset
GET  /v1/events/{eventId}                        -> { event, venue, sections, priceTiers }
GET  /v1/events/{eventId}/seatmap                -> { sections:[{seats:[{id,status,priceTier}]}] }
                                                    # status is a CACHED hint: available|held|sold
                                                    # truth is decided at hold time, not here

# --- Waiting room / admission (gate in front of the write path) ---
POST /v1/events/{eventId}/queue                  -> { queueToken, position, etaSeconds }
GET  /v1/events/{eventId}/queue/{queueToken}     -> { status: waiting|admitted, accessToken? }
                                                    # accessToken (short-lived, signed) is required
                                                    # to call the hold endpoint below

# --- Write path (strongly consistent, admission-gated, idempotent) ---
POST /v1/events/{eventId}/holds
  Access-Token: <signed admission token>          # REQUIRED: proves you passed the waiting room
  Idempotency-Key: <client UUID>                  # REQUIRED on every booking write
  { seatIds: ["S-14A","S-14B"] }                  # OR { sectionId, quantity } for GA
  -> 201 { holdId, seatIds, expiresAt }           # 409 if any seat already held/sold

POST /v1/holds/{holdId}/checkout
  Idempotency-Key: <client UUID>
  { paymentMethodToken: "tok_x" }
  -> { orderId, status: pending|confirmed|failed }

DELETE /v1/holds/{holdId}                          -> 204    # user-initiated release
GET    /v1/orders/{orderId}                        -> { status, tickets[...] }
```

Notes I'd say out loud:
- The **seat-map `status` is a cache-derived hint**, not the source of truth. The race ("two people
  both see 14A available") is *expected* and is resolved authoritatively at `POST /holds` — exactly
  one of them gets the 201, the other gets a 409. Don't try to make browse perfectly consistent;
  it's a losing, expensive battle. Make the *hold* consistent.
- **`Idempotency-Key` on both `holds` and `checkout`** so a double-click or a retry after a lost
  response returns the *original* hold/order rather than creating a second. The dedup mechanism is a
  UNIQUE constraint in the DB, same as the payment design (`prep/10-...`).
- The **access token** binds the write path to the waiting room — you physically cannot attempt a
  hold without having been admitted, which is how we flatten the spike.

---

## Step 4 — Data model (5 min)

Entities + access patterns. The access pattern dictates the store. The headline decision: **the
transactional core (seats, holds, orders) is relational**, and I'll justify why rather than reflex.

```
Event        (event_id PK, venue_id, name, sale_start_at, status)
Venue        (venue_id PK, name, layout_ref)                      # seat map template
Section      (section_id PK, event_id, name, price_tier)

Seat         (seat_id PK, event_id, section_id, row, number,
              status ENUM('available','held','sold'),             # the contended cell
              hold_id NULL, version INT)                          # version for optimistic CAS
   -- index: (event_id, status) for availability counts
   -- the (event_id, seat_id) row is THE hot, contended resource

Hold         (hold_id PK, event_id, user_id, seat_ids[],
              status ENUM('active','checked_out','expired','released'),
              expires_at TIMESTAMP, idempotency_key UNIQUE)
   -- index: (expires_at) for the sweeper; (idempotency_key) UNIQUE for dedup

Order        (order_id PK, hold_id, user_id, amount, currency,
              status ENUM('pending','confirmed','failed'),
              payment_ref, idempotency_key UNIQUE, created_at)

Ticket       (ticket_id PK, order_id, seat_id, barcode)           # issued only on confirm
```

**Store choice — relational (e.g., Postgres/MySQL or a NewSQL like CockroachDB/Spanner), and here's
why, not by reflex:**
- The core operation is **"atomically flip N specific seat rows from `available` → `held`, all-or-
  nothing, with no two transactions winning the same row."** That is *the* definition of an ACID
  transaction with row-level locking. A relational engine gives me `SELECT ... FOR UPDATE`,
  transactions, and UNIQUE constraints **for free** — exactly the primitives I need.
- The data is **small** (step 2: single-digit GB hot, ~50 GB/year of orders) and **highly
  relational** (event→section→seat, hold→order→ticket). None of the NoSQL-justifying conditions
  (massive scale, flexible schema, denormalized access) hold. → `prep/03-databases-deep-dive.md`,
  `prep/04-sharding-and-partitioning.md`.
- **Sharding/partitioning by `event_id`** is natural and useful — it spreads *different* hot events
  (different concerts) across nodes so one mega-sale doesn't starve everything else. Crucially, a
  single hold is always within one event → **never a cross-shard transaction.** That's a clean shard
  key with no distributed-transaction tax.
- I'd keep the **browse/seatmap read model in a cache (Redis) and CDN**, derived from this source of
  truth via change events — that's the CQRS split (deep dive 6.4).

> **The seat row is the hot key.** Everything in step 6 is about how to mutate that one row safely
> under a thousand concurrent attempts.

---

## Step 5 — High-level design (10 min)

Let me get the happy path end-to-end, simple first, then evolve under questioning.

```
                                  ┌─────────────────────────────────────────────┐
   Browse path (AP, cached)       │                                             │
   Client ─▶ CDN ─▶ Edge cache ───┤  Read replica / Redis seat-map cache         │
            (event pages,         │  (eventually consistent; refreshed via CDC)  │
             seat maps)           └───────────────▲─────────────────────────────┘
                                                  │ change events (seat status)
                                                  │
   Write path (CP, gated)                         │
   Client ─▶ API Gateway ─▶ Virtual Waiting Room ─┼─▶ Booking Service ─▶ Inventory DB
            (authn,          (admission control,   │   (holds, checkout)   (Postgres,
             rate limit)      issues access token) │                        sharded by
                  │                                │                        event_id,
                  └── prep/09 ──────────────────── │                        ROW LOCKS)
                                                   │            │
                                                   │            ├─▶ Payment Service ─▶ PSP
                                                   │            │    (prep/07 design)
                                                   │            │
                                                   │            └─▶ Hold-expiry sweeper
                                                   │                 (releases stale holds)
                                                   └── Kafka (outbox) ─▶ cache updater, tickets,
                                                                          notifications (prep/07)
```

**Walk the happy path out loud:**

1. **Browse.** User hits the event page and seat map. Served from **CDN + Redis cache** (read
   replica behind it). Never touches the inventory DB. Seat statuses are slightly stale hints.
2. **Sale opens (T=0).** Instead of letting everyone hit `POST /holds`, the gateway routes them into
   the **Virtual Waiting Room** (deep dive 6.3). They get a `queueToken` and a position. The room
   **admits users in controlled batches** (e.g., 1,000/s) by issuing short-lived signed
   **access tokens**. This converts a 100k-concurrent spike into a manageable steady trickle.
3. **Hold.** An admitted user calls `POST /holds` with seat IDs + access token + idempotency key.
   The **Booking Service** opens a transaction, **locks and flips** those seat rows
   `available → held` (the no-oversell core, deep dive 6.2), writes a `Hold` row with
   `expires_at = now + 10min`, commits. Returns the hold + countdown.
4. **Checkout.** User pays within the TTL. `POST /holds/{id}/checkout` kicks off the
   **hold → pay → confirm saga** (deep dive 6.5): call Payment Service / PSP, and on success flip
   seats `held → sold`, create the `Order` + `Tickets`, mark hold `checked_out`.
5. **Expiry.** If the user doesn't pay in time, the **sweeper** flips the seats back
   `held → available` and the hold to `expired`, returning inventory to the pool. (deep dive 6.1)
6. **Async fan-out.** Seat-status changes are published via an **outbox → Kafka** (`prep/07-...`) to
   refresh the read cache, issue tickets/emails, and feed analytics — all off the hot write path.

The thing to emphasize: **the write path is short, synchronous, and protected; the read path and all
fan-out are async and cached.** That separation is the design.

---

## Step 6 — Deep dives (15 min): where the round is won

I'd propose: "The four interesting parts are the **reservation state machine**, the **no-oversell
concurrency control**, the **waiting room**, and the **payment saga**. Can I go deep on those?"

### 6.1 — The reservation state machine + hold TTL + releasing expired holds

A seat is a tiny state machine, and getting the transitions *atomic and complete* is the whole job:

```
                 hold (FOR UPDATE, available→held)
   available ───────────────────────────────────▶ held
       ▲                                            │  │
       │   sweeper / user release                   │  │  checkout success
       │   (held→available, TTL expired)            │  │  (held→sold)
       └────────────────────────────────────────────┘  ▼
                                                       sold  (terminal; only via confirm)
```

- **Hold = a lease with a TTL.** `expires_at = now + 10min`. The TTL exists because a user who
  abandons a cart must not lock a seat forever — inventory has to flow back. This is the classic
  **availability-vs-correctness lever**: too short and you frustrate honest buyers mid-checkout; too
  long and you strand inventory during a sellout. 10 minutes is the usual compromise; I'd make it
  configurable per event.
- **Releasing expired holds — two mechanisms, belt and suspenders:**
  1. **A sweeper job** (`SELECT ... WHERE status='active' AND expires_at < now() FOR UPDATE SKIP
     LOCKED`, then flip seats back to `available` and hold to `expired`). Runs every few seconds.
     `SKIP LOCKED` lets multiple sweeper workers run without fighting each other.
  2. **Lazy/defensive check at read time:** a hold past `expires_at` is treated as expired even if
     the sweeper hasn't gotten to it yet, so a stale hold never blocks a new buyer. The sweeper is
     then just cleanup, not correctness-critical.
- **Why not a Redis TTL key as the source of truth?** Tempting (`SET seat:14A held EX 600`), and I'll
  use Redis for the *fast contended path* (6.2 option), but the **durable truth lives in the
  relational DB** so a Redis flush doesn't lose holds or, worse, silently release sold seats. Redis
  is an accelerator in front of the DB, not the system of record for money-adjacent state.

> **State-machine discipline is a staff signal:** every transition is guarded (you can only reach
> `sold` *from* `held`, only *via* a confirmed payment), and there are no dangling states. A junior
> design forgets the `held → available` release path and silently leaks inventory until the show
> sells "out" with empty seats.

### 6.2 — Preventing double-booking / overselling (THE core decision)

The question: thousands of concurrent transactions all try to flip the same seat row. Exactly one
must win; everyone else gets a clean 409. Four honest candidates:

| Approach | How | Pros | Cons / when it bites |
|---|---|---|---|
| **Pessimistic lock** (`SELECT … FOR UPDATE`) | Lock the seat rows in a txn, flip, commit | Dead simple, correct, uses the DB's own row locks; no lost updates | Lock held for the txn duration; under extreme contention on a hot row, requests **queue/serialize** → latency + risk of lock-wait timeouts. Must keep the txn *tiny* (no PSP call inside!) |
| **Optimistic concurrency** (version / CAS) | `UPDATE seat SET status='held', version=version+1 WHERE seat_id=? AND status='available' AND version=?`; check rows-affected | No locks held; great when contention is *low* (most seats aren't fought over) | Under *high* contention on the same row, most updates fail and **retry storm** — wasteful. Loser must retry/pick another seat. |
| **Atomic decrement** (Redis `DECR` / Lua) | Keep a per-section counter (or per-seat bitmap) in Redis; atomic decrement; reject at 0 | Extremely fast, single-threaded Redis serializes for free → **perfect for GA / quantity counters** | For *specific reserved seats* you need per-seat state, not just a count; durability/failover of Redis must be handled; truth must reconcile to the DB |
| **Distributed lock** (Redlock / ZooKeeper) | Acquire a lock per seat across nodes, then write | Works if state is spread across systems | **Overkill and risky** here — the DB already gives me linearizable row locks in one place; a distributed lock adds a failure mode (lock expiry mid-operation → split brain) for no benefit. → `prep/08-consensus-and-coordination.md` |

**My pick — and I'll state it decisively:**

- **Reserved seating (the hard case): pessimistic `SELECT … FOR UPDATE`** on the specific seat rows,
  inside a **very short** transaction that *only* touches inventory (no network/PSP call inside the
  lock). It's correct by construction, uses the database's own battle-tested row locking, and on a
  single primary the contended writes naturally **serialize** — which is exactly what we want for "at
  most one winner." This is the right default.
  - I'd actually combine it with the **optimistic guard in the WHERE clause** (`AND status =
    'available'`) so even without the explicit lock the flip is conditional and can't double-apply —
    defense in depth.
- **General admission (the easy case): atomic decrement in Redis** of a section counter, with the DB
  as durable backstop. There are no distinct seats to lock, just a count; single-threaded Redis
  serializes decrements perfectly and absorbs the spike far better than DB row locks. Reject when the
  counter hits 0.
- **For the very hottest reserved-seat events**, I'd put a **Redis seat bitmap / hash as the fast
  front line** (atomic Lua `CAS` to claim a seat), then **asynchronously persist** the claim to the
  DB, with the DB UNIQUE/`FOR UPDATE` as the ultimate authority on reconciliation. This trades a
  little complexity for absorbing the contention spike in-memory. I'd only reach for it if a single
  primary's lock contention is proven to be the bottleneck — otherwise it's premature.

> **Why strong consistency, not eventual, here — the sentence that wins this:** "An oversell is
> *irreversible from the customer's standpoint* — two people show up to the same seat. There is no
> 'eventually reconcile' that fixes a person standing in an aisle. So the seat flip must be
> **linearizable**: a single authoritative decision per seat, no concurrent winners. Eventual
> consistency would let two replicas both accept the same seat and merge to a contradiction. The
> browse path can be eventually consistent precisely *because* the hold re-checks the truth; the hold
> itself cannot be."

### 6.3 — The thundering herd: virtual waiting room + admission control

The problem: 1M users (step 2) hit `POST /holds` at T=0. Even perfect locking melts if a million
connections pile onto the inventory primary. **The fix is to never let them through at once.**

- **Virtual waiting room (queue / admission control).** At sale start, users are placed in a queue
  (a Redis sorted-set or a Kafka-backed FIFO keyed by event). They get a `queueToken` + position +
  ETA. A **dispatcher admits users in controlled batches** — say 1,000/s, tuned to what the inventory
  store can comfortably serve — by issuing **short-lived signed access tokens**. The hold endpoint
  **rejects any request without a valid token**. This converts an instantaneous spike into a steady,
  survivable trickle. (This is exactly how Ticketmaster's "you are in line" page works.)
  - The waiting-room page itself is **static + CDN-served** and polls a lightweight status endpoint —
    so even the 1M people *waiting* don't touch the DB.
  - **Fairness:** admit roughly FIFO by join time; add a bit of randomization/lottery to defeat bots
    that all arrive at the same millisecond. Bind tokens to user/session so they can't be shared.
- **Rate limiting at the edge** (`prep/09-api-gateway-loadbalancing-ratelimiting.md`): per-user and
  per-IP token buckets at the gateway stop a single client (or bot) from hammering. The waiting room
  is *macro* admission control (how many enter the building); the rate limiter is *micro* (one
  person can't spin the turnstile 100×/s).
- **Serve the browse path entirely from cache/CDN** so the only traffic that reaches the protected
  DB is admitted hold/checkout attempts. The 1M read QPS from step 2 is absorbed at the edge.
- **Backpressure / load shedding** (`prep/13-resilience-and-failure-handling.md`): if the inventory
  store's latency climbs, the dispatcher *slows the admission rate* — a closed feedback loop that
  protects the core instead of letting it collapse. Shed at the gate, not at the database.

> **The reframe that signals seniority:** "I'm not trying to make the database survive a million
> concurrent writes — that's a losing fight. I'm using admission control to make sure it **never
> sees** a million concurrent writes. The waiting room is a **shock absorber** that turns a spike
> into a rate I sized the DB for in step 2."

### 6.4 — Separating read and write paths (CQRS-ish)

- **Write model:** the relational inventory DB — small, strongly consistent, the source of truth for
  seat state. Only admitted, idempotent hold/checkout traffic touches it.
- **Read model:** a denormalized **seat-map view in Redis + CDN**, eventually consistent. Built from
  the write model via **change events** (CDC / outbox → Kafka → cache updater, `prep/07-...`).
- **Why this split is safe:** the read model only needs to be *approximately* right ("~47 left"), and
  the *hold operation re-validates against the write model* so a stale read never causes an oversell
  — at worst a user clicks a seat that just got taken and gets an honest 409. We get the read
  scalability of eventual consistency **without** risking the correctness invariant, because the
  invariant is enforced only at the consistent write point.
- This is the same instinct as fan-out in the Twitter design (`prep/.../01-design-twitter-newsfeed.md`):
  push expensive read state out to cheap, scalable, eventually-consistent storage; keep the truth
  small and consistent.

### 6.5 — Payment integration + the hold → pay → confirm saga

Checkout spans two systems (our inventory + an external PSP) that can't share a transaction, so it's
a **saga with compensation**, not a 2PC. This ties directly to `prep/07-design-payment-system.md` and
`prep/10-distributed-transactions-and-idempotency.md`.

```
   [hold active] ──checkout──▶ create Order(pending)  ── idempotency-key dedup
        │                            │
        │                            ▼
        │                     charge PSP (prep/07 design; idempotency-key forwarded)
        │                       /              \
        │            success  /                  \  failure / timeout
        │                    ▼                     ▼
        │   flip seats held→sold,            release hold (held→available),
        │   issue Tickets, Order→confirmed   Order→failed; user retries
        ▼
   (TTL still the safety net underneath the whole flow)
```

- **Idempotency end-to-end:** the client's `Idempotency-Key` on `checkout` dedups our Order creation
  (UNIQUE constraint); we **forward an idempotency key to the PSP** so a retry never double-charges.
  Exactly the recursive contract from the payment design.
- **The dangerous race — hold expires mid-payment.** The PSP call takes 8s, the hold had 10s left,
  the sweeper fires at 0. Now we might charge a card for a seat we already released. Mitigations,
  in order of preference:
  1. **Lock the hold at checkout start** (transition `active → checking_out`) so the sweeper skips
     it — the seat stays held for the duration of the payment attempt, with a longer hard cap.
  2. If the payment *succeeds but the seat was already released*, we have a **compensating action**:
     attempt to re-acquire the seat; if it's gone (someone else took it), **auto-refund** and
     apologize. Money is recoverable (refund) — the seat is not, so we never confirm an oversell.
- **Payment succeeds but confirm fails (our DB write of `held→sold` fails after the charge).** Never
  guess. The Order sits `pending`; an **outbox + reconciliation** loop (same pattern as the payment
  design) compares PSP truth to our orders and either completes the confirm (idempotently) or
  refunds. The **outbox pattern** ensures the "publish booking confirmed" event and the DB commit are
  atomic (`prep/10-...`).
- **Why a saga, not 2PC:** the PSP is an external system we don't control and can't enlist in a
  distributed commit; 2PC would also hold locks across a multi-second network call (fatal under our
  contention). The saga keeps each step local and idempotent, with **release-the-hold** as the
  compensation. → `prep/16-microservices-and-architecture.md` for orchestration.

---

## Step 7 — Wrap-up (3 min): failure modes, SPOFs

**Failure modes I'd name before being asked:**

- **Hold expires mid-payment** → lock the hold during checkout (`active → checking_out`, sweeper
  skips it); if a charge lands on a released seat, re-acquire or auto-refund. Seat never oversold.
  (6.5)
- **Payment succeeds but confirm write fails** → Order stays `pending`; outbox + reconciliation
  against the PSP completes or refunds idempotently. (6.5)
- **Two users grab the same seat** → expected and handled: one wins the `FOR UPDATE` flip, the other
  gets a clean 409. No oversell by construction. (6.2)
- **Client double-clicks "buy" / retries after a lost response** → `Idempotency-Key` UNIQUE
  constraint makes it a no-op returning the original hold/order. (Step 3, `prep/10-...`)
- **Sale-start stampede** → virtual waiting room admits in batches; browse served from CDN; the DB
  only ever sees the rate it was sized for. (6.3)
- **Redis (fast-path / counter) fails or flushes** → the relational DB is the durable source of
  truth; we degrade to DB-direct (slower, still correct) and reconcile. We never trust Redis alone
  for sold-seat state. (6.1, 6.2)
- **Sweeper dies / lags** → lazy expiry at read time means a stale hold never blocks a new buyer even
  before the sweeper catches up; correctness doesn't depend on the sweeper's punctuality. (6.1)
- **Inventory DB primary fails** → synchronous standby in another AZ promoted (CP: brief write
  outage over divergence); reads continue from replicas/cache. (`prep/05-...`)

**SPOFs and redundancy:** the **inventory DB is the most critical component** — multi-AZ synchronous
replication, automated failover, PITR from the WAL; sharded by `event_id` so one hot event can't
starve others. The **waiting-room dispatcher** is made redundant and its queue state lives in
replicated Redis/Kafka. The Booking Service, Payment Service, and sweeper are **stateless and
horizontally scaled** (state in the DB). The PSP is a SPOF we don't own → mitigate with multiple PSPs
and the saga/reconciliation closing the loop.

**What I'd do with more time:** anti-bot/fraud scoring at the waiting-room gate (the hardest real
Ticketmaster problem); seat-adjacency / best-available auto-assignment; dynamic pricing as a separate
service reading the same inventory; per-event shard placement and warming caches ahead of a known
sale time; a "next best seats" suggestion when a hold loses the race.

---

## What made this staff-level

- **Diagnosed the workload correctly up front:** read-*volume*-heavy but write-*contention*-heavy on
  a hot key — and built two systems with opposite consistency properties (CQRS-ish) instead of one
  compromise.
- **Made the no-oversell invariant the spine** and chose **strong/linearizable consistency on the
  seat flip** with a clear, justified concurrency-control pick (pessimistic `FOR UPDATE` for reserved
  seats, atomic Redis decrement for GA) — and *honestly compared* it against optimistic CAS and
  distributed locks rather than reciting one answer.
- **Treated the thundering herd as an admission-control problem, not a scaling problem** — the
  waiting room is a shock absorber that makes the DB never see the spike, sized to step-2 numbers.
- **Modeled the seat as an explicit, fully-guarded state machine** with a TTL'd hold and *two* release
  mechanisms (sweeper + lazy expiry), so inventory never leaks and correctness doesn't depend on a
  cron's punctuality.
- **Used a saga with hold-release compensation (not 2PC)** for the cross-system checkout, kept the PSP
  call *outside* the lock, and named the nasty races (hold-expires-mid-payment, charge-without-confirm)
  with idempotency + reconciliation as the resolution.
- **Resisted premature sharding/over-engineering** — recognized the data is small, sharded only by
  `event_id` for hot-event isolation (never a cross-shard transaction), and chose relational because
  the access pattern *is* an ACID transaction.
- Cross-referenced the building blocks instead of re-deriving them: consistency/replication (05),
  caching (06), messaging/outbox (07), consensus/locks (08), gateway/rate-limiting (09),
  idempotency/saga (10), resilience/backpressure (13), microservices orchestration (16), and the
  payment design (07).

### Self-check (answer from memory before the mock)

- [ ] Why is this a concurrency+consistency problem and not a raw-QPS problem? What number proves it?
- [ ] Why must the seat flip be linearizable while the browse path can be eventually consistent?
- [ ] Compare `FOR UPDATE` vs optimistic CAS vs atomic decrement vs distributed lock — when does each
      win, and what's your pick for reserved seats vs GA?
- [ ] Draw the seat/hold state machine. What are the two ways an expired hold gets released, and why
      both?
- [ ] How does the virtual waiting room actually flatten the spike, and how is the hold endpoint
      protected from un-admitted traffic?
- [ ] Walk the hold→pay→confirm saga. What's the compensation? What happens if the hold expires
      mid-payment? If payment succeeds but confirm fails?
- [ ] Why relational, and why is `event_id` a clean shard key (no cross-shard transactions)?
- [ ] Why is overselling asymmetrically worse than a declined hold, and how does that drive CP over
      AP/L on the write path?
