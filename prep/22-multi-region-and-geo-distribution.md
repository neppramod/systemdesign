# Topic 22: Multi-Region & Globally Distributed Architecture

> **Why this topic is hard at staff level:** Going multi-region is the one decision where the
> *physics* fight you. You cannot make a packet cross an ocean faster than ~70ms one way, and no
> amount of clever engineering buys it back. So multi-region is never a "scale the boxes" problem —
> it's a problem of deciding, per dataset, **where the authority lives** and **what you give up when
> a region you depend on is unreachable**. The candidate who jumps straight to "active-active across
> three regions" without naming the write-latency and conflict costs has failed the round. The one
> who says "stateless tier everywhere, data tier homed per requirement, and here's the consistency I
> trade" has passed it.

---

## Part A — Why go multi-region (and why it's expensive)

There are exactly three reasons to pay the multi-region tax. **Say which one you're solving for** —
they pull the design in *different* directions, and conflating them is the classic mistake.

| Driver | What it buys | What it demands |
|---|---|---|
| **Latency** | Serve users from a region near them → cut round-trip 150ms → 20ms | Data (or a usable copy) must be *close* to the user → replication or geo-homing |
| **Availability / DR** | Survive a full region outage → no global downtime | A *second* region that can take over → state must exist in ≥2 places |
| **Data residency / compliance** | EU user data stays in EU (GDPR), India data in India (sovereignty) | Data *pinned* to a region; you may be legally barred from replicating it out |

> **Say this in the room:** "Before I design anything, which of latency, DR, or residency are we
> optimizing? Because residency says 'pin data and *don't* replicate it', while latency says
> 'replicate it everywhere' — those are opposite. If it's all three, I'll partition by geo so each
> requirement is satisfied locally."

### The cost you're signing up for
- **Cross-region latency — the speed-of-light floor.** Fiber carries light at ~⅔ c. US-East ↔ EU is
  ~70–90ms *one way*, ~150ms round trip. US ↔ Asia is ~120–150ms one way. **No CDN, no cache, no
  protocol fixes this.** Any design that puts a synchronous cross-region hop on the request path eats
  this on *every* such request.
- **Replication.** Bytes must move between regions continuously — bandwidth cost, lag, and a new
  failure surface (the replication link itself).
- **Conflicts.** The moment two regions can accept writes to the same key, you have a distributed
  systems problem that does not exist single-region (see Part D).
- **Operational complexity.** Deploys, schema migrations, secrets, certs, observability — all now
  ×N regions, and they must roll out *consistently* or you get version skew across the fleet.

> **The framing that scores points:** "Multi-region is not free availability. A naive active-active
> setup can be *less* available than single-region, because now a network partition between regions
> can corrupt data via conflicting writes. I only go multi-region when a *requirement* forces it, and
> I pay only for the consistency the data actually needs."

---

## Part B — The evolution: single-region → multi-region

You almost never start multi-region. You earn your way there. Walk the interviewer up the ladder:

1. **Single region, multi-AZ.** This is the default and it is *not* multi-region. Three AZs in one
   region survive a datacenter/AZ failure with sub-ms inter-AZ latency and synchronous replication.
   **Most systems should stop here.** Say so. "Multi-AZ gives me ~99.99% and DR within the region
   with no cross-region latency. I'd only go further if I have a latency, full-region-DR, or
   residency requirement."
2. **Add a read region (read replicas).** First step out: stateless compute + read replicas in a
   second region. Writes still go home; reads served locally. Buys read latency + a warm standby.
3. **Active-passive (DR region).** A full standby region that can be promoted. Buys region-level DR.
4. **Active-active / geo-partitioned.** Multiple regions accept writes. Buys write latency + true
   active-active availability — and brings the full conflict problem.

### The three tiers, and which is hard
Every multi-region system decomposes into three tiers. The difficulty is *entirely* in the last one.

