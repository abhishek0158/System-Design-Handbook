# Order Management System (LLD)

## Problem

Design an order management system (OMS) for an e-commerce backend. The system must:

- Let a customer place an order with one or more line items (each item is a SKU and a quantity).
- Move an order through a clear **lifecycle**: created → confirmed → shipped → delivered, or cancelled.
- Only allow **valid status changes**. For example, a delivered order cannot go back to shipped, and a shipped order cannot be cancelled the normal way.
- Coordinate the steps of placing an order: **reserve stock**, **take payment**, then **confirm** — and if any step fails, **undo** the earlier steps cleanly (no charged customer without stock, no held stock without an order).
- Be **idempotent** on placement: if the same request is sent twice (a network retry), only one order and one charge must happen.
- Notify other parts of the system when an order's status changes (e.g. send an email, update analytics).

## Requirements & Clarifying Questions

1. **Single node or distributed?** Assume one JVM, in-memory storage behind a `Repository` interface. Notes below cover moving to a database or many nodes.
2. **What are the order states?** `CREATED`, `CONFIRMED`, `SHIPPED`, `DELIVERED`, and `CANCELLED`. `DELIVERED` and `CANCELLED` are terminal (no change after them).
3. **What happens during "place order"?** Three steps in order: (a) reserve stock for every line item, (b) charge the payment, (c) confirm. If step (a) or (b) fails, we roll back — release any stock we reserved and refund any charge we made — and mark the order `CANCELLED`. This "do steps, undo on failure" flow is a **saga**.
4. **What does idempotency mean here?** The client sends an **idempotency key** (a unique id it picks for this attempt). If the same key arrives twice, we return the **same** order and do **not** charge again.
5. **Can a shipped order be cancelled?** Not through the normal `cancel` call in this sheet. Cancelling after shipping is a **return/refund** flow — listed as a Follow-up.
6. **How is money stored?** As `long` minor units (cents). `double` cannot store values like `0.1` exactly, and the small errors add up on money.
7. **Do we need thread-safety on one order?** Yes. Two threads must not both move the same order out of `CREATED` (e.g. a `confirm` and a `cancel` arriving together). Status changes must be atomic.

Assumptions: single JVM, in-memory storage, payment and inventory behind interfaces (so they can be faked in tests), whole-unit quantities, money in minor units.

## Design / Approach

```
                         places order (+ idempotency key)
                                       |
                                       v
                        +-----------------------------+
                        |        OrderService         |   <- Facade + Saga orchestrator
                        |  (place / confirm / ship /  |
                        |   deliver / cancel)         |
                        +--+-----------+-----------+--+
        reserve/release    |           | charge/   |   publish status change
        +------------------+           | refund    +-----------------+
        v                              v                             v
+----------------+          +------------------+          +----------------------+
| InventoryService|         |  PaymentService  |          |  OrderEventPublisher |
| (ch.10 reserve) |         |   (interface)    |          |      (Observer)      |
+----------------+          +------------------+          +----------+-----------+
                                                                     |
                                                                     v
                                                          +----------------------+
                                                          |  OrderEventListener  |
                                                          | (email, analytics..) |
                                                          +----------------------+

                Order  ---- holds ---->  List<OrderItem>
                  |
                  | AtomicReference<OrderStatus>
                  | tryTransition(from -> to) checks the allowed-transition table
                  v
        OrderStatus state machine:
        CREATED --confirm--> CONFIRMED --ship--> SHIPPED --deliver--> DELIVERED
           |                     |
           +-------cancel--------+--> CANCELLED   (DELIVERED and CANCELLED are terminal)
```

Patterns and principles used:

- **Facade + Saga orchestrator (`OrderService`)** — one entry point for the whole order lifecycle. `placeOrder` runs the multi-step saga (reserve → pay → confirm) and, on any failure, runs the compensations (release → refund → cancel). Callers never wire these steps by hand.
- **State machine (the `OrderStatus` transition table)** — the order can only follow allowed edges. The rules live in **one** place (a map of allowed transitions), so adding a new state or rule is a one-line change, not a hunt through `if` statements scattered across the code.
- **Observer (`OrderEventPublisher` / `OrderEventListener`)** — status changes are published as events. Sending email or updating analytics is decoupled from the order logic. Adding a new listener needs no change to `OrderService` (**Open/Closed Principle**).
- **Repository (`OrderRepository`)** — the service talks to an interface, not to a `ConcurrentHashMap`. A database-backed repository can replace it later with no change to `OrderService`.
- **Dependency inversion** — `OrderService` depends on the `PaymentService` and `InventoryService` **interfaces**, not concrete classes, so payment and stock can be faked in tests and swapped in production.
- **Single Responsibility** — `Order` owns its own status rules; `OrderService` owns the flow; `PaymentService` owns money; `InventoryService` owns stock.

**Why an atomic status transition, not a plain `setStatus`.** A field like `order.setStatus(CONFIRMED)` is unsafe. Two threads — say a `confirm` and a `cancel` for the same order — could both read `status == CREATED`, both decide their change is valid, and both write. The order would end in a wrong or mixed state, and both the confirm side effects **and** the cancel compensations could run. The fix: hold the status in one `AtomicReference<OrderStatus>` and change it only with **compare-and-swap** against the exact state we expect. Only one thread can win the swap; the other sees it failed and stops. This also makes every transition **idempotent** — calling `confirm` twice moves the state once; the second call sees the state is no longer `CREATED` and does nothing.

## Java Solution

### Value Objects: OrderItem and the money helper

```java
import java.util.*;
import java.util.concurrent.*;
import java.util.concurrent.atomic.AtomicReference;

public final class OrderItem {
    private final String sku;
    private final String warehouseId;
    private final int quantity;
    private final long unitPriceMinorUnits; // price copied at order time, not read live later

    public OrderItem(String sku, String warehouseId, int quantity, long unitPriceMinorUnits) {
        if (quantity <= 0) throw new IllegalArgumentException("quantity must be positive");
        if (unitPriceMinorUnits < 0) throw new IllegalArgumentException("price cannot be negative");
        this.sku = sku;
        this.warehouseId = warehouseId;
        this.quantity = quantity;
        this.unitPriceMinorUnits = unitPriceMinorUnits;
    }
    public String getSku() { return sku; }
    public String getWarehouseId() { return warehouseId; }
    public int getQuantity() { return quantity; }
    public long getUnitPriceMinorUnits() { return unitPriceMinorUnits; }
    public long getLineTotalMinorUnits() { return unitPriceMinorUnits * quantity; }
}
```

> **Note on "price at order time".** We copy the unit price onto the `OrderItem` when the order is
> placed. We do **not** read the live product price later, because prices change — and the customer
> must be charged the price they saw. This is the same rule used in checkout and billing systems.

### The Order and its status state machine

