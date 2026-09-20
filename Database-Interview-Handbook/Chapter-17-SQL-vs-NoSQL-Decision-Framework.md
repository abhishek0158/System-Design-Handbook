# Chapter 17 — SQL vs NoSQL Decision Framework

"Which database would you use?" is a common interview question. Many candidates fail it because they repeat a myth: "NoSQL scales, SQL does not." This chapter kills that myth and gives you a clear framework to answer the question well.

## Key Concepts

### 1. The myth: "NoSQL scales, SQL does not"

This idea was popular around 2010. It is outdated today. Here is why it is wrong.

Modern relational (SQL) databases scale very far. Postgres and MySQL can handle **read replicas** (copies of the data for read traffic — see Chapter 13), **partitioning** (splitting one big table into smaller pieces — see Chapter 14), and **sharding** (splitting data across many servers). Large companies run SQL databases at massive scale. Sharded Postgres and MySQL power some of the biggest systems in the world.

At the same time, not every NoSQL database is easy to scale for every workload. A document store with many secondary indexes, or a graph database, can also struggle under heavy write load. Scaling depends on the workload and the design, not just the database family.

**The real difference between SQL and NoSQL is not "scales" vs "does not scale."** The real difference is about **data shape** (how your data is naturally structured) and **access patterns** (how you read and write it). Say this clearly in an interview. It shows you understand the topic, not just a slogan.

A second myth to avoid: "SQL means a relational database, NoSQL means fast." Both are wrong framings. SQL is a query language, most closely tied to the relational model. NoSQL is a big umbrella term for very different systems: document stores, key-value stores, wide-column stores, and graph databases (see Chapter 16 for the four NoSQL types in detail). They do not all behave the same way, so "NoSQL" alone is not a full answer in an interview.

### 2. The five questions to ask before picking a database

Use these five questions as your framework. Ask them out loud in the interview, in this order. This shows structured thinking, which interviewers reward more than a fast guess.

**Question 1: What is the shape of the data?**

Data shape means how naturally your data connects and repeats.

- **Relational shape**: many entities with fixed relationships between them (a customer has many orders, an order has many items). Rows have a fixed, well-known set of columns.
- **Document shape**: each record is a self-contained blob (a user profile, a product page) that does not need to join with other records for most reads.
- **Key-value shape**: you just need to fetch a value by a single key, fast (a session token, a cache entry).
- **Graph shape**: the connections *between* records matter more than the records themselves (friend-of-friend, fraud rings, recommendation paths).

**Question 2: What are the main access patterns?**

Ask: will the queries be **known in advance**, or does the team need **flexible, ad-hoc queries** that nobody can predict today?

- If access patterns are fixed and known (e.g., "always fetch a product by its ID"), a key-value or document store can be tuned for that one pattern and be very fast.
- If people will ask new questions all the time (reporting, analytics, "show me all orders above $500 from customers in Germany last month"), you need flexible querying. SQL with `WHERE`, `JOIN`, and `GROUP BY` is built for this. Most NoSQL stores are weak here, because they are not built for ad-hoc joins across data.

**Question 3: Do you need multi-row transactions and strong consistency?**

A **transaction** is a group of operations that must all succeed or all fail together (see Chapter 9). **Strong consistency** means every read sees the latest write, right away, with no delay.

- If you must update two or more rows together, and it would be a serious bug if only one succeeded (e.g., move money from account A to account B), you need real multi-row transactions.
- Relational databases give you this by design (ACID transactions, see Chapter 9).
- Most NoSQL databases give strong guarantees only at the level of a single document or a single key. Multi-document transactions exist in some NoSQL databases (e.g., MongoDB added them), but they are usually slower and less central to the design than in SQL databases.

**Question 4: How big is the scale, and is the workload write-heavy?**

Ask about the numbers: how many rows, how many requests per second, and what is the read/write ratio.

- A few million rows and a few hundred requests per second: a single well-tuned SQL database, maybe with a read replica, is usually enough. Do not over-engineer.
- Tens of millions of writes per day, or a workload that must scale horizontally across many cheap servers with very simple per-key access: this is where key-value and wide-column NoSQL stores were designed to shine (e.g., Cassandra, DynamoDB).
- Remember: SQL databases can also scale horizontally with sharding, but it takes more manual engineering effort than a NoSQL store built for it from day one.

**Question 5: How connected is the data?**

Ask: do queries mostly need to "hop" between many related records (find friends of friends of friends, find the shortest path between two accounts)?

- If yes, a **graph database** (like Neo4j) may fit best, because it stores relationships as first-class data and can traverse them fast (see Chapter 16).
- Relational databases can do joins, but many-hop traversal queries (5+ joins deep) get slow and hard to write. NoSQL document or key-value stores are usually worse at this than SQL, because they do not support joins well at all.

### 3. Comparison table: SQL strengths vs NoSQL strengths

