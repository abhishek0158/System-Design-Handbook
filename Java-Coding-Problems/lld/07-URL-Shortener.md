# URL Shortener (LLD)

## Problem

Design a URL shortener, like TinyURL or Bitly. It has two main operations:

- `shorten(longUrl)`: take a long URL and return a short code (for example, `abc123`). Visiting `https://short.ly/abc123` should lead back to the long URL.
- `expand(shortCode)`: take a short code and return the original long URL.

This document covers the **LLD (low-level design)** version of the problem. This is the version asked in a machine-coding round: write clean Java classes, pick a good encoding scheme, apply SOLID principles, and handle edge cases like custom aliases, collisions, and expiry.

This is **not** the distributed system version (HLD — high-level design). The HLD version deals with things like: sharding the ID space across many servers, caching hot URLs, and handling a 100:1 read-to-write ratio at internet scale. That version is covered separately in the System Design handbook. Here, we build a correct, thread-safe, single-process service with clean interfaces — the kind of code you would write inside one node of that bigger system.

## Requirements & Clarifying Questions

Ask these before coding, to show the interviewer you scope the problem first.

1. **What should the short code look like?** We will assume alphanumeric characters (`a-z`, `A-Z`, `0-9`), which gives 62 possible characters per position. This is called **Base62**. It avoids symbols like `/` or `+` that are awkward in a URL path.
2. **How long should a short code be?** We will not fix a length. Instead, the code grows as needed, based on the numeric id being encoded. In practice, 6–8 Base62 characters cover billions of URLs, so we will mention that as the expected range.
3. **Can a user request a custom alias?** Yes — for example, `short.ly/my-brand`. We must check the alias is not already taken, and reject it with a clear error if it is.
4. **Can the same long URL be shortened twice?** We will allow it, and return a **new** short code each time, unless the interviewer says otherwise. We will note the one-line change needed to make it idempotent (reuse an existing code for a URL already seen).
5. **Should short codes expire?** Yes — support an optional **TTL (time to live)**. After the expiry time, `expand()` must treat the code as not found, even though the mapping still exists in memory until cleaned up.
6. **What happens on a hash collision, if we use hashing instead of an auto-incrementing id?** We must detect it and retry with a different input, not silently overwrite an existing mapping.
7. **Do we need to support deleting a short URL, or usage analytics (click counts)?** Out of scope for the core design, but we will mention where a `clickCount` field would go, since it is a common follow-up.
8. **Is persistence required?** No. We use an in-memory store behind a `UrlRepository` interface, so a real database (SQL or NoSQL) can be swapped in later without touching the service logic.

## Design / Approach

The design has four pieces, kept separate on purpose, so each can change without breaking the others. This follows the **Single Responsibility Principle (SRP)**: each class has exactly one reason to change.

```
                     ┌───────────────────────────┐
      shorten() ───► │      UrlShortenerService    │
      expand()  ◄─── │  (orchestrates the flow)    │
                     └───────────────────────────┘
                        │                    │
                        ▼                    ▼
          ┌───────────────────────┐   ┌───────────────────────┐
          │   EncodingStrategy      │   │    UrlRepository        │
          │   (interface)           │   │    (interface)           │
          │  - encode(id): String   │   │  - save(code, entry)     │
          │  - decode(code): long   │   │  - find(code): entry     │
          └───────────────────────┘   │  - existsByCode(code)     │
              ▲              ▲         └───────────────────────┘
              │              │                    ▲
    ┌──────────────┐  ┌──────────────┐   ┌──────────────┐
    │ Base62Encoding │  │ HashEncoding   │   │ InMemoryUrlRepository │
    │ (uses AtomicLong│  │ (MD5/SHA-256, │   │ (ConcurrentHashMap)   │
    │  id generator)  │  │  first N chars)│   └──────────────┘
    └──────────────┘  └──────────────┘
```

