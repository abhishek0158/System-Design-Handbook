# Connection / Object Pool

## Problem

Some objects are expensive to create. A database connection is one example. Opening a connection means a TCP handshake, authentication, and setup on the database server. This can take tens of milliseconds. If your service creates a new connection for every request, it becomes slow and it can overload the database.

The fix is object pooling. We create a small, fixed number of expensive objects once. We keep them in a pool. When code needs one, it "borrows" it from the pool. When code is done, it "returns" it to the pool. The object gets reused by the next borrower, instead of being destroyed and recreated.

Design a generic, thread-safe object pool. It must work for any type of pooled object (database connections, network sockets, thread-safe parser instances, and so on), not just one specific type. Many threads will call the pool at the same time, so it must be safe under concurrent use.

## Requirements & Clarifying Questions

Ask these questions before coding. They show the interviewer you think about real system behavior, not just data structures.

1. **What happens when the pool is empty and at max size?** Should the caller block and wait for a free object, block with a timeout, or fail right away with an exception?
2. **What is the maximum pool size?** Is it fixed at creation time, or can it change while the pool is running?
3. **Are objects created eagerly or lazily?** Do we create all `maxSize` objects up front, or create them one at a time, only when needed, up to the max?
4. **How do we know an object is still good to use?** A pooled database connection can go stale — the network can drop, or the database can close it from its side. Do we validate an object before handing it out?
5. **What if a borrowed object turns out to be broken?** Should the pool destroy it and create a fresh replacement, or just remove it and shrink the pool?
6. **Do idle objects need to be closed after sitting unused for too long?** Many real database pools close idle connections after some minutes, to free resources and to match database-side idle timeouts.
7. **What if a caller forgets to return an object?** This is a resource leak. Should the pool detect this, log a warning, or use a design that makes leaks hard to write by mistake?
8. **Is `close()` on the pool itself supported?** When the whole application shuts down, the pool should close every real object it holds (for example, closing every actual database connection).

For this solution, we build a fixed-max-size pool, lazy creation, blocking borrow with an optional timeout, validation before handing out an object, and idle eviction. We use a generic factory so the pool works for any object type.

## Design / Approach

We split the pool into three pieces:

1. **`PooledObjectFactory<T>`** — an interface that knows how to create, validate, and destroy the pooled object type. The pool itself never has type-specific logic.
2. **`ObjectPool<T>`** — the pool. It holds free objects in a `BlockingQueue<T>`. It uses a `Semaphore` to track and limit the total number of objects that exist (both free and currently borrowed), so we never create more than `maxSize` objects.
3. **`PooledConnection` wrapper (optional but recommended)** — an `AutoCloseable` wrapper around a borrowed object, so the caller can use try-with-resources. Calling `close()` on the wrapper returns the real object to the pool, instead of destroying it.

Why both a `BlockingQueue` and a `Semaphore`? The queue holds only the objects that are free right now. The semaphore tracks the *total* count, including objects that are currently checked out by some other thread. We need this total count so we know when we are allowed to create a brand new object (we are under `maxSize`) versus when we must wait for someone to return one (we are at `maxSize` and all are checked out).

`acquire()` flow:
1. Try to get a permit from the semaphore, waiting up to the given timeout. A permit means "you are allowed to hold one object, either an existing one or a newly created one."
2. Once we have a permit, try to take a free object from the queue without waiting (`poll()`).
3. If the queue had a free object, validate it. If it is valid, return it. If it is not valid, destroy it and create a new one in its place (the permit is already accounted for, so this does not exceed `maxSize`).
4. If the queue was empty, it means we are allowed to grow (the semaphore gave us a permit, so we are under `maxSize`). Create a brand new object.
5. If the semaphore timeout expires before we get a permit, throw a timeout exception (or return `null`/`Optional.empty()`, depending on the interface style).

`release()` flow:
1. Validate the object one more time (optional, but a good safety check).
2. If valid, put it back in the free queue.
3. If not valid, destroy it and create a fresh replacement, so the pool's total object count does not shrink over time due to bad objects.
4. Release the semaphore permit, whether the object went back to the queue or was replaced — the permit tracks "one object slot is now free," not "this specific object is free."

## Java Solution

### 1. The factory interface

```java
public interface PooledObjectFactory<T> {
    T create() throws Exception;
    boolean validate(T object);
    void destroy(T object);
}
```

