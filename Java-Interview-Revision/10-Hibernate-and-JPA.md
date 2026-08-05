# 10. Hibernate & JPA

ORM tools let you work with database rows as Java objects, instead of writing raw SQL by hand. JPA is the Java standard for this, and Hibernate is the most used implementation of that standard. Interviewers at 3–4 years focus less on annotations and more on how the persistence context works, why N+1 queries happen, and how to fix them.

## Key Concepts (Quick Revision)

- **ORM (Object-Relational Mapping)**: A technique that maps Java classes to database tables, and object fields to table columns. It lets you read and write data using objects, not SQL strings.
- **JPA (Java Persistence API)**: A specification (a set of interfaces and rules). It says *what* an ORM tool must do, not *how*. It does not include actual code that runs.
- **Hibernate**: An implementation of JPA. It contains the real code that talks to the database. You code against JPA interfaces, and Hibernate runs behind them.
- **EntityManager**: The main JPA interface to save, find, update, and delete entities. It comes from JPA.
- **SessionFactory**: Hibernate's own factory object. It creates `Session` objects. One `SessionFactory` per application, expensive to create.
- **Session**: Hibernate's own interface to talk to the database. It does the same job as `EntityManager`, but it is Hibernate-specific and has some extra methods.
- **Persistence context**: A cache that holds entities during one `EntityManager`/`Session` lifetime. Also called the **first-level cache**. It is always on, you cannot turn it off.
- **Entity lifecycle states**: transient, managed (persistent), detached, removed. Explained below.
- **Lazy loading**: Related data is fetched only when you access it, not when the parent entity loads.
- **Eager loading**: Related data is fetched immediately, along with the parent entity.
- **N+1 select problem**: A performance bug where 1 query loads a list of parents, then N more queries load each parent's children one by one.
- **Second-level cache**: An optional cache shared across sessions, and across the whole application.
- **Optimistic locking**: Assumes conflicts are rare. Checks a version number at commit time to detect conflicts.
- **Pessimistic locking**: Assumes conflicts are common. Locks the database row as soon as you read it.

## Important Interview Questions

### 1. What is ORM, and why do we use it?

ORM (Object-Relational Mapping) maps Java objects to database tables. Instead of writing SQL for every operation, you call methods like `save()` or `find()`, and the ORM tool generates the SQL for you.

Benefits:
- Less boilerplate JDBC code (no manual `ResultSet` mapping).
- Database-independent code (switch databases with less pain).
- Built-in caching, dirty checking, and transaction handling.

Trade-off: less control over exact SQL, and it can hide performance problems (like N+1) if used carelessly.

**Follow-up**: *Is ORM always the right choice?* No. For reporting queries, bulk updates, or very complex SQL, plain JDBC or native SQL is often faster and clearer.

### 2. What is the difference between JPA and Hibernate?

| | JPA | Hibernate |
|---|---|---|
| What it is | A specification (interfaces + rules) | An implementation of that specification |
| Contains code? | No, just contracts | Yes, actual working code |
| Vendor | Part of Java EE / Jakarta EE standard | Built by Red Hat (open source) |
| Can you swap it? | Yes, other implementations exist (EclipseLink, OpenJPA) | It is one specific implementation |
| Main interfaces | `EntityManager`, `EntityManagerFactory` | `Session`, `SessionFactory` (Hibernate also implements JPA interfaces) |

Think of JPA as an interface, and Hibernate as a class that implements it. In Spring Boot, `spring-boot-starter-data-jpa` uses Hibernate as the JPA provider by default.

**Follow-up**: *Can you use Hibernate without JPA?* Yes. Hibernate existed before JPA and has its own native API (`Session`, `SessionFactory`, `Criteria`). Most Spring Boot apps use the JPA interfaces (`EntityManager`) for portability, but Spring Data JPA still uses Hibernate underneath.

### 3. What is EntityManager? How is it different from SessionFactory and Session?

`EntityManager` is the JPA interface used to perform CRUD operations, run queries, and manage transactions on entities. `EntityManagerFactory` creates `EntityManager` instances (like a connection pool factory).

