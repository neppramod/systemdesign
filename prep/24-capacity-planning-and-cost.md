# Topic 24: Capacity Planning, Cost & Cloud Economics

> **Why this topic is a staff differentiator:** Almost every candidate can draw a *correct* design.
> Very few can tell you what it costs, and fewer still can hand you a cheaper variant for a stated
> relaxation of requirements. The senior signal is finishing a deep dive with: *"This is correct,
> and it runs about \$42k/month. The egress is 60% of that. If we can tolerate a 90% CDN offload and
> drop cross-region reads, it's \$16k/month."* That sentence — tying architecture to a dollar figure
> and back to a requirement — is the thing junior candidates never say. Capacity planning is the
> *forward* version of the same skill: turning the estimate from Topic 2 into a count of machines
> and a monthly bill before you've spent a cent.

---

## Part A — Capacity Planning: From Load Estimate to Instance Count

The method is a pipeline. Each stage feeds the next, and the whole thing starts from the estimate
you already did in **Topic 2**.

```
load estimate (QPS, data, bandwidth)   ← Topic 2
        │
        ▼
per-unit resource cost (CPU/mem/IOPS/net per request)
        │
        ▼
resource demand at peak  =  peak QPS × per-request cost
        │
        ▼
÷ usable capacity per instance (after utilization target)
        │
        ▼
× headroom / redundancy factor (N+1, N+2, multi-AZ)
        │
        ▼
instance count  →  $/month
```

### 1. Start from the load estimate

From Topic 2 you have: average QPS, **peak QPS (2–3× average)**, payload sizes, data volume/yr.
Capacity planning is sized for **peak**, not average — the system has to stand up on Black Friday,
not on a Tuesday at 3am. The gap between peak and average is exactly what autoscaling exists to
recover (Part E).

> **Say this in the room:** "I size compute for peak, storage for cumulative retention, and
> bandwidth for peak — three different curves. Conflating them is the classic under-provisioning bug."

### 2. Resource sizing — the four axes

Every workload is bottlenecked on one of four resources. Find the binding constraint *first*; sizing
the other three to match it is wasted money.

| Resource | What pins it | Symptom when it's the bottleneck | Typical offenders |
|---|---|---|---|
| **CPU** | request compute, serialization, TLS, compression | high CPU%, latency climbs with load | API services, encoding, ML inference |
| **Memory** | working set, cache, in-flight requests, JVM heap | OOM kills, GC thrash, swap | caches, in-memory joins, JVM apps |
| **IOPS / disk** | random reads/writes, fsync, WAL | disk queue depth, p99 spikes | databases, log stores |
| **Network** | payload × QPS, replication, egress | NIC saturation, retransmits | media, fan-out, replication-heavy DBs |

The estimate per request is back-of-envelope, not a benchmark: *"each feed read touches ~3 cache
gets (sub-ms) + serializes ~20 KB → CPU-bound at a few ms, network ~20 KB."* You don't need a load
test in the room; you need a defensible per-request cost so the multiplication means something.

### 3. Usable capacity per instance — and why you don't run at 90%

A box rated for "100% CPU" is not a box you can fill to 100%. You target **60–70% utilization at
peak** and treat the rest as a buffer. The reasons, stated as you'd say them:

- **Queueing theory is brutal near saturation.** Latency under an M/M/1-ish model scales as
  `1/(1 − ρ)` where ρ is utilization. At 50% load, latency is ~2× the unloaded baseline; at 90% it's
  ~10×; at 95% it's ~20×. **The last 10% of utilization buys you a cliff, not capacity.** This is the
  single most important number in this doc.
- **Headroom for failure.** If you run 10 instances at 90% and lose one, the surviving 9 are now at
  100% → cascade. At 65%, losing one pushes the rest to ~72% — survivable.
- **Headroom for spikes faster than autoscaling can react.** Scaling takes 30s–several minutes
  (Part E). The buffer absorbs the spike *while* you scale.
