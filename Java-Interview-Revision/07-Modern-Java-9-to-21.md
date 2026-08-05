# 7. Modern Java (9–21)

Java changed a lot after Java 8. New versions came out every 6 months, and every 2–3 years one is marked LTS (Long Term Support). Interviewers now expect you to know Java 11 and Java 17 features at least at a usage level, and Java 21 virtual threads are a hot topic.

## Key Concepts (Quick Revision)

**`var` — local variable type inference (Java 10)**
- `var` lets the compiler guess the type from the right-hand side. It is not a dynamic type. The type is fixed at compile time.
```java
var list = new ArrayList<String>(); // inferred as ArrayList<String>
var name = "John";                  // inferred as String
```
- You CANNOT use `var`:
    - For fields (instance or static variables).
    - For method parameters or return types.
    - Without an initializer (`var x;` is illegal).
    - When the initializer is `null` (compiler cannot guess the type).
    - For lambda bodies without a target type (`var x = () -> {}` is illegal).

**Records (Java 16, preview in 14)**
- A record is a shorter way to write an immutable data class.
- One line gives you: private final fields, a constructor, getters (named after the field, no `get` prefix), `equals()`, `hashCode()`, and `toString()`.
```java
public record Point(int x, int y) { }

Point p = new Point(1, 2);
p.x();          // getter, not getX()
p.toString();   // Point[x=1, y=2]
```
- You can add extra methods or a compact constructor for validation:
```java
public record Point(int x, int y) {
    public Point {                 // compact constructor
        if (x < 0) throw new IllegalArgumentException("x must be positive");
    }
}
```
- Limits: fields are always `final`. A record cannot extend another class (it silently extends `Record`). It can implement interfaces. Use records for simple immutable DTOs, not for entities that need mutation (like JPA entities).

**Sealed classes and interfaces (Java 17)**
- "Sealed" means you control exactly which classes are allowed to extend or implement a type. This gives you exhaustive control, similar to a fixed enum of types.
```java
public sealed interface Shape permits Circle, Square { }
public final class Circle implements Shape { double radius; }
public final class Square implements Shape { double side; }
```
- Each permitted subclass must be declared `final`, `sealed`, or `non-sealed`.
- Useful with pattern matching in `switch` because the compiler can check all cases are covered (no `default` needed).

**Switch expressions (Java 14)**
- Old `switch` was a statement with fall-through risk. New `switch` can be an expression that returns a value.
```java
String day = switch (dayNum) {
    case 1, 7 -> "Weekend";
    case 2, 3, 4, 5, 6 -> "Weekday";
    default -> "Invalid";
};
```
- Arrow (`->`) form does not fall through. No `break` needed.
- Use `yield` when the case body needs multiple statements:
```java
int result = switch (x) {
    case 1 -> 10;
    default -> {
        int computed = x * 2;
        yield computed;
    }
};
```

**Pattern matching for `switch` (Java 21)**
- `switch` can now check the type of an object and bind it to a variable in the same line.
```java
static String describe(Shape s) {
    return switch (s) {
        case Circle c -> "Circle with radius " + c.radius();
        case Square sq -> "Square with side " + sq.side();
    };
}
```
- Works well with sealed types: the compiler checks all subtypes are handled, so you can skip `default`.
- Supports record patterns (destructuring) too:
```java
static String show(Point p) {
    return switch (p) {
        case Point(int x, int y) when x == y -> "Diagonal point";
        case Point(int x, int y) -> "Point " + x + "," + y;
    };
}
```

**Pattern matching for `instanceof` (Java 16)**
- Old way needed a manual cast after the check. New way binds the variable directly.
```java
// Old
if (obj instanceof String) {
    String s = (String) obj;
    System.out.println(s.length());
}

// New
if (obj instanceof String s) {
    System.out.println(s.length());
}
```

**Text blocks (Java 15)**
- A text block is a multi-line string literal. It removes the need for `\n` and `+` concatenation.
```java
String json = """
    {
      "name": "John",
      "age": 30
    }
    """;
```
- Starts with `"""` followed by a newline. Leading whitespace common to all lines is stripped automatically.

**Helpful new methods**
| Method | What it does |
|---|---|
| `List.of(1, 2, 3)` | Creates an immutable list. Throws on `null` elements. |
| `Map.of("a", 1, "b", 2)` | Creates an immutable map. |
| `Set.of(1, 2, 3)` | Creates an immutable set. |
| `stream.toList()` | Shorter than `.collect(Collectors.toList())`. Returns an immutable list. |
| `String.isBlank()` | `true` if string is empty or only whitespace. |
| `String.strip()` | Like `trim()`, but Unicode-aware. |
| `String.repeat(n)` | Repeats the string `n` times. |
| `String.lines()` | Splits the string into a `Stream<String>` of lines. |
| `Optional.isEmpty()` | Opposite of `isPresent()`. Easier to read in `if` checks. |

