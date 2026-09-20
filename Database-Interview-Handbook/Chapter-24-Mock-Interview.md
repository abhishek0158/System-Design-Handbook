# Chapter 24 — Mock Interview

This is the capstone. It shows one full database interview from start to end. The candidate is a backend engineer with about 3–4 years of experience. The round runs about 45 minutes. You will read the real turns of talk, marked **Interviewer:** and **Candidate:**. Between the turns you will see short **> Commentary:** notes. These notes explain why an answer is strong, what a weak answer looks like, and what the interviewer is really testing.

Read this chapter last. It ties together the SQL round (Chapters 1–7), the internals round (Chapters 8–15), and the design round (Chapters 18–21). The goal is not to memorize the answers. The goal is to see the *shape* of a good interview: say your assumptions out loud, reason about trade-offs, and state the "it depends" clearly.

## How this round is scored

Most companies score a database interview on four areas. Keep these in your head while you read.

| Area | What the interviewer watches for |
|---|---|
| **SQL** | Can you write correct queries live? Do you reach for the right tool (joins, aggregation, window functions)? |
| **Internals** | Do you understand *why* the database behaves the way it does — indexes, ACID, isolation, MVCC, replication? |
| **Design** | Can you turn vague needs into a schema, defend your keys, and solve concurrency and scale problems? |
| **Communication** | Do you ask before you assume? Do you narrate your thinking? Do you handle a mistake without panic? |

A good candidate does not need a perfect answer on every point. A good candidate reasons well, recovers from small mistakes, and knows the trade-offs. That is what earns a "hire."

---

## The transcript

### Part 1 — Warm-up SQL (about 10 minutes)

**Interviewer:** Hi, thanks for joining. Let's start easy. We have the standard HR tables: `departments(dept_id, dept_name)` and `employees(emp_id, emp_name, dept_id, manager_id, salary, hire_date)`. Can you find the highest-paid employee in each department?

**Candidate:** Sure. Quick question first — if two employees in a department share the same top salary, do you want both of them, or just one?

> **Commentary:** Strong opening. The candidate asked about ties *before* writing any SQL. Ties are the exact twist the interviewer usually adds next. Asking early shows the candidate already sees the edge case. A weak candidate writes code first and gets surprised later.

**Interviewer:** Good question. For now, just give me one row per department. We will talk about ties in a moment.

**Candidate:** Okay. One row per department, and I do not care which one wins a tie. I will use a window function. I will rank employees inside each department by salary, then keep rank 1.

```sql
SELECT emp_id, emp_name, dept_id, salary
FROM (
    SELECT emp_id, emp_name, dept_id, salary,
           ROW_NUMBER() OVER (PARTITION BY dept_id ORDER BY salary DESC) AS rn
    FROM employees
) ranked
WHERE rn = 1;
```

I used `ROW_NUMBER()` on purpose. It gives exactly one row per department, even on a tie, because it never repeats a number inside a partition.

> **Commentary:** The candidate picked `ROW_NUMBER()` and said *why*. This links straight to Chapter 5: `ROW_NUMBER` breaks ties, `RANK`/`DENSE_RANK` keep them. Naming the reason is what turns a correct answer into a strong answer.

**Interviewer:** Nice. Could you also do this without a window function?

**Candidate:** Yes, a couple of ways. The classic one is a correlated subquery on the max salary:

```sql
SELECT e.emp_id, e.emp_name, e.dept_id, e.salary
FROM employees e
WHERE e.salary = (
    SELECT MAX(e2.salary)
    FROM employees e2
    WHERE e2.dept_id = e.dept_id
);
```

But note this version returns *all* tied top earners, not one row per department. So it answers a slightly different question. If a department has two people at the top salary, both show up.

> **Commentary:** The candidate did not just show a second solution. They pointed out that the two solutions behave differently on ties. That awareness is exactly what the next twist tests. This is a subtle point many candidates miss — they think the two queries are equal.

**Interviewer:** That is the twist I wanted. Now I *do* want to handle ties. Give me every employee who has the top salary in their department.

