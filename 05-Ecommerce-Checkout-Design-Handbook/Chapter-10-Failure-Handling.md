# Chapter 10 — Failure Handling

Every earlier chapter talked about the happy path. In the happy path, the buyer has a full
cart. The Inventory Service is healthy. The payment provider replies fast. The Order DB
answers in a few milliseconds. Real traffic does not behave this well.

At GlobalMart's scale, about 2,300 orders happen every second on average. At peak times, like
a flash sale, this can reach 50,000 orders per second. At this scale, something is always
a little broken somewhere in the system. A server loses power. A payment provider has a bad
five minutes. A Kafka broker (a message queue system we use to pass events between services)
falls behind. Kafka lets one service publish an event and other services read it later.

This chapter is not about how to prevent failure. We cannot prevent it. This chapter answers a
different question: **what should the system do when a step in the checkout saga fails, and
how do we make sure that failure never corrupts money or stock?** A saga is the multi-step
workflow that runs during checkout: reserve stock, charge the buyer, create the order, and
confirm it. Chapter 6 explained the saga in detail.

Let us restate the main rule from Chapter 1, because every idea in this chapter follows from
it.

> **Fail safe, not fail happy.** When the system is unsure what happened, it should reject the
> checkout, retry safely, or hold the record for later checking. It should never guess in a way
> that could double-charge a buyer or oversell a seller's stock. A failed checkout is a bad
> experience for the buyer. But a double charge or a phantom order for stock that does not exist
> is a trust problem and can even be a legal problem. Speed and uptime can be sacrificed a
> little. Money correctness and stock correctness cannot.

This is the same idea from Chapter 1 about CP versus AP systems, now applied to failures. A CP
system chooses consistency (correctness) over availability when it must pick one. An AP system
chooses availability over consistency. Search (from the companion handbook) is AP. It degrades
by showing slightly old results. Checkout is CP on the money path. It degrades by **refusing to
continue** rather than giving a wrong answer.

---

## 10.1 What Can Go Wrong: A List of Failure Types

Not every failure is the same size. Some are small and heal themselves. Some need an automatic
undo step. A few need a human to step in. It helps to sort failures into types first. This way,
the Checkout Orchestrator (the service that runs the saga, from Chapter 6) can react
automatically most of the time, instead of treating every failure as a brand-new mystery.

