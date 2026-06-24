# Design 24: Content Delivery Network (CDN)

> **Why this problem is different from every other in the canon:** in most designs the CDN is a
> *block you reach for* — "put the static stuff behind a CDN" — and you move on. Here the CDN **is the
> system**. That inversion is the whole interview: you are no longer a *customer* of caching, anycast,
> and origin offload, you are the *implementer* of them. The trap is to describe a CDN the way you'd
> use one ("edge caches static files, done") instead of deriving *why the cache hierarchy has exactly
> those tiers*, *how a user gets routed to the right edge*, *how a purge reaches 200 PoPs*, and *what
> happens when an edge or the origin dies*. The staff-level signal is to recognize in the first two
> minutes that a CDN is **three hard problems stacked**: (1) **request routing** (get the user to the
> nearest healthy edge), (2) **a cache hierarchy that funnels misses** so the origin survives planetary
> read scale, and (3) **invalidation/consistency across a globally distributed, eventually-consistent
> fleet**. This walkthrough runs the 7-step framework from `prep/01-framework-and-building-blocks.md`
> end to end and leans hard on the caching (`prep/06`) and networking (`prep/19`) deep dives, since a
> CDN is literally those two docs made into a product.

---

## Step 1 — Requirements (5 min)

I'll drive scope. "Design a CDN" is enormous; I'll carve it to the parts that decide the architecture.

**Functional (the verbs the system performs):**
- A **user requests an asset** (image, JS/CSS, video segment, API response) and gets it **from a nearby edge** with low latency.
- The edge **serves from cache** on a hit, or **pulls from origin** (through intermediate tiers) on a miss, then caches it.
- A **content owner publishes/updates** an asset and can **invalidate/purge** it globally.
- The system **routes each user to the nearest healthy edge** and **routes around failures** (dead edge, dead origin).
- The system **terminates TLS** at the edge and enforces **access control** (signed URLs, tokens) on protected content.
- The system **absorbs traffic spikes and DDoS** at the edge so they never reach origin.

**Explicitly out of scope** (say it, protect your time): the BGP/peering and physical-network buildout, the billing/metering pipeline, full edge-compute runtime internals, WAF rule authoring, and DNS-registrar mechanics. I'll cover **static + media delivery as the main case**, treat **dynamic content / edge functions** as a high-level deep dive, and contrast **push vs pull**.

**Non-functional (where the design is actually decided):**

| Dimension | Target | Consequence it forces |
|---|---|---|
| **Scale** | Planetary read scale: tens of **millions of req/s**, **tens of Tbps** egress, hundreds of PoPs | No origin can serve this directly → the hierarchy exists to *protect origin* |
| **Read:write ratio** | Pathologically read-heavy. Publishes are rare; reads are a firehose (**10,000:1+**) | The entire design optimizes the read path; writes (publish/purge) are a control plane |
| **Latency** | Edge response **p99 < 50 ms** for a cache hit (mostly RTT-bound); first-byte must beat fetching from origin | Edges must be *near users* (geography) and *warm* (high hit ratio) |
| **Availability** | **99.99%+** on the read path — a CDN outage blacks out everyone's sites at once | Read path is the most resilient thing we build; degrade by serving stale, never 5xx |
| **Consistency** | **Eventually consistent** by design. A purge propagates in seconds; bounded staleness is acceptable and expected | Lets us cache aggressively, coalesce, and tolerate per-PoP divergence |
| **Durability** | The CDN owns **no durable truth** — origin is the source of truth; edges hold disposable copies | Losing an edge's disk loses nothing; it just re-pulls. Huge simplification |
| **Offload** | **Origin offload ratio > 95%, target 99%+** | This is *the* KPI of a CDN; everything below is in service of it |

> **The sentence that frames everything:** "A CDN is a globally distributed, eventually-consistent
> read-through cache whose *only* durable state lives at origin. So I get to be aggressive — cache
> hard, serve stale under failure, coalesce misses — because the worst case of being wrong is bounded
> staleness, not lost data. My three jobs are: **route users to the nearest healthy edge, funnel cache
> misses through a hierarchy so the origin survives, and propagate invalidations across the fleet.** I
> measure success with one number: **origin offload ratio**, and I'll prove the targets with the
> estimate."

