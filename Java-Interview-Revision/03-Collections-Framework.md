# 3. Collections Framework

The Collections Framework gives ready-made data structures — lists, sets, maps, queues — so you don't write them from scratch. At 3–4 years, interviewers care most about `HashMap` internals, thread-safety choices, and time complexity trade-offs, not just method names.

## Key Concepts (Quick Revision)

**Hierarchy**
- `Collection` is the root interface for groups of objects. It has three main children: `List`, `Set`, `Queue`.
- `Map` is **not** a `Collection` — it stores key-value pairs, a different shape of data.
- `List` — ordered, allows duplicates (`ArrayList`, `LinkedList`, `Vector`).
- `Set` — no duplicates (`HashSet`, `LinkedHashSet`, `TreeSet`).
- `Queue` — designed for holding items before processing (`LinkedList`, `PriorityQueue`, `ArrayDeque`).
- `Map` — key-value store, keys unique (`HashMap`, `LinkedHashMap`, `TreeMap`, `Hashtable`, `ConcurrentHashMap`).

**ArrayList vs LinkedList**
- `ArrayList` — backed by a resizable array (`Object[]`). Fast random access by index: O(1). Insert/delete in the middle: O(n), because elements shift.
- `LinkedList` — backed by a doubly linked list (nodes with `prev`/`next` pointers). Random access by index: O(n), must walk from the start or end. Insert/delete at a known node (e.g., via iterator): O(1), because it's just pointer changes.
- When to use: `ArrayList` for most cases — reads dominate, memory is more compact (no per-node pointer overhead). `LinkedList` only if you do many insert/delete operations in the middle, and rarely need random access. In practice, `ArrayDeque` often replaces `LinkedList` even for queue/stack use, because it's faster and uses less memory.

**Vector**
- `Vector` is like `ArrayList`, but every method is `synchronized`. This is legacy from Java 1.0, before the Collections Framework existed.
- It is rarely used today because:
    - Synchronizing every single method (even `get()`) is slow, even in single-threaded code.
    - Locking one method at a time does not make compound actions (like "check then add") thread-safe anyway — you still need external locking for that.
    - `ArrayList` + explicit locking, or `CopyOnWriteArrayList`, or `Collections.synchronizedList()` are better modern choices.

**HashMap internals (core topic — expect deep questions)**
- Backing structure: an array of "buckets" — `Node<K,V>[] table`. Each bucket can hold a linked list (or a tree) of entries that landed in the same slot.
- To find a bucket for a key: `hash(key)` is computed, then `index = hash & (n - 1)`, where `n` is the array length (table size). This is equivalent to `hash % n` but faster, and it only works correctly because `n` is always a power of 2.
- `hashCode()` decides which bucket a key goes to. `equals()` decides if two keys in the same bucket are actually the same key (used to detect duplicates and to fetch the right value).
- Collision — two different keys can land in the same bucket (same index) if their hashes collide. Before Java 8, collisions were handled purely as a linked list on that bucket — O(n) lookup within a bad bucket in the worst case.
- Java 8 change: if a single bucket's linked list grows to more than 8 entries (`TREEIFY_THRESHOLD = 8`), **and** the table has at least 64 buckets, that bucket is converted into a **red-black tree** instead of a linked list. This changes worst-case lookup in that bucket from O(n) to O(log n). If entries are later removed and the bucket shrinks below 6 (`UNTREEIFY_THRESHOLD`), it converts back to a linked list.
- Load factor — default is `0.75`. It means: when the map is 75% full (`size > capacity * loadFactor`), it resizes (doubles capacity). This is a trade-off between space (lower load factor = more empty buckets, more memory) and time (higher load factor = more collisions, slower lookups).
- Resizing/rehashing — when the table doubles in size, every existing entry may move to a new bucket, because `index = hash & (n-1)` changes when `n` changes. Java 8 optimized this: since capacity always doubles, an entry either stays at the same index or moves to `oldIndex + oldCapacity`, so no full rehash of the hash value is needed — just a bit check.
- Why capacity is a power of 2: it makes `hash & (n-1)` work exactly like `hash % n`, but with a fast bitwise AND instead of a slower division/modulo operation. It also makes the Java 8 resize trick above possible.
- Default initial capacity: 16. Default load factor: 0.75. So resize happens by default once size exceeds 12.

**HashSet**
- `HashSet` is backed internally by a `HashMap`. Every element you add becomes a **key** in that map, with a constant dummy object as the value.
- This is why `HashSet` needs `hashCode()`/`equals()` correctly implemented on elements, just like `HashMap` keys — for the same reasons (bucket + duplicate detection).

