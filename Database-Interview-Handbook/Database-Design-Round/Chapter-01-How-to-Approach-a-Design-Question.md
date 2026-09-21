# Chapter 1 — How to Approach a Design Question

A database design question can feel open-ended and scary. This chapter gives you a fixed method. Follow the same steps every time, and you will always look organized, even on a system you have never seen before.

## The Method — Step by Step

### Step 1: Clarify requirements and rough scale

Before you draw any table, ask questions. An interviewer wants to see that you do not jump to a schema without understanding the problem. This is the single biggest score driver in the round.

Ask about:
- **Features.** What must the system do? Pick the 3–5 core features. Ignore edge features for now.
- **Users.** Who uses this system? One type of user, or many (buyer and seller, rider and driver)?
- **Read vs write heavy.** Do users read data much more than they write it (a news feed), or write a lot too (a chat app)? This changes where you put your design effort.
- **Rough numbers.** How many users? How many rows added per day? This tells you if you need one PostgreSQL server or a much bigger design. A system with 10,000 users needs a very different answer than one with 100 million users.

You will not get exact numbers. That is fine. State a reasonable assumption out loud, for example: "Let's assume 1 million users and 10 million posts a day, mostly read traffic." Write this assumption down, because it drives later choices like indexing and sharding.

### Step 2: Find the entities (the nouns)

An **entity** is a "thing" the system stores data about. Find entities by listing the nouns in the requirements. For a blog: user, post, comment. For an e-commerce site: customer, product, order, payment.

Do not go overboard. Pick entities that need their own identity and their own row. A "post title" is not an entity; it is a column on `posts`. A "comment" is an entity, because each comment has its own id, its own author, and its own life cycle (it can be edited or deleted on its own).

### Step 3: Find the relationships

Once you have entities, connect them. There are three relationship types:
- **One-to-one (1:1)**: one row in table A matches exactly one row in table B. Example: a user and their profile settings. Rare in practice — often you just add columns to the same table instead.
- **One-to-many (1:many)**: one row in table A matches many rows in table B. Example: one user writes many posts.
- **Many-to-many (many:many)**: many rows in A can relate to many rows in B, and the other way round. Example: many students enroll in many courses.

Say the relationship out loud for each pair of entities: "one user has many posts," "one post has many comments," "posts and tags is many-to-many." This step decides your foreign keys in the next step.

### Step 4: Turn entities into tables with keys

Now write real tables.
- Each entity becomes a table. Give it a **primary key** (a column, or set of columns, that uniquely identifies each row). Use a surrogate key like `id BIGSERIAL` or `id UUID` unless there is a strong reason for a natural key.
- For a 1:many relationship, put a **foreign key** (a column that points to the primary key of another table) on the "many" side. A `posts` table gets a `user_id` column that points to `users.id`.
- For a many:many relationship, add a **junction table** (a table that sits between two tables and holds one row per pairing, usually with two foreign keys). `course_enrollments` has `student_id` and `course_id`, both foreign keys.

At this point, write real PostgreSQL. Something like:

```sql
CREATE TABLE users (
  id BIGSERIAL PRIMARY KEY,
  email TEXT NOT NULL UNIQUE,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE posts (
  id BIGSERIAL PRIMARY KEY,
  user_id BIGINT NOT NULL REFERENCES users(id),
  title TEXT NOT NULL,
  body TEXT NOT NULL,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

Keep the schema focused. Show the columns that matter for the discussion. You do not need to list every field a real production table would have.

### Step 5: Add indexes for the main access patterns

A table without the right index is slow once it grows. Design the index for how the data is **read**, not just how it looks on paper.

Ask: "what queries run most often?" For a blog, the main query is "show the latest posts by a user." So you add an index on `posts(user_id, created_at DESC)`. This lets PostgreSQL find one user's posts in sorted order without scanning the whole table.

Rules of thumb:
- Every foreign key column used in joins or lookups usually needs an index.
- If you filter by column A and sort by column B, a composite index `(A, B)` often serves both in one lookup.
- Do not index everything. Every index slows down writes (each insert or update must also update the index), so pick indexes for the queries that actually matter.

### Step 6: Decide normalize vs denormalize

**Normalize** means splitting data into separate tables to avoid storing the same fact twice, so updates only happen in one place. This is the safe, default choice. Start here.

**Denormalize** means copying some data into another table on purpose, to make a common read faster, at the cost of extra storage and a small chance of the copy going stale. Do this only for one specific, hot read path, and say why.

Example: a `posts` table could store `comment_count` as a column, updated each time a comment is added, instead of running `COUNT(*)` on the `comments` table every time someone views the post. You picked a faster read over a perfectly live number. Say this trade-off out loud — that a lot of interviewers want to hear exactly this sentence: "I am denormalizing here because reads are far more common than writes, and a slightly stale count is fine."

Do not denormalize everywhere. If you copy data in many places, updates need to touch many rows, and it becomes easy for those copies to drift apart and disagree.

### Step 7: Think about scaling and NoSQL fit

Only after the schema is solid, talk about scale. Ask: "at our estimated numbers, what breaks first?" Usually the order is:
1. Missing index → slow queries. Fix with the right index (Step 5).
2. Too much read load on one server → add a **cache** (like Redis) in front of the database, or add **read replicas** (copies of the database that only serve reads).
3. Too much data or too much write load for one server → **shard** (split the data across many database servers, usually by a key like `user_id`).

Also ask if any single part of the system fits a different kind of database better than a relational table. Examples:
- A social feed's "who follows whom" graph can grow huge — a graph database can help, but a normal join table also works fine at moderate scale.
- Chat messages are append-only and looked up by conversation and time — a wide-column store (like Cassandra) fits well at very large scale, though PostgreSQL is fine to start.
- A product catalog with free-text search benefits from a search index (like Elasticsearch) next to the main database, not instead of it.

You do not need to switch away from PostgreSQL to sound good. Most interviewers are happy if you say: "I would start with PostgreSQL, because it gives me strong consistency and joins, and I would only reach for NoSQL for one part, if that part has a special access pattern PostgreSQL handles badly at scale." That one sentence shows good judgment.

## How to Talk in the Interview

The schema is only half the score. How you get there matters just as much.

- **Ask clarifying questions first.** Do not start drawing tables in silence. Spend the first two or three minutes on Step 1. This shows you gather requirements before building, which is a real-world skill, not just an interview trick.
- **State your assumptions out loud.** If the interviewer will not give you a number, pick one yourself and say it: "I will assume 100,000 daily active users and mostly-read traffic." An assumption you say out loud is a strength. An assumption you make silently and never mention looks like a gap.
- **Think aloud.** Say what you are doing at each step: "Now I will list the entities... now I will connect them... now I will think about which queries are common, so I can pick indexes." Silence makes it hard for the interviewer to follow or help you.
- **Explain every trade-off.** Nearly every design choice trades one thing for another: normalize (safer, slower joins) vs denormalize (faster reads, risk of stale data); a UUID key (safe to generate anywhere) vs a serial integer key (smaller, sorts by creation order); one big table vs several smaller ones. Say the trade-off, then say which side you pick and why, given the requirements from Step 1. Interviewers score your **reasoning**, not just your final diagram. A perfect schema with no explanation scores worse than a decent schema with a clear reasoning trail.
- **It is fine to change your mind.** If the interviewer adds a new requirement halfway through, say how it changes your schema. Adaptability is a good sign, not a failure.

## A Quick Worked Example

Let's run the method on a small, classic case: a simple blog with users, posts, and comments.

**Step 1 — Clarify.** Assume: users can sign up, write posts, and comment on any post. Say 500,000 users, 2 million posts, 20 million comments. Reads (viewing posts and comments) happen far more than writes.

**Step 2 — Entities.** `user`, `post`, `comment`. (We treat "comment" as its own entity, because it has its own author and its own timestamp.)

**Step 3 — Relationships.** One user has many posts (1:many). One user has many comments (1:many). One post has many comments (1:many). No many-to-many here — this is a simple case on purpose.

**Step 4 — Tables and keys.**

```sql
CREATE TABLE users (
  id BIGSERIAL PRIMARY KEY,
  email TEXT NOT NULL UNIQUE,
  display_name TEXT NOT NULL,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE posts (
  id BIGSERIAL PRIMARY KEY,
  user_id BIGINT NOT NULL REFERENCES users(id),
  title TEXT NOT NULL,
  body TEXT NOT NULL,
  comment_count INT NOT NULL DEFAULT 0,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE comments (
  id BIGSERIAL PRIMARY KEY,
  post_id BIGINT NOT NULL REFERENCES posts(id),
  user_id BIGINT NOT NULL REFERENCES users(id),
  body TEXT NOT NULL,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

**Step 5 — Indexes.** The main reads are "latest posts by a user" and "comments on a post, oldest first." So:
- `posts(user_id, created_at DESC)`
- `comments(post_id, created_at ASC)`

**Step 6 — Normalize vs denormalize.** Keep users, posts, and comments normalized — each fact lives in one place. But add `comment_count` on `posts` as one deliberate denormalization: viewing a post is common, and counting comments live on every view would mean scanning the `comments` table every time. Update `comment_count` with a small increment when a comment is inserted. State the trade-off: the count can lag by a few milliseconds under heavy concurrent writes, and that is fine for a blog.

**Step 7 — Scaling.** At these numbers, a single PostgreSQL server handles this fine with the indexes above. If traffic grows 100x and reads dominate, add a cache for hot posts and read replicas for browsing traffic. No part of this system needs NoSQL — it is a plain relational fit.

That is the full method, start to finish, in about ten minutes of interview time.

## Quick Recall

1. Clarify features, users, read/write mix, and rough scale — say assumptions out loud.
2. Find the entities (the nouns).
3. Find the relationships: 1:1, 1:many, many:many.
4. Build tables with primary keys and foreign keys (junction table for many:many).
5. Add indexes for the main read patterns, not for every column.
6. Normalize by default; denormalize one hot read path on purpose, and say why.
7. Check what breaks first at scale (index → cache → replica → shard), and note if one part fits NoSQL better.
8. Talk the whole time: ask questions, state assumptions, explain trade-offs. Reasoning is scored, not just the schema.

Chapter 2 covers the reusable building blocks (junction tables, status history, append-only ledgers, and more) that you will reuse in almost every design. Chapters 3 and onward are full worked case studies using this exact method.
