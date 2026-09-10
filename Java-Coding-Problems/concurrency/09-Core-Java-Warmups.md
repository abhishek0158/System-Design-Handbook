# Core Java Warm-ups (equals/hashCode, Immutable Class, Comparator)

These are three small questions. Interviewers ask them as warm-ups before the main design question. They look easy. But most candidates make small mistakes. At 3–4 years experience, you are expected to get every detail right.

## equals() and hashCode()

### Problem

Write a `Point` class with two fields, `x` and `y`. Override `equals()` and `hashCode()` so two `Point` objects with the same `x` and `y` are treated as equal. Explain the rules behind these two methods.

### Java Solution

```java
import java.util.Objects;

public final class Point {

    private final int x;
    private final int y;

    public Point(int x, int y) {
        this.x = x;
        this.y = y;
    }

    public int getX() {
        return x;
    }

    public int getY() {
        return y;
    }

    @Override
    public boolean equals(Object o) {
        if (this == o) {
            return true;
        }
        if (o == null || getClass() != o.getClass()) {
            return false;
        }
        Point other = (Point) o;
        return x == other.x && y == other.y;
    }

    @Override
    public int hashCode() {
        return Objects.hash(x, y);
    }

    @Override
    public String toString() {
        return "Point{x=" + x + ", y=" + y + "}";
    }
}
```

Walk through the `equals()` method line by line:

1. `if (this == o) return true;` — This is a fast check. If both references point to the same object, they are equal. No need to check fields.
2. `if (o == null || getClass() != o.getClass()) return false;` — This handles two cases at once. If `o` is `null`, they cannot be equal. If `o` is a different class (for example, a subclass), we reject it. Using `getClass()` instead of `instanceof` keeps the equals contract symmetric when subclasses add new fields. Some teams prefer `instanceof` for flexibility with subclasses — both are valid choices, but `getClass()` is safer by default.
3. Cast `o` to `Point` and compare each field.

For `hashCode()`, `Objects.hash(x, y)` builds a combined hash from both fields. It is short and correct. Under the hood it does something like `31 * (31 * 1 + x) + y`. You do not need to write this by hand. `Objects.hash` is idiomatic in modern Java and is easy to read in a review.

`Objects.equals(a, b)` is also useful. It compares two objects and returns `true` if both are `null`, or if `a.equals(b)` is `true`. It saves you from writing null checks by hand. Use it when a field can be `null`, for example a `String name` field:

```java
return Objects.equals(name, other.name) && x == other.x;
```

### Key Points & Gotchas

**The equals() contract.** Java says `equals()` must follow five rules:

- **Reflexive**: `a.equals(a)` must be `true`.
- **Symmetric**: if `a.equals(b)` is `true`, then `b.equals(a)` must also be `true`.
- **Transitive**: if `a.equals(b)` is `true` and `b.equals(c)` is `true`, then `a.equals(c)` must be `true`.
- **Consistent**: repeated calls to `a.equals(b)` must return the same result, as long as the fields used in the comparison do not change.
- **Null comparison**: `a.equals(null)` must return `false`, never throw an exception.

**The hashCode() contract.** There is one rule that matters most: **if two objects are equal according to `equals()`, they must return the same `hashCode()`.** The reverse is not required. Two unequal objects can share a hash code (this is called a "hash collision", and it is allowed).

**What breaks if you override equals() but not hashCode().** This is the classic interview trap. Say you override `equals()` only, and leave the default `hashCode()` from `Object` (which is based on memory address). Now:

```java
Set<Point> points = new HashSet<>();
points.add(new Point(1, 2));
System.out.println(points.contains(new Point(1, 2))); // prints false!
```

This looks wrong, but it is expected behavior. `HashSet` and `HashMap` use `hashCode()` first to pick a "bucket" to search in. Only within that bucket do they call `equals()` to check for a match. Since the two `Point` objects have different default hash codes (different memory addresses), the set looks in the wrong bucket and never calls `equals()`. It reports `contains() = false`, even though logically the two points are equal.

