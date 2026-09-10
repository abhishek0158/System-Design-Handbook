# Custom Thread Pool

## Problem

Build your own thread pool in Java. Do not use `Executors` or `ExecutorService` from `java.util.concurrent`. Build the logic yourself, using raw `Thread` objects and a queue.

A thread pool keeps a fixed number of worker threads alive. Each worker thread runs in a loop. It takes a task from a shared queue and runs it. When there is no task, the worker waits.

The pool must support:

- `submit(Runnable task)` — add a task to the pool. If the internal queue is full, decide what happens (block the caller, or reject the task).
- `shutdown()` — a **graceful** shutdown. Stop accepting new tasks, but let all queued and running tasks finish first.
- `shutdownNow()` — an **immediate** shutdown. Stop accepting new tasks, and try to stop running tasks as soon as possible.

This is a common interview question for 3–4 years experience level. It tests whether you understand what `ThreadPoolExecutor` does internally, not just whether you know how to call `Executors.newFixedThreadPool(n)`.

## Requirements & Clarifying Questions

Before coding, ask these questions. Here are the answers we will assume for the main solution.

1. **How many worker threads should the pool have?**
   A fixed number, given in the constructor. This is the simplest model. We will discuss a variable-size pool (core vs. maximum) in the follow-ups section.

2. **What happens when a task is submitted but the queue is full?**
   We need a policy. The two common choices are: block the calling thread until space frees up, or reject the task right away (for example, by throwing an exception). We will build the **bounded queue that blocks**, and then show how to plug in a **rejection policy** instead, matching what `ThreadPoolExecutor` does.

3. **What does "graceful shutdown" mean exactly?**
   No new tasks are accepted after `shutdown()` is called. But tasks already in the queue, and tasks already running, must be allowed to finish normally.

4. **What does `shutdownNow()` mean exactly?**
   No new tasks are accepted. Queued tasks that have not started are discarded (or returned to the caller). Running tasks are asked to stop via **thread interruption** — Java has no safe way to force-kill a thread, so "stop now" really means "ask nicely, right now."

5. **Should `submit()` return a result, like `Future` does?**
   Not for the base version. We will keep `submit(Runnable)` simple. We will mention how to add a `Future`-like result in the follow-ups.

6. **What happens if a task throws an exception?**
   The worker thread must catch it, log it (or otherwise report it), and keep running. One bad task must not kill a worker thread, because a dead worker permanently shrinks the pool.

7. **Is the queue unbounded or bounded?**
   Bounded, with a fixed capacity. An unbounded queue can grow forever under heavy load and use up all memory. This is a real, well-known production bug, so the interviewer usually wants a bounded queue by default.

## Design / Approach

A thread pool has three moving parts:

1. **A work queue.** A thread-safe queue that holds `Runnable` tasks waiting to run. We can build this from scratch, as shown in a previous problem (see the custom blocking queue), or reuse `java.util.concurrent.BlockingQueue`, which is a standard, well-tested interface for exactly this purpose. For this problem, we use `BlockingQueue` for the queue itself, since the question here is about the **pool logic** — worker threads, submit, shutdown — not about re-deriving `wait()`/`notify()` again.

2. **A fixed set of worker threads.** Each worker thread runs a loop:
    - Take a task from the queue. This call blocks if the queue is empty.
    - Run the task.
    - Catch any exception the task throws, so the loop does not stop.
    - Repeat, until told to stop.

3. **Shutdown coordination.** We need a shared flag (or a small state machine) that tells workers "no new tasks, but finish what is queued" (graceful), or "stop right now" (immediate). We also need a way to wake up threads that are blocked waiting on the queue, so they notice the shutdown signal instead of waiting forever.

**Why a `BlockingQueue` fits naturally here.** `submit()` needs to add a task and, if the queue is full, either block or reject. `BlockingQueue.put()` blocks when full. `BlockingQueue.offer()` returns `false` right away when full, without blocking. This maps directly to the two policies the requirements ask for.

**State machine for the pool.** We track pool state with a simple `enum` or `volatile` flags:

- `RUNNING` — accepts tasks normally.
- `SHUTDOWN` — rejects new tasks, but workers keep pulling from the queue until it is empty.
- `STOP` — rejects new tasks, discards remaining queued tasks, and interrupts running workers.
- `TERMINATED` — all worker threads have exited.

**Waking up idle workers on shutdown.** A worker blocked in `queue.take()` will not notice a plain boolean flag change, because it is asleep, not checking the flag in a loop. Two common fixes:
- Use `queue.poll(timeout, unit)` instead of `take()`, so the worker wakes up periodically on its own and checks the shutdown state.
- For `shutdownNow()`, call `interrupt()` on every worker thread. `poll()`/`take()` both throw `InterruptedException` when interrupted, which breaks the worker out of its blocked wait immediately.

We use the interrupt approach for `shutdownNow()`, and rely on poison-pill-free polling (or interruption plus queue draining) for graceful shutdown. Below, we use interruption for both, since it is the standard, reliable mechanism, and explain the small difference in behavior between the two shutdown modes.

## Java Solution

```java
import java.util.ArrayList;
import java.util.List;
import java.util.concurrent.ArrayBlockingQueue;
import java.util.concurrent.BlockingQueue;
import java.util.concurrent.TimeUnit;
import java.util.concurrent.atomic.AtomicInteger;

public class CustomThreadPool {

    // Pool lifecycle states.
    private enum State { RUNNING, SHUTDOWN, STOP, TERMINATED }

    private volatile State state = State.RUNNING;

    private final BlockingQueue<Runnable> taskQueue;
    private final List<WorkerThread> workers;
    private final AtomicInteger activeWorkerCount;

    public CustomThreadPool(int poolSize, int queueCapacity) {
        if (poolSize <= 0 || queueCapacity <= 0) {
            throw new IllegalArgumentException("poolSize and queueCapacity must be > 0");
        }
        this.taskQueue = new ArrayBlockingQueue<>(queueCapacity);
        this.workers = new ArrayList<>(poolSize);
        this.activeWorkerCount = new AtomicInteger(0);

        for (int i = 0; i < poolSize; i++) {
            WorkerThread worker = new WorkerThread("pool-worker-" + i);
            workers.add(worker);
            worker.start();
        }
    }

    /**
     * Submit a task. Blocks the caller if the queue is full.
     * Throws if the pool is no longer accepting tasks.
     */
    public void submit(Runnable task) throws InterruptedException {
        if (task == null) {
            throw new NullPointerException("task cannot be null");
        }
        if (state != State.RUNNING) {
            throw new RejectedExecutionException("Pool is not accepting new tasks, state=" + state);
        }
        taskQueue.put(task); // blocks here if the queue is full
    }

    /**
     * Submit a task without blocking. Returns false right away if the
     * queue is full or the pool is not running (a "reject" policy).
     */
    public boolean trySubmit(Runnable task) {
        if (task == null) {
            throw new NullPointerException("task cannot be null");
        }
        if (state != State.RUNNING) {
            return false;
        }
        return taskQueue.offer(task); // false immediately if full, no waiting
    }

    /** Graceful shutdown: stop accepting tasks, let queued and running tasks finish. */
    public void shutdown() {
        state = State.SHUTDOWN;
        // Workers keep draining the queue on their own; no interrupt needed here.
        // We do interrupt idle workers so they wake from a blocked take()/poll()
        // and notice the new state instead of waiting for the next task forever.
        for (WorkerThread worker : workers) {
            worker.interruptIfIdle();
        }
    }

    /** Immediate shutdown: stop accepting tasks, drop queued tasks, interrupt running tasks. */
    public List<Runnable> shutdownNow() {
        state = State.STOP;
        List<Runnable> notStarted = new ArrayList<>();
        taskQueue.drainTo(notStarted); // remove and return everything still queued
        for (WorkerThread worker : workers) {
            worker.thread.interrupt(); // ask running tasks to stop, wake idle workers
        }
        return notStarted;
    }

    /** Blocks until all worker threads have exited, or the timeout expires. */
    public boolean awaitTermination(long timeout, TimeUnit unit) throws InterruptedException {
        long deadline = System.nanoTime() + unit.toNanos(timeout);
        for (WorkerThread worker : workers) {
            long remainingNanos = deadline - System.nanoTime();
            if (remainingNanos <= 0) {
                return isTerminated();
            }
            worker.thread.join(remainingNanos / 1_000_000, (int) (remainingNanos % 1_000_000));
        }
        return isTerminated();
    }

    public boolean isTerminated() {
        if (state == State.RUNNING) {
            return false;
        }
        for (WorkerThread worker : workers) {
            if (worker.thread.isAlive()) {
                return false;
            }
        }
        state = State.TERMINATED;
        return true;
    }

    public int getQueueSize() {
        return taskQueue.size();
    }

    public int getActiveCount() {
        return activeWorkerCount.get();
    }

    /** A rejection exception, similar in spirit to the built-in RejectedExecutionException. */
    public static class RejectedExecutionException extends RuntimeException {
        public RejectedExecutionException(String message) {
            super(message);
        }
    }

    private class WorkerThread {
        private final Thread thread;
        private volatile boolean idle = true;

        WorkerThread(String name) {
            this.thread = new Thread(this::runLoop, name);
        }

        void start() {
            thread.start();
        }

        void interruptIfIdle() {
            if (idle) {
                thread.interrupt();
            }
        }

        private void runLoop() {
            while (true) {
                // STOP means: do not even try to pull more tasks, exit now.
                if (state == State.STOP) {
                    return;
                }
                Runnable task;
                try {
                    idle = true;
                    task = taskQueue.poll(200, TimeUnit.MILLISECONDS);
                } catch (InterruptedException e) {
                    // Woken up on purpose (shutdown signal) or spuriously.
                    // Loop back to the top and re-check the state.
                    continue;
                }
                idle = false;

                if (task == null) {
                    // No task arrived within the poll timeout.
                    // If we are shutting down and the queue is now empty, exit.
                    if (state != State.RUNNING && taskQueue.isEmpty()) {
                        return;
                    }
                    continue; // still RUNNING with an empty queue, keep polling
                }

                activeWorkerCount.incrementAndGet();
                try {
                    task.run();
                } catch (Throwable t) {
                    // A task must never kill the worker thread.
                    System.err.println("Task threw an exception: " + t);
                } finally {
                    activeWorkerCount.decrementAndGet();
                }
            }
        }
    }
}
```

