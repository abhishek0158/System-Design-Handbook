# ReadWriteLock Cache

## Problem

Build a thread-safe, in-memory cache with three operations: `get(key)`, `put(key, value)`, and `evict(key)`. Many threads call these methods at the same time.

Most caches are read-heavy. Many threads call `get` at once. Fewer threads call `put` or `evict`. A plain lock (like `synchronized` or `ReentrantLock`) blocks every thread, even two threads that only want to read. This wastes concurrency.

Use `java.util.concurrent.locks.ReentrantReadWriteLock` instead. It has two locks: a **read lock** and a **write lock**. Many threads can hold the read lock at the same time. Only one thread can hold the write lock, and no other thread (reader or writer) can enter while the write lock is held.

This is a common interview question for 3-4 years experience. It tests whether you understand lock types beyond `synchronized`, and whether you know when a read-write lock actually helps.

## Requirements & Clarifying Questions

Ask these questions before coding. They show you think about trade-offs, not just syntax.

1. **Is the workload read-heavy?** A read-write lock helps most when reads outnumber writes by a large margin. If writes are frequent, the benefit shrinks or disappears.
2. **How long does each operation take?** If `get` and `put` are very short (just a map lookup), lock overhead can be bigger than the work itself. In that case, a simple lock or `ConcurrentHashMap` may be faster.
3. **Do we need strong consistency, or is a slightly stale read OK?** This decides if `StampedLock` optimistic reads are safe to use. Optimistic reads can return a value that changes mid-read, so the code must validate before trusting it.
4. **Is the lock reentrant?** Can the same thread call `get` again while already holding a lock (nested calls)? `ReentrantReadWriteLock` supports reentrancy. `StampedLock` does not.
5. **Do we need fairness?** Should waiting writers get priority over new readers, to avoid writer starvation? `ReentrantReadWriteLock` supports a fair mode.
6. **What about eviction policy, TTL, size limits?** Out of scope here. This problem focuses on the locking strategy, not cache policy. (See the TTL cache problem for that.)
7. **Single JVM or distributed?** This is a single-JVM, in-memory cache. A distributed cache (like Redis) needs different tools (network calls, replication), not just a Java lock.

## Design / Approach

### The core idea: read-write lock

A read-write lock splits one lock into two:

- **Read lock (shared lock)**: Many threads can hold it at the same time, as long as no thread holds the write lock. Use it for `get`.
- **Write lock (exclusive lock)**: Only one thread can hold it. No reader and no other writer can enter while it is held. Use it for `put` and `evict`.

The rule is simple: readers do not block other readers. Writers block everyone.

### Why this helps for read-heavy caches

Say 100 threads call `get` and 1 thread calls `put`, over and over. With a plain `synchronized` block, every `get` call blocks every other `get` call, even though reads do not conflict with each other. Throughput is low.

With `ReentrantReadWriteLock`, all 100 reader threads can run `get` at the same time. Only when the writer calls `put` does everyone else have to wait. Throughput goes up a lot, because the common case (read) is now parallel.

### When a read-write lock does NOT help

1. **Write-heavy workloads.** If writes are frequent, writers keep blocking readers anyway. The extra bookkeeping in `ReentrantReadWriteLock` (tracking read count, write count, waiting threads) adds overhead compared to a plain lock, with little benefit.
2. **Very short critical sections.** `ReentrantReadWriteLock` is heavier than `synchronized` or `ReentrantLock`, because it must track multiple reader threads internally. If the protected code is just one map lookup (a few nanoseconds), the lock's own overhead can be bigger than the work it protects. In that case, `ConcurrentHashMap` (with no explicit lock in your code) usually wins.
3. **High reader-writer contention with long write pauses.** If writers hold the lock for a long time, readers queue up and latency spikes, even though throughput math looks fine on paper.

Rule of thumb: measure first. Read-write locks help when reads are common, writes are rare, and each critical section does enough work to make the lock overhead worth it.

### Lock downgrading (write to read)

A thread holding the write lock can acquire the read lock before releasing the write lock. This is called **downgrading**. It lets a thread finish a write, then continue reading its own fresh data, without letting another writer sneak in between the write and the read.

Steps for downgrading:

1. Acquire the write lock.
2. Update the data.
3. Acquire the read lock (while still holding the write lock).
4. Release the write lock. (You now hold only the read lock.)
5. Read the data.
6. Release the read lock.

**Upgrading (read to write) is not allowed.** If a thread holds the read lock and tries to acquire the write lock without releasing the read lock first, it will deadlock. Why? The write lock needs no other thread to hold any lock, including the current thread's own read lock. If two threads both hold the read lock and both try to upgrade, neither can proceed, and both wait forever. `ReentrantReadWriteLock` javadoc calls this out directly: upgrading is not supported. To "upgrade," release the read lock fully, then acquire the write lock (accepting that another thread might change the data in between).

### StampedLock and optimistic reads

`java.util.concurrent.locks.StampedLock` (added in Java 8) is a faster alternative for read-heavy workloads. It has three modes:

