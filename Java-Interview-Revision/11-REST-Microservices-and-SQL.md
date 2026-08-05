# 11. REST, Microservices & SQL

This sheet covers three areas that come up together in backend Java interviews: REST API design, microservices basics, and SQL fundamentals. A 3–4 year Spring Boot developer is expected to know all three at a practical level, not just theory.

## Key Concepts (Quick Revision)

**Part A — REST**
- REST (Representational State Transfer) is a style for designing APIs over HTTP. Resources are identified by URLs. Actions use HTTP methods.
- Main methods: GET (read), POST (create), PUT (replace), PATCH (partial update), DELETE (remove).
- Safe method: does not change server state (GET, HEAD). Idempotent method: calling it many times gives the same result as calling it once (GET, PUT, DELETE). POST is neither safe nor idempotent.
- Status codes: 2xx success, 3xx redirect, 4xx client error, 5xx server error.
- Path variable = which resource. Query param = filter/sort/page. Body = data for create/update.
- DTO (Data Transfer Object) is a plain object used only to move data in and out of the API. Do not return JPA entities directly.
- API versioning keeps old clients working while the API changes.
- Pagination avoids sending huge lists in one response.
- REST is stateless: server does not store client session between calls.
- HATEOAS adds links in the response so the client can discover next actions.
- Idempotency key lets a client retry a POST safely without creating duplicates.

**Part B — Microservices**
- Monolith: one deployable unit. Microservices: many small, independently deployable services.
- Services talk sync (REST, gRPC) for immediate answers, or async (Kafka, RabbitMQ) for events and decoupling.
- Service discovery: services find each other's network location automatically (e.g. Eureka).
- API gateway: single entry point that routes requests to the right service.
- Database-per-service: each service owns its data, so services can change independently.
- Distributed transaction: a transaction that spans multiple services. Saga pattern manages this using a sequence of local transactions plus compensating actions.
- Resilience patterns: timeout, retry, circuit breaker protect a service from a slow or failing dependency.
- Centralized logging and tracing help debug requests that cross many services.
- 12-factor app: a set of practices for building cloud-friendly services (config in environment, stateless processes, etc.).

**Part C — SQL**
- Joins combine rows from two or more tables.
- GROUP BY groups rows; HAVING filters groups; WHERE filters rows before grouping.
- Index speeds up reads on a column but slows down writes (insert/update/delete) because the index must also update.
- ACID: Atomicity, Consistency, Isolation, Durability — properties of a reliable transaction.
- Isolation levels control what one transaction can see of another's uncommitted or in-progress changes.
- Normalization removes duplicate data by splitting tables. Denormalization adds duplication back for read speed.

---

## PART A — REST APIs

### What is REST

REST means Representational State Transfer. It is a set of rules for designing APIs over HTTP. Each thing you work with (a user, an order) is a **resource** with a URL, like `/users/42`. You act on it using HTTP methods.

### HTTP methods and when to use each

| Method | Use | Body? |
|---|---|---|
| GET | Read a resource or a list | No |
| POST | Create a new resource, or trigger an action | Yes |
| PUT | Replace a resource fully | Yes |
| PATCH | Update part of a resource | Yes |
| DELETE | Remove a resource | Usually no |

Example: `POST /orders` creates an order. `GET /orders/10` reads order 10. `PUT /orders/10` replaces order 10 with new data. `PATCH /orders/10` changes only one field, like status. `DELETE /orders/10` removes it.

### Safe vs idempotent methods

- **Safe** method: it does not change anything on the server. You can call it any number of times with no side effect. Example: GET, HEAD.
- **Idempotent** method: calling it once or many times leaves the server in the same state. It can still change data, but repeating it does not change the result further.

| Method | Safe? | Idempotent? |
|---|---|---|
| GET | Yes | Yes |
| PUT | No | Yes |
| DELETE | No | Yes |
| PATCH | No | Not guaranteed |
| POST | No | No |

Why PUT and DELETE are idempotent:
- PUT replaces a resource with a fixed value you send. Send the same PUT ten times, the resource ends up the same as after the first call.
- DELETE removes a resource. Delete it once, it's gone. Delete it again, it's still gone (server usually returns 404 or 204 on the repeat, but the end state — resource absent — does not change).

Why POST is not idempotent:
- POST usually creates a new resource. Send the same POST twice, and you normally get two new resources (e.g. two orders), not one. So the result changes each time you call it.

