# Job Queue with Workers

## Problem

Design and build a **job queue with a worker pool**. Producers submit jobs. A fixed pool of worker threads pulls jobs from a shared queue and runs them. This is the classic **producer–consumer pattern**, applied to a background job system (similar to how a real task processor, like Sidekiq or Celery, works).

The system must support:

- Submitting a job from any thread (a "producer").
- Multiple worker threads pulling jobs from one shared queue (the "consumers").
- Job **priority**: high-priority jobs should run before low-priority ones.
- **Retry** of a failed job, up to a maximum number of attempts.
- A **dead-letter list**: jobs that fail even after all retries go here, instead of being lost.
- **Graceful shutdown**: stop accepting new jobs, let in-flight and already-queued jobs finish, then stop all workers cleanly.

## Requirements & Clarifying Questions

Before coding, an interviewer expects you to clarify scope. Good questions to ask:

1. **Is this single-JVM or distributed?** We assume single JVM, in-memory. (Distributed queues are a follow-up.)
2. **What decides priority?** We assume an integer priority field, set by the producer when it creates the job.
3. **What counts as "failure"?** Any exception thrown by the job's `run()` method.
4. **Should retries happen immediately or after a delay?** We keep it simple: retry by putting the job back on the queue right away. A delayed retry (backoff) is mentioned as an extension.
5. **What happens to jobs still in the queue during shutdown?** We drain them — workers keep processing until the queue is empty, then stop. We do **not** accept new submissions once shutdown starts.
6. **Do we need job results returned to the caller?** Not required here. We assume "fire and forget" with logging. (`CompletableFuture`-based result passing is a natural follow-up, but out of scope to keep the core design focused.)
7. **Thread safety**: multiple producer threads and multiple worker threads will touch the queue and the dead-letter list at the same time. Every shared structure must be safe for concurrent use.

## Design / Approach

The design has four pieces:

1. **`Job`** — represents one unit of work. It carries an id, a payload (the data the job needs), a priority, a max-attempts limit, and a running count of attempts made so far. It implements `Comparable` so a priority queue can order jobs.
2. **`JobQueue`** — a thin wrapper around `PriorityBlockingQueue<Job>`. `PriorityBlockingQueue` is a blocking queue that also keeps its elements ordered by priority. It is safe for many threads to call `put` and `take` on it at the same time. The wrapper also owns the **dead-letter list** and an `acceptingJobs` flag used during shutdown.
3. **`Worker`** — a `Runnable` that loops: take a job, run it, catch failures, retry or dead-letter. Each worker runs on its own thread, managed by an `ExecutorService`.
4. **`JobQueueManager`** — the entry point. It starts the workers, exposes `submit(Job)`, exposes the dead-letter list for inspection, and drives graceful shutdown.

### Why `PriorityBlockingQueue`?

A normal `BlockingQueue` (like `LinkedBlockingQueue`) is First-In-First-Out. We need "highest priority first" ordering. `PriorityBlockingQueue` keeps elements sorted using `compareTo` (or a `Comparator`), and it is unbounded and thread-safe, with blocking `take()` for consumers. That is exactly what workers need: block until a job is available, then get the most important one.

One catch: `PriorityBlockingQueue` does not guarantee FIFO order for elements with **equal** priority. We fix this by adding a **sequence number** to each job (assigned at creation time) and using it as a tie-breaker in `compareTo`. This gives us stable ordering: same priority → earlier job runs first.

### Why retries go back into the same queue

When a job fails and has attempts left, we simply call `queue.put(job)` again. It re-enters the priority queue like any other job. This is simple and correct, but it means a failing job can be retried "immediately" (as soon as a worker is free), with no delay between attempts. For real systems, you want a backoff delay — this is called out in the follow-ups section using `DelayQueue`.

### Why a `CopyOnWriteArrayList` for dead letters

The dead-letter list is written rarely (only on final failure) but may be read often (e.g., a monitoring endpoint listing failed jobs). `CopyOnWriteArrayList` is a good fit: reads never block, and writes are safe, because they happen much less often than reads.

