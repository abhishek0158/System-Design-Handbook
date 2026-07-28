# Chapter 12 — Mock Interview

This chapter shows a full 45-minute system design interview. The question is the
one this whole handbook is about: **"Design the checkout system for a global
e-commerce marketplace."**

You will read a real-style conversation between an interviewer and a candidate.
The candidate does a good job. She asks questions first. She states her
assumptions. She puts correctness before speed. She derives the numbers. She
draws the design. She goes deep on the hard parts. She handles push-back well.

After each important moment you will see a short note like this:

> **Commentary:** This explains why the answer was good, what a weak answer would
> look like, and what the interviewer wants to hear.

At the end you get a scorecard, a hire decision, and a "How to Practice"
section. All of it uses simple English so it is easy to follow.

---

## 12.0 How to read this chapter

The interview follows the same design we built across Chapters 1 to 11. So the
numbers, names, and decisions match the rest of the handbook. Nothing new is
invented here. The goal is to show you how to *say* the design out loud, in a
calm and clear way, in 45 minutes.

The candidate is called **Priya**. The interviewer is called **Sam**. Sam is a
senior engineer at GlobalMart.

---

## 12.1 The interview format

Before the transcript, here is how a senior system design interview is scored.
Knowing this helps you spend your time on the right things.

An interviewer usually grades you on **eight areas**:

1. **Requirements** — Did you scope the problem? Did you find the functional and
   non-functional needs? Did you set priorities?
2. **Estimation** — Can you do quick back-of-envelope math? Orders per second,
   storage per day, and so on.
3. **High-level design** — Can you draw a clear architecture with the main
   services and how data flows?
4. **Deep dive** — Can you go deep on one or two hard parts and show real
   understanding?
5. **Correctness reasoning** — Can you argue *why* your design never
   double-charges and never oversells, even during failures?
6. **Scaling** — Can you handle 10x to 20x traffic spikes and large data?
7. **Trade-offs** — Can you compare options and explain your choice? For example,
   saga versus two-phase commit.
8. **Communication** — Are you clear, organised, and easy to follow? Do you
   listen and adjust?

> **Commentary:** Notice that **correctness** is its own area here. For most
> systems it is folded into the deep dive. But for checkout, money correctness is
> the whole point. A candidate who builds a fast system that can double-charge
> will fail, no matter how clever the rest is.

### A rough timeline for 45 minutes

| Time | Phase | What happens |
|---|---|---|
| 0–5 min | Requirements | Ask questions, agree on scope and priorities |
| 5–10 min | Estimation | Derive orders/sec, storage, peak load |
| 10–20 min | High-level design | Draw the architecture and the place-order flow |
| 20–35 min | Deep dives | Inventory (no oversell) and payments (no double-charge) |
| 35–42 min | Scaling and failure | Flash sales, PSP outage, reconciliation |
| 42–45 min | Trade-offs and wrap-up | Saga vs 2PC, CP vs AP, what you would do next |

> **Commentary:** You do not need to hit these times exactly. But you must not
> spend 20 minutes on requirements and leave no time for the deep dive. A good
> candidate watches the clock and keeps things moving.

---

## 12.2 The transcript

### Phase 1 — Requirements (0–5 min)

**Interviewer (Sam):** Hi Priya, thanks for coming in. Today I would like you to
design the checkout system for a large global e-commerce marketplace. Think of
something at the scale of Amazon or AliExpress. Where would you like to start?

**Candidate (Priya):** Thanks Sam. Before I design anything, I want to make sure
I understand the problem. Can I ask a few questions first?

**Interviewer:** Please do.

**Candidate:** First, what is checkout here? I will treat it as the flow that
turns a shopping cart into one or more paid, confirmed orders. It starts when the
buyer clicks "checkout" and ends when the order is placed and paid. Is that the
right scope?

**Interviewer:** Yes, that is right.

**Candidate:** Second, this is a marketplace. So one cart can have items from
many different sellers. Do I need to handle that?

**Interviewer:** Good catch. Yes. A single cart often has items from three or
four sellers.

**Candidate:** Okay. That is important. It means one checkout can become several
orders, one per seller. I will come back to that.

Third, what is *out* of scope? I assume I do not need to build the product search
or the catalog. I also assume the warehouse and shipping carrier systems are
separate. I will just hand off a "the order is placed" event to them. And I will
mention refunds only as a way to undo a payment, not build the full returns
screen. Is that fair?

**Interviewer:** That is fair. Focus on the checkout path itself.

> **Commentary:** This is strong opening behavior. Priya does not jump into
> drawing boxes. She defines the problem, spots the multi-seller twist on her own,
> and cuts scope so she does not waste time. A weak candidate starts drawing an
> architecture in the first minute and never confirms what "checkout" even means.

**Candidate:** Now the most important question. What matters most for this
system? I want to agree on priorities before I design, because the priorities
shape every later choice.

For checkout, I believe the top priority is **correctness with money and
inventory**. We must never charge a buyer twice. We must never sell the same last
unit to two buyers. We must never lose an order that a buyer paid for. Would you
agree that correctness comes first, even above speed?

**Interviewer:** I agree. Say more about why.

**Candidate:** Because a mistake here is not just a slow page. It is real money
and real trust. If we double-charge a customer, we lose their trust and we may
face legal problems. If we oversell, a seller cannot fulfill and we must cancel a
paid order, which is a terrible experience. So correctness is the top
non-functional need.

After correctness, I would rank:
- **High availability.** Checkout down means lost sales every second. I will aim
  for 99.99% uptime, which is about 53 minutes of downtime per year.
- **Low latency,** but only after the first two. The review page should feel
  fast. The final "place order" can be a little slower because it must talk to an
  outside payment company.
- **Surge tolerance.** Flash sales and events like Black Friday can bring 10x to
  20x normal traffic on a few hot products.

**Interviewer:** Good. Let me note the latency targets you would set.

