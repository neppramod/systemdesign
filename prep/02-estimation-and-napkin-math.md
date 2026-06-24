# Topic 2: Back-of-the-Envelope Estimation & The Numbers Every Engineer Must Know

> **Why this is Topic 2:** Estimation is the cheapest way to sound senior and the fastest way to
> *derive* an architecture instead of recalling one. The whole game is this move:
> "We're at ~50k write QPS and 100 TB/yr → a single Postgres won't hold that → we shard / we add a
> queue / we denormalize." The estimate is not there to impress with arithmetic — it exists to
> **justify the next box you draw**. If you can't connect a number to a decision, don't say it.

---

## Part A — The Latency Numbers Every Engineer Must Know

These are Jeff Dean's "numbers everyone should know," rounded to memorable values. You don't recite
them for points — each one teaches a **design lesson** you act on.

| Operation | Time | Memorable form | Lesson it teaches |
|---|---|---|---|
| L1 cache reference | 0.5 ns | half a ns | — (the baseline) |
| Branch mispredict | 5 ns | | |
| L2 cache reference | 7 ns | ~14× L1 | Locality matters even inside the CPU |
| Mutex lock/unlock | 25 ns | | Contention is real but cheap vs I/O |
| Main memory (RAM) reference | 100 ns | 0.1 µs | RAM is ~200× slower than L1, still "free" vs disk/net |
| Compress 1 KB (Snappy) | 3,000 ns | 3 µs | Compress before you ship over the network |
| Send 1 KB over 1 Gbps net | 10,000 ns | 10 µs | | 
| **Read 1 MB sequentially from RAM** | 0.25 ms | 250 µs | RAM bandwidth ~4 GB/s |
| Round trip within a datacenter | 0.5 ms | 500 µs | A same-DC RPC is "cheap" — but it's still 5,000× a RAM hit |
| **SSD random read** | 100–150 µs | ~0.1 ms | SSD random read ≈ 1000× RAM, but 100× faster than disk seek |
| **Read 1 MB sequentially from SSD** | ~1 ms | | SSD seq read ~1 GB/s |
| **Disk seek (spinning)** | 10 ms | | Why we avoid random disk I/O at all costs |
| **Read 1 MB sequentially from disk** | 20–30 ms | | Disk seq read ~30–100 MB/s |
| **Cross-region RTT (same continent)** | ~30–50 ms | | Every cross-region hop costs you a frame of video |
| **Cross-continent RTT (US↔EU)** | ~80–100 ms | | This is *physics*, not your code — speed of light in fiber |
| **US↔India / US↔Australia RTT** | ~150–250 ms | | Why global apps need regional replicas + edge |

### The lessons, stated as one-liners (say these out loud)

- **RAM is ~100 ns, SSD random read is ~100 µs, disk seek is ~10 ms.** That's roughly
  **1 : 1,000 : 100,000**. Memorize this ladder — it's the spine of every caching argument.
- **A same-DC round trip (0.5 ms) is 5,000× a memory read.** So "just one more network hop" is not
  free; chatty microservices die here.
- **Cross-region RTT is ~30–100 ms and you cannot beat it** — it's the speed of light. The only
  fixes are: put data near the user (replicas/CDN), or do fewer round trips (batching, locality).
- **Sequential beats random by 100×** on both disk and SSD. This is *why* LSM-trees, append-only
  logs (Kafka, WAL), and columnar formats exist.
- **Compression (3 µs/KB) is far cheaper than transmission.** Compress before crossing the network
  or the disk boundary almost always.

> **The reflex this section buys you:** any time a design touches disk on the read path, you should
> twitch. "That's a 10 ms seek per request → at 10k QPS we're toast → cache it / keep the working
> set in RAM / use SSD + sequential layout." The number *forces* the architecture.

A useful derived fact: **a single modern machine does ~order 100k–1M simple in-memory ops/sec
per core comfortably, but only ~hundreds of disk-seek-bound ops/sec.** That gap is the entire
reason caches exist.

