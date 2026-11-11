# Test Slices Overview (Boot 2.0.3)

Selecting the right test level on Spring Boot 2.0.3. Every slice annotation below exists in this version — nothing newer is listed.

## Decision matrix

| Approach | Use when | Loads | Speed |
|---|---|---|---|
| **No annotation** (plain JUnit + Mockito) | pure business logic | nothing | fastest |
| `@JsonTest` | JSON serialization | `ObjectMapper` only | fastest slice |
| `@WebMvcTest` | controller + HTTP layer | controllers, MVC, Jackson, security filters | fast |
| `@DataJpaTest` | repository queries | repositories, JPA, embedded DataSource | fast |
| `@JdbcTest` | plain JDBC / `JdbcTemplate` | DataSource, `JdbcTemplate` | fast |
| `@RestClientTest` | code calling an external HTTP API | `RestTemplateBuilder`, `MockRestServiceServer` | fast |
| `@JooqTest` | jOOQ `DSLContext` | DataSource, jOOQ | fast |
| `@DataMongoTest` | MongoDB repositories | repositories, `MongoTemplate` | medium |
| `@DataRedisTest` | Redis repositories | repositories, Redis templates | medium |
| `@DataLdapTest` | LDAP repositories | repositories, `LdapTemplate` | medium |
| `@DataNeo4jTest` | Neo4j repositories | repositories, session factory | medium |
| `@WebFluxTest` | reactive controllers | controllers, WebFlux, `WebTestClient` | fast |
| `@SpringBootTest` | the whole application | full context, optional embedded server | slow |

Not available on 2.0.3: `@DataJdbcTest` (Boot 2.1), `@WebServiceClientTest` (Boot 2.1). See [boot2-constraints.md](boot2-constraints.md).

**Every slice above needs `@ExtendWith(SpringExtension.class)`** on this version, including `@SpringBootTest`.

## No annotation — plain unit test

```java
class PriceCalculatorTest {

  private final PriceCalculator calculator = new PriceCalculator();

  @Test
  void shouldApplyDiscount() {
    BigDecimal result = calculator.applyDiscount(new BigDecimal("100.00"), 0.1);

    assertThat(result).isEqualByComparingTo(new BigDecimal("90.00"));
  }
}
```

**When**: pure logic; dependencies either absent or injectable as mocks. This is where most of the suite should live → [mockito-unit-tests.md](mockito-unit-tests.md).

## @WebMvcTest

```java
@ExtendWith(SpringExtension.class)
@WebMvcTest(OrderController.class)
class OrderControllerTest {

  @Autowired
  private MockMvc mvc;

  @MockBean
  private OrderService orderService;
}
```

**When**: request mapping, status codes, validation errors, JSON shape, filters, exception handlers.

**What you get**: `MockMvc`, Jackson `ObjectMapper`, `@ControllerAdvice`, Spring Security filters (if `spring-security-test` is on the classpath), bean validation.

Details: [webmvctest.md](webmvctest.md), API: [mockmvc-classic.md](mockmvc-classic.md).

## @DataJpaTest

```java
@ExtendWith(SpringExtension.class)
@DataJpaTest
class OrderRepositoryTest {

  @Autowired
  private OrderRepository orderRepository;

  @Autowired
  private TestEntityManager entityManager;
}
```

**When**: custom `@Query` methods, entity mapping, cascades, lazy loading, pagination.

**What you get**: repository beans, `EntityManager`, `TestEntityManager`, transaction per test with rollback, embedded H2 in place of the real DataSource.

Details: [datajpatest.md](datajpatest.md).

## @JdbcTest

For code that uses `JdbcTemplate` directly, with no JPA:

```java
@ExtendWith(SpringExtension.class)
@JdbcTest
@Sql("/schema-orders.sql")
class OrderDaoTest {

  @Autowired
  private JdbcTemplate jdbcTemplate;

  private OrderDao orderDao;

  @BeforeEach
  void setUp() {
    orderDao = new OrderDao(jdbcTemplate);
  }
}
```

`@JdbcTest` does not scan for your DAO beans — construct the DAO yourself, or add it with `@Import`/`@TestConfiguration`.

## @RestClientTest

```java
@ExtendWith(SpringExtension.class)
@RestClientTest(WeatherService.class)
class WeatherServiceTest {

  @Autowired
  private WeatherService weatherService;

  @Autowired
  private MockRestServiceServer server;
}
```

