---
name: spring-boot-testing
description: 'Expert Spring Boot testing specialist for legacy stacks: Java 8, Spring Boot 2.0.3, JUnit 5 (Jupiter), Mockito 3, AssertJ 3.9.1, MockMvc and JaCoCo. Picks the narrowest test that gives confidence and never silently upgrades a managed dependency.'
---

# Spring Boot Testing (Java 8 / Spring Boot 2.0.3)

Expert guidance for testing Spring Boot applications on a **frozen legacy stack**. Every example in this skill compiles on **Java 8** and runs against **Spring Boot 2.0.3.RELEASE**.

## Target stack — do not deviate

| Component | Version | Source |
|---|---|---|
| Java | 1.8 | `maven-toolchains-plugin` forks the test JVM to JDK 1.8 |
| Spring Boot | 2.0.3.RELEASE | project POM |
| Spring Framework | 5.0.7.RELEASE | Boot 2.0.3 BOM |
| Spring Security | 5.0.6.RELEASE | Boot 2.0.3 BOM |
| JUnit Jupiter | 5.9.3 (`junit-jupiter-api` + `-engine` + `-params`) | pinned, overrides Boot's 5.1.1 |
| JUnit Platform | 1.9.3 | transitive, keep in sync |
| Mockito | 3.12.4 (`mockito-core` + `mockito-junit-jupiter`) | pinned, overrides Boot's 2.15.0 |
| Byte Buddy | 1.11.13 | pinned, overrides Boot's 1.7.11 |
| Objenesis | 3.2 | transitive of Mockito 3 |
| AssertJ | 3.9.1 | Boot 2.0.3 BOM |
| Surefire | 2.22.2 | needed to run Jupiter at all |
| JaCoCo | 0.8.8 | coverage |

Full POM and verification checklist: [references/dependencies.md](references/dependencies.md).

**Never** raise Spring Framework, Spring Security or the JDBC driver to make a test pass. If a test needs an API that does not exist here, write the test differently.

## Hard constraints — APIs that DO NOT exist on this stack

Do not emit these, even though they are everywhere in modern Spring Boot material:

| Not available | Introduced in | Use instead |
|---|---|---|
| `@MockitoBean` / `@MockitoSpyBean` | Spring Framework 6.2 / Boot 3.4 | `@MockBean` / `@SpyBean` (`org.springframework.boot.test.mock.mockito`) |
| `MockMvcTester` (AssertJ-style MockMvc) | Boot 3.2 | classic `MockMvc.perform(...)` → [references/mockmvc-classic.md](references/mockmvc-classic.md) |
| `RestTestClient` | Boot 4 | `TestRestTemplate` |
| `@ServiceConnection` | Boot 3.1 | not applicable (no Testcontainers in this stack) |
| `@DynamicPropertySource` | Spring Framework 5.2.5 | `ApplicationContextInitializer` + `TestPropertyValues`, or `@TestPropertySource` |
| `RestClient` | Spring Framework 6.1 | `RestTemplate` |
| `@DataJdbcTest`, `@WebServiceClientTest` | Boot 2.1+ | `@JdbcTest` / plain unit test |
| `mockStatic` / `mockConstruction` | needs `mockito-inline` (not in the dependency set) | refactor the static call behind an injectable seam |
| `usingRecursiveComparison()`, `isNotBlank()`, `hasSizeGreaterThan()`, `satisfiesExactly()` | AssertJ 3.11–3.19 | see [references/assertj-basics.md](references/assertj-basics.md) for the 3.9.1-safe equivalents |
| Java 9+ syntax: `var`, text blocks, records, `List.of`, `getFirst()`, sequenced collections, switch patterns | Java 9–21 | Java 8: explicit types, `Arrays.asList`, `get(0)`, string concatenation |

## Mandatory boilerplate on this stack

Spring Boot only meta-annotates its test annotations with `@ExtendWith(SpringExtension.class)` **from 2.1.0**. On 2.0.3 every Spring-backed Jupiter test must declare the extension itself:

```java
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.ExtendWith;
import org.springframework.boot.test.autoconfigure.web.servlet.WebMvcTest;
import org.springframework.test.context.junit.jupiter.SpringExtension;

@ExtendWith(SpringExtension.class)   // REQUIRED on Boot 2.0.3
@WebMvcTest(OrderController.class)
class OrderControllerTest {
}
```

Forget it and the Spring annotations are silently ignored — no context, `@Autowired` fields stay null. `@RunWith(SpringRunner.class)` is the JUnit 4 equivalent and has **no effect** under Jupiter.

Plain Mockito unit tests use the Mockito extension instead, and load no Spring context at all:

```java
@ExtendWith(MockitoExtension.class)
class PriceCalculatorTest {
}
```

## Core principles

1. **Test pyramid**: unit (no Spring context) > slice > `@SpringBootTest`
2. **Narrowest slice that gives confidence** — a Spring context is the single biggest cost in this suite
3. **AssertJ `assertThat`** over JUnit assertions and Hamcrest, except inside `jsonPath(...)` where Hamcrest is the only option
4. **Pin, don't drift** — a test must never be the reason a production dependency version changes

## Which test?

| Scenario | Approach | Reference |
|---|---|---|
| Service / utility business logic | plain JUnit 5 + Mockito, no Spring | [references/mockito-unit-tests.md](references/mockito-unit-tests.md) |
| Controller + HTTP semantics | `@WebMvcTest` + classic MockMvc | [references/webmvctest.md](references/webmvctest.md) |
| Repository queries, entity mapping | `@DataJpaTest` + embedded H2 | [references/datajpatest.md](references/datajpatest.md) |
| REST client calling an external API | `@RestClientTest` + `MockRestServiceServer` | [references/restclienttest.md](references/restclienttest.md) |
| JSON (de)serialization | `@JsonTest` + `JacksonTester` | [references/test-slices-overview.md](references/test-slices-overview.md) |
| Replacing one bean in a Spring context | `@MockBean` / `@SpyBean` | [references/mockbean.md](references/mockbean.md) |
| Full application wiring | `@SpringBootTest` | [references/test-slices-overview.md](references/test-slices-overview.md) |

## Reference index

**Writing tests**

- [references/mockito-unit-tests.md](references/mockito-unit-tests.md) — JUnit 5 + Mockito 3.12.4 patterns, `@Mock`, `@InjectMocks`, `ArgumentCaptor`, strict stubs
- [references/mockbean.md](references/mockbean.md) — `@MockBean` / `@SpyBean` on Boot 2.0.3, and why they cost you a context
- [references/test-slices-overview.md](references/test-slices-overview.md) — the 11 slices that exist in 2.0.3 and how to choose
- [references/webmvctest.md](references/webmvctest.md) — web layer slice
- [references/mockmvc-classic.md](references/mockmvc-classic.md) — the full MockMvc API available in Spring 5.0.7
- [references/datajpatest.md](references/datajpatest.md) — data layer slice with H2
- [references/restclienttest.md](references/restclienttest.md) — `MockRestServiceServer`

**Assertions**

- [references/assertj-basics.md](references/assertj-basics.md) — scalars, strings, dates, exceptions, custom asserts (3.9.1 API only)
- [references/assertj-collections.md](references/assertj-collections.md) — lists, sets, maps, arrays (3.9.1 API only)

**Build and performance**

- [references/dependencies.md](references/dependencies.md) — POM blocks, version pinning rules, post-integration checklist
- [references/surefire-jacoco.md](references/surefire-jacoco.md) — Surefire 2.22.2 + toolchains, JaCoCo 0.8.8, `argLine` conflicts
- [references/context-caching.md](references/context-caching.md) — make the slice tests stop reloading contexts
- [references/boot2-constraints.md](references/boot2-constraints.md) — the full list of modern APIs that are unavailable, with the 2.0.3 replacement for each

## Quick decision tree

