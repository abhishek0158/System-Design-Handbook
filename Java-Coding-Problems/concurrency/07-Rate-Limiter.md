# Rate Limiter

## Problem

Design and implement a rate limiter for a backend service. A rate limiter controls how many requests a client can send in a given time period. If a client sends too many requests, the rate limiter blocks the extra requests.

Example: allow each client at most 5 requests per second. If a client sends a 6th request within the same second, reject it.

You must support multiple algorithms behind one common interface, and the limiter must work correctly when many threads call it at the same time.

## Requirements & Clarifying Questions

Before coding, ask these questions in an interview. They show you think about real systems, not just algorithms.

1. **What is the limit?** Is it "N requests per second" or "N requests per minute"? Is the limit the same for every client, or can it differ per client (for example, paid users get a higher limit)?
2. **Per-client or global?** Do we limit each client separately (per API key, per user ID, per IP), or do we limit the whole service together?
3. **What happens on rejection?** Do we return an HTTP 429 (Too Many Requests) error, queue the request, or drop it silently?
4. **Do we need bursts?** Can a client send a short burst of requests above the average rate, as long as the long-term average stays under the limit? This decides if Token Bucket is a better fit than Fixed Window.
5. **Single server or distributed?** Do we run one instance of the service, or many instances behind a load balancer? A single-JVM in-memory limiter does not work correctly across many servers unless we share state (for example, in Redis).
6. **How exact must the limit be?** Some algorithms allow small overshoot near window boundaries. Is that acceptable, or must the limit be exact at all times?
7. **What is the cost of a rejected check?** Should `allowRequest` be fast (O(1) or close to it) since it runs on every request?

For this solution sheet, we assume: per-client limits, in-memory (single JVM) implementation first, with notes on how to go distributed. We build a common interface so the algorithm can be swapped without changing calling code.

## Design / Approach

We define one interface that every algorithm implements:

```java
public interface RateLimiter {
    /**
     * @param clientId unique key for the caller (API key, user ID, IP, etc.)
     * @return true if the request is allowed, false if it must be rejected
     */
    boolean allowRequest(String clientId);
}
```

Each algorithm keeps its own per-client state in a `ConcurrentHashMap<String, ...>`. We use `computeIfAbsent` to create state for a new client safely, and we use either `synchronized` blocks or atomic classes to update that state safely under concurrent access.

We cover five algorithms, from simplest to most useful:

1. **Fixed Window Counter** — count requests in a fixed time window (for example, "the current second"). Simple, but allows a burst at window boundaries.
2. **Sliding Window Log** — keep a timestamp for every request and count how many fall within the last time period. Accurate, but uses more memory.
3. **Sliding Window Counter** — an approximation that blends the current and previous fixed windows. Good balance of accuracy and memory.
4. **Token Bucket** — a bucket holds tokens. Each request takes one token. Tokens refill over time. Allows controlled bursts. This is the most common answer expected in interviews, so we give the full implementation.
5. **Leaky Bucket** — requests go into a queue (bucket) and leave at a fixed rate. Smooths bursts into a steady output rate. We cover this briefly.

## Java Solution

### 1. Fixed Window Counter

We divide time into fixed windows (for example, one window per second). We count requests in the current window. When the window changes, we reset the count to zero.

```java
import java.util.concurrent.ConcurrentHashMap;
import java.util.concurrent.atomic.AtomicLong;

public class FixedWindowRateLimiter implements RateLimiter {

    private static class WindowCounter {
        long windowStart;   // start time of current window, in millis
        int count;          // requests seen in current window
    }

    private final int maxRequestsPerWindow;
    private final long windowSizeMillis;
    private final ConcurrentHashMap<String, WindowCounter> clients = new ConcurrentHashMap<>();

    public FixedWindowRateLimiter(int maxRequestsPerWindow, long windowSizeMillis) {
        this.maxRequestsPerWindow = maxRequestsPerWindow;
        this.windowSizeMillis = windowSizeMillis;
    }

    @Override
    public boolean allowRequest(String clientId) {
        WindowCounter counter = clients.computeIfAbsent(clientId, k -> new WindowCounter());
        long now = System.currentTimeMillis();

        synchronized (counter) {
            long currentWindow = now - (now % windowSizeMillis);
            if (counter.windowStart != currentWindow) {
                // New window started. Reset the counter.
                counter.windowStart = currentWindow;
                counter.count = 0;
            }
            if (counter.count < maxRequestsPerWindow) {
                counter.count++;
                return true;
            }
            return false;
        }
    }
}
```