### Important status codes

| Code | Meaning | When |
|---|---|---|
| 200 | OK | Successful GET/PUT/PATCH |
| 201 | Created | Successful POST that created a resource |
| 204 | No Content | Successful call with nothing to return (e.g. DELETE) |
| 400 | Bad Request | Invalid input from client |
| 401 | Unauthorized | Missing or invalid authentication |
| 403 | Forbidden | Authenticated but not allowed |
| 404 | Not Found | Resource does not exist |
| 409 | Conflict | Request conflicts with current state (e.g. duplicate entry, version mismatch) |
| 429 | Too Many Requests | Client hit a rate limit |
| 500 | Internal Server Error | Unhandled server-side error |

Quick rule: 4xx means the client made a mistake. 5xx means the server made a mistake.

### Path vs query vs body

- **Path variable**: identifies which resource. Example: `/users/42` — 42 is the path variable.
- **Query parameter**: used for filtering, sorting, searching, paging. Example: `/users?status=active&sort=name`.
- **Body**: carries data for create or update requests. Example: JSON payload in a POST or PUT.

Rule of thumb: path says "which resource," query says "how to narrow or shape the result," body says "what data to send."

### DTOs — why not expose entities directly

A DTO (Data Transfer Object) is a plain object used only to carry data between the client and the API layer. It is not the same as your JPA `@Entity` class.

Reasons to use DTOs instead of returning entities:
- Entities carry lazy-loaded relationships; serializing them directly can trigger extra queries or errors (like `LazyInitializationException`).
- Entities may expose internal fields you don't want the client to see (like a password hash).
- DTO decouples the API contract from the database schema, so the table structure can change without breaking the API, and different endpoints can shape the same data differently (summary vs detail view).

### API versioning approaches

- **URI versioning**: `/api/v1/users`, `/api/v2/users`. Simple, easy to see in logs. Most common choice.
- **Header versioning**: client sends a custom header like `X-API-Version: 2`. Keeps URLs clean but harder to test in a browser.
- **Media type versioning**: `Accept: application/vnd.company.v2+json`. More "correct" REST style, rarely used in practice.

URI versioning is simplest and most widely used; the others trade simplicity for purity.

### Pagination

Sending a huge list in one response is slow and wastes memory. Pagination splits it into pages.

- **Offset-based**: `/orders?page=2&size=20`. Simple, but can skip or repeat rows if data changes between calls, and gets slow for very large offsets.
- **Cursor-based**: `/orders?after=order_1234&size=20`. Uses a pointer (cursor) instead of a page number. Faster and more stable for large or changing datasets.

Spring Data supports `Pageable` and returns a `Page<T>` with content, total count, and page info.

### Statelessness

REST APIs are stateless: the server does not remember anything about the client between requests. Every request must carry all the information needed to process it (like an auth token). This makes scaling easy — any server instance can handle any request, since no session is tied to one server.

### HATEOAS (brief)

HATEOAS = Hypermedia As The Engine Of Application State. The API response includes links to related actions or resources, so the client can navigate without hardcoding URLs.

Example: a response for an order might include a link to "cancel" if the order is still cancellable. In practice, few teams fully implement this — most REST APIs in industry are "RESTful" without full HATEOAS. Know the term and idea; do not expect to need deep implementation detail.

### Idempotency keys for safe retries

Problem: a client sends `POST /payments`. The server processes it, but the network fails before the response reaches the client. The client retries the same POST. Without protection, this creates a duplicate payment.

Fix: the client sends a unique **idempotency key** (e.g. a UUID) in a header, like `Idempotency-Key: abc-123`. The server stores the result against that key. If the same key arrives again, it returns the stored result instead of processing again. Common for payment APIs and any POST that must not be duplicated on retry.

---

## PART B — Microservices Basics

### Monolith vs microservices trade-offs

| | Monolith | Microservices |
|---|---|---|
| Deployment | One unit | Many independent units |
| Scaling | Scale the whole app | Scale only the busy service |
| Development | Simple to start | Complex — needs infra, monitoring |
| Team size | Works well for small teams | Fits larger teams, each owns a service |
| Data | One shared database | Database per service |
| Failure impact | One bug can affect the whole app | Failure can be isolated to one service |
| Testing | Easier end-to-end | Harder — needs contract/integration tests |

Takeaway: microservices solve organizational and scaling problems, but add operational cost. Use them when team size or scaling needs justify that cost, not because it's trendy.

