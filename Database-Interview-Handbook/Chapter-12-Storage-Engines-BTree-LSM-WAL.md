# Chapter 12 — Storage Engines: B-tree, LSM & WAL

A storage engine is the part of a database that decides how data sits on disk, and how reads and writes touch that data. Interviewers ask about this to see if you understand what happens below the SQL layer, and whether you can pick the right database for a write-heavy or read-heavy system.

## Key Concepts

### Pages, blocks, and the buffer pool

A database does not read or write single rows to disk. It reads and writes fixed-size chunks called **pages** (also called **blocks**). A typical page is 4 KB, 8 KB, or 16 KB. Postgres uses 8 KB pages by default. Each page holds many rows, plus a small header.

Why pages, not rows? Disks (and even SSDs) are fast at reading a full block, but slow at doing many tiny, separate reads. Grouping rows into pages means one disk read can fetch many rows at once. It also matches how the operating system and disk hardware already work — they move data in blocks too.

The **buffer pool** is an in-memory cache of pages. When a query needs a row, the engine first checks if the row's page is already in the buffer pool. If yes, it reads from memory — fast. If no, it loads the page from disk into the buffer pool, then reads it. Written pages are marked **dirty** and flushed to disk later, not on every single write. This buffering is why a database can be much faster than "one disk access per row."

```
Query needs row X
   |
   v
Is X's page in the buffer pool (RAM)?
   |-- yes --> read from RAM (fast)
   |-- no  --> load page from disk into buffer pool, then read
```

A **heap table** is the simplest way to store rows: rows are appended in no particular order, spread across pages, with no sort order guaranteed. Postgres tables are heap tables by default. To find a row quickly in a heap table, you need an index (Chapter 8) — otherwise the engine must scan every page.

If a page fills up, the engine either finds free space in another existing page (Postgres tracks this with a **free space map**) or allocates a brand-new page at the end of the file. A very large row — one bigger than a single page can hold, such as a row with a large text or JSON column — gets special handling: Postgres moves the oversized value out to a separate storage area (called **TOAST**) and keeps only a small pointer in the main page. You do not need the details for an interview, just the idea: pages have a fixed size, and the engine has to work around that limit for big values.

MySQL note: MySQL is unusual in that its **storage engine is pluggable** — you choose it per table with `ENGINE=InnoDB` (or, historically, `ENGINE=MyISAM`). InnoDB is the default today, uses a B-tree with pages and a buffer pool exactly like Postgres, and adds full transaction and crash-recovery support. The older MyISAM engine did not support transactions or crash-safe recovery, and is now mostly a legacy option. Postgres, by contrast, has one built-in storage engine — you do not choose it per table.

### The Write-Ahead Log (WAL) and durability

Durability is the "D" in ACID (Chapter 9): once a transaction commits, its changes must survive a crash, even if the server loses power one second later. But writing every change straight into the data pages on disk, for every transaction, would be slow — data pages can be scattered all over the disk, and a crash mid-write could leave a page half-written and corrupt.

The fix is the **Write-Ahead Log (WAL)**, also called a **redo log** or **commit log** in some databases. The rule is simple: **before changing a data page, first write a record of that change to the log, and make sure the log record is safely on disk.** Only after that is the actual data page allowed to be updated.

```
1. Transaction changes a row
2. Write a WAL record describing the change  -----> fsync() to disk
3. Only now: commit is acknowledged to the client
4. Data page is updated in the buffer pool, flushed to disk later
```

**fsync** is a system call that forces the operating system to actually write data to physical disk, instead of leaving it sitting in an OS-level write cache. Without fsync, "written to disk" might really mean "sitting in a cache that disappears if the machine loses power." The WAL record must be fsynced before the transaction is told "committed" — this is the actual durability guarantee.

Why is this safe and also fast?
- **Safe:** if the server crashes right after commit, the WAL record is already durable on disk. On restart, the database **replays the WAL** — reapplies every committed change that was not yet written to the data pages. This is called **crash recovery** or **redo**.
- **Fast:** the WAL is written **sequentially** — one record after another, always at the end of the log file. Sequential writes are much cheaper than random writes to scattered data pages, especially on spinning disks, and still cheaper on SSDs. The data pages themselves can be updated later, in memory, and flushed lazily.

