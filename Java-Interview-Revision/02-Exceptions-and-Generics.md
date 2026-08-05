# 2. Exceptions & Generics

Exceptions handle errors at runtime. Generics give type safety at compile time. Interviewers ask
these two topics together because both are about writing safe, predictable code. At 3–4 years,
they expect you to know internals (what `finally` does to a return value, how erasure works),
not just syntax.

## Key Concepts (Quick Revision)

**Exceptions**
- `Throwable` is the root class. It has two children: `Error` and `Exception`.
    - `Error` — serious problems the app should not try to catch (e.g., `OutOfMemoryError`, `StackOverflowError`).
    - `Exception` — problems the app can handle.
- Under `Exception`:
    - **Checked exceptions** — must be declared (`throws`) or caught. Checked at compile time. Example: `IOException`, `SQLException`.
    - **Unchecked exceptions** (`RuntimeException` and its children) — not checked at compile time. Example: `NullPointerException`, `IllegalArgumentException`.
- `try-with-resources` — auto-closes resources that implement `AutoCloseable`. No need for a manual `finally` block to close them.
- Multi-catch — one `catch` block can handle more than one exception type: `catch (IOException | SQLException e)`.
- Exception chaining — wrap one exception inside another using the `cause` field, so you don't lose the original error.

**Generics**
- Generics let a class or method work with any type, decided at compile time. This gives type safety and removes manual casting.
- Bounded type — `<T extends Number>` limits `T` to `Number` or its subclasses.
- Wildcards — `? extends T` (read-only, producer) and `? super T` (write-only, consumer). Remembered as **PECS**: Producer Extends, Consumer Super.
- Type erasure — the compiler removes generic type info after compile time. At runtime, `List<String>` and `List<Integer>` are both just `List`.
- Diamond operator (`<>`) — lets the compiler infer the generic type on the right side, so you don't repeat it.

## Important Interview Questions

**Q1: What is the difference between checked and unchecked exceptions? When do you use which?**
Checked exceptions are for conditions a caller can reasonably recover from, and they are outside the program's control — like a file not found, or a network call failing. The compiler forces you to handle them.
Unchecked exceptions (`RuntimeException`) are for programming errors — bugs like passing a null where it's not allowed, or a bad array index. The caller usually cannot recover; the fix is to correct the code.
Rule of thumb: use checked exceptions for expected, recoverable failures in business logic (rare in modern practice). Use unchecked exceptions for bugs and broken invariants. Most modern Java code (including Spring) prefers unchecked exceptions, because checked exceptions force every method up the call chain to declare or catch them, which clutters code.
*Follow-up: Why does Spring convert checked JDBC exceptions to unchecked `DataAccessException`?*
So calling code is not forced to catch exceptions it usually cannot recover from. It keeps service and controller code clean.

**Q2: What happens if a `finally` block has a `return` statement?**
The `return` in `finally` **overrides** any `return` or exception from the `try` or `catch` block. This is a common trap.
```java
static int test() {
    try {
        return 1;
    } finally {
        return 2; // this wins, method returns 2
    }
}
```
Even worse: if `try` throws an exception, and `finally` has a `return`, the exception is **swallowed** — the caller never sees it. This is a real bug source, so avoid `return` inside `finally`.
*Follow-up: What is the order of execution — try, catch, finally, then return?*
Java evaluates the return value from `try`/`catch` first, but before actually returning it, `finally` runs. If `finally` does not have its own `return` or `throw`, the earlier saved return value is used. If it does, that new value/exception replaces it.

**Q3: What is try-with-resources? Why is it better than manual `finally` closing?**
`try-with-resources` is a try block that declares one or more resources (objects implementing `AutoCloseable`). Java calls `close()` on them automatically, in reverse order of declaration, when the block ends — even if an exception is thrown.
```java
try (BufferedReader br = new BufferedReader(new FileReader("f.txt"))) {
    return br.readLine();
}
```
It is better than manual `finally` because:
- Less boilerplate (no null checks before calling `close()`).
- Handles **suppressed exceptions** correctly — if both the try block and `close()` throw, the `close()` exception is attached as a "suppressed" exception instead of hiding the original one.
  *Follow-up: What interface must a resource implement?* `AutoCloseable` (or its stricter child `Closeable`, used by I/O classes).

**Q4: What is exception chaining and why does it matter?**
Exception chaining means passing the original exception as the `cause` when you throw a new one:
```java
try {
    repository.save(data);
} catch (SQLException e) {
    throw new ServiceException("Failed to save data", e); // e is the cause
}
```
It matters because without it, the original stack trace is lost. Debugging becomes very hard if you only see the wrapper exception with no idea what caused it underneath.
*Follow-up: How do you retrieve the cause later?* `exception.getCause()`. The full chain prints automatically with `printStackTrace()` as "Caused by: ...".

