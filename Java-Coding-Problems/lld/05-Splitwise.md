# Splitwise (Expense Splitter) (LLD)

## Problem

Design a simplified Splitwise. Splitwise is an app that helps a group share expenses. One person pays a bill. The app splits the cost among the group. Over time, it tracks who owes money to whom.

The system must support:

- Users and groups (a group has many users; a user can be in many groups).
- Adding an expense, paid by one or more users, split among members using different rules.
- A balance sheet that tracks, for every pair of users, who owes whom and how much.
- Showing balances for one user.
- Settling up (recording a payment between two users).
- Debt simplification: reducing a chain like "A owes B, B owes C" toward fewer direct payments.

## Requirements & Clarifying Questions

1. **Which split types?** Three are required: Equal, Exact (fixed amount per person), Percentage. Design so a fourth (e.g. split by shares) is easy to add later.
2. **Can more than one person pay one expense?** Yes — e.g. A pays 300 and B pays 200 of a 500 bill. This affects how we compute who owes whom.
3. **How precise must money be?** Must support minor units (paise/cents), not just whole currency units.
4. **Do balances need a group, or can two friends owe each other directly?** Both, so the balance sheet must not depend on a group ID.
5. **Can settle-up be partial?** Yes.
6. **How exact must debt simplification be?** True minimum transactions is a hard problem; we use a fast greedy method, not a perfect minimizer.
7. **Single JVM or distributed?** Single-JVM, in-memory here, with notes on moving to a database.

This sheet assumes single JVM, in-memory storage, multiple payers per expense, and greedy debt simplification.

## Design / Approach

```
+--------+       +-------+        +------------------+
|  User  |<>-----| Group |        |  ExpenseManager  |
+--------+       +-------+        |     (Facade)      |
                                  +--------+----------+
                                           |
                    creates / applies      |
                +--------------------------+-------------------+
                v                                              v
         +-------------+                                +---------------+
         |   Expense   |                                |  BalanceSheet |
         +------+------+                                +-------+-------+
                |  uses (Strategy pattern)                       ^
                v                                                |
         +----------------+   built by SplitStrategyFactory      |
         | SplitStrategy  |<--------------------------------------+
         +----------------+                                       |
         /       |        \                                       |
  EqualSplit ExactSplit PercentSplit --> List<Split> --> net/user --> DebtSimplifier (greedy)
```

Patterns and principles used:

- **Strategy (`SplitStrategy`)** — each split rule is its own class behind one interface. A new rule means one new class, no edits to existing code (**Open/Closed Principle**), with a **Factory** (`SplitStrategyFactory`) picking the right one for a given `SplitType`.
- **Facade (`ExpenseManager`)** — the rest of the app talks only to `ExpenseManager`, which hides `BalanceSheet` and `DebtSimplifier`.
- **Single Responsibility** — `BalanceSheet` only stores balances, `DebtSimplifier` only computes minimal transactions, each strategy only computes shares.
- **Immutable value objects** (`Split`, `Transaction`, `Expense`) — safe to read from many threads with no locking.

**Money as minor units, not `double`:** every amount is a `long` in the smallest currency unit (paise/cents). `double` uses binary floating-point, which cannot represent many decimals exactly (`0.1 + 0.2 != 0.3`) — unacceptable for money. `long` gives exact integer math if remainders are handled with care (shown below). `BigDecimal` is used only for percentages (not money) and at the input boundary, where `"12.50"` is parsed into `1250` minor units once.

## Java Solution

### Domain Model

```java
import java.util.*;
import java.util.concurrent.*;
import java.math.BigDecimal;
import java.math.RoundingMode;

public class User {
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

// Group only groups Users for display; balances always key on User pairs, not group IDs.
public class Group {
    private final List<User> members = new ArrayList<>();
    public void addMember(User user) { members.add(user); }
}
```

### Split and Expense

`Split` is how much one user owes for one expense; `Expense` is immutable once built.

