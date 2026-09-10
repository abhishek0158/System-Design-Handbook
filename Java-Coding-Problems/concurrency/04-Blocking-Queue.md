# Custom Blocking Queue

## Problem

Build your own bounded blocking queue in Java. Do not use `java.util.concurrent.BlockingQueue` or any class from that package. Build the logic yourself.

A bounded blocking queue has a fixed capacity. It supports two main operations:

- `put(item)` — adds an item. If the queue is full, the calling thread must wait until space is free.
- `take()` — removes and returns an item. If the queue is empty, the calling thread must wait until an item is added.

This is a classic producer-consumer problem. It is a common interview question because it tests your knowledge of Java concurrency primitives: `wait()`/`notify()`, `Lock`, and `Condition`.

## Requirements & Clarifying Questions

Before coding, an interviewer expects you to ask a few questions. Here are the important ones, with answers we will assume:

1. **Is the capacity fixed or can it grow?**
   Fixed. This is set once, in the constructor. This is what makes it "bounded."

2. **What type of data does it hold?**
   Any type. The queue must be generic, using `<T>`.

3. **What backing storage should we use — an array or a `LinkedList`?**
   Both work. We will use a **circular array** in the final solution. It avoids extra node allocation and has good cache behavior. We will mention the `LinkedList` option too.

4. **What happens if many threads call `put()` or `take()` at the same time?**
   All of them must be handled correctly. No item should be lost or read twice. This is the core thread-safety requirement.

5. **Should there be a timeout version, like `offer(item, timeout)`?**
   Not required for the base solution, but we will discuss it as a follow-up.

6. **Is `null` allowed as an item?**
   No. We will reject `null` with a `NullPointerException`, matching the convention used by `java.util.concurrent` classes.

## Design / Approach

We will build two versions. Both solve the same problem. The second version is the one you should lead with in an interview, because it is more efficient.

**Version 1: `synchronized` + `wait()` / `notifyAll()`**

This is the "classic" Java way, available since Java 1.0. The idea:

- The queue object itself is the lock. Every method that touches the shared state (`items`, `count`, `head`, `tail`) is `synchronized`.
- If `put()` finds the queue full, it calls `wait()`. This releases the lock and pauses the thread.
- If `take()` removes an item, it calls `notifyAll()`. This wakes up all waiting threads, so any thread waiting in `put()` gets a chance to check again.
- The same pattern applies in reverse: `take()` waits when empty, `put()` notifies when it adds an item.

The problem with this design: `notifyAll()` wakes up **every** waiting thread, not just the ones that can now proceed. Imagine 10 threads are blocked in `put()` because the queue is full, and 5 threads are blocked in `take()` because... wait, that cannot happen at the same time in a bounded queue (full and empty are different states). But a more realistic case: 10 threads are blocked in `put()`. One `take()` call frees one slot and calls `notifyAll()`. All 10 waiting producer threads wake up. Only one can succeed (there is only one free slot). The other 9 threads wake up, check the condition, see the queue is full again, and go back to sleep. This wastes CPU time. This effect is called a **thundering herd**.

**Version 2: `ReentrantLock` with two `Condition` objects**

This is the better design, and it is how `java.util.concurrent.ArrayBlockingQueue` actually works internally. The idea:

- Use one `ReentrantLock` to protect the shared state, just like the `synchronized` keyword does.
- Create **two separate `Condition` objects** from that lock: `notFull` and `notEmpty`.
- `put()` waits on `notFull` when the queue is full. `take()` signals `notFull` after removing an item (because removing an item always frees a slot).
- `take()` waits on `notEmpty` when the queue is empty. `put()` signals `notEmpty` after adding an item (because adding an item always makes the queue non-empty).

**Why two conditions are more efficient than `notifyAll()`:**

With one lock, `wait()`/`notifyAll()` puts every waiting thread — producers and consumers — into the same "waiting room." When you call `notifyAll()`, you wake up everyone in that room, even threads that are waiting for a completely different condition. There is no way to wake up only the producers or only the consumers with plain `wait()`/`notify()`, unless you accept the cost of `notifyAll()` waking everyone and having most of them go back to sleep.

`Condition` objects solve this by giving you separate waiting rooms tied to the same lock. `notFull.signal()` wakes up only a thread waiting on `notFull` — that is, only a producer. `notEmpty.signal()` wakes up only a thread waiting on `notEmpty` — only a consumer. This means:

- Fewer threads wake up unnecessarily. Less wasted CPU on context switches and re-checking conditions.
- You can use `signal()` (wakes one thread) instead of `signalAll()` in many cases, because you know exactly which "room" needs waking. This scales much better when many threads are waiting.

