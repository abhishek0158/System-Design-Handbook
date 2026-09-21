# Chapter 13 — Photo Sharing & Feed (Instagram-style)

Design the backend for a photo-sharing app, like Instagram. Users post photos, follow each
other, like posts, and comment. This question tests one thing above all: do you know where
the actual image file should live.

## Requirements & Clarifying Questions

Key features:
- A user can post a photo with a caption.
- A user can follow other users (one-way, same as Chapter 4).
- A user can like a post. A user can like a post only once.
- A user can comment on a post.
- A user sees a feed of posts from people they follow.

Questions to ask the interviewer:
- Where are the actual image files stored? (This is the key question. The answer is: not in
  the database. More below.)
- Do we need multiple image sizes (thumbnail, full-size)? Assume yes, most apps generate a few
  sizes for different screens.
- Can a user edit or delete a comment? Assume yes for delete, and we skip edit history for now.
- Is the feed reverse-chronological, or ranked by an algorithm? We assume reverse-chronological,
  and reuse the feed design from Chapter 4.
- Rough scale for this chapter: 50 million users, average user posts a few photos a week, likes
  and views vastly outnumber posts.

## The Schema

```sql
CREATE TABLE users (
    id              BIGSERIAL PRIMARY KEY,
    username        TEXT NOT NULL UNIQUE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
-- One row per user.

CREATE TABLE follows (
    follower_id     BIGINT NOT NULL REFERENCES users(id),
    followee_id     BIGINT NOT NULL REFERENCES users(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (follower_id, followee_id)
);
CREATE INDEX idx_follows_followee ON follows(followee_id, follower_id);
-- Same junction table as Chapter 4. One row = "follower_id follows followee_id".

CREATE TABLE posts (
    id              BIGSERIAL PRIMARY KEY,
    user_id         BIGINT NOT NULL REFERENCES users(id),
    image_url       TEXT NOT NULL,   -- points to object storage, not a blob column
    caption         TEXT,
    like_count      BIGINT NOT NULL DEFAULT 0,  -- cached, see below
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_posts_user_created ON posts(user_id, created_at DESC);
-- One row per post. The photo file itself is NOT in this table.

CREATE TABLE likes (
    post_id         BIGINT NOT NULL REFERENCES posts(id),
    user_id         BIGINT NOT NULL REFERENCES users(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (post_id, user_id)
);
-- One row = "user_id liked post_id". The primary key is the unique constraint.

CREATE TABLE comments (
    id              BIGSERIAL PRIMARY KEY,
    post_id         BIGINT NOT NULL REFERENCES posts(id),
    user_id         BIGINT NOT NULL REFERENCES users(id),
    body            TEXT NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_comments_post_created ON comments(post_id, created_at);
-- One row per comment. Read pattern: "all comments on this post, oldest first".
```

## Key Design Decisions — the Reasoning

### 1. Never store the image file in the database

This is the single most important lesson in this chapter. The `posts` table has an
`image_url` column, a plain text field with a link. It does **not** have a `BLOB` (binary
large object) column holding the actual photo bytes.

Why not? A relational database like Postgres is built and tuned to manage rows of structured
data — numbers, text, dates — and to search and index them fast. It is not built to store
large binary files well. Storing images as blobs causes three concrete problems:

- **Database bloat.** A photo can be a few megabytes. Millions of photos in blob columns can
  make your database hundreds of times bigger than it needs to be, when the actual structured
  data (captions, likes, comments) might fit in a few gigabytes on its own.
- **Cost.** Database storage (fast SSD-backed disks tuned for random access) is far more
  expensive per gigabyte than object storage, a storage system built specifically for large
  files (like Amazon S3). Paying database prices for photo storage is a waste of money.
- **Slow backups.** Every backup of the database now has to copy every photo byte along with
  it. Backups get slower, take more disk space, and take longer to restore during a recovery.
  A database that should back up in minutes can end up taking hours.

The fix: store the photo file in **object storage** (like Amazon S3, or Google Cloud Storage).
Object storage is a simple store for files, addressed by a key, built to hold huge numbers of
large files cheaply and reliably. When a user uploads a photo, the app writes the file to
object storage, gets back a URL, and saves only that URL (plus small metadata like caption) in
the `posts` row. The rule to remember: **the database holds the address of the file, not the
file.** This same rule applies any time you store user-uploaded files — videos, PDFs,
attachments.

### 2. Likes: unique constraint plus a cached counter

A user should be able to like a post only once. The `likes` table enforces this with its
primary key: `PRIMARY KEY (post_id, user_id)`. A primary key must be unique, so the database
itself rejects a second insert of the same `(post_id, user_id)` pair. This is much safer than
checking "does a like already exist?" in application code first, because two requests arriving
at the same instant (a race condition) could both pass that check before either insert
happens. A database constraint cannot be bypassed this way.

