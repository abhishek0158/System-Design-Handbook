# Chapter 5 — Window Functions

Window functions let you compute a value across a set of related rows, without collapsing those rows into one. This is the single most tested SQL topic in interviews, because it shows up in real problems like "rank employees by salary" or "find the running total of sales." If you can write these fluently, you stand out.

## Key Concepts

### What is a window function?

A **window function** computes a value using a group of rows, called a **window**, but it keeps every row in the output. This is the key difference from `GROUP BY`.

- `GROUP BY` takes many rows and returns **fewer rows** — one row per group.
- A window function takes many rows and returns the **same number of rows** — it just adds a computed column.

Example. You want each employee's salary, plus the average salary of their department, on the same row.

```sql
SELECT emp_id, emp_name, dept_id, salary,
       AVG(salary) OVER (PARTITION BY dept_id) AS dept_avg_salary
FROM employees;
```

This returns one row per employee (not one row per department). Each row also shows the average salary of that employee's department. `GROUP BY` cannot do this directly — it would only give you one row per department, and you would lose the individual employee data. You would need a subquery or a self-join instead. A window function does it in one simple pass.

### The syntax: OVER, PARTITION BY, ORDER BY

Every window function uses the `OVER (...)` clause. Inside `OVER`, you can add two optional pieces:

- `PARTITION BY column` — splits rows into groups (partitions). The function resets for each partition. This is like `GROUP BY`, but rows are not collapsed.
- `ORDER BY column` — sorts rows inside each partition. This matters for ranking, running totals, and `LAG`/`LEAD`. Without `ORDER BY`, there is no defined "previous" or "next" row.

```sql
SELECT emp_id, emp_name, dept_id, salary,
       RANK() OVER (PARTITION BY dept_id ORDER BY salary DESC) AS salary_rank
FROM employees;
```

Read this as: "For each department (`PARTITION BY dept_id`), sort employees by salary from high to low (`ORDER BY salary DESC`), and rank them (`RANK()`)."

If you skip `PARTITION BY`, the whole table is treated as one partition. If you skip `ORDER BY`, ranking functions still run but the order is undefined — this is usually a bug, not a feature.

### Ranking functions: ROW_NUMBER, RANK, DENSE_RANK

These three functions all assign a rank number to each row inside a partition, ordered by some column. They only differ in **how they handle ties** (rows with equal values in the `ORDER BY` column).

| Function | On a tie | After a tie |
|---|---|---|
| `ROW_NUMBER()` | Gives each row a different number, even if tied | Continues normally: 1, 2, 3, 4 |
| `RANK()` | Gives tied rows the same number | **Skips** the next number(s): 1, 2, 2, 4 |
| `DENSE_RANK()` | Gives tied rows the same number | **Does not skip**: 1, 2, 2, 3 |

Example with two employees tied on salary:

```sql
SELECT emp_name, dept_id, salary,
       ROW_NUMBER() OVER (PARTITION BY dept_id ORDER BY salary DESC) AS rn,
       RANK()       OVER (PARTITION BY dept_id ORDER BY salary DESC) AS rnk,
       DENSE_RANK() OVER (PARTITION BY dept_id ORDER BY salary DESC) AS drnk
FROM employees
WHERE dept_id = 10;
```

Suppose department 10 has salaries `90000, 80000, 80000, 70000` (sorted). The result:

| emp_name | salary | rn | rnk | drnk |
|---|---|---|---|---|
| A | 90000 | 1 | 1 | 1 |
| B | 80000 | 2 | 2 | 2 |
| C | 80000 | 3 | 2 | 2 |
| D | 70000 | 4 | 4 | 3 |

Notice: `RANK()` jumps from 2 straight to 4 (it "uses up" the two tied slots). `DENSE_RANK()` goes 1, 2, 2, 3 — no gaps. `ROW_NUMBER()` never repeats a number, even on a tie — the choice between B and C getting rank 2 or 3 is arbitrary unless you add more columns to `ORDER BY` to break the tie.

**When to use which:**
- `ROW_NUMBER()` — when you need a unique number per row. Common use: picking exactly one row per group (see "nth row per group" later in this chapter).
- `RANK()` — when ties should share a rank, and you want the ranking to reflect "how many rows are ahead of me" (matches how people rank in a race — if two people tie for 2nd, the next person is 4th).
- `DENSE_RANK()` — when ties should share a rank, but you want no gaps (useful for "top 3 distinct salary levels" type questions).

