# Logging Framework (LLD)

## Problem

Design and build a small logging framework in Java. Other developers will call it from their code, like this:

```java
Logger logger = LoggerFactory.getLogger(MyClass.class);
logger.info("User {} logged in", userId);
logger.error("Payment failed", exception);
```

The framework must support log levels, more than one output target (console, file), and pluggable message formatting. It should work correctly when many threads log at the same time. This is a common machine-coding question. It tests your knowledge of design patterns (Strategy, Chain of Responsibility, Singleton, Factory) and concurrency.

## Requirements & Clarifying Questions

Before coding, an interviewer expects you to ask questions. Here are the important ones, with the answers we assume for this design.

**Functional requirements**

1. Support 4 log levels: `DEBUG`, `INFO`, `WARN`, `ERROR`. Order of severity: `DEBUG < INFO < WARN < ERROR`.
2. Each logger has a configured minimum level. Only messages at or above that level get logged. For example, if the level is `INFO`, then `DEBUG` messages are dropped, but `INFO`, `WARN`, `ERROR` are logged.
3. Support more than one output target (called an **appender**) at the same time. For example, log to console and to a file at once.
4. Support pluggable message format. For example, `[2026-08-24 10:15:00] [INFO] [MyClass] User 42 logged in`.
5. Allow each appender to have its own minimum level. Example: console shows `INFO` and above, but the file logs everything from `DEBUG`.
6. Provide one `Logger` instance per class name (like SLF4J does), through a factory.

**Clarifying questions to ask the interviewer**

- Do we need log rotation (splitting file logs by size or date)? *Assume: out of scope, but mention it as an extension.*
- Do we need structured (JSON) logging? *Assume: not required now, but the `Formatter` interface must support it later.*
- Should logging be synchronous or asynchronous? *Assume: support both. Synchronous by default, with an optional async mode.*
- Is thread-safety required? *Assume: yes. Many threads will call the same logger.*
- Do we need to support changing the log level at runtime, without restarting the app? *Assume: yes, this is a common real-world need.*

**Non-functional requirements**

- Logging must not slow down the main application too much (this drives the async design).
- The framework must be easy to extend: adding a new appender (e.g., a network appender) should not need changes to existing code (Open/Closed Principle).

## Design / Approach

### Key design decisions

1. **Strategy pattern for appenders.** An `Appender` is a strategy for "where to write the log line." `ConsoleAppender` and `FileAppender` both implement the same `Appender` interface. The `Logger` does not care which one it uses.
2. **Strategy pattern for formatting.** A `Formatter` turns a `LogEvent` (level, message, timestamp, logger name, exception) into a `String`. This is separate from the appender, so you can mix and match: JSON format to file, plain text to console.
3. **Chain of Responsibility for level filtering.** Each appender wraps a level check as a "handler" step. When a log event arrives, it passes through a chain: first the logger's own level check, then each appender's own level check. If any check fails, that step stops processing for that appender only (other appenders still get a chance). This models how real frameworks let each appender have its own threshold, independent of the others.
4. **Singleton for `LoggerFactory`.** Only one factory should exist, holding one map of logger-name to `Logger` instance. This avoids creating duplicate loggers and wastes memory.
5. **Thread-safety.** Multiple threads can call `logger.info(...)` at the same time. We use `ConcurrentHashMap` for the logger cache, `volatile` for the mutable log level, and either synchronized writes or a lock-free queue for the actual I/O.
6. **Optional async mode.** Writing to console or file is a slow, blocking I/O operation. If we do this on the caller's thread, we slow down the whole application. In async mode, the caller just puts the `LogEvent` on a queue and returns right away. A background thread reads from the queue and does the real writing.

### ASCII sketch

