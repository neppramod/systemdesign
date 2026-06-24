# Design 06: Web Crawler (Google-Scale)

> **Why this problem is a great teacher:** a crawler looks deceptively simple — "fetch a page,
> extract links, repeat" — but at Google scale it becomes a masterclass in *flow control under
> politeness constraints*. The whole design tension is: you want to crawl as fast as physically
> possible (billions of pages), but you are *not allowed* to hammer any single host, you must not
> re-fetch what you've seen, and you must keep a freshness budget over an effectively infinite,
> adversarial space. Almost every building block shows up: a **queue** (the frontier), **Bloom
> filters** (dedup), **consistent hashing** (partition by host), **object storage** (raw pages), a
> **DNS cache**, and **backpressure** everywhere. This is the canonical "producer feeds itself" design.

This is a full 45-minute worked walkthrough using the 7-step framework from
`prep/01-framework-and-building-blocks.md`. Cross-references to building-block docs are inline.

---

## Step 1 — Requirements (5 min)

I'll drive this. A "web crawler" can mean three very different products, so I'll pin scope first.

**Scope I'm designing:** a general-purpose crawler that discovers and downloads the public web,
stores raw pages, and **feeds a downstream indexing/search pipeline** (cross-ref
`prep/11-search-systems.md`). I am *not* designing the search index, ranking, or query serving here —
the crawler's output contract is "raw pages + a link graph in storage, plus a stream of
fetched-document events." I'll also skip JavaScript rendering as the default path (call it out as an
extension), and skip authenticated/paywalled content.

### Functional requirements (the verbs)

- **Seed** the crawl with an initial URL set.
- **Fetch** a page over HTTP(S) given a URL.
- **Parse** the page; **extract outbound links** (and metadata: title, content, last-modified).
- **Normalize + dedup** discovered URLs, and feed new ones back into the frontier (the loop).
- **Store** the raw page (object store) and the **link graph** (who links to whom).
- **Recrawl** pages over time to keep content fresh (re-discovery, not just first discovery).
- **Respect politeness**: obey `robots.txt`, crawl-delay, and per-host rate limits.
- **Be extensible**: pluggable per-content-type processing (HTML now; images, PDFs, JS-rendered
  pages later) without redesigning the core loop.

### Non-functional requirements (the "how well" — where staff candidates separate)

| Dimension | Target / stance | Why it shapes the design |
|---|---|---|
| **Scale** | Crawl ~**1B pages/day**; maintain a corpus of ~**40B pages** | Drives sharding, frontier size, storage tiering |
| **Politeness** | Never exceed per-host rate limit; obey `robots.txt`/crawl-delay | **The hardest constraint.** Forces a host-aware frontier + worker assignment |
| **Throughput over latency** | This is a **batch/throughput** system, not request/response | We optimize pages/sec aggregate, not p99 of a single fetch |
| **Freshness** | News re-crawled in minutes–hours; static pages in weeks | Drives an **adaptive recrawl scheduler** with priority |
| **Dedup** | URL dedup (don't fetch the same URL twice) + **content** dedup (don't store near-duplicate pages) | Two distinct mechanisms (Bloom filter vs SimHash) |
| **Robustness** | Survive crawler traps, infinite spaces, malformed/huge pages, slow/dead servers | Timeouts, depth/budget limits, trap detection |
| **Availability** | Crawler can tolerate worker death; no SPOF in the loop | Frontier must be durable + partition-tolerant |
| **Extensibility** | Add content processors without touching the frontier | Plugin/strategy pattern at the parser stage |

> **The single most senior framing to say out loud:** "This system is **bottlenecked on politeness
> and network/DNS, not on CPU.** I can parse HTML far faster than I'm *allowed* to fetch it. So the
> entire architecture is really a giant, distributed, *politeness-aware rate limiter* wrapped around a
> BFS of the web graph." Say that and the interviewer knows you understand the actual problem.

