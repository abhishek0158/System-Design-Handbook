# Chapter 14 — Reusable Patterns: Tagging, Threaded Comments & Multi-Tenancy

These three problems show up in many different systems, under different names. Learn each pattern once, and you can reuse it in any interview.

## Pattern: Tagging Across Many Entity Types

**The problem.** You want to tag things. A blog post can have tags like `postgres` or `career`. A photo can have tags too. So can a question on a Q&A site. You need one tagging feature that works for many different entity types (posts, photos, questions, …), without copying the whole tagging logic for each one.

**Option 1: a junction table per type.**
For each entity type, add a normal many-to-many junction table (Chapter 2 covers junction tables). One table links posts to tags, another links photos to tags.

```sql
CREATE TABLE tags (
  id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  name TEXT NOT NULL UNIQUE
);

CREATE TABLE post_tags (
  post_id BIGINT NOT NULL REFERENCES posts(id),
  tag_id  BIGINT NOT NULL REFERENCES tags(id),
  PRIMARY KEY (post_id, tag_id)
);

CREATE TABLE photo_tags (
  photo_id BIGINT NOT NULL REFERENCES photos(id),
  tag_id   BIGINT NOT NULL REFERENCES tags(id),
  PRIMARY KEY (photo_id, tag_id)
);
```

Each `post_id` is a real foreign key into `posts`. The database checks it. If you delete a post, `ON DELETE CASCADE` can clean up its tag rows automatically. Queries stay simple: "all tags for post 5" is one join on one table.

**Option 2: one polymorphic `taggables` table.**
A **polymorphic association** is one table that can point to rows in more than one other table. You store the target's type as a string, plus its id, instead of a normal foreign key.

```sql
CREATE TABLE taggables (
  id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  tag_id BIGINT NOT NULL REFERENCES tags(id),
  taggable_type TEXT NOT NULL,   -- 'post', 'photo', 'question'
  taggable_id BIGINT NOT NULL,   -- id in the posts/photos/questions table
  UNIQUE (tag_id, taggable_type, taggable_id)
);

CREATE INDEX ON taggables (taggable_type, taggable_id);
```

One table now handles tagging for every entity type. Adding a new taggable type (say, `comments`) needs no new table and no migration.

**The reasoning — why this trade-off matters.**
The junction-table-per-type approach keeps a **real foreign key**: `post_tags.post_id REFERENCES posts(id)`. The database itself guarantees the post exists. You cannot end up with a tag row pointing at a post that was deleted. This is called **referential integrity** — the guarantee that a foreign key always points to a row that really exists.

The polymorphic table cannot do this. `taggable_id` is just a plain integer. It has no `REFERENCES` clause, because it might point to `posts`, `photos`, or `questions` — a foreign key can only point to one table. So the database cannot check that the row exists. If you delete post 5 and forget to also delete its `taggables` rows, you get an **orphan row**: a tag entry that points to nothing. Nobody notices until a join returns null and something in the app crashes.

The polymorphic table also loses a clean join. You cannot write `JOIN posts ON taggables.taggable_id = posts.id` for every row, because some rows are photos, not posts. Your application code has to branch on `taggable_type` and query the right table, or you write awkward per-type joins.

So: prefer a **junction table per type** when you can. It is more tables, but each one is simple, indexed, and safe — the database does the integrity checking for you. This fits when the list of taggable types is small and known up front (posts, photos, comments — maybe five types, not fifty).

Reach for the **polymorphic table** only when the number of taggable types is large, growing, or not fully known at design time (a generic "tag anything" platform feature, or an admin-configurable list of content types). You are trading integrity and clean joins for flexibility. If you do this, add the missing safety back in the application layer: delete orphan rows when the parent is deleted (a background job, or an application-level cascade), and validate `taggable_type` against an allowed list before insert.

Most interviewers want to hear you say this trade-off out loud — foreign-key safety vs one flexible table — more than they want a specific final answer.

## Pattern: Threaded / Nested Comments

**The problem.** A comment can reply to another comment, which can reply to another one, and so on. This makes a tree: a post has top-level comments, and each comment can have child replies, several levels deep. You need to store this tree and read a full thread (a comment and all its replies, in order) efficiently.

**Option 1: adjacency list (`parent_id`).**
Each comment stores the id of the comment it replies to. This is the pattern from Chapter 2's hierarchy preview, applied to comments.

