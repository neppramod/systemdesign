# Design 02: URL Shortener (TinyURL / bit.ly)

> **How to read this doc:** This is a *full worked walkthrough* of the 7-step framework from
> [Topic 1](../prep/01-framework-and-building-blocks.md), solved live as if in a 45-min round.
> The URL shortener looks trivial ("it's just a hash map") — which is exactly the trap. The
> staff-level signal is treating the *short-code generation* and the *read-heavy redirect path*
> as the real problems and reasoning about them with numbers. Everything in **bold/blockquote**
> is something I'd actually say out loud.

---

## 0. Why this problem is worth taking seriously

A junior says "store `long → short` in a map, done." A staff engineer hears four hard sub-problems:

1. **Generating short codes** that are short, unique, non-guessable-enough, and don't require global coordination on every write.
2. A **brutally read-heavy** workload (redirects ≫ creates) → the design is really a *caching* problem.
3. **Cleanup / expiration** of dead links at scale.
4. **Click analytics** that must not slow the redirect.

I'll drive the conversation toward those.

---

## 1. Requirements (5 min) — I drive this

### Functional
- **Create**: given a long URL, return a short URL (`https://sho.rt/{code}`).
- **Redirect**: `GET /{code}` → HTTP redirect to the original long URL.
- **Custom alias** (optional): user can request `sho.rt/my-brand`.
- **Expiration** (optional): link can have a TTL or expiry date.
- **Analytics** (optional): click counts, referrer, geo, device.

> "I'll treat create + redirect as the core, and explicitly scope custom aliases, expiry, and
> analytics as features I'll layer on in the deep dive. I'll skip user accounts, billing, and the
> UI."

### Non-functional (this is where the design is decided)
| Dimension | Target | Why it matters |
|---|---|---|
| **Read:write ratio** | **~100:1 to 1000:1** | Redirects vastly outnumber creates. This makes it a *caching/CDN* problem, not a write-scaling problem. |
| **Latency** | Redirect **p99 < ~50–100ms** | A redirect sits in the user's critical path before they even reach the destination. Must feel instant. |
| **Availability** | **99.99%+ on the redirect path** | A dead shortener breaks every link ever printed on a billboard, email, QR code. Redirects must basically never fail. |
| **Consistency** | **Eventual is fine** | If a freshly created link takes a few seconds to be globally readable, nobody cares. Codes are immutable once issued, which makes caching trivial. |
| **Durability** | **High** | Losing the mapping = breaking permanent links. This is the one place we can't be sloppy. |

> **The framing sentence:** "This is an extremely read-heavy, eventually-consistent,
> high-durability key-value workload with a tiny write path and a huge cache-friendly read path.
> That single sentence picks most of my building blocks."

Map to blocks ([Topic 1, Part B](../prep/01-framework-and-building-blocks.md)): **KV store**, **cache (Redis)**, **CDN/edge**, **a key-generation service**, **a queue** for analytics.

---

## 2. Estimations (3 min) — to justify, not to impress

Assume a bit.ly-scale service.

### Write (create) volume
| Quantity | Value |
|---|---|
| New URLs / month | **100M** |
| Writes / sec (avg) | `100M ÷ (30 × 86,400)` ≈ **~40 writes/sec** |
| Peak writes/sec (3×) | **~120/sec** |

Writes are *trivial*. A single modest DB node could absorb this. **Writes are not the bottleneck.**

### Read (redirect) volume
| Quantity | Value |
|---|---|
| Read:write ratio | **100:1** (conservative) |
| Reads / sec (avg) | `40 × 100` = **~4,000/sec** |
| Peak reads/sec (3×) | **~12,000/sec** |

> 4k–12k QPS of point lookups by key. That is *easily* cacheable. With an even more aggressive
> 1000:1 ratio you're at ~40k QPS — still well within a Redis cluster + CDN.

