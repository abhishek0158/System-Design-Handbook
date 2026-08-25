# Inventory Management (LLD)

## Problem

Design an inventory management system for an e-commerce backend. The system must:

- Model products (SKU, name, price) and track stock quantity per product.
- Support one or more warehouses. The same product can sit in many warehouses, each with its own quantity.
- Add stock (goods received), remove/deduct stock (write-off, damage), and reserve stock for an order, so the same unit is never sold twice.
- Prevent overselling when many orders arrive for the same product at the same time.
- Raise a low-stock alert when stock drops below a threshold, so a reorder can be triggered.

## Requirements & Clarifying Questions

1. **Single node or distributed?** Assume one JVM, in-memory storage, behind a `Repository` interface. Notes below cover moving to a database or multiple nodes.
2. **What does "reserve" mean?** An order first reserves stock (holds it), then either confirms it (order ships, stock is permanently deducted) or releases it (order cancelled, stock returns to available). This two-step flow is why `onHand` and `reserved` are separate numbers.
3. **What is "available" stock?** `available = onHand - reserved`. Customers can only buy what is available, never raw `onHand`.
4. **Can one order pull from more than one warehouse?** No, for this sheet — one reservation is scoped to one SKU and one warehouse, chosen by the caller. Splitting an order across warehouses is a Follow-up.
5. **How exact must the overselling fix be?** Fully exact — with 10 units on hand, no combination of concurrent orders may reserve more than 10. This is the main constraint of this sheet.
6. **What triggers a reorder?** A low-stock alert, fired once available stock crosses below a per-product threshold. We must not flood the supplier with a duplicate order every time stock ticks down further while a reorder is already in flight.
7. **Is stock a whole number?** Yes, `int` quantities. Price uses `long` minor units (cents), since `double` cannot represent decimals like `0.1` exactly and small errors would add up.

Assumptions: single JVM, in-memory storage, one warehouse per reservation, whole-unit quantities, lock-free concurrency control on stock counts.

## Design / Approach

```
+---------+      +-----------+        +------------------------+
| Product |      | Warehouse |        |    InventoryService     |
+---------+      +-----------+        |        (Facade)         |
                                       +-----------+--------------+
                                                   |
                        reserve / confirm / release | addStock / removeStock
                +----------------------------------+----------------------------+
                v                                                               v
        +---------------+    findOrCreate / find      +----------------------------+
        |   StockItem   |<---------------------------- |   InventoryRepository     |
        | (per SKU +    |                               |      (interface)          |
        |  warehouse)   |                               +-------------+--------------+
        +-------+-------+                                             |
                | AtomicReference<StockSnapshot>                      | implemented by
                | CAS loop: reserve() / removeStock()                 v
                | available = onHand - reserved              InMemoryInventoryRepository
                v
        isLowStock() ?
                |
                v
        +---------------------+   Observer pattern   +-----------------------+
        | StockAlertPublisher  | -------------------> |  StockAlertListener   |
        +----------------------+   notifies subscribers +---------+-----------+
                                                                    ^
                                                            +-------+--------+
                                                            | ReorderListener |
                                                            +-------+--------+
                                                                    |
                                                                    v
                                                             SupplierService
                                                                    |
                                                                    v
                                                             PurchaseOrder
```

Patterns and principles used:

- **Repository (`InventoryRepository`)** — the rest of the system talks to an interface, not to `ConcurrentHashMap` directly. Swapping in a database-backed repository later needs no change to `InventoryService`.
- **Facade (`InventoryService`)** — the single entry point for add, remove, reserve, confirm, release. Callers never touch `StockItem` or the reservation map directly.
- **Observer (`StockAlertPublisher` / `StockAlertListener`)** — low-stock detection is decoupled from what happens next. `InventoryService` only asks "is this item low?" and publishes an event; it does not know or care that a reorder happens. Adding a new listener (e.g. an email alert) needs no change to `InventoryService` (**Open/Closed Principle**).
- **Single Responsibility** — `StockItem` only owns one product's stock count and its atomic update rules. `ReorderListener` only decides when to call the supplier. `SupplierService` only manages purchase orders.
- **Immutable value objects + compare-and-swap (`StockSnapshot`)** — this is the key design decision, explained next.

**Why `AtomicReference<StockSnapshot>`, not two separate counters.** A naive design keeps `onHand` and `reserved` as two independent `AtomicInteger` fields. That is unsafe: a thread could read `onHand`, then another thread changes `reserved`, then the first thread decides based on stale data. The check "is available >= requested quantity" must see `onHand` and `reserved` **together, as one atomic unit**, or two threads can both see "enough is available" for the same last unit and both succeed — an oversell.

