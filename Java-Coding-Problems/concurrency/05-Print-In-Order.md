# Print in Order (Odd/Even & N Threads)

## Problem

This problem has three common variants. Interviewers ask one or more of them together, because they test the same skill: making threads run in a fixed order.

1. Two threads print odd and even numbers. The output must be `1, 2, 3, 4, 5, ...` up to N. One thread only prints odd numbers. The other thread only prints even numbers.
2. Three threads print the letters `A`, `B`, `C`. Each thread prints its own letter, one at a time, forever (or for a fixed number of rounds). The output must be `ABCABC...`.
3. N threads print numbers 1 to N. Thread `i` must print the number `i`. The output must be `1, 2, 3, ..., N`, in order, no matter how the operating system schedules the threads.

In all three cases, threads must not use a loop that keeps checking a condition without sleeping (this is called "busy-waiting"). Busy-waiting wastes CPU. The correct solution must make a thread sleep until it is its turn, then wake up only when needed.

## Requirements & Clarifying Questions

Before coding, ask these questions in the interview:

- Is N fixed and known in advance, or can it grow?
- Should the program stop after N numbers, or run forever (for the ABC case)?
- Can I use `java.util.concurrent` classes like `Semaphore`, `ReentrantLock`, and `Condition`? (Most interviewers say yes. A few want raw `wait()`/`notify()` to test base knowledge.)
- Do threads need to be reused, or can I create a new thread each time?
- Is thread-safety of the print statement itself a concern? (Usually no, because only one thread prints at a time by design.)

Clarifying questions show the interviewer that you think about edge cases before writing code. Always ask at least two of the above.

## Design / Approach

All three variants share one pattern: **shared state + a signal**.

