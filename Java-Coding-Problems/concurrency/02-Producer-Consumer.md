# Producer–Consumer

## Problem

Design a system with two kinds of threads: producers and consumers.

- Producers create items and add them to a shared buffer.
- Consumers take items from the shared buffer and process them.
- The buffer has a fixed size. It is "bounded". It cannot hold unlimited items.
- If the buffer is full, producers must wait. They cannot add more items.
- If the buffer is empty, consumers must wait. They cannot take items.
- No item should be lost. No item should be processed twice.
- Threads should not "busy wait". Busy wait means checking a condition in a loop again and again, wasting CPU. Threads should sleep until they are woken up.

This problem is a classic concurrency problem. It tests if you understand locks, wait/notify, and thread-safe queues.

## Requirements & Clarifying Questions

Before coding, ask these questions in an interview. They show you think about edge cases.

1. How many producers and how many consumers? One of each, or many of each? (Assume many of each here.)
2. What is the buffer size? Fixed at creation time, or configurable?
3. What happens if a producer is faster than a consumer? (Producer must block, not drop items.)
4. What happens if a consumer is faster than a producer? (Consumer must block, not spin.)
5. How do we shut down cleanly? Do producers stop first, then consumers drain the rest and exit? Or do we need an interrupt-based stop?
6. Do we need FIFO order (first in, first out)? Usually yes, for fairness.
7. Should the solution use low-level `wait()`/`notify()`, or is a `java.util.concurrent` class allowed? (In interviews, expect both answers to be asked.)

## Design / Approach

There are two ways to build this. An interviewer often asks for both.

**Approach 1: Build it from scratch.**
Use a plain queue (like `LinkedList`) as the buffer. Use `synchronized` blocks to protect it. Use `wait()` and `notifyAll()` to make threads sleep and wake up. This shows you understand the low-level mechanics of the Java Memory Model and monitor locks.

**Approach 2: Use `BlockingQueue`.**
Java's `java.util.concurrent` package already has a thread-safe bounded queue: `ArrayBlockingQueue`. It has `put()` (blocks if full) and `take()` (blocks if empty) built in. This is the way you would write it in real production code. You do not reinvent locks in production.

Both approaches support multiple producers and multiple consumers. For shutdown, we use a **poison pill**. A poison pill is a special marker item. When a consumer reads it, the consumer knows there is no more work, and it stops.

## Java Solution

### Approach 1: From scratch with `synchronized` + `wait()`/`notifyAll()`

```java
import java.util.LinkedList;
import java.util.Queue;

public class BoundedBuffer<T> {

    private final Queue<T> queue = new LinkedList<>();
    private final int capacity;

    public BoundedBuffer(int capacity) {
        this.capacity = capacity;
    }

    public synchronized void put(T item) throws InterruptedException {
        // Use a while loop, not an if. Explained below.
        while (queue.size() == capacity) {
            wait(); // releases the lock and sleeps
        }
        queue.add(item);
        // Wake up any thread waiting on this same lock.
        // Could be a consumer waiting for an item, or (in some
        // designs) another producer. notifyAll() is safe for both.
        notifyAll();
    }

    public synchronized T take() throws InterruptedException {
        while (queue.isEmpty()) {
            wait();
        }
        T item = queue.poll();
        notifyAll();
        return item;
    }

    public synchronized int size() {
        return queue.size();
    }
}
```

A producer thread and a consumer thread using this buffer:

```java
public class Producer implements Runnable {
    private final BoundedBuffer<Integer> buffer;
    private final int id;

    public Producer(BoundedBuffer<Integer> buffer, int id) {
        this.buffer = buffer;
        this.id = id;
    }

    @Override
    public void run() {
        try {
            for (int i = 0; i < 5; i++) {
                int item = id * 100 + i;
                buffer.put(item);
                System.out.println("Producer " + id + " made " + item);
            }
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt(); // restore interrupt flag
        }
    }
}

public class Consumer implements Runnable {
    private final BoundedBuffer<Integer> buffer;
    private final int id;
    private static final int POISON_PILL = Integer.MIN_VALUE;

    public Consumer(BoundedBuffer<Integer> buffer, int id) {
        this.buffer = buffer;
        this.id = id;
    }

    @Override
    public void run() {
        try {
            while (true) {
                int item = buffer.take();
                if (item == POISON_PILL) {
                    buffer.put(POISON_PILL); // pass it on for the next consumer
                    break;
                }
                System.out.println("Consumer " + id + " took " + item);
            }
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }
    }
}
```

Wiring it together, with a poison pill per consumer for shutdown:

```java
public class ProducerConsumerDemo {
    public static void main(String[] args) throws InterruptedException {
        BoundedBuffer<Integer> buffer = new BoundedBuffer<>(10);
        int numProducers = 3;
        int numConsumers = 3;

        Thread[] producers = new Thread[numProducers];
        Thread[] consumers = new Thread[numConsumers];

        for (int i = 0; i < numProducers; i++) {
            producers[i] = new Thread(new Producer(buffer, i));
            producers[i].start();
        }
        for (int i = 0; i < numConsumers; i++) {
            consumers[i] = new Thread(new Consumer(buffer, i));
            consumers[i].start();
        }

        // Wait for all producers to finish making items.
        for (Thread p : producers) {
            p.join();
        }

        // Send one poison pill. The first consumer that reads it
        // puts it back, so the next consumer also sees it. This way
        // one pill can stop all consumers, one by one.
        buffer.put(Integer.MIN_VALUE);

        for (Thread c : consumers) {
            c.join();
        }
        System.out.println("All done.");
    }
}
```

### Approach 2: The production way, with `ArrayBlockingQueue`

```java
import java.util.concurrent.ArrayBlockingQueue;
import java.util.concurrent.BlockingQueue;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;
import java.util.concurrent.TimeUnit;

public class BlockingQueueDemo {

    private static final Integer POISON_PILL = Integer.MIN_VALUE;

    public static void main(String[] args) throws InterruptedException {
        BlockingQueue<Integer> queue = new ArrayBlockingQueue<>(10);
        int numProducers = 3;
        int numConsumers = 3;

        ExecutorService producerPool = Executors.newFixedThreadPool(numProducers);
        ExecutorService consumerPool = Executors.newFixedThreadPool(numConsumers);

        for (int p = 0; p < numProducers; p++) {
            int producerId = p;
            producerPool.submit(() -> {
                try {
                    for (int i = 0; i < 5; i++) {
                        int item = producerId * 100 + i;
                        queue.put(item); // blocks if full
                        System.out.println("Producer " + producerId + " made " + item);
                    }
                } catch (InterruptedException e) {
                    Thread.currentThread().interrupt();
                }
            });
        }

        for (int c = 0; c < numConsumers; c++) {
            int consumerId = c;
            consumerPool.submit(() -> {
                try {
                    while (true) {
                        int item = queue.take(); // blocks if empty
                        if (item == POISON_PILL) {
                            queue.put(POISON_PILL); // pass it on
                            break;
                        }
                        System.out.println("Consumer " + consumerId + " took " + item);
                    }
                } catch (InterruptedException e) {
                    Thread.currentThread().interrupt();
                }
            });
        }

        producerPool.shutdown();
        producerPool.awaitTermination(1, TimeUnit.MINUTES);

        queue.put(POISON_PILL); // stop the consumers, one pill chains to all

        consumerPool.shutdown();
        consumerPool.awaitTermination(1, TimeUnit.MINUTES);

        System.out.println("All done.");
    }
}
```

## How It Works

**Approach 1 details.**

