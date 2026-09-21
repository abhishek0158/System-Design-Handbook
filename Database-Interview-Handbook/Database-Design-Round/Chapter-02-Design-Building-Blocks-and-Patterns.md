# Chapter 2 — Design Building Blocks & Reusable Patterns

Every schema design, no matter the system, reuses the same small set of pieces. Learn these once, and every case study in this book becomes easier.

## Relationships

A relationship says how two tables connect. There are three kinds.

**One-to-one.** One row in table A matches exactly one row in table B. Use this to split a table when one part is optional or rarely read (for example, a big profile blob you don't need on every query).

```sql
CREATE TABLE users (
  id BIGINT PRIMARY KEY,
  email TEXT NOT NULL UNIQUE
);

CREATE TABLE user_profiles (
  user_id BIGINT PRIMARY KEY REFERENCES users(id),
  bio TEXT,
  avatar_url TEXT
);
```

The `user_id` in `user_profiles` is both the primary key and a foreign key. This forces exactly one profile row per user.

**One-to-many.** One row in table A can match many rows in table B, but each row in B belongs to only one row in A. This is the most common relationship. The foreign key always goes on the "many" side.

```sql
CREATE TABLE customers (
  id BIGINT PRIMARY KEY,
  name TEXT NOT NULL
);

CREATE TABLE orders (
  id BIGINT PRIMARY KEY,
  customer_id BIGINT NOT NULL REFERENCES customers(id),
  created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

One customer has many orders. Each order has one customer. `orders.customer_id` is the foreign key. A **foreign key** is a column that stores the primary key of a row in another table, so the database can check the link is valid.

**Many-to-many.** Rows on both sides can match many rows on the other side. Model this with a **junction table** (also called a join table): a third table whose only job is to hold pairs of foreign keys.

```sql
CREATE TABLE students (
  id BIGINT PRIMARY KEY,
  name TEXT NOT NULL
);

CREATE TABLE courses (
  id BIGINT PRIMARY KEY,
  title TEXT NOT NULL
);

CREATE TABLE enrollments (
  student_id BIGINT NOT NULL REFERENCES students(id),
  course_id  BIGINT NOT NULL REFERENCES courses(id),
  enrolled_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  PRIMARY KEY (student_id, course_id)
);
```

A student takes many courses, and a course has many students. `enrollments` holds one row per pair. The primary key is the pair itself, so the same student can't enroll in the same course twice. You will use this pattern constantly: tags on posts, roles for users, products in orders.

## Keys

A **key** is a column (or set of columns) that identifies a row, or links it to another row.

**Primary key.** The column that uniquely identifies a row in its own table. Every table should have one. No two rows can share it, and it can never be null.

**Foreign key.** A column that points to a primary key in another table. It is how relationships are enforced. The database rejects an insert if the referenced row does not exist (unless you turn that check off).

**Surrogate vs natural key.** A **natural key** is a value from the real world that could identify a row, like an email address or a country code. A **surrogate key** is a made-up id with no business meaning, like an auto-incrementing number or a UUID.

Prefer a surrogate key as the primary key, almost always. Reasons:
- Natural keys change. A user's email changes; a product's SKU gets reissued. If other tables have foreign keys pointing to that value, every one of them breaks.
- Natural keys are often bigger and slower to index (a long email string vs an 8-byte bigint).
- Some natural keys are not as unique as they look (two people can share a name).

Still add a `UNIQUE` constraint on the natural key if it must stay unique (like email). You get uniqueness without using it as the primary key.

```sql
CREATE TABLE users (
  id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,  -- surrogate key
  email TEXT NOT NULL UNIQUE                            -- natural key, unique but not primary
);
```

**Composite key.** A primary key made of more than one column. Common in junction tables, where the pair of foreign keys is naturally unique together, as in `enrollments (student_id, course_id)` above.

## Reusable Patterns

These patterns show up in almost every design in this book. Learn the reasoning once.

**1. Status as a column, or a status-history table.**
Most rows have a state, like an order's status: `pending`, `paid`, `shipped`. The simple option is one column plus `updated_at`:

```sql
ALTER TABLE orders ADD COLUMN status TEXT NOT NULL DEFAULT 'pending';
ALTER TABLE orders ADD COLUMN updated_at TIMESTAMPTZ NOT NULL DEFAULT now();
```

This tells you the current state, but not the path it took to get there. Add a separate append-only history table when you need that path — for audits, disputes, support tickets, or "how long did each step take":

```sql
CREATE TABLE order_status_history (
  id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  order_id BIGINT NOT NULL REFERENCES orders(id),
  status TEXT NOT NULL,
  changed_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

Rule of thumb: keep just the column if nobody asks "what happened before this?" Add history the moment that question matters, especially where money or compliance is involved (see Chapter 8).

**2. Soft delete.**
A **soft delete** marks a row as gone without actually removing it, usually with `is_deleted BOOLEAN` or `deleted_at TIMESTAMPTZ`.

```sql
ALTER TABLE products ADD COLUMN deleted_at TIMESTAMPTZ;
-- a live row has deleted_at IS NULL
```

Why: you keep the data for audits, undo, and any foreign keys that still point to it (a past order line still needs to show the product name). The trade-off: every query must now remember to filter `WHERE deleted_at IS NULL`, or you will show "deleted" rows by mistake. Unique constraints get trickier too — two "deleted" rows with the same email will clash unless you write the constraint carefully (for example, a partial unique index that only applies when `deleted_at IS NULL`).

**3. Audit columns.**
Almost every table should track who created or changed a row, and when:

```sql
created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
created_by BIGINT REFERENCES users(id),
updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
```

This costs almost nothing and answers "who did this and when" — a question that comes up constantly in support and debugging.

**4. Lookup tables vs enum types.**
When a column can only take a few fixed values (like order status, or user role), you have two choices.

A **lookup table** (also called a reference table) is a small table listing the allowed values, with other tables pointing to it by foreign key:

```sql
CREATE TABLE order_statuses (
  code TEXT PRIMARY KEY   -- e.g. 'pending', 'paid', 'shipped'
);

ALTER TABLE orders ADD COLUMN status TEXT NOT NULL REFERENCES order_statuses(code);
```

An **enum type** is a type built into the database that only accepts a fixed list of values:

```sql
CREATE TYPE order_status AS ENUM ('pending', 'paid', 'shipped', 'cancelled');
ALTER TABLE orders ADD COLUMN status order_status NOT NULL DEFAULT 'pending';
```

Prefer a lookup table when the list of values changes often, or needs extra fields (like a label to show users, or a sort order). Prefer an enum type when the list is small, rarely changes, and you want the database to reject bad values cheaply. Plain `TEXT` with a `CHECK` constraint is a fine middle ground too — easy to read, easy to change.

**5. Money: integer minor units or NUMERIC, never float.**
`FLOAT` and `DOUBLE` store numbers in binary, so amounts like 0.10 cannot be stored exactly. Round tiny errors keep piling up across millions of rows, and money must be exact.

Two safe choices:
- Store the amount as an integer count of the smallest unit (cents for USD): `amount_cents BIGINT`. 999 means $9.99.
- Store it as `NUMERIC(12, 2)`, which is exact decimal math, not binary.

```sql
CREATE TABLE payments (
  id BIGINT PRIMARY KEY,
  amount_cents BIGINT NOT NULL,     -- $19.99 -> 1999
  currency TEXT NOT NULL DEFAULT 'USD'
);
```

Integer cents are simpler to add and compare. `NUMERIC` reads more naturally if you need fractional cents (like a per-unit price with 4 decimal places). Either is fine — `FLOAT` never is. See Chapter 8 for a full ledger design.

**6. Timestamps in UTC.**
Always store timestamps in UTC (Coordinated Universal Time, the single global reference time), using a type that keeps the time zone information, like Postgres's `TIMESTAMPTZ`. Convert to the user's local time only in the application or the UI. If you store local time without a zone, you cannot safely compare timestamps from users in different countries, and daylight-saving changes will corrupt your data.

**7. Copy the value at the time of the event.**
Prices, exchange rates, and discounts change over time. If an order line only stores a `product_id` and looks up the current price to show a past order, the customer's receipt changes every time the price changes. Instead, copy the price onto the row at the moment of the event:

```sql
CREATE TABLE order_items (
  id BIGINT PRIMARY KEY,
  order_id BIGINT NOT NULL REFERENCES orders(id),
  product_id BIGINT NOT NULL REFERENCES products(id),
  unit_price_cents BIGINT NOT NULL,   -- price AT THE TIME of purchase, not today's price
  quantity INT NOT NULL
);
```

This is one of the most tested ideas in design interviews: never trust a "live" value for something that happened in the past. Copy it, freeze it.

**8. Hierarchical data — a quick preview.**
Some data is naturally tree-shaped: categories inside categories, comments replying to comments, an org chart. The simplest storage is the **adjacency list**: each row has a `parent_id` pointing to its own table.

```sql
CREATE TABLE categories (
  id BIGINT PRIMARY KEY,
  name TEXT NOT NULL,
  parent_id BIGINT REFERENCES categories(id)   -- NULL means top-level
);
```

This is easy to write to, but reading a whole subtree takes a recursive query. Chapter 14 covers this in full, along with other ways to store trees.

## Quick Recall

- **One-to-one**: foreign key is also the primary key on the child table.
- **One-to-many**: foreign key lives on the "many" side.
- **Many-to-many**: junction table with a composite primary key of both foreign keys.
- **Primary key**: unique row id. **Foreign key**: points to another table's primary key.
- Prefer a **surrogate key** (auto id) over a natural key; add `UNIQUE` on the natural value if needed.
- **Composite key**: primary key made of two or more columns, common in junction tables.
- **Status**: a column + `updated_at` for current state; add a history table when you must know the past path.
- **Soft delete**: `deleted_at` keeps data but every query must filter it out.
- **Audit columns**: `created_at`, `created_by`, `updated_at` — cheap and always useful.
- **Lookup table vs enum type**: table for flexible/growing lists, enum for small fixed lists.
- **Money**: integer minor units or `NUMERIC`, never `FLOAT`.
- **Timestamps**: always UTC, with time zone-aware types.
- **Freeze the value**: copy price/amount onto the row at event time, don't trust "live" lookups.
- **Hierarchy**: `parent_id` adjacency list is the simple default (full detail in Chapter 14).