```
                 ┌───────────────┐
   app code ---> │    Logger      │  (one per class, from LoggerFactory)
                 │  level: INFO   │
                 └───────┬────────┘
                         │ log(level, msg)
                         │ 1. check logger level (Chain step 0)
                         ▼
                 ┌───────────────────────────────┐
                 │        List<Appender>          │
                 │  each appender = 1 chain link  │
                 └───────────────────────────────┘
                    │             │            │
             (level filter) (level filter) (level filter)
                    ▼             ▼            ▼
            ┌──────────────┐ ┌──────────┐ ┌───────────────┐
            │ConsoleAppender│ │FileAppender│ │ (future: DB,  │
            │ + Formatter   │ │ + Formatter│ │  Network...)  │
            └──────────────┘ └──────────┘ └───────────────┘

  Async mode inserts a queue + worker thread between Logger and appenders:

   Logger.log() --> BlockingQueue<LogEvent> --> AsyncWorker thread --> Appenders
   (fast, non-blocking)                          (does the slow I/O)
```

Patterns used: **Strategy** (`Appender`, `Formatter`), **Chain of Responsibility** (level filtering per appender), **Singleton** (`LoggerFactory`), **Factory Method** (`LoggerFactory.getLogger(...)`), and (in extensions) **Observer** for the async queue consumer.

## Java Solution

