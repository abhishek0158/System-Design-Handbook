# Chapter 2 — Capacity Estimation

Chapter 1 set the scope. GlobalMart Checkout must never double-charge a buyer. It must never
oversell a seller's stock. It must stay available 99.99% of the time. And it must survive
flash-sale spikes that are 10 to 20 times normal traffic.

This chapter turns that scope into numbers. How many requests per second? How much data do we
store? How much memory do we need? How many database shards? Later chapters use these numbers
directly. Chapter 7 uses the shard count. Chapter 8 uses the connection-pool sizing. Chapter 9
uses the surge numbers. If the math in this chapter is right, the rest of the handbook stands on
solid ground. If it is wrong, every later design decision is built on sand.

## 2.1 Why Estimation Matters, and How to Present It

In an interview, and in a real design review, capacity estimation does three jobs.

**First, it finds the bottleneck before you build anything.** Suppose a quick calculation shows
that one database row must handle 10,000 writes per second at peak. Now you know, before writing
any code, that a simple command like `UPDATE inventory SET qty = qty - 1 WHERE sku = ?` will fail
under that load. This one insight drives the whole design of Chapter 7.

**Second, it sizes the cost.** Needing 90 TB of fast storage and 1,024 database shards are real
purchasing decisions, not just interesting facts. A design that works but needs 50 times more
hardware than it needs is not a good design.

**Third, it sets the frame for the rest of the conversation.** Once everyone agrees that peak
load is about 50,000 orders per second, that checkout API peak is about 300,000 requests per
second, and that payment authorization takes 300 to 1500 milliseconds, every later design choice
can point back to these numbers. This is better than guessing.

Here are simple rules for doing this well.

- **Round at the end, but show the exact math first.** Compute 200,000,000 divided by 86,400. That
  gives 2,314.8. Then say "call it about 2,300 per second." The precise number lives in your
  working, not in the final headline.
- **State every assumption out loud, in one place.** Say clearly: "I assume peak retry overhead is
  1.3 to 1.5 times the average." Do not hide this kind of guess inside a formula.
- **Build new numbers from the canonical numbers.** Every number in this chapter either comes
  straight from the design brief's scale table, or is built by combining two or more of those
  numbers. If a number you build does not match a canonical number (see the checkout-API check in
  §2.2), that is a signal to recheck your math, not to ignore it.
- **Use Little's Law whenever both a rate and a duration matter.** The rule is: concurrency equals
  arrival rate multiplied by time spent in the system. This chapter uses this rule three times: for
  payment calls, for inventory holds, and, in a hidden way, for every open HTTP request. It is the
  single most useful tool in this whole handbook.
- **Always separate the average number from the peak number.** A system sized for 2,300 orders per
  second will completely fail during a 50,000-per-second flash sale. That is not "slightly
  short." That is a 22-times overload. So give both numbers, every time.

## 2.2 Traffic: From Daily Active Users to Peak Checkout API Load

### 2.2.1 The order funnel

The design brief gives us GlobalMart's scale. These are the starting numbers for everything else
in this chapter.

| Quantity | Value |
|---|---|
| Registered users | 2 billion |
| Daily active users (DAU) | 500 million |
| Orders per day | about 200 million |
| Checkout sessions per day | about 600 million |
| Conversion (sessions that become orders) | about 33% |
| Average items per order | about 3 (so about 600 million line items per day) |
| Checkout API calls per session | about 6 |

A "checkout session" is one shopping trip through checkout. It may or may not end in a paid
order.

**Step 1: average order-placement rate.**

```
200,000,000 orders/day ÷ 86,400 sec/day = 2,314.8 orders/sec  ≈ ~2,300 orders/sec
```

This matches the brief's canonical number exactly. If someone asks "how many orders does
GlobalMart place per second on a normal day," this is the answer.

**Step 2: average checkout-session rate.**

```
600,000,000 sessions/day ÷ 86,400 sec/day = 6,944.4 sessions/sec  ≈ ~7,000 sessions/sec
```

**Step 3: check the conversion rate.** About 1 in 3 sessions becomes a real order.

```
200M orders / 600M sessions = 0.333 → ~33% conversion (about 67% of sessions are abandoned)
```

This 33% number matters a lot. It is the reason checkout cannot be sized the same simple way as a
search system. Two out of three sessions never reach the final "place order" step. Buyers look at
shipping costs, compare totals, and then leave. As we will see next, this abandonment behavior
changes sharply during a flash sale.

**Step 4: peak order-placement rate.**

```
Peak = ~20× average = 2,300 × 20 ≈ 46,000–50,000 orders/sec  → canonical: ~50K orders/sec
```

