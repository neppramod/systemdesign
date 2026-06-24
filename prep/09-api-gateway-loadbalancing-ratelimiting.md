# Topic 9: The Edge — API Design, Gateways, Load Balancing & Rate Limiting

> **Why this topic matters:** Every request in your design crosses *the edge* before it touches a
> single service. It's the layer the interviewer reaches for when they want to test whether you
> think about real systems or only happy-path boxes. "How do you stop one abusive client from
> taking down the fleet?" "How does traffic find a healthy server?" "Why gRPC here and REST there?"
> These are edge questions. They also include one of the most common standalone prompts in the
> circuit — **design a rate limiter** — so this doc ends with a full worked walkthrough of it.
> The unifying idea: the edge is where you *shape, protect, and route* traffic before it can hurt you.

---

## Part A — API Design Choices

The interviewer rarely asks "REST or gRPC?" directly. They ask "how do clients talk to this?" and
judge whether your answer *fits the access pattern*. Pick the protocol from the traffic shape, not by reflex.

### The menu, and what each is actually for

- **REST / HTTP+JSON** — request/response over HTTP, resource-oriented, stateless. The default for
  public/external APIs and CRUD. Human-readable, cacheable (HTTP caching is free), every tool speaks it.
  Cost: verbose payloads, over-/under-fetching, no streaming, weak typing.
- **gRPC** — HTTP/2 + protobuf, binary, strongly typed via IDL, code-gen for clients. The default for
  **internal service-to-service** calls. Multiplexed streams, bidirectional streaming, ~5–10× smaller
  payloads, low latency. Cost: not browser-native (needs gRPC-Web + proxy), binary is harder to debug,
  schema/versioning discipline required.
- **GraphQL** — single endpoint, client specifies exactly the fields it wants. Solves over-/under-fetching
  for **rich client UIs** with many heterogeneous screens (mobile + web hitting different field sets).
  Cost: HTTP caching breaks (everything is `POST /graphql`), the **N+1 query** problem, complex queries
  can DoS your backend (you must add depth/complexity limits), harder to rate-limit by endpoint.
- **WebSocket** — full-duplex, persistent TCP connection. For **bidirectional, low-latency** comms:
  chat, multiplayer, collaborative editing, live trading. Cost: stateful (sticky connections complicate
  LB and scaling), connection management, no built-in request/response semantics.
- **SSE (Server-Sent Events)** — server→client stream over a single long-lived HTTP response.
  For **one-way server push**: notifications, live feeds, LLM token streaming, progress bars.
  Simpler than WebSocket (plain HTTP, auto-reconnect built in), but **unidirectional** and limited by
  browser per-host connection caps on HTTP/1.1 (HTTP/2 fixes this).
- **Long-polling** — client makes a request, server holds it open until data or timeout, client
  re-requests. The **fallback** when you need push but can't hold persistent connections (legacy
  proxies, simple infra). Cost: latency, wasted connections, header overhead per cycle.

### How to pick in the room (say this)

> "Internal hop, high QPS, latency-sensitive → **gRPC**. Public API, third-party developers, needs
> caching → **REST**. Rich client with wildly different data needs per screen → consider **GraphQL** but
> only with complexity limits. Server needs to *push* — one-way is **SSE**, two-way interactive is
> **WebSocket**, and **long-polling** is the degrade path."

A clean default for a typical design: **REST/gRPC at the edge, gRPC between services, SSE for streaming, WebSocket only when genuinely bidirectional.**

| Protocol | Direction | Transport | Best for | Main cost |
|---|---|---|---|---|
| REST | req/resp | HTTP/1.1 | Public CRUD APIs | Verbose, over/under-fetch |
| gRPC | req/resp + streaming | HTTP/2 + protobuf | Internal service-to-service | Not browser-native, binary debug |
| GraphQL | req/resp | HTTP | Rich UIs, varied field needs | Caching breaks, N+1, query DoS |
| WebSocket | full-duplex | TCP (upgraded) | Chat, games, collab | Stateful, sticky LB |
| SSE | server→client | HTTP | Notifications, token streaming | One-way only |
| Long-poll | req/resp (held) | HTTP | Push fallback | Latency, wasted conns |

### Idempotency of HTTP methods (know this cold)

