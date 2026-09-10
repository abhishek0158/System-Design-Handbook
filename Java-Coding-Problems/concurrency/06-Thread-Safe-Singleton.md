# Thread-safe Singleton

## Problem

Design a Singleton class in Java. A Singleton is a class that allows only one instance in the whole application. Many threads may ask for the instance at the same time. Your job is to make sure all threads get the same, fully-built instance. No thread should ever see a broken or half-built object.

This is a very common interview question for 3–4 years experience. Interviewers do not just want "add `synchronized`". They want you to know the different ways to build a Singleton, and to explain the trade-offs of each way.

## Requirements & Clarifying Questions

Before coding, ask these questions:

1. Do we need lazy initialization (create the object only when first needed), or is eager initialization (create it at class load time) fine? This depends on how expensive the object is to build.
2. Will the Singleton be serialized (converted to bytes and back)? If yes, we must stop a new instance from being created during deserialization.
3. Do we need to defend against reflection attacks? Reflection is a Java feature that lets code call a private constructor directly, bypassing our normal checks.
4. Is the class loaded by a single classloader, or could there be multiple classloaders? Multiple classloaders can create multiple "singleton" instances — an edge case worth mentioning, but not always in scope.
5. Do we need high read concurrency, meaning many threads calling `getInstance()` very often? If yes, the performance of the read path after the instance is created matters a lot.

For this sheet, we assume: lazy initialization is preferred, many threads call `getInstance()` concurrently, and we want protection against both reflection and serialization attacks.

## Design / Approach

There are five common ways to write a thread-safe Singleton in Java, in order of maturity:

1. Eager initialization — simplest, not lazy.
2. Lazy initialization with a `synchronized` method — simple, but slow.
3. Double-checked locking with `volatile` — fast, but tricky to get right.
4. Bill Pugh static inner holder class — lazy, fast, and simple. The recommended approach for plain classes.
5. Enum singleton — the safest option overall, because the JVM (Java Virtual Machine) itself protects it from reflection and serialization problems.

We will also cover how reflection and serialization can break a singleton, and how each pattern defends against these attacks.

## Java Solution

### 1. Eager Initialization

The instance is created when the class is loaded, not when it is first used. The JVM guarantees that static field initialization happens once, safely, before any thread can use the class. So this is thread-safe by default, with no extra code needed.

```java
public final class EagerSingleton {

    // Created once, when the class is loaded by the JVM.
    private static final EagerSingleton INSTANCE = new EagerSingleton();

    private EagerSingleton() {
    }

    public static EagerSingleton getInstance() {
        return INSTANCE;
    }
}
```

**Pros:** Very simple. Thread-safe with no locks. No risk of partially-built objects.

**Cons:** The instance is created even if the application never calls `getInstance()`. This wastes memory and startup time if the object is expensive to build (for example, it opens a database connection or reads a large file).

### 2. Lazy Initialization with `synchronized` Method

Here we create the instance only when someone first calls `getInstance()`. To make this safe for many threads, we mark the whole method `synchronized`. A `synchronized` method can be entered by only one thread at a time; other threads must wait.

```java
public final class SynchronizedSingleton {

    private static SynchronizedSingleton instance;

    private SynchronizedSingleton() {
    }

    public static synchronized SynchronizedSingleton getInstance() {
        if (instance == null) {
            instance = new SynchronizedSingleton();
        }
        return instance;
    }
}
```

**Pros:** Simple to write. Correct — no thread can ever see a half-built object, and only one instance is ever created.

**Cons:** It is slow under heavy read load. Every single call to `getInstance()` must acquire a lock, even after the instance already exists and there is nothing left to build. If 1,000 threads call `getInstance()` every second, all 1,000 calls go through the lock, one at a time. This turns a fast read into a serialized (one-at-a-time) operation, which hurts performance a lot in high-concurrency systems.

### 3. Double-Checked Locking with `volatile`

The idea: only take the lock the first time, when the instance does not exist yet. After the instance is built, later calls should skip the lock completely. This is called "double-checked locking" because we check `instance == null` twice: once without the lock (fast path), and once inside the lock (safe path).

```java
public final class DoubleCheckedSingleton {

    // volatile is required here. See explanation below.
    private static volatile DoubleCheckedSingleton instance;

    private DoubleCheckedSingleton() {
    }

    public static DoubleCheckedSingleton getInstance() {
        DoubleCheckedSingleton result = instance;      // first read
        if (result == null) {
            synchronized (DoubleCheckedSingleton.class) {
                result = instance;                      // second read, inside lock
                if (result == null) {
                    instance = result = new DoubleCheckedSingleton();
                }
            }
        }
        return result;
    }
}
```

