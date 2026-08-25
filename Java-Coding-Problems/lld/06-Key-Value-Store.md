# In-Memory Key-Value Store (LLD)

## Problem

Design an in-memory key-value store. It must support three basic operations:

- `get(key)`: return the value for a key, or "not found" if the key does not exist.
- `put(key, value)`: insert a new key-value pair, or update the value if the key already exists.
- `delete(key)`: remove a key-value pair.

The store must be generic. It should work for any key type `K` and any value type `V`, not just strings.

The store must be thread-safe. Many threads can call `get`, `put`, and `delete` at the same time, and the data must stay correct.

The store must support two extra features, often asked as follow-ups in the same interview:

1. **TTL (time to live).** A key can have an optional expiry time. After this time passes, the key acts as if it does not exist, even if no one deleted it.
2. **Simple transactions.** The store supports `begin()`, `commit()`, and `rollback()`. Inside a transaction, changes are held aside, not applied right away. `commit()` applies all held changes at once. `rollback()` throws them away. Reads from other threads must only ever see **committed** data — never a change that is still inside someone else's open transaction.

This question tests three skills at once: generic API design, concurrency control, and a mini version of database transaction logic. It is a common LLD (low-level design) round for backend roles.

## Requirements & Clarifying Questions

Ask these questions before coding. They show the interviewer you think about scope and edge cases.

1. **Are transactions per-thread, or shared across the whole store?** Real databases tie a transaction to one connection (one thread, in most simple designs). We will assume **per-thread** transactions: each thread has its own open transaction, invisible to other threads.
2. **Can transactions be nested?** For example, calling `begin()` twice before a `commit()`. We will support this, using a stack, so a nested transaction acts like a "savepoint."
3. **What isolation level do we need?** Full serializability (like a real database) is a large topic. We will aim for a simpler, common-sense rule: "no dirty reads" — other threads never see uncommitted changes. We will call out where this is weaker than full isolation.
4. **What happens if `commit()` or `rollback()` is called with no open transaction?** We will throw `IllegalStateException`. Silent no-ops hide bugs.
5. **Does `delete` inside a transaction need special handling?** Yes. If a transaction deletes a key that already has a committed value, the transaction must remember "this key is now deleted," not just "forget about this key." Otherwise, a read inside the same transaction would incorrectly fall back to the old committed value.
6. **How precise does TTL need to be?** We will use millisecond precision (`System.currentTimeMillis()`), checked lazily on read, plus a background sweep for keys that are never read again.
7. **Do we need persistence (surviving a restart)?** No — this is an in-memory store. We will mention persistence briefly as an extension, the way Redis does it.

## Design / Approach

The store has three layers:

```
                         ┌───────────────────────────────┐
                         │   Thread A: open transaction    │
                         │   changes = { x: PUT 5 }        │  <- ThreadLocal, private to Thread A
                         └───────────────────────────────┘
                                        │ commit()
                                        ▼
        get()/put()/delete()   ┌─────────────────────┐
   ─────────(no tx)──────────► │  Committed Store      │  <- ConcurrentHashMap<K, VersionedValue<V>>
                                │  { x: (5, ttl=none)   │      visible to ALL threads
                                │    y: (9, ttl=12:00) }│
                                └─────────────────────┘
                                        ▲
                         ┌───────────────────────────────┐
                         │   Thread B: no open transaction │
                         │   get("x") reads store directly │
                         └───────────────────────────────┘
```

**Base store.** A `ConcurrentHashMap<K, VersionedValue<V>>` holds all committed data. `VersionedValue<V>` is a small wrapper: it holds the value plus an optional expiry timestamp. `ConcurrentHashMap` gives us safe, lock-free-for-most-cases `get`/`put`/`remove` on individual keys, without us writing any lock code for the simple, non-transactional path.

**Transactions as a "scratch map."** When a thread calls `begin()`, we create a private `Map<K, Change<V>>` — a scratch space — just for that thread. A `Change` is one of two things: "put this value" or "delete this key" (a tombstone). While the transaction is open:
- `put`/`delete` write only into this scratch map, never into the shared store.
- `get` first checks the scratch map. If the key is there, return that (even a tombstone, which means "not found"). If not there, fall through to the shared, committed store.

