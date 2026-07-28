# Chapter 11 — Trade-offs

## 11.0 Why this chapter matters

Every earlier chapter made a choice and moved on. We picked a saga over two-phase commit. We picked orchestration over choreography. We picked strong consistency for inventory and payment. We picked reservations with a time limit.

This chapter goes back to each choice. For each one, we explain the other option, what it would have cost us, and the one sentence an interviewer wants to hear. Interviewers rarely mind if you pick a saga. They do mind if you cannot explain what a saga gives up. Good judgment, not memory, is what makes a design senior-level.

Every section below follows the same pattern. First, we state the decision. Then we compare the options in a table. Then we give our final call for GlobalMart Checkout, and why. Then we give one short line: "we would change our mind if…". Eleven decisions. No fence-sitting.

---

## 11.1 Saga vs. two-phase commit (2PC)

**The decision:** Inventory Service, Payment Service, and Order Service are three separate systems. Each one is owned, deployed, and scaled on its own. How do we make a change across all three feel like one atomic operation, without a real database transaction across all of them?

**Two-phase commit (2PC)** is the classic textbook answer. A coordinator asks every system to "prepare" a change. Each system locks its resources and promises to commit. Once every system says yes, the coordinator tells everyone to commit for real. It is simple to reason about, because there is no half-finished state visible to anyone. So why do we not use it?

| Point | 2PC | Saga (our choice) |
|---|---|---|
| Atomicity | Real all-or-nothing commit | Built from forward steps plus compensating (undo) steps. Looks all-or-nothing from outside, once finished |
| Locking | Each system holds locks from "prepare" until the coordinator's final decision | No locks held across systems. Each step commits right away, locally |
| What happens if the coordinator crashes | Every system stays stuck, holding its locks, until the coordinator comes back. This is 2PC's classic failure | The orchestrator can restart and read the last saved state. No system is stuck waiting for someone else's crash |
| Availability during a network problem | Low. A system that cannot reach the coordinator must wait | High. Each system keeps working on its own |
| Can outside partners join? | Every partner must speak the 2PC protocol (prepare, commit, abort) | Every partner just needs one normal API call and one undo API call |
| Can the payment provider (PSP) join? | **No.** Visa, Mastercard, Stripe, and Adyen do not offer a "prepare" step. A card charge is a real action with its own rules, not something you can lock and vote on | Not needed. "Authorize" is the forward step. "Void" is the undo step |
| Speed | Extra round trip for prepare, then another for commit | One pass through the steps, only as sequential as the data needs |
| What failure looks like | One stuck coordinator can block everyone | Failure leaves a clear, known state, like "inventory reserved, payment failed." Compensations fix it |

The most important row is "Can the payment provider join?" This is not a trade-off we chose. It is a hard wall. When we call the PSP to charge a card, money moves on rails we do not control. We only get a result back. There is no way for the PSP to "prepare" a charge and wait days for our system to say commit or abort. The moment one partner in the transaction is an outside payment network, 2PC cannot work, no matter how we feel about its other problems.

Even without the PSP, 2PC's "stuck forever" failure mode breaks our availability goal. Our target is 99.99% uptime, or about 53 minutes of downtime a year (see Chapter 2). If the coordinator crashes during prepare at our peak of 50,000 orders per second, thousands of reservations and payment locks would freeze until it recovers. That kind of outage is exactly what our SLA cannot afford.

The saga pattern (Chapter 6) trades away true atomicity. In exchange, it gives us clear, planned undo actions: release the inventory hold, void the payment, cancel the order. Every service stays available at all times. A "distributed transaction" becomes "a sequence of small local transactions, plus a recovery plan we already wrote."

**Our choice for GlobalMart:** a saga with compensations, run by an orchestrator (see 11.2). **We would change our mind** only if every partner, including the PSP, moved onto one shared system that supports prepare and commit. That is not realistic for outside payment networks today.

---

## 11.2 Orchestration vs. choreography

**The decision:** Inside the saga, who decides what happens next? A single coordinator, or each service reacting to messages from its neighbors?

**Choreography** means each service sends out an event when it finishes its work, like `inventory.reserved`. Other services listen for events they care about. No single place knows the whole flow. It emerges from many small reactions.

**Orchestration** means one component, the Checkout Orchestrator, calls each service directly, checks the result, and decides the next step, including which undo action to run if something fails.

| Point | Choreography | Orchestration (our choice) |
|---|---|---|
| Coupling between services | Low. Services only need to know the event format, not each other | Higher. The orchestrator knows every step and every service |
| Can you see the saga's current state? | Hard. To answer "where is checkout_group_id 12345 right now," you must piece together events from four or more services | Easy. The orchestrator's own record answers this in one lookup |
| Adding a new step, like a fraud check | You must update every service that needs to react to the new event | You only update the orchestrator's logic |
| Undo (compensation) logic | Spread out. Each service must know which of its own past actions to undo, based on which failure event it sees | Centralized. One place says "if capture fails, void the payment and release the reservation" |
| Debugging at 2 a.m. | Hard. You must gather logs across services and message queues | Easier. The orchestrator's own state and log tell the story |
| What happens if one part fails | High resilience. There is no single coordinator to fail | The orchestrator itself must be made highly available (Chapter 6), or it becomes a bottleneck |
| Best used for | Many independent listeners reacting to one fact, like "order placed" going to email, analytics, and fulfillment | A small, fixed number of steps that must happen in a strict order, with required undo actions |

