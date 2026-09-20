# Chapter 22 — ORMs, JPA & Hibernate Pitfalls

An ORM (Object-Relational Mapping) tool maps Java objects to database rows, so you write less SQL by hand. Interviewers ask about this because ORMs hide real queries, and hidden queries cause real production bugs. You must know what runs under the hood.

## Key Concepts

### What is an ORM, and why use one

An ORM is a library that converts your Java objects into database rows, and rows back into objects. In Java, **JPA** (Java Persistence API) is the standard specification. **Hibernate** is the most common implementation of JPA. Spring Data JPA sits on top of Hibernate and gives you repository interfaces, so you often do not write JPA code directly.

Why teams use an ORM:
- Less boilerplate. You do not write `ResultSet` mapping code by hand.
- You work with Java objects (entities), not raw rows.
- It handles common tasks: generating INSERT/UPDATE/DELETE, caching, dirty checking (see below), and transaction management.
- Database-agnostic SQL generation. The same code can run on PostgreSQL or MySQL with small config changes.

The cost: the ORM writes SQL for you. If you do not understand what SQL it writes, you will ship slow queries without knowing it. This chapter is about that gap.

A simple entity, using the HR schema from earlier chapters:

```java
@Entity
@Table(name = "employees")
public class Employee {
    @Id
    private Long empId;
    private String empName;
    private BigDecimal salary;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "dept_id")
    private Department department;
}
```

### The persistence context (first-level cache)

The **persistence context** is a in-memory area that Hibernate keeps for one `EntityManager` (one unit of work, usually one transaction). When you load an entity, Hibernate stores it here. If you ask for the same entity again in the same context, Hibernate returns the cached object instead of hitting the database again. This is called the **first-level cache**. It is always on, and it is scoped to one session or transaction, not shared across requests.

The persistence context also tracks changes. This is called **dirty checking**: Hibernate compares the current state of an entity to the state it loaded, and if a field changed, it generates an UPDATE automatically when the transaction commits. You do not call `save()` for updates on a managed entity — Hibernate does it for you.

```java
@Transactional
public void giveRaise(Long empId, BigDecimal amount) {
    Employee emp = employeeRepository.findById(empId).orElseThrow();
    emp.setSalary(emp.getSalary().add(amount));
    // no explicit save() call needed — Hibernate detects the change
    // and issues an UPDATE when the transaction commits
}
```

### The N+1 query problem

This is the most common ORM interview question. It happens when you load a list of N parent rows with one query, then the ORM issues one extra query per row to load a related object. Total: 1 + N queries, instead of 1 or 2.

**Example.** You load all employees, then print each one's department name:

```java
List<Employee> employees = employeeRepository.findAll(); // 1 query
for (Employee e : employees) {
    System.out.println(e.getDepartment().getDeptName()); // 1 query PER employee
}
```

If `department` is lazy (loaded only when accessed), Hibernate runs:

```sql
SELECT * FROM employees;                          -- 1 query, returns N rows
SELECT * FROM departments WHERE dept_id = 1;       -- N more queries,
SELECT * FROM departments WHERE dept_id = 2;       -- one per employee
SELECT * FROM departments WHERE dept_id = 1;       -- (repeats if not cached)
...
```

**How to spot it:** turn on SQL logging (`spring.jpa.show-sql=true`, or better, log at DEBUG for `org.hibernate.SQL`). You will see one query, then many near-identical queries with only the WHERE value changing. In production, this shows up as high query count per request in APM tools like New Relic or Datadog.

**How to fix it**, three common ways:

1. **`JOIN FETCH`** in a JPQL query — pulls the related entity in the same query.
```java
@Query("SELECT e FROM Employee e JOIN FETCH e.department")
List<Employee> findAllWithDepartment();
```
This produces one query:
```sql
SELECT e.*, d.* FROM employees e JOIN departments d ON e.dept_id = d.dept_id;
```

2. **`@EntityGraph`** — tells Spring Data JPA which lazy fields to fetch eagerly, for this one query, without writing JPQL.
```java
@EntityGraph(attributePaths = {"department"})
List<Employee> findAll();
```

