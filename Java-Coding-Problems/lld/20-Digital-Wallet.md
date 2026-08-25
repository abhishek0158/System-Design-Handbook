# Digital Wallet (LLD)

## Problem

Design a digital wallet system, like the wallet inside a payment app. Each user has one wallet, and a wallet holds a balance of money.

The system must support:

- Creating a user and a wallet for that user.
- Adding money to a wallet (credit), for example from a linked bank card.
- Withdrawing money from a wallet (debit).
- Transferring money from one wallet to another.
- A record of every change to a wallet, for audits (an audit is a check of past activity).
- No overdraft: a wallet balance can never go negative.

This sounds simple. The hard parts show up under concurrency (many operations at the same time) and under retries (a client sends the same request twice). Interviewers focus there, not on basic CRUD (create, read, update, delete) code.

## Requirements & Clarifying Questions

1. **How is money stored?** Never as `double`. We use `long` minor units (for example, paise or cents) or `BigDecimal`. Explained below.
2. **Must a transfer be atomic?** Yes. Atomic means: either both wallets change, or neither changes.
3. **What about two transfers between the same two wallets at the same time?** For example, A sends money to B, and at the same moment B sends money to A. Locking in the wrong order here causes a deadlock (two threads stuck, each waiting for a lock the other holds). We must prevent this.
4. **What if a client retries a transfer request?** A network timeout might make a client resend the same transfer. The system must not charge the sender twice — this is idempotency, where repeating a request has the same effect as doing it once.
5. **Can a wallet go negative?** No. Every debit (money leaving a wallet) must check the balance first.
6. **Single-JVM or distributed?** This sheet is single-JVM, in-memory, with notes on moving to a real database.
7. **Multiple currencies?** Out of scope; one currency per wallet, noted as a follow-up.

This sheet assumes single JVM, in-memory storage, long minor units for money, and one currency.

## Design / Approach

```
+--------+        +-----------------------+
|  User  |        |    WalletService      |
+--------+        |       (Facade)        |
                  +-----------+-----------+
                        |            |
             locks &    |            |  records
             mutates    v            v
                  +----------+  +----------------+
                  |  Wallet  |  |     Ledger      |
                  | balance  |  | (append-only)   |
                  | lock     |  +--------+--------+
                  +----------+           |
                                         v
                                  +--------------+
                                  | Transaction  |
                                  | (immutable)  |
                                  +--------------+

  +--------------------------+
  |  IdempotencyStore        |
  |  requestId -> Transaction|
  +--------------------------+
```

Patterns and principles used:

- **Facade (`WalletService`)** — the only entry point client code calls. It hides wallet locking, ledger writes, and idempotency checks. This matches **Single Responsibility Principle (SRP)**: `Wallet` only holds a balance and a lock, `Ledger` only stores records, `WalletService` only coordinates them.
- **Immutable value objects (`Transaction`)** — once built, a `Transaction` never changes, so many threads can read it safely with no lock needed.
- **Dependency Inversion Principle (DIP)** — `WalletService` depends on a `TransactionLedger` interface, not a concrete class, so swapping in-memory storage for a database later needs no change to `WalletService`.
- **Guarded critical section with ordered locking** — each `Wallet` owns a `ReentrantLock` (a lock object one thread can hold at a time). A transfer locks both wallets involved, always in the same order (by wallet ID), to avoid deadlock. Details below.
- **Idempotency via a request ID cache** — every call carries a client-supplied request ID. The service remembers results by that ID, so a retried request returns the same result instead of running twice.

**Why not `double` for money.** `double` is binary floating-point and cannot store many decimal values exactly — `0.1 + 0.2` in Java gives `0.30000000000000004`, not `0.3`. Over many transactions these small errors compound. We store money as a `long` count of minor units (`100 rupees` becomes `10000` paise), so all math is exact integer math. `BigDecimal` is used only when parsing input like `"100.50"` into minor units.