This targeted wake-up is the single biggest reason interviewers want to see the `Lock` + two-`Condition` version. It shows you understand that `Condition` is not just a "fancier `wait()`" — it exists to solve the thundering-herd problem.

**The `while` loop guard**

In both versions, we check the wait condition inside a `while` loop, not an `if` statement:

```java
while (count == capacity) {
    notFull.await();
}
```

This is critical. There are two reasons for it:

1. **Spurious wakeups.** The JVM is allowed, by specification, to wake up a waiting thread even when nobody called `signal()`. This is rare but real. If we used `if`, the thread would proceed without re-checking the actual condition, and could corrupt shared state.
2. **Stolen conditions.** Even with a real `signal()` call, by the time the woken thread gets the lock back, another thread might have already taken the free slot. For example: two producer threads are both waiting on `notFull`. A consumer frees one slot and calls `notFull.signal()`. One producer wakes up. But before it acquires the lock, a third, different producer thread — one that was never waiting, just arriving fresh — calls `put()`, acquires the lock first, and fills the slot. Now the queue is full again. When the originally-woken thread finally gets the lock, it must check the condition again. If it does not, it will insert into a full queue.

The rule is simple: always re-check the condition after waking up. A `while` loop does this automatically, because after `await()` returns, the loop condition is evaluated again before proceeding.

## Java Solution

### Version 1 — `synchronized` + `wait()`/`notifyAll()`

```java
import java.util.LinkedList;
import java.util.Queue;

public class BlockingQueueV1<T> {

    private final Queue<T> queue = new LinkedList<>();
    private final int capacity;

    public BlockingQueueV1(int capacity) {
        if (capacity <= 0) {
            throw new IllegalArgumentException("capacity must be > 0");
        }
        this.capacity = capacity;
    }

    public synchronized void put(T item) throws InterruptedException {
        if (item == null) {
            throw new NullPointerException("item cannot be null");
        }
        while (queue.size() == capacity) {
            wait(); // releases the lock, waits, re-acquires lock on wakeup
        }
        queue.add(item);
        notifyAll(); // wake up any thread waiting in take()
    }

    public synchronized T take() throws InterruptedException {
        while (queue.isEmpty()) {
            wait();
        }
        T item = queue.poll();
        notifyAll(); // wake up any thread waiting in put()
        return item;
    }

    public synchronized int size() {
        return queue.size();
    }
}
```

### Version 2 — `ReentrantLock` with two `Condition` objects (preferred)

```java
import java.util.concurrent.locks.Condition;
import java.util.concurrent.locks.ReentrantLock;

public class BlockingQueueV2<T> {

    private final Object[] items;
    private int head = 0;   // index to take() from
    private int tail = 0;   // index to put() into
    private int count = 0;  // number of items currently stored

    private final ReentrantLock lock = new ReentrantLock();
    private final Condition notFull = lock.newCondition();
    private final Condition notEmpty = lock.newCondition();

    public BlockingQueueV2(int capacity) {
        if (capacity <= 0) {
            throw new IllegalArgumentException("capacity must be > 0");
        }
        this.items = new Object[capacity];
    }

    public void put(T item) throws InterruptedException {
        if (item == null) {
            throw new NullPointerException("item cannot be null");
        }
        lock.lock();
        try {
            while (count == items.length) {
                notFull.await(); // sleep until a slot frees up
            }
            items[tail] = item;
            tail = (tail + 1) % items.length; // wrap around, circular array
            count++;
            notEmpty.signal(); // exactly one consumer can now proceed
        } finally {
            lock.unlock();
        }
    }

    @SuppressWarnings("unchecked")
    public T take() throws InterruptedException {
        lock.lock();
        try {
            while (count == 0) {
                notEmpty.await(); // sleep until an item is available
            }
            T item = (T) items[head];
            items[head] = null; // help garbage collection
            head = (head + 1) % items.length;
            count--;
            notFull.signal(); // exactly one producer can now proceed
            return item;
        } finally {
            lock.unlock();
        }
    }

    public int size() {
        lock.lock();
        try {
            return count;
        } finally {
            lock.unlock();
        }
    }
}
```

## How It Works

The circular array uses two index pointers: `head` (next slot to read from) and `tail` (next slot to write to). Both wrap around to `0` when they reach the end of the array, using the modulo operator (`% items.length`). This avoids shifting elements, unlike a plain array-based list. The `count` field tracks how many slots are filled, so we do not need to compare `head` and `tail` to tell full from empty — that comparison alone is ambiguous when the array wraps around.

`put()` and `take()` both start by acquiring the `ReentrantLock`. This lock plays the same role as the implicit monitor lock used by `synchronized`, but it lets us create multiple `Condition` objects tied to it.

