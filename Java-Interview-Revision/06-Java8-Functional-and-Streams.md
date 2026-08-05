# 6. Java 8 — Functional & Streams

Java 8 added functional programming style to Java. You write *what* to do, not *how* to loop.
This topic is asked in almost every Java interview at 3–4 years level. Interviewers check both
syntax and internal working (laziness, parallel streams, Optional misuse).

## Key Concepts (Quick Revision)

- **Lambda expression**: a short way to write a method body without a class name. It gives an
  implementation of a functional interface.
- **Functional interface**: an interface with exactly one abstract method (SAM — Single Abstract
  Method). A lambda can only implement a functional interface.
- **`@FunctionalInterface`**: an annotation. It tells the compiler "this interface must have only
  one abstract method." If someone adds a second abstract method, compilation fails.
- **Built-in functional interfaces** (in `java.util.function`): `Predicate<T>`, `Function<T,R>`,
  `BiFunction<T,U,R>`, `Consumer<T>`, `Supplier<T>`, `UnaryOperator<T>`, and `Comparator<T>`.
- **Method reference**: a short form of a lambda, when the lambda just calls one existing method.
  Four kinds: static, instance (particular object), instance (arbitrary object of a type),
  constructor.
- **Default method**: a method in an interface with a body, using the `default` keyword. Lets you
  add new methods to an interface without breaking old classes that implement it.
- **Static method in interface**: a utility method that belongs to the interface itself, not to
  any implementing object.
- **Stream**: a sequence of elements that supports pipeline operations (filter, map, reduce, ...).
  A stream is not a data structure; it does not store data.
- **Stream pipeline**: `source → intermediate operations → terminal operation`. Intermediate
  operations are lazy. Nothing runs until a terminal operation is called.
- **Collectors**: helper class with ready-made terminal operations for `collect()`, like
  `toList()`, `groupingBy()`, `joining()`.
- **`Optional<T>`**: a container object that may or may not hold a value. It is used to avoid
  `NullPointerException` and to force the caller to think about the "no value" case.
- **`java.time` package**: the new date/time API (`LocalDate`, `LocalDateTime`, `Instant`,
  `Duration`, `Period`). It is immutable and thread-safe, unlike old `Date` and `Calendar`.

---

## Important Interview Questions

### Q1. What is a lambda expression? What is its syntax?
A lambda expression is an anonymous (nameless) function. It gives the body of a method that
belongs to a functional interface.

```java
// (parameters) -> expression
Runnable r = () -> System.out.println("Hello");

// (parameters) -> { statements; }
Comparator<String> byLength = (a, b) -> {
    return a.length() - b.length();
};

// single parameter, type can be left out
Function<Integer, Integer> square = x -> x * x;
```

Rules:
- Parameter types are optional; the compiler infers them from the functional interface (called
  **target typing**).
- Curly braces and `return` are optional for a single expression.
- A lambda has no name, no explicit return type, and no `throws` clause — it takes these from the
  functional interface's abstract method (the "descriptor").

**Follow-up**: Can a lambda have its own `this`?
Answer: No. `this` inside a lambda refers to the enclosing class instance, not the lambda itself.
This is different from an anonymous inner class, where `this` refers to the anonymous class.

### Q2. What is "effectively final" capture in lambdas?
A lambda can use a local variable from the enclosing scope only if that variable is **final** or
**effectively final** (never reassigned after its first value).

```java
int count = 10;
Runnable r = () -> System.out.println(count); // OK, count never changes

int total = 0;
// total++;  // if you did this anywhere, the lambda below would NOT compile
Runnable r2 = () -> System.out.println(total);
```

Why this rule exists: a lambda may run later, maybe on another thread. Java copies the value of
the local variable into the lambda. If the original variable could still change, the copy and the
original would go out of sync. Making it effectively final avoids this confusion.

**Follow-up**: Can a lambda change an instance field?
Answer: Yes. Instance fields and static fields are not local variables; they live on the heap and
have no "effectively final" restriction. Only **local** variables and method parameters need this.

### Q3. What is a functional interface? Give examples.
An interface with exactly **one abstract method**. It can have any number of `default` and
`static` methods — those do not count.

```java
@FunctionalInterface
interface Calculator {
    int calculate(int a, int b);          // the one abstract method
    default void print(int r) {           // allowed, does not break SAM rule
        System.out.println("Result: " + r);
    }
}
```

