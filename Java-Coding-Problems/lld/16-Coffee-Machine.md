# Coffee Machine (LLD)

## Problem

Design a coffee vending machine. The machine can make several drinks, for example Espresso, Latte, and Cappuccino. Each drink has a recipe. A recipe is a map from ingredient to amount. Ingredients are things like water, milk, coffee, and sugar.

The machine keeps an inventory of ingredients. The machine must support these operations:

- Select a drink.
- Check if the machine has enough ingredients for that drink.
- Make the drink. This must reduce the inventory by the recipe amount.
- Refill ingredients.
- Report when an ingredient is low, so an operator can refill it early.

Real vending machines have more than one outlet. Two people can press "Espresso" and "Latte" at almost the same moment, on two different outlets, but they share one inventory. The design must stop both drinks from using the same water at the same time. In other words, the "check ingredients, then use ingredients" step must be thread-safe.

The design must also make it easy to add a new drink later, without changing the machine's core code.

## Requirements & Clarifying Questions

Functional requirements:

1. Support multiple drinks, each with its own recipe.
2. Show which drinks can be made right now, based on current stock.
3. Make a drink: check stock, then deduct stock, as one step.
4. Refill any ingredient by a given amount.
5. Notify an observer (for example, a console log or an alert service) when an ingredient drops below a low-stock threshold.
6. Allow adding a new drink type without editing the `CoffeeMachine` class.

Non-functional requirements:

1. Thread safety. Several outlets can call "make drink" at the same time on the same machine. The inventory must never go negative, and two drinks must never "spend" the same unit of an ingredient.
2. Good performance under load. The lock we use should be held for a short time only.
3. Easy to extend: new drinks, new ingredients, new observers.

Clarifying questions I would ask the interviewer:

- Q: How many outlets does one machine have? Do they share one inventory, or does each outlet have its own tank?
  A (assumed): One shared inventory for the whole machine. This is the harder and more common case, so I will design for it.
- Q: Should "make drink" block and wait if ingredients are short, or fail immediately?
  A (assumed): Fail immediately with a clear error. No queueing in this version.
- Q: Is price/payment part of this problem?
  A (assumed): Out of scope. I will add a `price` field on the recipe for realism, but I will not build a payment flow.
- Q: Can recipes change at runtime (operator edits a recipe)?
  A (assumed): Recipes are fixed at machine setup, but new recipes can be registered later.

## Design / Approach

Key design decisions and the patterns behind them:

1. **Builder pattern for `Recipe`.** A recipe has a name, a price, and a map of ingredient-to-amount pairs. Building this map by hand in many places is error-prone. A `Recipe.Builder` gives a clean, readable way to construct a recipe: `new Recipe.Builder("Latte").with(WATER, 100).with(MILK, 150)...build()`.

2. **Factory pattern for default recipes.** `RecipeFactory` creates the built-in drinks (Espresso, Latte, Cappuccino). The `CoffeeMachine` does not know how a recipe is built. It only asks the factory for a starting set of recipes. This keeps recipe-creation logic in one place.

3. **Open/Closed Principle for new drinks.** The machine stores recipes in a `Map<String, Recipe>`. Adding a new drink means calling `machine.addRecipe(recipe)` with a new `Recipe` object. We do not touch `CoffeeMachine`'s code or add a new `if` branch. This satisfies "open for extension, closed for modification."

4. **Observer pattern for low-stock alerts.** `Inventory` holds a list of `InventoryObserver` objects. After every deduction, `Inventory` checks each ingredient level. If a level is below its threshold, it notifies all observers. A `ConsoleLowStockNotifier` is one observer implementation. We could add an `EmailNotifier` or `SmsNotifier` later, again without changing `Inventory`'s core logic.

5. **Thread-safe check-and-deduct.** This is the core of the problem. "Check if enough" and "deduct" must happen as one atomic step, or two threads can both pass the check and then both deduct, taking the inventory below zero. We protect this with a `ReentrantLock` inside `Inventory`. The lock is held only for the short check-and-deduct block, not for the whole drink-making process (which might include a simulated brew delay). This keeps the lock's critical section small, so the machine still serves many outlets with good throughput.

