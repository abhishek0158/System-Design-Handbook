# Chapter 1 — Requirement Gathering

This chapter is about the first step of designing a checkout system: gathering requirements.
We will design the checkout system for **GlobalMart**, a large online marketplace. GlobalMart
has 2 billion registered users and 500 million daily active users. Many different sellers list
products on GlobalMart. A buyer's cart can hold items from many sellers at once.

Checkout is the part of the system that turns a shopping cart into a paid, confirmed order.
This sounds simple. But at GlobalMart's scale, it is one of the hardest parts of the whole
system to get right. In this chapter, we will learn how to gather requirements for this system
in a structured way, the same way you should do it in a real interview.

## 1.1 Why Requirements Matter Most

In any system design interview, many candidates rush to draw boxes and arrows. They want to
show off their knowledge of databases and caches. But this is a mistake. If you do not
understand the requirements first, you may build a fast system that does the wrong thing.

For checkout, this mistake is very costly. Checkout moves real money and real inventory. A
small mistake here does not just slow down a page. It can charge a customer twice for one
order. It can sell the same product to ten buyers when there is only one unit left. These are
not small bugs. They cost the company money, and they break customer trust.

This gives us the single most important idea in this whole handbook:

> **Correctness before latency.**

This means: it is more important that checkout is correct than that checkout is fast. A slow
checkout page is annoying. A checkout page that charges the wrong amount, or sells stock that
does not exist, is a serious business problem. So when we later have to make trade-offs
between speed and correctness, we will almost always choose correctness.

Compare this to a search system. If a search system is slow by 200 milliseconds, or if it
shows a slightly wrong ranking of products, the harm is small. The buyer still finds a good
product. So search systems often choose to be fast and "mostly right" over being slow and
"exactly right." Checkout cannot make this choice on the money path. We will return to this
contrast many times in this handbook.

This does not mean speed does not matter. It means we rank correctness above speed. A checkout
system that is correct but a little slow is acceptable. A checkout system that is fast but
sometimes double-charges buyers is not acceptable, no matter how fast it is.

## 1.2 A Simple Scoping Framework

Before listing detailed requirements, it helps to use a simple framework. This framework works
for any system design interview, not just checkout. It has five steps.

**Step 1: Confirm the scenario.** State the business in one sentence. For us: "GlobalMart is a
global marketplace with many sellers. A buyer's cart can have items from several sellers.
Checkout must turn that cart into one or more paid, confirmed orders."

**Step 2: Identify the main actors.** Who uses this system? Here we have three actors: the
**buyer** (places the order), the **seller** (owns the inventory and receives the order), and
the **payment service provider**, or **PSP** (a third-party company, like a card network or
wallet provider, that actually moves the money).

**Step 3: List the core user journeys.** What are the main steps a buyer takes? For checkout,
this is: open the cart, choose a shipping address, see the final price, add a payment method,
review everything, and place the order.

**Step 4: Split requirements into functional and non-functional.** Functional requirements
describe *what* the system does. For example, "the system must let a buyer place an order."
Non-functional requirements describe *how well* the system does it. For example, "the system
must respond in under 300 milliseconds" or "the system must never double-charge a buyer."

**Step 5: Write down assumptions and out-of-scope items.** No interview or design document can
cover everything. State clearly what you assume, and what you will not cover. This way everyone
agrees on the boundary of the problem.

We will now use this framework to build the requirements for GlobalMart checkout.

## 1.3 Functional Requirements (FR1–FR8)

Functional requirements describe the features the checkout system must support. We group them
into eight requirements, numbered FR1 to FR8. We use a running example: **a buyer has a cart
with items from three different sellers — Seller A, Seller B, and Seller C.**

### FR1 — Create a checkout session

When a buyer decides to check out, the system creates a **checkout session**. A checkout
session is a temporary record. It holds the buyer's cart, along with the price, shipping cost,
tax, and any discounts.

In our example, the buyer's cart has items from Seller A, Seller B, and Seller C. The system
splits this one cart into **three sub-orders**, one for each seller. All three sub-orders are
linked together under one checkout session. This way the buyer sees one clean checkout screen.
But underneath, GlobalMart tracks each seller's part separately. This is needed because each
seller ships and gets paid separately.

