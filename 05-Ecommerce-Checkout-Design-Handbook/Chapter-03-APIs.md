# Chapter 3 — APIs

Chapter 1 listed the requirements. Chapter 2 sized the system. We saw ~2,300 orders per second on
average, and ~50,000 orders per second at flash-sale peak. Checkout APIs must handle ~300,000
calls per second at peak. The place-order call must finish in under 2.5 seconds for 99% of
requests.

This chapter turns those requirements into a real contract. An **API** is a promise between a
client and a server about what requests look like and what responses mean. The client here is
GlobalMart's web app, its mobile apps, and maybe partner apps in future.

For checkout, this promise must hold even when things go wrong. Networks drop requests. Clients
time out and retry. Thousands of buyers may race for the same item in a flash sale. The API must
still make sure we never charge a buyer twice, and never sell more stock than we have. Every
design choice in this chapter serves that goal.

## 1. API Design Principles for Checkout

Before we look at each endpoint, let's fix four principles. Every endpoint follows these.

**Principle 1: Resources, not actions.** Checkout has two main resources: a **checkout session**
and an **order**. A resource is a "thing" you can create, read, or update through the API. Each
resource has a small set of operations. The server gives each resource an ID, like `session_id`
or `order_id`. The client never picks or guesses these IDs. The client also never talks to
inventory or payment systems directly. Every change goes through one service, the **Checkout
Orchestrator** (see Chapter 5). This service is the only one allowed to touch Inventory, Payment,
and Order data, always in that order (see Chapter 6).

**Principle 2: Idempotency is part of the contract, not an extra feature.** In a distributed
system, any network call can be retried. A mobile app may retry after a timeout. A load balancer
may fail over mid-request. The tricky part: the retry might arrive *after* the first request
already succeeded on the server. If `place-order` were not idempotent, a retry could charge the
buyer twice or create two orders for one purchase. An **idempotent** call means: if you call it
many times with the same key, you get the same result every time, and the action happens only
once. We explain this fully in §4, because interviewers ask about it more than any other checkout
detail.

**Principle 3: A closed set of error codes.** When something goes wrong, the client must know
exactly what happened. It should not guess by reading a human sentence. So we define a fixed,
small list of error codes (see §7). Every service in the system must translate its failures into
one of these codes before sending a response to the client.

**Principle 4: Versioning only adds, never removes.** The API path includes a version, like
`/v1/...`. Inside `v1`, we only add new optional fields. We never remove a field. We never change
what a field means. If we need a real breaking change, for example restructuring how sub-orders
work, we release it as `/v2`. Both versions run at the same time during a transition period (more
on multi-region rollout in Chapter 9). Internal APIs, used only between our own services, can
version on their own schedule, since they have far fewer users than the public API.

With these principles fixed, here is the full API surface from the design brief:

| Method | Path | Purpose |
|---|---|---|
| `POST` | `/v1/checkout/sessions` | Create a checkout session from a cart |
| `GET` | `/v1/checkout/sessions/{id}` | Read current totals, shipping, tax, and promos |
| `PUT` | `/v1/checkout/sessions/{id}/shipping` | Set the shipping address and method |
| `PUT` | `/v1/checkout/sessions/{id}/payment` | Attach a tokenized payment method |
| `POST` | `/v1/checkout/sessions/{id}/place-order` | Idempotent order placement |
| `GET` | `/v1/orders/{id}` | Order details plus status history |
| internal, mTLS only | inventory `reserve`/`commit`/`release`, payment `authorize`/`capture`/`void`/`refund` | Saga steps, not public |

The base URL is `https://api.globalmart.com`. All requests and responses use JSON over HTTPS.
Every public call carries a buyer JWT, sent by the API Gateway (see §8). A **JWT** (JSON Web
Token) is a signed piece of text that proves who the buyer is. This table matches the functional
requirements FR1 through FR7 from Chapter 1, and the saga steps from the design brief.

## 2. `POST /v1/checkout/sessions` — Create a Session

This call is the starting point. It takes the buyer's cart and turns it into a **priced,
reviewable session**. This is FR1 from Chapter 1. The server, not the client, works out the real
price, tax, and any discounts, using the Pricing & Promotions Service and the Tax Service. The
price the buyer saw on the product page was only a cached estimate, not a final quote.

**Request**