Example factory for a database connection, using a real `javax.sql.DataSource` under the hood:

```java
public class ConnectionFactory implements PooledObjectFactory<Connection> {
    private final String url, user, password;

    public ConnectionFactory(String url, String user, String password) {
        this.url = url;
        this.user = user;
        this.password = password;
    }

    @Override
    public Connection create() throws Exception {
        return DriverManager.getConnection(url, user, password);
    }

    @Override
    public boolean validate(Connection connection) {
        try {
            // isValid runs a lightweight check against the database,
            // with a timeout in seconds.
            return connection != null && connection.isValid(2);
        } catch (SQLException e) {
            return false;
        }
    }

    @Override
    public void destroy(Connection connection) {
        try {
            if (connection != null) connection.close();
        } catch (SQLException ignored) {
            // Closing a broken connection can itself throw. Safe to ignore here.
        }
    }
}
```

### 2. The generic object pool

```java
import java.util.concurrent.*;
import java.util.Set;
import java.util.concurrent.atomic.AtomicBoolean;

public class ObjectPool<T> implements AutoCloseable {

    private final PooledObjectFactory<T> factory;
    private final int maxSize;
    private final BlockingQueue<T> freeObjects;
    private final Semaphore permits;
    // Tracks every live object, free or borrowed, so close() can destroy all of them.
    private final Set<T> allObjects = ConcurrentHashMap.newKeySet();
    private final AtomicBoolean closed = new AtomicBoolean(false);

    public ObjectPool(PooledObjectFactory<T> factory, int maxSize) {
        this.factory = factory;
        this.maxSize = maxSize;
        this.freeObjects = new LinkedBlockingQueue<>();
        this.permits = new Semaphore(maxSize, true); // fair: first-come, first-served
    }

    /** Borrow an object, waiting up to timeout for one to become available. */
    public T acquire(long timeout, TimeUnit unit) throws InterruptedException, TimeoutException {
        if (closed.get()) {
            throw new IllegalStateException("Pool is closed");
        }
        if (!permits.tryAcquire(timeout, unit)) {
            throw new TimeoutException("Timed out waiting for a pooled object");
        }

        try {
            T object = freeObjects.poll(); // non-blocking; we already hold a permit
            if (object == null) {
                // We are under maxSize (the permit proves it). Create a new one.
                object = createTracked();
            } else if (!factory.validate(object)) {
                // Stale or broken object. Drop it and create a fresh one.
                allObjects.remove(object);
                factory.destroy(object);
                object = createTracked();
            }
            return object;
        } catch (Exception e) {
            permits.release(); // creation failed; give the permit back
            throw new RuntimeException("Failed to create pooled object", e);
        }
    }

    /** Return a borrowed object to the pool. */
    public void release(T object) {
        if (object == null) return;

        if (closed.get() || !factory.validate(object)) {
            allObjects.remove(object);
            factory.destroy(object);
        } else {
            freeObjects.offer(object);
        }
        permits.release();
    }

    private T createTracked() throws Exception {
        T object = factory.create();
        allObjects.add(object);
        return object;
    }

    @Override
    public void close() {
        if (!closed.compareAndSet(false, true)) return;
        T obj;
        while ((obj = freeObjects.poll()) != null) {
            allObjects.remove(obj);
            factory.destroy(obj);
        }
        // Any objects still checked out get destroyed when release() is called,
        // because closed is now true.
        for (T obj2 : allObjects) {
            factory.destroy(obj2);
        }
        allObjects.clear();
    }
}
```

### 3. A safe wrapper for try-with-resources

Calling `acquire()`/`release()` by hand is risky — a caller can forget `release()` on an exception path. Wrap the borrowed object so it can be used in try-with-resources:

```java
public final class PooledConnection implements AutoCloseable {
    private final Connection connection;
    private final ObjectPool<Connection> pool;

    PooledConnection(Connection connection, ObjectPool<Connection> pool) {
        this.connection = connection;
        this.pool = pool;
    }

    public Connection get() {
        return connection;
    }

    @Override
    public void close() {
        pool.release(connection); // "closing" the wrapper returns it, does not destroy it
    }
}
```

Usage:

```java
ObjectPool<Connection> pool = new ObjectPool<>(
        new ConnectionFactory(url, user, password), 10);

try (PooledConnection pc = new PooledConnection(pool.acquire(5, TimeUnit.SECONDS), pool)) {
    Connection conn = pc.get();
    // use conn for a query
} // pc.close() runs automatically here, even if an exception was thrown above
catch (TimeoutException e) {
    // pool was busy; handle it (retry, fail the request, etc.)
}
```

### 4. Idle eviction (optional background task)

A pooled connection that sits idle for a long time can go stale, or waste a database-side slot. A background thread can scan the free queue and evict old, unused entries:

```java
public class IdleEvictor<T> {
    private final ScheduledExecutorService scheduler =
            Executors.newSingleThreadScheduledExecutor(r -> {
                Thread t = new Thread(r, "pool-idle-evictor");
                t.setDaemon(true);
                return t;
            });

    public void start(ObjectPool<T> pool, long checkIntervalSeconds) {
        scheduler.scheduleAtFixedRate(() -> {
            // In a full implementation, freeObjects would store
            // (object, lastReturnedTimestamp) pairs, so we can check age here
            // and call pool internals to evict and destroy old idle entries.
        }, checkIntervalSeconds, checkIntervalSeconds, TimeUnit.SECONDS);
    }

    public void stop() {
        scheduler.shutdownNow();
    }
}
```

To support this properly, change `freeObjects` from `BlockingQueue<T>` to `BlockingQueue<IdleEntry<T>>`, where `IdleEntry` wraps the object with a `lastReturnedAt` timestamp. The evictor task polls entries, checks age, and either destroys stale ones (releasing a permit and removing from `allObjects`) or puts fresh ones back.

## How It Works

The `Semaphore` is the key to correctness. It always has exactly `maxSize` permits in total. A permit represents "the right to own one object instance," whether that instance is sitting free in the queue or currently in a caller's hand. Because `acquire()` always takes a permit *before* looking at the queue, the pool can never create more than `maxSize` objects — the semaphore blocks the `maxSize + 1`-th caller until someone releases.

The `BlockingQueue` only needs to answer "is there a free object right now?" It does not need to block callers itself, because the semaphore already handles waiting. That is why we use `poll()` (non-blocking) on the queue inside `acquire()`, not `take()` (blocking) — blocking would be wrong here, since an empty queue while holding a permit means "create a new one," not "wait."

