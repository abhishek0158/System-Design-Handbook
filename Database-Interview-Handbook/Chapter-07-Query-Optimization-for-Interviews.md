# Chapter 7 — Query Optimization for Interviews

Interviewers love this task: "Here is a slow query. How do you make it faster?" They want
to see your process, not a memorized trick. This chapter teaches you how to read an
`EXPLAIN ANALYZE` plan and how to fix the slow-query patterns that show up again and again.

## Key Concepts

### `EXPLAIN` vs `EXPLAIN ANALYZE`

`EXPLAIN` shows the **plan** the database will use. It does not run the query. It only
shows guesses (estimates) based on table statistics.

`EXPLAIN ANALYZE` actually **runs** the query, then shows the plan with real numbers next
to the guesses. Use `EXPLAIN ANALYZE` when you need real timings. Add `BUFFERS` to also see
how much data came from cache versus disk:

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT order_id, order_date, total_amount
FROM orders
WHERE customer_id = 501
ORDER BY order_date DESC;
```

Caution: `EXPLAIN ANALYZE` really executes the query. Do not run it on a live
`DELETE` or `UPDATE` in production without wrapping it in a transaction you roll back.

### How to read a plan: bottom-up, inside-out

A plan is a tree of **nodes**. Each node is one step, like "scan this table" or "join these
two results". Postgres prints the tree with indentation. The most deeply indented node runs
**first**. Data flows upward, from the bottom nodes to the top node, which is the final
result.

Example plan:

```
Limit  (cost=8.44..8.44 rows=1 width=24) (actual time=0.05..0.05 rows=1 loops=1)
  ->  Sort  (cost=8.44..8.44 rows=3 width=24) (actual time=0.05..0.05 rows=1 loops=1)
        Sort Key: order_date DESC
        ->  Index Scan using idx_orders_customer on orders
              (cost=0.29..8.42 rows=3 width=24) (actual time=0.02..0.03 rows=3 loops=1)
              Index Cond: (customer_id = 501)
```

Read it as: scan `orders` using the index on `customer_id` first (innermost), then sort the
rows by `order_date`, then keep only the top row for `Limit`. Read from the bottom node
upward to follow the real order of work.

### Cost numbers vs actual time

Each node shows `cost=startup..total`. This is not milliseconds. It is an arbitrary unit
the planner uses to compare plans against each other. A higher cost usually means more
work, but you cannot convert it to seconds directly.

`actual time=startup..total` (only with `ANALYZE`) **is** real milliseconds, averaged
over `loops`. If a node has `loops=50`, its shown `actual time` is per loop, so multiply by
50 to get the total time that node used.

### Estimated rows vs actual rows

`rows=N` in the cost part is the planner's **guess**, based on table statistics. `rows=N`
in the actual part is the **real** count returned by that node when the query ran.

This comparison is one of the most useful things you can point out in an interview:

- Estimated 3, actual 3 → statistics are good, the planner had the right picture.
- Estimated 3, actual 300,000 → statistics are stale or the predicate is hard to estimate
  (for example, a function on a column). This kind of mismatch often causes the planner to
  pick a bad join method, because it thinks one side of the join is tiny when it is not.

### Seq Scan vs Index Scan vs Index-Only Scan vs Bitmap Scan

**Sequential scan (`Seq Scan`)**: reads every row in the table, in storage order, and
checks each one against the filter. Cost grows with table size. Not automatically bad — for
small tables, or when a query needs a large fraction of the rows, a seq scan can beat an
index scan, because reading one big block of disk is cheaper than jumping around randomly.

**Index scan (`Index Scan`)**: walks a B-tree index to find matching rows, then jumps to
the table (the "heap") to fetch the actual row. Good when the predicate matches a small
fraction of rows (high **selectivity** — few rows match, out of the total).

**Index-only scan (`Index Only Scan`)**: same as an index scan, but Postgres can answer the
whole query using only columns stored in the index. It skips the trip to the table. This
needs all selected and filtered columns to be in the index (a "covering index"), and needs
the visibility map to be up to date (recent `VACUUM`).

**Bitmap scan (`Bitmap Index Scan` + `Bitmap Heap Scan`)**: builds a bitmap of matching row
locations from the index, sorts them, then fetches rows from the table in physical order.
Used when an index scan would match too many scattered rows to be efficient one-by-one, but
still fewer rows than a full seq scan would justify. It also lets Postgres combine two
different indexes with `AND` or `OR` (a `BitmapAnd` / `BitmapOr` node).

| Scan type | Reads table rows? | Best when |
|---|---|---|
| Seq Scan | Yes, all rows | Small table, or predicate matches most rows |
| Index Scan | Yes, one by one | Predicate is very selective (few matching rows) |
| Index Only Scan | No (index has all needed columns) | Covering index exists, table is vacuumed |
| Bitmap Heap Scan | Yes, in batch, sorted by location | Medium selectivity, or combining two indexes |

### Join methods

**Nested Loop**: for each row in the outer input, scan the inner input (often using an
index) to find matches. Good when the outer side is small, and the inner side has a useful
index. Bad when both sides are large and there is no index, because it becomes
row-count-of-outer × row-count-of-inner work.

**Hash Join**: build an in-memory hash table from the smaller input (on the join key), then
scan the larger input and probe the hash table. Good for equality joins (`=`) between two
large sets with no useful index. Needs enough `work_mem` to hold the hash table, or it
spills to disk and gets slower.

**Merge Join**: both inputs must be sorted on the join key (either they already are, from an
index, or Postgres adds a `Sort` node). Then it walks both sorted lists together, like a
zipper. Good when both sides are already sorted, or the query also needs `ORDER BY` on the
same column.

The planner picks the join method based on estimated row counts and available indexes. If
statistics are wrong, it can pick nested loop when hash join would have been much faster
(or the reverse).

### Why a query is slow: the common causes

**1. Missing index.** The plan shows a `Seq Scan` with a `Filter:` line that removes most
rows. This is the classic "add an index" case.

**2. Non-sargable predicate.** "Sargable" means the predicate can use an index directly
("Search ARGument ABLE"). Wrapping the indexed column in a function or a type cast breaks
this, because Postgres cannot look up "the result of a function" in a plain B-tree index
built on the raw column:

```sql
-- Non-sargable: index on customers(name) cannot be used
WHERE lower(name) = 'anita sharma'

