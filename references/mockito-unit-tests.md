# Plain Unit Tests — JUnit 5 + Mockito 3.12.4

The default choice on this stack: no Spring context, no database, no Docker. Milliseconds per test.

## When to use this reference

- Testing a service, utility, mapper, validator or calculator
- Collaborators can be passed in (constructor injection, setter, or a field you can set in the test)
- You do **not** need Spring auto-configuration, JPA or HTTP semantics
- If you need a Spring bean you cannot build by hand → [mockbean.md](mockbean.md) and the slice references instead

## Basic structure

```java
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.ExtendWith;
import org.mockito.InjectMocks;
import org.mockito.Mock;
import org.mockito.junit.jupiter.MockitoExtension;

import static org.assertj.core.api.Assertions.assertThat;
import static org.mockito.ArgumentMatchers.any;
import static org.mockito.BDDMockito.given;
import static org.mockito.Mockito.verify;

@ExtendWith(MockitoExtension.class)
class OrderServiceTest {

  @Mock
  private OrderRepository orderRepository;

  @Mock
  private PaymentGateway paymentGateway;

  @InjectMocks
  private OrderService orderService;

  @Test
  void shouldCreateOrderAndCharge() {
    OrderRequest request = new OrderRequest("Laptop", 2);
    given(orderRepository.save(any(Order.class))).willAnswer(invocation -> invocation.getArgument(0));
    given(paymentGateway.charge(new BigDecimal("1999.98"))).willReturn("txn-1");

    Order created = orderService.create(request);

    assertThat(created.getStatus()).isEqualTo("PAID");
    assertThat(created.getTransactionId()).isEqualTo("txn-1");
    verify(orderRepository).save(any(Order.class));
  }
}
```

`@InjectMocks` prefers constructor injection. If the class under test has a constructor taking its collaborators, prefer building it explicitly — it is clearer and survives refactoring:

```java
private OrderService orderService;

@BeforeEach
void setUp() {
  orderService = new OrderService(orderRepository, paymentGateway);
}
```

## Without the extension

If a test class cannot use the extension (for example it extends a legacy base class):

```java
class OrderServiceTest {

  private AutoCloseable mocks;

  @Mock private OrderRepository orderRepository;

  @BeforeEach
  void setUp() {
    mocks = MockitoAnnotations.openMocks(this);   // Mockito 3.4+; replaces initMocks
  }

  @AfterEach
  void tearDown() throws Exception {
    mocks.close();
  }
}
```

## Stubbing

```java
// value
given(orderRepository.findById(1L)).willReturn(Optional.of(order));

// empty Optional — Java 8 has no Optional.isEmpty(), but Mockito/AssertJ are fine
given(orderRepository.findById(999L)).willReturn(Optional.empty());

// exception
given(paymentGateway.charge(any(BigDecimal.class)))
    .willThrow(new PaymentDeclinedException("insufficient funds"));

// consecutive calls
given(clock.now()).willReturn(t1, t2, t3);

// answer based on the argument
given(orderRepository.save(any(Order.class)))
    .willAnswer(new Answer<Order>() {
      @Override
      public Order answer(InvocationOnMock invocation) {
        Order o = invocation.getArgument(0);
        o.setId(42L);
        return o;
      }
    });

// void method
willThrow(new IllegalStateException()).given(auditLog).record(any(Order.class));
```

Lambdas work for `Answer` too (Java 8): `willAnswer(invocation -> ...)`.

Prefer the BDD aliases (`given`/`willReturn`/`then`) over `when`/`thenReturn` for consistency with the Given-When-Then layout of the test — both exist, do not mix them inside one class.

## Argument matchers

```java
given(orderRepository.findByStatus("PENDING")).willReturn(orders);      // exact value
given(orderRepository.save(any(Order.class))).willReturn(order);        // any instance
given(pricing.quote(eq("SKU-1"), anyInt())).willReturn(price);          // mix: eq() required
given(repo.find(argThat(new ArgumentMatcher<Query>() {                  // custom predicate
  @Override public boolean matches(Query q) { return q.getLimit() == 10; }
}))).willReturn(rows);
```

Rule: as soon as one argument uses a matcher, **all** arguments must use matchers — wrap literals in `eq(...)`.

## Verification

```java
verify(orderRepository).save(order);                 // exactly once (default)
verify(orderRepository, times(2)).save(any(Order.class));
verify(orderRepository, never()).delete(any(Order.class));
verify(auditLog, atLeastOnce()).record(any(Order.class));
verifyNoMoreInteractions(paymentGateway);

InOrder inOrder = inOrder(orderRepository, paymentGateway);
inOrder.verify(paymentGateway).charge(any(BigDecimal.class));
inOrder.verify(orderRepository).save(any(Order.class));
```

Verify only what matters. A test that verifies every interaction breaks on every refactoring and asserts nothing about behavior.

## ArgumentCaptor

Use it to assert on what was passed to a collaborator:

```java
@Captor
private ArgumentCaptor<Order> orderCaptor;

@Test
void shouldPersistOrderWithCalculatedTotal() {
  orderService.create(new OrderRequest("Laptop", 2));

  verify(orderRepository).save(orderCaptor.capture());

  Order saved = orderCaptor.getValue();
  assertThat(saved.getTotal()).isEqualTo(new BigDecimal("1999.98"));
  assertThat(saved.getStatus()).isEqualTo("PENDING");
}
```

For repeated calls: `orderCaptor.getAllValues()` returns a `List` in call order.

## Strict stubs

`MockitoExtension` defaults to `Strictness.STRICT_STUBS`: an unused stub fails the test, and an argument mismatch reports it clearly instead of returning `null`. Keep it.

When one stub is genuinely only needed by some test methods in the class:

```java
@Test
void someTest() {
  lenient().when(clock.now()).thenReturn(FIXED_TIME);
}
```

Or relax the whole class only as a last resort:

```java
@ExtendWith(MockitoExtension.class)
@MockitoSettings(strictness = Strictness.LENIENT)
class LegacyServiceTest { }
```

## What cannot be mocked here

`mockito-inline` is not part of the dependency set, so there is **no** `mockStatic` and **no** `mockConstruction`, and final classes / final methods cannot be mocked.

If the code under test calls a static utility:

```java
// Hard to test: static call baked into the method
public Order create(OrderRequest request) {
  LocalDateTime now = DateUtils.nowInSystemZone();   // static
  ...
}

// Testable: the clock becomes a collaborator
public class OrderService {
  private final Clock clock;

  public OrderService(OrderRepository repo, Clock clock) { ... }

  public Order create(OrderRequest request) {
    LocalDateTime now = LocalDateTime.now(clock);
    ...
  }
}
```

In the test: `Clock.fixed(Instant.parse("2024-01-15T10:30:00Z"), ZoneOffset.UTC)` — deterministic, no mocking framework needed.

## Parameterized tests

Needs `junit-jupiter-params` (5.9.3):

```java
@ParameterizedTest
@CsvSource({
  "100, VIP,      10",
  "100, STANDARD,  0",
  "500, VIP,      50"
})
void shouldCalculateDiscount(BigDecimal amount, String tier, BigDecimal expected) {
  assertThat(calculator.discountFor(amount, tier)).isEqualByComparingTo(expected);
}

@ParameterizedTest
@ValueSource(strings = {"", "   ", "\t"})
void shouldRejectBlankProductName(String name) {
  assertThatThrownBy(() -> new OrderRequest(name, 1))
      .isInstanceOf(IllegalArgumentException.class);
}

@ParameterizedTest
@MethodSource("invalidRequests")
void shouldRejectInvalidRequests(OrderRequest request, String expectedMessage) {
  assertThatThrownBy(() -> orderService.create(request))
      .hasMessageContaining(expectedMessage);
}

static Stream<Arguments> invalidRequests() {
  return Stream.of(
    Arguments.of(new OrderRequest("Laptop", 0), "quantity"),
    Arguments.of(new OrderRequest(null, 1), "product")
  );
}
```

`@MethodSource` factory methods must be `static` (Java 8 has no static methods on inner classes, so keep them on the outer test class or use `@TestInstance(Lifecycle.PER_CLASS)`).

## Grouping with @Nested

```java
class OrderServiceTest {

  @Nested
  @ExtendWith(MockitoExtension.class)
  class Create {
    @Test void shouldChargeCustomer() { }
    @Test void shouldRejectEmptyCart() { }
  }

  @Nested
  @ExtendWith(MockitoExtension.class)
  class Cancel {
    @Test void shouldRefund() { }
  }
}
```

Note: `@Nested` classes are inner classes, so on Java 8 they cannot declare `static` members — keep `@MethodSource` factories and constants in the outer class.

## Test data without a generator

No Instancio or Datafaker in this dependency set. Use hand-written fixtures, and keep them next to the tests that use them:

```java
final class OrderFixtures {

  static Order pendingOrder() {
    return pendingOrder(1L);
  }

  static Order pendingOrder(long id) {
    Order order = new Order();
    order.setId(id);
    order.setStatus("PENDING");
    order.setTotal(new BigDecimal("99.99"));
    order.setCreatedAt(LocalDateTime.of(2024, 1, 15, 10, 30));
    return order;
  }

  private OrderFixtures() { }
}
```

A fluent builder pays off once more than three tests need variations of the same object. Keep fixed timestamps and IDs — random data makes failures unreproducible.

## Key points

1. **No Spring context** — if the test needs one, you picked the wrong level
2. **`@ExtendWith(MockitoExtension.class)`**, not `SpringExtension`
3. **Strict stubs on** — delete the stub the error points at instead of switching to lenient
4. **Mock collaborators, never the class under test**
5. **`ArgumentCaptor` for "what was saved"**, `verify` for "was it called"
6. **No `mockStatic`** — refactor the static seam instead
7. **Java 8 syntax only** — explicit types, `Arrays.asList`, no text blocks
