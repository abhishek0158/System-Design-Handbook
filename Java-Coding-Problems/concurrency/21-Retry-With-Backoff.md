# Retry with Backoff

## Problem

Design and build a reusable **Retry** helper in Java. It should wrap a piece of code (a `Callable` or `Supplier`) and retry that code if it fails with a temporary error.

Here is why this matters. In a backend system, calls often go over a network. Examples: calling another microservice, calling a database, calling a payment gateway. Networks are not perfect. Sometimes a call fails for a short time only. Reasons include:

- A brief network glitch (a dropped packet, a DNS hiccup).
- The other service is briefly overloaded and returns a "server busy" error.
- A connection pool was temporarily full.
- A load balancer routed the request to an instance that was just restarting.

These are **transient failures** (transient means "short-lived, temporary"). If we retry the same call a moment later, it often succeeds. So retry logic is a normal part of any resilient backend service.

But retries are dangerous if done carelessly. If a service is already struggling (for example, it is overloaded and slow), and thousands of client instances all retry immediately and repeatedly, the retries add even more load. This can turn a small problem into a full outage. This effect is called a **retry storm**: many clients hammer a weak service with retries, and the service never gets a chance to recover.

So the real task is not just "try again on failure." The real task is "retry in a safe, controlled way." That is what this problem is about.

## Requirements & Clarifying Questions

Before coding, an interviewer expects you to ask questions and state assumptions. Good questions to ask:

1. **What counts as a retryable error?** Should we retry on every exception, or only on specific ones (for example, network timeout, HTTP 503 "Service Unavailable")? Should we skip retry on a client error like HTTP 400 "Bad Request", because retrying the same bad input will never succeed?
2. **Is the operation idempotent?** Idempotent means: running it many times has the same effect as running it once. We should only retry idempotent operations. (More on this below.)
3. **How many retries are allowed?** We need a maximum attempt count, so we do not retry forever.
4. **What delay strategy do we use between attempts?** Fixed delay (same wait every time) or exponential backoff (wait time doubles each time)?
5. **Do we need jitter?** Jitter means adding a small random amount to the delay, so many clients do not retry at the exact same moment.
6. **Is there a maximum delay cap?** Without a cap, exponential backoff can grow to minutes or hours, which is too slow for most use cases.
7. **Should this run synchronously (blocking the calling thread) or asynchronously (using a separate thread pool)?** For this problem, we assume a simple synchronous helper, since that is what most interviews ask for first. We mention the async version in follow-ups.

Stated assumptions for this solution:

- The helper wraps a `Callable<T>` (a piece of code that returns a value and can throw a checked exception).
- We use **exponential backoff with jitter**, because it is the industry-standard strategy.
- We let the caller decide which exceptions are retryable, using a `Predicate<Exception>`.
- We add a maximum delay cap.
- We only retry operations the caller has confirmed are safe to repeat (idempotent).

## Design / Approach

The helper has four parts:

1. **Retry policy (configuration)**: max attempts, base delay, max delay, and a multiplier for exponential growth.
2. **Retryable check**: a function that looks at the exception and decides "should we retry this, or fail immediately?"
3. **Backoff calculator**: computes the delay before the next attempt, using exponential growth plus jitter, capped at a maximum value.
4. **Retry loop**: runs the task, catches exceptions, checks if retryable, sleeps for the backoff delay, and tries again — until success or until attempts run out.

### Why exponential backoff?

With **fixed delay**, we wait the same time (say, 500 ms) between every retry. This is simple, but it does not help much if the target service needs more time to recover from real overload.

With **exponential backoff**, the delay doubles each time: 500 ms, 1000 ms, 2000 ms, 4000 ms, and so on. This gives the failing service more and more breathing room as failures continue. It reduces load on a struggling service much faster than fixed delay.

### Why jitter?

Imagine 1,000 client instances all call a service at the same time, and the service fails for all of them. Without jitter, all 1,000 clients compute the exact same backoff delay (say, 2000 ms) and all retry at the exact same moment. This creates a new burst of load — a "thundering herd." The service gets hit by a wave of retries all at once, again and again, at predictable intervals.

**Jitter** adds a random amount to the delay, so each client waits a slightly different time. This spreads the retries out over a time window instead of a single instant. The most common approach is called "full jitter": pick a random delay between 0 and the computed exponential delay. This is the approach we use below.

