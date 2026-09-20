# Chapter 16 — NoSQL Types & When to Use

**NoSQL** is a short name for "not only SQL" — databases that do not use the relational
table-and-join model. Interviewers ask about NoSQL because real systems mix it with SQL, and
they want to know if you can pick the right tool, not just the one you know best. This chapter
covers the four main NoSQL families, a real database for each, and how modeling data for NoSQL
is different from modeling it for SQL.

## Key Concepts

### 1. Why NoSQL appeared

Relational databases (like Postgres or MySQL) store data in tables with fixed columns, and they
enforce relationships with foreign keys and joins. This works very well for many systems. But
around the mid-2000s, some companies hit three problems that relational databases struggled with
at large scale:

- **Scale-out.** A single relational database server has a limit on how much traffic and data it
  can handle. The classic fix is a bigger machine (**scale-up**). But at some point, buying a
  bigger machine is not enough, or gets very expensive. **Scale-out** means adding more machines
  and spreading data across them, instead of buying one huge machine. Relational databases were
  historically hard to scale out, because joins and strict consistency across many machines are
  expensive. Many NoSQL databases were built to scale out from day one.
- **Flexible schema.** A relational table has a fixed set of columns, decided in advance. Adding
  a column later means an `ALTER TABLE`, which can be slow or risky on a huge table. Some
  applications — for example, a product catalog where a "laptop" and a "T-shirt" have very
  different attributes — do not fit neatly into one fixed set of columns. NoSQL databases often
  let each record have a different shape, with no upfront schema.
- **Specific access patterns.** Some applications need one very specific, very fast operation —
  for example, "look up a user's session by session ID" — millions of times per second. A
  general-purpose relational engine, built to handle arbitrary joins and queries, carries
  overhead that a specialized store does not need. NoSQL databases are usually built around one
  or two access patterns and optimized hard for those, instead of trying to support every kind
  of query well.

**Important nuance for interviews:** NoSQL is not "better" than SQL. It trades away some
features — usually joins, and often strict transactional guarantees across multiple records — to
gain scale-out and flexibility. Chapter 17 covers the full decision framework for choosing
between SQL and NoSQL. This chapter focuses on the four NoSQL families themselves.

### 2. The four main families

There are more than four kinds of NoSQL databases in the wild, but almost every interview
question maps onto these four families. Each one is built around a different shape of data and a
different access pattern.

| Family | Shape of data | Real database | Good use case | Bad use case |
|---|---|---|---|---|
| Key-value | key → opaque value | Redis, DynamoDB | Caching, sessions, counters | Complex queries across records |
| Document | key → JSON-like document | MongoDB | Flexible or nested records (product catalog, CMS) | Data with many-to-many relationships needing joins |
| Wide-column | row key + column families, sparse | Cassandra | High write volume, time-series, huge scale | Ad-hoc queries not matching the row key design |
| Graph | nodes + relationships (edges) | Neo4j | Relationship-heavy data (social graph, recommendations) | Simple lookups or bulk aggregate reports |

The rest of this section explains each family in more depth.

### 3. Key-value stores

A **key-value store** is the simplest NoSQL model. Every piece of data is a pair: a **key**
(a unique identifier, like a string) and a **value** (whatever you store under that key — a
number, a string, a blob of JSON, anything). The database does not look inside the value. It
only knows how to store a value under a key, and fetch a value by key. There is no query
language for searching inside values, and usually no joins at all.

**Real database:** Redis is an in-memory key-value store, often used as a cache. DynamoDB is
AWS's managed key-value (and document) database, built to scale out automatically and stay fast
at huge request volumes.

**How you use it:**

```
SET session:abc123  "{ user_id: 42, expires_at: ... }"
GET session:abc123
INCR page_views:homepage
```

- `SET` and `GET` store and fetch a value by key — like a giant, distributed hash map.
- `INCR` atomically increases a number stored at a key — useful for counters, because it avoids
  the read-then-write race condition you would get doing "read value, add 1, write value back"
  in application code.

