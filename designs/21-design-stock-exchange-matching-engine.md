# Design 21: Stock Exchange / Order Matching Engine (Low-Latency Trading)

> **Why this problem is its own category:** almost every other design in this prep is a *distributed*
> system whose job is to *scale out* — shard it, replicate it, cache it, fan it out across a hundred
> machines, and tolerate a little staleness for availability. A matching engine is the **opposite
> animal.** The hard part is a tiny, sharp core that must be **single-threaded, in-memory,
> deterministic, and correct to the microsecond** — because it is matching buy and sell orders for
> *money*, in a regulated market, where two participants who sent orders one nanosecond apart have
> *legally* different priority. If you remember one sentence: **this is an ordering + determinism +
> latency problem, and the entire design is about establishing one global order of events and then
> replaying that order identically forever.** The classic junior instinct — "throw it on a cluster,
> add locks, scale horizontally" — is *exactly wrong* here. We go the other way: a lock-free,
> single-writer, in-memory state machine fed by one totally-ordered log.

I'll run the standard 7-step framework. I'm flagging up front that step 6 is where this is won, and the
four deep dives are: the **limit-order-book data structure + matching algorithm**, the **single-threaded
deterministic engine (LMAX-style) and why**, **sequencing → an ordered replicated log for durability +
deterministic replay**, and **HA via hot replicas consuming the same ordered input with no split-brain**.
Those four are the round.

---

## Step 1 — Requirements (5 min)

I'll drive this and narrow scope deliberately.

### Functional

- **Submit a limit order**: buy/sell N shares of a symbol at a price *no worse than* a limit
  (buy ≤ limit, sell ≥ limit). Rests in the book if it can't fully match.
- **Submit a market order**: buy/sell N shares at *whatever* the book offers right now (no price
  bound). Never rests — fills as much as possible immediately, cancels the remainder (or rejects).
- **Cancel / amend an order**: remove a resting order, or change its qty/price (amend = cancel +
  re-add, which **loses time priority** — an important fairness detail).
- **Match orders** by **price-time priority** (a.k.a. FIFO): best price first; within a price level,
  the order that arrived *first* fills *first*. This is the heart of the system.
- **Emit execution reports** (fills/partial-fills/acks/rejects/cancels) back to the order owner, and
  **market data** (top-of-book, depth, last trade) to everyone.

Scope I'll explicitly cut to protect time: I'll skip the *exotic* order types (stop, iceberg,
fill-or-kill, pegged — I'll mention how they slot in), the **opening/closing auction** (a batch
single-price cross — mention it), client connectivity protocol details (FIX/binary — mention),
and **clearing & settlement** beyond acknowledging it's a downstream async system (T+1). I'll say this
out loud so the interviewer can pull anything back in.

### Non-functional — this is the whole game

- **Latency: microseconds, not milliseconds.** The engine's order-to-ack and order-to-match should be
  **single-digit microseconds** internally; wire-to-wire (gateway in → market data out) on the order
  of tens of µs. This single number kills entire architectures: no per-order DB write, no network hop
  *inside* the match, no GC pause, no lock contention. → `prep/19-networking-fundamentals.md`.
- **Strict ordering & fairness.** There must be **one canonical order** in which orders are processed,
  identical for everyone, and matching must honor **price-time priority** exactly. If Alice's order
  arrived before Bob's at the same price, Alice fills first — always, deterministically. This is a
  regulatory/fairness requirement, not a nicety.
- **Correctness: no lost, no duplicated, no reordered orders.** Every accepted order is processed
  **exactly once**, in sequence. Money is at stake; an oversold/double-matched order is an incident.
- **Determinism.** Given the same ordered sequence of input events, the engine must produce the
  **bit-identical** sequence of trades and book states — every time, on every replica, on replay
  after a crash. This is the property that makes everything else (HA, recovery, audit) possible.
- **Throughput: high but bounded.** A single liquid symbol can see **hundreds of thousands to low
  millions of messages/sec** at peak (most are cancels/amends, not trades). The engine must absorb
  this on one core.
- **Durability & auditability.** Every input event and every resulting trade must be **persisted
  durably before it's acted on**, and the full ordered log retained for **regulatory replay/audit**
  (years). Regulators can and will ask "reconstruct the book as of 10:31:42.000123."
- **Availability.** Trading halts are extremely costly and visible. We need **hot failover in
  milliseconds**, but — critically — **never at the expense of determinism or ordering.** A split-brain
  where two engines accept the same order and disagree on the match is *catastrophic*. So when forced,
  I choose **consistency/ordering over availability**: better a brief halt than two divergent books.

> **The line that signals seniority:** "This is not a scale-out problem; it's a *scale-in* problem. The
> matching core is deliberately a **single-threaded, in-memory, deterministic state machine** — one
> writer, no locks, no shared mutable state, everything in L1/L2 cache. All the distributed-systems
> machinery (replication, consensus, recovery) lives *around* that core in the form of a single
> **totally-ordered, replicated input log**: we agree on the order *once*, then the engine is a pure
> function `state' = f(state, event)` replayed identically everywhere. Determinism is the invariant
> the entire design protects."