-- Non-sargable: index on orders(order_date) cannot be used
WHERE order_date::text = '2026-01-01'
```

Fix by rewriting the predicate so the column is bare, or by indexing the expression itself
(an "expression index"):

```sql
CREATE INDEX idx_customers_name_lower ON customers (lower(name));
-- now WHERE lower(name) = 'anita sharma' can use the index
```

**3. `SELECT *`.** Pulling every column forces a trip to the table heap even when an
index-only scan was possible, and wastes I/O and network bandwidth on columns nobody reads.
Select only the columns you need.

**4. Leading wildcard `LIKE '%text'`.** A plain B-tree index stores values in sorted order,
so it can jump straight to `'abc%'` (prefix match). It cannot use that order to find
`'%abc'` (the match could be anywhere), so Postgres falls back to a seq scan. Fix with a
trigram index from the `pg_trgm` extension, or a full-text search index, if this search
pattern is common:

```sql
CREATE EXTENSION IF NOT EXISTS pg_trgm;
CREATE INDEX idx_products_name_trgm ON products USING gin (name gin_trgm_ops);
-- now WHERE name LIKE '%phone%' can use the index
```

**5. `OR` across different columns.** `WHERE customer_id = 5 OR product_id = 10` cannot
use one single-column index efficiently, because each side of the `OR` needs a different
index. Postgres can sometimes combine two index scans with a `BitmapOr` node, but often it
just gives up and scans the whole table. A safer rewrite, when each side is selective on
its own, is a `UNION`:

```sql
SELECT * FROM orders WHERE customer_id = 5
UNION
SELECT * FROM orders WHERE product_id = 10;  -- example only; product_id is not on orders
```

(In the real e-commerce schema, `OR` conditions on the same table usually look like
`WHERE status = 'pending' OR status = 'delivered'`. Here, rewrite as `status IN ('pending',
'delivered')`. This is not the problematic case — `IN` on one column can use one index
fine. The problematic case is `OR` across **different columns**.)

**6. Bad statistics.** Postgres decides on a plan using stored statistics (row counts,
common values, histograms), refreshed by `ANALYZE` (usually run automatically by
autovacuum). If a table changed a lot recently (bulk load, bulk delete) and has not been
analyzed, the planner's row estimates are wrong, and it can pick a bad join method or skip
a good index. Fix by running `ANALYZE table_name;` manually, or checking
`SELECT * FROM pg_stat_user_tables WHERE relname = 'orders';` for the last `analyze` time.

### Which index to add, and column order

For a composite (multi-column) index, put **equality** columns before **range** columns.
An index is a sorted structure. Once a range condition is applied, the remaining columns in
the index are no longer sorted in a way that helps.

```sql
-- Query: WHERE customer_id = 501 AND order_date > '2026-01-01'
CREATE INDEX idx_orders_cust_date ON orders (customer_id, order_date);
-- Good: equality column (customer_id) first, range column (order_date) second
```

If the query also sorts, matching that column's direction in the index avoids an extra
`Sort` node:

```sql
-- Query: WHERE customer_id = 501 ORDER BY order_date DESC
CREATE INDEX idx_orders_cust_date_desc ON orders (customer_id, order_date DESC);
```

Between two equality columns, put the more **selective** one first (the one that filters
out more rows), so fewer rows need checking against the second condition. See Chapter 8
(Indexing) for a deeper look at index internals and index types.

### When NOT to add an index

Every index has a cost. It is not free:

- **Write cost.** Every `INSERT`, `UPDATE`, or `DELETE` must also update every index on
  that table. A table with 8 indexes is much slower to write to than one with 2.
- **Disk space.** Each index is stored separately and takes its own space, sometimes as
  much as the table itself.
- **Low selectivity.** An index on a boolean column, or a status column with only 3
  possible values spread evenly, often does not help. The planner may still choose a seq
  scan, because reading half the table through an index (with random jumps) can cost more
  than reading it all sequentially.
- **Small table.** If a table fits in a few disk pages, a seq scan is already fast. An
  index adds overhead with no benefit.
- **Redundant index.** An index on `(customer_id)` is redundant if you already have one on
  `(customer_id, order_date)` — the composite index already serves any query that filters
  on `customer_id` alone (the "leftmost prefix" rule).
- **Rarely-run query.** If a slow query runs once a month in a report, it may not be worth
  the ongoing write cost of a new index just to speed up that one report.

Always say this trade-off out loud in an interview: indexes speed up reads but slow down
writes and use disk space. The right answer is often "it depends on the read/write ratio of
this table."

## The Questions They Ask

**Q1: "Here is a slow query. Walk me through how you would speed it up."**

This is the main task in this chapter. Below is a full worked example on the e-commerce
schema (`customers`, `orders`, `order_items`, `products`).

Slow query: get the 20 most recent delivered orders for customers in India.

```sql
SELECT *
FROM orders o
JOIN customers c ON c.customer_id = o.customer_id
WHERE lower(c.country) = 'india'
  AND o.status = 'delivered'