---

## Part B — Powers of Two & Fast Mental Math

### Data size cheat sheet (powers of 2, but you compute in powers of 10)

| Power | Exact-ish | Name | "Think of it as" |
|---|---|---|---|
| 2^10 | 1,024 | KB | ~10^3 (a thousand) |
| 2^20 | ~1.05 M | MB | ~10^6 (a million) |
| 2^30 | ~1.07 B | GB | ~10^9 (a billion) |
| 2^40 | ~1.1 T | TB | ~10^12 (a trillion) |
| 2^50 | ~1.1 quadrillion | PB | ~10^15 |

> **The only trick you need:** in the room, **treat KB/MB/GB/TB as 10^3 / 10^6 / 10^9 / 10^12.**
> The error is ~7% per step — irrelevant for sizing. Never burn time on 1,024.

### Byte sizes to keep in your pocket

| Thing | Rule-of-thumb size |
|---|---|
| ASCII char | 1 byte |
| Unicode char (UTF-8 typical) | 1–3 bytes |
| 64-bit int / timestamp / ID | 8 bytes |
| UUID | 16 bytes |
| A tweet (text only) | ~200–300 bytes (assume 280 chars ≈ **300 B**) |
| A tweet row w/ metadata (IDs, ts, counts) | ~1 KB (safe overestimate) |
| A small JSON API response | ~1–4 KB |
| A thumbnail | ~10–20 KB |
| A web-sized photo | ~200 KB – 2 MB (assume **~500 KB–1 MB**) |
| 1 minute of 1080p video | ~50–100 MB (assume **~50 MB**) |
| 1 hour of 1080p video | ~3 GB |

### Fast division tricks (the "rule of 72" style mental math)

- **Seconds in a day ≈ 86,400 ≈ 10^5.** Memorize: **~100k seconds/day.** (Real value 86.4k; using
  100k makes QPS math trivial and slightly conservative-low — note it out loud.)
  - Many people use **~86,400 ≈ 9 × 10^4** and round QPS up. Either is fine; pick one and be
    consistent.
- **1 million/day ≈ ~12 QPS.** (10^6 / 86,400 ≈ 11.6.) Memorize **1M/day ≈ ~10 QPS**.
  - Therefore **1 billion/day ≈ ~12,000 QPS**, and **100M/day ≈ ~1,200 QPS**.
- **Seconds in a year ≈ 3.15 × 10^7 ≈ 30M.** Memorize **~31.5M sec/yr ≈ 3 × 10^7**.
- **Per-year from per-day:** ×365 ≈ **×400** in your head (overestimate, safe), or ×365 exactly if
  the number is round.
- **Powers-of-ten division:** subtract the exponents. 10^9 events / 10^5 sec = 10^4 QPS. Do
  *everything* in scientific notation so you never lose a zero — losing a zero is the #1 way to
  blow this section.

> **Say-this-out-loud anchors:** "100k seconds in a day, so 1 million events/day is about 12 QPS,
> and a billion a day is about 12k QPS." With just those, you can QPS almost any prompt in 10
> seconds.

---

## Part C — QPS Estimation

### The core formula

```
Average QPS = (DAU × actions per user per day) / seconds per day
            = (DAU × actions) / 86,400      (≈ /100,000)

Peak QPS    = Average QPS × peak multiplier  (use 2–3×; spiky apps 5–10×)
```

Then split by operation type using the **read:write ratio** — because reads and writes scale
differently (reads → replicas/cache; writes → sharding/queues), this split drives the whole design.

### Peak multipliers — what to assume and why

| App type | Peak multiplier | Why |
|---|---|---|
| Steady B2B / internal | ~2× | Traffic spread across business hours |
| Consumer social/feed | ~2–3× | Evening prime-time spike |
| E-commerce | ~3–5× | Normal day; **10–100× on Black Friday / flash sale** — call this out |
| Live event / sports / ticketing | ~10–50× | Everyone hits at the same instant |
| Messaging | ~2–3× | Diurnal, plus event-driven bursts |

