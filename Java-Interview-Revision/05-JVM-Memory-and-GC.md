# 5. JVM, Memory & Garbage Collection

This topic covers how Java code runs inside the JVM and how memory is managed. Interviewers at 3–4 years expect you to know memory areas, class loading, and garbage collection in some depth. They also check if you can debug memory leaks and read basic GC logs.

## Key Concepts (Quick Revision)

- **JDK (Java Development Kit)**: Tools to write and build Java programs. It has a compiler (`javac`), debugger, and other dev tools. It includes JRE.
- **JRE (Java Runtime Environment)**: Everything needed to *run* a Java program. It has the JVM and standard libraries. No compiler.
- **JVM (Java Virtual Machine)**: The engine that runs compiled `.class` bytecode. It is platform-specific, but bytecode is not. This is why Java is "write once, run anywhere."
- **JVM memory areas**:
    - **Heap**: Stores all objects and arrays. Shared by all threads. Divided into young and old generation. Managed by GC.
    - **Stack**: One stack per thread. Stores stack frames — one frame per method call. Each frame holds local variables, method parameters, and partial results. Freed automatically when a method returns.
    - **Metaspace**: Stores class metadata (class structure, method info, runtime constant pool). Since Java 8. Lives in native (OS) memory, not the JVM heap.
    - **Program Counter (PC) register**: One per thread. Holds the address of the current instruction being executed.
    - **Native method stack**: Used for native (non-Java, e.g. C/C++) method calls, for example JNI calls.
- **PermGen removed in Java 8**: Before Java 8, class metadata lived in PermGen, a fixed-size part of heap. It often caused `OutOfMemoryError: PermGen space` because its size was hard to tune, especially with many classes (e.g. app servers reloading apps). Metaspace replaced it. Metaspace grows automatically using native memory, so it does not hit a fixed heap limit.
- **Stack vs heap, simple rule**: Local primitive variables and object *references* live on the stack. The actual objects (what the reference points to) live on the heap. Stack memory is freed when the method returns. Heap memory is freed by GC when no reference points to it.
- **Class loading process** — three steps:
    1. **Loading**: Reads the `.class` file bytes and creates a `Class` object in memory.
    2. **Linking**: Has three parts:
        - **Verify**: Checks bytecode is valid and safe (no corrupted or malicious code).
        - **Prepare**: Allocates memory for static fields and sets them to default values (like `0`, `null`).
        - **Resolve**: Replaces symbolic references (names) with direct references (memory addresses).
    3. **Initialization**: Runs static initializers and assigns actual values to static fields. Happens once, on first active use of the class.
- **Class loader hierarchy** (parent-first):
    - **Bootstrap class loader**: Loads core Java classes (`java.lang.*`, etc.) from the JDK. Written in native code.
    - **Platform/Extension class loader**: Loads classes from extension libraries.
    - **Application class loader**: Loads classes from the application classpath. Your own code is loaded here.
- **Delegation model**: A class loader first asks its parent to load a class. Only if the parent fails (`ClassNotFoundException`), it tries to load the class itself. This stops the same class being loaded twice and protects core classes from being overridden.
- **Object memory layout (high level)**: Every object on the heap has:
    - **Header**: Mark word (hash code, GC info, lock info) + class pointer (points to class metadata).
    - **Instance data**: Actual field values.
    - **Padding**: Extra bytes added so object size is a multiple of 8 bytes (for memory alignment).
- **GC roots**: Starting points GC uses to find live (reachable) objects. Examples: local variables on thread stacks, active threads, static fields, JNI references. Anything reachable from a GC root is "alive." Everything else is garbage.
- **Young vs old generation**:
    - **Young generation**: Where new objects are created. Split into Eden and two Survivor spaces (S0, S1). Most objects die young (weak generational hypothesis).
    - **Old (tenured) generation**: Objects that survive many young GC cycles get promoted here. Collected less often, but collection is more expensive.