6. **Single Responsibility Principle.** `Recipe` only describes a drink. `Inventory` only manages stock and thread safety. `CoffeeMachine` only coordinates recipes and inventory. `InventoryObserver` implementations only handle notification. Each class has one reason to change.

ASCII class sketch:

```
                +----------------------+
                |     Ingredient       |  (enum: WATER, MILK, COFFEE, SUGAR)
                +----------------------+

+-------------------+        uses         +----------------------+
|      Recipe        |<--------------------|   Recipe.Builder     |
+-------------------+                      +----------------------+
| name: String        |
| price: BigDecimal    |
| ingredients: Map<>   |
+-------------------+
          ^
          | creates
+-------------------+
|   RecipeFactory     |  (Factory pattern)
+-------------------+

+---------------------------+          notifies         +-----------------------+
|        Inventory           |-------------------------->|  InventoryObserver     |
+---------------------------+                            +-----------------------+
| stock: Map<Ingredient,Int> |                            | onLowStock(ingr, qty)  |
| lock: ReentrantLock         |                            +-----------------------+
| hasEnough(recipe): boolean  |                                     ^
| checkAndDeduct(recipe)      |                                     |
| refill(ingr, qty)            |                     +--------------------------+
| addObserver(obs)              |                     | ConsoleLowStockNotifier   |
+---------------------------+                     +--------------------------+

+---------------------------+
|      CoffeeMachine          |
+---------------------------+
| recipes: Map<String,Recipe> |
| inventory: Inventory          |
| addRecipe(recipe)              |
| makeDrink(name): Receipt        |   <-- called concurrently by outlets
| refill(ingr, qty)                 |
+---------------------------+
```

## Java Solution

