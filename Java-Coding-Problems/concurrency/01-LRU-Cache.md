# LRU Cache

## Problem

Design a cache with a fixed capacity. The cache stores key-value pairs.

LRU means "Least Recently Used". When the cache is full and a new item must be added, the cache removes the item that was used least recently. "Used" means read (get) or written (put).

The cache must support two operations, both in O(1) time:

- `get(key)`: return the value for the key. If the key is not there, return a signal for "not found" (for example, `null` or `-1`). This operation also marks the key as "recently used".
- `put(key, value)`: insert or update the key-value pair. This also marks the key as "recently used". If the cache is full and the key is new, remove the least recently used item first.

This is a very common machine-coding interview question. It tests your knowledge of data structures, and it often leads into a discussion about thread safety.

## Requirements & Clarifying Questions

Before coding, ask these questions. They show the interviewer that you think about edge cases.

1. **What is the capacity?** Is it fixed at creation, or can it change later? (Assume fixed for the main solution.)
2. **What happens on `get` for a missing key?** Return `null`, `-1`, or throw an exception? (We will return `null`.)
3. **Does `put` on an existing key count as "use"?** Yes, in the standard version. Updating a key also makes it "recently used".
4. **What is "capacity 0"?** Usually not allowed, or it means every put evicts immediately. Handle it gracefully or throw `IllegalArgumentException`.
5. **Is the cache accessed by multiple threads?** This changes the design a lot. Ask this early. The interviewer usually says "start single-threaded, then make it thread-safe" as a follow-up.
6. **Do we need TTL (time to live) for entries?** Usually a follow-up, not the base requirement.
7. **What key and value types?** Use generics `<K, V>` so the cache works for any type.

## Design / Approach

We need two things at the same time:

- **Fast lookup by key** — this needs a hash map. A hash map gives O(1) average time for get and put by key.
- **Fast tracking of usage order** — we need to know, in O(1) time, which item was used least recently, and we need to move an item to "most recently used" in O(1) time.

A plain hash map does not track order. A plain list can track order, but finding and moving an item in a list takes O(n) time.

The classic solution combines two structures:

1. A **HashMap<K, Node<K,V>>** for O(1) lookup of the node by key.
2. A **doubly linked list** that keeps nodes in usage order. The front (head) of the list is the most recently used item. The back (tail) of the list is the least recently used item.

A doubly linked list is a list where each node has a pointer to the next node AND a pointer to the previous node. This lets us remove a node from the middle in O(1) time, without walking the list. We just need a reference to the node itself, which the hash map gives us.

**Flow for `get(key)`:**
1. Look up the node in the hash map. If not found, return `null`.
2. If found, remove the node from its current position in the list.
3. Add the node back at the front (most recently used position).
4. Return the node's value.

**Flow for `put(key, value)`:**
1. If the key already exists, update its value. Remove the node from its current position and move it to the front.
2. If the key is new:
    - If the cache is at full capacity, remove the tail node (least recently used). Also remove its key from the hash map.
    - Create a new node. Add it to the front. Add it to the hash map.

We use two dummy nodes, `head` and `tail`, as fixed markers. This avoids null checks when the list is empty or has one item. Real nodes sit between `head` and `tail`.

After the from-scratch version, we show a shorter version using Java's built-in `LinkedHashMap`. It already keeps a doubly linked list internally. Then we cover thread safety.

## Java Solution

### Version 1: HashMap + Doubly Linked List (from scratch)

