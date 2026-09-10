# LFU Cache

## Problem

Design a cache with a fixed capacity. The cache stores key-value pairs.

LFU means "Least Frequently Used". When the cache is full and a new item must be added, the cache removes the item that was used the **fewest number of times**. "Used" means read (`get`) or written (`put`).

This is different from LRU ("Least Recently Used"), which removes the item used **longest ago**, no matter how many times it was used. LFU cares about *how often* an item is used. LRU cares about *how recently* an item was used.

LFU has one extra rule for ties. Suppose two items both have the lowest frequency count. Which one do we evict? The standard rule is: evict the one that is **least recently used among the tied items**. So LFU eviction is really "lowest frequency first, then least recently used as a tie-breaker."

The cache must support two operations, both in O(1) time:

- `get(key)`: return the value for the key. If the key is not there, return a signal for "not found" (for example, `null`). This operation also counts as one "use", so it increases the key's frequency by 1.
- `put(key, value)`: insert or update the key-value pair. This also counts as one "use". If the key is new and the cache is full, remove the least frequently used item first (using the tie-break rule above).

This is a common coding interview question at the 3-4 year experience level. It builds on the LRU cache question, but adds frequency as a second dimension on top of recency. It tests whether you can combine several data structures correctly, and it often leads into a discussion about real-world cache designs like Caffeine.

## Requirements & Clarifying Questions

Before coding, ask these questions. They show the interviewer that you think about edge cases, not just the happy path.

1. **What is the capacity?** Assume it is fixed at creation for the main solution.
2. **What happens on `get` for a missing key?** We return `null`, and we do **not** count a miss as a use, since nothing was actually read.
3. **Does `put` on an existing key count as a use, and bump frequency?** Yes. Updating a key's value is also a "use" of that key.
4. **What is the tie-break rule when two keys have the same frequency?** Evict the least recently used among them. State this out loud, since "LFU" alone does not define what happens on ties.
5. **What is "capacity 0"?** We will throw `IllegalArgumentException` for capacity <= 0.
6. **Is the cache accessed by multiple threads?** Ask this early. Start single-threaded, then add locking as a follow-up.
7. **Do frequencies ever decay over time?** Not in the base version. This is a common follow-up, because a key that was popular yesterday but is never touched today should not stay "safe from eviction" forever.
8. **What key and value types?** Use generics `<K, V>` so the cache works for any type.

## Design / Approach

We need to track two things at once, both in O(1) time:

- **Fast lookup by key**, and its current value and frequency count.
- **Fast lookup of "the least frequently used key, breaking ties by recency"**, so eviction never has to scan the whole cache.

A naive approach stores a frequency counter per key in a single `HashMap<K, Integer>`. On eviction, it scans all keys to find the minimum frequency. This works, but scanning is O(n) per eviction. That fails the O(1) requirement.

The standard O(1) design uses **three pieces working together**:

1. **`HashMap<K, Node<K,V>>` — `keyToNode`.** Gives O(1) lookup of a node (value + frequency) by key.
2. **`HashMap<Integer, DoublyLinkedList<K,V>>` — `freqToList`.** Maps a frequency count to a doubly linked list of *all* nodes that currently have that exact frequency. A doubly linked list is a list where each node points to both its next and previous neighbor, so removing a node from the middle takes O(1) time, given a direct reference to it.
   Inside each frequency's list, nodes are kept in **recency order**, exactly like in an LRU cache: the front of the list is the most recently used node at that frequency, and the back of the list is the least recently used node at that frequency. This is what gives us the tie-break rule for free.
3. **`minFrequency` — an integer field.** Always holds the smallest frequency count that currently has at least one node. This lets us jump straight to the list we need to evict from, without searching.

**Flow for `get(key)`:** look up the node in `keyToNode`; if missing, return `null`. Otherwise call a shared helper `touch(node)` (below), which bumps frequency and repositions the node, then return its value.