- **Bursty GC / compaction / background jobs** eat into the headroom you thought was free.

> **The one-liner:** "I provision to ~65% peak utilization. Above ~70–80% latency goes non-linear,
> and I need slack to survive an instance loss without cascading."

### 4. Headroom & redundancy — N+1, N+2, multi-AZ

On top of the utilization buffer you add **redundancy** for instance/AZ failure:

- **N+1**: provision one extra instance beyond what peak needs, so one can die with no degradation.
- **N+2** or **2N**: for critical tiers, or when a single AZ failure must be invisible.
- **Multi-AZ**: spread across ≥3 AZs so losing one AZ loses ≤1/3 of capacity. To survive a full AZ
  outage at peak you must run each AZ at ≤66% so the other two absorb the load — this is *why
  multi-AZ HA roughly costs you 50% more compute* (you carry a third you only need during failure).

### 5. Worked instance count

Say a feed-read service: **peak 50k QPS**, each request costs ~5 ms of CPU on a 4-vCPU box.

- One 4-vCPU box at 100% does `4 cores / 0.005 s = 800 req/s`. At **65% target → ~520 req/s usable**.
- Instances for demand: `50,000 / 520 ≈ 97`.
- Spread across 3 AZs, each AZ at ≤66% so it survives losing one AZ: divide usable by 0.66 again
  → ~**147 instances**, round to **150** (50 per AZ).
- That's the number you put a price on. (See Part J for the full bill.)

---

## Part B — Cost Is a Property of Every Design Decision

The mental shift: **there is no "free" architectural choice — every box, arrow, and replica has a
price tag.** The five cost dimensions, and which design choices move each:

| Dimension | Priced on | Design choices that move it |
|---|---|---|
| **Compute** | instance-hours (vCPU, RAM) | replication factor, redundancy, sync vs async work, language efficiency |
| **Storage** | \$/GB-month × tier × replication | retention, replication factor, hot/warm/cold tiering, format/compression |
| **Network egress** | \$/GB out (the silent killer) | cross-AZ, cross-region, internet egress, CDN offload, chatty services |
| **Managed-service premium** | per-request / per-instance markup | build-vs-buy, serverless vs self-hosted |
| **Request-based pricing** | per-million requests / per-invocation | API Gateway, Lambda, S3 GET/PUT, DynamoDB RCU/WCU |

The discipline is to **annotate the architecture with these as you draw it.** "This replica is for
read scaling — it's another \$X. This cross-region replica is for DR — it doubles storage *and* adds
egress on every write." When you narrate cost alongside function, the interviewer sees a staff
engineer who has actually owned a budget.

---

## Part C — Egress Economics (the part everyone gets wrong)

This is the single most under-appreciated line item in cloud bills, and the fastest way to signal
seniority. **Inbound traffic is almost always free. Outbound (egress) is where the money goes.** And
egress is tiered by *how far the bytes travel*.

### The egress ladder (AWS-ish ballparks; GCP/Azure are similar shape)

| Traffic path | Rough \$/GB | Notes |
|---|---|---|
| **Inbound (internet → cloud)** | \$0.00 | almost always free |
| **Same-AZ, private IP** | \$0.00 | free — keep traffic here when you can |
| **Cross-AZ (within region)** | ~\$0.01/GB **each way** | charged on *both* send and receive — ~\$0.02/GB round trip |
| **Cross-region** | ~\$0.02/GB | replication, DR, multi-region reads |
| **Internet egress (to users)** | **~\$0.05–0.09/GB** (tiered down at volume) | the big one; what your CDN offsets |
| **Via NAT Gateway** | + ~\$0.045/GB processing **on top** | a hidden tax on private-subnet egress |

### Why egress dominates many bills

A media or feed app serving lots of bytes to users will see **internet egress as the largest single
line item — often bigger than all compute.** Consider 1 PB/month served to users at \$0.05–0.09/GB:
that's **\$50k–\$90k/month in egress alone**. Compute to serve it might be \$10k. *The bytes leaving
your cloud cost more than the machines producing them.*