A checkout session does not last forever. It expires after about 30 minutes if the buyer does
not finish. This keeps the system clean. It also stops old, stale sessions from causing
problems later.

### FR2 — Address and shipping selection

The buyer must be able to choose or enter a shipping address. Once the address is known, the
system calculates shipping options and their costs. Different sellers may offer different
shipping options. For example, Seller A might offer next-day delivery. Seller C might only
offer a five-day standard option. The system must show correct shipping costs for each
sub-order.

### FR3 — Price, tax, and promotions (authoritative re-pricing)

The price the buyer saw while browsing the catalog is not always the final price at checkout.
Prices can change. A discount code might apply. Tax depends on the shipping address, which the
buyer may not have entered yet while browsing.

So at checkout time, the system must calculate the **final, correct price** again. We call this
**re-pricing**. For example, a product priced at $50 in the catalog might now be on sale for
$45. Or a new 8% tax might apply because of the buyer's state. The buyer must always see the
true final price before placing the order. They should never see the old catalog price.

### FR4 — Payment method

The buyer must be able to add a payment method. This can be a credit or debit card, a digital
wallet, a gift card, or even a split across more than one method. The system does not store the
raw card number. Instead, it uses a **token**. A token is a safe, random-looking ID that stands
in for the card number. The PSP creates this token. We explain this more in Section 1.4 under
PCI compliance.

### FR5 — Place order (idempotent)

This is the single most important operation in checkout. When the buyer clicks "Place Order,"
the system must:

1. Reserve the inventory for each item.
2. Authorize the payment for the full amount.
3. Save the order (or orders) permanently.
4. Confirm the order back to the buyer.

This must feel like one single, all-or-nothing action to the buyer. But really it is several
steps across several services. We call this operation **idempotent**. Idempotent means: if the
same request is sent more than once, the result stays the same as sending it once. For example,
the buyer's phone might lose network connection, and the app might retry the request. Even
then, the buyer is charged only once. Only one order is created, no matter how many times the
request is retried.

GlobalMart achieves this using an **Idempotency-Key**. This is a unique ID the client sends
with each place-order request. If the same key is sent twice, the system returns the same saved
result instead of doing the work again. We study this in full detail in Chapter 8.

### FR6 — Inventory reservation

Before we can confirm an order, we must make sure the stock really exists. Imagine Seller B has
only 1 unit of a popular phone case left. If 500 buyers try to buy it at the same time, only one
buyer can succeed. The system must **hold**, or reserve, the stock for a buyer during checkout.
This stops a second buyer from also "winning" the same unit.

This hold is temporary. It has a timeout of about 15 minutes. If the buyer does not finish
checkout in that time, or if checkout fails, the hold is released. This gives back the stock so
another buyer can buy it. We call this rule **no oversell**: the system must never promise more
stock than truly exists.

### FR7 — Order lifecycle and status

Once an order exists, it moves through different statuses over time. The simplest path is
**CREATED**, then **CONFIRMED**, and then it moves on to fulfillment. Fulfillment means packing
and shipping the order, and it is handled by a different system outside our scope. The buyer
must be able to check the current status and the full history of status changes at any time.
This is a simple query, like "where is my order."

### FR8 — Notifications and downstream events

Once an order is confirmed, other parts of GlobalMart need to know. The warehouse needs to
start packing it. The buyer needs a confirmation email. The analytics team needs the data for
reports. The checkout system announces this by emitting an event called `order.placed`. Other
services listen for this event and react to it. This keeps checkout itself simple. It does not
need to know the details of shipping or email. It only needs to announce that the order
happened.

### Putting FR1–FR8 together: the three-seller example

Let's walk through our three-seller example end to end. The buyer has: 2 shirts from Seller A
($30 total), 1 phone case from Seller B ($15), and 1 book from Seller C ($20).

