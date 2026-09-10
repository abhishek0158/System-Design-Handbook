# Dining Philosophers

## Problem

Five philosophers sit at a round table. Between each pair of philosophers, there is one fork. So there are five forks in total.

Each philosopher does two things, again and again: think, and eat. To eat, a philosopher needs two forks — the one on the left, and the one on the right. A fork can be held by only one philosopher at a time.

Write code that lets all five philosophers eat, without the program getting stuck forever. "Stuck forever" is called deadlock. Deadlock means every philosopher is waiting for a fork, and no one can move.

This problem is a classic way to test if you understand deadlock, and if you can write safe multi-threaded code. Each philosopher runs as a separate thread. Each fork is a shared resource, like a lock.

## Requirements & Clarifying Questions

Before coding, an interviewer expects you to ask a few questions. This shows you think about edge cases, not just the happy path.

Questions to ask:

1. How many philosophers and forks? (Default: 5, but the solution should work for any number N.)
2. Should philosophers eat a fixed number of times, or run forever until stopped?
3. Do we need fairness? Fairness means no philosopher should starve. Starvation means a philosopher never gets to eat, even though the system is not deadlocked.
4. Can we change the problem rules (like using a waiter), or must we keep the exact naive setup?
5. Is the goal to just avoid deadlock, or also to get high throughput (many philosophers eating at the same time)?

Requirements we will assume for this solution:

- N philosophers, N forks, arranged in a circle. Default N = 5.
- Each philosopher repeats: think, pick up two forks, eat, put down two forks.
- The solution must never deadlock.
- The solution should try to avoid starvation, but this is a secondary goal.
- We will use `java.util.concurrent` classes: `ReentrantLock`, `Semaphore`, and `ExecutorService`.

## Design / Approach

### Why the naive solution deadlocks

The naive solution is simple. Each philosopher is assigned a left fork and a right fork. The rule is: pick up the left fork first, then pick up the right fork.

Here is the naive logic in pseudocode:

```
lock(leftFork)
lock(rightFork)
eat()
unlock(rightFork)
unlock(leftFork)
```

If all five philosophers start at the same time, this can happen:

1. Philosopher 0 picks up fork 0 (their left fork).
2. Philosopher 1 picks up fork 1 (their left fork).
3. Philosopher 2 picks up fork 2 (their left fork).
4. Philosopher 3 picks up fork 3 (their left fork).
5. Philosopher 4 picks up fork 4 (their left fork).

Now every philosopher holds one fork. Every philosopher's right fork is the left fork of their neighbor. So every philosopher waits forever for their right fork. No one moves. This is deadlock.

To understand deadlock, it helps to know the four conditions that must all be true for deadlock to happen. These are called the Coffman conditions:

1. **Mutual exclusion** — a resource (fork) can be held by only one thread at a time.
2. **Hold and wait** — a thread holds one resource while waiting for another.
3. **No preemption** — a resource cannot be forcibly taken from a thread. The thread must release it on its own.
4. **Circular wait** — there is a cycle of threads, where each thread waits for a resource held by the next thread in the cycle.

In the naive solution, all four conditions are true. Fork 0 to fork 4 to fork 3 ... back to fork 0 forms a circle. Each philosopher waits for the next one in the circle. This circular wait is the direct cause of the deadlock.

To fix deadlock, we only need to break **one** of these four conditions. We do not need to break all four. Below are three solutions. Each one breaks a different condition.

### Solution 1: Resource ordering (asymmetric pickup)

**Idea:** Number the forks from 0 to N-1. Every philosopher always picks up the lower-numbered fork first, then the higher-numbered fork. This breaks the **circular wait** condition, because now there is a fixed global order for picking up forks. A cycle cannot form when everyone follows the same order.

For the last philosopher (who connects fork N-1 back to fork 0), this rule naturally makes them pick fork 0 first, then fork N-1 — which is the opposite order compared to other philosophers. That is why this is sometimes called "asymmetric" — one philosopher behaves differently, and that breaks the symmetry that caused the deadlock.

