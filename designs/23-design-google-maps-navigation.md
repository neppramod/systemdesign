# Design 23: Google Maps / Navigation + Routing Service

> **Why this problem is a staff filter:** "show a map and draw a line between two points" sounds like
> a tile server with a Dijkstra call bolted on, and juniors design it that way — until you say the
> graph size out loud. The road network is a **directed weighted graph with ~hundreds of millions of
> nodes and edges spanning continents**, and the central operation — shortest path on it — has a
> textbook algorithm (Dijkstra) that **provably cannot meet a sub-100 ms p99 at that scale per
> request**. The staff move is recognizing that routing is won at *precompute time*, not query time:
> you trade a massive offline build (contraction hierarchies / highway hierarchies) for queries that
> touch a few thousand edges instead of hundreds of millions. Layer on a **GPS-probe firehose** that
> updates edge weights in near-real-time, a **read-heavy tile CDN path** that dwarfs everything, and
> **geocoding/search**, and you have four very different systems wearing one map. This walkthrough
> runs the [Topic 1 framework](../prep/01-framework-and-building-blocks.md) end to end and leans on the
> [geospatial](../prep/12-geospatial-systems.md), [streaming](../prep/18-batch-and-stream-processing.md),
> [search](../prep/11-search-systems.md), [blob/CDN](../prep/14-blob-storage-and-media.md), and
> [ML](../prep/23-ml-and-ai-system-design.md) building blocks.

---

## 1. Requirements (5 min) — drive this, don't wait

I'll state the buckets out loud and **scope aggressively** — Maps is enormous, so naming what I skip
is itself a seniority signal.

### Functional (the verbs)
- A user **views the map**: pans/zooms; the client fetches **map tiles** for the visible area at the
  current zoom level.
- A user **searches for a place** by name or address ("coffee near me", "1600 Amphitheatre Pkwy") and
  gets ranked results with coordinates — **geocoding + place search**.
- A user **requests a route** from origin → destination (optionally with waypoints, mode = drive/walk/
  transit) and gets one or more routes with **distance, ETA, and turn-by-turn directions**.
- During navigation the client gets **live ETA updates and reroutes** as traffic changes or the driver
  goes off-route.
- The system **ingests GPS probes** from millions of devices in motion and turns them into **live
  per-segment traffic speeds** that feed back into edge weights and ETAs.

> **Scope out loud:** "I'll build the four pillars — **tile serving, place search/geocoding, routing
> with live traffic, and the probe-ingestion → traffic loop** — with **routing-at-scale** and the
> **traffic pipeline** as the deep dives. I'll treat the **map-data pipeline** (how raw road geometry
> from satellite/Street View/municipal data becomes a graph) as an offline black box, and I'll skip
> transit/bike-specific routing detail, indoor maps, AR navigation, and Street View imagery unless you
> want them. Shout if you'd rather I go deep elsewhere."

### Non-functional (where staff candidates separate)
- **Overwhelmingly read-heavy, but on two different axes.** Tile + search + route reads are a massive
  read fan-out (CDN territory). The *one* write firehose is **GPS probes**, and like Uber's pings it's
  data we mostly **aggregate then discard**, not durably persist per-event.