Why 20 times? This is not a random guess. Real large marketplaces see traffic land unevenly
during flash sales and Black Friday events. Demand does not spread evenly across the day. It
crashes into a short 10-to-30-minute window right when a big deal or midnight sale starts.

### 2.2.2 Average checkout-API load

Each checkout session makes about 6 API calls on average. These are: create session, set
shipping, get a priced quote, attach a payment method, review, and place the order. (Chapter 3
lists these exact calls.) Using the average session rate:

```
7,000 sessions/sec × 6 calls/session = 42,000 checkout-API calls/sec (average)
```

Out of these six calls, only one matters for money and inventory correctness: "place order." The
other five calls are cheap reads or simple writes on session data. So the average rate of "place
order" attempts should roughly match the average order rate:

```
Average place-order attempts/sec ≈ 2,300–2,500/sec (successful orders plus a small retry margin)
```

We explain the retry margin in §2.3.

### 2.2.3 Peak checkout API load, and why it is not simply 20 times bigger

You might guess that peak API load is 20 times the average: 20 × 42,000 = 840,000 requests per
second. But the brief's canonical peak number is only about 300,000 requests per second. This is
lower than that naive guess. Let's work out why. This is the most important idea in this section.

```
Peak checkout-API load ≈ peak order-placement rate × calls per session
                       = 50,000/sec × 6
                       = 300,000 QPS
```

This math lands exactly on the canonical number. But notice what we multiplied: peak order rate,
not peak session rate. This is deliberate, and it reflects something real about flash-sale
traffic.

- On a normal day, most of the 600 million sessions are casual browsers. They open a session,
  check shipping costs, and leave (the 67% who abandon). Most calls during normal traffic are the
  cheap, early-funnel calls.
- During a flash sale, the funnel gets much shorter. Buyers arrive already committed. They raced
  to a landing page for a doorbuster deal. Very few of them abandon. A large share of the traffic
  becomes "check session status" calls and "place order" calls. It also becomes many repeated
  retries of "place order," because a buyer whose order fails (someone else got the item) taps
  "Buy Now" again and again. We look closely at this retry pattern in §2.6.
- So at peak, the ratio of API calls to successful orders stays close to 6-to-1. But the number you
  multiply by changes. It shifts from "session rate," which does not spike much (browsing traffic
  stays fairly flat), to "order-attempt rate," which spikes hard because it concentrates on one
  API call and a small number of hot products.

**The line to say in an interview:** flash-sale peaks are spikier than normal traffic, not because
everyone browses 20 times more. It is because a small number of hot products, and a single API
call ("place order"), absorb almost all of the extra load. This is why Chapter 9's scaling story
is about protecting one hot path carefully, not just adding more servers everywhere.

### 2.2.4 Traffic summary

| Metric | Average | Peak | Peak divided by average |
|---|---|---|---|
| Order-placement rate | ~2,300/sec | ~50,000/sec | ~20× |
| Checkout-session rate | ~7,000/sec | spikes less than orders (see above) | lower than 20× |
| Checkout-API QPS | ~42,000/sec | ~300,000/sec | ~7× |
| "Place order" calls/sec | ~2,300–2,500/sec | tens of thousands (orders plus retries) | ~20×+ |

## 2.3 Payment Throughput and Little's Law

### 2.3.1 How many payment transactions per day

The brief states payment transactions per day at about 220 million, against about 200 million
orders per day. Where does the extra 20 million come from? It comes from authorization retries.
A declined card gets retried with a different card. A slow PSP (payment service provider, the
outside company that actually processes the card) times out and gets retried safely. A split
payment (part gift card, part credit card) needs more than one authorization call for a single
order.

```
Retry/split overhead factor = 220M / 200M = 1.10  (about 10% more transactions than orders)

Average payment-transaction rate = 220,000,000 / 86,400 ≈ 2,546/sec  ≈ ~2,550/sec
```

**What happens at peak?** The 10% overhead is a daily average number. Under flash-sale load, this
overhead likely grows, for two reasons. First, the PSP itself is handling surge traffic from many
other merchants at the same time, so its own response time gets worse, which causes more of our
timeout-based retries. Second, fraud-detection systems that were tuned for normal buying speed
tend to flag more false declines during fast, bursty buying. This is a stated assumption, not a
canonical number. Let's assume the peak overhead factor rises to 1.3 to 1.5 times normal.

```
Peak payment-transaction rate ≈ 50,000/sec × 1.3–1.5 ≈ 65,000–75,000 transactions/sec
```

### 2.3.2 Why "transactions per second" is the wrong number to build for