When `put()` sees the queue is full, it calls `notFull.await()`. This atomically releases the lock and puts the thread to sleep in the "notFull waiting room." When some other thread calls `notFull.signal()`, one thread from that room wakes up, re-acquires the lock, and resumes right after the `await()` call — at the top of the `while` loop, where it re-checks the condition.

The same logic applies to `take()` and `notEmpty`, in the opposite direction.

Every `put()` call ends by signaling `notEmpty`, because adding an item always means the queue is no longer empty — a consumer might be waiting for exactly that. Every `take()` call ends by signaling `notFull`, because removing an item always frees a slot — a producer might be waiting for exactly that.

The `try`/`finally` block around the lock is required. If an exception is thrown after `lock.lock()`, the `finally` block guarantees `lock.unlock()` still runs. Without this, the lock would stay held forever, and every other thread would block forever too. This is a common bug in interviews — do not forget it.

## How to Extend (Follow-ups)

Interviewers often ask you to extend the basic queue. Common follow-ups:

- **`offer(item, timeout, unit)` — a put with a timeout.** Instead of `notFull.await()`, use `notFull.awaitNanos(remainingNanos)`, which returns early if the wait times out. Track remaining time across loop iterations, because spurious wakeups can happen more than once before the real timeout expires. Return `false` if the timeout expires without success, instead of throwing an exception.

- **`poll(timeout, unit)` — a take with a timeout.** Same idea, applied to `notEmpty.awaitNanos(...)`.

- **`size()` is already shown above.** You can also add `isEmpty()` and `isFull()`, both trivial checks under the lock.

- **Fairness.** `new ReentrantLock(true)` creates a fair lock. Fair locks grant access in roughly FIFO order, which avoids starving threads that have been waiting a long time. The cost is lower throughput, because fairness adds bookkeeping overhead.

- **Draining multiple items at once**, similar to `drainTo()` in `java.util.concurrent`. Useful for batch processing consumers.

- **Backing with a `LinkedList` instead of an array.** This trades a small allocation cost per node for simpler code, since you do not need to manage `head`/`tail`/wraparound. For a fixed, small-to-medium capacity, the array version is usually faster and uses less memory, because it avoids per-node object overhead.

## Complexity & Thread-Safety Notes

- **Time complexity:** `put()` and `take()` are both O(1) — no shifting, no scanning, just index math and array access.
- **Space complexity:** O(capacity) — the array is allocated once, up front, at the fixed size.
- **Thread safety:** All mutable state (`items`, `head`, `tail`, `count`) is only ever touched while holding `lock`. This guarantees mutual exclusion — only one thread can execute inside `put()` or `take()`'s critical section at a time — and visibility — a write made by one thread while holding the lock is guaranteed to be seen by the next thread that acquires the same lock.
- **No lost wakeups:** because `signal()` is called only after the state actually changes, and only while holding the lock, a signal is never "missed" between checking the condition and going to sleep.
- **No busy-waiting:** threads sleep completely while waiting, instead of spinning in a loop and burning CPU.
- **Interruption:** `await()` throws `InterruptedException` if the thread is interrupted while waiting. Both `put()` and `take()` propagate this checked exception upward, so a caller can cancel a blocked thread cleanly using `Thread.interrupt()`.

## Interview Tips & Common Mistakes

- **Do not use `if` instead of `while` around `await()` or `wait()`.** This is the single most common mistake, and interviewers watch for it closely. Always re-check the condition after waking up.

- **Do not forget `try`/`finally` around `lock.unlock()`.** A lock that never releases freezes the entire queue.

- **Explain why two `Condition` objects beat one `synchronized` block with `notifyAll()`.** This is usually the main point of the question. Mention the thundering-herd problem by name if you can.

- **Know when `signal()` is safe versus when you need `signalAll()`.** In this queue, only one thread can make progress per state change (one `put()` frees at most one wait on `notEmpty`), so `signal()` is correct and efficient. If multiple threads could progress from a single state change, you would need `signalAll()` instead, to avoid missed wakeups.

- **Handle `InterruptedException` correctly.** Do not swallow it silently. Either propagate it (as we do here) or restore the interrupt flag with `Thread.currentThread().interrupt()` if you must catch it.

- **Reject `null` items explicitly.** Silently accepting `null` causes confusing bugs later, because `null` often means "empty slot" internally.

- **Mention the built-in alternative.** In real production code, you would use `java.util.concurrent.ArrayBlockingQueue` or `LinkedBlockingQueue`, both already well-tested. This exercise exists to prove you understand what is happening inside those classes, not to replace them.