**Candidate:** Then I switch from `ROW_NUMBER()` to `RANK()`. `RANK()` gives the same rank to tied rows, so all top earners get rank 1.

```sql
SELECT emp_id, emp_name, dept_id, salary
FROM (
    SELECT emp_id, emp_name, dept_id, salary,
           RANK() OVER (PARTITION BY dept_id ORDER BY salary DESC) AS rnk
    FROM employees
) ranked
WHERE rnk = 1;
```

The correlated subquery above also works for this case. I would pick the window version if I also needed the second or third place, because it scans the table once.

**Interviewer:** Good, you read my mind. Now give me the *second* highest salary in each department.

**Candidate:** Here I need to be careful about what "second highest" means when there are ties. Two readings:

1. The second *distinct* salary. If a department pays 100, 100, 90, then second highest is 90.
2. The second *row* by salary. Then the second 100 would count as second.

I will assume you mean the second distinct salary — that is the more common interview meaning. For that I use `DENSE_RANK()`, because it does not skip numbers after a tie.

```sql
SELECT emp_id, emp_name, dept_id, salary
FROM (
    SELECT emp_id, emp_name, dept_id, salary,
           DENSE_RANK() OVER (PARTITION BY dept_id ORDER BY salary DESC) AS drnk
    FROM employees
) ranked
WHERE drnk = 2;
```

If you wanted the second distinct salary as a single value per department, and not the people, I would instead group and rank the distinct salaries. But this version also returns *who* earns it, which is usually what is wanted.

> **Commentary:** This is the high point of the SQL round. The candidate spotted that "second highest" is ambiguous with ties, named both readings, chose one, and said why `DENSE_RANK` fits. This is the difference between the three ranking functions from Chapter 5, applied under pressure. A weak candidate writes `WHERE rnk = 2` with plain `RANK()` and silently returns wrong rows when a tie exists at the top.

**Interviewer:** Exactly right. Let's move on.

---

### Part 2 — Internals grilling (about 15 minutes)

**Interviewer:** Here is a query that runs on a large `orders` table — millions of rows. It is slow.

```sql
SELECT * FROM orders WHERE customer_id = 4213 ORDER BY order_date DESC;
```

Why might it be slow, and what would you do?

**Candidate:** First, let me say what I would actually do before guessing: I would run `EXPLAIN ANALYZE` on it to see the real plan. But let me reason about the likely cause.

If there is no index on `customer_id`, the database must do a **sequential scan** — read every row in the table and check the filter. On millions of rows that is slow. The fix is an index on `customer_id`. That turns it into an **index scan**: the database jumps to the matching rows instead of reading all of them.

But there is a second cost here — the `ORDER BY order_date DESC`. Even after the index finds the customer's rows, the database still has to sort them. So I would use a **composite index**:

```sql
CREATE INDEX idx_orders_customer_date
ON orders (customer_id, order_date DESC);
```

Now the index gives the rows already filtered by customer *and* already in date order. The sort step disappears. This is a covering-ish idea from Chapter 8 — order the index columns as equality-first (`customer_id`), then the range/sort column (`order_date`).

> **Commentary:** Three strong moves here. First, the candidate said they would measure with `EXPLAIN ANALYZE` before guessing (Chapter 7). Second, they named the exact plan change: sequential scan → index scan. Third, they saw the *second* cost — the sort — and picked a composite index in the right column order. A weak answer stops at "add an index on customer_id" and forgets the sort.

**Interviewer:** One catch — the query is `SELECT *`. Does your index fully solve it?

**Candidate:** Good point. No, not fully. My index has only `customer_id` and `order_date`. Because the query asks for all columns with `SELECT *`, the database still has to go back to the table to fetch the other columns. That extra step is a **heap fetch**. So the sort is gone, but the row lookups remain.

Two options. One, in real code I would select only the columns I need, not `*`. Two, if this query is very hot, I could add an **INDEX INCLUDE** to store the extra columns in the index itself, so it becomes a covering index and skips the heap. But that makes the index bigger and writes slower, so I would only do it if the query really needs it.

