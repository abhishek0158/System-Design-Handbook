# Chapter 5 — Locks & MVCC

Two transactions touch the same row at the same time. What happens? This is the core question
behind locks and MVCC. Interviewers ask this to check if you understand how databases give you
correctness *and* speed at the same time.

## The Idea

A **lock** is a flag a database puts on some data. The flag says "someone is using this, wait
your turn." Locks stop two transactions from stepping on each other.

There are two common lock types:

- **Shared lock (read lock).** Many transactions can hold a shared lock on the same row at the
  same time. It only blocks a transaction that wants to *write* that row. Think of it like many
  people reading the same book at once — fine, as long as no one is erasing pages.
- **Exclusive lock (write lock).** Only one transaction can hold this. It blocks everyone else —
  no other read lock, no other write lock. Think of someone erasing and rewriting a page — no one
  else can read or write it until they finish.

Locks also differ by **scope**:

- **Row-level lock.** Locks just one row. Other transactions can freely read or write other rows
  in the same table. Most modern databases (PostgreSQL, MySQL InnoDB) use row-level locks by
  default.
- **Table-level lock.** Locks the whole table. Even a row no one else cares about becomes
  unreachable for writes. This is simpler for the database to manage, but it kills concurrency —
  many unrelated transactions now wait for each other.

Example: `orders(order_id, customer_id, order_date, status, total_amount)`. If transaction A takes
a row lock on `order_id = 101`, transaction B can still update `order_id = 102` freely. If A took
a table lock instead, B would have to wait even though it wants a completely different row.

Now the second big idea: **MVCC**, or **Multi-Version Concurrency Control**. Instead of making
readers take a lock before they read, the database keeps **multiple versions of a row**. When a
writer updates a row, the old version is not deleted right away — it is kept around. A reader that
started earlier keeps seeing the old version (a consistent snapshot), while the writer works on
the new version. No lock needed for the read.

A **snapshot** here just means "the state of the data as of some point in time." Under MVCC, when
your transaction starts (or, depending on isolation level, when each statement starts — see
Chapter 4), the database decides which row versions are visible to you. Any version created by a
transaction that committed after that point is invisible to you, even though it physically exists
in the table. This is how PostgreSQL and MySQL InnoDB give you a stable, repeatable view of data
without ever making a reader wait for a lock.

A simple mental model: imagine every row has a hidden version number, like `v1`, `v2`, `v3`. An
`UPDATE` does not overwrite `v1` in place — it creates `v2` and marks `v1` as "old." Readers that
began before `v2` was committed keep reading `v1`. Readers that begin after keep reading `v2`.
Nobody blocks anybody. Eventually, once no transaction can possibly need `v1` anymore, the
database reclaims that space.

Result: **readers do not block writers, and writers do not block readers.** Both PostgreSQL and
MySQL (InnoDB engine) use MVCC as their default concurrency model.

Sometimes you want to lock a row on purpose before you update it — for example, to stop two
transactions from both reading a stale balance and both deciding to deduct from it. PostgreSQL
gives you `SELECT ... FOR UPDATE` for this. It reads the row and also takes a write lock (an
exclusive lock) on it, right inside the transaction. Any other transaction that tries to
`SELECT ... FOR UPDATE` (or update) the same row must wait until the first transaction commits or
rolls back.

```sql
BEGIN;
SELECT total_amount FROM orders WHERE order_id = 101 FOR UPDATE;
-- do some logic, then:
UPDATE orders SET status = 'shipped' WHERE order_id = 101;
COMMIT;
```

## Why It Matters — the Reasoning

Here is the real trade-off, and it is the point interviewers want you to explain.

**Plain locking (readers take locks too) hurts concurrency.** If every `SELECT` had to take a
shared lock, and every shared lock blocked writers, then a table with lots of reads would have
writers waiting all the time. On a read-heavy system (most web apps: many reads, fewer writes),
this is a disaster. Reads pile up, writers starve, throughput drops.

**MVCC's win is exactly here: reads never block writes, and writes never block reads.** A reader
just gets handed the version of the row that was valid when its transaction (or statement)
started. It does not care that a writer is, right now, creating a newer version. This is why
MVCC databases handle heavy read traffic so well — the "give me an old snapshot" case is nearly
free.

**But MVCC is not free — it has a cleanup cost.** Every update makes an old row version, not just
overwriting the old one. If nobody ever needed the old row versions, we could throw them away
immediately. But some transaction, still running, might need it. So the database must keep old
versions around until no one needs them, then clean up. In PostgreSQL, this cleanup job is called
**VACUUM**. If VACUUM does not run often enough, the table fills up with dead old row versions —
this is called "table bloat." It wastes disk space and slows down scans, because the database
still has to skip over dead versions to find the live one.

