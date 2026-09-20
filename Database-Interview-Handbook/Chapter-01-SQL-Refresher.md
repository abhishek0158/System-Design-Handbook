# Chapter 1 — SQL Refresher (fast)

This chapter is a fast reset on SQL basics. It is not for beginners. It focuses on the small
details that interviewers use to check if you *really* understand SQL, not just write it.

## Key Concepts

### 1. The mental model of a query

A `SELECT` query does not run top to bottom, the way it is written. It runs in a fixed
**logical order**. The database builds a result set step by step, and each step feeds the
next one.

Here is the logical order for a query with all clauses:

```
FROM   →  WHERE  →  GROUP BY  →  HAVING  →  SELECT  →  DISTINCT  →  ORDER BY  →  LIMIT
```

Think of it as a pipeline. Each stage takes rows from the stage before it and passes a
smaller or reshaped set of rows to the next stage.

- **FROM**: pick the source tables. Join them if there is more than one.
- **WHERE**: filter individual rows, before any grouping.
- **GROUP BY**: collapse rows into groups, one row per group.
- **HAVING**: filter groups, after they are formed.
- **SELECT**: pick and compute the output columns.
- **DISTINCT**: remove duplicate output rows.
- **ORDER BY**: sort the final rows.
- **LIMIT**: cut the result down to N rows.

This order is why some things that "look right" fail.

**Example: you cannot use a SELECT alias in WHERE.**

```sql
-- This FAILS in PostgreSQL:
SELECT emp_id, salary * 12 AS annual_salary
FROM employees
WHERE annual_salary > 600000;
```

This fails because `WHERE` runs before `SELECT` in the logical order. At the point `WHERE`
runs, the alias `annual_salary` does not exist yet. The fix is to repeat the expression, or
use a subquery, or use `HAVING` (only if you are also grouping).

```sql
-- Fix 1: repeat the expression
SELECT emp_id, salary * 12 AS annual_salary
FROM employees
WHERE salary * 12 > 600000;

-- Fix 2: wrap it in a subquery
SELECT * FROM (
  SELECT emp_id, salary * 12 AS annual_salary
  FROM employees
) t
WHERE t.annual_salary > 600000;
```

**Aliases DO work in `ORDER BY` and `GROUP BY`** in PostgreSQL, because those clauses run
after `SELECT` (or, for `GROUP BY`, Postgres allows it as a convenience even though standard
SQL logical order says `GROUP BY` runs first — more on this below).

```sql
-- This WORKS: ORDER BY runs after SELECT
SELECT emp_id, salary * 12 AS annual_salary
FROM employees
ORDER BY annual_salary DESC;
```

**Why `HAVING` exists separately from `WHERE`.** `HAVING` filters on grouped, aggregated
values. `WHERE` cannot do this, because at the time `WHERE` runs, groups do not exist yet.

```sql
-- Departments with average salary above 80000
SELECT dept_id, AVG(salary) AS avg_salary
FROM employees
GROUP BY dept_id
HAVING AVG(salary) > 80000;
```

If you wrote `WHERE AVG(salary) > 80000`, PostgreSQL raises an error. Aggregate functions
are not allowed in `WHERE`, because `WHERE` runs before grouping happens.

### 2. NULL and three-valued logic

`NULL` means "unknown" or "missing value". It is not zero, and it is not an empty string.
This one idea causes more interview mistakes than anything else in basic SQL.

**Rule 1: `NULL` is never equal to `NULL`.** Comparing `NULL = NULL` gives `NULL`, not
`true`. This is because both sides are "unknown" — the database cannot say if two unknown
things are equal.

```sql
SELECT NULL = NULL;   -- returns NULL, not true
SELECT NULL <> NULL;  -- returns NULL, not true either
```

**Rule 2: use `IS NULL` / `IS NOT NULL` to test for NULL.** Never use `= NULL`.

```sql
-- Find employees with no manager (top of the org chart)
SELECT emp_id, emp_name
FROM employees
WHERE manager_id IS NULL;
```

