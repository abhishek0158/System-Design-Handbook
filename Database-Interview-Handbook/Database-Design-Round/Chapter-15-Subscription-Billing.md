# Chapter 15 — Subscription Billing / SaaS

A SaaS (Software as a Service) product charges customers on a repeating schedule — monthly or
yearly. This is a common design question because it tests two things: can you model a
subscription's lifecycle correctly, and can you charge money safely, even when a network retry
happens.

## Requirements & Clarifying Questions

**Key features to support:**
- A customer picks a plan (e.g. "Pro", $29/month) and subscribes.
- The system bills the customer once per billing period (e.g. every month).
- A subscription can be trialing, active, past due (payment failed), or canceled.
- A customer can see their invoices (bills) and payment history.
- A plan's price can change later, but old invoices must show the old price.

**Clarifying questions to ask:**
- Do we support free trials before the first charge? (We assume yes — a `trialing` state.)
- What happens if a payment fails? Do we retry, downgrade, or cancel the subscription? (We
  assume a `past_due` state with a few retries, then cancellation. We note this is a policy
  choice, not just a schema choice.)
- Do we support switching plans mid-period (upgrade/downgrade)? This raises proration — charging
  or crediting a partial amount for the switch. (We assume yes, and cover it briefly below.)
- Is billing usage-based (e.g. per API call) or flat-rate per period? (We assume flat-rate per
  period here. Usage-based billing adds a metering table and is a related, larger topic.)
- Who actually charges the card — do we call a payment gateway like Stripe? (We assume yes. Our
  `payments` table records the *result* of that call; the gateway itself is external.)

**Rough scale:** Assume 200,000 customers, mostly on monthly plans. That is about 200,000
invoices generated a month, and a similar number of payment attempts. Small compared to a
consumer app, but every charge must be correct — a duplicate charge is a real complaint, not just
a bug.

## The Schema

```sql
CREATE TABLE customers (
    customer_id     BIGSERIAL PRIMARY KEY,
    email           TEXT NOT NULL UNIQUE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
    -- ...name, billing address, default payment method reference
);
-- One row per paying account.

CREATE TABLE plans (
    plan_id         BIGSERIAL PRIMARY KEY,
    name            TEXT NOT NULL,             -- e.g. 'Pro'
    price_minor     BIGINT NOT NULL,           -- price in minor units (cents), e.g. 2900 = $29.00
    currency        TEXT NOT NULL DEFAULT 'USD',
    interval        TEXT NOT NULL,             -- 'month' or 'year'
    is_active       BOOLEAN NOT NULL DEFAULT true  -- old plans stay for history, but can't be picked
);
-- The current, live price list. A plan's price CAN change over time (see Decision 3).

CREATE TABLE subscriptions (
    subscription_id       BIGSERIAL PRIMARY KEY,
    customer_id           BIGINT NOT NULL REFERENCES customers(customer_id),
    plan_id               BIGINT NOT NULL REFERENCES plans(plan_id),
    status                TEXT NOT NULL,       -- 'trialing' | 'active' | 'past_due' | 'canceled'
    current_period_start  TIMESTAMPTZ NOT NULL,
    current_period_end    TIMESTAMPTZ NOT NULL,
    canceled_at           TIMESTAMPTZ,
    updated_at            TIMESTAMPTZ NOT NULL DEFAULT now()
);
-- One row per customer's subscription to one plan. status + period columns drive all billing.
CREATE INDEX idx_subscriptions_billing_due
    ON subscriptions(current_period_end) WHERE status IN ('trialing', 'active', 'past_due');

CREATE TABLE invoices (
    invoice_id         BIGSERIAL PRIMARY KEY,
    subscription_id    BIGINT NOT NULL REFERENCES subscriptions(subscription_id),
    amount_minor       BIGINT NOT NULL,        -- total owed, copied at invoice time (see Decision 3)
    status              TEXT NOT NULL,          -- 'open' | 'paid' | 'failed' | 'void'
    period_start        TIMESTAMPTZ NOT NULL,
    period_end          TIMESTAMPTZ NOT NULL,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);
-- One row per bill for one period. Never edited after payment is attempted.
CREATE INDEX idx_invoices_subscription ON invoices(subscription_id, created_at);

CREATE TABLE invoice_line_items (
    line_item_id    BIGSERIAL PRIMARY KEY,
    invoice_id      BIGINT NOT NULL REFERENCES invoices(invoice_id),
    description     TEXT NOT NULL,             -- e.g. 'Pro plan — Sep 1 to Sep 30'
    amount_minor    BIGINT NOT NULL,           -- this line's price, copied at invoice time
    quantity        INTEGER NOT NULL DEFAULT 1
);
-- Breaks one invoice into parts: base plan charge, proration credit, add-ons, etc.
CREATE INDEX idx_line_items_invoice ON invoice_line_items(invoice_id);

CREATE TABLE payments (
    payment_id        BIGSERIAL PRIMARY KEY,
    invoice_id        BIGINT NOT NULL REFERENCES invoices(invoice_id),
    amount_minor      BIGINT NOT NULL,
    status             TEXT NOT NULL,           -- 'pending' | 'succeeded' | 'failed'
    idempotency_key    TEXT NOT NULL,
    created_at         TIMESTAMPTZ NOT NULL DEFAULT now()
    -- ...gateway reference id (e.g. Stripe charge id)
);
-- One row per attempt to charge a card for an invoice. Guarded against double-charging.
CREATE UNIQUE INDEX uq_payments_idempotency ON payments(idempotency_key);
```