**Read vs write character:** unusual for this series — the crawler is *write-heavy to storage* and
its "reads" are outbound HTTP fetches to the open internet (which we don't control). There's no user
QPS. The load is self-generated and we throttle it ourselves.

---

## Step 2 — Estimation (3 min)

Numbers exist to justify boxes. Let me size the four things that drive design: **page volume**,
**crawl rate**, **storage**, **bandwidth**. (Method per `prep/02-estimation-and-napkin-math.md`.)

### Crawl rate

We want **1B pages/day**.

```
1e9 pages/day ÷ 86,400 s/day ≈ 11,600 pages/sec  (average)
peak ≈ 2× ≈ ~23,000 pages/sec
```

That's the aggregate fetch rate the whole fleet must sustain. The politeness constraint means we
*cannot* get there by hammering a few hosts — we get there by **fanning out across millions of
hosts in parallel**, each crawled gently.

### How many workers / connections?

A single page fetch is dominated by network RTT + server response, call it **~500 ms wall-clock**
per fetch (DNS + connect + transfer + server think time; many sites are slow). But fetches are I/O
bound, so one worker holds **thousands of concurrent connections** (async I/O / event loop).

```
required concurrency = rate × latency = 23,000 pages/s × 0.5 s ≈ 11,500 in-flight fetches
```

So ~12k concurrent connections at peak. With, say, **5,000 connections per worker node**, that's
**~3–5 worker nodes** for raw fetching — tiny. The fleet is sized by **parsing + storage + frontier
management + headroom + geo-distribution**, not by the fetch concurrency itself. Realistically run
**dozens to a few hundred** nodes for redundancy, regional presence, and processing throughput.

> **Insight to voice:** "The fetch tier is almost embarrassingly small in raw connection terms —
> politeness, not hardware, is the limiter. The fleet grows for *parsing, dedup, storage I/O, and
> regional spread*, not to push more bytes."

### Storage

| Item | Per-page | × Corpus (40B pages) | Notes |
|---|---|---|---|
| **Raw HTML (compressed)** | ~100 KB raw → ~**25 KB** gzipped (4:1) | **~1 PB** | Object store; compress on the way in |
| **Page metadata** (URL, hash, fetch time, status, content-hash, ETag) | ~**500 B** | **~20 TB** | Hot lookup store (key-value) |
| **Link graph** (avg ~10 outlinks/page, edge ≈ 16 B as IDs) | ~**160 B** | **~6.4 TB** edges (~400B edges) | Adjacency lists; feeds PageRank-style scoring |
| **URL "seen" set** (40B URLs, fingerprinted) | — | **~80 GB** as a Bloom filter (see below) | Fits in RAM, sharded |

So raw page storage dominates at **~1 PB for the corpus**, growing by **~25 GB/day** of *new* unique
content (1B fetches/day, but most are re-crawls or near-dupes — net new unique is far smaller). Over
3 years with growth and recrawl versioning you're at the **multi-PB** scale → **object storage
(S3/GCS) with lifecycle tiering**, never a database (cross-ref `prep/14-blob-storage-and-media.md`).

### Bandwidth (ingress)

```
23,000 pages/s × 100 KB/page (uncompressed wire) ≈ 2.3 GB/s ≈ ~18 Gbps inbound at peak
```

~18 Gbps of sustained download. Non-trivial but well within a datacenter's capacity; the cost story
is *egress to the internet for requests is tiny; ingress of page bodies is the volume*. This also
says: **compress at the edge of the fetcher before it crosses internal links**, and **stream parse**
rather than buffering whole pages where possible.

### URL "seen" set — why Bloom filter, with numbers

We check "have I seen this URL?" on **every extracted link** — that's ~10 checks per fetched page →
**~230k membership checks/sec** at peak. A hash set of 40B URLs at, say, 50 B each = **2 TB** of RAM
→ too expensive to keep exactly in RAM. So:

```
Bloom filter, 40B items, target false-positive p = 1%:
  bits ≈ -n·ln(p) / (ln2)²  ≈  40e9 × 4.6 / 0.48  ≈  ~3.8e11 bits ≈ ~48 GB
  (≈ ~9.6 bits/element; ~7 hash functions)
```

**~48 GB of RAM** (sharded across nodes) instead of 2 TB, at the cost of a **1% false-positive rate**
— meaning occasionally we *skip a URL we haven't actually seen*. That's an acceptable loss for a
crawler (we'll discover it via another inlink later). Bloom filters can't have false negatives, so we
**never re-fetch something we've truly seen** — exactly the guarantee we want. (Cross-ref the Bloom
filter building block in `prep/01-framework-and-building-blocks.md` Part B.)

---

## Step 3 — API / Interfaces (3 min)

A crawler has almost no external user API; its "API" is **internal service contracts** plus the
**output contract** to downstream consumers. That framing alone is a senior signal.

**Control / seed API (operators):**
```
POST /seeds            { urls: [...], priority }        -> accepted
POST /recrawl          { urlPattern, priority }         -> schedule a refresh
GET  /stats            -> { pagesPerSec, frontierDepth, perHostBacklog }
```

**Internal frontier interface (the heart of the loop):**
```
frontier.enqueue(url, priority, host)      // dedup + politeness routing happen here
url = frontier.dequeue(workerId)           // returns a URL this worker is ALLOWED to fetch now
                                           //   (host not rate-limited, robots OK)
```

**Output contract to the indexing pipeline** (cross-ref `prep/11-search-systems.md` and the queue in
`prep/07-messaging-and-streaming.md`):
```
// emitted on the "fetched documents" topic (Kafka)
FetchedDoc {
  url, finalUrl (after redirects), fetchTime, httpStatus,
  contentHash (SimHash), rawBlobRef (s3://...), title, extractedText,
  outlinks: [...], lastModified, etag
}
```

The indexer subscribes to this topic; it never talks to the crawler synchronously. **Decoupling via a
log** means the crawler and indexer scale and fail independently.

---

## Step 4 — Data Model (5 min)

Entities, keyed by **access pattern** (the access pattern picks the store, per the framework).

### 1. URL Frontier (the priority + politeness queue)
- **Access pattern:** enqueue by (priority, host); dequeue the highest-priority URL whose host is
  *currently eligible* to be crawled. This is not a plain FIFO — it's a **two-level priority queue
  with per-host gating** (detailed in the deep dive).
- **Store:** durable queue. Conceptually thousands of per-host FIFO queues plus a priority selector.
  Backed by a partitioned log/queue (Kafka topics, or RocksDB-backed queues per shard) for
  durability so a worker crash doesn't lose the backlog.

### 2. URL Seen-Set
- **Access pattern:** point membership test, extremely high QPS, approximate-OK.
- **Store:** **sharded Bloom filter** in RAM (partitioned by URL-hash), backed by a durable
  key-value store (the "URL metadata" table) for authoritative lookups when needed.

### 3. URL / Page Metadata table
- **Access pattern:** point lookup by URL-fingerprint → last fetch time, status, ETag/Last-Modified,
  content-hash, next-recrawl-time.
- **Store:** a **wide-column / KV store** (Cassandra/Bigtable/DynamoDB), partitioned by
  URL-fingerprint. Key = `fingerprint(normalizedURL)`. (Cross-ref `prep/03-databases-deep-dive.md`,
  `prep/04-sharding-and-partitioning.md`.) Write-heavy, point-read — classic NoSQL fit.

| Field | Type | Purpose |
|---|---|---|
| `url_fp` (PK) | bytes (64-bit hash) | partition + dedup key |
| `url` | text | canonical normalized URL |
| `last_fetch_ts` | timestamp | freshness scheduling |
| `http_status` | int | skip/retry logic |
| `etag` / `last_modified` | text | conditional GET (304s) |
| `content_simhash` | int64 | near-dup detection |
| `next_recrawl_ts` | timestamp | adaptive scheduler reads this |
| `change_rate_est` | float | EWMA of observed change frequency |

### 4. Raw Page Store
- **Access pattern:** write-once, read-rarely (batch reads by the indexer). Huge objects.
- **Store:** **object storage** (S3/GCS), key = `url_fp + fetch_ts`, body = gzipped HTML. Lifecycle:
  hot → infrequent-access → cold/glacier as pages age. (`prep/14-blob-storage-and-media.md`.)

### 5. Link Graph
- **Access pattern:** "outlinks of page X" (write) and later "inlinks of page Y" (for importance
  scoring). Effectively an adjacency list, batch-analytics read pattern.
- **Store:** columnar/append in the data lake (Parquet) or a graph-friendly KV; processed in batch by
  the importance/PageRank job (`prep/18-batch-and-stream-processing.md`).

### 6. robots.txt cache & DNS cache
- **Access pattern:** point lookup by host, high read QPS, short-to-medium TTL.
- **Store:** in-memory cache (per-node) + shared Redis (`prep/06-caching-deep-dive.md`).

---

## Step 5 — High-Level Design (10 min)

Let me get the happy path end-to-end, then evolve under questioning.

### The crawl loop (the spine)

```
              ┌─────────────────────────────────────────────────────────────┐
              │                      THE CRAWL LOOP                           │
              │                                                               │
   seeds ──▶  ┌──────────────┐    dequeue (eligible)   ┌──────────────┐       │
              │ URL FRONTIER │ ──────────────────────▶ │   FETCHERS   │       │
              │ (priority +  │                          │ (async HTTP, │       │
              │  politeness) │ ◀──┐                     │  DNS cache,  │       │
              └──────────────┘    │                     │  robots chk) │       │
                    ▲             │                     └──────┬───────┘       │
                    │ new URLs    │ enqueue                    │ raw bytes     │
                    │ (deduped)   │                            ▼               │
              ┌─────┴────────┐    │                     ┌──────────────┐       │
              │   URL DEDUP  │ ◀──┘                     │   PARSER /   │       │
              │ (Bloom +     │                          │  EXTRACTOR   │       │
              │  normalize)  │ ◀───────────────────────│ (links, text,│       │
              └──────────────┘     extracted outlinks   │  metadata)   │       │
                                                        └──────┬───────┘       │
              ┌──────────────┐    content-hash check           │              │
              │ CONTENT DEDUP│ ◀───────────────────────────────┤              │
              │  (SimHash)   │                                  │              │
              └──────────────┘                                  ▼              │
              ┌──────────────┐   raw page   ┌──────────────┐  ┌────────────┐   │
              │ OBJECT STORE │ ◀────────────│   STORAGE    │─▶│  Kafka:    │───┼──▶ Indexer
              │  (S3, 1PB)   │              │   WRITER     │  │ FetchedDoc │   │   (search
              └──────────────┘              └──────────────┘  └────────────┘   │    pipeline)
              ┌──────────────┐  metadata + link graph                          │
              │ METADATA KV  │ ◀────────────────────────────                   │
              │ + LINK GRAPH │                                                  │
              └──────────────┘                                                  │
                                                                                │
              ┌──────────────┐  reads next_recrawl_ts, re-enqueues             │
              │  RECRAWL     │ ───────────────────────────────────────────────▶│
              │  SCHEDULER   │                          (feeds frontier)        │
              └──────────────┘                                                  │
              └────────────────────────────────────────────────────────────────┘
```

### Walk one URL through it (out loud)

1. **Frontier** hands a worker a URL whose host is eligible *right now* (not rate-limited, robots
   allows it).
2. **Fetcher** resolves the host via the **DNS cache**, opens an async connection, issues a
   (conditional) GET. Applies a timeout. Gets bytes (or a 304 / error).
3. **Parser** decompresses, extracts text + metadata + **outlinks**. Emits the page.
4. **Content dedup** computes a SimHash; if near-identical to a known page, we **don't store a new
   copy** (we still record we saw the URL).
5. **Storage writer** writes raw HTML to the **object store**, metadata+links to the **KV/graph**, and
   emits a `FetchedDoc` to **Kafka** for the indexer.
6. Each outlink goes through **normalize → URL dedup (Bloom)**. New URLs are **enqueued back into the
   frontier** — this is the feedback loop that makes it a BFS over the web graph.
7. The **recrawl scheduler** independently re-enqueues already-seen URLs when their `next_recrawl_ts`
   arrives.

> **BFS framing:** the frontier is a queue, so the natural traversal is **breadth-first** — we explore
> shallow, high-value pages before going deep. That's deliberate: BFS reaches important pages (close
> to seeds, high inlink count) sooner, and it avoids the rabbit-holes that DFS falls into. We bias it
> further with priority (next section).

