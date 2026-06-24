# Topic 3: Databases Deep Dive — Picking the Right Store and Knowing Why

> **Why this topic matters:** "Which database?" is the question where most candidates leak signal.
> They name-drop a store ("I'll use Cassandra") with no derivation, then can't defend it when the
> interviewer pushes. Staff-level is the opposite: you derive the store *from the access pattern*,
> you know *why the storage engine underneath behaves the way it does*, and you can model the
> schema concretely. This doc gives you the decision machinery, the engine internals that justify
> it, and one fully worked DynamoDB schema you can reproduce in the room.

---

## Part A — Choosing a Datastore From Access Patterns

The mistake is choosing a *technology* first. The discipline is: enumerate access patterns, then
let the store fall out. Go back to framework step 4 — **how is each entity read and written?**

For every entity, answer four questions out loud:

1. **Read shape** — point lookup by key? range scan? full-text? "nearby"? aggregation?
2. **Write shape** — append-only? update-in-place? high-cardinality updates?
3. **Relationships** — do you join across entities, or is each access self-contained?
4. **Consistency need** — must a read see the latest write (balance), or is stale OK (feed)?

### The decision flow (say this as a sequence, not a memorized verdict)

| If the dominant access pattern is… | …reach for | Because |
|---|---|---|
| Point get/put by a known key, sub-ms, ephemeral | **Key-value cache (Redis)** | No query planner, in-memory, O(1) |
| Point get/put by key, durable, huge scale, predictable | **DynamoDB / wide-column** | Partitioned KV, horizontal scale |
| Rich ad-hoc queries, joins, multi-row transactions | **Relational (Postgres/MySQL)** | Joins + ACID + flexible querying |
| Write-heavy, time-ordered, range scans by partition | **Wide-column (Cassandra/Bigtable)** | LSM writes, partition+clustering keys |
| Nested/variable documents read whole, few joins | **Document (MongoDB)** | Schema flexibility, locality |
| Deep relationship traversal (friends-of-friends) | **Graph (Neo4j)** | Index-free adjacency |
| Append-only metrics, downsampling, time-window queries | **Time-series (Timescale/Influx)** | Time-partitioned, rollups |
| Full-text / fuzzy / relevance-ranked search | **Search (Elasticsearch)** | Inverted index, scoring |
| Large immutable bytes (images, video, backups) | **Object store (S3)** — *not a DB* | Cheap, durable, served via CDN |

> **Say this in the room:** "The access pattern is point-lookup by user ID at very high write
> volume with no joins, so a relational DB's join engine and single-writer ACID are dead weight —
> I'll use a partitioned KV/wide-column store. If the requirement had joins or multi-entity
> transactions, I'd flip back to Postgres."

### SQL vs NoSQL beyond the cliché

The tired version is "SQL = structured, NoSQL = scale." That's wrong enough to hurt you. The real
distinctions:

- **SQL gives you a query planner.** You can ask questions you didn't anticipate at schema-design
  time. NoSQL generally makes you *design the schema around the queries you already know* — change
  the access pattern later and you may have to remodel or migrate.
- **SQL gives you multi-row ACID transactions on one node trivially.** Most NoSQL stores give you
  single-item atomicity and bolt on limited transactions later (DynamoDB TransactWriteItems,
  Mongo multi-doc txns) with cost and constraints.
- **NoSQL was built to scale horizontally by default.** Relational DBs *can* shard (Vitess, Citus,
  Aurora) but it's bolted on and cross-shard joins/transactions hurt. NoSQL assumes partitioning
  from day one — which is exactly why it makes you commit to a partition key early.
- **"Schemaless" is a half-truth.** The schema doesn't vanish; it moves from the database into your
  application code. You still pay for it — just at read time, in every service that parses the doc.

> **The honest framing:** Modern Postgres handles JSON, full-text, geospatial, and scales to
> surprisingly large single instances. "Default to Postgres until an access pattern forces you off
> it" is a *defensible* staff answer. Reach for NoSQL when a specific axis — write volume, data
> size beyond one node, or a specialized access pattern (graph, search, time-series) — breaks the
> relational model.

---

## Part B — Storage Engines: B-Tree vs LSM-Tree

This is the layer most candidates can't go below, and it's where you separate yourself. The
read/write behavior of every DB above is a *consequence* of its storage engine. Two families
dominate.

