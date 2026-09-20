# Chapter 9 — Transactions & ACID

A **transaction** is a group of database steps that must all succeed, or all fail together.
Interviewers ask about transactions because almost every real backend feature — payments,
orders, signups — needs one. This chapter covers ACID, and the `COMMIT` / `ROLLBACK` /
`SAVEPOINT` commands you use to control a transaction.

*New table for this chapter:* to show a money transfer, we add a small `accounts` table:
`accounts(account_id PK, owner_name, balance)`. The shared schemas (HR, E-commerce) do not
have a "balance" concept, so this one extra table makes the classic example realistic.

## Key Concepts

### What is a transaction?

A transaction is one unit of work made of one or more SQL statements. The database treats
the whole unit as a single step. Either every statement in it takes effect, or none do.

Think of transferring money between two bank accounts. This needs two steps:
1. Subtract 100 from Account A.
2. Add 100 to Account B.

If the database applies step 1 and then crashes before step 2, money disappears. A
transaction stops this. It tells the database: "run both steps as one unit, or run neither."

In PostgreSQL, you start a transaction with `BEGIN`, run your statements, then end it with
`COMMIT` (keep the changes) or `ROLLBACK` (undo the changes):

```sql
BEGIN;

UPDATE accounts SET balance = balance - 100 WHERE account_id = 1;
UPDATE accounts SET balance = balance + 100 WHERE account_id = 2;

COMMIT;
```

If anything goes wrong before `COMMIT` — a crash, an error, or you calling `ROLLBACK` — both
updates are undone. The accounts look as if the transaction never ran.

### ACID, one letter at a time

ACID is a set of four guarantees a database gives you for every transaction. Each letter
answers a different question. We will use the money transfer example for all four.

#### A — Atomicity ("all or nothing")

**Atomicity** means a transaction's statements either all apply, or none apply. There is no
state where only half the transaction happened.

In our transfer, atomicity means you can never end up with the money removed from Account A
but not added to Account B. If the debit succeeds but the credit fails (say, Account B does
not exist), the database undoes the debit too. Money is never created or lost mid-transaction.

```sql
BEGIN;

UPDATE accounts SET balance = balance - 100 WHERE account_id = 1;
-- suppose this next line fails because account_id 999 does not exist
UPDATE accounts SET balance = balance + 100 WHERE account_id = 999;

ROLLBACK;  -- undoes the first UPDATE too; Account A's balance is unchanged
```

Atomicity is the "all or nothing" part of ACID. It says nothing about correctness of the
final values or about other transactions running at the same time — those are the next two
letters.

#### C — Consistency ("rules stay valid")

**Consistency** means a transaction can only move the database from one valid state to
another valid state. "Valid" means all rules you defined still hold: primary keys, foreign
keys, `CHECK` constraints, `UNIQUE` constraints, and your own application logic.

Example: add a rule that a balance can never go negative.

```sql
ALTER TABLE accounts ADD CONSTRAINT balance_non_negative CHECK (balance >= 0);
```

Now, if a transaction tries to debit more money than an account has, the `CHECK` constraint
fails. The database rejects that statement and the whole transaction rolls back. The
database never saves a negative balance, so it never becomes "inconsistent" with your rule.