**Checkpoints.** The WAL cannot grow forever — replaying years of history after a crash would take forever, and the log file would use unlimited disk space. A **checkpoint** is a point where the database guarantees all changes up to that point are safely written to the actual data pages on disk. After a checkpoint, older WAL records are no longer needed for crash recovery and can be discarded (or archived, for point-in-time recovery). On restart, the database only needs to replay WAL records written after the last checkpoint.

```
WAL:   [rec1][rec2][rec3][CHECKPOINT][rec4][rec5]  <- crash here
                                       ^^^^^^^^^^
                              only these need replay after crash
```

One line to remember: **WAL gives durability and turns random writes into sequential writes; checkpoints keep the log from growing forever.**

### B-tree storage engine

A **B-tree** is a balanced tree data structure where each node can have many children, and the tree stays balanced (all leaf nodes at the same depth) as data is inserted or deleted. Both Postgres and MySQL's default engine, **InnoDB**, store table data using a B-tree structure (InnoDB calls its main structure a **clustered index**, which is a B-tree where the leaf pages hold the actual row data).

How writes work in a B-tree engine: to update a row, the engine finds the exact page holding that row (via WAL for durability, as above) and **updates it in place** — the row's page on disk is modified directly. This means:
- **Reads are fast.** Data for a given key lives at one place. A lookup walks down the tree — usually 3-4 levels even for millions of rows — and finds the row directly. Range scans (`WHERE id BETWEEN 10 AND 20`) are efficient too, since B-tree leaf pages are linked in sorted key order.
- **Random writes can be expensive.** An update might touch a page that is not in the buffer pool, causing a random disk read first, then eventually a random write when that dirty page gets flushed. If updates are scattered across many different keys, this means many scattered random I/O operations, which are slow on spinning disks and still add overhead on SSDs (plus wear).

This connects directly to indexing (Chapter 8): in InnoDB, the table itself is stored as a B-tree keyed on the primary key — this is called a **clustered index**, because the actual row data sits in the same tree as the primary key. Every other (secondary) index is a separate, smaller B-tree that stores the indexed column plus the primary key, then does a second lookup into the clustered index to fetch the full row. Postgres works a little differently: the heap table is separate from all its indexes, so even the primary key is a normal index pointing at heap page locations, not a clustered structure. You do not need to reproduce this difference in full, but knowing "InnoDB clusters on the primary key, Postgres does not" is a common follow-up.

```
B-tree (simplified, one branch):

           [ 50 ]
          /       \
     [20, 35]    [70, 90]
     /  |  \        |   \
  leaf pages with actual rows, sorted by key, linked left-to-right
```

### LSM-tree storage engine

An **LSM-tree (Log-Structured Merge tree)** takes a different approach: it never updates data in place. Instead, it makes writing fast by always writing sequentially, and pushes the cost of organizing data to a background process. Cassandra, RocksDB, and LevelDB all use an LSM-tree.

How writes work:
1. A write first goes to an **in-memory memtable** — a sorted, in-memory structure (often a skip list or balanced tree). This is very fast, since it is a memory write, not a disk write.
2. For durability, the write is also appended to a WAL on disk first (same idea as before — log first, so a crash does not lose data sitting only in the memtable).
3. When the memtable gets full, it is **flushed** to disk as a new, immutable, sorted file called an **SSTable (Sorted String Table)**. "Immutable" means once written, an SSTable is never modified — only replaced.
4. Over time, many SSTables pile up on disk. A background process called **compaction** merges multiple SSTables into fewer, larger ones — combining sorted files (like a merge step in merge-sort), dropping deleted or overwritten values along the way.

```
Write -----> WAL (disk, for durability)
Write -----> memtable (in-memory, sorted)
                 |  (memtable fills up)
                 v
            flush to SSTable (disk, sorted, immutable)
                 |
     [SSTable1] [SSTable2] [SSTable3] ...
                 |  (background compaction)
                 v
            fewer, larger, merged SSTables
```

