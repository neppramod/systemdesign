# Topic 6: Caching Deep Dive — From CPU Cache to CDN

> **Why this topic earns its keep:** Caching shows up in *every* read-heavy design, and most
> candidates wave at it ("I'll add Redis") without a single tradeoff. Staff-level is the opposite:
> you name the layer, the pattern, the invalidation strategy, the failure mode, and the consistency
> hole — and you do it before the interviewer asks. Caching is also the cleanest place to show you
> understand that **a cache is a correctness liability you accept for latency**. Everything below is
> built so you can derive the right cache for an unseen problem instead of reciting "use Redis."

---

## Part A — Where caches live (the hierarchy)

A cache is just a faster, smaller copy of data placed closer to the reader. The whole game is
**latency vs staleness vs cost**. Walk the hierarchy from the CPU outward; the further from the
consumer, the larger and staler the cache.

| Layer | Example | Typical latency to *hit* | What it saves you | Staleness risk |
|---|---|---|---|---|
| CPU L1/L2/L3 | hardware | ~1–10 ns | RAM round-trip | none (coherent) |
| RAM (vs disk) | OS page cache | ~100 ns | SSD/disk I/O | low |
| Client / browser | HTTP cache, app state | 0 (local) | the entire network round-trip | high (you can't invalidate it) |
| CDN / edge | CloudFront, Fastly, Cloudflare | ~10–50 ms (vs origin RTT) | origin RTT + origin compute | medium |
| API gateway / reverse proxy | Varnish, nginx, Envoy | ~1 ms | service hop + compute | medium |
| App-local (in-process) | Caffeine, Guava, a HashMap | ~100 ns–1 µs | the network hop to Redis | medium–high (per-node, inconsistent) |
| Distributed cache | Redis, Memcached | ~0.5–2 ms (same DC) | the DB query | medium |
| DB buffer pool / query cache | Postgres shared_buffers, InnoDB | ~1 ms (RAM hit inside DB) | disk read | none (it's the DB) |

The orders of magnitude are the point. Memorize this ladder — it's the "numbers every engineer
should know" lens applied to caching:

| Operation | Rough latency |
|---|---|
| L1 cache reference | ~1 ns |
| Main memory reference | ~100 ns |
| Read 1 MB sequentially from RAM | ~10 µs |
| SSD random read | ~100 µs |
| Same-DC network round trip | ~0.5 ms |
| Redis GET (same DC, incl. network) | ~0.5–1 ms |
| DB query (indexed, RAM-resident) | ~1–5 ms |
| DB query (hits disk) | ~10–50 ms |
| Cross-region round trip | ~50–150 ms |

> **Say this in the room:** "Each cache layer buys back a specific latency: the CDN buys back the
> origin RTT, the distributed cache buys back the DB query, the in-process cache buys back the Redis
> hop. I add a layer only when the latency it saves is worth the staleness it introduces — caches
> are the easiest place to ship a correctness bug."

**A practical default for a web service:** browser cache for static assets → CDN for static + cacheable
API responses → app-local cache for tiny, hot, read-mostly config → Redis for shared hot data → and
the DB's own buffer pool underneath. Don't reach for all five reflexively; each is a knob you justify.

---

## Part B — Caching patterns (data flow, consistency, failure, when)

This is the section interviewers probe hardest. The differentiator: knowing **who writes the cache,
who writes the DB, and what happens when one of those fails halfway**.

### Cache-aside (lazy loading) — the default

The application owns the cache. The cache is "dumb storage."

```
READ:   look in cache → HIT: return
                       → MISS: read DB → write cache (with TTL) → return
WRITE:  write DB → invalidate (delete) cache key
```

- **Consistency:** only requested keys are ever cached; data is loaded lazily on first miss.
- **Failure modes:** every cache miss hits the DB (cold start = stampede risk); the cache can drift
  from the DB if a write updates the DB but the delete fails. The classic stale-read race lives here
  (see Part H).
- **When:** read-heavy, the **default for almost every interview**. Resilient — if the cache dies,
  you just serve from the DB (slower, but correct).

> **Say this in the room:** "I'll default to cache-aside with a TTL as a safety net, and on writes
> I **delete** the key rather than update it." Deleting is safer than writing the new value into the
> cache, because a delete loses a stale-overwrite race gracefully (next read repopulates), whereas a
> write can lose to a concurrent write and persist a wrong value.

### Read-through

The cache sits *inline*. The app talks only to the cache; the cache library loads from the DB on a miss.

```
READ:  app → cache → (miss) cache loads from DB, stores, returns
```

- **Consistency:** same as cache-aside on reads, but the load logic is centralized in the cache layer
  (a provider/loader), so it's consistent across all callers.
- **Failure modes:** the cache becomes a dependency on the read path — if it's down, reads fail unless
  you add a fallback. Couples you to a cache that supports loaders (e.g., Caffeine `LoadingCache`,
  some Redis client modules, DAX for DynamoDB).
- **When:** you want the loading logic in one place and a clean app abstraction; common with a caching
  library in front of a single store.

### Write-through

Writes go *through* the cache, which synchronously writes the DB before acking.

```
WRITE: app → cache (write) → cache writes DB synchronously → ack
READ:  served from cache (always warm for written keys)
```

- **Consistency:** cache and DB are consistent at write time (no stale window for written keys).
- **Failure modes:** **higher write latency** (you pay cache + DB on every write); if the DB write
  fails after the cache write, you need a transaction or compensating logic or the two diverge. Caches
  keys that may never be read (write amplification on the cache).
- **When:** read-after-write needs to be fast and consistent, and writes aren't the bottleneck. Often
  paired with read-through.

### Write-back / write-behind

Writes hit the cache and ack immediately; the cache flushes to the DB asynchronously (batched).

```
WRITE: app → cache (ack immediately) → [async, batched] flush to DB
```

- **Consistency:** the DB lags the cache. There is a window where committed-to-cache data is **not
  durable**.
- **Failure modes:** **data loss if the cache node dies before flush** — this is the scary one. Out-of-order
  flushes, dropped batches. You need persistence/replication on the cache to make this safe.
- **When:** write-heavy with tolerance for loss or with a durable cache (e.g., metrics, counters, view
  counts, "likes" you can afford to lose a few of). Great for **absorbing write spikes** and batching
  (turn 10k INCRs into one DB write).

### Write-around

Writes go straight to the DB, skipping the cache; the cache populates lazily on later reads.

```
WRITE: app → DB (cache untouched)
READ:  cache-aside (miss → DB → populate)
```

- **Consistency:** avoids caching write-only data; a freshly written key is a guaranteed miss on first
  read.
- **Failure modes:** "read-after-write" is slow (cold miss); if combined badly with cache-aside you can
  cache a stale value if a read races the write (Part H).
- **When:** write-heavy data that's rarely read soon after writing (logs, audit records, bulk imports).
  Keeps the cache from being polluted by data nobody reads.

**Cheat-sheet:**

| Pattern | Who writes DB | Stale window | Failure risk | Best for |
|---|---|---|---|---|
| Cache-aside | app | yes (until invalidate) | drift if delete fails | general read-heavy (default) |
| Read-through | cache loader | yes | cache on read path | centralized load logic |
| Write-through | cache | none for written keys | write latency, divergence | fast consistent read-after-write |
| Write-back | cache (async) | DB lags | **data loss on crash** | write spikes, counters |
| Write-around | app | first-read miss | stale on read/write race | write-once-read-rarely |

---

## Part C — Invalidation strategies ("why cache invalidation is hard")

> *"There are only two hard things in computer science: cache invalidation and naming things."*
> The reason invalidation is hard, concretely: **the truth lives in two places, writes are
> concurrent, and the network can drop the invalidation message.** Any of those three, alone, is
> enough to serve stale data.

| Strategy | How it works | Pros | Cons / when it bites |
|---|---|---|---|
| **TTL (expiry)** | every key expires after N seconds | dead simple; bounds staleness; self-healing | data stale for up to TTL; mass expiry → avalanche |
| **Explicit invalidation** | app deletes/updates key on write | fresh immediately | the delete can fail or race; you must find *every* key affected |
| **Write-through coupling** | write path updates cache + DB together | no stale window for that key | latency; transactional coupling |
| **Versioned / immutable keys** | key includes a version: `user:42:v7`, `asset.a3f9.css` | never invalidate — just write a new key; old entries age out | need to bump the version everywhere; old keys waste memory until evicted |
| **Event / CDC-driven** | DB change → CDC stream (Debezium/binlog) → invalidator deletes keys | decouples invalidation from app writes; catches *all* writers incl. batch jobs | infra complexity; invalidation lag = stale window |

**The hard part made concrete.** Suppose a user's profile feeds three cached things: `user:42`,
`feed:42` (denormalized name), and a search-index entry. One write to the users table must invalidate
*all three*, possibly across services. Miss one and you serve stale data forever (until TTL, if there
is one). This is why people reach for:

- **TTL as a backstop** even when they do explicit invalidation — so a missed invalidation self-heals
  within a bounded window.
- **CDC-driven invalidation** so there's *one* place that reacts to writes regardless of which service
  or batch job made them.
- **Versioned keys** for content-addressed or immutable data (static assets, config snapshots), which
  sidesteps invalidation entirely.

> **Say this in the room:** "I'll use explicit delete-on-write for freshness, but I always set a TTL as
> a backstop so a dropped invalidation self-heals. For multi-service fan-out, I'd move invalidation to
> a CDC stream so there's a single source of truth for 'data changed.'"

---

## Part D — Eviction policies

Eviction is what happens when the cache is *full* — distinct from invalidation (which is about
*correctness*). Eviction is about *which valid entry to drop*.

| Policy | Evicts | Strength | Weakness |
|---|---|---|---|
| **LRU** (least recently used) | the entry untouched longest | great for temporal locality; simple | a single scan (one pass over many keys) flushes the whole cache |
| **LFU** (least frequently used) | the entry hit fewest times | keeps genuinely hot keys | slow to adapt; old-but-once-hot keys linger; needs aging |
| **FIFO** | oldest inserted | trivial | ignores access pattern entirely |
| **ARC** (adaptive replacement) | balances recency + frequency adaptively | resists scans; self-tuning | patented history, more memory/bookkeeping |
| **TinyLFU / W-TinyLFU** | admission-controlled LFU with a small LRU window | near-optimal hit rates; scan-resistant; tiny metadata (count-min sketch) | more complex; the modern default in Caffeine |

**W-TinyLFU** is worth knowing by name at staff level: it uses a **count-min sketch** to estimate
frequency cheaply, an **admission filter** (a new item only enters if it's "worthier" than the victim),
and a small LRU **window** to handle bursts. It's why Caffeine beats Guava — name-drop it when asked
about in-process caches.

### How Redis `maxmemory-policy` works

Redis enforces a `maxmemory` cap and, when full, evicts per `maxmemory-policy`:

| Policy | Behavior |
|---|---|
| `noeviction` | reject writes with an error (good for a cache you treat as a data store) |
| `allkeys-lru` | LRU across all keys — the typical "use Redis as a cache" choice |
| `volatile-lru` | LRU only among keys with a TTL |
| `allkeys-lfu` / `volatile-lfu` | LFU variants (Redis 4+), better for skewed hot-key workloads |
| `allkeys-random` / `volatile-random` | random victim |
| `volatile-ttl` | evict the key with the shortest remaining TTL |

Redis LRU/LFU are **approximate** — it samples N keys (`maxmemory-samples`, default 5) rather than
maintaining a perfect global order, trading a little accuracy for a lot of speed.

> **Choosing one:** uniform access → LRU is fine. Skewed/Zipfian access with stable hot keys → LFU
> (`allkeys-lfu`). If you use Redis as a primary store, not a cache → `noeviction` and size for the
> full dataset. For in-process caches, just use **Caffeine (W-TinyLFU)** and move on.

---

## Part E — The famous failure modes (and the fixes)

This is the highest-yield section. Each has a name, a cause, and a fix — recite them cold.

### 1. Thundering herd / cache stampede
**Cause:** a hot key expires (or the cache restarts cold) and thousands of concurrent requests all miss
and slam the DB simultaneously to recompute the *same* value.
**Fixes:**
- **Request coalescing / single-flight:** only one request recomputes; the rest wait for and share the
  result (a per-key in-flight lock or `singleflight`).
- **Distributed lock on recompute:** first miss takes a short lock (e.g., `SET key lock NX PX 5000`),
  recomputes, populates; others briefly serve stale or wait.
- **Early / probabilistic recompute:** refresh the key *before* it expires, asynchronously, so it never
  goes cold (XFetch / probabilistic early expiration — recompute with probability rising as TTL nears).
- **Stale-while-revalidate:** serve the stale value and refresh in the background.

### 2. Cache penetration
**Cause:** requests for keys that **don't exist anywhere** (e.g., random/garbage IDs, an attack). Every
one misses the cache *and* misses the DB, so the cache provides zero protection.
**Fixes:**
- **Negative caching:** cache the "not found" result with a short TTL.
- **Bloom filter** in front of the cache: "is this key *definitely not* present?" → if the filter says
  no, skip the DB entirely. (Recall: bloom filters have false positives, no false negatives, no deletes.)

### 3. Cache avalanche
**Cause:** a large set of keys expires at the **same instant** (e.g., everything loaded at deploy with
the same TTL), or the cache cluster dies — and the entire load lands on the DB at once.
**Fixes:**
- **TTL jitter:** add randomness — `TTL = base + rand(0, spread)` — so expirations spread out.
- **Multi-tier / replicated cache** and graceful degradation so a cache outage doesn't equal a DB outage.
- **Circuit breaker** in front of the DB to shed load rather than topple it.

### 4. Hot key
**Cause:** one key is so popular (a celebrity profile, a viral tweet) that the single Redis node/shard
owning it saturates — CPU or network bound on one node.
**Fixes:**
- **Client-side / app-local cache** for the few hottest keys (L1 in front of Redis) — absorbs most reads
  before they ever hit Redis.
- **Replicate the key**: read from replicas, or copy the key to N nodes and read a random one.
- **Key splitting:** `counter` → `counter:0..9`, write to a random shard, sum on read (great for hot
  counters).

### 5. Big key
**Cause:** one value is huge (a multi-MB serialized blob, a list with millions of elements). Reads/writes
of it block Redis's single thread, cause latency spikes, and skew memory across the cluster.
**Fixes:** split it into smaller keys; use a hash with field-level access (`HGET` one field instead of
fetching the whole object); paginate large collections; store the blob in object storage and cache only
a pointer/metadata.

| Failure mode | One-line cause | One-line fix |
|---|---|---|
| Stampede / thundering herd | hot key expires, all miss at once | single-flight / lock / early recompute |
| Penetration | key exists nowhere | bloom filter + negative caching |
| Avalanche | many keys expire together / cache dies | TTL jitter + circuit breaker |
| Hot key | one key saturates one node | local cache + replication + key splitting |
| Big key | one value too large | split / hash fields / externalize |

---

## Part F — Redis vs Memcached

| Dimension | Redis | Memcached |
|---|---|---|
| Data model | rich: strings, hashes, lists, sets, sorted sets, streams, HLL, bitmaps, geo | strings/blobs only |
| Threading | single-threaded core (I/O threads in 6+); atomic by nature | multi-threaded; scales on one box with many cores |
| Persistence | RDB snapshots + AOF; can survive restart | none (pure cache; restart = cold) |
| Replication / HA | replicas + Sentinel + Cluster | none built-in; client-side sharding |
| Eviction | many `maxmemory-policy` options | LRU (slab-based) |
| Memory efficiency | more overhead per key | very efficient slab allocator for simple values |
| Use when | you need data structures, persistence, HA, pub/sub, atomic ops | dead-simple, huge, multi-core single-purpose KV cache |

> **Say this in the room:** "Default to **Redis** — the data structures and HA are worth it and it's
> what everyone runs. I'd reach for **Memcached** only for a pure, very large, multi-core string cache
> where its slab efficiency and multithreading actually matter and I need nothing else." In practice
> Redis wins ~95% of interview scenarios; the value is in *knowing why* Memcached could win.

---

## Part G — Redis deep dive

### Data structures and their interview uses

| Structure | Killer use in a design | Why |
|---|---|---|
| **String** + `INCR` | counters, rate limiting (fixed window), atomic IDs | atomic increment, no read-modify-write race |
| **Hash** | store an object's fields, partial updates | `HGET`/`HSET` one field; avoids big-key full reads |
| **List** | queues, recent-N feeds | `LPUSH`/`LRANGE`/`LTRIM` for a capped timeline |
| **Set** | membership, dedupe, tags, "who's online" | `SADD`/`SISMEMBER`, set intersections |
| **Sorted set (ZSET)** | **leaderboards**, **sliding-window rate limiting**, priority queues, time-ordered feeds | score-ordered; `ZADD`/`ZRANGEBYSCORE`/`ZREVRANK` |
| **HyperLogLog** | **cardinality at scale** — unique visitors / DAU | ~12 KB for billions of items, ~0.81% error |
| **Bitmap** | per-user daily activity, feature flags, presence | 1 bit/user; `BITCOUNT` for "how many active today" |
| **Streams** | event log / lightweight queue with consumer groups | append-only, IDs, at-least-once with acks |
| **Geo** (geohash on ZSET) | "nearby" — drivers, stores | `GEOADD`/`GEOSEARCH` for radius queries |

Two you should be able to *derive* on the spot:

- **Sliding-window rate limiter with a ZSET:** member = request ID, score = timestamp. On each request,
  `ZREMRANGEBYSCORE` to drop entries older than the window, `ZADD` the new one, `ZCARD` to count — allow
  if under limit. Precise sliding window, all atomic in a Lua script or `MULTI`.
- **Leaderboard:** `ZADD board score user`; `ZREVRANK board user` for rank; `ZREVRANGE board 0 9` for
  top 10. O(log n). This is *the* canonical ZSET answer.

### Persistence: RDB vs AOF

| | RDB (snapshot) | AOF (append-only file) |
|---|---|---|
| What | periodic point-in-time fork+dump | log every write command |
| Recovery loss | up to the snapshot interval (minutes) | up to `fsync` policy (≤1s with `everysec`) |
| Restart speed | fast (load one compact file) | slower (replay the log) |
| File size | small | larger (rewritten/compacted periodically) |
| Use | backups, fast restart, tolerate some loss | minimal data loss |

Production often runs **both**: AOF for durability, RDB for fast restart/backups. Remember: Redis
persistence makes it *survivable*, not a system of record — for write-back caching it's what keeps the
"data loss on crash" failure mode in check.

### Replication, Sentinel, Cluster

- **Replication:** async leader→replica. Replicas serve reads (with lag) and provide failover candidates.
  Async means a failover can **lose recent writes** — Redis is not strongly consistent.
- **Sentinel:** monitors the leader, does automatic failover and client discovery. Gives you HA for a
  **single shard** (one dataset that fits on one node).
- **Cluster:** **sharding** across nodes via 16384 hash slots, each slot owned by a primary (with
  replicas). Scales beyond one node's RAM/throughput. Caveats: multi-key ops must land in the same slot
  (use **hash tags** `{user42}:profile`), and cross-slot transactions aren't supported.

> **Say this in the room:** "If the dataset fits on one node, Redis + Sentinel for HA. If it doesn't,
> Redis Cluster for sharding — and I'll design keys with hash tags so multi-key ops stay in one slot."

### Single-threaded implications

The command-execution core is single-threaded, which is *why* operations are atomic without locks — but
it also means **one slow command (a big `KEYS *`, a giant `ZRANGE`, a big key) blocks everything**.
Consequences to state: never run `KEYS` in prod (use `SCAN`); avoid big keys; watch for O(n) commands;
a single CPU core caps throughput per node (scale via Cluster, not bigger boxes).

---

## Part H — Consistency between cache and DB

This is the deepest water and a favorite staff probe. The core issue: a cache write and a DB write are
**two operations on two systems**, and you can't make them atomic cheaply (the **dual-write problem**).

### Why "update DB, then delete cache" is the standard

Consider the options on a write:

- **Update DB, then update cache:** two concurrent writers can interleave so the cache ends up with the
  older write's value while the DB has the newer — persistent inconsistency. Also caches values nobody
  reads.
- **Update cache, then update DB:** if the DB write fails, the cache now holds uncommitted data.
- **Delete cache, then update DB:** a concurrent read can miss, load the *old* DB value, and repopulate
  the cache right before the DB update lands → stale until TTL.
- **Update DB, then delete cache** (Cache-Aside + delete, a.k.a. the standard): the next read repopulates
  with the fresh DB value. Deleting (not writing) means a lost race just causes a recompute, not a wrong
  value.

> **Say this in the room:** "The convention is **write the DB, then delete the cache key** — not update
> it. Delete is idempotent and a lost race degrades to a recompute, while an update can persist a stale
> value. It's still not perfectly consistent — there's a residual race — so I pair it with a TTL backstop
> and, if I need stronger guarantees, CDC-driven invalidation."

### The residual race (know it explicitly)

Even "update DB then delete cache" has a window:

```
1. Reader: cache miss, reads OLD value from DB     (key currently empty)
2. Writer: writes NEW value to DB
3. Writer: deletes cache key                       (no-op, it's empty)
4. Reader: writes OLD value into cache             ← stale, persists until TTL
```

It's rare (requires a read miss interleaving precisely with a write) and TTL bounds it. Mitigations when
you can't tolerate it:
- **Delayed double-delete:** delete the key, write the DB, then delete again after a short delay to clear
  any stale repopulate from an in-flight reader.
- **Short TTL** to cap exposure.
- **Read your own writes** routed to the primary / a fresh read for the writing user.

### CDC-based invalidation (the clean answer at scale)

Instead of the application coordinating two writes, let the **database be the single source of truth** and
derive cache invalidation from its change log:

```
app → write DB only
DB binlog/WAL → CDC (Debezium / Kafka) → consumer → delete/refresh cache keys
```

- Eliminates dual-write coordination in the app; **every** writer (services, batch jobs, manual fixes) is
  captured because it's reading the DB's own log.
- Tradeoff: invalidation lag (the stale window = CDC pipeline latency) and real operational complexity
  (Kafka, connectors, ordering). Use when many writers touch the same data or invalidation correctness
  matters more than simplicity.

---

## Part I — Sizing a cache and measuring it

You can't claim a cache helps without numbers. The two that matter: **hit ratio** and **working-set fit**.

- **Hit ratio** = `hits / (hits + misses)`. Below ~80% on a read-heavy cache, ask whether the cache is too
  small, the TTL too short, or the access pattern too uniform to cache well.
- **Working set:** cache the *hot* subset, not everything. Access is usually **Zipfian** — a small fraction
  of keys gets the bulk of traffic. You often hit 90%+ with a cache far smaller than the dataset.
- **Sizing:** `entries × (avg value size + key + overhead)`, then add headroom for eviction churn and
  fragmentation. For Redis, budget ~1.5× your raw data for overhead and `maxmemory` below physical RAM.

### The marginal value of cache

Hit ratio has **diminishing returns**: going 90% → 95% may double the cache size while the *effective*
latency improvement shrinks. Reason about expected latency:

```
E[latency] = hit_ratio × cache_latency + (1 − hit_ratio) × db_latency
```

At 90% hit with 1 ms cache / 20 ms DB: `0.9×1 + 0.1×20 = 2.9 ms`. At 95%: `0.95×1 + 0.05×20 = 1.95 ms`.
At 99%: `1.19 ms`. The first 90% bought you the most; chasing the last few percent costs RAM for shrinking
gains. **Say:** "I'd size for the working set to hit ~90–95%, then stop — beyond that the marginal RAM
isn't worth the marginal latency."

> **Watch for:** a high *hit ratio on garbage* (caching things rarely re-read) wastes memory; a low hit
> ratio means the cache is mostly overhead on the read path. Measure hits, misses, evictions, and p99 with
> and without the cache.

---

## Part J — CDN specifics

A CDN is a geographically distributed cache at the network edge. It buys back the **origin round trip and
origin compute** for cacheable responses, and serves bytes from a PoP near the user.

### Push vs pull

| | Push CDN | Pull CDN |
|---|---|---|
| How | you upload content to the CDN ahead of time | CDN fetches from origin on first request (lazy), then caches |
| Pro | content ready instantly; control over what's cached | no upload step; self-managing; only popular content cached |
| Con | you manage population + invalidation; wasted storage for cold content | first request per edge is a slow miss to origin |
| Use | large, stable, predictable assets (software releases, video libraries) | typical websites/APIs (the common default) |

### Cache-Control headers (the contract)

The CDN and browser obey HTTP cache headers — know these:

- `Cache-Control: public, max-age=3600` — cacheable by shared caches for an hour.
- `s-maxage=86400` — separate, longer TTL for *shared* (CDN) caches vs the browser.
- `no-store` — never cache (sensitive/personalized). `no-cache` — cache but **revalidate** before use.
- `private` — browser only, not the CDN (per-user data).
- `ETag` / `Last-Modified` + conditional `If-None-Match` → **304 Not Modified** revalidation without
  re-sending the body.
- `stale-while-revalidate` — serve stale while fetching fresh in the background (kills edge stampedes).

### Edge invalidation

CDNs are large and distributed, so invalidation is slow and sometimes costly:
- **Purge / invalidation API** — explicitly evict a path; can take seconds to minutes to propagate and may
  be rate-limited or billed.
- **Versioned URLs** (the better pattern) — `app.a3f9c.js`, `/v2/logo.png`. New content = new URL = no
  invalidation needed; old URLs age out. This is why build tools hash asset filenames.

### When a CDN helps vs not

| Helps | Doesn't help (or hurts) |
|---|---|
| Static assets (JS/CSS/images/video) | Highly personalized, per-user responses |
| Geographically dispersed users | Single-region users near the origin |
| Cacheable, read-heavy API responses | Rapidly changing / uncacheable data |
| Absorbing traffic spikes at the edge | Write-heavy endpoints |
| Large media (offloads origin bandwidth) | Anything requiring strong freshness |

> **Say this in the room:** "I'll put static and media behind a pull CDN with long `max-age` and
> **versioned/hashed URLs** so I never have to purge. Personalized responses get `Cache-Control: private`
> and stay off the edge. For semi-static API data I'd use a short `s-maxage` with `stale-while-revalidate`
> to absorb spikes without serving badly stale data."

---

## Part K — How to use this in the room

1. **Name the layer before the product.** "This is read-heavy with global users → CDN at the edge,
   Redis for shared hot data, maybe an L1 in-process cache for the hottest keys."
2. **State the pattern + the delete-not-update rule.** "Cache-aside, delete on write, TTL backstop."
3. **Volunteer one failure mode unprompted.** "The risk here is a stampede on this hot key, so I'd add
   single-flight / early recompute."
4. **Own the consistency hole.** "This is eventually consistent; the residual stale window is bounded by
   the TTL; if that's unacceptable I'd go CDC-driven."
5. **Put a number on it.** "~90% hit ratio takes p99 from 20 ms to ~3 ms; chasing 99% isn't worth the RAM."

> **Mental checklist for any caching decision:**
> Which layer? → Which pattern (and do I delete or update)? → How do I invalidate, and what's the
> backstop? → Which failure mode is most likely here? → What's the consistency window, and is it
> acceptable? → What hit ratio do I expect, and is the cache worth it?

---

### Self-check before the mock (answer these from memory)
- [ ] List the cache layers from CPU to CDN and the latency each one buys back.
- [ ] Describe all five patterns and which one risks **data loss on crash**.
- [ ] Why "update DB then **delete** cache," and what's the residual race?
- [ ] Name the five famous failure modes and one fix for each.
- [ ] Explain Redis `maxmemory-policy` and when you'd pick LFU over LRU.
- [ ] Give the ZSET design for a leaderboard *and* a sliding-window rate limiter.
- [ ] RDB vs AOF, and Sentinel vs Cluster — when each.
- [ ] When does a CDN *not* help, and why are versioned URLs better than purges?
- [ ] Write the expected-latency formula and explain the marginal value of cache.