**Candidate:** For the checkout review pages, I want p99 under 300 milliseconds.
That means 99% of requests are faster than 300 milliseconds. For the final place
order call, I will allow p99 up to about 2.5 seconds. The reason is that this
call waits for an outside payment provider to approve the card. That step alone
can take 300 to 1500 milliseconds and we do not control it.

**Interviewer:** You said 99.99% uptime. What does that actually cost you if you
miss it?

**Candidate:** Let me turn it into real numbers. 99.99% means about 53 minutes of
downtime allowed per year. At an average of 2,300 orders per second, one minute of
full downtime loses about 138,000 orders. At roughly 60 dollars per order, that is
around 8 million dollars of lost sales per minute of downtime. So the availability
budget is not abstract. Each minute has a clear price. This is why I will spend
effort on graceful degradation, so a partial failure does not become full downtime.

> **Commentary:** Priya converts an SLO into money per minute. This shows she
> understands *why* the 99.99% target exists. Connecting reliability to revenue is
> exactly the business-aware reasoning senior interviews reward. A weak candidate
> quotes "four nines" without knowing what a breach costs.

> **Commentary:** Priya sets numbers, not vague words. She also explains *why* the
> place-order target is looser. This shows she understands that the slow part is
> the external payment authorization, not our own code. Naming the exact targets
> early gives her something to design against later.

**Candidate:** Let me quickly list the main functions I need to support:

- Create a checkout session from a cart, with correct prices, tax, and shipping.
- Let the buyer pick a shipping address and method.
- Re-price everything at checkout, because prices and promotions can change.
- Attach a payment method safely.
- Place the order. This is the key operation. It must be safe to retry.
- Reserve inventory so two buyers cannot grab the last unit.
- Track order status over time.
- Send a "the order is placed" event to other systems.

**Interviewer:** That is a good list. Before math, tell me the consistency stance
in one sentence, since you keep saying correctness first.

**Candidate:** Happy to. For inventory, payment, and order creation I choose
**strong consistency**. Strong consistency means every reader sees the latest
write, so two buyers cannot both see the last unit as available. For order
history, dashboards, and analytics I choose **eventual consistency**, which means
readers may see slightly old data for a short time. That is fine there. So the
system is strict where money and stock live, and relaxed everywhere else.

**Interviewer:** Good. That is the CP versus AP line. Keep it in mind. Now math.

> **Commentary:** Priya compresses the whole consistency philosophy into two
> sentences and defines both terms plainly. Stating this early means every later
> decision has a rule to point back to. When she later rejects 2PC and designs the
> hot-SKU path, she is just applying this one stance.

---

### Phase 2 — Estimation (5–10 min)

**Interviewer:** Give me a sense of scale. How big is this?

**Candidate:** Let me do the math out loud. I will start from a daily order
number and work down to per-second rates.

Say we do about **200 million orders per day**. Let me turn that into a rate.
There are 86,400 seconds in a day. So:

```
200,000,000 orders / 86,400 seconds ≈ 2,300 orders per second (average)
```

So on an average day, we place about **2,300 orders per second**.

**Interviewer:** And at peak?

**Candidate:** Peak is much higher. Flash sales and holidays concentrate traffic.
I will assume peak is about 20 times the average. So:

```
2,300 × 20 ≈ 46,000 ≈ ~50,000 orders per second at peak
```

So I will design for a peak of about **50,000 orders per second**. This peak is
the number that stresses the system, so I will keep it in mind.

**Interviewer:** What about checkout sessions, not just final orders?

**Candidate:** Many people start checkout but do not finish. Cart abandonment is
real. I will assume only about a third of sessions become orders. So we have
about **600 million sessions per day**, which is roughly **7,000 sessions per
second** on average.

Also, each session makes several API calls. Create the session, set shipping, get
a quote, set payment, review, and place. That is about 6 calls per session. But
during a flash sale, traffic bunches up on the place-order call. So the peak
checkout API load can reach around **300,000 requests per second**.

**Interviewer:** Now storage. How much data do orders create?

**Candidate:** Let me size one order record. It holds line items, addresses,
payment references, status history, and the per-seller sub-orders. I will estimate
about **5 kilobytes** per order.

```
200,000,000 orders/day × 5 KB ≈ 1 TB of order data per day
1 TB/day × 365 ≈ ~365 TB per year (raw)
```

Financial rules mean we must keep this for about **7 years**. So the cold storage
grows into the multi-petabyte range. But the "hot" data that we read often is
just the last 90 days or so, which is roughly **90 TB**. I will keep hot data on
fast storage and push old data to cheaper cold storage.

**Interviewer:** And the idempotency records you mentioned?

**Candidate:** Payment transactions are about 220 million per day, a bit more than
orders because of retries. Each idempotency record is around 1 kilobyte and we
only need to keep it for a day or two. So that is roughly **250 to 400 gigabytes**
of hot data. I will keep it in Redis with a durable backup. Redis is a fast
in-memory store.

> **Commentary:** This is textbook estimation. Priya starts from one anchor number
> (200M orders/day) and derives everything else with simple division and
> multiplication. She says the numbers out loud so the interviewer can follow. She
> also connects each number to a design need: the 50K peak drives scaling, the
> 90 TB hot tier drives storage tiers, and the idempotency size drives the Redis
> choice. A weak candidate either skips the math or produces random big numbers
> with no reasoning.

---

### Phase 3 — High-level design (10–20 min)

**Interviewer:** Good. Now show me the architecture.

**Candidate:** Let me draw the main pieces first, then walk through the
place-order flow. I will keep one idea central: a component I call the **Checkout
Orchestrator**. It runs the checkout as a **saga**.

Let me define saga in one sentence. A **saga** is a sequence of small steps, where
each step has an "undo" action, so if a later step fails we can cleanly undo the
earlier ones. We use a saga because we cannot wrap payment, inventory, and orders
in one big database transaction. They live in different services and even outside
companies.

Here is the architecture.