---

## Step 2 — Estimation (3 min)

Numbers that *justify decisions*, not vanity. (Methodology: `prep/02-estimation-and-napkin-math.md`.)

**Traffic & the offload argument**

| Quantity | Math | Result |
|---|---|---|
| Global requests/s (peak) | assume a large CDN: ~10M edge req/s avg, ~3× peak | **~30M req/s** |
| Avg object size (mixed: small assets + video segments) | — | ~100 KB (blended) |
| **Total edge egress** | 30M × 100 KB × 8 | **~24 Tbps** |
| **Origin egress at 95% hit ratio** | 24 Tbps × 5% | **~1.2 Tbps** |
| **Origin egress at 99% hit ratio** | 24 Tbps × 1% | **~240 Gbps** |
| **Origin egress at 99.9% hit ratio** | 24 Tbps × 0.1% | **~24 Gbps** |

> **This table is the entire interview.** 24 Tbps cannot come from any origin you can build. The
> hierarchy exists to drive that 24 Tbps down to something an origin fleet survives. **Each nine of
> hit ratio is a 10× reduction in origin load.** Going 95% → 99% isn't a 4% improvement — it's a **5×
> cut in origin bandwidth** (1.2 Tbps → 240 Gbps). That non-linearity is *why* we add the regional and
> shield tiers: each tier exists to buy another nine. **Offload ratio = `1 − (origin egress / total
> egress)`.** Quantify it; it is the read-side KPI (`prep/06` Part I, marginal value of cache).

**Edge footprint & per-PoP sizing**

| Quantity | Math | Result |
|---|---|---|
| PoPs | covering populated regions w/ <50ms RTT | **~200 PoPs** worldwide |
| Egress per PoP (avg) | 24 Tbps ÷ 200 | **~120 Gbps/PoP** (top PoPs 5–10×) |
| Hot working set (the content that earns caching) | Zipfian: ~the top **few %** of objects serve ~the bulk of requests | small relative to catalog |
| Disk per edge node | NVMe SSD | **~10–50 TB/node**, tens of nodes/PoP → **~PB-class per large PoP** |
| Cache memory (hottest tier) | RAM/NVMe for the very hot set | tens of GB–TB hot |

**Why the hierarchy, numerically (the funnel):** suppose each edge alone achieves an 80% hit ratio
(small disk, only the hottest content fits). The other 20% would all hammer origin → 24 Tbps × 20% =
**~4.8 Tbps at origin**. Insert a **regional/mid-tier** that aggregates dozens of edges and holds a
larger set: it absorbs ~90% of *those edge misses*. Insert an **origin shield** (one designated cache
per origin) that collapses all regional misses into a single stream and dedupes concurrent fetches.
Net effective hit ratio climbs to 99%+ → **240 Gbps at origin**. The tiers multiply, not add:
`origin_load = total × (1−h_edge) × (1−h_regional) × (1−h_shield)`. *That product is the reason the
hierarchy has exactly these layers* (`prep/06` Part A, where caches live).

**Cache-fill (origin pull) write rate** — publishes/purges are a control-plane trickle: maybe
thousands/s globally even for a huge customer base. The data plane is 30M req/s; the control plane is
~3 orders of magnitude smaller. *That contrast is why routing/config is a separate, smaller, strongly-
consistent-ish system from the read path.*

---

## Step 3 — API Design (3 min)