### LAG and LEAD: previous and next row

`LAG(column, n)` reads a value from `n` rows **before** the current row, inside the same partition and order. `LEAD(column, n)` reads a value from `n` rows **after**. If `n` is not given, it defaults to 1 (the immediately previous or next row).

A classic real use: **month-over-month change** in revenue.

Assume a monthly revenue summary table `monthly_revenue(month, total_revenue)` built from `orders`:

```sql
WITH monthly AS (
  SELECT date_trunc('month', order_date) AS month,
         SUM(total_amount) AS total_revenue
  FROM orders
  GROUP BY date_trunc('month', order_date)
)
SELECT month,
       total_revenue,
       LAG(total_revenue) OVER (ORDER BY month) AS prev_month_revenue,
       total_revenue - LAG(total_revenue) OVER (ORDER BY month) AS change,
       ROUND(
         (total_revenue - LAG(total_revenue) OVER (ORDER BY month))
         / LAG(total_revenue) OVER (ORDER BY month) * 100, 2
       ) AS pct_change
FROM monthly
ORDER BY month;
```

This builds monthly totals first with a CTE (covered in Chapter 4), then uses `LAG` to pull last month's revenue onto the same row as this month's revenue. The first row has `NULL` for `prev_month_revenue`, because there is no month before it. That `NULL` is expected — always mention it in an interview, since it often trips up a division (dividing by `NULL` gives `NULL`, not an error, but dividing by `0` would error).

`LEAD` works the same way but looks forward. A common use is "days until the next order" per customer:

```sql
SELECT customer_id, order_id, order_date,
       LEAD(order_date) OVER (PARTITION BY customer_id ORDER BY order_date) AS next_order_date,
       LEAD(order_date) OVER (PARTITION BY customer_id ORDER BY order_date) - order_date AS days_to_next_order
FROM orders;
```

Here `PARTITION BY customer_id` makes sure `LEAD` only looks at the next order from the **same** customer, not the next row in the whole table.

### Running totals and moving averages

A **running total** (also called cumulative sum) adds up values as you move down the sorted rows. You get it with `SUM() OVER (ORDER BY ...)`.

```sql
WITH daily AS (
  SELECT order_date::date AS day, SUM(total_amount) AS daily_revenue
  FROM orders
  GROUP BY order_date::date
)
SELECT day,
       daily_revenue,
       SUM(daily_revenue) OVER (ORDER BY day) AS running_total
FROM daily
ORDER BY day;
```

Why does `SUM(daily_revenue) OVER (ORDER BY day)` give a running total, and not the grand total on every row? Because when you add `ORDER BY` inside `OVER`, PostgreSQL uses a default **frame**: "from the start of the partition up to the current row." Without `ORDER BY`, the default frame is the whole partition, so you would get the grand total repeated on every row. This default-frame behavior is explained fully in the frame clause section below — it is one of the most important details in this chapter.

A **moving average** (a smoothed average over a fixed number of recent rows, for example the last 7 days) needs an explicit frame:

```sql
SELECT day,
       daily_revenue,
       AVG(daily_revenue) OVER (
         ORDER BY day
         ROWS BETWEEN 6 PRECEDING AND CURRENT ROW
       ) AS moving_avg_7d
FROM daily
ORDER BY day;
```

`ROWS BETWEEN 6 PRECEDING AND CURRENT ROW` means "the current row plus the 6 rows before it" — 7 rows total. This is a **7-day moving average**, useful for smoothing out day-to-day noise in a revenue or traffic chart.

### NTILE: splitting rows into buckets

`NTILE(n)` splits the rows in a partition into `n` roughly equal-sized buckets, numbered `1` to `n`. It is used for percentile-style grouping, for example splitting customers into 4 spending quartiles.

```sql
SELECT customer_id,
       SUM(total_amount) AS total_spent,
       NTILE(4) OVER (ORDER BY SUM(total_amount) DESC) AS spending_quartile
FROM orders
GROUP BY customer_id;
```

Quartile 1 has the top-spending 25% of customers, quartile 4 has the bottom 25%. If the row count does not divide evenly by `n`, Postgres puts the extra rows in the earlier buckets, so bucket sizes can differ by at most 1 row.

### FIRST_VALUE and LAST_VALUE

`FIRST_VALUE(column)` returns the value from the **first row** in the current frame. `LAST_VALUE(column)` returns the value from the **last row** in the current frame. Both need `OVER (...)` with an `ORDER BY` to make "first" and "last" meaningful.