```
 Client
   │
   ▼
 CDN / Edge
   │
   ▼
 API Gateway   (auth, TLS, rate limit)
   │
   ▼
 Checkout Orchestrator  (the SAGA coordinator; keeps no long-lived state; idempotent)
   │
   ├─▶ Cart Service
   ├─▶ Pricing & Promotions Service
   ├─▶ Tax Service
   ├─▶ Inventory Service ──▶ Inventory DB (reservations, sharded by SKU)
   ├─▶ Payment Service ──▶ PSP Adapters ──▶ external payment providers (PSPs)
   ├─▶ Order Service ──▶ Order DB (sharded by order_id, strongly consistent)
   └─▶ Idempotency Store (Redis + durable backup)
   │
   ▼
 Kafka  (order.placed events, saga events, outbox)
   │
   ▼
 Notification Service · Fulfillment (downstream) · Analytics
```

Let me name the key parts in plain words:

- **CDN / Edge and API Gateway.** The gateway checks the login token, ends the
  secure connection, and limits abusive traffic.
- **Checkout Orchestrator.** This is the brain. It runs the saga steps in order.
  It holds no long state itself, so any instance can pick up any request. That
  makes it easy to scale and restart.
- **Cart, Pricing, Tax Services.** These build the priced session.
- **Inventory Service.** This holds and releases stock. It is the core of
  no-oversell.
- **Payment Service and PSP Adapters.** A **PSP** is a Payment Service Provider,
  an outside company like a card processor. The adapter is the code that talks to
  each PSP.
- **Order Service and Order DB.** This is the durable source of truth for orders.
- **Idempotency Store.** This remembers past requests so retries are safe.
- **Kafka.** A durable log of events. We use it to tell other systems that an
  order was placed, without blocking checkout.

**Interviewer:** Why put a single orchestrator in the middle? Why not let the
services talk to each other directly?

**Candidate:** Good question. There are two styles. One is **choreography**, where
each service listens for events and reacts, with no central boss. The other is
**orchestration**, where one coordinator calls each step in order.

I chose orchestration for checkout for one main reason: the flow is complex and
must undo cleanly on failure. With a central orchestrator, the whole order of
steps and undo steps lives in one place. I can read it, test it, and reason about
it. With choreography, the logic is spread across many services and it is very
hard to answer the question "if step 3 fails, what exactly gets undone?" For a
money path, that clarity is worth a lot.

> **Commentary:** This is exactly the trade-off the interviewer wants. Priya names
> both patterns, defines them simply, and gives a *reason* tied to correctness, not
> just "I like orchestration." The reason (undo logic in one readable place) is the
> real one senior engineers use.

**Interviewer:** Quickly, what are the main data records behind this?

**Candidate:** Four core records. First, the **CheckoutSession**. It is temporary,
lives about 30 minutes, and holds the cart snapshot, the per-seller sub-orders, the
totals, and the session version. Second, the **Order**, which is durable and the
source of truth. There is one order per seller, and they are grouped by a shared
`checkout_group_id`. Third, the **InventoryReservation**, a transient hold with
state HELD, COMMITTED, or RELEASED and a 15-minute TTL. Fourth, the
**PaymentAttempt**, which tracks the money with state AUTHORIZED, CAPTURED, VOIDED,
or FAILED and the provider reference. Plus the **IdempotencyRecord** that ties a
key to its saved response.

The important design point is the `checkout_group_id`. One cart from three sellers
becomes three orders under one group id. That grouping is what lets the saga handle
partial success, where one seller succeeds and another is out of stock.

> **Commentary:** Priya keeps the data-model answer short but hits the load-bearing
> detail: the `checkout_group_id` that links sub-orders and enables partial success.
> She does not dump every field. In a 45-minute interview, naming the records and
> the one relationship that matters is the right depth.

**Interviewer:** Walk me through what happens when the buyer clicks "place order."

**Candidate:** Sure. This is the saga. I will draw it as a state machine and then
explain each step.

```
                 place-order (with Idempotency-Key)
                              │
                              ▼
                    ┌───────────────────┐
                    │ 1. REVALIDATE     │  prices, promos, tax still valid?
                    └─────────┬─────────┘
                              ▼
                    ┌───────────────────┐   fail ──▶ (nothing to undo) ─▶ FAIL
                    │ 2. RESERVE stock  │  HELD, per sub-order, TTL 15 min
                    └─────────┬─────────┘
                              ▼
                    ┌───────────────────┐   fail ──▶ release reservations ─▶ FAIL
                    │ 3. AUTHORIZE pay  │  hold funds on card (not captured yet)
                    └─────────┬─────────┘
                              ▼
                    ┌───────────────────┐   fail ──▶ void auth, release stock ─▶ FAIL
                    │ 4. CREATE order(s)│  Order DB, status CONFIRMED; commit stock
                    └─────────┬─────────┘
                              ▼
                    ┌───────────────────┐
                    │ 5. CAPTURE + emit │  take the money; emit order.placed
                    └─────────┬─────────┘
                              ▼
                           CONFIRMED
```

The five steps are:

1. **Revalidate the session.** Check that prices, promotions, and tax are still
   correct. Check the buyer is still allowed to buy. Catalog prices can change
   between adding to cart and paying.
2. **Reserve inventory.** For each seller's sub-order, ask the Inventory Service
   to hold the stock. The hold has a short life, about 15 minutes. If we do not
   finish, the hold expires and the stock returns.
3. **Authorize payment.** Ask the payment provider to hold the money on the card
   for the grand total. This is a hold, not a charge yet.
4. **Create the orders.** Write the order records to the Order DB with status
   CONFIRMED. Turn the inventory holds into real commits.
5. **Capture payment and emit the event.** Take the money for real. Then publish
   an `order.placed` event to Kafka so fulfillment, email, and analytics can act.

If any step fails, we run the **compensations** in reverse. We void the payment
hold. We release the reservations. We cancel the order. Every step is idempotent
and keyed by the checkout group id plus the idempotency key.

**Interviewer:** Why reserve stock *before* you authorize payment? Why that order?

**Candidate:** Because the cheap, reversible step should come first. Reserving
stock is a local operation in our own database. It is fast and easy to undo.
Authorizing payment calls an outside company and involves real money. If I
authorized payment first and then found the item was out of stock, I would have
to reverse a money operation, which is slower and riskier. So I check the thing
most likely to fail and easiest to undo first. I only touch money once I know the
stock is held.

