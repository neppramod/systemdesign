# Topic 19: Networking Fundamentals for System Design

> **Why this matters in a design round:** Networking is the substrate every box-and-arrow lives
> on. The interviewer rarely asks "explain TCP" — but they constantly ask things whose *real*
> answer is networking: "why is your cross-region read slow?", "why pool connections?", "where do
> you terminate TLS?", "what happens when the network partitions?". A staff candidate doesn't
> recite the OSI model; they *reason from the wire* to justify edge placement, caching, timeouts,
> and protocol choice. Master the request lifecycle and the latency floor and most of these
> questions answer themselves.

---

## Part A — The request lifecycle (your universal framing tool)

When you're stuck, walk a single request from the user's keystroke to bytes on screen. It forces
you to name every hop, every place latency hides, and every component that can fail. **Say this
sequence out loud** — it signals depth and buys thinking time.

`https://app.example.com/feed` typed into a browser:

1. **DNS resolution** — turn `app.example.com` into an IP. Browser cache → OS cache → recursive
   resolver → root → TLD → authoritative. Returns an A/AAAA record. *Cost: 0ms if cached, 20–120ms if not.*
2. **TCP handshake** — SYN → SYN-ACK → ACK to the resolved IP (or to the nearest edge/anycast PoP).
   *Cost: 1 RTT.*
3. **TLS handshake** — negotiate cipher, exchange keys, verify cert. *Cost: 1 RTT (TLS 1.3) or 2 RTT (TLS 1.2).*
4. **HTTP request** — `GET /feed`, headers, cookies go out. Server (often a reverse proxy / LB
   first) routes to a service.
5. **Server work** — authn, cache lookups, DB queries, fan-out. *This is the part the rest of your design is about.*
6. **HTTP response** — bytes stream back. Browser parses HTML, discovers sub-resources (CSS/JS/img),
   and **repeats 1–6 for each** (unless multiplexed / cached / same-origin keep-alive).
7. **Render** — paint to screen.

> **The reframing trick:** "Before I optimize the server, notice we paid ~3 round trips
> (DNS + TCP + TLS) before a single byte of application data moved. For a user 150ms away, that's
> ~450ms of overhead. That's *why* we put an edge/CDN in front — it collapses those round trips to
> a nearby PoP." This one observation justifies edge, keep-alive, TLS 1.3, and HTTP/3 all at once.

The whole rest of this doc is "go deep on one of those seven steps."

---

## Part B — DNS in depth

DNS is a distributed, hierarchical, cached key→value lookup. It's also a surprisingly common
**single point of failure** and a sneaky source of latency and stale-config bugs.

### Resolution steps
1. **Browser cache** → 2. **OS cache** (stub resolver) → 3. **Recursive resolver** (your ISP's, or
   8.8.8.8 / 1.1.1.1). If the recursive resolver doesn't have it cached, it walks the hierarchy:
4. **Root servers** (`.`) → point to the TLD server. 5. **TLD servers** (`.com`) → point to the
   domain's authoritative server. 6. **Authoritative server** (the one *you* control via your DNS
   provider) → returns the actual record.

### Recursive vs authoritative — say this distinction cleanly
- **Recursive resolver**: does the legwork *on behalf of the client*, caching aggressively. The
  client asks one question and gets one answer.
- **Authoritative server**: the source of truth for a zone. It answers "I *am* the authority for
  example.com, here's the record." It does not chase referrals.

### TTL & caching
Every record carries a **TTL**. Resolvers cache until it expires. This is the lever for everything:
- **Low TTL (e.g. 60s)**: fast failover / fast config change, but more DNS query load and more
  resolver chatter.
- **High TTL (e.g. 24h)**: cheap, resilient to DNS outages, but changes propagate *slowly*.

> **The propagation-delay gotcha — interviewers love this:** "I'll just repoint DNS to the new
> region" is not instant. Clients keep using the old IP until their *cached* TTL expires, and many
> resolvers (and some clients/JVMs) **ignore TTL** and cache longer. So DNS failover is **minutes**,
> not seconds. If you need fast failover, don't rely on DNS alone — front it with an anycast VIP or
> a load balancer that can flip backends behind a stable IP.

### DNS-based load balancing & GeoDNS
- **Round-robin DNS**: return multiple A records; clients pick one. Crude LB, no health awareness,
  caching defeats even distribution.
