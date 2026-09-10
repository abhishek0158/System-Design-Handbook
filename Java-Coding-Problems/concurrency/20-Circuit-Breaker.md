# Circuit Breaker

## Problem

A backend service often calls other services. For example, an order service calls a payment service. Sometimes the payment service becomes slow or fails. If the order service keeps calling it again and again, bad things happen:

- Threads in the order service get stuck waiting for a slow reply.
- The order service runs out of threads or connections.
- The order service also becomes slow or crashes, even though the real problem is in the payment service.

This is called a **cascading failure**. One failing service takes down other services that depend on it.

A **circuit breaker** is a pattern that stops this. It sits between the caller and the downstream service (the service being called). It watches for failures. When failures cross a limit, it "opens" and stops calling the downstream service for a while. Calls fail fast instead of waiting and failing slowly. After a cooldown time, it lets a few test calls through, to check if the downstream service has recovered.

The name comes from an electrical circuit breaker in a house. If there is too much current (a fault), the breaker trips and cuts the circuit. This stops the fault from damaging the rest of the house wiring. A software circuit breaker does the same thing for service calls.

In this problem, you must design and implement a thread-safe `CircuitBreaker` class in Java. It wraps a call (given as a `Supplier` or `Callable`). It tracks failures, and it moves between three states: CLOSED, OPEN, and HALF_OPEN.

## Requirements & Clarifying Questions

Ask these questions in an interview. They show that you think about real production behavior, not just the state diagram.

1. **What counts as a failure?** An exception thrown by the call? A timeout? An HTTP 5xx response? Usually you pass in the definition (for example, "any exception counts as a failure"), and the interviewer expects you to make this configurable.
2. **Failure threshold: count or rate?** Do we open the circuit after "5 failures in a row," or after "50% of the last 20 calls failed"? A simple count is easier to build first. A rate (over a sliding window) is more correct for a service with mixed traffic.
3. **How long does the circuit stay OPEN?** This is the "open timeout" or "cooldown period" (for example, 30 seconds). After this time, we move to HALF_OPEN to test the service again.
4. **How many trial calls in HALF_OPEN?** Do we allow only 1 trial call, or a small number (for example, 3)? What happens if some trial calls succeed and some fail?
5. **What happens on a rejected call (when OPEN)?** Do we throw an exception, return a default value, or call a fallback function? A fallback (for example, "return cached data" or "return a default price") is common and often expected.
6. **Is this per-downstream-service, or shared?** Usually one `CircuitBreaker` instance guards one downstream dependency (for example, one instance for the payment service, another for the inventory service).
7. **Single JVM or distributed?** A circuit breaker's state (CLOSED/OPEN/HALF_OPEN) is usually kept in-memory, per JVM instance. In a multi-instance deployment, each instance has its own circuit breaker and makes its own decision. This is normal and accepted — it does not need to be shared across instances like a rate limiter sometimes does.

For this solution sheet, we build: a count-based failure threshold first (simple, and it is what most interviews expect you to code from scratch), then explain how to extend it to a sliding-window rate-based version.

## Design / Approach

### The state machine

A circuit breaker has three states:

- **CLOSED**: normal state. Calls pass through to the real downstream service. We count failures. If failures reach the threshold, we move to OPEN.
- **OPEN**: the circuit has "tripped." Calls do not reach the downstream service. They fail fast (return a fallback, or throw an exception immediately) with no wait. We stay in OPEN for a fixed cooldown time (the "open timeout"). After the cooldown, we move to HALF_OPEN.
- **HALF_OPEN**: a trial state. We allow a small, limited number of calls to go through to the real service, to test if it has recovered.
    - If enough trial calls succeed, we move back to CLOSED and reset the failure count.
    - If any trial call fails (or enough of them fail, depending on policy), we move back to OPEN and restart the cooldown timer.

Here is the state transition diagram in words:

```
CLOSED --(failures >= threshold)--> OPEN
OPEN   --(cooldown time passed)--> HALF_OPEN
HALF_OPEN --(trial calls succeed)--> CLOSED
HALF_OPEN --(a trial call fails)--> OPEN
```