ORDER BY o.order_date DESC
LIMIT 20;
```

Step 1 — run `EXPLAIN ANALYZE` and read the plan. Suppose it shows:

```
Limit (actual time=420.11..420.13 rows=20 loops=1)
  ->  Sort (actual time=420.10..420.11 rows=20 loops=1)
        Sort Key: o.order_date DESC
        ->  Hash Join (actual time=12.40..410.02 rows=8500 loops=1)
              Hash Cond: (o.customer_id = c.customer_id)
              ->  Seq Scan on orders o (rows=250000, actual rows=250000)
                    Filter: (status = 'delivered')
              ->  Hash
                    ->  Seq Scan on customers c (rows=1000, actual rows=1000)
                          Filter: (lower(country) = 'india')
```

Step 2 — spot the problems, one by one:

- `Seq Scan on customers` with `Filter: (lower(country) = 'india')` — this is a
  **non-sargable predicate**. Even if an index existed on `country`, `lower(country)`
  cannot use it.
- `Seq Scan on orders` with `Filter: (status = 'delivered')` scans all 250,000 rows — a
  **missing index** on `status`, or better, on `(status, order_date)` since the query also
  sorts by `order_date`.
- `SELECT *` pulls every column from both tables, including ones the caller may not need.
- A `Sort` node runs after the join, because nothing returned the rows pre-sorted.

Step 3 — fix each one:

```sql
-- Fix the non-sargable predicate with an expression index
CREATE INDEX idx_customers_country_lower ON customers (lower(country));

-- Fix the missing index, ordered for the equality + sort pattern
CREATE INDEX idx_orders_status_date ON orders (status, order_date DESC);

-- Rewrite the query to select only needed columns and use the bare predicate form
SELECT o.order_id, o.order_date, o.total_amount, c.name
FROM orders o
JOIN customers c ON c.customer_id = o.customer_id
WHERE lower(c.country) = 'india'
  AND o.status = 'delivered'
