# _DESIGN BRIEF — Canonical Source of Truth

> **Purpose:** This file fixes every shared fact for the *E-commerce Checkout System Design
> Handbook*. Every chapter MUST use these numbers, names, APIs, and schemas so the handbook
> stays internally consistent. If a chapter needs a new number, derive it from these — don't
> invent a contradicting one.

---

## 0. The Scenario — "GlobalMart Checkout"

We are designing the **checkout system for GlobalMart**, the same global multi-seller
marketplace (Amazon + AliExpress scale) whose search system we designed previously. A buyer
has items in a cart from **multiple sellers**; checkout must turn that cart into one or more
confirmed, paid orders **without ever double-charging the buyer or overselling a seller's
inventory**, even during flash sales when tens of thousands of buyers race for the same SKU.

Checkout is the **revenue-critical, correctness-critical** path. Its defining tension is the
opposite of search: search chose availability over consistency (AP); checkout must be
**strongly consistent where money and inventory are involved** (CP where it counts), while
degrading gracefully everywhere else.

The core of the system is a **saga**: reserve inventory → authorize payment → create order →
capture payment → confirm, with **compensating actions** if any step fails, and
**idempotency** everywhere so retries never double-charge or double-reserve.

---

## 1. Requirements Summary

### Functional
- **FR1 — Create checkout session:** turn a cart into a priced, reviewable session (subtotal,
  shipping, tax, discounts, per-seller sub-orders).
- **FR2 — Address & shipping selection:** set shipping address; compute shipping options/costs.
- **FR3 — Price, tax & promotions:** authoritative re-pricing at checkout (prices/promos/tax
  can differ from the catalog snapshot the buyer saw).
- **FR4 — Payment method:** attach/tokenize a payment method (card, wallet, gift card, split).
- **FR5 — Place order (idempotent):** the atomic-feeling operation — reserve inventory,
  authorize payment, persist order(s), confirm. Safe to retry via an **Idempotency-Key**.
- **FR6 — Inventory reservation:** hold stock so two buyers can't buy the last unit; release
  on timeout/failure.
- **FR7 — Order lifecycle & status:** CREATED → CONFIRMED → (fulfillment) with a queryable
  status and history.
- **FR8 — Notifications & downstream events:** emit `order.placed` for fulfillment, email,
  analytics.

### Non-Functional (prioritized)
1. **Correctness / no money or inventory errors** — no double-charge, no oversell, no lost
   orders. This is the top priority (above latency).
2. **High availability** — 99.99% for the checkout path (downtime = direct lost revenue;
   budget ~53 min/year).
3. **Low latency** — checkout review p99 ≤ 300 ms; place-order p99 ≤ 2.5 s (dominated by the
   external payment authorization).
4. **Surge tolerance** — absorb 10–20× flash-sale spikes on hot SKUs without oversell.
5. **Consistency model:** **strong** for inventory reservation, payment, and order creation;
   **eventual** acceptable for order-history reads, recommendations, analytics.
6. **Auditability / PCI compliance** — every money movement traceable; card data never
   touches our servers (tokenized via PSP).

