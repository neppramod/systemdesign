# Topic 11: Search Systems — Inverted Indexes, Ranking & Autocomplete

> **Why this topic earns its keep:** "Search" shows up in half of all product prompts (search a
> product catalog, search messages, search users, typeahead) and almost nobody can explain *how the
> index actually works* under the hood. The candidates who can — "term → posting list, intersect with
> skip pointers, score with BM25, scatter-gather across shards, rerank top-k" — instantly read as
> senior. This doc gives you the data structure, the ranking math (just enough), the
> Elasticsearch/Lucene operational reality, the keep-it-in-sync pipeline, and a from-scratch
> autocomplete design. The recurring theme: **a search index is a denormalized read-optimized copy
> of your data, so everything reduces to "how do I build it fast, query it fast, and keep it in
> sync."**

---

## Part A — The Inverted Index

### Forward vs inverted

A **forward index** maps `document → terms` ("doc 7 contains {red, running, shoes}"). That's how the
data naturally arrives. It's useless for search: to answer "which docs contain *shoes*?" you'd scan
every document. This is exactly what `WHERE body LIKE '%shoes%'` does — a full table scan, no index.

An **inverted index** flips it: `term → list of documents containing that term`. Now "which docs
contain *shoes*?" is one lookup. This is *the* core data structure of every search engine.

```
Documents:
  doc1: "red running shoes"
  doc2: "blue running socks"
  doc3: "red shoes on sale"

Inverted index (term → posting list):
  blue    -> [doc2]
  red     -> [doc1, doc3]
  running -> [doc1, doc2]
  sale    -> [doc3]
  shoes   -> [doc1, doc3]
  socks   -> [doc2]
```

The list of docs for a term is the **posting list**. The set of all terms is the **dictionary** (or
term dictionary / lexicon), usually kept sorted so you can binary-search or use an FST (finite state
transducer, what Lucene uses) for prefix lookups.

### What goes in a posting

A posting is not just a doc ID. For ranking and phrase queries it carries more:

| Field in a posting | Why it's there |
|---|---|
| **Document ID** | which doc the term appears in |
| **Term frequency (TF)** | how many times the term appears in *that* doc → ranking |
| **Positions** | offsets within the doc → phrase queries ("running shoes" adjacent) and proximity |
| **Field info** | which field (title vs body) → field-weighted scoring |

So `shoes -> [(doc1, tf=1, pos=[2]), (doc3, tf=1, pos=[1])]`. Posting lists are **sorted by doc ID**
— that sort is what makes intersection fast.

### Building the index: the analysis pipeline

You don't index raw text; you run it through an **analyzer** to produce terms. Same pipeline runs at
**index time** and **query time** — and they *must match*, or your query terms won't equal your
indexed terms.

| Step | What it does | Example |
|---|---|---|
| **Tokenization** | split text into tokens | `"Red Running-Shoes!"` → `[Red, Running, Shoes]` |
| **Lowercasing / normalization** | case-fold, strip accents (`café`→`cafe`), Unicode NFC | `Red` → `red` |
| **Stop-word removal** | drop high-frequency low-signal words (`the, a, is, of`) | optional, language-specific |
| **Stemming / lemmatization** | reduce to root form | `running, runs, ran` → `run`; `shoes` → `shoe` |
| **Synonyms** (optional) | expand or normalize | `tv` → `television` |
| **N-grams / edge-grams** (optional) | for substring / autocomplete | `shoe` → `s, sh, sho, shoe` |

> **Say this in the room:** "The single most common search bug is an analyzer mismatch — you index
> with one analyzer and query with another, so a query for *Running* never matches the stemmed term
> *run*. Index-time and query-time analysis must be the same (or deliberately paired)."

**Stemming tradeoff:** more recall, less precision. Aggressive stemmers (Porter) over-stem
(`university`/`universe` → `univers`). Lemmatization is dictionary-based and more accurate but
slower. Stop-words save index space and speed up common-word queries, but break phrases like
`"to be or not to be"` and `"The Who"` — modern engines mostly keep them and lean on ranking instead.

