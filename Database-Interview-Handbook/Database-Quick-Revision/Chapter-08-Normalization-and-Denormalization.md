# Chapter 8 — Normalization & Denormalization

Interviewers ask this to check if you can spot duplicated data and reason about the read/write
trade-off it creates. This chapter covers normal forms (1NF, 2NF, 3NF) and when to break them on
purpose.

## The Idea

**Normalization** means organizing your tables so each piece of data is stored in only one place.
The goal: remove duplicated data, so you never have two copies of the same fact that can drift
apart.

Start with a bad table, one that repeats customer info on every order:

| order_id | customer_id | customer_name | customer_address | product   |
|----------|-------------|----------------|-------------------|-----------|
| 1        | 101         | Asha Rao       | Pune, India       | Keyboard  |
| 2        | 101         | Asha Rao       | Pune, India       | Mouse     |
| 3        | 102         | Ravi Shah      | Delhi, India      | Monitor   |

Asha's name and address are stored twice. If she moves to Mumbai, you must update every row that
mentions her, or her address becomes inconsistent across rows. That is the problem normalization
fixes.

**1NF — First Normal Form: atomic values, no repeating groups.**
Every column must hold one single value, not a list. Suppose an order row stored products as
`"Keyboard, Mouse"` in one cell. That breaks 1NF — you cannot easily search, count, or join on a
value packed inside a string. Fix: one row per product, like the table above already does.

**2NF — Second Normal Form: no partial dependency on part of a composite key.**
This applies only when a table has a **composite key** (a primary key made of two or more
columns). Say we have an `order_items(order_id, product_id, product_name, quantity)` table, where
the key is `(order_id, product_id)`. `quantity` depends on both columns (how many of *this*
product in *this* order — correct). But `product_name` depends only on `product_id`, not on
`order_id`. That is a **partial dependency** — a non-key column depending on only part of the
composite key. Fix: move `product_name` to a separate `products(product_id, product_name)` table.

**3NF — Third Normal Form: no transitive dependency.**
A non-key column must depend only on the key — not on another non-key column. Back to our orders
table: `customer_address` depends on `customer_id`, and `customer_id` is what identifies the
customer on the order — but `customer_address` does not describe the order itself, it describes
the customer. This chain (order → customer_id → customer_address) is a **transitive
dependency**. Fix: move customer details into their own `customers(customer_id, customer_name,
customer_address)` table. The orders table then just stores `customer_id` as a foreign key
(Chapter 1).

After fixing 1NF, 2NF, and 3NF, our example becomes three clean tables:

`customers(customer_id, customer_name, customer_address)`
`orders(order_id, customer_id, order_date)`
`order_items(order_id, product_id, quantity)`
`products(product_id, product_name)`

Now Asha's address lives in exactly one row. Change it once, and every order automatically
"sees" the new address, because orders only store her `customer_id`, not a copy of her address.

**Denormalization** is the opposite move, done on purpose: you add some duplicated data back into
a table, to make reads faster. For example, you might add `customer_name` back onto the `orders`
table, so a page that lists "recent orders" can read one table instead of joining `orders` to
`customers` every time.

## Why It Matters — the Reasoning