```java
public final class Split {
    private final User user;
    private final long amountOwed; // minor units

    public Split(User user, long amountOwed) { this.user = user; this.amountOwed = amountOwed; }
    public User getUser() { return user; }
    public long getAmountOwed() { return amountOwed; }
}

public enum SplitType { EQUAL, EXACT, PERCENT }

public final class Expense {
    private final String id;
    private final long totalAmount;         // minor units
    private final Map<User, Long> paidBy;    // who paid how much
    private final List<Split> splits;        // who owes how much

    public Expense(String id, long totalAmount, Map<User, Long> paidBy, List<Split> splits) {
        this.id = id;
        this.totalAmount = totalAmount;
        this.paidBy = Collections.unmodifiableMap(new LinkedHashMap<>(paidBy));
        this.splits = Collections.unmodifiableList(new ArrayList<>(splits));
    }
    public long getTotalAmount() { return totalAmount; }
    public Map<User, Long> getPaidBy() { return paidBy; }
    public List<Split> getSplits() { return splits; }
}
```

### Split Strategy (Strategy Pattern)

```java
public interface SplitStrategy {
    // inputValues: ignored for EQUAL; exact minor-unit amount per user for EXACT;
    // percentage per user (e.g. 33.33) for PERCENT.
    List<Split> calculateSplits(long totalAmount, List<User> participants, Map<User, BigDecimal> inputValues);
}
```

**Equal split.** Integer division can leave a remainder (100 split 3 ways is 33.33... each). We give the leftover minor units, one each, to the first few participants, so the sum matches the total exactly.

```java
public class EqualSplitStrategy implements SplitStrategy {
    @Override
    public List<Split> calculateSplits(long totalAmount, List<User> participants, Map<User, BigDecimal> inputValues) {
        int n = participants.size();
        if (n == 0) throw new IllegalArgumentException("Expense must have at least one participant");
        long baseShare = totalAmount / n;
        long remainder = totalAmount % n;

        List<Split> splits = new ArrayList<>();
        for (int i = 0; i < n; i++) {
            long share = baseShare + (i < remainder ? 1 : 0); // spread leftover minor units
            splits.add(new Split(participants.get(i), share));
        }
        return splits;
    }
}
```

**Exact split.** Every participant needs an exact amount; amounts must sum to the total, or the expense is rejected.

```java
public class ExactSplitStrategy implements SplitStrategy {
    @Override
    public List<Split> calculateSplits(long totalAmount, List<User> participants, Map<User, BigDecimal> inputValues) {
        List<Split> splits = new ArrayList<>();
        long sum = 0;
        for (User user : participants) {
            BigDecimal amount = inputValues.get(user);
            if (amount == null) throw new IllegalArgumentException("Missing exact amount for " + user.getName());
            long minorUnits = amount.longValueExact();
            sum += minorUnits;
            splits.add(new Split(user, minorUnits));
        }
        if (sum != totalAmount) {
            throw new IllegalArgumentException("Exact amounts sum to " + sum + " but total is " + totalAmount);
        }
        return splits;
    }
}
```

**Percentage split.** Percentages must sum to 100 (small tolerance allowed). The last participant absorbs the rounding remainder, so no minor units are lost or invented.

```java
public class PercentageSplitStrategy implements SplitStrategy {
    private static final BigDecimal HUNDRED = BigDecimal.valueOf(100);
    private static final BigDecimal TOLERANCE = new BigDecimal("0.01");

    @Override
    public List<Split> calculateSplits(long totalAmount, List<User> participants, Map<User, BigDecimal> inputValues) {
        BigDecimal percentSum = BigDecimal.ZERO;
        for (User user : participants) {
            BigDecimal pct = inputValues.get(user);
            if (pct == null) throw new IllegalArgumentException("Missing percentage for " + user.getName());
            percentSum = percentSum.add(pct);
        }
        if (percentSum.subtract(HUNDRED).abs().compareTo(TOLERANCE) > 0) {
            throw new IllegalArgumentException("Percentages must sum to 100, got " + percentSum);
        }

        List<Split> splits = new ArrayList<>();
        long allocated = 0;
        for (int i = 0; i < participants.size(); i++) {
            User user = participants.get(i);
            long share;
            if (i == participants.size() - 1) {
                share = totalAmount - allocated; // last user absorbs rounding remainder
            } else {
                share = BigDecimal.valueOf(totalAmount).multiply(inputValues.get(user))
                        .divide(HUNDRED, 0, RoundingMode.HALF_UP).longValueExact();
                allocated += share;
            }
            splits.add(new Split(user, share));
        }
        return splits;
    }
}

public class SplitStrategyFactory {
    public static SplitStrategy getStrategy(SplitType type) {
        switch (type) {
            case EQUAL:   return new EqualSplitStrategy();
            case EXACT:   return new ExactSplitStrategy();
            case PERCENT: return new PercentageSplitStrategy();
            default: throw new IllegalArgumentException("Unknown split type: " + type);
        }
    }
}
```

