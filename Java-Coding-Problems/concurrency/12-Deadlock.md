# Deadlock — Create, Detect & Fix

## Problem

Write code that shows a deadlock between two threads. Then show how to detect a deadlock while the program is running. Then fix the code so the deadlock does not happen.

A **deadlock** is a state where two or more threads wait for each other forever. Each thread holds a lock. Each thread also wants a lock held by another thread. No thread can move forward. The program hangs. It does not crash. It just stops making progress.

This problem is common in concurrency interviews for developers with 3–4 years of experience. Interviewers want to see that you understand locks, you can reproduce a deadlock on purpose, you know how to find one in production, and you know at least two ways to prevent one.

## Requirements & Clarifying Questions

Before coding, ask these questions. They show the interviewer you think about the problem first.

1. Are we using `synchronized` blocks, or `java.util.concurrent.locks.ReentrantLock`? (The fix code differs a bit.)
2. Do we need to detect the deadlock automatically, or is a manual thread dump enough?
3. If we use lock ordering as a fix, can we always define a fixed, global order for the locks? (Sometimes locks are created dynamically, so ordering is harder.)
4. Is it acceptable to give up and retry (using `tryLock`) instead of forcing a strict lock order?
5. Do we care about fairness — should a thread that waited longest get the lock first?

For this solution sheet, we assume: two shared resources (bank accounts), two threads that transfer money in opposite directions, and we want both a `synchronized` example and a `ReentrantLock` example.

## Design / Approach

### What causes a deadlock

A deadlock needs four conditions to be true at the same time. These are called the **Coffman conditions**. If we break even one of them, deadlock cannot happen.

1. **Mutual exclusion** — a resource (lock) can be held by only one thread at a time. You cannot share it.
2. **Hold and wait** — a thread holds one lock and waits for another lock, without releasing the first one.
3. **No preemption** — nobody can force a thread to give up a lock. The thread must release it on its own.
4. **Circular wait** — there is a cycle of threads. Thread A waits for a lock held by Thread B. Thread B waits for a lock held by Thread A (or a longer chain that loops back).

In real code, mutual exclusion is usually required (that is the whole point of a lock). So most fixes target **hold and wait** or **circular wait**.

### How to create a deadlock on purpose

Take two objects, `accountA` and `accountB`. Create two threads:

- Thread 1 locks `accountA`, then tries to lock `accountB`.
- Thread 2 locks `accountB`, then tries to lock `accountA`.

If both threads lock their first object at nearly the same time, each one then waits forever for the second object. This is a circular wait. This is the classic deadlock setup.

### How to detect a deadlock

There are two common ways:

1. **Thread dump with `jstack`**. Run `jstack <pid>` on the running Java process. The JVM prints the state of every thread. If there is a deadlock, `jstack` prints a section that says `Found one Java-level deadlock`, and it lists the threads and locks involved.
2. **Programmatic detection with `ThreadMXBean`**. Java gives us `ThreadMXBean.findDeadlockedThreads()`. This method returns the thread IDs that are stuck in a deadlock, or `null` if there is none. We can run this on a background timer, and log or alert when a deadlock is found. This is useful for production monitoring, not just for debugging by hand.

### How to fix a deadlock

The main fixes are:

1. **Global lock ordering.** Always acquire locks in the same order, no matter which thread you are. For example, always lock the account with the smaller ID first. This removes the circular wait condition, because a cycle cannot form if every thread agrees on one order.
2. **Lock timeout with `tryLock`.** Instead of `lock()`, which waits forever, use `tryLock(timeout, unit)` from `ReentrantLock`. If a thread cannot get the second lock within the timeout, it gives up, releases the lock it already holds, waits a little, and retries. This is called **back off**. It breaks the "no preemption" idea in effect — the thread preempts itself.
3. **Reduce lock scope.** Hold the lock for the shortest time possible. Copy the data you need, then release the lock, then do slow work outside the lock. Less time holding a lock means less chance of overlap with another thread.
4. **Use a single lock.** If two resources are almost always updated together, protect both with one lock instead of two. This removes the possibility of a cycle, because there is only one lock to grab.
5. **Lock-free structures.** Use classes like `AtomicInteger`, `AtomicReference`, or `ConcurrentHashMap`, which use compare-and-swap (CAS) instead of locks. No lock means no deadlock. This works well for simple counters and maps, but it is harder to use for multi-step operations like "transfer money between two accounts."

We will show fix #1 (lock ordering) and fix #2 (`tryLock` with back off) in full code, since these are the two the interviewer expects most often.

## Java Solution

### Part 1 — Creating the deadlock

