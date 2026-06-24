# Topic 14: Object/Blob Storage, Media & Large Files

> **Why this topic matters in the room:** The moment a prompt contains the words *photo, video,
> file, document, attachment, avatar, backup,* or *upload*, half the candidates immediately try to
> stuff bytes into Postgres or stream them through their app servers. Both are wrong, and an
> interviewer will pounce. The staff-level move is to *reflexively* separate **metadata (small,
> structured, queryable → DB)** from **bytes (large, opaque, immutable → object store)**, push the
> bytes path off your servers entirely (presigned URLs + CDN), and then spend your deep-dive budget
> on the *interesting* problems: dedup, the dual-write between DB and store, the async processing
> pipeline, and garbage collection. This doc gives you the reflexes and the math.

---

## Part A — The one pattern that governs everything

**Metadata in the DB, bytes in the object store, joined by a key.** Internalize this picture:

```
                          ┌──────────────┐
   client ──upload──────► │  app server  │  writes a row, returns a presigned URL
                          └──────┬───────┘
                                 │  (1) INSERT metadata row (id, owner, key, size, status=PENDING)
                                 ▼
                          ┌──────────────┐
                          │  metadata DB │  id → { bucket, key, size, content-type, owner, status }
                          └──────────────┘
                                 │  (2) hand back presigned PUT URL
   client ──PUT bytes─────────────────────────────────► ┌──────────────┐
                                                          │ object store │  bucket/key → bytes
                                                          └──────┬───────┘
                                                                 │ (3) event/callback flips status=READY
   client ──GET────► CDN ──miss──► object store (signed URL) ────┘
```

The DB holds the *small, hot, relational* part you query and join on (who owns it, when, tags,
permissions, processing status). The object store holds the *large, cold, opaque* part. The link
between them is a single string: the **object key**.

### Why blobs don't belong in a relational DB

Say this list cold:

| Problem if you store bytes in the DB | Consequence |
|---|---|
| **Row/page bloat** | A 4 MB image in a row destroys the buffer-pool cache hit rate; queries that never touch the blob get slower. |
| **Backups & replication explode** | Your `pg_dump` / replication stream now carries terabytes of immutable bytes that never change. RPO/RTO blow up. |
| **No CDN edge delivery** | You can't put a CDN in front of a SQL `SELECT`. Every read hits your origin DB. |
| **Connection-bound, not bandwidth-optimized** | DBs are tuned for many small transactions, not for streaming MB-scale payloads over a held connection. |
| **Cost** | DB storage (provisioned IOPS SSD, replicated 3×) is 10–50× the $/GB of object storage. |
| **Memory pressure** | Reading a blob pulls it through the DB process memory and your app heap. OOMs under load. |

> **Say this in the room:** "Postgres `BYTEA` / MySQL `BLOB` is fine for a few KB — a thumbnail, an
> icon, a signature. The line is roughly **single-digit KB**. Past that, the bytes go to object
> storage and the row holds only the key plus metadata." Naming the *threshold* signals you've
> actually operated this, not just read about it.

The mirror-image anti-pattern is just as bad: **don't put queryable metadata only in the object
store.** S3 has no "list all photos owned by user 42 taken in 2025 tagged #beach" query. That's a
DB index. Object stores list by key prefix and nothing else.

---

## Part B — Object storage internals (S3-style)

### Buckets, keys, and the flat namespace

- A **bucket** is a top-level container (a namespace + a policy/billing boundary). Bucket names are
  globally unique in S3.
- An **object** is `key → (bytes + metadata + version)`. The key is an opaque string up to ~1 KB.
- **The namespace is flat.** `photos/2025/06/cat.jpg` is *one key string*, not a directory tree.
  The `/` is a convention; "list with prefix `photos/2025/`" simulates folders. There are no real
  directories, no atomic rename of a "folder", no `mv` — a rename is copy + delete.
- Objects are **immutable**: you replace a whole object, you don't patch a byte range in place.
  (Some stores offer append; treat object PUT as whole-object replace for design purposes.)

> Implication you should state: because keys are a flat string space, **key design = your read
> pattern.** If you'll list "all of a user's files," prefix the key with the user id
> (`user/42/...`). Historically a *random* high-entropy prefix spread load across partitions to
> avoid hot spots (S3 now auto-scales prefixes, but the principle — spread writes across the
> keyspace — still matters for other stores).