Hibernate's own versions:
- `SessionFactory` = Hibernate's `EntityManagerFactory`. One per application. Expensive to build (reads all mappings), so it is built once at startup.
- `Session` = Hibernate's `EntityManager`. One per unit of work (usually one per request/transaction). Cheap to create, not thread-safe.

In Spring Boot, you rarely create these yourself. Spring manages the `EntityManager` for you and injects it, or you use Spring Data JPA repositories which hide it completely.

**Follow-up**: *Is Session thread-safe?* No. Never share one `Session`/`EntityManager` across threads.

### 4. What is the persistence context? How does it give dirty checking and identity?

The **persistence context** is an in-memory cache tied to one `EntityManager`. Every entity you load, save, or update during that `EntityManager`'s life is tracked here. This is also called the **first-level cache**.

Two behaviors come from this:
- **Identity**: If you call `find()` twice for the same ID in the same persistence context, you get back the exact same Java object (same memory reference), not two separate copies. Hibernate checks the cache before hitting the database.
- **Dirty checking**: Hibernate compares the current state of a managed entity to the state it had when loaded. If a field changed, Hibernate knows the entity is "dirty" and generates an `UPDATE` statement automatically at flush time — you do not need to call `save()` again.

```java
@Transactional
public void updateEmail(Long userId, String newEmail) {
    User user = entityManager.find(User.class, userId); // now "managed"
    user.setEmail(newEmail); // no explicit save() call needed
    // Hibernate detects the change and issues UPDATE at flush/commit time
}
```

**Follow-up**: *Does the first-level cache work across two different HTTP requests?* No. It lives only as long as the `EntityManager`/`Session`, which is normally one transaction. A new request gets a new persistence context.

### 5. What are the entity lifecycle states?

| State | Meaning | How you get there |
|---|---|---|
| **Transient** | A plain Java object. Not linked to the database, not tracked by Hibernate. | `new User()` |
| **Managed (Persistent)** | Tracked by the current persistence context. Changes are auto-saved on flush. | `persist()`, `merge()` (returns managed copy), `find()`, result of a query |
| **Detached** | Was managed once, but the persistence context is now closed, or it was explicitly detached. Changes are NOT tracked anymore. | `detach()`, end of transaction, `clear()` |
| **Removed** | Marked for deletion. Will be deleted from the database on flush. | `remove()` |

```
transient --persist()--> managed --remove()--> removed
   ^                        |
   |                    detach() / close session
   |                        v
   +------ merge() <---- detached
```

Key methods:
- `persist(entity)`: makes a transient entity managed. Schedules an `INSERT`. Does not return anything (void). Must not be called on an entity that already has a database row with the same ID, or you get an error.
- `merge(entity)`: takes a detached (or transient) entity, copies its state onto a managed entity (loading it if needed), and **returns the managed copy**. The object you passed in stays detached.
- `find(Class, id)`: loads an entity by ID, returns it as managed. Returns `null` if not found (unlike `getReference()`/JPQL, which can throw).
- `remove(entity)`: entity must be managed first. Marks it removed; deleted on flush.
- `detach(entity)`: removes one entity from the persistence context. It becomes detached; changes to it are no longer tracked.

**Follow-up**: *What happens if you call `setX()` on a detached entity and then the transaction ends?* Nothing gets saved. Detached entities are not tracked, so changes are silently lost unless you call `merge()`.

### 6. Explain entity mapping annotations with an example.

```java
@Entity
@Table(name = "orders")
public class Order {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(name = "order_number", nullable = false, unique = true)
    private String orderNumber;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "customer_id")
    private Customer customer;

    @OneToMany(mappedBy = "order", cascade = CascadeType.ALL, orphanRemoval = true)
    private List<OrderItem> items = new ArrayList<>();
}
```

