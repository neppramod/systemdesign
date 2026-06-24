# Design 26: Distributed File System / Object Store Backend (GFS / HDFS / S3-internals)

> **Why this is a foundational design:** almost every "storage" system you'll be asked about — a blob
> store, a data lake, a database's underlying disk layer, even the backing store for [Drive/Dropbox
> (Design 14)](14-design-google-drive-dropbox.md) — is *this* design underneath. The problem is the
> purest statement of distributed storage: **store more bytes than any one machine can hold, on disks
> that are constantly dying, and never lose data.** Get this right and you've internalized the
> primitives (chunking, replication vs erasure coding, control-plane/data-plane split, a metadata
> master, heartbeat-driven re-replication, checksums/scrubbing) that every later storage question
> recombines.
>
> The one-line thesis I'll state up front and keep returning to: **separate the control plane
> (metadata — small, in-memory, consistency-critical) from the data plane (bulk bytes — enormous,
> streamed directly, throughput-critical), and treat hardware failure as the steady state, not an
> exception.** Every decision falls out of those two ideas. I'll run the full [7-step
> framework](../prep/01-framework-and-building-blocks.md) but spend most of my budget in **Step 6**,
> leaning hard on [replication & consistency](../prep/05-replication-and-consistency.md),
> [blob storage](../prep/14-blob-storage-and-media.md), [consensus &
> coordination](../prep/08-consensus-and-coordination.md), and [batch/stream
> processing](../prep/18-batch-and-stream-processing.md).

---

## Step 1 — Requirements (5 min) — I drive this

"Design a distributed file system" is unbounded, so I scope hard and pin the non-functional
requirements that actually decide the architecture.

**Functional — deliberately tiny**
- `create(path)` / `open(path)` — a namespace of named files (GFS/HDFS) or named objects in buckets (S3).
- `read(file, offset, length)` — read a byte range. Reads are dominated by **large sequential** scans.
- `write` / `append(file, data)` — write bytes. The headline write pattern is **append** (logs, MapReduce output, ingested datasets), not random in-place overwrite.
- `delete(path)`.
- `stat / list` — metadata about a file (size, chunk locations, mtime).

> **Explicit non-goals I'll call out:** I am **not** building a POSIX filesystem with cheap small random
> writes, hard links, or low-latency tiny-file access. GFS/HDFS explicitly optimize for **a modest
> number of large files (multi-GB), append-heavy, read-sequentially**. If the interviewer wants
> millions of tiny files or random in-place mutation, that's a *different* design (a small-file store /
> a B-tree-backed FS) and I'd say so. Scoping this out is itself a seniority signal — it's the
> assumption GFS was explicitly built around.

