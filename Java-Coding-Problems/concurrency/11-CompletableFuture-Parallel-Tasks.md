# CompletableFuture — Parallel Task Execution

## Problem

A backend service often needs to call several other services or do several slow tasks at the same time. For example, to build one product page, you may need to call an inventory service, a pricing service, and a reviews service. These calls are independent of each other. If you call them one after another (sequentially), the total time is the sum of all three calls. If you call them in parallel, the total time is close to the slowest single call.

`CompletableFuture` is a Java class that represents a result that is not ready yet, but will be ready later. It is an async (asynchronous) result. "Async" means the calling thread does not block and wait. Instead, it gets a `CompletableFuture` object right away, and the real work runs on another thread. You can attach steps to a `CompletableFuture`. Each step runs automatically when the previous step finishes. This is called chaining.

You are asked: design a solution that calls 3 independent services in parallel, merges their results into one response, handles errors from any of the calls, and applies a timeout so the whole operation does not hang forever.

## Requirements & Clarifying Questions

Before coding, ask these questions in an interview. They show that you think about real systems, not just one method call.

1. **Are the calls independent, or does one depend on another's result?** If Call B needs the output of Call A, you must chain them (`thenCompose`). If all calls are independent, you can run them fully in parallel and combine results at the end (`thenCombine`, or `allOf`).
2. **What thread pool should run the tasks?** Should we use the JVM's shared default pool (`ForkJoinPool.commonPool()`), or a custom `Executor`? This matters a lot when calls do blocking I/O (like a network call or a JDBC query).
3. **What happens if one call fails?** Do we fail the whole operation, or do we return a partial result (for example, show the page without reviews if the reviews service is down)?
4. **Is there a timeout?** How long do we wait for each call, or for the whole group of calls, before giving up?
6. **Do we need the result of all tasks, or just the fastest one?** `allOf` waits for all. `anyOf` returns as soon as one finishes. Some systems call the same service on two servers and use whichever answers first — that needs `anyOf`.
7. **How do we report errors to the caller?** Do we throw an exception, log and return a default value, or return a result object that carries both data and an error flag?

For this solution sheet, we assume: the three calls are independent, we use a custom `Executor` for I/O-bound work, we return a partial result when a non-critical call fails, and we apply a timeout to the whole merged operation.

## Design / Approach

### What is CompletableFuture?

`CompletableFuture<T>` is Java's class for an async result of type `T`. It implements both `Future<T>` (an older interface for "a result that will arrive later") and `CompletionStage<T>` (a newer interface that adds chaining methods). The chaining methods are what make `CompletableFuture` powerful. You can attach a next step, and it runs automatically, without you writing code that blocks and waits.

### Starting a task: supplyAsync and runAsync

- `CompletableFuture.supplyAsync(Supplier<T> task)` — runs `task` on a background thread and completes the future with the value the task returns. Use this when the task produces a result.
- `CompletableFuture.runAsync(Runnable task)` — runs `task` on a background thread, but the task returns nothing (`Void`). Use this for a side-effect action, like sending a message.

Both methods have an overload that takes an `Executor`:

```java
CompletableFuture<Product> future = CompletableFuture.supplyAsync(
    () -> callInventoryService(productId),
    myExecutor
);
```

**Why not the default ForkJoinPool for blocking I/O?** If you call `supplyAsync` without an `Executor`, it uses `ForkJoinPool.commonPool()`. This pool is shared by your whole JVM, including things like parallel streams. It is sized to the number of CPU cores (`Runtime.getRuntime().availableProcessors()`), because it is designed for CPU-bound work, not for waiting on network calls.

If your task does blocking I/O, such as an HTTP call to another service or a JDBC query, the thread sits idle while waiting for a reply. If you run many such tasks on the small common pool, you can exhaust all its threads. This can silently slow down or starve unrelated code elsewhere in your application that also depends on the common pool. This is a common production bug: an unrelated feature becomes slow because someone else's blocking call is hogging the shared pool.

The fix: create your own `Executor` (usually backed by a `ThreadPoolExecutor`), sized for I/O-bound work (often more threads than CPU cores, since threads spend most of their time waiting, not computing). Pass it explicitly to every `supplyAsync` / `runAsync` call, and to every chained step you want to run off the calling thread.