- **Idempotent**: same request N times = same effect as once. `GET`, `PUT`, `DELETE`, `HEAD`, `OPTIONS`.
- **Not idempotent**: `POST` (creates a new resource each call), `PATCH` (depends on semantics).
- **Why it matters**: retries. The network is unreliable; clients and gateways retry. An idempotent
  operation is **safe to retry**. A `POST` is not — so for "create payment", you pass an
  **idempotency key** (client-generated UUID) the server dedupes on, turning an unsafe POST into a
  safe-to-retry one. This is *the* mechanism that makes at-least-once delivery survivable.

> **In the room:** "I'll make the create-order endpoint accept an `Idempotency-Key` header. The
> service stores `key → result` for 24h; a retry with the same key returns the stored result instead of
> double-charging. This is how I reconcile at-least-once retries with exactly-once *effects*."

### Pagination — cursor vs offset (and why cursor)

- **Offset/limit** (`?offset=40&limit=20`): jump to page N. Simple, allows random page access.
  **Breaks under inserts/deletes** — if a row is inserted at the front between page loads, page 3 repeats
  rows from page 2 (the "drifting page" problem). Also `OFFSET 100000` makes the DB scan and discard
  100k rows — O(offset) cost, slow deep into the set.
- **Cursor/keyset** (`?cursor=<opaque>&limit=20`): "give me 20 items after *this* item." The cursor
  encodes the last seen sort key (e.g. `(created_at, id)`). Query is `WHERE (created_at, id) < (?, ?)
  ORDER BY ... LIMIT 20` — uses the index, **O(limit)** regardless of depth, and is **stable under
  inserts**. Cost: no random page jumps, sort key must be stable + unique.

> "For an infinite feed I use **cursor pagination** — it's stable when new items are inserted and stays
> fast at any depth because it's an index seek, not a scan. Offset only earns its place when users need
> to jump to an arbitrary page number, like a paginated admin table."

### Versioning strategies

- **URI path** (`/v1/users`) — most visible, easy to route at the gateway, easy to cache. Most common public choice.
- **Header** (`Accept: application/vnd.api.v2+json`) — keeps URLs clean, content-negotiation purist, less discoverable.
- **Query param** (`?version=2`) — easy but pollutes caching and logs.

Pick **URI versioning** for public APIs (clarity + gateway routing). Internally, prefer **additive,
backward-compatible** changes (protobuf shines here: add fields, never reuse tag numbers) so you rarely
bump a major version at all. Say: "I version only on breaking changes; everything else is additive."

### Error & retry semantics

- Use HTTP status codes meaningfully: `4xx` = client's fault, don't retry (except `429`/`408`);
  `5xx` = server's fault, *maybe* retry. `429 Too Many Requests` and `503 Service Unavailable` → retry **with backoff**.
- **Retries need three things**: exponential backoff, **jitter** (so retries don't synchronize into a
  thundering herd), and a **retry budget / cap** (so a struggling service isn't buried by retries —
  this is a top cause of retry-storm outages). Pair with **circuit breakers**.
- Only retry **idempotent** operations (or POSTs guarded by an idempotency key). Surface a machine-readable
  error body (`{ "code": "RATE_LIMITED", "retryAfter": 30 }`) so clients act correctly.

---

## Part B — Load Balancing

### L4 vs L7 (the first distinction to draw)

- **L4 (transport)** — routes on IP/port. Forwards TCP/UDP without inspecting payload. **Fast, cheap,
  protocol-agnostic**, preserves connections. Can't make content-based decisions (can't route `/api` vs
  `/images`, can't do per-path rate limiting). Examples: AWS NLB, IPVS, classic hardware LBs.
- **L7 (application)** — terminates the connection, reads HTTP. Can route by **path/header/cookie**, do
  TLS termination, retries, header rewriting, sticky sessions, request-level observability. Cost: more CPU
  (it parses requests), it's a TCP endpoint so it sees every byte. Examples: AWS ALB, Nginx, Envoy, HAProxy.

> "L4 for raw throughput and non-HTTP traffic; **L7 when I need content-based routing, TLS termination,
> or per-request logic** — which is almost always true at the API edge. A common pattern is L4 in front
> (DSR, huge throughput) feeding an L7 tier."

### Algorithms

- **Round robin** — rotate through backends. Simple, fine when requests are uniform and servers identical.
  Ignores actual load → a slow request on one box still gets the next turn.
