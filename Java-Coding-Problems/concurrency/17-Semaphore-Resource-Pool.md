# Semaphore-based Resource Limiter

## Problem

Design a component that limits how many threads can use a resource at the same time.

Example: your service calls a slow downstream API. The downstream API can handle only 5 calls at once from you. If you send more, it slows down or fails. You need a "limiter" that lets at most 5 threads call it at the same time. Other threads must wait, or give up after a timeout.

This is a common interview question. It tests if you know `java.util.concurrent.Semaphore`, and if you can avoid common bugs with permits.

## Requirements & Clarifying Questions

Before coding, ask these questions. They show you think about edge cases, not just the happy path.

1. What is the limit? Is it fixed (say, 5), or can it change at runtime?
2. Should a caller wait forever if no slot is free, or should it time out?
3. What happens if the task inside the limiter throws an exception? Must the slot still be freed?
4. Do we need fairness? Fairness means threads get a slot in the order they asked for it (first-come, first-served). Without fairness, a thread can "jump the queue" if it asks at the right moment. Fairness costs some performance.
5. Is this limiter used by one JVM (single process), or across multiple servers? A `Semaphore` only works inside one JVM. Across servers, you need a distributed limiter (for example, Redis-based).
6. Do we need metrics, like how many permits are free right now, or how many threads are waiting?

For this problem, we assume: a fixed limit set at creation time, support for both blocking wait and timed wait, guaranteed release even on exception, and single-JVM scope.

## Design / Approach

### What is a Semaphore?

A `Semaphore` is a counter that controls how many threads can enter a section of code at the same time. Think of it as a set of "permits" (tickets). A thread must get a permit before it enters. When it leaves, it gives the permit back.

- `acquire()` — take one permit. If no permit is free, the thread waits (blocks).
- `release()` — give one permit back.
- The counter starts at some number N. N is the maximum number of threads allowed in at once.

### Counting semaphore vs binary semaphore

A **counting semaphore** starts with N permits, where N can be any positive number (1, 5, 100, ...). It allows up to N threads in at the same time. This is the general case, and it is what `java.util.concurrent.Semaphore` gives you.

A **binary semaphore** is a special case where N = 1. Only one thread can hold the permit at a time. It looks like a lock, but it is not the same thing (see next section).

### How a Semaphore differs from a Lock

This is a key interview point. Many candidates confuse the two.

| | Lock (e.g. `ReentrantLock`) | Semaphore |
|---|---|---|
| Ownership | Owned by the thread that locked it. Only that thread can unlock it. | Not owned by any thread. Any thread can call `release()`, even a thread that never called `acquire()`. |
| Purpose | Mutual exclusion — protect a shared variable so only one thread touches it at a time. | Limit concurrency — allow up to N threads into a section. |
| Reentrancy | `ReentrantLock` lets the same thread lock it again without deadlock. | `Semaphore` has no owner, so "reentrant acquire" makes no sense the same way. If a thread calls `acquire()` twice, it takes two permits. |
| Typical count | Always 1 (exclusive access). | Any N ≥ 1. |

In short: a lock protects data from being changed by two threads at once. A semaphore controls how many threads may run a piece of code at once. A binary semaphore (N=1) can act like a lock, but since it has no owner, any thread can release it. This makes it unsafe to use as a lock replacement unless you design carefully.

### tryAcquire with timeout

`acquire()` blocks forever if no permit is free. In real systems, we often do not want a thread to wait forever. We use `tryAcquire(timeout, unit)` instead. It returns `true` if it got a permit within the time limit, or `false` if it timed out. The caller then decides what to do — for example, reject the request, or retry later.

### Fairness

`Semaphore` has a constructor `Semaphore(int permits, boolean fair)`.

- **Non-fair** (default, `fair = false`): when a permit becomes free, the JVM may give it to any waiting thread, or even to a new thread that just called `acquire()` and did not wait at all. This is called "barging." It gives higher throughput, because the JVM does not need to manage a strict queue.
- **Fair** (`fair = true`): permits go to threads in the order they requested them (FIFO — first in, first out). This avoids starvation (a thread waiting forever while others keep jumping ahead), but it is slower, because of the extra bookkeeping.

Rule of thumb for interviews: use non-fair by default. Use fair only if you have evidence that some threads are starved (waiting much longer than others) under non-fair mode.

### Classic bugs to avoid