- **Write lock**: exclusive, like before.
- **Read lock (pessimistic)**: shared, like `ReentrantReadWriteLock`'s read lock, but `StampedLock` is not reentrant, so be careful with nested calls.
- **Optimistic read**: does not block at all. It just checks a "stamp" (a version number) before and after reading. If a write happened in between, the stamp changed, and the code knows the read might be wrong. It then retries with a real read lock.

Optimistic read has no memory barrier cost for readers when there is no writer active. This makes it very fast for read-heavy, low-write workloads. The cost is code complexity: you must validate the stamp and be ready to fall back to a locked read.

`StampedLock` does not support reentrancy and does not track thread ownership like `ReentrantReadWriteLock`. Calling `unlock` from the wrong thread, or locking twice on the same thread, causes bugs that are hard to find. Use it carefully, and only when profiling shows the extra speed is needed.

## Java Solution

```java
import java.util.HashMap;
import java.util.Map;
import java.util.concurrent.locks.ReentrantReadWriteLock;
import java.util.concurrent.locks.StampedLock;

/**
 * Thread-safe cache using ReentrantReadWriteLock.
 * Many threads can read at the same time.
 * A write (put/evict) needs exclusive access.
 */
public class ReadWriteLockCache<K, V> {

    private final Map<K, V> store = new HashMap<>();
    private final ReentrantReadWriteLock rwLock = new ReentrantReadWriteLock();
    private final ReentrantReadWriteLock.ReadLock readLock = rwLock.readLock();
    private final ReentrantReadWriteLock.WriteLock writeLock = rwLock.writeLock();

    public V get(K key) {
        readLock.lock();
        try {
            return store.get(key);
        } finally {
            readLock.unlock();
        }
    }

    public void put(K key, V value) {
        writeLock.lock();
        try {
            store.put(key, value);
        } finally {
            writeLock.unlock();
        }
    }

    public void evict(K key) {
        writeLock.lock();
        try {
            store.remove(key);
        } finally {
            writeLock.unlock();
        }
    }

    /**
     * Shows lock downgrading: write, then downgrade to read,
     * then return a value we know is fresh and stable.
     */
    public V putAndReadBack(K key, V value) {
        writeLock.lock();
        V result;
        try {
            store.put(key, value);
            readLock.lock(); // acquire read lock while still holding write lock
        } finally {
            writeLock.unlock(); // release write lock; we still hold read lock
        }
        try {
            result = store.get(key); // safe: no other writer can run now
        } finally {
            readLock.unlock();
        }
        return result;
    }
}
```

Here is the same cache using `StampedLock` with an optimistic read, as a faster alternative:

```java
import java.util.HashMap;
import java.util.Map;
import java.util.concurrent.locks.StampedLock;

public class StampedLockCache<K, V> {

    private final Map<K, V> store = new HashMap<>();
    private final StampedLock lock = new StampedLock();

    public V get(K key) {
        long stamp = lock.tryOptimisticRead(); // non-blocking, returns a version stamp
        V value = store.get(key);              // read without any lock held

        if (!lock.validate(stamp)) {
            // A write happened during our read. The value above may be wrong.
            // Fall back to a real read lock.
            stamp = lock.readLock();
            try {
                value = store.get(key);
            } finally {
                lock.unlockRead(stamp);
            }
        }
        return value;
    }

    public void put(K key, V value) {
        long stamp = lock.writeLock();
        try {
            store.put(key, value);
        } finally {
            lock.unlockWrite(stamp);
        }
    }

    public void evict(K key) {
        long stamp = lock.writeLock();
        try {
            store.remove(key);
        } finally {
            lock.unlockWrite(stamp);
        }
    }
}
```

## How It Works

**`ReentrantReadWriteLock` version:**

- `get` takes the read lock. If no writer holds the write lock, the read lock is granted right away, even if other readers already hold it.
- `put` and `evict` take the write lock. This call blocks until all current readers release the read lock, and no other writer holds the write lock. While the write lock is held, new readers must wait too.
- The lock is **reentrant**: if a thread already holds the write lock, it can acquire the write lock again (or the read lock, for downgrading) without blocking on itself.
- `putAndReadBack` shows downgrading: the thread holds the write lock, grabs the read lock too, then releases the write lock. Now it only holds the read lock, and can safely read its own fresh write without another writer jumping in.

**`StampedLock` version:**

- `tryOptimisticRead()` returns a stamp (a `long` number) right away. It does not block, and it does not stop writers from running.
- The code reads `store.get(key)` without holding any real lock.
- `lock.validate(stamp)` checks if a write happened since the stamp was issued. If a write happened, `validate` returns `false`.
- If validation fails, the code falls back to `lock.readLock()`, a normal blocking read lock, and reads again. This guarantees a correct value.
- `put` and `evict` use `writeLock()`, which is exclusive, same idea as before.

The optimistic path is fast because, in the common case (no concurrent write), it does zero blocking and zero CAS-based lock acquire. It only pays the cost of one volatile-like read and one validate check.

## How to Extend (Follow-ups)

Interviewers often ask these follow-ups. Know the answers.

