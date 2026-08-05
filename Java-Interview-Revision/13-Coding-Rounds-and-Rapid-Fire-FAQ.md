# 13. Coding Rounds & Rapid-Fire FAQ

This sheet covers the coding round for a 3–4 years Java backend role. It has common coding patterns, ready-to-use Java solutions, Stream tasks, a big rapid-fire Q&A list, and a short behavioral section. Use it as your last-minute revision before the interview.

## How Java Coding Rounds Work

At 3–4 years experience, the coding round is usually not hard algorithm puzzles. It is a mix of:

- Easy to medium DSA problems (arrays, strings, linked lists, hashmaps, trees).
- Java-specific problems (LRU cache, producer-consumer, thread-safe counter, custom comparator).
- Sometimes a small design-and-code task (rate limiter, parking lot, in-memory cache).

**How to communicate during the round:**
1. Repeat the problem in your own words. Confirm input and output format.
2. Ask about edge cases before coding: empty input, null, duplicates, negative numbers, very large input.
3. Say your approach out loud first (brute force, then optimized). Get a nod before typing.
4. Write clean code. Use meaningful variable names, not `a`, `b`, `x`.
5. After coding, trace through one example by hand.
6. State time and space complexity at the end. Mention trade-offs if there is more than one approach.

**Edge cases interviewers expect you to check:**
- Empty array or string.
- Single element.
- All elements same.
- Null input.
- Very large input (does your solution scale, O(n²) vs O(n log n)?).

**Complexity analysis tip:** Always name both time and space complexity. If you use extra data structures (HashMap, HashSet), say why, and what space it costs.

## Must-Know Java Coding Patterns

- **Two pointers**: Use two index variables moving from both ends or at different speeds. Good for sorted array pair-sum, reversing in place, cycle detection.
- **Sliding window**: Keep a window `[left, right]` over an array or string. Grow `right`, shrink `left` when a condition breaks. Good for substrings, max sum subarray.
- **Hashing with HashMap**: Store value-to-index or value-to-count. Gives O(1) average lookup. Used in two-sum, duplicates, frequency problems.
- **Frequency count**: Use `HashMap<Character, Integer>` or an `int[26]` array for lowercase letters. Faster than HashMap when the key space is small.
- **Sorting + Comparator**: Use `Arrays.sort(arr, Comparator...)` or `list.sort(...)`. Custom sort logic goes in a `Comparator` (lambda or method reference).
- **Recursion / backtracking basics**: Define base case first, then the recursive case. For backtracking, add a choice, recurse, then undo the choice (backtrack).
- **BFS/DFS on trees and graphs**: DFS uses recursion or an explicit `Deque` as a stack. BFS uses a `Queue` (usually `ArrayDeque`) and processes level by level.
- **Deque as stack or queue**: `ArrayDeque` is the recommended class for both. Use `push`/`pop` for stack behavior, `offer`/`poll` for queue behavior. Faster than `Stack` (legacy, synchronized) and `LinkedList`.
- **PriorityQueue (heap)**: `PriorityQueue<Integer>` is a min-heap by default. Pass a `Comparator` to make it a max-heap. Used for "top K", "kth largest", merging sorted lists.

## Common Coding Questions (with Java solutions)

### 1. Reverse a string

```java
public String reverse(String s) {
    return new StringBuilder(s).reverse().toString();
}
```
Time: O(n). Space: O(n) for the new string.

### 2. Reverse a linked list

```java
public ListNode reverse(ListNode head) {
    ListNode prev = null;
    while (head != null) {
        ListNode next = head.next;
        head.next = prev;
        prev = head;
        head = next;
    }
    return prev;
}
```
Iterative. Time: O(n). Space: O(1).

### 3. Check palindrome

```java
public boolean isPalindrome(String s) {
    int i = 0, j = s.length() - 1;
    while (i < j) {
        if (s.charAt(i) != s.charAt(j)) return false;
        i++; j--;
    }
    return true;
}
```
Two pointers. Time: O(n). Space: O(1).

### 4. Find duplicates in an array

```java
public List<Integer> findDuplicates(int[] nums) {
    Set<Integer> seen = new HashSet<>();
    List<Integer> result = new ArrayList<>();
    for (int n : nums) {
        if (!seen.add(n)) result.add(n);
    }
    return result;
}
```
`Set.add` returns false if the value is already present. Time: O(n). Space: O(n).

### 5. Two-sum