The `synchronized` keyword on `put()` and `take()` means only one thread can run inside either method at a time, for a given `BoundedBuffer` object. This lock is called a "monitor lock". It protects the shared `queue` field from being changed by two threads at once. Without this lock, two threads could both check `queue.size()`, both see room, and both add an item. This could break the size limit. This bug is called a "race condition".

`wait()` does two things at once: it releases the lock, and it puts the thread to sleep. This is important. If `wait()` did not release the lock, no other thread could ever call `put()` or `take()`, and the sleeping thread would wait forever. This is called a "deadlock". When another thread calls `notifyAll()`, the sleeping thread wakes up and tries to get the lock again before it continues.

**Why `while`, not `if`, around `wait()`?**

This is the most common interview question on this topic. Suppose thread A is a consumer, waiting because the queue is empty. Thread B, a producer, adds one item and calls `notifyAll()`. This wakes up thread A. But thread A does not run right away. Maybe thread C, another consumer, was also woken up (or was already running), gets the lock first, and takes that one item. Now the queue is empty again. If thread A used `if (queue.isEmpty()) wait();`, it would not re-check the condition. It would try to `poll()` from an empty queue and either crash or return `null`. This bug is called a "spurious wakeup" problem, or more precisely here, a "lost wakeup race" between two waiters. The `while` loop makes the thread re-check the condition every time it wakes up. If the condition is still true (queue still empty), it calls `wait()` again. This is always the correct pattern. The Java documentation itself warns that `wait()` can also wake up for no reason at all (a true "spurious wakeup" from the JVM), so the `while` loop protects against that too.

**Why `notifyAll()`, not `notify()`?**

`notify()` wakes up only one waiting thread, chosen at random by the JVM. `notifyAll()` wakes up all waiting threads. In our buffer, threads wait for two different reasons: some wait because the buffer is full (producers), others wait because it is empty (consumers). If we call `notify()` after adding an item, the JVM might wake up a producer (which is also waiting on the same lock, for a different reason) instead of a consumer. That producer wakes up, re-checks its own `while` condition, finds the buffer still not full for it, and goes back to sleep. No consumer was woken up at all. The item sits unread. This is called a "missed signal" bug. `notifyAll()` avoids this by waking everyone up. Each thread then re-checks its own condition in its `while` loop. Threads whose condition is still false just go back to sleep. This costs a bit more CPU, because more threads wake up and re-check, but it is correct. In simple cases with only one type of wait condition, `notify()` can be enough, but `notifyAll()` is the safe default, especially with mixed producers and consumers.

**Approach 2 details.**

`ArrayBlockingQueue` does all of the above internally, using a `ReentrantLock` and two `Condition` objects (one for "not full", one for "not empty") instead of raw `wait()`/`notify()`. `put()` blocks the calling thread if the queue is full. `take()` blocks the calling thread if the queue is empty. This is exactly the same behavior as our hand-built `BoundedBuffer`, but tested, tuned, and safe. This is why real Java code uses `BlockingQueue` implementations instead of writing `wait()`/`notify()` by hand.

**Poison pill.**

A poison pill is a sentinel value. It is a value that does not represent real data. It only signals "stop". We pick a value consumers will never see as real data (here, `Integer.MIN_VALUE`). When a consumer reads it, it puts the pill back into the queue before exiting. This lets the pill "chain" through all consumers, one at a time, until every consumer has seen it and stopped. An alternative is to add one pill per consumer up front, so each consumer eats exactly one pill.

## How to Extend (Follow-ups)

Interviewers often push further. Common follow-ups:

- **Multiple item types or priorities.** Swap the queue for a `PriorityBlockingQueue`. Note: this queue is unbounded by default, so you lose the "bounded" backpressure unless you add your own limit check.
- **Timeouts.** Use `offer(item, timeout, unit)` and `poll(timeout, unit)` instead of `put()`/`take()`. This lets a thread give up after waiting too long, instead of waiting forever.
- **Graceful shutdown without poison pills.** Use an `ExecutorService` with `shutdown()` (finish current tasks, reject new ones) or `shutdownNow()` (interrupt running threads). Combine with `awaitTermination()` to wait for a clean stop. This is often cleaner than poison pills when producers and consumers are managed by thread pools.
- **Batch processing.** Consumers pull many items at once using `drainTo(collection, maxItems)`, instead of one item at a time. This reduces lock contention.
- **Backpressure signaling.** Instead of blocking forever, use `offer()` (non-blocking, returns `false` if full) so a producer can decide to drop, retry, or log an error.
- **Metrics.** Track queue size over time, average wait time, and throughput, to detect if producers or consumers are the bottleneck.
- **Multiple queues (work stealing).** Give each consumer its own queue, and let idle consumers "steal" work from busy consumers' queues. This is how `ForkJoinPool` improves throughput.

## Complexity & Thread-Safety Notes

- **Time complexity:** `put()` and `take()` are O(1) in both approaches (`LinkedList.add`/`poll`, and `ArrayBlockingQueue`'s ring buffer).
- **Space complexity:** O(capacity), since the buffer never holds more than its fixed size.
- **Thread safety in Approach 1:** correctness depends on every access to `queue` going through a `synchronized` method. Never read or write `queue` directly outside `put()`/`take()`/`size()`. If you add a new method, make it `synchronized` too, or use the same lock object.
- **Thread safety in Approach 2:** `ArrayBlockingQueue` is fully thread-safe on its own. You do not add any extra `synchronized` keyword around it.
- **Liveness:** both approaches avoid deadlock because `wait()` releases the lock before sleeping. Both avoid busy waiting because threads sleep instead of looping and checking.
- **Fairness:** `ArrayBlockingQueue` has a constructor argument for fairness (`new ArrayBlockingQueue<>(capacity, true)`). Fair mode uses a FIFO order for threads waiting on the lock, so no thread waits forever while others repeatedly jump ahead. It is slower, so use it only if starvation is a real risk.
- **Memory visibility:** the Java Memory Model guarantees that changes made by one thread inside a `synchronized` block (or inside `ArrayBlockingQueue`'s internal lock) are visible to the next thread that enters a `synchronized` block on the same lock. Without this guarantee, one thread might never see updates made by another thread, due to CPU caching.

## Interview Tips & Common Mistakes

- **Do not forget `while`, not `if`, around `wait()`.** This is the single most checked detail in this problem. Explain the spurious wakeup and lost wakeup reasons out loud.
- **Do not use `notify()` by default.** Say clearly why `notifyAll()` is the safer choice when there is more than one reason to wait.
- **Do not check `queue.size()` outside the lock, then act on it.** This is called "check-then-act" and it is not atomic. Another thread can change the queue between your check and your action.
- **Do not swallow `InterruptedException` silently.** Always either handle it or call `Thread.currentThread().interrupt()` to restore the interrupt flag, so callers higher up still know the thread was interrupted.
- **Say that `ArrayBlockingQueue` is the production answer.** Interviewers want to see that you know the hand-built version is for learning, not for real code. Mention `LinkedBlockingQueue` (optionally bounded, uses two locks internally for higher throughput) as an alternative.
- **Explain the poison pill trade-off.** It only works cleanly if you know exactly how many consumers to stop. For dynamic pools, `ExecutorService.shutdown()` with `awaitTermination()` is more robust.
- **Mention `volatile` is not enough here.** Some candidates think adding `volatile` to the queue reference fixes thread safety. It does not. `volatile` only makes a single variable's reads and writes visible across threads. It does not make compound actions (like "check size, then add") atomic. You still need locking for that.
- **Know the difference between `wait()`/`notify()` and `Object.wait()` requiring the lock.** You must call `wait()`, `notify()`, and `notifyAll()` only from inside a `synchronized` block on the same object. Calling them without holding the lock throws `IllegalMonitorStateException`. This is a quick way interviewers test if you truly understand the mechanism, not just memorized the code.