### Storage (the math that actually matters)
| Quantity | Value |
|---|---|
| Bytes per record | short code (~8B) + long URL (~500B avg) + metadata (~100B) ≈ **~600B–1KB** |
| Records / year | `100M × 12` = **1.2B / year** |
| Storage / year | `1.2B × ~1KB` ≈ **~1.2 TB / year** |
| Over 10 years | **~12 TB**, plus replication ×3 → **~36 TB** |

> Tens of TB. Fits comfortably in a sharded KV store. Not a "we must shard or die today" number,
> but enough that I'll design for horizontal sharding from the start.

### Keyspace / code-length math (the headline estimate)
We use **base62** (`[a-z A-Z 0-9]`, 62 symbols). Number of codes for `n` characters = `62^n`:

| Code length | Capacity (`62^n`) | Enough for? |
|---|---|---|
| 5 | ~916 million | ~9 years at 100M/yr — too tight |
| 6 | ~56.8 billion | ~47 years at 100M/yr ✅ comfortable |
| 7 | ~3.5 trillion | basically forever |
| 8 | ~218 trillion | overkill |

> **I'll pick a 7-character base62 code.** 6 covers the realistic horizon, but 7 gives headroom for
> growth, for burning codes on expiry/collision/custom aliases, and keeps us from a painful
> migration. Codes stay short enough to be "tiny." This is the kind of number an interviewer wants
> to hear derived, not guessed.

---

## 3. API design (3 min)

```
POST /api/v1/shorten
  body: { longUrl, customAlias?, expiresAt? }
  -> 201 { shortUrl: "https://sho.rt/aB3xK9p", code: "aB3xK9p" }
  errors: 409 if customAlias taken, 400 if URL invalid

GET /{code}
  -> 301 or 302 redirect, Location: <longUrl>   (see deep dive on 301 vs 302)
  -> 404 if unknown / expired

GET /api/v1/stats/{code}        (owner only)
  -> 200 { clicks, lastAccessed, topReferrers, geoBreakdown }

DELETE /api/v1/{code}           (owner only)
```

- Creates are authenticated + **rate-limited at the gateway** (prevents abuse / spam-link farms).
- The redirect endpoint is *unauthenticated and public* — it has to be, links are shared openly.
- Note the bare-path redirect (`GET /{code}`) so the URL stays as short as possible — no `/r/` prefix.

---

## 4. Data model (5 min)

### Access patterns first (these decide the store)
- **Write**: insert one immutable record keyed by `code`. ~40/sec.
- **Read**: point lookup by `code`. ~4k–40k/sec. **No range scans, no joins, no ad-hoc queries.**
- **Secondary**: custom-alias uniqueness check (point lookup by alias = same key); analytics (separate path).

> "The dominant access is a *point lookup by a single key* with no relational queries. That's the
> textbook signature for a **key-value / wide-column store**, not a relational DB."
> (See SQL-vs-NoSQL note in [Topic 1, Part B](../prep/01-framework-and-building-blocks.md) — *decide
> from access patterns, not reflex*.)

### The mapping table (KV store — DynamoDB / Cassandra)
| Field | Notes |
|---|---|
| `code` (PK) | 7-char base62 short code (or custom alias) |
| `longUrl` | the destination |
| `creatorId` | owner, for stats/delete + rate-limit attribution |
| `createdAt` | timestamp |
| `expiresAt` | optional; drives TTL cleanup |
| `isCustom` | flag |

**Why KV over SQL here:**
- Access is pure key lookup → no need for SQL's joins/transactions.
- Horizontal scale + high availability come free; the data shards naturally on `code`.
- We don't need cross-row transactions (each link is independent).

> The one place SQL would tempt you is custom-alias uniqueness (a constraint). But a KV
> conditional write (`put-if-absent` / DynamoDB `attribute_not_exists`) gives us that atomically
> without a relational DB. So KV wins.