> **Commentary:** The ordering question is a favorite. Priya's answer shows she
> thinks about failure cost, not just the happy path. "Do the cheap reversible
> thing before the expensive irreversible thing" is exactly the principle. A weak
> candidate picks an order at random and cannot defend it.

**Interviewer:** You keep saying "revalidate the session." Why? The buyer already
saw a price when they added to cart.

**Candidate:** Because the price the buyer saw can be stale. Time passes between
adding to cart and paying. A promotion may have ended. A seller may have changed
the price. Tax depends on the final shipping address, which the buyer may set only
at checkout. So the catalog price the buyer saw is just a snapshot, not a promise.

At place-order I re-fetch the authoritative price, promotions, and tax and compute
the true grand total. If it differs from what the buyer last saw, I do not silently
charge more. I stop and show the new total for the buyer to confirm. This protects
the buyer from surprise charges and protects us from selling below cost by mistake.

I attach a `session_version` to each session. Every change bumps the version. At
place-order the client sends the version it last saw. If the server version is
newer, I know something changed and I force a re-review. This is optimistic
concurrency: I do not lock the session, I just detect a stale version and react.

> **Commentary:** Revalidation is easy to forget, and forgetting it causes real
> money bugs. Priya explains the *why* (stale snapshot, tax depends on address) and
> then adds the `session_version` mechanism for detecting change without locking.
> This ties back to her correctness-first stance: charge the true amount, never a
> stale one.

---

### Phase 4 — Deep dive 1: Inventory reservation, no oversell (20–28 min)

**Interviewer:** Let's go deep. Two buyers, one unit left. Walk me through it, step
by step. I do not want either one to buy a unit that is not there.

**Candidate:** This is the heart of no-oversell. Let me define the goal clearly.
Only one buyer should get the last unit. The other should be told it is out of
stock. And we must never let both succeed, even if their requests arrive at the
exact same millisecond.

The key idea is that the inventory decrement must be **atomic**. Atomic means the
"check if enough stock, then reduce it" happens as one indivisible step, so two
requests cannot both see "1 left" and both take it.

Here is the flow for the last unit:

```
Buyer A                         Inventory DB (row for SKU X: available = 1)          Buyer B
   │                                                                                    │
   │ reserve 1 of SKU X ─────────▶                                                      │
   │                          ┌────────────────────────────────────────┐               │
   │                          │ atomic: UPDATE ... SET available = 0    │               │
   │                          │ WHERE sku = X AND available >= 1        │               │
   │                          │ rows affected = 1  ✓                    │               │
   │                          └────────────────────────────────────────┘               │
   │  ◀──── reservation HELD, id=r1                                                      │
   │                                                     ◀───────── reserve 1 of SKU X   │
   │                          ┌────────────────────────────────────────┐               │
   │                          │ atomic: UPDATE ... SET available = ...  │               │
   │                          │ WHERE sku = X AND available >= 1        │               │
   │                          │ rows affected = 0  ✗ (available is 0)   │               │
   │                          └────────────────────────────────────────┘               │
   │                                                     ──────▶ OUT OF STOCK            │
```

So Buyer A's request runs the atomic update first. It sets available from 1 to 0
and returns "1 row changed." A gets a reservation. Buyer B's request runs the same
update, but now the `WHERE available >= 1` check fails, so "0 rows changed." B is
told out of stock. The database row lock makes sure only one of them can win. No
oversell.

**Interviewer:** What holds the row so they cannot both win?

**Candidate:** The database gives us a row-level lock inside the transaction. When
A's update touches that row, B's update must wait until A's is done. By the time
B runs, available is already 0, so B's condition fails. This is a **conditional
update**: we only reduce stock if enough is still there. We check the row count to
know if we won.

I also store the hold as a reservation record with state HELD and a TTL. **TTL**
means time-to-live, a timer after which the hold expires. If the buyer never
finishes, the hold auto-releases after about 15 minutes and the unit goes back to
available. This stops abandoned carts from locking stock forever.

**Interviewer:** And when the order is confirmed?

**Candidate:** At step 4 of the saga, I commit the reservation. The state moves
from HELD to COMMITTED. The stock is now truly sold. If the saga fails before
that, I release the hold, state RELEASED, and available goes back up. So a
reservation has three states: HELD, COMMITTED, RELEASED.

```
   reserve            commit (order created)
  ────────▶  HELD  ─────────────────────────▶  COMMITTED
              │
              │ release (failure or TTL expiry)
              ▼
          RELEASED  ─▶ stock returns to available
```

> **Commentary:** This is a very strong deep dive. Priya reduces "no oversell" to
> one core mechanism: an atomic conditional update guarded by a row lock, checked
> by rows-affected. She draws the race and shows exactly why the loser loses. Then
> she adds the reservation lifecycle and TTL for abandoned carts. A weak candidate
> says "use a lock" but cannot show what happens at the exact moment of the race,
> or forgets that abandoned holds must expire.

---

### Phase 5 — Deep dive 2: Payments, no double-charge (28–35 min)

**Interviewer:** Now the money side. The client places an order, gets no response
because the network hiccuped, and retries. How do you stop a double charge?

**Candidate:** This is the most important correctness case, so let me be careful.
The tool is the **idempotency key**.

Let me define it simply. An **idempotency key** is a unique id the client
generates for one logical "place order" action. The client sends the same key on
the first try and on every retry. Our server uses that key to make sure the action
happens **at most once**, no matter how many times the request arrives.

Here is how it works:

```
place-order  { Idempotency-Key: K, body: ... }
        │
        ▼
 Look up K in the Idempotency Store
        │
   ┌────┴─────────────────────────┐
   │ K not seen before            │  K already seen
   ▼                              ▼
 Save K = IN_PROGRESS         Return the SAVED response
 Run the saga once            (do NOT run the saga again)
 Save K = DONE + response
 Return the response
```