> **Commentary:** The interviewer probed with a follow-up meant to catch overconfidence. The candidate did not get defensive. They accepted the gap, explained the heap fetch, and offered a measured fix with its cost. Handling a follow-up like this — calmly, correctly — scores as high as the first answer.

**Interviewer:** Let's switch to transactions. Explain ACID in your own words.

**Candidate:** ACID is four promises a database makes about a transaction — a transaction being a group of statements that should act as one unit.

- **Atomicity** — all or nothing. Either every statement in the transaction commits, or none does. If it fails halfway, it rolls back.
- **Consistency** — the transaction moves the database from one valid state to another. It never breaks rules like constraints or foreign keys.
- **Isolation** — concurrent transactions do not step on each other. The result should look as if they ran in some order, even when they run at the same time.
- **Durability** — once committed, the data survives a crash. This is why databases use a write-ahead log — the change is written to a durable log before commit is confirmed.

> **Commentary:** Clean and correct (Chapter 9). The candidate tied durability to the write-ahead log without being asked, which shows they understand the *mechanism*, not just the word. Interviewers love when the acronym is backed by "how."

**Interviewer:** Isolation is the interesting one. Walk me through the isolation levels and which anomaly each one prevents.

**Candidate:** There are four standard levels. Each higher level prevents one more read anomaly, and costs more in concurrency.

| Level | Dirty read | Non-repeatable read | Phantom read |
|---|---|---|---|
| Read Uncommitted | possible | possible | possible |
| Read Committed | prevented | possible | possible |
| Repeatable Read | prevented | prevented | possible* |
| Serializable | prevented | prevented | prevented |

Quick definitions:

- A **dirty read** is reading another transaction's uncommitted change, which might roll back.
- A **non-repeatable read** is reading the same row twice and getting different values, because someone committed an update in between.
- A **phantom read** is running the same filter twice and getting new rows that appeared in between.

One Postgres note, since you use Postgres: Postgres does not really have Read Uncommitted — it behaves as Read Committed. And Postgres Repeatable Read is implemented with snapshots, so it already blocks most phantoms, which is why I put an asterisk. The strict SQL standard allows phantoms at Repeatable Read; Postgres is stronger.

> **Commentary:** The table plus the plain definitions is exactly Chapter 10. The Postgres-specific note is a bonus — it shows the candidate knows the standard *and* the real engine. That asterisk is honest: it flags where the standard and Postgres differ, instead of pretending the table is universal.

**Interviewer:** You mentioned snapshots. How does MVCC let readers not block writers?

**Candidate:** MVCC means Multi-Version Concurrency Control. The idea: the database keeps more than one version of a row.

When a transaction updates a row, Postgres does not overwrite the old row in place. It writes a **new version** of the row and marks the old version as still valid for transactions that started earlier. Each transaction reads from a **snapshot** — a view of the data as of the moment it started (or the moment each statement started, depending on level).

So a reader looks at the version that was valid for its snapshot. A writer creates a new version. They do not fight over the same copy. That is why **readers do not block writers, and writers do not block readers**. Locks are still needed when two writers touch the same row — that part still serializes.

The cost is that old versions pile up. That is what **VACUUM** cleans up in Postgres — it removes dead row versions no transaction can still see.

> **Commentary:** This is Chapter 11 delivered well. The candidate explained the mechanism (new version, snapshot), stated the payoff (readers and writers do not block), *and* named the trade-off (dead tuples, VACUUM). Mentioning the cost unprompted is a senior signal. A weak answer says only "readers don't block writers" like a slogan, with no idea why.

**Interviewer:** Two transactions read the same account balance of 100, both add 50, and both write 150. The final balance is 150, but it should be 200. What is this, and how do you stop it?

**Candidate:** That is a **lost update**. One transaction's write is silently overwritten by another, because both read the old value first. Read Committed does not stop it, because each read was of committed data — the problem is the read-modify-write gap.

There are two clean fixes.

**Pessimistic locking.** Lock the row when you read it, so the second transaction waits:

```sql
SELECT balance FROM accounts WHERE account_id = 1 FOR UPDATE;
-- compute new balance, then
UPDATE accounts SET balance = balance + 50 WHERE account_id = 1;
```

