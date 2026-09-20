# Chapter 6 — SQL Problem Patterns

These are the classic SQL problems that show up again and again in interviews. Each one is a
"pattern" — once you know the shape, you can solve many variations fast. Practice writing these
from memory. All examples use the shared HR schema (`departments`, `employees`) and the
e-commerce schema (`customers`, `products`, `orders`, `order_items`) from the design brief. A few
patterns need one small extra table — each of those says so clearly.

## Pattern 1: Nth Highest Salary

**The problem:** Find the Nth highest salary in the `employees` table. This is the single most
asked SQL question. Interviewers want to see if you know more than one way, and if you know the
duplicate-salary trap.

**Way 1 — Correlated subquery.** For each row, count how many *distinct* salaries are strictly
greater than it. If that count is `N - 1`, this row holds the Nth highest salary.

```sql
-- 2nd highest salary
SELECT salary
FROM employees e1
WHERE 1 = (
  SELECT COUNT(DISTINCT e2.salary)
  FROM employees e2
  WHERE e2.salary > e1.salary
);
```

**Way 2 — DENSE_RANK().** Rank all salaries from highest to lowest, then pick rank N.

```sql
SELECT salary
FROM (
  SELECT salary, DENSE_RANK() OVER (ORDER BY salary DESC) AS rnk
  FROM employees
) ranked
WHERE rnk = 2;   -- change 2 to N
```

**Way 3 — LIMIT / OFFSET.** Sort distinct salaries and skip the first `N - 1`.

```sql
SELECT DISTINCT salary
FROM employees
ORDER BY salary DESC
LIMIT 1 OFFSET 1;   -- OFFSET N-1 for the Nth highest
```

**How it works:** All three ways answer "what is the Nth distinct salary value, counting from the
top?" The correlated subquery counts greater values. `DENSE_RANK` labels each distinct value with
a rank, with no gaps. `LIMIT/OFFSET` just sorts and skips rows.

**The duplicate-salary subtlety:** Say three employees earn 90000, and the next distinct salary is
85000. If you use `RANK()` instead of `DENSE_RANK()`, the three tied rows all get rank 1, but the
       next row gets rank 4 (rank skips ahead by the number of ties). So "2nd highest" using `RANK()`
       would return nothing until you ask for rank 4. `DENSE_RANK()` gives the tied rows rank 1 and the
       next distinct value rank 2 — this matches what most interviewers mean by "2nd highest salary."
       Also, `LIMIT/OFFSET` without `DISTINCT` on the sorted column will return the same salary value
       multiple times if there are ties, which is usually wrong. Always ask the interviewer: "Do you want
       the 2nd highest distinct value, or just the row at position 2 when sorted?" — these can differ.

**When it is asked:** Almost every SQL round, often as the opening warm-up question. Follow-ups:
"Now write it for Nth highest per department" (combine with Pattern 3), or "What if salary can be
NULL?" (add `WHERE salary IS NOT NULL`, since `NULL` never compares as greater or smaller than a
number, and it would confuse `COUNT(DISTINCT ...)` and the ranking order).

**Performance note:** the correlated subquery (Way 1) runs the inner query once per outer row, so
it can be slow on a large table without an index on `salary`. `DENSE_RANK()` (Way 2) scans the
table once and is usually the faster choice for a big `employees` table — mention this trade-off
if the interviewer asks "which one would you use in production?"

## Bonus: Nth Highest Salary Per Department

A very common follow-up combines Pattern 1 and Pattern 3: "find the 2nd highest salary in each
department."

```sql
SELECT dept_id, salary AS second_highest_salary
FROM (
  SELECT dept_id, salary,
         DENSE_RANK() OVER (PARTITION BY dept_id ORDER BY salary DESC) AS rnk
  FROM employees
) ranked
WHERE rnk = 2;
```

