# Topic 27: Observability & SRE — Operating Large Systems

> **Why this topic earns the offer:** Doc 13 gave you the failure-mode vocabulary — timeouts, circuit
> breakers, degraded modes, DR. This doc is the other half of the same coin: **how you *know***. Any
> candidate can draw the boxes; the staff signal is the reflex to ask, of every box and arrow, *"how
> would I know this is failing, and how fast?"* Interviewers love to push here precisely because it
> can't be memorized as a pattern — it reveals whether you've actually carried a pager. This doc goes
> deep on the three pillars, SLOs and error budgets, alerting that doesn't burn people out, incident
> response, and how to weave all of it into a design answer. **Resilience mechanics (retries, breakers,
> chaos, deploy strategies, RTO/RPO) live in doc 13 — I cross-reference rather than repeat them.**

---

## Part A — The mindset: observability ≠ monitoring

Say the distinction out loud, because it's the framing the whole topic hangs on:

- **Monitoring** answers *known* questions: "is CPU above 80%? is error rate above 1%?" You decided
  in advance what to watch. Great for failure modes you've already seen.
- **Observability** is the property that lets you answer questions you *didn't* predefine, from the
  data you already emit — "why are *Canadian* users on the *new* app version seeing p99 latency spike,
  but only for *checkout*?" You never built a dashboard for that exact slice, but the data has enough
  dimensionality to let you ask it after the fact.

At scale, almost every serious incident is *novel* — a combination of conditions no one foresaw. So
you can't monitor your way out; you need the ability to *interrogate* a running system.

> **The line to say:** "I don't just want dashboards for the failures I expect. I want enough signal
> in the system that when something I *didn't* expect breaks at 3am, I can ask a new question and get
> an answer in minutes — that's the difference between monitoring and observability."

The three properties that buy you that: high-dimensional data (rich labels/attributes), the ability
to correlate across pillars (metric → trace → log for one request), and not having pre-aggregated
away the detail you'll wish you had.

---

## Part B — The three pillars, done right

| Pillar | What it is | Best at | Cost driver | Cardinality tolerance |
|---|---|---|---|---|
| **Metrics** | Numeric time series, pre-aggregated | "Is something wrong? trend over time? alert" | # of time series (cardinality) | **Low** — must bound labels |
| **Logs** | Discrete timestamped events, high detail | "What exactly happened in this one case?" | Volume × retention (bytes) | High — index what you query |
| **Traces** | One request's path across services, timed | "*Where* did the latency/error happen?" | Spans × sample rate | High — IDs belong here |

The workflow that ties them together (memorize this sentence): **a metric alert tells you *that*
something is wrong → a trace localizes *where* → logs tell you *what* and *why*.** When asked "how do
you debug this," walking that path is the answer.

### Metrics: the math matters

Three metric *types*, and you should know what each is for:

- **Counter** — monotonically increasing (requests_total, errors_total, bytes_sent). You never read
  the raw value; you read its **rate** (`rate(requests_total[5m])`). Counters survive restarts because
  rate functions handle the reset.
- **Gauge** — a value that goes up and down (queue_depth, memory_in_use, active_connections,
  temperature). Snapshot of "right now."
- **Histogram** — bucketed distribution of observations (request_duration). This is the one that
  separates people who've operated systems from people who haven't.

**Why histograms/percentiles, not averages — the p99 vs mean lesson.** Averages lie about latency,
and they lie in the direction that hurts you. Two reasons, say both:

1. **The average hides the tail.** If 99% of requests take 10ms and 1% take 5s, your mean is ~60ms —
   looks fine, and you'd never page on it. But 1 in 100 users is having a 5-second experience, and a
   page that loads 30 resources will hit that tail almost every time (see Part H, the tail-latency
   problem). **The mean describes a user who doesn't exist.** The percentile describes a real one.
2. **You can't average percentiles.** This is the subtle, load-bearing point. If host A reports a p99
   of 100ms and host B reports a p99 of 200ms, the fleet p99 is **not** 150ms — averaging percentiles
   is mathematically meaningless. To get a correct aggregate percentile you must aggregate the
   *histogram buckets* across hosts and compute the percentile from the merged distribution. This is
   exactly why metrics systems store latency as histograms (bucket counters), not as a precomputed
   p99 per host — so they can be merged correctly at query time.

> **Say this:** "I'd track latency as a histogram and alert on p99/p99.9, never the mean. The mean
> describes an average of a user who doesn't exist; the tail is where real users live. And I'd store
> buckets, not a pre-computed percentile per host, because you can't average percentiles across hosts."

Pick percentiles deliberately: **p50** = the typical experience, **p99** = your unlucky-but-common
user, **p99.9** = where systemic tail problems show up (and where a high-fan-out request lands). For a
page making N backend calls, the *page's* latency tracks roughly the p(1 − (1/N)) of a single call —
fan-out turns your p99 into the median experience. (Tail-amplification math is in Part H.)