| Failure type | Example | How far the damage spreads | What we do by default |
|---|---|---|---|
| **A service is down** | The Tax Service crashes repeatedly | Just that one dependency, but many checkouts use it | Use a circuit breaker, degrade gracefully, or fail fast (§10.7) |
| **Payment provider outage or timeout** | The payment provider (PSP) returns an error or does not reply at all | Every payment authorization | Retry with backoff, switch to a backup PSP, or fail fast (§10.3) |
| **Database failover** | The Order DB's main copy dies, and a backup copy is promoted to be the new main copy | Writes to one shard, for a few seconds | Retry safely until the new main copy is ready (§10.5) |
| **Partial saga failure** | Stock was reserved, but the payment was declined | One `checkout_group_id` (the group of sub-orders in one buyer's checkout) | Undo the completed steps, in reverse order (§10.2) |
| **Poison message** | A broken event that always crashes the service reading it | One group of readers, and it can block everything behind it if not handled | Send it to a Dead Letter Queue and alert someone (§10.4) |
| **Duplicate delivery** | Kafka sends the same `order.placed` event twice | One event is processed twice by a reader | Make the reader logic safe to run twice (§10.4) |
| **Network split** | Region A cannot talk to Region B for a while | Only cross-region copying, not local writes | Each region keeps working correctly on its own; we fix the gap later (Chapter 9) |
| **Orchestrator crash mid-saga** | The orchestrator process dies between reserving stock and charging the card | One or more checkouts that were in progress | Saved saga state plus a recovery process picks it up again (§10.6) |

Notice the pattern: every single row above is handled using the same small set of tools we
already built in earlier chapters. These tools are: idempotency keys (Chapter 8), compensating
actions (Chapter 6), saved state instead of memory-only state, and reconciliation
(Chapter 8, and §10.8 here). An idempotency key is a unique ID attached to a request so that
running the same request twice has the same effect as running it once. A compensating action is
a step that undoes an earlier step's business effect, like releasing a stock hold. This chapter
is mostly about using these same tools everywhere they are needed, not inventing new ones.

---

## 10.2 Partial Saga Failure and How We Undo It

Chapter 6 defined the main saga for placing an order:

```
1. Revalidate the session   (check price, tax, promo, and buyer eligibility are still correct)
2. Reserve stock             (per seller sub-order)         -> stock state becomes HELD
3. Authorize payment         (for the full amount)          -> AUTHORIZED
4. Create the order(s)       (write to Order DB)            -> CONFIRMED, stock becomes COMMITTED
5. Capture payment           (now, or later)                -> CAPTURED; send out order.placed event
```

A "partial saga failure" means the saga finished step *k* successfully, but step *k+1* failed.
Because GlobalMart uses **orchestration**, not **choreography** (explained in Chapter 6), the
Checkout Orchestrator is the one component that decides how to undo things. Orchestration means
one central coordinator runs every step and knows the full plan. Choreography means each service
reacts to events from other services, with no single coordinator. The orchestrator knows exactly
which steps finished for a given checkout, because it saves its progress after every single step
(more on this in §10.6). This means recovery never has to guess whether stock was reserved or
not. It just reads the saved state.

The table below is the most important part of this chapter. It maps each saga step to its
failure mode, how we detect the failure, and how we undo it.

| Saga step | What can fail | How we detect it | How we undo it | Why retrying it is safe |
|---|---|---|---|---|
| 1. Revalidate | Price or tax changed since the buyer last saw the total; buyer is blocked for fraud or region rules | The system compares a `session_version` number and finds a mismatch, or a re-check finds new numbers | Reject the checkout and show the buyer the new total. Nothing else has happened yet, so there is nothing to undo. | Not needed — this step only reads data, it does not change anything |
| 2. Reserve stock | The item is out of stock; the Inventory Service times out; only some of the seller sub-orders got reserved | Inventory Service replies `INSUFFICIENT_STOCK`, or the orchestrator's timer for this step runs out | **Release** any stock holds already made for this checkout. If the cart has many sellers, decide the policy: fail everything, or drop only the unavailable seller's part and continue with the rest | Releasing a hold that is already released, or already expired, simply does nothing. It is a safe no-op. |
| 3. Authorize payment | The payment provider declines the card (not enough funds, fraud rule), the call times out, or the reply is unclear | The Payment Service gets a `DECLINED` reply, or the call times out | **Release the stock holds** from step 2. If the reply was unclear, do not assume the charge went through. Check its real status first (see §10.3) | Retrying `authorize()` with the same key is safe. The provider returns the original result instead of charging twice. |
| 4. Create the order(s) | An Order DB shard (a database partition) is unreachable; writing the same row twice by mistake; one seller's order is written but another seller's order in the same checkout is not | The write fails or times out; or, later, a reconciliation job finds a charged payment with no matching order | **Cancel the payment authorization** and release the stock holds. If some sellers' orders in the same checkout did succeed, do not undo those too. Only undo the ones that failed (see Chapter 8's rules for split payments) | Order creation uses `INSERT ... ON CONFLICT (idempotency_key) DO NOTHING`. A retried write either finds the row already there, or creates exactly one row. |
| 5. Capture payment | Capturing the money fails even though the order is already confirmed; the capture call times out | Payment Service returns `CAPTURE_FAILED`, or reconciliation later finds a confirmed order with no matching capture | Keep the order as confirmed, since the buyer is already expecting it. **Retry the capture** with backoff instead of cancelling. Only cancel the order if retries fail for too long, past the time the authorization hold can stay open | Capture uses the payment ID as its key. A repeated capture call on an already-captured payment simply does nothing new. |

Two points from this table are worth remembering, because interviewers often test exactly these
ideas.

**First, "undo" does not mean deleting a database row.** It means reversing the business effect
in a way we can still see later. Releasing a stock hold does not delete the row. It changes the
row's state from `HELD` to `RELEASED`, and lets a timer clean it up later. This matters because
a competing retry might still be racing to finish that same hold. Cancelling a payment does not
delete the `PaymentAttempt` record either. It marks the record as `VOIDED`. We keep this history
because financial audits require it (Chapter 1's requirement NFR6: every money movement must be
traceable).

**Second, not every step is equally easy to undo.** Steps 1 through 3 are cheap to undo, because
nothing permanent or external has happened yet. Step 5, capture, is different: real money has
already moved by that point. So for step 5, the safer choice flips from "undo it" to "keep
retrying to finish it." This is the same idea Chapter 11 discusses when comparing sagas to
two-phase commit: a saga only works if every single step has a workable undo action, and capture
is the hardest step to undo cleanly, so we avoid undoing it whenever possible.

Here is a simple picture of the saga states and where the undo path branches off:

```
Saga states, with the undo path shown below each forward step

REVALIDATE -> RESERVING -> RESERVED -> AUTHORIZING -> AUTHORIZED -> CREATING_ORDER -> CONFIRMED -> CAPTURING -> CAPTURED
     |            |                        |                          |
     | fails      | fails                  | fails                    | fails
     v            v                        v                          v
  REJECTED   RELEASING -> RELEASED   CANCEL PAYMENT + RELEASE    CANCEL PAYMENT + RELEASE
                                         -> RELEASED                  -> CANCELLED
```

---

## 10.3 Payment Provider Outage

Authorizing a payment is the slowest step in the whole saga. It takes 300 to 1,500 milliseconds,
out of a 2.5 second total budget (see Chapter 2). It is also the one step that depends on a
system outside GlobalMart's control: the PSP, short for Payment Service Provider, is the outside
company (like a card network partner) that actually moves the money. We cannot scale it up or
fix its bugs ourselves. It will have bad days. The Payment Service and its PSP Adapters
(described in Chapter 5) are built assuming this will happen.