The `FOR UPDATE` makes the second transaction block until the first commits, then it reads the fresh 150 and writes 200.

**Optimistic locking.** Do not lock. Add a `version` column, and only update if the version has not changed:

```sql
UPDATE accounts
SET balance = 150, version = version + 1
WHERE account_id = 1 AND version = 5;
```

If another transaction already bumped the version, this update affects zero rows. The app sees that, retries with the fresh value, and wins the second time.

I would pick optimistic locking when conflicts are rare — it avoids holding locks and scales better. I would pick pessimistic when conflicts are common, so I do not waste work on retries. And the simplest fix of all: for a plain increment, do the math inside SQL — `SET balance = balance + 50` — so there is no read-modify-write gap in the first place.

> **Commentary:** The candidate named the anomaly correctly (lost update, Chapter 10), then gave both the pessimistic and optimistic fix *with the rule for choosing between them*. The final tip — push the arithmetic into the UPDATE so there is no gap — is the answer a real engineer reaches for. Naming the trade-off (rare conflicts → optimistic, frequent → pessimistic) is what lifts this above a textbook reply.

**Interviewer:** And if two transactions each hold a lock the other wants?

**Candidate:** That is a **deadlock** — a cycle of waiting. Postgres detects it automatically. It picks one transaction as the victim, aborts it with an error, and lets the other proceed. My job in the application is to catch that error and retry the aborted transaction. To *reduce* deadlocks, I make all transactions acquire locks in the same order — for example, always lock the lower account id first in a transfer. Consistent lock order breaks the cycle before it forms.

> **Commentary:** Short, correct, complete (Chapter 11). The candidate knew the database resolves the deadlock itself, that the app must retry, and gave the standard prevention: consistent lock ordering. That last point is the practical knowledge interviewers dig for.

**Interviewer:** Suppose reads are growing fast. How would you scale reads, and what problem does that introduce?

**Candidate:** The first move is **read replicas**. I keep one primary for writes, and add one or more replicas that copy the primary's data. Then I route read queries to the replicas and writes to the primary. This spreads read load across many machines.

The problem it introduces is **replication lag**. The replica is a little behind the primary, because the changes take time to travel and apply. So a user might write something, then read from a replica that has not caught up yet, and not see their own change. That is a **read-your-own-writes** problem.

A few ways to handle it. One, route reads that must be fresh — like right after a write — to the primary. Two, use "sticky" routing so a user who just wrote reads from the primary for a short window. Three, if the platform supports it, wait for the replica to reach the write's log position before reading. The general rule from Chapter 13: async replication gives you speed but eventual consistency; sync replication gives you freshness but higher write latency. You pick based on how much staleness the feature can tolerate.

> **Commentary:** Full marks for structure: the fix (replicas), the new problem (lag), the concrete mitigations, and the underlying trade-off (async vs sync). This mirrors Chapters 13 and 21. The phrase "you pick based on how much staleness the feature can tolerate" is the "it depends" the design brief wants — grounded, not vague.

---

### Part 3 — Design problem (about 18 minutes)

**Interviewer:** Let's design something. Design the database for a hotel booking system. Take your time. Start wherever you like.

**Candidate:** Before I draw tables, let me ask a few questions to fix the scope. I would rather build the right thing than a big thing.

1. Is this one hotel, or many hotels on one platform, like a booking site?
2. Do we book a **specific room**, or just a **room type** — like "a deluxe king" — and assign the exact room at check-in?
3. What is the main read pattern — searching availability for a date range? And what is the main write — creating a booking?
4. Do we need to store payments, or is that a separate service?

> **Commentary:** This is the single most important habit in the design round (Chapter 18). The candidate refused to draw tables before scoping. Question 2 in particular — specific room vs room type — completely changes the schema and the double-booking logic. A weak candidate starts drawing `rooms` and `bookings` immediately and boxes themselves in.

**Interviewer:** Good questions. Many hotels on one platform. Book a specific room for now — it keeps the double-booking problem sharp. Main read is availability search by date range. Main write is creating a booking. Payments are a separate service; just store a payment status.

