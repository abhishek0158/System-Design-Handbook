# 8. Design Patterns & SOLID

Design patterns are reusable solutions to common design problems. SOLID principles are rules that keep code easy to change and test. Interviewers at 3–4 years check if you know *why* a pattern exists, not just its name. This sheet gives short code for each pattern and tells you where it lives in the JDK or Spring.

## Key Concepts (Quick Revision)

### SOLID Principles

**S — Single Responsibility Principle (SRP)**
A class should have only one reason to change. It should do one job.
```java
// Bad: one class does DB save AND report formatting
class OrderService {
    void save(Order o) { /* DB code */ }
    String toPdf(Order o) { /* report code */ }
}
// Good: split into two classes
class OrderRepository { void save(Order o) { } }
class OrderReportGenerator { String toPdf(Order o) { return ""; } }
```
Why it matters: smaller classes are easier to test and change without breaking unrelated features.

**O — Open/Closed Principle (OCP)**
A class should be open for extension but closed for modification. Add new behavior by adding new code, not by editing old code.
```java
interface DiscountPolicy { double apply(double price); }
class NoDiscount implements DiscountPolicy { public double apply(double p) { return p; } }
class FestivalDiscount implements DiscountPolicy { public double apply(double p) { return p * 0.9; } }
// New discount type = new class. No change to old classes.
```
Why it matters: old, tested code stays untouched, so you do not break existing behavior.

**L — Liskov Substitution Principle (LSP)**
A subclass object must work correctly wherever the parent class is expected. It must not break the parent's contract.
```java
class Rectangle { void setWidth(int w) {} void setHeight(int h) {} }
class Square extends Rectangle { // breaks LSP: setWidth also changes height
    void setWidth(int w) { /* changes both width and height */ }
}
```
Why it matters: if a subclass changes expected behavior, callers using the parent type get bugs.

**I — Interface Segregation Principle (ISP)**
Do not force a class to implement methods it does not need. Prefer many small interfaces over one large interface.
```java
// Bad: one fat interface
interface Worker { void work(); void eat(); }
// Good: split
interface Workable { void work(); }
interface Eatable { void eat(); }
class Robot implements Workable { public void work() { } } // no eat() forced on it
```
Why it matters: classes stay simple and do not carry unused methods.

**D — Dependency Inversion Principle (DIP)**
High-level code should depend on abstractions (interfaces), not on concrete classes. Low-level classes also depend on the same abstraction.
```java
interface MessageSender { void send(String msg); }
class EmailSender implements MessageSender { public void send(String msg) { } }
class NotificationService {
    private final MessageSender sender; // depends on interface, not EmailSender
    NotificationService(MessageSender sender) { this.sender = sender; }
}
```
Why it matters: you can swap `EmailSender` for `SmsSender` without changing `NotificationService`. This is the base idea behind Dependency Injection.

**Other short principles**
| Principle | Meaning | One line |
|---|---|---|
| DRY | Don't Repeat Yourself | Do not copy-paste logic; extract it into one method/class. |
| KISS | Keep It Simple, Stupid | Prefer the simple solution over a clever, complex one. |
| YAGNI | You Aren't Gonna Need It | Do not build features "just in case" you need them later. |

---

### Creational Patterns
Creational patterns control **how objects are created**.

**Singleton** — only one instance of a class exists in the whole application.
- When to use: shared config, connection pool, cache, logger.
```java
// 1. Double-checked locking with volatile
class Config {
    private static volatile Config instance;
    private Config() { }
    public static Config getInstance() {
        if (instance == null) {
            synchronized (Config.class) {
                if (instance == null) instance = new Config();
            }
        }
        return instance;
    }
}

// 2. Static holder (lazy, thread-safe, no locking cost)
class ConfigHolder {
    private ConfigHolder() { }
    private static class Holder { static final ConfigHolder INSTANCE = new ConfigHolder(); }
    public static ConfigHolder getInstance() { return Holder.INSTANCE; }
}

// 3. Enum singleton (safest, handles serialization and reflection attacks)
enum ConfigEnum {
    INSTANCE;
    void doWork() { }
}
```
- Pitfalls: reflection can break singletons made with a private constructor (enum singleton resists this). Serialization can create a second instance unless you add `readResolve()`. Singleton hides dependencies and makes unit testing harder — that is why Spring beans are a better fit (Spring manages one instance per container, but you still inject it, so it stays testable).
- Where seen: `Runtime.getInstance()` in JDK. Spring beans are singleton-scoped by default.