```java
import java.math.BigDecimal;
import java.util.*;
import java.util.concurrent.ConcurrentHashMap;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;
import java.util.concurrent.locks.ReentrantLock;

// ---------- Ingredient ----------
enum Ingredient {
    WATER, MILK, COFFEE, SUGAR
}

// ---------- Recipe (built with Builder pattern) ----------
final class Recipe {
    private final String name;
    private final BigDecimal price;
    private final Map<Ingredient, Integer> ingredients;

    private Recipe(Builder builder) {
        this.name = builder.name;
        this.price = builder.price;
        this.ingredients = Collections.unmodifiableMap(new EnumMap<>(builder.ingredients));
    }

    public String getName() { return name; }
    public BigDecimal getPrice() { return price; }
    public Map<Ingredient, Integer> getIngredients() { return ingredients; }

    public static class Builder {
        private final String name;
        private BigDecimal price = BigDecimal.ZERO;
        private final Map<Ingredient, Integer> ingredients = new EnumMap<>(Ingredient.class);

        public Builder(String name) { this.name = name; }

        public Builder with(Ingredient ingredient, int amount) {
            ingredients.put(ingredient, amount);
            return this;
        }

        public Builder price(BigDecimal price) {
            this.price = price;
            return this;
        }

        public Recipe build() {
            if (ingredients.isEmpty()) {
                throw new IllegalStateException("Recipe must have at least one ingredient");
            }
            return new Recipe(this);
        }
    }
}

// ---------- RecipeFactory (Factory pattern) ----------
final class RecipeFactory {

    private RecipeFactory() { }

    public static List<Recipe> defaultRecipes() {
        List<Recipe> recipes = new ArrayList<>();

        recipes.add(new Recipe.Builder("Espresso")
                .with(Ingredient.WATER, 50)
                .with(Ingredient.COFFEE, 18)
                .price(new BigDecimal("2.50"))
                .build());

        recipes.add(new Recipe.Builder("Latte")
                .with(Ingredient.WATER, 100)
                .with(Ingredient.MILK, 150)
                .with(Ingredient.COFFEE, 18)
                .with(Ingredient.SUGAR, 5)
                .price(new BigDecimal("3.50"))
                .build());

        recipes.add(new Recipe.Builder("Cappuccino")
                .with(Ingredient.WATER, 80)
                .with(Ingredient.MILK, 100)
                .with(Ingredient.COFFEE, 18)
                .price(new BigDecimal("3.75"))
                .build());

        return recipes;
    }
}

// ---------- Observer pattern for low stock ----------
interface InventoryObserver {
    void onLowStock(Ingredient ingredient, int remaining);
}

final class ConsoleLowStockNotifier implements InventoryObserver {
    @Override
    public void onLowStock(Ingredient ingredient, int remaining) {
        System.out.printf("[ALERT] %s is low: only %d units left.%n", ingredient, remaining);
    }
}

// Custom exception for a clear, specific error
class InsufficientIngredientException extends RuntimeException {
    public InsufficientIngredientException(String message) {
        super(message);
    }
}

// ---------- Inventory: thread-safe stock management ----------
final class Inventory {
    private final Map<Ingredient, Integer> stock = new EnumMap<>(Ingredient.class);
    private final Map<Ingredient, Integer> lowStockThreshold = new EnumMap<>(Ingredient.class);
    private final List<InventoryObserver> observers = new CopyOnWriteArrayList<>();
    private final ReentrantLock lock = new ReentrantLock();

    public Inventory() {
        // starting stock; a real machine would load this from config
        stock.put(Ingredient.WATER, 1000);
        stock.put(Ingredient.MILK, 1000);
        stock.put(Ingredient.COFFEE, 500);
        stock.put(Ingredient.SUGAR, 500);

        lowStockThreshold.put(Ingredient.WATER, 100);
        lowStockThreshold.put(Ingredient.MILK, 100);
        lowStockThreshold.put(Ingredient.COFFEE, 50);
        lowStockThreshold.put(Ingredient.SUGAR, 50);
    }

    public void addObserver(InventoryObserver observer) {
        observers.add(observer);
    }

    /**
     * Checks stock and deducts it in ONE atomic step.
     * This is the method that must be thread-safe: it is the only place
     * where "read current stock" and "write new stock" happen together.
     * Two outlets calling this at the same time must not both succeed
     * on the same last unit of water.
     */
    public void checkAndDeduct(Map<Ingredient, Integer> required) {
        lock.lock();
        try {
            // Step 1: check everything first, so we never deduct halfway
            // through a recipe and then fail partway.
            for (Map.Entry<Ingredient, Integer> entry : required.entrySet()) {
                int available = stock.getOrDefault(entry.getKey(), 0);
                if (available < entry.getValue()) {
                    throw new InsufficientIngredientException(
                            "Not enough " + entry.getKey() + ". Needed " + entry.getValue()
                                    + ", available " + available);
                }
            }
            // Step 2: now it is safe to deduct, because we still hold the lock.
            for (Map.Entry<Ingredient, Integer> entry : required.entrySet()) {
                int newLevel = stock.get(entry.getKey()) - entry.getValue();
                stock.put(entry.getKey(), newLevel);
                if (newLevel <= lowStockThreshold.getOrDefault(entry.getKey(), 0)) {
                    notifyLowStock(entry.getKey(), newLevel);
                }
            }
        } finally {
            lock.unlock();
        }
    }

    public boolean hasEnough(Map<Ingredient, Integer> required) {
        lock.lock();
        try {
            for (Map.Entry<Ingredient, Integer> entry : required.entrySet()) {
                if (stock.getOrDefault(entry.getKey(), 0) < entry.getValue()) {
                    return false;
                }
            }
            return true;
        } finally {
            lock.unlock();
        }
    }

    public void refill(Ingredient ingredient, int amount) {
        lock.lock();
        try {
            stock.merge(ingredient, amount, Integer::sum);
        } finally {
            lock.unlock();
        }
    }

    public int getLevel(Ingredient ingredient) {
        lock.lock();
        try {
            return stock.getOrDefault(ingredient, 0);
        } finally {
            lock.unlock();
        }
    }

    private void notifyLowStock(Ingredient ingredient, int remaining) {
        for (InventoryObserver observer : observers) {
            observer.onLowStock(ingredient, remaining);
        }
    }
}

// ---------- Receipt: result of making a drink ----------
final class Receipt {
    private final String drinkName;
    private final BigDecimal price;

    public Receipt(String drinkName, BigDecimal price) {
        this.drinkName = drinkName;
        this.price = price;
    }

    @Override
    public String toString() {
        return "Receipt{drink='" + drinkName + "', price=" + price + "}";
    }
}

// ---------- CoffeeMachine: coordinates recipes and inventory ----------
final class CoffeeMachine {
    private final Map<String, Recipe> recipes = new ConcurrentHashMap<>();
    private final Inventory inventory;

    public CoffeeMachine(Inventory inventory, List<Recipe> startingRecipes) {
        this.inventory = inventory;
        for (Recipe recipe : startingRecipes) {
            recipes.put(recipe.getName(), recipe);
        }
    }

    /** Open/Closed: add a brand-new drink without touching this class's code. */
    public void addRecipe(Recipe recipe) {
        recipes.put(recipe.getName(), recipe);
    }

    public boolean canMake(String drinkName) {
        Recipe recipe = getRecipeOrThrow(drinkName);
        return inventory.hasEnough(recipe.getIngredients());
    }

    /**
     * Called by any outlet, on any thread. Many outlets can call this
     * concurrently on the same machine. The actual stock change happens
     * inside Inventory.checkAndDeduct, which is guarded by a lock, so
     * this method is safe to call from multiple threads at once.
     */
    public Receipt makeDrink(String drinkName) {
        Recipe recipe = getRecipeOrThrow(drinkName);
        inventory.checkAndDeduct(recipe.getIngredients());
        // Brewing itself (heating, pouring) can take time, but it happens
        // AFTER the lock is released, so it does not block other outlets.
        simulateBrewing();
        return new Receipt(recipe.getName(), recipe.getPrice());
    }

    public void refill(Ingredient ingredient, int amount) {
        inventory.refill(ingredient, amount);
    }

    private Recipe getRecipeOrThrow(String drinkName) {
        Recipe recipe = recipes.get(drinkName);
        if (recipe == null) {
            throw new NoSuchElementException("Unknown drink: " + drinkName);
        }
        return recipe;
    }

    private void simulateBrewing() {
        try {
            Thread.sleep(50);
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }
    }
}

// ---------- Demo: several outlets making drinks at the same time ----------
public class CoffeeMachineDemo {
    public static void main(String[] args) throws InterruptedException {
        Inventory inventory = new Inventory();
        inventory.addObserver(new ConsoleLowStockNotifier());

        CoffeeMachine machine = new CoffeeMachine(inventory, RecipeFactory.defaultRecipes());

        // Add a new drink at runtime, with no change to CoffeeMachine's code.
        machine.addRecipe(new Recipe.Builder("Black Coffee")
                .with(Ingredient.WATER, 120)
                .with(Ingredient.COFFEE, 15)
                .price(new BigDecimal("2.00"))
                .build());

        String[] drinkNames = {"Espresso", "Latte", "Cappuccino", "Black Coffee"};
        ExecutorService outlets = Executors.newFixedThreadPool(4);

        for (int i = 0; i < 20; i++) {
            String drink = drinkNames[i % drinkNames.length];
            outlets.submit(() -> {
                try {
                    Receipt receipt = machine.makeDrink(drink);
                    System.out.println(Thread.currentThread().getName() + " made " + receipt);
                } catch (InsufficientIngredientException e) {
                    System.out.println(Thread.currentThread().getName() + " failed: " + e.getMessage());
                }
            });
        }

        outlets.shutdown();
        outlets.awaitTermination(5, java.util.concurrent.TimeUnit.SECONDS);

        System.out.println("Remaining water: " + inventory.getLevel(Ingredient.WATER));
    }
}
```