Transactions per second only tells you the arrival rate. It does not tell you the number that
actually decides how big your infrastructure needs to be: how many authorization calls are open
and waiting on the PSP at the same instant. That number depends on both the arrival rate and how
long each call takes to finish. According to the brief's latency budget, PSP authorization takes
300 to 1500 milliseconds. This is by far the biggest and most unpredictable part of the whole 2.5
second "place order" time budget.

**Little's Law** is a simple and powerful rule from queuing theory. It says: in a steady system,
`L = λ × W`. Here `L` is the average number of items currently in the system. `λ` (the Greek
letter lambda) is the arrival rate. `W` is the average time each item spends in the system.

Applied here, this means: the number of authorization calls that are open and unfinished at any
moment equals the arrival rate of authorization requests, multiplied by the PSP's average response
time.

```
L (concurrent in-flight authorizations) = λ (auth requests/sec) × W (PSP latency, sec)
```

**At average load:**

```
Best case  (W = 0.3s):  L = 2,550 × 0.3 = 765 concurrent in-flight authorizations
Worst case (W = 1.5s):  L = 2,550 × 1.5 = 3,825 concurrent in-flight authorizations
```

**At peak load** (using the midpoint of the 65,000 to 75,000 range, about 70,000 per second):

```
Best case  (W = 0.3s):  L = 70,000 × 0.3  ≈ 21,000 concurrent in-flight authorizations
Worst case (W = 1.5s):  L = 70,000 × 1.5  ≈ 105,000 concurrent in-flight authorizations
```

If you use the full range of 65,000 to 75,000 transactions per second, the answer spans about
19,500 to 112,500 concurrent calls.

This is a huge gap. The gap between "requests per second" (70,000) and "how many calls you must
support at once" (up to over 100,000) is two orders of magnitude apart in feel. This gap exists
purely because of the PSP's own slow response time, something GlobalMart does not control and
cannot make faster.

### 2.3.3 Turning this concurrency number into an infrastructure decision

This is where the math stops being just numbers and starts deciding how we build the system.

**A thread-per-request design cannot handle this many open calls.** Holding over 100,000 blocked
program threads at once, each one waiting up to 1.5 seconds for a socket to respond, is too heavy
for almost any normal program design. This is exactly why the Payment Service and the PSP Adapters
(described in Chapter 5) must use async, non-blocking input/output. In simple words: instead of
one thread waiting per call, the program uses a small pool of threads that can each juggle
thousands of waiting calls at once, because "waiting" here just means a parked task, not a
full occupied thread.

**The size of the connection pool follows directly from this L number.** Suppose one PSP Adapter
program instance can safely hold about 2,000 open outbound connections at once. Then, at the
worst-case peak of 112,500 concurrent calls, we need:

```
112,500 concurrent connections / 2,000 per instance ≈ 57 instances (minimum, connection-bound)
```

In real life, you would run more than this bare minimum. You need spare capacity for CPU-heavy
work, for retries, and in case one PSP fails and its traffic moves to another PSP. So a real
deployment would likely run over 100 PSP Adapter instances spread across regions, not the small
handful you might guess just from looking at "70,000 requests per second."

**The connection pool must be split per PSP, not shared globally.** GlobalMart connects to
several PSPs, covering different card networks, digital wallets, and regional payment methods. If
one PSP slows down to 5-second response times, Little's Law says that PSP's share of `L` grows 3
to 15 times larger. Without a separate connection pool and a circuit breaker (explained in Chapter
10) for each PSP, one slow PSP could eat up the entire payment concurrency budget. That would break
    authorization for every PSP, not just the slow one.

**This is one of the strongest signals you can give in an estimation interview.** Anyone can
divide a number of requests by seconds. But noticing that duration multiplies the arrival rate
into a much bigger concurrency number, and that this concurrency number (not the raw request rate)
is what actually decides connection-pool size and server count, is what makes an estimate feel
senior rather than junior.

## 2.4 Storage: Orders, Retention, and Idempotency

### 2.4.1 Order data

An "order record" stores the line items, addresses, payment references, status history, and the
per-seller sub-orders for one order. It is estimated at about 5 KB.

```
Order record size          ≈ 5 KB
Daily order data           = 200,000,000 orders/day × 5 KB
                           = 1,000,000,000 KB/day
                           = 1,000,000 MB/day
                           = 1 TB/day

Annual order data (raw)    = 1 TB/day × 365 days ≈ 365 TB/year
```

### 2.4.2 Retention and tiering

Financial and legal rules require GlobalMart to keep order records for 7 years.

```
7-year raw order data = 365 TB/year × 7 ≈ 2,555 TB ≈ 2.56 PB (raw, single copy)
```

