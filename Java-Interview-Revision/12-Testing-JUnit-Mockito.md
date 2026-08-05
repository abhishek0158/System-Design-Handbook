# 12. Testing — JUnit & Mockito

Testing means checking that your code works as expected, automatically, without a human clicking through the app. For a 3-4 year backend developer, this is a daily skill, not a theory topic. Interviewers expect you to write clean JUnit 5 tests, mock dependencies with Mockito, and test Spring Boot layers correctly.

## Key Concepts (Quick Revision)

**Why unit testing matters**
- Catches bugs early, before they reach QA or production.
- Acts as a safety net when you refactor code. If tests still pass, your change is probably safe.
- Works as living documentation. A test shows how a method is supposed to behave.
- Makes code review and CI/CD pipelines reliable (build fails fast if logic breaks).

**Test types and the Test Pyramid**
The test pyramid is a model that shows how many tests of each type you should write.

| Type | What it checks | Speed | Count |
|---|---|---|---|
| Unit test | One class/method in isolation, dependencies mocked | Very fast | Many (base of pyramid) |
| Integration test | Multiple components together (e.g., service + real DB) | Slower | Fewer (middle) |
| End-to-end (E2E) test | Full system, through UI or API, like a real user | Slowest | Very few (top) |

Idea: write lots of cheap, fast unit tests. Write fewer, costlier integration and E2E tests.

**JUnit 5 structure**
JUnit 5 is the standard testing framework for Java. It has three modules: JUnit Platform (runs tests), Jupiter (new API), Vintage (runs old JUnit 3/4 tests).

- `@Test` — marks a method as a test case.
- `@BeforeEach` / `@AfterEach` — run before/after **every** test method. Good for setup/cleanup like creating fresh objects.
- `@BeforeAll` / `@AfterAll` — run **once** before/after all tests in the class. Must be `static` (unless you use `@TestInstance(PER_CLASS)`).
- `@DisplayName("...")` — gives a readable name for test reports, instead of the method name.
- `@Disabled("reason")` — skips a test temporarily. Always give a reason.
- Assumptions (`assumeTrue(...)`) — skip a test if a condition is false (e.g., skip on a certain OS). Different from assertions: assumption failure = test skipped, not failed.

**Common assertions**
```java
assertEquals(expected, actual);
assertTrue(condition);
assertNull(value);
assertThrows(IllegalArgumentException.class, () -> service.process(null));
assertAll(
    () -> assertEquals(2, list.size()),
    () -> assertTrue(list.contains("A"))
);
```
`assertAll` runs all checks even if one fails, and reports all failures together. Normal assertions stop at the first failure.

**@ParameterizedTest**
Runs the same test logic with different input values. Avoids copy-pasting near-identical tests.
```java
@ParameterizedTest
@ValueSource(strings = {"", " ", "\t"})
void blankStrings_shouldBeInvalid(String input) {
    assertFalse(validator.isValid(input));
}
```
Other sources: `@CsvSource`, `@MethodSource`, `@EnumSource`.

**AAA Pattern (Arrange-Act-Assert)**
A simple structure for writing any test, in three steps:
1. **Arrange** — set up objects, inputs, and mocks.
2. **Act** — call the method under test.
3. **Assert** — check the result is what you expect.

Keeping these three parts separate (even with blank lines or comments) makes tests easy to read.

**What makes a good unit test (F.I.R.S.T)**
- **Fast** — runs in milliseconds, no network/DB calls.
- **Isolated/Independent** — does not depend on other tests or run order.
- **Repeatable** — same result every time, on any machine.
- **Self-validating** — pass/fail is automatic (no manual log checking).
- **Timely** — written close to when the code is written (ideally before, in TDD).

**Mockito — mock, stub, spy definitions**
- **Mock**: a fake object that simulates a real dependency. You control what it returns. By default, methods return null/0/false unless you tell it otherwise.
- **Stub**: the act of telling a mock what to return for a specific call. `when(x).thenReturn(y)` creates a stub.
- **Spy**: wraps a **real** object. Real methods run unless you explicitly override them. Useful when you want most real behavior but need to override one method.

**Core Mockito annotations and methods**
- `@Mock` — creates a mock object.
- `@InjectMocks` — creates the real object under test and injects the `@Mock` fields into it (via constructor, setter, or field injection).
- `@ExtendWith(MockitoExtension.class)` — JUnit 5 extension that initializes `@Mock`/`@InjectMocks` annotations automatically. Without it, you need `MockitoAnnotations.openMocks(this)`.
- `when(mock.method()).thenReturn(value)` — stub a return value.
- `verify(mock).method(args)` — check a method was actually called (with given args). Confirms interaction, not just result.
- `verify(mock, times(2)).method()` — check call count. Other options: `never()`, `atLeastOnce()`, `atMost(n)`.
- Argument matchers — `any()`, `anyString()`, `eq(value)`. Rule: if you use one matcher in a call, **all** arguments in that call must use matchers.
- Mocking void methods — `when().thenReturn()` does not work for void methods. Use:
    - `doNothing().when(mock).voidMethod();`
    - `doThrow(new RuntimeException()).when(mock).voidMethod();`