The fix: bundle `onHand` and `reserved` into one immutable object, `StockSnapshot`, held behind a single `AtomicReference`. Every state change follows the same **compare-and-swap (CAS) loop**:

1. Read the current snapshot.
2. Compute a new snapshot from it, validating the business rule (e.g. enough available stock).
3. Try to atomically swap the reference from the old snapshot to the new one with `compareAndSet`.
4. If another thread changed the reference first, the swap fails; go back to step 1 and retry.

This is lock-free: no thread ever blocks. Under contention a thread may retry a few times, but it always makes progress, and the check-then-act is atomic because both fields move together in one object swap — the same idea used in classic checkout/stock-decrement problems: "check and decrement" must be one indivisible step, not two.

## Java Solution

### Product, Warehouse, and the Stock Key

```java
import java.util.*;
import java.util.concurrent.*;
import java.util.concurrent.atomic.AtomicReference;

public final class Product {
    private final String sku;
    private final String name;
    private final long priceMinorUnits; // e.g. cents
    private final int lowStockThreshold;

    public Product(String sku, String name, long priceMinorUnits, int lowStockThreshold) {
        this.sku = sku;
        this.name = name;
        this.priceMinorUnits = priceMinorUnits;
        this.lowStockThreshold = lowStockThreshold;
    }
    public String getSku() { return sku; }
    public String getName() { return name; }
    public long getPriceMinorUnits() { return priceMinorUnits; }
    public int getLowStockThreshold() { return lowStockThreshold; }
}

public final class Warehouse {
    private final String id;
    private final String location;

    public Warehouse(String id, String location) { this.id = id; this.location = location; }
    public String getId() { return id; }
    public String getLocation() { return location; }
}

// Identifies one product's stock inside one warehouse. Used as a map key.
public final class StockKey {
    private final String sku;
    private final String warehouseId;

    public StockKey(String sku, String warehouseId) {
        this.sku = sku;
        this.warehouseId = warehouseId;
    }
    public String getSku() { return sku; }
    public String getWarehouseId() { return warehouseId; }

    @Override
    public boolean equals(Object o) {
        if (!(o instanceof StockKey)) return false;
        StockKey k = (StockKey) o;
        return sku.equals(k.sku) && warehouseId.equals(k.warehouseId);
    }
    @Override
    public int hashCode() { return Objects.hash(sku, warehouseId); }
}
```

### Stock Snapshot and Stock Item — the concurrency core

`StockSnapshot` is an immutable point-in-time view of one product's stock in one warehouse. `StockItem` owns the `AtomicReference` and every CAS loop that changes it.

```java
public final class StockSnapshot {
    private final int onHand;
    private final int reserved;

    public StockSnapshot(int onHand, int reserved) {
        this.onHand = onHand;
        this.reserved = reserved;
    }
    public int getOnHand() { return onHand; }
    public int getReserved() { return reserved; }
    public int getAvailable() { return onHand - reserved; }
}

public class StockItem {
    private final String sku;
    private final String warehouseId;
    private final int lowStockThreshold;
    private final AtomicReference<StockSnapshot> state;

    public StockItem(String sku, String warehouseId, int lowStockThreshold) {
        this.sku = sku;
        this.warehouseId = warehouseId;
        this.lowStockThreshold = lowStockThreshold;
        this.state = new AtomicReference<>(new StockSnapshot(0, 0));
    }

    /** Goods received. No condition to check, so a single updateAndGet is enough. */
    public void addStock(int qty) {
        if (qty <= 0) throw new IllegalArgumentException("qty must be positive");
        state.updateAndGet(s -> new StockSnapshot(s.getOnHand() + qty, s.getReserved()));
    }

    /**
     * Direct deduction, bypassing reservation (e.g. damaged goods write-off).
     * Only free (unreserved) stock can be removed this way.
     */
    public boolean removeStock(int qty) {
        if (qty <= 0) throw new IllegalArgumentException("qty must be positive");
        while (true) {
            StockSnapshot current = state.get();
            if (current.getAvailable() < qty) return false;
            StockSnapshot updated = new StockSnapshot(current.getOnHand() - qty, current.getReserved());
            if (state.compareAndSet(current, updated)) return true;
            // another thread updated first; current is stale, loop and retry with a fresh read
        }
    }

    /**
     * THE KEY OPERATION. Atomically checks available stock and reserves it in one step.
     * Safe when many threads call this at the same time for the same SKU + warehouse.
     */
    public boolean reserve(int qty) {
        if (qty <= 0) throw new IllegalArgumentException("qty must be positive");
        while (true) {
            StockSnapshot current = state.get();
            if (current.getAvailable() < qty) {
                return false; // not enough available; caller must reject the order, no retry needed
            }
            StockSnapshot updated = new StockSnapshot(current.getOnHand(), current.getReserved() + qty);
            if (state.compareAndSet(current, updated)) {
                return true; // won the race: this thread's reservation is now recorded
            }
            // lost the race to another thread's update; retry the check against the latest snapshot
        }
    }

    /** Order shipped: the reservation becomes a permanent deduction. */
    public void confirmReservation(int qty) {
        state.updateAndGet(s -> new StockSnapshot(s.getOnHand() - qty, s.getReserved() - qty));
    }

    /** Order cancelled: give the reserved units back to available stock. */
    public void releaseReservation(int qty) {
        state.updateAndGet(s -> new StockSnapshot(s.getOnHand(), s.getReserved() - qty));
    }

    public StockSnapshot getSnapshot() { return state.get(); }
    public boolean isLowStock() { return getSnapshot().getAvailable() <= lowStockThreshold; }
    public String getSku() { return sku; }
    public String getWarehouseId() { return warehouseId; }
    public int getLowStockThreshold() { return lowStockThreshold; }
}
```