```java
Executor ioExecutor = Executors.newFixedThreadPool(
    16,
    r -> {
        Thread t = new Thread(r, "io-pool");
        t.setDaemon(true);
        return t;
    }
);
```

### Chaining: thenApply, thenAccept, thenRun

Once you have a `CompletableFuture<T>`, you can attach a next step:

- `thenApply(Function<T, R>)` — transforms the result from type `T` to type `R`. This is like `map` on a stream. Example: turn a raw JSON string into a parsed object.
- `thenAccept(Consumer<T>)` — consumes the result, but returns nothing. Use this when the next step is a side effect, like logging or writing to a cache.
- `thenRun(Runnable)` — runs an action that does not need the result at all. Only used to say "after this finishes, do this next thing."

Each of these has an `Async` variant: `thenApplyAsync`, `thenAcceptAsync`, `thenRunAsync`. The non-async version runs the step on the same thread that completed the previous stage (this can even be the calling thread, if the future was already done). The async version submits the step to a thread pool — either the common pool, or an `Executor` you pass in. For I/O-bound chains, always pass your own executor to the async variant, for the same reason explained above.

### Combining two independent results: thenCombine

`thenCombine` merges the results of two independent `CompletableFuture`s using a `BiFunction`, once both are done:

```java
CompletableFuture<Price> priceFuture = CompletableFuture.supplyAsync(() -> callPricingService(id), ioExecutor);
CompletableFuture<Stock> stockFuture = CompletableFuture.supplyAsync(() -> callInventoryService(id), ioExecutor);

CompletableFuture<ProductView> combined = priceFuture.thenCombine(
    stockFuture,
    (price, stock) -> new ProductView(price, stock)
);
```

Use `thenCombine` when you have exactly two independent futures and you want to merge their results into one value. For three or more independent futures, `allOf` (shown below) is the better tool.

### Chaining a dependent async call: thenCompose

`thenCompose` is used when one async call depends on the result of another async call. This is the async version of "flatMap." The function you pass must itself return a `CompletableFuture`, not a plain value.

```java
CompletableFuture<Order> orderFuture = CompletableFuture
    .supplyAsync(() -> lookupUserId(sessionToken), ioExecutor)
    .thenCompose(userId -> CompletableFuture.supplyAsync(() -> fetchLatestOrder(userId), ioExecutor));
```

**Why not `thenApply` here?** If you used `thenApply` with a function that returns a `CompletableFuture`, you would end up with a `CompletableFuture<CompletableFuture<Order>>` — a future nested inside a future. That is awkward to use; you would need an extra step to unwrap it. `thenCompose` flattens this automatically, giving you a plain `CompletableFuture<Order>`.

**Rule of thumb:** use `thenApply` when your function returns a plain value. Use `thenCompose` when your function itself returns a `CompletableFuture` (because the next step is another async call).

### Running many tasks in parallel: allOf and anyOf

`CompletableFuture.allOf(futures...)` takes any number of futures and returns a `CompletableFuture<Void>` that completes only when **all** of them complete (successfully or with an error). It does not give back their results directly — you must read each original future's result yourself after `allOf` completes, usually with `.join()`, which is safe to call at that point since we know each future is already done.

```java
CompletableFuture<Void> all = CompletableFuture.allOf(future1, future2, future3);
all.join(); // waits for all three
// now it is safe to call future1.join(), future2.join(), future3.join()
```

`CompletableFuture.anyOf(futures...)` returns a `CompletableFuture<Object>` that completes as soon as **any one** of the given futures completes. This is useful when you send the same request to two redundant servers and want to use whichever replies first, or when you race a task against a fallback.

### Exception handling: exceptionally, handle, whenComplete

If the task inside a `CompletableFuture` throws an exception, the future does not throw right away. Instead, it completes "exceptionally." The exception only surfaces when someone calls `.get()` or `.join()`, or when the next chained stage checks for it. `.get()` wraps the original exception in a checked `ExecutionException`. `.join()` wraps it in an unchecked `CompletionException`. In both cases, call `.getCause()` to get the real, original exception.

- `exceptionally(Function<Throwable, T>)` — runs only if the previous stage failed. It lets you supply a fallback value, turning a failed future back into a successful one.

```java
CompletableFuture<Reviews> reviewsFuture = CompletableFuture
    .supplyAsync(() -> callReviewsService(id), ioExecutor)
    .exceptionally(ex -> {
        log.warn("Reviews service failed, showing empty list", ex);
        return Reviews.empty(); // fallback value
    });
```