### Why a queue, why async, why decoupled

- The frontier **is** the queue building block (`prep/07-messaging-and-streaming.md`) — it absorbs the
  fact that we discover URLs in bursts but must drain them at a politeness-limited rate. That gap *is*
  the buffer's job.
- Fetching is **async I/O** because it's network-bound: one thread, thousands of sockets.
- Parsing/storage are decoupled behind the queue + Kafka so a slow object store can't stall fetching
  (backpressure pushes back into the frontier, not into dropped pages).

---

## Step 6 — Deep Dives (15 min) — where the round is won

I'll propose the two hardest parts up front: **the URL frontier (politeness + priority)** and
**dedup (URL + content)**. Then DNS, distribution, traps, and freshness.

### 6.1 The URL Frontier in depth — the two-level front-queue/back-queue design

This is the crux. The frontier must satisfy **two competing goals simultaneously**:

1. **Prioritization** — crawl *important* and *fresh* pages first (high PageRank, news, frequently
   changing). This wants a **priority** ordering.
2. **Politeness** — never send more than one request at a time (or 1 per crawl-delay seconds) to any
   single host, regardless of how many high-priority URLs that host has. This wants a **per-host**
   ordering with rate gating.

A single priority queue can't do both: if `cnn.com` has 10,000 high-priority articles, a naive
priority queue would dispatch them all at once and **DDoS the host**. The classic solution
(Mercator-style) is a **two-level queue**:

