# Design 17: Search Autocomplete / Typeahead (Google-Suggest Scale)

> **Why this problem is a great teacher:** autocomplete looks trivial — "show suggestions as the user
> types" — but at Google-suggest scale it is a masterclass in **decoupling an extreme read path from a
> slow write path**. The user fires a query *on every keystroke*, so your QPS is multiplied by word
> length and your latency budget is single-digit milliseconds *server-side* because it's hidden inside
> the gap between two keypresses. Meanwhile the *truth* you're serving — "what are the most popular
> queries starting with this prefix?" — comes from aggregating billions of historical queries, which
> is a slow, batchy, offline job. The entire design is the tension between those two clocks. The hero
> data structure is the **trie with precomputed top-K at every node**, and the hero architectural move
> is **building the trie offline and swapping it in atomically** so the read path never pays for the
> write path. Almost every building block shows up: a **trie/prefix index**, **precomputation**,
> **caching/CDN at the edge**, **sharding by prefix**, a **batch/stream aggregation pipeline**, and an
> **alias/atomic-swap deploy**.

This is a full 45-minute worked walkthrough using the 7-step framework from
`prep/01-framework-and-building-blocks.md`. Cross-references to building-block docs are inline.

---

## Step 1 — Requirements (5 min)

I'll drive this. "Autocomplete" can mean several things, so I'll pin scope first.

