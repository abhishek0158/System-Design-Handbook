# Chapter 6 — Optimistic vs Pessimistic Locking

Two people can try to update the same row at the same time. This chapter is about two ways to
handle that. Interviewers ask this to check if you can reason about trade-offs, not just recite
definitions.

## The Idea

**Pessimistic locking** means: lock the row first, before you touch it. No one else can change
that row until you finish and release the lock. In PostgreSQL, you do this with
`SELECT ... FOR UPDATE`.

Example: two clerks try to book the last seat on a flight.

```sql
BEGIN;
SELECT seats_left FROM flights WHERE flight_id = 101 FOR UPDATE;
-- seats_left = 1, so we can book it
UPDATE flights SET seats_left = seats_left - 1 WHERE flight_id = 101;
COMMIT;
```

The `FOR UPDATE` locks the row. A second clerk's `SELECT ... FOR UPDATE` on the same row must
wait until the first clerk commits or rolls back. This stops both clerks from booking the same
seat.

**Optimistic locking** means: do NOT lock. Read the row like normal. But the row carries a
**version number** (a counter that goes up by one on every update) or a timestamp. When you write
back, you check that the version is still the same one you read. If it is, your update goes
through, and you bump the version. If it is not — someone else changed the row in between — your
`UPDATE` matches zero rows. You detect this (rows affected = 0) and retry: read the row again,
redo your logic, try the update again.

Tiny example with a version column:

```sql
-- table: accounts(account_id, balance, version)

-- Step 1: read
SELECT balance, version FROM accounts WHERE account_id = 55;
-- suppose balance = 1000, version = 3

-- Step 2: write, checking the version has not moved
UPDATE accounts
SET balance = balance - 200, version = version + 1
WHERE account_id = 55 AND version = 3;
```

If another transaction updated this row after our `SELECT`, its version is now 4, not 3. Our
`UPDATE` matches 0 rows. Our application code sees 0 rows affected, knows there was a conflict,
and retries the whole read-modify-write step.

## Why It Matters — the Reasoning

This is a trade-off, and the right choice depends on **how often two people are likely to touch
the same row.**

**Pessimistic locking is good when conflicts are likely, or a retry is not acceptable.** Think of
the flight-seat example: many clerks may race for the last seat. You want to be sure, right away,
that only one booking wins. Locking guarantees this with no retry logic needed in the application.

But locking has a cost. While one transaction holds the lock, every other transaction that wants
the same row must **wait**. If many transactions want the same row often, they queue up, and
throughput drops. Worse, if two transactions each hold a lock the other one needs, you get a
**deadlock** (both wait forever) — we cover this in Chapter 7. Pessimistic locking also ties up a
database connection and a transaction for longer, which is expensive at scale.

**Optimistic locking is good when conflicts are rare, and most operations are reads.** Think of a
user editing their own profile. Two people almost never edit the exact same profile row at the
exact same second. So why pay the cost of locking on every read? With optimistic locking, reads
never block anyone. You only pay a cost — a retry — in the rare case there really was a conflict.
This gives you much better concurrency (many transactions running at once) when contention (two
transactions wanting the same row) is low.

The catch: optimistic locking pushes work onto your application. You must write retry logic. And
if conflicts are actually common (**high contention**), optimistic locking becomes bad — you keep
retrying, wasting work, and under very high contention it can even be slower than just locking
once. So:

- **High contention or correctness cannot tolerate any retry loop** → pessimistic locking.
- **Low contention, many reads, want high concurrency** → optimistic locking.

This connects to Chapter 5 (Locks & MVCC): pessimistic locking uses real database locks.
Optimistic locking often runs happily under MVCC (multi-version concurrency control), because
MVCC already lets readers proceed without blocking — optimistic locking just adds the version
check at write time to catch the rare conflict.

## Common Interview Questions

**Q1: What is the difference between optimistic and pessimistic locking, and when would you use
each?**
Pessimistic locking locks the row up front (`SELECT ... FOR UPDATE`), so no one else can touch it
until you are done. Optimistic locking does not lock; it checks a version number at write time and
retries on conflict. Use pessimistic when conflicts are likely or a retry loop is not acceptable
(for example, booking the last seat). Use optimistic when conflicts are rare and you want high
read concurrency (for example, editing a user profile). The reasoning: locking always costs
something (blocking); only pay that cost when you expect to need it.

**Q2: How does optimistic locking actually work? Walk me through it.**
The row has a version column (or an updated-at timestamp). You read the row along with its
version. You do your business logic in the application. When you write back, your `UPDATE`
statement includes `WHERE id = ? AND version = ?` — the version you read earlier. If no one else
changed the row, the version still matches, so the update succeeds, and you increment the version.
If someone else updated the row in between, the version has already changed, so your `WHERE`
clause matches nothing, and your `UPDATE` affects 0 rows. That is your signal to retry.

**Q3: What happens exactly when there is a conflict in optimistic locking?**
Your `UPDATE ... WHERE id = ? AND version = ?` returns "0 rows affected" instead of an error. Your
application code must check this (most frameworks throw an "optimistic lock exception" for you).
On conflict, you re-read the row (getting the new version and new data), redo your logic on the
fresh data, and try the update again. This is why optimistic locking needs retry logic in the
application — the database will not do this for you.

**Q4: Would you use optimistic or pessimistic locking for a high-contention counter (for example,
counting page views)? What about a rarely-edited user profile?**
For a high-contention counter, many transactions want to update the same row constantly. Optimistic
locking would cause endless retries, wasting work — bad fit. Pessimistic locking would cause
long queues of waiting transactions — also not great, at true high scale you would rather use an
atomic increment (`UPDATE counter SET value = value + 1`) which does not need read-then-write at
all, avoiding the conflict problem entirely. For a rarely-edited user profile, conflicts almost
never happen, so optimistic locking is the better fit: no locking cost on the common case (just
reading and displaying the profile), and the rare conflict is cheap to retry.

**Q5: Is optimistic locking really "locking"? It does not take a lock, so why is it called
that?**
It is called locking because the *goal* is the same as pessimistic locking: prevent one
transaction from silently overwriting another transaction's change (a "lost update"). The
*method* is different — instead of blocking others with a real lock, it detects the conflict
after the fact using the version check, and relies on the application to retry. So it protects
data correctness without ever blocking a reader or writer. Some people prefer the term "optimistic
concurrency control" for this reason.

**Q6: Can you combine both, or do you have to pick one?**
Yes, you can mix them in the same system. Use optimistic locking as the default for most tables
(since most rows in most apps are rarely contested), and use pessimistic locking (`FOR UPDATE`)
for specific hot spots where you know contention is high or correctness cannot risk a retry loop,
such as a shared inventory counter or a seat-booking row. Choosing per use case, not globally, is
often the best answer.

## Quick Recall

- Pessimistic = lock row up front (`SELECT ... FOR UPDATE`); blocks others; safe but can cause
  waiting or deadlocks (Ch7).
- Optimistic = no lock; check a version/timestamp at update time (`WHERE id=? AND version=?`);
  0 rows affected means conflict — retry.
- Choose based on contention: likely conflicts → pessimistic; rare conflicts + many reads →
  optimistic.
- Optimistic locking pushes retry logic onto the application; it is a poor fit under very high
  contention.
- **Biggest gotcha:** forgetting to check "rows affected" after an optimistic `UPDATE`. If you
  ignore it, you silently accept a lost update instead of catching the conflict — this defeats the
  whole point of optimistic locking.