## How It Works

**Startup.** The constructor creates `poolSize` `WorkerThread` instances and starts them right away. Each one runs `runLoop()` on its own `Thread`.

**The worker loop.** Each worker repeatedly calls `taskQueue.poll(200, TimeUnit.MILLISECONDS)`. This is a **timed** blocking call: it waits up to 200 milliseconds for a task, then returns `null` if nothing arrived. Using a timed poll instead of an untimed `take()` gives the worker a chance to wake up on its own, on a short cycle, and check whether the pool is shutting down. This is what lets a worker with an empty queue notice a graceful `shutdown()` without needing to be interrupted.

**`submit()` versus `trySubmit()`.** `submit()` calls `taskQueue.put(task)`, which blocks the caller when the queue is full, until a worker frees up space. `trySubmit()` calls `taskQueue.offer(task)`, which returns `false` immediately instead of blocking. Both check `state` first and refuse new work once the pool is no longer `RUNNING`. This directly matches the two options the requirements list: block, or reject.

**Graceful `shutdown()`.** It sets `state = SHUTDOWN`. No thread stops immediately. Workers keep running their loop: they keep pulling and running tasks from `taskQueue` until it becomes empty. Once a worker's `poll()` times out with `task == null` **and** the state is no longer `RUNNING` **and** the queue is empty, that worker exits. We also interrupt any worker that is currently idle (waiting in `poll()`), so it does not have to wait out the rest of its 200ms cycle — this makes shutdown feel more responsive without changing the graceful contract, since an idle worker has no task in flight to interrupt.

