# Chapter 3 — Transactions & ACID

Interviewers ask about transactions because most real bugs in production come from someone
skipping one. This chapter covers what a transaction is, what ACID means, and why the database
needs all four guarantees.

## The Idea

A **transaction** is a group of database steps that must all succeed, or all fail, together.
There is no middle state. You start one with `BEGIN`, and end it with either `COMMIT` (keep all
changes) or `ROLLBACK` (undo all changes).

**ACID** is four guarantees a database gives you for every transaction:

- **Atomicity** — all steps in the transaction happen, or none of them happen. No partial work.
- **Consistency** — the transaction cannot leave the data in a state that breaks your rules
  (constraints like `NOT NULL`, `FOREIGN KEY`, `CHECK`). See Chapter 1 for constraints.
- **Isolation** — two transactions running at the same time do not see each other's half-finished
  work. The database gives each transaction the illusion that it runs alone. Chapters 4 and 5
  cover this in full detail — isolation levels, anomalies, locks, and MVCC.
- **Durability** — once a transaction commits, the data survives even if the server crashes right
  after. Postgres does this with a **write-ahead log (WAL)**: it writes the change to a log file on
  disk before saying "done," so it can replay the log after a crash.

### Tiny example

```sql
BEGIN;

UPDATE accounts SET balance = balance - 100 WHERE account_id = 1;
UPDATE accounts SET balance = balance + 100 WHERE account_id = 2;

COMMIT;
```

If both updates succeed, `COMMIT` makes them permanent together. If something goes wrong midway
(say, account 2 does not exist and the second update fails), you run `ROLLBACK` instead, and the
first update is undone too. Account 1 never loses its 100, even though its update alone had
already "worked."

```sql
BEGIN;
UPDATE accounts SET balance = balance - 100 WHERE account_id = 1;
-- something fails here
ROLLBACK;  -- account 1's balance is back to what it was
```

## Why It Matters — the Reasoning

**Why transactions exist:** think about a money transfer. You debit account A, then credit
account B. These are two separate SQL statements. What if the debit succeeds, and then the
server crashes, or the second statement fails because of a network blip, before the credit runs?
Without a transaction, the money just vanishes — account A is down 100, account B never got it.
The bank's total money is now wrong, and there is no clean way to know where the app was in the
middle of that operation.

A transaction fixes this by making the two updates one atomic unit. Either both happen, or
neither happens. The database, not your application code, guarantees this — even across crashes,
because of durability (the WAL) and atomicity working together.

**The cost side — correctness vs. concurrency:** giving each transaction the illusion that it runs
alone (isolation) is not free. To stop transactions from seeing each other's half-done work, the
database must use locks or keep multiple versions of a row. This means:

- More isolation (stricter rules) usually means more locking, or more work checking for
  conflicts.
- More locking means transactions wait for each other more often, so fewer transactions run at the
  same time — lower **concurrency** (throughput).
- Less isolation lets transactions run faster and in parallel, but opens the door to **anomalies**
  — one transaction reading or acting on another transaction's incomplete data.

So there is a trade-off: correctness (strict isolation) vs. speed (high concurrency). You do not
always need the strictest isolation — it depends on what your business logic can tolerate. This
exact trade-off, and the specific anomalies each isolation level allows, is the topic of
Chapter 4. Chapter 5 explains the actual mechanism (locks and MVCC) Postgres uses to give you
isolation without stopping all transactions dead in their tracks.

## Common Interview Questions

**1. Explain ACID with an example.**
Use the money transfer: debit account A, credit account B, inside one transaction.
*Atomicity* — both updates commit, or both roll back; money is never half-moved.
*Consistency* — a `CHECK (balance >= 0)` constraint stops the transaction from ever leaving an
account negative.
*Isolation* — another transaction reading account A's balance never sees a state where A is
debited but B is not yet credited.
*Durability* — once the transfer commits, it survives a crash a second later, because Postgres
already wrote it to the WAL on disk.

**2. Give a real case that needs a transaction.**
Any operation touching more than one row or more than one table where they must stay in sync.
Examples: placing an order (insert into `orders`, decrease `stock` in `products`), signing up a
user (insert into `users`, insert a row into `wallets` with a starting balance). If the second
step fails and the first is not undone, you get orphaned or inconsistent data.

**3. What happens if the database crashes in the middle of a transaction?**
If the transaction never reached `COMMIT`, Postgres treats it as if it never happened. On restart,
it uses the WAL to figure out which transactions were committed before the crash and replays only
those. Anything mid-flight, not committed, is rolled back automatically. This is durability and
atomicity working together — you never end up with a half-applied transaction after a crash.

**4. Is a single SQL statement a transaction?**
Yes. In Postgres, every standalone SQL statement runs inside its own implicit transaction, even if
you never type `BEGIN`. For example, a single `UPDATE` that changes 1,000 rows either updates all
1,000 or none — it is still atomic. You only need explicit `BEGIN ... COMMIT` when you want
*multiple* statements to succeed or fail as one unit.

**5. Which letter of ACID does isolation-level tuning affect?**
Isolation, obviously by name — but it is worth explaining the ripple effect. Changing the
isolation level (Chapter 4) changes what one transaction can see of another's uncommitted or
concurrently committed work. It does not touch atomicity or durability — a transaction still
either fully commits or fully rolls back, and committed data still survives a crash, no matter
what isolation level you picked. Isolation level tuning is really a concurrency vs. correctness
knob, layered on top of the other three guarantees, which stay fixed.

**6. Does consistency mean the same thing as in CAP theorem?**
No, and interviewers like to check this. ACID's "Consistency" means your data always obeys the
constraints and rules you defined (foreign keys, checks, uniqueness). CAP's "Consistency" means
all nodes in a distributed system see the same data at the same time. They share a name, not a
meaning. Do not mix them up.

## Quick Recall

- A transaction = a group of steps that succeed or fail together. `BEGIN`, then `COMMIT` or
  `ROLLBACK`.
- Atomicity = all-or-nothing. Consistency = rules stay valid. Isolation = concurrent transactions
  do not see each other's half-done work. Durability = committed data survives a crash, via the
  WAL.
- Money transfer is the classic example for why transactions exist: two updates that must not
  happen halfway.
- A single SQL statement is already an implicit transaction in Postgres.
- **Biggest gotcha:** isolation is not free — stricter isolation means more locking and less
  concurrency. This trade-off is the whole subject of Chapter 4.
