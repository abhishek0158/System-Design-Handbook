# 1. Core Java & OOP

Core Java and OOP are the base of every Java interview. Interviewers use this topic to check if you
really understand the language, not just the syntax. At 3–4 years, they expect you to explain
**how** things work internally (like the `equals`/`hashCode` contract, or the Integer cache) and
**why** Java made these design choices, not just definitions.

## Key Concepts (Quick Revision)

### The 4 OOP Pillars

| Pillar | Meaning | Real Example |
|---|---|---|
| **Encapsulation** | Hiding internal data. Only expose it through methods. | Private fields in a class, accessed via getters/setters. `BankAccount` hides `balance`, exposes `deposit()`/`withdraw()`. |
| **Abstraction** | Hiding *how* something works, showing only *what* it does. | `List` interface hides whether it is backed by an array (`ArrayList`) or a linked list (`LinkedList`). |
| **Inheritance** | A class reuses fields/methods of another class. | `class Car extends Vehicle` — `Car` reuses `Vehicle`'s `start()` method. |
| **Polymorphism** | One interface, many implementations. Two kinds: compile-time (overloading) and runtime (overriding). | `List<String> list = new ArrayList<>();` — calling `list.add()` runs `ArrayList`'s code, decided at runtime. |

- **Compile-time polymorphism (overloading)**: same method name, different parameter list, resolved by the compiler.
- **Runtime polymorphism (overriding)**: subclass gives its own version of a parent method, resolved at runtime using the actual object type (dynamic dispatch).

### Class vs Object

- A **class** is a blueprint. It defines fields and behavior, but takes no memory for data until used.
- An **object** is an instance of a class. It exists in heap memory and holds actual data.
- One class can produce many objects, each with its own state but sharing the same code (methods).

### Interface vs Abstract Class

| Aspect | Interface | Abstract Class |
|---|---|---|
| Purpose | Defines a contract ("what") | Partial implementation ("what" + some "how") |
| Fields | Only `public static final` (constants) | Any type of field, any access modifier |
| Constructors | No | Yes (called by subclass constructor) |
| Multiple inheritance | A class can implement many interfaces | A class can extend only one abstract class |
| Methods (pre-Java 8) | All abstract | Can mix abstract and concrete methods |
| Methods (Java 8+) | Can have `default` and `static` methods with body | Same as before |

**Java 8 default and static methods — why they matter:**
- `default` methods let you add a new method to an interface without breaking every class that already implements it. Example: `List.stream()` was added as a `default` method — all old `List` implementations kept working.
- `static` methods in interfaces hold utility logic tied to the interface, like `Comparator.comparing(...)`.
- **Diamond problem with default methods**: if a class implements two interfaces that both have the same default method, the class must override it, or it will not compile.

```java
interface A { default void hello() { System.out.println("A"); } }
interface B { default void hello() { System.out.println("B"); } }
class C implements A, B {
    // must override, otherwise compile error
    public void hello() { A.super.hello(); }
}
```

- When to choose which: use an interface for a pure contract or when you need multiple inheritance of type. Use an abstract class when subclasses share common state or common code, and you control the class hierarchy.

### Access Modifiers

| Modifier | Same class | Same package | Subclass (different package) | Different package |
|---|---|---|---|---|
| `private` | Yes | No | No | No |
| default (no modifier) | Yes | Yes | No | No |
| `protected` | Yes | Yes | Yes | No |
| `public` | Yes | Yes | Yes | Yes |

- `protected` is often misunderstood: a subclass in another package can access the protected member, but only through a reference of the subclass type, not through a parent-class reference.

### `static`, `final`, `this`, `super`

- **`static`**: belongs to the class, not to an instance. Shared across all objects. Static methods cannot use `this` (no instance context) and cannot call non-static (instance) methods directly.
- **`final`**:
    - `final` variable — value cannot be reassigned after first assignment (but if it's an object reference, the object's internal state can still change).
    - `final` method — cannot be overridden.
    - `final` class — cannot be extended (example: `String`, `Integer`).
- **`this`**: reference to the current object. Used to resolve field/parameter name conflicts, or to pass the current object to another method/constructor (`this(...)` calls another constructor in the same class).
- **`super`**: reference to the parent class. Used to call the parent constructor (`super(...)`, must be the first line), or to call a parent method that a subclass has overridden (`super.methodName()`).

### `==` vs `.equals()`

- `==` compares **references** for objects (do both variables point to the same memory address?). For primitives, it compares **values**.
- `.equals()` compares **logical/content equality**. Default `Object.equals()` behaves like `==` unless the class overrides it (e.g., `String`, wrapper classes, and `record` classes override it to compare content).