### Consistency model

| Era / store | Model |
|---|---|
| Old S3 | Read-after-write for new objects, **eventual** for overwrites and LIST. The classic "I PUT then GET and got a 404 / stale version" gotcha. |
| Modern S3 (since Dec 2020) | **Strong read-after-write consistency** for PUTs, overwrites, and LIST, within a region. |
| GCS / Azure Blob | Strongly consistent for object reads after write. |

Why interviewers still care: **the metadata DB and the object store are two systems, so you have a
dual-write even if each is internally strongly consistent** (see Part H). And cross-region
replication is *always* eventual. Don't claim "S3 is strongly consistent so I'm fine" — the
consistency gap that bites you is *between* the DB and the store, and *across* regions.

### Durability: erasure coding vs replication (know the math)

Object stores advertise **eleven nines** of durability (99.999999999%) — expected to lose ~1 object
per 10 million per ~10,000 years. Two ways to get there:

**1. N-way replication.** Keep N full copies on independent failure domains.
- Survives N−1 failures. Simple, fast to read (any copy), fast to rebuild.
- **Storage overhead = N×.** 3-way replication (HDFS default, Cassandra-style) = **200% overhead**
  (store 3 GB to keep 1 GB). Read amplification on write = 3×.

**2. Erasure coding (Reed–Solomon `k` data + `m` parity).** Split object into `k` data shards,
compute `m` parity shards, scatter all `k+m` across failure domains.
- Survives loss of any `m` shards. Reconstruct from any `k`.
- **Storage overhead = (k+m)/k.**

| Scheme | Data + parity | Tolerates | Storage overhead | Usable / raw |
|---|---|---|---|---|
| 3× replication | 1 + 2 copies | 2 losses | 3.0× (**200%**) | 33% |
| RS(6,3) | 6 + 3 | 3 losses | 1.5× (**50%**) | 67% |
| RS(10,4) | 10 + 4 | 4 losses | 1.4× (**40%**) | 71% |
| RS(12,4) (typical cold) | 12 + 4 | 4 losses | 1.33× (**33%**) | 75% |

> **The tradeoff to say out loud:** "Erasure coding gives the same or better durability at ~1.4×
> instead of 3× — that's why hot-but-large object stores use it; for petabytes the storage saving
> is enormous. The cost is **higher reconstruction CPU and read latency, and a 'small-write'
> penalty** — you can't cheaply update a single shard, and you need `k` shards online to read.
> So I'd erasure-code large, cold, immutable objects, and use replication for small, hot, or
> latency-critical data and for the metadata DB." That sentence is the staff signal.

### Storage classes / tiering

Bytes have a temperature. Match the class to the access pattern.

| Class (S3 names) | Use | Retrieval | Relative $/GB-month |
|---|---|---|---|
| Standard | Hot, frequently read | instant | 1.0× (baseline) |
| Standard-IA / One-Zone-IA | Infrequent but needs to be instant | instant + per-GB retrieval fee | ~0.55× / ~0.43× |
| Glacier Instant Retrieval | Archive, rare reads, instant | instant + retrieval fee | ~0.2× |
| Glacier Flexible / Deep Archive | Cold archive, compliance | **minutes to hours** to restore | ~0.1× / ~0.04× |
| Intelligent-Tiering | Unknown/changing pattern | auto-moves between tiers | baseline + monitoring fee |

Mention **lifecycle policies**: "transition objects to IA after 30 days, Glacier after 90, delete
after 7 years" — declarative rules, not application code. For an interview, knowing that *cold
archive trades retrieval latency for ~10–25× cheaper storage* is enough.

---

## Part C — Upload & download at scale

**The cardinal rule: never proxy large bytes through your app servers.** If a 1 GB upload streams
*through* your stateless app tier, you've coupled bandwidth, memory, and request timeout to file
size; one big upload pins a worker, and N concurrent uploads OOM the box. The bytes must go
**client → object store directly.** Your servers only touch *metadata* and *authorization*.

### Presigned URLs (the workhorse)

A **presigned URL** is a time-limited, signed URL that grants a *specific operation* (`PUT` to this
key, or `GET` of this key) without the client holding any credentials.