**Why atomicity needs two locks.** A transfer changes two wallets. Locking only the sender's wallet would let another thread read the receiver's wallet mid-update. Both locks must be held for the whole operation.

**Why lock order prevents deadlock.** If thread 1 runs "transfer A to B" and thread 2 runs "transfer B to A" at the same time, and each locks its sender first, thread 1 holds A waiting for B while thread 2 holds B waiting for A — a deadlock. The fix: lock wallets in a fixed order, for example ascending wallet ID, regardless of who sends and who receives. Both threads then compete for A first; the winner finishes and releases both locks before the other proceeds. No cycle of waiting, so no deadlock.

## Java Solution

### Domain Model: User and Wallet

```java
import java.time.Instant;
import java.util.*;
import java.util.concurrent.*;
import java.util.concurrent.locks.ReentrantLock;

public final class User {
    private final long id;
    private final String name;

    public User(long id, String name) { this.id = id; this.name = name; }
    public long getId() { return id; }
    public String getName() { return name; }

    @Override
    public boolean equals(Object o) {
        return (o instanceof User) && id == ((User) o).id;
    }
    @Override
    public int hashCode() { return Long.hashCode(id); }
}

// Balance in minor units (a long), never double. Each wallet owns its own lock.
public final class Wallet {
    private final long id;
    private final long userId;
    private long balance; // minor units; guarded by "lock"
    private final ReentrantLock lock = new ReentrantLock();

    public Wallet(long id, long userId, long openingBalance) {
        this.id = id;
        this.userId = userId;
        this.balance = openingBalance;
    }

    public long getId() { return id; }
    public long getUserId() { return userId; }

    public void lock() { lock.lock(); }
    public void unlock() { lock.unlock(); }

    // Callers must hold "lock" (via lock()/unlock()) before calling these three methods.
    public long getBalanceUnsafe() { return balance; }

    public void creditUnsafe(long amount) {
        balance += amount;
    }

    public void debitUnsafe(long amount) {
        if (amount > balance) {
            throw new InsufficientBalanceException("Wallet " + id + " has insufficient balance");
        }
        balance -= amount;
    }
}
```

`Wallet` does not lock itself inside `creditUnsafe`/`debitUnsafe` — the caller (`WalletService`) locks first. This lets a single-wallet operation take one lock, while a transfer holds two, using the same lock object with no risk of double-locking.

### Transaction (Immutable Ledger Entry)

```java
public enum TransactionType { CREDIT, DEBIT, TRANSFER_OUT, TRANSFER_IN }

public final class Transaction {
    private final String transactionId;   // unique ID for this ledger entry
    private final String requestId;       // client-supplied ID, used for idempotency
    private final Long fromWalletId;      // null for a pure credit
    private final Long toWalletId;        // null for a pure debit
    private final long amount;            // minor units, always positive
    private final TransactionType type;
    private final Instant timestamp;

    public Transaction(String transactionId, String requestId, Long fromWalletId, Long toWalletId,
                        long amount, TransactionType type, Instant timestamp) {
        this.transactionId = transactionId;
        this.requestId = requestId;
        this.fromWalletId = fromWalletId;
        this.toWalletId = toWalletId;
        this.amount = amount;
        this.type = type;
        this.timestamp = timestamp;
    }

    public String getTransactionId() { return transactionId; }
    public String getRequestId() { return requestId; }
    public Long getFromWalletId() { return fromWalletId; }
    public Long getToWalletId() { return toWalletId; }
    public long getAmount() { return amount; }
    public TransactionType getType() { return type; }
    public Instant getTimestamp() { return timestamp; }
}
```

Every field is `final` with no setter. Once created and stored, no code path can edit a `Transaction` — this is what makes the ledger trustworthy for audits.

### Ledger (Append-Only Store)

