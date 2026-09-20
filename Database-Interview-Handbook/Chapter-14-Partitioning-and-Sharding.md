# Chapter 14 — Partitioning & Sharding

A single database server has limits: disk size, memory, and how many writes one machine can handle per second. Partitioning and sharding are two ways to break past those limits by splitting one big table into smaller pieces. Interviewers ask about this topic to check if you can reason about scale, not just write queries.

## Key Concepts

### Partitioning vs sharding — the core difference

**Partitioning** means splitting one big table into smaller pieces, called partitions, **inside one database server**. The database still runs on one machine. The query planner knows about the partitions and picks the right one automatically.

**Sharding** means splitting the data **across many separate database servers**. Each server is called a shard. Each shard holds a subset of the rows, in its own, independent database. No single machine holds all the data.

The simplest way to remember it: partitioning is "one server, many parts." Sharding is "many servers, each with a part." Sharding is often built using the same range/list/hash ideas as partitioning, just applied across machines instead of inside one machine. Some engineers use "sharding" loosely to mean both — but in an interview, state the distinction clearly, since it shows you understand the difference between scaling within one box and scaling across many boxes.

| | Partitioning | Sharding |
|---|---|---|
| Where | One database server | Many database servers |
| Who manages it | The database engine (e.g., Postgres) | Your application, or a routing layer |
| Goal | Manage a huge table, speed up scans | Scale beyond one machine's disk/CPU/RAM |
| Cross-part JOIN | Easy — same engine, one query | Hard — needs the app to fan out |
| Transactions | Normal, same engine | Hard across shards (Chapter 9, Chapter 15) |

### Vertical vs horizontal partitioning

These two terms describe **which direction** you cut the table.

**Vertical partitioning** splits a table by **columns**. You keep all rows, but move some columns into a separate table. A common case: a `products` table has a large `description` text column that is rarely read. You move it to a separate `product_details` table, keyed by the same `product_id`. The main `products` table becomes smaller and faster to scan for common queries.

```sql
-- Before: one wide table
products(product_id PK, name, category, price, description, long_specs_json)

-- After: vertical split
products(product_id PK, name, category, price)
product_details(product_id PK/FK, description, long_specs_json)
```

**Horizontal partitioning** splits a table by **rows**. Every partition has the same columns, but a different slice of the rows — for example, orders from 2023 in one partition, orders from 2024 in another. This is the kind most interviewers mean when they say "partitioning," and it is also the basis for sharding.

```
orders table, split horizontally by year:

  orders_2023   <- rows where order_date is in 2023
  orders_2024   <- rows where order_date is in 2024
  orders_2025   <- rows where order_date is in 2025
```

### The three horizontal partitioning strategies

All three strategies decide **which partition a row belongs to**, based on a column called the **partition key** (inside one database) or the **shard key** (across many databases). The three strategies apply to both partitioning and sharding — the difference is just how many machines are involved.

**1. Range partitioning** — each partition holds a range of values. Good fit for time-series data, like orders by date.

```sql
CREATE TABLE orders (
    order_id     BIGINT NOT NULL,
    customer_id  BIGINT NOT NULL,
    order_date   DATE NOT NULL,
    status       TEXT,
    total_amount NUMERIC
) PARTITION BY RANGE (order_date);

CREATE TABLE orders_2024 PARTITION OF orders
    FOR VALUES FROM ('2024-01-01') TO ('2025-01-01');

CREATE TABLE orders_2025 PARTITION OF orders
    FOR VALUES FROM ('2025-01-01') TO ('2026-01-01');
```

A query like `WHERE order_date >= '2025-06-01'` only touches the `orders_2025` partition. The planner skips the rest. This is called **partition pruning**.

**2. List partitioning** — each partition holds a fixed list of values, usually a category.

```sql
CREATE TABLE customers (
    customer_id BIGINT NOT NULL,
    name        TEXT,
    country     TEXT NOT NULL,
    created_at  TIMESTAMP
) PARTITION BY LIST (country);

CREATE TABLE customers_us PARTITION OF customers
    FOR VALUES IN ('US');

CREATE TABLE customers_row PARTITION OF customers
    FOR VALUES IN ('IN', 'UK', 'DE', 'FR');
```