A "PB," or petabyte, equals 1,000 terabytes. This raw number is not what actually sits on disk.
Real storage systems keep extra copies for safety. A common, conservative choice for a strongly
consistent, transactional store is 3 copies (called 3x replication).

```
7-year footprint with 3× replication ≈ 2.56 PB × 3 ≈ ~7.7 PB
```

This lands comfortably in the "multi-petabyte cold storage" range that the brief expects. In
practice, older data (from year 2 through year 7) is rarely read. So instead of keeping 3 full
copies, engineers often use a technique called erasure coding, which needs only about 1.4 to 1.5
times the raw data size, instead of 3 times, while still keeping strong safety guarantees. Using
that method, the real disk footprint could drop to about 3.5 to 4 PB. Either way, this data is
petabyte scale, and it lives in cheap object storage or cold storage, not on the fast primary
database shards.

The **hot tier** is the part of the data actually served live, for order-status lookups, for
reconciliation checks (Chapter 8), and for active disputes. The brief sets this at the last 90
days.

```
Hot tier (raw) = 1 TB/day × 90 days = 90 TB     (matches the brief's canonical ~90 TB)
Hot tier (with 3× replication, on fast NVMe drives) ≈ 270 TB actually provisioned
```

### 2.4.3 Idempotency records

An "idempotency record" is a small saved record that lets the system safely handle a repeated
request without charging or reserving stock twice. Every payment transaction, not just every
successful order, needs one of these. Each record is about 1 KB in size (it holds a key, a
fingerprint of the request, a snapshot of the response, a state, and an expiry time).

```
Daily idempotency writes = 220,000,000 × 1 KB = 220,000,000 KB/day ≈ 220 GB/day
```

The time-to-live (TTL) for these records is 24 to 48 hours. This means the actual amount of data
stored at any one moment is not simply "one day's worth" (220 GB). It is the daily write rate
multiplied by how long, on average, a record stays alive before it expires. If we assume records
live an average of 30 to 42 hours (a blend across the 24-to-48-hour range, since not every key
sits right at the edge of expiry):

```
Steady-state idempotency store size ≈ 220 GB/day × (1.25 to 1.75 days live)
                                     ≈ 275 GB to 385 GB
```

This lands right inside the brief's canonical range of about 250 to 400 GB. This store lives in
Redis, an in-memory data store, so lookups happen in under a millisecond. This matters because
every "place order" call checks this store on its critical path. Redis also keeps a durable
backup copy, so a Redis restart does not accidentally forget which charges already happened. That
would be a correctness problem, not just a performance one.

### 2.4.4 Inventory reservations: kept small on purpose

An "inventory reservation" is a temporary hold on stock, made while a buyer is checking out. It is
deliberately not a long-lived, permanent record. Its TTL is only about 15 minutes. It exists only
to hold stock during the checkout window. We cover the sizing question for this store together
with the hot-product problem in §2.6 and §2.7. That's because the interesting number here is not
the total size of the data (which turns out to be small). It is the number of write attempts
hitting individual rows at once.

### 2.4.5 Storage summary

| Data | Size | Notes |
|---|---|---|
| Order data / day | ~1 TB | 200M orders × 5 KB |
| Order data / year (raw) | ~365 TB | |
| Order data / 7 years (raw) | ~2.56 PB | before replication |
| Order data / 7 years (replicated or coded) | ~3.5–7.7 PB | 3× replication vs. erasure coding |
| Hot tier (last 90 days, raw) | ~90 TB | served by the live Order DB |
| Hot tier (replicated) | ~270 TB | fast NVMe primaries |
| Idempotency store (steady state) | ~250–400 GB | Redis plus durable backup, TTL 24–48h |
| Inventory reservations | small (see §2.7) | temporary, TTL ~15 min |

## 2.5 Order Database Sizing: Why 1,024 Shards

The brief fixes the Order Database at about 1,024 shards. A "shard" is one piece of a database
that has been split into many smaller pieces, each holding a portion of the data. Sharding is done
by `hash(order_id)`, meaning each order's ID is passed through a hash function to decide which
shard stores it. This spreads orders evenly and randomly across all shards.

This number, 1,024, should come out of real math, not be picked at random. It turns out two
separate limits point to roughly the same answer.

### 2.5.1 Limit 1: how many writes one shard can handle

A single relational database shard, writing data safely to disk and keeping at least one
synchronized backup copy (needed for the strong consistency stance described in Chapter 6), can
reasonably handle a few hundred order-writes per second with room to spare. Let's assume, as a
careful working number, about 500 writes per second per shard.

```
Shards needed for peak throughput = 50,000 orders/sec ÷ 500 writes/sec/shard = 100 shards
```