Consequences of this design:
- **Writes are fast.** A write is just an append to a log plus a memory update — no seeking to find the "right" existing page, no in-place modification.
- **Reads can be slower.** The engine may need to check the memtable, then possibly several SSTables, since the latest value for a key could be in any of them (newest first, until found). LSM engines use tricks like **Bloom filters** (a compact structure that quickly says "this key is definitely not in this SSTable," avoiding needless disk reads) to speed this up, but a read can still touch more files than a B-tree read touches pages.
- **Compaction uses background CPU and I/O.** It is necessary to keep read performance reasonable and to reclaim space from deleted/overwritten rows, but it competes with live traffic for disk bandwidth.

There are two common compaction strategies, and interviewers sometimes ask you to name them: **size-tiered compaction** merges SSTables of similar size into a larger one once there are enough of them — simple, and good for write-heavy workloads, but reads may need to check more files. **Leveled compaction** organizes SSTables into levels of increasing size, and guarantees keys within a level do not overlap — this keeps read amplification lower and bounds how much disk space old versions can waste, at the cost of more total compaction I/O. RocksDB and Cassandra both let you choose between strategies per table; the "right" one again depends on whether the workload leans more toward writes or reads.

A **tombstone** is how an LSM-tree represents a delete: instead of removing the value immediately, it writes a special marker recording "this key is deleted." The real removal only happens later, when compaction merges that SSTable with the one containing the original value and drops both. This is why deletes in an LSM-tree database are not instantly free — they add a write, and disk space is not reclaimed until compaction runs.

### The core trade-off, and read/write amplification

**Write amplification** means: for one logical write from the application, how much data actually gets written to disk in total (including background work like compaction or page flushes)? Higher write amplification wears out SSDs faster and uses more disk bandwidth.

**Read amplification** means: for one logical read, how many separate disk reads (or files, or pages) does the engine have to check before it can return the answer?

- **B-tree:** in-place updates keep write amplification lower for random small updates only in some workloads, but each in-place update to a large page can still rewrite a full page for a tiny change — and a random write pattern can be genuinely expensive on top of that. Reads are direct — usually one page lookup path — so read amplification is low. **B-trees are generally read-optimized.**
- **LSM-tree:** writes are cheap up front (just append), but the same piece of data gets rewritten multiple times as it moves through compaction (memtable → SSTable → merged SSTable → merged again). This is classic write amplification, but it is *sequential*, which is still often faster overall for heavy write throughput than B-tree random writes. Reads may need to check multiple SSTables, so read amplification is higher. **LSM-trees are generally write-optimized.**

**The one-line trade-off to say in an interview:** B-trees favor fast, predictable reads with in-place updates; LSM-trees favor fast, high-throughput writes by turning random writes into sequential ones, at the cost of extra background compaction work and slightly slower reads.

| | B-tree | LSM-tree |
|---|---|---|
| Update style | in place | append-only, then compact |
| Best at | reads, range scans | high write throughput |
| Write pattern | can be random | always sequential |
| Read cost | one lookup path | may check multiple SSTables |
| Background cost | page flushing | compaction |
| Used by | PostgreSQL, MySQL InnoDB | Cassandra, RocksDB, LevelDB |

## The Questions They Ask

**Q1: How does a database guarantee durability after a commit?**
It uses a Write-Ahead Log. Before a change is allowed to affect the actual data pages, the database writes a log record describing the change and calls fsync to force it onto physical disk. Only after that fsync succeeds does the database tell the client the transaction committed. If the server crashes right after, the WAL record is already safe on disk, so on restart the database replays the WAL and reapplies the change. This is why the log is called "write-ahead" — the log write always happens ahead of the data page write.
*Follow-up:* "What if the crash happens before the fsync completes?" — Then the client never received a commit acknowledgment, so from the client's point of view the transaction is not confirmed to have happened. This is correct behavior, not a bug — durability only applies to transactions the database has actually told you succeeded.