**3. Hash partitioning** — a hash function turns the key into a number, and that number decides the partition. Good when there is no natural range or category, and you just want an even spread.

```sql
CREATE TABLE customers (
    customer_id BIGINT NOT NULL,
    name        TEXT,
    country     TEXT,
    created_at  TIMESTAMP
) PARTITION BY HASH (customer_id);

CREATE TABLE customers_p0 PARTITION OF customers
    FOR VALUES WITH (MODULUS 4, REMAINDER 0);
CREATE TABLE customers_p1 PARTITION OF customers
    FOR VALUES WITH (MODULUS 4, REMAINDER 1);
-- ... p2, p3
```

Here, `hash(customer_id) % 4` picks one of four partitions. Every `customer_id` lands in exactly one partition, and (with a good hash function) the rows spread out evenly.

### Why shard — scaling past one machine

Partitioning helps one server manage a huge table, but the server is still one machine. It still has one CPU budget, one disk, and one write-ahead log (Chapter 12). At some point, write volume is too high for one machine to keep up, no matter how well you tune it, or the data is simply too large for one disk.

Sharding fixes this by adding more machines. Each shard is a full, independent database, so:
- **Writes scale**, because different shards accept writes at the same time, on different machines.
- **Storage scales**, because each shard only needs to hold its own slice of the data, not the whole dataset.
- **Read load also spreads out**, since reads for a given row only hit that row's shard.

The trade-off: you now manage many databases instead of one, and some operations that used to be simple (joins, transactions, unique constraints across the whole dataset) become hard. Sharding is a scaling tool for **writes and storage**; adding a read replica (Chapter 13) is usually the simpler first step if your problem is only "too many reads."

### Choosing a shard key

The **shard key** (also called the partition key when sharding) is the column used to decide which shard a row goes to. This is one of the most important decisions in a sharded system, and it is very hard to change later. Interviewers care a lot about this choice.

A good shard key has two properties:
1. **Even spread of data and traffic.** Every shard should get roughly the same amount of data and roughly the same number of reads/writes.
2. **Most queries can be answered using only the shard key.** If most queries already know the shard key (e.g., "get orders for customer 42"), the application can route the query straight to the right shard, without asking every shard.

**Example of a bad shard key: `country`.**
Suppose you shard a global e-commerce `customers` table by `country`. If the company has a huge customer base in the US and a small one in Iceland, the "US shard" gets far more rows and far more traffic than the "Iceland shard." This is called a **hot shard** (or **hotspot**): one shard does most of the work while others sit idle. Adding more shards does not help, because all the new load still goes to the same "US" shard. The whole point of sharding — spreading load — fails.

**Better shard keys for this case:**
- `customer_id`, hashed. Every customer, regardless of country, spreads evenly across shards. Most customer queries ("show me my orders") already know `customer_id`, so they route to one shard directly.
- A combination, like `hash(customer_id)`, is common in practice — you use the ID for even spread, but a hash function to avoid clustering (e.g., sequential IDs all landing near each other if you used range partitioning on a monotonically increasing ID).

**Rule of thumb:** pick a key that (a) appears in most of your queries' `WHERE` clauses, and (b) has high cardinality (many distinct values) with no single value dominating. `customer_id` or `user_id` are common good choices. `status` (only a few values, one of them usually huge, like "completed") or `country` (skewed population) are common bad choices.

### Consistent hashing — why it matters when servers change

A naive way to pick a shard from a hash is `hash(key) % N`, where `N` is the number of shards. This works, but it has a big problem: if you add or remove a shard, `N` changes, and almost every key's `hash(key) % N` result changes too. That means almost all the data has to move to a new shard. For a large dataset, that is a huge, slow, risky operation.

**Consistent hashing** solves this. The idea:

1. Imagine a circle of numbers, from 0 up to some large maximum, then wrapping back to 0. This is called the **hash ring**.
2. Each shard (server) is placed at one or more points on this ring, based on `hash(server_id)`.
3. Each key (row) is also placed on the ring, based on `hash(key)`.
4. A key belongs to the **first shard found going clockwise** from the key's position on the ring.