**Trade-off — boundary burst problem:** Say the limit is 5 requests per second. A client can send 5 requests at 0.99 seconds (end of window 1) and 5 more at 1.01 seconds (start of window 2). That is 10 requests in 20 milliseconds, even though the average rate looks fine. This is the main weakness of Fixed Window.

### 2. Sliding Window Log

We store the timestamp of every request in a log (a queue). On each new request, we drop timestamps older than the window, then check if the remaining count is under the limit.

```java
import java.util.ArrayDeque;
import java.util.Deque;
import java.util.concurrent.ConcurrentHashMap;

public class SlidingWindowLogRateLimiter implements RateLimiter {

    private final int maxRequests;
    private final long windowSizeMillis;
    private final ConcurrentHashMap<String, Deque<Long>> clientLogs = new ConcurrentHashMap<>();

    public SlidingWindowLogRateLimiter(int maxRequests, long windowSizeMillis) {
        this.maxRequests = maxRequests;
        this.windowSizeMillis = windowSizeMillis;
    }

    @Override
    public boolean allowRequest(String clientId) {
        Deque<Long> log = clientLogs.computeIfAbsent(clientId, k -> new ArrayDeque<>());
        long now = System.currentTimeMillis();

        synchronized (log) {
            long windowStart = now - windowSizeMillis;
            // Remove timestamps that fell outside the window.
            while (!log.isEmpty() && log.peekFirst() <= windowStart) {
                log.pollFirst();
            }
            if (log.size() < maxRequests) {
                log.addLast(now);
                return true;
            }
            return false;
        }
    }
}
```

**Trade-off:** This is exact — no boundary burst problem. But memory use grows with the number of requests in the window (one `long` per request per client). For a high limit (say, 10,000 requests per minute per client), this uses real memory and CPU time to clean the log on every call.

### 3. Sliding Window Counter

This algorithm approximates the sliding window using only two counters: the count in the previous fixed window and the count in the current fixed window. It weights the previous window's count by how much of it still overlaps with the sliding window.

```java
import java.util.concurrent.ConcurrentHashMap;

public class SlidingWindowCounterRateLimiter implements RateLimiter {

    private static class State {
        long currentWindowStart;
        int currentCount;
        int previousCount;
    }

    private final int maxRequests;
    private final long windowSizeMillis;
    private final ConcurrentHashMap<String, State> clients = new ConcurrentHashMap<>();

    public SlidingWindowCounterRateLimiter(int maxRequests, long windowSizeMillis) {
        this.maxRequests = maxRequests;
        this.windowSizeMillis = windowSizeMillis;
    }

    @Override
    public boolean allowRequest(String clientId) {
        State state = clients.computeIfAbsent(clientId, k -> new State());
        long now = System.currentTimeMillis();

        synchronized (state) {
            long windowStart = now - (now % windowSizeMillis);

            if (state.currentWindowStart == 0) {
                state.currentWindowStart = windowStart;
            } else if (windowStart != state.currentWindowStart) {
                // Moved forward. Figure out how many windows passed.
                long windowsPassed = (windowStart - state.currentWindowStart) / windowSizeMillis;
                state.previousCount = (windowsPassed == 1) ? state.currentCount : 0;
                state.currentCount = 0;
                state.currentWindowStart = windowStart;
            }

            double elapsedInCurrent = now - windowStart;
            double weightOfPrevious = 1.0 - (elapsedInCurrent / windowSizeMillis);
            double estimatedCount = state.previousCount * weightOfPrevious + state.currentCount;

            if (estimatedCount < maxRequests) {
                state.currentCount++;
                return true;
            }
            return false;
        }
    }
}
```