```
                        ┌───────────────── FRONT QUEUES (prioritization) ──────────────────┐
   enqueue(url) ──▶  prioritizer ──▶  [ F1 ] [ F2 ] ... [ Fk ]   (k priority bands, FIFO each)
                        │              high prio ............. low prio
                        │
                        ▼
                   ┌───────────────────── BACK QUEUES (politeness) ───────────────────────┐
                   each back queue B_i holds URLs for EXACTLY ONE host
                   [ B1: only host A ] [ B2: only host B ] ... [ Bm: only host Z ]
                        │
                        ▼
                   ┌──────────────── HOST→QUEUE TABLE + MIN-HEAP of "next-ready times" ────┐
                   heap keyed by earliest time each back-queue is allowed to be read again
                        │
                        ▼
                   worker pulls from the back queue whose host is ready NOW
```

**Front queues (the *what to crawl next* layer):** `k` FIFO queues, one per priority band. A
**prioritizer** assigns each incoming URL a band from a score: importance (inlink count / PageRank
estimate), freshness need (news domains, change rate), and depth penalty. Higher bands are drained
preferentially.

**Back queues (the *who we're allowed to hit* layer):** `m` FIFO queues, with the **invariant that
each back queue contains URLs for exactly one host**, and **each active host maps to exactly one back
queue**. A router moves URLs from front → back, maintaining that mapping. When a back queue empties,
its host slot is freed and refilled from the front queues (picking by priority).

**The politeness gate — a min-heap of host-ready timestamps:** a priority queue (min-heap) holds, for
each back queue, the **timestamp at which that host may be contacted again** = `last_fetch_time +
crawl_delay`. A worker thread:
1. pops the heap entry with the **earliest ready time**,
2. if it's in the future, sleeps until then (or picks another that's ready),
3. fetches one URL from that back queue,
4. pushes the entry back with `now + crawl_delay(host)`.

