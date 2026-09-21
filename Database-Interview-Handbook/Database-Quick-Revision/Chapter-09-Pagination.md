# Chapter 9 — Pagination

Every list API needs pagination: show 20 rows at a time, not a million. Interviewers ask this
because a naive design works fine in a demo and falls over in production, on page 5,000.

## The Idea

Pagination means splitting a large result set into small pages. There are two common ways to do
it in SQL.

**1. Offset pagination.** You ask the database to skip a number of rows, then return the next
batch. This gives you page-number style paging ("go to page 6").

```sql
-- Page 6, 20 rows per page (skip the first 100 rows)
SELECT order_id, customer_id, order_date, status, total_amount
FROM orders
ORDER BY order_date DESC, order_id DESC
LIMIT 20 OFFSET 100;
```

`LIMIT` caps how many rows come back. `OFFSET` says how many rows to skip first.

**2. Keyset pagination (also called cursor pagination).** Instead of a page number, you remember
the last row you saw on the previous page. The next query asks: "give me rows that come after
this one, in the same sort order."

```sql
-- Last row seen: order_date = '2026-01-05', order_id = 8341
SELECT order_id, customer_id, order_date, status, total_amount
FROM orders
WHERE (order_date, order_id) < (:last_date, :last_id)
ORDER BY order_date DESC, order_id DESC
LIMIT 20;
```

The `(order_date, order_id)` pair is the **cursor**. It is a snapshot of where you stopped
reading. `order_id` is added as a tie-breaker, because many rows can share the same
`order_date`, and you need a unique, ordered value to know exactly where you are.

## Why It Matters — the Reasoning

The core question is: how does the database do `OFFSET`? It cannot skip rows by magic. It must
first find the rows in sort order, and then throw away the first N of them, one by one, before it
can start returning rows to you.

So `LIMIT 20 OFFSET 100000` does not "cost" 20 rows. It costs 100,020 rows: the database walks the
index (or scans the table) 100,020 rows deep, discards 100,000, and hands you the last 20. If a
user pages deep into search results, or a background job walks a big table page by page, this
cost keeps growing. Page 1 is instant. Page 5,000 is slow. **The cost of an offset query grows
with the page number, not with the page size.**

Keyset pagination avoids this. The `WHERE (order_date, order_id) < (:last_date, :last_id)` clause
is a direct condition on indexed columns. Postgres uses a B-tree index (see Chapter 2) to jump
straight to that value and read the next 20 rows from there. It does not touch the rows before
the cursor at all. **The cost stays flat, no matter how deep you page.** This is the single
biggest reasoning point in this chapter: offset pays for the rows it skips, keyset does not skip
anything, it jumps.

The trade-off: keyset pagination cannot jump to an arbitrary page number. It only supports
"next page" and "previous page" (you can support "previous" by reversing the comparison and sort
order). There is no way to ask keyset for "page 50" directly, because there is no concept of a
page number, only a position. If your UI needs numbered page links (1, 2, 3, ... 50), you need
offset, or some other trick (see Quick Recall). If your UI only needs "load more" or infinite
scroll, keyset is the better fit almost always.

There is a second problem with offset that is easy to miss: **consistency under changing data.**
Offset pagination re-runs the sort and re-counts rows from zero on every page request. If a row
is inserted or deleted between the time a user loads page 1 and page 2, the row positions shift.
The user can see the same row twice (a duplicate), or skip a row entirely (a missed row), even
though nothing looks wrong on screen. This happens because offset has no memory of what the user
already saw. It only knows a row count to skip.

Keyset pagination does not have this problem. The cursor is a real row's values, not a row count.
Even if rows are inserted or deleted elsewhere in the table, the condition `(order_date, order_id)
< (:last_date, :last_id)` still means the same thing: "rows strictly after this exact anchor."
The page stays stable because it is anchored to real data, not to a position that can shift.

## Common Interview Questions

**Q1: What is the difference between `LIMIT/OFFSET` pagination and keyset (cursor) pagination?**
Offset pagination skips N rows and returns the next batch; it supports jumping to any page number
but gets slower as the page number grows. Keyset pagination remembers the last row seen and asks
for rows after it; it is fast at any depth but only supports next/previous, not arbitrary page
numbers.

**Q2: Why is a deep `OFFSET` (like `OFFSET 100000`) slow?**
The database cannot skip rows without first reading them in sort order. `OFFSET 100000` forces it
to read and discard 100,000 rows before it can return the next 20. The work done is proportional
to `OFFSET + LIMIT`, not just `LIMIT`. So the deeper the page, the slower the query, even with an
index on the sort column.

**Q3: You have a table with 50 million rows and need to let users page through it. How do you
design this efficiently?**
Use keyset pagination on an indexed, unique, ordered column (or column pair). Each page request
carries the last row's key values as a cursor. This keeps every page query fast, because the
database jumps to the cursor position using the index instead of scanning from the start. Avoid
offset for anything beyond the first few pages.

**Q4: How do you keep pagination results stable if rows are being inserted or deleted while a
user pages through results?**
Use keyset pagination. It anchors each page on the last row's actual values, not a row count, so
inserts and deletes elsewhere in the table do not shift what "page 2" means. Offset pagination has
no such anchor — it just skips a count of rows, so shifting data causes duplicate or missing rows
across pages.

**Q5: Can keyset pagination jump straight to page 50?**
No. Keyset only knows how to go to "the rows after this cursor" or "the rows before this cursor."
There is no row-count concept, so there is no direct way to compute where page 50 starts without
walking through the pages before it. If you need numbered-page jumping, use offset pagination, or
accept an approximate jump (for example, jump by an indexed value range instead of an exact page).

**Q6: What column, or columns, should you use as the cursor in keyset pagination?**
Use a column, or combination of columns, that is unique and has a strict order, and that already
has an index matching your sort order. A single non-unique column (like `order_date` alone) is
not enough, because ties between rows with the same date make the "next row" ambiguous. Adding a
unique tie-breaker, like the primary key `order_id`, fixes this: `(order_date, order_id)` is
always unique and always ordered, so the cursor always points to exactly one position.

## Quick Recall

- Offset pagination: `LIMIT 20 OFFSET 100`. Simple, supports page numbers, gets slower as the page
  number grows.
- Keyset pagination: `WHERE (order_date, order_id) < (:last_date, :last_id) ORDER BY order_date
  DESC, order_id DESC LIMIT 20`. Fast at any depth, but only next/previous, no jumping to an
  arbitrary page.
- Offset cost grows with page depth, because the database must read and discard every skipped
  row. Keyset cost stays flat, because it jumps straight to the cursor using the index.
- Offset can show duplicate or missing rows if data changes between page loads. Keyset stays
  stable because it anchors on a real row, not a row count.
- **Biggest gotcha:** always pick a unique, ordered cursor (add the primary key as a tie-breaker).
  A cursor on a non-unique column alone can skip or repeat rows that share the same value.