### How services talk

- **Synchronous**: REST (HTTP + JSON) or gRPC (binary, faster, uses protobuf). Caller waits for a response — simple, but creates tight coupling since a down or slow callee affects the caller.
- **Asynchronous / messaging**: Kafka, RabbitMQ. Caller publishes an event and moves on. Decouples services, since the callee can be down temporarily without blocking the caller. Used for events like "order placed" that multiple services react to.

Rule of thumb: use sync calls when you need an immediate answer; use async messaging when you just need to notify other services that something happened.

### Service discovery and API gateway (one line each)

- **Service discovery**: a registry (like Eureka or Consul) that tracks where each service instance is running, so services can find each other without hardcoded IPs.
- **API gateway**: a single entry point (like Spring Cloud Gateway or Zuul) that routes external requests to the right internal service, and can also handle auth, rate limiting, and logging in one place.

### Database-per-service and why

Each microservice has its own database, and no other service can access it directly — only through that service's API.

Why:
- Keeps services independently deployable. Changing one service's schema doesn't break another service.
- Prevents services from becoming coupled through a shared database (a common way "microservices" secretly become a monolith).

Cost: no easy cross-service JOIN. Queries across services need API calls or event-based data replication.

### Distributed transactions and the saga pattern (brief)

Problem: an order flow might update the Order, Payment, and Inventory services together. There is no single database transaction across all three.

**Saga pattern**: break the transaction into a sequence of local transactions, one per service. If one step fails, run **compensating actions** to undo earlier steps. Example: place order → charge payment → reserve inventory. If inventory reservation fails, refund the payment and cancel the order.

