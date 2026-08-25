# Vending Machine (LLD)

## Problem

Design and build a vending machine in Java. A user walks up, inserts coins or notes, selects an item by its slot code (like `A1`), and the machine dispenses the item and returns any extra change. The machine must track how much money each slot's item costs, and how many items are left in each slot.

This is the classic example used to teach the **State design pattern**. The machine's behavior changes completely depending on its current state. Before money is inserted, "select item" should fail. After money is inserted, "insert more money" should still work, but "dispense" should not work until enough money is in. This naturally maps to a small set of states, each with its own rules. We will build the machine around a `VendingMachineState` interface instead of one big class full of `if/else` checks on a status flag.

## Requirements & Clarifying Questions

Before coding, an interviewer expects you to ask questions. Here are the important ones, with the answers we assume for this design.

**Functional requirements**

1. The machine has many **slots**. Each slot has a code (e.g., `A1`), an item name, a price, and a quantity in stock.
2. A user can insert coins and notes, one at a time. The machine keeps a running total of inserted money.
3. A user can select a slot code.
4. If the slot is out of stock, tell the user and let them insert more money or cancel.
5. If the inserted money is less than the price, tell the user the shortfall and wait for more money or a cancel.
6. If the inserted money is enough, dispense the item and return change (inserted amount minus price).
7. A user can cancel at any point before dispensing. All inserted money is refunded.
8. The machine can go into an **out-of-service** state (e.g., no items left anywhere, or a hardware fault). In this state, it should reject money and selections, and only allow refund of any money already in the machine, or an admin restock action.

**Clarifying questions to ask the interviewer**

- What kind of money does the machine accept? *Assume: a fixed set of coin and note denominations, modeled as an enum, in cents.*
- Can a user select an item before inserting money? *Assume: no. The classic machine wants money first, then item, but we will also cover the reverse flow as a follow-up.*
- Does the machine need to compute exact change using specific coins (e.g., "2 quarters and 1 dime"), or is a total refund amount enough? *Assume: a total amount is enough for the base design; exact coin dispensing is a follow-up.*
- Should multiple items be selectable in one transaction (a "cart")? *Assume: no, one item per transaction, to keep the base design simple. Mention it as a follow-up.*
- Is thread-safety needed? *Assume: yes. A real machine has one keypad and one coin slot, but the software should still guard shared state, in case of concurrent admin restock calls or multiple machine instances sharing an inventory service.*

**Non-functional requirements**

- Adding a new state (e.g., a "maintenance" state) should not force changes inside unrelated states. This is the Open/Closed Principle.
- The code must not use one large `if (status == X) ... else if (status == Y) ...` block. State-specific logic must live inside state classes.

## Design / Approach

### Why the State pattern, and not a big if/else

A vending machine's allowed actions depend on "where it is" in a flow: no money in, money in, dispensing, or broken. Without the State pattern, you would write one `VendingMachine` class with a `status` enum field, and every method (`insertMoney`, `selectItem`, `dispense`, `refund`) would start with a long `switch` or `if/else` on that field. Every time you add a new state, you must revisit and edit every one of those methods. This violates the Open/Closed Principle, and the methods grow long and hard to read.

With the State pattern, each state is its own class. Each class implements only the behavior that is valid for that state, and defines what state comes next. The `VendingMachine` class (called the **context**) just holds a reference to the "current state" object and forwards every call to it. Adding a new state means writing one new class; existing classes do not change. This is the same idea used in TCP connection state machines, order-processing workflows, and traffic-light controllers.

### Key design decisions

1. **State pattern.** `VendingMachineState` is an interface with `insertMoney`, `selectItem`, `dispense`, and `refund`. Four implementations: `IdleState`, `HasMoneyState`, `DispensingState`, `OutOfServiceState`.
2. **Context holds shared data, not behavior.** `VendingMachine` (the context) holds the inventory, the running balance, the selected slot, and a reference to the current `VendingMachineState`. States read and mutate this shared data through the context reference, but the *decision logic* ("is this allowed right now, what happens next") lives in the state classes.
3. **Enum for money.** `Coin` and `Note` enums each carry a value in cents. A `Money` value class wraps a total in cents, so we never do floating-point math with currency.
4. **Slot / Inventory as a separate concern.** `Slot` holds an item, price, and quantity. `Inventory` is a map of slot code to `Slot`, with its own thread-safe update methods. This keeps stock management independent of the state machine, following the Single Responsibility Principle.
5. **Factory-style singletons for states.** Since state objects hold no per-transaction data (all mutable data lives in the context), each state can be a stateless singleton, saved as a `static final` field. This avoids allocating a new state object on every transition.
6. **Thread-safety.** The context uses a lock around state transitions and balance changes, since inventory updates and money handling must not interleave incorrectly if multiple threads (e.g., a maintenance thread doing restock, and a user thread) touch the same machine.

