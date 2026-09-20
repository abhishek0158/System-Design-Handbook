# Chapter 23 — Rapid-Fire Q&A

This chapter is for last-minute revision. It has 60+ short questions with 1–2 line answers.
Use it the night before an interview, or right before you walk in. Each question links back
to a full chapter if you need more depth. Read top to bottom once, then scan again and skip
what you already know.

---

## SQL Basics

**1. WHERE vs HAVING — what is the difference?**
`WHERE` filters rows before grouping. `HAVING` filters groups after `GROUP BY`. You cannot use
an aggregate function like `COUNT()` in `WHERE`. See Chapter 3.

**2. UNION vs UNION ALL — what is the difference?**
`UNION` removes duplicate rows, so it needs a sort or hash step. `UNION ALL` keeps all rows and
is faster. Use `UNION ALL` unless you truly need duplicates removed.

**3. DELETE vs TRUNCATE vs DROP — what is the difference?**
`DELETE` removes rows one by one, is logged, and can use a `WHERE` clause. `TRUNCATE` removes
all rows fast with less logging, and resets identity counters. `DROP` removes the whole table.

**4. Primary key vs unique key — what is the difference?**
A primary key allows only one per table, cannot be `NULL`, and is the main way to find a row.
A unique key can allow `NULL` values (in most databases, one `NULL` or more, depending on the
engine) and a table can have many unique keys.

**5. INNER JOIN vs LEFT JOIN — what is the difference?**
`INNER JOIN` returns only rows that match in both tables. `LEFT JOIN` returns all rows from the
left table, with `NULL` for the right side when there is no match. See Chapter 2.

**6. CHAR vs VARCHAR — what is the difference?**
`CHAR(n)` is fixed length and pads with spaces. `VARCHAR(n)` is variable length and stores only
the actual characters, up to the limit. Use `VARCHAR` unless every value has the same length.

**7. What does `DISTINCT` do?**
It removes duplicate rows from the result. It works on the whole selected row, not just one
column, unless you name only one column.

**8. COUNT(*) vs COUNT(column) — what is the difference?**
`COUNT(*)` counts all rows, including rows with `NULL` values. `COUNT(column)` counts only rows
where that column is not `NULL`.

**9. How does `NULL` behave in comparisons?**
`NULL` means "unknown". Any comparison with `NULL` (like `NULL = NULL`) returns `NULL`, not
`true`. Use `IS NULL` or `IS NOT NULL` to check for it.

**10. Can you use a column alias in the `WHERE` clause?**
No, in most databases. `WHERE` runs before `SELECT`, so the alias does not exist yet. You can
use an alias in `ORDER BY` and, in Postgres, in `GROUP BY` and `HAVING`.

**11. What is the logical order of SQL clause execution?**
`FROM` → `WHERE` → `GROUP BY` → `HAVING` → `SELECT` → `DISTINCT` → `ORDER BY` → `LIMIT`. This
is why you cannot use a `SELECT` alias in `WHERE`. See Chapter 1.

**12. What is a self-join, and when do you need one?**
A self-join joins a table to itself, using two aliases. You need it for data like
`employees.manager_id` pointing back to `employees.emp_id`.

**13. What is the difference between `IN` and `EXISTS`?**
`IN` checks a value against a list or subquery result. `EXISTS` checks if a subquery returns
any row at all. `EXISTS` can be faster when the subquery result is large, since it can stop at
the first match.

**14. What does `LIMIT` / `OFFSET` do, and what is its downside?**
`LIMIT` caps the number of rows returned. `OFFSET` skips rows before that. Large offsets are
slow, because the database still has to scan and discard the skipped rows.

**15. What is a CTE (Common Table Expression)?**
A CTE is a named temp result set, written with `WITH name AS (...)`, used to break a complex
query into readable steps. See Chapter 4.

**16. What is the difference between a subquery and a CTE?**
A subquery is nested inside another query and has no name. A CTE has a name and sits before the
main query, so you can reuse it, and it is often easier to read.

**17. What does `GROUP BY` do, and what rule must every `SELECT` column follow?**
`GROUP BY` collapses rows that share the same value in the grouped columns into one row per
group. Every non-aggregated column in `SELECT` must also appear in `GROUP BY`.