**`EncodingStrategy` interface (Strategy pattern).** This is the core design decision: encoding is *pluggable*. The service does not know or care whether codes come from Base62 counting or from hashing. This follows the **Open/Closed Principle** — we can add a new encoding scheme later without changing `UrlShortenerService`.

**Approach 1: Base62 of an auto-incrementing id.**
Give every new URL a unique, ever-growing numeric id (`0, 1, 2, 3, ...`), using an `AtomicLong` counter — this keeps id generation **thread-safe** without a manual `synchronized` block. Then convert that number to a short string using **Base62**: the same idea as converting a number to base 16 (hex) or base 2 (binary), but using 62 symbols per digit (`0-9`, `a-z`, `A-Z`) instead of 16 or 2.

Why Base62 and not Base10 or Base16? Because more symbols per position means fewer positions needed for the same range of numbers. A 6-character Base62 string can represent `62^6 ≈ 56.8 billion` distinct ids. The same range in Base10 would need 11 digits. Fewer characters means a shorter, more shareable URL — the whole point of a "short" URL.

The encode/decode logic is simple math, similar to decimal-to-binary conversion:
- **Encode:** repeatedly divide the id by 62. The remainder at each step picks one character from the 62-character alphabet. Stop when the number reaches 0. The digits come out in reverse order, so we reverse the string at the end.
- **Decode:** read the string left to right. For each character, multiply the running total by 62, then add the character's position in the alphabet.

This approach is **collision-free by construction**: since each id is unique (the counter never repeats a number), and the encoding is a one-to-one mapping, two different URLs can never get the same code.

**Approach 2: Hashing (MD5/SHA-256), the alternative.**
Instead of a counter, take the long URL's bytes, run them through a hash function (MD5 or SHA-256), and take the first few characters of the resulting hash (usually converted to Base62 or hex for readability). This has one advantage: the same input URL always produces the same code, so it is naturally idempotent — no need to look up "have we seen this URL before."

But hashing has a real weakness: two different URLs can, rarely, produce the same short prefix. This is a **hash collision**. We must handle it: after computing a candidate code, check the repository. If the code is already taken by a *different* long URL, do not save. Instead retry — for example, append a small salt (like an incrementing counter or a fixed suffix) to the input before hashing again, and check again. Repeat a bounded number of times before giving up. This is why Approach 1 (auto-incrementing id) is usually the simpler, safer default for an interview: it needs no collision-retry loop at all. We implement Approach 1 as the primary path and Approach 2 as a second `EncodingStrategy`, to show both are pluggable.

**`UrlRepository` interface (Repository pattern).** Storage is abstracted behind an interface with `save`, `find`, and `existsByCode`. The in-memory implementation uses a `ConcurrentHashMap<String, UrlEntry>`, which gives thread-safe reads and writes without extra locking for single-key operations. Swapping to a real database later means writing one new class, not touching the service.

**`UrlEntry`** is a small immutable value object: it holds the long URL, the short code, the creation time, and an optional expiry time (TTL). Immutability avoids accidental shared-state bugs if the same entry object is read by two threads.

**`UrlShortenerService`** ties it together. It:
1. Validates the long URL is well-formed.
2. If a custom alias is given, checks it is free, and uses it directly (skipping the encoder).
3. Otherwise, generates a new id, encodes it, and — for the hashing strategy — retries on collision.
4. Saves the mapping through the repository.
5. On `expand()`, looks up the code, checks expiry, and returns the long URL or throws a clear exception.

**Patterns used, summarized:**
- **Strategy** — `EncodingStrategy` makes the encoding algorithm swappable.
- **Repository** — `UrlRepository` decouples storage from business logic.
- **Builder** (optional, shown for `UrlEntry`) — clean, readable construction of an immutable object with several optional fields (alias, TTL).
- **SOLID**: SRP (each class, one job), OCP (add new strategies/repositories without editing existing code), DIP (service depends on interfaces, not concrete classes).

## Java Solution