```java
public enum OrderStatus {
    CREATED, CONFIRMED, SHIPPED, DELIVERED, CANCELLED;

    // The allowed edges of the state machine, defined in ONE place.
    private static final Map<OrderStatus, Set<OrderStatus>> ALLOWED = new EnumMap<>(OrderStatus.class);
    static {
        ALLOWED.put(CREATED,   EnumSet.of(CONFIRMED, CANCELLED));
        ALLOWED.put(CONFIRMED, EnumSet.of(SHIPPED, CANCELLED));
        ALLOWED.put(SHIPPED,   EnumSet.of(DELIVERED)); // no normal cancel after shipping
        ALLOWED.put(DELIVERED, EnumSet.noneOf(OrderStatus.class)); // terminal
        ALLOWED.put(CANCELLED, EnumSet.noneOf(OrderStatus.class)); // terminal
    }

    public boolean canMoveTo(OrderStatus target) {
        return ALLOWED.get(this).contains(target);
    }
}

public final class Order {
    private final String id;
    private final String customerId;
    private final List<OrderItem> items;
    private final long totalMinorUnits;
    private final AtomicReference<OrderStatus> status;
    private volatile String paymentTransactionId; // set once payment succeeds; used for refunds

    public Order(String id, String customerId, List<OrderItem> items) {
        if (items == null || items.isEmpty()) throw new IllegalArgumentException("order needs items");
        this.id = id;
        this.customerId = customerId;
        this.items = List.copyOf(items); // defensive copy; the list cannot be changed after creation
        this.totalMinorUnits = items.stream().mapToLong(OrderItem::getLineTotalMinorUnits).sum();
        this.status = new AtomicReference<>(OrderStatus.CREATED);
    }

    /**
     * Atomically moves the order from 'expected' to 'target', but only if:
     *  - the order is currently in 'expected', AND
     *  - the state machine allows expected -> target.
     * Returns false if another thread already changed the status, or the move is not allowed.
     * This is what makes transitions both thread-safe and idempotent.
     */
    public boolean tryTransition(OrderStatus expected, OrderStatus target) {
        if (!expected.canMoveTo(target)) return false;
        return status.compareAndSet(expected, target);
    }

    public String getId() { return id; }
    public String getCustomerId() { return customerId; }
    public List<OrderItem> getItems() { return items; }
    public long getTotalMinorUnits() { return totalMinorUnits; }
    public OrderStatus getStatus() { return status.get(); }
    public String getPaymentTransactionId() { return paymentTransactionId; }
    public void setPaymentTransactionId(String txnId) { this.paymentTransactionId = txnId; }
}
```

### Repository (Repository Pattern)

```java
public interface OrderRepository {
    void save(Order order);
    Optional<Order> findById(String orderId);
}

public class InMemoryOrderRepository implements OrderRepository {
    private final ConcurrentHashMap<String, Order> orders = new ConcurrentHashMap<>();

    @Override public void save(Order order) { orders.put(order.getId(), order); }
    @Override public Optional<Order> findById(String orderId) {
        return Optional.ofNullable(orders.get(orderId));
    }
}
```

### Payment and Inventory dependencies (interfaces)

`OrderService` depends on these interfaces, not concrete classes, so they can be faked in tests.
`InventoryService` is the one from the Inventory Management sheet (chapter 10); only the methods we
use are shown here.

```java
public final class PaymentResult {
    private final boolean success;
    private final String transactionId; // present only on success
    private PaymentResult(boolean success, String transactionId) {
        this.success = success; this.transactionId = transactionId;
    }
    public static PaymentResult ok(String txnId) { return new PaymentResult(true, txnId); }
    public static PaymentResult failed() { return new PaymentResult(false, null); }
    public boolean isSuccess() { return success; }
    public String getTransactionId() { return transactionId; }
}

public interface PaymentService {
    // idempotencyKey lets the payment provider dedupe a retried charge on its side too.
    PaymentResult charge(String customerId, long amountMinorUnits, String idempotencyKey);
    void refund(String transactionId);
}

// From chapter 10 (Inventory Management). Shown here as the slice OrderService needs.
public interface InventoryService {
    Optional<Reservation> reserveStock(String sku, String warehouseId, int qty, String orderId);
    void confirmReservation(String reservationId);
    void releaseReservation(String reservationId);
}
```

### Order status events (Observer Pattern)

```java
public interface OrderEventListener {
    void onStatusChanged(String orderId, OrderStatus from, OrderStatus to);
}

public class OrderEventPublisher {
    private final List<OrderEventListener> listeners = new CopyOnWriteArrayList<>();

    public void subscribe(OrderEventListener listener) { listeners.add(listener); }

    public void publish(String orderId, OrderStatus from, OrderStatus to) {
        for (OrderEventListener listener : listeners) {
            listener.onStatusChanged(orderId, from, to);
        }
    }
}
```

### Order Service (Facade + Saga orchestrator)