Checkout is not "many independent listeners reacting to one fact." It is five fixed steps: revalidate, reserve, authorize, create order, capture. Step 2 cannot start before step 1 succeeds. If step 3 fails, we must undo a very specific set of earlier steps, not "whatever listener happens to be around." This is exactly the shape orchestration handles well and choreography handles badly.

We also need one clear answer to "what is the status of this checkout right now." Customer support, fraud review, and the buyer's own order-status page all need this answer fast. Rebuilding it from scattered events across five services, during a live Black Friday incident at 3 a.m., is a debugging tax we do not want to pay.

That said, we do use choreography after the saga finishes. The `order.placed` event on Kafka (our message queue) fans out to Notification Service, Fulfillment, and Analytics. This part really is "many independent listeners," with no required order and no undo actions between them. The lesson worth saying out loud: it is not orchestration *or* choreography for the whole system. It is orchestration for the tightly linked, undo-able core, and choreography for the loosely linked fan-out at the edges.

**Our choice for GlobalMart:** orchestration for the saga itself, choreography for everything after the order is confirmed. **We would change our mind** only if the number of steps grew so large, for example dozens of seller-specific checks, that the orchestrator's own logic became too big to manage. Even then, we would likely split it into smaller, still-orchestrated sub-sagas, not switch to pure choreography.

---

## 11.3 Strong consistency (CP) vs. eventual consistency (AP) on the money path

**The decision:** where do we spend effort on strong consistency, and where do we allow some staleness for the sake of speed and uptime?

A quick reminder of the terms. **CAP** is a rule that says a distributed system under a network split must choose between staying perfectly Consistent (every reader sees the same, latest data) or staying Available (every request gets an answer, even if the data might be slightly old). **CP** means we choose consistency. **AP** means we choose availability, and accept that data can be a little stale, or "eventual consistency."

This is not one company-wide setting. GlobalMart makes opposite choices on purpose, in two different systems.

| Data or action | Model we use | Why |
|---|---|---|
| Inventory reservation (Ch. 7) | Strong (CP) | Two buyers racing for the last unit must get one clear winner. Not "probably one winner, fixed later" |
| Payment authorize/capture (Ch. 8) | Strong (CP) | A double-charge takes real money from a real person. There is no such thing as "eventually not double-charged" |
| Order creation (Ch. 6) | Strong (CP) | An order is a legal and money record. A lost or duplicate order is a real incident, not a small annoyance |
| Order-history page ("my orders") | Eventual (AP) | A history page that is a few seconds old is a minor annoyance, not a bug that costs money |
| Seller dashboards, sales reports | Eventual | Sellers accept a dashboard being a few minutes behind. Nobody double-ships because of a stale number |
| "Buyers also bought" suggestions | Eventual, and can be skipped under load | Purely a nice-to-have. Show nothing rather than slow down checkout |
| Search and catalog browsing (the companion Search handbook) | AP everywhere | A stale search result, or a slightly wrong "in stock" badge on a browse page, costs nothing. It gets fixed on the next request |

**Why does the same company make opposite choices?** In the Search handbook, nothing is a "conserved quantity" that must never be wrong. A search index that is a few seconds old, or a ranking score from last night's data, makes results a bit less relevant. Nothing breaks. So search buys speed and uptime with both hands, and pays for it with some staleness, because staleness is cheap there.

Checkout is the mirror image. Money and inventory *are* conserved quantities. You cannot "fix a double-charge on the next request," because the money already left a real card. You cannot "un-sell" an item once two buyers both saw a success screen. There is no amount of eventual consistency that makes "sorry, we charged you twice" okay, the way "sorry, that search result was three seconds old" is a non-event.

So this is not really "checkout cares more about correctness than search does." The two systems are solving different problems. Search's mistakes are cheap and reversible. Checkout's mistakes on the money path are expensive and cannot be undone. Good engineering puts strong consistency where mistakes are irreversible, and spends availability everywhere else. This is also why Chapter 10's failure plan turns off *features* under stress, never *correctness*. We will gladly turn off recommendations or delay a notification email, before we ever loosen an inventory check or an idempotency rule.

**Our choice for GlobalMart:** strong consistency (CP) on writes to inventory, payment, and orders. Eventual consistency (AP) on everything else: reads, analytics, and recommendations. **We would change our mind** only if some "eventual" read started to have real money consequences. For example, if a seller dashboard number ever triggered an automatic payout, that number would need to move to the strong side.

---

## 11.4 Strict no-oversell vs. oversell-and-apologize

**The decision:** do we hold, or "reserve," stock before charging the buyer, so we never promise a unit we do not actually have? Or do we let checkout succeed right away and fix mistakes later, apologizing when we occasionally oversell?