**Long-running transactions make this worse.** If a transaction stays open for a long time (say,
someone forgot to commit, or a batch job runs for an hour), the database cannot clean up any row
version that the transaction might still need to see. Old versions pile up until that transaction
finishes. This is a classic "why is my database slow" interview trap: the answer is often "check
for long-running or idle transactions."

**Row-level vs table-level locks is really about the same theme: contention.** A row lock only
blocks the transactions that actually care about that row. A table lock blocks everyone, even
transactions working on unrelated rows. Fine-grained locks cost the database a bit more to track
(it must remember locks per row, not just one flag per table), but they buy much better
concurrency. This is a general pattern in system design: finer-grained locking almost always
trades a little bookkeeping overhead for a lot more parallelism.

You will still see table-level locks used on purpose sometimes — for example, `ALTER TABLE` often
needs to lock the whole table, because it changes the shape of every row. Some batch jobs also
take a table lock deliberately when they must process the whole table and cannot allow anyone
else to change it midway. So table locks are not "wrong," just much more expensive — use them only
when the operation truly touches everything.

**`SELECT ... FOR UPDATE` is the escape hatch.** MVCC is great for plain reads, but sometimes you
need to *coordinate* — for example, "read the balance, then write a new balance, and make sure
nobody else changed it in between." For that you deliberately give up MVCC's non-blocking read
and take an old-style exclusive lock. This connects directly to Chapter 6 (Optimistic vs
Pessimistic Locking) — `SELECT ... FOR UPDATE` is the classic tool for pessimistic locking.

## Common Interview Questions

**Q1: What is the difference between a shared lock and an exclusive lock?**
A shared lock lets many transactions read the same row at once; it blocks only writers. An
exclusive lock lets exactly one transaction touch the row, blocking every other reader and writer.
The reasoning: shared locks allow concurrent reads because reads do not conflict with each other,
but any write must have the row all to itself to guarantee no one sees a half-finished change.

**Q2: What is MVCC, and why is it a good design?**
MVCC (Multi-Version Concurrency Control) means the database keeps multiple versions of a row.
Readers see a consistent snapshot from an older version while a writer builds a new one. It is
good because it avoids the classic problem of plain locking: readers taking locks and blocking
writers. On a read-heavy system, that would badly hurt throughput. With MVCC, reads are close to
free and never wait on writes.

**Q3: In MVCC, does a read ever block a write, or a write block a read?**
No — that is the whole point of MVCC. A reader works off a snapshot version that already exists,
so it does not need to wait for a writer to finish. A writer creates a new version without
needing to wait for readers to finish looking at the old one. They only conflict with each other
in the write-write case, or when you explicitly ask for a lock (like `FOR UPDATE`).

**Q4: What does `SELECT ... FOR UPDATE` do, and why would you use it?**
It reads a row and immediately takes an exclusive (write) lock on it, inside the current
transaction. You use it when you must read a value and act on it — for example, check a balance
before deducting — and you cannot risk another transaction changing that row in between. It is a
deliberate opt-out of MVCC's non-blocking read, in exchange for safety on a critical update.

**Q5: What is the cost of MVCC?**
Old row versions do not disappear immediately — they must be kept until no running transaction
needs them, then cleaned up. In PostgreSQL, this cleanup is VACUUM. If it falls behind, the table
bloats with dead versions, wasting space and slowing scans. Long-running transactions make this
worse, because they force the database to keep old versions alive for longer.

**Q6: What is the difference between a row-level lock and a table-level lock, and why does it
matter?**
A row-level lock blocks only transactions touching that same row; a table-level lock blocks every
transaction touching the table, even unrelated rows. It matters because table locks are simpler
but destroy concurrency — one transaction can stall many others that do not even conflict with it.
Row-level locks cost a bit more bookkeeping but scale much better under concurrent load.

## Quick Recall

- Shared (read) lock: many can hold it, blocks writers only. Exclusive (write) lock: only one
  holder, blocks everyone.
- Row-level lock blocks only that row; table-level lock blocks the whole table — much worse for
  concurrency.
- MVCC keeps multiple row versions so readers get a consistent snapshot without locking; readers
  never block writers, and writers never block readers.
- PostgreSQL and MySQL InnoDB both use MVCC.
- `SELECT ... FOR UPDATE` takes a deliberate exclusive lock on a read, for cases where you must
  read-then-write safely (see Chapter 6, pessimistic locking).
- **Biggest gotcha:** MVCC is not free. Old versions must be cleaned up (VACUUM in PostgreSQL),
  and long-running transactions delay that cleanup, causing table bloat and slower queries.