```java
import java.time.LocalDateTime;
import java.time.format.DateTimeFormatter;
import java.util.List;
import java.util.Map;
import java.util.concurrent.BlockingQueue;
import java.util.concurrent.ConcurrentHashMap;
import java.util.concurrent.CopyOnWriteArrayList;
import java.util.concurrent.LinkedBlockingQueue;
import java.util.concurrent.atomic.AtomicBoolean;

// ---------- Level ----------
public enum LogLevel {
    DEBUG(0), INFO(1), WARN(2), ERROR(3);

    private final int rank;
    LogLevel(int rank) { this.rank = rank; }

    // true if "this" level is severe enough to pass the "minimum" threshold
    public boolean isAtLeast(LogLevel minimum) {
        return this.rank >= minimum.rank;
    }
}

// ---------- Log event (immutable data carried through the system) ----------
public final class LogEvent {
    private final LogLevel level;
    private final String loggerName;
    private final String message;
    private final Throwable throwable; // may be null
    private final long timestampMillis;
    private final String threadName;

    public LogEvent(LogLevel level, String loggerName, String message, Throwable throwable) {
        this.level = level;
        this.loggerName = loggerName;
        this.message = message;
        this.throwable = throwable;
        this.timestampMillis = System.currentTimeMillis();
        this.threadName = Thread.currentThread().getName();
    }

    public LogLevel getLevel() { return level; }
    public String getLoggerName() { return loggerName; }
    public String getMessage() { return message; }
    public Throwable getThrowable() { return throwable; }
    public long getTimestampMillis() { return timestampMillis; }
    public String getThreadName() { return threadName; }
}

// ---------- Formatter (Strategy) ----------
public interface Formatter {
    String format(LogEvent event);
}

public class SimpleTextFormatter implements Formatter {
    private static final DateTimeFormatter TS_FORMAT =
            DateTimeFormatter.ofPattern("yyyy-MM-dd HH:mm:ss");

    @Override
    public String format(LogEvent event) {
        String time = LocalDateTime.now().format(TS_FORMAT); // in real code, derive from event.getTimestampMillis()
        StringBuilder sb = new StringBuilder();
        sb.append('[').append(time).append(']')
          .append(" [").append(event.getLevel()).append(']')
          .append(" [").append(event.getThreadName()).append(']')
          .append(" [").append(event.getLoggerName()).append(']')
          .append(' ').append(event.getMessage());
        if (event.getThrowable() != null) {
            sb.append(" | exception=").append(event.getThrowable());
        }
        return sb.toString();
    }
}

// A second formatter to show pluggability (e.g. for machine parsing)
public class JsonFormatter implements Formatter {
    @Override
    public String format(LogEvent event) {
        return String.format(
            "{\"time\":%d,\"level\":\"%s\",\"logger\":\"%s\",\"thread\":\"%s\",\"message\":\"%s\"}",
            event.getTimestampMillis(), event.getLevel(), event.getLoggerName(),
            event.getThreadName(), event.getMessage().replace("\"", "'"));
    }
}

// ---------- Appender (Strategy + one link in the Chain of Responsibility) ----------
public interface Appender {
    void append(LogEvent event);
    void setThreshold(LogLevel level);
    LogLevel getThreshold();
    void close();

    // The "chain" step: only pass the event down to the real write logic
    // if it clears this appender's own threshold. This is the Chain of
    // Responsibility idea applied per-appender rather than one global chain.
    default void handle(LogEvent event) {
        if (event.getLevel().isAtLeast(getThreshold())) {
            append(event);
        }
    }
}

public class ConsoleAppender implements Appender {
    private volatile LogLevel threshold;
    private final Formatter formatter;
    private final Object writeLock = new Object();

    public ConsoleAppender(LogLevel threshold, Formatter formatter) {
        this.threshold = threshold;
        this.formatter = formatter;
    }

    @Override
    public void append(LogEvent event) {
        String line = formatter.format(event);
        // synchronized so lines from different threads do not interleave
        synchronized (writeLock) {
            if (event.getLevel() == LogLevel.ERROR) {
                System.err.println(line);
            } else {
                System.out.println(line);
            }
        }
    }

    @Override public void setThreshold(LogLevel level) { this.threshold = level; }
    @Override public LogLevel getThreshold() { return threshold; }
    @Override public void close() { /* nothing to close for System.out */ }
}

public class FileAppender implements Appender {
    private volatile LogLevel threshold;
    private final Formatter formatter;
    private final java.io.PrintWriter writer;
    private final Object writeLock = new Object();

    public FileAppender(String filePath, LogLevel threshold, Formatter formatter) throws java.io.IOException {
        this.threshold = threshold;
        this.formatter = formatter;
        this.writer = new java.io.PrintWriter(
                new java.io.BufferedWriter(new java.io.FileWriter(filePath, true)), true);
    }

    @Override
    public void append(LogEvent event) {
        String line = formatter.format(event);
        synchronized (writeLock) {
            writer.println(line);
        }
    }

    @Override public void setThreshold(LogLevel level) { this.threshold = level; }
    @Override public LogLevel getThreshold() { return threshold; }

    @Override
    public void close() {
        synchronized (writeLock) {
            writer.close();
        }
    }
}

// ---------- Logger ----------
public class Logger {
    private final String name;
    private volatile LogLevel level; // can change at runtime, so volatile
    private final List<Appender> appenders = new CopyOnWriteArrayList<>();
    private final AsyncLogDispatcher asyncDispatcher; // null if sync mode

    Logger(String name, LogLevel level, AsyncLogDispatcher asyncDispatcher) {
        this.name = name;
        this.level = level;
        this.asyncDispatcher = asyncDispatcher;
    }

    public void addAppender(Appender appender) {
        appenders.add(appender);
    }

    public void setLevel(LogLevel level) {
        this.level = level;
    }

    public void debug(String message) { log(LogLevel.DEBUG, message, null); }
    public void info(String message)  { log(LogLevel.INFO, message, null); }
    public void warn(String message)  { log(LogLevel.WARN, message, null); }
    public void error(String message) { log(LogLevel.ERROR, message, null); }
    public void error(String message, Throwable t) { log(LogLevel.ERROR, message, t); }

    private void log(LogLevel msgLevel, String message, Throwable throwable) {
        // Step 1 of the chain: the logger's own level gate.
        // Cheap check first, so a DEBUG log call in a hot loop costs almost nothing
        // when the logger is set to INFO or above.
        if (!msgLevel.isAtLeast(this.level)) {
            return;
        }
        LogEvent event = new LogEvent(msgLevel, name, message, throwable);

        if (asyncDispatcher != null) {
            asyncDispatcher.enqueue(event, appenders);
        } else {
            dispatchToAppenders(event);
        }
    }

    // Step 2 of the chain: hand the event to every appender; each appender
    // applies its OWN threshold before actually writing.
    void dispatchToAppenders(LogEvent event) {
        for (Appender appender : appenders) {
            try {
                appender.handle(event);
            } catch (Exception e) {
                // one broken appender must not break the others or the caller
                System.err.println("Appender failed: " + e.getMessage());
            }
        }
    }
}

// ---------- Async dispatcher (queue + background writer) ----------
public class AsyncLogDispatcher {
    private static final class QueuedEvent {
        final LogEvent event;
        final List<Appender> appenders;
        QueuedEvent(LogEvent event, List<Appender> appenders) {
            this.event = event;
            this.appenders = appenders;
        }
    }

    private final BlockingQueue<QueuedEvent> queue = new LinkedBlockingQueue<>(10_000);
    private final Thread workerThread;
    private final AtomicBoolean running = new AtomicBoolean(true);

    public AsyncLogDispatcher() {
        this.workerThread = new Thread(this::processQueue, "async-log-writer");
        this.workerThread.setDaemon(true);
        this.workerThread.start();
    }

    void enqueue(LogEvent event, List<Appender> appenders) {
        boolean added = queue.offer(new QueuedEvent(event, appenders));
        if (!added) {
            // Queue full: drop the event rather than block the caller.
            // A production system would count drops and expose that as a metric.
            System.err.println("Log queue full, dropping event: " + event.getMessage());
        }
    }

    private void processQueue() {
        while (running.get() || !queue.isEmpty()) {
            try {
                QueuedEvent qe = queue.poll(500, java.util.concurrent.TimeUnit.MILLISECONDS);
                if (qe == null) continue;
                for (Appender appender : qe.appenders) {
                    appender.handle(qe.event);
                }
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
        }
    }

    public void shutdown() {
        running.set(false);
        workerThread.interrupt();
    }
}

// ---------- LoggerFactory (Singleton + Factory Method) ----------
public final class LoggerFactory {
    private static final LoggerFactory INSTANCE = new LoggerFactory();

    private final Map<String, Logger> loggers = new ConcurrentHashMap<>();
    private volatile LogLevel defaultLevel = LogLevel.INFO;
    private volatile boolean asyncMode = false;
    private AsyncLogDispatcher sharedDispatcher; // created lazily if async mode is on

    private LoggerFactory() { }

    public static LoggerFactory getInstance() {
        return INSTANCE;
    }

    public void configureAsync(boolean enabled) {
        this.asyncMode = enabled;
        if (enabled && sharedDispatcher == null) {
            synchronized (this) {
                if (sharedDispatcher == null) {
                    sharedDispatcher = new AsyncLogDispatcher();
                }
            }
        }
    }

    public void setDefaultLevel(LogLevel level) {
        this.defaultLevel = level;
    }

    // computeIfAbsent guarantees only one Logger is built per name,
    // even if many threads call getLogger("X") at the same time.
    public Logger getLogger(String name) {
        return loggers.computeIfAbsent(name,
                n -> new Logger(n, defaultLevel, asyncMode ? sharedDispatcher : null));
    }

    public Logger getLogger(Class<?> clazz) {
        return getLogger(clazz.getName());
    }
}
```

