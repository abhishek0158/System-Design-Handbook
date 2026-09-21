# Chapter 6 — Booking / Reservation (no double-booking)

A booking system lets a guest reserve a resource, like a hotel room, for a time range. Interviewers love this question because it is easy to get the double-booking rule wrong.

## Requirements & Clarifying Questions

Core features:
- A guest books a room for a date range (check-in to check-out).
- Two bookings for the same room must never overlap.
- A booking has a status: held, confirmed, or cancelled.
- A guest can search for rooms that are free in a date range.

Questions to ask the interviewer:
- Is this one hotel, or many hotels (multi-tenant)? Assume one hotel with many rooms, for focus.
- Can a guest "hold" a room while paying, before the booking is confirmed? Assume yes.
- Do we need per-night pricing, or is a flat total price enough? Assume flat price for this chapter.
- Scale: assume a mid-size hotel chain, thousands of rooms, tens of thousands of bookings a day at peak (like a big sale or event).

Out of scope for this chapter: payments, refunds, per-night dynamic pricing. We focus on the booking table and the no-overlap rule.

## The Schema

```sql
CREATE TABLE rooms (
    id          BIGSERIAL PRIMARY KEY,
    hotel_id    BIGINT NOT NULL REFERENCES hotels(id),
    room_number TEXT NOT NULL,
    room_type   TEXT NOT NULL,   -- e.g. 'single', 'suite'
    ...
);
-- One row per physical room. The unit we book.

CREATE TABLE guests (
    id      BIGSERIAL PRIMARY KEY,
    email   TEXT NOT NULL UNIQUE,
    name    TEXT NOT NULL,
    ...
);
-- One row per guest. Simple lookup table.

CREATE TABLE bookings (
    id          BIGSERIAL PRIMARY KEY,
    room_id     BIGINT NOT NULL REFERENCES rooms(id),
    guest_id    BIGINT NOT NULL REFERENCES guests(id),
    start_date  DATE NOT NULL,
    end_date    DATE NOT NULL,
    status      TEXT NOT NULL DEFAULT 'held',  -- held, confirmed, cancelled
    hold_expires_at TIMESTAMPTZ,               -- only set when status = 'held'
    total_price NUMERIC(10,2) NOT NULL,
    created_at  TIMESTAMPTZ NOT NULL DEFAULT now(),
    CHECK (start_date < end_date),
    EXCLUDE USING gist (
        room_id WITH =,
        daterange(start_date, end_date) WITH &&
    ) WHERE (status <> 'cancelled')
);
-- One row per booking attempt. The EXCLUDE constraint is what stops double-booking.
-- It needs the btree_gist extension (CREATE EXTENSION btree_gist) for the room_id equality check.
```

A one-line note: `rooms` is what you book, `guests` is who books it, `bookings` is the fact that links them with a date range and a status.

## Key Design Decisions — the Reasoning

### 1. The overlap rule

Two bookings for the same room overlap if they share at least one day. In simple terms:

`booking A overlaps booking B` when `A.start < B.end` AND `B.start < A.end`.

Picture two bars on a timeline. Each bar starts at check-in and ends at check-out. If you can slide the bars so they touch but do not cross, they do not overlap. They overlap only if one bar's start is before the other bar's end, in both directions. A booking from day 1 to day 5, and another from day 5 to day 8, do NOT overlap — day 5 is a checkout day for one and a checkin day for the other, and hotels treat that as a clean handover.

This is why `end_date` is "checkout day, exclusive." The range `[start, end)` includes the start day but not the end day. That matches how hotels work: you leave on day 5, and the room is free from day 5 onward.

### 2. Enforcing no-overlap safely under concurrency

This is the star of the chapter, so let's build it up.

**Naive approach (has a race condition):** Before inserting a booking, run a `SELECT` to check if any existing booking for that room overlaps the new date range. If none found, `INSERT`. This looks safe, but it is not, when two guests book the same room at the same time.

Here is the race:
1. Guest A runs the overlap check for room 101, days 10–12. No conflict found.
2. Guest B runs the overlap check for room 101, days 10–12, at almost the same moment. No conflict found either — guest A has not inserted yet.
3. Guest A inserts their booking.
4. Guest B inserts their booking.

Both checks passed because they ran before either `INSERT` committed. Now room 101 has two confirmed bookings for the same days. The check and the insert are two separate steps, and nothing stops another transaction from running its own check in the gap between them. This is called a "check-then-act" race condition, and it happens under real concurrent load, not just in theory.

**Fix, option 1 — row locking inside a transaction:**