So on the first request, we see the key is new. We mark it IN_PROGRESS, run the
saga, save the result under the key, and return it. On the retry, we see the key
already exists. We do not run the saga again. We just return the exact same saved
response. The buyer sees one order and one charge.

**Interviewer:** What if the retry arrives while the first one is still running?
Both are in flight at the same time.

**Candidate:** Good, that is the hard race. When the first request marks the key
IN_PROGRESS, it does so as an atomic "create only if not exists" write. The second
request tries the same write and fails because the key already exists. So the
second request knows another attempt is running. It can either wait a moment and
then return the finished result, or return a "still processing, please wait"
response. Either way, it does not start a second saga. So we never authorize the
card twice.

**Interviewer:** Now a nastier one. Your Payment Service calls the provider to
authorize. The provider times out. You do not know if the charge went through or
not. Did we charge the customer?

**Candidate:** This is the classic "unknown result" problem. A timeout does not
mean failure. It means we do not know. The money may or may not be held. So I must
never assume.

I handle it with two ideas.

First, I also send an **idempotency key to the payment provider** on the authorize
call. Real providers support this. So if I retry the authorize with the same key,
the provider returns the same result and does not create a second hold. This makes
the retry safe even toward the outside company.

Second, if the timeout happens and a retry still does not give a clear answer, I do
not guess. I move that payment attempt to a "needs check" state and I query the
provider for the status of that key. This is a **get status** call. The provider
tells me the true state: was it authorized or not? I make my record match the
truth.

So the rule is: on a timeout, retry with the same key, and if still unsure, ask the
provider for the truth. Never blindly re-charge, and never silently drop the order.

**Interviewer:** How do you store the idempotency record so it survives a crash?
Redis is in memory. If it restarts, do you lose the keys and start double-charging?

**Candidate:** Good worry. I do not rely on Redis alone for correctness. Redis is
the fast path, the first place I check. But I also write the idempotency record to
a durable store, so it survives a restart. The state machine for a key has three
states: IN_PROGRESS, DONE, and FAILED. I write IN_PROGRESS durably before I start
the saga. So even if the whole service crashes right after, a retry finds the key
in IN_PROGRESS and knows not to start a fresh saga.

There is one more subtle rule. I store a **request fingerprint** with the key. The
fingerprint is a hash of the important request fields, like the cart, the amount,
and the buyer. If a retry arrives with the same key but a *different* body, that is
a client bug or an attack, and I reject it instead of returning the old result.
This stops someone reusing a key for a different order.

> **Commentary:** The interviewer probes whether Priya treats Redis as a cache or
> as the source of truth. She correctly says it is a cache in front of a durable
> record, and that she writes IN_PROGRESS durably *before* running the saga. The
> request fingerprint detail is a senior touch: an idempotency key must match the
> same request, or it is not really idempotent.

**Interviewer:** And if you never get a clear answer in time?

**Candidate:** Then I fail safe toward the customer. I do not confirm the order and
I release the inventory. If it later turns out a hold was created at the provider,
my reconciliation job finds it and voids it, so the customer is not charged for an
order they did not get. I would rather cancel a possible order than risk a charge
with no order behind it.

> **Commentary:** This is excellent. Priya separates "timeout" from "failure" out
> loud. That single distinction is what most candidates miss. She then gives a
> concrete recovery: retry with the same key, then query status, then fall back to
> reconciliation. She also states her fail-safe direction (protect the customer
> from a charge). This is exactly how senior payment engineers reason.

**Interviewer:** When do you actually take the money? At authorize or later?

**Candidate:** I split it into two steps: **authorize** and **capture**.
Authorize places a hold on the funds. Capture actually takes them. I authorize
during checkout so I know the money is good. I can capture right away, or defer
capture until the item ships. Deferring is common because you should charge when
you fulfill. If I authorize but then cannot fulfill, I void the authorization and
the customer is never charged. This two-step design gives me a clean undo before
capture.

**Interviewer:** One more. The buyer pays with a gift card plus a credit card. The
gift card covers 20 dollars and the card covers the rest. How does that fit your
saga?

**Candidate:** A split payment is just more than one payment step inside the same
authorize stage. I treat each method as its own PaymentAttempt with its own
idempotency key, all under the same `checkout_group_id`. So I redeem 20 dollars
from the gift card and authorize the remaining amount on the card.

The important rule is all-or-nothing across the methods. If the card authorization
fails after the gift card redemption succeeds, I must undo the gift card. So the
gift card redemption has a compensation, which is a refund back to the gift card.
This is the same saga idea applied inside the payment step. Each sub-step has an
undo. I only move forward once every method succeeds. If any method fails, I
reverse the ones that already succeeded and fail the whole payment.

> **Commentary:** Split payment is a curveball, but Priya folds it into the pattern
> she already has. Each method is a keyed sub-step with its own compensation, and
> the group is all-or-nothing. She does not invent a new mechanism. Reusing the saga
> and idempotency ideas for a new case shows the design is coherent.

---

### Phase 6 — A moment of push-back (the 2PC detour) (35–37 min)

**Interviewer:** Step back. You have inventory in one database, payment at an
outside provider, and orders in another database. Why not just wrap all of this in
one distributed transaction, a two-phase commit, so it all commits or all rolls
back together? That sounds simpler and safer.

**Candidate:** My first instinct is that a single all-or-nothing transaction would
be the cleanest correctness story. Let me think about whether it actually works
here.

Actually, no, I do not think two-phase commit fits this problem. Let me explain
why I am changing my answer.

**Two-phase commit**, or **2PC**, needs every participant to support a shared
prepare-and-commit protocol and to hold locks until the coordinator says commit.
There are three problems here:

1. **The payment provider is an outside company.** I cannot make an external card
   processor join my 2PC and hold a lock waiting for my coordinator. It simply does
   not offer that. So 2PC is impossible across the boundary that matters most.
2. **Locks are held for the whole transaction.** Payment authorization can take up
   to 1.5 seconds. Holding an inventory row lock for that long, during a flash sale
   with 50,000 orders per second on one hot item, would create a huge queue. It
   would destroy availability, which is my second priority.
