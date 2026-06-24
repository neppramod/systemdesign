# Design 14: Google Drive / Dropbox — File Storage, Sync & Sharing

> **Why this problem is a staff filter:** "file storage" sounds like S3 with a UI — `POST /upload`,
> `GET /download`, a `files` table — and juniors design exactly that, then stall the moment the
> interviewer says "now the same file is open on my laptop and my phone and I edit both offline."
> The whole difficulty is *not* storing bytes; object stores already do that. The difficulty is the
> **sync engine**: detecting local change, computing the minimal delta, shipping it, and resolving the
> case where two devices changed the same file — *without ever silently losing a byte a user typed*.
> The staff signals are that you (a) **split metadata from content** and give them *different*
> consistency models — metadata strongly consistent and serialized, blocks eventually consistent and
> content-addressed; (b) **chunk files into content-addressed blocks** so dedup and delta-sync fall out
> for free; (c) treat sync as a **client-server reconciliation protocol with a versioned namespace**,
> not a glorified upload; (d) make conflict resolution **never lose data** (conflicted copies, not
> last-write-wins); and (e) get **sharing/permissions** right as an authorization model, not an
> afterthought. This walkthrough runs the [Topic 1 framework](../prep/01-framework-and-building-blocks.md)
> end to end and leans on [blob storage](../prep/14-blob-storage-and-media.md),
> [real-time/push](../prep/15-realtime-and-push.md), [security/auth](../prep/17-security-and-auth.md),
> [replication/consistency](../prep/05-replication-and-consistency.md), and
> [search](../prep/11-search-systems.md).

---

## 1. Requirements (5 min) — drive this, don't wait

I'll state the buckets out loud and **scope aggressively** to protect my 45 minutes.

### Functional (the verbs)
- A user **uploads a file** from one device; it appears on **all their other devices** automatically.
- A user **edits a file locally** (offline included); the change **syncs up**, and **down** to every
  other device — shipping *only what changed*, not the whole file.
- A user **organizes files into folders** (a namespace/hierarchy); renames, moves, deletes sync too.
- **Version history** — past versions are recoverable; a delete is recoverable (trash/retention).
- **Sharing** — share a file or a whole folder with specific users (view/edit), or generate a
  **shareable link** (anyone-with-link, optionally view-only / password / expiry).
- **Real-time notification** — when a collaborator changes a shared file, my devices learn quickly,
  not on a 10-minute poll.
- **Search** — find a file by name, and (stretch) by content.

> **Scope out loud:** "I'll build upload/download, the local↔server **sync engine** with delta sync and
> conflict resolution, the **metadata/namespace** service, **sharing + permissions (ACLs/link sharing)**,
> change **notifications**, and version history. I'll treat **real-time collaborative editing of document
> bodies** (Google Docs OT/CRDT character-merge) as *out of scope* — that's a different system (Design:
> collaborative editor); Drive syncs *files as opaque blobs* and resolves whole-file conflicts with
> conflicted copies. I'll cover search at a high level and skip mobile-specific offline UX, billing, and
> admin/compliance unless you want one as the deep dive. Shout if you'd rather redirect."

### Non-functional (where staff candidates separate)
- **Durability is the top promise — 11 nines.** A file a user saved must **never** be lost. This is the
  whole product. Availability matters, but a lost file is unforgivable in a way a 30-second sync delay is not.