**Mockito test example**
```java
@ExtendWith(MockitoExtension.class)
class OrderServiceTest {

    @Mock
    private PaymentGateway paymentGateway;

    @InjectMocks
    private OrderService orderService;

    @Test
    void placeOrder_shouldChargeCustomer_andReturnConfirmation() {
        // Arrange
        when(paymentGateway.charge(100.0)).thenReturn(true);

        // Act
        String result = orderService.placeOrder(100.0);

        // Assert
        assertEquals("CONFIRMED", result);
        verify(paymentGateway).charge(100.0);
    }

    @Test
    void placeOrder_shouldThrow_whenPaymentFails() {
        doThrow(new PaymentException("declined"))
            .when(paymentGateway).charge(anyDouble());

        assertThrows(PaymentException.class, () -> orderService.placeOrder(50.0));
    }
}
```

**A plain JUnit 5 test**
```java
class CalculatorTest {

    private Calculator calculator;

    @BeforeEach
    void setUp() {
        calculator = new Calculator();
    }

    @Test
    @DisplayName("adding two positive numbers returns their sum")
    void add_twoPositiveNumbers_returnsSum() {
        int result = calculator.add(2, 3);
        assertEquals(5, result);
    }

    @Test
    void divide_byZero_throwsException() {
        assertThrows(ArithmeticException.class, () -> calculator.divide(10, 0));
    }
}
```

**Testing in Spring Boot**
- `@SpringBootTest` — loads the **full** application context (all beans). Slow but tests real wiring. Use for integration tests.
- Slice tests — load only the layer you need, faster than `@SpringBootTest`:
    - `@WebMvcTest(Controller.class)` — loads only the web layer (controllers, filters, `@ControllerAdvice`). Service/repository beans are not loaded; you mock them.
    - `@DataJpaTest` — loads only JPA repositories and an in-memory DB (like H2) by default. Good for testing queries.
- `@MockBean` — replaces a real Spring bean with a Mockito mock, inside the Spring context. Different from `@Mock`, which is plain Mockito with no Spring context involved.
- `MockMvc` — a tool to test controllers by simulating HTTP requests, without starting a real server.

**MockMvc controller test example**
```java
@WebMvcTest(UserController.class)
class UserControllerTest {

    @Autowired
    private MockMvc mockMvc;

    @MockBean
    private UserService userService;

    @Test
    void getUser_shouldReturnUserJson() throws Exception {
        when(userService.findById(1L)).thenReturn(new User(1L, "Alice"));

        mockMvc.perform(get("/users/1"))
            .andExpect(status().isOk())
            .andExpect(jsonPath("$.name").value("Alice"));
    }
}
```

**Test coverage**
Test coverage is a percentage that shows how much of your code ran during tests (e.g., line coverage, branch coverage). Tools: JaCoCo (common with Maven/Gradle).
- Limits: 100% coverage does **not** mean bug-free code. A test can execute a line without actually asserting anything meaningful.
- Coverage tells you what code was **touched**, not whether the logic is **correct**. Treat it as a guide, not a goal.

**TDD (Test-Driven Development)**
TDD is a way of writing code where you write the test **before** the actual code. The cycle is: **Red** (write a failing test) → **Green** (write the minimum code to pass it) → **Refactor** (clean up the code, keeping tests green). The benefit is that it forces you to think about the required behavior first, and it naturally leads to high test coverage. In real projects, teams often do not follow strict TDD, but they still value "test as you code" instead of leaving tests for the end.

**Best practices**
- One logical assert (one behavior) per test. Multiple `assertEquals` calls are fine if they check the same outcome.
- No logic (loops, if/else) inside test code. Tests should be simple and obviously correct.
- Test **behavior**, not **implementation**. Do not assert on private fields or internal method calls that are not part of the contract.
- Use clear naming: `methodName_condition_expectedResult`.
- Keep tests independent — do not rely on test execution order or shared mutable state.

## Important Interview Questions

**Q1: What is the difference between `@Mock` and `@MockBean`?**
`@Mock` is plain Mockito. It creates a mock object with no Spring context involved — used in plain unit tests. `@MockBean` is a Spring Boot Test annotation. It replaces a real bean in the Spring application context with a mock, used inside `@SpringBootTest` or slice tests.
*Follow-up: Why is `@MockBean` slower than `@Mock`?* Because it needs a Spring context to be loaded (even if partial), while `@Mock` needs nothing beyond the JVM.

**Q2: What is the difference between a mock and a spy?**
A mock is fully fake — all methods return default values unless stubbed. A spy wraps a real object — real methods run unless you explicitly override them with `when()` or `doReturn()`.
*Follow-up: Why is `doReturn().when()` recommended over `when().thenReturn()` for spies?* Because `when(spy.method())` actually calls the real method first (to record it), which can throw exceptions or have side effects. `doReturn()` never calls the real method.