**`customers`** are who we bill. **`plans`** is the price list. **`subscriptions`** links a
customer to a plan and tracks its lifecycle. **`invoices`** are bills generated per period.
**`invoice_line_items`** break an invoice into priced parts. **`payments`** record charge
attempts against an invoice.

## Key Design Decisions — the Reasoning

**1. Subscription lifecycle: a `status` column, driven by a state machine.**
A state machine is a model where something can only be in one of a fixed set of states, and only
certain moves between states are allowed. Here, the states are `trialing` → `active` → `past_due`
→ `canceled`. A subscription starts in `trialing` (if there is a free trial) or `active`. When a
scheduled payment fails, it moves to `past_due`. If payment then succeeds (on retry), it goes back
to `active`. If it keeps failing, or the customer cancels, it moves to `canceled` — a final state,
it never leaves.

Why a single `status` column, and not a separate "current state" table: reads are simple and fast
("show me all active subscriptions") with one indexed column. The important discipline is that
your application code — not random `UPDATE` statements — must be the only thing that changes
`status`, and it must check "is this move allowed?" before writing. For example, code should
reject `canceled` → `active` directly (a canceled customer must create a *new* subscription, not
revive the old one). We add `updated_at` so we know when the last state change happened, and
`canceled_at` specifically because "when did they leave" is a common business question (churn
reports). If the interviewer asks for full history of every state change, add an append-only
`subscription_events` table (see Chapter 2's history pattern) — but for most designs, a status
column plus timestamps is enough, and keeps the schema light.

**2. Billing periods: `current_period_start`/`current_period_end` on the subscription, one invoice
per period.**
A billing period is the date range a payment covers — for a monthly plan, roughly 30 days. We
store the *current* period's start and end directly on `subscriptions`. A periodic job (see
Scaling It) looks for subscriptions whose `current_period_end` has passed, generates an invoice
for that period, attempts payment, and then advances the subscription to the next period (new
`current_period_start`/`current_period_end`, one interval later).

Why store the period on the subscription, instead of computing it from `created_at` each time:
plans can be paused, trials can run different lengths, and customers can change plans mid-cycle.
Storing the actual current window directly is simple and correct for all these cases — the job
never needs to re-derive "what period are we in" with date math that can drift or get a leap-year
edge case wrong. The `idx_subscriptions_billing_due` index makes "find everything due to bill
right now" a fast, indexed range scan, which matters once you have hundreds of thousands of
subscriptions.

**3. Copy the price onto the invoice and line item — never point back to the live plan price.**
This is the same "price at time of the event" pattern used across this handbook (see Chapter 3 on
orders). `plans.price_minor` is the *current* price — it can go up next year. But an invoice is a
legal record of what a customer was charged *at that time*. If `invoices.amount_minor` were
computed by joining to `plans` every time someone views an old invoice, then raising the Pro plan
price from $29 to $35 would silently rewrite every past invoice to say $35. That is wrong, and in
some places it is a compliance problem, not just a bug.