This is the classic **Unit of Work** pattern: gather changes, then apply them as one batch. It is also similar to the **Memento** idea — the scratch map is a snapshot of pending edits that can be thrown away cleanly.

**Nested transactions as a stack.** Each thread's transactions live in a `Deque<Transaction<K,V>>` (used as a stack), stored in a `ThreadLocal`. `begin()` pushes a new scratch map. `commit()` pops the top scratch map:
- If the stack is now empty, this was the outermost transaction — apply its changes straight to the shared `ConcurrentHashMap`.
- If the stack still has a parent transaction, merge the changes into the parent's scratch map instead. Nothing becomes visible to other threads until the *outermost* commit happens.

`rollback()` just pops and discards the top scratch map — no merge.

**Why `ThreadLocal` gives correct isolation for free.** Since each thread's scratch map is private (a `ThreadLocal`, not shared state), no other thread can ever read it, by construction. This automatically satisfies "no dirty reads": other threads can only ever see the `ConcurrentHashMap`, which only changes at commit time. We do not need extra locks to hide in-progress work — the JVM's `ThreadLocal` mechanism already hides it.

**TTL.** `VersionedValue<V>` stores an expiry timestamp (or a sentinel value meaning "never expires"). Expiry is checked two ways:
- **Lazily**, inside `get()`: if the stored value is expired, treat it as missing, and clean it up.
- **Actively**, with a background `ScheduledExecutorService` that sweeps the map every second and removes anything expired. This matters for keys that are written once and never read again — without a sweep, they would sit in memory forever.

## Java Solution

