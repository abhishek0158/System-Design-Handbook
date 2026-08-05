# 4. Multithreading & Concurrency

Multithreading lets a program do many things at the same time. This is one of the most tested topics in Java interviews at 3-4 years experience. Interviewers expect you to know internals (JMM, locks, CAS), not just API names.

## Key Concepts (Quick Revision)

- **Process**: an independent running program. It has its own memory space. Processes do not share memory by default.
- **Thread**: a lightweight unit of execution inside a process. Threads in the same process share heap memory (objects, static fields) but each thread has its own stack and program counter.
- **Ways to create a thread**: extend `Thread`, implement `Runnable`, implement `Callable` (returns a value, can throw checked exceptions), or submit tasks to an `ExecutorService`.
- **Thread lifecycle**: NEW → RUNNABLE → (BLOCKED / WAITING / TIMED_WAITING) → TERMINATED.
- **`synchronized`**: a keyword that gives mutual exclusion using a monitor lock. Only one thread can hold the lock on an object at a time.
- **`volatile`**: a keyword that guarantees visibility of a variable's value across threads. It does NOT guarantee atomicity.
- **Java Memory Model (JMM)**: the rulebook that defines when a write by one thread becomes visible to another thread.
- **happens-before**: a JMM rule. If action A happens-before action B, then A's effects are visible to B.
- **Race condition**: a bug where the result depends on the timing/order of threads.
- **Atomicity**: an operation that completes fully or not at all, with no partial state visible to other threads.
- **Deadlock**: two or more threads wait for each other's locks forever.
- **Livelock**: threads keep changing state in response to each other but make no progress.
- **Starvation**: a thread never gets CPU time or a lock because other threads keep getting priority.
- **`ReentrantLock`**: a lock class with more features than `synchronized` (tryLock, fairness, interruptible).
- **CAS (compare-and-swap)**: a CPU instruction used to update a value without locks.
- **`ExecutorService`**: a framework to manage a pool of worker threads.
- **`CompletableFuture`**: a class to write async code that can be chained and combined.
- **`ThreadLocal`**: gives each thread its own private copy of a variable.

## Important Interview Questions

### 1. Process vs Thread

A process is an independent program with its own memory. A thread is a smaller unit inside a process. Threads share the heap (objects, static data) with other threads of the same process, but each thread has its own stack, program counter, and register set.

| Aspect | Process | Thread |
|---|---|---|
| Memory | Separate | Shared (heap), private (stack) |
| Creation cost | High | Low |
| Communication | IPC (pipes, sockets) | Shared variables |
| Crash impact | Isolated | Can affect whole JVM |

**Follow-up**: Why are threads cheaper than processes? Because creating a thread does not need a new memory space or OS-level resource duplication. Only a new stack and metadata are created.

### 2. Ways to create a thread

**a) Extend `Thread`**
```java
class MyThread extends Thread {
    public void run() { System.out.println("running"); }
}
new MyThread().start();
```

**b) Implement `Runnable`** (preferred, since Java does not allow multiple inheritance, and it separates the task from the thread mechanism)
```java
Runnable task = () -> System.out.println("running");
new Thread(task).start();
```

**c) Implement `Callable` with `Future`** (used when you need a result or need to throw a checked exception)
```java
ExecutorService executor = Executors.newSingleThreadExecutor();
Callable<Integer> task = () -> 10 + 20;
Future<Integer> future = executor.submit(task);
Integer result = future.get(); // blocks until result is ready
executor.shutdown();
```

**Follow-up**: `Runnable.run()` returns nothing and cannot throw checked exceptions. `Callable.call()` returns a value and can throw checked exceptions. This is why `Callable` is used with `ExecutorService.submit()`.

**Follow-up**: Why is `start()` used and not `run()` directly? Calling `run()` directly just runs the code on the current thread, like a normal method call. Calling `start()` creates a new OS thread and then that thread calls `run()`.