| Point | Strict reservation (our choice) | Oversell-and-apologize |
|---|---|---|
| Buyer trust | High. "Confirmed" always means confirmed | Falls fast. A cancellation email after a "success" screen is one of the worst experiences in online shopping |
| Engineering effort | Higher. Needs a reservation system, a time limit (TTL), and a plan for releasing holds that expire, plus handling for hot items (Ch. 7) | Lower. Checkout can just save the order and reduce a stock counter later, without waiting |
| Behavior under a flash sale | Bounded. 40 units in a flash sale produce exactly 40 confirmed orders. Buyer number 41 is told "sold out" right away | Unbounded risk. The bigger the spike, the worse the oversell. It fails worst exactly when strictness matters most |
| Cost of being wrong | Not a factor, by design | Refunds, goodwill credits, extra support work, and damaged trust with the seller, multiplied by every unit oversold |
| Extra time cost | One reservation round trip, about 80 ms in our latency budget (Ch. 2), before payment | None. You skip the hold step |
| When this is the right choice | Any marketplace with scarce or limited-run goods, where "sold out" is common and real | High-margin digital goods with unlimited supply, or physical goods with huge safety stock, where a rare oversell is cheap to fix |

GlobalMart is built around flash sales and scarce stock. Our worked example is 40 pairs of limited sneakers facing tens of thousands of buyers at once. For this kind of business, oversell-and-apologize is not a small annoyance. It destroys trust, and it happens most during the exact traffic pattern, flash sales, that creates the most oversell risk. It also hurts the seller relationship. A seller who gets 200 orders for 40 units does not just lose money. They lose trust in the whole platform.

Our top priority, correctness, ranks above availability and above speed (see Chapter 1). "No oversell" is a hard requirement, not a nice-to-have.

To be fair, oversell-and-apologize is not always wrong. Picture a grocery delivery service selling a common item, with 10,000 units in a warehouse. Adding an 80 ms reservation check on every single checkout, for a stockout that almost never happens, is a real speed cost for very little benefit. The right answer depends on how scarce the item is, multiplied by how much a mistake would cost the brand.

**Our choice for GlobalMart:** strict reservation, for every item, all the time. Having one simple rule for all items, with no special cases per SKU, is worth more than the small speed saved on items that never run out. **We would change our mind** only for a clearly separate product line, for example bulk goods shipped from a partner warehouse with a six-week lead time and huge safety stock. Even then, we would keep strict reservation on our main flash-sale marketplace path.

---

## 11.5 Inventory concurrency: pessimistic lock vs. optimistic lock vs. Redis counter

**The decision:** once we commit to strict reservation, how do we handle many buyers trying to reduce the stock of the same hot item at the same time? Think of 40 sneakers and 50,000 buyers a second, with most of them chasing that one item.

Three common tools solve this problem.

**Pessimistic locking** means a request locks the row in the database (`SELECT ... FOR UPDATE`) and every other request must wait its turn.

**Optimistic concurrency** means a request reads a version number, then tries to write with "only update if the version is still the same." If someone else already changed it, the write fails and must be retried.

**Redis atomic counters** use Redis, a fast in-memory data store, to do a check-and-decrease operation in one atomic step, using a command like `DECRBY` or a small script.

| Point | Pessimistic lock | Optimistic lock | Redis atomic counter (our choice for hot items) |
|---|---|---|---|
| How it works | Lock the row for the whole transaction; others wait | Read a version, write only if version matches, retry on conflict | A script checks stock and decreases it in one fast, atomic in-memory step |
| Speed under heavy load on one item | Low. Every request queues behind the lock, and the queue explodes during a flash sale | Better than pessimistic locking under light load, but collapses under heavy load. Every retry reads again, tries again, and most attempts still lose | High. Redis processes one operation at a time per key, so there is no lock to wait for, just a fast queue |
| Correctness | Strong; truly one-at-a-time | Strong, if you actually check the version and limit retries. An unlimited retry storm during a flash sale can itself cause an outage | Strong for the counter itself, but the counter is a fast cache in front of the real database, so it needs to be kept in sync |
| Behavior for a normal, low-demand item | Fine. Locks are held briefly | Best. No lock cost at all, and the first attempt almost always succeeds | Overkill. A Redis round trip for something the database could do in one simple write |
| Is it durable on its own? | Yes, it is the real database | Yes, same | No. Redis is the fast path. The database, or a ledger updated soon after, remains the source of truth, and the two are checked against each other regularly (Ch. 7, Ch. 10) |
| What can go wrong | Database connection pool runs out under heavy locking; long waits cause timeouts elsewhere | Retry storm makes load worse exactly when the item is hottest | Redis becomes a hot spot for one key. We fix this by spreading load across shards, not by rethinking the whole approach |

