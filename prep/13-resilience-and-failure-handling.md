# Topic 13: Resilience, Failure Handling & Operating at Scale

> **Why this topic wins rounds:** Anyone can draw the happy path. Staff candidates are hired on the
> *unhappy* path — what happens when a dependency goes dark, a region drops, a deploy goes bad, or
> traffic 10×'s in 90 seconds. The interviewer's favorite move is "okay, now node X dies — what
> happens?" If your answer is a confident walk through timeouts → circuit breakers → degraded mode →
> recovery, you've signaled the thing they're actually screening for: that you can be trusted to run
> this in production. This doc is the failure-mode vocabulary you recite cold.

---

## Part A — The mindset: everything fails, design for it

The senior framing to *say in the room*: **"I don't design for things working; I design for the
moment they don't."** At scale, failure is not an edge case — it's the steady state. With 10,000
nodes and a per-node MTBF of three years, you have a machine dying roughly every few hours. Disks
fail, networks partition, GC pauses stall a process for 8 seconds, a dependency 99.9%-availability
service is down ~43 minutes a month, and a bad config rollout takes out a fleet in seconds.

So the goal is never "prevent all failure." It's: **contain it, detect it fast, degrade gracefully,
recover automatically.**

Three concepts to name early:

- **Failure domain** (a.k.a. fault domain): the set of things that fail *together*. A process, a
  host, a rack, an AZ, a region, a deploy unit, a shared dependency (one DB, one config service).
  Good design makes failure domains small and *independent*.
- **Blast radius**: how much of the system/users a single failure can take down. Your job is to
  shrink it. A failure that takes down 100% of users is a design smell; one that takes down 1 shard
  / 1 cell / 1 AZ is a design.
- **Graceful degradation**: when a dependency is gone, serve a *worse but valid* answer instead of
  an error. Feed can't reach the ranking service? Serve reverse-chronological. Recommendations
  down? Show popular items. The system gets *dumber*, not *dead*.

> **The line to say:** "Availability isn't a property of any one component — it's a property of how
> components fail *together*. I'll spend my design budget on isolating failure domains and shrinking
> blast radius, not on pretending components won't fail."

### The hierarchy of responses to a failing dependency
From best to worst, and you should reach for them in this order:

1. **Serve from cache / a degraded fallback** (stale-but-valid beats nothing).
2. **Fail fast with a sane default** (empty recommendations, not a 30s hang).
3. **Shed the request** (return 429/503 cheaply to protect the rest).
4. **Queue it for later** (only if the work is async-tolerable).
5. **Return an honest error** (last resort, and still fast).

A request that hangs is worse than a request that fails — it holds a thread, a connection, and a
caller's patience, and it's how local failures become global ones.

---

## Part B — Timeouts, retries, and the retry storm

### Timeouts: the foundation everything else sits on
**Every network call gets a timeout. No exceptions.** The default in most clients is *infinite* or
*absurdly high*, and an unbounded wait is the seed of every cascading failure: a slow dependency
ties up your threads/connections, your queue backs up, and you fall over even though nothing
"crashed."

- Set timeouts from your **latency budget**, not from the dependency's p50. If your SLA is 300ms and
  you call three services, you can't give each a 1s timeout. Work *backwards* from the budget.
- Use **deadlines, not per-hop timeouts**, when you can. Propagate an absolute "respond by T"
  deadline through the call chain so a request that's already doomed (caller gave up) doesn't keep
  consuming downstream resources. This is **deadline propagation** and it's a great staff-level
  detail to drop.
- Distinguish **connect timeout** (fast, ~100ms — TCP handshake) from **request/read timeout**
  (sized to the operation).

> **Say this:** "A missing timeout isn't a smaller bug than a missing retry — it's a bigger one. The
> timeout is what *bounds* the blast radius of a slow dependency."

### Retries: necessary, and dangerous
Retries fix **transient** failures (a dropped packet, a brief blip, one bad node behind a LB). They
*amplify* **systemic** failures (an overloaded dependency). The whole art is telling them apart and
retrying only the first kind.

