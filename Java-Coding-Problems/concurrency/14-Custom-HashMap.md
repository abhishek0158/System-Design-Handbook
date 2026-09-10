# Custom HashMap

## Problem

Build a simplified version of Java's `HashMap` from scratch. Do not use `java.util.HashMap` inside your code. Your class must support generics, so it can store any key type `K` and any value type `V`.

Your custom map must support:

- `put(key, value)`: insert a new key, or update the value if the key already exists.
- `get(key)`: return the value for a key, or `null` if the key is not present.
- `remove(key)`: delete a key and return its old value, or `null` if it was not present.
- `containsKey(key)`: check if a key exists in the map.
- `size()`: return the number of key-value pairs stored.

The map must handle hash collisions (two different keys that land in the same bucket). It must also grow its internal array when it gets too full, so lookups stay fast.

This question tests core data structure knowledge. Interviewers ask it to check if you understand what happens "under the hood" when you call `map.put(...)` in real Java code.

## Requirements & Clarifying Questions

Ask these questions before you start coding. They show the interviewer that you think about design trade-offs, not just syntax.

1. **What key types must we support?** Any object with a correct `hashCode()` and `equals()`, not just `String` or `Integer`.
2. **Can the key be `null`?** Real `HashMap` allows one `null` key. We will support one, matching JDK behavior.
3. **Do we need to preserve insertion order?** No. Plain `HashMap` does not guarantee order. (`LinkedHashMap` is the order-preserving variant, but we will not build that here.)
4. **What load factor should trigger a resize?** The JDK default, `0.75`, which balances space use against collision rate.
5. **Do we need thread safety?** No, for the base version, matching plain `HashMap`. We discuss `ConcurrentHashMap` differences at the end.
6. **Should we implement "treeify" (turning a long collision chain into a red-black tree)?** No. We explain how and why the JDK does this, but keep our chains as simple linked lists to stay focused on the core mechanism.
7. **What is the initial capacity?** 16, like the JDK default, always kept a power of two.

## Design / Approach

### The big picture

A hash map stores data in an array called the **bucket array** (or table). Each slot is a **bucket**. To find where a key goes, we:

1. Compute `hashCode()` of the key.
2. "Mix" that hash code to spread bits better (explained below).
3. Turn the mixed hash into an array index, using `index = hash & (capacity - 1)`.
4. Go to `buckets[index]`. If it is empty, we place the new entry there. If it is not empty, another key already hashes to this same index — this is a **collision**.

### Handling collisions with chaining

When two keys map to the same bucket index, we do not overwrite one with the other. Instead, we store both entries in that bucket, linked together in a singly linked list. This technique is called **separate chaining**.

Each bucket holds a reference to the first `Node` in its chain. Each `Node` holds:

- `key`: the key object.
- `value`: the value object.
- `hash`: the cached, mixed hash code of the key.
- `next`: a reference to the next `Node` in the same bucket's chain (or `null` if it is the last one).

We cache the hash inside the node so we do not need to call `hashCode()` again during a resize or a lookup comparison. This is a small speed optimization.

When we look up a key, we walk the chain at the target bucket. For each node, we first compare cached `hash` values (a cheap integer comparison). Only if the hashes match, we call `equals()` to confirm the keys are truly the same (a more expensive comparison, especially for objects like `String`). This two-step check — hash first, then `equals()` — is the standard pattern used by the JDK.

### Why we "mix" the hash code

`key.hashCode()` can return any 32-bit integer. We turn this into a bucket index using `hash & (capacity - 1)`. This is a fast way to compute `hash % capacity`, but only when `capacity` is a power of two (explained in the next section).

The problem: this index calculation only looks at the **low bits** of the hash code. If a key's `hashCode()` produces different values only in the high bits, many different keys could collide in the same few buckets, even though their full hash codes differ.

To fix this, we "spread" the hash by mixing the high bits into the low bits, using a XOR shift:

```
mixedHash = hash ^ (hash >>> 16)
```

Here, `hash >>> 16` is an unsigned right shift by 16 bits. It moves the top half of the 32-bit hash code into the bottom half. XOR-ing this with the original hash mixes high-bit information into the low bits, without changing the top 16 bits. This is the exact trick the real JDK `HashMap` uses in its internal `hash()` method. It is cheap (one shift, one XOR) but meaningfully reduces collisions for common hash code patterns.

### Why capacity must be a power of two

We use `index = hash & (capacity - 1)` instead of `index = hash % capacity`. These two formulas give the same result, but only when `capacity` is a power of two.

Here is why. If `capacity` is 16, then `capacity - 1` is `15`, or `0000...01111` in binary. `&` (bitwise AND) with this mask keeps only the lowest 4 bits of the hash. That is mathematically the same as `hash % 16`, but `&` is a single, fast CPU instruction, while `%` (modulo) is a slower division.

