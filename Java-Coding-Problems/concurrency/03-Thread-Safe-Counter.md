# Thread-safe Counter

## Problem

Build a counter class. Many threads can call `increment()` at the same time. Many threads can also call `get()` to read the current value.

The final count must be correct. If 100 threads each call `increment()` 1,000 times, the final value must be exactly 100,000. No count can be lost.

This sounds simple. But it is one of the most common concurrency interview questions. It tests if you understand *why* a simple `count++` fails, and if you know the right tool to fix it.

## Requirements & Clarifying Questions

Before coding, ask these questions. They show the interviewer that you think about trade-offs, not just syntax.

1. **How many threads will call `increment()`?** Few threads (say, 4–8) behave differently from many threads (say, 64+). This affects which fix is best.
2. **Is contention low or high?** Contention means how often threads try to update the counter at the same moment. High contention needs a different design than low contention.
3. **Do we need `get()` to return a value that is instantly correct (strict consistency), or is a slightly stale value okay?**
4. **Do we need other operations, like `decrement()`, `reset()`, or `compareAndSet()`?**
5. **Is this counter used inside a larger object that already holds a lock?** If yes, we may need to reuse that lock instead of adding a new one.

For this problem, assume: many threads call `increment()` and `get()`. We want the count to always be correct. We will compare four solutions and say when to use each.

## Design / Approach

### Why `count++` is not atomic

Here is the broken version:

```java
public class BrokenCounter {
    private int count = 0;

    public void increment() {
        count++;
    }

    public int get() {
        return count;
    }
}
```

`count++` looks like one step. It is not. The Java Virtual Machine (JVM) breaks it into three separate steps:

1. **Read** the current value of `count` from memory into a register.
2. **Modify** the value: add 1 to it.
3. **Write** the new value back to memory.

This is called a **read-modify-write** sequence. Each step is atomic on its own. But the three steps together are not atomic. "Atomic" means the operation happens as one indivisible unit — no other thread can see it half-done, and no other thread can interleave in the middle of it.

Here is the race condition. Assume `count` starts at 5.

| Time | Thread A | Thread B | Actual value in memory |
|------|----------|----------|------------------------|
| t1 | reads count = 5 | | 5 |
| t2 | | reads count = 5 | 5 |
| t3 | adds 1, gets 6 | | 5 |
| t4 | | adds 1, gets 6 | 5 |
| t5 | writes 6 | | 6 |
| t6 | | writes 6 | 6 |

Both threads called `increment()`. We expected the final value to be 7. But it is 6. One increment was **lost**. This is called a **lost update**.

This race happens because both threads read the same starting value before either thread writes back its result. The more threads you have, and the more often they call `increment()`, the more updates get lost.

### Why `volatile` does NOT fix this

A common mistake is to write:

```java
private volatile int count = 0;
```

`volatile` guarantees **visibility**. This means: when one thread writes to `count`, other threads will see that new value right away. Without `volatile`, a thread might cache the value in a CPU register or local cache and never see updates from other threads.

But `volatile` does **not** guarantee **atomicity**. It does not stop two threads from both reading the same value and both writing back the same result. The read-modify-write sequence is still three separate steps. `volatile` only makes sure each individual read and each individual write is visible immediately. It does nothing to combine the three steps into one atomic unit.

So `volatile int count` still has the lost-update race. This is a very common interview trap. Remember: **`volatile` fixes visibility, not atomicity.**

Now let's look at four correct solutions.

## Java Solution

### 1. `synchronized` method or block

```java
public class SynchronizedCounter {
    private int count = 0;

    public synchronized void increment() {
        count++;
    }

    public synchronized int get() {
        return count;
    }
}
```

`synchronized` uses a **monitor lock** (also called an intrinsic lock). Only one thread can hold this lock at a time. When a thread calls `increment()`, it must first acquire the lock on `this` object. Any other thread that calls `increment()` or `get()` at the same time must wait until the lock is released.

This makes the read-modify-write sequence behave as one atomic block, because no other thread can enter a synchronized method on the same object while one thread is inside it.

**When to use it:** Low to medium contention. Simple code, easy to read and reason about. Good default choice when you are not sure yet, or when you need to protect more than one field together (for example, `count` and a `lastUpdatedTime` field that must change together).