- **Minor GC**: Cleans young generation only. Fast, frequent.
- **Major/Full GC**: Cleans old generation (major) or the whole heap including metaspace (full). Slower, less frequent, causes longer pauses.
- **Mark-Sweep-Compact**: The basic GC algorithm.
    - **Mark**: Find all reachable (live) objects starting from GC roots.
    - **Sweep**: Remove unreachable (dead) objects, freeing their memory.
    - **Compact**: Move live objects together to remove gaps (fragmentation), so allocation stays fast (bump-the-pointer).
- **Why generational GC**: Splitting heap into generations lets GC scan only the small young generation most of the time (cheap), instead of scanning the whole heap every time (expensive). This works well because most objects die quickly.
- **Main garbage collectors**:

| Collector | Style | Best for | Notes |
|---|---|---|---|
| Serial | Single-threaded, stop-the-world | Small apps, single CPU, client-side tools | Simplest, `-XX:+UseSerialGC` |
| Parallel (Throughput) | Multi-threaded, stop-the-world | Batch jobs, throughput over pause time | Old default JVM collector, `-XX:+UseParallelGC` |
| CMS (Concurrent Mark Sweep) | Mostly concurrent, low pause | Old low-latency needs | **Deprecated** (Java 9), removed in Java 14. Had fragmentation issues |
| G1 (Garbage First) | Region-based, concurrent + parallel | General purpose, balances throughput and pause time | Default since Java 9. Splits heap into many small regions, collects region with most garbage first |
| ZGC | Concurrent, region-based, colored pointers | Very large heaps, very low pause time (sub-millisecond target) | Production-ready since Java 15 |
| Shenandoah | Concurrent, low pause | Similar goal to ZGC, low pause on large heaps | From Red Hat, available in OpenJDK builds |

- **Stop-the-world (STW) pause**: All application threads are paused so GC can safely do its work (like moving objects) without the app changing things at the same time. Goal of modern collectors (G1, ZGC, Shenandoah) is to shrink STW pause time.
- **`System.gc()`**: Only a *suggestion* to the JVM to run GC. JVM can ignore it. Avoid calling it in production code — it can cause a full GC pause and hurt performance.
- **`finalize()`**: A method called by GC before reclaiming an object, meant for cleanup. **Deprecated since Java 9**, removed in later versions from practical use. Problems: no guarantee it runs, or when it runs; can slow down GC; can "resurrect" objects (make them reachable again). Use `try-with-resources` or `Cleaner` (Java 9+) instead.
- **Reference types** (control how GC treats an object):
    - **Strong reference**: Normal reference (`Object o = new Object()`). Object is never GC'd while a strong reference exists.
    - **Soft reference** (`SoftReference`): GC'd only when JVM really needs memory (before throwing `OutOfMemoryError`). Good for memory-sensitive caches.
    - **Weak reference** (`WeakReference`): GC'd on the next GC cycle if no strong reference exists, even if memory is not low. Used in `WeakHashMap` — keys are weak references, so entries are auto-removed once the key is no longer used elsewhere. Good for caches keyed by objects that should not be kept alive artificially.
    - **Phantom reference** (`PhantomReference`): `get()` always returns `null`. Used to know *after* an object is finalized and about to be reclaimed, mainly for cleanup actions via `ReferenceQueue`. Replaces `finalize()` in modern code (used internally by `Cleaner`).
- **Common memory leak causes in Java** (a "leak" here means objects stay reachable, so GC cannot free them, even though the app no longer needs them):
    - **Static collections**: A `static List`/`Map` keeps growing and never removes old entries.
    - **Unclosed resources**: DB connections, streams, sockets not closed — holds native/heap memory.
    - **ThreadLocal in thread pools**: Thread pool threads live long. If `ThreadLocal.remove()` is not called, the value stays attached to the pooled thread forever.
    - **Listener/callback leaks**: Registering a listener/observer but never unregistering it. The listener holds a reference to an object that should have been discarded.