**Immediate `shutdownNow()`.** It sets `state = STOP` and immediately drains the queue with `drainTo()`, so no new worker will start any queued task. It also calls `interrupt()` on every worker thread unconditionally. If a worker is running a task, and that task is written to check `Thread.interrupted()` or respects `InterruptedException` (for example, it calls `Thread.sleep()` or a blocking I/O call), the interrupt will cause it to stop early. If the task ignores interruption, it will keep running until it finishes naturally — Java cannot force-kill a thread. After the current task (if any) finishes, the worker's next loop iteration sees `state == STOP` and returns right away, without polling the queue again.

**`awaitTermination()`.** This lets the caller block until every worker thread has actually exited, similar to `ExecutorService.awaitTermination()`. It uses `Thread.join(timeout)` on each worker in turn, budgeting the remaining time across all of them.

## How to Extend (Follow-ups)

Interviewers commonly push this problem further. Here is how to map each extension to real `ThreadPoolExecutor` behavior.

- **Core pool size vs. maximum pool size.** Real `ThreadPoolExecutor` has two sizes. `corePoolSize` threads stay alive even when idle. If the queue fills up, the pool can create extra threads, up to `maximumPoolSize`, to handle the burst. Extra threads beyond `corePoolSize` that stay idle for longer than `keepAliveTime` are shut down and removed. To add this to our pool: track worker count as a variable, not a fixed list; when `submit()` finds the queue full and current worker count is below `maximumPoolSize`, spawn a new temporary worker; give temporary workers an idle timeout on their `poll()` call, and let them exit (not loop) if `poll()` times out and current worker count is above `corePoolSize`.

- **`keepAliveTime`.** This is exactly the idle timeout mentioned above. It only matters for threads beyond `corePoolSize` — core threads normally wait forever (unless `allowCoreThreadTimeOut(true)` is set on the real `ThreadPoolExecutor`).