### Statistics the index keeps (the ranking inputs)

- **TF — term frequency:** count of term *t* in document *d*. More occurrences ≈ more relevant *to a
  point*.
- **DF — document frequency:** number of documents containing *t*. High DF = common word = low
  signal.
- **IDF — inverse document frequency:** `log(N / DF)`. Rare terms discriminate; common terms don't.
- **Document length:** for length normalization (a 5-word title matching "shoes" beats a 5000-word
  doc that mentions it once).

### How a query executes

Query `red AND shoes`:

1. Analyze the query → terms `{red, shoes}` (same pipeline as indexing).
2. Fetch posting list for each: `red -> [1,3]`, `shoes -> [1,3]`.
3. **Intersect** (AND) the sorted lists with a merge walk → `[1,3]`.
   For **OR**, you **union** them. For **NOT**, you subtract.
4. Score each surviving doc (BM25), keep a top-k heap.
5. Return top-k, then fetch the stored docs for those IDs.

**Merge-intersection** walks both lists with two pointers, advancing the smaller doc ID — O(n+m).
The catch: if `red` has 10M postings and `shoes` has 1k, you don't want to step through all 10M.

**Skip pointers** fix this. Posting lists embed a skip list: every √n entries, a pointer says "doc
ID here jumps to X." When intersecting, if you're looking for doc 5000 in the big list, you skip
forward in big jumps instead of one-by-one. This turns the rare cost from O(n) toward O(n/skip + matches).
Lucene stores postings in compressed blocks (≈128 docs) with block-level skip data, so you can skip
whole blocks.

**Query-evaluation strategies** worth naming:
- **Term-at-a-time (TAAT):** process one posting list fully, accumulate partial scores, then the next.
- **Document-at-a-time (DAAT):** advance all lists together per doc; standard for AND/top-k.
- **WAND / BlockMax-WAND:** the big one. Maintain a max-possible-score bound per term; if a doc
  can't beat the current k-th best score even at max, skip it entirely. This lets you *skip
  scoring* most candidates — it's why a 100M-doc index answers in milliseconds. Mention BlockMax-WAND
  and you sound like you've read the Lucene source.

**Compression:** posting lists are huge, so doc IDs are **delta-encoded** (store gaps, not absolutes)
and packed (variable-byte, Frame-of-Reference, PForDelta). Smaller postings = more fits in page
cache = faster. This is why sorted-by-doc-ID matters: deltas are small and monotonic.

---

## Part B — Ranking

Boolean matching tells you *which* docs match. Ranking tells you the *order*. This is where search
quality lives.

### TF-IDF — the intuition

Score a term in a doc by `TF × IDF`:
- **TF** rewards docs that use the term a lot.
- **IDF** = `log(N/DF)` down-weights terms that appear everywhere.

