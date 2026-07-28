# Chapter 9 — Scaling

Chapters 5 to 8 built a *correct* checkout system. It uses a saga (a sequence of steps with
rollback actions) to reserve inventory, authorize payment, save the order, and never
double-charge or oversell. This chapter answers the next question every interviewer asks:
"Okay, now make it work at GlobalMart's real scale." A design that is correct only at 100
requests per second is not really correct. It is just a demo.

Scaling checkout is different from scaling search. Search scales by adding more copies of
read-only, loosely consistent shards (a shard is one partition of the data). More copies give
more capacity, in a simple straight line, because one search request never needs to coordinate
with another. Checkout cannot do this on the money path. The whole point of Chapters 6 to 8 was
to make sure only one buyer can win a scarce resource, like the last unit of a product or one
use of a payment idempotency key. So scaling checkout means scaling **around** that
serialization, not removing it. We do this by spreading the serialization across many
independent shards, absorbing bursts of traffic until more capacity arrives, and keeping every
non-essential piece of work off the critical path.

## 1. What Scaling Means Here

Let us recall the canonical numbers from Chapter 2.

| Quantity | Value |
|---|---|
| Orders / day | ~200 M |
| Average order-placement rate | ~2,300 orders/sec |
| **Peak order-placement rate** | **~50,000 orders/sec** (about 20x average) |
| Average checkout-session rate | ~7,000 sessions/sec |
| **Peak checkout-API QPS** | **~300,000 QPS** |
| Checkout API calls per session | ~6 (create, shipping, quote, payment, review, place) |

Here is the most important point of this whole chapter: **the average load is not the hard
part.** 2,300 orders per second, spread across about 1,024 Order DB shards and thousands of
different products, is an easy problem. It works out to about 2.2 orders per second per shard.
A system built for this average, with a little extra room, would handle a normal Tuesday
without any trouble.

The hard part is the **shape** of the peak. 50,000 orders per second is not spread evenly
across GlobalMart's whole catalog. It is a flash sale. Tens of thousands of buyers hit
"place order" for the same small group of products, all within the same 60-second window. This
moment can be planned in advance, like a marketing "drop" at exactly 12:00 noon. Or it can be a
surprise, like a celebrity posting about a product on social media. Either way, this means
GlobalMart's scaling problem is really two separate problems. If we mix them up, we design the
wrong solution.

1. **Scaling steady, spread-out load.** This means adding more orchestrator instances, more
   database shards, and more connection-pool slots (a connection pool is a set of ready-to-use
   network connections a service keeps open). This is normal horizontal scaling. Sections 2 and
   3 cover it.
2. **Scaling a spike aimed at a tiny slice of data.** This means one database row, one
   inventory counter, or one payment provider account getting 50 times its normal share of
   traffic in the same second. Adding more servers elsewhere does not help here. Every request
   still needs to check the *same* row to see if the last 200 units of a $9.99 phone case are
   still available. This is the hot-SKU problem (SKU means "stock keeping unit," a specific
   product). It needs special surge tools (Section 4) and a splitting trick borrowed from
   Chapter 7 (Section 5).

Everything else in this chapter — multi-region design, caching rules, moving work off to the
background, capacity testing — exists to support these two problems. We keep the stateless
services and the sharded databases fast and easy to scale out. And we make sure that hot-key
spikes degrade gently, instead of crashing the payment tier or the whole platform.

```
                     STEADY LOAD                         FLASH-SALE SPIKE
                 (spread across products)            (aimed at a few products)
   +--------------------------------+        +--------------------------------+
   |  2,300 orders/sec average      |        |  50,000 orders/sec peak        |
   |  spread over millions of SKUs  |        |  90% aimed at ~50-200 hot SKUs |
   |  -> solved by adding more      |        |  -> solved by admission        |
   |     stateless instances and    |        |     control, queueing, stock-  |
   |     more DB shards             |        |     splitting, load shedding   |
   +--------------------------------+        +--------------------------------+
```

## 2. Scaling the Stateless Services

Every service above the databases is designed in Chapter 5 to be **stateless**. This means the
service does not keep session data or saga progress in its own memory between requests. This
list includes the API Gateway, Checkout Orchestrator, Cart Service, Pricing & Promotions
Service, Tax Service, the Payment Service's PSP Adapters, and the Notification Service. The
Checkout Orchestrator runs the saga, but it saves its progress in a store, not just in one
process's memory.

Being stateless is the reason this whole layer scales out easily.

- **Any request can go to any instance.** The API Gateway can send a `place-order` call to any
  Checkout Orchestrator instance. No "sticky session" is needed. If an instance crashes
  mid-request, the client (or the gateway's retry logic) just sends the same request again with
  the same `Idempotency-Key`. Chapter 8's idempotency system makes this safe.
- **Scaling out means "add more instances."** Kubernetes has a feature called the Horizontal Pod
  Autoscaler (HPA). It watches CPU use, the number of requests in flight, and queue length for
  each service, and adds more instances when needed. Because there is no shared memory between
  instances, a new instance is useful the moment it passes its health check.
- **Each service scales on its own**, based on its own bottleneck. The Tax Service is limited by
  CPU, because it evaluates tax rules. The Payment Service is limited by waiting on network
  calls to payment providers. The Cart Service is limited by memory and cache size. If we put
  all of this in one big service, we waste capacity. Splitting them into separate services (as
  Chapter 5 does) lets each one scale to match its own real bottleneck.

**One place needs extra care: planning capacity for slow, external calls.** The clearest
example is the payment authorization call. This call dominates the place-order latency budget.
It takes 300 to 1,500 milliseconds out of a total 2.5-second target (from Chapter 2). This is
exactly where Little's Law helps us.

### 2.1 Little's Law and Payment Connection-Pool Sizing

Chapter 2 introduced Little's Law for estimating capacity. In a stable system:

```
L = lambda x W
```

Here, **L** is the average number of requests "in flight" at the same time. **lambda** is the
arrival rate of requests. **W** is the average time each request spends in the system. Let us
apply this to the Payment Service's outbound calls to payment providers (called PSPs, short for
Payment Service Providers). This formula tells us exactly how many concurrent connections or
threads we need. If we do not provision enough, requests do not just slow down. They also start
queuing behind a full connection pool, which adds even more delay. This delay can cascade into
timeouts across the whole saga.

**Worked example.** At peak, roughly 50,000 orders per second each need one payment
authorization call (sometimes more, for split payments or retries). Let us assume:

- Average payment provider latency **W is about 500 ms** (inside the 300–1,500 ms range, closer
  to the common case).
- Peak arrival rate **lambda is about 50,000 per second** (worst case: every order at peak needs
  an authorization call).

```
L = lambda x W = 50,000/sec x 0.5 s = 25,000
```