Two completely separate surfaces: the **data plane** (what end-users hit — implicit, it's just HTTP)
and the **control plane** (what content owners hit — publish, purge, config). No control-plane call
ever sits on the user's hot path.

```
# ---- DATA PLANE (the firehose): just HTTP/HTTPS to the edge ----
GET https://cdn.example.com/assets/app.a3f9c.js        # user → nearest edge (anycast/GeoDNS)
   Request headers honored:  If-None-Match, If-Modified-Since, Range, Accept-Encoding
   Response headers set/obeyed: Cache-Control, ETag, Age, X-Cache: HIT|MISS, Vary
   # there is NO custom API here — the contract IS HTTP caching semantics (prep/06 Part J)

# ---- CONTROL PLANE (the trickle): owner-facing, authenticated, rate-limited ----
PUT  /v1/zones/{zoneId}/config        { originUrl, ttlRules, signedUrlKeys, tlsCert, wafRules }
POST /v1/zones/{zoneId}/purge         { paths:["/img/logo.png"] | tags:["product-42"] | purgeAll:true }
   -> { purgeId, status:"PROPAGATING" }
GET  /v1/zones/{zoneId}/purge/{purgeId} -> { status:"DONE", popsAcked: 198/200 }
POST /v1/zones/{zoneId}/prefetch      { paths:[...] }   # push/pre-warm a hot release to edges
GET  /v1/zones/{zoneId}/analytics?from=&to= -> { hitRatio, egress, topPaths[] }
```

- The **data-plane contract is HTTP itself** — `Cache-Control`/`ETag`/`Range`/conditional requests. Saying "I don't invent an API for the read path; I obey HTTP caching semantics" is the staff move (`prep/06` Part J).
- **Purge by tag**, not just by path — a single product update may touch dozens of URLs; tagging lets one call invalidate a coherent set (the surrogate-key pattern).
- Purge returns **202 + a job id you poll** for `popsAcked`, not 200 — because **propagation across 200 PoPs is async and eventually consistent**. That status code signals you understand the consistency model.
- Control plane is authenticated + rate-limited at its own gateway (`prep/09`); a purge-all is expensive and abuse-prone, so it's throttled and billed.

---

## Step 4 — Data Model (5 min)

A CDN has almost no "data model" in the DB sense — its state is **(a) cached objects** (disposable, on
edge disk) and **(b) routing/config** (small, control-plane). The interesting modeling is the **cache
key** and the **object metadata**, because they decide hit ratio.

**The cached object (per edge, in the cache store):**

| Field | Notes |
|---|---|
| **Cache key** | The identity of a cacheable response — *the single most important design choice* (below) |
| Body bytes | The response payload, on NVMe/disk; hottest also in RAM |
| `ETag` / `Last-Modified` | For revalidation against the next tier / origin (conditional `If-None-Match` → 304) |
| `expiresAt` (from TTL) | When it goes **stale**; computed from `s-maxage`/`max-age` |
| `staleWhileRevalidateUntil` | Window in which we may serve stale + refresh in background |
| `surrogateKeys`/tags | For tag-based purge |
| LRU/LFU metadata | Recency/frequency for eviction |

**Cache key design (this is the staff detail):** the key is *not* just the URL. It's a deliberate
tuple, because the wrong key tanks hit ratio (over-fragmenting) or serves wrong content (over-merging):
```
cache_key = (scheme, host, normalized_path+query, Vary dimensions)
```
- **Normalize the path/query**: drop tracking params (`utm_*`, `fbclid`), sort remaining query params, lowercase host. Otherwise `?utm=a` and `?utm=b` are two cache entries for one object → hit-ratio collapse.
- **`Vary` carefully**: vary on `Accept-Encoding` (gzip vs br are genuinely different bytes) — yes. Vary on full `User-Agent` — *no*, it shatters the cache into millions of variants. Bucket UA into device class instead.
- **Strip cookies** from the key for static assets; a `Set-Cookie` on a cacheable response is a classic accidental-`private` bug.
- For protected content, the **auth token is validated but excluded from the key** (so all authorized users share one cache entry).

**Routing / config state (control plane, small, replicated):**

| Entity | Keyed by | Notes |
|---|---|---|
| **Zone / config** | `zoneId` (customer domain) | origin URL, TTL rules, signing keys, TLS cert, WAF — pushed to every PoP |
| **Edge health / topology** | `popId`, `nodeId` | liveness, load, capacity — feeds the routing decision (`prep/19`) |
| **Purge log** | `purgeId` | append-only stream of invalidations fanned out to all PoPs |

- **Access pattern → store choice** (`prep/01` step 4): config is read constantly by every edge but written rarely → distribute it as a **replicated, versioned config blob** (think a globally-replicated KV / config service) pushed to PoPs, with the edge holding a local copy and serving even if the control plane is unreachable. Routing health is a fast-changing, eventually-consistent gossip/health feed. Neither is on the user's latency path.

---

## Step 5 — High-Level Design (10 min)

Two planes that barely touch: a fat, geographically-distributed **data plane** (the read firehose) and
a thin **control plane** (publish/purge/config). The data plane is a **funnel**: user → edge → regional
→ shield → origin, where each tier exists to absorb the misses of the one in front of it.

```
                    ┌──────────────── CONTROL PLANE (the trickle) ────────────────┐
 content owner ─────► API GW ──► Config Svc ──► (replicated config) ──► every PoP  │
       │             (authn,                  Purge Svc ──► purge log ──► fan-out ──┤──► all edges
       │              rate-limit)                                                   │
       └─────────────────────────────────────────────────────────────────────────┘

 DATA PLANE (the firehose)

  user ──DNS/anycast──►  EDGE PoP (nearest, healthy)
  (GeoDNS or                │   • TLS terminate (1-RTT TLS 1.3 close to user)
   anycast VIP)             │   • check cache  ── HIT (X-Cache: HIT) ──► serve  (≈95%+)
                            │   • edge function / signed-URL check / WAF
                            ▼ MISS
                    REGIONAL / MID-TIER cache (aggregates dozens of edges)
                            │   • HIT ──► fill edge, serve
                            ▼ MISS
                    ORIGIN SHIELD (one designated cache per origin)
                            │   • request coalescing: collapse N concurrent misses → 1 origin fetch
                            │   • HIT ──► fill down, serve
                            ▼ MISS (the only requests origin ever sees)
                    ORIGIN (customer's server / object store — the source of truth)
```

Walk one **cache hit** out loud (the 95%+ case): *user resolves `cdn.example.com` → anycast/GeoDNS
routes the packet/lookup to the nearest healthy PoP → TCP+TLS handshake terminates at the edge (close,
so ~1 RTT, fast) → edge computes the cache key, finds a fresh object → returns it with `X-Cache: HIT`,
`Age: …`. **The origin is never contacted.** This is the path 95–99% of requests take, and it's almost
entirely RTT-bound — which is why edge proximity is the whole latency story (`prep/19` Part A).*

Walk one **cache miss** out loud (the long tail): *edge has no fresh copy → forwards to its regional
mid-tier → regional misses too → goes to the origin shield → shield finds **other edges already asked
for this**, so it **coalesces**: one in-flight fetch to origin, everyone else waits and shares the
result → origin returns the object once → it's cached at shield, regional, and edge on the way back
down (so the *next* request anywhere in that region is a hit). One origin fetch served thousands of
users.*

> **The one thing to never do:** let every edge talk directly to origin. With 200 PoPs and a thundering
> herd on a newly-popular object, that's 200+ simultaneous origin fetches for one object — a stampede
> that melts the origin (`prep/06` Part E). The **shield tier + request coalescing** is precisely the
> fix: it turns N concurrent misses into **one** origin request. The hierarchy isn't decoration; each
> tier buys a nine of offload and a layer of stampede protection.

---

## Step 6 — Deep Dives (15 min)

I'll propose the five that win this round: (A) routing users to the nearest healthy edge, (B) cache
mechanics + the origin pull + request coalescing, (C) invalidation/purge at global scale, (D) the
consistency model + serving stale, (E) security (signed URLs, DDoS, shielding). Then edge compute and
TLS briefly.

### 6A — Routing: getting the user to the nearest *healthy* edge (`prep/19` Parts B & E)

This is half the value of a CDN and the most under-discussed part. Two mechanisms, used together:

| Mechanism | How it works | Pro | Con |
|---|---|---|---|
| **Anycast** | The same IP is announced (BGP) from *every* PoP. The internet's routing **delivers the packet to the topologically nearest** PoP automatically. | No DNS step in the path; **instant failover** — withdraw the route from a dead PoP and traffic reroutes in seconds; absorbs DDoS by spreading it across all PoPs | "Nearest by BGP hops" ≠ "lowest latency"; mid-connection rerouting can break long-lived TCP (rare for HTTP) |
| **DNS-based (GeoDNS / latency-based)** | The authoritative DNS server returns **different edge IPs based on the resolver's location / measured latency**, and **stops handing out IPs of dead PoPs** (health-checked DNS). | Fine-grained, can do real latency/load steering and weighted spillover | Bounded by **DNS TTL** — some resolvers ignore it; failover is **minutes, not seconds** (`prep/19` Part B propagation gotcha) |

> **Say it crisply:** "I'd use **anycast for the edge VIPs** so failover is BGP-fast and DDoS spreads
> across PoPs, and layer **GeoDNS/latency-based DNS** for coarse geo-steering and load spillover. The
> two cover each other's weakness: DNS gives me steering knobs, anycast gives me sub-second failover
> that DNS's TTL can't (`prep/19` Parts B & E)."

**Health-aware routing is non-negotiable.** Routing must steer around *unhealthy* edges, not just
distant ones:
- **Continuous health checks** + load/capacity signals feed the routing decision. A PoP that's overloaded or failing is **drained**: anycast withdraws its BGP announcement (traffic reroutes to the next-nearest PoP); GeoDNS stops returning its IP.
- **Failover target:** when PoP A drains, its users land on the next-nearest PoP B. B's hit ratio dips briefly (cold for A's traffic) and it re-pulls through the hierarchy — graceful degradation, not an outage.
- **The DNS-as-SPOF caveat** (`prep/19` Part B): a DNS outage blacks out everything downstream even with healthy edges. Mitigate with a **secondary DNS provider**, sane TTLs, and anycast VIPs so the IP itself doesn't depend on a fresh DNS lookup.