Sum across query terms. Simple, explainable, and the basis for everything. **The problems:** raw TF
grows linearly (a doc with the word 100× isn't 100× more relevant), and it doesn't normalize for
document length well.

### BM25 — the default, and why

BM25 ("Best Match 25") is TF-IDF grown up. Same IDF, but two fixes:

1. **TF saturation** — term frequency contributes with *diminishing returns*. Going from 1→2
   occurrences matters a lot; 100→101 matters almost nothing. Controlled by `k1` (≈1.2).
2. **Length normalization** — penalizes long documents that match by sheer size, relative to the
   average doc length. Controlled by `b` (≈0.75).

```
score(d,q) = Σ_t  IDF(t) · ( tf(t,d)·(k1+1) ) / ( tf(t,d) + k1·(1 - b + b·|d|/avgdl) )
```

> **Say this in the room:** "I'd use **BM25** — it's the default in Lucene/Elasticsearch since 2016.
> It's TF-IDF with term-frequency saturation and document-length normalization, so a spammy
> keyword-stuffed page or a giant document doesn't dominate. It needs no training data, which is why
> it's the baseline before you reach for ML."

**Field boosting:** match in `title` weighted higher than match in `body`. In ES this is
`title^3`. Combine fields with `dis_max`/`best_fields` (take the best-matching field) or
`cross_fields`.

### Learning-to-rank (LTR)

BM25 ignores signals like clicks, recency, popularity, price, personalization. **Learning-to-rank**
trains a model (gradient-boosted trees — LambdaMART — or a neural model) on features
(BM25 score, recency, CTR, price, user signals) to produce the final order, optimizing a ranking
metric (NDCG). Architecture is **two-phase**: BM25 retrieves a cheap candidate set (top ~1000), then
the expensive LTR model **reranks** just those. You never run the heavy model over the whole corpus.

### Vector / semantic search

BM25 is **lexical** — it matches words. It fails on synonyms and intent: a query for "affordable
laptop" won't match a doc that says "budget notebook." **Embedding search** fixes this: encode query
and docs into dense vectors (via a model) where semantic similarity = vector proximity (cosine /
dot product). Then "search" = **nearest-neighbor in vector space**.

Exact nearest-neighbor over millions of vectors is too slow, so you use **Approximate Nearest
Neighbor (ANN)**:

| ANN method | Idea | Tradeoff |
|---|---|---|
| **HNSW** (Hierarchical Navigable Small World) | layered proximity graph; greedy-walk from a top entry node down | Fast, high recall, **default in most vector DBs**; high memory, costly inserts/rebuild |
| **IVF** (Inverted File) | k-means cluster vectors; search only the nearest *n* clusters | Less memory than HNSW; recall depends on `nprobe`; needs training |
| **PQ** (Product Quantization) | split vector into sub-vectors, quantize each to a codebook | Massive memory compression (32×+); lossy → lower recall; pairs with IVF (**IVF-PQ**) |

**Hybrid search** is the modern production answer: run BM25 *and* vector search, fuse the results
(e.g. **Reciprocal Rank Fusion**, or weighted score blend). Lexical handles exact terms / SKUs /
names; vector handles intent and synonyms. ES 8+, OpenSearch, and dedicated vector DBs (Pinecone,
Weaviate, Milvus, pgvector) all support this.

> **Say this in the room:** "I'd start with BM25 because it needs no training and is explainable.
> If we see synonym/intent misses, I'd add a vector field and do **hybrid retrieval with RRF**,
> then a learning-to-rank reranker on the top candidates if we have click data. I would not jump
> straight to embeddings — they're more infra, harder to debug, and worse at exact-match (SKUs,
> part numbers)."

---

## Part C — Elasticsearch / Lucene Architecture

Elasticsearch is a distributed wrapper around **Lucene** (the single-node index engine). You need
both layers.

### The layered vocabulary

| Layer | What it is |
|---|---|
| **Index** | logical collection of documents (like a DB table). Has a **mapping** (schema) + analyzers. |
| **Shard** | an index is split into shards; **each shard is a complete Lucene index**. Unit of scaling + distribution. |
| **Primary vs replica shard** | each shard has 1 primary + N replicas. Writes go to primary then replicate; replicas serve reads + provide HA. |
| **Segment** | within a shard, a Lucene index is a set of immutable **segments** (mini inverted indexes). |

### Near-real-time (NRT) and the immutable-segment trick

This is the core Lucene insight and a great deep-dive:

1. Incoming docs go into an in-memory **buffer**.
2. A **refresh** (default every **1s**) flushes the buffer into a new **segment** and makes it
   searchable. This is why ES is "near-real-time," not real-time — there's a ~1s lag.
3. **Segments are immutable.** You never modify a segment. New docs → new segments. An **update** =
   index a new version + mark the old doc deleted (a tombstone in a `.del` file); a **delete** = just
   the tombstone. Space is reclaimed later.

