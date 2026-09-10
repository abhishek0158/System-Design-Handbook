# In-Memory Cache (TTL + Eviction)

## Problem

Design and build an in-memory cache for a Java application. The cache stores key-value pairs. It must support these features:

- `get(key)`, `put(key, value)`, and `delete(key)` operations.
- Per-entry time-to-live (TTL). TTL means each entry can expire after a fixed time.
- Size-based eviction. When the cache is full, it must remove entries using a policy, for example Least Recently Used (LRU).
- Thread safety. Many threads can call the cache at the same time.

This is a common interview question. It tests your knowledge of Java collections, concurrency, and design patterns (Strategy pattern).

## Requirements & Clarifying Questions

Before coding, ask these questions in an interview. They show you think about edge cases.

1. **What is the eviction policy?** LRU (Least Recently Used) is common. But the interviewer may want LFU (Least Frequently Used) too. This tells you to use a **Strategy pattern**, so the policy is swappable.
2. **Does every entry need its own TTL, or is TTL global for the whole cache?** Per-entry TTL is more flexible and more common in real systems (like Redis).
3. **What happens on expiry?** Do we remove the entry only when someone reads it (lazy expiry), or do we also remove it in the background (active expiry)? Most production caches do both.
4. **What is the max size?** Is it a count of entries, or a memory size (bytes)? For this problem, we assume a max entry count.
5. **Is the cache used by many threads?** Yes. We must handle concurrent `get`, `put`, and `delete` calls safely, without corrupting internal data structures.
6. **What happens on eviction: do we notify anyone?** Optional. We can add a listener callback for evicted entries (useful for logging or metrics).
7. **What should `get` do for an expired key?** Return `null` (or `Optional.empty()`), and remove the entry as a side effect.
8. **Do we need to update "last used" time on `put` as well as `get`?** Yes, for LRU, both reads and writes count as "use."

## Design / Approach

### Core pieces

1. **Cache store**: A `ConcurrentHashMap<K, CacheEntry<V>>` holds the key-value data. `ConcurrentHashMap` gives thread-safe reads and writes without one big lock for the whole map.
2. **CacheEntry**: Wraps the value, the expiry time, and metadata needed by the eviction policy (like last-access time, for LRU).
3. **EvictionPolicy interface** (Strategy pattern): Defines how the cache decides which key to remove when it is full. `LRUEvictionPolicy` is the default. `LFUEvictionPolicy` can be added later without changing the cache class.
4. **Lazy expiry**: On every `get`, we check if the entry's expiry time has passed. If yes, we remove it and return "not found." This catches expired entries even if the background thread has not run yet.
5. **Active expiry**: A background thread runs on a fixed schedule (using `ScheduledExecutorService`). It scans the cache and removes expired entries. This is needed because some entries may never be read again. Without active expiry, expired entries stay in memory forever ("memory leak").
6. **Locking for eviction order**: `ConcurrentHashMap` is thread-safe for basic get/put, but it does not track "order of use" safely by itself. To track LRU order under concurrent access, we need one more layer:
    - Option A: Use `Collections.synchronizedMap` with a `LinkedHashMap` in access-order mode. Simple, but it locks the whole map on every operation.
    - Option B (chosen here): Use `ConcurrentHashMap` for the data, plus a separate thread-safe structure (a `ConcurrentLinkedDeque` or a doubly linked list guarded by a lock) to track LRU order. We only lock this small tracking structure, not the whole map. This gives better concurrency.
    - We use a `ReentrantLock` to guard the eviction-order updates, since they involve multiple steps (remove from list, add to front) that must happen as one atomic unit.

### Why both lazy and active expiry?

- **Lazy expiry** is cheap. It only checks the accessed key. But it does nothing for keys that are never read again. Those keys stay in memory and waste space.
- **Active expiry** cleans up in the background. It finds and removes expired keys even if nobody reads them. It runs on a timer, so there is a small delay before cleanup, but it stops memory leaks.
- Real systems, like Redis, use a similar mix: check on read, plus a periodic background sweep.

### Why Strategy pattern for eviction policy?

The eviction policy decides "which entry to remove when the cache is full." Different use cases need different policies:

- **LRU**: Remove the entry that was not used for the longest time. Good for general-purpose caching.
- **LFU**: Remove the entry used the fewest times. Good when some keys are always hot and some are rarely used.

If we hard-code LRU logic inside the cache class, we cannot switch policies later, and we cannot test them separately. The Strategy pattern fixes this: the cache class only calls `evictionPolicy.recordAccess(key)` and `evictionPolicy.evictKey()`. It does not know or care which policy is used underneath.