### Graceful shutdown strategy

Graceful shutdown must do three things, in order:

1. Stop accepting new submissions (`acceptingJobs = false`). New callers to `submit()` get a clear rejection instead of silently queuing forever.
2. Let workers keep pulling and running jobs **until the queue is empty**. This "drains" in-flight and already-queued work — we do not kill jobs mid-run.
3. Once the queue is empty and each worker sees no more work, each worker exits its loop, and the manager waits for all workers to finish using a `CountDownLatch`.

We use a short poll timeout (e.g. 500 ms) instead of the blocking `take()` during shutdown checks. This lets a worker periodically check "is shutdown requested AND is the queue empty?" without spinning the CPU or blocking forever on an empty queue.

## Java Solution

```java
import java.util.List;
import java.util.Collections;
import java.util.UUID;
import java.util.concurrent.*;
import java.util.concurrent.atomic.AtomicInteger;
import java.util.concurrent.atomic.AtomicLong;

// ---------- Job ----------

public abstract class Job implements Comparable<Job> {

    // Gives every job a unique, ever-increasing number, used to break priority ties.
    private static final AtomicLong SEQUENCE_GENERATOR = new AtomicLong(0);

    private final String id;
    private final Object payload;
    private final int priority;      // higher number = more important
    private final int maxAttempts;
    private final long sequenceNumber;
    private final AtomicInteger attempts = new AtomicInteger(0);

    protected Job(Object payload, int priority, int maxAttempts) {
        this.id = UUID.randomUUID().toString();
        this.payload = payload;
        this.priority = priority;
        this.maxAttempts = maxAttempts;
        this.sequenceNumber = SEQUENCE_GENERATOR.getAndIncrement();
    }

    /** The actual work. Throw an exception to signal failure. */
    public abstract void run() throws Exception;

    public String getId() { return id; }
    public Object getPayload() { return payload; }
    public int getPriority() { return priority; }
    public int getMaxAttempts() { return maxAttempts; }
    public int getAttempts() { return attempts.get(); }

    /** Called by a worker right before running the job. Returns the new attempt count. */
    public int recordAttempt() { return attempts.incrementAndGet(); }

    @Override
    public int compareTo(Job other) {
        int byPriority = Integer.compare(other.priority, this.priority); // descending
        if (byPriority != 0) return byPriority;
        return Long.compare(this.sequenceNumber, other.sequenceNumber);  // ascending (FIFO)
    }
}

// ---------- JobQueue ----------

public class JobQueue {

    private final PriorityBlockingQueue<Job> queue = new PriorityBlockingQueue<>();
    private final List<Job> deadLetterJobs = new CopyOnWriteArrayList<>();
    private volatile boolean acceptingJobs = true;

    /** Called by producers. */
    public void submit(Job job) {
        if (!acceptingJobs) {
            throw new RejectedExecutionException(
                "Queue is shutting down; rejected job " + job.getId());
        }
        queue.put(job);
    }

    /** Called by workers to retry a failed job. Bypasses the "accepting" check
        so in-flight retries can finish during shutdown. */
    void requeue(Job job) {
        queue.put(job);
    }

    /** Blocks up to the given timeout for a job. Returns null on timeout. */
    Job poll(long timeout, TimeUnit unit) throws InterruptedException {
        return queue.poll(timeout, unit);
    }

    boolean isEmpty() {
        return queue.isEmpty();
    }

    void addToDeadLetter(Job job) {
        deadLetterJobs.add(job);
    }

    public List<Job> getDeadLetterJobs() {
        return Collections.unmodifiableList(deadLetterJobs);
    }

    void stopAccepting() {
        acceptingJobs = false;
    }

    public int pendingCount() {
        return queue.size();
    }
}

// ---------- Worker ----------

public class Worker implements Runnable {

    private final String name;
    private final JobQueue jobQueue;
    private final CountDownLatch doneLatch;
    private volatile boolean stopRequested = false;

    public Worker(String name, JobQueue jobQueue, CountDownLatch doneLatch) {
        this.name = name;
        this.jobQueue = jobQueue;
        this.doneLatch = doneLatch;
    }

    /** Asks this worker to stop once the queue is drained. Non-blocking. */
    public void requestStop() {
        stopRequested = true;
    }

    @Override
    public void run() {
        try {
            while (true) {
                Job job;
                try {
                    job = jobQueue.poll(500, TimeUnit.MILLISECONDS);
                } catch (InterruptedException e) {
                    Thread.currentThread().interrupt();
                    return;
                }

                if (job != null) {
                    process(job);
                    continue;
                }

                // No job was available in this poll window.
                if (stopRequested && jobQueue.isEmpty()) {
                    return; // nothing left to do; exit cleanly
                }
            }
        } finally {
            doneLatch.countDown();
        }
    }

    private void process(Job job) {
        int attemptNumber = job.recordAttempt();
        try {
            System.out.printf("[%s] running job %s (attempt %d/%d)%n",
                name, job.getId(), attemptNumber, job.getMaxAttempts());
            job.run();
            System.out.printf("[%s] job %s succeeded%n", name, job.getId());
        } catch (Exception e) {
            System.out.printf("[%s] job %s failed on attempt %d: %s%n",
                name, job.getId(), attemptNumber, e.getMessage());
            if (attemptNumber < job.getMaxAttempts()) {
                jobQueue.requeue(job);
            } else {
                System.out.printf("[%s] job %s moved to dead-letter list%n", name, job.getId());
                jobQueue.addToDeadLetter(job);
            }
        }
    }
}

// ---------- JobQueueManager ----------

public class JobQueueManager {

    private final JobQueue jobQueue = new JobQueue();
    private final List<Worker> workers = new CopyOnWriteArrayList<>();
    private final ExecutorService executor;
    private final CountDownLatch allWorkersDone;

    public JobQueueManager(int workerCount) {
        this.executor = Executors.newFixedThreadPool(workerCount);
        this.allWorkersDone = new CountDownLatch(workerCount);
        for (int i = 0; i < workerCount; i++) {
            Worker worker = new Worker("worker-" + i, jobQueue, allWorkersDone);
            workers.add(worker);
            executor.submit(worker);
        }
    }

    public void submit(Job job) {
        jobQueue.submit(job);
    }

    public List<Job> getDeadLetterJobs() {
        return jobQueue.getDeadLetterJobs();
    }

    public int pendingCount() {
        return jobQueue.pendingCount();
    }

    /** Stops accepting new jobs, drains what's left, then stops all workers. */
    public void shutdown(long awaitTimeout, TimeUnit unit) throws InterruptedException {
        jobQueue.stopAccepting();
        for (Worker worker : workers) {
            worker.requestStop();
        }
        boolean finishedInTime = allWorkersDone.await(awaitTimeout, unit);
        if (!finishedInTime) {
            System.out.println("Shutdown timeout reached; forcing executor stop.");
        }
        executor.shutdownNow();
    }
}
```

