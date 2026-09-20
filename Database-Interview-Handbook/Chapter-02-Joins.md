# Chapter 2 — Joins

A join combines rows from two or more tables using a matching condition. Almost every real query touches more than one table, so interviewers use joins to test if you can read a schema and reason about which rows show up in the result. This chapter uses Schema A (departments, employees) from the design brief.

## Key Concepts

### The setup

We use two tables:

```sql
departments(dept_id PK, dept_name)
employees(emp_id PK, emp_name, dept_id FK, manager_id FK -> employees.emp_id,
          salary, hire_date)
```

`manager_id` points to another row in the same `employees` table. This is what makes the self-join example work later.

A simple way to think about any join: first picture the **cross product** (every row of table A paired with every row of table B). Then a join is a filter on top of that cross product, plus a rule for what happens to rows that do not find a match. That one mental model explains all six join types below.

### INNER JOIN

An inner join keeps only the rows where the join condition matches on both sides. If an employee's `dept_id` does not match any row in `departments`, that employee is dropped.

```sql
SELECT e.emp_name, d.dept_name
FROM employees e
INNER JOIN departments d ON e.dept_id = d.dept_id;
```

**Which rows survive:** only rows with a match on both sides. Think of it as the overlap in a Venn diagram of the two tables.

### LEFT JOIN (LEFT OUTER JOIN)

A left join keeps every row from the left table. If a matching row exists on the right, it is attached. If not, the right-side columns are filled with `NULL`.

```sql
SELECT e.emp_name, d.dept_name
FROM employees e
LEFT JOIN departments d ON e.dept_id = d.dept_id;
```

**Which rows survive:** all rows from the left table, always. This is the join you use when you must not lose any row from one side — for example, "list all employees, and show their department if they have one."

### RIGHT JOIN (RIGHT OUTER JOIN)

A right join is the mirror of a left join. It keeps every row from the right table, and fills `NULL` on the left side when there is no match.

```sql
SELECT e.emp_name, d.dept_name
FROM employees e
RIGHT JOIN departments d ON e.dept_id = d.dept_id;
```

**Which rows survive:** all rows from the right table, always. In practice, most engineers avoid `RIGHT JOIN` and just swap the table order to write a `LEFT JOIN` instead, because it reads more naturally. But you should still recognize it and know it behaves the same way, mirrored.

### FULL OUTER JOIN

A full outer join keeps every row from both tables. Where a match exists, columns from both sides are filled in. Where no match exists on one side, that side is `NULL`.

```sql
SELECT e.emp_name, d.dept_name
FROM employees e
FULL OUTER JOIN departments d ON e.dept_id = d.dept_id;
```

**Which rows survive:** everything. Unmatched employees show up with `dept_name = NULL`. Unmatched departments (no employees at all) show up with `emp_name = NULL`. This is the join you use when you need to see gaps on both sides at once, for example, "show me every employee and every department, matched where possible."

MySQL note: MySQL does not support `FULL OUTER JOIN` directly. You get the same result with `LEFT JOIN ... UNION ... RIGHT JOIN`, or a `LEFT JOIN UNION` with a `RIGHT JOIN` that excludes the overlap.

### CROSS JOIN

A cross join returns the full cross product: every row of table A paired with every row of table B. There is no join condition, or the condition is always true.

```sql
SELECT e.emp_name, d.dept_name
FROM employees e
CROSS JOIN departments d;
```

If `employees` has 50 rows and `departments` has 5 rows, this returns 250 rows. **Which rows survive:** all combinations, no filtering at all. Real use cases are rare — one common one is generating a full calendar of dates crossed with a list of stores, so every store has a row for every day, even with zero sales.

### SELF JOIN

A self join is not a new join type. It is a regular join where a table is joined to itself, usually because one row refers to another row in the same table. `employees.manager_id` is exactly this case: it points back to `employees.emp_id`.

To show each employee with their manager's name, join the table to itself with two aliases — one alias playing "the employee," the other playing "the manager":

```sql
SELECT e.emp_name        AS employee,
       m.emp_name        AS manager
FROM employees e
LEFT JOIN employees m ON e.manager_id = m.emp_id;
```

Notice this uses a `LEFT JOIN`, not an `INNER JOIN`. Why? Because the CEO (or anyone at the top) has `manager_id = NULL`. An inner join would drop that person entirely, since `NULL` never matches. A left join keeps them, with `manager = NULL`.

**Which rows survive:** every employee, because the left join guarantees it. This is the standard interview answer for "show employees with their manager's name" — always use `LEFT JOIN` for this, not `INNER JOIN`, unless the interviewer explicitly says "only employees who have a manager."