```json
POST /v1/checkout/sessions
Authorization: Bearer <buyer_jwt>
Content-Type: application/json

{
  "cart_id": "cart_8f2a1c",
  "buyer_id": "buyer_9a7e21",
  "items": [
    { "listing_id": "lst_5001", "seller_id": "sel_100", "qty": 2 },
    { "listing_id": "lst_7788", "seller_id": "sel_100", "qty": 1 },
    { "listing_id": "lst_3342", "seller_id": "sel_204", "qty": 1 }
  ],
  "currency": "USD"
}
```

Notice what is missing from this request: there is no unit price. The client only says *what*
the buyer wants to buy. The server decides *what it costs*, right now. This stops a common
attack, where a client sends a fake, lower price. It also fulfills FR3: the price at checkout can
be different from the price shown earlier in the catalog.

**Response — `201 Created`**

```json
{
  "session_id": "sess_c19f0e7b",
  "buyer_id": "buyer_9a7e21",
  "status": "OPEN",
  "sub_orders": [
    {
      "seller_id": "sel_100",
      "items": [
        { "listing_id": "lst_5001", "qty": 2, "unit_price": 24.99 },
        { "listing_id": "lst_7788", "qty": 1, "unit_price": 12.50 }
      ],
      "subtotal": 62.48,
      "shipping": 4.99,
      "tax": 5.40
    },
    {
      "seller_id": "sel_204",
      "items": [
        { "listing_id": "lst_3342", "qty": 1, "unit_price": 89.00 }
      ],
      "subtotal": 89.00,
      "shipping": 0.00,
      "tax": 7.12
    }
  ],
  "totals": {
    "subtotal": 151.48,
    "shipping": 4.99,
    "tax": 12.52,
    "discount": 5.00,
    "grand_total": 163.99,
    "currency": "USD"
  },
  "shipping_address": null,
  "payment_token": null,
  "session_version": 1,
  "expires_at": "2026-07-15T14:32:00Z"
}
```

Two details matter a lot here. First, **one cart from two sellers becomes two `sub_orders`**
inside the session. This is the multi-seller marketplace shape described in the design brief.
Later, this same shape becomes two separate `Order` records, both linked by one
`checkout_group_id` (see §4). Second, look at **`session_version`**. Every change to the session,
like setting shipping or payment, increases this number by one. Both `place-order` and `GET` show
this number, so the client can tell if the totals changed since it last looked. For example, a
discount code may expire between two screens. This stops the buyer from confirming an order at
the wrong price. The session itself is temporary, not permanent. It has a TTL, or time-to-live, of
about 30 minutes (see Chapter 4). It is a staging area, not the final record. The Order database
is the final record.

## 3. Reading and Changing the Session

**`GET /v1/checkout/sessions/{id}`** returns the same shape as the create response. It shows the
current totals, shipping options, tax, and any applied promo codes. This lets the client redraw
the review screen after the app was closed and reopened, without repeating any earlier step. This
call has no side effects, so it is safe to call more than once. In practice, the client calls it
once when the screen loads.

**`PUT /v1/checkout/sessions/{id}/shipping`** covers FR2. It sets the address and shipping method,
and it recalculates shipping cost and tax for every sub-order. Tax often depends on the delivery
address, so this step must run again after the address changes.

```json
PUT /v1/checkout/sessions/sess_c19f0e7b/shipping
Authorization: Bearer <buyer_jwt>

{
  "session_version": 1,
  "address": {
    "line1": "221B Baker St", "city": "London", "postal_code": "NW1 6XE", "country": "GB"
  },
  "method": "STANDARD"
}
```

```json
200 OK
{
  "session_id": "sess_c19f0e7b",
  "status": "OPEN",
  "sub_orders": [ /* shipping and tax recalculated for each seller */ ],
  "totals": { "subtotal": 151.48, "shipping": 6.50, "tax": 14.10, "discount": 5.00,
              "grand_total": 167.08, "currency": "USD" },
  "shipping_address": { "line1": "221B Baker St", "city": "London", "postal_code": "NW1 6XE",
                          "country": "GB" },
  "session_version": 2
}
```

The client must send the `session_version` number it last saw. If this number is old, for
example another device changed the same session, the server rejects the update with
`409 Conflict` and returns the current session. It does not silently overwrite the newer data.
This is a cheap and effective form of **optimistic locking**: a way to catch conflicting updates
without locking the record all the time. It works well here because one session usually belongs
to one buyer, so real conflicts are rare.

