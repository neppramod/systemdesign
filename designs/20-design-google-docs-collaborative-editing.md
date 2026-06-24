# Design 20: Google Docs / a Real-Time Collaborative Editor

> **Why this problem is a staff filter:** a text editor looks trivial — a `documents` table, a
> `PUT /doc` that saves a blob, a `GET /doc` that loads it. That design is *correct for one editor* and
> *catastrophically wrong for two*, and the wrongness is the entire interview. The instant a second
> person types, "save the document" becomes a **lost-update race**: A and B both load version 5, both
> type, both save, and the last writer silently erases the other's work. The staff signal is that you
> recognize the problem is not storage — it's **concurrency control for a shared mutable sequence**:
> how do N users edit the *same* characters at the *same* time with low latency and still **converge**
> to one identical final state? You stop drawing "the WebSocket" and start drawing **the edit pipeline
> (optimistic local apply → op → server ordering/transform → broadcast)**, the **per-document
> collaboration session with a single ordering point**, and the **op-log-as-source-of-truth** behind it.
> The crown-jewel deep dive is **OT vs CRDTs** — and a staff candidate explains *how each achieves
> convergence*, not just names them. This walkthrough runs the
> [Topic 1 framework](../prep/01-framework-and-building-blocks.md) end to end and leans hard on
> [advanced theory / OT-vs-CRDT](../prep/21-advanced-distributed-systems-theory.md),
> [real-time/push](../prep/15-realtime-and-push.md),
> [event sourcing](../prep/25-workflow-orchestration-and-event-sourcing.md), and
> [security/authz](../prep/17-security-and-auth.md).

---

## 1. Requirements (5 min) — drive this, don't wait

I'll state the buckets out loud and **scope aggressively** to protect my 45 minutes.

### Functional (the verbs)
- Multiple users **edit the same document concurrently**; each sees others' edits in near-real-time.
- **Everyone converges** — after the dust settles, all editors see the *identical* final document.
  This is the non-negotiable correctness property; a "save" that loses a keystroke is a bug.
- **Optimistic local editing** — my keystroke appears instantly under my cursor (no server round-trip),
  even though the server is the ordering authority.
- **Presence + live cursors** — see who's in the doc and where their caret/selection is.
- **Comments & suggestions** — anchored annotations and tracked-change proposals.
- **Sharing & access control** — owner/editor/commenter/viewer roles, link sharing.
- **Offline edits + sync on reconnect** — edit on a plane, reconcile cleanly on landing.
- **Version history** — see and restore past states.

> **Scope out loud:** "I'll build the real-time concurrent-editing core (the convergence engine, the
> edit pipeline, the per-doc session), persistence as an op log + snapshots, offline sync, presence,
> comments/suggestions, and sharing/authz. I'll treat **rich-text formatting** as 'just more op types'
> rather than designing the full schema, and I'll skip spreadsheets/slides, the rendering/layout
> engine, and import/export unless you want one as the deep dive. Shout if you'd rather I redirect."

### Non-functional (where staff candidates separate)
- **Convergence is the top correctness promise.** All replicas that have applied the same set of edits
  **must** end in the same state — *Strong Eventual Consistency*
  ([Topic 21](../prep/21-advanced-distributed-systems-theory.md)). This is stricter than "eventually
  consistent": it's *eventually identical*, deterministically, regardless of edit order.
- **Intention preservation.** Beyond byte-equality: if A bolds a word while B inserts inside it, the
  merged result should reflect *both intentions*, not a coin-flip. This is what makes collaborative
  editing harder than a CRDT counter.