**Timeouts.** Every call to a PSP has a fixed maximum wait time, usually 2 to 4 seconds. This is
set based on the PSP's own normal response time, with some safety margin added. We never wait
forever. A single stuck call to a PSP must not use up the whole 2.5 second checkout time budget.

**Retries with backoff and jitter.** If a call to the PSP times out or returns a server error, we
retry it. But we do not retry it right away, and not always at the same delay. We use backoff,
meaning each retry waits longer than the last one. We also add jitter, meaning we add a small
random extra delay. Here is an example retry schedule:

```
attempt 1: try immediately
attempt 2: wait 200ms,  plus a random 0-100ms
attempt 3: wait 800ms,  plus a random 0-200ms
attempt 4: wait 2000ms, plus a random 0-400ms (only if there is still time left in the budget)
```

Jitter matters because without it, many checkouts would all retry at exactly the same moment.
Imagine the PSP has a short glitch for 500 milliseconds, and 10,000 checkouts all retry at
exactly +200ms. That flood of retries hits the PSP at the exact moment it is already weak,
possibly making things worse. Jitter spreads the retries out over time.

Every retry to the PSP reuses the **same idempotency key**. Most PSPs let the caller attach a
key of their choice to the authorize request. Reusing the same key on every retry is what makes
retries safe (see Chapter 8): the PSP promises that a repeated request with the same key will
either return the exact same result as before, or safely do nothing new. It will never create a
second charge.

**Failover to another PSP.** GlobalMart connects to two or more PSPs per region, or a PSP plus a
direct path to an acquiring bank. If PSP-A's circuit breaker is open (explained below), new
payment attempts get routed to PSP-B instead. This failover only applies to **new** attempts. If
a payment is already in flight against PSP-A, we never silently retry it against PSP-B. PSP-B
has no way to know if PSP-A already completed that charge. Doing this could cause the worst kind
of double charge: two different companies both charging the buyer for the same order.

**Circuit breakers.** A circuit breaker is a safety switch that stops calling a broken dependency
for a while, instead of letting every request wait and time out one by one. Each PSP connection
has its own circuit breaker with three states:

```
CLOSED  --(error rate goes above 50% over the last 30 seconds)-->  OPEN
OPEN    --(after a cooldown period, e.g. 30 seconds)-->            HALF_OPEN
HALF_OPEN --(one test request succeeds)-->                          CLOSED
HALF_OPEN --(the test request fails)-->                             OPEN
```

While the breaker is OPEN, we do not even try calling that PSP. We fail fast instead. We either
send the request to a backup PSP, or, if every PSP is down, we fail the checkout right away
instead of letting it wait. Failing fast, in bulk, is actually the right behaviour here. Imagine
50,000 checkouts per second each waiting out a 4-second timeout against a dead PSP. That would
use up all our server connections and threads, and would take down capacity that healthy
checkouts need.

### The "Did It Charge?" Problem

This is the trickiest failure case in the whole handbook, so it deserves its own explanation.
The orchestrator calls `authorize()`. The request reaches the PSP. The PSP actually processes
it. But then the connection drops, or the reply times out, before GlobalMart learns the result.
Did the card actually get charged, or not? We genuinely do not know yet.

If we guess wrong in one direction, we might charge the card twice. If we guess wrong in the
other direction, we might think the charge failed and quietly cancel the order, even though the
buyer's card already has a pending hold on it. Both outcomes are bad.

The fix is a fixed, three-step process. It reuses the same tools from Chapter 8: idempotency
plus reconciliation.

1. **Ask the PSP directly, first.** Before retrying or undoing anything, the Payment Service
   calls the PSP's status-check endpoint, using the same idempotency key from the original
   request. Almost every major PSP supports this kind of lookup exactly for this situation. This
   solves the mystery directly, in most cases.
2. **If even the status check is unreachable** (meaning the PSP is fully down, not just slow),
   the orchestrator marks this saga as `AUTHORIZATION_UNKNOWN`, instead of guessing. It then
   either waits a bounded amount of time and checks again, or fails the checkout and tells the
   buyer to try again. Importantly, it never silently retries with a brand-new idempotency key.
   That is exactly the mistake that would cause a real double charge.
3. **Reconciliation is the final backstop** (§10.8, and Chapter 8). This is a background job that
   compares GlobalMart's own payment records against the PSP's own transaction report, every few
   minutes to every few hours. If an authorization exists at the PSP with no matching confirmed
   order at GlobalMart, this is called an "orphan authorization." The job automatically cancels
   it if it was never captured, or flags it for a refund if money was actually taken. This is why
   reconciliation is not optional. It is the piece that makes "leave it unresolved for now" a
   genuinely safe choice, because nothing stays unresolved forever.

### Should We Wait, or Fail Right Away?

When a PSP is just slow, not fully dead, GlobalMart has two choices. The right choice depends on
what is best for the buyer in that moment.

