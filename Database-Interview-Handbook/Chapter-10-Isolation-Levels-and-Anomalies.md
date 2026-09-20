# Chapter 10 — Isolation Levels & Anomalies

This chapter is about what can go wrong when two transactions run at the same time. It
is one of the most common topics in database interviews, because it has a clean table of
facts you can be tested on. Chapter 9 covered the "I" in ACID at a high level. This
chapter goes deep into it.

## Key Concepts

### 1. Why isolation is a problem at all

A database rarely runs one transaction at a time. Many users read and write the same
rows at the same moment. **Isolation** is the rule that decides how much one transaction
can see of another transaction's unfinished work. If isolation is weak, transactions can
see strange, wrong, or half-done data from each other. These wrong views are called
**anomalies**.

There is a trade-off. Strong isolation avoids more anomalies. But it does this by
blocking transactions or making them retry more often. So strong isolation usually means
less concurrency (fewer transactions can run at full speed at the same time). This
trade-off is the main thing interviewers want you to explain.

### 2. The read anomalies

Each anomaly below uses two transactions, `T1` and `T2`, running on our HR schema
(`employees(emp_id, emp_name, dept_id, manager_id, salary, hire_date)`). We show them as
a timeline, step by step, in time order.

#### Dirty read

A **dirty read** happens when a transaction reads data that another transaction wrote
but has not committed yet. If that other transaction rolls back, the first transaction
has read data that never really existed.

```
Time  T1                                      T2
----  --------------------------------------  --------------------------------------
t1    UPDATE employees SET salary = 90000
      WHERE emp_id = 101;
      -- not committed yet
t2                                             SELECT salary FROM employees
                                                WHERE emp_id = 101;
                                                -- reads 90000 (dirty read!)
t3    ROLLBACK;
      -- salary is back to the old value,
      -- but T2 already used 90000
```

`T2` read a value that was never actually committed. This is the worst anomaly. Most
databases, including PostgreSQL, do not allow it in practice (more on this below).

#### Non-repeatable read

A **non-repeatable read** happens when a transaction reads the same row twice, and gets
two different values, because another transaction committed a change in between.

```
Time  T1                                      T2
----  --------------------------------------  --------------------------------------
t1    SELECT salary FROM employees
      WHERE emp_id = 101;
      -- reads 80000
t2                                             UPDATE employees SET salary = 90000
                                                WHERE emp_id = 101;
                                                COMMIT;
t3    SELECT salary FROM employees
      WHERE emp_id = 101;
      -- reads 90000 (different from t1!)
```

`T1` ran the exact same query twice in one transaction and got two different answers.
The row it already read "changed under its feet."

#### Phantom read

A **phantom read** happens when a transaction re-runs a query that selects a *set* of
rows, and a new row appears (or an old row disappears), because another transaction
inserted or deleted a matching row and committed.

```
Time  T1                                      T2
----  --------------------------------------  --------------------------------------
t1    SELECT COUNT(*) FROM employees
      WHERE dept_id = 10;
      -- reads 5 rows
t2                                             INSERT INTO employees
                                                (emp_name, dept_id, salary)
                                                VALUES ('New Hire', 10, 60000);
                                                COMMIT;
t3    SELECT COUNT(*) FROM employees
      WHERE dept_id = 10;
      -- reads 6 rows (a "phantom" row appeared!)
```

The difference from a non-repeatable read: non-repeatable read is about one row
changing value. Phantom read is about the *set* of rows matching a condition changing
size.

#### Lost update (briefly)

A **lost update** happens when two transactions both read the same row, then both write
back a new value based on what they read. The second write overwrites the first, and the
first transaction's change is silently lost.