```
1. client → app: "I want to upload avatar.jpg, 2 MB, image/jpeg"
2. app: authz check → INSERT metadata(status=PENDING) → S3.generate_presigned_url(PUT, key, expires=15min)
3. app → client: { uploadUrl, key }
4. client → S3: PUT uploadUrl  (bytes go straight to S3, never touching your servers)
5. (S3 event or client callback) → app: flip status=READY, record final size/etag
```

Benefits to state: **bytes offloaded from your fleet**, you keep authz + metadata control, and the
URL is scoped (one key, one verb, short TTL). For uploads you can even pin `Content-Length`,
`Content-Type`, and content-MD5 into the signature so the client can't lie about what it uploads.

The same pattern for **download**: hand out a presigned `GET` (or a CDN signed URL, Part D) so the
read also bypasses your origin.

### Multipart / chunked upload + resumable uploads

For large files you do **not** want one giant PUT. Split into parts (S3: 5 MB–5 GB each, up to
10,000 parts → 5 TB max object).

```
InitiateMultipartUpload → uploadId
  for each part i:  UploadPart(uploadId, partNumber=i, bytes) → ETag_i   (parallel!)
CompleteMultipartUpload(uploadId, [ {i, ETag_i} ... ])  → object assembled server-side
  (or AbortMultipartUpload to discard)
```

Why this wins:

| Benefit | Mechanism |
|---|---|
| **Parallelism** | Upload parts concurrently → saturate the client's uplink, faster wall-clock. |
| **Resumability** | A failed part is retried alone; you don't re-send 4.9 GB because byte 4.91 GB failed. |
| **Throughput on long-haul** | Many TCP streams beat one over high-latency links. |
| **Flaky networks** | Mobile uploads survive connectivity blips. |

**Resumable upload** = the client remembers which parts succeeded (by part number / ETag) and only
re-uploads the missing ones. GCS exposes a resumable session URI; S3 uses the `uploadId` + listing
already-uploaded parts. Mention a **lifecycle rule to abort incomplete multipart uploads after N
days** — otherwise orphaned parts silently accumulate and you pay for them forever.

### Direct-to-storage, end to end

Combine the two: the client gets **presigned URLs per part**, uploads parts directly to the store
in parallel, and the only thing hitting your servers is "init" and "complete." Your app tier
becomes a thin control plane over a fat, dumb byte pipe. That is the whole game.

---

## Part D — CDN integration for read-heavy media

Media is overwhelmingly **read-heavy** (upload once, serve millions of times). The object store is
your origin; the CDN is the cache that sits in front of it at the edge.

```
client ──► CDN edge (PoP near user) ──hit──► serve from edge cache
                     │
                     └──miss──► origin (object store / origin shield) ──► fill edge, serve
```

- **Why:** latency (edge is geographically near the user), **origin offload** (you serve the long
  tail from the store but the hot set from cache), and bandwidth cost (CDN egress is cheaper and
  absorbs spikes). For a viral video the CDN is the only thing standing between you and an origin
  meltdown.
- **Cache key & immutability:** treat media as immutable and **version it in the key/URL**
  (`/img/abc123_v2.jpg` or content-hash names). Then you can set `Cache-Control: max-age=31536000,
  immutable` and never invalidate — a new version is a new URL. This sidesteps the hardest part of
  CDN caching.
- **Origin shield / tiered caching:** a mid-tier cache so that a cold edge miss doesn't stampede
  your origin; collapses many concurrent misses for the same object into one origin fetch.

### Signed URLs for access control

Public media → plain CDN URL. Private media (your photos, a paid video) → **CDN signed URL / signed
cookie**: a token in the URL signed by your key, with an expiry and optional IP/path scope. The
edge validates the signature *without calling your origin*, so access control stays off your servers.

| | Presigned (object-store) URL | CDN signed URL/cookie |
|---|---|---|
| Signed by | The object store's credentials | Your CDN key pair |
| Served from | Origin store | Edge cache (fast, offloaded) |
| Use for | Uploads, internal byte transfer | Public-facing reads of private media |

> Common follow-up: "How do you revoke access to media you already handed out a signed URL for?"
> Answer: signed URLs are bearer tokens — you can't un-issue one. Mitigate with **short TTLs**
> (minutes–hours), per-user **key rotation**, and for true revocation, route through a path your
> origin can deny and **purge the edge cache**. Be honest that there's an inherent
> share-the-link leak window; you trade it for not hitting origin on every read.

### Cache invalidation

The two hard problems, again. Strategies, best to worst:

1. **Immutable + versioned URLs (preferred).** Never invalidate; publish a new key. The hot set
   self-manages by TTL.
2. **TTL expiry.** Set `max-age`; stale content self-heals after the window. Cheap, eventual.
3. **Explicit purge / invalidation API.** Forcible, but slow to propagate across all PoPs (seconds
   to minutes), rate-limited, and sometimes billed. Use for "take this down *now*" (legal/abuse),
   not routine updates.

---

## Part E — File-sync (Dropbox-style) deep dive

This is the canonical "large files + dedup + sync" prompt. The defining ideas:

### Chunk the file; address chunks by content

- Split each file into **chunks** (fixed ~4 MB, or content-defined chunking via a rolling hash so
  an insert near the start doesn't shift every downstream boundary).
- **Content-addressed storage (CAS):** the chunk's key *is the hash of its bytes*
  (`sha256(chunk) → chunkId`). Identical bytes ⇒ identical key ⇒ stored once.

This gives you **dedup for free**, at two levels:

| Dedup level | Benefit | Note |
|---|---|---|
| **Cross-file / global** | The same attachment emailed to 1,000 users is stored once. | Massive savings for shared/popular content. |
| **Within-user / cross-version** | Edit a 1 GB file's last page → only changed chunks are new. | This is **delta sync**. |

### The file-vs-block split (two services)

| Service | Stores | Why separate |
|---|---|---|
| **Block / chunk store** | `chunkHash → bytes` in object storage (CAS) | Big, immutable, dedup-able, CDN-able. |
| **Metadata service** | `file → ordered list of chunkHashes`, versions, namespace tree, device cursors | Small, relational, transactional, *strongly consistent*. The source of truth for "what a file is." |

A file is just an **ordered list of chunk hashes** in the metadata DB. To reconstruct: look up the
hash list, fetch each chunk (most from local cache / CDN), concatenate. To sync a change, the client
hashes its local chunks, asks the server **"which of these hashes do you already have?"**, and only
uploads the missing ones (often *zero* — pure metadata update).

### Delta sync & the sync protocol

```
client edits file → re-chunk → compute hashes
client → server:  "has(chunkHashes[])?"        # cheap existence check
server → client:  missing = [h3, h7]
client → store:   PUT only h3, h7 (presigned, direct)
client → server:  commit new version = [h1,h2,h3',h4,...,h7',...]   # metadata write
server → other devices: push notification "file X changed to version v+1"
other devices: pull metadata diff, fetch only the chunks they lack
```

Use a **long-poll / push channel** (the "notification service") so idle clients learn of changes
fast, plus a **cursor/version per device** so a reconnecting client syncs only the delta since its
last cursor — not a full re-scan.

### Conflict handling

Concurrent edits to the same file from two offline devices:

- **Detect** via version vectors / a per-file monotonically increasing version. If a client commits
  against a stale base version, it's a conflict.
- **Resolve** — Dropbox's pragmatic answer: **don't lose data; create a conflicted copy**
  (`report (Alice's conflicted copy).docx`) and let the human merge. For text/structured docs you
  can do operational transforms or CRDTs (collaborative-editing territory, see the consistency
  topic), but for opaque binary files, *fork-and-let-the-user-decide* is the correct, honest answer.

> **Say this:** "For arbitrary binary files I can't auto-merge, so I optimize for **never silently
> losing a write** — last-writer-wins would destroy data. I keep both versions and surface the
> conflict. The metadata DB is the linearization point that lets me *detect* the conflict at commit
> time."

---

## Part F — Image/video upload + processing pipeline

You almost never serve the original bytes the user uploaded. You serve **derivatives**: resized
images, transcoded video renditions. That processing is **asynchronous** — never block the upload
response on it.

### The async pipeline

```
upload (direct-to-store) ──► S3 PUT event / enqueue job
                                    │
                              ┌─────▼─────┐
                              │  queue    │  (SQS/Kafka) — absorbs spikes, decouples
                              └─────┬─────┘
                                    ▼
                          ┌───────────────────┐
                          │ transcode workers │  (autoscaled pool, GPU for video)
                          └─────────┬─────────┘
        derivatives ──► object store (thumbs, sizes, HLS segments)
        metadata    ──► DB: status PENDING → PROCESSING → READY (or FAILED)
```

State the properties: **queue absorbs upload spikes** so a flash of uploads doesn't melt the
encoder fleet; **workers autoscale on queue depth**; **idempotent jobs keyed by content hash** so
at-least-once delivery / retries don't double-encode; a **dead-letter queue** for poison inputs
(corrupt files); and the client sees `status=PROCESSING` and gets a push when it flips to `READY`.

### Images

Generate a fixed set of **derivatives** on upload: thumbnail, small, medium, large, plus
format variants (WebP/AVIF for modern clients, JPEG fallback). Strip EXIF, re-encode to defeat
malicious payloads. Name derivatives deterministically off the original key
(`<id>/thumb.webp`, `<id>/1080.jpg`) so they're cacheable and regenerable. Consider **on-the-fly
resize** behind the CDN for the long tail of rarely used sizes (compute once on first request,
cache at edge) vs. pre-generating the common ones.

### Video: adaptive bitrate streaming

You don't serve one MP4. You transcode each upload into a **ladder** of resolutions/bitrates,
segment each into a few-second chunks, and let the *player* pick a rung based on live bandwidth.

| Concept | What it is |
|---|---|
| **Transcoding ladder** | Encode to 240p, 480p, 720p, 1080p, 4K at increasing bitrates. |
| **Segmenting** | Cut each rendition into ~2–10 s segments (`.ts`/fMP4). |
| **Manifest / playlist** | HLS `.m3u8` or DASH `.mpd` lists segments + the available bitrate variants. |
| **ABR** | Player monitors throughput/buffer and switches rungs *per segment* — smooth quality, no rebuffer. |
| **HLS vs DASH** | HLS = Apple-origin, ubiquitous on iOS/Safari; DASH = codec-agnostic ISO standard. Often package both, or use CMAF fMP4 to share segments. |

Segments are immutable files in the object store, fronted by the CDN — perfect cacheability. The
manifest is tiny and also cached. For video this is *the* answer to "how do you serve at scale":
**chunked, immutable, CDN-fronted, adaptive.** Parallelize transcoding by **fanning each segment
out as its own job** so a 2-hour movie isn't one serial encode.

---

## Part G — The serving math (do this out loud)

A back-of-envelope for "design a photo/video sharing service." Make up defensible numbers and
*derive* the architecture.

**Assume:** 500 M users, 100 M DAU. Each DAU uploads **2 photos/day**, average **1.5 MB** after
re-encode (you also keep an original, call it ~4 MB → ~5.5 MB stored per photo across derivatives).

**Write / storage growth**
- Uploads/day = 100 M × 2 = **200 M photos/day** ≈ 200M / 86,400 ≈ **~2,300 writes/sec avg**,
  peak ~3× ≈ **~7,000/sec**. → justifies queue + horizontally sharded metadata + direct-to-store.
- Bytes/day = 200 M × 5.5 MB ≈ **1.1 PB/day** ≈ **~400 PB/year**. → justifies erasure coding
  (1.4× not 3× → saves ~160 PB/yr of raw) and lifecycle tiering of old photos to cold storage.

**Read / bandwidth (the dominant axis — media is read-heavy)**
- Assume **100:1 read:write** → ~200 M × 100 = **20 B reads/day** ≈ **~230k reads/sec** avg.
- If average served object (thumb/medium) is ~300 KB: egress = 20 B × 300 KB ≈ **6 PB/day** of
  reads. **No origin survives that** → CDN is mandatory, not optional.

**CDN offload ratio**
- If CDN hit ratio = **95%**, origin sees only 5% → ~300 TB/day to origin instead of 6 PB.
  That **20× reduction** is the entire reason the CDN exists; quantify it.
- Origin offload = `1 − (origin egress / total egress)`. Push the hot set's hit ratio toward 99%
  with long TTLs + immutable URLs and you cut origin traffic another 5×.

**Metadata DB**
- Photo rows are ~few hundred bytes. 200 M/day × 300 B ≈ 60 GB/day of *metadata* — trivial for the
  store, but **400+ PB/yr of bytes** is why metadata and bytes live in different systems with
  wildly different cost/scaling profiles. That contrast *is* the answer to "why the split."

> The estimation isn't decoration — each number *forces* a decision: 7k write QPS → queue +
> sharding; 400 PB/yr → erasure coding + tiering; 6 PB/day reads → CDN; 20× offload → quantified
> CDN value. Walk the chain and the design derives itself.

---

## Part H — Consistency between DB and object store (the dual-write)

You write to **two** systems per upload — the metadata DB and the object store — with no shared
transaction. Same dual-write demon as the cache and the search index. Failure modes:

| What fails | Result | Fix |
|---|---|---|
| DB row written, bytes upload never completes | **Dangling metadata** pointing at a missing object | `status=PENDING` until upload confirmed; reader treats PENDING as "not there." |
| Bytes uploaded, DB write fails | **Orphaned blob** burning storage, invisible to the app | GC sweep (Part I). |
| Bytes deleted, DB still references | Read → 404 | Soft-delete + delete bytes only after metadata commit. |

### The ordering rule

**Always create the metadata row *first* (PENDING), upload bytes second, then flip to READY.**
Reverse it and a crash leaves you with bytes nobody knows about. Confirm the upload via an
**object-store event** (S3 → SQS/Lambda) or a client `complete` call that the server *verifies*
(HEAD the object, check size/etag) before flipping to READY — never trust the client's word that the
upload finished.

> **The pattern to name:** this is **claim-check** + a small state machine
> (`PENDING → READY → DELETED`). The DB is the source of truth for *whether an object logically
> exists*; the store is the source of truth for *the bytes*. Reconcile the two with a background
> job, never assume they're in sync in real time.

### Reconciliation

A periodic job that walks both sides:
- **Orphan detection:** list store keys with no READY metadata row older than a grace period → GC.
- **Dangling detection:** metadata rows stuck in PENDING past a TTL → mark FAILED, prompt re-upload.
- Drive it off an event stream where possible (process S3 inventory reports / event log) rather than
  full O(N) scans at petabyte scale.

---

## Part I — Garbage collection of orphaned blobs

In a content-addressed/dedup store you **cannot** delete a chunk just because *one* file referenced
it — others may share it. This is the GC problem.

| Approach | How | Tradeoff |
|---|---|---|
| **Reference counting** | Each chunk has a count; `+1` on link, `−1` on unlink; delete at 0. | Exact and immediate, but counts must be transactional and survive crashes; miscount = leak or premature delete. Hard with concurrency. |
| **Mark-and-sweep** | Periodically walk all live file→chunk references (the roots), mark reachable chunks, sweep the unmarked. | Self-healing (recovers from missed decrements), but a heavy batch job at scale; must handle objects created *during* the sweep (grace period / generation stamps). |
| **Soft delete + TTL** | Deletes flip a tombstone/`status=DELETED`; a delayed sweep reclaims after a retention window. | Enables undo/restore and trash; pay storage during the window. Almost always wanted by the product anyway. |

> **Say this:** "I'd never hard-delete on the user's delete click. **Soft-delete** (tombstone +
> retention window) gives undo, protects against bugs, and decouples the user action from byte
> reclamation. Reclamation runs **async via ref-counting for the common path, backstopped by a
> periodic mark-and-sweep** to catch leaked decrements. The store's own lifecycle rule then does
> the final physical delete." Pairing the fast path (ref-count) with the safety net (sweep) is the
> staff-level answer — pure ref-counting drifts, pure sweeping is too slow alone.