### Repository (Repository Pattern)

```java
public interface InventoryRepository {
    StockItem findOrCreate(String sku, String warehouseId, int lowStockThreshold);
    Optional<StockItem> find(String sku, String warehouseId);
    List<StockItem> findBySku(String sku);
}

public class InMemoryInventoryRepository implements InventoryRepository {
    private final ConcurrentHashMap<StockKey, StockItem> stock = new ConcurrentHashMap<>();

    @Override
    public StockItem findOrCreate(String sku, String warehouseId, int lowStockThreshold) {
        // computeIfAbsent is atomic per key: two threads creating the same StockItem
        // for the first time can never end up with two different instances.
        return stock.computeIfAbsent(new StockKey(sku, warehouseId),
                k -> new StockItem(sku, warehouseId, lowStockThreshold));
    }

    @Override
    public Optional<StockItem> find(String sku, String warehouseId) {
        return Optional.ofNullable(stock.get(new StockKey(sku, warehouseId)));
    }

    @Override
    public List<StockItem> findBySku(String sku) {
        List<StockItem> result = new ArrayList<>();
        for (StockItem item : stock.values()) {
            if (item.getSku().equals(sku)) result.add(item);
        }
        return result;
    }
}
```

### Reservation

A reservation is a receipt for a `reserve()` call. Its status uses the same compare-and-swap idea, so `confirm` and `release` are each safe to call exactly once, even if called twice by mistake (e.g. a retried request).

```java
public enum ReservationStatus { ACTIVE, CONFIRMED, RELEASED }

public final class Reservation {
    private final String id;
    private final String sku;
    private final String warehouseId;
    private final int quantity;
    private final String orderId;
    private final AtomicReference<ReservationStatus> status;

    public Reservation(String id, String sku, String warehouseId, int quantity, String orderId) {
        this.id = id;
        this.sku = sku;
        this.warehouseId = warehouseId;
        this.quantity = quantity;
        this.orderId = orderId;
        this.status = new AtomicReference<>(ReservationStatus.ACTIVE);
    }

    /** Moves status from 'from' to 'to' only if it is still 'from'. Returns false if already moved. */
    public boolean tryTransition(ReservationStatus from, ReservationStatus to) {
        return status.compareAndSet(from, to);
    }

    public String getId() { return id; }
    public String getSku() { return sku; }
    public String getWarehouseId() { return warehouseId; }
    public int getQuantity() { return quantity; }
    public String getOrderId() { return orderId; }
    public ReservationStatus getStatus() { return status.get(); }
}
```

### Low-Stock Alerts (Observer Pattern)