### Key design decisions

- We use an `enum` for the three states: `CLOSED`, `OPEN`, `HALF_OPEN`.
- We store the current state in an `AtomicReference<State>`, so reads and updates are safe without a full lock for the common read path.
- We use `AtomicInteger` for the failure count and the half-open trial counters, so increments are thread-safe and lock-free.
- We use a `volatile long` for "when did we last open the circuit," so every thread sees the latest value right away.
- The actual state **transition** (for example, CLOSED to OPEN, or checking "has the cooldown passed, so move to HALF_OPEN") uses a short `synchronized` block or a `ReentrantLock`. This is because a transition reads more than one field and must act on them together, as one atomic step. Using only atomics for this part can cause two threads to both trigger the same transition, or to disagree about the state.
- We wrap the call in a `Supplier<T>` (or `Callable<T>` if the call can throw a checked exception), so the `CircuitBreaker` is generic and works for any downstream call, not just one specific method.
- We accept an optional fallback function, called when the circuit is OPEN (or when the call still fails). This is what makes the circuit breaker usable in real code — the caller gets a safe value instead of an exception.

## Java Solution

```java
import java.util.concurrent.atomic.AtomicInteger;
import java.util.concurrent.atomic.AtomicReference;
import java.util.function.Supplier;

public class CircuitBreaker {

    public enum State { CLOSED, OPEN, HALF_OPEN }

    // ---- Configuration (set once, at construction time) ----
    private final int failureThreshold;       // failures allowed in CLOSED before we open
    private final long openTimeoutMillis;     // how long we stay OPEN before trying HALF_OPEN
    private final int halfOpenTrialLimit;     // number of trial calls allowed in HALF_OPEN
    private final int halfOpenSuccessToClose; // successes needed in HALF_OPEN to go back to CLOSED

    // ---- Mutable state (shared across threads) ----
    private final AtomicReference<State> state = new AtomicReference<>(State.CLOSED);
    private final AtomicInteger failureCount = new AtomicInteger(0);
    private final AtomicInteger halfOpenTrialsUsed = new AtomicInteger(0);
    private final AtomicInteger halfOpenSuccesses = new AtomicInteger(0);
    private volatile long lastOpenedAtMillis = 0L;

    // Guards state TRANSITIONS only (not every field read).
    private final Object transitionLock = new Object();

    public CircuitBreaker(int failureThreshold,
                           long openTimeoutMillis,
                           int halfOpenTrialLimit,
                           int halfOpenSuccessToClose) {
        this.failureThreshold = failureThreshold;
        this.openTimeoutMillis = openTimeoutMillis;
        this.halfOpenTrialLimit = halfOpenTrialLimit;
        this.halfOpenSuccessToClose = halfOpenSuccessToClose;
    }

    /**
     * Runs the given call through the circuit breaker.
     * If the circuit is OPEN (and cooldown has not passed), the call is
     * rejected immediately and the fallback is used instead.
     */
    public <T> T call(Supplier<T> primaryCall, Supplier<T> fallback) {
        if (!allowRequest()) {
            return fallback.get();
        }
        try {
            T result = primaryCall.get();
            onSuccess();
            return result;
        } catch (Exception e) {
            onFailure();
            return fallback.get();
        }
    }

    /**
     * Decides if a call is allowed right now, and performs the
     * OPEN -> HALF_OPEN transition if the cooldown has passed.
     */
    private boolean allowRequest() {
        State current = state.get();

        if (current == State.CLOSED) {
            return true;
        }

        if (current == State.OPEN) {
            long elapsed = System.currentTimeMillis() - lastOpenedAtMillis;
            if (elapsed < openTimeoutMillis) {
                return false; // still cooling down; fail fast
            }
            // Cooldown passed. Try to move OPEN -> HALF_OPEN.
            synchronized (transitionLock) {
                // Re-check inside the lock, in case another thread already moved us.
                if (state.get() == State.OPEN) {
                    state.set(State.HALF_OPEN);
                    halfOpenTrialsUsed.set(0);
                    halfOpenSuccesses.set(0);
                }
            }
            current = state.get();
        }

        if (current == State.HALF_OPEN) {
            // Allow only a limited number of trial calls through.
            int used = halfOpenTrialsUsed.incrementAndGet();
            if (used > halfOpenTrialLimit) {
                halfOpenTrialsUsed.decrementAndGet(); // give back the slot we did not use
                return false;
            }
            return true;
        }

        return false;
    }

    private void onSuccess() {
        State current = state.get();
        if (current == State.HALF_OPEN) {
            int successes = halfOpenSuccesses.incrementAndGet();
            if (successes >= halfOpenSuccessToClose) {
                synchronized (transitionLock) {
                    if (state.get() == State.HALF_OPEN) {
                        state.set(State.CLOSED);
                        failureCount.set(0);
                    }
                }
            }
        } else if (current == State.CLOSED) {
            // A success in CLOSED resets the failure count.
            // (Simple policy: any success clears the streak. See "sliding window"
            //  follow-up for a rate-based alternative.)
            failureCount.set(0);
        }
    }

    private void onFailure() {
        State current = state.get();

        if (current == State.HALF_OPEN) {
            // A single failed trial call sends us right back to OPEN.
            synchronized (transitionLock) {
                if (state.get() == State.HALF_OPEN) {
                    openCircuit();
                }
            }
            return;
        }

        if (current == State.CLOSED) {
            int failures = failureCount.incrementAndGet();
            if (failures >= failureThreshold) {
                synchronized (transitionLock) {
                    if (state.get() == State.CLOSED) {
                        openCircuit();
                    }
                }
            }
        }
        // If already OPEN, a failure here does nothing extra (should not happen,
        // since allowRequest() rejects calls while OPEN).
    }

    // Must be called while holding transitionLock.
    private void openCircuit() {
        state.set(State.OPEN);
        lastOpenedAtMillis = System.currentTimeMillis();
        failureCount.set(0);
    }

    public State getState() {
        return state.get();
    }
}
```