- **Weighted round robin** — assign weights for heterogeneous hardware. Static; doesn't react to real-time load.
- **Least connections** — send to the backend with fewest active connections. Adapts to uneven request
  durations. But it requires **global state** (every LB must know every backend's count), and with many
  LB instances each has only its *own* view → they can stampede the same "idle-looking" server.
- **Consistent hashing / sticky sessions** — hash a key (client IP, session ID, cache key) to a backend so
  the *same key always lands on the same server*. Essential for **session affinity** and **cache locality**
  (e.g. routing a user to the node that holds their warm cache). Consistent hashing minimizes reshuffling
  when nodes are added/removed. Cost: hot keys → hot servers; stickiness fights even load distribution.
- **Power of two choices (P2C)** — pick **two backends at random, send to the less loaded of the two.**
  Nearly as good as global least-connections but with almost no coordination.

### Why power-of-two-choices beats pure least-connections

Pure least-connections is theoretically optimal *only with a single LB that has perfect global state*.
In reality you run **many LB instances**, each with a stale/partial view. They all independently see the
same server as "least loaded" and **herd onto it** — the very server they thought was idle gets crushed,
oscillation ensues. P2C breaks the herd: by sampling two random servers and choosing the better, the
probability that the *same* unlucky server is repeatedly chosen drops exponentially. Mathematically, the
maximum load goes from O(log n / log log n) under pure random to **O(log log n)** under two choices — an
exponential improvement in worst-case imbalance, achieved with **zero global coordination**. That's why
modern LBs/meshes (Envoy, Finagle) default to P2C (often "P2C + least-request").

> "I'd use **power-of-two-choices**: each LB picks two backends at random and routes to the less busy one.
> It gets you near-optimal balancing without the herding failure mode of naive least-connections across a
> fleet of load balancers."

### Health checks

- **Active**: LB probes `/healthz` on an interval; failing N in a row → mark **unhealthy**, stop routing.
- **Passive**: observe live traffic; a backend returning errors/timeouts → eject (outlier detection).
- Distinguish **liveness** (process up) from **readiness** (ready to serve — deps connected, caches warm).
  Route only to *ready* instances; a box can be alive but not ready during startup.
- Use both: active gives fast detection of dead boxes; passive catches partial failures active probes miss.

### Connection draining (graceful shutdown)

On deploy or scale-in you don't kill a backend mid-request. **Drain**: stop sending it *new* connections,
let in-flight requests finish (up to a timeout), then terminate. Without draining, every deploy throws
errors at users. Pair with **`SIGTERM` → stop accepting → finish in-flight → exit** in the app, and a
readiness probe that flips to "not ready" so the LB deregisters it first.

---

## Part C — Global Traffic Management

Load balancing *within* a region is one thing; getting users to the *right region* is another.

- **DNS-based LB** — return different A/AAAA records per client. Cheap, works everywhere, but **DNS TTL
  caching** means failover is slow (clients cache the old IP for the TTL — and many resolvers ignore low
  TTLs). Coarse-grained; good for steering, bad for instant failover.
- **GeoDNS** — DNS answers depend on the resolver's geographic location → send EU users to the EU region.
  Caveat: it sees the *resolver's* location, not the user's (mitigated by EDNS Client Subnet).
- **Anycast** — the *same* IP is advertised from many locations via BGP; the network routes each user to
  the **topologically nearest** PoP. Used by CDNs and DNS providers (and increasingly for L4 LB). Failover
  is **fast** (BGP reroutes, no DNS TTL to wait on) and it's inherently DDoS-absorbing (traffic spreads
  across PoPs). Cost: routing is controlled by BGP, not you; mid-connection re-routing can break TCP (fine for UDP/QUIC).

### Global LB vs regional LB

- **Global LB** decides *which region/PoP* a request enters (GeoDNS, anycast, a global L7 like Cloudflare/
  GCLB). Optimizes for **proximity, region health, and disaster failover**.
- **Regional LB** decides *which instance within a region* serves it (ALB/NLB/Envoy). Optimizes for
  **per-instance balancing, health, draining**.