### Example usage

```java
JobQueueManager manager = new JobQueueManager(3); // 3 workers

manager.submit(new Job("send-email-1", /* priority */ 5, /* maxAttempts */ 3) {
    public void run() {
        System.out.println("Sending email...");
    }
});

manager.submit(new Job("generate-report", /* priority */ 1, /* maxAttempts */ 2) {
    public void run() throws Exception {
        throw new RuntimeException("Report service down");
    }
});

// later, at application shutdown:
manager.shutdown(10, TimeUnit.SECONDS);
System.out.println("Dead letters: " + manager.getDeadLetterJobs().size());
```

## How It Works

**Submitting a job.** A producer calls `manager.submit(job)`. This calls `jobQueue.submit(job)`, which checks `acceptingJobs` and then calls `queue.put(job)`. `PriorityBlockingQueue.put` never actually blocks (the queue is unbounded), but it is still the correct method to call because it is the queue's standard "add" operation and matches the blocking `take`/`poll` on the consumer side.

**Ordering.** Every job carries a `priority` and a `sequenceNumber`. `compareTo` sorts by priority descending, then by sequence ascending. So the head of the queue is always the highest-priority, oldest-submitted job.

**Workers pulling jobs.** Each `Worker` runs a loop on its own thread (managed by a fixed-size `ExecutorService`). It calls `jobQueue.poll(500ms)`. If a job comes back, it processes it and loops again immediately. If nothing comes back within 500 ms, the worker checks: "has someone asked me to stop, **and** is the queue now empty?" If both are true, it exits. Otherwise, it loops and polls again. This poll-with-timeout pattern is what makes shutdown detection possible without busy-waiting.