**Trade-off:** This assumes requests are spread evenly inside the previous window, which is not always true. But it uses constant memory per client (just a few numbers), unlike Sliding Window Log. This is a good default choice when you need both accuracy and low memory use.

### 4. Token Bucket (full implementation)

A bucket holds up to `capacity` tokens. Each request takes one token. Tokens refill at a fixed rate (for example, 5 tokens per second) up to the capacity. If the bucket has no tokens, the request is rejected. Because tokens can build up while idle, a client can send a burst of requests right after being idle — up to the bucket capacity.

```java
import java.util.concurrent.ConcurrentHashMap;
import java.util.concurrent.locks.ReentrantLock;

public class TokenBucketRateLimiter implements RateLimiter {

    private static class Bucket {
        double tokens;
        long lastRefillTimestampNanos;
        final ReentrantLock lock = new ReentrantLock();
    }

    private final double capacity;          // max tokens the bucket can hold
    private final double refillTokensPerSecond;
    private final ConcurrentHashMap<String, Bucket> buckets = new ConcurrentHashMap<>();

    public TokenBucketRateLimiter(double capacity, double refillTokensPerSecond) {
        this.capacity = capacity;
        this.refillTokensPerSecond = refillTokensPerSecond;
    }

    @Override
    public boolean allowRequest(String clientId) {
        Bucket bucket = buckets.computeIfAbsent(clientId, k -> {
            Bucket b = new Bucket();
            b.tokens = capacity;                              // start full
            b.lastRefillTimestampNanos = System.nanoTime();
            return b;
        });

        bucket.lock.lock();
        try {
            refill(bucket);
            if (bucket.tokens >= 1.0) {
                bucket.tokens -= 1.0;
                return true;
            }
            return false;
        } finally {
            bucket.lock.unlock();
        }
    }

    // Must be called while holding bucket.lock.
    private void refill(Bucket bucket) {
        long now = System.nanoTime();
        double secondsElapsed = (now - bucket.lastRefillTimestampNanos) / 1_000_000_000.0;
        if (secondsElapsed <= 0) {
            return;
        }
        double tokensToAdd = secondsElapsed * refillTokensPerSecond;
        bucket.tokens = Math.min(capacity, bucket.tokens + tokensToAdd);
        bucket.lastRefillTimestampNanos = now;
    }
}
```

Note: we use `System.nanoTime()` for elapsed-time math, not `System.currentTimeMillis()`. `nanoTime()` is monotonic (it never goes backward), so it is safe against clock adjustments. `currentTimeMillis()` can jump if the system clock is corrected (for example, by NTP sync), which would corrupt our refill math.

**Trade-off:** Token Bucket allows bursts up to `capacity`, which is often what real APIs want (for example, "100 requests per minute, but you can send up to 20 at once"). It uses constant memory per client. The lock per bucket means threads for the *same* client serialize on that bucket, but different clients do not block each other, since each has its own `Bucket` and its own lock.

### 5. Leaky Bucket (brief)

Leaky Bucket models a bucket with a hole in the bottom. Requests come in and fill the bucket (up to its capacity). The bucket "leaks" (processes) requests at a fixed, constant rate. If the bucket is full when a new request arrives, the request is dropped.

The key difference from Token Bucket: Leaky Bucket smooths output to a strictly constant rate. Token Bucket allows bursts to pass through immediately, as long as tokens are available. Leaky Bucket does not allow bursts in the output, even if the input burst is small — every request waits its turn.

A simple way to implement it: use a `BlockingQueue` with fixed capacity as the bucket, and a single background thread (or scheduled task) that polls the queue at a fixed rate and processes one request at a time. `allowRequest` becomes "try to add to the queue; if the queue is full, reject."