**`PUT /v1/checkout/sessions/{id}/payment`** covers FR4. This is the most sensitive call in this
chapter, because it deals with payment details. The request carries a **PSP token**, never a raw
card number, CVV, or expiry date. A PSP is a Payment Service Provider, a company like Stripe that
handles card payments for us.

```json
PUT /v1/checkout/sessions/sess_c19f0e7b/payment
Authorization: Bearer <buyer_jwt>

{
  "session_version": 2,
  "payment_method": {
    "type": "CARD",
    "psp": "stripe",
    "psp_token": "tok_1P8x...a92f",
    "last4": "4242",
    "brand": "VISA"
  }
}
```

```json
200 OK
{
  "session_id": "sess_c19f0e7b",
  "status": "READY",
  "payment_token": "tok_1P8x...a92f",
  "session_version": 3,
  "totals": { "...": "unchanged" }
}
```

Here is why we use a token instead of the real card number. The buyer's device, either a browser
using the PSP's secure form, or a mobile app using the PSP's SDK, sends the card details straight
to the PSP. This happens before the card number ever reaches a GlobalMart server. In simple
words: the client turns the card into a safe token itself, and our servers only ever see that
token. We also see a non-sensitive `last4` and `brand`, just to show "Visa ending in 4242" on
screen. This keeps GlobalMart's checkout servers out of the strictest level of card-security
audits (PCI-DSS SAQ D), because we never store, process, or move a real card number. This follows
the design brief's rule directly: raw card numbers never touch our servers. The Payment Service
later uses this same token when it calls `authorize` (see §6). If a buyer splits payment between,
say, a gift card and a credit card, the request carries a list of payment methods, each with its
own token and its own amount.

## 4. `POST /v1/checkout/sessions/{id}/place-order` — Idempotent Order Placement

This is the most important call in this chapter, and maybe in the whole handbook. It starts the
saga described in Chapter 6: revalidate the session, reserve inventory, authorize payment, create
the order or orders, then capture payment and confirm. A **saga** is a sequence of steps across
several services, with a defined "undo" step for each one if something later fails. This call
must feel like one atomic action to the buyer, even though it is really several steps across
several systems.

```json
POST /v1/checkout/sessions/sess_c19f0e7b/place-order
Authorization: Bearer <buyer_jwt>
Idempotency-Key: 7c1e9b2a-3f4d-4e2b-9a10-6b8f0c5d2e11
Content-Type: application/json

{
  "session_version": 3
}
```

### 4.1 Why we need an `Idempotency-Key`, and how it works

Here is the simple rule: **the same key, with the same request, always returns the same result,
and the action never happens twice.**

The client generates one random ID, called a UUID, at the exact moment the buyer taps "Place
Order." This single ID is reused for every retry of that same tap. It is not a new ID for every
network attempt. This one decision turns "we don't know if the request worked" into "it is
always safe to retry."

The Checkout Orchestrator saves this key in an `IdempotencyRecord`. This record has:

```jsonc
{
  "idempotency_key": "7c1e9b2a-3f4d-4e2b-9a10-6b8f0c5d2e11",
  "request_fingerprint": "sha256(session_id + session_version + normalized_body)",
  "state": "IN_PROGRESS | COMPLETED | FAILED",
  "response_snapshot": null,
  "expires_at": "2026-07-17T14:32:00Z"
}
```

The **request fingerprint** is a hash, or short fixed-length code, made from the session ID,
session version, and the request body. It lets the server tell apart two very different cases.
Case one: this is a genuine retry of the exact same action. Case two: this key was reused for a
different request, which is likely a client bug, or possibly a replay attack.

**Here is the full decision table for `place-order`:**

| Situation | Server behavior | Status |
|---|---|---|
| New key, first request | Save a new `IdempotencyRecord` with state `IN_PROGRESS`, then run the saga | 200/201 on success |
| Same key, same fingerprint, and the earlier request already **completed** | Skip the saga. Return the saved response as-is | Same status and body as before |
| Same key, same fingerprint, but the earlier request is **still running** (a concurrent retry) | Do not run the saga again. Either wait briefly for the result, or point the client back to check later | `202 Accepted` (check again soon), or the final result once ready |
| Same key, but a **different** fingerprint | Reject the request. This key already belongs to a different request | `409 Conflict`, error code `idempotency_key_reused` |
| Key is older than 24–48 hours and reused | Treat it as a brand-new key, with no history | Proceeds as normal |
| No `Idempotency-Key` header at all | Reject the request outright, since this header is required | `400 Bad Request`, error code `idempotency_key_required` |