```java
public interface StockAlertListener {
    void onLowStock(String sku, String warehouseId, int available, int threshold);
}

public class StockAlertPublisher {
    private final List<StockAlertListener> listeners = new CopyOnWriteArrayList<>();

    public void subscribe(StockAlertListener listener) { listeners.add(listener); }

    public void publishIfLow(StockItem item) {
        if (!item.isLowStock()) return;
        int available = item.getSnapshot().getAvailable();
        for (StockAlertListener listener : listeners) {
            listener.onLowStock(item.getSku(), item.getWarehouseId(), available, item.getLowStockThreshold());
        }
    }
}
```

### Reorder Listener and Supplier (a simple reorder flow)

```java
public class ReorderListener implements StockAlertListener {
    private final SupplierService supplierService;
    // Tracks SKUs with an order already placed, so a further stock dip does not
    // trigger a second, duplicate purchase order while the first is in flight.
    private final Set<StockKey> pendingReorders = ConcurrentHashMap.newKeySet();

    public ReorderListener(SupplierService supplierService) {
        this.supplierService = supplierService;
    }

    @Override
    public void onLowStock(String sku, String warehouseId, int available, int threshold) {
        StockKey key = new StockKey(sku, warehouseId);
        if (!pendingReorders.add(key)) return; // a reorder is already pending for this SKU + warehouse
        int reorderQty = Math.max(threshold * 3, 10); // simple fixed-multiple reorder policy
        supplierService.placeOrder(sku, warehouseId, reorderQty);
    }

    /** Called once the supplier's order is received, so future low-stock dips can reorder again. */
    public void clearPending(String sku, String warehouseId) {
        pendingReorders.remove(new StockKey(sku, warehouseId));
    }
}

public final class PurchaseOrder {
    private final String id;
    private final String sku;
    private final String warehouseId;
    private final int quantity;
    private volatile boolean received;

    public PurchaseOrder(String id, String sku, String warehouseId, int quantity) {
        this.id = id;
        this.sku = sku;
        this.warehouseId = warehouseId;
        this.quantity = quantity;
    }
    public String getId() { return id; }
    public String getSku() { return sku; }
    public String getWarehouseId() { return warehouseId; }
    public int getQuantity() { return quantity; }
    public boolean isReceived() { return received; }
    public void markReceived() { received = true; }
}

public class SupplierService {
    private final InventoryService inventoryService;
    private final ReorderListener reorderListener;
    private final Map<String, PurchaseOrder> orders = new ConcurrentHashMap<>();

    public SupplierService(InventoryService inventoryService, ReorderListener reorderListener) {
        this.inventoryService = inventoryService;
        this.reorderListener = reorderListener;
    }

    public String placeOrder(String sku, String warehouseId, int quantity) {
        String id = UUID.randomUUID().toString();
        orders.put(id, new PurchaseOrder(id, sku, warehouseId, quantity));
        return id;
    }

    /** Simulates the supplier's delivery arriving; feeds the stock back into inventory. */
    public void receiveOrder(String orderId) {
        PurchaseOrder order = orders.get(orderId);
        if (order == null || order.isReceived()) return;
        order.markReceived();
        inventoryService.addStock(order.getSku(), order.getWarehouseId(), order.getQuantity());
        reorderListener.clearPending(order.getSku(), order.getWarehouseId());
    }
}
```

### Inventory Service (Facade)

