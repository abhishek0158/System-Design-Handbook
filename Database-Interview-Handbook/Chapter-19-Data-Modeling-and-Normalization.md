# Chapter 19 — Data Modeling & Normalization

Data modeling is how you turn a real-world problem into tables, columns, and relationships.
Normalization is a set of rules that remove duplicate data and stop bad updates. Interviewers
ask about this because a good schema prevents bugs later, and a bad one causes them for years.

## Key Concepts

### Entities, attributes, and relationships

An **entity** is a real-world thing you store data about, such as a customer, a product, or an
order. Each entity becomes a table. An **attribute** is a property of an entity, such as a
customer's name or a product's price. Each attribute becomes a column.

A **relationship** describes how two entities connect. There are three kinds.

**One-to-one (1:1)**: one row in table A matches exactly one row in table B. Example: one
customer has one profile with extra details you rarely read (bio, avatar, date of birth). You
split it out so the main `customers` table stays small and fast to scan.

```sql
CREATE TABLE customers (
    customer_id BIGSERIAL PRIMARY KEY,
    name        TEXT NOT NULL,
    country     TEXT,
    created_at  TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE customer_profile (
    customer_id   BIGINT PRIMARY KEY REFERENCES customers(customer_id),
    bio           TEXT,
    avatar_url    TEXT,
    date_of_birth DATE
);
```

Notice `customer_profile.customer_id` is both the primary key and a foreign key. That is what
forces a true 1:1 relationship: each customer can have at most one profile row.

**One-to-many (1:N)**: one row in table A matches many rows in table B. Example: one customer
places many orders. You put the foreign key on the "many" side.

```sql
CREATE TABLE orders (
    order_id     BIGSERIAL PRIMARY KEY,
    customer_id  BIGINT NOT NULL REFERENCES customers(customer_id),
    order_date   DATE NOT NULL,
    status       TEXT NOT NULL,
    total_amount NUMERIC(12,2) NOT NULL
);
```

**Many-to-many (N:N)**: many rows in table A relate to many rows in table B. Example: one order
can contain many products, and one product can appear in many orders. A plain foreign key
cannot express this. You need a third table called a **junction table** (also called a join
table or bridge table). It holds one row per pairing, plus any attribute that belongs to the
pairing itself, such as quantity.

```sql
CREATE TABLE products (
    product_id BIGSERIAL PRIMARY KEY,
    name       TEXT NOT NULL,
    category   TEXT,
    price      NUMERIC(10,2) NOT NULL
);

CREATE TABLE order_items (
    order_id   BIGINT NOT NULL REFERENCES orders(order_id),
    product_id BIGINT NOT NULL REFERENCES products(product_id),
    quantity   INT NOT NULL CHECK (quantity > 0),
    unit_price NUMERIC(10,2) NOT NULL,
    PRIMARY KEY (order_id, product_id)
);
```

The composite primary key `(order_id, product_id)` does two jobs: it stops the same product
being added twice to one order, and it is what makes this table a true many-to-many bridge.
Always index both foreign key columns in a junction table — Postgres indexes the primary key
automatically, but if you query "all orders for product X" often, add a separate index on
`product_id` alone too, since the composite index only helps when `order_id` is also filtered.

### Primary keys, foreign keys, surrogate vs natural keys

A **primary key (PK)** is the column, or set of columns, that uniquely identifies each row. It
must be unique and never null. A **foreign key (FK)** is a column that points to the primary
key of another table. The database uses it to enforce **referential integrity** — a rule that
you cannot insert a row pointing to a parent that does not exist, and by default you cannot
delete a parent row while children still point to it.

A **natural key** is a real-world attribute that is already unique, such as an email address, a
national ID, or a product barcode. A **surrogate key** is an artificial ID with no business
meaning, usually an auto-incrementing integer or a UUID, such as `customer_id` in the schema
above.

**Prefer surrogate keys in almost all cases.** Reasons:
- Natural keys can change. A user can change their email; a country can rename itself. If that
  value is a primary key, the change must cascade into every table that stores it as a foreign
  key. That is slow and risky.