## Java Solution

```java
import java.time.Instant;
import java.util.Map;
import java.util.Optional;
import java.util.concurrent.ConcurrentHashMap;
import java.util.concurrent.ConcurrentLinkedDeque;
import java.util.concurrent.Executors;
import java.util.concurrent.ScheduledExecutorService;
import java.util.concurrent.TimeUnit;
import java.util.concurrent.locks.ReentrantLock;
import java.util.function.BiConsumer;

/**
 * Strategy interface for eviction. Any policy (LRU, LFU, FIFO...) must
 * implement this. The cache calls these methods; it does not know the
 * internal logic of the policy.
 */
interface EvictionPolicy<K> {

    /** Called every time a key is read or written (a "use" event). */
    void recordAccess(K key);

    /** Called when a key is removed from the cache (expired or deleted). */
    void remove(K key);

    /** Returns the key that should be evicted next, or null if none. */
    K evictKey();
}

/**
 * LRU (Least Recently Used) policy.
 * Keeps keys in a deque. Most recently used key is at the tail.
 * Least recently used key is at the head. A lock guards the "move to
 * tail" step, because it is really two operations (remove + add) that
 * must happen together.
 */
class LRUEvictionPolicy<K> implements EvictionPolicy<K> {

    private final ConcurrentLinkedDeque<K> order = new ConcurrentLinkedDeque<>();
    private final ReentrantLock lock = new ReentrantLock();

    @Override
    public void recordAccess(K key) {
        lock.lock();
        try {
            // Remove old position (if present) then add to tail.
            // This keeps only one entry per key and marks it "most recent."
            order.remove(key);
            order.addLast(key);
        } finally {
            lock.unlock();
        }
    }

    @Override
    public void remove(K key) {
        lock.lock();
        try {
            order.remove(key);
        } finally {
            lock.unlock();
        }
    }

    @Override
    public K evictKey() {
        lock.lock();
        try {
            return order.pollFirst(); // head = least recently used
        } finally {
            lock.unlock();
        }
    }
}

/** Internal wrapper: holds the value and its absolute expiry time. */
class CacheEntry<V> {
    final V value;
    final long expiryEpochMillis; // Long.MAX_VALUE means "never expires"

    CacheEntry(V value, long expiryEpochMillis) {
        this.value = value;
        this.expiryEpochMillis = expiryEpochMillis;
    }

    boolean isExpired(long nowMillis) {
        return nowMillis >= expiryEpochMillis;
    }
}

/**
 * Generic, thread-safe, in-memory cache.
 * Supports per-entry TTL and pluggable eviction policy.
 */
public class InMemoryCache<K, V> {

    private final ConcurrentHashMap<K, CacheEntry<V>> store = new ConcurrentHashMap<>();
    private final EvictionPolicy<K> evictionPolicy;
    private final int maxSize;
    private final ScheduledExecutorService cleaner;
    private BiConsumer<K, V> onEvict; // optional listener, set via setter

    public InMemoryCache(int maxSize, EvictionPolicy<K> evictionPolicy,
                          long cleanupIntervalSeconds) {
        this.maxSize = maxSize;
        this.evictionPolicy = evictionPolicy;

        // Active expiry: background thread sweeps expired entries.
        this.cleaner = Executors.newSingleThreadScheduledExecutor(r -> {
            Thread t = new Thread(r, "cache-cleaner");
            t.setDaemon(true); // do not block JVM shutdown
            return t;
        });
        cleaner.scheduleAtFixedRate(this::removeExpiredEntries,
                cleanupIntervalSeconds, cleanupIntervalSeconds, TimeUnit.SECONDS);
    }

    public void setOnEvictListener(BiConsumer<K, V> listener) {
        this.onEvict = listener;
    }

    /** Put a value with a TTL (time-to-live) in seconds. Use -1 for "never expires." */
    public void put(K key, V value, long ttlSeconds) {
        long expiry = (ttlSeconds < 0)
                ? Long.MAX_VALUE
                : Instant.now().toEpochMilli() + TimeUnit.SECONDS.toMillis(ttlSeconds);

        // Evict BEFORE inserting a brand-new key, if we are at capacity.
        // (An update to an existing key does not grow the map size.)
        if (!store.containsKey(key) && store.size() >= maxSize) {
            evictOne();
        }

        store.put(key, new CacheEntry<>(value, expiry));
        evictionPolicy.recordAccess(key);
    }

    /** Reads a value. Returns empty if missing or expired (lazy expiry). */
    public Optional<V> get(K key) {
        CacheEntry<V> entry = store.get(key);
        if (entry == null) {
            return Optional.empty();
        }
        if (entry.isExpired(Instant.now().toEpochMilli())) {
            // Lazy expiry: clean up now, since we noticed it here.
            removeInternal(key, entry.value, "expired-on-read");
            return Optional.empty();
        }
        evictionPolicy.recordAccess(key); // reading counts as "use" for LRU
        return Optional.of(entry.value);
    }

    /** Deletes a key. Returns true if a key was actually present and removed. */
    public boolean delete(K key) {
        CacheEntry<V> removed = store.remove(key);
        if (removed != null) {
            evictionPolicy.remove(key);
            return true;
        }
        return false;
    }

    public int size() {
        return store.size();
    }

    /** Shuts down the background cleaner thread. Call this on app shutdown. */
    public void shutdown() {
        cleaner.shutdownNow();
    }

    // ---- internal helpers ----

    private void evictOne() {
        K victim = evictionPolicy.evictKey();
        if (victim != null) {
            CacheEntry<V> removed = store.remove(victim);
            if (removed != null && onEvict != null) {
                onEvict.accept(victim, removed.value);
            }
        }
    }

    private void removeExpiredEntries() {
        long now = Instant.now().toEpochMilli();
        for (Map.Entry<K, CacheEntry<V>> e : store.entrySet()) {
            if (e.getValue().isExpired(now)) {
                removeInternal(e.getKey(), e.getValue().value, "expired-by-cleaner");
            }
        }
    }

    private void removeInternal(K key, V value, String reason) {
        // remove(key, value)-style check avoids removing an entry that was
        // replaced by a newer put() between our expiry check and this call.
        if (store.remove(key) != null) {
            evictionPolicy.remove(key);
            if (onEvict != null) {
                onEvict.accept(key, value);
            }
        }
    }
}
```