State your multiplier and **why**: "Consumer social, so I'll take 3× for prime time; if this were a
ticketing flash sale I'd take 20× and design the write path for that."

### Read:write ratios by archetype (memorize these)

| Archetype | Read:Write | Notes / what it implies |
|---|---|---|
| **Social feed (Twitter/IG)** | ~100:1 (often 1000:1 for celebs' content) | Read-heavy → cache + read replicas + fan-out strategy |
| **E-commerce browse/buy** | ~100:1 (views) to ~10:1 if you count cart actions | Read-heavy catalog, but writes (orders/payments) need strong consistency |
| **Messaging (WhatsApp/Slack)** | ~1:1 to ~10:1 (each msg written once, read by recipients) | More balanced; fan-out to N recipients turns 1 write into N "deliveries" |
| **Video (YouTube/Netflix)** | ~1000:1+ on bytes | Read is the *bandwidth* problem (CDN); writes (uploads) rare but huge |
| **Analytics / logging ingest** | write-heavy (1:100 the other way) | Append-only, LSM/columnar, batch reads |
| **Collaborative doc (Google Docs)** | ~10:1 with high write *concurrency* | Conflict resolution (CRDT/OT) matters more than raw QPS |

> **The point of the ratio:** it tells you which axis to scale. 100:1 read-heavy → "I'll throw a
> cache and read replicas at reads and not worry about write scaling yet." 1:1 with N-way fan-out →
> "the write path *is* my problem; I need a queue and idempotent delivery."

---

## Part D — Storage Estimation (Worked Examples)

General formula:
```
Storage/yr = writes/day × bytes/write × 365 × replication factor
```
Always state your **replication factor** (typically **3×**) and **retention** — interviewers love
to see both, because they 2–3× the answer.

### Example 1 — Tweets per year

Assume 300M DAU, each posts 2 tweets/day, **1 KB/tweet** (text + metadata, generous):

- Writes/day = 300M × 2 = **600M tweets/day**
- Raw/day = 600M × 1 KB = **600 GB/day**
- Raw/year = 600 GB × 365 ≈ **~220 TB/yr**
- With **3× replication** ≈ **~660 TB/yr ≈ ~0.6–0.7 PB/yr**

Lesson: text is *small*. Even all of Twitter's text is sub-petabyte/yr. So **the metadata DB is
shardable but not scary; the scary storage is always media.**

### Example 2 — Photos (Instagram-style)

Assume 500M DAU, 0.5 photos uploaded/day, **1.5 MB/photo** (original) + 3 resized versions ≈ **2 MB
total stored**:

- Uploads/day = 500M × 0.5 = **250M photos/day**
- Storage/day = 250M × 2 MB = **500 TB/day**
- Storage/year = 500 TB × 365 ≈ **~180 PB/yr** (before replication; ×3 → ~0.5 EB/yr)

Lesson: **3 orders of magnitude bigger than the text.** This *forces* object storage (S3/blob) +
CDN, not a database. The DB stores the *pointer* (a 100-byte row), not the bytes.

### Example 3 — Video (YouTube-style)

Assume 500 hours uploaded/minute (real YouTube figure), **~3 GB/hr stored**, ×~3 for multiple
encodings/resolutions:

- Hours/day = 500 × 60 × 24 = **720,000 hours/day**
- Storage/day = 720k × 3 GB × 3 (encodings) ≈ **~6.5 PB/day** of new video
- Per year ≈ **~2.3 EB/yr**

Lesson: this is **exabyte scale** → tiered storage (hot/warm/cold), aggressive transcoding, and the
*real* cost shifts from storage to **egress bandwidth** (Part E). Storage is cheap; serving it is
not.

---

## Part E — Bandwidth Estimation

```
Bandwidth (bytes/sec) = QPS × payload size
```
Convert to bits for network sizing (×8). State **ingress vs egress separately** — egress is what
costs money and what CDNs exist for.

### Worked: serving a read-heavy media app

Say 1M read QPS (peak), average response 1 MB (a photo):

- Egress = 1M × 1 MB = **1 TB/sec = 8 Tbps**
- That **cannot come from origin** → CDN offloads the bytes; origin serves cache misses only.
- If CDN hit rate is 95%, origin sees 5% → still 50k QPS × 1 MB = **50 GB/s = 400 Gbps** at origin.

### Worked: video streaming

10M concurrent viewers, 1080p ≈ **5 Mbps/stream**:

- Total = 10M × 5 Mbps = **50 Tbps**
- Single 10 Gbps NIC → you'd need **5,000 NICs' worth** → this *is* the CDN/edge argument, full stop.

> **Bandwidth lesson:** for media, **bandwidth — not storage and not QPS — is usually the binding
> constraint.** The moment egress hits Tbps, the answer is "CDN + edge caching," and your estimate
> just *proved* it instead of asserting it.

---

## Part F — Memory / Cache Sizing

### How big should the cache be? The 80/20 working-set rule

You don't cache everything. You cache the **hot working set**. Standard assumption:

> **80% of requests hit 20% of the data** (often even 90/10 for feeds). So size the cache to hold
> the hot ~20%, and you serve the large majority of reads from RAM.

```
Cache size ≈ (hot fraction) × (total dataset) , OR
Cache size ≈ (items read per day) × (item size) × (working-set window, e.g. 1 day)
```

### Worked: cache for a feed/timeline

300M DAU, each reads a feed of ~200 cached tweet-IDs, value ~ 200 IDs × 8 B = **~1.6 KB/timeline**,
round to **~2 KB**:

- Caching all 300M users' timelines = 300M × 2 KB = **600 GB**
- A single big Redis box holds ~100–300 GB usable → **need ~3–6 shards** to hold all timelines in RAM.
- Or cache only **active** users' timelines (the 20%) → ~120 GB → **fits in 1–2 nodes.**

### How many machines to hold X in RAM

```
# machines = ceil( dataset to cache / usable RAM per node )
```
Assume **~64–256 GB RAM/node, ~75% usable** for data (leave headroom for overhead/fragmentation).
Example: 600 GB working set / (256 × 0.75 ≈ 190 GB) ≈ **~4 nodes** (then ×2 for replication/HA → ~8).

> **Cache-sizing lesson to say:** "I won't size the cache to the whole dataset — I'll size it to the
> hot 20%, validate the hit rate in prod, and grow it. Caching the cold tail wastes RAM for marginal
> hit-rate." That sentence signals you've actually run a cache.

---

## Part G — Number of Servers Estimation

```
# servers (stateless) = ceil( peak QPS / QPS-per-server ) × redundancy factor
```

### QPS-per-server assumptions (and how to justify them)

| Workload | Assume per server | Justification to say |
|---|---|---|
| Simple stateless API (mostly proxy/validate, cache-backed) | **~1,000–10,000 QPS** | "CPU-light, I/O to cache; modern box + async handles thousands" |
| CPU-bound service (serialization, business logic) | **~500–2,000 QPS** | "Each request burns real CPU; fewer per box" |
| DB-backed write (no cache, disk-bound) | **~hundreds–1,000 QPS/node** | "Bounded by disk fsync / index writes" |
| In-memory cache node (Redis) | **~100k+ ops/sec** | "Single-threaded but RAM-fast; network-bound" |
| Connection-heavy (WebSocket/long-poll) | **~10k–100k conns/node, low msg rate** | "Memory per connection, not CPU, is the limit" |

The honest move: **pick a number, state the assumption, then show the math.** Interviewers don't
care if it's 5k or 8k QPS/server; they care that you *named the assumption* and could revise it.

### Worked

Peak 60k QPS, assume 6k QPS/server → **10 servers**, ×1.5 for headroom/HA → **~15 servers**, spread
across ≥2 AZs. "I'd autoscale on CPU/latency rather than pin to 15 — this is the *floor*."

---

## Part H — Fully Worked: "Design Twitter at Scale" Estimation Walkthrough

The interviewer says "design Twitter." Here's the entire estimation pass, end to end, in the order
you'd speak it.

**1. Establish the scale knobs (ask or assume out loud):**
- DAU = **300M**, avg follows = 200, avg followers = 200 (power-law: a few have 100M).
- Each user: posts **2 tweets/day**, reads feed **20×/day**.
- Tweet ≈ **1 KB** stored (text + metadata), timeline entry ≈ **8–16 B** (just IDs).

**2. Write QPS (tweets):**
- 300M × 2 = 600M tweets/day. ÷ 86,400 ≈ **~7k writes/sec avg**, ×3 peak ≈ **~20k write QPS**.

**3. Read QPS (feed loads):**
- 300M × 20 = 6B reads/day ÷ 86,400 ≈ **~70k reads/sec avg**, ×3 peak ≈ **~210k read QPS**.
- Ratio ≈ **210k : 20k ≈ 10:1** at the request level (higher per-item because each feed shows many
  tweets). **Read-heavy → cache + replicas + precomputed timelines.**

**4. Fan-out write amplification (the real load):**
- Avg 200 followers → each tweet write fans out to ~200 timeline inserts.
- 7k tweets/sec × 200 = **~1.4M timeline writes/sec** (fan-out on write).
- **This is the number that breaks the naive design.** 1.4M writes/sec is not a single DB; it's a
  sharded timeline store + a queue (Kafka) to absorb and parallelize fan-out.
- Celebrities (100M followers) make a *single* tweet = 100M writes → **hybrid: fan-out on write for
  normal users, fan-out on read (pull) for celebrities.** The estimate *forced* the hybrid.

**5. Storage:**
- Tweets: 600M/day × 1 KB × 365 × 3 (repl) ≈ **~0.65 PB/yr** for tweet content. Shardable, fine.
- Timelines (if cached, not stored): hot users × ~2 KB ≈ low hundreds of GB → fits in a Redis
  cluster.
- Media (if we include it): dominates — pushes us to **blob storage + CDN**, DB stores pointers.

**6. Bandwidth:**
- Read egress: 210k QPS × ~10 KB/feed-response ≈ **~2 GB/s ≈ 16 Gbps** for text feeds — manageable
  at origin with a few LBs; media would add Tbps → CDN.

**7. Cache / memory:**
- Active-user timelines in RAM: ~60M active × 2 KB ≈ **~120 GB** → ~2–4 Redis nodes (+HA).

**8. Servers (stateless feed/post services):**
- Peak ~230k total QPS / ~5k QPS-per-server ≈ **~46 servers**, ×1.5 → **~70**, across AZs,
  autoscaled.

**Now the payoff — every architectural decision is justified by a number:**

| Number we computed | Decision it forces |
|---|---|
| 1.4M timeline writes/sec | Queue (Kafka) + sharded timeline store; fan-out workers |
| Celebrity = 100M writes/tweet | Hybrid fan-out (push for normal, pull for celebs) |
| 10:1 read-heavy | Read replicas + Redis timeline cache |
| 0.65 PB/yr text, but media >> that | Blob store + CDN for media, DB for pointers |
| 230k peak QPS | ~70 stateless app servers across AZs, autoscaled |
| 120 GB hot timelines | Redis cluster sized to hot set, not full dataset |

> **This is the whole skill in one table.** You didn't *remember* the Twitter architecture — you
> *derived* it from six numbers. That is exactly what transfers to an unseen problem.

---

## Part I — Using Estimates to Justify Architecture (the whole point)

A cheat sheet of "number → decision" reflexes. When a number crosses a threshold, a box appears.

| If you compute… | …then say |
|---|---|
| Write QPS > ~5–10k, or > what one node's disk does | "Single primary won't hold this → **shard** the write path." |
| Read QPS ≫ write QPS (100:1) | "Read-heavy → **cache + read replicas**; writes aren't my bottleneck yet." |
| Fan-out makes 1 write → N writes, N large | "**Queue** to absorb + parallelize; consider **fan-out on read** for the heavy tail." |
| Working set > one machine's RAM | "**Shard the cache** / use a cluster; size to the hot 20%." |
| Egress > tens of Gbps, or any media at scale | "**CDN + edge**; origin serves misses only." |
| Storage > what fits/grows on one DB (~TBs/yr+) | "**Shard / tier** storage; cold data to cheaper tiers." |
| Spiky writes (flash sale 10–50×) | "**Queue + buffer**; protect the DB with backpressure/rate limits." |
| Strong consistency needed on a hot path | "Limit the strongly-consistent surface; everything else eventual." |

> **The rule:** never introduce a complex component (shard, queue, CDN, consensus) without a number
> that demands it. "We shard because it's web-scale" is junior. "We shard because 1.4M writes/sec
> exceeds a single node by ~1000×" is staff.

---

## Part J — Common Estimation Mistakes & How Interviewers Judge This Section

### Mistakes that cost you

- **Losing a zero.** The #1 killer. Work in scientific notation (10^x) and the zeros can't escape.
- **Estimating for its own sake.** Spitting numbers that connect to no decision. If a number doesn't
  change a box, don't compute it.
- **Forgetting replication (×3) and retention (×years).** These swing storage 5–10×.
- **Using *average* QPS to size capacity.** You provision for **peak**, not average. Always apply
  the multiplier.
- **Ignoring fan-out / read amplification.** "1 tweet" is not 1 write when it hits 200 timelines.
  Missing this hides the real bottleneck.
- **Over-precision.** Spending 4 minutes computing 86,400 × 365 to the digit. Round hard, move on.
- **Media-blindness.** Treating photos/video like rows in a DB. Media → blob + CDN, always.
- **No stated assumptions.** A number with no assumption behind it can't be reasoned about or revised.

### What interviewers are actually scoring

- **Did you derive decisions from numbers, or assert architecture and backfill?** (The big one.)
- **Did you state assumptions and round sensibly?** Speed + honesty beats false precision.
- **Did you find the binding constraint?** (write fan-out? egress? storage?) Naming it is the signal.
- **Could you redo the math if they change an input** ("what if DAU is 3B?") without restarting?
- **Time discipline.** This is a **~3-minute** part of a 45-min round. Don't let it eat your deep
  dive. Get the 2–3 numbers that drive the design, then move on.

---

### Self-check before the mock (answer these from memory)
- [ ] Recite the latency ladder: RAM ≈ ___ ns, SSD random read ≈ ___ µs, disk seek ≈ ___ ms, same-DC RTT ≈ ___ ms, cross-continent RTT ≈ ___ ms.
- [ ] What's the RAM : SSD : disk-seek ratio, and what does it justify?
- [ ] Seconds in a day ≈ ? Seconds in a year ≈ ? 1M/day ≈ ? QPS? 1B/day ≈ ? QPS?
- [ ] Give the QPS, storage, bandwidth, and #-of-servers formulas cold.
- [ ] Read:write ratios for social, messaging, video, e-commerce — and what each implies.
- [ ] Peak multipliers: consumer social vs flash sale vs live event.
- [ ] Why is media storage/bandwidth the usual binding constraint, not the text DB?
- [ ] State the 80/20 working-set rule and how you'd size a cache from it.
- [ ] Walk the Twitter estimation end to end and name the one number that forces the hybrid fan-out.
- [ ] For each of: 50k write QPS, 100:1 reads, Tbps egress, working-set > 1 box — name the decision it forces.