| Tier | What it is | How hard to make multi-region |
|---|---|---|
| **Edge / CDN** | Static assets, TLS termination, edge cache, sometimes edge compute | **Easy.** This is a solved product (CloudFront, Cloudflare, Fastly). |
| **Regional stateless compute** | App servers, API layer — no local state | **Easy.** Deploy the same artifact to each region behind regional LBs. It's a config + CI/CD problem. |
| **Data tier** | Databases, queues, caches — anything stateful | **The whole problem.** Everything below is about this tier. |

> **Say this in the room:** "Compute is cattle — I stamp it out in every region and forget about it.
> The interesting question, and where I'll spend my time, is the data: who owns the write, how far is
> the copy, and what happens on a partition."

---

## Part C — Data strategies across regions

This is the core decision. There are three families. Internalize the **write-latency** and
**consistency** consequences of each — that's what gets probed.

### 1. Single write region + read replicas (active-passive writes)
One region is the leader for writes; all other regions hold read replicas, asynchronously replicated.

- **Reads:** local everywhere → fast.
- **Writes:** every write from every region travels to the *one* write region. A user in Sydney
  writing to a US-East leader eats ~200ms+ per write.
- **Consistency:** strong at the leader; replicas are eventually consistent (lag = replication delay).
- **Failover:** lose the write region → promote a replica (DR event, has RPO/RTO — Part H).

**Use when:** read-heavy, writes tolerate latency or are geographically concentrated, you want to
*avoid conflicts entirely*. This is the **simplest correct** multi-region data design — reach for it
first.

### 2. Active-active (multi-write)
Every region accepts writes locally and they replicate to each other.

- **Writes:** local → fast everywhere.
- **Reads:** local → fast everywhere.
- **Consistency:** the catch — two regions can write the *same key* concurrently → **conflicts** that
  must be resolved (Part D). Asynchronous replication means you've chosen AP (eventually consistent)
  *unless* you use a globally-consistent store that pays the latency tax synchronously (Part E).
- **Failover:** trivial in principle (other regions already live) — but a partition risks split-brain.

**Use when:** you need low write latency in multiple regions *and* the data model tolerates
conflict resolution (or is naturally conflict-free, like append-only events or per-user counters).

### 3. Partitioned by geo (each user/tenant homed in one region)
Shard the *data ownership* by geography. Each user's data lives in exactly one home region; that
region is the sole writer for that data. There is no overlap, so **there are no conflicts** — it's
single-write *per shard*, just with the shards spread across regions.

- **Writes:** local for users in their home region; cross-region only if a user travels (rare).
- **Reads:** local for home users; a roaming user pays cross-region or reads a cached copy.
- **Consistency:** strong *within* a home region (no two regions own the same data).
- **Bonus:** this is also the natural answer to **data residency** — pin EU users to EU.

**Use when:** data partitions cleanly by user/tenant/geo (most consumer and SaaS apps do!), and/or
you have residency requirements. This is often the **best** answer and candidates under-propose it.

### The comparison table (recite this shape)

| Strategy | Write latency | Read latency | Conflicts? | Consistency | Best for |
|---|---|---|---|---|---|
| **Single write + read replicas** | High (remote writers) | Low | None | Strong at leader, eventual at replicas | Read-heavy, concentrated writes, simplicity |
| **Active-active** | Low everywhere | Low everywhere | **Yes — must resolve** | Eventual (or pay for global strong) | Low write latency in many regions, conflict-tolerant data |
| **Geo-partitioned** | Low for home users | Low for home users | None (single owner per shard) | Strong within home region | Clean per-user partitioning, residency |

> **The write-latency problem, stated cleanly:** "You can have local reads everywhere cheaply via
> replicas. You *cannot* have local low-latency writes everywhere without either (a) accepting
> conflicts — active-active eventual — or (b) accepting that each piece of data has one home —
> geo-partitioned. The only way to get globally-consistent low-latency writes to the *same* key from
> everywhere is to violate physics, so no one offers it. Pick (a) or (b)."

