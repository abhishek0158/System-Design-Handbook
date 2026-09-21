# Chapter 7 — URL Shortener

A URL shortener turns a long link into a short one, like `bit.ly/a1B2c3`. It is a common design question because the schema is small, but the read/write pattern and the ID generation trick reveal how well you reason about scale.

## Requirements & Clarifying Questions

Core features:
- A user submits a long URL and gets back a short code.
- Anyone who opens the short URL gets redirected to the long URL.
- A user can optionally pick a custom alias (like `bit.ly/my-brand`).
- A link can expire after a set time.
- We may want basic click stats (how many times a link was opened).

Questions to ask the interviewer:
- Do short codes need to be unique forever, or can they be reused after expiry? (Assume unique while active.)
- Can links be anonymous, or must every link have an owner? (Assume both — `created_by` is nullable.)
- Do we need real-time analytics (dashboards), or just a rough click count? (Assume rough count; heavy analytics would go to a separate system, like a log pipeline, not this database.)
- What is the read:write ratio? (Assume very read-heavy: for every 1 URL created, it may be clicked thousands of times.)

Rough scale: assume 100 million short URLs created, with billions of redirect reads per month. This ratio drives most of the design.

## The Schema

```sql
CREATE TABLE urls (
    id           BIGSERIAL PRIMARY KEY,
    short_code   VARCHAR(12) NOT NULL,
    long_url     TEXT NOT NULL,
    created_by   BIGINT REFERENCES users(id),   -- nullable, for anonymous links
    created_at   TIMESTAMPTZ NOT NULL DEFAULT now(),
    expires_at   TIMESTAMPTZ,                   -- null = never expires
    is_active    BOOLEAN NOT NULL DEFAULT true,
    UNIQUE (short_code)
);

CREATE TABLE url_click_stats (
    url_id       BIGINT PRIMARY KEY REFERENCES urls(id),
    click_count  BIGINT NOT NULL DEFAULT 0,
    last_clicked_at TIMESTAMPTZ
);
```

- `urls`: one row per short link. `short_code` is what the browser sends us; `long_url` is where we redirect to.
- `url_click_stats`: a separate table for the click counter. One row per URL, updated on every click. It is split out so that hot writes (click counts) do not lock or bloat the main `urls` table, which we read far more often than we write.

That is the whole schema. This is a small, focused design — the interesting parts are in the reasoning, not the table count.

## Key Design Decisions — the Reasoning

**1. How to generate the short code: Base62 of an auto-increment ID, not a hash.**

There are two common approaches.

*Approach A — hash the long URL.* Run the URL through a hash function (like MD5), take the first 6–8 characters. Problem: two different URLs can produce the same short hash (a "collision"). You then need to check if the code is taken, and retry with a different hash (e.g., add a salt) if so. This adds a "check-then-retry" loop on every write.

*Approach B — Base62 encode an auto-increment ID.* Base62 is a way to write a number using 62 symbols instead of 10: digits `0-9`, lowercase `a-z`, uppercase `A-Z`. Same idea as decimal, but with a bigger "alphabet," so the same number takes fewer characters. Every row already gets a unique, ever-increasing `id` from `BIGSERIAL` (a normal auto-increment integer). We Base62-encode that `id` to get the short code — ID `125` might become `"cb"`. Since the source ID is already unique, the code is **automatically unique — no collision check needed.** A 6-character Base62 code covers up to 62^6 ≈ 56 billion IDs, enough for a very large service.

Trade-off: Base62-of-ID makes codes predictable and sequential, and needs the row inserted first to get the ID. Hashing gives a code before touching the database, but the collision-handling cost usually outweighs that. For an interview, Base62-of-ID is the cleaner answer — it removes a whole class of bugs.

**2. `short_code` has a unique index and is the main lookup key.**

Every redirect request looks up a row by `short_code`, not by `id`. So the query pattern is:

```sql
SELECT long_url FROM urls WHERE short_code = 'a1B2c3' AND is_active = true;
```

The `UNIQUE (short_code)` constraint both enforces no duplicates and gives us a B-tree index for free — so this lookup is fast even with billions of rows. Never scan or filter by `long_url`; there is no reason to index it, since we never search "find the code for this long URL" at read time.