Also: **abort incomplete multipart uploads** and **expire PENDING rows** as part of GC — these are
the two leak sources candidates forget.

---

## Part J — Worked mini-walkthrough: "Design Dropbox"

A compressed run using the Topic-1 framework, to show the shape of a strong answer.

**1. Requirements.** Upload/download files, sync across a user's devices, share, version history.
NFRs: durability is paramount (**never lose a byte**), strong consistency on the *metadata* (file
tree, versions), eventual is fine for cross-device propagation latency (seconds). Large files (GBs),
flaky/mobile networks, read-heavy on shared content.

**2. Estimation.** 50 M users, 10 M DAU, avg 100 GB stored each → 5 EB logical *before* dedup;
dedup + delta sync cut effective storage 2–3×. Most "uploads" are metadata-only (chunks already
exist) — *that's the insight dedup buys you.*

**3. API.**
```
POST /files            { path, size, chunkHashes[] }  -> fileId, version
POST /chunks/missing   { hashes[] }                   -> missing[]     (existence check)
PUT  <presigned>       (bytes per missing chunk, direct to store)
POST /files/{id}/commit{ version, chunkHashes[] }     -> newVersion
GET  /delta?cursor=                                   -> changes since cursor
```

**4. Data model.** Metadata DB (sharded by userId): `files(id, owner, path, currentVersion)`,
`versions(fileId, version, [chunkHashes], createdAt)`, `chunks(hash, size, refcount)`. Block store
(object storage, CAS): `chunkHash → bytes`, erasure-coded.