### 3. Thread lifecycle / states

Java thread states (from `Thread.State` enum):

- **NEW**: thread object created, `start()` not called yet.
- **RUNNABLE**: thread is running or ready to run (waiting for CPU).
- **BLOCKED**: thread is waiting to acquire a monitor lock (waiting to enter a `synchronized` block).
- **WAITING**: thread is waiting indefinitely for another thread (e.g., called `wait()`, `join()` with no timeout, or `LockSupport.park()`).
- **TIMED_WAITING**: like WAITING, but with a timeout (e.g., `sleep(ms)`, `wait(ms)`, `join(ms)`).
- **TERMINATED**: thread has finished execution.

**Follow-up**: Difference between BLOCKED and WAITING? BLOCKED means the thread wants a lock that another thread already holds. WAITING means the thread gave up the lock itself and is waiting to be notified or woken up.

### 4. `synchronized` — method vs block, object vs class lock

`synchronized` gives mutual exclusion. Only one thread can execute a synchronized block/method on the same lock object at any time.

- **Synchronized instance method**: lock is the current object (`this`).
```java
public synchronized void increment() { count++; }
```
- **Synchronized static method**: lock is the `Class` object (e.g., `MyClass.class`). This means all instances share the same lock.
```java
public static synchronized void log() { }
```
- **Synchronized block**: you choose the lock object. This is preferred because it locks only the critical section, not the whole method, which improves performance.
```java
public void increment() {
    synchronized (this) {
        count++;
    }
}
```

**How the monitor lock works**: every Java object has an associated monitor (intrinsic lock). When a thread enters a `synchronized` block, it must acquire the monitor for that object. If another thread already holds it, the requesting thread goes to BLOCKED state. When the lock owner exits the block (normally or via exception), the lock is released and one waiting thread is allowed to acquire it. The lock is **reentrant**: if a thread already holds the lock, it can enter another synchronized block on the same object without blocking itself.

**Follow-up**: What happens if an exception is thrown inside a synchronized block? The lock is still released automatically. `synchronized` uses a structure similar to try-finally at the JVM level.

**Follow-up**: Object lock vs class lock — locking on an instance (`this`) only blocks threads calling methods on that same instance. Locking on the class (`ClassName.class`) blocks all threads across all instances, because there is only one `Class` object in the JVM per class.

### 5. `volatile`

