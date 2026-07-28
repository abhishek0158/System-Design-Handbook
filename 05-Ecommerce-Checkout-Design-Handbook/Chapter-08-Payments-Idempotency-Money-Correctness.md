# Chapter 8 — Payments, Idempotency & Money Correctness

Every earlier chapter built up to this one. Chapter 6 gave us the saga steps for `place-order`.
The steps are: reserve inventory, authorize payment, create order, capture payment, confirm. If
any step fails, we run a compensation to undo earlier steps. Chapter 7 made sure the inventory
part of this saga never oversells a product. This chapter makes sure the payment part never
double-charges a buyer, never loses a charge, and never leaves our records in a state nobody can
explain.

Money is different from other kinds of data at GlobalMart. A stale search result is just a bad
experience. But a duplicate $340 charge on a buyer's card is a support ticket, a chargeback, and
a trust problem. It can appear on social media before it even shows up on our own dashboards. The
design brief ranks correctness above speed for this exact reason. NFR1 says: no double-charge, no
oversell, no lost orders is the top priority, above latency. This chapter earns that promise for
the payment side of the system.

We will build this chapter in layers. First, the lifecycle of a payment and its vocabulary:
authorize, capture, settle, void, refund. Second, how the **Payment Service** and its **PSP
Adapters** talk to the outside world without ever seeing a raw card number. Third, the main
topic of this chapter: **idempotency**. This means a request can be repeated safely and it will
never cause a second charge. Fourth, we look at the exact situations where double-charges usually
happen, and how each one is stopped. Fifth, we look at the hard case where a payment call times
out and we do not know if the charge happened. We solve this with idempotency, a status check,
and reconciliation (a background check that compares records). Sixth, the **outbox pattern**,
which safely sends events like `order.placed` without losing them. Then we cover money
representation, refunds and voids, split payments, currency, and finally security and compliance.

---

## 8.1 The Payment Lifecycle: Tokenize → Authorize → Capture → Settle → Refund/Void

A "payment" is not one single event. It is a **state machine** that moves across two different
clocks. A state machine is a system that moves through a fixed set of stages, one at a time. The
first clock is GlobalMart's own checkout clock, which works in seconds. The second clock is the
card network's settlement clock, which works in days. We need to understand what each stage means
for the buyer's money, and what it means for our order. This understanding is the base for
everything else in this chapter.

```
 TOKENIZE ──▶ AUTHORIZE ──▶ CAPTURE ──▶ SETTLE
                  │             │
                  │             └──▶ VOID (before capture, cancels the hold)
                  │
                  └────────────────▶ VOID (auth expires / cancelled before capture)

 CAPTURE ──▶ REFUND (full or partial, after capture/settlement)
```

| Stage | What happens at the PSP (payment provider) | What the buyer sees | What it means for us |
|---|---|---|---|
| **Tokenize** | The buyer's card details are swapped, on the buyer's device, for a safe **PSP token** (a code that stands in for the card) | Nothing is charged yet. The card is just "on file" | We now hold only a token, never the real card number (see §8.2) |
| **Authorize** | The card network places a **hold** on the buyer's available money for this amount | A "pending" line appears on their statement. The money is set aside but not yet taken | We get a `psp_reference` (a proof code) that the bank will honor a capture up to this amount, for a limited time window, usually 5–7 days |
| **Capture** | We tell the PSP to actually take the money | The pending charge becomes a **real, posted charge** | We can now recognize the revenue and ship the item |
| **Settle** | Banks clear and move the money between each other | The money lands in our account, usually one to three days later | This runs in the background. We don't make checkout wait for it, but reconciliation (§8.8) watches it |
| **Void** | An authorization that was never captured is cancelled | The pending hold disappears from their statement. This can take a few days to show, which often confuses buyers and creates support tickets | No money moves. Used when an order is cancelled or fails before capture |
| **Refund** | Money that was already captured is sent back | A new credit line appears on their statement | This is our way to undo a charge after cancellations or returns (§8.9) |

### Why authorize and capture are two separate calls

It may seem simpler to just combine "authorize" and "capture" into one call. If the bank says yes,
why not take the money right away? GlobalMart keeps them separate for two reasons.

