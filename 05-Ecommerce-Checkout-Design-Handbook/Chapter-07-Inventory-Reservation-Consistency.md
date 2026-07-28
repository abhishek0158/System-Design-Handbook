# Chapter 7 — Inventory Reservation & Consistency

Chapter 6 gave us the saga. It has these steps: revalidate the session, **reserve inventory**, authorize payment, create the order, capture payment, confirm. This chapter is a deep dive on step 2, inventory reservation.

Inventory reservation is the moment GlobalMart keeps its promise to sellers. The promise is simple: "we will not sell what you don't have." If we break this promise, a buyer gets very unhappy. This chapter explains one state machine and a few concurrency techniques. Together they make sure that "only one buyer wins the last unit" is always true, even under heavy load.

We will also look at what happens when one single product (one SKU) gets hit by thousands of requests per second during a flash sale. And we will explain why inventory is the one part of GlobalMart where we choose correctness over availability.

Chapter 6 was about running the whole saga. This chapter is about making **one step of that saga** provably correct under concurrency. Concurrency means many requests happening at the same time. This is the most asked topic in checkout system design interviews. Interviewers also ask about idempotency and payment gateways. But "how do you stop two buyers from buying the last unit" is the question that shows if a candidate has really built concurrent systems before.

## 1. The Problem: One Last Unit, Two Buyers

Picture a product listing with exactly **1 unit** left in stock. Two buyers, Alice and Bob, both have this item in their cart. Both click "Place Order" within the same 50 millisecond window. The Checkout Orchestrator (from Chapter 6) sends both requests to the Inventory Service at almost the same time.

We need three things to be true at the same time:

1. **No oversell.** At most one of Alice's or Bob's orders can reserve that one unit. The value `reserved + sold` must never go above `on_hand`.
2. **No silent loss.** The buyer who loses should not be left waiting with no answer. Bob should get an immediate and clear "out of stock" message. He can then retry, choose another product, or save it to his wishlist. He should not get a request that hangs, and he should not get a reservation that quietly expires 15 minutes later with no notice.
3. **A fast decision.** The whole decision must happen inside the 80 millisecond slice that the place-order latency budget (Chapter 2) gives to inventory reservation. The total budget is 2.5 seconds at p99, and most of that time is used by payment authorization (300 to 1500 milliseconds).

### Defining "available"