So, from a pure throughput point of view, only 100 shards would be enough to handle peak load.
The actual number, 1,024, is about 10 times more than that floor. This tells us that raw
throughput is not the real reason for choosing 1,024.

### 2.5.2 Limit 2: keeping each shard a manageable size

The second way to think about sharding is operational. How large should one shard's live data be,
so that backups, re-indexing, failovers, and resharding all stay fast? Dividing the 90 TB hot tier
across 1,024 shards:

```
Per-shard hot data = 90 TB / 1,024 ≈ 87.9 GB/shard
```

About 88 GB per shard is a comfortable size. It backs up quickly, restores quickly, and fits
well on fast NVMe drives. This operational reason is what really explains 1,024. It is chosen for
data locality, for containing failures to a small blast radius, and for making the system easy to
operate, not for raw peak request handling. At peak, each shard only sees:

```
Per-shard peak write rate = 50,000 / 1,024 ≈ 48.8 orders/sec/shard
```

That is about 10 times less than even the careful 500-writes-per-second estimate for one shard.
This extra room is valuable on purpose. It means a handful of shards can go completely down during
a regional problem (see Chapters 9 and 10) without threatening the whole system's throughput. It
also means one unusually busy shard, perhaps one that happens to store many orders tied to a
viral product, has plenty of room before it becomes a bottleneck.

**The idea worth remembering for an interview:** in real systems, sharding decisions are often
driven more by operational needs (shard size, blast radius, how fast you can reshard) than by the
peak-request number alone. The peak-request number only sets a floor. The operational needs decide
how far above that floor you actually build. Also, 1,024 equals 2 to the power of 10. This is a
convenient number for consistent hashing and for resharding by repeatedly splitting shards in
half, since each split only needs to move half of the data out of each parent shard.

### 2.5.3 Hot and cold tiers, and why we shard by order ID

- **Hot tier:** all 1,024 shards keep roughly the most recent 90 days of data on fast NVMe drives
  with synchronized backup copies. This tier serves order-status reads, the checkout saga's own
  read and write path (Chapter 6), and near-real-time reconciliation checks (Chapter 8).
- **Cold tier:** data older than 90 days moves to cheaper object or column storage (the roughly
  2.56 PB raw, or 3.5 to 7.7 PB replicated, figure from §2.4.2). This tier is organized by time,
  not by order ID, because the typical access pattern here is audits, legal holds, and tax
  queries. Those need scans across a date range, not a lookup of one specific order.
- **Why we shard by `hash(order_id)` and not by `seller_id` or `buyer_id`:** hashing the order's
  own ID spreads the load evenly, no matter how popular any one seller or buyer is. A viral seller,
  or a buyer with an unusually large order history, does not create a single overloaded shard.
  There is a trade-off, though (we cover it fully in Chapter 6): a single checkout with items from
  N different sellers creates N sub-orders, and those N sub-orders can land on N completely
  different shards. This means the checkout saga cannot rely on one single-shard transaction to
  create all of a checkout group's orders at once. It instead needs separate, idempotent writes
  per sub-order, plus a shared `checkout_group_id` to tie them back together.

## 2.6 Inventory Hotspots: Measuring Hot-Row Contention

This is the sharpest problem in this whole chapter. It deserves a full numeric walk-through,
rather than just saying "hot products are a problem" and moving on.

**The setup:** imagine a flash sale drops one single product with, say, 10,000 units in stock, and
far more buyers want it than there is stock. The brief describes this exactly: "tens of thousands
of buyers race for the same SKU." A SKU (stock keeping unit) is the unique code for one specific
product. Unlike the Order Database, which spreads load evenly across 1,024 shards by order ID,
the Inventory Database is sharded by `listing_id` or `sku`. This means every single reservation
attempt for this one popular product hits the exact same shard, and usually the exact same row in
that shard.

**Measuring the attempt rate on that one row.** Even if this one hot product makes up only a
modest slice of total peak traffic, say 5 to 10% of the roughly 50,000 orders per second peak (this
is a stated, illustrative assumption for one viral doorbuster item), that alone gives:

```
Reservation attempts on one row ≈ 50,000/sec × 5–10% = 2,500–5,000 attempts/sec
```

And this is before counting retries. A buyer whose reservation attempt fails, either because
someone else grabbed the last unit, or because the row was too busy and the request timed out,
usually tries again right away, often several times within a few seconds. This is the exact same
"place order" retry pattern flagged earlier in §2.2.3. Let's assume a careful retry multiplier of 2
to 3 times under this kind of contention:

```
Effective attempt rate on one row ≈ 2,500–5,000 × 2–3 ≈ 5,000–15,000 attempts/sec
```