- **Consistency — split it deliberately, this is the crux of the whole design:**
  - **Metadata** (namespace, file→block mapping, versions, ACLs) → **strongly consistent, serialized**.
    "What is the current version of file X and who can see it" must be authoritative; this is a
    relational/transactional workload.
  - **Block/content storage** → **eventually consistent, content-addressed, immutable**. Blocks are
    write-once, named by their hash; "eventual" is fine because a block never changes once written.
  - **Read-your-writes for your *own* files** (I save, I immediately see my save on the same device and
    on a refresh), **eventual propagation to collaborators** (your edit reaching my laptop in seconds is
    fine; sub-second isn't required).
- **Bandwidth efficiency is a first-class requirement, not an optimization.** Re-uploading a 2 GB video
  because one byte of metadata changed is unacceptable. **Delta sync (only changed blocks) + dedup** are
  *requirements*, derived below — they're why this isn't just S3.
- **Latency:** sync-detect-to-propagate target **single-digit seconds p99** for small edits between
  online devices. Upload/download bounded by bandwidth, not our servers (direct-to-blob).
- **Scale:** ~1B registered users, ~500M DAU, read:write ≈ heavily read-skewed for downloads but the
  *sync* path is write-amplified by device count.
- **Multi-device + offline-first reality:** a device can be offline for days, then reconnect with a
  divergent local state. The design must assume **divergence is normal**, reconcile it, and never lose work.

> **The single derived insight to state now:** "I'm going to **separate metadata from content** and run
> them on different stores with different consistency models, and I'm going to **chunk every file into
> content-addressed blocks**. Once a file is a *list of block hashes in a versioned metadata record*,
> dedup, delta-sync, and version history all fall out of the same primitive — and sync becomes
> *reconciling two lists of block hashes*, not shipping files. That one decision is the whole design."

---

## 2. Estimation (3 min) — justify the storage and dedup decisions

Numbers exist to *force* architecture, per [Topic 2](../prep/02-estimation-and-napkin-math.md).

**Assume:** ~500M DAU, average user stores ~50 GB, uploads/modifies ~20 files/day averaging ~1 MB each
(most edits are small; the big files are rare but dominate raw bytes).

### Storage (the headline number)

| Quantity | Calc | Result |
|---|---|---|
| Raw stored data | 500M users × 50 GB | **~25 EB** (exabytes) logical |
| New/changed data per day | 500M × 20 files × 1 MB | **~10 PB/day** of write churn |
| **After block-level dedup** | empirically ~30–50% savings (shared installers, forwarded docs, re-saves) | **~12–17 EB physical** |
| Metadata rows | files + versions + blocks × users | **~100s of billions of rows** |

### Throughput

| Quantity | Calc | Result |
|---|---|---|
| Upload/modify QPS (avg) | 500M × 20 ÷ 86,400 | **~115k file-ops/sec** |
| Peak (2–3×) | ×2.5 | **~300k file-ops/sec** |
| Metadata QPS (much higher) | sync polls/notifies + listings + version reads, ~10× file-ops | **~1–3M metadata QPS** |
| Block GET QPS (downloads) | read-skewed, served by CDN | **CDN-absorbed, origin sees a fraction** |

### Dedup + delta-sync savings math (the money calculation)

| Scenario | Naive (whole-file) | With chunking + delta | Savings |
|---|---|---|---|
| Edit 1 byte of a 2 GB file (say 4 MB block size → ~500 blocks) | re-upload 2 GB | re-upload **1 block ≈ 4 MB** | **~99.8%** |
| 1,000 employees receive the same 30 MB onboarding PDF | 30 GB stored | **30 MB stored once** + 1,000 tiny refs | **~99.9%** |
| Re-save a doc with minor changes | full file each save | only changed blocks per version | proportional to edit size |

> **The money sentences:** "**~25 EB logical, ~10 PB/day of churn**, but block-level dedup cuts physical
> storage 30–50% and delta-sync cuts upload bandwidth by **orders of magnitude** for the common case of a
> small edit to a large file. And the **metadata QPS is ~10× the file-op QPS** — sync is metadata-chatty,
> so the metadata service, not the byte store, is where I'll spend my consistency and scaling budget. The
> numbers picked *content-addressed chunking* and a *separate strongly-consistent metadata tier* for me;
> I didn't memorize them."

---

## 3. API design (3 min)

Two surfaces: a **metadata/control plane** (small, chatty, strongly consistent) and a **data plane**
(big bytes, direct-to-blob, bypasses our servers). Keeping them separate is the API-level expression of
the metadata/content split.

```
# ---- Control plane (metadata service) ----
POST /files                 { path, blockHashes[], size, mimeType }   -> { fileId, version }
GET  /files/{id}                                                       -> { metadata, version, blockHashes[] }
PATCH /files/{id}           { newBlockHashes[], baseVersion }          -> { version }   # commit an edit
POST /folders               { path }                                   -> { folderId }
GET  /namespace?cursor=&sinceCursor=                                   -> { changes[], nextCursor }  # the SYNC delta
POST /files/{id}/share      { granteeId|linkScope, role: view|edit }   -> { shareId | link }
GET  /search?q=&cursor=                                                -> { results[], nextCursor }

# ---- Data plane (blocks → object store, see §6) ----
POST /blocks/check          { hashes[] }                  -> { missing: [hashes] }     # dedup probe
POST /blocks/upload-url      { hash, size }                -> { presignedUrl }           # resumable for large
GET  /blocks/{hash}/url                                    -> { signedCdnUrl }           # download via CDN

# ---- Change notification (see §6f) ----
WS   /notifications          (auth in handshake)           -> push { namespaceCursor advanced }
```

- **`GET /namespace?sinceCursor=`** is the heart of sync: "tell me everything that changed in my
  namespace since cursor C." It's **cursor-based, not offset** (offset breaks under constant change),
  per Topic 1 — the cursor is a monotonic per-namespace version/logical-clock.
- **`POST /blocks/check`** is the dedup probe: the client hashes its blocks locally and asks "which of
  these do you *not* already have?" — only the missing ones get uploaded. This is delta-sync at the wire.
- **`PATCH` carries `baseVersion`** → optimistic concurrency: the server rejects if the file moved on,
  which is how we *detect* conflicts (§6d). Load-bearing.
- **Auth, rate-limit, TLS terminate at the gateway**, stated once here. See
  [Topic 9](../prep/09-api-gateway-loadbalancing-ratelimiting.md) and [Topic 17](../prep/17-security-and-auth.md).

---

## 4. Data model (5 min) — access pattern picks the store

The split is the whole story: **two stores, two consistency models.**

### Metadata store — relational, strongly consistent (Postgres/Spanner-class, sharded by user)

The access patterns are: "list a folder," "get the current version + block list of a file," "what
changed since cursor C," "who can access this," "commit an edit transactionally." These want
**transactions, secondary indexes, and a serialized per-namespace version** — that's relational.

**`files`** — `file_id`, `namespace_id` (owner/team root), `parent_folder_id`, `name`, `current_version_id`,
`is_deleted`, `updated_at`. The namespace hierarchy lives here.

**`file_versions`** — `version_id`, `file_id`, **`block_list` (ordered array of block hashes)**, `size`,
`created_by`, `created_at`. **Immutable.** A new edit = a new version row pointing at a new block list
(mostly-shared blocks). This *is* version history — keep N versions / 30 days, then prune.

**`blocks`** (the dedup index) — `block_hash` (PK, content address), `object_store_key`, `size`,
`refcount`. Global or per-shard; refcount drives garbage collection (§6e).

**`namespace_journal`** — append-only **`(namespace_id, cursor, change)`** log. Every metadata mutation
appends a monotonic `cursor`. This is what `GET /namespace?sinceCursor=` reads — the **ordered change
feed per namespace** that makes sync a cheap range-scan, not a full-tree diff. (Same shape as a WAL,
[Topic 1](../prep/01-framework-and-building-blocks.md).)

**`acls`** — `(resource_id, principal_id, role)` + shared-folder inheritance + link tokens (§6e). The
"who can access what" model.

### Content store — object storage, eventually consistent, immutable

**Blocks live in S3/GCS/Azure Blob**, keyed by `block_hash`. Write-once, never mutated, content-addressed
(see [Topic 14](../prep/14-blob-storage-and-media.md)). Eventual consistency is *safe here* precisely
because a block is immutable — there's no stale-read hazard when the value can never change.

> **SQL vs NoSQL, decided from access patterns not reflex (Topic 1):** metadata needs transactions
> (commit a version + journal entry + refcount atomically), a serialized per-namespace cursor, and ACL
> joins → **relational, sharded by `namespace_id`** (Spanner if I want horizontal scale *with* strong
> consistency; Postgres + Vitess otherwise). Blocks are immutable opaque bytes with a hash key and zero
> query needs → **object store**. **The consistency split is the design**: strong for the small metadata,
> eventual for the big bytes. Stating *why* — immutability makes eventual safe — is the staff move.

---

## 5. High-level design (10 min) — happy path end to end

The governing structural decision: **control plane (metadata, strong, our servers) is separate from the
data plane (blocks, direct client↔object-store).** Bytes never flow through our application servers.

```
                         ┌───────────────────────────────────────────────┐
   Client (sync agent)   │   API Gateway / LB  (authn, rate-limit, TLS)   │
   - watches local FS    └───────────────┬───────────────────────────────┘
   - chunks + hashes              control │ plane (metadata, small, STRONG)
   - local metadata DB                    ▼
   ┌──────────────┐         ┌──────────────────────────────┐      ┌──────────────────┐
   │ block cache  │         │   Metadata Service            │◄────►│ Metadata Store    │
   │ + chunker    │         │  (namespace, versions, ACLs,  │      │ (Spanner/Postgres │
   └──────┬───────┘         │   journal, dedup index)       │      │  sharded by ns)   │
          │                 └───────┬──────────────┬────────┘      └──────────────────┘
          │ data plane              │ append       │ enqueue
          │ (bytes, direct)         ▼              ▼ change event
          │                 ┌──────────────┐  ┌──────────────────────────────┐
          ▼                 │ namespace    │  │  Notification Service          │
   ┌──────────────┐         │ journal      │  │  (WS/long-poll fan-out)        │──► other devices
   │ Object Store │         └──────────────┘  └──────────────────────────────┘
   │ (blocks, S3) │◄── presigned PUT / signed GET (via CDN) ──┐
   └──────────────┘                                            └── client (download)
        ▲  block GETs served edge ──► [ CDN ]
```

**Walk one upload + sync through it (device A uploads, device B is online):**

1. **Device A's sync agent** detects a new/changed file, **chunks it into blocks** (~4 MB, content-defined
   — §6b), and **hashes each block** (SHA-256). It now has an ordered `blockHashes[]`.