- **Latency:** local keystroke echo **< ~16 ms** (a frame — it's local, never a round-trip); remote
  edits visible **< ~100–200 ms p99** while both are online. Cursors/presence can be looser.
- **Consistency — split it deliberately** (the recurring staff move):
  - Document content → **converged, ordered, durable** (the hard guarantee).
  - Presence / live cursors / "X is typing" → **ephemeral, best-effort, lossy-OK** (a dropped cursor
    update fixes itself on the next keystroke).
- **Availability:** the edit path should feel always-up. Optimistic local apply means a user keeps
  typing through a brief disconnect; we reconcile on reconnect. A doc that "freezes" reads as broken.
- **Durability:** an accepted edit is never lost. The op log is the source of truth; the rendered
  document is a derived projection we can always rebuild.
- **Read/write profile is unusual:** this is **write-heavy *per active document*** (keystrokes are
  writes) but the *number of concurrently-hot docs* is modest. The bottleneck is **per-doc ordering
  throughput and connection fan-out within a doc**, not global QPS.

> **The single derived insight to state now:** "The product feature is concurrent editing; the
> *engineering* problem is **concurrency control over a shared sequence** — make concurrent edits
> **converge deterministically** while **preserving intention**, with a **single ordering point per
> document** so I never have to resolve a global merge. The document isn't a blob I save; it's the
> **deterministic projection of an ordered op log.** Get those two sentences and the rest is plumbing."

---

## 2. Estimation (3 min) — justify the per-doc-server and storage decisions

Numbers exist to *force* architecture, per [Topic 2](../prep/02-estimation-and-napkin-math.md). The
goal here is to show that this is **not** a QPS-firehose problem — it's a *per-document* problem — and
to size the op log.

**Assume:** ~1B documents total, **50M DAU**, ~100M editing sessions/day. Of docs open *right now*,
say **~5M concurrently active**, the vast majority with **1 editor**, a long tail with 2–10, and rare
"hot" docs (a shared meeting-notes doc, a viral template) with **50–100+** simultaneous editors.

### Edit (op) throughput — per doc, then aggregate

| Quantity | Calc | Result |
|---|---|---|
| Keystrokes/sec, active typist | bursty human typing | **~5–10 ops/sec** (batched/debounced to fewer) |
| Ops/sec, single busy doc (10 editors) | 10 × ~5, batched | **~tens of ops/sec per doc** |
| Concurrently active docs | given | **~5M** |
| Aggregate op QPS | 5M docs × low single-digit ops/sec avg (most docs idle/1-editor) | **~1–5M ops/sec aggregate** |
| Per-doc ordering load | even a *hot* doc | **only tens–low-hundreds ops/sec** |

> **The first money sentence:** "A *single* document — even a hot one — emits only **tens to low
> hundreds of ops/sec**. That comfortably fits **one authoritative server/thread ordering that doc**.
> So I can give each active document a **single-writer ordering point** and *still* scale to millions of
> docs by spreading docs across a fleet. The hard concurrency is **bounded per-doc**, which is exactly
> why a single ordering point is affordable here and isn't at, say, a global tweet firehose."

### Connections (fan-out is intra-document, not global)

| Quantity | Calc | Result |
|---|---|---|
| Concurrent editor sockets | ~fraction of 50M DAU live in a doc | **~tens of millions of sockets** |
| Sockets per gateway box (Topic 15) | RAM/throughput-bound | **~500k–1M** |
| Gateway fleet | tens of M ÷ 500k | **~tens–low-hundreds of boxes** |
| Fan-out per edit | broadcast to *co-editors of that doc only* | **bounded by editors-per-doc (≤ ~100)**, not followers |

> The fan-out story is *gentle* compared to Twitter/WhatsApp: an edit only reaches the handful of people
> in **that** doc. There is no celebrity fan-out — the unit of fan-out is a single document's session.

### Storage (op log + snapshots — tie to event sourcing)

| Quantity | Calc | Result |
|---|---|---|
| Bytes per op (compact: type, pos id, char, author, rev) | | **~50–100 B** |
| Ops/day (aggregate) | ~2M ops/sec avg × 86,400 | **~170B ops/day** |
| Raw op log/day | 170B × ~75 B | **~12 TB/day** |
| Per year | ×365 | **~4.5 PB/yr** of raw ops |
| Mitigation | **compact old ops into snapshots**, keep recent tail | log stays bounded per doc; cold ops tier to blob |

> **The second money sentence:** "**~12 TB/day of ops, append-only, read by `(doc_id, revision)`
> range** — that's a textbook **append-only event log** ([Topic 25](../prep/25-workflow-orchestration-and-event-sourcing.md)),
> not a relational document table. I store the *edits*, periodically **snapshot** the materialized doc
> so I don't replay millions of ops on open, and tier cold history to blob storage. The numbers picked
> event-sourcing-style storage and the single-writer-per-doc model for me; I didn't memorize them."

---

## 3. API design (3 min)

Two surfaces: a **persistent WebSocket** to the per-document collaboration session for the live edit
loop, and a thin **REST** surface for open/bootstrap/sharing (a cold open doesn't need the socket yet).

```
# Bootstrap / REST
POST /docs                         { title }                     -> { doc_id }
GET  /docs/{id}                    (authz checked)               -> { snapshot, base_revision, acl }
GET  /docs/{id}/history?cursor=                                  -> [revisions], nextCursor   # cursor, not offset
POST /docs/{id}/share              { principal, role }           -> ok                          # §6h
GET  /docs/{id}/comments                                        -> [comments]

# Persistent edit channel (one socket joins one doc's session)
WS   /docs/{id}/connect            (auth token in handshake)     -> upgraded socket; server replies JOINED{ base_revision }
     ── client→server frames ──
        EDIT      { client_op_id, base_revision, op }       # op = insert(posId,char) | delete(posId) | format(range,attr)
        CURSOR    { range }                                 # fire-and-forget presence
        ACK       { up_to_revision }                        # client confirms it applied server broadcasts
     ── server→client frames ──
        APPLY     { revision, op, author }                  # an ordered, (transformed) op to apply
        ACK_OP    { client_op_id, assigned_revision }       # "your op landed at revision R"
        PRESENCE  { user, cursor }                          # lossy
        SNAPSHOT  { revision, state }                       # on join / resync
```

- **`client_op_id`** is a client-generated UUID — the **idempotency key** that makes resend-on-flaky-network
  safe (the client resends an unACKed op; the server dedups). Load-bearing.
- **`base_revision`** on every `EDIT` is the document revision the client *had applied* when it generated
  the op. This is the **causal context** the server needs to know what to transform against — it's the
  single most important field in the whole protocol (§6c).
- **Cursor-based pagination** for history (offset breaks under constant appends), per Topic 1.
- **Auth + sharing role enforced at the gateway/session on JOIN**, stated once here (§6h, [Topic 17](../prep/17-security-and-auth.md)).

---

## 4. Data model (5 min) — access pattern picks the store

The dominant access patterns are: **(a)** append an op to a doc's log and assign it a revision;
**(b)** load a doc = "give me the latest snapshot + ops since"; **(c)** range-scan history by revision.
That's an **append-only, sequence-keyed log with periodic snapshots** — event-sourcing shaped
([Topic 25](../prep/25-workflow-orchestration-and-event-sourcing.md)), not a "save the blob" table.

**`ops` / revision log (the source of truth — append-only)**
- **Partition key = `doc_id`** → all of one doc's history co-located; ordering is per-doc.
- **Clustering key = `revision` ASC** → a single, gapless, monotonically increasing per-doc sequence
  number. Range scan "ops after revision R" is a cheap head-of-partition read.
- Columns: `client_op_id`, `author_id`, `op` (encoded), `server_ts`, `base_revision`.
- Append-mostly, range-by-revision → **wide-column (Cassandra/Scylla)** or a log store; *never* update
  in place. This is the **WAL of the document** ([Topic 1 building blocks](../prep/01-framework-and-building-blocks.md)).

**`snapshots` (the optimization, not the truth)**
- `(doc_id, revision) → materialized document state`. Periodic checkpoints so open = `load snapshot @
  rev N` + `replay ops N+1…latest`. Snapshots in blob storage; pointer row in the metadata DB.
  **Snapshots are derived; you can always rebuild from the log** ([Topic 25](../prep/25-workflow-orchestration-and-event-sourcing.md)).

**`documents` / metadata (relational — Postgres)**
- `doc_id`, `owner_id`, `title`, `current_revision`, `latest_snapshot_ref`, timestamps. Low-volume,
  benefits from transactions + the occasional join. SQL is correct here precisely because the access
  pattern *isn't* a firehose.

**`acl` (relational)** — `(doc_id, principal_id) → role` (owner/editor/commenter/viewer); plus link-share
settings. Small, consistency-sensitive, read on every JOIN (§6h, [Topic 17](../prep/17-security-and-auth.md)).

**`comments` / `suggestions`** — anchored to **stable position ids** (not raw offsets — see §6g),
`{comment_id, anchor_pos_id, author, thread, resolved}`.

**`presence` / `cursors` (Redis, ephemeral)** — `doc_id → {user → cursor}` with TTL; never durable.

> **SQL vs NoSQL, decided from access patterns not reflex:** the *ops log* is append-only, partition-by-doc,
> range-by-revision, multi-PB → **wide-column / log store**. The *metadata and ACL* are low-volume,
> relational, transactional → **Postgres**. Two stores for two access patterns — deliberate, not a
> contradiction.

---

## 5. High-level design (10 min) — happy path end to end

The governing structural decision blends two ideas: **dumb edge, smart core** from
[Topic 15 Part B](../prep/15-realtime-and-push.md) (scale sockets independently), **and**
**single-writer-per-document** (one authoritative session orders each doc, so I never resolve a global
merge).

```
   editors ──WS──►┌──────────────────────────────┐
   (per doc)      │   Connection Gateway fleet     │  terminate TLS, hold sockets, auth handshake,
                  │   (tens–100s of boxes)         │  route by doc_id → owning collab server. NO logic.
                  └──────────────┬─────────────────┘
                    doc_id→server│ (consistent hash / registry)
       ┌───────────────┐         ▼
       │ Doc-Ownership │   ┌────────────────────────────────────┐
       │ Registry      │◄──│  Collaboration Server (per doc)      │  THE ordering point:
       │ (etcd/ZK lease│   │  ── in-memory: doc state, revision,  │  dedup → transform/merge → assign
       │  per active   │   │     connected editors, recent ops ── │  revision → broadcast → append to log
       │  doc)         │   └───────┬──────────────┬───────────────┘
       └───────────────┘           │ append        │ broadcast APPLY
                                    ▼               ▼ (to co-editors' sockets)
                            [ ops log ]      [ presence (Redis) ]
                            (Cassandra)
                                    │ periodic
                                    ▼
                            [ snapshots ]  ──► blob storage   ;   [ documents / acl ] (Postgres)
```

**Walk one keystroke through it (two editors A and B, both online):**

1. **A types "x".** The client **optimistically applies it locally instantly** (< one frame) so the
   caret never lags. It sends `EDIT{client_op_id, base_revision=5, op=insert(...)}` to its gateway.
2. The gateway (no logic) routes by `doc_id` to the **collaboration server that owns this doc** (looked
   up / leased in the ownership registry, §6e).
3. The collab server — the **single ordering point** — does, *in order*: **dedup** on `client_op_id` →
   **transform/merge** A's op against any ops committed since A's `base_revision=5` that A hadn't seen
   yet (§6b/§6c) → **assign the next revision = 6** → **append the (transformed) op to the durable ops
   log** → **then ACK** A (`ACK_OP{client_op_id, assigned_revision=6}`) and **broadcast `APPLY{rev 6,
   op}` to every *other* connected editor** (B). *Durability/ordering before broadcast.*
4. **B's client** receives `APPLY{rev 6}`, transforms it against any *local* unacknowledged edits B has
   in flight, applies it, and advances to revision 6. Now A and B agree.
5. **A late joiner C** opens the doc → gets `SNAPSHOT{revision, state}` (latest snapshot + replayed
   tail) and then the live `APPLY` stream from revision N onward.
6. **A goes offline mid-edit?** A keeps typing locally against its last-known revision; ops queue. On
   reconnect, A sends its queued ops with their `base_revision`s; the server transforms each against
   everything committed in the interim and integrates them (§6d).

> Keep it this simple first, then evolve. The two arrows carrying the staff signal are
> **"all of a doc's edits funnel through one ordering point"** (single-writer → no global merge) and
> **"append to log → then broadcast"** (the log is the truth; the live stream is derived). Everything in
> the deep dives hangs off those two.

---

## 6. Deep dives (15 min) — where the round is won

I'd propose the order: *"The crux is **how concurrent edits converge** — OT vs CRDTs, in depth. Then
the **edit pipeline** (optimistic apply, ordering, out-of-order handling), **doc ownership/routing**,
**offline sync**, **persistence as an op log**, then comments, authz, presence. Can I start with the
convergence engine?"*

### 6a. The core problem: why "just save it" loses data, and what convergence means

Two editors load revision 5. A inserts "cat" at position 0; B inserts "dog" at position 10. If each
client naively applies its *own* op and sends the *resulting full document*, the last save wins and one
edit vanishes — the **lost-update problem**. Even sending **operations** doesn't trivially work: A's op
says `insert at position 10`, but on B's replica a concurrent insert already shifted everything, so
applying A's op verbatim lands the text in the **wrong place** — the replicas **diverge**.

So the requirement, precisely: replicas that have seen the **same set of ops** must reach the **same
state** regardless of the **order** they arrived — *Strong Eventual Consistency*
([Topic 21 Part E](../prep/21-advanced-distributed-systems-theory.md)) — **and** the result must
**preserve user intention** (both inserts survive, in a sensible place). Two families solve this.

### 6b. Concurrency control IN DEPTH — Operational Transformation (OT)

**The mental model: edits are operations; you *rewrite* an op against the ops it didn't see so its
intention is preserved on top of the new state.**

- Ops are compact: `insert(pos, char)`, `delete(pos)`, `format(range, attr)`. The document stays a
  **plain sequence** with **no per-character metadata**.
- The magic is the **transformation function** `T(op_a, op_b)`: given two ops generated against the
  *same* base state, it produces `op_a'` = how `op_a` should be rewritten to apply *after* `op_b`.
  Example: A = `insert("X", 2)`, B = `insert("Y", 1)`, both against base. B's insert at position 1
  shifts everything right, so `T(A, B)` rewrites A to `insert("X", 3)`. Symmetrically B is transformed
  against A. Apply A-then-B' on one replica and B-then-A' on the other → **identical result**.
- For this to converge, the transform must satisfy the **transformation properties** (TP1: applying
  `a` then `T(b,a)` equals `b` then `T(a,b)`; TP2 for 3-way reorderings). The system funnels all ops
  through a **central server** that assigns a total order, so clients mostly transform against a *linear*
  history — which sidesteps the brutal TP2 cases.

**How convergence is achieved:** the server assigns each accepted op a **revision number** (a total
order). A client's op carries the `base_revision` it was generated against; the server transforms it
forward against every op committed since (`base_revision+1 … current`), commits it at the next revision,
and broadcasts. Each client, on receiving a remote op, transforms it against its own **in-flight,
not-yet-acknowledged** ops before applying. Everyone applies a consistently-transformed sequence → same
state.

> **OT's tradeoffs, said cold** ([Topic 21 Part F](../prep/21-advanced-distributed-systems-theory.md)):
> "**Strength:** compact ops, zero per-character metadata, the doc stays a plain string — memory-cheap
> and great for huge documents. **Weakness:** the transformation functions are **notoriously hard to get
> right** — the literature is littered with OT papers later shown buggy, especially TP2 — and classic OT
> **effectively assumes a central server** to linearize ops; true peer-to-peer OT is very hard. **Google
> Docs uses OT**, and the central-server assumption is *fine* for Docs because there already is one."

### 6c. Concurrency control IN DEPTH — CRDTs (RGA / sequence CRDTs)

**The mental model: every element carries a stable unique identity; concurrent ops *commute* by
construction, so there's nothing to transform and no coordinator required.**

- A **sequence CRDT** (RGA, or Logoot/LSEQ-style) gives every inserted character a **unique, densely-
  orderable position identifier** — not an integer offset (offsets shift), but an id drawn from a dense
  total order so you can always mint an id *between* any two existing ids
  ([Topic 21 Part E.2](../prep/21-advanced-distributed-systems-theory.md)).
- An insert says "place character with id `p`, between neighbors `(left_id, right_id)`." A delete
  **tombstones** the element by id (doesn't renumber anything).
- **How convergence is achieved:** because each element's identity and relative order are **intrinsic
  to the op** (not to the current positions), concurrent inserts "at the same place" **deterministically
  interleave the same way on every replica** — the merge is **commutative, associative, idempotent**.
  Two replicas that received the same ops in *any* order, with duplicates, end **byte-identical**. This
  is **Strong Eventual Consistency by construction** — provably convergent, no transform proofs, no
  central authority.
- Two flavors ([Topic 21 Part E.1](../prep/21-advanced-distributed-systems-theory.md)): **op-based
  (CmRDT)** ships tiny ops but needs **reliable, exactly-once, causal-order broadcast**; **state/δ-based
  (CvRDT)** ships (delta) state and tolerates loss/dup/reorder. **Yjs, Automerge** are the production
  text CRDTs; **δ-state CRDTs** are the modern middle ground.

> **CRDT tradeoffs, said cold:** "**Strength:** merge is **provably convergent** — no fragile transform
> functions — and needs **no central coordinator**, so it's **naturally peer-to-peer and offline-first**.
> **Weakness:** every element carries a unique id and deletes leave **tombstones** → **metadata/memory
> overhead** and the position-id space can grow; historically a problem for very large docs, though
> Yjs/Automerge have largely engineered it away with compact ids and run-length encoding."

### 6c′. OT vs CRDT — the pick, and *why* the industry split (the staff differentiator)

| | OT (Google Docs) | CRDT (Yjs/Automerge, Figma-flavored) |
|---|---|---|
| Convergence mechanism | Rewrite ops via transform functions against a total order | Commutative merge by stable element id — no transform |
| Coordinator | **Effectively needs a central server** to linearize ops | **Not required** — P2P / offline-friendly |
| Per-element metadata | None (plain sequence) — memory-cheap | Unique ids + tombstones — memory overhead |
| Correctness difficulty | Transform functions hard/bug-prone (TP2) | Provably convergent by construction |
| Offline / local-first | Awkward (depends on server ordering) | Natural |
| Why chosen | Predates mature text CRDTs; server-centric was already true | Local-first, multiplayer, P2P era |

> **The honest answer** ([Topic 21 Part F](../prep/21-advanced-distributed-systems-theory.md)):
> "**Docs uses OT** because it was built **server-centric before text CRDTs were mature**, and OT's
> compact ops were efficient on a server that already orders everything — so OT's biggest weakness (it
> needs a coordinator) **cost Docs nothing**. **Newer/local-first apps (Figma's multiplayer is
> CRDT-flavored, Notion blocks, Yjs, Automerge) pick CRDTs** because they **converge with no central
> authority and work offline** — at the cost of metadata. Neither is strictly better; it's a
> **coordinator-vs-metadata trade**. **My pick for *this* design:** since I'm already building a
> single-writer-per-doc collaboration server (§6e), **I have a coordinator for free**, so **OT** is the
> natural fit — compact ops, no tombstone bloat, and the central server makes the hard TP2 cases mostly
> vanish. If the requirement were **offline-first or P2P with no reliable server**, I'd flip to a
> **sequence CRDT** without hesitation."

### 6d. The edit pipeline: optimistic apply, ordering, and out-of-order / concurrent edits

The end-to-end loop, with the failure cases that actually trip people up:

1. **Optimistic local apply.** The client applies its own op to local state *immediately* — the caret
   must never wait a round-trip. It adds the op to a **pending (unacknowledged) buffer** tagged with
   `client_op_id` + `base_revision`.
2. **Send + server ordering.** The collab server **dedups** (`client_op_id`), **transforms** the op
   against everything committed since its `base_revision`, **assigns the next revision**, **appends to
   the log**, then **ACKs the sender** and **broadcasts** to others.
3. **Receiving a remote op.** A client that has its own ops *in flight* must **transform the incoming
   remote op against its pending buffer** before applying (OT), or just merge by id (CRDT). Then it
   advances its local revision.
4. **Receiving its own ACK.** The client removes that op from the pending buffer and advances its
   acknowledged revision; any still-pending ops are now rebased onto the new revision.

**Handling out-of-order & concurrency explicitly:**
- **Out-of-order arrival** is impossible to *apply* out of order because revisions are a **gapless
  sequence**: a client that receives `APPLY{rev 9}` while still at rev 7 **buffers it** until rev 8
  arrives, then applies 8 then 9. This is the causal-buffering trick from
  [Topic 21 Part D](../prep/21-advanced-distributed-systems-theory.md) — apply only when the predecessor
  is present.
- **Concurrent edits to the same spot** are *exactly* what OT transform / CRDT id-ordering resolve;
  the single ordering point gives a definitive "who was first" that both clients then respect.
- **The `base_revision` is the causal context** — it tells the server *which* prefix of history the
  client had seen, hence what to transform against. Without it the server can't know whether two ops
  are concurrent or sequential.

> **Why a single per-doc revision counter and not a global one:** a global sequencer would be a
> throughput ceiling and SPOF; per-doc, the counter lives on that doc's owning server and contention is
> bounded by *that doc's* edit rate (§2: tens of ops/sec). This is the document analog of WhatsApp's
> per-conversation `seq` — order *within the unit users perceive*, never globally.

### 6e. Sharding / ownership: single authoritative server per active document

The defining structural choice: **exactly one collaboration server owns an active document at a time**
(single-writer-per-doc). That's what lets me assign a clean total order **without consensus on the hot
path** — there's only one writer, so there's nothing to agree on.

- **Routing:** the gateway maps `doc_id → owning server` (consistent hashing on `doc_id`, or a lookup
  in a **doc-ownership registry**). Every editor of a doc is routed to the **same** server, so they
  share one in-memory doc state + revision counter.
- **Ownership as a lease.** A server acquires a **lease** on a doc (etcd/ZooKeeper,
  [Topic 8](../prep/08-consensus-and-coordination.md)) when the first editor joins; it holds the
  authoritative in-memory state + recent ops, appending to the durable log behind it. The lease prevents
  *two* servers from both claiming ownership and creating split-brain (which would fork the revision
  sequence — the one thing that breaks convergence).
- **Idle eviction:** when the last editor leaves, the server flushes a snapshot, releases the lease, and
  drops the doc from memory. Reopening re-leases (possibly elsewhere).
- **Hot doc (50–100 editors):** still **one** owning server — and §2 showed even a hot doc is only
  tens–low-hundreds of ops/sec, so one server handles it. The cost there is **broadcast fan-out**
  (≤ ~100 sockets), not ordering throughput. If a single doc ever truly exceeded one box (e.g. a
  10k-viewer live doc), I'd split **editors vs viewers**: one ordering server + read-replica fan-out
  servers that just relay the APPLY stream to viewers.

> **The tradeoff stated:** "Single-writer-per-doc trades a little availability (a doc is briefly
> unavailable during failover) for **enormously simpler correctness** — strong per-doc consistency via
> one ordering point, no distributed merge, no consensus per keystroke. Given convergence is the top
> requirement, that's the right trade. It's the same instinct as a single-leader partition in a
> sharded DB ([Topic 4](../prep/04-sharding-and-partitioning.md))."

### 6f. Persistence: op log + snapshots, rebuild, and offline sync (tie to event sourcing)

This is **event sourcing** applied to a document
([Topic 25 Part E](../prep/25-workflow-orchestration-and-event-sourcing.md)):

- **The op log is the source of truth.** Every accepted op is appended at `(doc_id, revision)`. The
  rendered document is a **projection** — "fold the ops." Per-doc ordering is enforced by the
  **revision = stream version** (optimistic concurrency: an op committing at the wrong base revision is
  rejected/rebased — [Topic 25](../prep/25-workflow-orchestration-and-event-sourcing.md)).
- **Snapshots for performance.** Replaying millions of ops on every open is absurd, so we periodically
  store a snapshot of the materialized doc at revision N; open = `load snapshot @ N` + `replay N+1…latest`.
  **Snapshots are an optimization, never the truth** — we can always rebuild from the log (which also
  gives **version history / time-travel restore for free** — the history *is* a product feature here, the
  exact case [Topic 25](../prep/25-workflow-orchestration-and-event-sourcing.md) says justifies event
  sourcing).
- **Log compaction / tombstone GC:** old ops behind a durable snapshot can be compacted; CRDT tombstones
  GC'd once all replicas are known to have seen the delete (causal-stability). Keep a recent tail hot;
  tier cold ops to blob storage.
- **Offline edits + sync on reconnect.** Offline, the client keeps applying ops locally against its last
  base revision, queued with `client_op_id`s. On reconnect it replays the queue; the server transforms
  each against everything committed in the interim (OT) or merges by id (CRDT) and integrates them. This
  is why **CRDTs shine for heavily-offline products** — merge-after-long-divergence is their native case;
  OT's transform chains get long and gnarly after a long offline window, which is a real reason
  local-first apps prefer CRDTs.

### 6g. Comments & suggestions

- **Anchoring is the subtle bit:** a comment on "characters 10–15" must survive other people inserting
  text before position 10 (which would shift raw offsets). So **anchor to stable position ids**, not
  integer offsets — the *same* densely-orderable ids the CRDT uses, or in OT a side-table of anchors that
  is **transformed alongside ops** (an insert before an anchor shifts the anchor). Get this wrong and
  comments drift onto the wrong text — the classic collaborative-editor bug.
- **Comments themselves are low-frequency, durable, queryable** → store relationally (`comments` table),
  not in the hot op stream. They ride a lighter path than keystrokes.
- **Suggestions / tracked changes** = ops that are **staged, not committed to the canonical sequence**
  until accepted. Model them as a parallel op layer keyed to a base revision; accept = transform onto and
  fold into the main sequence; reject = drop. Same transform machinery, different commit point.

### 6h. Access control / sharing (tie to authz)

- **Role model** ([Topic 17](../prep/17-security-and-auth.md)): per-doc ACL `(doc_id, principal) → role`
  ∈ {owner, editor, commenter, viewer}, plus link-share settings (anyone-with-link: editor/viewer).
  This is **resource-scoped RBAC**, checked against the doc, not a global role.
- **Enforced at JOIN and per-op.** On `WS /docs/{id}/connect` the gateway/session checks the ACL: a
  *viewer* gets the APPLY stream but **the server rejects their EDITs**; a *commenter* may write comments
  but not content ops. Enforcement is **server-side at the ordering point** — never trust the client to
  self-limit.
- **Sharing changes mid-session** (owner revokes editor) must take effect live: the session re-checks on
  ACL change and **downgrades or disconnects** the affected socket. Cache ACLs in the session with
  short TTL + invalidation on change.
- **Auth at the gateway handshake** (token → identity), authz at the session (identity + doc → role),
  per [Topic 17](../prep/17-security-and-auth.md). State once, don't re-explain.

### 6i. Presence + live cursors (tie to realtime)

Presence here is **cheap** — fan-out is bounded by editors-in-this-doc (≤ ~100), unlike Topic 15's
social-presence monster.

- **Live cursors** are **ephemeral, fire-and-forget**: broadcast `CURSOR{range}` over the **lossy Redis
  backplane / direct session relay**, never the durable log — losing one is fixed by the next keystroke.
- **But cursors must transform too:** my caret is "after position id p"; when a remote insert lands
  before p, my displayed cursor shifts. Cursors are transformed against ops just like anchors (§6g) — a
  detail juniors miss, producing cursors that visibly jump to the wrong place.
- **Presence/roster** (who's in the doc, the colored avatars) = `doc_id → {user}` in Redis with TTL +
  heartbeat; no heartbeat → expire → implicitly left, which **self-heals ungraceful disconnects** (the
  half-open-socket problem, [Topic 15 Part C](../prep/15-realtime-and-push.md)).

---

## 7. Wrap-up (3 min) — bottlenecks, failure modes, SPOFs

- **The collaboration server for a hot doc dies (the headline failure mode).** Every editor's socket
  drops. Recovery: the **lease expires**, a new server **acquires it, rebuilds in-memory state from
  `latest snapshot + replay tail of the op log`** (this is why the log is the truth and snapshots are
  just speed), and clients **reconnect and resync from their last acknowledged revision**. **No accepted
  op is lost** because every op was appended to the durable log *before* broadcast. *This is the whole
  reason the log — not the in-memory doc — is the source of truth.* The only at-risk edits are ops that
  were optimistically applied locally but never ACKed — and those are still in the client's pending
  buffer, so the client **resends them** on reconnect (idempotent via `client_op_id`).
- **Split-brain — the scariest mode and why we lease.** If two servers both believed they owned a doc,
  they'd fork the revision sequence and convergence would break irreparably. The **single-owner lease**
  (etcd/ZK, [Topic 8](../prep/08-consensus-and-coordination.md)) is precisely what prevents this; fencing
  tokens reject a stale ex-owner's late writes.
- **Remaining bottlenecks:** (1) **a single mega-doc** (thousands of live editors) exceeding one
  ordering box → split editors-vs-viewers with read-fanout relays (§6e); (2) **op-log storage growth**
  (multi-PB/yr) → snapshot + compaction + cold-tiering; (3) **CRDT tombstone/metadata growth** if I'd
  chosen CRDTs → causal-stable GC.
- **Failure modes named:** *gateway dies* → sockets reconnect, land elsewhere, resync from revision (no
  loss, log is durable); *Redis presence/cursor backplane drops a message* → no harm, ephemeral by
  design, next keystroke corrects; *ops-log node down* → quorum writes ride it out
  ([Topic 5](../prep/05-replication-and-consistency.md)); *long offline client reconnects* → transform/
  merge its queued ops against the interim history (CRDT does this most gracefully).
- **SPOFs — none global by design.** Collab servers are a fleet, each doc owned by one of them; the
  ownership registry (etcd/ZK) is itself a Raft cluster; the op log and metadata DB are replicated;
  presence is ephemeral. The closest hot spot is a single hot doc's owning server — mitigated by lease
  failover + the editors/viewers split.
- **With more time:** geo-distribution (route a doc's editors to one region's owning server to keep the
  ordering point coherent; cross-region replicate the log); rich-text/format-op transform details;
  CRDT migration path for offline-first; rate-limiting/backpressure on pathological paste storms
  ([Topic 13](../prep/13-resilience-and-failure-handling.md)); end-to-end encrypted docs (which, like
  WhatsApp, would forbid server-side OT — a reason E2EE collaborative editors *must* use client-side
  CRDTs).

---

## What made this staff-level

- **Reframed "save a doc" as concurrency control over a shared sequence** — named the lost-update race,
  defined **convergence (SEC) + intention preservation** as the real requirements, and made the document
  **the projection of an ordered op log** rather than a blob. That inversion is the whole design.
- **Explained OT and CRDTs by their *convergence mechanism*, not by name** — OT's transform-against-a-
  total-order (and TP1/TP2, and *why* the central server makes them tractable) vs CRDTs' commutative
  merge-by-stable-id — then **picked OT with a justification tied to already having a coordinator**, and
  said exactly when I'd flip to CRDTs (offline-first / P2P). Tied cleanly to
  [Topic 21](../prep/21-advanced-distributed-systems-theory.md).
- **Explained *why the industry split*** — Docs is OT because it predates mature text CRDTs and was
  server-centric (so OT's coordinator cost was free); Figma/Notion/Yjs/Automerge are CRDT because the
  local-first/P2P era values coordinator-free offline convergence. "Coordinator-vs-metadata trade," not
  "one is better."
- **Made single-writer-per-document the load-bearing structural choice** — one leased ordering point per
  doc gives strong per-doc consistency with **no consensus on the hot path**, and the estimation
  (**tens of ops/sec even for a hot doc**) is what *proved* one server suffices. Let the numbers pick the
  architecture.
- **Got the edit pipeline right at altitude** — optimistic local apply, `base_revision` as causal
  context, gapless-revision buffering for out-of-order, `client_op_id` idempotency, and pending-buffer
  rebasing — instead of waving at "the WebSocket."
- **Treated persistence as event sourcing** ([Topic 25](../prep/25-workflow-orchestration-and-event-sourcing.md)) —
  log-as-truth, snapshots-as-optimization, rebuild-on-failover, version-history-for-free, and offline
  sync as merge-after-divergence.
- **Named the subtle bugs before being asked** — comment/cursor **anchoring to stable ids** (not
  offsets), suggestions as a staged op layer, and **split-brain** as the convergence-killer that the
  ownership lease exists to prevent.

---

### Self-check before the mock (answer these from memory)
- [ ] State the core problem: why does "load → edit → save" lose data with two editors? Define
      convergence (SEC) *and* intention preservation.
- [ ] OT: what does a transformation function do? Give the `T(insert@2, insert@1)` example and explain
      how a central server + revision numbers make it converge. What are TP1/TP2 and why are they hard?
- [ ] CRDT (sequence/RGA): what is a position id and why not an integer offset? How does merge-by-id
      converge with *no* transform and *no* coordinator? Op-based vs state/δ-based network needs.
- [ ] Why does Docs use OT and Figma/Notion/Yjs/Automerge use CRDTs? Give the coordinator-vs-metadata
      trade and say which *you'd* pick for this design and why.
- [ ] Trace one keystroke through the pipeline: optimistic apply → server (dedup/transform/assign-rev/
      append) → broadcast → remote client transform. Where does "append before broadcast" sit and why?
- [ ] What is `base_revision` for? How do you handle an `APPLY{rev 9}` arriving while you're at rev 7?
- [ ] Why single-writer-per-document? How does the ownership lease prevent split-brain, and why would
      split-brain be catastrophic? Why does one server suffice even for a hot doc (cite the estimate)?
- [ ] Persistence: why is the op log the source of truth and the snapshot just an optimization? How do
      you rebuild a doc on failover, and how does this give version history for free?
- [ ] Offline edits: how do you reconcile on reconnect, and why are CRDTs more graceful than OT here?
- [ ] Why must comment anchors and cursors be transformed too? What breaks if you anchor to raw offsets?
- [ ] How is authz enforced (viewer vs editor vs commenter), and where? What happens when sharing is
      revoked mid-session?
- [ ] A hot doc's collab server dies — trace recovery end to end. Why is no accepted op lost and no SPOF
      global?