### The ON vs WHERE trap for outer joins

This is one of the most common interview gotchas. Read it carefully.

When you write an outer join (`LEFT`, `RIGHT`, `FULL`), the `ON` clause and the `WHERE` clause behave differently:

- A condition in `ON` is applied **while building the join**, before deciding which rows are unmatched. It can filter the right-side table without removing left-side rows.
- A condition in `WHERE` is applied **after** the join is complete. If that condition references a column from the outer side, it will throw away the `NULL`-filled rows that the outer join was supposed to keep.

Example. Suppose we want all employees, with department info only for the "Engineering" department (department info should be blank for everyone else).

**Correct — filter in ON:**

```sql
SELECT e.emp_name, d.dept_name
FROM employees e
LEFT JOIN departments d
       ON e.dept_id = d.dept_id AND d.dept_name = 'Engineering';
```

Here, every employee still appears. Only the ones in Engineering get a `dept_name`; everyone else gets `NULL` in `dept_name`, but they are still in the result.

**Wrong — filter in WHERE (turns it into an INNER JOIN):**

```sql
SELECT e.emp_name, d.dept_name
FROM employees e
LEFT JOIN departments d ON e.dept_id = d.dept_id
WHERE d.dept_name = 'Engineering';
```

Here, the `LEFT JOIN` first produces all employees, filling `NULL` for those without a matching department, or in departments other than Engineering. Then `WHERE d.dept_name = 'Engineering'` runs. For any row where `d.dept_name` is `NULL`, the condition `NULL = 'Engineering'` is not true — it is unknown, and `WHERE` drops rows unless the condition is true. So all non-Engineering employees, and all employees with no department, silently disappear. The query "acts like" an `INNER JOIN`, even though you wrote `LEFT JOIN`.

**The rule to remember:** for outer joins, put conditions that filter the *outer* (nullable) side inside `ON`. Put conditions that filter the *preserved* side (or a condition on a column you know is never `NULL`, like a value already fixed on the left table) in `WHERE` — that is safe, because it does not depend on the join's outcome.

If your interviewer asks "why does my LEFT JOIN act like an INNER JOIN?" — this is almost always the answer: a `WHERE` clause is filtering out the `NULL` rows the left join created.

### Multi-table joins

Real queries often join three or more tables. Using Schema B (customers, orders, order_items, products), here is order line items with customer name and product name:

```sql
SELECT c.name AS customer_name,
       o.order_id,
       p.name AS product_name,
       oi.quantity,
       oi.unit_price
FROM orders o
JOIN customers c   ON o.customer_id = c.customer_id
JOIN order_items oi ON oi.order_id  = o.order_id
JOIN products p     ON p.product_id = oi.product_id;
```

Tips for multi-table joins in an interview:

1. Draw the tables and the foreign keys before writing SQL. This avoids joining on the wrong column.
2. Join order in the `FROM`/`JOIN` clauses does not usually change the result for inner joins — the query planner picks its own order. It does matter for readability, and it matters a lot for outer joins, since the order affects what gets preserved.
3. If you mix `LEFT JOIN` with later `INNER JOIN`s, watch out: an inner join later in the chain can silently undo the "keep all rows" effect of an earlier left join, if it filters on a column that is `NULL` for the unmatched rows. Same root cause as the `ON` vs `WHERE` trap.

### Semi-join and anti-join (EXISTS / NOT EXISTS)

A **semi-join** answers: "give me rows from A that have at least one match in B" — but only columns from A, and it does not duplicate A's rows even if there are multiple matches in B. An **anti-join** is the opposite: "give me rows from A that have no match in B at all."

Postgres does not have a `SEMI JOIN` keyword. You write these with `EXISTS` and `NOT EXISTS`.

**Semi-join example — departments that have at least one employee:**

```sql
SELECT d.dept_name
FROM departments d
WHERE EXISTS (
    SELECT 1 FROM employees e WHERE e.dept_id = d.dept_id
);
```

**Anti-join example — departments with no employees:**

```sql
SELECT d.dept_name
FROM departments d
WHERE NOT EXISTS (
    SELECT 1 FROM employees e WHERE e.dept_id = d.dept_id
);
```

**Anti-join example — employees with no manager (top of the hierarchy):**

```sql
SELECT e.emp_name
FROM employees e
WHERE NOT EXISTS (
    SELECT 1 FROM employees m WHERE m.emp_id = e.manager_id
);
```