```sql
BEGIN;
SELECT id FROM bookings
WHERE room_id = 101
  AND status <> 'cancelled'
  AND daterange(start_date, end_date) && daterange('2026-10-10', '2026-10-12')
FOR UPDATE;
-- if no rows returned, safe to insert
INSERT INTO bookings (room_id, guest_id, start_date, end_date, status, total_price)
VALUES (101, 55, '2026-10-10', '2026-10-12', 'held', 250.00);
COMMIT;
```

`SELECT ... FOR UPDATE` locks the matching rows so a second transaction cannot read past them until the first transaction commits or rolls back. But this only locks rows that already exist. If there are zero existing bookings for that room and date range, there is nothing to lock, and two transactions can still both pass the check and both insert. To close this gap fully, you need a lock on the *room itself*, not just on matching rows — for example `SELECT id FROM rooms WHERE id = 101 FOR UPDATE` first, which serializes all booking attempts for that room. This works, but it means every booking attempt for a room queues up one at a time, even for non-overlapping dates.

**Fix, option 2 — a database EXCLUSION constraint (the strong option):**

PostgreSQL can enforce "no two rows for the same room may have overlapping date ranges" as a constraint, the same way `UNIQUE` enforces "no two rows may have the same value." This is the `EXCLUDE` constraint shown in the schema above:

```sql
EXCLUDE USING gist (
    room_id WITH =,
    daterange(start_date, end_date) WITH &&
) WHERE (status <> 'cancelled')
```

Read it as: "reject a new row if it has the same `room_id` (`=`) AND an overlapping (`&&`) date range as an existing row." It uses a GiST index (a general-purpose index type good at range and overlap queries, unlike the usual B-tree index which only handles equality and ordering) to check overlaps efficiently. The `WHERE (status <> 'cancelled')` part means a cancelled booking does not block a new one for the same dates — the room is free again once a booking is cancelled.

With this constraint, you can skip the manual `SELECT ... FOR UPDATE` locking dance. You just `INSERT` and let the database reject the insert if it overlaps. This is safe under any level of concurrency, because the check and the insert happen as one atomic operation inside the database engine — there is no gap for a second transaction to sneak into. If the insert fails, your application code catches the error and tells the guest the room is no longer available.

Trade-off: the `EXCLUDE` constraint is PostgreSQL-specific (MySQL does not have it). If you must support MySQL, you fall back to option 1 (lock the room row, then check, then insert) inside a transaction.

### 3. Booking status and short holds

A booking is not always final the moment it is created. Many booking flows work like this: the guest picks dates, the system creates a `held` booking so nobody else can grab the room while the guest enters payment details, and then the hold turns into `confirmed` once payment succeeds.

A hold should not last forever — if the guest abandons the page, the room must free up. That is what `hold_expires_at` is for. A background job (or a check at read time) treats any `held` booking past its `hold_expires_at` as expired, and either deletes it or flips it to `cancelled`. A typical hold window is 10–15 minutes.

Why not just use `confirmed` and `cancelled` only? Because without a `held` state, you would have to fully commit a booking (and take payment) before locking the room, which is a bad user experience — the guest could enter payment details for a room that gets taken by someone else in the meantime. The `held` status buys a short, safe window.

## Scaling It

- **Index first.** The GiST index used by the `EXCLUDE` constraint already speeds up "is this room free in this range?" lookups. Add a plain B-tree index on `(room_id, status)` for fast lookup of a room's active bookings.
- **Read replicas** for search traffic ("show me free rooms this weekend") — search reads don't need the absolute latest data, a few seconds of lag is fine.
- **Partition the `bookings` table by resource (room or hotel) or by date range** once it grows very large (tens of millions of rows). Partitioning by `hotel_id` works well if queries almost always filter by hotel. Partitioning by month works well if old bookings are rarely queried and can be archived.
- **Hot resources.** A handful of rooms (or one popular event venue) can get a burst of concurrent booking attempts — think a flash sale for a famous suite. The `EXCLUDE` constraint keeps correctness under this load, but many attempts will fail fast with a conflict error, which is fine and expected. Do not queue everyone through a single lock unless you must (that serializes and slows down unrelated rooms).

## Interview Tips & Common Mistakes

- Always state the overlap formula (`start1 < end2 AND start2 < end1`) out loud. It is the crux of the question.
- If asked "how do you stop double-booking," do not stop at "check before insert." Say why that has a race condition, then name the fix: an `EXCLUDE` constraint, or explicit row/resource locking in a transaction.
- Do not forget the `CHECK (start_date < end_date)` guard — a zero-length or backwards range is a common bug.
- Remember to exclude cancelled bookings from the overlap check, or guests will never be able to rebook a room after a cancellation.
- Mention the hold-and-expiry pattern if payment is part of the flow — it shows you have thought about the real user journey, not just the schema.
