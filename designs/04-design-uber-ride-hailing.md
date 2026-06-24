# Design 04: Uber / Lyft (Ride-Hailing)

> **Why this problem is a staff filter:** ride-hailing looks like a CRUD app with a map on it, and
> juniors design it that way — a `drivers` table, a `trips` table, `SELECT … WHERE distance < r`.
> It collapses the moment you say the numbers out loud: a **write firehose** of millions of location
> pings per second that you must *not* persist, a 2D "nearby" query a B-tree can't serve, a
> rider-to-driver match that must be **exactly-once** (two riders cannot get the same car), and a
> trip-state machine plus payments that need **strong consistency and idempotency** where everything
> else is happily eventual. The staff signal is holding all four of those tensions at once and
> putting each on the right substrate. This walkthrough runs the [Topic 1 framework](../prep/01-framework-and-building-blocks.md)
> end to end and leans hard on the [geospatial](../prep/12-geospatial-systems.md),
> [idempotency/ledger](../prep/10-distributed-transactions-and-idempotency.md), and
> [real-time/push](../prep/15-realtime-and-push.md) building blocks.

---

## 1. Requirements (5 min) — drive this, don't wait

I'll state the buckets out loud and **scope aggressively** so I protect my 45 minutes.

### Functional (the verbs)
- A **driver** goes online and emits a location ping every few seconds; goes offline.
- A **rider** requests a ride from pickup → dropoff; sees nearby cars and an ETA/price estimate.
- The system **matches** the rider to the best nearby available driver and **dispatches** an offer.
- A driver **accepts / declines / times out**; on decline or timeout we fall through to the next.
- A **trip** runs through a lifecycle: requested → matched → en-route-to-pickup → in-progress →
  completed (or cancelled), with both parties getting **real-time updates** (driver location on the
  rider's map, status changes).
- At trip end we **charge the rider and pay the driver**.

> **Scope out loud:** "I'll build request → match → dispatch → trip lifecycle → payment, with the
> location firehose and matching as the deep dives. I'll treat **routing/ETA** as a black-box service
> (I won't build a maps engine), **surge** at the pricing-signal level, and I'll skip ratings,
> driver onboarding, fraud, pool/carpool matching, and multi-stop trips unless you want them. Shout
> if you'd rather I go deep on one of those instead."

### Non-functional (where staff candidates separate)
- **Write-heavy, asymmetrically.** The location ping firehose dwarfs everything else — this single
  fact drives the whole architecture. Matches/trips are orders of magnitude rarer than pings.
- **Freshness over durability for location.** A driver's position is worthless in 5 s; I tolerate a
  few seconds of staleness but **ghost drivers** (offline cars still shown/dispatched) are *not*
  acceptable.
- **Latency:** nearby-drivers query p99 < ~200 ms; end-to-end dispatch (request → driver's phone
  rings) < ~2 s; rider's live map update < ~1 s.
- **Consistency — split it deliberately:**
  - Location store → **eventual / best-effort**, in-memory, self-healing.
  - Match & trip state → **strong**. Exactly one driver per request; a trip can't be in two states.
    Money and the "who's assigned" decision are unforgiving.
- **Availability:** matching must stay up regionally — if a city's matching dies, that city stops
  earning money. Target 99.99% on the dispatch path, degrade gracefully (wider radius, retries).
- **Geographic partitioning is inherent.** A rider in Tokyo never matches a driver in Berlin. This
  is a gift — the problem **shards by geography naturally**, which I'll exploit everywhere.

> **The single derived insight to state now:** "This is two systems wearing one coat. A *write-heavy,
> ephemeral, eventually-consistent* location/matching plane, and a *low-volume, durable,
> strongly-consistent* trip/payment plane. I'll design them separately and connect them at the
> dispatch handoff." That sentence is the whole interview in miniature.

---

## 2. Estimation (3 min) — justify the in-memory firehose decision

The numbers exist to *force* the architecture, per [Topic 2](../prep/02-estimation-and-napkin-math.md).

**Assume:** 5M active drivers globally at peak, 100M riders, ping every **4 s**.

### The write firehose (location pings)

| Quantity | Calc | Result |
|---|---|---|
| Avg location writes/sec | 5M drivers ÷ 4 s | **~1.25M writes/sec** |
| Peak (2–3×, rush hour) | 1.25M × 2.5 | **~3M writes/sec** |
| Bytes per ping | (driverId, lat, lng, ts, heading, speed) | ~40 B |
| Ping ingest bandwidth | 3M × 40 B | ~120 MB/s |
| **Live working set** | only the *latest* per driver: 5M × ~120 B | **~600 MB** |

### The matching / trip side (the other axis)

| Quantity | Calc | Result |
|---|---|---|
| Trips/day | say 50M | 50M |
| Trip request QPS avg | 50M ÷ 86,400 | **~600/sec** |
| Trip request QPS peak | ~2.5× | **~1,500/sec** |
| Nearby-driver read QPS | riders browsing the map, ~10× the requests | **~15k/sec peak** |
| Trip storage | 50M/day × ~2 KB × 5 yr retention | ~180 TB |

> **The asymmetry, said out loud — this is the money sentence:** "Location is **~3M writes/sec** of
> *overwrites where only the latest value matters*, and the entire live set is **~600 MB**. Matching
> is **~1,500 requests/sec** — *three orders of magnitude* smaller. So: persisting every ping would
> be ~3M IOPS of data I throw away in 4 seconds — absurd. The firehose lives **in memory,
> updated-in-place**, and the durable database **never sees a single ping**. It stores trips, which
> arrive 2,000× slower." This derivation *is* the design; everything below follows from it.

The ~600 MB live set is the other half: location data **fits in RAM trivially** — sharded across a
Redis/in-memory cluster it's nothing. The problem was never volume; it's the **write rate** and the
**freshness/expiry**.

---

## 3. API design (3 min)

A handful of endpoints to pin the contract. Auth + rate-limiting at the gateway
(see [Topic 9](../prep/09-api-gateway-loadbalancing-ratelimiting.md)) so I don't re-explain it.

```
# Driver location plane (the firehose) — note: NOT through the normal request gateway,
# pushed over a persistent connection (see §10), shown here as REST for clarity.
POST /drivers/{id}/location   { lat, lng, heading, speed, ts }   -> 200   (heartbeat: refreshes TTL)
POST /drivers/{id}/status     { online | offline | on_trip }     -> 200

# Rider plane
GET  /drivers/nearby?lat=&lng=&radius=&limit=        -> [ {driverId, etaSec, distM} ]   (map dots)
POST /trips/estimate          { pickup, dropoff }    -> { etaSec, fareEstimate, surgeMultiplier }
POST /trips/request           { riderId, pickup, dropoff, productType }
        Idempotency-Key: <client-uuid>               -> { tripId, status: REQUESTED }
GET  /trips/{id}                                     -> { status, driver?, etaSec, route? }
POST /trips/{id}/cancel                              -> 200

# Driver dispatch plane
# (offer is PUSHED to the driver, §10; these are the responses)
POST /offers/{offerId}/accept                        -> { tripId, pickup }   (idempotent)
POST /offers/{offerId}/decline                       -> 200
```

Notes that surface hidden requirements:
- **`POST /trips/request` carries an `Idempotency-Key`** — a rider double-taps "request" or retries
  on a flaky network; we must not create two trips or dispatch two cars. Straight out of the
  [idempotency playbook](../prep/10-distributed-transactions-and-idempotency.md#part-e--idempotency-in-depth).
- **`/estimate` is separate from `/request`** — the price/ETA quote is a cheap read; committing a
  trip is an expensive, stateful write. Separating them mirrors the
  [chat "persist before ACK"](../prep/15-realtime-and-push.md) discipline: don't do the heavy thing
  until the user commits.
- **The offer is pushed, not polled.** The driver's app holds a connection; we push the ring. The
  accept/decline come back over it. (§10.)

---

## 4. Data model (5 min) — access pattern picks the store

The split from §1 shows up concretely as **two stores with opposite characteristics**.

### 4a. Live location store (ephemeral, in-memory)

Access pattern: **write** = overwrite one driver's current cell membership; **read** = "give me the
drivers in this cell + the ring of neighboring cells." This is the
[Uber firehose problem](../prep/12-geospatial-systems.md#d2--uber--lyft--real-time-driver-matching-write-heavy-pings--nearby-queries),
verbatim.

- **Structure:** the world is partitioned into **H3 hexagonal cells** (or a quadtree per region).
  Each cell holds a set of `{driverId, lat, lng, status, ts}`.
- **Backing store:** **Redis GEO** (`GEOADD`/`GEOSEARCH`) per region, or a custom sharded in-memory
  grid. Redis GEO *is* a geohash index riding on a sorted set, and `GEOSEARCH … BYRADIUS` does the
  neighbor-union + filter for you — see
  [the datastore table](../prep/12-geospatial-systems.md#part-g--datastore-options-pick-with-a-reason).
- **TTL per driver key, ~10–15 s.** The ping refreshes it. **No heartbeat → the key expires → the
  driver vanishes from results.** This is how offline/lost-signal drivers self-evict with *no
  explicit "I'm leaving" message* — the same self-cleaning-presence trick as
  [chat presence](../prep/15-realtime-and-push.md#heartbeats--ttl).

### 4b. Durable source of truth (relational, strongly consistent)

| Entity | Key | Notes / access pattern |
|---|---|---|
| `drivers` | `driver_id` | profile, vehicle, current `status`, `region`. **Not** location. |
| `riders` | `rider_id` | profile, payment method token. |
| `trips` | `trip_id` (Snowflake) | the **state machine** (§7). Read by `trip_id`; listed by `rider_id`/`driver_id`. |
| `trip_events` | `(trip_id, seq)` | append-only audit of state transitions (event-sourced trip). |
| `payments` / `ledger` | `entry_id`, indexed by `transfer_id` | **append-only double-entry ledger** (§8). |
| `idempotency_keys` | `key` (unique) | dedup for `/trips/request`, accepts, charges. TTL'd. |

- **SQL here, deliberately.** Trips need **transactions** (assign driver + set status atomically),
  **strong consistency** (no two states), and relational integrity (trip ↔ driver ↔ payment). This
  is the textbook
  [SQL pick from access patterns](../prep/03-databases-deep-dive.md), not reflex.
- **`trip_id` is a Snowflake id** — 64-bit, time-sortable, coordination-free, from
  [Topic 10 Part G](../prep/10-distributed-transactions-and-idempotency.md#part-g--distributed-id-generation-for-idempotency-keys--ordering).
- **Shard the trip DB by `region`/`city`** (geo sharding, §9) — a trip never spans cities, so
  cross-shard transactions are essentially nonexistent. That's the geographic gift again.

---

## 5. High-level design (10 min) — the happy path end to end

```
                          LOCATION / MATCHING PLANE (ephemeral, eventual, write-heavy)
 Driver app ──pings (persistent conn)──▶ Location Ingest (geo-sharded by cell)
                                                │ update-in-place
                                                ▼
                                         In-memory Geo Store  (H3 cells + TTL heartbeat, Redis GEO)
                                                ▲
 Rider app ──nearby / request──▶ API GW ──▶ Matching Service ──queries cell + ring──┘
                                                │ rank by ETA (calls Routing svc)
                                                │ lock candidate driver (idempotent)
                                                ▼
                                         Dispatch Service ──push offer──▶ Driver app (persistent conn)
                                                │ on accept
            ─────────────────────────────────  ▼  ───────────────────────────────────────────
                          TRIP / PAYMENT PLANE (durable, strongly consistent, low-volume)
                                         Trip Service (state machine) ──▶ Trip DB (SQL, geo-sharded)
                                                │ writes trip_events (outbox)
                                                ▼
                                         Kafka (trip events) ──▶ { Payment, Notification, Analytics, Surge }
                                                                          │
 Rider + Driver apps ◀── live updates ── Realtime Gateway ◀── pub/sub backplane ◀──┘
```

**Walk one ride through it out loud:**
1. Drivers stream pings → **Location Ingest**, geo-sharded by cell, overwriting position in the
   **in-memory geo store**, refreshing each driver's TTL.
2. Rider opens app → `GET /drivers/nearby` → **Matching service** queries the rider's H3 cell + ring
   → returns dots for the map (cheap read).
3. Rider taps request (with idempotency key) → Matching finds candidates in the cell+ring, filters
   by true distance, ranks by **road-network ETA** (Routing service), **locks** the top driver, and
   hands an offer to **Dispatch**.
4. Dispatch **pushes the offer** to that driver's phone. Accept → confirm; decline/timeout → release
   the lock, fall through to the next candidate.
5. On accept, **Trip service** creates the trip (`MATCHED`) in the durable SQL DB and starts the
   **state machine**. It emits trip events via the **outbox** to Kafka.
6. **Payment**, **Notification**, **Surge**, **Analytics** consume those events. Both apps get
   **live updates** (driver moving toward pickup, status changes) over the **realtime gateway**.
7. Trip completes → Trip service finalizes state → emits `TRIP_COMPLETED` → Payment charges the
   rider and credits the driver against the **ledger**, idempotently.

Keep it this simple first; the next section is where the round is won.

---

## 6. Deep dives (15 min) — propose the hard parts

> **Open with:** "The four interesting problems are (1) surviving the location firehose, (2) the
> nearby-driver spatial query, (3) **exactly-once** dispatch — no double-booking, and (4) the trip
> state machine + payment correctness. Can I go deep on the firehose and dispatch — they're the
> ones most people hand-wave?"

### 6.1 Deep dive — the location ingestion firehose (write-heavy)

This is the asymmetry from §2 made real. ~3M writes/sec of overwrites; never persist them.

**Why not a disk DB?** Every ping would be a write + a spatial-index update; a B-tree/LSM index would
churn constantly on data we discard in 4 s. ~3M IOPS for garbage. Out of the question. Instead:

- **In-memory, update-in-place.** Overwrite the driver's last position; never append history. The
  live set is ~600 MB (§2) — RAM is not the constraint, write rate is.
- **TTL / heartbeat for liveness.** Each ping `SET`s the driver key with a ~10–15 s TTL. A driver who
  loses signal or quits simply **expires out of the index** — no ghost drivers, no explicit teardown.
  *The ping is the heartbeat.* (Same mechanism as
  [presence TTL](../prep/15-realtime-and-push.md#heartbeats--ttl).)
- **Tame the rate further:**
  - **Throttle/adapt** ping frequency: 4 s when idle, faster when on a trip near pickup, slower when
    parked. The client is dumb about *what* matters; the server tells it the cadence.
  - **Batch** at the edge if a driver app buffers a couple of pings on a flaky link.
  - **Only re-bucket on actual cell change**, not every ping. If the driver hasn't crossed an H3 cell
    boundary, we update lat/lng in place but skip the (more expensive) cell-membership move. This is
    the [granularity write-cost lever](../prep/12-geospatial-systems.md#part-e--the-read-vs-write-tradeoff-in-spatial-indexing).
- **Shard the location service by geo cell** (§9), so a region's writes land on one node — locality,
  no cross-talk, and the firehose splits cleanly across the cluster. A 3M writes/sec global firehose
  is, per city, a very manageable number.

> **Tradeoff sentence:** "I keep the firehose *entirely out* of the durable DB. Location is in-memory,
> overwritten in place, geo-sharded, with a TTL heartbeat so offline drivers self-evict. I trade
> durability (which I don't need — a lost ping is replaced in 4 s) for the ability to absorb millions
> of writes a second on commodity RAM."

**Ingestion path detail:** drivers hold a **persistent connection** (WebSocket/gRPC stream) to a
location-ingest gateway rather than hammering a request gateway with 3M HTTP POSTs/sec — see §10.
The gateway is a **dumb edge** ([dumb-edge/smart-core](../prep/15-realtime-and-push.md#the-architecture-dumb-edge-smart-core)):
terminate the connection, parse the ping, write to the in-memory cell store for its shard. No
business logic, so it scales purely on connection + write count.

### 6.2 Deep dive — the nearby-driver spatial query

Why the naive `WHERE lat BETWEEN … AND lng BETWEEN …` fails is a
[dimensionality problem](../prep/12-geospatial-systems.md#why-the-naive-query-doesnt-scale): a B-tree
orders one axis, so you read a long thin latitude strip and scan the second dimension. The fix is a
**spatial index that turns 2D proximity into a cell lookup.**

**Index choice — H3 (hexagons), and I can defend it:**
- Data is **moving constantly** → I want an **in-memory, density-adaptive** structure, not a
  disk-indexed geohash *column* I'd rewrite millions of times/sec.
- **Hexagons have 6 equidistant edge-neighbors**, vs a square's 4 edge + 4 corner neighbors at
  different distances. For "expand the search ring" and for surge demand-smoothing the uniform
  adjacency makes the math clean — exactly
  [why Uber built H3](../prep/12-geospatial-systems.md#b4--google-s2-and-uber-h3--the-modern-cell-systems).
  (Geohash on Redis GEO is a perfectly defensible alternative for the in-memory store; I'd name it as
  the simpler option and H3 as the one tuned for movement/surge.)

**The query (coarse-then-fine):**
1. Compute the rider's H3 cell at a resolution where a cell ≈ a few city blocks.
2. **Read the center cell + its ring of neighbors** (k-ring). *Not* just one cell — a driver
   physically 50 m away can sit in the adjacent hex; querying one cell silently misses them. This is
   the [boundary problem](../prep/12-geospatial-systems.md#part-c--a-radius-query-with-geohash-worked),
   solved by the neighbor ring.
3. **Filter the candidates by true haversine distance** to carve the circle out of the block of
   hexes. Cells are the coarse filter; haversine is the fine filter.
4. Keep only `status = available` drivers, then rank.

**Granularity is a dial** ([Topic 12 Part E](../prep/12-geospatial-systems.md#part-e--the-read-vs-write-tradeoff-in-spatial-indexing)):
fine cells → cheap precise reads but a moving car re-buckets constantly (write cost); coarse cells →
cheap writes but each read scans many drivers. I pick a resolution where a car doesn't cross a
boundary every ping, and recover read precision with the haversine post-filter — *same index family,
tuned for the write-heavy side.*

**Ranking the candidates:** straight-line distance is only the *candidate* filter. Final ranking uses
**road-network ETA** (§6.5) — a river or a one-way street ruins crow-flies ranking. Secondary signals:
driver heading (already pointed toward pickup), acceptance-rate, vehicle type, and a fairness/utilization
term so we don't starve drivers.

### 6.3 Deep dive — dispatch: matching exactly once (the correctness crux)

> **This is the part juniors skip.** "Match rider to nearby driver" hides the hard invariant:
> **two riders must never be dispatched the same car, and one rider must never hold two cars.** This
> is an [exactly-once-effect](../prep/10-distributed-transactions-and-idempotency.md#part-f--exactly-once-demystified)
> problem on a contended resource (the driver).

**The request → offer → accept flow with a lock:**

```
Rider requests (Idempotency-Key K)
  └─ dedup on K  → if replay, return the existing trip (no double dispatch)
  └─ query cell+ring → rank candidates [d1, d2, d3, …]
  OFFER LOOP:
    try acquire lock on d1 (status AVAILABLE → OFFERED, atomic CAS, lease TTL ~15s)
      ├─ won lock  → push offer to d1's phone, await accept/decline/timeout
      │     ├─ ACCEPT  → CAS d1 OFFERED→ON_TRIP, create trip MATCHED, release others. DONE.
      │     ├─ DECLINE → release d1 (OFFERED→AVAILABLE), try d2
      │     └─ TIMEOUT (no answer in ~15s) → lease expires, release d1, try d2
      └─ lost lock (someone else offered d1) → skip to d2
```

The mechanics, named precisely:
- **The driver is a lockable resource with a state field** — `AVAILABLE → OFFERED → ON_TRIP`. The
  transition `AVAILABLE → OFFERED` is a **compare-and-set** (atomic). This is a
  **[semantic lock](../prep/10-distributed-transactions-and-idempotency.md#semantic-locks-regaining-a-little-isolation)**:
  an application-state flag, not a DB row lock, that says "don't offer this driver to anyone else."
- **The lock has a lease / TTL (~15 s).** If the dispatcher crashes mid-offer, the lease expires and
  the driver returns to `AVAILABLE` — no stuck-forever cars. This is exactly why we avoid raw
  long-held locks; the lease is self-healing. (Where to keep it: a fast strongly-consistent store —
  Redis with a Lua CAS, or a coordination service; see
  [distributed locks](../prep/08-consensus-and-coordination.md). For the hardest correctness, the
  driver's authoritative state lives in the trip DB and the lock is a row CAS within a transaction.)
- **Idempotency end to end:** `/trips/request` dedups on the client key so a retry returns the
  *existing* dispatch, not a new one. `/offers/{id}/accept` is idempotent — a driver double-taps
  accept or retries on flaky signal; the second accept finds the trip already `MATCHED` to them and
  returns the same result (a conditional `UPDATE … WHERE status='OFFERED'` updates zero rows on
  replay). Same [Stripe idempotency-key race](../prep/10-distributed-transactions-and-idempotency.md#idempotency-keys-the-stripe-model)
  discipline.
- **Declines / timeouts** are normal flow, not errors: release the lease, advance to the next
  candidate. After N misses or T seconds, **widen the radius / loosen filters** and re-query, then
  finally tell the rider "no cars available."
- **Double-offer to fill faster?** Some systems offer to the top 2–3 simultaneously and take the
  first accept (then cancel the losers). This trades a small double-dispatch window for lower
  time-to-match; the CAS on accept still guarantees only one wins the *trip*, and the losers' leases
  release. I'd call this out as a tunable.

> **Tradeoff sentence:** "Dispatch is exactly-once on a contended resource. I model the driver as a
> state machine with a **leased semantic lock** — `AVAILABLE→OFFERED→ON_TRIP` via compare-and-set —
> so two riders can't grab the same car, the lease auto-releases on crash so no car gets stuck, and
> idempotency keys on request and accept make retries safe. I get strong consistency exactly where I
> need it and nowhere else."

### 6.4 Deep dive — trip lifecycle, state machine & consistency

A trip is a **state machine**, and *this* is where I switch from eventual to **strong consistency**.

```
REQUESTED ─▶ MATCHED ─▶ EN_ROUTE_TO_PICKUP ─▶ ARRIVED ─▶ IN_PROGRESS ─▶ COMPLETED
     │           │              │                │            │
     └───────────┴──────────────┴────────────────┴──── CANCELLED (by rider/driver/system)
```

**Why strong consistency here (say it explicitly):** a trip in two states, or two drivers thinking
they own one trip, or a trip that's `COMPLETED` for billing but `IN_PROGRESS` for the driver, is a
*correctness* failure that touches money and safety. Unlike a stale map dot (who cares, it refreshes),
a wrong trip state is unacceptable. So:
- **Transitions are conditional updates inside a DB transaction:**
  `UPDATE trips SET status='IN_PROGRESS' WHERE trip_id=? AND status='ARRIVED'`. Updating zero rows
  means an illegal/duplicate transition — reject it. This makes transitions **idempotent and
  ordered**, defeating out-of-order or replayed events from a flaky driver connection.
- **Single-writer per trip.** All transitions go through the Trip service, which owns the row. Since
  trips are **geo-sharded by city**, the authoritative trip lives on one shard with normal local-ACID
  semantics — no distributed transaction needed for the trip itself.
- **Event-sourced audit:** every transition also appends to `trip_events` (immutable), giving a
  replayable history for disputes, analytics, and recovery.

**Emitting events without the dual-write bug.** Downstream (payment, notifications, surge, analytics)
must learn about transitions. Writing the trip row *and then* publishing to Kafka is the
[dual-write anti-pattern](../prep/10-distributed-transactions-and-idempotency.md#the-dual-write-problem-the-bug-you-will-be-asked-to-find) —
a crash between them diverges the DB and the stream. **Fix: the outbox pattern** — write the trip
change and an outbox row in the *same local transaction*, and a relay (CDC/Debezium tailing the WAL)
ships them to Kafka **at-least-once**. Consumers dedup (inbox pattern). This is the
[outbox + CDC plumbing](../prep/10-distributed-transactions-and-idempotency.md#part-d--outbox--cdc-atomic-update-db-and-publish-event).

**Surge pricing (high level, as promised):** surge is a **demand/supply ratio per H3 cell**. The
matching plane already knows, per cell, how many `available` drivers and how many open requests exist;
a streaming job ([Topic 18](../prep/18-batch-and-stream-processing.md)) computes a multiplier per cell
every few seconds and publishes it. `/estimate` reads the multiplier for the pickup cell. Hexagons'
uniform neighbors make the smoothing/gradient across cells clean — the *other* reason H3 (§6.2). Surge
is a **read-side pricing signal**, not on the latency-critical dispatch path; it can be eventually
consistent (a few seconds stale is fine).

### 6.5 Deep dive — ETA / routing (black-box it, but show you know the shape)

> **Say this:** "I won't build a maps engine in 45 minutes, but here's the shape so you know I'm not
> hand-waving."

- The road network is a **weighted directed graph**: nodes = intersections, edges = road segments,
  edge weight = current traversal time (varies by time-of-day + live traffic).
- Naive Dijkstra/A* per request across a continent-sized graph is too slow. Production routing
  **precomputes**: partition the graph into regions, precompute shortest paths between region
  **boundary nodes** (contraction hierarchies / customizable route planning), so a live query stitches
  short local searches to a precomputed long-haul skeleton → sub-100 ms routes.
- **Live traffic** updates edge weights from the very location firehose we're already collecting —
  aggregated driver speeds per segment feed back as edge weights. Nice closed loop to mention.
- For **matching ranking** I don't even need the full route — a fast **ETA estimate** (often an ML
  model over distance + historical segment speeds) is enough to rank candidates; the precise route is
  computed once after dispatch and re-computed as conditions change.

It's a separate service with its own cache (popular OD pairs, precomputed segments). Matching calls it
read-only.

### 6.6 Deep dive — payments at trip end (ledger + idempotency)

Money is the highest-stakes correctness problem, so I lean entirely on
[Topic 10 Part H](../prep/10-distributed-transactions-and-idempotency.md#part-h--handling-money-ledgers-double-entry-reconciliation).

On `TRIP_COMPLETED` (consumed from Kafka), the Payment service:
- Computes fare = base + distance + time + surge multiplier (captured at request time, not now).
- **Authorize at trip start, capture at trip end.** We pre-authorize the rider's card when the trip
  begins so we know funds exist, and **capture** at completion. This mirrors the saga
  [pivot ordering](../prep/10-distributed-transactions-and-idempotency.md#compensating-transactions) —
  don't actually move money until the irreversible step (the ride happened).
- **Append-only double-entry ledger:** the fare writes paired entries summing to zero — debit the
  rider, credit the driver (minus the platform's commission, a third entry). Tagged with one
  `transfer_id` (= a function of `trip_id`). The balance is a **derived sum**, never a mutable column.
- **Posting is naturally idempotent:** the unique `transfer_id` means a redelivered `TRIP_COMPLETED`
  event collides and is a no-op — **no double charge**. This is why append-only + unique key beats
  `balance -= fare` (which is not idempotent and races).
- **Reconciliation job** recomputes balances from the ledger and diffs against the external payment
  processor (Stripe/Adyen) on a schedule; drift = a bug → alert. This async authoritative re-check is
  *why* the payment pipeline can be eventually consistent off the Kafka stream rather than a
  synchronous 2PC with the trip.
- The external charge call carries a **Stripe-style idempotency key** so processor retries don't
  double-charge either.

> **Tradeoff sentence:** "Money rides an append-only double-entry ledger, posted idempotently via a
> `transfer_id` derived from the trip id, consumed at-least-once off the trip-completed event, with a
> reconciliation job catching drift. Authorize-then-capture keeps me from moving real money until the
> ride is irreversibly done. I never need a synchronous distributed transaction with the trip — the
> ledger's idempotency plus reconciliation buys me the correctness asynchronously."

---

## 7. Real-time updates to rider + driver (notifications)

Both apps need server-push: the rider watches the driver's car move toward pickup; the driver gets
the offer and trip updates. This is the
[real-time/push toolkit](../prep/15-realtime-and-push.md).

- **Transport:** persistent connection while the app is foregrounded — **WebSocket** for the
  bidirectional driver app (offers in, accept out), **SSE/WebSocket** for the rider's live map. When
  the app is **backgrounded**, the OS kills your socket → fall back to **APNs/FCM push**
  ("Your driver is arriving"). Production uses
  [all three](../prep/15-realtime-and-push.md#part-a--delivery-mechanisms-pick-the-cheapest-thing-that-meets-the-requirement).
- **Dumb-edge / smart-core:** a **realtime gateway fleet** holds the sockets (no business logic, rare
  deploys); stateless services publish "send event E to user U" to a **pub/sub backplane** and the
  gateway holding U delivers it. The
  [connection registry](../prep/15-realtime-and-push.md#the-session--connection-registry-the-crux)
  (`userId → gatewayId`) or a `user:{id}` pub/sub channel routes it. **Redis pub/sub for the
  low-latency last-hop push, Kafka for the durable event log.**
- **The rider's live-car-position stream** is the highest-volume real-time path: the assigned driver's
  pings are forwarded to the one rider watching. Crucially this is a **targeted 1:1 forward**, not a
  fan-out — only the matched rider subscribes, so it's cheap. Coalesce to the latest position
  ([backpressure](../prep/15-realtime-and-push.md#backpressure-on-slow-clients)); a stale dot is fine.
- **Reconnect + resume:** mobile sockets die constantly. The client tracks a **last-seen cursor** per
  trip/event stream and replays from the durable trip-event log on reconnect — at-least-once +
  idempotent client dedup by event id. Trip-state pushes are ordered by the state machine's sequence.

---

## 8. Sharding location data by geography (and the hot-city problem)

The geographic gift, made concrete (general principles: [Topic 4](../prep/04-sharding-and-partitioning.md)).

- **Shard the location store and ingest fleet by geo region/cell.** A driver's pings and a rider's
  nearby query for the same area land on the **same shard** — locality, no cross-node fan-out for the
  common case. A city is independent: 3M global writes/sec becomes a comfortable per-shard rate.
- **Cross-cell queries at boundaries:** a rider near a region edge whose neighbor ring spills into an
  adjacent shard needs a cross-shard scatter to 2 shards — rare, bounded to the ring, acceptable.
  Drivers **hand off** between shards as they cross region boundaries (remove from old, add to new).

> **The hot-city problem (name it before asked):** geography is wildly non-uniform — **Manhattan at
> rush hour has 1000× the drivers and requests of rural Montana.** A uniform grid puts a brutal hot
> shard on downtown and idle shards everywhere else.

Mitigations:
- **Uneven shard boundaries / finer sub-sharding in dense areas.** Don't shard on a fixed cell size;
  split dense cells finer so each shard owns a roughly *equal load*, not equal *area*. (H3's
  hierarchy lets a hot cell subdivide into children owned by separate nodes.)
- **The cellular sharding already localizes the spike** — a Manhattan surge stresses Manhattan's
  shards, not the planet. Scale those shards independently (more replicas/nodes for hot regions).
- **Don't cache the location plane** — freshness is the whole point; caching stale positions defeats
  it. Scale the owning shard instead. (Caching wins are on the *static* Yelp-style problem, the
  [opposite design](../prep/12-geospatial-systems.md#d1--proximity-service--yelp-nearby-restaurants-read-heavy-static-data).)
- Special-case **mega-events** (stadium emptying, NYE) with predictive pre-positioning and pre-scaled
  capacity.

---

## 9. Wrap-up (3 min) — bottlenecks, failure modes, SPOFs

**Remaining bottlenecks**
- **Hot cells** at peak in dense cities — addressed by finer sub-sharding + independent scaling, but
  it's the perennial tuning problem.
- **Dispatch contention** in a hot cell — many riders competing for few cars means lock contention on
  drivers; the CAS keeps it *correct* but time-to-match degrades; mitigate by widening radius and
  surge (which pulls in more supply).

**Failure modes (name them unprompted, per [Topic 13](../prep/13-resilience-and-failure-handling.md))**
- **A region's matching service dies.** This is the scary one — that city stops earning. Mitigations:
  matching is **stateless** (state is in the in-memory geo store + trip DB), so it's behind a load
  balancer with health checks and **fails over** to healthy instances; run **N+ replicas per region**;
  the in-memory location store is **self-healing** — drivers re-ping within seconds and rebuild the
  cells from scratch (TTL means there's no stale state to reconcile). Worst case, degrade to a wider,
  coarser match.
- **In-memory geo store node dies.** Drivers in that shard re-ping and rebuild within ~one TTL window;
  replicate hot shards (primary + replica) so reads survive a node loss without even that gap.
- **The Kafka trip-event bus dies.** It's the durable backbone for payments/notifications. Use a
  replicated Kafka; the **outbox rows persist in the trip DB** regardless, so the relay replays from
  the WAL once Kafka recovers — **no event lost**, just delayed. Payment is idempotent so replay is
  safe.
- **Payment processor down.** Trips still complete (state machine doesn't block on payment); charges
  queue and retry with backoff + DLQ + circuit breaker; reconciliation catches anything that slipped.
- **GPS jitter / half-open connections.** Smoothing + don't re-bucket on sub-cell noise; TTL expiry
  handles the half-open "driver vanished but TCP didn't notice" case.

**Single points of failure & redundancy**
- The **trip DB** per shard is a single-leader SQL store → leader-follower replication + automated
  failover; geo-sharding limits blast radius to one city.
- The **lock/coordination store** for dispatch → must be HA (replicated Redis or a consensus-backed
  store, [Topic 8](../prep/08-consensus-and-coordination.md)); leases make a lock-store blip
  self-correct rather than wedge.
- **No global SPOF by design** — the system is regionally partitioned end to end, so a single region's
  total failure can't take down the planet. Multi-region failover for the durable plane is the
  with-more-time item.

**With more time:** predictive pre-positioning (forecast demand per cell, nudge drivers), batched/pool
matching (multiple riders one car — a harder optimization problem), driver incentives, fraud detection,
and full multi-region active-active for the trip/payment plane.

---

## What made this staff-level

- **Derived the whole architecture from one estimation insight** — the ~3M-writes/sec vs ~1,500-req/sec
  asymmetry — rather than reciting a remembered Uber diagram. The numbers *forced* the in-memory
  firehose decision.
- **Split consistency deliberately by plane** and *said why*: eventual/ephemeral for location, strong
  for trip-state and money. Knowing where you *don't* need strong consistency is the senior move.
- **Treated dispatch as an exactly-once problem on a contended resource** — leased semantic lock + CAS
  + idempotency keys — instead of "match nearby driver" hand-waving. This is the part that separates
  candidates.
- **Named the dual-write bug** in the trip→downstream path and fixed it with outbox/CDC, rather than
  drawing an arrow to Kafka and moving on.
- **Put money on an append-only idempotent ledger with reconciliation**, and justified *why* that lets
  payments be async instead of a synchronous distributed transaction.
- **Picked H3 with a real tradeoff** (uniform hexagon neighbors for ring queries + surge smoothing),
  acknowledged geohash/Redis-GEO as the simpler alternative, and tied granularity to the read/write
  dial.
- **Named the hot-city problem and failure modes** (a region's matching dying, the firehose store
  rebuilding via TTL self-healing) before being asked, and showed there's no global SPOF.

---

### Self-check before the mock (answer these from memory)
- [ ] State the write asymmetry: location pings/sec vs trip requests/sec, and what it forces.
- [ ] Why does the durable DB never see a location ping? What replaces persistence (TTL heartbeat)?
- [ ] Why does a B-tree fail the nearby query, and what's the cell + ring + haversine recipe?
- [ ] Why H3 over geohash here, and how does cell granularity trade read cost vs write cost?
- [ ] Walk the request → offer → accept/decline/timeout flow. Where exactly is the exactly-once
      guarantee, and what makes the lock self-healing?
- [ ] Which two operations carry idempotency keys and why (request, accept)?
- [ ] Why does trip state need strong consistency when location doesn't? How is a transition made
      idempotent and ordered?
- [ ] Where's the dual-write bug in the trip→payment path, and how does outbox/CDC fix it?
- [ ] How is a charge made idempotent, and why does that let payment be eventually consistent?
- [ ] How do you shard location by geography, and what's the hot-city problem + mitigations?
- [ ] What happens when a region's matching service dies, and why is the location store self-healing?