- Natural keys are often long strings (email, SKU code). A `BIGINT` surrogate key is smaller,
  which makes indexes smaller and joins faster.
- Business rules change. Today a barcode is globally unique; next year the business merges with
  another company and barcodes collide.

Use a natural key only when the value is short, truly immutable, and controlled by a standard,
not by your users. Example: a `country_code CHAR(2)` (ISO code) as the primary key of a small,
static `countries` lookup table. Even then, many teams still add a surrogate key out of habit
for consistency across all tables.

Still, keep a **unique constraint** on the natural key even when you use a surrogate primary
key. Example: `customers.email` should have `UNIQUE` even though `customer_id` is the PK. This
gives you the best of both — stable joins through the surrogate key, and no duplicate emails.

### Normalization — a worked example

**Normalization** means organizing columns and tables so that each fact is stored in exactly
one place. This removes duplicate data and stops three kinds of update anomalies:
- **Insert anomaly**: you cannot add a fact (e.g., a new product) without also having an
  unrelated fact (e.g., an order that uses it).
- **Update anomaly**: the same fact is stored in many rows, so you must update all of them, and
  if you miss one, the data becomes inconsistent.
- **Delete anomaly**: deleting one row accidentally removes a fact you still needed.

Each **normal form (NF)** is a stricter rule. You normalize step by step. Start from this
single flat table that a junior developer might design first — note this table is only for
this example, not one of the shared schemas used elsewhere in this book:

| order_id | order_date | cust_name | cust_email | cust_city | prod1_name | prod1_cat | prod1_qty | prod2_name | prod2_cat | prod2_qty |
|---|---|---|---|---|---|---|---|---|---|---|
| 101 | 2026-01-05 | Amit | amit@x.com | Pune | Mouse | Electronics | 2 | Keyboard | Electronics | 1 |

**Problem with this table:** it has a fixed number of product slots (`prod1`, `prod2`). An
order with three products does not fit. An order with one product wastes two empty columns.
You also cannot easily answer "find every order containing the Mouse" — you would have to
check every `prodN_name` column.

**Step 1 — First Normal Form (1NF).** Rule: every column must hold one atomic (single,
indivisible) value, and there must be no repeating groups of columns (like `prod1`, `prod2`,
`prod3`...). Fix: give each product its own row instead of its own set of columns.

```sql
-- order_line: one row per product per order
order_line(order_id, order_date, cust_name, cust_email, cust_city,
           product_name, product_category, quantity)
-- primary key: (order_id, product_name)
```

**What this removes:** the repeating-group problem. Now an order can have any number of
products — just add more rows. Querying "orders containing Mouse" is now a simple `WHERE
product_name = 'Mouse'`.

**Step 2 — Second Normal Form (2NF).** Rule: 2NF applies only to tables with a **composite
primary key** (a key made of more than one column). It says every non-key column must depend on
the *whole* key, not on just part of it. This is called a **partial dependency**, and 2NF
removes it.

Look at `order_line`. Its key is `(order_id, product_name)`. But `product_category` depends
only on `product_name` — it has nothing to do with which order it is in. That is a partial
dependency: a non-key column depending on only part of the composite key. Fix: move
product details into their own table.

```sql
products(product_id PK, name, category, price)

order_items(order_id FK, product_id FK, quantity, unit_price)
-- primary key: (order_id, product_id)
```

**What this removes:** now `product_category` is stored once per product, not once per order
line. Before this fix, if "Mouse" changed category, you had to update every order row that ever
included a Mouse — an update anomaly.

Meanwhile, keep the order-level facts in a separate table with a single-column key:

```sql
order_header(order_id PK, order_date, cust_name, cust_email, cust_city)
```

A table with a single-column primary key automatically satisfies 2NF — partial dependency can
only happen when the key has more than one column.

**Step 3 — Third Normal Form (3NF).** Rule: no **transitive dependency** — a non-key column
must not depend on another non-key column. It must depend only on the primary key, directly.

