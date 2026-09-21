
# Chapter 2 — Indexes: B-tree, Composite, Covering & Selectivity

An index is one of the first things interviewers ask about, because it tests if you understand
the read-vs-write trade-off in a database. This chapter covers the B-tree index, composite
indexes, covering indexes, and why an index can be ignored.

## The Idea

An **index** is a sorted lookup structure that helps the database find rows fast, without
scanning the whole table. Think of it like the index at the back of a book. You look up a word,
get a page number, and jump there. Without it, you would read every page.

**B-tree index.** This is the default index type in PostgreSQL. B-tree stands for "balanced tree."
It keeps values sorted, and it is "balanced," meaning every path from the top of the tree to the
bottom is about the same length. This gives **log-time lookup**: to find a row, the database does
only a few comparisons, not one per row. A table with 1 million rows might need only about 20
steps to find a value, instead of up to 1 million steps for a full scan.

B-tree is good for:
- **Equality lookups**: `WHERE order_id = 105`.
- **Range lookups**: `WHERE order_date BETWEEN '2026-01-01' AND '2026-01-31'`.
- **Sorting**: `ORDER BY order_date` can reuse the sorted index instead of sorting rows at query
  time.

Example table:
```sql
CREATE TABLE orders (
  order_id     INT PRIMARY KEY,
  customer_id  INT,
  order_date   DATE,
  status       VARCHAR(20),
  total_amount NUMERIC
);

CREATE INDEX idx_orders_customer ON orders (customer_id);
```
Now `WHERE customer_id = 42` uses the index instead of scanning every row in `orders`.

**Composite index.** This is an index on more than one column, for example `(customer_id,
order_date)`. The database sorts rows first by `customer_id`, and within each `customer_id`
group, by `order_date`. This is the **left-prefix rule**: the index helps a query only if the
query uses a matching prefix of the columns, starting from the left.

```sql
CREATE INDEX idx_orders_cust_date ON orders (customer_id, order_date);
```
- `WHERE customer_id = 42` — uses the index (uses the first column only).
- `WHERE customer_id = 42 AND order_date = '2026-01-01'` — uses the index fully (uses both
  columns, in order).
- `WHERE order_date = '2026-01-01'` — **does not** use this index well. The index is not sorted
  by `order_date` on its own, so the database cannot jump straight to the matching rows. It would
  need to check the whole index, which is usually not worth it.

Column order matters. If most of your queries filter by `order_date` alone, put `order_date`
first, or create a separate index on it.

**Covering index and index-only scan.** A covering index is an index that includes all the
columns a query needs, not just the columns in the `WHERE` clause. When this happens, the
database can answer the query by reading only the index — it never has to open the actual table
rows. This is called an **index-only scan**, and it is faster because it avoids an extra trip to
the table.

```sql
CREATE INDEX idx_orders_cover ON orders (customer_id, order_date) INCLUDE (total_amount);

SELECT order_date, total_amount FROM orders WHERE customer_id = 42;
```
Here, the index alone holds `customer_id`, `order_date`, and `total_amount`. The query never
touches the `orders` table.

**Index selectivity.** Selectivity means how many distinct values a column has, compared to the
number of rows. A column like `order_id` is highly selective — every value is unique. A column
like `status` (say, only `'pending'`, `'shipped'`, `'delivered'`) is **not** selective — few
distinct values, and many rows share each one.

An index on a low-selectivity column is often not useful. If 40% of all orders have
`status = 'pending'`, an index does not narrow the search much. The database planner will often
skip the index and just scan the whole table, because reading 40% of the table row by row through
an index is slower than one straight scan.

## Why It Matters — the Reasoning

**The core trade-off: faster reads, slower writes, more storage.**
An index is not free. Every `INSERT`, `UPDATE`, or `DELETE` must also update every index on that
table, to keep it sorted and correct. So more indexes mean slower writes and more disk space used.
This is why you should not index every column "just in case." You index columns that are
actually used in `WHERE`, `JOIN`, or `ORDER BY`, on tables where reads matter more than write
speed.

**Why column order matters in a composite index.**
A B-tree composite index is sorted by the first column first, then the second column within each
group of the first, and so on — like a phone book sorted by last name, then first name. You can
jump straight to "Smith," and then to "Smith, John" quickly. But you cannot jump straight to
everyone named "John" across all last names — you would need to check every last-name group. That
is why `(a, b)` helps `WHERE a = ..` and `WHERE a = .. AND b = ..`, but not `WHERE b = ..` alone.