- `handle(BiFunction<T, Throwable, R>)` — runs whether the previous stage succeeded or failed. Exactly one of the two arguments is non-null: `T` on success, `Throwable` on failure. This is more general than `exceptionally`, because it also gives you a hook to inspect and transform the success case.
- `whenComplete(BiConsumer<T, Throwable>)` — similar to `handle`, but it does not transform the result. It is used for side effects, like logging, on either outcome. It passes through the original result (or exception) unchanged to the next stage.

**Key difference:** `exceptionally` only reacts to failure and recovers with a value. `handle` reacts to both outcomes and can change the result. `whenComplete` reacts to both outcomes but never changes the result — it only observes.

### Timeouts: orTimeout and completeOnTimeout

Both methods were added in Java 9. They protect against a task that never completes (for example, a service that hangs and never answers).

- `orTimeout(long timeout, TimeUnit unit)` — if the future is not done within the given time, it completes exceptionally with a `TimeoutException`. You then handle this like any other failure, typically with `exceptionally` or `handle`.
- `completeOnTimeout(T value, long timeout, TimeUnit unit)` — if the future is not done within the given time, it completes successfully with the given fallback `value`, instead of throwing.

```java
CompletableFuture<Reviews> reviewsFuture = CompletableFuture
    .supplyAsync(() -> callReviewsService(id), ioExecutor)
    .completeOnTimeout(Reviews.empty(), 500, TimeUnit.MILLISECONDS);
```

Important: a timeout on the `CompletableFuture` only stops *waiting* for the task. It does not stop the underlying thread from still running the slow call in the background. If you need to truly cancel the work (for example, close an HTTP connection), you need cancellation support from the underlying client (like an `HttpClient` with its own timeout), not just `orTimeout`.

## Java Solution

Here is the worked example: call 3 independent services in parallel, merge their results, handle errors so one failing call does not break the whole response, and apply an overall timeout.

```java
import java.util.concurrent.*;

public class ProductPageAggregator {

    // Custom executor for I/O-bound calls. Sized larger than CPU count,
    // because threads mostly wait on network I/O, not on computation.
    private final ExecutorService ioExecutor = new ThreadPoolExecutor(
        8, 32,
        60L, TimeUnit.SECONDS,
        new LinkedBlockingQueue<>(),
        r -> {
            Thread t = new Thread(r, "product-io");
            t.setDaemon(true);
            return t;
        }
    );

    // Simulated remote calls. In real code these would use an HTTP client.
    private Price callPricingService(String productId) { /* blocking network call */ return new Price(productId, 19.99); }
    private Stock callInventoryService(String productId) { /* blocking network call */ return new Stock(productId, 42); }
    private Reviews callReviewsService(String productId) { /* blocking network call */ return new Reviews(productId, 4.5, 120); }

    public ProductView buildProductPage(String productId) {
        CompletableFuture<Price> priceFuture = CompletableFuture
            .supplyAsync(() -> callPricingService(productId), ioExecutor)
            .orTimeout(1, TimeUnit.SECONDS)
            .exceptionally(ex -> {
                // Pricing is critical. We cannot show a page without a price.
                throw new CompletionException(new PricingUnavailableException(productId, ex));
            });

        CompletableFuture<Stock> stockFuture = CompletableFuture
            .supplyAsync(() -> callInventoryService(productId), ioExecutor)
            .orTimeout(1, TimeUnit.SECONDS)
            .exceptionally(ex -> {
                // If inventory is down, assume out of stock rather than fail the page.
                return Stock.unknown(productId);
            });

        CompletableFuture<Reviews> reviewsFuture = CompletableFuture
            .supplyAsync(() -> callReviewsService(productId), ioExecutor)
            .completeOnTimeout(Reviews.empty(), 500, TimeUnit.MILLISECONDS)
            .exceptionally(ex -> Reviews.empty());

        CompletableFuture<Void> allDone = CompletableFuture.allOf(priceFuture, stockFuture, reviewsFuture);

        try {
            // Overall safety net: do not wait forever even if something above misbehaves.
            allDone.orTimeout(3, TimeUnit.SECONDS).join();
        } catch (CompletionException ex) {
            Throwable cause = ex.getCause();
            if (cause instanceof PricingUnavailableException) {
                throw (PricingUnavailableException) cause; // no page without price: propagate
            }
            throw new ProductPageException("Failed to build product page for " + productId, cause);
        }

        // Safe to call join() here: allDone already completed, so each future is done too.
        return new ProductView(priceFuture.join(), stockFuture.join(), reviewsFuture.join());
    }

    public void shutdown() {
        ioExecutor.shutdown();
    }
}
```