```java
import java.util.ArrayDeque;
import java.util.Deque;
import java.util.HashMap;
import java.util.Map;
import java.util.concurrent.ConcurrentHashMap;
import java.util.concurrent.Executors;
import java.util.concurrent.ScheduledExecutorService;
import java.util.concurrent.TimeUnit;

public class InMemoryKeyValueStore<K, V> {

    private static final long NO_EXPIRY = -1L;

    // Wraps a value together with its optional expiry time (epoch millis).
    private static class VersionedValue<V> {
        final V value;
        final long expiryAtMillis; // NO_EXPIRY means "never expires"

        VersionedValue(V value, long expiryAtMillis) {
            this.value = value;
            this.expiryAtMillis = expiryAtMillis;
        }

        boolean isExpired() {
            return expiryAtMillis != NO_EXPIRY && System.currentTimeMillis() > expiryAtMillis;
        }
    }

    // One buffered change inside an open transaction.
    // PUT carries a value (and an optional ttl). DELETE is a "tombstone" —
    // it must be remembered, not just skipped, so a later get() inside the
    // same transaction returns "not found" instead of falling through to
    // the old committed value.
    private static class Change<V> {
        enum Type { PUT, DELETE }

        final Type type;
        final V value;
        final Long ttlMillis; // null = no TTL; only meaningful for PUT

        static <V> Change<V> put(V value, Long ttlMillis) {
            return new Change<>(Type.PUT, value, ttlMillis);
        }

        static <V> Change<V> delete() {
            return new Change<>(Type.DELETE, null, null);
        }

        private Change(Type type, V value, Long ttlMillis) {
            this.type = type;
            this.value = value;
            this.ttlMillis = ttlMillis;
        }
    }

    // One level of an open transaction: its own scratch map of changes.
    private static class Transaction<K, V> {
        final Map<K, Change<V>> changes = new HashMap<>();
    }

    // Committed data, visible to every thread. ConcurrentHashMap gives
    // per-key atomic get/put/remove without any extra locking from us.
    private final ConcurrentHashMap<K, VersionedValue<V>> store = new ConcurrentHashMap<>();

    // Each thread owns its own stack of open transactions. A stack lets
    // begin() be called more than once (nested transactions / savepoints).
    // Because this is a ThreadLocal, no other thread can ever read it —
    // that is what keeps uncommitted changes hidden from everyone else.
    private final ThreadLocal<Deque<Transaction<K, V>>> txStack =
            ThreadLocal.withInitial(ArrayDeque::new);

    private final ScheduledExecutorService janitor = Executors.newSingleThreadScheduledExecutor(r -> {
        Thread t = new Thread(r, "kv-store-janitor");
        t.setDaemon(true);
        return t;
    });

    public InMemoryKeyValueStore() {
        // Proactive TTL cleanup, so keys that are never read again do not
        // sit in memory forever after they expire.
        janitor.scheduleAtFixedRate(this::sweepExpired, 1, 1, TimeUnit.SECONDS);
    }

    // ---------- Public API ----------

    public V get(K key) {
        requireNonNullKey(key);
        Deque<Transaction<K, V>> stack = txStack.get();

        // Walk this thread's transaction stack, top (most recent) first.
        for (Transaction<K, V> tx : stack) {
            Change<V> change = tx.changes.get(key);
            if (change != null) {
                return change.type == Change.Type.DELETE ? null : change.value;
            }
        }

        // No pending change at any level — read the committed store.
        VersionedValue<V> versioned = store.get(key);
        if (versioned == null) {
            return null;
        }
        if (versioned.isExpired()) {
            store.remove(key, versioned); // remove only if still unchanged
            return null;
        }
        return versioned.value;
    }

    public void put(K key, V value) {
        put(key, value, null);
    }

    public void put(K key, V value, Long ttlMillis) {
        requireNonNullKey(key);
        Deque<Transaction<K, V>> stack = txStack.get();
        if (stack.isEmpty()) {
            store.put(key, toVersionedValue(value, ttlMillis));
        } else {
            stack.peek().changes.put(key, Change.put(value, ttlMillis));
        }
    }

    public void delete(K key) {
        requireNonNullKey(key);
        Deque<Transaction<K, V>> stack = txStack.get();
        if (stack.isEmpty()) {
            store.remove(key);
        } else {
            stack.peek().changes.put(key, Change.delete());
        }
    }

    public void begin() {
        txStack.get().push(new Transaction<>());
    }

    public void commit() {
        Deque<Transaction<K, V>> stack = txStack.get();
        Transaction<K, V> finished = popOrThrow(stack, "commit");

        if (stack.isEmpty()) {
            // Outermost transaction: apply straight to the shared store.
            // This is the moment changes become visible to other threads.
            for (Map.Entry<K, Change<V>> entry : finished.changes.entrySet()) {
                applyChange(entry.getKey(), entry.getValue());
            }
        } else {
            // Nested transaction: fold into the parent's scratch map.
            // Still invisible to other threads until the outer commit.
            stack.peek().changes.putAll(finished.changes);
        }
        cleanupIfIdle(stack);
    }

    public void rollback() {
        Deque<Transaction<K, V>> stack = txStack.get();
        popOrThrow(stack, "rollback"); // discard — no merge into parent
        cleanupIfIdle(stack);
    }

    public void shutdown() {
        janitor.shutdownNow();
    }

    // ---------- Internal helpers ----------

    private void applyChange(K key, Change<V> change) {
        if (change.type == Change.Type.DELETE) {
            store.remove(key);
        } else {
            store.put(key, toVersionedValue(change.value, change.ttlMillis));
        }
    }

    private VersionedValue<V> toVersionedValue(V value, Long ttlMillis) {
        long expiryAt = (ttlMillis == null) ? NO_EXPIRY : System.currentTimeMillis() + ttlMillis;
        return new VersionedValue<>(value, expiryAt);
    }

    private Transaction<K, V> popOrThrow(Deque<Transaction<K, V>> stack, String op) {
        if (stack.isEmpty()) {
            throw new IllegalStateException("No active transaction to " + op);
        }
        return stack.pop();
    }

    // Removes the empty per-thread Deque once it is no longer needed.
    // Matters mainly in thread pools, where threads are reused for a long
    // time — we do not want stale ThreadLocal objects piling up.
    private void cleanupIfIdle(Deque<Transaction<K, V>> stack) {
        if (stack.isEmpty()) {
            txStack.remove();
        }
    }

    private void sweepExpired() {
        long now = System.currentTimeMillis();
        for (Map.Entry<K, VersionedValue<V>> entry : store.entrySet()) {
            VersionedValue<V> v = entry.getValue();
            if (v.expiryAtMillis != NO_EXPIRY && now > v.expiryAtMillis) {
                store.remove(entry.getKey(), v); // conditional remove
            }
        }
    }

    private void requireNonNullKey(K key) {
        if (key == null) {
            throw new IllegalArgumentException("Key must not be null");
        }
    }
}
```