(For this particular case, `WHERE manager_id IS NULL` is simpler and works too, since `manager_id` is a direct column. `NOT EXISTS` is the general pattern you need when the "no match" condition is more complex, or spans another table.)

You can also write semi-joins and anti-joins using `LEFT JOIN ... WHERE right.key IS NULL` for the anti-join case:

```sql
SELECT d.dept_name
FROM departments d
LEFT JOIN employees e ON e.dept_id = d.dept_id
WHERE e.emp_id IS NULL;
```

This works: the left join keeps every department, filling `NULL` for departments with no employees. The `WHERE e.emp_id IS NULL` then keeps only those. This is a safe use of `WHERE` on the outer side, because we are deliberately looking for the `NULL` rows, not accidentally destroying them.

**Which one should you use — `NOT EXISTS` or `LEFT JOIN ... IS NULL`?** Both give the same result here. `NOT EXISTS` is usually preferred: it is easier to read as "no match exists," and in Postgres it often performs at least as well, sometimes better, because the planner can stop at the first match instead of building the full join.

## The Questions They Ask

**Q1: What is the difference between INNER JOIN and LEFT JOIN?**
`INNER JOIN` keeps only rows with a match on both sides. `LEFT JOIN` keeps all rows from the left table, filling `NULL` on the right side when there is no match.
*Follow-up:* "What happens if there are two matching rows on the right side?" — Answer: the left row gets duplicated once per match. Joins can multiply row counts; always check for this when a total looks too high.

**Q2: How do you find rows that exist in table A but not in table B?**
Use `NOT EXISTS` with a correlated subquery, or a `LEFT JOIN` plus `WHERE right.key IS NULL`. Avoid `NOT IN` with a subquery that can return `NULL` values — if the subquery returns even one `NULL`, `NOT IN` returns no rows at all, because `x NOT IN (1, NULL)` evaluates to unknown for every `x`. This is a real bug people hit in production.
*Follow-up:* "Why is `NOT EXISTS` usually safer than `NOT IN`?" — Because `NOT EXISTS` handles `NULL`s in the subquery correctly; `NOT IN` does not.

**Q3: Why does my LEFT JOIN act like an INNER JOIN?**
Because a `WHERE` clause is filtering on a column from the right (outer) side, and that column is `NULL` for the unmatched rows. `NULL` never satisfies a `WHERE` condition like `= 'value'`, so those rows get dropped, even though the join itself preserved them. Fix it by moving that condition into the `ON` clause.
*Follow-up:* "Does this happen with `IS NULL` checks too?" — No, if you are explicitly checking `WHERE right.col IS NULL`, that is the correct, intentional way to find unmatched rows (the anti-join pattern above). The trap is checking `WHERE right.col = 'something'`, not `IS NULL`.

**Q4: What is a self-join, and when do you need one?**
A self-join joins a table to itself, using two aliases, when a row references another row in the same table — like an employee referencing their manager, or a category referencing its parent category. Always consider whether it should be a `LEFT JOIN` (to keep rows with no self-reference, like the top manager) or `INNER JOIN` (if you only want rows that do have a match).
*Follow-up:* "How would you find employees who earn more than their manager?" —
```sql
SELECT e.emp_name, e.salary, m.emp_name AS manager_name, m.salary AS manager_salary
FROM employees e
JOIN employees m ON e.manager_id = m.emp_id
WHERE e.salary > m.salary;
```
Here `INNER JOIN` is correct, because an employee with no manager cannot be compared to a manager's salary at all — there's nothing to compare against.

**Q5: What is the difference between a semi-join and a regular inner join?**
An inner join returns matched columns from both tables, and duplicates the left row once per match on the right. A semi-join only checks "does a match exist," returns only columns from the left table, and never duplicates left rows, no matter how many matches exist on the right. `EXISTS` is how you write a semi-join in standard SQL.
*Follow-up:* "Could you rewrite a semi-join using DISTINCT and a normal join instead?" — Yes, `SELECT DISTINCT d.dept_name FROM departments d JOIN employees e ON ...` gives the same rows. But `EXISTS` is usually clearer and can be faster, since the planner can stop scanning after the first match.

**Q6: Explain nested loop join, hash join, and merge join. When does the optimizer pick each one?**
These are three ways the database engine can physically execute a join — the SQL you write does not choose the algorithm; the query planner does, based on table sizes, indexes, and sort order.