**Module system (JPMS, Java 9)**
- JPMS = Java Platform Module System. It groups packages into a named module with an explicit `module-info.java` file.
```java
module com.myapp {
    requires java.sql;
    exports com.myapp.service;
}
```
- Why it exists: before Java 9, everything on the classpath was visible to everything else ("classpath hell"). Modules let you declare exactly what a module needs (`requires`) and what it shares (`exports`), so internal packages stay hidden.
- In practice: most Spring Boot apps do not use JPMS directly. Know the concept and be able to explain `requires`/`exports` at a high level. Deep module design questions are rare at 3–4 YOE interviews.

**`CompletableFuture` improvements (Java 9)**
- Added timeout support: `orTimeout(long, TimeUnit)` and `completeOnTimeout(value, long, TimeUnit)`.
- Added `Executor`-aware delay: `Executors.newVirtualThreadPerTaskExecutor()` works well with async pipelines in Java 21.
- Small but useful: you no longer need manual `ScheduledExecutorService` tricks just to add a timeout to a future.

**Virtual threads (Project Loom, Java 21)**
- A virtual thread is a lightweight thread managed by the JVM, not by the OS.
- The problem it solves: a normal ("platform") thread maps 1-to-1 to an OS thread. OS threads are expensive (heavy stack, limited count, usually a few thousand max). If your app does blocking I/O (DB calls, HTTP calls) and you use one thread per request, you run out of threads under high load.
- Virtual threads let you create millions of threads. The JVM schedules many virtual threads on a small pool of OS ("carrier") threads. When a virtual thread blocks on I/O, the JVM unmounts it from the OS thread and lets another virtual thread run. You get blocking-style code that scales like non-blocking code.
```java
try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {
    executor.submit(() -> {
        // looks like normal blocking code
        String result = callSlowService();
        System.out.println(result);
    });
}
```
| | Platform thread | Virtual thread |
|---|---|---|
| Managed by | OS | JVM |
| Stack size | ~1 MB (fixed) | Small, grows as needed |
| Max count | Thousands | Millions |
| Best for | CPU-bound work | I/O-bound, blocking work |
| Creation cost | High | Very low |
- When to use: high-throughput services that spend most of their time waiting on I/O (REST calls, DB queries, file reads). Not useful for CPU-bound work (heavy computation) — virtual threads do not add CPU parallelism, they only remove the OS thread bottleneck for blocking calls.
- Spring Boot 3.2+ supports virtual threads for Tomcat request handling with one config flag: `spring.threads.virtual.enabled=true`.

**Version-by-version headline features**
| Version | Type | Headline features |
|---|---|---|
| Java 8 | LTS | Lambdas, Streams API, `Optional`, default methods in interfaces, new `java.time` package |
| Java 11 | LTS | `var` in lambda params, new `String` methods (`isBlank`, `strip`, `repeat`), `HttpClient` API (standard), single-file source launch (`java File.java`) |
| Java 17 | LTS | Sealed classes, pattern matching for `instanceof`, records (finalized), text blocks (finalized), strong encapsulation of JDK internals |
| Java 21 | LTS | Virtual threads, pattern matching for `switch`, record patterns, sequenced collections |

## Important Interview Questions

**Q1: What is the difference between `var` and dynamic typing (like in JavaScript or Python)?**
`var` is resolved at compile time. The compiler looks at the right-hand side once, fixes the type, and that type never changes. In dynamic typing, the type can change at runtime. `var` is just less typing for the developer; it does not change Java's static type system.
- Follow-up: Can you reassign a `var` variable to a different type later? No. Once inferred, the variable has a fixed type, like any other Java variable.

**Q2: Why would you use a record instead of a normal class with Lombok's `@Data`?**
Records are a language feature, so they work everywhere without needing a library. They are immutable by design (no setters), which is safer for DTOs, value objects, and API responses. `@Data` classes are usually mutable and need an external dependency. Use records when you want a simple, immutable data carrier; use classes when you need mutability, inheritance, or JPA entity mapping.
- Follow-up: Can a record implement an interface? Yes. Can it extend a class? No, because it implicitly extends `Record`.

**Q3: What problem do sealed classes solve?**
Before sealed classes, any public class could be extended by any other class, even outside your control. Sealed classes let you list the exact permitted subclasses. This is useful for modeling a fixed set of types (like a `Shape` with only `Circle` and `Square`), and it lets the compiler check that a `switch` on that type covers all cases.
- Follow-up: What is `non-sealed`? It is used on a subclass of a sealed class to reopen it, allowing further unrestricted extension.

**Q4: How is a switch expression different from a switch statement?**
A switch statement executes code and does not return a value; it needs `break` to avoid fall-through. A switch expression returns a value directly, using `->` arrows, and does not fall through by default. This removes a common bug source (forgetting `break`).
- Follow-up: When do you need `yield`? When a case body has multiple statements and you need to send back a value from inside a block.

