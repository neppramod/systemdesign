# Topic 4: Sharding & Partitioning Deep Dive

> **Why this matters:** Sharding is the answer the interviewer is fishing for the moment your
> estimation says "one node can't hold the data or the write throughput." But *announcing* "I'll
> shard" is worth nothing. The staff signal is the **shard key**, the **hot-spot reasoning**, and
> the **resharding story** — because that's the part that pages you at 3am in real life. Get the
> key right and the system scales linearly for years; get it wrong and you're doing a six-month
> live migration. This doc is about choosing well and defending it.

---

## Part A — Vocabulary: get the distinctions razor-sharp

These words get used loosely. Say them precisely in the room; it instantly reads as senior.

### Vertical vs horizontal partitioning
- **Vertical partitioning** — split a table by **columns**. Hot, small, frequently-read columns in
  one place; cold, large blobs (the `bio`, the `avatar_blob`, the `description`) in another. Also
  describes splitting one DB into per-feature DBs (the "users service DB" vs "orders service DB"
  decomposition). Solves: row width, mixing access patterns, contention.
- **Horizontal partitioning** — split a table by **rows**. Users 1–1M here, 1M–2M there. Same
  schema, different rows on each node. This is the one people mean 95% of the time.

> Vertical = cut the table top-to-bottom (columns). Horizontal = cut it left-to-right (rows).

### Partitioning vs sharding vs replication
These are **orthogonal axes**, not a hierarchy. A real system does all three at once.

| Term | What it splits | Goal | Same data or different? |
|---|---|---|---|
| **Partitioning** | Rows of a dataset into subsets | Manageability, parallelism | Different rows per partition |
| **Sharding** | Partitions spread across **separate machines** | Scale beyond one node | Different rows per shard |
| **Replication** | Copies of the *same* data on multiple nodes | Durability + read scaling + HA | **Same** data, N copies |

- **Partitioning is the logical concept** (subsetting the data). **Sharding is partitioning where
  the partitions live on different physical nodes** — partitioning *for horizontal scale*. In
  casual usage they're synonyms; the precise distinction is "partition = logical subset, shard =
  that subset placed on its own machine." Postgres partitions a table on one box; sharding puts
  those partitions on different boxes.
- **Replication is about copies, sharding is about division.** Sharding gives you more *capacity*;
  replication gives you *redundancy* and read throughput. They compose: each shard is itself a
  **replica set** (1 leader + N followers). 16 shards × 3 replicas = 48 nodes.

> One line for the room: *"Partitioning divides the data, sharding puts those divisions on separate
> machines, replication copies each division for durability and reads. Every serious deployment is
> a grid: shard across the columns, replicate down the rows."*

---

## Part B — Partitioning strategies

For each: how it routes, **what query patterns it enables vs breaks**, and **its hot-spot risk**.

### 1. Hash partitioning
Apply a hash to the key, mod (or map) into a shard. `shard = hash(key) % N`.

- **Enables:** uniform load distribution; fast **point lookups** by the exact key (`get user 42`).
- **Breaks:** **range queries** — adjacent keys scatter across all shards, so "all users created
  last week" becomes a full scatter-gather. Sorting/ordering is gone at the shard layer.
- **Hot-spot risk:** **low for keys, real for hot *individual* keys.** Good hash spreads keys
  evenly, so no shard is structurally hotter — *unless* one key (one celebrity, one viral tweet) is
  itself enormously popular. Hashing distributes keys, not load-per-key.

### 2. Range partitioning
Contiguous key ranges per shard. `A–F → s0, G–M → s1, …`. (HBase, Bigtable, DynamoDB
internally, range-partitioned Postgres.)

- **Enables:** **range scans and ordered iteration** are cheap and local — "all orders between
  these timestamps," "users with names G–M." This is the whole reason to choose it.
- **Breaks:** uniform distribution, *if* your key space is skewed (most surnames cluster, most
  traffic is "today").