**Rule to remember**: always override `hashCode()` when you override `equals()`. Most IDEs (IntelliJ, Eclipse) can generate both together — use that to avoid mistakes. Lombok's `@EqualsAndHashCode` annotation does the same job if your project uses Lombok.

---

## Immutable Class

### Problem

Write a fully immutable `Employee` class. It should hold a name, a salary, and a `List<String>` of skills. Once created, no part of an `Employee` object should be changeable, even indirectly.

### Java Solution

```java
import java.util.ArrayList;
import java.util.Collections;
import java.util.List;

public final class Employee {

    private final String name;
    private final double salary;
    private final List<String> skills;

    public Employee(String name, double salary, List<String> skills) {
        this.name = name;
        this.salary = salary;
        // Defensive copy: protect against the caller changing the list later.
        this.skills = new ArrayList<>(skills);
    }

    public String getName() {
        return name;
    }

    public double getSalary() {
        return salary;
    }

    public List<String> getSkills() {
        // Return an unmodifiable view, so callers cannot mutate our internal list.
        return Collections.unmodifiableList(skills);
    }
}
```

### Key Points & Gotchas

An immutable object cannot change state after it is built. Follow these rules to make a class immutable:

1. **Mark the class `final`.** This stops other classes from extending it and adding mutable behavior.
2. **Mark every field `private final`.** `private` hides the field from outside code. `final` means the field can be set only once, in the constructor.
3. **Do not write setters.** A setter is a method that changes a field after construction. An immutable class has none.
4. **Set every field in the constructor.** All values must be provided at creation time.
5. **Make defensive copies of mutable fields.** This is the step most people forget. `String` and `double` are immutable types already, so they are safe to store directly. But `List`, `Date`, arrays, and other mutable types are not safe. If you store the caller's list directly, the caller can still change it after the fact:

```java
List<String> skills = new ArrayList<>(List.of("Java", "SQL"));
Employee emp = new Employee("Asha", 90000, skills);
skills.add("Python"); // this would silently change emp's internal skills, if we
                       // had NOT made a defensive copy in the constructor!
```

Copying the list inside the constructor (`new ArrayList<>(skills)`) breaks this link. The `Employee` object now owns its own private copy.

6. **Return defensive copies (or unmodifiable views) from getters too.** Even if the constructor makes a safe copy, a getter that returns the raw internal list still leaks a mutable reference:

```java
emp.getSkills().add("Python"); // without protection, this mutates emp's internal state!
```

`Collections.unmodifiableList(skills)` wraps the list so any attempt to modify it throws `UnsupportedOperationException`. This protects the object's internal state.

**Why immutability helps with thread-safety.** A mutable object shared between two threads can run into a "race condition" — a bug where the result depends on the timing of thread execution. One thread might read a field while another thread is halfway through updating it. To avoid this safely, you normally need locks (`synchronized`) or other coordination.

An immutable object removes this problem completely. Once built, its state never changes. Multiple threads can read it at the same time, with no lock needed, and no risk of seeing a half-updated value. This is why immutable classes like `String` are considered safe to share freely across threads.

**Records as a modern shortcut.** Since Java 16, you can use a `record` to get most of this for free:

```java
public record EmployeeRecord(String name, double salary, List<String> skills) {
    public EmployeeRecord {
        skills = List.copyOf(skills); // compact constructor: validate/copy here
    }
}
```

A `record` auto-generates a constructor, `equals()`, `hashCode()`, `toString()`, and read-only accessor methods (`name()`, `salary()`, `skills()`). Fields are `private final` by default. But a record does **not** automatically make mutable fields safe — you still must copy a `List` yourself, as shown above, usually inside a "compact constructor" (the constructor block with no parameter list, used only for validation or defensive copying). Records are a good shortcut for simple data carriers, but understanding the manual rules above is still needed for anything more complex, or for interviews.

---

## Custom Comparator

### Problem

You have a `List<Employee>` (name, salary, age). Sort it by salary descending, and for employees with the same salary, sort by name ascending. Show how to do this with `Comparator`, and explain how it differs from `Comparable`.