### Out of scope
Catalog/search (that's the other handbook), the fulfillment/warehouse & shipping-carrier
systems (we hand off via `order.placed`), returns/refunds UI (mention refund as a
compensation), the seller payout/settlement ledger (mention only).

---

## 2. Canonical Scale Numbers (USE THESE EXACTLY)

| Quantity | Value | Notes |
|---|---|---|
| Registered users | 2 B | same GlobalMart universe as the Search handbook |
| Daily active users | 500 M | |
| **Orders / day** | **~200 M** | |
| **Avg order-placement rate** | **~2,300 orders/sec** | 200M / 86,400 |
| **Peak order-placement rate** | **~50 K orders/sec** | ~20× avg (Black Friday / flash sale) |
| Checkout sessions / day | ~600 M | ~33% convert to orders (cart abandonment) |
| Avg checkout-session rate | ~7,000 sessions/sec | 600M / 86,400 |
| Checkout API calls / session | ~6 | create, shipping, quote, payment, review, place |
| **Peak checkout-API QPS** | **~300 K QPS** | flash sales concentrate on place-order |
| Payment transactions / day | ~220 M | orders + auth retries |
| Avg items per order | ~3 | ~600 M line items/day |
| Avg order value (illustrative) | ~$60 | for GMV framing only |

### Derived storage (canonical)
- Order record ≈ **~5 KB** (line items, addresses, payment refs, status history, per-seller
  sub-orders) ⇒ 200M/day × 5KB = **~1 TB/day** of order data ⇒ **~365 TB/year** raw.
- Retention: **7 years** (financial/legal) ⇒ multi-PB cold; **hot tier ≈ last 90 days ≈ ~90 TB**.
- Idempotency records: 220M/day × ~1KB, TTL 24–48 h ⇒ **~250–400 GB** hot (Redis + durable backup).
- Inventory reservations: transient, TTL ~**15 min**.
- **Order DB:** sharded ~**1,024 shards** by `hash(order_id)`; hot shards on NVMe, strong consistency.
- **Inventory DB:** sharded by `listing_id`/`sku`; the oversell-prevention core.

### Latency budget (place-order, target p99 2.5 s)
Gateway+auth 20 ms · session revalidation (price/tax) 50 ms · inventory reservation 80 ms ·
**payment authorization (external PSP) 300–1500 ms (dominant, variable)** · order persist 40 ms ·
response 20 ms · buffer. Review/cart pages: p99 ≤ 300 ms.

---

## 3. Canonical API Surface

Base: `https://api.globalmart.com` · JSON over HTTPS · versioned `/v1` · buyer JWT via gateway.

- `POST /v1/checkout/sessions` — create a checkout session from a cart. Returns session with
  per-seller sub-orders, subtotal, and a `session_version`.
- `GET /v1/checkout/sessions/{id}` — current totals, shipping options, tax, applied promos.
- `PUT /v1/checkout/sessions/{id}/shipping` — set shipping address / method.
- `PUT /v1/checkout/sessions/{id}/payment` — attach a tokenized payment method (PSP token).
- `POST /v1/checkout/sessions/{id}/place-order` — **idempotent** order placement. Requires an
  **`Idempotency-Key`** header. Returns order id(s) + status. Safe to retry.
- `GET /v1/orders/{id}` — order + status history.
- Internal (mTLS, not public): inventory `reserve`/`commit`/`release`; payment
  `authorize`/`capture`/`void`/`refund`; order `create`.

**Idempotency:** the `Idempotency-Key` header on place-order is the linchpin — the same key +
same request returns the same result and never re-charges. Keys stored with the response for 24–48 h.
**Money data:** card PANs never hit our servers — the client tokenizes with the PSP; we store
only the PSP token + last4.

---

## 4. Canonical Data Model (core entities)

```jsonc
// CheckoutSession — ephemeral, TTL ~30 min
{ "session_id", "buyer_id", "cart_snapshot": [ {listing_id, seller_id, qty, unit_price} ],
  "sub_orders": [ {seller_id, items, subtotal, shipping, tax} ],
  "totals": {subtotal, shipping, tax, discount, grand_total, currency},
  "shipping_address", "payment_token", "session_version", "status", "expires_at" }

// Order — durable, source of truth, one per seller (sub-order) grouped by a checkout_group_id
{ "order_id", "checkout_group_id", "buyer_id", "seller_id",
  "line_items": [ {listing_id, qty, unit_price, tax} ],
  "amounts": {subtotal, shipping, tax, total, currency},
  "status": "CREATED|CONFIRMED|CANCELLED|...",
  "payment_ref", "reservation_ids", "status_history": [ {status, at} ],
  "idempotency_key", "created_at", "updated_at" }

// InventoryReservation — transient hold, TTL ~15 min
{ "reservation_id", "listing_id", "seller_id", "qty", "checkout_group_id",
  "state": "HELD|COMMITTED|RELEASED", "expires_at" }

// PaymentAttempt
{ "payment_id", "checkout_group_id", "psp", "psp_token", "amount", "currency",
  "state": "AUTHORIZED|CAPTURED|VOIDED|FAILED", "psp_reference", "idempotency_key" }

// IdempotencyRecord
{ "idempotency_key", "request_fingerprint", "response_snapshot", "state", "expires_at" }
```

- **Source of truth** = the transactional **Order DB** (sharded SQL, strong consistency) and
  **Inventory DB**. Payment truth is reconciled against the PSP.
- A cart from N sellers becomes **N sub-orders** under one `checkout_group_id` — the saga must
  handle partial success (one seller's item out of stock while others succeed).

---

## 5. Canonical High-Level Architecture (names to reuse)

```
Client ─▶ CDN/Edge ─▶ API Gateway ─▶ Checkout Orchestrator (SAGA)
                                        │  coordinates, holds no long state, idempotent
                                        ├─▶ Cart Service
                                        ├─▶ Pricing & Promotions Service
                                        ├─▶ Tax Service
                                        ├─▶ Inventory Service ──▶ Inventory DB (reservations)
                                        ├─▶ Payment Service ──▶ PSP Adapters ──▶ external PSPs
                                        ├─▶ Order Service ──▶ Order DB (sharded, strong-consistent)
                                        └─▶ Idempotency Store (Redis + durable)
                                        ▼
                                      Kafka (order.placed, saga events, outbox)
                                        ▼
                         Notification Service · Fulfillment (downstream) · Analytics
```

Canonical component names (use these exact names across chapters):
**CDN/Edge, API Gateway, Checkout Orchestrator (the saga coordinator), Cart Service,
Pricing & Promotions Service, Tax Service, Inventory Service, Payment Service, PSP Adapters,
Order Service, Notification Service, Idempotency Store (Redis), Kafka, Order DB, Inventory DB.**

### Canonical saga for `place-order` (orchestration, not choreography)
1. **Revalidate** session (prices, promos, tax still valid; buyer still eligible).
2. **Reserve inventory** for each sub-order (Inventory Service) — HELD with TTL.
3. **Authorize payment** for the grand total (Payment Service → PSP).
4. **Create order(s)** in Order DB (status CONFIRMED), commit reservations.
5. **Capture payment** (now, or deferred to fulfillment) and emit `order.placed`.
- **Compensations** (reverse order): void authorization, release reservations, cancel order.
- Every step is **idempotent and keyed** by `checkout_group_id` + `Idempotency-Key`.

---

## 6. Canonical Consistency & Correctness Stance (for Ch6/7/8/11)
- **Strong consistency** for: inventory reservation (no oversell), payment (no double-charge),
  order creation (no lost/duplicate orders). This is a deliberate **CP** choice on the money path.
- **Eventual consistency** acceptable for: order-history reads, seller dashboards, analytics,
  recommendations.
- **Exactly-once effect** is achieved via **at-least-once delivery + idempotency keys**, not by
  distributed 2PC across services. Saga + idempotency, not XA transactions.
- **Reconciliation** jobs continuously compare our payment/order records against the PSP and
  inventory ledger to catch and repair drift.

---

## 7. House Style for Every Chapter

### 7.0 LANGUAGE — SIMPLE ENGLISH (the most important rule)
The reader is a **non-native English speaker** and wants **simple, easy-to-read English**
(Indian-English friendly). Follow these rules strictly:
- **Short sentences. One idea per sentence.** Aim for ~15–20 words. Split any longer sentence.
- Use **common, everyday words**. Avoid rare, literary, or fancy vocabulary.
- **No idioms, slang, metaphors, or cultural phrases.** Do not write things like "sets fire to
  the budget", "the nasty one", "eats your candidates". State the plain meaning instead.
- Avoid long sentences joined by many dashes, semicolons, or brackets. Prefer separate sentences.
- Use **active voice** and direct statements. ("The service reserves the stock." not "Stock is
  reserved by the service.")
- **Define every technical term in one simple sentence the first time it appears.**
- Explain the idea in plain words first, then give the detail. Use concrete examples with numbers.
- Short paragraphs (3–5 sentences). Use simple connectors: "So", "But", "Also", "For example",
  "This means", "In short".
- It is okay to repeat a key idea in simple words to make it clear.
- **Keep the full technical depth and accuracy — only the language becomes simpler, not the content.**

### 7.1 Other style rules
- **Voice:** a senior engineer teaching a learner, plus interview coaching. Explain the *why*, then the *how*.
- Use **ASCII or mermaid diagrams**, **tables**, **worked numeric examples**, **state machines**,
  and concrete **JSON / SQL / pseudo-code** where relevant.
- Each chapter ends with **"Interview Tips"** (what to say / common traps) and **"Key Takeaways"**.
- Refer back to earlier chapters by number. Reuse the canonical numbers/names above verbatim.
- Target the page counts in the README. A "page" ≈ ~450–550 words of substantive content
  (no filler). Prefer depth and concrete detail over padding.
- Recurring themes to weave through: **idempotency, saga + compensations, no-oversell,
  no-double-charge, graceful degradation, reconciliation, and the CP-vs-AP contrast with the
  Search handbook.**
