# Topic 12: Geospatial Systems — "Nearby", Maps & Real-Time Location

> **Why this topic earns its own doc:** "Find things near me" looks trivial and is a trap. The naive
> answer (`WHERE lat BETWEEN … AND lng BETWEEN …`) passes the eye test and fails at scale for a
> reason most candidates can't articulate. The staff signal here is (a) knowing *why* a 2D range
> scan can't use a 1D index well, (b) being fluent in the spatial-index zoo (geohash / quadtree /
> R-tree / S2 / H3) and able to *pick one with a tradeoff sentence*, and (c) splitting the
> read-heavy "Yelp nearby" problem from the write-heavy "Uber driver firehose" problem — they look
> identical and want opposite designs.

---

## Part A — The core problem

The question is always some flavor of: **"Given a point and a radius (or a bounding box), return the
entities inside it, ranked by distance."** Restaurants near me, drivers near a rider, friends near a
venue, geofence membership.

### Why the naive query doesn't scale

```sql
SELECT * FROM places
WHERE lat BETWEEN :lat - d AND :lat + d
  AND lng BETWEEN :lng - d AND :lng + d;
-- then compute true distance in app code and filter to the circle
```

This is correct. It is also slow at scale, and you must be able to say *why* in the room:

- **A B-tree index is one-dimensional.** An index on `lat` can satisfy the latitude range
  efficiently, but then you have *every* row in that latitude band — the entire equator's worth of
  longitudes — and you filter the second dimension by scanning. A composite `(lat, lng)` index only
  helps the *first* column's range; the second column's range can't be seek-driven once the first is
  a range, not an equality. So you read a long thin strip, not a box.
- **Two independent single-column indexes don't compose into a 2D box** either — the planner picks
  one, or does a bitmap-AND of two large rowsets. Both touch far more rows than the answer set.
- **The answer set is tiny relative to the scan.** A 1 km query in Manhattan might return 50 rows
  after touching tens of thousands. That ratio gets worse as the table grows, because the strip's
  length grows with data density, not with the query radius.

> **Say this in the room:** "The problem is *dimensionality*. A standard index orders data on one
> axis; 'nearby' is inherently 2D. I need a spatial index that maps 2D proximity down to something a
> 1D index *or* a tree can answer with a bounded scan — that's geohash, quadtree, R-tree, or a cell
> system like S2/H3. The whole game is **converting 2D proximity into a 1D-prefix or tree lookup.**"

The unifying idea behind every technique below: **impose a hierarchical grid on the surface of the
Earth so that "close in space" becomes "close in the index" (shared prefix, same tree node, same
cell id).**

---

## Part B — Spatial indexing techniques (in depth)

### B.1 Geohash

A **geohash** encodes a (lat, lng) pair into a short base32 string where **a shared prefix means
spatial proximity**. It is the workhorse because it turns 2D into a *string prefix*, which any
ordinary B-tree / sorted index (or Redis sorted set) can range-scan.

**How the encoding works — the part interviewers probe:**

1. Take two ranges: latitude `[-90, 90]`, longitude `[-180, 180]`.
2. **Bisect repeatedly.** For longitude: is the point in the left half `[-180, 0]` or right half
   `[0, 180]`? Left → bit `0`, right → bit `1`. Then recurse into the chosen half. Same for
   latitude.
3. **Interleave the bits**, longitude first by convention: `lng, lat, lng, lat, …`. Each pass adds
   one bit per axis and *halves the cell* in that dimension.
4. Group the resulting bitstream into **5-bit chunks**, and map each chunk (0–31) through **base32**
   (`0-9` + `b,c,d,e,f,g,h,j,k,m,n,p,q,r,s,t,u,v,w,x,y,z` — note: no `a,i,l,o` to avoid
   ambiguity). Each base32 character = 5 bits = one more level of subdivision.

**Worked micro-example (intuition, not full precision):** for San Francisco (~37.77, -122.41):
- lng -122.41 is in left half of `[-180,180]` → `0`; lat 37.77 is in right half of `[-90,90]` → `1`;
  next lng bit, next lat bit… interleave to `01101 11111 11000 …` and base32-encode to `9q8yy…`.