```java
import java.time.Instant;
import java.util.Map;
import java.util.Objects;
import java.util.Optional;
import java.util.concurrent.ConcurrentHashMap;
import java.util.concurrent.atomic.AtomicLong;
import java.security.MessageDigest;
import java.security.NoSuchAlgorithmException;
import java.nio.charset.StandardCharsets;

// ---------- Value object ----------

/** Immutable record of one short-code-to-long-URL mapping. */
final class UrlEntry {
    private final String shortCode;
    private final String longUrl;
    private final Instant createdAt;
    private final Instant expiresAt; // nullable: null means "never expires"

    UrlEntry(String shortCode, String longUrl, Instant createdAt, Instant expiresAt) {
        this.shortCode = Objects.requireNonNull(shortCode);
        this.longUrl = Objects.requireNonNull(longUrl);
        this.createdAt = Objects.requireNonNull(createdAt);
        this.expiresAt = expiresAt;
    }

    boolean isExpired(Instant now) {
        return expiresAt != null && now.isAfter(expiresAt);
    }

    String getShortCode() { return shortCode; }
    String getLongUrl() { return longUrl; }
    Instant getExpiresAt() { return expiresAt; }
}

// ---------- Encoding strategy (Strategy pattern) ----------

interface EncodingStrategy {
    /** Turn a numeric id (or seed) into a short code string. */
    String encode(long id);
}

/** Base62 encoding of an auto-incrementing id. Collision-free by construction. */
final class Base62EncodingStrategy implements EncodingStrategy {
    private static final String ALPHABET =
            "0123456789abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ";
    private static final int BASE = ALPHABET.length(); // 62

    @Override
    public String encode(long id) {
        if (id == 0) return String.valueOf(ALPHABET.charAt(0));
        StringBuilder sb = new StringBuilder();
        long n = id;
        while (n > 0) {
            int remainder = (int) (n % BASE);
            sb.append(ALPHABET.charAt(remainder));
            n /= BASE;
        }
        return sb.reverse().toString();
    }

    /** Decode is the reverse: read left to right, base-62 positional value. */
    public long decode(String code) {
        long result = 0;
        for (char c : code.toCharArray()) {
            int digit = ALPHABET.indexOf(c);
            if (digit < 0) {
                throw new IllegalArgumentException("Invalid character in short code: " + c);
            }
            result = result * BASE + digit;
        }
        return result;
    }
}

/**
 * Alternative strategy: hash the long URL and take the first few characters.
 * Same input always gives the same code (idempotent), but needs collision
 * handling, because two different URLs can map to the same short prefix.
 */
final class HashEncodingStrategy implements EncodingStrategy {
    private static final int CODE_LENGTH = 7;

    /** id is unused here; this strategy hashes a String input instead. See encodeUrl(). */
    @Override
    public String encode(long id) {
        return encodeUrl(String.valueOf(id), 0);
    }

    /** attempt lets the caller add a salt to change the hash on collision retry. */
    String encodeUrl(String longUrl, int attempt) {
        try {
            MessageDigest digest = MessageDigest.getInstance("SHA-256");
            String input = attempt == 0 ? longUrl : longUrl + "#" + attempt;
            byte[] hashBytes = digest.digest(input.getBytes(StandardCharsets.UTF_8));
            StringBuilder hex = new StringBuilder();
            for (byte b : hashBytes) {
                hex.append(String.format("%02x", b));
            }
            return hex.substring(0, CODE_LENGTH);
        } catch (NoSuchAlgorithmException e) {
            throw new IllegalStateException("SHA-256 not available", e);
        }
    }
}

// ---------- Repository (Repository pattern) ----------

interface UrlRepository {
    void save(UrlEntry entry);
    Optional<UrlEntry> findByCode(String shortCode);
    boolean existsByCode(String shortCode);
}

/** Thread-safe in-memory store. Swap this class for a JDBC/NoSQL one later. */
final class InMemoryUrlRepository implements UrlRepository {
    private final Map<String, UrlEntry> store = new ConcurrentHashMap<>();

    @Override
    public void save(UrlEntry entry) {
        store.put(entry.getShortCode(), entry);
    }

    @Override
    public Optional<UrlEntry> findByCode(String shortCode) {
        return Optional.ofNullable(store.get(shortCode));
    }

    @Override
    public boolean existsByCode(String shortCode) {
        return store.containsKey(shortCode);
    }
}

// ---------- Custom exceptions ----------

class AliasAlreadyTakenException extends RuntimeException {
    AliasAlreadyTakenException(String alias) {
        super("Alias already in use: " + alias);
    }
}

class ShortUrlNotFoundException extends RuntimeException {
    ShortUrlNotFoundException(String code) {
        super("No URL found (or it expired) for code: " + code);
    }
}

class InvalidUrlException extends RuntimeException {
    InvalidUrlException(String url) {
        super("Not a valid URL: " + url);
    }
}

// ---------- Core service ----------

final class UrlShortenerService {
    private final EncodingStrategy encodingStrategy;
    private final UrlRepository repository;
    private final AtomicLong idGenerator = new AtomicLong(1); // thread-safe counter
    private static final int MAX_COLLISION_RETRIES = 5;

    UrlShortenerService(EncodingStrategy encodingStrategy, UrlRepository repository) {
        this.encodingStrategy = encodingStrategy;
        this.repository = repository;
    }

    /** Shorten with default settings: no custom alias, no expiry. */
    String shorten(String longUrl) {
        return shorten(longUrl, null, null);
    }

    /**
     * @param customAlias optional user-chosen code; null means auto-generate
     * @param ttlSeconds  optional expiry, in seconds from now; null means never expires
     */
    String shorten(String longUrl, String customAlias, Long ttlSeconds) {
        validateUrl(longUrl);
        Instant now = Instant.now();
        Instant expiresAt = ttlSeconds == null ? null : now.plusSeconds(ttlSeconds);

        String shortCode = (customAlias != null)
                ? reserveCustomAlias(customAlias)
                : generateUniqueCode(longUrl);

        repository.save(new UrlEntry(shortCode, longUrl, now, expiresAt));
        return shortCode;
    }

    String expand(String shortCode) {
        UrlEntry entry = repository.findByCode(shortCode)
                .orElseThrow(() -> new ShortUrlNotFoundException(shortCode));
        if (entry.isExpired(Instant.now())) {
            throw new ShortUrlNotFoundException(shortCode); // treat expired as not found
        }
        return entry.getLongUrl();
    }

    private String reserveCustomAlias(String alias) {
        if (repository.existsByCode(alias)) {
            throw new AliasAlreadyTakenException(alias);
        }
        return alias;
    }

    /** Generates a code and retries on the rare case of a collision (mainly matters for hashing). */
    private String generateUniqueCode(String longUrl) {
        if (encodingStrategy instanceof HashEncodingStrategy hashStrategy) {
            for (int attempt = 0; attempt < MAX_COLLISION_RETRIES; attempt++) {
                String candidate = hashStrategy.encodeUrl(longUrl, attempt);
                if (!repository.existsByCode(candidate)) {
                    return candidate;
                }
            }
            throw new IllegalStateException("Could not generate a unique code after "
                    + MAX_COLLISION_RETRIES + " attempts");
        }
        // Base62-of-counter path: collision-free by construction, no retry needed.
        long id = idGenerator.getAndIncrement();
        String candidate = encodingStrategy.encode(id);
        // Defensive check only: guards against a custom alias earlier claiming this exact code.
        while (repository.existsByCode(candidate)) {
            id = idGenerator.getAndIncrement();
            candidate = encodingStrategy.encode(id);
        }
        return candidate;
    }

    private void validateUrl(String url) {
        if (url == null || url.isBlank() || !(url.startsWith("http://") || url.startsWith("https://"))) {
            throw new InvalidUrlException(url);
        }
    }
}

// ---------- Demo ----------

public class UrlShortenerDemo {
    public static void main(String[] args) throws InterruptedException {
        UrlShortenerService service =
                new UrlShortenerService(new Base62EncodingStrategy(), new InMemoryUrlRepository());

        String code1 = service.shorten("https://example.com/very/long/path/one");
        String code2 = service.shorten("https://example.com/very/long/path/two", "my-brand", null);
        String code3 = service.shorten("https://example.com/temp-page", null, 1L); // expires in 1 sec

        System.out.println("code1 -> " + code1 + " -> " + service.expand(code1));
        System.out.println("code2 -> " + code2 + " -> " + service.expand(code2));

        Thread.sleep(1100);
        try {
            service.expand(code3);
        } catch (ShortUrlNotFoundException e) {
            System.out.println("code3 expired as expected: " + e.getMessage());
        }

        try {
            service.shorten("https://example.com/other", "my-brand", null);
        } catch (AliasAlreadyTakenException e) {
            System.out.println("Alias correctly rejected: " + e.getMessage());
        }
    }
}
```

