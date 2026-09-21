# Chapter 3 — E-commerce / Orders

An online store needs to sell products, take orders, and track stock. This is a very common
design question, because it tests many core skills at once: history tracking, money handling,
and avoiding overselling.

## Requirements & Clarifying Questions

Before designing, ask questions. This shows the interviewer how you think.

**Key features to support:**
- Customers browse products and place orders.
- An order can have many products, each with a quantity.
- The store tracks stock (how many units of each product are left).
- An order moves through states: `pending` → `paid` → `shipped` → `delivered` (or `cancelled`).

**Clarifying questions to ask:**
- Can one order have items from different sellers, or is this a single-seller store? (We
  assume single-seller here, to keep the schema focused.)
- Do we need to support partial shipments (one order, multiple packages)? (We assume no, for
  simplicity. Note it as a follow-up if asked.)
- Is payment handled by this service, or by an external payment gateway? (We assume external.
  We just store the order status.)
- What is the read/write pattern? Product pages are read very often (high read traffic).
  Orders are written less often, but must never be lost or corrupted.

**Rough scale:** Assume 1 million products, 100,000 orders a day, and product pages getting
10x more reads than orders get writes.

## The Schema

```sql
CREATE TABLE customers (
    id          BIGSERIAL PRIMARY KEY,
    email       TEXT NOT NULL UNIQUE,
    name        TEXT NOT NULL,
    created_at  TIMESTAMPTZ NOT NULL DEFAULT now()
);
-- One row per customer. Email is unique, used for login.

CREATE TABLE products (
    id          BIGSERIAL PRIMARY KEY,
    sku         TEXT NOT NULL UNIQUE,
    name        TEXT NOT NULL,
    price_cents BIGINT NOT NULL,           -- current price, in minor units (cents)
    is_active   BOOLEAN NOT NULL DEFAULT true,
    created_at  TIMESTAMPTZ NOT NULL DEFAULT now()
    -- ...more columns: description, category_id, etc.
);
-- One row per product. price_cents is the CURRENT price. It changes over time.

CREATE TABLE inventory (
    product_id          BIGINT PRIMARY KEY REFERENCES products(id),
    quantity_on_hand     INT NOT NULL DEFAULT 0,
    quantity_reserved    INT NOT NULL DEFAULT 0,
    updated_at           TIMESTAMPTZ NOT NULL DEFAULT now(),
    CONSTRAINT chk_stock_non_negative
        CHECK (quantity_on_hand >= 0 AND quantity_reserved >= 0)
);
-- One row per product. Tracks how many units are in stock and how many are reserved
-- for orders that are not yet confirmed.

CREATE TABLE orders (
    id           BIGSERIAL PRIMARY KEY,
    customer_id  BIGINT NOT NULL REFERENCES customers(id),
    status       TEXT NOT NULL DEFAULT 'pending',  -- pending, paid, shipped, delivered, cancelled
    total_cents  BIGINT NOT NULL,
    created_at   TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at   TIMESTAMPTZ NOT NULL DEFAULT now()
);
-- One row per order. status holds the CURRENT state only, for fast reads.
CREATE INDEX idx_orders_customer_created ON orders(customer_id, created_at DESC);

CREATE TABLE order_items (
    id               BIGSERIAL PRIMARY KEY,
    order_id         BIGINT NOT NULL REFERENCES orders(id),
    product_id       BIGINT NOT NULL REFERENCES products(id),
    quantity         INT NOT NULL CHECK (quantity > 0),
    unit_price_cents BIGINT NOT NULL   -- price AT THE TIME OF ORDER, copied, not live
);
-- One row per product line inside an order. This is the junction table between
-- orders and products (many-to-many): one order has many items, one product
-- appears in many orders.
CREATE INDEX idx_order_items_order ON order_items(order_id);

CREATE TABLE order_status_history (
    id          BIGSERIAL PRIMARY KEY,
    order_id    BIGINT NOT NULL REFERENCES orders(id),
    status      TEXT NOT NULL,
    changed_at  TIMESTAMPTZ NOT NULL DEFAULT now(),
    note        TEXT
);
-- Append-only log of every status change for an order. Never updated, only inserted.
CREATE INDEX idx_status_history_order ON order_status_history(order_id, changed_at);
```

## Key Design Decisions — the Reasoning

**1. Copy the price onto `order_items` at order time.**
`products.price_cents` is the *current* price. It changes when the store runs a sale or raises
prices. But an order placed last month must show last month's price, forever. If we only stored
`product_id` and looked up the price live, every old order's total would silently change when
the price changes. This is wrong, and it breaks refunds and accounting.

