# Topic 3a: Storage Engine Foundations — B-Tree and LSM-Tree From First Principles

> **Why this topic matters:** Prep 3 (Part B) tells you *that* B-trees do in-place random writes and
> LSM-trees do append-only sequential writes, and hands you the amplification table. But that table
> is a *conclusion*. If you can't derive it from the structure, you'll parrot "LSM is write-optimized"
> and then freeze when the interviewer asks "why, mechanically?" This doc builds both engines from the
> ground up — the disk realities that force the design, the data structures, the read/write/delete
> paths step by step — so that every cell in prep 3's comparison table becomes obvious instead of
> memorized. Read this *before* prep 3 Part B, or alongside it when a cell doesn't click.

---

## Part A — The Disk Reality That Forces Both Designs

Before any tree: every storage engine is shaped by one brutal fact about persistent storage.

### Sequential I/O is dramatically faster than random I/O

- **Spinning disk (HDD):** the head must physically seek to a track and wait for rotation. A random
  4KB read can cost ~10ms; sequential reads stream at hundreds of MB/s. **Random is ~100–1000× slower.**
- **SSD:** no moving parts, so the gap narrows — but it's still real. SSDs read/write in **pages**
  (~4KB) and erase in larger **blocks** (~256KB+). A small random write can force a read-modify-write
  of a whole block, plus the SSD's own internal garbage collection. Sequential writes let the SSD
  pack data cleanly and minimize that internal churn (this is also why SSD write amplification matters).

> **The one sentence that drives everything:** *Disks love sequential access and punish random
> access.* A B-tree fights this with caching and accepts random writes; an LSM-tree restructures
> writing itself so that all writes become sequential. Everything below is a consequence.

### The vocabulary you'll keep using

- **WAL (Write-Ahead Log):** an append-only file every durable write hits *first*, before touching the
  main structure. If the process crashes, you replay the WAL to recover. Both engines use one; it's
  how a write becomes durable instantly without the main structure being fully updated yet.
- **Page / block:** the fixed-size unit (commonly 4–16KB) the engine reads and writes as a whole. You
  never read "one row" off disk — you read the page it lives on.
- **Amplification** — the three taxes you trade against each other:
  - **Write amplification:** bytes physically written to disk ÷ bytes of logical data you asked to write.
  - **Read amplification:** disk reads (or pages touched) per logical read.
  - **Space amplification:** bytes on disk ÷ bytes of live logical data.

Keep these three in your head; the whole B-tree-vs-LSM story is *which two you optimize at the third's expense.*

---

## Part B — The B-Tree, Built Up

### What it is

A **B-tree** (in practice almost always a **B+tree**) is a balanced, high-fan-out search tree of
fixed-size pages. "Balanced" = every leaf is the same depth, so every lookup costs the same. "High
fan-out" = each internal page holds *hundreds* of keys and child pointers, so the tree stays shallow.

```
                 [ • 50 • 100 • ]                 ← root (internal page: keys + child pointers)
                /      |        \
       [•20•35•]   [•65•80•]   [•120•160•]         ← internal pages
        / | \        ...           ...
   [10..19][20..34]...                             ← leaf pages: the actual rows (B+tree) + sibling links →
```

- **B+tree specifically:** *all data lives in the leaves*; internal pages hold only keys + pointers to
  route you down. Leaves are linked left-to-right into a sorted linked list — this is what makes range
  scans cheap.
- **Fan-out and depth:** with fan-out ~500, a tree of 3 levels indexes ~125 million keys; 4 levels
  ~62 billion. So **even a huge table is ~3–4 page reads deep**, and the top 1–2 levels are almost
  always cached in RAM. That's the whole reason B-tree reads are fast and predictable.

### The read path (point lookup)

1. Start at the root page (cached). Binary-search its keys to pick the child pointer.
2. Follow to the next internal page (usually cached). Repeat.
3. Reach the leaf page. Binary-search it for the key. Return the row.

Cost: `O(log n)` page reads, but in practice **only the leaf (sometimes one level above) is an actual
disk read** because upper levels stay in the buffer cache. This is why prep 3 calls B-tree reads
"excellent and predictable" — one logical read maps to ~one cold page read.

