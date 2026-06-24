# Topic 23: ML & AI System Design — Recommendations, Ranking, Feature Stores & LLM Apps

> **Why this topic now:** Staff interviews increasingly hand you "design YouTube recommendations,"
> "design a feed ranker," or "design a RAG-based Q&A assistant." These are **systems** problems
> wearing an ML costume. Nobody expects you to derive gradient descent on a whiteboard — they expect
> you to reason about *data flow, latency budgets, training/serving skew, freshness, and feedback
> loops*. This doc is the infrastructure view of ML. The math is somebody else's deep dive; the
> pipelines, stores, and serving paths are yours.

The one sentence to internalize: **an ML system is a data pipeline with a model bolted into the
middle.** Everything hard about it — skew, staleness, drift, the feedback loop — is a data problem,
not a model problem. Frame it that way in the room and you'll sound senior immediately.

---

## Part A — The ML system lifecycle as infrastructure

Treat the lifecycle as a loop with two clocks: an **offline** clock (batch, hours-to-days, optimize
for throughput and correctness) and an **online** clock (request path, milliseconds, optimize for
latency and availability). Almost every ML design question is really "which parts live offline, which
live online, and how do you keep them consistent?"

```
        OFFLINE (throughput)                         ONLINE (latency)
  ┌─────────────────────────────────────┐     ┌──────────────────────────────┐
  │ data collection → feature eng. →     │     │ request → feature lookup →    │
  │ training → eval → registry           │     │ candidate gen → rank → serve  │
  └───────────────┬─────────────────────┘     └──────────────┬───────────────┘
                  │ pushes model + features                   │ emits logs/labels
                  └──────────────────► feedback loop ◄────────┘
```

| Stage | What it does | Where it lives | The thing that bites you |
|---|---|---|---|
| **Data collection** | Log events (impressions, clicks, purchases), ingest source data | Stream (Kafka) + lake (S3) | Logging the *features as they were at inference time*, not as they are now |
| **Feature engineering** | Transform raw events into model inputs | Batch (offline) + stream (online) | The same transform must run both places — or you get skew |
| **Training** | Fit the model on historical labeled data | Offline, batch, GPU | Point-in-time correctness (no label leakage from the future) |
| **Evaluation** | Validate before promotion | Offline metrics, then online A/B | Offline win ≠ online win |
| **Serving** | Produce predictions for live requests | Online (real-time) or precomputed (batch) | Latency budget, GPU batching, fallback when the model is down |
| **Monitoring** | Watch data drift, model decay | Online + scheduled | You won't get a 500 — you'll get *quietly worse predictions* |
| **Feedback loop** | Serving logs become tomorrow's training labels | Closes offline ← online | The model trains on its own outputs → bias amplification |

> **Say this in the room:** "Before I draw a model anywhere, I want to nail down the data path —
> where features are computed, how labels are collected, and how I guarantee the features at training
> time match the features at serving time. That consistency is where ML systems actually fail in prod."

The single most important seam is the **offline/online boundary**. A feature like `user_avg_session_length_7d`
is cheap to compute in a nightly batch job but must also be *readable in <10ms* on the request path.
If the batch job and the online service compute it differently, your model sees one distribution in
training and another in production. That's training/serving skew, and it's the #1 ML production bug
(more on this in Part C).

---

## Part B — The recommendation/ranking architecture (the canonical ML design)

This is the design that shows up most. Search (Topic 11) and recommendations are the **same shape**:
narrow a huge corpus to a tiny ranked list under a tight latency budget. The dominant pattern is a
**multi-stage funnel**.

```
  Corpus (10^8 items)
        │  candidate generation / retrieval  — cheap, recall-oriented, ~10ms
        ▼
  Candidates (~1000)
        │  ranking — expensive model, precision-oriented, ~50ms
        ▼
  Ranked (~100)
        │  re-ranking — business rules, diversity, dedup, freshness
        ▼
  Final list (~10)   → served to user
```

### Why two (or three) stages? Latency vs quality.

You cannot run a heavy ranking model over 100 million items in 100ms. So you split the problem:

- **Candidate generation (retrieval).** Job: take you from 10⁸ → ~10³ with **high recall** — don't miss
  the good stuff. Must be *cheap per item* because it touches everything. Techniques: approximate
  nearest-neighbor (ANN) over embeddings, inverted-index lookups (tie directly to **Topic 11** —
  retrieval is retrieval), heuristic sources ("recently popular," "from people you follow"). You
  typically run **several retrieval sources in parallel** and union them.
- **Ranking.** Job: take ~10³ → ~10² with **high precision** — order them well. Now you can afford a
  heavy model (gradient-boosted trees, a deep neural net) with hundreds of features per item, because
  you're only scoring a thousand candidates, not a billion.
- **Re-ranking.** Job: apply what the model doesn't know — diversity (don't show 10 near-duplicates),
  business rules, freshness boosts, dedup, policy filters, sponsored-content blending.

> **The tradeoff sentence:** "I split retrieval from ranking because the cost structures are opposite.
> Retrieval is O(corpus) so it must be cheap and lean toward recall; ranking is O(candidates) so it
> can be expensive and lean toward precision. One model trying to do both either misses good items or
> blows the latency budget."

| Stage | Optimizes for | Cost per item | Typical latency | Model |
|---|---|---|---|---|
| Candidate gen | **Recall** | Very cheap | ~10ms | ANN / two-tower / heuristics |
| Ranking | **Precision** | Expensive | ~50ms | GBDT / deep net, 100s of features |
| Re-ranking | Diversity, policy | Cheap | ~5ms | Rules + light model |

### Two-tower models (conceptually)

The standard way to power retrieval. Train **two encoders**: a **user/query tower** that maps the
user+context to a vector, and an **item tower** that maps each item to a vector in the *same* space.
Train them so that relevant (user, item) pairs have high dot-product/cosine similarity.

The systems payoff: the item tower is **offline**. You precompute every item's embedding in a batch
job and load them into an ANN index. At request time you only run the *user* tower (one forward pass),
get a query vector, and do an ANN lookup. That's why two-tower retrieval is fast — the expensive half
(embedding 100M items) happened last night.

> **Say this:** "Two-tower decouples the two halves of relevance. Item embeddings are precomputed and
> indexed offline; only the user tower runs on the request path. Retrieval becomes a single forward
> pass plus an ANN query — both sub-10ms."

---

## Part C — Feature stores, training/serving skew, and freshness

This is the part most candidates skip and the part that separates "I've read about ML" from "I've run
ML in prod." A **feature store** is the system that serves features consistently to both training and
serving.

### The two stores

| | Offline store | Online store |
|---|---|---|
| **Purpose** | Build training datasets, batch scoring | Serve features on the request path |
| **Backed by** | Data warehouse / lake (BigQuery, S3, Parquet) | Low-latency KV store (Redis, DynamoDB, Cassandra) |
| **Access pattern** | Large range scans, joins, historical | Point lookups by entity key, <10ms |
| **Volume** | All history | Latest value per entity |
| **Optimize for** | Throughput, correctness | Latency, availability |

A feature is **defined once** and materialized to *both* stores from the same transformation logic.
That single-definition property is the whole point.

### Training/serving skew — the #1 ML production bug

Skew is when the feature values a model sees during **training** differ from what it sees during
**serving**. The model was fit on one distribution and is asked to predict on another. Predictions
silently degrade — no error, no alarm, just worse results.

Where skew comes from:

- **Two codepaths.** The data scientist computes `avg_purchase_value` in a pandas notebook for
  training; an engineer reimplements it in Java for the serving service. They round differently,
  handle nulls differently, use a different time window. Now the model is wrong in prod.
- **Time travel.** Training accidentally uses a feature value computed *after* the label event
  (label leakage) — the model looks brilliant offline and useless online.

> **The feature store's core job:** kill skew by making training and serving read features from the
> **same definition**. You write the transform once; the store materializes it to the offline store
> (for training set generation) and the online store (for serving). One source of truth, one
> distribution.

### Point-in-time correctness

When building a training set, for each labeled event you must join the feature values **as they were
at that event's timestamp** — not the current values. If a label is "user bought on Jan 3," the
features must be the user's state *as of Jan 3*, not today. This is a **point-in-time (as-of) join**,
and it's the offline store's most important capability. Get it wrong and you leak the future into
training, which is the most insidious form of skew.