Now, what happens when two identical requests arrive at almost the same time? This is common
during a flash sale, when a buyer double-taps the button, or a client retries too fast. The rule
is: **one request wins, and the other waits.**

The server handles this with a **conditional write**: it inserts the record into the Idempotency
Store (which is Redis, per our canonical architecture) only if no record for that key exists yet.
Think of it like a race for one empty parking spot: only one car can park there. Whichever
request wins this insert goes ahead and runs the saga. The other request sees that a record
already exists and is `IN_PROGRESS`. It then either waits a short time for the result, or
immediately returns `202` with a `Retry-After` hint, telling the client to check back using
`GET /v1/orders` or a similar status check. This guarantees that even at huge scale, two racing
retries can never both reserve the same stock or both charge the same card. Chapters 7 and 8 go
deeper into the inventory and payment side of this. This chapter only fixes the HTTP-level rule.

Idempotency records are kept for **24 to 48 hours**. Based on the design brief's numbers, about
220 million payment attempts per day at roughly 1 KB each adds up to about 250–400 GB of hot
storage. This window is long enough to cover normal retries, even "the buyer closed the app and
reopened it three hours later." It is also short enough to keep the storage size under control.
After the window passes, the same key can be reused as if it were new. This is safe, because the
order itself, once created, never changes, and can always be looked up separately by its
`order_id` or `checkout_group_id`.

### 4.2 Success response — multiple orders, and partial success

A cart can include items from more than one seller. So `place-order` can create **more than one
order**, all linked by one `checkout_group_id`. Here is a detail many candidates miss in
interviews: these orders do not all have to succeed together. For example, if seller 204's item
sells out in the few milliseconds between session creation and inventory reservation, seller
100's items can still be confirmed successfully. The response shape makes this explicit.

```json
201 Created
{
  "checkout_group_id": "cg_44a1e0",
  "session_id": "sess_c19f0e7b",
  "status": "PARTIAL_SUCCESS",
  "orders": [
    {
      "order_id": "ord_9931aa",
      "seller_id": "sel_100",
      "status": "CONFIRMED",
      "amounts": { "subtotal": 62.48, "shipping": 4.99, "tax": 5.40, "total": 72.87,
                    "currency": "USD" }
    },
    {
      "order_id": "ord_9931ab",
      "seller_id": "sel_204",
      "status": "CANCELLED",
      "error": {
        "code": "out_of_stock",
        "message": "lst_3342 is no longer available in the requested quantity.",
        "listing_id": "lst_3342"
      },
      "amounts": { "subtotal": 89.00, "shipping": 0.00, "tax": 7.12, "total": 0.00,
                    "currency": "USD" }
    }
  ],
  "payment": {
    "psp_reference": "pi_3P9y...aK2",
    "state": "CAPTURED",
    "amount_captured": 72.87,
    "currency": "USD"
  }
}
```

Notice that the amount charged only covers the **confirmed** sub-orders. The Payment Service
recalculates the total and authorizes only that new amount, after dropping seller 204's failed
item. Chapter 6 explains this saga branch in detail: when part of the inventory reservation
fails, we re-check the payment amount before authorizing, rather than authorizing the full amount
first and then reversing part of it. This avoids an extra, unnecessary authorize-then-cancel round
trip with the PSP.

The overall `status` field on the response is one of three values. `CONFIRMED` means every
sub-order succeeded. `PARTIAL_SUCCESS` means some succeeded and some did not. `FAILED` means none
succeeded (see §7 for this case). When the client sees `PARTIAL_SUCCESS`, its job is simple: show
the confirmed orders normally, and show the cancelled ones with their `error.code`, offering the
buyer a chance to add that item to a new cart. The client must never retry `place-order` with the
same key hoping for a different result. The idempotency rule guarantees it will return this exact
same snapshot every time.

### 4.3 Retrying `place-order`

The rule for the client is simple, exactly because the server-side rule is strict. Whenever the
outcome is unclear, for example a timeout, a 5xx server error, or a dropped connection, the client
should **retry the exact same request with the exact same `Idempotency-Key`**. There is no case
where retrying with the same key causes a second charge or a second order. The worst outcome is
just an extra round trip to fetch a result that already existed.

