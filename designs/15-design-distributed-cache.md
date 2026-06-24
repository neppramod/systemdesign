# Design 15: Distributed Cache (Redis / Memcached-cluster-style)

> **Why this one matters:** this is the *infrastructure-primitive* design — the box that shows up
> inside almost every other system in this folder (Twitter feed, URL shortener, Instagram, flash
> sale). Building *the cache itself* forces you to own the parts you usually hand-wave: how keys
> map to nodes without a global reshuffle on resize (consistent hashing — sharding doc 04), how a
> shard survives a node death (replication + failover — replication doc 05), how memory is reclaimed
> under pressure (eviction — caching doc 06), and the failure modes a cache *must* survive (hot key,
> stampede, big key — caching doc 06). It also has a sharp single-node-systems flavor most candidates
> miss: the **event-loop concurrency model** and **why a cache persists (or deliberately doesn't).**
>
> The one-line thesis I'll state up front and keep returning to: **a cache is an AP, in-memory,
> horizontally-partitioned store whose correctness model is "lose data freely, never lose
> availability or latency."** Every decision flows from that. We optimize sub-millisecond latency and
> aggregate memory; we treat the data as *regenerable from the source of truth*, so durability is a
> nice-to-have, not a requirement. That single inversion (data is disposable) is what separates a
> cache design from the KV-store design (15 vs 05): same ring, *opposite* priorities.
>
> I run the full 7-step framework but spend most of the budget in **Step 6 (deep dives)** — that's
> where the round is won.

---

## Step 1 — Requirements (5 min) — I drive this

I'll state the buckets out loud and scope hard, because "a cache" is unbounded.

**Functional**
- `GET(key)` → value or miss. Point lookup by key only.
- `SET(key, value, ttl?)` → ack. Optional TTL (expiry).
- `DEL(key)` → ack.
- A few atomic extras that callers genuinely rely on: `INCR/DECR` (counters), `SETNX` (set-if-not-exists → distributed lock primitive), `EXPIRE`. I'll mention these because they're *why* people pick Redis over a plain hashmap, but I'll keep the data plane simple.
- **Explicit non-goals I'll call out:** no SQL, no secondary indexes, no cross-key range scans, no multi-key transactions *across shards* (single-shard MULTI/EXEC is fine), no joins. This is a key-value cache — the access pattern is "I know the key."

**Non-functional** — this is where the design is actually decided:
- **Sub-millisecond latency** — p99 GET in the **~0.1–1 ms** range on the server, single-digit-ms end-to-end including network. This is the headline. It rules out anything synchronous and cross-region on the hot path, and it's *the* reason the data lives in RAM, not on disk.
- **High throughput** — **hundreds of thousands to millions of ops/sec** aggregate. A single node does ~100k–200k ops/sec on a Redis-style event loop; millions ⇒ we *must* shard.
- **Large aggregate memory** — the working set is bigger than one machine's RAM (say hundreds of GB to multiple TB). One node can't hold it ⇒ we *must* partition memory across nodes.
- **High availability** — the cache is on the critical read path of everything upstream; if it's down, the origin DB gets the full unfiltered load and may topple (a cache outage is often *worse* than no cache — see Step 7). Target 99.99%. So each shard needs a replica + automatic failover.
- **Weak consistency is acceptable** — stale/lost cache entries are fine; the source of truth is the DB. We will *deliberately* trade consistency and durability for latency and availability. This is the inversion vs the KV store (05).
- **Eviction under memory pressure** — RAM is finite and the dataset doesn't fit, so the cache *must* evict. The eviction policy is a first-class requirement, not an afterthought.
- **Read-heavy** — caches exist because reads dominate (often 100:1). Writes are cache *population* and invalidation.

> **The staff move in Step 1:** I'm *positioning* the system before drawing anything. The thesis —
> **"data is disposable; never sacrifice latency or availability for it"** — is what makes every later
> choice derivable. Per **PACELC**: under Partition we choose **A** over C (serve possibly-stale,
> never block); and Else we choose **L** over C (no synchronous cross-node coordination on a GET). So
> the cache is a **PA/EL** system, same family as Dynamo — but with the extra freedom that we may
> *drop* data entirely, which a database may not.

---

## Step 2 — Estimations (3 min) — to justify, not impress

Let me pick numbers that *force* the architecture, so nothing is arbitrary. The chain is:
**working-set size → number of nodes → memory per node → QPS per node.**

**Working set → node count (the memory-bound calc):**
- Suppose the application caches **500M hot objects**, average serialized value **2 KB**, plus ~100 B key + per-entry overhead (Redis object header, expiry, dict bucket) ≈ **~2.2 KB/entry**.
- Raw working set ≈ `500M × 2.2 KB` ≈ **1.1 TB** of *logical* cache data.
- **Replication factor 2** (one replica per shard for HA) → **~2.2 TB physical RAM**.
- **Usable RAM per node:** a 64 GB box, but you *never* fill a cache node to 100% — fragmentation + replication buffers + fork-for-snapshot copy-on-write headroom mean you plan for **~60–70% usable**, call it **~40 GB usable per node**.
- Nodes for capacity ≈ `2.2 TB / 40 GB` ≈ **~55 nodes**. Round up for headroom and even shard counts → **~64 nodes (32 shards × 2 copies)**.

| Quantity | Value | Drives |
|---|---|---|
| Hot objects | 500M | working-set size |
| Avg entry (value + key + overhead) | ~2.2 KB | memory math |
| Logical working set | ~1.1 TB | — |
| × replication (RF=2) | ~2.2 TB physical | node count |
| Usable RAM/node (~65% of 64 GB) | ~40 GB | node count |
| **Nodes** | **~55–64** | cluster topology |

**QPS → per-node throughput (the throughput-bound calc):**
- Target **3M GET/sec** peak aggregate, ~30k SET/sec (read:write ≈ 100:1).
- Spread over **32 primary shards** → `3M / 32` ≈ **~94k GET/sec per primary node**.
- A single-threaded Redis event loop sustains ~**100k–200k** simple ops/sec → **~94k fits, but with little headroom.** That's a useful finding: at this QPS we are *both* memory-bound and throughput-bound, and a single hot shard at 94k is close to the ceiling — so **hot-key handling (6F) is not optional.**

| Quantity | Value | Drives |
|---|---|---|
| Peak GET/sec | 3M | shard count |
| Primary shards | 32 | — |
| GET/sec per primary | ~94k | near single-thread ceiling ⇒ hot-key risk |
| Per-node ceiling (Redis-style) | ~100–200k ops/s | concurrency model (6H) |

**Bandwidth sanity check:** `3M ops/s × 2.2 KB` ≈ **~6.6 GB/s** aggregate egress → ~200 MB/s per node → comfortable on 10–25 GbE NICs, but big values would change this (see big-key, 6F).

> What the math *justifies*: ~60 nodes ⇒ a hand-maintained shard map is a liability ⇒ **we need a
> systematic data-distribution scheme (consistent hashing / hash slots)** and **automated failover**.
> The 94k-per-shard number ⇒ **a single hot key can saturate a whole node**, which pre-justifies the
> hot-key deep dive. The estimate did its job: it made partitioning, HA, *and* hot-key mitigation
> *necessary*, not assumed.

---

## Step 3 — API design (3 min)

Deliberately tiny — the contract *is* the simplicity argument.

```
GET key                       -> value | (nil)            # point read
SET key value [EX ttl]        -> OK                        # upsert, optional expiry
DEL key                       -> count
EXPIRE key ttl                -> 0|1
INCR key / DECR key           -> integer                   # atomic counter
SETNX key value               -> 0|1                       # set-if-absent: lock primitive
```

Two contract decisions I'll flag now and justify later:
1. **TTL is per-key and first-class.** TTL is the cheapest, most important invalidation mechanism a cache has — "let it expire" sidesteps the hard distributed-invalidation problem for most data (**caching doc 06**). I'll lean on it heavily.
2. **A few atomic ops (`INCR`, `SETNX`) are exposed.** These are *only* trivially atomic because of the single-threaded execution model (6H) — no locks needed, because there's no concurrency *within* a node. That's a design property worth surfacing, and it's why Redis (not Memcached) is the lock/counter substrate.

> Auth/TLS/rate-limiting terminate at the client library or a thin proxy; I won't re-explain them
> (**API-gateway doc**). The interesting surface here is the data plane and the cluster control plane.

---

## Step 4 — Data model (5 min)

- **Logical model**: opaque keys → values. In Memcached, values are opaque blobs. In Redis, values are *typed* (string, hash, list, set, sorted-set, stream) — richer, but each value still lives entirely on **one** shard (a key is the unit of distribution; you cannot split one value across shards).
- **Access pattern**: 100% point access by key. **No requirement reads by anything but the key.** Per the **databases doc decision flow**, "known access pattern, point lookups, latency-critical, eventual consistency acceptable" → in-memory KV. The access pattern *chose* the store.
- **What each entry needs stored with it**:
  - the value,
  - an **expiry timestamp** (for TTL),
  - **LRU/LFU metadata** — a last-access clock tick or a frequency counter (8-bit, see 6E) used by eviction,
  - in Redis, the type tag.
- **Key design is the user's job, and it's load-bearing:** the cardinality and access skew of the key space directly cause (or avoid) hot keys and big keys (6F). I'll note good practice: bound value sizes, avoid one giant collection key, use a hash-tag (`{user123}`) when you *need* related keys co-located on one shard for a multi-key op.

> **Why I'm not normalizing anything:** there are no relations. The model is intentionally
> denormalized blobs/objects — read speed over write simplicity (**framework tradeoff list:
> normalization vs denormalization**). The *application* decides what to denormalize into a cache
> entry; the cache just stores bytes.

---

## Step 5 — High-level design (10 min) — happy path end to end

Boxes and arrows first; deliberately simple, then evolve under questioning.

```
                          ┌── routing: hash(key) → shard ──┐
   app ── GET/SET ──▶  [ client / proxy / smart-client ]   │
   (cache-aside)               │                           │
                               ▼                           │
                  ┌────────────────────────────────────────┐
                  │   the cluster: 32 shards (key space)    │
                  └────────────────────────────────────────┘
                       │ shard 7        │ shard 19   ...
                       ▼                ▼
                  ┌─────────┐      ┌─────────┐
                  │ PRIMARY │      │ PRIMARY │     each shard:
                  │ (RAM)   │      │ (RAM)   │     1 primary + 1..k replicas
                  └────┬────┘      └────┬────┘     async replication
                       │ async          │
                  ┌────▼────┐      ┌────▼────┐
                  │ REPLICA │      │ REPLICA │     replica = read scaling + failover
                  └─────────┘      └─────────┘
```

**Key architectural properties of the happy path:**
- **The key space is partitioned across shards** (32 of them). Routing is `hash(key) → shard`; *which mechanism does the routing* (client, proxy, or smart client) is the topology question in 6B.
- **Each shard is a small replicated unit:** one **primary** (serves writes + reads) and one or more **replicas** (async copies for failover and optional read scaling). Within a shard this is leader-follower replication (**replication doc 05, Part A.1**).
- **The data lives in RAM**, single-threaded event loop per node (6H). No disk on the hot path.

**The pattern the cache must support — cache-aside (look-aside), the default:**
```
read:   v = cache.GET(key)
        if v == miss:                       # cache miss
            v = db.read(key)                # go to source of truth
            cache.SET(key, v, ttl)          # populate (lazy)
        return v

write:  db.write(key, v)                    # write source of truth first
        cache.DEL(key)                      # invalidate (not update) — avoids races
```
I'll name the alternatives (**caching doc 06**) and *why I default to cache-aside*:
- **Cache-aside (look-aside):** app owns the cache; cache is dumb. Resilient (cache down ⇒ app still reads DB), simple, but first read is always a miss and there's a known invalidation race. **Default.**
- **Read-through / write-through:** the cache layer itself loads/writes the DB. Simpler app code, but couples the cache to the DB and the cache must understand the data. Write-through keeps cache fresh at the cost of write latency.
- **Write-back (write-behind):** write to cache, flush to DB async. Fast writes + absorbs spikes, but **a cache node death loses un-flushed writes** — only acceptable for tolerant data (counters, metrics). Directly conflicts with our "data is disposable" thesis, so I use it *only* where loss is OK.

> **Why `DEL` not `SET` on write:** invalidating (delete) then lazily repopulating on next read is
> more robust than updating the cache in place — two concurrent writers updating the cache can
> interleave and leave a stale value pinned, whereas delete-then-miss always reloads the latest from
> the DB. This is the classic invalidation-race nuance from the **caching doc 06**.

**Walking one request out loud:** app `GET user:42` → client hashes `user:42` → routes to shard 19's primary → primary checks its dict in RAM → hit ⇒ return in ~0.2 ms; miss ⇒ app reads DB, `SET`s back with a TTL. Everything else (how routing survives resize, how shard 19 survives its primary dying, how RAM is reclaimed) is the deep dive.

---

## Step 6 — Deep dives (the bulk of the round)

I'll propose the order: **(A) data distribution via consistent hashing + vnodes → (B) cluster topology: client-side vs proxy vs smart-client/hash-slots → (C) replication & HA per shard → (D) failover & the consistency-vs-availability tradeoff → (E) eviction & memory management → (F) the failure modes a cache must survive (hot key, stampede/herd, big key) → (G) single-threaded vs multi-threaded concurrency model → (H) persistence: should a cache persist? → (I) adding/removing nodes & slot migration → (J) the full read/write path end to end → (K) when NOT to add a cache.** That's the dependency order: you can't talk topology before you've decided how keys map to nodes, can't talk failover before replication, can't talk eviction before the memory model.

---

### 6A. Data distribution: consistent hashing with virtual nodes

**The problem:** ~60 nodes, every key must map to a shard, and **adding/removing a node must move as little data as possible** — because in a cache, *moved or dropped data is a cold miss*, and a flood of cold misses stampedes the origin DB (the very thing the cache protects). So resize disruption isn't just a data-movement cost; it's a *correctness/availability* risk for the whole system.

**Why naive `hash(key) % N` fails (sharding doc 04, Part C: "why naive modulo fails on resize"):**
- With `shard = hash(key) % N`, changing `N` (add/remove a node) changes the modulus, so **almost every key remaps to a different shard.** Concretely: go from 4 → 5 nodes and roughly **80%** of keys move.
- For a *database* that means a giant reshuffle. For a *cache* it's worse: every remapped key is now a **miss** on its new node ⇒ the entire fleet goes cold at once ⇒ a synchronized **stampede onto the origin DB** ⇒ the DB may fall over. Naive modulo turns "add a node" into "outage."

**Consistent hashing — the ring (sharding doc 04, Part C):**
- Hash the key space onto a fixed ring `[0, 2^32)` (or 2^64) with a uniform hash (Murmur/CRC).
- Hash each node to one or more positions on the ring.
- A key is owned by the **first node clockwise** from `hash(key)`.
- Add/remove a node → **only the keys between the new node and its predecessor move.** On average only **K/N** keys relocate, not all of them. So adding a node cold-misses ~1/N of keys instead of ~all — a survivable trickle to the DB, not a flood.

**Why plain consistent hashing isn't enough — virtual nodes (sharding doc 04, Part C):**
With one ring position per node you get two problems:
1. **Uneven load** — random placement leaves big gaps; some nodes own huge arcs (and thus more memory + more QPS), others tiny. Memory imbalance means one node OOMs and evicts heavily while another is half empty.
2. **Lumpy rebalancing** — when a node dies, its *entire* arc (and all its cold-miss load) dumps onto exactly one successor.

**Fix:** give each physical node **V virtual nodes** (e.g. 100–256 ring positions per node).
- **Even load:** each node owns many small scattered arcs ⇒ by law of large numbers everyone owns ~1/M of keys *and* ~1/M of memory.
- **Heterogeneity:** a bigger-RAM node just gets more vnodes ⇒ proportionally more data. Free weighting — useful when the fleet is mixed hardware.
- **Graceful rebalancing:** when a node dies, its ~200 small arcs spread across ~200 different successors, so the cold-miss recovery load is shared by the whole cluster, not dumped on one neighbor.

> **Tradeoff stated:** vnodes cost a bigger ring/membership table and more bookkeeping, but they buy
> even memory + even QPS + smooth rebalancing + heterogeneity. Easy yes at ~60 nodes. This is the
> **consistent-hashing-with-vnodes** strategy from **sharding doc 04, Part G**. Note Redis Cluster
> uses a *fixed* 16384-hash-slot variant of the same idea (6B) — slots are essentially a fixed, large
> set of vnodes, which makes the slot→node map small and explicit.

---

### 6B. Cluster topology: client-side sharding vs proxy vs smart-client (hash slots)

Three industry approaches to *where the routing logic lives*. This is a core cache-specific decision; I'll compare and pick.

**Option 1 — Client-side sharding (consistent hashing in the client library).**
- Each client embeds the ring and hashes keys itself, connecting directly to the owning node. (Classic Memcached deployments, `ketama` hashing.)
- **Pros:** zero extra hops ⇒ lowest latency; no middle component to scale or fail; dead simple servers (servers are dumb, independent caches).
- **Cons:** **every client must agree on the ring** — membership changes require coordinating *all* clients; no single source of truth for topology; clients in different languages can hash inconsistently (split-brain on the key map ⇒ duplicated/missed entries); operational changes (add a node) are a fleet-wide client config push.

**Option 2 — Proxy / router tier (Twemproxy, Envoy, mcrouter).**
- Clients talk to a stateless proxy; the proxy holds the ring and routes to the right node; you scale the proxy tier behind an LB.
- **Pros:** clients are trivial (one endpoint); topology lives in one place (the proxy config) ⇒ adding a node is a proxy reconfig, not a client push; proxy can do connection pooling, request coalescing (helps thundering herd, 6F), and even cross-shard scatter-gather.
- **Cons:** **an extra network hop** (+0.2–0.5 ms — meaningful when the target is sub-ms); the proxy tier is another thing to run, scale, and make HA (it must not become a SPOF — run N of them behind an LB); proxies are usually stateless routers and don't themselves handle failover, so you still need a control plane for membership.

**Option 3 — Smart-client cluster mode with hash slots (Redis Cluster).**
- The key space is split into a **fixed 16384 hash slots**; `slot = CRC16(key) % 16384`. Each primary owns a contiguous **range of slots**. The **slot→node map is gossiped among the nodes themselves** (Redis Cluster bus), and clients **cache** that map.
- A client hashes the key → slot → looks up the node in its cached map → connects directly (no proxy hop). If it guesses wrong (map is stale mid-migration), the node replies `-MOVED <slot> <node>` (permanent reassignment) or `-ASK <slot> <node>` (slot is *mid-migration*), and the smart client updates its map and retries. So the client is *eventually* correct without a central router.
- **Pros:** no proxy hop (direct to node) *and* no fleet-wide config push (clients self-correct via MOVED/ASK + gossip); the cluster owns its own topology and failover. Best of both — the production default for Redis at scale.
- **Cons:** the **client library must be cluster-aware** (more complex client); multi-key ops only work if keys are in the same slot (forces hash-tags `{...}`); the 16384-slot granularity caps how finely you can rebalance.

| | **Client-side hashing** | **Proxy (Twemproxy/Envoy/mcrouter)** | **Smart-client / hash slots (Redis Cluster)** |
|---|---|---|---|
| Where routing lives | in every client | in a middle tier | in the cluster + cached in client |
| Extra hop | none | **+1 hop** | none |
| Topology source of truth | clients must agree (none) | proxy config | gossiped slot map (self-correcting) |
| Add/remove node | push to all clients | reconfig proxies | live slot migration + MOVED/ASK |
| Failover handling | external | external | **built in (gossip + promotion)** |
| Client complexity | medium (ring lib) | trivial | high (cluster-aware) |
| Best for | simple Memcached fleets | polyglot clients, want thin clients, coalescing | large Redis deployments wanting self-managing HA |

> **What I'd pick and why:** for our ~60-node, HA-critical, sub-ms target I'd choose **smart-client
> hash-slot mode (Redis Cluster)** — it avoids the proxy hop (protecting the latency budget) *and*
> gives built-in failover and self-correcting routing (no fleet-wide config pushes). I'd add a
> **proxy tier (Envoy/mcrouter) only if** my clients are polyglot/thin or I want request coalescing
> as a stampede defense — accepting the extra hop as the cost. **Tradeoff named:** slots cap rebalance
> granularity and force hash-tags for multi-key ops; I accept that because single-key point access is
> 99% of cache traffic.

---

### 6C. Replication for HA: primary/replica per shard

A node *will* die (process crash, host failure, OOM-kill). If a shard is a single node, its death drops ~1/32 of the cache cold *and* loses the ability to serve those keys until a replacement warms up — a partial outage and a DB stampede. So each shard is **primary + ≥1 replica.**

- **Within a shard: leader-follower (primary-replica) replication** (**replication doc 05, Part A.1**). The primary takes writes; replicas receive a stream of changes.
- **Replication is asynchronous** (Redis default). The primary acks the client immediately and ships the write to replicas in the background. **Why async, not sync:** synchronous replication would add a cross-node round-trip to *every* SET, blowing the sub-ms budget. Given "data is disposable," we don't pay that — we accept that a primary can ack a write and die before the replica sees it (that write is lost; the DB still has the truth on next miss).
- **How a replica catches up:** a new/restarted replica does a full sync — the primary forks and snapshots its dataset (RDB, 6H), ships it, then streams the backlog of commands buffered since the fork (the replication backlog buffer). Steady state is a continuous command stream.
- **Replicas double as read-scaling** for read-heavy hot shards: clients may read from replicas (`READONLY` in Redis Cluster), accepting **replication lag → stale reads.** Fine for a cache; the data was already a possibly-stale copy of the DB.

> **Cross-reference:** this is **replication doc 05, Part A.1 (leader-follower)** made concrete, with
> the cache-specific choice of **async over sync** justified by the latency budget and the disposable-
> data thesis. Contrast the KV store (05) which uses *leaderless* quorum replication for tunable
> consistency — a cache doesn't need tunable consistency, so it picks the simpler, faster leader-
> follower model.

---

### 6D. Failover: detection, promotion, and the consistency-vs-availability tradeoff

When a primary dies we must **promote a replica** automatically and fast (HA is 99.99%). Two industry mechanisms:

**Redis Sentinel (the classic, for non-clustered / simple setups):**
- A separate fleet of **Sentinel** processes monitors primaries. A Sentinel marks a primary **subjectively down** on missed heartbeats, then they vote; once a **quorum** of Sentinels agree it's **objectively down**, they run a leader election among themselves (a small Raft-like vote) and the elected Sentinel **promotes a replica** to primary and reconfigures the others + notifies clients. Quorum (≥ majority of Sentinels) prevents a single confused Sentinel from triggering a failover.

**Redis Cluster gossip (built into cluster mode — what I'd use here):**
- Nodes gossip health over the **cluster bus**. When enough primaries agree a primary is failing (`PFAIL` → `FAIL`), the dead primary's **replicas request votes from the other primaries**; a replica that wins a majority promotes itself, claims the dead primary's hash slots, and gossips the new map. Clients learn via gossip + `-MOVED`. No separate Sentinel tier — the cluster self-heals.

**The consistency-vs-availability tradeoff on failover (replication doc 05, Part H — this is the crux):**
Because replication is **async**, a failover can **lose acknowledged writes**:
- Primary P accepts `SET x=5`, acks the client, but **dies before** replicating to replica R.
- R is promoted; it has the *old* `x` (or no `x`). The acked write `x=5` is **gone.**
- This is the classic async-replication data-loss window. For a database it'd be a correctness bug; **for a cache it's acceptable** — the DB still has the truth, and the next read just misses and repopulates.

And the **split-brain** danger:
- Network partition isolates P from the majority. The majority side promotes R. If P keeps accepting writes on the minority side (clients still reaching it), now **two primaries** own the same slots → divergent data → on heal, one side's writes are discarded.
- **Mitigation:** a primary that **can't reach a quorum of the cluster stops accepting writes** (`cluster-require-full-coverage` / min-replicas-to-write). This is choosing **C over A** *on the minority side* to prevent split-brain — even a cache draws the line at "two authoritative copies." We sacrifice availability on the isolated minority to avoid serving two contradictory views.

> **The honest framing (replication doc 05, Part H):** failover is exactly where a cache makes its
> CAP choice concrete. **Normal operation:** PA/EL — async replication, accept lost-write-on-failover
> for latency. **Under partition:** the *minority* side must stop writing (C over A locally) to avoid
> split-brain; the *majority* side stays available. The tunable knob is `min-replicas-to-write` /
> failover timeout: dial it conservative (slow failover, fewer false promotions, fewer lost writes) or
> aggressive (fast failover, risk flapping + more lost writes). For a cache I lean **aggressive**
> (fast failover, availability first) because lost writes are cheap.

---

### 6E. Eviction policies and memory management

RAM is finite and the working set doesn't fit (Step 2) ⇒ the cache **must evict**. This is where `maxmemory` and the policy choice live (**caching doc 06**).

**`maxmemory` + policy:** each node has a `maxmemory` cap. When a write would exceed it, the node applies an **eviction policy** to free space before accepting the write (or it rejects writes if policy is `noeviction`). The policies:

| Policy | What it evicts | Use it for |
|---|---|---|
| **LRU** (Least Recently Used) | the entry unused for longest | general default; assumes recency predicts reuse |
| **LFU** (Least Frequently Used) | the entry accessed least often | skewed workloads where a few keys are hot long-term; resists a one-off scan evicting your hot set |
| **TTL-based** (`volatile-ttl`/`volatile-lru`) | among keys *with* a TTL, the soonest-to-expire / LRU | when you tag cacheable data with TTLs and want only those evicted |
| **Random** (`allkeys-random`) | a random key | cheapest; when access has no useful pattern |
| **noeviction** | nothing — writes error | when stale-but-present beats evict; e.g. a cache used as a lock store |

**Why approximate LRU/LFU (a key implementation nuance):** true LRU needs a doubly-linked list touched on every access — too much memory + pointer churn at millions of ops/sec. Redis uses **sampled (approximate) LRU**: store a 24-bit access clock per object, and on eviction **sample K random keys (default 5–10)** and evict the oldest of the sample. Near-LRU quality at O(1) memory and tiny CPU. **LFU** similarly uses an **8-bit probabilistic frequency counter** (logarithmic increment + time-based decay) so it doesn't saturate and can forget old popularity.

**TTL is the other half of memory management:** expired keys are removed two ways — **lazy** (checked and dropped on access) and **active** (a background cycle samples keys with TTLs and expires a fraction each tick). Lazy alone would leak memory for keys never read again; the active sweep bounds that.

**Memory management beyond eviction:**
- **Fragmentation:** the allocator (jemalloc) leaves gaps; `used_memory` < `RSS`. Plan capacity at ~65% (Step 2). Redis can do active defrag.
- **Eviction storms:** if `maxmemory` is hit hard, every write triggers sampling+eviction → latency spikes + a wave of misses. Mitigate by alerting on memory headroom and scaling out *before* the cliff.

> **Tradeoff stated (caching doc 06):** **LRU vs LFU** is recency-vs-frequency — LRU is simpler and
> handles changing hotsets, but a big sequential scan (a batch job touching every key once) **pollutes**
> LRU and evicts your real hot set; LFU resists that. I'd default to **LFU for skewed read-heavy
> caches** (most of them) and LRU when the hotset shifts quickly. **The deeper point:** eviction is a
> *cache-specific* concern that a durable KV store (05) never has — there, you never silently drop
> data. This is the clearest place the "data is disposable" thesis shows up in the machinery.

---

### 6F. The failure modes a cache must survive: hot key, thundering herd / stampede, big key

These are the cache-specific pathologies (**caching doc 06**). Consistent hashing spreads *keys* evenly but does **not** save you from a single *value* being pathological. Naming and mitigating these is a strong staff signal.

**Hot key — one key gets a disproportionate share of traffic.**
- Symptom: a celebrity's profile, a viral post, a global config flag → that single key lives on **one** shard (one key = one slot = one node), and that node's event loop saturates (recall Step 2: a shard is already near 94k ops/sec — one hot key can take it over the ceiling). **Consistent hashing solves hot *shards*, not a hot *single key*** (**sharding doc 04, Part E**).
- **Mitigations:**
  - **Client/local cache (near-cache):** clients cache ultra-hot keys in process for a few seconds; absorbs the bulk of reads before they ever hit the cache. Accept brief extra staleness.
  - **Key replication/fan-out:** store the hot value under N suffixed keys (`config:v1#0..#7`) hashed to different shards; readers pick a random replica. Spreads the read load across nodes. Cost: N× writes/invalidations.
  - **Read from replicas:** route hot-key reads across the shard's replicas, multiplying read capacity for that shard.

**Thundering herd / cache stampede — a hot key expires (or the cache is cold) and N concurrent requests all miss simultaneously and hammer the DB.**
- Symptom: TTL on a popular key fires → 10,000 in-flight requests all miss → 10,000 identical DB reads at once → DB melts. A cache *outage* (cold restart) is the extreme version: the whole fleet stampedes.
- **Mitigations (caching doc 06):**
  - **Request coalescing / single-flight:** the first miss takes a per-key lock (e.g. `SETNX lock:key`); it alone hits the DB and repopulates; the others wait briefly and re-read the cache. (A proxy tier, 6B, can do this centrally.) One DB read instead of 10,000.
  - **Probabilistic early expiration (XFetch):** recompute the value *before* the TTL fires, with a probability that ramps up as expiry approaches, so one request refreshes it early while others still serve the old value — no synchronized cliff.
  - **Stale-while-revalidate:** serve the stale value past TTL while one background request refreshes it. Availability over freshness.
  - **Jittered TTLs:** never set the same TTL on a batch of keys populated together (they'd all expire at once → synchronized stampede). Add random jitter so expiries spread out.
  - **Warm-up on cold start:** pre-load the known-hot set before taking traffic; ramp traffic gradually.

**Big key — one value is huge (a multi-MB blob, or a Redis collection with millions of elements).**
- Symptom: a single 50 MB value or a 10M-element set. It (a) skews memory onto one shard, (b) makes every op on it slow (and on a *single-threaded* node, one slow op **blocks every other request** on that node, 6H), (c) blows bandwidth (`DEL` of a giant key can stall the loop).
- **Mitigations:** bound value sizes at write time; **shard the big value** across keys (chunking) or use Redis's lazy/async deletion (`UNLINK`) so freeing a big key doesn't block the loop; for big collections, model them as multiple smaller keys.

> **The unifying point:** these are *data-shape* failures, not topology failures — they slip past
> consistent hashing because they're about skew in *access* and *size*, not in key *count*. A
> mid-level answer stops at "consistent hashing balances load"; the staff answer is "balances key
> count, but I still have to defend against hot keys, stampedes, and big keys explicitly." All four
> trace to the **caching doc 06** failure-mode list.

---

### 6G. Concurrency model: single-threaded event loop (Redis) vs multi-threaded (Memcached)

A genuinely interesting single-node systems tradeoff, and a favorite follow-up.

**Redis — single-threaded event loop (for command execution):**
- One thread runs an epoll/kqueue event loop: accept connections, read requests, **execute commands one at a time**, write responses. (Modern Redis offloads *network I/O* to helper threads and does background tasks — RDB fork, AOF fsync, lazy-free — on other threads, but **command execution is serialized on one core.**)
- **Pros:** **no locks, no lock contention, no race conditions** inside the data structures ⇒ simpler, fewer bugs, and **every command is trivially atomic** (this is *why* `INCR`/`SETNX`/`MULTI` are atomic with zero locking — there's no concurrency to guard against). Predictable, low tail latency; cache-friendly (one core's L1/L2). Easier to reason about.
- **Cons:** **a single core caps throughput** (~100–200k ops/sec, Step 2) ⇒ you scale by **sharding across nodes/processes, not by adding cores.** And **one slow command blocks everyone** — a big-key op or a `KEYS *` scan stalls the whole node (ties straight back to big-key, 6F). You run multiple Redis processes per host to use multiple cores.

**Memcached — multi-threaded:**
- A pool of worker threads; connections are spread across them; internal data structures are guarded by **fine-grained locks** (per-bucket / item locks).
- **Pros:** **uses all cores in one process** ⇒ higher single-instance throughput for a simple GET/SET workload; vertical scaling within a box.
- **Cons:** **lock contention** under high concurrency on hot buckets; more complex internals; and the value model is **opaque blobs only** — no atomic server-side data-structure ops, so you can't build counters/locks the way Redis does.

> **The framing I'd give:** Redis bets that **horizontal sharding beats vertical threading** for a
> cache — keep each node simple, lock-free, and predictable, and add nodes for throughput (which we're
> already doing for memory anyway). Memcached bets on **using the whole box** for raw simple-KV
> throughput. For our design — rich types, atomic ops as a feature, and we're sharding for memory
> regardless — **single-threaded Redis-style is the better fit**, and it makes the hot-key/big-key
> defenses (6F) *more* important because one slow op blocks the whole node. If the workload were
> pure opaque GET/SET at extreme per-node throughput with no need for atomic ops, Memcached's
> multithreading would win on hardware efficiency. **Tradeoff named, side picked.**

---

### 6H. Persistence: RDB vs AOF — and should a cache persist at all?

The thesis says **data is disposable**, so the honest first answer is: **a pure cache often shouldn't persist** — the source of truth is the DB, and persistence costs IO, fork-pause latency, and disk. But there's one strong reason a cache *does* persist: **avoiding a cold-start stampede.** If a node restarts empty, every key on it misses and stampedes the DB (6F). Persistence lets it **reload its warm dataset on restart** instead of cold-missing the whole fleet. So persistence here is a *warm-restart* feature, not a durability guarantee.

**RDB — point-in-time snapshot:**
- Periodically **fork** the process and dump the whole dataset to a compact binary file (copy-on-write means the child sees a frozen snapshot while the parent keeps serving).
- **Pros:** compact, fast to load on restart (great for warm-up), minimal steady-state overhead.
- **Cons:** **you lose everything since the last snapshot** on a crash (could be minutes); the **fork can cause a latency blip** on a huge dataset (COW page-copying under write load, and on overcommit a big fork can OOM).

**AOF — append-only file (command log):**
- Append every write command to a log; replay it on restart. `fsync` policy tunes durability (`always` = safest/slowest, `everysec` = ~1s loss window, the common choice).
- **Pros:** much smaller loss window (≤1s with `everysec`); a true-ish durability story.
- **Cons:** larger files, slower restart (must replay) — mitigated by **AOF rewrite/compaction**; per-write fsync overhead competes with the latency budget.

**Common production choice:** **both** (RDB for fast reloads + AOF for a small loss window), or **neither** on a pure throwaway cache where the replica is the only redundancy and cold-start is handled by gradual warm-up.

> **The staff answer to "should a cache persist?":** *It depends what the cache is for.* As a pure
> look-aside cache of a durable DB → **persistence is optional and primarily a warm-restart / anti-
> stampede tool, not durability** (the DB is the truth). The moment the cache holds data with **no
> other source of truth** — Redis used as a primary store, a session store, a job queue, a rate-limit
> counter you can't recompute — persistence becomes **mandatory** and you'd run AOF+RDB and treat it
> like a real database. Knowing *which mode you're in* is the call. (Cross-ref **caching doc 06**: a
> cache is defined by having a backing store; once it doesn't, it's a database and inherits database
> obligations.)

---

### 6I. Adding/removing nodes: rebalancing and slot migration with minimal disruption

The cache *will* be resized (scale out for memory/QPS, replace dead hosts). The hard constraint, from 6A: **minimize cold misses during the move**, because cold misses stampede the DB.

**With hash slots (Redis Cluster) — live slot migration:**
- To add a node, you **reassign some slots** from existing primaries to the newcomer. A slot migrates *live*, one slot at a time:
  1. Source marked `MIGRATING <slot>`, target marked `IMPORTING <slot>`.
  2. Keys in the slot are moved in batches (`MIGRATE`), copying values over.
  3. **During migration, requests for a not-yet-moved key are served by the source; requests for an already-moved key get `-ASK` redirecting the client to the target for that one key.** So the slot stays *fully available* throughout — no downtime, no mass cold-miss.
  4. When the slot is empty on the source, ownership flips and the new map gossips out; clients update via `-MOVED`.
- Because data is *copied*, not dropped, **migrated keys stay warm** — the great advantage of slot migration over a naive rehash: we don't cold-miss the moved keys.

**With consistent hashing + vnodes (Memcached-style):** adding a node steals a slice of each of many arcs (vnodes spread the source). Those keys become misses on the new node and **repopulate lazily from the DB** — a *trickle* (~1/N of keys) rather than the *flood* naive modulo would cause. To avoid even the trickle stampeding, ramp the new node in and/or warm it.

**Removing a node:** in cluster mode, migrate its slots to peers first, then decommission (failover its replicas if it was a primary). Never just kill a primary — that drops its slots cold until failover and forfeits the warm data.

> **Staff nuance — temporary vs permanent (mirrors KV doc):** don't rebalance for a *temporary* blip
> (reboot, GC, brief partition) — sloppy failover / replica promotion (6D) covers it, and moving GBs
> of RAM for a 30-second outage is pure waste that itself destabilizes the cluster. Rebalance only for
> *permanent* membership changes (capacity add, dead host replacement), and do it via **live slot
> migration so keys stay warm.** Auto-rebalancing on a false-positive failure detection triggers a
> data-movement storm — gate it behind conservative timeouts.

---

### 6J. The full read/write path, end to end

**`GET key` (the dominant path):**
1. Client computes `slot = CRC16(key) % 16384`, looks up the owning **primary** in its cached slot map, connects directly (no proxy hop in cluster mode).
2. (If the map is stale mid-migration → node replies `-MOVED`/`-ASK`; client updates its map and retries. Self-correcting.)
3. Primary's event loop reaches the command, looks up the key in its in-RAM dict.
4. **Hit:** check TTL (lazy expiry); if live, bump LRU/LFU metadata, return value in ~0.1–0.3 ms. Optionally served from a **replica** for read scaling (accepting replication-lag staleness).
5. **Miss:** return nil → the *application* (cache-aside) reads the DB and `SET`s back with a jittered TTL. Stampede defenses (6F: single-flight lock) gate the DB read for hot keys.

**`SET key value EX ttl`:**
1. Route to the owning **primary** (writes never go to replicas).
2. If accepting the write would exceed `maxmemory`, the primary **evicts** per policy (sampled LRU/LFU, 6E) to make room — or errors if `noeviction`.
3. Write applied to the in-RAM dict; **acked to the client immediately** (no waiting on replicas — async).
4. The write **streams asynchronously to replicas** (6C) and, if enabled, appends to the **AOF** (6H).
5. On a primary failure before the replica caught up, that write is **lost on failover** — acceptable (6D), DB has the truth.

> Walking this end-to-end and pausing at each step to name *which property it buys* — direct routing
> (no hop → latency), MOVED/ASK (self-correcting topology → no config push), sampled eviction (bounded
> memory), async ack (sub-ms writes, at the cost of failover loss), single-flight on miss (DB
> protection) — is the move that shows you understand the cache as a *whole*, not a bag of features.

---

### 6K. When NOT to add a cache (the senior judgment call)

The strongest cache-design signal is knowing when a cache is the *wrong* tool — adding one is not free.

- **Write-heavy / low-reread data:** a cache pays off only when an entry is read many times before it changes. If every read is unique (or data churns faster than it's reused), the cache is all misses + invalidation overhead — pure cost. Don't cache.
- **Strong-consistency reads:** if a stale read is a correctness violation (account balance, inventory count at checkout), a look-aside cache's invalidation race makes it unsafe. Read from the DB (or use a CP store) — don't paper over it with a cache.
- **The cache becomes a SPOF / load amplifier:** if losing the cache means the DB instantly dies under unfiltered load, the cache hasn't *added* resilience — it's *hidden* a capacity gap. A cache should *reduce* origin load you can also survive without (degraded), not be load-bearing for survival. If you can't survive a cold cache, you have a capacity problem the cache is masking.
- **Premature caching:** if the DB comfortably serves the load, a cache just adds an invalidation problem ("one of the two hard things") and a staleness surface for no benefit. Profile first.

> **The decision rule:** *Is this data read far more than written, tolerant of staleness, and is the
> origin able to survive a cold cache?* All three yes → cache it. Any no → reconsider. This is the
> **caching doc 06** "when not to cache" rule, stated as a gate.

---

## Step 7 — Wrap-up (3 min): tradeoffs, failure modes, SPOFs

**The thesis held:** an AP, in-memory, partitioned store where **data is disposable** — we optimized latency, throughput, and aggregate memory, and we traded consistency + durability away wherever it bought us those. Every block served it: async replication (latency over durability), eviction (drop data freely), aggressive failover (availability over lost writes), persistence only for warm-restart.

**Failure modes to name before asked:**
- **Node dies:** replica promoted via gossip/Sentinel (6D); the in-flight async-replicated writes are lost (acceptable — DB is truth). Mitigate cold-miss with persistence/warm-up.
- **Split brain (partition):** two primaries for the same slots → divergence. Mitigated by `min-replicas-to-write` + requiring quorum to accept writes — the minority side stops writing (C over A locally), 6D.
- **Hot key / stampede / big key:** the data-shape failures (6F) consistent hashing doesn't catch — near-cache, key fan-out, single-flight, jittered TTLs, chunking.
- **Eviction storm / OOM:** hitting `maxmemory` causes latency spikes + a miss wave; alert on headroom, scale out before the cliff (6E).
- **Cold-start stampede:** an empty node (or fleet) floods the DB; persistence reload + gradual warm-up + single-flight (6F/6H).
- **Replication lag:** reading from replicas serves stale; fine for a cache, but don't read-your-own-writes from a replica.

**SPOFs and how I remove them:**
- *Per-shard:* primary is a SPOF → **replica + auto-failover** (6C/6D).
- *Routing tier:* a single proxy is a SPOF → run **N proxies behind an LB**, or use smart-client cluster mode (no proxy) (6B).
- *Control plane:* a single Sentinel/config node is a SPOF → **quorum of Sentinels** or **gossip** (no central control plane) (6D).
- *Whole-cache dependency:* the cache itself is a SPOF for the *system* if the DB can't survive without it → ensure the origin can serve degraded (the "don't make the cache load-bearing" rule, 6K).

**What I'd tune for different workloads:**
- **Read-heavy app cache (default):** cache-aside, LFU eviction, RF=2 async, jittered TTLs, smart-client cluster mode.
- **Session store (no other source of truth):** treat as a *database* — AOF+RDB persistence mandatory, more replicas, slower/conservative failover.
- **Counter/rate-limit store:** lean on atomic `INCR` + single-threaded atomicity; persistence depends on whether the count is recomputable.
- **Throwaway hot-read cache at extreme per-node QPS, opaque values, no atomic ops:** consider **Memcached** (multi-threaded) for hardware efficiency.

**With more time I'd add:** cross-region replication (active-passive with async geo-replication, or CRDT-based active-active à la Redis Enterprise), client-side near-caching tier with invalidation push, tiered storage (RAM + SSD for cold entries), and observability (per-shard hit rate, p99, eviction rate, memory headroom) — because a cache you can't observe is a stampede waiting to happen (**resilience doc 13**).

---

## What made this staff-level

- **Stated a thesis up front — "data is disposable; never sacrifice latency/availability for it" (PA/EL)** — and derived every choice from it: async replication, eviction, optional persistence, aggressive failover all fall out of one inversion. Positioning > enumeration.
- **Let estimation *force* the architecture:** the memory math made partitioning necessary; the 94k-per-shard number *pre-justified* the hot-key deep dive before it was asked. The math did work, it didn't just decorate.
- **Compared three real topologies (client-side / proxy / hash-slots) and picked one with a reason** — most candidates name only "consistent hashing" and stop; comparing routing *placement* and the MOVED/ASK self-correction shows production depth.
- **Made the failover consistency tradeoff explicit** — async replication ⇒ lost-write-on-failover, and split-brain ⇒ minority must stop writing (C over A locally). Naming *exactly where* the cache makes its CAP choice is the senior move.
- **Owned the cache-specific failure modes** (hot key, stampede, big key) as *data-shape* problems consistent hashing doesn't solve — with concrete, distinct mitigations for each.
- **Connected single-node systems to distributed concerns** — the single-threaded event loop isn't trivia: it's *why* atomic ops are free, *why* one big-key op blocks the node, and *why* we scale by sharding not threading. Tying the concurrency model to hot-key/big-key defenses is full-stack depth.
- **Answered "should a cache persist?" with judgment, not dogma** — persistence is a warm-restart/anti-stampede tool for a look-aside cache, but *mandatory* the moment the cache is the only source of truth.
- **Closed with "when NOT to add a cache"** — knowing a cache can be a SPOF, a load-amplifier, or premature is the strongest seniority signal.

---

## Self-check (answer from memory before the mock)

- [ ] State the cache thesis in one line, and give the PACELC classification with justification.
- [ ] Run the estimation chain: working-set → nodes → memory/node → QPS/node. Why does it pre-justify hot-key handling?
- [ ] Why does `hash % N` fail on resize, and why is it *worse* for a cache than for a DB? How many keys move with consistent hashing?
- [ ] Two reasons plain consistent hashing isn't enough; how vnodes fix both. How do hash slots relate to vnodes?
- [ ] Compare client-side sharding vs proxy vs smart-client/hash-slots — hops, topology source of truth, failover. Which would you pick and why?
- [ ] Why async (not sync) replication per shard? What exactly is lost on failover, and why is that OK for a cache?
- [ ] Explain split-brain on failover and the `min-replicas-to-write` mitigation — which CAP side does it pick, and where?
- [ ] LRU vs LFU vs TTL eviction — when each? Why *approximate* (sampled) LRU/LFU? What is `maxmemory`?
- [ ] Define hot key, thundering herd/stampede, and big key — and give a distinct mitigation for each. Why doesn't consistent hashing solve them?
- [ ] Single-threaded event loop vs multi-threaded — pros/cons, why atomic ops are free on Redis, why one big op blocks the node.
- [ ] RDB vs AOF; should a cache persist? State the rule for when persistence becomes mandatory.
- [ ] How does live slot migration keep keys warm during a resize? Temporary vs permanent — when do you rebalance?
- [ ] Walk the GET and SET paths end to end, naming what each step buys.
- [ ] Give the "when NOT to add a cache" gate in one sentence, and name how a cache can become a SPOF.