3. **The coordinator is a single point of failure.** If it dies mid-commit,
   participants are stuck holding locks, unsure whether to commit or roll back.

So 2PC trades away availability and cannot even reach the external provider. Its
strong all-or-nothing promise is not worth those costs here.

Instead I use the **saga plus idempotency** approach I already described. The saga
gives me step-by-step progress with undo actions. Idempotency keys give me
"at most once" effects even with retries. Together they give the same practical
result, exactly-once effect, without holding cross-service locks or needing the
provider to join a protocol. The difference is that the saga reaches a consistent
state *eventually and correctly*, using compensations, instead of instantly with
locks.

> **Commentary:** This is one of the best moments in the interview. Priya first
> leans toward 2PC, which sounds safe. Then, when pushed, she does not get
> defensive. She re-examines it and changes her answer with clear reasons. Showing
> that you can update your view under scrutiny is a strong senior signal. Even
> better, her three reasons are the real ones: external participant, long-held
> locks under load, and coordinator failure. She lands on saga plus idempotency and
> explains what you give up (instant consistency) and what you keep (correctness).

---

### Phase 7 — Scaling and flash sales (37–41 min)

**Interviewer:** Black Friday. Fifty thousand orders per second, and they are all
fighting over one hot SKU, say a popular game console. What breaks first, and how
do you fix it?

**Candidate:** The first thing that breaks is the single inventory row for that
SKU. Every one of those 50,000 requests per second wants to do an atomic update on
the exact same row. That row becomes a **hot spot**. A hot spot is one piece of
data that everyone hits at once, so it becomes a queue and a bottleneck. My general
sharding by SKU does not help here, because all this traffic is on one SKU, so it
all lands on one shard.

Let me fix it with a few techniques.

First, **admission control and a queue at the front.** For a known hot sale, I do
not let all 50,000 requests hit the inventory row at once. I put them through a
queue or a virtual waiting room. If only 10,000 units exist, I do not need a
million requests pounding the row. I can let a controlled number through. This is a
form of graceful degradation. Buyers see "you are in line" instead of errors.

Second, **split the hot counter into buckets.** Instead of one row with
"available = 10,000", I split it into, say, 50 buckets of 200 each. A request
hashes to one bucket and decrements that bucket. Now the contention is spread
across 50 rows instead of one. When a bucket hits zero, the request can try
another bucket or be told out of stock. This trades a little complexity for much
higher throughput on the hot item.

Third, **reserve stock in memory near the inventory service** for the hot SKU,
backed by the durable store, so the atomic decrement is a fast in-memory operation
with periodic durable checkpoints. The key rule stays: the decrement is still
atomic, so we still never oversell.

**Interviewer:** Does the payment provider become a bottleneck too?

**Candidate:** It can. We send a burst of authorize calls to the PSP. Two
defenses. First, we have **multiple PSP adapters** and can spread load across more
than one provider. Second, we can **defer capture** and even smooth the authorize
load a little using the waiting room, since we already control admission. Also,
because place-order is idempotent and the saga can pause and resume, a slow PSP
makes checkout slower but not wrong.

**Interviewer:** And the overall system, how do you scale the rest?

**Candidate:** The Orchestrator holds no long state, so I just run more copies
behind the gateway. The Order DB is sharded into about 1,024 shards by a hash of
the order id, so writes spread evenly. Hot shards sit on fast NVMe storage. For
multi-region, I run active-active, but I keep the inventory for a given SKU owned
by one region or one authority so the atomic decrement stays correct. Cross-region
strong consistency on every SKU would be too slow, so I localize the authority.

**Interviewer:** Say a buyer in Europe wants a SKU whose authority lives in the US
region. What happens?

**Candidate:** The reservation for that SKU must go to the owning region, because
that is where the one true count lives. So the European checkout makes a
cross-region call just for the reserve step. That call is slower, maybe an extra
100 to 150 milliseconds. But it keeps the atomic decrement in one place, so we
never oversell. Everything else, the session, pricing, and order write, can stay
local to Europe. So only the one step that needs the global truth pays the
cross-region cost.

For SKUs that are hot in one region, I place their authority in that region, so
most traffic is local. If a SKU is hot worldwide, I can move to the bucketed
counter and even split buckets across regions, with each region owning some
buckets. The rule never changes: the decrement stays atomic per bucket, so the sum
can never go below zero.

> **Commentary:** Priya does not pretend multi-region is free. She isolates the one
> step that needs global truth (the reserve) and pays the latency only there, while
> keeping the rest local. This is the "localize the expensive consistency" pattern.

**Interviewer:** How many buckets would you use for the console at 50K per second?

**Candidate:** Let me reason from the target, not guess. A single row on good
hardware might handle a few thousand atomic updates per second before it becomes
the bottleneck. Say roughly 2,000 to 5,000 per second per row under contention. To
absorb 50,000 per second, I need at least 50,000 divided by, say, 3,000, which is
about 17 buckets. I would round up for headroom, so around 32 to 50 buckets. Then
I would load-test to confirm the real per-row limit and adjust. The point is I size
the bucket count from the peak target and the measured per-row throughput, not from
a round number I like.

> **Commentary:** Priya derives the bucket count from the 50K target divided by a
> per-row throughput estimate, then adds headroom and says she would measure.
> Deriving a design parameter from the numbers is a strong signal.

> **Commentary (whole phase):** The flash-sale question separates good from great.
> Priya names the exact failure (the single hot row), explains why sharding by SKU
> does not save her, and then gives layered fixes: admission control, counter
> bucketing, and in-memory reservation. Crucially, every fix keeps the atomic
> decrement, so no-oversell survives. She also handles the PSP as a second
> bottleneck. A weak candidate says "add more servers," which does nothing for a
> single hot row.

---

### Phase 8 — Failure handling and partial success (41–43 min)

**Interviewer:** One cart, three sellers. During place-order, one of the three
sellers is out of stock. What happens to the other two?