**Scope I'm designing:** the **search-box suggestion service** — as a user types a prefix, return the
**top-K most relevant completions** (whole queries, not spell-corrections of the current word),
ranked primarily by **historical popularity**, refreshed periodically. The corpus of suggestions is
**derived from query logs** (what other people searched), not a static dictionary. I am *not*
designing the search engine that runs after the user hits enter (that's `prep/11-search-systems.md`),
nor the query-log collection product itself — I consume its output. I'll treat personalization and
typo-tolerance as **deep-dive extensions**, not the core.

### Functional requirements (the verbs)

- **Suggest(prefix) → top-K completions**, ordered by relevance, returned **on every keystroke**.
- Suggestions are **ranked by popularity** (how often that full query was issued), with **recency** as
  a secondary signal (trending queries should surface).
- The suggestion set is **fresh-ish**: new popular queries should appear within hours, not seconds —
  but a brand-new viral query appearing within minutes is a *nice-to-have*, not core.
- **Filtering**: never suggest blocklisted / offensive / unsafe queries (a hard requirement at scale).
- (Extension) **Typo tolerance** — `"reciept"` should still surface `"receipt"`-rooted suggestions.
- (Extension) **Personalization** — bias toward this user's / this region's history.

### Non-functional requirements (the "how well" — where staff candidates separate)

| Dimension | Target / stance | Why it shapes the design |
|---|---|---|
| **Latency** | **p99 < ~50 ms end-to-end; server compute < ~10 ms** | It runs *between keystrokes*. Slower than the next keypress = useless. Forces precomputation + edge caching |
| **Read volume** | Enormous — **multiplied by keystrokes** (every char typed is a request) | The defining constraint. The whole design optimizes the read path |
| **Write/update path** | Slow, batchy, offline — rebuild from logs | We *deliberately* decouple it from reads; staleness of minutes–hours is fine |
| **Consistency** | **Eventual / stale-OK.** Suggestions need not reflect the last second of global queries | This is what *lets* us precompute and cache aggressively |
| **Availability** | Very high (99.9%+) but **degradable** — a missing suggestion is harmless | We can serve a slightly stale trie, or none, without breaking search |
| **Ranking quality** | Popularity-first, recency-aware, safe | Drives the offline aggregation + scoring pipeline |
| **Scale** | ~**5B searches/day**, prefixes drawn from ~**hundreds of millions** of distinct queries | Sizes the trie, the shard count, and the QPS math |

> **The single most senior framing to say out loud:** "This is a **read-mostly, staleness-tolerant**
> system where the read QPS is **multiplied by query length** and the latency budget is *hidden inside
> a keystroke gap*. So I will not compute anything at query time that I can precompute offline. The
> core trick is **storing the answer (top-K) at every prefix node ahead of time**, and **decoupling the
> trie *build* from the trie *serve*** so reads never wait on writes. Everything else is caching and
> sharding around that." Say that and the interviewer knows you understand the actual problem.

**Read vs write character:** wildly **read-heavy** (easily 10,000:1 effective, once you count
keystroke multiplication against a once-every-few-hours rebuild). That ratio justifies *every*
precompute-and-cache decision below.

---

## Step 2 — Estimation (3 min)

Numbers exist to justify boxes. Let me size the four things that drive design: **effective read QPS**
(the keystroke multiplier is the whole point), **trie size**, **top-K storage overhead**, and the
**log volume** feeding the build.

### Read QPS — and why keystrokes multiply it

Start from searches, then **multiply by the per-search keystroke count**, because every character the
user types fires a suggest request.

```
5e9 searches/day ÷ 86,400 s/day ≈ ~58,000 searches/sec (average)

But each search = a user typing a prefix. Say the average issued query is ~20 chars,
and a suggest request fires per keystroke (before debouncing):

  raw suggest QPS  ≈ 58,000 × 20  ≈ ~1.16 MILLION req/sec (average)
  peak (2–3×)      ≈ ~3 MILLION req/sec
```

That multiplier *is* the headline. It's also exactly why **client-side debouncing** matters: if we
fire only after a ~50–100 ms typing pause (and not on every single char), we cut that by **3–5×**:

```
  debounced suggest QPS ≈ ~250k–400k req/sec average, ~1M peak
```

> **Insight to voice:** "Naively this is a *million*-QPS service, an order of magnitude more than the
> search engine it feeds. Two cheap levers crush it before it reaches my servers: **debounce on the
> client**, and **cache prefixes at the edge/CDN**. Most prefixes are short and wildly popular
> (`"f"`, `"fa"`, `"fac"`…), so a CDN hit rate north of 90% is realistic. My origin fleet only sees
> the long-tail prefixes." That reframing — most of the load never reaches you — is the senior move.

### Trie size

Suppose **~400M distinct historical queries** are worth suggesting (after frequency-thresholding —
we drop the millions of queries seen once). Average query ~20 chars. A trie shares prefixes, so node
count ≈ total distinct characters after prefix-merging — empirically on the order of the total
characters divided by a sharing factor.

```
Upper-bound (no sharing): 400e6 queries × 20 chars ≈ 8e9 char-nodes — too pessimistic.
With prefix sharing (common heads collapse), assume ~3–5 nodes per query effective:
  nodes ≈ 400e6 × 4 ≈ ~1.6e9 nodes
```

Per node we store: children pointers (a map, costly), a flag for "is-a-word", and — the expensive
part — **the precomputed top-K list** (see next). A bare node is ~tens of bytes; *with* an embedded
top-K it's much more. So the raw trie skeleton is **single-digit to low-tens of GB**, but the
**top-K storage dominates** — that's the number that decides sharding.

### Top-K storage — the cost of precomputation

If we store top-K=**10** suggestions *at every node*, and each suggestion is a (string-ref + score)
≈ ~30 B (reference into a string pool, not a copy), that's ~300 B/node *just for the answers*:

```
top-K storage ≈ 1.6e9 nodes × 300 B ≈ ~480 GB   (!!)
```

That single number drives two decisions: **(1) we shard the trie** (480 GB + skeleton won't sit on
one box's RAM comfortably alongside QPS headroom), and **(2) we don't store full top-K at *every*
node** — we store it at every node but **dedup via references** and optionally only materialize top-K
at nodes above a depth/frequency threshold, computing the deepest few levels on the fly. (Tradeoff
explored in the deep dive.)

| Item | Size estimate | Drives |
|---|---|---|
| Trie skeleton (nodes + child maps) | ~tens of GB | RAM footprint |
| **Precomputed top-K per node** | **~480 GB** (K=10) | **Sharding** — the dominant cost |
| String pool (distinct query strings, deduped) | ~400M × ~40 B ≈ ~16 GB | Shared across nodes via refs |
| Total in-RAM working set | **~0.5 TB** | → shard across ~10–20 nodes + replicas |

### Log volume feeding the build

```
5e9 searches/day × ~50 B/log-record (query, ts, region, anonymized user bucket)
   ≈ ~250 GB/day of raw query logs
```

That's a **batch/stream aggregation** input (cross-ref `prep/18-batch-and-stream-processing.md`),
not something you touch on the read path. ~250 GB/day is trivial for a Spark/Flink job to roll up.

---

## Step 3 — API / Interfaces (3 min)

The external API is almost embarrassingly small — one read endpoint — which is itself a signal: *the
complexity is all behind it, in the index and the pipeline.*

**Public read API (the hot path):**
```
GET /suggest?q=<prefix>&k=10&lang=en&region=US        -> { suggestions: [ {text, score}, ... ] }
   - q:      the prefix typed so far
   - k:      number of suggestions (default 10)
   - cache:  responses are CDN-cacheable; short TTL (see Step 6)
```
- This is a **GET with cacheable query params** *on purpose* — it lets the CDN/edge cache it by full
  URL. Keep `k`, `lang`, `region` in the path/query so the cache key is complete.
- Auth/rate-limit live at the gateway (`prep/09-api-gateway-loadbalancing-ratelimiting.md`); a missing
  suggestion is harmless, so we rate-limit generously and **fail open** (return empty, never 500 the
  search box).

**Logging API (implicit, async):** when the user *submits* a search, the search service emits a
`QuerySubmitted{ query, ts, region, anonUserBucket }` event to the log stream. The autocomplete
service is a **consumer** of that stream, not the producer — clean seam.

**Admin / pipeline API (internal):**
```
POST /trie/deploy   { versionId, shardManifest }   -> atomically alias-swap live trie to versionId
GET  /trie/version  -> currently-serving versionId per shard
POST /blocklist     { terms[] }                    -> filter terms out at serve time
```

---

## Step 4 — Data Model (5 min)

Entities, keyed by **access pattern** (the access pattern picks the store, per the framework).

### 1. The Trie (prefix tree) — the serving structure
- **Access pattern:** given a prefix string, walk to its node, return the node's **precomputed top-K**.
  Pure point-walk + read; no scan, no aggregation, at query time.
- **Store:** an **in-memory trie** on the serving fleet, built offline and loaded read-only. Persisted
  as a serialized artifact (the "trie blob") in object storage; loaded into RAM on deploy. It is
  **immutable while serving** — never mutated in place by reads (this is the decoupling, detailed in
  the deep dive).

### 2. Aggregated query counts — the build input
- **Access pattern:** "for each distinct query, its total/recency-weighted count over the window."
  Bulk-aggregate read by the build job; never read on the hot path.
- **Store:** a **data-lake table** (Parquet in S3/GCS) produced by the batch/stream pipeline
  (`prep/18`). Columnar, append-heavy, OLAP — exactly the wrong thing to query at 1M QPS, which is
  *why* we compile it into the trie instead.

| Field | Type | Purpose |
|---|---|---|
| `query` (PK) | text (normalized, lowercased) | the suggestion candidate |
| `count` | int64 | total occurrences in window |
| `decayed_score` | float | time-decayed popularity (recency) |
| `region` / `lang` | text | for regional/lingual tries |
| `is_safe` | bool | passed safety/blocklist filter |

### 3. Raw query log stream — the source of truth for popularity
- **Access pattern:** append-only, consumed by both a **stream** job (fast, approximate, recent) and a
  **batch** job (slow, exact, full window).
- **Store:** a partitioned log (**Kafka**), retained, partitioned by query-hash or region
  (`prep/07-messaging-and-streaming.md`).

### 4. Blocklist / safety table
- **Access pattern:** membership test at serve time (is this suggestion allowed?).
- **Store:** small in-memory set replicated to every serving node; also applied at build time so most
  unsafe queries never enter the trie. Serve-time check is the last line of defense.

### 5. Trie version / shard manifest (deploy metadata)
- **Access pattern:** "which trie version + which shard owns which prefix range is live right now?"
- **Store:** a coordination store (**ZooKeeper/etcd**, `prep/08-consensus-and-coordination.md`) holding
  the **alias → versionId** pointer per shard and the **prefix → shard** routing map.

> **The data-model insight to voice:** "There are *two completely different data models* here. The
> **build side** is an OLAP aggregation table — columnar, batchy, queried in bulk. The **serve side**
> is an in-RAM trie — point-walk, read-only, latency-critical. The pipeline's whole job is to
> **compile** the former into the latter. Mixing them — querying the aggregation store live, or
> mutating the trie on every search — is the classic mistake."

---

## Step 5 — High-Level Design (10 min)

Let me get the happy path end-to-end, then evolve under questioning. There are **two loosely-coupled
subsystems**: the fast **read/serve path** (top) and the slow **build/update path** (bottom). They
meet only at the trie artifact.

```
        ╔══════════════════════════ READ / SERVE PATH (fast, 1M QPS) ══════════════════════════╗
        ║                                                                                        ║
 user   ║  keystroke (debounced) ─▶ ┌──────┐  miss  ┌─────────────┐  route   ┌───────────────┐  ║
 types  ║ ────────────────────────▶ │ CDN/ │ ─────▶ │ API GATEWAY │ ───────▶ │ SUGGEST SVC   │  ║
        ║   GET /suggest?q=fac      │ EDGE │        │ (LB, auth,  │  by      │ (stateless,   │  ║
        ║                           │ CACHE│ ◀───── │  ratelimit) │  prefix  │  routes to    │  ║
        ║   ◀── top-K JSON ──────── └──────┘  fill  └─────────────┘          │  trie shard)  │  ║
        ║                              ▲ 90%+ hit on short prefixes          └──────┬────────┘  ║
        ║                                                                            │ walk      ║
        ║                          ┌────────────────── TRIE SHARDS (in-RAM) ─────────▼──────┐    ║
        ║                          │ [shard a–c] [shard d–f] ... [shard w–z]  (+ replicas) │    ║
        ║                          │  each: immutable trie, top-K precomputed at each node │    ║
        ║                          └────────────────────────────────────────────────────┬─┘    ║
        ╚═══════════════════════════════════════════════════════════════════════════════│══════╝
                                                                       load artifact on  │ deploy
        ╔══════════════════════ BUILD / UPDATE PATH (slow, batchy) ═════════════════════│══════╗
        ║                                                                                ▼       ║
        ║  search   ┌────────────┐   ┌──────────────────┐   ┌──────────────┐   ┌──────────────┐ ║
        ║ submitted │   KAFKA    │──▶│  AGGREGATION JOB  │──▶│  TRIE BUILDER │──▶│ TRIE ARTIFACT│ ║
        ║ events ──▶│ query log  │   │ (batch: Spark exact│  │ (compute top-K│   │ store (S3) + │ ║
        ║           │ ~250 GB/day│   │  + stream: Flink   │  │  per node,    │   │ alias swap   │ ║
        ║           └────────────┘   │  recent/trending)  │  │  shard, serialize│ │ via etcd     │ ║
        ║                            └─────────┬──────────┘  └──────────────┘   └──────────────┘ ║
        ║                                      │ aggregated counts (Parquet)                     ║
        ║                            ┌─────────▼──────────┐                                      ║
        ║                            │  SAFETY / BLOCKLIST │ (filter at build + serve)           ║
        ║                            └────────────────────┘                                      ║
        ╚════════════════════════════════════════════════════════════════════════════════════════╝
```

### Walk one keystroke through it (out loud)

1. User types `f`, `a`, `c`. The client **debounces** (~50 ms) and fires `GET /suggest?q=fac`.
2. The request hits the **CDN/edge**. `"fac"` is a hugely popular short prefix → **cache hit**, top-K
   returned in a few ms without ever touching origin. (This is where most traffic dies.)
3. On a **miss** (long-tail prefix like `"facund"`), the edge forwards to the **API gateway** (auth,
   rate-limit, fail-open), which **routes by prefix** to the owning **trie shard** (e.g. prefixes
   `d–f` live on shard 2).
4. The **suggest service** on that shard **walks the trie** to the `"fac"` node — a few pointer hops —
   and reads the node's **precomputed top-K list**. No subtree traversal, no aggregation. Apply the
   serve-time **blocklist filter**, serialize, return.
5. The edge caches the response with a short TTL and serves it to subsequent typers of `"fac"`.

Meanwhile, **completely asynchronously**, the user's eventual submitted search flows into **Kafka**,
gets aggregated by the **batch/stream jobs**, and — hours later — is reflected in the **next trie
build** that gets atomically swapped in. *The read path never waited on any of that.*

### Why this shape

- **Precompute top-K at every node** so query time is O(prefix length) pointer-walk + one read, not a
  subtree scan. This is the core idea (deep dive 6.1).
- **Decouple build from serve** so a million reads/sec never contend with the write path, and the trie
  can be a lock-free, immutable, read-only structure (deep dive 6.3).
- **CDN/edge cache + client debounce** to kill 90%+ of the keystroke-multiplied load before origin.
- **Shard the trie by prefix** because the precomputed top-K (~0.5 TB) doesn't fit one box with
  headroom (deep dive 6.5).

---

## Step 6 — Deep Dives (15 min) — where the round is won

I'll propose the hardest parts up front: **the trie with precomputed top-K** (the data structure),
**why we don't update it on every query** (the decoupling), and **how we build/deploy it** (the
pipeline). Then ranking, serving/latency, typo-tolerance, and sharding.

### 6.1 The trie, in depth — and why we precompute top-K at every node

A **trie (prefix tree)** is a tree where each edge is labeled with a character and the path from root
to a node spells a prefix. All queries sharing a prefix share the path to that prefix's node; a node
flagged "terminal" means a complete query ends there.

```
            (root)
           /  |   \
          c   f    t
          |   |    |
          a   a    e
         /|   |   ...
        t r   c
       cat|   |
          re  e          "cat", "car", "care", "face", ...
```

**The naive query algorithm** is: walk to the prefix node (O(prefix length)), then **traverse the
entire subtree** below it to collect every completion, score them, and take the top-K. That subtree
can have *millions* of leaves for a short prefix like `"f"`. Doing that at query time, at 1M QPS,
within a 10 ms budget, is hopeless.

**The fix — precompute and store top-K *at every node*:** at build time, for each node, we compute the
K highest-scoring completions in its subtree and **store that list right on the node**. Now the query
is:

```
suggest(prefix):
    node = walk(root, prefix)        # O(len(prefix)) pointer hops
    if node is null: return []       # prefix not in trie
    return node.topK                 # ONE read — already the answer
```

**O(prefix length) total, no subtree traversal.** We traded **build-time compute and memory** for
**query-time speed** — the right trade in a 10,000:1 read-heavy system. Computing top-K bottom-up at
build time is cheap and elegant: a node's top-K is a **K-way merge of its children's top-K lists plus
its own terminal entry** (if it terminates a query). One post-order pass over the trie computes every
node's top-K in O(nodes × K) — we never recompute a subtree.

> **The tradeoff to name explicitly:** precomputing top-K is **memory-for-latency**. We saw in Step 2
> it costs ~480 GB at K=10 — by far the dominant memory cost. The alternative (traverse-on-read) is
> ~10× less memory but **orders of magnitude more latency** and unbounded fan-out on hot prefixes.
> For *this* workload, latency is sacred and RAM is cheap, so precompute wins decisively. (If memory
> were the binding constraint — say an embedded/mobile use — you'd flip it: store top-K only at the
> top few levels and traverse the small subtrees below.)

**Memory-cost mitigations worth mentioning:**
- **Reference, don't copy** the suggestion strings — a node's top-K stores *integer ids* into a shared
  **string pool**, so `"facebook"` is stored once even though it appears in the top-K of `"f"`, `"fa"`,
  `"fac"`, …. This is the difference between ~480 GB and several TB.
- **Don't materialize the deepest levels.** Below some depth/frequency threshold, subtrees are tiny
  (few completions), so traverse-on-read is fine there. Materialize top-K only where the subtree is
  big. Hybrid: precompute high, traverse low.
- **Compress the structure.** A plain trie has a costly child-map per node; a **radix/PATRICIA trie**
  (collapse single-child chains into one edge labeled with a substring) and **double-array tries** or
  **FST/DAWG** representations (used by Lucene for exactly this) cut the skeleton dramatically by also
  sharing *suffixes*, not just prefixes. Worth naming as the production reality.

### 6.2 Ranking — popularity, recency, personalization

The "top" in top-K is a **score**, computed at build time and stored with each suggestion:

- **Popularity (primary):** the count of how often that full query was issued in the window. This is
  the dominant signal and comes straight from the aggregation job.
- **Recency / trending (secondary):** raw lifetime counts would freeze the suggestions in the past and
  never surface a query that just went viral. So we use a **time-decayed score** — e.g. an
  exponentially-weighted moving average, or summing counts in time-bucketed windows with decay
  `score = Σ count_bucket × e^(−λ·age)`. Recent occurrences weigh more. This is also how a **stream**
  job can inject "trending now" suggestions ahead of the next full batch (6.4).
- **Quality/safety adjustments:** demote or drop unsafe, spammy, or low-CTR queries; a blocklist hard-
  removes the disallowed.
- **Personalization (high level):** the *core* trie is global. Personalization is a **rerank layer on
  top**, not a per-user trie (you can't store 1B tries). Approaches, cheapest first:
  - **Regional / lingual tries** — separate (smaller) tries per region/language; pick by `region`/
    `lang` param. Cheap, big quality win, no per-user state.
  - **Client-side personal history** — the user's own past searches live on the device; the client
    *merges* its local history into the server's global suggestions. Zero server state, great privacy.
  - **Server-side rerank** — fetch global top-N (N>K) from the trie, then rerank by a lightweight
    per-user feature vector (recent categories, location) and return top-K. Adds latency + state, so
    reserve it for logged-in surfaces where it pays off.

> **Tradeoff to name:** personalization fights caching. A per-user response is **uncacheable at the
> edge** — you lose the 90% CDN hit rate that makes the economics work. So I'd keep the **hot, cacheable
> global path** as the default and layer personalization only where it clearly wins (and accept it
> bypasses the edge cache). State that explicitly; it's the staff-level reasoning.

### 6.3 Building/updating the trie — and **why we do NOT update it on every query**

This is the architectural crux, and it ties straight to `prep/18-batch-and-stream-processing.md`.

**The naive idea:** every time a user searches, increment that query's count and update the top-K of
every prefix node along its path, live, in the serving trie. **Why this is wrong:**
1. **It couples a 1M-QPS read path to a write path.** Now every read might contend with a write for the
   same node's top-K list → locks, cache-line contention, GC pressure. Your immutable, lock-free,
   blisteringly-fast read structure becomes a mutable concurrent data structure — orders of magnitude
   slower and harder.
2. **A single search barely moves the ranking.** One increment among billions changes essentially no
   top-K list. We'd pay enormous per-write cost for ranking changes too small to matter.
3. **We don't need that freshness.** The requirements say staleness of *hours* is fine. Real-time
   ranking accuracy is not a product requirement, so paying for it is pure waste.

**So we decouple read from write** (the recurring framework theme — `prep/01` Part B "decoupling"):
the serving trie is **immutable**; we **rebuild it offline** from aggregated logs and **swap it in**.

**The build pipeline (cross-ref `prep/18`):**
```
Kafka query log
   │
   ├──▶ BATCH job (Spark, runs e.g. every few hours):
   │       group by query → exact counts over the window → time-decay → safety filter
   │       → write aggregated (query, score) table (Parquet)
   │
   └──▶ STREAM job (Flink, continuous):
           windowed approximate counts of the LAST few minutes → "trending" deltas
           (top-K heavy-hitters via Count-Min Sketch / Space-Saving, see prep/18)
                         │
                         ▼
   TRIE BUILDER: read aggregated table → build trie → post-order pass computes
                 top-K at every node → serialize sharded artifact → upload to S3
                         │
                         ▼
   DEPLOY: load artifact into a NEW set of (or warmed) serving nodes →
           health-check → ATOMIC ALIAS SWAP (etcd pointer flips live → new versionId)
```

**Periodic full rebuild + incremental updates — two cadences:**
- **Full rebuild** (batch, every few hours): exact, complete, the source of truth. Heavy but
  infrequent. Produces a fresh trie blob.
- **Incremental / trending overlay** (stream, minutes): the batch trie is hours stale, which is fine
  for the body of suggestions but misses a query that just went viral. The stream job emits a small
  set of **trending deltas**; the serving node keeps a tiny, separate **mutable overlay** (just the
  trending heavy-hitters) and **merges** it into results at query time. This is a *small, bounded*
  mutable structure — not the whole trie — so it doesn't reintroduce the write/read coupling problem.
  Best of both: stable precomputed base + fresh trending tip.

> **The tradeoff to name:** **freshness vs cost.** Rebuilding more often → fresher suggestions but more
> compute + more frequent disruptive swaps. Rebuilding less often → cheaper but staler. The
> stream-overlay is how we get *most* of the freshness benefit *without* paying for frequent full
> rebuilds — we rebuild the expensive base slowly and patch the fast-moving tip cheaply.

### 6.4 Deploy — atomic alias swap (no read ever sees a half-built trie)

The trie is loaded read-only into RAM. Deploying a new version must be **atomic from a reader's
perspective** — no request should ever traverse a partially-loaded or inconsistent tree.

- Build the new versioned artifact (`trie-v1234`) and **load it fully into RAM on the serving nodes
  alongside the currently-live one** (briefly 2× memory — sized for in Step 2).
- **Health-check / shadow-test** the new version (replay sample prefixes, compare quality, verify it
  loaded).
- Flip the **alias pointer** in etcd (`live → v1234`) — an atomic compare-and-swap. New requests read
  the new trie; in-flight requests finish on the old one. Then drop the old version's memory.
- **Rollback** is just flipping the alias back to the previous versionId — instant, no rebuild.

This is the classic **blue-green / immutable-artifact deploy** applied to a data structure: you never
mutate the thing serving traffic; you build a new one and swap the pointer.

### 6.5 Sharding the trie across servers — by prefix, and the routing

At ~0.5 TB working set + replicas + QPS headroom, the trie doesn't live on one box. **Shard it.**

**Shard by prefix range (the first 1–2 characters):**
```
shard 0: prefixes a–c     shard 1: prefixes d–f   ...   shard N: prefixes w–z, digits, other
```
- **Routing:** the suggest service (or gateway) inspects the **first character(s)** of the prefix and
  routes to the owning shard. A query for `"facebook"` → first char `f` → shard owning `d–f`. The
  **prefix → shard map** lives in etcd (`prep/08`); the routing tier watches it.
- **Why prefix-range, not hash(prefix)?** Because a suggest query *is* a prefix walk — all the data a
  query needs is **localized under its first characters**, so range-by-prefix keeps a whole query's
  trie path **on one shard**. Hashing the prefix would scatter the path across shards and you couldn't
  walk it locally. (Contrast with the crawler in Design 06, which hashes by host precisely *because*
  it wants to scatter — different access pattern, different choice. Naming that contrast is a senior
  tell.)

**The hot-shard problem (and fix):** letter frequency is wildly skewed — far more queries start with
`s` or `t` than with `x` or `z`. A naive A–Z split makes the `s`-shard a hot spot.
- **Fix 1 — balance by *traffic*, not alphabet.** Assign prefix ranges so each shard gets roughly equal
  *query volume* (measured from logs), not equal *letters*. The `s`-prefix might be its own shard;
  `x`,`y`,`z` might share one.
- **Fix 2 — split deeper.** Shard hot single-letter prefixes by their second character (`sa–sm` on one
  shard, `sn–sz` on another).
- **Fix 3 — replicate hot shards more.** Reads are stateless against an immutable trie, so we can put
  **many read replicas** behind hot shards. This is the main lever — read scaling is *easy* here
  precisely because the trie is read-only.
- And remember: **the CDN already absorbs the hottest (shortest) prefixes**, so origin shards mostly
  see the long tail, which is *less* skewed than raw traffic.

> **Why sharding reads is easy here:** the serving trie is **immutable and read-only**, so a shard is
> trivially replicable — no write coordination, no replication lag to reason about, no consistency
> question. Scaling reads = "add more identical read replicas of the shard." That's the payoff of the
> decoupling in 6.3. Say it: "**I made reads easy by making the read structure immutable.**"

### 6.6 Serving path & the latency budget per keystroke

The budget is brutal and worth itemizing out loud:
```
total user-perceived budget ............ ~50–100 ms (must beat the next keystroke)
  client debounce wait ................. ~50 ms (deliberate — cuts QPS, costs a little latency)
  network RTT (client→edge) ............ ~10–30 ms (the edge is close — that's the point of CDN)
  edge cache lookup (on hit) ........... ~1–3 ms  ← ~90% of requests end HERE
  --- on miss, additionally: ---
  edge→origin RTT + gateway ............ ~5–20 ms
  trie walk + read top-K ............... < ~1 ms  (pointer hops + one list read)
  serialize + blocklist filter ......... ~1–2 ms
```
- The **server compute is sub-millisecond** *by design* — that's the whole point of precomputing
  top-K. The budget is spent on **network**, which is why we push the data to the **edge**.
- **Client debounce** (~50 ms idle) is the cheapest possible optimization: it both cuts QPS 3–5× and
  avoids firing requests for prefixes the user is typing straight through. Pair it with **request
  cancellation** (cancel the in-flight request when a new keystroke arrives) so stale responses don't
  flicker.
- **Edge caching** with a **short TTL** (seconds to low minutes): suggestions are staleness-tolerant,
  and short prefixes are stable, so a short TTL gives a huge hit rate while bounding staleness.
  (`prep/06-caching-deep-dive.md`.) Watch for the **thundering herd** on cache expiry of a hot prefix
  — use request coalescing / stale-while-revalidate at the edge so one origin fetch refills it.

### 6.7 Typo tolerance / fuzzy matching (high level)

Exact-prefix matching means `"reciept"` returns nothing useful. Real autocomplete is **fuzzy**.

- **Edit distance (Levenshtein):** define a match as "a known query within edit distance ≤ d of the
  typed prefix" (insertions/deletions/substitutions/transpositions). The challenge is doing it fast —
  you can't compute edit distance against 400M strings per keystroke.
- **The fast way — a Levenshtein automaton over the trie/FST:** Lucene's approach. Build a
  finite-state automaton that accepts all strings within distance `d` of the input, then **intersect
  it with the trie/FST** — you walk the trie and the automaton in lockstep, pruning any branch that
  can't possibly stay within distance `d`. This finds all fuzzy matches without enumerating the
  dictionary. State the name; you don't need to derive it in the room.
- **Cheaper approximations** worth mentioning: index common misspellings as alternate trie entries
  (data-driven, from the logs themselves — people who typed `"reciept"` then `"receipt"`); a
  **symspell**-style precomputed deletion dictionary; or **phonetic** keys (Soundex/Metaphone) for
  sound-alike matches.

> **Tradeoff:** fuzzy matching trades precision and CPU for recall. Keep `d` small (1, maybe 2 for
> longer prefixes), and **bias exact-prefix matches above fuzzy ones** in the ranking so a clean prefix
> isn't drowned by typo-corrections. It's a quality knob, not a free win.

### 6.8 Filtering & safety (don't skip this at scale)

Unsafe/offensive suggestions are a real product and PR hazard. Defense in depth:
- **At build time:** the aggregation job drops blocklisted and policy-violating queries, so most never
  enter the trie. This is the bulk filter.
- **At serve time:** a final blocklist membership check on the returned top-K (the blocklist updates
  faster than the trie rebuilds, so this catches newly-banned terms immediately). Cheap in-memory set.
- **Threshold by frequency:** only suggest queries seen by **enough distinct users** (a frequency
  floor), which both improves quality and naturally filters out rare malicious/PII queries — a single
  user's odd query never becomes a global suggestion. (This also doubles as privacy protection.)

---

## Step 7 — Wrap-Up (3 min)

**Bottlenecks (in order of how much they actually bite):**
1. **Read QPS multiplied by keystrokes** — the *designed-around* bottleneck. Beaten by **debounce +
   edge cache** (kills ~90%+) and **read replicas** of immutable shards.
2. **Top-K memory footprint (~0.5 TB)** — the cost of precomputation; drives sharding and the
   string-pool/reference trick.
3. **Hot prefix shards** (letter-frequency skew) — beaten by traffic-balanced ranges + more replicas
   + the CDN already eating the hottest prefixes.
4. **Trie build/swap cost** — heavy but infrequent; the freshness/cost knob.

**Failure modes & SPOFs:**
- **A trie shard dies** → its prefix range loses suggestions. Mitigate with **replicas per shard** and
  **fail-open** (return empty, never break the search box). A missing suggestion is harmless.
- **Bad trie build / corrupt artifact** → health-check before the alias swap; the **atomic alias swap
  + instant rollback** means a bad version is never live for long.
- **CDN/edge outage** → load falls through to origin; origin must be sized for a *fraction* of full
  load, so size with headroom or shed gracefully (serve fewer suggestions, raise debounce).
- **Pipeline (Kafka/Spark/Flink) outage** → the trie simply goes stale; **serving is unaffected**
  because it's decoupled. Staleness is tolerable for hours — graceful degradation by design.
- **etcd (routing/alias) down** → routing map and live-version pointer are unavailable; serving nodes
  **cache the last-known map** and keep serving the current trie. Quorum cluster for etcd.
- **Thundering herd** on hot-prefix cache expiry → request coalescing / stale-while-revalidate.

**What I'd do with more time:**
- A learned ranker (CTR + features) instead of pure popularity decay.
- Per-region/-language tries as first-class, plus language detection on the prefix.
- Richer trending (anomaly detection on the stream) for breaking-news spikes.
- Multi-token / mid-query suggestion (suggest the *next word*, not just completions of the last).

**Scaling story in one line:** scale **reads** by adding replicas of **immutable** trie shards behind a
**CDN that absorbs the hot short prefixes**, and scale **freshness** independently by tuning the
**rebuild cadence + stream overlay** — the two clocks (serve and build) scale separately *because we
decoupled them*.

---

## What Made This Staff-Level

- **Reframed the problem correctly up front:** "read QPS multiplied by keystrokes, latency hidden in a
  keystroke gap, staleness-tolerant → precompute the answer and decouple build from serve." Naming the
  real constraint before drawing boxes is the single biggest senior tell.
- **The trie with precomputed top-K at every node**, with the explicit O(prefix length) vs
  subtree-traversal tradeoff, the **bottom-up K-way-merge build**, and the **string-pool reference**
  trick — plus a *numeric* memory justification (~480 GB) that drove sharding.
- **The decoupling argument**: *why* you don't update the trie on every query (read/write contention,
  negligible ranking impact, unneeded freshness), and the **immutable trie + atomic alias swap** that
  makes reads lock-free and trivially replicable.
- **Two cadences** (slow exact batch rebuild + fast approximate stream overlay for trending), framed as
  the **freshness-vs-cost** knob, tied cleanly to `prep/18`.
- **Prefix-range sharding with traffic-balanced ranges**, and explaining *why* prefix-range beats
  hash-by-prefix here (locality of the walk) — explicitly contrasted with the crawler's host-hash.
- **An itemized latency budget per keystroke** showing server compute is sub-ms by design and the
  budget is really network → push to the edge; **debounce + CDN** kill most load before origin.
- **Failure modes named before being asked** — and the recurring theme that **fail-open is acceptable
  because a missing suggestion is harmless**, which is itself a product-aware, staff-level judgment.
- **Every number drove a decision** (1M QPS → debounce+CDN; 0.5 TB → shard; 250 GB/day → batch pipeline).

## Self-Check (answer from memory before the mock)

- [ ] Why does read QPS get *multiplied*, and what are the two cheapest levers to cut it before origin?
- [ ] What does precomputing top-K at every node buy you, what does it cost, and how is top-K computed
      at build time (the bottom-up merge)? How do you keep the memory cost down?
- [ ] Why do we NOT update the trie on every query? Give the three reasons.
- [ ] Describe the build pipeline end-to-end (Kafka → aggregate → build → deploy) and the two cadences
      (batch rebuild vs stream trending overlay). What's the freshness-vs-cost tradeoff?
- [ ] What is the atomic alias swap and why does it matter for a read-only in-RAM structure?
- [ ] Why shard the trie by prefix range and not by hash(prefix)? How do you handle the hot-shard skew?
- [ ] Give the per-keystroke latency budget. Where does the time actually go, and why is server compute
      sub-millisecond?
- [ ] How does fuzzy/typo matching work at a high level (Levenshtein automaton over the trie/FST), and
      what's the precision tradeoff?
- [ ] Why is scaling *reads* unusually easy in this design? (One sentence.)
