# Chapter 4 — Isolation Levels & Anomalies

When many transactions run at the same time, they can step on each other's data. Isolation levels
control how much one transaction can see of another's unfinished work. Interviewers ask this to
check if you understand the trade-off between correctness and speed.

## The Idea

A **transaction** is a group of database operations that run as one unit (see Chapter 3). When
two transactions run at the same time, strange things can happen if the database lets them see
each other's half-done changes. These strange things are called **anomalies**.

There are three main read anomalies:

**1. Dirty read** — Transaction A reads data that Transaction B has changed but not yet
committed. If B later rolls back, A has read data that never really existed.

```
Time   T1 (reader)                    T2 (writer)
1                                      UPDATE accounts SET balance = 0
                                       WHERE id = 1;   -- not committed yet
2      SELECT balance FROM accounts
       WHERE id = 1;  -- reads 0 (dirty!)
3                                      ROLLBACK;        -- balance is back to 100
```
T1 saw a balance of 0, but that change never actually happened. That is a dirty read.

**2. Non-repeatable read** — Transaction A reads the same row twice, and gets two different
answers, because Transaction B committed a change in between.

```
Time   T1 (reader)                    T2 (writer)
1      SELECT balance FROM accounts
       WHERE id = 1;  -- reads 100
2                                      UPDATE accounts SET balance = 50
                                       WHERE id = 1;
                                       COMMIT;
3      SELECT balance FROM accounts
       WHERE id = 1;  -- reads 50 (different!)
```
Same transaction, same row, two different values. That breaks the idea that a transaction should
see a stable snapshot of data.

**3. Phantom read** — Transaction A runs the same query twice, and the second time, new rows
appear (or old rows vanish), because Transaction B inserted or deleted rows in between.

```
Time   T1 (reader)                              T2 (writer)
1      SELECT COUNT(*) FROM orders
       WHERE status = 'PENDING';  -- returns 5
2                                                INSERT INTO orders (..., status)
                                                  VALUES (..., 'PENDING');
                                                  COMMIT;
3      SELECT COUNT(*) FROM orders
       WHERE status = 'PENDING';  -- returns 6 (phantom row!)
```
The difference from a non-repeatable read: here whole *rows* appear or disappear, not just a
value inside one row changing.

There is a fourth problem worth a quick mention: **lost update**. Two transactions read the same
row, both calculate a new value, and both write it back. One of the writes silently overwrites
the other, so an update is lost. Example: two transactions both read `balance = 100`, both add 10,
both write `110` — but the correct answer after both updates should be `120`. This is less about
"reading" and more about **write conflicts**, and it needs extra care (locks or optimistic checks)
even at higher isolation levels. Chapter 6 covers this in detail.

## The Four Isolation Levels

An **isolation level** is a setting that tells the database how much protection to give against
these anomalies. The SQL standard defines four levels, from loosest to strictest:

1. **READ UNCOMMITTED** — A transaction can see uncommitted changes from other transactions.
   Allows dirty reads. Very few databases actually let dirty reads happen in practice (PostgreSQL
   treats this level the same as READ COMMITTED).
2. **READ COMMITTED** — A transaction only ever sees data that has been committed. No dirty
   reads. But if you read the same row twice, you might see different values, because another
   transaction could commit a change in between.
3. **REPEATABLE READ** — A transaction takes a **consistent snapshot** at the start and keeps
   seeing that same snapshot for its whole duration. Reading the same row twice always gives the
   same value. No dirty reads, no non-repeatable reads. Phantom reads are still possible in the
   standard definition, though PostgreSQL's implementation also blocks most phantom reads using
   snapshots.
4. **SERIALIZABLE** — The strictest level. Transactions behave as if they ran one after another
   (serially), even though they actually run at the same time. Blocks all three anomalies,
   including phantom reads.

### The Table — Which Level Prevents Which Anomaly

| Isolation Level    | Dirty Read | Non-Repeatable Read | Phantom Read |
|---------------------|:----------:|:--------------------:|:-------------:|
| READ UNCOMMITTED    | Possible   | Possible             | Possible      |
| READ COMMITTED      | Prevented  | Possible             | Possible      |
| REPEATABLE READ     | Prevented  | Prevented            | Possible*     |
| SERIALIZABLE        | Prevented  | Prevented            | Prevented     |

\* In the SQL standard, REPEATABLE READ still allows phantom reads. In practice, PostgreSQL's
REPEATABLE READ (built on snapshots) blocks most phantom reads too, but the standard does not
guarantee this for every database. MySQL InnoDB's REPEATABLE READ also blocks most phantoms using
similar snapshot tricks plus gap locks. Always check the real behaviour of the database you use,
not just the label.

**Real-world defaults, in one line:** PostgreSQL defaults to **READ COMMITTED**; MySQL InnoDB
defaults to **REPEATABLE READ**.

## Why It Matters — the Reasoning

This is the key trade-off: **higher isolation gives you more correctness, but less concurrency.**

To stop transactions from seeing each other's half-done work, the database has to do extra work —
hold locks longer, keep more old versions of rows around, or check for conflicts and abort
transactions that clash. All of that costs time and can make transactions wait for each other or
fail and retry. So as you climb from READ UNCOMMITTED to SERIALIZABLE, you get stronger
correctness guarantees, but the database allows fewer transactions to run smoothly in parallel.