```
Time  T1                                      T2
----  --------------------------------------  --------------------------------------
t1    SELECT salary FROM employees
      WHERE emp_id = 101;  -- reads 80000
t2                                             SELECT salary FROM employees
                                                WHERE emp_id = 101;  -- reads 80000
t3    UPDATE employees SET salary = 80000 + 5000
      WHERE emp_id = 101;  -- writes 85000
      COMMIT;
t4                                             UPDATE employees SET salary = 80000 + 2000
                                                WHERE emp_id = 101;  -- writes 82000
                                                COMMIT;
      -- T1's raise of 5000 is gone. Final salary is 82000, not 87000.
```

This matters a lot in real systems (for example, two requests both incrementing a
wallet balance). `SELECT ... FOR UPDATE`, or an atomic `UPDATE employees SET salary =
salary + 5000`, fixes it by not letting `T2` read the row until `T1` finishes.

#### Write skew (briefly)

**Write skew** happens when two transactions read overlapping data, then each writes to
a *different* row, and the two writes together break a rule that should hold across both
rows. Each write looks fine alone, but the combination is wrong.

Classic example: a rule says "at least one manager must be on call at all times." Two
managers, `T1` and `T2`, are both on call. Both transactions check "is at least one
other manager on call?", both see yes, and both then set themselves to off call. Now
zero managers are on call, breaking the rule. Neither transaction saw the other's write,
because both read before either wrote. `REPEATABLE READ` does not stop this. Only
`SERIALIZABLE` stops it.

Here is the same idea using our HR schema, with two managers who both plan to go off
call by setting a boolean-like `on_call` flag (imagine an extra column on `employees`
for this example):

```
Time  T1 (manager 101 goes off call)          T2 (manager 102 goes off call)
----  --------------------------------------  --------------------------------------
t1    SELECT COUNT(*) FROM employees
      WHERE manager_id IS NULL AND on_call = true;
      -- reads 2 (both 101 and 102 are on call)
t2                                             SELECT COUNT(*) FROM employees
                                                WHERE manager_id IS NULL AND on_call = true;
                                                -- also reads 2
t3    -- T1 sees 2 on call, thinks it is safe
      UPDATE employees SET on_call = false
      WHERE emp_id = 101;
      COMMIT;
t4                                             -- T2 also saw 2 on call, thinks it is safe
                                                UPDATE employees SET on_call = false
                                                WHERE emp_id = 102;
                                                COMMIT;
      -- Now 0 managers are on call. The rule "at least one" is broken.
```

Both updates touch different rows (`101` and `102`), so REPEATABLE READ has nothing to
block. Each transaction, taken alone, did exactly what it read and checked. The bug only
exists when you look at both transactions together. This is the hallmark of write skew,
and it is why it needs SERIALIZABLE, not just row-level snapshotting, to catch it.

### 3. The four standard SQL isolation levels

The ANSI SQL standard defines four isolation levels, from weakest to strongest:

1. **READ UNCOMMITTED** — a transaction can see uncommitted writes from other
   transactions. Allows dirty reads.
2. **READ COMMITTED** — a transaction only ever sees data that was committed at the
   moment it read it. Blocks dirty reads. Still allows non-repeatable reads and phantoms,
   because a second read in the same transaction can see newly committed changes.
3. **REPEATABLE READ** — once a transaction reads a row, it keeps seeing the same value
   for that row for the rest of the transaction. Blocks dirty reads and non-repeatable
   reads. By the ANSI standard, phantoms are still allowed.
4. **SERIALIZABLE** — the strongest level. The database makes transactions behave as if
   they ran one after another (serially), even though they really ran at the same time.
   Blocks all of the anomalies above, including write skew.

### 4. The classic table

This is the table interviewers expect you to draw from memory.

| Isolation Level | Dirty Read | Non-Repeatable Read | Phantom Read | Write Skew |
|---|:---:|:---:|:---:|:---:|
| READ UNCOMMITTED | Possible | Possible | Possible | Possible |
| READ COMMITTED | Prevented | Possible | Possible | Possible |
| REPEATABLE READ | Prevented | Prevented | Possible (standard) | Possible |
| SERIALIZABLE | Prevented | Prevented | Prevented | Prevented |