This **guarantees at most one in-flight request per host** and enforces crawl-delay *by construction*
— politeness isn't a check we hope to pass, it's a structural invariant of the data structure.

> **The tradeoff to name:** front queues optimize *importance/throughput*; back queues optimize
> *politeness*. They pull in opposite directions. The two-level split is what lets us honor politeness
> **without** sacrificing global prioritization — high-priority URLs still jump the front-queue line,
> they just can't violate a host's rate when they reach the back. If `#hosts < #workers`, some workers
> idle (politeness-bound); if `#hosts ≫ #workers`, we're throughput-bound — both are fine and
> self-balancing.

**Number of back queues `m`:** roughly **2–3× the number of worker threads**, so workers rarely block
waiting for a ready host. With millions of hosts in the backlog, only `m` are "active" at once;
the rest wait in front queues.

**Durability:** the frontier must survive crashes — we'd lose days of crawl progress otherwise. Back
it with a **persistent, partitioned log** (RocksDB-backed queues per shard, or Kafka topics keyed by
host). A worker checkpoints its position; on crash, another worker resumes that host's queue.

### 6.2 Dedup — two completely different problems

**(a) URL dedup — "have I already queued/fetched this exact URL?"**

First **normalize** (canonicalize) so trivially-different URLs collapse to one:
- lowercase scheme+host, strip default ports (`:80`/`:443`),
- remove fragments (`#...`), sort or strip tracking query params (`utm_*`, `gclid`),
- resolve `.`/`..`, decode unnecessary percent-encoding, strip trailing `/` consistently,
- (carefully) collapse known session-id params.