### The range-scan path

Find the start key as above, then **walk the sibling pointers** across leaves in sorted order. Because
leaves are physically a sorted linked list, `WHERE created_at BETWEEN x AND y ORDER BY created_at` is a
seek + a sequential leaf walk. This is the structural reason B-trees win at range scans.

### The write path — and where the cost hides

To write a row:

1. Append the change to the **WAL** (sequential, fast) — this makes it durable immediately.
2. Find the target leaf page (a lookup) and **modify it in place** in the buffer pool.
3. Eventually flush the dirty page back to disk — at its existing on-disk location, which can be
   *anywhere*. **That flush is a random write.**

Now the two costs prep 3 names fall out mechanically:

- **Write amplification:** one logical row write caused (a) a WAL append and (b) at minimum one full
  *page* write back — you rewrite a whole 8KB page to change one 200-byte row. More if a split happens (next).
- **Random I/O:** the page's on-disk home is fixed and scattered, so flushes hit random locations.

### Page splits — the cascade

A page has fixed size. Insert into a full leaf and it must **split**: allocate a new page, move half
the entries over, and **insert a pointer to the new page into the parent**. If the parent is also full,
*it* splits too — the split can cascade up to the root (and grow the tree a level). One unlucky insert
can rewrite several pages. This is the "page splits cascade" line in prep 3 — now you know it's the
fixed-page-size constraint forcing structural surgery.

(The inverse, merging under-full pages on delete, also happens; same idea in reverse.)

### Concurrency — latches and the WAL

Because writes mutate shared pages in place, two writers touching the same page must be serialized.
Engines use short-lived **latches** (page-level locks) and careful protocols (latch-crabbing: hold the
child's latch before releasing the parent's) to keep the tree consistent under concurrent access. The
WAL also underpins crash recovery: replay it to redo committed changes that hadn't flushed. This is the
machinery prep 3 gestures at with "in-place updates need locking/latching."

### Where B-trees live

Postgres, MySQL/InnoDB, SQL Server, Oracle, SQLite, and the embedded key-value store **LMDB**. The
relational world is overwhelmingly B-tree because relational workloads are read-heavy, transactional,
and want fast point lookups + range scans + predictable latency — exactly the B-tree's strengths.

> **Say this in the room:** "B-trees update pages in place, so a write is a WAL append plus a random
> page write, and a full page splits and can cascade to the parent — that's the write amplification. In
> exchange, reads are a 3–4-level traversal with the upper levels cached, so a lookup is essentially one
> cold page read, and range scans walk linked leaf pages. Read-optimized, predictable, transactional."

---

## Part C — The LSM-Tree, Built Up

The LSM-tree starts from the opposite premise: *if random writes are the enemy, never do them.* Turn
**every** write into a sequential append, and pay the cost later, in the background, batched.

### The components

1. **Memtable** — an in-memory **sorted** structure (commonly a skip list or balanced tree). All writes
   go here. Sorted so it can be flushed in sorted order and so it's range-scannable in memory.
2. **WAL** — every write also appends to an on-disk log *before* hitting the memtable, so an in-memory
   write is still durable. (Memtable is volatile; WAL is the durability.)
3. **SSTables (Sorted String Tables)** — when the memtable fills, it's flushed to disk as an
   **immutable, sorted file**. Immutable is the key word: once written, an SSTable is never modified.
   Each SSTable carries a **sparse index** (key → offset, every Nth key) and usually a **bloom filter**.
4. **Compaction** — a background process that merges SSTables into fewer/larger ones, discarding
   superseded and deleted entries.

```
   write ──► WAL (append)  ──►  Memtable (in-RAM, sorted)
                                      │  (fills up)
                                      ▼  flush — sequential write of a sorted file
                                 SSTable L0  SSTable L0  ...
                                      │  (compaction merges)
                                      ▼
                                 SSTable L1 (larger, sorted, non-overlapping)
                                      ▼
                                 SSTable L2 ...
```

### The write path — why it's "blazing"

1. Append to WAL (sequential).
2. Insert into the memtable (in-memory, cheap).
3. Done — acknowledge the write. **No disk seek, no page read, no in-place update.**