- **Shared state** is a variable that says whose turn it is now. For odd/even, this can be a simple flag (`isOdd`) or a shared counter. For ABC, it can be an integer that cycles through 0, 1, 2. For N threads, it is a counter that goes from 1 to N.
- **A signal** is a way for one thread to tell another thread: "your turn is ready, wake up." Java gives us two common signal tools:
    - `wait()` / `notifyAll()` on a shared lock object (the object's own built-in monitor).
    - `Semaphore`, a counter-based lock that threads can acquire and release.

A thread that is not on turn must block (sleep) without using CPU. It must not spin in a `while` loop that keeps checking a flag with no sleep. The fix for that is to call `wait()` inside the loop, or call `acquire()` on a semaphore. Both block the thread safely until another thread signals it.

The general recipe for any print-in-order problem:

1. Pick a shared "turn" variable.
2. Each thread, before printing, waits until the turn variable says it is that thread's turn.
3. After printing, the thread updates the turn variable and signals the next thread.
4. Repeat until done.

Now let's apply this to each variant.

## Java Solution

### Variant 1: Odd/Even Threads (wait/notify approach)

```java
class OddEvenPrinter {
    private final int max;
    private int current = 1;
    private final Object lock = new Object();

    public OddEvenPrinter(int max) {
        this.max = max;
    }

    public void printOdd() throws InterruptedException {
        synchronized (lock) {
            while (current <= max) {
                while (current % 2 == 0) {
                    lock.wait(); // not our turn, sleep
                }
                if (current > max) break;
                System.out.println("Odd: " + current);
                current++;
                lock.notifyAll(); // wake the other thread
            }
        }
    }

    public void printEven() throws InterruptedException {
        synchronized (lock) {
            while (current <= max) {
                while (current % 2 != 0) {
                    lock.wait();
                }
                if (current > max) break;
                System.out.println("Even: " + current);
                current++;
                lock.notifyAll();
            }
        }
    }

    public static void main(String[] args) {
        OddEvenPrinter printer = new OddEvenPrinter(10);
        Thread t1 = new Thread(() -> {
            try { printer.printOdd(); } catch (InterruptedException ignored) {}
        });
        Thread t2 = new Thread(() -> {
            try { printer.printEven(); } catch (InterruptedException ignored) {}
        });
        t1.start();
        t2.start();
    }
}
```

### Variant 1 (alternative): Odd/Even with two Semaphores

```java
class OddEvenSemaphore {
    private final int max;
    private int current = 1;
    private final Semaphore oddTurn = new Semaphore(1);  // odd goes first
    private final Semaphore evenTurn = new Semaphore(0);

    public OddEvenSemaphore(int max) {
        this.max = max;
    }

    public void printOdd() throws InterruptedException {
        while (current <= max) {
            oddTurn.acquire();
            if (current > max) { evenTurn.release(); return; }
            System.out.println("Odd: " + current++);
            evenTurn.release();
        }
    }

    public void printEven() throws InterruptedException {
        while (current <= max) {
            evenTurn.acquire();
            if (current > max) { oddTurn.release(); return; }
            System.out.println("Even: " + current++);
            oddTurn.release();
        }
    }
}
```

### Variant 2: A, B, C in Order (Semaphore approach)

```java
class ABCPrinter {
    private final Semaphore aTurn = new Semaphore(1); // A starts
    private final Semaphore bTurn = new Semaphore(0);
    private final Semaphore cTurn = new Semaphore(0);
    private final int rounds;

    public ABCPrinter(int rounds) {
        this.rounds = rounds;
    }

    public void printA() throws InterruptedException {
        for (int i = 0; i < rounds; i++) {
            aTurn.acquire();
            System.out.print("A");
            bTurn.release();
        }
    }

    public void printB() throws InterruptedException {
        for (int i = 0; i < rounds; i++) {
            bTurn.acquire();
            System.out.print("B");
            cTurn.release();
        }
    }

    public void printC() throws InterruptedException {
        for (int i = 0; i < rounds; i++) {
            cTurn.acquire();
            System.out.println("C");
            aTurn.release();
        }
    }

    public static void main(String[] args) {
        ABCPrinter printer = new ABCPrinter(5);
        new Thread(() -> { try { printer.printA(); } catch (InterruptedException ignored) {} }).start();
        new Thread(() -> { try { printer.printB(); } catch (InterruptedException ignored) {} }).start();
        new Thread(() -> { try { printer.printC(); } catch (InterruptedException ignored) {} }).start();
    }
}
```

### Variant 3: N Threads, Thread `i` Prints `i` (generalized with Lock + Condition)

```java
class SequencePrinter {
    private final Lock lock = new ReentrantLock();
    private final Condition condition = lock.newCondition();
    private int turn = 1;
    private final int n;

    public SequencePrinter(int n) {
        this.n = n;
    }

    public void printNumber(int myNumber) throws InterruptedException {
        lock.lock();
        try {
            while (turn != myNumber) {
                condition.await(); // sleep until it is my turn
            }
            System.out.println(myNumber);
            turn++;
            condition.signalAll(); // wake all waiting threads, they re-check the condition
        } finally {
            lock.unlock();
        }
    }

    public static void main(String[] args) {
        int n = 10;
        SequencePrinter printer = new SequencePrinter(n);
        for (int i = 1; i <= n; i++) {
            final int myNumber = i;
            new Thread(() -> {
                try { printer.printNumber(myNumber); } catch (InterruptedException ignored) {}
            }).start();
        }
    }
}
```

This same `Lock` + `Condition` pattern also solves variant 1 and variant 2. You just change the condition check inside `await()`. For N threads with a semaphore array, you can also give each thread its own `Semaphore(0)` and have thread `i` release the semaphore for thread `i+1` after it prints. Both designs are correct; the `Lock`/`Condition` version scales better because it needs only one lock, not N semaphores.

## How It Works

All three solutions follow the same steps:

1. A thread takes the lock (or tries to acquire a permit).
2. It checks: "Is it my turn?" If not, it calls `wait()` (or `condition.await()`, or blocks on `semaphore.acquire()`). This releases the lock and puts the thread to sleep. The thread uses no CPU while sleeping.
3. When the correct thread finishes its own print, it updates the shared "turn" variable and calls `notifyAll()` (or `condition.signalAll()`, or `semaphore.release()`). This wakes up sleeping threads.
4. Woken-up threads re-check the condition (this is why `wait()` is always inside a `while` loop, not an `if`). Only the thread whose turn it now is will pass the check and continue. The rest go back to sleep.

The `while` loop around `wait()` is important. This guards against **spurious wakeup**, a rare case where a thread can wake up from `wait()` even though no one called `notify()`. If you used `if` instead of `while`, a spurious wakeup could let the wrong thread print out of order.

`notifyAll()` is safer than `notify()`. `notify()` wakes only one waiting thread, chosen at random by the JVM. If it wakes the wrong thread, that thread checks the condition, sees it is not its turn, and goes back to sleep — but now no other thread got woken up, and the program can freeze forever (this is called a "deadlock" from a missed signal). `notifyAll()` wakes every waiting thread. Each one checks the condition. Only the correct one proceeds. The rest go back to sleep. This costs a little more CPU (because many threads wake up briefly) but it is much safer.

## How to Extend (Follow-ups)

Interviewers often push further after the basic solution works. Common follow-ups:

- **"Print 1 to N with M threads, where M < N (threads print numbers in round-robin)."** Change the condition check from `turn == myNumber` to `turn % totalThreads == myThreadIndex`.
- **"Make it work with a `BlockingQueue` instead of locks."** You can use an `ArrayBlockingQueue<Integer>` of size 1 as a "token." A thread takes the token, prints, then puts the token back for the next thread. This avoids `wait()`/`notify()` code directly.
- **"What if a thread can be interrupted or should support cancellation?"** Use `Thread.interrupt()` and check `Thread.currentThread().isInterrupted()` inside the loop, and let `InterruptedException` propagate or handle it by exiting cleanly.
- **"Use `CyclicBarrier` for a case where all N threads must reach a point together before continuing."** This is a related but different tool: it is for threads that must all wait for each other at a checkpoint, not for a strict order.
- **"Avoid a fixed loop count and support infinite ABC printing until a stop signal."** Replace the `for` loop bound with a `while (!stopped)` check based on a shared `volatile boolean stopped` flag.

## Complexity & Thread-Safety Notes

- **Time complexity:** All solutions are O(N) for printing N items — each item is printed exactly once, and each hand-off between threads is O(1).
- **Space complexity:** O(1) extra space beyond the threads themselves. No extra data structure grows with N (except thread objects, which is O(N) if you create N threads for variant 3).
- **Thread-safety:** Correctness depends on the "turn" variable being read and written only inside a lock or through a semaphore. Do not read or write it outside synchronized code — this is called a "race condition" and can silently break the order.
- **No busy-waiting:** All designs shown here use blocking calls (`wait()`, `condition.await()`, `semaphore.acquire()`). None use a `while(true) { if (condition) break; }` loop without a blocking call. That kind of loop would spin the CPU at 100% and is always a wrong answer in interviews.
- **Livelock risk:** Using `if` instead of `while` around `wait()` can cause missed wakeups if paired with `notify()` (not `notifyAll()`). Always pair `notify()` with an `if` only when you are 100% sure only one specific thread can be waiting; otherwise use `notifyAll()` with `while`.

## Interview Tips & Common Mistakes

- **Compare `wait()`/`notify()` vs `Semaphore`:** `wait()`/`notify()` is built into every Java object and needs no extra import, but the code is more verbose and easy to get wrong (missing `while`, forgetting `notifyAll`). `Semaphore` from `java.util.concurrent` is cleaner to read, because "acquire" and "release" map directly to "wait for turn" and "signal next turn." For interviews at 3-4 years experience, **Semaphore is usually the cleaner and preferred answer**, especially for the ABC problem, because the code reads almost like plain English. Use `Lock` + `Condition` when you need more than a plain flag, like the N-thread generalized version, because `Condition` supports naming multiple wait conditions on one lock (`newCondition()` can be called multiple times).
- **Common mistake 1:** Using `if` instead of `while` around `wait()`. This breaks on spurious wakeup.
- **Common mistake 2:** Forgetting to call `notifyAll()` after updating shared state — the other thread waits forever.
- **Common mistake 3:** Reading or writing the "turn" variable outside a `synchronized` block or outside the lock. This is a race condition, even if it looks harmless.
- **Common mistake 4:** Creating a new lock object per thread instead of sharing one lock — then `wait()`/`notify()` do nothing, because each thread is talking to a different monitor.
- **Common mistake 5:** Busy-waiting with `while (turn != myTurn) { }` and no `sleep()` or `wait()` inside. This burns CPU and is an instant red flag in an interview.
- **Talking point:** Mention that `Semaphore` also works well when you don't need mutual exclusion, just signaling — a semaphore with 0 permits is a clean way to say "block until someone tells you to go," without needing a lock object at all.
- **Talking point:** If asked "how would you test this?", mention running the program multiple times and checking output order, or adding assertions inside each print step that check the "turn" value matches the expected thread ID.
