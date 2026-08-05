# 9. Spring & Spring Boot

Spring is a framework that manages your objects and their dependencies for you. Spring Boot sits on top of Spring and removes most manual setup. This is the most asked topic at 3–4 years experience, so interviewers go deep here — expect "how does it work internally" style questions, not just definitions.

## Key Concepts (Quick Revision)

**IoC (Inversion of Control) and the Spring container**
- Normally, your code creates its own objects (`new Service()`). This is called normal control.
- IoC means you give up this control. The framework creates objects and gives them to you.
- The **Spring container** is the part of Spring that does this. It reads your configuration (annotations or XML), creates objects (called **beans**), and manages their full life.
- **Dependency Injection (DI)** is the technique the container uses to do IoC. Instead of a class creating the objects it needs, the container "injects" (passes in) those objects.

**Types of Dependency Injection**
| Type | How | Notes |
|---|---|---|
| Constructor injection | Dependency passed in the constructor | Preferred. Object is always complete. Works well with `final` fields. Easy to test. |
| Setter injection | Dependency set via a setter method | Used for optional dependencies. Object can exist without it for some time. |
| Field injection | `@Autowired` directly on the field | Shortest to write, but hard to test and hides dependencies. Avoid in real projects. |

Why constructor injection is preferred:
- Fields can be `final` — object is immutable once built.
- You cannot create the object with missing dependencies (fails fast, at startup, not later).
- No reflection needed to set fields, so it plays well with plain unit tests (no Spring container needed — just call `new Service(mockDep)`).
- Circular dependency between two beans shows up as a startup error immediately, instead of a silent bug.