This is exactly Way 2 from Pattern 1, with `PARTITION BY dept_id` added so the ranking restarts
for every department. This combination — "take a single-table pattern and add `PARTITION BY`" —
is the single highest-leverage trick in this whole chapter. It turns almost any of the 10 patterns
below into a "per group" version.

## Pattern 2: Find and Remove Duplicate Rows

**The problem:** The `employees` table has duplicate rows — same `emp_name`, `dept_id`, `salary`,
and `hire_date`, but different `emp_id` (because `emp_id` is an auto-generated key, two inserts of
the same data still get different keys). Find the duplicates, then remove all but one copy of
each.

**Find duplicates:**

```sql
SELECT emp_name, dept_id, salary, hire_date, COUNT(*) AS copies
FROM employees
GROUP BY emp_name, dept_id, salary, hire_date
HAVING COUNT(*) > 1;
```

**Remove duplicates, keeping the row with the smallest `emp_id`:**

```sql
DELETE FROM employees a
USING employees b
WHERE a.emp_id > b.emp_id
  AND a.emp_name = b.emp_name
  AND a.dept_id = b.dept_id
  AND a.salary = b.salary
  AND a.hire_date = b.hire_date;
```

**A cleaner way with `ROW_NUMBER()`:**

```sql
WITH ranked AS (
  SELECT emp_id,
         ROW_NUMBER() OVER (
           PARTITION BY emp_name, dept_id, salary, hire_date
           ORDER BY emp_id
         ) AS rn
  FROM employees
)
DELETE FROM employees
WHERE emp_id IN (SELECT emp_id FROM ranked WHERE rn > 1);
```

**How it works:** `GROUP BY` on the columns that define a "duplicate" plus `HAVING COUNT(*) > 1`
finds duplicate groups. For deletion, `ROW_NUMBER()` numbers each row within its duplicate group,
starting at 1 for the row you want to keep (the smallest `emp_id`, chosen by the `ORDER BY` inside
`PARTITION BY`). Any row with `rn > 1` is an extra copy, safe to delete. The `USING` self-join in
the first delete does the same thing without a CTE, useful in older Postgres versions.

**When it is asked:** Common as a two-part question: first "find duplicates" (a `GROUP BY /
HAVING` check), then "now delete them" (harder — many candidates delete *all* copies by mistake
with a naive `DELETE ... WHERE COUNT(*) > 1`, which is not valid SQL, or they delete every row in
the group). Say out loud which copy you plan to keep — this is a real design decision, not just
syntax.

**Postgres-only shortcut with `ctid`:** every Postgres row has a hidden physical address column
called `ctid`. If the table has no good "which copy to keep" rule, some candidates use it:

```sql
DELETE FROM employees
WHERE ctid NOT IN (
  SELECT MIN(ctid)
  FROM employees
  GROUP BY emp_name, dept_id, salary, hire_date
);
```

Mention `ctid` only as a "Postgres trick I know," not as your main answer — it does not exist in
other databases, and it is tied to physical storage, not to a stable business key. The
`ROW_NUMBER()` version is the answer most interviewers want to see, because it works the same way
in any database that supports window functions.

## Pattern 3: Top-N Per Group

**The problem:** Find the highest-paid employee in each department. More generally: "top N rows
per group" (e.g., top 3 highest-paid per department).

```sql
SELECT emp_id, emp_name, dept_id, salary
FROM (
  SELECT e.*,
         ROW_NUMBER() OVER (PARTITION BY dept_id ORDER BY salary DESC) AS rn
  FROM employees e
) ranked
WHERE rn = 1;         -- rn <= 3 for top 3 per department
```

**How it works:** `PARTITION BY dept_id` restarts the ranking for each department. `ORDER BY
salary DESC` inside the window ranks the highest earner first. `ROW_NUMBER()` gives each row in a
department a unique number 1, 2, 3, ... Filtering `rn = 1` keeps only the top earner per
department.

