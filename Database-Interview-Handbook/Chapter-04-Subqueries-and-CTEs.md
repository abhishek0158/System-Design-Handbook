# Chapter 4 — Subqueries & CTEs

A subquery is a query inside another query. Interviewers use them to check if you can break a
hard problem into small steps. CTEs (`WITH` blocks) do the same job, but keep your SQL readable.

## Key Concepts

### What is a subquery?

A **subquery** is a `SELECT` statement written inside another SQL statement. The inner query
runs first (in most cases), and the outer query uses its result. You can put a subquery in the
`SELECT` list, in the `FROM` clause, or in the `WHERE` clause.

```sql
-- Subquery in WHERE: employees who earn more than the company average
SELECT emp_name, salary
FROM employees
WHERE salary > (SELECT AVG(salary) FROM employees);
```

Here `(SELECT AVG(salary) FROM employees)` is the subquery. It returns one number. The outer
query then compares each employee's salary to that number.

### Scalar, row, and table subqueries

Subqueries return different shapes of data. The shape decides where you can use them.

- **Scalar subquery**: returns exactly one row and one column (a single value). You can use it
  anywhere a normal value is allowed, like `SELECT`, `WHERE`, or even inside a function.
- **Row subquery**: returns one row but more than one column. You compare it to a row value
  using `(col1, col2) = (SELECT c1, c2 FROM ...)`.
- **Table subquery**: returns many rows and many columns. You use it in `FROM` (as a derived
  table) or with operators like `IN`, `EXISTS`, `ANY`, `ALL`.

```sql
-- Scalar subquery in SELECT list: show each employee's salary gap from the average
SELECT emp_name,
       salary,
       salary - (SELECT AVG(salary) FROM employees) AS gap_from_avg
FROM employees;
```

```sql
-- Row subquery: find the employee with the exact same (dept_id, salary) pair as emp_id = 5
SELECT emp_name
FROM employees
WHERE (dept_id, salary) = (SELECT dept_id, salary FROM employees WHERE emp_id = 5)
  AND emp_id <> 5;
```

```sql
-- Table subquery in FROM: treat a summary as a temporary table
SELECT dept_id, avg_salary
FROM (
    SELECT dept_id, AVG(salary) AS avg_salary
    FROM employees
    GROUP BY dept_id
) AS dept_avg
WHERE avg_salary > 60000;
```

A subquery used in `FROM` is often called a **derived table**. It must have an alias
(`AS dept_avg` above). Postgres will reject it without one.

### Subqueries in SELECT, FROM, and WHERE

You have already seen all three. Here is the short summary:

| Location | Purpose | Must return |
|---|---|---|
| `SELECT` list | Add a computed value per row | Scalar (one value per outer row) |
| `FROM` clause | Build a temporary table to query further | Table (rows and columns) |
| `WHERE` clause | Filter rows using a computed condition | Scalar, row, or table, depending on the operator |

A common interview task: "Show each customer's order count next to their name."

```sql
SELECT c.customer_id,
       c.name,
       (SELECT COUNT(*) FROM orders o WHERE o.customer_id = c.customer_id) AS order_count
FROM customers c;
```

This subquery is scalar (`COUNT(*)` always returns one number), and it runs once per row of
`customers`. This brings us to correlated subqueries.

### Correlated vs non-correlated subqueries

A **non-correlated subquery** does not use anything from the outer query. It can run once, on
its own, and the result is reused for every row of the outer query.

```sql
-- Non-correlated: the inner query does not depend on the outer table
SELECT emp_name, salary
FROM employees
WHERE salary > (SELECT AVG(salary) FROM employees);
```

A **correlated subquery** uses a column from the outer query inside the inner query. Because
of this link, the database must, in theory, run the inner query once for every row the outer
query looks at.

```sql
-- Correlated: the inner query uses e.dept_id, which comes from the outer row
SELECT e.emp_name, e.salary, e.dept_id
FROM employees e
WHERE e.salary > (
    SELECT AVG(e2.salary)
    FROM employees e2
    WHERE e2.dept_id = e.dept_id
);
```