### ASCII sketch

```
                     ┌───────────────────────┐
                     │     VendingMachine      │  (Context)
                     │  - state: VendingMachineState
                     │  - inventory: Inventory │
                     │  - balanceCents: long   │
                     │  - selectedSlot: String │
                     └───────────┬─────────────┘
                                 │ delegates every call to current state
                                 ▼
                     ┌───────────────────────────┐
                     │  <<interface>>              │
                     │  VendingMachineState        │
                     │  + insertMoney(machine, m)  │
                     │  + selectItem(machine, code)│
                     │  + dispense(machine)        │
                     │  + refund(machine)          │
                     └───────────┬────────────────┘
              ┌──────────────┬───┴───────────┬───────────────────┐
              ▼              ▼                ▼                   ▼
      ┌──────────────┐ ┌──────────────┐ ┌────────────────┐ ┌────────────────┐
      │  IdleState    │ │ HasMoneyState │ │ DispensingState │ │ OutOfServiceState │
      │ money in ->   │ │ select item ->│ │ (transient,     │ │ everything      │
      │  HasMoneyState│ │  check price: │ │  runs dispense  │ │  rejected except │
      │ select/refund │ │  short: stay  │ │  then refund    │ │  refund/restock  │
      │  -> rejected  │ │  enough: go to│ │  change, then   │ │                  │
      │               │ │  DispensingSt.│ │  -> IdleState   │ │                  │
      └──────────────┘ └──────────────┘ └────────────────┘ └────────────────┘

State transitions:
  Idle --insertMoney--> HasMoney
  HasMoney --insertMoney--> HasMoney (adds to balance)
  HasMoney --selectItem, price met--> Dispensing
  HasMoney --selectItem, price NOT met--> HasMoney (stays, reports shortfall)
  HasMoney --refund--> Idle
  Dispensing --dispense (auto)--> Idle (or OutOfService if stock hits zero everywhere)
  Any state --admin fault / stock empty--> OutOfService
  OutOfService --admin restock/reset--> Idle
```

Patterns used: **State** (`VendingMachineState` and its four implementations), **Singleton** (one shared instance per stateless state class), **Facade** (`VendingMachine` exposes a simple API over inventory + state + money), and **Value Object** (`Money`, immutable, for currency amounts).

## Java Solution