## How It Works

1. `shorten()` first validates the URL, so we never store garbage input.
2. If the caller passed a custom alias, we check `existsByCode` first. If free, we use the alias directly as the short code, and skip the encoder entirely.
3. Otherwise, we pull the next id from `idGenerator` (an `AtomicLong`), and pass it to `encodingStrategy.encode(id)`.
4. `Base62EncodingStrategy.encode` repeatedly divides by 62, collecting one alphabet character per remainder, then reverses the collected characters. For example, id `125` in Base62 becomes `"2b"` (the exact letters depend on the alphabet order chosen).
5. We save the `UrlEntry` — short code, long URL, creation time, and optional expiry — into the repository.
6. `expand()` looks up the entry. If missing, or if `isExpired(now)` is true, we throw `ShortUrlNotFoundException`. Otherwise we return the long URL.
7. For `HashEncodingStrategy`, the flow differs slightly at code-generation time: we hash the URL, and if the resulting code is already taken (checked via `existsByCode`), we add a small salt (`attempt` number) and hash again, up to `MAX_COLLISION_RETRIES` times.

## How to Extend (Follow-ups)

- **Idempotent shortening (same URL, same code).** Add a second index, `Map<String, String> longUrlToCode`, inside a new repository method `findByLongUrl(longUrl)`. Check it before generating a new code.
- **Click analytics.** Add a `clickCount` field to `UrlEntry` (as an `AtomicLong`, since many threads may read the same entry at once), and increment it inside `expand()`.
- **Custom domains per user, or per-user URL lists.** Add a `userId` field to `UrlEntry`, and a repository method `findAllByUserId(userId)`.
- **Rate limiting.** Wrap `shorten()` with a token-bucket limiter (see the Rate Limiter LLD problem in this same set) so one client cannot flood the id space.
- **Real database backing.** Write `JdbcUrlRepository implements UrlRepository`, backed by a table `(short_code PRIMARY KEY, long_url, created_at, expires_at)`. No change needed to `UrlShortenerService`, because it only depends on the `UrlRepository` interface — this is the payoff of the Repository pattern and the Dependency Inversion Principle.
- **Multiple servers (the HLD version).** Once you need more than one server generating ids, a single `AtomicLong` no longer works, because each server would hand out the same numbers. This is where the distributed design takes over: pre-allocated id ranges per server (or a service like Zookeeper/Snowflake for id generation), a shared cache (Redis) for hot short codes, and read replicas for the heavy read load. That full design is in the System Design handbook, not here.
- **Case-insensitive short codes.** If you want codes to be case-insensitive (fewer possible values, easier for users to type), drop the uppercase letters from the Base62 alphabet and use Base36 instead — this document's `encode`/`decode` logic and `ALPHABET` constant are already positioned to make that a one-line change.