If you write `WHERE manager_id = NULL`, this returns **zero rows**, always. It looks like
valid SQL, and PostgreSQL will not stop you, but the condition never evaluates to true.

**Rule 3: SQL uses three-valued logic: `TRUE`, `FALSE`, `UNKNOWN`.** Any comparison
involving `NULL` gives `UNKNOWN`. A `WHERE` clause keeps only rows where the condition is
`TRUE`. Rows that evaluate to `UNKNOWN` or `FALSE` are both dropped.

```sql
-- salary IS NULL for some rows
SELECT emp_id FROM employees WHERE salary > 50000;
-- rows where salary IS NULL are silently excluded (condition is UNKNOWN, not TRUE)
```

This matters a lot for `NOT IN`. If the list used with `NOT IN` contains even one `NULL`,
the whole condition can never be `TRUE`, so the query returns **zero rows**.

```sql
-- Danger: if any employee has manager_id = NULL... wait, this is about a NULL IN THE LIST
-- Suppose some order_items.product_id can be NULL (bad data)
SELECT product_id, name
FROM products
WHERE product_id NOT IN (
  SELECT product_id FROM order_items   -- if this subquery returns even one NULL, result is empty
);
```

Why does this happen? `NOT IN (1, 2, NULL)` expands to `x <> 1 AND x <> 2 AND x <> NULL`.
The last part is always `UNKNOWN`. `AND` with `UNKNOWN` can never become `TRUE`. So the
whole row is dropped, for every row. The safe fix is `NOT EXISTS`, which does not have this
trap.

```sql
-- Safe version, using NOT EXISTS
SELECT p.product_id, p.name
FROM products p
WHERE NOT EXISTS (
  SELECT 1 FROM order_items oi WHERE oi.product_id = p.product_id
);
```

**Rule 4: aggregate functions ignore `NULL` values**, except `COUNT(*)`.

```sql
-- If some employees have salary = NULL:
SELECT
  COUNT(*)      AS total_rows,      -- counts all rows, NULL or not
  COUNT(salary) AS rows_with_salary, -- counts only non-NULL salary
  AVG(salary)   AS avg_salary,       -- average of non-NULL salaries only
  SUM(salary)   AS sum_salary        -- sum of non-NULL salaries only
FROM employees;
```

If every row has `salary = NULL`, `SUM(salary)` and `AVG(salary)` return `NULL`, not `0`.
This trips people up when they expect `0`.

**Rule 5: `NULL` in joins.** A join matches rows using `=`. Since `NULL = NULL` is never
`TRUE`, `NULL` values never match anything in a join, even another `NULL`.

```sql
-- If some employees.dept_id is NULL, they will NOT match any department
-- in a plain INNER JOIN, and will be missing from the result.
SELECT e.emp_name, d.dept_name
FROM employees e
JOIN departments d ON e.dept_id = d.dept_id;
```

An employee with `dept_id IS NULL` disappears from this result. Use a `LEFT JOIN` if you
want to keep them (their `dept_name` will show as `NULL`). Joins are covered in depth in
Chapter 2.

**Rule 6: `COALESCE` replaces `NULL` with a default value.** This is the standard way to
handle `NULL` in output or in calculations.

```sql
-- Show 'No Manager' instead of NULL
SELECT emp_name, COALESCE(manager_id::text, 'No Manager') AS manager
FROM employees;

-- Treat missing salary as 0 in a sum
SELECT SUM(COALESCE(salary, 0)) AS total_payroll
FROM employees;
```

### 3. DISTINCT

`DISTINCT` removes duplicate rows from the final result. It runs late, after `SELECT`
builds the row.

```sql
-- Unique list of countries that have customers
SELECT DISTINCT country
FROM customers;
```

`DISTINCT` applies to the **whole row** of selected columns, not to one column only, when
you select more than one column.

```sql
-- This gives unique (country, customer_id) pairs, not just unique countries
SELECT DISTINCT country, customer_id
FROM customers;
```

A common trap: `DISTINCT` is expensive on large tables, because the database must sort or
hash all rows to find duplicates. If you only need to check "does at least one row exist",
use `EXISTS` instead, which can stop at the first match.