- **Hot-spot risk:** **high.** Any **monotonic** key (timestamp, auto-increment ID, sequential
  order ID) sends *all writes to the last shard* — the dreaded "hot tail." This is the single most
  common range-partitioning footgun.

### 3. Directory / lookup-based partitioning
A **lookup service** (a table) maps key → shard explicitly. The router asks the directory "where
does tenant 9183 live?" and caches the answer.

- **Enables:** **maximum flexibility** — arbitrary placement, easy rebalancing (just update the
  directory entry and migrate that range), per-tenant isolation, mixing strategies.
- **Breaks:** nothing query-wise, but the directory is **another lookup hop** and a potential
  **single point of failure / bottleneck**. Must be replicated and aggressively cached.
- **Hot-spot risk:** **lowest, by design** — you can detect a hot range and reassign it. The cost
  is operational complexity and the directory's own availability.
- Used heavily in **multi-tenant SaaS** ("which DB cluster is this customer on?") and by systems
  that want to decouple key from placement.

### 4. Geo partitioning
Partition by region / locality (EU users in `eu`, US users in `us`).

- **Enables:** **data residency / compliance** (GDPR — keep EU data in EU), and **low latency**
  by keeping data near users.
- **Breaks:** global queries ("count all users") become cross-region scatter-gather; users who
  travel or cross-region operations are awkward.
- **Hot-spot risk:** **regional skew** — your US region may carry 10× the EU region. You still need
  a sub-strategy (hash/range) *within* each region.

> **Say this:** "Hash for even spread and point lookups; range when range scans dominate but watch
> the monotonic-key hot tail; directory when I need flexible placement or multi-tenancy; geo for
> residency and latency. They compose — geo at the top, hash within."

---

## Part C — Consistent hashing, in depth

This is the most asked-about sharding subtopic. Know it cold, including the math.

### Why naive modulo fails on resize
With `shard = hash(key) % N`, the shard for a key depends on `N`. The instant you go from 4 nodes
to 5, **almost every key remaps**. Concretely: a key lands on the same shard after the change only
when `hash(key) % 4 == hash(key) % 5`, which is true for a small fraction of keys. In general,
moving from `N` to `N+1` nodes remaps roughly **`N/(N+1)` of all keys** — ~80% at 4→5, ~90% at
9→10. For a cache that means a near-total **cold cache → thundering herd on the DB**; for a stateful
store it means moving almost all your data. Unacceptable.

> The defect: modulo couples *every* key's placement to the *total node count*. Change the count,
> disturb everything.

### The ring
Consistent hashing breaks that coupling. Hash the **output space into a fixed ring** (say
0 … 2³²−1, wrapping around). Then:
1. Hash each **node** to a position on the ring (`hash(node_id)`).
2. Hash each **key** to a position on the ring (`hash(key)`).
3. A key is owned by the **first node encountered walking clockwise** from the key's position.

Now adding/removing a node only disturbs the arc between it and its predecessor — **the keys
in that one segment move; everyone else stays put.**

### How many keys move on add / remove
With **K** keys and **N** nodes, each node owns ~**K/N** keys.
- **Add a node:** it claims the segment ahead of it, stealing ~**K/N** keys **from a single
  successor**. Only those ~K/N keys move. Compare to modulo's ~K.
- **Remove a node:** its ~**K/N** keys fall to the **next clockwise node**. Again only K/N move.

> Modulo on resize: ~K keys move (almost all). Consistent hashing: ~K/N keys move (one node's
> worth). *That* is the whole point — and it's why caches, Dynamo, Cassandra, CDNs and LBs use it.

### Virtual nodes — and why naive consistent hashing isn't enough
Plain consistent hashing has two problems:
1. **Uneven segments.** With few real nodes, random ring positions create wildly unequal arcs — one
   node owns 40% of the ring, another 5%. Load is lumpy.