**Q3: What is the difference between `@BeforeEach` and `@BeforeAll`?**
`@BeforeEach` runs before every single test method — good for fresh state per test. `@BeforeAll` runs once before all tests in the class — good for expensive one-time setup (e.g., starting a shared resource). `@BeforeAll` methods must be `static` by default.
*Follow-up: How do you avoid making `@BeforeAll` static?* Use `@TestInstance(Lifecycle.PER_CLASS)` on the class.

**Q4: Why do we use `@ExtendWith(MockitoExtension.class)`?**
It tells JUnit 5 to let Mockito initialize the `@Mock` and `@InjectMocks` fields automatically before each test, and it validates mock usage (e.g., detects unnecessary stubbing).
*Follow-up: What happens if you forget it?* Your `@Mock` fields stay `null` unless you manually call `MockitoAnnotations.openMocks(this)`.

**Q5: How do you test a method that returns `void`?**
You cannot use `when().thenReturn()` since there is no return value. Instead:
- Use `verify(mock).method(args)` to confirm it was called correctly.
- Use `doThrow()` or `doNothing()` if you need to define its behavior on a mock.

**Q6: What is the difference between `@WebMvcTest` and `@SpringBootTest`?**
`@WebMvcTest` loads only the web layer (fast, focused, needs `@MockBean` for services). `@SpringBootTest` loads the entire application context (slow, more realistic, closer to integration testing).
*Follow-up: When would you choose one over the other?* Use `@WebMvcTest` for testing controller logic, validation, and JSON mapping. Use `@SpringBootTest` when you need to verify real wiring across layers, or for true integration tests.

**Q7: What does high test coverage actually guarantee?**
It only guarantees that lines/branches were executed during tests. It does not guarantee the assertions are meaningful or that edge cases are handled. A test with no assertions can still give 100% coverage on a line.

**Q8: What is argument matcher misuse in Mockito, and why does it cause errors?**
If you use a matcher like `any()` for one argument in a mocked call, you must use matchers for **all** arguments in that same call (mixing raw values and matchers throws `InvalidUseOfMatchersException`). Example fix: replace raw values with `eq(value)` when mixing with `any()`.

## FAQ / Rapid-Fire

- **JUnit 4 vs JUnit 5?** JUnit 5 is modular (Platform + Jupiter + Vintage), supports `@ParameterizedTest`, and uses `@ExtendWith` instead of `@RunWith`.
- **What is `assertAll` for?** To check multiple assertions together and see all failures at once, instead of stopping at the first one.
- **Can you use both Mockito and real objects in one test?** Yes — this is common with spies, or when testing a real object with some mocked dependencies via `@InjectMocks`.
- **What is `@DataJpaTest` good for?** Testing repository queries against an in-memory DB (like H2), fast and isolated from the rest of the app.
- **Does `@SpringBootTest` use a real database?** By default, it uses whatever datasource is configured (can be a real DB or an in-memory one, depending on setup).
- **What is `verifyNoInteractions(mock)`?** Confirms that a mock was never called at all during the test.
- **What is `reset(mock)`?** Clears all stubbing and interactions on a mock. Rarely needed if tests are properly isolated; often a sign of a badly designed test.
- **Is `@Disabled` a good long-term solution?** No. It should be temporary, with a reason and a plan to fix or remove the test.
- **What is the difference between `assertThrows` and try-catch for exception testing?** `assertThrows` is cleaner, fails the test if no exception is thrown, and returns the exception so you can assert on its message.
- **Do slice tests load `@Service` beans?** No, `@WebMvcTest` does not load `@Service` or `@Repository` beans. You must mock them with `@MockBean`.

## Common Traps & Gotchas

- **Forgetting `@ExtendWith(MockitoExtension.class)`** — mocks stay null, tests fail with `NullPointerException`.
- **Using `when()` on a spy for a method with side effects** — it calls the real method first. Use `doReturn().when(spy)` instead.
- **Mixing matchers and raw values in the same call** — `verify(mock).method(any(), "literal")` throws an exception. Fix: `verify(mock).method(any(), eq("literal"))`.
- **Overusing `@SpringBootTest` for everything** — makes the test suite slow. Prefer slice tests or plain unit tests where possible.
- **Testing implementation details** — asserting a private method was called, or checking internal field values directly, makes tests brittle. Refactoring breaks tests even when behavior is correct.
- **Not resetting state between tests** — shared static fields or singleton state can leak between tests, causing random failures depending on run order.
- **Treating high coverage as "well tested"** — coverage tools do not check assertion quality. A test can touch every line and still test nothing meaningful.
- **Forgetting `MockMvc` needs the right annotation** — `@Autowired MockMvc` only works if the test class is annotated with `@WebMvcTest` or `@SpringBootTest(webEnvironment = ...)` with `@AutoConfigureMockMvc`.
- **Void method stubbing with `when()`** — `when(mock.voidMethod())` does not compile/work. Must use `doNothing()`/`doThrow()` syntax instead.
- **Unnecessary stubbing warnings** — Mockito's strict stubs (default with `MockitoExtension`) will fail tests if a stub is defined but never used. Remove unused `when()` calls.