### Example usage

```java
public class Demo {
    public static void main(String[] args) throws Exception {
        LoggerFactory factory = LoggerFactory.getInstance();
        factory.setDefaultLevel(LogLevel.DEBUG);
        // factory.configureAsync(true); // uncomment to try async mode

        Logger logger = factory.getLogger(Demo.class);
        logger.addAppender(new ConsoleAppender(LogLevel.INFO, new SimpleTextFormatter()));
        logger.addAppender(new FileAppender("app.log", LogLevel.DEBUG, new JsonFormatter()));

        logger.debug("This goes only to the file");   // console threshold is INFO, so skipped there
        logger.info("Service started");
        logger.warn("Cache miss for key=42");
        logger.error("DB connection failed", new RuntimeException("timeout"));
    }
}
```

## How It Works

1. **Getting a logger.** `LoggerFactory.getInstance()` returns the single, shared factory (Singleton). `getLogger(name)` looks up the cache. If no `Logger` exists yet for that name, `computeIfAbsent` creates exactly one, even under concurrent calls, because `ConcurrentHashMap.computeIfAbsent` is atomic per key.
2. **Level check at the logger.** When `logger.info(...)` is called, the `Logger` first compares the message level against its own configured `level`. This is the first, cheapest filter. If the message is below the threshold, the method returns immediately. No `LogEvent` object is even created.
3. **Building the event.** If the level passes, we build a `LogEvent`: it captures the level, message, logger name, thread name, and timestamp at the moment of the call (not later, when it is written — this matters for async mode, where writing can happen seconds later).
4. **Fan-out to appenders (Chain of Responsibility).** The `Logger` holds a list of `Appender` objects. It hands the event to each one in turn. Each appender applies its own threshold in `handle(event)` before writing. This means one event can be accepted by the file appender (threshold `DEBUG`) and rejected by the console appender (threshold `INFO`) at the same time. Each appender is an independent link; one rejecting the event does not stop others from processing it.
5. **Formatting and writing (Strategy).** Inside `append(event)`, each appender asks its `Formatter` to turn the event into a string, then writes that string to its target (console or file). Because `Formatter` is a separate interface, you can attach `JsonFormatter` to the file and `SimpleTextFormatter` to the console, using the very same `LogEvent`.
6. **Thread-safety.**
    - The logger cache uses `ConcurrentHashMap`.
    - The `level` field on `Logger` and `threshold` field on each `Appender` are `volatile`, so a level change made by one thread (e.g., an admin endpoint) is immediately visible to other threads that log.
    - The list of appenders is a `CopyOnWriteArrayList`. Adding an appender is rare; iterating it (once per log call) is frequent. This collection is optimized exactly for that pattern, and iteration never throws `ConcurrentModificationException`.
    - Each appender's actual write is wrapped in a `synchronized` block, so two threads writing at the same time do not interleave their characters into one broken line.