## 5. `GET /v1/orders/{id}` — Order and Status History

```json
GET /v1/orders/ord_9931aa
Authorization: Bearer <buyer_jwt>
```

```json
200 OK
{
  "order_id": "ord_9931aa",
  "checkout_group_id": "cg_44a1e0",
  "buyer_id": "buyer_9a7e21",
  "seller_id": "sel_100",
  "line_items": [
    { "listing_id": "lst_5001", "qty": 2, "unit_price": 24.99, "tax": 3.70 },
    { "listing_id": "lst_7788", "qty": 1, "unit_price": 12.50, "tax": 1.70 }
  ],
  "amounts": { "subtotal": 62.48, "shipping": 4.99, "tax": 5.40, "total": 72.87,
                "currency": "USD" },
  "status": "CONFIRMED",
  "status_history": [
    { "status": "CREATED",   "at": "2026-07-15T14:31:58Z" },
    { "status": "CONFIRMED", "at": "2026-07-15T14:31:59Z" }
  ],
  "payment_ref": "pi_3P9y...aK2",
  "reservation_ids": ["rsv_a01", "rsv_a02"],
  "created_at": "2026-07-15T14:31:58Z",
  "updated_at": "2026-07-15T14:31:59Z"
}
```

This is a simple read from the Order database, which is split into shards by `hash(order_id)`, as
covered in Chapter 2's storage plan. A **shard** is one piece of a database that has been split
into many pieces, so each piece stays small and fast. This read is strongly consistent for the
buyer who just placed the order, meaning the buyer always sees their own latest data right away.
This one endpoint serves the fulfillment team, customer support tools, and the buyer's "My Orders"
page. We keep `status_history` because "what state is this order in, and when did it get there"
is the most common question from support staff and from the reconciliation jobs described in
Chapter 8. A related call, `GET /v1/orders?buyer_id=...`, lists all orders for one buyer, with
pages of results, using the same shape as this single-order response.

## 6. Internal APIs (mTLS) — Saga Steps, Not Public

The Checkout Orchestrator is the only caller of these internal APIs. They cannot be reached from
the public internet. Instead, they use **mTLS**, or mutual TLS, where both the caller and the
callee prove their identity to each other with certificates (see §8). Each internal call also
carries its own idempotency key, separate from the buyer-facing one, scoped to just that one saga
step.

```
# Inventory Service
POST /internal/inventory/reserve
  { checkout_group_id, seller_id, items: [{listing_id, qty}], ttl_seconds: 900,
    idempotency_key }
  -> { reservation_ids: [...], state: "HELD", expires_at }

POST /internal/inventory/commit
  { reservation_ids, idempotency_key } -> { state: "COMMITTED" }

POST /internal/inventory/release
  { reservation_ids, idempotency_key } -> { state: "RELEASED" }

# Payment Service
POST /internal/payment/authorize
  { checkout_group_id, psp_token, amount, currency, idempotency_key }
  -> { payment_id, state: "AUTHORIZED", psp_reference }

POST /internal/payment/capture
  { payment_id, amount, idempotency_key } -> { state: "CAPTURED" }

POST /internal/payment/void
  { payment_id, idempotency_key } -> { state: "VOIDED" }

POST /internal/payment/refund
  { payment_id, amount, idempotency_key } -> { state: "REFUNDED" }
```

Each of these calls is idempotent on its own key, for the same reason `place-order` is. The
Orchestrator itself can crash and get restarted by its saga framework (see Chapter 6). When that
happens, it may resend the same internal call. At-least-once delivery between services must never
turn into "reserved the stock twice" or "charged the card twice." The two busiest steps during a
flash sale are `reserve` and `authorize`. `reserve` gets heavy contention on one `listing_id`, or
SKU, when many buyers want the same item (see Chapter 7). `authorize` relies partly on the PSP's
own idempotency support. Most PSPs, including Stripe, accept and honor their own version of an
idempotency key. So the Payment Service passes its key through to the PSP too, giving us
protection at both hops. The other three calls, `void`, `release`, and `refund`, are the
**compensating actions**: the "undo" steps the saga runs in reverse order if something fails
later (see the design brief §5). These must be safe to call even if the original action never
actually completed downstream. That is why they are built as idempotent no-ops, meaning "do
nothing and return success," when called on a resource that is already released or already void,
rather than throwing an error.

