# Chapter 21 — Scaling the Data Layer

"The database is slow, what do you do?" is one of the most common system design questions.
Interviewers do not want a random list of fixes. They want to see that you try the cheapest
fix first, and only move to a bigger, riskier fix when the cheap one is not enough.

## Key Concepts

### The one rule that matters: order of operations

Most candidates fail this question not because they don't know the techniques. They fail
because they jump straight to "add a cache" or "shard the database" without checking the
simple things first. A sharded database with a missing index is still slow, just slow at a
bigger scale, and now you also have shard-management pain.

The playbook, in the order you should try things:

1. Fix the queries and indexes.
2. Add a cache in front of the database.
3. Add read replicas, split reads from writes.
4. Partition or shard, when one machine is not enough.
5. Use materialized views or denormalized read tables for expensive reads.
6. Move a hot part of the data to a different store (polyglot persistence).

Alongside this, always check connection pooling. Too many database connections can make a
"scaling" problem that is really a configuration problem.

Say this order out loud in the interview. It shows judgment, not just knowledge.

### Step 1: Fix the queries and indexes (the cheapest win)

Before adding any new infrastructure, find out **why** the database is slow. Most of the
time, it is one of:
- A missing index, so the database scans the whole table.
- A bad query, like `SELECT *` when only two columns are needed, or a query that cannot use
  an index because of a function on the indexed column.
- Too much data being pulled per request (no pagination, no filtering).

**EXPLAIN** is a SQL command that shows the query plan: how the database will execute the
query, without running it. **EXPLAIN ANALYZE** runs the query too, and shows real timing.
Always start here.

```sql
EXPLAIN ANALYZE
SELECT order_id, order_date, total_amount
FROM orders
WHERE customer_id = 4821
ORDER BY order_date DESC;
```

If `orders` has no index on `customer_id`, the plan shows a `Seq Scan` (sequential scan): the
database reads every row in the table to find matches. On a table with millions of rows,
this is slow and gets slower as the table grows.

```
Seq Scan on orders  (cost=0.00..18412.00 rows=40 width=24) (actual time=0.03..142.91 rows=37 loops=1)
  Filter: (customer_id = 4821)
  Rows Removed by Filter: 999963
```

The fix is a normal **B-tree index** (see Chapter 8) on the filter column:

```sql
CREATE INDEX idx_orders_customer_id ON orders(customer_id);
```

Re-running `EXPLAIN ANALYZE` now shows an `Index Scan`, and the same query runs in
microseconds instead of tens of milliseconds. This is almost always the highest
return-on-effort fix. It costs a few minutes and no new infrastructure.

Other cheap query fixes to mention:
- Add a **composite index** matching the query's `WHERE` + `ORDER BY` columns, for example
  `(customer_id, order_date DESC)`, so the database does not need a separate sort step.
- Avoid `SELECT *`; fetch only needed columns, especially if the table has large text or
  JSON columns.
- Batch small repeated queries (the "N+1 query problem", see Chapter 22) into one query with
  a `JOIN` or `WHERE ... IN (...)`.
- Add pagination (`LIMIT` / `OFFSET`, or better, keyset pagination) so one request cannot pull
  a huge result set.

Only move to step 2 once you have checked: right indexes exist, the query plan looks sane,
and the query does not pull more data than it needs.

### Step 2: Add caching in front of the database

A **cache** is a fast, temporary store that holds a copy of data so you do not have to
re-read it from the database every time. Caches (like Redis or Memcached) store data in
memory, so reads are much faster than a database round trip.

**Cache-aside pattern** (also called "lazy loading") is the most common pattern:

1. Application asks the cache for the data.
2. **Cache hit**: data is found, return it. Fast path.
3. **Cache miss**: data is not found. Read it from the database, store a copy in the cache,
   then return it.

```
Read path:
  app -> cache.get(key)
       -> found? return value
       -> not found? app -> db.query() -> cache.set(key, value, ttl) -> return value

Write path:
  app -> db.update() -> cache.delete(key)   (invalidate, don't update the cache directly)
```

What is safe to cache:
- Data that is read far more often than it changes (a product catalog page, a user's
  profile, a leaderboard).
- Data where a few seconds or minutes of staleness is acceptable.
- Expensive-to-compute results (a report, an aggregation, a search result page).