```
        hash ring (0 ... max, wraps around)

                Shard A
                  |
      key1 ---->  |
                   \
                    \
      Shard D        \
         |             \
         |              Shard B
      key2 ---->|      /
                 |     /
                 |    /
              Shard C
```

Now, when you add a new shard, it takes a position somewhere on the ring. Only the keys that fall **between the new shard and the previous shard** (going clockwise) need to move — everyone else's "first shard clockwise" is unchanged. Removing a shard has the same small blast radius: only the keys that were assigned to it now move to the next shard clockwise.

This is the key benefit: **adding or removing one server only reshuffles a small fraction of the keys**, roughly `1/N` of the data on average, instead of nearly all of it.

In practice, each physical shard is usually placed at **many points** on the ring (called **virtual nodes**), not just one. This spreads a shard's data more evenly and avoids one shard getting an unlucky, oversized slice of the ring just by chance.

Consistent hashing is used by real systems like Cassandra, DynamoDB, and many custom sharding layers. You do not need to implement it in an interview — you need to explain *why* it beats plain `hash % N` when servers are added or removed.

### The pain of cross-shard queries, joins, and transactions

Once data is spread across shards, some operations that were trivial on one database become genuinely hard.

**Cross-shard queries.** If a query's filter does not include the shard key, the application (or a routing layer) does not know which shard has the answer. It must send the query to **every shard** (called **scatter-gather** or **fan-out**), wait for all of them to respond, then merge the results in the application. This is slower, and it means every shard must be up and fast for the query to complete quickly.

**Cross-shard joins.** A join needs matching rows from two tables, ideally on the same machine, so the database engine can do the matching efficiently. If `orders` are sharded by `customer_id`, but `order_items` are sharded differently, joining them means pulling rows from multiple shards over the network and joining them in application code — the database engine cannot do it for you across shards. The usual fix is to **shard related tables by the same key**, so an order and its order_items always land on the same shard (this is sometimes called **co-location**). Even then, a join against a table that is *not* sharded the same way (e.g., a small `products` table) usually means either replicating that small table to every shard, or doing the join in the application after fetching from both sides.

**Cross-shard transactions.** A normal transaction (Chapter 9) is atomic on one database: it holds locks and a write-ahead log on one engine. If a transaction must update rows on two different shards — for example, transferring loyalty points from a customer on shard A to a customer on shard B — there is no single engine to make it atomic. This needs a **distributed transaction protocol**, like two-phase commit, or an application-level pattern like the **saga pattern** (a sequence of local transactions with compensating actions if a later step fails). Both add real complexity and latency compared to a normal single-database transaction. This connects directly to the CAP trade-offs in Chapter 15: strong cross-shard consistency costs availability and latency.

### Resharding and rebalancing are hard

Even with a good shard key and consistent hashing, moving data between shards later (resharding) is one of the hardest operational tasks in a sharded system:

- Data must be copied to the new shard **without downtime**, while writes keep happening.
- Old and new locations must briefly agree on where a key "lives," to avoid lost or duplicated writes during the move.
- Client applications and routing layers need updated shard maps, without a moment where they disagree about which shard owns a key.

This is why the shard key decision matters so much upfront: consistent hashing makes resharding *less painful*, but it never makes it free. Many real systems (like Vitess for MySQL, or Citus for Postgres) build entire tools just to manage this migration safely. If you are asked to design a sharded system in an interview, mentioning that you planned for resharding — for example, by over-provisioning shard count early, or building a routing layer that can be updated safely — is a strong signal.

## The Questions They Ask

**Q1: What is the difference between partitioning and sharding?**
Partitioning splits one big table into smaller parts on **one** database server; the engine's query planner handles it, and cross-partition queries and joins still work normally. Sharding splits data across **many** database servers; the application or a routing layer must handle finding the right shard, and cross-shard joins and transactions become hard. Partitioning is often the first step; sharding is what you do when one server truly cannot hold or serve all the data.
*Follow-up:* "Can you have both?" — Yes. Each shard can itself be internally partitioned. For example, each shard might range-partition its local `orders` table by month, while the shards themselves are chosen by hashing `customer_id`.