1. FR1 creates one checkout session with three sub-orders: A, B, and C.
2. FR2 collects one shipping address. But shipping cost may differ per seller.
3. FR3 recalculates the true price, tax, and any discounts for each sub-order.
4. FR4 attaches one payment method for the whole session.
5. FR6 reserves 2 shirts, 1 phone case, and 1 book. This is one reservation per seller.
6. Suppose Seller B's phone case just sold out to another buyer a moment ago. FR6's reservation
   for Seller B fails.
7. The system must decide what to do with this **partial failure**. GlobalMart's design lets
   the sub-orders for Seller A and Seller C still succeed. Seller B's sub-order is cancelled,
   and the buyer is told that item is no longer available. The buyer is only charged for what
   actually succeeds.
8. FR5 places the two successful sub-orders as real, confirmed orders.
9. FR7 lets the buyer track both orders separately. Seller A and Seller C ship and arrive
   separately.
10. FR8 emits `order.placed` twice, once for each confirmed sub-order.

This example shows why multi-seller checkout is genuinely harder than single-seller checkout.
It is not one atomic transaction across the whole cart. It is a **saga**. A saga is a sequence
of steps across several sellers and services, where each step can succeed or fail on its own.
Failures must be handled cleanly, instead of leaving things half-done. We introduce the saga
pattern fully in Chapter 6.

## 1.4 Non-Functional Requirements: Ranking Them

Non-functional requirements describe the quality of the system, not its features. For
checkout, we must be clear about which qualities matter most. Sometimes they conflict with each
other. Here is the ranked list, from most important to least important.

**1. Correctness — no money or inventory errors.** This is the top priority, above all else,
including speed. Two specific promises matter most:

- **No double-charge.** A buyer's payment method must never be charged twice for one order,
  even if the network fails and the request is retried.
- **No oversell.** A seller's stock must never be sold to more buyers than the true quantity
  available.

We also must never "lose" an order. This means a case where the buyer was charged, but no order
record was ever saved. Every rupee or dollar charged must map to exactly one real order.

**2. High availability — 99.99%.** The checkout path must be available 99.99% of the time.
This sounds close to 100%, but the difference matters a lot. 99.99% availability allows only
about **53 minutes of downtime per year**. This target is strict, because every minute checkout
is down is lost revenue. If checkout is down for even 10 minutes during a big sale, GlobalMart
loses real orders that never come back. Buyers simply leave.

**3. Low latency.** The checkout review page (seeing your cart and price) should respond in
under 300 milliseconds for 99 out of 100 requests. We write this as **p99 ≤ 300 ms**. The
actual place-order step is allowed more time, up to **p99 ≤ 2.5 seconds**. This is because it
must wait for the external payment network to approve the charge. That step alone usually takes
300 to 1500 milliseconds. Latency matters, but it is ranked below correctness and availability.

**4. Surge tolerance.** GlobalMart runs flash sales. In a flash sale, a huge number of buyers
try to buy the same limited-stock item at the same moment. The system must absorb spikes of
**10 to 20 times** the normal traffic without overselling. On a normal day, orders arrive at
about 2,300 per second. During a flash sale or a Black Friday-type event, this can spike to
about 50,000 orders per second. The system must survive this spike without breaking
correctness, even if it must slow down or queue some requests.

**5. Consistency model.** Not every part of checkout needs the same level of consistency.
**Strong consistency** means every reader instantly sees the latest, guaranteed-correct value.
This is required for inventory reservation, payment, and order creation, because these involve
real money and real stock. **Eventual consistency** means a reader might briefly see slightly
stale data before it catches up. This is acceptable for things like order history pages, seller
dashboards, recommendations, and analytics. Nobody is harmed if an order history page takes one
extra second to show a new order. But real harm happens if two buyers are both told they
successfully bought the last unit.

**6. Auditability and PCI compliance.** Every movement of money must be traceable. GlobalMart
must be able to prove exactly what happened for any order, for accounting and legal reasons.
Also, **PCI DSS** is a security standard for handling card payments. It says raw card numbers
should never touch GlobalMart's own servers. Instead, the buyer's device sends the card details
straight to the PSP. The PSP returns a safe token, a random-looking ID that represents the
card. GlobalMart stores only this token and the last four digits of the card. It never stores
the full card number. This is both a legal requirement and good security practice.