```java
import java.util.concurrent.TimeUnit;

class Account {
    private final String id;
    private double balance;

    Account(String id, double balance) {
        this.id = id;
        this.balance = balance;
    }

    String getId() {
        return id;
    }
}

public class DeadlockDemo {

    // Transfers money from "from" to "to". Locks "from" first, then "to".
    static void transfer(Account from, Account to, double amount) {
        synchronized (from) {
            System.out.println(Thread.currentThread().getName() + " locked " + from.getId());
            sleepQuietly(100); // gives the other thread time to lock its first object too
            synchronized (to) {
                System.out.println(Thread.currentThread().getName() + " locked " + to.getId());
                // move money (details not important here)
            }
        }
    }

    static void sleepQuietly(long ms) {
        try {
            TimeUnit.MILLISECONDS.sleep(ms);
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }
    }

    public static void main(String[] args) {
        Account accountA = new Account("A", 1000);
        Account accountB = new Account("B", 1000);

        // Thread 1: A -> B  (locks A first, then B)
        Thread t1 = new Thread(() -> transfer(accountA, accountB, 100), "Thread-1");

        // Thread 2: B -> A  (locks B first, then A)
        Thread t2 = new Thread(() -> transfer(accountB, accountA, 50), "Thread-2");

        t1.start();
        t2.start();
    }
}
```

Run this program. Both threads print their first lock message, and then the program hangs forever. This is because Thread-1 holds `accountA` and waits for `accountB`. At the same time, Thread-2 holds `accountB` and waits for `accountA`. Neither thread can move. This is a circular wait.

### Part 2 — Detecting the deadlock programmatically

```java
import java.lang.management.ManagementFactory;
import java.lang.management.ThreadInfo;
import java.lang.management.ThreadMXBean;
import java.util.concurrent.Executors;
import java.util.concurrent.ScheduledExecutorService;
import java.util.concurrent.TimeUnit;

public class DeadlockDetector {

    public static void startMonitoring() {
        ThreadMXBean bean = ManagementFactory.getThreadMXBean();
        ScheduledExecutorService scheduler = Executors.newSingleThreadScheduledExecutor(r -> {
            Thread t = new Thread(r, "deadlock-monitor");
            t.setDaemon(true); // does not stop JVM shutdown
            return t;
        });

        scheduler.scheduleAtFixedRate(() -> {
            long[] deadlockedIds = bean.findDeadlockedThreads();
            if (deadlockedIds != null) {
                ThreadInfo[] infos = bean.getThreadInfo(deadlockedIds, true, true);
                System.out.println("DEADLOCK DETECTED among " + infos.length + " threads:");
                for (ThreadInfo info : infos) {
                    System.out.println(" - " + info.getThreadName()
                            + " is waiting for lock " + info.getLockName()
                            + " held by " + info.getLockOwnerName());
                }
                // In real systems: send an alert, log a metric, or dump full stack traces here.
            }
        }, 0, 5, TimeUnit.SECONDS);
    }
}
```

Call `DeadlockDetector.startMonitoring()` once, early in your application. It checks every 5 seconds. It does not fix the deadlock. It only reports it. To find a deadlock manually instead, run `jstack <pid>` from the command line while the process is stuck, and read the "Found one Java-level deadlock" section it prints.

### Part 3 — Fix using global lock ordering

```java
public class DeadlockFreeOrdering {

    static void transfer(Account from, Account to, double amount) {
        // Always lock the account with the smaller id first.
        Account first = from.getId().compareTo(to.getId()) < 0 ? from : to;
        Account second = first == from ? to : from;

        synchronized (first) {
            synchronized (second) {
                // move money between "from" and "to" here
                System.out.println(Thread.currentThread().getName()
                        + " moved " + amount + " from " + from.getId() + " to " + to.getId());
            }
        }
    }
}
```

Now, no matter which direction the transfer goes, both threads try to lock account `"A"` before account `"B"`. There is no way to form a cycle, so there is no deadlock.

### Part 4 — Fix using `tryLock` with timeout and back off

```java
import java.util.concurrent.ThreadLocalRandom;
import java.util.concurrent.TimeUnit;
import java.util.concurrent.locks.ReentrantLock;

class LockableAccount {
    private final String id;
    private final ReentrantLock lock = new ReentrantLock();

    LockableAccount(String id) {
        this.id = id;
    }

    ReentrantLock getLock() {
        return lock;
    }

    String getId() {
        return id;
    }
}

public class DeadlockFreeTryLock {

    static void transfer(LockableAccount from, LockableAccount to, double amount) throws InterruptedException {
        while (true) {
            boolean gotFrom = false;
            boolean gotTo = false;
            try {
                gotFrom = from.getLock().tryLock(200, TimeUnit.MILLISECONDS);
                if (gotFrom) {
                    gotTo = to.getLock().tryLock(200, TimeUnit.MILLISECONDS);
                    if (gotTo) {
                        // both locks acquired: safe to move money
                        System.out.println(Thread.currentThread().getName()
                                + " moved " + amount + " from " + from.getId() + " to " + to.getId());
                        return;
                    }
                }
                // could not get both locks: back off, then retry
            } finally {
                if (gotTo) to.getLock().unlock();
                if (gotFrom) from.getLock().unlock();
            }
            // small random pause before retrying, so both threads do not retry in lockstep
            TimeUnit.MILLISECONDS.sleep(ThreadLocalRandom.current().nextInt(50, 150));
        }
    }
}
```