Built-in functional interfaces you must know:

| Interface | Abstract method | Purpose | Example |
|---|---|---|---|
| `Predicate<T>` | `boolean test(T t)` | test a condition | `s -> s.isEmpty()` |
| `Function<T,R>` | `R apply(T t)` | transform T to R | `s -> s.length()` |
| `BiFunction<T,U,R>` | `R apply(T t, U u)` | transform two inputs to R | `(a,b) -> a + b` |
| `Consumer<T>` | `void accept(T t)` | use a value, no return | `s -> System.out.println(s)` |
| `Supplier<T>` | `T get()` | produce a value, no input | `() -> new ArrayList<>()` |
| `UnaryOperator<T>` | `T apply(T t)` | `Function<T,T>`, same input/output type | `s -> s.trim()` |
| `Comparator<T>` | `int compare(T a, T b)` | compare two values for sorting | `(a,b) -> a.age - b.age` |

**Follow-up**: What does `@FunctionalInterface` actually do at runtime?
Answer: Nothing at runtime. It is only a compile-time check. Even without it, an interface with
one abstract method is still a valid functional interface. The annotation just protects you from
accidentally adding a second abstract method later.

### Q4. What are method references? Explain all 4 kinds.
A method reference is shorthand for a lambda that only calls one existing method. Written with
`::`.

| Kind | Syntax | Example | Same as lambda |
|---|---|---|---|
| 1. Static method | `ClassName::staticMethod` | `Integer::parseInt` | `s -> Integer.parseInt(s)` |
| 2. Instance method of a particular object | `object::instanceMethod` | `myList::add` | `x -> myList.add(x)` |
| 3. Instance method of an arbitrary object of a type | `ClassName::instanceMethod` | `String::toUpperCase` | `s -> s.toUpperCase()` |
| 4. Constructor reference | `ClassName::new` | `ArrayList::new` | `() -> new ArrayList<>()` |

```java
List<String> names = List.of("bob", "alice", "carol");

names.forEach(System.out::println);           // kind 2 (System.out is an object)
names.stream().map(String::toUpperCase)       // kind 3
              .forEach(System.out::println);

List<Integer> lens = names.stream()
    .map(String::length)                      // kind 3
    .collect(Collectors.toList());

Supplier<ArrayList<String>> listMaker = ArrayList::new;  // kind 4
```

**Follow-up**: Why use method references instead of lambdas?
Answer: Only readability. They compile to the same bytecode idea. Use them when the lambda body
is just "call this one method, nothing else."

### Q5. What are default and static methods in interfaces? Why were they added?
- **Default method**: has a body, uses `default` keyword. Any implementing class gets this method
  for free, but can override it.
- **Static method**: belongs to the interface, called as `InterfaceName.method()`. Cannot be
  overridden by implementing classes.

They were added so interfaces like `List`, `Collection`, and `Comparator` could get new methods
(`forEach`, `stream`, `sort`, `thenComparing`) in Java 8 **without breaking every existing class**
that already implemented them.

```java
interface Vehicle {
    default void start() {
        System.out.println("Vehicle starting");
    }
    static Vehicle create() {
        return () -> System.out.println("Generic vehicle start"); // if SAM present
    }
}
```

**Follow-up (diamond problem)**: What if a class implements two interfaces that both have a
default method with the same signature?

```java
interface A {
    default void hello() { System.out.println("A"); }
}
interface B {
    default void hello() { System.out.println("B"); }
}
class C implements A, B {
    // Must override, or compile error: "class C inherits unrelated defaults for hello()"
    @Override
    public void hello() {
        A.super.hello();   // explicitly choose A's version
        B.super.hello();   // or B's version, or write new logic
    }
}
```
Java forces you to override the method and resolve the conflict yourself. It will not guess.

### Q6. What is a Stream? How is it different from a Collection?
A **stream** is a sequence of elements that supports functional-style operations, computed one
pipeline at a time. It comes from `java.util.stream`.

| Collection | Stream |
|---|---|
| Stores data in memory | Does not store data; just a view/pipeline over a source |
| Can be iterated many times | Can be consumed (terminal op run) only **once** |
| You write the loop (external iteration) | The library does the looping (internal iteration) |
| Eager — data exists now | Lazy — computed only when a terminal op is called |
| Mutable (add/remove elements) | Immutable pipeline; operations return a **new** stream |