Then fingerprint (64-bit hash) and test membership against the **sharded Bloom filter** (sized at
~48 GB for 40B URLs, 1% FP, from Step 2). Bloom gives **no false negatives** → we never refetch a
truly-seen URL; the 1% **false positives** mean we rarely skip a genuinely-new URL, which is
acceptable (it'll be rediscovered via another inlink). For the small fraction where correctness
matters (operator-forced recrawl), fall back to the authoritative KV lookup. (Bloom filter building
block: `prep/01`; sharding: `prep/04`.)

> Why not an exact distributed hash set? 40B × ~50 B = **~2 TB RAM** and a network round trip per
> check at 230k QPS. The Bloom filter is **40× cheaper in RAM** and answers locally. The price is a
> tunable, tiny false-positive rate. Textbook "approximate membership saves the day."

**(b) Content dedup — "is this page near-identical to one I've already stored?"**

The web is full of duplicates: mirrors, syndication, URL variants serving the same article,
boilerplate-heavy templated pages. Exact-match (MD5/SHA of the body) catches **byte-identical**
copies — cheap, do it first. But **near-duplicates** (same article, different ads/timestamps) need
fuzzy matching:

- **SimHash (Charikar):** hash the page's token shingles into a 64-bit fingerprint such that
  *similar documents get fingerprints within a small Hamming distance*. Two pages are near-dupes if
  their SimHashes differ in ≤ `k` bits (e.g. ≤3). This is what Google famously used. Storage: one
  int64 per page; comparison is a fast XOR + popcount.
- **MinHash + LSH:** estimate Jaccard similarity of shingle sets; bucket by LSH so you only compare
  pages likely to be similar. More accurate for set-similarity, heavier than SimHash.

I'd use **SimHash** as the default: compute on parse, store in the metadata table, and on each fetch
query an index of fingerprints (banded by prefix for near-neighbor lookup) to decide whether to store
a fresh copy or just note the duplication. **Why bother?** It saves a meaningful fraction of the 1 PB
store, and — more importantly — it stops the indexer from ranking 50 copies of the same article.

> **Tradeoff:** SimHash can false-positive (mark distinct pages as dupes) and false-negative; tune `k`.
> The cost of a missed dedup is wasted storage; the cost of an over-aggressive dedup is dropping a real
> page. I'd bias conservative (store when unsure) since storage is cheap relative to losing content.

### 6.3 Politeness, robots.txt, and DNS as the real bottleneck

**robots.txt:** before crawling any host, fetch `/robots.txt`, parse allow/disallow + `Crawl-delay`,
and **cache it** (per-host, TTL ~24h, in Redis + local). Honor it absolutely — ignoring robots gets
your IP block-listed and is legally/reputationally radioactive. The crawl-delay feeds directly into
the back-queue min-heap.

**Politeness beyond robots:** even with no crawl-delay specified, default to a polite floor (e.g. one
request every 1–2s per host, or adaptive based on the host's observed latency — slow hosts get
crawled *slower*). Identify with an honest `User-Agent` and provide an opt-out contact.

**DNS as a bottleneck — the under-appreciated killer:** every fetch needs a hostname→IP resolution,
and **DNS lookups are synchronous and slow (often 50–200 ms, sometimes seconds)**. At 23k pages/s
across millions of hosts, naive per-fetch DNS would dominate latency and overwhelm resolvers. Fixes:
- **Aggressive DNS caching** (respect TTLs, but cache hard) in a shared layer — most fetches hit the
  cache because we crawl many pages per host.
- **Pre-resolve** a host's IP when it enters a back queue, not per-URL.
- Run **dedicated recursive resolvers** (don't pound public DNS); some large crawlers run their own.
- Treat DNS resolution itself as an **async, batched, rate-limited subsystem** — it's a mini-frontier
  of its own.

> Say this out loud: "**DNS is the silent SPOF/bottleneck of crawlers.** People design the fetch loop
> and forget that resolving the hostname is often slower than the fetch. I'd cache hard and pre-resolve
> per host." That single observation reads as someone who has actually built one.

### 6.4 Distributed crawling — partition by host, avoid two workers hitting one host

The fleet is distributed; the **invariant we must preserve is politeness across the whole fleet**, not
just within one node. If two workers independently crawl `cnn.com`, we've doubled the rate and broken
the contract.

**Solution: partition the URL space by `hash(host)`**, so **all URLs for a given host always route to
the same shard/worker**. Now per-host rate limiting is a *local* decision on that one worker — no
cross-node coordination needed for the common case. (Consistent hashing so adding/removing workers
only remaps a fraction of hosts — building block in `prep/01`, detail in `prep/04`.)

```
extracted URL ──▶ hash(host) ──▶ shard = consistent_hash(host) ──▶ that shard's frontier
                                                                    (owns ALL of this host)
```

- **Coordination** (`prep/08-consensus-and-coordination.md`): a coordinator (ZooKeeper/etcd) holds the
  **host→shard assignment ring** and the **live worker membership**. Workers watch it; on a worker
  death, its host ranges are reassigned (consistent hashing → minimal reshuffle), and the new owner
  resumes from the durable frontier log.
- **Why host-level, not URL-level, partitioning:** URL-level hashing would scatter one host across
  many workers → you'd need distributed per-host rate limiting (a shared counter, a hot key, constant
  coordination). Host-level partitioning makes politeness a **local invariant** — far cheaper and the
  reason this is the standard choice.
- **Hot-host skew:** a few giant hosts (`wikipedia.org`, `youtube.com`) have billions of URLs and would
  overload their single owning shard's *backlog* (not its fetch rate — that's still politeness-capped).
  Mitigate by letting a hot host's *backlog storage* spill across nodes while keeping a single
  **fetch-coordination owner** (one leaseholder issues the actual requests, honoring the global
  per-host rate). This is the crawler version of the celebrity/hot-shard problem.