**Q2: The database's write throughput is maxed out on one machine. How do you scale writes?**
First check if the bottleneck is really about writes and not something fixable — like a missing index causing slow writes due to lock contention, or too many small transactions instead of batching. If write volume genuinely exceeds what one machine can do, adding read replicas (Chapter 13) does not help, since replicas do not accept writes in the common single-leader setup. The real fix is sharding: split the data (and its write load) across multiple independent database servers, each handling a slice of the writes. State clearly that this adds real complexity — cross-shard joins and transactions — so you should only shard once you have exhausted vertical scaling (a bigger machine) and query/index tuning, since those are far simpler.
*Follow-up:* "What if the write hot spot is on a small number of rows, like one popular product's stock count?" — Sharding by a coarse key does not fix a hot single row. That needs a different technique: batching updates, using an in-memory counter that flushes periodically, or splitting the counter itself into multiple shards that are summed on read.

**Q3: How do you pick a shard key for a new system?**
Look at your most common query patterns first, before picking anything. Choose a key that (a) appears in the `WHERE` clause of most queries, so the application can route directly to one shard, and (b) has high cardinality with an even distribution, so no single shard gets much more data or traffic than the others. For example, in a multi-tenant system, `tenant_id` is often a strong shard key if most queries are already scoped to one tenant. Avoid keys with a skewed value distribution, like `country` or `status`, since one value can dominate and create a hot shard. Always say this decision is hard to change later, so it is worth modeling expected traffic and data skew before committing.
*Follow-up:* "What if no single column gives even distribution or good query routing?" — Consider a composite or derived key (e.g., `hash(tenant_id) + region`), or accept that some queries will need scatter-gather across shards, and design for that cost explicitly rather than pretending it will not happen.

**Q4: Why are cross-shard joins hard? How would you handle them?**
A join is efficient when the database engine can match rows on one machine, usually using an index. Across shards, the two sides of a join can live on different servers, so there is no single engine that can do the matching — the application has to fetch rows from multiple shards over the network and join them in code, which is slower and more complex. The standard mitigation is **co-location**: shard related tables (like `orders` and `order_items`) by the same key, so a customer's orders and their order items always land on the same shard, keeping the join local. For a small, slow-changing table needed by every shard (like `products` or `departments`), a common trick is to **replicate** that whole table to every shard, so joins against it stay local everywhere.
*Follow-up:* "What if you genuinely need to join two large tables that cannot share a shard key?" — Say plainly that this is a real limitation of sharding: either denormalize (duplicate the needed columns into both tables so no join is needed), or accept an application-side join with the performance cost, or use a separate analytics system (a data warehouse fed by change data capture) for queries like this instead of the sharded operational database.

**Q5: What is consistent hashing, and why is it better than `hash(key) % N`?**
With plain `hash(key) % N`, changing the number of shards `N` changes almost every key's target shard, forcing a near-total data reshuffle whenever a server is added or removed. Consistent hashing places both shards and keys on a conceptual ring using a hash function; each key belongs to the next shard found going clockwise. Adding or removing one shard only moves the keys between it and its neighbor on the ring — roughly `1/N` of the data — leaving everything else in place. This makes scaling the cluster up or down far cheaper.
*Follow-up:* "What are virtual nodes, and why are they used?" — Placing each physical shard at many points on the ring, instead of one, spreads its share of the keyspace more evenly and avoids one unlucky shard getting a disproportionately large or small slice just from where its single hash landed.

**Q6: Why is resharding (changing the number of shards later) so hard?**
Because it means physically moving data between live servers without downtime and without losing or duplicating writes during the move, while every part of the system (application routing, in-flight transactions) needs a consistent view of which shard currently owns a given key. Consistent hashing reduces how much data must move, but the migration itself — copying data, cutting over safely, keeping routing correct mid-migration — is still an operational challenge. This is why picking a good shard key upfront, and sometimes over-provisioning the number of logical shards early (more logical shards than physical machines, so you can move logical shards between machines later without rehashing), is a common practical strategy.

