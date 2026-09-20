# Chapter 8 — Indexing

An index is a separate data structure that helps the database find rows fast, without reading the whole table. Interviewers ask about indexing more than almost any other topic, because it tests if you understand how a database actually works, not just how to write SQL. This chapter covers how indexes work, when they help, and when they silently fail to help.

## Key Concepts

### 1. What is an index, and why does it speed up reads?

A table without an index is stored as an unordered pile of rows, spread across disk pages. If you run a query like this:

```sql
SELECT * FROM employees WHERE emp_name = 'Aditi Rao';
```

The database has no way to know where `'Aditi Rao'` is. It must check every row, one by one. This is called a **sequential scan** (or **full table scan**). If the table has 10 million rows, the database reads all 10 million rows, even if only one row matches.

An **index** is a separate structure, stored next to the table, that keeps a sorted copy of one or more columns, along with a pointer back to the real row. Think of it like the index at the back of a textbook. You do not read every page to find "MVCC". You look it up in the index, alphabetically sorted, and it tells you the exact page number.

```sql
CREATE INDEX idx_emp_name ON employees(emp_name);
```

Now the database can jump almost straight to `'Aditi Rao'` using the sorted index, instead of scanning the whole table. This is called an **index scan**.

The trade-off: the index itself takes disk space, and it must be updated every time the table changes. We come back to this cost later.

You can see this difference directly with `EXPLAIN ANALYZE` (covered in depth in Chapter 7). A sequential scan looks like this:

```
Seq Scan on employees  (cost=0.00..18334.00 rows=1 width=64) (actual time=42.1..120.5 rows=1 loops=1)
  Filter: (emp_name = 'Aditi Rao'::text)
  Rows Removed by Filter: 999999
```

Notice `Rows Removed by Filter: 999999` — Postgres read almost every row, then threw away all but one. After adding the index, the same query looks like this:

```
Index Scan using idx_emp_name on employees  (cost=0.42..8.44 rows=1 width=64) (actual time=0.03..0.04 rows=1 loops=1)
  Index Cond: (emp_name = 'Aditi Rao'::text)
```

The cost estimate dropped from about 18,334 to about 8.44, and the actual run time dropped from milliseconds-times-thousands to a fraction of a millisecond. This is the concrete, measurable version of "an index avoids scanning the whole table" — always be ready to show this kind of before/after in an interview, not just describe it in words.

### 2. How a B-tree index works (high level)

Postgres's default index type is a **B-tree** (balanced tree). You do not need to implement one in an interview, but you must explain the idea clearly.

A B-tree is a tree of sorted values, built so that:
- It stays **balanced**: every leaf (the bottom-level node) is the same distance from the root. No path is much longer than another.
- Each node holds many sorted keys, not just one. This keeps the tree short and wide, not tall and thin.
- Lookups compare your search value against keys in a node, and follow one branch down, cutting the search space at each level.

This gives **O(log n)** lookup time: the number of steps grows very slowly as the table grows. Doubling the table size might add only one extra step to the search, instead of doubling the work (which is what a sequential scan does).

```
                [ M ]
              /       \
        [ D, H ]      [ R, W ]
        /  |  \        /  |   \
     A-C  E-G  I-L   N-Q  S-V  X-Z
```

A simplified B-tree: to find "K", start at the root, go right of D and left of R and W is not needed — you land in the I-L range in two hops, not a scan of 26 letters.

Two properties matter most for interviews:
- **Sorted order**: because the B-tree keeps keys sorted, it is also good for **range scans** — `WHERE salary BETWEEN 50000 AND 80000`, or `ORDER BY hire_date`. The database can walk the sorted leaves directly, instead of sorting the result afterward.
- **Log-time lookup**: equality lookups (`WHERE emp_id = 5`) and range lookups are both fast, because both use the sorted structure.

### 3. Clustered vs non-clustered index (and the Postgres vs MySQL difference)