### B-Tree (update-in-place)

A balanced tree of fixed-size pages (typically 4–16KB) on disk. Each node holds sorted keys and
pointers; leaves hold the data (or a pointer to it). A lookup is `O(log n)` page reads from root to
leaf. **Writes update the page in place** — find the leaf, modify it, write it back.

- **Reads:** excellent and predictable. One key = one root-to-leaf traversal, mostly cached upper
  levels. Range scans follow sibling leaf pointers — sequential and fast.
- **Writes:** a single logical write can touch a page plus a WAL entry, and page splits cascade.
  This is **write amplification**: one row write → multiple physical page writes. Writes are also
  *random* I/O (the target page can be anywhere on disk).
- **Concurrency:** in-place updates need locking/latching on pages; this is where contention and
  the WAL (write-ahead log, for crash recovery) live.

**Real DBs:** Postgres, MySQL/InnoDB, most relational engines, and embedded stores like LMDB.

### LSM-Tree (Log-Structured Merge-tree, append-only)

Writes go to an in-memory sorted structure (the **memtable**) plus an append-only WAL. When the
memtable fills, it's flushed to disk as an immutable, sorted **SSTable**. Reads check memtable,
then SSTables newest-to-oldest. Background **compaction** merges SSTables, discarding overwritten
and deleted keys.

- **Writes:** blazing. Every write is a sequential append (memtable + WAL) — no random I/O, no
  in-place update. This is why write-heavy systems pick LSM.
- **Reads:** can be slow — a key might live in any SSTable, so a read may touch several files. This
  is **read amplification**. Mitigated by **bloom filters** (skip SSTables that definitely don't
  contain the key) and block caches and per-SSTable sparse indexes.
- **Compaction:** the hidden tax. Merging SSTables rewrites data repeatedly — **write
  amplification over time** — and competes with foreground traffic for disk/CPU. Compaction
  strategy (size-tiered vs leveled) trades write amp against read/space amp.
- **Deletes:** don't delete in place; they write a **tombstone** marker that suppresses the key
  until compaction physically removes it. Tombstone buildup (e.g., from range deletes in Cassandra)
  is a classic production footgun.

**Real DBs:** Cassandra, ScyllaDB, HBase, Bigtable, RocksDB (the embedded engine under many of
these and under MyRocks, TiKV, CockroachDB), LevelDB.

### Side-by-side

| Dimension | B-Tree | LSM-Tree |
|---|---|---|
| Write path | In-place, random I/O | Append-only, sequential I/O |
| Write throughput | Lower (random + page splits) | High (sequential appends) |
| Read latency | Low, predictable | Higher, variable (multi-SSTable) |
| Write amplification | Moderate (page + WAL + splits) | High over time (compaction) |
| Read amplification | Low | High (mitigated by bloom filters) |
| Space amplification | Low | Higher (stale data until compaction) |
| Deletes | Immediate | Tombstone + later compaction |
| Range scans | Excellent (leaf pointers) | Good (SSTables are sorted) |
| Wins when | Read-heavy, transactional, ad-hoc | Write-heavy, ingest, time-series |

> **Say this in the room:** "It's write-heavy ingestion — sensor data, an event log, a feed of
> writes — so I want an LSM engine: sequential appends absorb the write volume, and I'll lean on
> bloom filters and a good partition key to keep reads cheap. The cost I accept is compaction load
> and read amplification. If it were a read-heavy transactional system with ad-hoc queries, I'd
> want a B-tree engine like Postgres instead."

There is no universal winner. The amplification triangle — **read amp, write amp, space amp** — is
a pick-your-poison; you can optimize two at the expense of the third, and the engine choice plus
compaction strategy is how you choose.

---

## Part C — The NoSQL Families and Exactly When to Reach for Each

For each: **data model, ideal access pattern, killer feature, failure mode.** That last column is
what makes you sound like you've operated these, not just read about them.

### Key-value — Redis, DynamoDB, Memcached

- **Data model:** opaque value behind a key. Redis adds rich value *types* (strings, hashes, lists,
  sets, sorted sets, streams, HyperLogLog).
- **Ideal access pattern:** point get/put by a known key, lowest possible latency. Caching,
  sessions, rate-limit counters, leaderboards (Redis sorted sets), feature flags.