Counting likes with `SELECT COUNT(*) FROM likes WHERE post_id = ?` works, but it gets slow as
a popular post collects millions of likes, and this count is needed on almost every feed
render. The fix is `like_count` on `posts`: a **denormalized** value — one we could always
compute from `likes`, but store directly for speed. On each like or unlike, update it in the
same transaction as the `likes` insert or delete:

```sql
BEGIN;
INSERT INTO likes (post_id, user_id) VALUES (:post_id, :user_id);
UPDATE posts SET like_count = like_count + 1 WHERE id = :post_id;
COMMIT;
```

Wrapping both writes in one transaction keeps them consistent: either both happen, or neither
does. The trade-off is that `like_count` can drift slightly out of sync under very high
concurrency (many simultaneous likes on the same row), so some systems reconcile it with a
periodic batch job that recomputes exact counts. For an interview, naming the denormalized
counter and the transaction is enough; you do not need to build the reconciliation job.

### 3. Comments: a simple child table, no surprises

`comments` is a plain one-to-many table: one post has many comments, each comment belongs to
one post and one user. The index `idx_comments_post_created` supports the main read pattern,
"show all comments on this post, oldest first." There is no unique constraint here, since a
user can comment on the same post many times.

One thing worth saying out loud in an interview: if the app later needs replies to comments
(threaded comments), that is a different pattern — a self-referencing `parent_comment_id`
column. Chapter 14 covers threaded comments in general. Do not over-build this for a first
pass; only add it if the interviewer asks for reply threads.

### 4. The feed: reuse the fan-out design from Chapter 4

Building "show me posts from people I follow, newest first" is the exact same problem as
Chapter 4's social news feed, just with photos instead of short text posts. Do not re-derive
it here — the reasoning is identical:

- **Fan-out on read (pull):** join `follows` to `posts` at read time. Simple, but slow once a
  user follows many accounts, since every feed open re-runs the join.
- **Fan-out on write (push):** when a user posts, insert an entry into a `feed_items` table for
  every follower. Feed reads become a single fast lookup, but a post from a hugely followed
  account causes a huge write burst — the celebrity problem.
- **The hybrid fix:** push for normal accounts, pull-and-merge at read time for accounts over a
  follower threshold (celebrities, influencers).

See Chapter 4's "Key Design Decisions" section 2 and 3 for the full walk-through, including the
SQL for both approaches and the exact trade-off reasoning. Everything there applies unchanged
to a photo feed.

## Scaling It

- **Media on a CDN.** A CDN (content delivery network) is a network of servers spread across
  many locations, each caching copies of files close to users. Point `image_url` at a CDN in
  front of object storage, not at object storage directly. This means a photo uploaded once in
  one region loads fast for a viewer anywhere in the world, without hitting the origin storage
  on every view.
- **The feed store.** As in Chapter 4, move `feed_items` out of Postgres and into a fast
  key-value store like Redis once feed read volume grows large. Cap each user's stored feed at
  a fixed size, and fan out asynchronously through a queue.
- **Cache hot posts.** A small number of posts (viral photos) get a huge share of all views and
  likes. Cache these hot posts' data (image URL, caption, like count) in Redis, so repeated
  reads do not all hit Postgres. Cache invalidation is simple here: refresh the cached
  `like_count` every few seconds instead of on every single like, since a like counter does not
  need to be exact to the second.
- **Index and replica for comments/likes.** Once a single post can have millions of likes or
  thousands of comments, read replicas for `likes` and `comments` take load off the primary
  database, since viewing a post is far more common than liking or commenting on it.
- **Shard by user_id or post_id** once a single Postgres instance cannot hold all data. Since
  most queries are scoped to one post (its likes, its comments) or one user (their posts, their
  feed), either key works as a shard key; pick post_id if hot posts dominate load, user_id if
  feed and profile queries dominate.

## Interview Tips & Common Mistakes

- The number one mistake in this question is proposing a `photo BYTEA` or `photo BLOB` column.
  If you do this, expect the interviewer to stop you immediately. Say "object storage, URL in
  the database" before they have to ask.
- Explain *why* blobs are bad, not just that they are bad: bloat, cost, and slow backups are the
  three concrete reasons. Naming all three shows real understanding, not a memorized rule.
- Do not forget the unique constraint on `likes`. A candidate who checks "already liked?" only
  in application code, without a database constraint, has a race condition bug waiting to
  happen under concurrent requests.
- When asked about the feed, do not re-invent fan-out from scratch under time pressure. Say
  "this is the same fan-out problem as a social feed" and give the short version: push for
  normal users, pull-and-merge for celebrities. Depth here matters more than speed.
- Mention the CDN. Many candidates stop at "store the URL in the database" and forget that the
  URL should point through a CDN, not straight at object storage, for a media-heavy product like
  this one.