**Q5: What is Project Loom trying to fix?**
Traditional Java concurrency uses platform threads, which map 1-to-1 to OS threads. OS threads are costly to create and limited in number. For I/O-heavy applications (most backend services), most threads spend most of their time blocked waiting for a network or database response, wasting OS resources. Virtual threads let the JVM run huge numbers of logical threads on a small number of OS threads, so blocking-style code scales without needing reactive programming (like WebFlux).
- Follow-up: Do virtual threads replace reactive programming? Not fully. Reactive is still useful for complex async pipelines. But for simple "call a service and wait" code, virtual threads let you keep the simpler blocking style while still scaling.

**Q6: What happens if you use `synchronized` blocks with virtual threads?**
If a virtual thread blocks inside a `synchronized` block, it cannot be unmounted from its carrier OS thread (this was true up to Java 21, improved later). This can reduce the scaling benefit if you have long, contended `synchronized` blocks. Prefer `java.util.concurrent` locks (`ReentrantLock`) for code that will run on virtual threads.
- Follow-up: Is this a big issue in practice? Only for code with heavy lock contention. Most typical request-handling code is fine.

**Q7: What does `module-info.java` give you that the classpath does not?**
It gives explicit, enforced boundaries. `requires` declares dependencies; `exports` declares what is public to other modules. Anything not exported is invisible outside the module, even if the class is `public`. This prevents accidental use of internal implementation classes across modules.
- Follow-up: Why don't most Spring Boot apps use JPMS? Spring Boot's classpath-based auto-configuration and many third-party libraries are not fully modularized, so adopting JPMS adds friction without much benefit for typical web apps.

## FAQ / Rapid-Fire

- **Is `var` the same as `Object`?** No. `var` still gets a specific, fixed type from inference. `Object` is a real, generic type. `var x = "hi";` makes `x` a `String`, not an `Object`.
- **Can records have static fields?** Yes. Static fields and static methods are allowed. Only instance fields are restricted to the ones declared in the header.
- **Is `List.of()` the same as `Arrays.asList()`?** No. `List.of()` is fully immutable (throws `UnsupportedOperationException` on modify, and rejects `null` elements). `Arrays.asList()` allows `set()` but not `add()`/`remove()`, and allows `null`.
- **What is `Stream.toList()` vs `collect(Collectors.toList())`?** `toList()` (Java 16+) is shorter and returns an unmodifiable list. `collect(Collectors.toList())` returns a mutable `ArrayList` by contract (though not guaranteed).
- **Do virtual threads need a special API to write code?** No. You write normal blocking code. Only the executor changes: `Executors.newVirtualThreadPerTaskExecutor()`.
- **Are virtual threads daemon threads?** Yes, virtual threads are always daemon threads. They do not stop the JVM from exiting.
- **What is a sequenced collection (Java 21)?** A new interface (`SequencedCollection`) that adds `getFirst()`, `getLast()`, `addFirst()`, `addLast()`, and `reversed()` to ordered collections like `List` and `LinkedHashSet`.
- **Can you use pattern matching `instanceof` with generics?** Yes, but with unchecked warnings for parameterized types, similar to normal casts.

## Common Traps & Gotchas

- **Using `var` for readability loss.** `var result = process();` hides the type if the method name is not clear. Use `var` when the type is obvious from the right side (`var list = new ArrayList<String>()`), avoid it when it makes code harder to read.
- **Thinking records are always better than classes.** Records cannot be extended, cannot have mutable state, and cannot be used as JPA entities (JPA needs a no-arg constructor and mutable fields). Do not force every DTO into a record if the framework needs mutability.
- **Forgetting `permits` clause placement.** All permitted subclasses in a sealed type must be known at compile time and normally must be in the same module or package (with some exceptions for named modules).
- **Assuming `switch` pattern matching needs `default` always.** If the type is `sealed` and all subtypes are covered, `default` is not required. But if the type is not sealed, the compiler cannot prove exhaustiveness, so you still need `default`.
- **Confusing text block indentation.** The closing `"""` position affects how much leading whitespace is stripped. Misplacing it can leave unwanted indentation in the string.
- **Using virtual threads for CPU-bound tasks.** Virtual threads do not add more CPU cores. For CPU-heavy work (image processing, complex calculations), platform threads with a fixed-size pool sized to CPU cores is still correct.
- **Pinning virtual threads accidentally.** Long `synchronized` blocks or native calls can pin a virtual thread to its carrier thread, blocking other virtual threads from using that carrier. Watch for this in legacy code with heavy `synchronized` usage.
- **Thinking JPMS is mandatory in modern Java.** It is optional. Most Spring Boot projects still run as an unnamed module directly on the classpath.