- **Finding memory leaks**: Take a **heap dump** (`jmap`, or automatically on OOM with `-XX:+HeapDumpOnOutOfMemoryError`). Analyze it with a profiler like **Eclipse MAT**, VisualVM, or JProfiler. Look for objects with unexpectedly high count or a growing retained size, and check what is holding a reference to them (path to GC root).
- **OutOfMemoryError (OOM) types**:
    - `java.lang.OutOfMemoryError: Java heap space` — heap full, cannot fit new objects.
    - `java.lang.OutOfMemoryError: Metaspace` — too many classes loaded, or class loader leak.
    - `java.lang.OutOfMemoryError: GC overhead limit exceeded` — JVM spends too much time (98%+) doing GC and recovers too little memory.
    - `java.lang.StackOverflowError` — not technically OOM, but related: stack too deep, usually infinite/too-deep recursion. Controlled by `-Xss`.
    - `java.lang.OutOfMemoryError: Unable to create new native thread` — OS cannot allocate more native memory/threads.
- **JIT (Just-In-Time) compilation**:
    - JVM starts by **interpreting** bytecode line by line (slow but starts fast).
    - **HotSpot** JVM tracks which methods run often ("hot" methods).
    - Hot methods get compiled to native machine code by the JIT compiler, so future calls run much faster.
    - **C1 (client compiler)**: Compiles quickly, does basic optimizations. Good for fast startup.
    - **C2 (server compiler)**: Compiles slower, but does aggressive optimizations. Good for long-running apps.
    - **Tiered compilation** (default since Java 8): Uses C1 first, then recompiles hot methods with C2 as they get hotter. Gets both fast startup and strong peak performance.
- **Basic JVM flags**:
    - `-Xms<size>`: Initial heap size (e.g. `-Xms512m`).
    - `-Xmx<size>`: Maximum heap size (e.g. `-Xmx2g`).
    - `-Xss<size>`: Stack size per thread (e.g. `-Xss512k`). Increase if you get `StackOverflowError` from legitimate deep recursion.

## Important Interview Questions

**Q1: What is the difference between JDK, JRE, and JVM?**
JVM runs bytecode. JRE = JVM + standard libraries, needed to run apps. JDK = JRE + compiler and dev tools, needed to build apps.
*Follow-up: Can you run a `.jar` file with just JRE?* Yes, JRE has everything needed to run compiled code.

**Q2: Explain the JVM memory areas.**
Heap (objects, shared), stack (per-thread, method frames, local variables), metaspace (class metadata, native memory), PC register (per-thread, current instruction), native method stack (native calls).
*Follow-up: Which of these can cause `OutOfMemoryError`?* Heap and metaspace. Stack overflow gives `StackOverflowError`, not `OutOfMemoryError` (except the native thread creation case).

**Q3: Why was PermGen removed and replaced with Metaspace?**
PermGen had a fixed max size set at JVM start, hard to size correctly, and caused frequent `OutOfMemoryError: PermGen space`, especially in apps that load/reload many classes. Metaspace uses native memory and grows automatically (up to `-XX:MaxMetaspaceSize` if set), so it's much less likely to run out with a wrong static setting.
*Follow-up: Is Metaspace unlimited?* Not truly. It uses native memory. If unbounded, it can exhaust OS memory. You can cap it with `-XX:MaxMetaspaceSize`.

**Q4: Walk me through the class loading process.**
Loading (read `.class`, create `Class` object) → Linking: verify (check bytecode safety), prepare (allocate memory, default values for statics), resolve (symbolic references → direct references) → Initialization (run static blocks, assign static field values).
*Follow-up: When does initialization happen exactly?* Lazily, on first active use — like creating an instance, calling a static method, or accessing a static field (not a constant).

**Q5: Explain the class loader delegation model. Why does it matter?**
Each class loader asks its parent to load a class first. Only if the parent cannot find it does the child try. Bootstrap → Extension/Platform → Application. This prevents duplicate loading and stops user code from overriding core classes like `java.lang.String` by accident (or on purpose, maliciously).
*Follow-up: Can you break delegation?* Yes, by writing a custom class loader that loads first before asking the parent. Some frameworks (Tomcat, OSGi) do this on purpose for module isolation.

**Q6: What are GC roots? Why do we need them?**
GC roots are starting references GC uses to decide what is "alive" — like local variables on stacks, static fields, active thread objects. GC walks from these roots. Anything not reachable from any root is garbage and can be collected. Without roots, GC would have no safe starting point to decide reachability.