---

## Part D — Active-active conflict handling

The moment you choose active-active, you owe an answer to "two regions wrote the same key — now
what?" (Ties to Topic 21 on advanced consistency / CRDTs.) Options, weakest to strongest:

| Technique | How it works | Cost / when it fails |
|---|---|---|
| **Last-Write-Wins (LWW)** | Attach a timestamp; highest wins | **Silently loses data**; needs synced clocks (HLC, not wall clock); fine for "latest profile photo," catastrophic for "account balance" |
| **CRDTs** (conflict-free replicated data types) | Data types whose merges are commutative/associative/idempotent → converge with *no coordination* (G-counters, OR-sets, LWW-registers, RGA for text) | Limited to expressible types; metadata overhead; great for counters, sets, collaborative text (Topic 21) |
| **Conflict-free by design** | Structure data so concurrent writes *can't* collide — append-only logs, per-region/per-user keyspaces, immutable events | Requires modeling discipline; the cleanest answer when achievable |
| **App-level / semantic resolution** | Detect conflict (vector clocks / version vectors), surface both versions, merge with business logic | Most correct, most work; e.g. shopping cart = union of items; Git-style merge |

> **Say this in the room:** "LWW is the default people reach for and it's a data-loss bug waiting to
> happen for anything that isn't last-writer-truly-wins. For counters or sets I'd use a CRDT so merges
> are deterministic with zero coordination. For real money I won't do active-active on that data at
> all — I'll geo-partition or single-write it so the conflict can't arise."

**The golden rule:** the cheapest conflict is the one you designed out of existence. Prefer
conflict-free structure (append-only, single-owner-per-key) over resolving conflicts after the fact.

---

## Part E — Globally-consistent stores vs eventually-consistent global stores

When you genuinely need strong consistency *across* regions, you must pay the cross-region latency
synchronously. Two product families; know when each is worth it.

### Globally strongly-consistent (pay the latency, get correctness)
- **Spanner / CockroachDB / YugabyteDB.** Data is sharded; each shard is a **Paxos/Raft group**
  replicated across regions. A write must reach a **quorum** of the group's replicas → if replicas
  span continents, a write commit includes a cross-region round trip (~hundreds of ms tail).
- **Spanner's TrueTime:** GPS + atomic clocks give a bounded-uncertainty clock; Spanner *waits out*
  the uncertainty (commit-wait) to guarantee external consistency (linearizability) globally.
- **CockroachDB:** uses **HLC** (hybrid logical clocks) instead of special hardware — slightly weaker
  guarantees, runs on commodity cloud.