### 2. `AtomicInteger` / `AtomicLong`

```java
import java.util.concurrent.atomic.AtomicInteger;

public class AtomicCounter {
    private final AtomicInteger count = new AtomicInteger(0);

    public void increment() {
        count.incrementAndGet();
    }

    public int get() {
        return count.get();
    }
}
```

`AtomicInteger` does not use a lock. It uses a hardware-level instruction called **Compare-And-Swap (CAS)**. The idea of CAS:

1. Read the current value, call it `oldValue`.
2. Compute the new value: `newValue = oldValue + 1`.
3. Ask the CPU: "If the value in memory is still `oldValue`, replace it with `newValue`. Do this as one atomic hardware step. Tell me if it worked."
4. If it worked, we are done.
5. If it did not work (because another thread changed the value between step 1 and step 3), retry from step 1.

This retry loop is called a **CAS loop** or a **spin loop**. `incrementAndGet()` internally does this loop until it succeeds.

CAS is faster than `synchronized` under low to medium contention because it never blocks a thread. A blocked thread must be put to sleep and woken up later by the operating system — this is expensive. CAS threads that fail just retry immediately, which is cheap.

**When to use it:** Low to medium contention, single-variable updates. This is usually the best default choice for a simple counter. It is simpler and faster than `synchronized` for this specific case.

### 3. `LongAdder` for high contention

```java
import java.util.concurrent.atomic.LongAdder;

public class HighContentionCounter {
    private final LongAdder count = new LongAdder();

    public void increment() {
        count.increment();
    }

    public long get() {
        return count.sum();
    }
}
```

Under **high contention** (many threads, all hitting the counter very often), `AtomicInteger` starts to slow down. Here is why: all threads do CAS on the **same single memory location**. When contention is high, many CAS attempts fail and retry. Threads keep "fighting" over one cache line. This wastes CPU cycles.

`LongAdder` solves this with a different data structure. Internally, it keeps an array of separate counter **cells**. Each thread (roughly) gets its own cell to update, based on a hash of the thread. This spreads the writes across multiple memory locations instead of one. This means far less CAS contention, because different threads are usually not fighting over the same cell.

When you call `sum()`, `LongAdder` walks through all the cells and adds them up. This is why `get()` on `LongAdder` (called `sum()`) is a bit more expensive than a plain `AtomicInteger.get()`. But `increment()` is much cheaper under high contention.

**Trade-off:** `LongAdder` is optimized for high write frequency, low read frequency. If you read the count constantly (for example, every increment is followed by a read), `AtomicInteger` may still be better. If you only read the count occasionally (for example, once at the end, or once per second for a metric), `LongAdder` wins under heavy write load.

**When to use it:** High contention counters, like request counters or metrics in a busy server, where many threads increment constantly but you only read the total occasionally.

### 4. `ReentrantLock` version

```java
import java.util.concurrent.locks.ReentrantLock;

public class LockCounter {
    private int count = 0;
    private final ReentrantLock lock = new ReentrantLock();

    public void increment() {
        lock.lock();
        try {
            count++;
        } finally {
            lock.unlock();
        }
    }

    public int get() {
        lock.lock();
        try {
            return count;
        } finally {
            lock.unlock();
        }
    }
}
```

`ReentrantLock` is similar to `synchronized`. It also blocks other threads until the lock is free. But it gives you more control:

- `tryLock()` — try to get the lock, but do not wait forever. Useful to avoid deadlock.
- `tryLock(timeout, unit)` — wait only for a limited time.
- `lockInterruptibly()` — allow a waiting thread to be interrupted.
- Fairness option — `new ReentrantLock(true)` gives the lock to the longest-waiting thread first, instead of any random thread.

The `try...finally` block is mandatory. If `count++` throws an exception (unlikely here, but a rule to always follow), the `finally` block still releases the lock. If you forget `unlock()` in a `finally` block, other threads can be stuck waiting forever — a permanent deadlock.

**When to use it:** When you need `tryLock`, fairness, or interruptible waiting. For a plain counter, `ReentrantLock` gives no real benefit over `synchronized` — it is more verbose and easier to misuse (forgetting `unlock()`). Use it here mainly to show you understand the API and its trade-offs.