> **Say this:** "I'd build training sets with a point-in-time join against the offline store — for
> each label, fetch the feature values as of that timestamp. Otherwise I leak future information and
> the offline metrics lie to me."

### Feature freshness

Features have different freshness requirements, and that decides *how* they're computed:

| Feature class | Example | Freshness | How it's computed |
|---|---|---|---|
| **Batch** | `user_lifetime_purchases` | Hours–days | Nightly batch job → online store |
| **Streaming** | `clicks_last_5_min` | Seconds | Stream processor (Flink/Kafka Streams) → online store |
| **Real-time / on-demand** | `time_since_last_action` | Request-time | Computed in the request itself |

Tie this to **Topic 18 (batch vs stream)**: batch features are a Spark job writing to the KV store;
streaming features are a Flink job maintaining windowed aggregates and upserting them. **Backfills**
matter — when you add a new feature, you must recompute its history over the offline store so you can
retrain; a feature with no history can't be used in training, only going forward.

---

## Part D — Embeddings + vector search / ANN at serving time

Retrieval over embeddings means **nearest-neighbor search** in a high-dimensional space. Exact NN is
O(N·d) per query — far too slow at 10⁸ items. So you use **Approximate** NN (ANN): trade a little
recall for orders-of-magnitude speed.

| Index | How it works | Strength | Tradeoff |
|---|---|---|---|
| **HNSW** (graph) | Navigable small-world graph; greedy walk to neighbors | Best recall/latency; great for high QPS | High **memory** (graph in RAM); slow/awkward incremental deletes |
| **IVF** (inverted file) | Cluster vectors into cells; search only nearest cells (`nprobe`) | Memory-efficient; tunable recall via `nprobe` | Recall drops near cell boundaries; needs training step |
| **PQ** (product quantization) | Compress vectors into codes | Massive memory savings; fits billions in RAM | Lossy → lower recall; usually combined (**IVF-PQ**) |

In practice: **HNSW** when recall and latency matter most and the index fits in memory; **IVF-PQ**
when the corpus is huge and you must compress to fit. The recall/latency/memory triangle is the
tradeoff to recite.

> **The tradeoff sentence:** "ANN trades exactness for speed. HNSW gives the best recall-latency curve
> but is memory-hungry and hard to update; IVF-PQ scales to billions by quantizing, at the cost of
> recall. I'd start with HNSW and move to IVF-PQ only when memory forces it."

**Vector DBs (Pinecone, Milvus, Weaviate, pgvector, Elasticsearch kNN).** These wrap an ANN index with
the operational stuff: persistence, replication, filtered search (ANN + metadata predicates), and
incremental upserts. Reach for a managed vector DB when you need filtered ANN, frequent updates, and
don't want to run a raw FAISS index yourself. Reach for a raw library (FAISS) when the index is mostly
static and you want maximum control. If you're already running Elasticsearch (Topic 11), its kNN
support may let you avoid a whole new system — call that out as the cheaper option.

> **When NOT to use a vector DB:** small corpus (brute force is fine), or your retrieval is genuinely
> keyword/filter-based (a normal inverted index beats embeddings on exact-match queries). Don't add a
> vector DB by reflex.

---

## Part E — Model serving

Two fundamental modes, and the choice is one of the first things to pin down.

### Batch (precompute) vs real-time inference

| | Batch / precompute | Real-time inference |
|---|---|---|
| **When predictions are made** | Ahead of time, on a schedule | On the request, on demand |
| **Stored where** | KV store (`user_id → recs`) | Computed live |
| **Latency at request** | A cache lookup (~1ms) | Full model forward pass |
| **Freshness** | Stale until next run | Always current |
| **Cost** | Predictable, can over-compute for inactive users | Scales with traffic |
| **Use when** | Recs change slowly; user set is bounded ("daily digest") | Inputs are request-specific (search query, session context) |

Many real systems are **hybrid**: precompute candidate sets per user offline, then rank in real time
with live session context. That's the pragmatic answer to "batch or real-time?" — *both, at different
funnel stages.*

> **Say this:** "I'd precompute the heavy, slowly-changing part — candidate generation per user — as a
> batch job into a KV store, and do the light, context-dependent part — ranking with this session's
> signals — in real time. Best of both: cheap retrieval, fresh ranking."

### Model-as-a-service, GPU batching, caching