This query finds employees who earn more than the average salary in their own department. The
inner query cannot run by itself — it needs `e.dept_id` from the current outer row.

**Performance note:** a correlated subquery can run once per outer row. On a large table, this
can be slow if there is no good index to support the inner query's filter. Postgres often
rewrites correlated subqueries into joins internally, but do not assume this always happens.
Say this in an interview: "This is correlated, so it may run per row. I would check
`EXPLAIN ANALYZE`, and consider a join rewrite if it is slow." (See Chapter 7 for `EXPLAIN`.)

### IN vs EXISTS vs JOIN

These three tools often solve the same problem: "find rows in table A that have a match in
table B." Interviewers like to ask which one you would pick, and why.

**IN** checks if a value is inside a list, or inside the result of a subquery.

```sql
-- Customers who have placed at least one order
SELECT customer_id, name
FROM customers
WHERE customer_id IN (SELECT customer_id FROM orders);
```

**EXISTS** checks if a subquery returns any row at all. It does not care about the actual
values, only whether at least one row exists. `EXISTS` subqueries are usually correlated.

```sql
-- Same result, using EXISTS
SELECT customer_id, name
FROM customers c
WHERE EXISTS (
    SELECT 1 FROM orders o WHERE o.customer_id = c.customer_id
);
```

Note the `SELECT 1` inside `EXISTS`. It does not matter what column you select, because
`EXISTS` only checks for row presence, not values. Some engineers write `SELECT *` instead —
both work the same way.

**JOIN** combines rows from two tables directly. If you need columns from both tables in your
output, a join is often the natural choice.

```sql
-- Distinct customers who have orders, using JOIN
SELECT DISTINCT c.customer_id, c.name
FROM customers c
JOIN orders o ON o.customer_id = c.customer_id;
```

**When to use which:**

- Use **JOIN** when you need columns from both tables in the result.
- Use **EXISTS** when you only need to check "does a match exist?", with no column from the
  inner table. It can stop as soon as it finds one matching row, which is often efficient.
- Use **IN** for membership against a short, static list, or a clean subquery result with no
  `NULL`. Avoid `NOT IN` on nullable columns (see the trap below).

For "not matching" problems, prefer **NOT EXISTS** over **NOT IN**. This is one of the most
common interview traps, explained next.

### The NULL trap with NOT IN

This is a classic interview gotcha. `NOT IN` behaves in a surprising way when the subquery's
result list contains a `NULL`.

Suppose `orders.customer_id` can be `NULL` for some rows (say, a guest checkout with no linked
customer). Now run this query to find customers who never placed an order:

```sql
-- DANGEROUS: this can silently return zero rows
SELECT customer_id, name
FROM customers
WHERE customer_id NOT IN (SELECT customer_id FROM orders);
```

If even **one row** in `orders.customer_id` is `NULL`, this query returns **no rows at all**,
even if many customers truly have no orders. This happens because of how SQL's three-valued
logic works.

**Why this happens:** `NOT IN (a, b, NULL)` is checked as `x <> a AND x <> b AND x <> NULL`.
Comparing anything to `NULL` gives `UNKNOWN`, not `TRUE` or `FALSE`. So the whole `AND` chain
becomes `UNKNOWN` for every row, and `WHERE` only keeps rows where the condition is `TRUE`.
`UNKNOWN` rows get dropped. The result: nothing matches, silently, with no error.

**The fix:** use `NOT EXISTS`, which does not have this problem, because it checks row
existence, not value equality.

```sql
-- SAFE: works correctly even if orders.customer_id has NULLs
SELECT c.customer_id, c.name
FROM customers c
WHERE NOT EXISTS (
    SELECT 1 FROM orders o WHERE o.customer_id = c.customer_id
);
```

If you must use `NOT IN`, guard it by filtering out `NULL` explicitly:

```sql
SELECT customer_id, name
FROM customers
WHERE customer_id NOT IN (
    SELECT customer_id FROM orders WHERE customer_id IS NOT NULL
);
```