| Approach | When we use it | Trade-off |
|---|---|---|
| **Fail fast** | The PSP's circuit breaker is OPEN, or there is a healthy backup PSP ready | The buyer sees an error right away, or is quietly switched to a backup PSP. This keeps our checkout speed target safe. |
| **Wait and retry in the background** | All PSPs are struggling at once (rare, but it happens, like a card-network-wide outage), and the buyer's item is not in a fast-selling flash sale | The checkout screen shows "processing your payment." We must extend the stock hold's expiry time to cover the extra wait. The buyer gets a slower, but successful, checkout instead of a fast failure. |

GlobalMart's default is to fail fast and switch PSPs, because that protects our speed target.
We only fall back to waiting in the background when switching PSPs has also failed, because at
that point a slow success is better for the business than a fast failure. But this wait is still
capped by the stock hold's expiry time, about 15 minutes (Chapter 7). Past that time, we release
the stock hold and fail the checkout cleanly. We never hold stock hostage for a payment that may
never finish.

---

## 10.4 The Dual-Write Problem and the Outbox Pattern

After the Order Service saves an order to the Order DB, it must also tell the rest of the system
about it, by sending an `order.placed` event to Kafka. Fulfillment, notifications, and analytics
all listen for this event (Chapter 5, Chapter 8).

The naive way to do this is: first write to the database, then publish to Kafka. This is two
separate writes to two separate systems, with nothing tying them together. This is called the
**dual-write problem**, and it fails in two different ways:

- The database write succeeds, but the Kafka publish fails (say, the Kafka broker is
  unreachable). Now an order exists, the buyer was charged, but nobody downstream ever finds out.
  The order never ships.
- Or, less commonly, the publish happens right before the commit, and the commit then fails.
  Now an `order.placed` event exists for an order that was never actually created.

Neither result is acceptable on the order-creation path. This is exactly the kind of silent,
invisible mistake the fail-safe rule is meant to stop.

**The outbox pattern fixes this.** It is the same pattern Chapter 8 uses for payment events,
applied here to order events. The idea: write the order row and the "please send this event"
row in the **same single database transaction**.

```
BEGIN TRANSACTION
  INSERT INTO orders (...) VALUES (...);
  INSERT INTO outbox_events (event_id, aggregate_id, event_type, payload, created_at, published)
    VALUES (uuid(), order_id, 'order.placed', {...}, now(), false);
COMMIT
```

Both rows are written together, inside one all-or-nothing database transaction. Either both
rows are saved, or neither is. There is no gap between them where one exists but not the other.

A separate, simple background process called the **outbox relay** watches the `outbox_events`
table for rows where `published` is still `false`. It reads them, sends them to Kafka, and then
marks them as published:

```
Order Service            Order DB shard              Outbox Relay             Kafka
     |  BEGIN TX               |                            |                    |
     |- insert order --------->|                            |                    |
     |- insert outbox row ---->|                            |                    |
     |  COMMIT ----------------->|                            |                    |
     |                          |<-- poll unpublished --------|                    |
     |                          |--- rows --------------------->|                    |
     |                          |                            |-- publish --------->|
     |                          |<-- mark published -----------|                    |
```

The relay can crash, retry, or fall behind, and none of that matters. It will always eventually
send every order's event. It may occasionally send the same event twice, for example if it
crashes right after Kafka confirms receipt but before it marks the row as published. This is
fine, because Kafka in this system is **at-least-once**, never exactly-once. At-least-once means
a message is guaranteed to arrive, but it might arrive more than once. To handle this, every
service that reads from Kafka must be written to be **idempotent**, meaning it produces the
same correct result even if it processes the same message twice.

- **The fulfillment reader** checks the `order_id` against its own table before creating a
  shipment. Processing the same event twice just does nothing the second time.
- **The notification reader** checks a combination of `order_id` and notification type before
  sending an email. This matters a lot, because sending "you were charged" twice to a buyer
  damages trust, even though no money actually moved twice.
- **The analytics reader** typically just updates its own record by `order_id`, so a duplicate
  event has no extra effect.

**Dead Letter Queues for poison messages.** A poison message is an event that a reader can never
successfully process. Maybe its data is broken, or a recent code change no longer understands its
format. If we do nothing about it, this bad message can get stuck at the front of a queue. Kafka
only lets a reader mark messages as done in order, one after another, within one partition (a
partition is one lane of a Kafka topic). So a stuck message blocks every message behind it. The
standard fix works like this:

1. The reader tries to process the message a small, fixed number of times, for example 3
   attempts, with backoff between tries.
2. If it still fails, the reader sends a copy of the message, along with details about the
   error, to a separate **Dead Letter Queue** (DLQ) topic, for example `order.placed.dlq`. Then it
   marks the original message as done anyway, even though it failed. This step is the key part:
   it unblocks the queue so healthy messages behind it can keep moving.
3. The DLQ is watched closely (§10.9). Any message sitting in it triggers an alert. A person or
   an automated tool investigates, fixes the real cause (for example, a bug in how the message is
   read), and then **replays** the DLQ messages back into the normal flow once the fix is ready.
4. DLQ messages are never quietly thrown away. Since these events trace back to a real paid
   order, every single one must be accounted for, even if that accounting happens hours later
   after a manual fix.

---