**RANK() vs ROW_NUMBER() here:** if two employees in the same department tie for the highest
salary, `ROW_NUMBER()` arbitrarily picks one as rank 1 and the other as rank 2 — you get only one
row. If you want **both** tied employees returned, use `RANK()` instead, and filter `rank = 1`.
Always clarify this with the interviewer before coding.

**When it is asked:** Extremely common, often phrased as "top-selling product per category" or
"most recent order per customer." It is the single most useful window-function pattern to have
ready — see also Chapter 5 (Window Functions).

**Without window functions (older-style answer):** some interviewers ask you to solve it with a
correlated subquery instead, to check you understand the idea without relying on syntax:

```sql
SELECT e.*
FROM employees e
WHERE e.salary = (
  SELECT MAX(e2.salary)
  FROM employees e2
  WHERE e2.dept_id = e.dept_id
);
```

This reads as "keep this employee if their salary equals the max salary in their own department."
It is shorter to write but has the same tie problem as `ROW_NUMBER()` in reverse: if two employees
tie for the top salary in a department, this version returns **both** — which is actually closer
to `RANK()` behavior than to `ROW_NUMBER()`. Point this out if asked.

## Pattern 4: Running Total

**The problem:** For each customer, show a running total of order amounts, ordered by order date.

```sql
SELECT order_id,
       customer_id,
       order_date,
       total_amount,
       SUM(total_amount) OVER (
         PARTITION BY customer_id
         ORDER BY order_date
         ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
       ) AS running_total
FROM orders
ORDER BY customer_id, order_date;
```

**How it works:** The window frame `ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW` tells
Postgres to sum every row from the start of the partition up to the current row. `PARTITION BY
customer_id` means the running total resets for each customer. Note: this frame is actually the
default when you write `ORDER BY` inside a window without stating a frame — so `SUM(total_amount)
OVER (PARTITION BY customer_id ORDER BY order_date)` gives the same result. Writing the frame out
explicitly is good practice in an interview because it shows you understand what is happening.

**When it is asked:** Common in e-commerce/finance-flavored questions: running balance, running
revenue, cumulative count of signups per day. Follow-up: "Now show a 7-day moving average instead"
— swap the frame to `AVG(total_amount) OVER (... ROWS BETWEEN 6 PRECEDING AND CURRENT ROW)`.
Another common follow-up: "reset the running total every year" — add `EXTRACT(YEAR FROM
order_date)` to the `PARTITION BY` list, alongside `customer_id`, so the sum starts over each
January.

## Pattern 5: Gaps and Islands (Consecutive Days)

**The problem:** Find streaks of consecutive login days for each customer. This needs one small
extra table not in the shared schema: `logins(customer_id, login_date)`, one row per day a
customer logged in.

```sql
WITH numbered AS (
  SELECT customer_id,
         login_date,
         ROW_NUMBER() OVER (PARTITION BY customer_id ORDER BY login_date) AS rn
  FROM logins
),
grouped AS (
  SELECT customer_id,
         login_date,
         login_date - (rn * INTERVAL '1 day') AS island_id
  FROM numbered
)
SELECT customer_id,
       MIN(login_date) AS streak_start,
       MAX(login_date) AS streak_end,
       COUNT(*) AS streak_length
FROM grouped
GROUP BY customer_id, island_id
ORDER BY customer_id, streak_start;
```

**How it works:** This is the classic "gaps and islands" trick. If login dates are consecutive,
then `login_date - rn` (the date minus a running counter) stays **constant** across the whole
streak, because both the date and the row number increase by 1 each day. When there is a gap, `rn`
keeps increasing but the date jumps, so `login_date - rn` changes to a new value. That new value
marks a new "island." Grouping by `(customer_id, island_id)` then collects each streak into one
row, and `MIN`/`MAX`/`COUNT` describe it.

**When it is asked:** A favorite "hard" question — attendance streaks, subscription-active
streaks, session gaps. It looks intimidating the first time but is completely mechanical once you
know the `date - row_number` trick. Say the idea out loud before coding: "same offset means same
streak" — interviewers want to hear that reasoning, not just see the SQL appear.