## How It Works

**Non-transactional path (most calls).** If a thread never calls `begin()`, its transaction stack is empty. `get`, `put`, and `delete` go straight to the `ConcurrentHashMap`. This path is as fast and safe as using `ConcurrentHashMap` directly, since that is exactly what it does.

**Opening a transaction.** `begin()` pushes a fresh, empty `Transaction` onto the calling thread's stack. From this point, `put`/`delete` on this thread write into `stack.peek().changes` — the top scratch map — instead of the shared store.

**Reading inside a transaction.** `get` walks the stack from the top down. Say a thread runs `begin(); put("x", 1); begin(); put("x", 2);`. The stack now has two levels: the inner one has `x -> PUT 2`, the outer one has `x -> PUT 1`. A `get("x")` call checks the inner level first and returns `2`. This is "read your own writes" — the transaction always sees its own pending changes, even before commit.

**Delete as a tombstone.** If a transaction calls `delete("x")` and `"x"` already has a committed value, the scratch map stores `x -> DELETE`, not simply nothing. Without this, a later `get("x")` inside the same transaction would find no entry in the scratch map and fall through to the old committed value — silently undoing the delete. The tombstone fixes this.

**Committing.** `commit()` pops the top `Transaction`. Two cases:
- **Outermost commit** (stack becomes empty): loop over the popped scratch map and apply each `Change` to the real `ConcurrentHashMap`. This is the exact instant the changes become visible to every other thread.
- **Nested commit** (a parent transaction is still on the stack): merge the popped changes into the parent's scratch map with `putAll`. Nothing reaches the shared store yet — only the outermost `commit()` does that. This makes nested transactions behave like "savepoints": committing an inner one just folds it into the outer one, which can still be rolled back as a whole.

**Rolling back.** `rollback()` pops and drops the top scratch map. No merge happens. Any parent transaction is unaffected — its own changes stay exactly as they were.

**TTL and expiry.** `put(key, value, ttlMillis)` stores `System.currentTimeMillis() + ttlMillis` as the expiry time. `get()` checks this on every read: if the current time has passed the expiry, the key is treated as missing, and it is opportunistically removed with `store.remove(key, versioned)` — a conditional remove that only succeeds if the entry has not changed since we read it (this avoids a race where we accidentally delete a newer value written by another thread a moment later). The background `janitor` thread does the same check every second for keys that are never read again, so expired entries do not sit in memory forever.

## How to Extend (Follow-ups)

**1. Make multi-key commits atomic across keys.**
As written, `commit()` applies each changed key with a separate `ConcurrentHashMap` call. Each individual `put`/`remove` is atomic, but the whole batch is not — another thread reading during the loop could see some keys already updated and others not yet. To fix this, wrap the apply loop in a `ReentrantReadWriteLock`: take the write lock during `commit()`'s apply step, and take the read lock in `get()`. This gives full "all keys in a transaction become visible together" atomicity, at the cost of some throughput, since every read now takes a lock too.

**2. Add a stronger isolation level (snapshot reads).**
Right now, a thread with no open transaction reads whatever is in the store *right now* — two reads of the same key, moments apart, can return different values if another thread committed in between. This is fine for "read committed" semantics, but not full snapshot isolation. A snapshot-read version would need to tag every value with a version number, and let a reader pin one version for its whole read sequence (similar to MVCC — Multi-Version Concurrency Control — used in real databases).

**3. Move TTL expiry off a linear scan.**
The background sweep in `sweepExpired()` is O(n) per run — it checks every key. For a store with millions of keys and only a few with TTLs, this wastes work. A better structure is a **min-heap (priority queue) ordered by expiry time**, so the janitor only looks at the few keys closest to expiring. Redis instead uses random sampling of a subset of keys with TTLs, repeated until the expired ratio drops low enough — cheaper than a full scan, at the cost of not being perfectly precise.

**4. Add a size cap with eviction.**
Combine this with the LRU (Least Recently Used) cache design: once the store reaches a maximum size, evict the least recently used key before inserting a new one. See the separate LRU Cache write-up for the doubly linked list technique.