So GlobalMart's Payment Service fleet needs about **25,000 concurrent in-flight calls** of
capacity at peak. This does not mean 25,000 threads sitting idle and blocked. That design (one
thread per request) does not scale well. Instead, it means 25,000 concurrent logical requests,
handled using async I/O (input/output that does not block a thread while waiting). This uses
tools like non-blocking HTTP clients, with a connection pool sized correctly for each payment
provider. If the Payment Service runs on, say, 200 instances, each instance needs to handle
about 125 concurrent outbound calls. A modern async HTTP client on one instance can easily
handle this, as long as the connection pool for each provider is sized for it (not left at a
default of 20-50 connections, which is too small).

Two things make this number worse in real life, and a good design plans for both.

1. **Tail latency matters more than average latency.** At the 99th percentile (the slowest 1% of
   calls, taking up to 1,500 ms), the same 50,000 per second arrival rate gives us
   `L = 50,000 x 1.5 = 75,000`. Connection pools sized only for the average will fill up exactly
   when payment latency gets worse. This is also exactly when a careless retry policy would add
   *even more* concurrent calls. Section 4.4 explains the fix: use circuit breakers, not more
   retries.
2. **Retries add extra load on top of lambda.** If about 2% of authorization calls time out and
   get retried once, the real lambda becomes about 1.02 times the normal rate. That is small on
   its own. But a "retry storm," where many clients time out and retry at the same moment, is
   not small. This is the thundering-herd problem, covered in Section 4.4.

**Design conclusion:** size payment connection pools, and rate limits per provider, using the
99th-percentile latency and the peak arrival rate, not the average. Also, when the pool is full,
reject new requests quickly with a clear message like "payment service busy, please retry."
Do not let requests queue forever. An unlimited queue in front of a slow outside dependency is
how one payment provider's bad day turns into GlobalMart's outage.

### 2.2 Sizing the Rest of the Stateless Tier

The same `L = lambda x W` idea applies to every other step in the saga that GlobalMart controls
itself, just with a much smaller W. These steps include inventory reservation (80 ms budget),
saving the order (40 ms), and re-checking price and tax (50 ms). Because GlobalMart controls
these services (they are not an outside company), W is small and steady. So L stays manageable
even at 300,000 QPS.

| Step | W (time budget) | lambda (peak rate) | L (concurrent requests) |
|---|---|---|---|
| Inventory reservation | 80 ms | 50,000/sec | ~4,000 |
| Save the order | 40 ms | 50,000/sec | ~2,000 |
| Re-check price/tax | 50 ms | 300,000/sec (all session calls) | ~15,000 |
| **Payment authorization (PSP)** | **500-1,500 ms** | **50,000/sec** | **25,000-75,000** |

This table makes the point clearly. The payment call needs 10 to 30 times more "concurrency
budget" than every internal step combined, even though it handles a smaller share of total QPS.
We scale internal services by adding more instances. We scale the payment call by careful pool
sizing, circuit breaking, and (as Section 4 explains) never letting the incoming load exceed
what our payment provider relationship can actually handle. GlobalMart can scale its own fleet
in minutes. It cannot scale Visa.

## 3. Sharding the Order DB

The Order DB stores every order and must stay strongly consistent (Chapter 6's saga writes
`CREATED` and `CONFIRMED` status changes here). At 200 million orders per day (about 1 TB per
day, about 365 TB per year, from Chapter 2), no single database can hold all of this. No single
write path can handle 50,000 order-creates per second. The answer used across this handbook is
**about 1,024 shards, based on `hash(order_id)`.**

### 3.1 Why hash(order_id), and How order_id Is Assigned

```
order_id = <shard-id: 10 bits><timestamp/sequence: 54 bits>   (Snowflake-style ID)
shard    = the shard-id is already inside the ID -> instant routing, no lookup needed
```

Two designs work here, and both are used in real systems. GlobalMart uses the first one.

1. **Shard number built into the ID (preferred).** The Order Service creates each `order_id`
   with the target shard number already inside it. This is a "Snowflake-style" 64-bit ID: some
   bits for the shard, some for a timestamp, some for a sequence number. So routing a write or a
   lookup by `order_id` is just simple math, with no extra lookup step needed. This is why the
   idea of "hash(order_id)" is described loosely, but the real system builds the shard number
   into the ID instead of hashing again at read time. This makes routing instant and avoids any
   collisions.
2. **Hash after the ID is created.** Generate `order_id` on its own (for example, as a UUID),
   then compute `shard = hash(order_id) mod 1024` every time we access it. This is simpler to
   generate, but every read needs to run the same hash. This is fine, as long as the hash
   function and shard count stay consistent everywhere (see Section 3.4 on adding more shards
   later).

Either way, at 1,024 shards and about 2,300 orders per second average, we get about 2.2 writes
per second per shard. This is deliberately much more capacity than we need for steady load. That
extra room is on purpose. It means that even at the flash-sale peak (50,000/sec divided by
1,024, about 49 writes per second per shard), one shard can easily handle the load. This is
true even before we consider that a flash sale creates much more contention on inventory
(Section 5) than on order writes. Writing an order is a simple insert into the database. It is
not a contested read-then-write like a shared inventory counter, so order writes do not create
their own hot spot.

### 3.2 The Buyer Order-History Problem: A Secondary Index

Sharding by `order_id` is great for writing new orders and for looking up one order by its ID
(for example, `GET /v1/orders/{id}`). But it makes one very common read query harder: "show me
this buyer's order history." A single buyer's orders can be scattered across all 1,024 shards
with no pattern. Asking "all orders for `buyer_id = X`" the naive way means checking every
single shard for every request. This does not scale when hundreds of millions of buyers check
their orders at the same time.

The fix follows the same rule stated in the design brief: order-history reads can tolerate being
slightly out of date (this is called eventual consistency). So we build a separate, purpose-made
read store instead of querying the main transactional database.

```
Order DB (1,024 shards, source of truth, sharded by order_id)
        |  outbox / change-data-capture (Chapter 8's outbox pattern)
        v
     Kafka (order.created, order.status_changed events)
        |
        v
Order-History Read Store
  - sharded/indexed by buyer_id, not order_id
  - stores a ready-to-use summary: order + status, no joins needed
  - backing store: a wide-column database or search-style index,
    built for "orders WHERE buyer_id = X ORDER BY created_at DESC"
        ^
        |
  GET /v1/orders?buyer_id=X   (this query is served here, not from the Order DB)
```

This pattern is called **CQRS**, which stands for Command Query Responsibility Segregation. It
just means: use one store for writing, and a different store for reading. The write store (the
Order DB, sharded by `order_id`, strongly consistent) is built for the saga's transactional
writes. The read store (the order-history store, sharded by `buyer_id`) is built for the query
pattern buyers actually use. This read store lags behind by a few seconds, through Kafka, and
that small delay is invisible to a buyer looking at "my orders." The design brief already says
this kind of delay is acceptable. A single order lookup (`GET /v1/orders/{id}`) still goes
straight to the Order DB shard, for a strong and immediate answer. Only the *list* view goes to
the read store. Seller dashboards, which need "all orders for a given `seller_id`," use the same
pattern with a `seller_id`-based read store, fed by the same Kafka stream.

