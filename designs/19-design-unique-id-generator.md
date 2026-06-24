# Design 19: Distributed Unique ID Generator (Snowflake-style)

> **How to read this doc:** This is a *full worked walkthrough* of the 7-step framework from
> [Topic 1](../prep/01-framework-and-building-blocks.md), solved live as if in a 45-min round.
> This is a *focused systems* problem — there's almost no estimation theatre and no sprawling
> architecture diagram. The entire round is decided in the **deep dive**: comparing ID schemes on
> explicit axes, then defending a 64-bit bit layout down to the individual bit and naming the
> clock-skew failure mode before the interviewer does. Everything in **bold/blockquote** is
> something I'd actually say out loud.

---

## 0. Why this problem is worth taking seriously

A junior says "use a UUID" or "use a database auto-increment column" and stops. A staff engineer
hears a set of conflicting constraints that *cannot all be satisfied by the obvious answers*:

1. **Uniqueness at scale with no coordination** — every node must mint IDs locally, at high rate,
   without a network round-trip to a shared counter on the hot path. The moment generation requires
   coordination *per ID*, you have a bottleneck and a single point of failure.
2. **Sortability vs. randomness** — we want IDs that are roughly time-ordered (so they cluster in a
   B-tree and double as a coarse timestamp), but time-ordering makes them *enumerable*, which leaks
   data. UUID v4 gives the opposite trade: random and non-enumerable, but terrible for index locality.
3. **64 bits, not 128** — the constraint that it fit in a `BIGINT` is what kills the easy answer
   (UUID is 128 bits) and forces a *packed bit layout*, which is the heart of Snowflake.
4. **The clock is a shared mutable resource you don't control** — Snowflake's correctness depends on
   wall-clock time monotonically advancing, and NTP makes that *false*. Handling clock-going-backwards
   is the single most important deep-dive moment.

I'll drive the conversation toward the bit layout, worker-ID assignment, and clock skew.

---

## 1. Requirements (5 min) — I drive this

### Functional
- **Generate** a unique 64-bit ID on request, returnable as an `int64`/`BIGINT`.
- IDs are **globally unique** across all generator nodes, forever (no reuse).
- IDs are **roughly time-sortable** — *k-sorted*: if A is generated meaningfully later than B, A > B,
  but two IDs minted within the same millisecond across different nodes may be out of strict order.
- The generator exposes a tiny surface: "give me an ID." No lookups, no storage of issued IDs.

> "I'll scope this to *minting* 64-bit, k-sorted, globally-unique IDs at high throughput. I'll
> explicitly skip: the *meaning* of the ID beyond ordering, cross-datacenter ID reconciliation
> beyond worker-ID assignment, and any persistence of issued IDs — by design we never store them."

### Non-functional (this is where the design is decided)
| Dimension | Target | Why it matters |
|---|---|---|
| **Uniqueness** | **Absolute, no duplicates ever** | A duplicate ID = two distinct objects collapsing into one. This is the one property we cannot relax. |
| **Throughput** | **10k+ IDs/sec/node, scalable to millions globally** | An ID generator sits in front of *every* write in the system. It must never be the throughput ceiling. |
| **Latency** | **Sub-microsecond, local** | Generation must be an in-process function call (a few bit-shifts), not a network hop. Anything else is a non-starter on the hot path. |
| **Sortability** | **k-sorted (roughly time-ordered)** | Buys us index locality, free pagination, and a coarse "created-at" — see [§6.6](#66-why-sortable-ids-matter--and-the-privacy-cost). |
| **Availability** | **No single point of failure** | If the ID generator is down, the whole system can't accept writes. Must be fully decentralized — no shared counter on the hot path. |
| **Coordination** | **None on the hot path** | This is *the* defining constraint. Coordination is allowed *rarely* (worker-ID assignment at startup), never per-ID. |

> **The framing sentence:** "I need locally-generated, 64-bit, collision-free, roughly-time-ordered
> IDs with **zero coordination per ID** and **no single point of failure**. That sentence alone rules
> out a shared counter and points me straight at a packed-bit scheme like Snowflake."

Map to blocks ([Topic 1, Part B](../prep/01-framework-and-building-blocks.md)): a **coordination
service** ([etcd/ZooKeeper](../prep/08-consensus-and-coordination.md)) — but used *only* at startup
for worker-ID assignment, never per request. Everything else is local arithmetic.

---

## 2. Estimations (3 min) — light here, but the throughput math is the whole point

There's no storage to size (we don't persist IDs) and no read path. The only number that matters is
**how many unique IDs a scheme can mint per unit time**, because that decides the bit layout.

