# Topic 28: Infrastructure, Containers, Orchestration & Deployment

> **Why this topic exists:** Most of your design lives in logical boxes — services, queues, stores.
> But the interviewer who's run production will eventually ask *"and how does this actually run?"*
> The honest answer to "how does it run" is almost always **containers on an orchestrator, deployed
> through a pipeline, scaled by a controller, declared as code.** You rarely *lead* with this in an
> interview — you stay logical (see Part H) — but when the follow-up comes ("how do you deploy a
> change without downtime?", "what happens when a node dies?", "how do you scale this to handle the
> Black Friday spike?"), the staff signal is answering it crisply *and immediately returning to the
> logical design.* This doc is the layer underneath every box you've drawn so far.

---

## Part A — The Compute Spectrum

Everything you deploy runs on *something*. There's a spectrum from "you manage the metal" to "you manage nothing but a function." Moving right trades **control and isolation** for **density, elasticity, and lower ops burden** — and introduces **cold starts**.

| | **Bare metal** | **VMs** | **Containers** | **Serverless / FaaS** |
|---|---|---|---|---|
| **Isolation unit** | Physical machine | Hardware-virtualized OS | Shared kernel, namespaced | Function invocation |
| **Isolation strength** | Total | Strong (hypervisor) | Process-level (weaker) | Provider-managed sandbox |
| **Startup** | Minutes (provision) | ~10s–min (boot OS) | ~100ms–s (start process) | **Cold start: 100ms–several s** |
| **Density** | 1 workload/box | ~10s VMs/host | 100s containers/host | Thousands of fn instances |
| **Idle cost** | You pay always | You pay always | You pay always | **Scale-to-zero: pay per call** |
| **Ops burden** | Highest (you own HW) | High (you own OS) | Medium (you own image) | Lowest (you own code) |
| **Best for** | Ultra-low-latency, compliance, GPU farms, predictable heavy load | Strong-isolation tenants, legacy, full OS control | Most modern services — the default | Spiky/event-driven, glue, low-traffic endpoints |

> **Say this in the room:** *"The decision isn't 'which is best' — it's 'how predictable and how
> isolated does this workload need to be.' Steady high-throughput → containers on owned nodes.
> Spiky and embarrassingly parallel → serverless. Hard multi-tenant isolation boundary → VMs or
> separate node pools."*

### Why containers became the default
A container is **your code + its dependencies + its runtime, packaged as an immutable image** and run as a namespaced/cgroup-limited process on a shared kernel. The three properties that won:

- **Immutable** — the image is a fixed artifact. "Works on my machine" dies because the machine *is* the image. You promote the same bytes from dev → staging → prod (Part F, artifact promotion).
- **Portable** — same image runs on any host with a container runtime. Decouples app from infra.
- **Dense** — no per-workload OS; 100s per host. Big cost win vs VMs.

**The image/registry model** (know this conceptually):
- An image is built in **layers** (a base layer, deps layer, app layer). Layers are content-addressed and **cached/shared** — pulling a new version only fetches changed layers. This is why builds and deploys are fast.
- Images are pushed to a **registry** (ECR, GCR, Docker Hub, Artifactory). The registry is the source of truth for "what can run." Tag with **immutable digests (sha256), not `latest`** — `latest` is a moving target and the #1 cause of "why is prod running different code than staging."
- A **runtime** (containerd, CRI-O) pulls the image and runs it. Docker is the dev-time tool; production orchestrators talk to the lower-level runtime.

> **Tradeoff to name:** containers share the host kernel → **weaker isolation than VMs.** A kernel
> exploit crosses the boundary. For hostile multi-tenancy (you run *untrusted* user code), reach for
> VM-level isolation (Firecracker microVMs, gVisor, Kata) — that's how Lambda/Fargate actually
> isolate tenants under the hood.

---

## Part B — Orchestration (Kubernetes) at the Level That Matters

You don't get points for reciting `kubectl` flags. You get points for understanding **the model**: Kubernetes is a **desired-state reconciliation engine.** You declare what you want; controllers continuously drive reality toward it. That single idea explains almost everything.

### Control plane vs data plane
| Plane | Components | Job |
|---|---|---|
| **Control plane** ("the brain") | **API server** (front door, all reads/writes go through it), **etcd** (the only stateful component — a Raft-backed KV store holding *all* cluster state), **scheduler** (decides which node a pod lands on), **controller manager** (runs reconciliation loops) | Decide *what should be true* and persist it |
| **Data plane** ("the muscle") | **Nodes** running **kubelet** (the per-node agent that starts/stops containers and reports health) + **kube-proxy** (programs node networking) + a container runtime | Actually *run* the workloads |

> **Say this in the room:** *"etcd is the single source of truth and the thing whose availability and
> consistency I care about most — it's a Raft consensus store (cross-ref doc 8). Lose quorum on
> etcd and you can't make changes, though already-running workloads keep running. The control plane
> being down doesn't take down the data plane — that's a deliberate decoupling."*

### The object hierarchy (what you'll actually reference)
- **Pod** — the smallest deployable unit: one (or a few tightly-coupled) containers sharing network + storage. Pods are **ephemeral and disposable** — never treat one as durable (this is "cattle not pets," Part E).
- **ReplicaSet** — "keep N copies of this pod alive." A controller that reconciles count.
- **Deployment** — manages ReplicaSets to give you **declarative rolling updates and rollback** (Part D). This is what you actually deploy stateless services as.
- **Service** — a *stable virtual IP + DNS name* in front of a set of ephemeral pods (selected by label). Pods come and go; the Service name is constant. This is in-cluster service discovery (Part C).
- **Ingress / Gateway** — L7 entry from outside the cluster: host/path routing, TLS termination, fronting Services (cross-ref doc 9 for the API-gateway role).

### The reconciliation / desired-state model (the core idea)
You `apply` a manifest: *"I want 5 replicas of v2."* That's written to etcd. A control loop notices **current state (3 replicas of v1) ≠ desired state** and takes corrective action — spin up v2 pods, drain v1 — repeating forever. A node dies → its pods vanish → the ReplicaSet controller observes a shortfall → schedules replacements elsewhere. **Self-healing is just reconciliation running on a loop.** No human paged for a dead node.

> This is why declarative beats imperative for infra: you state the *goal*, not the *steps*, and the
> system continuously closes the gap — including after failures you didn't anticipate.

### Scheduling + bin-packing (conceptual)
Each pod declares **resource requests** (what it's guaranteed) and **limits** (its ceiling). The scheduler is a **bin-packer**: for each unscheduled pod it (1) **filters** nodes that can't fit / don't match constraints, then (2) **scores** the survivors (spread for HA? pack tight for cost?) and picks the best.

- **Requests drive scheduling** (how much room a node "appears" to have); **limits drive runtime enforcement** (CPU throttled, memory over-limit → **OOM-killed**).
- Levers: **affinity/anti-affinity** ("spread replicas across AZs"), **taints/tolerations** ("only GPU jobs on GPU nodes"), **topology spread** (don't put all replicas in one rack).

### Namespaces, limits, and the noisy-neighbor problem
- **Namespaces** = soft logical partitions (per-team, per-env) with **ResourceQuotas** (caps total CPU/mem a namespace may consume) and **LimitRanges** (defaults/ceilings per pod).
- **Noisy neighbor:** containers share a kernel and a node. A pod with no limit can starve co-located pods of CPU or hog page cache/IO. **Mitigation:** always set requests *and* limits; for hard isolation use **separate node pools** (Part G) or anti-affinity to keep tenants off shared nodes.

> **The interview-relevant takeaway:** "requests vs limits" is the lever for both **bin-packing
> density (cost)** and **noisy-neighbor isolation (reliability)**. Set requests too high → wasted
> capacity, low density, high cost. Too low → overcommit, contention, OOM kills. This tension recurs
> in Part E (autoscaling headroom).

### Horizontal Pod Autoscaler (the orchestrator's reactive scaler)
The **HPA** is a reconciliation loop: watch a metric (CPU, RPS, queue depth), compare to a target, adjust replica count. It's the in-cluster expression of Part E's reactive autoscaling. (There's also **VPA** — right-sizes requests of a single pod — and the **Cluster Autoscaler** — adds/removes *nodes* when pods can't be scheduled. Three different layers: pods, pod-size, nodes.)

---

## Part C — Service Discovery & Networking Inside the Cluster

Pods are ephemeral and get new IPs constantly, so you never hardwire IPs. Discovery is by **name**.

- **Cluster DNS** — every Service gets a DNS name (`payments.prod.svc.cluster.local`). Resolve the name → get the Service's stable virtual IP → traffic is load-balanced across the live backing pods. This is the in-cluster equivalent of a service registry (cross-ref doc 16).
- **In-cluster load balancing** — kube-proxy (or eBPF dataplanes like Cilium) spreads requests across healthy pod endpoints. This is L4 by default; the **service mesh** adds L7.
- **Service mesh recap (cross-ref doc 16)** — a **sidecar proxy** (Envoy) next to each pod, controlled centrally, that takes over mTLS, retries, timeouts, circuit breaking, and traffic-splitting **off the application code**. In design terms: the mesh is *where you do canary % splits and per-route resilience policy* (cross-ref doc 13) without redeploying the app. **Tradeoff:** a proxy hop per call (latency + resource overhead) and real operational complexity — don't reach for it on a 3-service system; it earns its keep at dozens-to-hundreds of services.

> **Say this in the room:** *"East-west service-to-service traffic resolves via cluster DNS to a
> stable Service VIP that load-balances over healthy pods; north-south traffic comes through Ingress.
> If I need uniform mTLS and traffic-shaping for canaries across many services, that's the mesh's
> job, not the app's."*

---

## Part D — Deployment Strategies, Operationally

You covered these conceptually earlier; here's the **operational mechanics** and how the orchestrator makes each safe. Tie to observability (cross-ref doc 27) for the signals and resilience (cross-ref doc 13) for the safety net.

| Strategy | Mechanic | Cost | Blast radius | Rollback |
|---|---|---|---|---|
| **Rolling** | Replace pods batch-by-batch (maxSurge/maxUnavailable) | Low (≈1× capacity) | Gradual; mixed versions live at once | Roll forward to old ReplicaSet (slow-ish) |
| **Blue/green** | Stand up full v2 ("green") beside v1 ("blue"), flip the LB/Service | High (2× capacity briefly) | All-or-nothing at cutover | **Instant** flip back to blue |
| **Canary** | Route a small % to v2, watch metrics, ramp 1%→5%→50%→100% | Low–medium | Tiny (only the canary %) | Shift traffic back to 0% |
| **Feature flags** | Ship code dark, toggle behavior at runtime per user/segment | ~0 infra | Per-flag, per-cohort | Flip the flag (no redeploy) |

Key distinctions to articulate:
- **Deployment ≠ release.** Rolling/blue-green/canary control *what binary runs*; **feature flags decouple deploy from release** — code ships disabled, then you turn it on for 1% of users, independent of the deploy. This is why mature teams deploy continuously but release deliberately.
- **Progressive delivery** = canary + flags + **automated analysis.** A controller (Argo Rollouts, Flagger) ramps traffic *and* watches SLO metrics (error rate, p99 latency, saturation — cross-ref doc 27) at each step. If a metric breaches threshold → **automated rollback**, no human in the loop. This is the operational closing of the loop: deploy → observe → auto-revert.
- **Database migrations are the hard part of every rollout** (cross-ref doc 10/20). Mixed-version traffic means **the schema must be compatible with both old and new code at once** → expand/contract (add column → backfill → deploy code that writes both → deploy code that reads new → drop old). You can roll back code instantly; **you cannot roll back a dropped column.** Say this — it's a senior tell.

> **Say this in the room:** *"I default to canary with automated metric analysis and rollback for
> services, blue/green when I need an instant clean cutover and can afford 2× capacity, and feature
> flags to separate release from deploy so I can dark-launch and ramp by cohort. Migrations go
> expand/contract so every step is backward-compatible."*

---

## Part E — Autoscaling, In Depth

This is where infra knowledge pays off in interviews, because scaling is the whole point of system design. The frame: **match capacity to demand without paying for peak 24/7 and without falling over when demand jumps.**

### Reactive vs predictive vs scheduled
| Type | Trigger | Strength | Weakness |
|---|---|---|---|
| **Reactive (metric-based)** | Live signal: CPU%, RPS, **queue depth/lag**, p99 latency, concurrency | Simple, no forecast needed, follows reality | **Always lagging** — reacts *after* load arrives; can't beat scale-up latency |
| **Predictive** | ML/trend forecast of future load | Provisions *ahead* of the spike | Wrong forecasts → over/under-provision; complex |
| **Scheduled** | Clock/calendar (scale up 8:50am, before market open / flash sale) | Dead simple, perfect for *known* patterns | Useless for unexpected spikes |

**Picking the metric matters.** CPU is the lazy default and often wrong. For a queue worker, scale on **queue depth or consumer lag** (cross-ref doc 7) — that *directly* measures backlog. For a latency-bound web tier, scale on **RPS or concurrency**, because CPU can look fine while you're blocked on a downstream. *Scale on the thing that actually represents the work.*

### The cold-start + scale-up-lag problem (the crux)
Reactive scaling is fundamentally a **control loop with delay.** When load jumps, you must:
1. **Detect** (metrics scrape interval, ~15–60s) →
2. **Decide** (autoscaler evaluation + stabilization window) →
3. **Provision** (HPA adds pods in seconds; but if nodes are full, **Cluster Autoscaler must boot a node — minutes**) →
4. **Warm up** (image pull, JVM/runtime warmup, JIT, cache fill, connection-pool establish, mesh sidecar ready) → only *then* serving.

That total lag can be **seconds to minutes** — during which you're under-provisioned and shedding/queuing/erroring. **Serverless** trades this for a **per-request cold start**: first invocation on a fresh sandbox pays init cost (100ms–several seconds, worse for heavy runtimes/large deps).

**Mitigations to name:**
- **Over-provision headroom** — run at, say, 60% target utilization so you have slack to absorb the spike *while* new capacity warms. Directly trades **cost for safety** (cross-ref doc 24).
- **Pre-warmed / pre-pulled pools** — keep warm node pools or "pause" pods reserving capacity; provisioned-concurrency for FaaS to kill cold starts on the hot path.
- **Faster warmup** — smaller images, lazy init, readiness probes that gate traffic until truly ready.
- **Absorb the spike elsewhere** — a **queue** (cross-ref doc 7) turns a traffic spike into a *backlog* you drain at your own rate, so you scale on lag instead of dropping requests. Load shedding / rate limiting (cross-ref doc 13) protects you while you scale.

> **Say this in the room:** *"Autoscaling is a delayed control loop — by the time it reacts, you're
> already behind. So I size headroom for the worst spike I can't pre-warm, use a queue to convert
> spikes into drainable backlog, and pre-warm capacity for known events. Scaling is never instant;
> design assuming the lag."*

### Stateless vs stateful scaling
- **Stateless** scales trivially — any replica serves any request, so add/remove pods freely behind a load balancer. **Push all state out** (12-factor, Part E-config) to a DB/cache/object store so the compute tier is disposable.
- **Stateful** is hard — a node owns data/sessions, so adding a replica means **data placement and rebalancing** (cross-ref doc 4), and removing one means **safe handoff/drain** so you don't lose the only copy. You can't just `+1`. This is why the standard move is **make the service tier stateless and let the stateful tier (DB, Kafka) scale on its own, slower, more carefully managed path.**

### Scale-to-zero
Drop to **zero instances when idle**, spin up on first request. Great for low-traffic/internal/event-driven services and the big serverless cost win. **Tradeoff:** the first request after idle eats a **full cold start**, and "idle then sudden burst" is the worst case. Don't scale-to-zero anything latency-critical on the hot path.

---

## Part F — Stateful Workloads on Orchestrators

- **StatefulSet** — for pods that need **stable identity** (predictable name `db-0`, `db-1`), **stable storage** (each gets its own persistent volume that survives reschedule), and **ordered** startup/scaling. This is how you'd run Kafka, ZooKeeper, or a database on Kubernetes.
- **PersistentVolume / PersistentVolumeClaim** — decouples "I need 100GB of fast storage" (the claim) from the actual backing disk (EBS/PD/etc.). The volume's lifecycle is independent of the pod, so when `db-0` reschedules it **re-attaches the same disk.**

> **"Should you run databases on Kubernetes?" — know both sides, because it's a classic follow-up.**
>
> - **For:** one declarative control plane for everything; operators (CRDs) automate failover,
>   backups, version upgrades; great for dev/test and for teams already all-in on k8s.
> - **Against:** k8s was *designed for stateless, disposable* workloads; persistent storage,
>   reschedules, and the orchestrator's eagerness to **move pods** fight against a stateful system
>   that wants stable nodes and disks. Storage performance/consistency through the k8s storage layer
>   is fiddly. For a primary OLTP database, **managed services (RDS/Aurora/Cloud SQL/Spanner) are
>   usually the better answer** — let the cloud own the hard parts.
>
> **Staff answer:** *"Stateless tiers on k8s, always. For stateful, I lean managed services for the
> primary datastore and only run databases on k8s with a mature operator when there's a real reason —
> the orchestrator's whole disposition is 'pods are cattle,' which is exactly wrong for your primary
> data."*

---

## Part G — Infrastructure as Code, GitOps & Immutable Infra

### Cattle, not pets
- **Pets:** named, hand-tuned, irreplaceable servers you SSH into and nurse back to health. Configuration drifts; nobody can reproduce them; they become snowflakes.
- **Cattle:** identical, disposable, numbered instances. One misbehaves → you **kill it and let the system spin a fresh one** from the same definition. No SSH-to-fix.

This principle underlies everything else here. It only works if the *definition* is the source of truth, not the running machine.

### Immutable infrastructure
You **never modify a running server** — you build a new image/instance from a versioned definition and **replace** the old one. No in-place patching, no config drift, trivial rollback (boot the previous image). Containers are immutable infra by construction.

### Infrastructure as Code (Terraform)
Define infra — networks, clusters, DBs, IAM — as **declarative, version-controlled code**. Terraform reconciles a desired-state file against real cloud resources (same desired-state idea as k8s, one layer down).
- **Benefits:** reviewable in PRs, reproducible across environments, auditable history, disaster-recoverable ("rebuild the region from code").
- **Watch for:** **state file** management (it's the source of truth for what exists — protect/lock it), and **drift** when someone makes a manual console change (the cardinal sin — it breaks the "code is truth" guarantee).

### GitOps
Make **Git the single source of truth for both app and infra.** A controller (Argo CD, Flux) **continuously reconciles** the cluster to match the repo. You don't `kubectl apply` from your laptop — you **merge a PR**, and the controller pulls and converges. (Same reconciliation loop, now spanning the whole system.)
- **Wins:** every change is a reviewed, audited, revertible commit; the repo *is* the system; **rollback = `git revert`**; drift is auto-corrected.

### Configuration & secrets (12-factor)
- **Strict separation of config from code** — the *same image* runs in every environment; only injected **config** differs. Config comes from the **environment** (env vars / mounted ConfigMaps), never baked into the image. This is what makes artifact promotion (below) safe.
- **Secrets** (DB creds, API keys) go in a **dedicated secret manager** (Vault, AWS Secrets Manager, sealed/external secrets) — never in the image, never in Git plaintext, ideally short-lived and rotated. Pulled at runtime, mounted into the container.

### CI/CD pipeline & artifact promotion (high level)
```
commit → CI: build + test → build ONE immutable image (sha256) → push to registry
       → deploy to staging (same image) → run integration/smoke/e2e
       → promote SAME image to prod (canary → ramp) → automated metric gate → done | auto-rollback
```
The non-negotiable: **build the artifact once, promote the identical bytes through environments.** You never rebuild per environment — that reintroduces "works in staging, breaks in prod." Differences between environments live *only* in injected config (above). Promotion is "point prod at the digest staging just validated."

---

## Part H — Multi-Tenancy, Node Pools, and Multi-Region

### Multi-tenancy / isolation at the infra layer (cross-ref doc 16)
There's a spectrum of how hard you isolate tenants:
- **Soft (namespace + quotas)** — cheapest, densest; tenants share nodes and a kernel. Fine for internal teams; **not** for hostile/untrusted tenants (noisy-neighbor + shared-kernel blast radius).
- **Node pools** — dedicate node groups per tenant/workload class (GPU pool, "sensitive-data" pool, "batch" pool). Tenants no longer share a host → real isolation, less density. Use taints/affinity to pin workloads to their pool.
- **Cluster-per-tenant / VM-per-tenant** — strongest isolation, highest cost/ops. For regulated or fully untrusted tenants.

> **Say this in the room:** *"Isolation is a dial against cost. Internal multi-tenant → namespaces +
> quotas. Distinct security or compliance boundary → separate node pools. Untrusted code or hard
> regulatory boundary → separate clusters or VM-level isolation."*

### How infra maps to multi-region (cross-ref doc 22)
The logical multi-region design (active-active vs active-passive, data replication, failover) sits *on top of* this infra layer:
- A **cluster (or several) per region**, fronted by **global routing** (GeoDNS / anycast / global LB) that sends users to the nearest healthy region.
- **IaC makes regions reproducible** — the same Terraform spins up region N+1; same images deploy there. "Add a region" becomes a config change, not a project.
- **Edge** (CDN / edge functions, cross-ref doc 14) pushes static + light compute *closer than your nearest region* — the outermost layer of the same "move compute toward demand" idea.
- The genuinely hard part remains **stateful** (cross-ref doc 22): cross-region data replication, consistency, and failover. Infra (clusters, IaC, routing) is the easy 80%; the data tier is the 20% that's the real design.

---

## Part I — How Much Infra to Bring Into the Interview

This is itself a staff signal: **knowing the layer exists and staying logical anyway.**

- **Default: stay logical.** Draw services, queues, stores, load balancers. Do **not** open with "we deploy via Argo on EKS with Karpenter node pools." That's noise that buys no design points and eats your clock.
- **Go down a layer only when it's load-bearing or asked.** Triggers worth surfacing proactively:
  - **"Zero-downtime deploy?"** → rolling/canary + expand-contract migrations (Part D).
  - **"Handle a 10× spike?"** → autoscaling lag, headroom, queue-as-buffer (Part E).
  - **"A node/AZ dies?"** → reconciliation self-heals pods, anti-affinity spread, multi-AZ (Parts B/H).
  - **"How do you roll back?"** → image digests + GitOps revert + canary auto-rollback (Parts D/G).
- **Two sentences, then back up.** Answer the infra follow-up crisply and **immediately return to the logical design.** The skill is *bounded depth on demand*, not a Kubernetes monologue.

> **Mental model for the room:** logical design = *what* and *why*; infra = *how it actually runs.*
> Lead with the former; produce the latter on demand, then resurface. Going down the stack
> unprompted reads as junior; refusing to when asked reads as a gap.

---

### Self-check before the mock (answer these from memory)
- [ ] Walk the compute spectrum bare-metal → serverless and name what you trade at each step.
- [ ] Why containers? Name the three properties, and the image/registry/layer model.
- [ ] Control plane vs data plane — name the four control-plane components and what etcd is.
- [ ] Explain the desired-state reconciliation loop and how it gives you self-healing.
- [ ] Requests vs limits: which drives scheduling, which drives runtime, and how each relates to noisy-neighbor and cost.
- [ ] Rolling vs blue/green vs canary vs feature flags — cost, blast radius, rollback for each.
- [ ] Why is deploy ≠ release, and what is progressive delivery?
- [ ] Why are database migrations the hard part of any rollout? (expand/contract, and what you can't roll back)
- [ ] Reactive vs predictive vs scheduled autoscaling; what metric do you scale a queue worker on, and why not CPU?
- [ ] Explain the scale-up-lag control loop and three ways to mitigate it (and the cost tradeoff).
- [ ] Why is stateless trivial to scale and stateful hard? Why is "databases on k8s" debated?
- [ ] Cattle not pets, immutable infra, IaC, GitOps — and "build once, promote the same artifact."
- [ ] How do you isolate multi-tenant workloads at the infra layer, cheapest → strongest?
- [ ] How much infra detail do you volunteer in an interview, and when do you go a layer deeper?