**Q5: Why should you avoid using exceptions for normal control flow?**
Throwing and catching an exception is expensive — the JVM builds a full stack trace at the point of `throw`, which is a costly operation. Using exceptions for expected outcomes (like "item not found" in a loop) hurts performance and makes code harder to read, because exceptions should mean "something unexpected happened."
Better: use return values (`Optional`, boolean flags, null checks) for expected conditions, and reserve exceptions for truly exceptional situations.

**Q6: What are best practices for custom exceptions?**
- Extend `RuntimeException` unless there's a strong reason for a checked exception.
- Always provide constructors that accept a `cause` (message + `Throwable`), so chaining works.
- Name clearly, ending in `Exception` (e.g., `OrderNotFoundException`).
- Do not swallow exceptions — never leave an empty `catch` block. At minimum, log it.
- Fail fast — validate inputs early and throw immediately, instead of letting bad data travel deep into the system.
- Keep exception messages useful (include IDs, context) but never leak sensitive data (passwords, tokens) in messages or logs.

**Q7: What is `ConcurrentModificationException`? When does it happen?**
It's thrown when a collection is structurally modified (add/remove) while it is being iterated with a normal iterator, and the modification did not go through the iterator itself.
```java
List<Integer> list = new ArrayList<>(List.of(1, 2, 3));
for (Integer i : list) {
    if (i == 2) list.remove(i); // throws ConcurrentModificationException
}
```
It works through a `modCount` field: the iterator checks this count on each `next()` call. If the list changed outside the iterator, the count mismatches and the exception fires.
Fix: use `Iterator.remove()`, `CopyOnWriteArrayList`, or collect items to remove and remove them after the loop.

**Q8: Why do generics matter? What problem did they solve (pre-Java 5)?**
Before generics, collections stored `Object`. You had to cast manually on every read, and the compiler could not catch type mistakes.
```java
List list = new ArrayList();
list.add("hello");
list.add(42); // no error at compile time
String s = (String) list.get(1); // ClassCastException at runtime
```
With generics, `List<String>` only accepts `String`, and the compiler blocks wrong types and inserts the cast for you safely. This moves errors from runtime to compile time — much cheaper to fix.

**Q9: Explain type erasure. Why can't you do `new T()`, `instanceof T`, or `new T[]`?**
Type erasure means the compiler uses generic type information only during compilation, for type checking, then removes it. At runtime, all generic type parameters are replaced by their bound (`Object`, if unbounded) or the erased raw type. `List<String>` and `List<Integer>` become the same `List` class file at runtime.
This is why:
- `new T()` fails — the JVM does not know at runtime what `T` actually is, so it cannot pick a constructor to call.
- `instanceof T` fails — there is no `T` left at runtime to check against; only `Object` exists.
- `new T[]` fails — arrays in Java know their component type at runtime (for array-store checks); erasure would make this unsafe, so it is blocked.
  Workarounds: pass a `Class<T>` object and use `clazz.newInstance()` (or `getDeclaredConstructor().newInstance()`) for creating instances, and use `@SuppressWarnings("unchecked")` with `Object[]` casts for arrays.
  *Follow-up: Is erasure why generics are called "compile-time only"?* Yes — generics are a compile-time safety feature; the bytecode is not generic at all.

**Q10: How does type erasure affect method overloading?**
You cannot overload two methods that differ only by generic type parameter, because after erasure they have the same signature.
```java
// Compile error: both erase to process(List)
void process(List<String> list) { }
void process(List<Integer> list) { }
```
This is a common trap in interviews — it looks valid but fails to compile with "erasure of method process(List) is the same as..." error.

**Q11: What are bounded type parameters? Give an example.**
A bounded type parameter restricts what types can be used, using `extends` (works for both classes and interfaces in this context).
```java
static <T extends Number> double sum(List<T> list) {
    double total = 0;
    for (T t : list) total += t.doubleValue(); // Number methods available
    return total;
}
```
Without the bound, `T` would be treated as `Object`, and you could not call `doubleValue()`.
You can also have multiple bounds: `<T extends Number & Comparable<T>>` (only one class allowed, but multiple interfaces).