---

## Step 2 — Estimation (3 min)

Numbers exist to justify decisions. The point here is to **prove that one core can do it**, that the
data is tiny, and that the latency budget forbids the usual building blocks.

**Message rate (the real load is messages, not trades).** Take a major exchange-like venue:
~10,000 symbols, peak aggregate **~10M messages/sec** across all of them at the open. The key insight:
**partition by symbol.** A single hot symbol (think a mega-cap on a volatile day) might see
**~500k–1M messages/sec** alone; a typical symbol far less. So one engine instance owns a *slice* of
symbols, and we size each instance to its hottest symbol.

- A well-engineered single-threaded matching engine (LMAX published ~6M ops/sec on one thread years
  ago) comfortably handles **hundreds of thousands to a few million simple ops/sec on one core.** So
  **one core per symbol-group is plausible** — that's the whole reason single-threaded works here.
- **Cancel:trade ratio is high** — often 10:1 to 30:1. Most messages are limit orders that rest and
  cancels that pull them. Trades (matches) are a small fraction. Good: matching is the expensive part
  and it's the minority.

**Latency budget — the constraint that shapes everything.** Target wire-to-wire ~**10–50 µs**.
Decompose it:
- Network in (kernel-bypass NIC → user space): a few µs.
- Gateway validation/risk: a few µs.
- Sequencer assign + journal write: **the durability step must be µs**, so journal to a memory-mapped
  ring on fast NVMe / replicate over a low-latency link, *not* fsync-to-spinning-disk (ms) and *not* a
  remote DB write (ms). This single line eliminates "write each order to Postgres."
- Match in the engine: **sub-µs to low-µs** (in-memory, cache-resident).
- Market data out (multicast): a few µs.

> A single SSD `fsync` is ~hundreds of µs to ms; a cross-AZ network round trip is ~0.5–1 ms; a DB write
> is ms. **Every one of these is 100–1000× over our entire budget.** That math *forces* the design:
> in-memory engine, in-memory/replicated journal, no synchronous DB on the hot path. Persistence and
> DB-of-record happen **off the critical path**, fed asynchronously from the ordered log.

**Storage / audit.** An order/event is ~tens of bytes (ids, symbol, side, price, qty, ts). At 10M
msg/s × ~64 B × a 6.5-hour trading day ≈ **~15 TB/day** of raw event log — large but it's a sequential
append log on commodity storage, not a random-access DB, and it compresses well. Retained for years for
audit → cheap object storage in cold tiers. The **hot working set** (the live order books) is tiny:
even millions of resting orders × ~64 B ≈ **hundreds of MB to a few GB total**, easily RAM-resident per
engine. → `prep/24-capacity-planning-and-cost.md`.

> **The estimation conclusion I carry forward:** *the live state is tiny (RAM-resident, one core can
> own a symbol), the message rate is high but partitionable by symbol, and the latency budget (µs) is
> 100–1000× smaller than any disk/DB/network-RPC operation.* That trio is precisely why the answer is a
> **single-threaded in-memory engine fed by a fast ordered log**, with all durability and DB writes
> pushed **off** the hot path. I want the interviewer to hear me derive "no DB on the hot path" from the
> budget, not assert it.

---

## Step 3 — API design (3 min)

Two things to nail: orders are submitted over a **low-latency binary/FIX session** (not REST — REST/JSON
over TCP+TLS is ms-scale and absurd here), and every order carries a **client order id** for idempotent
dedup and an **assigned sequence number** from the venue for canonical ordering.

```
# --- Order entry session (binary or FIX over a persistent, often kernel-bypass, socket) ---
# Conceptually:

NEW_ORDER     { clOrdId, account, symbol, side(BUY|SELL), type(LIMIT|MARKET),
                price?, qty, tif(GTC|IOC|FOK|DAY) }
              -> ACK   { clOrdId, exchOrdId, seqNo, ts }            # accepted, maybe resting
              -> REJECT{ clOrdId, reason }                          # failed risk/validation
              -> FILL  { clOrdId, exchOrdId, execId, lastPx, lastQty, leavesQty, ts }  # 0..N of these

CANCEL_ORDER  { clOrdId, origClOrdId | exchOrdId }
              -> CANCEL_ACK   { ... }   |   CANCEL_REJECT { reason }   # too-late = already filled

AMEND_ORDER   { clOrdId, origClOrdId, newQty?, newPrice? }
              -> AMEND_ACK / REJECT      # NOTE: price change loses time priority (cancel+new)

# --- Market data (one-way fan-out, multicast; NOT request/response) ---
MD_TRADE      { symbol, px, qty, aggressorSide, seqNo, ts }
MD_BOOK_DELTA { symbol, side, price, newQtyAtLevel, seqNo, ts }     # incremental L2 update
MD_SNAPSHOT   { symbol, bids:[{px,qty}], asks:[{px,qty}], seqNo }   # periodic, for late joiners
```