**Q2: What is a WAL, and why not just write directly to the data files?**
A WAL is an append-only log of every change, written sequentially, and fsynced before a transaction is confirmed committed. Writing directly to data files first would mean random, scattered disk writes for every transaction, and a crash mid-write could leave a page half-updated and corrupt, with no way to know what state it was in. The WAL avoids both problems: it is sequential (fast) and it lets the database replay a clean, ordered history to recover a consistent state after a crash.
*Follow-up:* "How does the WAL relate to replication?" — Many databases (including Postgres) ship WAL records to replicas, and the replica replays the same log to stay in sync. This is covered in Chapter 13.

**Q3: B-tree vs LSM-tree — when would you pick each?**
Pick a **B-tree** engine (Postgres, MySQL/InnoDB) when the workload is read-heavy, or has a healthy mix of reads and writes, and you need fast point lookups and range scans with predictable latency. Pick an **LSM-tree** engine (Cassandra, RocksDB) when the workload is write-heavy — high volumes of inserts or updates, like event logging, time-series ingestion, or metrics — because it turns writes into fast sequential appends and defers the cost of organizing the data to background compaction. Say the "it depends" clearly: the honest answer always names the read/write ratio, not just the database name.
*Follow-up:* "Can a B-tree database still handle heavy writes?" — Yes, up to a point — good hardware, enough buffer pool memory, and batching help a lot. But at very high sustained write throughput, LSM-based systems typically scale further, because they avoid the random I/O pattern that hurts B-trees.

**Q4: Why is Cassandra good at writes?**
Cassandra uses an LSM-tree. A write goes to a commit log (its WAL) and an in-memory memtable — both fast, sequential or in-memory operations, with no read-before-write and no seeking to an existing row location. The engine never updates a row in place. This means write latency does not depend on where the row "lives" on disk, unlike a B-tree, where an update requires finding and modifying a specific existing page. The cost is paid later and separately, through background compaction, which does not block the write path that the client is waiting on.
*Follow-up:* "Does this mean Cassandra reads are slow?" — Not necessarily slow, but reads do more work per request than in a B-tree: potentially checking the memtable plus several SSTables. Bloom filters and caching reduce this cost in practice.

**Q5: What is compaction, and why is it needed?**
Compaction is a background process in LSM-tree engines that merges multiple SSTables into fewer, larger sorted files. It is needed for two reasons: first, without it, reads would have to check more and more SSTables over time, making reads progressively slower; second, it is when overwritten or deleted values actually get physically removed, reclaiming disk space. Without compaction, an LSM-tree would keep every historical version of every key forever.
*Follow-up:* "Can compaction cause performance problems?" — Yes, it competes with live read/write traffic for disk I/O and CPU. This is a known operational concern with LSM databases — sudden compaction load can cause latency spikes, and tuning compaction strategy is a real production task.

