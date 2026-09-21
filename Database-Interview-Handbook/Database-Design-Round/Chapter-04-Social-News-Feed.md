# Chapter 4 — Social News Feed

Design the backend for a social feed, like Twitter's home timeline. You follow people, they
post, and you see their posts in your feed. This question is popular because the simple
schema is easy, but the feed itself hides a hard scaling problem.

## Requirements & Clarifying Questions

Key features:
- A user can follow other users. Following is one-way (A follows B does not mean B follows A).
- A user can create a post (short text, maybe an image link).
- A user's home feed shows posts from everyone they follow, newest first.

Questions to ask the interviewer:
- How many users, and how many follows per user on average? Are there "celebrity" accounts
  with millions of followers?
- How fresh must the feed be? Is a few seconds of delay okay?
- Do we need to re-rank the feed ("best" posts first), or is reverse-chronological enough? We
  assume reverse-chronological here, since ranking is a separate, harder problem.
- What is the read-to-write ratio? Assume users check their feed many times a day but post
  rarely — this fact drives most of the design below.

Rough scale for this chapter: 100 million users, average user follows 200 people, most posts
are read within minutes of being written.

## The Schema

```sql
CREATE TABLE users (
    id              BIGSERIAL PRIMARY KEY,
    username        TEXT NOT NULL UNIQUE,
    follower_count  BIGINT NOT NULL DEFAULT 0,   -- denormalized, see below
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
-- One row per user.

CREATE TABLE follows (
    follower_id     BIGINT NOT NULL REFERENCES users(id),  -- the user who follows
    followee_id     BIGINT NOT NULL REFERENCES users(id),  -- the user being followed
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (follower_id, followee_id)
);
CREATE INDEX idx_follows_followee ON follows(followee_id, follower_id);
-- A junction table. One row = "follower_id follows followee_id".

CREATE TABLE posts (
    id              BIGSERIAL PRIMARY KEY,
    author_id       BIGINT NOT NULL REFERENCES users(id),
    body            TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_posts_author_created ON posts(author_id, created_at DESC);
-- One row per post. Read pattern: "latest posts by this author".

CREATE TABLE feed_items (
    user_id         BIGINT NOT NULL REFERENCES users(id),  -- whose feed this entry is in
    post_id         BIGINT NOT NULL REFERENCES posts(id),
    author_id       BIGINT NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL,
    PRIMARY KEY (user_id, created_at, post_id)
);
-- The pre-built feed. One row = "this post should show in this user's feed".
-- In practice this table lives in a fast store, not plain Postgres. See "Scaling It".
```

`follows` is a **junction table**. A junction table is a table that connects two rows in a
many-to-many relationship. Here it connects a user to another user, so it is a
"self-referencing" many-to-many: both columns point to the same `users` table.

## Key Design Decisions — the Reasoning

### 1. How follows are stored, and how to query them

The `follows` table has one row per follow edge: `(follower_id, followee_id)`. This is a
standard many-to-many junction table, except both sides point to `users`.

- **"Who do I follow?"** → `SELECT followee_id FROM follows WHERE follower_id = ?`. The
  primary key `(follower_id, followee_id)` already supports this, since `follower_id` is the
  first column.
- **"Who follows me?"** → `SELECT follower_id FROM follows WHERE followee_id = ?`. This needs
  a second index, `idx_follows_followee`, because the primary key alone cannot serve a lookup
  by the second column efficiently.

We keep two indexes on the same table, one for each direction. When a table is queried in two
directions, index both directions.

### 2. Building the feed: fan-out on write vs. fan-out on read

This is the core problem of the chapter. There are two ways to answer "what posts should I see
in my feed?"

**Fan-out on read (pull model).** Do nothing when a post is created. When a user opens their
feed, run a query like:

```sql
SELECT p.* FROM posts p
JOIN follows f ON f.followee_id = p.author_id
WHERE f.follower_id = :me
ORDER BY p.created_at DESC
LIMIT 20;
```

This is simple and needs no extra storage. But it is slow when a user follows many people,
because the database must fetch and merge recent posts from every followed account, every
single time the feed is opened. Feeds are opened far more often than posts are made, so this
query runs constantly under load. It is a **read-heavy** cost.

**Fan-out on write (push model).** When a user posts, immediately copy the post's id into the
`feed_items` table of every follower:

```sql
INSERT INTO feed_items (user_id, post_id, author_id, created_at)
SELECT follower_id, :new_post_id, :author_id, :now
FROM follows WHERE followee_id = :author_id;
```