**Factory Method** — a method decides which subclass to create, instead of using `new` directly.
- When to use: the exact class to build depends on input or config.
```java
interface Shape { void draw(); }
class Circle implements Shape { public void draw() { } }
class Square implements Shape { public void draw() { } }
class ShapeFactory {
    static Shape create(String type) {
        return switch (type) {
            case "circle" -> new Circle();
            case "square" -> new Square();
            default -> throw new IllegalArgumentException();
        };
    }
}
```
- Where seen: `Calendar.getInstance()`, `NumberFormat.getInstance()`, Spring's `BeanFactory.getBean()`.

**Abstract Factory** — a factory of factories. It creates families of related objects without naming their concrete classes.
- When to use: you need to create a group of related products that must stay consistent (e.g., all UI widgets for one theme).
```java
interface Button { }
interface Checkbox { }
interface GuiFactory {
    Button createButton();
    Checkbox createCheckbox();
}
class DarkFactory implements GuiFactory {
    public Button createButton() { return new Button() { }; }
    public Checkbox createCheckbox() { return new Checkbox() { }; }
}
```
- Where seen: `javax.xml.parsers.DocumentBuilderFactory`, `javax.xml.transform.TransformerFactory`.

**Builder** — builds a complex object step by step, using method chaining, instead of a constructor with many arguments.
- When to use: object has many optional fields. Avoids "telescoping constructors" (many overloaded constructors).
```java
class Pizza {
    private final String size;
    private final boolean cheese;
    private Pizza(Builder b) { this.size = b.size; this.cheese = b.cheese; }
    static class Builder {
        private String size = "medium";
        private boolean cheese = false;
        Builder size(String s) { this.size = s; return this; }
        Builder cheese(boolean c) { this.cheese = c; return this; }
        Pizza build() { return new Pizza(this); }
    }
}
Pizza p = new Pizza.Builder().size("large").cheese(true).build();
```
- Where seen: `StringBuilder`, `Stream.Builder`, Lombok `@Builder`, Spring `UriComponentsBuilder`.

**Prototype** — create a new object by copying (cloning) an existing object, instead of building it from scratch.
- When to use: object creation is costly (e.g., loads data from DB), and a copy is cheaper.
```java
class Report implements Cloneable {
    String data;
    public Report clone() {
        try { return (Report) super.clone(); }
        catch (CloneNotSupportedException e) { throw new AssertionError(e); }
    }
}
```
- Where seen: `Object.clone()`, `ArrayList` copy constructors act in a similar spirit.

**Factory Method vs Abstract Factory**
| | Factory Method | Abstract Factory |
|---|---|---|
| Creates | One product | A family of related products |
| Uses | One method, often overridden | An interface with multiple creation methods |
| Example | `ShapeFactory.create("circle")` | `GuiFactory` that makes matching `Button` + `Checkbox` |

---

### Structural Patterns
Structural patterns arrange classes and objects into larger structures.

**Adapter** — converts one interface into another interface that the client expects. Lets two incompatible interfaces work together.
- When to use: you must use an existing class, but its interface does not match what your code needs.
```java
interface MediaPlayer { void play(String file); }
class LegacyPlayer { void playOldFormat(String file) { } }
class PlayerAdapter implements MediaPlayer {
    private final LegacyPlayer legacy = new LegacyPlayer();
    public void play(String file) { legacy.playOldFormat(file); }
}
```
- Where seen: `Arrays.asList()` adapts an array to a `List`. `InputStreamReader` adapts a byte stream to a character stream.