**Why normalize:** duplicated data creates an **update anomaly** — a bug where changing one real-
world fact requires updating it in many places, and if you miss one row, your data becomes
inconsistent (some rows say Pune, some say Mumbai, and now which one is true?). Normalization
avoids this by storing each fact in exactly one place. A write (update Asha's address) touches
exactly one row. Writes stay simple and safe, and your data never contradicts itself.

**The cost of normalizing:** to answer a question like "list all orders with customer name and
address," you now need to **join** three or four tables together. Joins cost CPU time — the
database must match rows across tables. On a small table this is instant. On tables with millions
of rows, and several joins stacked together, it can slow reads down a lot, especially on a hot
read path (a query that runs very often, like a page every user loads).

**Why denormalize:** if a read path is hit constantly and joins are the bottleneck, you can copy
some data into the table you read from, so the read becomes a single-table lookup — no join
needed. Reads become fast. But now you pay the price normalization was trying to avoid: the same
fact (customer name) lives in two places (`customers` and `orders`). If Asha changes her name,
you must update both tables, or the `orders` copy goes stale. This is exactly the update anomaly
risk, reintroduced on purpose, in exchange for read speed.

**Seeing the anomaly happen — a concrete walk-through.** Say you denormalize by adding
`customer_name` straight onto `orders`, to skip the join:

| order_id | customer_id | customer_name | product   |
|----------|-------------|----------------|-----------|
| 1        | 101         | Asha Rao       | Keyboard  |
| 2        | 101         | Asha Rao       | Mouse     |

Now Asha gets married and updates her name to "Asha Mehta." Your application runs:

```sql
UPDATE orders SET customer_name = 'Asha Mehta' WHERE order_id = 1;
```

It only touched order 1, because that is the row the "edit name" screen happened to load. Order 2
still says "Asha Rao." Now a report that lists her order history shows two different names for
the same person, on the same page. That inconsistency is the update anomaly, in front of your
eyes. The fix is not to avoid denormalization forever — it is to always update `customer_name` in
every `orders` row (or better, in the same transaction that updates `customers`, or through one
shared code path that never forgets), so the copies do not drift apart. This is the real cost you
are signing up for when you denormalize: discipline around keeping copies in sync, forever, for
as long as that duplicated column exists.

**A middle ground worth knowing:** you do not have to choose one extreme for the whole database.
Most real systems normalize almost everything (correctness matters everywhere), and denormalize
just one or two columns on just one or two tables — the ones actually proven to be hot and
join-heavy. This keeps the anomaly risk small and contained, instead of spread across the whole
schema.

**The trade-off, stated plainly:**
- **Normalized:** clean writes (update one place), safe from anomalies, but reads may need many
  joins — slower on large, join-heavy queries.
- **Denormalized:** fast reads (no joins, one table), but writes get harder (update every copy) and
  you risk copies going out of sync if you forget one.

**When to choose which:** normalize by default. It is the safer starting point — your data stays
correct with minimal effort, and most queries on a reasonably sized table are fast enough even
with a join or two. Denormalize only for specific, proven hot read paths — a query you have
measured to be slow, and that runs often enough to be worth the trouble. When you do denormalize,
have a clear plan for keeping copies in sync: either update both places in the same transaction
(Chapter 3), or use a background job, or accept the copy is "slightly stale" if your business
can tolerate that (for example, a "product view count" shown with a few seconds of delay is
usually fine).

## Common Interview Questions

**1. What is normalization? Give an example.**
Organizing tables so each fact is stored once, to remove duplicated data. Example: instead of
repeating `customer_name` and `customer_address` on every order row, store them once in a
`customers` table, and have `orders` reference `customer_id`. Now the address lives in one place.

**2. What problem does normalization solve?**
It solves the **update anomaly**: when the same fact is duplicated across rows, updating it means
finding and updating every copy. Miss one, and your data contradicts itself (two different
addresses for the same customer). Normalization stores each fact once, so one update is always
enough, and the data can never disagree with itself.

**3. What is the difference between 2NF and 3NF?**
2NF is about a composite key: no column should depend on only part of the key (a **partial
dependency**). It only matters when your primary key has two or more columns. 3NF is about
non-key columns depending on each other instead of on the key (a **transitive dependency**) — it
applies even with a single-column key. Simple way to remember: 2NF asks "does this column need
the *whole* key, or just part of it?" 3NF asks "does this column depend on the key, or secretly
on another non-key column instead?"

**4. When would you denormalize?**
When you have a specific, measured hot read path that joins are slowing down, and the extra read
speed is worth the cost of keeping duplicated data in sync. Example: a dashboard that shows order
totals per customer thousands of times a minute — you might store a running `total_spent` on the
`customers` table instead of summing `orders` every time, updating it whenever a new order is
placed.

**5. Is a fully normalized database always best?**
No. Full normalization minimizes duplication, but it can mean many small tables and many joins
for common queries. If a query is run constantly and joins are provably the bottleneck,
some denormalization can be the right, deliberate trade-off. "Always fully normalized" is not
a rule — it is a good *default*, not a law. The right amount of normalization depends on your
actual read and write patterns.

**6. What is an update anomaly?**
A bug caused by duplicated data: when one real-world fact changes, you must update every
duplicate copy of it, and if you miss one, the database ends up storing two different answers
for the same fact (like two addresses for the same customer). It happens because the same fact is
stored in more than one row or table. Normalization prevents it by storing each fact exactly
once; denormalization reintroduces the risk, so it needs a clear plan for keeping copies in sync.

## Quick Recall

- Normalization = store each fact once. Denormalization = duplicate some data on purpose, for
  faster reads.
- 1NF: atomic values, no lists inside a cell. 2NF: no non-key column depends on only part of a
  composite key. 3NF: no non-key column depends on another non-key column (only on the key).
- Normalized: safe, simple writes; possibly slow reads (joins). Denormalized: fast reads; riskier,
  harder writes (must update every copy).
- Normalize by default. Denormalize only for a specific, measured hot read path, with a real plan
  to keep the duplicated copies in sync.
- **Biggest gotcha:** denormalization does not remove the update anomaly risk — it brings it back
  on purpose. Always know how (and when) the duplicated copy gets refreshed.