```java
public class InventoryService {
    private static final int DEFAULT_LOW_STOCK_THRESHOLD = 5; // in a real system, read from the Product catalog

    private final InventoryRepository repository;
    private final StockAlertPublisher alertPublisher;
    private final Map<String, Reservation> reservations = new ConcurrentHashMap<>();

    public InventoryService(InventoryRepository repository, StockAlertPublisher alertPublisher) {
        this.repository = repository;
        this.alertPublisher = alertPublisher;
    }

    public void addStock(String sku, String warehouseId, int qty) {
        StockItem item = repository.findOrCreate(sku, warehouseId, DEFAULT_LOW_STOCK_THRESHOLD);
        item.addStock(qty);
    }

    public boolean removeStock(String sku, String warehouseId, int qty) {
        Optional<StockItem> found = repository.find(sku, warehouseId);
        if (!found.isPresent()) return false;
        boolean removed = found.get().removeStock(qty);
        if (removed) alertPublisher.publishIfLow(found.get());
        return removed;
    }

    /** Reserve stock for an order. Returns empty if not enough is available right now. */
    public Optional<Reservation> reserveStock(String sku, String warehouseId, int qty, String orderId) {
        StockItem item = repository.findOrCreate(sku, warehouseId, DEFAULT_LOW_STOCK_THRESHOLD);
        if (!item.reserve(qty)) {
            return Optional.empty(); // atomic check-and-reserve failed: not enough available stock
        }
        Reservation reservation = new Reservation(UUID.randomUUID().toString(), sku, warehouseId, qty, orderId);
        reservations.put(reservation.getId(), reservation);
        alertPublisher.publishIfLow(item);
        return Optional.of(reservation);
    }

    /** Order shipped: convert the reservation into a permanent stock deduction. */
    public void confirmReservation(String reservationId) {
        Reservation reservation = requireReservation(reservationId);
        if (!reservation.tryTransition(ReservationStatus.ACTIVE, ReservationStatus.CONFIRMED)) {
            throw new IllegalStateException("Reservation " + reservationId + " is not active");
        }
        StockItem item = repository.find(reservation.getSku(), reservation.getWarehouseId())
                .orElseThrow(() -> new IllegalStateException("Missing stock item for reservation"));
        item.confirmReservation(reservation.getQuantity());
    }

    /** Order cancelled: give the reserved units back to available stock. */
    public void releaseReservation(String reservationId) {
        Reservation reservation = requireReservation(reservationId);
        if (!reservation.tryTransition(ReservationStatus.ACTIVE, ReservationStatus.RELEASED)) {
            throw new IllegalStateException("Reservation " + reservationId + " is not active");
        }
        StockItem item = repository.find(reservation.getSku(), reservation.getWarehouseId())
                .orElseThrow(() -> new IllegalStateException("Missing stock item for reservation"));
        item.releaseReservation(reservation.getQuantity());
        alertPublisher.publishIfLow(item);
    }

    public Optional<StockSnapshot> getSnapshot(String sku, String warehouseId) {
        return repository.find(sku, warehouseId).map(StockItem::getSnapshot);
    }

    private Reservation requireReservation(String id) {
        Reservation r = reservations.get(id);
        if (r == null) throw new IllegalArgumentException("Unknown reservation: " + id);
        return r;
    }
}
```

## How It Works

**Normal flow.** A warehouse worker calls `addStock("SKU1", "WH1", 100)`. `StockItem` moves from `(onHand=0, reserved=0)` to `(onHand=100, reserved=0)`, so `available = 100`. A checkout thread calls `reserveStock("SKU1", "WH1", 2, "order-42")`. `StockItem.reserve(2)` sees `available = 100 >= 2` and swaps in `(onHand=100, reserved=2)`. A `Reservation` is created with status `ACTIVE`.

**Order confirmed or cancelled.** Once the order ships, `confirmReservation` moves the reservation to `CONFIRMED` and calls `StockItem.confirmReservation(2)`, swapping to `(onHand=98, reserved=0)` — the units are now permanently gone. If instead the order is cancelled first, `releaseReservation` moves the reservation to `RELEASED` and swaps to `(onHand=100, reserved=0)` — the units are available again.

**The overselling scenario — why this is safe.** Suppose `available = 1` (the last unit), and three checkout threads call `reserveStock` for it at the same moment. All three read the same starting snapshot, `available=1`, pass the check, and build an updated snapshot `(reserved=1)`. All three call `compareAndSet(old, new)` on the same `AtomicReference`. The JVM guarantees only one `compareAndSet` can succeed against a given old value — it only swaps if the reference still equals what the thread last read. Exactly one thread wins. The other two see the swap fail, re-read the snapshot (now `available=0`), fail the check, and return `false` immediately — no further retry is useful, since the outcome cannot change until stock is added or released. Only one order gets the last unit.

**Low-stock alert and reorder.** Say the threshold for `SKU1` is 5. Once enough orders bring `available` to 5 or below, `publishIfLow` notices `item.isLowStock()` and calls every subscribed `StockAlertListener`. `ReorderListener.onLowStock` checks `pendingReorders`; if no reorder is already in flight, it calls `SupplierService.placeOrder`, which creates a `PurchaseOrder` and marks the SKU pending. Later, `SupplierService.receiveOrder` (e.g. a warehouse scan on delivery) adds the stock back via `InventoryService.addStock` and clears the pending flag, so a future dip can trigger a fresh reorder.

## How to Extend (Follow-ups)