**LinkedHashMap**
- Extends `HashMap`, but also keeps a doubly linked list running through all entries, to preserve order.
- Two modes: **insertion order** (default — order elements were first put) and **access order** (`new LinkedHashMap<>(cap, loadFactor, true)` — every `get()`/`put()` moves the entry to the end, most-recently-used last).
- Access-order mode + overriding `removeEldestEntry()` gives you an easy **LRU cache** in a few lines, because the least-recently-used entry is always at the front (head) of the linked list.

**TreeMap / TreeSet**
- Backed by a **red-black tree** (a self-balancing binary search tree). Keys are always kept sorted.
- Operations (`get`, `put`, `remove`, `containsKey`) are O(log n), because the tree stays balanced (height stays around log n).
- Sorting uses natural ordering (`Comparable`) by default, or a custom `Comparator` passed to the constructor.
- Extra features `HashMap` doesn't have: `firstKey()`, `lastKey()`, `headMap()`, `tailMap()`, `ceilingKey()`, `floorKey()` — all rely on the sorted structure.

**ConcurrentHashMap (CHM)**
- Goal: allow safe concurrent reads and writes without locking the whole map, unlike `Hashtable` or `Collections.synchronizedMap()`.
- Pre-Java 8: used **segment-level locking** — the map was split into a fixed number of segments (default 16), each with its own lock. Two threads writing to different segments did not block each other.
- Java 8 redesign: segments were removed. Now locking happens at the **bucket (bin) level**:
    - Reads (`get`) are mostly **lock-free**, using `volatile` reads and `Unsafe`/`VarHandle` memory access — no lock needed at all for a simple read.
    - Writes use **CAS (Compare-And-Swap)** to insert the first node into an empty bucket, with no lock needed.
    - If the bucket already has a node (collision, i.e., not empty), the write **synchronizes on that specific bucket's first node** — only that one bucket is locked, not the whole map, not even a segment.
    - Like `HashMap`, buckets with more than 8 entries treeify into red-black trees for better worst-case lookup.
- Resizing is also done cooperatively: multiple threads can help move nodes to the new table (transfer) at the same time.
- Why it beats `Hashtable`: `Hashtable` locks the **entire map** on every operation (one global lock), so only one thread can read or write at any time — a serious bottleneck under concurrency. CHM allows many threads to work on different buckets in parallel.
- `ConcurrentHashMap` does not allow `null` keys or `null` values (unlike `HashMap`). This is intentional — in a concurrent map, `map.get(key) == null` cannot tell you if the key is absent or the value is actually `null`, which is ambiguous under concurrent modification.

**Hashtable vs HashMap**
| | `Hashtable` | `HashMap` |
|---|---|---|
| Thread-safety | Synchronized (whole-map lock) | Not thread-safe |
| Null keys/values | Not allowed | One null key, many null values allowed |
| Performance | Slower (always locks) | Faster (single-threaded use) |
| Legacy | Yes (Java 1.0) | Modern (Java 1.2, Collections Framework) |
| Iterator | Fail-fast (mostly), but has legacy `Enumeration` too | Fail-fast |

**Fail-fast vs fail-safe iterators**
- **Fail-fast** — iterators of `ArrayList`, `HashMap`, `HashSet`, etc. They track a `modCount` (modification count) on the collection. Every structural change (add/remove) increments it. The iterator checks `modCount` against its own saved copy on each `next()` call — if they don't match, it throws `ConcurrentModificationException` immediately ("fails fast"), instead of giving wrong results silently.
- **Fail-safe** — iterators of `CopyOnWriteArrayList`, `ConcurrentHashMap`. They iterate over a snapshot (a separate copy of the data, or a structure designed for safe concurrent traversal) and do not throw `ConcurrentModificationException`, even if the collection changes during iteration. Trade-off: the iterator may not reflect the very latest changes (a form of eventual/weak consistency), and snapshot-based versions use more memory.

**Comparable vs Comparator**
- `Comparable` — the class itself defines its natural, default order via `compareTo()`. Only one ordering possible per class. Example: `String` implements `Comparable` (alphabetical order).
- `Comparator` — an external object that defines an order, separate from the class. You can have many different `Comparator`s for the same class (sort by name, then by age, then by salary).
- Java 8 chaining: `Comparator.comparing(Person::getAge).thenComparing(Person::getName).reversed()` — clean, readable, composable sorting instead of writing a manual `compareTo` with many `if` checks.