> "It's two tiers: a **global** layer (anycast/GeoDNS) routes you to the nearest healthy region; a
> **regional** layer (L7 LB) spreads you across instances. Failover differs — region loss is handled
> globally via DNS/anycast and health-checked steering; instance loss is handled regionally in seconds."

---

## Part D — API Gateway

The gateway is the **single front door**. It centralizes cross-cutting concerns so individual services don't each reimplement them.

### Responsibilities

- **AuthN/AuthZ** — validate JWT/API key/OAuth token once at the edge; pass a trusted identity downstream. Services don't re-auth.
- **Rate limiting & quotas** — per-user/IP/API-key throttling (Part E). The edge is the cheapest place to reject abuse.
- **Routing** — path/host/header → backend service. The service-discovery-aware dispatcher.
- **Request aggregation** — fan one client call out to several services and compose the response (a
  lightweight BFF pattern), saving mobile clients multiple round trips.
- **Protocol translation** — REST↔gRPC, terminate WebSocket, translate HTTP/1.1↔HTTP/2. Lets browsers talk REST while the backend speaks gRPC.
- **TLS termination** — decrypt once at the edge (Part F).
- **Observability** — central place for request logging, tracing (inject/propagate trace IDs), metrics, and audit.
- **Response caching, request/response transformation, payload validation, WAF.**

> **Caution:** Don't let the gateway become a monolith of business logic. It owns *cross-cutting* concerns
> only. Business rules belong in services. A "smart gateway, dumb services" topology is an anti-pattern.

### Gateway vs Service Mesh (a frequent staff-level distinction)

- **API Gateway = north-south traffic** — *external* clients → your system. One centralized edge hop.
- **Service Mesh = east-west traffic** — *service-to-service* calls *inside* the system. Implemented as a
  **sidecar proxy** (e.g. Envoy) next to every service instance, with a control plane (Istio/Linkerd).
- The mesh handles **mTLS between services, retries, circuit breaking, P2C load balancing, and
  per-hop observability** — *decentralized*, one proxy per pod, no single chokepoint.

> "The gateway is the front door for outside traffic; the mesh is the nervous system for internal traffic.
> Gateway = one centralized hop handling auth and external rate limiting. Mesh = a sidecar on every service
> doing mTLS, retries, and fine-grained internal LB. They compose — gateway at the edge, mesh behind it."

---

## Part E — Rate Limiting (in depth)

This is the highest-leverage part of the topic. Know the five algorithms cold, including their **burst
behavior, memory cost, and accuracy** — that's the table interviewers want to see.

### Why rate-limit at all

Protect against abuse/DDoS, enforce fair use and billing tiers, prevent one tenant starving others
(noisy neighbor), and protect downstream from overload. It is **traffic shaping**, applied per-dimension.

### The five algorithms

**1. Fixed window counter** — count requests per fixed clock window (e.g. per minute). Increment a counter
keyed by `(user, minute)`; reject when it exceeds the limit; reset at the boundary.
- *Burst:* **bad** — allows 2× at the boundary: 100 requests at 00:59 and 100 at 01:00 = 200 in 2 seconds.
- *Memory:* tiny (one counter per key). *Accuracy:* low (the boundary spike).

**2. Sliding window log** — store a **timestamp for every request** in a sorted set; on each request, drop
timestamps older than the window and count what remains.
- *Burst:* none — perfectly accurate, true rolling window.
- *Memory:* **expensive** — O(N) per key, stores every request timestamp. *Accuracy:* exact.