Look at `order_header`. The key is `order_id`. But `cust_city` does not really describe the
order — it describes the customer. The real dependency chain is `order_id → cust_email →
cust_city`. That is transitive: `cust_city` depends on `cust_email`, which depends on
`order_id`, not directly on `order_id`. Fix: move customer facts into their own table.

```sql
customers(customer_id PK, name, email, city)

orders(order_id PK, order_date, customer_id FK)
```

**What this removes:** now a customer's city is stored once, in the `customers` row, no matter
how many orders they place. Before this fix, if a customer moved city, you had to find and
update every one of their orders — another update anomaly. It also removes the delete anomaly:
deleting a customer's only order no longer erases the fact that "Amit lives in Pune."

**Final 3NF schema** — this now matches the shared e-commerce schema used across this book:

```sql
customers(customer_id PK, name, email, city, ...)
products(product_id PK, name, category, price)
orders(order_id PK, order_date, customer_id FK, status, total_amount)
order_items(order_id FK, product_id FK, quantity, unit_price)  -- PK (order_id, product_id)
```

**BCNF (Boyce-Codd Normal Form) — one line:** BCNF is a slightly stricter version of 3NF for
the rare case where a table has more than one candidate key and a non-key-looking column
actually determines part of a key (for example, a `(student, subject)` table where each subject
has exactly one `teacher`, and each teacher teaches only one subject, so `teacher → subject`
even though `teacher` is not part of the primary key). Most interview schemas stop at 3NF; you
rarely need to name BCNF explicitly, but knowing it exists shows depth.

### Denormalization

**Denormalization** means deliberately re-introducing duplicate or precomputed data that
normalization would remove, in order to make reads faster. You do this after normalizing
correctly first, and only where you have a real, measured reason.

Common reasons to denormalize:
- **Avoid expensive joins on the hot read path.** Example: `orders.total_amount` in the shared
  schema is already denormalized — it is the sum of `order_items.quantity * unit_price`, stored
  directly on the order so a listing page does not need to join and sum every time.
- **Reporting and dashboards.** Precomputed daily aggregates (e.g., `daily_sales(date,
  total_revenue)`) are far cheaper to read than scanning millions of order rows each time.
- **High read-to-write ratio.** If a table is read a thousand times for every one write, paying
  a small write cost to keep a copy in sync is worth it.

The cost is real and you must say it out loud in an interview:
- **Update anomalies come back.** If you copy `customer_name` onto every order row, and the
  customer changes their name, you must update every order too, or the copies go stale.
- **Extra write complexity.** You need application logic, a database trigger, or a background
  job to keep the copies in sync. This is more code to maintain and more ways to introduce bugs.
- **More storage.** Usually a minor cost compared to the read savings, but still worth
  mentioning.

**Interview framing:** normalize first for correctness, then denormalize selectively, based on
actual query patterns, not guesses. State the trade-off out loud: "I would keep the normalized
tables as the source of truth, and denormalize into a cache table or materialized view for the
read-heavy path, refreshed on write or on a schedule." This shows you understand "it depends"
rather than picking a side blindly.

### Common modeling patterns

**Hierarchical / tree data.** Example: an employee reporting chain, or nested product
categories. The simplest pattern is the **adjacency list**: a self-referencing foreign key, like
`employees.manager_id → employees.emp_id` in the shared HR schema.

```sql
SELECT e.emp_name, m.emp_name AS manager_name
FROM employees e
LEFT JOIN employees m ON e.manager_id = m.emp_id;
```

This is easy to write and update, but finding "all descendants of X" needs a recursive CTE
(covered in Chapter 4) or repeated self-joins. For deep trees read often, a **closure table**
(a separate table storing every ancestor-descendant pair) or a **materialized path** (a string
column like `'/1/4/9/'` showing the full path from the root) makes "find all descendants" a
single indexed query, at the cost of extra storage and more complex writes.