### 4. Basic filtering operators

These are simple, but interviewers check that you know the exact behavior.

```sql
-- BETWEEN is inclusive on both ends
SELECT * FROM orders WHERE order_date BETWEEN '2024-01-01' AND '2024-01-31';

-- IN checks membership in a list
SELECT * FROM products WHERE category IN ('Electronics', 'Books');

-- LIKE for pattern match. % = any characters, _ = exactly one character
SELECT * FROM customers WHERE name LIKE 'A%';       -- starts with A
SELECT * FROM customers WHERE name ILIKE 'a%';      -- case-insensitive (Postgres only)

-- Combining conditions: AND has higher precedence than OR, use parentheses to be clear
SELECT * FROM orders
WHERE status = 'PAID' AND (total_amount > 1000 OR customer_id = 5);
```

Always use parentheses when mixing `AND` and `OR`. Do not rely on remembering precedence
rules during an interview. It also makes the query easier for others to read.

### 5. A quick note on common data types

- `INTEGER` / `BIGINT`: whole numbers. Use `BIGINT` for IDs that may grow past ~2 billion.
- `NUMERIC(p, s)`: exact decimal number, `p` total digits, `s` digits after the decimal
  point. Use this for money. Never use `FLOAT` for money — floats lose small amounts of
  precision.
- `TEXT` / `VARCHAR(n)`: in PostgreSQL, both store text the same way. `VARCHAR(n)` just adds
  a length check. There is no performance gain from choosing `VARCHAR` over `TEXT`.
- `TIMESTAMP` vs `TIMESTAMPTZ`: `TIMESTAMPTZ` stores a point in time and converts it to the
  viewer's time zone on display. Plain `TIMESTAMP` stores a naive date and time, with no time
  zone. For most applications, prefer `TIMESTAMPTZ`.
- `BOOLEAN`: true, false, or `NULL` (unknown), not just true/false.

## The Questions They Ask

**Q1: What is the logical order of execution of a SELECT query? Why does it matter?**
Answer: `FROM → WHERE → GROUP BY → HAVING → SELECT → DISTINCT → ORDER BY → LIMIT`. It
matters because a clause can only use things that exist in an earlier stage. For example, a
column alias created in `SELECT` cannot be used in `WHERE`, because `WHERE` runs first.
*Follow-up: can you use a SELECT alias in GROUP BY?* Yes, in PostgreSQL, `GROUP BY` accepts
an output alias as a convenience. Standard SQL technically does not guarantee this, but
Postgres, MySQL, and most modern engines support it.

**Q2: Why does `WHERE manager_id = NULL` return no rows?**
Answer: `NULL` represents an unknown value. Comparing anything to `NULL` with `=` gives
`NULL` (unknown), not `true`. A `WHERE` clause only keeps rows where the condition is
`TRUE`, so rows evaluating to `UNKNOWN` are dropped. The correct form is `WHERE manager_id
IS NULL`.

**Q3: What happens if a subquery used with `NOT IN` returns a `NULL`?**
Answer: The whole query returns zero rows, for every outer row, even ones that clearly
should match. This is because `NOT IN` expands into a chain of `AND`-ed `<>` comparisons,
and any comparison with `NULL` gives `UNKNOWN`, which breaks the whole `AND` chain. Fix by
using `NOT EXISTS`, or by filtering out `NULL` from the subquery with `WHERE product_id IS
NOT NULL`.
*Follow-up: does plain `IN` have the same problem?* No. `IN` only fails silently for
`NOT IN`. A `NULL` in the list for a plain `IN` does not cause the whole query to break —
it simply cannot help a row match if nothing else does.

**Q4: Does `COUNT(*)` and `COUNT(column_name)` give the same result?**
Answer: Not always. `COUNT(*)` counts every row, no matter what is `NULL`. `COUNT(column)`
counts only rows where that column is not `NULL`. If `salary` has `NULL` in some rows,
`COUNT(salary)` will be smaller than `COUNT(*)`.