The flash-sale case shows the real problem clearly. With 50,000 buyers a second competing for 40 units, pessimistic locking turns the database's lock manager into the bottleneck. Optimistic locking turns the network and CPU into the bottleneck, because tens of thousands of doomed retries all try again and again, and 39,960 of them are always going to lose. Redis's atomic counter avoids both problems. There is no lock to wait for, and no retry storm to lose. Every request is a fast, in-order operation that either succeeds or returns "not enough stock" in microseconds. Chapter 7 covers the details, like spreading hot keys across shards, warming up counters, and syncing the counter back to the database. The trade-off here in one line: Redis buys the speed a flash sale needs, but in exchange, we must keep it in sync with the real source of truth.

For most ordinary items with low demand, plain optimistic concurrency against the database is simpler to run and fast enough. There is no reason to send every single item through Redis if only a small, detectable set of items are actually hot.

**Our choice for GlobalMart:** optimistic concurrency against the database as the default for normal items. Redis atomic counters as a fast path for hot items, turned on automatically when our system detects heavy demand on one SKU. We use pessimistic locking only briefly and narrowly, for example the final commit step inside one sub-order's own transaction, never to control competition between many different buyers. **We would change our mind** if our hot-item detection missed too many cases, causing surprise contention on "normal" items. In that case, we might route everything through Redis and accept the extra syncing cost everywhere.

---

## 11.6 Authorize-then-capture-later vs. charge-immediately

**The decision:** when the saga reaches the payment step, do we take the buyer's money right away, called an immediate charge? Or do we place a **hold**, called an authorization, and take the money later, called a **capture**, closer to when the order actually ships?

| Point | Charge immediately | Authorize now, capture later (our choice) |
|---|---|---|
| Where the money sits | Moves to GlobalMart right away | Held on the buyer's card, but not moved yet. If a sub-order later fails, we simply cancel the hold, called a **void**, instead of issuing a refund |
| Fits our multi-seller case | Poorly. If Seller C's item goes out of stock after we already charged the card, we must refund money that already settled, which is slower and more visible to the buyer than a void | Well. We authorize the full total once. If one seller's part fails before capture, we either capture a smaller amount or void and re-authorize the new total |
| Time to check for fraud | None. Money moves before any fraud signal has time to be checked | A window, often several days, to check for fraud and confirm the order can actually be fulfilled, before money moves |
| Support across payment methods | Works everywhere cards work | Works well for cards. Some digital wallets and "buy now, pay later" methods do not support a real hold, so we need a fallback |
| Time limit on the hold | None | Holds usually expire around 7 days, depending on the card network. Capture must happen before that, or we must re-authorize |
| What the buyer sees on their statement | A charge appears right away | A "pending" hold appears first, then a real charge at capture time. This pattern is familiar from hotel and car-rental bookings |
| Complexity | Lower. One call to the PSP does the whole job | Higher. Two calls to the PSP, a state to track in between, and logic to re-authorize if the hold expires |

Our saga (Chapter 6) authorizes the full total in step 3, and delays capture to step 5 or to fulfillment time. We do this because a cart from many sellers becomes many sub-orders, and any one of them can fail independently. Authorize-then-capture turns "Seller C sold out after we already took your money" into "Seller C sold out before we took any money for that item." That is a better experience for the buyer, and a cleaner accounting story, since there is no refund trail for money that should never have moved.

It also gives us a window to check for fraud and confirm the order can ship, before the money actually moves. A fraudulent or impossible-to-fulfill order never moves any money at all under this model, compared to charge-immediately, which always creates a refund record and a period where GlobalMart is holding a real buyer's money against an order that might not ship.

The cost is real too. It is two calls to the PSP instead of one. It needs a clock to track when holds expire, with logic to re-authorize if fulfillment takes longer, which happens often with international shipping. And some wallets or "buy now, pay later" methods cannot hold money at all, so we need a charge-and-refund fallback just for those.

**Our choice for GlobalMart:** authorize at place-order time, capture at or just before we hand off to fulfillment, with support for capturing a smaller amount if part of the order fails. For payment methods that cannot support a hold, we fall back to charge-immediately with a clear refund path, as a narrow exception, not our default. **We would change our mind** toward charge-immediately only for single-seller, single-item, always-in-stock categories, like digital goods with no inventory step at all, where the multi-seller partial-failure problem cannot happen in the first place.

---

## 11.7 Synchronous vs. asynchronous saga steps

**The decision:** which steps does the buyer wait for during the place-order request? Which steps happen after we already told the buyer "success"?

A step is **synchronous** if the buyer's app waits for it to finish before getting a response. A step is **asynchronous** if it happens in the background, after the response is already sent.

| Step | Sync or async | Why |
|---|---|---|
| Session revalidation (re-check price and tax) | Sync | The buyer needs today's real price before we commit to anything |
| Inventory reservation | Sync | The buyer must know, before paying, whether the item is actually available |
| Payment authorization | **Sync** (the biggest part of our 2.5-second budget, see Ch. 2) | The buyer needs a clear yes or no on payment before we call checkout "done." This is the one thing we cannot fake with "we'll email you later" |
| Saving the order | Sync | The order must be safely saved before we say success. A "successful" checkout that then loses the order is the exact failure this whole handbook exists to prevent |
| Payment capture (when delayed, see 11.6) | Async | The buyer already has a confirmed order. When we actually take the money is an internal finance detail |
| Handing off to fulfillment (`order.placed` to the warehouse) | Async | This is the warehouse's job. Making the buyer wait for a warehouse acknowledgment adds delay with no benefit to them |
| Notification email or push message | Async | Best-effort, and the buyer does not need it instantly |
| Analytics and recommendation updates | Async | These are the AP-side reads from section 11.3 |