**Candidate:** Clear. So the core entities are: hotels, rooms (each room belongs to a hotel), guests, and bookings. A booking reserves one room for a date range. Let me write the schema.

```sql
CREATE TABLE hotels (
    hotel_id     BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    name         TEXT NOT NULL,
    city         TEXT NOT NULL,
    created_at   TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE rooms (
    room_id      BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    hotel_id     BIGINT NOT NULL REFERENCES hotels(hotel_id),
    room_number  TEXT NOT NULL,
    room_type    TEXT NOT NULL,          -- e.g. 'deluxe_king'
    UNIQUE (hotel_id, room_number)
);

CREATE TABLE guests (
    guest_id     BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    name         TEXT NOT NULL,
    email        TEXT NOT NULL UNIQUE
);

CREATE TABLE bookings (
    booking_id     BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    room_id        BIGINT NOT NULL REFERENCES rooms(room_id),
    guest_id       BIGINT NOT NULL REFERENCES guests(guest_id),
    check_in       DATE NOT NULL,
    check_out      DATE NOT NULL,
    status         TEXT NOT NULL DEFAULT 'confirmed',   -- confirmed | cancelled
    payment_status TEXT NOT NULL DEFAULT 'pending',
    created_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    CHECK (check_out > check_in)
);
```

A few choices I want to call out. I used a `UNIQUE (hotel_id, room_number)` because room numbers repeat across hotels but must be unique inside one hotel. I added a `CHECK (check_out > check_in)` so a booking cannot end before it starts. I model the stay as `check_in` and `check_out` dates, where `check_out` is the morning the guest leaves — so a booking occupies the nights from `check_in` up to but not including `check_out`.

> **Commentary:** Solid, normalized schema (Chapters 19–20). The candidate justified the composite unique key and the check constraint instead of dropping tables silently. Defining the half-open date meaning (check-out not included) is important — it makes the overlap logic below correct. Interviewers notice when a candidate pins down the exact meaning of a date range.

**Interviewer:** Now the hard part. How do you stop two bookings for the same room on overlapping dates — the double-booking problem?

**Candidate:** Right, this is the heart of it. Let me first define overlap, then pick a mechanism.

Two date ranges for the same room overlap when: `existing.check_in < new.check_out` AND `existing.check_out > new.check_in`. Because check-out is not an occupied night, I use strict `<` and `>`, so a booking that checks out on the same day another checks in does *not* conflict.

Now, the naive approach is: in application code, run a `SELECT` to see if any overlapping booking exists, and if not, `INSERT`. That has a race condition. Two requests can both run the `SELECT`, both see no conflict, and both `INSERT`. Now the room is double-booked. A plain check-then-insert is not safe under concurrency.

I would fix it at the database level, not in app code. Two good options in Postgres.

**Option A — an exclusion constraint.** Postgres can enforce "no two rows overlap" directly, using a GiST index over a range type.

```sql
CREATE EXTENSION IF NOT EXISTS btree_gist;

ALTER TABLE bookings
ADD CONSTRAINT no_overlap
EXCLUDE USING gist (
    room_id WITH =,
    daterange(check_in, check_out, '[)') WITH &&
)
WHERE (status = 'confirmed');
```

This says: for the same `room_id` (`WITH =`), two confirmed bookings may not have overlapping date ranges (`&&` is the overlap operator). The `[)` makes the range include check-in and exclude check-out — exactly my rule. The `WHERE (status = 'confirmed')` means cancelled bookings do not block new ones. The database now rejects a conflicting insert itself, no matter how many requests race. This is my preferred answer because the guarantee lives in the schema.

> **Commentary:** This is the strongest possible answer to double-booking. The candidate first exposed the race in the naive check-then-insert, then pushed the guarantee *down into the database* with an exclusion constraint. The partial `WHERE` for cancelled bookings shows real depth — most candidates forget cancelled rows should not block. This is Chapters 11 and 20 combined.

**Interviewer:** And if the platform were MySQL, which has no exclusion constraint?

