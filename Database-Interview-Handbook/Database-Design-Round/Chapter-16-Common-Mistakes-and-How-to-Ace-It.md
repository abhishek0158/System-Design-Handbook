# Chapter 16 — Common Mistakes & How to Ace the Design Round

This chapter lists the mistakes that cost candidates the most points. It also gives you a
checklist and a time plan for the design round.

## Top Mistakes to Avoid

**1. Jumping to tables before asking about requirements and scale.**
Some candidates hear the problem and start writing `CREATE TABLE` right away. This looks fast,
but it is risky. You may build the wrong thing. The fix: ask questions first. Ask what the main
features are. Ask how many users, how many writes per second, how much data. Only then start the
schema. See Chapter 1 for the full method.

**2. Ignoring the access patterns.**
An access pattern is the main query your app runs, like "get all orders for a user" or "get the
latest 20 messages in a chat". If you do not know the access patterns, you cannot design a good
schema. Some candidates design tables that look neat but are slow for the real queries. The fix:
before you finish the schema, list the top 3–5 queries. Design the tables and indexes for those
queries.

**3. Forgetting indexes.**
A schema with correct tables but no indexes will be slow. This is a common gap. Every foreign key
that you filter or join on needs an index. Every column in your `WHERE` clause on a large table
needs an index. The fix: after you draw the tables, go through each main query and ask, "what
index does this need?" Say it out loud. See Chapter 2 for index patterns.

**4. Over-normalizing (too many joins).**
Normalization means splitting data into many small tables to avoid repeating it. This is good
practice, but too much of it creates a schema with 10 joins for one simple read. This is slow and
hard to reason about. The fix: normalize for correctness first. Then, if one read path is hot (run
very often), denormalize on purpose. Say why. For example: "I will copy the product name onto the
order line, because the order history must not change when the product name changes later."

**5. Over-denormalizing (data out of sync).**
The opposite mistake: copying the same data into many tables without a plan to keep it in sync.
For example, storing a user's total order count in three different tables. If one update fails or
is missed, the numbers disagree. The fix: denormalize only when you have a clear reason (a hot
read, or a value that must be frozen in time, like a price). State how you will keep the copies
correct — a background job, a database trigger, or "this value is copied once and never updated
again" (a price snapshot).

**6. Storing large files or blobs in the database.**
Some candidates store images, videos, or PDF files as `BYTEA` or `BLOB` columns. This makes the
database huge and slow to back up. It also makes normal queries slower, because the database
engine has to skip over large rows. The fix: store the file in an object store (like S3). Store
only the URL or file key in the database row. Say this out loud — it is a strong signal that you
know how real systems are built.

**7. Not handling concurrency where it matters.**
Concurrency means many requests happening at the same time. If two people try to book the same
seat, or a user double-clicks "pay", or a poll gets two votes from one click, you can get bad data
if you do not guard against it. Common failure cases: double-booking a room, double-charging a
card, double-counting a vote. The fix: use a **unique constraint** (e.g., one row per seat per
show, so a second booking fails), a **database transaction** with the right isolation level, or a
**row lock** (`SELECT ... FOR UPDATE`) to stop two requests from booking the same row at once. Say
which one you would use and why. See Chapter 6 for a full booking example.

**8. Using float for money.**
A `float` (floating-point number) cannot store some decimal values exactly. `0.1 + 0.2` may not
equal `0.3` in a float. Over many transactions, small errors add up. This is a classic red flag in
interviews. The fix: always store money as an integer count of the smallest unit (like cents), or
as `NUMERIC` with a fixed number of decimal places. Never use `FLOAT` or `DOUBLE` for money. See
Chapter 8.

**9. Not stating assumptions.**
If you design silently and make private assumptions ("I assume 1 million users"), the interviewer
cannot follow your reasoning, and may think you missed the scale question. The fix: say your
assumptions out loud, even if you are guessing. For example: "I will assume 10,000 writes per
second at peak, since you said this is a large e-commerce site. Let me know if that is far off."
This invites correction and shows your thinking.

**10. Going silent instead of thinking aloud.**
Some candidates think in silence for a long time, then present a finished answer. The interviewer
cannot see your reasoning this way, and may think you are stuck or unsure. The fix: narrate your
thinking. Say what you are considering, what trade-off you see, and why you pick one option. A
design round is not a written exam. It is a conversation. Half of your score comes from how you
explain your choices, not just the final schema.

