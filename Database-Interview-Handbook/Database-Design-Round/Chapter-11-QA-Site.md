# Chapter 11 — Q&A Site (Stack Overflow-style)

A Q&A site lets users post questions, post answers, vote on both, and tag questions by topic. Interviewers like this question because voting and tagging both hide small traps.

## Requirements & Clarifying Questions

Core features:
- A user posts a question. Other users post answers to it.
- Users vote on questions and on answers: up (+1) or down (−1).
- A question can have one accepted answer, chosen by the question's author.
- A question has one or more tags (like `sql`, `postgres`). Users browse or search by tag.
- Each user has a reputation score, built from the votes their posts receive.

Questions to ask the interviewer:
- Can a user vote more than once on the same post? Assume no — one vote per (user, post), and a user can change or remove their vote later.
- Can a user vote on their own post? Assume no, but this is an application rule, not something we enforce in the schema for this chapter.
- Do we need comments on questions/answers? Out of scope — treat them like a smaller version of answers (see Chapter 14 for threaded comments).
- Scale: assume a large site, millions of questions, tens of millions of answers, and reads far outnumber writes — most traffic is people searching and reading, not posting.

Out of scope: comment threads, edit history, moderation/flagging. We focus on posts, votes, and tags.

## The Schema

```sql
CREATE TABLE users (
    id          BIGSERIAL PRIMARY KEY,
    username    TEXT NOT NULL UNIQUE,
    email       TEXT NOT NULL UNIQUE,
    reputation  INT NOT NULL DEFAULT 0,   -- cached, derived from votes (see below)
    created_at  TIMESTAMPTZ NOT NULL DEFAULT now()
);
-- One row per user. reputation is a cache, not the source of truth.

CREATE TABLE questions (
    id                  BIGSERIAL PRIMARY KEY,
    author_id           BIGINT NOT NULL REFERENCES users(id),
    title               TEXT NOT NULL,
    body                TEXT NOT NULL,
    score               INT NOT NULL DEFAULT 0,   -- cached vote total
    accepted_answer_id  BIGINT REFERENCES answers(id),
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    ...
);
-- One row per question. score and accepted_answer_id are both caches / pointers, updated by writes.

CREATE TABLE answers (
    id            BIGSERIAL PRIMARY KEY,
    question_id   BIGINT NOT NULL REFERENCES questions(id),
    author_id     BIGINT NOT NULL REFERENCES users(id),
    body          TEXT NOT NULL,
    is_accepted   BOOLEAN NOT NULL DEFAULT false,
    score         INT NOT NULL DEFAULT 0,   -- cached vote total
    created_at    TIMESTAMPTZ NOT NULL DEFAULT now()
);
-- One row per answer. Belongs to exactly one question.

CREATE TABLE votes (
    id          BIGSERIAL PRIMARY KEY,
    user_id     BIGINT NOT NULL REFERENCES users(id),
    post_type   TEXT NOT NULL,     -- 'question' or 'answer'
    post_id     BIGINT NOT NULL,   -- id in questions or answers, depending on post_type
    value       SMALLINT NOT NULL CHECK (value IN (1, -1)),
    created_at  TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (user_id, post_type, post_id)
);
-- One row per (user, post). The UNIQUE constraint stops double voting.

CREATE TABLE tags (
    id    BIGSERIAL PRIMARY KEY,
    name  TEXT NOT NULL UNIQUE
);
-- One row per tag name, e.g. 'sql', 'postgres'.

CREATE TABLE question_tags (
    question_id  BIGINT NOT NULL REFERENCES questions(id),
    tag_id       BIGINT NOT NULL REFERENCES tags(id),
    PRIMARY KEY (question_id, tag_id)
);
-- Junction table for the many-to-many between questions and tags.
```

A one-line note on the flow: a `question` gets many `answers`; both get many `votes`; a question links to many `tags` through `question_tags`.

## Key Design Decisions — the Reasoning

### 1. One row per (user, post) for votes, with a cached score

A **vote** is one user's opinion (+1 or −1) on one post. The natural way to model it is one row per vote attempt, in a `votes` table, with a `UNIQUE (user_id, post_type, post_id)` constraint. This constraint is the whole trick: the database itself rejects a second vote from the same user on the same post. Your application does not need to "check then insert" — it can rely on the database to reject the duplicate, the same idea as the overlap constraint in Chapter 6.

Letting a user "change their vote" becomes an `UPDATE` or `DELETE` on their one existing row, not a new insert.

Now, how do you show a question's current score (say, "+42")? You could run `SELECT SUM(value) FROM votes WHERE post_type='question' AND post_id=123` on every view. That works, but it means every page view aggregates over a growing table, for an answer that barely changes between one view and the next.

So we keep a **cached score column**: `score` on `questions` and `answers`. When a vote is inserted, updated, or deleted, the application (or a trigger) adjusts `score` by the right amount — add 1, subtract 1, or flip by 2 on a direction change. Reads become a single indexed row lookup, no aggregation. This is the "denormalize a hot read path on purpose" pattern from Chapter 2: `votes` is the source of truth, `score` is a fast copy of the answer.

