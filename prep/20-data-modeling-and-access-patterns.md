# Topic 20: Data Modeling & Access-Pattern-Driven Schema Design

> **Why this topic matters in the room:** Most candidates draw an ER diagram out of habit — "users
> have many posts, posts have many comments" — and never ask *how the data is read*. At scale that
> is backwards. The schema is a *function of the queries*, not of the domain's Platonic shape. Staff
> signal here is: you enumerate access patterns first, then derive a schema that serves them cheaply,
> and you can articulate exactly what you traded away. This doc is the "step 4" of the framework
> (doc 01) done at depth, and it leans hard on sharding (doc 04) because partition-key choice *is*
> data modeling.

---

## Part A — The Central Principle: Model for Access Patterns, Not for Purity

In a single-node relational world, you normalize first and let the query planner figure out the
joins. That works because everything is on one box and joins are cheap. The moment your data spans
shards, replicas, or services, **a join is a distributed operation** — a scatter-gather across nodes,
or worse, an application-side N+1 loop. The cost model inverts. Now the cheapest read is the one
where all the data you need lives in one place, keyed by exactly how you ask for it.

So the discipline is:

1. **Enumerate access patterns first.** Write them as sentences: *"read X by Y", "write Z", "list
   the N most recent W for user U".* Be exhaustive — every screen, every API endpoint, every batch
   job is an access pattern.
2. **Tag each with its shape:** point lookup vs range scan, read vs write, frequency (QPS), latency
   target, consistency need, cardinality.
3. **Derive keys and tables from that list.** The primary key / partition key falls out of the
   highest-frequency, lowest-latency read. Indexes fall out of the secondary reads.
4. **Only then pick the store.** SQL vs NoSQL is *decided by the access patterns*, not by reflex.

> **Say this in the room:** "Before I draw any tables, let me list the access patterns — that's what
> decides the schema and the store. I'll write them as 'read this by that', and I'll tag each with
> frequency and consistency." This single sentence is worth more than a perfect ER diagram.