> **The lever — say this:** "Egress is my biggest cost here, so the CDN isn't a latency optimization,
> it's a *cost* optimization. Every byte the CDN serves from edge is a byte I don't pay origin egress
> on. At a 95% hit ratio I'm paying origin egress on 5% of traffic." (CDN egress is itself cheaper
> than direct cloud egress, *and* you only pay origin for misses — double win. See Part F.)

### Cross-AZ: the silent internal killer

Cross-AZ traffic feels "internal" so people forget it's metered. But a chatty multi-AZ design pays
**both directions** on every hop:

- **Replication across AZs** (DB leader→follower in another AZ): every write byte is egress, ×
  replication factor.
- **Service-to-service calls that ignore AZ affinity**: a request that bounces A→B→C across AZs pays
  egress on each leg. At scale this is real money and it's *invisible* in the architecture diagram.
- **Kafka/streaming across AZs**: producers and consumers in different AZs pay both ways on every
  message — often the surprise on a streaming bill.

Mitigations to name: **topology-aware / same-zone routing** (route to a same-AZ replica first), keep
high-volume chatter in-AZ, compress payloads, batch. The CAP-style cost: same-AZ routing trades some
load-balancing evenness and a sliver of availability for a much smaller egress bill.

### Cross-region: pay it on purpose

Cross-region egress shows up in **multi-region active-active** (every write replicated both ways) and
**DR** (continuous replication to a standby region). It's a *deliberate* cost you buy for
availability or latency — make sure the requirement justifies it. "Active-active across two regions
roughly doubles write egress and storage; I'd only do it if we need <50ms reads on both coasts *and*
region-failure survival. Otherwise active-passive DR is far cheaper."

> **The hierarchy to recite:** **same-AZ (free) < cross-AZ (cheap, ×2) < cross-region (more) <<
> internet egress (most).** Keep traffic as low on this ladder as your requirements allow.

---

## Part D — Storage Cost: Tiers, Replication, Retention

### \$/GB across tiers (object-storage ballparks, AWS S3-ish, per GB-month)

| Tier | \$/GB-mo | Retrieval | Use for |
|---|---|---|---|
| **Hot** (S3 Standard, SSD block) | ~\$0.023 | instant, free | active data, working set |
| **Warm** (S3 Standard-IA, Infrequent Access) | ~\$0.0125 | instant, small per-GB retrieval fee | data read occasionally |
| **Cold** (S3 Glacier Flexible) | ~\$0.0036 | minutes–hours, retrieval fee | compliance, backups |
| **Frozen** (Glacier Deep Archive) | ~\$0.00099 | hours | "we legally must keep it 7 years" |

Block SSD (EBS gp3) is ~\$0.08/GB-mo — **~3.5× object storage** — and you also pay for provisioned
IOPS. Lesson: don't park cold data on fast block storage. **Lifecycle policies** that auto-tier
hot→warm→cold are nearly free money; not having them is the most common storage-cost smell.

> **The trap:** retrieval fees + per-request fees on cold tiers. Glacier is cheap to *store* and
> expensive to *read*. If you'll read it monthly, IA beats Glacier despite the higher \$/GB. Tier by
> **access frequency**, not by age alone.

### Replication factor multiplies everything

Storing 100 TB at **RF=3** is 300 TB of physical storage. Replication is non-negotiable for
durability, but it's a **literal multiplier on your storage bill** and on replication egress. Levers:

- **Erasure coding** (e.g., 6+3) gives RF≈1.5× overhead instead of 3× for the same durability — at
  the cost of CPU on reconstruction and worse small-object/latency behavior. Great for cold/large
  objects, bad for hot small reads.
- **Tiered replication**: keep RF=3 hot, RF=2 + erasure-coded cold.

### Retention compounds

This is the one that sinks budgets quietly. Storage cost isn't `writes/day × size` — it's the
**integral over retention.**