### 6B — Cache mechanics, the origin pull, and request coalescing (`prep/06` Parts B, E, J)

**The cache contract is HTTP** — the edge obeys headers, it doesn't invent policy:
- `Cache-Control: public, max-age=…, s-maxage=…` — `s-maxage` is the **shared-cache (CDN) TTL**, separate from the browser's `max-age`. Edges key off `s-maxage`.
- `ETag` / `Last-Modified` → on expiry the edge **revalidates** with a conditional `If-None-Match`; origin replies **304 Not Modified** (no body) if unchanged → cheap freshness without re-transferring bytes.
- `no-store` (never cache — personalized), `no-cache` (cache but revalidate every time), `private` (browser only, never the edge).
- `Vary` — see cache-key design above.

**Pull vs Push CDN** (`prep/06` Part J) — pick per content class:

| | **Pull CDN** (lazy, the default) | **Push CDN** (proactive) |
|---|---|---|
| How | Edge fetches from origin on first miss, then caches | Owner pre-loads/prefetches content to edges before traffic |
| Use | Typical sites/APIs, the long tail — only requested content gets cached | Predictable hot set: a software release, a live event, tonight's popular video |
| Trade | First request *per edge* is a slow miss; risk of a synchronized first-miss stampede | You manage population + storage of cold content yourself |