**Iterator vs ListIterator**
- `Iterator` — works on any `Collection`. Supports `hasNext()`, `next()`, `remove()`. Forward-only.
- `ListIterator` — works only on `List`. Extends `Iterator` with `hasPrevious()`, `previous()`, `set()` (replace last returned element), `add()`, and index positions (`nextIndex()`, `previousIndex()`). Can move in both directions.

**Time complexity and thread-safety summary**

| Structure | Get/Contains | Insert | Delete | Ordered? | Thread-safe? |
|---|---|---|---|---|---|
| `ArrayList` | O(1) index, O(n) search | O(1) amortized end, O(n) middle | O(n) | Insertion order | No |
| `LinkedList` | O(n) | O(1) at known node | O(1) at known node | Insertion order | No |
| `HashMap` / `HashSet` | O(1) avg, O(log n) worst (treeified) | O(1) avg | O(1) avg | No order | No |
| `LinkedHashMap` | O(1) avg | O(1) avg | O(1) avg | Insertion/access order | No |
| `TreeMap` / `TreeSet` | O(log n) | O(log n) | O(log n) | Sorted | No |
| `Hashtable` | O(1) avg | O(1) avg | O(1) avg | No order | Yes (whole-map lock) |
| `ConcurrentHashMap` | O(1) avg, lock-free read | O(1) avg, bucket-level lock/CAS | O(1) avg | No order | Yes (fine-grained) |
| `CopyOnWriteArrayList` | O(1) index | O(n) (copies array) | O(n) (copies array) | Insertion order | Yes (snapshot-based) |

**How to choose the right collection**
- Need index-based fast access, few middle inserts → `ArrayList`.
- Need many inserts/deletes at ends or via iterator, rarely by index → `LinkedList` or `ArrayDeque`.
- Need fast lookup, no duplicates, order doesn't matter → `HashSet` / `HashMap`.
- Need lookup **and** insertion order preserved, or building an LRU cache → `LinkedHashMap`.
- Need sorted keys, or range queries (`headMap`, `tailMap`) → `TreeMap` / `TreeSet`.
- Need thread-safe map under heavy concurrency → `ConcurrentHashMap` (never `Hashtable`, rarely `synchronizedMap`).
- Need thread-safe list with rare writes, many reads (e.g., listener lists) → `CopyOnWriteArrayList`.
- Need FIFO/LIFO or priority-based processing → `ArrayDeque` (stack/queue) or `PriorityQueue` (priority order via heap).

## Important Interview Questions

**Q1: Walk through what happens internally when you call `map.put(key, value)` on a `HashMap`.**
1. Compute `hash(key)` — `HashMap` applies a supplemental hash function that XORs the higher bits of `key.hashCode()` into the lower bits, to spread out hash codes that differ mainly in high bits, so lower-order bits (used for indexing) are better distributed.
2. Compute bucket index: `index = hash & (n - 1)`, where `n` is the current table length.
3. If the bucket is empty, place a new `Node` there.
4. If not empty, walk the bucket's linked list (or tree). For each existing node, check `hash` first (cheap), then `equals()` (more expensive) to see if the key already exists. If found, replace the value. If not found, append a new node at the end.
5. If the bucket's list is now more than 8 nodes long, and the table has at least 64 buckets, convert that bucket to a red-black tree.
6. After insert, check if `size > capacity * loadFactor`. If so, resize (double capacity) and rehash/redistribute entries.
   *Follow-up: Why check `hash` before `equals()`?* Comparing `int` hash values is much cheaper than calling a potentially complex `equals()` method, so it's a fast pre-filter.

**Q2: Why does `HashMap` need both `hashCode()` and `equals()` to be overridden correctly for custom key objects?**
`hashCode()` decides which bucket the key lands in — without a good one, keys that should be equal might get different hash codes and never be found. `equals()` decides which entry inside that bucket actually matches. If you override `equals()` but not `hashCode()` (or vice versa), you break the contract: **equal objects must have equal hash codes.** A common bug: two objects are `equals()`-equal, but have different `hashCode()`s, so they land in different buckets, and `map.get()` never finds the second one even though logically it "equals" a stored key.
*Follow-up: What is the hashCode/equals contract?* If `a.equals(b)` is true, then `a.hashCode() == b.hashCode()` must be true. The reverse is not required — two unequal objects can share the same hash code (this is a normal collision, not a bug).