```
Is there a Spring bean you cannot construct by hand?
  No  → plain JUnit 5 + Mockito (@ExtendWith(MockitoExtension.class))
  Yes → you need a context; pick the narrowest slice:

        Controller / HTTP status / validation / JSON body?
          → @WebMvcTest + MockMvc, services as @MockBean

        Repository query, @Query, mapping, cascade?
          → @DataJpaTest (embedded H2, rolls back per test)

        Code that calls an external HTTP API via RestTemplate?
          → @RestClientTest + MockRestServiceServer

        Only Jackson mapping?
          → @JsonTest + JacksonTester

        Really the whole app (filters + security + DB together)?
          → @SpringBootTest(webEnvironment = RANDOM_PORT) + TestRestTemplate

  And in every one of those cases: @ExtendWith(SpringExtension.class) first.
```

## Testing best practices

### Code complexity assessment

When a method or class is too complex to test effectively:

1. **Analyze complexity** — if a single method needs more than 5–7 test cases, it is likely doing too much
2. **Recommend refactoring** — suggest breaking it into smaller, focused methods
3. **User decision** — if the user agrees, help identify extraction points
4. **Proceed anyway if asked** — if the user wants to keep the method as is, write the tests regardless

```java
// Before: hard to test
public Order processOrder(OrderRequest request) {
  // validation, discount, payment, inventory, notification... 50+ lines
}

// After: testable units
public Order processOrder(OrderRequest request) {
  validateOrder(request);
  Order order = createOrder(request);
  applyDiscount(order);
  processPayment(order);
  updateInventory(order);
  sendNotification(order);
  return order;
}
```

On this stack refactoring is often the *only* option: with no `mockito-inline`, a static call inside the method cannot be mocked at all — it has to be extracted behind an injectable collaborator.

### Avoid redundancy

Extract helper methods (`newPendingOrder()`, `stubFindById(...)`) for objects and stub setups used by several tests. There is no test-data generator in this dependency set, so hand-written builders and fixture factories are the pattern.

### Test organization with @DisplayName

```java
@Test
@DisplayName("Should calculate discount for VIP customer")
void shouldCalculateDiscountForVip() { }

@Test
@DisplayName("Should reject order when customer has insufficient credit")
void shouldRejectOrderForInsufficientCredit() { }
```

### Test coverage order

1. **Main scenario** — happy path
2. **Other paths** — alternative valid scenarios, edge cases
3. **Exceptions / errors** — invalid input, failure modes

### Test production scenarios

Write tests around real production scenarios, with realistic data, so the test doubles as documentation of actual behavior.

### Coverage goals

Aim for **80% line coverage** as a practical balance, measured with JaCoCo 0.8.8 (`prepare-agent` + `report`, report at `target/site/jacoco/index.html`).

**Prioritize:**

1. Business-critical paths (payment, order validation)
2. Complex algorithms (pricing, discounts)
3. Error handling (exceptions, edge cases)
4. Integration points (external APIs, repositories)

Coverage with no meaningful assertions is worthless — a test that only executes code is a regression in disguise.

## Reference documentation

- [Spring Boot 2.0.3 — Testing](https://docs.spring.io/spring-boot/docs/2.0.3.RELEASE/reference/html/boot-features-testing.html)
- [Spring Boot 2.0.3 — Dependency versions](https://docs.spring.io/spring-boot/docs/2.0.3.RELEASE/reference/html/appendix-dependency-versions.html)
- [Spring Framework 5.0.x — Testing](https://docs.spring.io/spring/docs/5.0.7.RELEASE/spring-framework-reference/testing.html)
- [JUnit 5.9.3 User Guide](https://docs.junit.org/5.9.3/user-guide/)
- [Mockito 3.12.4 javadoc](https://javadoc.io/doc/org.mockito/mockito-core/3.12.4/index.html)
- [AssertJ 3.9.1 javadoc](https://javadoc.io/doc/org.assertj/assertj-core/3.9.1/index.html)
- [JaCoCo Maven plugin](https://www.jacoco.org/jacoco/trunk/doc/maven.html)