Notes I'd say out loud:
- **`clOrdId` is the idempotency key.** A client retrying after a lost ack must not create a second
  live order; the gateway dedups on `(account, clOrdId)`. Exactly the idempotent-effect contract from
  `prep/10-distributed-transactions-and-idempotency.md`.
- **Every accepted message gets a monotonic `seqNo` from the sequencer** (step 6.3). That number *is*
  the canonical order; it appears on acks, fills, and market data so every participant can verify they
  saw events in the one true order and detect gaps (request a snapshot to recover).
- **Market data is fire-and-forget fan-out**, not a query API. Subscribers get a continuous stream of
  deltas + periodic snapshots; a late or gap-detecting subscriber re-syncs from the latest snapshot
  then resumes deltas. → `prep/15-realtime-and-push.md`.
- **Amend that changes price = cancel + new order at the back of the new level** — it loses time
  priority. Quote-stuffing and priority-jumping rules live here; I'll mention but not belabor.

---

## Step 4 — Data model (5 min)

There are really *two* data models: the **in-memory order book** (the hot, latency-critical
structure — the entire round depends on this being right and cache-friendly) and the **durable
event/trade log + DB-of-record** (off the hot path, for recovery, settlement, audit).

### The in-memory limit order book (per symbol)

```
OrderBook(symbol):
  bids: sorted map  price -> PriceLevel   (descending: highest buy = best)
  asks: sorted map  price -> PriceLevel   (ascending:  lowest sell = best)

PriceLevel:
  price
  totalQty
  fifoQueue: doubly-linked list of Orders   # arrival order == time priority

Order:
  exchOrdId, clOrdId, account, side, price, qty, leavesQty, seqNo, ts
  prev, next                                # intrusive list pointers within its PriceLevel

orderIndex: hash map  exchOrdId -> Order*   # O(1) lookup for cancel/amend
```

Access patterns and why this exact shape:
- **Match needs "best price, then oldest order at that price."** → the price→level map is kept sorted
  so the best bid/ask is **O(1)** to find (it's the front), and within a level the **FIFO doubly-linked
  list** gives the oldest order first and **O(1)** removal when it fully fills. Price-time priority falls
  directly out of "sorted levels + FIFO queue per level."
- **Cancel/amend needs "find this specific order fast."** → a hash map `exchOrdId -> Order*` gives
  **O(1)** lookup; because the order node carries intrusive `prev/next` pointers, unlinking it from its
  level's queue is **O(1)** too. No scanning.
- **Why not a balanced BST / `std::map` for prices and call it done?** For the *price ladder* a tree is
  fine, but in practice prices are bounded and discrete (tick size), so the highest-performance engines
  use a **flat array indexed by price-tick** (a "ladder") for O(1) best-level access and great cache
  locality — no pointer chasing through a tree. I'd state the tree as the simple version and the
  price-indexed array as the optimization, justified by the latency budget.

> **The data structure *is* the algorithm here.** "Sorted price levels + a FIFO queue per level + a
> hash index for cancels" gives price-time priority, O(1) best-price, and O(1) cancel — and it's chosen
> to be **cache-friendly and pointer-light** because we're spending nanoseconds, not milliseconds.

### The durable side (off the hot path)

```
EventLog   (seqNo PK, ts, type, payload)        # the ordered, replicated input journal (append-only)
TradeLog   (execId PK, seqNo, symbol, buyOrdId, sellOrdId, px, qty, ts)   # every match
OrderState (exchOrdId PK, account, symbol, side, price, origQty, leavesQty, status)  # DB-of-record
```

- The **EventLog is the source of truth for *order*** — it's the LMAX/event-sourcing journal
  (`prep/25-workflow-orchestration-and-event-sourcing.md`, `prep/07-messaging-and-streaming.md`). The
  in-memory book is a **derived projection** of replaying the log.
- **TradeLog + OrderState** feed downstream: clearing/settlement, risk, the client-facing "my orders"
  views, and regulatory audit. They're written **asynchronously by consumers of the log**, never on the
  matching hot path. This is event sourcing + CQRS: the engine emits events; many read models are built
  from them.

> **The seat/hold equivalent here is the EventLog ordering.** Everything in step 6 is about how we make
> that log *the one true total order*, persist it fast enough to meet the budget, and replay it
> deterministically.

---

## Step 5 — High-level design (10 min)

Let me get the happy path end-to-end. The shape is a **pipeline**, not a request/response service:
gateway → sequencer/journal → matching engine → market data, with everything downstream fed from the log.

```
                          ORDER ENTRY (the hot path — microseconds)

 Trader ──(binary/FIX, kernel-bypass)──▶  ORDER GATEWAY  ──┐
                                          - session/auth   │  validated, accepted orders
                                          - validation     │  (still UNORDERED across gateways)
                                          - risk checks     │
                                          - rate limit /    ▼
                                            throttle      ┌──────────────────────────────────┐
                                          (prep/09)       │   SEQUENCER  (single writer)      │
                                                          │   - assigns monotonic seqNo       │
                                                          │   - appends to ORDERED JOURNAL ───┼──▶ replicated to
                                                          │     (in-mem ring + fast journal)  │     2+ replicas (sync)
                                                          └───────────────┬──────────────────┘     (prep/08 ordering)
                                                                          │  one totally-ordered stream
                                          ┌───────────────────────────────┼───────────────────────────┐
                                          ▼                               ▼                            ▼
                              ┌────────────────────┐         ┌────────────────────┐        (cold standby /
                              │  MATCHING ENGINE    │         │  MATCHING ENGINE    │         replay-on-recover)
                              │  PRIMARY (symbol    │ same    │  REPLICA (HOT)      │
                              │  group A)           │ ordered │  - consumes SAME    │
                              │  - single-threaded  │ input   │    ordered log      │
                              │  - in-memory book   │────────▶│  - deterministic ⇒  │
                              │  - deterministic    │         │    identical book   │
                              └─────────┬───────────┘         └────────────────────┘
                                        │ emits: fills, acks, book deltas, trades (also ordered, with seqNo)
            ┌───────────────────────────┼───────────────────────────────────────────┐
            ▼                           ▼                                             ▼
   EXECUTION REPORTS           MARKET DATA PUBLISHER                      DOWNSTREAM CONSUMERS (async)
   back to the owning          - fan-out via MULTICAST / pub-sub          - TradeLog / OrderState DB-of-record
   trader's session            - L2 deltas + periodic snapshots           - Clearing & settlement (T+1)
   (prep/15)                   - to 1000s of subscribers (prep/15)        - Risk, surveillance, audit, analytics
                                                                          (prep/07 log → consumers)
```

**Walk the happy path out loud:**

1. **Submit.** A trader sends `NEW_ORDER` over a persistent low-latency session to the **Order
   Gateway**. The gateway authenticates the session, **validates** (well-formed, valid symbol/tick/lot
   size), runs **pre-trade risk** (does the account have buying power / position limits / fat-finger
   price band?), and **rate-limits/throttles** the session. Bad orders are rejected *here*, before they
   ever reach the engine — keeping the engine's input clean and its path short. (deep dive 6.4)
2. **Sequence + journal.** The accepted order flows to the **Sequencer** — the *single writer* that
   stamps it with the next **monotonic `seqNo`** and **appends it to the ordered journal**, which is
   **synchronously replicated** to standby replicas before the engine acts on it. This step *is* the
   commit point: once journaled+replicated, the order is durable and its position in the global order is
   fixed forever. (deep dives 6.2, 6.3)
3. **Match.** The **Matching Engine** (single-threaded, in-memory) consumes the ordered stream in
   `seqNo` order and processes each event: a new limit buy walks the asks; while the best ask price ≤
   the order's limit and qty remains, it **matches** against the oldest order at that level (price-time
   priority), generating fills; any unfilled remainder **rests** in the book. It emits acks, fills, and
   book deltas — themselves sequenced. (deep dive 6.1)
