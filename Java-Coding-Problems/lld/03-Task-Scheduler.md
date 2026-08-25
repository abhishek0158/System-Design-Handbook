# Task Scheduler (LLD)

## Problem

Design an in-memory **task scheduler**. A client can submit a task with:
- a **delay**, so it runs once after that delay, or
- a **fixed interval**, so it repeats again and again after each run.

The scheduler must run each task on time, using a pool of worker threads. It must pick the
task with the **earliest run time** first. It must support **cancel**, so a task does not run
again after cancel. A worker thread must not **busy-wait** (spin in a loop checking the clock
again and again). It should sleep until the next task is actually due.

This is a common machine-coding round for a 3–4 year backend developer. The interviewer wants
to see that you can build a correct, thread-safe scheduler from basic building blocks, not just
call `Executors.newScheduledThreadPool(...)`. You should also be able to explain how your
design compares to the built-in `ScheduledThreadPoolExecutor`.

## Requirements & Clarifying Questions

Questions to ask the interviewer before coding:

1. **One-time vs recurring** — Do we need both? (Yes, both. One-time = run once after delay.
   Recurring = run again and again at a fixed interval.)
2. **Fixed-rate or fixed-delay for recurring tasks?**
    - Fixed-rate: next run time = last **scheduled** time + interval (can catch up if a run was
      late).
    - Fixed-delay: next run time = **finish** time of last run + interval (never catches up).
    - For this design, we use **fixed-delay** semantics. It is simpler and safer, because a slow
      task cannot cause many queued-up runs to fire back to back.
3. **Cancel** — Can a task be cancelled while it is running, or only before its next run?
   (We support "cancel before next run." A task already running finishes normally, but a
   recurring task will not be scheduled again after cancel.)
4. **What happens if a task throws an exception?** (The worker should catch it, log it, and
   keep going. One bad task must not kill a worker thread or the whole scheduler.)
5. **How many worker threads?** (Fixed-size pool, configurable at construction. Interviewer
   may ask you to make it dynamic — see Follow-ups.)
6. **Ordering guarantee** — If two tasks are due at the same time, does order matter?
   (Not for correctness. We can break ties by insertion sequence for determinism.)
7. **Precision** — Do we need millisecond precision, or is "close enough" (a few ms of jitter)
   fine? (A few ms of jitter is fine. This is not a hard real-time system.)
8. **Shutdown** — Should `shutdown()` let running tasks finish, or stop everything at once?
   (Graceful: stop pulling new tasks, let in-flight tasks finish.)

These questions show the interviewer you think about semantics, not just about writing code
that compiles.

## Design / Approach

### Core idea

We keep all pending tasks in one **thread-safe, time-ordered queue**. The queue always gives
us the task with the smallest "next run time" first. A small pool of **worker threads** takes
tasks from this queue. Each worker either:
- finds a task that is already due, and runs it, or
- finds a task that is not due yet, and **waits** only until that task's due time (no busy loop).

After a recurring task finishes, the worker computes its next run time and puts it back into
the queue. A one-time task is simply dropped after it runs.

```
                +-----------------------------+
   submit() --> |  DelayQueue<ScheduledTask>  |  <-- thread-safe, orders by nextRunTime
   schedule()   |  (priority = earliest first)|
                +-----------------------------+
                         |   take() blocks until head is due
                         v
        +---------+  +---------+  +---------+
        | Worker1 |  | Worker2 |  | Worker3 |   <- fixed thread pool
        +---------+  +---------+  +---------+
             |             |             |
             v             v             v
        run task.run()  ... catch exceptions ...
             |
             v
     if recurring and not cancelled:
        compute next run time, queue.put(task) again
     else: drop it
```

### Key building blocks

1. **`ScheduledTask`** — wraps a `Runnable`, its next run time, its interval (0 if one-time),
   and a `cancelled` flag. Implements `Delayed` so it can sit in a `DelayQueue`.
2. **`DelayQueue<ScheduledTask>`** — a `BlockingQueue` from `java.util.concurrent`. It is
   already thread-safe, and it already blocks `take()` until the head element's delay has
   expired. This gives us "no busy-wait" and "earliest task first" for free. This is the
   standard interview-approved way to build this without writing your own wait/notify heap
   from scratch (though you can also do that — see the alternative below).