If `capacity` were not a power of two — say 10 — then `capacity - 1` is `9` (binary `1001`), and `hash & 9` does **not** give the same result as `hash % 10`. So the power-of-two rule is required for the bitmask trick to be correct, not just fast.

This is also why every resize doubles the capacity (16 to 32, 32 to 64, and so on). Doubling keeps the capacity a power of two at every step.

### Why `equals()` and `hashCode()` on the key matter

The whole map depends on a contract between these two methods:

- **If two keys are equal (`a.equals(b)` is `true`), they must have the same `hashCode()`.** Otherwise, `put(a, x)` then `get(b)` might look in the wrong bucket and miss the value, even though `a` and `b` are "equal."
- **`hashCode()` should spread different keys across different buckets.** If many keys share a `hashCode()`, they all collide into one chain, and the map degrades from average O(1) lookups to O(n) linear scans.
- If a custom key class overrides `equals()` but forgets `hashCode()`, the default identity-based `hashCode()` will not match for "equal" objects, and they land in different buckets. This is one of the most common real-world bugs with custom map keys.

### Resizing (rehashing)

If we never grow the bucket array, keys pile into a fixed number of buckets, chains get longer, and lookups get slower. To keep performance close to O(1), we grow the array when it gets too full.

We track this using a **load factor**, defined as `loadFactor = size / capacity`. The JDK default is `0.75`: when the map is 75% full on average, we double the capacity. This is a trade-off: a lower load factor (like 0.5) wastes more memory but has fewer collisions; a higher one (like 0.9) saves memory but slows lookups. `0.75` is a well-tested middle ground.

When we resize: create a new array at double the old capacity; walk every old bucket's chain; recompute each node's index with the new capacity (`hash & (newCapacity - 1)`) and insert it into the new array; then replace the old array reference.

This is O(n) work, but it happens rarely (only when the map doubles), so the cost is spread ("amortized") over many cheap `put` calls. On average, `put` still costs O(1).

### How the real JDK turns a long chain into a tree (treeify)

We will not implement this part, but you should be able to explain it in an interview.

In the real `HashMap` (since Java 8), if a single bucket's chain grows to 8 or more nodes (`TREEIFY_THRESHOLD = 8`), and the table has at least 64 buckets total (`MIN_TREEIFY_CAPACITY = 64`), the JDK converts that bucket's linked list into a **red-black tree** instead.

Why does this matter? A linked list chain needs O(n) time to search. This can happen in the worst case, for example if many keys have a poor `hashCode()` and collide into one bucket — this can even be triggered on purpose, as a denial-of-service attack against systems that put untrusted input into hash maps. A red-black tree, being a balanced binary search tree, guarantees O(log n) search instead, bounding the worst case.

To sort the tree, the JDK needs a way to order keys, even though keys may not implement `Comparable`. It uses the cached hash code as the primary sort key, and falls back to identity-based tie-breaking when hashes are equal. This is why keys do not need to implement `Comparable` for treeification to work.

If the bucket later shrinks (after enough `remove` calls, down to 6 or fewer nodes — `UNTREEIFY_THRESHOLD = 6`), the JDK converts it back into a plain linked list, since a tree has more memory overhead than a list for small chains.

In an interview, it is enough to say: "the JDK treeifies buckets with 8+ collisions, when the table is large enough, to bound worst-case lookup at O(log n) instead of O(n); I keep chains as simple linked lists in my version, since a correct hash function makes long chains rare in practice."

## Java Solution