4. **Hot replica.** A **replica engine** consumes the **exact same ordered log** and, being
   deterministic, produces the **identical book and trades** — ready for **instant failover** without
   re-deriving state. (deep dive 6.4 / 6.5)
5. **Report + publish.** Fills/acks route back to the **owning trader's session**; book deltas and
   trades go to the **Market Data Publisher**, which **fans out** to thousands of subscribers via
   **multicast / pub-sub** with periodic snapshots for late joiners. (deep dive 6.6)
6. **Async downstream.** Consumers of the ordered log write the **TradeLog/OrderState DB-of-record**,
   feed **clearing & settlement** (T+1, async), risk, surveillance, and audit — *all off the hot path*.
   (deep dive 6.7)

The thing to emphasize: **the hot path is gateway → sequence/journal → match → publish, all in-memory
and in microseconds; everything durable, queryable, or distributed (DB writes, settlement, analytics,
recovery) hangs off the ordered log asynchronously.** That separation is the design.

---

## Step 6 — Deep dives (15 min): where the round is won

I'd propose: "The four interesting parts are the **order book + matching algorithm**, **why the engine is
single-threaded/in-memory/deterministic**, **sequencing into an ordered replicated log for durability +
replay**, and **HA with hot replicas and no split-brain**. Can I go deep on those?"

### 6.1 — The order book + matching algorithm (make it concrete)

Let me walk an actual match. Book for symbol XYZ:

```
        ASKS (sells, ascending)                 BIDS (buys, descending)
   100.03 │ 200 (ord A, then ord B FIFO)    100.01 │ 500 (ord C)
   100.02 │ 300 (ord X)                      100.00 │ 800 (ord D)
   ───────┴── best ask = 100.02 ───────      ──────┴── best bid = 100.01 ──────
                              spread = 100.02 − 100.01
```

**Incoming: limit BUY 400 @ 100.03** (aggressive — willing to pay up to 100.03):

1. Best ask is **100.02 (300 @ ord X)**, and 100.02 ≤ 100.03 → **match.** Fill 300 against X.
   X is fully filled → unlink from the level (O(1)); level 100.02 now empty → remove. Remaining: 100.