- **Latency targets, per path:**
  - Tile fetch p99 < ~50 ms (it's a CDN edge hit — basically static).
  - Route computation p99 < ~150 ms for a continental route. **This is the hard one** and it's why we
    precompute.
  - Search/geocode autocomplete p99 < ~100 ms (feels instant as you type).
  - Live ETA refresh: every ~30–60 s during nav; not latency-critical to the millisecond.
- **Consistency — split it deliberately:**
  - **Routing is best-effort.** Two users asking the same route 200 ms apart may get slightly
    different ETAs because traffic moved — nobody cares. There's no "correct" single answer; there's a
    *good* one computed fast.
  - **Traffic is eventually consistent.** Edge weights lag real conditions by tens of seconds to a
    couple of minutes; that's inherent and fine.
  - **The base map graph is strongly versioned but slow-changing** — a new road appears in a
    *published map version*, not live. Routing reads an immutable snapshot.
- **Availability:** the read paths (tiles, search, routing) target 99.99% — a navigation outage strands
  drivers. Degrade gracefully: serve cached tiles, fall back to traffic-free ETAs if the live feed is
  stale.
- **Freshness vs staleness is the central tension everywhere.** Precomputed routing structures are
  fast but stale w.r.t. live traffic; tiles are cached at the edge but the world changes; traffic is
  fresh but noisy. Almost every deep dive is a freshness-vs-cost dial.

> **The single derived insight to state now:** "This is a **read-heavy, precompute-dominated** system.
> The expensive thinking — turning the planet's roads into a fast-queryable routing structure, and
> rendering the world into tiles — happens **offline, in batch**, so the online path is cheap reads
> against precomputed artifacts. The one online write firehose (GPS probes) is **aggregated, not
> stored per-event**, and feeds a *fast-updating overlay* on top of the slow-changing base graph.
> Hold the precompute/query split and the base/overlay split in your head and the whole design falls
> out."

---

## 2. Estimation (3 min) — justify the precompute and the CDN

Numbers exist to *force* the architecture, per [Topic 2](../prep/02-estimation-and-napkin-math.md).

**Assume:** 1B monthly users, ~100M daily active, peak concurrency ~20M. Of DAU, say 50M request at
least one route/day and ~10M are actively navigating (emitting probes) at peak.

### The graph itself (sizing the routing problem)

| Quantity | Calc | Result |
|---|---|---|
| Road intersections (nodes), global | known order of magnitude | **~few × 10^8 (~200M)** |
| Road segments (edges) | ~2–2.5× nodes (avg degree ~2.5, bidirectional) | **~500M edges** |
| Bytes per edge (geometry ptr, from/to, length, speed class, flags) | | ~50–100 B |
| Raw graph size | 500M × ~80 B + node table | **~50–100 GB** |
| With CH shortcuts + per-region precompute | adds another ~1–2× | **~150–300 GB** |

> **The graph fits on a beefy node's RAM/SSD — size was never the problem.** The problem is that a
> **single Dijkstra from SF to NYC settles tens of millions of nodes** before it reaches the target,
> at ~100s of ms–seconds of CPU *per request*. At route QPS (below) that's a non-starter. **The cost
> is in the search frontier, not the storage.** That sentence justifies the entire routing deep dive.

### The read fan-out (tiles + routes + search)

| Quantity | Calc | Result |
|---|---|---|
| Route requests/day | 50M users × ~3 routes | ~150M |
| Route QPS avg | 150M ÷ 86,400 | **~1,700/sec** |
| Route QPS peak | ~3× (commute hours) | **~5,000/sec** |
| Tile requests/sec peak | 20M concurrent × ~a few tiles/interaction, bursty pans | **~10^5–10^6/sec** |
| Search/autocomplete QPS peak | keystroke-level, ~2–3× route QPS | **~15,000/sec** |

### The GPS-probe write firehose

| Quantity | Calc | Result |
|---|---|---|
| Active navigators at peak | | ~10M |
| Probe every ~5 s | 10M ÷ 5 s | **~2M probes/sec** |
| Bytes per probe (deviceId-hash, lat, lng, speed, heading, ts) | | ~40 B |
| Probe ingest bandwidth | 2M × 40 B | **~80 MB/s** |
| Persisted? | **No** — aggregated to per-segment speeds, raw probes dropped/sampled | — |

> **Two money sentences:** (1) "**Tiles are ~10^5–10^6 req/sec of essentially static bytes** — that's
> a pure **CDN** problem; the origin should almost never be hit." (2) "**Routing is only ~5k QPS but
> each naive query is hundreds of ms of CPU over a 500M-edge graph** — multiply that out and a single
> machine does maybe a handful of full Dijkstras/sec. So I *must* precompute: contraction hierarchies
> turn each query into touching a few thousand edges (sub-ms–low-ms of work), and 5k QPS becomes
> trivial. The asymmetry — cheap-to-store graph, ruinously-expensive-to-search live — *is* the design."

---

## 3. API design (3 min)

A handful of endpoints to pin the contract. Auth + rate-limiting at the gateway
(see [Topic 9](../prep/09-api-gateway-loadbalancing-ratelimiting.md)) so I don't re-explain it.

```
# Tiles (CDN-fronted, essentially static; z/x/y = zoom + tile coords)
GET  /tiles/{z}/{x}/{y}.{fmt}                     -> tile bytes   (Cache-Control, ETag; CDN edge hit)

# Search / geocoding
GET  /search?q=&lat=&lng=&limit=                  -> [ {placeId, name, lat, lng, score} ]  (autocomplete)
GET  /geocode?address=                            -> { lat, lng, formattedAddress }
GET  /reverse-geocode?lat=&lng=                   -> { address, placeId }

# Routing
POST /route        { origin, destination, waypoints[], mode, departAt }
                                                  -> { routes: [ {polyline, distanceM, etaSec,
                                                                  steps:[turn-by-turn], trafficLevel} ] }
GET  /route/{id}/refresh?currentPos=              -> { etaSec, reroute? : {polyline, steps} }

# Probe ingestion (NOT through the normal request gateway — persistent/batched upload, §6.3)
POST /probes       { batch: [ {lat, lng, speed, heading, ts} ] }   -> 200   (anonymized, sampled)
```

Notes that surface hidden requirements:
- **`/route` returns multiple alternatives** with their own ETAs — fastest, shortest, avoid-tolls. The
  client/user picks; it also makes reroute decisions cheaper (a precomputed alternative may already
  avoid the new jam).
- **`departAt` matters** — ETA for a route *2 hours from now* must use **predicted** traffic (historical
  + ML), not current traffic. This single param forces the historical-speed-profile store (§6.5).
- **`/route/{id}/refresh` is the navigation loop** — the client periodically reports position; the
  server returns an updated ETA and, only if materially better, a reroute. Cheap, frequent, idempotent.
- **`/probes` is anonymized and batched** — privacy is non-negotiable for a probe firehose; the client
  strips identity, batches a few readings, and uploads opportunistically. It bypasses the request
  gateway like Uber's ping plane (§6.3).
- **Tiles carry `ETag`/`Cache-Control`** so the CDN and client cache aggressively; a tile only changes
  when the **map version** bumps.

---

## 4. Data model (5 min) — access pattern picks the store

The system splits into **distinct stores with opposite characteristics**, mirroring the four pillars.

### 4a. The routing graph (immutable, versioned, memory-resident)

The core entity. Access pattern: **read-only at query time** (the build pipeline is the only writer),
traversed as a graph, sharded by **geographic region/tile**.

| Entity | Key | Notes / access pattern |
|---|---|---|
| `node` (intersection) | `node_id` | lat/lng, region/cell id. ~200M of them. |
| `edge` (road segment) | `(from_node, to_node)` | length, road class, base speed, turn restrictions, flags (toll/oneway). Directed. |
| `shortcut` (CH) | precomputed | synthetic edges spanning many real edges, with the "contracted" path attached (§6.1). |
| `geometry` | `edge_id` | the polyline points for rendering the route line (separate from topology). |

- **This is not a SQL/NoSQL row store at query time.** It's a **purpose-built in-memory graph
  structure** (CSR — compressed sparse row adjacency arrays — for cache-friendly traversal), built
  offline and loaded as an immutable, versioned artifact. The source-of-truth map data lives in a
  geo-database / object store; the *query-time* representation is a compiled binary, like a search
  index. Same idea as [search's build-then-serve split](../prep/11-search-systems.md).
- **Versioned & immutable.** A new "map version" is built periodically (roads change slowly). Routing
  servers load version *N*; a new build publishes version *N+1*; servers hot-swap. Immutability means
  no locking on the read path and trivial rollback.
- **Sharded by region** (§6.6). A node table for North America, Europe, etc.; the CH boundary structure
  stitches cross-shard long-haul routes.

### 4b. The live-traffic overlay (fast-updating, in-memory, ephemeral)

Access pattern: **write** = aggregated speed for one segment, overwritten every ~minute; **read** =
"current speed multiplier for edge E" during routing.

- A keyed store (Redis / in-memory map, geo-sharded) of `edge_id → {liveSpeed, confidence, ts}`,
  covering only segments with recent probe coverage (a fraction of all edges).
- **TTL'd** — a segment with no recent probes falls back to its **historical/base** speed. The overlay
  is a *patch* on the base graph, never the whole thing.
- Eventually consistent by design (§6.4).

### 4c. Historical speed profiles (for prediction)

| Entity | Key | Notes |
|---|---|---|
| `speed_profile` | `(edge_id, dayType, timeBucket)` | typical speed for this segment at, e.g., Tue 8am. Batch-computed from months of probes. |

Read at routing time when `departAt` is in the future, or to fill gaps where live coverage is thin.
This is the [batch layer of a lambda/kappa pipeline](../prep/18-batch-and-stream-processing.md).

### 4d. Places index (for search/geocoding)

| Entity | Key | Notes / access pattern |
|---|---|---|
| `place` | `place_id` | name, address, lat/lng, category, popularity. Source of truth (NoSQL/document). |
| places **inverted index** | text → place_ids | full-text + prefix (autocomplete) — **Elasticsearch** (§6.6). |
| places **geospatial index** | S2/geohash cell → place_ids | "near me" filtering. |

- **Search picks a different store from routing** because the access pattern is full-text + geo, not
  graph traversal. This is the textbook
  [inverted-index + geospatial-index combo](../prep/12-geospatial-systems.md). Kept in sync from the
  `place` source of truth via CDC/queue.

### 4e. Tiles (static blobs)

Pre-rendered raster/vector tiles named by `(z, x, y, mapVersion)`, stored in **object storage** and
served through a **CDN** (§6.2). The ultimate read-heavy, cache-friendly artifact.

> **The unifying observation:** four of these five stores are **read-optimized artifacts produced by
> offline pipelines** (graph, profiles, places index, tiles). Only the live-traffic overlay is written
> online — and it's overwrite-in-place ephemeral, never the durable truth. *Precompute dominates.*

---

## 5. High-level design (10 min) — the happy path end to end

```
                         READ PLANE (cheap, precomputed artifacts, CDN/cache-fronted)
 User app ──tiles──▶ CDN ──(miss)──▶ Tile Storage (object store, pre-rendered by Render pipeline)
          ──search──▶ API GW ──▶ Search Service ──▶ Places ES index + S2 geo index
          ──route──▶ API GW ──▶ Routing Service ──reads──▶ [ Graph (CH, in-mem, versioned, geo-sharded) ]
                                       │                    [ Live-traffic overlay (Redis, TTL'd)     ]
                                       │                    [ Historical speed profiles               ]
                                       │ calls ETA model
                                       ▼
                                  Route + ETA + turn-by-turn ──▶ user; cache popular OD routes

 ─────────────────────────────────────────────────────────────────────────────────────────────────
                         WRITE / FEEDBACK LOOP (probe firehose → traffic, streaming)
 User apps ──GPS probes (anonymized, batched)──▶ Probe Ingest GW ──▶ Kafka (partitioned by geo)
                                                                          │
                                  Stream processor (map-match probe→edge, aggregate per-segment speed)
                                                                          │ every ~minute
                                                                          ▼
                                                          Live-traffic overlay  ──┐
                                                                          │       │ feeds back as
                                                          (also lands in cold store for batch) │ edge weights
                                                                          ▼       ▼
                                                          Batch job ──▶ Historical speed profiles, ML ETA model

 ─────────────────────────────────────────────────────────────────────────────────────────────────
                         OFFLINE BUILD PLANE (slow, heavy, periodic)
 Raw map data (geo DB / municipal / satellite) ──▶ Graph Build (CH precompute) ──▶ versioned Graph artifact
                                                ──▶ Tile Render ──▶ versioned Tiles
```

**Walk one navigation session through it out loud:**
1. User opens the app → client requests **tiles** for the viewport → **CDN edge** serves them (origin
   barely touched). Pans/zooms fetch more tiles, all edge hits.
2. User types a destination → **Search service** hits the places **inverted index** (prefix match) +
   **geo index** (bias to nearby), returns ranked autocomplete suggestions.
3. User taps "Directions" → `POST /route` → **Routing service** runs a **bidirectional CH query** over
   the in-memory graph for the region(s), reading **live-traffic overlay** weights (falling back to
   **historical profiles** where coverage is thin), calls the **ETA model** to refine arrival time,
   and returns the polyline + turn-by-turn steps + alternatives. Popular OD pairs hit a **route cache**.
4. Navigation begins → client emits **GPS probes** every few seconds (anonymized, batched) → **Probe
   Ingest** → **Kafka** (partitioned by geo) → **stream processor** map-matches each probe to a road
   segment and aggregates **per-segment live speeds** → writes the **live-traffic overlay**.
5. Periodically the client calls `/route/{id}/refresh` with its current position → the service
   recomputes ETA against fresh traffic and, if a materially better path exists or the driver went
   off-route, returns a **reroute**.
6. Asynchronously, the probe stream also lands in cold storage; a **batch job** rebuilds **historical
   speed profiles** and retrains the **ETA model**. Periodically the **graph build** and **tile render**
   pipelines produce a new **map version**, hot-swapped onto serving fleets.

Keep it this simple first; the next section is where the round is won.

---

## 6. Deep dives (15 min) — propose the hard parts

> **Open with:** "The interesting problems are (1) **shortest-path at continental scale** — why
> Dijkstra/A\* can't do it per-request and what contraction hierarchies buy us, (2) the **GPS-probe →
> live-traffic loop**, (3) the **tile CDN path**, and (4) **ETA prediction** blending live, historical,
> and ML. Can I go deep on routing-at-scale and the traffic loop — they're the parts most people
> hand-wave?"

### 6.1 Deep dive — shortest path at continental scale (the routing crux)

**Start from the graph.** The road network is a **directed weighted graph**: nodes = intersections,
edges = road segments, **edge weight = travel time** (not distance — a 1 km highway segment beats a
1 km surface street). Travel time varies by time-of-day and live traffic, so the same topological graph
carries *different weights* per query. ~200M nodes, ~500M edges, continent-spanning (§2).

**Why plain Dijkstra/A\* doesn't scale per request:**
- **Dijkstra** settles nodes in increasing distance from the source — for SF→NYC it expands a roughly
  *circular frontier* that grows until it engulfs the target, settling **tens of millions of nodes**.
  At ~100s of ms–seconds of CPU per query and only ~a handful of such queries/sec/core, ~5k route QPS
  would need an absurd fleet, and p99 blows way past 150 ms.
- **A\*** with a straight-line (great-circle) heuristic helps — it biases the frontier *toward* the
  destination instead of expanding a full circle — but on a continental route it still explores
  millions of nodes, because the heuristic is weak across long distances and around obstacles
  (mountains, water, limited bridges). **A\* shrinks the constant, not the asymptotics.** Still too slow.

> **The key realization:** the road network is **largely static** (topology changes slowly) and queries
> are **repetitive over the same long-haul corridors** (everyone driving the I-5 corridor uses the same
> highway backbone). That screams: **do the long-haul work *once*, offline, and reuse it.** This is the
> precompute-vs-query tradeoff, and it's the whole game.

**Hierarchical routing — the core idea.** Real road networks have **hierarchy**: to drive across a
continent you get onto a highway, stay on highways for the bulk, and exit near the destination. You
don't re-derive "use the interstate" for every query. So:

- **Highway hierarchies / Contraction Hierarchies (CH).** Offline, **rank every node by importance**
  and "**contract**" them from least to most important. Contracting a node removes it and adds
  **shortcut edges** between its neighbors that preserve shortest-path distances (e.g., a shortcut that
  represents "the 40-edge stretch of I-80 between these two interchanges, total 35 min"). After
  contracting all nodes you have the original graph **plus** a hierarchy of shortcuts that let a query
  *skip over* unimportant local roads.
- **The query becomes a bidirectional search that only ever goes "upward" in the hierarchy** — from the
  origin it climbs to important roads, from the destination it climbs likewise, and they meet on the
  highway backbone. Instead of settling tens of millions of nodes, **a CH query touches a few thousand
  edges** → sub-millisecond to low-millisecond. This is the orders-of-magnitude win.

**Graph partitioning into regions/tiles + precomputed boundary distances** (an alternative/complement,
"Customizable Route Planning"-style):
- Partition the graph into **regions/cells**; precompute shortest paths between each region's
  **boundary nodes**. A live query does a small local search inside the origin region, **jumps across
  the precomputed boundary-to-boundary skeleton**, and a small local search in the destination region —
  short local searches stitched to a precomputed long-haul backbone.
- The big practical advantage over plain CH: **separation of topology from weights**. The expensive
  *partition* is computed once (topology rarely changes); a cheaper "**customization**" phase recomputes
  the *weights* on the precomputed structure when traffic changes — minutes, not the full hours-long
  build. This is exactly how you reconcile precompute with **changing live traffic** (§6.4).

> **The precompute-vs-query tradeoff, said explicitly:** "I spend **hours of offline compute and ~2×
> the graph's storage** building contraction shortcuts / boundary distances, in exchange for turning a
> per-request continental shortest-path from **hundreds of millions of node-settles** into **a few
> thousand edge touches** — a sub-100 ms query. The cost is **staleness**: the precomputed structure
> reflects the graph/weights at build time, so live traffic needs a *cheaper weight-only re-customize*
> layered on top rather than a full rebuild. I trade build cost + freshness lag for query latency,
> which is exactly the right trade when the topology is near-static and queries are ~5k/sec."

**Practicalities:** the heavy CH/partition build runs offline per **map version**. Weight customization
(folding live + predicted traffic onto the precomputed structure) runs **far more often** — every few
minutes — and is cheap because the structure is fixed. Routing servers hold the graph + customized
weights **in memory**; queries are CPU-bound graph walks, stateless, trivially horizontally scaled.

### 6.2 Deep dive — map tiles + rendering (the read-heavy path)

This is the highest-*volume* path (~10^5–10^6 req/sec) and the easiest to get right *if* you recognize
it's a **static-asset / CDN** problem, not a compute problem. Ties to
[blob storage + CDN](../prep/14-blob-storage-and-media.md).

- **Tile pyramid.** The world is rendered into a **quadtree of tiles** at discrete **zoom levels**:
  z=0 is one tile for the whole world; each zoom level **quadruples** the tile count (z+1 splits each
  tile into 4). A tile is addressed by `(z, x, y)`. The client requests only the tiles intersecting the
  viewport at the current zoom — a bounded handful regardless of how far you've zoomed in.
- **Pre-render offline, serve static.** Tiles are **pre-rendered in batch** from the map data into
  object storage (raster PNG, or increasingly **vector tiles** the client styles/rotates itself —
  smaller, crisper, fewer zoom levels to store). Rendering on the request path would be insane at this
  QPS; we render once per map version.
- **CDN does the heavy lifting.** Tiles are immutable for a given map version and identical for all
  users → **near-100% CDN cache-hit ratio**. The origin (object store) is hit only on cold misses or
  version bumps. `Cache-Control` + `ETag`; the map version is part of the path/key so a new version is
  a **cache-busting key change**, not an invalidation storm.
- **Cost-shaping:** popular tiles (downtowns, highways at common zooms) are hot at every edge; the
  long-tail (ocean, remote zoom-18 tiles) may be rendered **lazily on first request** and then cached,
  rather than pre-rendering quadrillions of mostly-empty deep-zoom tiles. Vector tiles cut storage
  hugely because the client renders labels/rotation, so you store geometry once and skip per-zoom
  raster variants.

> **Tradeoff sentence:** "Tiles are immutable, user-agnostic, versioned blobs → I push them entirely
> to a **CDN**, render offline per map version, and make the version part of the key so updates are
> cache-busting rather than invalidation. The read firehose never reaches a compute tier. The cost is
> **storage** for the pyramid (mitigated by vector tiles + lazy deep-zoom rendering) and **staleness**
> bounded by the publish cadence."

### 6.3 Deep dive — GPS-probe ingestion firehose (write-heavy)

~2M probes/sec (§2) of data we **aggregate then mostly discard** — structurally the same firehose as
[Uber's location pings](../prep/12-geospatial-systems.md#d2--uber--lyft--real-time-driver-matching-write-heavy-pings--nearby-queries),
but the *destination* is different: Uber overwrites a per-driver position for nearby queries; here we
**aggregate probes per road segment into a live speed**.

- **Anonymized, batched, opportunistic upload.** Privacy first: the client strips identity, batches a
  few readings, and uploads over a persistent/batched channel — **not** 2M HTTP POSTs/sec through the
  request gateway. The ingest gateway is a **dumb edge**
  ([dumb-edge/smart-core](../prep/15-realtime-and-push.md#the-architecture-dumb-edge-smart-core)):
  terminate, validate, drop onto Kafka. No business logic, scales on connection + write count.
- **Kafka, partitioned by geography** ([streaming](../prep/18-batch-and-stream-processing.md)). Geo
  partitioning gives locality (a region's probes land together, near where they'll be consumed and
  where that region's graph shard lives) and absorbs the spiky firehose, decoupling ingest rate from
  processing rate.
- **Map-matching is the hard sub-step.** A raw `(lat, lng)` doesn't say *which road* — GPS is noisy and
  roads run parallel (a frontage road beside a highway). The stream processor runs **map-matching**
  (snap the probe trajectory to the most probable edge sequence, typically a hidden-Markov/Viterbi
  match against the graph geometry) to attribute each probe to a specific `edge_id`. This is the part
  juniors skip — "GPS gives you the road" is false.
- **Aggregate per segment, windowed.** Per `edge_id`, maintain a **sliding/tumbling window** of recent
  matched speeds → robust aggregate (median/trimmed mean to kill outliers) + a **confidence** from
  sample count → write to the **live-traffic overlay** every ~minute. Low-coverage segments get no
  live value and fall back to historical/base speed.
- **Don't persist raw probes hot.** Raw probes are sampled into cold storage for the batch layer
  (historical profiles, model training) and otherwise dropped. The online path keeps only **derived
  per-segment speeds**, which is tiny (a number per covered edge).

> **Tradeoff sentence:** "The probe firehose is **aggregate-then-discard**: anonymized batches → Kafka
> partitioned by geo → map-match each probe to an edge → windowed per-segment speed → overwrite an
> in-memory, TTL'd traffic overlay. I never persist raw probes on the hot path (sample to cold store
> for batch only). I trade per-event durability — which I don't need, a lost probe is one of thousands
> for that segment — for the ability to absorb millions of writes/sec and turn them into a fresh,
> tiny edge-weight overlay."

### 6.4 Deep dive — closing the loop: live traffic → edge weights → ETA (consistency)

Now the two halves meet: the aggregated speeds (§6.3) must **update the routing weights** (§6.1) the
queries read.

- **Base graph (slow) + live overlay (fast).** The CH/partition structure is built per map version
  (hours). It would be insane to rebuild it every minute when traffic shifts. Instead, traffic updates
  are a **weight-only re-customization**: the partitioned/CRP-style structure lets you recompute the
  *weights* on the fixed topology cheaply (minutes), and routing reads `liveWeight = overlay[edge] ??
  historical[edge] ?? baseWeight`. Plain CH is harder to update incrementally, which is itself a reason
  to favor a partition-based structure when live traffic matters — name that tradeoff.
- **Eventual consistency, and that's correct.** Edge weights lag reality by tens of seconds to a couple
  minutes (window + aggregation + customization cadence). **Routing is best-effort:** there is no single
  "right" ETA, only a good-enough one computed fast. Two identical requests moments apart may differ —
  fine. We never need strong consistency anywhere in the read plane.
- **ETA = function of the route's edge weights, refined.** Summing edge travel times gives a first ETA;
  but naive summation **systematically underestimates** (it misses turn/intersection delays, traffic
  light waits, merging, and correlated slowdowns). So ETA gets **refined by a model** (§6.5).
- **The closed loop is elegant — say it:** "The probes we collect from navigating users *become* the
  traffic that improves *everyone's* routes and ETAs, including the next reroute for the very user who
  emitted them. The system gets smarter the more it's used." That feedback loop (and the privacy
  obligations it creates) is a great staff-level observation.

### 6.5 Deep dive — ETA prediction (live + historical + ML)

> **Say this:** "ETA isn't just summing current edge speeds. It blends three sources, and I'd put a
> model on top." Ties to [ML system design](../prep/23-ml-and-ai-system-design.md).

- **Live traffic** (§6.4) covers segments with recent probe coverage — best signal where it exists.
- **Historical speed profiles** (§4c): for segments with thin live coverage, or for **future departures**
  (`departAt`), use the typical speed for *this segment, this day-type, this time bucket* — computed in
  batch from months of probes. A route at 8am Tuesday should assume rush-hour speeds even if you query
  it at midnight.
- **An ML model layers on top** to predict *actual arrival time* from features the simple sum misses:
  per-segment live + historical speeds, road class, number/type of turns and intersections, weather,
  recent trend (speeds rising or falling?), historical residuals on this corridor. Modern systems use a
  **graph neural network over the route's segments** to capture correlations (one jam predicts the next
  segment's slowdown) — but for an interview I'd say "**gradient-boosted trees or a GNN over route
  features; trained offline on the historical probe corpus, with the simple weighted-sum as a fallback
  and sanity bound.**"
- **Serving:** the model runs at route time on the candidate route(s) — low-latency inference, features
  precomputed/cached per segment. **Train offline, serve online**, the standard
  [ML serving split](../prep/23-ml-and-ai-system-design.md). Monitor prediction error (predicted vs
  actual arrival from completed trips) as the model's north-star metric.

### 6.6 Deep dive — geocoding & place search

Search is a **separate subsystem** from routing — full-text + geo, not graph traversal. Ties to
[search](../prep/11-search-systems.md) and [geospatial](../prep/12-geospatial-systems.md).

- **Autocomplete / forward geocoding** ("starbucks", "1600 Amphitheatre"): an **inverted index**
  (Elasticsearch) with **prefix/edge-ngram** analysis for type-ahead, ranked by **relevance × popularity
  × proximity** to the user. Bias to the user's location with a **geospatial filter** (S2/geohash cell
  of the user) so "coffee" returns *nearby* coffee, not global. Same prefix-index machinery as
  [autocomplete](../prep/17-security-and-auth.md) — er, [search-autocomplete](../prep/11-search-systems.md).
- **Reverse geocoding** (lat/lng → address): a **geospatial lookup** — find the containing address/
  parcel polygon or nearest road segment via an S2/geohash index, then format the address.
- **Source of truth → indexes via CDC.** Places live in a document store; changes propagate to the ES
  inverted index and the geo index through a [CDC/queue pipeline](../prep/18-batch-and-stream-processing.md),
  the standard "keep the search index in sync" pattern. Search is read-heavy and tolerates seconds of
  index lag.
- The output of search is **coordinates**, which become the origin/destination handed to **routing** —
  that's the seam between the two subsystems.

### 6.7 Deep dive — sharding the graph, routing requests, caching routes

- **Shard the graph by geographic region** (NA, EU, …; or finer cells). A route within a region is
  answered by that shard entirely in memory. The natural sharding is the same one the CH/partition
  build already produces.
- **Cross-region routes** (e.g., crossing a continent or a region boundary) use the **boundary/skeleton
  structure**: each region exposes its boundary-node distances; a coordinator stitches origin-region →
  inter-region backbone → destination-region. Most routes are intra-region; cross-region is the bounded
  minority, handled by the precomputed boundary graph.
- **Route requests are routed by origin geography** to the owning shard fleet; routing servers are
  **stateless** (graph is immutable read-only data) → behind an LB with health checks, scale by adding
  replicas. The hot path is CPU, so scale = more cores.
- **Cache popular routes.** Common OD corridors (downtown↔airport, major commute pairs) repeat
  constantly. Cache the **route geometry/topology** (which changes only with the map version) keyed by
  rounded OD + mode; recompute only the **ETA** against fresh traffic on a hit. This separates the
  expensive-but-stable part (the path) from the cheap-but-volatile part (the ETA) — a clean
  [caching](../prep/06-caching-deep-dive.md) win. Don't cache the full ETA'd route long, since traffic
  moves.

### 6.8 Turn-by-turn & rerouting (client + server)

- **Turn-by-turn directions** are derived from the route's edge sequence: each node transition becomes
  a maneuver ("turn left onto Main St in 200 m") from the geometry + road names + turn angles. Computed
  **server-side once** at route time and shipped with the polyline; the **client** advances through the
  steps locally using GPS, so it doesn't round-trip the server for every instruction (works offline /
  in tunnels).
- **Rerouting** has two triggers: (a) **off-route** — the client detects the user diverged from the
  polyline (a quick local check, no server needed to *detect*) and requests a fresh route from the
  current position; (b) **better-route-available** — the server, on the periodic `/route/{id}/refresh`,
  notices live traffic made an alternative materially faster and pushes a reroute. Only reroute on a
  **meaningful** improvement (hysteresis) — flapping the route every minute is a terrible UX.
- **Client does the cheap, frequent work; server does the expensive, occasional work** — same
  dumb-edge/smart-core split, applied to navigation.

---

## 7. Wrap-up (3 min) — bottlenecks, failure modes, SPOFs

**Remaining bottlenecks**
- **The graph build** is a long offline job; a continental CH/partition build is hours. It gates how
  fresh the *topology* (and full re-customization) can be. Mitigated by separating topology-build from
  weight-customization so live traffic doesn't wait on it.
- **Cross-region routing** adds latency vs intra-region; the boundary-skeleton keeps it bounded but
  it's the tail of the route-latency distribution.
- **Hot tiles / hot search terms** — handled by the CDN and ES caching, but downtown-at-rush-hour is
  always the stress point.

**Failure modes (name them unprompted, per [Topic 13](../prep/13-resilience-and-failure-handling.md))**
- **Live-traffic pipeline dies (Kafka/stream processor down).** The overlay goes stale → routing falls
  back to **historical profiles, then base speeds**. ETAs get less accurate but **routing still works** —
  graceful degradation, because the overlay is a *patch* on a self-sufficient base graph, not a hard
  dependency. The probe firehose buffers in Kafka and replays on recovery.
- **A routing shard dies.** It's stateless over immutable data → fail over to a replica that loads the
  same graph version; the artifact is read-only so there's no state to reconcile. Cross-region routes
  through that region degrade until a replica is up.
- **CDN edge / region outage.** Other edges serve; clients cache recently fetched tiles locally and can
  render a degraded view; origin object store has high redundancy.
- **Stale map version (new road missing / closed road still routed).** Bounded by publish cadence; the
  live-traffic overlay catches *closures that show up as zero-speed* even before the topology updates,
  which partly self-corrects.
- **Map-matching errors** (wrong road attributed) → bad segment speeds. Confidence weighting + outlier
  rejection + requiring a minimum sample count before trusting a live value contain it.

**Single points of failure & redundancy**
- **The graph-build pipeline** is a logical SPOF for *freshness* (not for serving — serving runs on the
  last good artifact). Keep prior versions for instant rollback; a bad build never reaches the fleet
  because publishing is gated on validation.
- **No global serving SPOF by design** — tiles via multi-edge CDN, routing geo-sharded + replicated,
  search replicated, traffic overlay regional and self-rebuilding from the probe stream. The base graph
  is replicated immutable data.

> **The freshness-vs-staleness tradeoff, the theme of the whole design:** "Everywhere I bought query
> speed with precompute, I paid in staleness — the CH structure, the tiles, the historical profiles are
> all snapshots. I reconcile this with **a fast overlay on a slow base**: an immutable precomputed
> foundation (graph, tiles) plus a cheap, frequently-updated patch (traffic weights, lazy-rendered
> tiles, ETA model) layered on top. That layering is what lets a system built almost entirely from
> offline batch artifacts still feel live."

**With more time:** multi-modal (transit/bike/walk) routing with time-dependent edge weights, predictive
traffic (forecast congestion before it happens and pre-route around it), incident detection from probe
anomalies, lane-level guidance, offline maps (ship a region's graph+tiles to the device), and
fairness/load-balancing of routes so navigation apps don't *create* jams by sending everyone down the
same "shortcut."

---

## What made this staff-level

- **Derived the architecture from one insight:** routing is won at **precompute time**, not query time —
  and justified it with the right number (a Dijkstra over a 500M-edge graph settles tens of millions of
  nodes; CH touches a few thousand). The estimation *forced* contraction hierarchies, not a memorized
  buzzword.
- **Explained *why* Dijkstra/A\* fail at scale** (circular frontier; A\* shrinks the constant not the
  asymptotics) and *what* hierarchical routing/CH/graph-partitioning actually do (contract nodes →
  shortcuts; boundary distances; topology-vs-weight separation) — not just naming "contraction
  hierarchies."
- **Made the precompute-vs-query tradeoff explicit** and tied it to the **live-traffic problem**:
  separating slow topology-build from cheap weight-customization is *why* a precomputed structure can
  still reflect live traffic. That connection is the deep insight.
- **Recognized tiles as a pure CDN/static-asset problem** (versioned, user-agnostic, immutable) and
  search as a *different* subsystem (inverted + geo index), rather than lumping everything into one DB.
- **Treated the probe firehose as aggregate-then-discard** with the **map-matching** hard step called
  out explicitly (GPS ≠ road), and closed the **probe → traffic → ETA → reroute feedback loop** — the
  users' data improves the users' routes.
- **Split consistency deliberately:** routing best-effort, traffic eventual, base map versioned — and
  said *why* there's no place that needs strong consistency, the inverse of the Uber/payments problem.
- **Layered ETA** (live + historical + ML over route features, with a sum-of-weights fallback) instead
  of "sum the edge times," and named turn/reroute as a client/server work split.
- **Named the freshness-vs-staleness theme** that unifies every deep dive, plus failure modes
  (degrade to historical, self-rebuilding overlay) and the absence of a global serving SPOF, before
  being asked.

---

### Self-check before the mock (answer these from memory)
- [ ] Describe the road network as a graph: what are nodes, edges, and the edge weight? Roughly how
      big (nodes/edges/bytes)?
- [ ] Why does plain Dijkstra fail at continental scale? What does A\* improve and what does it *not*?
- [ ] What does contracting a node produce, and how does a CH query exploit the hierarchy to touch a
      few thousand edges instead of millions?
- [ ] State the precompute-vs-query tradeoff in one sentence, including what you pay (build cost +
      staleness) for what you gain (query latency).
- [ ] How do you update routing weights for live traffic *without* rebuilding the whole CH? (topology
      vs weight separation / customization)
- [ ] Walk a GPS probe from device → live edge speed. What's the map-matching step and why is it hard?
- [ ] Why are tiles a CDN problem? What is the tile pyramid / zoom quadtree, and how does versioning
      avoid an invalidation storm?
- [ ] What three sources feed an ETA, and what does the ML model add over summing edge speeds?
- [ ] Why is routing best-effort / traffic eventual — where (if anywhere) do you need strong
      consistency, and why not?
- [ ] How do you shard the graph, handle cross-region routes, and cache popular routes (path vs ETA)?
- [ ] Name three failure modes and how the system degrades (live-traffic down, routing shard down,
      stale map version).