**3. Sliding window counter** — the practical compromise. Keep counters for the current and previous fixed
window, and **weight the previous window by how much of it overlaps** the rolling window:
`count ≈ curr + prev × (overlap fraction)`.
- *Burst:* smooths the fixed-window boundary spike. *Memory:* tiny (two counters). *Accuracy:* very good
  approximation (assumes uniform distribution within a window) — **this is what most production limiters use** (Cloudflare's classic approach).

**4. Token bucket** — a bucket holds up to `B` tokens, refilled at rate `r` tokens/sec. Each request consumes
a token; empty bucket → reject (or wait).
- *Burst:* **allows bursts up to the bucket size `B`** while enforcing the long-run average rate `r`. This is
  usually *desirable* — clients can briefly burst. *Memory:* tiny (token count + last-refill timestamp).
  *Accuracy:* enforces average rate with controlled burst. The most common API limiter (AWS, Stripe style).

**5. Leaky bucket** — requests enter a queue (the bucket); they **drain at a constant rate**. Overflow →
reject. Like a funnel.
- *Burst:* **smooths output to a constant rate** — no bursts pass through (good for protecting a downstream
  that needs steady load). *Memory:* the queue. *Accuracy:* enforces a strict constant outflow; adds latency
  (requests wait in the queue).

> **Token bucket vs leaky bucket** — token bucket allows bursts (good for user-facing APIs where bursting is
> fine), leaky bucket enforces a smooth constant rate (good for protecting a fragile downstream). Say which
> property you want and pick accordingly.

### Comparison table

| Algorithm | Burst behavior | Memory / key | Accuracy | Use when |
|---|---|---|---|---|
| Fixed window | Allows 2× at boundary | O(1), tiny | Low | Simplest, rough limits OK |
| Sliding window log | None (exact) | O(N), high | Exact | Need precision, low volume |
| Sliding window counter | Smoothed | O(1), tiny | Very good | **Default production choice** |
| Token bucket | Allows burst up to B | O(1), tiny | Avg rate + burst | **User-facing APIs, bursty OK** |
| Leaky bucket | None — constant outflow | O(queue) | Strict constant rate | Protect fragile downstream |

### Dimensions — what's the key?

Rate limits apply per **key**, and the choice of key is a design decision:
- **Per API key / per user** — fairness and billing tiers (free vs paid). The primary dimension for SaaS.
- **Per IP** — anonymous traffic and crude DDoS defense. Weak alone (NAT/CGNAT means many users share an IP;
  attackers rotate IPs) — use as a coarse layer, not the only one.
- **Per endpoint / per resource** — protect expensive operations (`POST /search` cheaper limit than `GET /healthz`).
- **Global** — protect the whole system regardless of who's calling (a backstop).

Usually you run **several limiters layered**: global backstop + per-IP + per-API-key + per-expensive-endpoint.

### Distributed rate limiting (the hard part)

A single in-memory counter works on one node. With a fleet of gateway instances behind an LB, each sees
only its share of traffic → you'd let through `N × limit`. You need **shared state**, almost always **Redis**.

**The race condition.** The naive `GET count → check → INCR` is a **read-modify-write race**: two instances
both read 99 under a limit of 100, both think they're under, both increment → 101, limit violated. The
window of vulnerability is between the read and the write.

**Fixes:**
- **`INCR` is atomic**, and `INCR` returns the new value — so `INCR` then check the *returned* value, and on
  the first increment set the TTL. For a simple fixed window: `INCR key; EXPIRE key 60 (if new)`. Still has
  the fixed-window boundary problem, but no race.
- **Atomic Lua script.** When the logic is multi-step (refill a token bucket: read tokens + timestamp,
  compute new tokens by elapsed time, check, decrement, write back), wrap it in a **Lua script via `EVAL`**.
  Redis executes the script **atomically** (single-threaded, no interleaving) — read-modify-write becomes one
  indivisible operation. This is the standard production technique for token-bucket-in-Redis.
- **Sliding window log with a sorted set.** Use a `ZSET` keyed by user, member = request id, score = timestamp:
  ```
  ZREMRANGEBYSCORE key 0 (now - window)   -- drop old entries
  ZADD key now <reqid>                    -- add this request
  ZCARD key                               -- count in window
  EXPIRE key window
  ```
  Wrap these in a Lua script (or `MULTI`) for atomicity. Exact sliding window, distributed — at the cost of
  storing every timestamp (memory) and the per-request ZSET ops.

**Tradeoffs of centralized Redis:** adds a network hop (latency) and a dependency (Redis down → fail open or
fail closed?). Mitigations:
- **Local + central two-tier:** each node keeps a local token bucket for the *fast path* and periodically
  syncs/reconciles with Redis, accepting slight over-admission for lower latency.
- **Fail open** (allow traffic if Redis is unreachable) for availability, or **fail closed** for protection —
  state which, and why, based on whether the limiter is for billing fairness (fail open) or overload
  protection (fail closed).
- Sharding/cluster Redis by key for throughput; the limiter key is naturally shardable by user/IP.

> "I'd do a **token bucket in Redis using an atomic Lua script** so the read-refill-decrement is race-free.
> For exact sliding windows I'd use a sorted set of timestamps. To cut the per-request hop I'd add a local
> bucket per node that reconciles with Redis, accepting a little over-admission. And I'd decide **fail-open vs
> fail-closed** explicitly based on what the limit is protecting."

### Where to enforce — edge vs service

- **At the edge/gateway** — reject abuse before it consumes any backend resource. Cheapest, protects the
  whole system, the default home for per-API-key quotas. But the gateway needs the shared counter.
- **At the service** — for limits that depend on business context the gateway can't see (per-tenant
  resource cost, per-feature limits). Finer-grained but the request already paid the cost of reaching the service.