- **GeoDNS / latency-based routing**: the authoritative server returns *different* IPs based on the
  resolver's location/latency, steering users to the nearest region. (Route 53 latency/geo routing.)
- **Health-checked DNS**: provider stops handing out IPs of dead endpoints — but bounded by TTL.

### Anycast
Announce the *same IP* from many physical locations via BGP; the network routes each client to the
**topologically nearest** instance. Used by DNS roots, big resolvers (1.1.1.1), and CDNs. Buys you
proximity and a natural DDoS sink (load spreads across PoPs) **without** the client doing anything.

### DNS as a failure point
DNS outages take down everything downstream even when your servers are healthy (the 2016 Dyn /
Mirai outage; multiple cloud incidents). Mitigations to name: **secondary DNS provider**, sane TTLs,
monitoring resolution from multiple geographies, and not putting your control-plane's only failover
behind a single zone.

---

## Part C — TCP vs UDP

This is the protocol fork everything above HTTP sits on. Know the table cold and know *when UDP wins*.

| | **TCP** | **UDP** |
|---|---|---|
| Connection | Connection-oriented (3-way handshake) | Connectionless (fire-and-forget) |
| Reliability | Guaranteed delivery, retransmits lost packets | None — app must handle loss |
| Ordering | In-order delivery | No ordering guarantee |
| Flow control | Yes (receiver window) | No |
| Congestion control | Yes (slow start, AIMD) | No (app's problem) |
| Head-of-line blocking | **Yes** — one lost segment stalls the stream | No |
| Overhead | Higher (handshake, ACKs, state) | Minimal (8-byte header) |
| Use when | Correctness > latency: web, APIs, DBs, file transfer | Latency > perfection: gaming, live video/voice, DNS, QUIC |

### TCP handshake & teardown
- **SYN → SYN-ACK → ACK** = 1 RTT before any data. Teardown is a 4-way FIN/ACK dance.
- This handshake cost is *per connection*, which is the entire argument for **connection reuse**.

### Reliability machinery (the parts that bite you)
- **Flow control** — the receiver advertises a window; sender won't overrun a slow receiver.
- **Congestion control** — **slow start** ramps the congestion window exponentially from a small
  value, then backs off on loss (AIMD). Consequence: **a brand-new TCP connection is slow at first**
  — it hasn't "warmed up." Long-lived pooled connections stay warm and run faster. *This is a real
  argument for connection pooling beyond just skipping the handshake.*
- **Head-of-line (HOL) blocking** — TCP delivers bytes in order, so a single lost segment stalls
  *everything* behind it until the retransmit lands. This is the flaw HTTP/2 couldn't escape (it
  multiplexed *above* one TCP connection) and HTTP/3 fixed (by moving to UDP/QUIC).

### When UDP wins
- **Real-time media / gaming**: a 200ms-late video frame is useless — better to drop it than stall
  the stream. Loss < latency.
- **DNS**: a single small request/response; setting up TCP would cost more than the query.
- **QUIC (HTTP/3)**: builds its *own* reliability + ordering *per-stream* on top of UDP, dodging
  TCP's kernel-level HOL blocking. UDP here is a transport substrate, not "unreliable on purpose."

### Connection setup cost → why pooling/keep-alive matters
Every new TCP+TLS connection = handshakes (≥2 RTT) + cold congestion window. For a service making
thousands of downstream calls, opening a fresh connection per call is brutal. **Reuse**:
- **HTTP keep-alive** — reuse one TCP connection for many sequential requests.
- **Connection pools** — service/DB clients keep a warm pool; you pay handshake + slow-start once.
- This is why "use a connection pool to the database" and "enable keep-alive at the LB" are
  reflexive staff answers.

---

## Part D — TLS handshake

TLS gives you confidentiality, integrity, and authentication (the server proves it's really
`example.com`). The cost is round trips *on top of* TCP.

- **TLS 1.2**: ~2 RTT to establish (cipher negotiation + key exchange + cert verify) before app data.
- **TLS 1.3**: **1 RTT** — streamlined handshake, weak ciphers removed. **0-RTT resumption** lets a
  returning client send data in the *first* packet (at the cost of replay risk for that early data,
  so only safe for idempotent requests).
- **Session resumption** (session IDs / tickets) lets repeat clients skip the full handshake — keep
  TLS sessions warm the same way you keep TCP connections warm.