```java
import java.util.Objects;

public class CustomHashMap<K, V> {

    private static final int DEFAULT_CAPACITY = 16; // must stay a power of two
    private static final float LOAD_FACTOR = 0.75f;

    /** A single key-value entry. Buckets hold a linked list of these. */
    private static class Node<K, V> {
        final int hash;
        final K key;
        V value;
        Node<K, V> next;

        Node(int hash, K key, V value, Node<K, V> next) {
            this.hash = hash;
            this.key = key;
            this.value = value;
            this.next = next;
        }
    }

    private Node<K, V>[] buckets;
    private int size; // number of key-value pairs stored

    @SuppressWarnings("unchecked")
    public CustomHashMap() {
        this.buckets = (Node<K, V>[]) new Node[DEFAULT_CAPACITY];
    }

    /** Spreads the hash code's high bits into the low bits, to reduce collisions. */
    private static int hash(Object key) {
        if (key == null) {
            return 0;
        }
        int h = key.hashCode();
        return h ^ (h >>> 16);
    }

    /** Turns a mixed hash into a bucket index. Works only because capacity is a power of two. */
    private static int indexFor(int hash, int capacity) {
        return hash & (capacity - 1);
    }

    public V put(K key, V value) {
        int hash = hash(key);
        int index = indexFor(hash, buckets.length);

        // Walk the chain at this bucket. Update value if key already exists.
        Node<K, V> current = buckets[index];
        while (current != null) {
            if (current.hash == hash && keysEqual(current.key, key)) {
                V oldValue = current.value;
                current.value = value;
                return oldValue;
            }
            current = current.next;
        }

        // Key not found: insert a new node at the head of the chain.
        Node<K, V> newNode = new Node<>(hash, key, value, buckets[index]);
        buckets[index] = newNode;
        size++;

        // Resize if we crossed the load factor threshold.
        if (size > buckets.length * LOAD_FACTOR) {
            resize();
        }
        return null;
    }

    public V get(K key) {
        int hash = hash(key);
        int index = indexFor(hash, buckets.length);

        Node<K, V> current = buckets[index];
        while (current != null) {
            if (current.hash == hash && keysEqual(current.key, key)) {
                return current.value;
            }
            current = current.next;
        }
        return null;
    }

    public V remove(K key) {
        int hash = hash(key);
        int index = indexFor(hash, buckets.length);

        Node<K, V> current = buckets[index];
        Node<K, V> previous = null;

        while (current != null) {
            if (current.hash == hash && keysEqual(current.key, key)) {
                if (previous == null) {
                    buckets[index] = current.next; // removing the head of the chain
                } else {
                    previous.next = current.next; // unlink from the middle or tail
                }
                size--;
                return current.value;
            }
            previous = current;
            current = current.next;
        }
        return null; // key not found
    }

    public boolean containsKey(K key) {
        int hash = hash(key);
        int index = indexFor(hash, buckets.length);

        Node<K, V> current = buckets[index];
        while (current != null) {
            if (current.hash == hash && keysEqual(current.key, key)) {
                return true;
            }
            current = current.next;
        }
        return false;
    }

    public int size() {
        return size;
    }

    public boolean isEmpty() {
        return size == 0;
    }

    private boolean keysEqual(K a, K b) {
        return Objects.equals(a, b); // handles null keys safely
    }

    @SuppressWarnings("unchecked")
    private void resize() {
        Node<K, V>[] oldBuckets = buckets;
        int newCapacity = oldBuckets.length * 2; // doubling keeps capacity a power of two
        Node<K, V>[] newBuckets = (Node<K, V>[]) new Node[newCapacity];

        // Re-insert every node into the new, bigger array.
        for (Node<K, V> head : oldBuckets) {
            Node<K, V> current = head;
            while (current != null) {
                Node<K, V> next = current.next; // save next, since we reuse the node
                int newIndex = indexFor(current.hash, newCapacity);
                current.next = newBuckets[newIndex]; // insert at head of new chain
                newBuckets[newIndex] = current;
                current = next;
            }
        }
        buckets = newBuckets;
    }
}
```

## How It Works

**`put(key, value)`**: Compute the mixed hash and bucket index, then walk the chain. A matching hash and key means "update": overwrite the value and return the old one. Reaching the end without a match means "insert": create a new `Node` at the **head** of the chain (O(1), unlike walking to the tail). Then check the load factor and resize if needed.

**`get(key)`**: Same hash and index calculation. Walk the chain, comparing cached hash first, then `equals()` only on a hash match. Return the value on a match, or `null` at the end of the chain.

**`remove(key)`**: Walk the chain with a `previous` pointer one step behind `current`. On a match, unlink the node: point `buckets[index]` past it if it was the head, or `previous.next` past it otherwise. Decrement `size` and return the removed value.

**`containsKey(key)`**: Same lookup as `get`, but returns a boolean. This matters because a key can legitimately map to a stored `null` value — `get(key) == null` alone cannot tell you if the key is missing or just mapped to `null`.

**`resize()`**: Allocate a new array at double the capacity, then walk every old bucket's chain and re-insert each node using the new capacity to recompute its index. We reuse the existing `Node` objects and just relink `next` pointers — this is why we cache `hash` in the node, so `hashCode()` need not be called again.

## How to Extend (Follow-ups)