1. **Forgetting to release on exception.** If the code between `acquire()` and `release()` throws an exception, and `release()` is not in a `finally` block, the permit is lost forever. Over time, all permits leak away, and the limiter deadlocks — no thread can get in.
2. **Releasing more permits than you acquired.** Since a `Semaphore` has no owner, nothing stops you from calling `release()` an extra time by mistake (for example, in a retry loop, or a duplicate `finally` block). This raises the permit count above the intended limit, silently breaking your concurrency guarantee. There is no exception thrown — the bug is silent and hard to find.
3. **Calling `release()` without a matching `acquire()`** (for example, in an `catch` block that runs even when `acquire()` itself failed). Always release only permits you are sure you acquired.
4. **Using `Semaphore` across JVMs and expecting it to work.** It only limits threads inside the same JVM process. For multiple servers, you need a distributed semaphore (Redis, ZooKeeper, database row lock, etc.).

The safe pattern is always:

```java
semaphore.acquire();
try {
    // use the resource
} finally {
    semaphore.release();
}
```

## Java Solution

Below is a small, reusable `ResourceLimiter` class. It wraps a `Semaphore` and gives a clean API: run a task only if a permit is available, with an optional timeout. It guarantees the permit is always released, even if the task throws.

```java
import java.util.concurrent.Callable;
import java.util.concurrent.Semaphore;
import java.util.concurrent.TimeUnit;
import java.util.concurrent.TimeoutException;

/**
 * Limits how many callers can run a piece of work at the same time.
 * Thread-safe. Safe to share one instance across many threads.
 */
public final class ResourceLimiter {

    private final Semaphore semaphore;

    /**
     * @param maxConcurrent max number of threads allowed to run at once
     * @param fair          true = FIFO order for waiting threads (avoids starvation,
     *                      slightly slower); false = higher throughput, no order guarantee
     */
    public ResourceLimiter(int maxConcurrent, boolean fair) {
        if (maxConcurrent <= 0) {
            throw new IllegalArgumentException("maxConcurrent must be positive");
        }
        this.semaphore = new Semaphore(maxConcurrent, fair);
    }

    /**
     * Runs the given task, blocking the caller until a permit is free.
     * Always releases the permit, even if the task throws.
     */
    public <T> T runBlocking(Callable<T> task) throws Exception {
        semaphore.acquire();               // waits forever until a permit is free
        try {
            return task.call();
        } finally {
            semaphore.release();           // ALWAYS release, success or failure
        }
    }

    /**
     * Runs the given task only if a permit becomes free within the timeout.
     * Throws TimeoutException if no permit was free in time.
     * Always releases the permit if it was acquired, even if the task throws.
     */
    public <T> T runWithTimeout(Callable<T> task, long timeout, TimeUnit unit)
            throws Exception {
        boolean acquired = semaphore.tryAcquire(timeout, unit);
        if (!acquired) {
            throw new TimeoutException(
                "Could not get a permit within " + timeout + " " + unit);
        }
        try {
            return task.call();
        } finally {
            semaphore.release();           // release only because acquire succeeded
        }
    }

    /** Number of permits currently free. Useful for metrics/monitoring. */
    public int availablePermits() {
        return semaphore.availablePermits();
    }

    /** Number of threads currently waiting for a permit. Useful for monitoring. */
    public int queueLength() {
        return semaphore.getQueueLength();
    }
}
```

Example usage — limiting concurrent downloads to 5:

```java
ResourceLimiter downloadLimiter = new ResourceLimiter(5, false);

public byte[] downloadFile(String url) throws Exception {
    return downloadLimiter.runBlocking(() -> {
        // real download logic here
        return httpClient.get(url);
    });
}

public byte[] downloadFileOrFail(String url) throws Exception {
    // give up if no slot is free within 2 seconds
    return downloadLimiter.runWithTimeout(
        () -> httpClient.get(url),
        2, TimeUnit.SECONDS
    );
}
```

## How It Works

1. `ResourceLimiter` creates one `Semaphore` with a fixed number of permits (`maxConcurrent`). This number equals the maximum number of threads allowed to run the task at the same time.
2. `runBlocking()` calls `semaphore.acquire()`. If a permit is free, the thread takes it and moves on immediately. If not, the thread sleeps (blocks) until another thread calls `release()`.
3. The task runs inside a `try` block. The `finally` block calls `release()` no matter what happens — normal return, checked exception, or unchecked exception (like `RuntimeException`). This is the most important line in the whole class. Without it, permits leak.
4. `runWithTimeout()` uses `tryAcquire(timeout, unit)` instead of `acquire()`. This method returns `false` if it could not get a permit in time, instead of waiting forever. We turn that `false` into a `TimeoutException` so the caller gets a clear signal.
5. Note the order of operations in `runWithTimeout`: we only call `release()` inside the `try/finally` that starts *after* `tryAcquire` succeeded. If `tryAcquire` returns `false`, we never took a permit, so we must not release one. This avoids bug #3 from the "classic bugs" section above.
6. `availablePermits()` and `queueLength()` are read-only helper methods. They are useful for logging or dashboards, so you can see how busy the limiter is. They do not change any behavior.