- Nearby points in SF share the `9q8yy` prefix. Points in LA share `9q5`. Points in NY share `dr5`.

**Precision ↔ prefix length** (memorize the rough ladder; exact numbers vary by latitude):

| Geohash length | Cell width × height (approx) |
|---|---|
| 4 | ~39 km × 20 km |
| 5 | ~5 km × 5 km |
| 6 | ~1.2 km × 0.6 km |
| 7 | ~150 m × 150 m |
| 8 | ~38 m × 19 m |
| 9 | ~5 m × 5 m |

**The query:** to find everything near a point, compute the point's geohash at the precision whose
cell ≈ your radius, then do a **prefix range scan**: `WHERE geohash BETWEEN '9q8yy' AND '9q8yz'`
(or `LIKE '9q8yy%'`). One index seek, bounded scan. This is the magic — proximity became a prefix.

> **The edge-case that separates seniors from juniors — the boundary problem.**
> Geohash cells are rectangles, and **two points can be physically adjacent but sit in different
> cells with *no shared prefix*** (they straddle a major bisection line — e.g. one just east of the
> prime longitude split, one just west). A naive single-prefix query *misses neighbors right across
> the border* and also *misses points that are inside your radius but technically in the next cell.*

**Fix: query the center cell + its 8 neighbors.** Compute the 8 surrounding cells (N, NE, E, SE, S,
SW, W, NW) using a geohash-neighbor algorithm, union the 9 prefix scans, then **filter by true
great-circle (haversine) distance** to get the exact circle out of the 3×3 block of squares. Almost
every geohash library ships a `neighbors()` function precisely for this. (See the worked radius
query in Part C.)

**Geohash properties to recite:**
- **Pros:** dead simple; works on *any* sorted store (RDBMS B-tree, Redis ZSET, even a sorted file);
  prefix = proximity; cheap to compute client-side; great for **mostly-static, read-heavy** data.
- **Cons:** boundary problem (the 8-neighbor dance); **non-uniform cell sizes** (cells shrink toward
  the poles because longitude degrees converge); fixed grid doesn't adapt to density — a precision-7
  cell holds 3 restaurants in Wyoming and 3,000 in Manhattan; abrupt precision jumps between levels.

### B.2 Quadtree

A **quadtree** recursively subdivides space into **four quadrants (NW, NE, SW, SE)**, but — and this
is the point — **only where there's data.** A node splits into 4 children when it exceeds a capacity
threshold (say 100 points). Sparse regions stay shallow; dense regions get deep.

- **Build:** insert points; when a leaf exceeds capacity, split into 4 and redistribute.
- **Query:** descend from the root, pruning any quadrant that doesn't intersect your bounding
  box/radius; collect points from the leaves that do.
- **Strength — non-uniform density.** Unlike geohash's fixed grid, the tree is *adaptive*: Manhattan
  becomes deep, the ocean stays one node. Leaf cells hold a roughly **constant number of entities**,
  which is exactly what you want for "return ~K nearest" and for balancing per-cell work.
- **Cons:** it's an **in-memory tree** (you build/hold it in process), so it's harder to shard and
  persist than a flat geohash column; **updates** (a point moving across a boundary) can trigger
  splits/merges, which matters for moving objects; rebuilding/rebalancing under a write firehose is
  real work. Classic for **Uber/Lyft-style** dispatch where you hold a live tree in memory per
  region.

### B.3 R-tree

An **R-tree** groups nearby objects into **minimum bounding rectangles (MBRs)** and nests those MBRs
into a balanced tree (think B-tree for rectangles). Built for indexing **shapes with extent**, not
just points: roads, polygons, building footprints.

- **Query:** descend, pruning any MBR that doesn't intersect the query region.
- **Strength:** handles **non-point geometries** and arbitrary shapes; great for "does this polygon
  intersect that one", containment, line/area data — the natural fit for **map features**.