### Analytics store (separate)
Click events are **append-heavy, time-series, aggregate-queried** — a totally different access
pattern. I'll keep them out of the mapping store and use a columnar/OLAP sink. More in the
analytics deep dive.

---

## 5. High-level design (10 min) — happy path end to end

```
                         ┌──────────────┐
        create  ───────► │  API Gateway │  (authn, rate-limit, TLS)
                         └──────┬───────┘
                                │
                        ┌───────▼────────┐      ┌──────────────────┐
                        │  Write Service │◄─────│ Key-Gen Service  │ (pre-issues codes)
                        └───────┬────────┘      └──────────────────┘
                                │ put-if-absent
                        ┌───────▼────────┐
                        │   KV Store     │  (DynamoDB/Cassandra, sharded by code, RF=3)
                        └───────┬────────┘
                                │
   redirect ──► CDN/edge ──► ┌──▼─────────────┐   miss   ┌──────────┐
   GET /{code}   (cache)     │ Redirect Svc   │◄────────►│  Redis   │ (cache-aside)
                             └──────┬─────────┘          └────┬─────┘
                                    │ 301/302                  │ miss
                                    │                          ▼
                                    │                     KV Store
                                    │ fire-and-forget click event
                              ┌─────▼──────┐    ┌──────────────┐    ┌───────────┐
                              │ Kafka/queue│───►│ Analytics    │───►│ OLAP store│
                              └────────────┘    │ consumer     │    └───────────┘
                                                └──────────────┘
```

### Create path (write)
1. Client → gateway (authn + rate-limit) → Write Service.
2. If `customAlias`: `put-if-absent(alias)` → 409 on conflict.
3. Else: pull a pre-generated code from the **Key-Gen Service**, `put` the record.
4. Return short URL. (~40/sec — boring on purpose.)

### Redirect path (read — the hot path)
1. `GET /{code}` hits **CDN/edge** first. For popular links, the edge can serve the redirect without ever touching origin.
2. On edge miss → Redirect Service → **Redis (cache-aside)**.
3. Cache hit (the common case) → return redirect immediately.
4. Cache miss → read KV store, populate cache, return.
5. **Asynchronously** emit a click event to Kafka (fire-and-forget — never blocks the redirect).

> "I'm walking the read path slowly because it's 99% of traffic. Notice the redirect itself does
> *zero* synchronous work beyond a cache lookup. Everything expensive — analytics — is shoved
> off the critical path onto a queue."

---

## 6. Deep dives (15 min) — where the round is won

### 6.1 Short-code generation — the central design decision

I'll compare four strategies on: **length, uniqueness/collisions, predictability, coordination cost.**

#### Option A — Hash of the long URL (e.g. MD5/SHA, take first N base62 chars)
- **Idea:** `code = base62(hash(longUrl))[:7]`.
- **Pro:** stateless; same URL naturally maps to same code (free idempotency).
- **Con — collisions:** truncating a hash to 7 chars *will* collide as you fill the keyspace (birthday paradox). You need collision handling: on `put-if-absent` conflict, re-hash with a salt/counter and retry. Each retry is an extra DB round-trip.
- **Con — predictability:** hashes are uniform but the codes are not sequential; fine. But you can't make them shorter without raising collision rate.
- **Con:** two users shortening the same URL share a code — usually *good*, but it breaks per-user analytics and per-user expiry (whose TTL wins?).

#### Option B — base62 of an auto-increment ID
- **Idea:** DB hands out a monotonically increasing integer `id`; `code = base62(id)`.
- **Pro:** **zero collisions by construction** (IDs are unique). Codes are as short as possible for the count issued (id=1 → "b", grows with N). Simple to reason about.
- **Con — predictability:** codes are sequential and **enumerable**. An attacker can walk `1,2,3…` and scrape every link. Mitigate by XOR/Feistel-permuting the ID into the base62 space (bijective scramble) so codes look random but stay collision-free.
- **Con — coordination (the real problem):** a single global auto-increment counter is a **write bottleneck and a single point of failure**. Every create must touch it. At 40/sec it's fine; the *concern* is coordination, not throughput. See 6.2.