First, **we may not be able to fulfill the order yet.** GlobalMart's cart often has items from
several sellers, grouped under one `checkout_group_id` (see Chapter 4's data model). The saga in
Chapter 6 authorizes the full total up front. This locks in the buyer's funds. But one seller's
part of the order can still fail later, for example a warehouse pick error or a shipping problem.
If we had already captured the money, that failure would need an instant refund. A refund is
worse for the buyer, worse for our books, and it makes a fulfillment problem look like a payment
problem in every report we run.

Second, there is a **rule that card networks expect us to follow.** Card networks support
"authorize now, capture on shipment" as the standard pattern for online stores. This is because a
merchant should not take payment for goods it has not shipped yet. In some countries, capturing
before shipment is treated as a compliance warning sign.

GlobalMart's rule, matching the canonical saga in `_DESIGN-BRIEF.md` §5, is: **authorize the full
amount at place-order time, but capture each seller's share only when that seller's part of the
order actually ships.** We call this deferred capture. So one buyer checkout with three sellers
might create one authorization, but up to three separate, smaller captures. Each capture happens
on its own, as each seller's warehouse confirms it picked the item. We explain exactly how these
smaller captures add up against the one big authorization in §8.9.

### The lifecycle across service boundaries

It also helps to look at the same lifecycle from the angle of "who calls whom." Each arrow below
is a network call that can fail on its own. Every failure case discussed later in §8.4 and §8.5 is
really about one of these exact arrows.

```
 Buyer         Checkout Orchestrator      Payment Service        PSP Adapter          PSP
   │  place-order      │                        │                     │                │
   ├───────────────────▶  authorize(total)      │                     │                │
   │                   ├────────────────────────▶  authorize(...)     │                │
   │                   │                        ├─────────────────────▶ POST /charges  │
   │                   │                        │                     ├────────────────▶
   │                   │                        │                     │◀── AUTHORIZED ──┤
   │                   │                        │◀────────────────────┤  psp_reference │
   │                   │◀───────────────────────┤  AUTHORIZED         │                │
   │◀── order CONFIRMED┤  (funds held, not captured)                  │                │
   │                   │                        │                     │                │
   │        ... hours/days later, seller confirms pick & pack ...     │                │
   │                   │                        │                     │                │
   │           capture(sub_order_amount)         │                     │                │
   │                   ├────────────────────────▶  capture(...)       │                │
   │                   │                        ├─────────────────────▶ POST /captures │
   │                   │                        │                     ├────────────────▶
   │                   │                        │                     │◀──── CAPTURED ──┤
   │                   │◀───────────────────────┤  CAPTURED           │                │
   │  (statement shows a posted charge now)      │                     │                │
```

Notice the gap between "order CONFIRMED" and "CAPTURED." This gap is the deferred-capture window
we just described. During this window, the buyer's only financial risk is the authorization hold,
not a finished charge.

There is one trade-off to remember. Authorizations have an **expiry window**, usually 5 to 7 days
depending on the card issuer. Some networks allow re-authorizing after that. If a seller's
shipping time is longer than this window, which does happen for cross-border or made-to-order
items, the Payment Service must **re-authorize** before the original hold expires. This is one job
of the reconciliation sweep in §8.8. That same sweep also flags authorizations that are about to
expire soon, so we can refresh them in time.

---

## 8.2 PSP Integration: The Payment Service and PSP Adapters

GlobalMart does not talk to Visa or Mastercard directly, and we never want to be in a position
where we could. We instead work through **Payment Service Providers**, or **PSPs** for short.
These are companies like Stripe, Adyen, or Braintree that carry the heavy compliance burden of
touching card data. They give us a clean, tokenized way to process payments. In our canonical
architecture, the **Payment Service** sits between the Checkout Orchestrator and a group of **PSP
Adapters**, one adapter for each outside provider.

```
Checkout Orchestrator
        │  authorize(amount, currency, payment_token, idempotency_key)
        ▼
  Payment Service  ── routing / retry / failover policy ──┐
        │                                                  │
        ▼                                                  ▼
  PSP Adapter: Stripe-style          PSP Adapter: Adyen-style   PSP Adapter: Braintree-style
        │                                    │                          │
        ▼                                    ▼                          ▼
   Stripe API                          Adyen API                  Braintree API
```

### Why we use an adapter layer, not a direct connection

**First, we always support more than one PSP.** This is not an afterthought, it is a core design
choice. GlobalMart processes about 220 million payment transactions a day, as stated in the
design brief. Using only one PSP creates two risks. One risk is that a single outage stops
checkout for the entire world. The other risk is losing negotiating power, since we would depend
on one company completely. The adapter interface has simple, shared functions:
`authorize(...)`, `capture(...)`, `void(...)`, `refund(...)`, and `query(...)`. Each adapter turns
these into that specific PSP's own API calls, and turns the PSP's different error codes and
responses into one common format that the rest of our system understands.

**Second, we route each payment intelligently.** The Payment Service picks which PSP to use for
each transaction. It looks at the buyer's region, since PSPs have different banking relationships
and success rates by country. It looks at the currency and payment method, since some PSPs are
better at handling wallets or local bank-transfer methods. It also looks at cost. This is a
routing table that we can tune over time, based on real authorization success rates. See the table
below for examples.

**Third, we fail over to a backup PSP when needed.** If the main PSP for a route becomes unhealthy,
meaning it shows high error rates, timeouts, or its circuit breaker is open (explained fully in
Chapter 10), the Payment Service switches new authorizations to a backup PSP. There is one strict
rule here: failover must **never** silently retry the same `Idempotency-Key` against a *different*
PSP. This is because the PSP's own idempotency protection (explained in §8.4) only works within
that one PSP. It does not protect us across two different PSPs. Failover must only happen by
creating a brand-new attempt with a fresh PSP-specific key, saved as a new row in
`PaymentAttempt`. We only do this after confirming, using the timeout process in §8.5, that the
first attempt truly did not succeed. Failing over blindly on a timeout is exactly how
double-authorizations happen in real systems.

| Routing factor | Example rule |
|---|---|
| Buyer region | EU buyers go first to a PSP with strong SEPA and 3DS2 support |
| Card network | Some PSPs get better success rates for certain issuing countries or card networks |
| Payment method | Wallet or buy-now-pay-later methods route to the PSP built for them |
| PSP health | If the primary PSP's circuit breaker is open, route to the backup and mark the primary as `DEGRADED` |
| Cost | If all else is equal, pick the cheaper PSP by negotiated rate |

### Reducing our PCI compliance scope: the SAQ-A approach

The single most important design decision in this section fits in one sentence from the design
brief: **card numbers never touch our servers.** Here is what that means in practice.

Our checkout page shows a **PSP-hosted field**. This is either a small secure iframe or a script
from the PSP, used just to collect the card number. The buyer's card number, security code, and
expiry date travel **directly from the buyer's browser to the PSP**, over an encrypted connection.
They never pass through our servers at all.

Our servers only ever receive back an **opaque token**, plus safe display information like
`last4` (the last four digits), `brand` (like Visa or Mastercard), and the expiry month and year.
This token is exactly the `payment_token` field in the `CheckoutSession` schema from Chapter 4.
The `PUT /v1/checkout/sessions/{id}/payment` call from Chapter 3 carries only this token, never a
real card number.

This design earns us **SAQ-A** status under PCI-DSS, the payment card industry's security
standard. SAQ-A is the lightest compliance level. It is reserved for merchants who fully hand off
card data handling to a certified third party, and who never receive, send, process, or store card
data themselves. Compare this to SAQ-D, the heaviest level, which applies the moment you touch raw
card numbers yourself. SAQ-D adds hundreds of extra controls: network audits, strict key
management, regular security scans, and a much bigger audit workload. At our scale of 220 million
transactions a day, the difference between SAQ-A and SAQ-D is not just paperwork. It decides
whether a whole category of data breach, a card number leaking from our own systems, is even
possible. With our design, it structurally cannot happen, because the card number never arrives
here in the first place.

The only sensitive item GlobalMart's systems store is the **PSP token**, along with the safe
display details. Even this token is treated as a secret. It is encrypted while stored (see
§8.11), every access to it is logged, and it is useless outside the exact PSP account it was
created for. Most PSPs also scope a token to one specific merchant account, so a leaked token
cannot be reused against a different merchant.

---

## 8.3 Idempotency — The Main Topic of This Chapter

Everything else in this chapter supports one single property. **Calling `place-order`, or any
payment action, more than once with the same key must have the same effect as calling it just
once.** We call this property **idempotency**. It is what lets us retry aggressively, which we
must do because networks drop packets, PSPs time out, and our own servers can crash. Idempotency
makes sure that this aggressive retrying never costs the buyer extra money.

### The end-to-end flow

```
 Client                 API Gateway        Checkout Orchestrator     Idempotency Store    Payment Service
   │  POST place-order        │                     │                       │                  │
   │  Idempotency-Key: K1     │                     │                       │                  │
   ├──────────────────────────▶                     │                       │                  │
   │                          ├─────────────────────▶                       │                  │
   │                          │                     │  lookup(K1)           │                  │
   │                          │                     ├──────────────────────▶│                  │
   │                          │                     │◀──── NOT FOUND ───────┤                  │
   │                          │                     │  insert NEW→IN_PROGRESS (fingerprint(req))│
   │                          │                     ├──────────────────────▶│                  │
   │                          │                     │   ... run saga: reserve, authorize ...    │
   │                          │                     ├────────────────────────────────────────────▶
   │                          │                     │◀───────────────── auth result ────────────┤
   │                          │                     │  create order, capture/defer, etc.        │
   │                          │                     │  store response, mark COMPLETED           │
   │                          │                     ├──────────────────────▶│                  │
   │                          │◀────────────────────┤                       │                  │
   │◀─────────────────────────┤  200 {order_id, status}                    │                  │
   │                          │                     │                       │                  │
   │  (buyer's app retries same request, same K1, e.g. after a UI timeout)  │                  │
   │  POST place-order        │                     │                       │                  │
   │  Idempotency-Key: K1     │                     │                       │                  │
   ├──────────────────────────▶─────────────────────▶  lookup(K1)           │                  │
   │                          │                     ├──────────────────────▶│                  │
   │                          │                     │◀── COMPLETED, resp ───┤                  │
   │◀─────────────────────────┤◀────────────────────┤  (no saga re-run — same response replayed) │
```

The key idea to notice in the second call: the orchestrator **never runs the saga again**. It
stops early at the idempotency lookup step, and just sends back the same stored response,
unchanged. This is what makes retries free and safe. They are free for the buyer, and safe for
GlobalMart.

### The idempotency record and its state machine

Chapter 4 gives us this schema:

```jsonc
// IdempotencyRecord
{ "idempotency_key", "request_fingerprint", "response_snapshot", "state", "expires_at" }
```

The `state` field moves through a small, strict set of stages.

```
        create record                 saga completes
  NEW ─────────────────▶ IN_PROGRESS ─────────────────▶ COMPLETED
                              │
                              │  saga fails terminally
                              │  (after compensations run)
                              ▼
                            FAILED
```

- **NEW** is only a concept, not something we actually see. In real code, the very first write
  goes straight into `IN_PROGRESS`, in one atomic step, using a conditional insert (shown in the
  code below). There is no moment where a record sits as `NEW` and unclaimed.
- **IN_PROGRESS** means a request with this key is actively running through the saga right now.
  If a second request comes in with the same key while this is happening, that is a **concurrent
  duplicate**. We handle it with a single-flight lock, described below, instead of running the
  saga a second time.
- **COMPLETED** means the saga finished successfully. Its final response is saved, word for word,
  in `response_snapshot`. Every future request using this key gets that exact saved response sent
  back, for as long as the record exists. The saga never runs again for this key.
- **FAILED** means the saga reached a clear, final failure. Examples are a declined card or
  permanently out-of-stock inventory, and all compensations already finished running. This is also
  a stable, safe-to-repeat result. A decline is not "unclear," it is a definite answer. So we cache
  it and replay it just like `COMPLETED`. The one thing we must never cache as final is an
  **unclear** result, such as a timeout. We cover that hard case in §8.5.

The **`request_fingerprint`** is a hash, for example a SHA-256 hash, of the important parts of the
request body. This includes the buyer id, session id, amounts, and payment token reference. It
does not include things that naturally differ between retries, like trace ids or timestamps. Its
purpose is to catch a subtle bug: **the same key used for two different requests.** If a client
sends `Idempotency-Key: K1` but the fingerprint does not match what we already stored for K1, this
is not a retry. It could be a client bug, or worse, someone testing for a security weakness. The
correct response is **`409 Conflict`**. We must never silently run the new request, and we must
never silently replay the old response either. Both of those would be wrong, because this is
genuinely a different request.

| Situation | Does the fingerprint match? | What we respond |
|---|---|---|
| A true retry, same request sent again after a timeout | Yes | Replay the stored response, or make the client wait if it is still `IN_PROGRESS` (see below) |
| Same key, but a different request body | No | `409 Conflict` |
| A brand-new key, never seen before | Not applicable | Run the request normally and store the result |
| The key was seen before, but its TTL (time-to-live) has expired | Not applicable | Treat it as a brand-new request. This is why the TTL must be longer than any realistic retry delay (see below) |

### Concurrent duplicates: one execution, not two

The trickiest case is not a retry sent later. It is **two requests with the same key arriving at
almost the same moment.** This really happens. A buyer might double-tap "Place Order" on a slow
connection. Or a client's automatic retry might fire while the first request is still traveling
over the network. Both requests must lead to just **one** saga execution.

We solve this with a **conditional insert**, which acts like a distributed lock.

```sql
-- Atomic claim: only one caller can succeed at inserting a given key
INSERT INTO idempotency_records (idempotency_key, request_fingerprint, state, expires_at)
VALUES ($1, $2, 'IN_PROGRESS', now() + interval '48 hours')
ON CONFLICT (idempotency_key) DO NOTHING
RETURNING idempotency_key;
```

If this `INSERT` returns a row, this caller "won the race" and goes on to run the saga. If it
returns nothing, some other request already claimed this key. In that case, this caller does
**not** run the saga. Instead, it does one of two things.

It can **wait a short time** for the record to reach `COMPLETED` or `FAILED`, then return that
same response. We bound this wait, for example up to the place-order latency budget of 2.5
seconds from Chapter 2, before giving up. Or it can **respond quickly with a `409` or `202`**,
telling the client "still in progress, check `GET /v1/orders/{id}` or retry the same key shortly."
This second option is safe, because the original request is still moving forward on its own and
will finish, whether or not this duplicate call is still waiting.

Either way, only **one** saga execution ever happens for a given key. This is called
single-flight: many identical requests arriving together collapse into one real execution, with
the result then delivered to all of them.

### Pseudo-code: the idempotent place-order handler

```python
def handle_place_order(request, idempotency_key):
    fingerprint = sha256(canonicalize(request.body))

    # 1. Attempt to atomically claim the key.
    claimed = idempotency_store.try_insert(
        key=idempotency_key,
        fingerprint=fingerprint,
        state="IN_PROGRESS",
        ttl=hours(48),
    )

    if not claimed:
        record = idempotency_store.get(idempotency_key)

        if record is None:
            # Expired between the failed insert and this read — extremely rare race;
            # treat as a fresh key and retry claim once.
            return handle_place_order(request, idempotency_key)

        if record.fingerprint != fingerprint:
            raise HttpError(409, "Idempotency-Key reused with a different request body")

        if record.state == "IN_PROGRESS":
            # Concurrent duplicate: wait briefly for the in-flight winner to finish.
            result = idempotency_store.wait_for_terminal(idempotency_key, timeout=2.0)
            if result is None:
                return HttpResponse(202, {"status": "processing", "retry_after_ms": 500})
            return to_http_response(result)

        # COMPLETED or FAILED: replay verbatim. No saga re-execution.
        return to_http_response(record.response_snapshot)

    # 2. We won the claim — run the saga exactly once.
    try:
        result = run_place_order_saga(request)          # reserve → authorize → create → capture
        idempotency_store.complete(idempotency_key, state="COMPLETED", response=result)
        return to_http_response(result)
    except TerminalFailure as e:
        # A definite, well-understood failure (declined, permanently out of stock).
        # Compensations have already run inside run_place_order_saga.
        idempotency_store.complete(idempotency_key, state="FAILED", response=e.as_response())
        return to_http_response(e.as_response())
    except AmbiguousFailure:
        # We do NOT know if the charge went through (timeout talking to PSP, our own crash
        # mid-saga, etc). Do NOT mark COMPLETED or FAILED. Leave state = IN_PROGRESS and let
        # reconciliation (§8.5/§8.8) or a resumed orchestrator instance resolve it.
        raise
```

The `AmbiguousFailure` branch is the most important detail here, and it is the one most designs
get wrong. An error during the saga does not automatically mean "safe to mark failed and let the
client retry." If we are unsure whether money actually moved, it is safer to leave the record as
`IN_PROGRESS` and let reconciliation handle it, rather than guess. We explain this fully in §8.5.

### TTL: why 24 to 48 hours, and what happens when it expires

Per the design brief, idempotency records live for **24 to 48 hours** in the Idempotency Store.
This store is built on Redis, but backed by a durable copy too. Redis is a fast in-memory data
store. We say "durable backup" because this data is too important to risk losing if memory runs
low. The TTL, meaning time-to-live, must be long enough to cover every realistic retry situation.
This includes a mobile buyer retrying after their connection comes back, a support agent manually
retrying on the buyer's behalf, and a queued retry from another system recovering from an outage.
A window of 24 to 48 hours safely covers all of these cases, while also keeping storage size
reasonable. At about 220 million records a day, each around 1 KB, this comes to roughly 250 to 400
GB of hot storage, matching Chapter 2's numbers.

After the TTL expires, sending the same key again is treated as a **brand-new request**. This is
safe only because, by that time, the earlier operation has definitely been resolved one way or
another. Either the order exists and was captured, or it genuinely never happened and was properly
compensated. This is why the TTL length must be based on the saga's own worst-case resolution
time, not just on how long a client might wait before retrying. If a saga could still be unclear
after 48 hours, the TTL would be set wrong. In practice, reconciliation (§8.8) resolves unclear
cases within minutes to a few hours, well inside this window.

---

## 8.4 Exactly-Once Charging = At-Least-Once Delivery + Idempotency

GlobalMart never promises "exactly-once delivery" for any message or request. No distributed
system can honestly promise this end-to-end without unlimited waiting. Instead, per the design
brief's stance, GlobalMart promises this:

> **Exactly-once effect comes from at-least-once delivery plus idempotency keys, not from a
> distributed two-phase commit.**

"At-least-once" means that our client apps, our gateway, our Kafka consumers, and our
orchestrator's own retry logic all choose to **retry instead of silently giving up**, whenever
something fails. This is the right default choice for correctness, since we never want to lose a
checkout attempt. But this choice guarantees that duplicates will happen, often. Idempotency is
what turns "duplicates will happen" into "duplicates cause no harm." This protection must exist
at **two layers at the same time**. The first layer is our own layer, using `Idempotency-Key` and
`IdempotencyRecord`, as covered in §8.3. The second layer is the **PSP's** own layer, since every
`authorize` or `capture` call we send to a PSP also carries a PSP-specific idempotency key.
Relying on only one of these layers leaves a gap, as shown in the scenarios below.

### Scenario 1 — The buyer taps the button twice

**What happens:** A buyer on a slow connection taps "Place Order," sees nothing happen after 3
seconds, and taps again. Our client app is designed to reuse the exact **same**
`Idempotency-Key` for both taps. We generate the key once per checkout attempt, not once per HTTP
call. This rule is a contract we own on the client side.

**Where it is stopped:** Our own idempotency layer (§8.3) stops it. The second request either
finds `COMPLETED` and replays that response, or finds `IN_PROGRESS` and waits through the
single-flight logic. The PSP is contacted **at most once** for this attempt. This is the simplest
case, and it is exactly what the name "Idempotency-Key" was designed for.

### Scenario 2 — A network timeout between us and the PSP, then our own retry

**What happens:** The Payment Service calls the PSP's `authorize` endpoint. The connection times
out before any response arrives. We genuinely do not know if the PSP received our request and
processed it, with the response simply lost on the way back, or if the PSP never received it at
all. Because our system follows the at-least-once rule, it correctly retries the call.

**Where it is stopped:** The **PSP-level idempotency key** stops it. Every `authorize` call from
the Payment Service includes a PSP-specific idempotency key. Most PSPs, including Stripe-, Adyen-,
and Braintree-style providers, support this exact feature natively, often literally using a
parameter called `Idempotency-Key`. It is important that this key is **derived directly from our
own `PaymentAttempt.idempotency_key`**, rather than created fresh for every HTTP attempt.

```python
psp_idempotency_key = f"{payment_attempt.idempotency_key}:{psp_adapter.name}:authorize"
```

When the retried request reaches the PSP using the same PSP-level key, the PSP itself recognizes
it as a duplicate of a request it already processed, or is still processing. It returns the
**original** result instead of authorizing a second hold. This is why §8.2 insisted that failing
over to a *different* PSP must never blindly reuse the same key. PSP-level idempotency only
protects us within one PSP. Retrying against PSP B with the "same" key gives us no protection if
PSP A already went ahead. So retries to the same PSP are safe. Failover to a different PSP is
only safe after we confirm, using a query described in §8.5, that the first PSP truly did not
authorize the charge.

**A worked numeric example.** Say a buyer's total is **$84.30**. Here is the timeline.

| Time (ms) | Event |
|---|---|
| 0 | Orchestrator calls Payment Service: `authorize($84.30, USD, token, key=K1)` |
| 10 | Payment Service calls the PSP Adapter, which calls the PSP using `psp_key = "K1:stripe-style:authorize"` |
| 4,200 | The PSP authorizes the card, creates `psp_reference=pi_7f2a...`, and starts writing its response |
| 5,000 | Our own HTTP client hits its 5-second timeout while waiting. We see a timeout, but the PSP already placed the hold |
| 5,010 | Our retry logic fires, and calls the PSP again with the **same** `psp_key` |
| 5,300 | The PSP recognizes `psp_key` as a duplicate of the request from t=4,200. It returns the **original** `psp_reference=pi_7f2a...`, and creates no new hold |
| 5,320 | The Payment Service returns `AUTHORIZED, $84.30, pi_7f2a...` to the orchestrator, exactly once, even though we made two HTTP calls |

If the PSP key had instead been randomly generated fresh for each HTTP attempt, a mistake that
happens if a developer creates a new random id inside a retry loop instead of reusing the stored
key, the retry at t=5,010 would look like a brand-new request to the PSP. The buyer's card would
then show **two** $84.30 holds. This exact mistake is the single most common cause of real-world
"why was my customer double-charged" incidents. It is purely a coding discipline problem, and the
fix is always the same: derive the PSP key from our own idempotency key, and never regenerate it
inside a retry.

### Scenario 3 — The orchestrator crashes mid-saga, and another instance resumes

**What happens:** The orchestrator instance handling a `place-order` call crashes. This could be
from a pod eviction, an out-of-memory error, or a deployment rollout. Say it crashes after it
called the PSP and got back `AUTHORIZED`, but before it wrote the `PaymentAttempt` row or created
the order. Until that moment, the saga's progress lived only in that one process's memory, plus
whatever it had already written durably.

**Where it is stopped:** This is exactly why every saga step must be **saved durably and
resumable**, a theme we introduced in Chapter 6. Any orchestrator instance can pick up a saga
from its last saved step. But the durable record of "I am calling authorize with key K1" must be
written **before** the call goes out, not after.

```python
def authorize_step(saga_state, idempotency_key):
    psp_key = derive_psp_key(idempotency_key)
    # Durably record intent BEFORE calling out. If we crash after this write but
    # before/during the PSP call, the resuming instance knows exactly which call
    # to check on / retry with which key — it does not have to guess.
    payment_attempts_table.upsert(
        idempotency_key=idempotency_key, psp_key=psp_key, state="AUTHORIZING"
    )
    result = psp_adapter.authorize(amount, currency, token, idempotency_key=psp_key)
    payment_attempts_table.update(idempotency_key=idempotency_key, state=result.state,
                                   psp_reference=result.reference)
    return result
```

A resumed instance finds a `PaymentAttempt` row in the `AUTHORIZING` state. It now knows: "someone
already started this exact authorize call using `psp_key`. Send the same call again with the same
`psp_key`, and let PSP-level idempotency tell us the true outcome." It does not start a fresh
authorization. This "write the intent first, then act, then confirm" order is the same idea as
write-ahead logging in a database. Here we apply it to a saga step that crosses a network
boundary.

### Scenario 4 — Kafka redelivers a message

**What happens:** After order creation, the outbox relay (explained in §8.6) publishes an
`order.placed` event to Kafka, our event streaming system. A consumer service, for example one
that triggers a receipt notification, processes the message, but crashes before it records that
it finished. Kafka works on an at-least-once basis, so it redelivers the same message after the
consumer restarts.

**Where it is stopped:** This is not a payment-idempotency problem exactly, but the same rule
applies. Any consumer of `order.placed`, or of any payment-related event, must be idempotent
based on the event's natural key, such as `checkout_group_id`, `order_id`, or `payment_id`. This
is usually done with an "insert if not already present" upsert on its own side-effect table, or by
checking "have I already sent this for this order_id" before acting. GlobalMart's rule of thumb
is simple: **any consumer of an at-least-once stream must either be naturally idempotent, using
upserts, or must keep its own list of already-handled message ids.**

### Summary table

| Scenario | Where the duplicate comes from | Layer that absorbs it | How it is stopped |
|---|---|---|---|
| Client double-submit | Buyer or UI retry | Our idempotency layer | `Idempotency-Key` plus the `IdempotencyRecord` state machine |
| Network timeout, our retry to the PSP | Network transport | The PSP's own layer | A PSP-scoped idempotency key, derived deterministically |
| Orchestrator crash and resume | Process failure | Both saga persistence and the PSP key | A write-ahead intent record, resumed with the same PSP key |
| Kafka at-least-once redelivery | Broker or consumer restart | Consumer layer | Upsert logic, or a dedup list keyed by message id |
| Cross-PSP failover | Our own routing decision | A status query before failover | Confirm the first PSP did not authorize, using a status query (§8.5), before trying a second PSP |

The lesson across all five rows is the same. **Idempotency must exist at every boundary where we
chose at-least-once delivery over the risk of losing a message.** This includes client to us, us
to the PSP, us across our own crashes, and broker to consumer. Miss even one of these boundaries,
and duplicates can slip through that one gap, no matter how solid the other boundaries are.

---

## 8.5 The Hard Question: "Did the Charge Actually Happen?"

Scenario 2 above skipped over the hardest sub-problem in this whole chapter: **the Payment
Service calls `authorize`, the connection times out with no response. Did the PSP actually charge
the card or not?** This is a basic truth about network calls with side effects. A timeout tells us
that a *response* did not arrive. It does not tell us whether the *request* ever landed. Three
different failure points can produce the exact same symptom, a timeout, but with different real
outcomes.

```
 Payment Service                          PSP
       │                                    │
       ├── authorize request ──── X ────────┤   (1) Request never arrived — nothing happened
       │
       ├── authorize request ───────────────▶
       │                          (2) PSP processes it, authorizes the card,
       │                              response lost on the way back  — IT HAPPENED
       │◀──────── X ─────── response ────────┤
       │
       ├── authorize request ───────────────▶
       │                          (3) PSP is still processing when our
       │                              client-side timeout fires           — UNKNOWN, IN FLIGHT
       │◀── (nothing yet) ────────────────────┤
```

If we blindly retry, we are assuming case (1). This risks a double-authorization if the true case
was actually (2) or (3), and the retry then succeeds on top of the earlier successful call. If we
instead blindly give up and tell the buyer "payment failed, please try again," we are assuming
case (1) too, or assuming a retry is safe for the buyer to start themselves. If the real case was
(2), the buyer now wrongly believes they were not charged, even though they were. This is often
worse than a duplicate charge, because it stays silent until the buyer's statement arrives later.

### The resolution steps, in order

**Step one: retry using the same PSP idempotency key, first.** This is almost always correct and
enough on its own, as we saw in Scenario 2 of §8.4. If the original request did land, which is
case 2, the PSP's own idempotency system recognizes the key and returns the original result. No
new charge happens, and now we actually **know** the outcome. If the request never landed, which
is case 1, this retry is simply the real first attempt.

**Step two: if the retry also times out, or the PSP itself is unreachable, ask the PSP for the
current status directly.** Every PSP adapter offers a `query(psp_reference_or_idempotency_key)`
call. This asks, "what is the current state of the transaction tied to this key?" This turns an
unclear local timeout into a clear, authoritative remote answer, and it creates no new side
effect, since it is only a read. The Payment Service's rule here is simple: **never guess the
outcome of an unclear authorization. Always ask the PSP what actually happened**, using the same
idempotency key to look it up.

**Step three: if even the status query cannot reach the PSP, because of a real outage rather than
one flaky call, leave the `PaymentAttempt` in a clear `AUTHORIZING` or `UNKNOWN` state.** Let
reconciliation (§8.8) resolve it once the connection to the PSP is restored. This is our final
backstop, and it is a **planned** backstop, not a bug. At our scale, some small fraction of calls
will always hit this path. The system must have a clear, monitored, time-bounded way to resolve
these cases, instead of depending on every single call always succeeding cleanly.

```python
def authorize_with_ambiguity_handling(saga_state, idempotency_key):
    psp_key = derive_psp_key(idempotency_key)
    try:
        return psp_adapter.authorize(amount, currency, token, idempotency_key=psp_key)
    except PspTimeout:
        try:
            # Retry is safe: PSP-level idempotency means "same key" == "same logical attempt"
            return psp_adapter.authorize(amount, currency, token, idempotency_key=psp_key)
        except PspTimeout:
            status = psp_adapter.query(psp_key)          # pure read, no new side effect
            if status.is_terminal():
                return status                              # now we know: authorized or declined
            # Still unresolved even via query — surface as ambiguous, don't guess.
            payment_attempts_table.update(idempotency_key=idempotency_key, state="UNKNOWN")
            raise AmbiguousFailure("authorization outcome unresolved; queued for reconciliation")
```

This is exactly why the pseudo-code in §8.3 has that `AmbiguousFailure` branch, which deliberately
avoids marking the `IdempotencyRecord` as `COMPLETED` or `FAILED`. Marking it either way would be
a lie, because we simply do not yet know which one is true. The whole point of this state machine
is that only genuinely final states get cached and replayed. A state of `UNKNOWN` or `AUTHORIZING`
is a completely normal resting state. Its resolution deadline belongs to reconciliation, not to
whoever made the original HTTP call, who has by now already received a `202` response or already
timed out on their own side.

---

## 8.6 The Outbox Pattern: Sending Events Reliably

Step 5 of our canonical saga is "capture payment, now or deferred, and emit `order.placed`." That
word "emit" hides a lot of important work. The order write goes to the **Order DB**, our sharded
SQL database. The event needs to reach **Kafka**, so that fulfillment, notifications, and
analytics can all react to it. These are two completely different systems. Writing to both of
them creates a classic problem called a **dual-write hazard**, meaning the two writes are not
guaranteed to succeed or fail together.

```
   BAD: two independent writes, no atomicity between them

   Order Service
        │
        ├──▶ write Order row to Order DB           ✔ succeeds
        │
        ├──▶ publish order.placed to Kafka          ✘ Kafka broker unreachable
        │
   Result: order exists, buyer was charged, but fulfillment/notifications
           never hear about it. Silent lost event — money moved, nothing downstream knows.
```

Swapping the order of these two writes does not fix the problem. It only changes *which* failure
we get instead. For example, if we publish first and the database write fails afterward, we end
up with an event describing an order that was never actually saved. There is no way to make
"write to our database" and "publish to Kafka" happen as one atomic step across two different
systems, without a distributed transaction. And per the design brief's stance, GlobalMart
deliberately avoids distributed transactions like two-phase commit, in favor of sagas plus
idempotency. The **outbox pattern** is how we get the atomic guarantee here, without using
two-phase commit. We do this by making the second write, "I need to publish this event," **part of
the exact same local database transaction as the order write.**

```
   GOOD: outbox table in the SAME transaction as the order write

   BEGIN TRANSACTION;
     INSERT INTO orders (order_id, checkout_group_id, ..., status) VALUES (...);
     INSERT INTO outbox (event_id, aggregate_id, event_type, payload, created_at, published_at)
       VALUES (uuid(), :order_id, 'order.placed', :payload_json, now(), NULL);
   COMMIT;
```

Both of these rows commit together, or neither commits at all. This uses ordinary database
transaction guarantees inside one single database, so no coordination across systems is needed. A
separate, simple **relay** process then does the actual publishing, running asynchronously in the
background.

```python
def outbox_relay_loop():
    while True:
        batch = outbox_table.select_unpublished(limit=500)   # published_at IS NULL, order by created_at
        for row in batch:
            kafka_producer.publish(
                topic=topic_for(row.event_type),
                key=row.aggregate_id,
                value=row.payload,
                headers={"event_id": row.event_id},
            )
            outbox_table.mark_published(row.event_id)         # idempotent: safe if retried
        sleep(poll_interval)
```

This design stays safe even if the relay process itself crashes, thanks to two properties.

**If the relay crashes after publishing, but before marking the row as `published_at`**, the next
pass of the relay simply republishes the same row. This makes Kafka delivery from the outbox
at-least-once, which is completely fine. As we said in §8.4, any consumer of `order.placed` is
already required to be idempotent using `event_id` or `order_id` anyway.

**The outbox table itself never loses a row it already wrote**, since it was written inside the
same durable transaction as the order. There is no window of time where the order exists but our
intent to publish it does not. That intent is durable from the same commit that created the order.

Some teams use a different, more operationally demanding option instead of a simple polling relay.
This option is called **CDC**, or change-data-capture, which means tailing the database's own
write log, for example using a tool called Debezium on Postgres or MySQL, and streaming outbox
rows straight to Kafka without any polling loop. GlobalMart's architecture treats this as just an
implementation detail of "the relay." What matters for correctness is the contract: a transactional
outbox, followed by at-least-once publishing, followed by idempotent consumers. It does not matter
whether the relay polls or reads a log directly. At 200 million orders a day, a CDC-based relay is
usually the more common production choice, for speed and throughput reasons. But a simple polling
relay is easier to reason about and works perfectly well at moderate scale. It is worth mentioning
both options if asked in an interview.

---

## 8.7 Money Representation and the Ledger Mindset

Before we discuss reconciliation, we need to be exact about how we represent money. Sloppy
representation is its own kind of correctness bug, separate from any distributed-systems issue.

### Store money as whole numbers of minor units, never as decimals

Every amount at GlobalMart, whether in `Order.amounts`, `PaymentAttempt.amount`, or a ledger
entry, is stored as a **whole number count of the currency's smallest unit.** For US dollars, that
smallest unit is cents. For British pounds, it is pence. Some currencies like Japanese yen or
Korean won have no smaller unit at all, and are stored as plain whole numbers too. So $12.50 is
stored as the number `1250`, and never as the decimal `12.50`. This is not just a small style
preference. Computers represent binary floating-point decimals imprecisely, so `0.1 + 0.2` does
not always equal exactly `0.3` in this format. At 200 million orders a day, even a one-in-a-
million rounding error becomes a real, auditable mismatch somewhere. Every amount field also
carries an explicit **currency code**. We never add or compare amounts across two different
currencies without an explicit, logged currency conversion step.

```jsonc
// Correct
{ "amount": 1250, "currency": "USD" }   // = $12.50

// Wrong — never do this
{ "amount": 12.50, "currency": "USD" }
```

### Keep records as double-entry, unchangeable, and append-only

GlobalMart's internal financial records follow a **ledger mindset**, borrowed from traditional
accounting. Every movement of money is recorded as a **matched pair of entries**, one debit and
one credit. Once written, a record is **never changed**. If a correction is needed, we write a new
entry that offsets the old one. We never edit or delete a past entry.

```jsonc
// LedgerEntry (illustrative — the seller payout ledger itself is out of scope per the brief,
// but the *mindset* applies equally to our own payment/order bookkeeping)
{ "entry_id", "ledger_account", "direction": "DEBIT|CREDIT", "amount": 1250, "currency": "USD",
  "related_order_id", "related_payment_id", "reason": "CAPTURE|REFUND|VOID",
  "created_at", "immutable": true }
```

| Event | Debit | Credit |
|---|---|---|
| Capture $12.50 | Accounts Receivable (buyer) goes down | Revenue or cash-in-transit goes up |
| Refund $12.50 | Revenue or cash-in-transit goes down | Accounts Receivable (buyer) goes up |
| Void (never captured) | No ledger entry needed | No ledger entry needed, since money never actually moved |

Why does this matter operationally? When a reconciliation job (§8.8) finds a mismatch, the correct
fix is to add a new corrective entry, with a clear reason and a link to the investigation that
found it. We never just "edit the row" to make the numbers match. Because the ledger never
changes past entries, the full history of *how* a mismatch was found and fixed becomes part of the
permanent audit trail. This is exactly what NFR6, "auditability and PCI compliance, every money
movement traceable," requires. It is also exactly what finance teams, compliance teams, and
outside auditors expect to be able to review years later. Chapter 4's requirement to keep records
for 7 years only makes sense if those records are trustworthy, unchangeable history, not just a
snapshot that could have been silently edited.

---

## 8.8 Reconciliation: The Safety Net

Idempotency keys, PSP-level duplicate protection, and the outbox pattern together prevent almost
all double-charge and lost-event risk, purely by how they are built. But "built to prevent it" is
not the same as "mathematically impossible." Real systems also face PSP-side bugs, our own bugs,
manual actions such as a support agent issuing a refund outside the normal flow, partial outages
that leave `UNKNOWN` states sitting around, and simply the fact that any big enough distributed
system eventually behaves in a way nobody planned for. **Reconciliation is our safety net. It
assumes our preventive systems will occasionally fail anyway, and it catches those failures.**

In practice, this means running a continuous job. It runs every few minutes for recent activity,
plus a thorough full sweep once a day. This job compares three separate records that *should*
always agree with each other.

1. **Our own `PaymentAttempt` table**: what we believe we authorized, captured, voided, or
   refunded.
2. **The PSP's own transaction records**, pulled through their reporting API or their settlement
   file feed.
3. **Our `Order` table**: which orders exist, and what status each one has.

```
                     ┌────────────────────┐
                     │   PaymentAttempt    │
                     │   (our record)      │
                     └─────────┬───────────┘
                               │
              ┌────────────────┼────────────────┐
              ▼                                 ▼
   ┌─────────────────────┐            ┌──────────────────────┐
   │   PSP transaction    │            │      Order table      │
   │   records (source     │◀─ compare ─▶  (order exists?      │
   │   of truth for $$)   │            │   status matches?)     │
   └─────────────────────┘            └──────────────────────┘
```

### Types of drift, and how each is fixed

The word "drift" here means a mismatch between what should be true and what our records actually
show.

| Drift type | What it looks like | Likely cause | How we fix it |
|---|---|---|---|
| **Orphan authorization** | The PSP shows an `AUTHORIZED` hold, but we have no matching `Order` in `CONFIRMED` state, or no `PaymentAttempt` row at all | The saga crashed right after authorizing, before the order was created, and the compensation never ran | Automatically **void** the stale authorization, which releases the buyer's hold. Alert a human only if this repeats for the same buyer or session |
| **Captured but no order** | The PSP shows a `CAPTURED` charge, but there is no matching confirmed `Order` | Rare. Usually a bug in the capture step firing without the earlier order-create step, or a capture call replayed outside our normal idempotent path | **Escalate this immediately**, since real money was taken with nothing to show for it. Only auto-refund after confirming there is truly no delayed order still on its way. Never auto-refund blindly |
| **Order but no capture** | A confirmed `Order`, marked shipped, but no matching `CAPTURED` `PaymentAttempt` | The deferred-capture step failed silently, or the authorization expired (§8.1) before capture happened | Retry the capture. If the original authorization expired, first **re-authorize** using the stored token, then capture. If the buyer's card now fails, this becomes a collections or support case, never a silent write-off |
| **Authorization about to expire** | Still `AUTHORIZED`, order not yet shipped, and the hold is close to the issuer's expiry window, usually 5 to 7 days | A long shipping time, for example cross-border or made-to-order items, taking longer than the standard authorization window | Proactively **re-authorize** before it expires, so the later capture does not fail |
| **Refund mismatch** | We recorded a refund, but the PSP shows it never landed, or shows a different amount | A network problem during the refund call, or a manual adjustment made directly at the PSP, outside our system | Re-send the refund using the same idempotency key, which is safe since refunds are idempotent too. If the amounts truly differ, flag it for manual finance review |

The "auto-void stale authorizations" job deserves special mention. It is one of the very few
reconciliation actions we let run **fully automatically**, without waiting for a human. Voiding an
orphaned, never-captured authorization has no downside for a real order, since by definition there
was no real order attached to it. It also has a clear upside for the buyer, since their hold is
released sooner. Compare this with "captured but no order," which we never auto-repair blindly.
This difference is deliberate: **we automate the fixes that are safe in every possible case, and
we escalate the fixes where safety depends on facts the job cannot fully check by itself.** For
example, the job cannot always tell "is there truly no order coming, or is this just a slightly
delayed read that has not caught up with a write that already happened."

**A worked numeric example.** Suppose a 15-minute reconciliation pass, covering the last hour of
activity, roughly 8.3 million payment transactions at GlobalMart's average rate of about 2,300
orders per second from Chapter 2, finds the following.

| Category | Count found | Total dollar exposure | Action taken |
|---|---|---|---|
| Orphan authorizations | 340 | $28,900 (held, not charged) | Auto-void all 340. Alert only if the same buyer or session appears more than once, which could signal a client retry-loop bug |
| Captured but no order | 2 | $147.98 | Escalate to on-call staff immediately, using a page, not just a ticket |
| Order but no capture, still within normal SLA | 1,150 | $71,200 | No action needed. This is an expected backlog, since these sub-orders simply have not shipped yet |
| Order but no capture, past the SLA deadline | 6 | $412.50 | Retry the capture. Re-authorize first if the original hold already expired |
| Refund mismatches | 0 | $0 | No action needed |

The *shape* of this table matters more than any single run's exact numbers. Rows like "orphan
authorizations" and "order but no capture within SLA" are expected, steady background noise at
this scale. A few hundred to a few thousand cases per hour is normal, caused by ordinary network
issues and shipping delays, and they get auto-repaired without ever paging anyone. But the row
"captured but no order" being non-zero should **always** trigger an immediate page. Unlike the
other rows, it represents real money taken with no legitimate order behind it. As mentioned
before, it must be investigated carefully, and never refunded blindly, in case it is actually a
delayed order write rather than a true orphan capture.

Reconciliation results go to two places: the automated repair jobs described above, and a **drift
dashboard**. This dashboard is treated as a first-class operational metric. It tracks the total
dollar amount of open drift, the age of the oldest unresolved case, and a count broken down by
category. In a healthy checkout system, the drift amount, as a fraction of total payment volume,
should be a very small and steady number. If this number starts trending upward, we treat it as a
production incident, not just a finance-team curiosity, because it usually means something in the
idempotency or outbox chain has broken.

---

## 8.9 Refunds and Voids as Saga Compensations

Chapter 6 introduced compensations as the saga's way to "undo a step that already finished," since
there is no automatic rollback across separate services. Payments are where this idea becomes
very concrete. A **void** undoes an authorization that was never captured. A **refund** undoes a
capture that already moved real money.

```
  Saga step                      Forward action         Compensating action
  ──────────                     ───────────────         ────────────────────
  Authorize payment      ──▶     hold funds       ──▶    void (release hold, pre-capture)
  Capture payment         ──▶     move funds       ──▶    refund (return funds, post-capture)
```

Both of these compensation actions are, themselves, idempotent operations, and each carries its
own idempotency key. "Refund this `payment_id` for this amount" is just as vulnerable to being
sent twice or retried after a timeout as the original charge was. So everything covered in §8.3
through §8.5 applies here too, in the same way. A retried void call must never fail simply because
the authorization was already voided by an earlier attempt. Our PSP adapter's `void` and `refund`
functions are expected to treat "already voided or refunded, same key" as a success, not as an
error.

### Partial refunds for orders split across multiple sellers

This is where GlobalMart's multi-seller design adds real complexity, beyond a simple textbook
single-merchant checkout. Remember that one `checkout_group_id` can cover N sub-orders, one per
seller. The saga authorizes the **full grand total** as a single authorization, but captures
**separately, per sub-order**, as each seller confirms fulfillment (§8.1). Now imagine that 2 of 3
sellers ship successfully and get captured, but the 3rd seller's item goes permanently out of
stock after the saga already authorized the full total.

```
   checkout_group_id: CG-9001         grand_total authorized: $150.00 (single PSP hold)

   Sub-order A (Seller 1): $60.00  ──▶ shipped ──▶ captured  $60.00
   Sub-order B (Seller 2): $50.00  ──▶ shipped ──▶ captured  $50.00
   Sub-order C (Seller 3): $40.00  ──▶ pick failed ──▶ cancelled, never captured

   Resolution:
     - The $40.00 portion attributable to Seller 3 was never captured — so there is nothing
       to refund for it; it simply expires as part of the original $150 authorization (which,
       once A and B are captured, most PSPs let you either void the *remaining* uncaptured
       portion explicitly, or let it lapse at auth expiry).
     - Result: buyer is charged exactly $110.00 total ($60 + $50), never touched for the
       $40.00 that never shipped. No refund transaction is even needed if we void the
       remainder proactively — proactive partial-void is preferable to "capture the full
       $150 then refund $40" because it avoids ever moving money that didn't need to move.
```

If GlobalMart's policy, or a specific PSP's limitations, instead captures the *full* authorized
amount right away, and only later discovers that sub-order C cannot ship, the compensation
becomes a real **partial refund**. We refund exactly $40.00 against the original `payment_id`,
tagging it to sub-order C's cancellation, with its own idempotency key and its own ledger entries.
Per §8.7, this is recorded as a $40.00 debit-then-credit pair, never as a silent edit to the
original $150 entry. Which of these two approaches, partial-capture-only versus capture-all-then-
partial-refund, gets used depends on each PSP's capabilities and our product policy. But the
underlying design principle stays the same either way: **a compensation is scoped exactly to the
sub-order that failed, calculated from the immutable ledger, and never approximated or batched
together with unrelated sub-orders.**

This is also exactly the scenario where the earlier decision to separate authorize from capture,
and defer capture per sub-order (§8.1), pays off. Because capture is deferred and scoped to each
sub-order, a partial failure for one seller becomes a *non-event* for the two sellers who
succeeded. Their captures were never touched by the third seller's problem. Had we instead
captured the full grand total eagerly right at order creation, every partial-fulfillment failure
across GlobalMart's whole marketplace would generate its own refund transaction, constantly, at
full marketplace scale.

---

## 8.10 Split Payments and Currency/FX, Briefly

GlobalMart lets a buyer pay for one checkout using **more than one payment method** at once. The
most common combination is a **gift card plus a card**, or a **wallet balance plus a card**.
Occasionally, for very large baskets, buyers use two different cards to stay under each card's own
limit.

The design impact of this is narrow, but still important. A `checkout_group_id` using split
payment produces **multiple `PaymentAttempt` records**, one for each method. Each one has its own
`idempotency_key`, built from the parent key plus a method index, for example `K1:method-1` and
`K1:method-2`. Each is independently authorized, captured, voided, or refunded on its own. But all
of them are still linked under the same `checkout_group_id` for saga-level compensation purposes.
If a later step fails and the saga compensates, **every** payment method used in the split must be
compensated, not just the first one that was processed. The compensation step must loop through
the full set. Gift cards and wallet balances usually work as an immediate debit rather than a true
hold-then-capture, since there is no card-network settlement delay behind an internal wallet
balance. So their compensation is a direct credit-back, rather than a formal void. The Payment
Service's PSP Adapter design allows for this, since each adapter can implement `void` as either a
true network void or an immediate reversal, and the orchestrator does not need to know the
difference.

**A brief note on currency and FX (foreign exchange).** When a buyer's payment method uses a
different currency than the order's settlement currency, for example a cross-border purchase, the
exact conversion rate used must be **captured and stored at the exact moment of authorization.**
This means one specific rate, at one specific timestamp, tied to that one `PaymentAttempt`. It
must never be recalculated later from a live, changing rate table. This matters for the same
reason integer minor units matter in §8.7. A refund issued days later must reverse the *same*
amount, using the *same* currency terms the original charge used. Otherwise the buyer receives
back a different real-world value than they originally paid, and reconciliation (§8.8) has no
stable number left to check against. FX itself, along with the seller-payout and settlement
ledger, is explicitly outside this handbook's deep scope, per the design brief. Treat FX as a
solved input, meaning a rate locked at the moment of authorization, rather than something we need
to design here.

---

## 8.11 Security and Compliance: The Envelope Around All of This

Everything covered so far assumes a surrounding set of controls. These controls are not this
chapter's main design topic, but they cannot be separated from "money correctness" either.

- **PCI-DSS scope**: as covered in §8.2, client-side tokenization keeps GlobalMart at the light
  SAQ-A compliance level. We check this continuously, not just once. Any engineering change that
  could cause a real card number to pass through or land on our servers, for example a
  well-meaning "let's log the request body for debugging" change on the payment path, counts as a
  PCI scope violation. We treat it with the same seriousness as a security incident. Automatic
  log-scrubbing and CI checks catch this by failing any build that contains card-number-shaped
  patterns in code or test files.
- **Encrypting stored tokens**: PSP tokens are not real card numbers, but they still act as
  access to a payment method. So we encrypt them while stored, using envelope encryption with keys
  held in a managed key management system and rotated on a regular schedule. They never appear as
  plain text in application logs. Display details like `last4` and `brand` are logged freely,
  since they are not sensitive on their own.
- **Audit logging**: every state change on a `PaymentAttempt`, meaning every authorize, capture,
  void, refund, or reconciliation-driven fix, gets written to an **append-only audit log**. This
  log is separate from the ledger, but cross-linked to it. It records who or what triggered the
  action, whether that was the buyer, the orchestrator, a reconciliation job, or a specific support
  agent's manual action with their identity attached, when it happened, and which idempotency key
  was involved. This is what makes a question like "why did this refund happen" answerable months
  later, without guesswork. It is also exactly what auditors and compliance reviewers actually
  check.
- **The fraud-check hook**: deep fraud detection is outside this chapter's scope, but the design
  seam where it plugs in still matters. A fraud-scoring call, covering things like velocity checks,
  device fingerprinting, and risk scoring, sits **between session revalidation and payment
  authorization** in the saga. It can stop a place-order attempt *before* it ever reaches the PSP,
  if the signals look high-risk. It can also flag a lower-risk but still suspicious authorization
  for extra step-up authentication, such as a 3DS-style challenge, without blocking it outright.
  The Payment Service exposes this as a policy hook that the Checkout Orchestrator calls. The
  fraud service itself, and its underlying models, belong to a separate system.

---

## Interview Tips

- **Start by naming idempotency at every layer, explicitly.** The strongest answers separate "our
  own `Idempotency-Key`, protecting against client retries" from "the PSP's own idempotency key,
  protecting against our retries to the PSP" from "consumer-side dedup, protecting against Kafka's
  at-least-once redelivery." A weak answer just says "we use an idempotency key" as if one key at
  one layer solves everything. Walk through why it does not, using Scenarios 2, 3, and 4 from
  §8.4, and you will stand out.
- **Bring up the timeout-ambiguity problem before the interviewer asks it.** "What if the
  authorize call times out, did it charge or not?" is the single most common follow-up question on
  this topic. Volunteering the answer, meaning retry with the same PSP key, then query PSP status,
  and only then fall back to reconciliation, without ever guessing, shows that you have really
  thought about failure cases, not just the happy path.
- **Describe reconciliation as a planned backstop, not an afterthought.** Say clearly: "Idempotency
  and the outbox pattern prevent almost all of these issues by their design, but at this scale I
  would still run continuous reconciliation. No preventive system has a formal proof of zero
  failures, given bugs, PSP-side issues, and manual actions." This shows real systems maturity,
  knowing the difference between "usually prevented" and "provably impossible."
- **Know the auth-versus-capture timing trade-off cold**, and connect it to the multi-seller model.
  Deferred, per-sub-order capture is exactly *why* a partial marketplace fulfillment failure does
  not turn into a flood of refunds. This ties Chapter 8 back to the marketplace framing from
  Chapter 1, and shows you are not treating payments as an isolated textbook problem.
- **If asked "why not use 2PC or a distributed transaction across the database and Kafka, or across
  services,"** have the outbox-pattern answer ready. Two-phase commit does not work well with
  Kafka, since there is no native distributed-transaction coordinator spanning a relational
  database and a message broker at this scale, and even where one is offered, it costs
  availability during a network partition. Instead, we get atomicity locally, meaning a single
  database transaction covering both the order row and the outbox row, and we get correctness
  downstream through at-least-once delivery plus idempotent consumers.
- **Watch for the "just dedupe on amount plus buyer plus timestamp" trap.** A common trick question
  is "why not just dedupe using amount, buyer, and timestamp instead of a key?" Explain why an
  explicit, client-generated key with a full request fingerprint, not just the amount, is
  necessary. Two genuinely *different* orders can share the same amount, buyer, and a near-identical
  timestamp, for example buying the same $12.99 item twice on purpose, back to back. A scheme
  without a fingerprint would wrongly treat these two separate purchases as one duplicate.

## Key Takeaways

- The payment lifecycle, **tokenize, authorize, capture, settle, refund or void**, separates
  "funds set aside" from "funds actually moved." GlobalMart authorizes the full total at
  place-order time, but **defers capture to each sub-order's own fulfillment**. This keeps
  multi-seller partial failures independent of each other, and avoids unneeded refund traffic.
- The **Payment Service plus PSP Adapters** hide multiple PSPs behind one interface, for routing,
  failover, and resilience. **Client-side tokenization ensures raw card numbers never reach our
  servers**, which keeps GlobalMart at the lightweight SAQ-A PCI-DSS compliance level.
- **Idempotency is the main topic of this chapter.** An `Idempotency-Key` maps to an
  `IdempotencyRecord` that moves through `NEW`, then `IN_PROGRESS`, then `COMPLETED` or `FAILED`.
  This is protected by an atomic claim using a conditional insert, single-flighting for concurrent
  duplicates, a `409` response for key reuse with a different body, and verbatim replay of stored
  responses instead of re-running the saga.
- **Exactly-once effect equals at-least-once delivery plus idempotency, applied at every
  boundary.** This covers client to us, us to the PSP using a PSP-scoped key derived from our own
  key, across our own process crashes using write-ahead intent records, and broker to consumer
  using idempotent or upserting consumers of Kafka's at-least-once delivery.
- **A timeout on an authorize call is genuinely unclear.** Resolve it by retrying with the same PSP
  idempotency key first, then by querying PSP status directly, and only as a last resort by
  parking the attempt as `UNKNOWN` for reconciliation. Never guess, and never mark an unclear
  outcome as if it were final.
- The **outbox pattern** makes emitting `order.placed` reliable, without needing two-phase commit.
  We write the event to an outbox table inside the *same local transaction* as the order write, and
  let an asynchronous relay publish it to Kafka at-least-once, relying on idempotent downstream
  consumers.
- Money is stored as **whole numbers in minor units, with an explicit currency**, and recorded in
  an **unchangeable, double-entry-style ledger.** Corrections are always new offsetting entries,
  never edits, which is what makes 7-year auditability actually meaningful.
- **Reconciliation is a deliberate safety net, not a nice-to-have.** Continuous jobs compare our
  records against the PSP's records and against our orders. They auto-repair safe drift, such as
  auto-voiding orphan authorizations, and escalate unsafe drift, such as captured-but-no-order,
  rather than ever guessing.
- **Refunds and voids are saga compensations**, from Chapter 6. They follow the same idempotency
  rules as the original charge. For GlobalMart's multi-seller orders, they are scoped exactly to
  the sub-order that failed, and never batched or approximated across sellers.
- Split payments, such as card plus gift card plus wallet, create multiple linked
  `PaymentAttempt` records under one `checkout_group_id`. Each is independently idempotent, and
  all are compensated together. FX rates are locked at the moment of authorization, and reused
  exactly for any later refund.