### Example usage

```java
public class CacheDemo {
    public static void main(String[] args) throws InterruptedException {
        InMemoryCache<String, String> cache =
                new InMemoryCache<>(3, new LRUEvictionPolicy<>(), 5);

        cache.setOnEvictListener((k, v) -> System.out.println("Evicted: " + k));

        cache.put("a", "apple", 10);   // expires in 10 seconds
        cache.put("b", "banana", -1);  // never expires
        cache.put("c", "cherry", 10);

        cache.get("a"); // "a" is now most recently used

        cache.put("d", "date", 10); // cache full (size 3): evicts "b" (least recently used)

        System.out.println(cache.get("b")); // Optional.empty (evicted)
        System.out.println(cache.get("a")); // Optional[apple]

        Thread.sleep(11_000);
        System.out.println(cache.get("a")); // Optional.empty (expired, lazy check)

        cache.shutdown();
    }
}
```

## How It Works

**`put(key, value, ttl)`**: First, it checks if the cache is full and the key is new. If so, it asks the eviction policy for a victim key and removes it. Then it stores the new entry with its computed expiry time. Finally, it tells the eviction policy that this key was just used.

**`get(key)`**: It looks up the entry in the map. If missing, returns empty. If present but expired, it removes the entry right away (lazy expiry) and returns empty. If present and valid, it tells the eviction policy "this key was used" (this moves it to the "most recently used" end for LRU), then returns the value.

**Background cleaner**: A `ScheduledExecutorService` runs `removeExpiredEntries()` on a timer. It loops through all entries and removes any that are expired. This is active expiry. It runs on a **daemon thread**, so it will not stop the JVM from shutting down if we forget to call `shutdown()`.

**LRU tracking**: The `LRUEvictionPolicy` keeps a separate deque of keys in "least to most recently used" order. Every access moves the key to the tail. Eviction always removes from the head. A `ReentrantLock` protects the "remove-then-add" step, because doing it as two separate atomic operations (without a lock) could let two threads interleave and corrupt the order (for example, both threads see the key as "not present," and both add it, causing a duplicate).

**Thread safety overall**: `ConcurrentHashMap` handles safe concurrent access to the actual key-value data. The eviction-order tracking is a smaller, separate data structure with its own lock, so we do not lock the whole cache on every `get`. This means many threads can read different keys at the same time without blocking each other on the main map; they only briefly contend on the small lock inside the eviction policy.

## How to Extend (Follow-ups)