```java
List<String> list = List.of("a", "bb", "ccc");
Stream<String> stream = list.stream();
stream.forEach(System.out::println);
// stream.forEach(System.out::println); // throws IllegalStateException: stream already operated upon or closed
```

**Follow-up**: What does "lazy evaluation" mean for streams?
Answer: Intermediate operations (`filter`, `map`, ...) do not run when you call them. They just
build up a pipeline description. Only when a **terminal operation** (`collect`, `forEach`,
`reduce`, ...) is called does the whole pipeline actually run, element by element.

```java
Stream.of(1, 2, 3)
    .filter(x -> { System.out.println("filter " + x); return x > 1; })
    .map(x -> { System.out.println("map " + x); return x * 2; });
// Nothing is printed! No terminal operation was called.
```

### Q7. Explain the stream pipeline: source → intermediate ops → terminal op.
- **Source**: where the stream comes from — a collection (`list.stream()`), an array
  (`Arrays.stream(arr)`), `Stream.of(...)`, a file (`Files.lines(path)`), or a generator.
- **Intermediate operations**: return a new stream, so you can chain more operations. Lazy.
  Examples: `filter`, `map`, `flatMap`, `distinct`, `sorted`, `limit`, `skip`, `peek`.
- **Terminal operation**: triggers the actual processing and produces a result (or a side
  effect). After this, the stream is "consumed" and cannot be reused. Examples: `forEach`,
  `collect`, `reduce`, `count`, `anyMatch`, `findFirst`, `toArray`.

```java
long count = list.stream()          // source
    .filter(s -> s.length() > 1)    // intermediate
    .map(String::toUpperCase)       // intermediate
    .distinct()                     // intermediate
    .count();                       // terminal
```

**Table of common intermediate ops:**

| Operation | What it does |
|---|---|
| `filter(Predicate)` | keeps elements that match a condition |
| `map(Function)` | transforms each element to another value |
| `flatMap(Function)` | transforms each element into a stream, then flattens all into one stream |
| `distinct()` | removes duplicates (uses `equals()`) |
| `sorted()` / `sorted(Comparator)` | sorts elements |
| `limit(n)` | keeps only first `n` elements |
| `peek(Consumer)` | looks at each element without changing the stream; mainly for debugging |

**Table of common terminal ops:**

| Operation | What it does |
|---|---|
| `forEach(Consumer)` | performs an action on each element |
| `collect(Collector)` | gathers elements into a List, Set, Map, String, etc. |
| `reduce(...)` | combines all elements into one result |
| `count()` | counts elements |
| `anyMatch(Predicate)` | true if any element matches (short-circuits) |
| `findFirst()` | returns first element as `Optional` (short-circuits) |

### Q8. Explain `reduce()` with an example.
`reduce` combines all stream elements into a single result, using a combining function.

```java
List<Integer> nums = List.of(1, 2, 3, 4, 5);

// with identity value
int sum = nums.stream().reduce(0, (a, b) -> a + b);   // 15

// without identity, returns Optional (stream could be empty)
Optional<Integer> max = nums.stream().reduce((a, b) -> a > b ? a : b);  // Optional[5]

// 3-argument form: (identity, accumulator, combiner) — used mainly in parallel streams
int sumParallel = nums.parallelStream()
    .reduce(0, (a, b) -> a + b, (a, b) -> a + b);
```
The **combiner** (3rd argument) is used only in parallel streams, to merge partial results from
different threads.

### Q9. Explain `flatMap()` with an example.
`flatMap` is used when each element maps to **multiple** elements (like a list inside a list). It
flattens a "stream of streams" into one single stream.

```java
List<List<Integer>> listOfLists = List.of(
    List.of(1, 2, 3),
    List.of(4, 5),
    List.of(6)
);

List<Integer> flatList = listOfLists.stream()
    .flatMap(List::stream)     // each List<Integer> becomes a Stream<Integer>, then merged
    .collect(Collectors.toList());
// [1, 2, 3, 4, 5, 6]
```

**Follow-up**: `map` vs `flatMap` — what's the difference?
Answer: `map` gives one output for one input (`Stream<List<Integer>>` stays `Stream<List<Integer>>`
transformed). `flatMap` gives zero, one, or many outputs for one input, and merges them into a
single flat stream (`Stream<List<Integer>>` becomes `Stream<Integer>`).