```java
public class OrderService {
    private final OrderRepository orderRepository;
    private final InventoryService inventoryService;
    private final PaymentService paymentService;
    private final OrderEventPublisher eventPublisher;

    // Idempotency: one client-chosen key maps to exactly one created order.
    private final ConcurrentHashMap<String, Order> idempotencyStore = new ConcurrentHashMap<>();

    public OrderService(OrderRepository orderRepository, InventoryService inventoryService,
                        PaymentService paymentService, OrderEventPublisher eventPublisher) {
        this.orderRepository = orderRepository;
        this.inventoryService = inventoryService;
        this.paymentService = paymentService;
        this.eventPublisher = eventPublisher;
    }

    /**
     * Places an order as a saga: reserve stock -> charge payment -> confirm.
     * On any failure, undoes the earlier steps and cancels the order.
     * Idempotent: the same idempotencyKey always returns the same order, charged once.
     */
    public Order placeOrder(String customerId, List<OrderItem> items, String idempotencyKey) {
        // If this key was already handled, return the same order. computeIfAbsent runs the
        // body at most once per key. (Trade-off discussed in the thread-safety notes.)
        return idempotencyStore.computeIfAbsent(idempotencyKey,
                key -> runPlaceOrderSaga(customerId, items));
    }

    private Order runPlaceOrderSaga(String customerId, List<OrderItem> items) {
        Order order = new Order(UUID.randomUUID().toString(), customerId, items);
        orderRepository.save(order);

        // Step 1: reserve stock for every line item. Remember reservations so we can undo them.
        List<String> reservationIds = new ArrayList<>();
        for (OrderItem item : items) {
            Optional<Reservation> reservation = inventoryService.reserveStock(
                    item.getSku(), item.getWarehouseId(), item.getQuantity(), order.getId());
            if (reservation.isEmpty()) {
                releaseAll(reservationIds);      // compensate: give back what we already reserved
                cancel(order);                   // mark the order cancelled
                return order;                    // out-of-stock: order ends CANCELLED
            }
            reservationIds.add(reservation.get().getId());
        }

        // Step 2: charge the payment.
        PaymentResult payment = paymentService.charge(
                customerId, order.getTotalMinorUnits(), order.getId());
        if (!payment.isSuccess()) {
            releaseAll(reservationIds);          // compensate: release the stock we held
            cancel(order);                       // payment failed: order ends CANCELLED
            return order;
        }
        order.setPaymentTransactionId(payment.getTransactionId());

        // Step 3: confirm. Convert reservations to permanent deductions and move the order forward.
        for (String reservationId : reservationIds) {
            inventoryService.confirmReservation(reservationId);
        }
        transition(order, OrderStatus.CREATED, OrderStatus.CONFIRMED);
        return order;
    }

    public void shipOrder(String orderId) {
        Order order = require(orderId);
        transition(order, OrderStatus.CONFIRMED, OrderStatus.SHIPPED);
    }

    public void deliverOrder(String orderId) {
        Order order = require(orderId);
        transition(order, OrderStatus.SHIPPED, OrderStatus.DELIVERED);
    }

    /** Customer or system cancels an order that has not shipped yet. Refunds and releases stock. */
    public void cancelOrder(String orderId) {
        Order order = require(orderId);
        cancel(order);
    }

    // --- helpers ---

    private void cancel(Order order) {
        OrderStatus current = order.getStatus();
        // Only CREATED or CONFIRMED can be cancelled (the state machine enforces this too).
        if (!transition(order, current, OrderStatus.CANCELLED)) {
            return; // already terminal or someone else changed it first; nothing to do
        }
        // Compensations run only for the thread that actually won the transition.
        if (order.getPaymentTransactionId() != null) {
            paymentService.refund(order.getPaymentTransactionId());
        }
        // Reservations for a confirmed order are already deducted; a full system would also
        // add the stock back here. For a created-but-not-confirmed order they are released
        // inside the saga. (See Follow-ups for restocking on cancel.)
    }

    /** Runs the atomic status change and publishes the event only if the change actually happened. */
    private boolean transition(Order order, OrderStatus from, OrderStatus to) {
        boolean moved = order.tryTransition(from, to);
        if (moved) {
            orderRepository.save(order);
            eventPublisher.publish(order.getId(), from, to);
        }
        return moved;
    }

    private void releaseAll(List<String> reservationIds) {
        for (String reservationId : reservationIds) {
            inventoryService.releaseReservation(reservationId);
        }
    }

    private Order require(String orderId) {
        return orderRepository.findById(orderId)
                .orElseThrow(() -> new IllegalArgumentException("Unknown order: " + orderId));
    }

    public Optional<Order> getOrder(String orderId) { return orderRepository.findById(orderId); }
}
```