- **Cons:** MBRs can **overlap**, so a query may descend multiple branches (worse pruning than a
  disjoint grid); insert/split heuristics are fiddly; harder to distribute. **This is what PostGIS
  uses under the hood (GiST = generalized R-tree).** If the prompt has polygons or "is point inside
  region", reach for R-tree/PostGIS.

### B.4 Google S2 and Uber H3 — the modern cell systems

Both project the Earth's surface onto a **hierarchical cell grid with stable integer cell IDs**, and
both are *much* better than geohash at neighbor uniformity. The difference is the cell shape.

- **Google S2:** projects the sphere onto the **6 faces of a cube**, then recursively subdivides each
  face into **squares** via a Hilbert space-filling curve. The Hilbert curve gives the killer
  property: **cell IDs that are numerically close are spatially close** (better locality than
  geohash's Z-order, fewer nasty jumps), and a cell id is a single 64-bit integer. 30 levels, cell
  sizes from whole-face down to ~1 cm². Used by Google Maps, and famously by Uber's *original* geo
  index, Foursquare, MongoDB's geo.
- **Uber H3:** tiles the world in **hexagons** (12 pentagons exist as unavoidable artifacts of
  tiling a sphere). 16 resolutions. Each hex has a 64-bit id.

> **Why hexagons beat squares — the one-liner that lands.**
> "A square cell has **two kinds of neighbors**: 4 edge-neighbors (distance = side length) and 4
> corner-neighbors (distance = side×√2). The distance from a cell's center to its neighbors is
> *non-uniform*. A **hexagon has 6 neighbors, all sharing an edge, all equidistant** from the
> center. For movement, flow, ride-demand smoothing, and 'expanding rings of nearby cells', that
> uniform adjacency makes the math clean and the gradients smooth — which is exactly why Uber built
> H3 for surge/dispatch." Hexagons also tile with the *lowest perimeter-to-area ratio* of the
> regular tilings (least edge distortion).
>
> The catch hexagons can't escape: **hexagons don't subdivide cleanly** into smaller hexagons (7
> child hexes don't perfectly tile a parent), so H3's parent/child hierarchy is *approximate* — fine
> for most uses, but S2's squares nest exactly.

**Compare them all:**

| Technique | Shape / structure | Adapts to density? | Neighbor uniformity | Persists in a plain sorted store? | Best for |
|---|---|---|---|---|---|
| **Geohash** | Fixed grid, base32 prefix | No (fixed grid) | Poor (boundary problem, pole skew) | **Yes** (string prefix) | Simple read-heavy "nearby"; Redis GEO |
| **Quadtree** | Recursive 4-way, adaptive | **Yes** | OK | No (in-memory tree) | Non-uniform density, in-mem dispatch |
| **R-tree** | Nested MBRs (shapes) | Yes | n/a (shapes) | Via GiST (PostGIS) | Polygons, containment, map features |
| **S2** | Cube faces → squares, Hilbert | Hierarchical levels | Good (Hilbert locality) | Yes (int cell id) | Google-scale point indexing, coverings |
| **H3** | Hexagons | Hierarchical resolutions | **Best (6 equidistant)** | Yes (int cell id) | Movement/flow, surge, Uber dispatch |

> **Decision heuristic for the room:** Static points + want it on the DB/Redis you already have →
> **geohash**. Non-uniform density, in-memory, moving objects → **quadtree**. Polygons / "inside
> region" → **R-tree / PostGIS**. Planet scale with clean neighbor math / flow / surge → **S2 or H3**.

---

## Part C — A radius query with geohash, worked

**Goal:** find restaurants within **1 km** of (37.7749, -122.4194), San Francisco.

1. **Pick precision from radius.** 1 km radius → a precision-6 cell is ~1.2 km × 0.6 km, comparable
   to the radius. (Rule of thumb: choose the precision whose cell is *just larger* than the diameter
   so the 3×3 block of cells comfortably covers the circle. Going too coarse scans too much; too fine
   means the 8 neighbors don't cover the radius and you need a ring of 24.)
2. **Geohash the center** at precision 6 → `9q8yyk` (illustrative).
3. **Compute the 8 neighbors** with the library's `neighbors()`:
   `9q8yym, 9q8yyt, 9q8yys, 9q8yye, 9q8yy7, 9q8yy5, 9q8yyh, 9q8yyj` (N, NE, E, SE, S, SW, W, NW).
4. **Range-scan all 9 prefixes.** In SQL: `WHERE geohash LIKE '9q8yyk%' OR … (×9)`; in Redis the GEO
   commands do this for you; in a generic store, 9 bounded index seeks.
5. **Filter by true distance.** Compute **haversine distance** from the center to each candidate and
   keep those ≤ 1000 m. This carves the **circle out of the 3×3 square block** and silences the
   boundary problem — any neighbor that's physically near but in an adjacent cell is now included,
   and any corner of the block that's beyond 1 km is dropped.
6. **Rank** by distance, apply business filters (open now, rating), paginate, return.

> **Say this:** "The 8-neighbor union plus a haversine post-filter is the standard geohash radius
> recipe. The cells are the *coarse* filter that lets the index do the heavy lifting; the haversine
> is the *fine* filter for exactness. Without the neighbors I silently miss results across cell
> borders." That sentence alone signals you've actually built this.

---

## Part D — Designing the two classic problems (they pull in opposite directions)

The single most important framing move: **is the data mostly static and read-heavy (Yelp), or is it
a write firehose of moving objects (Uber)?** Same "nearby" query, opposite designs.

### D.1 Proximity service — Yelp "nearby restaurants" (READ-heavy, static data)

**Shape:** millions of POIs that change rarely (a restaurant doesn't move). Massive read QPS. Stale
data is fine for seconds–minutes. This is a **read-optimized** problem.

- **Index choice:** **Geohash** is the sweet spot. Data is static, so you precompute geohashes once
  and store them as an indexed column; no update cost to worry about. Store **multiple precisions**
  (e.g. geohash6 and geohash4 columns) so a "1 km" and a "20 km" query each hit an appropriately
  coarse grid without scanning too much. (If you have polygons — delivery zones, neighborhoods —
  add **PostGIS**.)
- **Storage / DB:** a relational store (PostGIS or even MySQL with a geohash column) is fine because
  it's read-heavy and you want rich filters (cuisine, rating, open-now) joined with the spatial
  filter. The spatial index narrows to ~hundreds of candidates; SQL filters/ranks the rest.
- **Sharding:** **shard by region / geohash prefix** so a query touches one (or few) shards. Beware
  the **hot-shard** problem — Manhattan has 1000× the density and traffic of rural Montana. Mitigate
  by sharding on a *finer* prefix in dense areas (uneven shard boundaries) rather than a uniform
  grid, and by leaning hard on caching.
- **Caching (this is where the wins are):** the workload is read-heavy and *spatially repetitive*
  (everyone in a neighborhood queries the same cells). Cache **per-cell result lists** in Redis,
  keyed by `geohash6` (+ filters). Hit rates are high because requests cluster. Put the popular
  "downtown" cells behind a CDN-style edge cache if responses are public. Pair with **read replicas**
  for the DB. Because data is static, **long TTLs** are safe; invalidate on the rare POI edit.
- **Why not a write-optimized design?** You'd be paying for update machinery you never use.

> **Tradeoff sentence:** "Yelp is read-heavy and static, so I pay the indexing cost *once* at write
> time and optimize relentlessly for reads — geohash column, region sharding, aggressive per-cell
> caching, read replicas. I accept seconds of staleness because a restaurant's location and hours
> don't change second-to-second."

### D.2 Uber / Lyft — real-time driver matching (WRITE-heavy pings + nearby queries)

**Shape:** millions of drivers each emitting a **location ping every 3–5 s**; riders issue "nearby
drivers" queries; the system must **dispatch** a match in well under a second. This is a **write
firehose with freshness requirements** — the exact opposite of Yelp.

The killer realization: **you cannot index millions of high-frequency moving points in a durable
disk DB.** Every ping would be a write + index update; the index churns constantly. So:

- **Don't persist every ping to the source-of-truth DB.** Location is **ephemeral, in-memory, and
  recent.** Keep the live location store in **Redis / an in-memory geo store** (or a sharded
  in-memory grid of quadtree/H3 cells). The relational DB holds *trips and drivers*, not the
  firehose.
- **Cellularize the world.** Partition into **H3 (or quadtree) cells**. A driver's ping updates
  *which cell they're in*; a nearby query reads **the rider's cell + the surrounding ring of cells**
  and filters by true distance — same coarse-then-fine pattern as geohash, but H3's uniform
  hexagonal neighbors make the ring math clean and the cell membership cheap to update.
- **Tame the firehose:**
  - **Throttle / batch** pings; you rarely need sub-3 s freshness for matching.
  - **In-memory only**, updated in place — overwrite the driver's last position, don't append.
  - **Shard the location service by geo cell**, so each node owns a region of the grid and its
    writes/queries stay local. Drivers crossing a shard boundary hand off.
  - **TTL / heartbeat for liveness:** each ping refreshes a key with a short **TTL (e.g. 30 s)**. A
    driver who goes offline or loses signal simply *expires* out of the index — no explicit
    "I'm leaving" message needed, no stale ghost drivers in results. Heartbeat = the location ping
    itself.
- **Dispatch:** rider requests → query rider's cell + neighbors → candidate drivers within radius →
  rank by **ETA (road network, not straight-line)** and availability → offer to top driver → on
  decline, fall through to next. Use a **lock / state machine** on the driver so two riders can't be
  matched to the same car (idempotency / "exactly-one-match" — see Topic on coordination).
- **Why H3/quadtree over a geohash column here?** The data moves constantly; you want an in-memory,
  density-adaptive, uniform-neighbor structure, not a disk-indexed string column that you'd be
  re-writing millions of times a second.

> **Tradeoff sentence:** "Uber is a write firehose of moving objects with hard freshness needs, so
> location lives in an **in-memory, cellular** store (H3/quadtree in Redis), updated in place, with a
> **TTL heartbeat** so offline drivers self-evict. I keep the firehose *out* of the durable DB
> entirely — the source of truth stores trips, not pings."

---

## Part E — The read-vs-write tradeoff in spatial indexing

This is the deep-dive that interviewers love because it's a genuine tension, not a recital.

**Granularity is a dial with opposite effects on reads and writes:**

- **Fine cells (small, high precision):** a "nearby" **read** touches few entities per cell → cheap,
  precise reads, small scans. **But** a moving object **crosses cell boundaries often** → frequent
  "remove from cell A, add to cell B" updates → expensive writes, more index churn.
- **Coarse cells (large, low precision):** a moving object **rarely changes cell** → cheap writes,
  stable membership. **But** each cell holds many entities → a read scans a big candidate set and
  does lots of haversine filtering → expensive reads.

> **The staff move:** "Granularity trades read cost against write cost. Yelp is read-heavy and
> static, so I go **fine** — pay precise small reads, never pay the update. Uber is write-heavy with
> moving objects, so I lean **coarser** (or store position-within-cell and only re-bucket on actual
> cell change) to keep the update rate sane, and recover read precision with the haversine
> post-filter. Same index family, opposite tuning, driven entirely by the read/write ratio."

Related update-cost levers: **update in place** (overwrite last position, don't append history);
**only re-bucket on cell change**, not on every ping; **TTL expiry** instead of explicit deletes;
**batch** updates.

---

## Part F — Geofencing

A **geofence** is a region (circle or polygon); you want to fire an event when an entity
**enters/exits/dwells** — "notify when the driver reaches the pickup", "promo when the user enters
the mall", "alert if the asset leaves the yard".

Two directions, and you should name which one the prompt needs:

- **Point-in-many-fences ("which fences contain this point?"):** you have many static fences and a
  stream of points. **Index the fences** (R-tree / PostGIS, or precompute each fence's covering set
  of S2/H3 cells). For each incoming point, compute its cell and look up which fences cover that
  cell → candidate fences → exact point-in-polygon test. The **cell covering** trick (represent each
  polygon as the set of grid cells it overlaps) turns expensive polygon math into a cheap cell-id
  hash lookup as the coarse filter.
- **Enter/exit detection:** you need the **previous** state to fire transitions, so keep last-known
  membership per entity; compare to current membership each tick; emit `ENTER`/`EXIT`/`DWELL`
  (dwell needs a timer). Debounce GPS jitter near the boundary (hysteresis / min-dwell) so you don't
  emit flapping events.

Architecturally this is a **stream-processing** problem: location stream → cell lookup → fence
match → state diff → event. Often built on Kafka + a stream processor, with fence coverings cached
in memory.

---

## Part G — Datastore options (pick with a reason)

| Store | Mechanism | Sweet spot | Watch out for |
|---|---|---|---|
| **PostGIS** (Postgres) | GiST/R-tree, full geometry ops (`ST_DWithin`, `ST_Contains`) | Rich queries, polygons, joins with business data, source-of-truth | Scaling writes; it's a single-leader RDBMS — shard/replicate yourself |
| **Redis GEO** | **Geohash-backed sorted set** (`GEOADD`/`GEOSEARCH`); 52-bit geohash as the ZSET score | Fast in-memory radius queries, real-time location, TTL freshness | In-memory size limits; no rich polygon ops; durability is your job |
| **Elasticsearch** | `geo_point` (BKD-tree) + `geo_shape`; geo queries alongside full-text | "Nearby" **plus** text/filters/aggregations (search-y proximity, geo facets) | Eventually consistent; cost; keep in sync via CDC/queue |
| **MongoDB** | `2dsphere` (S2-based) index | Document model with built-in geo, moderate scale | Same caveats as any geo on a general DB at high write rates |
| **Specialized / in-memory grid** | Custom quadtree/**H3** cells in memory, sharded by region | The Uber firehose; full control over update cost & dispatch | You build the durability, sharding, failover yourself |

> **Redis GEO is the answer they often fish for** on real-time location: it *is* a geohash index
> riding on a sorted set, `GEOSEARCH ... BYRADIUS` does the neighbor-union + filter for you, and a
> per-key **TTL** gives you free driver expiry. Name it when you want fast, in-memory, freshness-
> driven proximity. Name **PostGIS** when you have polygons/containment and rich filters. Name
> **Elasticsearch** when "nearby" rides alongside text search and aggregations.

---

## Part H — Worked end-to-end: "Design Uber's nearby-drivers"

A compressed run of the Topic-1 framework, geo-flavored.

**1. Requirements.**
- *Functional:* drivers send location pings; a rider requests nearby available drivers; system
  returns/dispatches the best match.
- *Non-functional:* **write-heavy** (the ping firehose) with **freshness** (a few seconds); nearby
  query p99 < ~200 ms; dispatch end-to-end < ~1 s; high availability for matching; stale driver
  positions tolerable for seconds but **ghost drivers (offline but still shown) are not acceptable**.

**2. Estimation (this *justifies* the in-memory firehose decision).**
- Say **3M active drivers**, ping every **4 s** → `3,000,000 / 4 ≈ 750k location writes/sec`.
- Peak ≈ 2–3× → **~1.5–2M writes/sec**.
- Rider "nearby" queries: say 1M concurrent riders searching, ~1 query each every few seconds →
  **hundreds of thousands of read QPS**.
- Per ping ≈ (driverId, lat, lng, ts) ≈ ~32 bytes; only the **latest** matters, so working set ≈
  `3M × ~100 bytes ≈ 300 MB` of *live* location → **fits in memory easily**; the firehose is the
  problem, not the volume.

> **Derived conclusion (say it):** "~1.5M writes/sec of *overwrites* where only the latest value
> matters and the whole live set is ~300 MB → this screams **in-memory store, update-in-place, not a
> disk DB**. Persisting every ping would be ~1.5M IOPS for data I throw away in 4 s. So the durable
> DB never sees the firehose."

**3. API.**
```
POST /drivers/{id}/location   { lat, lng, ts }            -> 200 (heartbeat, refreshes TTL)
GET  /drivers/nearby?lat=&lng=&radius=&limit=             -> [ {driverId, etaSec, distM} ]
POST /trips/request           { riderId, pickup, dropoff } -> tripId  (kicks off dispatch)
```

**4. Data model / access pattern.**
- **Live location (ephemeral):** sharded in-memory grid keyed by **H3 cell** → set of driver
  positions in that cell, each with a **TTL ~30 s**. Access pattern: write = update one driver's
  cell membership; read = "give me cell C + its ring of neighbors".
- **Durable (source of truth):** drivers, riders, trips in a relational/NoSQL store. **Not** pings.

**5. High-level design.**
```
Driver app ──pings──▶ Location ingest (LB, geo-sharded) ──▶ In-memory geo store (H3 cells, TTL)
                                                                     ▲
Rider app ──nearby/request──▶ Matching service ──queries cell+ring──┘
                                   │
                                   └─▶ ETA ranking ──▶ dispatch (lock driver) ──▶ Trip service ──▶ durable DB
```
- Ingest is **sharded by geo cell** so a region's writes land on one node (locality, no cross-talk).
- Matching queries the rider's H3 cell + neighbor ring, filters by haversine, ranks by **road-network
  ETA**, offers to the top driver, locks them, falls through on decline.

**6. Deep dives (propose the hard parts).**
- **The firehose** → in-memory, update-in-place, TTL heartbeat (covered). *"Only the latest ping
  matters, so I overwrite; a missed heartbeat expires the driver — that's how offline drivers vanish
  from results without an explicit message."*
- **Index granularity** → choose an H3 resolution where a cell ≈ a city block-ish so a moving car
  doesn't re-bucket every ping (write cost) yet a query scans few drivers (read cost); recover
  precision with the haversine post-filter. (Part E.)
- **Boundary/ring** → query center cell + ring of neighbors, not just one cell — H3's uniform hexagon
  neighbors make this clean. (Part C analog.)
- **Hot cells** (downtown at rush hour) → finer sub-sharding in dense regions; the cellular sharding
  already localizes load; cache nothing here because freshness matters — instead scale the owning
  shard.
- **Dispatch correctness** → distributed lock / state machine on the driver so two riders can't grab
  the same car (exactly-one-match; idempotent offers).
- **ETA vs straight-line** → straight-line distance is the *candidate* filter; final ranking uses a
  routing/ETA service because a river or one-way street ruins crow-flies ranking.

**7. Wrap-up / failure modes.**
- **In-memory store dies** → drivers re-ping within seconds and rebuild the cell (TTL means state is
  self-healing); replicate hot shards for availability.
- **Geo-shard hotspot** → rebalance cell ownership; finer partitioning downtown.
- **GPS jitter** → smoothing; don't re-bucket on sub-cell noise.
- **More time:** surge pricing off the same H3 demand/supply per-cell counts; predictive
  pre-positioning; multi-region failover.

---

### Self-check before the mock (answer these from memory)
- [ ] Why does a B-tree index fail at 2D "nearby" — what exactly is the dimensionality problem?
- [ ] Explain geohash encoding end-to-end: bisection, bit interleaving, base32, prefix = proximity.
- [ ] State the geohash **boundary problem** and the **8-neighbor + haversine** fix.
- [ ] When do you pick quadtree over geohash? R-tree over both? S2/H3 over all of them?
- [ ] **Why do hexagons beat squares** for neighbor distance — and what's the hexagon's one downside?
- [ ] How does the read-vs-write tradeoff push cell **granularity** in opposite directions?
- [ ] Why is Yelp's design (read-heavy/static) the *opposite* of Uber's (write-heavy/moving)?
- [ ] How do you handle the location **firehose**: in-memory, update-in-place, TTL heartbeat — and
      why does the durable DB never see the pings?
- [ ] What is Redis GEO actually built on, and when would you reach for it vs PostGIS vs Elasticsearch?
- [ ] Two flavors of **geofencing** (point-in-fences vs enter/exit) and the cell-covering trick.