2. A calls **`POST /blocks/check{hashes}`** → metadata service returns only the **missing** hashes (dedup:
   blocks A already shares with anyone aren't re-sent).
3. A gets **presigned upload URLs** for the missing blocks and **uploads bytes directly to the object
   store** (resumable/multipart for big blocks), bypassing our servers entirely.
4. **Only after all blocks are durably in the object store**, A calls **`PATCH /files/{id}{newBlockHashes,
   baseVersion}`**. The metadata service, in **one transaction**: validates `baseVersion` (conflict check),
   writes a new immutable `file_versions` row, bumps `blocks.refcount`, updates `files.current_version_id`,
   and **appends a `namespace_journal` entry** with a fresh cursor. *Commit metadata only after bytes are
   durable* — the dual-write ordering rule (§6, [Topic 14](../prep/14-blob-storage-and-media.md)).
5. The journal append emits a **change event** → **Notification service** pushes "namespace advanced past
   cursor C" to **device B** (and any collaborators with access).
6. **Device B** calls **`GET /namespace?sinceCursor=`**, sees file X changed, fetches the new version's
   `blockHashes[]`, diffs against the blocks it already has, **downloads only the missing blocks** (from
   the **CDN** via signed URLs), and **reassembles the file locally**. Delta sync, both directions.
7. **Device B offline?** No push reaches it. On reconnect it just calls `GET /namespace?sinceCursor=` from
   its last cursor and catches up — **the journal is the durable source of truth; the push is a best-effort
   tickle** (exactly the [Topic 15](../prep/15-realtime-and-push.md) pattern).

> Keep it this simple first, then evolve under questioning. The arrows carrying the staff signal:
> **"check blocks → upload missing → commit metadata last"** (dedup + delta + dual-write order), and
> **"journal is truth, push is a tickle"** (sync survives any offline gap). Everything in the deep dives
> hangs off those.

---

## 6. Deep dives (15 min) — where the round is won

I'd propose the order: *"The interesting parts are (a) the storage model — chunking, content-addressing,
dedup, delta-sync; and (b) the sync engine — change detection, the reconciliation protocol, and conflict
resolution without data loss. Can I go deep on those, then touch metadata consistency, sharing/authz,
notifications, large-file upload, and search?"*

### 6a. File storage: the metadata/content split and content-addressed blocks

The foundational decision (and the answer to "why isn't this just S3"):

- **Metadata DB ≠ object store.** The DB holds the *map* (namespace, file→version→block-list, ACLs); the
  object store holds the *bytes*. They scale and consist independently: metadata is small, strongly
  consistent, transactional; bytes are huge, eventually consistent, immutable. See
  [Topic 14](../prep/14-blob-storage-and-media.md) — never store blobs in the DB, never put the namespace
  in the object store.
- **Content-addressed storage:** a block's key **is its hash** (`SHA-256(content)`). Two consequences fall
  out for free: (1) **dedup** — identical content has identical key, so it's stored once globally; (2)
  **integrity** — re-hash on read to detect corruption; the name *is* the checksum.
- **Immutability:** blocks are write-once. An "edit" never mutates a block; it writes *new* blocks and a
  *new* version pointing at them. This is why eventual consistency on the byte store is safe — there's no
  such thing as a stale read of an immutable object.

> **The tradeoff stated:** content-addressing buys dedup + integrity + safe eventual consistency, at the
> cost of a hash computation on every block (cheap) and a **garbage-collection problem** — when no version
> references a block anymore, who deletes it? Answer: **refcount or mark-and-sweep GC** (§6e).

### 6b. Chunking + block-level dedup + delta sync (the bandwidth story)

**Chunk every file into blocks**, then operate on blocks, not files:

- **Block size ~4 MB** (Dropbox-like). Small enough that a 1-byte edit re-uploads ~4 MB not 2 GB; large
  enough that metadata (one hash per block) stays small.
- **Fixed-size vs content-defined chunking — name the tradeoff.** *Fixed* (every 4 MB) is simple but
  suffers the **boundary-shift problem**: insert one byte at the front and every subsequent block's
  content shifts, so every hash changes — dedup/delta collapses to zero. *Content-defined chunking* (CDC,
  a rolling hash like Rabin places boundaries on content patterns) keeps boundaries stable under
  insertions, so only the locally-affected blocks change. **I'd pick CDC** for documents that get edited
  in the middle; fixed-size is fine for append-mostly/whole-replace workloads.
- **Delta sync = diff two block-hash lists.** To sync a change, the client computes its new `blockHashes[]`
  and the server tells it which blocks are **missing** (`POST /blocks/check`). Only those upload. On
  download, device B diffs the new list against blocks it already holds and pulls only the gaps. **Sync is
  set-difference over hashes**, never whole-file transfer.
- **Block-level dedup** operates at two scopes: **per-user** (re-saving a file shares unchanged blocks
  across versions) and **cross-user/global** (the 1,000-employee PDF stored once). Global dedup maximizes
  savings but raises a **privacy/security flag I'd name**: cross-user dedup can leak existence of content
  ("does *anyone* have this block?") and complicate per-user encryption — many systems dedup
  *per-user* or per-team to avoid it. I'd call the tradeoff out loud and default to per-namespace dedup with
  global dedup for non-sensitive tiers.

> **This is the §2 dedup math made real:** "edit-one-byte-of-2GB → upload one 4 MB block (~99.8% saved);
> shared PDF → stored once (~99.9% saved)." Chunking + content-addressing is the single primitive behind
> dedup, delta-sync, *and* version history — that economy of mechanism is the staff signal.

### 6c. The sync engine: change detection + the client-server protocol

The sync agent on each device is the real complexity. Three loops:

1. **Detect local change.** Watch the filesystem (inotify/FSEvents/ReadDirectoryChangesW). On an event,
   **re-chunk + re-hash** the changed file and compare to the **local metadata DB** (a small SQLite of
   `path → version, blockHashes[]`). Hash mismatch = real change (ignore touch/no-op saves). This local DB
   is what lets the agent know its own state without calling the server.
2. **Upload loop (local → server).** For a detected change: `blocks/check` → upload missing blocks →
   `PATCH` with `baseVersion`. If `PATCH` is rejected (server moved on), enter conflict resolution (§6d).
3. **Download loop (server → local).** Triggered by a notification *or* a periodic poll: `GET
   /namespace?sinceCursor=` → for each changed file, diff block lists → download missing blocks → reassemble
   → update local DB → advance local cursor.

**The protocol invariants:**
- **The server's `namespace_journal` cursor is the synchronization clock.** Each device tracks "last cursor
  I've applied." Sync = "advance me from my cursor to head." This makes sync **resumable and idempotent** —
  reapplying a change is a no-op because block lists are content-addressed.
- **Multi-device:** every device of a user (and every collaborator) is just another cursor over the *same*
  namespace journal. Add a 4th device → it starts at cursor 0 and replays to head. **One mechanism — the
  per-namespace journal + per-device cursor — serves both multi-device sync and reconnect/resume.** (Same
  shape as the per-device cursor in [Design 08 chat](08-design-whatsapp-chat.md) — cross-reference it.)
- **Read-your-writes locally:** the device updates its *own* local DB on commit, so it sees its own write
  immediately; collaborators see it eventually, when their download loop runs. That's exactly the §1
  consistency split.

### 6d. Conflict detection + resolution — NEVER silently lose data

This is the part juniors fumble and the part interviewers push on hardest.

**Detection — optimistic concurrency via `baseVersion`.** Every `PATCH` carries the version the edit was
based on. The metadata service does a **compare-and-set**: if `files.current_version_id != baseVersion`,
someone else committed in between → **reject with a conflict**. Because the metadata store is strongly
consistent and the version bump is transactional (§6e), this detection is **exact** — no lost-update race.

**Resolution — the cardinal rule: never silently overwrite a user's work.**
- **Non-conflicting concurrent edits to *different* files** → both just apply; no conflict.
- **Concurrent edits to the *same* file** (the hard case) → we **keep both**. The losing edit becomes a
  **conflicted copy**: `report.docx` and `report (Alice's conflicted copy 2026-06-20).docx`, both synced
  to everyone. The user resolves manually. **This is the Dropbox behavior and the correct default** —
  data preservation over silent merge.
- **Why not last-write-wins?** LWW *loses data* — the slower writer's edit vanishes. Acceptable for a
  presence dot, unacceptable for a file someone spent an hour on. State this tradeoff explicitly.
- **Why not auto-merge (CRDT/OT)?** Because Drive treats files as **opaque blobs** — it can't semantically
  merge a binary `.psd` or `.xlsx`. Merge is the *collaborative-editor* problem (OT/CRDT,
  [Topic 1 conflict resolution](../prep/01-framework-and-building-blocks.md)) and out of scope; for opaque
  files, **conflicted copies are the honest answer**.
- **Folder/move conflicts** (A renames folder, B adds a file inside it) → reconcile at the metadata layer;
  the journal's serialized cursor gives a total order to *apply* operations, and structural conflicts
  (delete-a-folder-someone-added-to) resurface the orphaned items rather than dropping them.

> **The staff sentence:** "Detection is exact because metadata is strongly consistent and versioned;
> resolution **never loses a byte** — concurrent same-file edits produce a conflicted copy, not a silent
> overwrite. I reject LWW for files because it discards work, and I reject auto-merge because Drive's files
> are opaque blobs — semantic merge is the collaborative-editor problem, not this one."

### 6e. Metadata service: namespace, versions, consistency, GC, and sharing/authz

**Namespace + versions.** The `files`/`folders` tree + immutable `file_versions` give free history.
A commit is a **single metadata transaction**: insert version, update `current_version_id`, increment block
`refcount`s, append the journal entry — all atomic so a reader never sees a half-applied edit. **This is
where the strong consistency budget goes** (§4): the metadata, not the bytes.

**Sharding.** Shard the metadata store **by `namespace_id`** (a user's or team's root) so one user's
entire tree, journal, and ACLs are co-located → folder listings and namespace-delta scans are single-shard.
Cross-namespace operations (sharing across users) are the rare cross-shard case (handled below). Spanner
gives this *with* strong consistency and horizontal scale; Postgres+Vitess otherwise.
([Topic 4](../prep/04-sharding-and-partitioning.md).)

**Garbage collection.** Blocks are dedup'd and versioned, so a block dies only when **no version anywhere**
references it. **`refcount`** on commit/version-prune, or periodic **mark-and-sweep** as a safer backstop
(refcounts drift under failures). GC is *deferred and conservative* — deleting a still-referenced block
would corrupt files, so we err toward keeping garbage over losing data.

**Sharing & permissions — the "who can access what" model (tie to [Topic 17](../prep/17-security-and-auth.md)).**
- **ACLs as `(resource, principal, role)`.** Roles: `viewer`, `commenter`, `editor`, `owner`.
- **Shared folders + inheritance → this is ReBAC, not RBAC.** Permissions are *relationships* on a
  hierarchy: "Alice is editor on folder F ⇒ editor on everything under F." Checking access to a deep file
  means walking up to the nearest grant. This is exactly the **Google Zanzibar** model
  ([Topic 17](../prep/17-security-and-auth.md)): permissions as a graph of relation tuples
  (`doc:123#viewer@user:alice`, `folder:F#editor@group:eng`), with **inheritance edges** so a folder grant
  flows down. I'd name Zanzibar/ReBAC explicitly — it's the staff reference for "who can access what" at scale.
- **Authorization check on every metadata op:** `check(user, action, resource)` resolved against the
  relation graph, **cached** (per Zanzibar, with a consistency token so a just-changed ACL isn't read
  stale — you don't want to keep serving a file to someone you just unshared from).
- **Link sharing** = a capability token: a signed, optionally-expiring, optionally-scoped (`view`/`edit`,
  password) URL that grants access **without** a per-user grant. Revocable by invalidating the token.
  Different trust model (bearer capability vs identity ACL) — state the distinction.
- **Cross-user/cross-shard sharing:** when Bob shares with Alice (different namespaces), the shared subtree
  appears in Alice's namespace as a **mount/pointer**, and changes fan out to *both* journals. This is the
  rare cross-shard write; keep it transactional via the share record being the single source of the grant.

### 6f. Notifying other clients of changes (tie to realtime/push)

Pushing change to other devices is the [Topic 15](../prep/15-realtime-and-push.md) problem in miniature.

- **Each online device holds a long-poll or WebSocket** to the **Notification service**. When a namespace's
  journal advances, the service pushes a **lightweight tickle** — just "namespace N advanced, go sync" — not
  the change payload. The device then pulls via `GET /namespace?sinceCursor=`. **Thin notification + pull**
  keeps the push path cheap and the journal the single source of truth.
- **Long-poll vs WebSocket:** long-poll is simpler, firewall-friendly, and fine because change frequency is
  low (you don't edit a file 100×/sec); WebSocket if we want lower latency / bidirectional. Drive historically
  used **long-poll**; I'd default there and reserve WS for high-activity shared folders.
- **Fan-out scope = everyone with access to the namespace/resource** — resolved via the same ACL/ReBAC graph
  (§6e). A shared-folder edit notifies all collaborators' devices.
- **Push is best-effort; the journal is durable.** Miss a tickle (device asleep, push dropped)? On reconnect
  the device's periodic poll catches it up from its cursor. **No change is ever lost by a lost
  notification** — same dumb-tickle/durable-truth pattern as chat's inbox (Design 08).

### 6g. Upload/download: presigned, resumable, large files (tie to blob doc)

Per [Topic 14](../prep/14-blob-storage-and-media.md), bytes go **direct client↔object-store**, never
through our app servers:

- **Upload:** client gets a **presigned PUT URL** per missing block and uploads directly. For **large files
  / large blocks**, use **multipart + resumable** upload so a dropped connection resumes from the last
  completed part instead of restarting a 2 GB transfer. (Chunking already makes most uploads naturally
  resumable at block granularity.)
- **Download:** `GET /blocks/{hash}/url` returns a **signed CDN URL**; the **CDN** serves blocks at the edge
  (Topic 14 Part D), so origin/object-store sees a fraction of read traffic and downloads are fast globally.
  Signed URLs enforce access (don't let a leaked block hash = public read).
- **The dual-write ordering rule (Topic 14 Part H):** **upload blocks to the object store *first*, commit
  the metadata version *last*.** Never the reverse — a metadata record pointing at blocks that don't exist
  yet would let a collaborator try to download a missing block. Blocks-first, metadata-last, always.

### 6h. Search over files (tie to search doc, high level)

[Topic 11](../prep/11-search-systems.md), kept high-level since it's a stretch goal:

- **Filename/path search** is cheap — index `name`/`path` in the metadata store (or a small inverted index)
  scoped by what the user can access.
- **Full-content search** = a separate **inverted index (Elasticsearch)** fed **asynchronously via CDC/queue**
  off the journal: on a new version, an extraction worker pulls the blocks, extracts text (PDF/Docx/OCR),
  and indexes it. **Eventually consistent** — search lagging the latest save by seconds is fine.
- **Permission-filtered results are mandatory:** never return a hit on a file the searcher can't access.
  Either filter post-retrieval against the ACL graph (§6e) or index ACL terms alongside content. This is the
  classic search-meets-authz join — name it.

---

## 7. Wrap-up (3 min) — bottlenecks, failure modes, SPOFs

- **Sync conflict (the headline correctness failure).** Two devices edit the same file. Detected exactly by
  the strongly-consistent `baseVersion` compare-and-set; resolved by a **conflicted copy**, never a silent
  overwrite. **No data is lost** — that's the whole promise. LWW and auto-merge both rejected, with reasons.
- **Partial upload / crash mid-sync.** Because **blocks upload first and metadata commits last**, a crash
  before the `PATCH` leaves **orphan blocks** (harmless, GC'd) but **no corrupt metadata** — the file simply
  reflects its previous version. A crash after `PATCH` is fully committed and durable. There is no window
  where a file points at bytes that don't exist. Resumable upload means a dropped transfer resumes, not restarts.
- **Notification lost / device offline for a week.** No loss — the **journal is durable**; the device pulls
  from its cursor on reconnect and catches up. The push is a best-effort tickle by design (Topic 15).
- **Remaining bottlenecks:** (1) **metadata QPS** (~10× file-ops) — handled by sharding per namespace +
  caching the namespace-delta and ACL checks; (2) **GC correctness** under refcount drift — mark-and-sweep
  backstop, conservative deletion; (3) **hot shared folder** (a 10k-person team folder) — fan-out of
  notifications + ACL checks, mitigated by the thin-tickle/pull model and ACL caching; (4) **global dedup
  privacy** — defaulted to per-namespace dedup for sensitive tiers.
- **SPOFs — none global by design.** Object store is multi-region replicated (11 nines durability);
  metadata is sharded + replicated (Spanner/quorum, [Topic 5](../prep/05-replication-and-consistency.md));
  notification service is a stateless fleet with the journal as durable backstop; the namespace cursor is
  *per-namespace*, not a central sequencer. Closest hot spot is a giant shared folder's journal, mitigated
  by per-folder sub-journals.
- **With more time:** end-to-end encryption (client-side, which kills cross-user dedup + server search —
  the same constraint-engine as chat's E2EE); smart sync / on-demand hydration (placeholder files,
  download-on-open); LAN sync (peer-to-peer block transfer between same-network devices); cold-tiering old
  versions to cheaper storage; backpressure + rate-limiting on the sync API
  ([Topic 13](../prep/13-resilience-and-failure-handling.md)).

---

## What made this staff-level

- **Split metadata from content and gave them different consistency models** — strong/serialized/transactional
  metadata vs eventual/immutable/content-addressed bytes — and explained *why* immutability makes eventual
  safe. That split is the design; "S3 with a UI" is the junior answer.
- **Made content-addressed chunking the single primitive** behind dedup, delta-sync, *and* version history,
  and did the bandwidth/dedup math (edit-1-byte-of-2GB → 4 MB; shared PDF → stored once). Named the
  fixed-vs-content-defined-chunking boundary-shift tradeoff and picked CDC.
- **Treated sync as a versioned reconciliation protocol** — per-namespace journal as the synchronization
  clock, per-device cursor for multi-device + resume in one mechanism, set-difference-over-hashes as the
  wire protocol — rather than a glorified upload.
- **Got conflict resolution right: never silently lose data.** Exact detection via strongly-consistent
  `baseVersion` CAS; conflicted copies as resolution; explicitly rejected LWW (loses work) and auto-merge
  (files are opaque — that's the collaborative-editor problem) with reasons.
- **Modeled sharing as ReBAC/Zanzibar**, not bolted-on RBAC — relation tuples with folder inheritance, ACL
  checks cached with a consistency token, and link-sharing as a separate capability/bearer model.
- **Made the push a best-effort tickle over a durable journal** (thin-notification + pull), so no change is
  lost to a dropped notification or a week-long offline gap — the same durable-truth/fast-path inversion as chat.
- **Named the dual-write ordering rule (blocks first, metadata last)** and the partial-upload/sync-conflict
  failure modes before being asked, and showed there's no global SPOF by construction.

---

### Self-check before the mock (answer these from memory)
- [ ] State the governing insight: why split metadata from content, and why is eventual consistency *safe*
      on the block store but not the metadata store?
- [ ] What does content-addressed chunking buy you (name all three), and what new problem does it create?
- [ ] Walk the upload happy path: blocks/check → upload missing → commit metadata. Why is the order load-bearing?
- [ ] Fixed-size vs content-defined chunking — what's the boundary-shift problem and which do you pick?
- [ ] Do the dedup/delta math: edit 1 byte of a 2 GB file; 1,000 people get the same PDF.
- [ ] How does the namespace journal + per-device cursor serve *both* multi-device sync and reconnect-resume?
- [ ] Conflict detection: how does `baseVersion` CAS make detection exact? Why is it exact only because
      metadata is strongly consistent?
- [ ] Conflict resolution: why conflicted copies, not LWW, not auto-merge? Give the reason to reject each.
- [ ] Sharing: why is shared-folder permission ReBAC (Zanzibar), not RBAC? How does inheritance work, and
      how is link-sharing a different trust model?
- [ ] Why is the change notification a thin tickle over a durable journal, and why does a lost push lose nothing?
- [ ] Presigned/resumable upload + the dual-write rule: which goes first, blocks or metadata, and why?
- [ ] Garbage collection: when does a block die, and why is GC conservative (refcount vs mark-and-sweep)?
- [ ] Why is metadata QPS ~10× file-op QPS, and where does that send your scaling/caching budget?
- [ ] A device is offline for a week, then reconnects — trace catch-up. Why is no change lost and no SPOF global?