**Common follow-up — "longest streak per customer":** wrap the query above in one more layer and
pick the row with the largest `streak_length` per customer, using Pattern 3 (top-N per group):

```sql
SELECT customer_id, streak_start, streak_end, streak_length
FROM (
  SELECT customer_id, streak_start, streak_end, streak_length,
         ROW_NUMBER() OVER (PARTITION BY customer_id ORDER BY streak_length DESC) AS rn
  FROM (
    -- the full gaps-and-islands query from above, as a subquery
    SELECT customer_id, MIN(login_date) AS streak_start, MAX(login_date) AS streak_end,
           COUNT(*) AS streak_length
    FROM (
      SELECT customer_id, login_date,
             login_date - (ROW_NUMBER() OVER (PARTITION BY customer_id ORDER BY login_date)
                            * INTERVAL '1 day') AS island_id
      FROM logins
    ) grouped
    GROUP BY customer_id, island_id
  ) streaks
) ranked
WHERE rn = 1;
```

This shows the interviewer that gaps-and-islands is not a one-off trick — it composes cleanly with
the other patterns in this chapter.

**Gaps, not just islands:** the same numbering also finds the *gaps* — the missing days. If you
need "days a customer did not log in, between their first and last login," generate a full date
series with `generate_series(min_date, max_date, interval '1 day')` and use a `LEFT JOIN` against
`logins` to find the dates with no match. This is a common Postgres-specific extension of the
pattern, worth mentioning even if not asked directly.

## Pattern 6: Pivot Rows to Columns

**The problem:** Show total sales per month, as columns (Jan, Feb, Mar, ...) instead of one row
per month.

```sql
SELECT
  EXTRACT(YEAR FROM order_date)::int AS order_year,
  SUM(total_amount) FILTER (WHERE EXTRACT(MONTH FROM order_date) = 1)  AS jan,
  SUM(total_amount) FILTER (WHERE EXTRACT(MONTH FROM order_date) = 2)  AS feb,
  SUM(total_amount) FILTER (WHERE EXTRACT(MONTH FROM order_date) = 3)  AS mar,
  SUM(total_amount) FILTER (WHERE EXTRACT(MONTH FROM order_date) = 12) AS dec
FROM orders
GROUP BY EXTRACT(YEAR FROM order_date);
```

**How it works:** `FILTER (WHERE ...)` is Postgres's clean way to write a conditional aggregate —
it sums `total_amount` only for rows that pass the filter. This is the same idea as a `CASE WHEN`
inside `SUM`, and works in any SQL dialect:

```sql
SUM(CASE WHEN EXTRACT(MONTH FROM order_date) = 1 THEN total_amount ELSE 0 END) AS jan
```

Use `CASE WHEN` if the interviewer asks for portable SQL (`FILTER` is Postgres-only); mention that
trade-off out loud.

**MySQL note:** MySQL has no `FILTER` clause — always use `SUM(CASE WHEN ... THEN ... ELSE 0 END)`
there. MySQL 8 also does not have a generic pivot operator; some databases (SQL Server) have
`PIVOT`, but the `CASE WHEN` approach works everywhere and is what interviewers expect you to
write from memory.

**When it is asked:** Reporting-style questions — "sales by month as columns," "count of orders by
status as columns." Interviewers usually want to see that you know the trick, not that you memorize
12 months. Write 2-3 columns to prove the pattern, then say "repeat for the rest."

**The other direction — columns to rows (unpivot):** interviewers sometimes flip the question:
"the data comes in with one column per month, turn it into one row per month." In Postgres this is
done with `UNION ALL`:

```sql
SELECT order_year, 'jan' AS month, jan AS total FROM monthly_report
UNION ALL
SELECT order_year, 'feb' AS month, feb AS total FROM monthly_report;
-- repeat for each month column
```

