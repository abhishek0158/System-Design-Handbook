# Chapter 11 — Locking, Concurrency & MVCC

Many transactions run at the same time on a database. Locking and MVCC (Multi-Version Concurrency Control) are the two ways a database keeps this safe. Interviewers ask about this because it explains why your app hangs, deadlocks, or shows stale data under load. This chapter builds on Chapter 9 (Transactions & ACID) and Chapter 10 (Isolation Levels).

## Key Concepts

### What is a lock?

A lock is a flag a transaction places on some data to control who else can touch it. Before a transaction reads or writes a piece of data, the database may ask it to acquire a lock first. Other transactions that want a conflicting lock on the same data must wait.

Locks answer one question: "can two transactions touch the same row at the same time, and if not, who goes first?"

### Shared lock vs exclusive lock

There are two basic lock types.

- **Shared lock (S lock)**: taken for reading. Many transactions can hold a shared lock on the same row at the same time. A shared lock only blocks a transaction that wants an *exclusive* lock on that row.
- **Exclusive lock (X lock)**: taken for writing (`UPDATE`, `DELETE`, or `INSERT` on that row). Only one transaction can hold an exclusive lock on a row. It blocks every other lock, shared or exclusive, on that same row.

Think of it like a shared Google Doc: many people can have it open to read (shared lock), but only one person can be editing a specific paragraph at a time (exclusive lock), and no one else can read that exact paragraph mid-edit safely either.

| | Shared (S) held by another | Exclusive (X) held by another |
|---|---|---|
| Want Shared (S) | Allowed | Must wait |
| Want Exclusive (X) | Must wait | Must wait |

This table is called a **lock compatibility matrix**. Interviewers sometimes draw it and ask you to fill it in.

### Row-level lock vs table-level lock

A lock also has a **granularity**: how much data it covers.