3. **`TaskHandle`** — returned to the caller on `schedule(...)`. Lets the caller call
   `cancel()` on a specific task.
4. **`TaskScheduler`** — owns the `DelayQueue` and a fixed `ExecutorService` (worker pool).
   It runs one **dispatcher loop per worker thread**: `queue.take()` then run then (if
   recurring) requeue.

### Alternative to `DelayQueue`

You could use a `PriorityBlockingQueue<ScheduledTask>` with a comparator on `nextRunTime`,
plus a single dispatcher thread that does `wait(timeUntilNextTask)` on a lock, and
`notify()` when a new task is added with an earlier time than the current head. This is more
code, but it is a good follow-up to discuss: it shows you understand what `DelayQueue` does
internally. In this solution we use `DelayQueue`, because it is production-grade, less
error-prone, and still shows full understanding of the requirement ("thread-safe,
time-ordered, no busy-wait").

### Patterns used

- **Producer–Consumer**, with `DelayQueue` as the shared buffer. Client threads produce tasks.
  Worker threads consume them.
- **Command pattern**: each task wraps a `Runnable` (a command to execute later).
- **Strategy** (optional extension): the "next run time" calculation could be pulled out into a
  `RecurrenceStrategy` interface, to support cron-like schedules later (see Follow-ups).

## Java Solution

```java
import java.util.concurrent.*;
import java.util.concurrent.atomic.AtomicBoolean;
import java.util.concurrent.atomic.AtomicLong;

/**
 * A task that knows when it should run next, and whether it repeats.
 * Implements Delayed so it can be ordered inside a DelayQueue.
 */
final class ScheduledTask implements Delayed {

    private static final AtomicLong SEQUENCE = new AtomicLong();

    private final long id;
    private final Runnable job;
    private final long intervalMillis;   // 0 means "one-time task"
    private final long sequenceNo;       // tie-breaker for equal run times
    private volatile long nextRunTimeMillis; // epoch millis of next run
    private final AtomicBoolean cancelled = new AtomicBoolean(false);

    ScheduledTask(Runnable job, long initialDelayMillis, long intervalMillis) {
        this.id = SEQUENCE.incrementAndGet();
        this.sequenceNo = this.id;
        this.job = job;
        this.intervalMillis = intervalMillis;
        this.nextRunTimeMillis = System.currentTimeMillis() + initialDelayMillis;
    }

    long id() {
        return id;
    }

    boolean isRecurring() {
        return intervalMillis > 0;
    }

    boolean isCancelled() {
        return cancelled.get();
    }

    void cancel() {
        cancelled.set(true);
    }

    Runnable job() {
        return job;
    }

    /** Called by the worker after a fixed-delay run finishes, to push the task forward. */
    void scheduleNextRun() {
        this.nextRunTimeMillis = System.currentTimeMillis() + intervalMillis;
    }

    @Override
    public long getDelay(TimeUnit unit) {
        long diff = nextRunTimeMillis - System.currentTimeMillis();
        return unit.convert(diff, TimeUnit.MILLISECONDS);
    }

    @Override
    public int compareTo(Delayed other) {
        if (other == this) {
            return 0;
        }
        if (other instanceof ScheduledTask o) {
            int cmp = Long.compare(this.nextRunTimeMillis, o.nextRunTimeMillis);
            if (cmp != 0) {
                return cmp;
            }
            return Long.compare(this.sequenceNo, o.sequenceNo);
        }
        long diff = getDelay(TimeUnit.MILLISECONDS) - other.getDelay(TimeUnit.MILLISECONDS);
        return Long.compare(diff, 0L);
    }
}
```

```java
/** Handle returned to the caller. Lets the caller cancel a task later. */
public final class TaskHandle {

    private final ScheduledTask task;

    TaskHandle(ScheduledTask task) {
        this.task = task;
    }

    public void cancel() {
        task.cancel();
    }

    public long taskId() {
        return task.id();
    }
}
```

```java
import java.util.concurrent.*;
import java.util.logging.Level;
import java.util.logging.Logger;

/**
 * Thread-safe task scheduler.
 * - schedule(job, delay): run once after 'delay'.
 * - scheduleAtFixedDelay(job, initialDelay, interval): run repeatedly.
 * - A fixed worker pool pulls tasks from a DelayQueue, which blocks until
 *   the earliest task is due (no busy-wait) and orders tasks by run time.
 */
public final class TaskScheduler {

    private static final Logger log = Logger.getLogger(TaskScheduler.class.getName());

    private final DelayQueue<ScheduledTask> queue = new DelayQueue<>();
    private final ExecutorService workers;
    private final int workerCount;
    private volatile boolean running = true;

    public TaskScheduler(int workerCount) {
        if (workerCount < 1) {
            throw new IllegalArgumentException("workerCount must be >= 1");
        }
        this.workerCount = workerCount;
        this.workers = Executors.newFixedThreadPool(workerCount, r -> {
            Thread t = new Thread(r, "task-scheduler-worker");
            t.setDaemon(true); // do not block JVM shutdown
            return t;
        });
        startWorkers();
    }

    /** Schedule a one-time task to run once after delayMillis. */
    public TaskHandle schedule(Runnable job, long delayMillis) {
        return submit(job, delayMillis, 0);
    }

    /** Schedule a recurring task: first run after initialDelayMillis, then every
     *  intervalMillis after each run finishes (fixed-delay semantics). */
    public TaskHandle scheduleAtFixedDelay(Runnable job, long initialDelayMillis, long intervalMillis) {
        if (intervalMillis <= 0) {
            throw new IllegalArgumentException("intervalMillis must be > 0 for recurring tasks");
        }
        return submit(job, initialDelayMillis, intervalMillis);
    }

    private TaskHandle submit(Runnable job, long delayMillis, long intervalMillis) {
        if (!running) {
            throw new RejectedExecutionException("Scheduler is shut down");
        }
        ScheduledTask task = new ScheduledTask(job, delayMillis, intervalMillis);
        queue.put(task);
        return new TaskHandle(task);
    }

    private void startWorkers() {
        for (int i = 0; i < workerCount; i++) {
            workers.submit(this::workerLoop);
        }
    }

    private void workerLoop() {
        while (running) {
            ScheduledTask task;
            try {
                // Blocks here until the earliest task's delay has passed.
                // No busy-wait: DelayQueue internally uses a Condition with
                // a timed wait for exactly this amount of time.
                task = queue.take();
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
                return; // scheduler is shutting down
            }

            if (task.isCancelled()) {
                continue; // drop silently, do not run, do not reschedule
            }

            runSafely(task.job());

            // Re-check cancellation: task could have been cancelled while it ran.
            if (task.isRecurring() && !task.isCancelled() && running) {
                task.scheduleNextRun();
                queue.put(task);
            }
        }
    }

    private void runSafely(Runnable job) {
        try {
            job.run();
        } catch (Throwable t) {
            // One bad task must never kill a worker thread.
            log.log(Level.WARNING, "Task threw an exception", t);
        }
    }

    /** Stop accepting new tasks; let already-running tasks finish; wake up idle workers. */
    public void shutdown() {
        running = false;
        workers.shutdownNow(); // interrupts blocked queue.take() calls
    }

    public boolean awaitTermination(long timeout, TimeUnit unit) throws InterruptedException {
        return workers.awaitTermination(timeout, unit);
    }
}
```

```java
/** Small demo of how a caller would use the scheduler. */
public final class TaskSchedulerDemo {
    public static void main(String[] args) throws InterruptedException {
        TaskScheduler scheduler = new TaskScheduler(3);

        // One-time task: prints once, after 1 second.
        scheduler.schedule(() -> System.out.println("one-time task ran"), 1000);

        // Recurring task: prints every 2 seconds, starting after 500 ms.
        TaskHandle heartbeat = scheduler.scheduleAtFixedDelay(
                () -> System.out.println("heartbeat at " + System.currentTimeMillis()),
                500, 2000);

        Thread.sleep(7000);
        heartbeat.cancel(); // stop the heartbeat
        System.out.println("heartbeat cancelled");

        Thread.sleep(3000); // prove it does not print again
        scheduler.shutdown();
        scheduler.awaitTermination(5, TimeUnit.SECONDS);
    }
}
```

## How It Works

1. **Submitting a task.** `schedule(...)` or `scheduleAtFixedDelay(...)` builds a
   `ScheduledTask` with a computed `nextRunTimeMillis`, and puts it in the `DelayQueue`. The
   queue is safe to call from many threads at once. It uses a lock and a condition variable
   internally, similar to `PriorityBlockingQueue`.

2. **Ordering.** `ScheduledTask` implements `Delayed`. Its `compareTo` orders tasks by
   `nextRunTimeMillis`, and breaks ties using an insertion sequence number, so two tasks due at
   the same millisecond still get a stable, deterministic order. `DelayQueue` uses this
   ordering internally (backed by a heap, same idea as `PriorityQueue`) to always keep the
   soonest-due task at the head.

3. **Waiting without busy-wait.** Each worker thread calls `queue.take()`. If the head task's
   delay has not expired yet, `take()` puts the thread to sleep on a condition variable for
   exactly that delay (or less, if a new, earlier task is added while it waits). The thread
   wakes up exactly when the task becomes due, or earlier if a new earlier task arrives. This
   is what "no busy-wait" means: the thread is truly parked, not spinning and checking the
   clock in a loop.

4. **Running a task.** Once `take()` returns a due task, the worker checks the `cancelled`
   flag. If cancelled, it is dropped. Otherwise, `runSafely` calls `job.run()` inside a
   try/catch, so an exception in one task cannot kill the worker thread or crash the
   scheduler.

5. **Rescheduling.** After a recurring task finishes running (and is still not cancelled), the
   worker calls `task.scheduleNextRun()`, which sets `nextRunTimeMillis` to "now + interval".
   Because this is **after** the task finished, this gives fixed-delay behavior: slow tasks
   push their own next run later, and cannot pile up many overdue runs.

6. **Cancel.** `TaskHandle.cancel()` just flips an `AtomicBoolean` on the shared task object.
   Because the flag is checked both before running and before rescheduling, a task cannot run
   again after cancel, even if cancel() races with the worker.

7. **Shutdown.** `shutdown()` sets `running = false` and calls `workers.shutdownNow()`, which
   interrupts each worker thread. A worker blocked in `queue.take()` gets an
   `InterruptedException`, sets the interrupt flag back (good practice), and returns, ending
   its loop cleanly.

## How to Extend (Follow-ups)

Interviewers commonly push further with these questions:

- **Fixed-rate instead of fixed-delay.** Compute `nextRunTimeMillis` from the **previous
  scheduled time** plus interval, not from "now". This can cause back-to-back catch-up runs if
  a task is slow. Discuss the trade-off: fixed-delay is safer under load; fixed-rate keeps a
  strict cadence (e.g., "every hour on the hour").

- **Cron-like schedules** (e.g., "every day at 2 AM"). Replace the simple
  `intervalMillis` field with a `RecurrenceStrategy` interface that has one method,
  `nextRunTime(long lastRunTime)`. This is the Strategy pattern. `ScheduledTask` calls it
  instead of doing simple addition.

- **Dynamic thread pool / backpressure.** If the queue grows large because tasks are slower
  than the schedule needs, use a bounded pool sizing strategy, or reject new submissions once
  a queue-size threshold is hit (`RejectedExecutionException`), similar to how
  `ThreadPoolExecutor` handles saturation.

- **Persistence / crash recovery.** Right now everything is in memory. If the process
  restarts, all tasks are lost. A follow-up: persist tasks to a database or a durable queue
  (e.g., a `tasks` table with `next_run_time`), and reload pending tasks on startup. This is
  the real-world version (similar to Quartz Scheduler, or a distributed job queue).

- **Per-task timeout.** Wrap `job.run()` in a `Future` submitted to a second executor, and
  call `future.get(timeout, unit)`, cancelling if it exceeds a limit. Prevents one very slow
  task from tying up a worker thread forever.

- **Priority beyond time.** If two tasks are due, but one is "high priority", extend the
  comparator to also compare a priority field before falling back to time.

- **Observability.** Add hooks like `onTaskStarted`, `onTaskCompleted`, `onTaskFailed`, so
  callers can log or send metrics (task run count, last failure, average duration).

- **Distributed scheduler.** If this runs on multiple app instances, you need to avoid two
  instances running the same task at the same time. That needs a distributed lock (e.g.,
  database row lock, Redis lock, or leader election). Good to mention this is a much larger
  system design problem, not just LLD.

## Complexity & Thread-Safety Notes

- **Time complexity.** `DelayQueue` is backed by a binary heap (`PriorityQueue` internally,
  guarded by a lock). `put()` and `take()` are both **O(log n)**, where `n` is the number of
  pending tasks. This is the same cost as a normal priority queue, plus safe blocking.

- **Space complexity.** **O(n)** for `n` scheduled tasks (including recurring tasks, which are
  always present in the queue except for the brief moment they are being executed).

- **Thread-safety of the queue.** `DelayQueue` is a `java.util.concurrent` class. It is fully
  thread-safe: many threads can call `put()` and `take()` at the same time with no external
  locking needed.

- **Thread-safety of `ScheduledTask`.** The `cancelled` flag is an `AtomicBoolean`, safe to
  read and write from any thread. `nextRunTimeMillis` is `volatile`, so a worker's write
  (in `scheduleNextRun()`) is visible to any thread that later reads it (for example, if the
  queue's internal heap compares it during a concurrent `put()`).

- **No busy-wait, verified.** `DelayQueue.take()` uses `Condition.awaitNanos(delay)` internally
  (via a `ReentrantLock`), which parks the thread with the OS scheduler. CPU usage stays at 0%
  while waiting, unlike a `while (!dueYet) { }` loop or a tight `Thread.sleep(1)` poll loop.

- **Race on cancel-during-run.** If `cancel()` is called while a task's `job.run()` is already
  executing, the run is **not** interrupted (by design — "cancel before next run", per our
  clarifying question). The recheck of `isCancelled()` right after `runSafely()` guarantees the
  task will not be scheduled again.

- **Exception isolation.** Because `runSafely` catches `Throwable`, a broken task cannot crash
  the worker thread. Without this, an uncaught exception would kill that pool thread, and
  `Executors.newFixedThreadPool` does **not** automatically replace a dead thread with the same
  identity, though the pool will still create a new worker on the next submitted task.

## Interview Tips & Common Mistakes

- **Do not build your own busy-wait loop.** A common mistake under pressure is writing
  `while (true) { if (System.currentTimeMillis() >= task.time) run(); }`. This burns 100% CPU
  on every worker thread. Always use a blocking, condition-based wait — either `DelayQueue`, or
  your own lock + `Condition.awaitNanos(...)`.

- **Say why you picked `DelayQueue` over `PriorityBlockingQueue`.** `PriorityBlockingQueue`
  orders elements, but its `take()` does **not** wait for a delay to expire — it returns the
  head immediately, even if it is not due yet. You would have to add your own wait logic on
  top. `DelayQueue` is `PriorityBlockingQueue` semantics plus "wait until due," which is exactly
  the requirement here. Knowing this distinction is a strong signal in an interview.

- **Do not forget the tie-breaker in `compareTo`.** Comparing only by time and returning 0 for
  equal times is legal, but can make ordering behave oddly with the heap. A stable sequence
  number, as used above, avoids surprises and is a small, easy detail to mention.

- **Do not let one task's exception kill a worker.** Forgetting the try/catch around
  `job.run()` is a very common bug. Ask the interviewer directly, "what should we do if a task
  throws?", then implement catch-and-log.

- **Explain fixed-delay vs fixed-rate clearly**, even if you only implement one. This shows you
  understand recurring-task semantics deeply, not just "loop and re-add."

- **Relating this to `ScheduledThreadPoolExecutor`.** In real production Java code, you should
  almost always use `java.util.concurrent.ScheduledThreadPoolExecutor` (or its factory methods
  `Executors.newScheduledThreadPool(n)`), via `schedule(...)`, `scheduleAtFixedRate(...)`, and
  `scheduleWithFixedDelay(...)`. It already solves exactly this problem: a `DelayedWorkQueue`
  (its own internal variant of a `DelayQueue`), a worker pool, cancellation via the returned
  `ScheduledFuture`, and safe exception handling. It is well-tested, handles edge cases (clock
  drift, pool resizing, task removal from the middle of the queue), and is the right choice for
  any real system.
  You would build your own scheduler, like this one, only:
    - in an interview, to prove you understand the internals;
    - when you need custom behavior the built-in class does not support (for example, persistence
      to a database, cron expressions, or distributed coordination);
    - when learning, to understand what `ScheduledThreadPoolExecutor` is doing under the hood.
      Mentioning this trade-off, without being asked, is a strong way to end the discussion — it
      shows maturity: you know the hand-built version and the production version, and you know when
      to use each one.