- **Model-as-a-service.** Wrap the model behind an RPC service (TF Serving, TorchServe, Triton, or a
  plain gRPC service). Decouples model deploys from app deploys, lets you scale the model tier
  independently, and gives you one place to version, monitor, and roll back.
- **GPU batching.** GPUs are throughput devices — one request underutilizes them. **Dynamic batching**
  collects incoming requests for a few milliseconds and runs them as one batch. Classic
  latency/throughput tradeoff: bigger batches and longer wait windows = higher throughput, worse
  p99. Tune the max-batch-size and max-wait against your latency budget.
- **Caching predictions.** If the same (user, item) or same input recurs, cache the score. For LLMs,
  prompt/semantic caching (Part F) is the equivalent. Watch staleness — a cached prediction can
  outlive the feature values it was based on.

### A/B testing, shadow deployment, versioning, rollback

This is where serving meets *safe rollout*, and it's a strong staff signal.

- **Offline metrics never authorize a launch.** A model that wins on offline AUC can lose on the
  business metric online. You must A/B test.
- **A/B test (online experiment).** Route a slice of live traffic to the new model, compare the
  *business* metric (engagement, revenue, retention) with statistical significance, then ramp.
- **Shadow deployment (dark launch).** Send live traffic to the new model **but don't use its
  output** — log it and compare to the live model offline. Validates latency, error rate, and
  prediction distribution under real load with **zero user risk**. Do this *before* the A/B test.
- **Model versioning & registry.** Every model is an immutable, versioned artifact in a registry
  (version, training data snapshot, metrics, feature schema). Serving pins a version; you can pin a
  session/experiment to a version for reproducibility.
- **Rollback.** Because the model is a versioned artifact behind a service, rollback is "point serving
  at the previous version" — fast and boring, which is exactly what you want when a model regresses
  at 2am.

> **Say this:** "I'd shadow-deploy first to validate latency and prediction distribution with no user
> impact, then run an A/B test on the *business* metric — not offline AUC — and ramp on significance.
> Models are versioned artifacts in a registry, so rollback is just repointing the serving tier."

---

## Part F — Designing an LLM-powered application (RAG, agents, token economics)

Increasingly the second half of the interview. The canonical design is **RAG (Retrieval-Augmented
Generation)** — ground an LLM in your own/fresh data so it answers from facts instead of hallucinating.

> **Model default:** when you actually build one of these, default to the latest Claude models — e.g.
> **Claude Opus 4.x** for the hardest reasoning/agentic work, the **Sonnet** family for the
> high-volume balanced tier, **Haiku** for cheap/fast classification-style calls, and **Fable** when
> you want the most capable model available. In the room, say "I'd start on a frontier model like
> Claude Opus 4.x and push high-volume paths down to a smaller/cheaper model once I've measured
> quality." Pick the model from the latency/cost/quality tradeoff, exactly like any other tier choice.

### RAG architecture

```
  INGEST (offline)                          SERVE (online, per query)
  docs → chunk → embed → vector store        query → embed → retrieve top-k → assemble
                                              prompt (context + question) → LLM → answer
                                              (stream tokens back, with citations)
```

| Step | What it does | The decision that matters |
|---|---|---|
| **Chunk** | Split docs into passages | Chunk size + overlap: too big wastes context & dilutes relevance; too small loses context. ~200–500 tokens with overlap is a common start. |
| **Embed** | Vectorize each chunk | Same embedding model for chunks and queries — or retrieval breaks (a skew of its own) |
| **Vector store** | Index chunk embeddings | Same ANN tradeoffs as Part D; metadata filters for access control |
| **Retrieve** | Top-k nearest chunks for the query | k vs context budget vs noise; often add a reranker over the top-k |
| **Prompt assembly** | Stuff retrieved context + question into the prompt | Context-window management (below) |
| **Generate** | LLM produces the answer, grounded in context | Stream tokens; ask for citations to the retrieved chunks |

### Why RAG over fine-tuning (for fresh/proprietary data)

This is a near-guaranteed question. The answer:

| | RAG | Fine-tuning |
|---|---|---|
| **Best for** | Fresh, changing, proprietary *facts* | Teaching *behavior/format/style*, narrow domains |
| **Update cost** | Re-index a doc (seconds) | Retrain the model (hours–days) |
| **Freshness** | As fresh as your index | Frozen at training time |
| **Hallucination** | Lower — answer is grounded in retrieved text | Still hallucinates; bakes facts into weights |
| **Attribution** | Natural — cite the retrieved chunks | None |
| **Access control** | Filter at retrieval per user | Baked in — can't un-teach per user |

> **Say this:** "For fresh or proprietary *facts* I reach for RAG, not fine-tuning. Updating a fact is
> re-indexing one document, the answer is grounded and citable, and I can enforce per-user access
> control at retrieval time. Fine-tuning is for *behavior and format*, and it freezes knowledge at
> training time — wrong tool for facts that change."

### Context-window management, caching, guardrails

- **Context window.** Even with a large window, you can't (and shouldn't) stuff everything — more
  context = more cost, more latency, and the "lost in the middle" effect. Retrieve the *most relevant*
  k chunks, rerank, and budget the prompt: system instructions + retrieved context + question + room
  for the answer. For long multi-turn sessions, summarize/compact older turns rather than resending
  everything.