There is no built-in `UNPIVOT` in Postgres (SQL Server has one) — `UNION ALL` is the standard,
portable way to do it, and it is fine to say so if asked.

## Pattern 7: N Rows in a Row (Consecutive Condition)

**The problem:** Find days where sales strictly increased for 3 days in a row. Uses a daily
summary of `orders`: `SELECT order_date, SUM(total_amount) AS amount FROM orders GROUP BY
order_date`.

```sql
WITH daily AS (
  SELECT order_date, SUM(total_amount) AS amount
  FROM orders
  GROUP BY order_date
),
with_lags AS (
  SELECT order_date,
         amount,
         LAG(amount, 1) OVER (ORDER BY order_date) AS prev_1,
         LAG(amount, 2) OVER (ORDER BY order_date) AS prev_2
  FROM daily
)
SELECT order_date, amount
FROM with_lags
WHERE amount > prev_1 AND prev_1 > prev_2;
```

**How it works:** `LAG(amount, 1)` gets yesterday's amount, `LAG(amount, 2)` gets the amount two
days before. A row passes the filter only if today beats yesterday, and yesterday beat the day
before — that is exactly "3 increasing days in a row," with the returned row being the last day of
each such streak. To find N in a row, add `N - 1` `LAG` columns and chain the comparisons. For
large N, this gets clunky — that is a sign to switch to the gaps-and-islands pattern instead:
flag each row as `1` (condition true) or `0` (condition false) with a `CASE`, then group
consecutive `1`s using the same `date - row_number` trick from Pattern 5, and keep groups where
`COUNT(*) >= N`.

**When it is asked:** A well-known "hard" question (it maps to a famous "3 consecutive numbers"
style problem). Interviewers use it to see if you reach for `LAG`/`LEAD` naturally, and if you know
when to switch to gaps-and-islands for a general N.

**The general-N version, worked out:** flag each day, then reuse the exact gaps-and-islands
grouping from Pattern 5, but only on the days flagged `1`:

```sql
WITH daily AS (
  SELECT order_date, SUM(total_amount) AS amount
  FROM orders GROUP BY order_date
),
flagged AS (
  SELECT order_date, amount,
         CASE WHEN amount > LAG(amount) OVER (ORDER BY order_date)
              THEN 1 ELSE 0 END AS is_increase
  FROM daily
),
increase_days AS (
  SELECT order_date,
         order_date - (ROW_NUMBER() OVER (ORDER BY order_date) * INTERVAL '1 day') AS island_id
  FROM flagged
  WHERE is_increase = 1
)
SELECT MIN(order_date) AS streak_start, MAX(order_date) AS streak_end, COUNT(*) + 1 AS streak_length
FROM increase_days
GROUP BY island_id
HAVING COUNT(*) + 1 >= 3;   -- change 3 to N
```

The `+ 1` in `streak_length` and in the `HAVING` clause accounts for the first day of each streak,
which is not itself flagged as an "increase" (it has nothing before it to compare against, but it
is still part of the streak). This detail trips up most candidates — walk through a small example
out loud (e.g., amounts 10, 20, 30, 40) to check the off-by-one before you trust the query.

## Pattern 8: Median

**The problem:** Find the median salary — the middle value when salaries are sorted. Unlike
`AVG`, the median is not skewed by outliers.

**Easiest way — `percentile_cont`:**

```sql
SELECT percentile_cont(0.5) WITHIN GROUP (ORDER BY salary) AS median_salary
FROM employees;

-- per department
SELECT dept_id,
       percentile_cont(0.5) WITHIN GROUP (ORDER BY salary) AS median_salary
FROM employees
GROUP BY dept_id;
```

**Window-function way (if `percentile_cont` is not allowed, or to show you understand the math):**