**18. What is the difference between `ANY`/`SOME` and `ALL` in a subquery comparison?**
`x > ANY (subquery)` is true if `x` is bigger than at least one row from the subquery. `x > ALL
(subquery)` is true only if `x` is bigger than every row from the subquery.

**19. What is the difference between `ON` and `WHERE` in an outer join?**
A condition in `ON` decides what counts as a match before the outer join adds `NULL` rows. The
same condition in `WHERE` runs after, and can accidentally remove the `NULL` rows a `LEFT JOIN`
was meant to keep.

**20. What does `COALESCE()` do?**
It returns the first non-`NULL` value from a list of arguments. It is often used to show a
default value, like `COALESCE(discount, 0)`.

---

## Indexing

**21. What is an index?**
An index is a separate data structure that stores column values in sorted order with pointers
to the actual rows. It lets the database find rows without scanning the whole table.

**22. Clustered vs non-clustered index — what is the difference?**
A clustered index stores the actual table rows in index order — there can be only one per
table. A non-clustered index is a separate structure with pointers to the row. See Chapter 8.

**23. In a composite index, why does column order matter?**
A composite index on `(a, b)` can be used for lookups on `a` alone, or on `a` and `b` together,
but not for lookups on `b` alone. Put the most selective or most-filtered column first.

**24. What is a covering index?**
A covering index has all the columns a query needs, in the index itself. The database can
answer the query from the index alone, without touching the table.

**25. Why would the database not use an index that exists?**
Common reasons: the query applies a function to the indexed column (like `LOWER(name)`), the
query uses a leading wildcard (`LIKE '%abc'`), the optimizer thinks a full scan is cheaper for
a large result, or the column has low selectivity.

**26. Do indexes slow down writes?**
Yes. Every `INSERT`, `UPDATE`, or `DELETE` must also update every index on that table. More
indexes mean slower writes, so you should index only what you actually query.

**27. What is index selectivity?**
Selectivity is how well a column narrows down rows. A column like `email` is highly selective
(almost unique). A column like `status` with 3 values is low selectivity, and may not help much.

**28. What is a partial index?**
An index built on only a subset of rows, using a `WHERE` clause, like an index on `orders`
where `status = 'pending'`. It is smaller and faster for queries that filter that same way.

**29. What is the difference between a B-tree index and a hash index?**
A B-tree index supports range queries (`<`, `>`, `BETWEEN`) and sorting. A hash index supports
only exact equality lookups, and is faster for that one case.

**30. How do you check if a query is using an index?**
Run `EXPLAIN` or `EXPLAIN ANALYZE` on the query in Postgres. It shows the query plan, and you
look for `Index Scan` or `Index Only Scan` instead of `Seq Scan` on a large table.

**31. What is index bloat, and why does it matter?**
Bloat is wasted space in an index left behind by updates and deletes, especially in Postgres
before `VACUUM` cleans it up. Bloated indexes are bigger and slower to scan than they should be.

---

## Transactions & Isolation