**Q12: Explain wildcards and PECS with an example.**
Wildcards (`?`) are used when you don't need to know the exact generic type, only how you will use it.
- `List<? extends T>` — you can **read** `T` (or its subtype) from it, but cannot add anything (except `null`). Use when the list **produces** values for you.
- `List<? super T>` — you can **write** `T` (or its subtype) into it, but reading gives you only `Object`. Use when the list **consumes** values from you.

**PECS = Producer Extends, Consumer Super.**
```java
// copy from src (producer, we read from it) to dest (consumer, we write to it)
static <T> void copy(List<? extends T> src, List<? super T> dest) {
    for (T item : src) {
        dest.add(item);
    }
}
```
Here `src` only gives values out (`extends`), and `dest` only takes values in (`super`). This is exactly how `Collections.copy()` is written in the JDK.
*Follow-up: Why can't you add to a `List<? extends Number>`?* Because the compiler does not know the exact subtype — it could be `List<Integer>` or `List<Double>`. Adding any specific type could break it, so only reading is allowed.

**Q13: What is the diamond operator, and why was it added?**
The diamond operator `<>` (Java 7+) lets the compiler infer the generic type from the left-hand side, so you don't repeat it on the right.
```java
Map<String, List<Integer>> map = new HashMap<>(); // instead of new HashMap<String, List<Integer>>()
```
It reduces boilerplate. It works because the compiler has enough context from the declared type to infer the missing type arguments.

## FAQ / Rapid-Fire

- **Q: Can a `catch` block catch `Error`?** Yes, technically (`catch (Error e)` or `catch (Throwable e)`), but you should not — errors like `OutOfMemoryError` usually mean the JVM is in a bad state and recovery is not safe.
- **Q: Is `NullPointerException` checked or unchecked?** Unchecked — it's a `RuntimeException`.
- **Q: What is `ClassCastException`?** Thrown when you cast an object to a type it is not an instance of, at runtime.
- **Q: Can `finally` be skipped?** Yes, if the JVM exits with `System.exit()`, or the thread is killed, or the machine crashes.
- **Q: Can you have `try` without `catch`?** Yes, `try-finally` is valid without any `catch`.
- **Q: What is a suppressed exception?** An exception thrown by `close()` in try-with-resources, while another exception is already in flight from the try block. It's attached to the main exception, not thrown separately.
- **Q: Can generic types be primitives?** No. Use wrapper classes (`Integer`, `Double`) — this is why autoboxing exists.
- **Q: Can you create a generic array directly, like `T[] arr = new T[10]`?** No, due to type erasure. Use `(T[]) new Object[10]` with a suppress-warning, or ask the caller to pass a `Class<T>`.
- **Q: What is a raw type?** Using a generic class without its type parameter, like `List list = new ArrayList();`. It disables type checking and should be avoided — kept only for backward compatibility with old code.
- **Q: Are static members allowed to use a class's type parameter?** No. Static members belong to the class itself, not to any specific instantiation like `MyClass<String>`. But a static **method** can have its own separate type parameter: `static <T> T identity(T t)`.
- **Q: Does erasure apply to generic methods too?** Yes, the same rules apply — the type parameter exists only at compile time.

## Common Traps & Gotchas

- **`return` inside `finally` silently swallows exceptions.** If `try` throws, but `finally` has its own `return`, the exception disappears and the caller never knows. Never put `return` (or `throw` that replaces the original) in `finally`.
- **Catching `Exception` broadly hides real bugs.** Catching a generic `Exception` when you only expect `IOException` can hide `NullPointerException` and other real bugs. Catch specific types.
- **Empty catch blocks ("swallowing" exceptions)** — `catch (Exception e) {}` hides failures completely. Always log or rethrow.
- **Overload resolution with generics fails at compile time**, not runtime — due to erasure, two methods differing only by generic type are the same signature and won't compile.
- **`List<? extends T>` cannot be added to** (except `null`) — a very common wildcard trap.
- **Mixing checked exceptions with lambdas/streams is awkward** — functional interfaces like `Function<T,R>` don't declare `throws`, so checked exceptions inside lambdas must be wrapped in a try-catch or a custom wrapper.
- **`instanceof` with generics only works on raw types** — `obj instanceof List<String>` will not compile; you can only write `obj instanceof List<?>` or `obj instanceof List`.
- **`ConcurrentModificationException` can also appear in single-threaded code** — it is not only a multithreading issue; modifying a list mid-iteration in one thread also triggers it.
- **Custom exceptions without a `cause` constructor break chaining** — always add a constructor that takes `Throwable cause`, or you'll lose the original stack trace when wrapping.
- **Rethrowing a caught exception resets nothing** — the original stack trace is preserved unless you construct a brand-new exception without passing the old one as cause.