2. **No proportional load on removal.** When a node dies, *all* its load dumps on the single next
   node instead of spreading.

Fix: each physical node is hashed to **many positions** on the ring — **virtual nodes** (vnodes),
e.g. 100–256 per physical node. Now:
- Each physical node owns ~100 small scattered arcs → totals **even out** (law of large numbers).
- When a node dies, its 100 arcs are inherited by **100 different successors** → load spreads
  smoothly, no single hotspot.
- **Heterogeneous hardware:** give a beefier box *more* vnodes → it gets proportionally more data.

Cassandra calls these `num_tokens`. Dynamo uses vnodes for the same reasons.

### Worked mental model
Ring is a **clock face, 0–11**. Three nodes hash to positions:
- **A → 1**, **B → 5**, **C → 9**.

Ownership = walk clockwise to the next node:
- Keys at 2,3,4,5 → **B** (first node clockwise is B at 5).
- Keys at 6,7,8,9 → **C**.
- Keys at 10,11,0,1 → **A** (wraps past 12 to A at 1).

Now a key hashes to **3** → owned by **B**.

**Add node D → 7.** Only the arc `(5, 7]` changes hands: keys at 6 and 7 move from **C → D**.
Everything else — A's keys, B's keys, C's keys at 8–9 — **stays exactly where it was**. The key at
3 is still on B. *That* locality is what saves the cache and bounds the migration.

**Remove B (at 5).** B's keys (2,3,4,5) now fall clockwise to **C** (next node at 9). Only B's
arc moves; A, C(other arcs), D untouched. With vnodes, B's load would instead scatter across
many nodes rather than dumping wholesale on C.

> The mental picture to carry into the room: *"Nodes and keys on a clock; a key belongs to the next
> node clockwise; adding a node only steals the slice just behind it. Vnodes = each machine is many
> dots on the clock so the slices even out and a death spreads instead of dumping."*

### Where it's used
- **Amazon Dynamo / Cassandra / Riak / ScyllaDB** — partition placement + replication (replicas =
  next N distinct physical nodes clockwise).
- **Memcached clients / Redis Cluster-style** — distribute cache keys so adding a node doesn't
  cold-start the whole tier.
- **CDNs** — map URLs to edge cache servers.
- **Load balancers** — sticky routing (e.g. Envoy/Nginx "ring hash" / Maglev) so a given client or
  session keeps hitting the same backend even as the pool changes.

---

## Part D — Choosing the shard key (the single most important decision)

> Everything else is reversible-ish. The shard key is **baked into the data layout**; changing it
> later means re-reading and re-writing the entire dataset. Spend your deep-dive time here.

### The four criteria
1. **High cardinality** — many possible values, so you can split arbitrarily fine. `country` (≈200
   values) caps you at 200 partitions and lumps badly; `user_id` has millions.
2. **Even distribution** — values (and the *load* on them) spread uniformly. High cardinality isn't
   enough if 1% of values get 99% of traffic.
3. **Matches the query pattern** — the dominant query should be answerable by hitting **one shard**.
   If you read by `user_id`, shard by `user_id` so a user's read is single-shard. A shard key that
   doesn't match your reads forces scatter-gather on every request.
4. **Avoids cross-shard scatter-gather** — corollary of #3. Co-locate data that's read together.
   (Shard messages by `conversation_id` if you always fetch a whole conversation.)

