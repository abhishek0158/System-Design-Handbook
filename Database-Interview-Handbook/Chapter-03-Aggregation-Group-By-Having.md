# Chapter 3 — Aggregation, GROUP BY & HAVING

Aggregation turns many rows into one summary number, like a total or an average. Interviewers
test this because almost every real report ("revenue by category", "orders per customer") needs
it. This chapter covers aggregate functions, the `GROUP BY` rule, `WHERE` vs `HAVING`, and
conditional aggregation.

## Key Concepts

### 1. Aggregate functions

An **aggregate function** takes many rows and returns one value. The five common ones are:

| Function | What it does |
|---|---|
| `COUNT()` | Counts rows |
| `SUM()` | Adds up numbers |
| `AVG()` | Average of numbers |
| `MIN()` | Smallest value |
| `MAX()` | Largest value |

Example: total revenue and average order value across all orders.

```sql
SELECT
    COUNT(*)          AS total_orders,
    SUM(total_amount)  AS total_revenue,
    AVG(total_amount)  AS avg_order_value,
    MIN(total_amount)  AS smallest_order,
    MAX(total_amount)  AS biggest_order
FROM orders;
```

Without `GROUP BY`, the whole table is treated as **one group**. So this query returns exactly
one row, no matter how many rows are in `orders`.

### 2. COUNT(*) vs COUNT(column) vs COUNT(DISTINCT column)

This is a favorite interview trap because each form treats `NULL` differently. A `NULL` means
"no value here."

- `COUNT(*)` counts **all rows**, including rows where every column is `NULL`. It does not look
  at any specific column.
- `COUNT(column)` counts rows where that **column is not NULL**. It skips `NULL` values.
- `COUNT(DISTINCT column)` counts the **unique, non-NULL values** in that column.

Example on `orders`, where `status` can sometimes be `NULL` (not yet set):

```sql
SELECT
    COUNT(*)                    AS all_rows,
    COUNT(status)                AS rows_with_status,
    COUNT(DISTINCT status)       AS distinct_statuses,
    COUNT(DISTINCT customer_id)  AS unique_customers
FROM orders;
```

If `orders` has 100 rows, 5 of them with `status = NULL`, and 4 distinct non-null status values
(`pending`, `shipped`, `delivered`, `cancelled`):

- `all_rows` = 100
- `rows_with_status` = 95
- `distinct_statuses` = 4
- `unique_customers` = however many different `customer_id` values exist

Rule to remember: **`COUNT` never counts `NULL` as a value**, except `COUNT(*)`, which does not
care about `NULL` at all because it counts rows, not values.

### 3. The GROUP BY rule

`GROUP BY` splits rows into groups that share the same value in one or more columns, then runs
aggregate functions **per group** instead of over the whole table.

**The rule:** every column in `SELECT` that is not inside an aggregate function must appear in
`GROUP BY`. Postgres enforces this and throws an error if you break it. This is because for each
group, a plain (non-aggregated) column could have many different values — the database does not
know which one to show, so it refuses to guess.

Example: revenue per product category.

```sql
SELECT
    p.category,
    SUM(oi.quantity * oi.unit_price) AS category_revenue
FROM order_items oi
JOIN products p ON p.product_id = oi.product_id
GROUP BY p.category
ORDER BY category_revenue DESC;
```

Here `category` is in `GROUP BY`, and `category_revenue` is an aggregate. That is valid.

If you tried to also select `p.name` without grouping by it:

```sql
-- This FAILS in Postgres:
SELECT p.category, p.name, SUM(oi.quantity * oi.unit_price) AS revenue
FROM order_items oi
JOIN products p ON p.product_id = oi.product_id
GROUP BY p.category;
```

Postgres raises: `column "p.name" must appear in the GROUP BY clause or be used in an aggregate
function`. Fix it by adding `p.name` to `GROUP BY`, or by removing it from `SELECT`, or by
wrapping it in an aggregate like `MAX(p.name)` if you just want any one value.

### 4. WHERE vs HAVING