**3. Custom aliases and expiry.**

A custom alias (user picks `"my-brand"` instead of an auto-generated code) is just a different way to fill the same `short_code` column. The insert path becomes: if the user gave an alias, try to insert it and rely on the `UNIQUE` constraint to reject a duplicate (catch the constraint error and tell the user to pick another one). If no alias was given, generate one from the new row's ID as described above. Both cases write to the same column, so reads do not need to know which path created the code.

`expires_at` is a nullable timestamp. On each redirect, we check `expires_at IS NULL OR expires_at > now()`. A background job can periodically flip `is_active = false` (or hard-delete) for expired rows, so the hot read path does not always need to compute "is this expired" against the current time under load — though checking it inline is also fine at moderate scale.

**4. This system is read-heavy, so caching is the main design point.**

Assume a 1,000:1 or higher read-to-write ratio: one URL is created once, but clicked thousands of times over its life. Hitting Postgres for every redirect is wasteful once traffic grows, even with a good index — the database becomes a bottleneck from sheer request volume, not slow queries.

The fix: put a cache (like Redis, an in-memory key-value store) in front of the database, storing `short_code → long_url`. On a redirect request:
1. Check Redis for the code. If found (a "cache hit"), redirect immediately — no database hit at all.
2. If not found (a "cache miss"), read from Postgres, then write the result into Redis before replying, so the next request for that code is a hit.

Since a short URL's mapping never changes after creation, this cache has no tricky invalidation problem — the classic hard part of caching. We can cache with a long time-to-live, or no expiry at all, and only evict entries when the link expires or is deleted. This makes the URL shortener one of the cleanest real-world examples of "cache-aside" caching.

Click counting is a write, but it does not need to be synchronous. Increment an in-memory counter (in Redis) on every click and flush it to `url_click_stats` in batches (e.g., every few seconds), so the hot redirect path is never slowed down by a database write.

## Scaling It

- **Index first.** The unique index on `short_code` is enough for a long time — B-tree lookups by exact key stay fast even at billions of rows.
- **Cache next.** As covered above, this is the biggest lever here. A well-cached URL shortener can serve almost all redirects without touching the database.
- **Read replicas.** For the cache-miss traffic (and any admin/analytics queries), add Postgres read replicas. Redirect reads go to a replica or the cache; only writes (new URL, click-count flush) go to the primary.
- **ID generation at write scale.** If write volume grows enough to need multiple database instances (sharding), a single auto-increment column no longer works — two shards could generate the same ID. Two standard fixes: **ID ranges** (each shard is handed a block of IDs to use up, e.g., server A gets 1–1,000,000, server B gets 1,000,001–2,000,000, avoiding a round trip per insert), or **a counter service** (a small dedicated service, or a Snowflake-style ID generator, hands out unique IDs on request, often baking in a timestamp and machine ID so any node can generate IDs without colliding). In an interview, naming both and noting that writes are the smaller problem here — reads are what we optimize for first — is enough.

## Interview Tips & Common Mistakes

- Do not over-engineer the schema. This system has one main table. Adding many columns or tables signals you have missed the point — the interesting part is ID generation and caching, not table design.
- If you reach for hashing the URL for the short code, immediately mention the collision problem and how you would handle it (retry with a salt, or check-then-insert). Interviewers want to see you spot the issue yourself.
- Say out loud that this is read-heavy and that caching is the main lever. Candidates who jump straight to sharding the database without mentioning a cache first are missing the cheapest, biggest win.
- Do not forget the unique constraint on `short_code`. Without it, two concurrent writes could create the same code, and no one would know until users report broken redirects.
- Remember that the cache here has no invalidation problem, since a code's target URL never changes. Point this out — it shows you understand *why* caching is easy here, not just *that* you should do it.
- If asked about custom aliases, mention the race condition: two users might submit the same alias at once. Let the database's unique constraint be the single source of truth, and handle the resulting error gracefully, rather than checking "does this alias exist" and inserting as two separate steps (which has a race window).