**Soft deletes.** Instead of `DELETE`-ing a row, add a nullable `deleted_at TIMESTAMPTZ` column.
"Deleting" means setting it to the current time; every normal query filters with `WHERE
deleted_at IS NULL`. This keeps history and supports "undo," but it has costs: every query must
remember the filter, and a `UNIQUE` constraint (like on email) can wrongly block a new row from
reusing a value that only a soft-deleted row holds — you usually need a partial unique index,
e.g., `CREATE UNIQUE INDEX ON customers(email) WHERE deleted_at IS NULL`.

**Audit / history tables.** To track how a row changed over time, keep a separate table, such
as `order_status_history(order_id, old_status, new_status, changed_at, changed_by)`, written to
whenever the main row changes (usually via application code or a trigger). This answers "who
changed what, and when" without bloating the main table.

**Status columns.** A column like `orders.status` needs a controlled, small set of values.
Three common ways to enforce this: a Postgres `CHECK` constraint listing allowed values, a
Postgres `ENUM` type, or a small **lookup table** (see next). A `CHECK` or `ENUM` is simpler for
values that almost never change (e.g., `'pending'`, `'shipped'`, `'delivered'`, `'cancelled'`).

**Lookup / enum tables.** For values that might grow, or that need extra fields like a
description, use a real table instead of hardcoded strings:

```sql
order_status(status_code PK, description)
orders(order_id PK, ..., status_code FK REFERENCES order_status(status_code))
```

This lets you add a new status without a code deployment, and the foreign key stops typos like
`'shiped'` from ever entering the column — something a plain `VARCHAR` column cannot do.

## The Questions They Ask

**"Explain normalization with an example."**
Give the worked example above in short form: start with one flat table that repeats customer
and product details on every row. Show the insert/update/delete anomalies it causes. Then walk
through 1NF (split repeating product columns into rows), 2NF (move product details out of the
composite-key order line table, since they only depend on part of the key), and 3NF (move
customer details out of the orders table, since city depends on the customer, not the order).
End on the final schema matching `customers`, `products`, `orders`, `order_items`. Interviewers
want to see you name the anomaly each step removes, not just recite the rule.

**"What is the difference between 2NF and 3NF?"**
Both remove a kind of column depending on the "wrong" thing, but at different scope. 2NF is
only relevant when the primary key has more than one column — it removes a **partial
dependency**, where a column depends on only part of the composite key. 3NF applies to any
table — it removes a **transitive dependency**, where a non-key column depends on another
non-key column instead of depending directly on the primary key. Follow-up: "does 3NF require
2NF first?" Yes — normal forms are cumulative; a table cannot be in 3NF unless it is already in
2NF and 1NF.

**"When would you denormalize?"**
When a read path is hot (queried very often) and normalized joins are measured to be too slow,
or when building reporting/analytics tables that need precomputed aggregates. Say the honest
cost too: you accept the risk of stale copies and must add code (trigger, background job, or
application logic) to keep them in sync. State that you would normalize first, measure, then
denormalize the specific hot path — not the whole schema.

**"How do you model a many-to-many relationship?"**
With a junction table that has a composite primary key made of the two foreign keys, e.g.,
`order_items(order_id FK, product_id FK, ...)` with `PRIMARY KEY (order_id, product_id)`. Any
attribute that belongs to the pairing itself (quantity, price at time of order, a timestamp for
when a student enrolled in a course) lives on the junction table, not on either side. Follow-up:
"what if there is no extra attribute, like `students` and `courses`?" You still need the
junction table (`enrollments(student_id, course_id)`) — a foreign key alone can only express
one-to-many, never many-to-many.

**"Surrogate key or natural key — which do you use, and why?"**
Default to a surrogate key (auto-increment integer or UUID) as the primary key, because natural
keys can change, can be long, and business rules around them can change later. Still add a
`UNIQUE` constraint on the natural key (like email) so the database still enforces uniqueness.
Use a natural key directly only for small, static, standard-controlled reference data, like an
ISO country code. Follow-up: "integer or UUID for the surrogate key?" Integers (`BIGSERIAL`) are
smaller and faster to index; UUIDs are useful when IDs must be generated by the client or across
distributed systems before an insert, at the cost of larger index size (covered more in
Chapter 21).