3. **Batch size** — Hibernate fetches related rows in batches (e.g., `IN (1,2,3,...,20)`) instead of one row at a time. Set it globally or per entity:
```properties
spring.jpa.properties.hibernate.default_batch_fetch_size=20
```
This turns N queries into N/20 queries. It is not as good as a JOIN FETCH (still more than one query), but it is a quick global fix when you cannot change every query.

**Which to pick:** use `JOIN FETCH` when you always need the related data for that query. Use `@EntityGraph` when the same repository method is sometimes used with the association and sometimes without. Use batch size as a safety net for associations you did not think to fix everywhere.

A trap with `JOIN FETCH`: fetching two `List` associations with JOIN FETCH in the same query causes a Cartesian product (rows multiply), and Hibernate throws `MultipleBagFetchException` if both are of type `List`. Fetch one collection eagerly per query, or use a `Set` instead of `List`.

### Lazy vs eager loading, and LazyInitializationException

**Lazy loading** means the related entity or collection is not loaded from the database until you actually access it (call a getter). **Eager loading** means it loads immediately, in the same query or a follow-up query, whether you need it or not.

Defaults in JPA: `@ManyToOne` and `@OneToOne` are `EAGER` by default. `@OneToMany` and `@ManyToMany` are `LAZY` by default. Most teams override `@ManyToOne` to `LAZY` explicitly, because eager loading everywhere causes unexpected N+1 problems and loads data you may never use.

```java
@ManyToOne(fetch = FetchType.LAZY)
@JoinColumn(name = "dept_id")
private Department department;
```

**`LazyInitializationException`** happens when you try to access a lazy field after the persistence context (the session) is already closed. This is very common:

```java
@Transactional
public Employee getEmployee(Long id) {
    return employeeRepository.findById(id).orElseThrow();
} // transaction ends here, session closes

// later, outside any transaction:
Employee emp = service.getEmployee(1L);
emp.getDepartment().getDeptName(); // LazyInitializationException!
```

The fix is one of:
- Access the lazy field while still inside the transaction (fetch what you need before returning).
- Use `JOIN FETCH` or `@EntityGraph` to load it eagerly for that query.
- Map to a DTO (Data Transfer Object) inside the transactional method, and return the DTO instead of the entity.

The DTO approach is usually the cleanest for a web layer, because it also avoids sending unnecessary entity graphs over the wire.

### How `@Transactional` works, and the self-invocation pitfall

`@Transactional` in Spring is implemented using a **proxy**. When you call a method on a Spring bean, you are usually calling the proxy, not the real object. The proxy starts a transaction, calls the real method, then commits or rolls back based on whether an exception was thrown.

```
Caller → Proxy (starts transaction) → Real object.method() → Proxy (commits/rolls back)
```

This proxy step is why **self-invocation does not work**. If a `@Transactional` method calls another `@Transactional` method **on the same class**, the second call goes directly to `this`, not through the proxy. So no new transaction starts, and the annotation on the inner method is silently ignored.

```java
@Service
public class OrderService {

    public void placeOrder(Order order) {
        saveOrder(order); // calls "this.saveOrder()" directly — NOT through the proxy!
    }

    @Transactional
    public void saveOrder(Order order) {
        // this method runs with NO transaction, because it was called
        // from inside the same class, bypassing the Spring proxy
        orderRepository.save(order);
    }
}
```

**Fix:** move `saveOrder` to a different Spring bean and call it through that bean, or inject a self-reference (`@Autowired private OrderService self;` and call `self.saveOrder(order)`), or restructure so the outer method already carries `@Transactional` and covers the whole unit of work.

By default, Spring rolls back a transaction only on unchecked exceptions (`RuntimeException` and its subclasses), not on checked exceptions. Use `@Transactional(rollbackFor = Exception.class)` if you need checked exceptions to roll back too. This is a common interview follow-up.

### When to drop to native SQL

An ORM generates SQL from your entity mappings and query methods. For simple CRUD (create, read, update, delete) and small joins, generated SQL is usually fine. But the ORM can write bad SQL for:
- Complex aggregations, window functions, or reporting queries — JPQL cannot express many SQL features (e.g., `LATERAL` joins, some window function forms).
- Bulk updates/deletes across many rows — JPA loads entities one at a time unless you use a bulk `@Modifying` query.
- Deeply nested fetch graphs, which can trigger Cartesian product blow-ups.
- Cases where you need a specific index hint or query plan the ORM will not produce.