Consistency is partly enforced by the database (constraints, keys) and partly by your
application code (business rules the database does not know about, like "a discount code can
only be used once per customer"). The database guarantees the part it knows about.

#### I — Isolation ("concurrent transactions do not corrupt each other")

**Isolation** means that when many transactions run at the same time, each one behaves as if
it were running alone. One transaction should not see half-finished, temporary changes made
by another transaction.

Example: two transfers run at the same time on Account A, which starts with 500.
- Transaction 1: transfer 100 out of Account A.
- Transaction 2: transfer 200 out of Account A.

Without good isolation, both transactions could read the starting balance of 500 at the same
time, both compute "500 minus my amount," and one update could overwrite the other. The
account could end up with the wrong final balance, even though each transaction looked
correct on its own.

Isolation is the deepest and trickiest letter of ACID. There is a full spectrum of isolation
levels (Read Uncommitted, Read Committed, Repeatable Read, Serializable), each allowing or
blocking different anomalies (dirty reads, non-repeatable reads, phantom reads). We cover
this in detail in **Chapter 10 (Isolation Levels & Anomalies)** and how the database
implements it in **Chapter 11 (Locking, Concurrency & MVCC)**. For this chapter, just
remember: isolation is the guarantee that concurrent transactions do not step on each other.

#### D — Durability ("committed data survives a crash")

**Durability** means once a transaction commits, its changes are permanent. Even if the
server loses power one second after `COMMIT` returns, the data is still there when it
restarts.

How does the database do this? Before it changes the actual data files on disk, it first
writes a record of the change to a **write-ahead log (WAL)** — a sequential, append-only
file on disk. `COMMIT` does not return success until this log record is safely on disk. If
the server crashes right after, it replays the WAL on restart and reconstructs the committed
changes, even though the main data files were not fully updated yet. We explain the WAL and
how storage engines use it in full in **Chapter 12 (Storage Engines: B-tree, LSM & WAL)**.

For now, the key point for interviews: durability is not "trust the OS cache." It means the
change is on **durable storage** (disk, not just RAM) before the database tells you it
succeeded.

### COMMIT, ROLLBACK, and SAVEPOINT

- **`COMMIT`** ends the transaction and makes all its changes permanent and visible to other
  transactions.
- **`ROLLBACK`** ends the transaction and undoes all its changes, as if none of them happened.
- **`SAVEPOINT`** marks a point inside a transaction that you can roll back to, without
  undoing the whole transaction.

`SAVEPOINT` is useful when part of a transaction might fail, but you still want to keep the
earlier successful part. Example: place an order, then try to apply a discount code. If the
discount code is invalid, cancel just that part, not the whole order.

```sql
BEGIN;

INSERT INTO orders (order_id, customer_id, order_date, status, total_amount)
VALUES (5001, 42, CURRENT_DATE, 'pending', 250.00);

SAVEPOINT before_discount;

UPDATE orders SET total_amount = total_amount - 1000  -- bad discount, would go negative
WHERE order_id = 5001;

-- this fails a CHECK constraint (total_amount >= 0), so we roll back just this part
ROLLBACK TO SAVEPOINT before_discount;

COMMIT;  -- the order insert is still kept; the bad discount update is undone
```

A `SAVEPOINT` is like a checkpoint inside a longer transaction. You can have several
savepoints, and `ROLLBACK TO SAVEPOINT name` undoes everything after that named point, while
keeping everything before it.

### Autocommit

**Autocommit** is a mode where every single SQL statement is automatically wrapped in its own
transaction and committed right away, unless you explicitly start a transaction with `BEGIN`.

Most database clients (`psql`, your JDBC connection, your ORM) run in autocommit mode by
default. This means:

```sql
UPDATE accounts SET balance = balance - 100 WHERE account_id = 1;
-- this alone commits immediately, as its own tiny transaction
```

is very different from:

```sql
BEGIN;
UPDATE accounts SET balance = balance - 100 WHERE account_id = 1;
UPDATE accounts SET balance = balance + 100 WHERE account_id = 2;
COMMIT;
```

If you write two related `UPDATE` statements without `BEGIN`, and autocommit is on, each
`UPDATE` is its own transaction. If the second one fails, the first one is already
permanently committed. This is a common bug: developers assume statements are grouped, but
without an explicit `BEGIN`, they are not.

In Java with JDBC, this maps to `connection.setAutoCommit(false)` before running related
statements, then `connection.commit()` or `connection.rollback()` at the end. Spring's
`@Transactional` annotation does this for you around a method.

### The classic example: money transfer

This is the textbook case for why transactions exist, and interviewers like to ask it
directly. Transferring money between two accounts needs at least two writes (debit one
account, credit another). These two writes must succeed or fail as one unit.

```sql
BEGIN;

-- Step 1: debit the sender, but only if they have enough balance
UPDATE accounts
SET balance = balance - 100
WHERE account_id = 1 AND balance >= 100;

-- Step 2: check the debit actually happened (row count from step 1, checked in app code)
-- if 0 rows were updated, the sender did not have enough balance — ROLLBACK here

-- Step 3: credit the receiver
UPDATE accounts
SET balance = balance + 100
WHERE account_id = 2;

COMMIT;
```

Note the `WHERE balance >= 100` guard. This stops the balance from going negative even before
the `CHECK` constraint would catch it, and it lets your application code check "did this
actually update a row?" before moving on. In real code, you would check the update's affected
row count, and call `ROLLBACK` yourself if it is zero.

Without a transaction, if your application crashes or the network drops between step 1 and
step 3, Account A has lost 100 but Account B never received it. The money is gone. This is
the single clearest reason transactions exist.

## The Questions They Ask

**Q1: Explain ACID with an example.**
Use the money transfer example, one letter at a time, as done above. A strong answer says:
"Atomicity means the debit and credit either both happen or neither happens. Consistency
means the balance never violates rules like 'balance cannot go negative.' Isolation means
two transfers happening at the same time do not read each other's half-finished state.
Durability means once the transfer commits, it survives a crash right after." Naming a
concrete rule or scenario for each letter, instead of just defining the word, is what
separates a strong answer from a memorized one.

**Q2: What happens if the database crashes in the middle of a transaction?**
On restart, the database looks at its write-ahead log (WAL). Any transaction that had not
committed before the crash is treated as if it never happened — its partial changes are
discarded (or simply never applied to the main data files). Any transaction that *had*
committed, even if its changes were not yet fully written to the main data files, is
recovered by replaying its WAL records. This process is called **crash recovery**. The
result: after restart, the database only contains fully committed transactions, and none of
them are partially applied. This is atomicity and durability working together — atomicity
throws away the incomplete transaction, and durability guarantees the completed one is not
lost. We go deeper into WAL and recovery in Chapter 12.
*Follow-up: "What if the crash happens after COMMIT returns success but before the OS writes
to disk?"* Answer: it should not be possible. `COMMIT` should not return "success" to your
application until the WAL record is confirmed flushed to durable storage. If it returns
early, durability is broken — this is a real setting to know about (`synchronous_commit` in
PostgreSQL, `fsync`), and turning it off trades durability for speed.

**Q3: Give a real case that needs a transaction.**
Give more than the money transfer, since interviewers sometimes ask for a "different"
example to check you understand the pattern, not just memorized one case:
- **Placing an order** (Schema B): insert a row into `orders`, insert one or more rows into
  `order_items`, and reduce stock in `products`. All three must succeed together, or the
  customer is charged for an order that was never recorded properly.
- **Seat booking**: check a seat is free, then mark it booked. Must be one transaction (with
  proper isolation) so two customers cannot both book the same seat.
- **Employee transfer between departments** (Schema A): update `employees.dept_id`, and
  insert an audit row into a history table. Both must happen, or your audit trail is wrong.

**Q4 (follow-up): Does every multi-statement operation need a transaction?**
No. If the statements are independent and a partial failure is acceptable or easy to recover
from, you do not need one. Wrapping read-only reporting queries in a transaction, for
example, is unnecessary overhead. Use a transaction when partial completion would leave the
data in a state that breaks a business rule or loses information.

**Q5 (follow-up): What is the difference between a `ROLLBACK` and an application-level retry?**
`ROLLBACK` undoes changes inside the database for that specific transaction. A retry is an
application decision to attempt the whole operation again after a failure (for example,
after a deadlock or serialization error in Chapter 10/11). Many production systems `ROLLBACK`
first, then check the error, then decide whether to retry the whole transaction from the
start.

## Rapid-Fire

- **What is a transaction?** A group of SQL statements executed as a single all-or-nothing
  unit.
- **What does the "A" in ACID stand for?** Atomicity — all statements in the transaction
  succeed, or none do.
- **What does "C" stand for?** Consistency — the database only moves between states that obey
  all defined rules (constraints, keys).
- **What does "I" stand for?** Isolation — concurrent transactions do not see each other's
  incomplete work.
- **What does "D" stand for?** Durability — once committed, data survives a crash.
- **What ends a transaction and keeps its changes?** `COMMIT`.
- **What ends a transaction and undoes its changes?** `ROLLBACK`.
- **What is a `SAVEPOINT`?** A named point inside a transaction you can roll back to, without
  losing earlier work in the same transaction.
- **What is autocommit?** A mode where every statement is its own transaction, committed
  right away, unless you call `BEGIN` first.
- **How does the database guarantee durability?** By writing changes to the write-ahead log
  (WAL) on disk before confirming `COMMIT` (details in Chapter 12).
- **Do constraints get checked during a transaction or only at COMMIT?** By default, most
  constraints (`CHECK`, `NOT NULL`, `UNIQUE`) are checked immediately, statement by statement.
  Foreign keys can be declared `DEFERRABLE` to check only at `COMMIT`.
- **Can you `ROLLBACK` after `COMMIT`?** No. Once committed, the changes are permanent; you
  would need a new transaction to reverse them.

## Common Traps & Mistakes

- **Forgetting `BEGIN`.** Running two related `UPDATE` statements without wrapping them in a
  transaction means each one commits on its own (autocommit). If the second fails, the first
  is still permanently applied. Always wrap related writes in `BEGIN` / `COMMIT`.

- **Thinking Atomicity and Isolation are the same thing.** Atomicity is about one
  transaction's own steps (all or nothing). Isolation is about two *different* transactions
  running at the same time and not interfering with each other. A transaction can be
  perfectly atomic and still corrupt data if isolation is weak.

- **Thinking Consistency is only a database feature.** The database enforces the rules you
  declare (constraints, keys). It cannot enforce business rules it does not know about, like
  "a coupon code can only be redeemed once." Your application code must enforce those, often
  using a `UNIQUE` constraint or a transaction to make the check-and-use atomic.

- **Assuming `ROLLBACK` undoes everything in every database.** In PostgreSQL, DDL statements
  (`CREATE TABLE`, `ALTER TABLE`) are transactional and can be rolled back. **MySQL note:**
  in MySQL, most DDL statements cause an implicit commit — you cannot roll them back inside a
  transaction. If your interview mentions MySQL, mention this difference.

- **Leaving a transaction open too long.** A transaction that stays open (uncommitted) while
  waiting on a slow external call, or waiting on user input, holds locks and can block other
  transactions. Keep transactions short: do the slow work outside the `BEGIN`/`COMMIT` block
  where possible. This connects to lock contention, covered in Chapter 11.

- **Ignoring the "aborted transaction" state in PostgreSQL.** If any statement inside a
  transaction fails (say, a constraint violation), PostgreSQL puts the whole transaction into
  an aborted state. Every statement after that fails too, with "current transaction is
  aborted," until you call `ROLLBACK` (or `ROLLBACK TO SAVEPOINT`). New developers often see
  this error and are confused, because they expect only the failed statement to be rejected.

- **Confusing "the app got a success response" with "durability."** If your application code
  crashes right after calling `COMMIT`, but before it can act on the acknowledgment (say,
  send a confirmation email), the transaction is still durably committed. The data is safe.
  The follow-up action (the email) is a separate concern, not covered by ACID.

- **Believing a transaction protects you from a bug in your own SQL logic.** A transaction
  guarantees all-or-nothing execution of the statements you send. It does not check that your
  business logic is correct. If your transfer logic forgets to check the sender's balance, a
  transaction will happily and atomically commit an incorrect result.