```sql
CREATE TABLE comments (
  id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  post_id BIGINT NOT NULL REFERENCES posts(id),
  parent_id BIGINT REFERENCES comments(id),   -- NULL = top-level comment
  author_id BIGINT NOT NULL REFERENCES users(id),
  body TEXT NOT NULL,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

Writing a new comment is a single insert: just set `parent_id`. But reading "give me this comment and every reply, at every depth" needs a **recursive query** — a query that repeats itself, following `parent_id` links down one level at a time until there are no more children.

```sql
WITH RECURSIVE thread AS (
  SELECT * FROM comments WHERE id = 42          -- start comment
  UNION ALL
  SELECT c.* FROM comments c
  JOIN thread t ON c.parent_id = t.id            -- one level deeper each pass
)
SELECT * FROM thread;
```

This works, but it costs more the deeper the thread goes, and some ORMs and simple query builders do not support recursive queries at all.

**Option 2: materialized path.**
Store the full chain of ancestor ids as a string on each row, like `'1/4/9'` meaning "child of comment 9, which is a child of comment 4, which is a child of comment 1."

```sql
CREATE TABLE comments (
  id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  post_id BIGINT NOT NULL REFERENCES posts(id),
  path TEXT NOT NULL,     -- e.g. '1/4/9', own id is the last segment
  author_id BIGINT NOT NULL REFERENCES users(id),
  body TEXT NOT NULL,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX ON comments (path);
```

Reading a whole subtree (comment 4 and everything under it) is one query, no recursion:

```sql
SELECT * FROM comments WHERE path LIKE '1/4/%' ORDER BY path;
```

`ORDER BY path` also gives you the comments in a correct nested reading order for free, because the string sort matches the tree order. Writing a comment still needs one extra step: read the parent's path first, then append your own id to build the new path.

**Option 3: closure table.**
Keep the `comments` table plain (just an id, no parent pointer), and add a second table listing every ancestor-descendant pair, not just direct parent-child pairs.

```sql
CREATE TABLE comment_paths (
  ancestor_id   BIGINT NOT NULL REFERENCES comments(id),
  descendant_id BIGINT NOT NULL REFERENCES comments(id),
  depth INT NOT NULL,        -- 0 = the comment itself, 1 = direct child, etc.
  PRIMARY KEY (ancestor_id, descendant_id)
);
```

A comment with two levels of replies below it has one row in `comment_paths` for every descendant at every depth, not just its direct children. Reading "everything under comment 4" is a single simple join, with real foreign keys and no string parsing:

```sql
SELECT c.* FROM comments c
JOIN comment_paths p ON c.id = p.descendant_id
WHERE p.ancestor_id = 4 AND p.depth > 0;
```

The cost shows up on write: inserting one new comment means inserting one row into `comment_paths` for every one of its ancestors (a reply at depth 5 needs 5 new rows, plus one for itself), not just one row.

**The reasoning — easy write vs easy subtree read.**
This is the core trade-off across all three options, and it is worth stating plainly:

- **Adjacency list**: cheapest write (one insert), most expensive read (recursive query, cost grows with depth). Good default when threads are shallow (most comment sections rarely go past 3–5 levels) and your database supports recursive CTEs well, as Postgres does.
- **Materialized path**: medium write cost (read the parent's path, then insert), cheap read (one `LIKE` query, and free ordering). Good when you read subtrees far more often than you write, and depth can get large — this is the common choice for comment systems at scale, and for things like nested categories or file-system-style trees.
- **Closure table**: most expensive write (one row per ancestor level), cheapest and most flexible read (plain join, can also answer "how deep is this," "what are all ancestors of X," efficiently). Good when you need rich queries about the tree itself (not just "give me this subtree"), and reads vastly outnumber writes — but it is the most tables and the most bookkeeping.

For a typical comment thread interview question, saying "adjacency list is simplest and fine for shallow, low-traffic threads; if reads dominate and threads get deep, I'd move to materialized path for cheap subtree reads" is a strong, complete answer.

## Pattern: Multi-Tenancy

**The problem.** You are building a SaaS (Software as a Service) app used by many separate customer companies — each one is a **tenant**. Company A must never see Company B's data. You need to decide how to keep tenants apart in the database.

**Option 1: shared tables with a `tenant_id` column.**
Every table that holds tenant data gets one extra column, and every query filters by it.

```sql
CREATE TABLE tickets (
  id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  tenant_id BIGINT NOT NULL REFERENCES tenants(id),
  title TEXT NOT NULL,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX ON tickets (tenant_id);

SELECT * FROM tickets WHERE tenant_id = 501 AND id = 9001;
```

This is the cheapest option: one database, one schema, one set of tables for every tenant. But it is only as safe as your code. If one query anywhere forgets `WHERE tenant_id = ...`, it leaks data across tenants. Postgres has a real safety net for exactly this: **row-level security (RLS)**, a database feature that filters every query automatically based on a rule you define, even if the application code forgets to filter.

```sql
ALTER TABLE tickets ENABLE ROW LEVEL SECURITY;

CREATE POLICY tenant_isolation ON tickets
  USING (tenant_id = current_setting('app.tenant_id')::BIGINT);
```

With this policy on, Postgres silently adds the tenant filter to every query on `tickets`, for every user of the database connection, closing the "forgot to filter" mistake at the source.

**Option 2: schema-per-tenant.**
One Postgres database, but a separate **schema** (a named group of tables inside one database) per tenant: `tenant_501.tickets`, `tenant_502.tickets`, and so on, with the exact same table structure repeated in each schema.

**Option 3: database-per-tenant.**
Each tenant gets a fully separate database (possibly on a separate server). No two tenants share any table, connection pool, or index at all.

**The reasoning — cost and simplicity vs isolation and per-tenant scaling.**
Move down this list — shared table, schema-per-tenant, database-per-tenant — and you trade cost and simplicity for isolation and control:

- **Shared table + `tenant_id`** is the cheapest to run (one database to back up, patch, and monitor) and the simplest to build. It is the right default for most SaaS products, especially with many small tenants — you do not want thousands of near-empty databases. The risk is a bug that leaks data across tenants, which RLS greatly reduces but you must still remember to add `tenant_id` to every table and every index.
- **Schema-per-tenant** gives each tenant its own copy of the tables, which makes per-tenant backup, export, and "delete this whole tenant's data" trivial (drop one schema). It still shares one database's resources (connections, memory, CPU), so one very heavy tenant can still slow down others. It also gets awkward past a few hundred tenants — migrations must run once per schema.
- **Database-per-tenant** gives full isolation: one tenant's load, backup schedule, or even Postgres version can be tuned independently, and a legal requirement like "this customer's data must live on its own server" is easy to satisfy. The cost is real: many databases to patch, monitor, and back up, and cross-tenant reporting (e.g., "total users across all customers") now needs a separate process to pull data out of every database.

A common real-world pattern is to start with shared tables, and offer schema- or database-per-tenant only for large or regulated customers who pay for it and need the isolation. Say this out loud in the interview: default to the cheap shared model, name the trigger for isolating a specific tenant (compliance, noisy-neighbor performance, contractual isolation), and mention RLS as the safety net for the shared model. That shows you know both the simple path and its real risk.

## Quick Recall

- **Tagging**: prefer one junction table per entity type (real foreign keys, safe deletes). Use a polymorphic `taggable_type` / `taggable_id` table only when entity types are many or growing, and patch the missing integrity checks yourself.
- **Threaded comments — adjacency list** (`parent_id`): cheapest write, needs a recursive query to read a subtree. Good default for shallow threads.
- **Threaded comments — materialized path** (`path = '1/4/9'`): slightly costlier write, one indexed `LIKE` query reads a whole subtree, ordering comes free. Good when reads dominate and depth is large.
- **Threaded comments — closure table** (ancestor/descendant pairs with `depth`): costliest write (one row per ancestor level), cheapest and richest reads. Good when you need many kinds of tree queries.
- **Multi-tenancy — shared tables + `tenant_id`**: cheapest, simplest, right default; back it with row-level security (RLS) so a missing filter cannot leak data.
- **Multi-tenancy — schema-per-tenant**: easy per-tenant export/delete, still shares database resources; gets heavy past a few hundred tenants.
- **Multi-tenancy — database-per-tenant**: full isolation and independent scaling, most operational cost; reserve for large or regulated tenants.
- Across all three patterns, the real question is the same: how much safety and structure do you want the database to enforce for you, versus how much flexibility and low cost do you want instead?