### 6.5 Robustness — traps, infinite spaces, bad servers

The open web is adversarial and broken. Defenses, each as a one-liner you'd state:

- **Crawler traps / infinite spaces:** calendars (`?date=2099-01-01` forever), faceted-search URL
  explosions, session-id loops. Defend with: **max depth** per host, **per-host URL budget**, detect
  **high-fan-out low-value** patterns, URL-length caps, and **path-repetition detection**
  (`/a/a/a/...`). Demote or blocklist offending patterns.
- **Soft-infinite content:** dynamically generated pages that never end → budgets + dedup (SimHash
  catches the templated sameness and we stop).
- **Malformed / huge pages:** **cap the download size** (e.g. 10 MB), stream-parse, and treat parse
  failures as errors-with-backoff, not crashes.
- **Slow / dead servers:** **strict per-connection timeouts** (connect + read), exponential backoff on
  errors, and a **per-host failure circuit breaker** (`prep/13-resilience-and-failure-handling.md`) —
  after N failures, stop hitting the host and recheck much later. A slow host must never tie up a
  worker slot indefinitely.
- **Redirect loops & chains:** cap redirect depth; record the `finalUrl` and dedup on it too.
- **Politeness honesty:** respect `429 Too Many Requests` / `Retry-After` by backing off that host.

### 6.6 Freshness — adaptive recrawl scheduling

First discovery is only half the job; the corpus rots. We need to **recrawl**, but recrawling
everything uniformly wastes the budget (a static "About" page doesn't change; a news homepage changes
hourly). So schedule **adaptively based on observed change rate**:

- On each recrawl, compare the new content-hash/SimHash to the stored one → did it change?
- Maintain an **EWMA of change frequency** per URL (or per URL-pattern/site). Pages that keep changing
  get **shorter `next_recrawl_ts`**; pages that never change get exponentially longer intervals.
- This approximates the **Poisson-process optimal-revisit** result: visit frequency ∝ change rate,
  capped by importance (don't waste freshness budget on low-value pages even if they churn).
- Use **conditional GETs** (`If-None-Modified` / `If-None-Match` with ETag): an unchanged page returns
  **304 Not Modified** — near-zero bandwidth, and it still refreshes our liveness signal cheaply.

The **recrawl scheduler** is a separate component that scans the metadata KV for `next_recrawl_ts <=
now` and re-enqueues those URLs into the frontier (with appropriate priority). It competes for the
same politeness budget as new discovery — so freshness vs coverage is a **tunable budget split**, not
an accident.

> **Tradeoff to name:** freshness budget is finite (politeness-capped total fetch rate). Every recrawl
> is a fetch we *didn't* spend discovering new pages. The split between "discover new" and "refresh
> known" is a product/business decision (a search engine weights its index's high-traffic pages
> heavily). State it as a knob.

### 6.7 Storage + feeding the indexing pipeline

- **Raw pages → object store** (S3/GCS), compressed, lifecycle-tiered. Immutable, versioned by
  fetch-time so we keep history for diffing/change-detection. (`prep/14`.)
- **Metadata → KV** (Bigtable/Cassandra/Dynamo), the operational hot path for dedup + scheduling.
- **Link graph → data lake** (Parquet), consumed by the **batch importance job** (PageRank-style) that
  feeds priority scores *back into the frontier prioritizer* — a slower outer loop closing on the fast
  crawl loop.
- **`FetchedDoc` events → Kafka**, consumed by the **indexing pipeline** (analysis → inverted index,
  cross-ref `prep/11-search-systems.md`). The crawler's job ends at "durable raw page + event"; the
  indexer owns tokenization, the inverted index, and ranking. Clean seam, independent scaling.

### 6.8 Extensibility — the plugin seam