### Balance Sheet

One signed number per unordered pair of users (`UserPair`) — a relationship is never stored twice or allowed to contradict itself.

```java
public final class UserPair {
    private final User first;  // always the smaller id
    private final User second;

    private UserPair(User first, User second) { this.first = first; this.second = second; }

    public static UserPair of(User a, User b) {
        return (a.getId() < b.getId()) ? new UserPair(a, b) : new UserPair(b, a);
    }
    public User getFirst() { return first; }
    public User getSecond() { return second; }

    @Override
    public boolean equals(Object o) {
        if (!(o instanceof UserPair)) return false;
        UserPair p = (UserPair) o;
        return first.equals(p.first) && second.equals(p.second);
    }
    @Override
    public int hashCode() { return Objects.hash(first, second); }
}

public class BalanceSheet {
    // positive: pair.first owes pair.second; negative: pair.second owes pair.first
    private final ConcurrentHashMap<UserPair, Long> balances = new ConcurrentHashMap<>();

    public void addDebt(User debtor, User creditor, long amount) {
        if (debtor.equals(creditor) || amount <= 0) return;
        UserPair pair = UserPair.of(debtor, creditor);
        long signed = (debtor.getId() < creditor.getId()) ? amount : -amount;
        balances.compute(pair, (k, existing) -> {
            long updated = (existing == null ? 0 : existing) + signed;
            return updated == 0 ? null : updated; // fully settled -> remove entry
        });
    }

    public List<String> getBalancesForUser(User user) {
        List<String> lines = new ArrayList<>();
        for (Map.Entry<UserPair, Long> e : balances.entrySet()) {
            UserPair pair = e.getKey();
            long value = e.getValue();
            if (!pair.getFirst().equals(user) && !pair.getSecond().equals(user)) continue;
            User owes = value > 0 ? pair.getFirst() : pair.getSecond();
            User owed = value > 0 ? pair.getSecond() : pair.getFirst();
            lines.add(owes.getName() + " owes " + owed.getName() + ": " + Math.abs(value));
        }
        return lines;
    }

    public Map<User, Long> getNetBalances(List<User> members) {
        // positive: this user is owed money overall; negative: this user owes money overall
        Map<User, Long> net = new HashMap<>();
        for (User user : members) net.put(user, 0L);
        for (Map.Entry<UserPair, Long> e : balances.entrySet()) {
            UserPair pair = e.getKey();
            long value = e.getValue(); // first owes second, if positive
            net.merge(pair.getFirst(), -value, Long::sum);
            net.merge(pair.getSecond(), value, Long::sum);
        }
        return net;
    }
}
```

### Debt Simplification

Take each user's **net position** (positive = owed overall, negative = owes overall). Repeatedly match the biggest debtor with the biggest creditor and settle the smaller of the two amounts. This **greedy algorithm** does not always find the mathematically smallest number of transactions — that exact problem is related to a known NP-hard problem when balances can be split across subsets — but it is fast (`O(n log n)`) and works well in practice. Say this trade-off out loud in an interview.