## 7. Error Handling

Every checkout error, whether from a public or internal call, comes back as an HTTP status code
plus a **domain error envelope**. This is a small JSON object with a fixed `code` field, so the
client can decide what to do just by checking that one field.

```json
{
  "error": {
    "code": "out_of_stock",
    "message": "lst_3342 is no longer available in the requested quantity.",
    "retryable": false,
    "details": { "listing_id": "lst_3342", "requested_qty": 1, "available_qty": 0 }
  }
}
```

| HTTP status | `code` | Meaning | What the client should do |
|---|---|---|---|
| 409 | `out_of_stock` | Inventory reservation failed for one or more items | Drop that item, suggest alternatives. Do not retry with the same key |
| 409 | `price_changed` | The real price, promo, or tax at place-order time is too different from the session's last known totals | Show the buyer the updated totals, ask for confirmation, then place the order again after refreshing the session |
| 402 | `payment_declined` | The PSP declined the authorization | Ask the buyer for a different payment method. Safe to retry, but only with a **new** `Idempotency-Key`, once new payment details exist |
| 410 | `session_expired` | The session's 30-minute TTL has passed | Create a new session from the cart. There is nothing to retry |
| 409 | `idempotency_key_reused` | Same key was sent with a different request | This is a client bug. Always generate a new key for each new buyer action, not for each network attempt |
| 400 | `idempotency_key_required` | `place-order` was called with no header | Fix the client to always send this header |
| 409 | `session_version_conflict` | The `session_version` sent was out of date | Call `GET` on the session again, then retry the change against the current version |
| 429 | `rate_limited` | The client went over its allowed rate (see §8) | Wait, following the `Retry-After` value. Do not just keep sending new keys |
| 503 | `saga_step_unavailable` | A downstream system, like the PSP or the Inventory database, is currently degraded | Retry the exact same `place-order` request and key after a short wait. This is always safe |

This list is deliberately small and fixed. Whenever the saga discovers a new kind of failure, it
must be mapped onto one of these codes, or a genuinely new code can be added later as an
additive change to `v1`. We never send a raw downstream error straight to the client, such as a
PSP-specific decline reason or a database exception message. Doing that would tie the client to
our internal implementation, and would break the versioning promise from §1. The `retryable` flag
is included in the envelope on purpose, so a generic retry loop on the client side can decide
whether to back off and retry, or show the error to the buyer, without needing a hardcoded list of
error codes.

## 8. Auth, Rate Limiting, and Versioning

**Auth.** Every public endpoint requires a **buyer JWT**. This token is issued when the buyer logs
in or refreshes their session. The API Gateway checks the token's signature, expiry, and buyer ID
before the request ever reaches the Checkout Orchestrator. Services behind the gateway trust the
gateway's forwarded identity information, rather than checking the JWT again themselves. Calls
between our own services, for example from the Orchestrator to the Inventory Service, use mutual
TLS. Each service proves its own identity with a certificate, and a mesh policy (see Chapter 9)
only allows the specific caller-to-callee connections that the saga actually needs. For example,
only the Orchestrator is allowed to call `payment/authorize`. This design gives us two separate
trust models for two separate risks. Buyer-facing auth protects us from the open internet. mTLS
protects us from a compromised or misconfigured service inside our own network.

**Rate limiting.** We use two layers of limits, because our main worry is a flash-sale stampede,
not everyday abuse.

- **Per-buyer limits**, tied to the JWT, apply to session and mutation endpoints. These limits are
  generous, since a real human buyer never legitimately calls `place-order` 50 times per second.
  This layer mainly stops scripted "buy bots."
- **Per-SKU, or hot-key, limits** sit at the API Gateway and the Inventory Service, right in front
  of the `reserve` call, and they work independently of buyer identity. During a flash sale, the
  real problem is thousands of *different* buyers all hitting one `listing_id` at once. Per-buyer
  limits do nothing to stop that, since each buyer is only calling once. This gateway-level limit
  is the front-line version of the hot-shard protection covered in depth in Chapter 7, and the
  surge handling covered in Chapter 9. The main point for this chapter: the rate-limit decision
  happens right at the edge, returning `429 rate_limited` with a `Retry-After` value, before a
  doomed request can use up capacity in the Orchestrator, Inventory Service, or PSP that it has no
  real chance of using well.