- `@Entity`: marks the class as a JPA entity (maps to a table).
- `@Table(name=...)`: names the table. Optional; defaults to the class name.
- `@Id`: marks the primary key field.
- `@GeneratedValue`: tells JPA how to generate the ID. Strategies:
    - `IDENTITY`: database auto-increment column generates the value. Simple, but disables JDBC batch inserts in Hibernate (each insert must run immediately to get the ID back).
    - `SEQUENCE`: uses a database sequence. Allows batching, works well with Hibernate's ID pre-allocation.
    - `TABLE`: uses a separate table to simulate a sequence. Portable but slow, rarely used.
    - `AUTO`: Hibernate picks a strategy based on the database.
- `@Column`: customizes the column (name, nullable, length, unique).

**Follow-up**: *Why does IDENTITY hurt batch insert performance?* Because Hibernate needs the generated key immediately for the persistence context, so it cannot delay and batch multiple inserts together. `SEQUENCE` avoids this since Hibernate can pre-fetch IDs.

### 7. Explain the relationship mappings and `mappedBy`.

| Annotation | Meaning | Example |
|---|---|---|
| `@ManyToOne` | Many rows here point to one row there. Owning side, usually has the foreign key. | Many `Order`s → one `Customer` |
| `@OneToMany` | One row here relates to many rows there. Usually the inverse side. | One `Order` → many `OrderItem`s |
| `@OneToOne` | One row here relates to exactly one row there. | One `User` → one `UserProfile` |
| `@ManyToMany` | Many rows on both sides relate to many rows on the other side. Needs a join table. | Many `Student`s ↔ many `Course`s |

`mappedBy` marks the **inverse (non-owning)** side of a bidirectional relationship. It tells Hibernate: "I do not own the foreign key column; look at the field named X on the other entity for that." Only one side should have `mappedBy`; the other side (the owning side) has the actual `@JoinColumn`.

```java
// Owning side (has the foreign key)
@ManyToOne
@JoinColumn(name = "order_id")
private Order order;

// Inverse side (mappedBy points to the field name above)
@OneToMany(mappedBy = "order")
private List<OrderItem> items;
```

For `@ManyToMany`, a join table is needed since neither side can hold a single foreign key:

```java
@ManyToMany
@JoinTable(
    name = "student_course",
    joinColumns = @JoinColumn(name = "student_id"),
    inverseJoinColumns = @JoinColumn(name = "course_id")
)
private List<Course> courses;
```

**Follow-up**: *What happens if you forget `mappedBy` and put `@JoinColumn` on both sides?* Hibernate treats both sides as owning. It may create an extra join table or duplicate columns, and updates from one side can silently overwrite the other. Always pick one owning side.

### 8. Lazy vs eager loading, and LazyInitializationException.

- **Eager (`FetchType.EAGER`)**: related data loads immediately, in the same query or a follow-up query, as soon as the parent loads.
- **Lazy (`FetchType.LAZY`)**: related data is not loaded until you actually call a getter on it. Hibernate returns a proxy object instead of the real one.

Defaults: `@ManyToOne` and `@OneToOne` are **EAGER** by default. `@OneToMany` and `@ManyToMany` are **LAZY** by default. In practice, most teams override `@ManyToOne`/`@OneToOne` to LAZY too, since EAGER can silently pull in a lot of unrelated data.

**LazyInitializationException**: thrown when you try to access a lazy field/collection *after* the persistence context (session) is already closed. Common case: load an entity inside a `@Transactional` service method, return it to a controller, and try to access a lazy list in the view layer — the transaction (and session) is already closed by then.

```java
Order order = orderRepository.findById(1L).get(); // session closes when method returns
order.getItems().size(); // BOOM: LazyInitializationException, if accessed outside transaction
```

Fixes:
- Access lazy fields while the session is still open (inside the `@Transactional` method).
- Use `JOIN FETCH` or `@EntityGraph` to load what you need up front.
- Use DTO projections instead of returning entities outside the service layer.
- (Avoid `Open Session In View` as a fix — it hides the problem and can cause its own performance issues.)

**Follow-up**: *Why not just make everything EAGER to avoid this exception?* Because EAGER loading pulls extra data on every query, even when you do not need it. It can cause the N+1 problem too, and it removes your control over what gets fetched when.

### 9. Explain the N+1 select problem with an example, and how to fix it.