2. Next best ask is **100.03 (ord A=120, ord B=80 FIFO)**, 100.03 ≤ 100.03 → match the remaining 100.
   Fill 100 against **A** (the *oldest* at this level — time priority). A had 120 → 20 leaves; A stays at
   the front with `leavesQty=20`. Incoming order now fully filled (0 leaves) → done, rests nothing.
3. **Emit:** two FILLs (vs X 300@100.02, vs A 100@100.03), an ACK, and book deltas (100.02 removed,
   100.03 reduced to 100). Trades carry the **aggressor side** and a `seqNo`.

Key algorithmic points I'd state:
- **Aggressor vs resting / price improvement:** the *resting* order's price is the trade price (here X's
  100.02 and A's 100.03), so the incoming buyer who was willing to pay 100.03 got *price improvement* on
  the first 300. This is standard and worth naming.
- **Order types fall out of the same loop:** a **market** order is "match while qty remains, ignore the
  price bound, never rest" (and cancel/reject any unfilled remainder per the TIF). **IOC** = match now,
  cancel remainder. **FOK** = check fillable-in-full first, else reject untouched. **Stop** orders sit in
  a side structure and are *injected* as market/limit orders when the trigger price trades. I'd mention,
  not implement.
- **Complexity:** best-price access **O(1)** (front of sorted ladder), match per level walks the FIFO
  in arrival order, cancel/amend **O(1)** via the hash index. A burst of cancels (the common case) is
  cheap. The expensive case — an aggressive order sweeping many levels — is bounded by the levels it
  crosses, which is small in a liquid book.

> **Determinism check, stated explicitly:** given the same ordered input, this loop produces the *same*
> trades every time — there is no clock-reads, no randomness, no concurrency, no floating point in the
> price math (use **integer ticks/cents**, never floats — float non-determinism would break replay).
> That discipline is what makes 6.4/6.5 work.

### 6.2 — Why a single-threaded, in-memory, deterministic engine (the LMAX Disruptor pattern)

This is the counterintuitive heart of the design, and I'd defend it head-on against the naive "just use
threads and locks and a database."

- **Single-threaded = no locks, no contention, no nondeterminism.** A multithreaded book needs locks
  around price levels; locks mean contention (the hottest symbol is exactly where threads collide),
  context switches, and — fatally — **nondeterministic interleaving**, which destroys replay. One thread
  owning one symbol-group's book has **zero lock overhead** and a **single, total processing order by
  construction.** Counterintuitively, *one core with no synchronization beats many cores fighting over
  locks* for this workload. (This is the LMAX result: ~6M ops/sec on a single thread.)
- **In-memory = the latency budget demands it.** Per step 2, any disk/DB/network touch on the hot path
  is 100–1000× over budget. The working set is small (sub-GB), so the entire book lives in RAM,
  ideally hot in **L1/L2 cache** — hence the cache-friendly, pointer-light structures in 6.1.
- **The Disruptor / mechanical-sympathy pattern** wraps this: a **pre-allocated ring buffer** as the
  inter-stage hand-off (gateway → journaller → replicator → matcher), single-writer per stage,
  **no garbage allocation on the hot path** (object pooling / reuse — a GC pause of even 10 ms is ~1000×
  our budget and would be a visible market event), and **cache-line padding** to avoid false sharing.
  Producers and consumers spin on sequence counters instead of locking.
- **Deterministic = state machine over an ordered log.** The engine is a **pure function**
  `bookState' = apply(bookState, event)`. No wall-clock reads in matching logic (timestamps come *in*
  the event from the sequencer); no randomness; integer arithmetic only. Given the log, the output is
  fixed. This single property is what unlocks: **(a)** crash recovery by replay, **(b)** hot replicas
  that stay bit-identical, **(c)** regulatory "reconstruct the book at time T," and **(d)** testing by
  feeding a recorded log and asserting identical output.