7. **Async mode.** When async mode is on, `logger.log()` does not call the appenders directly. Instead, it packages the event and the appender list into a `QueuedEvent` and puts it on a `BlockingQueue`. This call returns almost instantly. A single background thread (`async-log-writer`) continuously takes items off the queue and does the slow part: formatting and disk/console I/O. This means the application thread (e.g., a thread handling a web request) is not blocked by disk speed. The trade-off: if the process crashes, queued-but-not-yet-written log lines are lost, and if the queue fills up (very high log rate), we drop events rather than block the caller — a design choice to protect application latency over log completeness.

## How to Extend (Follow-ups)

Interviewers often ask "what if we needed X?" Have answers ready for these:

- **Add a new appender type** (e.g., send logs to a network service or a database). Just implement `Appender`. No change needed in `Logger` or any other appender. This is the Open/Closed Principle in action — the system is open to new appenders, closed to modification of existing code.
- **Add a new format** (e.g., XML). Implement `Formatter`. Attach it to any appender.
- **Log rotation.** Change `FileAppender` to check file size or date before each write, and roll over to a new file (`app.log.1`, `app.log.2`, ...) when a limit is hit. This can be wrapped as a `RollingFileAppender` that decorates `FileAppender` (Decorator pattern).
- **Async batching for higher throughput.** Instead of writing one event at a time, have the worker thread drain many events from the queue (`queue.drainTo(list)`) and write them together. This reduces the number of system calls to disk.
- **Multiple queues / multiple worker threads.** For very high log volume, use one queue and worker thread per appender, so a slow appender (e.g., network) does not delay a fast one (e.g., console).
- **MDC (Mapped Diagnostic Context).** Real frameworks let you attach context, like a request ID, to all logs on the current thread, using a `ThreadLocal<Map<String,String>>`. You would add this data to `LogEvent` and let the `Formatter` include it.
- **Filters beyond level.** Add a `Filter` interface (e.g., "only log messages containing 'payment'") that runs before the appender writes. This extends the Chain of Responsibility with more link types.
- **Configuration from a file.** Read levels and appender setup from a properties or YAML file at startup, instead of hardcoding it in `main`. This is what Log4j/Logback do with `log4j2.xml` or `logback.xml`.