Validation happens at both borrow and return time. Checking at borrow time catches objects that went stale while sitting idle in the queue (for example, the database closed the connection from its side). Checking at return time catches objects that broke while in use (for example, the caller's SQL query failed due to a dropped network connection). Either way, a bad object is destroyed and a fresh one takes its place, so the *pool's health* stays independent of any one bad connection.

The `fair` flag on `Semaphore` (`new Semaphore(maxSize, true)`) makes waiting threads get served in the order they arrived (FIFO). Without fairness, a thread could wait a long time under high contention, because newer threads might "jump the queue" when a permit frees up. Fairness has a small throughput cost, but for a connection pool, predictable wait order is usually worth it.

## How to Extend (Follow-ups)

Interviewers often ask about these directions after the base solution works:

**1. What if a caller never calls `release()`?**
This is a connection leak. The pool slowly runs out of objects, and every later caller times out waiting, even though the database itself is healthy. Three defenses help:
- Encourage (or force) the try-with-resources pattern shown above, so `close()` on the wrapper runs even when an exception is thrown.
- Add a "leak detection" timer: record a stack trace or a timestamp when an object is borrowed. A background task checks objects that have been checked out for longer than some threshold (say, 60 seconds) and logs a warning with the stack trace of who borrowed it. HikariCP calls this `leakDetectionThreshold`.
- As a last resort, some pools force-reclaim an object after a very long checkout, but this is dangerous, since the original caller might still be using it. Logging is usually safer than force-reclaiming.

**2. Minimum idle size.**
Instead of only lazy creation, keep a minimum number of idle objects ready at all times (for example, always keep at least 2 free connections), so a burst of new requests does not have to pay the connection-creation cost. A background task tops up the pool toward `minIdle` when it drops below that number.

**3. Per-borrow timeout vs. pool-wide timeout.**
Our `acquire(timeout, unit)` takes a timeout per call. Some designs instead configure one fixed `connectionTimeout` on the pool itself, so every caller gets the same wait budget, and callers cannot accidentally starve others by passing `Long.MAX_VALUE`.

**4. Metrics.**
Track active count, idle count, total created, total destroyed due to failed validation, and average wait time. These numbers are essential in production to size the pool correctly and to catch leaks early.

**5. Compare with HikariCP.**
HikariCP is the most common production-grade connection pool for Java today. Compared to our simple design, it adds:
- A lock-free, highly tuned data structure (`ConcurrentBag`) instead of a plain `BlockingQueue`, to cut down on thread contention under very high load.
- Connection aliveness checks that use a fast internal ping instead of always running a full validation query.
- Automatic detection and closing of leaked connections, with a configurable `leakDetectionThreshold`.
- `minimumIdle` and `maximumPoolSize` settings, plus `idleTimeout` and `maxLifetime` (a connection is retired after living too long, even if it is healthy, to avoid very old connections that might be handled differently by network infrastructure like load balancers).
- Careful handling of connection state reset (auto-commit, transaction isolation level) before returning a connection to the pool, so a connection never leaks unfinished transaction settings to the next borrower.

Mentioning these HikariCP details in an interview shows you understand that a real pool must handle far more edge cases than a class-room version, even though the core idea (bounded count, blocking wait, reuse) is the same.

## Complexity & Thread-Safety Notes

| Operation | Time | Notes |
|---|---|---|
| `acquire()` | O(1) plus possible wait time | Semaphore acquire + non-blocking queue poll; object creation cost only on pool growth |
| `release()` | O(1) | Validate, offer to queue, release permit |
| `close()` (pool shutdown) | O(maxSize) | Destroys every live object once |

Space use is O(maxSize) — never more than `maxSize` real objects exist at once, tracked in `allObjects`.

**Thread-safety approach used above:**
- `Semaphore` enforces the hard cap on total object count. It is safe for many threads to call `acquire`/`release` on it concurrently; this is exactly what it is designed for.
- `LinkedBlockingQueue` is itself thread-safe for concurrent `offer`/`poll` calls, so the free-object hand-off needs no extra locking.
- `ConcurrentHashMap.newKeySet()` gives a thread-safe set for tracking every live object, needed only for a clean `close()`.
- `AtomicBoolean closed` gives a safe, race-free way to mark the pool closed exactly once (`compareAndSet`), and to have `release()` react correctly to a pool that closed while an object was checked out.
- The permit-then-queue order in `acquire()` is essential: doing it in the other order (checking the queue before taking a permit) opens a race where two threads could both see an empty queue and both try to create a new object, breaking the `maxSize` guarantee.
- No object is ever in two places at once (in the queue and also considered "borrowed") because `poll()` and `offer()` on the queue are atomic per element, and every code path that removes an object from `allObjects` does so before calling `destroy()`.

## Interview Tips & Common Mistakes

- **Explain the semaphore-plus-queue combination clearly.** Many candidates use only a `BlockingQueue` and call `take()` to borrow. This works only if all `maxSize` objects are created up front (eager creation). If you want lazy creation, you need something like a semaphore to know when you are allowed to grow versus when you must wait, since an empty queue is ambiguous otherwise — it could mean "all objects are checked out" or "we have not created any yet."
- **Do not check the queue before acquiring a permit.** This ordering mistake breaks the `maxSize` cap under concurrency, as explained above. State the correct order out loud: permit first, then queue.
- **Always validate before handing out an object.** A common gap is to only validate on return, not on borrow. An object can go bad while sitting idle in the pool (for example, a database-side connection timeout), so borrow-time validation matters too.
- **Talk about the leak problem, even if not asked.** Mentioning try-with-resources and leak detection shows production experience, not just algorithm knowledge.
- **Do not forget `close()` on the pool.** On application shutdown, every real resource (every actual `Connection`) must be closed. Forgetting this leaks connections at the database server even after your JVM exits.
- **Be ready to discuss fairness vs. throughput.** A fair semaphore avoids starvation but can reduce raw throughput slightly. Know this trade-off exists and be able to name it.
- **Know the difference between "pool of objects" and "cache of objects."** A pool hands out one object to one borrower at a time (mutual exclusion of use). A cache can let many callers read the same cached value at once. Confusing the two designs is a common conceptual slip in interviews.
- **Relate to HikariCP by name.** Interviewers evaluating 3–4 years of experience often want to hear that you have used or at least studied a real pool implementation, not just built a toy version from scratch.