```sql
WITH ranked AS (
  SELECT salary,
         ROW_NUMBER() OVER (ORDER BY salary) AS rn,
         COUNT(*) OVER () AS total_count
  FROM employees
)
SELECT AVG(salary) AS median_salary
FROM ranked
WHERE rn IN ((total_count + 1) / 2, (total_count + 2) / 2);
```

**How it works:** `percentile_cont(0.5)` is Postgres's built-in "continuous percentile" function —
0.5 means the 50th percentile, which is the median. It is an ordered-set aggregate, so it needs
`WITHIN GROUP (ORDER BY ...)` instead of a normal `ORDER BY`. The manual version sorts all
salaries with `ROW_NUMBER()`, counts the total rows with `COUNT(*) OVER ()`, then picks the middle
row. For an odd count, both expressions `(total_count+1)/2` and `(total_count+2)/2` (integer
division) point to the same middle row, so `AVG` of one value is just that value. For an even
count, they point to the two middle rows, and `AVG` averages them — which is the standard
definition of median for an even-sized set.

**When it is asked:** Common at companies that care about statistics-flavored SQL (analytics,
fintech). Interviewers often ask "how would you do this without `percentile_cont`?" as a
follow-up — have the window-function version ready.

**Median vs. average vs. mode — know the difference cold:** `AVG` is pulled toward outliers (one
employee earning 1,000,000 skews the average salary up for everyone). The median is not — it only
cares about the middle position, so it is the better summary when data is skewed (salaries,
house prices, latency numbers are classic skewed examples). The mode (the most frequent value) is
a third option, useful for categorical data, and has no built-in Postgres aggregate — you would
compute it with `GROUP BY value ORDER BY COUNT(*) DESC LIMIT 1`. If an interviewer asks "why not
just use AVG," this is the answer they want.

**Other useful percentiles:** `percentile_cont(0.9) WITHIN GROUP (ORDER BY latency_ms)` gives the
p90 — a very common ask in performance/SRE-flavored interviews ("what is our p99 response time?").
The same syntax works for any percentile between 0 and 1.

## Pattern 9: Month-over-Month / Year-over-Year Change

**The problem:** Show monthly revenue, and the percentage change from the previous month.

```sql
WITH monthly AS (
  SELECT date_trunc('month', order_date) AS month,
         SUM(total_amount) AS revenue
  FROM orders
  GROUP BY date_trunc('month', order_date)
)
SELECT month,
       revenue,
       LAG(revenue) OVER (ORDER BY month) AS prev_month_revenue,
       ROUND(
         (revenue - LAG(revenue) OVER (ORDER BY month))
         / LAG(revenue) OVER (ORDER BY month) * 100,
       2) AS mom_pct_change
FROM monthly
ORDER BY month;
```

**How it works:** `date_trunc('month', order_date)` rounds every date down to the first day of its
month, so grouping by it gives one row per month. `LAG(revenue)` fetches the previous row's
revenue once the rows are ordered by month — that is last month's revenue, sitting right next to
this month's. The percent-change formula is `(new - old) / old * 100`.

**Year-over-year** is the same idea, but you compare the same month last year. If your data is
already one row per month, use `LAG(revenue, 12)` to jump back 12 rows. If you want to compare
full years, group by `date_trunc('year', order_date)` instead and use `LAG(revenue, 1)`.

**Watch division by zero:** if `prev_month_revenue` is 0 or `NULL` (e.g., the very first month,
or a month with no orders), the division fails or returns `NULL`. Guard it with `NULLIF`:
`(revenue - prev) / NULLIF(prev, 0) * 100`.

**Full year-over-year worked example**, comparing each month to the same month one year earlier:

```sql
WITH monthly AS (
  SELECT date_trunc('month', order_date) AS month,
         SUM(total_amount) AS revenue
  FROM orders
  GROUP BY date_trunc('month', order_date)
)
SELECT month,
       revenue,
       LAG(revenue, 12) OVER (ORDER BY month) AS revenue_same_month_last_year,
       ROUND(
         (revenue - LAG(revenue, 12) OVER (ORDER BY month))
         / NULLIF(LAG(revenue, 12) OVER (ORDER BY month), 0) * 100,
       2) AS yoy_pct_change
FROM monthly
ORDER BY month;
```

