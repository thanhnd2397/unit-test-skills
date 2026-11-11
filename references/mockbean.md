# @MockBean / @SpyBean (Boot 2.0.3)

Replacing a bean inside a Spring test context. The Boot 2.0.3 equivalent of the modern `@MockitoBean` / `@MockitoSpyBean`.

## Imports

```java
import org.springframework.boot.test.mock.mockito.MockBean;
import org.springframework.boot.test.mock.mockito.SpyBean;
```

`@MockitoBean` and `@MockitoSpyBean` (Spring Framework 6.2 / Boot 3.4) do not exist here. `@MockBean` is **not** deprecated on this version — it is the correct annotation.

## When to use which

| Need | Annotation |
|---|---|
| No Spring context at all | plain `@Mock` → [mockito-unit-tests.md](mockito-unit-tests.md) |
| Replace a bean in the context with a mock | `@MockBean` |
| Keep the real bean but stub/verify one method | `@SpyBean` |
| Supply a different real implementation | `@TestConfiguration` with a `@Bean` method |

## @MockBean in a slice test

```java
@ExtendWith(SpringExtension.class)
@WebMvcTest(OrderController.class)
class OrderControllerTest {

  @Autowired
  private MockMvc mvc;

  @MockBean
  private OrderService orderService;

  @Test
  void shouldReturnOrder() throws Exception {
    given(orderService.findById(1L)).willReturn(OrderFixtures.pendingOrder());

    mvc.perform(get("/orders/1"))
       .andExpect(status().isOk())
       .andExpect(jsonPath("$.status").value("PENDING"));
  }
}
```

Every collaborator the controller needs must be a `@MockBean`, otherwise the context fails to start with `NoSuchBeanDefinitionException`.

## @MockBean in @SpringBootTest

```java
@ExtendWith(SpringExtension.class)
@SpringBootTest
class OrderIntegrationTest {

  @Autowired
  private OrderService orderService;

  @MockBean
  private PaymentGateway paymentGateway;   // never call the real payment provider in a test

  @Test
  void shouldMarkOrderPaid() {
    given(paymentGateway.charge(any(BigDecimal.class))).willReturn("txn-1");

    Order order = orderService.create(new OrderRequest("Laptop", 1));

    assertThat(order.getStatus()).isEqualTo("PAID");
  }
}
```

## @SpyBean

Wraps the real bean; un-stubbed methods run the real implementation.

```java
@ExtendWith(SpringExtension.class)
@SpringBootTest
class NotificationServiceTest {

  @SpyBean
  private NotificationService notificationService;

  @Autowired
  private OrderService orderService;

  @Test
  void shouldNotifyCustomerOnce() {
    orderService.create(new OrderRequest("Laptop", 1));

    verify(notificationService).notifyCustomer(any(Order.class));
  }
}
```

Stub a spy with `willDoNothing()` / `doReturn(...)` form to avoid actually invoking the real method while stubbing:

```java
willDoNothing().given(notificationService).notifyCustomer(any(Order.class));
```

`given(spy.method()).willReturn(x)` calls the real method once during stubbing — use the `given(...).will*` form shown above (`willDoNothing().given(spy)`) for spies.

## Reset behavior

Boot resets `@MockBean` / `@SpyBean` after every test method (`MockReset.AFTER`), so stubs do not leak between tests. You do not need `Mockito.reset(...)` — and calling it manually usually signals that the test is doing too much.

## Cost: each mock set is a new context

The set of `@MockBean` / `@SpyBean` declarations is part of the **context cache key**. Two otherwise identical test classes with different mock sets load two contexts.

```java
// context A
@WebMvcTest(OrderController.class)
class OrderControllerTest {
  @MockBean private OrderService orderService;
}

// context A — reused
@WebMvcTest(OrderController.class)
class OrderControllerValidationTest {
  @MockBean private OrderService orderService;
}

// context B — extra mock means a second context
@WebMvcTest(OrderController.class)
class OrderControllerAuditTest {
  @MockBean private OrderService orderService;
  @MockBean private AuditService auditService;
}
```

Keep the mock set identical across test classes in the same slice where you can. More in [context-caching.md](context-caching.md).

## Mockito version caveat on this stack

`@MockBean` is implemented by Boot's `MockitoPostProcessor`, which touches Mockito internals (`MockUtil`, `MockCreationSettings`, `MockReset`). Boot 2.0.3 was built against Mockito 2.15.0 and the project overrides it to 3.12.4.

That combination works in practice, but it is the one place where a Mockito bump can break the test suite in a confusing way (`NoSuchMethodError`, `AbstractMethodError`, or mocks that are not injected). If that happens:

1. Confirm `byte-buddy` / `byte-buddy-agent` resolve to **1.11.13** and `objenesis` to **3.2** (see [dependencies.md](dependencies.md))
2. Run one `@MockBean` test and one `@SpyBean` test in isolation to confirm it is the post-processor and not your test
3. Step Mockito back (3.12.4 → 2.28.2) only after the classpath is verified

Pure `@Mock` unit tests never hit this path, which is another reason to keep the Spring-context tests to a minimum.

## Key points

1. **`@MockBean`, not `@MockitoBean`** on this stack
2. Always pair with `@ExtendWith(SpringExtension.class)`
3. Mock **every** collaborator the slice needs, or the context will not start
4. Prefer `@Mock` with no context when the class is constructible by hand
5. Mock sets change the context cache key — keep them uniform
6. For spies, stub with `willDoNothing().given(spy)` rather than `given(spy.call())`