**Q7: If you shard a database, how do you keep something like an auto-incrementing primary key unique across all shards?**
A simple per-shard auto-increment will produce duplicate IDs across shards (both shard 1 and shard 2 might generate ID `501`). Common fixes: generate IDs centrally (a dedicated ID-generation service or sequence), embed the shard number inside the ID itself (e.g., high bits = shard ID, low bits = a local counter, similar to Twitter's Snowflake ID scheme), or use globally unique IDs like UUIDs, accepting they are larger and not naturally sortable by creation time unless you pick a time-ordered UUID variant.

## Rapid-Fire

- **Partitioning** → split one table into parts, inside one database server.
- **Sharding** → split data across many database servers.
- **Vertical partitioning** → split by columns (e.g., move a rarely-read text column to another table).
- **Horizontal partitioning** → split by rows (e.g., orders by year); the basis for sharding.
- **Range partitioning** → split by a value range; good for time-series data; enables partition pruning.
- **List partitioning** → split by a fixed set of category values.
- **Hash partitioning** → split by `hash(key) % N`; good for even spread with no natural range/category.
- **Why shard** → scale writes and storage past what one machine can hold or handle; not a fix for a read-only bottleneck (use replicas for that, Chapter 13).
- **Good shard key** → matches most query filters, high cardinality, even data/traffic spread.
- **Bad shard key example** → `country`; a large country becomes a hot shard, and adding shards does not fix it.
- **Hot shard / hotspot** → one shard gets far more data or traffic than the others.
- **Consistent hashing** → places shards and keys on a ring; adding/removing a shard only moves keys near it, not the whole dataset.
- **Virtual nodes** → one physical shard placed at many ring points, for a more even spread.
- **Cross-shard query without the shard key** → must fan out to every shard (scatter-gather), then merge in the app.
- **Cross-shard join** → hard, because the engine can't match rows across machines; fix with co-location (shard related tables by the same key) or replicate small reference tables to every shard.
- **Cross-shard transaction** → needs two-phase commit or the saga pattern; not a normal single-engine transaction.
- **Resharding** → moving data when shard count changes; hard to do live without downtime, even with consistent hashing.

## Common Traps & Mistakes

- **Using "partitioning" and "sharding" as if they are the same thing.** They solve different problems. Partitioning does not add machines; sharding does. State the distinction clearly if asked.
- **Picking a shard key based only on what looks "natural," like `country` or `signup_date`, without checking distribution.** Always check: will one value or range dominate? A skewed key creates a hot shard no matter how many shards you add.
- **Forgetting that a good shard key also needs to match query patterns.** Even distribution alone is not enough — if queries do not include the shard key, they must fan out to every shard, losing most of the benefit of sharding.
- **Assuming sharding fixes a read scaling problem.** If the real problem is too many reads, a read replica (Chapter 13) is simpler and avoids the cross-shard join/transaction complexity entirely. Only shard when writes or storage — not reads — are the bottleneck.
- **Thinking consistent hashing means zero data movement when a shard is added.** It means only a **fraction** of the data moves (roughly `1/N`), not none. Some rebalancing is still required.
- **Ignoring co-location when designing related tables for sharding.** If `orders` and `order_items` are sharded by different keys, every join between them becomes a cross-shard join. Shard related tables by the same key whenever they are usually queried together.
- **Assuming a cross-shard transaction behaves exactly like a normal ACID transaction (Chapter 9).** Without a distributed transaction protocol, a failure partway through can leave shards inconsistent. This must be designed for explicitly, not assumed away.
- **Under-estimating how hard resharding is.** Treating "we'll just add more shards later" as a free option ignores the real migration cost. It is fine to reshard later, but say out loud that it is a real project, not a config change.
- **Using a plain auto-increment primary key per shard without a plan for global uniqueness.** This produces duplicate IDs across shards. Plan for this with a shard-aware ID scheme, a central ID generator, or UUIDs.