What is **not** safe to cache without care:
- Data that must always be fresh and correct, like an account balance or an inventory count
  right before checkout.
- Data tied to a security check (a stale "is this user an admin" flag is dangerous).

**TTL** (time to live) is how long a cached value is kept before it expires automatically.
A short TTL (seconds to a few minutes) limits how stale data can get, at the cost of more
cache misses and more database load. A long TTL reduces database load but risks serving old
data longer.

**Invalidation** means removing or updating a cache entry when the underlying data changes.
This is the hard part of caching. There is an old joke in computer science: "There are only
two hard things: cache invalidation and naming things." Two common strategies:
- **TTL-only**: don't invalidate manually, just let entries expire. Simple, but the cache can
  serve stale data until the TTL runs out.
- **Write-through invalidation**: on every write, delete (or update) the matching cache key.
  Fresher, but you must remember to do this on every write path, and it is easy to miss one.

**Cache stampede** (also called "thundering herd") happens when a popular cache key expires,
and many requests arrive for it at the same time. All of them see a cache miss, and all of
them hit the database at once with the same expensive query. This can overload the database
right when the cache was supposed to protect it.

Fixes for cache stampede:
- **Locking / single-flight**: only let one request recompute the value; other requests wait
  for it or get a slightly stale value in the meantime.
- **Early refresh**: refresh the cache entry slightly before it expires (a background job or
  a probabilistic check), so it never fully falls through to zero.
- **Jittered TTLs**: add a small random amount to each TTL, so many keys set at the same time
  do not all expire at the exact same second.

Caching is cheap to add and gives a big win for read-heavy workloads, which is why it comes
before replicas and sharding.

### Step 3: Add read replicas, split reads from writes

A **replica** is a copy of the database that stays in sync with the main database (the
**primary** or **leader**). The primary handles writes. One or more **read replicas** handle
reads. This is called **read/write splitting**. See Chapter 13 for how replication works in
detail.

```
                 writes
  App  ────────────────────────►  Primary (leader)
   │                                  │  replicates changes
   │        reads                     ▼
   └───────────────────────►  Replica 1, Replica 2, ...
```

This helps when your workload is **read-heavy** (many more reads than writes), which is true
for most web applications. Reads can be spread across many replicas, so no single machine is
the bottleneck for reads. Writes still all go to one primary, so this does not help a
write-heavy bottleneck.

The main risk is **replication lag**: replicas apply changes a little after the primary, so
a replica can serve slightly old data for a short time. Common bugs caused by this:
- A user updates their profile, then immediately reloads the page, and the read replica
  still shows the old value (a **read-after-write consistency** problem).
- A report reads from a replica and briefly disagrees with the primary.

Mitigations:
- Route read-after-write cases (like "show me the order I just placed") to the primary, or
  to a replica that is confirmed to be caught up.
- Show a "your changes may take a moment to appear" message for less critical flows.
- Monitor replication lag and alert if it grows too large.

### Step 4: Partition or shard, when one machine is not enough

**Partitioning** splits one large table into smaller pieces, usually still on the same
database server, based on a rule like date range or a hash of a column. **Sharding** goes
further: it splits data across multiple separate database servers (see Chapter 14). You move
to sharding when the data or the write load is too big for any single machine, even after
indexing, caching, and read replicas.

The most important decision when sharding is the **shard key** (also called partition key):
the column used to decide which shard a row lives on. A good shard key:
- Spreads data and traffic evenly across shards (no **hotspot**, meaning no single shard gets
  much more load than the others).
- Matches how the application queries the data, so most queries can go to one shard, instead
  of asking every shard and merging results (a **scatter-gather** query, which is slow).

Example: for the `orders` table, `customer_id` is often a good shard key, because most
queries are "get orders for this customer." Sharding by `order_date` can create a hotspot,
since all of today's writes land on the same shard.

```sql
-- Conceptually: shard_id = hash(customer_id) % number_of_shards
-- orders for customer_id 4821 always land on the same shard
```

Sharding adds real complexity: cross-shard joins are hard, transactions across shards are
hard, and rebalancing shards later is a big migration. This is why it is step 4, not step 1.

### Step 5: Materialized views and denormalized read tables

A **materialized view** is a saved, precomputed result of a query, stored like a table. A
normal SQL view just re-runs its query every time you read it. A materialized view runs the
query once, stores the result, and you refresh it on a schedule or on demand.