## How It Works

A caller uses the circuit breaker like this:

```java
CircuitBreaker paymentServiceBreaker =
        new CircuitBreaker(
                5,      // open after 5 failures in a row
                30_000, // stay OPEN for 30 seconds
                3,      // allow 3 trial calls in HALF_OPEN
                2       // need 2 successful trial calls to go back to CLOSED
        );

String result = paymentServiceBreaker.call(
        () -> paymentClient.charge(orderId),      // primary call
        () -> "FALLBACK: payment queued for retry" // fallback
);
```

Walk through the flow:

1. **CLOSED state.** Every call to `call(...)` goes to `allowRequest()`, which returns `true` right away, since `current == State.CLOSED`. The real call runs. If it throws, `onFailure()` increments `failureCount`. Once `failureCount` reaches the threshold, we grab `transitionLock` and flip to OPEN, recording `lastOpenedAtMillis`.
2. **OPEN state.** Now `allowRequest()` checks the elapsed time since `lastOpenedAtMillis`. While elapsed time is less than `openTimeoutMillis`, every call is rejected immediately — no real call happens. The fallback runs instead. This is the "fail fast" behavior. The caller does not wait for a timeout on the slow downstream service; it gets an answer instantly.
3. **Cooldown passes.** Once enough time has passed, the next call to `allowRequest()` acquires `transitionLock` and moves state to HALF_OPEN, resetting the trial counters.
4. **HALF_OPEN state.** A limited number of calls (`halfOpenTrialLimit`) are allowed through as **trial calls**. Each call to `allowRequest()` in this state increments `halfOpenTrialsUsed` with `incrementAndGet()`. If the count exceeds the limit, the call is rejected and the reserved slot is given back with `decrementAndGet()`.
    - If a trial call **succeeds**, `onSuccess()` increments `halfOpenSuccesses`. Once enough trials succeed, we move back to CLOSED and reset `failureCount` to zero. The service is trusted again.
    - If a trial call **fails**, `onFailure()` immediately sends the breaker back to OPEN and restarts the cooldown timer. We do not wait for more trials to fail — one failure during the trial period is enough proof the service is still broken.