### Why cap the maximum delay?

If we let the delay double without limit — 500 ms, 1s, 2s, 4s, 8s, 16s, 32s, 64s... — after just a few attempts, the wait becomes minutes long. Most callers (for example, a user waiting for an API response) cannot wait that long. So we cap the delay at some fixed maximum, like 10 or 30 seconds. After the cap is reached, the delay stays flat at the cap (still with jitter applied).

### Why check "is this exception retryable"?

Not all errors are temporary. A 400 Bad Request means the caller sent invalid data. Retrying the same invalid data will always fail the same way — it wastes time and load. Similarly, a security error (401 Unauthorized) will not fix itself by retrying. We should retry only on errors that are likely temporary, such as timeouts, connection failures, or 503 Service Unavailable. This is why the retry helper takes a `Predicate<Exception>` that decides retryable vs non-retryable.

## Java Solution

```java
import java.util.concurrent.Callable;
import java.util.concurrent.ThreadLocalRandom;
import java.util.function.Predicate;

/**
 * Configuration for a retry attempt sequence.
 */
public final class RetryPolicy {

    private final int maxAttempts;      // total attempts, including the first try
    private final long baseDelayMillis; // delay used for attempt #1's backoff
    private final long maxDelayMillis;  // upper cap on any single delay
    private final double multiplier;    // exponential growth factor, usually 2.0

    public RetryPolicy(int maxAttempts, long baseDelayMillis,
                        long maxDelayMillis, double multiplier) {
        if (maxAttempts < 1) {
            throw new IllegalArgumentException("maxAttempts must be >= 1");
        }
        if (baseDelayMillis < 0 || maxDelayMillis < baseDelayMillis) {
            throw new IllegalArgumentException("invalid delay bounds");
        }
        this.maxAttempts = maxAttempts;
        this.baseDelayMillis = baseDelayMillis;
        this.maxDelayMillis = maxDelayMillis;
        this.multiplier = multiplier;
    }

    public int getMaxAttempts() {
        return maxAttempts;
    }

    /**
     * Computes the exponential delay for a given attempt number (1-based),
     * before jitter is applied, capped at maxDelayMillis.
     */
    public long exponentialDelay(int attempt) {
        double raw = baseDelayMillis * Math.pow(multiplier, attempt - 1);
        // Guard against overflow when attempt is large.
        if (raw >= maxDelayMillis) {
            return maxDelayMillis;
        }
        return (long) raw;
    }

    /**
     * Applies "full jitter": a random value between 0 and the capped
     * exponential delay. This spreads out retries from many clients.
     */
    public long delayWithJitter(int attempt) {
        long capped = exponentialDelay(attempt);
        if (capped <= 0) {
            return 0;
        }
        return ThreadLocalRandom.current().nextLong(capped + 1);
    }
}
```

```java
/**
 * Thrown when all retry attempts are used up and the task still fails.
 * Wraps the last exception seen, so callers do not lose the root cause.
 */
public class RetryExhaustedException extends RuntimeException {
    public RetryExhaustedException(String message, Throwable cause) {
        super(message, cause);
    }
}
```

```java
import java.util.concurrent.Callable;
import java.util.function.Predicate;

/**
 * Reusable retry helper with exponential backoff and jitter.
 * Thread-safe: holds no mutable shared state; each call() invocation
 * is independent.
 */
public final class Retry {

    private final RetryPolicy policy;
    private final Predicate<Exception> isRetryable;

    public Retry(RetryPolicy policy, Predicate<Exception> isRetryable) {
        this.policy = policy;
        this.isRetryable = isRetryable;
    }

    /**
     * Runs the given task. Retries on failure according to the policy,
     * but only if the exception is judged retryable.
     *
     * @throws RetryExhaustedException if all attempts fail
     * @throws Exception if a non-retryable exception is thrown by the task
     */
    public <T> T call(Callable<T> task) throws Exception {
        Exception lastException = null;

        for (int attempt = 1; attempt <= policy.getMaxAttempts(); attempt++) {
            try {
                return task.call();
            } catch (Exception ex) {
                lastException = ex;

                boolean lastAttempt = (attempt == policy.getMaxAttempts());
                if (!isRetryable.test(ex) || lastAttempt) {
                    // Either the error is permanent, or we are out of tries.
                    // Do not sleep; fail now.
                    break;
                }

                long delay = policy.delayWithJitter(attempt);
                sleepQuietly(delay);
                // Loop continues to the next attempt.
            }
        }

        if (lastException instanceof RuntimeException && !isRetryable.test(lastException)) {
            // A non-retryable error should surface as-is, not wrapped.
            throw lastException;
        }
        throw new RetryExhaustedException(
                "All " + policy.getMaxAttempts() + " attempts failed", lastException);
    }

    private void sleepQuietly(long millis) {
        if (millis <= 0) {
            return;
        }
        try {
            Thread.sleep(millis);
        } catch (InterruptedException ie) {
            // Preserve the interrupt flag so calling code and thread pools
            // can react correctly; do not swallow it silently.
            Thread.currentThread().interrupt();
            throw new RetryExhaustedException("Retry interrupted", ie);
        }
    }
}
```