"Prevented" means the database will not let that anomaly happen at that level. "Possible"
means the standard allows it, though a specific database may prevent it anyway (see the
Postgres note next).

### 5. Real defaults: PostgreSQL and MySQL

This is where the standard and real databases split. Say this clearly in an interview:
**the ANSI standard describes minimum guarantees. A real database can do more.**

- **PostgreSQL's default is READ COMMITTED.** Most Postgres apps run at this level and
  never change it.
- **Postgres does not implement true READ UNCOMMITTED.** If you ask for it, Postgres
  silently gives you READ COMMITTED instead. So dirty reads are never possible in
  Postgres, at any level.
- **Postgres's REPEATABLE READ is stronger than the standard.** Postgres implements it
  using **snapshot isolation**: at the start of the transaction, it takes a consistent
  snapshot of the whole database, and the transaction reads only from that snapshot for
  its whole life. Because of this, Postgres REPEATABLE READ also blocks phantom reads.
  This is a very common interview trap — the standard says REPEATABLE READ allows
  phantoms, but Postgres's REPEATABLE READ does not.
- Postgres REPEATABLE READ still allows write skew. Only Postgres SERIALIZABLE blocks
  it (Postgres uses a technique called **SSI**, serializable snapshot isolation, which
  detects unsafe overlapping patterns and aborts one transaction).
- **MySQL (InnoDB) note:** InnoDB's default isolation level is **REPEATABLE READ**, not
  READ COMMITTED. InnoDB's REPEATABLE READ also blocks most phantom reads for plain
  `SELECT`, using a similar snapshot mechanism, plus special locks called **next-key
  locks** for `SELECT ... FOR UPDATE`. This is different from Postgres's default, so if
  an interviewer asks "what changes if this were MySQL," mention the different default.

### 6. How to set the isolation level in SQL

```sql
-- Set for one transaction only (most common in interviews and in practice)
BEGIN;
SET TRANSACTION ISOLATION LEVEL REPEATABLE READ;
-- ... your queries ...
COMMIT;

-- Or in one line
BEGIN ISOLATION LEVEL SERIALIZABLE;
-- ... your queries ...
COMMIT;

-- Set the default for the whole session
SET SESSION CHARACTERISTICS AS TRANSACTION ISOLATION LEVEL SERIALIZABLE;
```

If you use `SERIALIZABLE` in Postgres, your application code must be ready to retry a
transaction. Postgres can abort a serializable transaction with a **serialization
failure** error, even if nothing looks obviously wrong, because it detected an unsafe
pattern like write skew. Retrying the transaction is expected and normal. The error code
to catch, if you want to be precise in an interview, is SQLSTATE `40001`.

### Where MVCC fits in

Postgres does almost all of this using **MVCC** (multi-version concurrency control),
covered fully in Chapter 11. In short: instead of blocking readers, Postgres keeps
several versions of each row, tagged with the transaction ID that created them. A
transaction only sees row versions that were already committed at the right point in
time. This is *how* READ COMMITTED and REPEATABLE READ manage to avoid dirty reads and
non-repeatable reads without heavy locking on reads. Locking (Chapter 11) is still used
for writes, to stop two transactions from changing the same row at the same time.

### 7. The trade-off: isolation vs. concurrency

Higher isolation gives you fewer anomalies, but it costs concurrency in one of two ways:

- **More blocking.** A stricter level may make one transaction wait for another to
  finish, instead of letting both run freely.
- **More retries.** Postgres SERIALIZABLE does not block much. Instead, it lets
  transactions run, then aborts one if it finds a conflict. Your application must catch
  this and retry.

The interview answer to "which level should I use?" is always **"it depends."** Say this
out loud, then reason:

- Most web apps use **READ COMMITTED** (the Postgres default) for everyday reads and
  writes. It is fast and good enough for most screens.