```java
public int[] twoSum(int[] nums, int target) {
    Map<Integer, Integer> map = new HashMap<>();
    for (int i = 0; i < nums.length; i++) {
        int need = target - nums[i];
        if (map.containsKey(need)) return new int[]{map.get(need), i};
        map.put(nums[i], i);
    }
    return new int[]{-1, -1};
}
```
One pass with HashMap. Time: O(n). Space: O(n).

### 6. First non-repeating character

```java
public char firstNonRepeating(String s) {
    Map<Character, Integer> count = new LinkedHashMap<>();
    for (char c : s.toCharArray()) count.merge(c, 1, Integer::sum);
    for (Map.Entry<Character, Integer> e : count.entrySet()) {
        if (e.getValue() == 1) return e.getKey();
    }
    return '\0';
}
```
`LinkedHashMap` keeps insertion order, so we return the first one. Time: O(n). Space: O(n).

### 7. Anagram check

```java
public boolean isAnagram(String a, String b) {
    if (a.length() != b.length()) return false;
    int[] count = new int[26];
    for (char c : a.toCharArray()) count[c - 'a']++;
    for (char c : b.toCharArray()) count[c - 'a']--;
    for (int c : count) if (c != 0) return false;
    return true;
}
```
Assumes lowercase letters only. Time: O(n). Space: O(1) (fixed size array).

### 8. Fibonacci (memoized)

```java
public long fib(int n, Map<Integer, Long> memo) {
    if (n <= 1) return n;
    if (memo.containsKey(n)) return memo.get(n);
    long result = fib(n - 1, memo) + fib(n - 2, memo);
    memo.put(n, result);
    return result;
}
```
Top-down recursion with cache. Time: O(n). Space: O(n) for memo and call stack.

### 9. Merge two sorted lists

```java
public ListNode merge(ListNode a, ListNode b) {
    ListNode dummy = new ListNode(0);
    ListNode tail = dummy;
    while (a != null && b != null) {
        if (a.val <= b.val) { tail.next = a; a = a.next; }
        else { tail.next = b; b = b.next; }
        tail = tail.next;
    }
    tail.next = (a != null) ? a : b;
    return dummy.next;
}
```
Dummy node avoids special-casing the head. Time: O(n+m). Space: O(1).

### 10. Find the second highest number

```java
public int secondHighest(int[] nums) {
    int first = Integer.MIN_VALUE, second = Integer.MIN_VALUE;
    for (int n : nums) {
        if (n > first) { second = first; first = n; }
        else if (n > second && n != first) { second = n; }
    }
    return second;
}
```
One pass, no sorting needed. Time: O(n). Space: O(1).

### 11. Implement an LRU cache using LinkedHashMap

LRU means "Least Recently Used". When the cache is full, it removes the item used longest ago.

```java
class LRUCache<K, V> extends LinkedHashMap<K, V> {
    private final int capacity;

    public LRUCache(int capacity) {
        super(capacity, 0.75f, true); // true = access order
        this.capacity = capacity;
    }

    @Override
    protected boolean removeEldestEntry(Map.Entry<K, V> eldest) {
        return size() > capacity;
    }
}
```
`LinkedHashMap` with `accessOrder = true` moves an entry to the end on every `get`. Overriding `removeEldestEntry` auto-evicts the oldest entry when size exceeds capacity. `get` and `put` are O(1).

### 12. Producer-consumer using BlockingQueue

`BlockingQueue` is a queue that blocks the thread when it is full (on `put`) or empty (on `take`). It handles the waiting and locking for you.

```java
BlockingQueue<Integer> queue = new LinkedBlockingQueue<>(10);

Runnable producer = () -> {
    for (int i = 0; i < 100; i++) {
        try { queue.put(i); } catch (InterruptedException e) { Thread.currentThread().interrupt(); }
    }
};

Runnable consumer = () -> {
    while (true) {
        try {
            int val = queue.take();
            System.out.println("Consumed: " + val);
        } catch (InterruptedException e) { Thread.currentThread().interrupt(); break; }
    }
};

new Thread(producer).start();
new Thread(consumer).start();
```
No manual `wait`/`notify` needed. `put` blocks if the queue is full; `take` blocks if it is empty.

### 13. Count word frequency using Streams

```java
String text = "the quick brown fox the lazy fox";
Map<String, Long> freq = Arrays.stream(text.split(" "))
    .collect(Collectors.groupingBy(w -> w, Collectors.counting()));
```
`groupingBy` groups words as keys; `counting()` counts occurrences per group. Time: O(n).