**Why `volatile` is required here — the full explanation:**

The line `new DoubleCheckedSingleton()` looks like one step, but the JVM breaks it into three steps internally:

1. Allocate memory for the new object.
2. Run the constructor, to set up the object's fields.
3. Set the `instance` variable to point to the new memory address.

Without `volatile`, the JVM and the CPU (central processing unit) are allowed to reorder steps 2 and 3. This is called "instruction reordering." It is done for performance, and it is legal under the Java Memory Model as long as a single thread cannot tell the difference. But other threads can tell the difference.

Imagine step 3 runs before step 2. Now `instance` is not `null` anymore, but the constructor has not finished yet. If another thread calls `getInstance()` at this exact moment, it reads `instance` in the first check, sees it is not `null`, and returns it right away, skipping the lock. But the object is only partially built — some fields may still hold default values. This is the "partially-constructed object" problem. The second thread now works with a broken object, and this kind of bug is hard to reproduce, because it only shows up under rare timing.

Marking `instance` as `volatile` fixes this in two ways:

- It stops the compiler and CPU from reordering the write to `instance` ahead of the constructor call. This is a "happens-before" guarantee: everything inside the constructor is guaranteed to finish, and be visible, before the `instance` field is set.
- It makes the write to `instance` immediately visible to all other threads. Without `volatile`, one thread's write might sit in that CPU's local cache for a while, and other threads reading `instance` might still see the old `null` value, even after the assignment already happened.

So `volatile` solves two problems at once: reordering, and cross-thread visibility. Without it, double-checked locking is broken, even though the code looks correct.

**Pros:** Fast after the instance is created — the common case skips the lock. Correct, when `volatile` is used properly.

**Cons:** The code is harder to read and easy to get wrong (for example, forgetting `volatile`, or forgetting the second null check inside the lock). Many interviewers ask this question specifically to test if the candidate remembers `volatile` and can explain why.

### 4. Bill Pugh Static Inner Holder Class (Recommended)

This pattern uses a nested static class to hold the instance. The nested class is only loaded — and its static field only initialized — when it is first accessed, which happens the first time `getInstance()` is called.

```java
public final class HolderSingleton {

    private HolderSingleton() {
    }

    // Not loaded until getInstance() is called for the first time.
    private static final class Holder {
        private static final HolderSingleton INSTANCE = new HolderSingleton();
    }

    public static HolderSingleton getInstance() {
        return Holder.INSTANCE;
    }
}
```

**Why this is thread-safe without any explicit locks:**

Class loading in Java is handled by the JVM's classloader, and the JVM already guarantees that class initialization is thread-safe. When multiple threads try to load and initialize the same class at the same time, the JVM makes sure only one thread actually runs the static initializer, and all other threads wait until it is done. This is a rule from the Java Language Specification, not something we coded ourselves — we get correctness for free.

Because the `Holder` class is not touched until `getInstance()` runs, the outer class `HolderSingleton` loads without building the singleton. This gives us:

- **Laziness**: the instance is built only when needed, just like the `synchronized` and double-checked versions.
- **No explicit locking on the read path**: after the class is loaded, reading `Holder.INSTANCE` is just a normal static field read — no `synchronized` block, no `volatile` overhead.
- **Simplicity**: no need to reason about `volatile` or reordering. The JVM's class-loading lock does that work for us.

This is why the Bill Pugh idiom is usually the best default choice for a plain (non-enum) Java Singleton: it is lazy, thread-safe, and fast, with the simplest code.

### 5. Enum Singleton (Safest Overall)

Joshua Bloch (author of *Effective Java*) recommends using a single-element `enum` as the best way to implement a Singleton in Java.

```java
public enum EnumSingleton {
    INSTANCE;

    public void doWork() {
        // business logic goes here
    }
}
```

Usage: `EnumSingleton.INSTANCE.doWork();`

**Why this is the safest choice:**

- **Thread-safety**: the JVM guarantees that enum constants are created exactly once, at class loading time, in a thread-safe way — the same class-loading guarantee used by the Holder pattern.
- **Serialization safety**: normally, deserializing a class (rebuilding an object from saved bytes) calls a private mechanism that can create a brand new object, bypassing the constructor entirely. This would produce a second instance, breaking the Singleton guarantee. For `enum` types, the Java serialization mechanism is special-cased: the JVM never creates a new enum instance during deserialization. It always returns the existing constant, matched by name. So enum singletons are automatically safe against this attack, with no extra code.
- **Reflection safety**: Java's reflection API blocks calling a constructor for an `enum` type. If you try `Constructor.newInstance()` on an enum, the JVM throws `IllegalArgumentException` at runtime, before your code even runs. This protection is built into the `Constructor` class itself, so we do not need to write any defensive code.