**Cardinality explosions — how they kill a metrics system.** (Doc 13 §J flagged this; here's the
depth.) Cardinality = the number of *distinct time series* a metric produces, which is the product of
the distinct values of every label. A metric `http_requests{status, method, endpoint}` with 5
statuses × 4 methods × 20 endpoints = 400 series. Fine. Now someone adds `user_id` as a label. With
10M users you now have **4 billion** series. Each series is a separate object the TSDB must keep in
RAM (active series live in memory in Prometheus-class systems), index, and persist. The failure mode
is brutal and specific:

- Memory blows up → the TSDB OOMs and restarts → you lose monitoring *during* the incident the bad
  metric probably caused.
- Ingestion and query latency degrade for *every other metric*, so one team's bad label poisons the
  whole platform.
- The bill explodes (most hosted vendors price on active series / datapoints).

The rule: **metric labels must be low, bounded cardinality.** Good labels: status code, method, region,
instance, endpoint *template* (`/users/{id}`, never `/users/12345`). Forbidden in labels: `user_id`,
`request_id`, `email`, `session_id`, full URLs with IDs, error *messages* (use error *type*), raw
timestamps. High-cardinality identifiers belong in **logs and traces**, which are built to index them.
Practical defenses: a **cardinality budget** per service, linters/relabeling rules that drop or
template offending labels at ingestion, and alerts on series-count growth.

> **The trap to call out:** "The fastest way to take down your monitoring is a high-cardinality label
> like user_id on a metric. It's not a cost problem first — it's an availability problem: the TSDB
> OOMs right when you need it. user_id goes in traces and logs, never in a metric label."

### Logs: detail you pay for by the byte

- **Structured logging.** Emit JSON (or another machine-parseable format), not free-text
  `printf`. `{"ts":..., "level":"error", "trace_id":"abc", "user_id":123, "latency_ms":812, "msg":...}`.
  Structured logs are queryable ("all errors where latency_ms > 500 and region=eu") and joinable to
  traces by `trace_id`. Free-text logs force regex archaeology at 3am. This is non-negotiable at scale.
- **Always include the correlation/trace ID** (Part H) so a log line can be tied to its request and
  trace. A log without a correlation ID is an orphan.
- **Log levels and discipline.** ERROR = something needs human attention (and should be rare enough
  that it *can*). WARN = recoverable/degraded. INFO = significant business events. DEBUG = off in
  prod by default, flippable per-request or per-service. The anti-pattern: logging at INFO inside a
  hot loop — you've built a log firehose that costs a fortune and drowns the signal. **If everything
  is logged, nothing is found.**
- **Cost and sampling.** Logs are the most expensive pillar at volume (ingest + index + retention,
  often $/GB). Controls: **sample** high-volume low-value logs (keep 1% of successful-request access
  logs but **100% of errors** — never sample away the thing you'll need); **tiered retention** (hot
  searchable for 7–30 days, then cheap cold object storage); **drop/redact** noisy or sensitive fields
  at the collector. A good default: sample the boring, keep all the interesting.

### Traces: where the latency went

(Doc 13 §J defined trace/span/context propagation; this is the operating depth.)

- A **trace** is one request's whole journey; a **span** is one operation within it (an RPC, a DB
  query, a cache lookup). Spans nest into a tree with parent/child links and per-span timing, so a
  trace is a flame graph of where a request spent its time across services.
- **Context propagation** is the mechanism. At ingress you mint a trace ID; every outbound call
  carries `traceparent` (W3C Trace Context: trace-id, parent span-id, sampling flag) in its headers,
  and each service creates child spans under it. **A hop that doesn't propagate context breaks the
  trace there** — this is the single most common reason traces are useless in practice, and it's
  usually a thread pool, a queue, or an async boundary that dropped the context. Propagating through
  **async/queue boundaries** (stash the context in the message) is the hard part; call it out.

**Sampling strategies — head vs tail (know the tradeoff cold):**