### Java Solution

```java
import java.util.Comparator;
import java.util.List;

public class Employee {
    private final String name;
    private final double salary;
    private final Integer age; // Integer, not int, to allow null in this example

    public Employee(String name, double salary, Integer age) {
        this.name = name;
        this.salary = salary;
        this.age = age;
    }

    public String getName() { return name; }
    public double getSalary() { return salary; }
    public Integer getAge() { return age; }

    @Override
    public String toString() {
        return name + "(" + salary + ", age=" + age + ")";
    }
}
```

Sorting logic:

```java
List<Employee> employees = ... ; // some list

Comparator<Employee> bySalaryThenName =
        Comparator.comparingDouble(Employee::getSalary)
                   .reversed()                          // salary descending
                   .thenComparing(Employee::getName);    // name ascending, tie-break

employees.sort(bySalaryThenName);
```

If `age` can be `null` and you want to sort by age too, with `null` values placed first:

```java
Comparator<Employee> byAgeNullsFirst =
        Comparator.comparing(Employee::getAge, Comparator.nullsFirst(Comparator.naturalOrder()));

employees.sort(byAgeNullsFirst);
```

You can chain all three together:

```java
Comparator<Employee> fullOrder =
        Comparator.comparingDouble(Employee::getSalary).reversed()
                   .thenComparing(Employee::getName)
                   .thenComparing(Employee::getAge, Comparator.nullsFirst(Comparator.naturalOrder()));

employees.sort(fullOrder);
```

### Key Points & Gotchas

**`Comparator.comparing(...)`** builds a comparator from a "key extractor" — a method reference that pulls out the field to sort by. `Comparator.comparingDouble` and `Comparator.comparingInt` are specialized versions for primitive `double` and `int`, which avoid unnecessary boxing (wrapping a primitive into an object like `Double` or `Integer`) and are slightly more efficient.

**`.reversed()`** flips the order of the comparator it is called on. Call it right after the first `comparing(...)` if you want that one field sorted in descending order. Note that `.reversed()` only reverses the comparator on which you call it — placing it at the end of a chain reverses the whole combined order, not just the last field. So `.reversed()` after the first `comparing` reverses only salary; a `.reversed()` at the very end would reverse everything, including the name tie-break.

**`.thenComparing(...)`** adds a tie-break rule. It runs only when the earlier comparator says two elements are equal (returns `0`). You can chain as many `thenComparing` calls as you need.

**`Comparator.nullsFirst(...)` and `Comparator.nullsLast(...)`** wrap another comparator to handle `null` values safely. Without this wrapper, comparing a `null` field throws a `NullPointerException`. `nullsFirst` places `null` values at the start of the sorted list; `nullsLast` places them at the end. Pass the "real" comparator (for example, `Comparator.naturalOrder()`) as the argument, so non-null values are still ordered correctly among themselves.

**`Comparable` vs `Comparator` — the core difference:**

- **`Comparable<T>`** is a single method, `compareTo(T other)`, implemented **inside** the class itself. It defines the class's one "natural ordering". For example, `String` implements `Comparable<String>` with alphabetical order as its natural order. A class can have only one natural ordering.
- **`Comparator<T>`** is a separate object, defined **outside** the class. It can express any number of orderings — by salary, by name, by age, or any combination. You do not need to modify the target class at all. This is why `Comparator` is more flexible: you often need different sort orders in different parts of an application, and you cannot express all of them through a single `compareTo()` method.

A simple rule for interviews: use `Comparable` when there is one obvious, default way to order objects of that class. Use `Comparator` for every other case, especially when sorting rules can change per use case, or when you cannot modify the class's source code (for example, a class from a third-party library).

Both `compareTo()` and `compare()` return an `int`: negative if the first argument is smaller, zero if equal, positive if larger. `Collections.sort()` and `List.sort()` accept either style — a class that implements `Comparable` can be sorted with no arguments (`list.sort(null)` or `Collections.sort(list)`), while an external `Comparator` is passed explicitly (`list.sort(comparator)`).
