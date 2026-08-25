# ATM (LLD)

## Problem

Design the software for an ATM. A user inserts a card, enters a PIN, then can check balance, withdraw cash, or deposit cash, and can eject the card at any time.

The ATM's valid actions change with its stage: reject a PIN entry before a card is in, reject a withdrawal before the PIN is verified. This fits the **State pattern**: each stage (idle, card inserted, authenticated, dispensing) becomes its own class, instead of one class full of `if/else` on a status flag.

The hardest part is **cash dispensing**. The ATM holds a limited count of notes per denomination (say $100, $50, $20, $10). For a $170 withdrawal, it must pick notes that sum to exactly $170 from what is left. We use a **greedy algorithm** (Strategy pattern), show where greedy fails even though a valid combination exists, and explain the fix. The bank is a separate service behind an interface, with an **atomic debit**, so the design stays correct when many ATMs touch the same account at once.

## Requirements & Clarifying Questions

**Functional requirements**

1. Insert a card. Only one session runs at a time.
2. Enter a PIN, checked by the bank. Allow 3 attempts; retain the card on the 3rd wrong try.
3. After a correct PIN: check balance, withdraw, or deposit. Allow more than one operation per session.
4. Eject the card any time; this ends the session and returns to idle.
5. Before withdrawal, check the balance is enough, and the ATM has notes that add up to the exact amount.
6. Deposits credit the account and add the deposited notes to the ATM's own stock.
7. The ATM tracks its own cash stock as a count per denomination.

**Clarifying questions, with the assumptions used here**

- Bank called directly, or through a service? *Assume: a `BankService` interface, since the real bank is a separate system (Dependency Inversion).*
- One account per card, or many? *Assume: one, to keep the base design simple; multi-account cards are a follow-up.*
- What withdrawal amounts are allowed, and must notes equal the exact amount? *Assume: positive multiples of the smallest note ($10), always exact, never rounded.*
- Is deposited cash counted automatically? *Assume: given as a note breakdown, as if a bill acceptor already counted it.*
- Is thread-safety needed? *Assume: yes. One ATM handles one session step at a time, but several ATMs may debit the same account at once (two cards, joint account); that debit must be atomic, or the balance can go negative.*

**Non-functional:** a new state must not require editing existing state classes (Open/Closed), and the note-dispensing algorithm must be swappable without touching the state machine or bank logic (Strategy, Single Responsibility).

## Design / Approach

### Why the State pattern fits the ATM flow

An ATM's valid actions depend entirely on its stage. Before a card is in, only "insert card" works. After a card is in but before the PIN check, only "enter PIN" or "eject" works. Once authenticated, withdraw, deposit, and check balance stay valid until eject. Without the State pattern, one class needs a status field and a long switch in every method; every new stage forces edits everywhere. With the State pattern, each stage is its own class that decides what comes next, and `ATMMachine` (the context) just forwards each call to its current state.

### Key design decisions

1. **State pattern** for the flow: `ATMState` interface with `IdleState`, `CardInsertedState`, `AuthenticatedState`, and a transient `DispensingCashState`.
2. **Default methods on the interface.** An invalid action falls back to a default "not allowed" message, so each state overrides only what it handles, while the interface still forces every state to support every method.
3. **Strategy pattern** for notes: `NoteDispenser` interface, with `GreedyNoteDispenser` as the default. Swapping strategies needs no change to any state class.
4. **Dependency Inversion** for the bank: `BankService` interface, with `InMemoryBankService` standing in for a real core-banking system.
5. **Cash as its own component.** `CashInventory` holds a count per `Denomination`, with a two-step withdrawal (`previewWithdrawPlan`, then `commitWithdrawPlan`), so a plan is checked before money moves.
6. **Context holds data, not decisions.** `ATMMachine` stores the card, account ID, PIN attempts, and state; states decide what is allowed.
7. **Debit only after confirming cash is available, with compensation on rare failure** — a small, single-node "saga," as used in payment systems.

### ASCII sketch