The N+1 problem: you run **1** query to get a list of N parent rows. Then, for **each** parent, Hibernate runs **1 more** query to lazily load its related child data. Total queries = 1 + N, instead of 1.

Example:

```java
List<Order> orders = orderRepository.findAll(); // 1 query: SELECT * FROM orders

for (Order order : orders) {
    System.out.println(order.getCustomer().getName()); // lazy @ManyToOne
    // 1 query PER order: SELECT * FROM customer WHERE id = ?
}
```

If there are 100 orders, this runs 1 query to fetch orders, then 100 more queries to fetch each customer — 101 queries total. This is slow mainly because of network round-trips to the database, not the query complexity itself.

**Fixes**:

1. **JOIN FETCH** (JPQL) — pulls the related entity in the same query:
```java
@Query("SELECT o FROM Order o JOIN FETCH o.customer WHERE o.status = :status")
List<Order> findByStatusWithCustomer(@Param("status") String status);
```

2. **@EntityGraph** — tells Spring Data JPA which associations to fetch eagerly, for a specific query, without writing JPQL:
```java
@EntityGraph(attributePaths = {"customer", "items"})
List<Order> findByStatus(String status);
```

3. **Batch fetching** (`@BatchSize` or `hibernate.default_batch_fetch_size`) — instead of 1 query per parent, Hibernate loads related rows for a batch of parents in one `WHERE id IN (...)` query. Reduces N queries to N/batchSize queries. Good when JOIN FETCH is not practical (e.g., multiple lazy collections on the same entity, which JOIN FETCH cannot combine well).
```java
@OneToMany(mappedBy = "order")
@BatchSize(size = 20)
private List<OrderItem> items;
```

4. **DTO projection** — select only the fields you need with a JPQL constructor expression, avoiding entity loading and its lazy fields entirely.

**Follow-up**: *Can JOIN FETCH cause problems?* Yes — fetching multiple `@OneToMany` collections with JOIN FETCH in one query causes a "cartesian product" (duplicate rows), which can be worse than N+1. Fetch at most one collection per query with JOIN FETCH; use batch size for the rest.

**Follow-up**: *How do you detect N+1 in practice?* Enable SQL logging (`show-sql`, or better, a tool like p6spy / Hibernate statistics) and count queries per request. Many teams also add a test that asserts the number of SQL statements for a given operation.

### 10. What are cascade types and orphanRemoval?

**Cascade** controls whether an operation on the parent entity also applies to its related (child) entities.

| Cascade type | Effect |
|---|---|
| `PERSIST` | Saving the parent also saves new children |
| `MERGE` | Merging the parent also merges children |
| `REMOVE` | Deleting the parent also deletes children |
| `REFRESH` | Refreshing the parent also refreshes children |
| `DETACH` | Detaching the parent also detaches children |
| `ALL` | All of the above |

```java
@OneToMany(mappedBy = "order", cascade = CascadeType.ALL, orphanRemoval = true)
private List<OrderItem> items = new ArrayList<>();
```

**`orphanRemoval = true`**: if a child is removed from the parent's collection (e.g., `order.getItems().remove(item)`), Hibernate deletes that child row from the database automatically — even without calling `remove()` on it directly. This only makes sense when the child cannot exist without the parent (a true "owned" relationship).

**Follow-up**: *Difference between `CascadeType.REMOVE` and `orphanRemoval`?* `REMOVE` cascades only when you delete the *parent itself*. `orphanRemoval` also triggers when a child is simply taken *out of the collection*, even if the parent still exists.

### 11. What is the second-level cache and query cache?

- **Second-level cache (L2 cache)**: An optional cache that lives at the `SessionFactory` level (not per-session). It is shared across all sessions, and can even be shared across the whole application (or cluster, with a distributed cache provider). Stores entity data by ID. Needs a cache provider like Ehcache or Redis (Hibernate does not ship a full cache implementation itself).
- **Query cache**: Caches the *result of a query* (a list of entity IDs), not full entity data. Needs L2 cache enabled too, since it looks up the actual entity data there.

