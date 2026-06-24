# Topic 16: Service Architecture — Monolith vs Microservices, Mesh & Multi-Tenancy

> **Why this topic matters in a staff round:** Most candidates reach for "microservices" the way
> they reach for "add a cache" — reflexively, as a sign of sophistication. It's the opposite. The
> staff-level signal here is *restraint*: knowing that microservices buy you team-scaling and deploy
> independence at the price of distributed transactions, network failures, and an observability tax
> you pay forever. This doc is about earning the split, decomposing on the right seams, and then
> surviving the consequences — data consistency, communication, mesh, and multi-tenancy.

---

## Part A — Monolith vs Microservices vs Modular Monolith

### The three shapes

| | **Monolith** | **Modular Monolith** | **Microservices** |
|---|---|---|---|
| Code | One codebase | One codebase, enforced module boundaries | Many repos/services |
| Deploy | One artifact | One artifact | N independently deployable artifacts |
| Data | One DB | One DB (logical module ownership) | DB-per-service |
| Calls | In-process | In-process | Network (sync/async) |
| Team fit | 1–2 teams | A few teams, shared release train | Many teams, independent release |
| Failure | Process up or down | Process up or down | Partial failure is the *normal* state |

### The real tradeoffs

The thing candidates get wrong is framing this as "old vs modern." It's a tradeoff between two
kinds of cost, and you pick based on **how many teams you have**, not how fancy you want to look.

- **What microservices actually buy you:**
  - **Deploy independence** — team A ships without coordinating a release with team B. This is the
    *real* prize, and it's an org benefit, not a tech one.
  - **Team scaling / Conway's law** — "organizations design systems that mirror their communication
    structure." Past ~2 pizza-teams on one codebase, merge conflicts, release-train coupling, and
    "who broke the build" dominate. Service boundaries become *team* boundaries.
  - **Independent scaling** — scale the search service's CPU without scaling the whole app.
  - **Fault isolation** — *if* you do it right, a failing recommendations service degrades to "no
    recommendations," not a site outage. (This requires real work; it isn't free.)
  - **Tech heterogeneity** — the ML service in Python, the payments service in Go. Usually overrated.