## The Design-Round Checklist

Run through this list as you design. It is a quick self-check, not a strict order.

- [ ] Did I ask about the main features and the users, before drawing tables?
- [ ] Did I ask about scale (rough number of users, rows, writes per second)?
- [ ] Did I list the top 3–5 access patterns (main queries) before finalizing tables?
- [ ] Does every table have a clear primary key?
- [ ] Does every foreign key have an index (if I will filter or join on it)?
- [ ] Did I check each main query and confirm it has a supporting index?
- [ ] Is money stored as an integer (cents) or `NUMERIC`, never `float`?
- [ ] Did I copy any "at the time" values (price, name) that must not change later?
- [ ] Did I find any place where two requests at the same time could cause bad data (double-
  booking, double-charge, double-vote)? Did I add a unique constraint, lock, or transaction there?
- [ ] Did I keep large files (images, videos, PDFs) out of the database, using an object store
  instead?
- [ ] Did I say out loud what breaks first as this system grows, and how I would fix it (index,
  cache, read replica, or shard)?
- [ ] Did I state my assumptions clearly, instead of guessing silently?
- [ ] Did I keep talking through my reasoning, instead of going quiet while I think?

## How to Structure Your Time

A design round is usually 30–40 minutes. Here is a rough time split. Adjust it to the interviewer's
pace, but do not skip a step.

**1. Requirements & scale (5–7 minutes).**
Ask what the system must do. Ask about read vs write volume, and rough scale (users, rows per
day). Do not skip this, even under time pressure. It sets up everything else.

**2. Entities & relationships (5 minutes).**
Name the main "things" in the system (user, order, product, and so on). Say how they relate: one-
to-many, many-to-many. Draw this as a quick list or a small diagram. Keep it high level here — do
not write columns yet.

**3. Schema & keys (8–10 minutes).**
Now write the tables. Show the primary keys and foreign keys. Keep each table focused — the
important columns only, not every field. Say "…" for the rest.

**4. Indexes & access patterns (5 minutes).**
Go back to the main queries from step 1. For each one, name the index that makes it fast. If a
query is still slow even with an index, consider a denormalized read table.

**5. One hard problem, deep-dive (8–10 minutes).**
Most rounds have one hard part: a concurrency problem (double-booking), a money problem
(consistent balances), or a scale problem (a huge feed). Spend real time here. This is where you
earn most of your score. Show the exact fix — a unique constraint, a lock, a transaction, or a
queue — and explain the trade-off.

**6. Scaling it (3–5 minutes).**
End with a short note on what breaks first as load grows, and the next step: add an index, add a
cache, add a read replica, or shard the data. You do not need to design the full scaled system.
Naming the right next step is enough.

If time runs short, protect steps 1, 4, and 5. Those show the most reasoning per minute.

## Phrases That Score

Strong candidates use plain, direct phrases like these. Borrow them and make them your own.

- "Let me confirm the main read patterns first, before I lock in the schema."
- "What is the rough scale here — are we talking thousands or millions of rows?"
- "I will copy the price onto the order line, so the order history stays stable even if the
  product price changes later."
- "This needs a unique constraint to stop double-booking under concurrency."
- "I will use a database transaction here, so the balance check and the deduction happen
  together, or not at all."
- "I am normalizing this for correctness. If this read turns out to be very hot, I would
  denormalize it on purpose, and explain why."
- "I will keep the file itself in an object store like S3, and store only the file key here."
- "Let's assume 5,000 writes per second at peak — tell me if that is off, and I will adjust."
- "The first thing to break here is probably this table, once it gets very large. I would add an
  index first, then consider a read replica."
- "I am not fully sure yet, so let me think out loud: one option is X, another is Y, and here is
  the trade-off."

## Quick Recall

- Ask before you design: features, access patterns, scale.
- Design for the real queries. Add the index for each one.
- Normalize for correctness. Denormalize only on purpose, for a clear reason.
- Keep big files out of the database. Store only a reference.
- Guard concurrency with unique constraints, locks, or transactions, wherever double-writes can
  happen.
- Money is always an integer (cents) or `NUMERIC`. Never `float`.
- Say your assumptions. Think out loud. A quiet, silent design loses points, even if it is
  correct.
- Time plan: requirements → entities → schema → indexes → one hard problem → scaling.