The fix: when the billing job creates an invoice, it reads the plan's price *at that moment* and
writes it directly into `invoices.amount_minor` and `invoice_line_items.amount_minor`. After that,
the invoice never changes, even if the plan's price changes ten times afterward. This is why
`invoice_line_items` exists separately from a single total on `invoices`: it lets one invoice show
a clear breakdown — "Pro plan: $29.00" as one line, "Proration credit: -$4.20" as another — each
frozen at its own price, while `invoices.amount_minor` is the sum, also frozen.

**4. Payments must be idempotent — a unique `idempotency_key` stops double charges.**
Idempotent means: doing the same operation twice has the same effect as doing it once. Charging a
card is a network call to a payment gateway. Network calls time out and get retried — by the
billing job itself, or by a background retry worker. If a charge actually succeeded on the
gateway's side, but the response was lost before our server saw it, a naive retry would charge the
customer's card a second time for the same invoice.

The fix is the same idempotency-key pattern as Chapter 8's wallet ledger: before attempting a
charge, generate one key per attempt (often deterministic, like `invoice_id` + attempt number),
and insert a `payments` row with that key inside a unique index, `uq_payments_idempotency`. If a
retry tries to insert the same key again, the unique constraint fails, and the code knows this
attempt was already made — it can look up the existing row's result instead of calling the
gateway again. Most real payment gateways (Stripe included) also accept an idempotency key on
their API call directly, so you get protection on both sides: our database, and theirs.

**5. Proration on a plan change, in one line.**
Proration means charging or crediting a partial amount when something changes mid-period, instead
of waiting for the next full bill. If a customer upgrades from a $10 plan to a $30 plan halfway
through a 30-day period, a fair charge is roughly for the 15 days remaining at the new price, minus
a credit for the 15 unused days at the old price. In our schema, this becomes two extra
`invoice_line_items` rows on the next invoice (one negative credit line, one positive charge
line) — the math is a policy decision (how you round, whether you bill immediately or wait), but
the schema already supports it, because line items are already separate, priced rows.

## Scaling It

**Invoices and payments only grow — partition and archive old ones.** Every period, every active
subscription adds one invoice row and at least one payment row. After a few years at scale, these
tables hold tens of millions of rows, but almost all reads are for recent invoices (this month's
bill, last payment). Partition `invoices` and `payments` by month (on `created_at`), and move old
partitions to cheaper, slower storage once they are past the age most support and analytics
queries need. Keep them queryable for audits, just not on the fast, expensive tier.

**The billing job runs periodically and must be safe to re-run.** A scheduled job (e.g. every
hour) scans for subscriptions where `current_period_end` has passed, using the
`idx_subscriptions_billing_due` index so it only looks at the small set actually due — not the
full table. Because the job can crash mid-run or be triggered twice, each step it takes (create
invoice, attempt payment, advance the period) should be safe to retry: check "does an invoice
already exist for this subscription and period?" before creating a new one, the same way payments
check the idempotency key before charging.

**Reads (customer viewing their invoices) are simple — index and maybe cache.** A customer's
invoice list is one indexed lookup on `idx_invoices_subscription`. This read pattern rarely needs
more than an index; a cache in front of it only matters if invoice history pages become unusually
hot, which is uncommon for this kind of workload.

## Interview Tips & Common Mistakes

- **Say "copy the price onto the invoice" out loud, unprompted.** This is the single most tested
  idea in this chapter. A design that computes an old invoice's total from the live `plans` table
  is a common and serious mistake.
- **Name the states of the subscription lifecycle clearly.** `trialing`, `active`, `past_due`,
  `canceled` — and say which moves are allowed (e.g. `canceled` is final; a customer must
  re-subscribe, not "un-cancel").
- **Do not forget the idempotency key on payments.** If asked "what if the retry happens after the
  charge already succeeded," a design without a unique key on the attempt will double-charge a
  real customer's card — this is treated as a serious red flag, just like in wallet questions.
- **Do not store money as float.** Always `_minor` integer columns, said out loud.
- **Mention proration, even briefly, if plan changes come up.** You do not need the exact rounding
  formula in an interview — you need to say "extra line items on the next invoice, at the old and
  new price" to show you understand the idea.
- **Do not model billing periods only with `created_at`.** Deriving "which period am I in" purely
  from account creation date breaks on trials, pauses, and plan switches. Store the current
  period's start/end directly, and advance it explicitly each cycle.