- Real systems do **both**: coarse limits at the edge, fine business-aware limits in services.

### Returning 429 properly

On reject, return **`429 Too Many Requests`** with a **`Retry-After`** header (seconds or HTTP-date) so the
client backs off intelligently instead of hammering. Also send `X-RateLimit-Limit`, `X-RateLimit-Remaining`,
`X-RateLimit-Reset` so well-behaved clients self-throttle. A limiter that returns 429 without `Retry-After`
just invites a retry storm.

---

## Part F — Load Shedding & Admission Control (≠ rate limiting)

This distinction is a strong staff-level signal.

- **Rate limiting** is **per-client, configured ahead of time, fairness/quota-driven.** "User X gets 100 rps."
  It's about *who* and *how much they're entitled to*, regardless of current system health.
- **Load shedding** is **system-health-driven, reactive.** When the server is *actually* overloaded (queue
  depth, CPU, p99 latency, thread pool saturation crossing a threshold), it **drops the lowest-priority
  requests** to keep the system alive and protect the requests it *does* accept. It doesn't care who you are
  — it cares whether accepting your request will tip the system over.
- **Admission control** is the gate: decide *at intake* whether to admit a request at all, often using
  **priority/criticality tiers** (shed health-check pings and batch traffic before user-facing reads;
  Google's "criticality" buckets) and techniques like **LIFO queues** or **CoDel-style** queue-latency
  shedding (drop requests that have already waited too long — they're probably abandoned).

> "Rate limiting enforces *contracts* — what each client is entitled to. Load shedding is *self-preservation*
> — under real overload I drop low-priority work so the system survives and high-priority requests still get
> served. You need both: limits stop abuse; shedding handles the surprise overload that slipped past the limits."

Related guardrails to name: **circuit breakers** (stop calling a failing downstream), **bulkheads** (isolate
resource pools so one tenant/feature can't drain everything), **backpressure** (signal upstream to slow down),
and **concurrency limits** (cap in-flight requests — often more robust than rps limits because it directly
tracks resource use; see Netflix's adaptive concurrency limits).

---

## Part G — Reverse Proxy, TLS & mTLS

- **Reverse proxy** — sits in front of servers, accepts client connections, forwards to backends. The L7 LB,
  the gateway, and the CDN edge are all reverse proxies. It enables TLS termination, caching, compression,
  routing, and hides backend topology. (A *forward* proxy fronts the *client* side, e.g. a corporate egress proxy.)
- **TLS termination** — decrypt HTTPS **once at the edge** (LB/gateway) so backends handle plaintext HTTP and
  don't each pay the TLS handshake cost or manage certs. Centralizes cert rotation. Cost: traffic is plaintext
  *inside* your network past the edge — fine within a trusted VPC, **not** fine for zero-trust.
- **TLS re-encryption / passthrough** — when you can't trust the internal network, either re-encrypt
  edge→backend, or pass TLS straight through to the backend (L4) so the LB never sees plaintext.
- **mTLS (mutual TLS)** — *both* sides present certificates; each verifies the other. The foundation of
  **zero-trust service-to-service** auth: every service proves its identity with a cert, so a compromised
  service can't impersonate another. This is exactly what a **service mesh** automates — the sidecar handles
  cert issuance, rotation (short-lived certs from a CA like SPIFFE/SPIRE), and mTLS handshakes transparently,
  so app code stays unaware.

> "External edge: terminate TLS at the gateway for cert centralization. Internal east-west: **mTLS via the
> mesh sidecars** so every hop is mutually authenticated and encrypted — services trust *identity*, not the
> network. That's zero-trust internally."

---

## Part H — Worked Mini-Walkthrough: "Design a Rate Limiter"

A real standalone interview prompt. Run the framework on it.

**1. Requirements.**
- Functional: given a request with an identity (API key/user/IP), decide allow/deny; enforce a configurable
  limit (e.g. 100 rps per key) across **distributed** gateway nodes; return `429 + Retry-After` on deny.
- Non-functional: **low added latency** (it's on every request — single-digit ms), **highly available**
  (it can't be the thing that takes the system down), **accurate enough** (a little over-admission is fine for
  fairness; less fine for hard quotas), and horizontally scalable to the fleet's QPS.
- Clarify: hard limit or soft? Per-key only, or multiple dimensions? Fail open or closed? These change the design.

**2. Estimation.** Say 1M rps across the fleet, millions of distinct keys. → State is small per key (a counter
or a few values), so total state is modest (GBs) — fits in Redis. The challenge is **rps on the counter store**,
not storage. → Shard Redis by key.

**3. API.**
```
allow(key, cost=1) -> { allowed: bool, remaining: int, retryAfterMs: int }
```
Called by the gateway middleware before routing. On `allowed=false` → `429` with `Retry-After`.

**4. Algorithm choice.** "I'll use a **token bucket** — it enforces the average rate while allowing a
controlled burst, which is what API clients actually want. Bucket size = burst allowance, refill rate = the
sustained limit." (If they want strict no-burst, switch to leaky bucket; if they want exact rolling windows,
sorted-set sliding window log.)

**5. High-level design.**
- Gateway middleware calls a **rate-limiter component** → **Redis** holding `{tokens, lastRefillTs}` per key.
- The allow check is a **Lua script** run via `EVAL` so refill+check+decrement is **atomic** (no read-modify-write race).
- TTL on idle keys so unused buckets expire and reclaim memory.

**6. Deep dives (where it's won).**
- **Race condition** → atomic Lua script (explain why `GET`/`INCR` separately races; why single-threaded Redis Lua fixes it).
- **Latency of the Redis hop** → two-tier: **local in-process bucket** for the hot path that periodically
  reconciles with Redis, trading a little over-admission for ~zero added latency on most requests.
- **Redis as a SPOF** → cluster + replicas; decide **fail-open** (availability-first, default for fairness
  limits) vs **fail-closed** (protection-first). Say which and why.
- **Hot key** (one giant tenant) → shard that key across multiple Redis counters (split limit/N) or give them a dedicated bucket.
- **Multiple dimensions** → run several limiters (per-key AND per-IP AND global); deny if *any* trips, return the most restrictive `Retry-After`.
- **Clock skew across nodes** → use Redis server time / a monotonic source for refill math, not each node's wall clock.

**7. Wrap-up.** "Token bucket in Redis with an atomic Lua script for correctness, a local tier for latency, and
explicit fail-open semantics. Remaining risks: the local tier over-admits slightly, and a Redis-wide outage
falls back to local-only limiting. With more time I'd add adaptive concurrency limiting as a health-based
backstop beneath the static rate limits."

---

### Self-check before the mock (answer these from memory)
- [ ] When do you pick gRPC vs REST vs GraphQL vs WebSocket vs SSE vs long-polling?
- [ ] Which HTTP methods are idempotent, and how does that interact with retries and idempotency keys?
- [ ] Why is cursor pagination better than offset for a feed? Name both failure modes of offset.
- [ ] L4 vs L7 LB — what can L7 do that L4 can't, and what's the cost?
- [ ] Why does power-of-two-choices beat naive least-connections across a fleet of LBs?
- [ ] Global LB vs regional LB — what does each decide, and how does failover differ?
- [ ] Anycast vs GeoDNS vs DNS-based LB — and why is DNS failover slow?
- [ ] Gateway vs service mesh — north-south vs east-west, centralized vs sidecar.
- [ ] Name the 5 rate-limiting algorithms with their burst behavior, memory cost, and accuracy.
- [ ] Token bucket vs leaky bucket — which allows bursts, which smooths, when each?
- [ ] What's the distributed rate-limiting race condition, and how do atomic Lua / sorted sets fix it?
- [ ] Difference between rate limiting and load shedding / admission control.
- [ ] When do you terminate TLS at the edge vs use mTLS internally?
- [ ] What do you return on a rate-limit rejection, and why does `Retry-After` matter?