Trade-off: the cache can drift if a bug skips an update. Fix this with a periodic job that recomputes `score` from `votes` and corrects any mismatch.

### 2. Votes on both questions and answers: one table or two?

A vote can target a question or an answer. Two posts, one action. There are two ways to model this:

**Option A — separate tables**, `question_votes` (with `question_id REFERENCES questions(id)`) and `answer_votes` (with `answer_id REFERENCES answers(id)`). Each has a real foreign key, so the database guarantees the target exists. Simple and safe, but you now have two near-identical tables and two queries for "does this user have a vote here."

**Option B — one polymorphic table**, the `votes` table shown above, with `post_type` ('question' or 'answer') plus `post_id`. One table, one index, one code path for voting on anything. The cost: `post_id` cannot be a real foreign key, since it points to two different tables depending on `post_type`. The database can no longer stop a vote from pointing at a post that does not exist — you must enforce that in application code or a trigger.

This is the classic **foreign-key integrity vs fewer tables** trade-off. Pick option A when correctness matters most and you only have two or three post types. Pick option B when you expect many more votable types later (comments, answers to answers) and do not want a new table each time. Chapter 14 covers this "polymorphic association" pattern in more depth. Either answer is fine in an interview — what matters is naming the trade-off out loud.

### 3. Tags as many-to-many, and finding questions by tag

A question can have several tags, and a tag applies to many questions. This is a classic **many-to-many** relationship (see Chapter 2), modeled with a junction table: `question_tags`, holding just the two foreign keys. The composite primary key `(question_id, tag_id)` also acts as the "no duplicate tag on the same question" constraint — you get that for free, no extra unique index needed.

To find all questions tagged `postgres`, you join: `questions JOIN question_tags ON ... JOIN tags ON ... WHERE tags.name = 'postgres'`. Add an index on `question_tags(tag_id, question_id)` (the reverse of the primary key order) so this lookup is fast; the primary key already covers "given a question, find its tags" in the other direction.

Why not a `tags TEXT[]` array column on `questions` instead? It looks simpler, but array-contains queries are slower and harder to index than a join, you cannot enforce consistent tag spelling, and you cannot easily count "how many questions use this tag." The junction table costs one extra join but stays clean as the site grows.

### 4. The accepted answer

Only the question's author can mark one answer as "accepted" — the answer that solved their problem. We store this two ways, and both matter:

- `answers.is_accepted` — a boolean on the answer itself. Useful for showing a checkmark next to the accepted answer when listing a question's answers.
- `questions.accepted_answer_id` — a pointer on the question, straight to the accepted answer. Useful for showing "accepted answer" at the top of a question page without scanning all its answers.

When an author accepts an answer, the application does two things in one transaction: set `answers.is_accepted = true` on the chosen answer (and `false` on any previously accepted one, if the author changes their mind), and set `questions.accepted_answer_id` to that answer's id. Doing both keeps the two views consistent. A `CHECK` at the application level (or a trigger) should enforce "at most one accepted answer per question."

### 5. Reputation is derived from votes

A user's reputation is not a fact you record directly — it is a rollup of every vote their posts have received (usually with different weights, e.g. an upvoted answer gives more reputation than an upvoted question). Like `score`, `reputation` on `users` is a **cache**; the true source of truth is still `votes`, joined through the user's posts.

We cache it for the same reason: showing "1,204 reputation" next to a username, on every page, is too common a read to recompute each time. When a vote is added, changed, or removed, the application updates the affected user's `reputation` by the right delta, alongside the post's `score`.

## Scaling It

- **Index first.** `answers(question_id)`, `votes(post_type, post_id)`, and `question_tags(tag_id, question_id)` carry the main read patterns: list a question's answers, show a post's votes, browse by tag.
- **This system is very read-heavy.** Most visitors read and never post. Cache hot questions (and their answers) in Redis, keyed by question id, and invalidate the cache when a new answer or vote changes them.
- **Read replicas** for browsing traffic — a few seconds of lag on a vote count is fine for someone just reading.
- **Full-text search** ("search by keyword") should not run on the primary database at scale — offload it to a search index (Elasticsearch, or Postgres full-text search on a replica) once volume grows.

## Interview Tips & Common Mistakes

- Always mention the `UNIQUE (user_id, post_type, post_id)` constraint on `votes` — it is the one-line answer to "how do you stop double voting," and interviewers listen for it.
- Do not compute `score` with `SUM(value)` on every read. Naming the cached-column pattern, and how you keep it in sync, is the strongest signal in this chapter.
- If asked about votes on multiple post types, do not just pick a table design — state the trade-off (foreign-key safety vs a single flexible table) and say which one you would pick and why.
- Remember the junction table for tags. A raw array column is a common shortcut that breaks down once you need to query or count by tag.
- Reputation and score are both derived data. Say this out loud — it shows you know the difference between a fact (a vote) and a rollup of facts (a score).