> **The decision:** "Default to **pull** — it's self-managing, only popular content occupies cache. But
> for a **known surge** (new game release, a film premiere) I **pre-warm/prefetch** the hot set to edges
> *before* the spike, so the first million viewers don't all miss simultaneously and stampede origin.
> Netflix's Open Connect is the extreme of this — ship appliances into ISPs, preload tonight's titles
> overnight."

**Request coalescing / single-flight — the stampede fix** (`prep/06` Part E, #1 thundering herd): when
a hot object expires (or is first requested), thousands of concurrent edge requests miss at once. Naively
that's thousands of identical origin fetches.
- **At each tier**, the cache uses **single-flight**: the *first* miss for a key locks and fetches; concurrent requests for the same key **wait and share** the one in-flight result. One origin fetch satisfies the herd.
- **`stale-while-revalidate`** (`prep/06` Part J): when an object goes stale, serve the **stale copy immediately** while one background request refreshes it. Users never block on a revalidation; origin sees one refresh, not a herd. This is *the* edge-stampede killer.
- The **origin shield** amplifies this: all regional misses for an origin route through *one* shield cache, so coalescing happens at a single chokepoint and origin sees exactly one fetch per object per TTL — across the entire planet.

**Hit-ratio math (the KPI, restated as engineering):** from Step 2, 95% → 99% hit ratio is a **5× cut**
in origin egress (1.2 Tbps → 240 Gbps). Levers to push it up: **long TTLs** on immutable assets,
**versioned URLs** (infinitely cacheable), the **regional + shield tiers** (each buys a nine), and
**coalescing** (so a herd counts as one miss). Chasing the last nine costs disproportionate cache/disk —
spend it where origin protection actually matters.

### 6C — Invalidation & purge at global scale (`prep/06` Parts C & J)

"There are only two hard problems…" — this is the cache-invalidation one, at 200-PoP scale. Three tools,
in order of preference:

1. **TTL expiry (passive, the default).** Set a TTL; content self-expires. **No propagation needed** — each edge independently re-validates when its copy ages out. Cost: bounded staleness up to one TTL. This handles the overwhelming majority of content. *The art is choosing TTLs:* long for immutable assets, short for semi-dynamic, with `stale-while-revalidate` to hide the refresh.

2. **Versioned / content-hashed URLs (the *best* pattern — avoid purge entirely).** `app.a3f9c.js`, `/v2/logo.png`, `image.png?v=hash`. New content = **new URL = new cache key**; old URLs simply age out via LRU. **No invalidation event ever propagates** — publishing a new version *is* the invalidation, and it's atomic and instant because clients fetch the new URL from the updated HTML/manifest. This is why build tools fingerprint asset filenames, and it's the answer you lead with.

3. **Explicit purge (active, when you must).** For content at a *stable* URL that genuinely changed (a corrected article at the same path, a leaked file to pull). This is the **propagation-to-all-PoPs problem**:
   - A purge request enters the control plane → written to a **purge log/stream** → **fanned out to all 200 PoPs** (pub/sub from a central purge service, often hierarchical: central → regional → edge).
   - Propagation takes **seconds to low minutes**, is **eventually consistent**, and is rate-limited/billed (purge-all is expensive — it cold-starts the cache).
   - **Soft purge vs hard purge:** soft purge marks objects *stale* (so `stale-while-revalidate` can still serve them while refreshing — protects origin from the post-purge miss storm); hard purge evicts immediately (correct but risks a stampede on a hot object). Prefer **soft purge + tag-based** invalidation.
   - **Tag/surrogate-key purge:** tag related objects (`product-42`) so one call invalidates the whole coherent set without enumerating URLs.

> **Say this:** "I purge as little as possible. **Immutable + versioned URLs** mean publishing is the
> invalidation — no propagation problem at all. I reserve **explicit purge** for must-change-in-place
> content, do it as a **soft purge by tag** so it propagates as a stale-marking event over a pub/sub
> fan-out, and I accept it's **eventually consistent across PoPs in seconds**. That's the honest
> consistency model and it's fine for everything a CDN should be serving (`prep/06` Parts C & J)."

### 6D — Consistency: edges are eventually consistent; handling stale content

A CDN is, by definition, **eventually consistent**: 200 PoPs each hold independent copies that expire/
refresh/purge on their own schedule. Two edges can serve different versions of the same URL for the
length of a TTL or a purge-propagation window. **That's a feature, not a bug** — it's what buys the
availability and offload.

- **Bounded staleness** is the contract: the maximum a user sees stale content is one TTL (or the purge propagation delay). You *choose* that bound per content type via TTL.
- **Serve-stale-on-error (`stale-if-error`)** — if origin is **down**, the edge **serves the last-known-good stale copy** rather than 5xx. A CDN that serves slightly-old content during an origin outage is doing its most important job: shielding users from origin failure. This is the read path degrading gracefully, not failing.
- **Read-your-own-writes** is *not* offered for cacheable content — and that's correct. If a piece of content needs strict freshness (account balance, inventory count), it shouldn't be cached at the edge at all: mark it `Cache-Control: private, no-store` and let it pass through to origin. **Matching cacheability to freshness requirement is the discipline** (`prep/06` Part J, "when a CDN helps vs not").

> **Consistency stance:** "Edges are eventually consistent with bounded staleness, and I'd spend zero
> effort making them strongly consistent — that would defeat the point. Content that *needs* strong
> freshness is marked uncacheable and bypasses the edge. Everything else accepts a TTL-bounded stale
> window, and under origin failure I deliberately **serve stale** (`stale-if-error`) — availability over
> freshness, which is exactly PACELC's 'else, choose latency/availability' for a read cache (`prep/06`,
> `prep/01` Part B)."