## 10.5 Order DB Failover

The Order DB (Chapters 4 and 9) is split into about 1,024 shards, or partitions. Each shard has
one main copy plus one or more backup copies, kept in sync for strong consistency. The main copy
of a shard can fail at any time, for example due to a hardware fault or a server restart. When
that happens, two things must both stay true.

**First, we must never lose a committed order.** This requires that every write to the main copy
is also copied to at least one backup copy **before** we tell the caller the write succeeded.
This is often done using a quorum-based replication protocol, such as Raft or Paxos, which are
techniques for keeping multiple copies of data in agreement. If GlobalMart instead copied data
in the background, without waiting, then a main copy could fail right after telling the buyer
"order confirmed," but before the backup copy ever received that order. The backup copy would
then be promoted with no knowledge of that order at all. That would be a lost, paid-for order.
This directly breaks the top priority from Chapter 1: never lose an order. This is why the Order
DB accepts the extra wait time needed to confirm a write on more than one copy.

**Second, a retry after failover must never create a duplicate order.** A failover, where a
backup copy becomes the new main copy, usually takes just a few seconds. Any order-creation call
that gets retried during those few seconds must still land exactly once, no matter how many
times it is retried. This uses the exact same trick from §10.2:

```sql
INSERT INTO orders (order_id, checkout_group_id, idempotency_key, ...)
VALUES (:order_id, :checkout_group_id, :idem_key, ...)
ON CONFLICT (idempotency_key) DO NOTHING
RETURNING order_id;
```

If the original write actually succeeded on the old main copy before it failed, and it was
copied over correctly (per the point above), then the retry against the new main copy finds the
existing row and simply returns it. No duplicate is created, and the buyer sees no error. If the
original write never actually succeeded, the retry creates the row fresh. The orchestrator does
not need to know which of these two cases happened. The database's own uniqueness rule sorts it
out automatically.

```
Timeline of one Order DB shard failing over

t0   Main copy A is healthy and serving writes
t1   Main copy A fails (crash, or loses power)
t2   Health checks notice A is unreachable (roughly 1-3 seconds)
t3   Backup copy B is promoted to become the new main copy
t4   The routing layer redirects new writes to B
t5   The orchestrator retries the order-creation call against B
     -> the idempotent insert either finds the already-committed row, or creates it fresh
```

The orchestrator treats "the database is unreachable" the same way it treats any other step
timeout (§10.6): it retries a limited number of times within a time budget. If the shard is
still unreachable after that, the orchestrator gives up on this attempt and undoes the earlier
steps instead, releasing the stock hold and cancelling the payment authorization. It is better to
ask the buyer to retry the whole checkout than to keep a payment authorization open against a
database shard that might be down for a long time, not just a few seconds.

---

## 10.6 Orchestrator Crash in the Middle of a Saga

The Checkout Orchestrator is built, on purpose, to keep **no long-term memory of its own**
(Chapters 5 and 6 describe it as coordinating without holding long state). This design choice is
exactly what makes an orchestrator crash a safe, boring event, instead of a data-loss disaster.
Three parts work together to make this possible.

**First, durable saga state.** Before running each step, the orchestrator saves the saga's
current progress to a database. This saved record includes the `checkout_group_id`, which step
it is on, and the results so far, such as which stock-hold IDs are `HELD` or which payment ID is
`AUTHORIZED`. This saved state lives in a fast, strongly consistent store, often placed near the
Order DB, never only in memory. This is the same idea as the `IdempotencyRecord` from Chapter 8,
but applied across the whole saga, not just to the final response.

**Second, per-step timeouts** (from Chapter 6). Each step of the saga, such as reserve, authorize,
create order, and capture, has its own maximum time it is allowed to take. This limits how long
any one orchestrator instance can hold a saga "in progress" before something notices and steps
in.

**Third, a recovery sweeper.** This is a background process that continuously scans the saved
saga states, looking for any saga that has been stuck in a non-final state, such as `RESERVING`
or `AUTHORIZING`, for longer than that step's allowed time. When it finds one, it does the
following:

- It reads the saved state to see exactly which steps already finished.
- It picks the saga back up starting from the next unfinished step. It does not start over from
  the beginning. It will not re-reserve stock that is already `HELD`, and it will not
  re-authorize a payment that already has an `AUTHORIZED` record. Every step's own idempotency
  key makes "pick up from step N" and "retry step N" exactly the same safe operation.
- If the saga has been stuck for too long overall, or has been retried too many times, the
  sweeper stops trying to move it forward. Instead, it drives the saga through the undo path:
  release the stock, cancel the payment, and cancel the order.

```
Orchestrator instance A                    Saved saga state              Recovery sweeper
   | saves state: RESERVING -------------->|                                  |
   | (crashes) X                           |                                  |
   |                                        |<-- scan finds: state=RESERVING, |
   |                                        |    stuck too long --------------|
   |                                        |--- picks up from "reserve" ---->|
   |                                        |    (safe, because it's idempotent) |
```