Example usage:

```java
RetryPolicy policy = new RetryPolicy(
        5,      // max attempts
        200,    // base delay: 200 ms
        5000,   // max delay cap: 5 seconds
        2.0     // multiplier: delay doubles each time
);

Predicate<Exception> retryableErrors = ex ->
        ex instanceof java.net.SocketTimeoutException
        || ex instanceof java.io.IOException
        || (ex instanceof ServiceException
            && ((ServiceException) ex).getStatusCode() == 503);

Retry retry = new Retry(policy, retryableErrors);

String result = retry.call(() -> paymentServiceClient.checkStatus(orderId));
```

## How It Works

The `call` method runs the task inside a loop, from attempt 1 up to `maxAttempts`.

- If the task succeeds, we return the result right away. No retry needed.
- If the task throws, we save the exception and check two things: is it retryable, and is this the last allowed attempt? If either answer stops us (not retryable, or no attempts left), we break out of the loop without sleeping.
- Otherwise, we compute a delay using `delayWithJitter`. This delay grows exponentially with the attempt number (`baseDelay * multiplier^(attempt-1)`), capped at `maxDelayMillis`, and then a random jitter is applied (a random value between 0 and that capped delay). We sleep for that long, then loop again for the next attempt.
- If we exit the loop without returning, all attempts failed. We throw `RetryExhaustedException`, wrapping the last real exception as the cause, so nothing is silently lost. If the very last failure was a non-retryable error, we rethrow it directly, since wrapping it would hide the true reason (for example, a 400 Bad Request should look like a 400 error to the caller, not like "retry exhausted").

Numeric example with `baseDelay = 200`, `multiplier = 2.0`, `maxDelay = 5000`:

| Attempt | Exponential delay (before jitter) | Actual delay (jitter, random) |
|---|---|---|
| 1 | 200 ms | random(0, 200) |
| 2 | 400 ms | random(0, 400) |
| 3 | 800 ms | random(0, 800) |
| 4 | 1600 ms | random(0, 1600) |
| 5 | 3200 ms | random(0, 3200) |
| 6 | 5000 ms (capped) | random(0, 5000) |

### Idempotency: the most important rule

**Idempotency** means: calling an operation two or more times has the same effect as calling it once. Retry logic is only safe on idempotent operations.

- **Safe to retry**: `GET /orders/123` (just reads data), `PUT /users/42 {name: "Alice"}` (setting a field to a fixed value is safe even if repeated), a database read query.
- **Dangerous to retry blindly**: `POST /payments` (charges a credit card). If the first request actually succeeded on the server, but the response was lost on the way back to the client (a real, common failure mode), a naive retry will charge the customer twice.

The fix used in real systems is an **idempotency key**: the client generates a unique ID for the operation (for example, a UUID) and sends it with the request. The server stores which idempotency keys it has already processed. If the same key arrives again, the server returns the original result instead of doing the action again. This makes even a "create payment" call safe to retry. As a rule: before adding retry to any write operation, check that it is naturally idempotent or has idempotency-key support. If not, do not retry it automatically — surface the error and let a human or a higher-level workflow decide.

## How to Extend (Follow-ups)