There's an inherent **tension between #2 and #3**: hashing the key gives even spread (#2) but
destroys locality/range reads (#3). Choosing the key *is* resolving that tension for your dominant
access pattern.

### Anti-patterns (name these as things you'd avoid)
- **Monotonic keys** (timestamp, auto-increment ID, snowflake-by-time) → with range partitioning,
  **all writes hit the last shard** (hot tail). Even with hashing they're fine for spread but
  useless for the locality you presumably wanted. *Fix:* hash them, or prefix/bucket them.
- **Low-cardinality keys** (`status`, `country`, `boolean`) → too few partitions, gross imbalance.
- **Celebrity / hot keys** (`celebrity_user_id` for a follower table, one viral `tweet_id`) → one
  value is so popular its shard melts regardless of how evenly *keys* are spread. Covered next.
- **Mutable keys** — if the key can change, the row has to move shards. Pick something immutable.

### Composite & derived keys
You're not limited to a raw column. Common moves:
- **Composite key**: `(tenant_id, user_id)` — co-locates a tenant, spreads users within it.
- **Hash-prefixed key**: prepend `hash(user_id) % 16` to a timestamp → spreads the monotonic write
  load across 16 buckets while keeping per-bucket time ordering (DynamoDB write-sharding pattern).

---

## Part E — The hot shard / hot key problem

Two distinct failures — don't conflate them:
- **Hot shard** = a *partition* gets disproportionate load. Usually a **bad key** (low cardinality,
  monotonic tail, skewed range). Fix the key / split the range.
- **Hot key** = a *single value* is wildly popular (the celebrity, the trending item). Even a
  perfect hash can't help — all requests for that key, by definition, route to one place.

### Mitigations
| Technique | How it works | Best for |
|---|---|---|
| **Salting / write-sharding** | Append/prepend a random or hashed suffix (`key#0..N`) so one logical key spreads across N physical keys; reads fan out to all N and merge | Hot **write** keys, monotonic keys |
| **Splitting the partition** | Detect a hot range and split it into smaller ranges on more nodes | Hot **shard** from range skew |
| **Dedicated shard** | Give the celebrity its own shard/replica set with extra capacity | A known small set of hot entities |
| **Request coalescing** | Collapse concurrent identical in-flight requests into one backend call (single-flight) | Read stampede on one key |
| **Caching** | Put the hot key in an in-memory cache (or local/edge cache) so reads never reach the shard | Hot **read** keys (the common case) |
| **Replication of the hot key** | Replicate just that key to many read replicas / all cache nodes | Read-dominated hot key |

> The clean framing: *"Hot **shard** is a key-design problem — I fix the key or split the range. Hot
> **key** is a physics problem — one value, one home — so I attack it with caching, replication,
> coalescing, and salting on writes, not by re-sharding."*

---

## Part F — Cross-shard queries

Once sharded, any query that doesn't include the shard key is expensive. Know the patterns.

### Scatter-gather
Query that can't be localized → fan out to **all** shards, gather, merge.
- **Cost:** latency is bounded by the **slowest shard** (tail latency amplification — with 100
  shards, p99-per-shard becomes near-certain *somewhere*). Throughput drops because every query
  touches every node.
- **Mitigation:** design the key so the *dominant* query is single-shard; accept scatter-gather only
  for rare/analytical queries; push them to a separate read-optimized store (search index, OLAP).

### Distributed joins
Joining across shards is the expensive thing sharded DBs try hardest to avoid.
- **Co-locate** rows that join (shard both tables on the same key → join is local per shard).
- Otherwise: **broadcast join** (replicate the small table to every shard) or **shuffle/repartition
  join** (network-shuffle both sides on the join key — costly). This is why sharded OLTP designs
  **denormalize** to dodge joins entirely.

### Secondary indexes across shards — local vs global
You shard by `user_id` but need to query by `email`. Two architectures:

| | **Local secondary index** | **Global secondary index** |
|---|---|---|
| Where it lives | Each shard indexes only its own rows | A separate index, itself partitioned by the **index key** |
| Write | Cheap, local, same partition | Expensive — write goes to one shard, index entry to another (cross-partition write, harder consistency) |
| Read by index key | **Scatter-gather** across all shards | **Single index partition** → targeted |
| Examples | Cassandra secondary index, ES per-shard | DynamoDB GSI, Cassandra materialized views |

> Local index = cheap writes, scatter-gather reads. Global index = targeted reads, expensive
> cross-partition writes (and usually only **eventually consistent**). Pick by whether the
> non-key query is hot. If it's hot, pay the global-index write cost.

---

## Part G — Resharding / rebalancing

The operational nightmare. Demonstrate you've thought past day one.

### Pre-splitting (fixed partition count)
Create **many more partitions than nodes up front** (e.g. 1024 partitions, 16 nodes → 64
partitions/node). Rebalancing = **move whole partitions between nodes**; you never split keys.
- Used by **Riak, Elasticsearch, Couchbase, Kafka (partitions)**. This is the cleanest model.
- Tradeoff: partition count is **fixed at creation** (or expensive to change), so you must size for
  future scale. Too few → can't spread; too many → per-partition overhead.

### Dynamic splitting (HBase / DynamoDB / Bigtable style)
Partitions **split automatically** when they exceed a size/throughput threshold; the new halves
can move to other nodes. Adapts to growth and hot ranges with no manual sizing.
- Tradeoff: splits are operationally heavy moments (compaction, region reassignment), can cause
  latency blips, and a hot key *within* a partition still can't be split below one key.

### Consistent hashing with vnodes
Covered in Part C — adding a node steals ~K/N keys from neighbors automatically. The "rebalancing
strategy" for Cassandra/Dynamo.

> **Never rebalance by `hash % N`.** It's the modulo trap from Part C: changing N reshuffles
> everything. Interviewers love to catch this.

### Live migration with dual writes
The real-world procedure to move/repartition without downtime:
1. **Backfill** — bulk-copy existing data to the new shard layout.
2. **Dual-write** — application writes to **both** old and new locations.
3. **Verify** — compare/reconcile; backfill any gaps from before dual-writes began.
4. **Shadow-read / cutover reads** — read from new, compare to old (dark reads), then flip reads to
   new once confidence is high.
5. **Stop dual-writes**, decommission old.

The hard parts: keeping the two in sync under concurrent writes (idempotency, ordering),
reconciling the backfill-vs-live race, and being able to **roll back** at any step. This is weeks-
to-months of work; that's *why* the shard key choice matters so much.

---

## Part H — Routing: where does the shard map live?

Something must translate key → shard. Three placements:

| Placement | How | Examples | Tradeoff |
|---|---|---|---|
| **Client-side** | Client library knows the topology and routes directly | Cassandra/Redis Cluster smart clients, Dynamo | Fewest hops, lowest latency; but topology logic in every client, hard to update fleets |
| **Proxy / router tier** | A middle tier owns routing; clients are dumb | **Vitess** (MySQL), **Twemproxy / Redis Cluster proxy**, ProxySQL | Clients stay simple, central place to evolve sharding; extra hop + a tier to scale/operate |
| **Coordinator node** | Any node can receive the request and forward to the owner | Cassandra coordinator, MongoDB `mongos` (a router) | Simple clients; coordinator does the fan-out/forward, adds a hop |

- The **shard map / topology** itself usually lives in a strongly-consistent coordination store
  (**ZooKeeper / etcd**) or gossip (Cassandra), so all routers agree on who owns what.
- **Vitess** is the canonical name-drop: it's a routing/sharding layer in front of MySQL (powers
  YouTube/Slack), handling query routing, resharding, and connection pooling transparently.