- Use **REPEATABLE READ** when you need a stable, consistent view for a report, or a
  multi-step read-then-write flow like transferring money between two rows.
- Use **SERIALIZABLE** only for the few operations where correctness must be airtight
  (for example, seat booking, or the "at least one manager on call" rule), and you are
  willing to pay for retries.

## The Questions They Ask

**Q1: Name the anomalies that can happen when transactions run at the same time.**
Dirty read, non-repeatable read, phantom read, lost update, and write skew. Define each
in one sentence: dirty read is reading uncommitted data; non-repeatable read is a single
row changing value between two reads in the same transaction; phantom read is the set of
matching rows changing between two reads; lost update is one transaction's write being
silently overwritten by another; write skew is two transactions each writing a different
row, based on stale reads, breaking a rule that spans both rows.
*Follow-up: which of these can READ COMMITTED prevent?* Only the dirty read.

**Q2: Draw the isolation level table.**
Draw the 4x4 table from Key Concepts, section 4. Say the standard's rule out loud level
by level, from READ UNCOMMITTED (prevents nothing) up to SERIALIZABLE (prevents
everything). Then add the Postgres note: Postgres's REPEATABLE READ also blocks phantom
reads, which is stronger than the ANSI standard requires.

**Q3: What is the default isolation level in PostgreSQL? In MySQL?**
PostgreSQL defaults to READ COMMITTED. MySQL (InnoDB) defaults to REPEATABLE READ. This
difference matters because code that runs correctly on MySQL's default might see more
anomalies if moved to Postgres's default without change, or vice versa for phantom reads.

**Q4: What is the difference between REPEATABLE READ and SERIALIZABLE?**
REPEATABLE READ (in Postgres) gives each transaction one consistent snapshot of the data
for its whole life, so re-reading a row or a query's result set never changes within
that transaction. But two transactions can still each write to different rows based on
what they saw, and together break a cross-row rule — this is write skew.
SERIALIZABLE removes even that risk. Postgres does this by watching for unsafe read/write
patterns between concurrent transactions and aborting one of them with a serialization
error, forcing a retry. So SERIALIZABLE gives the same result as if the transactions had
run one at a time, in some order, with no exceptions.
*Follow-up: what must your application do differently at SERIALIZABLE?* It must catch
serialization failure errors and retry the whole transaction.

**Q5: What is snapshot isolation, and how does it relate to REPEATABLE READ?**
Snapshot isolation means a transaction reads from a fixed, consistent snapshot of the
database, taken at the moment the transaction starts. Postgres implements its
REPEATABLE READ level using snapshot isolation. This is why Postgres's REPEATABLE READ
is stronger than the ANSI minimum — it blocks phantom reads too, as a side effect of
using one snapshot for the whole transaction.

**Q6: Give a real example of a lost update, and how to prevent it.**
Example: two requests both read a product's stock count, both subtract one, and both
write back. One decrement is lost, and stock count is wrong. Prevent it with an atomic
update (`UPDATE products SET stock = stock - 1 WHERE product_id = 5`), or with
`SELECT ... FOR UPDATE` to lock the row before making the decision, or by using
SERIALIZABLE and retrying on conflict.

**Q7: Why doesn't REPEATABLE READ stop write skew?**
Because REPEATABLE READ only protects the rows a transaction actually reads or writes
directly — it keeps them consistent for that one transaction. It has no idea about a
business rule that spans two different rows written by two different transactions. Each
transaction's own read and write are internally fine. The violation only shows up when
you combine both transactions' effects, which REPEATABLE READ never checks.

**Q8: If Postgres's REPEATABLE READ already blocks phantom reads, why would anyone use
SERIALIZABLE at all?**
Because REPEATABLE READ only guarantees a consistent snapshot for reads. It does not
check whether two transactions' writes, taken together, break a rule that spans more
than one row — that is write skew, covered in section 2. SERIALIZABLE adds a safety net
on top: Postgres tracks the pattern of reads and writes across concurrent transactions,
and if it finds a pattern that could not have happened in any true one-at-a-time order,
it aborts one of the transactions. Use SERIALIZABLE when a business rule spans multiple
rows, such as "at least one manager on call," "do not double-book this seat," or "the
sum of these two accounts must not go negative."