Everything in this chapter depends on one simple formula. For one `listing_id` (a seller's specific product offer):

```
available = on_hand − reserved − sold
```

- `on_hand` is the stock count the seller told us about.
- `reserved` is the sum of quantities across all reservations that are currently in the **HELD** state. These are buyers who are in the middle of checkout right now.
- `sold` is the number of units already locked into **COMMITTED** orders. GlobalMart actually subtracts sold units directly from `on_hand` when an order is committed. So `on_hand` always means "what is left after real sales." The `reserved` number is just a temporary hold on top of that.

A reservation request for quantity `q` succeeds only if `available >= q` at the exact moment we check it. This whole chapter is about how to check and update this rule safely, even when many requests arrive together.

Here is the naive, wrong way to do it. First read `available` in your application code. Then check if it is `>= q`. Then write a new reservation. This is a classic **check-then-act race**. A race means two things happen in an order we did not plan for, and it breaks our logic.

```
T1: read available = 1        (Alice's request)
T2: read available = 1        (Bob's request, same instant)
T1: available >= 1 → reserve 1, available becomes 0
T2: available >= 1 → reserve 1, available becomes -1   ← OVERSOLD
```

Both requests read the same stale value of `available = 1` before either one wrote anything. This bug is called Time-Of-Check-To-Time-Of-Use, or TOCTOU. Every concurrency-control method in this chapter exists to close this exact gap. Section 3 of this chapter shows four ways to turn "check" and "act" into one single atomic step, instead of two separate steps.

It helps to be precise about what "exactly one must win" really means. The reservation decision itself is instant: it is either HELD or rejected right away. But the order is not final yet. Alice's reservation still has to survive payment authorization. If her card gets declined, her HELD reservation is released (see Section 6). The unit becomes available again, and Bob might get it on a retry.

So "exactly one wins" really means "at most one buyer holds the unit at any single instant." It does not mean "the first buyer to click always gets the sale." This is a subtle but important point to say out loud in an interview. It is also why the reservation step and the payment step are kept as two separate steps in the saga, instead of one combined step.

## 2. Reservation Lifecycle: HELD → COMMITTED → RELEASED

The `InventoryReservation` record, from the canonical data model in Chapter 4, is a small state machine. A state machine is a model where something can only be in one of a few defined states at a time, and it moves between them in fixed ways. This is not just a simple true/false flag.

```
{ "reservation_id", "listing_id", "seller_id", "qty", "checkout_group_id",
  "state": "HELD | COMMITTED | RELEASED", "expires_at" }
```

### The state machine diagram

```
                    create (reserve)
                          │
                          ▼
                  ┌───────────────┐
        ┌────────▶│     HELD      │◀────────┐
        │         │ (TTL ~15 min) │          │
        │         └───────┬───────┘          │
        │                 │                  │
   re-reserve        order placed        TTL expiry /
   (retry, same       successfully       explicit release
   idempotency key)   (saga step 4)      (saga compensation,
        │                 │               buyer abandons,
        │                 ▼               payment declined)
        │         ┌───────────────┐          │
        └─────────│   COMMITTED   │          │
                   │  (terminal)   │          │
                   └───────────────┘          │
                                               ▼
                                       ┌───────────────┐
                                       │   RELEASED    │
                                       │  (terminal)    │
                                       └───────────────┘
```

- **HELD** is created the moment the Inventory Service accepts a reservation request, in step 2 of the saga. It gets a hard time limit, called a TTL (Time To Live), of about **15 minutes** (from Chapter 4). It is linked to a `checkout_group_id`. This means all sub-orders in a multi-seller cart can be found and released together if the saga fails.
- **COMMITTED** is set in saga step 4, when the order is created, right after payment authorization succeeds in step 3. This is the only step that turns a soft, temporary hold into a real, permanent sale. It usually happens inside the same database transaction that reduces `on_hand` and creates the `Order` row. So "commit the reservation" and "reduce on_hand" happen together, as one atomic step.
- **RELEASED** can happen in three ways. First, a **TTL sweeper** job finds an expired HELD row and flips it to RELEASED. Second, the saga runs a **compensating action** to release it directly. This happens if payment is declined, the buyer leaves, or another sub-order in the same cart fails and the whole checkout group must be rolled back. Third, if the same request retries with the same idempotency key, and a HELD reservation already exists for that key, the system reuses it instead of creating a second one.

Both COMMITTED and RELEASED are final states. A reservation never comes back to life after reaching one of these. If a buyer wants to try buying again after a release, that creates a brand new reservation with a new `reservation_id`. This matters for record-keeping. The list of all reservation rows becomes a permanent history, not just one field that keeps changing. This history is what makes the reconciliation checks in Section 6 possible.

### The TTL sweeper

A background job checks for reservations that ran out of time. This job could be a scheduled task, a Redis key-expiry event, or a delayed Kafka message. The choice depends on the implementation, and Section 3's recommendation covers this. The job continuously looks for rows like this:

```sql
SELECT reservation_id, listing_id, qty
FROM inventory_reservations
WHERE state = 'HELD' AND expires_at < NOW()
LIMIT 500;
```

For each row found, the sweeper flips its state to RELEASED. At the same time, in the same transaction, it reduces the listing's `reserved` counter. Both changes must happen together. If only one of them happens, the counter drifts away from the truth, which we discuss in Section 6.

Why 15 minutes? This time is chosen to be comfortably longer than the place-order p99 latency of 2.5 seconds, plus some room for retries. So almost every HELD reservation reaches COMMITTED or an explicit RELEASED state well before the sweeper ever needs to step in. The sweeper's real job is to recover stock from buyers who close their browser tab mid-checkout, whose payment step times out with no clean response, or whose saga process crashes before it can run its cleanup step.

## 3. Ways to Control Concurrency

This section is the technical heart of the chapter. We look at four ways to make the check-then-act sequence from Section 1 into one atomic step. All four methods are judged against the same goal: decide "hold or reject" for one listing, correctly, well inside the 80 millisecond budget, even under heavy contention.

### (a) Pessimistic row lock — `SELECT ... FOR UPDATE`

"Pessimistic" locking means we assume a conflict will happen, so we lock the row first. We take an exclusive lock on the inventory row before we even read it. No other transaction can read or write that row until we finish and either commit or roll back.

```sql
BEGIN;

SELECT on_hand, reserved
FROM inventory
WHERE listing_id = :listing_id
FOR UPDATE;                                  -- blocks other writers/lockers

-- application checks: on_hand - reserved - sold >= :qty
UPDATE inventory
SET reserved = reserved + :qty
WHERE listing_id = :listing_id;

INSERT INTO inventory_reservations
  (reservation_id, listing_id, qty, checkout_group_id, state, expires_at)
VALUES (:res_id, :listing_id, :qty, :group_id, 'HELD', NOW() + INTERVAL '15 minutes');

COMMIT;
```

- **Correctness:** This is easily correct. The lock forces every reservation attempt on this row to happen one at a time, in a queue. So the TOCTOU gap from Section 1 simply cannot open.
- **Cost:** Every concurrent request for the *same* listing must wait in line behind whoever holds the lock. Throughput on one hot row drops to roughly `1 / (lock hold time)`. Lock hold time includes the full round trip of the transaction, both statements and the network delay, usually 5 to 15 milliseconds. This limits one row to about 70 to 200 reservations per second. That is fine for 99.9% of products, but it becomes a serious bottleneck for a flash-sale product (see Section 4).
- **What breaks under load:** Lock waits build up. Under heavy contention, you get lock-wait timeouts, the connection pool runs out of free connections, and this slowness spreads into unrelated requests that share the same pool.

### (b) Optimistic concurrency — conditional `UPDATE`

"Optimistic" means we assume most requests will not conflict, so we do not lock ahead of time. Instead, we make the write itself conditional. The rule check and the update happen as one single atomic database statement.

```sql
UPDATE inventory
SET reserved = reserved + :qty,
    version   = version + 1
WHERE listing_id = :listing_id
  AND on_hand - reserved - sold >= :qty;      -- the atomic guard

-- application checks affected-row count:
-- 1 row updated  → success, insert the HELD reservation row
-- 0 rows updated → available < qty → reject ("out of stock" / partial-fill decision)
```

The `version` column here is optional, because the `WHERE` condition itself already acts as the safety check. But keeping a `version` column is useful for other simple read-then-write flows elsewhere in the system, like an admin manually adjusting stock, using `WHERE version = :expected_version`.

- **Correctness:** This is correct because the database applies the `WHERE` check and the `SET` change as one single atomic action per row. No other transaction can see or act on a half-finished state. Two `UPDATE` statements hitting the same row at the same time still get serialized internally by the database engine. But the application code never holds an open lock across a network round trip. The row is only "busy" for the length of one single statement, not for a full read-then-write cycle.
- **Cost:** This has a much shorter conflict window than option (a), often under one millisecond, because there is no round trip between reading and writing. Throughput on one hot row improves roughly 5 to 10 times compared to pessimistic locking.
- **What breaks under load:** Under very high contention, you get many "0 rows updated" results. This result is actually correct, because those buyers really did lose the race. But each of those failed attempts still uses real database write capacity on one row, just to get told "no." At large scale this wastes a lot of write capacity, and Section 4 gives exact numbers.
- **Why we don't need an explicit lock:** It is worth being able to explain exactly why this is safe, even with the database's normal `READ COMMITTED` isolation level, without extra locking. A single-statement `UPDATE` is itself the smallest unit of atomic work the database guarantees. The engine takes whatever internal lock it needs, but only for the length of that one statement. It checks the `WHERE` condition against the latest committed value and applies the `SET` change before releasing the lock. Two `UPDATE` statements on the same row still get queued internally, just like in option (a). But this queueing is invisible to the calling application, because the application never opens a second statement while still holding the first transaction open. This is the real difference between "optimistic concurrency" and "no concurrency control at all." It is not skipping locking. It is shrinking the locked section down to exactly one statement, instead of a full client round trip.

### (c) Atomic decrement in Redis

Redis is an in-memory data store often used as a fast cache or counter. Here we move the hot counter entirely out of the relational database and into Redis, using its built-in atomic operations as a fast admission gate.

```lua
-- Redis Lua script — atomic: read, check, decrement in one round trip
local available = redis.call('GET', KEYS[1])
if tonumber(available) >= tonumber(ARGV[1]) then
    redis.call('DECRBY', KEYS[1], ARGV[1])
    return 1   -- admitted
else
    return 0   -- rejected
end
```

```
EVALSHA <script_sha> 1 inv:available:{listing_id} 1
```

This script runs as one call using `EVALSHA` (or a plain `DECRBY` guarded by Lua, as shown above). Redis executes this on a single thread, so there is no race between two requests at all. This single-threaded execution is itself the guarantee of safety, with no extra locking needed.

- **Correctness:** This works correctly as an admission gate. No two callers can decrement the counter below zero. But Redis by itself is **not** the durable source of truth. Durable means the data survives crashes and restarts safely. If a Redis node restarts without correct persistence settings, or a network split lets a stale replica answer requests, the counter can drift away from the real state stored in the Inventory DB. GlobalMart's recommended pattern, described below, is this: Redis is the fast gate. A durable reservation row is written to the Inventory DB right after the Redis check passes, and this write happens synchronously, not "eventually" (Section 8 explains why eventual writing is not acceptable here). The two records are then kept in sync by the reconciliation job in Section 6.
- **Cost:** This is by far the fastest option. It takes single-digit milliseconds and can handle tens of thousands of operations per second on one key, using modest hardware, because there is no disk-based transaction in the hot path.
- **What breaks under load:** A Redis failover or restart can lose the last few decrements if it is not set up with durable disk writes (AOF fsync) or a backing durable write. Because of this, Redis must always be paired with option (b) or (d) as the real durable record. Redis should never be the only place that remembers the count.

### (d) Reservation rows plus a derived count

Instead of keeping one mutable `reserved` number, we treat "reserved" as a value we calculate by adding up all HELD rows in an append-only table. Append-only means rows are only ever added, never changed in place.

```sql
BEGIN;

SELECT COALESCE(SUM(qty), 0) AS reserved_total
FROM inventory_reservations
WHERE listing_id = :listing_id AND state = 'HELD'
FOR UPDATE;                                   -- lock the reservation rows for this listing

-- application checks: on_hand - reserved_total - sold >= :qty

INSERT INTO inventory_reservations
  (reservation_id, listing_id, qty, checkout_group_id, state, expires_at)
VALUES (:res_id, :listing_id, :qty, :group_id, 'HELD', NOW() + INTERVAL '15 minutes');

COMMIT;
```

- **Correctness:** This is correct, and it gives us a full audit trail for free. Every hold, commit, and release becomes a row with a timestamp. This is exactly what the reconciliation jobs in Section 6, and the audit requirements in Chapter 4, need.
- **Cost:** This is worse than option (b). You are adding up rows from a table that keeps growing. You can reduce this cost with an index on `(listing_id, state)` and by regularly archiving old, finished rows. But you still need a lock, or a serializable isolation level, to make the "sum, then insert" step safe. So it inherits the same contention problems as pessimistic locking on hot rows.
- **What breaks under load:** The table can grow very large for hot products, unless finished rows (COMMITTED or RELEASED) are archived away aggressively. A slow `SUM` calculation done under a lock is even worse than a slow simple read done under a lock.

### Comparing the four options

| Strategy | How it stays atomic | Typical latency | Hot-row throughput ceiling | Audit trail | Durable by itself? |
|---|---|---|---|---|---|
| (a) Pessimistic `FOR UPDATE` | Explicit row lock across the round trip | 5–15 ms | ~100–200/sec/row | Only if paired with (d) | Yes |
| (b) Optimistic conditional `UPDATE` | Atomic single-statement guard | 1–3 ms | ~500–1,000/sec/row | No — needs a companion reservation-row insert | Yes |
| (c) Redis atomic decrement | Single-threaded execution | <1 ms | ~10,000+/sec/key | No | No — needs a durable backstop |
| (d) Reservation rows + SUM | Lock + aggregate | 5–20 ms (grows with row count) | ~100–200/sec/row | Yes, naturally | Yes |

### Our recommendation

GlobalMart does not pick just one method. It uses a **layered combination**, because no single method gives you both speed and durability on its own.

1. **Every listing's counter update uses method (b)**, the conditional `UPDATE`, as the main durable check inside the Inventory DB. This alone handles almost all products, since most products never see any real contention.
2. **Every reservation still writes a row**, which is the audit ledger idea from method (d). But this row insert does not decide whether the reservation is allowed. The result of the conditional `UPDATE` is what decides that. The row insert only records what happened. This gives us the audit trail without paying the "lock plus SUM" cost of method (d) on the busy path.
3. **For products flagged as hot** (either known ahead of time because a flash sale is planned, or detected live by a requests-per-second alarm on that listing), we add method (c), a Redis atomic counter, in front of the check. The conditional `UPDATE` still runs right behind it as the durable write, and it always runs synchronously, never deferred to a queue for later. This is because a durable inventory write can never be allowed to be "eventually" true (see Section 8). Redis here works like a shock absorber for the volume of reads and decrements. It is not a replacement for the real database transaction.

This layered answer is exactly what interviewers want to hear. Pessimistic locking is the easy, correct, but non-scaling default choice. Optimistic conditional updates are the right general-purpose answer for almost everything. Redis is the right answer specifically for the hot-product case, and only when combined with a durable write, never used alone.

**Fitting inside the latency budget.** Chapter 2's place-order budget gives 80 milliseconds to inventory reservation, out of a 2.5 second total p99 budget, most of which is used by payment authorization. For a normal, non-hot product, the optimistic conditional `UPDATE` (1 to 3 ms) plus a reservation-row insert (1 to 3 ms) plus network round trips easily fits inside 80 ms, with room to spare. Most products never need the Redis layer at all. The Redis layer exists specifically to keep hot products inside that same 80 ms budget, even under contention that would push a pessimistic-locking approach's queue delay past the budget within the first few hundred milliseconds of a flash sale (see Section 4's worked numbers). So the budget, not just good taste, is why GlobalMart does not use Redis everywhere. Redis adds extra durability and reconciliation cost (Section 6), and that cost is only worth paying once a product's write rate threatens to break the latency target.

## 4. The Hot-SKU Problem at Flash-Sale Scale

Chapter 2 set the peak checkout-API rate at **about 300,000 requests per second (QPS)**. This peak concentrates especially on place-order during flash sales, with order-placement itself peaking at **about 50,000 orders per second**, which is 20 times the normal average of about 2,300 per second. A flash sale, by definition, means many buyers converging on *one or a few products*. So this peak load does not spread evenly across GlobalMart's whole catalog. It lands hard on just one row.

**A worked example.** Say a flash-sale listing has 5,000 units. GlobalMart sends a push notification to 2 million buyers who had wishlisted this item. Based on past conversion rates, this leads to about **8,000 place-order attempts per second** on this single `listing_id`, in just the first 10 seconds. This number is small compared to the system's overall 300K QPS peak budget. But it is entirely concentrated on one row, on one shard, because the Inventory DB is sharded by `listing_id`/SKU (Chapter 2's sharding rule). This one shard now has to handle 8,000 attempts per second against a single row.

- With **pessimistic locking (a)**, at a ceiling of about 150 reservations per second per row, the queue of waiting requests grows by about 7,850 every single second. At an 80 ms budget per request, this queue alone breaks the latency budget within about the first 10 milliseconds of the sale starting. Lock-wait timeouts then start cascading into the shared connection pool, which can slow down other, unrelated products on the same database shard too.
- With **optimistic conditional updates (b)**, at about 800 per second per row, the shard lasts a bit longer, but still falls behind by roughly 7,200 requests per second. These rejected requests are not wrong, since rejecting once stock hits zero is the correct outcome. But each one still uses a full database round trip and a connection-pool slot, just to be told "no." This eats into the shard's capacity to serve legitimate work.
- **About 7,995 of those 8,000 buyers per second will lose** the race for these 5,000 units within a few seconds, no matter which method we use. The real engineering problem is not "how do we let everyone win." It is "how do we absorb 8,000 requests per second of contention without overloading the shard or the connection pool for everyone else on it, while rejecting losing buyers fast and honestly instead of making them wait and eventually time out."

This reframes the real goal. **We should not try to make one row handle 8,000 writes per second. We should cut the contention before it ever reaches the row.**

### Mitigation 1: Inventory sharding, or splitting stock into buckets

We split one listing's stock into **N smaller virtual buckets** (for example, `listing_id#0` through `listing_id#15`). Each bucket has its own row and its own slice of `on_hand`. A reservation request is routed to one bucket, either by a random value, round-robin order, or a hash of the `buyer_id` or `checkout_group_id`. That request only competes with roughly 1/N of the total traffic, the share that lands on the same bucket.

```
listing_id = "SKU-9981"        on_hand = 5,000
   ├─ SKU-9981#0   on_hand = 313
   ├─ SKU-9981#1   on_hand = 313
   ├─ ...
   └─ SKU-9981#15  on_hand = 312
```

- 8,000 requests per second divided by 16 buckets gives about **500 requests per second per bucket**. This is comfortably inside even pessimistic locking's ceiling, and well inside optimistic locking's ceiling.
- **Trade-off:** Stock becomes fragmented, or split up. If bucket #3 runs empty while bucket #7 still has 40 units left, a request sent to bucket #3 gets rejected even though stock technically still exists somewhere else. This is a false rejection. We can reduce this problem by rebalancing buckets as they drain, either by merging near-empty buckets, or by falling back to checking across all buckets once the total remaining stock drops below a small threshold. For example, we could consolidate the last 5% of stock into a single bucket, to avoid fragmenting the very last units. This general idea, splitting one hot partition into several smaller ones, is one of the most common fixes for hot-partition problems in distributed systems. Here we apply the same idea to a single database row instead of a whole shard.

  A simple rule for merging the last few buckets keeps this fragmentation problem bounded, without needing a full rebalance every time:

  ```
  on reservation_rejected(listing_id, bucket_id):
      total_remaining = SUM(on_hand across all buckets for listing_id)
      if total_remaining > 0 and total_remaining < TAIL_THRESHOLD:
          # collapse the few remaining buckets into one, atomically,
          # so the last units aren't invisible behind a false negative
          merge_buckets(listing_id) -> single_bucket
      # else: genuine sellout, reject is correct
  ```

  Here `TAIL_THRESHOLD` might be 5% of the original `on_hand`, or a small fixed number like 20 units. This value limits how long we accept fragmentation risk before we trade it back for full correctness, at the cost of some single-row contention again. This trade-off is acceptable at this point, because volume naturally drops as the product nears sellout.

### Mitigation 2: Queueing or serializing requests per product

Instead of letting all 8,000 requests per second hit the database at once, we route every request for that `listing_id` through a **single queue or single-threaded worker** (for example, a Kafka topic partitioned by `listing_id`, consumed by exactly one worker for that key, or an in-process worker holding the hot counter in memory). Requests are then processed strictly one at a time, in the order they arrive. This worker holds the correct in-memory count.

- **Effect:** Contention is now solved by waiting in a queue, not by rejected database connections. There are no lock waits and no connection-pool pressure. Throughput is limited only by how fast the single worker can process one decrement, which is very fast in memory, easily tens of thousands per second.
- **Trade-off:** This intentionally creates one single point where everything is processed in order. If this one worker or partition falls behind, everyone waiting behind it also waits longer. We need backpressure, meaning we reject new requests once the queue gets too deep, instead of letting wait times grow without limit. This way buyers get a fast "no" instead of a slow timeout. This method is really method (c) from Section 3, generalized into an explicit worker model, instead of relying on Redis's built-in single-threaded execution.

### Mitigation 3: Admission control, or a virtual waiting room

Before any request even reaches the Inventory Service, an **edge-level gate** (this could be the API Gateway from Chapter 5, or a dedicated waiting-room service) limits how many buyers are let through to attempt a reservation, in any given time window. The rest wait in a queue, or are given a random position, with a message like "you are number 14,532 in line."

- **Effect:** This is the only mitigation that reduces load *before* it reaches any backend service at all. The Inventory Service, the database shard, and even the Checkout Orchestrator never see the full 8,000 requests per second. They only see whatever rate the waiting room decides the backend can handle, for example, releasing 1,000 requests per second, matching the real rate at which the 5,000 units are actually being sold.
- **Trade-off:** There is a real user-experience cost, since buyers must explicitly wait. This feels worse than an instant rejection. But it is much better than a slow, hung request that eventually times out and looks broken. This is why virtual waiting rooms are the standard approach for concert-ticket sales and sneaker-drop platforms. For products that are genuinely scarce and where demand is predictable, honest queueing beats a fake sense of real-time competition.

### Mitigation 4: In-memory counter in front of the database, with fast persistence

This generalizes the Redis-gate pattern from Section 3(c). We keep the authoritative decision counter in memory, either in Redis or an in-process cache warmed up from the database. This counter answers admit-or-reject decisions in microseconds. We still write the durable reservation row, but we write it **as soon as possible after admitting the request, and always before telling the buyer it succeeded**. In other words, "fast" here means "off the lock-contended database row," not "eventually, whenever we get to it." A truly fire-and-forget approach, where the durable record is written later with no guarantee, would break the CP (strong consistency) guarantee explained in Section 8. So GlobalMart never uses that approach for inventory.

- **Effect:** The database is touched exactly once per admitted reservation. That means 5,000 durable writes total for this sale, not 8,000 writes per second for the whole duration of the sale. The roughly 3,000 requests per second that get rejected after stock hits zero never reach the database at all, once the in-memory counter reads zero.
- **Trade-off:** The in-memory counter must be seeded with the correct starting value when the sale begins, and it must be the actual, real source used for the admit-or-reject decision, not a cache that might return a stale answer. This is really the same Redis atomic-decrement pattern from Section 3. Its correctness depends on Redis's single-threaded execution, plus a durable backup write, exactly as discussed there.

### Putting the four mitigations together

| Mitigation | How it reduces contention | Best for | Weakness |
|---|---|---|---|
| Stock splitting (sharding a SKU) | Spreads writes across N rows | Predictable, high-volume flash sales | Stock fragmentation near depletion |
| Per-product queueing | Serializes requests without lock contention | Any hot key, found reactively | Needs backpressure or a shedding policy |
| Virtual waiting room | Caps arrival rate at the edge | Scheduled drops with known scarcity | User-experience cost of explicit waiting |
| In-memory counter + fast persist | Removes rejected attempts from the database entirely | Very short, very hot spikes | Counter must be seeded correctly and backed durably |

GlobalMart's flash-sale playbook, covered in more depth in Chapter 9, combines all four mitigations. A **waiting room** at the edge limits the incoming rate to something the backend can handle safely. Requests that get through hit a **Redis-fronted, sharded (stock-split) counter**. This counter's decrements are the real admit-or-reject decision. A durable reservation row is written right away, but only for each *admitted* request. So the database never sees the full 8,000 requests per second. It only sees the much smaller number of requests that were actually admitted.

## 5. Oversell-and-Apologize vs. Reserve-Strictly

Everything so far in this chapter assumes GlobalMart never oversells. This is the right default choice for a marketplace. But it is a deliberate business decision, not a fixed law of computer science. It is worth naming the alternative approach directly, because interviewers often test exactly this trade-off.

| | **Reserve-strictly** (GlobalMart's default) | **Oversell-and-apologize** |
|---|---|---|
| How it works | A hard reservation check runs before payment; we reject if unavailable | We accept the order optimistically first, then check real stock later. If it turns out oversold, we cancel and refund the loser |
| Buyer experience | Buyer gets an immediate, honest "out of stock" message, never a false success | Buyer gets an order confirmation that might later be cancelled. This is worse when it happens, but it never happens as long as demand stays under 100% of stock |
| Throughput under contention | Limited by the concurrency-control method (Sections 3 and 4) | No contention at all when committing the order, because the check happens off the hot path entirely |
| Where it is used | Marketplace products in general, and always for scarce, high-value, or seller-owned inventory. A marketplace has a contractual duty to sellers, and it should never sell stock that is not actually theirs to sell | High-volume, low-margin, statistically predictable inventory. Examples include grocery or quick-commerce items, where warehouse counts naturally have some slack, or digital/on-demand inventory, where restocking a losing buyer is trivial |
| Compensation cost | Close to zero, since rejections happen before anything is committed | Needs the full saga compensation path: void the authorization, cancel the order, refund it if payment was already captured, and often a goodwill gesture like a credit or discount to keep the buyer's trust |
| Failure blast radius | Limited to individual reservation attempts | Can spike unpredictably. For example, a viral social media post could drive 10,000 orders against only 200 real units. This becomes a support and public relations problem, not just a technical one |

GlobalMart's default choice is reserve-strictly, and this matches the reason stated in the original design brief. GlobalMart is a **multi-seller marketplace**, and the inventory belongs to sellers, not to GlobalMart itself. Overselling a seller's last unit is not just a bad experience for the buyer. It means GlobalMart is making a promise on behalf of a seller who never agreed to keep that promise.

Retailers who own their own supply chain, and who can absorb overselling as a controlled and budgeted cost with automatic compensation, sometimes choose oversell-and-apologize on purpose. They accept a rare, bounded bad experience in exchange for removing inventory contention from the checkout hot path entirely. Neither approach is simply "correct" or "incorrect." They represent a trade between latency and complexity on one side, and trust and compensation cost on the other. The right choice depends on who actually owns the inventory, and how expensive a broken promise is for that business.

## 6. Releasing Stranded Reservations and Fixing Drift

A HELD reservation that never resolves is called stranded stock. This is real inventory that a real buyer cannot get access to, because it is stuck holding a place for someone who is never coming back. There are three sources of stranded reservations, and each one is closed in a different way.

1. **Buyer abandonment.** The buyer closes the browser tab in the middle of checkout. No explicit release signal is ever sent, because nothing tells the Inventory Service that the buyer left. **Only the TTL sweeper** (from Section 2) can recover this stock. This is exactly why the TTL exists, and why it is a bounded time of about 15 minutes, instead of letting a hold last forever.
2. **Saga failure with a working cleanup step.** This happens when payment gets declined, or when a sibling sub-order in a multi-seller cart fails. The Checkout Orchestrator's compensation step calls `release` directly on the reservation, flipping it to RELEASED immediately, with no need to wait for the TTL. This is the normal, well-behaved path, and it should account for most releases.
3. **Saga failure with a broken cleanup step.** This happens if the orchestrator's server crashes in the middle of running its cleanup, or if the release call itself times out and is never retried correctly. This is why the TTL sweeper is not just a convenience for cleanup case 1. It is a **required safety net** for case 3 as well. Every HELD reservation will eventually be reclaimed, even if the orchestrator that created it never sends a follow-up signal at all.

### Fixing drift between the fast counter and the real record

In the recommended layered approach from Section 3, the `reserved` count is kept in two places at once: a fast in-memory or Redis counter, and a durable sum calculated from the reservation ledger table. These two numbers can drift apart from each other. Causes include a Redis node restart, a missed decrement during a release, or a bug in the code. This can leave the fast counter disagreeing with the real truth, which is the sum of all currently-HELD rows in the durable ledger.

A periodic reconciliation job closes this gap. Reconciliation means regularly comparing two records and fixing any differences. This follows the same idea as Chapter 8's payment reconciliation against the PSP (payment service provider).

```sql
-- Ground truth: sum of currently-HELD reservations per listing
SELECT listing_id, SUM(qty) AS true_reserved
FROM inventory_reservations
WHERE state = 'HELD'
GROUP BY listing_id;
```

The job compares this `true_reserved` value against the live counter, whether that counter lives in Redis or in the `reserved` column of the `inventory` row, for each listing. Wherever they disagree, the job **always corrects the fast counter to match the ledger**, never the other way around. The append-only reservation ledger, with its clear state transitions, is the real durable source of truth. Any in-memory or cached counter is just a performance shortcut built on top of it, not an independent fact of its own. This matches the general reconciliation principle from the design brief: fast paths and caches are allowed to drift briefly under load, but a background job must always be the backstop that turns temporary drift back into an accurate record.

**A worked example.** Say a flash-sale listing's Redis counter shows `available = 0` at the end of a sale. But a node failover during the sale lost the last few in-flight decrements before they could be copied to other nodes. The durable ledger shows that 5,012 units were actually HELD or COMMITTED against an `on_hand` of only 5,000. That is a 12-unit overshoot that the fast counter never saw, because those 12 reservations' durable rows were saved, but their matching Redis decrements did not survive the failover.

The reconciliation job's query, `SUM(qty) WHERE state IN ('HELD','COMMITTED')`, reveals the true number of 5,012 and flags this 12-unit breach for review, either manual or automated. At this point it becomes a business decision, following the ideas in Section 5, not a purely technical one. GlobalMart can cancel the 12 lowest-priority orders, such as the latest ones placed, and compensate those buyers. Or it can choose to absorb this small overshoot as a rare, bounded cost. Either way, the reconciliation job's role is to **detect and surface this kind of drift fast**, usually within a minute of the sale ending, rather than waiting for an end-of-day batch job. This speed matters because inventory drift quietly grows worse if left unchecked, unlike something like an analytics count, which can safely lag behind by hours with no real harm.

## 7. Multi-Warehouse and Multi-Region Inventory

So far, this chapter has treated a listing's stock as just one single number. In reality, a seller's stock for one `listing_id` can be split across several **fulfillment warehouses**. GlobalMart itself also runs across multiple regions, which Chapter 9 covers in more depth. Two questions matter here.

```
                    seller's total on_hand for listing_id = 240
                                 │
                     ┌───────────┴───────────┐
                     │      Inventory DB      │   ← single shard, single
                     │  listing_id → on_hand   │     source of truth,
                     │  = 240 (aggregate)      │     reservation logic
                     └───────────┬───────────┘     from §3 applies here
                                 │  (fulfillment routing happens
                                 │   AFTER commit, not before)
              ┌──────────────────┼──────────────────┐
              ▼                  ▼                  ▼
     Warehouse A (US-East) Warehouse B (US-West) Warehouse C (EU)
        physical: 100         physical: 80          physical: 60
```

**Where does the real record live?** Following the canonical architecture, the **Inventory DB**, sharded by `listing_id`/SKU, is the single source of truth for a listing's total stock and its reservations. This is true globally, not separately per warehouse and not separately per region. A `listing_id` always maps to exactly one shard, following Chapter 2's canonical sharding rule. So the question "how much is available" and the question "who currently holds a reservation" both always have exactly one correct answer, computed against one shard. This is true no matter how many physical warehouses stand behind that stock, and no matter which region the buyer is checking out from.

**How does splitting stock across warehouses work, if there is still just one source of truth?** There are two workable models.

- **Aggregate-then-allocate, which is GlobalMart's default model.** The Inventory DB keeps a single `on_hand` number per `listing_id`. This is the seller's total stock across all their warehouses for that listing. All the reservation and commit logic from Section 3 works against this one combined number, exactly as described earlier in this chapter. Deciding *which* physical warehouse actually fulfills a committed order is a separate, downstream fulfillment decision. This decision is outside the scope of this chapter, per the design brief. It is typically made after the order commits, often based on which warehouse is closest to the shipping address, and it is not something the reservation step itself needs to worry about. This keeps the concurrency-control problem exactly as simple as everywhere else in this chapter: one row, or split sub-buckets of one row as covered in Section 4, and one rule to enforce, in one place.
- **Per-warehouse reservation.** Some sellers explicitly want warehouse-level guarantees. For example, a listing might only be fulfillable from Warehouse A, and Warehouse A's stock must never be counted twice against Warehouse B's stock. For these sellers, the Inventory DB instead keeps one row per `(listing_id, warehouse_id)` combination. The reservation step then first picks a specific warehouse row, either the nearest one to the buyer, or one chosen by the seller's own routing rules, and reserves against that specific row. This adds an extra warehouse-choice step before the same Section 3 mechanics apply to that one row. It can also *increase* the risk of false rejections from stock fragmentation, similar to the trade-off in Section 4's stock-splitting method, except here the fragmentation is forced by business rules rather than chosen for performance reasons. A buyer might get rejected against a nearby but empty warehouse, while a farther warehouse still has stock, unless the reservation logic is built to explicitly check other warehouses before rejecting.

**Reads can cross regions, but writes cannot.** No matter which of the two models above is used, the *write* path for a reservation, meaning the actual admit-or-reject decision, is always pinned to the one region that owns the shard for that `listing_id`. This rule is non-negotiable, given the CP (strong consistency) stance explained in Section 8. You cannot correctly check `available >= qty` against two copies of the same counter in two different regions that can both be written to independently, unless you add a coordination protocol. And adding such a protocol brings back exactly the latency cost this whole chapter is trying to avoid.

Buyers located in other regions do pay the cost of a cross-region round trip, specifically to reach the region that owns the shard, just for the reservation step. Meanwhile, everything else in their checkout session, such as pricing display, cart contents, and other non-inventory reads, can be served from a local, eventually consistent replica in their own region. Chapter 9 covers in detail how GlobalMart's multi-region active-active setup routes this particular write to the correct region, and what happens during a regional failover of an inventory shard. The short version to remember here is this: **inventory writes are always single-region and strongly consistent, by design. Multi-region setup only changes how far the write has to travel. It never changes whether concurrent writes are allowed to race against each other.**

## 8. Why Inventory Needs Strong Consistency (CP)

The design brief states this stance plainly: inventory reservation is strongly consistent on purpose. In CAP terms, this makes it a CP system, meaning it chooses Consistency over Availability during a network problem. This is a **direct contrast to the companion Search handbook**, which deliberately chose AP, meaning Availability over Consistency, for search and catalog reads.

It is worth explaining clearly *why* these two systems land on opposite sides of the same trade-off. "When do you choose CP versus AP" is one of the most common system design interview questions, and comparing inventory against search is the cleanest real example to use.

**What would break under eventual consistency.** Imagine two reservation attempts for the last unit were allowed to be checked against two database replicas that had not yet synced with each other. This is exactly the normal, accepted behavior of any AP system. Both replicas could independently see `available = 1` and both could admit the request, simply because neither replica has learned about the other one's decrement yet.

At GlobalMart's scale, this is not some rare edge case. It is the *default* outcome for any contended product under eventual consistency, and it is exactly the oversell scenario described back in Section 1.

Now compare this to a stale search result. A buyer sees a listing that has actually already sold out. They click it, and only then find out it is gone. This is annoying, but the truth reaches the buyer just one click later, at basically zero cost, because search reads do not commit any money or make any promises. But an oversold unit only reveals the truth *after* GlobalMart has already told a seller "sold" and a buyer "confirmed." This turns a harmless moment of staleness into a real compensation process: voiding an authorization, cancelling a confirmed order, refunding a payment that may already have been captured, and apologizing to a buyer who had planned around a delivery that is now not coming.

Every single mitigation covered in Sections 3 and 4 is, at its core, an exercise in making that one inequality check atomic, *without ever relaxing it*. None of the hot-product mitigations trade away correctness for more throughput. They only change *where and how much* contention happens, while keeping the final admit-or-reject decision exactly as strict as a single global lock would give us.

**Why search could afford AP, and checkout cannot.** Search's core job is "read the best available snapshot of the catalog, quickly, everywhere." Being wrong for a few seconds after a catalog update is invisible to the buyer and costs nothing. Checkout's core job over inventory is different: "decide, exactly once, who gets the last unit." Being wrong here is not just a small staleness window. It becomes a real double-sold unit, with a real seller and a real buyer standing on each side of the mistake.

This is the same recurring idea from the design brief, applied at the scale of a whole chapter: use strong consistency specifically where money and inventory are involved, and allow eventual consistency everywhere else, such as order history, dashboards, analytics, and recommendations. Correctness always has a real cost: throughput limits, cross-region latency, and engineering complexity, as shown throughout Sections 3 and 4. GlobalMart chooses to pay that cost *only* on the narrow slice of the system where being wrong is genuinely expensive. Every other read-heavy, non-transactional part of checkout is allowed to be eventually consistent and fast.

## Interview Tips

- **Start with the race condition, not the fix.** Clearly describe the TOCTOU bug first: two reads of `available`, then two writes, both succeeding. Only after that, name your fix. This shows you understand *why* the naive check-then-act approach fails, not just that some fix happens to exist.
- **Naming optimistic versus pessimistic concurrency control, by name, is a strong signal.** Be ready to write out the SQL for both approaches: `SELECT ... FOR UPDATE` versus `UPDATE ... WHERE on_hand - reserved - sold >= qty`. Be able to explain clearly why the optimistic approach wins on throughput, since it never holds a lock across a network round trip, while the pessimistic approach wins on simplicity for rows with low contention. Most candidates can name both approaches. Fewer candidates can explain *why* one is actually faster than the other, in terms of how long each one holds a lock.
- **The hot-row or hot-SKU fix is the other strong signal to give.** If asked "what if this product suddenly gets 100 times more traffic," do not just answer "cache it." Name a specific mechanism instead: stock splitting, per-key queueing, a waiting room, or a Redis-fronted counter with a durable synchronous backup write. Be explicit about its trade-off too, such as stock fragmentation, queue backpressure, user-experience cost, or the need for correct counter seeding. Naming the trade-off clearly is what separates a senior-level answer from a buzzword answer.
- **Be ready to defend reserve-strictly versus oversell-and-apologize as a business decision**, not as a fixed technical rule. Interviewers sometimes push back with "why not just let it oversell and fix it afterward?" to test whether you can explain the seller-trust and compensation-cost argument from Section 5, instead of just defaulting to "strict is always the right answer."
- **Connect this chapter's CP stance back to the Search handbook's AP stance**, if the interviewer is testing your understanding of the CAP theorem. The strongest answer is the one from Section 8: staleness costs nothing on a read path, but it triggers a full compensation saga on a commit path. This asymmetry between the two, not some general universal rule, is what actually decides CP versus AP for each different subsystem.
- **Do not forget about the "losing" buyer's experience.** A great answer covers not only how the winning buyer gets the unit, but also how the losing buyer gets a fast, honest rejection, instead of a hang or a silent 15-minute wait for the TTL to expire. Interviewers listen carefully for whether you have thought through the unhappy path, not only the happy one.
- **If asked about multi-item carts or multi-warehouse routing, do not overcomplicate the reservation step.** The strongest answer keeps the concurrency-critical rule, `available >= qty`, as a single-row problem against one combined number. Push all warehouse and fulfillment routing decisions to a separate step that happens *after* the commit, and that is not concurrency-critical at all. Mixing the two together is a common way candidates accidentally make this problem harder than it needs to be.

## Key Takeaways

- Inventory correctness comes down to one atomic inequality check: `available = on_hand − reserved − sold >= qty`, checked and acted on as one indivisible step. Every concurrency strategy in this chapter is really just a different way of making that check-and-act step atomic.
- A reservation is a three-state machine: **HELD (TTL about 15 minutes) → COMMITTED or RELEASED**. It is not a simple true/false flag. The TTL and its sweeper job guarantee that stranded holds, such as abandoned checkouts or crashed sagas, are always eventually reclaimed, even with no explicit release call.
- Four concurrency mechanisms exist: pessimistic row locks, optimistic conditional updates, atomic Redis decrements, and reservation-row ledgers. Each one trades off latency, hot-row throughput, and audit-trail quality differently. GlobalMart's answer layers them together: optimistic conditional updates as the durable default, backed by a reservation-row audit ledger, and fronted by a Redis atomic counter specifically for products identified as hot.
- Flash-sale contention concentrates thousands of requests per second onto a single row on a single shard. The fix is never "make the row faster." It is always **reducing contention before it ever reaches the row**, using stock splitting, per-product queueing, admission control or waiting rooms, and in-memory counters backed by a synchronous durable write.
- Reserve-strictly, which is GlobalMart's default and fits marketplace or seller-owned inventory, and oversell-and-apologize, which fits owned, high-slack, or digital inventory, are both legitimate designs. The choice trades checkout-time complexity against after-the-fact compensation cost and seller or buyer trust.
- Drift between the fast counter and the durable ledger is expected under heavy load. It is closed by periodic reconciliation, which always corrects the fast path *toward* the durable append-only ledger, and never the other way around.
- Multi-warehouse stock is usually combined into one authoritative number per `listing_id`, so the reservation rule stays a single-row problem. Multi-region topology only changes how far a reservation write has to travel. It never changes whether concurrent writes are allowed to race against each other (see Chapter 9).
- Inventory is a deliberate CP domain, because staleness here is not a harmless delay before the truth catches up, unlike in search, which is Chapter 8's companion AP system. Here, staleness becomes an oversold unit that has already been promised to two different people, and it now requires a real compensation saga to undo.