**Why immutable?** Immutability is a gift:
- No locks on read — segments never change, so many readers run concurrently, lock-free.
- Cacheable — OS page cache and filesystem cache stay hot; nothing gets invalidated mid-flight.
- Simple crash semantics — a segment is either fully written or not.
- Compression-friendly — write once, optimize layout, never patch.

**The cost — and the fix:** you accumulate many small segments + tombstones. Searching means
querying *every* segment and merging. So a background **merge** process continuously combines small
segments into larger ones, dropping deleted docs in the process. Merging is I/O-heavy (it rewrites
data) and is the usual culprit behind indexing-time I/O spikes.

> **Say this in the room:** "Segments are immutable, which is what makes reads lock-free and
> cacheable. The price is that deletes are just tombstones and you accumulate small segments, so a
> background merge compacts them and physically removes deleted docs. Updates aren't in-place —
> they're delete-plus-reindex."

### The translog (durability)

Refresh makes docs *searchable* but doesn't make them *durable* — a refreshed segment may still be
only in memory/filesystem cache. So every write is also appended to a **translog** (write-ahead log)
and fsync'd. On crash, ES replays the translog. A **flush** periodically commits segments to disk
(fsync) and truncates the translog. So: **refresh = visibility (1s), flush = durability (commit to
disk).** Two different clocks, commonly confused.

### Mapping & analyzers

The **mapping** is the schema: field types (`text` vs `keyword` is the classic gotcha — `text` is
analyzed/tokenized for full-text; `keyword` is stored whole for exact match, sorting, aggregations),
which analyzer per field, whether a field is indexed/stored. Mappings are mostly immutable once set
(you can add fields, not retype them) — which is *why reindexing exists* (Part D).

### How indexing scales

- Document is routed to a shard by `hash(routing_key) % num_primary_shards` (default routing key =
  doc `_id`). This fixes shard count at index-creation time — you can't change primary shard count
  without reindexing.
- Write goes to the primary, then fans out to replicas. Acknowledged when enough copies have it.
- More indexing throughput → more primary shards (parallelism) and/or bulk requests. But too many
  tiny shards is the most common ES anti-pattern (overhead per shard). Rule of thumb: target tens of
  GB per shard.

### How a query scatters and gathers

Querying is **scatter-gather** in two round-trips:

1. **Query phase (scatter):** coordinating node sends the query to **one copy of every shard**
   (primary or replica — load balanced). Each shard runs the query locally against its segments and
   returns just the **top-k doc IDs + scores** (not the documents).
2. **Coordinator merges:** combines all shards' top-k into a global top-k by score.
3. **Fetch phase (gather):** coordinator asks the relevant shards for the **full documents** of only
   the final top-k.

> **The deep-pagination problem lives here.** To return results 10,000–10,010, *every shard* must
> return its top 10,010 to the coordinator, which sorts `num_shards × 10,010` and discards almost
> all of it. Cost grows with the **offset**, on every shard. See Part F.

**Scoring caveat:** IDF is a per-shard statistic by default, so scores can differ slightly across
shards depending on term distribution. For consistency you can use `dfs_query_then_fetch` (a pre-pass
to gather global term stats) at a latency cost — usually not worth it with reasonably balanced shards.

---

## Part D — Keeping the Index in Sync with the Source of Truth

The search index is a **secondary, denormalized copy**. Your DB is the source of truth. The whole
problem is propagating DB changes into ES without losing or duplicating data. **This is the
dual-write problem again** (see the messaging/outbox material).

### The naive (broken) approach: dual write

```
app → write to DB
app → write to Elasticsearch   ← second write
```

If the second write fails (ES down, app crashes between the two), the index silently drifts from the
DB. No transaction spans both systems. **Don't do this.** Name it as a trap.

### The right approaches