This is one of the most commonly confused topics, and interviewers often use it to check if you understand more than one database engine.

- A **clustered index** stores the actual table rows in the same physical order as the index. There can be only one clustered index per table, because rows can only be physically sorted one way.
- A **non-clustered (secondary) index** is a separate structure. It stores sorted key values plus a pointer to where the real row lives. A table can have many secondary indexes.

**MySQL (InnoDB) note:** InnoDB always stores the table itself as a clustered index, ordered by the primary key. Every secondary index stores the primary key value as its pointer, not a raw disk address. So a secondary index lookup in InnoDB does two steps: find the primary key in the secondary index, then look up the full row using the clustered primary key index. This second step is called a "bookmark lookup" in some databases.

**PostgreSQL is different.** Postgres does **not** have a true clustered index by default. The table is stored as an unordered heap (Postgres calls it a "heap table"). Every index in Postgres, including the primary key index, is a secondary structure that points to a row's physical location using a **TID** (tuple ID: page number + offset in that page). Postgres does have a `CLUSTER` command that can physically reorder a table's rows to match one chosen index, but this is a one-time, manual operation. The table does not stay clustered automatically as new rows are added — you would need to re-run `CLUSTER` again later.

| | MySQL (InnoDB) | PostgreSQL |
|---|---|---|
| Table storage | Clustered on primary key, always | Unordered heap, always |
| Primary key index | Same structure as the table | A secondary index, like any other |
| Secondary index lookup | Index → PK → row (2 steps) | Index → TID → row (also effectively 2 steps, via heap fetch) |
| Manual reordering | Not needed (always clustered) | `CLUSTER table USING index_name` (one-time) |

Say this trade-off out loud in an interview: clustering makes primary-key range scans very fast (rows are physically together), but it makes secondary index lookups a bit slower (extra hop), and it makes inserts in random primary-key order slower (rows must be placed in sorted position). Postgres's heap approach is more uniform: every index costs about the same to use.

Think about a concrete example: `SELECT * FROM orders WHERE order_id BETWEEN 1000 AND 2000`. In InnoDB, if `order_id` is the primary key, these 1,000 rows sit next to each other on disk, so the database reads a handful of contiguous pages. In Postgres, the same range scan reads the sorted index quickly, but the matching rows may be scattered anywhere in the heap, so it may need to fetch pages from many different places — unless the table happens to already be in roughly that order (for example, because rows were inserted in `order_id` order and never updated much), or `CLUSTER` was run recently. This is one reason Postgres users sometimes run `CLUSTER orders USING orders_pkey` on large, mostly-read tables after a big load, to physically group rows the way MySQL does automatically.

### 4. Composite (multi-column) indexes and the left-prefix rule

A **composite index** (also called a multi-column index) indexes more than one column together, in a fixed order.

```sql
CREATE INDEX idx_emp_dept_salary ON employees(dept_id, salary);
```

Think of this as sorting by `dept_id` first, and within each `dept_id`, sorting by `salary`. It is like a phone book sorted by last name, then first name. You can jump straight to "Rao", and within "Rao" jump straight to "Aditi". But you cannot jump straight to all people named "Aditi" across every last name — you would have to scan the whole book.

This is the **left-prefix rule**: a composite index on `(a, b)` can be used for:
- `WHERE a = ?` (uses only the first column — still helpful)
- `WHERE a = ? AND b = ?` (uses both columns — most helpful)
- `WHERE a = ? AND b > ?` (range on the second column, after an equality on the first — still helpful)
- `ORDER BY a, b` (matches the sort order directly)

But it is **not** used for:
- `WHERE b = ?` alone — because the index is sorted by `a` first. Postgres cannot skip to "all rows where `b = 5`" without scanning through every `a` value.

```sql
-- Uses idx_emp_dept_salary fully
SELECT * FROM employees WHERE dept_id = 3 AND salary > 60000;

-- Uses idx_emp_dept_salary partially (only the dept_id part)
SELECT * FROM employees WHERE dept_id = 3;

-- Cannot use idx_emp_dept_salary at all
SELECT * FROM employees WHERE salary > 60000;
```