```java
import java.util.EnumMap;
import java.util.Map;
import java.util.concurrent.ConcurrentHashMap;
import java.util.concurrent.locks.ReentrantLock;

// ---------- Money (enums + value object) ----------
public enum Coin {
    PENNY(1), NICKEL(5), DIME(10), QUARTER(25);
    private final int cents;
    Coin(int cents) { this.cents = cents; }
    public int getCents() { return cents; }
}

public enum Note {
    ONE_DOLLAR(100), FIVE_DOLLAR(500), TEN_DOLLAR(1000);
    private final int cents;
    Note(int cents) { this.cents = cents; }
    public int getCents() { return cents; }
}

// Immutable value object: all currency math happens in integer cents,
// never in floating point, to avoid rounding bugs.
public final class Money {
    private final long cents;
    private Money(long cents) { this.cents = cents; }
    public static Money ofCents(long cents) { return new Money(cents); }
    public static Money zero() { return new Money(0); }
    public long getCents() { return cents; }
    public Money plus(Money other) { return new Money(this.cents + other.cents); }
    public Money minus(Money other) { return new Money(this.cents - other.cents); }
    public boolean isAtLeast(Money other) { return this.cents >= other.cents; }
    @Override public String toString() {
        return String.format("$%d.%02d", cents / 100, cents % 100);
    }
}

// ---------- Inventory ----------
public final class Item {
    private final String name;
    public Item(String name) { this.name = name; }
    public String getName() { return name; }
}

public final class Slot {
    private final String code;
    private final Item item;
    private final Money price;
    private int quantity;

    public Slot(String code, Item item, Money price, int quantity) {
        this.code = code;
        this.item = item;
        this.price = price;
        this.quantity = quantity;
    }

    public String getCode() { return code; }
    public Item getItem() { return item; }
    public Money getPrice() { return price; }
    public synchronized int getQuantity() { return quantity; }
    public synchronized boolean isInStock() { return quantity > 0; }

    // Only the Inventory calls this, itself guarded, but kept synchronized
    // here too, so Slot stays safe even if used directly elsewhere.
    synchronized void decrementStock() {
        if (quantity <= 0) throw new IllegalStateException("Slot " + code + " is empty");
        quantity--;
    }

    synchronized void restock(int extra) {
        quantity += extra;
    }
}

public final class Inventory {
    private final Map<String, Slot> slots = new ConcurrentHashMap<>();

    public void addSlot(Slot slot) {
        slots.put(slot.getCode(), slot);
    }

    public Slot getSlot(String code) {
        Slot slot = slots.get(code);
        if (slot == null) throw new NoSuchElementException("No such slot: " + code);
        return slot;
    }

    public void restock(String code, int extra) {
        getSlot(code).restock(extra);
    }

    // Used by the machine to decide if it should go OUT_OF_SERVICE
    public boolean isCompletelyEmpty() {
        return slots.values().stream().noneMatch(Slot::isInStock);
    }
}

// A tiny local exception type, so this file compiles standalone
class NoSuchElementException extends RuntimeException {
    NoSuchElementException(String msg) { super(msg); }
}

// ---------- State interface ----------
public interface VendingMachineState {
    void insertMoney(VendingMachine machine, Money amount);
    void selectItem(VendingMachine machine, String slotCode);
    void dispense(VendingMachine machine);
    void refund(VendingMachine machine);
}

// ---------- Idle: no money inserted yet ----------
public final class IdleState implements VendingMachineState {
    public static final IdleState INSTANCE = new IdleState();
    private IdleState() { }

    @Override
    public void insertMoney(VendingMachine machine, Money amount) {
        machine.addToBalance(amount);
        machine.setState(HasMoneyState.INSTANCE);
        System.out.println("Money inserted: " + amount + ". Balance: " + machine.getBalance());
    }

    @Override
    public void selectItem(VendingMachine machine, String slotCode) {
        System.out.println("Please insert money before selecting an item.");
    }

    @Override
    public void dispense(VendingMachine machine) {
        System.out.println("Nothing to dispense. Insert money and select an item first.");
    }

    @Override
    public void refund(VendingMachine machine) {
        System.out.println("No money to refund.");
    }
}

// ---------- HasMoney: at least one coin/note inserted, no item picked yet, or price not yet met ----------
public final class HasMoneyState implements VendingMachineState {
    public static final HasMoneyState INSTANCE = new HasMoneyState();
    private HasMoneyState() { }

    @Override
    public void insertMoney(VendingMachine machine, Money amount) {
        machine.addToBalance(amount);
        System.out.println("Money inserted: " + amount + ". Balance: " + machine.getBalance());
    }

    @Override
    public void selectItem(VendingMachine machine, String slotCode) {
        Slot slot = machine.getInventory().getSlot(slotCode);

        if (!slot.isInStock()) {
            System.out.println("Slot " + slotCode + " is out of stock. Pick another, or press refund.");
            return; // stay in HasMoneyState; money is kept for another attempt
        }

        if (!machine.getBalance().isAtLeast(slot.getPrice())) {
            Money shortfall = slot.getPrice().minus(machine.getBalance());
            System.out.println("Insufficient money. Need " + shortfall + " more.");
            return; // stay in HasMoneyState
        }

        // Enough money and in stock: lock in the selection and move to Dispensing.
        machine.setSelectedSlot(slotCode);
        machine.setState(DispensingState.INSTANCE);
        machine.dispense(); // auto-trigger the dispensing step
    }

    @Override
    public void dispense(VendingMachine machine) {
        System.out.println("Select an item first.");
    }

    @Override
    public void refund(VendingMachine machine) {
        Money refunded = machine.getBalance();
        machine.resetBalance();
        machine.setState(IdleState.INSTANCE);
        System.out.println("Refunded: " + refunded);
    }
}

// ---------- Dispensing: transient state, does the actual work, then transitions out ----------
public final class DispensingState implements VendingMachineState {
    public static final DispensingState INSTANCE = new DispensingState();
    private DispensingState() { }

    @Override
    public void insertMoney(VendingMachine machine, Money amount) {
        System.out.println("Please wait, dispensing in progress.");
    }

    @Override
    public void selectItem(VendingMachine machine, String slotCode) {
        System.out.println("Please wait, dispensing in progress.");
    }

    @Override
    public void dispense(VendingMachine machine) {
        String slotCode = machine.getSelectedSlot();
        Slot slot = machine.getInventory().getSlot(slotCode);

        slot.decrementStock();
        Money change = machine.getBalance().minus(slot.getPrice());

        System.out.println("Dispensing item: " + slot.getItem().getName());
        if (change.getCents() > 0) {
            System.out.println("Returning change: " + change);
        }

        machine.resetBalance();
        machine.setSelectedSlot(null);

        // Decide the next state: if the whole machine is now empty, go OUT_OF_SERVICE.
        if (machine.getInventory().isCompletelyEmpty()) {
            machine.setState(OutOfServiceState.INSTANCE);
            System.out.println("Machine is now out of service: all slots empty.");
        } else {
            machine.setState(IdleState.INSTANCE);
        }
    }

    @Override
    public void refund(VendingMachine machine) {
        System.out.println("Cannot refund while dispensing.");
    }
}

// ---------- OutOfService: rejects normal use, allows refund and admin restock ----------
public final class OutOfServiceState implements VendingMachineState {
    public static final OutOfServiceState INSTANCE = new OutOfServiceState();
    private OutOfServiceState() { }

    @Override
    public void insertMoney(VendingMachine machine, Money amount) {
        System.out.println("Machine is out of service. Money not accepted.");
    }

    @Override
    public void selectItem(VendingMachine machine, String slotCode) {
        System.out.println("Machine is out of service. Cannot select items.");
    }

    @Override
    public void dispense(VendingMachine machine) {
        System.out.println("Machine is out of service.");
    }

    @Override
    public void refund(VendingMachine machine) {
        if (machine.getBalance().getCents() > 0) {
            Money refunded = machine.getBalance();
            machine.resetBalance();
            System.out.println("Refunded: " + refunded);
        } else {
            System.out.println("No money to refund.");
        }
    }
}

// ---------- VendingMachine (Context / Facade) ----------
public final class VendingMachine {
    private final Inventory inventory;
    private volatile VendingMachineState state;
    private volatile Money balance;
    private volatile String selectedSlot;
    private final ReentrantLock lock = new ReentrantLock();

    public VendingMachine(Inventory inventory) {
        this.inventory = inventory;
        this.state = IdleState.INSTANCE;
        this.balance = Money.zero();
    }

    // ---- public API: every call is forwarded to the current state ----
    public void insertCoin(Coin coin) {
        lock.lock();
        try {
            state.insertMoney(this, Money.ofCents(coin.getCents()));
        } finally {
            lock.unlock();
        }
    }

    public void insertNote(Note note) {
        lock.lock();
        try {
            state.insertMoney(this, Money.ofCents(note.getCents()));
        } finally {
            lock.unlock();
        }
    }

    public void selectItem(String slotCode) {
        lock.lock();
        try {
            state.selectItem(this, slotCode);
        } finally {
            lock.unlock();
        }
    }

    public void dispense() {
        // called internally by HasMoneyState after a successful selection,
        // kept public in case an interviewer wants a manual "confirm" button
        lock.lock();
        try {
            state.dispense(this);
        } finally {
            lock.unlock();
        }
    }

    public void refund() {
        lock.lock();
        try {
            state.refund(this);
        } finally {
            lock.unlock();
        }
    }

    // ---- helpers used by state classes (package-visible would be enough;
    //      public here so all states live in the same package cleanly) ----
    void setState(VendingMachineState newState) { this.state = newState; }
    void addToBalance(Money amount) { this.balance = this.balance.plus(amount); }
    void resetBalance() { this.balance = Money.zero(); }
    void setSelectedSlot(String code) { this.selectedSlot = code; }

    public Inventory getInventory() { return inventory; }
    public Money getBalance() { return balance; }
    public String getSelectedSlot() { return selectedSlot; }
    public VendingMachineState getState() { return state; }
}
```