- **Caching — two kinds, both real cost levers.**
  - **Prompt caching:** cache the static prefix of the prompt (system instructions, fixed context) so
    repeated calls don't re-pay to process it. Huge cost/latency win when a large stable preamble is
    reused. The rule: stable content first, volatile content (the user's question) last — any byte
    change in the prefix invalidates the cache.
  - **Semantic caching:** if a *new* query is semantically near a previously answered one (embedding
    similarity above a threshold), return the cached answer. Saves an LLM call entirely. Risk:
    returning a stale or subtly-wrong answer for a query that only *looks* similar — tune the
    threshold and scope it to safe domains.
- **Guardrails.** Input side: prompt-injection defense, PII scrubbing, topic/abuse filters. Output
  side: validate format/schema, check for unsupported claims (does the answer cite retrieved
  context?), block policy violations. Treat the LLM as an untrusted component in the middle of your
  system — validate what goes in and what comes out.

### Evaluating LLM outputs

There's no single accuracy number. You combine:

- **Offline eval set** — curated query→expected-answer pairs, scored by exact/semantic match.
- **LLM-as-judge** — use a strong model (e.g. Claude Opus 4.x) to grade answers against a rubric for
  faithfulness (grounded in context?), relevance, and completeness. Scalable, but the judge needs its
  own validation against human labels.
- **Human review** — gold standard, expensive; sample it.
- **Online signals** — thumbs up/down, follow-up rate, escalation-to-human rate, task completion.

> **Say this:** "I'd evaluate on two axes: a fixed offline set with an LLM-as-judge for faithfulness
> and relevance before launch, and online signals — thumbs, follow-ups, escalations — after. The one
> I care most about for RAG is *faithfulness*: is every claim grounded in a retrieved chunk?"

### Latency / cost — token economics & streaming

LLM cost and latency scale with **tokens**, and input vs output tokens are priced and timed
differently (output is generated one at a time, so it dominates latency). Levers:

- **Stream the response.** Time-to-first-token, not time-to-full-answer, is the perceived latency.
  Streaming is nearly always the right default for user-facing generation, and it also avoids request
  timeouts on long outputs.
- **Right-size the model per call.** Route easy calls (classification, routing, extraction) to a small
  fast model; reserve the frontier model for hard reasoning. A cheap router model in front is a common
  pattern.
- **Cut tokens.** Prompt caching for the stable prefix; retrieve fewer/better chunks; cap output length.
- **Batch where you can.** Non-latency-sensitive bulk work (nightly summarization of new docs) goes
  through a batch path at a discount, not the interactive path.

### Agentic / tool-use systems (high level)

When the task needs the model to *act* — call APIs, query a DB, run code, search the web — you give it
**tools** (functions with typed schemas) and run an **agent loop**: the model decides which tool to
call, your harness executes it, feeds the result back, and the model continues until done.

Systems concerns at staff level:
- **The harness owns the loop and the trust boundary.** The model emits tool *requests*; your code
  decides whether to execute them. Gate destructive/irreversible actions (sends, deletes, payments)
  behind confirmation or policy.
- **Bound it.** Cap iterations, set timeouts, set a token/cost budget per task — agent loops can
  runaway.
- **Observability.** Log every tool call and result; you need the trace to debug *why* it did what it
  did.
- **Start simple.** A single LLM call or a fixed code-orchestrated workflow handles most tasks. Reach
  for an open-ended agent only when the task is genuinely multi-step and hard to specify in advance —
  agents cost more latency, more tokens, and more failure modes.

> **Say this:** "I'd start with the simplest thing that works — a single call or a fixed workflow —
> and only go agentic when the task genuinely needs open-ended tool use. Then the hard parts are the
> trust boundary (the harness gates side-effecting tools), bounding the loop (max steps, token
> budget), and tracing every tool call for debuggability."

---

## Part G — Monitoring ML in production

ML systems fail *silently*. The service returns 200s; the predictions just quietly get worse. You
monitor for that.

| Failure | What it is | How you detect it |
|---|---|---|
| **Data drift** | Input feature distribution shifts (new user behavior, seasonality) | Compare live feature distributions vs training (KL divergence, PSI); alert on shift |
| **Concept drift** | The *relationship* between features and label changes (what users want changed) | Watch the business metric and prediction-vs-outcome over time |
| **Model decay** | Performance degrades as the world moves away from training data | Track online metrics vs a baseline; schedule retraining |
| **Prediction skew** | Live prediction distribution diverges from offline/expected | Log and compare distributions (this is what shadow scoring catches) |

- **Shadow scoring.** Run the candidate (or current) model on live traffic and log predictions without
  serving them — compare distributions and catch regressions before they reach users. Same mechanism
  as shadow deployment, used continuously for monitoring.
- **Retraining cadence.** Drift is the signal to retrain. Some systems retrain on a schedule (nightly,
  weekly); mature ones retrain *when drift crosses a threshold*. Always re-validate (offline → shadow →
  A/B) before promoting a retrained model — a retrain is a deploy.

> **Say this:** "I'd monitor input drift and prediction-distribution skew continuously via shadow
> scoring, watch the business metric for concept drift, and trigger retraining on drift thresholds —
> with the retrained model going through the same shadow→A/B gate as any deploy. The failure mode I'm
> guarding against is silent degradation, not a 500."

---

## Part H — The feedback loop and its dangers

The feedback loop is what makes recommenders improve: serving logs (impressions, clicks) become the
next training set's labels. It's also where the subtlest failures live, and naming them unprompted is
a strong signal.

- **Popularity bias.** Popular items get shown more → get more clicks → look more relevant → get shown
  even more. The rich get richer; the long tail starves. Counter with exploration (show some
  uncertain/novel items), diversity in re-ranking, and inverse-propensity weighting.
- **Feedback loops / self-reinforcement.** The model trains on data *it generated by choosing what to
  show.* You never observe the counterfactual (what the user would have clicked on items you didn't
  show). The model's biases get baked into the training data and amplified each cycle.
- **Position bias.** Users click the top result because it's on top, not because it's best. If you
  train on raw clicks, you teach the model to predict *position*, not *relevance*. Counter by modeling
  position explicitly or de-biasing the labels.
- **The explore/exploit tradeoff.** Pure exploitation (always show the predicted-best) maximizes
  short-term metrics but starves the model of data on everything else and locks in bias. You need
  **exploration** — show some items the model is uncertain about — to keep the model learning and the
  catalog discoverable.

> **Say this:** "The feedback loop is what lets the recommender learn, but it's also a bias amplifier:
> the model only sees outcomes for items it chose to show, so its biases reinforce themselves. I'd
> budget for exploration, de-bias clicks for position, and enforce diversity in re-ranking — otherwise
> the system collapses onto a few popular items and stops learning."

---

## Part I — Worked mini-walkthrough

### Design 1: "Design a recommendation system" (e.g. a video feed)

1. **Requirements.** Functional: given a user, return a ranked feed of N videos. Non-functional: p99
   <200ms; 10⁸ items, 10⁷ DAU; reads ≫ writes; stale-by-minutes is fine (it's a feed, not a balance).
2. **Estimation.** 10⁷ DAU × ~20 feed refreshes/day ÷ 86,400 ≈ ~2.3k QPS average, peak ~3×. That QPS
   at 10⁸ items is exactly why a single heavy model won't fit the budget → **funnel**.
3. **High-level.** Client → gateway → recs service → [retrieval (ANN over two-tower item embeddings +
   "from follows" + "trending") → ranking model → re-ranking (diversity/policy)] → feed. Feature
   lookups hit the **online store** (Redis); the ranking model is a **model-as-a-service** behind gRPC.
4. **Offline path.** Nightly: item-tower embeddings → ANN index; batch features → online store;
   training set built with **point-in-time joins** from the offline store; train → registry.
   Streaming: `clicks_last_5_min`-type features via Flink → online store for freshness.
5. **Deep dives (pick the bottleneck).**
   - *Latency:* retrieval is ANN (~10ms over precomputed item embeddings), ranking scores only ~1000
     candidates (~50ms), GPU dynamic batching on the ranking tier. Precompute candidates per active
     user offline; rank live with session context (**hybrid**).
   - *Skew:* feature store with one definition materialized to both stores; point-in-time correctness
     in training.
   - *Feedback loop:* exploration budget + position de-biasing + diversity in re-rank, or the feed
     collapses onto popular items.
6. **Rollout.** New ranking model → shadow deploy (validate latency + distribution) → A/B on watch-time
   (the business metric, not offline AUC) → ramp on significance. Versioned in registry; rollback =
   repoint serving.
7. **Monitoring.** Drift on input features, shadow scoring on prediction distribution, watch-time as
   the concept-drift canary, retrain on drift threshold.

### Design 2: "Design a RAG-based Q&A assistant" (e.g. over internal docs)

1. **Requirements.** Functional: user asks a question, gets a grounded, cited answer over the company's
   docs. Non-functional: answers fresh as docs change; per-user access control; p95 time-to-first-token
   <1s; cost-controlled.
2. **Why RAG.** Facts are proprietary and change → RAG, not fine-tuning (re-index on change, grounded +
   citable, access control at retrieval).
3. **Ingest (offline).** Docs → chunk (~300 tokens, overlap) → embed (one embedding model) → vector DB
   with per-chunk metadata (owner, ACL, source). Re-index incrementally on doc change.
4. **Serve (online).** Query → embed (same model) → ANN top-k with **ACL filter** → rerank top-k → assemble
   prompt (system + context + question) → LLM (default **Claude Opus 4.x**, or Sonnet for the
   high-volume tier) → **stream** the answer with citations.
5. **Cost/latency.** Prompt-cache the stable system prefix; semantic cache for repeated questions
   (tuned threshold); right-size the model per call (a cheap router/classifier in front); stream for
   perceived latency.
6. **Guardrails.** Input: prompt-injection + PII filter. Output: verify every claim cites a retrieved
   chunk (faithfulness), block policy violations.
7. **Eval.** Offline set + LLM-as-judge on faithfulness/relevance before launch; thumbs/follow-up/
   escalation online. Faithfulness is the metric that matters most.
8. **Monitoring.** Retrieval hit rate, "no good chunk found" rate, answer latency/token cost,
   thumbs-down clusters → surface gaps in the doc corpus.

---

### Self-check before the mock (answer these from memory)
- [ ] Draw the ML lifecycle as an offline/online loop. Where's the feedback loop?
- [ ] Why two stages in recommendation (retrieval vs ranking)? What does each optimize for, and why can't one model do both?
- [ ] What is training/serving skew, and how does a feature store prevent it?
- [ ] What is point-in-time correctness and what breaks without it?
- [ ] Name the three feature freshness classes and how each is computed (tie to batch vs stream).
- [ ] HNSW vs IVF-PQ — the recall/latency/memory tradeoff, and when you'd pick each.
- [ ] What does a two-tower model let you precompute, and why does that make retrieval fast?
- [ ] Batch (precompute) vs real-time inference — when each, and what does a hybrid look like?
- [ ] Shadow deployment vs A/B test — what does each validate, and in what order?
- [ ] Why is offline AUC not enough to launch a model?
- [ ] RAG vs fine-tuning for fresh/proprietary facts — give the four-line answer.
- [ ] Prompt caching vs semantic caching — what each saves and its risk.
- [ ] Name three feedback-loop dangers (popularity/position/self-reinforcement) and one mitigation each.
- [ ] Data drift vs concept drift vs model decay — how do you detect each?
- [ ] For an LLM app: what's your default model, and how do you decide when to drop to a cheaper one?