```sql
CREATE MATERIALIZED VIEW daily_revenue_by_product AS
SELECT p.product_id, p.name, date_trunc('day', o.order_date) AS day,
       SUM(oi.quantity * oi.unit_price) AS revenue
FROM order_items oi
JOIN orders o ON o.order_id = oi.order_id
JOIN products p ON p.product_id = oi.product_id
GROUP BY p.product_id, p.name, date_trunc('day', o.order_date);

REFRESH MATERIALIZED VIEW daily_revenue_by_product;
```

This is useful for expensive reads: dashboards, reports, and aggregations that scan many
rows and many joins. Instead of recomputing the aggregation on every page load, you compute
it once and serve the saved result. The trade-off is staleness: the view is only as fresh as
its last refresh. `REFRESH MATERIALIZED VIEW CONCURRENTLY` lets reads continue while it
refreshes, but needs a unique index on the view.

A **denormalized read table** is a similar idea done by hand: you copy and combine data from
several normalized tables into one wide table, built for a specific read pattern, instead of
joining at read time. This trades write complexity (you must update the copy) for much
faster, simpler reads.

### Step 6: Move a hot part of the data to a different store (polyglot persistence)

**Polyglot persistence** means using more than one type of database, each for what it is
best at, instead of forcing every workload into one relational database. This is usually
the last step, because it adds an entire new system to run and keep in sync.

Common patterns:
- **Redis for counters and hot, small state**: a "views count" or "likes count" that gets
  updated on every page view would create huge write pressure on a relational database. Redis
  can do atomic increments (`INCR`) in memory, then periodically flush a batch total back to
  the main database.