## Complexity & Thread-Safety Notes

- **Time complexity.** `encode(id)` and `decode(code)` both run in `O(log_62(id))` time — in practice, a small constant number of loop iterations (at most about 11 for the full range of a `long`). `shorten()` and `expand()` are `O(1)` on top of that, since `ConcurrentHashMap` gives constant-time average-case `get`/`put`.
- **Space complexity.** `O(n)` for `n` stored URLs, since we keep one `UrlEntry` per short code.
- **Thread safety of id generation.** `AtomicLong.getAndIncrement()` is lock-free and atomic at the hardware level (compare-and-swap). Two threads calling `shorten()` at the same time always get two different ids — never a duplicate, and never a lost update. This is the key thread-safety property in this design: **without `AtomicLong`, a plain `long id++` would race**, and two threads could read the same value before either increments it, producing a duplicate short code.
- **Thread safety of storage.** `ConcurrentHashMap` handles concurrent `save`/`findByCode`/`existsByCode` safely without any extra `synchronized` blocks, because it locks internally at a fine-grained level (per bucket segment), not the whole map.
- **A subtle race in `generateUniqueCode`'s hashing path.** Between the `existsByCode` check and the later `save`, in theory two threads could both check the same code as free right before either saves it. In the Base62 path, this cannot happen, because each thread gets a distinct id from `AtomicLong` before encoding — no two threads ever try to claim the same code. In the hashing path, it is a real (if rare) gap; a production system would close it with an atomic `putIfAbsent`-based check-and-save instead of a separate check-then-save. This is a good detail to raise proactively in an interview.
- **Custom alias race.** Same issue: `reserveCustomAlias` checks `existsByCode`, then `save()` happens later, not atomically. Mention this, and note the fix: use `ConcurrentHashMap.putIfAbsent` directly as the reservation step, so check-and-reserve becomes one atomic operation.