```java
String a = new String("test");
String b = new String("test");
a == b;        // false, different objects on heap
a.equals(b);   // true, same content
```

### `equals()` and `hashCode()` Contract

Rules you must follow when overriding both:
1. If two objects are equal by `.equals()`, they **must** return the same `hashCode()`.
2. If two objects have the same `hashCode()`, they are **not required** to be equal (hash collision is allowed).
3. `hashCode()` must be consistent — same object, same hash code, across calls, as long as fields used in `equals` don't change.

**What breaks in a `HashMap` if you get this wrong:**
- `HashMap` uses `hashCode()` to find the right bucket, then `.equals()` to find the exact key inside that bucket.
- If you override `equals()` but not `hashCode()`: two "equal" objects can land in different buckets. `map.get(key)` can fail to find a value you just put, because the lookup key's hash code differs from the stored key's hash code.
- If `hashCode()` always returns the same value (like `0`): technically correct, but every key lands in one bucket — the map turns into a linked list, and performance drops from O(1) to O(n).

```java
class Point {
    int x, y;
    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (!(o instanceof Point)) return false;
        Point p = (Point) o;
        return x == p.x && y == p.y;
    }
    @Override
    public int hashCode() {
        return Objects.hash(x, y);
    }
}
```

### Making a Class Immutable

An immutable class is one whose state cannot change after creation. Steps:
1. Mark the class `final` (prevent subclassing that could add mutable behavior).
2. Make all fields `private final`.
3. No setters.
4. Initialize all fields via the constructor.
5. For mutable fields (like `Date` or a `List`), do a **defensive copy** on the way in (constructor) and on the way out (getter) — otherwise the caller can still mutate the internal state through the reference.

```java
final class Employee {
    private final String name;
    private final List<String> skills;

    Employee(String name, List<String> skills) {
        this.name = name;
        this.skills = new ArrayList<>(skills); // defensive copy in
    }
    List<String> getSkills() {
        return new ArrayList<>(skills); // defensive copy out
    }
}
```

**Why `String` is immutable:**
- **Security** — strings are used for class names, file paths, network URLs, DB credentials. If mutable, one part of the code could change it after a security check passed.
- **String pool (interning) works only if immutable** — safe to share the same object among many references.
- **Hashcode caching** — `String` caches its hash code, since it never changes. This makes it fast as a `HashMap` key.
- **Thread safety** — no synchronization needed to share strings across threads.

**String pool and `intern()`**:
- String literals (`"abc"`) are stored in a special memory area called the **String pool** (part of heap since Java 7). Reusing a literal reuses the same object.
- `new String("abc")` always creates a new object on the heap, outside the pool.
- `.intern()` puts a string into the pool (or returns the existing pooled reference if already present).

```java
String a = "abc";              // pool
String b = "abc";              // same pool reference
String c = new String("abc");  // new heap object
a == b;            // true
a == c;            // false
a == c.intern();   // true
```

### String vs StringBuilder vs StringBuffer

| | Mutable? | Thread-safe? | Use case |
|---|---|---|---|
| `String` | No | Yes (immutable, inherently safe) | Fixed or rarely-changed text |
| `StringBuilder` | Yes | No | String building in a single thread (most common — loops, concatenation) |
| `StringBuffer` | Yes | Yes (synchronized methods) | String building shared across threads (rarely used now; prefer `StringBuilder` + external sync if needed) |

- Every `+` concatenation on `String` in a loop creates a new object each time — this is O(n²) for n concatenations. Use `StringBuilder` in loops.

### Wrapper Classes, Autoboxing/Unboxing

- Each primitive has a wrapper class: `int` → `Integer`, `char` → `Character`, `boolean` → `Boolean`, etc.
- **Autoboxing**: automatic conversion from primitive to wrapper (`Integer i = 5;`).
- **Unboxing**: automatic conversion from wrapper to primitive (`int x = i;`).

**Pitfalls:**
- **Integer cache**: Java caches `Integer` objects for values **-128 to 127** (`IntegerCache` class). Values in this range reuse the same cached object; outside this range, `new` objects are created each time.

```java
Integer a = 100, b = 100;
a == b;   // true, both from cache

Integer c = 200, d = 200;
c == d;   // false, different objects, outside cache range
```
This is a classic interview trap. Always use `.equals()` to compare wrapper objects, never `==`.
- **NullPointerException on unboxing**: if a wrapper object is `null` and gets auto-unboxed, it throws NPE.
```java
Integer count = null;
int x = count; // NullPointerException
```
- **Autoboxing in loops/collections hurts performance**: boxing/unboxing repeatedly (e.g., `Long sum = 0L; for (...) sum += i;`) creates many short-lived objects. Use primitive accumulators when performance matters.