```java
import java.util.concurrent.ArrayBlockingQueue;
import java.util.concurrent.BlockingQueue;

public class LeakyBucketRateLimiter {
    private final BlockingQueue<Runnable> bucket;

    public LeakyBucketRateLimiter(int capacity) {
        this.bucket = new ArrayBlockingQueue<>(capacity);
    }

    public boolean submit(Runnable request) {
        return bucket.offer(request); // false if bucket is full
    }
    // A separate scheduled thread calls bucket.poll() and runs it,
    // at a fixed rate, to "leak" requests out at a constant speed.
}
```

Leaky Bucket is less common in interviews than Token Bucket, but it is useful when the goal is to protect a downstream system that can only handle a fixed, steady rate of work (for example, a slow database).

## How It Works

All five classes implement `RateLimiter` (except the queue-based Leaky Bucket sketch, which needs a scheduler and is shown separately). A caller only needs this:

```java
RateLimiter limiter = new TokenBucketRateLimiter(20, 5); // burst 20, refill 5/sec

if (limiter.allowRequest(clientId)) {
    // process the request
} else {
    // return HTTP 429 Too Many Requests
}
```

Per-client state lives in a `ConcurrentHashMap`, keyed by `clientId`. `computeIfAbsent` guarantees only one state object is created per client, even if many threads ask for the same new client at the same time. Each state object has its own lock (or is only ever touched inside one `synchronized` block), so requests for *different* clients never block each other. Only concurrent requests for the *same* client serialize briefly, which is correct — we must count them one at a time to enforce the limit.

**Per-client limits:** Every algorithm above already keys its state by `clientId`. To give different clients different limits, wrap the constructor arguments in a small config lookup:

```java
public class ConfigurableTokenBucketLimiter implements RateLimiter {
    private final ConcurrentHashMap<String, TokenBucketRateLimiter> perClientLimiters = new ConcurrentHashMap<>();
    private final Function<String, ClientConfig> configLookup; // e.g., DB or cache lookup

    @Override
    public boolean allowRequest(String clientId) {
        TokenBucketRateLimiter limiter = perClientLimiters.computeIfAbsent(clientId, id -> {
            ClientConfig cfg = configLookup.apply(id);
            return new TokenBucketRateLimiter(cfg.capacity(), cfg.refillRate());
        });
        return limiter.allowRequest(clientId);
    }
}
```

## How to Extend (Follow-ups)

Interviewers often push into these directions after the base solution:

**1. Make it distributed (multi-instance, using Redis).**
A plain in-memory `ConcurrentHashMap` only works if all requests from one client land on the same server instance. Behind a load balancer, requests from one client can hit different servers. To fix this, move the shared state (the counter, or the token bucket) into Redis, since Redis is fast and shared by all instances.

- For Token Bucket: store `tokens` and `lastRefillTimestamp` as a Redis hash, keyed by `clientId`. Do the "check and update" as a single Lua script run with `EVAL`, so the read-refill-check-decrement steps are atomic on the Redis server. This avoids race conditions between multiple app servers.
- For Fixed Window: use `INCR` on a key like `rate:{clientId}:{windowNumber}`, with `EXPIRE` set on first increment so old windows clean up automatically.
- Watch out for network latency: every `allowRequest` call now needs a round trip to Redis. This adds latency to every request. A common fix is to batch or locally cache "definitely allowed" decisions for a very short time, but this trades off some accuracy.

**2. Per-user vs. global limits.**
A real system often needs both: a per-user limit (so one user cannot flood the system) and a global limit (so the whole service does not overload a downstream dependency). Implement this by checking two `RateLimiter` instances: a per-client one, keyed by `clientId`, and a global one, keyed by a constant key like `"__global__"`. Reject the request if *either* check fails.

**3. Handle clock issues.**
- Never use `System.currentTimeMillis()` for measuring elapsed time (durations), because the system clock can jump backward or forward (NTP sync, manual changes, leap seconds). Use `System.nanoTime()` instead — it only measures elapsed time and is monotonic (never goes backward) on a single JVM.
- In a distributed setup, do not trust each server's local clock for shared state. Either do the time-based logic inside Redis (Redis has its own consistent view, or you pass in a Lua script that uses Redis's `TIME` command), or use a single logical clock source.
- Handle daylight saving time and leap seconds by working in UTC milliseconds/nanoseconds, never in local wall-clock date-time objects, for any rate-limiting math.