### 6E — Security: signed URLs, DDoS absorption, origin shielding (`prep/17`, `prep/19`)

- **Signed URLs / tokens** for protected content (paid video, private files): the URL carries a **time-limited, signed token** (HMAC of path + expiry + optional client IP) that the edge validates *before* serving. The edge enforces access **without a round trip to origin** — the signing key is in the pushed zone config. Crucially, the **token is excluded from the cache key** so all authorized users share one cached object. Expiry + IP-binding limit link sharing.
- **DDoS absorption at the edge** — the CDN's enormous distributed capacity *is* a DDoS defense. **Anycast spreads a volumetric attack across all 200 PoPs** instead of concentrating it on one origin, so each PoP absorbs a fraction. Layer **rate limiting, connection limits, and WAF** at the edge; bot/L7 attacks are filtered *before* they ever reach origin. The origin only ever sees clean, coalesced misses.
- **Origin shielding (a security + offload role).** The shield tier is the *only* thing allowed to talk to origin; **origin firewalls to accept connections only from the CDN's shield IPs**. This hides origin from the internet entirely — you can't attack what you can't reach. It also caps origin's concurrent connections to a known, small number (the shields), so even a cache-busting attack (random query strings to force misses) is bounded by shield capacity, not by attacker volume. (Cache-busting attacks are mitigated by query normalization — see cache-key design.)
- **TLS termination at the edge** (`prep/19` Part D): terminate TLS at the PoP **close to the user**, so the expensive handshake (TLS 1.3 = 1 RTT, 0-RTT resumption) happens over a short RTT instead of a transcontinental one — a big latency win. Then **re-encrypt edge→origin** if the customer needs end-to-end encryption (regulated environments). The CDN manages certs centrally and pushes them to PoPs.