### Q10. Explain `Collectors` in depth.
`Collectors` is a helper class with ready-made collector implementations, used with
`stream.collect(...)`.

```java
List<String> names = List.of("Amit", "Bala", "Amit", "Charlie");

// toList / toSet
List<String> asList = names.stream().collect(Collectors.toList());

// toMap — key, value functions
Map<String, Integer> nameLength = names.stream()
    .distinct()
    .collect(Collectors.toMap(n -> n, String::length));

// groupingBy — groups elements by a classifier function
Map<Integer, List<String>> byLength = names.stream()
    .collect(Collectors.groupingBy(String::length));
// {4=[Amit, Bala, Amit], 7=[Charlie]}

// partitioningBy — splits into exactly two groups: true / false
Map<Boolean, List<String>> partitioned = names.stream()
    .collect(Collectors.partitioningBy(n -> n.length() > 4));

// joining — combine strings
String joined = names.stream().collect(Collectors.joining(", ", "[", "]"));
// "[Amit, Bala, Amit, Charlie]"

// counting, mapping — often used inside groupingBy as a "downstream" collector
Map<Integer, Long> countByLength = names.stream()
    .collect(Collectors.groupingBy(String::length, Collectors.counting()));

Map<Integer, List<Character>> firstLetters = names.stream()
    .collect(Collectors.groupingBy(String::length,
        Collectors.mapping(n -> n.charAt(0), Collectors.toList())));
```

**`toMap` duplicate-key trap**: if two elements produce the same key, `toMap` throws
`IllegalStateException: Duplicate key` — it does NOT silently overwrite.

```java
// names has "Amit" twice — CRASHES at runtime:
Map<String, Integer> bad = names.stream()
    .collect(Collectors.toMap(n -> n, String::length));
// java.lang.IllegalStateException: Duplicate key Amit

// Fix: give a merge function (3rd argument) to say what to do on conflict
Map<String, Integer> fixed = names.stream()
    .collect(Collectors.toMap(n -> n, String::length, (existing, replacement) -> existing));
```

### Q11. Why do primitive streams (`IntStream`, `LongStream`, `DoubleStream`) exist?
`Stream<Integer>` boxes every `int` into an `Integer` object. This wastes memory and CPU time
(boxing/unboxing) for large data. Primitive streams store primitives directly, avoiding this.

```java
IntStream.rangeClosed(1, 5)          // 1,2,3,4,5 (inclusive)
    .filter(x -> x % 2 == 0)
    .forEach(System.out::println);

int sum = IntStream.of(1, 2, 3).sum();
OptionalDouble avg = IntStream.of(1, 2, 3).average();

// convert between Stream<Integer> and IntStream
IntStream is = list.stream().mapToInt(Integer::intValue);
Stream<Integer> back = is.boxed();
```
Primitive streams also give extra methods not on `Stream<T>`: `sum()`, `average()`, `range()`,
`rangeClosed()`.

### Q12. How do parallel streams work? When do they help, and when do they hurt?
`stream.parallelStream()` or `stream.parallel()` splits the data into chunks and processes chunks
on multiple threads, then combines results. It uses the **common `ForkJoinPool`** (shared
JVM-wide, size = number of CPU cores by default), the same pool used by `CompletableFuture`'s
default async methods.

```java
List<Integer> nums = IntStream.rangeClosed(1, 1_000_000).boxed().collect(Collectors.toList());
long sum = nums.parallelStream().mapToLong(Integer::longValue).sum();
```

**When it helps**:
- Large data sets (typically hundreds of thousands+ elements).
- CPU-heavy, independent operations per element (no shared state).
- The source splits well (`ArrayList`, arrays, `IntStream.range` split well; `LinkedList` does
  not).

**When it hurts**:
- **Small data**: the overhead of splitting, thread coordination, and merging is bigger than the
  actual work. Sequential is usually faster for small lists.
- **Stateful operations**: like `sorted()` on unordered data, or a lambda with shared mutable
  state — hard to parallelize correctly, and can give wrong results.
- **Shared mutable state**: if the lambda writes to a shared variable/collection (e.g.
  `list.add()` inside `forEach` from multiple threads), it causes race conditions since these
  collections are not thread-safe.
- **Blocking I/O tasks**: parallel streams use the shared common pool. If one task blocks
  (network/disk I/O), it can starve other unrelated parallel stream usages in the whole
  application, since they all share the same pool.