## How It Works

**Placing an order (the happy path).** A customer calls `placeOrder("cust-1", items, "key-abc")`.
The service creates an `Order` in status `CREATED`. It reserves stock for each line item through
`InventoryService` (the same atomic reserve from chapter 10, so no overselling). It charges the
payment. All three succeed, so it confirms each reservation (stock is now permanently deducted) and
moves the order `CREATED → CONFIRMED`. An event is published, and any listener (email, analytics)
reacts.

**Out of stock or payment failure (the saga rollback).** Say the second line item cannot be
reserved. The service **releases** the reservation it already made for the first item, moves the
order `CREATED → CANCELLED`, and returns. No stock stays held, and no payment was taken. If instead
stock was fine but the **charge fails**, the service releases all reservations and cancels the
order. If the charge had already succeeded before a later failure, `cancel` calls
`paymentService.refund` using the saved transaction id. The rule is simple: **every step that can
succeed has a matching undo, and we run the undos in reverse on failure.**

**Idempotent placement.** Two identical requests with the same `key-abc` arrive at once (a client
retry). `idempotencyStore.computeIfAbsent(key, ...)` runs the saga **once** for that key; the second
call gets the **same** `Order` object back and never charges again. Without this, a retry could
create two orders and charge the customer twice.

**Safe status changes.** Suppose a `confirm` and a `cancel` reach the same `CREATED` order at the
same instant. Both read `status == CREATED`. Both try `compareAndSet(CREATED, ...)`. The JVM lets
only one succeed. If `confirm` wins, the order becomes `CONFIRMED` and the `cancel` thread's
`compareAndSet(CREATED, CANCELLED)` fails, so it does nothing — the compensations (refund) never
run by mistake. The order can never end in a torn state, and no side effect runs twice.

**Invalid moves are blocked.** A call to ship a `CREATED` order fails, because the transition table
only allows `CONFIRMED → SHIPPED`. A delivered order cannot move anywhere, because its allowed set
is empty. The rules are enforced in one place.

## How to Extend (Follow-ups)

- **Returns / refunds after shipping.** Add states `RETURN_REQUESTED` and `RETURNED`, allowed only
  from `DELIVERED`, with a compensation that refunds and adds stock back. This keeps the "cancel
  before ship, return after ship" split clean.
- **Restock on cancel of a confirmed order.** When cancelling after confirm, call
  `inventoryService.addStock(...)` for each item so the deducted units return to available stock.
- **Order state as the State pattern (behaviour, not just data).** If each state needs very
  different behaviour (e.g. what "cancel" means differs a lot per state), replace the enum table
  with an `OrderState` interface and one class per state. The table is simpler when the difference
  is only "which moves are allowed"; the State pattern wins when each state carries a lot of logic.
- **Pricing, discounts, and tax.** Add a `PricingStrategy` (Strategy pattern) that computes the
  total from items + coupons + tax rules, instead of a plain sum. This keeps `Order` free of pricing
  policy.
- **Persistence with a real database.** Implement `OrderRepository` on SQL. Guard the status column
  with an optimistic-lock `version` (`UPDATE ... SET status=?, version=version+1 WHERE id=? AND
  version=?`) to keep the same "change only if unchanged" guarantee across processes.
- **Idempotency across many nodes.** The in-memory `idempotencyStore` only works in one JVM. Use a
  unique constraint on the idempotency key in the database, or a distributed store (e.g. Redis
  `SET key value NX`), so a retry hitting another node is still deduped.