**Decorator** — adds new behavior to an object at runtime, by wrapping it, without changing its class.
- When to use: you want to add features (e.g., logging, buffering) that can combine in any order.
```java
InputStream in = new BufferedInputStream(new FileInputStream("data.txt"));
// FileInputStream is wrapped by BufferedInputStream, adding buffering
```
- `java.io` is built almost entirely on Decorator: `FileInputStream`, `BufferedInputStream`, `GZIPInputStream` all wrap a base `InputStream` and add behavior in layers.

**Proxy** — a stand-in object that controls access to the real object. Same interface as the real object.
- When to use: lazy loading, access control, logging, remote calls, or adding cross-cutting logic without touching business code.
```java
interface UserService { void save(User u); }
class UserServiceImpl implements UserService { public void save(User u) { } }
class LoggingProxy implements UserService {
    private final UserService target;
    LoggingProxy(UserService target) { this.target = target; }
    public void save(User u) {
        System.out.println("before save");
        target.save(u);
    }
}
```
- Spring AOP builds a proxy around your bean at runtime (JDK dynamic proxy if the bean implements an interface, or CGLIB subclass proxy if it does not). This is how `@Transactional`, `@Async`, and `@Cacheable` work — the proxy adds behavior around your method call.

**Facade** — a single, simple interface that hides a complex set of subsystems.
- When to use: you want to give client code one simple entry point to a complex module.
```java
class OrderFacade {
    private final InventoryService inventory = new InventoryService();
    private final PaymentService payment = new PaymentService();
    void placeOrder(Order o) {
        inventory.reserve(o);
        payment.charge(o);
    }
}
```
- Where seen: `javax.faces.context.FacesContext`, most Spring `Service` classes act as a facade over repositories and other services.

**Composite** — treats a group of objects and a single object the same way, using a shared interface. Builds tree structures (part-whole hierarchy).
- When to use: you have a tree, like files/folders or UI components, and want uniform code for leaf and group nodes.
```java
interface FileSystemItem { int getSize(); }
class File implements FileSystemItem {
    int size;
    public int getSize() { return size; }
}
class Folder implements FileSystemItem {
    List<FileSystemItem> children = new ArrayList<>();
    public int getSize() { return children.stream().mapToInt(FileSystemItem::getSize).sum(); }
}
```
- Where seen: `java.awt.Container` and `Component` (Swing UI tree). JSON/XML tree libraries.

**Proxy vs Decorator vs Adapter**
| | Adapter | Decorator | Proxy |
|---|---|---|---|
| Purpose | Change interface to match client | Add new behavior | Control access |
| Interface | Different from wrapped object | Same as wrapped object | Same as wrapped object |
| Adds features? | No | Yes, always | Sometimes (logging), or restricts access |
| Example | `InputStreamReader` | `BufferedInputStream` | Spring AOP proxy, Hibernate lazy-load proxy |

---

### Behavioral Patterns
Behavioral patterns manage how objects talk to each other and share responsibility.

**Strategy** — defines a family of algorithms, puts each in its own class, and makes them interchangeable at runtime.
- When to use: you have several ways to do one task (e.g., sorting, pricing) and want to switch between them without `if-else` chains.
```java
interface SortStrategy { void sort(int[] arr); }
class QuickSort implements SortStrategy { public void sort(int[] a) { } }
class BubbleSort implements SortStrategy { public void sort(int[] a) { } }
class Sorter {
    private SortStrategy strategy;
    Sorter(SortStrategy s) { this.strategy = s; }
    void execute(int[] a) { strategy.sort(a); }
}
```
- Lambdas connect directly to Strategy: since Java 8, a one-method interface (functional interface) can be passed as a lambda instead of a class.
```java
Comparator<String> byLength = (a, b) -> a.length() - b.length();
list.sort(byLength); // Comparator IS a Strategy
```
- Where seen: `Comparator`, `Runnable`, `Collections.sort(list, strategy)`.