```
steady-state storage  ≈  daily_ingest × retention_days   (until you delete)
monthly growth bill   ≈  daily_ingest × 30 × $/GB-mo  (added every month, forever, if infinite retention)
```

10 GB/day with **infinite retention** is 3.65 TB/yr *growing without bound* — year 5 you're storing
~18 TB and the bill only goes up. **A retention policy is a cost-control mechanism**, not just a
compliance one. Always ask "how long do we keep it?" and price the *steady state*, not month one.

### Build vs buy for storage

Self-managing Cassandra/Ceph/MinIO can beat S3 on raw \$/GB at huge scale — but you pay in SRE
headcount, replication ops, and capacity planning. **For most designs, managed object storage wins on
TCO** because the eng time to run it reliably costs more than the premium (see Part H).

---

## Part E — Compute: Pricing Models, Autoscaling, Right-Sizing

### Reserved vs On-Demand vs Spot

| Model | Discount vs on-demand | Commitment | Use for |
|---|---|---|---|
| **On-demand** | baseline (0%) | none | spiky/unpredictable, dev, short-lived |
| **Reserved / Savings Plans** | ~30–60% off | 1–3 yr | your **steady-state baseline** load |
| **Spot / preemptible** | ~60–90% off | can be reclaimed with ~2 min notice | fault-tolerant, stateless, batch, async workers |

**The standard pattern, say it:** "I'd cover the predictable baseline with **reserved/savings
plans** (~40% off), absorb the daily peak with **on-demand** autoscaling, and run **batch and
stateless async work on spot** (~70% off) with checkpointing so reclamation is cheap. That blend
typically cuts the compute bill 40–60% versus all-on-demand."

> **Spot's catch:** it can vanish in ~2 minutes. Never put a stateful leader or a synchronous
> user-facing tier with no failover on spot. Perfect for Kafka consumers, video transcoding, ETL,
> ML training with checkpoints.

### Autoscaling: reactive vs predictive, and cold start

- **Reactive** (target-tracking on CPU/QPS): scales *after* load rises. Simple, but always lagging —
  you scale *up* during the spike, after p99 has already suffered. The utilization headroom in
  Part A is what covers this lag.
- **Predictive / scheduled**: pre-warm capacity ahead of known patterns (the 9am login surge, the
  ad campaign, Black Friday). Eliminates the lag for *predictable* demand. Most mature shops do
  **predictive for the daily curve + reactive for surprises.**
- **The cold-start problem**: new instances aren't instantly useful — VM boot + app init + JIT warmup
  + cache fill can take seconds to minutes. Serverless cold starts (Lambda) are 100ms–several
  seconds. This is *why* reactive autoscaling can't be your only defense and why you keep warm
  headroom. Mitigations: pre-warmed pools, provisioned concurrency (you pay to keep it warm — a
  direct latency-for-cost trade), smaller/faster-booting images.

### Serverless vs Containers vs VMs — the cost crossover

These don't have a universal winner; they **cross over by utilization**:

| Model | Pricing | Cheapest when | Watch out for |
|---|---|---|---|
| **Serverless** (Lambda) | per-invocation + per-GB-second | **low / spiky / bursty** traffic, idle most of the time | gets expensive at sustained high QPS; cold starts; per-request fees |
| **Containers** (ECS/Fargate/K8s) | per-task-hour or node-hour | **medium, variable** load; good bin-packing | orchestration complexity |
| **VMs** (reserved EC2) | per-instance-hour | **steady, high, predictable** load | you pay for idle; slow to scale |

> **The crossover line:** "Serverless wins until you're busy enough that you'd keep a box hot anyway.
> Past roughly 30–50% sustained utilization, a reserved container/VM is cheaper per request than
> Lambda. So: serverless for the spiky edges and infrequent jobs, reserved compute for the
> always-on core." Quantify if pushed: a function billed per-100ms run flat-out 24/7 costs far more
> than the equivalent reserved instance.

### Right-sizing