- **Killer feature:** O(1) in-memory access (Redis) or infinitely scalable managed KV (DynamoDB).
- **Failure mode:** Redis is memory-bound — eviction under pressure, and **a single hot key** can
  saturate one shard no matter how big the cluster. Persistence (RDB/AOF) is a tradeoff; treat
  Redis as a cache unless you've deliberately configured durability.

### Wide-column — Cassandra, ScyllaDB, Bigtable, HBase

- **Data model:** rows keyed by a **partition key**, within which data is sorted by **clustering
  columns**. Think "a giant distributed sorted map of maps." *Not* relational despite SQL-like CQL.
- **Ideal access pattern:** high write volume, queries that hit a single partition and range-scan
  within it by clustering key (time-ordered events per user, messages per channel).
- **Killer feature:** linear write scalability and tunable consistency (per-query quorum); no
  single point of failure (leaderless, Dynamo-style).
- **Failure mode:** you must know your queries up front — there are no ad-hoc joins, and querying by
  a non-key column means a full-cluster scan or a secondary index that's a trap at scale.
  **Hot partitions** and **tombstone-laden range deletes** are the operational killers.

### Document — MongoDB, Couchbase, DocumentDB

- **Data model:** JSON/BSON documents in collections; nested structure, per-document flexible schema.
- **Ideal access pattern:** entities read and written as a whole, where the natural shape is a
  nested object (a product with variants, a user profile with embedded settings). Few cross-document
  joins.
- **Killer feature:** schema flexibility + locality — the whole aggregate is one read, no joins.
  Decent secondary indexing and aggregation pipeline.
- **Failure mode:** people use it as a relational DB and then need joins (`$lookup` is slow and
  awkward). Unbounded array growth in a document, and the temptation to embed when you should
  reference. Multi-document transactions exist but carry a cost.

### Graph — Neo4j, Amazon Neptune, JanusGraph

- **Data model:** nodes + edges, both with properties. Relationships are first-class.
- **Ideal access pattern:** deep traversals — "friends of friends who like X," fraud rings,
  recommendations, dependency graphs. Queries where the *number of joins* in a relational version
  would explode.
- **Killer feature:** **index-free adjacency** — each node directly references its neighbors, so a
  hop is O(1) regardless of total graph size. A 5-hop traversal stays cheap where SQL would do 5
  self-joins.
- **Failure mode:** hard to shard (graphs don't partition cleanly — edges cross any boundary you
  draw), so they scale up before they scale out. Wrong tool for simple aggregate queries.

### Time-series — InfluxDB, TimescaleDB, Prometheus

- **Data model:** (timestamp, metric, tags, value). Append-heavy, time-partitioned.
- **Ideal access pattern:** ingest high-frequency metrics, then query by time window with
  downsampling/rollups ("p99 latency per minute over the last 24h").
- **Killer feature:** time-partitioning + automatic retention/downsampling + compression tuned for
  monotonic timestamps. Timescale gives you this *inside* Postgres (hypertables), so you keep SQL.
- **Failure mode:** **high tag cardinality** (e.g., user-id as a tag) blows up the index and
  memory. Not for arbitrary updates to old data — it's append-oriented.

### Search — Elasticsearch, OpenSearch, Solr

- **Data model:** documents indexed into an **inverted index** (term → list of docs containing it),
  plus analyzers, tokenizers, relevance scoring (BM25).
- **Ideal access pattern:** full-text search, fuzzy/typo-tolerant matching, faceting, relevance
  ranking, log analytics.
- **Killer feature:** fast text search with scoring and aggregations over large corpora.
- **Failure mode:** **it is not your source of truth.** It's near-real-time (refresh interval), can
  lose/lag data, and must be fed from your primary store (CDC, dual-write via queue). Treat it as a
  derived index you can rebuild, never the system of record.

> **Reframe for the room:** "Search and analytics indexes are *derived* stores. The primary DB
> owns the truth; I feed Elasticsearch via change-data-capture off the DB's WAL or via an event
> queue. If ES is lost, I rebuild it. I never dual-write to both synchronously — that's an
> inconsistency bug waiting to happen."

---

## Part D — Indexing Deep Dive

An index is a separate sorted data structure that trades **write speed and space** for **read
speed**. Everything below is about that trade.