- **Iteration (`keySet()`, `values()`, `entrySet()`)**: Add an `Iterator` that walks the bucket array from index 0 to `capacity - 1`, and within each bucket, walks the chain. Real `HashMap` tracks a `modCount` field and throws if the map changes mid-iteration; consider adding the same guard.
- **Treeify long chains**: Implement the red-black tree conversion described above, for buckets that exceed 8 nodes (with a 64+ capacity table), to bound worst-case lookup at O(log n).
- **Custom load factor and initial capacity**: Add a constructor like `CustomHashMap(int initialCapacity, float loadFactor)`, rounding `initialCapacity` up to the next power of two.
- **`LinkedHashMap`-style ordering**: Add a doubly linked list across all nodes (insertion order or access order), similar to how the TTL cache problem in this series tracks LRU order.
- **Thread safety**: Wrap all methods with a single lock (simple, but slow under contention), or use bucket-level locking like `ConcurrentHashMap` does. See the notes below.
- **Null key handling variations**: Some interviewers want `null` keys explicitly rejected with an exception, instead of silently supported. Clarify this up front.

## Complexity & Thread-Safety Notes

**Time complexity**:

- `put`, `get`, `remove`, `containsKey`: average case **O(1)**, assuming a good hash function spreads keys evenly across buckets. Worst case **O(n)**, if all keys collide into a single bucket's chain (for example, keys with a broken or malicious `hashCode()`).
- `resize()`: **O(n)**, since every existing entry is re-inserted. This happens on roughly every `put` that doubles the size, so the extra cost is spread ("amortized") across many `put` calls, keeping the **amortized** cost of `put` at O(1).

**Space complexity**: **O(n)**, where `n` is the number of key-value pairs, plus the bucket array itself, which is proportional to capacity (at most about `n / 0.75` slots at any point, due to the load factor).

**Thread safety**: This `CustomHashMap` is **not thread-safe**, exactly like the real `java.util.HashMap`. If two threads call `put` at the same time, several bad things can happen:

- Both threads may read the same bucket as empty and both insert a node, causing one entry to silently overwrite or "lose" the other (a **race condition**).
- During a resize, one thread might be mid-way through relinking nodes into the new array while another thread reads the old array, seeing partial or inconsistent state.
- In older JDK versions (Java 7 and before), concurrent resizes on `HashMap` could even create a **cycle** in a bucket's linked list, causing `get()` to loop forever — an infinite loop bug that caused real production outages.

If many threads need safe access, use `java.util.concurrent.ConcurrentHashMap` instead of writing your own. Key differences:

- **Locking granularity**: `ConcurrentHashMap` (Java 8+) locks only a single bucket, using a synchronized block on the bucket's head node, instead of locking the whole map. This lets many threads write to different buckets at once. Older versions (Java 7) used a fixed set of "segments," each with its own lock (**lock striping**).
- **Reads do not block**: `get()` in `ConcurrentHashMap` takes no lock. It relies on `volatile` reads of node references, so it always sees consistent, published state without waiting for writers.
- **No `null` keys or values**: `ConcurrentHashMap` throws `NullPointerException` for a `null` key or value. In a concurrent setting, `map.get(key) == null` cannot tell you if a key is missing or genuinely holds `null`, and checking `containsKey` afterward would still race with another thread's change.
- **Resize cooperation**: Multiple threads can help move nodes from the old array to the new array together, instead of one thread doing all the work alone.
- **Atomic compound operations**: Methods like `putIfAbsent`, `computeIfAbsent`, and `merge` perform a read-then-write as one atomic step — something a plain `HashMap`, even wrapped in an external lock, cannot give you without extra care from the caller.

## Interview Tips & Common Mistakes

- **Cache the hash in the `Node`.** It avoids recomputing `hashCode()` during resize and lookups, and lets you do a cheap integer comparison before an expensive `equals()` call.
- **Always compare `hash` first, then `equals()`**, never `equals()` alone — the hash check cheaply confirms you are even in the right bucket.
- **Explain why `capacity` must be a power of two before writing `hash & (capacity - 1)`.** Interviewers often ask "why not just use `%`?" — explain the bitmask trick, its speed benefit, and why it is only correct under the power-of-two constraint.
- **Do not forget the resize threshold check after `put`.** Forgetting `resize()` entirely means the map "works" for small inputs in a demo, but degrades badly as it grows.
- **Insert new nodes at the head of the chain, not the tail.** Head insertion is O(1); walking to the tail is wasted O(n) work, since chain order does not matter here.
- **Remember `containsKey` is not the same as `get(key) != null`.** A stored `null` value would break that check.
- **Be ready to discuss a bad `hashCode()`.** If every key's `hashCode()` returns `0`, every key lands in one bucket, and the map degrades to O(n) — exactly why the JDK adds treeification as a safety net.
- **Do not over-engineer the live coding.** Get `put`, `get`, `remove`, `containsKey`, `size`, and resize correct first. Discuss treeify, custom load factors, or thread safety verbally unless asked to code them.
- **Mention the `equals()`/`hashCode()` contract proactively.** It is one of the most common causes of "my map is not finding my key" bugs, especially with custom key classes that override one method but not the other.