Our rule is simple: anything that changes whether we can tell the buyer "yes, this worked" is synchronous. Anything that happens *because* it already worked is asynchronous. Payment authorization sits right on this line, on the sync side. That is why it takes up most of our latency budget, roughly 300 to 1,500 milliseconds out of a 2.5-second target. We cannot make it async, because doing so risks telling the buyer "success" and then later saying "actually, your card was declined." Capture, on the other hand, is safe to delay, because the authorization step already gave us a firm answer about the buyer's ability to pay. Capture is just bookkeeping on money we were already promised, not new information the buyer needs.

This split is also what makes our 2.5-second target realistic at all. If we made fulfillment handoff, capture, and notification all synchronous too, the buyer would be stuck waiting on warehouse systems, email providers, and internal batch jobs, none of which they have any stake in watching finish.

**Our choice for GlobalMart:** revalidation, reservation, authorization, and saving the order are synchronous inside place-order. Capture, fulfillment handoff, notification, and analytics are asynchronous, triggered by the `order.placed` event. **We would change our mind** and make capture synchronous only for a specific country or payment method where a law requires capture to happen at the exact same time as order confirmation. That is a narrow, rule-driven exception, not a general rethink.

---

## 11.8 Idempotency key: who creates it, and how long do we keep it?

**The decision:** an **idempotency key** is a unique ID sent with a request, so that if the same request arrives twice, the server can recognize it and return the same result instead of doing the action again. Who should create this key, the client app or our server? And how long should we remember it?

| Point | Client-generated key (our choice) | Server-generated key |
|---|---|---|
| What it protects against | The real problem we need to solve: a mobile app that times out and resends a request, or a flaky network causing a double tap | Nothing extra beyond what a client-generated key already covers, and it adds a round trip |
| Extra network round trip needed? | No. The client creates a key, often a random ID, at the moment the buyer taps "Place Order," and sends it with the request | Yes. The client would first have to ask the server for a key, then send the real request with it. That is two round trips instead of one, which hurts our 2.5-second target |
| Works after the app crashes and restarts? | Yes, as long as the client saves the key locally before the first try | Only if the server-given key survives the same problem. No real advantage |
| Risk of two requests using the same key by accident | Low, if we use a proper random ID. The server also double-checks that the key matches the right buyer and the right request details, so a copied or guessed key from someone else is rejected | Same requirement applies either way |
| How much do we trust the client? | The server never blindly trusts the key alone. It always rechecks the buyer, session, and amount against the key | Same requirement either way |

Client-generated keys win because the whole point of this mechanism is protecting against problems the *client* notices, like a timeout or a lost response. Only the client knows "this is a retry of something I already tried." So it makes sense for the client to create the key, and doing so costs zero extra round trips, since the key travels with the very first request. Asking the server for a key first would add pure delay for no extra safety, since the server already checks the key against buyer ID, session ID, and amount regardless of who created it.

**How long do we keep the key?** This is a second, smaller decision. Our design brief sets this at 24 to 48 hours, which needs about 250 to 400 GB of storage at our scale.

| TTL (time to remember the key) | Pros | Cons |
|---|---|---|
| Too short, like 5 minutes | Uses very little storage | A buyer whose app was offline for 20 minutes and then retries looks like a brand new request. This is exactly the double-charge case idempotency is meant to prevent |
| **24 to 48 hours (our choice)** | Covers real cases: an app crash and relaunch the next day, a delayed retry, or a customer support agent resubmitting an order soon after a failure | Uses about 250 to 400 GB of storage at our scale, which is a real but planned-for cost |
| Too long, like 30 days | Covers even rarer cases | Storage grows for little extra benefit. Most real retries happen within hours, not weeks. Anything older than 48 hours is better caught by our reconciliation checks (Ch. 8, Ch. 10) |

**Our choice for GlobalMart:** the client creates the idempotency key. The server checks it against a fingerprint of the real request. We keep the key for 24 to 48 hours, stored in Redis for speed, with a backup in a durable store in case Redis restarts. **We would change our mind** and extend the TTL only if we saw real data showing many legitimate retries arriving after 48 hours, for example due to one specific app's background-retry behavior. Even then, reconciliation would still be a better fix than simply keeping every key forever.

---

## 11.9 Exactly-once through idempotency vs. distributed transactions

**The decision:** how do we make sure the buyer is charged exactly once, and the order is created exactly once, when every network call in our saga can only promise "at least once" delivery? Retries, timeouts, and duplicate messages are normal, not rare accidents.