> **Latency engineering — at a *conceptual* level (don't overdo it):** the µs budget is met not by
> clever distributed tricks but by **mechanical sympathy** — keep work on one core, keep data in cache,
> never allocate or GC on the hot path, use **kernel-bypass networking** (e.g., DPDK/Solarflare-style
> user-space NICs) so packets skip the kernel TCP stack, and **colocate** the engine, gateways, and
> participants' servers in the same datacenter (cross-DC light-speed alone is ~5 ms/1000km — fatal).
> The point in an interview: *I know the budget forbids the usual building blocks, and I know the
> category of techniques that buy back the microseconds*, without pretending to hand-tune assembly.

> **The reframe that signals seniority:** "The instinct to 'scale horizontally with locks and a DB' is
> exactly wrong here. Correctness and ordering are *easier and faster* with a single deterministic
> writer than with a distributed lock dance — so I deliberately make the core a lock-free single-threaded
> state machine and put all the distributed-systems work into getting **one ordered, durable input log**
> *in front of* it."

### 6.3 — Sequencing → ordered, replicated event log (durability + deterministic replay)

This is where event sourcing (`prep/25-...`) and messaging/logs (`prep/07-...`) meet ordering
(`prep/08-consensus-and-coordination.md`). The crux: **establish one total order, durably, before the
engine matches.**

- **The Sequencer is the single point that defines order.** Many gateways accept orders concurrently
  (orders arrive unordered, racing). The sequencer is the **one writer** that imposes a total order by
  stamping a monotonic `seqNo` and appending to the **journal**. Whatever order the sequencer picks *is*
  the truth — fairness is "first to reach the sequencer wins," and venues invest heavily in making that
  arrival fair (equalized cable lengths, etc.). I'd name that fairness-at-the-sequencer point.
- **Journal before match (the durability commit point).** The event is appended to a durable,
  append-only log **and synchronously replicated to ≥2 replicas** *before* the engine acts on it. To
  meet the budget, this is **not** an fsync-to-disk-per-event (ms); it's an append to a memory-mapped
  ring streamed to replicas over a low-latency link, with the *replicas' acknowledgment* providing
  durability (data exists on N machines) rather than waiting on a single disk. Disk persistence of the
  log happens continuously but is not in the per-order critical wait.
- **The engine is a deterministic projection of the log.** `book = replay(log)`. This means:
  - **Crash recovery:** a fallen engine restarts and **replays the log from the last checkpoint**
    (periodic in-memory snapshots bound replay time) to rebuild the *exact* book — no separate
    "recovery DB," no ambiguity. The log is the source of truth.
  - **Audit/regulatory replay:** replay to any `seqNo`/timestamp reconstructs the book and every trade
    exactly. Auditability is free because we kept the ordered input forever.
- **Idempotency / exactly-once *processing*:** the `seqNo` makes dedup trivial — the engine and every
  consumer track "last seqNo applied" and ignore anything ≤ it on replay, so a re-delivered event during
  failover is applied **exactly once**. Combined with gateway `clOrdId` dedup, no order is lost or
  duplicated. → `prep/10-...`.

> **The invariant, stated plainly:** *the ordered, replicated log is the source of truth; the order book
> is a deterministic, replayable projection of it.* Once you accept that, durability is "the log is
> replicated," recovery is "replay the log," HA is "replay the same log on another node," and audit is
> "we kept the log." One idea pays for four hard requirements.

### 6.4 — High availability without losing determinism (hot replicas, no split-brain)

The tension: we need failover in milliseconds, but **two engines must never both accept orders and
diverge.** I'd attack this directly.

- **Primary + hot replica(s) consuming the *same* ordered log.** Because matching is deterministic and
  the input is one ordered stream, a replica that reads the same log is **bit-identical** to the primary
  at every `seqNo`. So failover is "promote the replica" — it already *is* the live book, no rebuild,
  no warm-up. (This is the payoff of 6.2 + 6.3.)
- **The single sequencer prevents split-brain at the source.** Split-brain in a matching context would
  be two engines both deciding "Alice's order matched Bob's" differently. We make that *impossible* by
  having **exactly one sequencer** define order. The engines don't *vote* on order — they *obey* the
  log. So the only HA question is "who is the sequencer," which is a classic **leader-election** problem.
- **Sequencer failover via consensus / fencing.** The sequencer's role is guarded by leader election
  (Raft/ZooKeeper-style — `prep/08-...`) with a **fencing token / epoch**: when a new sequencer is
  elected, the epoch increments, and journal replicas **reject appends from a stale (lower-epoch)
  sequencer.** So even if the old sequencer is merely slow (not dead) and wakes up, its writes are
  fenced out — **no two sequencers can both extend the one log.** This is the precise mechanism that
  kills split-brain. → `prep/08-consensus-and-coordination.md`.
- **Choosing C over A when forced.** If we genuinely can't establish a single sequencer (network
  partition isolating the consensus group), the correct move is to **halt trading for that symbol-group**
  rather than let two partitions accept divergent orders. A brief, clean halt is recoverable and
  auditable; a divergent book is a regulatory catastrophe. **PACELC: under partition I pick C
  (ordering/consistency); and even in normal operation I pick the single-writer path over a lower-latency
  multi-writer one** because determinism is non-negotiable.
- **Failure detection & gap recovery.** Replicas and consumers track `seqNo` continuity; a gap triggers
  re-request from the log. Heartbeats + the consensus group detect a dead primary engine/sequencer in
  milliseconds and promote.

> **The sentence that wins this:** "I get HA *without* sacrificing determinism by never asking nodes to
> *agree on the match* — I ask them only to *agree on the order of inputs*, once, via a single fenced
> sequencer, and then every deterministic engine independently derives the *same* book. Consensus is
> used narrowly — to elect the one sequencer and fence the old one — not on the hot path per order. That's
> why failover is a promotion, not a reconciliation."

### 6.5 — Recovery, checkpointing, and the determinism invariant in practice

