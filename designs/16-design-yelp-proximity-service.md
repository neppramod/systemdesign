# Design 16: Yelp / Proximity Service (Find Nearby Businesses + Reviews)

> **Why this problem is a staff filter:** "find restaurants near me" looks like the *easier* twin of
> Uber — and the trap is designing it *like* Uber. The signal here is recognizing the inversion the
> moment you say the numbers: Yelp is **read-heavy and mostly static** (a restaurant doesn't move;
> its hours change monthly), where Uber is a **write firehose of moving objects**. Same `WHERE near`
> query, *opposite* design. Get that framing wrong and you spend the round building a TTL-heartbeat
> location store for data that never changes, or you under-build the caching/replication layer that
> is actually the whole game. The staff move is: pay the spatial-indexing cost **once at write time**,
> then optimize relentlessly for reads — geohash column, region sharding, read replicas, and
> aggressive multi-layer caching of the searches that repeat — *and* layer name/category **search**
> and a **reviews/ratings** subsystem on top without breaking that read-optimized spine. This
> walkthrough runs the [Topic 1 framework](../prep/01-framework-and-building-blocks.md) end to end and
> leans on the [geospatial](../prep/12-geospatial-systems.md), [search](../prep/11-search-systems.md),
> [caching](../prep/06-caching-deep-dive.md), and [replication/consistency](../prep/05-replication-and-consistency.md)
> building blocks.

---

## 1. Requirements (5 min) — drive this, don't wait

I'll state the buckets out loud and **scope aggressively** to protect my 45 minutes.