**5. Persistence and replication (how Redis extends this idea).**
Redis is, at its core, the same idea — an in-memory map — with production concerns layered on top:
- **Persistence:** Redis periodically snapshots the whole dataset to disk (RDB), and/or appends every write command to a log file (AOF, Append-Only File) that can be replayed after a crash. Our store has neither; a process restart loses everything.
- **Eviction policies:** when memory is full, Redis picks a policy such as "evict least recently used" or "evict keys closest to expiring," instead of just rejecting new writes.
- **Replication:** Redis supports a leader-follower setup, where writes on the leader are streamed to follower copies, so a follower can take over if the leader crashes. Our single-process store has no such redundancy.

**6. Wrap transactions with a try-with-resources helper.**
A small `AutoCloseable` wrapper around `begin()`/`commit()`/`rollback()` prevents a common bug: forgetting to call `rollback()` on an exception path, which would leave a transaction open on that thread forever (and silently swallow later `put` calls into a scratch map no one ever commits).

## Complexity & Thread-Safety Notes

- **Time complexity (no open transaction):** `get`, `put`, and `delete` are O(1) average time, same as `ConcurrentHashMap` directly.
- **Time complexity (inside a transaction):** `get` is O(d), where `d` is the current transaction nesting depth (how many `begin()` calls are still open without a matching `commit`/`rollback`). In almost all real use, `d` is 1 or 2. `put`/`delete` inside a transaction are O(1), since they only touch the top scratch map.
- **Commit cost:** O(m), where `m` is the number of distinct keys changed inside that transaction — not the size of the whole store.
- **Space:** O(n) for the committed store, plus O(m) for each open transaction's scratch map, where `n` is total keys and `m` is keys touched in that transaction.
- **Thread safety of the base store:** `ConcurrentHashMap` makes every single-key `get`/`put`/`remove` safe on its own. We never manually lock this map for the non-transactional path.
- **Thread safety of transactions:** Each thread's scratch maps live behind a `ThreadLocal`, so they are private by construction — no other thread has a reference to them, and therefore no locking is needed to keep them hidden. This is what gives us "no dirty reads" almost for free.
- **What is NOT fully atomic:** a transaction that changes several keys does not become visible to other threads as one indivisible step, unless you add the lock described in the follow-ups. Say this out loud in an interview — it shows you understand the difference between "each operation is thread-safe" and "the whole transaction is atomic."
- **ThreadLocal cleanup matters in thread pools.** If threads are reused (for example, in a web server's thread pool), a `ThreadLocal` that is never cleared can quietly hold onto stale objects for the life of the pool. `cleanupIfIdle` removes the entry once a thread's stack is empty again, avoiding this leak.

## Interview Tips & Common Mistakes

- **Say "ConcurrentHashMap alone is not enough for transactions" early.** It correctly protects single-key operations, but a transaction needs to hold several changes aside and apply them as a group — that needs its own structure (the scratch map), not just a thread-safe map.
- **Do not forget the delete tombstone.** A common bug is representing "deleted inside this transaction" by just not adding an entry to the scratch map. That silently falls through to the old committed value on a later read. It must be a real marker (like our `Change.Type.DELETE`).
- **Be clear about what "committed" means for concurrency.** State plainly: other threads only ever see the shared `ConcurrentHashMap`, never a thread's private scratch map. This single sentence answers most of the isolation question interviewers are looking for.
- **Handle `commit()`/`rollback()` with no open transaction.** Throwing `IllegalStateException` is better than a silent no-op, which would hide a real bug in the caller's code.
- **Decide nested transaction behavior and say it out loud.** Some interviewers expect nested `begin()` calls to be rejected (throw an exception); we chose to support them as savepoints. Either answer is fine — just be clear and consistent.
- **Do not check TTL only in a background thread.** A lazy check inside `get()` is required too, so a just-expired key never leaks out to a reader waiting on the next sweep cycle (up to a full second late, in our design).
- **Watch for the classic TTL race:** removing an expired entry with plain `store.remove(key)` can delete a fresh value that another thread wrote a moment earlier. Use the conditional `store.remove(key, expectedValue)` instead, as shown in the solution.
- **Do not over-promise isolation.** If asked "is this fully ACID?", the honest answer is: atomicity and isolation for single-key operations, yes; full cross-key transaction atomicity visible to readers, only with the extra lock described in the follow-ups. Naming this trade-off shows maturity.