**Observer** — one object (subject) notifies many other objects (observers) automatically when its state changes.
- When to use: event handling, pub/sub systems, UI updates.
```java
interface Observer { void update(String event); }
class EventBus {
    private final List<Observer> observers = new ArrayList<>();
    void subscribe(Observer o) { observers.add(o); }
    void publish(String event) { observers.forEach(o -> o.update(event)); }
}
```
- This is the base idea behind pub/sub messaging (Kafka, RabbitMQ, `ApplicationEventPublisher` in Spring). The publisher does not know who is listening; it just broadcasts.
- Where seen: `java.util.Observer` (deprecated since Java 9), Spring's `ApplicationListener` and `@EventListener`.

**Template Method** — defines the skeleton of an algorithm in a parent class, and lets subclasses override only specific steps.
- When to use: many classes share the same overall flow but differ in a few steps.
```java
abstract class DataProcessor {
    final void process() { // template: fixed order of steps
        readData();
        transformData();
        saveData();
    }
    abstract void readData();
    abstract void transformData();
    void saveData() { System.out.println("saved"); } // default step
}
```
- Where seen: `JdbcTemplate` in Spring (you provide the query and row-mapping; Spring handles connection/close). `AbstractList`, `HttpServlet.service()` (calls `doGet`, `doPost`).

**Command** — wraps a request (an action plus its data) into an object, so it can be queued, logged, or undone.
- When to use: undo/redo, task queues, GUI button actions.
```java
interface Command { void execute(); }
class LightOnCommand implements Command {
    Light light;
    public void execute() { light.on(); }
}
class RemoteControl {
    void press(Command c) { c.execute(); }
}
```
- Where seen: `Runnable`, `java.lang.Thread`, task objects submitted to `ExecutorService`.

**Iterator** — gives a way to access elements of a collection one by one, without exposing how the collection is stored inside.
- When to use: you always use it, often without noticing, when you loop over a collection.
```java
Iterator<String> it = list.iterator();
while (it.hasNext()) {
    System.out.println(it.next());
}
```
- Where seen: `java.util.Iterator`, the for-each loop (`for (String s : list)`) uses this pattern behind the scenes.

**Chain of Responsibility** — passes a request along a chain of handlers. Each handler decides to process it or pass it to the next one.
- When to use: multiple objects can handle a request, and you do not want to hard-code who handles what.
```java
abstract class Handler {
    protected Handler next;
    Handler setNext(Handler next) { this.next = next; return next; }
    abstract void handle(String request);
}
class AuthHandler extends Handler {
    void handle(String req) {
        if (!req.contains("token")) { System.out.println("blocked"); return; }
        if (next != null) next.handle(req);
    }
}
```
- Servlet filters (`javax.servlet.Filter`) and Spring Security's `FilterChain` are a direct, real-world use of this pattern. Each filter does its check and calls `chain.doFilter()` to pass control to the next filter.
- Where seen: Servlet `FilterChain`, Spring's `HandlerInterceptor` chain, logging frameworks (`log4j` appender chain).

**Strategy vs State**
| | Strategy | State |
|---|---|---|
| Purpose | Choose one algorithm among many | Change behavior when internal state changes |
| Who switches it | Client code picks the strategy | The object itself switches state, often internally |
| Awareness | States/strategies do not know each other | States often know about and trigger other states |
| Example | `Comparator` passed to `sort()` | An `Order` object behaving differently in `NEW`, `PAID`, `SHIPPED` states |

---

### Dependency Injection (DI) as a pattern
DI means an object receives its dependencies from outside, instead of creating them itself. It is a specific way to apply the Dependency Inversion Principle.
```java
// Without DI: class creates its own dependency (tight coupling)
class OrderService { private PaymentGateway gateway = new StripeGateway(); }

// With DI: dependency is passed in (constructor injection)
class OrderService {
    private final PaymentGateway gateway;
    OrderService(PaymentGateway gateway) { this.gateway = gateway; }
}
```
Spring implements DI through its **IoC (Inversion of Control) container**. You mark classes with `@Component`/`@Service`/`@Repository`, and Spring:
1. Scans and creates the beans.
2. Resolves dependencies between beans.
3. Injects them, usually through constructor injection (recommended, since it allows `final` fields and easy unit testing with plain `new`).