1. **Add a fair mode.** `new ReentrantReadWriteLock(true)` makes the lock fair: threads get the lock roughly in the order they asked for it. This stops writer starvation (where writers wait forever because readers keep arriving), but it can lower throughput, since it stops new readers from jumping ahead of a waiting writer.
2. **Add TTL or eviction (LRU/LFU).** Combine this design with the TTL cache problem. The read-write lock still guards the map; eviction bookkeeping (like an access-order list) needs its own lock or must happen inside the write lock.
3. **Replace with `ConcurrentHashMap` and compare.** For a plain cache with no extra invariants across keys, `ConcurrentHashMap` alone (no `ReentrantReadWriteLock` needed) is often simpler and just as fast, or faster, because it uses lock striping (splitting the map into segments, so unrelated keys do not block each other) and lock-free reads.
4. **Support batch operations.** For example, `putAll(Map)` that must apply atomically. This needs the write lock held for the whole batch, which is easy with `ReentrantReadWriteLock`, but harder to make atomic with plain `ConcurrentHashMap`.
5. **Add a `computeIfAbsent`-style method.** This needs care: you cannot upgrade from read lock to write lock. The usual pattern is: try a read lock first; if the key is missing, release the read lock, acquire the write lock, check again (another thread may have added it while you waited), then compute and insert.
6. **Try `StampedLock`'s read-write mode too**, not just optimistic mode, and discuss when to pick each of the three modes (write, pessimistic read, optimistic read).

## Complexity & Thread-Safety Notes

- **Time complexity**: `get`, `put`, `evict` are O(1) on average (backed by `HashMap`), plus lock acquire/release overhead.
- **Space complexity**: O(n), where n is the number of cached entries.
- **Thread safety of `ReentrantReadWriteLock` version**: Safe. The read lock and write lock are mutually exclusive with the write lock, so `HashMap` (which is not thread-safe on its own) is always accessed under some lock.
- **Thread safety of `StampedLock` version**: Safe, but only if you always call `validate` after an optimistic read, and always fall back to a real lock on failure. Forgetting to validate is a common bug that leads to reading a half-updated `HashMap`, which can throw exceptions or return wrong data (`HashMap` is not safe for concurrent structural changes without a lock).
- **`StampedLock` is not reentrant.** Calling a `StampedLock`-guarded method recursively from the same thread, or acquiring the write lock while already holding it, causes a deadlock. `ReentrantReadWriteLock` allows this, because it tracks the owning thread and hold count.
- **`StampedLock` has no ownership tracking.** Any thread can call `unlockWrite(stamp)` if it has the stamp value, even if it did not acquire the lock. This means passing the stamp value around, or mismatched lock/unlock calls, can silently break the lock's guarantees. Keep stamp usage inside one method, with try/finally.
- **Starvation**: With a non-fair `ReentrantReadWriteLock` (the default), a steady stream of readers can, in theory, starve a writer, because new readers can keep entering while a writer waits. Use the fair constructor if this matters.

## Interview Tips & Common Mistakes

1. **Always release locks in a `finally` block.** If `store.get(key)` throws an exception (unlikely here, but possible with custom objects), the lock must still be released, or every other thread will hang forever. Interviewers watch for this.
2. **Do not try to upgrade a read lock to a write lock.** This is one of the most common mistakes in interviews. Explain clearly: release the read lock first, then acquire the write lock. Downgrading (write to read) is fine and different from upgrading.
3. **Do not assume `ReentrantReadWriteLock` is always faster than `synchronized`.** It is only faster for read-heavy workloads with critical sections big enough to make the lock overhead worth it. Say this out loud in the interview. It shows judgment, not just memorized syntax.
4. **Know the difference between `StampedLock` optimistic read and `ReentrantReadWriteLock` read lock.** Optimistic read never blocks and never stops a writer. Pessimistic read (in both lock types) blocks writers. If a `get` method must never return stale-but-inconsistent data, and you cannot easily "validate and retry," prefer the pessimistic (blocking) read.
5. **Mention `ConcurrentHashMap` as a baseline.** For a simple key-value cache with no multi-key invariants, `ConcurrentHashMap` alone often beats a hand-written read-write lock, because of its internal lock striping and non-blocking reads. Only add `ReentrantReadWriteLock` or `StampedLock` when you need to protect an invariant across more than one field or more than one map (for example, size counters, secondary indexes, or eviction lists that must stay in sync with the main map).
6. **Do not forget `synchronized` map is the baseline to compare against, too.** `Collections.synchronizedMap(new HashMap<>())` uses one lock for every operation, including reads. It is simple and correct, but it does not let readers run in parallel. It is a fine answer for "the simplest safe cache," but a weak answer for "the fastest safe cache under read-heavy load."
7. **Be ready to explain why `HashMap` needs a lock at all.** Under concurrent structural changes (resize, bucket linking), `HashMap` can corrupt its internal state, cause infinite loops, or lose entries. This is why plain `HashMap` is never safe to share across threads without an external lock or a swap to `ConcurrentHashMap`.
8. **If asked to benchmark**, mention tools like JMH (Java Microbenchmark Harness) rather than a simple `System.nanoTime()` loop, since JIT warmup and other effects can mislead simple timing code.