**Q7: Why does Java use generational garbage collection?**
Most objects die young (weak generational hypothesis) — e.g. temporary objects in a loop. So splitting heap into young and old generations lets GC do frequent, cheap scans of just the young generation (minor GC), instead of always scanning the whole heap. Only surviving objects get promoted to old generation, which is scanned less often.

**Q8: What's the difference between minor GC and full GC? Why does full GC take longer?**
Minor GC cleans young generation only, and is fast and frequent. Full GC cleans the entire heap (young + old) and sometimes metaspace, and is slower because old generation is usually much bigger and has more live objects to scan and possibly compact. Full GC usually causes a longer stop-the-world pause.

**Q9: Compare G1 and CMS. Why is CMS deprecated?**
CMS does concurrent marking to reduce pause time, but it does not compact the heap during normal cycles, causing fragmentation over time, which can force a slow full GC. It was also becoming hard for JVM developers to maintain alongside newer collectors. G1 is region-based, does compaction, and is more predictable at meeting pause-time goals (`-XX:MaxGCPauseMillis`). G1 replaced CMS as default from Java 9, and CMS was removed in Java 14.

**Q10: When would you pick ZGC or Shenandoah over G1?**
When the app needs very large heaps (tens/hundreds of GB) and very low pause times (sub-millisecond to low-millisecond), for example latency-sensitive trading or real-time systems. G1 is a good general default; ZGC/Shenandoah trade some throughput for much lower and more consistent pause times.

**Q11: Does calling `System.gc()` guarantee garbage collection?**
No. It only requests GC. The JVM can ignore the request. It is generally discouraged in production, because if it does trigger a full GC, it can cause a noticeable pause for no guaranteed benefit.

**Q12: Why is `finalize()` deprecated? What should be used instead?**
`finalize()` gives no guarantee on when (or if) it runs, can delay GC, can throw exceptions that are swallowed silently, and can "resurrect" an object by re-adding a reference to it inside `finalize()`. Since Java 9 it's deprecated. Use `try-with-resources` with `AutoCloseable` for deterministic cleanup, or `java.lang.ref.Cleaner` for GC-triggered cleanup.

**Q13: Explain the four reference types and one real use case each.**
Strong (default, normal usage), Soft (`SoftReference`, memory-sensitive caches, cleared only under memory pressure), Weak (`WeakReference`, `WeakHashMap` — auto-remove entries when key is unused elsewhere), Phantom (`PhantomReference`, post-mortem cleanup via `ReferenceQueue`, used internally by `Cleaner`).
*Follow-up: How does `WeakHashMap` prevent memory leaks?* Its keys are wrapped in weak references. Once no strong reference to a key exists elsewhere, GC can collect the key, and the entry is removed from the map automatically.

**Q14: Give real examples of memory leaks in a Spring Boot application, and how would you debug one.**
Examples: a `static Map` cache that never expires entries; a singleton bean holding a growing `List` across requests; `ThreadLocal` set in a request-scoped filter but not cleared, leaking into pooled worker threads; event listeners registered but never removed. To debug: reproduce heap growth, take a heap dump (`jmap -dump` or auto on OOM), open it in Eclipse MAT or VisualVM, find the objects with the largest retained size, and trace the path to GC roots to find what's holding them.

**Q15: What is the difference between `OutOfMemoryError: Java heap space` and `GC overhead limit exceeded`?**
`Java heap space` means heap is full and a new allocation cannot fit even after GC. `GC overhead limit exceeded` means GC keeps running (recovering very little memory each time, JVM default threshold is 98% of time spent in GC for <2% heap recovered) — the JVM decides the app is thrashing and stops it early, instead of hanging forever doing useless GC cycles.

**Q16: What is JIT compilation? Why does the JVM interpret first instead of compiling everything upfront?**
JIT compiles bytecode of "hot" (frequently run) methods into native machine code at runtime, so they run much faster on later calls. The JVM interprets first because compiling everything upfront (like a fully static compiler) would slow down startup and would waste time compiling code that runs only once. Interpreting lets the app start fast, then JIT optimizes only what matters.
*Follow-up: What's the difference between C1 and C2?* C1 compiles fast with light optimization (favors quick startup), C2 compiles slower with heavy optimization (favors peak throughput for long-running code). Tiered compilation uses both together.