## Java 8 Streams — Common Coding Asks

**1. Group a list by a property**
```java
Map<String, List<Employee>> byDept = employees.stream()
    .collect(Collectors.groupingBy(Employee::getDept));
```

**2. Find duplicates in a list**
```java
Set<Integer> seen = new HashSet<>();
List<Integer> dups = nums.stream()
    .filter(n -> !seen.add(n))
    .collect(Collectors.toList());
```

**3. Sum and average of a list**
```java
int sum = nums.stream().mapToInt(Integer::intValue).sum();
double avg = nums.stream().mapToInt(Integer::intValue).average().orElse(0);
```

**4. Sort a Map by value**
```java
Map<String, Integer> sorted = map.entrySet().stream()
    .sorted(Map.Entry.comparingByValue())
    .collect(Collectors.toMap(Map.Entry::getKey, Map.Entry::getValue,
        (a, b) -> a, LinkedHashMap::new));
```
`LinkedHashMap` keeps the sorted order; a plain `HashMap` would lose it.

**5. Count occurrences of each element**
```java
Map<String, Long> counts = words.stream()
    .collect(Collectors.groupingBy(w -> w, Collectors.counting()));
```

**6. Find the maximum element**
```java
Optional<Integer> max = nums.stream().max(Integer::compareTo);
```
`Optional` is a wrapper that may or may not hold a value. It avoids returning `null` directly.

## Rapid-Fire — 60+ One-Line Q&A

**Core Java**