```java
import java.util.HashMap;
import java.util.Map;

public class LRUCache<K, V> {

    // A node in the doubly linked list. Holds key too, so we can
    // remove it from the map when we evict the tail.
    private static class Node<K, V> {
        K key;
        V value;
        Node<K, V> prev;
        Node<K, V> next;

        Node(K key, V value) {
            this.key = key;
            this.value = value;
        }
    }

    private final int capacity;
    private final Map<K, Node<K, V>> map;
    private final Node<K, V> head; // dummy: most-recently-used side
    private final Node<K, V> tail; // dummy: least-recently-used side

    public LRUCache(int capacity) {
        if (capacity <= 0) {
            throw new IllegalArgumentException("Capacity must be positive");
        }
        this.capacity = capacity;
        this.map = new HashMap<>();
        this.head = new Node<>(null, null);
        this.tail = new Node<>(null, null);
        head.next = tail;
        tail.prev = head;
    }

    public V get(K key) {
        Node<K, V> node = map.get(key);
        if (node == null) {
            return null;
        }
        moveToFront(node);
        return node.value;
    }

    public void put(K key, V value) {
        Node<K, V> existing = map.get(key);
        if (existing != null) {
            existing.value = value;
            moveToFront(existing);
            return;
        }

        if (map.size() == capacity) {
            evictTail();
        }

        Node<K, V> newNode = new Node<>(key, value);
        map.put(key, newNode);
        addToFront(newNode);
    }

    // --- helper methods on the linked list ---

    private void addToFront(Node<K, V> node) {
        node.prev = head;
        node.next = head.next;
        head.next.prev = node;
        head.next = node;
    }

    private void remove(Node<K, V> node) {
        node.prev.next = node.next;
        node.next.prev = node.prev;
    }

    private void moveToFront(Node<K, V> node) {
        remove(node);
        addToFront(node);
    }

    private void evictTail() {
        Node<K, V> lru = tail.prev;
        remove(lru);
        map.remove(lru.key);
    }

    // useful for debugging and tests
    public int size() {
        return map.size();
    }
}
```

### Version 2: LinkedHashMap (short version)

Java's `LinkedHashMap` already keeps entries in a linked list. If you pass `accessOrder = true` in its constructor, it reorders entries on every `get` and `put`, moving the touched entry to the end (most recently used). We only need to override `removeEldestEntry` to trigger eviction.

```java
import java.util.LinkedHashMap;
import java.util.Map;

public class LRUCacheSimple<K, V> extends LinkedHashMap<K, V> {

    private final int capacity;

    public LRUCacheSimple(int capacity) {
        // initialCapacity, loadFactor, accessOrder=true (order by access, not insertion)
        super(16, 0.75f, true);
        this.capacity = capacity;
    }

    @Override
    protected boolean removeEldestEntry(Map.Entry<K, V> eldest) {
        // Called automatically after each put(). Returning true removes
        // the eldest (least recently used) entry.
        return size() > capacity;
    }

    // get() and put() are inherited directly from LinkedHashMap.
    // No override needed — accessOrder=true already handles reordering.
}
```

This version is much shorter. In an interview, mention it as an alternative, but be ready to write Version 1 by hand. Interviewers often ask for the "from scratch" version to test your understanding of data structures.

### Version 3: Thread-Safe LRU Cache

A plain `HashMap` is not thread-safe. `ConcurrentHashMap` is thread-safe for individual operations, but it does **not** help here. The reason: our cache needs two data structures (map + linked list) to change together, as one atomic step. `ConcurrentHashMap` only makes the map itself safe. It says nothing about the linked list, and it cannot make "update map AND update list" a single atomic action. Two threads could both pass the map check, then corrupt the linked list pointers at the same time.

The simplest correct fix is a single lock (`ReentrantLock` or `synchronized`) around every operation. This means only one thread accesses the cache at a time.

```java
import java.util.HashMap;
import java.util.Map;
import java.util.concurrent.locks.ReentrantLock;

public class ThreadSafeLRUCache<K, V> {

    private static class Node<K, V> {
        K key;
        V value;
        Node<K, V> prev;
        Node<K, V> next;

        Node(K key, V value) {
            this.key = key;
            this.value = value;
        }
    }

    private final int capacity;
    private final Map<K, Node<K, V>> map;
    private final Node<K, V> head;
    private final Node<K, V> tail;
    private final ReentrantLock lock = new ReentrantLock();

    public ThreadSafeLRUCache(int capacity) {
        if (capacity <= 0) {
            throw new IllegalArgumentException("Capacity must be positive");
        }
        this.capacity = capacity;
        this.map = new HashMap<>();
        this.head = new Node<>(null, null);
        this.tail = new Node<>(null, null);
        head.next = tail;
        tail.prev = head;
    }

    public V get(K key) {
        lock.lock();
        try {
            Node<K, V> node = map.get(key);
            if (node == null) {
                return null;
            }
            moveToFront(node);
            return node.value;
        } finally {
            lock.unlock();
        }
    }

    public void put(K key, V value) {
        lock.lock();
        try {
            Node<K, V> existing = map.get(key);
            if (existing != null) {
                existing.value = value;
                moveToFront(existing);
                return;
            }
            if (map.size() == capacity) {
                evictTail();
            }
            Node<K, V> newNode = new Node<>(key, value);
            map.put(key, newNode);
            addToFront(newNode);
        } finally {
            lock.unlock();
        }
    }

    private void addToFront(Node<K, V> node) {
        node.prev = head;
        node.next = head.next;
        head.next.prev = node;
        head.next = node;
    }

    private void remove(Node<K, V> node) {
        node.prev.next = node.next;
        node.next.prev = node.prev;
    }

    private void moveToFront(Node<K, V> node) {
        remove(node);
        addToFront(node);
    }

    private void evictTail() {
        Node<K, V> lru = tail.prev;
        remove(lru);
        map.remove(lru.key);
    }
}
```