### Example usage

```java
public class Demo {
    public static void main(String[] args) {
        Inventory inventory = new Inventory();
        inventory.addSlot(new Slot("A1", new Item("Chips"), Money.ofCents(150), 2));
        inventory.addSlot(new Slot("A2", new Item("Soda"), Money.ofCents(125), 1));

        VendingMachine machine = new VendingMachine(inventory);

        machine.selectItem("A1");            // rejected: no money yet (Idle)
        machine.insertCoin(Coin.QUARTER);     // 25 cents, moves to HasMoney
        machine.insertNote(Note.ONE_DOLLAR);  // +100 cents, balance 125
        machine.selectItem("A1");             // price is 150, still short by 25
        machine.insertCoin(Coin.QUARTER);     // balance now 150
        machine.selectItem("A1");             // enough: dispenses "Chips", 0 change

        machine.insertNote(Note.FIVE_DOLLAR); // 500 cents
        machine.selectItem("A2");             // price 125, dispenses "Soda", change 375
        machine.selectItem("A2");             // now out of stock in A2

        machine.insertCoin(Coin.DIME);
        machine.refund();                     // refunds 10 cents
    }
}
```

## How It Works

1. **The context never decides "what is allowed."** `VendingMachine` methods like `insertCoin` and `selectItem` do only two things: convert the input into a domain call, and forward it to `state.insertMoney(...)` or `state.selectItem(...)`. The decision of whether that action is valid right now, and what happens next, lives entirely inside the current state object. This is the core idea of the State pattern: behavior is attached to state objects, not scattered across `if` checks in the context.
2. **`IdleState`** only accepts `insertMoney`. Any money coming in immediately moves the machine to `HasMoneyState`. Selecting an item or asking to dispense in this state is simply rejected with a message; there is no balance to check against yet.
3. **`HasMoneyState`** accepts more money (adding to the running balance) and handles `selectItem`. This is where the important business rule lives: look up the slot, check stock, then check if the balance covers the price. If stock is missing or money is short, the machine **stays in `HasMoneyState`**, so the user's money is preserved and they can try again or ask for a refund. Only when both checks pass does it record the selected slot and transition to `DispensingState`.
4. **`DispensingState`** is a short-lived state. As soon as the machine enters it, `HasMoneyState.selectItem` immediately calls `machine.dispense()`, so a real user never has to make a separate call. Inside `dispense`, the slot's stock is decremented, change is computed as `balance - price`, the balance is reset to zero, and the machine decides its next state: back to `IdleState` normally, or forward to `OutOfServiceState` if that was the last item across the entire inventory.
5. **`OutOfServiceState`** blocks money and selection, but still allows `refund`, in case money was already in the machine when it went out of service (for example, an admin trips a fault after a partial transaction). A real system would add an admin-only `restock` and `reset` action to bring the machine back to `IdleState`.
6. **Why `dispense` is on the interface even though only `DispensingState` really does something with it:** every state must define every method from `VendingMachineState`, even if the implementation is just "print a message and do nothing." This is what makes the pattern safe: the compiler forces you to handle every action in every state, so you cannot forget a case, unlike a `switch` where a missing branch silently falls through.
7. **Thread-safety.** `VendingMachine` wraps every public call in a `ReentrantLock`, so only one logical transaction step (insert, select, dispense, refund) runs at a time on a given machine instance. `Slot`'s stock fields are also individually `synchronized`, as a second layer of protection, in case `Inventory` is ever shared and updated (e.g., admin restock) outside the lock held by `VendingMachine`. The `state` and `balance` fields are `volatile`, so a state change made under the lock is visible to any thread reading `getState()` without needing to acquire the lock, useful for a monitoring or display thread.