### Solution 2: Waiter / arbitrator (Semaphore)

**Idea:** Add a waiter (an arbitrator). The waiter only allows at most N-1 philosophers (4 out of 5) to try picking up forks at the same time. One philosopher must wait outside.

This breaks the **hold and wait** condition in a practical way: with only 4 philosophers allowed to attempt picking up forks, at least one of those 4 is guaranteed to get both their forks. Here is why: 4 philosophers need at most 8 fork-slots, but there are only 5 forks arranged so that neighbors share. With one seat empty, the circular chain is broken, so at least one philosopher has both neighbors' forks free.

We implement the waiter using a `Semaphore` with N-1 permits. A `Semaphore` is a counter that controls how many threads can enter a section of code. A thread calls `acquire()` to take a permit, and `release()` to give it back. If no permits are left, the thread blocks (waits).

### Solution 3: tryLock with timeout and back off

**Idea:** Instead of blocking forever on `lock()`, use `tryLock(timeout)`. This method tries to get the lock, but gives up after a timeout, and returns `false` if it fails. If a philosopher gets the left fork but cannot get the right fork in time, they put down the left fork, wait a bit, and try again from the start.

This breaks the **no preemption** condition. The philosopher voluntarily gives up a resource (the left fork) instead of holding it forever while waiting.

We will code Solution 1 and Solution 2 in full. Solution 3 is shown as a shorter extra, since it is a common follow-up question.

## Java Solution

### Common code: the Fork class

```java
import java.util.concurrent.locks.ReentrantLock;

class Fork {
    private final int id;
    private final ReentrantLock lock = new ReentrantLock();

    Fork(int id) {
        this.id = id;
    }

    int getId() {
        return id;
    }

    ReentrantLock getLock() {
        return lock;
    }
}
```

We use `ReentrantLock` instead of `synchronized`. `ReentrantLock` gives us extra features we need, like `tryLock(timeout)`. A plain `synchronized` block cannot time out.

### Solution 1: Resource ordering

```java
import java.util.List;
import java.util.concurrent.ThreadLocalRandom;
import java.util.concurrent.locks.ReentrantLock;

class OrderedPhilosopher implements Runnable {
    private final int id;
    private final Fork leftFork;
    private final Fork rightFork;
    private final int mealsToEat;

    OrderedPhilosopher(int id, Fork leftFork, Fork rightFork, int mealsToEat) {
        this.id = id;
        // Always lock the fork with the smaller id first.
        // This gives every philosopher the SAME global lock order.
        if (leftFork.getId() < rightFork.getId()) {
            this.leftFork = leftFork;
            this.rightFork = rightFork;
        } else {
            this.leftFork = rightFork;
            this.rightFork = leftFork;
        }
        this.mealsToEat = mealsToEat;
    }

    @Override
    public void run() {
        try {
            for (int i = 0; i < mealsToEat; i++) {
                think();
                ReentrantLock first = leftFork.getLock();
                ReentrantLock second = rightFork.getLock();
                first.lock();
                try {
                    second.lock();
                    try {
                        eat();
                    } finally {
                        second.unlock();
                    }
                } finally {
                    first.unlock();
                }
            }
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }
    }

    private void think() throws InterruptedException {
        System.out.println("Philosopher " + id + " is thinking");
        Thread.sleep(ThreadLocalRandom.current().nextInt(50, 150));
    }

    private void eat() throws InterruptedException {
        System.out.println("Philosopher " + id + " is eating "
                + "(forks " + leftFork.getId() + " and " + rightFork.getId() + ")");
        Thread.sleep(ThreadLocalRandom.current().nextInt(50, 150));
    }
}
```

Note: the field names `leftFork` and `rightFork` inside this class now mean "fork with smaller id" and "fork with bigger id." This is a naming shortcut. The real left/right fork assigned by seating position is passed in from outside, and the constructor re-orders them for locking purposes only.

Driver code:

```java
import java.util.ArrayList;
import java.util.List;

public class OrderedDiningDemo {
    public static void main(String[] args) throws InterruptedException {
        int n = 5;
        List<Fork> forks = new ArrayList<>();
        for (int i = 0; i < n; i++) {
            forks.add(new Fork(i));
        }

        List<Thread> threads = new ArrayList<>();
        for (int i = 0; i < n; i++) {
            Fork left = forks.get(i);
            Fork right = forks.get((i + 1) % n);
            OrderedPhilosopher philosopher = new OrderedPhilosopher(i, left, right, 3);
            Thread t = new Thread(philosopher, "Philosopher-" + i);
            threads.add(t);
            t.start();
        }

        for (Thread t : threads) {
            t.join();
        }
        System.out.println("All philosophers finished eating.");
    }
}
```

### Solution 2: Waiter using Semaphore

```java
import java.util.List;
import java.util.concurrent.Semaphore;
import java.util.concurrent.ThreadLocalRandom;

class WaiterPhilosopher implements Runnable {
    private final int id;
    private final Fork leftFork;
    private final Fork rightFork;
    private final Semaphore waiter;
    private final int mealsToEat;

    WaiterPhilosopher(int id, Fork leftFork, Fork rightFork, Semaphore waiter, int mealsToEat) {
        this.id = id;
        this.leftFork = leftFork;
        this.rightFork = rightFork;
        this.waiter = waiter;
        this.mealsToEat = mealsToEat;
    }

    @Override
    public void run() {
        try {
            for (int i = 0; i < mealsToEat; i++) {
                think();
                // Ask the waiter for permission before touching any fork.
                waiter.acquire();
                try {
                    leftFork.getLock().lock();
                    try {
                        rightFork.getLock().lock();
                        try {
                            eat();
                        } finally {
                            rightFork.getLock().unlock();
                        }
                    } finally {
                        leftFork.getLock().unlock();
                    }
                } finally {
                    waiter.release();
                }
            }
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }
    }

    private void think() throws InterruptedException {
        System.out.println("Philosopher " + id + " is thinking");
        Thread.sleep(ThreadLocalRandom.current().nextInt(50, 150));
    }

    private void eat() throws InterruptedException {
        System.out.println("Philosopher " + id + " is eating");
        Thread.sleep(ThreadLocalRandom.current().nextInt(50, 150));
    }
}
```

Driver code:

```java
import java.util.ArrayList;
import java.util.List;
import java.util.concurrent.Semaphore;

public class WaiterDiningDemo {
    public static void main(String[] args) throws InterruptedException {
        int n = 5;
        List<Fork> forks = new ArrayList<>();
        for (int i = 0; i < n; i++) {
            forks.add(new Fork(i));
        }

        // Only n - 1 philosophers can attempt to pick up forks at once.
        Semaphore waiter = new Semaphore(n - 1);

        List<Thread> threads = new ArrayList<>();
        for (int i = 0; i < n; i++) {
            Fork left = forks.get(i);
            Fork right = forks.get((i + 1) % n);
            WaiterPhilosopher philosopher = new WaiterPhilosopher(i, left, right, waiter, 3);
            Thread t = new Thread(philosopher, "Philosopher-" + i);
            threads.add(t);
            t.start();
        }

        for (Thread t : threads) {
            t.join();
        }
        System.out.println("All philosophers finished eating.");
    }
}
```

### Solution 3 (extra): tryLock with timeout and back off