Note: `Inventory` uses `CopyOnWriteArrayList` for `observers`. Add the import `java.util.concurrent.CopyOnWriteArrayList` at the top with the other `java.util.concurrent` imports.

## How It Works

`Recipe.Builder` builds a `Recipe` step by step. Each `.with(ingredient, amount)` call adds one entry to an `EnumMap`. `EnumMap` is a good choice here because the key type is a small, fixed enum, so it is faster and uses less memory than a `HashMap`. The final `Recipe` wraps its ingredient map with `Collections.unmodifiableMap`, so no one can change a recipe after it is built. This makes `Recipe` immutable, which is a safe default for any object that many threads read at the same time.

`RecipeFactory` builds the three starting drinks. `CoffeeMachine` does not know how each recipe is put together. It just receives a `List<Recipe>` in its constructor and stores them in a map keyed by name.

`Inventory` is the heart of the thread-safety story. It holds a `ReentrantLock`. The method `checkAndDeduct` does two things inside one `lock()`/`unlock()` block: first it checks every ingredient has enough stock, then it deducts every ingredient. Because both steps happen while the lock is held, no other thread can enter this method in between the check and the deduct. This closes the "race window" where two threads could both check "yes, we have 50ml of water," and then both try to take it, leaving the total short.

After the deduct step, `checkAndDeduct` also checks each new stock level against its low-stock threshold. If a level has dropped below the threshold, it calls `notifyLowStock`, which loops through all registered `InventoryObserver` objects and calls their `onLowStock` method. This is the Observer pattern: `Inventory` does not need to know what a `ConsoleLowStockNotifier` does with the alert. It could just as easily be an email sender or a dashboard update, and `Inventory`'s code would not change.