| | SQL (relational) strengths | NoSQL strengths |
|---|---|---|
| Relationships | Strong. Foreign keys, joins across tables. | Weak. Usually denormalized (data repeated inside one document) to avoid joins. |
| Transactions | Strong. Multi-row, multi-table ACID transactions. | Usually limited to a single document/key. Some support multi-document transactions, but it is not the core design. |
| Querying | Flexible. Any `WHERE`, `JOIN`, `GROUP BY` you can imagine, even ones you did not plan for. | Rigid. Fast for the one access pattern you designed for; weak for new, unplanned queries. |
| Schema | Fixed schema, enforced by the database. Catches bad data early. | Flexible (loose) schema. Easy to add new fields per record, but the database will not catch a mistake. |
| Consistency | Strong consistency by default. | Often tunable. Many NoSQL stores default to eventual consistency for higher availability (see Chapter 15). |
| Horizontal scale | Possible, but takes real engineering work (sharding, see Chapter 14). | Often built in from day one, for a specific access pattern. |
| Write speed at huge scale | Good, but joins and constraints add overhead. | Can be very high, because writes are simple key/value or append-style operations. |
| Best for | Data with real relationships, needed transactions, and reporting/analytics needs. | Very high write throughput, simple known access patterns, or naturally document/graph-shaped data. |

Two rows above deserve one more sentence each, because interviewers probe them:

- **Schema.** A "flexible schema" is not automatically a benefit. It means the application code, not the database, is responsible for keeping data correct. This can cause bugs later, when different documents in the same collection have different shapes because no one enforced a rule.
- **Consistency.** "Tunable" means many NoSQL databases let you choose, per request, between fast-but-maybe-stale reads and slower-but-fresh reads. This is a real trade-off, not a free win — see Chapter 15 for the CAP theorem and quorum reads/writes.

### 4. Polyglot persistence: use more than one database

**Polyglot persistence** means using more than one type of database in the same system, each for the job it is best at. Most real systems at scale do this. It is not a compromise — it is normal, good design.

Example: an e-commerce system (Schema B from the shared schemas) might use:
- **PostgreSQL** for orders, payments, and inventory — because these need strong consistency and multi-row transactions (an order and its payment must both succeed or both fail).
- **Elasticsearch** for product search — because it is built for full-text search and ranking, which SQL `LIKE` queries handle poorly.
- **Redis** (key-value store) for the shopping cart and session data — because it needs very fast reads and writes on a simple key, and losing a cart occasionally is not a disaster.
- **A graph database** for "customers who bought this also bought" recommendations — because that is a connected-data query.

In an interview, mentioning polyglot persistence shows maturity. It tells the interviewer you are not looking for one "best" database — you are matching each part of the system to the right tool.

The cost of polyglot persistence: more operational complexity. You now run, monitor, and back up multiple systems. You also need a plan to keep data in sync across them (e.g., an event stream that updates Elasticsearch whenever an order changes in Postgres). Always mention this cost — do not present polyglot persistence as free.

## The Questions They Ask

**Q1: "NoSQL scales better than SQL — true or false?"**

False, stated that simply. Modern SQL databases scale very far, using read replicas, partitioning, and sharding. The real difference between SQL and NoSQL is about data shape and access patterns, not raw scalability. Some NoSQL databases make horizontal scaling easier out of the box for simple key-based access, but that is a design choice, not proof that SQL "cannot" scale.
*Follow-up: "Then why did NoSQL get popular?"* Because early NoSQL databases made horizontal write scaling and flexible schemas easier to set up quickly, at a time when sharding SQL databases by hand was hard. Tooling for SQL sharding has since improved a lot.

**Q2: "Which database would you use to build [some system]?"**

Do not jump to an answer. State your assumptions first (expected scale, read/write ratio, whether relationships matter, whether transactions matter). Then pick one database, and justify it using the five questions above. End with the trade-off you are accepting. See the worked examples below for the exact structure to use.
*Follow-up: "What if scale grows 100x?"* Explain what would break first (usually: too many joins, or a single database server running out of capacity) and what you would change (add read replicas, then partition, then consider a different store for the hot path).

**Q3: "Can you use both SQL and NoSQL in the same system? Give an example."**

Yes — this is called polyglot persistence. Example: Postgres for orders and payments (needs transactions), Redis for session/cart data (needs speed, low risk if lost), and Elasticsearch for search (needs full-text ranking). Mention the cost: you must keep data in sync across systems, usually with an event-driven pipeline.

**Q4: "If NoSQL has a flexible schema, isn't that always better? You can add fields anytime."**

No. A flexible schema pushes the responsibility for data correctness from the database to the application code. If two different services write to the same collection with slightly different field names or types, nothing catches the mistake at write time. This can lead to inconsistent data that is hard to fix later. A fixed schema is a safety net, not just a limitation.

**Q5: "Give an example of a workload where SQL is clearly the wrong choice."**

A system that needs very high write throughput on simple key-based data, with no relationships and no need for joins — for example, storing raw IoT sensor readings by device ID and timestamp, at 100,000 writes per second. A key-value or wide-column store built for this access pattern (e.g., Cassandra) will out-perform a single SQL server, because SQL's constraint checks and index maintenance add overhead you do not need here.