But the safest habit is: **default to `NOT EXISTS` for "not matching" queries.** Say this rule
out loud in interviews — it shows you understand `NULL` semantics, which many candidates miss.

### CTEs with WITH

A **CTE** (Common Table Expression) is a named, temporary result set that you define using
`WITH`, then use later in the same query. It works like a subquery in `FROM`, but it is easier
to read, especially when you have several steps.

```sql
WITH dept_avg AS (
    SELECT dept_id, AVG(salary) AS avg_salary
    FROM employees
    GROUP BY dept_id
)
SELECT e.emp_name, e.salary, d.avg_salary
FROM employees e
JOIN dept_avg d ON d.dept_id = e.dept_id
WHERE e.salary > d.avg_salary;
```

Compare this to writing the same logic as a nested subquery in `FROM` — the CTE version reads
top to bottom, like a set of labeled steps. This matters a lot in interviews: a clean CTE shows
the interviewer you can structure a hard query, not just produce a correct one.

You can chain multiple CTEs, and later CTEs can use earlier ones:

```sql
WITH order_totals AS (
    SELECT customer_id, SUM(total_amount) AS total_spent
    FROM orders
    GROUP BY customer_id
),
top_spenders AS (
    SELECT customer_id, total_spent
    FROM order_totals
    WHERE total_spent > 10000
)
SELECT c.name, t.total_spent
FROM top_spenders t
JOIN customers c ON c.customer_id = t.customer_id
ORDER BY t.total_spent DESC;
```

**A Postgres-specific note:** in older Postgres versions (before 12), a CTE was always
"materialized" — computed fully once, as a separate step, acting like an optimization fence.
From Postgres 12 onward, the planner can **inline** a simple CTE into the main query, the same
as a subquery, unless you write `MATERIALIZED` explicitly. If you want the old, always-separate
behavior, write `WITH x AS MATERIALIZED (...)`. This is a good detail to mention if the
interviewer asks "is a CTE always faster or slower than a subquery?" — the honest answer is "it
depends on the Postgres version and the planner's choice."

### Recursive CTEs

A **recursive CTE** is a `WITH` block that refers to itself. It is the standard way to walk a
tree or a chain, such as a manager-to-employee hierarchy, or generate a sequence of numbers or
dates.

A recursive CTE has two parts, joined by `UNION` or `UNION ALL`:

1. **Base case** (also called the anchor): the starting rows.
2. **Recursive case**: a query that refers back to the CTE's own name, run again and again,
   until it produces no new rows.

**Example: print the manager chain for one employee (org hierarchy, HR schema).**

```sql
WITH RECURSIVE manager_chain AS (
    -- Base case: start with the employee we care about
    SELECT emp_id, emp_name, manager_id, 1 AS level
    FROM employees
    WHERE emp_id = 101

    UNION ALL

    -- Recursive case: go up to each employee's manager
    SELECT e.emp_id, e.emp_name, e.manager_id, mc.level + 1
    FROM employees e
    JOIN manager_chain mc ON e.emp_id = mc.manager_id
)
SELECT * FROM manager_chain
ORDER BY level;
```

This starts at employee `101`, then joins `employees` to `manager_chain` again and again,
each time moving one level up, until it reaches an employee whose `manager_id` matches nobody
left in the chain (usually the CEO, whose `manager_id` is `NULL`).

**Example: print the full org tree downward, from the top, with indentation.**

```sql
WITH RECURSIVE org_tree AS (
    -- Base case: top-level employees (no manager)
    SELECT emp_id, emp_name, manager_id, 1 AS depth,
           emp_name::text AS path
    FROM employees
    WHERE manager_id IS NULL

    UNION ALL

    -- Recursive case: find direct reports of everyone found so far
    SELECT e.emp_id, e.emp_name, e.manager_id, ot.depth + 1,
           ot.path || ' > ' || e.emp_name
    FROM employees e
    JOIN org_tree ot ON e.manager_id = ot.emp_id
)
SELECT REPEAT('  ', depth - 1) || emp_name AS org_chart, path
FROM org_tree
ORDER BY path;
```