## Rapid-Fire

- **What is an entity?** A real-world thing you model as a table, e.g., a customer or an order.
- **What is a repeating group?** Multiple similar columns like `prod1_name`, `prod2_name` in one
  row — 1NF forbids this; use separate rows instead.
- **What does 1NF require?** Atomic column values, no repeating groups.
- **What does 2NF add?** No partial dependency — every non-key column depends on the whole
  composite key, not just part of it.
- **What does 3NF add?** No transitive dependency — non-key columns depend only on the primary
  key, not on another non-key column.
- **Is BCNF stricter than 3NF?** Yes, it closes a rare gap in 3NF around overlapping candidate
  keys. Rarely needed by name in interviews.
- **What is a junction table?** A table that holds foreign keys from two tables plus a
  composite primary key, used to model many-to-many relationships.
- **Can a foreign key be null?** Yes, unless you add `NOT NULL` — a null FK means "no parent
  yet," useful for optional relationships.
- **Surrogate key example?** An auto-increment `customer_id BIGSERIAL`.
- **Natural key example?** An email address, an ISBN, an ISO country code.
- **Why avoid natural keys as PKs?** They can change, can be long, and business meaning can
  shift — all of which force painful cascading updates.
- **What is denormalization?** Deliberately duplicating or precomputing data to make reads
  faster, at the cost of extra write complexity and possible stale copies.
- **Give a denormalization example already in this book's schema.** `orders.total_amount` —
  precomputed instead of summed from `order_items` on every read.
- **How do you keep denormalized copies in sync?** Application code, a database trigger, or a
  scheduled background job.
- **Best pattern for deep tree queries?** A closure table or materialized path, not repeated
  self-joins on an adjacency list.
- **How do you soft delete safely with a unique column?** Use a partial unique index that only
  applies `WHERE deleted_at IS NULL`.

## Common Traps & Mistakes

**Confusing partial and transitive dependency.** Candidates often can define 2NF and 3NF but
mix up which is which under pressure. Remember: 2NF is about the *key* (part of a composite
key), 3NF is about *other non-key columns*. If the table's primary key is a single column, 2NF
is automatically satisfied — only 3NF (and BCNF) can still be violated.

**Storing comma-separated values in one column.** Writing `product_ids = '12,45,90'` in an
`orders` row is a 1NF violation — the value is not atomic, and you cannot join, index, or
constrain individual IDs inside it. Use a junction table instead.

**Forgetting the composite primary key on a junction table.** Without `PRIMARY KEY (order_id,
product_id)`, nothing stops the same product being inserted twice into the same order, which
silently corrupts totals and reports.

**Not indexing foreign key columns.** Postgres does not automatically index a foreign key
column (it only indexes the primary key). On a junction table, add an index on the "second"
column of the composite key too if you query from that side often, or joins and cascading
deletes will be slow.

**Choosing a natural key that later turns out to be mutable.** Email as a primary key seems
fine until a user asks to change it, and now every foreign key referencing it across the schema
must cascade. This is the single most common "we regret this schema decision" story in real
systems.

**Denormalizing before measuring.** Copying data around "for performance" without first
checking whether the join is actually a bottleneck adds update-anomaly risk for no proven gain.
Normalize first; denormalize a specific, measured hot path.

**Over-normalizing a hot read path.** The opposite trap: chasing 3NF or BCNF purity on a table
that is read thousands of times per second, forcing four or five joins for one page load. State
the trade-off — correctness and no duplication versus read latency — instead of treating 3NF as
a law that can never bend.

**Forgetting soft-delete filters.** Once a table has `deleted_at`, every `SELECT`, `JOIN`, and
uniqueness rule must account for it, or "deleted" rows silently reappear in reports or block new
inserts that should be allowed.

See Chapter 18 for how to structure a design interview end to end, and Chapter 20 for full
worked schema designs that apply these normalization and modeling patterns to real systems.