Both filter rows, but at **different stages** of query execution.

- `WHERE` filters **individual rows**, **before** grouping happens. It cannot use aggregate
  functions, because aggregates do not exist yet at this stage.
- `HAVING` filters **groups**, **after** grouping and aggregation happen. It is built to use
  aggregate functions like `SUM()` or `COUNT()`.

Think of the order of steps in a `GROUP BY` query like this:

```
FROM / JOIN  →  WHERE  →  GROUP BY  →  HAVING  →  SELECT  →  ORDER BY
```

Example: find categories with total revenue above 10,000, but only counting orders that are not
cancelled.

```sql
SELECT
    p.category,
    SUM(oi.quantity * oi.unit_price) AS category_revenue
FROM order_items oi
JOIN products p ON p.product_id = oi.product_id
JOIN orders o ON o.order_id = oi.order_id
WHERE o.status <> 'cancelled'          -- filters rows, before grouping
GROUP BY p.category
HAVING SUM(oi.quantity * oi.unit_price) > 10000   -- filters groups, after aggregation
ORDER BY category_revenue DESC;
```

`WHERE o.status <> 'cancelled'` removes cancelled order rows first. Then Postgres groups the
remaining rows by `category` and sums them. Only after that does `HAVING` throw away any category
whose total is 10,000 or less.

A common mistake: writing `WHERE SUM(...) > 10000`. This fails, because `WHERE` runs before
`SUM()` is computed — the aggregate does not exist yet at that point.

### 5. Grouping by more than one column

You can group by two or more columns. Each **unique combination** of those columns becomes one
group. This is useful for reports broken down by two dimensions.

Example: order count and total revenue, per customer per year.

```sql
SELECT
    o.customer_id,
    EXTRACT(YEAR FROM o.order_date) AS order_year,
    COUNT(*)                         AS order_count,
    SUM(o.total_amount)              AS total_spent
FROM orders o
GROUP BY o.customer_id, EXTRACT(YEAR FROM o.order_date)
ORDER BY o.customer_id, order_year;
```

Each row in the result is one `(customer_id, order_year)` pair. If customer 5 placed orders in
both 2024 and 2025, that customer gets two rows — one per year.

### 6. Conditional aggregation

Sometimes you need to count or sum only rows that meet a condition, but you want the result as a
**column**, not a filtered row. The classic pattern is `SUM(CASE WHEN ... THEN 1 ELSE 0 END)`.
`CASE WHEN` is SQL's if/else expression: it checks a condition and returns one value if true,
another if false.

Example: for each customer, count how many orders are `delivered` and how many are `cancelled`,
side by side.

```sql
SELECT
    o.customer_id,
    COUNT(*) AS total_orders,
    SUM(CASE WHEN o.status = 'delivered' THEN 1 ELSE 0 END) AS delivered_count,
    SUM(CASE WHEN o.status = 'cancelled' THEN 1 ELSE 0 END) AS cancelled_count
FROM orders o
GROUP BY o.customer_id;
```

Postgres also supports the `FILTER` clause, which does the same thing but reads more cleanly:

```sql
SELECT
    o.customer_id,
    COUNT(*) AS total_orders,
    COUNT(*) FILTER (WHERE o.status = 'delivered') AS delivered_count,
    COUNT(*) FILTER (WHERE o.status = 'cancelled') AS cancelled_count
FROM orders o
GROUP BY o.customer_id;
```

`FILTER (WHERE ...)` tells `COUNT` to only look at rows matching that condition, for that one
aggregate. It is Postgres-specific.

> **MySQL note:** MySQL does not support `FILTER`. Use
> `SUM(CASE WHEN condition THEN 1 ELSE 0 END)` instead — it works in both Postgres and MySQL.

### 7. GROUPING SETS and ROLLUP (short note)

Sometimes you want subtotals **and** a grand total in one query, instead of running several
separate queries. `ROLLUP` and `GROUPING SETS` do this.