**Candidate:** This is why the multi-seller point from the start matters. One cart
becomes three sub-orders under one checkout group id. I have a choice, and it is a
product choice, so ideally I confirm it with the product team. But here is a sane
default.

I treat the three sub-orders as independent as much as possible. If seller B is out
of stock, I do not have to cancel sellers A and C. I can confirm A and C, and tell
the buyer that B could not be fulfilled. The payment then only captures for A and
C. Since I authorize and capture carefully, I can authorize the full amount and
then capture only what shipped, or re-authorize the reduced amount.

The alternative is all-or-nothing: if any seller fails, cancel the whole checkout.
That is simpler and some businesses prefer it. But it is worse for revenue and for
the buyer, because two good items get thrown away for one bad one. So my default is
partial success, but I would confirm the business rule.

Either way, the saga's compensations make it clean. For the failed sub-order, I
release its reservation and do not create that order. For the good ones, I commit
and confirm. Nothing is half-done, because each step is idempotent and keyed.

**Interviewer:** What if the payment provider is completely down for ten minutes?

**Candidate:** Then place-order cannot complete, because I refuse to confirm an
order without knowing the money is good. But I can degrade gracefully. I can still
create the session, reserve inventory, and hold the order in a PENDING_PAYMENT
state. I put the payment step on a durable retry queue. When the provider comes
back, the saga resumes from where it paused and completes. The buyer can be told
"your order is being confirmed" rather than getting a hard error. If it never
recovers within the reservation TTL, I release the stock and fail cleanly, so I do
not hold inventory forever.

Failed events also go to a **dead-letter queue**, a DLQ, which is a holding area
for messages that could not be processed. An on-call engineer or an automated job
can inspect and replay them. Nothing is silently lost.

> **Commentary:** Priya turns the partial-failure question into a design strength,
> because she planted the multi-seller sub-order idea early. She gives a default
> (partial success) but flags it as a product decision, which is mature. For the
> PSP outage she shows graceful degradation: pause the saga, hold state, resume
> later, and a clean give-up path via the TTL. She names the DLQ. These are the
> answers that show real operational experience.

**Interviewer:** After the order is confirmed, how do the warehouse and the email
service find out? And what if that step fails?

**Candidate:** I do not call them directly inside the saga. That would tie the
buyer's checkout to systems that have nothing to do with taking the money. Instead,
in the same database transaction that confirms the order, I also write an event row
to an **outbox** table. The outbox is a table where I record events I intend to
publish. A separate process reads the outbox and publishes `order.placed` to Kafka.
Kafka is a durable log, so the event is not lost even if the warehouse is down.

This is the **outbox pattern**. It solves a real problem. If I wrote the order and
then tried to publish to Kafka as two separate steps, a crash in between would
confirm the order but never tell fulfillment. By writing the order and the event in
one transaction, they always agree. The publisher then delivers at least once, and
each downstream consumer is idempotent, so a repeated event does no harm.

If a consumer keeps failing on an event, that event moves to the dead-letter queue
for a human or a job to inspect and replay. So the order is confirmed and paid the
instant the saga finishes. The downstream work happens reliably in the background,
even if some system is briefly down.

> **Commentary:** Priya reaches for the outbox pattern without being asked, which
> shows she has felt this pain before. The key insight she states plainly is that
> the order write and the event write must share one transaction, or a crash can
> split them. Pairing at-least-once delivery with idempotent consumers is the
> standard reliable-eventing answer.

---

### Phase 9 — Correctness proof: reconciliation (43–44 min)

**Interviewer:** Last correctness question. At the end of the day, how do you
*know* your money records are right? How do you know you did not quietly
double-charge someone or lose an order?

**Candidate:** I do not just trust the live system. I verify it with
**reconciliation**. Reconciliation is a background job that compares our records
against the source of truth and finds any differences.

I run three comparisons regularly:

1. **Our payments versus the PSP's records.** I pull the provider's list of
   authorizations and captures and match them, one by one, to my PaymentAttempt
   records by the provider reference and idempotency key. If the provider shows a
   charge with no matching confirmed order, that is a charge I must refund or a
   hold I must void. If I show a capture the provider does not, that is a bug to
   fix. Any mismatch raises an alert.
2. **Orders versus captured money.** Every CONFIRMED order should have exactly one
   matching capture. Two captures for one order means a double-charge, which I
   refund and then investigate the root cause. Zero captures for a confirmed,
   shippable order means money we should have collected.
3. **Inventory ledger.** Committed reservations should match the stock actually
   sold. This catches oversell drift, for example a hold that both expired and
   committed due to a bug.

The important idea is that idempotency and the saga stop *most* errors in real
time, but reconciliation is my safety net. It catches the rare drift that slips
through, and it repairs it. This is how I can honestly say the books are correct at
the end of the day, not just hope so.

> **Commentary:** Asking "how do you *know* it is correct" is a trap for candidates
> who think idempotency alone is enough. Priya passes cleanly. She treats
> reconciliation as a first-class part of the design, names the three comparisons,
> and says what each mismatch means and how she repairs it. "Prevent in real time,
> detect and repair with reconciliation" is the mature two-layer stance.

---

### Phase 10 — Trade-offs and wrap-up (44–45 min)

**Interviewer:** We are almost out of time. Summarize the big trade-offs you made.

**Candidate:** Three big ones.

First, **saga plus idempotency instead of two-phase commit.** I gave up instant
all-or-nothing consistency. In return I got availability under load and the ability
to include an external payment provider. I reach a correct state with
compensations and reconciliation instead of cross-service locks.

Second, **CP where money and stock live, AP elsewhere.** For inventory
reservation, payment, and order creation, I chose strong consistency. I would
rather reject or slow a request than oversell or double-charge. This is a CP
choice, meaning I favor consistency over availability for that data. But for
order-history reads, seller dashboards, and analytics, I chose eventual
consistency, which is AP, favoring availability. Those can be a little stale
without harm. This is the opposite of a search system, which picks availability
almost everywhere.

Third, **authorize now, capture later.** This adds a two-step payment flow and
some complexity. But it gives me a clean undo window before money actually moves,
which supports partial success and safe cancellation.