Here is the honest truth: true exactly-once delivery does not exist in a distributed system with independent parts. You can only choose how you handle the risk of "maybe I already did this." Distributed transactions, like 2PC from section 11.1, try to solve this by making the *delivery* itself atomic. Idempotency keys solve it a different way. They allow delivery to stay at-least-once, but they make repeating the same logical action provably harmless.

| Point | Distributed transaction (2PC) approach | At-least-once delivery plus idempotency (our choice) |
|---|---|---|
| Where "exactly once" is enforced | At the network and coordination layer. The protocol itself guarantees one outcome | At the application layer. Every action that changes data carries a key, and repeating it with the same key is detected and skipped |
| Works with an outside payment provider? | No, as explained in 11.1 | Yes. The call to the PSP itself carries an idempotency key, which most real PSPs support natively |
| How it recovers from failure | The coordinator must resolve every unfinished transaction before anyone can release their locks | Any retry, from any part of the system, hits the same idempotency record and gets back the original result. No locks were ever held across the network, so there is nothing to release |
| What it requires operationally | Every step must speak the same transaction protocol | Every step must pass the key along, and every action that changes data must check it. Simpler rule, but more places where someone must remember to apply it |
| What can still go wrong | The coordinator or a participant crashes mid-protocol, causing everyone to block (see 11.1) | A developer forgets to check the idempotency key on a brand-new code path. This is a coding discipline risk, not a design flaw |

This is really the same answer as 11.1, said again at a different level. We cannot force every partner, especially the PSP, to join a shared transaction protocol. So instead, we require every partner to accept at-least-once delivery and to be safe under retries. This is a weaker, and far more realistic, requirement.

In practice, the client's idempotency key flows from the client, through the orchestrator, into the PSP call, and into the Order Service's database insert, which has a uniqueness rule on the key. Message delivery to Notification Service and Analytics is also deduplicated the same way, using the `order_id` as the key. Every hop only promises at-least-once delivery. Every hop also treats a repeat as a harmless no-op. That combination gives us what the design brief calls "exactly-once effect, not exactly-once delivery," and it works everywhere that 2PC-style exactly-once delivery cannot.

**Our choice for GlobalMart:** at-least-once delivery everywhere, with idempotency keys at every point where data changes: client to orchestrator, orchestrator to PSP, orchestrator to Order database, and message queue to each downstream service. Reconciliation (regularly comparing our records against the PSP and inventory ledger) catches anything that slips through. **We would change our mind** only if some future partner offered a truly transactional interface, and every other partner could too. A card network is very unlikely to ever offer this.

---

## 11.10 Multi-region inventory: one global count vs. region-pinned stock

**The decision:** GlobalMart runs in multiple geographic regions (Chapter 9). For inventory, where a wrong answer means a direct oversell, should we keep one single, strongly-consistent global view of stock? Or should we split stock by region, and accept looser coordination between regions?