### 3.3 Hot / Warm / Cold Tiers and 7-Year Retention

Laws about tax and finance (tax audits, chargeback disputes, legal holds) require GlobalMart to
keep order records for **7 years**. But almost all everyday reads (a buyer checking a recent
order, a support agent looking at a live issue) only touch the last few months. Serving all 7
years of data (multiple petabytes, where 1 petabyte equals 1,000 terabytes) from the same fast
NVMe disks that serve live writes would waste money. It would also slow down the writes that
actually matter for speed.

| Tier | Age | Approx. size | Storage type | How it is accessed |
|---|---|---|---|---|
| **Hot** | 0-90 days | ~90 TB (about a quarter of 365 TB/yr) | Fast NVMe SSD, the live 1,024-shard cluster | Reads/writes in under 10 ms, used by the live saga |
| **Warm** | 90 days to 1 year | ~275 TB | Cheaper SSD, read replicas, maybe fewer shards | Order-history lookups, support, returns; a few seconds delay is fine |
| **Cold** | 1-7 years | multi-petabyte | Object storage (like S3 or Glacier), columnar format, split by order date | Only batch or audit queries (tax, legal cases); minutes to hours delay is fine |

Moving data between tiers is done by a background **archival job**, not by the live checkout
flow. A nightly (or slow, continuous, low-priority) process finds orders older than the hot-tier
limit. It copies them to warm or cold storage in a format made for later queries. Once the copy
is confirmed safe, it removes the record from the hot shards, keeping those shards small and
fast. Removing from the hot tier never means deleting the record forever (we must keep it for 7
years by law). It only means moving it. Cold-tier data is still saved by `order_id` and
`checkout_group_id`, so a specific record can always be found again on demand. For example, a
chargeback dispute from 4 years ago can still be looked up, even though the cold tier is not
built for high query volume.

This tiering is what makes the "about 1 TB per day, about 365 TB per year" growth from Chapter 2
manageable. The fast, expensive tier stays around the same size forever (about 90 TB), while
cheap object storage absorbs the multi-year, multi-petabyte tail.

### 3.4 Adding More Shards Later (Resharding)

1,024 shards is a starting point, not a fixed limit forever. GlobalMart doubles the Order DB
shard count from time to time as order volume grows year over year (for example, from 1,024 to
2,048). Because the shard number is built into the ID for the life of an order, adding more
shards only affects how *new* order IDs are created. We update the ID-assignment rule so new
orders go into the wider shard map. We do not need to rewrite the IDs of existing orders. Old
orders keep pointing to their original shard (which might now sit behind a routing layer), while
new orders spread across more shards. This "never move old data, only widen the map for new
data" approach avoids a live migration of the main transactional database. The cost is that the
shard map only ever grows. This is acceptable, because storage is cheap compared to write speed,
and old data moves to cold storage anyway (Section 3.3).

## 4. Flash-Sale Surge Handling

This section is the heart of what makes "scaling checkout" different from "scaling a normal web
service." A flash sale is a spike with a **known shape**, and often a **known time**: 10 to 20
times normal load, arriving within seconds, aimed at a small group of products. The goal is not
to make the backend fast enough to serve unlimited demand for a fixed supply. That goal makes no
sense when 40,000 buyers want 200 units of the same phone case. The real goal is to **let in
exactly as much load as the system, and the payment provider relationship, can safely handle**.
We turn away or queue the rest in a fair, predictable way. We make sure the money and inventory
path never sees more traffic than it was built for. And everyone waiting gets an honest, clear
experience, not a spinning wheel or a silent failure.

### 4.1 Admission Control: The First and Cheapest Defense