| Approach | How | When |
|---|---|---|
| **CDC-driven** (Debezium → Kafka → indexer) | tail the DB's replication log (binlog/WAL); stream changes into a topic; an indexer consumer applies them to ES | Best for keeping ES in sync with an existing OLTP DB; no app code change; captures *every* change |
| **Transactional outbox** | app writes business row + an `outbox` row in the **same DB transaction**; a relay publishes outbox rows to a queue → indexer | When you control the app and want app-level events, not raw row changes |
| **Queue-driven from the app** | app emits a domain event to Kafka/SQS after commit; indexer consumes | Simple; risks the same dual-write gap unless paired with outbox/CDC |

All of these make indexing **asynchronous** (eventual consistency, typically sub-second to seconds)
and require the indexer to be **idempotent** — at-least-once delivery means you'll see duplicates,
so upsert by doc ID and use a version/timestamp to drop stale out-of-order updates (ES external
versioning: reject a write whose version < current).

### Reindexing — you will need it

Mapping changes (new analyzer, retyped field), shard-count changes, schema migrations, or a corrupted
index all force a **full rebuild**. You cannot mutate the old index in place. The pattern:

1. Create `products_v2` with the new mapping/settings.
2. Backfill from the source of truth (DB or a reindex from `products_v1`).
3. **Keep the live CDC/queue pipeline writing to *both* v1 and v2** during the backfill (or replay
   from a known offset) so v2 catches up to live.
4. **Atomically swap an alias.** Clients query the alias `products`, never the concrete index. Repoint
   `products: v1 → v2` in one atomic alias operation. This is **blue/green** for search.
5. Verify (doc counts, sample queries, latency), keep v1 around briefly to roll back, then delete it.

> **Say this in the room:** "Clients should always hit an **alias**, never a concrete index name.
> That alias indirection is what makes zero-downtime reindexing possible — build v2 in the
> background, swap the alias atomically, roll back by swapping it back."

---

## Part E — Autocomplete / Typeahead (the classic question)

The prompt: "user types `sho`, instantly show top suggestions (`shoes`, `shorts`, `shower`)." This is
its own subsystem — *not* the same as running the main search engine per keystroke. Goals: **p99
< ~50–100ms**, ranked by popularity, typo-tolerant, scalable to huge QPS (a request *per keystroke*).

### Core structures

**Trie (prefix tree).** Each node is a character; a path is a prefix; nodes mark word-ends. Lookup of
a prefix is O(length of prefix). Natural fit for "all completions of `sho`" — walk to the `sho` node,
then collect descendants.

```
         (root)
          / \
        s    ...
        |
        h
        |
        o
       / \
      e   r
      |   |
      s   t
   "shoes" "short"
```

**The catch:** collecting *all* descendants of a popular prefix (`a`, `s`) can be thousands of words
— too slow per keystroke. So you don't collect at query time.

### Top-k per prefix precomputation (the key move)

Store, **at each trie node, the precomputed top-k completions** (by popularity/weight) for that
prefix. Now answering `sho` is: walk to the `sho` node, return its cached top-k list. **O(prefix
length), no traversal.** This is the single most important optimization and the thing interviewers
want to hear.

- Each node holds its top-k (say k=10) child completions with scores.
- Built offline / in batch from query logs; refreshed periodically.
- Memory cost is the tradeoff — k entries per node — but it's bounded and worth it.