Keep the loop generic; make the **parser/processor a strategy chosen by `Content-Type`**:
- `text/html` → HTML parser + link extractor (default).
- `application/pdf`, images, video → their own processors (extract text/metadata, maybe skip).
- **JS-rendered pages** → route to a **headless-browser render farm** (Puppeteer/Chromium) *only* for
  hosts/pages flagged as needing it — it's 10–100× more expensive than a plain GET, so it's a
  separate, smaller, opt-in tier, not the default path.

The frontier, dedup, politeness, and storage layers never change when you add a processor — that's the
extensibility requirement satisfied by **separating transport (fetch) from interpretation (parse)**.

---

## Step 7 — Wrap-Up (3 min)

**Bottlenecks (in order of how much they actually bite):**
1. **Politeness** — the *designed* bottleneck. We can't crawl faster than hosts let us. Fanning across
   millions of hosts is the only way up.
2. **DNS resolution** — the *accidental* bottleneck. Cache hard, pre-resolve per host, run own
   resolvers.
3. **Network ingress** (~18 Gbps) — real but manageable; compress early.
4. **Object-store write throughput + frontier I/O** — sizes the fleet more than fetch concurrency does.

**Failure modes & SPOFs:**
- **Frontier loss** = catastrophic (lose the backlog) → durable, partitioned, replicated log; workers
  checkpoint and resume.
- **Coordinator (ZK/etcd) down** → can't reassign shards; mitigate with a quorum cluster and the fact
  that workers keep crawling their current assignment during a brief outage (graceful degradation).
- **Bloom filter node loss** → rebuild from the authoritative metadata KV (slow but not data-losing);
  shard + replicate so one loss isn't fleet-wide.
- **A single slow/hostile host** → circuit breaker + timeouts isolate it; it can't stall a worker.
- **Poison page** (parser-crashing input) → sandbox the parser, size-cap, treat crashes as fetch
  errors with backoff.

**What I'd do with more time:**
- Geo-distributed crawling (crawl regional content from nearby DCs to cut RTT and respect locality).
- A learned prioritizer (predict change rate + importance) instead of hand-tuned bands.
- Tighter freshness SLAs for a "news" tier with its own dedicated budget.
- Politeness reputation system per host (auto-tune rate from observed 429s/latency).

**Scaling story in one line:** scale **out** by adding shards/workers and rebalancing the
consistent-hash ring by `host`; politeness stays a *local* invariant per shard, so the design scales
near-linearly until you saturate ingress or DNS — both of which we've already hardened.

---

## What Made This Staff-Level

- **Reframed the problem correctly up front:** "this is a distributed, politeness-aware rate limiter,
  bottlenecked on network/DNS, not CPU." Naming the *real* constraint before drawing boxes is the
  single biggest senior tell.
- **The two-level front-queue/back-queue frontier** with a host-ready min-heap — and explaining *why*
  one priority queue can't satisfy prioritization and politeness simultaneously. Made politeness a
  **structural invariant**, not a hoped-for check.
- **Two distinct dedup mechanisms** (Bloom for URL membership with a *numeric* RAM justification;
  SimHash/MinHash for near-duplicate content) — and the tradeoff for each.
- **Host-level partitioning via consistent hashing** so per-host rate limiting is a *local* decision —
  and explaining why URL-level partitioning would force expensive distributed coordination.
- **Called DNS the silent bottleneck** and the frontier the catastrophic SPOF — failure modes named
  before being asked.
- **Adaptive, change-rate-driven recrawl** tied to conditional GETs/304s, framed as a finite-budget
  tradeoff against discovery.
- **Every number drove a decision** (48 GB Bloom vs 2 TB hash set; ~12k connections → tiny fetch tier;
  1 PB → object store + tiering).
- **Clean output seam** to the search/indexing pipeline via Kafka — knowing where *this* system ends.

## Self-Check (answer from memory before the mock)

- [ ] Why can't a single priority queue serve the frontier? What do the front and back queues each
      optimize, and what enforces per-host politeness structurally?
- [ ] Size the URL seen-set as a Bloom filter (items, FP rate → bits/RAM) and justify it over an exact
      hash set.
- [ ] URL dedup vs content dedup — which structure for each, and what's the failure cost of each?
- [ ] Why partition the frontier by `hash(host)` and not by URL? What problem does that *avoid*?
- [ ] Why is DNS a bottleneck, and three ways you'd mitigate it.
- [ ] How does adaptive recrawl decide frequency, and how do 304s + ETags help?
- [ ] Name three crawler traps and the defense for each.
- [ ] Where does the crawler hand off to the indexer, and over what contract?