### Why this ranking, and why it can conflict

These six qualities can pull in different directions. For example, being very strict about
correctness, like double-checking every step and waiting for confirmations, can add latency.
Being very fast, like skipping checks and assuming success, can hurt correctness. The ranking
tells us how to break these ties. When correctness and speed conflict, we choose correctness.
When availability and strict consistency conflict in some edge case, we lean toward
availability for non-money reads, like showing order history. But we never do this for the
money path itself.

## 1.5 Clarifying Questions to Ask the Interviewer

In a real interview, you do not get requirements handed to you. You must ask good clarifying
questions. This also shows the interviewer that you think like an engineer who plans before
building. Below are grouped questions, with a short reason for each.

**Group 1 — Scope of "checkout"**
- "Does checkout include catalog browsing and search, or does it start once the buyer has a
  cart?" *Why it matters:* this sets the boundary of the whole design. We assume checkout
  starts from an existing cart.
- "Should I design refunds and returns in detail, or just mention them?" *Why it matters:*
  refunds are a large topic on their own. We will only mention them as a compensating action.

**Group 2 — Scale**
- "How many orders per day, and what is the peak traffic during sales?" *Why it matters:* the
  whole architecture, including sharding, caching, and queueing, depends on this number. We use
  200 million orders per day, peaking near 50,000 orders per second.
- "How many sellers can be in a single cart?" *Why it matters:* this decides how complex the
  saga and sub-order logic must be. We assume a cart can span many sellers, as shown in our
  three-seller example.

**Group 3 — Money and consistency**
- "Is it acceptable to briefly oversell and fix it later with a refund, or must we prevent
  oversell entirely?" *Why it matters:* this decides whether we need real-time inventory
  reservation, which is strict but more complex, or a simpler "sell then fix later" approach.
  For GlobalMart, we must prevent oversell up front. Fixing it after the fact damages trust and
  creates support costs.
- "Do we need exactly-once payment charging, or is at-least-once with deduplication okay?"
  *Why it matters:* true exactly-once delivery across a network is very hard to guarantee. We
  will use at-least-once delivery plus idempotency keys. This gives the same safe result in
  practice.

**Group 4 — Multi-seller behavior**
- "If one seller's item fails during checkout, should the whole order fail, or should the other
  sellers' items still go through?" *Why it matters:* this decides our partial-failure policy.
  We assume the other sellers' sub-orders can still succeed on their own, as shown in our
  three-seller example.

**Group 5 — Payments**
- "Do we integrate with one payment provider or many, such as cards, wallets, and
  region-specific methods?" *Why it matters:* this affects how much abstraction the Payment
  Service needs. We assume multiple PSPs, reached through PSP Adapters. This way GlobalMart is
  not locked to one provider.
- "Is card data allowed to touch our servers at all?" *Why it matters:* this is a compliance
  question, not just a technical one. The answer is no. This drives the tokenization design in
  FR4.

**Group 6 — Availability and global reach**
- "Is this a single region system, or does it need to run in multiple regions actively?" *Why
  it matters:* multi-region active-active designs are far more complex, especially for strong
  consistency needs like inventory. We assume a multi-region, active-active design, matching
  GlobalMart's global scale. We cover this in depth in Chapter 9.

Asking these questions early prevents wasted design effort later. It also signals to an
interviewer that you understand which decisions are foundational, and which are just details.

## 1.6 Assumptions and Out-of-Scope

Since we cannot ask a real interviewer in this handbook, we state clear assumptions instead.

**Assumptions:**
- The buyer already has a cart. We start our design from checkout, not from browsing or
  search.
- A cart can contain items from multiple sellers. This becomes multiple sub-orders under one
  checkout session.
- GlobalMart already has separate systems for catalog, search, and cart management. Checkout
  calls into these systems but does not own them.
- Card numbers are handled by the buyer's device and the PSP directly. GlobalMart's servers
  only ever see a token, never the raw card number.
- The system runs across multiple regions in an active-active setup. This means more than one
  region can serve live traffic at the same time.