**Good use case: caching, sessions, counters.**
- **Caching:** store the result of an expensive database query or computation under a key (for
  example, `product:1001`), so the next request for that product skips the expensive work and
  just reads the cached value. Chapter 21 covers caching layers in more depth.
- **Sessions:** a web session (who is logged in, what is in their shopping cart) is naturally a
  single blob of data looked up by a session ID. No joins are needed.
- **Counters:** view counts, like counts, rate-limit counters. These need one atomic
  increment operation, not a relational query.

**Bad use case: complex queries across records.** If you need "find all sessions created in the
last hour where the user is from Germany," a key-value store cannot help — it has no way to
search by anything other than the exact key. You would need to scan every key, which defeats the
purpose. If your access pattern needs filtering, sorting, or joining across many records, a
key-value store is the wrong tool.

### 4. Document stores

A **document store** keeps data as **documents** — self-contained records, usually written in a
JSON-like format, that can have nested structures (a list inside a field, an object inside an
object). Unlike a key-value store, the database understands the structure of the document. You
can query by fields inside the document, not just by a single key.

**Real database:** MongoDB is the most common document database used in interviews.

**Example document**, for a product in a catalog (using ideas close to the shared e-commerce
schema, `products`, from earlier chapters):

```json
{
  "_id": "P1001",
  "name": "Wireless Mouse",
  "category": "Electronics",
  "price": 25.99,
  "attributes": {
    "color": "black",
    "wireless": true,
    "battery_type": "AA"
  },
  "reviews": [
    { "user": "alice", "rating": 5, "comment": "Great mouse" },
    { "user": "bob",   "rating": 4, "comment": "Works well" }
  ]
}
```

Compare this to the relational `products` table from Chapter 2 onward, which has fixed columns
(`product_id`, `name`, `category`, `price`). In the relational world, reviews would live in a
separate `reviews` table, joined to `products` by `product_id`. In the document world, the
reviews are **embedded** directly inside the product document. One read gets the product and all
its reviews together, with no join.

**Good use case: flexible or nested records.** A product catalog is a good example, because
different product types have different attributes — a laptop has "RAM" and "screen size," a
T-shirt has "size" and "color." Forcing this into one fixed relational table means either many
`NULL` columns or a messy key-value "attributes" table. A document lets each product store only
the fields it actually needs. A content management system (CMS), where each article can have a
different set of custom fields, is another common example.

**Bad use case: data with many-to-many relationships needing joins.** If your data is naturally
relational — for example, students enrolled in many courses, and courses having many students —
a document store forces you to either duplicate data heavily or perform joins in application
code, which the database does not help with. Chapter 17 discusses this trade-off further.

### 5. Wide-column stores

A **wide-column store** organizes data by a **row key** and groups related columns into **column
families**. Unlike a relational table, not every row needs to have the same columns — the data
is **sparse**, meaning a row can skip columns it doesn't need at almost no storage cost. Wide-
column stores are built to spread data across many machines and handle a very high rate of
writes.

**Real database:** Cassandra is the classic example. (HBase, built on Hadoop, is another; Google
Bigtable is the original design both were inspired by.)

**Example**, storing sensor readings by device and time (a time-series pattern):

```
Row key: device_id = "sensor-42"
Columns: 2026-09-01T10:00 -> 21.5°C
         2026-09-01T10:05 -> 21.7°C
         2026-09-01T10:10 -> 21.6°C
         ...
```

Each device gets one row, and new readings are just new columns added to that row. Cassandra
decides which machine stores a row based on the row key (this uses **consistent hashing**,
covered in Chapter 14 on partitioning and sharding). All queries in Cassandra should go through
the row key (or a well-designed partition key) — this is the access pattern the whole system is
built around.