```java
import java.util.List;
import java.util.concurrent.ThreadLocalRandom;
import java.util.concurrent.TimeUnit;

class TimeoutPhilosopher implements Runnable {
    private final int id;
    private final Fork leftFork;
    private final Fork rightFork;
    private final int mealsToEat;

    TimeoutPhilosopher(int id, Fork leftFork, Fork rightFork, int mealsToEat) {
        this.id = id;
        this.leftFork = leftFork;
        this.rightFork = rightFork;
        this.mealsToEat = mealsToEat;
    }

    @Override
    public void run() {
        try {
            int eaten = 0;
            while (eaten < mealsToEat) {
                think();
                if (tryToEat()) {
                    eaten++;
                }
            }
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }
    }

    private boolean tryToEat() throws InterruptedException {
        boolean gotLeft = leftFork.getLock().tryLock(100, TimeUnit.MILLISECONDS);
        if (!gotLeft) {
            return false;
        }
        try {
            boolean gotRight = rightFork.getLock().tryLock(100, TimeUnit.MILLISECONDS);
            if (!gotRight) {
                // Could not get the right fork in time.
                // Give up the left fork instead of holding it and waiting forever.
                return false;
            }
            try {
                eat();
                return true;
            } finally {
                rightFork.getLock().unlock();
            }
        } finally {
            leftFork.getLock().unlock();
        }
        // If we failed, back off before the next think-and-retry cycle.
        // (Backoff sleep can be added here, e.g. Thread.sleep(random delay).)
    }

    private void think() throws InterruptedException {
        System.out.println("Philosopher " + id + " is thinking");
        Thread.sleep(ThreadLocalRandom.current().nextInt(50, 150));
    }

    private void eat() throws InterruptedException {
        System.out.println("Philosopher " + id + " is eating");
        Thread.sleep(ThreadLocalRandom.current().nextInt(50, 150));
    }
}
```

## How It Works

**Solution 1 (resource ordering):** Each philosopher's constructor compares the two fork ids. It always stores the smaller-id fork as `leftFork` (lock order first) and the bigger-id fork as `rightFork` (lock order second). So philosopher 4, who sits between fork 4 and fork 0, will lock fork 0 first and fork 4 second — the opposite order from philosopher 0, who locks fork 0 first and fork 1 second. Because every thread in the system follows the rule "always lock lower id before higher id," no thread can be stuck holding a high fork while waiting for a low fork that another thread holds while waiting for an even lower fork. The chain of waiting can only go in one direction (low to high), so it cannot loop back and form a circle.

**Solution 2 (waiter):** The `Semaphore waiter` starts with 4 permits (N-1). Each philosopher must call `waiter.acquire()` before locking any fork. Since only 4 permits exist, at most 4 philosophers can be in the "trying to get forks" section at once. The 5th philosopher blocks at `acquire()`, before touching any fork at all. This guarantees at least one philosopher's neighbors are both free from contention, so the deadlock cycle cannot fully close. After eating, the philosopher calls `waiter.release()` in a `finally` block, which lets a waiting philosopher in.

**Solution 3 (tryLock):** `tryLock(timeout, unit)` tries to get the lock, but returns `false` if it cannot get it within the given time, instead of blocking forever. If the philosopher gets the left fork but not the right fork, the code returns `false` from `tryToEat()`, and the outer `finally` block releases the left fork. The philosopher will think again and retry later. This avoids the "hold and wait forever" pattern.

## How to Extend (Follow-ups)

Interviewers often ask these follow-up questions:

- **"How do you prevent starvation?"** Starvation means a philosopher keeps failing to eat, even though the system is not deadlocked. In Solution 1, a philosopher's neighbor might keep grabbing the shared fork first, again and again. To fix this, use a fair lock: `new ReentrantLock(true)`. The `true` flag makes the lock fair — it gives the lock to the thread that has been waiting the longest, instead of any random waiting thread.
- **"What if N philosophers change at runtime?"** Build the forks and philosophers in a loop, driven by a variable `n`, as shown in the driver code above. The resource-ordering rule and the semaphore permit count (`n - 1`) both scale automatically with `n`.
- **"Can you use `synchronized` instead of `ReentrantLock`?"** Yes, for Solution 1. Since `synchronized` blocks always block until they succeed, this works for resource ordering. But `synchronized` cannot do `tryLock` with a timeout, so Solution 3 needs `ReentrantLock`.
- **"How would you test this?"** Run the program many times with a short random `think()` and `eat()` time. Add a watchdog thread: if the program does not finish within some seconds, dump all thread stacks with `Thread.getAllStackTraces()` or use `jstack` to check for deadlock. You can also use `ThreadMXBean.findDeadlockedThreads()` to programmatically detect deadlock.
- **"What about using `Lock.lockInterruptibly()`?"** This lets a blocked thread respond to interruption, so you can cancel a stuck philosopher thread cleanly, instead of it hanging forever.
- **"Can this be modeled with a `CyclicBarrier` or `CountDownLatch`?"** Not directly for solving deadlock, but a `CyclicBarrier` can be useful for testing, to make all philosopher threads start trying to eat at exactly the same time. This is a good way to force the worst-case deadlock scenario in tests, to prove your fix works.