Because every orchestrator instance is interchangeable, and none of them keep their own private
memory, the sweeper (or even just the next orchestrator instance that happens to pick up a
retried request with the same `checkout_group_id`) can safely finish work that a now-dead
instance started. There is no special handoff message needed between the old, dead instance and
the new one. The saved saga state, together with each step's idempotency key, is the entire
handoff mechanism.

---

## 10.7 Graceful Degradation: What Can Bend, and What Cannot

Not every dependency in checkout is on the money-and-stock path. Chapter 5's design already
separates services that touch money or stock, which must always be correct, from services that
just make the checkout experience nicer but are not required for correctness. The rule is
simple: **degrade the second group aggressively to protect the first group's uptime.**

If the **Tax Service** is down, checkout can still continue. We use a cached tax rate for that
buyer's region and item type, or a conservative estimate, and let the checkout finish. A batch
job later recalculates the exact tax once the Tax Service recovers, and issues a small
adjustment or credit if the estimate was off.

If the **Pricing & Promotions Service** is down, checkout can still continue too. The buyer's
already-known base price still applies. Any promo code they already applied earlier in their
session is still honoured, since it was saved in the session snapshot. But no brand-new promo
codes can be applied while the service is down. If this genuinely cost the buyer a discount they
were entitled to, a customer support credit is issued afterward, rather than blocking the whole
checkout while we wait for Promotions to come back.