**Good use case: very high write volume, time-series, huge scale.** Examples: IoT sensor data,
application logs, metrics, event tracking (a user clicked a button, a page was viewed). These
workloads write constantly, rarely update old data, and need to handle spikes without falling
over. Cassandra is designed to accept writes very fast (explained more in "The Questions They
Ask" below) and to keep working even if some machines in the cluster fail or go offline.

**Bad use case: ad-hoc queries not matching the row key design.** Cassandra query performance
depends entirely on how well your query matches the partition key you designed up front. If you
suddenly need to ask "find all sensors with a reading above 30°C, across all devices, at any
time" — a query that does not go through the row key — Cassandra will perform very badly or
refuse the query entirely. A relational database with a proper index handles this kind of
unplanned, ad-hoc query far more gracefully.

### 6. Graph databases

A **graph database** stores data as **nodes** (things, like a person or a product) and
**relationships** (also called edges — the connections between nodes, like "FRIENDS_WITH" or
"BOUGHT"). Both nodes and relationships can carry their own properties (for example, a
"FRIENDS_WITH" relationship might have a `since` date). The defining strength of a graph database
is **traversal** — walking from node to node, following relationships, efficiently, even many
hops deep.

**Real database:** Neo4j is the most common graph database mentioned in interviews. It uses a
query language called **Cypher**.

**Example**, a small social graph in Cypher:

```
(alice:Person)-[:FRIENDS_WITH]->(bob:Person)
(bob:Person)-[:FRIENDS_WITH]->(carol:Person)
```

A "friends of friends" query — find people who are friends with Alice's friends, but not friends
with Alice herself:

```cypher
MATCH (alice:Person {name: 'Alice'})-[:FRIENDS_WITH]->(friend)-[:FRIENDS_WITH]->(fof)
WHERE fof <> alice
  AND NOT (alice)-[:FRIENDS_WITH]->(fof)
RETURN DISTINCT fof.name;
```

This traverses two relationship hops from Alice. In a relational database, the same query needs
a **self-join on the `employees`-style table twice** (once per hop), and a three-hop or four-hop
query needs three or four joins, each one slower. In a graph database, each hop is a direct
pointer-following step, so the traversal cost grows much more slowly as the number of hops
increases — because the relationships are stored physically as connections, not recomputed by
matching keys at query time.

**Good use case: data that is mostly about relationships and traversals.** Social networks
(friends, followers), recommendation engines ("people who bought this also bought..."), fraud
detection (finding rings of connected accounts), and knowledge graphs are the classic examples.
Anything where the *question* is naturally phrased as "how are these things connected" fits a
graph database well.

**Bad use case: simple lookups or bulk aggregate reports.** If your workload is mostly "fetch
this one record by ID" or "sum total revenue by month across all orders," a graph database adds
complexity for no benefit. Bulk scans and aggregations over the whole dataset are not what graph
databases are optimized for — a relational or wide-column store handles that better and simpler.

### 7. How NoSQL modeling differs from relational modeling

This is one of the most important ideas in this chapter, and a common interview question on its
own.

In relational modeling (Chapter 19 covers this fully), you normalize: you design tables to avoid
storing the same fact twice, and you use joins at query time to bring related data back together.
The schema is designed around the **entities** — customers, orders, products — without needing
to know every query in advance.

In NoSQL modeling, the rule flips: **you model your data around the queries you need to run, not
around the entities.** Since most NoSQL databases have weak or no join support, you often
**duplicate (denormalize) data on purpose**, so that one read (one document, one row, one key
lookup) has everything the query needs, without a join.

**Worked example.** Take the shared e-commerce schema: `customers`, `orders`, `order_items`,
`products`. In SQL, showing an order with the customer's name and every item in it needs a join
across three or four tables:

```sql
SELECT c.name, o.order_id, o.order_date, p.name AS product_name, oi.quantity, oi.unit_price
FROM orders o
JOIN customers c ON c.customer_id = o.customer_id
JOIN order_items oi ON oi.order_id = o.order_id
JOIN products p ON p.product_id = oi.product_id
WHERE o.order_id = 501;
```

In a document store, if the main access pattern is "show me one order with everything in it,"
you design the document to already contain that:

```json
{
  "order_id": 501,
  "customer_name": "Priya Shah",
  "order_date": "2026-08-14",
  "items": [
    { "product_name": "Wireless Mouse", "quantity": 2, "unit_price": 25.99 },
    { "product_name": "USB Cable",      "quantity": 1, "unit_price": 5.50 }
  ]
}
```

Notice `customer_name` and `product_name` are copied directly into the order document, instead
of being looked up through a foreign key at read time. One read returns everything — no join
needed. The trade-off: if a customer changes their name, every order document that copied that
name is now stale, unless the application updates them all. This is the classic NoSQL trade-off:
**faster, simpler reads, in exchange for update complexity and duplicated data.**

The same idea applies to wide-column stores: you design the row key and column layout around
"what will I query by," not around "what are the clean, non-duplicated entities." And it applies
loosely to graph databases too, though there the main design decision is which things should be
**nodes** versus **properties on a node** (should "city" be its own node, connected by a `LIVES_IN`
relationship, or just a text property on the person? — it depends on whether you need to query
"who else lives in this city" as a first-class traversal).

**Rule of thumb to say out loud in interviews:** "In SQL, model the entities and figure out
queries later, using joins. In NoSQL, figure out the queries first, then model the data — often
duplicating data — so each query needs one lookup, not a join."

## The Questions They Ask

**Q1: What are the main types of NoSQL databases? Give a real example of each.**
Four main families: key-value (Redis, DynamoDB), document (MongoDB), wide-column (Cassandra),
and graph (Neo4j). Each is built around a different shape of data and a different access
pattern — key-value for simple lookups, document for flexible nested records, wide-column for
very high write volume at scale, and graph for relationship-heavy traversal queries.
*Follow-up:* "Which one would you use for a shopping cart?" A key-value store like Redis is a
common choice — the cart is naturally "one blob of data per user session," looked up by a single
key, with no need for joins.

**Q2: Give a use case for each of the four NoSQL types.**
- Key-value: caching, session storage, counters (Redis, DynamoDB).
- Document: a product catalog or CMS with varying, nested fields per record (MongoDB).
- Wide-column: high-volume time-series data, like IoT sensor readings or application logs
  (Cassandra).
- Graph: a social network's friend graph, or a recommendation engine (Neo4j).
  *Follow-up:* "Could you use a document store for the time-series use case instead of
  wide-column?" You could, but at very high write volumes and huge data sizes, wide-column stores
  like Cassandra are usually a better fit — they are built specifically to sustain extremely high
  write throughput and scale out across many machines with minimal coordination between them.

**Q3: How is data modeling in NoSQL different from data modeling in a relational database?**
In a relational database, you normalize the data into separate tables to avoid duplication, and
you bring related data back together with joins at query time. In NoSQL, most databases have
weak or no join support, so you model your data around your queries: you design each record (a
document, a row, a key's value) to already contain everything one query needs, even if that
means duplicating (denormalizing) the same piece of data in multiple places. The trade-off is
faster, simpler reads in exchange for more complex, sometimes inconsistent, updates.
*Follow-up:* "What happens if the duplicated data changes — like a customer's name?" You either
accept that older copies are stale until updated, or you update every copy (which can be slow or
require a background job), or you avoid duplicating fields that change often. This is a real
design decision, and interviewers want you to say this trade-off out loud, not pretend it away.

**Q4: Why is Cassandra good for high write volume?**
Cassandra uses a storage engine based on a **log-structured merge-tree (LSM tree)**, covered in
depth in Chapter 12. In short: writes are first appended to an in-memory structure and a
write-ahead log, both very fast **sequential** operations, instead of Cassandra hunting for the
right place to update data on disk (like a B-tree index update would). Because Cassandra rarely
does slow, random-access updates on write, it can sustain very high write throughput. On top of
that, Cassandra scales out: data is partitioned across many machines using consistent hashing
(Chapter 14), so as you add more machines, total write capacity keeps growing.
*Follow-up:* "Does this speed come free?" No. Reads can be more expensive than in a B-tree-based
database, because Cassandra may need to check multiple on-disk files to answer a read. Cassandra
also uses **eventual consistency** by default (Chapter 15 covers CAP and consistency) — a read
right after a write might not see that write yet on every replica, depending on the consistency
level you choose.

**Q5: When would you avoid NoSQL and stick with a relational database?**
When your data has many relationships that need to be queried flexibly and you cannot predict
every access pattern in advance, when you need strong multi-record transactional guarantees
(ACID, Chapter 9), or when the team needs ad-hoc reporting with joins and aggregations that were
not designed for upfront. Relational databases are the safer default when requirements are still
changing, because they do not force you to pick your queries before you build the schema.
Chapter 17 has the full decision framework.

## Rapid-Fire

- **What does NoSQL stand for?** "Not only SQL" — non-relational databases.
- **Three reasons NoSQL appeared?** Scale-out, flexible schema, and optimizing for specific
  access patterns.
- **Key-value store, one line?** Key looks up an opaque value; no querying inside the value.
- **Document store, one line?** Key looks up a JSON-like document; you can query fields inside
  it.
- **Wide-column store, one line?** Row key plus sparse column families, built for huge scale and
  high write volume.
- **Graph database, one line?** Nodes and relationships, optimized for traversal queries.
- **Redis is used for?** Caching, sessions, counters.
- **MongoDB is used for?** Flexible or nested records, like a product catalog.
- **Cassandra is used for?** Time-series data, logs, very high write volume, huge scale.
- **Neo4j is used for?** Friend-of-friend queries, recommendations, fraud rings.
- **Main modeling difference vs SQL?** Model around your queries, and duplicate data on purpose,
  instead of normalizing and joining.
- **Why is Cassandra fast at writes?** LSM-tree storage engine (sequential writes), plus
  scale-out partitioning across machines.
- **What do NoSQL databases usually give up?** Joins, and often strong multi-record
  transactions.

## Common Traps & Mistakes

- **Saying "NoSQL is faster than SQL."** This is too vague to be a real answer. NoSQL databases
  are optimized for specific access patterns, often by giving up joins or strong consistency.
  Say what is being traded away, not just "faster."
- **Picking a NoSQL type without a use case.** If asked "would you use MongoDB or Cassandra
  here," never answer with just the database name. Say what access pattern makes it a good fit
  (or a bad one).
- **Forgetting that NoSQL databases still need modeling.** A common beginner mistake is treating
  a document store as "no design needed, just dump the JSON." Poor document design (for example,
  embedding data that changes often and is shared across many documents) causes real production
  pain later.
- **Confusing "flexible schema" with "no rules."** A document store lets different documents have
  different fields, but the application still needs a consistent idea of what a "valid" document
  looks like. Skipping this leads to messy, inconsistent data over time.
- **Assuming a graph database is always the right choice for "relationships."** Every relational
  database also models relationships, using foreign keys. A graph database is worth it
  specifically when the *query* is a deep or unpredictable traversal (many hops, or "how are
  these two things connected") — not just "this table has a foreign key to that table."
- **Ignoring the consistency trade-off.** Many NoSQL databases (including Cassandra and
  DynamoDB) use eventual consistency by default for speed and availability. Candidates who assume
  every read always sees the latest write can walk into a design mistake. Chapter 15 covers this
  trade-off (CAP theorem) in detail.
- **Trying to force a many-to-many, highly relational domain into a document store.** If the data
  is naturally a web of relationships queried in many different ways, denormalizing heavily into
  documents leads to painful, error-prone updates. This is often a sign the domain fits a
  relational or graph database better.

*Next: Chapter 17 gives the full SQL vs NoSQL decision framework — a structured way to reason
through this trade-off out loud in an interview, instead of guessing.*