The only change from month-over-month is `LAG(revenue, 12)` instead of `LAG(revenue)` — 12 rows
back in a table with one row per month is exactly "the same month, last year." This only works if
the monthly data has no missing months; if a month can be missing (no orders at all that month),
generate a complete month series first with `generate_series` and `LEFT JOIN` it against the
aggregated orders, so `LAG(..., 12)` always counts real calendar months, not just "12 rows back."

**When it is asked:** Very common in analytics/product-team-facing roles — "show growth rate,"
"show trend." It is really Pattern 4 (window functions) applied with `LAG` instead of `SUM`.

## Pattern 10: Bought Product X But Not Product Y

**The problem:** Find customers who bought product X, but never bought product Y. Classic
"set difference" question.

**Way 1 — EXISTS / NOT EXISTS:**

```sql
SELECT c.customer_id, c.name
FROM customers c
WHERE EXISTS (
  SELECT 1
  FROM orders o
  JOIN order_items oi ON oi.order_id = o.order_id
  WHERE o.customer_id = c.customer_id AND oi.product_id = 101   -- product X
)
AND NOT EXISTS (
  SELECT 1
  FROM orders o
  JOIN order_items oi ON oi.order_id = o.order_id
  WHERE o.customer_id = c.customer_id AND oi.product_id = 102   -- product Y
);
```

**Way 2 — EXCEPT:**

```sql
SELECT o.customer_id
FROM orders o
JOIN order_items oi ON oi.order_id = o.order_id
WHERE oi.product_id = 101
EXCEPT
SELECT o.customer_id
FROM orders o
JOIN order_items oi ON oi.order_id = o.order_id
WHERE oi.product_id = 102;
```

**How it works:** `EXISTS` checks "is there at least one matching row?" without caring how many —
this is normally faster than a `JOIN` + `COUNT` for this kind of question, because the database can
stop scanning as soon as it finds one match. `NOT EXISTS` is the correct way to express "never
bought" — do **not** use `NOT IN` with a subquery that might return `NULL` values, because `NOT
IN` returns no rows at all if the subquery result contains even one `NULL` (a classic trap, see
below). `EXCEPT` is simpler to read: it is set subtraction — "customers who bought X" minus
"customers who bought Y."

**Way 3 — GROUP BY / HAVING with conditional aggregation:** a third style some interviewers like,
because it needs only one pass over `order_items`:

```sql
SELECT o.customer_id
FROM orders o
JOIN order_items oi ON oi.order_id = o.order_id
WHERE oi.product_id IN (101, 102)
GROUP BY o.customer_id
HAVING SUM(CASE WHEN oi.product_id = 101 THEN 1 ELSE 0 END) > 0
   AND SUM(CASE WHEN oi.product_id = 102 THEN 1 ELSE 0 END) = 0;