```sql
SELECT
    p.category,
    o.status,
    SUM(oi.quantity * oi.unit_price) AS revenue
FROM order_items oi
JOIN products p ON p.product_id = oi.product_id
JOIN orders o ON o.order_id = oi.order_id
GROUP BY ROLLUP (p.category, o.status);
```

`ROLLUP (category, status)` produces: one row per `(category, status)` pair, then one subtotal
row per `category` (with `status` shown as `NULL`), then one grand-total row (both `NULL`). This
is handy for dashboards. Interviewers rarely ask you to write `ROLLUP` from scratch at the 3–4 YOE
level, but knowing it exists — and that the `NULL` in the output means "this is a subtotal row,"
not "missing data" — is a good signal.

### 8. Worked example: average salary per department (HR schema)

```sql
SELECT
    d.dept_name,
    COUNT(e.emp_id)   AS headcount,
    AVG(e.salary)     AS avg_salary,
    MAX(e.salary)     AS top_salary
FROM employees e
JOIN departments d ON d.dept_id = e.dept_id
GROUP BY d.dept_name
ORDER BY avg_salary DESC;
```

This joins `employees` to `departments` (see Chapter 2 for join details), then groups by
department name. Each department becomes one row, with its headcount, average salary, and top
salary.

### 9. GROUPING SETS in full

`GROUPING SETS` is the more general form behind `ROLLUP`. It lets you list exactly which
groupings you want, in one query, instead of running separate `GROUP BY` queries and combining
them with `UNION ALL`.

```sql
SELECT
    p.category,
    o.status,
    SUM(oi.quantity * oi.unit_price) AS revenue
FROM order_items oi
JOIN products p ON p.product_id = oi.product_id
JOIN orders o ON o.order_id = oi.order_id
GROUP BY GROUPING SETS (
    (p.category, o.status),  -- revenue per category and status
    (p.category),             -- revenue per category, all statuses combined
    ()                         -- grand total, everything combined
);
```

Each set of parentheses is one grouping. Postgres runs the aggregation once and produces rows for
all three groupings together, which is faster than three separate queries scanning the same data.
`ROLLUP (category, status)` is just shorthand for the `GROUPING SETS` above, when you want a
"drill-down" order (full detail, then subtotal by the first column, then grand total).

### 10. A short note on performance

`GROUP BY` needs to bring together all rows that share the same group value, so Postgres must
either **sort** the rows by the group columns first, or build an in-memory **hash table** keyed by
the group columns. You can see which one it picked by running `EXPLAIN ANALYZE` on the query:
Postgres shows `GroupAggregate` (sort-based) or `HashAggregate` (hash-based) in the plan.

An index on the `GROUP BY` columns can help, because Postgres can then read rows already sorted by
that column, skipping the separate sort step. This matters most on large tables. Chapter 7 covers
`EXPLAIN ANALYZE` and index-driven optimization in more depth — for the SQL round, it is usually
enough to know that grouping is not "free," and that indexes are not only for `WHERE` clauses.

### 11. Worked example: average order value per country

```sql
SELECT
    c.country,
    COUNT(o.order_id)        AS order_count,
    AVG(o.total_amount)      AS avg_order_value
FROM customers c
JOIN orders o ON o.customer_id = c.customer_id
GROUP BY c.country
ORDER BY avg_order_value DESC;
```

This answers: "which country's customers place the highest-value orders, on average?" It groups
by `country`, so all customers from the same country fall into one group.

## The Questions They Ask

**Q1: What is the difference between `WHERE` and `HAVING`? Why can't you just use `WHERE` for
everything?**
`WHERE` filters rows before grouping. `HAVING` filters groups after aggregation. You cannot use an
aggregate function like `SUM()` or `COUNT()` inside `WHERE`, because at the point `WHERE` runs,
the rows have not been grouped or aggregated yet. `HAVING` runs later, after `GROUP BY`, so it can
see and filter on aggregate results.
*Follow-up:* "Can `HAVING` be used without `GROUP BY`?" Yes. If there is no `GROUP BY`, the whole
table is one group, so `HAVING` filters that single group (rarely useful, but valid).