**Why READ COMMITTED is the common default:** For most everyday applications (a web app placing
orders, updating a user profile), dirty reads are the anomaly you truly cannot tolerate — reading
data that turns out to be fake is a serious bug. But non-repeatable reads and phantom reads are
often fine, because each individual SQL statement still sees a correct, committed snapshot at the
moment it runs. Most application code reads a row, does something, and moves on — it rarely needs
the same row to stay frozen for the whole transaction. So READ COMMITTED gives good protection for
the common case, while still letting many transactions run concurrently. That is why PostgreSQL
and most default database configurations pick it.

**Why you rarely need SERIALIZABLE:** SERIALIZABLE is the safest level — it removes every
anomaly, and code written for it is the easiest to reason about (as if nothing else runs at the
same time). But that safety has a real cost: it needs more aggressive locking or conflict
detection, so transactions wait more, or the database aborts and retries transactions after they
clash. This hurts throughput, especially under high load. Most business logic can tolerate a bit
of raciness, or can be fixed with a targeted tool (a `SELECT ... FOR UPDATE` lock, a unique
constraint, or an optimistic version check — see Chapter 6) instead of paying the SERIALIZABLE
cost for every transaction in the system. Use SERIALIZABLE only for the few operations where a
subtle race condition would cause real damage — for example, moving money between accounts, or
enforcing a business rule that depends on counting rows (like "only 10 seats left").

In short: **pick the loosest isolation level that still protects the specific anomaly your
business logic cannot survive.** Do not reach for SERIALIZABLE everywhere "to be safe" — that is
usually a performance mistake, not a correctness win.

## Common Interview Questions

**1. Name the three main read anomalies. Can you give a one-line example of each?**
Dirty read (reading uncommitted data that might vanish on rollback), non-repeatable read (same
row gives different values on re-read within one transaction), and phantom read (same query
returns a different set of rows on re-read, because rows were inserted or deleted). The reasoning
interviewers want: each anomaly happens because one transaction is peeking at another
transaction's changes at a different level of "commitment" — uncommitted data, updated data, or
newly inserted/deleted rows.

**2. Draw the isolation-level table. Which level stops which anomaly?**
Draw the table above: READ UNCOMMITTED stops nothing, READ COMMITTED stops dirty reads,
REPEATABLE READ also stops non-repeatable reads, and SERIALIZABLE stops all three including
phantom reads. The reasoning: each level is a superset of protection over the one below it — you
never lose a guarantee by going up a level, you only gain more of them (at a concurrency cost).

**3. What is the default isolation level in PostgreSQL and in MySQL?**
PostgreSQL defaults to READ COMMITTED. MySQL InnoDB defaults to REPEATABLE READ. The reasoning
behind the difference: PostgreSQL leans towards giving good concurrency out of the box, while
MySQL's InnoDB engine was designed to make REPEATABLE READ affordable using snapshots plus gap
locks, so it made that the safer default.

**4. When would you actually reach for SERIALIZABLE?**
When a race condition would cause real, hard-to-detect damage — for example, double-booking the
same seat, double-spending the same funds, or violating a business rule that depends on counting
rows across the whole table (like enforcing a maximum inventory count). The reasoning: SERIALIZABLE
is worth its performance cost only when the alternative is a bug that is rare, silent, and
expensive to fix after the fact.

**5. What is the difference between REPEATABLE READ and SERIALIZABLE?**
REPEATABLE READ guarantees your own transaction sees a stable snapshot — the rows you already
read will not change value under you. But in the standard, other transactions can still insert new
rows that match your query filter (a phantom), and true write-write conflicts across concurrent
transactions are not fully controlled. SERIALIZABLE goes further: the database guarantees the
overall result is *as if* all transactions ran one at a time, one after another, with no
overlapping effects at all — no phantoms, no subtle ordering issues. The reasoning: REPEATABLE
READ protects *your own reads*; SERIALIZABLE protects the *whole interaction* between all
concurrent transactions.

**6. How do you actually set the isolation level in SQL?**
You set it per transaction (recommended), or per session:
```sql
BEGIN;
SET TRANSACTION ISOLATION LEVEL REPEATABLE READ;
-- ... your queries ...
COMMIT;
```
The reasoning to mention: set it as narrowly as possible — per transaction, not globally for the
whole application — so that only the operations that truly need the stronger guarantee pay its
performance cost.

## Quick Recall

- Three read anomalies: **dirty read** (uncommitted data), **non-repeatable read** (same row
  changes value), **phantom read** (same query returns different rows). Lost update is a related
  write-conflict problem, covered more in Chapter 6.
- Four levels, loosest to strictest: **READ UNCOMMITTED → READ COMMITTED → REPEATABLE READ →
  SERIALIZABLE**. Each higher level prevents more anomalies but allows less concurrency.
- Defaults: **PostgreSQL = READ COMMITTED**, **MySQL InnoDB = REPEATABLE READ**.
- SERIALIZABLE is the safest, not the "always correct choice" — its cost in waiting and retries
  means you should reserve it for the few operations where a race condition would really hurt.
- **Biggest gotcha:** isolation level is a trade-off knob, not a "more is always better" setting.
  The right interview answer is always "it depends on what anomaly the business logic cannot
  survive," not "just use SERIALIZABLE everywhere."