**Out of scope for this handbook:**
- The catalog and search system itself. This is a separate handbook.
- The fulfillment and warehouse systems, and shipping-carrier integrations. Checkout only hands
  off work to them by emitting the `order.placed` event.
- The detailed returns and refunds user interface. We mention refunds only as a "compensating
  action," used when a saga step must be undone.
- The seller payout and settlement ledger. This means how and when GlobalMart actually pays
  sellers their share of revenue. We mention it only briefly.

Stating these boundaries clearly, out loud, in an interview is a strong signal. It shows you
know the difference between "core to this problem" and "a whole other system."

## 1.7 Success Metrics

Once we build the checkout system, how do we know it is actually working well? We define clear,
measurable success metrics. Some measure business health. Some measure technical health.

| Metric | What it means | Target |
|---|---|---|
| Conversion rate | Percent of checkout sessions that become real orders | Track and improve; baseline ~33% |
| Checkout success rate | Percent of "place order" attempts that succeed without system error | As close to 100% as possible |
| Double-charge rate | Fraction of orders where a buyer was charged more than once | **Must be 0** |
| Oversell rate | Fraction of orders sold beyond true available stock | **Must be 0** |
| p99 latency (place-order) | 99% of place-order requests finish within this time | ≤ 2.5 seconds |
| p99 latency (review/cart pages) | 99% of review page loads finish within this time | ≤ 300 milliseconds |
| Payment success rate | Percent of payment authorization attempts that succeed | Track against PSP baseline; a low rate signals PSP or fraud-rule issues |
| Reconciliation drift | Difference between our recorded orders/payments and the true records at the PSP and inventory ledger | Should trend to 0; any drift must be found and fixed automatically |

Two of these targets are absolute, not "as low as possible." The **double-charge rate** and the
**oversell rate** must be exactly zero. This is unusual. Most systems accept a small error rate
as a normal cost of running at scale. Checkout cannot accept this. Each such error is a
directly provable mistake with real money, and it destroys buyer trust. In later chapters,
especially Chapter 6, 7, and 8, we will see exactly which mechanisms make a "zero" target
realistic, not just a hopeful number. These mechanisms are idempotency keys, inventory
reservations, and reconciliation jobs.

**Reconciliation drift** deserves one more sentence, since the term is new. Reconciliation
means regularly comparing GlobalMart's own records against an outside source of truth. This
outside source can be the PSP's payment records, or the real inventory count. Drift means a
mismatch between them. A healthy system finds and fixes drift automatically, in minutes. It
should not need a human to notice a customer complaint days later.

## 1.8 Turning Requirements into a Design Contract

Once requirements are gathered, we do not just remember them informally. We turn them into a
**design contract**. This is a short, explicit list of promises the system makes. Every later
design decision must honor these promises. Think of it as the rules of the game for the rest of
this handbook.

Our design contract for GlobalMart checkout is:

1. **Every place-order request is idempotent.** Retrying with the same Idempotency-Key never
   creates a second charge or a second order.
2. **Inventory is never oversold.** A reservation step always runs before payment is
   authorized. Reservations expire safely if unused.
3. **Money and inventory operations are strongly consistent.** Everything else, like history,
   analytics, and dashboards, is allowed to be eventually consistent.
4. **Failures are handled by compensation, not left half-done.** If step 3 of a saga fails,
   steps 1 and 2 are actively undone, not just abandoned.
5. **Checkout stays available even under 10 to 20 times traffic spikes.** This works by design
   choices like queuing and graceful degradation, not by hoping traffic stays low.
6. **No raw card data ever touches GlobalMart's servers.**

This is the same idea as an API contract, just at the level of the whole system's behavior.
Every chapter after this one is really just an answer to one question: "What design makes these
six promises actually true, at 200 million orders a day?"

### The simple contrast with search: AP versus CP

It helps to compare checkout to its sibling system, search, which we assume was designed
earlier for the same GlobalMart platform. Search and checkout make **opposite** choices about
consistency. This is worth remembering clearly.