**Q6: "Give an example of a workload where NoSQL is clearly the wrong choice."**

A financial reporting system that needs ad-hoc joins across customers, orders, payments, and refunds, with strict consistency (the numbers must always add up). Document or key-value stores are weak at ad-hoc joins and often give only eventual consistency by default. A relational database is the safer, correct choice here.

## Rapid-Fire

- **Does NoSQL always scale better than SQL?** No. Modern SQL scales far with replication, partitioning, and sharding. The real choice is about data shape and access patterns.
- **What is data shape?** How your data naturally structures and connects: relational, document, key-value, or graph.
- **What is an access pattern?** How your application reads and writes data — a fixed known query vs an ad-hoc flexible query.
- **When do you need SQL?** When you need multi-row transactions, strong consistency, or flexible ad-hoc queries with joins.
- **When do you lean NoSQL?** When the access pattern is simple and known, write volume is very high, or the data is naturally document- or graph-shaped.
- **What is polyglot persistence?** Using more than one type of database in one system, each for the job it fits best.
- **What is the cost of polyglot persistence?** More systems to run and monitor, and a need to keep data in sync across them.
- **What should you always do before answering "which database"?** State your assumptions out loud, then pick one, then justify with trade-offs.
- **Is "flexible schema" always a benefit?** No. It moves the job of catching bad data from the database to the application code.
- **Can NoSQL databases do transactions?** Some support single-document transactions well; multi-document transactions exist in a few, but are usually not the core strength.

## Common Traps & Mistakes

- **Repeating the "NoSQL scales, SQL doesn't" line.** This is the single most common mistake in this topic. It signals outdated knowledge. Always say the real difference is data shape and access pattern.
- **Picking a database without stating assumptions.** Jumping straight to "I'd use MongoDB" without saying what scale, what consistency needs, and what query patterns you assumed, looks like a guess, not reasoning.
- **Treating "NoSQL" as one thing.** Document stores, key-value stores, wide-column stores, and graph databases are very different from each other. Saying "NoSQL is fast" without naming which type is too vague for a 3-4 YOE interview.
- **Ignoring the transaction question.** Candidates often forget to ask "do I need multi-row transactions?" This single question decides a lot of "which database" answers, especially for anything involving money or inventory.
- **Assuming flexible schema means no schema at all.** Even document stores usually need a consistent shape in practice, enforced by application code or a schema validation layer, or the data becomes messy.
- **Forgetting the cost side of polyglot persistence.** Mentioning multiple databases without mentioning the sync and operational cost looks like you have not thought it through.
- **Never mentioning "it depends."** Interviewers want to see trade-off reasoning, not a single "best" answer. Always name the trade-off you are accepting, even after you pick a database.
- **Confusing "eventually consistent" with "wrong."** Eventual consistency is a valid design choice for many systems (e.g., a "like" counter on a post). It becomes a mistake only when the system actually needs strong consistency and the candidate does not notice.

## Worked "Which DB" Examples

**Example 1: Design the data store for a URL shortener.**

Assumptions: very high read volume (redirects), simple access pattern (short code → long URL), no relationships between records, and losing strict consistency for a moment is not a big deal.

Answer: a key-value store (e.g., DynamoDB or Redis backed by a durable store). Justification: the access pattern is a single, known lookup by key. There are no joins and no multi-row transactions needed. High read throughput at low latency matters more than flexible querying. Trade-off accepted: if you later need analytics (e.g., "which links get the most clicks by country and day"), you will need to also feed click events into a separate analytics store, since a key-value store is weak at ad-hoc aggregation.

**Example 2: Design the data store for an e-commerce order and payment system.**

Assumptions: an order and its payment must succeed or fail together, inventory must not go negative, and the business needs reporting across orders, customers, and products (Schema B from the shared schemas).

Answer: PostgreSQL. Justification: this workload needs multi-row ACID transactions (decrement inventory, create the order, and record the payment together), strong consistency (money must be exact), and flexible ad-hoc queries for reporting (`JOIN` across `orders`, `order_items`, `products`, `customers`). Trade-off accepted: scaling writes far beyond a single server will need sharding or read replicas later, which takes more engineering work than a NoSQL store built for horizontal writes from day one — but this system's correctness needs outweigh that cost.

**Example 3: Design the data store for a social network's "friends of friends" feature.**

Assumptions: the core need is to find people connected within 2-3 hops of a user, quickly, and the network has millions of users with many connections each.

Answer: a graph database (e.g., Neo4j), or a specialized graph layer alongside the main store. Justification: this is a connected-data query. In SQL, a friends-of-friends query needs several self-joins on a large table (see the self-join pattern in Chapter 2), which gets slow as the hop count grows. A graph database stores relationships as first-class data and can traverse hops directly. Trade-off accepted: the graph database becomes a second system alongside the main user-profile store (polyglot persistence), which needs a sync pipeline to stay up to date when new friendships are created.