## Complexity & Thread-Safety Notes

- **Time complexity per log call:** O(1) for the level check, O(k) to fan out to `k` appenders, and O(m) to format a message of length `m`. This is the same cost model as real logging frameworks.
- **Space:** One `LogEvent` object is allocated per accepted log call (garbage collected soon after). In async mode, the queue holds up to its capacity (10,000 in our example) of pending events, bounding memory use.
- **Thread-safety summary:**
    - `LoggerFactory.getLogger`: safe, due to `ConcurrentHashMap.computeIfAbsent`.
    - `Logger.setLevel` / `Appender.setThreshold`: safe, due to `volatile`. Note that `volatile` gives visibility, not atomicity of compound actions — here we only ever do a single read or single write of the field, so that is enough.
    - `Logger.addAppender`: safe, due to `CopyOnWriteArrayList`. Adding while another thread iterates never crashes; the iterating thread simply may not see the brand new appender for events already in flight.
    - Actual writes in `ConsoleAppender` / `FileAppender`: safe, due to `synchronized` blocks around the shared `PrintWriter`/`System.out`. Without this, two threads' output could interleave mid-line, producing garbled log files — a classic bug in hand-rolled loggers.
    - Async queue: `BlockingQueue` implementations (like `LinkedBlockingQueue`) are already thread-safe for multiple producers and one consumer, so no extra locking is needed there.
- **Why async helps performance:** disk and network I/O can take milliseconds, while the rest of a request may need only microseconds. If every log call blocks on I/O, logging can dominate request latency, especially under load. Moving the I/O to a background thread turns each log call into a fast, in-memory queue push, at the cost of a small delay before the line is actually persisted, and a small risk of loss on crash.

## Interview Tips & Common Mistakes

- **Mention SOLID explicitly.** `Appender` and `Formatter` are Strategy interfaces — this gives you the Open/Closed Principle (add new appenders without touching `Logger`) and Dependency Inversion (`Logger` depends on the `Appender` abstraction, not concrete classes). Say this out loud during the interview; it shows you know why you picked these patterns, not just that you used them.
- **Do not make `Logger`'s constructor public** and let users call `new Logger(...)` directly. Real frameworks force you through a factory, so the cache stays consistent and levels can be centrally managed. If asked why, explain that having many disconnected `Logger` instances for the same class name would make it hard to change the level for that class at runtime.
- **A common mistake:** forgetting the level check at the `Logger` before building a `LogEvent` and formatting a string. If you always build and format the message, then filter, you waste CPU cycles on every disabled `DEBUG` call in a hot path. Always filter first, using the cheapest check available.
- **Another common mistake:** writing to `System.out` without any lock, then being surprised when concurrent writes produce interleaved, broken lines. Always demonstrate you understand *why* a lock is needed here — it is not about protecting a data structure, but about keeping one write call's bytes together.
- **Do not confuse Chain of Responsibility with a linked list of appenders that stop processing after one handles it.** In classic logging, every appender gets a chance regardless of what other appenders decided. If the interviewer expects the "stop at first handler" version of the pattern (like in a support-ticket routing example), clarify: here, "handling" means "each appender independently decides to accept or reject," not "only one appender wins."
- **Be ready to compare with real frameworks.** SLF4J is a *facade*: your code calls `SLF4J`'s `Logger` interface, and at runtime a real engine (Logback or Log4j2) does the actual work. This is the same idea as our `LoggerFactory.getLogger(...)` returning an abstraction. Logback's `Appender` and `Layout` (their name for `Formatter`) map directly to our `Appender` and `Formatter`. Log4j2's `AsyncAppender` uses a ring buffer (via the LMAX Disruptor library) instead of a simple `BlockingQueue`, for higher throughput — mention this as a real-world optimization beyond our simple queue.
- **If asked to make it more "enterprise":** mention MDC for request tracing, structured JSON output for log aggregation tools (like ELK or Splunk), and dynamic reconfiguration via a config file watcher, without necessarily coding all of it live.