A simpler alternative: wrap Version 2 (`LinkedHashMap`) using `Collections.synchronizedMap`, or just add `synchronized` to a small wrapper class. This is less code but has the same "one thread at a time" limit. For a read-heavy cache, you can improve throughput with a `ReadWriteLock`, but note: in LRU, even a `get` writes to the linked list (it moves the node to the front). So a plain read-write lock does not help much here, unless you use tricks like a "lock-free read path with a background reorder queue" (advanced, not expected at this level).

## How It Works

**Doubly linked list node:** Each node stores a key, a value, and two pointers (`prev`, `next`). Storing the key inside the node matters — when we evict the tail node, we need its key to remove it from the map too.

**Dummy head and tail:** These are placeholder nodes with no real data. `head.next` always points to the most recently used real node. `tail.prev` always points to the least recently used real node. Using dummy nodes means `addToFront` and `remove` never need to check for `null`, because there is always a node on each side.

**`addToFront(node)`:** Inserts a node right after `head`. This is the "most recently used" position.

**`remove(node)`:** Unlinks a node from wherever it is in the list, by connecting its neighbors to each other. This works in O(1) time because we already have a direct reference to the node (from the hash map). We do not need to search for it.

**`moveToFront(node)`:** Just `remove` followed by `addToFront`. Used both in `get` (mark as used) and in `put` (update existing key).

**`evictTail()`:** Removes the node just before the dummy `tail`. This is the least recently used real node. We remove it from the list and from the map.

**Why HashMap + List together:** The HashMap gives O(1) lookup by key. The list gives O(1) reordering and O(1) eviction, because we always have a direct node reference, never needing to scan.

## How to Extend (Follow-ups)

**1. Make it thread-safe.**
Covered above (Version 3). Key points to say out loud in an interview:
- `ConcurrentHashMap` alone is not enough, because get/put on our cache touch two structures (map and list) and both must change together, as one atomic unit.
- A single lock around each public method is the simplest correct fix.
- For higher throughput, consider **sharding**: split the cache into N smaller LRU caches, each with its own lock, and pick the shard using `key.hashCode() % N`. This reduces lock contention, but the eviction is now "least recently used within a shard", not globally exact. This trade-off is worth mentioning.

**2. Add TTL (time to live).**
TTL means each entry expires after a fixed time, even if capacity is not full.
- Store an `expiryTime` (for example, `System.currentTimeMillis() + ttlMillis`) inside each `Node`.
- On `get`, check if the entry is expired. If yes, remove it and return `null`, as if it were never found.
- For proactive cleanup (not just on access), add a background thread using `ScheduledExecutorService` that scans and removes expired entries every few seconds. Mention that this scan needs the same lock, to avoid corrupting the list while scanning.
- Alternative: keep a separate min-heap ordered by expiry time, for O(log n) "find next expiring entry" instead of scanning everything.