`CoffeeMachine.makeDrink` looks up the `Recipe` by name, then calls `inventory.checkAndDeduct(recipe.getIngredients())`. Notice that the "brewing" delay (`simulateBrewing`) happens after this call, outside the lock. This detail matters: if we held the lock during brewing, only one drink could brew at a time across the whole machine, even though brewing itself does not touch shared inventory state anymore (the ingredients are already deducted). Keeping the lock's critical section small is what lets several outlets brew drinks in parallel while still keeping the shared stock numbers correct.

In the demo, `ExecutorService` with four threads represents four outlets. Twenty drink orders run across these threads. Each call to `machine.makeDrink(...)` is independent and safe, because the shared state changes only inside the locked block.

## How to Extend (Follow-ups)

- **Add a new drink:** call `machine.addRecipe(new Recipe.Builder("Mocha")...)`. No change needed in `CoffeeMachine`, `Inventory`, or `Recipe`. This is the Open/Closed Principle in action.
- **Add a new ingredient:** add a new constant to the `Ingredient` enum, and add a starting stock value in `Inventory`'s constructor (or better, load stock from a config file or database instead of hardcoding it).
- **Support payment:** add a `PaymentStrategy` interface (Strategy pattern) with implementations like `CashPayment` and `CardPayment`. `CoffeeMachine.makeDrink` would take a `PaymentStrategy` and charge it for `recipe.getPrice()` before brewing.
- **Support a request queue instead of immediate failure:** if ingredients are short, instead of throwing right away, add the request to a queue and retry after the next refill. This would need a `BlockingQueue` and a background worker thread.
- **Per-outlet locking:** if the machine design changes so each outlet has its own separate tank for some ingredients (for example, its own milk frother) but shares others (like the main water tank), split `Inventory` into per-ingredient locks, so an outlet using only milk does not block another outlet that only needs water. `java.util.concurrent.locks.ReentrantLock`, one per `Ingredient`, would work well here.
- **Undo on partial failure:** currently we check everything before deducting anything, so a `makeDrink` call cannot fail halfway through a recipe. If we ever split deduction across multiple locks, we would need a rollback step (or a two-phase commit style approach) to keep this same guarantee.
- **Metrics:** add an observer that counts drinks made per type, useful for a "best seller" report. This reuses the same `InventoryObserver`-style hook, or a new `DrinkMadeObserver` interface on `CoffeeMachine`.