**Alternative: prefix hashing.** Instead of a trie, a hash map `prefix → top-k list` for every prefix
up to some length (`s`, `sh`, `sho`, `shoe`...). Dead simple, O(1) lookup, trivially shardable by
prefix, cache-friendly (it's basically a key-value store — fronted by Redis). Tradeoff: more storage
(every prefix is a key), and prefixes longer than the cap fall back to the engine. Many real
typeaheads are exactly this: **Redis with `prefix → JSON top-k`**, rebuilt from logs.

### Weighted / ranked suggestions

Suggestions are ranked by a **weight**: query popularity (count from logs), recency, personalization,
business value (promote in-stock high-margin products), or a blend. The precomputed top-k is sorted
by this weight. Personalization usually means a small per-user re-rank layered on top of a global
list (don't precompute per-user × per-prefix — combinatorial explosion).

### Handling typos (fuzzy)

User types `shooes`. Options:
- **Edit distance (Levenshtein):** suggest words within edit distance 1–2. Brute force is too slow;
  use a **Levenshtein automaton** (what Lucene's fuzzy query uses) or a **BK-tree** (metric tree for
  edit distance) to find near matches efficiently.
- **N-gram / edge-gram index:** index substrings so misspellings still share grams.
- **Phonetic** (Soundex/Metaphone) for sound-alike names.
- Pragmatic combo: try exact prefix first; if too few results, fall back to fuzzy. Cap edit distance
  by term length (allow more edits on longer words).

### Architecture & scaling reads

Typeahead is **extremely read-heavy** and latency-critical, so it's almost pure caching:

```
client (debounced keystrokes)
   │
   ▼
CDN / edge cache  ──(hit)──► top-k for prefix
   │ miss
   ▼
Typeahead service  ◄── in-memory trie / Redis prefix→top-k (precomputed)
   ▲
   │ (batch rebuild)
Query-log pipeline (Kafka → aggregation → top-k builder)
```

- **Debounce client-side** (~100–300ms) and cancel in-flight requests so you don't fire on every
  keystroke.
- **Cache aggressively** — prefixes are heavily skewed (Zipf); cache the head at the CDN/edge.
- **Serve from memory** — the trie/Redis sits in RAM; never hit disk on the hot path.
- **Shard by prefix** when one box can't hold it (`a–f`, `g–m`...). Reads route by first letters.

### Updating popularity (the write path)

Popularity changes constantly; you don't want to rebuild the whole structure live:
- **Batch:** stream query logs → Kafka → aggregate counts (hourly/daily) → recompute top-k per
  prefix → atomically swap the new structure in (same alias/blue-green idea). Simple, slightly stale —
  fine for most cases.
- **Near-real-time for trending:** maintain approximate counts (**Count-Min Sketch**) and a sliding
  window; periodically merge a "trending" overlay into the served lists so breaking terms surface in
  minutes, not a day.
- Updating one node's top-k can ripple to ancestor nodes' top-k lists — another reason batch
  recompute + swap is cleaner than in-place mutation.

> **Say this in the room:** "Typeahead is a separate read-optimized service, not the search engine
> per keystroke. The core trick is **precomputed top-k completions per prefix**, served from
> memory/Redis behind a CDN, rebuilt in batch from query logs. Add fuzzy fallback with a Levenshtein
> automaton, debounce on the client, and surface trending with a Count-Min sketch overlay."

---

## Part F — Pagination, Faceting, and Latency

### Search-as-you-type latency budget

A rough p99 budget for a typeahead keystroke (target ~100ms end-to-end):

| Stage | Budget |
|---|---|
| Client debounce | 100–300ms (hides the rest; not counted in server budget) |
| Network (RTT) | 20–40ms |
| CDN/edge cache hit | ~5ms (most requests end here) |
| Service + Redis/trie lookup | 5–15ms |
| Fuzzy fallback (only on miss) | +10–30ms |

The whole game is **maximizing cache hits** so the median request never touches the service.

### Pagination: deep pagination is expensive

| Method | How | Use when |
|---|---|---|
| **`from` / `size` (offset)** | skip `from`, return `size` | Shallow paging only (first few pages of a UI). |
| **`search_after`** | pass the **sort values of the last result** as a cursor; "give me items after this point" | Deep pagination, infinite scroll, "next page." The default for going deep. |
| **Scroll / PIT** (Point-In-Time) | snapshot the index, iterate everything | Bulk export / reindex, **not** live user paging (holds resources, frozen view). |

**Why deep offset is expensive (recite this):** in a scatter-gather engine, `from=10000, size=10`
forces *every shard* to compute and return its top `10010`, the coordinator to merge
`num_shards × 10010` candidates, sort them, and **throw away 10000**. Cost scales with the offset, on
every shard, every request. ES caps it at `index.max_result_window` (default 10,000) for exactly this
reason. `search_after` is O(page size) instead — it uses the last sort value to *resume* the
posting-list walk, so there's nothing to skip-and-discard. The price: you can only go forward
sequentially (no random jump to "page 500"), which is fine for infinite scroll.

### Faceting / aggregations and their cost

Facets are the "Brand (42), Color (red 18, blue 9), Price < $50 (31)" sidebar counts. They're
**aggregations**: for the current result set, count documents grouped by field value. Cost drivers:

- They run over the **entire matching set**, not just the page — you can't shortcut with top-k.
- **High-cardinality** fields (group by user_id, by free-text) are expensive in memory and CPU.
- They need a **columnar** view of the field (Lucene **doc values** — a forward, on-disk
  column store separate from the inverted index) to iterate values per doc efficiently. This is why
  you set `doc_values` (on by default for `keyword`/numeric, off for analyzed `text`).
- Mitigate: facet only on `keyword`/numeric fields, cap cardinality, sample/approximate
  (`cardinality` uses HyperLogLog), cache common facet combinations.

---

## Part G — Decision: Search Engine vs DB LIKE vs Vector DB

| You need… | Use | Why / why not the others |
|---|---|---|
| Exact / prefix match, low cardinality, small data, occasional | **DB query (`=`, `LIKE 'foo%'`, trigram index)** | A real index handles `LIKE 'foo%'` (left-anchored) fine. No second system to sync. Postgres `pg_trgm`/`tsvector` covers light full-text. |
| Full-text relevance ranking, faceting, fuzzy, big corpus, high QPS | **Search engine (Elasticsearch/OpenSearch)** | Inverted index + BM25 + analyzers + facets. `LIKE '%foo%'` (leading wildcard) is a **full scan** and won't rank — that's the line where you graduate to a search engine. |
| Semantic / similarity / "find things *like* this", RAG, image/embedding search | **Vector DB or ES vector field** | ANN over embeddings. Lexical engines can't match by meaning. Use a dedicated vector DB (Pinecone/Milvus/Weaviate/pgvector) or ES `dense_vector` if you want one system. |

> **The cheap heuristic:** `LIKE '%term%'` (leading wildcard) or "rank by relevance" → you've outgrown
> the DB, reach for a search engine. "Find similar by meaning" → vector. Anything left-anchored,
> exact, and modest → keep it in the DB and avoid the sync tax.

**Never** forget the sync tax: every search engine and vector DB is a *second copy* of your data that
must be kept in sync (Part D). Don't add one for a problem a DB index solves.

---

## Part H — Worked Mini-Walkthrough: Search + Typeahead for a Large Product Catalog

**Prompt:** "Design search and typeahead for an e-commerce catalog: 50M products, 100M searches/day,
heavy on weekends, results in <300ms, typeahead <100ms."

**1. Requirements.**
- Functional: full-text product search (title, description, brand), filters/facets (category, price,
  brand, in-stock), sort (relevance/price/rating), typeahead.
- Non-functional: search p99 < 300ms; typeahead p99 < 100ms; ~1.2k search QPS avg, peak 3–4k;
  typeahead QPS far higher (per keystroke). Read-heavy ~100:1. Eventual consistency on the index is
  fine (new product searchable within seconds). Catalog is the source of truth — strong consistency
  on *that*.

**2. Estimation.** 50M docs × ~2KB ≈ 100GB raw → with index structures, fits in a handful of shards
across a few nodes; target ~30GB/shard → ~5 primary shards + 1 replica each. Typeahead structure is
tiny (top queries/products) → fits in RAM/Redis.

**3. Data model & store.** Catalog of record in a relational/OLTP DB (`products` table, transactions,
inventory). **Elasticsearch** as the search index (denormalized product doc: title, description,
brand `keyword`, category `keyword`, price, rating, in_stock, popularity). Mapping: `text` +
analyzer on title/description; `keyword` on brand/category for facets+sort; `dense_vector` for
optional semantic field.

**4. Keep in sync.** **CDC from the OLTP DB** (Debezium → Kafka) → an indexer consumer upserts into
ES by product ID, idempotent with external version = `updated_at`. No dual write. Inventory changes
(in_stock) flow the same way. Clients query the **alias** `products`.

**5. Query path.** Client → API gateway → search service → ES (alias). Service builds a query:
`multi_match` over title^3/brand^2/description with BM25, `post_filter` for facets, `aggregations`
for the facet sidebar, sort by relevance (default) or price/rating. Scatter-gather across 5 shards,
coordinator merges top-k. Paginate with **`search_after`** for infinite scroll, cap offset paging.
Cache hot/popular queries (head of the distribution) in Redis with short TTL.

**6. Typeahead path (separate subsystem).** Precompute **top-k completions per prefix** from
(a) the product titles/brands and (b) the query logs, weighted by search popularity + in-stock +
margin. Store as `prefix → top-k JSON` in **Redis**, fronted by a **CDN/edge cache**. Client debounces
~150ms. Fuzzy fallback via ES `completion`/fuzzy (Levenshtein automaton) only on near-empty exact
results. Rebuild the prefix map **in batch hourly** from Kafka query logs; swap atomically. Surface
trending terms via a Count-Min-sketch overlay merged every few minutes.

**7. Ranking evolution.** Start with **BM25 + field boosts + business signals** (boost in-stock,
down-rank low-rating). If we have click/purchase logs, add a **learning-to-rank reranker** over the
top ~500 BM25 candidates (features: BM25, CTR, conversion, price, recency). If users search by intent
("warm winter jacket"), add a **vector field** and do **hybrid retrieval (BM25 + ANN/HNSW, fused with
RRF)**.

**8. Reindex strategy.** New analyzer or field type → build `products_v2`, backfill from DB while CDC
double-writes to v1+v2, atomically swap the `products` alias, verify, drop v1. Zero downtime.

**9. Failure modes / wrap-up.** ES cluster down → serve a degraded experience from the Redis query
cache + DB `LIKE` fallback for exact SKU. Indexer lag → monitor Kafka consumer lag; new products
appear late but DB is still correct. Hot shard from a celebrity product/brand → custom routing or
more shards. Deep pagination abuse → enforce `search_after`, cap window. Typeahead cache stampede on
a viral term → request coalescing + jittered TTL.

---

### Self-check before the mock (answer these from memory)
- [ ] What is an inverted index, and what's in a single posting beyond the doc ID?
- [ ] Walk a `red AND shoes` query through the index. What do **skip pointers** and **WAND** buy you?
- [ ] Why is **BM25** the default over plain TF-IDF? Name its two corrections (`k1`, `b`).
- [ ] What's the difference between **refresh** and **flush** in Elasticsearch? Why are segments immutable, and what does **merging** do?
- [ ] Trace a query's **scatter-gather** (query phase vs fetch phase) and explain why **deep `from`/`size` pagination** is expensive. What replaces it?
- [ ] Describe the **dual-write problem** for a search index and the **CDC/outbox** fix. How does **alias-swap reindexing** give zero downtime?
- [ ] Design typeahead: what does **precomputed top-k per prefix** give you, where do you store it, and how do you handle **typos** and **updating popularity**?
- [ ] When do you use a **DB `LIKE`** vs a **search engine** vs a **vector DB**?
- [ ] What makes **faceting/aggregations** expensive, and what are **doc values**?