```java
public interface TransactionLedger {
    void record(Transaction transaction);
    List<Transaction> getHistory(long walletId);
}

public final class InMemoryLedger implements TransactionLedger {
    // One append-only list per wallet.
    private final ConcurrentHashMap<Long, List<Transaction>> historyByWallet = new ConcurrentHashMap<>();

    @Override
    public void record(Transaction transaction) {
        if (transaction.getFromWalletId() != null) {
            addTo(transaction.getFromWalletId(), transaction);
        }
        if (transaction.getToWalletId() != null) {
            addTo(transaction.getToWalletId(), transaction);
        }
    }

    private void addTo(long walletId, Transaction transaction) {
        historyByWallet
            .computeIfAbsent(walletId, id -> new CopyOnWriteArrayList<>())
            .add(transaction);
    }

    @Override
    public List<Transaction> getHistory(long walletId) {
        return Collections.unmodifiableList(
            historyByWallet.getOrDefault(walletId, Collections.emptyList()));
    }
}
```

### Exceptions

```java
public class InsufficientBalanceException extends RuntimeException {
    public InsufficientBalanceException(String message) { super(message); }
}

public class WalletNotFoundException extends RuntimeException {
    public WalletNotFoundException(String message) { super(message); }
}
```

### WalletService (Facade + Idempotency + Ordered Locking)

```java
public final class WalletService {
    private final ConcurrentHashMap<Long, Wallet> wallets = new ConcurrentHashMap<>();
    private final TransactionLedger ledger;

    // Idempotency: remembers the result per request ID, so a retry returns
    // the same Transaction instead of moving money again. A lock per request
    // ID keeps two threads racing the SAME request ID from double-processing,
    // while different request IDs never block each other.
    private final ConcurrentHashMap<String, Transaction> idempotencyStore = new ConcurrentHashMap<>();
    private final ConcurrentHashMap<String, Object> requestLocks = new ConcurrentHashMap<>();

    public WalletService(TransactionLedger ledger) { this.ledger = ledger; }

    public Wallet openWallet(long walletId, long userId, long openingBalance) {
        Wallet wallet = new Wallet(walletId, userId, openingBalance);
        wallets.put(walletId, wallet);
        return wallet;
    }

    private Wallet getWallet(long walletId) {
        Wallet wallet = wallets.get(walletId);
        if (wallet == null) throw new WalletNotFoundException("No wallet with id " + walletId);
        return wallet;
    }

    // ---- Idempotency wrapper: shared by credit, withdraw, and transfer ----
    private Transaction withIdempotency(String requestId, Supplier<Transaction> action) {
        Transaction existing = idempotencyStore.get(requestId);
        if (existing != null) return existing; // fast path: already processed

        Object requestLock = requestLocks.computeIfAbsent(requestId, id -> new Object());
        synchronized (requestLock) {
            existing = idempotencyStore.get(requestId); // re-check inside the lock
            if (existing != null) return existing;

            Transaction result = action.get();
            idempotencyStore.put(requestId, result);
            return result;
        }
    }

    public Transaction addMoney(long walletId, long amount, String requestId) {
        if (amount <= 0) throw new IllegalArgumentException("Amount must be positive");
        return withIdempotency(requestId, () -> {
            Wallet wallet = getWallet(walletId);
            wallet.lock();
            try {
                wallet.creditUnsafe(amount);
                Transaction txn = new Transaction(UUID.randomUUID().toString(), requestId,
                        null, walletId, amount, TransactionType.CREDIT, Instant.now());
                ledger.record(txn);
                return txn;
            } finally {
                wallet.unlock();
            }
        });
    }

    public Transaction withdraw(long walletId, long amount, String requestId) {
        if (amount <= 0) throw new IllegalArgumentException("Amount must be positive");
        return withIdempotency(requestId, () -> {
            Wallet wallet = getWallet(walletId);
            wallet.lock();
            try {
                wallet.debitUnsafe(amount); // throws InsufficientBalanceException if too little
                Transaction txn = new Transaction(UUID.randomUUID().toString(), requestId,
                        walletId, null, amount, TransactionType.DEBIT, Instant.now());
                ledger.record(txn);
                return txn;
            } finally {
                wallet.unlock();
            }
        });
    }

    public Transaction transfer(long fromWalletId, long toWalletId, long amount, String requestId) {
        if (fromWalletId == toWalletId) throw new IllegalArgumentException("Cannot transfer to the same wallet");
        if (amount <= 0) throw new IllegalArgumentException("Amount must be positive");

        return withIdempotency(requestId, () -> doTransfer(fromWalletId, toWalletId, amount, requestId));
    }

    private Transaction doTransfer(long fromWalletId, long toWalletId, long amount, String requestId) {
        Wallet from = getWallet(fromWalletId);
        Wallet to = getWallet(toWalletId);

        // Fixed lock order by wallet ID avoids deadlock: a concurrent "A to B" and
        // "B to A" transfer both try to lock the lower ID first, so neither thread
        // can hold one lock while waiting forever for the other.
        Wallet first = (from.getId() < to.getId()) ? from : to;
        Wallet second = (from.getId() < to.getId()) ? to : from;

        first.lock();
        try {
            second.lock();
            try {
                if (from.getBalanceUnsafe() < amount) {
                    throw new InsufficientBalanceException("Wallet " + fromWalletId + " has insufficient balance");
                }
                // Both mutations happen while both locks are held: either both
                // succeed, or (on the balance check above) neither runs.
                from.debitUnsafe(amount);
                to.creditUnsafe(amount);

                Transaction txn = new Transaction(UUID.randomUUID().toString(), requestId,
                        fromWalletId, toWalletId, amount, TransactionType.TRANSFER_OUT, Instant.now());
                ledger.record(txn);
                return txn;
            } finally {
                second.unlock();
            }
        } finally {
            first.unlock();
        }
    }

    public List<Transaction> getHistory(long walletId) {
        return ledger.getHistory(walletId);
    }

    public long getBalance(long walletId) {
        Wallet wallet = getWallet(walletId);
        wallet.lock();
        try {
            return wallet.getBalanceUnsafe();
        } finally {
            wallet.unlock();
        }
    }
}
```