**5. High-level.** Client chunker + local cache → metadata service (strongly consistent, sharded) →
block store (presigned direct upload) → notification service (push deltas) → CDN for shared
downloads.

**6. Deep dives — propose the hard parts:**
- **Dedup + delta sync:** content-addressed chunks; client uploads only missing hashes; edits cost
  only changed chunks. (Part E.)
- **Sync protocol & conflicts:** per-device cursor, push channel, version-vector conflict detection,
  conflicted-copy resolution — never silently lose data. (Part E.)
- **Dual-write / consistency:** PENDING→READY state machine; verify upload before commit;
  reconciliation job. (Part H.)
- **GC:** ref-count chunks (can't delete a shared chunk on one unlink) + mark-and-sweep backstop +
  soft delete. (Part I.)

**7. Wrap-up.** SPOFs: metadata DB → shard + replicate, it's the source of truth. Notification
service down → clients fall back to periodic poll (degraded, not broken). Block store is the
durability anchor (11 nines, erasure-coded). With more time: encryption at rest + per-file keys,
LAN sync between same-network devices, bandwidth throttling, and abuse/CSAM scanning in the pipeline.

---

## The tradeoffs you must recite cold

- **Metadata in DB vs bytes in object store** — query/transactions/joins vs cheap durable bulk
  storage + CDN delivery. The split is non-negotiable above a few KB.
- **Replication vs erasure coding** — 3× overhead, simple, fast rebuild/low-latency vs ~1.4×
  overhead, CPU + reconstruction cost + small-write penalty. Replicate hot/small, erasure-code
  large/cold.
- **Presigned/direct upload vs proxying through app servers** — offloads bytes + bandwidth vs you
  never want app servers in the byte path for large files.
- **Single PUT vs multipart/resumable** — simple vs parallel + resumable + spike-tolerant for large
  files.
- **Immutable versioned URLs vs explicit cache invalidation** — never-invalidate (publish new key)
  vs slow, rate-limited purges; prefer the former.
- **Ref-counting vs mark-and-sweep GC** — exact/immediate but drifts under concurrency vs
  self-healing but heavy; use both.
- **Pre-generate derivatives vs on-the-fly + cache** — pay compute upfront for the hot set vs lazy
  compute for the long tail.
- **HLS vs DASH (vs CMAF)** — ecosystem reach vs standard/codec-agnostic vs shared segments.

> **Mental checklist when an unseen "files/media" problem appears:**
> Where do the *bytes* live (object store, never the DB or app tier)? → Where does the *metadata*
> live (DB, source of truth)? → How do bytes get *in* without touching my servers (presigned +
> multipart)? → How do bytes get *out* at scale (CDN + signed URLs)? → What's the *dual-write*
> between DB and store, and how do I reconcile? → Is there *dedup* (content-addressing)? →
> Is there *async processing* (queue + workers)? → How do I *GC* orphans?

---

### Self-check before the mock (answer these from memory)
- [ ] State the metadata-in-DB / bytes-in-object-store split and the ~KB threshold for inlining.
- [ ] Give the storage-overhead math for 3× replication vs RS(6,3) vs RS(10,4).
- [ ] When do you erasure-code vs replicate? Name the small-write penalty.
- [ ] Walk the presigned-URL upload flow and the PENDING→READY state machine.
- [ ] Why multipart/resumable upload, and what leak does it create (and the fix)?
- [ ] Presigned URL vs CDN signed URL — who signs, where served, what for? How do you "revoke"?
- [ ] Explain content-addressed storage and the two levels of dedup it buys.
- [ ] Describe delta sync end-to-end and how you detect + resolve a conflict on a binary file.
- [ ] Sketch the async transcoding pipeline and what ABR/HLS/DASH solve for video.
- [ ] Do the photo-service math: write QPS, PB/year, reads/sec, and CDN offload ratio.
- [ ] Name the three GC strategies and why you combine ref-counting with mark-and-sweep.
- [ ] What are the dual-write failure modes between DB and store, and how does reconciliation fix them?