This time the base case starts from the top (`manager_id IS NULL`), and the recursive case
walks downward, finding direct reports at each step. The `path` column builds a readable
breadcrumb, like `CEO > VP Eng > Team Lead`.

**Example: generate a series of numbers or dates without a real table.**

```sql
-- Generate numbers 1 to 10 using a recursive CTE
WITH RECURSIVE numbers AS (
    SELECT 1 AS n
    UNION ALL
    SELECT n + 1 FROM numbers WHERE n < 10
)
SELECT n FROM numbers;
```

```sql
-- Generate every date from 2024-01-01 to 2024-01-07
WITH RECURSIVE date_series AS (
    SELECT DATE '2024-01-01' AS d
    UNION ALL
    SELECT d + INTERVAL '1 day' FROM date_series WHERE d < DATE '2024-01-07'
)
SELECT d FROM date_series;
```

In real Postgres code, you would normally use the built-in `generate_series()` function for
this instead of a recursive CTE — it is simpler and faster. Interviewers still ask for the
recursive CTE version, to test if you understand the base case / recursive case pattern.

**Safety tip:** always make sure the recursive case has a condition that eventually becomes
false (like `WHERE n < 10`). Without one, the recursion never stops. Do not rely on database
defaults — always write a clear stopping condition yourself.

## The Questions They Ask

**Q1: Write a query using a correlated subquery to find employees who earn more than their
department's average salary.**

Answer:

```sql
SELECT e.emp_name, e.salary, e.dept_id
FROM employees e
WHERE e.salary > (
    SELECT AVG(e2.salary)
    FROM employees e2
    WHERE e2.dept_id = e.dept_id
);
```

Explain why it is correlated: the inner query's `WHERE e2.dept_id = e.dept_id` uses `e.dept_id`
from the outer row. This means, conceptually, the inner average is computed once per
department context, tied to the current outer row.

**Follow-up:** "Can you write this without a correlated subquery?" Yes — rewrite using a CTE or
derived table with `GROUP BY`, then join:

```sql
WITH dept_avg AS (
    SELECT dept_id, AVG(salary) AS avg_salary
    FROM employees
    GROUP BY dept_id
)
SELECT e.emp_name, e.salary, e.dept_id
FROM employees e
JOIN dept_avg d ON d.dept_id = e.dept_id
WHERE e.salary > d.avg_salary;
```

This version computes each department's average once, then joins. On a large table, this often
runs faster than the correlated version, because the average is not recomputed per row.

**Q2: Why does NOT EXISTS beat NOT IN? Give an example.**

Answer: `NOT IN` fails silently when the subquery's list contains a `NULL`. Because SQL
compares using three-valued logic (`TRUE`, `FALSE`, `UNKNOWN`), a `NULL` in the `NOT IN` list
turns every comparison into `UNKNOWN`, and the whole query returns zero rows — even when
correct answers exist.

```sql
-- If orders.customer_id has even one NULL row, this returns 0 rows, always
SELECT customer_id, name
FROM customers
WHERE customer_id NOT IN (SELECT customer_id FROM orders);

-- This is correct regardless of NULLs
SELECT c.customer_id, c.name
FROM customers c
WHERE NOT EXISTS (
    SELECT 1 FROM orders o WHERE o.customer_id = c.customer_id
);
```

`NOT EXISTS` checks row presence directly and is not affected by `NULL` values in the inner
table's columns. It is also usually just as fast, or faster, because the planner can stop as
soon as it confirms no match exists.

**Follow-up:** "Would you always use NOT EXISTS then?" Say: "For 'not matching' queries, yes,
it is the safer default. For plain `IN` (not `NOT IN`), the `NULL` problem does not apply the
same way, so `IN` is usually fine there."

**Q3: Print the org hierarchy for a company: each employee, their manager, and their depth in
the tree.**

Answer: use a recursive CTE, starting from the top of the org (employees with no manager), and
walk down.