Example: for each department, show every employee alongside the department's top salary.

```sql
SELECT emp_name, dept_id, salary,
       FIRST_VALUE(salary) OVER (
         PARTITION BY dept_id ORDER BY salary DESC
       ) AS top_salary_in_dept
FROM employees;
```

This works correctly, because the default frame (explained next) already covers "from the start up to the current row," and the first row in that ordering is always the highest salary. `FIRST_VALUE` is safe with the default frame. `LAST_VALUE` is not — see the trap below.

### The frame clause: ROWS BETWEEN ... AND ...

The **frame** is the exact set of rows, inside the current partition, that a window function looks at for the current row. You control it with:

```sql
ROWS BETWEEN <start> AND <end>
```

Common frame boundaries:
- `UNBOUNDED PRECEDING` — from the very first row of the partition
- `N PRECEDING` — N rows before the current row
- `CURRENT ROW` — the current row
- `N FOLLOWING` — N rows after the current row
- `UNBOUNDED FOLLOWING` — to the very last row of the partition

**The default frame** (the one Postgres uses automatically when you write `ORDER BY` inside `OVER` but do not write a frame clause) is:

```sql
RANGE BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
```

In plain words: "from the start of the partition up to and including the current row." This is exactly why `SUM() OVER (ORDER BY ...)` gives a running total by default — the frame grows one row at a time as you move down.

If you omit `ORDER BY` entirely, the default frame becomes the **whole partition** (every row), because there is no defined order to stop at.

### The LAST_VALUE trap

This is a very common interview trap. Look at this query:

```sql
SELECT emp_name, dept_id, salary,
       LAST_VALUE(salary) OVER (
         PARTITION BY dept_id ORDER BY salary DESC
       ) AS bottom_salary_in_dept
FROM employees;
```

A candidate expects `bottom_salary_in_dept` to show the lowest salary in the department, on every row. But it does **not**. It shows the current row's own salary, repeated back.

Why? Because the default frame is "up to and including the current row." `LAST_VALUE` returns the value at the **end of the frame** — and the end of the frame is always the current row, not the end of the partition. So `LAST_VALUE` just returns the current row's salary, which is useless.

**The fix:** explicitly widen the frame to cover the whole partition:

```sql
SELECT emp_name, dept_id, salary,
       LAST_VALUE(salary) OVER (
         PARTITION BY dept_id ORDER BY salary DESC
         ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
       ) AS bottom_salary_in_dept
FROM employees;
```

Now the frame is "the entire partition, for every row," so `LAST_VALUE` correctly returns the lowest salary in the department on every row. Always remember: **`FIRST_VALUE` is usually safe with the default frame, but `LAST_VALUE` almost always needs an explicit frame.** Saying this out loud in an interview is a strong signal that you understand window functions deeply, not just by memorizing syntax.

## The Questions They Ask

**Q1: What is the difference between GROUP BY and a window function?**
`GROUP BY` collapses rows: you get one output row per group, and you lose access to individual row detail unless it is inside an aggregate. A window function keeps every row: it computes a value across a group of related rows (the "window") but attaches that value back onto each individual row. Use `GROUP BY` when you want a summary. Use a window function when you want detail rows plus a computed comparison, like "this employee's salary vs. their department's average."
*Follow-up: can you use both together?* Yes. You can `GROUP BY` first, and then in an outer query (or CTE), apply window functions on the grouped results — for example, ranking departments by their average salary.

**Q2: Explain the difference between ROW_NUMBER, RANK, and DENSE_RANK.**
All three number rows within a partition, ordered by a column. They differ only in how ties are handled. `ROW_NUMBER` never repeats a number — ties get arbitrary distinct numbers. `RANK` gives ties the same number, then skips ahead (1, 2, 2, 4). `DENSE_RANK` gives ties the same number, with no gap (1, 2, 2, 3).
*Follow-up: when would DENSE_RANK cause a bug?* If you use `DENSE_RANK() = 3` to mean "3rd highest salary" but there are ties, you might get more or fewer rows than expected, because multiple people can share rank 1 or 2. Use `ROW_NUMBER` instead if you truly need exactly one row.