- **Reliable events (outbox).** In-process `OrderEventPublisher` can lose events if the process
  crashes. Write the event to an **outbox** table in the same transaction as the order change, then
  a separate publisher sends it — so an order change and its event are never out of sync.
- **Timeouts and the real saga engine.** For long steps and partial failures across services, move
  the saga to an orchestrator (e.g. a workflow engine) with retries, timeouts, and durable state,
  rather than an in-memory method.

## Complexity & Thread-Safety Notes

| Operation | Time | Notes |
|---|---|---|
| `OrderRepository.save` / `findById` | O(1) amortized | `ConcurrentHashMap` put/get |
| `Order.tryTransition` | O(1) | one map lookup in the transition table + one `compareAndSet` |
| `placeOrder` | O(n) | n = number of line items (one reserve + one confirm each) |
| `idempotencyStore.computeIfAbsent` | O(1) amortized | runs the saga body at most once per key |
| `OrderEventPublisher.publish` | O(k) | k = number of subscribed listeners |

**Thread-safety:**

- **Status changes are atomic.** `Order` holds its status in an `AtomicReference<OrderStatus>` and
  changes it only with `compareAndSet(expected, target)`. Two threads acting on the same order can
  never both win; the loser sees the swap fail and stops. This is what stops a confirm and a cancel
  from both taking effect.
- **Transitions are idempotent.** Because the swap only succeeds from the exact expected state,
  calling `confirmOrder` (or any transition) twice — a retried request — moves the state once. The
  second call finds the state already changed and does nothing, so no side effect runs twice.
- **Placement is idempotent.** `computeIfAbsent` on the idempotency key runs the whole saga at most
  once per key. **Trade-off to state in the interview:** `computeIfAbsent` holds a lock on that key's
  bucket while the saga runs, which is fine for short work but not ideal for slow network calls
  (payment). In production you would reserve the key first (a quick `putIfAbsent` of a placeholder or
  a DB unique constraint), then do the slow work — so one key never blocks unrelated keys.
- **Compensations run once, on the winning thread only.** In `cancel`, the refund and restock run
  **after** the `compareAndSet` to `CANCELLED` succeeds, so only the single thread that actually
  cancelled the order runs the undo. A losing thread never double-refunds.
- **Listeners use `CopyOnWriteArrayList`.** Listeners are registered rarely (at startup) and read on
  every status change — cheap reads, rare expensive writes, which fits copy-on-write.
- **The item list is immutable.** `Order` stores `List.copyOf(items)`, so no other thread can change
  an order's items after it is created; the total is computed once and never drifts.

## Interview Tips & Common Mistakes

- **Never use a plain `setStatus`.** "Read status, decide, write status" as separate steps is a
  race. Make the status change one atomic step with `compareAndSet(expected, target)`, and keep the
  allowed moves in a single transition table so the rules are not scattered across `if` blocks.
- **Say the word "saga" and pair every step with its undo.** Placing an order is reserve → pay →
  confirm. Interviewers want to hear the compensations: if pay fails, release stock; if a later step
  fails after paying, refund. "Do steps forward, undo in reverse on failure" is the whole idea.
- **Do not forget idempotency on placement.** A retried "place order" must not create two orders or
  charge twice. Use a client-supplied idempotency key and run the work at most once per key. This is
  one of the most common misses in this problem.
- **Confirm/cancel must be idempotent too.** Networks retry. Guard each transition with a
  compare-and-swap on the expected state so a repeated confirm or cancel takes effect only once.
- **Depend on interfaces for payment and inventory.** Hard-coding a concrete `StripePayment` inside
  `OrderService` makes it untestable and rigid. Depend on `PaymentService` / `InventoryService`
  interfaces so they can be faked in tests and swapped in production.
- **Copy the price onto the order.** Charge the price the customer saw. Reading the live product
  price at charge time is a real bug when prices change between viewing and paying.
- **Keep money in integer minor units.** Using `double` for money gives rounding errors that add up.
  Use `long` cents (or `BigDecimal` if you must divide).
```