- **What they cost you (the distributed-systems tax):**
  - **Distributed transactions** — you can no longer wrap a multi-entity change in one ACID
    transaction. You're now in saga / outbox / eventual-consistency land (Part E).
  - **Network failures** — every in-process call that *couldn't* fail now can: timeout, partial
    failure, retries, partial reads. You inherit the [fallacies of distributed computing](#).
  - **Observability burden** — a single user action is now 8 hops across 6 services. Without
    distributed tracing you are debugging blind.
  - **Operational surface** — N deploy pipelines, N on-call rotations, N dashboards, service
    discovery, a mesh, contract versioning.
  - **Latency amplification** — in-process nanoseconds become network milliseconds, *multiplied* by
    hop count (Part C).

> **Say this in the room:** "I would **not** start with microservices. I'd start with a
> **modular monolith** — strong internal module boundaries, one deployable, one database with clear
> per-module ownership. That gives me 80% of the architectural cleanliness with none of the
> distributed-systems tax. I extract a service only when a *specific* pressure forces it: a team
> that needs independent deploy cadence, a component with a wildly different scaling profile, or a
> bounded context that's genuinely autonomous. Premature decomposition is the #1 way to turn a hard
> problem into a *distributed* hard problem."

### The modular monolith is the senior default

It is the answer to "we want clean boundaries but we're 15 engineers." Enforce module boundaries
in code (package-private APIs, an architecture-test that fails the build on illegal cross-module
imports, module-owned schemas). When a module *earns* its independence, the seam is already drawn
and extraction is mechanical. **Decompose along seams you've already discovered, not seams you're
guessing at.** A distributed monolith — services that must deploy together and share a DB — is the
worst of both worlds: network tax with none of the independence.

---

## Part B — Service Decomposition

### Decompose by business capability / domain, not by layer

The classic mistake is decomposing by technical layer ("UI service, business-logic service, data
service") — that maximizes the coupling you wanted to break, because every feature change touches
all three. Decompose by **business capability** / **DDD bounded context**: `Orders`, `Payments`,
`Inventory`, `Shipping`. Each owns its data, its rules, and its vocabulary.

- **Bounded context** = a boundary within which a model and its language are consistent. "Customer"
  in Sales (a lead with deal stages) is a *different* model from "Customer" in Support (a ticket
  history). Forcing one shared "Customer" model across both is exactly the coupling DDD warns you
  about. One bounded context → at most one service (sometimes a few).
- **Aggregate** = the transactional consistency boundary *inside* a context (e.g., an `Order` and
  its `OrderLines`). A single service should fully own its aggregates so it can keep them
  transactionally consistent locally.

### Database-per-service is the load-bearing rule

| Pattern | Consequence |
|---|---|
| **DB-per-service** (correct) | Each service owns its schema and is the *only* writer. Others go through its API. |
| **Shared DB** (anti-pattern) | Every service couples to every other's schema. No one can migrate. Hidden write paths. |

**Why a shared DB is an anti-pattern:**
- It re-couples the services you just split — a schema change in one breaks others silently.
- It destroys deploy independence — you can't migrate a table without coordinating every consumer.
- It allows multiple writers to the same table → no single source of truth for invariants.
- It leaks the encapsulation: the database becomes the integration contract instead of the API.

> **The data-duplication consequence — and you must own it:** DB-per-service means the `Orders`
> service can't `JOIN` the `Customers` table. So data gets **duplicated** — Orders keeps a local
> copy of the customer fields it needs (name, address-at-time-of-order), fed by events from the
> Customers service. This is a *feature*, not a bug: it's denormalization across a service boundary,
> and it gives you autonomy and a point-in-time snapshot. The cost is **eventual consistency** of
> the copy and the machinery to keep it fresh (events / CDC / outbox — Part E). Cross-service joins
> become **API composition** or a read model (Part C/E), never a SQL join.

---

## Part C — Inter-Service Communication

### Sync vs async — the first fork

| | **Synchronous (REST / gRPC)** | **Asynchronous (events / messaging)** |
|---|---|---|
| Coupling | Temporal — callee must be up *now* | Decoupled — consumer can be down |
| Model | Request/response, immediate answer | Fire event, react later |
| Consistency | Read-after-write within the call | Eventual |
| Failure | Caller blocks; needs timeout/retry/breaker | Broker buffers; needs idempotency |
| Use when | You need the answer to proceed | You're notifying / triggering side effects |

- **REST** — ubiquitous, human-debuggable, cache-friendly, loose contracts. Good for external/public.
- **gRPC** — binary (protobuf), HTTP/2 multiplexing, codegen'd typed stubs, streaming, ~lower
  latency and payload. Good for internal east-west traffic. Cost: less human-readable, needs tooling.
- **Async (Kafka/SQS/RabbitMQ)** — the default when one event has many consumers or when you want to
  decouple producer and consumer lifecycles. Cost: eventual consistency, at-least-once delivery →
  **idempotent consumers**, and harder tracing. (See Topic 1's queue block, Topic 10's outbox.)

**Default heuristic:** sync for queries that need an immediate answer; async for commands/notifications
that trigger downstream work. Prefer async at boundaries — it's how you cut temporal coupling.

### The chatty-services / latency-amplification problem

This is the deep-dive trap interviewers love. If rendering one page fans out to 12 sequential
service calls, your p99 isn't the *average* hop latency — it's dominated by the **slowest** hop and
*compounds*. With 12 sequential 20 ms calls you've spent 240 ms before any work. Worse, p99
*amplifies*: if each service is 99% within 20 ms, the chance *all 12* are fast is `0.99^12 ≈ 89%` —
so ~11% of requests hit at least one slow hop. Tail latency is contagious.

**Mitigations to name:**
- **Don't make boundaries chatty** — a chatty interface is a sign you drew the boundary wrong; the
  data wants to live together. Redesign coarser-grained APIs (return the aggregate, not 5 calls).
- **Parallelize** independent calls (scatter-gather) instead of sequential chains.
- **API composition / BFF** at the edge (below) to fan out once, server-side and close to the
  services, rather than from the client over the WAN.
- **Cache** and **denormalized read models** (Part E) so the hot read doesn't fan out at all.

### Service discovery

Services come and go (autoscaling, restarts, failures), so you can't hardcode addresses.

| | **Client-side discovery** | **Server-side discovery** |
|---|---|---|
| How | Client queries registry, picks an instance, calls it directly | Client calls a stable LB/router; it consults the registry |
| LB logic | In the client (library) | In the infra (LB / mesh / k8s Service) |
| Pros | One less hop; client-aware LB | Clients stay dumb; language-agnostic |
| Cons | LB logic in every language's client | Extra hop; LB must be HA |
| Examples | Netflix Eureka + Ribbon | k8s Service + kube-proxy, AWS ALB, service mesh |

A **service registry** (Consul, etcd, Eureka, or k8s' built-in) is the source of truth: instances
register on startup and send heartbeats; unhealthy ones are evicted. In modern stacks this is mostly
handled *for* you by Kubernetes DNS + a service mesh (Part D), so say that.

### API composition and BFF

- **API composition** — a composer service (or the gateway) calls several services and stitches the
  result. It's the cross-service "join" you gave up in Part B. Cost: the composer's latency is the
  slowest dependency; partial-failure handling (return partial data vs fail).
- **BFF (Backend-for-Frontend)** — a *per-client* composition layer. The mobile app, the web SPA,
  and partners each have different payload/round-trip needs; a single API serving all three is a
  compromise that fits none. A BFF tailors aggregation, shaping, and chattiness-reduction per client.
  Owned by the *frontend* team. Cost: another deployable per client; risk of business logic leaking in.
- **API gateway** — the single edge entry point: TLS termination, authn, rate limiting, routing,
  request aggregation. This ties directly to your **edge doc** — the gateway is where cross-cutting
  edge concerns live so individual services don't reimplement them. A BFF is a *specialized,
  client-specific* gateway; an API gateway is the general one.

> **Say this in the room:** "I'll put an API gateway at the edge for authn, TLS, and rate limiting,
> and add BFFs if the web and mobile clients have meaningfully different aggregation needs. I will
> *not* let the gateway become a god-service with business logic — it routes and composes; the
> domain logic stays in the services."

---

## Part D — Service Mesh

Once you have many services talking over the network, every one of them needs the *same* hard
networking code: retries, timeouts, circuit breaking, mTLS, load balancing, tracing headers. Writing
that correctly in five languages, consistently, is a losing battle. A **service mesh** moves it out
of the app.

### The sidecar pattern

A proxy (typically **Envoy**) is deployed **next to every service instance** — same pod in
Kubernetes. All inbound/outbound traffic flows through the sidecar. The app just makes a plain
`localhost` call to "the order service"; the sidecar handles discovery, mTLS, retries, routing.

### What it moves out of app code

| Concern | Without mesh | With mesh |
|---|---|---|
| mTLS / identity | Each service does TLS + cert rotation | Sidecar does mutual TLS automatically (SPIFFE identities) |
| Retries / timeouts | Per-language libraries, inconsistent | Declarative policy, uniform |
| Circuit breaking | App-level library | Sidecar enforces |
| Load balancing | Client library | Sidecar (locality-aware, etc.) |
| Traffic splitting | Custom routing code | Declarative (90/10 canary, mirroring) |
| Observability | Per-service instrumentation | Uniform metrics/traces/golden signals at the proxy |

### Data plane vs control plane

- **Data plane** = the fleet of sidecar proxies actually moving the bytes (Envoy).
- **Control plane** = the brain that configures them — pushes routing rules, certs, policy
  (Istio's `istiod`, Linkerd's control plane). You write policy as config; the control plane
  distributes it to every data-plane proxy.

**Istio** (Envoy-based) is feature-rich and heavy. **Linkerd** is lighter, opinionated, simpler,
Rust micro-proxy. The tradeoff is power vs operational weight.

### The cost — name it, don't sell the mesh blindly

- **Latency** — every call now traverses two extra proxy hops (caller sidecar → callee sidecar).
  Usually sub-millisecond, but it's not zero and it's per-hop.
- **Resource overhead** — a sidecar container per pod (memory/CPU) across the whole fleet.
- **Operational complexity** — the mesh itself is a distributed system you now operate and debug;
  control-plane outages and Envoy config bugs are real.
- **Sidecar-less variants** (Istio Ambient, eBPF-based Cilium) trade some isolation to cut the
  per-pod overhead — worth a one-liner to show currency.

> **Say this in the room:** "A mesh is the right answer once I have *many* polyglot services and I'm
> tired of reimplementing mTLS, retries, and tracing in every one. For a handful of services I'd use
> libraries and skip the mesh — the operational weight isn't justified yet. The mesh's job is to
> push cross-cutting *network* concerns out of app code via sidecars, configured by a control plane."

---

## Part E — Data Consistency Across Services

You gave up the distributed transaction in Part B. Here's how you live without it.

### Sagas + the outbox (ties to Topic 10)

- **Saga** — a business transaction split into a sequence of *local* transactions, each in one
  service, with **compensating transactions** to undo on failure (no 2PC). *Orchestration* (a central
  coordinator drives the steps) vs *choreography* (services react to each other's events). Tradeoff:
  no global atomicity → states are *eventually* consistent and you must design compensations.
- **Transactional outbox** — to publish an event reliably *and* commit your DB change atomically,
  write the event into an `outbox` table **in the same local transaction**, then a relay/CDC process
  publishes it. Solves the dual-write problem (DB committed but event lost, or vice-versa). See Topic 10.

### CQRS — Command Query Responsibility Segregation

Separate the **write model** (commands, normalized, enforces invariants) from one or more **read
models** (queries, denormalized, shaped per query). Writes emit events; a projector **materializes**
read models from them.

- **When it's worth it:** read and write workloads are wildly asymmetric or have conflicting shapes;
  you need cross-service composed views without runtime fan-out; complex domains where the write
  model shouldn't be contorted to serve queries.
- **The read-model materialization:** a consumer subscribes to events and maintains a denormalized
  table/index purpose-built for one query (e.g., an Elasticsearch index, a wide "order summary"
  table joining customer + items). This is how you do cross-service "joins" without joins.
- **Cost:** eventual consistency between write and read side (the read model lags); more moving
  parts; you maintain projections and can rebuild them.

### Event sourcing

Instead of storing current state, store the **append-only log of events** that produced it; current
state is a fold over the log.

- **Benefits:** perfect **audit** trail (the log *is* the history); **replay** to rebuild any read
  model or recover from a projection bug; time-travel/temporal queries; natural fit with CQRS.
- **Cost — and this is real:** querying current state needs snapshots/projections; **schema
  evolution of events** is hard (old events are immutable forever — you must upcast them); steep
  conceptual learning curve; "fix a bad event" isn't a simple `UPDATE`.

> **Say this in the room:** "CQRS and event sourcing are powerful and *frequently over-applied*. I'd
> reach for CQRS when read/write shapes genuinely conflict or I need composed read models. I'd reach
> for event sourcing only when audit/replay is a first-class requirement — finance, ledgers,
> compliance. For a normal CRUD service, plain DB-per-service with an outbox is the right amount of
> machinery. Don't pay the event-sourcing tax for a domain that doesn't need its history."

---

## Part F — Multi-Tenancy

Serving many customers (tenants) from shared infrastructure. The core tension: **isolation/security/
blast-radius** vs **cost/operational efficiency**.

### The three models

| Model | What it is | Isolation | Cost/Density | Fits |
|---|---|---|---|---|
| **Silo** | Dedicated stack per tenant (own DB, sometimes own compute) | Strongest | Most expensive, lowest density | Enterprise, regulated, "no shared anything" |
| **Pool** | All tenants share infra and tables; tenant identified by a column | Weakest (logical) | Cheapest, highest density | Long-tail SMB / freemium |
| **Bridge** | Mix — shared compute, isolated data (schema or DB per tenant), or tiered | Tunable | Middle | Most real SaaS, tiered by plan |

Most mature SaaS runs **bridge / tiered**: free and small tenants pooled, big/enterprise tenants
siloed — often *the same codebase* deployed in both modes.

### Tenant data isolation — three levels

| Level | Isolation | Per-tenant ops (migrate, restore, delete) | Number of tenants it scales to |
|---|---|---|---|
| **Row-level** (`tenant_id` column) | Logical only — one bad query leaks across tenants | Hard (filtered) | Many thousands |
| **Schema-per-tenant** | Stronger; separate namespace | Easier | Hundreds–low thousands (schema count limits) |
| **DB-per-tenant** | Strongest; separate backup/restore/encryption keys | Easiest per-tenant | Hundreds (operational ceiling) |

For **row-level**, the non-negotiable safety net is enforcing the tenant filter *below* application
code — **Postgres Row-Level Security (RLS)**, or a mandatory tenant predicate in a data-access layer
no query can bypass. "Every developer remembers the `WHERE tenant_id = ?`" is how you get a breach.

### The noisy-neighbor problem

In pooled models, one tenant's heavy workload (a runaway report, a bulk import, a hot tenant) starves
everyone else sharing the resource. Mitigations:

- **Per-tenant rate limits / quotas** — cap QPS, concurrency, and resource consumption per tenant at
  the gateway (ties to your rate-limiter and edge docs).
- **Tenant-aware sharding** — make `tenant_id` part of the shard key so a tenant's data and load are
  bounded to a shard; **place whale tenants on their own shard** (or silo them) so they can't starve
  the pool. Watch for hot shards from a single mega-tenant.
- **Bulkheads / fair scheduling / work isolation** — separate thread pools or queues per tenant tier
  so one tenant's backlog can't drain shared workers.
- **Tiering** — promote heavy tenants from pool → bridge → silo as they grow.

> **Say this in the room:** "I'd start pooled with a `tenant_id` enforced by row-level security and
> per-tenant rate limits, shard by tenant so load is bounded, and keep a *silo* path for enterprise
> customers who require data isolation or live in a specific region. Multi-tenancy is mostly a
> blast-radius and noisy-neighbor problem — I design the isolation level to the tier."

---

## Part G — Versioning, Compatibility, Schema Evolution

Independent deploy means **a producer and consumer will run different versions simultaneously**.
Your contracts must survive that.

- **Backward compatible** — new code reads old data / old requests. (New consumer, old producer.)
- **Forward compatible** — old code tolerates new data it doesn't understand (ignores unknown fields).
- The goal: **never break a running consumer.** You almost never get to deploy everything at once.

### Schema evolution with protobuf / Avro

- **Protobuf** — fields identified by **tag number**, not name. Rules: never reuse/renumber a tag;
  add new fields as **optional**; `reserved` removed tags. Unknown fields are preserved/ignored →
  built-in forward+backward compatibility. This is *why* gRPC ecosystems evolve cleanly.
- **Avro** — relies on a **schema registry** with reader/writer schemas; default values let readers
  cope with missing fields. Enforces compatibility (BACKWARD/FORWARD/FULL) at registration time, so
  an incompatible schema is rejected before it ships. Common in Kafka pipelines.
- **REST/JSON** — additive changes only; tolerant readers (ignore unknown fields); version in the URL
  (`/v2/`) or header for breaking changes; deprecate on a schedule, never yank.

### The expand-contract (parallel-change) migration pattern

The canonical way to make a breaking change without breaking anyone:

1. **Expand** — add the new field/column/endpoint alongside the old. Deploy. Nothing reads it yet.
2. **Migrate** — dual-write to both old and new; backfill; move readers to the new path. Both work.
3. **Contract** — once no one uses the old path (verify via metrics), remove it. Deploy.

Each step is independently deployable and reversible. This applies equally to **DB column renames**
(add new col → dual-write → backfill → switch reads → drop old col) and **API changes**. Memorize the
three words — interviewers love hearing "I'd do an expand-contract migration so nothing breaks
mid-rollout."

---

## Part H — Deployment & Release

### Containers & orchestration (conceptual)

- **Container** — package app + deps into one immutable, portable image; "runs the same everywhere."
- **Kubernetes** — declarative orchestration: you describe desired state (N replicas, this image,
  these resources), the control loop reconciles reality to it. Gives you self-healing (restart dead
  pods), rolling updates, horizontal autoscaling, service discovery (k8s Service + DNS), and the
  substrate a service mesh plugs into. At staff level, know the *concepts* (desired-state
  reconciliation, pods/replicas/services, rollouts) — not YAML trivia.

### Release strategies (ties to the resilience doc)

| Strategy | How | Pro | Con |
|---|---|---|---|
| **Rolling** | Replace instances batch by batch | Simple, no extra fleet | Both versions live mid-roll; slow rollback |
| **Blue/Green** | Stand up full new env (green), flip traffic, keep blue | Instant rollback (flip back) | 2× resources during cutover; DB schema must support both |
| **Canary** | Route a small % to new version, watch metrics, ramp | Limits blast radius; data-driven promote/rollback | Needs good observability + traffic routing (mesh helps) |

**Canary** is the staff default for risky changes: ship to 1% → watch error rate / latency / business
metrics → automatically promote or roll back. A service mesh's traffic splitting (Part D) makes this
trivial. Pair with **feature flags** to decouple *deploy* from *release* — ship code dark, turn it on
for a cohort, kill it instantly without a redeploy.

### 12-factor config

Config that varies by environment lives in the **environment**, not the code or the image. Same
immutable artifact promotes dev → staging → prod; only injected config (env vars, mounted secrets)
differs. Secrets come from a vault (not the repo), strict separation of config from code. This is what
makes blue/green and canary safe — the *same* tested image runs everywhere.

---

## Part I — "Should this be one service or many?" decision guide

Walk this out loud; it's a high-signal way to show restraint.

1. **How many teams?** One or two → monolith / modular monolith. Don't split for splitting's sake.
2. **Is there a clear bounded context with its own data and language?** No → keep it a module. A
   service that can't own its data isn't a service.
3. **Does a component need an independent deploy cadence?** A piece that ships 10×/day while the rest
   ships weekly is a candidate. This is the strongest *real* reason to split.
4. **Does it have a radically different scaling profile?** CPU-bound ML inference vs IO-bound CRUD →
   worth isolating so you can scale them independently.
5. **Would splitting force a distributed transaction across the new boundary?** If two pieces must
   change atomically together, **keep them in one service.** Don't cut through an aggregate.
6. **Will the interface be chatty?** If A calls B 8 times per request, the data wants to be together —
   wrong seam.
7. **Can you afford the tax?** Tracing, mesh, N pipelines, on-call. No platform maturity → don't.

> **Default position to state:** "Start as a modular monolith. Extract a service only when a concrete
> force appears — team autonomy, deploy cadence, or a divergent scaling profile — and only along a
> bounded context that owns its own data and won't require a distributed transaction across the new
> seam. Microservices are an answer to an *organizational* scaling problem, and I treat them as a cost
> to be justified, not a default."

---

## Part J — How to use this in the room

1. **Lead with restraint.** "I wouldn't start with microservices" is a stronger opening than naming
   12 services. Earn the split.
2. **Tie data ownership to the boundary.** DB-per-service + the data-duplication/eventual-consistency
   consequence is the detail that proves you've actually run microservices.
3. **Name the tax every time you split** — distributed transactions, network failure, observability.
4. **Reach for sagas/outbox/CQRS when, and only when, the consistency requirement demands it.**
5. **For multi-tenancy, design isolation to the tier** (pool the long tail, silo the whales) and
   always raise the noisy-neighbor problem.
6. **Make breaking changes safe** — expand-contract, every time.

> **Mental checklist for an architecture prompt:**
> How many teams? → Where are the bounded contexts? → Does each own its data? → Sync or async at the
> seam? → What's the consistency story (saga/outbox/CQRS)? → How do I observe it (tracing/mesh)? →
> How do I roll it out safely (canary + flags)? → If multi-tenant, what isolation per tier?

---

### Self-check before the mock (answer these from memory)
- [ ] Give the real reason to choose microservices, and the three taxes you pay for it.
- [ ] Why is a modular monolith the senior default, and what makes a "distributed monolith" the worst case?
- [ ] Why is a shared database an anti-pattern, and what does DB-per-service force on your data (and how do you cope)?
- [ ] Explain latency amplification across chatty services and three ways to fix it.
- [ ] Client-side vs server-side service discovery — who holds the LB logic in each?
- [ ] What does a service mesh move out of app code, and what's the difference between data plane and control plane?
- [ ] When is CQRS worth it? When is event sourcing worth it? What's the cost of each?
- [ ] Silo vs pool vs bridge — and what isolation level stops a noisy neighbor and a cross-tenant data leak?
- [ ] Walk the expand-contract migration in three steps for a column rename.
- [ ] Blue/green vs canary — when do you pick each, and how do feature flags change the picture?