- **The lever:** place a shard's replicas to control its write cost. Pin a shard's quorum to one
  region for fast local writes (sacrificing that shard's cross-region survivability), or spread it
  for survivability (paying latency). This is the real design knob.

### Eventually-consistent global (cheap, fast, no global ordering)
- **DynamoDB Global Tables, Cassandra multi-DC.** Multi-master, asynchronous cross-region
  replication. Local reads/writes are fast; convergence is eventual; conflicts resolved by **LWW**
  (Dynamo) or tunable quorum (Cassandra `LOCAL_QUORUM` for local strong, `EACH_QUORUM` for stronger).
- No global ordering, no cross-region transactions, but **never blocks on a cross-region hop**.

### When to pay for global strong consistency

| Pay for it (Spanner-class) | Don't (Dynamo/Cassandra-class) |
|---|---|
| Cross-region invariants that must hold *now* (global uniqueness, financial ledgers, inventory you can oversell) | Per-user data with no cross-user invariant |
| Correctness > latency, low/moderate write rate | High write rate, latency-sensitive, tolerates staleness |
| You can't tolerate any anomaly | Conflicts are resolvable or data is conflict-free |

> **Say this in the room:** "Global strong consistency is a checkbook decision — every strongly
> consistent cross-region write pays the round-trip. I reserve it for data with a *cross-region
> invariant* — global uniqueness, money, oversell-able inventory. For everything else (the 90%) I use
> a geo-partitioned or eventually-consistent store and keep writes local."

---

## Part F — Routing users to regions

How does a user's request reach the right region? (Ties to Topic 19, networking.)

| Mechanism | How it routes | Tradeoff |
|---|---|---|
| **GeoDNS** | DNS resolver returns a region's IP based on resolver's geo | Coarse (resolver ≠ user location); **DNS TTL caching delays failover** (clients hold stale IPs for the TTL) |
| **Anycast** | Same IP announced from many regions; BGP routes to nearest by network topology | Fast failover (BGP reconverges, no DNS TTL), used by CDNs/DNS; harder to do stateful right (flows can shift mid-connection) |
| **Latency-based routing** | DNS/global LB returns the region with lowest *measured* latency to the client | More accurate than pure geo; still DNS-TTL-bound for failover |
| **Global load balancer / app-layer** | A global front door (e.g. anycast edge) terminates and forwards to a healthy region | Centralizes routing + health-aware failover; the edge tier itself must be globally redundant |

### Failover-region mechanics
- Health checks at the global routing layer detect a region down → routing **withdraws** the dead
  region (DNS record change, or BGP withdrawal for anycast) → traffic shifts to the next-best region.
- **The DNS TTL trap:** with GeoDNS, clients/resolvers cache the IP for the TTL. Set TTLs low
  (30–60s) for routing records *intended* to fail over, but know that some clients ignore TTL. Anycast
  sidesteps this — failover is at the network layer, near-instant.
- **Capacity for failover:** if Region A fails over to Region B, B must have headroom to absorb A's
  traffic. Active-active with N regions means each carries ~1/N; lose one and the rest absorb it —
  size for `N-1`.

> **Say this in the room:** "I'll route with latency-based DNS for steady state, but I won't rely on
> DNS TTL for fast failover — I'll front it with an anycast/global LB that does health-aware routing,
> so a region going down shifts traffic in seconds, not minutes. And I'll size each region for N-1 so
> the survivors don't fall over from the redistributed load."

---

## Part G — Data residency & sovereignty

A *legal* constraint, not a performance one — and it overrides your other instincts.

- **The requirement:** EU users' personal data must be stored (and sometimes *processed*) within the
  EU (GDPR); some jurisdictions (India, China, Russia, KSA) require in-country storage and forbid
  export. "The data may not leave the boundary" is the hard line.
- **The architecture:** **geo-partitioning is the answer.** Home each user's data in their compliant
  region and *do not replicate it across the boundary*. This is exactly Part C strategy 3 — residency
  and geo-partitioning are the same shape.
- **Modeling implications:**
  - A **region/home field** on every user (or tenant) record, set at signup, that determines the
    physical home of their data. Routing and data access both key off it.
  - **Split the schema:** PII / regulated data is geo-pinned; non-regulated metadata (e.g. a global
    username uniqueness index) may live in a global store. You often need a thin **global directory**
    ("which region homes user X?") that itself contains no regulated data.
  - **Cross-border reads** (a roaming EU user hitting a US edge) must route back to the home region or
    be denied — you cannot cache regulated data outside the boundary.
  - **Backups, logs, and analytics** also carry the data — they must respect the boundary too. This is
    the part people forget; the audit catches it.

> **Say this in the room:** "Residency turns into geo-partitioning: each user has a home region set at
> signup, regulated data never crosses that boundary — including backups, logs, and the analytics
> pipeline. I keep a small global directory mapping user → home region (no PII in it) so routing knows
> where to send each request."

---

## Part H — Failover & DR across regions

Ties to Topic 13 (resilience). The two numbers that define DR:

- **RPO (Recovery Point Objective):** how much *data* you can afford to lose. Synchronous replication
  → RPO ≈ 0 (but pays write latency). Async replication → RPO = replication lag (seconds to minutes).
- **RTO (Recovery Time Objective):** how fast you must be *back up*. Active-active → RTO ≈ 0 (other
  regions already serving). Active-passive → RTO = time to detect + promote + redirect (minutes).

| Posture | RTO | RPO | Cost | Notes |
|---|---|---|---|---|
| **Active-passive (cold/warm standby)** | Minutes–hours | Async lag (sec–min) | Lower (standby underutilized) | Promote replica, repoint traffic; simplest DR |
| **Active-active** | ~0 | ~0 (sync) or lag (async) | Highest (full capacity ×N) | No promotion needed; split-brain risk on partition |

### The split-brain risk on a region partition
This is the deep-dive trap. If the link *between* regions fails (regions are up, can't see each
other), an active-active system can have **both sides accept writes** to the same data and diverge —
split brain. On reunion you have conflicting truths.

How to avoid it:
- **A coordination authority that needs a majority.** For data requiring a single owner, use a
  consensus group (Raft/Paxos) spanning **≥3 regions** so a minority side *cannot* elect itself
  leader / accept authoritative writes — it knows it lost quorum and steps down (CP behavior).
- **Fencing / leases.** A region only acts as writer while holding a valid, majority-granted lease.
- **Accept AP and reconcile.** For conflict-tolerant data, let both sides write and merge on heal
  (CRDTs / app resolution) — you chose availability over consistency knowingly (PACELC).

> **Say this in the room:** "The scary failure isn't a region dying — it's a region *partition* where
> both sides think they're in charge. For data that must have one owner I run a Raft group across
> three regions so a minority partition steps down and can't split-brain. For data that's
> conflict-free I let both sides take writes and merge on heal. The choice is CP vs AP, made
> per-dataset, not for the whole system."

### Traffic shifting
Don't fail over all-or-nothing. Shift a **percentage** of traffic and watch error rates / latency —
the same way you'd canary a deploy. Tooling: weighted DNS / global LB weights. Practice it: **DR you
haven't drilled is DR you don't have** (game days, regular failover exercises).

---

## Part I — Replication lag and the features it breaks

Async cross-region replication means a write in Region A isn't instantly visible in Region B. The lag
is normally seconds but spikes under load or link trouble. What it breaks:

- **Read-your-own-writes across regions.** User writes in A (routed there transiently), next request
  lands in B before replication → their own change is "missing." This is the #1 user-visible bug.
  Fixes:
  - **Sticky routing** — pin a user's session to their home region so their reads and writes hit the
    same place.
  - **Read-from-leader / read-your-writes tokens** — after a write, route that user's reads to the
    region that has the data (or carry a version/LSN and wait for the replica to catch up).
- **Monotonic reads.** Two reads from different regions can go *backwards* in time. Pin reads to a
  region or a consistency token.
- **Cross-region "see it immediately" features.** "User A in EU posts, User B in US sees it now" —
  there's a replication-lag floor; design for "eventually visible," show optimistic UI locally.

> **Say this in the room:** "Replication lag means I can't promise read-your-writes across regions for
> free. I'll pin a user's session to their home region so their own reads are consistent, and for the
> rare cross-region case I'll carry a write token and route reads to a region that's caught up.
> Everything else I let be eventually consistent and design the UX around it."

---

## Part J — Cell-based architecture & shuffle sharding (blast-radius reduction)

Multi-region limits the blast radius of a *region* failure. **Cell-based architecture** limits the
blast radius of a *software/poison* failure — a bad deploy, a hot tenant, a poison message — which a
region boundary does *not* contain (a bad deploy ships to all regions).

- **Cell = a fully independent, self-contained slice of the stack** (its own compute + data) serving a
  subset of users/tenants. Many cells per region. A failure (bug, overload, corrupt state) is
  contained to one cell → only that fraction of users affected.
- **A thin routing layer** maps each user/tenant → a cell (and stays simple/highly available so it
  isn't itself the SPOF).
- **Shuffle sharding:** instead of assigning each tenant to one cell, assign each to a *random small
  subset* of cells (a "shard" of, say, 2 of 8). With enough cells, the probability that any two
  tenants share their *entire* subset is tiny — so one abusive/poison tenant degrades only the few
  tenants who overlap, not everyone in a single shared cell. AWS uses this to isolate noisy neighbors.
- **Deploy safety:** roll a change cell-by-cell (and region-by-region) so a bad deploy is caught in
  cell 1 before it reaches cell 50.

| Boundary | Contains | Doesn't contain |
|---|---|---|
| **Region** | Datacenter/region infra failure | Bad global deploy, poison data, hot tenant |
| **Cell** | Bug, overload, corrupt state, hot tenant | (smaller unit, so very little escapes) |
| **Shuffle shard** | Noisy-neighbor / poison-tenant impact | — minimizes overlap between tenants |

> **Say this in the room:** "Multi-region protects against a region dying. It does *nothing* against a
> bad deploy or a poison record — that ships everywhere. So for blast-radius I add cells: independent
> stack slices serving a subset of users, deployed progressively. Shuffle sharding spreads tenants
> across random cell subsets so one bad tenant can't take everyone down."

---

## Part K — Decision guide: which strategy for which requirement

| Your requirement | Data strategy | Store class | Routing |
|---|---|---|---|
| **Only DR** (no latency/residency need) | Single-write + cross-region replica; promote on failure | Anything with cross-region replication (Postgres async replica, Aurora Global) | Failover DNS / global LB |
| **Low read latency, writes concentrated** | Single-write region + read replicas | Replicated SQL/NoSQL | Latency-based reads, writes home |
| **Low write latency in many regions, conflict-tolerant data** | Active-active, conflict-free or CRDT | DynamoDB Global Tables / Cassandra multi-DC | Latency/anycast, local writes |
| **Clean per-user partitioning** | Geo-partition (one home region per user) | Sharded-by-home store | Route by user's home region |
| **Data residency / sovereignty** | Geo-partition + no cross-border replication | Per-region store + thin global directory | Route by home region; deny/forward cross-border |
| **Cross-region invariant (money, global uniqueness, inventory)** | Globally strong, place quorum deliberately | Spanner / CockroachDB / YugabyteDB | Latency-based; accept write round-trip |
| **Limit blast radius of bad deploys / hot tenants** | Cell-based + shuffle sharding (orthogonal to all above) | Per-cell stacks | Tenant → cell mapping |

> **The meta-rule:** "Default to single-region multi-AZ. Add regions only when a *named* requirement
> forces it. Then pick the data strategy from the requirement, not from a desire to look
> sophisticated — geo-partitioning satisfies latency *and* residency at once for most apps and is the
> answer I reach for before active-active."

---

### Self-check before the mock (answer these from memory)
- [ ] Name the three reasons to go multi-region and why latency and residency pull in opposite directions.
- [ ] What is the cross-continent speed-of-light round-trip floor, and what can fix it? (trick — nothing)
- [ ] Why is multi-AZ *not* multi-region, and when should you stop at multi-AZ?
- [ ] Contrast the three data strategies (single-write replicas, active-active, geo-partitioned) on write latency and conflicts.
- [ ] State the write-latency problem in one sentence (why you can't have local strong writes to the same key everywhere).
- [ ] Active-active conflict options, weakest to strongest — and why LWW is dangerous.
- [ ] When do you pay for Spanner-class global strong consistency vs use DynamoDB Global Tables?
- [ ] Why is anycast better than GeoDNS for failover? What's the DNS TTL trap?
- [ ] How does data residency turn into geo-partitioning, and what does the global directory hold?
- [ ] Define RTO and RPO; map active-passive vs active-active onto them.
- [ ] Explain split-brain on a region partition and two ways to prevent it (CP via quorum vs AP via merge).
- [ ] What breaks read-your-own-writes across regions, and how do you fix it?
- [ ] What does a cell contain that a region boundary doesn't, and what is shuffle sharding for?