**Q3: Write a query to find the running total of daily revenue.**
```sql
SELECT order_date::date AS day,
       SUM(total_amount) AS daily_revenue,
       SUM(SUM(total_amount)) OVER (ORDER BY order_date::date) AS running_total
FROM orders
GROUP BY order_date::date
ORDER BY day;
```
Here we combine `GROUP BY` (to get one row per day) with a window function (to accumulate the per-day totals). The inner `SUM(total_amount)` aggregates order amounts into one number per day. The outer `SUM(...) OVER (ORDER BY day)` then adds those daily numbers cumulatively. This nested-aggregate pattern ("aggregate then window") is common and worth practicing.

**Q4: How do you find the nth highest salary in each department?**
Use `ROW_NUMBER`, because it guarantees exactly one row per rank, even with ties.
```sql
SELECT emp_name, dept_id, salary
FROM (
  SELECT emp_name, dept_id, salary,
         ROW_NUMBER() OVER (PARTITION BY dept_id ORDER BY salary DESC) AS rn
  FROM employees
) ranked
WHERE rn = 2;   -- change to any n
```
Window functions cannot go directly in a `WHERE` clause (they run after `WHERE` in query evaluation order), so you must wrap the query in a subquery or a CTE and filter in the outer layer. This is a very common mistake — see Common Traps below.
*Follow-up: what if you want the 2nd highest distinct salary, ignoring duplicates?* Use `DENSE_RANK` instead of `ROW_NUMBER`, so tied salaries count as one rank level.

**Q5: How would you compute month-over-month percentage growth in revenue?**
Use `LAG` to bring last month's value onto the current row, then compute the percentage difference:
```sql
WITH monthly AS (
  SELECT date_trunc('month', order_date) AS month, SUM(total_amount) AS revenue
  FROM orders
  GROUP BY date_trunc('month', order_date)
)
SELECT month, revenue,
       LAG(revenue) OVER (ORDER BY month) AS prev_revenue,
       ROUND((revenue - LAG(revenue) OVER (ORDER BY month))
             / NULLIF(LAG(revenue) OVER (ORDER BY month), 0) * 100, 2) AS pct_growth
FROM monthly
ORDER BY month;
```
Note the `NULLIF(..., 0)` — it protects against a division-by-zero error if a previous month had zero revenue. This detail shows real production thinking, not just textbook syntax.

**Q6: What does LAST_VALUE return, and why do people get it wrong?**
Covered in detail above. Short answer: with the default frame, `LAST_VALUE` returns the current row's own value, because the frame ends at the current row, not at the end of the partition. Fix it with `ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING`.

**Q7: Can you use a window function result in the WHERE clause?**
No, not directly. SQL evaluates clauses in this logical order: `FROM` → `WHERE` → `GROUP BY` → `HAVING` → window functions → `SELECT` → `ORDER BY`. Since window functions run after `WHERE` and `HAVING`, you cannot filter on their output in the same query level. You must compute the window function in a subquery or CTE, then filter in an outer `SELECT`.

**Q8: How do PARTITION BY and GROUP BY relate?**
They both split rows into groups by column values. `GROUP BY` does the split and then reduces each group into one row. `PARTITION BY` does the split, but the function still evaluates per row, keeping the row count unchanged. Conceptually, `PARTITION BY` is "windowed GROUP BY" — it groups without collapsing.

**Q9: How would you find, for each customer, their first order and their most recent order, on the same row as every order?**
This is a good test of `FIRST_VALUE` and the frame trap together:
```sql
SELECT customer_id, order_id, order_date,
       FIRST_VALUE(order_date) OVER (
         PARTITION BY customer_id ORDER BY order_date
       ) AS first_order_date,
       LAST_VALUE(order_date) OVER (
         PARTITION BY customer_id ORDER BY order_date
         ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
       ) AS most_recent_order_date
FROM orders;
```
`FIRST_VALUE` works with the default frame, because the first row in the `ORDER BY order_date` sequence is always the earliest order. `LAST_VALUE` needs the explicit full-partition frame, otherwise it just repeats each row's own `order_date`. This question is popular because it forces the candidate to apply the frame-clause trap from Key Concepts, not just recite it.

**Q10: What is the difference between RANK() and PERCENT_RANK()?**
`RANK()` gives an integer position within the partition (1, 2, 2, 4, ...). `PERCENT_RANK()` converts that position into a relative value between 0 and 1, using the formula `(rank - 1) / (total_rows - 1)`. It answers "what fraction of rows rank below me?" rather than "what is my exact position?" It is less commonly asked than `RANK`/`DENSE_RANK`, but interviewers sometimes mention it to see if you know the ranking family goes beyond the three basic functions.