**Processing and retries.** `process(job)` increments the job's attempt counter (using `AtomicInteger`, so this is safe even though, in this design, only one worker ever touches a given job's counter at a time — but atomicity is good practice regardless). It then calls `job.run()`. If that throws, the worker checks `attemptNumber < maxAttempts`. If there are attempts left, it puts the job back on the queue (`requeue`, which skips the `acceptingJobs` check so in-flight retries can still complete during shutdown). If attempts are exhausted, the job goes to the dead-letter list instead.

**Dead-letter list.** This is a `CopyOnWriteArrayList<Job>`, safe for concurrent reads (e.g., a health-check endpoint) and writes (workers appending failed jobs) at the same time.

**Graceful shutdown.** `manager.shutdown(...)` does three things: (1) flips `acceptingJobs` to `false`, so any further `submit()` call throws immediately instead of silently queueing; (2) tells every worker to stop via `requestStop()` (a `volatile boolean`, so the change is visible across threads without needing a lock); (3) waits on a `CountDownLatch` until every worker has actually exited its loop, meaning the queue was fully drained. Only after that does it call `executor.shutdownNow()` as a final cleanup, which by that point should have nothing left to interrupt.

## How to Extend (Follow-ups)

**Delayed jobs (run-at or backoff).** Replace immediate `requeue` with a delayed retry: wrap `Job` so it also implements `Delayed`, and use a `DelayQueue<Job>` instead of (or in front of) the `PriorityBlockingQueue`. `Delayed.getDelay()` would return the time left until the job is eligible to run — for example, `attemptNumber * attemptNumber * 1000 ms` for exponential backoff. A `DelayQueue.take()` blocks until the earliest-delay item's delay has expired, then returns it. You would need a small feeder thread or logic that moves jobs from the delay queue into the priority queue once their delay expires, since `DelayQueue` does not have a separate priority field of its own — only delay-based ordering.

**Rate limiting workers.** If jobs call an external API with a rate limit, add a `Semaphore` (or a token-bucket rate limiter) that a worker must acquire a permit from before calling `job.run()`, and release after. This caps how many jobs run per second across the whole pool, independent of how many worker threads exist. Alternatively, put a rate limiter check inside `process()` before calling `run()`, and if no permit is available, sleep briefly or push the job back for a later retry.

**Persistence (surviving a restart).** Right now, all jobs live only in memory — a process crash loses the queue and the dead-letter list. To persist, write each job to a database or an append-only log (a **write-ahead log**, meaning changes are written to a durable log before being applied) when it is submitted, mark it as "done" when it succeeds, and mark it as "dead-lettered" when it exhausts retries. On startup, reload any job still marked "pending" or "in-progress" back into the in-memory queue. This turns the in-memory queue into a cache in front of durable storage, and is exactly what tools like Sidekiq (backed by Redis) or a database-backed job table do.