```sql
WITH RECURSIVE org_tree AS (
    SELECT emp_id, emp_name, manager_id, 1 AS depth
    FROM employees
    WHERE manager_id IS NULL

    UNION ALL

    SELECT e.emp_id, e.emp_name, e.manager_id, ot.depth + 1
    FROM employees e
    JOIN org_tree ot ON e.manager_id = ot.emp_id
)
SELECT emp_id, emp_name, depth
FROM org_tree
ORDER BY depth, emp_id;
```

Walk the interviewer through the base case (top of the tree) and the recursive case (children
of rows found so far), and mention the stopping condition: recursion ends naturally when a
level produces no more matching rows.

**Follow-up:** "How would you detect a cycle, like an employee wrongly set as their own
manager's manager?" Add a `visited` array to the CTE, and stop once the current `emp_id` is
already inside it: `WHERE NOT e.emp_id = ANY(ot.visited)`, carrying `ot.visited || e.emp_id`
forward each step. This turns an infinite loop into a query that stops safely.

**Q4: What is the difference between a subquery and a CTE? Are they always the same speed?**

Answer: a non-recursive CTE and an equivalent subquery in `FROM` often produce the same result.
The difference is mainly readability, and, sometimes, planning behavior. In Postgres 12+, the
planner can inline a simple CTE, treating it like a subquery, unless you force materialization
with `MATERIALIZED`. Say clearly: "It depends on the Postgres version and whether the CTE is
referenced once or many times."

## Rapid-Fire

- **What is a scalar subquery?** A subquery that returns exactly one row and one column, usable
  as a single value.
- **What is a correlated subquery?** A subquery that uses a column from the outer query, so it
  can conceptually run once per outer row.
- **IN vs EXISTS — main difference?** `IN` checks value membership in a list; `EXISTS` checks
  if any row exists, without caring about actual values.
- **Why avoid NOT IN with a subquery?** If the subquery's result contains a `NULL`, `NOT IN`
  returns zero rows for the whole query, silently.
- **What fixes the NOT IN / NULL problem?** Use `NOT EXISTS`, or filter `NULL` out of the
  subquery with `IS NOT NULL`.
- **What is a CTE?** A named temporary result set defined with `WITH`, used later in the same
  query, for readability.
- **What is a recursive CTE made of?** A base case (anchor) and a recursive case, joined by
  `UNION` or `UNION ALL`, that refers back to itself.
- **Why must a subquery in FROM have an alias?** Postgres requires every derived table to have
  a name, since it acts like a temporary table.
- **Does Postgres always materialize a CTE?** No. From Postgres 12 onward, simple CTEs can be
  inlined by the planner, unless you write `MATERIALIZED` explicitly.
- **What stops infinite recursion in a recursive CTE?** A `WHERE` condition in the recursive
  case that eventually becomes false for every remaining row.

## Common Traps & Mistakes

- **Using NOT IN on a nullable column.** The single most common mistake here. Always ask: "Can
  this column contain NULL?" If yes, avoid `NOT IN` for it.
- **Forgetting the alias on a FROM subquery.** Postgres throws a syntax error if a derived
  table has no name.
- **Writing a correlated subquery without thinking about performance.** It is easy to write a
  correct one that is slow on a large table. Mention `EXPLAIN ANALYZE` and a join rewrite
  (Chapter 5) if performance matters.
- **Assuming a CTE is always slower or faster than a subquery.** It depends on the Postgres
  version and whether the planner inlines it. Do not state this as a fixed rule.
- **Using `UNION` instead of `UNION ALL` in a recursive CTE.** `UNION` removes duplicates,
  which adds extra work and can hide real duplicate paths in a graph.
- **No stopping condition in a recursive CTE.** This causes infinite recursion. Always add a
  clear `WHERE` filter in the recursive case, plus a `visited` array if cycles are possible.
- **Mixing up EXISTS and IN performance assumptions.** Postgres's planner often rewrites both
  into similar plans. The difference that matters here is correctness with `NULL`, not speed.
- **Using SELECT \* inside EXISTS and thinking it matters.** It does not matter what columns you
  select inside `EXISTS` — only row existence is checked. `SELECT 1` is a common convention.