**Candidate:** Then I fall back to **Option B — locking.** I make the check and the insert one atomic transaction, and I lock so no other transaction can slip in between.

```sql
BEGIN;

-- lock the room row so concurrent bookings for this room serialize
SELECT room_id FROM rooms WHERE room_id = 101 FOR UPDATE;

-- now check for overlap
SELECT 1 FROM bookings
WHERE room_id = 101
  AND status = 'confirmed'
  AND check_in < DATE '2026-10-05'   -- new check_out
  AND check_out > DATE '2026-10-01'  -- new check_in
LIMIT 1;

-- if that returned nothing, insert
INSERT INTO bookings (room_id, guest_id, check_in, check_out)
VALUES (101, 55, '2026-10-01', '2026-10-05');

COMMIT;
```

The `SELECT ... FOR UPDATE` on the room row is the key. It takes a lock on that room. A second request for the same room must wait until this transaction commits, then it re-checks and sees the new booking. So the two requests for the same room serialize, but bookings for *different* rooms still run in parallel — I locked the room, not the whole table. This links back to MVCC in Chapter 11: normal reads do not block, but `FOR UPDATE` takes an explicit row lock on purpose.

> **Commentary:** The candidate adapted cleanly to a new engine. They chose row-level locking, not a table lock, and explained why that keeps throughput high (different rooms stay parallel). This is exactly the pessimistic-locking pattern. Note the candidate locks the *room* row as the serialization point, which is a common and correct trick when the conflicting rows do not exist yet.

**Interviewer:** Small correction — with the exclusion constraint in Option A, do you still need the application-level SELECT check at all?

**Candidate:** Good catch. No — with the exclusion constraint, the `INSERT` will fail on its own if there is an overlap, so I do not need a separate SELECT to be *correct*. I would just catch the constraint violation error and turn it into a friendly "room no longer available" message. I might still run a SELECT earlier for the search page, to show availability — but that is for user experience, not for correctness. The correctness guard is the constraint alone.

> **Commentary:** A second follow-up meant to test whether the candidate understands their own design. They corrected themselves without flinching, and drew the right line: the constraint is for correctness, the SELECT is only for UX. Separating "correctness" from "user experience" is a mature distinction that many miss.

**Interviewer:** Now let's add the search query. Find all available rooms in a hotel for a date range.

**Candidate:** Available means: rooms in the hotel that have *no* confirmed booking overlapping the requested range. I will use `NOT EXISTS`.

```sql
SELECT r.room_id, r.room_number, r.room_type
FROM rooms r
WHERE r.hotel_id = 7
  AND NOT EXISTS (
      SELECT 1 FROM bookings b
      WHERE b.room_id = r.room_id
        AND b.status = 'confirmed'
        AND b.check_in  < DATE '2026-10-05'
        AND b.check_out > DATE '2026-10-01'
  );
```

To make this fast on a large `bookings` table, I would index the lookup — `bookings (room_id, check_in, check_out)` filtered on confirmed, or rely on the GiST index from the exclusion constraint, which already covers room plus range and is well suited to overlap search.

> **Commentary:** `NOT EXISTS` is the right pattern for "rows with no matching child" (Chapters 4 and 6). The candidate reused the exact overlap condition, which shows consistency. Pointing out that the GiST index doubles as the search index is a neat, real-world observation.

**Interviewer:** Last part. This platform grows to thousands of hotels and very high traffic. How would you scale this database?

**Candidate:** Let me go in order, cheapest and safest first. I do not want to shard on day one.

**1. Indexes and query tuning first.** Most "we need to scale" problems are really missing indexes or bad queries. I would confirm the availability search and booking insert are using indexes with `EXPLAIN ANALYZE` before anything else.

**2. Read replicas for read-heavy load.** Search and browse are reads and hugely outnumber bookings. I put those on read replicas and keep writes — the actual booking — on the primary. As we discussed, I route the confirmation read right after a booking to the primary to avoid replication lag showing a stale result.

**3. Caching.** Popular searches — a city for a common weekend — can be cached for a short time. But I would be careful: availability changes on every booking, so I would cache with a short TTL or cache the slow-changing parts, like the hotel and room details, not the live availability.