**Flow for `put(key, value)`:** if the key exists, update its value and call `touch(node)`. If the key is new and the cache is full, evict first: find the list at `minFrequency`, remove its tail node (least recently used at that frequency), and remove that key from `keyToNode`. Then create a new node with frequency 1, add it to `keyToNode` and to the front of the frequency-1 list, and set `minFrequency = 1` (a brand-new node is always the current minimum).

**The shared `touch(node)` helper**, used by `get` and by updating `put`: read the node's current frequency `f`; remove it from the list at `freqToList.get(f)`; if that list is now empty, delete it from the map, and if `f` equaled `minFrequency`, increase `minFrequency` by 1; then set the node's frequency to `f + 1` and add it to the front of the list at `freqToList.get(f + 1)`, creating that list if needed.

`minFrequency` only ever needs to go up by 1, never jump further. This is because every node's frequency increases by exactly 1 at a time, and a brand-new node always starts at frequency 1.

## Java Solution

```java
import java.util.HashMap;
import java.util.Map;

public class LFUCache<K, V> {

    // A node holds the key, value, and current frequency count.
    // It also has prev/next pointers, because it lives inside a
    // doubly linked list (the list for its current frequency).
    private static class Node<K, V> {
        K key;
        V value;
        int freq = 1;
        Node<K, V> prev;
        Node<K, V> next;

        Node(K key, V value) {
            this.key = key;
            this.value = value;
        }
    }

    // A small doubly linked list, ordered by recency.
    // Front (head.next) = most recently used node at this frequency.
    // Back (tail.prev)  = least recently used node at this frequency.
    private static class DoublyLinkedList<K, V> {
        final Node<K, V> head = new Node<>(null, null);
        final Node<K, V> tail = new Node<>(null, null);
        int size = 0;

        DoublyLinkedList() {
            head.next = tail;
            tail.prev = head;
        }

        void addFront(Node<K, V> node) {
            node.prev = head;
            node.next = head.next;
            head.next.prev = node;
            head.next = node;
            size++;
        }

        void remove(Node<K, V> node) {
            node.prev.next = node.next;
            node.next.prev = node.prev;
            size--;
        }

        Node<K, V> removeLast() {
            if (size == 0) {
                return null;
            }
            Node<K, V> last = tail.prev;
            remove(last);
            return last;
        }

        boolean isEmpty() {
            return size == 0;
        }
    }

    private final int capacity;
    private final Map<K, Node<K, V>> keyToNode;
    private final Map<Integer, DoublyLinkedList<K, V>> freqToList;
    private int minFrequency;

    public LFUCache(int capacity) {
        if (capacity <= 0) {
            throw new IllegalArgumentException("Capacity must be positive");
        }
        this.capacity = capacity;
        this.keyToNode = new HashMap<>();
        this.freqToList = new HashMap<>();
        this.minFrequency = 0;
    }

    public V get(K key) {
        Node<K, V> node = keyToNode.get(key);
        if (node == null) {
            return null;
        }
        touch(node);
        return node.value;
    }

    public void put(K key, V value) {
        if (capacity == 0) {
            return;
        }

        Node<K, V> existing = keyToNode.get(key);
        if (existing != null) {
            existing.value = value;
            touch(existing);
            return;
        }

        if (keyToNode.size() == capacity) {
            evict();
        }

        Node<K, V> newNode = new Node<>(key, value);
        keyToNode.put(key, newNode);
        freqToList.computeIfAbsent(1, f -> new DoublyLinkedList<>())
                  .addFront(newNode);
        minFrequency = 1;
    }

    // Moves a node from its current frequency list to the next one up.
    // Used by get() and by put() when updating an existing key.
    private void touch(Node<K, V> node) {
        int oldFreq = node.freq;
        DoublyLinkedList<K, V> oldList = freqToList.get(oldFreq);
        oldList.remove(node);

        if (oldList.isEmpty()) {
            freqToList.remove(oldFreq);
            if (minFrequency == oldFreq) {
                minFrequency++;
            }
        }

        node.freq = oldFreq + 1;
        freqToList.computeIfAbsent(node.freq, f -> new DoublyLinkedList<>())
                   .addFront(node);
    }

    // Removes the least frequently used node, breaking ties by
    // least recently used, and removes it from the key map too.
    private void evict() {
        DoublyLinkedList<K, V> minList = freqToList.get(minFrequency);
        Node<K, V> victim = minList.removeLast();
        if (minList.isEmpty()) {
            freqToList.remove(minFrequency);
        }
        keyToNode.remove(victim.key);
    }

    // useful for debugging and tests
    public int size() {
        return keyToNode.size();
    }
}
```