**Why one database row cannot handle this.** A simple, naive approach looks like this: read the
current quantity with a lock (`SELECT qty FROM inventory WHERE sku=? FOR UPDATE`), check if enough
stock remains, then write the new quantity (`UPDATE ... SET qty = qty - 1`). This approach forces
every single writer to wait in line for that one row's lock. Even a fast, NVMe-backed database
typically tops out at only a few thousand of these read-then-write cycles per second on a single
row. The reason is not disk speed. The reason is that holding a lock and committing a transaction
both take time, and both must happen one after another for a single row that many writers are
fighting over.

So we have 5,000 to 15,000 attempts per second landing on a resource that tops out at a few
thousand per second. This is not something you fix by adding more database servers elsewhere,
because adding shards somewhere else does nothing for this one specific hot row. The overload shows
up as growing queues, rising response times, timeouts, and then, because those timeouts trigger
even more retries, a self-feeding pile-up. This is the same kind of "retry storm" problem seen in
many distributed systems.

**Why this matters more than all the other numbers in this chapter combined.** Every other system
described in this chapter scales in a simple way: add more shards, more instances, or more
connections, and capacity grows roughly in proportion. A single hot inventory row does not scale
this way. No amount of adding hardware elsewhere touches it, because correctness (never overselling
that one product) requires that all decrements to that one product's count happen one at a time,
somewhere, no matter what. This exact problem is what Chapter 7 is built to solve, using
techniques like: atomic single-step decrements that avoid the risky read-then-write pattern
entirely; splitting one hot product's count into several smaller counters that get combined later;
queue-based ordering of contended requests; and, for the most extreme cases (covered in Chapter
9), routing all traffic for one hot product into a separate "virtual waiting room" lane, so the
contention stays bounded and visible instead of quietly dragging down the whole platform.

**The idea worth stating clearly in an interview:** a system can look completely healthy overall,
with plenty of request capacity, plenty of shards, and plenty of storage, while one single logical
row is badly overloaded by a full order of magnitude. Recognizing that this is a fundamentally
different kind of problem from "we don't have enough servers" is one of the highest-value moments
in a checkout system design interview.

## 2.7 Memory and Cache Sizing

### 2.7.1 Idempotency store (Redis)

From §2.4.3, the steady-state working set for idempotency records is about 250 to 400 GB.
Suppose we provision this as a Redis cluster with one primary copy plus two backup replicas per
shard, for safety and for spreading out read load from retry checks:

```
Provisioned Redis memory ≈ 300 GB (midpoint working set) × 3 (primary + 2 replicas)
                          ≈ ~900 GB total cluster RAM
```

With typical cluster machines offering 64 to 128 GB of usable memory each, this needs roughly 10
to 15 machines for the idempotency cluster alone, before adding any extra safety margin. Raw
request throughput is not the limiting factor here. Even at peak (about 300,000 checkout-API
requests per second, where every "place order" call does at least one idempotency lookup and one
write), individual Redis nodes can each handle over 100,000 operations per second. So a modestly
split cluster comfortably clears this bar. The real limiting factor is memory. This is exactly why
the TTL of 24 to 48 hours matters as a capacity lever, not just a data-cleanup detail: doubling
the TTL roughly doubles this cluster's memory bill.

### 2.7.2 Inventory reservation store

Using Little's Law again, this time for reservations rather than payments, gives a clean picture
of why this store's total size is small, even while its pressure on individual hot keys (§2.6) is
severe.

```
λ (avg reservation creation rate) = 600M line items/day ÷ 86,400 ≈ 6,944/sec ≈ ~7,000/sec
W (reservation TTL)               = 15 min = 900 sec

Average concurrent HELD reservations = λ × W = 7,000 × 900 ≈ 6.3 million
```

At peak, the order rate is about 50,000 per second, times about 3 items per order, giving roughly
150,000 reservation attempts per second. If this peak rate were somehow sustained for the entire
15-minute TTL window (an extreme, unrealistic case, used only to find an upper bound):

```
Peak concurrent HELD reservations (sustained) = 150,000 × 900 = 135 million
```

In real life, flash-sale peaks last only a few minutes, not a full sustained 15-minute window. So
the real peak concurrency stays well below 135 million. But even taking this extreme number at
face value:

```
Reservation record size ≈ 200 bytes (small: ids, quantity, state, expiry)
135,000,000 × 200 bytes ≈ 27 GB
```