```

This filters down to only the two relevant products first (cheap), then for each customer checks
"at least one row for X" and "zero rows for Y" using conditional sums inside `HAVING`.

**When it is asked:** Very common — "users who did A but not B" appears in growth/retention
questions constantly (e.g., "users who signed up but never placed an order"). Interviewers listen
for whether you reach for `NOT EXISTS` (safe) or `NOT IN` (has a `NULL` trap) instinctively. A
frequent follow-up: "now do it for customers who bought **both** X and Y" — that only needs the
`EXISTS` half of Way 1, written twice with `AND` instead of `AND NOT EXISTS`, or `INTERSECT`
instead of `EXCEPT` in Way 2.

## Rapid-Fire

- **Q: DENSE_RANK vs RANK for "Nth highest"?** DENSE_RANK — it does not skip ranks after ties, so
  it matches "Nth distinct value" as most interviewers mean it.
- **Q: Fastest way to get the 2nd highest salary for a quick answer?** `ORDER BY salary DESC
  LIMIT 1 OFFSET 1` with `DISTINCT` — but say the duplicate caveat out loud.
- **Q: How do you find duplicate rows?** `GROUP BY` the columns that define "duplicate," then
  `HAVING COUNT(*) > 1`.
- **Q: How do you delete duplicates safely?** `ROW_NUMBER()` partitioned by the duplicate columns,
  delete where `rn > 1`. Never delete the whole group.
- **Q: Top-N per group tool?** `ROW_NUMBER()` (or `RANK()` if ties should all appear) inside a
  `PARTITION BY`.
- **Q: Running total window frame?** `ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW` (also the
  default frame when `ORDER BY` is present).
- **Q: Core trick behind gaps-and-islands?** `date - ROW_NUMBER()` is constant within one
  unbroken streak, and changes at every gap.
- **Q: Portable way to pivot rows to columns?** `SUM(CASE WHEN condition THEN value ELSE 0 END)`.
  `FILTER (WHERE ...)` is the shorter Postgres-only version.
- **Q: Function for median in Postgres?** `percentile_cont(0.5) WITHIN GROUP (ORDER BY col)`.
- **Q: Why NOT IN is risky for "not exists" questions?** If the subquery returns any `NULL`,
  `NOT IN` silently returns zero rows. Use `NOT EXISTS` instead.
- **Q: How do you find N consecutive rows matching a condition?** Chain `LAG(col, 1)`,
  `LAG(col, 2)`, ... up to N-1 for small fixed N; use gaps-and-islands for general N.

## Common Traps & Mistakes

- **Forgetting DISTINCT with LIMIT/OFFSET.** If two people share the top salary, `ORDER BY salary
  DESC LIMIT 1 OFFSET 1` without `DISTINCT` returns the *same* top salary again as "2nd highest,"
  which is usually not what is wanted.
- **Using RANK() when DENSE_RANK() is meant, or vice versa.** Always ask: "should ties share a
  rank and skip the next one (RANK), or share a rank with no skip (DENSE_RANK)?" State your
  assumption before coding.
- **Deleting the whole duplicate group instead of keeping one copy.** A `DELETE ... WHERE
  emp_id IN (SELECT emp_id FROM employees GROUP BY ... HAVING COUNT(*) > 1)` deletes **every**
  copy, leaving none. Always number rows first and keep exactly one.
- **NOT IN with a nullable column.** `WHERE product_id NOT IN (SELECT product_id FROM ... )` fails
  silently (returns 0 rows) if the subquery can return `NULL`. Use `NOT EXISTS`.
- **Off-by-one in window frames.** Confusing `ROWS BETWEEN 1 PRECEDING AND CURRENT ROW` (2-row
  window: today and yesterday) with `ROWS BETWEEN 2 PRECEDING AND CURRENT ROW` (3-row window).
  Always count how many rows the frame actually spans.
- **Forgetting PARTITION BY, so a running total or rank leaks across groups.** Without
  `PARTITION BY customer_id`, a "running total per customer" becomes one running total for the
  whole table.
- **Dividing by a NULL or zero previous value in MoM/YoY.** Wrap the denominator in `NULLIF(...,
  0)`, and remember the first row in any `LAG` series is always `NULL` — decide how to show it
  (usually as `NULL` or "N/A," not 0).
- **Assuming gaps-and-islands only works on integers.** The same `value - ROW_NUMBER()` trick
  works on dates (using `INTERVAL '1 day'`) and even on any sequential numeric ID — the concept is
  general, not just for dates.
- **Not stating assumptions about what counts as a "duplicate."** For Pattern 2, some
  interviewers mean "same primary key inserted twice" (should not happen) and others mean "same
  business data, different key" (the common real case). Ask which one before writing `GROUP BY`.