> Say: *"I'd put routing in a proxy tier like Vitess so clients stay dumb and I can reshard without
> redeploying every service; the topology lives in etcd so routers agree. For ultra-low-latency I'd
> consider a smart client to drop the extra hop."*

---

## Part I — Celebrity problem, worked

**Setup.** Twitter-style follow graph, sharded by `user_id`. The `followers` and per-user write
fan-out are partitioned by user. Most users have hundreds of followers; a celebrity has **80M**.

**What breaks.**
- The celebrity's row/partition is read on every follower's feed load and written-around on every
  one of their posts → **one shard saturates** while others idle. Classic hot shard caused by a hot
  key.
- Fan-out-on-write for a celebrity post = **80M feed inserts** in a burst → write storm concentrated
  by the recipients' shards too.

**Mitigations (combine them):**
1. **Hybrid fan-out** — fan-out-on-write for normal users (cheap, few followers); **fan-out-on-read
   for celebrities** (don't pre-push 80M copies; instead followers *pull* celebrity tweets at read
   time and merge them in). This is the canonical answer — it directly removes the write storm.
2. **Cache the celebrity's recent tweets** hard (edge + app cache) — reads for the hot key never
   touch the shard. Replicate that cache entry everywhere.
3. **Dedicated / isolated capacity** for known-celebrity accounts so their load can't take down
   neighbors on a shared shard.