**Interviewer:** If you had more time, what would you add?

**Candidate:** I would detail the multi-region ownership of hot SKUs more
carefully, add fraud checks in the flow, and design the seller payout ledger, which
I left out of scope. I would also load-test the hot-SKU bucketing to pick the right
number of buckets.

**Interviewer:** Great. Thank you, Priya.

**Candidate:** Thank you, Sam. I enjoyed it.

> **Commentary:** A crisp close. Priya restates the trade-offs as "what I gave up
> versus what I got," which is exactly the right framing. She repeats the CP-versus-AP
> contrast with search, a theme that ties the whole design together. And when asked
> what is missing, she names real gaps honestly instead of pretending the design is
> perfect. Admitting scope limits at the end builds trust.

---

## 12.3 Scorecard

Here is how Sam would score this interview. Each area gets a score out of 4, where
1 is poor, 2 is below the bar, 3 is at the senior bar, and 4 is above it.

| Area | Score (1–4) | One-line reason |
|---|:---:|---|
| Requirements | 4 | Scoped early, found the multi-seller twist, put correctness first with clear priorities. |
| Estimation | 4 | Derived 2,300 avg and 50K peak orders/sec and storage from one anchor number, out loud. |
| High-level design | 4 | Clear architecture, central orchestrator, defended orchestration vs choreography. |
| Deep dive | 4 | Showed the exact atomic conditional update and the two-buyer race for no-oversell. |
| Correctness reasoning | 4 | Idempotency for retries, timeout ≠ failure, and reconciliation as the safety net. |
| Scaling | 4 | Named the hot-row bottleneck and fixed it with admission control and counter bucketing. |
| Trade-offs | 4 | Compared saga vs 2PC and CP vs AP with real reasons; changed her mind well under push-back. |
| Communication | 4 | Calm, structured, watched the clock, checked assumptions, easy to follow. |

**Overall verdict: Strong Hire.**

Priya cleared the senior bar in every area. Her stand-out strengths were putting
correctness above latency from the first minute, going truly deep on both the
no-oversell race and the no-double-charge flow, and changing her 2PC answer
gracefully when pushed. She never lost sight of the two things that define this
problem: never double-charge, never oversell.

### What would make this even stronger

- **Fraud and abuse.** Checkout is a target for fraud. She left it out of scope. A
  brief mention of where a fraud-scoring step fits in the saga, and how it fails
  safe, would round out the design.
- **Seller payout ledger.** She correctly scoped this out, but a staff-level answer
  would sketch how captured money later settles to each seller, and how that ledger
  reconciles against captures.
- **Backpressure detail.** Her waiting room controls admission, but she did not say
  how the queue itself scales or what the buyer sees when the wait is very long. A
  deeper answer would cover queue limits and a clear "sold out" cutoff.
- **Testing the correctness claims.** She proves no-oversell and no-double-charge by
  design. A staff answer would add how she *tests* them: fault injection, forced
  retries, and chaos on the PSP, plus replaying real traffic against the buckets.

None of these change the verdict. They are the difference between a strong senior
and a staff-level answer.

---

## 12.4 How to Practice

You will not sound like Priya on your first try. That is normal. Here is how to get
there.

**1. Practice the opening out loud.** The first five minutes set the tone. Practice
asking clarifying questions and stating priorities until it feels natural. For
checkout, always lead with "correctness before latency" and say why.

**2. Drill the estimation from one anchor.** Pick one number, like 200 million
orders per day, and practice deriving everything else: orders per second, peak,
storage per day, and hot data size. Do it on paper until the divisions are fast.

**3. Master one deep dive cold.** You cannot be deep on everything in 45 minutes.
Pick the two that matter here: the atomic inventory decrement and the idempotency
key flow. Be able to draw the two-buyer race and the retry race from memory.

**4. Rehearse the failure answers.** Say out loud: "A timeout is not a failure, it
means we do not know." Practice the recovery: retry with the same key, then query
status, then reconcile. These sentences should come out smoothly.

**5. Learn to change your mind well.** Interviewers push on 2PC on purpose. Practice
saying "Let me reconsider that" and then giving clean reasons. Being able to update
your view calmly is a strong signal, not a weakness.

**6. Always end with trade-offs.** Save two minutes to say "here is what I gave up
and here is what I got." Name the CP-versus-AP choice and the saga-versus-2PC
choice. This is what senior interviews reward most.

**7. Time yourself.** Do full 45-minute runs with a friend or a timer. The most
common failure is not wrong answers. It is running out of time before the deep
dive. Watch the clock and keep moving.

**8. Reuse a consistent example.** Just like this handbook uses GlobalMart
everywhere, keep one running example in your head with fixed numbers. It saves you
from re-deriving the scale every time and keeps your answer coherent.

---

## Interview Tips

- Lead with correctness for any money system. Say it in the first two minutes.
- Derive numbers out loud from one anchor. Do not recite memorized figures.
- Draw the saga as a state machine with explicit undo steps. It reads clearly.
- For no-oversell, always show the atomic conditional update and the rows-affected
  check. That is the proof, not the word "lock."
- For no-double-charge, always mention the idempotency key and treat a timeout as
  "unknown," not "failed."
- Name reconciliation. It is how you *prove* correctness, not just claim it.
- When pushed on 2PC, do not dig in. Explain external participants, long locks
  under load, and coordinator failure, then choose saga plus idempotency.

## Key Takeaways

- A senior interview scores eight areas. Correctness is its own area for checkout.
- The whole design turns on two promises: never double-charge, never oversell.
- Saga plus idempotency gives exactly-once effect without cross-service locks.
- No-oversell is an atomic conditional decrement guarded by a row lock.
- No-double-charge is an idempotency key that makes place-order at-most-once.
- Flash sales fail at the single hot row; fix it with admission control and buckets.
- Reconciliation is the safety net that catches and repairs any drift.
- The strong signals are: correctness-first, deep dives, clean trade-offs, and
  changing your mind well under push-back.