**Why `synchronized` around transitions, but atomics elsewhere?** Reading a single `AtomicReference` or `AtomicInteger` value is fast and lock-free, and it is safe for the common case of "just check the current state." But a transition (for example, "if state is OPEN and cooldown passed, become HALF_OPEN and reset two counters") touches multiple fields together, and it must happen exactly once, even if ten threads reach that code at the same instant. The `synchronized (transitionLock)` block, combined with a re-check of the state right after acquiring the lock ("if (state.get() == State.OPEN)"), makes sure only the first thread through actually performs the transition. The other threads see the already-updated state and skip it. This pattern is called **double-checked locking** style re-verification, and it prevents duplicate transitions.

## How to Extend (Follow-ups)

Interviewers commonly ask you to extend the basic version. Here are the usual directions.

**1. Sliding-window failure counting (rate-based, not just a raw count).**

A raw failure count (like our `failureThreshold`) has a problem: if a service handles 1,000 calls per second and 5 of them fail, that is not a real problem, but our breaker would still open after 5 failures. A better approach tracks a **failure rate** over a rolling window of recent calls, for example, "open if 50% or more of the last 20 calls failed."

One clean way to build this: keep a fixed-size circular buffer (an array) of the last N call outcomes (`true` = success, `false` = failure), plus a running count of failures in the buffer. On each new call outcome, overwrite the oldest slot, adjust the running failure count up or down as needed, and check the failure count against N to compute a rate. Guard the buffer and the running count with a single lock (or use `LongAdder` plus a time-bucketed approach, similar to the Sliding Window Counter rate limiter pattern) so updates stay consistent under concurrent calls. This is more complex than a plain counter, but it is much closer to what production libraries actually do.

**2. Combine circuit breaker with retry and timeout.**

In real systems, these three patterns work together, and the order matters:

- **Timeout** wraps the single call, so it does not wait forever for a reply (for example, using `CompletableFuture.orTimeout(...)` or a client library's built-in timeout).
- **Retry** wraps the timeout-protected call, and tries again a few times on failure, usually with a backoff delay between attempts (do not retry instantly, and do not retry forever).
- **Circuit breaker** wraps the retry, so that once retries are clearly not helping (the service is down, not just briefly slow), we stop trying altogether and fail fast, instead of every caller retrying uselessly against a dead service.

A common bug: putting the circuit breaker *inside* the retry loop. This causes one logical call to count as many failures (one per retry attempt), which trips the breaker too early. The circuit breaker should wrap the whole retry-with-timeout unit, so one logical call equals one failure or one success from the breaker's point of view.

**3. How Resilience4j (and similar libraries) do it.**

Resilience4j is a popular Java resilience library. Its `CircuitBreaker` is close to what we built, with these refinements:

- It uses a sliding window by default (either count-based or time-based), not a simple consecutive-failure count.
- It supports a `slowCallDurationThreshold` — a call that takes too long, even if it "succeeds," can count as a failure. This catches services that are alive but degraded (very slow), which a plain success/failure check would miss.
- Its HALF_OPEN state has a configurable "permitted number of calls," similar to our `halfOpenTrialLimit`.
- It exposes events (state transition events, call events) so you can wire up metrics and alerts (for example, to Micrometer or Prometheus).
- It integrates with `CompletableFuture` and reactive types, not just plain synchronous calls.

Mentioning these details in an interview shows you know the difference between a hand-built teaching version and a production-grade implementation.

**4. Other useful extensions.**

- Add a `getMetrics()` method that reports current state, failure count, and time until the next HALF_OPEN attempt — useful for a health dashboard.
- Support per-downstream-service breakers, held in a `ConcurrentHashMap<String, CircuitBreaker>`, similar to how a rate limiter keys state per client.
- Allow a manual override, so an operator can force the circuit OPEN (for planned maintenance on the downstream service) or force it CLOSED.

## Complexity & Thread-Safety Notes

- **Time per call:** O(1). Every check (`allowRequest`, `onSuccess`, `onFailure`) does a constant number of atomic reads/writes, plus at most one short `synchronized` block during a state transition. There are no loops over stored data in the basic (count-based) version.
- **Space:** O(1). We store a small, fixed number of fields — no per-call data is kept (unlike the sliding-window follow-up, which is O(N) for a window of N recent outcomes).
- **Why atomics are not enough by themselves:** `AtomicInteger.incrementAndGet()` is safe for updating one counter. But a state transition must check a condition and update several fields together, as a single unit. If we used only atomics with no lock, two threads could both see `failures >= threshold` and both try to open the circuit, or worse, race between reading the state and reading `lastOpenedAtMillis`, leading to an inconsistent view. The short `synchronized (transitionLock)` block, plus a re-check of the condition right after acquiring the lock, fixes this without needing a lock around every single field read.
- **Lock granularity:** We lock only during the rare event of a state transition, not during every call. Most calls in CLOSED or OPEN state complete without ever touching `transitionLock`. This keeps the breaker fast under normal load, since the lock is a bottleneck only during the brief moments when the state actually changes.
- **Visibility:** `AtomicReference<State>` and `AtomicInteger` give both atomicity (safe read-modify-write) and visibility (a write by one thread is guaranteed to be seen by other threads' later reads) due to their use of `volatile`-like semantics internally. `lastOpenedAtMillis` is declared `volatile` for the same reason — a plain (non-volatile) `long` field would not guarantee that other threads see a freshly written value right away.
- **No blocking on the hot path:** Rejected calls in the OPEN state return immediately (fallback runs right away). This is the whole point of "fail fast" — we never make a caller wait on a lock or on the network when we already know the downstream service is failing.

## Interview Tips & Common Mistakes

- **Draw the state diagram first, out loud, before coding.** Interviewers want to see that you understand CLOSED, OPEN, and HALF_OPEN as a proper state machine, with clear conditions for each transition, before you write a single line of Java.
- **Explain "fail fast" clearly.** The main value of a circuit breaker is not preventing failures — it is preventing a *pile-up* of slow, doomed calls. Say this directly: "when OPEN, we skip the real call entirely, so the caller gets an instant answer instead of waiting for a timeout."
- **Do not confuse a circuit breaker with a retry mechanism.** Retry means "try the same call again, hoping it works this time." Circuit breaker means "stop trying, because we already know it will not work right now." They solve different problems and are normally used together, not as substitutes for each other.
- **Do not put the whole method body in one big `synchronized` block.** This is the most common mistake. It works, but it turns the circuit breaker into a bottleneck, since every call — even simple state reads while CLOSED — would wait on a shared lock. Use atomics for the fast, common path, and a lock only for the rare transition logic.
- **Handle the HALF_OPEN trial limit carefully.** A common bug is letting unlimited calls through during HALF_OPEN, instead of a small, fixed number. If you allow every call through during HALF_OPEN, you lose the benefit of "testing carefully" — a spike of concurrent traffic could send many calls to a barely-recovered service all at once.
- **Be ready to say why the breaker's state is usually per-JVM-instance, not shared.** Unlike a distributed rate limiter, most circuit breaker implementations keep state local to one instance, since the goal is protecting that instance's threads and connections from a slow dependency. If asked "would you put this in Redis," the honest answer is usually "no, not normally — each instance should make its own fail-fast decision based on what it sees."
- **Mention the "half-open success count" detail.** A single successful trial call is often not proof the service fully recovered. Requiring 2 or 3 successes (a small threshold, using `halfOpenSuccessToClose` in our code) before fully closing the circuit is a more careful, production-realistic choice than closing on the very first success.
- **Know the Resilience4j comparison.** Even a short, accurate comparison ("Resilience4j uses a sliding window and also tracks slow calls, not just failures") signals you have looked at a real production library, not just a textbook diagram.