4. **Request coalescing** on the celebrity read so a spike collapses into one backend fetch.

> The interview move: identify it's a **hot-key** (not hot-shard) problem → re-sharding won't help →
> reach for **hybrid fan-out + caching + isolation**. Stating "this is the celebrity problem and
> re-sharding is the wrong tool" is the staff signal.

---

## Part J — Tradeoffs table

| Strategy | Point lookup | Range scan | Even load | Resize cost | Hot-spot risk | Use when |
|---|---|---|---|---|---|---|
| **Hash** | Fast | Bad (scatter) | Excellent | Bad if `%N`; good with consistent hashing | Low (except hot key) | Point reads, want even spread |
| **Range** | Fast | Excellent | Skew-dependent | Cheap (split a range) | High (monotonic tail) | Range/ordered queries dominate |
| **Directory** | Extra hop | Depends on sub-strategy | Tunable | Cheapest (edit the map) | Lowest (reassign hot range) | Multi-tenant, flexible placement |
| **Geo** | Fast (in-region) | In-region only | Regional skew | Region-level | Regional | Residency/compliance, latency |
| **Consistent hashing** | Fast | Bad | Even (with vnodes) | Excellent (~K/N moves) | Low (except hot key) | Elastic clusters, caches, Dynamo-style |

---

## Part K — How to pick a shard key (checklist)

Run this out loud in the room:
1. **What's the dominant query?** Shard so it hits **one** shard. (Read by `user_id` → shard by
   `user_id`.)
2. **Is the key high-cardinality?** Enough distinct values to split fine?
3. **Is load even across values?** Not just key count — *traffic* per key. Any celebrities?
4. **Is it monotonic?** If yes and you're range-partitioning → hot tail → hash it or bucket it.
5. **Is it immutable?** A changing key means moving rows.
6. **What queries does this key *break*?** Name the scatter-gather victims; decide if they go to a
   secondary index (local vs global) or a separate store.
7. **How will I reshard?** Pre-split, dynamic split, or consistent hashing — pick now.
8. **Where does routing live?** Client / proxy / coordinator.

> If you can answer 1–4 crisply for your chosen key and name the scatter-gather it causes, you've
> hit the staff bar for this topic.

---

### Self-check before the mock (answer from memory)
- [ ] Distinguish partitioning vs sharding vs replication in one sentence each.
- [ ] Why does `hash(key) % N` fail on resize, and roughly how many keys move 4→5 nodes?
- [ ] On a consistent-hashing ring, how many keys move when you add one node, and why?
- [ ] What two problems do virtual nodes solve?
- [ ] State the four criteria for a good shard key.
- [ ] Give three shard-key anti-patterns.
- [ ] Difference between a hot *shard* and a hot *key* — and a different mitigation for each.
- [ ] Local vs global secondary index: which is cheap to write, which is cheap to read?
- [ ] Outline live migration with dual writes (the 5 steps).
- [ ] Where can shard routing live, and name a proxy that does it.
- [ ] Walk the celebrity problem and say why re-sharding is the wrong tool.