## Complexity & Thread-Safety Notes

- `checkAndDeduct`: O(k) time, where k is the number of ingredients in a recipe (usually 2 to 4). This is fast, so the lock is held only briefly.
- `hasEnough`: O(k) time, same reasoning.
- `refill`: O(1) time.
- Space: O(number of distinct ingredients) for the stock map, plus O(number of recipes × ingredients per recipe) for all recipes stored in the machine.
- Thread safety: `ReentrantLock` around `checkAndDeduct`, `hasEnough`, and `refill` in `Inventory` makes these three methods mutually exclusive. Only one thread can run any of them at a time on one `Inventory` instance. This is a coarse-grained lock: simple and correct, but it means all outlets serialize on this one lock for the brief check-and-deduct step. For a small number of outlets (say, up to 10), this is not a bottleneck, because the critical section is short (a few map look-ups).
- Why not `synchronized`? A plain `synchronized` method on `Inventory` would also work and would be simpler. `ReentrantLock` is shown here because it supports extra features like `tryLock(timeout)`, which is useful if we later want "wait up to 2 seconds for ingredients, then fail" instead of failing immediately. For this problem's scope, either choice is a correct, defendable answer in an interview.
- `recipes` map uses `ConcurrentHashMap`, so reading and adding recipes is also thread-safe, even if an operator adds a new drink while outlets are actively making other drinks.
- `observers` list uses `CopyOnWriteArrayList`, a good fit because observers are added rarely (usually just at startup) but read often (every time stock changes). Reads never block on writes with this collection.
- No deadlock risk: `Inventory` uses exactly one lock, and no method that holds the lock calls another method that tries to take the same lock again in a way that could cause a cycle. There is only one lock in the whole design, so a deadlock between two locks is not possible here.

## Interview Tips & Common Mistakes

- **The most common mistake:** doing `if (hasEnough(recipe)) { deduct(recipe); }` as two separate method calls, each with its own lock. This looks safe but is not. Between the `hasEnough` call returning `true` and the `deduct` call starting, another thread can slip in and take the same ingredients. Always combine "check" and "act" into one atomic operation, guarded by one lock, one method. This is called the "check-then-act" race condition, and interviewers often listen for you to name it.
- **Explain why you chose a lock over `synchronized` or vice versa.** Either is fine, but you should be able to say why. `ReentrantLock` gives more control (timeouts, fairness); `synchronized` is simpler and less error-prone (no risk of forgetting `unlock()` in a `finally` block, since the JVM manages it). If you use `ReentrantLock`, always release it in a `finally` block, exactly as shown above. Forgetting this is a common bug: if an exception is thrown after `lock()` but before `unlock()`, the lock stays held forever, and the whole machine freezes.
- **Do not hold the lock longer than needed.** A common mistake is to lock around the entire `makeDrink` method, including the brewing delay. This turns a machine with many outlets into a machine that can only make one drink at a time, which defeats the purpose of "concurrent outlets." Show that you understand the difference between the shared-state part (inventory) and the non-shared part (brewing).
- **Immutability matters.** Making `Recipe` immutable (final fields, unmodifiable map) removes an entire class of bugs: no thread can accidentally change a recipe while another thread reads it. Point this out; it shows attention to detail beyond just the main concurrency question.
- **Mention the design patterns by name.** Interviewers at this level expect you to say "this is a Builder," "this is the Observer pattern," "this follows Open/Closed," not just write the code silently. Naming the pattern shows you understand why the shape of the code is what it is, not just that it compiles.
- **Do not over-engineer.** Some candidates jump straight to per-ingredient locks or lock-free atomic structures. For a machine with a handful of outlets, a single lock around a short critical section is simpler, easier to reason about, and fast enough. Only propose finer-grained locking if asked to scale further, and explain the trade-off (more complexity, harder to avoid deadlocks with multiple locks) when you do.