**3. Turn it into an LFU cache (Least Frequently Used).**
LFU evicts the item used the **fewest number of times**, not the one used longest ago.
- Naive approach: store a counter (`frequency`) per key. On eviction, scan for the minimum. This is O(n) per eviction — too slow for the O(1) requirement.
- O(1) approach (the standard LFU design):
    - Keep a `HashMap<K, Node>` for key lookup, same as before.
    - Keep a `HashMap<Integer, DoublyLinkedList>` that maps a frequency count to a doubly linked list of all keys with that frequency. Within each list, order by recency (like our LRU list), to break ties.
    - Track `minFrequency`, the smallest frequency currently in use.
    - On `get` or `put` (update), move the node from its current frequency list to the `frequency + 1` list. Update `minFrequency` if the old list becomes empty and it was the minimum.
    - On eviction, remove the tail node from the list at `minFrequency`.
    - This gets true O(1) get and put, but the code is noticeably more complex than LRU. Mention this complexity trade-off if asked to implement it live — many interviewers accept a clear design explanation plus partial code, given time limits.

**4. Other common follow-ups:**
- **Resize capacity at runtime:** add a `setCapacity(int newCapacity)` method that evicts from the tail until size fits, under the same lock.
- **Bulk operations / iteration:** if you need to iterate all entries (for example, for debugging), decide whether iteration order should be insertion order, access order, or does not matter. Also decide if iteration needs to hold the lock for its full duration (usually yes, to avoid seeing a half-updated list).
- **Persistence:** periodically snapshot the cache to disk. This needs careful thought about consistency — usually done by copying entries under the lock, then writing the copy without holding the lock.

## Complexity & Thread-Safety Notes

- **Time complexity:** `get` and `put` are both O(1) average time, for all three versions. The HashMap lookup is O(1) average (it can degrade to O(n) in the rare case of many hash collisions, but this is not something to worry about in an interview). The linked list operations (`addToFront`, `remove`, `moveToFront`) are always O(1), because we never search the list — we always have a direct node reference from the map.
- **Space complexity:** O(capacity). We store at most `capacity` nodes, plus the same number of map entries.
- **Version 1 vs Version 2:** Both have the same time and space complexity. Version 1 (from scratch) shows you understand the data structure. Version 2 (`LinkedHashMap`) is production-ready and shorter, good to mention as "what I'd actually use in real code, unless I need custom eviction logic."
- **Thread safety:** Version 1 and Version 2, as shown, are **not** thread-safe. Multiple threads calling `put` at the same time can corrupt the linked list pointers (for example, two threads both reading `tail.prev` before either updates it, causing lost nodes or broken links). Version 3 fixes this with a `ReentrantLock`, making the whole cache single-threaded in effect — correct, but not highly concurrent.
- **Lock granularity trade-off:** A single lock is simple and correct, but it limits throughput under heavy concurrent load, since only one thread can use the cache at any moment. Sharding (multiple smaller caches, each with its own lock) improves throughput but loses perfectly exact global LRU order — eviction only sees "least recently used within its shard." This is often an acceptable trade-off in real systems (for example, how Guava's `Cache` and Caffeine handle sharding internally).

## Interview Tips & Common Mistakes

- **Do not forget the dummy head and tail nodes.** Without them, your code needs many `null` checks for the empty-list and single-node cases. This is a common source of bugs under interview time pressure.
- **Store the key inside the Node.** Many candidates forget this, then get stuck when they need to remove the evicted node's key from the map — they only have the node's value, not its key.
- **Update value AND move to front on `put` for an existing key.** A common mistake is only updating the value and forgetting to also mark it as recently used.
- **Decide and state your "not found" behavior early** (`null`, `-1`, `Optional`, or exception). Saying this out loud shows attention to API design.
- **Say out loud why `ConcurrentHashMap` is not enough**, even if not asked. It shows you understand that atomicity must cover the whole operation (map + list), not just individual map calls.
- **Test with capacity 1.** This is a common edge case: every `put` after the first should evict the previous entry immediately.
- **Do not over-engineer the base solution.** Write the clean HashMap + doubly linked list version first. Only bring up sharding, TTL, and LFU when asked, or briefly mention them at the end as "things I would add given more time."
- **Watch for off-by-one errors in `removeEldestEntry`** (Version 2). It should return `size() > capacity`, not `size() >= capacity`, since it is called after the new entry is already added.
- **Mention `synchronized` methods as a simpler, if less flexible, alternative to `ReentrantLock`** — some interviewers prefer to see you know both, and know when a plain `synchronized` block is "good enough."