**32. What does ACID stand for?**
Atomicity (all or nothing), Consistency (valid state to valid state), Isolation (transactions
don't interfere), Durability (committed data survives a crash). See Chapter 9.

**33. What is a dirty read?**
A transaction reads a row that another transaction changed but has not committed yet. If that
other transaction rolls back, the first transaction read data that never really existed.

**34. What is a non-repeatable read?**
A transaction reads the same row twice and gets different values, because another transaction
updated and committed that row in between.

**35. What is a phantom read?**
A transaction runs the same query twice and gets a different set of rows, because another
transaction inserted or deleted rows that match the query's condition.

**36. What are the four standard isolation levels?**
Read Uncommitted, Read Committed, Repeatable Read, and Serializable. Each one blocks more
anomalies than the last, but costs more in locking or version-checking. See Chapter 10.

**37. What isolation level do most databases use by default?**
PostgreSQL, Oracle, and SQL Server default to Read Committed. MySQL (InnoDB) defaults to
Repeatable Read.

**38. Optimistic vs pessimistic locking — what is the difference?**
Pessimistic locking locks the row up front and makes others wait. Optimistic locking lets
everyone read and write freely, then checks a version number at commit time to detect conflicts.

**39. What is MVCC (Multi-Version Concurrency Control)?**
MVCC keeps multiple versions of a row, so readers see a consistent snapshot without blocking
writers, and writers don't block readers. Postgres and MySQL InnoDB both use it. See Chapter 11.

**40. What is a deadlock?**
Two transactions each hold a lock the other one needs, so both wait forever. The database
detects this and kills one transaction (the "victim") to break the cycle.

**41. What is a "lost update"?**
Two transactions read the same row, both compute a new value, and the second write overwrites
the first, so the first transaction's change is silently lost.

**42. What is the difference between a shared lock and an exclusive lock?**
A shared lock (read lock) allows other transactions to also read the row, but not write it.
An exclusive lock (write lock) blocks both other reads and other writes.

**43. What is a savepoint?**
A savepoint marks a point inside a transaction that you can roll back to, without rolling back
the whole transaction.

**44. Can a `SELECT` statement start a transaction lock?**
A plain `SELECT` normally does not lock rows. `SELECT ... FOR UPDATE` does lock the rows it
reads, so no one else can change them until the transaction ends.

**45. What is a write skew anomaly?**
Two transactions each read overlapping data, then each write to a different row, based on what
they read. Both commit fine alone, but together they break a rule that spans both rows. Only
Serializable isolation (or explicit locking) stops this.

**46. What is two-phase locking (2PL)?**
A locking protocol with a growing phase (only acquire locks) and a shrinking phase (only release
locks, never acquire after releasing). It guarantees serializable-like behavior for lock-based
databases.

---

## Storage & Durability

**47. What is a WAL (Write-Ahead Log)?**
A log file where the database writes every change before applying it to the actual data files.
If the database crashes, it replays the WAL to recover. See Chapter 12.

**48. Why is a WAL faster than writing changes directly to data files?**
WAL writes are sequential (append-only), which is fast on disk. Writing directly to data files
means random writes scattered across the file, which is slower.

**49. B-tree vs LSM-tree — what is the main trade-off?**
A B-tree updates data in place, so reads are fast but writes can be slower (random I/O). An
LSM-tree (Log-Structured Merge-tree) buffers writes and flushes them in batches, so writes are
fast, but reads may need to check several places.

**50. Which databases use LSM-trees?**
Cassandra, RocksDB, and LevelDB use LSM-trees. They are built for write-heavy workloads.

**51. What is the buffer pool (or page cache)?**
An in-memory area where the database keeps recently used data pages, so it can avoid slow disk
reads. Writes usually go to the buffer pool first, then get flushed to disk later.

**52. What is a checkpoint?**
A point where the database writes all dirty (changed) pages from memory to disk, so recovery
after a crash has less WAL to replay.

**53. What is compaction in an LSM-tree?**
A background process that merges smaller sorted files into fewer, larger ones, and removes
deleted or overwritten data. It keeps read performance from degrading over time.

**54. What is a tombstone in an LSM-tree database?**
A marker written to record that a value was deleted, instead of removing it right away. The
actual data is cleared later, during compaction.

---

## Scaling: Replication & Partitioning

**55. What is replication?**
Copying data from one database (the primary) to one or more other databases (replicas), so
reads can be spread out and there is a backup if the primary fails. See Chapter 13.

**56. Synchronous vs asynchronous replication — what is the difference?**
Synchronous replication waits for the replica to confirm before the primary commits — safer,
but slower. Asynchronous replication commits on the primary right away and copies data later —
faster, but a replica can lose recent data if the primary crashes.

**57. What is replication lag?**
The delay between a write on the primary and that same write showing up on a replica. If an
app reads from a lagging replica right after writing, it may not see its own write.

**58. Partitioning vs sharding — what is the difference?**
Partitioning splits one table into smaller pieces, usually on one database server. Sharding
splits data across multiple separate database servers. Sharding is partitioning plus
distribution. See Chapter 14.

**59. What is a shard key?**
The column (or columns) used to decide which shard a row goes to, like `customer_id`. A bad
shard key causes uneven load, called a "hot shard".

**60. What is consistent hashing, and why do we need it?**
A hashing scheme that maps both servers and keys onto a ring, so adding or removing one server
only moves a small share of the keys, instead of reshuffling everything.

**61. What is range partitioning vs hash partitioning?**
Range partitioning splits data by value ranges, like `order_date` by month — good for range
queries. Hash partitioning spreads data evenly using a hash function — good for even load, bad
for range queries.

**62. What is a "hot shard" or "hot partition"?**
A shard that gets much more traffic than others, often because the shard key is skewed, like
one key value (a huge customer) getting far more writes than the rest.

**63. What is read replica lag risk in an app that just wrote data?**
The app writes to the primary, then immediately reads from a replica that has not caught up
yet, so it looks like the write did not happen. Fix: read from the primary right after a write,
or use "read-your-writes" consistency tricks.

**64. What is multi-primary (multi-leader) replication, and what is its main risk?**
More than one node can accept writes, and changes replicate between them. The main risk is a
write conflict, when two primaries change the same row at the same time, and something has to
resolve it.

---

## CAP Theorem & NoSQL

**65. What does the CAP theorem say?**
During a network partition, a distributed system must choose between Consistency (all nodes
see the same data) and Availability (every request gets a response). You cannot have both.
See Chapter 15.

**66. CP vs AP — give an example of each.**
CP systems (like HBase, MongoDB in some configs) refuse requests during a partition to stay
consistent. AP systems (like Cassandra, DynamoDB) keep answering, and fix up data later.

**67. What is eventual consistency?**
If no new writes happen, all replicas will eventually show the same data. Right after a write,
different nodes may return different (stale) values for a short time.

**68. What are the four main types of NoSQL databases?**
Key-value stores (Redis, DynamoDB), document stores (MongoDB), column-family stores (Cassandra,
HBase), and graph databases (Neo4j). See Chapter 16.

**69. When would you use a document database over a relational database?**
When your data is naturally nested and read together as one unit (like a product with reviews),
and your schema changes often. Not ideal when you need strong multi-record transactions.

**70. When would you use a key-value store?**
For simple, fast lookups by a single key, like a session store or a cache. Not good when you
need to query by other fields or run complex queries.

**71. When would you use a graph database?**
When your data is mostly about relationships, like social networks or fraud detection, and you
need to traverse many connected hops fast. A relational join gets slow for deep traversals.

**72. When should you pick SQL over NoSQL?**
When you need strong consistency, complex joins, multi-table transactions, or a schema that is
well understood and stable. Most backend systems with financial or order data start with SQL.
See Chapter 17.

**73. What is quorum in a distributed database?**
A rule that a read or write must be acknowledged by a minimum number of nodes (like `W` writes
and `R` reads, with `W + R > N`) to guarantee the read sees the latest write.

---

## Database Design & Normalization

**74. What is 1NF (First Normal Form) in one line?**
Each column holds one atomic value — no lists or repeating groups inside a single cell.

**75. What is 2NF (Second Normal Form) in one line?**
It is in 1NF, and every non-key column depends on the whole primary key, not just part of it
(this only matters with composite keys).

**76. What is 3NF (Third Normal Form) in one line?**
It is in 2NF, and no non-key column depends on another non-key column (no transitive
dependency). See Chapter 19.

**77. When should you denormalize a schema?**
When read performance matters more than write simplicity, and you are willing to store some
data twice to avoid expensive joins, like caching a `total_amount` on an `orders` row.

**78. Surrogate key vs natural key — what is the difference?**
A surrogate key is a made-up ID with no business meaning, like an auto-increment `emp_id`. A
natural key is a real-world value, like an email or a national ID number, used as the key.

**79. Why prefer a surrogate key over a natural key in most designs?**
Natural keys can change (a person can change their email), and using them as a key means
changing every row that references it. A surrogate key never needs to change.

**80. One-to-many vs many-to-many — how do you model each?**
One-to-many: put a foreign key on the "many" side (like `dept_id` on `employees`). Many-to-many:
use a junction table with foreign keys to both sides (like `order_items` between `orders` and
`products`).

**81. What is a junction table (or bridge table)?**
A table used only to connect two other tables in a many-to-many relationship. It usually holds
just the two foreign keys, plus maybe extra fields like `quantity` or `created_at`.

**82. When is it fine to break normalization rules on purpose?**
When you have measured a real read-performance problem, and the extra write cost and risk of
stale duplicate data are acceptable trade-offs for your use case. Always say this out loud in
an interview — "it depends on the read/write ratio."

---

That is the full rapid-fire list. If any answer feels shaky, go back to the chapter it points
to and read the full explanation with examples. In the actual interview, say your answer in one
or two lines first, then offer to go deeper — that is exactly the format this chapter trains.