**Q3: Why is the default `HashMap` capacity a power of 2, and why does resizing double the capacity?**
Power-of-2 capacity lets the JVM compute `index = hash & (n - 1)` instead of `hash % n`. Bitwise AND is much faster than modulo/division. It only gives a correct, uniform distribution when `n` is a power of 2 — with any other `n`, `hash & (n-1)` would not evenly cover the range and would cause more collisions.
Doubling on resize keeps this power-of-2 property. It also enables a neat Java 8 optimization: since capacity always doubles, when a table resizes from size `n` to `2n`, each old entry either stays at the same index in the new table, or moves to `index + n` — determined by one extra bit of the hash. This avoids fully recomputing hashes for every entry during resize.

**Q4: How does Java 8's treeify change help, and what is the trade-off?**
Before Java 8, a bucket with many collisions degraded to a linked list — lookup in that bucket was O(n). In rare cases (many keys engineered to collide, sometimes called a hash-flooding attack), this could make a `HashMap` extremely slow, close to O(n) for every `get()`.
Java 8 converts a bucket to a red-black tree once it exceeds 8 entries (and the table has ≥ 64 buckets — if the table is small, it just resizes instead of treeifying, because resizing is often enough to fix small tables). This bounds worst-case lookup in that bucket to O(log n).
Trade-off: a tree node is heavier in memory than a plain linked-list node (it needs parent/left/right/color pointers), and treeify/untreeify itself has a cost. It's rarely triggered in normal use with a good `hashCode()` — it's mainly a safety net for pathological cases.

**Q5: Explain `ConcurrentHashMap`'s locking model in Java 8, and why it's faster than `Hashtable`.**
`Hashtable` locks the entire map object on every method call — `get`, `put`, `size`, all serialize through one lock, so only one thread can touch the map at any time, even for reads.
`ConcurrentHashMap` (Java 8+) has no map-wide lock:
- Reads are lock-free — they read `volatile` fields directly.
- The first insert into an empty bucket uses CAS, no lock at all.
- Only when a bucket already has entries (a real collision) does the thread `synchronized`-lock that single bucket's head node, blocking only other writers to that same bucket — reads and writes to other buckets proceed freely.
  This design means throughput scales much better with more threads, since contention is limited to individual buckets instead of the whole structure.
  *Follow-up: Is `size()` on `ConcurrentHashMap` perfectly accurate?* Not guaranteed at the exact instant it's called under concurrent modification — it's a best-effort estimate (using internal counters), consistent with the map being under concurrent updates. For most practical purposes it's accurate enough.

**Q6: What causes `ConcurrentModificationException`, and how do you safely remove items while iterating?**
It's thrown by fail-fast iterators (`ArrayList`, `HashMap`, etc.) when the collection is structurally modified outside the iterator during iteration. Internally, the collection keeps a `modCount` counter, incremented on structural change. The iterator saves `expectedModCount` when created, and checks it matches the live `modCount` on every `next()` call — mismatch throws the exception right away.
Safe ways to remove during iteration:
- Use `iterator.remove()` — this updates `modCount` and the iterator's expected count together, so no mismatch.
- Use `list.removeIf(condition)` — handles this internally.
- Iterate over a copy, and modify the original.
- Use `CopyOnWriteArrayList` if concurrent, low-write use case fits.

**Q7: Why does `LinkedHashMap` work well for building an LRU cache?**
`LinkedHashMap` keeps entries in a doubly linked list, in either insertion order or access order. In access-order mode (constructor flag `true`), every `get()` or `put()` moves that entry to the end of the list (most recently used). The least recently used entry naturally sits at the head.
Override `removeEldestEntry(Map.Entry eldest)` to return `true` once size exceeds your cache limit — `LinkedHashMap` then automatically removes the head entry after every insert.
```java
class LRUCache<K, V> extends LinkedHashMap<K, V> {
    private final int capacity;
    LRUCache(int capacity) {
        super(16, 0.75f, true); // true = access order
        this.capacity = capacity;
    }
    protected boolean removeEldestEntry(Map.Entry<K, V> eldest) {
        return size() > capacity;
    }
}
```
This gives O(1) get/put with automatic eviction, in a handful of lines.