**Versioning.** As explained in §1, `/v1` only ever adds new optional fields and new error codes.
A future `/v2`, for example one that restructures `sub_orders` to support combined shipping across
sellers, would run behind the same API Gateway. Both versions would route to the same Checkout
Orchestrator logic wherever possible, using a thin adapter layer that translates `v1` shapes into
the internal model and back. This way, only the edge-facing format changes per version, not the
core business logic.

## Interview Tips

- **Idempotency-Key is the topic interviewers dig into deepest. Go three levels down, not just
  one.** Level 1 is saying "we use an idempotency key," which is only the basic answer. Level 2
  explains storage: where the key lives (the Idempotency Store, backed by Redis, with a durable
  backup), what is stored with it (the fingerprint, the saved response, the state, and the TTL),
  and why the fingerprint matters. Without it, a reused key could silently return a wrong, stale
  answer for a genuinely different request. Level 3, the level that separates strong answers, is
  the **race condition**: two requests with the same key arriving within milliseconds of each
  other. Say clearly that this is solved with a conditional write, an insert-only-if-absent
  operation on the idempotency record. Say that the losing request does not run the saga again. It
  either waits briefly or returns `202`. Never say "the second request also just processes it,"
  since that answer shows you have not thought through the race at all.
- **Expect the question "what if the retry has a different body?"** This tests whether you truly
  understand why the fingerprint exists, not just that it exists. The correct answer is
  `409 idempotency_key_reused`. It is wrong to say the server should overwrite the old request, and
  wrong to say it should process both.
- **Expect the question "what does the client do on `PARTIAL_SUCCESS`?"** Say plainly: never retry
  `place-order` with the same key hoping the cancelled sub-order will now succeed. Idempotency
  guarantees the exact same result comes back. Instead, the buyer adds the failed item to a new
  cart and checks out again, which creates a brand-new key. This shows you can tell apart "a retry
  of the same intent" from "a new intent that happens to look similar," a distinction many
  candidates blur together.
- **Connect the PSP-token answer to PCI compliance scope, not just "security."** The real
  architectural reason client-side tokenization matters is that it keeps GlobalMart's servers
  completely out of the strictest PCI-DSS audit scope (SAQ D). Say this explicitly. Just saying "we
  don't store card numbers" undersells the point.
- **If asked to design one endpoint on a whiteboard, pick `place-order`.** It touches every major
  theme the interviewer wants to see: idempotency, multi-seller partial success, the saga's fixed
  order of steps (reserve, then authorize, then create, then capture), and a closed set of error
  codes. No other endpoint packs in this much depth per minute of discussion.

## Key Takeaways

- The public API is small and built around two resources: the checkout session and the order.
  One call, `place-order`, does the real work. Every other call either sets up state or reads it.
- `session_version` gives cheap, simple protection against conflicting updates to a session.
  `Idempotency-Key` gives an exactly-once effect for `place-order`, even though the network only
  guarantees at-least-once delivery. These are two different problems, solved by two different,
  complementary tools.
- The `Idempotency-Key` rule is precise. Same key plus same fingerprint returns the same saved
  response, with no re-run. Same key plus a different fingerprint returns `409`. Two identical
  requests at the same time: one proceeds through a conditional write, and the other waits or
  polls. This is a concrete mechanism, not just a slogan.
- A cart from multiple sellers produces multiple orders under one `checkout_group_id`. The API is
  built to allow **partial success**: one seller's item failing does not fail the whole order, and
  the amount actually charged only covers the confirmed sub-orders.
- Card data never reaches GlobalMart's servers. The client turns the card into a token using the
  PSP, and `PUT /payment` only ever carries that `psp_token`. This is the architectural reason we
  stay out of the strictest card-data compliance scope.
- Errors always come back as a closed, fixed set of codes, such as `out_of_stock`,
  `price_changed`, `payment_declined`, and `session_expired`, each with a clear `retryable` flag.
  Clients act on this fixed contract, not on free-text error messages.
- Auth splits cleanly by trust boundary: a buyer JWT protects public calls from the open internet,
  and mTLS protects internal saga calls between our own services. Rate limiting splits by attack
  shape: per-buyer limits catch scripted abuse, while per-SKU or hot-key limits catch flash-sale
  stampedes that per-buyer limits cannot see at all.