### Where you terminate TLS matters
- **Terminate at the edge / CDN / LB**: shortest encrypted path from the user (the expensive
  handshake happens near them), and your origin offloads crypto. But traffic between the LB and
  backend is now plaintext unless you re-encrypt.
- **Terminate at the origin / pass-through**: end-to-end encryption, but the user pays the full
  handshake RTT to a possibly-distant origin.
- **Re-encrypt (TLS at edge *and* edge→origin)**: common in regulated environments — encrypted on
  every hop, more CPU.

> **Say this:** "I'd terminate TLS at the edge for latency, then re-encrypt to origin if we have a
> compliance requirement for in-transit encryption inside our own network."

### mTLS
**Mutual TLS** — *both* sides present certs, so the server authenticates the client too. The
backbone of zero-trust service meshes (Istio/Linkerd): every service-to-service call is mutually
authenticated and encrypted. Mention it when the prompt is about internal east-west security or a
service mesh, not for public client traffic.

---

## Part E — HTTP evolution

Each version exists to fix the previous version's bottleneck. Know *which problem* each solves.

| | **HTTP/1.1** | **HTTP/2** | **HTTP/3** |
|---|---|---|---|
| Transport | TCP | TCP | **QUIC over UDP** |
| Concurrency | One request per connection (browsers open 6) | **Multiplexed** streams over 1 connection | Multiplexed streams over QUIC |
| HOL blocking | Application-level (one slow response blocks the connection) | **TCP-level** (one lost packet stalls all streams) | **None** — independent per-stream delivery |
| Headers | Plaintext, repeated | **HPACK** compression | **QPACK** compression |
| Handshake | TCP + TLS (≥2–3 RTT) | TCP + TLS | **Combined QUIC+TLS 1.3 (1 RTT, 0-RTT resume)** |
| Server push | No | Yes (largely deprecated in practice) | (Deprecated) |

### HTTP/1.1
- **Keep-alive** reuses the TCP connection across requests (the big 1.1 win over 1.0).
- **Application HOL blocking**: responses must come back in order on a connection, so one slow
  response blocks the rest. Browsers worked around this by opening ~6 parallel connections per
  origin (multiplying handshake cost).
- **Pipelining** (send multiple requests without waiting) was specced but **failed in practice** —
  buggy proxies, and it still suffered HOL blocking. Effectively dead.

### HTTP/2
- **Multiplexing**: many concurrent streams over **one** TCP connection — kills application-level
  HOL blocking and the 6-connection hack. One warm connection per origin.
- **Header compression (HPACK)**: headers are repetitive and large (cookies!); compress them.
- **Server push**: server preemptively sends resources. Sounded great, **deprecated** — hard to get
  right, usually pushed things the client already had cached. Don't propose it.
- **The remaining flaw**: it still rides *one TCP connection*, so a single lost packet causes
  **TCP-level HOL blocking** that stalls *all* streams. You can't multiplex your way out of the
  transport's in-order delivery.

### HTTP/3 / QUIC
- Runs over **UDP**, implementing reliability + ordering **per-stream in user space**. A lost packet
  on stream A no longer stalls stream B — **no transport HOL blocking**.
- **Combined transport + crypto handshake** (QUIC integrates TLS 1.3): typically **1 RTT**, **0-RTT**
  on resumption. Faster first byte, especially on lossy/mobile networks.
- **Connection migration**: a connection ID survives IP changes (Wi-Fi → cellular) without a new
  handshake — big for mobile.

### Practical implications for API/service design
- **Use HTTP/2 between your edge and backends and for gRPC** — multiplexing makes one warm
  connection carry huge concurrency; great for chatty microservices.
- **HTTP/3 at the edge** helps real users on lossy/mobile last-miles most; inside a clean datacenter
  the gains shrink.
- Don't design around server push. Do design around **fewer, longer-lived, multiplexed connections**.

---

## Part F — Latency vs bandwidth vs throughput

Mix these up and a senior interviewer notices immediately.

- **Latency** — *time* for one trip (or round trip). Measured in ms. Dominated by distance and hops.
- **Bandwidth** — the *capacity* of the pipe: bits/sec it *could* carry.
- **Throughput** — the bits/sec you *actually* achieve (≤ bandwidth, capped by latency, loss,
  window size, congestion).
- **RTT** — round-trip time. The unit your handshakes and chatty protocols are billed in.

> **The mental model:** bandwidth is the *width* of the pipe; latency is the *length*. A fat pipe
> doesn't help a chatty protocol that's paying many serial round trips — that's a latency problem,
> and you fix it with fewer round trips (multiplexing, pipelining the *right* way, edge), not a
> bigger pipe.