```
              ┌────────────────────┐
              │     ATMMachine       │  (Context)
              │ state, card, acctId  │
              │ cashInventory        │
              │ bankService          │
              │ noteDispenser        │
              └──────────┬────────────┘
                         │ delegates to current state
                         ▼
              <<interface>> ATMState (default = "not allowed")
     insertCard / enterPin / withdraw / deposit / checkBalance / ejectCard
                         │
   ┌───────────┬─────────┴──────────┬───────────────────┐
   ▼           ▼                    ▼                    ▼
 Idle    CardInserted         Authenticated      DispensingCash (transient)

Transitions:
  Idle --insertCard--> CardInserted
  CardInserted --pin correct--> Authenticated
  CardInserted --pin wrong, attempts left--> CardInserted
  CardInserted --pin wrong, none left--> Idle (card retained)
  CardInserted --eject--> Idle
  Authenticated --withdraw--> DispensingCash --done--> Authenticated
  Authenticated --deposit / checkBalance--> Authenticated
  Authenticated --eject--> Idle
```

Patterns used: **State** (`ATMState` and its four classes), **Strategy** (`NoteDispenser`), **Singleton** (one instance per stateless state class), **Facade** (`ATMMachine`), and **Dependency Inversion** (`BankService` interface).

## Java Solution