```java
// WRONG: shared mutable state, not thread-safe
List<Integer> results = new ArrayList<>();
nums.parallelStream().forEach(results::add);  // race condition, may lose elements

// RIGHT: use collect, which handles merging safely
List<Integer> results2 = nums.parallelStream().collect(Collectors.toList());
```

**Follow-up**: Can you control which thread pool a parallel stream uses?
Answer: Not directly per-stream in a clean, guaranteed way. All parallel streams share the common
`ForkJoinPool` unless you submit the whole parallel stream operation inside a custom
`ForkJoinPool.submit(...)` call, which is a known workaround, not an official API.

### Q13. Why does `Optional` exist? How do you use it correctly?
`Optional<T>` is a container that may hold a value, or may be empty. It exists to make "this might
have no value" **visible in the method signature**, instead of silently returning `null` and
risking a `NullPointerException` later.

```java
Optional<String> name = Optional.ofNullable(getName());  // may be null

// map: transform value only if present
Optional<Integer> len = name.map(String::length);

// flatMap: when the transformation itself returns an Optional
Optional<String> upper = name.flatMap(n -> Optional.of(n.toUpperCase()));

// orElse: default value if empty
String result = name.orElse("Unknown");

// orElseGet: default value from a Supplier, computed only if needed (lazy)
String result2 = name.orElseGet(() -> computeDefault());

// orElseThrow: throw a custom exception if empty
String result3 = name.orElseThrow(() -> new IllegalStateException("Name missing"));

// safe consuming
name.ifPresent(n -> System.out.println("Name: " + n));
```

**Common misuses (avoid these)**:
```java
// BAD: calling get() without checking — same NPE risk as before, just moved
String bad = name.get(); // throws NoSuchElementException if empty

// BAD: using Optional as a method parameter or class field
void process(Optional<String> input) { ... }  // don't do this

// BAD: using isPresent() + get() instead of map/orElse (defeats the purpose)
if (name.isPresent()) {
    System.out.println(name.get().toUpperCase());
}
// GOOD:
name.map(String::toUpperCase).ifPresent(System.out::println);
```
Rule of thumb: use `Optional` only as a **method return type**, to tell the caller "this may not
have a result." Do not use it for fields, parameters, or collections (use an empty collection
instead of `Optional<List<T>>`).

### Q14. What is the new Date/Time API? Why did it replace `Date` and `Calendar`?
Old `java.util.Date` and `java.util.Calendar` had problems: mutable (not thread-safe), months
started at 0, confusing API, and `Date` mixed up date and time and time zone concepts.

Java 8 added `java.time` package, based on the joda-time library:

| Class | Represents |
|---|---|
| `LocalDate` | date only, no time, no zone (`2026-08-04`) |
| `LocalTime` | time only, no date (`14:30:00`) |
| `LocalDateTime` | date + time, no zone |
| `Instant` | a point in time on the UTC timeline (like a machine timestamp) |
| `Duration` | amount of time in seconds/nanoseconds — for time-based amounts (`PT2H` = 2 hours) |
| `Period` | amount of time in years/months/days — for date-based amounts (`P1Y2M` = 1 year 2 months) |
| `ZonedDateTime` | date + time + time zone |

```java
LocalDate today = LocalDate.now();
LocalDate birthday = LocalDate.of(1995, Month.JANUARY, 15);
Period age = Period.between(birthday, today);

LocalDateTime meeting = LocalDateTime.of(2026, 8, 4, 15, 0);
LocalDateTime later = meeting.plusHours(2);

Instant now = Instant.now();                  // machine timestamp, UTC
Duration gap = Duration.between(now, now.plusSeconds(90));

// All these classes are IMMUTABLE — every "change" returns a new object
LocalDate tomorrow = today.plusDays(1);  // today itself is unchanged
```
All `java.time` classes are **immutable and thread-safe**, unlike `Date`/`Calendar`, which are
mutable and unsafe to share between threads.

---

## FAQ / Rapid-Fire

- **Q: Can a lambda throw a checked exception?**
  Only if the functional interface's abstract method declares it in `throws`. Most built-in
  interfaces (`Function`, `Predicate`, ...) do not declare checked exceptions, so you must
  wrap/catch inside the lambda.