**Non-functional — where the design is actually decided:**
- **Massive scale** — petabytes today, exabytes as a fleet. Thousands of machines, tens of thousands of disks. The data does not fit on one node, one rack, or one row of racks.
- **High durability — no data loss is the cardinal requirement.** A stored byte must survive disk failures, machine failures, rack power loss, and bit rot. Target **11 nines** of durability (S3's stated 99.999999999%) — i.e. expected annual loss of ~1 object per 10 billion.
- **High sequential throughput** over low latency. We optimize aggregate read/write *bandwidth* (GB/s across the fleet feeding a MapReduce/Spark job), not single-op p99. A read can take tens of ms; that's fine.
- **Tolerate constant hardware failure** — at thousands of machines, **something is always broken.** With 10,000 disks at a 2-4%/yr AFR, you lose **several disks per week**, plus daily machine reboots, kernel panics, and network blips. Failure is the *common case*; the system must self-heal without operator intervention.
- **Horizontal scale on commodity hardware** — cheap disks, no RAID controllers, no SAN. Reliability comes from **software replication across machines**, not from expensive reliable hardware. Adding capacity = racking more cheap boxes.
- **Concurrent appenders** — many clients appending to the same file (e.g. many mappers writing one output) must work without a distributed lock per write. (This drives the relaxed consistency model.)

> **The staff move in Step 1:** I'm *positioning* before drawing. "Big files, append-heavy,
> sequential, commodity hardware, failure-is-normal, durability above all" is the thesis that makes
> every later choice — large chunks, replication, a metadata master, heartbeats + re-replication,
> relaxed append consistency — *derivable* instead of memorized. Per PACELC this is **PA/EC-ish**:
> the *data plane* is availability-leaning and eventually consistent for appends, but the *metadata
> plane* is strongly consistent (you cannot have two masters disagree about where a chunk lives).

---

## Step 2 — Estimations (3 min) — to justify, not impress

Pick numbers that *force* the architecture.

| Quantity | Assumption | Result |
|---|---|---|
| Logical data stored | — | **10 PB** logical |
| Replication factor | 3× (hot data) | **30 PB** physical |
| Disk per machine | 12 × 4 TB = 48 TB | ~48 TB raw / node |
| **Chunkservers needed** | 30 PB ÷ 48 TB | **~640 nodes** (call it ~1,000 with headroom + skew) |
| Chunk/block size | 64 MB (GFS) → 128 MB (HDFS) | see below |
| **Number of chunks** | 10 PB ÷ 128 MB | **~80 million** logical chunks (×3 = 240M replicas) |
| Metadata per chunk | ~64 bytes (id, version, refcount) at the master | 80M × 64 B ≈ **~5 GB** master RAM for chunk index |
| Metadata per file | ~100s of bytes (name, chunk list) | tens of millions of files → **a few GB more** |

**Throughput sizing:**
- A single disk streams ~150-200 MB/s sequential. 12 disks/node ≈ ~2 GB/s/node aggregate.
- 1,000 nodes → **~2 TB/s aggregate sequential bandwidth** if perfectly balanced — which is exactly the point: a 1,000-mapper job reads from 1,000 different chunkservers in parallel, **not** through any central box.

> **What the math *justifies* — three load-bearing conclusions:**
> 1. **The whole metadata index fits in RAM on one machine (~single-digit GB).** This is the single
>    most important estimate: it's *why* a centralized in-memory metadata master is viable at all, and
>    why metadata is the eventual scaling ceiling.
> 2. **~80M chunks, not ~80 billion** — because chunks are huge. Had we used 4 KB blocks like a local FS,
>    we'd have 2.5 *trillion* chunks and ~150 TB of metadata — un-RAM-able. **Large chunks exist to keep
>    metadata small.** (Expanded in 6A.)
> 3. **Aggregate bandwidth scales with node count *only if* clients stream data straight from
>    chunkservers** and never funnel bytes through the master. That forces the control/data-plane split
>    (6B). The estimate made the architecture necessary, not assumed.

---

## Step 3 — API design (3 min)

Two distinct surfaces — and the split *is* the architecture, so I draw it explicitly.

```
# ---- Control plane: client <-> MASTER (metadata only, no file bytes) ----
open(path)                         -> fileHandle
getChunkLocations(file, chunkIdx)  -> { chunkHandle, version, [replica endpoints] }   # cached by client
allocateChunk(file)                -> { chunkHandle, primary replica, secondary replicas, lease }
list(path) / stat(path) / delete(path)

# ---- Data plane: client <-> CHUNKSERVERS (bulk bytes, never through master) ----
readChunk(chunkHandle, offset, length)        -> bytes        # client picks nearest replica
writeChunk(chunkHandle, data) / append(...)   -> offset/ack   # pipelined to all replicas
```

**S3 object-store framing of the same thing** (I'd mention both vocabularies):
```
PUT  /bucket/key        (body = object bytes)        -> ETag, version-id
GET  /bucket/key        [Range: bytes=...]           -> bytes
DELETE /bucket/key
# multipart for large objects:
POST   /bucket/key?uploads                 -> uploadId
PUT    /bucket/key?partNumber=N&uploadId=  -> ETag(part)
POST   /bucket/key?uploadId=  (manifest)   -> commit object
```

Three contract decisions I'll flag now and justify later:
1. **The master returns *locations*, then gets out of the way.** The heavy bytes flow client↔chunkserver. The master is a *lookup service*, not a data pipe (6B).
2. **Clients cache chunk→location maps** so they don't hit the master on every read (offloads the master; 6E).
3. **Append is a first-class op with relaxed semantics** (`record append` returns the offset GFS chose, not one the client demanded) — this is the consistency-model decision (6G), surfaced early.

> Auth, TLS, signed URLs, and bucket policies terminate at an S3-style gateway / the client library —
> see the [security/auth doc](../prep/17-security-and-auth.md). I won't re-explain them; the
> interesting surface is the storage internals.

---

## Step 4 — Data model (5 min)

Two stores with **deliberately different consistency models** — this is the heart of the design.

**Control-plane state (the master, strongly consistent, in RAM + on a log):**
- **Namespace**: `path -> file metadata`. In GFS this is a flat lookup table with full pathnames (no per-directory inode tree, so no per-directory locks — cheaper); in HDFS it's a real inode tree. S3 is a flat `bucket/key` keyspace.
- **File → chunk list**: `fileId -> [chunkHandle_0, chunkHandle_1, ...]` in order. This is the *durable* mapping (persisted in the operation log).
- **Chunk → locations**: `chunkHandle -> [chunkserver A, B, C], version`. **This is NOT persisted** — the master *rebuilds it from chunkserver heartbeats on startup* (6D). The master doesn't pretend to know where replicas are; the chunkservers are the ground truth and *report in*.
- **Per-chunk**: version number (to detect stale replicas), reference count, lease holder.

**Data-plane state (the chunkservers, the actual bytes):**
- Each **chunk** = a fixed-size (64-128 MB) Linux file on the local FS, named by an opaque global **chunkHandle** (64-bit). Stored on plain disks under the local ext4/xfs — chunks are just files; the OS page cache caches hot ones for free.
- Each chunk carries a **per-block checksum** (e.g. 32 KB sub-blocks, CRC32C each) stored alongside it — for bit-rot detection (6F).

> **Why the two-consistency split is the whole game (tie to [Design 14](14-design-google-drive-dropbox.md)
> and the [data-modeling doc](../prep/20-data-modeling-and-access-patterns.md)):** metadata is small,
> mutated rarely-ish, and *must* be correct — two clients can't disagree about which chunks make up a
> file → **strongly consistent, single authority, replicated log.** Bytes are enormous, written by
> many appenders, and tolerate "eventually all replicas converge" → **relaxed, replicated, streamed.**
> Picking *different* consistency for the two planes is exactly the move that makes the system both
> correct and scalable. A mid-level answer applies one consistency model to everything.

---

## Step 5 — High-level design (10 min) — happy path end to end

Boxes and arrows, kept simple first.

```
                         ┌──────────────────────────────────────────────┐
                         │                 MASTER                        │
   (1) open / locate     │  namespace + file→chunks (in RAM, op-log on   │
  ┌────────────────────▶ │  disk + checkpoints)                          │
  │   chunkHandle +       │  chunk→locations (rebuilt from heartbeats)    │
  │   replica list        │  + standby/shadow masters (Raft/ZK)           │
  │                       └───────────────────────────────────────────────┘
  │                            ▲   heartbeats + chunk reports (every few s)
  │                            │   (also: re-replication & rebalance commands)
CLIENT                         │
  │             ┌──────────────┼───────────────┬───────────────┐
  │  (2) read/   │              │               │               │
  │  write bytes ▼              ▼               ▼               ▼
  │      ┌────────────┐  ┌────────────┐  ┌────────────┐  ┌────────────┐
  └────▶ │ ChunkSrv A │  │ ChunkSrv B │  │ ChunkSrv C │  │ ... ~1000  │
         │ chunks+CRC │  │ chunks+CRC │  │ chunks+CRC │  │            │
         └────────────┘  └────────────┘  └────────────┘  └────────────┘
              (data plane: bytes stream client↔chunkserver, pipelined replica→replica)
```

**Architectural properties of the happy path:**
- **One logical master** for the control plane (made HA with a replicated log + standby — 6C), **many chunkservers** for the data plane.
- **Client talks to the master once per chunk** (and caches the answer), then **streams bytes directly to/from chunkservers.** The master never touches file data — this is the offload that lets ~1,000 chunkservers deliver TB/s while one master handles only metadata lookups.
- **Chunkservers are the source of truth for "what's on disk."** They report their chunks to the master via periodic heartbeats; the master never persists locations, it *learns* them.

**Walking one read out loud:** client → master `getChunkLocations(file, idx)` → master returns `{chunkHandle, [A,B,C], version}` → client **caches** it → client picks the **nearest/least-loaded replica** (say B) → `readChunk(handle, off, len)` straight from B's disk (served from B's page cache if hot). Subsequent reads of nearby chunks skip the master entirely (cache).

**Walking one append out loud (preview of 6E):** client asks master to allocate/locate the chunk → master designates one replica as **primary** and grants it a **lease**, returns all three replica endpoints → client **pushes data to all three replicas in a pipeline** (decoupled from control), then sends the commit to the **primary**, which **picks the offset and order**, applies, and tells secondaries to apply at the same offset → primary acks the client. The master was involved only to hand out the lease.

---

## Step 6 — Deep dives (the bulk of the round)

Proposed order, in dependency sequence: **(A) why huge chunks → (B) control/data-plane split → (C) the
master: in-RAM, op-log, HA → (D) durability: replication vs erasure coding + the math → (E) the write/
append path (pipelining + primary lease) → (F) failure handling: heartbeats, re-replication, checksums/
scrubbing → (G) the consistency model (GFS record-append vs strongly-consistent S3) → (H) rebalancing &
hot spots → (I) scaling the metadata plane (the real ceiling) → (J) how this underpins S3 and HDFS.**

---

### 6A. Why fixed, *huge* chunks (64-128 MB)?

Files are split into fixed-size chunks; the chunk is the unit of placement, replication, and recovery. The non-obvious decision is the *size* — orders of magnitude larger than a local-FS block (4 KB).

**Three reasons large chunks win, all derived from Step 2:**
1. **Metadata stays in RAM.** Master metadata is **per-chunk**, not per-byte. 128 MB chunks → ~80M chunks → ~5 GB of chunk metadata (fits in RAM). 4 KB blocks → trillions of entries → impossible to centralize. **Large chunks exist primarily to shrink the metadata index** so one master can hold it all. This is the single most important "why."
2. **Throughput / fewer round-trips.** A big sequential read touches one chunk for 128 MB, so the client makes **one** master lookup and **one** chunkserver connection per 128 MB, then streams. Small chunks → constant master chatter and TCP setup. Large chunks amortize per-op overhead and keep a persistent connection hot — exactly right for sequential scans.
3. **Reduced master load via client caching.** One cached location entry covers 128 MB of reads, so a client doing a multi-GB scan hits the master a handful of times total.

**The cost (named honestly):**
- **Small files become hot spots.** A 1 MB file is one chunk on 3 chunkservers; if it's popular (e.g. an executable many machines pull at once), those 3 nodes get hammered. GFS's fix was to bump the *replication factor* for such files and stagger client start times. (S3 sidesteps this by treating objects independently and load-balancing at the front end.)
- **Internal fragmentation** — a 1 MB file in a 128 MB chunk wastes space *only if* chunks are pre-allocated; GFS uses **lazy space allocation** (the chunk file grows as written), so a small file's chunk is small on disk. Problem dodged.

> **Tradeoff stated:** huge chunks trade fine-grained access + even small-file load for tiny metadata,
> high sequential throughput, and low master load. For an **append-heavy, large-file, sequential**
> workload (our Step 1 thesis) that's the obvious win; for a tiny-random-file workload it's wrong —
> which is exactly why GFS scoped that out. HDFS bumped the default to 128 MB (then 256 MB) as disks
> and bandwidth grew — same reasoning, bigger constant.

---

### 6B. Control plane vs data plane — the master must never touch bytes

**The problem:** if every read and write flowed *through* the master, the master's NIC (say 25 Gbps ≈ 3 GB/s) would cap the *entire fleet* at 3 GB/s — throwing away the ~2 TB/s the disks can deliver. The master would be the bottleneck and the SPOF for data, not just metadata.

**The split:**
- The master serves **only metadata**: namespace ops and "where does chunk X live?" These are small messages (bytes, not megabytes) and the master can do tens of thousands/sec.
- **Bulk bytes never pass through the master.** The client gets the replica list and talks **straight to chunkservers**. The data plane scales with the number of chunkservers; the control plane scales with metadata op rate (independently).

**Why this is the canonical pattern:** it's the same control/data-plane separation you see in [video streaming (Design 03)](03-design-youtube-netflix.md) (manifest server vs CDN edge), in SDN, and in S3 (the request router/index layer vs the storage nodes). Separating the planes lets each scale on its own axis and keeps the consistency-critical part small.

> **Tradeoff stated:** the cost is **two round-trips** for the first access (master lookup, then
> chunkserver), and the client must handle a **stale cache** (a location it cached may have moved —
> handled by versioning + retry, 6E/6F). We accept it because the alternative — funneling exabytes
> through one box — is a non-starter. The lookup cost is amortized away by client-side caching of
> locations (6E).

---

### 6C. The metadata master: in-RAM, operation log, and the SPOF problem

The master keeps the namespace, file→chunk mapping, and chunk→location map **in memory** (Step 2 showed it fits). In-RAM = microsecond lookups, tens of thousands of metadata ops/sec from one machine. But two questions follow immediately: **how is it durable?** and **what happens when it dies?**

**Durability of metadata — operation log + checkpoints:**
- Every metadata *mutation* (create file, allocate chunk, delete) is first **appended to an operation log** (a WAL — [framework toolkit: WAL/commit log](../prep/01-framework-and-building-blocks.md)) on the master's disk **and replicated to remote standbys**, *before* the op is acked. This is the durable record of truth.
- Replaying the whole log from genesis would be slow, so the master periodically writes a **checkpoint** (a compact, mmap-able snapshot of the in-memory state). Recovery = load latest checkpoint + replay only the log tail since it. Recovery is seconds-to-minutes, not hours.
- **What's NOT logged: chunk locations.** Those are rebuilt from chunkserver heartbeats at startup (6D). Logging them would be a lie — chunkservers are the ground truth, and a logged location could be stale after a crash. So the master persists the *namespace and file→chunk mapping*, and *learns* the *chunk→location mapping*.

**The SPOF problem and HA (tie to [consensus doc](../prep/08-consensus-and-coordination.md)):**
- A single master is a **single point of failure** for the control plane (the data plane keeps serving reads from cached locations even if the master is briefly down, which softens the blow — a nice property).
- **HA via replicated log + standby:** the operation log is synchronously replicated to **shadow/standby masters**. On master failure, a standby with the up-to-date log is promoted. **Leader election + which-replica-is-master** is exactly a job for **Raft/Paxos or an external ZooKeeper/etcd lock** ([consensus doc, Part C/H](../prep/08-consensus-and-coordination.md)) — you must avoid **split-brain** (two masters both editing the namespace). GFS used a primary master + replicas + an external lock service (Chubby) for election; HDFS uses an Active/Standby NameNode with a **shared edit log (QJM — a quorum of JournalNodes, itself a Paxos-like majority)** plus ZooKeeper for failover.
- **Shadow masters** (read-only, slightly stale) can serve **stale-OK metadata reads** even during failover, improving read availability — explicitly relaxed-consistency reads.

> **Staff nuance — why metadata is the scaling ceiling, and how modern systems break it:** one master
> in RAM is elegant but bounds the system to **how much metadata fits in one machine** and **how many
> metadata ops one machine can serve.** GFS hit this; the successor **Colossus** replaced the single
> master with a **distributed, sharded metadata service backed by a Bigtable/Spanner-style store**, so
> the namespace itself is partitioned across many servers. HDFS added **Federation** (multiple
> independent NameNodes, each owning a namespace sub-tree) and **Router-based Federation** on top. S3's
> index is a **horizontally sharded, replicated key→location database**, not one box. The lesson:
> centralized in-RAM metadata is the *right first design* and the *first thing to shard* when you scale
> — I'd state it as the known endgame, not pretend the single master scales forever.

---

### 6D. Durability: replication (3×) vs erasure coding (Reed-Solomon) — and the math

Durability is the cardinal requirement, and it's purely a function of **how many independent copies/fragments** survive failure. Two mechanisms, and **knowing when to use each is the staff signal.**

**Replication (N copies, classic GFS/HDFS default N=3):**
- Store **3 full copies** on 3 chunkservers, deliberately placed in **different failure domains**: not on the same machine, not on the same rack (so a rack power loss / top-of-rack switch failure can't take all 3), and across AZs for higher tiers.
- **Storage overhead: 200% (3× the data for 1× logical).** Expensive, but dead simple: any one surviving copy can serve a read at full speed, and recovery = copy one chunk from a survivor.
- **Why 3, not 2:** with 2 copies, while you re-replicate after one failure you're one failure from data loss — and at fleet scale a second failure during the recovery window is likely. 3 gives a comfortable margin and tolerates a full rack loss plus a disk.

**Erasure coding (Reed-Solomon, what cold/large stores actually use):**
- Split data into **k data fragments**, compute **m parity fragments**, store all **n = k+m** on distinct nodes. Any **k of the n** fragments reconstruct the original (Reed-Solomon over a Galois field). A `(k=6, m=3)` code (HDFS-EC default, "RS-6-3") survives **any 3 simultaneous losses**.
- **Storage overhead: `n/k`.** RS-6-3 → 9/6 = **1.5× (50% overhead)** while tolerating 3 failures — vs replication needing **4×** to tolerate 3 failures. RS-10-4 → 1.4× and tolerates 4 losses. **This is the headline: erasure coding gives the same-or-better durability at a third of the storage cost.**

**The overhead-vs-durability table (the math I'd put on the board):**

| Scheme | Storage overhead | Failures tolerated | Read cost | Recovery cost (rebuild 1 lost piece) | Best for |
|---|---|---|---|---|---|
| 3× replication | **3.0× (200%)** | any 2 | cheapest: read 1 copy | cheap: copy 1 chunk from 1 peer | **hot data**, low-latency reads, small files |
| RS-6-3 | **1.5× (50%)** | any 3 | read k=6 fragments to reconstruct (or 1 if you keep a systematic copy) | **expensive: read k=6 fragments + recompute** | **cold/warm data**, large objects, archival |
| RS-10-4 | **1.4× (40%)** | any 4 | read 10 fragments | very expensive rebuild | bulk archival |

**The tradeoff that decides which to use:**
- **Replication is cheap to *read and recover*, expensive to *store*.** A lost replica is restored by a single chunk-to-chunk copy; a read hits one local copy. Great when data is **hot** (read often, latency matters) or **small/many** (recovery copies are tiny).
- **Erasure coding is cheap to *store*, expensive to *read-degraded and recover*.** Reconstructing a lost fragment requires **reading k surviving fragments across the network and recomputing** — a "rebuild storm" that's k× the I/O of a replication repair, and a *degraded read* (when a fragment is missing) pays the same. Great when data is **cold** (rarely read, recovery is rare) and **large** (the 50% storage saving on a petabyte is enormous).

> **The decision rule I'd state:** **hot/small/latency-sensitive → replicate; cold/large/cost-sensitive
> → erasure-code.** Real systems do *both, tiered*: new objects land **3× replicated** for fast writes
> and reads, then a background job **transcodes aged/cold data to RS** to reclaim ~50% of the space. This
> is exactly the [blob-storage doc](../prep/14-blob-storage-and-media.md)'s durability section — S3,
> Azure (Local Reconstruction Codes / LRC), Facebook f4, and HDFS-EC all do replicate-hot /
> erasure-code-cold. LRC is the refinement worth a sentence: it adds *local* parity groups so a single
> lost fragment is rebuilt from a small local set, not all k — cutting the rebuild-storm cost that's EC's
> main weakness.

**Durability math (why "11 nines" is achievable):** with independent failures, durability is dominated by **the probability of losing more copies than your redundancy *before re-replication completes*.** With 3 copies across racks and a re-replication window of minutes (6F), the chance of 3 *independent* failures hitting the same chunk's 3 replicas inside that window is astronomically small — that's where 11 nines comes from. **The real enemy is *correlated* failure** (a whole rack, a bad firmware push, a fat-fingered config) — which is why placement spreads replicas across failure domains and why you cap how fast you push changes. I'd name correlated failure as the thing the math *assumes away* and placement *defends against*.

---

### 6E. The write / append path: pipelined replication + primary lease

Writes (especially **append**) are the subtle part. Naively writing to 3 replicas raises: who orders concurrent writes? how do we not bottleneck on the slowest replica's network? GFS's answers are **pipelining** (decouple data flow from control) and a **per-chunk primary lease** (cheap ordering without consulting the master per write).

**Step 1 — decouple data flow from the control flow (pipelining):**
- The client pushes the bytes to **all three replicas in a chain/pipeline**: client → nearest replica → next-nearest → last. Each forwards the bytes onward *as it receives them* (store-and-forward in small buffers), so the data streams through the pipeline using **every machine's full outbound bandwidth** and minimizes total transfer time. The data is buffered at each replica but **not yet committed**.
- Critically, **data flow is separated from control flow** — bytes can be pushed before the order is decided.

**Step 2 — the primary replica and the lease (ordering without the master in the loop):**
- For each chunk, the master grants a **lease** (~60s, renewable) to one replica, making it the **primary**. The other replicas are **secondaries**.
- A lease is a **time-bounded leadership grant** ([consensus doc — leases/leader election](../prep/08-consensus-and-coordination.md)): only the lease-holder may define the mutation order, and the lease expires so a dead primary can't hold leadership forever. This lets the **primary serialize writes for that chunk without asking the master each time** — the master hands out leadership once and steps back.

**Step 3 — commit:**
1. Once data is buffered at all replicas, the client sends a **write/append request to the primary.**
2. The **primary assigns a serial order** (an offset) to this mutation and any others arriving concurrently, applies it locally.
3. The primary forwards the commit (with the chosen offset/order) to **all secondaries**, which apply in the **same order**.
4. Secondaries ack the primary; the primary acks the client.
5. **If any secondary fails to apply**, the primary reports an error to the client; the client **retries** (the write may have partially applied — handled by the relaxed consistency model in 6G).

**Record append (the GFS special op):** instead of the client choosing the offset (which breaks when many clients append to one file concurrently), `record append` lets the **primary choose the offset** and return it. The primary guarantees the record is appended **atomically at least once** somewhere in the chunk. This is what makes **many concurrent appenders to one file** work without a distributed lock per append — the killer feature for MapReduce-style "1,000 workers write one output file."

> **Tradeoff stated:** pipelining + lease gives near-line-rate writes (no single replica is a fan-out
> bottleneck) and lets the primary order writes locally (master not in the write loop). The cost is the
> **relaxed consistency** that record-append's "at-least-once, primary-chosen offset" implies — duplicates
> and padding can appear (6G). We accept it because the alternative (a master-coordinated, exactly-once,
> client-chosen-offset write) would put the master in every write and require distributed locking — a
> throughput killer. **This is a textbook [replication doc, leader-per-shard](../prep/05-replication-and-consistency.md)
> pattern**: the primary is a per-chunk leader that serializes writes; the lease is how we elect it
> cheaply and safely.

---

### 6F. Failure handling: heartbeats, re-replication, checksums & scrubbing

Failure is the steady state (Step 1), so self-healing is most of the system. Three mechanisms, for three failure classes.

**(1) Detecting machine/disk death — heartbeats:**
- Every chunkserver sends a **heartbeat** to the master every few seconds, piggybacking a **chunk report** (which chunks it holds). This is how the master (re)builds the chunk→location map (6C/6D) and how it detects death.
- If a chunkserver misses heartbeats past a threshold, the master marks it **dead** and its chunk replicas as lost. (Like the [KV-store gossip/φ-accrual detector (Design 05)](05-design-distributed-kv-store.md), but centralized — the master is the detector here, which is fine because there's one authority.)

**(2) Restoring redundancy — re-replication:**
- When a replica is lost (dead node, or a scrub finds corruption), the chunk drops below its replication factor. The master **prioritizes** re-replication by **how far below target** the chunk is (a chunk down to 1 copy is re-replicated *before* one down to 2 — it's closest to total loss) and schedules a **copy from a surviving replica to a new chunkserver**, respecting failure-domain placement.
- **Re-replication is throttled** (bounded fleet-wide bandwidth and per-node clone limits) so a big failure doesn't trigger a self-inflicted bandwidth storm that starves client traffic — a classic correlated-failure amplifier. The **re-replication window** (minutes) is precisely the interval the durability math (6D) integrates over: faster re-replication → fewer nines lost → why we keep recovery fast but bounded.

**(3) Bit rot / silent corruption — checksums + scrubbing:**
- Disks flip bits silently (latent sector errors, firmware bugs). A correct-looking chunk can rot. So each chunk is split into **sub-blocks (e.g. 32-64 KB) each with its own CRC32C checksum**, stored with the chunk.
- **On every read**, the chunkserver **verifies the checksum of the blocks it returns**; a mismatch → it returns an error and reports the corruption to the master, which **re-replicates from a good copy** and the bad replica is discarded. The reader never sees corrupt data.
- **Scrubbing**: a **background scrubber** periodically reads and re-verifies *all* chunks (especially cold ones that are never read, so read-time verification never fires) to catch rot proactively. Found corruption → re-replicate. This is what keeps cold/archival data from silently rotting to death.

> **Staff nuance:** checksums must be **end-to-end and verified at the place that has the redundancy to
> fix it** — verifying at the reading client is too late (it has no good copy), verifying at the
> chunkserver lets the master re-replicate. And the master must distinguish **stale** (down then back, but
> behind) from **corrupt** (bad bits) from **dead** (gone): a returning node's chunks are checked by
> **version number** (6C) — a chunk whose version is older than the master's record is **stale** (it
> missed mutations while down) and is garbage-collected, not trusted. Versioning is how we avoid serving
> data from a replica that came back from the dead with old contents.

---

### 6G. The consistency model: GFS relaxed appends vs strongly-consistent modern object stores

This is the conceptual crux and where I'd be most explicit about a tradeoff.

**GFS's relaxed model (the honest, classic answer):**
GFS deliberately offers **weak consistency for the data plane** to keep appends fast and lock-free:
- A region of a file is **consistent** if all replicas have the same bytes, and **defined** if additionally a reader sees a mutation in its entirety.
- After a **successful record append**, the record is **defined** but the file may also contain:
  - **duplicates** — a retried append (after a partial failure) appends the record again on a different attempt; the record appears **at least once**, possibly more.
  - **padding / inconsistent regions** — when the primary pads a chunk to keep a record from spanning a chunk boundary, or when a failed append left differing bytes across replicas.
- **The contract pushed to the application:** records carry **self-identifying checksums and unique IDs**, and readers **skip padding and dedup duplicates** at read time. GFS chose to make the **application tolerate at-least-once, self-validating records** rather than pay for exactly-once in the storage layer.

**Why GFS accepted this:** the target workload (MapReduce) appends self-describing records and is naturally tolerant — a reducer can dedup by record ID. Trading exactly-once-strong-consistency for **lock-free concurrent appends at line rate** was the right call *for that workload*.

**Contrast — strongly-consistent modern object stores (S3):**
- **S3 is now strongly read-after-write consistent** for `PUT`/`GET`/`DELETE` (since 2020): after a successful `PUT`, every subsequent `GET` sees the new object, globally. No "eventually." It achieves this with a **strongly-consistent metadata/index layer** (a replicated, consensus-backed key→version→location store) that's the authority for "what's the current version of this key," while the bytes themselves are immutable, content/version-addressed blobs (write a *new* version, never mutate in place).
- The trick: **objects are immutable; mutation is metadata.** An overwrite is a *new* immutable blob + an **atomic metadata pointer swap** in a strongly-consistent index. There's no concurrent-append-to-same-bytes problem because S3 has **no append-in-place** — you `PUT` a whole object (or assemble via multipart then atomically commit). Immutability + a consistent index = strong consistency *without* the GFS append headaches.

> **The tradeoff stated cleanly:** GFS chose **relaxed consistency to enable lock-free concurrent
> append** (great for batch pipelines that tolerate dups). S3 chose **immutable objects + a
> strongly-consistent metadata index** to give clean read-after-write at the cost of "no in-place append"
> (you version whole objects). Which is right depends on the workload: **mutable append-heavy logs →
> GFS/HDFS relaxed model; whole-object PUT/GET → S3 strong + immutable.** Naming *why* the model differs
> — append-in-place vs immutable-versioned — is the staff insight; juniors just say "S3 is consistent now"
> without the *because immutable objects move the consistency problem to a small metadata swap.*

---

### 6H. Rebalancing and hot-spot / hot-file handling

**Rebalancing — keeping disks evenly full and load even:**
- The master places new chunks on chunkservers with **below-average disk utilization** and **recent low write traffic** (so a freshly added empty node doesn't get flooded with all new writes at once, then become a read hotspot).
- A **background rebalancer** gradually migrates chunks off over-full / hot nodes onto under-full ones — important after **adding new chunkservers** (which start empty) so capacity and load equalize over time. Like re-replication, it's **throttled** so it never competes with client traffic.

**Hot-spot / hot-file handling:**
- **Hot chunk (a single popular chunk):** replication spreads chunks across nodes, but a *single* ultra-hot chunk (e.g. a small popular file, or the first chunk of a dataset everyone reads) still lands on only its 3 replicas. Mitigations: **raise that chunk's replication factor** (more replicas to spread reads across — GFS's literal fix for hot executables), **client-side caching**, and for read-only hot data front it with a **CDN/cache layer** (S3 → CloudFront). This is the same limitation as the [KV-store hot-key problem (Design 05)](05-design-distributed-kv-store.md): even spread solves hot *shards/nodes*, not a hot *single object*.
- **Write hot spot (many appenders to one file):** record-append (6E) already spreads the *bytes* across that file's chunks, but the *current* (last) chunk's primary takes all appends. For extreme cases you shard the logical output across **many files** (which MapReduce does — one output file per reducer) so there's no single hot chunk.

> **Tradeoff stated:** rebalancing/re-replication consume the same fleet bandwidth clients need, so
> everything background is **throttled and prioritized** (most-under-replicated first). The art is
> recovering fast enough to keep the durability nines (6D) without starving foreground traffic — a
> bandwidth budget the master allocates.

---

### 6I. Scaling the metadata plane — the real long-term ceiling

I'd raise this proactively because it's where the *classic* design breaks and where staff candidates show they know the frontier.

**Why metadata is the bottleneck, not data:** the data plane scales trivially — add chunkservers, get more TB and more GB/s. But **one master holds all metadata in RAM and serves all metadata ops.** That bounds (a) **total metadata size** (≈ total namespace + chunk count → RAM on one box), and (b) **metadata op throughput** (opens, creates, lookups/sec on one box). At enough files/chunks, you run out of RAM or QPS on the master long before you run out of disks.

**How modern systems shard the metadata plane:**
- **Colossus (GFS successor):** replaced the single master with **curators** — a *distributed* metadata service whose state lives in **Bigtable/Spanner** (themselves sharded, consensus-backed stores). The namespace is partitioned; there's no single in-RAM master. Recursive, elegantly.
- **HDFS Federation:** multiple **independent NameNodes**, each owning a **subtree** of the namespace, sharing the same pool of DataNodes. Plus **Router-Based Federation** (a routing layer hiding the split from clients).
- **S3 index layer:** a **horizontally partitioned, replicated key→location database** (sharded by key prefix / hashed key), fronted by stateless request routers — pure horizontal scale, no single index box.

> **Staff nuance — the partitioning gotcha:** once you shard the namespace, **cross-partition operations
> get hard** — an atomic rename/move *across* two metadata shards now needs a distributed transaction or
> careful ordering ([distributed-transactions doc](../prep/10-distributed-transactions-and-idempotency.md)).
> This is the same tension as sharding any database ([sharding doc](../prep/04-sharding-and-partitioning.md)):
> hash-partition the keyspace for even load (S3-style) and you lose cheap directory-tree operations;
> range/subtree-partition (HDFS Federation) and you keep subtree ops but risk hot subtrees. I'd state:
> **single in-RAM master is the right first design; sharding metadata is the known endgame, and the price
> of admission is giving up cheap cross-shard namespace operations.**

---

### 6J. How this underpins object stores (S3) and big data (HDFS)

The payoff — this one design is the substrate for two huge classes of system.

**As an object store (S3, GCS, Azure Blob) — tie to [blob-storage doc](../prep/14-blob-storage-and-media.md):**
- Objects = (large) files; buckets = a flat namespace. The same control/data split: a **request-routing + strongly-consistent index layer** (the "master," sharded) maps `bucket/key/version → storage locations`; **storage nodes** hold the immutable object bytes (replicated when hot, **erasure-coded when cold**).
- **Multipart upload** is exactly chunked write: split a huge object into parts, `PUT` parts in parallel to different storage nodes (parallel pipelined writes), then **atomically commit** the manifest in the index (the metadata pointer swap of 6G). Lifecycle policies transition cold objects to cheaper EC / colder tiers — the replicate-hot/EC-cold tiering of 6D made into a product feature (S3 Standard → IA → Glacier).

**As HDFS for batch analytics — tie to [batch/stream doc](../prep/18-batch-and-stream-processing.md):**
- HDFS *is* GFS's open design: **NameNode = master**, **DataNodes = chunkservers**, **blocks = chunks (128 MB)**, 3× replication (or HDFS-EC for cold).
- The killer property for MapReduce/Spark is **data locality**: the NameNode tells the scheduler *which DataNodes hold each block*, and the job scheduler **ships the computation to the data** — a mapper runs on (or near) the node holding its input block, so the 2 TB/s aggregate disk bandwidth (Step 2) is read **locally**, not pulled across the network. **This is why the master exposing chunk→location is load-bearing for big data**: locality-aware scheduling is impossible without it. Sequential 128 MB block reads feed a scan-heavy MapReduce/Spark stage at line rate; relaxed append consistency (6G) is fine because the records are self-describing.
- Spark/Presto/Hive read directly from HDFS or S3; the **separation of storage (this design) from compute** (the engine) is the foundation of the modern lakehouse — and it works *because* the data plane streams straight to compute, bypassing any central box.

> This is the "so what" I'd close the deep dives on: **DFS is not a niche — it's the storage substrate
> under object stores, data lakes, and analytics engines.** Mastering it is why every storage question
> later feels like a re-skin.

---

## Step 7 — Wrap-up (3 min): tradeoffs, failure modes, durability

**The thesis held:** split the control plane (small, strongly-consistent, in-RAM metadata) from the data plane (huge, relaxed, streamed bytes), and treat hardware failure as normal. Every piece served those two ideas.

**Remaining bottlenecks / failure modes to name before asked:**
- **The master is the SPOF and the scaling ceiling.** Mitigated short-term by **op-log + standby + Raft/ZK election + shadow read replicas**; mitigated long-term by **sharding the metadata plane** (Colossus / HDFS Federation / S3 index) — at the cost of hard cross-shard namespace ops (6I). The data plane survives a brief master outage on cached locations, which is the saving grace.
- **Re-replication / rebalance storms** — a big correlated failure (rack loss, bad config push) triggers mass re-replication that can starve client traffic; **throttle + prioritize most-under-replicated** and spread replicas across failure domains so correlated loss can't drop a whole chunk's redundancy.
- **Hot single file/chunk** — even placement solves hot *nodes*, not a hot *object*; raise replication factor / cache / CDN it (6H).
- **Stale client cache** — a cached chunk location can be wrong after a move; **chunk version numbers + retry** make a stale read fail safely and re-fetch (6E/6F).
- **Bit rot on cold data** — caught by **per-block checksums + a background scrubber** (6F); without scrubbing, never-read data silently rots to unrecoverable.
- **Correlated failure** is the real durability enemy the 11-nines math assumes away — defended by failure-domain-aware placement and slow, staged config rollouts.

**Durability math recap:** 3× cross-rack replication tolerates any 2 (and typically a full rack) losses with a minutes-long re-replication window → ~11 nines under *independent* failure; **erasure coding (RS-6-3, 1.5× overhead)** matches it at a third of the storage cost for cold data, paying with expensive rebuilds. Tier: **replicate hot, erasure-code cold.**

**What I'd tune for different workloads:**
- **Batch analytics (HDFS/MapReduce):** 128-256 MB blocks, 3× replication for hot input, **HDFS-EC for cold**, locality-aware scheduling, relaxed append consistency — lean into sequential throughput.
- **Object store (S3):** sharded strongly-consistent index, **immutable versioned objects** (clean read-after-write), multipart upload, **lifecycle tiering to EC/cold storage**, front hot objects with a CDN.
- **Archival (Glacier-class):** wide erasure codes (RS-10-4 or LRC), aggressive scrubbing, accept slow/degraded reads for minimal storage cost.

**With more time I'd add:** cross-region replication & geo-placement ([multi-region doc](../prep/22-multi-region-and-geo-distribution.md)), snapshots/versioning & a copy-on-write namespace, quotas & QoS per tenant, encryption-at-rest + per-object KMS keys ([security doc](../prep/17-security-and-auth.md)), and a proper small-file strategy (pack many tiny objects into one chunk — the thing this design deliberately scoped out).

---

## What made this staff-level

- **Stated a thesis up front (control-plane/data-plane split + failure-is-normal)** and derived every choice — huge chunks, in-RAM master, replication-vs-EC, heartbeats, relaxed appends — from it instead of listing GFS features.
- **Let estimation *force* the architecture:** the ~5 GB metadata estimate is *why* an in-RAM master works *and* *why* it's the eventual ceiling; the ~2 TB/s aggregate is *why* bytes must bypass the master. The math did real work.
- **Nailed the replication-vs-erasure-coding tradeoff with the actual overhead math** (3× → 200% vs RS-6-3 → 50% for the same durability) and gave a clean decision rule (hot/small→replicate, cold/large→EC), plus the tiering reality and the LRC refinement.
- **Explained the consistency model honestly** — GFS's relaxed at-least-once record-append *and why* (lock-free concurrent appends for MapReduce), contrasted with S3's strong consistency *and why it's achievable* (immutable objects + a small strongly-consistent metadata swap). The "because immutable" insight is the staff nuance.
- **Volunteered the failure machinery most candidates skip:** version numbers to reject resurrected-stale replicas, checksums verified *where the redundancy is*, scrubbing for cold-data rot, prioritized+throttled re-replication, and correlated failure as the thing the durability math assumes away.
- **Identified the *real* long-term bottleneck (metadata, not data)** and the known endgame (Colossus / Federation / sharded index) with its cost (hard cross-shard namespace ops) — knowing where your own elegant design breaks is the strongest seniority signal.
- **Closed with the payoff** — that this single design is the substrate under S3, HDFS, and the analytics lakehouse, with data locality as the load-bearing reason the master exposes chunk→location.

---

## Self-check (answer from memory before the mock)

- [ ] Why are chunks **64-128 MB** instead of 4 KB? Give the three reasons and the one that's primary (metadata-in-RAM).
- [ ] Why must bulk bytes **never flow through the master**? What caps the fleet if they do?
- [ ] What does the master keep **in RAM**, what does it **persist (op-log + checkpoint)**, and what does it **rebuild from heartbeats** — and why isn't chunk→location persisted?
- [ ] Replication vs erasure coding: give the **storage overhead** of 3× and RS-6-3 and the **failures each tolerates**. State the hot-vs-cold decision rule.
- [ ] Walk the **append path**: pipelined data push, primary lease, who chooses the offset, what record-append guarantees (at-least-once).
- [ ] Three failure classes and the mechanism for each: dead node (**heartbeat → re-replication**), bit rot (**checksums + scrubbing**), resurrected-stale replica (**version numbers**).
- [ ] State GFS's **relaxed consistency** (duplicates/padding, app dedups by record ID) and contrast with **S3 strong** consistency — *why* is S3 strong achievable? (immutable objects + consistent metadata swap)
- [ ] Why is **metadata the scaling ceiling** and not data? Name two ways modern systems shard it and the cost (cross-shard namespace ops).
- [ ] Where does **11 nines** come from, and what's the assumption the math makes (independent failure) that **correlated failure** violates?
- [ ] How does this design enable **MapReduce/Spark data locality**, and why is exposing chunk→location load-bearing for it?