## How It Works

**`Node<K,V>`:** Stores the key, the value, and the current `freq`. Storing the key inside the node matters, just like in LRU: when we evict a node, we need its key to remove it from `keyToNode`, and at that point we only have a reference to the node, not the key.

**`DoublyLinkedList<K,V>`:** A small linked list, one instance per frequency count. It uses the same dummy `head`/`tail` trick as an LRU cache, so `addFront` and `remove` never need `null` checks. Nodes inside a list are ordered by recency, so `removeLast()` always returns the least recently used node among those sharing that frequency.

**`freqToList`:** Maps a frequency number to the list of nodes currently at that frequency. When a node's frequency changes, it moves from one list to another. When a list becomes empty, we delete its map entry, so we do not keep useless empty lists around.

**`minFrequency`:** Always points at the list we would evict from right now. It is set to 1 whenever a new node is inserted (new nodes are always the current minimum), and incremented in `touch` when the *old* minimum list empties out because its last node moved up to a higher frequency.

**`touch(node)`:** The shared step used by `get` and by updating `put`. It removes the node from its current list, bumps `freq` by 1, and reinserts the node at the front of the list for the new frequency. Reinserting at the front keeps the recency tie-break correct.

**`evict()`:** Looks up the list at `minFrequency` directly (no scanning), removes its last node, and removes that key from `keyToNode`.

**Why three structures together:** `keyToNode` gives O(1) access to a node from a key. `freqToList` plus `minFrequency` gives O(1) access to "the exact node to evict next," with no scan. Every step touches only a fixed, small number of nodes, never the whole cache.

## How to Extend (Follow-ups)

**1. Make it thread-safe.**
As with LRU, `ConcurrentHashMap` alone is not enough. A single logical operation touches *three* shared structures (`keyToNode`, `freqToList`, `minFrequency`), and all must change together as one atomic step. Wrap every public method in a single `ReentrantLock`:

```java
private final java.util.concurrent.locks.ReentrantLock lock = new java.util.concurrent.locks.ReentrantLock();

public V get(K key) {
    lock.lock();
    try {
        // same body as before
    } finally {
        lock.unlock();
    }
}
```

Do the same for `put`. This makes the cache correct, but effectively single-threaded. For higher throughput, mention **sharding**: split into N smaller LFU caches, each with its own lock, and route keys using `key.hashCode() % N`. This trades exact global LFU accuracy for much less lock contention.

**2. LRU vs LFU: how to choose.**
- LRU is simpler, and works well when recent access predicts future access ("if it was read a moment ago, it will likely be read again soon").
- LFU works better when some items stay popular over a long time, even without being touched in the last few seconds. LRU can wrongly evict a very popular item just because it was not accessed a moment ago.
- LFU's weakness: a key can build up a high count from a past burst, then become irrelevant, but still block eviction because its old count is still highest. This is "cache pollution," and it leads into aging.

**3. Aging / decay of frequencies.**
- **Periodic halving:** a background job runs every fixed interval and divides every node's frequency by 2. This needs the same lock as `get`/`put`, and it must rebuild `freqToList`, since many nodes' frequencies change at once.
- **Time-weighted frequency:** store a value combining count and recency, for example `frequency * decayFactor^(timeSinceLastAccess)`. More accurate, but more expensive.
- Aging adds fairness but adds complexity. In an interview, describing periodic halving and why it is needed is usually enough, without writing the full background-thread code.