A shorter version, if you prefer to collect results with `thenCombine` chained twice instead of `allOf` plus manual `join` calls:

```java
CompletableFuture<ProductView> pageFuture = priceFuture
    .thenCombine(stockFuture, (price, stock) -> new Object[]{price, stock})
    .thenCombine(reviewsFuture, (arr, reviews) ->
        new ProductView((Price) arr[0], (Stock) arr[1], reviews));
```

This works, but it is less readable once you go past two futures, because you lose type safety along the way (notice the `Object[]` and casts). For three or more independent futures, prefer `allOf` with named local variables, as in the main example above. It reads cleanly and each future keeps its own type.

## How It Works

1. Three `supplyAsync` calls start three independent tasks on `ioExecutor`. They all begin running right away, in parallel, on separate pool threads. The calling thread does not block here — it gets three `CompletableFuture` objects immediately and moves on to set up the next steps.
2. Each future gets its own timeout and its own error handling, chained directly onto it. This means pricing, inventory, and reviews each fail independently, and each recovers in the way that makes sense for that specific data (pricing must fail the whole page; inventory falls back to "unknown"; reviews falls back to "empty").
3. `CompletableFuture.allOf` builds one future that completes only when all three are done — whether they succeeded or already recovered with a fallback value via `exceptionally`.
4. We call `.join()` on `allDone`, with one more `orTimeout` as an overall safety net. This is the one place the calling thread actually blocks and waits, for at most 3 seconds.
5. After `allDone` finishes, we know each of the three futures is done, so calling `.join()` on each one just reads its already-computed value — this does not block again.
6. If `priceFuture` failed even after its own `exceptionally` step re-threw a domain-specific exception, that exception surfaces from `allDone.join()`, wrapped in a `CompletionException`. We unwrap it with `.getCause()` and re-throw a clear error to the caller.

## How to Extend (Follow-ups)

Interviewers often push into these directions after the base solution:

**1. Collect results from a dynamic list of futures (not a fixed 3).**
If the number of calls is not fixed at compile time (for example, one call per item in a shopping cart), build a `List<CompletableFuture<Item>>`, then combine them:

```java
List<CompletableFuture<ItemDetails>> futures = cartItemIds.stream()
    .map(id -> CompletableFuture.supplyAsync(() -> fetchItemDetails(id), ioExecutor))
    .toList();

CompletableFuture<List<ItemDetails>> allResults = CompletableFuture
    .allOf(futures.toArray(new CompletableFuture[0]))
    .thenApply(v -> futures.stream().map(CompletableFuture::join).toList());
```

**2. Use anyOf for redundant calls (racing).**
If you call the same data from two mirrored services and want whichever answers first:

```java
CompletableFuture<Object> fastest = CompletableFuture.anyOf(replicaAFuture, replicaBFuture);
```

Remember `anyOf` returns `CompletableFuture<Object>`, so you need a cast, or you write a small typed helper method around it.

**3. Compare with `Future` + `ExecutorService`.**
Before `CompletableFuture` (added in Java 8), the standard way to run async tasks was `ExecutorService.submit(Callable<T>)`, which returns a plain `Future<T>`. A plain `Future` has real limits:

- No chaining. You cannot say "when this finishes, automatically run that next step." You must call the blocking `.get()` method yourself, which blocks the calling thread.
- No easy way to combine two or more futures. You would have to call `.get()` on each one, one after another, which is really sequential waiting, not true parallel composition.
- No built-in exception recovery step, like `exceptionally`. You wrap `.get()` in a try/catch and handle `ExecutionException` yourself, every time.
- No built-in timeout on the async pipeline itself (only `.get(timeout, unit)` blocks with a timeout, but that does not cancel or route to a fallback automatically).

`CompletableFuture` solves all of these with chaining methods (`thenApply`, `thenCompose`, `thenCombine`), combinators (`allOf`, `anyOf`), and built-in recovery and timeout methods (`exceptionally`, `handle`, `orTimeout`, `completeOnTimeout`). This is why `CompletableFuture` is almost always the better choice when you need to compose several async operations, not just run one task in the background.