The most common waste: **over-provisioned instances running at 10% utilization** because someone
guessed big "to be safe." Right-sizing = measure actual usage, drop to the smallest instance that
holds peak + headroom, pick the family that matches your bottleneck (compute-optimized vs
memory-optimized vs storage-optimized — don't pay for RAM you don't use on a CPU-bound service).
Also: **ARM/Graviton** is ~20% cheaper per equivalent throughput for most workloads — free money if
your stack runs on it.

---

## Part F — Caching & CDN as Cost Optimizations

Caching is taught as a latency tool. At staff level you also frame it as a **cost** tool, because the
math is the same lever from both sides:

```
origin load        = total requests × (1 − hit_ratio)
origin cost (egress + compute)  ∝  origin load
```

A small move in hit ratio is a *large* move in origin load:

| Hit ratio | Origin load (fraction) | Relative origin cost |
|---|---|---|
| 0% (no cache) | 100% | 1.0× |
| 80% | 20% | 0.2× |
| 95% | 5% | 0.05× |
| 99% | 1% | 0.01× |

**Going from 80% → 95% hit ratio cuts origin load by 4×.** That's 4× less origin compute *and* 4×
less origin egress. For a media app where egress dominates (Part C), the CDN is the dominant cost
lever, full stop.

> **Say this:** "I'll quote the design's cost at the *expected* hit ratio, and call out that hit
> ratio is the most sensitive cost knob. The cheap variant is just 'invest in cacheability' —
> longer TTLs, cache-friendly URLs, collapsing requests — which costs eng time, not infra."

Same logic for an in-memory cache in front of a database: every cache hit is a query you don't pay
for in DB IOPS/CPU and (if cross-AZ) in egress. Caching is *the* highest-ROI cost optimization in
most read-heavy systems.

---

## Part G — The Latency / Cost / Consistency Triangle

The classic CAP/PACELC framing (Topic 5) has a **cost shadow** that candidates rarely name: **buying
lower latency costs money, and buying stronger consistency costs money.** Cost is the silent third
axis.

- **Stronger consistency costs more.** Quorum reads/writes hit more replicas → more compute and
  cross-AZ/region egress per operation. Synchronous cross-region replication for global strong
  consistency means every write pays a cross-region round trip *and* the egress. Linearizability is
  not free; it's a recurring per-operation tax.
- **Lower latency costs more.** Edge presence (CDN, regional replicas) costs money. More replicas =
  more reads served close = more infra. Provisioned concurrency to kill cold starts is latency you
  *buy*. Over-provisioned headroom to hold p99 under load is, again, money.
- **The cheap corner is: eventual consistency + relaxed latency.** Async replication, read-from-any-
  replica, generous TTLs. That's why "feed can be a few seconds stale" is such a powerful
  requirement — it lets you sit in the cheap corner.

> **The staff move:** "Tell me the latency and consistency targets and I'll tell you the cost. If we
> can relax read consistency to eventual and p99 to 300ms, I drop the cross-region quorum and the
> provisioned warm pool, and the bill roughly halves." *Every relaxed requirement is a discount.*

---

## Part H — Build vs Buy: TCO of Managed Services

Managed services charge a **premium** over raw infra — but the comparison is not "premium vs zero,"
it's **premium vs the fully-loaded cost of running it yourself.**

| Cost bucket | Self-hosted | Managed |
|---|---|---|
| Infra \$ | lower per unit | higher (the premium) |
| **Eng/SRE time** | high — patching, upgrades, on-call, capacity, backups, scaling | near-zero |
| Time-to-market | slow | fast |
| Failure blast radius | yours to debug at 3am | vendor's SLA |
| Lock-in / exit cost | low | higher |

The honest math: a self-managed datastore might save \$X/month in infra but cost **1–2 engineers'
loaded salary** (~\$200k–\$400k/yr each, all-in) to operate well. **The managed premium is almost
always cheaper than the headcount until you're at a scale where the premium exceeds a team's salary**
— at which point hyperscale shops do build their own (that's why Netflix, Uber, etc. self-host at the
bottom of their stacks).