## Important Interview Questions

**Q1: Why is constructor injection preferred over field injection in Spring?**
Constructor injection lets you make fields `final`, makes dependencies clear and required, and lets you unit test the class with plain `new` calls, without starting Spring. Field injection (`@Autowired` on a field) hides dependencies and needs reflection to set them in tests.
*Follow-up: When would you still use field/setter injection?* For optional dependencies, or in legacy code you cannot refactor easily.

**Q2: How does Spring AOP use the Proxy pattern for `@Transactional`?**
Spring wraps your bean in a proxy object. When you call a method with `@Transactional`, you are actually calling the proxy first. The proxy starts a transaction, calls your real method, then commits or rolls back. This is why calling a `@Transactional` method from another method **inside the same class** does not trigger the proxy — it bypasses it because you call `this.method()` directly, not the proxy.

**Q3: Why is Singleton called an anti-pattern by some developers, if the GoF book lists it as a valid pattern?**
Because it introduces global state, hides real dependencies (a class using `Config.getInstance()` does not show that dependency in its constructor), and makes unit testing hard (you cannot easily swap it with a mock). Spring solves this by keeping the "one instance" benefit but injecting it as a dependency, so tests can still supply a different bean.

**Q4: What is the difference between Factory Method and Abstract Factory, with a real example?**
Factory Method makes one product using one creation method, often through inheritance (a subclass overrides `create()`). Abstract Factory makes a **family** of related products through composition (you hold a factory object with several creation methods). Example: `ShapeFactory.create("circle")` is Factory Method. A `GuiFactory` that creates a matching `Button` and `Checkbox` for one theme is Abstract Factory.

**Q5: How does the Decorator pattern in `java.io` avoid class explosion?**
Without Decorator, you would need one class for every combination of feature (buffered file stream, buffered zipped file stream, and so on). With Decorator, each wrapper class (`BufferedInputStream`, `GZIPInputStream`) adds one feature and can wrap any other `InputStream`. You combine them at runtime by nesting constructors, so features multiply without new classes for every combination.
*Follow-up: What is the downside?* Debugging is harder because you must trace through several wrapper layers to see actual behavior.

**Q6: Explain Open/Closed Principle with a Strategy pattern example.**
Say you calculate discounts with `if-else` on discount type. Adding a new discount type means editing that method (violates OCP). With Strategy, each discount type is its own class implementing `DiscountPolicy`. Adding a new discount means adding a new class, and the code that uses `DiscountPolicy` never changes.

**Q7: Chain of Responsibility — how do servlet filters use it, and what happens if a filter forgets to call `chain.doFilter()`?**
Each filter in the chain gets a reference to the next filter (through `FilterChain`). It does its own work (like checking auth), then calls `chain.doFilter(request, response)` to pass control forward. If a filter does not call this, the request stops there — later filters and the actual servlet never run. This is a common bug in custom Spring Security filters.

**Q8: Why does Liskov Substitution Principle say `Square extends Rectangle` is often wrong?**
Because a `Rectangle` contract expects `setWidth()` and `setHeight()` to be independent. A `Square` must keep width equal to height, so setting one also changes the other. Code written for `Rectangle` (assuming independent sides) breaks when given a `Square`. This shows inheritance should model true "is-a" behavior, not just shared data.

**Q9: When would you use Builder instead of a constructor with default values (telescoping constructors)?**
When a class has many optional fields (say, 6 or more) and only a few are required. Builder gives named, chainable methods (`.size("large").cheese(true)`), which is much more readable than a constructor call with many positional arguments, some of which are hard to tell apart (e.g., two `boolean` parameters in a row).