When to use:
- Good for read-heavy, rarely-changing reference data (e.g., country list, product categories).
- Risky for frequently updated data — stale cache can cause wrong reads, and cache invalidation adds complexity.
- Not a default-on feature; must be explicitly configured and enabled per-entity (`@Cacheable`).

**Follow-up**: *Why is first-level cache always on, but second-level cache optional?* First-level cache is required for Hibernate's core correctness guarantees (identity, dirty checking) within one transaction. Second-level cache is purely a performance optimization across transactions, and it introduces staleness risk, so it is opt-in.

### 12. JPQL vs Criteria API vs native queries.

| | JPQL | Criteria API | Native Query |
|---|---|---|---|
| What it is | Object-oriented query language, similar to SQL but uses entity/field names | Build queries with Java method calls (type-safe) | Plain SQL |
| Type safety | No (string-based) | Yes (compile-time checked with metamodel) | No |
| Readability | Good, close to SQL | Verbose, harder to read | Best for complex/DB-specific SQL |
| Dynamic queries | Hard to build dynamically | Best fit for dynamic filters | Possible but messy |
| Database portability | Portable | Portable | Not portable (DB-specific SQL) |

```java
// JPQL
@Query("SELECT o FROM Order o WHERE o.status = :status")
List<Order> findByStatus(@Param("status") String status);

// Native
@Query(value = "SELECT * FROM orders WHERE status = :status", nativeQuery = true)
List<Order> findByStatusNative(@Param("status") String status);
```

**Follow-up**: *When would you use native SQL over JPQL?* When you need database-specific features (window functions, CTEs, hints), or very complex reporting queries where JPQL/Criteria become unreadable.

### 13. Optimistic locking vs pessimistic locking.

- **Optimistic locking**: Assumes conflicts are rare. Add a `@Version` field. Hibernate checks this version number when updating. If another transaction changed the row (and bumped the version) since you read it, your update fails with `OptimisticLockException`.

```java
@Entity
public class Account {
    @Id
    private Long id;

    private BigDecimal balance;

    @Version
    private Long version; // Hibernate manages this automatically
}
```

How it works: `UPDATE account SET balance=?, version=version+1 WHERE id=? AND version=?`. If zero rows are updated (because version no longer matches), Hibernate throws `OptimisticLockException`.

- **Pessimistic locking**: Assumes conflicts are common. Locks the row in the database as soon as you read it (`SELECT ... FOR UPDATE`), blocking other transactions from modifying (or sometimes even reading) it until you commit.

```java
Account account = entityManager.find(Account.class, id, LockModeType.PESSIMISTIC_WRITE);
```

| | Optimistic | Pessimistic |
|---|---|---|
| Locking cost | None until commit | DB row lock held for transaction duration |
| Best for | Low-conflict, high-concurrency scenarios | High-conflict scenarios, short transactions |
| Failure mode | Exception at commit, need retry logic | Threads wait or fail immediately, risk of deadlock |

**Follow-up**: *What do you do when `OptimisticLockException` happens?* Usually retry the whole transaction from scratch, re-reading the fresh data. It is the application's job to handle the retry, Hibernate just detects the conflict.

### 14. `save()` vs `saveAndFlush()`. How does flush relate to transaction commit?

- **`save()`** (Spring Data JPA): persists (or merges) the entity in the persistence context. It does **not** guarantee the SQL runs immediately — Hibernate may delay the actual `INSERT`/`UPDATE` until flush time.
- **`saveAndFlush()`**: does the same, but also forces an immediate flush — the SQL is sent to the database right away, within the current transaction.

**Flush**: the act of syncing the persistence context's in-memory changes to the database, by sending SQL statements. Flush does **not** mean commit — the changes are sent as SQL, but they are still inside the current transaction and can be rolled back.

**Commit**: ends the transaction. On commit, Hibernate flushes first (if not already flushed), then tells the database to make the changes permanent (`COMMIT`).

Flush happens automatically:
- Right before a query runs (so the query sees pending changes) — this is the default `FlushModeType.AUTO`.
- At transaction commit time.
- When `flush()` is called manually.

**Follow-up**: *Why would you ever call `saveAndFlush()` manually?* When you need the generated ID or a database-level error (e.g., a constraint violation) immediately, before continuing more logic in the same method, instead of finding out only at commit time.