If a **non-critical part fails**, the buyer should still be able to pay. For example, if the
Notification Service is down, the order is still created and confirmed normally. The
`order.placed` event simply waits in Kafka until the Notification Service comes back and reads
it. Because Kafka delivery is at-least-once (§10.4), no notification is permanently lost. It is
just delayed. Similarly, if a recommendation engine or an analytics pipeline is down, we simply
drop or delay that data. We never let it block checkout, because it was never part of the
correctness-critical path in the first place (Chapter 1's non-functional requirement 5).

The table below lists what can degrade and what cannot.

| Dependency down | What we do instead | What we fix up later |
|---|---|---|
| **Tax Service** | Use a cached or estimated tax rate; let checkout continue | Recompute the exact tax later; issue a small adjustment if needed |
| **Pricing & Promotions Service** | Continue at the last known price; do not apply new promo codes; keep already-applied promos | Give a support credit later if a promo was missed |
| **Notification Service** | Create and confirm the order as normal; the event just waits in Kafka | At-least-once delivery means the message eventually gets sent |
| **Cart Service** (after the session was created) | Use the cart snapshot saved inside the checkout session; no need to re-read the live cart | Not needed — this is exactly why the snapshot exists |
| **Recommendations or analytics** | Simply drop or delay this data under load; never block checkout for it | Not needed |
| **Inventory Service** | **Do not degrade.** If it fails, fail the reservation step and undo the saga | Not applicable |
| **Payment Service / PSP** | **Do not degrade.** Retry, switch providers, or fail the checkout (§10.3) | Not applicable |
| **Order Service / Order DB** | **Do not degrade.** There is no such thing as a "cached" or "estimated" order | Not applicable |

Every row that can degrade follows one pattern: **use a safe stand-in value, let the buyer pay,
then correct the small difference later, quietly, in the background.** Every row that cannot
degrade follows the opposite pattern: **there is no safe stand-in for money or stock, so the only
acceptable option is refusing the operation.** This is the single most important rule in this
whole chapter. State it plainly to yourself: *if not knowing a dependency's answer would force
you to guess about a charge or a stock count, that dependency does not get a soft fallback. It
gets a hard stop.*

---

## 10.8 Reconciliation: The Final Safety Net

Every idea in this chapter so far, like idempotency keys, undo actions, the outbox pattern,
step timeouts, and circuit breakers, reduces the **chance** of a mismatch between GlobalMart's
records and reality. None of them reduce that chance all the way to zero. This is because
keeping several independent systems, like our own database, an outside payment provider, and a
stock ledger, perfectly in sync at every instant has no purely synchronous solution, unless we
use a distributed transaction protocol like two-phase commit. Chapter 11 explains why GlobalMart
deliberately avoids that approach. **Reconciliation is the process that turns "a very small
chance of drift" into "zero drift, eventually, guaranteed."** It is not a nice extra audit
feature. It is a required part of how we prove correctness at all.

Reconciliation jobs run all the time. Some checks run every few minutes for the highest-risk
cases. Others run hourly or daily. They all compare GlobalMart's own records against the
payment provider's ledger and against the stock ledger, looking for specific known patterns of
drift.

| Drift found | How it is detected | How it gets fixed |
|---|---|---|
| **Orphan authorization** — the PSP shows a charge hold with no matching confirmed order at GlobalMart | Compare the PSP's transaction report against our Order DB, matching by checkout ID | Automatically cancel the hold, if it has not been captured yet. Alert someone if the hold has already expired on its own. |
| **Captured but no order** — the PSP shows money actually taken, with no order at all | Same comparison, but only looking at captured charges | Automatically refund the buyer. This is the one case where a refund happens automatically, because there is clearly no order behind the charge. |
| **Leaked stock reservation** — a stock hold stuck in `HELD` past its expiry, with no order or active saga behind it | Scan for reservations where the state is `HELD` and the expiry time has already passed (Chapter 7) | Automatically release it. This should be rare, since the expiry timer already fixes most of these on its own. A rising leak rate is itself a warning sign of a stuck-saga bug. |
| **Order with no payment** — a confirmed order exists with no authorized or captured payment behind it at all | Join the Order DB to the payment records by checkout ID | This should never be structurally possible, since order creation happens only after authorization in the saga. Treat any case of this as a serious bug to investigate immediately, not a routine automatic fix. |
| **Double charge** — two separate captured payments exist for the same checkout | Group payment records by checkout ID and count how many were captured | Refund the extra charge immediately, and open a high-priority investigation. This count must be zero in normal operation (§10.9). Any non-zero count means an idempotency safeguard failed somewhere and needs a real fix, not just a refund. |
| **Oversell** — the total quantity committed for one item is more than the stock the seller actually has | Compare committed reservation totals against the seller's reported stock count | Cancel the most recent order(s) that pushed the total over the limit, refund them, and notify the buyer. This is also treated as a serious incident, with the same "must be zero" standard as double charges. |

The last two rows in that table matter more than all the others combined. They deserve to be
stated as the north star of this entire chapter: **the double-charge count and the oversell
count are the two numbers that decide whether checkout is working at all.** Every other
mechanism described in this chapter, sagas, outboxes, circuit breakers, dead letter queues, all
exist to keep those two numbers at zero. Reconciliation is the tripwire that catches it the
moment either one ever slips above zero.

---

## 10.9 Watching the System: Metrics, Alerts, Runbooks, and Chaos Tests

None of the mechanisms above are worth much if nobody is watching them. This section lists the
minimum dashboard for the team responsible for keeping checkout running.

Some of the metrics below have a hard target of exactly zero, instead of a percentage target.
This is unusual and worth calling out clearly. For most systems, "99.9% correct" sounds like a
strong number. But for money and stock, 99.9% correct actually means 1 out of every 1,000
buyers gets double-charged, or 1 out of every 1,000 orders oversells an item. At 200 million
orders a day, that would mean about 200,000 incidents every single day. That is not an
acceptable outcome for a production checkout system.

| Metric | What it measures | Target or alert rule |
|---|---|---|
| Checkout success rate | Successful `place-order` calls divided by total attempts | Alert if it drops well below its normal recent average |
| Payment authorization success rate | Successful authorizations divided by attempts, per PSP | Alert on a sustained drop for any one PSP; this also feeds the circuit breaker decision |
| **Double-charge count** | How many checkouts got charged more than once | **Must be zero.** Any non-zero value pages someone immediately as a top-priority incident. |
| **Oversell count** | How many orders committed more stock than a seller actually had | **Must be zero.** Any non-zero value pages someone immediately as a top-priority incident. |
| Stuck-saga count | Sagas sitting in a non-final state far longer than expected | Alert if this rises above a small number, since it suggests the recovery sweeper is falling behind or something downstream is jammed |
| Reservation-leak rate | Stock holds found stuck past their expiry, before the timer clears them | Alert if this is consistently above zero, since the expiry timer should normally keep this near zero on its own |
| Reconciliation drift | Count of orphan authorizations, captured-but-no-order cases, and leaked reservations found per run | Watch the trend; a sudden spike right after a deployment usually points at a new bug, not ordinary background noise |
| PSP circuit breaker state changes | How often and how long each PSP's breaker stays OPEN | Alert whenever a breaker opens; track total time spent OPEN for conversations with payment provider partners |
| Dead Letter Queue depth, per topic | How many poison messages are waiting to be fixed and replayed | Alert on any sustained non-zero depth |
| Checkout latency (p99) | The slowest 1% of place-order calls, by response time | Target 2.5 seconds at the 99th percentile, per Chapter 2 |

**Runbooks.** Every alert above is paired with a specific, rehearsed set of steps, so that
nobody has to improvise during an incident. For example, the runbook for "PSP-A's circuit
breaker is open" says: first confirm PSP-B is correctly absorbing the failover traffic and is
healthy, then check PSP-A's own public status page, and never manually force the circuit closed
without first letting a test request succeed on its own. The runbook for "double-charge count is
above zero" starts with refunding the extra charge immediately, to protect the buyer first, and
only afterward moves on to finding out which idempotency safeguard actually failed.

**Chaos testing and game days.** The mechanisms in this chapter only prove they work when
something actually fails. Waiting for a real production incident to test them is too risky, so
GlobalMart tests them on purpose, regularly.

- **Chaos experiments**, run in a staging environment and, carefully, on a small slice of real
  production traffic: deliberately kill an orchestrator instance in the middle of a saga and
  confirm the recovery sweeper picks it up correctly; deliberately make PSP calls time out and
  confirm the circuit breaker opens and failover kicks in within the expected time; deliberately
  kill an Order DB's main copy and confirm zero committed orders are lost, and zero duplicate
  orders are created on retry.
- **Poison message injection**: deliberately publish a broken event and confirm it lands safely
  in the Dead Letter Queue without blocking the rest of the queue, instead of discovering this
  gap for the first time during a real bad software release.
- **Game days**: planned exercises, run across multiple teams, that simulate a specific scenario
  from start to finish, for example "our main PSP goes down for 20 minutes during a flash sale."
  These exercises rehearse the runbook, measure how fast the team actually detects and fixes the
  problem, and reveal gaps between what the design assumes on paper and what really happens
  under real load. The companion search handbook uses this same discipline for search index
  failures. Here, the cost of missing a gap is much higher, because it involves real money and
  real stock.

The purpose of all this monitoring and testing is not to stop every incident from ever
happening. That is not realistically possible at this scale. The purpose is to guarantee that
whenever something does fail, it fails in a way that is noticed quickly, undone correctly, and,
if anything still slips through, caught and repaired by reconciliation before it ever becomes a
buyer-visible double charge or a seller-visible oversell.

---

## Interview Tips

- **Start with the fail-safe principle, out loud.** When an interviewer gives you a failure
  scenario, such as "what if the PSP times out in the middle of authorizing a payment," your
  strongest opening line is: "I would rather fail the checkout, or leave it unresolved for
  reconciliation, than risk a double charge." Say the principle before you describe the
  mechanism. This shows you understand *why* the design looks the way it does, not just *what*
  the design is.
- **Naming the outbox pattern is a strong signal.** If asked how to reliably send an event after
  a database write, immediately naming the dual-write problem, and then the fix, writing the
  event into the same transaction as the main write and relaying it asynchronously, is one of the
  fastest ways to show senior-level thinking in this kind of interview. A weak answer just says
  "publish to Kafka after the commit" without noticing the gap between the two writes.
- **Bring up reconciliation yourself, without being asked.** Many candidates design good sagas
  and good idempotency keys, and then stop, as if the system is now airtight. Saying clearly that
  "none of this is airtight without reconciliation catching whatever slips through" shows a
  deeper understanding that distributed correctness is about probability, not a hard guarantee,
  unless something closes the loop at the end.
- **Know the "did it charge?" answer cold.** The order is: check the real status by idempotency
  key first, then wait a bounded amount of time or fail cleanly, then let reconciliation handle
  anything still unresolved. Interviewers often use this exact question to check whether you will
  wrongly say "just retry" with no further thought.
- **Be ready to draw the undo table from memory**, at least for the core saga steps: reserve then
  release, authorize then cancel, create order then cancel order. Walking through each step out
  loud, explaining how it is detected and how it is undone, is usually worth more in this
  discussion than any single clever answer.
- **Have one short, clear answer ready for "what never degrades."** The answer is: stock
  correctness and payment correctness. Everything else, like tax, promotions, and notifications,
  can degrade to a cached, estimated, or delayed version. Practice saying this short line clearly,
  so it comes out right even under pressure.

## Key Takeaways

- **Fail safe, not fail happy.** When a saga step fails in an unclear way, the default action is
  to reject the checkout, retry it safely, or park it for reconciliation. Never guess in a
  direction that could double-charge a buyer or oversell a seller's stock.
- **Every saga step has a matching undo action**, and the undo action reverses the business
  effect, such as releasing a hold or cancelling a charge, not the raw database row, because we
  must keep the history for audits. Capture is the one step where retrying forward is safer than
  undoing backward, because by that point real money has already moved.
- **The "did it charge?" mystery is solved with a fixed process**: check the real status using
  the idempotency key first; if that is unreachable too, park the attempt instead of guessing;
  reconciliation catches anything that still slips through later.
- **The outbox pattern solves the dual-write problem.** Write the database row and its outgoing
  event together in one local transaction, send the event out asynchronously afterward, and rely
  on at-least-once Kafka delivery plus readers that are safe to run twice, instead of trying to
  make the publish step perfectly exactly-once.
- **Poison messages go to a Dead Letter Queue instead of blocking everything behind them.** We
  alert on DLQ depth and replay the fixed messages once the root cause is resolved.
- **Database failover and orchestrator crashes are both survived the same way**: durable, safely
  copied state, for the database this means synchronous replication, for the orchestrator this
  means saved saga state, combined with idempotent retries. This means no committed order is
  ever lost, and no retry ever creates a duplicate.
- **Degrade aggressively everywhere except money and stock.** Tax, promotions, and notifications
  can run on stale, cached, or delayed data so that the buyer can still pay. Stock reservation,
  payment authorization, and order creation never get a soft fallback. They fail the checkout
  instead.
- **Reconciliation is not just cleanup work. It is the final proof of correctness.** Every other
  mechanism in this chapter only lowers the chance of drift. Reconciliation is what actually
  guarantees the double-charge count and the oversell count reach, and stay at, zero.
- **Watch the hard-zero metrics separately from the percentage-based metrics.** Checkout success
  rate and latency get percentile targets. Double-charge count and oversell count get a hard zero
  target and an immediate high-priority alert on any deviation. Treating them the same as
  ordinary error-rate metrics understates just how strict this bar really is.