**Q8: Compare `ArrayList` and `LinkedList` with real numbers, and explain when `LinkedList` actually helps.**
- `ArrayList.get(i)`: O(1) — direct array index.
- `LinkedList.get(i)`: O(n) — must traverse from head or tail (whichever is closer).
- `ArrayList.add(item)` at the end: O(1) amortized (occasionally O(n) when the backing array must grow and copy).
- `ArrayList.add(index, item)` in the middle: O(n) — shifts all elements after it.
- `LinkedList.add`/`remove` **at a node you already have** (via a `ListIterator`): O(1) — pointer relinking only.
  `LinkedList` only wins when you already hold a reference/iterator to the exact node, and do lots of inserts/removes there without needing random access. In most real code, that pattern is rare — which is why `ArrayList` (or `ArrayDeque` for queue/stack use) is almost always the default choice.

**Q9: Why is `TreeMap` O(log n) and not O(1) like `HashMap`? When would you choose it over `HashMap`?**
`TreeMap` is a red-black tree, a balanced binary search tree — every operation walks down the tree height, which stays around `log n` because the tree self-balances (rotates) on insert/delete. It trades speed for **order**.
Choose `TreeMap`/`TreeSet` when you need:
- Sorted iteration order (no manual sort step needed later).
- Range queries: `headMap(key)`, `tailMap(key)`, `subMap(from, to)`.
- Nearest-match lookups: `floorKey()`, `ceilingKey()`, `higherKey()`, `lowerKey()`.
  If you don't need order, `HashMap` is faster for plain lookups.

**Q10: What's the difference between `Comparable` and `Comparator`? Show Java 8-style chaining.**
`Comparable.compareTo()` is implemented inside the class itself — it defines the class's single "natural" order (e.g., `Integer` sorts numerically, `String` sorts lexicographically). `Comparator.compare()` is a separate object, so you can define as many different orderings as you want, without touching the original class.
```java
List<Person> people = ...;
people.sort(
    Comparator.comparing(Person::getLastName)
              .thenComparing(Person::getFirstName)
              .thenComparingInt(Person::getAge)
              .reversed()
);
```
This reads top to bottom: sort by last name, break ties by first name, then by age, then reverse the whole result.
*Follow-up: Can you sort `null`-safe?* Yes — `Comparator.nullsFirst(Comparator.naturalOrder())` handles `null` values without a `NullPointerException`.

**Q11: What is the difference between `Iterator` and `ListIterator`? When do you need `ListIterator`?**
`Iterator` only moves forward and supports `remove()`. `ListIterator` (available only on `List` types) can move both forward and backward, supports `set()` to replace the last returned element in place, `add()` to insert during iteration, and exposes the current index via `nextIndex()`/`previousIndex()`.
You need `ListIterator` when you must modify elements in place while iterating (not just remove), or need to traverse backward, or need index information mid-iteration — plain `Iterator` cannot do any of these.

**Q12: How does `HashMap` behave differently in a multi-threaded environment without synchronization? Why is this dangerous?**
`HashMap` is not thread-safe. Under concurrent modification without external synchronization, several bad things can happen:
- Silent data loss or corrupted entries.
- In older Java versions (pre-8), concurrent resize could create a **cyclic linked list** inside a bucket, causing an infinite loop and 100% CPU usage on a later `get()` — a well-known production bug.
- `ConcurrentModificationException` if a fail-fast iterator is active while another thread writes.
  Fix: use `ConcurrentHashMap` for concurrent access, or wrap with `Collections.synchronizedMap()` (coarse locking, and you must manually synchronize when iterating), or use external locking around compound operations.

## FAQ / Rapid-Fire