```java
public final class Transaction {
    private final User debtor, creditor;
    private final long amount;

    public Transaction(User debtor, User creditor, long amount) {
        this.debtor = debtor; this.creditor = creditor; this.amount = amount;
    }
    public User getDebtor() { return debtor; }
    public User getCreditor() { return creditor; }
    public long getAmount() { return amount; }
}

public class DebtSimplifier {
    private static final class Balance {
        final User user; long amount;
        Balance(User user, long amount) { this.user = user; this.amount = amount; }
    }

    public static List<Transaction> simplify(Map<User, Long> netBalances) {
        List<Balance> creditors = new ArrayList<>();
        List<Balance> debtors = new ArrayList<>();
        for (Map.Entry<User, Long> e : netBalances.entrySet()) {
            long v = e.getValue();
            if (v > 0) creditors.add(new Balance(e.getKey(), v));
            else if (v < 0) debtors.add(new Balance(e.getKey(), -v));
        }
        creditors.sort((a, b) -> Long.compare(b.amount, a.amount));
        debtors.sort((a, b) -> Long.compare(b.amount, a.amount));

        List<Transaction> transactions = new ArrayList<>();
        int i = 0, j = 0;
        while (i < debtors.size() && j < creditors.size()) {
            Balance debtor = debtors.get(i), creditor = creditors.get(j);
            long settled = Math.min(debtor.amount, creditor.amount);
            transactions.add(new Transaction(debtor.user, creditor.user, settled));
            debtor.amount -= settled;
            creditor.amount -= settled;
            if (debtor.amount == 0) i++;
            if (creditor.amount == 0) j++;
        }
        return transactions;
    }
}
```

### Expense Manager (Facade)

Single entry point: validates, builds splits, folds the expense into the balance sheet. `DebtSimplifier` is reused twice — once per expense (many payers/owers to a few direct debts) and once for a whole group (a "settle up now" plan).

```java
public class ExpenseManager {
    private final BalanceSheet balanceSheet = new BalanceSheet();
    private final List<Expense> expenses = new CopyOnWriteArrayList<>();

    public Expense addExpense(long totalAmount, Map<User, Long> paidBy, List<User> participants,
                               SplitType splitType, Map<User, BigDecimal> splitInputs) {
        long totalPaid = paidBy.values().stream().mapToLong(Long::longValue).sum();
        if (totalPaid != totalAmount) {
            throw new IllegalArgumentException("Amount paid (" + totalPaid + ") != total (" + totalAmount + ")");
        }
        SplitStrategy strategy = SplitStrategyFactory.getStrategy(splitType);
        List<Split> splits = strategy.calculateSplits(totalAmount, participants, splitInputs);

        Expense expense = new Expense(UUID.randomUUID().toString(), totalAmount, paidBy, splits);
        expenses.add(expense);
        applyToBalanceSheet(expense);
        return expense;
    }

    private void applyToBalanceSheet(Expense expense) {
        // net = amount paid minus amount owed, per participant, for this one expense
        Map<User, Long> net = new HashMap<>();
        for (Split split : expense.getSplits()) net.merge(split.getUser(), -split.getAmountOwed(), Long::sum);
        for (Map.Entry<User, Long> paid : expense.getPaidBy().entrySet()) net.merge(paid.getKey(), paid.getValue(), Long::sum);
        for (Transaction t : DebtSimplifier.simplify(net)) {
            balanceSheet.addDebt(t.getDebtor(), t.getCreditor(), t.getAmount());
        }
    }

    public void settleUp(User payer, User payee, long amount) {
        balanceSheet.addDebt(payee, payer, amount); // payer clears part of what they owe payee
    }

    public List<String> getBalancesForUser(User user) { return balanceSheet.getBalancesForUser(user); }

    public List<Transaction> simplifyGroupDebts(List<User> members) {
        return DebtSimplifier.simplify(balanceSheet.getNetBalances(members));
    }
}
```

## How It Works

**Single payer, equal split.** A pays 900 for dinner with A, B, C. Each owes 300. The per-expense net is A `+600`, B `-300`, C `-300`. `DebtSimplifier` turns this into two transactions: B owes A 300, C owes A 300, written via `addDebt`.