Now reading the feed is just: `SELECT * FROM feed_items WHERE user_id = :me ORDER BY
created_at DESC LIMIT 20`. This is a single-table lookup by primary key — very fast. But
writing a single post can mean thousands or millions of inserts, one per follower. It is a
**write-heavy** cost, paid once per post instead of once per feed view.

**The trade-off:** feeds are read far more often than posts are written, so most systems
prefer fan-out on write. It moves the expensive work to post time (rare) and keeps feed reads
(frequent) cheap. The cost is paid by the writer's follower count, not by every reader.

### 3. The celebrity problem, and the hybrid fix

Fan-out on write breaks down for accounts with millions of followers. If a celebrity with 50
million followers makes one post, fan-out on write means 50 million inserts for that one post.
This is slow, and it can overload the system — a burst of writes far bigger than normal.

The fix used by real systems (Twitter is the classic example) is a **hybrid model**:
- For a normal user (say, under some threshold like 10,000 followers), use fan-out on write.
  Push their posts into every follower's `feed_items` when they post.
- For a celebrity (over the threshold), do **not** fan out on write. Instead, at read time,
  pull their recent posts directly and merge them into the follower's feed in memory, alongside
  the pre-built `feed_items` rows.

So a user's final feed is: `feed_items` (pushed posts from normal accounts) + posts pulled
live from any celebrities they follow, merged by time. This limits the expensive push to
accounts where it is cheap, and uses pull only where push would be too costly.

`follower_count` on `users` is **denormalized** — a value we could compute from `follows`, but
store directly for speed. It lets the write path check "is this a celebrity?" with one row
lookup, instead of running `COUNT(*)` on `follows` on every post.

### 4. Why the feed is usually not a plain SQL join at scale

The fan-out-on-read query above (`JOIN follows ... WHERE follower_id = :me`) looks like a
normal SQL join, and it works fine at small scale. It breaks down at scale for two reasons:

- A user who follows 200 people needs the database to touch recent posts from all 200 authors,
  merge them, and sort by time, on every feed load. Multiply this by millions of feed loads
  per minute, and it is too much random I/O for a relational database tuned for transactional
  writes.
- The pre-built `feed_items` table solves the read cost, but at the size of a real feed
  product (billions of feed entries, extremely high read volume, need for very low latency),
  even a well-indexed Postgres table becomes a bottleneck. This is why real systems store the
  feed in a purpose-built fast store instead. More in "Scaling It".

## Scaling It

- **Index first.** Start with `idx_follows_followee` and `idx_posts_author_created` above.
  These make the basic queries correct and reasonably fast at moderate scale.
- **Cache the hot feed.** Most users only look at the first page of their feed. Cache each
  user's latest ~100 feed item ids in Redis, a fast in-memory key-value store. A common
  structure is a Redis **sorted set** per user: the post id is the member, the post's
  timestamp is the score. This gives "newest N posts" in one fast lookup, no Postgres needed.
- **Move `feed_items` out of Postgres.** Once feed writes and reads are both very high volume,
  keep `feed_items` mainly in Redis, and treat Postgres `posts` as the durable source of truth
  for post content. Redis holds only the lightweight list of post ids per feed, which is cheap
  to fan out and cheap to read.
    - Cap each feed list at a fixed size (say, 800 items) and trim older entries. Nobody scrolls
      back forever.
    - Run the fan-out job through a queue (a background worker pulls "post X was just published,
      fan it out" jobs), so posting does not block on writing to millions of feeds at once.
- **Read replicas** for `posts` and `follows` help once celebrity pull-reads and profile pages
  add load, since these are read-heavy tables.
- **Shard by user_id** if one Redis or Postgres instance cannot hold all feeds. Almost every
  feed query is scoped to one `user_id`, so sharding by `user_id` keeps each query on one shard.

## Interview Tips & Common Mistakes

- Do not jump straight to "just join posts and follows." That answer is correct at small scale,
  but the interviewer wants to see if you know it will not survive at scale. Bring up fan-out
  on write yourself.
- Always mention the celebrity problem when you propose fan-out on write. An interviewer who
  hears "we push every post to every follower's feed" will almost always ask "what if that
  user has 50 million followers?" Have the hybrid answer ready before they ask.
- Say out loud that feed reads vastly outnumber post writes. This one fact is the reason
  fan-out on write is usually the right default — it explains the whole design.
- Do not forget the second index on `follows`. Many candidates index only `follower_id` (via
  the primary key) and forget that "who follows me" needs `followee_id` indexed too.
- It is fine, and expected, to say the real feed store is not plain Postgres. Naming Redis (or
  a similar cache/store) and explaining why — fast reads, simple list-of-ids structure, cheap
  fan-out — shows you understand production systems, not just schema syntax.