A worked enumeration for a tiny "task app" (we'll model it both ways later):

| # | Access pattern | Shape | Freq | Consistency |
|---|----------------|-------|------|-------------|
| AP1 | Get a user's profile by userId | point read | high | strong-ish |
| AP2 | List a user's projects, newest first | range read | high | eventual ok |
| AP3 | List tasks in a project, by status | range read | very high | eventual ok |
| AP4 | Get a single task by taskId | point read | medium | strong |
| AP5 | List all tasks assigned to a user, across projects | range read | medium | eventual ok |

Notice AP5: "across projects" cuts *against* the natural hierarchy (user → project → task). That one
pattern is what will force a denormalized copy or a secondary index. **The awkward access pattern is
where the design lives** — find it early.

---

## Part B — Normalization vs Denormalization

### The normal forms, in one breath

You don't recite definitions in an interview, you recite *intuition*:

- **1NF** — atomic columns, no repeating groups. (No comma-separated `tags` column; no `phone1,
  phone2, phone3`.)
- **2NF** — every non-key column depends on the *whole* key, not part of it. (In an `order_items`
  table keyed by `(order_id, product_id)`, the `product_name` doesn't belong — it depends only on
  `product_id`. Pull it out.)
- **3NF** — non-key columns depend on the key and *nothing but* the key; no transitive deps. (Don't
  store `zip` and `city` in the same row if `zip` determines `city`.)

The intuition that matters: **normalization stores each fact exactly once.** That makes *writes*
cheap and correct — one update, one place, no anomalies. The cost is that *reads* must reassemble
the fact from many tables via joins.

### The read-vs-write tradeoff

| | Normalized | Denormalized |
|--|-----------|--------------|
| Writes | cheap, one place, no anomalies | expensive — fan-out to every copy |
| Reads | expensive — joins / multiple round trips | cheap — one read, data co-located |
| Storage | minimal | duplicated |
| Correctness | easy (single source of truth) | hard (copies drift; **update anomalies**) |
| Best when | write-heavy, ad-hoc queries, strong consistency | read-heavy, known queries, scale |

Denormalization is **trading write complexity and storage for read speed and avoided joins.** You do
it when:

- The read is hot and latency-critical, and the join is expensive or cross-shard.
- The read pattern is *known and stable* (you're not doing ad-hoc analytics on this data).
- The denormalized field changes rarely relative to how often it's read (e.g. `author_name` on a
  post — read millions of times, changed approximately never).

You **don't** denormalize a field that mutates constantly and is read rarely — that's all
write-amplification cost for no read benefit.

### The cost: anomalies and fan-out writes

When you store `author_name` on every one of a user's 50,000 posts, a name change is now a
50,000-row update. That's the **fan-out write**. And if any of those updates fails or is skipped, you
have an **update anomaly** — copies that disagree with the source of truth. This is the entire reason
denormalization is dangerous: you've created multiple sources of truth and now must keep them in
sync.

### Maintaining denormalized data: do it async

You almost never want the fan-out write to be *synchronous* on the user's request path. Instead:

- **Event-driven / CDC.** The write to the source-of-truth table emits an event (or is captured by
  Change Data Capture off the WAL, à la Debezium). A consumer fans out the update to the
  denormalized copies asynchronously. You accept **eventual consistency** between source and copies
  in exchange for a fast write path and decoupling.
- **The contract:** the source of truth is authoritative; copies are a *cache you happen to store in
  the database*. If a copy drifts, a reconciliation job (or the next CDC event) heals it.

> **Say this in the room:** "I'll denormalize `author_name` onto the post for read speed, but the
> `users` table stays the source of truth. On a name change I emit an event and fan out the update to
> posts asynchronously via CDC. I accept that for a few seconds posts may show the old name — that's
> fine for this domain." Naming the source of truth and the staleness window is the senior move.

---

## Part C — Relational Modeling and the Join Cost at Scale

The relational toolkit:

- **Entities** become tables; each row has a surrogate **primary key** (usually a monotonic int or a
  UUID — prefer UUIDv7/ULID if you want time-sortable keys without hot tail inserts on one shard).
- **One-to-many** (user → posts): the "many" side carries a **foreign key** (`posts.user_id`).
- **Many-to-many** (students ↔ courses): a **junction / join table** (`enrollments(student_id,
  course_id, enrolled_at)`) with a composite PK. The junction table is also the natural home for
  attributes *of the relationship itself* (the enrollment date, the role, the permission level).
- **Foreign keys** enforce referential integrity — but FKs across shards don't exist, and even
  in-shard they add write-time check cost.

### Why joins get expensive — and why you sometimes avoid them

A join on one node is a hash/merge over in-memory or local-disk data: cheap, planner-optimized. The
problems appear when:

- **The tables are sharded on different keys.** Joining `orders` (sharded by `order_id`) to `users`
  (sharded by `user_id`) is a cross-shard scatter-gather. There's no local co-location to exploit.
- **The result set is large** and the join is in a hot read path with a tight p99.
- **You've gone to NoSQL**, where the engine offers *no* join at all — you join in application code,
  which is the dreaded N+1.

The escapes, in rough order of preference:

1. **Co-locate via the same shard key.** Shard `orders` *and* `order_items` by `user_id` (or
   `customer_id`) so a customer's whole order graph lives on one shard — the join is local again.
   This is the relational analog of single-table design.
2. **Denormalize the joined field** onto the row that's read (carry `customer_name` on the order).
3. **Precompute a materialized view** of the join result and read that.

> **Say this in the room:** "Joins aren't evil; *cross-shard* joins are. My first lever is to pick a
> shard key that co-locates the things I join together. Only if I can't co-locate do I denormalize or
> materialize."

---

## Part D — NoSQL / Wide-Column / Document Modeling: Query-First Design

In a relational store you model the data and the queries follow. In a NoSQL store you **model the
queries and the data follows.** There is no query planner to rescue a bad schema — the schema *is*
the access plan. This is liberating and unforgiving in equal measure.

### Wide-column (Cassandra/Scylla) and the partition+clustering key

The unit of physical storage is the **partition**, addressed by the **partition key**. Within a
partition, rows are sorted by the **clustering key**. So the mental model is "a giant sorted hash
map: partition key picks the box, clustering key sorts inside the box."

- A query *must* supply the partition key (or you scatter across the whole cluster — forbidden in
  practice). It *may* range-scan on the clustering key.
- Therefore: **one table per access pattern is normal and expected.** You duplicate data across
  tables, each keyed for one query. Cassandra people say "denormalize until it works."

### Single-table design (DynamoDB)

DynamoDB takes this further: model *all* your entities into **one table**, using a generic partition
key (`PK`) and sort key (`SK`) whose meaning is **overloaded** per item type. You pack multiple
entity types and multiple relationships into the same table so that a single `Query` on one partition
returns a pre-joined, heterogeneous result set. Composite/overloaded **GSIs** (Global Secondary
Indexes) cover the access patterns the base table's key can't.

The rationale: DynamoDB charges and scales per-partition, has no joins, and a single-partition
`Query` is the only O(1)-ish, cheap, transactional-locality operation. Single-table design makes the
common access patterns *single requests*.

#### Worked example — the task app's 5 access patterns into ONE table

Recall AP1–AP5 from Part A. We use a generic `PK`/`SK` and prefix values with the entity type so they
don't collide.

Base table, key = `PK` (partition), `SK` (sort):

| PK | SK | type | attributes |
|----|----|------|-----------|
| `USER#u1` | `PROFILE` | User | name, email |
| `USER#u1` | `PROJECT#p1` | Project | title, createdAt |
| `USER#u1` | `PROJECT#p2` | Project | title, createdAt |
| `PROJECT#p1` | `TASK#t1` | Task | title, status=OPEN, assignee=u2 |
| `PROJECT#p1` | `TASK#t2` | Task | title, status=DONE, assignee=u1 |

How each access pattern is served:

- **AP1 — user profile:** `Query PK=USER#u1, SK=PROFILE`. One item, point read. ✅
- **AP2 — user's projects newest-first:** `Query PK=USER#u1, SK begins_with PROJECT#`. Because items
  in a partition are sorted by `SK`, encode the project id as a sortable value (or add `createdAt`
  into the SK: `PROJECT#<ts>#<id>`) and read in reverse. ✅
- **AP3 — tasks in a project by status:** `Query PK=PROJECT#p1, SK begins_with TASK#`. To filter by
  status cheaply, fold status into the SK or use a GSI (below). ✅
- **AP4 — task by id:** if `taskId` is globally unique, either make `PK=TASK#t1` its own partition, or
  add a GSI keyed on `taskId`. ✅
- **AP5 — tasks assigned to a user across projects:** the base table can't do this — assignee isn't
  in the key. Add a **GSI**.

GSI1, key = `GSI1PK = ASSIGNEE#<userId>`, `GSI1SK = STATUS#<status>#TASK#<taskId>`:

| GSI1PK | GSI1SK | (projects task attrs) |
|--------|--------|----------------------|
| `ASSIGNEE#u2` | `STATUS#OPEN#TASK#t1` | title, project=p1 |
| `ASSIGNEE#u1` | `STATUS#DONE#TASK#t2` | title, project=p1 |

- **AP5** is now `Query GSI1PK=ASSIGNEE#u2` — all of u2's tasks across all projects, sortable by
  status. ✅ This is an **overloaded GSI**: the same index serves "tasks by assignee" and "by
  assignee + status" via the composite sort key.

> **Say this in the room:** "DynamoDB single-table: I overload PK/SK so a project and its tasks share
> a partition and one Query returns them together. The cross-cutting pattern — tasks by assignee —
> doesn't fit the base key, so it gets a GSI keyed on assignee. I'm trading a readable schema for
> single-request reads on every access pattern."

### Embedding vs referencing in document stores (MongoDB)

The document analog of the normalize/denormalize choice:

- **Embed** the child inside the parent document (comments inside a post) when: the child is read
  *with* the parent, the child has bounded cardinality, and the child doesn't need independent
  queries. One read gets everything.
- **Reference** (store the child's id, fetch separately / `$lookup`) when: the child is large or
  unbounded (a post with 2M comments blows the 16MB doc limit), is shared across parents, or is
  queried on its own.
- The killer is **unbounded growth inside a document** — embedding an ever-growing array means every
  write rewrites the whole doc and you eventually hit the size cap. Bounded embed, unbounded
  reference.

---

## Part E — Modeling Relationships Without Joins

When the engine won't join for you, you precompute the join. Three patterns:

### 1. Precomputed / materialized views

Run the join (or aggregation) ahead of time and store the result keyed by how it's read. A "user
dashboard" that aggregates across five tables becomes one `dashboard` row per user, refreshed on
write or on a schedule. Reads are O(1); the cost is staleness and the refresh pipeline.

### 2. Fan-out-on-write of denormalized copies

The Twitter-timeline pattern (doc 01's fan-out): on write, push a denormalized copy into every place
it'll be read. Writing a tweet writes a copy into each follower's timeline partition, so the timeline
read is a single partition scan with no join and no fan-out *at read time*. Trade: writes amplify
massively (the celebrity problem), so you hybridize — fan-out-on-write for normal users, fan-out-on-
read for high-fan-out accounts.

### 3. Adjacency list / graph patterns

Model edges explicitly as rows. An **adjacency list** stores, per node, its neighbors:

| PK (node) | SK (edge) | attrs |
|-----------|-----------|-------|
| `USER#u1` | `FOLLOWS#u2` | since |
| `USER#u1` | `FOLLOWS#u3` | since |
| `POST#p9` | `LIKEDBY#u1` | ts |

`Query PK=USER#u1, SK begins_with FOLLOWS#` gives u1's out-edges as a single partition read — no
join. For the *reverse* edge ("who follows u2?"), you store the inverse edge too, or a GSI:
`GSI: PK=FOLLOWEDBY#u2`. **Bidirectional adjacency = store both directions.** For deep multi-hop
traversal (friends-of-friends-of-friends), a purpose-built graph DB (Neo4j) beats hand-rolled
adjacency lists, because each hop in adjacency-list land is another round trip.

---

## Part F — One-to-Many and Many-to-Many at Scale, and the Hot Partition

Relationships are where scale bites, because **cardinality is unbounded and skewed.**

- **One-to-many — a user's posts.** Fine until one user has 50M posts in one partition. Now that
  partition is a hot spot for writes and too big to scan. Mitigation: **partition the children**
  (`USER#u1#BUCKET#<n>`), or time-bucket them (next section).
- **Many-to-many — followers / memberships.** A celebrity with 100M followers is a single logical
  edge set you cannot store in one partition. The "followers of X" partition is both **huge** and
  **write-hot** (everyone follows/unfollows). Mitigations: shard the follower set across sub-
  partitions; treat super-nodes specially (the hybrid fan-out again); cap or sample where product
  allows.

**The hot-partition risk** is the recurring villain. Any partition key whose value distribution is
skewed — a celebrity, a "system" user, a single popular product, a `status=PENDING` that 99% of rows
share — concentrates load on one physical shard and defeats horizontal scaling. Detect it by asking
of every partition key: *"what's the most popular value, and can one value's traffic exceed one
node's capacity?"* If yes, add a salt / sub-partition (`key#<bucket>`), or choose a higher-cardinality
key. (This ties directly to doc 04 — shard-key selection.)

> **Say this in the room:** "My partition key is `userId`, which is fine for the median user. But a
> celebrity is a hot partition — one value exceeds one node. So for accounts over a threshold I salt
> the key into N sub-partitions and scatter-gather on read, or I special-case them entirely."

---

## Part G — Time-Series and Append-Only / Event Modeling

Time-series and event logs are their own modeling discipline because the data is **immutable and
arrives in time order** — which is exactly what creates a hot partition if you're naive.

- **Append-only, immutable.** Never update an event; correct it with a new event. This makes the
  store a source of truth you can replay (event sourcing), and it sidesteps update anomalies
  entirely.
- **The naive trap:** partitioning by `timestamp` (or a monotonic id) sends *all* current writes to
  the *same* newest partition — a write hot spot, and old partitions go cold. Classic time-series
  anti-pattern.
- **Time-bucketed composite partition key.** Partition by `(entityId, timeBucket)` — e.g.
  `PK = sensor123#2026-06-20`, clustering key = full timestamp. This (a) spreads writes across many
  sensors *and* days, (b) makes "last hour of sensor 123" a single-partition range scan, and (c)
  makes retention trivial — **drop whole old partitions/buckets** instead of deleting rows. Choose the
  bucket granularity (hour/day/month) so a bucket holds a query-friendly amount of data and not too
  much.

Specialized stores (Cassandra, InfluxDB, TimescaleDB, ClickHouse) bake this in, but the modeling
instinct — *bucket time, co-locate by entity, immutable rows, drop don't delete* — is what they're
asking for.

---

## Part H — Choosing Partition + Sort Keys From Access Patterns

This is the synthesis of everything above, and it's the same skill as shard-key selection (doc 04).
The algorithm:

1. **Partition key** = the value present in your *highest-frequency, lowest-latency* read, and which
   has **high cardinality and even distribution** (no hot value). It's almost always the entity you
   "read by" — `userId`, `accountId`, `(entityId, timeBucket)`.
2. **Sort/clustering key** = whatever you **range-scan, filter, or order by** within that partition —
   timestamp, status, sub-entity type. Compose it (`STATUS#TS#ID`) to serve multiple ordered queries
   from one key.
3. **Secondary indexes** = the access patterns whose "read by" value is *not* the partition key
   (Part D's AP5).
4. **Check for hot partitions** against every key (Part F).

The tension to verbalize: a key that's great for one access pattern (point read by userId) can be
useless for another (range by time). You resolve it with composite sort keys, GSIs, or a second
table — and you state which access pattern you optimized the base key for and why.

---

## Part I — Schema Evolution and Migrations Without Downtime

A live system's schema changes constantly. The staff skill is changing it **without a long lock and
without breaking in-flight code.**

### The danger: long locks

A naive `ALTER TABLE ... ADD COLUMN ... DEFAULT ...` or `ADD INDEX` on a huge table can take an
exclusive lock for minutes — an outage. Modern engines do some of these online; for the rest you use
tooling (`pt-online-schema-change`, `gh-ost`) that builds a shadow table, backfills it, and atomically
swaps. Know that **adding a nullable column with no default is cheap; rewriting every row is not.**

### Expand–Contract (a.k.a. parallel-change)

The universal pattern for any change that old and new code can't both tolerate (rename a column,
change a type, split a table). Never change in place — **expand, migrate, contract:**

1. **Expand.** Add the new structure *alongside* the old (new column / new table). Schema now
   supports both shapes. Deploy code that **writes to both** old and new, reads from old.
2. **Migrate / backfill.** Backfill the new structure from the old in batches (small transactions, no
   long lock). Flip reads to the new structure once it's fully populated and verified.
3. **Contract.** Once nothing reads the old structure and you're confident, stop writing it and drop
   it.

Each step is independently deployable and reversible. **At no point do the database and the running
code disagree** — that's the whole point.

> **Say this in the room:** "I never rename a column in place — old pods would break mid-deploy. I
> expand-contract: add the new column, dual-write, backfill in batches, switch reads, then drop the
> old. Every step is reversible."

### Backfills

Run in **idempotent batches** (`WHERE id BETWEEN x AND y`), rate-limited so you don't saturate the
DB or replication. Make the backfill resumable (track progress) and re-runnable. For huge tables,
backfill off a replica's snapshot or via the CDC stream.

### Versioned schemas and additive changes

- **Additive-only is the safe default.** Adding optional fields never breaks an old reader; removing
  or repurposing a field does. In document stores and event payloads, carry a `schemaVersion` and have
  readers tolerate multiple versions (upcasting old events on read).
- Prefer **forward- and backward-compatible** wire formats (Protobuf/Avro with reserved field
  numbers) so producers and consumers can deploy independently.

---

## Part J — Soft Deletes, Audit Columns, Tombstones

- **Soft delete** — a `deleted_at` (or `is_deleted`) column instead of a physical `DELETE`. Buys
  recoverability, audit, and referential safety; costs query noise (every read must filter
  `deleted_at IS NULL` — easy to forget, so use a view or default scope) and the rows still occupy
  space. Use a **partial index** so the index only covers live rows.
- **Audit columns** — `created_at`, `updated_at`, `created_by`, `updated_by`, and often a `version`
  for optimistic concurrency. Cheap to add, invaluable in incidents. The `version` column also gives
  you **optimistic locking** (`UPDATE ... WHERE version = :v`) to avoid lost updates without a
  distributed lock.
- **Tombstones** — in distributed/LSM stores (Cassandra, Dynamo), a delete is written as a
  *tombstone* marker, not an immediate removal, because the delete must propagate to all replicas
  before space is reclaimed during compaction. Two gotchas to name: (1) **tombstone buildup** —
  deleting many rows in a partition then range-scanning it can read thousands of tombstones and time
  out; (2) deletes only become permanent after the GC grace period, so a resurrected replica won't
  un-delete data.

---

## Part K — Secondary Indexes: Local vs Global, Cost and Consistency

An index is itself a denormalized, query-shaped copy of the data — so all of Part B applies.

- **Local secondary index (LSI)** — lives *within each partition*; lets you query by an alternate
  sort key *for the same partition key*. Same-partition, so it can be **strongly consistent** and
  cheap, but it cannot answer cross-partition queries and (in DynamoDB) bounds partition size.
- **Global secondary index (GSI)** — a *separate*, independently partitioned copy keyed on a
  different attribute; answers "read by something other than the base partition key" (Part D's AP5).
  Because it's a separate structure maintained asynchronously, it's **eventually consistent** with
  the base table and has its own capacity/cost.
- **General cost of any index:** every write must update every index covering the table →
  **write amplification** and more storage. Index only the attributes you actually query, and watch
  that a GSI's own partition key isn't itself a hot spot (indexing `status` where 99% of rows are
  `ACTIVE` recreates the hot-partition problem inside the index).

> **Say this in the room:** "A GSI lets me read by assignee, but it's eventually consistent and it
> doubles my write cost on that table. I'll add it only because AP5 is a real, frequent pattern, and I
> accept a small read-after-write lag on it."

---

## Part L — Worked Example: 5 Access Patterns, Two Schemas

Same task app, same AP1–AP5. Side by side so you can show you'd pick the store *from the patterns*.

### Relational schema (Postgres)

```sql
users      (user_id PK, name, email, created_at, updated_at)
projects   (project_id PK, owner_id FK->users, title, created_at, deleted_at)
tasks      (task_id PK, project_id FK->projects, assignee_id FK->users,
            title, status, created_at, updated_at, deleted_at)

-- indexes derived from the access patterns:
-- AP2: projects by owner, newest first
CREATE INDEX idx_projects_owner ON projects(owner_id, created_at DESC)
  WHERE deleted_at IS NULL;
-- AP3: tasks in a project filtered by status
CREATE INDEX idx_tasks_project_status ON tasks(project_id, status)
  WHERE deleted_at IS NULL;
-- AP5: tasks across projects by assignee
CREATE INDEX idx_tasks_assignee ON tasks(assignee_id, status)
  WHERE deleted_at IS NULL;
```

- AP1 = PK lookup. AP4 = PK lookup. AP2/AP3/AP5 = covered indexes above. Soft delete via
  `deleted_at` + partial indexes. Audit columns present.
- **When to pick this:** moderate scale, you want ad-hoc queries and transactions (move a task
  between projects atomically), the team knows SQL, and you can fit on one primary + replicas. AP5's
  cross-project query is a *non-issue* here — it's just an index.

### NoSQL schema (DynamoDB single-table)

As built in Part D: base table `PK/SK` co-locating user→projects and project→tasks; `GSI1` on
`ASSIGNEE#<id>` / `STATUS#...` for AP5.

- Every access pattern is a single `Query` on a single partition. No joins, scales horizontally per
  partition, predictable latency.
- **When to pick this:** very high scale on these *exact* patterns, you can live with eventual
  consistency on the GSI, and you will *not* need ad-hoc queries (the schema is welded to AP1–AP5 —
  a new access pattern means a new GSI or a table migration). Moving a task between projects is now a
  delete+put across partitions, needing a transaction or accepting a brief inconsistency.

The decision sentence: **"If the access patterns are stable and the scale is extreme, single-table
NoSQL gives me single-request reads and linear scaling. If I need flexibility, transactions across
entities, and ad-hoc queries, I take Postgres and index for AP2/AP3/AP5 — the cross-project query is
trivial there."**

---

## Part M — How to Use This in the Room

1. **Enumerate access patterns before drawing anything.** "Read X by Y, write Z" — with frequency and
   consistency tags. Find the awkward cross-cutting one.
2. **Derive keys from the hottest read; derive indexes from the rest.**
3. **Normalize by default; denormalize deliberately,** naming the source of truth and the async
   mechanism (CDC/events) that keeps copies in sync.
4. **Co-locate before you join; materialize before you scatter-gather.**
5. **Interrogate every partition key for a hot value.** Salt or sub-partition the skew.
6. **Treat schema change as expand-contract with batched backfills** — never an in-place rename under
   live traffic.
7. **Every modeling choice gets a tradeoff sentence:** "I duplicate X for read speed; cost is fan-out
   writes; I keep it correct via CDC; I accept N-second staleness because W."

> **Mental checklist for an unseen data-modeling problem:**
> What are the access patterns? → Which is hottest? → What key serves it (and is that key hot)? →
> Which patterns don't fit the key → index or second table? → Read-heavy → denormalize/materialize?
> → How do copies stay in sync? → How does this schema evolve without a lock?

---

### Self-check before the mock (answer these from memory)
- [ ] Why do you model for access patterns instead of normalized purity in a distributed system?
- [ ] Give the 1NF/2NF/3NF intuition in one line each.
- [ ] State the read-vs-write tradeoff of denormalization and how you keep copies in sync.
- [ ] Why are *cross-shard* joins the real enemy, and what are your three escapes?
- [ ] Walk the task-app 5 access patterns into a DynamoDB single-table design (PK/SK + GSI).
- [ ] When do you embed vs reference in a document store?
- [ ] How do you avoid a hot partition for a celebrity / a monotonic timestamp?
- [ ] Describe expand-contract and why dual-write + backfill avoids breaking live code.
- [ ] LSI vs GSI: consistency and cost difference?
- [ ] Soft delete vs tombstone — and the two tombstone gotchas.