**Column order matters.** If most queries filter by `salary` alone, put `salary` first, or create a separate index on `salary`. Deciding column order based on real query patterns is a very common interview question — always ask "which column is filtered alone most often?" before answering.

A useful rule of thumb for ordering composite index columns: put **equality** columns before **range** columns. `WHERE dept_id = 3 AND salary > 60000` works better with `(dept_id, salary)` than `(salary, dept_id)`, because the equality column narrows the search first, and the range scan then happens within that narrowed slice.

### 5. Covering index and index-only scan

A **covering index** is an index that contains every column a query needs, so Postgres never has to open the actual table to get more data.

```sql
CREATE INDEX idx_emp_covering ON employees(dept_id, salary) INCLUDE (emp_name);

SELECT emp_name FROM employees WHERE dept_id = 3 AND salary > 60000;
```

Here, `dept_id` and `salary` are used to filter and sort, and `emp_name` is stored alongside them (using Postgres's `INCLUDE` clause) purely so it can be returned, without a lookup on the table. When every column a query needs is available directly from the index, Postgres can perform an **index-only scan**: it reads only the index, and skips the heap (table) entirely.

Check this in `EXPLAIN ANALYZE` — look for the plan node named `Index Only Scan`, versus a plain `Index Scan` (which still visits the table) or `Seq Scan`.

One catch in Postgres: index-only scans still need to check row **visibility** (whether a row is visible to your transaction, part of MVCC — see Chapter 11). Postgres keeps a **visibility map** to speed this check up, but if a table has many recently changed rows, Postgres may still need to check the heap, weakening the benefit. Running `VACUUM` updates the visibility map.

### 6. Unique and partial indexes

A **unique index** enforces that no two rows share the same value (or combination of values) in the indexed column(s). Postgres automatically creates a unique index whenever you define a `PRIMARY KEY` or a `UNIQUE` constraint.

```sql
CREATE UNIQUE INDEX idx_customer_email ON customers(email);
```

A **partial index** only indexes rows that match a condition. This is useful when queries almost always filter on the same condition, and the rest of the table is irrelevant to that query.

```sql
CREATE INDEX idx_orders_pending ON orders(order_date) WHERE status = 'pending';
```

If most queries look like `WHERE status = 'pending' ORDER BY order_date`, this partial index is much smaller than a full index on all orders, because it skips `'shipped'`, `'delivered'`, and `'cancelled'` rows entirely. Smaller index means faster scans and less disk space.

### 7. A note on hash indexes

Postgres also supports a **hash index** (`CREATE INDEX ... USING HASH`). It is only useful for plain equality checks (`WHERE x = ?`), never for range queries or sorting, because a hash function scrambles order on purpose. In practice, B-tree indexes are used almost everywhere, because they handle both equality and range queries well. Mention hash indexes briefly if asked, but do not over-invest here.

### 8. Index selectivity

**Selectivity** means how many distinct values a column has, compared to the total row count. A column with many distinct values (like `emp_id` or `email`) is **highly selective**. A column with few distinct values (like `status`, with only 3–4 possible values) has **low selectivity**.

If a `status` column has values `'pending'`, `'shipped'`, `'delivered'`, `'cancelled'`, and 40% of all rows are `'delivered'`, an index on `status` does not help much for `WHERE status = 'delivered'`. The query still has to fetch 40% of the table. At that point, Postgres's query planner may decide a **sequential scan is actually cheaper** than jumping around the index and then fetching each matching row from the table one at a time (random I/O is more expensive than sequential I/O).

This is why "my index exists, but Postgres is not using it" is often not a bug — it is the planner making a sensible cost-based choice. Always check selectivity before creating an index on a low-variety column.

You can check selectivity yourself with a quick query before deciding whether an index is worth it:

```sql
SELECT status, COUNT(*), COUNT(*) * 100.0 / SUM(COUNT(*)) OVER () AS pct
FROM orders
GROUP BY status;
```

If one value like `'delivered'` covers 40% of rows, an index on `status` alone will rarely be chosen by the planner for `WHERE status = 'delivered'`. But the same index can still help a lot for a rare value, like `'cancelled'` at 2% of rows, because fetching 2% of the table through an index is far cheaper than scanning all of it. This is also why a **partial index** (Key Concept 6) on just the rare status values is often a better design than a plain index on the whole column — it stays small and useful, instead of being large and half-ignored.

### 9. The cost of indexes

Indexes are not free. Every index adds:
- **Slower writes**: every `INSERT`, `UPDATE`, or `DELETE` must also update every index on that table, not just the table itself. A table with five indexes does up to six pieces of write work per row change.
- **More disk space**: each index is its own stored structure, sometimes nearly as large as the table itself.
- **More vacuum work**: Postgres uses MVCC (see Chapter 11), so updates create new row versions. Indexes need cleanup too, which adds to `VACUUM` overhead.

The real interview point: indexing is always a **trade-off between read speed and write speed**. Say this out loud. Do not add an index just because a query is slow — check if that column is actually filtered often, and if the table is read-heavy or write-heavy.

## The Questions They Ask

**Q1: What is an index, and how does it work internally?**
An index is a separate, sorted structure that lets the database find rows without scanning the whole table. Postgres's default index type is a B-tree: a balanced, sorted tree structure that gives log-time lookups for both equality and range queries. Explain the sorted-and-balanced idea, then mention that it trades write speed and disk space for read speed.
*Follow-up: "Why not just index every column?"* Because every index slows down writes and uses disk space. Only index columns that are actually filtered, joined, or sorted on frequently.

**Q2: In a composite index on `(a, b)`, does it help a query filtering only on `b`?**
No. This is the left-prefix rule. A composite index is sorted by the first column first, so it can only be used starting from the first column. A query on `b` alone would need a separate index on `b`, or `b` would need to be the leading column of some index.
*Follow-up: "How do you decide column order?"* Put the column used for equality checks first, and put the column with the highest selectivity for standalone queries first if it is queried alone often. Look at real query patterns before deciding.

**Q3: Why is my index not being used, even though I created it?**
Several reasons, in order of likelihood:
1. **Low selectivity** — the column has few distinct values, so a sequential scan is cheaper than random index lookups.
2. **Function or type mismatch** — `WHERE UPPER(emp_name) = 'RAO'` cannot use a plain index on `emp_name`, because the stored value and the compared value differ. You would need a functional index: `CREATE INDEX ON employees(UPPER(emp_name))`.
3. **Implicit type cast** — comparing a text column to a number, or similar mismatches, can block index use.
4. **Leading wildcard in `LIKE`** — `WHERE emp_name LIKE '%Rao'` cannot use a standard B-tree index, because the unknown prefix means Postgres cannot start from a fixed point in the sorted order. `LIKE 'Rao%'` (no leading wildcard) can use it.
5. **Stale statistics** — the query planner uses stored statistics about data distribution to estimate costs. If a table changed a lot and `ANALYZE` has not run recently, the planner may make a bad choice. Running `ANALYZE table_name` refreshes these statistics.
6. **Small table** — for a tiny table, a sequential scan is genuinely faster than the overhead of using an index. This is expected, not a bug.

Always confirm with `EXPLAIN ANALYZE` before guessing — it shows exactly which plan Postgres chose, and the real vs. estimated row counts (see Chapter 7).

**Q4: What is the difference between a clustered and a non-clustered index?**
A clustered index stores the actual table rows in the physical order of the index. There can be only one per table. A non-clustered (secondary) index is a separate structure with pointers back to the row's real location. MySQL's InnoDB always clusters the table on the primary key. Postgres never automatically clusters — every index, including the primary key's, is a secondary structure pointing to a heap row using a TID. State this difference clearly if asked, since many candidates assume all databases behave like MySQL.

**Q5: What is a covering index, and when would you use one?**
A covering index includes every column a query needs, either as part of the indexed key or via `INCLUDE`. This lets Postgres do an index-only scan, skipping the table entirely. Use it for frequent, performance-critical read queries where the extra index size is worth avoiding the heap fetch. Mention the visibility-map caveat if pushed further: Postgres may still touch the heap if the visibility map is not up to date, so `VACUUM` matters.

**Q6: How would you decide whether to add an index to a table?**
Say the trade-off out loud: check how often the column appears in `WHERE`, `JOIN`, or `ORDER BY` clauses; check the column's selectivity; check whether the table is read-heavy or write-heavy; check current index count (too many indexes already slows writes a lot). Then confirm the choice using `EXPLAIN ANALYZE` before and after.

## Rapid-Fire

- **What is an index?** A sorted structure that lets the database find rows without scanning the whole table.
- **Default Postgres index type?** B-tree — sorted, balanced, gives log-time lookup, supports range queries.
- **Does a composite index on `(a, b)` help `WHERE b = 5`?** No — left-prefix rule. It helps `WHERE a = ...` and `WHERE a = ... AND b = ...`.
- **Clustered index in Postgres?** Not automatic — Postgres tables are heaps; `CLUSTER` can do it once, manually.
- **Clustered index in MySQL InnoDB?** Always — the table itself is stored sorted by primary key.
- **What is an index-only scan?** A scan that reads only the index, skipping the table, because the index has every needed column.
- **What is a covering index?** An index that includes all columns a query needs, enabling an index-only scan.
- **What is a partial index?** An index built on only the rows matching a condition, smaller and faster for that filtered query.
- **What is selectivity?** The fraction of distinct values in a column; high selectivity (many distinct values) favors index use.
- **Why might Postgres ignore my index?** Low selectivity, function/type mismatch, leading wildcard `LIKE`, stale statistics, or a small table.
- **Main cost of an index?** Slower writes (every index needs updating) and extra disk space.
- **What is a hash index used for?** Only equality lookups; never range queries or sorting.
- **How to check if an index is used?** Run `EXPLAIN ANALYZE` on the query and read the plan.

## Common Traps & Mistakes

- **Assuming more indexes always help.** Every extra index slows down every `INSERT`/`UPDATE`/`DELETE`. Interviewers want to hear the write-cost side, not just the read-speed side.
- **Forgetting the left-prefix rule.** A common mistake is creating `(a, b)` and expecting it to speed up queries that filter on `b` only. It will not.
- **Assuming Postgres behaves like MySQL's clustered table.** Saying "the primary key index stores the row" is true for InnoDB, but false for Postgres, where the primary key index is just another secondary structure pointing to a heap row.
- **Adding an index on a low-selectivity column and being confused when it is unused.** An index on `status` with only 3 values rarely helps, because the planner correctly judges a sequential scan as cheaper.
- **Not running `ANALYZE` after bulk loads.** Stale statistics can make the planner pick a bad plan even when the right index exists.
- **Wrapping indexed columns in functions in `WHERE` clauses without a matching functional index.** `WHERE LOWER(emp_name) = 'aditi'` will not use a plain index on `emp_name`.
- **Using a leading wildcard in `LIKE` and expecting index use.** `LIKE '%Rao'` cannot use a standard B-tree index; only `LIKE 'Rao%'` can.
- **Forgetting `INCLUDE` for covering indexes.** Some candidates add extra columns into the sort key itself, e.g. `(dept_id, salary, emp_name)`, when `emp_name` is only ever returned, never filtered or sorted on. Using `INCLUDE (emp_name)` keeps the index smaller and the key structure clean.
- **Ignoring `EXPLAIN ANALYZE` and guessing.** Interviewers often ask "how would you confirm that", and the correct answer is always to check the actual query plan, not assume.