- **Elasticsearch for search**: full-text search ("find products matching this text, ranked
  by relevance") is not what a relational database does well at scale. Elasticsearch is
  built for this, and syncs from the primary database via change events or a periodic job.
- **A time-series database for metrics**: high-volume, append-only, time-ordered data (like
  monitoring metrics) fits time-series databases (e.g., InfluxDB, TimescaleDB) better than a
  general-purpose relational database.

The trade-off: you now have two sources of data that must stay in sync, and you must decide
what happens if they briefly disagree. This is why polyglot persistence is a late step, used
for a specific hot spot, not a first move.

### Also check: connection pooling

Every database connection uses server memory and resources, even when idle. If every request
opens a new database connection, or if you run many application instances each holding many
open connections, you can exhaust the database's `max_connections` limit. This causes new
requests to fail or queue, and it looks exactly like "the database is slow", even though the
real problem is connection management, not query performance or hardware capacity.

A **connection pool** keeps a small, fixed set of already-open connections and hands them out
to requests as needed, instead of opening a new connection per request. In Java, this is
usually **HikariCP** inside a Spring Boot app. At the database-server level, tools like
**PgBouncer** (for PostgreSQL) pool connections outside the application too, useful when many
application instances would otherwise each hold their own large pool.

Signs the real issue is connections, not query speed:
- The database is "slow" under load but individual query times, checked with `EXPLAIN
  ANALYZE`, look fine.
- Errors like "too many connections" or "connection pool exhausted" appear in logs.
- Adding more application instances makes things worse, not better, because each instance
  opens more connections.

Fix: size the pool correctly (a small pool per instance, not hundreds), reuse connections,
and add a pooler like PgBouncer if you run many application instances against one database.

### Summary table

| Step | Fixes | Cost / risk | Helps with |
|---|---|---|---|
| 1. Query & index tuning | Missing indexes, bad queries, N+1 | Low | Any bottleneck |
| 2. Caching | Repeated reads of the same data | Low–medium (staleness, stampede) | Read load |
| 3. Read replicas | Read-heavy workload | Medium (replication lag) | Read scaling |
| 4. Partition / shard | Data or write volume too big for one machine | High (complexity, migration) | Read + write scaling |
| 5. Materialized views | Expensive aggregations, dashboards | Medium (staleness, refresh cost) | Expensive reads |
| 6. Polyglot persistence | One workload type dominates (search, counters) | High (new system, sync) | A specific hot spot |
| Connection pooling | Too many open connections | Low | Stability under load |

## The Questions They Ask

**Q: Your database is slow. How do you approach fixing it?**
Say you would not jump straight to buying bigger hardware or adding a cache. First, measure:
find the slow queries (using slow query logs or `EXPLAIN ANALYZE`), and check for missing
indexes or bad query patterns. Fix those first, since they are cheap and often solve the
whole problem. If reads are still heavy after that, add caching for hot, read-mostly data.
If reads are still a bottleneck after caching, add read replicas. Only move to partitioning
or sharding if a single machine truly cannot hold or serve the data, since that adds real
complexity. Mention connection pooling too, since a "slow database" is sometimes just
connection exhaustion.
*Follow-up: "What would you check first, in the first 5 minutes?"* Slow query logs and
`EXPLAIN ANALYZE` on the top few slow queries, plus current connection count against
`max_connections`.

**Q: How do you scale reads versus writes?**
These need different fixes. Reads scale well with caching and read replicas, because you can
have many copies of the same data serving reads in parallel. Writes are harder to scale,
because all writes usually must go to one place to stay consistent (the primary). To scale
writes, you need sharding (splitting the write load across multiple primaries, each owning a
different shard key range) or moving write-heavy hot data to a store built for high write
throughput, like Redis for counters. Say clearly: "reads are the easy scaling problem, writes
are the hard one."
*Follow-up: "Why can't you just add more replicas to scale writes?"* Because replicas only
copy the primary; the primary still takes every write itself. More replicas add more read
capacity and more replication overhead on the primary, they do not reduce write load on it.

**Q: When would you add a cache versus a read replica versus sharding?**
- **Cache** first, when the same data is read repeatedly and can tolerate a little
  staleness. Cheapest, fastest to add, no schema change needed.
- **Read replica** when reads are spread across many different queries or rows (so caching
  hit rate would be low), but writes are still small enough for one primary to handle.
- **Sharding** when even the primary alone cannot hold the data or handle the write volume,
  or the working set is too big to fit in memory for caching to help much.
  State the "it depends" clearly: a read-heavy dashboard favors caching; a large multi-tenant
  table with heavy per-tenant writes favors sharding by tenant.

**Q: What is cache invalidation and why is it considered hard?**
Cache invalidation is removing or refreshing cached data when the source data changes. It is
hard because you must find every code path that changes the data and make sure each one also
updates or deletes the matching cache key. Miss one path, and the cache silently serves stale
data with no error to signal the bug. TTLs limit the damage but do not remove the problem.
*Follow-up: "How would you invalidate a cache key that depends on multiple tables?"* Either
invalidate on every write to any of those tables (more invalidation code, but fresher), or
accept a short TTL and skip manual invalidation (simpler, slightly staler).

**Q: What is a cache stampede, and how do you prevent it?**
It happens when a popular cache key expires and many concurrent requests all miss the cache
at once, then all hit the database with the same expensive query at the same time. Prevent it
with a lock so only one request recomputes the value while others wait, by refreshing the
value slightly before it expires, or by adding random jitter to TTLs so keys do not all
expire together.

**Q: How do you pick a shard key?**
Pick a column that spreads data and traffic evenly (no hotspot), and that matches your most
common query pattern, so most queries hit one shard, not all of them. Think about how the
system will grow: a shard key like `customer_id` is stable and query-friendly for an
e-commerce app, while a key like `country` can create a hotspot if most customers are in one
country. Also think about resharding: some shard keys make it easier to add shards later
without moving huge amounts of data.
*Follow-up: "What goes wrong if you pick a bad shard key?"* Hotspotting (one shard takes
most of the load, so sharding does not actually help), or most queries becoming
scatter-gather queries across all shards, which is slower than a single well-indexed table.

**Q: What is replication lag, and what problems can it cause?**
Replication lag is the delay between a write on the primary and that write appearing on a
read replica. It can cause read-after-write bugs (a user does not see their own update right
away) and reports that briefly disagree with the primary. Fix it by routing
read-your-own-write queries to the primary, monitoring lag, and alerting when it grows.

**Q: What is the difference between a materialized view and a normal view?**
A normal view is just a saved query; it runs against live data every time you select from
it. A materialized view runs the query once and stores the result like a table, so reads are
fast, but the data can be stale until the next `REFRESH`. Use a materialized view for
expensive, repeatedly-read aggregations where some staleness is fine.

**Q: When would you use polyglot persistence, and what is the cost?**
Use it when one workload type clearly does not fit your main relational database, such as
full-text search or very high-write counters, and a specialized store solves it far better.
The cost is running and operating another system, and keeping the two stores in sync, which
adds a new class of "the two stores disagree" bugs. Only do this for a specific hot spot, not
as a first move.

**Q: Why do too many database connections hurt performance?**
Each open connection uses memory and other resources on the database server, even when idle.
Past a limit, the database refuses new connections, or performance degrades because resources
that could serve queries are tied up holding connections. Use a connection pool sized
correctly, and a database-side pooler like PgBouncer if many application instances connect to
one database.

## Rapid-Fire

- **Q: First step when the database is slow?** Measure with `EXPLAIN ANALYZE`, fix missing
  indexes and bad queries.
- **Q: Cheapest way to reduce database load?** A cache in front of it (cache-aside pattern).
- **Q: Cache-aside pattern in one line?** Read from cache; on a miss, read the database,
  store it in the cache, then return it.
- **Q: What does TTL control?** How long a cached value lives before it expires.
- **Q: What is cache invalidation?** Removing or refreshing a cache entry when the source
  data changes.
- **Q: What is a cache stampede?** Many requests recomputing the same expired key at once,
  overloading the database.
- **Q: How do you fix a cache stampede?** Lock so one request recomputes it, refresh early,
  or jitter TTLs.
- **Q: What do read replicas fix?** Read-heavy load, by spreading reads across copies of the
  data.
- **Q: What do read replicas not fix?** Write load; all writes still go to the primary.
- **Q: What is replication lag?** The delay before a replica has the primary's latest write.
- **Q: When do you shard?** When one machine cannot hold or serve the data or write load,
  even after caching and replicas.
- **Q: What makes a good shard key?** Even data and traffic spread, and it matches common
  query patterns.
- **Q: What is a hotspot?** One shard (or key) getting far more traffic than the others.
- **Q: Materialized view vs normal view?** Materialized view stores the result; normal view
  re-runs the query each time.
- **Q: What is polyglot persistence?** Using different database types for different
  workloads (e.g., Redis for counters, Elasticsearch for search).
- **Q: Why use a connection pool?** To reuse a small set of open connections instead of
  opening a new one per request.
- **Q: Symptom of connection exhaustion?** "Too many connections" errors, even though
  individual queries are fast.
- **Q: Correct scaling order?** Query/index tuning, then cache, then read replicas, then
  sharding, then materialized views or polyglot stores as needed.

## Common Traps & Mistakes

- **Jumping straight to sharding.** Many candidates hear "database is slow" and go straight
  to sharding, skipping indexing and caching. Interviewers see this as a lack of judgment,
  since sharding is the most complex, highest-risk fix, not the first one.
- **Caching everything blindly.** Not all data is safe to cache. Caching an account balance
  or a permission check without careful invalidation can cause real bugs, like showing a
  stale balance or letting a removed permission still work.
- **Forgetting about cache stampede.** Candidates describe cache-aside correctly but forget
  what happens when a hot key expires under high traffic. Always mention the stampede risk
  and a fix for it.
- **Assuming replicas give the same read-after-write guarantee as the primary.** Reads from
  a replica can be stale by design. Forgetting this leads to bugs where a user's own update
  seems to "disappear" for a moment.
- **Picking a shard key based only on the primary key, without checking query patterns.**
  A shard key that does not match how data is queried leads to slow scatter-gather queries
  across every shard.
- **Confusing partitioning and sharding.** Partitioning splits a table, often on one server.
  Sharding splits data across multiple servers. Interviewers expect you to know the
  difference (see Chapter 14).
- **Treating a materialized view like always-fresh data.** Forgetting to refresh it, or not
  understanding that `REFRESH` can lock reads unless done `CONCURRENTLY`, causes stale
  dashboards or refresh-time outages.
- **Ignoring connection pooling entirely.** A surprising number of "database is too slow"
  incidents are actually connection exhaustion from too many application instances, each
  opening too many connections, not a query performance problem at all.
- **Adding a new data store (polyglot persistence) too early.** This adds a second system to
  operate and a sync problem, for a gain that a cache or an index might have already given
  you more cheaply.
- **Not saying "it depends."** The best answer to "cache vs replica vs shard" names the
  trade-off for each and ties it to the workload (read-heavy vs write-heavy, tolerance for
  staleness), instead of picking one technique as a universal answer.