`volatile` is a keyword for a variable. It guarantees:
- **Visibility**: a write to a volatile variable by one thread is immediately visible to other threads. The value is always read from main memory, not from a CPU cache or thread-local copy.
- **No reordering**: the JVM will not reorder instructions around a volatile read/write (this matters for the JMM's happens-before rule).

It does **NOT** guarantee atomicity. Example: `count++` on a volatile `int` is NOT thread-safe, because `++` is really three steps: read, increment, write. Two threads can interleave these steps and lose an update.

```java
private volatile boolean running = true;
public void stop() { running = false; }         // Thread A
public void run() { while (running) { ... } }    // Thread B sees the update
```

**`volatile` vs `synchronized`**:

| | `volatile` | `synchronized` |
|---|---|---|
| Visibility | Yes | Yes |
| Atomicity | No | Yes |
| Blocks threads | No | Yes |
| Use case | Simple flags, single writer | Compound operations, critical sections |
| Performance | Cheaper | More expensive (lock overhead) |

**Follow-up**: When is `volatile` enough? When you have a simple flag that one thread writes and others only read (e.g., a shutdown flag), and there is no compound read-modify-write operation.

### 6. Java Memory Model (JMM) and happens-before

The JMM defines how and when changes made by one thread become visible to other threads. Without the JMM rules, each CPU core may cache variables locally, and a thread may never see another thread's update, or may see it out of order.

**Why this matters**: modern CPUs and compilers reorder instructions for performance, as long as the reordering does not change the result for a single thread. In a multithreaded program, this reordering can break correctness because another thread might observe the reordering.

**happens-before** is a set of rules the JMM guarantees. If action A happens-before action B, then A's result is visible to B, and A is ordered before B. Key happens-before rules:

- **Program order rule**: within one thread, code runs in program order (from that thread's own view).
- **Monitor lock rule**: unlocking a monitor happens-before every later lock of that same monitor by another thread.
- **Volatile variable rule**: a write to a volatile variable happens-before every later read of that same variable.
- **Thread start rule**: `Thread.start()` happens-before any action inside the started thread.
- **Thread join rule**: all actions in a thread happen-before another thread successfully returns from `join()` on it.

**Follow-up**: Why does a plain (non-volatile, non-synchronized) shared variable sometimes appear "stuck" to a thread in an infinite loop? Because there is no happens-before relationship, the JIT compiler may cache the variable in a CPU register and never re-read main memory, so the thread never sees the update.

### 7. Race condition and atomicity

A **race condition** happens when the correctness of a program depends on the timing or interleaving of multiple threads accessing shared data. Classic example:

```java
int count = 0;
void increment() { count++; }   // NOT thread-safe
```

If two threads call `increment()` at the same time, both can read `count = 5`, both increment to `6`, both write `6`. One update is lost.

**Fixes**: `synchronized`, `AtomicInteger`, or locks.

**Follow-up**: Is `count++` one CPU instruction? No. It compiles to read, add, write — three separate steps, so it is not atomic.

### 8. `wait()`, `notify()`, `notifyAll()`

These are methods on `Object` (not `Thread`), used for **inter-thread communication**. A thread calls `wait()` to give up the lock and pause until another thread calls `notify()`/`notifyAll()` on the same object.

- Must be called inside a `synchronized` block on the same object, otherwise `IllegalMonitorStateException` is thrown.
- `wait()` releases the monitor lock and puts the thread in WAITING state.
- `notify()` wakes up one waiting thread (unspecified which one). `notifyAll()` wakes up all waiting threads.
- The woken thread does NOT resume immediately — it must re-acquire the lock first.

```java
synchronized (lock) {
    while (!condition) {
        lock.wait();
    }
    // proceed
}
```

**Why must this be inside `synchronized`?** Because `wait()`/`notify()` operate on the object's monitor. Without holding the monitor, the JVM cannot safely coordinate the release-and-wait or the wake-up. This prevents a race between checking a condition and going to sleep.

**Follow-up**: Why use `while` and not `if` to check the condition? Because of **spurious wakeups** (a thread can wake up without an actual `notify()`) and because another thread might change the condition again before this thread runs. Always re-check the condition in a loop.

**Follow-up**: `wait/notify` vs modern alternatives — most code today prefers `java.util.concurrent` classes like `BlockingQueue`, `CountDownLatch`, or `Condition` (from `ReentrantLock`), because they are safer and easier to get right.

### 9. Deadlock, livelock, starvation

**Deadlock**: two or more threads are stuck forever, each waiting for a lock the other holds.

```java
// Thread 1: synchronized(lockA) { synchronized(lockB) { } }
// Thread 2: synchronized(lockB) { synchronized(lockA) { } }
// If both run at the same time, deadlock can occur.
```

Four necessary conditions (all must hold for deadlock to happen):
1. **Mutual exclusion**: a resource can be held by only one thread.
2. **Hold and wait**: a thread holds one lock while waiting for another.
3. **No preemption**: a lock cannot be forcibly taken from a thread.
4. **Circular wait**: a cycle of threads, each waiting for a lock held by the next.

**How to avoid deadlock**:
- Always acquire locks in the same fixed order across all threads.
- Use a lock timeout (`tryLock(timeout)`) instead of waiting forever.
- Reduce the scope of locking; avoid nested locks when possible.
- Use higher-level concurrency utilities instead of manual locks.

**Livelock**: threads are not blocked, but they keep responding to each other and change state without making progress. Example: two people in a hallway keep stepping aside for each other, both moving but never passing.

**Starvation**: a thread is repeatedly denied access to a resource because other threads keep getting priority (e.g., unfair scheduling, or a thread always losing the race for a lock). Using a **fair lock** (`new ReentrantLock(true)`) can reduce starvation.

**Follow-up**: How do you detect a deadlock in production? Take a thread dump (`jstack <pid>`); the JVM directly reports "Found one Java-level deadlock" with the threads and locks involved.

### 10. `ReentrantLock` and `ReadWriteLock` vs `synchronized`

`ReentrantLock` (in `java.util.concurrent.locks`) is a lock class offering more control than `synchronized`.

```java
Lock lock = new ReentrantLock();
lock.lock();
try {
    // critical section
} finally {
    lock.unlock();
}
```

| Feature | `synchronized` | `ReentrantLock` |
|---|---|---|
| Lock release | Automatic (JVM) | Manual (must call `unlock()` in `finally`) |
| Try without blocking | No | Yes, `tryLock()` |
| Interruptible wait | No | Yes, `lockInterruptibly()` |
| Fairness option | No | Yes, `new ReentrantLock(true)` |
| Condition variables | One implicit (`wait/notify`) | Multiple, via `newCondition()` |
| Performance | Similar in modern JVMs | Similar, slightly more flexible |

**`ReadWriteLock`** (`ReentrantReadWriteLock`) allows multiple readers at the same time, but only one writer, and no readers while writing. Useful when reads are far more frequent than writes.

```java
ReadWriteLock rwLock = new ReentrantReadWriteLock();
rwLock.readLock().lock();   // many threads can hold this together
rwLock.writeLock().lock();  // exclusive
```

**Follow-up**: Why is `finally` mandatory with `ReentrantLock`? Because unlike `synchronized`, the JVM does not auto-release the lock. If an exception occurs and you forget `unlock()` in `finally`, the lock stays held forever, causing a deadlock-like situation.

**Follow-up**: When would you pick `ReentrantLock` over `synchronized`? When you need `tryLock` with timeout, fairness, interruptible locking, or multiple condition queues.

### 11. Atomic classes and CAS

**Atomic classes** (`AtomicInteger`, `AtomicLong`, `AtomicReference`, etc.) provide lock-free, thread-safe operations on a single variable.

```java
AtomicInteger counter = new AtomicInteger(0);
counter.incrementAndGet();   // thread-safe, no lock needed
```

**CAS (compare-and-swap)** is the mechanism behind atomic classes. In simple words: "Change the value only if it still equals what I expect. Otherwise, try again." CAS is a single CPU instruction, so it is much faster than acquiring a lock.

Simple explanation of how `incrementAndGet()` works internally:
1. Read the current value (say `5`).
2. Compute the new value (`6`).
3. Ask the CPU: "if the current value is still `5`, set it to `6`." This is one atomic CPU instruction.
4. If another thread changed the value in between (so it's no longer `5`), the CAS fails, and the method retries from step 1.

This is called **optimistic locking** — no thread is blocked; if a conflict happens, it just retries.

**Follow-up**: What is the ABA problem? If a value changes from A to B and back to A, CAS thinks nothing changed, but it may have. `AtomicStampedReference` solves this by adding a version stamp.

### 12. `ExecutorService` and thread pools

Creating a new thread per task is expensive and unbounded (can crash the JVM under load). A **thread pool** reuses a fixed set of worker threads for many tasks.

```java
ExecutorService executor = Executors.newFixedThreadPool(4);
executor.submit(() -> doWork());
executor.shutdown(); // stops accepting new tasks, lets running tasks finish
```

**Common `Executors` factory methods** (and why they are risky in production):
- `newFixedThreadPool(n)`: fixed threads, **unbounded queue** — can cause `OutOfMemoryError` if tasks pile up faster than they finish.
- `newCachedThreadPool()`: **unbounded threads** — can create too many threads under heavy load and exhaust memory/CPU.
- `newSingleThreadExecutor()`: one thread, unbounded queue — same queue risk.
- `newScheduledThreadPool(n)`: for delayed/periodic tasks.

Because of these hidden unbounded resources, many teams (and tools like SonarQube / Effective Java) recommend building a `ThreadPoolExecutor` manually, so every limit is explicit:

```java
ThreadPoolExecutor executor = new ThreadPoolExecutor(
    4,                                  // core pool size
    10,                                 // max pool size
    60L, TimeUnit.SECONDS,              // idle thread timeout
    new LinkedBlockingQueue<>(100),     // bounded work queue
    new ThreadPoolExecutor.CallerRunsPolicy() // rejection policy
);
```

**How the pool decides what to do with a new task**:
1. If threads < core size, start a new thread.
2. Else, if the queue has space, put the task in the queue.
3. Else, if threads < max size, start a new (extra) thread.
4. Else, apply the **rejection policy**.

**Rejection policies**:
- `AbortPolicy` (default): throws `RejectedExecutionException`.
- `CallerRunsPolicy`: the submitting thread runs the task itself (slows down the producer, acts as backpressure).
- `DiscardPolicy`: silently drops the task.
- `DiscardOldestPolicy`: drops the oldest queued task, then retries.

**Follow-up**: Why is an unbounded queue dangerous? Because the pool will never grow past core size (since the queue "always has room"), and memory can grow without limit if producers are faster than consumers — this often causes production `OutOfMemoryError` incidents.

### 13. `CompletableFuture`

`CompletableFuture` supports writing asynchronous, non-blocking code that can be composed into pipelines, unlike the plain `Future` (which only supports blocking `get()`).

```java
CompletableFuture<Integer> future = CompletableFuture
    .supplyAsync(() -> fetchOrderCount())      // runs async, returns a value
    .thenApply(count -> count * 2)             // transform the result
    .thenApply(result -> result + 1);

future.thenAccept(System.out::println);        // consume result, no return value
```

Key methods:
- **`thenApply(fn)`**: transforms the result (like `map`). Runs on the same thread as the previous stage (or a common pool thread).
- **`thenCompose(fn)`**: chains another `CompletableFuture`-returning function (like `flatMap`). Use this when the next step is itself async, to avoid nested futures.
- **`thenCombine(other, fn)`**: combines results of two independent futures once both complete.
- **`thenAccept(consumer)`**: consumes the result, returns nothing.
- **`exceptionally(fn)`** / **`handle(fn)`**: handle errors in the chain.
- Async variants (`thenApplyAsync`, etc.) run the step on a separate thread from a thread pool (default: `ForkJoinPool.commonPool()`, or a custom `Executor` you pass in).

```java
CompletableFuture<Integer> a = CompletableFuture.supplyAsync(() -> 10);
CompletableFuture<Integer> b = CompletableFuture.supplyAsync(() -> 20);
CompletableFuture<Integer> sum = a.thenCombine(b, Integer::sum); // 30
```

**Follow-up**: `thenApply` vs `thenCompose`? Use `thenApply` when your function returns a plain value. Use `thenCompose` when your function itself returns a `CompletableFuture` — otherwise you would end up with a nested `CompletableFuture<CompletableFuture<T>>`.

### 14. `ThreadLocal`

`ThreadLocal` gives each thread its own independent copy of a variable. Useful for per-thread context, like a database connection, a user session, or a `SimpleDateFormat` instance (which is not thread-safe).

```java
private static final ThreadLocal<SimpleDateFormat> formatter =
    ThreadLocal.withInitial(() -> new SimpleDateFormat("yyyy-MM-dd"));
```

**Memory leak risk in thread pools**: thread pool threads are long-lived and reused across many tasks. If a `ThreadLocal` value is set but never removed, it stays attached to the pooled thread forever, even after the task finishes. Over time, this can leak memory (values from old tasks pile up) and can leak sensitive data across unrelated tasks/requests (e.g., user context from Request A leaking into Request B on the same pooled thread).

**Fix**: always call `threadLocal.remove()` in a `finally` block after use, especially in web request filters and pooled workers.

### 15. Concurrency utilities (`java.util.concurrent`)

- **`CountDownLatch`**: lets one or more threads wait until a set of operations finish. It has a counter that starts at N; `countDown()` decreases it; `await()` blocks until it hits 0. **One-time use only** — cannot be reset.
```java
CountDownLatch latch = new CountDownLatch(3);
// each worker calls latch.countDown() when done
latch.await(); // main thread waits for all 3
```
- **`CyclicBarrier`**: lets a fixed group of threads wait for each other at a common point, then all proceed together. Unlike `CountDownLatch`, it **can be reused** for multiple rounds.
- **`Semaphore`**: controls access to a limited number of permits, useful for limiting concurrent access to a resource (e.g., a connection pool of size 5).
```java
Semaphore semaphore = new Semaphore(5);
semaphore.acquire();
try { /* use resource */ } finally { semaphore.release(); }
```
- **`BlockingQueue`**: a thread-safe queue where `put()` blocks if full, and `take()` blocks if empty. The classic backbone of the producer-consumer pattern. Implementations: `ArrayBlockingQueue` (bounded), `LinkedBlockingQueue`, `PriorityBlockingQueue`.
- **`ConcurrentHashMap`**: a thread-safe map. It does not lock the whole map; internally it splits work across bucket-level locks (or CAS for many operations in modern versions), so many threads can read/write different parts concurrently. Much faster than `Collections.synchronizedMap()` under contention.
- **`CopyOnWriteArrayList`**: a thread-safe list where every write (add/remove) creates a new copy of the underlying array. Reads never block and never see a `ConcurrentModificationException`. Best for read-heavy, write-rare scenarios (e.g., listener lists).

**Follow-up**: `CountDownLatch` vs `CyclicBarrier`? `CountDownLatch` is for one-time "wait for N things to finish." `CyclicBarrier` is for "N threads wait for each other repeatedly, at each round."

### 16. `ForkJoinPool` basics

`ForkJoinPool` is designed for **divide-and-conquer** tasks: split a big task into smaller subtasks recursively, run them in parallel, then combine results. It powers `parallelStream()` and classes like `RecursiveTask`/`RecursiveAction`.

It uses **work-stealing**: each worker thread has its own task queue. If a thread finishes its own queue early, it "steals" tasks from the back of another busy thread's queue. This keeps all CPU cores busy and improves throughput for uneven workloads.

```java
ForkJoinPool.commonPool().submit(() -> {
    // recursive divide-and-conquer task
});
```

**Follow-up**: When should you NOT use `parallelStream()`? For small collections (overhead outweighs benefit), for I/O-bound tasks (ForkJoinPool is tuned for CPU-bound work), or when task order/thread-safety of the operation is not guaranteed.

### 17. How to make code thread-safe

Three general strategies:
1. **Immutability**: if an object cannot change after creation (all fields `final`, no setters), it is automatically thread-safe. No synchronization needed. (e.g., `String`, records with final fields.)
2. **Thread confinement**: keep data local to one thread only, so no sharing happens (e.g., `ThreadLocal`, or local variables that are never shared).
3. **Synchronization**: use `synchronized`, `ReentrantLock`, or concurrent collections to control access to shared mutable state.

**Rule of thumb for interviews**: prefer immutability first, then confinement, then synchronization — in that order, because less shared mutable state means fewer bugs.

## FAQ / Rapid-Fire

- **Q: Can two threads call two different synchronized methods on the same object at the same time?**
  No, if both methods are instance-level synchronized. They share the same object lock, so only one can run at a time.

- **Q: Is `String` thread-safe?**
  Yes, because `String` is immutable.

- **Q: Is `ArrayList` thread-safe?**
  No. Use `Collections.synchronizedList()`, `CopyOnWriteArrayList`, or external synchronization.

- **Q: What is the default value of a `volatile` field's visibility guarantee for compound actions like `count++`?**
  None. `volatile` gives visibility only, not atomicity. Use `AtomicInteger` for compound actions.

- **Q: Difference between `Runnable` and `Callable`?**
  `Runnable.run()` returns `void` and cannot throw checked exceptions. `Callable.call()` returns a value (`Future<T>`) and can throw checked exceptions.

- **Q: What does `Future.get()` do?**
  It blocks the calling thread until the task's result is ready, or throws an exception if the task failed.

- **Q: Why avoid `Thread.stop()`?**
  It is deprecated and unsafe — it can release locks mid-operation, leaving shared objects in a broken state. Use a `volatile` flag or `interrupt()` instead.

- **Q: What does `Thread.interrupt()` do?**
  It sets an interrupt flag on the thread. If the thread is blocked in `sleep()`/`wait()`/`join()`, it throws `InterruptedException`. It does not forcibly stop the thread — the thread must check the flag or handle the exception.

- **Q: What is a daemon thread?**
  A background thread that does not stop the JVM from exiting (e.g., garbage collector thread). Set with `thread.setDaemon(true)` before `start()`.

- **Q: Why is `Vector` considered outdated for concurrency?**
  It synchronizes every method individually, which is slower and still not safe for compound operations (like check-then-act), compared to modern `java.util.concurrent` classes.

## Common Traps & Gotchas

- **Assuming `volatile` makes `count++` safe.** It does not. `volatile` gives visibility, not atomicity. Use `AtomicInteger` or `synchronized`.
- **Forgetting `finally` with `ReentrantLock`.** If you skip `finally { lock.unlock(); }`, an exception can leave the lock held forever.
- **Using `Executors.newFixedThreadPool()` / `newCachedThreadPool()` in production without limits.** Their queues or thread counts are effectively unbounded, which can cause `OutOfMemoryError` under load. Prefer a manually configured `ThreadPoolExecutor` with a bounded queue and a rejection policy.
- **Calling `wait()`/`notify()` outside a `synchronized` block.** This throws `IllegalMonitorStateException`.
- **Using `if` instead of `while` to check a wait condition.** Spurious wakeups can cause the thread to proceed incorrectly. Always loop and re-check.
- **Not calling `ThreadLocal.remove()` in pooled threads.** This leaks memory and can leak stale data into the next task on the same reused thread.
- **Forgetting `thenCompose` vs `thenApply`.** Using `thenApply` when the mapping function itself returns a `CompletableFuture` creates a nested future (`CompletableFuture<CompletableFuture<T>>`) instead of a flattened one.
- **Iterating a `HashMap`/`ArrayList` from multiple threads without synchronization.** This can throw `ConcurrentModificationException` or corrupt internal structure. Use `ConcurrentHashMap` / `CopyOnWriteArrayList`, or synchronize externally.
- **Assuming `ConcurrentHashMap` makes compound operations (like check-then-put) atomic.** Individual method calls are thread-safe, but a sequence like `if (!map.containsKey(k)) map.put(k, v)` is not atomic across two calls. Use `putIfAbsent()` or `compute()` instead.
- **Locking on a mutable or non-final field, or on a `String` literal / boxed `Integer`.** These can be unintentionally shared across unrelated code (due to string interning or integer caching), causing surprise contention or missed exclusion. Lock on a dedicated private `final Object` instead.
- **Nested locks acquired in inconsistent order across different code paths.** This is the most common real-world cause of deadlocks. Always acquire multiple locks in the same global order.
- **Forgetting `executor.shutdown()`.** An `ExecutorService` with non-daemon threads keeps the JVM alive even after `main()` finishes, if never shut down.