**Q2: Write a query to find customers who placed more than 5 orders.**
```sql
SELECT customer_id, COUNT(*) AS order_count
FROM orders
GROUP BY customer_id
HAVING COUNT(*) > 5;
```
*Follow-up:* "What if I only want to count orders from 2025?" Add a `WHERE order_date >=
'2025-01-01' AND order_date < '2026-01-01'` before the `GROUP BY`. Filter rows first with `WHERE`,
then count what is left.

**Q3: This query throws an error — why? How do you fix it?**
```sql
SELECT customer_id, order_date, COUNT(*)
FROM orders
GROUP BY customer_id;
```
`order_date` is selected as a plain column but is not aggregated and not in `GROUP BY`. Postgres
does not know which `order_date` to show for a customer with many orders, so it errors. Fix by
either adding `order_date` to `GROUP BY` (this changes the grouping — now each `customer_id` +
`order_date` pair is its own group), or removing `order_date` from `SELECT`, or wrapping it in an
aggregate such as `MAX(order_date)`.
*Follow-up:* "Does MySQL also error here?" By default, modern MySQL (5.7+) also errors, because of
the `ONLY_FULL_GROUP_BY` mode, which is on by default. Older MySQL versions silently picked an
arbitrary row — a common source of production bugs, so Postgres's strictness is actually safer.

**Q4: How do you count how many orders are `pending` vs `delivered`, for each customer, in a
single query?**
Use conditional aggregation:
```sql
SELECT
    customer_id,
    COUNT(*) FILTER (WHERE status = 'pending')   AS pending_count,
    COUNT(*) FILTER (WHERE status = 'delivered') AS delivered_count
FROM orders
GROUP BY customer_id;
```
*Follow-up:* "Why not just run two separate queries with `WHERE status = 'pending'` and `WHERE
status = 'delivered'`?" That works too, but it needs two round trips (or a `UNION`), and you lose
the side-by-side comparison per customer in one row. Conditional aggregation gives one row per
customer with both numbers.

**Q5: What does `COUNT(DISTINCT customer_id)` do differently from `COUNT(customer_id)`?**
`COUNT(customer_id)` counts every non-null row — if one customer placed 10 orders, that customer
is counted 10 times. `COUNT(DISTINCT customer_id)` counts each unique customer only once, no
matter how many orders they placed. Use `COUNT(DISTINCT ...)` when you want "how many unique
customers," not "how many order rows."

**Q6: Can you `GROUP BY` a column that is not in the `SELECT` list?**
Yes. `GROUP BY` and `SELECT` do not have to list the exact same columns — `GROUP BY` can include
extra columns not shown in the output. But the reverse is not allowed: every non-aggregated
`SELECT` column must be in `GROUP BY`.

**Q7: How would you find the top-selling product category by revenue, but only among categories
with more than 50 orders?**
```sql
SELECT
    p.category,
    COUNT(DISTINCT o.order_id)        AS order_count,
    SUM(oi.quantity * oi.unit_price)  AS revenue