**When**: the class under test talks to an external API through `RestTemplate` built from `RestTemplateBuilder`.

Details: [restclienttest.md](restclienttest.md).

## @JsonTest

```java
@ExtendWith(SpringExtension.class)
@JsonTest
class OrderJsonTest {

  @Autowired
  private JacksonTester<Order> json;

  @Test
  void shouldSerializeOrder() throws Exception {
    Order order = OrderFixtures.pendingOrder();

    JsonContent<Object> content = json.write(order);

    assertThat(content).hasJsonPathStringValue("$.status");
    assertThat(content).extractingJsonPathStringValue("$.status").isEqualTo("PENDING");
  }

  @Test
  void shouldDeserializeOrder() throws Exception {
    String payload = "{\"id\":1,\"status\":\"PENDING\"}";

    ObjectContent<Order> parsed = json.parse(payload);

    assertThat(parsed.getObject().getStatus()).isEqualTo("PENDING");
  }
}
```

**When**: custom serializers/deserializers, `@JsonView`, date formats, field naming strategies.

Note the `throws Exception` — `JacksonTester` methods throw `IOException`.

## @SpringBootTest

```java
@ExtendWith(SpringExtension.class)
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
class OrderIntegrationTest {

  @Autowired
  private TestRestTemplate restTemplate;

  @MockBean
  private PaymentGateway paymentGateway;

  @Test
  void shouldCreateOrderEndToEnd() {
    given(paymentGateway.charge(any(BigDecimal.class))).willReturn("txn-1");

    ResponseEntity<Order> response = restTemplate.postForEntity(
        "/orders", new OrderRequest("Laptop", 1), Order.class);

    assertThat(response.getStatusCode()).isEqualTo(HttpStatus.CREATED);
    assertThat(response.getBody().getStatus()).isEqualTo("PAID");
  }
}
```

`TestRestTemplate` is the HTTP client for this stack — `RestTestClient` (Boot 4) and `WebTestClient` bound to a controller (Spring 5.3) do not exist here.

`webEnvironment` options on 2.0.3: `MOCK` (default, no server), `RANDOM_PORT`, `DEFINED_PORT`, `NONE`.

**When**: the interaction between layers is the thing under test — security filters plus controller plus repository together. Keep the count of these low; they dominate suite runtime.

## Common mistakes

1. **Forgetting `@ExtendWith(SpringExtension.class)`** — annotations ignored, `@Autowired` null. The number one Boot 2.0.x + Jupiter failure
2. **`@SpringBootTest` for everything** — a slow suite nobody runs locally
3. **`@WebMvcTest` without mocking every collaborator** — context startup failure, `NoSuchBeanDefinitionException`
4. **`@MockBean` in a `@DataJpaTest`** — defeats the purpose; you want the real repository
5. **Several slices on one class** — `@WebMvcTest` + `@DataJpaTest` together is not a thing; split the class
6. **Asserting business rules through MockMvc** — test the rule in a unit test, test the HTTP contract in the slice

## Java 8 notes for test code

```java
// No records: plain classes for request/response fixtures
public class OrderRequest {
  private final String product;
  private final int quantity;

  public OrderRequest(String product, int quantity) {
    this.product = product;
    this.quantity = quantity;
  }
  // getters, equals, hashCode
}

// No text blocks: escaped strings for small payloads
String json = "{\"product\":\"Laptop\",\"quantity\":2}";

// Larger payloads: a file under src/test/resources
String json = StreamUtils.copyToString(
    new ClassPathResource("orders/create-request.json").getInputStream(),
    StandardCharsets.UTF_8);

// No List.of / getFirst
List<Order> orders = Arrays.asList(order1, order2);
assertThat(orders.get(0).getStatus()).isEqualTo("NEW");
```

## Dependencies by slice

On 2.0.3 there is a single test starter — the modular `spring-boot-starter-*-test` artifacts are a Boot 4 feature:

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-test</artifactId>
    <scope>test</scope>
</dependency>
```

Slice-specific extras:

- `@DataJpaTest` → `spring-boot-starter-data-jpa` (compile) and an embedded DB, normally `com.h2database:h2` (test)
- `@WebMvcTest` with `@WithMockUser` → `spring-security-test` (test); it is managed by the Boot BOM but is not part of the baseline dependency set, so add it deliberately
- `@JsonTest`, `@RestClientTest`, `@JdbcTest` → nothing beyond the starter

Full list and pinning rules: [dependencies.md](dependencies.md).