ORDER BY o.order_date DESC
LIMIT 20;
```

Step 4 — re-run `EXPLAIN ANALYZE`. Expect to see `Index Scan` (or `Bitmap Heap Scan`) on
`orders` using `idx_orders_status_date`, already sorted by `order_date DESC`, so the `Sort`
node disappears. The `Hash Join` may now use a smaller, already-filtered set of India
customers from the new expression index. Total actual time should drop sharply — say, from
420 ms to under 5 ms.

Step 5 — say the trade-off out loud: the two new indexes add write overhead to `orders` and
`customers`. If `orders` is inserted into thousands of times per second, weigh that cost
against how often this "recent delivered orders by country" query actually runs.

**Q2: "The plan shows the planner picked a Seq Scan even though there's an index. Why?"**

Usually one of: (a) the predicate matches a large fraction of the table, so a seq scan is
genuinely cheaper; (b) statistics are stale, so the planner underestimates or overestimates
selectivity; (c) the predicate is non-sargable (function or cast on the column); (d) the
table is small enough that a seq scan is always cheaper than an index scan plus heap fetch.

**Q3: "What is the difference between cost and actual time in EXPLAIN ANALYZE?"**

`cost` is the planner's estimate in arbitrary units, used only to compare candidate plans.
`actual time` is real milliseconds measured by running the query. You can only get `actual
time` from `EXPLAIN ANALYZE`, not plain `EXPLAIN`.

**Q4: "How would you speed up a search like `WHERE name LIKE '%phone%'`?"**

Explain that a leading wildcard cannot use a normal B-tree index, because the index is
sorted by prefix. Propose a trigram index (`pg_trgm`) or full-text search
(`tsvector`/`tsquery`) if this kind of search is common. If the search is always a prefix
match (`'phone%'`), a plain B-tree index already works — no extension needed.

**Q5: "Would you always add an index if a query is slow?"**

No. State the trade-off: check the table's write volume, the table's size, the
selectivity of the column, and whether an existing composite index already covers it as a
prefix. If the query runs rarely, or the table is tiny, or the column has very few distinct
values, the index may not pay for itself. This "it depends" answer, backed by reasoning, is
what interviewers want to hear — see the framing note in Chapter 1.

**Q6 (follow-up): "The query got faster after adding the index, but writes to the table got
slower. What do you say to the team?"**

Quantify both sides if possible: how much read time was saved, how many writes per second
hit the table, and how much slower each write became. Suggest alternatives before accepting
the cost: a partial index (`CREATE INDEX ... WHERE status = 'delivered'`) if only one status
value is queried often, or a covering index that also enables an index-only scan.

## Rapid-Fire

- **What does `EXPLAIN ANALYZE` do that plain `EXPLAIN` does not?** It actually runs the
  query and shows real row counts and real time, not just estimates.
- **Seq Scan vs Index Scan — which is always better?** Neither. Seq scan wins for small
  tables or low-selectivity predicates; index scan wins for high-selectivity predicates.
- **What is an index-only scan?** A scan that answers the query using only the index,
  without visiting the table, because the index has all needed columns.
- **What makes a predicate non-sargable?** Wrapping the indexed column in a function or a
  type cast, so a plain index on the raw column cannot be used.
- **Fix for `WHERE lower(col) = 'x'`?** Create an expression index: `CREATE INDEX ON
  table (lower(col));`.
- **Fix for `WHERE col LIKE '%text%'`?** Use a trigram index (`pg_trgm`) or full-text
  search; a plain B-tree index cannot help with a leading wildcard.
- **Why is `SELECT *` a problem?** It can force a table lookup even when an index-only scan
  was possible, and it wastes I/O on unused columns.
- **Composite index column order rule?** Equality columns first, then range columns; match
  `ORDER BY` direction if the query sorts.
- **When does `OR` hurt performance?** When it spans different columns with separate
  single-column indexes; a `UNION` of two indexed queries is often faster.
- **What causes bad plan choices from good indexes?** Stale statistics; fix with
  `ANALYZE table_name;`.
- **Give one reason not to add an index.** High write volume on that table — every write
  must also update every index.
- **Nested Loop vs Hash Join — when does the planner prefer each?** Nested loop for a small
  outer input with an indexed inner input; hash join for large equality joins without a
  useful index.
- **What is a covering index?** An index that includes every column a query needs, so
  Postgres can skip the table lookup (index-only scan).

## Common Traps & Mistakes

- **Reading a plan top-down instead of bottom-up.** The topmost node is the final output,
  not the first thing that ran. Always find the deepest, most indented node and read
  upward.
- **Confusing cost with time.** Cost is an arbitrary unit for comparing plans, not
  milliseconds. Only `actual time` (from `EXPLAIN ANALYZE`) is real time.
- **Assuming an index is unused because of a bug.** Often the planner is right to skip it —
  check selectivity and table size before assuming something is broken.
- **Forgetting that `loops` scales `actual time`.** If a node ran 1,000 times, its shown
  `actual time` is the average per loop; multiply to get the real total contribution.
- **Adding a single-column index when a composite index would serve more queries.** Check
  existing indexes first — a new composite index might make an old single-column index
  redundant.
- **Wrong column order in a composite index.** Putting a range column before an equality
  column breaks the sort order the rest of the index needs.
- **Forgetting `ANALYZE` after a bulk load.** Fresh data with stale statistics can cause the
  planner to make bad row-count guesses and pick a slow plan.
- **Adding indexes without mentioning the write-cost trade-off.** Interviewers often push
  back with "would you always do this?" — say the trade-off before they ask.
- **Ignoring `LIMIT` when a query still scans everything.** A `LIMIT` alone does not stop a
  `Sort` node from sorting the full result set first, unless an index already provides the
  needed order.
- **Treating `IN (...)` the same as cross-column `OR`.** `IN` on one column is fine for
  indexes; `OR` across different columns is the pattern that needs a rewrite.
