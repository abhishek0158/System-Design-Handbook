# Chapter 10 — Food Delivery

A food delivery app (like DoorDash or Zomato) connects three people: a customer, a restaurant,
and a courier. It is a common design question because it tests how you link three parties and
track two related but separate lifecycles at once.

## Requirements & Clarifying Questions

**Key features to support:**
- Restaurants list menu items with a name and a price.
- A customer places an order from one restaurant, with one or more menu items.
- The order moves through states: `placed` → `accepted` → `preparing` → `ready`.
- A courier is assigned to carry the order, and the delivery moves through its own states:
  `assigned` → `picked_up` → `delivered`.

**Clarifying questions to ask:**
- Can one order include items from more than one restaurant? (We assume no. One order belongs
  to exactly one restaurant.)
- Who assigns the courier — the system, or a dispatcher? (We assume automatic: the system offers
  the delivery to nearby couriers, and the first one to accept gets it.)
- Can a courier carry more than one order at once (batching)? (We assume no, for simplicity.)
- Read/write pattern: menu pages are read very often. Orders and deliveries are written far
  less often, but each write must be correct and never lost.

**Rough scale:** Assume 50,000 restaurants, 2 million menu items, and 500,000 orders a day at
peak. Menu reads outnumber order writes by a large factor.

## The Schema

```sql
CREATE TABLE restaurants (
    id          BIGSERIAL PRIMARY KEY,
    name        TEXT NOT NULL,
    is_active   BOOLEAN NOT NULL DEFAULT true,
    created_at  TIMESTAMPTZ NOT NULL DEFAULT now()
    -- ...more columns: address, cuisine_type, rating, etc.
);
-- One row per restaurant.

CREATE TABLE menu_items (
    id              BIGSERIAL PRIMARY KEY,
    restaurant_id   BIGINT NOT NULL REFERENCES restaurants(id),
    name            TEXT NOT NULL,
    price_minor     BIGINT NOT NULL,   -- CURRENT price, in minor units (cents)
    is_available    BOOLEAN NOT NULL DEFAULT true
);
-- One row per dish a restaurant sells. price_minor is the current price.
-- It changes over time (the restaurant edits its menu).
CREATE INDEX idx_menu_items_restaurant ON menu_items(restaurant_id);

CREATE TABLE customers (
    id          BIGSERIAL PRIMARY KEY,
    email       TEXT NOT NULL UNIQUE,
    name        TEXT NOT NULL
);
-- One row per customer.

CREATE TABLE couriers (
    id          BIGSERIAL PRIMARY KEY,
    name        TEXT NOT NULL,
    is_online   BOOLEAN NOT NULL DEFAULT false
    -- ...more columns: vehicle_type, current_location, etc.
);
-- One row per courier (delivery rider).

CREATE TABLE orders (
    id             BIGSERIAL PRIMARY KEY,
    customer_id    BIGINT NOT NULL REFERENCES customers(id),
    restaurant_id  BIGINT NOT NULL REFERENCES restaurants(id),
    status         TEXT NOT NULL DEFAULT 'placed',  -- placed, accepted, preparing, ready, cancelled
    total_minor    BIGINT NOT NULL,                  -- order total, in minor units (cents)
    created_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at     TIMESTAMPTZ NOT NULL DEFAULT now()
);
-- One row per order. Links a customer to one restaurant. status tracks the
-- restaurant-side lifecycle only (nothing about the delivery).
CREATE INDEX idx_orders_customer_created ON orders(customer_id, created_at DESC);
CREATE INDEX idx_orders_restaurant_status ON orders(restaurant_id, status);

CREATE TABLE order_items (
    id               BIGSERIAL PRIMARY KEY,
    order_id         BIGINT NOT NULL REFERENCES orders(id),
    menu_item_id     BIGINT NOT NULL REFERENCES menu_items(id),
    quantity         INT NOT NULL CHECK (quantity > 0),
    unit_price_minor BIGINT NOT NULL   -- price AT THE TIME OF ORDER, copied, not live
);
-- One row per dish line inside an order. Junction table between orders and
-- menu_items: one order has many items, one menu item appears in many orders.
CREATE INDEX idx_order_items_order ON order_items(order_id);

CREATE TABLE deliveries (
    id           BIGSERIAL PRIMARY KEY,
    order_id     BIGINT NOT NULL UNIQUE REFERENCES orders(id),
    courier_id   BIGINT REFERENCES couriers(id),   -- NULL until a courier is assigned
    status       TEXT NOT NULL DEFAULT 'unassigned', -- unassigned, assigned, picked_up, delivered
    assigned_at  TIMESTAMPTZ,
    updated_at   TIMESTAMPTZ NOT NULL DEFAULT now()
);
-- One row per order. Tracks the courier-side lifecycle only, separate from
-- the order's own status. order_id is UNIQUE: one order has at most one
-- active delivery (no batching, per our assumption).
CREATE INDEX idx_deliveries_courier ON deliveries(courier_id, status);
```

## Key Design Decisions — the Reasoning

**1. Three parties, linked through two tables, not one giant table.**
A food delivery order has three sides: the customer who pays, the restaurant that cooks, and
the courier who carries the food. It is tempting to put `customer_id`, `restaurant_id`, and
`courier_id` all on one `orders` row. But the courier is decided later than the restaurant, and
a courier's job (pick up, deliver) is different work from a restaurant's job (accept, cook). So
we split it: `orders` links customer and restaurant — what the restaurant needs to act on.
`deliveries` links the order to a courier — what the courier app needs to act on. One order has
one row in each table. This split lets each side move independently, which leads to the next
decision.