### Pass-by-Value in Java

**Java is always pass-by-value. There is no pass-by-reference — not even for objects.**
- For primitives: a copy of the value is passed.
- For objects: a copy of the **reference** (the address) is passed, not the object itself, and not a reference to the reference.

This means:
- If you change the object's *fields* through the passed reference, the caller sees the change (because both references point to the same object).
- If you *reassign* the parameter to point to a new object, the caller's original reference is unaffected (because you only changed your local copy of the reference).

```java
void modify(StringBuilder sb) {
    sb.append("world");     // caller sees this — same object
}
void reassign(StringBuilder sb) {
    sb = new StringBuilder("new"); // caller does NOT see this — local copy only
}
```
This is the single most common Core Java misconception. Say clearly: "Java passes the value of the reference, not the reference itself."

### Class Initialization Order

When a class is loaded and an object is created, the order is:
1. **Static variables and static blocks** — run once, in the order they appear, when the class is first loaded (parent class's statics run before child class's statics).
2. **Instance variables and instance initializer blocks** — run every time an object is created, in the order they appear, before the constructor body.
3. **Constructor body** — runs last (but implicit or explicit `super()` call happens first, before step 2 of the current class).

Full order across a hierarchy for `new Child()`:
Parent static → Child static → Parent instance blocks → Parent constructor → Child instance blocks → Child constructor.

```java
class Parent {
    static { System.out.println("Parent static"); }
    { System.out.println("Parent instance"); }
    Parent() { System.out.println("Parent constructor"); }
}
class Child extends Parent {
    static { System.out.println("Child static"); }
    { System.out.println("Child instance"); }
    Child() { System.out.println("Child constructor"); }
}
// new Child() prints:
// Parent static -> Child static -> Parent instance -> Parent constructor -> Child instance -> Child constructor
```

### Nested / Inner / Anonymous / Static Nested Classes

| Type | Needs outer instance? | Access outer's private fields? | Typical use |
|---|---|---|---|
| **Static nested class** | No | Only static members of outer | Grouping a helper class inside its logical owner, e.g., `Map.Entry` |
| **Inner class (non-static)** | Yes, holds implicit reference to outer object | Yes, all members | Class tightly bound to an outer instance's state |
| **Local class** | Defined inside a method | Yes (plus effectively-final local variables) | Rarely used, small helper scoped to one method |
| **Anonymous class** | Defined inline, no name, extends/implements one type | Yes | One-off implementation, e.g., old-style `Runnable` or listener |

```java
class Outer {
    int x = 10;
    class Inner {              // non-static inner class
        void show() { System.out.println(x); } // can access outer's x
    }
    static class Nested {      // static nested class
        void show() { System.out.println("no access to outer x"); }
    }
}
Outer o = new Outer();
Outer.Inner inner = o.new Inner();  // needs outer instance
Outer.Nested nested = new Outer.Nested(); // no outer instance needed
```

- A non-static inner class holds a hidden reference to its enclosing instance. This can cause **memory leaks** if the inner class object outlives the intended lifetime of the outer object (common issue with anonymous inner classes used as listeners in Android/Swing, or in long-lived caches).

### Enums

- An `enum` is a special class where every constant is a fixed, pre-created instance — type-safe, unlike plain `int` constants.
- Enums can have fields, constructors (always `private`), and methods. Each constant can override a method.

```java
enum Operation {
    ADD { public int apply(int a, int b) { return a + b; } },
    SUB { public int apply(int a, int b) { return a - b; } };
    public abstract int apply(int a, int b);
}
```