## Rapid-Fire

- **Does a window function reduce row count?** No. It keeps every row and adds a computed column.
- **What clause makes "previous row" meaningful for LAG?** `ORDER BY` inside `OVER`. Without it, row order is undefined.
- **RANK() vs DENSE_RANK() on 3 tied rows at rank 2 — what is the next rank?** `RANK` gives 5, `DENSE_RANK` gives 3.
- **Default frame when ORDER BY is used inside OVER?** `RANGE BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW`.
- **Which is safe with the default frame: FIRST_VALUE or LAST_VALUE?** `FIRST_VALUE` is safe. `LAST_VALUE` usually needs an explicit frame.
- **How do you get a 7-row moving average?** `AVG(x) OVER (ORDER BY d ROWS BETWEEN 6 PRECEDING AND CURRENT ROW)`.
- **Can window functions appear in WHERE?** No. Use a subquery or CTE and filter in the outer query.
- **What does NTILE(4) do?** Splits partition rows into 4 roughly equal buckets, numbered 1 to 4.
- **LAG(col, 2) means what?** The value of `col` from 2 rows before the current row.
- **Best function for "exactly the 2nd row per group"?** `ROW_NUMBER`, because it never repeats a number.

## Common Traps & Mistakes

**Trap 1: Filtering a window function result directly in WHERE.**
```sql
-- WRONG — fails with "window functions are not allowed in WHERE"
SELECT emp_name, ROW_NUMBER() OVER (PARTITION BY dept_id ORDER BY salary DESC) AS rn
FROM employees
WHERE rn = 1;
```
Fix: wrap it in a subquery or CTE, then filter in the outer `SELECT`.

**Trap 2: Forgetting ORDER BY inside OVER for ranking or LAG/LEAD.**
Without `ORDER BY`, `ROW_NUMBER` and `RANK` still run, but the order they assign is whatever order Postgres happens to read rows in — not guaranteed or repeatable. `LAG`/`LEAD` without `ORDER BY` are close to meaningless, since "previous row" needs a defined sequence.

**Trap 3: Assuming LAST_VALUE gives "the last row of the group."**
As explained above, the default frame stops at the current row. Always pair `LAST_VALUE` with an explicit `ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING` frame, unless you deliberately want the running-frame behavior.

**Trap 4: Using RANK() when you need exactly N rows.**
If you ask for "top 3 employees by salary" using `RANK() <= 3`, and there is a 3-way tie at rank 2, you get more than 3 rows (rank 1, then three rows at rank 2). This may be exactly what the interviewer wants (all employees tied for a spot), or it may be a bug depending on the question's wording. Always clarify: "do you want exactly N rows, or N rank levels?" This is a great example of stating an assumption out loud, which interviewers reward.

**Trap 5: Confusing PARTITION BY with GROUP BY and combining them incorrectly.**
You cannot mix a window function's `PARTITION BY` as a replacement for `GROUP BY` when you also select non-aggregated, non-grouped columns outside a window function — that still triggers a "column must appear in GROUP BY" error. Window functions and `GROUP BY` solve different problems and can be used together, but one is not a drop-in substitute for the other's column rules.

**Trap 6: Forgetting that ROWS and RANGE can behave differently with ties.**
By default, Postgres uses `RANGE`, not `ROWS`, when you write only `ORDER BY` without a frame. With `RANGE`, rows with equal `ORDER BY` values are treated as part of the same frame boundary (they enter or leave the frame together). With `ROWS`, each physical row is counted individually, regardless of ties. For most interview-level answers, this distinction rarely changes the result, but if the interviewer pushes on it, mention that `ROWS BETWEEN ...` counts physical rows, while `RANGE BETWEEN ...` groups by the `ORDER BY` value.

**Trap 7: Nesting window functions or using a window function inside an aggregate directly.**
You cannot write `SUM(RANK() OVER (...))` in the same `SELECT` list expression the way you might hope — actually you can call an aggregate on a window function's output, but you cannot nest one window function inside another window function's `OVER` clause. If you need to do math on a window function's result, compute it in a CTE first, then reference the computed column in an outer query.

Window functions connect closely to Chapter 4 (Subqueries & CTEs), since almost every real window-function query needs a CTE or subquery to pre-aggregate data or to filter on the window's result. They also show up heavily in Chapter 6 (SQL Problem Patterns), which covers classic problems like "top N per group" and "gaps and islands" that lean on the ranking functions from this chapter.
