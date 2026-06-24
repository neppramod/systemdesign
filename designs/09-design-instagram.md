# Design 9: Instagram (Photo Sharing + Feed)

> **How to read this doc:** This is a full 45-minute worked answer, run through the 7-step
> framework from `prep/01-framework-and-building-blocks.md`. The blockquotes marked
> **"Say this in the room"** are the lines a strong candidate would actually speak. The point is
> not to memorize *this* design — it's to watch the framework *derive* it, so you can derive an
> unseen one the same way. This problem is two systems wearing one costume: a **media pipeline**
> (upload → process → CDN-deliver, the bytes path) bolted onto a **feed** (the fan-out path).
> The trap is spending all your time on the feed and waving at the photos, or vice versa. I'll
> name both bottlenecks up front and split my deep-dive budget between them. Cross-references point
> at the toolkit in `prep/01` Part B and the deep-dive topics (e.g. *blob doc 14*, *analytics doc
> 18*, *search doc 11*) — but this answer stands alone.

---

## Step 1 — Requirements (5 min)

> **Say this in the room:** "Instagram is enormous, so first I'll pin functional + non-functional
> requirements and scope hard. I want to nail two loops — **post a photo** and **read the home
> feed** — because those are the two genuine bottlenecks: the bytes pipeline and the fan-out. I'll
> mention everything else at a high level."

### Functional (the verbs)

- **Upload a photo** — user picks an image, it's stored, processed into multiple resolutions, and a post is created with an optional caption.
- **Follow / unfollow** a user — builds the social graph that drives the feed.
- **Read the home feed** — open the app, see recent posts *from people you follow*, newest-ish first, paginated, with images delivered fast.
- **Like and comment** on a post; see counts.
- **View a user's profile grid** — that one user's posts (mostly a single-key range scan — easy, not the hard part).

### Mentioned at a high level (deep-dive only if asked) — say this, it shows seniority

- **Stories / ephemeral content** (24h TTL) — I'll sketch the data model and TTL story (Step 6.6).
- **Search / Explore / hashtags** — inverted index + a recommendation candidate generator (*see search doc 11*), Step 6.7.
- **Direct messages, Reels/video, ads, notifications, moderation** — out of scope. (Video is the YouTube design; the *image* pipeline is the focus here.)

> **Say this in the room:** "I'll focus on **upload + processing**, **feed**, and **likes/comments
> at scale**. Stories, search, and explore I'll cover at the level of 'which block I'd reach for.'"

### Non-functional (where staff candidates separate themselves)

- **Read-heavy, hard.** People scroll far more than they post, and each feed item drags *image bytes*. I'll assume **~100:1 read:write** at the request level — and far higher at the *byte* level, since one upload (a few MB) is served thousands of times. This single fact justifies the CDN, aggressive caching, and precomputed feeds.
- **Two latency budgets, deliberately different:**
  - **Feed metadata read p99 < 200 ms** — the JSON page of post IDs + captions + counts.
  - **Image delivery** — first byte from a CDN edge in tens of ms; this is a *network/CDN* problem, not an app problem.
  - **Upload** — the user's request returns as soon as bytes land + a `PENDING` row is written. Processing (thumbnails/resolutions) is **async** — the user must *not* wait on it.
- **Availability over consistency** on the read path: the feed must always load. 99.99% on reads.
- **Consistency: eventual is fine** for feed delivery and for like/comment **counts**; I'll defend this explicitly (Step 6.5/6.8). The *exceptions*: a posted photo, once acknowledged, must be **durable**; and read-your-own-writes (I see my own post / my own like immediately).
- **Durability:** the original uploaded bytes and the post metadata must survive node loss. Derived artifacts (thumbnails, feeds, counts caches) are **rebuildable**.

> **Say this in the room:** "The load-bearing facts: it's ~100:1 read-heavy and *way* more so by
> bytes, so a CDN is mandatory not optional; processing is async so upload returns fast; eventual
> consistency is fine for feed + counts; and originals + post rows must be durable. Every box I draw
> follows from those."

---

## Step 2 — Estimation (3 min)

> **Say this in the room:** "I'm not chasing precision — I want numbers that *force* decisions:
> whether one DB survives, whether I need a CDN, and how big storage grows. Each number should
> point at a box."

Assume an Instagram-scale service.

### Users, uploads, traffic

| Quantity | Assumption | Result |
|---|---|---|
| DAU | given | **100 M** |
| Photos uploaded per DAU per day | avg | **2** |
| **Uploads/day** | 100M × 2 | **200 M/day** |
| **Avg write QPS** | 200M ÷ 86,400 | **~2,300 uploads/s** |
| **Peak write QPS** | 3× avg | **~7,000 uploads/s** |
| Feed opens/refreshes per DAU/day | avg | **~20** |
| **Feed reads/day** | 100M × 20 | **2 B/day** |
| **Avg feed read QPS** | 2B ÷ 86,400 | **~23,000 reads/s** |
| **Peak feed read QPS** | 3× avg | **~70,000 reads/s** |
| Images served per feed page | ~10 posts × ~1 image | ~10 |
| **Image GET/day** | 2B pages × 10 | **~20 B image GET/day** ≈ **~230k img GET/s** |