**4. Other useful extensions:**
- Add structured logging inside `whenComplete` on the top-level future, so every request logs its success/failure and latency in one place, without touching the business logic in `thenApply`/`thenCompose` steps.
- For very large fan-out (hundreds of calls), consider a bounded executor and a semaphore to cap the number of truly concurrent in-flight calls, so you do not overwhelm a downstream service.
- Mention Java 21's virtual threads as an alternative for I/O-bound fan-out: with virtual threads, you can write plain blocking code (no `CompletableFuture` chaining) and still get high concurrency, because a virtual thread that blocks on I/O does not tie up an OS thread. `CompletableFuture` is still useful there for readable composition, but the performance reason for it (avoiding platform-thread exhaustion) matters less with virtual threads.

## Complexity & Thread-Safety Notes

| Aspect | Notes |
|---|---|
| Time (wall clock) | Close to the slowest of the parallel calls, not the sum of all calls, since they run concurrently on the pool. |
| Time (with timeout) | Bounded by `max(per-call timeouts, overall timeout)`. A hung call still occupies a pool thread until it truly finishes or the underlying client cancels it. |
| Space | O(number of in-flight futures). Each `CompletableFuture` is a small object; the real cost is the thread pool and any buffered request/response data. |
| Thread safety of `CompletableFuture` itself | `CompletableFuture` is thread-safe. Multiple threads can safely call `.complete()`, attach callbacks, or read the result on the same instance without external locking. |
| Callback execution thread | The non-`Async` chain methods (`thenApply`, `thenAccept`, and so on) may run on whichever thread completes the previous stage — this can be a pool thread, or even the thread that called `.complete()`. Use the `Async` variant with an explicit `Executor` whenever you need a guarantee about which pool runs the code, especially for blocking work. |
| Shared mutable state in callbacks | `CompletableFuture` does not protect you from bugs in your own callback code. If two chained callbacks both write to the same shared mutable object (for example, a shared `List` or `Map` outside the future's own data), you still need your own synchronization, since callbacks can run concurrently on different pool threads. |
| Executor sizing | Use a bounded, custom `Executor` for blocking I/O tasks. Never run blocking I/O on the default `ForkJoinPool.commonPool()`, since it is small and shared across the whole JVM. |

## Interview Tips & Common Mistakes

- **Say why you avoid the common pool for I/O.** This is one of the most valued things to say out loud: "I will pass my own `Executor` because the default `ForkJoinPool.commonPool()` is sized for CPU-bound work and shared JVM-wide; blocking I/O on it can starve unrelated code."
- **Know the difference between `thenApply` and `thenCompose` cold.** This is a very common follow-up question. `thenApply` is for a plain value transform. `thenCompose` is for when the next step is itself another async call (it flattens `CompletableFuture<CompletableFuture<T>>` into `CompletableFuture<T>`).
- **Know the difference between `exceptionally`, `handle`, and `whenComplete`.** `exceptionally` only fires on failure and returns a fallback value. `handle` fires on both outcomes and can change the result. `whenComplete` fires on both outcomes but never changes the result — only used for side effects like logging.
- **Do not call `.get()` or `.join()` too early.** A common mistake is calling `.get()` right after `supplyAsync`, which blocks the calling thread immediately and throws away all the benefit of running in parallel. Only block once, at the very end, after all independent work has been started and chained.
- **Remember `allOf` does not return combined results directly.** It returns `CompletableFuture<Void>`. You must read each original future yourself afterward (usually with `.join()`, which is now safe since `allOf` guarantees they are done).
- **Handle exceptions at the right level.** Deciding "does this one failure sink the whole operation, or can we use a fallback" is a design decision, not just a coding detail. Say this out loud, as shown with pricing (critical, fails the whole page) versus reviews (optional, falls back to empty).
- **Do not forget to shut down custom executors.** A `ThreadPoolExecutor` you create must be shut down (`shutdown()`) when your application or component stops, or its non-daemon threads can keep the JVM alive.
- **Mention virtual threads as a modern alternative, but do not confuse the two topics.** `CompletableFuture` is about composing async steps cleanly. Virtual threads (Java 21+) are about running blocking code cheaply at scale. They solve related but different problems, and a good answer keeps them distinct.