- **Enum singleton**: the safest way to write a singleton in Java. JVM guarantees a single instance, and it is inherently serialization-safe and reflection-safe (can't create a second instance via reflection, unlike a normal class singleton).

```java
enum Singleton {
    INSTANCE;
    void doWork() { }
}
Singleton.INSTANCE.doWork();
```

### Marker Interfaces

- A marker interface has **no methods**. It just "tags" a class with some meta-information, read by the JVM or a framework at runtime.
- Examples: `Serializable` (tells JVM the object can be converted to a byte stream), `Cloneable` (tells `Object.clone()` it is allowed to clone this object), `Remote` (RMI).
- Modern alternative: annotations (like `@FunctionalInterface`, custom annotations) are now preferred over marker interfaces for adding metadata, because they can carry extra data and be processed more flexibly.

### `Object` Class Methods

Every class in Java implicitly extends `Object`. Key methods:
- `equals(Object o)` — logical equality, default is reference equality (`==`).
- `hashCode()` — returns an int hash, default based on memory address.
- `toString()` — human-readable representation, default is `ClassName@hashCode` in hex.
- `getClass()` — returns runtime class metadata (`Class<?>` object), used in reflection.
- `clone()` — creates a copy of the object; class must implement `Cloneable`, otherwise throws `CloneNotSupportedException`. Default is a **shallow copy**.
- `wait()`, `notify()`, `notifyAll()` — used for thread communication, must be called inside a `synchronized` block, or throws `IllegalMonitorStateException`.
- `finalize()` — deprecated since Java 9, called by GC before reclaiming an object; unreliable timing, avoid using it.

## Important Interview Questions

**Q1: Why does Java not support multiple inheritance of classes, but allows it through interfaces?**
Multiple inheritance of classes causes the **diamond problem** — if two parent classes have the same method, the compiler cannot decide which version the child should use. Java avoids this by allowing a class to extend only one class. Interfaces used to be safe from this because all their methods were abstract (no ambiguous implementation). Since Java 8 added `default` methods, interfaces can technically hit the diamond problem too, but Java forces the implementing class to resolve it explicitly by overriding the method — the ambiguity becomes a compile error, not a runtime surprise.

**Q2: If you override `equals()`, why must you also override `hashCode()`?**
Because of the `equals`-`hashCode` contract: equal objects must produce equal hash codes. Hash-based collections (`HashMap`, `HashSet`, `HashTable`) rely on this to locate objects correctly. If you skip `hashCode()`, two logically equal objects can get different hash codes, land in different buckets, and the collection will behave incorrectly — lookups can silently fail even though the key "looks" present.
*Follow-up: What if you override `hashCode()` but not `equals()`?* Then two different objects might return the same hash code and still be treated as different when compared with `.equals()`. That is legal (collisions are allowed) but usually a sign of an incomplete design if you meant them to be logically equal.

**Q3: Why is `String` immutable in Java?**
Four reasons: security (strings are used in class loading, file paths, network calls — mutation could break access checks after validation), string pool reuse (safe sharing of one object across many references, saves memory), hash code caching (safe to cache since it never changes, making `String` fast as a `HashMap` key), and thread safety (shared freely across threads with no locks).
*Follow-up: How would you create your own immutable class?* See the "Making a Class Immutable" steps above — `final` class, `private final` fields, no setters, defensive copies for mutable fields.

**Q4: Explain the Integer cache (`-128` to `127`) and why `==` gives inconsistent results.**
Java caches `Integer` objects for the range -128 to 127 through `Integer.valueOf()`, because these are the most common values (like caching common flyweight objects). Autoboxing calls `Integer.valueOf()`, so values in this range reuse cached objects, and `==` returns `true`. Values outside this range create new objects each time, so `==` returns `false`. This is a JVM optimization detail, not something to rely on — always use `.equals()` for wrapper comparison.

**Q5: Is Java pass-by-value or pass-by-reference? Explain with an example.**
Java is always **pass-by-value**. For objects, the value passed is a copy of the reference (the memory address), not the object and not a reference-to-the-reference. So changes made *through* the reference (calling a setter, modifying a field) are visible to the caller. But reassigning the parameter inside the method to point to a different object does not affect the caller's original variable, because only the local copy of the reference changed.

**Q6: What is the difference between an abstract class and an interface? When would you choose one over the other?**
An abstract class can hold state (instance fields), constructors, and a mix of implemented and abstract methods; a class can extend only one. An interface defines a contract, can now have `default`/`static` methods with bodies (Java 8+), but still cannot hold instance state or constructors; a class can implement many interfaces. Choose an abstract class when subclasses share common code/state and you control the hierarchy. Choose an interface for a pure capability contract, or when a class needs to satisfy multiple contracts (multiple inheritance of type).

**Q7: What are static, instance, and initializer blocks, and what order do they run in?**
Static blocks run once, when the class is loaded by the JVM, in the order written, and only after the parent class's static blocks run. Instance initializer blocks run every time an object is created, right before the constructor body, after the implicit/explicit `super()` call. So the overall order is: parent static → child static (once, at class load) → parent instance block → parent constructor → child instance block → child constructor (every time a `new Child()` happens).

**Q8: How do you make a thread-safe singleton, and why is enum considered the best way?**
Common approaches: eager initialization, synchronized `getInstance()`, and double-checked locking with a `volatile` field. Enum-based singleton is the safest because the JVM guarantees only one instance exists across the app, even under reflection or serialization attacks — a normal class singleton can be broken using reflection (`setAccessible(true)` on the private constructor) or improper deserialization, but an `enum` cannot be instantiated through reflection or deserialized into a second instance.

**Q9: What is the difference between method overloading and method overriding?**
Overloading is having multiple methods with the same name but different parameter lists in the same class; it is resolved at compile time based on the reference type and argument types (static/compile-time polymorphism). Overriding is a subclass providing its own version of a parent's method with the same signature; it is resolved at runtime based on the actual object type (dynamic dispatch, runtime polymorphism).
*Follow-up: Can you overload by changing only the return type?* No. The parameter list must differ; return type alone is not enough to distinguish overloaded methods.

**Q10: What happens if you don't override `toString()`? What issues does that cause?**
Without overriding, `toString()` returns `ClassName@hexHashCode`, which is not useful for logging or debugging. In real projects, this makes logs hard to read (e.g., `com.app.User@1b6d3586` instead of showing the user's ID or name). It is good practice to override `toString()` on domain/DTO classes for readable logs. Tools like Lombok's `@ToString` or IDE-generated methods are common in real codebases.

## FAQ / Rapid-Fire

- **Can a `static` method be overridden?** No, it can be hidden (method hiding), not overridden. Resolution is by reference type, not object type.
- **Can an interface have a constructor?** No, interfaces cannot have constructors; they cannot be instantiated directly.
- **Can you instantiate an abstract class?** No, but it can have a constructor, called by its subclass via `super()`.
- **Is `final` variable always a constant?** Only if it holds a primitive or an immutable object. A `final` object reference can still have its internal state mutated.
- **What is the default value of an object reference field?** `null` (for instance/static fields; local variables have no default and must be initialized before use).
- **Can a class implement two interfaces with the same abstract method signature?** Yes, one implementation satisfies both.
- **Does `String s = "abc"` create an object on the heap?** Yes, inside the String pool, which is part of the heap since Java 7.
- **What's the difference between `Integer.valueOf()` and `new Integer()`?** `valueOf()` may return a cached instance; `new Integer()` (deprecated since Java 9) always creates a new object. Prefer `valueOf()` or autoboxing.
- **Can constructors be `final`, `static`, or `abstract`?** No, none of these apply to constructors.
- **What is covariant return type?** An overriding method can return a subtype of the return type declared in the parent method.
- **Does `this()` and `super()` work together in one constructor?** No, only one of them can be the first statement in a constructor, and only one can be used.
- **Why can't static methods access instance variables directly?** Because static methods belong to the class, and run without any specific object instance to reference.

## Common Traps & Gotchas

- **Comparing wrapper objects with `==`**: works "by accident" for -128 to 127 due to the Integer cache, fails outside that range. Always use `.equals()`.
- **`return` inside `finally`**: silently swallows exceptions and overrides return values from `try`/`catch`. Avoid it.
- **Autoboxing `null`**: unboxing a `null` wrapper (e.g., in a ternary mixing primitive and wrapper) throws `NullPointerException` unexpectedly.
- **Overriding `equals()` without `hashCode()`**: breaks `HashMap`/`HashSet` lookups silently — no compile error, just wrong runtime behavior.
- **Thinking `final` makes an object immutable**: `final List<String> list = new ArrayList<>();` — you cannot reassign `list`, but you can still `list.add(...)`. `final` only locks the reference, not the object's internal state.
- **Believing Java has pass-by-reference for objects**: it does not. Only the reference value is copied. Reassigning the parameter inside a method never affects the caller.
- **Assuming interface fields can be changed**: interface fields are implicitly `public static final` — always constants — even without writing those keywords.
- **String concatenation in a loop with `+`**: creates a new `String` object every iteration, causing O(n²) time and heavy garbage. Use `StringBuilder`.
- **Forgetting `super()` call ordering**: if a subclass constructor doesn't explicitly call `super(...)`, Java inserts a no-arg `super()` call automatically — this fails to compile if the parent class has no no-arg constructor.
- **Non-static inner classes causing memory leaks**: they hold an implicit reference to the outer object; if the inner object is kept alive (e.g., stored in a static collection or used as a long-lived listener), the outer object cannot be garbage collected.
- **Confusing marker interfaces with annotations**: both can "tag" a class, but only annotations can carry extra metadata (values), and only interfaces participate in `instanceof` checks.
- **Enum singleton and lazy initialization**: enum singletons are always eagerly initialized when the enum class is loaded, not lazily — usually fine, but a subtle difference from a lazy double-checked-locking singleton.
