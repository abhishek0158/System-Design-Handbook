# Chapter 7 — Deadlocks

Interviewers ask about deadlocks because they show up in real production incidents, and because
the fix is a simple rule that is easy to forget. This chapter covers what a deadlock is, why it
happens, how the database handles it, and how you stop it from happening.

## The Idea

A **deadlock** is a stuck situation. Transaction A holds a lock that transaction B wants.
Transaction B holds a lock that transaction A wants. Neither one will let go until it gets what
it is waiting for. So both wait forever, and neither ever finishes.

A **lock** is a hold a transaction takes on a row (or table) so no other transaction can change it
at the same time. See Chapter 5 for how locks work in general. Deadlocks mostly happen when you
use **pessimistic locking** — locking rows up front with `SELECT ... FOR UPDATE`, as covered in
Chapter 6. If two transactions grab their locks in different orders, they can trap each other.

### Tiny example

Two rows: `order_id = 1` and `order_id = 2`. Two transactions run at the same time.

```sql
-- Transaction A
BEGIN;
UPDATE orders SET status = 'shipped' WHERE order_id = 1;  -- locks row 1
-- ... A now wants row 2 next ...
UPDATE orders SET status = 'shipped' WHERE order_id = 2;  -- waits, B holds row 2
```

```sql
-- Transaction B
BEGIN;
UPDATE orders SET status = 'shipped' WHERE order_id = 2;  -- locks row 2
-- ... B now wants row 1 next ...
UPDATE orders SET status = 'shipped' WHERE order_id = 1;  -- waits, A holds row 1
```

A locks row 1, then asks for row 2. B locks row 2, then asks for row 1. Each one is waiting for a
lock the other one is holding. Neither will finish, so neither will release its lock. This is a
**circular wait** — A waits on B, and B waits on A, forming a loop.

## Why It Matters — the Reasoning

**The root cause is the lock order, not the locking itself.** Locking rows is normal and safe. The
problem only appears when different parts of your code lock the *same* rows in *different* order.
If every transaction always locked row 1 before row 2, this deadlock could never happen — one
transaction would simply wait its turn for row 1, get it, finish, and move on to row 2. The wait
would be a straight line, not a loop. So the real bug is not "we used locks." The real bug is
"we did not agree on one order to take them in."

**Why the database cannot just avoid this by being clever.** The database does not know in advance
which rows your code will touch next. Transaction A's next statement could ask for any row —
the database finds out only when it happens. So it cannot always stop a deadlock before it starts.
Instead, it waits for a short time, checks if the transactions waiting on locks form a cycle, and
if they do, it picks one transaction as the **victim** and forcibly kills it (rolls it back). That
frees up its locks, so the other transaction can continue. The killed transaction gets an error
back and should retry.

**Why the app, not the database, must prevent most deadlocks.** The database can only detect and
break deadlocks after they happen — it cannot stop your code from writing lock order differently in
two different places. Prevention is a coding discipline: always acquire locks in the same order,
everywhere in your codebase. This is the single biggest fix, and it is cheap once you know the
rule.

**The other side — shorter transactions, fewer locked rows.** Even with a consistent lock order,
deadlocks (and plain lock waiting) get worse the longer a transaction holds its locks, and the more
rows it touches. A transaction that locks 50 rows has 50 chances to collide with another
transaction's lock. A transaction that finishes in 2 milliseconds has almost no time window to
collide with anyone. So the practical advice is: keep transactions short (do not do slow work like
calling an external API inside a transaction), and touch as few rows as you can. This does not
remove the need for consistent lock ordering, but it shrinks how often collisions occur at all.

**Retry is not optional — it is part of the design.** Because the database resolves a deadlock by
killing a transaction, that transaction's work is thrown away. The application must treat this as
an expected, recoverable error — not a bug — and simply retry the transaction from the start. A
deadlock error is the database telling you "try again, this was not your fault."

**Why detection takes a moment, not zero time.** The database does not know a deadlock exists the
instant it happens. It has to wait until both transactions are actually blocked — sitting idle,
waiting on a lock — and only then can it walk the wait graph and see the loop. This means there is
always a small delay between the moment the deadlock forms and the moment the victim gets killed.
In Postgres this delay is controlled by the `deadlock_timeout` setting (1 second by default): a
transaction that has been waiting on a lock for longer than this triggers a deadlock check. This is
also why deadlocks feel rare in light traffic but show up more under load — more concurrent
transactions means more chances for two of them to grab the same two rows in opposite order at the
same time.

## Common Interview Questions

**1. What is a deadlock?**
Two transactions each hold a lock the other one needs. Both wait for the other to finish and let
go, so both wait forever. The database has to step in and forcibly resolve it.

**2. Give a concrete example.**
Transaction A locks row 1, then wants row 2. Transaction B locks row 2, then wants row 1. Both
now wait on each other. Say it out loud with rows, not just "two transactions" — interviewers want
to see you can trace the opposite order clearly.

**3. How does the database resolve a deadlock?**
It runs a **deadlock detector**. Postgres periodically checks the transactions that are waiting on
locks, and builds a "who is waiting for whom" graph. If it finds a cycle (a loop), it knows this is
a deadlock, not just normal waiting. It picks one transaction — usually the one that will be
cheapest to undo, or simply the one that got stuck last — and rolls it back with an error. The
other transaction's lock is now free, so it can go on and commit.

**4. How do you prevent deadlocks in your code?**
Main fix: **always lock resources in the same order**, everywhere in the codebase. For example, if
you ever update two rows in one transaction, always sort by `order_id` and lock the lower ID
first. This turns a possible circular wait into a simple queue — no loops, no deadlocks. Secondary
fixes: keep transactions short (do less work between `BEGIN` and `COMMIT`), touch fewer rows per
transaction, and use the lowest isolation level and lock strength your logic actually needs (see
Chapters 4 and 6).

**5. What should the application do when it gets a deadlock error?**
Catch the specific error (Postgres returns SQLSTATE `40P01` for deadlock detected), and retry the
whole transaction from the beginning, usually after a short random delay (called **backoff**) so
the same two transactions do not just collide again immediately. Do not show this as a hard
failure to the user — it is a normal, expected event under concurrency, and a fresh retry almost
always succeeds because the conflicting transaction has already gone away by then.

**6. Deadlock vs. livelock — what is the difference?**
A deadlock: both sides are stuck and stop moving entirely. A **livelock** is different: both sides
keep being active — retrying, backing off, retrying again — but still make no real progress,
often because they keep colliding with each other again and again. Deadlock is a frozen photo;
livelock is a hamster wheel. Both need the same underlying fix: change the pattern (lock order, or
retry timing) so the collision stops repeating.

## Quick Recall

- A deadlock = a circular wait. A waits for a lock B holds, B waits for a lock A holds. Neither
  ever finishes.
- Classic example: A locks row 1 then wants row 2; B locks row 2 then wants row 1 — opposite
  order.
- The database detects the cycle and kills one transaction (the victim), which gets an error and
  must retry.
- Prevention: always lock rows in the same order everywhere, keep transactions short, touch fewer
  rows.
- Deadlocks mostly come from pessimistic locking (Chapter 6) on top of row locks (Chapter 5).
- **Biggest gotcha:** the bug is never "we used locks" — it is "we locked the same rows in
  different orders in different places." Fix the order, and the deadlock disappears.
