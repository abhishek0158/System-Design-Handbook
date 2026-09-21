# Chapter 9 — Ride-Hailing (Uber-style)

Design the backend for a ride-hailing app: a rider requests a trip, a nearby driver is
assigned, and the trip runs to completion. This question is popular because it mixes a clear
state machine, a location search problem, and a "do not double-book a driver" concurrency
problem, all in one small schema.

## Requirements & Clarifying Questions

Key features:
- A rider requests a trip from a pickup point to a drop point.
- The system finds a nearby available driver and assigns the trip to them.
- The trip moves through clear stages: requested, assigned, ongoing, completed (or cancelled).
- The fare (the price of the trip) is calculated and stored.

Questions to ask the interviewer:
- How fresh must driver location be? Sub-second tracking, or is every few seconds fine? We
  will assume updates every 3–5 seconds.
- How is the fare calculated — flat rate, distance-based, or surge pricing (price rises when
  demand is high)? We treat this as a pluggable calculation and just store the result.
- What happens if no driver is found nearby? Assume the trip is cancelled after a timeout.
- Can a rider or driver cancel mid-trip? Assume yes, before the trip becomes "ongoing."

Rough scale for this chapter: 5 million trips per day, 500,000 active drivers, each sending a
location update every 4 seconds while online. That is over 100,000 location writes per second
at peak — a huge number, and it drives one of the biggest design decisions below.

## The Schema

```sql
CREATE TABLE users (
    id              BIGSERIAL PRIMARY KEY,
    name            TEXT NOT NULL,
    phone           TEXT NOT NULL UNIQUE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
-- One row per rider.

CREATE TABLE drivers (
    id              BIGSERIAL PRIMARY KEY,
    name            TEXT NOT NULL,
    phone           TEXT NOT NULL UNIQUE,
    status          TEXT NOT NULL DEFAULT 'offline',  -- available | busy | offline
    current_lat     DOUBLE PRECISION,
    current_lng     DOUBLE PRECISION,
    location_updated_at TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_drivers_status ON drivers(status);
-- One row per driver. current_lat/current_lng hold the LATEST known position only — not a
-- history table. See "Key Design Decisions" for why this is not how we search nearby drivers.

CREATE TABLE trips (
    id              BIGSERIAL PRIMARY KEY,
    rider_id        BIGINT NOT NULL REFERENCES users(id),
    driver_id       BIGINT REFERENCES drivers(id),      -- NULL until a driver is assigned
    status          TEXT NOT NULL DEFAULT 'requested',  -- requested | assigned | ongoing |
                                                          -- completed | cancelled
    pickup_lat      DOUBLE PRECISION NOT NULL,
    pickup_lng      DOUBLE PRECISION NOT NULL,
    drop_lat        DOUBLE PRECISION,
    drop_lng        DOUBLE PRECISION,
    fare_minor      BIGINT,                              -- fare in cents, set at completion
    requested_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
    assigned_at     TIMESTAMPTZ,
    started_at      TIMESTAMPTZ,
    completed_at    TIMESTAMPTZ,
    ...
);
CREATE INDEX idx_trips_rider ON trips(rider_id, requested_at DESC);
CREATE INDEX idx_trips_driver ON trips(driver_id, requested_at DESC);
-- One row per trip. status tracks where the trip is in its lifecycle. The *_at timestamp
-- columns double as a lightweight history of when each stage was reached.

CREATE TABLE trip_status_history (
    id              BIGSERIAL PRIMARY KEY,
    trip_id         BIGINT NOT NULL REFERENCES trips(id),
    status          TEXT NOT NULL,
    changed_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_trip_history_trip ON trip_status_history(trip_id, changed_at);
-- Append-only log: one row per status change. Useful for support tickets and analytics.
```

`fare_minor` stores money as an integer in minor units (cents), never as a float, so that
rounding errors do not creep into billing.

## Key Design Decisions — the Reasoning

### 1. The trip state machine: a status column plus a history table

A trip moves through a fixed set of stages, in a fixed order:

```
REQUESTED → ASSIGNED → ONGOING → COMPLETED
                  \-------------→ CANCELLED
```

This is a **state machine**: a system always in exactly one of a known set of states, moving
between states only along allowed paths. We model the current state with one column,
`trips.status`. It answers "where is this trip right now?" with a single cheap lookup.

But a single column cannot answer "when did each stage happen?" or "how many trips get
cancelled after assignment, versus before?" For that, we add `trip_status_history`, an
**append-only table** (insert-only, never updated or deleted). Every status change writes one
new row here, alongside updating `trips.status`. This gives a full audit trail without slowing
down the main "what is the current status" query, which still just reads one column.

The trade-off: keeping both means writing to two tables on every status change, usually in the
same transaction. This is a small extra cost, paid rarely, in exchange for a full history the
single column cannot give. Chapter 6 uses the same "status column + updated_at" pattern for
bookings — this is a general building block, not something specific to ride-hailing.

### 2. Finding nearby drivers: why a plain table scan does not work

When a rider requests a trip, the system must answer: "which available drivers are near this
pickup point?" The `drivers` table stores `current_lat` and `current_lng`, so it is tempting to
write:

```sql
SELECT id FROM drivers
WHERE status = 'available'
  AND current_lat BETWEEN :lat - 0.05 AND :lat + 0.05
  AND current_lng BETWEEN :lng - 0.05 AND :lng + 0.05;
```

This does not scale. A normal B-tree index (the default index type in Postgres) is built for
comparing one value at a time — it is very good at "give me rows where `x > 5`," but it cannot
efficiently answer "give me rows near this point in two-dimensional space" at the same time.
With hundreds of thousands of drivers moving every few seconds, a query like this ends up
scanning far more rows than it needs, or needs constant, expensive re-indexing.

Real systems use a **geo index** instead — an index built specifically for "find things near a
point," not "compare one value at a time." Two common choices:
- **PostGIS**, a Postgres extension that adds real geographic data types and a spatial index
  (built on a structure called an R-tree). It lets you write `ST_DWithin(location, point,
  radius)` and get a fast, correct nearest-neighbor answer inside Postgres itself.
- **Geohash or a quadtree, in a fast in-memory store.** A geohash turns a `(lat, lng)` pair into
  a short string, where nearby points share the same string prefix. A quadtree is a tree that
  splits the map into smaller and smaller squares, so a search only checks drivers in nearby
  squares, not the whole table. Uber's own systems use this kind of structure, kept in memory
  for speed, rather than a plain SQL table.

Say this reasoning out loud in an interview: *storing* "this driver's current location" is a
simple row update, but *searching* by location is a different problem, and it needs a different
data structure. A normal index is the wrong tool for "nearest neighbor" queries.

### 3. Claiming a driver atomically: no driver on two trips at once

Two riders might request a trip near the same driver at almost the same moment. The system must
assign that driver to only one of them. This is the same double-booking problem as Chapter 6,
just for a different resource (a driver instead of a room).

The fix is an atomic conditional update — one that checks and changes a row in a single
database step, so no other request can slip in between the check and the change:

```sql
UPDATE drivers
SET status = 'busy'
WHERE id = :driver_id AND status = 'available';
```

If this statement reports that it updated one row, the claim worked, and we can safely set
`trips.driver_id` and move the trip to `assigned`. If it reports zero rows updated, another
request already claimed this driver a moment earlier, and we must pick a different driver.

The key point: never do this as two separate steps — first `SELECT` the status to check it,
then `UPDATE` it. Two steps leave a gap where a second request can read "available" before the
first request writes "busy." Here, the check and the write happen as one atomic operation, so
the gap cannot occur, even under heavy concurrent load.

### 4. Fare stored on the trip, not looked up later

`trips.fare_minor` stores the actual price charged, filled in once the trip completes. We do
not store a rate and calculate the fare on demand later, because rates change over time (surge
pricing, promotions). If we only stored a reference to "the current rate," an old trip's
receipt would show today's price, not what the rider actually paid. This is the same "price at
the time of the event" rule from Chapter 3 (order line items) and Chapter 8 (ledger entries) —
copy the value onto the row when the event happens, never rely on a live lookup for something
that must stay fixed in history.

## Scaling It

- **Index first.** `idx_drivers_status` speeds up "find available drivers" as a first filter,
  before any geo search narrows it further. `idx_trips_rider` and `idx_trips_driver` cover the
  "my trip history" screens.
- **Location updates do not belong in the main database.** At 100,000+ writes per second, where
  each new update just overwrites the last one, this is a huge write load for data we do not
  need to keep permanently. Route these updates to a fast in-memory store — Redis (an in-memory
  key-value store) is a common choice, often paired with a geohash/quadtree index or Redis's
  own built-in geospatial commands. Keep Postgres for data that must be durable: users, driver
  identity fields, and trips.
- **Cache the "nearby available drivers" answer** for a few seconds in the same fast store,
  since many riders in the same area trigger nearly the same query.
- **Read replicas** for trip history and analytics, so heavy reporting does not slow down the
  live matching path.
- **Shard trips by region or rider_id** once a single Postgres instance cannot keep up. Most
  trip queries are scoped to one rider, one driver, or one region, so sharding this way keeps
  most queries on one shard.

## Interview Tips & Common Mistakes

- Do not treat `current_lat`/`current_lng` on `drivers` as the answer to "find nearby drivers."
  Say clearly that this column holds "latest position only," and that searching by location
  needs a separate geo index, not a scan of this table.
- Always name PostGIS or a geohash/quadtree when asked about nearby search. Saying "we'd need a
  spatial index" without naming a real option is a weaker answer.
- Bring up the double-assignment race condition yourself, before the interviewer asks "what if
  two riders get the same driver?" Show the atomic `UPDATE ... WHERE status = 'available'` and
  explain why a `SELECT` then `UPDATE` is unsafe.
- Do not forget to say the fare is snapshotted onto the trip. Candidates who describe a live
  "look up the current rate" approach usually get pushed on "what if rates changed since?"
- Location writes are the single biggest volume number in this system. Say the write rate out
  loud (100,000+ writes/sec at this scale) — it is the clearest justification for moving
  location data out of the primary database.