## Interview Tips & Common Mistakes

- **Explain why Base62, specifically.** Interviewers often ask "why not Base64?" Answer: Base64 includes `+` and `/`, which need URL-encoding (escaping) to be safe inside a URL path — an extra complication with no real benefit. Base62 avoids that by using only alphanumeric characters.
- **Do not confuse "hashing" with "encoding."** Base62 encoding is a **reversible**, one-to-one transform of a number — you can always decode it back to the exact same id. A hash (MD5/SHA) is a **one-way** function — you cannot recover the original URL from the hash. That is exactly why the hashing approach needs a lookup table (the repository) to work at all: the hash is only ever used as a short *key*, not as a way to store or recover the URL.
- **Always mention the collision problem for hashing, even if the interviewer does not ask.** It shows you understand hash functions have a limited output space (a fixed number of characters can only represent so many distinct values), so collisions are a "when," not an "if," at scale.
- **Do not hardcode a fixed short-code length up front for the Base62 approach.** A common mistake is padding every code to, say, 6 characters from the start. This wastes space for small ids (id `5` does not need 6 characters) and, worse, breaks if you ever exceed `62^6` ids. Let the length grow naturally with the id.
- **Watch for the check-then-act race** described above (`existsByCode` followed by a separate `save`). Naming it, even if you do not fix it in the time given, is a strong signal of concurrency awareness.
- **Do not skip URL validation.** A quick `startsWith("http")` check is enough for an interview; do not over-engineer a full RFC 3986 URL parser unless asked.
- **Keep the service class free of encoding and storage details.** A common mistake under time pressure is to inline the Base62 math or the `HashMap` directly inside `UrlShortenerService`. This defeats the purpose of the `EncodingStrategy` and `UrlRepository` interfaces, and it is usually the first thing a senior interviewer will point out.
- **State clearly, at the start, that this is the LLD version.** If the interviewer starts asking about sharding or load balancers, say so directly: "That is the distributed/HLD version of this problem — should I switch to that, or continue with the in-process class design?" This shows you know the boundary between the two, rather than mixing concerns.