## How to Extend (Follow-ups)

Interviewers often ask "what if we needed X?" Have answers ready for these:

- **Exact change dispensing.** Instead of returning a lump `Money` amount, add a `CashBox` component that tracks how many of each coin/note the machine holds, and a greedy or dynamic-programming algorithm to pick the combination of coins/notes that sums to the change amount, using the fewest pieces. If the exact change cannot be made, the machine should refuse the transaction and refund instead of dispensing, rather than shortchanging the customer.
- **Select item first, then pay.** Add a state, say `ItemSelectedState`, entered directly from `IdleState` if item-first flow is required, that shows the price and waits for money, then transitions to `DispensingState` once paid. This shows that adding a state does not require touching `HasMoneyState` or `DispensingState`.
- **Multiple items per transaction (a cart).** Replace the single `selectedSlot` field with a list of selections and running total, and only trigger `DispensingState` when the user explicitly confirms checkout. The state classes' responsibilities stay the same; only the context's stored data grows.
- **Card / digital payment.** Add a new `Payment` strategy interface (`CashPayment`, `CardPayment`), and let `insertMoney`-equivalent calls go through it. The states do not need to know which payment type was used, only that money arrived.
- **Admin operations (restock, set price, force reset).** Add an `AdminState` or, more simply, handle admin calls outside the customer-facing state machine entirely, directly on `Inventory`, since restocking should be legal regardless of the machine's current customer-facing state.
- **Timeouts.** If a user inserts money but never selects an item, add a scheduled task that calls `refund()` automatically after, say, 30 seconds, using a `ScheduledExecutorService`. This does not need a new state; it just calls the existing `refund` action on whatever state is current.
- **Auditing / event logging.** Wrap state transitions with an `Observer` (a `TransitionListener` interface) so an external system can log every state change for analytics, without modifying the state classes themselves.