> **Say this:** "I'd buy the managed version. The premium is real but it's less than the loaded cost
> of an on-call rotation to run it, and it gets us to launch faster. I'd revisit build-vs-buy only
> when the line item crosses ~a full team's cost — that's the crossover, and we're nowhere near it."

---

## Part I — Multi-Tenancy and the Cost of Availability

### Multi-tenancy = pooling (ties to Topic 16)

The cheapest way to serve many customers is to **pool** them on shared infrastructure rather than
**silo** each on dedicated infra. Statistical multiplexing wins: tenants peak at different times, so
pooled capacity sized for the *aggregate* peak is far less than the sum of per-tenant peaks.

| Model | Cost | Isolation | When |
|---|---|---|---|
| **Pooled** (shared everything, tenant_id column) | lowest — best utilization | weakest (noisy neighbor) | many small tenants, SaaS free/low tier |
| **Bridge** (shared compute, per-tenant schema/DB) | medium | medium | mid-market |
| **Silo** (dedicated stack per tenant) | highest — no multiplexing | strongest | enterprise, compliance, "single-tenant" SKU |

> **The lever:** "Pooling is the core SaaS cost advantage — I get utilization no single tenant could.
> I'd pool the long tail and offer silo as a *priced* premium SKU for enterprises who'll pay for
> isolation. Don't silo everyone by default; you throw away the multiplexing that makes SaaS
> economical." Noisy-neighbor risk is the tradeoff — bound it with per-tenant rate limits and quotas.

### Each extra 9 roughly multiplies cost

Availability is bought, and the price is roughly **super-linear per nine**:

| SLA | Downtime/yr | Roughly what it takes | Relative cost |
|---|---|---|---|
| 99% (two 9s) | ~3.65 days | single region, basic redundancy | 1× |
| 99.9% (three 9s) | ~8.8 hr | multi-AZ, health checks, auto-failover | ~1.5–2× |
| 99.99% (four 9s) | ~52 min | multi-region, automated failover, no SPOF | ~3–5× |
| 99.999% (five 9s) | ~5.3 min | active-active multi-region, heavy automation, chaos testing | ~5–10×+ |

Each nine roughly **adds redundancy (more idle capacity), more regions (more egress + storage), and
more engineering**. The discipline is to **match the SLA to business value**: an internal analytics
dashboard does not need five 9s; a payments authorization path might.

> **Say this:** "What's the cost of a minute of downtime? That number sets the SLA, and the SLA sets
> the bill. I won't pay for five 9s on a path where an hour of downtime costs us a support ticket.
> I'll spend the nines where downtime costs revenue or trust."

---

## Part J — A Worked Cost Estimate

Let's price the feed service from Part A — a read-heavy consumer feed. **The goal is a defensible
ballpark, not accounting.** State assumptions, multiply, total, then find the dominant term.

**Assumptions** (from a Topic 2 estimate): 50M DAU, peak 50k read QPS, each feed response ~20 KB,
plus media served separately at ~500 TB/month to users. Data: 100 TB hot, RF=3.

### Compute (feed service)

- From Part A: **~150 instances** (4-vCPU, multi-AZ, 65% target). Say c-family at ~\$0.15/hr
  on-demand equivalent.
- Cover baseline ~100 instances with reserved (~40% off → ~\$0.09/hr); peak ~50 on-demand.
- `100 × $0.09 × 730 hr ≈ $6,570` + `50 × $0.15 × 730 ≈ $5,475` → **~\$12,000/mo compute.**

### Storage

- 100 TB × RF=3 = 300 TB hot at ~\$0.023/GB-mo → `300,000 × 0.023 ≈ ` **\$6,900/mo.**
- (Lifecycle older data to IA/Glacier would cut this materially — flag it.)

### Network egress — the dominant term