```java
import java.util.*;
import java.util.concurrent.*;
import java.util.concurrent.locks.ReentrantLock;

// ---------- Denominations & cash inventory ----------
public enum Denomination {
    HUNDRED(100), FIFTY(50), TWENTY(20), TEN(10);
    private final int value;
    Denomination(int value) { this.value = value; }
    public int getValue() { return value; }
}

public final class CashInventory {
    private final Map<Denomination, Integer> counts = new EnumMap<>(Denomination.class);
    private final ReentrantLock lock = new ReentrantLock();

    public CashInventory() {
        for (Denomination d : Denomination.values()) counts.put(d, 0);
    }

    public void loadNotes(Denomination d, int count) {
        lock.lock();
        try { counts.merge(d, count, Integer::sum); } finally { lock.unlock(); }
    }

    public Map<Denomination, Integer> previewWithdrawPlan(int amount, NoteDispenser dispenser) {
        lock.lock();
        try { return dispenser.computeNotes(amount, new EnumMap<>(counts)); } finally { lock.unlock(); }
    }

    public void commitWithdrawPlan(Map<Denomination, Integer> plan) {
        lock.lock();
        try {
            for (Map.Entry<Denomination, Integer> e : plan.entrySet()) {
                int have = counts.get(e.getKey());
                if (have < e.getValue()) throw new IllegalStateException("Inventory changed since preview.");
                counts.put(e.getKey(), have - e.getValue());
            }
        } finally { lock.unlock(); }
    }

    public void depositNotes(Map<Denomination, Integer> notes) {
        lock.lock();
        try {
            for (Map.Entry<Denomination, Integer> e : notes.entrySet()) counts.merge(e.getKey(), e.getValue(), Integer::sum);
        } finally { lock.unlock(); }
    }
}

// ---------- Note-dispensing strategy ----------
public interface NoteDispenser {
    Map<Denomination, Integer> computeNotes(int amount, Map<Denomination, Integer> available);
}

public final class InsufficientCashException extends RuntimeException {
    public InsufficientCashException(String msg) { super(msg); }
}

// Greedy: takes as many of the largest note as possible; can fail even when a valid combination exists.
public final class GreedyNoteDispenser implements NoteDispenser {
    @Override
    public Map<Denomination, Integer> computeNotes(int amount, Map<Denomination, Integer> available) {
        List<Denomination> byValueDesc = new ArrayList<>(Arrays.asList(Denomination.values()));
        byValueDesc.sort((a, b) -> b.getValue() - a.getValue());

        Map<Denomination, Integer> plan = new EnumMap<>(Denomination.class);
        int remaining = amount;
        for (Denomination d : byValueDesc) {
            int have = available.getOrDefault(d, 0);
            int use = Math.min(have, remaining / d.getValue());
            if (use > 0) {
                plan.put(d, use);
                remaining -= use * d.getValue();
            }
        }
        if (remaining != 0) {
            throw new InsufficientCashException("Cannot make " + amount + "; short by " + remaining + ".");
        }
        return plan;
    }
}

// ---------- Bank side ----------
public final class InsufficientFundsException extends RuntimeException {
    public InsufficientFundsException(String msg) { super(msg); }
}

public interface BankService {
    boolean validatePin(String cardNumber, String pin);
    int getBalance(String accountId);
    void debit(String accountId, int amount);   // must be atomic
    void credit(String accountId, int amount);
}

public final class InMemoryBankService implements BankService {
    private static final class Account {
        int balance;
        Account(int balance) { this.balance = balance; }
    }

    private final Map<String, String> cardToPin = new ConcurrentHashMap<>(); // demo only
    private final Map<String, Account> accounts = new ConcurrentHashMap<>();

    public void registerCard(String cardNumber, String pin, String accountId, int openingBalance) {
        cardToPin.put(cardNumber, pin);
        accounts.put(accountId, new Account(openingBalance));
    }

    @Override public boolean validatePin(String cardNumber, String pin) {
        return Objects.equals(cardToPin.get(cardNumber), pin);
    }

    @Override public int getBalance(String accountId) { return getAccount(accountId).balance; }

    @Override public void debit(String accountId, int amount) {
        Account account = getAccount(accountId);
        synchronized (account) {
            if (account.balance < amount) throw new InsufficientFundsException("Balance too low: " + accountId);
            account.balance -= amount; // atomic
        }
    }

    @Override public void credit(String accountId, int amount) {
        Account account = getAccount(accountId);
        synchronized (account) { account.balance += amount; }
    }

    private Account getAccount(String accountId) {
        Account account = accounts.get(accountId);
        if (account == null) throw new IllegalArgumentException("Unknown account: " + accountId);
        return account;
    }
}

// ---------- Card ----------
public final class Card {
    private final String cardNumber;
    private final String accountId;
    public Card(String cardNumber, String accountId) { this.cardNumber = cardNumber; this.accountId = accountId; }
    public String getCardNumber() { return cardNumber; }
    public String getAccountId() { return accountId; }
}

// ---------- State pattern (defaults = "not allowed"; a state overrides only what it handles) ----------
public interface ATMState {
    default void insertCard(ATMMachine atm, Card card) { System.out.println("Not allowed right now."); }
    default void enterPin(ATMMachine atm, String pin) { System.out.println("Not allowed right now."); }
    default void withdraw(ATMMachine atm, int amount) { System.out.println("Not allowed right now."); }
    default void deposit(ATMMachine atm, Map<Denomination, Integer> notes) { System.out.println("Not allowed right now."); }
    default void checkBalance(ATMMachine atm) { System.out.println("Not allowed right now."); }
    default void ejectCard(ATMMachine atm) { System.out.println("No card to eject."); }
}

public final class IdleState implements ATMState {
    public static final IdleState INSTANCE = new IdleState();
    private IdleState() { }

    @Override public void insertCard(ATMMachine atm, Card card) {
        atm.setCurrentCard(card);
        atm.setPinAttemptsLeft(3);
        atm.setState(CardInsertedState.INSTANCE);
        System.out.println("Card inserted. Enter your PIN.");
    }
}

public final class CardInsertedState implements ATMState {
    public static final CardInsertedState INSTANCE = new CardInsertedState();
    private CardInsertedState() { }

    @Override public void insertCard(ATMMachine atm, Card card) { System.out.println("A card is already inserted."); }

    @Override public void enterPin(ATMMachine atm, String pin) {
        Card card = atm.getCurrentCard();
        if (atm.getBankService().validatePin(card.getCardNumber(), pin)) {
            atm.setCurrentAccountId(card.getAccountId());
            atm.setState(AuthenticatedState.INSTANCE);
            System.out.println("PIN accepted. Select an operation.");
        } else {
            atm.decrementPinAttempts();
            if (atm.getPinAttemptsLeft() <= 0) {
                System.out.println("Too many wrong attempts. Card retained.");
                atm.clearSession();
                atm.setState(IdleState.INSTANCE);
            } else {
                System.out.println("Wrong PIN. Attempts left: " + atm.getPinAttemptsLeft());
            }
        }
    }

    @Override public void ejectCard(ATMMachine atm) {
        System.out.println("Card ejected.");
        atm.clearSession();
        atm.setState(IdleState.INSTANCE);
    }
}

public final class AuthenticatedState implements ATMState {
    public static final AuthenticatedState INSTANCE = new AuthenticatedState();
    private AuthenticatedState() { }

    @Override public void withdraw(ATMMachine atm, int amount) {
        if (amount <= 0 || amount % 10 != 0) {
            System.out.println("Enter a positive amount that is a multiple of 10.");
            return;
        }
        atm.setState(DispensingCashState.INSTANCE);
        atm.getState().withdraw(atm, amount);
    }

    @Override public void deposit(ATMMachine atm, Map<Denomination, Integer> notes) {
        int total = 0;
        for (Map.Entry<Denomination, Integer> e : notes.entrySet()) total += e.getKey().getValue() * e.getValue();
        atm.getCashInventory().depositNotes(notes);
        atm.getBankService().credit(atm.getCurrentAccountId(), total);
        System.out.println("Deposited " + total + ". New balance: " + atm.getBankService().getBalance(atm.getCurrentAccountId()));
    }

    @Override public void checkBalance(ATMMachine atm) {
        System.out.println("Balance: " + atm.getBankService().getBalance(atm.getCurrentAccountId()));
    }

    @Override public void ejectCard(ATMMachine atm) {
        System.out.println("Card ejected. Thank you.");
        atm.clearSession();
        atm.setState(IdleState.INSTANCE);
    }
}

// Transient state: does the withdrawal, then returns to AuthenticatedState.
public final class DispensingCashState implements ATMState {
    public static final DispensingCashState INSTANCE = new DispensingCashState();
    private DispensingCashState() { }

    @Override
    public void withdraw(ATMMachine atm, int amount) {
        String accountId = atm.getCurrentAccountId();

        if (atm.getBankService().getBalance(accountId) < amount) {
            System.out.println("Insufficient funds.");
            atm.setState(AuthenticatedState.INSTANCE);
            return;
        }

        Map<Denomination, Integer> plan;
        try {
            plan = atm.getCashInventory().previewWithdrawPlan(amount, atm.getNoteDispenser());
        } catch (InsufficientCashException e) {
            System.out.println("Cannot dispense " + amount + ": " + e.getMessage());
            atm.setState(AuthenticatedState.INSTANCE);
            return;
        }

        try {
            atm.getBankService().debit(accountId, amount);
        } catch (InsufficientFundsException e) {
            System.out.println("Debit failed: " + e.getMessage());
            atm.setState(AuthenticatedState.INSTANCE);
            return;
        }

        try {
            atm.getCashInventory().commitWithdrawPlan(plan);
        } catch (IllegalStateException e) {
            atm.getBankService().credit(accountId, amount); // compensate: refund the debit
            System.out.println("Cash unavailable at the last moment; amount refunded.");
            atm.setState(AuthenticatedState.INSTANCE);
            return;
        }

        System.out.println("Dispensing: " + plan);
        atm.setState(AuthenticatedState.INSTANCE);
    }
}

// ---------- Context / Facade ----------
public final class ATMMachine {
    private final CashInventory cashInventory;
    private final BankService bankService;
    private final NoteDispenser noteDispenser;
    private final ReentrantLock lock = new ReentrantLock();

    private volatile ATMState state = IdleState.INSTANCE;
    private volatile Card currentCard;
    private volatile String currentAccountId;
    private volatile int pinAttemptsLeft;

    public ATMMachine(CashInventory cashInventory, BankService bankService, NoteDispenser noteDispenser) {
        this.cashInventory = cashInventory;
        this.bankService = bankService;
        this.noteDispenser = noteDispenser;
    }

    public void insertCard(Card card) { withLock(() -> state.insertCard(this, card)); }
    public void enterPin(String pin) { withLock(() -> state.enterPin(this, pin)); }
    public void withdraw(int amount) { withLock(() -> state.withdraw(this, amount)); }
    public void deposit(Map<Denomination, Integer> notes) { withLock(() -> state.deposit(this, notes)); }
    public void checkBalance() { withLock(() -> state.checkBalance(this)); }
    public void ejectCard() { withLock(() -> state.ejectCard(this)); }

    private void withLock(Runnable action) {
        lock.lock();
        try { action.run(); } finally { lock.unlock(); }
    }

    // helpers for state classes
    void setState(ATMState s) { this.state = s; }
    void setCurrentCard(Card c) { this.currentCard = c; }
    void setCurrentAccountId(String id) { this.currentAccountId = id; }
    void setPinAttemptsLeft(int n) { this.pinAttemptsLeft = n; }
    void decrementPinAttempts() { this.pinAttemptsLeft--; }
    void clearSession() { this.currentCard = null; this.currentAccountId = null; this.pinAttemptsLeft = 0; }

    public ATMState getState() { return state; }
    public Card getCurrentCard() { return currentCard; }
    public String getCurrentAccountId() { return currentAccountId; }
    public int getPinAttemptsLeft() { return pinAttemptsLeft; }
    public CashInventory getCashInventory() { return cashInventory; }
    public BankService getBankService() { return bankService; }
    public NoteDispenser getNoteDispenser() { return noteDispenser; }
}
```