> **Say this in the room:** "7k upload QPS at peak → I can't have the user's request block on
> processing, and I want a **queue** to absorb spikes and a **horizontally sharded** metadata store.
> 230k image GET/s is the dominant axis — that's the number that makes the CDN non-negotiable."

### Storage growth (the bytes vs metadata split, quantified)

We keep, per upload, an **original** plus a set of **derivatives** (thumbnail, small, medium, large).

| Item | Math | Result |
|---|---|---|
| Original (after re-encode) | ~4 MB | — |
| Derivatives (thumb+small+med+large) | ~1.5 MB total | — |
| **Stored bytes per photo** | original + derivatives | **~5.5 MB** |
| Bytes/day | 200M × 5.5 MB | **~1.1 PB/day** |
| **Photo bytes / year** | 1.1 PB × 365 | **~400 PB/yr** |
| Metadata row per post | IDs, caption, author, ts, status | **~300 B** |
| **Metadata/day** | 200M × 300 B | **~60 GB/day** (~22 TB/yr) |

> **Say this in the room:** "Here's the punchline that *derives the whole storage architecture*:
> the **bytes grow at ~400 PB/yr** while the **metadata grows at ~22 TB/yr** — a ~20,000× gap.
> Those two have wildly different cost and scaling profiles, so they live in **different systems**:
> bytes in an object/blob store (S3-style, erasure-coded), metadata in a sharded database. That
> contrast *is* the answer to 'why split metadata from blobs' (*see blob doc 14*). 400 PB/yr also
> justifies **erasure coding** (~1.4× overhead, not 3× replication) and **lifecycle tiering** old
> photos to cold storage."

### Read bandwidth and the CDN offload (the dominant axis)

| Quantity | Math | Result |
|---|---|---|
| Image GET/day | from above | **~20 B/day** |
| Avg served object (thumb/medium) | — | **~300 KB** |
| **Total egress/day** | 20B × 300 KB | **~6 PB/day** |
| Egress to **origin** at 95% CDN hit ratio | 5% of 6 PB | **~300 TB/day** |
| **Origin offload** | 1 − (300 TB / 6 PB) | **~95% → 20× reduction** |

> **Say this in the room:** "No origin survives **6 PB/day** of image reads — that single number is
> the entire reason the CDN exists. At a 95% edge hit ratio the origin sees ~300 TB/day instead of
> 6 PB — a **20× reduction** — and immutable URLs + long TTLs push that toward 99% (another ~5×).
> I'll quantify the CDN's value, not just name it (*the offload math is from blob doc 14, Part G*)."

The chain: 7k write QPS → queue + sharded metadata + direct-to-store upload; 400 PB/yr → erasure
coding + tiering; 6 PB/day reads → CDN; 20× offload → quantified CDN value; 20,000× bytes-vs-
metadata gap → two stores. **The estimate derived the architecture before I drew a box.**

---

## Step 3 — API Design (3 min)

REST over HTTPS, JSON. Auth + rate-limiting terminate at the **API gateway** (*see rate limiter /
LB block*); `userId` comes from the auth token, never the body. The key idea: **the bytes never
flow through my app servers** — the client uploads *directly* to the blob store via a presigned URL.

```
# --- Upload (two-phase: get URL, PUT bytes direct-to-blob, then finalize) ---
POST /v1/uploads
  Authorization: Bearer <token>
  body: { contentType: "image/jpeg", bytes: <size> }
  -> 201 { uploadId, presignedPutUrl, blobKey }      # row written status=PENDING

PUT  <presignedPutUrl>                                # client → BLOB STORE directly, not my server
  body: <raw image bytes>
  -> 200 (from object store)

POST /v1/posts
  Authorization: Bearer <token>
  Idempotency-Key: <client-uuid>
  body: { uploadId, caption?, location? }
  -> 202 { postId, status: "PROCESSING" }             # enqueues processing + fan-out

# --- Feed (read hot path) ---
GET  /v1/feed?limit=10&cursor=<opaque>
  -> 200 { posts: [ {postId, authorId, caption, imageUrls:{thumb,med,large},
                      likeCount, commentCount, createdAt} ],
           nextCursor: <opaque|null> }

# --- Social + engagement ---
POST   /v1/users/{targetId}/follow      -> 204
DELETE /v1/users/{targetId}/follow      -> 204
POST   /v1/posts/{postId}/likes         -> 204     # idempotent: re-like is a no-op
DELETE /v1/posts/{postId}/likes         -> 204
POST   /v1/posts/{postId}/comments      { text }  -> 201 { commentId }
GET    /v1/posts/{postId}/comments?cursor=        -> 200 { comments:[...], nextCursor }
GET    /v1/users/{userId}/posts?cursor=           -> 200 { posts:[...], nextCursor }   # profile grid
```