**Why a non-selective index is often useless.**
The point of an index is to narrow down the search fast. If a `WHERE` condition still matches a
large share of the table, walking through the index and then fetching each matching row from the
table (called a "random I/O" pattern) ends up costing more than just reading the whole table in
order (a "sequential scan"). The query planner in PostgreSQL estimates this cost using stored
statistics on column value distribution, and picks whichever plan is cheaper.

**Why the planner may skip your index.**
This surprises many engineers: "I have an index, why is the query still slow?" The honest answer
is that having an index does not force the database to use it. The **query planner** compares the
estimated cost of using the index versus a full table scan, and picks the cheaper one. If most
rows match your filter, a full scan wins. This is a feature, not a bug — but it means low-
selectivity filters, or stale statistics, are common causes of "my index is not being used."

## Common Interview Questions

**Q1: How does a B-tree index work, and why is it fast?**
A B-tree keeps keys sorted and balanced, so every lookup takes about the same, small number of
steps — this is log-time lookup. Instead of checking every row (linear time), the database
follows a few branches down the tree to reach the value. This is why B-tree is great for equality
and range queries, and also helps `ORDER BY` since the data is already sorted.

**Q2: I have an index on `(a, b)`. Which queries actually use it?**
Queries that filter on `a` alone, or on `a` and `b` together, use it well — this is the left-
prefix rule, because the index is sorted by `a` first. A query that filters on `b` alone usually
cannot use this index efficiently, because the index is not sorted by `b` on its own. If `b`-only
lookups are common, add a separate index on `b`, or consider `(b, a)` if that is more common.

**Q3: What is a covering index, and why does it help?**
A covering index holds every column the query needs — both the filter columns and the columns
being selected. This lets the database do an **index-only scan**: it reads the answer straight
from the index and skips the extra step of fetching the row from the table. This cuts down disk
reads, which is the main cost in most queries.

**Q4: Why is my index not being used?**
A few common reasons: the column is not selective (many rows share the same value, so a full scan
is cheaper); the table is small (a full scan of a small table is already fast); the query applies
a function to the column (like `WHERE UPPER(name) = 'X'`, which needs a matching function-based
index); or the planner's statistics are stale, so it misjudges the cost. The reasoning always
comes back to cost: the planner picks whichever plan it thinks reads less data.

**Q5: Do indexes slow down writes? Why?**
Yes. Every `INSERT`, `UPDATE`, or `DELETE` must also update each index on that table, to keep the
sorted structure correct. This is the classic index trade-off: faster reads, slower writes, and
more storage used. This is why write-heavy tables should have only the indexes they truly need.

**Q6: What is the difference between a clustered and a non-clustered index?**
A **clustered index** decides the actual physical order of rows on disk — there can be only one
per table, since rows can only be sorted one way. A **non-clustered index** is a separate sorted
structure that points back to the row's location; a table can have several. PostgreSQL does not
have a true clustered index by default — every index (including the primary key) is non-
clustered, that is, a separate structure from the table (though `CLUSTER` can physically reorder
a table's rows once, as a one-time operation, not something kept up automatically). Databases like
MySQL's InnoDB, however, always store the table itself sorted by the primary key, so the primary
key acts as a clustered index there.

## Quick Recall

- An index is a sorted structure for fast lookup — it avoids scanning the whole table.
- B-tree: sorted, balanced, log-time lookup. Good for equality, range, and `ORDER BY`.
- Composite index `(a, b)` follows the left-prefix rule: helps `WHERE a=..` and
  `WHERE a=.. AND b=..`, not `WHERE b=..` alone. Column order matters.
- Covering index + `INCLUDE`: all needed columns live in the index, so the query can skip the
  table (index-only scan).
- Selectivity: an index on a column with few distinct values (like `status`) is often skipped by
  the planner, because a full scan is cheaper when most rows match.
- The trade-off to always mention: indexes speed up reads, but slow down writes and use more
  storage.
- **Biggest gotcha:** having an index does not guarantee it is used. The query planner picks the
  plan it estimates is cheapest, and a low-selectivity filter often loses to a plain full table
  scan.

*(See Chapter 1 for primary keys, which are usually backed by a unique B-tree index
automatically.)*