### Example usage

```java
public class Demo {
    public static void main(String[] args) {
        InMemoryBankService bank = new InMemoryBankService();
        bank.registerCard("1111-2222-3333-4444", "1234", "ACC-1", 500);

        CashInventory cash = new CashInventory();
        cash.loadNotes(Denomination.HUNDRED, 5);
        cash.loadNotes(Denomination.FIFTY, 1);
        cash.loadNotes(Denomination.TWENTY, 3);

        ATMMachine atm = new ATMMachine(cash, bank, new GreedyNoteDispenser());

        atm.withdraw(50);                                    // rejected: no card yet
        atm.insertCard(new Card("1111-2222-3333-4444", "ACC-1"));
        atm.enterPin("0000");                                // wrong, 2 attempts left
        atm.enterPin("1234");                                // correct
        atm.checkBalance();                                  // 500

        atm.withdraw(60);                                    // greedy FAILS (see below)
        atm.withdraw(40);                                    // greedy succeeds

        Map<Denomination, Integer> deposit = new EnumMap<>(Denomination.class);
        deposit.put(Denomination.HUNDRED, 1);
        atm.deposit(deposit);                                // credits 100

        atm.checkBalance();                                  // 560
        atm.ejectCard();
    }
}
```

## How It Works

1. **The context never decides what is allowed.** `ATMMachine` forwards every call to the current `ATMState`. Every decision lives inside the state object.
2. **Default methods cut boilerplate.** Each state overrides only what it supports; the rest falls back to the interface's default "not allowed" message. `IdleState` overrides only `insertCard`, yet all four states still support all six actions.
3. **`CardInsertedState`** handles `enterPin`. A correct PIN reads the account ID off the card and moves to `AuthenticatedState`. A wrong PIN decrements the attempt counter; at zero, the card is retained.
4. **`AuthenticatedState`** handles `withdraw`, `deposit`, and `checkBalance`, staying in itself after the last two so several operations can run per session. `withdraw` only checks the amount shape, then hands off to `DispensingCashState`.
5. **`DispensingCashState` runs four steps:** a balance check, a note plan from `CashInventory` **before** touching the bank account, the one atomic `bankService.debit(...)`, then removing the notes. Checking the plan before the debit means a cash-shortage failure never costs a debit; a rare last-step failure is fixed with a compensating credit.
6. **The greedy failure, worked out.** Before the first withdrawal, inventory is `{HUNDRED: 5, FIFTY: 1, TWENTY: 3}`. A request for **$60** makes `GreedyNoteDispenser` take the one `FIFTY` first, leaving $10 that no remaining note can cover (`TWENTY` needs a full $20; no `TEN` notes exist). It throws `InsufficientCashException`, even though **three $20 notes sum to exactly $60**. Greedy fails because it commits to the biggest note first and never reconsiders. No debit happens for this attempt.
7. **The next withdrawal, $40, succeeds**: greedy picks two `TWENTY` notes. The account drops to 460, one `TWENTY` note remains.
8. **Deposit** adds notes straight into `CashInventory` and credits the same total. It skips `NoteDispenser`, since a deposit just counts what came in.