**Cons:** Some developers find `enum` singletons unusual to read at first, especially if the class needs to extend another class (Java does not allow an enum to extend a class, though it can implement interfaces). Also, if the object needs lazy, expensive setup that should not run at class-load time, you may prefer the Holder pattern instead.

## How It Works

All five patterns solve the same core problem — make sure only one instance exists, and every thread sees a fully-built instance — but they use different tools:

- **Eager** and **Enum** rely on the JVM's class-loading guarantee, applied to a static field or enum constant, so the object is ready before any thread can reach it.
- **Synchronized method** relies on a lock held for every call, trading performance for simplicity.
- **Double-checked locking** relies on `volatile` to fix visibility and ordering, so the lock is needed only once.
- **Holder class** relies on lazy class loading plus the same JVM guarantee used by Eager and Enum, giving laziness without any lock in the code.

## How to Extend (Follow-ups)

Interviewers often push further. Common follow-ups:

- **"How does reflection break a singleton, and how do you stop it?"**
  Reflection lets code call `Constructor.setAccessible(true)` and then invoke a private constructor directly, creating a second instance. Defense: in the constructor, check if an instance already exists, and throw an exception if it does. This has a small chicken-and-egg problem with the Holder pattern's own reference, so a separate boolean flag is often used instead. Enum is immune to this attack by design, with no extra code.

- **"How does serialization break a singleton, and how do you stop it?"**
  If the Singleton class implements `Serializable`, Java's default deserialization creates a new object using low-level tricks that skip the constructor. Defense: add a `readResolve()` method that returns the existing instance instead of the new one:
  ```java
  protected Object readResolve() {
      return getInstance();
  }
  ```
  Enum types need no such fix — the JVM handles this automatically.

- **"What about cloning?"** `clone()` can also create a second instance. Defense: override `clone()` and throw `CloneNotSupportedException`.

- **"What if the Singleton needs constructor arguments, like a config object?"** Use a static `initialize(Config config)` method, called once at startup before any `getInstance()` call, and store the config in the instance. Guard against double initialization with a check or an `AtomicBoolean`.

- **"What about multiple classloaders?"** Each classloader can load its own copy of the same class, giving multiple "singleton" instances, one per classloader. This is rare, but worth mentioning as a known limitation in application servers or OSGi environments.

## Complexity & Thread-Safety Notes

`getInstance()` is O(1) — constant time — in all five patterns, once the instance exists. The real difference is the constant-factor cost per call, not algorithmic complexity:

- **Eager**: no lock ever, but the instance is built at class-load time even if unused.
- **Synchronized method**: every call pays for acquiring a lock. Under heavy concurrent read load, this becomes a bottleneck because calls are serialized (run one at a time).
- **Double-checked locking**: no lock on the common path after first creation. Requires `volatile` for correctness — without it, this pattern has a race condition that can hand out a partially-built object.
- **Holder class**: no lock in code at all; correctness comes from the JVM's class initialization lock, used only once, during class loading.
- **Enum**: same JVM guarantee as Holder, plus automatic protection from reflection and serialization.

All patterns here avoid the partially-constructed-object problem, as long as `volatile` is used correctly in the double-checked version.

## Interview Tips & Common Mistakes

- Do not just say "make the method `synchronized`" and stop. Interviewers at 3–4 years level expect you to know the performance cost, and to offer the Holder or double-checked alternative.
- If you write double-checked locking, always explain **why** `volatile` is needed. Many candidates write the code correctly but cannot explain the reordering problem — this is often the actual test.
- Do not forget the **second null check inside the synchronized block** in double-checked locking. Without it, two threads could both pass the first check, then both build a new instance, defeating the whole pattern.
- Mention that the Holder pattern is often the best default answer for plain classes — it is simple, lazy, and needs no `volatile` or manual locking.
- Know that `enum` singleton is Joshua Bloch's recommendation, and know the two specific reasons: JVM-level reflection blocking, and automatic serialization safety via constant matching.
- A small but real mistake: making the constructor public, or forgetting `private`. A Singleton constructor must always be `private` (enum constructors are implicitly private).
- Remember: "thread-safe" for a Singleton means two things together — only one instance is ever created, and every thread that reads the instance sees a fully-built, correct object. Many candidates only think about the first point and forget the second, which is exactly what the `volatile` explanation covers.