### Functional (the verbs)
- A user **searches for businesses near a location** — by point + radius (or the map's bounding box),
  optionally filtered by **category** (cuisine, "coffee"), **name** ("Blue Bottle"), price, rating,
  and **open-now**.
- The system returns a **ranked list** of nearby businesses (distance + rating + relevance).
- A user **views a business page**: details, hours, photos, aggregate rating, and reviews.
- A user **writes a review** (text + star rating); the business's **aggregate rating** updates.
- Businesses are **created/edited** (by owners or an internal catalog pipeline) rarely.

> **Scope out loud:** "I'll build the **nearby search** (the core), the **business page read**, and
> the **reviews + aggregate-rating** subsystem, with name/category search combined with the geo
> filter as a deep dive. I'll treat **photo/media storage** as a black-box blob+CDN service
> ([Topic 14](../prep/14-blob-storage-and-media.md)), and skip reservations, messaging, ads/ranking
> auctions, and the business-owner dashboard unless you want them. Shout if you'd rather I go deep on
> one of those."

### Non-functional (where staff candidates separate)
- **Read-heavy, overwhelmingly.** Searches and page views dwarf writes by ~1000:1. **This single
  fact drives the whole architecture** — it's the mirror image of Uber's write-firehose framing.
- **Data is mostly static.** A business's location *never* changes; hours/menu change monthly;
  reviews trickle in. So I pay the indexing cost **once** and never fight write churn on the geo
  index — the opposite of the
  [Uber moving-object problem](../prep/12-geospatial-systems.md#d2--uber--lyft--real-time-driver-matching-write-heavy-pings--nearby-queries).
- **Latency:** nearby-search p99 < ~200 ms (it's interactive, map-driven); business-page read
  < ~100 ms (cacheable). Writing a review can be slower (< ~1 s) — it's rare and the user expects a
  "submitting…" beat.
- **Consistency — split it deliberately, and lean *eventual* almost everywhere:**
  - Search results / aggregate ratings → **eventual** is fine. If a brand-new review takes 30 s to
    move the average from 4.31 → 4.32, *nobody can tell and nobody is harmed.* I'll justify this
    explicitly in §6.4 — it's what *unlocks* the caching and async rating pipeline.
  - The **raw review you just posted** → read-your-own-writes (you must see your review immediately,
    even if the aggregate lags). A small, targeted consistency requirement.
- **Availability:** 99.9%+ on the read path — Yelp being down is embarrassing, not a money-on-fire
  emergency like Uber dispatch. Reads should **degrade gracefully** (serve slightly stale cached
  results) rather than error.
- **Abuse surface:** reviews are user-generated → **spam/fake-review** vector. I'll leave a hook,
  not build the ML.

> **The single derived insight to state now:** "This is a **read-optimized, mostly-static** system.
> The geo index is computed once and barely touched after; the wins are all on the read side —
> **caching the repeated searches**, **read replicas**, and a **CDN** for popular business pages.
> Eventual consistency is acceptable almost everywhere, which is *precisely what lets me cache so
> aggressively.* Contrast Uber: same nearby query, but write-heavy + freshness-critical forces an
> in-memory, update-in-place store with TTL heartbeats — I'd build *none* of that machinery here
> because I'd be paying for updates I never make." That sentence is the whole interview in miniature.

---

## 2. Estimation (3 min) — justify the read-optimized decision

The numbers exist to *force* the architecture, per [Topic 2](../prep/02-estimation-and-napkin-math.md).

**Assume:** ~50M businesses worldwide, 100M DAU, each doing ~5 searches/day, and ~1 review per user
*per month* (reviews are rare).

### The read side (searches + page views)

| Quantity | Calc | Result |
|---|---|---|
| Search QPS (avg) | 100M DAU × 5 searches ÷ 86,400 | **~6,000 QPS** |
| Search QPS (peak, 3×) | 6,000 × 3 (lunch/dinner spikes) | **~18,000 QPS** |
| Business-page views | ~2–3× searches (each search → a couple opens) | **~15,000 QPS avg, ~45,000 peak** |
| **Total read QPS (peak)** | search + page | **~60,000 QPS** |

### The write side (reviews + edits)

| Quantity | Calc | Result |
|---|---|---|
| Reviews/day | 100M users × 1/month ÷ 30 | **~3.3M/day** |
| Review write QPS (avg) | 3.3M ÷ 86,400 | **~40 QPS** |
| Review write QPS (peak) | ~3× | **~120 QPS** |
| Business edits | far rarer than reviews | negligible (handful/sec) |

### Storage (it all fits — volume is not the problem)

| Data | Calc | Result |
|---|---|---|
| Businesses | 50M × ~2 KB (name, address, lat/lng, geohash, category, hours) | **~100 GB** |
| Reviews (5 yr) | 3.3M/day × 365 × 5 × ~1 KB | **~6 TB** |
| Photos | blob store + CDN, off the main DB | (separate, [Topic 14](../prep/14-blob-storage-and-media.md)) |

> **The asymmetry, said out loud — this is the money sentence:** "Reads are **~60,000 QPS at peak**;
> writes are **~120 QPS** — a **~500:1 read:write ratio**. And the writable geo data (locations) is
> *static* — businesses don't move. So: the entire **business dataset is ~100 GB**, which fits on a
> single beefy node, let alone a small cluster. The problem was **never data volume or write rate**
> — it's **read throughput and latency at scale**. That's the inverse of Uber, where ~3M writes/sec
> *forced* an in-memory firehose. Here the numbers force the *opposite*: index once, then **scale
> reads** with replicas + caching + CDN, and accept staleness because nothing changes fast." This
> derivation *is* the design; everything below follows from it.

The ~500:1 ratio plus *spatial repetition* (everyone at lunch in SoMa queries the same cells) means
**cache hit rates will be very high** — which is the single biggest lever in the whole design.

---

## 3. API design (3 min)

A handful of endpoints to pin the contract. Auth + rate-limiting at the gateway
(see [Topic 9](../prep/09-api-gateway-loadbalancing-ratelimiting.md)) so I don't re-explain it.

```
# Search / discovery (the read core)
GET  /search?lat=&lng=&radius=&q=&category=&price=&openNow=&sort=&cursor=
        -> { results: [ {bizId, name, distM, rating, reviewCount, priceLevel} ], nextCursor }
        # bounding-box variant for map pans:
GET  /search?bbox=swLat,swLng,neLat,neLng&category=&...        -> same shape

# Business detail (the other big read)
GET  /businesses/{bizId}                 -> { name, address, lat, lng, hours, rating, reviewCount, photos[] }
GET  /businesses/{bizId}/reviews?cursor= -> { reviews: [...], nextCursor }

# Review write path
POST /businesses/{bizId}/reviews   { userId, stars (1-5), text, photoRefs[] }
        Idempotency-Key: <client-uuid>          -> { reviewId, status: ACCEPTED }

# Catalog admin (rare writes)
POST /businesses           { name, address, lat, lng, category, hours, ... }   -> { bizId }
PUT  /businesses/{bizId}    { ...partial... }                                  -> 200
```

Notes that surface hidden requirements:
- **Two search shapes — `radius` and `bbox`.** Mobile sends a center+radius ("near me"); the desktop
  map sends the **viewport bounding box** as the user pans/zooms. Both reduce to the same spatial
  query; I'll mention the bbox case so map-pan caching (§6.5) makes sense.
- **Cursor-based pagination**, not offset — offset breaks under inserts and is expensive deep into
  results ([search pagination](../prep/11-search-systems.md#pagination-deep-pagination-is-expensive)).
  For geo, the cursor naturally encodes "next ring out + last distance seen."
- **`POST …/reviews` carries an `Idempotency-Key`** — a user double-taps submit or retries on a flaky
  network; we must not post the review twice (which would also double-count it in the average). Out of
  the [idempotency playbook](../prep/10-distributed-transactions-and-idempotency.md#part-e--idempotency-in-depth).
- **`openNow` is computed**, not stored as a boolean — it's `now()` vs the business's hours in the
  user's timezone, applied as a post-filter. Worth flagging because it *can't* be precomputed into the
  geo index (it changes every minute) — it's a cheap app-side filter on the candidate set.

---

## 4. Data model (5 min) — access pattern picks the store

The access patterns are: (1) "businesses near a point/box, filtered + ranked" (the hot path),
(2) "one business by id" (page view), (3) "reviews for a business, paginated", (4) "append a review +
bump an aggregate." These point at a **relational source of truth** with a **geo index column**, a
denormalized **aggregate**, and a **search index** alongside.

### 4a. Core entities (relational source of truth)

| Entity | Key | Notes / access pattern |
|---|---|---|
| `businesses` | `biz_id` | name, address, **lat, lng**, **geohash6 + geohash5/geohash4** (precomputed), category, price, hours, region/shard key. Read by id; spatially by geohash prefix. |
| `business_stats` | `biz_id` | **denormalized aggregate**: `rating_avg`, `review_count`, `sum_of_stars`. Updated async (§6.4). The hot read field. |
| `reviews` | `(biz_id, review_id)` | one row per review; `review_id` is a [Snowflake id](../prep/10-distributed-transactions-and-idempotency.md#part-g--distributed-id-generation-for-idempotency-keys--ordering) (time-sortable). Listed by `biz_id`, newest-first. |
| `users` | `user_id` | profile; their reviews indexed by `user_id`. |
| `idempotency_keys` | `key` (unique) | dedup review posts; TTL'd. |

- **SQL for the source of truth, deliberately.** It's read-heavy with **rich filters** (category +
  price + rating + open-now) joined to a spatial predicate, the data is relational (business ↔ reviews
  ↔ stats), and the write rate (~120 QPS) is trivial for a single-leader RDBMS. This is the textbook
  [SQL pick from access patterns](../prep/03-databases-deep-dive.md), not reflex. **PostGIS** is the
  natural choice — it gives me `ST_DWithin` / `geo` indexing *and* the relational filters in one query;
  even plain MySQL/Postgres with an indexed `geohash` column works because the data is static
  ([Topic 12 datastore table](../prep/12-geospatial-systems.md#part-g--datastore-options-pick-with-a-reason)).
- **Store *multiple geohash precisions* as indexed columns** — `geohash4` (~39 km), `geohash5` (~5 km),
  `geohash6` (~1.2 km). A "1 km near me" query hits geohash6; a "20 km in this metro" query hits a
  coarser column without scanning a million fine cells. Because data is static, **I precompute these
  once at insert/edit time** and never recompute — no write-churn cost
  ([Yelp index choice](../prep/12-geospatial-systems.md#d1--proximity-service--yelp-nearby-restaurants-read-heavy-static-data)).
- **`business_stats` is denormalized on purpose** — the aggregate rating is read on *every* search
  result and *every* page view (~60k QPS), but recomputing `AVG(stars)` over a business's reviews on
  each read would be insane. Normalization vs denormalization, decided by the read pattern:
  **denormalize, maintain it async** (§6.4).

### 4b. Search index (name / category, alongside geo)

Name and category search ("Blue Bottle near me", "vegan thai") is a **full-text + filter** problem the
RDBMS does poorly, so I add an **Elasticsearch** index ([Topic 11](../prep/11-search-systems.md)):
each business is a doc with `name`, `category[]`, `geo_point` (lat/lng), `rating`, `price`, fed from
the SQL source of truth via **CDC** (§6.3). ES does `geo_point` (BKD-tree) + text + filters +
aggregations in one query — exactly the "nearby **plus** text" sweet spot from the
[geo datastore table](../prep/12-geospatial-systems.md#part-g--datastore-options-pick-with-a-reason).

> **Which store serves which query? Say it crisply:** "**Pure geo + structured filter** (no text)
> → I can serve from the geohash-indexed RDBMS *or* ES; I'll lean on ES for everything so I have one
> ranking path. **Text query ('blue bottle') + geo** → ES, because the RDBMS can't rank text. The
> RDBMS is the durable **source of truth**; ES is a derived, rebuildable read index kept in sync by
> CDC. If ES is down, I can degrade pure-geo searches back to the geohash column."

---

## 5. High-level design (10 min) — the happy path end to end

```
                                    READ PATH (≈500× the traffic — optimize this)
  Browser/App ──▶ CDN ──(miss)──▶ API Gateway ──▶ Search Service ──┐
   (popular biz pages,            (authn,            │ build cell set (geohash + neighbors)
    static search results)         rate-limit)       │ query ▼
                                                 ┌── Result Cache (Redis, keyed by geohash6+filters) ──┐
                                                 │     (miss) ▼                                         │ hit
                                                 │   Elasticsearch (geo_point + text + filters) ───────┤
                                                 │     │ rank (distance + rating + relevance)          │
                                                 │     ▼ haversine post-filter                         ▼
  Browser/App ──▶ Business Service ──▶ Read replicas (PostGIS / SQL) + biz-page cache ──▶ results + ratings
                                                       ▲ (leader → replicas)
            ─────────────────────────────────────────  │  ──────────────────────────────────────────────
                                    WRITE PATH (≈120 QPS — rare, can be async)
  App ──post review──▶ API Gateway ──▶ Review Service ──(idempotency check)──▶ Leader DB (reviews row)
                                                │ outbox row in same txn
                                                ▼
                                         Kafka (review events) ──▶ { Rating Aggregator, Spam/Abuse, Search Indexer }
                                                                          │ updates           │ flag         │ CDC → ES
                                                                          ▼                    ▼              ▼
                                                                   business_stats         moderation     ES doc refresh
                                                                   (rating_avg)            queue
```

**Walk one search through it out loud:**
1. App issues `GET /search?lat=&lng=&radius=&q=&category=` → **CDN** (for cacheable public result
   pages) → on miss, **API Gateway** (authn, rate-limit) → **Search Service**.
2. Search Service checks the **Redis result cache**, keyed by `(geohash6, filters, sort)`. **High hit
   rate** because requests cluster spatially (everyone at this corner queries the same cell). Hit →
   return immediately.
3. Miss → compute the center geohash at the precision matching the radius, **add the 8 neighbor
   cells**, and query **Elasticsearch** with `geo` filter + text + category/price filters.
4. ES returns candidates; Search Service applies the **haversine post-filter** (carve the circle out
   of the 3×3 cell block), **ranks** (distance + rating + relevance, §6.6 below), paginates, and joins
   `business_stats` (rating/count) — caching the result back into Redis with a TTL.
5. User opens a business → **Business Service** → **read replica** (or biz-page cache / CDN) → details
   + reviews. Static, highly cacheable.
6. User posts a review → **Review Service** dedups on the idempotency key, writes the `reviews` row to
   the **leader DB** plus an **outbox** row in the same transaction, and ACKs the user (they
   immediately see their own review — read-your-own-writes).
7. The outbox relay ships the event to **Kafka**; the **Rating Aggregator** updates `business_stats`,
   the **Search Indexer** refreshes the ES doc, and the **Spam/Abuse** consumer scores it — all async.

Keep it this simple first; the next section is where the round is won.

---

## 6. Deep dives (15 min) — propose the hard parts

> **Open with:** "The interesting problems are (1) the **nearby spatial query** — index choice and the
> cell+neighbor+haversine execution, (2) **scaling the reads** — sharding static geo data, replicas,
> and the multi-layer cache where the searches *repeat*, (3) combining **name/category search** with
> the geo filter, and (4) the **reviews subsystem** — write path, the **hot-business aggregate-rating**
> problem, and a spam hook. Can I go deep on the spatial query and the read-scaling/caching — they're
> the heart of why this differs from Uber?"

### 6.1 Deep dive — the nearby spatial query (the core)

Why the naive `WHERE lat BETWEEN … AND lng BETWEEN …` fails is a
[dimensionality problem](../prep/12-geospatial-systems.md#why-the-naive-query-doesnt-scale): a B-tree
orders **one** axis, so an index on `lat` gives you the entire latitude band — the equator's worth of
longitudes — and you scan the second dimension. You read a long thin strip, touching tens of thousands
of rows to return 50. The fix is a **spatial index that turns 2D proximity into a 1D-prefix or cell
lookup.**

**Index choice — geohash, and I can defend it for *this* problem:**
- **The data is static and read-heavy.** I precompute each business's geohash **once** at write time
  and store it as an **indexed column** — there is *no update cost* to worry about, which is the whole
  reason geohash (a flat, sorted, persisted representation) beats an in-memory quadtree here. Proximity
  becomes a **string prefix** any B-tree or sorted store can range-scan.
- **Contrast with Uber's H3 pick:** Uber chose H3 because objects *move* — it wanted an in-memory,
  density-adaptive, uniform-neighbor structure it rewrites millions of times/sec. **None of that
  applies to static businesses.** I'd name H3/S2 as fine alternatives (and ES/PostGIS use S2/R-tree
  internally), but the *reason* to prefer them — movement, surge smoothing — is absent here. Picking
  geohash *and saying why the moving-object justification doesn't apply* is the staff signal.
- (If the prompt adds **polygons** — delivery zones, neighborhood boundaries — I'd reach for
  **R-tree/PostGIS** for containment, per [Topic 12 B.3](../prep/12-geospatial-systems.md#b3--r-tree).)

**The query execution (coarse-then-fine) — the recipe that proves I've built this:**
1. **Pick precision from radius.** A 1 km query → geohash6 (~1.2 km × 0.6 km cell), comparable to the
   radius. Too coarse scans too much; too fine and the 8 neighbors don't cover the circle and you'd
   need a 24-cell ring.
2. **Geohash the center** at that precision → e.g. `9q8yyk`.
3. **Compute the center cell + its 8 neighbors** (N, NE, E, SE, S, SW, W, NW) with the library's
   `neighbors()`. *This is the part juniors skip.* Geohash cells are rectangles, and **two points can
   be physically adjacent but sit in cells with no shared prefix** when they straddle a bisection line.
   A single-prefix query **silently misses neighbors right across the border**. Union the 9 prefix
   scans. This is the [boundary problem and its fix](../prep/12-geospatial-systems.md#part-c--a-radius-query-with-geohash-worked).
4. **Filter by true great-circle (haversine) distance** to the center, keeping only ≤ radius. This
   carves the **circle out of the 3×3 block of squares** — the cells are the *coarse* filter that lets
   the index do the heavy lifting; haversine is the *fine* filter for exactness.
5. Apply business filters (category, price, **open-now** computed against hours), **rank** (§6.6),
   paginate, return.

> **Tradeoff sentence:** "I convert 2D proximity to a **geohash prefix**, precomputed once because the
> data is static, and execute the standard **center-cell + 8-neighbors + haversine post-filter**
> recipe. The neighbor union is non-negotiable — without it I silently miss results across cell
> borders. I store **multiple precisions** so different radii hit appropriately coarse grids. I get
> precise, bounded reads for free because I never pay the update cost Uber pays — that's the
> read/write inversion made concrete."

**Granularity here leans *fine*** ([Topic 12 Part E](../prep/12-geospatial-systems.md#part-e--the-read-vs-write-tradeoff-in-spatial-indexing)):
fine cells → cheap, precise reads with small scans, and the *only* cost of fine cells (frequent
re-bucketing of moving objects) **doesn't exist** because businesses never move. "Same index family,
opposite tuning from Uber, driven entirely by the read/write ratio."

### 6.2 Deep dive — scaling the reads: sharding, replication, and the cache where searches repeat

This is where the round is actually won for Yelp, because **reads are 500× the writes**. Three layers,
each pulling a different lever.

**(a) Sharding the (mostly static) geo data — [Topic 4](../prep/04-sharding-and-partitioning.md).**
- The data is small (~100 GB) so I might not *need* to shard for capacity — but I shard for **read
  throughput** and **locality**. **Shard by region / geohash prefix** so a "near me" query touches one
  (or, at a boundary, two) shards rather than scattering across the cluster.
- **The hot-shard problem (name it before asked):** geography is wildly non-uniform — **Manhattan has
  1000× the businesses and search traffic of rural Montana.** A uniform geohash-prefix shard map puts a
  brutal hot shard on downtown and idle shards elsewhere.
  - **Mitigation: uneven shard boundaries / finer prefix splits in dense regions** — shard for equal
    *load*, not equal *area*. A dense city's cells subdivide across more nodes.
  - **Mitigation: lean hard on caching and replicas** (below) — for a *read-heavy static* workload,
    caching absorbs the hot region far more cheaply than re-sharding, which is the key difference from
    Uber (where freshness forbade caching the location plane).

**(b) Read replicas for read scaling — [Topic 5](../prep/05-replication-and-consistency.md).**
- The leader takes the trivial ~120 write QPS; **a fleet of read replicas serves the ~60k read QPS.**
  Single-leader, async replication. **Replication lag is fine** — a business that appears in search
  100 ms late, or an aggregate rating that lags by a second, is invisible to users. This is *why*
  eventual consistency (§1) is worth so much: it lets me scale reads with cheap async replicas.
- **The one exception — read-your-own-writes for the review author.** After posting, route *that user's*
  read of *their* review to the leader (or serve it from the write-through cache), so they see their
  own review immediately even while the aggregate and replicas lag. A targeted pin, not a global
  strong-consistency tax. (Classic [don't-read-your-own-writes-from-a-replica](../prep/05-replication-and-consistency.md) caveat.)

**(c) The cache — the single biggest win — [Topic 6](../prep/06-caching-deep-dive.md).**
The workload is read-heavy *and spatially repetitive*: everyone at a corner at lunch issues the **same
cell + same filters**. So hit rates are high. Multiple layers:
- **Search result cache (Redis), keyed by `(geohash6, category, price, sort)`.** Cache-aside, **long
  TTL** (minutes) because the data is static. A "coffee near this cell" result is reused by thousands
  of users. *This is the cache that makes the hot-shard problem mostly evaporate* — Manhattan's
  popular cells are served from Redis, never touching ES/DB.
- **Business-page cache** for the per-business detail blob (details + rating + first page of reviews),
  keyed by `biz_id`. Long TTL; invalidated on the rare edit or rating update.
- **CDN at the edge** (§6.5) for public, anonymous searches and popular business pages.
- **Guard the failure modes:** **hot keys** (the #1 cell) → can replicate that key across cache nodes
  or add a tiny local in-process cache; **thundering herd** when a popular key's TTL expires → use a
  **request-coalescing / single-flight** lock so one request recomputes while others wait, and/or
  stale-while-revalidate. Both straight from the [caching pitfalls](../prep/06-caching-deep-dive.md).

> **Tradeoff sentence:** "Because the data is static and staleness is acceptable, I scale reads with a
> **3-layer cache (CDN → Redis result cache → biz-page cache) + read replicas**, and shard by region
> with finer splits in dense cities. The cache *absorbs* the hot-shard problem cheaply — which is only
> possible because I *don't* need freshness. That's the exact lever Uber couldn't pull."

### 6.3 Deep dive — name/category search combined with the geo filter

A user types "blue bottle" or "vegan ramen" *and* wants it near them — **full-text ranking + a geo
constraint in one query.** The RDBMS does text ranking poorly, so this rides
[Topic 11 / Elasticsearch](../prep/11-search-systems.md).

- **Each business is an ES document** with an analyzed `name`, `category[]` keywords, a `geo_point`
  (BKD-tree indexed), and structured fields (`rating`, `price`, `priceLevel`). One query does the
  **`geo_distance`/`geo_bounding_box` filter + the text match + the category/price filter +
  aggregations** (facet counts: "12 coffee, 4 tea nearby") — exactly the "nearby **plus** text **plus**
  facets" sweet spot.
- **Query structure:** the geo predicate is a **filter** (binary, cacheable, no scoring), while the
  text is a **scoring** clause ([BM25](../prep/11-search-systems.md#bm25--the-default-and-why)). So
  ES first cheaply restricts to the geo cells, then ranks the survivors by text relevance — coarse geo
  filter, fine text rank, same coarse-then-fine spirit as the haversine pattern.
- **Typeahead** for the search box (suggest "Blue Bottle Coffee" as you type) is the
  [autocomplete subsystem](../prep/11-search-systems.md#part-e--autocomplete--typeahead-the-classic-question):
  a prefix structure with **top-k-per-prefix precomputed**, weighted by popularity, optionally
  geo-biased. I'd flag it as its own small read-heavy service, not build it in full.

**Keeping ES in sync without the dual-write bug.** ES is a **derived** index; the SQL DB is the source
of truth. Writing the DB *and then* writing ES is the
[dual-write anti-pattern](../prep/11-search-systems.md#the-naive-broken-approach-dual-write) — a crash
between them diverges them. **Fix: CDC** — the same Kafka stream (from the outbox/WAL, §6.4) feeds a
**Search Indexer** that updates the ES doc at-least-once; the indexer is idempotent (upsert by
`biz_id`). ES's [near-real-time segment model](../prep/11-search-systems.md#near-real-time-nrt-and-the-immutable-segment-trick)
means new/edited businesses are searchable within ~1 s — fine, since edits are rare and seconds of lag
is invisible. I keep a **reindex-from-source** job for schema changes and to repair drift.

> **Tradeoff sentence:** "Text-plus-geo goes to **Elasticsearch** — geo as a cheap filter, text as the
> scored clause — kept in sync from the SQL source of truth by **CDC**, never a dual write. If ES is
> down I **degrade** to pure-geo from the geohash column, losing only text ranking, not availability."

### 6.4 Deep dive — the reviews subsystem: write path, aggregate rating, and the hot-business problem

> **This is the part that distinguishes a real design.** "User writes a review" hides a genuine
> consistency-and-contention problem: **maintaining the aggregate rating** correctly, idempotently, and
> *without serializing all writes to a popular business.*

**The write path (idempotent, async-aggregated):**
1. `POST …/reviews` with an **idempotency key**. Review Service checks the key → on replay, returns the
   existing `reviewId` (so a double-tap doesn't post twice *and* doesn't double-count the rating).
2. In **one local transaction**, write the `reviews` row **and** an `outbox` row. ACK the user — they
   immediately read their own review (read-your-own-writes via leader/write-through cache, §6.2b).
3. The **outbox relay (CDC/Debezium tailing the WAL)** ships a `REVIEW_CREATED` event to **Kafka**
   at-least-once — fixing the [dual-write problem](../prep/10-distributed-transactions-and-idempotency.md#the-dual-write-problem-the-bug-you-will-be-asked-to-find).
4. Three consumers fan out: **Rating Aggregator** (updates `business_stats`), **Search Indexer** (CDC →
   ES), **Spam/Abuse** (scores the review).

**Computing the aggregate rating — running average, done right:**
- **Maintain a running aggregate, never recompute on read.** `business_stats` holds
  `(sum_of_stars, review_count)`; `rating_avg = sum/count`. On `REVIEW_CREATED`:
  `sum += stars; count += 1` — **O(1)**, vs `AVG()` over a popular business's millions of reviews on
  every read (insane at 60k read QPS). Denormalization decided by the read pattern.
- **Idempotency for the aggregate (the subtle bug):** Kafka is **at-least-once**, so the same
  `REVIEW_CREATED` can be redelivered → naive `sum += stars` **double-counts**. Fix: the aggregator is
  **idempotent** — either track processed `review_id`s (an inbox/dedup set), or make the increment
  conditional/keyed on `review_id` so a replay is a no-op. *This is exactly the ledger-posting
  idempotency discipline from [Topic 10](../prep/10-distributed-transactions-and-idempotency.md#part-h--handling-money-ledgers-double-entry-reconciliation),
  applied to a rating instead of money.*

> **The hot-business problem (name it):** a viral restaurant or a chain HQ gets a burst of reviews;
> if every review did `UPDATE business_stats SET sum=sum+? WHERE biz_id=X` synchronously, they'd all
> **contend on one row** (write hotspot / lock contention). Two mitigations: **(a)** the aggregation is
> already **async off Kafka**, partitioned by `biz_id`, so a single business's updates are *serialized
> on one partition* (correct, ordered) without blocking the user's write path — the user already got
> their ACK in step 2. **(b)** For an extreme hotspot, **batch**: the aggregator buffers N increments
> for that biz over a short window and applies one combined update — far fewer row writes. Staleness of
> a few seconds in the *average* is acceptable (§1), which is *what lets this be async and batched.*

**Why eventual consistency is the right call here (say it explicitly):** the aggregate rating moving
from 4.31 → 4.32 thirty seconds late is **imperceptible and harmless** — unlike a bank balance or
Uber's "who's assigned" decision. So I deliberately make the rating pipeline **async, idempotent, and
eventually consistent**, which is precisely what buys me the caching, replicas, and batching above. The
*only* strong-ish requirement is the author seeing their own raw review, handled by the targeted
read-your-own-writes pin — **a small consistency requirement carved out of an otherwise eventual
system.**

**Spam / abuse hook:** the **Spam/Abuse consumer** scores each review (rate per user, device/IP
clustering, text similarity to known fake-review templates, sentiment-vs-rating mismatch, new-account
signals). High-risk → **hold for moderation** (don't include in the aggregate or surface it until
cleared) or shadow-publish. I'd **leave this as a pluggable hook**, not build the ML, but flag that
running it *async, off the same event stream* means abuse scoring never slows the user's write — and a
held review simply isn't counted until it passes, so the aggregate stays clean.

### 6.5 Deep dive — CDN & edge caching for popular queries and business pages

Reads are public, repetitive, and mostly anonymous — ideal for the edge ([Topic 6](../prep/06-caching-deep-dive.md), [Topic 14](../prep/14-blob-storage-and-media.md) for media).

- **Business pages** (details + first reviews page + photos) are **near-static and globally popular**
  → cache the rendered page/JSON at the **CDN**, keyed by `biz_id`, long TTL, **purge on edit/rating
  update**. Photos already live in blob storage behind the CDN.
- **Popular searches** — "coffee near Times Square" — are anonymous and repeat constantly → cacheable
  at the CDN/edge keyed by `(geohash6, filters)`. **Personalized or auth'd** searches bypass the CDN
  and use the Redis result cache instead.
- **Map-pan bounding-box caching:** as a user pans the map, viewports overlap heavily. Snapping the
  bbox to a **grid of geohash cells** (rather than caching arbitrary float boxes) makes pans hit the
  *same* cached cell results — turning a continuous pan into discrete, cacheable cell fetches. A nice
  detail that shows I've thought about the *map* UX, not just the API.
- **Invalidation** is the hard half ("only two hard problems…"): edits and rating updates emit an event
  that **purges the affected `biz_id` and its cells** from CDN + Redis. Because writes are rare,
  invalidation volume is tiny — another gift of the static/read-heavy shape.

### 6.6 Deep dive — ranking nearby results (distance + rating + relevance), at a high level

> **Say this:** "Ranking isn't just nearest-first — Yelp's value is surfacing the *best* nearby
> option, so I blend signals. I'll keep it high-level, not build an LTR model in 45 minutes."

- A weighted score per candidate, e.g. `score = w₁·proximity + w₂·rating_quality + w₃·text_relevance
  + w₄·popularity`:
  - **Proximity** — inverse of haversine distance (closer is better, but not the *only* thing).
  - **Rating quality** — *not raw average.* A 5.0 from 3 reviews should rank below a 4.6 from 2,000.
    Use a **Bayesian/shrinkage average** (pull small-sample ratings toward the global mean) or a lower
    confidence bound — a cheap statistical fix I'd name explicitly, because "sort by avg rating" is the
    junior answer.
  - **Text relevance** — the BM25 score from ES when there's a query term.
  - **Popularity / engagement** — review count, views, click-through.
- For a text query, **relevance dominates**; for a bare "near me" browse, **proximity + rating
  dominate**. The weights are tunable and, in production, learned via
  [learning-to-rank](../prep/11-search-systems.md#learning-to-rank-ltr) from click/visit signals — I'd
  flag LTR as the evolution, not the v1.
- This blend is computed in the Search Service after the haversine filter, over the (small) candidate
  set — cheap, and cacheable along with the result.

---

## 7. Wrap-up (3 min) — bottlenecks, failure modes, SPOFs

**Remaining bottlenecks**
- **Hot cells / hot cities** at peak — addressed by the result cache (the big win) + finer sub-sharding
  + more replicas for dense regions, but it's the perennial tuning knob.
- **Hot-business aggregate writes** — addressed by async, per-`biz_id`-partitioned, optionally batched
  aggregation; the user write path never contends.
- **Deep pagination** of huge result sets — bounded by cursor pagination + capping result depth (you
  rarely need page 50 of "coffee near me").

**Failure modes (name them unprompted, per [Topic 13](../prep/13-resilience-and-failure-handling.md))**
- **Elasticsearch down / lagging.** ES is a *derived* index → **degrade gracefully**: serve pure-geo
  searches from the geohash-indexed RDBMS (losing only text ranking), and rebuild ES from the source of
  truth via the reindex job. Reads stay up. CDC catches ES back up from Kafka with no data loss.
- **Redis cache down.** Reads fall through to replicas/ES — **slower, not broken**. Guard against the
  resulting load spike (cold-cache thundering herd) with request coalescing and gradual warm-up;
  consider a small in-process L1 cache so a Redis blip doesn't fully expose the DB.
- **A read replica dies.** LB health-checks route around it; spin a replacement; the leader and other
  replicas carry on. No user impact.
- **The Kafka review-event bus dies.** The **outbox rows persist in the SQL DB** regardless, so the
  relay replays from the WAL once Kafka recovers — **no review/rating event lost**, just delayed.
  Aggregation and indexing are idempotent, so replay is safe.
- **A geo shard's leader dies.** Single-leader per shard → **leader-follower failover** promotes a
  replica; geo-sharding limits blast radius to one region. Writes for that region pause briefly; reads
  continue from replicas.
- **Spam flood.** The async spam consumer + held-for-moderation gate means a flood of fake reviews
  *never enters the aggregate*; rate-limit review posts per user/IP at the gateway as the first line.

**Single points of failure & redundancy**
- The **leader DB** per shard is single-writer → replicas + automated failover; geo-sharding bounds the
  blast radius to one city.
- The **result cache** is not a SPOF — it's an optimization; loss degrades latency, not correctness.
- **No global SPOF by design** — the read path is multi-layered (CDN → cache → replicas → ES → DB) and
  each layer degrades to the next; the system is regionally shardable end to end. The
  with-more-time item is **multi-region active-active** read serving (the static dataset replicates
  trivially across regions — another gift of the read-heavy/static shape).

**With more time:** learning-to-rank with click/visit feedback, personalization (your cuisine history),
geo-biased typeahead, photo/menu understanding, reservation + waitlist integration, a fuller
fraud/fake-review ML pipeline, and full multi-region active-active.

---

## What made this staff-level

- **Derived the whole architecture from one estimation insight** — the **~500:1 read:write ratio on
  *static* data** — and explicitly framed it as the **inverse of Uber's write-firehose**, instead of
  reusing Uber's machinery. Recognizing same-query/opposite-design is the headline signal.
- **Picked geohash with a real tradeoff** — precompute once because data is static; named H3/S2 as
  alternatives *and explained why their moving-object/surge justification doesn't apply here.* Choosing
  *and* defending the non-default is stronger than reciting H3 from the Uber answer.
- **Executed the spatial query correctly** — center cell + 8 neighbors + haversine post-filter — and
  said *why* the neighbor union is non-negotiable (the boundary problem), not just "use geohash."
- **Treated read-scaling as the actual problem** — 3-layer cache (CDN → Redis result cache → biz-page)
  + read replicas + region sharding — and showed the **cache absorbs the hot-shard problem cheaply**,
  the exact lever Uber *couldn't* pull because of freshness.
- **Solved the aggregate rating as a real consistency+contention problem** — running average,
  idempotent against at-least-once Kafka redelivery, and the **hot-business write hotspot** mitigated by
  async per-`biz_id` partitioning + batching — instead of `UPDATE … SET avg=AVG(...)`.
- **Split consistency deliberately** — eventual everywhere (ratings, search, replicas), with a *single*
  carved-out **read-your-own-writes** pin for the review author — and justified that eventual
  consistency is *what unlocks* the caching/replication wins.
- **Named the dual-write bug twice** (DB→ES and review→downstream) and fixed both with outbox/CDC,
  rather than drawing an arrow to Kafka/ES and moving on.
- **Named failure modes and graceful degradation** (ES down → pure-geo fallback; cache down → slower not
  broken; Kafka down → outbox replays) and showed there's no global SPOF.

---

### Self-check before the mock (answer these from memory)
- [ ] State the read:write ratio and what it forces — and why this is the *inverse* of Uber.
- [ ] Why geohash here and not H3? Which of H3's advantages don't apply to static businesses?
- [ ] Give the cell+neighbor+haversine recipe, and explain *why* the 8 neighbors are non-negotiable.
- [ ] Why store multiple geohash precisions, and why does precompute-once cost nothing here?
- [ ] Name the three read-scaling layers and which one absorbs the hot-shard problem — and why Uber
      couldn't use that lever.
- [ ] How is the aggregate rating maintained, and what's the at-least-once double-count bug + fix?
- [ ] What is the hot-business write-hotspot problem, and the two mitigations?
- [ ] Why is eventual consistency acceptable for ratings, and what *single* exception do you carve out?
- [ ] Where are the two dual-write bugs (DB→ES, review→downstream), and how does outbox/CDC fix them?
- [ ] How does ranking blend distance + rating + relevance — and why isn't raw average rating enough?
- [ ] What happens when Elasticsearch is down, and how does the system degrade?