**The point worth remembering:** the total memory used by the reservation store is tiny, only tens
of gigabytes even at an extreme, fully-sustained peak. So memory size is never the real constraint
for this store. The real constraint, as shown in §2.6, is write pressure on individual hot keys,
not the size of the whole dataset. This is a useful contrast with the idempotency store, where the
opposite is true: total size (driven by TTL) is the binding constraint, and no single key there is
ever especially "hot."

## 2.8 Bandwidth and Growth Projection

### 2.8.1 Bandwidth

Checkout API messages (session data, cart lines, totals, shipping options) typically run about
2 to 5 KB per request or response. Let's use about 4 KB combined for one request-and-response pair
as a round working number.

```
Peak checkout-API bandwidth ≈ 300,000 QPS × 4 KB ≈ 1.2 GB/sec ≈ ~9.6 Gbps

With TLS/HTTP overhead (about 25–30% extra) ≈ ~12–13 Gbps sustained at peak
```

That is just the payload layer, spread across many regional gateways and load balancers behind
the CDN/Edge tier described in Chapter 5. It is not a single network link's number. It is the total
that the whole fleet must absorb. Even "just JSON" traffic needs serious network planning at this
scale, meaning multiple 10, 40, or 100 gigabit links per region, not a single connection.

Every confirmed order also sends an `order.placed` event, plus saga and outbox events, onto
Kafka (a message queue system) for fulfillment, notifications, and analytics teams to consume:

```
Order events/day ≈ 200M orders × ~2.5 KB/event ≈ 500 GB/day (average)
```

This is a modest, steady stream compared to the API traffic layer. But it still spikes along with
orders (see §2.2), and it must be spread across enough Kafka partitions so that flash-sale order
bursts do not create a "hot partition," which is the same kind of hotspot problem described for
inventory in §2.6.

### 2.8.2 Growth projection

Let's treat this as a clearly stated assumption, not a canonical fact. Assume GlobalMart's orders
per day grow by about 25% each year, a reasonable growth rate for a large marketplace.

```
Year 0 (today):  200M orders/day
Year 1:          200M × 1.25        ≈ 250M orders/day
Year 2:          200M × 1.25²       ≈ 313M orders/day
Year 3:          200M × 1.25³       ≈ 391M orders/day
```

Carrying this forward into storage and peak rate:

```
Year 3 annual order storage (raw) ≈ 391M × 5 KB × 365 ≈ ~713 TB/year (up from ~365 TB today)
Year 3 peak order rate (still about 20× average) ≈ (391M/86,400) × 20 ≈ ~90,500 orders/sec
```

The point of this projection is not the exact numbers. It is to show that the current design, with
1,024 shards holding about 10 times more throughput than needed today, a Redis idempotency layer
sized by TTL rather than a fixed cap, and clear hot and cold storage tiers, has room to absorb
several years of growth simply by adding more of the same kind of hardware. It does not need a
full redesign at year one. That question, "does this design scale by adding boxes, or does it need
a rebuild," is exactly what a growth projection is meant to answer.

## 2.9 Summary: The Provisioned System