**Q5: Can you use `DISTINCT` and `GROUP BY` interchangeably?**
Answer: Sometimes, for simple "unique values" cases, but they mean different things.
`GROUP BY` is for grouping rows to compute aggregates per group. `DISTINCT` just removes
duplicate rows from the output. `SELECT DISTINCT dept_id FROM employees` and `SELECT dept_id
FROM employees GROUP BY dept_id` give the same rows, but only `GROUP BY` lets you also
compute `AVG(salary)` or `COUNT(*)` per group.

**Q6: If I do `SUM(salary)` on a set of rows where salary is `NULL` for all of them, what do
I get?**
Answer: `NULL`, not `0`. Aggregate functions other than `COUNT` ignore `NULL` inputs. If
there is nothing to sum, the result is `NULL`. Use `COALESCE(SUM(salary), 0)` if you need
`0` instead.

## Rapid-Fire

- **Q: What is the logical order of a SELECT?**
  A: `FROM → WHERE → GROUP BY → HAVING → SELECT → DISTINCT → ORDER BY → LIMIT`.

- **Q: Can `WHERE` use an aggregate function like `AVG()`?**
  A: No. Use `HAVING` for filtering on aggregates, since it runs after `GROUP BY`.

- **Q: Is `NULL = NULL` true?**
  A: No, it evaluates to `NULL` (unknown). Use `IS NULL` to check for `NULL`.

- **Q: Does `ORDER BY` accept a `SELECT` alias?**
  A: Yes, because `ORDER BY` runs after `SELECT` in the logical order.

- **Q: What does `COUNT(*)` count that `COUNT(col)` does not?**
  A: Rows where `col` is `NULL`.

- **Q: Why is `NOT IN` risky with subqueries?**
  A: If the subquery can return `NULL`, the whole `NOT IN` query returns zero rows. Prefer
  `NOT EXISTS`.

- **Q: Does a `NULL` join key match another `NULL` join key?**
  A: No. `NULL` never equals `NULL`, so join conditions never match on `NULL`.

- **Q: What is `COALESCE` for?**
  A: It returns the first non-`NULL` value from a list of expressions. Common for defaults.

- **Q: `VARCHAR(n)` vs `TEXT` in PostgreSQL?**
  A: Same storage and speed. `VARCHAR(n)` just adds a max-length check.

- **Q: Why prefer `NUMERIC` over `FLOAT` for money?**
  A: `NUMERIC` is exact. `FLOAT` can introduce small rounding errors.

## Common Traps & Mistakes

- **Using a SELECT alias in WHERE.** This fails because `WHERE` runs before `SELECT` in the
  logical order. Repeat the expression, or wrap the query in a subquery.

- **Writing `= NULL` instead of `IS NULL`.** This is valid SQL syntax, so PostgreSQL does not
  give you an error. It just silently returns wrong results (zero rows).

- **Using `NOT IN` with a subquery that can contain `NULL`.** This is one of the most common
  real bugs in production code, not just an interview trick. Always check if the subquery
  column can have `NULL`, or use `NOT EXISTS` as a safer default habit.

- **Assuming `SUM()` or `AVG()` on all-`NULL` input gives `0`.** It gives `NULL`. Wrap with
  `COALESCE` if you need a numeric default.

- **Forgetting that `INNER JOIN` drops rows with `NULL` in the join column.** If you need to
  keep unmatched rows, use `LEFT JOIN` or `FULL OUTER JOIN` (see Chapter 2).

- **Thinking `DISTINCT` only applies to the first column listed.** It applies to the
  combination of all selected columns.

- **Confusing precedence of `AND` and `OR`.** `AND` binds tighter than `OR`. Always add
  parentheses to make the intent explicit, instead of relying on memory during a live coding
  round.

- **Assuming `HAVING` can replace `WHERE` everywhere.** `HAVING` only makes sense after
  `GROUP BY`, and it is slower for row-level filters because it filters after grouping work is
  already done. Use `WHERE` to filter rows early, and `HAVING` only for group-level
  conditions.
