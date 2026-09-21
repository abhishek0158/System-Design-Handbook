# Chapter 1 — Primary / Foreign Keys & Constraints

Keys and constraints are rules the database enforces on your data. Interviewers ask about them
because they test if you know *where* data correctness should live — in the database, or in your
application code.

## The Idea

A **primary key** is a column (or set of columns) that uniquely identifies each row in a table.
Two rules come with it: the value must be unique, and it can never be `NULL`. A table can have
only one primary key.

```sql
CREATE TABLE orders (
    order_id    INT PRIMARY KEY,
    customer_id INT NOT NULL,
    order_date  DATE NOT NULL,
    status      TEXT NOT NULL DEFAULT 'pending',
    total_amount NUMERIC(10,2) CHECK (total_amount >= 0)
);
```

A **foreign key** is a column that points to a primary key in another table. It links two tables
together and stops you from inserting a value that does not exist in the parent table. This is
called **referential integrity** — every reference points to something real.

```sql
CREATE TABLE order_items (
    item_id  INT PRIMARY KEY,
    order_id INT NOT NULL REFERENCES orders(order_id),
    product  TEXT NOT NULL,
    qty      INT CHECK (qty > 0)
);
```

Here, `order_items.order_id` must match an existing `orders.order_id`. You cannot insert an item
for an order that does not exist.

Other constraints, in plain words:

- **UNIQUE** — no two rows can have the same value in this column. Unlike a primary key, a table
  can have many unique columns, and a unique column *can* allow one `NULL` (in PostgreSQL, `NULL`
  is not compared as equal to another `NULL`, so many nulls are allowed too).
- **NOT NULL** — this column must always have a value.
- **CHECK** — a custom rule the value must pass, for example `total_amount >= 0`.
- **DEFAULT** — a value used automatically when you do not supply one, for example
  `status DEFAULT 'pending'`.

**Composite key**: a primary key (or unique constraint) made of more than one column, used when
no single column is unique by itself. For example, in a table `enrollments(student_id, course_id)`,
neither column alone is unique, but the pair together is.

```sql
CREATE TABLE enrollments (
    student_id INT REFERENCES students(student_id),
    course_id  INT REFERENCES courses(course_id),
    PRIMARY KEY (student_id, course_id)
);
```

**Surrogate vs natural key**: a **natural key** is a real-world value, like an email address or a
national ID number, used as the primary key. A **surrogate key** is an artificial value, like an
auto-incrementing integer or a UUID, that has no business meaning — it exists only to identify the
row.

## Why It Matters — the Reasoning

**Why put rules in the database, not just in application code?**
The database is the **last line of defence**. Application code can have bugs. There may be more
than one application, or more than one service, writing to the same table — a batch job, an admin
script, a new microservice. If the rule (uniqueness, not-null, valid reference) lives only in one
app's code, every other writer can break it. The database is the one place all writers must pass
through, so it is the only place that can *guarantee* the rule.

There is also a **race condition** problem. Imagine two application servers both check "does this
email already exist?" at almost the same time, both see "no," and both then insert the same email.
Without a `UNIQUE` constraint in the database, you now have a duplicate — the check-then-insert
pattern is not safe under concurrency. A database constraint checks and rejects atomically at
insert time, so this race cannot happen. This is a strong interview point: constraints are not
just "extra safety," they solve a real concurrency bug that application-level checks cannot solve
alone.

That said, good practice is usually **both**: validate early in the app for a fast, friendly error
message, but always enforce the real rule in the database too, because the app-level check can be
skipped, buggy, or bypassed.

**The foreign-key trade-off.**
A foreign key gives you integrity — you can never end up with an `order_items` row pointing to a
deleted order. But it costs something:
- Every insert or update on the child table needs an extra check against the parent table. This
  is a small but real write-cost.
- Deleting or updating a parent row is no longer simple. The database must decide what happens to
  the children. You control this with **ON DELETE** behaviour:
    - **CASCADE** — delete the children automatically when the parent is deleted. Convenient, but
      dangerous: a single delete can silently remove a lot of connected data.
    - **RESTRICT** (or the default, `NO ACTION`) — block the delete of the parent if children still
      reference it. Safer, but means you must delete children first, in the right order.
    - **SET NULL** — the child row survives, but its foreign-key column is set to `NULL` (only
      possible if that column allows `NULL`). Useful when the child does not really "belong" only to
      that parent, for example an `orders.assigned_employee_id` — if the employee is deleted, the
      order should stay, just unassigned.