### The speed-of-light floor (the number that forces architecture)
Light in fiber is ~200,000 km/s. That sets a **hard floor** you cannot engineer away:
- Cross-US round trip ≈ **~60–80ms**. Trans-Atlantic ≈ **~70–90ms**. Trans-Pacific ≈ **~150ms+**.
- These are *minimums* assuming a straight path; real routing adds more.

**What it forces:**
- You **cannot** serve a global user base from one region at low latency. Physics says no.
- → **Edge/CDN** to terminate connections near users and serve cached bytes locally.
- → **Regional replicas / multi-region** so reads (and ideally writes) happen near the user.
- → **Caching** so you avoid the cross-region trip entirely.
- → **Async / batching** so you stop paying that RTT serially per item.

### Bandwidth-delay product (BDP)
`BDP = bandwidth × RTT` = the amount of in-flight data needed to keep a high-latency pipe *full*.
On a high-bandwidth, high-latency link (a "long fat network"), a too-small TCP window leaves the
pipe under-utilized — you're latency-bound even though bandwidth is plentiful. This is *why* big
transfers across regions need tuned window sizes / parallel streams, and why slow-start hurts on
long links.

---

## Part G — Proxies, L4/L7, and NAT

### Forward vs reverse proxy
- **Forward proxy**: sits in front of *clients*, acting on their behalf (corporate egress filter,
  VPN, caching proxy). The server sees the proxy, not the client.
- **Reverse proxy**: sits in front of *servers*, acting on their behalf (nginx, Envoy, an LB, a
  CDN edge). The client sees the proxy, not the backend. This is your LB / API gateway / TLS
  terminator / edge — the workhorse of every design.

### L4 vs L7 (ties to the edge/LB topic)
- **L4 (transport) load balancer**: routes by IP/port, forwards TCP/UDP without reading payload.
  Fast, cheap, protocol-agnostic, can't make content decisions. (AWS NLB.)
- **L7 (application) load balancer / reverse proxy**: parses HTTP — can route by path/header/cookie,
  terminate TLS, do retries, rate-limit, and load-balance per-request across a multiplexed
  connection. More CPU, more features. (AWS ALB, Envoy, nginx.)

> **Say this:** "I'd use an L7 proxy at the edge for path-based routing and TLS termination, and L4
> deeper in the stack where I just need fast, dumb fan-out."

### NAT basics
**Network Address Translation** maps many private IPs to one public IP (and back) by rewriting
address/port. It's why your private subnet of servers can reach the internet through a gateway, and
it's why **inbound** connections to instances behind NAT need explicit port forwarding / a public
LB. Relevant when you discuss VPC layout, private subnets, and why databases live in private
subnets reachable only through the app tier.

---

## Part H — CDN / edge from the network angle

Everything above converges here. A CDN is "move the bytes *and the connection termination* close to
the user":
- **Proximity collapses round trips**: DNS, TCP, and TLS all complete against a nearby PoP instead
  of a distant origin. For a static asset that's the *whole* request; for dynamic content you still
  shave the handshake overhead and keep a warm origin connection.
- **Anycast routing** sends each user to the nearest PoP automatically (Part B).
- **Origin offload**: cache hits never touch your origin, cutting both load and the long-haul RTT.
- Tie back to the latency floor: the CDN is how you *defeat* the speed-of-light tax for cacheable
  content and *minimize* it for the rest.

---

## Part I — Networking failure modes (the interview gold)

When asked "what can go wrong," reach for these. They're the difference between a system that works
in the demo and one that survives production.

### The classes of failure
- **Network partition** — a link/segment can't talk to another; both sides may think the other is
  dead. Forces the CAP choice (consistency vs availability). Beware **split-brain** (two leaders).
- **Slow network (gray failure)** — *worse than a clean failure*. Latency spikes but packets still
  trickle, so health checks pass while real requests time out. Causes cascading retries and queue
  buildup.