## How It Works

All four solutions fix the same root problem: they make the read-modify-write sequence behave as one atomic unit, so no thread can see or cause a half-finished update.

- `synchronized` and `ReentrantLock` do this with **mutual exclusion**: only one thread is allowed inside the critical section at a time. Other threads block (sleep) and wait.
- `AtomicInteger` does this with **CAS**: no thread ever sleeps. Threads that lose the race just retry immediately.
- `LongAdder` does this with **striping**: it uses many CAS cells instead of one, so most threads do not even compete with each other.

## How to Extend (Follow-ups)

Interviewers often ask follow-up questions. Be ready for these:

1. **Add a `reset()` method.** With `AtomicInteger`, use `count.set(0)`. With `LongAdder`, use `count.reset()`. Note: `reset()` on `LongAdder` is not atomic with other threads calling `increment()` at the same moment — mention this trade-off.
2. **Add `decrementIfPositive()`** — decrement only if count is greater than 0. This needs a CAS loop written by hand with `AtomicInteger.compareAndSet()`, because you must check-then-act atomically.
3. **What if you need to update two related fields together** (for example, count and a timestamp)? `AtomicInteger` cannot do this — you would need `synchronized` or `ReentrantLock`, since they protect the whole block, not just one variable.
4. **What about a distributed counter** (across multiple servers, not just threads)? This needs a different design: a shared store like Redis with `INCR`, or a database counter table, or an approach like sharded counters.
5. **What if you need to see live increments without any lock at all, for pure read speed?** Discuss `LongAdder.sum()` being an approximate snapshot when reads happen during concurrent writes — it may not be perfectly exact at that exact instant, since it reads each cell one by one.

## Complexity & Thread-Safety Notes

- **Time complexity:** `increment()` and `get()` are O(1) for all four approaches. The difference is in **constant factors** under contention, not in big-O terms.
- **`synchronized` / `ReentrantLock`:** Correct and simple. Under high contention, threads block and get put to sleep, then woken up — this context-switching cost is the main slowdown.
- **`AtomicInteger`:** Faster than locks under low/medium contention, because failed threads retry instead of sleeping. Under very high contention, CAS retries increase and performance can drop, though it is usually still better than a lock.
- **`LongAdder`:** Best for high-contention writes. Uses more memory than `AtomicInteger` (an internal array of cells that grows as contention increases). `sum()` is not perfectly atomic as a whole — it is a best-effort total across cells at read time.
- **`volatile int count`:** Never thread-safe for increment, no matter how it looks. Only safe for simple flag variables where one thread writes and others only read, with no read-modify-write step.
- **General rule:** As contention increases, prefer: `synchronized`/`ReentrantLock` → `AtomicInteger` → `LongAdder`, in that order of scaling.

## Interview Tips & Common Mistakes

- **Do not say "volatile makes it thread-safe."** This is the single most common mistake in this problem. Always explain the visibility-versus-atomicity difference clearly.
- **Always explain the three-step read-modify-write breakdown** of `count++`. Interviewers want to see you understand *why* it fails, not just that it fails.
- **Know what CAS means and how it works**, including the retry loop. Many candidates use `AtomicInteger` correctly but cannot explain what happens inside `incrementAndGet()`.
- **Mention `LongAdder` even if not asked.** It shows you know about high-contention scenarios beyond the basic `AtomicInteger`, which is a strong signal for 3–4 years of experience.
- **Never forget `try...finally` with `ReentrantLock`.** Forgetting `unlock()` is a real production bug pattern, and interviewers watch for it.
- **Do not over-engineer.** If the interviewer says "just a few threads, simple case," `synchronized` or `AtomicInteger` is enough. Do not jump straight to `LongAdder` without justifying the high-contention need.
- **Be ready to write a hand-rolled CAS loop**, for example:

```java
public void incrementIfLessThan(int max) {
    int current;
    int next;
    do {
        current = count.get();
        if (current >= max) {
            return;
        }
        next = current + 1;
    } while (!count.compareAndSet(current, next));
}
```

This shows you understand the "read, compute, compare-and-swap, retry if failed" pattern behind every atomic class in `java.util.concurrent`.