### Key API choices (call these out)

- **Presigned direct-to-blob upload.** `POST /uploads` writes a `PENDING` metadata row and hands back a **presigned PUT URL** (time-limited, signed, scoped to one key). The client `PUT`s the bytes **straight to the object store** — those PB/day never touch my app servers (*see blob doc 14, Part C*). My servers stay stateless and small.
- **Finalize is a separate call.** `POST /posts` flips the row to live and **enqueues** processing + fan-out, returning `202` immediately. The user does not wait on thumbnail generation.
- **`imageUrls` are a map of renditions.** The feed returns CDN URLs for each resolution; the client picks the one its layout needs (thumb for grid, medium for feed, large for full-screen). Saves bytes.
- **Cursor-based pagination, not offset** — a feed has items inserted at the head constantly; `OFFSET` skips/duplicates as it shifts. Opaque base64 cursor encodes `(score, postId)` so paging is stable and I can change the backing store without breaking clients. *(Framework's explicit rule for feeds.)*
- **Idempotency keys** on `POST /posts` and likes — the fan-out queue is at-least-once, and clients retry on timeout; the key prevents double-posts / double-likes (*see idempotency block*).

---

## Step 4 — Data Model (5 min)

> **Say this in the room:** "I pick stores from *access patterns*, not by reflex. Let me state how
> each entity is read and the store falls out — and the headline is the **metadata/blob split** the
> estimate already forced."

### The two-store split (the headline)

| Lives in | What | Why |
|---|---|---|
| **Object/blob store** (S3-style, erasure-coded) | The image **bytes** — original + derivatives | 400 PB/yr, immutable, served via CDN; never belongs in a DB (*blob doc 14, Part A*) |
| **Metadata DB** (sharded) | Posts, users, follows, likes, comments — the *rows* | 22 TB/yr, point + range queries, relational-ish, needs indexing |

> **Say this in the room:** "A relational `BLOB`/`BYTEA` column is fine for a few KB, but multi-MB
> images bloat the DB, blow the buffer cache, and can't be CDN-served. So bytes go to object storage
> and the DB holds a **key/URL pointer**. The 20,000× growth gap from my estimate is exactly why."

### Entities and access patterns

**`users`** — point lookups by `userId`. Low write volume. Carries a denormalized `followerCount` and a `isHighFanout` flag (the celebrity bit, see 6.4). → **SQL or KV**: `userId → {handle, displayName, bio, followerCount, isHighFanout}`.

**`posts`** — write-once-ish (caption editable), read by `postId` (point) and by `authorId` (range, for the profile grid). Massive volume. → **Sharded by `postId`**, secondary access path by `authorId`.

```
posts
  PK: postId            # Snowflake-style: [timestamp | shardId | seq] -> time-sortable, no central counter
  authorId
  caption, location
  blobKeys: { original, thumb, small, medium, large }   # pointers into the object store
  status                # PENDING | PROCESSING | LIVE | FAILED  (the dual-write guard, see 6.2)
  createdAt
  -- like/comment counts kept SEPARATELY (hot, high-churn) --
```

> **Say this in the room:** "`postId` is a **Snowflake-style ID** — 64-bit, roughly time-sortable,
> with an embedded timestamp. Globally unique without a central counter, *and* the ID sorts by
> creation time so feed merges sort by ID. The `status` field is the dual-write guard: a reader
> treats anything not `LIVE` as 'not there yet.'"

**`follows`** — the social graph, stored **both ways** (denormalized) because there are two access patterns:
- "who does X follow?" → needed at read time (to pull celebrity posts).
- "who follows X?" → needed at write time (the fan-out target set).

```
follows_by_follower:  followerId -> [followeeId...]   # X's following list
follows_by_followee:  followeeId -> [followerId...]   # X's followers (the fan-out targets; can be 100M+)
```

A celebrity follower list is huge — you never load it whole, you **page** it. → **Cassandra / wide-column** (write-heavy, append-friendly; *see LSM-tree block*).

**`feed` (materialized home timeline)** — the derived, precomputed feed. Read by `userId`, returns a *capped* reverse-chronological list of `postId`s. The read hot path.

```
home_feed:  userId -> [ (postId, score), ... ]   # Redis ZSET, capped at ~500
```

- **Redis sorted set** per user, scored by timestamp/rank — `O(log n)` insert, `O(log n + k)` page read, `ZREMRANGEBYRANK` to trim (*see caching deep dive*).
- **Stores post IDs, not post content or image bytes.** The body lives once in `posts`; bodies are hydrated at read time from a post cache, and image URLs point at the CDN. Storing IDs keeps the heavily-duplicated (fan-out 200×) feed rows tiny and gives one source of truth for edit/delete.
- Backed by a rebuildable durable copy; if Redis loses it, rebuild from `follows_by_follower` + recent posts.

**`likes`** — `(postId, userId)` membership (for "did *I* like this" + dedupe) plus an aggregate **count**. The membership set and the count have *different* scaling needs (6.8). Membership: wide-column keyed by `postId`. Count: sharded counter / Redis, reconciled async.

**`comments`** — `postId → [comment...]`, range scan newest-first, paginated. Append-heavy, point-read by post. → wide-column keyed by `postId`, cursor-paginated. Plus a `commentCount` aggregate (same as likes).

**`stories`** — ephemeral, 24h TTL (6.6). `userId → [ (storyId, blobKey, expiresAt) ]`, with a native **TTL** so they self-expire. Bytes in the blob store like any image.

> **Say this in the room:** "Critical denormalization call: the feed stores post **IDs**, not post
> **content** and certainly not image bytes. Copying full posts into 200 followers' feeds would 200×
> my storage and leave 200 stale copies to fix on edit/delete. IDs + a post-body cache + CDN image
> URLs gives the read speed of denormalization without the duplication tax."

---

## Step 5 — High-Level Design (10 min)

> **Say this in the room:** "Let me get both happy paths end-to-end with the simplest thing that
> works — the **bytes path** (upload → process → CDN) and the **feed path** (fan-out → read) — then
> evolve under questions."

```
                                  ┌───────────────────────────────────────────┐
   Mobile / Web ────────────────► │  API Gateway: TLS · auth · rate-limit · L7 │
                                  └───┬───────────────┬──────────────────┬─────┘
              (1) get presigned URL   │       (3) finalize post          │ feed read
                                      ▼               ▼                  ▼
                            ┌──────────────┐  ┌──────────────┐   ┌─────────────────┐
                            │ Upload Svc   │  │  Post Svc    │   │  Feed Service   │
                            └──────┬───────┘  └───┬──────┬───┘   └───┬─────────────┘
                                   │ PENDING row  │ LIVE │ publish    │ ZREVRANGE
                                   ▼              ▼      ▼            ▼
                            ┌──────────────────────────┐ ┌────────┐ ┌──────────────┐
   (2) PUT bytes DIRECT ───►│   OBJECT / BLOB STORE     │ │ Kafka  │ │  Redis ZSET  │
   (client → blob, signed)  │   original + derivatives  │ │ topics │ │ home_feed:u  │
                            └──────────┬───────────────┘ └──┬──┬──┘ └──────┬───────┘
                                       │ origin pull         │  │          │ IDs
                            ┌──────────▼───────┐    ┌────────▼┐ │   ┌──────▼───────┐
                            │       CDN        │◄───┤ process │ │   │  Post Cache  │
                            │ (edge, 95% hit)  │    │ workers │ │   │  (bodies)    │
                            └──────────────────┘    └────┬────┘ │   └──────────────┘
                              ▲ image GET                │ write derivative keys + flip LIVE
            client reads images via CDN          ┌───────▼────────┐
                                                  │ Fan-out Workers│──► ZADD into each
                                                  │  (post.created)│    follower's home_feed
                                                  └────────────────┘
                                                  (reads follows_by_followee)
```

### Happy path A — upload + processing (the bytes path)

1. Client `POST /v1/uploads` → gateway authn/rate-limit → **Upload Service** writes a `PENDING` post row and returns a **presigned PUT URL**.
2. Client `PUT`s the raw bytes **directly to the object store** (not through my servers). The PB/day never touch my fleet.
3. Client `POST /v1/posts {uploadId, caption}` → **Post Service** verifies the object exists, sets the row to `PROCESSING`, publishes `post.created`, returns `202`.
4. **Processing workers** consume `post.created`: read the original from the object store, generate the **derivative set** (thumbnail, small, medium, large — and strip/normalize EXIF), write each back to the object store under derivative keys, then **flip the row to `LIVE`** with the `blobKeys` filled in. *(Async — the user never waited.)*
5. When `LIVE`, the Post Service publishes `post.fanout` so **fan-out workers** materialize the feed (path B). Images are served thereafter from the **CDN**, which origin-pulls from the object store on first miss and caches at the edge.

### Happy path B — fan-out + feed read (the social path)

1. **Fan-out workers** consume `post.fanout`. Normal author → page `follows_by_followee[authorId]` and `ZADD (postId, score)` into each follower's `home_feed:<followerId>` ZSET, trimming to ~500. Celebrity author → **do nothing** (6.4).
2. Client `GET /v1/feed?cursor=` → **Feed Service** → `ZREVRANGE home_feed:<userId>` for the page → list of `postId`s + scores (precomputed, fast).
3. **Merge in celebrity posts on read**: pull the few celebrities this user follows from their cached recent-posts list, merge by score.
4. **Hydrate**: batch-fetch post bodies from the **Post Cache** (fall back to `posts` on miss), attach like/comment counts, build CDN `imageUrls`, return the page + `nextCursor`. The client then fetches images from the **CDN**.

> **Say this in the room:** "Two decouplings make this work. **Bytes go direct-to-blob and are
> served by the CDN**, so my fleet handles JSON, not petabytes. And **processing + fan-out are async
> behind a queue**, so the user's upload returns in one DB write. The read path is then a single
> Redis range read + a batched hydrate — that's how I hit sub-200ms p99 on the metadata at 70k QPS,
> while images come from an edge tens of ms away."

---

## Step 6 — Deep Dives (15 min) — *where the round is won*

> **Say this in the room:** "There are two genuine bottlenecks: the **photo pipeline** and the
> **feed fan-out**. I'll split my time — pipeline + CDN first, then fan-out + the celebrity problem,
> then counters at scale — and touch consistency throughout. Can I drive?"

### 6.1 Photo upload + processing pipeline (presigned, async, queue-driven) — *see blob doc 14*

**Why direct-to-blob.** If bytes flowed through my app servers I'd need to scale the fleet for 1.1 PB/day of *ingress* and the same for egress — absurd. A **presigned PUT URL** lets the client upload straight to the object store; my server only writes a tiny `PENDING` row and signs a URL. For large originals on flaky mobile networks, hand out **multipart/resumable** presigned URLs (per-part) so a dropped connection resumes instead of restarting (*blob doc 14, Part C*).

**Why processing is async via a queue.** Generating four derivatives + EXIF normalization per image is CPU work; doing it inline would blow the upload latency budget and couple upload availability to worker availability. Instead:

```
post.created ──► Kafka ──► [processing worker pool, autoscaled on consumer lag]
                              │  read original from blob store
                              │  generate {thumb, small, medium, large}, strip EXIF
                              │  write derivatives back to blob store
                              └► flip post row status -> LIVE, publish post.fanout
```

- **Autoscale workers on Kafka consumer lag** — peak 7k uploads/s creates spikes; the queue **absorbs** them (the queue's whole job) and workers drain at their rate. A processing backlog delays *visibility* by seconds, not the upload itself.
- **Idempotent workers.** At-least-once delivery means a message can be reprocessed; deriving the same keys from `postId` makes regeneration idempotent (overwrite, don't duplicate).
- **Failure isolation.** If processing fails (corrupt image), the row goes `FAILED`, not stuck — surfaced to the user, no feed entry. A poison message goes to a dead-letter queue.

> **Say this in the room:** "Presigned direct-to-blob keeps petabytes off my servers; async
> processing behind a queue keeps the upload fast and decouples it from CPU-heavy work. The user's
> request is one signed URL + one row write; everything expensive is downstream and autoscaled."

### 6.2 The dual-write between DB and blob store (the demon nobody mentions) — *blob doc 14, Part H*

Every upload writes **two** systems with no shared transaction: the metadata DB and the object store. Failure modes and fixes:

| What fails | Result | Fix |
|---|---|---|
| Row written, bytes never uploaded | Dangling metadata → missing image | `status=PENDING` until upload confirmed; readers treat non-`LIVE` as "not there" |
| Bytes uploaded, finalize never called | Orphaned blob burning storage | Periodic **GC sweep** deletes blobs with no `LIVE` row past a TTL |
| Processing crashes mid-way | Some derivatives missing | Idempotent reprocess from the original; row stays `PROCESSING` until all keys present |

> **Say this in the room:** "The `status` field is load-bearing — it's how I make a two-system write
> safe without a distributed transaction. PENDING blobs and orphaned blobs are cleaned by a GC sweep.
> This is the same dual-write demon as the cache and the search index; the cure is always 'one side
> is the source of truth + a reconciliation sweep.'"

### 6.3 Metadata/blob split + CDN delivery (the read-heavy offload) — *see CDN block*

- **Why two stores** (from Step 4): bytes grow ~20,000× faster than rows and need different cost/scaling/serving. Bytes → erasure-coded object store (~1.4× overhead vs 3× replication → saves ~160 PB/yr) with **lifecycle tiering** (old photos → cold storage; they're rarely re-read).
- **CDN is mandatory.** 6 PB/day of image reads — no origin survives it. At 95% edge hit ratio the origin sees ~300 TB/day (**20× reduction**); immutable, content-addressed URLs + long TTLs push the hot set toward 99% (another ~5×).
- **Immutable URLs.** Derivatives are write-once; their URLs embed a content hash or version, so they're cacheable **forever** at the edge with no invalidation needed. Editing a photo writes new keys → new URLs, never a cache-purge race (*blob doc 14, Part D*).
- **Signed CDN URLs** for private accounts — time-limited tokens so only authorized clients fetch the bytes.

> **Say this in the room:** "The split is forced by the estimate: 22 TB/yr of metadata vs 400 PB/yr
> of bytes. The CDN is forced by 6 PB/day of reads — I quantify its value as a 20× origin offload.
> And immutable, versioned URLs let me cache forever and sidestep invalidation entirely — the
> hardest part of CDNs becomes a non-problem by construction."

### 6.4 The feed: fan-out on write vs read vs **hybrid** + the celebrity problem — *ties to Twitter design 01*

The **push vs pull** tradeoff, applied. (Self-contained recap of the Twitter design's core.)

**Fan-out on write (push).** On post, `ZADD` the `postId` into every follower's `home_feed`.
- ✅ **Reads dirt cheap** — feed is precomputed; a read is one range scan. Perfect for 100:1 read-heavy.
- ❌ **Writes amplify by follower count.** 200× is fine; a celebrity at **50M followers = 50M ZADDs for one post** — a write storm that saturates workers + Redis, delays everyone, and wastes work on inactive followers.

**Fan-out on read (pull).** Store nothing; at read time fetch recent posts of everyone you follow and merge-sort.
- ✅ **Writes trivial** — one store, no amplification, no celebrity storm.
- ❌ **Reads slow + expensive** — following 1,000 people = 1,000 lookups + merge **on every open**, 70k times/sec. Kills the p99 budget.

| | Write cost | Read cost | Breaks on |
|---|---|---|---|
| **Push** (fan-out on write) | O(followers) — explodes for celebs | O(page) — cheap | celebrities, inactive followers |
| **Pull** (fan-out on read) | O(1) | O(followees) merge — slow | users following many; high read QPS |
| **Hybrid (chosen)** | O(followers) for normals, O(1) for celebs | cheap + small celeb merge | nothing catastrophic |

**The hybrid.** Define a **celebrity** as `followerCount > ~100k` (the `isHighFanout` flag on the user row; tunable, per-account overridable).
- **On write:** normal author → push to all followers. Celebrity author → **skip fan-out**; their post lives in `posts` + a per-celebrity recent-posts cache (`celeb_recent:<id>`, last ~50). No 50M-write storm.
- **On read (Feed Service):** `ZREVRANGE` the precomputed feed (all *normal* followees), then look up the **handful of celebrities this user follows** (cached per user), fetch their `celeb_recent`, **merge-sort by score**, take top-k, hydrate.

The pull cost is bounded — following 20 celebrities = 20 tiny cache reads + a small merge. We pulled *only* the expensive accounts; everything else was pushed.

- **Inactive users**: don't materialize feeds for users dormant N days — fan-out workers check a "last active" bit and skip them; their feed builds on return (pull). The dormant long tail is huge — massive savings.
- **Crossing the threshold**: when an account grows past the line, stop fan-out-on-write going forward and switch to pull; existing entries age out under the ~500 cap. No backfill.

> **Say this in the room:** "Neither pure approach survives — push dies on the celebrity write storm,
> pull dies on read QPS. The staff answer is a **hybrid**: push the common case (200 followers is
> nothing), pull the pathological 0.1% tail (50M followers is catastrophic and most are inactive
> anyway). 'Apply the cheaper strategy per-account by follower count,' with a config-knob threshold."

### 6.5 Timeline cache in Redis: precompute, cap, treat as derived state (*see caching deep dive*)

- **Where:** per-user **Redis sorted set** `home_feed:<userId>`, scored by timestamp/rank. `ZADD` insert, `ZREVRANGE` page, `ZREMRANGEBYRANK` trim to ~500.
- **Capped at ~500:** nobody scrolls past a few hundred items; capping bounds memory + write cost. Page past the cap (rare) → graceful fall-through to fan-out-on-read from `posts`.
- **IDs not bodies:** ZSET holds `(postId, score)`; bodies hydrate from the **Post Cache**, images from the CDN. One source of truth, ~200× less duplicated storage, correct on edit/delete.
- **Derived + rebuildable:** Redis is the hot copy; if it's flushed, rebuild from `follows_by_follower` + recent posts. The feed is *never* the source of truth, which is what lets me run the cache hard and recover cheaply.
- **Sharded by `userId`** across a Redis cluster (consistent hashing); hot keys (a viral post's body) handled by **request coalescing / single-flight** so a viral hydration causes *one* DB read not a million, plus probabilistic early refresh so hot keys don't expire hard (the **thundering-herd** mitigation from the caching block).

> **Say this in the room:** "I precompute because reads are 100× writes — pay once at write time, not
> every read. But the feed is disposable, capped, rebuildable Redis state, and I skip inactive users.
> Precompute for the hot read path; treat the cache as a cache."

### 6.6 Stories / ephemeral content (24h TTL) — high level

- **Data model:** `stories: userId -> [ (storyId, blobKey, createdAt, expiresAt) ]`. Bytes go to the blob store + CDN exactly like a post image (same pipeline).
- **Expiry via native TTL.** Each story row carries a TTL = `createdAt + 24h`; the store (Redis/DynamoDB/Cassandra) expires it automatically — no cron scan. CDN objects get a matching max-age so edges drop them too.
- **Read path:** a "stories tray" is a small **fan-out-on-read** — pull the active (`expiresAt > now`) stories of accounts you follow and order by recency. Volume is tiny vs the feed (only last-24h, only people you follow have one), so pull is fine; no need to materialize.
- **View-state** (which stories you've seen) is per-viewer, eventually consistent, also TTL'd.

> **Say this in the room:** "Stories are just images with a **TTL** and a *pull* read model. Native
> store TTL means they self-expire with no sweeper, and because the active set is small (last 24h),
> fan-out-on-read is cheaper than maintaining a precomputed tray."

### 6.7 Likes / comments counters at scale (the hot-post problem) — *ties to analytics doc 18*

A viral post gets **millions of likes in minutes**. Naively `UPDATE posts SET likeCount = likeCount + 1` on one row = a **single-row hot-write contention** disaster (lock convoy on one key).

- **Split membership from the count.** Membership `(postId, userId)` (for "did I like this" + idempotent dedupe) goes to a wide-column store keyed by `postId`. The **count** is a separate, hot aggregate.
- **Sharded / distributed counters.** Shard the counter into N sub-counters (`like_count:<postId>:<shard>`), increment a random shard (spreads the hot write across N keys), and **sum on read**. Removes the single-row hot spot.
- **Approximate + batched at extreme scale.** For a megaviral post, exact real-time counts aren't worth the cost. Buffer increments in Redis and **batch-flush** to the durable count every few seconds (the classic write-back / micro-batch). Display can be **approximate** ("1.2M likes") — nobody needs the exact number, and small lag is invisible. For *distinct*-type metrics (unique viewers of a Story) use a **HyperLogLog** — ~1.5KB, ~2% error, mergeable across shards (*analytics doc 18*). For "top hashtags / trending," **Count-Min Sketch + a min-heap** for heavy hitters.
- **Comments** are similar but lighter: append to the wide-column comment list (keyed by `postId`, cursor-paginated) and increment a sharded `commentCount`. Comments are read in full (paginated), so no approximation there — just the count is aggregated.

> **Say this in the room:** "The hot-post counter is a hot-key write problem. I shard the counter to
> spread the write, batch-flush from a Redis buffer, and serve an **approximate** count — exactness
> isn't worth single-row contention on a megaviral post. For distinct counts (Story unique views) I
> reach for HyperLogLog, and for trending hashtags, Count-Min + a heap. This is the analytics
> toolkit applied to engagement metrics."

### 6.8 Search / Explore / hashtags — high level — *ties to search doc 11*

- **Search (users, hashtags, captions):** an **inverted index** (Elasticsearch) kept in sync from the post/user stream via **CDC / a Kafka consumer** — never a synchronous dual-write into the search index (same dual-write demon; the queue makes it eventually consistent and decoupled). Tokenize captions + hashtags; n-grams for handle prefix search.
- **Hashtags** are just an index term; "posts for #x" is an inverted-index lookup ranked by recency/engagement.
- **Explore** is a **recommendation** problem, not pure search: a candidate generator (popular + affinity-based posts) feeds a ranking model, computed **offline/near-line** and cached per user. Architecturally it's a separate read-time cache, not on the feed's critical path.

> **Say this in the room:** "Search is an inverted index fed asynchronously by CDC off the post
> stream, so it never blocks writes and stays eventually consistent. Explore is recommendations —
> offline candidate generation + ranking, cached per user — which I'd keep off the feed hot path."

### 6.9 Consistency: why eventual is OK (and the exceptions) — *CAP/PACELC*

- **Feed delivery is eventual and that's correct.** If your post lands in my feed 2–5s late, no invariant is violated, no money lost. PACELC: in the normal case I choose **L over C** (latency over consistency) because the product needs a fast feed; under partition I choose **A over C** — serve a slightly stale feed rather than fail the read. The async fan-out path is inherently eventual, and that's fine.
- **Counts are eventual/approximate** — already argued in 6.7; a like count off by a few for a second harms nothing.
- **Exception 1 — read-your-own-writes.** I must see my own post / my own like immediately or it feels broken. Handle at the edge: client optimistically renders its own action; Feed Service merges the author's own very-recent posts on read. *Global* state is eventual; *self* state is immediate.
- **Exception 2 — durability is strong.** The uploaded original + the post row, once acked, must never be lost. Derived artifacts (thumbnails, feeds, count caches) are rebuildable, so only the originals + rows need strong durability.

> **Say this in the room:** "Eventual consistency here isn't a compromise — it's correct for a
> derived, non-critical, read-heavy feed and for engagement counts. I trade freshness for latency +
> availability per PACELC, and force immediacy only for read-your-own-writes, handled at the edge.
> The one strong guarantee is **durability** of originals + post rows."

---

## Step 7 — Wrap-Up (3 min)

> **Say this in the room:** "Let me name what still hurts, how it fails, and what I'd add — before
> you ask."

### Remaining bottlenecks
- **CDN + image bandwidth** is the dominant cost axis (6 PB/day). Managed by high edge hit ratios, immutable URLs, rendition sizing, and tiering originals to cold storage.
- **Redis feed fleet** is the metadata read hot path — sharded by `userId`, hot keys via single-flight + early refresh.
- **Processing worker throughput** at peak (7k uploads/s × 4 derivatives) — horizontally scaled, autoscaled on Kafka lag.
- **Hot-post counters** — sharded counters + batched/approximate counts.

### Failure modes (degrade, don't fail)
- **Processing queue/workers die or lag.** Originals + `PENDING/PROCESSING` rows are durable; nothing lost. New posts simply appear with a delay; on recovery workers drain the backlog. Worst case: regenerate derivatives from originals (idempotent).
- **Fan-out queue dies/lags.** Posts are durable; feeds stop updating until recovery, then catch up. Worst case: rebuild feeds from the graph + `posts`.
- **Redis feed cache cold/flushed.** Reads fall back to **fan-out-on-read** from `follows_by_follower` + `posts` (slower, still loads) and re-warm. The reason the feed is rebuildable derived state.
- **CDN edge miss storm / origin spike.** Immutable URLs + request coalescing at origin; the object store is the durable origin. Worst case slower images, never lost.
- **Metadata shard down.** Affects posts on that shard only; leader-follower replication provides failover (*see replication block*).

### Single points of failure
- No single DB/queue/cache/store node is a SPOF: metadata + follows are **sharded + replicated**, object store is **erasure-coded across AZs**, Kafka **partitioned + replicated**, Redis **clustered + replicated**, services run N replicas behind L7 LBs with health checks; the gateway/LB and CDN tiers are multi-region.

### With more time, I'd add
- **Reels / video** (transcode ladder + ABR streaming — the YouTube design; *blob doc 14, Part F*).
- **Ranked feed v2** — a scoring layer over the same candidate set (precomputed + pulled), with a **chronological fallback** so the ranker is never a read-path SPOF.
- **Notifications / DMs** as separate fan-out problems reusing this queue.
- **Multi-region active-active** — geo-sharded feeds, regional CDN + object-store replicas, async cross-region metadata replication for read locality.
- **Abuse/moderation** — an async classification pipeline on the processing stream.

---

## What made this a staff-level answer

- **The estimate derived the architecture, not vibes.** The **20,000× bytes-vs-metadata gap** forced the two-store split; **6 PB/day reads** forced the CDN (quantified as a 20× offload); **7k write QPS** forced the queue + direct-to-blob upload. Each number pointed at a box.
- **Recognized this is two systems** — a media pipeline *and* a feed — and split the deep-dive budget deliberately instead of over-indexing on one.
- **Kept petabytes off the app fleet** via presigned direct-to-blob upload + CDN delivery, and made the upload fast by pushing processing async behind a queue.
- **Named the core feed tradeoff and picked a side** — push vs pull → hybrid, with explicit per-axis costs and a tunable celebrity threshold (the framework's stated staff signal).
- **Treated derived state as derived** — feeds, thumbnails, and count caches are disposable + rebuildable; originals + post rows are the durable source of truth. That distinction unlocked aggressive caching, cheap recovery, and the eventual-consistency argument.
- **Solved the hot-post counter** with sharded counters + batched/approximate counts and reached for the right sketch (HyperLogLog / Count-Min) per metric shape.
- **Handled the dual-write demon** (DB ↔ blob) with a `status` guard + GC sweep — the failure mode most candidates never mention.
- **Defended eventual consistency with PACELC** and still handled read-your-own-writes + strong durability — consistency was *chosen*, not defaulted.
- **Named failure modes and graceful degradation before being asked**, and showed every path degrades rather than fails.

> **One-line takeaway:** *Instagram = a bytes pipeline (presigned direct-to-blob → async derivative
> processing → CDN, where the estimate makes the CDN a 20× offload) bolted onto a feed (hybrid
> fan-out with the celebrity tail pulled), with everything derived — feeds, thumbnails, counts —
> kept rebuildable and eventually consistent, and only originals + post rows held strongly durable.*

---

### Self-check (answer from memory before the mock)
- [ ] Why do bytes and metadata live in different stores? (give the growth-rate numbers)
- [ ] Walk the presigned-upload flow and the `PENDING → PROCESSING → LIVE` state machine. What guards the dual-write?
- [ ] Quantify the CDN offload: total egress, origin egress at 95% hit, the multiple.
- [ ] Push vs pull vs hybrid — and exactly what a celebrity author/reader does differently.
- [ ] How do you count likes on a megaviral post without melting one DB row? When do you go approximate, and which sketch for distinct counts vs heavy hitters?
- [ ] Where is eventual consistency OK, and what are the two exceptions?
- [ ] How does the feed rebuild after a Redis flush?