- **Q: Is `Comparator` a functional interface?**
  Yes. It has one abstract method, `compare(T, T)`. It also has useful default/static methods:
  `reversed()`, `thenComparing()`, `Comparator.comparing()`.

- **Q: Difference between `Predicate.and()`, `or()`, `negate()`?**
  These are default methods on `Predicate` to combine conditions:
  `p1.and(p2)`, `p1.or(p2)`, `p1.negate()`.

- **Q: Can an interface with default methods still be a functional interface?**
  Yes, as long as it has exactly one **abstract** method. Default and static methods don't count.

- **Q: What happens if you call a terminal operation twice on the same stream?**
  `IllegalStateException: stream has already been operated upon or closed`.

- **Q: Is `peek()` meant for changing data?**
  No, it is meant for debugging/logging only. Do not rely on it to modify stream elements.

- **Q: `Stream.of()` vs `Arrays.stream()`?**
  `Stream.of(1,2,3)` creates a stream from varargs. `Arrays.stream(arr)` creates a stream from an
  existing array. For primitive arrays, `Arrays.stream(intArr)` returns an `IntStream`.

- **Q: What is short-circuiting in streams?**
  Some operations stop processing early once the result is known: `anyMatch`, `allMatch`,
  `noneMatch`, `findFirst`, `findAny`, `limit`. They don't need to check every element.

- **Q: `findFirst()` vs `findAny()`?**
  `findFirst()` always returns the first element in encounter order. `findAny()` may return any
  matching element — faster in parallel streams since it doesn't need to keep order.

- **Q: Can you reuse a `Collectors.groupingBy` result as a `Map<K, V>` or does it need casting?**
  No casting needed; `groupingBy` returns `Map<K, List<T>>` by default (or `Map<K, D>` with a
  downstream collector), already correctly typed.

- **Q: `Instant` vs `LocalDateTime` — when to use which?**
  Use `Instant` for machine-to-machine timestamps (logs, storing "when did this happen" in UTC).
  Use `LocalDateTime`/`ZonedDateTime` when human-readable date and time in a specific zone matter.

---

## Common Traps & Gotchas

- **Modifying a captured local variable**: you cannot reassign a local variable used inside a
  lambda. Compile error, not a runtime error. Use an array or an `AtomicInteger` wrapper as a
  workaround if you truly need mutable state (usually a sign of bad design though).

- **Reusing a stream**: a stream can be consumed only once. Calling any operation on it again
  throws `IllegalStateException`. If you need to run it twice, create the stream again from the
  source (`list.stream()` each time).

- **`toMap` duplicate keys crash**: silently expecting the last value to win — it actually throws
  `IllegalStateException`. Always pass a merge function if duplicates are possible.

- **`Optional.get()` without checking**: this can throw `NoSuchElementException`, defeating the
  whole purpose of `Optional`. Prefer `orElse`, `orElseGet`, `orElseThrow`, or `map`.

- **Using `Optional` as a field or parameter type**: not its intended use; makes serialization and
  APIs awkward. Use it only as a return type.

- **Parallel streams on small lists**: this often makes code **slower**, not faster, because of
  thread setup and merging overhead. Always measure before using `parallelStream()`.

- **Shared mutable state in parallel streams**: writing to a normal `ArrayList` or a counter
  variable from a parallel stream causes race conditions, since these are not thread-safe.
  Use `collect()` or thread-safe structures instead.

- **`peek()` used to mutate data**: relying on `peek()` for real logic (not just debug printing)
  is fragile — behavior can differ between sequential and parallel streams, and the JIT may skip
  `peek()` entirely if the result is never used.

- **Forgetting streams are lazy**: writing intermediate operations and expecting them to run
  immediately. Nothing happens until a terminal operation is called.

- **`LocalDate`/`LocalDateTime` are immutable**: calling `date.plusDays(1)` does NOT change
  `date`. You must assign the result: `date = date.plusDays(1)`. This is the same mistake people
  make with `String`.

- **Mixing up `Duration` and `Period`**: `Duration` is for time-based amounts (hours, minutes,
  seconds) and works with `Instant`/`LocalTime`. `Period` is for date-based amounts (years,
  months, days) and works with `LocalDate`. Using the wrong one on the wrong type causes
  `UnsupportedTemporalTypeException`.

- **`flatMap` misused where `map` is enough**: if the mapping function does not return a
  stream/collection, you don't need `flatMap`. Using it anyway is confusing and may not compile.