- **Split one order across warehouses.** Add an allocation `Strategy` interface (`NearestWarehouseFirst`, `SplitAcrossWarehouses`) that calls `reserveStock` against several warehouses and releases any partial reservations if the full quantity cannot be met — a mini two-phase commit over `StockItem`s.
- **Reservation expiry.** Add a `reservedAt` timestamp and a background task that scans `ACTIVE` reservations older than a TTL (e.g. 15 minutes) and releases them — covers abandoned checkouts.
- **Persistence.** Implement `InventoryRepository` against a database. `onHand`/`reserved` need an optimistic-lock `version` column to keep the same "read, compute, conditional write, retry on conflict" pattern in SQL (`UPDATE ... WHERE version = ?`).
- **Multiple nodes.** `AtomicReference` only helps within one process. Scaling out needs row-level optimistic locking in a database (as above), or a distributed atomic operation, e.g. a Redis Lua script that performs check-and-decrement atomically.
- **Smarter reorder policy.** Replace the fixed `threshold * 3` with a `ReorderPolicy` interface (Strategy pattern) — e.g. an Economic Order Quantity (EOQ) calculation using demand rate and supplier lead time.
- **Audit trail.** Every state-changing call can append an immutable `StockEvent` to a log, giving "why does this SKU show 42 units" a full history.

## Complexity & Thread-Safety Notes

| Operation | Time | Notes |
|---|---|---|
| `InMemoryInventoryRepository.findOrCreate` / `find` | O(1) amortized | `ConcurrentHashMap` lookup; `computeIfAbsent` is atomic per key |
| `StockItem.reserve` / `removeStock` | O(1) amortized | CAS loop; a thread may retry a few times under contention, but each retry does O(1) work |
| `StockItem.addStock` / `confirmReservation` / `releaseReservation` | O(1) | no failure condition, so a single `updateAndGet` suffices, no retry needed for correctness |
| `StockAlertPublisher.publishIfLow` | O(k) | k = number of subscribed listeners |
| `ReorderListener.onLowStock` | O(1) | `ConcurrentHashMap.newKeySet().add` is the atomic dedupe check |

**Thread-safety:**

- The heart of the design is `AtomicReference<StockSnapshot>` plus a compare-and-swap loop — **lock-free**, no thread ever blocks another. Under very high contention on one hot SKU, threads may retry several times (a livelock risk), which can be bounded by capping retries and falling back to a short lock if needed — a trade-off worth stating, not hiding.
- Bundling `onHand` and `reserved` into one immutable `StockSnapshot` is what makes "check available, then reserve" atomic. Two separate `AtomicInteger` fields cannot give this guarantee, since a thread could act on one field while the other has already changed underneath it.
- `Reservation.status` uses the same CAS idea, so calling `confirmReservation` or `releaseReservation` twice — e.g. a retried network request — only succeeds once. The second call sees the status is no longer `ACTIVE` and fails loudly instead of double-deducting stock.
- `ConcurrentHashMap.computeIfAbsent` guarantees two threads racing to create the `StockItem` for the same SKU and warehouse never end up with two different instances holding two different `AtomicReference`s, which would silently split the truth about one product's stock.
- `StockAlertPublisher` uses `CopyOnWriteArrayList` for listeners, since listeners are registered rarely (at startup) and read often (every stock change) — cheap reads, expensive writes fits.

## Interview Tips & Common Mistakes

- **Never split "check" and "act" into two steps guarded by nothing.** `if (available >= qty) { reserved += qty; }` written naively is a classic race: two threads can both pass the `if` before either updates `reserved`. Fix it with a CAS loop on one combined state object (shown here), or a lock that wraps both the check and the update as one critical section.
- **Do not use two independent counters for `onHand` and `reserved`.** Even if each is individually atomic, "read both, decide, write one" is not atomic as a whole. Bundle fields that must be read and written together into one immutable object behind one atomic reference.
- **Explain the CAS retry loop out loud.** Interviewers want to hear "read, compute, compareAndSet, and on failure retry with a fresh read" — not just "I used `AtomicReference`." A failed `compareAndSet` means another thread won; the fix is to retry, not treat it as an error.
- **Lock granularity matters.** A lock (or shared `AtomicReference`) per `(SKU, warehouse)` pair, not per service, lets unrelated products update in parallel — a global lock over the whole inventory kills throughput.
- **Make confirm/release idempotent.** Networks retry requests. If `confirmReservation` can run twice and both times deduct stock, that is a silent, hard-to-find bug. Guard the reservation's own status with a compare-and-swap, exactly like the stock count itself.
- **Do not reorder on every low-stock check.** Without the `pendingReorders` guard, a SKU that stays below threshold for an hour while ten more orders arrive would fire ten purchase orders. Track "reorder already placed" and clear it only when new stock actually arrives.