- **Checkpoints (snapshots) bound replay time.** Replaying a full 15 TB/day log on restart is too slow,
  so the engine periodically writes a **consistent snapshot of the book at a known `seqNo`** (off-thread,
  from a copy, so it doesn't stall matching). Recovery = load latest snapshot + replay only the log
  *after* that `seqNo`. Standard event-sourcing snapshotting (`prep/25-...`).
- **Replay must be deterministic to be trustworthy.** This is why 6.1/6.2 banned wall-clock reads,
  randomness, and floats in matching: replay on a different machine, a year later, must yield the
  identical trades — that's the audit guarantee. Any nondeterminism (e.g., iterating a hash map in
  insertion-undefined order) is a *bug that corrupts replay* and I'd call it out as a thing to test for
  by replaying recorded production logs and diffing output.
- **End-of-day / start-of-day:** at close, the book is snapshotted and persisted; at open it's restored
  (or the day starts flat, depending on order persistence rules), and the auction cross runs. Mention,
  don't deep-dive.

### 6.6 — Market data dissemination (the read fan-out)

The write side is one core; the **read side fans out to thousands of subscribers** and must also be
low-latency and fair. → `prep/15-realtime-and-push.md`.

- **One-way streaming, not request/response.** The engine emits a continuous, sequenced stream of
  **book deltas** (L2 updates) and **trades**. The Market Data Publisher **fans these out**, classically
  via **UDP multicast** on a colocated network so one packet reaches all subscribers simultaneously (fair
  + scalable — fan-out cost is independent of subscriber count). Over the internet, this becomes a
  pub-sub / WebSocket fan-out tier (CDN-of-streams style), which is higher latency and for non-colocated
  consumers.
- **Snapshots + deltas for joiners and gap recovery.** A subscriber that joins late or detects a `seqNo`
  gap **applies the latest periodic snapshot then resumes deltas** — the same snapshot+log idea as
  recovery, applied to the read side. This is why every message carries a `seqNo`.
- **Fairness in dissemination matters too:** all subscribers should receive market data
  near-simultaneously (multicast gives this); selectively delaying some feeds is a regulated concern.
  Tiered feeds (top-of-book free vs full-depth paid) are a product/regulatory layer on top.
- **Decouple from the hot path:** the publisher reads from the engine's output ring/log and does the
  fan-out work itself, so a slow subscriber **never backpressures the matching engine** — slow consumers
  are dropped/reset and re-sync from a snapshot. The engine's latency is never hostage to the slowest
  reader.

### 6.7 — Post-trade: clearing, settlement, ledger, audit (all async)

Once a trade is matched, the *matching* job is done in µs; everything after is **asynchronous** and
correctness-by-reconciliation, much closer to the payment design (`prep/07-design-payment-system.md`).

- **Clearing & settlement is T+1 (or T+2), async.** The exchange matches and reports trades; a clearing
  house nets positions and the actual exchange of cash and securities settles a day later. So settlement
  is a **downstream consumer of the TradeLog**, not on the hot path — it's a workflow/saga over the
  durable trade events. → `prep/25-...`, `prep/10-...`.
- **The ledger / books-of-record** are built by consuming the ordered log (event sourcing): positions,
  balances, and a double-entry ledger derived from trades — same instinct as the payment system's ledger.
  Because it's derived from the immutable log, it's **auditable and reproducible**.
- **Surveillance & audit** also consume the log: spoofing/layering detection, regulatory reporting, and
  the ability to **replay any moment** for an investigation. Retaining the full ordered event log for
  years is what makes all of this possible — and it's cheap sequential cold storage.

> **Why async is correct post-trade (and why it'd be *wrong* on the trade itself):** the *match* must be
> immediate, ordered, and deterministic because it decides priority and price for money. But the *money
> movement* is inherently a multi-party, multi-system workflow that can tolerate seconds-to-days and is
> made correct by reconciliation, not by being inline. Keeping it off the hot path is what lets the
> engine stay in µs. This is the same "tiny consistent core, async everything else" split as the other
> money designs — just with the consistent core being *ordering/determinism* rather than a balance
> decrement.

---

## Step 7 — Wrap-up (3 min): failure modes, SPOFs

**Failure modes I'd name before being asked:**

- **Matching engine crashes** → restart a replica/standby that's already hot (deterministic replay of the
  same ordered log) or rebuild from latest snapshot + log tail. No state is lost because the **log**, not
  the engine's RAM, is the source of truth. (6.3, 6.4, 6.5)
- **Sequencer fails** → leader election promotes a new sequencer; **epoch/fencing** rejects writes from
  the old one, so the single-total-order invariant holds. No split-brain. (6.4)
- **Network partition isolates the consensus group** → **halt** that symbol-group rather than risk two
  divergent books. Chosen C over A deliberately; halt is auditable and recoverable. (6.4)
- **GC pause / allocation on the hot path** → eliminated by design: object pooling, pre-allocated ring
  buffers, no per-order allocation; a pause would be a visible latency spike. (6.2)