**Scaling to many machines.** A single-JVM `PriorityBlockingQueue` cannot be shared across machines. For real distributed scale, replace the whole `JobQueue` with a managed message queue or log: **Amazon SQS** (a managed queue service; supports visibility timeouts, which work like a built-in retry mechanism, and dead-letter queues natively) or **Apache Kafka** (a distributed log; good for very high throughput and when multiple independent consumer groups need to process the same jobs). Workers then become consumer processes across many machines, pulling from the shared external queue instead of an in-memory one. Priority in Kafka is usually handled with separate topics per priority level, since Kafka itself has no per-message priority.

## Complexity & Thread-Safety Notes

- **`submit` / `poll`**: O(log n) each, where n is the number of jobs currently in the queue. This is the cost of maintaining a heap-based priority queue.
- **`isEmpty` / `pendingCount`**: O(1).
- **Dead-letter add**: O(k) where k is the current dead-letter list size, because `CopyOnWriteArrayList` copies its backing array on every write. This is fine because dead-letter writes are rare (only on final failure) and reads are frequent and cheap (O(1) to get a reference, O(k) to iterate — no locking needed).
- **Thread safety**:
    - `PriorityBlockingQueue` internally uses a lock to protect the heap; all `put`/`take`/`poll` calls are safe for concurrent producers and consumers.
    - `AtomicInteger` for `attempts` avoids race conditions if a job's attempt count is ever read or written from more than one place.
    - `volatile boolean acceptingJobs` and `volatile boolean stopRequested` are simple flags, not compound state, so `volatile` alone is enough to make writes visible to other threads immediately — no separate lock is needed for a plain read/write flag.
    - `CopyOnWriteArrayList` gives the dead-letter list safe concurrent access without explicit locking, at the cost of a full array copy per write — acceptable given how rarely writes happen.
    - `CountDownLatch` gives a clean one-way signal ("all workers are done") without polling or spin-waiting from the shutdown caller's side.
- **No deadlocks possible** in this design because there is no scenario where a thread holds one lock while waiting for another. Each worker only ever touches the queue and, on failure, the dead-letter list — never both while holding a lock across the call, since both structures manage their own internal locking.

## Interview Tips & Common Mistakes

- **Do not use `synchronized` blocks around a hand-rolled list as the queue.** Interviewers want to see that you know `java.util.concurrent` already gives you a correct, well-tested building block (`PriorityBlockingQueue`). Reinventing it with your own locks is more code and more risk of bugs.
- **Remember the FIFO tie-breaker.** Many candidates use only `priority` in `compareTo` and forget that `PriorityBlockingQueue` gives no ordering guarantee between equal-priority elements. Adding a sequence number is a small but important detail interviewers look for.
- **Do not call `Thread.stop()` or forcibly kill worker threads for shutdown.** That can leave a job half-done and shared state inconsistent. Always prefer a cooperative flag (`volatile boolean`) checked in the loop, plus a bounded wait like `awaitTermination` or a `CountDownLatch`.
- **Watch for the "busy loop" mistake.** If you use `queue.poll(0, ...)` or check `isEmpty()` in a tight `while` loop with no wait, you burn CPU for nothing. Always poll with a small timeout (hundreds of milliseconds) so the thread actually sleeps between checks.
- **Be clear about what "graceful" means.** Some candidates confuse "graceful shutdown" with "stop everything immediately." Explain clearly: stop **accepting new work**, but **finish what's already queued or running**.
- **Explain why retries go back into the same queue** rather than a separate one — it reuses the same priority ordering and worker pool, so a retried high-priority job still jumps ahead of low-priority new jobs.
- **A common follow-up trap**: candidates say "just add a `Thread.sleep()` before retrying" inside the worker itself. This is wrong because it blocks that worker thread from picking up other jobs while it sleeps. The correct answer is a delay queue or a scheduled re-submission, not a blocking sleep inside the worker loop.
- **Mention the max-attempts edge case**: what happens if `maxAttempts` is 1? The job should go straight to the dead-letter list on the first failure, with no retry. Trace through the code to confirm `attemptNumber < maxAttempts` handles this correctly (1 < 1 is false, so it dead-letters immediately) — interviewers like seeing you verify boundary conditions out loud.