When the memtable is full, it flushes to a new SSTable in **one sequential write** of an already-sorted
file. So the entire write path is sequential appends + in-memory inserts. *That* is why prep 3 says
write-heavy systems pick LSM — it's not a tuning trick, it's the structure refusing to do random I/O.

### The read path — and why it can be slow

A key could be in the memtable, or in *any* SSTable, with newer versions shadowing older ones. So a read:

1. Check the memtable. Found? Return (it's the newest).
2. Otherwise check SSTables **newest → oldest**. For each one:
   - Ask its **bloom filter** "could this key be here?" If *no* (definitive), skip the file entirely.
   - If *maybe*, use the sparse index to seek near the key, read the block, look.
3. First hit wins (newest version), because compaction guarantees ordering.

Worst case you probe several SSTables → multiple disk reads for one logical read. That's **read
amplification**, and it's the price of never updating in place. The mitigations are now meaningful, not magic:

- **Bloom filter:** a tiny probabilistic bitmap per SSTable. "Definitely not present" or "maybe
  present" — never a false negative. It lets a read *skip* SSTables that can't contain the key, cutting
  most of the wasted probes. This is the single most important thing making LSM reads tolerable.
- **Sparse index + block cache:** seek straight to the right block instead of scanning the file, and
  cache hot blocks in RAM.

### Deletes — the tombstone

You can't modify an immutable SSTable, so you can't erase a key in place. Instead you **write a new
record: a tombstone** — a marker meaning "this key is deleted as of now." On read, encountering a
tombstone (as the newest version) means "return not-found." The actual bytes of the old value live on
in older SSTables until **compaction** finally drops both the old value and the tombstone.

Consequence (prep 3's "footgun"): if you do many deletes — especially **range deletes** in Cassandra —
tombstones pile up. Reads must scan *through* them, and a range query can read thousands of tombstones
to return a handful of live rows. Tombstone buildup is a classic production incident.

### Compaction — the hidden tax, in detail

Compaction is the merge-sort that keeps the LSM from degenerating into thousands of SSTables. It reads
several sorted SSTables, merge-sorts them, keeps only the newest version of each key, drops tombstoned
keys, and writes fresh larger SSTables — then deletes the inputs.

What it buys you: fewer SSTables per read (less read amp), reclaimed space from dead versions (less
space amp). What it costs you: it **rewrites data that was already on disk**, repeatedly, over a key's
lifetime — that's **write amplification over time** — and it competes with live traffic for disk
bandwidth and CPU, causing latency spikes if it isn't paced. The two main strategies are a direct
amplification trade:

| Strategy | How it merges | Optimizes | Pays in |
|---|---|---|---|
| **Size-tiered (STCS)** | Merge SSTables of similar size into a bigger one | **Low write amp**, high write throughput | High **space amp** (multiple big copies coexist) + high read amp |
| **Leveled (LCS)** | Keep levels of non-overlapping SSTables; each level ~10× the last | **Low read & space amp** (≤1 SSTable per level to check) | High **write amp** (data rewritten on each level promotion) |

This *is* the amplification triangle made concrete: STCS trades space/read for write; LCS trades write
for space/read. Cassandra defaults to STCS (write-ingest friendly); RocksDB defaults to leveled
(read/space friendly). Knowing *why* you'd switch is a strong staff signal.

### Where LSM-trees live

Cassandra, ScyllaDB, HBase, Bigtable, **RocksDB** (the embedded engine under MyRocks, TiKV,
CockroachDB, and many others), and LevelDB. Write-heavy ingest, time-series, event logs, and
distributed wide-column stores cluster here.

> **Say this in the room:** "LSM buffers writes in a sorted in-memory memtable plus a WAL, then flushes
> immutable sorted SSTables — every write is a sequential append, which is why it eats write volume.
> The cost moves to reads (a key can be in any SSTable, so I lean on bloom filters to skip files) and
> to background compaction, which rewrites data to keep file count and dead versions down. Deletes are
> tombstones removed only at compaction. I'd tune compaction strategy — leveled for read-heavy, size-
> tiered for write-heavy — to pick which amplification I pay."

---

## Part D — The Two, Side by Side (now you can derive every row)

This is prep 3's table, but each cell is annotated with *the structural reason* so you can reconstruct
it rather than recall it.

| Dimension | B-Tree | LSM-Tree | The structural reason |
|---|---|---|---|
| Write path | In-place page update, random I/O | Append to memtable+WAL, sequential | B-tree's page has a fixed home; LSM never updates in place |
| Write throughput | Lower | High | Random writes + page splits vs pure sequential appends |
| Read latency | Low, predictable | Higher, variable | One cached traversal vs probing N SSTables |
| Write amplification | Moderate (page + WAL + splits) | High over lifetime | Rewriting a whole page vs rewriting data on every compaction |
| Read amplification | Low | High (bloom-mitigated) | One leaf vs many SSTables to check |
| Space amplification | Low | Higher | In-place = one copy; LSM holds stale versions until compaction |
| Deletes | Immediate (in place) | Tombstone + later compaction | Can't mutate an immutable SSTable |
| Range scans | Excellent (linked leaves) | Good (SSTables are sorted) | Both keep data sorted; B-tree's leaf list is tighter |
| Concurrency | Page latches, WAL | Append-friendly, compaction in background | In-place mutation needs locking; appends contend less |
| Wins when | Read-heavy, transactional, ad-hoc | Write-heavy, ingest, time-series | Pick which amplification hurts least for your workload |

### The mental model to leave with

- **B-tree = pay at write time, once.** Keep the structure perfectly organized on every write so reads
  are always cheap and predictable. Random-write cost is the bill.
- **LSM = defer and batch.** Make writes trivially cheap now; pay later in background compaction and in
  read-time probing. Compaction load and read amplification are the bill.
- **The amplification triangle is inescapable.** You can be good at two of {read amp, write amp, space
  amp}; the third suffers. Engine family + compaction strategy is *how you choose your poison.* There
  is no free lunch and no universal winner — say that out loud.

---

## Part E — How This Connects Back to Prep 3

When prep 3 says these things, you can now justify them mechanically:

- **"Wide-column (Cassandra/Bigtable) → LSM writes, partition+clustering keys"** — Cassandra is LSM, so
  it absorbs high write volume; its clustering-key range scans within a partition map directly onto
  SSTables being sorted. The "tombstone-laden range deletes" failure mode is the delete path above.
- **"Postgres/MySQL → B-tree"** — relational, read-heavy, transactional, ad-hoc queries → exactly the
  B-tree's predictable-read, range-scan, MVCC-friendly sweet spot.
- **Part D's indexing** — a secondary index is itself a B-tree (or an extra LSM keyspace). "5 indexes ≈
  5× write amplification" is literally "5 more B-trees to keep balanced (page splits and all) on every
  write." On an LSM engine it's "5 more keyspaces to compact." Same idea, both engines.
- **Part E's MVCC + VACUUM** — Postgres keeps old row versions and needs VACUUM to reclaim them; that's
  *space amplification and background cleanup in a B-tree world* — the direct analog of LSM compaction
  reclaiming dead versions. Both engines defer garbage collection; both make you pay for it eventually.

> **The bridge sentence:** "The storage engine is the lens. Once I know a store is B-tree or LSM, its
> whole read/write/space/delete personality — and most of its failure modes — falls out, and prep 3's
> 'when to reach for which' table stops being a list to memorize and becomes something I can rederive."

---

### Self-check before the mock (answer these from memory)
- [ ] Why is sequential I/O so much faster than random I/O, on both HDD and SSD?
- [ ] Draw a B+tree. Where does the data live, and why are range scans cheap?
- [ ] Walk the B-tree write path and name the two sources of write amplification (page rewrite + splits).
- [ ] What is a page split, and why can it cascade to the root?
- [ ] Name the four LSM components (memtable, WAL, SSTable, compaction) and what each does.
- [ ] Walk the LSM read path and explain exactly what a bloom filter saves you.
- [ ] Why can't an LSM delete in place, and what does it write instead? What's the downstream risk?
- [ ] What does compaction do, and contrast size-tiered vs leveled in amplification terms.
- [ ] State the amplification triangle and which two each engine optimizes.
- [ ] Given "write-heavy time-series ingest" vs "read-heavy transactional with ad-hoc queries," pick the engine and justify it from the structure.