When this happens, use a native query or a handwritten SQL query, still inside the same repository:

```java
@Query(value = """
    SELECT d.dept_name, AVG(e.salary) AS avg_salary
    FROM employees e
    JOIN departments d ON e.dept_id = d.dept_id
    GROUP BY d.dept_name
    HAVING AVG(e.salary) > 80000
    """, nativeQuery = true)
List<Object[]> findHighPayingDepartments();
```

**Rule of thumb for interviews:** say that the ORM is good for the 80% of queries that are simple lookups and small joins, and native SQL is fine, even expected, for the 20% that are reporting, bulk, or performance-critical queries. Say this trade-off out loud — interviewers want to hear that you know when to step outside the ORM, not that you avoid it, or use it blindly everywhere.

### Optimistic locking with `@Version`

**Optimistic locking** assumes conflicts are rare, so it lets both transactions read data without blocking each other, and only checks for a conflict at write (commit) time. This is different from pessimistic locking, which takes a database lock at read time (covered in Chapter 11).

JPA implements optimistic locking with a `@Version` column:

```java
@Entity
public class Employee {
    @Id
    private Long empId;

    @Version
    private Integer version;

    private BigDecimal salary;
}
```

When Hibernate updates the row, it adds the version to the WHERE clause and increments it:

```sql
UPDATE employees
SET salary = 90000, version = 6
WHERE emp_id = 101 AND version = 5;
```

If another transaction already updated this row (so `version` is no longer 5), zero rows match, and Hibernate throws `OptimisticLockException`. Your code must catch this and decide what to do — usually, retry the operation by reloading the row and re-applying the change, or show the user a "someone else updated this" message.