**4. Window-TinyLFU (what Caffeine uses), at a high level.**
- Instead of an exact list per frequency, Window-TinyLFU keeps an **approximate** frequency count per key using a **Count-Min Sketch**: a probabilistic structure that estimates a count in very little memory, at the cost of sometimes overestimating (never underestimating). This avoids the memory cost of exact per-key tracking at millions of keys.
- The cache has two regions: a small **window** (a simple LRU) and a larger **main** region. New items enter the window first, so a rare, bursty key gets a chance before being judged on long-term frequency — this avoids the "one-hit wonder" problem.
- When an item is promoted from the window to the main region, TinyLFU compares its estimated frequency against the item that would be evicted from the main region; only the more frequent one survives.
- The exact design in this document is what you should produce by hand in an interview. Window-TinyLFU is what you cite when asked how a real, high-scale library solves the same problem with less memory.

**5. Other follow-ups.** Resize capacity at runtime (evict from `minFrequency` upward under the lock). Expose top-K most frequent keys by walking `freqToList` from `minFrequency` up. Combine with TTL (time to live) by storing an expiry timestamp per node and treating expired keys as misses on `get`.

## Complexity & Thread-Safety Notes

- **Time complexity:** `get` and `put` are both O(1) average time. Every step touches a fixed, small number of nodes, never a scan of the whole cache. This beats the naive "scan for the minimum frequency" approach, which is O(n) per eviction.
- **Space complexity:** O(capacity). At most `capacity` nodes live in `keyToNode`, distributed across at most `capacity` lists in `freqToList`.
- **Why `minFrequency` never needs a scan to recompute:** a node's frequency only ever increases by exactly 1 per `touch` call, and a new node always starts at 1. So `minFrequency` only ever needs to increase by 1 at a time, and it is reset to 1 on every new insert. Without this property, we would need to search for the new minimum after every eviction.
- **Thread safety:** the code above is **not** thread-safe. Two threads calling `put` at the same time can corrupt the linked list pointers, or leave `minFrequency` wrong, since three shared structures must change together atomically. Wrap every public method in a `ReentrantLock` (or `synchronized`), as shown above. This makes the cache effectively single-threaded, a reasonable trade-off unless very high concurrency is required.
- **Lock granularity trade-off:** a single lock is simple and correct but limits throughput. Sharding into several smaller LFU caches, each with its own lock, improves throughput at the cost of only approximate, per-shard LFU ordering. This is the same idea, in spirit, behind how libraries like Caffeine avoid one global lock.

## Interview Tips & Common Mistakes

- **State the tie-break rule out loud before coding.** "LFU" alone is ambiguous about ties. Say "when frequencies are equal, I evict the least recently used among them."
- **Do not forget to bump frequency on `put` for an existing key.** A common mistake is updating only the value and skipping `touch`, which silently breaks frequency tracking.
- **Do not count a `get` miss as a use.** Return early on a missing key, before touching any frequency data.
- **Remember to reset `minFrequency` to 1 on every new insert.** Forgetting this leaves `minFrequency` stuck at a stale, higher value, causing eviction to pick the wrong list or throw a `NullPointerException`.
- **Clean up empty lists from `freqToList`.** Otherwise the map slowly fills with useless empty entries.
- **Store the key inside the node**, exactly like in LRU, so `evict()` can remove the victim from `keyToNode`.
- **Test with capacity 1** and **test the tie-break case directly**: insert three keys (all frequency 1), read one of them (frequency 2), then insert a fourth key. The evicted key should be the least recently used one among the two still at frequency 1.
- **Be ready to explain why the naive "scan for minimum" approach fails O(1)**, before jumping into the three-structure design.
- **Do not over-engineer the base solution.** Write the clean three-structure version first. Only bring up aging, sharding, and Window-TinyLFU when asked, or mention them briefly at the end.