**2. Two separate status lifecycles, not one.**
The order has a status: `placed` → `accepted` → `preparing` → `ready`. This is the
**restaurant's** view: has the kitchen seen it, started cooking, finished cooking? The delivery
has its own status: `assigned` → `picked_up` → `delivered`. This is the **courier's** view: has
someone taken the job, collected the food, dropped it off?

These two are not the same thing, and they do not move in lockstep. An order can be `ready`
(food cooked, waiting on the counter) while its delivery is still `assigned` (courier is five
minutes away, has not arrived). If we crammed both into one `status` column — say `placed,
accepted, preparing, ready, picked_up, delivered` — we would hit two problems: we could not
represent "ready but not yet picked up" and "preparing but courier already assigned" at the same
time, since one column holds only one value; and the restaurant app and courier app would both
write to the same column, raising the chance that one side overwrites the other's update.

The fix: two tables, two status columns, each owned by a different actor. The restaurant updates
`orders.status`. The courier updates `deliveries.status`. `deliveries.order_id` links them. A
"track my food" screen for the customer just reads both and shows a combined picture — for
example, "preparing" + "assigned" means "your food is being cooked, a courier is on the way."

**3. Copy the menu price onto `order_items` at order time.**
`menu_items.price_minor` is the *current* price. A restaurant can change a price any time — for
example, a lunch discount that ends at 3 PM. If `order_items` only stored `menu_item_id` and
looked up the price live, an order placed at 2:55 PM would show a different total if viewed
again at 3:05 PM. That is wrong: a receipt must never change after the order is placed, and a
refund must match what the customer actually paid.

The fix: copy the price into `order_items.unit_price_minor` when the order is placed. This is
the "price at the time of the event" pattern (Chapter 2). `orders.total_minor` is computed from
these copied prices, never from a live join to `menu_items`.

**4. Assigning a courier must be an atomic claim.**
When an order is `ready`, the system offers the delivery to nearby free couriers — often more
than one at the same time. If two couriers try to accept the same delivery, only one should win.
This is the same core problem as a ride-hailing app assigning a driver to a ride request: many
candidates, one job, must not double-assign.

A naive approach — read `deliveries.status`, check it is `unassigned`, then write `courier_id`
— has a race condition. Two couriers can both read "unassigned" before either writes, and both
then write themselves in as the courier. The fix is a single atomic statement:

```sql
UPDATE deliveries
SET courier_id = :courier_id, status = 'assigned', assigned_at = now()
WHERE order_id = :order_id AND status = 'unassigned';
```

The database checks `status = 'unassigned'` and updates it as one atomic step — no other
transaction can slip in between the check and the write. If the statement affects 0 rows, this
courier lost the race and the app tells them the job is taken. If it affects 1 row, this courier
won, and no one else can win the same row afterward. This "conditional `UPDATE` that checks and
claims in one step" pattern is worth naming out loud — it is the same trick used for seat
booking (Chapter 6) and driver assignment in ride-hailing apps.

## Scaling It

**Orders and deliveries grow forever.** Every order stays in the table, and it is rarely read
again once delivered (only for support or receipts). After a few years, this can be billions of
rows. Partition `orders` (and `order_items`, `deliveries`) by `created_at`, monthly or yearly.
Old partitions can move to cheap cold storage without slowing down queries on active orders.

**Menu reads are the hottest, most cache-friendly path.** Customers browse many restaurants
before placing one order, so `restaurants` and `menu_items` get far more reads than `orders` get
writes. Menu items also change rarely. This is a great fit for a cache (like Redis): cache a
restaurant's full menu under one key, and invalidate it only when the restaurant edits a price
or availability. Add read replicas too, so menu browsing does not slow down order writes.

**Courier assignment is a write hotspot at peak hours.** During lunch and dinner rush, many
`deliveries` rows get claimed within seconds of becoming `unassigned`. The atomic conditional
`UPDATE` already avoids double-assignment, but at very high volume, consider moving the "offer
to nearby couriers" matching logic into an in-memory or queue-based system, using the database
only to record the final claim.

## Interview Tips & Common Mistakes

- **Do not merge the order status and delivery status into one column.** This is the signature
  mistake in this design. Say clearly: "the restaurant's lifecycle and the courier's lifecycle
  are different concerns, owned by different actors, so they get different tables."
- **Do not forget `unit_price_minor` on `order_items`.** Live-joining to `menu_items.price_minor`
  makes old receipts change when the menu price changes. Always copy the price at order time.
- **Do not use `float` for money.** Use integer minor units (cents) or `NUMERIC`.
- **Say "atomic claim" out loud when assigning the courier.** A plain read-then-write is a race
  condition. Mention the conditional `UPDATE` even if not asked — it shows you think about
  correctness under load.
- **Keep `deliveries.order_id` UNIQUE** under the no-batching assumption. If asked about batching
  multiple orders per courier trip, say this constraint would need to change.
- **State your single-restaurant-per-order assumption out loud.** It simplifies the schema, and
  inviting the interviewer to correct you shows structured thinking.