| Dimension | Average | Peak | Basis |
|---|---|---|---|
| Order-placement rate | ~2,300/sec | ~50,000/sec (~20×) | 200M/day ÷ 86,400; canonical peak multiplier |
| Checkout-session rate | ~7,000/sec | spikes less than orders | 600M/day ÷ 86,400 |
| Checkout-API QPS | ~42,000/sec | ~300,000/sec | sessions × 6 (avg); orders × 6 (peak) — matches exactly |
| Payment transactions/sec | ~2,550/sec | ~65,000–75,000/sec | 220M/day; assumed 1.3–1.5× peak retry overhead |
| Concurrent in-flight authorizations (Little's Law) | ~765–3,825 | ~19,500–112,500 | λ × W, W = 300–1500ms PSP latency |
| Order data | ~1 TB/day, ~365 TB/yr | — | 200M × 5 KB |
| 7-year order retention | ~2.56 PB raw, ~3.5–7.7 PB provisioned | — | replication vs. erasure coding |
| Hot order tier (90 days) | ~90 TB raw, ~270 TB provisioned | — | 1 TB/day × 90 |
| Idempotency store | ~250–400 GB working set, ~900 GB provisioned | — | 220 GB/day × TTL 24–48h × 3 (replicas) |
| Order DB shards | 1,024 | ~88 GB/shard hot, ~49 orders/sec/shard at peak | data-size and blast-radius bound, not QPS bound |
| Hot-SKU inventory row (single flash-sale SKU) | — | ~5,000–15,000 attempts/sec on one row | contention, not aggregate capacity |
| Reservation store (concurrent HELD) | ~6.3M | up to ~135M (extreme, sustained case) | λ × W, W = 15 min TTL; ~27 GB even at extreme peak |
| Checkout-API bandwidth | — | ~9.6–13 Gbps | 300K QPS × ~4KB × TLS overhead |
| Order-event stream (Kafka) | ~500 GB/day | spikes with orders | 200M × ~2.5 KB |

## Interview Tips

- **Start with the funnel math, and show it matches the canonical number.** Working out that peak
  checkout-API load equals peak order rate (50K) times calls per session (6), and landing exactly
  on 300K, is a small but strong moment. It shows you understand why peak traffic concentrates on
  "place order" instead of spreading evenly, the way average traffic does. Say this out loud. Name
  the shift, not just the number.
- **Little's Law is the single most useful tool in this chapter. Use it twice, and say so clearly.**
  First for payment authorizations: concurrent in-flight calls equal arrival rate times PSP
  latency, landing around 20,000 to 112,000, depending on your latency assumption. Then for
  inventory reservations: concurrent held reservations equal arrival rate times the 15-minute TTL,
  landing around 6.3 million on average. Naming the law, applying it correctly twice, and then
  drawing the design conclusion (async I/O and connection-pool sizing for payments; TTL as a
  memory lever for reservations) is a very strong signal.
- **The hot-row contention idea is the other strong signal, and it is a qualitative insight, not
  just arithmetic.** Point out clearly that a system can look perfectly sized overall (enough
  shards, enough spare request capacity everywhere) while one single logical row, for one viral
  product, is overloaded by ten times or more. Explain that this cannot be fixed by "add more
  database capacity," because correctness requires serializing writes to that one row somewhere.
  Name this as a completely different class of problem from aggregate capacity. You do not need to
  design the full fix here (atomic decrements, counter splitting, queueing, or a dedicated surge
  lane); just naming the problem correctly is the right depth for this chapter.
- **Always state your assumption when a number is not canonical, and always give a range.** The
  1.3 to 1.5 times peak payment-retry overhead, the 2 to 3 times hot-product retry multiplier, and
  the 25% yearly growth rate are all clearly flagged as assumptions in this chapter. Do the same
  out loud in an interview. It is more convincing to say "I assume X because Y, which gives a range
  of A to B" than to state one suspiciously exact number.
- **Explain clearly why 1,024 shards is not simply "peak load divided by per-shard capacity."**
  The gap between the throughput floor (100 shards) and the actual count (1,024), a 10 times
  difference, is exactly the kind of detail that shows you have thought about real operations,
  such as shard size, backup and restore time, and blast radius, not just steady-state math.

## Key Takeaways

- **Build numbers from other numbers. Do not invent them.** Every number in this chapter comes
  from multiplying or dividing two or more of the brief's canonical figures, such as DAU, orders
  per day, conversion rate, and item counts. When a number you build lands exactly on a canonical
  number, as peak checkout-API load did, treat that as a strong internal consistency check, not a
  coincidence to skip past.
- **Checkout peaks are spikier than normal traffic because flash sales concentrate load onto one
  API call ("place order") and a small number of hot products**, not because overall browsing
  scales up 20 times. Average traffic spreads across a six-call funnel with 67% abandonment. Peak
  traffic compresses toward the calls that matter for money and inventory.
- **Little's Law turns "requests per second" into the concurrency number that actually decides
  infrastructure size.** At peak, payment authorizations need tens of thousands, and possibly over
  100,000, concurrent in-flight calls. This is two orders of magnitude above what a naive glance at
  request-per-second numbers would suggest, because external PSP latency (300 to 1500
  milliseconds) dominates. This single fact drives the async I/O and per-PSP connection-pool
  decisions made in Chapters 5 and 8.
- **A single hot inventory row can be overloaded by 5,000 to 15,000 attempts per second, while the
  rest of the system is comfortably sized.** Aggregate capacity and hot-key contention are
  separate problems, with separate solutions, covered in Chapters 7 and 9.
- **Storage sizing is a tiering story, not one single number.** It is about 1 TB per day and about
  365 TB per year of raw order data, about 90 TB kept hot (90 days, fast NVMe storage) versus
  multiple petabytes kept cold (7-year retention), plus a Redis-backed idempotency store sized by
  TTL (about 250 to 400 GB) rather than by raw daily volume.
- **Shard count is often set by operational limits, such as per-shard data size, blast radius, and
  resharding speed, not purely by peak throughput.** 1,024 Order Database shards give about 88 GB
  per shard and about 10 times more throughput headroom than the bare peak-load floor requires.
- **Total memory size and hot-key write pressure are two different axes of the same problem.** The
  idempotency store is limited by size (TTL multiplied by volume). The reservation store is limited
  by write pressure on individual products, not by its own small total size.