- **Media to users: 500 TB/mo.** Without CDN at ~\$0.07/GB → `500,000 × 0.07 ≈ ` **\$35,000/mo.**
- **With a CDN at 95% hit ratio:** origin egress on 5% = 25 TB → `25,000 × 0.07 ≈ $1,750` origin,
  plus CDN delivery ~\$0.02–0.04/GB on 500 TB → `500,000 × 0.03 ≈ $15,000`. CDN total ≈ **\$16,750/mo**
  — and far better latency. **CDN saves ~\$18k/mo here while improving UX.**
- Cross-AZ replication + chatter: assume ~50 TB/mo cross-AZ at ~\$0.02/GB round trip → **~\$1,000/mo.**

### Managed services

- API Gateway / LB, managed cache (Redis), monitoring: lump **~\$3,000/mo** (request fees + instances).

### Total

| Line item | Without CDN | With CDN (95% hit) |
|---|---|---|
| Compute | \$12,000 | \$12,000 |
| Storage (hot, RF=3) | \$6,900 | \$6,900 |
| Media egress | \$35,000 | \$16,750 |
| Cross-AZ | \$1,000 | \$1,000 |
| Managed services | \$3,000 | \$3,000 |
| **Total / month** | **~\$57,900** | **~\$39,650** |

**The findings to narrate:**
1. **Egress is the largest term** (60% without CDN). The CDN is a *cost* lever, not just latency.
2. **The cheap variants:** lifecycle storage to IA (saves ~\$3k), push CDN hit ratio to 99% (saves
   most of the remaining origin egress), spot for any async/batch tier.
3. **State the relaxation:** "If we accept eventual consistency and serve feeds from same-AZ replicas,
   we cut cross-AZ and a chunk of compute headroom — roughly \$2–4k more."

> This — **a total, the dominant term, and a cheaper variant tied to a relaxed requirement** — is the
> whole skill. Most candidates never produce the number.

---

## Part K — How to Bring Cost Up in the Room (the seniority signal)

You don't need a price list memorized. You need to *reach for cost* at the right moments:

1. **After the high-level design:** "This is correct. Let me put a rough cost on it so we can see
   where the money is." Then find the dominant term.
2. **When proposing redundancy/consistency:** annotate it. "This cross-region replica is for DR — it
   doubles write egress and storage. Worth it only if region failure is in scope."
3. **Offer the cheaper variant unprompted:** "Here's the \$X/mo design. If we relax requirement Y to
   eventual consistency / 300ms p99 / single-region, here's the \$X/2 variant. Which matches the
   business?"
4. **Tie SLA to dollars:** "What does a minute of downtime cost? That sets how many nines I buy."
5. **Name the silent killers:** egress (cross-AZ, internet), infinite retention, idle over-provisioned
   instances, per-request fees at scale. Naming these unprompted is a clear staff tell.

> **The sentence that lands:** *"The design is correct but it's ~\$X/month, dominated by egress.
> Here's the variant that's half the cost if we can relax Y."* Function, cost, and a tradeoff in one
> breath.

---

### Self-check before the mock (answer these from memory)
- [ ] Walk the capacity-planning pipeline: load estimate → resource sizing → utilization target → headroom → instance count.
- [ ] Why don't you run at 90% utilization? (state the `1/(1−ρ)` intuition)
- [ ] Why does multi-AZ HA cost ~50% more compute?
- [ ] Recite the egress ladder: same-AZ vs cross-AZ vs cross-region vs internet egress, cheapest to most expensive.
- [ ] Why is the CDN a *cost* optimization, and how does hit ratio map to origin cost?
- [ ] Reserved vs on-demand vs spot — what runs on each, and the rough discounts?
- [ ] Where's the serverless↔reserved-compute cost crossover, and why?
- [ ] State the latency/cost/consistency triangle: which two cost money, and what's the cheap corner?
- [ ] Build vs buy: what's the real crossover point for self-hosting a datastore?
- [ ] How much does each extra nine of availability roughly cost, and how do you decide how many to buy?
- [ ] Produce a one-line worked-cost summary: total, dominant term, cheaper variant tied to a relaxed requirement.