**4. Partitioning.** The `bookings` table grows forever. I would partition it, most likely by date range — for example, by month of `check_in`. Old bookings move to cold partitions and searches for current dates only scan recent partitions. This is Chapter 14 partitioning inside one database, which is simpler than sharding.

**5. Sharding, only if needed.** If one primary cannot take the write load even after all that, I would shard. The natural shard key here is `hotel_id`, because a booking, its rooms, and its availability search all belong to one hotel. So a single booking stays inside one shard — no cross-shard transaction, which keeps the double-booking guarantee simple. The trade-off is that queries spanning many hotels — like "all bookings by one guest across hotels" — now hit many shards. I would accept that, because the hot path is per-hotel booking, and cross-hotel queries are rare and can go to a separate analytics store.

> **Commentary:** This is a model answer for scaling (Chapters 20–21). The candidate went cheapest-first (indexes → replicas → cache → partition → shard), and refused to shard prematurely. The shard-key reasoning is the best part: choosing `hotel_id` keeps each booking transaction on one shard, so the exclusion-constraint guarantee still holds. They also named the cost — cross-hotel queries fan out — and gave a place to put them. Trade-off stated clearly, exactly as the round rewards.

**Interviewer:** That is a good stopping point. Thanks — that was a strong session.

---

## Scorecard

| Area | Score (1–5) | One-line reason |
|---|:---:|---|
| **SQL** | 5 | Fluent window functions; spotted the tie ambiguity in "second highest" and chose `DENSE_RANK` for the right reason. |
| **Internals** | 4.5 | Clear on indexes, ACID, isolation, MVCC, and replication lag; tied each concept to its mechanism and trade-off. |
| **Design** | 5 | Scoped before drawing; solved double-booking with an exclusion constraint and a locking fallback; sharded on `hotel_id` with correct reasoning. |
| **Communication** | 5 | Asked clarifying questions first, narrated trade-offs, and recovered cleanly from two follow-up corrections. |

**Overall verdict: Strong hire.**

The candidate showed the three things every database round scores: correct SQL under a twist, real understanding of internals (not memorized slogans), and a design that survives concurrency and scale. Just as important, they communicated like an engineer — assumptions out loud, trade-offs named, mistakes owned.

### What would have made this even stronger

- **Numbers.** The scaling answer would be sharper with rough figures: expected bookings per second, read/write ratio, or table size. "At 500 writes per second on one primary, replicas are enough; sharding is premature" is more convincing than "if it grows."
- **Failure and recovery.** No one asked, but volunteering a line on backups, or what happens if the primary fails (failover, promoting a replica), would show operational maturity.
- **Idempotency.** For the booking write, mentioning an idempotency key — so a retried request does not create two bookings — would round out the concurrency story beyond just double-booking.

None of these are gaps that sink the verdict. They are the difference between "strong hire" and "raise the level."

---

## How to Practice

You cannot cram this. You build it. Here is a simple plan.

1. **Do it out loud.** Read a question from Chapters 6, 20, or 23 and answer it *speaking*, not in your head. Interviews test talking through a problem, not silent solving.

2. **Time yourself.** Give a warm-up SQL question 10 minutes, an internals question 5, a design question 20. Practice fitting a real answer in real time.

3. **Always add the twist yourself.** After you solve a query, ask: "What if there are ties? What if the table is huge? What index helps?" The interviewer will — so beat them to it.

4. **Say assumptions and trade-offs every time.** Force the habit: one clarifying question before any design, and one named trade-off after any choice. This alone lifts most candidates a full grade.

5. **Pair up.** Trade mock interviews with a friend. Being the interviewer teaches you what a weak answer sounds like from the other chair.

6. **Keep a mistake log.** Every time you get one wrong — a missed tie, a forgotten `WHERE status = 'confirmed'`, a premature shard — write it down. Review the log before the real interview.

Work through the chapters, then come back to this transcript. When the arc here — scope, reason, trade-off, recover — feels natural rather than scripted, you are ready.