Here, a thread never waits forever for a lock. If it cannot get both locks quickly, it releases what it has and tries again after a short, random pause. The random pause is important. Without it, both threads could keep failing and retrying at the exact same rhythm, forever. This is a **livelock**, not a deadlock — the threads are not stuck, but they still make no real progress.

## How It Works

- **Deadlock demo:** each thread locks its first account, then sleeps briefly. This sleep is only there to make the deadlock happen reliably in a demo. In real bugs, the timing happens by accident under load. After the sleep, each thread wants the second account, which the other thread already holds. Both wait forever.
- **`ThreadMXBean` detector:** the JVM already tracks which thread owns which lock. `findDeadlockedThreads()` walks this ownership graph and looks for a cycle. If it finds one, it returns the thread IDs in that cycle. We use `getThreadInfo` to turn IDs into readable names and lock names.
- **Ordering fix:** we pick a consistent rule — compare account IDs as strings — to decide which lock to take first. Every thread, no matter its role (sender or receiver), follows the same rule. A cycle needs at least two threads that disagree on order. Since nobody disagrees, no cycle can form.
- **`tryLock` fix:** `ReentrantLock.tryLock(timeout, unit)` returns `false` if it cannot get the lock in time, instead of blocking forever. We check this return value. If we fail partway through (got the first lock but not the second), we release the first lock in the `finally` block before retrying. This guarantees we never hold a lock while waiting indefinitely for another one.

## How to Extend (Follow-ups)

- **N locks, not 2.** Extend the ordering fix to a `transferChain` over many accounts. Sort all accounts by ID first, then lock them in that order.
- **Detect without stopping the app.** Wire the `ThreadMXBean` check into a health-check endpoint or a metrics system, so an operations team gets alerted.
- **Lock-free alternative.** Redesign the transfer with `AtomicLong` balances and a compare-and-swap retry loop, so there is no lock for the balance update.
- **Add fairness.** Create `ReentrantLock` with `new ReentrantLock(true)` (fair mode). Trade-off: fairness lowers starvation risk but also lowers throughput.
- **Simulate starvation.** A low-priority thread never gets a lock because higher-priority threads keep cutting in. This is **starvation** — no cycle, but no progress either. It differs from deadlock; interviewers often ask you to tell them apart.
- **Database deadlocks.** Ask how this maps to database transactions, where the database engine can detect a deadlock and kill one transaction automatically (a "victim").

## Complexity & Thread-Safety Notes

- Detecting a deadlock with `ThreadMXBean.findDeadlockedThreads()` costs very little. It only reads JVM-internal lock ownership data. It is safe to run this check every few seconds in production.
- The lock-ordering fix has no extra runtime cost. It only changes the order of two `lock()` calls. It is the cheapest and most reliable fix when you can define a global order.
- The `tryLock` fix adds retry overhead. Under heavy contention, threads may retry several times before both locks are free. This is a trade-off: we accept some wasted work in exchange for never hanging forever.
- Always release locks in a `finally` block. If code between `lock()` and `unlock()` throws an exception, a missing `finally` leaves the lock held forever, which can cause a hang that looks like a deadlock but is really a **lock leak**.
- `synchronized` blocks release automatically when the block exits, even on exception. This is a safety advantage over manual `ReentrantLock.lock()` / `unlock()`, where you must remember the `finally` block yourself.
- Holding a lock while calling into unknown or external code is risky. That code might, in turn, try to acquire a lock on you, creating a hidden circular wait. Keep lock scope small and predictable.

## Interview Tips & Common Mistakes

- Always state the four Coffman conditions and explain that a fix must break at least one of them. Interviewers use this to check you understand the theory, not just the code.
- A common mistake: writing a "deadlock demo" that does not reliably deadlock, because the timing is too fast. Add a small `sleep` between the two lock calls so the interviewer can actually see it hang.
- Do not confuse deadlock, livelock, and starvation. **Deadlock**: threads are stuck, holding locks, waiting on each other, forever. **Livelock**: threads are not stuck, they keep changing state (like backing off and retrying), but they still make no real progress. **Starvation**: a thread can, in theory, get the lock, but in practice other threads always get there first.
- When asked to "fix" a deadlock, do not just add `sleep` calls to change timing. That only hides the bug; it does not remove the circular wait. The real fixes are ordering, timeout with back off, smaller lock scope, one lock instead of two, or lock-free code.
- Mention that `jstack` is a fast, no-code way to confirm a deadlock in a running process, before you write any detection code. This shows practical, production debugging experience.
- If asked about fairness or starvation trade-offs, be ready to say that fair locks reduce starvation risk but reduce throughput, because of extra bookkeeping and context switches.
- Remember: `wait()`/`notify()` misuse can also cause a similar-looking hang (a thread waits forever because nobody calls `notify()`). This is not a classic lock-ordering deadlock, but interviewers sometimes test if you can tell the two apart.