### 15. Common performance tips for Hibernate/JPA.

- Default `@ManyToOne`/`@OneToOne` to `LAZY`, and fetch what you need with `JOIN FETCH` or `@EntityGraph`.
- Watch for N+1: enable SQL logging in dev, or use a query-counting test.
- Use `SEQUENCE` instead of `IDENTITY` if you need batch inserts.
- Use DTO projections for read-heavy endpoints instead of loading full entities.
- Use pagination (`Pageable`) for large result sets; never load an entire table into memory.
- Set `hibernate.jdbc.batch_size` for bulk inserts/updates.
- Avoid `CascadeType.ALL` and `orphanRemoval` on large collections if not truly needed — accidental deletes can cascade further than expected.
- Keep transactions short — long transactions hold locks and connections longer than needed.

## FAQ / Rapid-Fire

- **Is `EntityManager` thread-safe?** No. Use one per unit of work.
- **Does `persist()` return the entity?** No, it returns `void`. Use `merge()` if you need a managed reference back.
- **Is the first-level cache shared across users/requests?** No, it is per `EntityManager`, usually per transaction.
- **Default fetch type for `@OneToMany`?** LAZY.
- **Default fetch type for `@ManyToOne`?** EAGER (often manually overridden to LAZY).
- **What triggers `LazyInitializationException`?** Accessing a lazy field/collection after the session is closed.
- **What causes the N+1 problem?** Lazy associations accessed in a loop, one query per iteration.
- **Fastest general fix for N+1?** `JOIN FETCH` for a single collection, `@BatchSize` for multiple.
- **What does `@Version` do?** Enables optimistic locking; Hibernate auto-increments it and checks it on update.
- **Does flush mean the transaction is committed?** No. Flush sends SQL; commit makes it permanent. Flush can still be rolled back.
- **Which is the owning side in a bidirectional `@OneToMany`/`@ManyToOne`?** The `@ManyToOne` side (it holds the foreign key).
- **What is `mappedBy` for?** Marks the inverse (non-owning) side of a relationship.
- **Does `orphanRemoval` need `CascadeType.REMOVE` too?** No, `orphanRemoval = true` works on its own for removal-from-collection cases.
- **JPQL works on tables or entities?** Entities and their fields, not raw table/column names.
- **Is second-level cache on by default?** No, it must be configured and enabled explicitly.

## Common Traps & Gotchas

- **Assuming `save()` always hits the database immediately.** It may just update the persistence context; the real SQL can wait until flush.
- **Returning entities with lazy fields straight from a controller.** Once outside the transaction, lazy access throws `LazyInitializationException`. Use DTOs.
- **Fixing N+1 by making everything EAGER.** This just moves the problem — you now always fetch extra data, even when not needed, and can still get N+1-like behavior with EAGER collections.
- **Using JOIN FETCH on two `@OneToMany` collections in one query.** Causes a cartesian product (duplicate, bloated rows). Fetch one collection at a time, or use `@BatchSize`.
- **Forgetting `mappedBy`, and putting `@JoinColumn` on both sides of a relationship.** Leads to confusing extra columns/tables and conflicting updates.
- **Calling `persist()` on an entity that already exists in the database.** Throws an error. Use `merge()` for updating detached entities instead.
- **Modifying a detached entity and expecting it to save.** Changes to detached entities are not tracked. You must call `merge()` first.
- **Relying on second-level cache for frequently updated data.** Leads to stale reads. Reserve it for slow-changing reference data.
- **Ignoring `OptimisticLockException` instead of retrying.** The failed transaction's data is lost unless you catch the exception and retry with fresh data.
- **Using `IDENTITY` generation and expecting JDBC batch inserts to work.** Hibernate cannot batch inserts efficiently with `IDENTITY`, since it needs each generated key right away.
- **Thinking `@Transactional` alone prevents N+1 or lazy exceptions.** It only keeps the session open longer; it does not fetch anything for you. You still need `JOIN FETCH`/`@EntityGraph` for real fixes.