The choice is really a business decision, not just a technical one: "if the parent disappears,
what should happen to the children?" Getting this wrong (e.g. `CASCADE` where you meant
`RESTRICT`) is a classic way to lose data by accident.

**Some teams skip foreign keys entirely** at high write volume or in sharded/distributed setups,
because the extra check on every write and the locking it can cause become a bottleneck, and
because the parent and child rows may not even live on the same node. This is a real trade-off:
you gain write speed and flexibility, but you push the "do not leave orphan rows" responsibility
back onto the application — and lose the last line of defence. Mention this trade-off in an
interview if asked "would you always use foreign keys."

## Common Interview Questions

**1. Primary key vs unique key — what is the difference?**
Both guarantee uniqueness. A primary key also disallows `NULL` and there can be only one per
table; PostgreSQL uses it as the default way to identify a row. A unique key can allow a `NULL`
(NULLs are not treated as equal, so you can even have several NULLs), and a table can have many
unique constraints. Reasoning: a primary key is "the" identity of the row and other tables use it
to reference this one; a unique key is just "this column happens to have no duplicates."

**2. Can a primary key be `NULL`?**
No, never. Reasoning: a primary key's whole job is to let you uniquely find one exact row. `NULL`
means "unknown value," and you cannot compare two unknowns and say they are the same row. If
`NULL` were allowed, "find the row where id = NULL" would be meaningless, so the identity
guarantee would break.

**3. What does a foreign key actually do?**
It forces every value in the child column to already exist as a primary key value in the parent
table (referential integrity). It also controls what happens when the parent row is deleted or
updated (`CASCADE` / `RESTRICT` / `SET NULL`). Reasoning: without it, nothing stops an
`order_items` row from pointing at an `order_id` that was deleted — your data would have "orphan"
rows referring to nothing, and any report or join using that reference could silently drop or
break.

**4. `ON DELETE CASCADE` vs `RESTRICT` — when would you use each?**
Use `CASCADE` when the child row has no meaning without the parent — for example, deleting an
`order` should also delete its `order_items`, because an order item cannot exist alone. Use
`RESTRICT` when the child row is independently important and an accidental delete would be costly
— for example, you probably do not want deleting a `product` to silently cascade-delete every
past `order_item` that ever referenced it; you would rather the delete fail and force a human to
check. Reasoning: `CASCADE` trades safety for convenience; `RESTRICT` trades convenience for
safety. Pick based on how bad an accidental mass-delete would be.

**5. Should you enforce a rule in the database or in the application?**
Enforce correctness-critical rules (uniqueness, required fields, valid references) in the
database, because it is the single shared point all writers must go through, and it closes race
conditions that application-level checks cannot close. Use the application layer for fast
feedback and friendlier error messages, and for rules that need business context the database
does not have (for example, "only a manager can approve this"). Reasoning: defence in depth — the
app layer is for user experience, the database layer is for guaranteed correctness.

**6. Surrogate key vs natural key — which would you pick and why?**
Prefer a surrogate key (auto-increment integer or UUID) in most systems. Natural keys look
convenient (like email or a national ID) but real-world values can change (a person changes their
email), can have surprising duplicates (two systems issue the same ID differently), or can be
large and slow to index (a long text string used in every foreign key). A surrogate key is stable,
small, and never changes, so every foreign key pointing to it never needs to update. Reasoning:
the primary key's job is stability of identity, and business data is rarely stable — it changes
for business reasons outside the database's control.

## Quick Recall

- Primary key = unique + `NOT NULL` + one per table. It is the row's identity.
- Foreign key = must match an existing primary key value in another table (referential
  integrity); you choose `CASCADE` / `RESTRICT` / `SET NULL` for what happens on parent delete.
- `UNIQUE` allows `NULL`s (multiple, even); primary key never allows `NULL`.
- `CHECK` and `DEFAULT` are value-level rules, not identity rules.
- Composite key = uniqueness needs more than one column together.
- Prefer surrogate keys; natural keys can change and break every table referencing them.
- **Biggest gotcha:** enforcing a rule only in application code does not protect you from race
  conditions (two writers both "check then insert" at the same time) or from other
  apps/scripts writing to the same table. The database constraint is the only place that check
  and write happen atomically together.