## Complexity & Thread-Safety Notes

- **Time complexity:** every operation (`insertMoney`, `selectItem`, `dispense`, `refund`) runs in O(1) time, aside from `Inventory.isCompletelyEmpty()`, which is O(n) in the number of slots. This check only runs once per successful dispense, so it does not affect the cost of the common path (insert money, check price).
- **Space:** O(n) for n slots in the inventory. State objects are singletons (`INSTANCE` fields), so no extra memory is used per transaction; the only per-transaction data is the balance and selected-slot fields on the context, both fixed-size.
- **Thread-safety summary:**
    - All customer-facing operations on `VendingMachine` go through a single `ReentrantLock`, which serializes state transitions. This is correct behavior for a real machine, since only one physical coin slot and one keypad exist; two customers cannot truly act at the same time.
    - `Slot` stock updates are individually `synchronized`, so admin restocking is safe even if it happens outside the main lock (for example, from a separate admin thread that talks to `Inventory` directly, bypassing `VendingMachine`).
    - `state` and `balance` are `volatile`, giving safe, lock-free reads for monitoring code, while all writes still happen under the lock for correctness of the read-modify-write sequences (like "read balance, then reset it").
    - The state objects themselves (`IdleState.INSTANCE`, and so on) hold **no mutable fields**, so they are inherently safe to share across threads; all mutable data lives in the context, matching the classic "stateless flyweight state objects" style of the State pattern.

## Interview Tips & Common Mistakes

- **Explain the State pattern in your own words before coding.** Say something like: "the set of valid actions and their effects change based on where the machine is in its flow, so I will model each stage as its own class, instead of a status flag with a big switch statement." This shows the interviewer you understand *why*, not just *that*, you are using this pattern.
- **A common mistake:** putting business logic (like the price check) inside the `VendingMachine` context class instead of inside `HasMoneyState`. If you do this, you are back to a disguised `if/else` machine, and adding a new state again forces edits everywhere. Keep every decision inside the state class that owns it.
- **Another common mistake:** creating a new state object on every transition (`new HasMoneyState()` each time). Since our states hold no per-transaction data, this wastes memory for no benefit. Use a singleton per state class instead, exactly like the `IdleState.INSTANCE` pattern shown here.
- **Do not forget the "stay in the same state" case.** A frequent bug in interview solutions: when money is insufficient, some candidates accidentally transition to a new state anyway, or throw an exception instead of just returning and staying in `HasMoneyState`. Be explicit that "insufficient funds" and "out of stock" are *not* errors, they are just messages, and the machine should remain ready for more input.
- **Be ready to discuss "why not just an enum with a switch."** An enum-and-switch design does work for very small state machines, but it fails the Open/Closed Principle: every new state requires editing every switch statement across the codebase. The State pattern turns "add a state" into "add a class," which is safer and easier to test in isolation, since each state class can be unit tested with a mock context.
- **Mention that `DispensingState` is a good example of an internal/transient state.** Not all states need to be triggered by external user actions; some, like this one, are triggered internally by another state's transition and immediately do their work before moving on. This is a good detail to bring up if asked how you would model a "processing" or "loading" state in other systems (like an order-processing pipeline).
- **If asked about scaling to many machines** (e.g., a fleet of vending machines reporting to a central service), mention that the state and balance are per-machine, in-memory data here, but a real fleet system would persist `state` and `balance` per machine ID in a database or cache, so a machine's in-progress transaction survives a process restart, and central inventory reporting can run across machines without needing to lock any single machine's `VendingMachine` object.