FROM order_items oi
JOIN products p ON p.product_id = oi.product_id
JOIN orders o ON o.order_id = oi.order_id
GROUP BY p.category
HAVING COUNT(DISTINCT o.order_id) > 50
ORDER BY revenue DESC
LIMIT 1;
```
Note `COUNT(DISTINCT o.order_id)`, not `COUNT(*)`. Since joining `order_items` can produce
multiple rows per order (one row per product in that order), plain `COUNT(*)` would overcount
orders. `DISTINCT` corrects for that.

**Q8: You run `EXPLAIN ANALYZE` on a `GROUP BY` query and see `HashAggregate`. What does that
mean, and is it a problem?**
It means Postgres built an in-memory hash table keyed by the group columns, instead of sorting the
rows first. This is usually fine and often faster than sorting, as long as the hash table fits in
memory (controlled by the `work_mem` setting). If it does not fit, Postgres spills part of the
hash table to disk, which slows things down. If you see the plan is slow, check whether an index
on the `GROUP BY` columns lets Postgres switch to a `GroupAggregate` that reads pre-sorted data
instead — but do not assume one is automatically better than the other; always check the actual
plan and timing.
*Follow-up:* "Would adding an index always speed up a `GROUP BY`?" Not always. It depends on data
size, how many distinct groups exist, and whether the index is already useful for the query's
`WHERE` clause too. Say "it depends" and explain the reasoning — that is what interviewers want to
hear.

## Rapid-Fire

- **`COUNT(*)` vs `COUNT(col)`?** `COUNT(*)` counts all rows. `COUNT(col)` skips `NULL` values in
  that column.
- **`COUNT(DISTINCT col)`?** Counts unique, non-null values only.
- **Order of execution for `GROUP BY` queries?** `FROM/JOIN → WHERE → GROUP BY → HAVING → SELECT
  → ORDER BY`.
- **Can `WHERE` use aggregate functions?** No. Aggregates don't exist yet when `WHERE` runs.
- **Can `HAVING` use aggregate functions?** Yes, that is its main purpose.
- **What must every non-aggregated `SELECT` column do?** Appear in `GROUP BY`.
- **How to count rows matching a condition, as one column?** `COUNT(*) FILTER (WHERE cond)` or
  `SUM(CASE WHEN cond THEN 1 ELSE 0 END)`.
- **Does MySQL support `FILTER`?** No. Use `CASE WHEN` there instead.
- **What does `NULL` mean in a `ROLLUP` subtotal row?** "This is a subtotal," not "missing data."
- **`AVG()` and `NULL`s?** `AVG()` ignores `NULL` values; it does not treat them as zero.

## Common Traps & Mistakes

- **Putting an aggregate condition in `WHERE` instead of `HAVING`.** `WHERE SUM(total_amount) >
  1000` fails. Move it to `HAVING`.
- **Forgetting a `SELECT` column in `GROUP BY`.** Postgres will error immediately, which is
  actually helpful — but candidates sometimes panic and don't know why. Check every plain column
  in `SELECT` is also in `GROUP BY`.
- **Using `COUNT(*)` when you meant `COUNT(DISTINCT ...)`.** After a `JOIN` that produces multiple
  rows per parent row (like `orders` joined to `order_items`), `COUNT(*)` counts line items, not
  orders. Use `COUNT(DISTINCT order_id)` to count actual orders.
- **Assuming `AVG()` treats `NULL` as 0.** It does not. `AVG()` only averages non-null values.
  `AVG(salary)` over 3 employees where one has `NULL` salary computes the average of the other 2,
  not the average of 3 with one counted as zero. If you truly want `NULL` treated as `0`, use
  `AVG(COALESCE(salary, 0))`.
- **Confusing "filter rows" with "filter groups."** A candidate might write `WHERE status =
  'delivered' AND COUNT(*) > 5` — mixing row-level and group-level conditions in one clause. Split
  them: row conditions go in `WHERE`, group conditions go in `HAVING`.
- **Grouping by the wrong granularity.** `GROUP BY customer_id` gives one row per customer across
  all time. If the question wants "per customer per month," you must also group by a
  month/year expression, or the result silently merges all months together.
- **Thinking `GROUP BY` and `DISTINCT` are the same.** `DISTINCT` removes duplicate rows in the
  output. `GROUP BY` combines rows into groups so aggregate functions can run per group. They can
  produce similar-looking results for simple cases, but `GROUP BY` is built for aggregation;
  `DISTINCT` is not.
- **Forgetting that `ROLLUP` adds extra rows.** If you don't expect subtotal rows, `ROLLUP` output
  can look like "duplicate" or "wrong" data. Always check for `NULL` markers in grouped columns
  when using `ROLLUP` or `GROUPING SETS`.

*Next: Chapter 4 covers subqueries and CTEs — useful when you need to filter on an aggregate
result from a different table, or break a complex query into named, readable steps.*