**Q6: What is a checkpoint, and why does the WAL need one?**
A checkpoint marks a point where the database guarantees all changes up to that point are safely on the actual data pages, not just in the log. This lets the database discard (or archive) older WAL records, since they are no longer needed to reconstruct state after a crash. Without checkpoints, the WAL would grow forever, and crash recovery would need to replay the entire history of the database since it started, which would take longer and longer over time.
*Follow-up:* "What decides when a checkpoint happens?" — Usually a time interval, a size threshold on the WAL, or both, configurable in the database settings (for example, Postgres's `checkpoint_timeout` and `max_wal_size`).

**Q7: Why can deletes be tricky in a database like Cassandra?**
Cassandra never updates or removes data in place, so a delete is written as a **tombstone** — a marker saying "this key is deleted" — instead of physically erasing anything. The old value and the tombstone both sit on disk until a future compaction merges them and drops both. Until that happens, the tombstone itself takes up space and read time, since the engine has to see it to know the key is deleted. If an application deletes and recreates the same keys very often, or accumulates a huge number of tombstones without letting compaction catch up, reads can slow down noticeably. This is a well-known operational issue with LSM-based systems, and it is worth mentioning if asked about deletes.
*Follow-up:* "Does a B-tree engine have the same problem?" — No, a B-tree delete removes (or marks free) the row's space directly in its page, in place. Postgres has a related but different concept — it marks rows as dead and reclaims space with `VACUUM` (covered in Chapter 11 in the MVCC context) — but this is not a tombstone in the LSM sense; it does not need to be found and skipped by every future read the way an LSM tombstone can.

## Rapid-Fire

- **Page / block** → fixed-size unit of storage (commonly 8 KB), the unit the engine reads and writes.
- **Buffer pool** → in-memory cache of pages, avoids hitting disk on every read.
- **Heap table** → rows stored with no particular order; needs an index for fast lookup.
- **WAL** → append-only log of changes, fsynced before commit, replayed on crash recovery.
- **fsync** → forces data out of OS cache onto physical disk; the real durability point.
- **Checkpoint** → guarantees data pages are caught up to a point in the WAL, so older log records can be discarded.
- **B-tree** → balanced tree, updates in place, used by Postgres and MySQL/InnoDB, read-optimized.
- **LSM-tree** → memtable + SSTables + compaction, append-only, used by Cassandra/RocksDB/LevelDB, write-optimized.
- **Memtable** → in-memory sorted structure that receives writes before they flush to disk.
- **SSTable** → immutable, sorted file on disk; new writes never modify an existing SSTable.
- **Compaction** → background merge of SSTables; reclaims space, keeps reads fast.
- **Bloom filter** → quick "definitely not here" check, used to skip SSTables during a read.
- **Clustered index** → InnoDB's table storage, where row data lives inside the primary-key B-tree itself; Postgres does not do this.
- **Tombstone** → marker written for a delete in an LSM-tree; the real removal happens later, during compaction.
- **Size-tiered vs leveled compaction** → size-tiered merges similar-sized SSTables (write-friendly); leveled keeps non-overlapping levels (read-friendly, more compaction I/O).
- **Write amplification** → total bytes physically written per logical write; LSM-trees have more of this, but it is sequential.
- **Read amplification** → number of disk reads/files touched per logical read; LSM-trees have more of this than B-trees.
- **Core trade-off** → B-tree favors reads, LSM-tree favors writes.

## Common Traps & Mistakes

- **Saying "WAL makes writes slower."** It is the opposite of the intent, even though it does add a write. The WAL write is sequential and small, and it lets the actual data page write happen later and lazily. The real cost it removes is random writes on every commit and unsafe crash behavior.
- **Confusing "commit" with "data page updated on disk."** After commit, only the WAL record is guaranteed durable. The corresponding data page may still be dirty in the buffer pool and get flushed later. This is normal and correct — recovery replays the WAL to reach the same end state.
- **Thinking LSM-trees have no durability guarantee because they use an in-memory memtable.** They still write to a WAL/commit log on disk before acknowledging the write, exactly like B-tree engines. The memtable is a performance layer, not a durability shortcut.
- **Assuming LSM-trees are strictly "better" because they are write-optimized.** They trade this for higher read amplification and background compaction overhead. Whether that trade-off is worth it depends entirely on the read/write ratio of the actual workload — always say "it depends" and name the ratio.
- **Forgetting that B-tree range scans are a real strength.** Because B-tree leaf pages are linked in sorted order, range queries (`BETWEEN`, `ORDER BY` on the indexed column) are efficient. This is one reason relational databases with B-trees remain a strong default for typical OLTP workloads, not just because of joins.
- **Mixing up SSTable and memtable.** Memtable = in-memory, mutable, receives new writes. SSTable = on-disk, immutable, sorted, created by flushing a memtable. Once written, an SSTable is only replaced by compaction, never edited in place.
- **Thinking compaction is optional or just cleanup.** Skipping or falling behind on compaction directly hurts read latency, since more SSTables must be checked per read. It is a core part of how LSM-trees keep working, not a "nice to have."
- **Assuming a delete in an LSM-tree frees space immediately.** It writes a tombstone, which is itself a new write, and the space is only reclaimed later during compaction. Heavy delete-and-recreate patterns can pile up tombstones and slow down reads until compaction catches up.
- **Mixing up "clustered index" with "any index that is used often."** A clustered index has a specific meaning: the table's row data is physically stored inside the index structure, keyed on the primary key (InnoDB). Postgres indexes, including the primary key index, always point to a separate heap page — none of them are clustered in this sense, even though the term sounds generic.