- **Q: What is the default capacity and load factor of `HashMap`?** Capacity 16, load factor 0.75. Resize threshold is 12 (16 × 0.75).
- **Q: Can a `HashMap` have a `null` key?** Yes, exactly one. It's stored at bucket index 0. Values can have multiple `null`s.
- **Q: Can `ConcurrentHashMap` have `null` keys or values?** No — neither, to avoid ambiguity between "not present" and "value is null" under concurrency.
- **Q: What's the difference between `size()` and `capacity` in a `HashMap`?** `size()` is the number of entries stored. Capacity is the length of the internal bucket array — always a power of 2.
- **Q: Is `HashMap` iteration order guaranteed?** No, it can even change between runs or after a resize. Use `LinkedHashMap` if you need predictable order.
- **Q: What is `TREEIFY_THRESHOLD`?** 8 — the bucket size at which a linked list converts to a red-black tree (only if table has ≥ 64 buckets).
- **Q: What is `UNTREEIFY_THRESHOLD`?** 6 — if a treeified bucket shrinks to 6 or fewer entries (via removal), it converts back to a linked list.
- **Q: Why 8 and not some other number for treeify?** JDK authors chose it based on statistical analysis — with a good hash function, the chance of a bucket reaching 8 entries under normal use is extremely low (Poisson distribution), so treeifying is a rare safety net, not a common path.
- **Q: Difference between `Set` and `List`?** `Set` has no duplicates and (usually) no guaranteed order; `List` allows duplicates and keeps insertion order, with index-based access.
- **Q: What does `Collections.unmodifiableList()` do?** Wraps a list in a read-only view — any mutation attempt throws `UnsupportedOperationException`. The underlying list can still change if you keep a reference to it.
- **Q: What's the difference between `HashSet` and `LinkedHashSet`?** `LinkedHashSet` preserves insertion order (backed by `LinkedHashMap` internally), `HashSet` does not guarantee any order.
- **Q: Is `String` a good `HashMap` key?** Yes — it's immutable, and caches its `hashCode()` after the first computation, making repeated lookups cheap.
- **Q: Why must keys used in a `HashMap` be immutable (or at least not mutated after insertion)?** If a key's `hashCode()` changes after insertion, it may land in the wrong bucket on lookup, and the entry becomes "lost" even though it's still in the map.
- **Q: What is `PriorityQueue`?** A queue backed by a binary heap. It always returns the smallest (or by custom `Comparator`, largest/custom priority) element first. Insert/remove is O(log n).
- **Q: What is `ArrayDeque`, and why is it usually preferred over `Stack`/`LinkedList` for stack/queue use?** A resizable array-based double-ended queue. Faster than `LinkedList` (better cache locality, no per-node object overhead) and not synchronized like the legacy `Stack` class.
- **Q: Is `Stack` legacy?** Yes — like `Vector`, it's synchronized (slow) and extends `Vector`, which is an odd design. Use `ArrayDeque` instead.
- **Q: What's `Collections.synchronizedMap()` vs `ConcurrentHashMap`?** `synchronizedMap()` wraps a map with a single lock for all operations (like `Hashtable`), and you must manually synchronize externally when iterating. `ConcurrentHashMap` has fine-grained internal locking and safe iteration without external locks.
- **Q: How do you make an `ArrayList` thread-safe quickly?** `Collections.synchronizedList(new ArrayList<>())`, or `CopyOnWriteArrayList` for read-heavy cases, or a `List` backed by a lock you manage yourself.

## Common Traps & Gotchas

- **Forgetting to override `hashCode()` when overriding `equals()` (or vice versa).** Breaks the `HashMap`/`HashSet` contract — objects that should be found are silently "lost."
- **Mutating a key object after it's already a `HashMap` key.** The key's hash bucket location becomes stale; `map.get(key)` may return `null` even though the entry is still physically present.
- **Assuming `HashMap` preserves insertion order.** It does not — that's `LinkedHashMap`'s job. Relying on `HashMap` order is a common, hard-to-spot bug that "seems to work" until the map grows or resizes.
- **Modifying a list while using a `for-each` loop.** Throws `ConcurrentModificationException`. Use `iterator.remove()`, `removeIf()`, or iterate a copy.
- **Confusing `Hashtable` and `HashMap` null-key behavior.** `Hashtable.put(null, x)` throws `NullPointerException`; `HashMap` allows one `null` key.
- **Thinking `ConcurrentHashMap` fully prevents all race conditions.** It only makes individual operations (`get`, `put`) thread-safe. Compound actions like "check-then-act" (`if (!map.containsKey(k)) map.put(k, v)`) can still race — use `putIfAbsent()`, `computeIfAbsent()`, or `merge()` instead, which are atomic.
- **Using `Vector`/`Hashtable`/`Stack` in new code "because they're thread-safe."** They use a single coarse lock and are slower than modern alternatives (`ConcurrentHashMap`, `ArrayDeque`, `CopyOnWriteArrayList`) that give better real concurrency.
- **Expecting `TreeMap`/`TreeSet` to work with objects that have no `Comparable` implementation and no `Comparator` supplied.** Throws `ClassCastException` at runtime, not compile time.
- **Believing a `HashMap` resize is "free."** It's O(n) — every entry may be rehashed/moved. If you know the final size upfront, pass an initial capacity to the constructor to avoid repeated resizes.
- **Autoboxing surprises in collections of primitives wrappers**, e.g., `List<Integer>` — using `==` to compare values pulled from a collection compares object references, not values, outside the cached range (-128 to 127). Use `.equals()` or `Objects.equals()`.
- **Treeified buckets are per-bucket, not per-map.** A `HashMap` can have both plain linked-list buckets and tree buckets at the same time — treeify only happens to buckets that individually exceed the threshold.