**Q17: What do `-Xms` and `-Xmx` control? What happens if you set them to different values vs the same value?**
`-Xms` is the initial heap size, `-Xmx` is the max heap size. If they differ, the heap can grow/shrink at runtime as needed, which can cause a resize pause. Setting them equal avoids runtime resizing, giving more predictable performance, common in production tuning.

## FAQ / Rapid-Fire

- **Is metaspace part of the heap?** No, it uses native (off-heap) memory.
- **What's on the stack — object or reference?** Reference (and primitives). The object itself is on the heap.
- **Are static variables stored on the heap?** The values live in metaspace as part of class metadata (in modern JVMs); if a static variable is an object reference, the reference is in metaspace but the object itself is on the heap.
- **What is String pool and where does it live?** It is a special area inside the heap holding interned string literals, since Java 7 (before that it was in PermGen).
- **What triggers a minor GC?** Eden space becomes full.
- **What triggers promotion of an object to old generation?** It survives enough minor GC cycles (age threshold), or it's too large to fit in young generation.
- **Is G1 concurrent or stop-the-world?** Both — it does some work concurrently with the app, but still has short stop-the-world pauses for parts like the final marking and evacuation.
- **Default collector in modern Java (9+)?** G1.
- **Can you force full GC to run?** Not with certainty. `System.gc()` is just a hint.
- **Does `finalize()` run guaranteed before JVM exits?** No guarantee at all — might never run.
- **What is a "dangling" thread-local leak?** When a `ThreadLocal` value is not removed, and the thread is reused from a pool, the value stays attached to that thread indefinitely.
- **Difference between memory leak and OOM?** A leak is unused objects staying reachable over time; OOM is the actual crash when no more memory can be allocated. A leak, if it keeps growing, usually eventually causes OOM.
- **What tool shows live heap usage graphically?** VisualVM, JConsole, or JFR (Java Flight Recorder) with JMC.
- **What does `-XX:+HeapDumpOnOutOfMemoryError` do?** Automatically writes a heap dump file when the JVM throws `OutOfMemoryError`, useful for post-mortem debugging.
- **Can an object be garbage collected while still in scope (variable holding it)?** No — if a strong reference is reachable from a GC root, the object cannot be collected, no matter how big it is.

## Common Traps & Gotchas

- Saying "Java is garbage collected so leaks are impossible." Wrong — leaks happen whenever objects stay reachable but are logically unused (static caches, listeners, `ThreadLocal`).
- Confusing `StackOverflowError` with `OutOfMemoryError`. They are different errors from different memory areas (stack vs heap/metaspace).
- Thinking `System.gc()` forces GC immediately. It's a request; JVM may ignore it.
- Thinking finalize() is a safe way to release resources (file handles, sockets). It's unreliable and deprecated; use `try-with-resources`.
- Believing PermGen still exists in modern Java. It was removed in Java 8, replaced by Metaspace.
- Assuming CMS is still a good default choice. It's deprecated (Java 9) and removed (Java 14); use G1 or ZGC instead.
- Thinking a `WeakHashMap` clears values eagerly the moment a key becomes weakly reachable. Removal happens lazily, usually on next map access after GC has collected the key.
- Assuming larger heap (`-Xmx`) always means better performance. A bigger heap can mean longer GC pauses during full GC, since there is more to scan/compact.
- Forgetting that class loader delegation is parent-first by default — assuming your custom class loader's classes are used over JDK's own automatically.
- Mixing up "major GC" and "full GC" as always meaning the same thing. Usage varies; many people use them interchangeably, but full GC often specifically includes metaspace and the entire heap, while major GC can refer to just old-gen collection depending on the collector.
- Believing JIT compiles code once and for good at startup. It compiles methods only after detecting they are "hot," at runtime, and even then can deoptimize and go back to the interpreter if assumptions break (e.g. after class loading changes).