Use optimistic locking when conflicts are rare and you want good throughput (e.g., updating a user's profile). Use pessimistic locking (`SELECT ... FOR UPDATE`) when conflicts are frequent and you need to guarantee no lost updates (e.g., decrementing inventory stock during a flash sale).

## The Questions They Ask

**Q1: What is the N+1 query problem, and how do you fix it?**
It happens when loading a list of N rows triggers one extra query per row to fetch a related entity, giving 1 + N queries instead of 1 or 2. You spot it by looking at SQL logs: one query followed by many similar ones with only the ID changing. Fix it with `JOIN FETCH` in JPQL, `@EntityGraph` on the repository method, or a global batch fetch size, which turns N queries into N/batchSize queries. Follow-up: interviewers often ask you to write the JOIN FETCH query yourself — practice this.

**Q2: Lazy vs eager loading — what is the difference, and what is `LazyInitializationException`?**
Lazy loading defers fetching an association until it is accessed. Eager loading fetches it immediately. `@ManyToOne` and `@OneToOne` default to eager; `@OneToMany` and `@ManyToMany` default to lazy. `LazyInitializationException` happens when you access a lazy field after the session (persistence context) has closed, usually after a `@Transactional` method has returned. Fix by fetching eagerly for that query, accessing the field inside the transaction, or returning a DTO.

**Q3: How does `@Transactional` work internally?**
Spring wraps the bean in a proxy at startup (using JDK dynamic proxies or CGLIB). Calls from outside the bean go through the proxy, which opens a transaction before the method runs and commits or rolls back after. Calls to a `@Transactional` method from another method **in the same class** bypass the proxy (self-invocation), so no transaction management happens. By default, only unchecked exceptions trigger rollback.

**Q4: When would you not use an ORM, or drop to native SQL?**
For complex reporting queries, window functions, bulk updates/deletes, or when the generated SQL is provably slow (checked via `EXPLAIN ANALYZE`, Chapter 7). Also for very high-throughput hot paths where object mapping overhead matters. State the trade-off: ORMs save time on typical CRUD; native SQL gives you control when performance or SQL features matter.

**Q5: What is the persistence context, and how does dirty checking work?**
It is an in-memory area, scoped to one `EntityManager`/transaction, that tracks loaded entities (first-level cache) and their original state. At commit, Hibernate compares current state to the loaded state for each managed entity, and generates UPDATE statements for anything changed. This is why you do not need to call `save()` after modifying a managed entity inside a transaction.

**Q6: How does optimistic locking work with `@Version`, and when would you use pessimistic locking instead?**
A `@Version` column is included in the WHERE clause of every UPDATE and incremented on each write. If the row changed since it was read, zero rows match, and Hibernate throws `OptimisticLockException`. Use it when conflicts are rare and you want high throughput. Use pessimistic locking (`SELECT FOR UPDATE`) when conflicts are frequent, such as inventory decrements, where you cannot afford failed retries under load.

**Q7 (follow-up): Does `@Transactional` on a `private` method work?**
No. Spring's proxy-based AOP (Aspect-Oriented Programming, a way to add behavior like transactions without changing the method body) can only intercept calls to public methods that go through the proxy. `private` methods, and self-invoked calls, are both invisible to the proxy.

## Rapid-Fire

- **What is an ORM?** A library that maps Java objects to database rows and back, so you write less manual SQL.
- **What is JPA vs Hibernate?** JPA is a specification (interfaces and rules). Hibernate is a popular implementation of that specification.
- **What is the persistence context?** A per-transaction, in-memory cache of loaded entities, also called the first-level cache.
- **What causes N+1?** Accessing a lazy association inside a loop over a list that was loaded with one query.
- **Best general fix for N+1 in a list endpoint?** `JOIN FETCH` or `@EntityGraph` for that specific query.
- **Default fetch type for `@ManyToOne`?** EAGER (most teams override it to LAZY).
- **Default fetch type for `@OneToMany`?** LAZY.
- **What throws `LazyInitializationException`?** Accessing a lazy field after the session/transaction has closed.
- **How does Spring implement `@Transactional`?** With a proxy wrapped around the bean.
- **Why does self-invocation break `@Transactional`?** The call goes directly to `this`, skipping the proxy that manages transactions.
- **What exceptions trigger rollback by default?** Unchecked exceptions (`RuntimeException` and subclasses), not checked exceptions.
- **What does `@Version` do?** Adds an optimistic lock check: the UPDATE fails (matches 0 rows) if the version does not match.
- **Optimistic vs pessimistic locking, in one line?** Optimistic checks for conflict at commit time; pessimistic locks the row at read time.
- **When should you write native SQL instead of JPQL?** For complex reporting, window functions, bulk operations, or proven slow generated queries.
- **What is dirty checking?** Hibernate auto-generates UPDATE statements for changed fields on a managed entity, without an explicit save call.

## Common Traps & Mistakes

- **Not noticing N+1 until production.** It is invisible in local testing with a handful of rows, but a 10-row N+1 becomes a 10,000-row disaster in production. Always check SQL logs during development, not just correctness.
- **Fixing N+1 by making the association eager everywhere.** This just moves the problem: now every query that loads the entity pays the join cost, even when it does not need the related data. Fix it per-query with `JOIN FETCH` or `@EntityGraph`, not globally.
- **Fetching two `List` collections eagerly in one JOIN FETCH query.** This causes a Cartesian product or `MultipleBagFetchException`. Fetch one `List` collection per query, or change to `Set`.
- **Returning entities directly from a REST controller, then accessing lazy fields during JSON serialization.** Serialization happens outside the transaction, so this throws `LazyInitializationException`. Use DTOs.
- **Assuming `@Transactional` works when called from the same class.** This is the most common Spring interview trap. Remember: only calls that go through the Spring proxy get transaction behavior.
- **Forgetting that checked exceptions do not roll back by default.** If your code throws a custom checked exception inside a `@Transactional` method, the transaction still commits unless you set `rollbackFor`.
- **Ignoring `OptimisticLockException` in code, letting it bubble up as a raw 500 error.** Catch it and either retry with fresh data or return a clear "conflict, please retry" response to the caller.
- **Treating the ORM as a black box you must never look inside.** In an interview, and in production, you are expected to read the generated SQL (`show-sql`, or `EXPLAIN ANALYZE` from Chapter 7) and judge if it is good. Blind trust in the ORM is a red flag.
- **Using `@OneToMany(fetch = FetchType.EAGER)` on a collection "just to be safe."** This almost always causes N+1 or Cartesian product problems at scale. Keep collections lazy, and fetch explicitly per use case.
