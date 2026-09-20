# Chapter 18 — How to Approach a DB Design Question

The database design round asks you to design a schema for a system, live, while talking.
Interviewers care less about a "perfect" schema and more about your **method**: do you ask
questions, state assumptions, and reason about trade-offs? This chapter gives you a
repeatable seven-step method, and shows the exact things to say out loud at each step.

## Key Concepts

### Why interviewers run this round

A design question (for example, "design a database for a blog" or "design a database for a
ride-sharing app") tests three things at once:
1. Can you turn a vague, real-world problem into concrete data (tables, columns, keys)?
2. Do you know **why** you chose a structure, not just how to draw it?
3. Can you think ahead to scale, and know when SQL is not the right tool?

A candidate who jumps straight to drawing tables, without asking a single question, looks
like someone who has never designed a real system. **Talking through your steps out loud is
not optional — it is most of what is being graded.**

### The seven-step method

Use this order every time. It matches how real design work happens: understand the problem,
model the world, then model the storage, then think about scale.

| Step | What you do | What you say out loud |
|---|---|---|
| 1. Clarify | Ask about users, features, scale, read/write ratio | "Before I design, let me confirm a few things..." |
| 2. Entities | List the nouns and how they relate | "The main entities here are..." |
| 3. ER sketch | Draw entities, relationships, cardinality | "Let me sketch this as boxes and arrows..." |
| 4. Tables | Turn entities into tables with PK/FK | "Now I will turn each entity into a table..." |
| 5. Indexes | Add indexes for the main queries | "The main queries will be X, so I need an index on..." |
| 6. Normalize / denormalize | Decide how much to split or combine | "I will normalize this part, but denormalize that part, because..." |
| 7. Scale | Replicas, sharding, cache, NoSQL fit | "If this grows to N users, I would add..." |

We now go through each step in detail, using a running example: **a simple blog** (users
write posts, readers leave comments).

### Step 1 — Clarify requirements and scale

**Never start drawing tables before this step.** A design question is deliberately vague, like
a real product ask from a manager. The interviewer wants to see you narrow it down yourself.

Ask about:
- **Who uses it?** Is this consumer-facing (millions of users) or internal (hundreds)?
- **Main features?** For a blog: can any user write a post? Can posts have tags? Do not
  assume — ask which features are "in scope" for this interview.
- **Read-heavy or write-heavy?** A blog is read-heavy: many people read one popular post, few
  people write posts. This changes your indexing and caching decisions later.
- **Rough numbers.** Ask for an order of magnitude: "10,000 users or 10 million?" You do not
  need an exact number — you need to know if this is a single-server problem or a "we need
  sharding" problem.

Worked example — questions to ask for the blog:
> "Can any registered user write posts, or only some? Can comments be nested (replies to
> replies), or are they flat? Roughly how many users and posts should I design for — thousands
> or millions? Is this mostly people reading posts, or mostly people writing?"

Suppose the interviewer answers: any user can write posts, comments are flat (no replies to
replies), around 1 million users, and it is heavily read-heavy.

**Say your assumptions out loud even if the interviewer does not answer everything.** For
example: "I will assume a post belongs to exactly one author." This protects you — if it is
wrong, the interviewer corrects you immediately, before you build 10 minutes of design on top
of it.

### Step 2 — List entities and relationships

An **entity** is a "thing" your system needs to remember — usually a noun in the requirements
you just gathered. Go through the requirements sentence by sentence and pull out the nouns.

For the blog: "users write posts, readers leave comments" gives us three entities:
**User**, **Post**, **Comment**.

Now describe the **relationship** between each pair — how many of one connects to how many of
another. This is called **cardinality**. The three common cardinalities are:
- **One-to-many (1:N):** one row on one side connects to many rows on the other. Example: one
  user writes many posts, but each post has exactly one author.
- **Many-to-many (M:N):** many rows on each side connect to many rows on the other. Example:
  many posts can each have many tags, and each tag can appear on many posts.
- **One-to-one (1:1):** rare, each row on one side connects to exactly one row on the other.
  Example: one user has exactly one profile settings row.