### Target volume
Assume a large system: **1M new objects/sec at peak** globally (every write across every service —
orders, messages, events, rows). Spread over, say, ~100 generator nodes → **~10k IDs/sec/node**
average, with bursts well above that.

### The headline number: IDs per millisecond per node
This is the number that justifies the **sequence bit width** in Snowflake (see [§6.5](#65-snowflake-the-bit-layout-defended-bit-by-bit)).

| Quantity | Value |
|---|---|
| Sequence bits per node per ms | **12 bits** → `2^12` = **4,096 IDs / ms / node** |
| Per node per second | `4,096 × 1,000` = **~4.1M IDs / sec / node** |
| With ~1,024 worker IDs | `4.1M × 1,024` ≈ **~4.2 billion IDs / sec** globally |

> "12 sequence bits gives **4,096 IDs per millisecond per node** — over 4 million per second per node.
> Our requirement is 10k/sec/node, so we have **~400× headroom** per node before we'd ever roll over
> the per-ms sequence. That headroom is exactly what I'll point to when I justify the bit split."

### Lifespan of the timestamp field
| Quantity | Value |
|---|---|
| Timestamp bits | **41 bits** of milliseconds |
| Span | `2^41` ms = `2^41 / (1000 × 60 × 60 × 24 × 365)` ≈ **~69.7 years** |

> "41 bits of milliseconds buys ~69 years from whatever custom epoch I pick. Choosing a recent epoch
> (e.g. 2020-01-01) instead of the Unix epoch means those 69 years start *now*, not in 1970 — so the
> service is good until ~2090. That epoch choice is a free 50 years and a classic thing interviewers
> probe."

---

## 3. API design (3 min)

The API is deliberately tiny — most of the "API" is a library call, not a network call.

```
// In-process library (the common, low-latency path):
nextId() -> int64        // a few bit-shifts; no network, no lock contention across nodes

// Optional network service (when many languages / sidecar pattern):
GET /id            -> 200 { id: 1530412345678901234 }
GET /ids?count=n   -> 200 { ids: [ ... ] }   // batch to amortize the network hop
```

- **Prefer the library.** The whole point is *no network hop per ID*. A network ID service
  reintroduces latency and a dependency on the hot path — only justified when you have many runtimes
  and want one implementation (then batch with `?count=n` to amortize).
- No auth on the generator itself (internal service); it sits behind the
  [API gateway / mTLS mesh](../prep/09-api-gateway-loadbalancing-ratelimiting.md) like any internal RPC.
- Note there is **no "lookup" or "validate" endpoint** — IDs aren't stored. Uniqueness is structural,
  not checked.

---

## 4. Data model (5 min) — almost none, and that's the insight

> "The striking thing about this problem is there's **no data store for the IDs themselves**. We never
> persist what we've issued. Uniqueness comes from the *structure* of the ID, not from a uniqueness
> check against a table. That's the whole trick — and it's why this scales without coordination."

The only state that exists:

| State | Where it lives | Notes |
|---|---|---|
| `workerId` | Assigned at node startup | From config, or leased from [etcd/ZooKeeper](../prep/08-consensus-and-coordination.md). Must be **unique per live node**. See [§6.5.1](#651-assigning-worker-ids-the-coordination-that-is-allowed). |
| `lastTimestamp` | In-memory, per node | The last ms we minted in. Used to detect clock-going-backwards and to roll the sequence. |
| `sequence` | In-memory, per node | Counter within the current ms, reset each new ms. |

That's it. Three small pieces of per-node state, two of them purely in-memory and thread-local-ish.
There is no shared mutable state on the hot path — which is precisely why there's no bottleneck.

---

## 5. High-level design (10 min) — the happy path

```
   ┌──────────────────────────────────────────────────────────────┐
   │  Startup (rare, coordinated):                                  │
   │     Node ──register──► etcd / ZooKeeper ──leases──► workerId   │
   └──────────────────────────────────────────────────────────────┘

   Per-ID (hot path, NO coordination):

   caller ──nextId()──► ┌─────────────────────────────────────┐
                        │  Local generator (in-process)        │
                        │   1. now = currentMillis()           │
                        │   2. if now < lastTs  → CLOCK SKEW!   │  (see §6.7)
                        │   3. if now == lastTs → seq++         │
                        │        if seq overflows → spin to next ms
                        │   4. if now >  lastTs → seq = 0       │
                        │   5. id = (ts<<22)|(worker<<12)|seq   │
                        └──────────────────┬──────────────────┘
                                           │
                                       int64 id  (returned in nanoseconds)
```

### The mint algorithm, walked out loud
1. Read the current wall-clock time in milliseconds.
2. **If the clock went backwards** (`now < lastTimestamp`): we are in danger of producing a duplicate
   or out-of-order ID — handle it (wait or reject; see [§6.7](#67-the-clock-skew-problem-the-make-or-break-deep-dive)).
3. **If we're still in the same millisecond** (`now == lastTimestamp`): increment the sequence
   counter. If the sequence overflows (4,096 IDs already minted this ms), **busy-wait until the next
   millisecond** and reset the sequence.
4. **If we've advanced to a new millisecond** (`now > lastTimestamp`): reset the sequence to 0.
5. Compose the 64-bit integer by **bit-shifting** the three fields into place and OR-ing them. Update
   `lastTimestamp`. Return.

> "Notice step 5 is just three shifts and two ORs — a handful of CPU cycles. There is **no lock that
> spans nodes, no network call, no DB write**. Two different nodes can never collide because their
> `workerId` bits differ. One node never collides with itself because within a ms the sequence is
> unique, and across ms the timestamp differs. Uniqueness is *guaranteed by construction*."

The startup path (worker-ID assignment) is the *only* place coordination happens, and it happens once
per node lifetime, not per ID.

---

## 6. Deep dives (15 min) — where the round is won

### 6.1 The candidate approaches, compared on explicit axes

I'll evaluate every scheme on five axes: **size, uniqueness guarantee, sortability, coordination on
the hot path, and SPOF.** Here's the summary; the rows are defended below.

| Scheme | Size | Unique by | Sortable? | Hot-path coordination | SPOF |
|---|---|---|---|---|---|
| **UUID v4 (random)** | **128 bit** | randomness (prob.) | **No** | none | none |
| **DB auto-increment** | 64 bit | shared counter | Yes (strict) | **yes — every write** | **yes — the DB** |
| **DB ticket servers (step/offset)** | 64 bit | disjoint residues | Yes (k-sorted) | none (per ID) | reduced (N servers) |
| **Snowflake** | **64 bit** | structure (worker+seq) | **Yes (k-sorted)** | **none** | none (after startup) |
| **ULID / KSUID** | 128 bit | time + randomness | **Yes (lexicographic)** | none | none |

> "Three things jump out: only Snowflake and the DB approaches hit 64 bits *and* sortable *and*
> coordination-free, and only Snowflake also has no SPOF. So Snowflake is my default — but I want to
> walk through *why* the others lose, because the reasons are the interesting part."

### 6.2 UUID v4 (random) — the seductive wrong answer

- **Idea:** 128 random bits (122 random + version/variant bits). E.g. `f47ac10b-58cc-4372-a567-0e02b2c3d479`.
- **Pro — zero coordination:** generated entirely locally, no shared state, no SPOF. Collision
  probability is astronomically low (birthday bound over 122 bits). This is genuinely attractive.
- **Con — 128 bits, not 64:** **double the storage** of a `BIGINT` in every primary key, every foreign
  key, and *every index that references it*. In a system with billions of rows and many indexes, that's
  a large, permanent tax. It fails our 64-bit requirement outright.
- **Con — not sortable, and this is the killer:** v4 is *random*, so consecutive IDs land at random
  positions in the index. This destroys **B-tree locality**.

> **Tie to [databases doc §03](../prep/03-databases-deep-dive.md):** a B-tree (the index behind most
> SQL primary keys) keeps keys *sorted on disk in pages*. With a **monotonic** key (Snowflake,
> auto-increment), every insert appends to the **right-most leaf page** — the hot page stays in cache,
> pages fill densely, and you get sequential I/O. With a **random** key (UUID v4), each insert hits a
> *random* leaf page: you constantly fault cold pages into the buffer pool, pages split and leave
> half-empty (**index fragmentation / low fill factor**), and write amplification climbs. On a large
> table this can be a multiple-x throughput difference on inserts. This is the canonical reason
> "UUID v4 as a clustered primary key" is an anti-pattern, and it's the strongest argument *for*
> time-sortable IDs.

> "So UUID v4 wins on simplicity and loses on the two things we explicitly asked for — 64 bits and
> sortability. If someone *must* use UUIDs, I'd push them to **UUID v7** (time-ordered, standardized
> 2024) which fixes the locality problem — it's essentially ULID in UUID clothing."

### 6.3 DB auto-increment / single-server ticket — simple, sortable, fatal SPOF

- **Idea:** a single database table with an `AUTO_INCREMENT` / `SERIAL` column; every ID request does
  an insert and returns the generated value. Or a dedicated "ticket server" doing `REPLACE INTO` and
  `SELECT LAST_INSERT_ID()`.
- **Pro:** dead simple, **strictly** monotonic (perfect sortability), exactly 64 bits, no collisions
  by construction.
- **Con — single point of failure:** if that DB is down, **no new IDs can be issued anywhere** → the
  whole system stops accepting writes. Catastrophic.
- **Con — throughput bottleneck:** every ID across the entire system serializes through one counter.
  Even with the row lock held briefly, you're capped at the IOPS of one node, and you've added a
  **network round-trip on the hot path** — violating our sub-microsecond, no-coordination requirement.

> "It's the textbook 'works on day one, melts at scale' answer. The SPOF and the per-ID network hop
> are disqualifying. But it leads naturally to the fix — *shard the counter*."

### 6.4 DB ticket servers with step/offset (Flickr-style) — sharding the counter

This is the clever evolution of §6.3 and worth knowing cold, because it shows you understand how to
**remove a single counter without coordination**.

- **Idea:** run **N** ticket DB servers. Give each a different **starting offset** and the **same
  step = N**, so their issued IDs are *disjoint residue classes* mod N:
  - Server A: `auto_increment_offset=1, auto_increment_increment=2` → mints **1, 3, 5, 7, …**
  - Server B: `auto_increment_offset=2, auto_increment_increment=2` → mints **2, 4, 6, 8, …**
- **How it avoids collisions:** the two sequences are mathematically disjoint (odds vs. evens). No two
  servers can ever produce the same value, *with no communication between them*. Generalizes to N
  servers with step N and offsets 1..N.
- **Pro:** removes the single-counter SPOF (any one server can die and the others keep issuing); N× the
  throughput; still 64-bit; coordination happens once at *config* time (assigning offsets), not per ID.
- **Con — only k-sorted, not strict:** across servers, IDs interleave (server A may be at 1,001 while B
  is at 6) so global order is *approximate*. Usually fine.
- **Con — still a network hop per ID** unless you batch. Mitigate by having clients **lease a range**
  ("give me IDs 1000–1999") and burn it locally — which is *exactly the same idea* as the
  KGS/range-allocation pattern in the [URL shortener](02-design-url-shortener.md) deep dive.
- **Con — rebalancing pain:** adding a server later means re-choosing step/offsets carefully so you
  don't overlap historically-issued ranges; this is the operational wart.

> "Flickr's step/offset trick is the bridge between 'single counter' and 'fully distributed.' It
> proves you can get disjoint IDs from independent nodes with **no per-ID coordination** — and that's
> precisely the principle Snowflake takes to its logical conclusion by encoding the 'which node' part
> directly into the bits."

### 6.5 Snowflake: the bit layout, defended bit by bit

Snowflake (originated at Twitter) is my pick. A 64-bit integer, partitioned into four fields:

```
 0 | 0000000000 0000000000 0000000000 0000000000 0 | 0000000000 | 000000000000
 ▲   ▲                                              ▲              ▲
 │   │  41 bits: timestamp (ms since custom epoch)  │ 10 bits:     │ 12 bits:
 │   │                                              │ machine/     │ sequence
 1 bit: sign (always 0 → ID stays positive)         │ worker id    │ (per-ms counter)
```

| Field | Bits | Buys us | Limit |
|---|---|---|---|
| **Sign** | **1** | Keeps the value **positive** as a signed `int64`/`BIGINT` — avoids negative IDs that break sorting and surprise languages without unsigned ints (Java!). | always 0 |
| **Timestamp (ms)** | **41** | The **sortability** — high bits = time, so numeric order ≈ time order. Doubles as a coarse `createdAt`. | `2^41` ms ≈ **~69 years** from the chosen epoch |
| **Machine / worker ID** | **10** | The **uniqueness across nodes** with no coordination — each node owns a distinct value, so their ID spaces never overlap (same principle as Flickr offsets). | `2^10` = **1,024** concurrent workers |
| **Sequence** | **12** | The **uniqueness within a node within a millisecond** — a per-ms counter. | `2^12` = **4,096 IDs / ms / node** |

> "Read top to bottom: timestamp first means **numeric comparison ≈ chronological comparison**, which
> is the entire point. Worker ID guarantees two machines never collide *without talking to each other*.
> Sequence guarantees one machine never collides *with itself* inside a millisecond. The sign bit is a
> footgun-avoidance bit — without it, a future timestamp flips the sign and your IDs go negative and
> sort wrong, which is a real Java-era Snowflake bug."

**The split is a tunable budget, not sacred.** 41/10/12 is Twitter's choice. The trade is:
- More **machine** bits → more nodes, fewer IDs/ms/node.
- More **sequence** bits → higher per-node burst rate, fewer nodes.
- Some variants split the 10 machine bits into **5 datacenter + 5 worker** (32 datacenters × 32 workers).

> "I'd state the split I'm using and *why*: 1,024 workers and 4,096 IDs/ms covers our 100-node,
> 10k/sec/node target with ~400× headroom. If we needed 100k nodes, I'd steal bits from the sequence;
> if we needed million/sec/node bursts, I'd steal bits from the worker field. **Naming that this is a
> bit-budget trade is the staff signal.**"

#### 6.5.1 Assigning worker IDs — the coordination that *is* allowed

Worker uniqueness is the one correctness dependency Snowflake has. Options, weakest to strongest:

| Method | How | Risk |
|---|---|---|
| **Static config** | Bake `workerId` into each host's config/env. | Human error → **two hosts with the same ID = silent collisions**. Fine for a fixed fleet. |
| **Coordination service (preferred)** | On startup, the node **leases** a free worker ID from [ZooKeeper / etcd](../prep/08-consensus-and-coordination.md) via a sequential ephemeral node / a lease with TTL. On crash, the lease expires and the ID returns to the pool. | Adds a startup dependency, but **not on the hot path** — leased once, then cached in memory. |
| **Derive from infra** | Use the host's ordinal in a StatefulSet, or a hash of a stable host attribute. | Hash collisions; ordinal reuse on rescheduling. |

> **Tie to [consensus doc §08](../prep/08-consensus-and-coordination.md):** "This is exactly the kind
> of thing consensus stores are *for* — agreeing on one value (which worker owns which ID) across
> nodes. I use it sparingly: the lease is acquired **once at startup** and held; the generator never
> talks to ZooKeeper to mint an ID. That keeps the hot path coordination-free while still preventing
> the worker-ID-collision failure mode. The ephemeral-lease-with-TTL pattern also means a dead node's
> ID is automatically reclaimable instead of being stranded forever."

A subtle correctness point: a node must **not** mint IDs until it has *confirmed* its lease, and on
losing its lease (e.g. ZK session expiry / long GC pause) it must **stop minting** until it re-acquires
one — otherwise two nodes could briefly share an ID and collide.

### 6.6 ULID / KSUID — the modern lexicographically-sortable alternatives

If the "must be 64-bit / `BIGINT`" constraint is *relaxed* and you just want a sortable, string-friendly,
coordination-free ID, these are the modern answers and worth naming:

| | **ULID** | **KSUID** |
|---|---|---|
| Size | 128 bit | 160 bit |
| Layout | 48-bit ms timestamp + 80-bit randomness | 32-bit sec timestamp + 128-bit randomness |
| Encoding | 26-char Crockford base32 | 27-char base62 |
| Sortable | **Lexicographically** (string sort = time sort) | **Lexicographically** |
| Coordination | none | none |

- **Key idea:** put the **timestamp in the high bits** (like Snowflake) but fill the low bits with
  **randomness** instead of a worker+sequence. Randomness replaces the worker ID as the
  collision-avoidance mechanism — so you get sortability *and* no worker-ID assignment problem.
- **Pro over UUID v4:** time-ordered → good B-tree locality (fixes §6.2's problem); encodes to a
  sortable string, nice for log filenames, S3 keys, cursors.
- **Pro over Snowflake:** **no worker-ID coordination at all** (randomness handles uniqueness), and no
  clock-going-*backwards* duplicate risk (random bits differ even if the ms repeats).
- **Con:** 128/160 bits — **fails the 64-bit requirement**, same storage tax as UUID. And ordering is
  only *coarse* (per-ms), with intra-ms order being random.

> "If the interviewer says 'do the IDs really have to fit in a BIGINT?' I'd pivot to **ULID** (or
> **UUID v7**, which is the IETF-standardized version of the same time-prefixed idea). They give
> Snowflake's sortability *and* drop the worker-ID coordination entirely — at the cost of 2× storage.
> So: **Snowflake when 64 bits matters; ULID/UUIDv7 when string-friendliness and zero coordination
> matter more than size.**"

### 6.7 The clock-skew problem — the make-or-break deep dive

> "Snowflake's correctness rests on an assumption that is *false in practice*: that wall-clock time
> only moves forward. **NTP corrections, leap seconds, and VM live-migration can make the clock jump
> backwards.** If `now < lastTimestamp`, naively we'd start re-issuing timestamps we've already used —
> and produce **duplicate IDs**. This is the failure mode I'd raise unprompted."

**Why it produces duplicates:** if the clock rewinds by, say, 5ms, the generator will, over the next
5ms of re-traversed time, emit the *same* (timestamp, worker, sequence) tuples it already emitted →
collisions. Sortability also breaks.

Strategies to handle `now < lastTimestamp`, weakest to strongest:

| Strategy | Behavior | When to use |
|---|---|---|
| **Wait it out** | Block (busy-wait) until `now ≥ lastTimestamp`, then resume. | **Small** skews (a few ms). Simple, preserves uniqueness, brief latency blip. The common default. |
| **Reject / error** | If the backwards jump exceeds a threshold (e.g. > a few ms or > some max), **refuse to mint** and throw, so the caller fails fast / fails over to another worker. | Large jumps — better to error than risk duplicates. This is what Twitter's implementation does past a threshold. |
| **Borrow extra bits / alternate clock** | Some variants keep an extra "rollback" counter or use a logical clock that only moves forward (`max(now, lastTs+1)`). | When you can spend bits and want to *never* block or reject. |
| **Randomness instead of clock** | Switch to ULID/UUIDv7 — random low bits make a repeated ms harmless. | When clock discipline can't be guaranteed at all. |

**Operational hygiene that prevents most of this:**
- Run **NTP in slew mode**, not step mode — corrections are applied *gradually* (speeding/slowing the
  clock) so it **never jumps backwards**, only runs slightly fast/slow. This is the single most
  important fix.
- **Disable leap-second steps**; use leap smearing.
- Monitor clock drift and alert; a node whose clock is misbehaving should be pulled.

> "My answer: **NTP slew mode + wait-on-small-skew + reject-on-large-skew.** Slewing means the clock
> essentially never goes backwards; the wait handles tiny residual jitter without dropping requests;
> the reject is the safety valve that guarantees we *never* sacrifice uniqueness — we'd rather return
> an error and fail over than issue a duplicate ID. Uniqueness is the one requirement we said we can't
> relax, so when forced to choose, we trade availability for it."

### 6.8 Why sortable IDs matter — and the privacy cost

**Why we want sortability (the upside):**
- **DB locality / insert throughput:** monotonic keys append to the right-most B-tree page → dense
  pages, sequential I/O, cache-hot leaf (the inverse of §6.2's UUID-v4 problem). Directly ties to
  [databases doc §03](../prep/03-databases-deep-dive.md).
- **Free pagination:** `WHERE id > :lastSeenId ORDER BY id LIMIT n` gives stable, efficient
  **cursor-based pagination** with no separate sort column and no offset-skew on inserts.
- **No extra sort key:** the ID *is* the chronological order, so you don't carry and index a separate
  `created_at`. One column does double duty.
- **Coarse timestamp for free:** you can extract approximate creation time straight from the ID's high
  bits — handy for debugging, TTLs, range scans by time.

**The privacy downside (raise this unprompted):**
- Time-ordered IDs are **enumerable and leak volume.** Because they increase roughly monotonically, an
  observer who sees two IDs minted an hour apart can estimate **how many objects you created in that
  hour** (the "German tank problem"). Sequential exposure also enables **scraping** by walking IDs.
- **Mitigations:** never expose the raw ID where enumeration matters — map to an opaque external token
  (hashids / a separate random public ID), or use a non-sortable scheme (UUID v4) *specifically* for
  externally-visible identifiers while keeping a Snowflake internally for storage.

> "So sortability is a genuine **two-edged trade**: it's a gift to the database and a leak to the
> outside world. I'd use Snowflake for the **internal** primary key (locality, pagination) and a
> separate **opaque/random public ID** for anything a user or competitor can see. Naming this split is
> the staff-level move — junior answers expose the sortable ID directly and leak business metrics."

---

## 7. Wrap-up (3 min)

**Failure modes & mitigations:**
- *Clock goes backwards (skew/leap second):* the headline risk → NTP slew mode + wait-on-small,
  reject-on-large ([§6.7](#67-the-clock-skew-problem-the-make-or-break-deep-dive)). Never trade away
  uniqueness.
- *Worker-ID collision:* two nodes with the same ID silently collide → lease worker IDs from
  etcd/ZooKeeper, stop minting on lease loss ([§6.5.1](#651-assigning-worker-ids-the-coordination-that-is-allowed)).
- *Sequence rollover within a ms:* handled by spinning to the next ms — only an issue if a single node
  exceeds 4,096 IDs/ms (we have ~400× headroom; if it became real, re-budget bits).
- *Timestamp epoch exhaustion (~2090):* a known, far-off limit; documented, and a 41→42 bit migration
  or new epoch is the eventual fix.
- *No SPOF on the hot path:* by design — generation is local arithmetic; the only shared dependency
  (ZooKeeper) is touched once at startup, and nodes cache their lease.

**Tradeoffs restated — when each approach wins:**
- **Snowflake:** default when you need **64-bit, sortable, coordination-free, no-SPOF** IDs. The price
  is the clock dependency and worker-ID management.
- **UUID v4:** when you want **zero infra and non-enumerable** IDs and don't care about 64 bits or
  index locality (e.g. external tokens, or short-lived objects).
- **ULID / UUID v7:** when you want Snowflake's sortability **without** worker-ID coordination and can
  spend 128 bits — the modern sweet spot if `BIGINT` isn't mandatory.
- **DB ticket / step-offset:** when you're small, want strict monotonicity, and a relational DB is
  already there — but mind the SPOF/throughput ceiling.

**What I'd do with more time:** a sidecar ID service with batch leasing for polyglot fleets; metrics on
clock drift and sequence-rollover rate per node; a Feistel/encrypted wrapper to expose non-enumerable
external IDs while keeping the sortable internal ID; and a migration plan for the year-2090 epoch limit.

---

> ## What made this staff-level
> - **Derived the bit layout from the throughput math** (12 sequence bits → 4,096/ms → ~400× headroom)
>   and the timestamp lifespan (41 bits → ~69 years), rather than reciting "41/10/12" by rote — and
>   framed the split as a *tunable bit budget* with explicit trades.
> - **Compared five schemes on five explicit axes** (size, uniqueness mechanism, sortability,
>   hot-path coordination, SPOF) and traced the *evolution* single-counter → Flickr step/offset →
>   Snowflake, showing the shared principle (disjoint ID spaces, no per-ID coordination).
> - **Tied UUID-v4's weakness to B-tree internals** ([databases §03](../prep/03-databases-deep-dive.md)):
>   random keys fragment the index and kill insert locality — the real reason sortable IDs matter.
> - **Raised clock-skew unprompted** as the make-or-break failure mode, and answered with a layered
>   strategy (NTP slew + wait-small + reject-large) that *refuses to trade away uniqueness*.
> - **Used consensus correctly and sparingly** ([§08](../prep/08-consensus-and-coordination.md)) —
>   ZooKeeper/etcd leases for worker IDs at startup *only*, never on the hot path.
> - **Named the sortability privacy trade** (enumerable IDs leak volume) and split internal vs. external
>   IDs accordingly.

> ## Self-check (answer from memory before the mock)
> - [ ] Draw the 64-bit Snowflake layout: how many bits for sign / timestamp / worker / sequence, and what does each buy?
> - [ ] Why 41 timestamp bits → ~69 years, and why pick a custom (recent) epoch?
> - [ ] How many IDs/ms/node do 12 sequence bits give, and how does the generator handle running out within a ms?
> - [ ] Why is UUID v4 bad as a clustered primary key? (answer in terms of B-tree pages)
> - [ ] Explain the Flickr step/offset trick and how it avoids collisions with no coordination.
> - [ ] What is the clock-going-backwards problem, why does it cause *duplicates*, and what are the three handling strategies?
> - [ ] How are worker IDs assigned without collisions, and what must a node do if it loses its lease?
> - [ ] Name the upside *and* the privacy downside of time-sortable IDs, and how you'd resolve it.
> - [ ] When would you pick ULID / UUID v7 over Snowflake?