| Point | One global count (needs agreement across regions on every write) | Region-pinned stock (our default choice) | Active-active with strong consistency across regions |
|---|---|---|---|
| Write speed | Slow. Every reservation must get agreement from other regions first, adding tens to over 100 milliseconds of pure network and coordination delay | Fast. A reservation is a local write, inside one region | Slow. Same cross-region cost as the global-count option, paid on every single write in every region |
| What happens if one region goes down | The isolated region cannot commit writes without reaching other regions | A region works fully on its own. An outage in one region only affects that region's writes | Depends on the exact rules. A minority region going down may not block the majority, but this is the hardest option to reason about |
| Data residency and legal rules | Often breaks residency rules, since stock data for one country might need to sit physically in another region to reach agreement | Naturally follows residency rules, since a region's stock is written and stored inside that region | Complex. You must design the agreement rules around legal boundaries, not just around speed |
| Risk of the same physical stock being sold from two regions | Not possible, since there is only one global count | A real risk we must design around directly. In practice, a seller's stock for one warehouse belongs to the region that warehouse serves. A sneaker shipped from a US warehouse belongs to the US region's stock. A different, separate EU pool serves EU buyers | Not possible, since there is only one global count, by design |
| Effort to build and run | High. Needs strong cross-region agreement, like a Spanner-style or Paxos-based system, for every reservation | **Low to medium.** Just normal, strong, in-region consistency (Chapter 7's tools), plus a light sync of overall numbers for seller dashboards | Highest. Combines the delay cost of global agreement with the operational complexity of running two active regions at once |

Here is a fact often missed in interviews: physical inventory is already tied to a region. A seller's warehouse is a real place. A unit sitting in a US warehouse cannot ship next-day to an EU buyer, no matter what any database says. So the scary scenario a less experienced candidate imagines, two regions both claiming to sell "the last unit" of one shared global count, usually does not match how a real, fulfillment-aware marketplace actually works.

GlobalMart pins each inventory write to the region that owns that warehouse and listing combination. This gives every region a **single writer** for its own stock. We get full, strong consistency inside each region (Chapter 7's tools apply directly, per region), zero cross-region delay on the hot reservation path, and we naturally follow data residency rules, since a region never writes another region's stock record.

There is one tricky case worth knowing: a large seller who runs one shared warehouse serving buyers worldwide. Even here, we still keep a single writer, just per warehouse instead of per buyer region. One region acts as the owner for that warehouse's stock, no matter where the order came from. Other regions route their reservation requests to that owning region, accepting the cross-region delay only for this smaller group of globally-shared sellers, instead of paying that cost on every single reservation everywhere.

Full active-active with strong agreement across every region is the option we reject outright for the reservation hot path. It pays the worst possible delay on every write, and adds the operational complexity of running active-active, for a level of correctness that region-pinning already achieves more cheaply, given that inventory is inherently physical. Orders and payments are a bit less physically tied down than inventory, but they follow a similar rule: one region owns a given `checkout_group_id` for its whole life, usually the buyer's home region, with async, eventually-consistent copies sent to other regions for reads like order history and support tools. This matches section 11.3's rule of strong writes and eventual reads, just applied across regions instead of across services.

**Our choice for GlobalMart:** region-pinned inventory, with a single writer per stock pool, usually the warehouse's home region. Orders and payments have a single writer per checkout group, usually the buyer's home region. Reads across regions use async replication. **We would change our mind** toward a truly global count only for a product category where "one shared count, sellable from anywhere" becomes an actual requirement, for example digital goods with no warehouse at all. At that point, the physical argument for region-pinning no longer applies.

---

## 11.11 Build vs. buy: payment providers and the saga engine

**The decision:** two "build it ourselves or buy it" questions sit under everything above. Should we build our own payment processing? Should we build our own saga workflow engine?

**Payments: buy, clearly and without doubt.**

| Point | Build our own (become our own payment processor) | Buy (use PSPs like Stripe or Adyen, our choice) |
|---|---|---|
| PCI-DSS compliance (the security standard for handling card data) | Huge. Touching raw card numbers directly puts nearly every system that sees them under strict compliance rules | Small. Since the client tokenizes the card directly with the PSP, card numbers never reach GlobalMart's servers at all |
| Relationships with card networks | Requires direct deals with banks and card networks, years of compliance work, and separate licensing in every country | PSPs already hold these relationships across many countries. Using several PSP adapters (Ch. 8, 9) gives us multi-region coverage without negotiating with every network ourselves |
| Fraud detection tools | Would need to be built from nothing | PSPs already include mature fraud scoring, extra security checks like 3D Secure, and chargeback tools |
| Does this make GlobalMart special to buyers? | Almost never. Buyers do not choose GlobalMart because of how it processes a card | Every engineer-year spent building payment rails is a year not spent on marketplace features that actually make GlobalMart stand out |
| What we do build ourselves | The **Payment Service and PSP Adapter layer**: the routing, idempotency handling, backup PSP failover, and reconciliation logic that sits around the PSPs (Ch. 8) | — |

This decision is not close. Payment processing is a regulated, common, non-differentiating capability with a huge amount of compliance work attached to it. GlobalMart buys the rails, using several PSPs for backup and country coverage (Ch. 9, 10), and builds only the thin, valuable layer on top: idempotency handling, multi-PSP routing and failover, and checking our records against the PSP's own ledger. That layer is genuinely GlobalMart-specific, and it is exactly where our correctness effort belongs.

**The saga engine: build a small, purpose-built orchestrator, not a big general workflow framework. This one is a closer call.**

| Point | Build our own, focused orchestrator (our choice) | Buy or adopt a general workflow framework (like Temporal or Cadence) |
|---|---|---|
| Time to build | Faster, for one fixed, well-understood five-step saga | Slower to set up. It is new infrastructure we must learn to run and trust with the money path |
| Fit for our exact, known flow | Excellent. Every step and every undo action is already fully known (design brief §5). We do not need a tool built for handling any arbitrary long workflow | More than we need for a flow this narrow, though it becomes genuinely valuable if the number of different saga shapes grows, for example different flows per payment method, region, or order type |
| Recovering from a crash mid-flow | We must build this ourselves. The orchestrator saves its state after each step, so a crash resumes from the last saved point, not from scratch (Ch. 6, 10) | Comes built in. Durable, resumable execution is the whole point of these frameworks |
| New dependency risk | One more critical system, but one we fully control and can tune exactly for our workload, 50,000 orders per second at peak, under a 2.5-second budget | Adds a brand-new critical dependency. The framework's own uptime now becomes part of checkout's uptime target, which is real risk on a path with a 99.99% goal |
| Cost over time | Grows if we add more saga shapes, since each new flow means more hand-written state-machine code | Spreads out better across many saga shapes, since the framework already handles the durable-execution plumbing that each new flow would otherwise need to rebuild |

We choose to build, but we openly admit this is the closest call in the whole chapter, and a fair place for a candidate to argue either side well. Our saga is one flow, five steps, fully known in advance, with a known and limited set of undo actions. That is exactly the situation where a general workflow framework's main selling point, handling arbitrary and evolving long-running flows, is paid for but not actually used. Given our latency budget, dominated by 300 to 1,500 milliseconds of payment authorization, and our 99.99% uptime target, adding a big new general-purpose dependency to the busiest and most correctness-sensitive path in the company is a real cost, not a free upgrade. Every dependency we add to place-order becomes a limit on checkout's own uptime. A small, focused orchestrator that GlobalMart fully owns, built exactly for this saga shape, with its own clear checkpoint and recovery logic (Ch. 6, 10), keeps our dependency list as small as correctness allows.

**We would change our mind** on this specific point the moment our saga *shapes* multiply. For example, if GlobalMart adds very different checkout flows, like bulk B2B orders with 30-day payment terms, subscriptions, or region-specific legal flows, we might end up hand-building something that looks like a poorly-made workflow framework, one flow at a time. At that turning point, a real workflow framework's cost, spread across many flows, can become the cheaper option. But for the one, well-defined place-order saga this handbook describes, building it ourselves wins.

---

## Interview Tips

- **Explain every trade-off out loud, do not just state the answer.** Saying "we use a saga" earns partial credit. Saying "we use a saga because the PSP cannot join a prepare-commit protocol, and because a crashed 2PC coordinator leaves everyone stuck, which we cannot allow on a 99.99% uptime target" earns full credit. The option you rejected, and the *exact reason* you rejected it, is what shows real understanding.
- **The CP-here, AP-there contrast is the strongest single point you can make**, especially if you have also designed a search system. Say it plainly: "Search chose availability, because stale data is cheap there. Checkout chooses consistency on the money path, because a double-charge cannot be undone. Same company, same CAP rule, opposite choice, because the cost of a mistake is different." That one sentence shows you understand CAP is a choice made per system, not a company-wide personality.
- **Have a short "when would you change this" line ready for every big decision.** An interviewer pushing on your reservation choice, your authorize-vs-charge choice, or your build-vs-buy choice wants to see that you know the edges of your own answer, not just that you can defend it forever. "We would change this if X happens" is a stronger closing line than repeating your answer.
- **Do not say "always buy for infrastructure, always build for what makes us different."** The saga-engine decision in this chapter is deliberately not simple. Admitting that a general workflow framework becomes the better choice once we have many different saga shapes shows judgment that adapts to changing conditions, which is exactly what senior engineers are graded on.
- **If asked to pick just one trade-off to go deep on, pick 11.1 (saga vs. 2PC) or 11.3 (strong vs. eventual consistency).** These two get asked most often in real interviews, and both have a hard, clear reason behind them: an outside payment provider structurally cannot join a 2PC protocol, and mistakes that cannot be undone deserve strong consistency while reversible ones do not. Neither answer is a soft "it depends."

## Key Takeaways

- **We use a saga, not 2PC**, because the payment provider is an outside partner that structurally cannot join a prepare-commit protocol, and because 2PC's "stuck forever" failure mode does not fit a 99.99% uptime target. This is not a style preference.
- **We use orchestration for the saga, not choreography**, because five ordered, undoable steps need one clear, easy-to-query place that tracks the saga's state. We still use choreography, just after the saga finishes, for the genuinely independent fan-out that follows.
- **We use strong consistency on inventory, payment, and order writes, and eventual consistency everywhere else.** The Search handbook chooses eventual consistency everywhere, because search mistakes are cheap and reversible, while checkout's money-path mistakes are not. Same CAP rule, opposite choice, for a specific, statable reason.
- **We use strict reservation, not oversell-and-apologize**, because GlobalMart is a scarce-inventory, flash-sale marketplace, where broken trust from a false "confirmed" promise costs more than the extra time spent reserving stock.
- **We use Redis atomic counters for hot items, and optimistic locking for the rest.** Pessimistic locks and unlimited optimistic retries both collapse under flash-sale-level competition. Single-threaded atomic counters do not, at the cost of needing a separate sync-and-check process.
- **We authorize now and capture later**, to match the reality that a multi-seller cart can partly fail, and to buy a window of time to check for fraud and confirm fulfillment before money actually moves.
- **We keep steps synchronous when the buyer needs a clear yes-or-no answer** (reserve, authorize, save the order), **and asynchronous when the buyer is just being informed of something that already happened** (capture, fulfillment, notification).
- **We use client-generated idempotency keys, checked by the server against the real request details, kept for 24 to 48 hours.** This is sized to real client retry behavior, not padded "just in case."
- **We get exactly-once effect through at-least-once delivery plus idempotency, not through distributed transactions.** This is the same PSP-cannot-join reason from decision one, said again at the level of individual actions.
- **We use region-pinned inventory, with one writer per stock pool**, because physical inventory is already tied to a region. The scary idea of "oversell across two regions" mostly does not match a real shared stock pool, once you consider where warehouses actually are.
- **We buy payment processing, and build only the thin layer of idempotency, routing, and reconciliation around it. We build our own small saga orchestrator instead of adopting a general workflow framework, for now.** Both of these are judgment calls with a clear trigger for revisiting them later, not permanent rules.