### Dynamic content & edge compute (high-level)

A CDN isn't only for static files. Two extensions:
- **Dynamic content acceleration:** even uncacheable responses benefit — the edge holds a **warm, pooled, TLS-terminated connection to origin** over an optimized backbone, so a dynamic request rides a fast edge↔origin path instead of the user's slow last mile. You cache nothing but still cut latency.
- **Edge functions / edge compute** (Cloudflare Workers, Lambda@Edge): run small logic *at the PoP* — A/B routing, auth/token checks, header rewrites, personalization, request normalization, even rendering. Runs **near the user, before the cache lookup or on miss**, avoiding an origin round trip for logic that used to require one. Trade-off: a constrained runtime (CPU/memory/time limits, no long-lived state) and a new place for bugs to live across 200 PoPs. High-level here; the point is *the edge is becoming a compute tier, not just a cache*.

### Storage & eviction at the edge (hot vs long-tail)

Each edge has **limited, fast disk** (NVMe), far smaller than the full catalog — so eviction policy
directly sets hit ratio:
- **LRU (or LFU/segmented-LRU) eviction** (`prep/06` Part D): evict the least-recently/frequently-used objects when the disk fills. Content is **Zipfian** — a small hot set serves most requests, a vast long tail is rarely touched. LRU naturally keeps the hot set resident.
- **Tiered caching for the long tail:** the **edge** holds the *hottest* set (small, fast); the **regional/shield** tier holds a *much larger* set (the warm long tail) on bigger disk. A long-tail object that misses at the edge often **hits at the regional tier** rather than going to origin — that's the regional tier earning its nine of offload. Hot content lives everywhere; long-tail content lives once per region.
- **Admission control** (e.g., don't cache an object until it's been requested twice) avoids polluting the cache with one-hit-wonders that would evict genuinely hot content.

---

## Step 7 — Wrap-Up (3 min)