Admission control means deciding, as early as possible, how many requests are even allowed in.
Ideally this happens at the CDN or edge layer (from Chapter 5's design), before load even
reaches the API Gateway. It is cheap to reject a request early. It is expensive to let a request
fail deep inside the system.

- **Treat endpoints differently based on importance.** Not all 6 checkout-API calls in a session
  matter equally. `place-order` (the money path) gets the highest priority and the strongest
  protection, because failing there is the worst outcome. A simple page refresh, like
  `GET /v1/checkout/sessions/{id}`, can be dropped first under pressure (Section 4.3). General
  browsing or catalog traffic, even though it is outside this handbook's scope, gets dropped
  before any checkout traffic.
- **Rate limiting** happens at the API Gateway using a method called a token bucket (a simple
  counter that refills over time and limits how many requests pass through). We limit per buyer
  (so one script cannot grab a whole hot product), per IP address (a blunt but useful defense
  against bot attacks), and per product for flash sales (a global limit on how many `place-order`
  attempts per second are even allowed to reach the Inventory Service for one hot product, no
  matter how many buyers are trying).
- **Reject based on load, instead of queuing at the edge.** A gateway that queues extra requests
  forever just moves the problem further down the pipeline, and adds delay. It is better to
  reject fast with a clear signal (HTTP status codes 429 or 503, with a `Retry-After` header
  telling the client when to try again), or to send the buyer into the waiting room described
  next, than to let a queue grow without limit.

### 4.2 The Virtual Waiting Room

For a **known** flash-sale event, like a scheduled product drop or an announced restock,
GlobalMart puts a **virtual waiting room** in front of checkout-session creation for that
product's page. A virtual waiting room is a system that gives buyers a ticket and a place in
line, instead of letting everyone try to check out at the exact same second. This is the single
most useful tool for handling a surge, because it turns an uncontrolled stampede of arrivals
into a controlled, steady stream.

```
Buyer clicks "Buy Now" on a hot product
        |
        v
  Waiting Room Gateway  -- gives out a ticket (a signed token with issue time and position)
        |                   buyer's browser checks their position regularly
        |
        |  lets tickets through at a fixed rate the backend can handle
        |  (matched to how fast the Inventory Service can process
        |   reservations for that product's sub-buckets -- Section 5)
        v
  POST /v1/checkout/sessions  (only for tickets that were let through)
        |
        v
  the normal saga: reserve -> authorize -> create -> capture (Chapter 6)
```

Some key properties of this design:

- **It limits the rate of new arrivals, not how many people are already checking out.** The
  waiting room does not simply cap "how many buyers can be in checkout at once." It controls the
  *rate* at which new buyers get a checkout session. This rate is matched to what the inventory
  layer (Section 5) can actually sustain without falling over.
- **It is fair, using tickets, not a race.** Buyers get a ticket with a position and an estimated
  wait time. This is much better than a race where the fastest click wins. It also reduces
  buyers hitting "refresh" over and over, which itself reduces load, because a buyer who trusts
  the queue stops retrying. Tickets are short-lived and can only be used once. This connects to
  Chapter 8's idea of idempotency: a ticket, just like an idempotency key, should let a buyer in
  exactly once.
- **It stays separate from the money path.** The waiting room sits in front of *session
  creation*, not inside the saga itself. So it never adds delay to `place-order`. It only
  controls how many sessions are allowed to start competing for the hot product in the first
  place. This keeps the 2.5-second budget for `place-order` unchanged for anyone who does get
  through.
- **It works best for events we know about ahead of time.** For a surprise viral spike, we can
  still turn on the waiting room reactively, by detecting a sudden concentration of traffic on
  one product. But it works best when it is set up ahead of a planned marketing event. This is
  also why the game days in Section 9 matter: the waiting room's admission rate needs to be
  tested and rehearsed, not figured out for the first time at noon on Black Friday.

### 4.3 Load Shedding: Protect the Money Path by Giving Up Everything Else

Under heavy, sustained overload, GlobalMart drops work in a strict order of priority. The least
important work gets dropped first.

| Priority | Work | Dropped first? | Why |
|---|---|---|---|
| 1 (never dropped) | `place-order` for a session already let in | No | This is the money path; dropping it means a lost sale, or worse, a saga stuck halfway |
| 2 | Inventory reservation and payment authorization (internal saga steps) | No | Same saga; dropping a step here forces expensive rollback actions (Chapter 6) |
| 3 | Creating a session for a brand-new buyer | Yes, first | Turning someone away at the door, via the waiting room or a 429 response, is cheaper than failing halfway through |
| 4 | Refreshing shipping or tax quotes on an existing session | Yes | Fall back to the last known price instead of recalculating live (Section 7) |
| 5 | Order-history reads, recommendations, seller dashboards | Yes, and aggressively | These are already eventually consistent (Section 3.2); a few extra seconds of delay is invisible |
| 6 | Analytics, and non-critical notifications | Yes, first and hardest | Fully handled in the background (Section 8); falling behind here just means Kafka consumer lag, not lost data |

The mechanism behind this table is called **bulkheads**, borrowed from ship design, where
separate compartments stop one leak from sinking the whole ship. Here it means separate thread
pools, connection pools, and server pools for each priority level. This way, a flood of
low-priority traffic physically cannot starve the high-priority traffic of CPU, connections, or
database capacity. This is the same idea used to protect the payment connection pool in
Section 2.1. A pool used up by one caller should hurt only that caller, not everyone sharing the
system.

### 4.4 Autoscaling Lead Time and Pre-Scaling for Known Events

Reactive autoscaling, where Kubernetes HPA watches CPU or queue depth and reacts, usually takes
**1 to 3 minutes**. This time is needed to notice the pressure, schedule new instances, download
container images, pass health checks, and warm up (fill caches, open connection pools). A flash
sale can go from normal load to 20 times normal load in **under 10 seconds**, the moment the
"Buy Now" button becomes active. Reactive autoscaling is simply too slow for the opening seconds
of a scheduled drop. By the time new instances are ready, the sharpest part of the spike has
already hit the fleet that was sized for yesterday's average load.

The answer is **pre-scaling for known events**. This means treating Black Friday, a marketing
flash sale at noon, or a big regional holiday sale as planned capacity events on a calendar, not
as surprises.

- The stateless layer (Checkout Orchestrator, Cart, Pricing, Tax, and Payment services, and the
  API Gateway) is scaled up *ahead of time*. This usually happens 30 to 60 minutes before a known
  event, to the fleet size that the expected peak requires. This size is confirmed by the load
  tests described in Section 9, not just guessed and left for the autoscaler to discover live.
  Scaling back down afterward can happen gradually and reactively, but scaling up for a known
  cliff cannot wait for a reaction. For unknown or viral spikes, GlobalMart still relies on
  HPA plus the load-shedding and waiting-room tools above as a safety net, since there is no
  calendar entry to prepare for in advance.
- The payment provider relationship is also *notified ahead of time*. Merchants routinely tell
  their payment processor about an expected spike in volume before a major sale. This lets the
  processor prepare extra capacity and raise the merchant's temporary transaction limit. As
  Section 2.1 explained, GlobalMart can scale its own servers in minutes, but it cannot
  unilaterally scale another company's fraud and authorization systems. Skipping this step is a
  common real-world cause of "the website was fine, but every checkout suddenly started failing
  or timing out" during a big sale. In that case, the real bottleneck had quietly moved outside
  GlobalMart's own systems.
- **Protecting the payment tier from a sudden flood** is a specific failure to design against.
  If the payment provider slows down under the opening surge, a careless Payment Service that
  aggressively retries failed calls will make things worse for an already-struggling provider.
  This can turn a slow provider into a fully broken one, for every merchant using it, not just
  GlobalMart. The defenses, used together:
    1. **A circuit breaker for each payment provider.** After the error rate crosses a threshold,
       stop sending new authorization calls to that provider for a cool-down period. Fail fast
       with a message like "payment temporarily unavailable, try again shortly," instead of
       queuing behind a dependency that is already failing.
    2. **Exponential backoff with random jitter** on legitimate retries. This spreads retries out
       over time, instead of letting them bunch up into another sudden flood.
    3. **A limited, prioritized queue** in front of the payment call. This is like a bucket with a
       small hole that lets requests out at a steady, limited rate (sometimes called a "leaky
       bucket"), not an unlimited buffer. This makes sure the Payment Service's own outgoing
       traffic never goes above the pool size set in Section 2.1, no matter how much demand comes
       in.
    4. **Using more than one payment provider as a backup** (covered further in Chapter 10). If
       GlobalMart uses more than one provider for a payment method or region, a broken primary
       provider can send some traffic to a backup. This must be done carefully, so it does not
       break the idempotency guarantees from Chapter 8 across two different providers for the same
       `checkout_group_id`.

## 5. Hot-SKU Scaling: Stock Splitting as a Scaling Tool

Chapter 7 introduced stock splitting, per-product serialization, and in-memory counters as
tools for **correctness**. They stop us from overselling under contention (contention means many
requests competing for the same resource at once). Let's look at these same tools again, but
this time as **scaling** tools. They are also the only reason a single hot product can survive
50,000 concurrent reservation attempts without either overselling or crashing.

### 5.1 The Problem, Restated as a Scaling Problem

A simple inventory design stores `available_qty` as one row, or one counter, per product. Under
flash-sale load, say 40,000 buyers hit "Buy Now" on that product within a few seconds. Every one
of them must check and update **that one row** to avoid overselling (this is Chapter 7's core
correctness rule). But one row has a hard limit on throughput. This limit comes from lock
contention and disk speed on a single partition. A well-tuned system might handle a few hundred
to a few thousand writes per second on one row, nowhere near 40,000 per second. This is called a
**hot partition**. It is the exact same problem that sharding (used for the Order DB and
Inventory DB) is designed to avoid, except sharding by product ID does not help *inside* one
extremely popular product.

### 5.2 Stock Splitting: Turn One Hot Row Into K Parallel Rows

```
Product "SKU-9921"    total stock = 2,000 units
                |
                v  split into K = 20 sub-buckets when the sale starts
   +-----+-----+-----+-----+---     ...      ---+
   | B0  | B1  | B2  | B3  |                  | B19 |
   | 100 | 100 | 100 | 100 |       ...        | 100 |  <- 100 units each
   +-----+-----+-----+-----+---      ...     ---+
      ^     ^     ^     ^                        ^
      |     |     |     |                        |
   a consistent hash of the reservation request routes it to one bucket
   (for example, hash of buyer_id or request_id, mod K, or round-robin)
```

Splitting the 2,000-unit counter into 20 separate 100-unit sub-buckets, each its own row, ideally
on a different server, turns one impossible-to-handle hot row into 20 rows. Each one now faces
about 1/20th of the contention, roughly 2,000 attempts per second each instead of 40,000 per
second on one row. This is horizontal scaling applied *inside* a single product, similar to how
we shard the Order DB in Section 3. The difference is that here the "shard key" is artificial (a
bucket number), not natural (like `order_id`), because the goal is to spread out contention, not
to spread out data size.

Being honest about the trade-off: sub-buckets can run out unevenly by pure chance. For example,
bucket B3 might empty out while B17 still has 40 units left. This can cause two problems: (a) a
buyer sees "sold out" too early, even though stock technically remains elsewhere, or (b) we need
an **overflow step**, where a bucket that hits zero checks a neighboring bucket, or a small
shared backup pool set aside for this purpose, before declaring the item out of stock. GlobalMart
accepts a small amount of problem (a) as the price of much higher throughput. It reduces the
problem with a small backup pool, for example holding back the last 2 to 5% of stock unsplit,
checked only when a buyer's assigned bucket is empty. This is simpler than trying to perfectly
balance load across buckets under extreme time pressure.

### 5.3 Serializing Requests Per Product

*Inside* one sub-bucket, correctness still requires serializing concurrent updates (from
Chapter 7). Two buyers cannot both take the "last unit" of bucket B3. The scaling idea here is
that serializing at the level of one sub-bucket (100 units' worth of contention) is cheap. But
serializing at the level of the whole product (2,000 units' worth of contention) is the
bottleneck we are trying to remove. In practice, this is often done with a single-threaded actor
(a small unit that processes one request at a time) per sub-bucket, or with an atomic
decrement operation (below) that the database or cache handles per key on its own. Either way,
the serialization point is now doing 1/K as much work.

### 5.4 In-Memory Counters as a Shock Absorber

The other half of this technique is to put a fast, atomic, in-memory counter (using Redis, an
in-memory data store, or a similar in-process cache) **in front of** the durable Inventory DB for
hot sub-buckets. This lets the counter absorb the raw volume of the spike before it ever reaches
disk.

```lua
-- Redis-side atomic check-and-decrement (one single command, no race window)
local remaining = redis.call('GET', KEYS[1])
if tonumber(remaining) >= tonumber(ARGV[1]) then
    redis.call('DECRBY', KEYS[1], ARGV[1])
    return 1   -- reservation granted (fast path)
else
    return 0   -- bucket is empty; try the overflow bucket, or fail fast
end
```

- Every reservation attempt hits the in-memory counter first. This operation takes microseconds,
  not the milliseconds a durable database write takes. So the *rate* at which we can say "no" to
  the roughly 99% of buyers on a 2,000-unit product who are going to lose the race anyway, is
  basically unlimited compared to 40,000 per second.
- Only the small number of requests that pass the fast check (a granted reservation) go on to
  the slower, durable, strongly consistent write in the Inventory DB (from Chapters 4 and 7).
  This expensive path now only handles traffic that matches the *actual remaining stock*, not
  the *actual demand*. For a 2,000-unit product, that means at most 2,000 durable writes for the
  whole event, no matter how many buyers tried.
  This follows the same rule that governs Section 7 below: caching should only be used to
  protect the system from load, never as the source of truth for anything involving money. The
  durable database stays authoritative, and the in-memory layer is a strict, careful gate in
  front of it. We check it regularly against the database (see Chapters 7 and 8) to catch drift.
  For example, a Redis server might fail right after granting a reservation, but before the
  durable write completes. This case is handled by the same reservation TTL and release pattern
  from Chapter 7, and never by simply trusting Redis as the final truth.

**Seen through the scaling lens, not just correctness:** stock splitting turns a problem limited
by one partition into a problem spread across K partitions. This is classic horizontal scaling.
The in-memory counter turns a problem limited by disk speed into a problem limited by memory
speed, at least for the reject path, which is where over 99% of flash-sale traffic for any
popular product actually ends up. Together, these two tools let a single $9.99 product survive
a stampede of 40,000 buyers, without overselling even one unit, and without overloading the
durable Inventory DB.

## 6. Multi-Region Active-Active

GlobalMart runs checkout from several regions at the same time. This is called **active-active**,
meaning every region serves live reads *and* writes. It is not one main region with passive
backups waiting to take over. The hard design question is not "how do we copy data between
regions." It is **which region owns each piece of strongly consistent data**. The strong
consistency stance from Chapter 6 (strong consistency for inventory, payment, and order
creation) does not get easier just because the system is now global. It actually gets harder,
because a network round trip between regions (tens to a few hundred milliseconds) is large
compared to the whole 2.5-second place-order budget.

### 6.1 The Main Rule: One Owning Region Per Piece of State

GlobalMart avoids a synchronous agreement process across regions (like a system that requires
every region to agree before completing a transaction) on the checkout hot path. Paying for a
cross-region round trip on *every* inventory reservation or payment authorization would eat up a
large, unpredictable chunk of an already tight time budget. It would also make the whole
system's availability depend on the *weakest* region being reachable, instead of just the
*local* region being healthy. This is exactly the failure that active-active design is meant to
avoid.

Instead, **every strongly consistent piece of data has exactly one owning region at a time**,
and all writes to it go there.

```
                     Region: US-EAST
Buyer (US) --------> Checkout Orchestrator -> Order DB shards 0-511
                     (this buyer's data lives and is owned here)

                     Region: EU-WEST
Buyer (EU) --------> Checkout Orchestrator -> Order DB shards 512-1023
                     (this buyer's data lives and is owned here)

              Async, slightly-delayed copies (using Kafka MirrorMaker
              or cross-region change-data-capture) are used for:
              order-history read stores (Section 3.2), analytics,
              cross-region reports, and disaster-recovery backups.
```

- **Order data and payment state are pinned by buyer region**, and secondly by data-residency
  rules (see Section 6.3). A buyer's checkout session, saga, and order records are created and
  owned in that buyer's home region's shard range. A buyer routed to US-EAST always creates
  orders in the US-EAST shard range. There is no cross-region write for a normal, single-region
  buyer's order. This is a simple extension of Section 3's shard-by-`order_id` idea. The shard
  map itself is split by region (for example, shards 0-511 physically live in US-EAST, and
  512-1023 live in EU-WEST). Routing is decided once, at session creation, based on where the
  buyer's request enters the system (Chapter 5's CDN/Edge layer).
- **Copying data across regions is asynchronous, and used only for reading, never for
  writing.** A US-EAST order is copied to EU-WEST, and the other way around, using Kafka-based
  change data capture. This supports disaster-recovery backups and global read stores. For
  example, a buyer traveling abroad can still *view* their order history, served from a
  slightly-delayed copy, using the same pattern from Section 3.2. But EU-WEST never *writes* to
  an order owned by US-EAST. This keeps the "one writer per piece of data" rule, which is what
  makes strong consistency possible without needing every region to agree on every write.

### 6.2 Inventory: Region-Pinned Stock vs. Global Stock

Inventory is the trickier case. Unlike an order, which naturally belongs to one buyer in one
region, one seller's stock might need to be sellable to buyers *anywhere in the world*.

- **Region-pinned inventory (the common case).** A seller's warehouse sits in one physical
  region. Stock for that product is owned by that region's Inventory DB shard. If a buyer in a
  *different* region wants to buy it, the system makes a cross-region reservation call. This
  case is rare on purpose, and checkout clearly labels it as a cross-border order with longer
  shipping time. The Checkout Orchestrator, running in the buyer's home region, makes a direct
  call to the Inventory Service in the seller's region for that one step. This adds real
  cross-region delay (tens to a few hundred milliseconds) to that specific step. This is
  acceptable, because cross-border purchases are a small share of all orders, and buyers already
  expect longer shipping times for them. The design does not try to make this rare case as fast
  as the common local case. Instead, it isolates the extra cost to only the orders that actually
  need it. A cart with sellers from multiple regions creates several sub-orders (using
  Chapter 4's `checkout_group_id`), and their reservation calls go out to each seller's region.
  This is an explicit multi-region step inside a single saga, with the usual rollback logic from
  Chapter 6 covering the case where a cross-region call fails or times out (release any
  reservations already granted elsewhere in the same checkout group).
- **Globally-fulfilled products (rarer).** Some products ship from a single global warehouse and
  are sold worldwide, with no natural connection to one region. There are two workable
  approaches here, and both involve a real trade-off.
    1. **Pick one home region to own it.** One region is the only writer for that product's
       inventory. Every other region's reservation attempt becomes a cross-region call into the
       home region. This keeps the correctness story simple (still one writer), but every buyer
       outside the home region pays the cross-region delay. Also, the home region becomes a
       concentration point for that product's traffic. This matters a lot if that product is also
       a flash-sale hot product (Section 5), because now the stock-splitting sub-buckets should be
       thought of as region-local caches of a piece of the home region's official count, rather
       than being separately rebuilt for each region.
    2. **Split stock by region ahead of time.** Divide the global product's total stock into
       region-sized shares *before the sale starts*. This is the same sub-bucket idea from
       Section 5, except the "buckets" are whole regions instead of in-process shards. Each region
       can then reserve purely against its own local share, with no cross-region call needed on the
       hot path. The cost is possible imbalance (one region sells out its share while another
       still has stock), plus the extra work of deciding share sizes ahead of time and rebalancing
       occasionally. GlobalMart prefers this approach for *known* global flash-sale events, for the
       same reason Section 4.4 prefers pre-scaling over reactive scaling: it trades a bit of
       precision in stock allocation for removing cross-region delay, and a cross-region single
       point of failure, from the highest-stakes moment.

### 6.3 Data Residency

Some laws (like GDPR in the EU, and data-localization rules in several other countries) require
a buyer's personal and order data to physically stay within a certain country or region. This
supports, rather than complicates, the region-pinning design above. A buyer's home-region
assignment is driven mainly by *legal residency*. Network speed and closeness are only used as a
secondary factor, when residency rules do not already require a specific region. The Order DB
and payment records for that buyer never leave the required region, except as clearly allowed,
anonymized totals (for example, for global analytics, Section 8). This is also why payment
processing is naturally tied to specific regions in practice. GlobalMart usually works with
different payment providers in different regions (local card networks, regional wallets). So the
Payment Service's PSP Adapters are themselves set up per region. A cross-region order (from
Section 6.2's cross-border inventory case) still authorizes payment through the *buyer's*
regional payment provider, not the seller's.

### 6.4 What Active-Active Gives Us, and What It Costs

- **What it gives us:** no single region is a single point of failure for the whole world. A
  regional outage (Chapter 10) only affects that region's buyers, who can be redirected to the
  nearest healthy region for *new* sessions. However, sagas that were already in progress and
  owned by the failed region need special handling and reconciliation (from Chapter 10). A saga
  cannot simply "resume" in a region that never held its data.
- **What it costs:** truly global, single-writer pieces of data, like a rare global product, or
  a buyer who permanently moves to a different region, need deliberate, explicit handling. They
  do not come for free just because the architecture is active-active. Nothing in this design
  makes cross-region strong consistency cheap. The whole strategy is to make cross-region strong
  consistency *rare*, by pinning ownership to one region, and to pay the cost for it openly, only
  on the small share of requests that truly need it.

## 7. Caching: What Is Safe, and What Is Not

Caching means storing a copy of data somewhere fast, so we do not have to fetch or recompute it
every time. It is the most useful, lowest-risk scaling tool for **read-heavy, slow-changing,
non-money-related** data. But it becomes a real danger if applied to anything the strong
consistency stance from Chapter 6 depends on. The rule that matters is not "is this data read
often." The rule is: **"could a stale (out-of-date) read of this data cause an oversell, a
double charge, or the wrong amount charged?"**

| Data | Can we cache it? | Why, or what to watch for |
|---|---|---|
| Product catalog snapshot (title, images, description) | **Yes**, aggressively (at the CDN edge, with a long expiry time) | This is purely informational, and it is never the final source for price at checkout (a rule that requires re-pricing every time) |
| Shipping-rate tables (zone and weight to cost) | **Yes**, with a version tag and expiry time | These change rarely (only when a carrier contract updates); cache with a version tag, and clear the cache when a "shipping rate changed" event fires |
| Tax-rate tables (region to rate) | **Yes**, refreshed daily, plus event-based updates | Rates change rarely, but must reflect new laws quickly; refresh on a schedule *and* on a "tax rate changed" event; never let it go stale past a fixed limit |
| Static promotion rules (who qualifies, percentage off) | **Yes**, with a short expiry time | The *rule* can be cached; the *count* of how many times it has been used cannot (see below) |
| **Inventory counts / availability** | **No** (not as a simple time-based cache) | A stale read here directly causes overselling, which is the exact problem Chapter 7 exists to prevent. The in-memory counters from Section 5.4 are *not* a cache in this sense. They are an authoritative fast path with strict, exact rules and regular reconciliation, not a "might be a few seconds old" cache |
| **Payment authorization or capture status** | **No** | A cached, out-of-date "AUTHORIZED" status risks charging twice. A cached, out-of-date "FAILED" status risks wrongly declining a payment that actually succeeded. Either mistake breaks the no-double-charge rule |
| **Idempotency records** | **No** (must not be eventually consistent) | Chapter 8's idempotency guarantee needs a strongly consistent answer to "have I seen this key before." A cache that can lag behind brings back the exact race condition idempotency is meant to close |
| Session pricing at the place-order moment | **No** | The rules require an authoritative re-check of price at the exact moment of payment, specifically to stop a stale or cached price from being honored |

**The cache invalidation strategy** (invalidation means telling the cache that old data is no
longer valid) for the "yes" rows above uses several layers, instead of relying only on a simple
expiry time.

1. **Event-driven invalidation, using Kafka.** Whenever the catalog, shipping rates, or tax rates
   change, an event is published (`catalog.updated`, `shipping_rate.changed`,
   `tax_rate.changed`). Cache layers listen for these events and refresh or clear the affected
   entries within seconds, instead of waiting for a timer to expire. This keeps staleness much
   tighter than a timer alone, without needing every single read to check freshness.
2. **A fixed expiry time as a backup, not the main method.** This protects against a missed or
   delayed event. It is set short, in minutes rather than hours, especially for tax and shipping
   data, so even a worst-case miss stays acceptable.
3. **Versioned cache keys, tied to the checkout session.** A `CheckoutSession` (from Chapter 4)
   carries a `session_version`. Any price, tax, or shipping quote calculated for that session is
   tagged with the catalog or rate version it used. This means a session never silently drifts
   onto a newer cached value in the middle of checkout. Instead, Chapter 6's re-check step
   clearly detects a version mismatch and re-calculates the price. This is the correct, visible
   behavior. A price change is shown to the buyer for confirmation, instead of quietly causing a
   hidden cache bug.

The rule to remember for interviews, and for real design reviews: **cache the display data and
calculation inputs that do not directly move money. Never cache the actual money and inventory
decisions themselves.**

## 8. Async Offloading: Keeping the Fast Path Short

The place-order time budget (2.5 seconds at the 99th percentile) has no room for anything that
is not strictly needed to answer one question: "is this order confirmed, charged correctly, and
is inventory correctly reduced — yes or no." Everything else that *feels* like it should happen
during checkout, like sending a confirmation email, notifying the warehouse, feeding a
recommendation model, or updating a seller's dashboard, is moved completely off this fast path.
This is done using Kafka (a message queue system) and the outbox pattern already introduced for
money correctness in Chapter 8.

```
The synchronous saga (Chapter 6) ends when the order is CONFIRMED and payment is captured:
  ... -> Order Service writes the order row AND an outbox row, in the same database transaction
                                |
                    a separate outbox relay process publishes to Kafka
                                v
                    Kafka topic: order.placed  (split by checkout_group_id)
                    +-----------+------------+--------------+---------------+
                    v           v            v              v               v
             Notification   Fulfillment   Analytics /   Seller           Loyalty /
             Service        (handoff to   BI pipeline   Dashboard        points
             (email, SMS,    the ware-    (recommen-     (eventually     ledger
              push)          house)       dation data)   consistent,
                                                           Section 3.2 style)
```

Why this matters for scaling, beyond just keeping latency low:

- **It isolates backpressure.** Backpressure means a slow consumer causing pressure to build up
  behind it. Kafka's durable log means a slow consumer (say, the analytics pipeline having a bad
  day) simply falls behind, and its lag grows, but it never blocks the producer (the Order
  Service). It never adds delay to the next `place-order` call. Compare this to a design where
  checkout directly calls a notification API. A slow notification provider there would directly
  slow down place-order. Under enough load, it could even use up the payment connection pool for
  a completely unrelated reason (Chapter 10 covers this kind of cascading failure in more
  detail).
- **Each downstream consumer scales on its own.** The Notification Service, the Fulfillment
  handoff, and the Analytics pipeline each scale their own group of workers on their own, sized
  to their own needs and cost. A burst of 50,000 orders per second does not require the
  Analytics pipeline to also handle 50,000 per second in real time. It can catch up over the next
  few minutes. This matches the relaxed, eventually-consistent rule the design brief already
  allows for analytics.
- **The outbox pattern avoids the "dual write" problem**, and this matters more, not less, at
  large scale. At 50,000 orders per second, even a small percentage of cases where "the database
  write succeeded but the Kafka publish failed" (or the other way around) adds up to a large
  number of lost or duplicated events. Writing the outbox row in the *same* database transaction
  as the order row, and having a separate process publish it (retrying until it succeeds, with
  the consumer side designed to handle duplicate messages safely), removes this failure mode
  entirely. It does not rely on hoping two separate systems both succeed at the same time.
- **How we split Kafka topics affects ordering.** Splitting the `order.placed` topic by
  `checkout_group_id`, instead of just round-robin, guarantees that all events for one order
  arrive at any single consumer in the order they were created. This matters for a Fulfillment
  consumer, which needs to see `order.created` before `order.status_changed: SHIPPED`, for
  example. This does not need a single global order across all 200 million orders per day, which
  Kafka does not provide anyway, and checkout does not need it.

## 9. Capacity Headroom, Load Testing, and Game Days

Everything above is a design on paper. This section explains how GlobalMart checks that the
design actually holds up, before a real Black Friday finds out the hard way.

### 9.1 Steady-State Headroom

GlobalMart deliberately runs the checkout fleet well below its maximum capacity during normal
times. A common target is **30 to 40% average CPU use** on the stateless layer, and low
single-digit percent use on the Order and Inventory DB shards (recall Section 3.1: about 2.2
writes per second per shard on average, against a much higher per-shard limit). This extra
headroom exists for three reasons, and they all work together. First, it absorbs *unplanned*
spikes, like a viral moment rather than a scheduled event, during the time before reactive
autoscaling catches up (Section 4.4). Second, it keeps tail latency (the slowest requests, not
just the average) stable. Any system with queues tends to see tail latency balloon once
utilization goes much above 60-70%, and this is exactly what the 2.5-second budget cannot
absorb. Third, it gives operators room to safely take some capacity offline, for example rolling
back a bad deployment, or doing maintenance on a shard, without immediately putting the
availability target at risk.

### 9.2 Load Testing Methods

- **Replay a synthetic flash sale.** Capture the shape of real traffic from a past flash sale
  (with personal data removed), and replay it at increasing multiples: 1x, 1.5x, 2x the peak of
  about 50,000 orders per second, against a staging environment built exactly like production.
  The goal is to find the real breaking point *before* a real event finds it for us. Testing at
  1.5 to 2 times the intended peak specifically checks that the system **degrades gracefully**
  (load shedding and the waiting room kick in as designed in Section 4) rather than
  **collapsing** (failures cascading, timeouts causing more retries, causing more timeouts) once
  it goes past its rated capacity. This matters because the real peak sometimes exceeds the
  forecast, and how the system fails at 1.1 times its design capacity matters just as much as
  how it performs at design capacity.
- **Hot-SKU-specific tests**, run separately from general throughput tests. Replay a single-product
  stampede (the exact scenario from Section 5) against the stock-splitting and in-memory-counter
  path specifically. This checks for zero overselling under deliberately tricky, overlapping
  request timing, not just checking that overall QPS holds up.
- **Payment provider sandbox stress tests.** Deliberately inject extra latency and higher error
  rates into the payment provider's test environment, to check that the circuit breakers and
  backoff logic from Section 4.4 actually trigger and recover correctly. Testing only against a
  provider that always responds instantly would miss this entirely.
- **Shadow traffic, also called dark launches**, for architectural changes, like a new sharding
  scheme or a new stock-splitting factor K. This means mirroring a portion of real production
  requests to the new path, without actually sending its response back to the buyer, and
  comparing the results for correctness before switching real traffic over.

### 9.3 Game Days

Beyond automated load tests, GlobalMart runs scheduled **game days**. These are cross-team
exercises that simulate a peak event from start to finish, including parts that load testing
alone does not cover.

- Deliberately shut down a whole region in the middle of the exercise, and check that buyer
  failover (Section 6.4) and reconciliation of in-progress sagas (Chapter 10) both actually work,
  not just work in theory.
- Overload the payment provider's test environment, and check that the on-call runbook, alert
  thresholds, and dashboards, not just the underlying code, correctly show "payment tier
  degraded" fast enough for a human to react.
- Rehearse pre-scaling: actually run through the "scale up 60 minutes before the event" checklist
  from start to finish on a schedule, including the step of notifying the payment provider
  (Section 4.4). This way, it becomes a tested, trusted runbook by the time a real Black Friday
  arrives, instead of a document nobody has actually followed. Lessons from each game day, and
  from each real peak event, feed back into the next round of capacity planning and load-test
  scenarios. Capacity planning at this scale is a continuous loop, not a number calculated once
  in Chapter 2 and never looked at again.

## Interview Tips

- **Start with the reframe, not the mechanics.** When asked "how does checkout scale to 50,000
  orders per second," the strongest opening is not "we add more servers." It is naming the real
  problem: the average load is easy, and the flash-sale spike aimed at a few hot products is
  hard. Then walk briefly through steady-state scaling before spending most of your time on
  surge handling. Interviewers listen for whether you see this distinction clearly.
- **Pair the waiting room with stock-splitting when talking about flash sales.** These two tools
  solve two different halves of the same problem. The waiting room controls the *rate at which
  new requests enter* the system, protecting everything downstream, including the payment tier.
  Stock-splitting controls *contention once requests are already inside*, protecting the one hot
  inventory row from becoming a bottleneck that no amount of adding servers can fix. Mention both
  together, and be clear about stock-splitting's real trade-off (uneven depletion across
  buckets, fixed with a small backup pool). Naming that trade-off yourself, without being asked,
  is a strong signal of seniority.
- **On multi-region, the one sentence to remember is: "strongly consistent data has exactly one
  owning region at a time."** If asked to make checkout active-active, resist describing a
  single globally-consistent database as the answer. Instead, describe *ownership partitioning*:
  data is owned by buyer or by order region, inventory is either pinned to the seller's region
  or split into regional shares ahead of time for global products, and reads are copied
  asynchronously across regions. This mirrors the "strongly consistent for money, loosely
  consistent everywhere else" pattern that runs through this whole handbook, now applied across
  regions.
- **Justify caching decisions with the oversell/double-charge test**, not a generic "cache
  whatever is read often" answer. The strongest move is to bring up the counter-examples
  yourself: "we would never cache inventory counts or payment status, because a stale read there
  is exactly the failure this whole system is built to prevent."
- **If asked "what breaks first at 2x the stated peak," give a specific answer.** Do not just say
  "everything degrades gracefully." Name a real bottleneck, such as the payment connection pool
  from Section 2.1, or the shared backup pool used in stock-splitting, and describe exactly how
  it degrades, tying it back to the load-shedding order in Section 4.3. Interviewers ask this
  question specifically to tell apart candidates who understand the design from those just
  reciting a list of tools.

## Key Takeaways

- Scaling checkout is really two separate problems: **steady, spread-out load** (about 2,300
  orders per second on average, solved by adding more stateless instances and sharding the Order
  DB), and **spiky, concentrated flash-sale load** (about 50,000 orders per second at peak, aimed
  at a few hot products, solved by admission control, queueing, and stock-splitting). Mixing
  these two up leads to over-building the easy half and under-building the hard half.
- The stateless layer (Orchestrator plus services) scales by simply adding more instances. The
  one tricky part is sizing payment connection pools using **Little's Law (L = lambda x W)**,
  based on tail latency and peak arrival rate, because the payment call dominates both the time
  budget and the concurrency budget of the whole saga.
- The Order DB is split into about 1,024 shards by `order_id`, for write scaling. But this makes
  buyer order-history queries harder. The fix is a **CQRS read store keyed by `buyer_id`**, fed
  asynchronously through Kafka and change-data-capture, deliberately allowed to lag slightly, per
  the design brief's consistency rules. **Hot, warm, and cold tiers** keep the expensive live
  tier small (about 90 TB), while 7 years of required legal retention lives in cheap cold
  storage.
- Flash-sale surges are handled with **admission control and a virtual waiting room** (which
  controls the rate of entry, not raw concurrency), a strict **load-shedding priority order**
  (the money path is never dropped; analytics is dropped first), **pre-scaling ahead of known
  events** (because reactive autoscaling is 1 to 3 minutes too slow for a 10-second cliff), and
  explicit **circuit breakers and backoff** to stop a struggling payment provider from being
  turned into a fully broken one by a retry storm.
- **Stock splitting, per-sub-bucket serialization, and in-memory counters** (Chapter 7's
  correctness tools) also work, from a scaling point of view, by turning one impossibly hot
  partition into K parallel ones, and by shielding the durable Inventory DB from the over 99% of
  flash-sale demand that was always going to be rejected anyway.
- Multi-region active-active design works by giving every strongly consistent piece of data
  **exactly one owning region**, pinned by buyer residency and data-locality rules, with async
  cross-region copies used only for reads. Inventory is usually pinned to the seller's warehouse
  region. Truly global products need an explicit choice between single home-region ownership and
  splitting stock into regional shares ahead of time. Neither comes free from some automatic
  distributed-consensus mechanism.
- Cache anything that does not move money, like the catalog, shipping rates, and tax rates,
  aggressively, with event-driven invalidation. Never cache inventory counts, payment status, or
  idempotency records. Those must stay authoritative and immediately correct, with no exceptions.
- Async offloading through Kafka, using the outbox pattern, keeps notifications, the fulfillment
  handoff, and analytics completely off the 2.5-second place-order path. This isolates their
  backpressure, so a slow downstream consumer can never slow down, or break, checkout.
- None of this works without **continuous validation**: steady-state headroom (about 30 to 40%
  utilization), load tests run past the intended peak to confirm graceful degradation instead of
  collapse, and recurring game days that rehearse the pre-scaling runbook and failure responses
  before a real peak event forces us to get it right on the first try.