## How to Extend (Follow-ups)

- **Fix the greedy weakness.** Use a backtracking/DP dispenser: if `k` notes of a denomination lead to a dead end, retry with fewer before moving on. This finds a valid combination whenever one exists, with no change to any state class, since `NoteDispenser` is a Strategy.
- **Multi-account cards:** add an `AccountSelectionState` between `CardInsertedState` and `AuthenticatedState`; other states do not change.
- **Card retention as a real state:** a `CardRetainedState` blocking all actions except an admin "release" call, instead of the inline retain-and-go-Idle logic.
- **Bill-counting on deposit:** a real bill acceptor counts notes as inserted; it just hands a `Map<Denomination, Integer>` to the existing `deposit` method.
- **Idempotent retries:** add a transaction ID per withdrawal and store completed IDs on the bank side, so a resent request cannot debit twice.
- **Receipts/audit logs:** an `Observer` (`TransactionListener`) reacts to completed transactions without touching state classes.
- **Timeouts:** an unattended session can call the existing `ejectCard()` via a `ScheduledExecutorService`; no new state needed.

## Complexity & Thread-Safety Notes

- **Time/space:** every state transition is O(1). `GreedyNoteDispenser.computeNotes` is O(d log d) for sorting plus O(d) to scan, where `d` is the small, fixed number of denominations. State objects are singletons, so no memory is allocated per transaction.
- **Thread-safety, two layers.** Per machine: `ATMMachine` wraps every call in a `ReentrantLock`, so one machine handles one session step at a time, matching one card slot and keypad. Per account, across machines: `InMemoryBankService.debit` wraps check-then-subtract inside `synchronized (account)`. Without this, two ATMs reading the same balance at once could both see "enough funds" and both subtract, driving it negative. The lock is per-account, so unrelated accounts never block each other.
- **Cash inventory** uses one `ReentrantLock` across `previewWithdrawPlan` and `commitWithdrawPlan`. The debit happens between these calls, outside that lock, leaving a narrow window where a concurrent admin refill could change the inventory; `commitWithdrawPlan` detects this and the caller compensates with a credit.
- `state`, `currentCard`, and `currentAccountId` are `volatile`, so a monitoring thread can read them without the lock, while writes still happen under it.

## Interview Tips & Common Mistakes

- **State the reasoning before coding:** "valid actions depend on where the ATM is in the session, so each stage is its own class, not a status flag with a long if/else." This shows *why*, not just *that*, you picked the pattern.
- **The most ATM-specific mistake: debiting before confirming cash can be dispensed.** If `debit()` runs first and the note plan then fails, the account is short with no cash out. Check dispensability first, or explain a compensating credit.
- **Do not skip the atomic debit.** A common bug is `getBalance()` then a separate `setBalance()`, with no lock between: a check-then-act race, where two concurrent withdrawals can both pass the check before either subtracts.
- **Have the greedy-failure example ready.** Candidates often assume greedy always works for round denominations; it does not once note *counts* are limited. The `{FIFTY: 1, TWENTY: 3}` case for $60 is a clean, memorable proof.
- **Use singletons for state objects,** since they hold no per-transaction data; do not `new` one on every transition.
- **Mention the default-method trick:** it removes repetitive "reject" overrides while keeping compiler-checked coverage of every action in every state.
- **If asked about a bank call timing out mid-transaction,** mention logging it "pending" before calling the bank and reconciling later by transaction ID, instead of assuming it only ever fully succeeds or fails.