**Bottlenecks that remain:**
- **Origin offload ratio is THE metric.** At 24 Tbps total, the difference between 99% and 99.9% hit ratio is 240 Gbps vs 24 Gbps at origin. Levers: long TTLs, versioned URLs, the regional+shield tiers, coalescing, `stale-while-revalidate`.
- **Cache-busting / cold-cache events** — a code push that re-fingerprints every asset, or a purge-all, cold-starts caches and briefly spikes origin. Mitigate with **soft purge**, **staggered TTLs** (avoid synchronized expiry → cache avalanche, `prep/06` Part E #3), and pre-warming.
- **Top PoPs are hot** — traffic is geographically skewed; the busiest PoPs run 5–10× average and need the most capacity headroom.

**Failure modes (name them before asked):**
- **An edge PoP goes down** → **reroute**: anycast withdraws the BGP announcement (sub-second) / GeoDNS stops returning its IP; users land on the next-nearest PoP, which re-pulls through the hierarchy. Brief hit-ratio dip, no outage. *This is why anycast beats DNS for failover — TTL can't fail over in seconds.*
- **Origin goes down** → **serve stale** (`stale-if-error`): edges keep serving the last-known-good copy for the hot set; only true cold-cache long-tail requests 404. The read path survives an origin outage by design — its single most valuable behavior.
- **A hot object expires everywhere at once** (synchronized TTL) → **cache avalanche/stampede**: mitigated by jittered TTLs, request coalescing, and the shield collapsing the herd into one origin fetch.
- **Control plane (purge/config) unreachable** → edges **keep serving from their last pushed config + cached content**; purges queue and apply when it recovers. The data plane is independent of the control plane on purpose — a control-plane outage doesn't black out delivery.
- **DNS outage** → blacks out routing even with healthy edges → secondary DNS provider + anycast VIPs.

**Single points of failure & redundancy:** no single PoP, origin, or DNS provider can be a SPOF on the
read path. Anycast (multi-PoP) + secondary DNS removes the routing SPOF; the **origin itself** is the
closest thing to a real SPOF — mitigated by the shield serving stale during origin outages and by
customers running **multi-region origins**. The purge fan-out is made resilient by being a replicated
log, not a single broadcaster.

**With more time I'd cover:** multi-CDN strategy (steer across providers for resilience/cost), per-PoP
capacity planning and traffic engineering, the billing/metering pipeline, WAF/bot-management depth,
HTTP/3-QUIC at the edge (`prep/19` Part E) for faster connection setup on lossy mobile links, and
brotli/codec optimization to cut egress.

---

### What made this staff-level
- **Reframed the CDN as the system, not a block.** Named the three stacked hard problems — routing, the miss-funnel hierarchy, and global invalidation — instead of "edge caches static files."
- **Led with and quantified the binding metric.** Origin offload ratio, with the *non-linear* math: each nine of hit ratio is a 10× origin reduction, and that non-linearity is *why* the regional and shield tiers exist (`origin_load = total × ∏(1−h_tier)`).
- **Derived the hierarchy from the funnel**, not by recitation — each tier buys a nine of offload *and* a layer of stampede protection (coalescing + shield), with the product-of-misses math.
- **Routing done right:** anycast *and* GeoDNS together, with the explicit reason (anycast = sub-second failover DNS TTL can't give; DNS = steering knobs), plus health-aware draining (`prep/19`).
- **Owned the consistency model:** eventually consistent, bounded-stale, **serve-stale-on-error** as a deliberate availability choice, and "uncacheable content bypasses the edge" as the discipline — instead of pretending edges are consistent.
- **Invalidation answered in priority order:** TTL → versioned URLs (avoid purge entirely) → soft tag-purge over a pub/sub fan-out, with the propagation-to-all-PoPs problem named honestly.
- **Tied every block back to its deep-dive doc** (`prep/06` for coalescing/stale-while-revalidate/eviction/headers, `prep/19` for anycast/DNS/TLS) rather than reinventing them, and **named failure modes + SPOFs unprompted**.

### Self-check (answer from memory before the mock)
- [ ] Why does the cache hierarchy have *exactly* edge → regional → shield → origin? (the miss-funnel + product-of-misses math, and what each tier buys)
- [ ] Recite the offload math: 24 Tbps total at 95% vs 99% vs 99.9% — what's at origin, and why is it non-linear?
- [ ] Anycast vs GeoDNS: how each routes, why you use both, and which one fails over in seconds (and why)?
- [ ] How does request coalescing + the origin shield prevent a thundering herd on a newly-hot object?
- [ ] Design a cache key — what to normalize, what to `Vary` on, what *not* to (and why UA/cookies are traps)?
- [ ] Invalidation in priority order: TTL vs versioned URLs vs explicit purge — when each, and what's the propagation problem?
- [ ] What's the consistency model, and what does an edge do when origin is *down*? (serve stale / `stale-if-error`)
- [ ] How do signed URLs work at the edge without an origin round trip, and why is the token excluded from the cache key?
- [ ] Edge down → what happens? Origin down → what happens? DNS down → what happens?