**Q9: How would you test that your code is safe under a given isolation level?**
Open two `psql` sessions (or two database connections in a script). Start a transaction
in each. Interleave the statements by hand, in the order you want to test — for example,
both `SELECT`, then both `UPDATE`, then both `COMMIT` — and check the final data. This
manual two-session test is exactly the timeline style used in this chapter, and it is
also a fair thing to describe out loud in an interview if asked "how would you verify
this."

## Rapid-Fire

- **Dirty read?** Reading another transaction's uncommitted write.
- **Non-repeatable read?** Same row, read twice, two different values, in one transaction.
- **Phantom read?** Same query, run twice, different number of matching rows.
- **Lost update?** Two writes based on stale reads; one write silently disappears.
- **Write skew?** Two transactions each write a different row; combined result breaks a rule.
- **Postgres default isolation level?** READ COMMITTED.
- **MySQL (InnoDB) default isolation level?** REPEATABLE READ.
- **Does Postgres support true READ UNCOMMITTED?** No, it silently behaves like READ COMMITTED.
- **Does Postgres REPEATABLE READ block phantom reads?** Yes, because it uses snapshot isolation.
- **Which level blocks write skew?** Only SERIALIZABLE.
- **What must the app do at SERIALIZABLE?** Retry on serialization failure errors.
- **General trade-off?** Higher isolation = fewer anomalies, but more blocking or more retries.
- **How to set isolation level for one transaction?** `SET TRANSACTION ISOLATION LEVEL ...` right after `BEGIN`.

## Common Traps & Mistakes

- **Saying "REPEATABLE READ allows phantom reads" without naming the database.** This is
  true for the ANSI standard, but false for Postgres's actual REPEATABLE READ. Always say
  which database you mean.
- **Mixing up non-repeatable read and phantom read.** Non-repeatable read is about one
  row's *value* changing. Phantom read is about the *number of rows* matching a
  condition changing. Keep the two-transaction examples straight in your head.
- **Forgetting write skew exists.** Many candidates only prepare the three read anomalies
  and lost update. Write skew is a favorite follow-up because it shows whether you truly
  understand why SERIALIZABLE is different from REPEATABLE READ.
- **Assuming higher isolation is always better.** It is safer, but it costs performance.
  A senior answer always mentions the concurrency trade-off, not just the correctness
  side.
- **Thinking isolation levels are only about `SELECT`.** They also affect what locks
  `UPDATE`, `DELETE`, and `SELECT ... FOR UPDATE` take, and how long other transactions
  wait.
- **Forgetting that SERIALIZABLE in Postgres can abort transactions that look correct.**
  It aborts based on a detected *pattern* of conflict, not because any single statement
  was wrong. Application code must retry, not treat the error as a bug.
- **Confusing isolation level with locking strategy.** Isolation level is a guarantee
  about what a transaction can see. Locking (see Chapter 11) is one of the mechanisms
  used to deliver that guarantee, but not the only one — Postgres mostly uses MVCC
  (multi-version concurrency control) and snapshots, not blocking locks, for reads.
- **Not knowing that isolation level is set per-transaction, not fixed forever.** You can
  run most of your application at READ COMMITTED and switch to SERIALIZABLE for just the
  one function that does seat booking or money transfer. It is not an all-or-nothing,
  database-wide setting.
- **Treating a serialization failure as a rare edge case you can ignore.** Under real
  concurrent load, SERIALIZABLE transactions can fail often enough that skipping retry
  logic will cause visible errors for users. Always wrap SERIALIZABLE transactions in a
  retry loop with a small limit, such as three attempts.