**1. Combine with a circuit breaker and a timeout.**
Retry alone can still make things worse if the target service is down for a long time (not just briefly slow). A **circuit breaker** tracks recent failure rates. If failures cross a threshold, the breaker "opens" and blocks calls for a cool-down period, failing fast instead of retrying. This protects both the caller (no wasted time) and the callee (no added load while it is down). A **timeout** on each individual attempt is also needed — without it, a single hanging call could block far longer than the whole retry budget allows, since `Thread.sleep` between attempts does not limit how long the task itself may run. A recommended order is: timeout wraps each attempt, retry wraps the timeout, and circuit breaker wraps the retry (or sits in front of it), so a fast-failing breaker skips retries entirely.

**2. Retry budgets.**
A budget limits the total amount of "extra" retry traffic that a service instance (or the whole client fleet) is allowed to send, for example, "retries may add at most 10% more request volume." This is a defense used at scale, when even individually well-behaved retry logic can still add up to too much aggregate traffic across thousands of instances. It is usually done with a shared counter or a token-bucket structure at the client library level.

**3. How this is done in real libraries.**
In real projects, do not hand-write retry logic like above for production. Use a proven library.

- **Resilience4j** offers `Retry`, `CircuitBreaker`, `TimeLimiter`, and `RateLimiter` modules that can be chained together (called "decorators"). Its `IntervalFunction.ofExponentialRandomBackoff(...)` gives exponential backoff with jitter out of the box.
- **Spring Retry** offers the `@Retryable` annotation with `@Backoff(delay = 200, multiplier = 2, maxDelay = 5000)`, plus `@Recover` for a fallback method when retries are exhausted.

Knowing these libraries exist, and mentioning them, shows the interviewer you understand this is a solved, well-studied problem in real systems — while still being able to write the core logic by hand when asked.

## Complexity & Thread-Safety Notes

- **Time complexity**: the retry loop itself does O(1) work per attempt (excluding the task's own cost), so total cost is O(maxAttempts) attempts, plus the sum of sleep delays between them.
- **Space complexity**: O(1) extra space; we only keep the last exception.
- **Thread-safety**: the `Retry` and `RetryPolicy` classes hold no mutable shared state, so a single `Retry` instance can be safely reused and called concurrently from many threads. Each call to `call(...)` uses only local variables. `ThreadLocalRandom.current()` is used for jitter instead of a shared `Random` instance, which avoids lock contention when many threads generate random delays at the same time.
- **Blocking behavior**: `Thread.sleep` blocks the calling thread. This is fine for a background worker thread, but it is not appropriate inside code that must stay non-blocking (for example, inside a Netty event loop or a reactive pipeline). In those cases, use a scheduled executor to delay the next attempt instead of sleeping, so the thread is freed up in between attempts.
- **Interrupt handling**: if the thread is interrupted during the sleep, we restore the interrupt flag with `Thread.currentThread().interrupt()`. This is important — swallowing an interrupt silently can break cooperative cancellation elsewhere in the application (for example, a thread pool shutdown that relies on interrupts to stop running tasks).

## Interview Tips & Common Mistakes

- **Do not forget jitter.** Many candidates implement plain exponential backoff and stop there. Bringing up the retry storm problem and adding jitter on your own, without being asked, is a strong signal of real production experience.
- **Do not retry everything.** Retrying a 400 Bad Request or an authentication failure is a common, serious mistake. Always mention the retryable-exception check.
- **Always mention idempotency.** This is the single most common gap in interview answers about retries. State clearly: "I would only apply this retry wrapper to idempotent calls, or calls that support an idempotency key."
- **Cap the maximum delay.** Forgetting the cap means backoff can grow unbounded, which is unrealistic for user-facing systems.
- **Handle `InterruptedException` correctly.** Do not swallow it. Restore the interrupt flag, as shown above.
- **Mention the difference between per-attempt timeout and total retry time budget.** Some interviewers will push on this: "what if the whole retry sequence still takes 30 seconds and the caller only wants to wait 2 seconds?" A good answer combines an overall deadline (stop retrying once total elapsed time is close to a hard budget) with the per-attempt timeout, not just per-attempt logic in isolation.
- **Be ready to extend the demo to `CompletableFuture`.** Some interviewers ask for an asynchronous version. The core idea (attempt, check retryable, compute backoff, schedule next attempt) stays the same; only the mechanism for "waiting" changes from `Thread.sleep` to a `ScheduledExecutorService` that completes a future after the delay.