Two styles: **choreography** (services react to each other's events, no central controller) and **orchestration** (a central coordinator tells each service what to do next).

### Resilience patterns

- **Timeout**: don't wait forever for a slow dependency; fail fast after a limit.
- **Retry**: automatically try again on a transient failure (with backoff, to avoid overwhelming the dependency).
- **Circuit breaker**: after repeated failures, stop calling the failing service for a while and fail fast instead. After a cooldown, it lets a few test calls through to check recovery. Resilience4j is the common Java library (replacing the older Hystrix).

These three are often used together: timeout limits the wait, retry handles brief blips, circuit breaker protects against sustained outages.

### Config and centralized logging/tracing (brief)

- **Centralized config**: services pull configuration from a shared config server (like Spring Cloud Config) instead of hardcoding properties, so settings can change without redeploying.
- **Centralized logging**: all services send logs to one place (like the ELK stack) so you can search across services in one tool.
- **Distributed tracing**: tools like Zipkin or Jaeger track a request as it flows through multiple services, using a shared trace ID — essential for debugging latency or errors in a microservices call chain.

### The 12-factor idea (brief)

The 12-factor app is a set of practices for building services that run well in the cloud. Key ones: store config in environment variables (not in code); treat backing services (databases, queues) as swappable attached resources; keep processes stateless; log as an event stream (write to stdout, let the platform collect it). You don't need to recite all 12 — just show you know the idea: build services that are portable, stateless, and config-driven.

---

## PART C — SQL Essentials

### Joins

| Join | Returns |
|---|---|
| INNER JOIN | Only rows that match in both tables |
| LEFT JOIN | All rows from left table, matched rows from right (NULL if no match) |
| RIGHT JOIN | All rows from right table, matched rows from left (NULL if no match) |
| FULL JOIN | All rows from both tables, NULL where no match on either side |

Example tables: `employees(id, name, dept_id)` and `departments(id, name)`.

```sql
SELECT e.name, d.name AS dept
FROM employees e
INNER JOIN departments d ON e.dept_id = d.id;
```

This returns only employees who have a valid department. `LEFT JOIN` here would also include employees with no department (dept shown as NULL).

### GROUP BY and HAVING

`GROUP BY` groups rows that share a value, usually to use with aggregate functions like `COUNT`, `SUM`, `AVG`.

```sql
SELECT dept_id, COUNT(*) AS cnt
FROM employees
GROUP BY dept_id
HAVING COUNT(*) > 5;
```

This lists departments with more than 5 employees. `HAVING` filters groups after aggregation; you cannot use `WHERE` for this because `WHERE` runs before grouping and doesn't know the aggregate result yet.

### WHERE vs HAVING

| | WHERE | HAVING |
|---|---|---|
| Filters | Individual rows | Groups (after GROUP BY) |
| Runs | Before grouping | After grouping |
| Can use aggregate functions? | No | Yes |

### Indexes

An index is a separate data structure (usually a B-tree) that lets the database find rows fast, without scanning the whole table — similar to an index at the back of a book.

- Helps: `SELECT` queries with `WHERE`, `JOIN`, `ORDER BY` on the indexed column, especially on large tables.
- Cost: every `INSERT`, `UPDATE`, `DELETE` must also update the index, so too many indexes slow down writes and use extra storage.
- Rule of thumb: index columns used often in `WHERE`/`JOIN`/`ORDER BY`, but don't index every column — balance read speed against write cost.
- A primary key is indexed automatically. Composite indexes (multiple columns) help when queries filter on that exact combination, in that column order.

### ACID

- **Atomicity**: a transaction either fully completes or fully rolls back. No partial changes.
- **Consistency**: a transaction moves the database from one valid state to another, respecting constraints.
- **Isolation**: concurrent transactions don't interfere with each other's intermediate state.
- **Durability**: once committed, data survives even a crash right after.

### Transaction isolation levels

Isolation level controls how much one transaction can see of another transaction that hasn't committed yet. Higher isolation = safer, but more locking and lower concurrency.

| Level | Dirty read | Non-repeatable read | Phantom read |
|---|---|---|---|
| Read Uncommitted | Possible | Possible | Possible |
| Read Committed | Prevented | Possible | Possible |
| Repeatable Read | Prevented | Prevented | Possible (MySQL InnoDB mostly prevents this too) |
| Serializable | Prevented | Prevented | Prevented |

Definitions:
- **Dirty read**: reading another transaction's uncommitted change, which might later be rolled back.
- **Non-repeatable read**: reading the same row twice in one transaction and getting different values, because another transaction updated and committed in between.
- **Phantom read**: re-running the same query twice in one transaction and getting a different set of rows, because another transaction inserted or deleted rows that match the query in between.

Default in most relational databases: **Read Committed** (PostgreSQL, Oracle, SQL Server). MySQL InnoDB defaults to **Repeatable Read**.

### Normalization vs denormalization (brief)

- **Normalization**: organize tables to remove duplicate data, by splitting into related tables (e.g. separate `departments` table instead of repeating department name in every employee row). Reduces update anomalies, saves storage.
- **Denormalization**: intentionally add duplicate data back (e.g. store department name directly in the employee row) to avoid joins and speed up reads.
- Trade-off: normalization is better for write-heavy, consistency-sensitive systems. Denormalization is common in read-heavy systems or reporting tables, where join cost matters more than storage or update complexity.

### Common SQL query questions

**1. Find the second highest salary**

```sql
SELECT MAX(salary) AS second_highest
FROM employees
WHERE salary < (SELECT MAX(salary) FROM employees);
```

Alternative using window function (works for Nth highest too, and handles ties better):

```sql
SELECT salary
FROM (
  SELECT salary, DENSE_RANK() OVER (ORDER BY salary DESC) AS rnk
  FROM employees
) ranked
WHERE rnk = 2;
```

`DENSE_RANK` is preferred when there can be duplicate salary values, since it doesn't skip ranks for ties.

**2. Find duplicate rows (e.g. duplicate emails)**

```sql
SELECT email, COUNT(*) AS cnt
FROM users
GROUP BY email
HAVING COUNT(*) > 1;
```

To delete duplicates and keep one copy (e.g. lowest id):

```sql
DELETE FROM users
WHERE id NOT IN (
  SELECT MIN(id)
  FROM users
  GROUP BY email
);
```

---

## Important Interview Questions

**Q1: Why is POST not idempotent but PUT is?**
PUT replaces a resource with a specific value — repeating it gives the same end state. POST typically creates a new resource each time, so repeating it creates duplicates. Follow-up: "Can POST ever be idempotent?" — Yes, if the server deduplicates using an idempotency key.

**Q2: Why should you not return JPA entities directly from a REST controller?**
Entities can trigger lazy-loading errors, expose internal fields, and tightly couple your API contract to your database schema. DTOs solve all three. Follow-up: "How do you convert between entity and DTO?" — Manually, or with a mapping library like MapStruct.

**Q3: What's the difference between 401 and 403?**
401 means the request has no valid authentication. 403 means the server knows who you are, but you don't have permission. Follow-up: "When would an API return 404 instead of 403?" — To avoid revealing a resource exists to unauthorized users.

**Q4: Explain the saga pattern with a simple example.**
A saga breaks a distributed transaction into local transactions per service, each with a compensating action if a later step fails. Example: order → payment → inventory; if reservation fails, refund payment and cancel order. Follow-up: "Choreography vs orchestration?" — Choreography: services react to events, no central controller. Orchestration: a central coordinator directs each step.

**Q5: What is a circuit breaker and why use it?**
It stops calling a failing dependency after repeated failures, so the caller fails fast instead of waiting. After a cooldown, it lets a few requests through to check recovery. Follow-up: "Difference from retry?" — Retry tries again on failure; circuit breaker stops trying altogether for a while to protect both sides.

**Q6: Why use database-per-service instead of one shared database?**
It keeps services independently deployable — a schema change in one service can't silently break another service reading its tables directly. Follow-up: "How do you JOIN across two services then?" — Through API calls, or replicated data via events (CQRS-style read models).

**Q7: What is the difference between WHERE and HAVING?**
WHERE filters individual rows before grouping; HAVING filters groups after aggregation and can use aggregate functions. Follow-up: "Can you use HAVING without GROUP BY?" — Yes, it treats the whole table as one group, but it's unusual style.

**Q8: Explain isolation levels and the anomaly each one fixes.**
Read Uncommitted allows dirty reads. Read Committed prevents dirty reads but allows non-repeatable reads. Repeatable Read also prevents non-repeatable reads but may allow phantom reads. Serializable prevents all three but has the highest locking cost. Follow-up: "Default in Postgres vs MySQL?" — Postgres: Read Committed. MySQL InnoDB: Repeatable Read.

**Q9: How do you make a POST endpoint safe to retry?**
Use an idempotency key: the client sends a unique key; the server stores the result against it and returns the stored result if the same key is seen again. Follow-up: "Where do you store the key mapping?" — Usually a fast store like Redis, with an expiry.

**Q10: When would you choose gRPC over REST between microservices?**
When you need low latency, high throughput internal calls, and both services are under your control (so you can share the protobuf contract). REST/JSON stays more common for public or external APIs since it's human-readable. Follow-up: "Does gRPC support streaming?" — Yes, unlike typical REST, it supports client, server, and bidirectional streaming.

---

## FAQ / Rapid-Fire

- **Is PATCH idempotent?** Not guaranteed. If it sets a field to a fixed value, yes. If it increments a counter, no.
- **What does 204 mean and when is it used?** No Content — success, but nothing to return in the body. Common for DELETE.
- **Why is REST called "stateless"?** The server stores no session info about the client between requests; each request carries what it needs (like a token).
- **Kafka: sync or async?** Async — the producer doesn't wait for the consumer to process the message.
- **What is the N+1 query problem?** Fetching a list, then running one extra query per item for related data, instead of one query with a join. Common with lazy-loaded JPA relationships.
- **Does an index always help?** No. On small tables, or low-cardinality columns (like a boolean), it may not help and can even be ignored by the query planner.
- **What is the CAP theorem in one line?** During a network partition, a distributed system must choose between Consistency and Availability — not both fully.
- **What's the point of the 12-factor "config in environment" rule?** The same build can move from dev to staging to prod without code changes, just different environment variables.

## Common Traps & Gotchas

- Saying PUT is idempotent because "it doesn't change data" — wrong reason. It's idempotent because repeating it gives the same end state, even though it does change data on the first call.
- Confusing 401 and 403 — 401 is "who are you," 403 is "I know you, but no."
- Returning JPA entities directly from a controller, causing `LazyInitializationException` when the serializer accesses an uninitialized lazy relationship outside the transaction.
- Using offset-based pagination on a huge, frequently-changing table, causing skipped or duplicated rows across pages.
- Treating microservices as "just split the code" — without database-per-service and clear API boundaries, it becomes a "distributed monolith" with all the network cost and none of the independence benefit.
- Assuming Kafka guarantees order across all partitions — it only guarantees order within a single partition.
- Using `WHERE` with an aggregate function like `WHERE COUNT(*) > 5` — invalid; must use `HAVING`.
- Adding an index to every column "just in case" — slows down every write and may not even help reads (e.g. index unused if a function is applied to the column in the query).
- Assuming Repeatable Read fully prevents phantom reads in every database — some engines reduce this risk, but it is not guaranteed by the standard SQL definition.