| | **Head-based sampling** | **Tail-based sampling** |
|---|---|---|
| Decision made | At ingress, *before* the request runs | After the trace completes, seeing the whole thing |
| Keeps | A random % (e.g., 1%) of everything | The *interesting* ones — errors, slow (>p99), specific routes |
| Cost / complexity | Cheap, stateless, trivial | Must buffer all spans of a trace until complete → memory + a stateful collector layer |
| Blind spot | Throws away most errors and slow traces (they're rare!) | Operationally heavier; buffering window bounds trace length |

The punchline: **head sampling is cheap but discards exactly the traces you want** (errors and the
slow tail are by definition rare, so 1% random keeps almost none of them). **Tail sampling keeps the
interesting traces but is expensive and stateful** because you must hold every span of a trace until
it finishes to judge it. Real systems often do both: a low head-sample baseline for "normal" traffic
shape plus tail sampling to guarantee you capture every error and slow trace. **OpenTelemetry** is the
standard plumbing for this (see Part C).

### When to reach for which pillar

| Question you're asking | Pillar |
|---|---|
| "*Is* something wrong? Should I be paged? What's the trend?" | **Metrics** (cheap, aggregate, alertable) |
| "*Where* in this multi-service request did the time/error go?" | **Traces** |
| "*What exactly* happened in this one request / what was the payload / stack trace?" | **Logs** |
| "What's the p99 across the fleet?" | **Metrics** (histograms) |
| "Why is *this specific* high-cardinality slice (one user, one tenant) broken?" | **Logs / Traces** (metrics can't hold that cardinality) |

---

## Part C — Metrics systems: pull vs push, storage, and OpenTelemetry

### Pull (Prometheus) vs push

| | **Pull** (Prometheus scrapes `/metrics`) | **Push** (app sends to a gateway/agent) |
|---|---|---|
| Who initiates | Monitoring system scrapes targets on an interval | Application/agent pushes datapoints |
| Service discovery | Central; the scraper has the full target list → **target health is itself a signal** (a target that can't be scraped is "down") | App must know where to send |
| Short-lived jobs | Awkward (job exits before scrape) → needs a pushgateway | Natural fit (batch jobs, lambdas, client-side) |
| Firewall/network | Scraper needs network *to* targets | Targets need network *out*; friendlier across NAT/edge |
| Back-pressure / overload | Scraper controls rate; a slow target just gets scraped less | A spike in senders can overwhelm the ingest endpoint |

Say the nuance: **pull gives you free liveness** (failure to scrape = an alertable signal, "up == 0")
and centralized control of rate; **push is better for ephemeral/edge/client-side** sources that the
scraper can't reach or that don't live long enough to be scraped. Many setups are hybrid: pull inside
the cluster, push from batch jobs and the edge via a gateway. Don't be dogmatic — pick per source.

### Time-series storage and aggregation

- A TSDB stores `(metric, label-set) → [(timestamp, value)...]`. The (metric + label-set) identity is
  the **series**; series count is what you budget (Part B).
- Data is **aggregated/downsampled over time**: raw high-resolution recent data (seconds), rolled up
  to coarser resolution as it ages (minutes → hours) with shorter retention for the fine grain. You
  rarely need second-resolution data from six months ago.
- **Recording rules / pre-aggregation**: precompute expensive queries (e.g., fleet-wide p99) on a
  schedule so dashboards and alerts are fast and cheap, rather than re-scanning raw series each query.
- **Aggregation must respect the math** (Part B): sum counters, but recompute percentiles from merged
  histogram buckets — never average percentiles.

### OpenTelemetry — the converging standard

OTel is the vendor-neutral standard for **generating and shipping** all three pillars (metrics, logs,
traces) with a single set of SDKs, a common wire format (OTLP), and the **Collector** (a pipeline that
receives, processes — sampling, redaction, relabeling — and exports to any backend). Why it matters
in a design answer:

- **Instrument once, switch backends freely.** You're not locked to one vendor's agent; you can move
  from a hosted vendor to self-hosted Prometheus/Tempo/Loki without re-instrumenting.
- **Unified context propagation** across pillars — the same trace context links your metrics, logs,
  and traces, which is what makes the metric → trace → log workflow actually work.
- The Collector is the natural place to enforce **tail sampling, cardinality limits, and PII
  redaction** centrally instead of in every app.

> **Say this:** "I'd instrument with OpenTelemetry so we emit metrics, traces, and logs in one
> standard and aren't locked to a backend, and run a Collector to do tail sampling and cardinality
> control centrally. Metrics into a Prometheus-style TSDB on a pull model for free liveness, push from
> the edge and batch jobs."

---

## Part D — RED and USE: two methods, know which is which

(Doc 13 §J introduced these; here's the operating depth and where each fails.)

- **RED — for services / anything request-driven.** From the *caller's* perspective:
  - **R**ate — requests per second.
  - **E**rrors — failed requests per second (and as a %). Be explicit about what "error" means (5xx?
    5xx + timeouts? business-level failures?).
  - **D**uration — the latency *distribution* (p50/p95/p99/p99.9 from a histogram), never the mean.
  - This is your default service dashboard, and it maps cleanly onto SLIs (availability ← Errors,
    latency ← Duration).
- **USE — for resources** (CPU, memory, disk, network, connection pool, thread pool, queue):
  - **U**tilization — % of time the resource was busy / fraction of capacity used.
  - **S**aturation — the degree to which work is *queued* waiting for the resource (run-queue length,
    pool wait time, queue depth). **Saturation is the leading indicator** — a resource at 100%
    utilization with no queue is fine; one with a growing queue is in trouble. This is the signal that
    predicts the cliff.
  - **E**rrors — error events for that resource (failed mallocs, disk errors, pool-exhaustion errors).

| | RED | USE |
|---|---|---|
| Subject | A service / endpoint | A resource (CPU, pool, disk, queue) |
| Viewpoint | The caller's experience | The machine's internal pressure |
| Answers | "How is this service doing for users?" | "*Which resource* is the bottleneck?" |
| Best for | SLOs, alerting on symptoms | Capacity, finding the saturated component |

Use them together: **RED tells you the service is slow (the symptom); USE tells you it's the DB
connection pool that's saturated (the cause).** That pairing is also the symptom-vs-cause distinction
that drives alerting (Part F). Critically, **utilization without saturation is not an emergency** — a
box at 90% CPU serving everything under SLO is *efficient*, not *broken*. Alert on saturation and
SLO impact, not on a utilization number in isolation (Part F).

---

## Part E — SLI / SLO / SLA and error budgets (in depth)

Doc 13 §J gave the one-liners. This is the part interviewers probe hardest, so go deep.

### The definitions, precisely

- **SLI (Indicator)** — the *measurement*, expressed as a ratio of good events to valid events:
  `good_events / valid_events`. E.g., "fraction of HTTP requests that returned non-5xx in < 300ms."
  An SLI is a number between 0 and 100%.
- **SLO (Objective)** — your *internal target* for the SLI over a window: "99.9% of requests are good
  over a rolling 28 days." Chosen by you, used to drive engineering decisions.
- **SLA (Agreement)** — the *external, contractual* promise to customers, with **financial/legal
  consequences** (credits, refunds) on breach. **Always set looser than your SLO** — you want your
  internal alarm (SLO) to fire with margin to spare before you breach the contract (SLA). Rule of
  thumb: if you promise customers 99.9%, run an internal SLO of 99.95%.

### Defining *good* SLIs

The single most common mistake is measuring what's easy (CPU, internal queue depth) instead of what
the user feels. Good SLIs:

- **Are user-centric / symptom-based.** Measure the experience: request success rate, request
  latency, freshness of data, correctness, availability of a critical journey. Not CPU. CPU is a
  cause; the SLI should be a *symptom the user would complain about*.
- **Are a ratio of good to valid events**, and you must define "valid" carefully — exclude health
  checks and bot traffic; decide whether 4xx (client's fault) counts against you (usually not, except
  429s, which often signal *your* overload).
- **Are measured where the user is**, as close to them as feasible (at the load balancer / edge, or
  client-side RUM), not deep in a backend that can look healthy while the edge is timing out.
- **Have a clear threshold.** "Latency < 300ms" requires picking 300ms and the percentile it applies
  to. A latency SLI is itself a good/bad split: requests faster than the threshold are "good."

The common SLI types: **availability** (success ratio), **latency** (fraction under a threshold),
**freshness/staleness** (for pipelines/replicas), **correctness**, **throughput/coverage**.

### Choosing SLO targets

- **Work backward from what users actually need**, not toward 100%. **100% is the wrong target** —
  it's infinitely expensive, and your dependencies (and the network, and the user's device) aren't
  100% anyway, so users can't perceive the difference between 99.99% and 100%. Chasing the last nines
  is spending exponential money on reliability the user never notices.
- Each "nine" is ~10× the cost. Pick the *lowest* number that keeps users happy and frees budget for
  features. Most internal services live at 99.9%; a critical payment path might be 99.99%; a batch
  analytics job might be 99%.
- Anchor to **allowed downtime** so the choice is concrete:

| SLO | Error budget (of 30 days) | "Bad" time allowed / month |
|---|---|---|
| 99% | 1% | ~7.2 hours |
| 99.9% ("three nines") | 0.1% | ~43 minutes |
| 99.95% | 0.05% | ~22 minutes |
| 99.99% ("four nines") | 0.01% | ~4.3 minutes |
| 99.999% ("five nines") | 0.001% | ~26 seconds |

### Error budgets and the error-budget *policy*

**Error budget = 100% − SLO.** At 99.9% you may be "bad" for ~43 min/month. The reframe that makes
this powerful: the budget is a **currency you spend on risk**. Every deploy, migration, chaos
experiment, and feature launch *spends* error budget; perfect uptime *banks* it.

The **error-budget policy** is the pre-agreed, written rule for what happens as the budget burns —
and the whole point is that it's agreed *before* the heat of an incident, so it's a policy decision,
not an argument:

| Budget state | Policy |
|---|---|
| Healthy (plenty left) | Ship features freely, take deploy risk, run chaos experiments — you've *earned* it |
| Burning fast / mostly spent | Slow down: more review, smaller canaries, more soak time |
| **Exhausted** | **Feature freeze.** All engineering effort redirects to reliability until the budget recovers. Often: no risky deploys until back in budget |

This is the cultural heart of SRE: it turns the eternal "dev wants to ship vs ops wants stability"
fight into **a number both sides already agreed to honor.** Reliability stops being a vibe and becomes
a budget. It also self-corrects: a team that's too conservative (never spends budget) is told to take
*more* risk and ship faster; a team that's reckless gets frozen.

> **Say this:** "I'd define a user-centric SLO — say 99.9% of checkout requests succeed in under
> 300ms over 28 days — and run to an error budget with a written policy: budget healthy, we ship; budget
> burned, we freeze features and fix reliability. That turns 'fast vs stable' from an argument into a
> number. The SLA we sign with customers is looser than the SLO, so we react well before we breach a
> contract."

### Burn-rate alerting — multi-window, multi-burn-rate (why it beats threshold alerts)

This is the staff-level payoff of having an SLO, so be ready to explain it.

**Burn rate** = how fast you're consuming the error budget relative to "even" consumption. Burn rate
1× means you'll exactly exhaust the budget over the SLO window; **14.4×** means you'll burn a 30-day
budget in ~2 days; 1000× means you're burning it in minutes (a hard outage).

Why this beats a static threshold like "alert if error rate > 1% for 5 minutes":

- A fixed threshold has **no notion of how much budget you have or how fast you're spending it.** It
  fires the same whether you have 90% of the month's budget left or 2%.
- A small elevation (say errors at 0.2% — below most static thresholds) sustained for *days* will
  silently blow your whole budget, and a static threshold never fires. Burn-rate catches the slow
  bleed.
- A static threshold tuned tight enough to catch the slow bleed will scream constantly on harmless
  blips → alert fatigue (Part F).

**Multi-window, multi-burn-rate** is the SRE-textbook answer. Run several burn-rate conditions in
parallel, each pairing a *high* burn rate (page immediately, catastrophe) with a *low* burn rate
(slow burn, ticket), and each gated by **two windows** — a long window (is the budget genuinely
burning?) and a short window (is it *still* burning right now, so we don't alert on a problem that
already self-resolved):

| Alert | Burn rate | Long window | Short window | Action | Catches |
|---|---|---|---|---|---|
| Fast burn | 14.4× | 1 hour | 5 min | **Page** | Acute outage — eats 2% of budget in an hour |
| Medium burn | 6× | 6 hours | 30 min | **Page** | Serious sustained degradation |
| Slow burn | 1× | 3 days | 6 hours | **Ticket** | The quiet bleed that static thresholds miss |

The long window confirms there's a real budget problem; the short window confirms it's *ongoing* (so
a fixed glitch that already recovered doesn't page someone). The result: **you page fast for fast
burns, ticket for slow burns, and rarely fire on noise** — alerts become proportionate to actual
user harm.

> **Say this:** "Instead of a static error-rate threshold, I'd alert on error-budget *burn rate* with
> multiple windows: a fast-burn condition pages within minutes for an acute outage, a slow-burn
> condition files a ticket for the quiet bleed a static threshold would never catch. Alerts then
> correspond to real budget being spent, not to arbitrary numbers — that's what kills alert fatigue."

---

## Part F — Alerting philosophy

The goal of an alert system is **a small number of actionable, urgent alerts that a human must respond
to** — and *nothing else*. Everything else is a dashboard or a ticket.

### Symptom-based vs cause-based

- **Symptom-based alerts** — page on what the *user experiences*: "checkout success rate dropped below
  SLO," "p99 latency exceeded budget," "the page won't load." These are the alerts that should page,
  because they mean *users are hurting right now*, regardless of the cause.
- **Cause-based alerts** — fire on an internal condition that *might* cause user pain: "CPU > 90%,"
  "disk 80% full," "one replica down." These are useful as *diagnostic* signals and capacity tickets,
  but mostly should **not page**, because (a) the system often absorbs them (90% CPU under SLO is
  fine), so they're false alarms, and (b) there are thousands of possible causes — you can't and
  shouldn't enumerate them all as pages.

The rule: **page on symptoms, diagnose with causes.** A box at 95% CPU that's still serving every
request under SLO is *not an incident* — it's efficiency. Page when users feel it (the SLO burns);
use the cause metrics (USE, Part D) to find *why* once you're already looking. Exceptions that
legitimately page on a cause are *leading indicators of imminent unavoidable symptom* — "disk fills
in 2 hours at current rate," "certificate expires in 24 hours," "you're out of error budget" — where
waiting for the symptom means waiting for the outage.

### Actionability and alert fatigue

Every paging alert must pass: **"Is a human required to act *right now*, and does this alert tell
them roughly what to do?"** If the answer is no — it's informational, it auto-recovers, or there's no
action — it must **not page**. Route it to a ticket or a dashboard.

- **Alert fatigue** is the real failure mode and it's a *safety* problem, not an annoyance: when a
  pager cries wolf, on-call engineers start ignoring it, miss the real one, burn out, and quit. A
  noisy alert that fires 50× a week and is always ignored is *worse than no alert* — it trains people
  to ignore the pager. Ruthlessly delete or downgrade alerts that are chronically non-actionable.
- Every alert should link a **runbook** (Part G) — what it means, how to diagnose, how to mitigate.
  An alert with no runbook is a 3am puzzle.

### Paging vs ticketing

| | **Page** (wake someone) | **Ticket** (handle in business hours) |
|---|---|---|
| Urgency | User-impacting *now*, can't wait | Important, not urgent |
| Examples | SLO fast-burn, total outage, payment failures | Slow-burn budget, disk filling in 3 days, single-node failure with redundancy intact |
| Test | "Would I want to be woken for this?" | "Can this wait till morning?" |

> **Say this:** "I page only on user-facing symptoms — SLO burn — and route causes to tickets and
> dashboards, because a box at 90% CPU that's still meeting SLO isn't an incident. The number one
> thing that kills an on-call rotation is alert fatigue, so every page must be actionable and have a
> runbook; anything that isn't, I delete or downgrade."

---

## Part G — Incident management

Detection and pretty dashboards are worthless if the *human* response is chaos. Staff engineers are
expected to know the operating model, not just the tooling.

### Severity levels

A shared severity scale so everyone calibrates response without debating it mid-incident:

| Sev | Meaning | Example | Response |
|---|---|---|---|
| **SEV1** | Critical — major outage, broad user/revenue impact | Checkout down globally; data loss | All-hands, page leadership, war room, IC now |
| **SEV2** | Significant — major feature degraded / subset of users | One region down; payments slow | Page on-call + IC, urgent |
| **SEV3** | Minor — limited impact, workaround exists | Non-critical feature broken | Business-hours, normal priority |

### The incident commander (IC) role

In any non-trivial incident, separate **coordination** from **hands-on debugging**. The **Incident
Commander owns the incident, not the fix** — they coordinate, decide, and communicate; they don't put
their head down in the code. Roles to name:

- **Incident Commander (IC)** — runs the incident: assigns tasks, tracks state, makes the call on
  mitigations, keeps the timeline. The single decision-maker, so there's no "who's in charge?"
  confusion.
- **Ops/Subject-matter leads** — the people actually debugging and applying fixes.
- **Communications lead** — owns external/internal updates (status page, stakeholders) so the
  debuggers aren't interrupted to write updates.
- **Scribe** — timestamps every action and decision (gold for the postmortem and for not repeating
  steps).

Why it matters: without a clear IC you get duplicated effort, conflicting mitigations applied
simultaneously (two people "fixing" it in opposite directions), and no one talking to stakeholders.
The IC is the antidote to the headless-chicken incident.

### The lifecycle: detect → mitigate → resolve → review

1. **Detect** — alert fires (ideally an SLO burn-rate page) or a report comes in. **MTTD** (Mean Time
   To Detect) is the clock here.
2. **Triage / declare** — assess severity, declare an incident, page the IC, open a channel.
3. **Mitigate** — **stop the bleeding first; understand it later.** Roll back the deploy, flip the
   feature flag off, fail over the region, shed load, scale up. **Mitigation ≠ root-cause fix** — and
   that distinction is everything: get users healthy *now* (mitigate), diagnose the root cause
   *after*. The most common rookie incident error is debugging root cause while users suffer instead
   of rolling back first.
4. **Resolve** — service restored to normal, incident closed.
5. **Review** — the **blameless postmortem** (below).

### MTTD and MTTR (define and decompose)

- **MTTD — Mean Time To Detect**: failure start → you know. Driven by good SLI alerting.
- **MTTR — Mean Time To Recover/Resolve**: failure start → service restored. The headline operational
  metric. It decomposes into **detect + acknowledge/respond + diagnose + mitigate + verify**, and you
  improve it by attacking whichever segment dominates. The biggest MTTR wins usually come from **faster
  mitigation** (one-click rollback, a tested failover, a kill-switch flag) — *not* faster root-causing,
  because mitigation doesn't require understanding. (Reliability ≈ a function of MTBF and MTTR;
  lowering MTTR is often cheaper than raising MTBF.)

> **Say this:** "I optimize MTTR before MTBF — failures are inevitable, so the lever is recovering
> fast. And I separate mitigation from root cause: roll back or flip the flag to make users healthy
> *now*, then diagnose at leisure. The fastest mitigations are a one-click rollback and a kill-switch
> feature flag." (Deploy/rollback mechanics: doc 13 §K.)

### Blameless postmortems

After every significant incident, a written postmortem: timeline, impact (users/revenue/budget),
root cause, what went well, what went poorly, and **concrete action items with owners and due dates**.

**Blameless** is the load-bearing word: focus on **systems and process, not on punishing the person**
who ran the command. The reasoning is practical, not just kind — if people get blamed, they hide
mistakes and stop sharing information, and you lose the very data you need to prevent recurrence. The
premise: a human could make that mistake means the *system* allowed it (no guardrail, no canary, a
footgun CLI), and the fix is a better system, not a more careful human. Hold the line on action items
actually getting done — a postmortem whose action items rot is theater.

### Runbooks

A **runbook** is a per-alert/per-service operational playbook: what this alert means, how to confirm
it, how to diagnose, how to mitigate (with exact commands), and when/how to escalate. Every paging
alert links one. Runbooks turn a 3am incident from "improvise under stress" into "follow the steps,"
which is what lets a generalist on-call handle a service they don't own day-to-day — and they're a
prime target for automation (a runbook step you run every time should become a script, then a
self-healing automation).

---

## Part H — Debugging distributed systems

### Correlation IDs

Generate a unique **correlation ID (request ID)** at the edge for every incoming request and
propagate it through *every* downstream call (headers for sync, message attributes for async) and into
*every* log line. Then one ID reconstructs the entire fan-out of a single request across dozens of
services from your logs. This is the cheapest, highest-leverage observability practice there is, and
it's the bridge between logs and traces (the trace ID *is* a correlation ID). Without it, debugging a
distributed request is correlating timestamps by hand across services — hopeless at scale.

### The tail-latency problem — why p99 matters and what causes it

This is a favorite deep-dive, so own it.

**Why the tail matters more than it looks (fan-out amplification).** If a single backend call has a
p99 of 100ms — only 1 in 100 is slow — and a user request must fan out to **100** backends *and wait
for all of them*, then the probability that *at least one* is slow is `1 − 0.99^100 ≈ 63%`. **Your
once-in-a-hundred tail just became the majority experience.** This is why high-fan-out systems
(search, feed assembly, anything that scatter-gathers) care intensely about p99/p99.9 of their
*components*: the slowest component dominates the user's latency, every time. The mean of a component
is irrelevant; its tail is the whole story.

**What *causes* tail latency (the usual suspects — name several):**

- **Queueing** — the dominant cause. A request that arrives when the server is momentarily busy waits
  in a queue. Queueing delay grows non-linearly as utilization approaches 100% (queuing theory: wait
  time ∝ 1/(1−utilization) — it explodes near saturation). This is *why* USE saturation (Part D) is
  the leading indicator of tail pain, and why running a box "hot" (95% util) trades a little money for
  a lot of tail latency.
- **GC pauses** — a stop-the-world garbage collection freezes a JVM/Go process for tens of ms to
  seconds at unpredictable times → that request lands in the tail. Affects a *random* subset, so it
  shows up as p99/p99.9, not p50.
- **Lock/resource contention** — threads blocked on a hot lock, a saturated connection pool, a hot DB
  row/partition. A slow shard or hot key makes a fraction of requests slow.
- **Noisy neighbors** — a co-located tenant/VM saturates a shared CPU/disk/network; your request on
  that host is slow through no fault of its own.
- **Cold caches, retries, slow disks, network blips, head-of-line blocking** behind a big request.

**The mitigation worth naming: hedged / backup requests.** For read-heavy fan-out, after a short delay
(e.g., past the p95) send the request to a *second* replica and take whichever responds first,
cancelling the other. This dramatically cuts the *tail* (any single slow replica no longer dominates)
at a small cost in extra load — the classic Dean & Barroso "tail at scale" technique. (Tie-in to doc
13: this is a latency-resilience pattern that complements timeouts/breakers.)

> **Say this:** "p99 isn't a vanity number — with fan-out, a 1% tail per component becomes the
> *majority* experience for a request that touches 100 of them, so the slowest component sets the
> user's latency. Tail latency is mostly queueing — which blows up as utilization nears 100% — plus GC
> pauses and contention. I'd attack it with headroom, then hedged requests against a second replica to
> chop the tail."

### Debugging across services with traces

When a metric alert says "checkout p99 is up," the trace is the localizer: pull a slow exemplar trace
(tail sampling guarantees you kept it) and read the flame graph — the long span *is* the culprit
service/operation. Then jump to that service's logs *filtered by the trace ID* to see exactly what
happened. That's the **metric → trace → log** workflow made concrete, and it's the answer to "how do
you debug a latency regression in a 30-service request path." **Exemplars** (linking a metric data
point directly to a representative trace) are the modern glue that lets you click from a spiking
latency graph straight to a slow trace.

---

## Part I — Capacity, saturation, and proactive scaling

You should detect "we're going to run out of capacity" *before* users feel it — that's a leading
indicator, and it's a cause-based signal that legitimately warrants a *ticket* (or a page if the
runway is short).

- **Saturation is the signal that predicts the cliff** (Part D): queue depth, pool wait time, run-queue
  length, connection-pool exhaustion. Utilization tells you how busy; saturation tells you how *close
  to falling over*. Because queueing delay explodes near 100% utilization (Part H), the safe operating
  point is well below 100% — you provision **headroom** (which is also doc 13's N+1 / load-redistribution
  argument: survivors must absorb a dead node).
- **Autoscaling** reacts to a signal (CPU, RPS, queue depth, or a custom SLI) — but it is **too slow to
  save you from a sudden spike** (new instances take seconds-to-minutes to boot and warm caches — doc
  13 §H). So: autoscaling handles *gradual* growth and diurnal cycles; **load shedding** (doc 13 §D) is
  your *fast* defense against the spike; **pre-provisioning/pre-warming** ahead of known events (sales,
  launches) handles the predictable surge. Say all three — autoscaling alone is a trap.
- **Capacity planning / headroom**: track growth trends, forecast, and provision so you're never
  running at the edge. The proactive version of an incident is a capacity ticket filed two weeks early.

---

## Part J — Chaos engineering and game days (brief — see doc 13 §L)

Observability and chaos are paired: chaos engineering injects controlled failure to *verify* your
resilience works, and **observability is what lets you see whether it did**. The connection to *this*
doc specifically:

- A **game day** is also a test of the *observability and on-call response*, not just the system: did
  the right alert fire? did the runbook work? could on-call find the problem with the traces/logs/
  dashboards available? It exercises the humans and the tooling together.
- Chaos experiments validate your **SLOs and alerting**: inject the failure, confirm the SLI moves and
  the burn-rate alert pages as designed. If you fail an AZ and no alert fires, your observability has a
  gap you just found cheaply.

(Mechanics — Chaos Monkey, blast-radius limits, abort switch — are in doc 13 §L. Don't re-derive them.)

---

## Part K — Deploy observability: watching a canary, auto-rollback on SLO breach

(Deploy *strategies* — canary, blue/green, flags — are doc 13 §K. This is the *observability* of a
deploy, which is where the two docs meet.)

A canary is only as good as the metrics you watch it with. The loop:

1. Route a small % of traffic to the new version (doc 13 §K).
2. **Compare the canary's RED metrics against the baseline (the old version) on the same dashboards** —
   error rate, p99 latency, and any key business SLI — *side by side*. Compare canary-vs-baseline, not
   canary-vs-historical, so you control for time-of-day and traffic-mix effects.
3. **Automated canary analysis (ACA):** a system statistically compares canary vs baseline and
   produces a pass/fail. If the canary's error rate or latency regresses beyond a threshold, **auto-
   rollback** — no human in the loop, because at 3am the human is slow. This is the concrete meaning
   of "automated rollback on SLO regression."
4. If healthy, ramp 1% → 10% → 50% → 100%, re-evaluating at each step.

> **Say this:** "I'd ship via canary and watch the canary's RED metrics against the baseline version
> side-by-side, with automated canary analysis that auto-rolls-back on an error-rate or latency
> regression. Most incidents are bad deploys (doc 13 §K), so making the deploy *observable* and the
> rollback *automatic* is one of the highest-leverage reliability investments — it collapses MTTR for
> the most common cause of outages."

This also closes the loop with error budgets (Part E): a deploy that burns budget on the canary is
caught at 1% of traffic instead of 100%, so the budget hit is ~1% of what an un-canaried bad deploy
would cost.

---

## Part L — How to weave observability into a system design answer

Interviewers reward the unprompted "how would I know this is failing?" reflex. Concrete moves:

1. **Attach an SLO to the design early**, alongside the non-functional requirements (doc 1, step 1):
   "Feed read SLO: 99.9% under 200ms over 28 days." Now availability and latency are *numbers* you
   design toward, and everything downstream (timeouts, replication, caching) has a target to justify it.
2. **For each critical component, name its RED dashboard and its key USE signal.** "The feed service:
   RED on the read endpoint; USE-saturation on the fan-out worker queue depth and the DB connection
   pool." This shows you'd actually operate it.
3. **In the wrap-up, do an observability pass** the same way you do a SPOF pass (doc 13 §M): "For each
   box — how would I detect it failing, in how long, and what alert fires?" Tie detection to MTTD and
   the burn-rate alerts.
4. **When asked "what if X is slow/failing,"** answer with the pillar workflow: "Burn-rate alert pages
   on the SLO → I pull a slow exemplar trace to localize which hop → filter that service's logs by the
   trace ID to see why → mitigate by rolling back / flipping the flag (doc 13), then root-cause in the
   postmortem." That single sentence demonstrates metrics, traces, logs, MTTR, and incident process at
   once.
5. **Volunteer the cardinality and tail-latency awareness** — "I'll keep user_id out of metric labels
   and put it in traces; and because this fans out to N services I care about component p99, not mean."
   These two details are disproportionately strong staff signals because they're things only operators
   know.

> **The framing that lands:** "A design isn't done when the happy path works — it's done when I can
> answer, for every component, *how I'd know it's failing, how fast, and what I'd do about it.*
> Observability and an SLO aren't an ops afterthought; they're how I decided where to spend the
> reliability budget in the first place."

---

### Self-check before the mock (answer these from memory)
- [ ] Observability vs monitoring — what can one answer that the other can't?
- [ ] Counter vs gauge vs histogram — and why store histograms, not a per-host p99?
- [ ] Why is the *mean* latency dangerous, and why can't you average percentiles across hosts?
- [ ] Explain a cardinality explosion: why it's an *availability* problem, and where user_id belongs.
- [ ] Head vs tail sampling — the tradeoff in one sentence each.
- [ ] Pull vs push metrics — what does pull give you for free? When is push better?
- [ ] What is OpenTelemetry and why does it matter in a design answer?
- [ ] RED vs USE — which subject, which viewpoint? Why is saturation the leading indicator?
- [ ] SLI vs SLO vs SLA — and why is the SLA looser than the SLO? Why is 100% the wrong target?
- [ ] What does an error-budget *policy* do, and what argument does it end?
- [ ] Explain multi-window multi-burn-rate alerting and why it beats a static error-rate threshold.
- [ ] Symptom vs cause alerts — what should page, what should ticket, and why?
- [ ] What's the #1 killer of an on-call rotation, and how do you prevent it?
- [ ] Incident roles: what does the IC own (and not own)? Mitigation vs root-cause — which comes first?
- [ ] MTTD vs MTTR — decompose MTTR; which segment gives the biggest win, and why?
- [ ] Why "blameless," in practical (not just ethical) terms?
- [ ] Tail latency: the fan-out amplification math, three causes, and the hedged-request mitigation.
- [ ] Walk the metric → trace → log debugging workflow on a latency regression.
- [ ] Why is autoscaling not your defense against a sudden spike? What is?
- [ ] How do you make a *deploy* observable, and what does auto-rollback-on-SLO-breach mean?