Rules to recite:

- **Only retry idempotent / retry-safe operations.** Reads, yes. A `POST /charge` — only with an
  idempotency key (see Part F). Otherwise you double-charge.
- **Only retry retryable errors.** Retry on timeouts, connection failures, 503/429-with-retry-after.
  *Never* retry a 400/422 (your request is wrong — it'll be wrong again) or a 401/403.
- **Cap the attempts.** 2–3 total, not "until it works."
- **Exponential backoff**: wait `base × 2^attempt` between tries (e.g., 100ms, 200ms, 400ms). Gives
  a struggling dependency room to recover instead of kicking it while it's down.
- **Add jitter** — randomize the backoff. This is the part people forget and it matters enormously.

### Why jitter matters (be ready to explain this cold)
Without jitter, every client that failed at the same instant (say, a dependency hiccuped at
12:00:00) retries at the *same* backoff intervals — at 12:00:00.1, 12:00:00.3, 12:00:00.7… They've
**synchronized**. The dependency gets slammed by perfectly-aligned retry waves, never gets a quiet
moment to recover, and you've built a self-inflicted DDoS. Backoff alone spaces out *one* client's
retries; it does nothing about *many* clients colliding.

Jitter spreads those retries randomly across the window so the load smooths out:

- **Full jitter** (the one to mention): `sleep = random(0, base × 2^attempt)`. AWS's analysis found
  this minimizes both contention and completion time. It's the default recommendation.
- **Equal jitter**: `sleep = (base × 2^attempt)/2 + random(0, (base × 2^attempt)/2)`.
- **Decorrelated jitter**: `sleep = min(cap, random(base, prev_sleep × 3))`.

> **Say this:** "Backoff keeps one client from hammering a dependency; jitter keeps *all* clients
> from hammering it *in unison*. Backoff without jitter just synchronizes the herd."

### The retry storm / retry amplification
Here's the killer, and it's why budgets matter. Picture a chain: A → B → C, each configured to retry
3×. C gets slow. B retries C 3× per call. A retries B 3× per call. Now one user request becomes
**3 × 3 = 9** calls to C. Add a fourth layer and it's 27×. **Retries multiply along the call
chain.** The moment a deep dependency degrades, the *retries themselves* become the load that keeps
it down — a **retry storm**. The dependency can't recover because the instant it serves one request
it's buried by retries of the ones it dropped.

Mitigations (name several):

- **Retry budgets / retry quotas.** Cap retries as a *fraction* of total requests (e.g., "retries
  may be at most 10% of outbound traffic"). When the budget is exhausted, you stop retrying and fail
  fast. This is what makes retries safe at scale — it bounds amplification regardless of how many
  layers there are. (gRPC and Envoy implement exactly this.)
- **Retry only at one layer.** Pick the layer closest to the user (or closest to the failure) to own
  retries; everyone else fails fast. Stops the multiplicative blowup.
- **Circuit breakers** (Part C) so a struggling dependency gets *fewer* calls, not more.
- **Don't retry on the server side what the client will also retry.**

---

## Part C — Circuit breakers (in depth), bulkheads, and the difference

### Circuit breaker
A circuit breaker is a stateful proxy around a remote call that **stops sending requests to a
dependency that's clearly failing**, so you (a) fail fast instead of piling up timeouts, and (b)
give the dependency room to recover. Named after the electrical kind: it trips to protect the
circuit.

**Three states:**

| State | Behavior | Transition |
|---|---|---|
| **Closed** | Requests flow normally. Failures are counted. | Failure rate crosses threshold → **Open**. |
| **Open** | Requests **fail immediately** (no call made) — return cached/fallback/error fast. | After a cooldown timer → **Half-Open**. |
| **Half-Open** | Allow a *limited* number of trial requests through. | Trials succeed → **Closed** (reset). Any trial fails → back to **Open**. |

**Thresholds you must be able to discuss:**

- **Trip condition**: usually an *error rate* over a rolling window (e.g., ">50% of the last 20
  requests failed"), not a raw count — raw counts misbehave at low and high volume. Often a minimum
  request volume gate too ("at least 20 requests in the window") so one failure on a quiet endpoint
  doesn't trip it.
- **Open cooldown** (the "sleep window"): how long to stay open before probing — typically seconds
  to tens of seconds. Too short → you hammer a still-broken dependency; too long → slow recovery.
- **Half-open probe count**: how many trial requests, and the success ratio needed to close.

The half-open state is the clever bit: it's how the breaker **self-heals** without a human, while
controlling the thundering herd on recovery — only a trickle of probes go through, not the full
firehose the instant the timer expires.

> **Say this:** "The circuit breaker converts a *slow* failure into a *fast* one. A pile of 30-second
> timeouts is what kills you; an instant 'circuit open, here's the fallback' is what saves you. And
> half-open means it recovers on its own without re-melting the dependency."

### Bulkhead
Named after a ship's compartments: if one floods, the watertight bulkhead keeps the rest afloat. In
software, a bulkhead **isolates resources so one failing dependency can't consume all of them**.

Concretely: give each downstream dependency its **own** connection pool / thread pool / semaphore.
If dependency X is slow and saturates *its* pool of 20 threads, dependencies Y and Z still have
their pools and keep working. Without bulkheads, X's slowness consumes *every* thread in a shared
pool and your whole service is down — even the endpoints that never touch X.

Also applies at coarser grains: **cell-based architecture** (partition users into isolated "cells"
each with its own full stack), separate fleets for different tenants, shuffle sharding.

### Circuit breaker vs bulkhead — the difference (a common interview question)
- **Circuit breaker** is about *time*: stop calling a dependency that's failing *right now*, fail
  fast, recover later. It's reactive to a dependency's health.
- **Bulkhead** is about *space*: cap how many of your resources any one dependency can ever consume,
  so its failure is *contained* and can't starve the others. It's a static isolation boundary.

They compose: bulkheads limit the blast radius of a slow dependency; circuit breakers stop you from
even spending those (bulkheaded) resources on a dependency you already know is down. Use both.

---

## Part D — Backpressure, load shedding, fail-fast vs queueing

When demand exceeds capacity, you have exactly three choices: **slow the producer (backpressure),
drop work (load shedding), or buffer it (queue)**. Pretending you have infinite capacity is the
fourth, and it's how you crash.

- **Backpressure**: signal upstream to *slow down*. TCP flow control, bounded queues that block, gRPC
  flow control, reactive streams. The system communicates "I'm full" rather than silently accepting
  work it can't do. Best when the producer can actually slow down (internal services).
- **Load shedding**: when overloaded, **reject excess requests cheaply and early** (return 429/503
  *before* doing expensive work) to keep the accepted ones healthy. Shed the *least valuable* traffic
  first — prioritize (e.g., drop background/batch before user-facing; drop free tier before paid;
  shed retries before first attempts). The key insight: **a server that accepts more than it can
  serve serves *nothing* well** — goodput collapses to zero under overload. Shedding keeps goodput
  high.
- **Queueing**: buffer work for later. Smooths spikes for *async-tolerable* work (video encoding,
  emails). But a queue is **not** a fix for sustained overload — if arrival rate > service rate, the
  queue grows without bound and you just move the failure (and add latency + memory pressure).
  Always **bound** the queue and decide what happens when it's full (shed, or backpressure).

### Fail-fast vs queueing
- **Fail-fast** when the work is **latency-sensitive and synchronous** — a user is waiting and a
  late answer is worthless. Better to return an error/fallback in 50ms than a correct answer in 30s.
  Also: stale queued work is often useless work (the user already gave up / retried).
- **Queue** when the work is **async-tolerable and you value throughput/completion over latency**,
  and the spike is **temporary** (queue drains during the trough).

> **Say this:** "Under overload I'd rather shed 20% of traffic and keep 80% healthy than queue 100%
> and serve all of it past the deadline. Goodput, not throughput, is what the user feels. And a queue
> only absorbs *bursts* — for sustained overload it just relocates the failure and adds latency."

The **bounded queue** is the unifying idea: a full bounded queue *is* your backpressure/shedding
signal. An unbounded queue is a latent OOM and an infinite latency machine.

---

## Part E — Idempotency for safe retries

Retries, at-least-once queues, and client reconnects all mean **the same operation can arrive more
than once**. If processing it twice corrupts state (double charge, double order, duplicate post),
your resilience machinery is *creating* bugs. Idempotency is the contract that makes
"deliver-at-least-once + retry" safe. (This ties directly to the message-queue and exactly-once
discussions in the queues/consistency docs.)

- **Idempotency key**: client generates a unique key per logical operation; server records "I've
  seen this key + the result." On a retry with the same key, return the stored result instead of
  re-executing. This is how Stripe makes `POST /charge` safe to retry.
- **Natural idempotency**: `PUT x=5` is inherently idempotent; `balance += 5` is not. Prefer
  set-to-a-value over increment-by where you can.
- **Dedup window**: the server stores keys for some TTL; sized to your max retry horizon.
- **Idempotent consumers**: in a queue, dedupe on a message ID, or make the downstream write a
  conditional upsert keyed by something stable.

> **Say this:** "I can only safely retry an operation if it's idempotent — so for any mutating call
> in the retry path I'll require an idempotency key and dedupe server-side. Retries and at-least-once
> delivery are the *reason* idempotency isn't optional at scale."

---

## Part F — Redundancy, multi-AZ/region, and DR (RTO/RPO)

No redundancy = the component is a single point of failure. The questions are *what kind* of
redundancy and *how much*.

### Active-active vs active-passive
- **Active-active**: all replicas serve traffic simultaneously. Pros: no idle capacity, instant
  failover (just stop routing to the dead one), load is shared. Cons: harder — needs conflict
  resolution / careful state handling if they all write, and you must run with enough headroom that
  the survivors can absorb the dead one's share.
- **Active-passive (standby)**: one serves, a standby waits to take over. Pros: simpler, no write
  conflicts. Cons: idle (paid-for) capacity, and **failover takes time** (detect + promote +
  redirect) — that gap is downtime.

### N+1 (and N+M) redundancy
Provision enough that you can lose **one** (N+1) — or **M** (N+M) — instances and still serve full
load. If N instances are needed at peak, run N+1. The classic mistake: running exactly at capacity
across 3 AZs so that losing one AZ means losing 33% of capacity and falling over. Provision so the
*survivors* carry the load.

### Multi-AZ vs multi-region
- **Multi-AZ** (availability zones in one region): cheap, low-latency (sub-ms between AZs), the
  *default* for any serious system. Survives a datacenter/AZ failure (power, network, flood).
  Synchronous replication across AZs is viable.
- **Multi-region**: expensive and hard (cross-region latency = tens to >100ms → usually async
  replication → you accept some RPO, and the CAP tradeoff bites). Survives a whole-region outage and
  serves users globally with low latency. Do it when you need region-failure survival or global low
  latency — not by reflex.

### RTO and RPO — define them precisely
- **RTO (Recovery Time Objective)**: how long you can be **down** — the max acceptable time to
  restore service. "We must be back within 1 hour." Drives *how fast you fail over*.
- **RPO (Recovery Point Objective)**: how much **data** you can afford to lose — the max acceptable
  age of the data you recover to. "We can lose at most 5 minutes of writes." Drives *how often /
  how synchronously you replicate/back up*.

Memory hook: **RTO = time, RPO = data.** RPO near zero ⇒ synchronous replication. RTO near zero ⇒
hot standby already running.

### DR strategies vs RTO / RPO / cost
| Strategy | What's running | RTO | RPO | Cost | When |
|---|---|---|---|---|---|
| **Backup & restore** | Nothing; restore from backups/snapshots | Hours–days | Hours (since last backup) | $ | Non-critical, cost-sensitive, big tolerance |
| **Pilot light** | Core data replicated; servers off/minimal | 10s of min–hours | Minutes | $$ | Important but can tolerate a spin-up |
| **Warm standby** | Scaled-down full stack running, replicating | Minutes | Seconds–minutes | $$$ | Low tolerance, manageable cost |
| **Hot standby / active-active** | Full-capacity stack live in 2nd region | Seconds (near-zero) | Near-zero | $$$$ | Mission-critical, near-zero RTO/RPO |

The line of the table: **lower RTO/RPO costs exponentially more.** You buy exactly as much DR as the
business value of uptime justifies — say that explicitly. "Payments get warm standby; the marketing
blog gets backup-and-restore."

---

## Part G — Cascading failures and how to prevent them

A cascading failure is when a *local* failure triggers a chain reaction that takes down far more than
the original fault. The exam favorite. The common mechanisms:

1. **Resource exhaustion**: a slow dependency makes calls pile up → threads/connections/memory
   exhaust → the service can't serve *anything*, including healthy endpoints → its callers now see
   *it* as failed → propagates upward.
2. **Retry storms** (Part B): a blip triggers retries that multiply along the chain and become the
   load that prevents recovery.
3. **Load redistribution**: one node dies → its traffic shifts to the survivors → they're now over
   capacity → one falls over → its load shifts to the rest → **death spiral**. (This is why N+1 and
   headroom matter — survivors must be able to take the load.)
4. **Thundering herd on recovery**: the dependency comes back, *every* waiting client hits it at
   once, and it immediately falls over again.

**Prevention toolkit (recite this list):**
- **Timeouts + bounded resources** so a slow dependency can't exhaust you.
- **Circuit breakers** so you stop feeding a dying dependency.
- **Bulkheads** so one dependency's failure can't starve the others.
- **Load shedding + backpressure** so overload is rejected, not absorbed.
- **Retry budgets + backoff with jitter** so retries don't become the attack.
- **Headroom / N+1** so load redistribution doesn't tip survivors over.
- **Autoscaling** (but note: it's *too slow* to save you from a sudden spike — protective shedding is
  your fast defense; autoscaling is the slow follow-up).

> **Say this:** "Cascading failures almost always come down to *unbounded* something — unbounded
> timeouts, unbounded retries, unbounded queues, unbounded load shift. My job is to put a bound on
> each one so a local fault stays local."

---

## Part H — Thundering herd on restart / cold cache

Two related "recovery is the dangerous part" problems:

- **Cold cache after restart**: a cache (or a freshly-restarted service whose cache is empty)
  sends *all* misses straight to the database. The DB, sized for a 95%-cache-hit steady state, gets
  20× its expected load and falls over — right when you're trying to recover. Same thing on a cache
  flush or a failover to a cold replica.
- **Thundering herd on a hot key**: a popular key expires; thousands of concurrent requests all miss
  and all hit the DB to recompute the same value simultaneously.

**Mitigations:**
- **Request coalescing / single-flight**: only one request recomputes a missing key; the rest wait
  for and share its result. (Kills the hot-key herd.)
- **Cache warming / pre-warming**: populate the cache before taking traffic; on deploy, route traffic
  in gradually so caches fill incrementally.
- **Staggered / jittered TTLs**: don't expire everything at once — add randomness to TTLs so
  expirations spread out.
- **Stale-while-revalidate**: serve the stale value while one worker refreshes in the background.
- **Gradual ramp on restart / slow-start LB**: bring a recovered node back at a *fraction* of full
  traffic and ramp up, so its cold cache fills before it's at full load. (Same idea as the
  circuit-breaker half-open trickle.)
- **Lock/lease for recomputation** so only one recompute happens per key.

> **Say this:** "Recovery is often more dangerous than the failure. An empty cache + full traffic is
> how a recovery turns into a second outage — so I ramp traffic, coalesce misses, and jitter TTLs."

---

## Part I — Health checks, graceful shutdown, connection draining

### Liveness vs readiness (know the difference)
- **Liveness**: "Is this process alive, or wedged?" Fails → the orchestrator **restarts** it. Keep it
  cheap and *local* — don't check downstream dependencies in liveness, or a DB blip will trigger a
  fleet-wide restart loop (you'll kill healthy processes for a dependency's fault).
- **Readiness**: "Is this instance ready to *take traffic right now*?" Fails → the LB/orchestrator
  **stops routing** to it (but doesn't kill it). This *can* check dependencies and can fail
  temporarily (warming up, overloaded, lost DB connection) and recover.
- (Also **startup probes**: give a slow-booting app time before liveness kicks in.)

The classic bug: putting dependency checks in liveness. A shared dependency hiccups → every instance
fails liveness → orchestrator restarts the entire fleet → guaranteed full outage. Dependency checks
belong in *readiness*.

### Graceful shutdown & connection draining
When you deploy, scale-in, or terminate an instance, *don't* just kill it — you'll drop in-flight
requests. The sequence:

1. **Fail readiness** (or deregister from the LB) so no *new* requests are routed in.
2. **Drain**: let in-flight requests finish (with a deadline). The LB stops sending new connections;
   existing ones complete. This is **connection draining** / connection draining timeout.
3. **Stop accepting**, finish work, flush buffers/commit offsets, close connections cleanly.
4. **Exit.**

In Kubernetes terms: catch `SIGTERM`, fail readiness, sleep past the deregistration propagation
delay, finish in-flight, then exit before `terminationGracePeriod` triggers `SIGKILL`. Without this,
every deploy is a small burst of 5xx — which silently eats your error budget.

> **Say this:** "Liveness restarts a wedged process; readiness gates traffic. I keep dependency
> checks out of liveness so a dependency blip doesn't restart the whole fleet, and I drain
> connections on shutdown so deploys don't throw 5xx."

---

## Part J — Observability: pillars, RED/USE, SLI/SLO/SLA, tracing

You can't operate what you can't see. **Observability is part of the design**, not an afterthought —
mention it unprompted in the wrap-up.

### The three pillars
- **Metrics**: numeric time series, cheap, aggregatable. Great for dashboards, alerting, trends.
  ("Error rate is 3% and climbing.") Low cardinality.
- **Logs**: discrete events, high detail. Great for *what exactly happened* in one request. Expensive
  at volume; use structured (JSON) logs.
- **Traces**: the path of *one* request across services, with timing per hop. Great for "where did
  the 800ms go in this distributed call?"

Use them together: metric alerts you *that* something's wrong → trace localizes *where* → logs tell
you *what*.

### RED vs USE (two methods, know when each applies)
- **RED** (for **services / request-driven things**): **R**ate (requests/sec), **E**rrors
  (failed/sec), **D**uration (latency distribution — p50/p95/p99). "How is this service doing from
  the *caller's* view?"
- **USE** (for **resources** — CPU, disk, pool, queue): **U**tilization, **S**aturation (queue depth
  / how-overloaded), **E**rrors. "Is this *resource* the bottleneck?"

RED for the service, USE for the machine/resource. They're complementary.

### SLI / SLO / SLA and error budgets
- **SLI** (Indicator): the *measurement*. "Fraction of requests served < 300ms" or "fraction of
  requests that are non-5xx."
- **SLO** (Objective): your *internal target* for the SLI. "99.9% of requests succeed over 30 days."
- **SLA** (Agreement): the *external, contractual* promise with **consequences** (refunds/credits).
  Always looser than your SLO — you alert and act well before you breach the contract.
- **Error budget**: `100% − SLO`. At 99.9%, you may be "bad" for ~43 min/month. This budget is a
  *currency*: spend it on risk. Budget remaining → ship features, run chaos experiments. Budget
  exhausted → freeze risky deploys, focus on reliability. It turns the "move fast vs stay up"
  argument into a *number* both sides agree on, which is the real point.

> **Say this:** "I'd define an SLO — say 99.9% of feed reads under 300ms — and run to an error budget.
> The budget tells us objectively whether we have room to take deploy risk or need to freeze and
> stabilize. The SLA is the looser external promise; we act on the SLO."

### Distributed tracing
- A **trace** = one request's whole journey; a **span** = one operation/hop within it (parent/child
  spans form a tree with timings).
- **Context propagation** is the mechanism: a **trace ID** (+ parent span ID) is passed in headers
  (W3C `traceparent`) across every service hop so the spans can be stitched into one trace. If a hop
  doesn't propagate context, the trace breaks there.
- **Sampling**: you don't trace 100% (too expensive) — sample (head-based: decide at ingress;
  tail-based: keep the interesting/slow/errored ones). Mention OpenTelemetry as the standard.

### Cardinality pitfalls (a strong staff detail)
**Cardinality** = number of distinct label/tag combinations on a metric. It's the #1 way to blow up
a metrics bill and OOM your TSDB. Each unique combination of label values is a *separate* time
series. Put `user_id`, `request_id`, `email`, or `URL-with-IDs` in a metric label and you create
millions of series — death.
- **Rule**: metric labels must be **low-cardinality** (status code, region, endpoint *template* like
  `/users/{id}`, instance) — never unbounded user/request identifiers.
- High-cardinality data (user IDs, request IDs) belongs in **logs and traces**, which are indexed for
  it, not in metrics.

---

## Part K — Deployment safety (this is an availability feature)

Most outages aren't random hardware — they're **changes** (bad deploys, bad configs). So deployment
strategy *is* resilience. Treat config rollouts with the same rigor as code.

- **Blue/green**: run two identical environments. Deploy to green (idle), test it, then flip the
  router from blue to green. Instant cutover, **instant rollback** (flip back). Cost: double the
  infra during the switch; DB schema changes need care (must be compatible with both).
- **Canary**: route a *small* % of traffic to the new version, watch metrics (error rate, latency),
  ramp up gradually if healthy, **auto-rollback** if not. Catches bad deploys with tiny blast radius.
  The default for high-scale services. Pair with **automated rollback** on SLO regression.
- **Rolling deploy**: replace instances a few at a time. Cheap, no double infra, but slower rollback
  and a bad version briefly coexists.
- **Feature flags**: decouple *deploy* from *release*. Ship code dark, turn it on for 1% → 10% →
  100%, and **kill-switch it off instantly** without a redeploy. Also enables progressive rollout and
  A/B. The fastest possible "rollback" is a flag flip.
- **Rollback plan**: every deploy needs one, and it must be *fast*. "How do we undo this in under a
  minute?" Backward-compatible schema migrations (expand/contract pattern) are what *make* rollback
  possible — never ship a migration you can't roll back.

> **Say this:** "Most incidents are self-inflicted by changes, so I'd deploy via canary with
> automated rollback on SLO regression, and gate risky features behind flags so release is decoupled
> from deploy and I can kill a bad feature in seconds without a redeploy."

---

## Part L — Chaos engineering and game days

You don't actually know the system survives failure until you *cause* failure. **Chaos engineering**
= deliberately injecting failures (kill instances, add latency, drop a dependency, partition the
network, fail an AZ) — ideally in production, with a hypothesis and a blast-radius limit — to verify
your resilience mechanisms work *before* reality tests them at 3am.

- Start in staging, then production with a small blast radius and an abort switch.
- Netflix's Chaos Monkey (kills instances) is the canonical example; the discipline scaled to
  region-level "Chaos Kong."
- **Game days**: scheduled exercises where the team rehearses an incident (simulate a region loss,
  a dependency outage) to test both the *system* and the *humans/runbooks/on-call*. The failure you
  rehearse is the one you survive calmly.

> **Say this:** "I'd validate all of this with chaos experiments and game days — injecting the
> failure on purpose, with a bounded blast radius, so we learn the system survives an AZ loss in a
> controlled test instead of discovering it during a real one."

---

## Part M — Single-point-of-failure hunting checklist (great for wrap-ups)

Walk the architecture and for **every** box and arrow ask "what if this dies?" The places SPOFs hide:

- [ ] **Load balancer** — is the LB itself redundant? (Multiple LBs, DNS failover / anycast.)
- [ ] **Single DB primary** — what's the failover story? Replica promotion — automatic, and how fast?
- [ ] **Single region / single AZ** — survives an AZ loss? A region loss (if required)?
- [ ] **The cache** — does a cache failure dump full load on the DB? (Cold-cache / Part H.)
- [ ] **Message queue / broker** — replicated? What if it fills or dies?
- [ ] **Shared config / service discovery / secrets** (ZooKeeper, etcd, Consul) — often a hidden SPOF
  whose outage takes down everything that depends on it.
- [ ] **DNS** — a real and historically common SPOF.
- [ ] **A single shared dependency** all services call (auth service, a feature-flag service) — its
  failure domain spans your whole system.
- [ ] **A leader / coordinator** — is leader election automatic? Split-brain protection?
- [ ] **Deploy pipeline / a bad config** — can one push take everything down? (Canary gates it.)
- [ ] **Human/process** — only one person who knows how to recover X? Runbook exists?
- [ ] **Capacity headroom** — can survivors absorb a dead node's load (N+1), or do they tip over?

For each SPOF found: *redundancy* (more than one), *failover* (detect + switch automatically), and
*isolation* (shrink its failure domain).

---

## Part N — How to talk about failure modes in an interview

A repeatable script when the interviewer says "what if X fails?":

1. **Name the failure domain and blast radius.** "If this DB primary dies, every write in this shard
   stops — reads can continue from replicas."
2. **Detection.** "Health checks / a missed heartbeat detect it in ~Ns."
3. **Immediate response.** "Circuit breaker opens, we fail fast and serve a degraded read-only mode
   from the cache + replicas."
4. **Recovery.** "A replica is promoted (automated failover, ~30s RTO); we accept up to a few seconds
   RPO from async replication."
5. **Preventing the cascade.** "Timeouts + retry budgets stop callers from storming the new primary;
   we ramp traffic to avoid a cold-start herd."
6. **What it costs / what you'd improve.** "Synchronous replication would cut RPO to zero but add
   write latency — I'd only do that for the balance table, not the activity log."

Proactively volunteer failure modes *before* you're asked — it's the strongest seniority signal in
the room. End the design with: "Let me call out the failure modes and SPOFs and how I'd handle each."

> **The framing that lands:** "Every distributed system is a negotiation between consistency,
> availability, and cost *under failure*. I've told you where this one chooses to degrade, where it
> chooses to fail fast, and where it spends money to stay up — those choices come straight from the
> non-functional requirements we set at the start."

---

### Self-check before the mock (answer these from memory)
- [ ] Define failure domain, blast radius, and graceful degradation in one sentence each.
- [ ] Why does jitter matter on top of exponential backoff? What's "full jitter"?
- [ ] Explain a retry storm and how a retry *budget* bounds it.
- [ ] Walk the three circuit-breaker states and what each threshold controls.
- [ ] Circuit breaker vs bulkhead — the one-line difference (time vs space).
- [ ] When do you fail-fast vs queue? Why is goodput the right metric under overload?
- [ ] RTO vs RPO — which is time, which is data? Map the four DR strategies onto them.
- [ ] Name three mechanisms that cause a cascading failure and a bound for each.
- [ ] Liveness vs readiness — why must dependency checks stay out of liveness?
- [ ] RED vs USE — which is for services, which for resources?
- [ ] SLI vs SLO vs SLA, and what an error budget *buys* you.
- [ ] Why is high metric cardinality dangerous, and where does user_id belong instead?
- [ ] Canary vs blue/green vs feature flag — and why deployment is an availability feature.
- [ ] Run the SPOF-hunting checklist on the last system you designed.