- **Packet loss / reordering** — triggers TCP retransmits and slow-start backoff; throughput craters.
- **Retries & timeouts** — your main tool *and* your main footgun. Naive retries amplify load on an
  already-struggling service (**retry storm**). Mitigate with: **timeouts on every call**,
  **exponential backoff + jitter**, **idempotency keys** (so retries don't double-charge), **circuit
  breakers**, and **retry budgets** (cap retries as a % of traffic).

> **Say this:** "Every network call gets a timeout and a bounded retry with jitter. Without a
> timeout, one slow dependency exhausts my thread/connection pool and the failure cascades. Retries
> without backoff turn a brownout into an outage."

### The Fallacies of Distributed Computing (recite the list + the lesson)
1. **The network is reliable** → it isn't; design for loss, retry idempotently.
2. **Latency is zero** → distance has a floor; minimize round trips, go to the edge.
3. **Bandwidth is infinite** → payloads cost; compress, paginate, avoid chatty N+1 calls.
4. **The network is secure** → encrypt in transit (TLS/mTLS), don't trust the wire.
5. **Topology doesn't change** → nodes/IPs move; use service discovery, don't hardcode IPs.
6. **There is one administrator** → many teams/clouds; expect config drift and coordinate changes.
7. **Transport cost is zero** → serialization + bandwidth + egress fees are real; batch and compress.
8. **The network is homogeneous** → mixed devices/protocols/MTUs; don't assume uniform behavior.

The point isn't to recite them as trivia — it's that *each* maps to a design defense you can name on
the spot.

---

## Part J — The latency budget exercise (do this out loud)

When someone asks "why is this cross-region request 600ms?", **decompose where the milliseconds go.**
Take a user in Europe hitting an origin in `us-east-1`, dynamic (uncacheable) response:

| Stage | Cost | Note |
|---|---|---|
| DNS (cache miss) | 20–80ms | 0 if cached / behind anycast resolver |
| TCP handshake | ~90ms | 1 RTT trans-Atlantic |
| TLS 1.3 handshake | ~90ms | 1 RTT (2 RTT if TLS 1.2 → ~180ms) |
| Request → origin | ~45ms | half-RTT for the request to arrive |
| Server processing | 50–200ms | DB, cache, fan-out — *your design* |
| Response → client | ~45ms | half-RTT back |
| **Total** | **~340–550ms** | before any sub-resources |

**What the budget tells you to do:**
- ~225ms of that is *pure handshake + transit overhead*, paid before/around the server work. Put an
  **edge PoP in Europe**: DNS/TCP/TLS now terminate ~10ms away, collapsing ~270ms to ~30ms.
- Use **TLS 1.3 + session resumption** to drop a handshake RTT; **0-RTT** for idempotent reads.
- **Keep-alive / HTTP/2 multiplexing** so subsequent requests skip handshakes entirely.
- If the data is cacheable, the CDN serves it from Europe and the origin RTT *disappears*.
- If it's *not* cacheable, you need a **read replica / regional deployment** in Europe — physics
  won't let you serve it fast from Virginia.

> **The staff move:** never answer "the server is slow" without decomposing the budget. Often the
> server is fine and 60% of the latency is round trips you can eliminate with edge + connection reuse.

---

## Part K — How to use this in the room

1. **Walk the lifecycle** when asked anything latency-shaped: DNS → TCP → TLS → HTTP → server → render.
2. **Quote the latency floor** to justify edge, regional replicas, and caching — physics is an
   unarguable reason.
3. **Reach for connection reuse reflexively**: pooling, keep-alive, HTTP/2 — and say *why* (handshake + slow-start).
4. **Pick the protocol with a tradeoff sentence**: "TCP for correctness; UDP/QUIC where a late packet is worthless."
5. **Name the failure mode and its defense together**: timeout + backoff + jitter + idempotency + circuit breaker.
6. **Decompose the latency budget** instead of hand-waving "it's slow."

---

### Self-check before the mock (answer these from memory)
- [ ] Walk a request from URL to render, naming every round trip before app data flows.
- [ ] Recursive vs authoritative DNS; why is DNS failover *minutes* not seconds?
- [ ] TCP vs UDP across the full table — and three places UDP wins.
- [ ] Why does connection pooling help *beyond* skipping the handshake? (slow start)
- [ ] TLS 1.2 vs 1.3 round trips; where would you terminate TLS and why?
- [ ] What flaw does HTTP/2 *not* fix, and how does HTTP/3 fix it?
- [ ] Latency vs bandwidth vs throughput; what does the speed-of-light floor force you to build?
- [ ] L4 vs L7 proxy — when each.
- [ ] List the 8 fallacies of distributed computing and one design defense for each.
- [ ] Decompose a 500ms cross-region request; where do the milliseconds go and what removes them?