**4. Other useful extensions:**
- Add a `Retry-After` header or return value, computed from the algorithm's state (for example, time until the next token is available), so clients know when to retry.
- Support "cost per request" instead of a flat 1 token — some requests (large queries) can cost more tokens than others.
- Add metrics (allowed count, rejected count) per client, useful for monitoring abuse.

## Complexity & Thread-Safety Notes

| Algorithm | Time per call | Space per client | Boundary burst? | Notes |
|---|---|---|---|---|
| Fixed Window Counter | O(1) | O(1) | Yes | Simplest; not accurate at window edges |
| Sliding Window Log | O(1) amortized* | O(N), N = requests in window | No | Most accurate; memory grows with request rate |
| Sliding Window Counter | O(1) | O(1) | Small (approximation) | Good balance of accuracy and memory |
| Token Bucket | O(1) | O(1) | No (controlled burst by design) | Best general-purpose choice; allows intended bursts |
| Leaky Bucket | O(1) for enqueue | O(capacity) | No | Smooths to constant output rate; needs a background worker |

*Sliding Window Log removes old entries from the front of the deque on each call. Each entry is added once and removed once, so the cost is amortized O(1) per call, even though a single call can remove several old entries.

**Thread-safety approach used above:**
- `ConcurrentHashMap` for the per-client map itself, so lookups and insertions of new clients are safe without a global lock.
- `computeIfAbsent` to avoid creating duplicate state objects for the same client under a race.
- A lock scoped to each client's state object (`synchronized (counter)` or a `ReentrantLock` inside `Bucket`), so different clients never block each other, but the same client's concurrent requests are serialized correctly.
- Avoid a single global lock across all clients — that would turn the limiter into a bottleneck under load, since every request for every client would wait on the same lock.
- For the Token Bucket, all reads and writes to `tokens` and `lastRefillTimestampNanos` happen only while holding `bucket.lock`, so there is no need for `volatile` fields — the lock's happens-before relationship is enough.

## Interview Tips & Common Mistakes

- **State the trade-off, not just the code.** Interviewers want to hear you explain *why* Token Bucket is usually preferred (it allows bursts, but bounds them) over Fixed Window (simple, but bursts at boundaries are unbounded within a short window). Say this out loud before coding.
- **Do not use a single global lock.** A common mistake is wrapping the whole `allowRequest` method in one `synchronized` block shared by all clients. This is correct, but it serializes every request from every client through one lock, which does not scale. Lock per-client instead.
- **Do not forget to handle a new client.** If you use a plain `HashMap` with a manual "if absent, create" check, this is a race condition under concurrent access — two threads can both decide the client is missing and create two different state objects. Use `ConcurrentHashMap.computeIfAbsent`, which handles this atomically.
- **Do not use wall-clock time (`currentTimeMillis`) for elapsed-time math.** Use `nanoTime()`. This is a subtle but real bug that shows attention to detail.
- **Remember memory cleanup.** In a long-running service, `clients` (or `buckets`) grows forever if you never remove old, inactive clients. Mention this out loud: a production system needs a cleanup job (for example, a scheduled task that removes entries not used in the last hour), or use a cache with a time-to-live (TTL) eviction policy, such as Caffeine, instead of a plain `ConcurrentHashMap`.
- **Be ready to compare Token Bucket vs. Leaky Bucket clearly.** Token Bucket allows bursts to pass immediately (up to capacity), useful for user-facing APIs. Leaky Bucket forces a constant output rate, useful for protecting a fixed-capacity downstream system. Confusing these two is a common interview slip.
- **Mention distributed rate limiting proactively.** Even if the interviewer only asked for a single-JVM solution, saying "in a real distributed system, I would move this state to Redis with an atomic Lua script" shows you understand production concerns beyond the toy problem.