**Q10: How is Template Method different from Strategy, since both let you plug in custom behavior?**
Template Method uses **inheritance**: the parent class defines the fixed step order, and a subclass overrides some steps. Strategy uses **composition**: the algorithm is a separate object passed in, and you can change it at runtime without subclassing. Template Method is good when the overall flow is fixed; Strategy is better when you need to switch behavior dynamically.

## FAQ / Rapid-Fire

- **Is Singleton thread-safe by default?** No. A plain `if (instance == null) instance = new X();` is not thread-safe. Use double-checked locking with `volatile`, a static holder class, or an enum.
- **Why use `volatile` in double-checked locking?** Without it, another thread might see a half-constructed object, because of instruction reordering during object creation.
- **Is enum singleton really the best way?** Yes, for most cases. It is thread-safe by default, and the JVM guarantees only one instance, even against reflection and serialization attacks.
- **Does Spring use classic Java Singleton?** No. Spring's "singleton scope" means one instance per Spring container (`ApplicationContext`), managed by Spring, not a static `getInstance()` method.
- **What is the difference between Proxy and Decorator if both wrap an object?** Proxy controls access (may block or delay the real call); Decorator always adds new behavior and always calls through to the wrapped object.
- **Is `Optional` a design pattern?** No, it is a container type to avoid `null`. It is not one of the classic 23 GoF patterns.
- **Can one class use multiple patterns together?** Yes. Example: a Spring `@Service` (Facade) may use Strategy for pricing and be wrapped in a Proxy for `@Transactional`.
- **What is the difference between Composite and Decorator?** Composite builds a tree of part-whole objects (folder contains files/folders). Decorator wraps one object to add behavior. Both use recursion-like structure, but their intent is different.
- **Is DI only possible through frameworks like Spring?** No. You can do DI manually by passing dependencies through constructors. Spring just automates the wiring.
- **What is Inversion of Control (IoC)?** A general idea where the framework controls the flow and calls your code (instead of your code calling the framework). DI is one way to achieve IoC.
- **Does `Comparator` count as Strategy or a lambda?** Both. `Comparator` is a functional interface, so it is a Strategy pattern, and since Java 8 you usually supply it as a lambda instead of writing a class.
- **What replaced `java.util.Observer`?** It was deprecated in Java 9. Use `PropertyChangeListener`, reactive streams (`Flow` API), or an event bus / message broker instead.

## Common Traps & Gotchas

- **Singleton without `volatile` in double-checked locking**: looks correct but can return a partially built object to another thread. Always add `volatile`.
- **Calling a `@Transactional` method from inside the same class**: the internal call skips the Spring proxy, so the transaction never starts. Move the method to another bean, or use `AopContext.currentProxy()` (with `exposeProxy=true`) if you must call it internally.
- **Confusing Proxy with Decorator**: both wrap an object with the same interface. Ask: does it *add* behavior always (Decorator), or *control/restrict* access (Proxy)? Spring AOP proxies do both — they add advice, but also enable lazy or controlled behavior.
- **Overusing Singleton for things that are not truly single**: e.g., using it for a `Random` object shared across threads without synchronization, causing race conditions.
- **Fat interfaces (violating ISP)**: adding every possible method to one interface "to be safe" forces unrelated classes to implement methods they do not need, often as empty stubs.
- **Breaking LSP silently**: a subclass that throws `UnsupportedOperationException` for a parent method (like `List.add()` in an immutable list) technically compiles but breaks callers expecting the parent's contract.
- **Builder without validation**: forgetting to check required fields in `build()` lets you construct invalid objects (e.g., a `Pizza` with no size set silently defaulting).
- **Chain of Responsibility with no default handler**: if no handler in the chain processes the request and none calls the next handler, the request silently disappears with no error.
- **Applying Abstract Factory when Factory Method is enough**: this adds unnecessary classes and interfaces for a simple one-product case. Use the simplest pattern that solves the actual problem (this is the KISS principle applied to pattern choice).
- **YAGNI violation with patterns**: adding a Strategy or Observer "in case we need flexibility later" when there is only one implementation today. This adds complexity with no current benefit.