#### Option C — Key-Generation Service (KGS) with pre-generated keys
- **Idea:** an **offline** service generates billions of random unique 7-char codes *in advance*, stores them in a `available_keys` table, and hands them out in **batches** to write servers.
- Each write server checks out, say, 1,000 keys into memory, moves them to `used_keys`, and serves creates from memory — **no per-write coordination**.
- **Pro:** O(1) create with no synchronous counter hit, no collision check at write time (uniqueness pre-guaranteed). Codes are non-sequential / non-guessable. Decouples generation from the hot create path entirely.
- **Con:** extra service to run; must guard against handing the same batch twice (the checkout is the only point needing a transaction); a server crash "leaks" its in-memory batch (acceptable — keyspace is huge). Needs a background job to keep `available_keys` topped up.

#### Option D — Snowflake-style distributed IDs + base62
- **Idea:** each node generates 64-bit IDs locally = `timestamp | machineId | sequence`; encode to base62.
- **Pro:** fully distributed, no coordination per write, roughly time-ordered.
- **Con:** 64-bit IDs encode to **~11 base62 chars** — *too long* for a "tiny" URL. You'd have to truncate (reintroducing collisions) or accept long codes. Snowflake shines for tweet/object IDs, less so when *short* is the product.

> **Where ZooKeeper / counter-ranges fit:** instead of one global counter, ZooKeeper (or any
> consensus store) hands each write server a **disjoint range** of the ID space, e.g. server-1 gets
> `[1–1M)`, server-2 gets `[1M–2M)`. Each server burns its range locally with no contention and
> asks ZK for a new range only when it runs low. This is the *same idea* as the KGS batch checkout
> — coordinate rarely (per range), not per write. It removes the single-counter SPOF/bottleneck.