- **Add LFU (Least Frequently Used) policy**: Create `LFUEvictionPolicy<K>` implementing `EvictionPolicy<K>`. Track an access count per key (for example, in a `ConcurrentHashMap<K, AtomicInteger>`), and use a min-heap or sorted structure to find the key with the lowest count. Pass it into `InMemoryCache` constructor. No change needed in `InMemoryCache` itself. This shows the interviewer you understand the Strategy pattern's value.
- **Add a size-based limit using memory, not entry count**: Add a `weigher` function (`V -> long`) that estimates the size in bytes of each value. Track total weight instead of `store.size()`. This is how Caffeine's "maximumWeight" works.
- **Add write-through or write-behind support**: Add a `CacheLoader<K, V>` interface for computing a value on cache miss, and a way to write updates back to a database. This turns the cache into a "cache-aside" or "read-through" cache.
- **Add statistics**: Track hit count, miss count, and eviction count with `AtomicLong` fields. Expose a `getStats()` method. Useful for monitoring cache effectiveness (hit ratio).
- **Support `computeIfAbsent`-style atomic "get or load"**: Add a method like `V getOrCompute(K key, Function<K, V> loader, long ttl)`. This avoids the classic "cache stampede" problem, where many threads miss the cache at the same time and all call the expensive loader function together. Use `ConcurrentHashMap.compute` or a per-key lock to make sure only one thread loads a given key.
- **Bound the cleaner thread's work**: For very large caches, scanning the whole map on every cleanup cycle is costly. A better design uses a **timing wheel** or a **min-heap ordered by expiry time**, so the cleaner only looks at entries that are actually close to expiring.

## Complexity & Thread-Safety Notes

- **`get`**: O(1) average, for the map lookup. The LRU update (`recordAccess`) is also O(1) average (deque remove + add), but it briefly holds a lock.
- **`put`**: O(1) average for the map insert. If eviction is needed, `evictKey()` is O(1) (deque `pollFirst`).
- **`delete`**: O(1) average.
- **Active expiry sweep**: O(n), where n is the number of entries in the cache. This runs on a timer, not on every request, so it does not slow down normal `get`/`put` calls.
- **Thread safety**: `ConcurrentHashMap` guarantees safe concurrent `get`, `put`, and `remove` on the main data. The `ReentrantLock` inside `LRUEvictionPolicy` guarantees the "move to most recently used" step is atomic. Without this lock, two threads updating LRU order at the same time could corrupt the deque or lose track of a key.
- **Race condition to watch for**: Between "check expired" and "remove," another thread could have already updated the same key with a new `put`. The `removeInternal` method in the code above works on the key, not a stale reference, but for stricter correctness, you could use `store.remove(key, entryReference)` which only removes if the value object reference still matches. This avoids removing a fresh entry that replaced an expired one at nearly the same moment.
- **Memory note**: If TTL is set but the key is never read again, only the active-expiry background thread will ever free that memory. This is why active expiry is not optional in a long-running production cache.

## Interview Tips & Common Mistakes

- **Do not use `synchronized` on the whole cache class.** This would make every `get` and `put` block every other thread, even for unrelated keys. Interviewers want to see you understand fine-grained concurrency, not a single big lock.
- **Do not forget the eviction check happens BEFORE insert, only for new keys.** If you evict after inserting, the cache can temporarily exceed `maxSize`, or you may accidentally evict the entry you just added.
- **Explain the difference between lazy and active expiry clearly.** A common mistake is implementing only one of them. Lazy expiry alone leaves "dead" entries in memory forever if nobody reads them. Active expiry alone means a `get` can still return a stale value for a short window before the next sweep — but in this design, we do lazy expiry on `get` too, so a `get` always returns a fresh check.
- **Mention the Strategy pattern by name.** The `EvictionPolicy` interface is the key design decision. It shows you can design for change: swapping LRU for LFU only means writing a new class, with no change to `InMemoryCache`.
- **Real-world comparison**: mention **Caffeine** and **Guava Cache** in the interview, since interviewers often ask "why not just use a library?"
    - **Guava Cache**: An older, well-known Java caching library. Supports TTL (`expireAfterWrite`, `expireAfterAccess`), size limits, and weak/soft references for values.
    - **Caffeine**: The modern replacement for Guava Cache. It uses a better algorithm called **Window TinyLFU** for eviction, which gives a higher cache hit rate than plain LRU in most real workloads. It also supports async loading, refresh-ahead, and detailed statistics. In real production Java systems, use Caffeine instead of writing your own cache — but knowing how to build one from scratch, as shown here, proves you understand what the library does internally.
- **Common bug in interviews**: Using `HashMap` instead of `ConcurrentHashMap`, or using `LinkedHashMap` without synchronizing it, then claiming the cache is "thread-safe." Always name the exact concurrency tool you use and explain why.
- **Daemon threads**: Always set background threads (like the cleaner) as daemon threads, or the JVM will not exit cleanly when the application stops. Also always provide a `shutdown()` method to stop the scheduled task cleanly.