`Supplier<Transaction>` above is `java.util.function.Supplier`; add `import java.util.function.Supplier;` at the top of the file.

## How It Works

**Adding / withdrawing money.** `addMoney` locks one wallet, updates the balance, writes a `CREDIT` transaction, and unlocks. `withdraw` does the same but calls `debitUnsafe`, which checks `amount > balance` first and throws `InsufficientBalanceException` without touching `balance` if the check fails — the "no overdraft" rule.

**Transfer, step by step.** Say wallet 5 sends 200 to wallet 9. `transfer` calls `withIdempotency`, which finds no prior result for this `requestId` and runs the action. `doTransfer` picks the lower ID, wallet 5, to lock first, then wallet 9. With both locks held, it checks wallet 5 has at least 200, then debits wallet 5 and credits wallet 9 before either lock is released — so no thread ever observes money that left wallet 5 but has not yet reached wallet 9. One `Transaction` is written and returned; locks release in reverse order.

**Why this avoids deadlock.** If another thread runs "wallet 9 sends 50 to wallet 5" at the same time, it also computes `first = wallet 5`, because "lock the lower ID first" does not depend on who is the sender. Both threads compete for wallet 5's lock first; the winner finishes and releases both locks before the other proceeds. Neither thread ever holds one lock while waiting on a lock the other holds, so deadlock cannot happen.

**Why retries are safe.** If the client resends `transfer(5, 9, 200, "req-123")` after a lost response, `idempotencyStore.get("req-123")` now finds the stored `Transaction` and returns it immediately — `doTransfer` never runs again, and the sender is not charged twice. `synchronized (requestLock)` also protects two threads submitting the *same* `requestId` at the *same instant*: the second thread waits, then reads the stored result instead of racing to run `doTransfer` twice.