**Multi-payer expense.** A pays 300 and B pays 200 of a 500 bill split equally among A, B, C. The net per participant is A `+133`, B `+33`, C `-166`. Rather than hand-tracking "C owes both A and B," `applyToBalanceSheet` runs these net positions through `DebtSimplifier` — same "net positions to minimal payments" problem, just scoped to one expense — which greedily matches C against A first (largest creditor), then B, for at most two clean transactions.

**Reading a balance and settling up.** `getBalancesForUser` scans pairs containing the user; since each pair stores one signed number, "A owes B 100" and "B owes A 40" can never both exist — they are already netted to "A owes B 60." If B pays A 300 in cash, `settleUp(B, A, 300)` calls `addDebt(A, B, 300)`, cancelling the existing debt via the `compute` merge; if B owed exactly 300, the entry is removed — nothing left to track.

**Group-wide simplification.** `simplifyGroupDebts(members)` computes each member's overall net position, then reuses the same greedy algorithm to shrink a tangled web of pairwise debts into a few direct payments.

## How to Extend (Follow-ups)

- **Split by shares (weights).** Add `SharesSplitStrategy`: `share = totalAmount * userWeight / totalWeight`, last participant absorbs the remainder, same trick as Percentage split.
- **Persistence and multi-currency.** Move `BalanceSheet` to a table `(user_a_id, user_b_id, amount)` with `CHECK (user_a_id < user_b_id)`. Add `Currency` to `Expense`; key balances by `(UserPair, Currency)`, never netting across currencies.
- **Exact minimum-transaction simplification.** Backtracking guarantees the true minimum transaction count, at exponential worst-case cost — a trade-off worth stating against the greedy default.
- **Scale out and audit.** Shard `BalanceSheet` per group, or move it to Redis. `Expense` is already immutable and ordered, so an audit trail is free; add an Observer-style event to notify affected users.

## Complexity & Thread-Safety Notes

| Operation | Time | Notes |
|---|---|---|
| Equal / Exact / Percent `calculateSplits` | O(n) | n = participants |
| `BalanceSheet.addDebt` | O(1) amortized | one `ConcurrentHashMap.compute` |
| `BalanceSheet.getBalancesForUser` | O(p) | p = stored pairs; a per-user index gives O(1) per relation at the cost of extra writes |
| `DebtSimplifier.simplify` | O(n log n) | n = users with non-zero net balance; dominated by the sorts |

**Thread-safety:**
- `ConcurrentHashMap.compute` makes `addDebt` atomic per pair; different pairs never block each other. `Split`, `Transaction`, `Expense` are immutable, so any thread can read them without locking.
- `expenses` uses `CopyOnWriteArrayList`, good for "written occasionally, read often." Avoid it if expenses are added at very high frequency, since every write copies the backing array.
- `DebtSimplifier.simplify` touches no shared state — it takes a `Map` snapshot and returns a new list — so build that snapshot from one `getNetBalances` call, not several reads that could interleave with writes.

## Interview Tips & Common Mistakes

- **Never use `double` for money.** Binary floating-point cannot represent values like 0.1 exactly, and small errors compound over many expenses. Use `long` minor units or `BigDecimal`, and validate splits before committing them.
- **Handle the rounding remainder.** Naive division of 100 by 3 loses a minor unit (`33+33+33=99`). Spread it (Equal split) or let the last participant absorb it (Percentage split).
- **Do not store both directions of a debt separately.** Independent `debts[A][B]` and `debts[B][A]` numbers can drift out of sync. Store one signed number per unordered pair instead.
- **Admit greedy debt simplification is not optimal, and reuse it everywhere.** True minimization is a hard problem; "largest debtor pays largest creditor" is a fast heuristic close to what real apps use. Folding a multi-payer expense and simplifying a whole group are the same problem, so reuse one `DebtSimplifier`.
- **Keep `SplitStrategy` free of `BalanceSheet` knowledge.** A strategy only computes splits from given numbers, never overall balances — keeps each strategy easy to unit test.