## How to Extend (Follow-ups)

- **Dynamic resizing.** Interviewers may ask: "What if we want to change the limit at runtime, for example from 5 to 10?" `Semaphore` supports this: call `release(n)` to add `n` extra permits, or repeatedly call `acquire()` (in a background thread) to shrink the pool. There is no direct "setPermits" method — you simulate resizing by adding or removing permits.
- **Per-key limiting.** What if you need a separate limit per resource type (for example, 5 for downloads, 3 for uploads)? Use a `ConcurrentHashMap<String, ResourceLimiter>`, keyed by resource type, and create a new limiter per key (see `computeIfAbsent`).
- **Async version.** If you use `CompletableFuture` or a reactive framework, wrap the semaphore logic so the "wait for a permit" step does not block a thread — for example, queue the task and run it when a permit frees up, using an `ExecutorService` plus your own small queue.
- **Rate limiting vs concurrency limiting.** Make sure you can explain the difference. A semaphore limits how many calls run at the same time (concurrency). A rate limiter (token bucket, sliding window) limits how many calls happen per time unit (for example, 100 calls per second), regardless of how long each call takes. These solve different problems and can be combined.
- **Distributed limiter.** For multiple servers sharing one limit, replace the in-memory `Semaphore` with a Redis-based counter (using `INCR`/`DECR` with expiry, or a Redis library like Redisson which offers a distributed `RSemaphore`).
- **Bulkhead pattern.** This whole design is an example of the "bulkhead" pattern from resilience engineering: isolate resources into separate pools, so a failure or slowdown in one pool does not use up all threads and crash the whole system. Libraries like Resilience4j have a built-in `Bulkhead` that does the same thing as this class, with more features (metrics, event listeners).

## Complexity & Thread-Safety Notes

- **Time complexity:** `acquire()`, `release()`, and `tryAcquire()` all run in O(1) on average (they use an internal lock-free counter based on `AbstractQueuedSynchronizer`). There is no per-call scan or loop.
- **Space complexity:** O(1) beyond the waiting threads. Each thread that blocks on `acquire()` adds one node to an internal wait queue, so space grows with the number of blocked threads, not with N.
- **Thread safety:** `Semaphore` itself is fully thread-safe. Multiple threads can call `acquire()`, `release()`, and `tryAcquire()` on the same instance at the same time, with no extra locking needed from you.
- **No shared mutable state in `ResourceLimiter`:** the class holds only one `final Semaphore` field. There is nothing else to protect, so the class needs no other synchronization.
- **Blocking behavior:** `acquire()` and `tryAcquire(timeout, unit)` can throw `InterruptedException`. If a thread is interrupted while waiting, it should stop waiting and propagate the interruption (either rethrow, or call `Thread.currentThread().interrupt()` if you cannot rethrow). Do not swallow `InterruptedException` silently.
- **Fairness cost:** a fair `Semaphore` (`fair = true`) has more overhead per `acquire()`/`release()` call than a non-fair one, because it must maintain strict FIFO order. Use it only when starvation is a real, observed problem.

## Interview Tips & Common Mistakes

- If asked "what is a semaphore," give the short definition first: a counter that limits how many threads can use a resource at once. Then add the counting vs binary distinction, since interviewers often follow up with that.
- Always mention the lock-vs-semaphore ownership difference. It is the single most common follow-up question, and many candidates miss it.
- Always write `acquire()` followed immediately by a `try { ... } finally { release(); }`. If you write the release call anywhere else, an interviewer will likely ask "what if the code throws an exception here?" — be ready with the `finally` answer.
- Do not confuse `tryAcquire()` (no arguments, returns immediately) with `tryAcquire(timeout, unit)` (waits up to the timeout). Know both exist and know when to use each.
- Be ready to explain why releasing more permits than acquired is dangerous: it is a silent bug. The `Semaphore` will not throw an exception. Your effective limit just becomes higher than intended, and this can cause the very overload you were trying to prevent.
- If asked about testing, mention: to test a limiter, you can use a `CountDownLatch` or `CyclicBarrier` to make many threads try to enter at the same time, then assert that no more than N threads run their task at once (for example, using an `AtomicInteger` counter that tracks "current active threads" and checks it never exceeds N).
- Mention `Semaphore` is part of `java.util.concurrent`, added in Java 5, and it is built on top of `AbstractQueuedSynchronizer` (AQS), the same base class used by `ReentrantLock` and `CountDownLatch`. Knowing this shows depth, but you do not need to explain AQS internals unless asked.