## How to Extend (Follow-ups)

- **Transaction history with filters.** `getHistory(walletId)` returns every `Transaction` for a wallet. Add filtering by date range or `TransactionType`, and pagination, so one call does not return millions of rows.
- **Reversal / refund.** Model a reversal as a **new** `Transaction`, never an edit of the old one. A refund of transfer `T1` is a new transfer from receiver back to sender, tagged with a `reversalOf` field pointing at `T1`'s ID — the trail shows both the charge and the refund.
- **Idempotency store across restarts.** The `ConcurrentHashMap` here lives only in one JVM's memory. A real system stores `requestId` in the database with a **unique constraint** (a rule rejecting a second row with the same value), so a duplicate insert fails at the database level — surviving a restart or a request landing on a different server.
- **Database transaction, in practice.** The database enforces atomicity instead of Java locks: begin a transaction, `SELECT balance FROM wallets WHERE id = ? FOR UPDATE` on both wallets (lower ID first, same ordering idea, now via SQL row locks), check the balance, run two `UPDATE` statements, insert a row into `transactions` with a unique `request_id` column, then commit. Any failure rolls the whole transaction back.
- **Multi-currency wallets.** Add a `Currency` field to `Wallet` and reject transfers between mismatched currencies, or add a conversion step.
- **Notifications.** Add an Observer hook after `ledger.record(txn)` to notify listeners, without changing `WalletService` when a new listener is added — Open/Closed Principle in action.

## Complexity & Thread-Safety Notes

| Operation | Time | Notes |
|---|---|---|
| `addMoney` / `withdraw` | O(1) | one wallet lock, one ledger append |
| `transfer` | O(1) | two wallet locks (fixed order), one ledger append |
| `getHistory(walletId)` | O(k) | k = transactions for that wallet |
| `getBalance(walletId)` | O(1) | one wallet lock |
| Idempotency check | O(1) amortized | one `ConcurrentHashMap` lookup, fast path |

**Thread-safety:**
- Each `Wallet` has its own `ReentrantLock`. Operations on unrelated wallets never block each other — only operations touching the *same* wallet (or pair) contend for a lock.
- Fixed lock ordering by wallet ID removes the possibility of a circular wait between two wallets — the standard cause of deadlock.
- `Transaction` objects are immutable, so `getHistory` returns live references with no copying and no risk a reader sees a half-written object.
- `InMemoryLedger` uses `CopyOnWriteArrayList` per wallet: cheap concurrent reads, costlier writes (each write copies the array) — a good fit since history is read far more than it is written.
- The idempotency map plus a per-request lock gives "exactly-once effect" only within one JVM — weaker than a database unique constraint, which survives restarts and works across servers. Mention this trade-off proactively.

## Interview Tips & Common Mistakes

- **Never use `double` or `float` for money.** Say this early: binary floating-point cannot exactly represent most decimal fractions, and the error grows across many operations. Use `long` minor units or `BigDecimal` with a fixed scale.
- **Back up "atomic" with code.** Show that `debitUnsafe` and `creditUnsafe` run while both locks are held, and that the balance check happens before any mutation — so a failed check leaves both wallets untouched.
- **Explain lock ordering, not just "we use locks."** Locking wallets in sender-then-receiver order causes the exact deadlock interviewers test for. Locking by a fixed, sender-independent order (such as wallet ID) is the key insight they want to hear.
- **Do not forget idempotency.** A transfer API called over a network will eventually be retried after a timeout. With no request ID and no idempotency store, a retry double-charges the sender. Raise this even if not asked.
- **Keep the ledger append-only.** Never update or delete a `Transaction`. Model corrections, like refunds, as new transactions that reference the old one — this is what makes a ledger auditable.
- **Do not let `Wallet` lock itself inside `credit`/`debit`.** If each method locked internally, a transfer could not hold both wallets locked across both operations, breaking atomicity. The caller (`WalletService`) must control locking, since only it knows both wallets involved.