1. **Is String immutable? Why?** Yes. Once created, its value cannot change. This makes it safe to share, cache (string pool), and use as a HashMap key.
2. **== vs equals()?** `==` compares references (memory address) for objects. `equals()` compares logical content, if the class overrides it.
3. **final vs finally vs finalize?** `final` = constant/no override/no subclass. `finally` = block that always runs after try/catch. `finalize` = old method called before garbage collection (deprecated, don't use).
4. **Overloading vs overriding?** Overloading = same method name, different parameters, resolved at compile time. Overriding = subclass redefines parent method, resolved at runtime.
5. **Abstract class vs interface?** Abstract class can have state and constructors, supports single inheritance. Interface has only method contracts (plus default/static methods), supports multiple inheritance.
6. **Comparable vs Comparator?** `Comparable` defines natural order inside the class (`compareTo`). `Comparator` defines external, custom order (`compare`), can have many for one class.
7. **Shallow copy vs deep copy?** Shallow copy copies references to nested objects (both copies share them). Deep copy creates new copies of nested objects too.
8. **Static vs instance?** Static belongs to the class, shared by all objects. Instance belongs to each object separately.
9. **What is transient?** A field marked `transient` is skipped during serialization.
10. **String vs StringBuilder vs StringBuffer?** `String` is immutable. `StringBuilder` is mutable, fast, not thread-safe. `StringBuffer` is mutable and thread-safe (synchronized), slower.
11. **What is autoboxing?** Automatic conversion between primitive types and their wrapper classes, like `int` to `Integer`.
12. **What is the String pool?** A special memory area where string literals are stored and reused, to save memory.
13. **Can we override a static method?** No. Static methods are resolved at compile time (method hiding), not runtime polymorphism.
14. **What is the diamond problem in Java?** A conflict when a class inherits the same default method from two interfaces. Java forces you to override it and resolve the conflict.
15. **What are wrapper classes?** Object versions of primitives, like `Integer` for `int`, `Double` for `double`. Needed for collections, generics, and null values.

**Collections**

16. **ArrayList vs LinkedList?** `ArrayList` uses a resizable array, fast random access (O(1)), slow insert/delete in the middle (O(n)). `LinkedList` uses nodes, fast insert/delete at ends (O(1)), slow random access (O(n)).
17. **HashMap vs ConcurrentHashMap?** `HashMap` is not thread-safe. `ConcurrentHashMap` is thread-safe, uses internal locking on segments/buckets, allows concurrent reads and writes.
18. **Fail-fast vs fail-safe iterators?** Fail-fast throws `ConcurrentModificationException` if the collection changes during iteration (e.g., `ArrayList`). Fail-safe works on a copy or snapshot, no exception (e.g., `CopyOnWriteArrayList`).
19. **HashMap vs Hashtable?** `HashMap` allows one null key, not thread-safe. `Hashtable` is legacy, synchronized, no null keys/values.
20. **HashSet vs TreeSet vs LinkedHashSet?** `HashSet` = no order, O(1) add. `TreeSet` = sorted order, O(log n) add. `LinkedHashSet` = insertion order, O(1) add.
21. **How does HashMap work internally?** It uses an array of buckets. The key's `hashCode()` decides the bucket. Collisions in the same bucket are stored as a linked list (or a tree, if the list gets long).
22. **What happens if hashCode() is not overridden properly?** Objects that are logically equal may go to different buckets, breaking HashMap/HashSet lookups.
23. **Why override equals() and hashCode() together?** Equal objects must have equal hash codes. Breaking this contract causes bugs in hash-based collections.
24. **What is load factor in HashMap?** The threshold (default 0.75) at which the HashMap resizes (doubles) its internal array.
25. **Array vs ArrayList?** Array has a fixed size, can hold primitives. `ArrayList` is resizable, holds only objects (via autoboxing for primitives).

**Concurrency**

26. **Process vs thread?** A process is an independent running program with its own memory. A thread is a lightweight unit inside a process; threads share the same memory.
27. **sleep() vs wait()?** `sleep()` pauses the thread, keeps the lock, no need for synchronized block. `wait()` releases the lock, needs a synchronized block, waits until `notify()`/`notifyAll()`.
28. **Runnable vs Callable?** `Runnable.run()` returns nothing and cannot throw checked exceptions. `Callable.call()` returns a value and can throw checked exceptions.
29. **volatile vs synchronized?** `volatile` makes a variable's updates visible to all threads immediately (no caching), but does not lock. `synchronized` gives mutual exclusion (only one thread at a time) and visibility.
30. **What is a deadlock?** Two or more threads wait forever for locks held by each other. Avoid by locking resources in a consistent order.
31. **What is a race condition?** When two threads access shared data at the same time and the result depends on timing, causing bugs.
32. **What is ExecutorService?** A framework to manage a pool of threads, so you don't create raw `Thread` objects manually.
33. **What is a thread pool?** A group of reusable worker threads, so the app doesn't pay the cost of creating a new thread for every task.
34. **What is CompletableFuture?** A class for writing async, non-blocking code, with chaining methods like `thenApply`, `thenCombine`.
35. **What is the Java Memory Model?** It defines rules for how threads see changes to shared memory, and when those changes become visible to other threads.
36. **What is a daemon thread?** A background thread (like garbage collector) that does not stop the JVM from exiting.

**JVM**

37. **Heap vs stack memory?** Heap stores objects, shared across threads, managed by garbage collector. Stack stores method calls and local variables, one stack per thread.
38. **What is garbage collection?** The JVM's automatic process of freeing memory used by objects that are no longer reachable.
39. **What are the JVM memory generations?** Young generation (new objects, includes Eden and Survivor spaces) and Old generation (long-lived objects). Minor GC cleans young gen, Major/Full GC cleans old gen.
40. **What is a ClassLoader?** A JVM component that loads `.class` files into memory at runtime.
41. **What is JIT compilation?** Just-In-Time compilation converts bytecode to native machine code at runtime, for better performance on hot code paths.
42. **What is OutOfMemoryError vs StackOverflowError?** `OutOfMemoryError` = heap is full, cannot allocate more objects. `StackOverflowError` = stack is full, usually from deep or infinite recursion.

**Exceptions**

43. **Checked vs unchecked exceptions?** Checked exceptions must be declared or caught (like `IOException`), checked at compile time. Unchecked exceptions (like `RuntimeException`) are not checked at compile time.
44. **Error vs Exception?** `Error` is a serious problem, usually not recoverable (like `OutOfMemoryError`). `Exception` is a condition the app can catch and handle.
45. **Try-with-resources?** A try block that auto-closes resources (implementing `AutoCloseable`) at the end, no need for manual `finally` close.

**Java 8+**

46. **What is a functional interface?** An interface with exactly one abstract method, usable as a lambda target (e.g., `Runnable`, `Comparator`).
47. **map() vs flatMap() in Streams?** `map()` transforms each element one-to-one. `flatMap()` flattens nested structures (like a stream of lists) into a single stream.
48. **What is Optional used for?** A container that may or may not hold a value, used to avoid `NullPointerException` and force explicit null handling.
49. **Intermediate vs terminal stream operations?** Intermediate (`map`, `filter`) return a new stream and are lazy. Terminal (`collect`, `forEach`, `reduce`) trigger the actual execution.
50. **What is a default method in an interface?** A method with a body inside an interface, added so old interfaces can gain new methods without breaking existing classes.

**Spring / Spring Boot**

51. **What is Dependency Injection (DI)?** A pattern where an object's dependencies are provided (injected) from outside, instead of the object creating them itself.
52. **Types of DI in Spring?** Constructor injection (recommended), setter injection, field injection.
53. **What is a Spring Bean?** An object managed by the Spring IoC (Inversion of Control) container.
54. **Bean scopes in Spring?** `singleton` (one instance per container, default), `prototype` (new instance every time), plus web scopes like `request` and `session`.
55. **@Component vs @Service vs @Repository?** All are stereotypes for Spring beans. `@Service` = business logic layer. `@Repository` = data access layer, also translates DB exceptions. `@Component` = generic.
56. **What does @Transactional do?** Wraps a method in a database transaction, so all its DB operations commit together or roll back together on error.
57. **@Transactional propagation types (common ones)?** `REQUIRED` (default, join existing or create new), `REQUIRES_NEW` (always start a new transaction, suspend the current one), `NESTED` (a savepoint inside the current transaction).
58. **What is Spring Boot auto-configuration?** Spring Boot automatically configures beans based on the JARs present on the classpath and property values, reducing manual XML/Java config.
59. **@RestController vs @Controller?** `@RestController` = `@Controller` + `@ResponseBody`, returns data (JSON/XML) directly. `@Controller` typically returns a view name.
60. **What is a Spring profile?** A way to group configuration (like `application-dev.yml`) and activate it for a specific environment.

**Hibernate / JPA**

61. **What is the N+1 problem?** When fetching a list of N parent entities triggers N extra queries to fetch each one's related child entities, instead of one combined query. Fixed with `JOIN FETCH` or batch fetching.
62. **First-level vs second-level cache in Hibernate?** First-level cache is per session, always on. Second-level cache is shared across sessions, must be enabled explicitly (e.g., with EhCache).
63. **Lazy vs eager loading?** Lazy loading fetches related data only when accessed. Eager loading fetches related data immediately with the parent.
64. **What is the difference between save() and persist() in Hibernate?** `save()` returns the generated ID immediately and may execute an insert right away. `persist()` does not return the ID, and defers the insert until flush, per JPA spec.

## Behavioral & Project Questions

For your project, use the **STAR method** in one line: Situation (what was the context), Task (what you had to do), Action (what you did), Result (the outcome, with a number if possible).

**Common questions and how to answer them:**

- **"Tell me about a hard bug you fixed."** Pick a bug that needed real investigation, not a typo. Explain: how you found it (logs, debugger, metrics), the root cause, the fix, and how you prevented it from happening again (test, monitoring, code review rule).
- **"Tell me about a design decision you made."** Explain the problem, two or three options you considered, why you picked one (trade-offs: performance, simplicity, cost, team skill), and the outcome.
- **"How did you handle a production issue?"** Explain: how you got alerted, how you assessed impact (how many users, how severe), your immediate mitigation (rollback, feature flag, hotfix), then the root cause fix and postmortem.
- **"Tell me about a disagreement with a teammate."** Pick a real technical disagreement, not a personality conflict. Explain both viewpoints fairly, how you reached a decision (data, a small spike/prototype, escalation to a lead), and what you learned.

**General tips:**
- Keep each answer to 1–2 minutes. Do not ramble.
- Use real numbers when you can: "reduced latency by 30%", "handled 10,000 requests/day".
- It is fine to say "we" for team work, but be ready to explain your own specific part.
- Prepare 3–4 stories in advance (a bug, a design decision, a production issue, a disagreement) so you are not thinking from scratch in the interview.

## Final Revision Checklist

- [ ] HashMap internal working, and equals()/hashCode() contract.
- [ ] ArrayList vs LinkedList, and other common collection comparisons.
- [ ] Checked vs unchecked exceptions, try-with-resources.
- [ ] Thread basics: sleep vs wait, volatile vs synchronized, Runnable vs Callable.
- [ ] Streams: map/filter/collect, groupingBy, Optional.
- [ ] Practice writing two-sum, reverse linked list, and LRU cache from memory, without an IDE.
- [ ] Spring: DI types, bean scopes, @Transactional propagation.
- [ ] Hibernate: N+1 problem, lazy vs eager loading.
- [ ] Prepare 2–3 STAR-format project stories (bug, design decision, production issue).
- [ ] Review this rapid-fire list once, end to end, the night before.