### Primary vs secondary

- **Primary index** — built on the primary key; defines how rows are stored/located.
- **Secondary index** — any additional index to support lookups on non-key columns. Each one is a
  separate structure the engine must keep in sync on every write.

### Clustered vs non-clustered

- **Clustered index** — the table data *is* the index leaves; rows are physically stored in primary
  key order. InnoDB (MySQL) is always clustered on the primary key. Range scans on the PK are
  extremely fast. There's exactly one per table.
- **Non-clustered (secondary) index** — a separate structure whose leaves hold the indexed column +
  a pointer back to the row. In InnoDB that pointer is the *primary key*, so a secondary lookup is
  two hops: secondary index → PK → clustered index. (This is why a fat primary key bloats every
  secondary index in InnoDB.)
- **Postgres** differs: the heap is unordered; *all* indexes (including the PK) are non-clustered
  and point at a physical tuple location (ctid). `CLUSTER` is a one-time reorder, not maintained.

### Composite index column order — the rule that trips people up

A composite index on `(a, b, c)` is sorted by `a`, then `b` within equal `a`, then `c`. This is the
**leftmost-prefix rule**: the index can serve queries that filter on a left prefix —
`(a)`, `(a, b)`, `(a, b, c)` — but **not** `(b)` alone or `(b, c)`.

| Query | Uses index `(a, b, c)`? |
|---|---|
| `WHERE a = ?` | Yes |
| `WHERE a = ? AND b = ?` | Yes |
| `WHERE a = ? AND b = ? AND c = ?` | Yes (fully) |
| `WHERE b = ?` | No (skips the leftmost column) |
| `WHERE a = ? AND c = ?` | Partially — uses `a`, then filters `c` |
| `WHERE a = ? ORDER BY b` | Yes (range/sort served by index) |

**Ordering rules of thumb:**
- Put **equality-filter columns before range-filter columns.** A range (`>`, `<`, `BETWEEN`) stops
  the index from using any column after it for further seeking. So `(status, created_at)` for
  `WHERE status = 'active' AND created_at > ?` — not the reverse.
- Put **higher-selectivity (more distinct values) columns first** when all else is equal, to narrow
  the search fastest.
- Order to match your **`ORDER BY`** so the index also satisfies the sort and avoids a filesort.

### Covering index

If an index contains *every column a query needs* (filter + select), the engine answers entirely
from the index and never touches the table — an **index-only scan**. This is one of the cheapest
big wins. Postgres supports `INCLUDE` columns to add payload to an index without making them part
of the sort key.

> **Say this in the room:** "This query is hot and reads only three columns, so I'll make a covering
> index on `(tenant_id, created_at) INCLUDE (status)` — it serves the filter, the sort, and the
> projection from the index alone, no heap fetch."

### The cost of indexes on writes

Every secondary index is **another structure to update on every insert/update/delete**. Five
indexes = roughly 5× the write amplification on the index side, plus the space. On an LSM engine,
secondary indexes also mean more compaction. So:

- Index for the queries you actually run; drop unused indexes (they're pure write tax).
- On write-heavy tables, be stingy. A common production fix for slow ingest is *removing* indexes.
- Watch for redundant indexes — `(a)` is subsumed by `(a, b)` for prefix queries; drop the
  standalone `(a)`.

---

## Part E — ACID vs BASE, Isolation Levels, and MVCC

### ACID vs BASE

- **ACID** (Atomicity, Consistency, Isolation, Durability) — the relational guarantee. A
  transaction is all-or-nothing, leaves valid state, runs as if alone, and survives crashes once
  committed. The default mental model for money, inventory, anything where wrong = unacceptable.
- **BASE** (Basically Available, Soft state, Eventual consistency) — the distributed-NoSQL posture.
  Favor availability now; converge later. The right model for feeds, counts, recommendations —
  places where slightly stale is fine and downtime is worse than staleness.

This maps straight onto **CAP/PACELC**: choosing C vs A under partition, and (else) Latency vs
Consistency in normal operation. ACID stores typically pick consistency; BASE stores pick
availability/latency.

### Isolation levels and the anomaly each prevents

The "I" in ACID is a dial, not a constant. Weaker isolation = more concurrency/throughput but more
anomalies. Know each anomaly and the level that kills it.

| Isolation level | Dirty read | Non-repeatable read | Phantom | Write skew |
|---|---|---|---|---|
| **Read Uncommitted** | Possible | Possible | Possible | Possible |
| **Read Committed** | Prevented | Possible | Possible | Possible |
| **Repeatable Read** | Prevented | Prevented | Possible* | Possible |
| **Serializable** | Prevented | Prevented | Prevented | Prevented |

\* The SQL standard allows phantoms at Repeatable Read; Postgres's snapshot-based RR (and MySQL
InnoDB's gap locks) actually prevent most phantoms — a place where real engines beat the spec.

The anomalies, concretely:

- **Dirty read** — you read a row another transaction wrote but hasn't committed; it then rolls
  back, and you acted on data that never existed.
- **Non-repeatable read** — you read a row twice in one transaction and get different values because
  another transaction committed an update in between.
- **Phantom read** — you run the same range query twice and the *set of rows* changes (new rows
  match) because another transaction inserted.
- **Write skew** — two transactions each read an overlapping set, each makes a decision valid given
  what it saw, and both commit — together violating an invariant (the classic: two doctors each
  check "is someone else on call?", see yes, and both go off call). Only **Serializable** prevents
  it. This is the one that bites people who think snapshot isolation is enough.

> **Say this in the room:** "Snapshot isolation (Postgres Repeatable Read) stops dirty and
> non-repeatable reads and most phantoms, but it does *not* stop write skew. For the on-call /
> double-booking / overdraft-from-two-accounts invariant, I need Serializable, or I enforce it with
> an explicit `SELECT … FOR UPDATE` lock or a uniqueness/exclusion constraint."

### MVCC — how reads don't block writes

**Multi-Version Concurrency Control** is how Postgres, MySQL/InnoDB, and Oracle give you snapshot
reads without read locks. Each write creates a **new version** of the row tagged with transaction
IDs; a reading transaction sees the version that was committed as of its snapshot. So **readers
never block writers and writers never block readers** — a huge concurrency win.

The cost: old versions accumulate as garbage. Postgres needs **VACUUM** to reclaim them; neglecting
it causes table bloat and, in the extreme, transaction-ID wraparound. InnoDB keeps old versions in
the undo log/rollback segment. Knowing MVCC exists *and* that it has a GC cost is a strong signal.

---

## Part F — Distributed Transactions (Teaser)

Single-node ACID is easy: one writer, one WAL, one lock manager. The moment a transaction must span
**multiple shards or services**, it gets hard, for fundamental reasons:

- **No global clock / no shared lock manager.** Coordinating commit across nodes requires a protocol
  and consensus, which costs latency.
- **2PC (two-phase commit)** — a coordinator asks all participants to *prepare*, then *commit*. It's
  blocking: if the coordinator dies after prepare, participants hold locks indefinitely
  (the blocking problem). It trades availability for atomicity and doesn't survive coordinator
  failure gracefully.
- **Sagas** — break the transaction into local transactions per service with **compensating
  actions** to undo on failure. Eventually consistent, non-blocking, no global locks — but you give
  up isolation (intermediate states are visible) and must design every compensation.

> **Pointer:** This is its own deep dive — see the later **Sagas / 2PC / distributed transactions**
> doc. In an interview, the move is almost always: "I'll avoid a distributed transaction. I'll
> design so the transaction stays within one partition/service, or use a saga with compensations
> and idempotency keys, accepting eventual consistency." Reaching for 2PC across microservices is
> usually a design smell.

---

## Part G — Worked Example: DynamoDB Single-Table Design

This is the example that impresses interviewers, because almost nobody can model it live. The whole
philosophy is the inverse of relational: **you design the table around your access patterns, then
overload one table to serve all of them.**

### The model

DynamoDB items live in **partitions** chosen by a hash of the **partition key (PK)**. Within a
partition, items are sorted by the **sort key (SK)**, enabling range queries. A `Query` operation
hits **one partition** and range-scans by SK — that's the only cheap access. Anything else is a
`Scan` (full table, avoid) or a secondary index.

Two index types:
- **LSI (Local Secondary Index)** — alternate SK, *same* PK. Defined at table creation, shares the
  partition.
- **GSI (Global Secondary Index)** — *different* PK and SK; effectively a separate, eventually
  consistent replica of the table reorganized for another access pattern.

### Single-table technique

Instead of one table per entity (relational instinct), put *all* entities in one table with generic
attribute names `PK` and `SK`, and encode the entity type + relationship into the key values. This
lets a single `Query` fetch a parent and its children together (the **item collection**).

### Worked scenario — an e-commerce app

Access patterns to support:
1. Get a user by ID.
2. Get all orders for a user.
3. Get an order and all its line items in one query.
4. Get a product by ID.
5. Get all orders containing a given product (reverse lookup).

Design the keys so each pattern is a single-partition `Query`:

| Entity | PK | SK | Other attrs |
|---|---|---|---|
| User | `USER#u123` | `PROFILE` | name, email |
| Order (under user) | `USER#u123` | `ORDER#o555` | total, status, date |
| Order item (under order) | `ORDER#o555` | `ITEM#p987` | qty, price |
| Order metadata | `ORDER#o555` | `META` | total, status |
| Product | `PRODUCT#p987` | `META` | title, price |

How each pattern resolves:
- **(1) User by ID:** `Query PK = USER#u123, SK = PROFILE`.
- **(2) Orders for a user:** `Query PK = USER#u123, SK begins_with ORDER#` — one partition, returns
  all orders. The user's profile and orders live in the same item collection.
- **(3) Order + line items:** `Query PK = ORDER#o555` — returns the `META` item and all `ITEM#…`
  items in one shot. This is the join you can't do, done by colocation.
- **(4) Product by ID:** `Query PK = PRODUCT#p987, SK = META`.
- **(5) Orders containing a product (reverse):** add a **GSI** with `GSI1PK = PRODUCT#p987` on the
  order-item items, then `Query GSI1PK = PRODUCT#p987` to fan out to all orders with that product.

> **Say this in the room:** "I list every access pattern first, then design PK/SK so each one is a
> single-partition Query. I overload the PK/SK attributes to store multiple entity types in one
> table, and I colocate parent+children in an item collection so I get join-like reads without
> joins. For an access pattern the base keys can't serve, I add a GSI — that's a second physical
> layout of the same data, and I accept its eventual consistency and extra write cost."

### Partition key choice — the hot-partition trap

DynamoDB spreads load by PK hash. A low-cardinality or skewed PK (e.g., `status = ACTIVE` for most
items, or "today's date" for a time-series) concentrates traffic on one partition and throttles —
the **hot partition**. Fixes: choose a high-cardinality PK; **write-shard** a hot key by suffixing
`#0..#N` and scattering writes, reading by fanning out across the suffixes; or add randomness to
time-bucketed keys.

---

## Part H — Connections, Pools, and Replicas

### The "too many connections" failure

Relational DBs back each connection with a **server-side process or thread** plus memory
(Postgres famously forks a process per connection). They top out around a few hundred to low
thousands of connections. In a microservices/serverless world, hundreds of app instances each
opening their own pool can blow past that limit — new connections get rejected, the DB thrashes on
context-switching, and the whole system stalls. **Lambda is the classic offender:** each concurrent
invocation can open a connection, with no natural ceiling.

### Connection pooling

Don't open a connection per request — keep a **pool** of reusable connections.

- **App-side pool** (HikariCP, pgbouncer-as-library) — bounded, reused, fast checkout.
- **External pooler** (**PgBouncer**, RDS Proxy) — sits between app and DB, multiplexing thousands
  of client connections onto a small set of real backend connections (transaction-pooling mode).
  This is the standard fix for the serverless connection storm.

> **Say this in the room:** "Each app instance gets a bounded pool, and I front Postgres with
> PgBouncer in transaction-pooling mode so a thousand clients share, say, 50 real backend
> connections. Without this, scaling the app horizontally takes the database down — more app
> replicas means more connections, and the DB's connection limit, not its CPU, becomes the
> bottleneck."

### Read replicas vs primary

- **Primary** takes all writes (and strongly-consistent reads). **Read replicas** asynchronously
  copy the WAL and serve **reads**, scaling read throughput horizontally — exactly the move for the
  100:1 read-heavy consumer app.
- **The catch is replication lag.** A replica is milliseconds-to-seconds behind. So **don't
  read-your-own-writes from a replica** — a user who just posted and gets routed to a lagging
  replica won't see their post.
- **Mitigations:** route reads that must be fresh to the primary (read-after-write for the same
  user); pin a session to the primary for a short window after a write; or use a replica only for
  analytics/dashboards where staleness is fine.
- Replicas also give you **failover** (promote a replica if the primary dies) — but async
  replication means a small window of unacknowledged writes can be lost on failover; synchronous
  replication avoids that at a latency cost.

---

## Part I — When NOT to Use a Database

A database is the wrong tool for several common needs, and saying so signals maturity.

| Need | Don't use a DB — use… | Why |
|---|---|---|
| Large binary blobs (images, video, files) | **Object store (S3/GCS) + CDN** | DBs are expensive per byte and slow to stream BLOBs; store the bytes in S3, keep only the URL/metadata in the DB |
| Repeated reads of hot data | **Cache (Redis/Memcached)** | Don't hammer the DB for the same row; cache-aside in front |
| Full-text / relevance search | **Search index (Elasticsearch)** | `LIKE '%x%'` can't use an index and doesn't rank; use an inverted index fed from the DB |
| Cross-service event distribution | **Message queue / log (Kafka)** | A DB table as a queue (polling) is an anti-pattern at scale |
| High-frequency metrics/telemetry | **Time-series DB** | A general RDBMS chokes on the ingest + retention pattern |
| Ephemeral session/coordination state | **Redis / etcd / ZooKeeper** | Don't durably persist throwaway state |

> **The blob rule, said cleanly:** "I never put image/video bytes in the database. Bytes go to S3,
> the DB stores the object key plus metadata, and the CDN serves the bytes. The DB stays small,
> fast, and cheap."

---

## Part J — Decision Table: Given This Requirement, Reach for This Store

| Requirement | Store | One-line justification |
|---|---|---|
| Strong consistency, multi-row transactions, ad-hoc queries | **Postgres / MySQL** | ACID + join engine + query planner |
| Sub-ms cache, sessions, counters, leaderboards | **Redis** | In-memory KV, rich types, O(1) |
| Massive write volume, single-partition reads, no joins | **Cassandra / ScyllaDB** | LSM writes, partition+clustering, leaderless |
| Predictable KV at any scale, managed, serverless | **DynamoDB** | Partitioned KV, single-table design |
| Nested aggregates read/written whole, flexible schema | **MongoDB** | Document locality, schema flexibility |
| Deep relationship traversal, recommendations, fraud | **Neo4j / Neptune** | Index-free adjacency |
| Metrics, monitoring, time-window queries + retention | **TimescaleDB / Influx** | Time-partitioning, downsampling |
| Full-text search, faceting, relevance ranking | **Elasticsearch** | Inverted index, BM25 scoring (derived) |
| Images, video, backups, large files | **S3 + CDN** | Cheap, durable bytes; not a DB |
| Relational at scale needing horizontal sharding + SQL | **CockroachDB / Spanner / Vitess** | Distributed SQL, but watch cross-shard cost |
| Read-heavy relational, single-region, big single box | **Postgres + read replicas** | Scale reads via replicas, default until forced off |

> **The default posture:** "Start with Postgres. Move off it only when a specific axis breaks —
> write volume beyond one node, data beyond one node, or a specialized access pattern (graph,
> search, time-series). Name *which* axis broke; don't reach for NoSQL by reflex."

---

### Self-check before the mock (answer these from memory)
- [ ] Give the four access-pattern questions you ask of every entity.
- [ ] Explain B-tree vs LSM-tree in terms of read/write/space amplification, and name two real DBs of each.
- [ ] What is compaction, and what problem do tombstones and bloom filters solve in an LSM store?
- [ ] For each NoSQL family, state the killer feature and the failure mode.
- [ ] State the leftmost-prefix rule and why equality columns go before range columns in a composite index.
- [ ] What is a covering index and why is it cheap?
- [ ] Name the four anomalies and the isolation level that first prevents each — especially write skew.
- [ ] What does MVCC buy you, and what's its hidden cost?
- [ ] Why are distributed transactions hard, and what do you reach for instead of 2PC?
- [ ] Model a parent→children DynamoDB single-table query with PK/SK, and explain when you'd add a GSI.
- [ ] Why does horizontal app scaling take down a Postgres DB, and how does PgBouncer fix it?
- [ ] Name three things you should NOT store in a database, and where they go instead.