#### Decision
> **I'll use the Key-Generation Service (Option C), with ZooKeeper-style range allocation as the
> mechanism for hand-out.** Justification:
> - It removes the single-counter bottleneck/SPOF (the main weakness of B) by coordinating *per
>   batch*, not per write.
> - It gives non-sequential, non-enumerable codes (fixes B's predictability) without the collision
>   retries of A.
> - It keeps codes at a fixed, controllable **7 chars** (fixes D's length problem).
> - Create becomes an in-memory O(1) operation.
>
> The cost I accept: an extra background service and the "leaked batch on crash" — negligible
> against a 3.5-trillion keyspace.

| Strategy | Length | Collisions | Predictable? | Per-write coordination |
|---|---|---|---|---|
| A. Hash-of-URL | fixed (7) | yes → retries | no | none (but retry round-trips) |
| B. Auto-inc base62 | minimal, grows | none | **yes** (enumerable) | **single counter SPOF** |
| C. **KGS (chosen)** | **fixed (7)** | **none** (pre-checked) | **no** | **none** (batch checkout) |
| D. Snowflake | ~11 (too long) | none (if untruncated) | semi (time-ordered) | none |

### 6.2 The read-heavy nature → caching & CDN (why this workload loves caching)

This is the heart of the design. Why is it *exceptionally* cache-friendly?
- **Immutable values:** a `code → longUrl` mapping never changes once created. So a cache entry is **never stale** for the lifetime of the link → no invalidation problem (the hardest part of caching, per [Topic 1](../prep/01-framework-and-building-blocks.md), simply doesn't arise here).
- **Tiny payloads:** a row is < 1KB → millions fit in RAM cheaply.
- **Heavy skew:** link popularity follows a power law — a small set of viral links serves a huge fraction of clicks. A modest cache captures most traffic.

**Caching strategy: cache-aside with Redis.**
1. Redirect service reads Redis; on miss, reads KV store and back-fills Redis.
2. TTL on cache entries (e.g. 24h) for LRU-style eviction of cold links; popular links stay hot.
3. Because entries are immutable, we never invalidate on update — only on **delete/expiry** (rare).

**CDN / edge for the redirect itself.**
> A 301 redirect *is itself a cacheable HTTP response*. For very popular codes, we can let the CDN
> cache the redirect at the edge keyed by URL path, so the user gets a redirect from a nearby PoP
> without touching our origin at all. This is what lets the redirect feel instant globally and
> shaves origin QPS dramatically.

Failure modes to name: **hot key** (one viral link hammering a single cache node → replicate hot keys / use client-side caching at edge), and **thundering herd** on a cold-but-suddenly-viral key (→ request coalescing / single-flight on cache miss).

### 6.3 Redirect path: 301 vs 302 (the analytics tradeoff)

| | **301 Moved Permanently** | **302 Found (temporary)** |
|---|---|---|
| Browser behavior | **caches** the redirect; future clicks skip our server | does **not** cache; every click comes back to us |
| Latency / load | lowest (browser/CDN serve it) | higher (every click is a request) |
| **Analytics** | **lose clicks** — cached redirects never hit us | **capture every click** |
| Flexibility | hard to change destination (cached) | can change/expire destination anytime |

> **I'd default to 302.** A URL shortener's value-add is largely the *click analytics* and the
> ability to change/expire a destination — a 301 throws both away by letting browsers cache the
> hop. The cost is more redirect traffic, but we just established this is a cache/CDN-friendly path
> we can scale cheaply. If a customer explicitly wants max performance and no analytics, a 301 is
> the right call for them — so make it a per-link option.

### 6.4 Custom aliases, expiration & the cleanup problem

**Custom aliases:** route through the *same* `code` key. On create, do a conditional
`put-if-absent(alias)`; conflict → 409. The KGS keyspace and the custom-alias space share the table,
so reserve a format (e.g. KGS codes are always exactly 7 random chars) to avoid a user grabbing a
code the KGS will later issue.

**Expiration / TTL:**
- Store `expiresAt`. On read, if expired → 404 (lazy check, cheap, keeps the hot path simple).
- **Cleanup problem:** lazily-expired rows still occupy storage forever. Options:
  - **DB-native TTL** (DynamoDB TTL / Cassandra TTL): the store reaps expired rows automatically in the background. **Preferred** — no app-side job, and it also frees the code.
  - If the store has no native TTL: a **background sweeper** scans by `expiresAt` (needs a secondary index on expiry) and deletes in batches during off-peak.
- **Reclaiming codes:** expired codes *could* be returned to the KGS `available_keys` pool, but I'd usually **not** recycle — collisions with still-cached or printed links aren't worth the small keyspace savings (we have trillions).

### 6.5 Click analytics — async via queue

> "Analytics must never be on the redirect's critical path." (Ties to the streaming/queue building
> block in [Topic 1](../prep/01-framework-and-building-blocks.md) and the dedicated streaming doc.)

- On each redirect, fire a **fire-and-forget event** to **Kafka**: `{code, timestamp, referrer, ip→geo, userAgent→device}`.
- Consumers aggregate (per-link counts, time buckets, top referrers, geo) into an **OLAP / columnar store** (e.g. ClickHouse / Druid / a warehouse).
- This decouples the spiky, high-volume click stream from both the redirect latency and the mapping store.
- **At-least-once delivery → need idempotency / dedupe** in aggregation if exact counts matter; for click stats, approximate counts (even HyperLogLog for uniques) are usually fine — a deliberate consistency relaxation.
- Real-time counters (e.g. "clicks in last hour") can be served from a separate Redis counter incremented by the consumer.

### 6.6 Scaling: sharding & killing the single-counter bottleneck

- **Shard the KV store by `code`** (hash partitioning). Point lookups by `code` go straight to one shard → no scatter/gather, even spread, no hot shards from sequential keys (codes are random). This is why a random/KGS code beats a sequential one for sharding too.
- **The single-counter bottleneck** (Option B's flaw) is solved exactly as in 6.1: the **KGS hands out disjoint key batches/ranges** via ZooKeeper, so no write ever contends on a shared counter. Coordination happens once per batch (~1000 creates), not once per create → effectively removed.
- **Redirect service** is stateless → scale horizontally behind the LB; cache + CDN absorb read growth.
- **Multi-region:** replicate the KV store + run KGS per region (each region draws from a disjoint slice of the keyspace so codes never collide across regions). Redirects served from nearest region/edge for the availability and latency targets.

### 6.7 Idempotency: same long URL → same short code?

> A genuine tradeoff worth raising unprompted.

- **Dedupe (same URL → same code):** saves storage, and a hash-based scheme (Option A) gives it for free. **But** it breaks per-user features: two users get the *same* link, so you can't give them separate analytics, separate expiry, or separate custom branding. And you'd need a `longUrl → code` reverse index (another lookup) to detect duplicates on every create.
- **No dedupe (each create → new code):** every shorten is independent; per-user analytics/expiry work cleanly; no reverse-index lookup. Costs a little storage (cheap).

> **I'd default to NOT deduping** — each shorten gets its own code. The product value (per-link
> analytics, independent expiry, branded links) outweighs the trivial storage savings. I'd only
> dedupe within a single user's own account if explicitly desired, via an optional reverse lookup.
> This is consistent with choosing the KGS (which doesn't dedupe) over hash-of-URL (which does).

---

## 7. Wrap-up (3 min)

**Bottlenecks / SPOFs remaining & mitigations:**
- *KGS availability:* if it's down, creates stall. Mitigate — write servers hold a local batch buffer, so they keep serving creates through a KGS outage; replicate KGS + its key tables.
- *Cache/region failure:* redirects fall back to KV store (higher latency but still correct); KV store is RF≥3 and multi-region.
- *Queue (Kafka) down:* analytics lag or drop, but **redirects are unaffected** by design — the right thing to degrade.
- *Hot key:* edge/CDN + hot-key replication absorb viral links.

**What I'd do with more time:** rate-limit/abuse + malware-URL scanning on create (shorteners are phishing magnets — Safe Browsing check); GDPR deletion of analytics; per-link 301/302 policy; code recycling policy; bloom filter on `code` existence to short-circuit 404s without a DB hit.

---

> ## What made this staff-level
> - **Reframed a "trivial" problem** around its two real hard parts (code generation + read-heavy
>   caching) instead of drawing a hash map.
> - **Compared four code-generation strategies on explicit axes** (length, collisions,
>   predictability, coordination) and *picked one with justification* — and tied the choice back to
>   sharding, idempotency, and the single-counter bottleneck so it all hung together.
> - **Derived the 7-char base62 number** from capacity math rather than guessing.
> - **Recognized the immutable-value insight** that makes this the rare cache with *no invalidation
>   problem*, and exploited it (Redis + CDN-cached redirects).
> - **Named the 301-vs-302 / analytics tradeoff** and chose a side for product reasons.
> - **Put analytics on a queue** and ensured the redirect path degrades gracefully when every
>   downstream (KGS, cache, queue) fails.

> ## Self-check (answer from memory before the mock)
> - [ ] Why is this workload read-heavy, and what *two* things does that buy you (cache + CDN)?
> - [ ] How many base62 chars for 100M URLs/yr over a decade, and how did you get there?
> - [ ] Name the 4 code-gen strategies and the one axis each one *loses* on.
> - [ ] Why does the KGS / ZooKeeper-range approach remove the single-counter bottleneck?
> - [ ] 301 vs 302 — which, and what do you sacrifice either way?
> - [ ] Why is this cache *never* stale, and what's the one event that forces invalidation?
> - [ ] Same long URL → same code: which do you pick and why?
> - [ ] What still works when the KGS / cache / Kafka each go down?