- **Nested loop join:** for each row in the outer table, scan the inner table for matches. Simple, and very fast when the outer table is small and there is an index on the inner table's join column (so the "scan" is really an index lookup, not a full scan). Slow when both tables are large and there is no index, since it becomes close to a full cross product scan.
- **Hash join:** build an in-memory hash table on the smaller table's join key, then scan the larger table and probe the hash table for matches. Good for large, unsorted tables with no useful index, as long as the smaller table fits comfortably in memory (or Postgres's `work_mem`). Only works for equality conditions (`=`), not ranges.
- **Merge join:** if both inputs are already sorted on the join key (or the planner sorts them first), walk through both in order, like merging two sorted lists. Efficient when data is already sorted — for example, both sides use an index that stores rows in join-key order — because it avoids random access or building a hash table.

*Follow-up:* "How would you check which one Postgres actually used?" — Run `EXPLAIN ANALYZE` on the query. It prints the chosen plan, including `Nested Loop`, `Hash Join`, or `Merge Join`, along with estimated and actual row counts. (Query plans are covered in more depth in Chapter 7.)

**Q7: You have a LEFT JOIN with a subsequent INNER JOIN in the same query. Can it silently break your LEFT JOIN?**
Yes. If the inner join's `ON` condition needs a column that came from the left-joined table, and that column is `NULL` for unmatched rows, those rows will fail the inner join's condition and get dropped. The fix is either to make the second join also a `LEFT JOIN`, or to move the filtering condition to a place that tolerates `NULL`.

## Rapid-Fire

- **INNER JOIN** → only matched rows on both sides.
- **LEFT JOIN** → all rows from the left table, `NULL` on the right when unmatched.
- **RIGHT JOIN** → all rows from the right table, `NULL` on the left when unmatched; rarely used, usually rewritten as a `LEFT JOIN` with tables swapped.
- **FULL OUTER JOIN** → all rows from both tables, `NULL` on whichever side has no match.
- **CROSS JOIN** → full cross product, no matching condition.
- **SELF JOIN** → a table joined to itself using two aliases, for hierarchical or self-referencing data.
- **Filter in ON vs WHERE (outer joins)** → `ON` filters before deciding what is "unmatched"; `WHERE` filters after, and can silently drop the `NULL`-filled rows an outer join was meant to keep.
- **Semi-join** → "does a match exist" → write with `EXISTS`; no duplication, only left-table columns.
- **Anti-join** → "no match exists" → write with `NOT EXISTS`, or `LEFT JOIN ... WHERE right.key IS NULL`.
- **NOT IN vs NOT EXISTS** → avoid `NOT IN` if the subquery can return `NULL`; use `NOT EXISTS` instead.
- **Nested loop join** → good for small outer table plus an index on the inner table.
- **Hash join** → good for large unsorted tables with an equality condition.
- **Merge join** → good when both sides are already sorted on the join key.
- **Multiple joins, order matters for outer joins** → a later `INNER JOIN` can undo an earlier `LEFT JOIN`.

## Common Traps & Mistakes

- **Putting an outer-side filter in WHERE instead of ON.** This is the single most common join bug in interviews and in real code. Always ask: "is this condition supposed to limit which rows exist, or just which right-side data gets attached?" If it's the second, it belongs in `ON`.
- **Using `NOT IN` with a subquery that can contain `NULL`.** This silently returns zero rows. Prefer `NOT EXISTS`.
- **Forgetting `LEFT JOIN` on a self-join for hierarchy roots.** An `INNER JOIN` between `employees` and itself on `manager_id = emp_id` will quietly drop the CEO, or anyone with no manager, from the result.
- **Assuming join order changes the result for INNER JOINs.** It usually does not — the planner reorders inner joins freely. But it does affect readability, and it can matter a lot for outer joins.
- **Not checking for row duplication after a join.** If the right table has multiple matching rows, the left row is repeated once per match. This can silently inflate `SUM()` or `COUNT()` in a later aggregation — always check cardinality before joining into an aggregate query (Chapter 3 covers this in detail).
- **Confusing CROSS JOIN with a forgotten join condition.** If you accidentally omit the `ON` clause on a regular `JOIN` in some SQL dialects, or write two tables separated by a comma in the old-style `FROM a, b` syntax, you silently get a cross product. Always double check your row count matches expectations.
- **Thinking join algorithm is something you control in the SQL.** You cannot write `USE HASH JOIN` in standard SQL (Postgres has planner hints only through extensions, not by default). The algorithm is chosen by the optimizer based on statistics, indexes, and memory settings. You influence it indirectly — by adding indexes, updating table statistics, or adjusting `work_mem` — not by rewriting the join syntax.