- **Row-level lock**: locks a single row. Two transactions can update two different rows in the same table at the same time without blocking each other. This is what Postgres and InnoDB (MySQL's default storage engine) use for normal `UPDATE`/`DELETE` statements.
- **Table-level lock**: locks the entire table. No other transaction can write to any row in that table (and depending on the lock mode, sometimes cannot even read) until the lock is released.

Row-level locks give more concurrency (more transactions can proceed in parallel) but cost more bookkeeping (the database must track a lock per row). Table-level locks are simpler and cheaper to track, but they force transactions to queue up even when they touch completely different rows.

```sql
-- Row-level: only the row with order_id = 101 is locked
UPDATE orders SET status = 'shipped' WHERE order_id = 101;

-- Table-level: blocks other writers to the whole table
LOCK TABLE orders IN EXCLUSIVE MODE;
```

In interviews, the expected answer is: "Postgres and MySQL InnoDB use row-level locking for normal DML. Table-level locks mostly show up for schema changes (`ALTER TABLE`) or explicit `LOCK TABLE` statements."

### Optimistic vs pessimistic concurrency control

These are two different strategies for handling the case where two transactions might touch the same row.

**Pessimistic concurrency control** assumes conflicts are likely. It locks the row up front, before doing any work, so no one else can touch it until you are done.

```sql
BEGIN;
SELECT * FROM products WHERE product_id = 55 FOR UPDATE;
-- row 55 is now locked; other transactions trying to
-- SELECT ... FOR UPDATE or UPDATE this row must wait
UPDATE products SET price = price - 10 WHERE product_id = 55;
COMMIT;
```

`SELECT ... FOR UPDATE` reads a row and takes an exclusive lock on it in the same step. Any other transaction that tries to read this row with `FOR UPDATE`, or tries to update it, must wait until this transaction commits or rolls back. A plain `SELECT` (without `FOR UPDATE`) is usually not blocked, because it does not need a lock under MVCC (explained later in this chapter).

**Use pessimistic locking when:**
- Conflicts are common (many transactions fight over the same rows, for example, a limited-seat booking system).
- The cost of retrying a failed transaction is high or awkward.
- You want a simple mental model: "lock it, change it, release it."

**Optimistic concurrency control** assumes conflicts are rare. It does not lock anything up front. Instead, it lets every transaction read and prepare its change freely, and only checks for a conflict at the very end, right before saving. If a conflict is found, the write is rejected and the app must retry.

The usual way to implement this is a **version column** (or `updated_at` timestamp) on the row.

```sql
-- Table has a version column
-- products(product_id PK, name, price, version)

-- Step 1: read the row and remember its version
SELECT product_id, price, version FROM products WHERE product_id = 55;
-- app gets back: price = 100, version = 3

-- Step 2: app computes new price, then writes back
-- with a WHERE clause that checks the version has not changed
UPDATE products
SET price = 90, version = version + 1
WHERE product_id = 55 AND version = 3;
```

If another transaction updated this row in between (so its version is now 4, not 3), the `UPDATE` above matches zero rows. The app checks `rows affected` after the statement. If it is 0, the app knows a conflict happened, and it must re-read the row and retry the whole operation. If it is 1, the update succeeded.

**Use optimistic locking when:**
- Conflicts are rare (most reads never collide with a concurrent write).
- You want higher throughput, because no one is blocked waiting for a lock.
- The workload is read-heavy, with occasional writes.

**The trade-off, in one line:** pessimistic locking pays the cost up front (waiting for a lock) even when there was no conflict; optimistic locking pays the cost only when a conflict actually happens (a retry), but that retry logic must exist in the application.

### Deadlocks

A **deadlock** happens when two transactions each hold a lock the other one needs, so both wait forever.

Classic example: two transactions update the same two rows, but in opposite order.

```sql
-- Transaction A
BEGIN;
UPDATE accounts SET balance = balance - 100 WHERE account_id = 1;
-- A now holds lock on row 1
-- ... A tries to touch row 2 next, but B has it locked

-- Transaction B (running at the same time)
BEGIN;
UPDATE accounts SET balance = balance - 50 WHERE account_id = 2;
-- B now holds lock on row 2
-- ... B tries to touch row 1 next, but A has it locked
```

Now A is waiting for the lock on row 2 (held by B), and B is waiting for the lock on row 1 (held by A). Neither can move forward. This is a deadlock.

**How the database detects it:** Postgres (and MySQL InnoDB) run a background check that builds a "who is waiting for whom" graph. If this graph has a cycle (A waits for B, and B waits for A), that is a deadlock. The database picks one transaction as the **victim**, aborts it with an error, and releases its locks. The other transaction can then continue.

```
ERROR:  deadlock detected
DETAIL: Process 123 waits for ShareLock on transaction 456; blocked by process 456.
        Process 456 waits for ShareLock on transaction 123; blocked by process 123.
```

The application must catch this error and retry the aborted transaction.

**How to avoid deadlocks:**
1. **Acquire locks in a consistent order.** If every transaction always updates `account_id = 1` before `account_id = 2` (for example, always sort by primary key first), a cycle cannot form.
2. **Keep transactions short.** The less time a transaction holds a lock, the smaller the window for a conflict.
3. **Take all the locks you need early**, ideally in one statement, instead of one row at a time across several round trips.
4. **Use a lower isolation level or `FOR UPDATE` carefully** — do not lock more rows than you truly need to change.
5. **Set a `statement_timeout` or `lock_timeout`** so a stuck transaction fails fast instead of hanging.

### MVCC (Multi-Version Concurrency Control)

MVCC is the idea that solves a hard problem: if a reader and a writer touch the same row at the same time, must the reader wait for the writer to finish?

Under pure locking, the answer would often be "yes" — a reader might need a shared lock, and a writer holding an exclusive lock would block it. MVCC avoids this.

**The key idea:** instead of overwriting a row in place, the database keeps multiple versions of that row. When a transaction updates a row, the database does not destroy the old version. It creates a new version and marks the old one as "no longer current for new transactions." Each transaction sees a consistent **snapshot** — the versions of rows that were current when its snapshot was taken (the exact point depends on the isolation level; see Chapter 10).

This gives the core benefit stated as one rule: **readers do not block writers, and writers do not block readers.** A `SELECT` never needs to wait for an `UPDATE` to finish, because it can just read the older version of the row that still matches its snapshot. A writer never waits for a reader either, because the reader is not holding any lock that blocks writes — it is just reading an older, already-committed version.

Writers still block other writers, though. Two transactions trying to `UPDATE` the exact same row at the same time still need row-level exclusive locks against each other, because there can only be one "next version" of a row.

**How Postgres implements MVCC:**
- Every row has hidden system columns: `xmin` (the ID of the transaction that created this row version) and `xmax` (the ID of the transaction that deleted or replaced this row version, if any).
- An `UPDATE` in Postgres does not modify the row in place. It inserts a brand-new row version (with a new `xmin`) and sets `xmax` on the old version.
- A transaction's snapshot is basically a rule: "show me row versions created by transactions that had already committed before my snapshot was taken, and not yet deleted as of my snapshot."

**How MySQL InnoDB implements MVCC:**
- InnoDB keeps the current row in the main table, plus older versions in a separate area called the **undo log**.
- When a transaction needs an older version of a row (because a newer version was written after its snapshot started), InnoDB reconstructs that version by applying undo log entries backward.

Both databases reach the same goal (non-blocking reads) with a different storage trick: Postgres keeps old versions inline in the table; InnoDB keeps old versions separately in the undo log and rebuilds them on demand.

**The cost of MVCC:** old row versions do not disappear by themselves. They pile up as **dead tuples** (in Postgres) once no active transaction's snapshot needs them anymore, but they still take up disk space and slow down scans until they are cleaned.

- **Postgres** uses a background process called **VACUUM** to scan tables, find dead tuples that no transaction can see anymore, and reclaim that space so it can be reused. If `VACUUM` falls behind (for example, on a table with heavy update traffic and a long-running transaction holding an old snapshot open), the table bloats, and both storage and query performance degrade. `VACUUM` also protects against **transaction ID wraparound**, an internal limit on how many transaction IDs Postgres can hand out before it must recycle them.
- **MySQL InnoDB** trims its undo log through a background **purge** thread once no transaction still needs those old versions. The same risk applies: a long-running transaction can keep old undo log entries around for a long time.

**Interview one-liner:** "MVCC means the database keeps several versions of a row so reads and writes do not block each other. The cost is that old versions must be cleaned up later — `VACUUM` in Postgres, purge in InnoDB."

## The Questions They Ask

**Q1: What is the difference between optimistic and pessimistic locking? When would you use each?**
Pessimistic locking takes a lock before doing any work (`SELECT ... FOR UPDATE`), so conflicts are prevented up front but other transactions must wait. Optimistic locking does no locking up front; it checks a version column at write time, and if the version has changed, it rejects the write and the app retries. Use pessimistic locking when conflicts are frequent and retries are expensive (for example, seat booking). Use optimistic locking when conflicts are rare and you want higher throughput (for example, a user editing their own profile, where two people rarely edit the same row at once). Follow-up: interviewers often ask you to write both. Practice the version-column `UPDATE ... WHERE version = ?` pattern and the `SELECT ... FOR UPDATE` pattern until they are automatic.

**Q2: How does MVCC work, and why is it good?**
The database keeps multiple versions of each row instead of updating in place. Each transaction reads from a consistent snapshot: the versions that were committed and visible at the time its snapshot started. This means a `SELECT` never has to wait for a concurrent `UPDATE`, and an `UPDATE` never has to wait for a concurrent `SELECT`. It is good because read-heavy and write-heavy workloads can run at the same time without one starving the other. The cost is that old versions must be cleaned up (`VACUUM` in Postgres, purge in InnoDB), and a long-running transaction can delay this cleanup.

**Q3: How do you prevent a deadlock?**
Main technique: always acquire locks on multiple rows/tables in the same, consistent order across all transactions (for example, always by ascending primary key). Also keep transactions short, avoid unnecessary `FOR UPDATE` locks, and set a `lock_timeout` so a stuck transaction fails fast instead of blocking forever. Note that even with all this, deadlocks can still happen occasionally in a busy system, so the application should always be ready to catch a deadlock error and retry the transaction.

**Q4: What does `SELECT ... FOR UPDATE` do? What is the difference from a plain `SELECT`?**
`SELECT ... FOR UPDATE` reads matching rows and takes an exclusive row lock on each of them, inside the current transaction. Any other transaction trying to `UPDATE`, `DELETE`, or run its own `SELECT ... FOR UPDATE` on those same rows must wait until this transaction commits or rolls back. A plain `SELECT` takes no lock (under MVCC, it just reads a consistent snapshot), so it never blocks and is never blocked by writers. Use `FOR UPDATE` when you plan to read a row and then write to it based on what you read, and you must stop anyone else from changing it in between (a classic case is "read current stock, then decrement it").

**Q5: What is the difference between a shared lock and an exclusive lock?**
A shared lock is for reading; many transactions can hold it on the same row at once. An exclusive lock is for writing; only one transaction can hold it, and it blocks every other lock (shared or exclusive) on that row. Follow-up: "does a plain read take a shared lock in Postgres?" Usually no — Postgres relies on MVCC snapshots for plain reads, so ordinary `SELECT` statements do not take row locks at all. Shared/exclusive row locks in Postgres mainly come from explicit statements like `SELECT ... FOR SHARE` and `SELECT ... FOR UPDATE`.

**Q6: What causes lock contention, and how do you reduce it?**
Lock contention happens when many transactions want conflicting locks on the same rows (or the same table, if locking is coarse), so they queue up waiting. Reduce it by using row-level locks instead of table-level locks, keeping transactions short, locking only the rows you actually need to change, using an index so the database does not lock more rows than necessary while scanning, and considering optimistic locking when conflicts are actually rare.

**Q7: Give a real example where you would choose `SELECT ... FOR UPDATE` over a version column.**
A seat-booking or inventory-decrement flow, where two users might try to grab the last seat or last unit at the same time. With a version column, both users would read the same version, both would try to write, and one would fail and need a full retry — under high contention this causes many wasted round trips. With `SELECT ... FOR UPDATE`, the second user simply waits a short moment for the first user's transaction to finish, then proceeds correctly against the updated row. Say out loud: "it depends on contention — for rare conflicts I would still prefer optimistic, because it avoids blocking; for frequent conflicts on a hot row, pessimistic locking gives more predictable behavior."

## Rapid-Fire

- **Shared lock**: for reading; many transactions can hold it together on the same row.
- **Exclusive lock**: for writing; only one transaction can hold it, blocks all other locks on that row.
- **Row-level lock**: locks one row; more concurrency. Used by Postgres/InnoDB for normal DML.
- **Table-level lock**: locks the whole table; used for schema changes or explicit `LOCK TABLE`.
- **Pessimistic locking**: lock first, then work. Good when conflicts are common.
- **Optimistic locking**: work first, check a version column at write time, retry on conflict. Good when conflicts are rare.
- **`SELECT ... FOR UPDATE`**: reads rows and takes an exclusive lock on them until commit/rollback.
- **Deadlock**: two transactions each wait on a lock the other holds; a cycle of waits.
- **Deadlock detection**: the database finds the wait cycle and aborts one transaction (the victim).
- **Deadlock avoidance**: always lock resources in the same order; keep transactions short; set `lock_timeout`.
- **MVCC**: the database keeps multiple row versions so readers and writers do not block each other.
- **Snapshot**: the set of committed row versions a transaction is allowed to see.
- **`xmin` / `xmax`**: Postgres system columns marking which transaction created/deleted a row version.
- **Undo log**: MySQL InnoDB's place to store older row versions for MVCC.
- **`VACUUM`**: Postgres process that cleans up dead (no-longer-visible) row versions.
- **Purge thread**: InnoDB's equivalent background cleanup for the undo log.
- **Dead tuple**: an old row version in Postgres that no active transaction can see anymore, awaiting cleanup.

## Common Traps & Mistakes

**Thinking a plain `SELECT` always takes a lock.** Under MVCC, ordinary reads in Postgres and InnoDB do not take row locks. They read a snapshot. Locks come from writes, or from explicit `FOR UPDATE` / `FOR SHARE`.

**Forgetting to check `rows affected` in optimistic locking.** The whole scheme depends on checking that the `UPDATE ... WHERE version = ?` actually changed a row. If the app ignores this and assumes success, a lost update can slip through silently.

**Using `SELECT ... FOR UPDATE` on far more rows than needed.** If the `WHERE` clause is not selective (for example, missing an index, so Postgres scans and locks many rows before filtering), you lock rows you never intended to touch, causing needless contention. Always check that the column in `WHERE` is indexed.

**Assuming optimistic locking has zero cost.** It removes lock-wait time, but every conflict becomes a full retry: re-read, re-compute, re-write. Under high contention, this can be slower and more complex than just taking a pessimistic lock. Choose based on actual measured conflict rate, not instinct.

**Confusing deadlock with a slow query or a normal lock wait.** A lock wait is one transaction waiting for another to finish; it resolves once the first transaction commits. A deadlock is a cycle of waits that will never resolve on its own — the database must intervene. If a transaction is "just slow," that is a performance issue, not a deadlock, and the fix is different (indexing, shorter transaction, less work per transaction).

**Believing MVCC removes the need for locks entirely.** MVCC handles the reader-vs-writer case. Writer-vs-writer conflicts on the same row still need locking (exclusive row locks), and application-level "read then decide then write" logic still needs either `FOR UPDATE` or a version column to be correct.

**Forgetting that `VACUUM` matters for correctness, not just cleanup.** If autovacuum falls badly behind (often because of one long-running transaction holding a very old snapshot open), table bloat grows, queries slow down, and in extreme cases Postgres can approach transaction ID wraparound, which is a serious operational problem. In an interview, mentioning "long transactions delay VACUUM" is a strong, specific point that shows real production experience.

**Assuming a consistent lock order fully eliminates deadlocks in practice.** It removes the most common cause, but deadlocks can still arise from index-level locks, foreign key checks, or unique-constraint checks that touch rows in an order the application does not directly control. The honest answer in an interview is: "consistent lock order removes most deadlocks, but the app should still handle the rare deadlock error with a retry."