The fix: copy the price into `order_items.unit_price_cents` at the moment the order is placed.
This is the general pattern "price at the time of the event" (see Chapter 2). The trade-off is
small: a little duplicated data, but correctness is worth far more than saving a few bytes.

**2. Status column plus a status history table.**
The `orders.status` column answers "what is the state right now?" fast, with a single indexed
column. But interviewers often ask: "how do you know when an order became `paid`, or why it was
`cancelled`?" A single column cannot answer that, because writing a new status overwrites the
old one.

The fix: also write every status change to `order_status_history`, an append-only table (rows
are only inserted, never changed or deleted). The `orders.status` column is a fast summary.
The history table is the full audit trail. This combination — a status column for fast reads,
plus an append-only history table for the full story — is a pattern you will reuse in almost
every chapter (Chapter 2 calls this out as a recurring theme).

**3. Avoiding overselling stock.**
Two customers can try to buy the last unit of a product at the same time. If both requests read
"1 unit left," then both write, we sell 2 units of something we only had 1 of. This is called a
race condition.

At a high level, there are two common fixes:
- **Atomic decrement:** run `UPDATE inventory SET quantity_on_hand = quantity_on_hand - 1 WHERE
  product_id = ? AND quantity_on_hand >= 1`. The database checks the condition and updates the
  row in one atomic step. If the row count returned is 0, stock ran out; reject the order.
- **Reserve, then confirm:** move units from `quantity_on_hand` into `quantity_reserved` when
  the customer starts checkout. Only release the reservation back to stock if payment fails or
  times out. This avoids holding stock forever for abandoned carts.

Both need a database-level guarantee (a single atomic statement, or a transaction with row
locking) so that "check stock" and "reduce stock" cannot be split across two requests that
interleave. We only note the idea here. Chapter 6 (Booking/Reservation) covers this exact
problem — avoiding double-booking a limited resource — in full depth, because it is the same
core problem as overselling a hotel room or a concert seat.

**4. One order, many items — the many-to-many via a junction table.**
An order needs several different products, and a product is sold in many different orders. We
model this with `order_items` as a junction table (see Chapter 2's reusable patterns): it holds
`order_id` and `product_id` as foreign keys, plus data that belongs to the *pairing*, not to
either side alone — here, `quantity` and `unit_price_cents`. This is the standard way to model
many-to-many relationships in a relational database.

## Scaling It

**Product catalog reads are the hottest path.** Product pages get far more reads than orders
get writes. Add a cache (like Redis) in front of `products` and `inventory` for product detail
pages, and add read replicas for the database so read traffic does not compete with order
writes. Cache invalidation on price or stock change is the tricky part — keep the cache TTL
(time to live) short, or invalidate the specific product key on write.

**Orders grow forever and never shrink.** Every order placed stays in the table. After a few
years, this table can hold billions of rows, and old orders are rarely read (only for support or
legal reasons). The fix: archive old orders. Move orders older than, say, 2 years into a cheaper
`orders_archive` table or cold storage, keyed the same way. You can also partition the `orders`
table by `created_at` (monthly or yearly partitions) so old partitions can be detached and
archived cheaply, without one giant table slowing down every query.

**Inventory is a write hotspot for popular products.** A flash sale on one product means many
transactions try to update the same `inventory` row at once. This can cause lock contention.
Techniques like sharding the counter, or using a queue to serialize decrements, help here — but
that is an advanced topic beyond this chapter.

## Interview Tips & Common Mistakes

- **Do not skip `unit_price_cents`.** Forgetting to copy the price is the single most common
  mistake in this design. Say it out loud even if not asked: "I store the price at order time,
  not a live join to the product."
- **Do not use `float` for money.** Always use integer minor units (cents) or `NUMERIC`. Floats
  cause rounding errors.
- **Do not model status with only a column.** If asked "how do you audit state changes,"
  a status-only design has no answer. Mention the history table before being asked.
- **Do not forget the index on `order_items(order_id)`.** Without it, loading an order's items
  is a slow table scan once the table is large.
- **Mention the overselling race condition even briefly.** Interviewers listen for the words
  "atomic," "transaction," or "row lock." Do not assume a simple `UPDATE` is automatically safe
  without a WHERE condition that checks current stock.
- **Say what you are assuming out loud** (single-seller, no partial shipments, external
  payment gateway). This shows structured thinking and invites the interviewer to correct you
  if the scope is different.