For the blog: User → Post is one-to-many, Post → Comment is one-to-many, and User → Comment is
also one-to-many (a comment has its own author, separate from the post's author). If we add
tags, Post → Tag is many-to-many — say this out loud as an assumption you are adding.

Say this out loud as a short list: "I have three core entities: User, Post, and Comment. A
user can write many posts and many comments. A post can have many comments, but each comment
belongs to exactly one post and one author."

### Step 3 — Sketch the ER model

An **ER diagram** (entity-relationship diagram) is a picture: boxes for entities, lines for
relationships, with cardinality marked on each line. In an interview, draw it on a whiteboard
or shared doc — simple boxes and arrows are enough, no formal notation needed.

```
[User] ---1----N--- [Post] ---1----N--- [Comment]
   |                                        |
   +--------------------1------------N------+
        (a user also writes many comments)
```

Read this as: one User connects to many Posts, one Post connects to many Comments, and one
User also connects to many Comments directly (a comment's author is separate from the post's
author).

The sketch's job is to make sure you and the interviewer agree on entities and relationships
**before** you commit to table structures. Fixing a wrong relationship here is cheap; fixing
it after five `CREATE TABLE` statements is not.

### Step 4 — Turn entities into tables (PK and FK)

Now translate the ER sketch into real tables. The rule of thumb:
- Each entity becomes a table.
- Each entity gets a **primary key (PK)** — a column (or columns) that uniquely identifies
  each row. Prefer a simple surrogate key (`id BIGSERIAL` or `id UUID`) unless there is a
  natural key the business already relies on.
- A **one-to-many** relationship becomes a **foreign key (FK)** on the "many" side, pointing
  back to the "one" side's primary key.
- A **many-to-many** relationship becomes a new **junction table** (also called a bridge
  table) holding two foreign keys, one to each side.

Worked example — the blog tables:

```sql
users (
    user_id     BIGSERIAL PRIMARY KEY,
    username    VARCHAR(50) UNIQUE NOT NULL,
    email       VARCHAR(255) UNIQUE NOT NULL,
    created_at  TIMESTAMP NOT NULL DEFAULT now()
);

posts (
    post_id     BIGSERIAL PRIMARY KEY,
    author_id   BIGINT NOT NULL REFERENCES users(user_id),
    title       VARCHAR(200) NOT NULL,
    body        TEXT NOT NULL,
    created_at  TIMESTAMP NOT NULL DEFAULT now()
);

comments (
    comment_id  BIGSERIAL PRIMARY KEY,
    post_id     BIGINT NOT NULL REFERENCES posts(post_id),
    author_id   BIGINT NOT NULL REFERENCES users(user_id),
    body        TEXT NOT NULL,
    created_at  TIMESTAMP NOT NULL DEFAULT now()
);
```

Note the pattern: `posts.author_id` is the FK for the "User writes Post" relationship.
`comments.post_id` and `comments.author_id` are two separate FKs, because a comment relates to
both a post and a user, in two different one-to-many relationships.

If we add tags (many-to-many), we need a junction table:

```sql
tags (
    tag_id   BIGSERIAL PRIMARY KEY,
    name     VARCHAR(50) UNIQUE NOT NULL
);

post_tags (
    post_id  BIGINT NOT NULL REFERENCES posts(post_id),
    tag_id   BIGINT NOT NULL REFERENCES tags(tag_id),
    PRIMARY KEY (post_id, tag_id)
);
```

`post_tags` has a **composite primary key** — the pair `(post_id, tag_id)` together, since
neither column alone is unique but the pair is. This matches `order_items(order_id,
product_id)` from the shared E-commerce schema in this handbook.

### Step 5 — Add indexes for the main access patterns

An **index** is a data structure that lets the database find rows fast without scanning the
whole table. A primary key gets one automatically. Your main *queries* need indexes too,
decided from the **access patterns** — the actual queries the app will run, learned in Step 1.

For the blog (read-heavy: readers browse posts and comments):
- "Show all posts by a given author, newest first" → index on `posts(author_id, created_at)`.
- "Show all comments on a given post, oldest first" → index on `comments(post_id, created_at)`.
- "Look up a user by username at login" → the `UNIQUE` constraint on `users.username` already
  gives us this index for free (PostgreSQL creates an index behind every `UNIQUE` constraint).

```sql
CREATE INDEX idx_posts_author_created ON posts (author_id, created_at DESC);
CREATE INDEX idx_comments_post_created ON comments (post_id, created_at);
```

**Say this out loud as a reason, not a rule.** For example: "Since this is read-heavy and
readers mostly load one post's comments in order, I will add a composite index on
`(post_id, created_at)` so the database can fetch a page of comments without sorting." This
connects Step 5 back to Step 1's requirements — a strong signal that your design is driven by
the use case, not by habit. See Chapter 8 (Indexing) for how column order in a composite index
matters.

### Step 6 — Normalize or denormalize

**Normalization** means organizing tables so each fact is stored in exactly one place, to
avoid duplicate or inconsistent data. **Denormalization** means intentionally repeating some
data, to make reads faster at the cost of extra storage and more careful updates.

The blog schema above is already normalized: a post's title lives only in `posts`, a user's
username lives only in `users`. To show a comment's author name without a join every time, we
could denormalize by adding `author_username` onto `comments` — but then a username change
means updating every comment row, which is a real cost.

**The interview answer is "it depends," said with a reason:**
- Normalize when data changes often, and consistency matters more than raw read speed.
- Denormalize a specific, known hot path — not the whole schema — when you have measured (or
  are told) that a join is too slow at your read volume, and the duplicated data changes
  rarely.

For the blog: usernames rarely change, and the comment list is read very often (read-heavy,
per Step 1). Denormalizing `author_username` onto `comments` is a reasonable, targeted choice
— but say it as a trade-off: "I would keep this normalized until I see the join is actually a
bottleneck, since usernames can still change and I do not want stale duplicates everywhere."
Chapter 19 (Data Modeling & Normalization) covers the normal forms and this trade-off in
depth.

### Step 7 — Think about scale: replicas, sharding, cache, NoSQL

Only after the schema is solid do you talk about scaling it. Use the numbers from Step 1 to
decide how far to go — do not jump to "shard everything" for a system with 10,000 users.

- **Read replicas:** copies of the database that serve read queries, while one primary handles
  writes. Good fit here — the blog is read-heavy, so replicas absorb most traffic (Chapter 13).
- **Caching:** store hot data (a popular post, its comment count) in a fast in-memory store
  like Redis, in front of the database. Good fit for "the same 10 posts get read a million
  times a day."
- **Sharding:** splitting one table's rows across multiple servers, usually by a key like
  `user_id`. Only worth it once one server cannot hold the data or the write load, typically
  past tens of millions of rows (Chapter 14).
- **Does any part fit NoSQL?** Ask if a part of the system does not need joins or strict
  consistency. For the blog, a "view count per post" counter fits a key-value store like Redis
  better than a relational table, since it is a simple counter updated very often (Chapters 16
  and 17).

Worked example, said out loud: "Given 1 million users and read-heavy traffic, I would start
with one primary Postgres database and add read replicas for the post and comment reads, plus
a cache in front of popular posts. I would not shard yet — 1 million users easily fit on one
well-sized server. If view counts become a hot, high-write field, I would move just that
counter to Redis instead of updating a SQL row on every page view."

### Putting it together: the blog, end to end

Here is the whole method run back to back, as you would say it in an interview:

1. **Clarify:** any user can write posts, comments are flat, about 1 million users, read-heavy.
2. **Entities:** User, Post, Comment — users write posts and comments; posts have many comments.
3. **ER sketch:** User —1:N— Post —1:N— Comment, plus User —1:N— Comment directly.
4. **Tables:** `users`, `posts` (FK `author_id`), `comments` (FK `post_id`, FK `author_id`).
5. **Indexes:** `(author_id, created_at)` on posts, `(post_id, created_at)` on comments.
6. **Normalize/denormalize:** keep it normalized first; denormalize `author_username` onto
   comments only if that join becomes a measured bottleneck.
7. **Scale:** read replicas plus a cache for popular posts; no sharding yet at this size; move
   view-count counters to a key-value store if they get hot.

This is a complete, defensible answer, and it took seven small, ordered steps — none of them
guesswork.

## The Questions They Ask

**Q1: Walk me through how you would design a database for [system X].**
This is asking you to run the seven-step method live. Start by asking clarifying questions
(Step 1) before writing anything. A strong answer narrates each step as you do it, rather than
presenting a finished schema silently. Interviewers care more about hearing "I am choosing a
composite index here because the main query filters by post_id and sorts by created_at" than
about the final diagram looking pretty.

**Q2: Why did you ask that clarifying question?**
Explain what the answer would change in your design. For example: "I asked about read/write
ratio because if this were write-heavy, I would lean towards fewer indexes, since indexes slow
down writes." A good clarifying question always has a design decision attached to it — if the
answer would not change anything, it is not worth asking.
*Follow-up: "What if I had told you it was write-heavy instead?"* Be ready to redo Steps 5–7
with the opposite assumption — fewer indexes, more careful denormalization, sharding sooner.

**Q3: How do you decide between a foreign key on one table versus a junction table?**
Use the cardinality test from Step 2. If one side is "many" and the other is "one," put the
foreign key on the "many" side (like `posts.author_id`). If both sides can be "many" (like
posts and tags), use a junction table with a foreign key to each side, and usually a
composite primary key on both.

**Q4: When would you denormalize, and how do you defend that choice?**
State the trade-off directly: denormalization trades write complexity and storage for read
speed. Defend it with a concrete access pattern and a concrete cost, not a general statement.
For example: "The homepage loads post titles with author names 10,000 times a minute. Joining
every time is a real, measured cost. Usernames change less than once a year. I will
denormalize the username onto the post, and update it in the rare case a user renames
themselves." A vague "denormalize for performance" without a reason is a weak answer.

**Q5: At what point would you introduce sharding or a NoSQL store here?**
Anchor your answer to the scale numbers from Step 1, and name a real signal: "Once write
throughput on `posts` or `comments` approaches what one server can handle — or storage size
makes backups slow — I would shard by `user_id`, so a user's data stays on one shard." A real
signal (write throughput, storage size, replica lag) beats a vague "when it gets big."

## Rapid-Fire

- **What is the first thing you should do in a design interview?** Ask clarifying questions
  about users, features, and scale — never start drawing tables first.
- **What is an entity?** A "thing" the system must remember, usually a noun from the
  requirements (User, Post, Comment).
- **What is cardinality?** How many rows on one side of a relationship connect to how many
  rows on the other side (one-to-one, one-to-many, many-to-many).
- **How do you turn a one-to-many relationship into tables?** Put a foreign key on the "many"
  side, pointing to the primary key of the "one" side.
- **How do you turn a many-to-many relationship into tables?** Create a junction table with a
  foreign key to each side, usually with a composite primary key on both foreign keys.
- **What decides which indexes to add?** The main access patterns — the actual queries the app
  will run — not every column that looks important.
- **What is the trade-off in denormalization?** Faster reads, in exchange for more storage and
  the risk of inconsistent, duplicated data on writes.
- **When do you consider sharding?** When one server can no longer hold the data or handle the
  write load — anchor this to a real number, not a guess.
- **When does part of a design fit NoSQL instead of SQL?** When that part is simple
  (key-value, counter, or list access) and does not need joins or strict relational
  consistency, even if the rest of the system stays relational.
- **What should you say if you are unsure of a requirement?** State your assumption out loud
  and move on: "I will assume X" — this lets the interviewer correct you early and cheaply.

## Common Traps & Mistakes

- **Jumping straight to tables.** Drawing `CREATE TABLE` statements before asking a single
  question skips the whole point of the round: showing you can turn a vague problem into a
  clear model. Always start with Step 1.

- **Silence while designing.** Working out the schema in your head and only presenting the
  final version gives the interviewer nothing to grade except the end result. Narrate each
  step, including the ones you reject: "I considered putting tags directly on the post table
  as a text column, but a many-to-many junction table lets a tag be reused and queried across
  posts."

- **Treating every relationship as one-to-many by default.** Many candidates forget
  many-to-many relationships need a junction table, and instead store a list of IDs in one
  column (like `tag_ids` as a comma-separated string). This breaks normalization and makes
  queries like "all posts with tag X" slow and awkward.

- **Adding indexes without a reason.** Saying "I will index everything to be safe" is a red
  flag, not a strength. Every index has a write cost (Chapter 8). Tie every index you add to a
  specific query from Step 1.

- **Normalizing or denormalizing as a blanket rule.** "I always normalize fully" or "I always
  denormalize for speed" both sound like memorized rules, not reasoning. The right answer is
  "it depends on this system's read/write ratio and how often this specific data changes" —
  said with the specific data in mind.

- **Talking about sharding and NoSQL too early.** Bringing up sharding for a system with a
  clearly small, stated scale (Step 1) signals "scale talk sounds impressive," not reasoning
  from the numbers given. Match Step 7 to the scale you were told.

- **Forgetting to revisit earlier steps.** If a clarifying answer changes mid-way (the
  interviewer adds a feature), go back and update your entities and tables instead of bolting
  the feature onto the existing schema.

- **Confusing this chapter's checklist with a finished design.** This method gets you to a
  solid first design fast. Real, worked full designs, with harder scale numbers and deeper
  trade-off discussion, are in Chapter 20 (Design Case Studies). Use this chapter for the
  *process*, and Chapter 20 for *practice*.
