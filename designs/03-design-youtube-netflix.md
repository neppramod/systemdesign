# Design 03: YouTube / Netflix — Video Upload & Streaming at Scale

> **Why this problem is a rite of passage:** it is the canonical *media + read-heavy + bytes-not-
> rows* problem, and it forces every reflex from Topic 14 (Blob Storage). The trap is to spend your
> 45 minutes on the metadata service and CRUD, because that's the part you know how to build. The
> staff-level signal is to recognize within the first two minutes that **bandwidth (CDN egress) is
> the binding constraint, not compute and not storage**, and to architect the *entire* system around
> pushing bytes off your servers — direct-to-blob on the write side, CDN-everything on the read side.
> This walkthrough runs the 7-step framework from `prep/01-framework-and-building-blocks.md` end to
> end, and cross-references the deep-dive docs where each block lives.

---

## Step 1 — Requirements (5 min)

I'll drive this and narrow scope explicitly. "YouTube/Netflix" is huge; let me carve it.

**Functional (the verbs I'll build):**
- A **creator uploads** a video (large file, GB-scale) and gives it a title/description/tags.
- The system **transcodes** that video into multiple resolutions/codecs and generates thumbnails — asynchronously.
- A **viewer streams** a video: smooth playback, adaptive to their bandwidth, low startup latency.
- A viewer **searches/browses** for videos and gets **recommendations**.
- The system **counts views** and accepts **likes**.

**Explicitly out of scope** (say it, protect your time): comments, monetization/ads, DRM internals, channel/subscription management, the recommendation ML model itself (I'll treat it as a black box). I'll cover **VOD as the main case** and contrast **live streaming** at the end as a deep dive.

**Non-functional (where the design is actually decided):**

| Dimension | Target | Consequence it forces |
|---|---|---|
| **Scale** | YouTube-class: ~2B users, ~100M DAU; 500 hrs uploaded/min | Exabyte storage, Tbps egress |
| **Read:write ratio** | Extremely read-heavy. Uploads are rare; **views dwarf uploads ~1000:1+** | The whole design optimizes reads → CDN |
| **Latency** | Video **startup < 1–2s**; rebuffering near zero; metadata/page load < 200ms | ABR + CDN edge + small first segment |
| **Availability** | Playback 99.99% (a viewer's outage is a churn event); upload 99.9% (rarer, retryable) | Read path must be the most resilient |
| **Consistency** | Upload→playable: **eventual** (transcoding takes minutes — that's fine). View counts/likes: **eventual** (approximate, lazily reconciled). Account/billing: strong (out of scope) | Lets me use queues, caches, sketches freely |
| **Durability** | Originals must never be lost (can't ask a creator to re-upload a finished film) | 11-nines object store, erasure coding |

> **The sentence that frames everything:** "This is the most read-heavy system in the canon. Uploads
> are a trickle; views are a firehose. So I'll spend my write-side budget on getting bytes off my
> servers cheaply, and my entire read-side budget on the **CDN**, because **egress bandwidth is the
> dominant cost and the binding constraint** — and I'll prove that with the estimate."

---

## Step 2 — Estimation (3 min)

Numbers that *justify decisions*, not vanity. (Methodology: `prep/02-estimation-and-napkin-math.md` Parts D & E.)

**Upload / storage growth**

| Quantity | Math | Result |
|---|---|---|
| Hours uploaded/day | 500 hrs/min × 60 × 24 | **720,000 hrs/day** |
| Upload write QPS | trivially low — ~500 videos/min ≈ **~10/s** | Writes are *not* the problem |
| Raw bytes/day (source) | 720k hrs × ~3 GB/hr | ~2.2 PB/day source |
| Stored bytes/day (×~3 for the rendition ladder) | 720k × 3 GB × 3 | **~6.5 PB/day** |
| **Per year** | 6.5 PB × 365 | **~2.3 EB/year** |

→ Exabyte scale ⇒ erasure coding (1.4× overhead, not 3× replication — saves ~40%), plus **hot/cold tiering** (most of that 2.3 EB is long-tail that's almost never watched).

**Read / bandwidth — the dominant axis**

| Quantity | Math | Result |
|---|---|---|
| Concurrent viewers (peak) | assume ~10M concurrent streams | 10M |
| Bitrate per 1080p stream | — | ~5 Mbps |
| **Total egress** | 10M × 5 Mbps | **~50 Tbps** |
| In NICs | 50 Tbps ÷ 10 Gbps/NIC | **~5,000 NICs' worth** |

> **This single number is the whole interview.** 50 Tbps cannot come from any origin you can build.
> One 10 Gbps NIC serves 5,000 NICs short. **This is the CDN/edge argument, full stop** — and the
> estimate *proved* it rather than asserting it (`prep/02` Part E).

**CDN offload (why the origin survives at all)**

- If CDN hit ratio = **95%**, origin serves only 5% → ~2.5 Tbps at origin instead of 50 Tbps. A **20× reduction**.
- Push the hot set toward **99%+** with long TTLs on immutable segments → origin drops to ~0.5 Tbps. Each nine of offload is worth its weight in datacenter.
- Offload ratio = `1 − (origin egress / total egress)`. **Quantify it — it's the core read-side metric.**

**Metadata DB** — video rows are ~1 KB (title, description, owner, tags, status, key prefix). Even 720k new videos/day × 1 KB ≈ **<1 GB/day** of metadata. Trivial. *That contrast — <1 GB/day of metadata vs 6.5 PB/day of bytes — is itself the answer to "why split metadata from bytes."*

---

## Step 3 — API Design (3 min)

Three surfaces: **upload (write)**, **playback (read)**, **engagement**. Note that *no video bytes ever flow through these JSON APIs* — they only exchange metadata and **presigned URLs**.

```
# ---- Upload: initiate, then client uploads bytes DIRECTLY to blob store ----
POST /videos
  body: { title, description, tags[], fileSize, contentType }
  -> { videoId, uploadId, parts:[{partNumber, presignedPutUrl, byteRange}], status:"PENDING" }

# client PUTs each chunk directly to the object store (NOT to us), then:
POST /videos/{videoId}/complete
  body: { uploadId, parts:[{partNumber, etag}] }
  -> { status:"PROCESSING" }     # we verify the object, then enqueue transcode

# ---- Playback ----
GET /videos/{videoId}            -> { title, owner, durationMs, status, manifestUrl, thumbnails[] }
GET /videos/{videoId}/manifest   -> 302 redirect to CDN-hosted HLS/DASH manifest (or the .m3u8 itself)
   # segments themselves are plain CDN URLs listed inside the manifest

# ---- Browse / engage ----
GET  /search?q=&cursor=          -> { results:[...], nextCursor }   # cursor-based, not offset
GET  /feed?userId=&cursor=       -> { recommendations:[...] }
POST /videos/{videoId}/view      -> 202 Accepted   # fire-and-forget, batched downstream
POST /videos/{videoId}/like      -> 202 Accepted
```

- **Cursor-based pagination** for search/feed (offset breaks under inserts).
- Auth + rate-limiting at the **API gateway** (`prep/09`) so I don't re-explain it per endpoint. Presigned URLs are themselves scoped + time-limited credentials.
- `/view` returns **202**, not 200 — it's accepted into a pipeline, not transactionally committed. That single status code signals "I know view counts are eventual."

---

## Step 4 — Data Model (5 min)

The governing pattern (`prep/14` Part A): **metadata (small, structured, queryable) → DB; bytes (large, opaque, immutable) → object store; joined by a key.**

**Object store (S3-style)** — the bytes. Keyed by a flat namespace:
```
videos/{videoId}/source.mp4                 # original — keep forever, cold tier after processing
videos/{videoId}/hls/1080p/seg_00001.ts     # transcoded segments (immutable)
videos/{videoId}/hls/720p/seg_00001.ts
videos/{videoId}/hls/master.m3u8            # the manifest (tiny, points at the variants)
videos/{videoId}/thumbs/default.webp
```
Immutable + content-addressable naming ⇒ infinitely cacheable at the CDN with long TTLs.

**Metadata DB** — `videos` table (the queryable part):

| Field | Notes |
|---|---|
| `videoId` (PK) | Snowflake-style sortable ID |
| `ownerId`, `title`, `description`, `tags[]` | searchable fields (fed to search index via CDC) |
| `status` | `PENDING → PROCESSING → READY` (or `FAILED`) — the state machine |
| `durationMs`, `manifestKey`, `thumbnailKeys[]` | filled by the transcode pipeline |
| `createdAt`, `tier` | for tiering decisions |

- **Access pattern → store choice** (`prep/01` step 4, `prep/03` databases): point-read by `videoId` (the dominant read), plus "list by owner." That's a key-value/wide-column access pattern → **shard by `videoId`** (hash) on a horizontally scalable store (Cassandra/DynamoDB-style, or sharded Postgres). No cross-entity transactions on the hot path ⇒ NoSQL is defensible. `prep/04` covers the sharding.
- **View counts / likes** live in a *separate* high-write store, **not** as a column you `UPDATE` on the videos row — a viral video would create a single hot row hammered millions of times/sec. (Deep dive below.)
- **Search index** (Elasticsearch) is a *derived* store kept in sync via **CDC/queue**, never dual-written (`prep/11` Part D, `prep/07` Part I).

---

## Step 5 — High-Level Design (10 min)

Two independent paths that barely touch: a thin, async **write path** and a fat, CDN-dominated **read path**.

```
WRITE PATH (rare, async)
  creator ──(1) POST /videos──► Upload Svc ──► metadata DB (row: status=PENDING)
                                     │
                                     └──(2) returns presigned multipart PUT URLs
  creator ──(3) PUT chunks──────────────────────────────────► OBJECT STORE  (source bytes)
                                                                    │ (S3 event)
  creator ──(4) POST /complete──► Upload Svc (verify object) ──► QUEUE (Kafka/SQS)
                                                                    │
                                                          ┌─────────▼──────────┐
                                                          │ TRANSCODE WORKERS  │ (autoscaled, GPU)
                                                          │  ladder + segment  │
                                                          │  + thumbnails      │
                                                          └─────────┬──────────┘
                          renditions/manifest/thumbs ──► OBJECT STORE
                          status PROCESSING → READY    ──► metadata DB ──CDC──► SEARCH INDEX

READ PATH (firehose)
  viewer ──GET /videos/{id}──► Metadata Svc ──► cache (Redis) ──► metadata DB
  viewer ──player fetches manifest + segments──► CDN ──95% hit──► (5% miss) ──► OBJECT STORE (origin)
  viewer ──POST /view──► Ingest ──► QUEUE ──► stream aggregation ──► counts store
```

Walk one upload out loud: *creator calls POST /videos → we write a PENDING row and hand back presigned multipart URLs → creator uploads chunks directly to the blob store (our servers never see the bytes) → creator calls /complete → we HEAD the object to verify size/etag, flip to PROCESSING, enqueue a transcode job → workers fan each segment out as its own job, produce the rendition ladder + manifest + thumbnails into the store, flip status to READY → CDC propagates the new READY video into the search index.*

Walk one view out loud: *player requests metadata (cache hit) → reads the manifest URL → fetches the HLS master manifest from the CDN → picks a bitrate rung → fetches ~4s segments from the CDN edge, 95%+ of which are cache hits served near the viewer → as bandwidth changes, the player switches rungs per segment. The origin/object store only ever sees the 5% long-tail misses.*

> **The one thing to never do:** proxy video bytes through your app servers (deep dive below). The
> diagram above has bytes touching the object store and the CDN — *never* the Upload/Metadata
> services. Those services only move JSON and URLs.

---

## Step 6 — Deep Dives (15 min)

I'll propose the four that win the round: (A) the upload path & why bytes bypass app servers, (B) the transcode pipeline, (C) ABR + CDN read-side scaling, (D) view counting at scale. Then live vs VOD.

### 6A — Upload: direct-to-blob, presigned, resumable

The reflex (`prep/14` Parts C & A): **the client uploads bytes directly to the object store via a presigned URL; the app server only brokers metadata.**

**Why you must never proxy video bytes through app servers** — say all of these:
- **Bandwidth doubling.** Proxying means every byte traverses your fleet *twice* (in from client, out to store). A 5 GB upload pins a connection and a chunk of a server's NIC for minutes. Multiply by upload concurrency and your stateless tier is now bandwidth-bound by the *largest* payloads, defeating the point of being stateless.
- **Memory/heap pressure & OOM.** Streaming GB-scale bodies through an app process bloats heap and kills the box under load.
- **Connection exhaustion.** App servers are tuned for many short transactions, not few multi-minute byte streams. A handful of uploads starves everyone else.
- **It's pure cost for zero value.** The object store is already a purpose-built, horizontally scaled, durable byte sink with its own bandwidth. Putting your fleet in the middle adds latency, cost, and a failure point.

> **Say it crisply:** "App servers move *metadata and credentials*, never bytes. The presigned URL is
> a time-limited, scoped capability that lets the client talk straight to S3. My fleet's job is
> authz + bookkeeping, full stop."

**Resumable / multipart upload** (`prep/14` Part C) — a 5 GB file over a flaky mobile link *will* drop:
- Split into parts (e.g. 8–64 MB each). Client requests **per-part presigned PUT URLs**, uploads them **in parallel**, and on failure **retries only the failed part** — not the whole file.
- Each part returns an **ETag**; `/complete` sends the `{partNumber, etag}` list and the store assembles the object. We **verify** (HEAD: size + etag) before flipping status — never trust the client's "done."
- This is the **claim-check + state machine** (`PENDING → PROCESSING → READY → DELETED`) and the **dual-write ordering rule**: *metadata row first (PENDING), bytes second, flip to READY last.* Reverse it and a crash leaves orphaned bytes nobody knows about (`prep/14` Part H). A reconciliation job sweeps PENDING-past-TTL → FAILED and orphaned blobs → GC.

### 6B — Async transcoding pipeline

Never block the upload response on encoding — it takes minutes to hours (`prep/14` Part F).

- **Decouple with a queue** (Kafka/SQS). The `/complete` call / S3 event enqueues a job. The queue **absorbs upload spikes** so a burst doesn't melt the encoder fleet; **workers autoscale on queue depth**.
- **Transcoding ladder** — encode each source into a ladder of resolution/bitrate **renditions** (240p/480p/720p/1080p/4K) and **codecs** (H.264 for compatibility, VP9/AV1 for efficiency — AV1 cuts ~30% bitrate, directly cutting egress cost).
- **Parallelize by segment.** A 2-hour film is *not* one serial encode — split it into chunks and **fan each segment out as its own job** across the worker pool. Massively cuts wall-clock time and lets a huge fleet share the load.
- **Idempotency** — at-least-once delivery means retries; key jobs by **content hash** so a retry doesn't double-encode (`prep/07` Part E). Poison inputs (corrupt files) → **dead-letter queue**.
- **Thumbnails** generated in the same pipeline (sample frames, pick/resize, store derivatives). Strip/re-encode to defeat malicious payloads.
- On completion: write renditions + manifest to the store, flip `status=READY`, push a notification to the creator. Client polls/streams `status` until READY.

> GPU encoding is expensive — but it runs **once per upload**, offline, off the hot path. Trading a
> burst of one-time compute for permanently CDN-cacheable, ABR-ready output is the right trade.

### 6C — Streaming: ABR + CDN (the read-side core)

**Adaptive Bitrate Streaming (HLS/DASH)** (`prep/14` Part F). You don't serve one MP4:

| Concept | What it is |
|---|---|
| **Segments** | Each rendition cut into ~2–10s **immutable** chunks (`.ts`/fMP4) |
| **Manifest** | HLS `.m3u8` / DASH `.mpd` — a tiny text file listing the bitrate variants + their segment URLs |
| **ABR** | The **player** measures throughput + buffer health and switches rungs **per segment** |
| **HLS vs DASH** | HLS = Apple-origin, ubiquitous on iOS/Safari; DASH = ISO standard, codec-agnostic. Package both, or use **CMAF/fMP4** so a single set of segments serves both |

**Why ABR matters:** bandwidth is volatile (mobile, congestion). Without ABR you either pick a low quality for everyone or stall constantly for slow clients. With ABR the player downshifts to 480p when the network dips and upshifts to 1080p when it recovers — **no rebuffering, smooth quality**. Startup is fast because the first segment is small and the player can begin at a low rung then climb.

**CDN is the entire read-side scaling story** (`prep/14` Part D, estimate above):
- Segments and manifests are **immutable, content-addressed files** ⇒ perfect cacheability with long TTLs. This is *the* answer to "how do you serve at scale": **chunked, immutable, CDN-fronted, adaptive.**
- **Cache hierarchy:** viewer → **edge PoP** (closest, smallest) → **regional/shield cache** → **origin (object store)**. The shield layer collapses many edge misses into one origin fetch, protecting origin and lifting hit ratio.
- **Push vs pull, popular vs long-tail** — the decision that maps to content distribution:
  - **Pull (lazy)** for the **long tail**: the first viewer in a region misses → origin fills the edge → everyone after hits. Correct default; you don't pre-push billions of rarely watched videos.
  - **Push (proactive/pre-warm)** for the **predictable hot set**: a Netflix new-release or a video you *know* will spike — pre-position it at edges *before* the surge so the first million viewers don't all miss simultaneously (a thundering herd against origin). Netflix's Open Connect literally ships appliances into ISPs and pre-loads tonight's popular titles overnight.
- **Offload math (from Step 2):** 95% hit ratio = 20× origin reduction; 99% = 100×. Drive it up with long TTLs + immutable URLs + the shield tier. **This ratio is the read-side KPI.**

### 6D — View counting & likes at scale (tie to analytics)

A viral video gets **millions of views/sec**. You **cannot** `UPDATE videos SET views=views+1` — that's one hot row, a single-key write hotspot, contention meltdown.

Treat it as a **streaming analytics** problem (`prep/18` Part F):
- **Ingest as events.** `/view` → 202 → event onto Kafka (keyed by `videoId`). Decoupled, spike-absorbing.
- **Batch + pre-aggregate.** Stream processor (Flink/Spark Streaming) windows events and writes **rollups** (per-minute → per-hour → per-day counters) instead of incrementing per event. Reads hit the coarsest rollup. Massive write reduction.
- **In-flight counter in Redis** (`INCR`, sharded across keys to avoid a hot key) for the near-real-time displayed count; periodically flushed/reconciled against the durable rollups.
- **Approximate vs exact — pick deliberately.** Unique viewers ("12M unique views") → **HyperLogLog** (~1.5 KB, ~2% error, **mergeable** across shards/time buckets). Trending/top-K videos → **Count-Min Sketch + min-heap**. For a *displayed view count*, 2% error is invisible; for monetization/payouts, run **exact counting in batch** that corrects later. This is exactly the **Lambda** speed-layer-plus-batch-layer split (`prep/18` Parts E–F).
- **Likes** are the same shape: 202 → event → batched aggregate, with idempotency (a like is keyed by `{userId, videoId}` so a retry is a no-op — at-least-once delivery demands it).

> **Consistency stance:** view counts and likes are **eventually consistent and approximate** by
> design, and that's correct — nobody churns because a counter read 1.04M instead of 1.05M for a few
> seconds. Spending strong-consistency machinery here would be a senior-level mistake.

### Metadata, search, recommendations (briefly)
- **Metadata service** — point-read by `videoId`, fronted by Redis (cache-aside; `prep/06`). This is a textbook cacheable read with very high hit ratio.
- **Search** — title/description/tags indexed in Elasticsearch, kept in sync from the videos DB via **CDC/queue** (never dual-write), ranked with BM25 + signals like view count/recency (`prep/11`). Autocomplete via precomputed top-K-per-prefix.
- **Recommendations** — treat as a black box: an offline pipeline computes per-user candidate lists (collaborative filtering / embeddings) into a precomputed feed store; `/feed` is then a cheap cache read. Out of scope to design the model; in scope to say *where it plugs in* (a precomputed read store fed by a batch/stream pipeline, `prep/18`).

### Live streaming vs VOD (high-level contrast)
VOD is the main case; live differs on the axes that matter:

| | VOD | Live |
|---|---|---|
| **Source** | Uploaded file, transcoded offline | RTMP/SRT ingest of a continuous stream |
| **Latency goal** | Startup < 1–2s; total latency irrelevant | **Glass-to-glass low latency** (seconds), via LL-HLS/WebRTC, smaller segments + chunked transfer |
| **Transcode** | Offline, can be slow | **Real-time** transcode of the ingest as it arrives — can't fall behind |
| **Manifest** | Static, complete | **Rolling/sliding window** — segments appended live, old ones drop |
| **Caching** | Immutable, trivially cacheable | Short-TTL segments; manifest changes constantly → harder to cache; rely on huge fan-out at edges |
| **Hard part** | Egress | **Origin ingest + real-time encode + the thundering-herd of synchronized viewers** all arriving at the same live edge |

Live trades VOD's beautiful immutability for a real-time pipeline; the CDN still does the fan-out, but segments are short-lived and the manifest is a moving target.

---

## Step 7 — Wrap-Up (3 min)

**Bottlenecks that remain:**
- **CDN egress cost is THE bottleneck** — 50 Tbps peak. Mitigations: maximize hit ratio (long TTLs, immutable URLs, shield tier), better codecs (AV1 → ~30% less egress), ISP-embedded caches (Open Connect model), per-device bitrate caps.
- **Transcode fleet** under upload spikes — bounded by queue + autoscaling, but a viral-upload event or a re-encode-everything (new codec rollout) is a capacity event.
- **Hot video metadata/counters** — solved by caching + sharded counters + rollups, but the *very* viral object still needs key-sharding to spread the load.

**Failure modes (name them before asked):**
- **Queue dies** → uploads still *land* (bytes are in the store, row is PENDING); transcoding just stalls and resumes when the queue recovers — no data loss because the source object is durable. This is the payoff of decoupling.
- **Transcode worker crashes mid-job** → at-least-once redelivery + idempotent (content-hash-keyed) jobs → safe retry; poison files → DLQ.
- **Origin (object store) outage** → CDN keeps serving the hot set from cache (graceful degradation); only cold long-tail 404s. Read path survives an origin blip — by design it's the most resilient path.
- **Dual-write gaps** (DB vs store) → reconciliation job: PENDING-past-TTL → FAILED; orphaned blobs → GC sweep.

**Single points of failure & redundancy:** multi-region object store (cross-region replication for originals), multi-CDN or multi-PoP so one provider/PoP outage doesn't black out a region, replicated + sharded metadata DB, queue replication (Kafka ISR). The metadata DB is the closest thing to a SPOF on the read path → fronted by cache so a brief DB outage still serves cached metadata.

**With more time I'd cover:** DRM/license servers, monetization/ad insertion (SSAI), per-title encoding optimization, offline downloads, and the recommendation pipeline in depth.

---

### What made this staff-level
- **Led with the binding constraint.** Identified in Step 1 that *bandwidth/CDN egress*, not compute or storage, governs the design — and *proved* it with the 50 Tbps estimate before drawing a box.
- **Two-path architecture.** Recognized upload (thin, async, rare) and playback (fat, CDN, firehose) are near-independent and optimized each separately, instead of one monolithic flow.
- **Reflexive bytes/metadata split + bytes-never-touch-app-servers**, with the *concrete reasons* (bandwidth doubling, OOM, connection exhaustion) — not just "use S3."
- **Matched precision to need:** approximate sketches/rollups for view counts (eventual, cheap), durability + verification for uploads (can't lose a creator's film). Picking the *right* consistency per subsystem.
- **Quantified the CDN.** Offload ratio as the read-side KPI, with the 20×/100× math — turned "use a CDN" into an engineering argument.
- **Named tradeoffs and failure modes unprompted**, and tied each block back to its building-block doc rather than reinventing it.

### Self-check (answer from memory before the mock)
- [ ] Why must video bytes never flow through app servers? (give 3 concrete reasons)
- [ ] Walk the resumable multipart upload + the `PENDING→PROCESSING→READY` state machine and the dual-write ordering rule.
- [ ] What is ABR, what's in an HLS manifest, and why are segments perfectly cacheable?
- [ ] Recite the CDN offload math: 95% hit ratio → what origin reduction? When push vs pull?
- [ ] How do you count views on a viral video without a hot row? (events → rollups → HLL/CMS; Lambda split)
- [ ] What's the 50 Tbps number, where does it come from, and what does it prove?
- [ ] Three ways live streaming differs from VOD.
- [ ] If the transcode queue dies, what is lost? (answer: nothing — why?)