**Search chooses AP.** AP stands for "Availability and Partition tolerance." This comes from a
rule called the CAP theorem. The CAP theorem says a distributed system cannot always guarantee
perfect availability and perfect consistency at the same time during a network problem. Search
picks availability. It would rather show a slightly outdated list of products fast, than show
no results, or wait while it double-checks that every number is perfectly fresh. If a product's
stock count in search results is a few seconds old, no real harm is done.

**Checkout chooses CP on the money path.** CP stands for "Consistency and Partition tolerance."
For the specific steps that touch money and inventory, like reserving stock, charging a card,
and creating an order, checkout picks strong consistency over always being available. It would
rather briefly reject a request, or make a buyer wait an extra moment, than risk a wrong result.
A wrong result here means something like a double-charge or a sold-twice item.

This does not mean checkout is CP everywhere. As we said in Section 1.4, checkout is happy to
be eventually consistent for order history pages or analytics, similar to search. The important
point is this: **checkout is deliberately CP exactly where money and inventory are decided, and
can relax elsewhere.** This one sentence is one of the most important ideas in this entire
handbook. We will return to it again and again in later chapters.

## Interview Tips

- **Say "correctness before latency" early, out loud.** This one sentence tells the
  interviewer you understand what makes checkout different from a typical search or feed
  system. It sets the tone for every later decision you will make.
- **Do not skip clarifying questions to save time.** Interviewers often deliberately leave
  requirements vague, to see if you notice. Asking about multi-seller carts, oversell policy,
  and card data handling shows real experience, not just memorized diagrams.
- **Use a concrete example immediately.** Instead of talking about "the system" in the
  abstract, say "imagine a cart with items from three sellers" and walk through it. Concrete
  examples make your reasoning easy to follow and hard to poke holes in.
- **State the double-charge and no-oversell rules as absolute, not "best effort."** Many
  candidates say "we'll try to minimize double charges." A stronger answer is: "double-charge
  and oversell must be zero, and here is the specific mechanism that makes that true." That
  mechanism is idempotency keys plus inventory reservation, which we build in later chapters.
- **Bring up the CP-versus-AP contrast with search if you can.** It shows you can compare
  systems, not just describe one in isolation. This is exactly the kind of insight that
  separates a senior-level answer from a junior one.
- **A common trap: designing before scoping.** If you find yourself drawing a database schema
  in the first two minutes of an interview, stop. Go back to requirements first. A well-scoped,
  simply stated problem is worth more than an early, unscoped diagram.

## Key Takeaways

- Checkout is revenue-critical and correctness-critical. The guiding rule for the whole system
  is **correctness before latency**.
- A simple five-step scoping framework works for any system design interview, not just
  checkout: confirm the scenario, identify the actors, list the core journeys, split
  functional versus non-functional requirements, and state assumptions.
- The eight functional requirements, FR1 to FR8, cover creating a session, shipping, pricing
  and tax, payment methods, idempotent order placement, inventory reservation, order lifecycle,
  and notifications. A multi-seller cart becomes multiple linked sub-orders. One seller's
  failure should not have to block the others.
- Non-functional requirements are ranked, not just listed. Correctness comes first, meaning no
  double-charge and no oversell. Then comes 99.99% availability, which allows about 53 minutes
  of downtime per year. Then latency, with a p99 of 2.5 seconds for place-order and 300
  milliseconds for review. Then surge tolerance for 10 to 20 times traffic spikes. Then a mixed
  consistency model. Then auditability and PCI compliance.
- Good clarifying questions about scope, scale, money-consistency trade-offs, multi-seller
  behavior, payments, and multi-region availability show interview maturity. They also shape
  the entire design.
- Success is measured with clear metrics. Two of them are non-negotiable zeros: the
  double-charge rate and the oversell rate.
- Requirements become a short, explicit **design contract**. This is a fixed set of promises:
  no double-charge, no oversell, strong consistency for money, compensation on failure, surge
  survival, and no raw card data on our servers. Every later chapter must honor this contract.
- The simplest way to remember checkout's core stance is this: **search is AP, checkout is CP
  on the money path.** Checkout can still relax to eventual consistency for non-money reads,
  like order history and analytics.