- **Duplicate / retried order after a lost ack** → gateway dedups on `(account, clOrdId)`; engine/consumers
  dedup on `seqNo`. Exactly-once *processing*. (6.3, `prep/10-...`)
- **Slow market-data subscriber** → publisher drops/resets it and forces a snapshot re-sync; it **never
  backpressures the engine.** (6.6)
- **Bad/fat-finger order** → caught at the gateway by pre-trade risk + price bands before it reaches the
  engine; engine input stays clean. (6.4)
- **Disk/replica loss on the journal** → log is replicated to ≥2 nodes; lose one and durability holds;
  re-replicate from survivors. (6.3)
- **Nondeterminism bug (float math, map iteration order, clock read in match)** → caught by **replaying
  recorded production logs and diffing trade output**; this is the highest-severity class of bug because
  it silently breaks recovery, replicas, and audit. (6.5)

**SPOFs and redundancy:** the **sequencer + ordered journal is the most critical component** — it's
made HA via leader election + epoch fencing and synchronous replication, but it is *intentionally* a
single *logical* writer (that's the design, not a flaw). The **matching engine** is made redundant by hot
deterministic replicas. **Gateways and market-data publishers are stateless and horizontally scaled.**
The whole system is **partitioned by symbol-group** so one symbol's engine/sequencer failure or halt
doesn't take down the rest of the market — blast radius is one symbol-group.

**What I'd do with more time:** the **opening/closing auction** (a batch single-price uncross — different
algorithm from continuous matching); **exotic order types** (iceberg/pegged/stop) and how they integrate
with the same FIFO core; **self-trade prevention** and anti-manipulation/surveillance; the precise
**fairness mechanics at the sequencer** (equalized latency, anti-gaming); **cross-symbol** order types
(spreads/baskets) which break the clean per-symbol partition and need careful coordination; and a deeper
treatment of **kernel-bypass + FPGA** acceleration for the gateway/risk path.

---

## What made this staff-level

- **Diagnosed the workload as the *opposite* of a scale-out problem:** tiny in-memory state, µs latency
  budget, ordering+determinism as the dominant requirements — and chose a **single-threaded, lock-free,
  in-memory deterministic engine** (LMAX/Disruptor) *on purpose*, defending it against the naive
  "cluster + locks + DB" instinct.
- **Made determinism the spine of the whole design** and showed it pays for four hard requirements at
  once: durability (replicated log), crash recovery (replay), HA (hot replicas stay bit-identical), and
  audit (regulatory replay) — `book = replay(orderedLog)`.
- **Separated "agree on the order of inputs" from "compute the match"** — used a **single fenced
  sequencer + narrow consensus** to establish one total order, then let independent deterministic engines
  derive the *same* book, which is how failover becomes a *promotion* not a *reconciliation* and
  split-brain becomes impossible.
- **Derived "no DB on the hot path" from the latency budget** (disk/DB/network are 100–1000× over) and
  pushed all persistence, settlement, ledger, and analytics **off** the critical path as async consumers
  of the ordered log (event sourcing + CQRS).
- **Got the core concrete:** the limit-order-book structure (sorted price levels + FIFO queues + hash
  index), walked an actual price-time-priority match with price improvement, derived the order types from
  the same loop, and called out the **integer-price / no-clock / no-randomness** discipline that makes
  replay trustworthy.
- **Treated the read fan-out as its own low-latency, fair problem** (multicast + snapshots/deltas,
  decoupled so slow readers never backpressure the engine).
- **Cross-referenced the building blocks** instead of re-deriving them: networking (19), messaging/log
  (07), consensus/ordering/fencing (08), event sourcing + snapshots (25), idempotency (10), realtime
  fan-out (15), resilience (13), capacity (24), and the payment/ledger design (07).

### Self-check (answer from memory before the mock)

- [ ] Why is this a *scale-in* (single-threaded, in-memory, deterministic) problem rather than a
      scale-out one? What numbers (latency budget, state size, msg rate) prove it?
- [ ] Draw the order book. How do sorted price levels + per-level FIFO queues + a hash index give you
      price-time priority, O(1) best price, and O(1) cancel?
- [ ] Walk an aggressive limit order matching across two price levels. Who sets the trade price, and what
      gets emitted?
- [ ] Why single-threaded and lock-free beats multithreaded-with-locks here. What is the Disruptor
      pattern buying you (no GC, ring buffer, mechanical sympathy)?
- [ ] State the determinism invariant `book = replay(orderedLog)`. Which four requirements does it pay
      for, and why must matching avoid clocks/randomness/floats?
- [ ] How does sequencing + a replicated journal give durability and exactly-once processing within the
      µs budget (and why *not* fsync-per-order or a DB write)?
- [ ] How do you get HA without split-brain? What is the single sequencer's role, and how do
      leader-election + epoch fencing prevent two writers?
- [ ] When do you choose C over A (halt), and why is a halt better than divergent books?
- [ ] How is market data fanned out at low latency and fairly, and how does a slow subscriber *not*
      hurt the engine?
- [ ] Why is clearing/settlement async and off the hot path, while the match itself is not?