## Complexity & Thread-Safety Notes

**Time:** Each `eat()` call is O(1). Total time depends on the number of meals and the amount of contention (waiting) for forks. There is no algorithmic complexity like O(n log n) here — this is a concurrency problem, not a data-structure problem.

**Space:** O(N) for N forks and N philosopher threads.

**Thread-safety:**

- Each `Fork` has its own `ReentrantLock`. Only one thread can hold a specific fork's lock at a time. This gives mutual exclusion for each fork.
- Locks are always released in a `finally` block. This is critical. If `eat()` throws an exception, the lock must still be released, or the fork stays locked forever, causing a permanent block for the next philosopher who needs that fork.
- Locks are released in the reverse order of acquisition (`second.unlock()` before `first.unlock()`). This is a common convention, though not strictly required for correctness here. It keeps lock hold time symmetric and easy to reason about.
- `Thread.sleep()` inside a lock (as done here in `eat()`, for demo purposes) is normally something to avoid in production code, because it holds the lock longer than needed and hurts throughput. In real code, keep the critical section (the code between lock and unlock) as short as possible. It is shown here only to simulate "eating" taking some time.
- In Solution 2, the `Semaphore` permits and the `ReentrantLock` locks work at two different levels. The semaphore controls how many threads can *attempt* to get forks. The locks control who actually *holds* each fork. Both are needed together.

## Interview Tips & Common Mistakes

- **Say the four deadlock conditions out loud, and name which one your fix breaks.** This is the single most important thing interviewers look for. Just writing working code without explaining the "why" loses marks.
- **Common mistake: forgetting `finally` for `unlock()`.** If an exception happens between `lock()` and `unlock()`, and there is no `finally`, the lock never releases. This is a very common bug in interviews.
- **Common mistake: calling `tryLock()` without a timeout, then busy-looping (retrying instantly in a tight `while` loop).** This wastes CPU. Add a small sleep or backoff between retries.
- **Common mistake: locking forks in an order based on philosopher id, not fork id.** The fix must be based on a fixed, shared numbering of the *forks* (the shared resource), not the philosophers. If you order by philosopher id instead of fork id, the deadlock can still happen, because two neighbors do not agree on which fork is "first."
- **Common mistake: using `synchronized(fork)` with the fork object itself as the lock, but comparing forks by identity in the wrong order**, e.g. using `System.identityHashCode()` instead of a stable, assigned `id` field. Always assign a stable `id` to each fork when you create it, and order by that `id`.
- **Do not confuse deadlock with livelock.** Livelock is when threads keep changing state in response to each other, but no thread makes real progress — for example, two philosophers who both back off at the exact same time, again and again, in a repeating pattern. Adding a small random delay (as shown with `ThreadLocalRandom` in Solution 3) helps avoid livelock, because random timing breaks up repeating patterns.
- **Mention real-world analogies if asked**: this same pattern (resource ordering to avoid deadlock) is used in real database systems, where transactions lock rows. Databases often order lock acquisition by row id, or use deadlock detection with automatic transaction rollback, which is similar in spirit to the `tryLock` and back-off approach.
- **Be ready to explain trade-offs between the two main solutions:** resource ordering (Solution 1) has no extra waiting object, and gives good throughput, but needs every philosopher to correctly follow the global order. The waiter (Solution 2) is easier to reason about and extend to other resources, but the waiter itself can become a bottleneck if there are very many philosophers.