**ApplicationContext vs BeanFactory**
- `BeanFactory` is the basic container. It creates beans **lazily** — only when you ask for them (`getBean()`).
- `ApplicationContext` extends `BeanFactory`. It adds: eager loading of singleton beans at startup, event handling, internationalization support, and easy integration with AOP.
- In real projects, you always use `ApplicationContext` (or Spring Boot's auto-configured one). `BeanFactory` is mostly of historical/interview interest.

**Bean lifecycle** (order matters — common interview question)
1. **Instantiate** — container creates the object (calls constructor).
2. **Populate properties** — dependencies are injected (setters/fields).
3. **Aware callbacks** — if the bean implements `BeanNameAware`, `ApplicationContextAware`, etc., those methods are called.
4. **BeanPostProcessor (before init)** — any registered post-processors run.
5. **Init** — `@PostConstruct` method runs, then `InitializingBean.afterPropertiesSet()`, then any custom `init-method`.
6. **BeanPostProcessor (after init)** — post-processors run again (this is where proxies, like AOP or `@Transactional` proxies, are usually created).
7. Bean is ready to use.
8. **Destroy** — on container shutdown: `@PreDestroy` method runs, then `DisposableBean.destroy()`, then custom `destroy-method`. (Only for singleton beans — container does not manage destruction of prototype beans.)

```java
@Component
public class CacheLoader {

    @PostConstruct
    public void loadCache() {
        // runs once, right after dependencies are injected
    }

    @PreDestroy
    public void clearCache() {
        // runs once, right before the bean is destroyed
    }
}
```

**Bean scopes**
| Scope | Meaning | Default? |
|---|---|---|
| `singleton` | One instance per Spring container, shared everywhere | Yes (default) |
| `prototype` | New instance every time the bean is requested | No |
| `request` | One instance per HTTP request (web apps only) | No |
| `session` | One instance per HTTP session (web apps only) | No |

- **Singleton is the default.** This means one shared instance is used by the whole application. So a singleton bean must **not hold mutable state per-user/per-request** in instance fields — that causes bugs when many threads use it at once.
- Set scope with `@Scope("prototype")` on the bean.

**Core stereotype annotations**
- `@Component` — generic Spring-managed bean. Base annotation.
- `@Service` — same as `@Component`, but marks a service/business-logic class. Purely for readability (no extra behavior by default).
- `@Repository` — same as `@Component`, but also converts database exceptions into Spring's unchecked `DataAccessException` family. This is real extra behavior, not just naming.
- `@Controller` — marks a web controller (returns view names in traditional MVC).
- `@RestController` — combination of `@Controller` + `@ResponseBody`. Every method's return value is written directly to the HTTP response body (usually as JSON), instead of being resolved as a view name.

**Wiring annotations**
- `@Autowired` — tells Spring to inject a dependency automatically.
- `@Qualifier("beanName")` — used with `@Autowired` when more than one bean of the same type exists, to pick a specific one by name.
- `@Primary` — marks one bean as the default choice when multiple beans of the same type exist (used instead of `@Qualifier` everywhere it's injected).

**How `@Autowired` resolves a bean**
1. First, Spring looks **by type**. If exactly one bean of that type exists, it is injected.
2. If **more than one** bean of that type exists, Spring looks **by name** — it matches the field/parameter name (or `@Qualifier` value) to a bean name.
3. If it still cannot resolve to exactly one bean, Spring throws `NoUniqueBeanDefinitionException` at startup.
4. If no bean of that type exists, Spring throws `NoSuchBeanDefinitionException` (unless `required = false`).

**Circular dependency** — Bean A needs Bean B, and Bean B needs Bean A.
- With **constructor injection**, this fails at startup with `BeanCurrentlyInCreationException`, because Spring cannot fully build A without B, and cannot fully build B without A.
- With **setter/field injection**, Spring can sometimes solve it: it creates a raw (not-fully-initialized) instance of A, injects it into B, finishes B, then finishes A. This works only for singleton scope, using an internal "early bean reference" cache.
- Best practice: **redesign the code** so the circular dependency does not exist (e.g., extract common logic into a third bean). Relying on setter injection to "solve" it is a workaround, not a fix.

**`@Configuration` + `@Bean` vs component scanning**
- Component scanning (`@Component`, `@Service`, etc.) — Spring scans your packages and auto-registers annotated classes as beans. Best for classes you own and write.
- `@Configuration` + `@Bean` — you write a Java method that manually builds and returns an object. Best for:
    - Third-party classes you cannot annotate (e.g., a library's `RestTemplate`, `ObjectMapper`).
    - Beans that need custom construction logic or conditional creation.
```java
@Configuration
public class AppConfig {

    @Bean
    public RestTemplate restTemplate(RestTemplateBuilder builder) {
        return builder.setConnectTimeout(Duration.ofSeconds(5)).build();
    }
}
```

**Spring MVC / REST — request flow**
1. Request hits **`DispatcherServlet`** — the single front controller for all requests.
2. `DispatcherServlet` asks a **`HandlerMapping`** which controller method matches the URL.
3. It calls the controller method (via a `HandlerAdapter`).
4. Controller method runs and returns a result.
5. For `@RestController`, an `HttpMessageConverter` (like Jackson) converts the return value to JSON and writes it to the response directly.
6. For `@Controller`, a `ViewResolver` turns the returned view name into an actual view (e.g., JSP/Thymeleaf), which is rendered.

**Common REST annotations**
| Annotation | Purpose |
|---|---|
| `@RequestMapping` | Base mapping; can set path, HTTP method, headers |
| `@GetMapping`, `@PostMapping`, etc. | Shortcut for `@RequestMapping(method = ...)` |
| `@PathVariable` | Reads a value from the URL path, e.g. `/users/{id}` |
| `@RequestParam` | Reads a query parameter, e.g. `?page=2` |
| `@RequestBody` | Converts the JSON request body into a Java object |
| `ResponseEntity<T>` | Lets you control status code, headers, and body together |

```java
@RestController
@RequestMapping("/api/users")
public class UserController {

    private final UserService userService;

    public UserController(UserService userService) { // constructor injection
        this.userService = userService;
    }

    @GetMapping("/{id}")
    public ResponseEntity<UserDto> getUser(@PathVariable Long id) {
        UserDto user = userService.findById(id);
        return ResponseEntity.ok(user);
    }

    @PostMapping
    public ResponseEntity<UserDto> create(@Valid @RequestBody UserDto request) {
        UserDto created = userService.create(request);
        return ResponseEntity.status(HttpStatus.CREATED).body(created);
    }
}
```

**Exception handling — `@ControllerAdvice` + `@ExceptionHandler`**
- `@ControllerAdvice` marks a class as a global handler for exceptions across all controllers.
- `@ExceptionHandler` inside it maps a specific exception type to a response.

```java
@ControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(UserNotFoundException.class)
    public ResponseEntity<ErrorResponse> handleNotFound(UserNotFoundException ex) {
        return ResponseEntity.status(HttpStatus.NOT_FOUND)
                .body(new ErrorResponse(ex.getMessage()));
    }

    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ResponseEntity<ErrorResponse> handleValidation(MethodArgumentNotValidException ex) {
        return ResponseEntity.badRequest().body(new ErrorResponse("Validation failed"));
    }
}
```

**Bean validation — `@Valid`**
- Put `@Valid` on a `@RequestBody` parameter to tell Spring: "check this object's constraints before running the method."
- Constraints come from annotations on the DTO fields, e.g. `@NotNull`, `@NotBlank`, `@Size(min=1, max=50)`, `@Email`.
- If validation fails, Spring throws `MethodArgumentNotValidException` (handle it in `@ControllerAdvice`, as shown above).

**Spring Boot — auto-configuration**
- Plain Spring needs a lot of manual bean setup (data source, view resolver, JSON converter, etc.). **Spring Boot removes most of this** by guessing sensible defaults based on what is on the classpath.
- `@EnableAutoConfiguration` (bundled inside `@SpringBootApplication`) turns this on. It scans a list of auto-configuration classes and applies the ones whose conditions match.
- Each auto-configuration class is guarded by conditional annotations like `@ConditionalOnClass` (apply only if a class is on the classpath), `@ConditionalOnMissingBean` (apply only if you haven't defined your own bean of that type), `@ConditionalOnProperty` (apply only if a property is set).
- Example: if `spring-boot-starter-web` and Tomcat are on the classpath, and you haven't defined your own embedded server bean, Spring Boot auto-configures an embedded Tomcat server for you.
- **Starters** (`spring-boot-starter-web`, `spring-boot-starter-data-jpa`, etc.) are just curated dependency bundles — they pull in the libraries and versions that work well together, so you don't manage each version yourself.

**Configuration and profiles**
- `application.properties` / `application.yml` — external configuration file, kept outside the code, read at startup.
- `@Value("${some.property}")` — inject a single property value into a field.
- `@ConfigurationProperties(prefix = "app")` — bind a whole group of properties into one typed object (better than many `@Value` fields for related settings).
```java
@ConfigurationProperties(prefix = "app.mail")
public class MailProperties {
    private String host;
    private int port;
    // getters and setters
}
```
- **Profiles** (`@Profile("dev")`, `application-dev.yml`) — let you keep different configuration for different environments (dev, test, prod) and activate one with `spring.profiles.active=dev`.

**Embedded server and Actuator**
- Spring Boot apps run with an **embedded server** (Tomcat by default) packed inside the JAR. You run the app with `java -jar app.jar` — no separate server install needed.
- **Actuator** (`spring-boot-starter-actuator`) adds production-ready endpoints out of the box: `/actuator/health`, `/actuator/metrics`, `/actuator/info`, etc. Used for monitoring and health checks in production.

**`@Transactional` in depth**
- A **transaction** groups several database operations so that either all of them succeed, or none of them do (atomicity).
- `@Transactional` works through a **proxy**. Spring wraps your bean in a proxy object at startup. When you call the annotated method **from outside the bean** (e.g., from a controller calling a service), the call goes through the proxy first. The proxy starts a transaction, calls your real method, then commits or rolls back based on the result.
- **The classic self-invocation pitfall**: if a method inside the same class calls another `@Transactional` method (`this.otherMethod()`), the call does **not** go through the proxy — it's a plain Java method call. So the transaction annotation on `otherMethod()` is silently ignored.
```java
@Service
public class OrderService {

    public void placeOrder(Order order) {
        // this call bypasses the proxy — saveOrder()'s @Transactional has NO effect here
        this.saveOrder(order);
    }

    @Transactional
    public void saveOrder(Order order) {
        // ...
    }
}
```
- Fix: call the transactional method from a different bean, or inject a self-reference proxy, or restructure the code.

- **Propagation** — decides how a transactional method behaves when called inside an existing transaction.
  | Propagation | Behavior |
  |---|---|
  | `REQUIRED` (default) | Join the existing transaction if one exists; otherwise start a new one |
  | `REQUIRES_NEW` | Always start a new, independent transaction; suspend the existing one |
  | `SUPPORTS` | Join if a transaction exists; otherwise run without one |
  | `MANDATORY` | Must run inside an existing transaction; throws an exception if none exists |
  | `NOT_SUPPORTED` | Runs without a transaction; suspends the existing one if present |
  | `NEVER` | Must not run inside a transaction; throws an exception if one exists |
  | `NESTED` | Runs as a nested transaction (savepoint) inside the existing one, if the driver supports it |

- **Isolation** — controls how much one transaction can see of another transaction's uncommitted changes (`READ_COMMITTED`, `REPEATABLE_READ`, `SERIALIZABLE`, etc.). Default is usually the database's own default.
- **`readOnly = true`** — hints to the database/ORM that this transaction only reads data. Allows some performance optimizations (e.g., Hibernate skips dirty-checking). It does not enforce read-only at the SQL level by itself.
- **Rollback rules** — by default, Spring rolls back only on **unchecked exceptions** (`RuntimeException` and its subtypes), not on checked exceptions. To roll back on a checked exception too, use `@Transactional(rollbackFor = SomeCheckedException.class)`.

```java
@Service
public class PaymentService {

    private final PaymentRepository paymentRepository;

    public PaymentService(PaymentRepository paymentRepository) {
        this.paymentRepository = paymentRepository;
    }

    @Transactional(propagation = Propagation.REQUIRED, rollbackFor = PaymentException.class)
    public void processPayment(Payment payment) throws PaymentException {
        paymentRepository.save(payment);
        // if a RuntimeException or PaymentException is thrown here,
        // the save above is rolled back automatically
    }
}
```

**Spring AOP (Aspect-Oriented Programming)**
- AOP lets you add cross-cutting logic (logging, security checks, transactions) without writing that code inside every business method.
- Key terms:
    - **Aspect** — the module that holds the cross-cutting logic (e.g., a `LoggingAspect` class).
    - **Join point** — a point in program execution where the aspect can plug in (e.g., a method call).
    - **Advice** — the actual action taken at a join point (`@Before`, `@After`, `@Around`, `@AfterReturning`, `@AfterThrowing`).
    - **Pointcut** — an expression that defines **which** join points the advice applies to (e.g., "all methods in `com.app.service` package").
- Spring AOP is **proxy-based**. It creates a proxy around your bean and adds the aspect logic before/after/around the real method call. This is why `@Transactional` and `@Async` also rely on the same proxy mechanism, and why self-invocation breaks them too.

**Spring Security — short note**
- **Authentication** — checking *who* the user is (e.g., verifying username/password, validating a JWT token).
- **Authorization** — checking *what* the authenticated user is allowed to do (e.g., only `ADMIN` role can delete a user).
- Spring Security works using a **filter chain** — a series of servlet filters that a request passes through before reaching your controller. Each filter does one job (e.g., one filter checks the JWT, another checks CSRF, another checks the session).
- If any filter rejects the request (e.g., invalid token), the request is stopped early with a 401/403 response, and it never reaches your controller.

**Spring Data JPA repositories**
- `JpaRepository<Entity, IdType>` — an interface you extend. Spring generates the implementation at runtime. You get `save()`, `findById()`, `findAll()`, `delete()`, etc. for free, with no code written.
- **Derived query methods** — Spring can build a query just from the method name.
```java
public interface UserRepository extends JpaRepository<User, Long> {
    List<User> findByEmail(String email);
    List<User> findByAgeGreaterThanAndStatus(int age, String status);
}
```
Spring reads the method name (`findBy...And...GreaterThan...`) and builds the SQL/JPQL query automatically — you don't write it.

## Important Interview Questions

**1. What is the difference between `@Component`, `@Service`, and `@Repository`?**
All three make Spring register the class as a bean. `@Service` is just a more specific name for a business/service class — no extra behavior. `@Repository` adds real behavior: it translates database-specific exceptions into Spring's own unchecked `DataAccessException` hierarchy, so your service layer doesn't need to know which database driver threw the error.
*Follow-up: Can you use `@Component` on a DAO class instead of `@Repository`?* Yes, it will still work as a bean, but you lose the automatic exception translation.

**2. Why is constructor injection preferred over field injection?**
It makes dependencies explicit and required — the object cannot be built without them. It allows `final` fields (immutability). It makes unit testing easy, since you can build the object with `new` and pass mocks, without starting Spring at all. Field injection needs reflection or a running Spring context to set the field, which makes plain unit tests harder.
*Follow-up: When would you still use setter injection?* For optional dependencies that the bean can work without, or to allow re-configuring a value after construction.

**3. Explain the Spring bean lifecycle.**
Instantiate → populate properties (DI) → Aware callbacks → `BeanPostProcessor` (before init) → `@PostConstruct` / `afterPropertiesSet()` / custom init → `BeanPostProcessor` (after init) → bean ready → (on shutdown) `@PreDestroy` / `destroy()` / custom destroy.
*Follow-up: Where are AOP proxies created in this flow?* Usually in the `BeanPostProcessor` (after init) step.

**4. What happens if two beans of the same type exist and you use plain `@Autowired`?**
Spring first tries to match by type. If more than one candidate matches, it tries to match by field/parameter name to a bean name. If that still doesn't resolve to one bean, it throws `NoUniqueBeanDefinitionException`. You fix this with `@Qualifier("beanName")` or by marking one bean `@Primary`.

**5. How does Spring handle circular dependencies?**
With field/setter injection on singleton beans, Spring can resolve it using an internal cache of early (partially built) bean references. With constructor injection, it cannot — it fails at startup with `BeanCurrentlyInCreationException`. The real fix is to redesign the classes so neither depends directly on the other (e.g., pull shared logic into a third class).

**6. Explain `@Transactional` propagation `REQUIRED` vs `REQUIRES_NEW`.**
`REQUIRED` (default) joins an existing transaction if the caller already has one open; otherwise it starts a new one. `REQUIRES_NEW` always suspends any existing transaction and starts a completely new, independent one — so if the new one fails and rolls back, it does **not** roll back the outer transaction.
*Follow-up: Give a real use case for `REQUIRES_NEW`.* Writing an audit log entry that must be saved even if the main business transaction later fails and rolls back.

**7. Why doesn't `@Transactional` work when called from another method in the same class?**
Because `@Transactional` is implemented using a **proxy**. Spring wraps the bean in a proxy at startup, and only calls that come **through the proxy** (i.e., from outside the class, or through the Spring-managed bean reference) get the transactional behavior. A call like `this.method()` from inside the same class bypasses the proxy entirely — it's a normal Java call to the real object, so the transaction logic never runs.

**8. What is the difference between `ApplicationContext` and `BeanFactory`?**
`BeanFactory` is the basic container with lazy bean creation. `ApplicationContext` extends it and adds eager singleton instantiation at startup, event publishing, and AOP/i18n support. In practice, every real Spring/Spring Boot app uses `ApplicationContext`.

**9. What is Spring Boot auto-configuration, and how does it decide what to configure?**
It's a mechanism that configures beans automatically based on what's on the classpath and what you've already defined. It uses conditional annotations (`@ConditionalOnClass`, `@ConditionalOnMissingBean`, `@ConditionalOnProperty`) on a set of pre-written configuration classes. If the conditions match — e.g., a database driver is on the classpath and you haven't defined your own `DataSource` — Spring Boot creates a sensible default bean for you.

**10. How would you handle validation errors globally for a REST API?**
Annotate the DTO fields with constraints (`@NotNull`, `@Size`, etc.), add `@Valid` on the `@RequestBody` parameter, and add a `@ControllerAdvice` class with an `@ExceptionHandler(MethodArgumentNotValidException.class)` method that builds a clean error response instead of letting Spring return its default error format.

**11. What is the difference between `@RequestParam`, `@PathVariable`, and `@RequestBody`?**
`@RequestParam` reads a query string parameter (`?key=value`). `@PathVariable` reads a value embedded in the URL path (`/users/{id}`). `@RequestBody` reads and converts the whole request body (usually JSON) into a Java object.

**12. What does `@Qualifier` do, and how is it different from `@Primary`?**
Both solve the "multiple beans of the same type" problem. `@Primary` is set once, on the bean definition, and makes that bean the default choice everywhere. `@Qualifier` is set at each injection point, letting you pick a specific bean by name for that particular place — even overriding `@Primary` if needed.

## FAQ / Rapid-Fire

- **Is a Spring bean thread-safe by default?** No. Singleton scope means one shared instance — you must manage thread safety yourself if the bean has mutable state.
- **What is the default bean scope?** Singleton.
- **Does `@Transactional` work on a `private` method?** No, because Spring proxies work only on methods it can override/intercept (public methods are the safe default; `private`/`final` methods cannot be proxied).
- **What is `DispatcherServlet`?** The single front controller in Spring MVC that receives every HTTP request and routes it to the right handler.
- **Difference between `@Controller` and `@RestController`?** `@RestController` = `@Controller` + `@ResponseBody`; every method's return value is written directly as the response body (e.g., JSON), not resolved as a view name.
- **What is `ResponseEntity` used for?** To control the HTTP status code, headers, and body together in one return value.
- **Is `@Autowired` required on a constructor if there is only one constructor?** No, Spring 4.3+ auto-detects a single constructor and injects it even without `@Autowired`.
- **What is `@Primary` used for again?** Marking the default bean when several beans of the same type exist.
- **Does Spring Boot need a separate application server like Tomcat installed?** No, Tomcat (or another server) is embedded inside the JAR by default.
- **What does `spring.profiles.active` do?** Selects which profile's configuration (e.g., `application-dev.yml`) is active at runtime.
- **What is `@ConfigurationProperties` used for?** Binding a group of related properties into one typed Java object, instead of many separate `@Value` fields.
- **Is Spring AOP compile-time or runtime?** Runtime — it works via dynamic proxies (JDK proxy or CGLIB), not by modifying bytecode at compile time (that's AspectJ's compile-time weaving, which Spring AOP does not use by default).
- **What is the difference between checked and unchecked exception rollback in `@Transactional`?** By default, only unchecked exceptions trigger rollback; use `rollbackFor` to also roll back on checked exceptions.
- **What does `readOnly = true` guarantee?** It's a hint for optimization, not a hard guarantee — some databases/drivers may still allow writes.
- **What is a derived query method?** A repository method whose SQL is generated automatically from its name, e.g. `findByEmailAndStatus(...)`.

## Common Traps & Gotchas

- **Self-invocation breaks `@Transactional` and `@Async`.** Calling an annotated method from another method in the same class skips the proxy, so the annotation has no effect. This is the single most common Spring interview trap.
- **Assuming singleton beans are safe to hold request-specific state.** A singleton bean is shared by all threads/requests. Storing per-user data in an instance field causes data to leak between users.
- **Forgetting that `@Repository`'s exception translation only applies if Spring manages the exception-throwing code** (e.g., using Spring Data or `JdbcTemplate`). Plain custom JDBC code without Spring's helpers won't get translated automatically.
- **Using field injection everywhere.** It compiles fine but hides real dependencies, makes classes hard to unit test without a Spring context, and does not allow `final` fields.
- **Expecting `@Transactional` to roll back on checked exceptions by default.** It does not — only unchecked (`RuntimeException`) exceptions trigger rollback unless you set `rollbackFor`.
- **Confusing `@Component` scanning with `@Bean` methods.** If a class is both picked up by component scanning **and** also declared with `@Bean` somewhere, you can accidentally create two bean definitions and hit confusing startup errors.
- **Thinking `@Primary` and `@Qualifier` conflict.** `@Qualifier` at an injection point always wins over `@Primary` — they don't need to match; `@Qualifier` is simply more specific.
- **Believing more than one `@ExceptionHandler` for the same exception type in a `@ControllerAdvice` is fine.** It isn't — this causes an ambiguous mapping error at startup.
- **Forgetting `@EnableTransactionManagement` in plain (non-Boot) Spring.** Without it, `@Transactional` annotations are silently ignored. (Spring Boot enables this automatically, which is why the mistake mostly shows up in older/plain Spring projects.)
- **Mixing up `REQUIRES_NEW` with `NESTED`.** `REQUIRES_NEW` is a fully separate transaction (its own commit/rollback, needs its own DB connection). `NESTED` is a savepoint inside the same transaction/connection — rolling it back doesn't affect the outer transaction's already-done work, but a full outer rollback still undoes everything.
- **Using `@Autowired` on a field and expecting it to fail fast if the bean is missing.** By default `required = true` does throw at startup, but developers sometimes set `required = false` for convenience and then get a confusing `NullPointerException` much later, far from the real cause.