- **Rejection policies.** Real `ThreadPoolExecutor` takes a `RejectedExecutionHandler`. Its four built-in policies are: `AbortPolicy` (throw `RejectedExecutionException`, the default), `CallerRunsPolicy` (run the task on the caller's own thread — this slows the caller down, which acts as natural backpressure), `DiscardPolicy` (silently drop the task), and `DiscardOldestPolicy` (drop the oldest queued task, then retry). Our `trySubmit()` is closest to a "caller decides what to do next" version of `AbortPolicy` since it just returns `false`. Adding a pluggable handler interface, called from inside `submit()` when `offer()` fails, would make this configurable the same way.

- **Returning a result: a `Future`-like type.** Wrap each submitted `Runnable` or `Callable<T>` in a small task object holding a result field, an exception field, and a `CountDownLatch` (or `wait`/`notify`) so the submitting thread can call `get()` and block until the worker sets the result. This is a simplified version of what `FutureTask` does.

- **Named thread factory.** Accept a `ThreadFactory` in the constructor instead of hardcoding `new Thread(...)`, so callers can control thread names, daemon status, and priority. This is what real `ThreadPoolExecutor` does too.

- **Metrics.** Track `submittedCount`, `completedCount`, and `rejectedCount` using `AtomicLong`, so callers can monitor pool health, similar to `ThreadPoolExecutor.getCompletedTaskCount()`.

**Why prefer configuring `ThreadPoolExecutor` directly, instead of `Executors.newFixedThreadPool(n)` or similar factory methods:**

- `Executors.newFixedThreadPool(n)` creates a `ThreadPoolExecutor` with an **unbounded** `LinkedBlockingQueue`. Under sustained heavy load, this queue can grow without limit and exhaust memory, causing an `OutOfMemoryError`. There is no backpressure at all — `submit()` never blocks and never rejects, it just queues forever.
- `Executors.newCachedThreadPool()` has an unbounded **maximum** pool size (`Integer.MAX_VALUE`). Under a burst of short tasks, it can create an unbounded number of threads, which can also exhaust memory or overwhelm the OS scheduler.
- Both factory methods hide the real, important tuning knobs: queue capacity, `maximumPoolSize`, `keepAliveTime`, and the rejection policy. When you construct `new ThreadPoolExecutor(core, max, keepAlive, unit, boundedQueue, rejectionHandler)` directly, you are forced to make each of these choices explicitly, which avoids silent capacity problems in production.
- This is also why many style guides and static analysis tools (for example, ErrorProne, and Java library guidelines at larger companies) flag `Executors.newFixedThreadPool` and similar calls, and recommend the explicit `ThreadPoolExecutor` constructor instead.

## Complexity & Thread-Safety Notes

- **Time complexity:** `submit()` and `trySubmit()` are O(1) beyond the underlying `BlockingQueue` operation cost, which is O(1) for `ArrayBlockingQueue`. Each worker's loop iteration is O(1) plus the cost of running the task itself.
- **Space complexity:** O(queueCapacity + poolSize) — the queue holds at most `queueCapacity` pending tasks, and there are exactly `poolSize` worker threads (in the fixed-size version).
- **Thread safety of the queue.** `ArrayBlockingQueue` is already internally thread-safe, using a `ReentrantLock` with two conditions, much like the custom blocking queue built in an earlier problem. We do not need extra locking around it.
- **Thread safety of `state`.** `state` is `volatile`. This guarantees that once one thread writes a new state (for example, `shutdown()` setting `SHUTDOWN`), every other thread's next read of `state` sees that new value. `volatile` alone is enough here because we only ever assign whole new enum values, never read-then-update `state` based on its old value from multiple threads at once.
- **`activeWorkerCount`.** Uses `AtomicInteger` because multiple worker threads increment and decrement it concurrently, and we need atomic, race-free updates without a full lock.
- **No lost tasks under graceful shutdown.** Because we drain the queue naturally, through the normal `poll()`/`run()` loop, instead of clearing it, every task submitted before `shutdown()` was called is guaranteed to run exactly once (assuming it does not throw before completing, and the JVM does not crash).
- **Tasks may be lost under `shutdownNow()`.** This is by design and matches `ExecutorService.shutdownNow()`, which also returns the list of tasks that never started, so the caller can decide what to do with them (retry, log, discard).
- **Interruption is cooperative.** `shutdownNow()` interrupts worker threads, but a task that ignores `InterruptedException` and never checks `Thread.currentThread().isInterrupted()` will keep running to completion. This is a fundamental property of Java threads, not a bug in this design.

## Interview Tips & Common Mistakes

- **Do not let a task's exception kill the worker thread.** If `task.run()` throws and you do not catch it inside the loop, the worker thread's `run()` method exits, the thread dies, and the pool permanently loses one worker. Always wrap `task.run()` in `try { ... } catch (Throwable t) { ... }` inside the loop, not around the whole loop.

- **Distinguish `shutdown()` from `shutdownNow()` clearly, out loud.** Interviewers listen for the words "graceful" (finish what is queued) versus "best-effort immediate" (interrupt, drop the queue). Getting this distinction right, and explaining why Java cannot force-kill a thread, is often the main point of the question.

- **Explain why an unbounded queue is dangerous.** This is the single most common real-world mistake this problem is designed to catch. If you jump straight to "just use `Executors.newFixedThreadPool`," an interviewer at this level will likely ask you why that can be a problem in production — know the unbounded-queue answer.

- **Do not use `take()` everywhere without a plan for shutdown.** If a worker is blocked forever in `queue.take()`, and the queue never receives another item, that worker will never notice a shutdown request unless something interrupts it. Either use a timed `poll()` (as shown here), or make sure every shutdown path calls `interrupt()` on every worker.

- **Be ready to compare `submit()`-blocks versus reject-immediately.** State clearly which one you are building and why. Blocking, with `put()`, applies natural backpressure to the caller. Rejecting, with `offer()` or a policy object, gives the caller a chance to handle overload itself (retry later, run inline, drop the task, alert someone).

- **Mention `ThreadPoolExecutor`'s constructor parameters by name:** `corePoolSize`, `maximumPoolSize`, `keepAliveTime`, `TimeUnit`, `BlockingQueue<Runnable>`, and `RejectedExecutionHandler`. Being able to name all six, and explain what each does, is usually worth more in this interview than the from-scratch code itself.

- **Mention the built-in alternative.** In real production code, you would use `java.util.concurrent.ThreadPoolExecutor` directly, constructed explicitly rather than through `Executors`. This exercise exists to prove you understand what happens inside it, not to replace it.
