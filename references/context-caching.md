# Context Caching (Boot 2.0.3)

The single biggest lever on suite runtime once there is more than a handful of Spring tests. Spring's TestContext framework caches application contexts and reuses them across test classes — but only when the configuration matches exactly.

## How the cache works

Each test class produces a configuration "key". Two classes with the same key share one context; a different key loads another one and keeps both in memory.

### What is part of the cache key

- `@ContextConfiguration` classes / locations (which `@SpringBootTest` and every slice generate internally)
- The slice annotation and its attributes (`@WebMvcTest(OrderController.class)` differs from `@WebMvcTest`)
- `@TestPropertySource` locations and inline properties
- `@ActiveProfiles`
- `@WebAppConfiguration`
- **The set of `@MockBean` / `@SpyBean` declarations**
- `@TestConfiguration` / `@Import` classes
- `@DirtiesContext` (evicts the context after the test)
- `ContextCustomizer`s, including the one Boot adds for `webEnvironment`

## Same key — context reused

```java
@ExtendWith(SpringExtension.class)
@WebMvcTest(OrderController.class)
class OrderControllerTest {
  @MockBean private OrderService orderService;
}

@ExtendWith(SpringExtension.class)
@WebMvcTest(OrderController.class)
class OrderControllerValidationTest {
  @MockBean private OrderService orderService;
}
// one context, loaded once
```

## Different key — second context

```java
@ExtendWith(SpringExtension.class)
@WebMvcTest(OrderController.class)
@ActiveProfiles("test")
class OrderControllerTest { }

@ExtendWith(SpringExtension.class)
@WebMvcTest(OrderController.class)
@ActiveProfiles("integration")
class OrderControllerIntegrationTest { }
// two contexts
```

The same happens for an extra `@MockBean`, one inline property, or one more controller in the slice argument.

## Seeing what is happening

There is no reliable actuator metric for this on Boot 2.0.3 — use the framework's own logging:

```properties
# src/test/resources/logback-test.xml or application-test.properties
logging.level.org.springframework.test.context.cache=DEBUG
```

The log then reports, after each context load:

```
Spring test ApplicationContext cache statistics: [DefaultContextCache@... size = 4,
maxSize = 32, parentContextCount = 0, hitCount = 17, missCount = 4]
```

`missCount` is the number of contexts built. If it is close to the number of test classes, the suite is reloading the context for nearly every class and that is where the runtime goes.

Cache size is capped (default 32, LRU eviction):

```properties
spring.test.context.cache.maxSize=16
```

Lowering it reduces memory pressure; raising it helps a suite with many distinct configurations, at the cost of heap. On a JDK 8 build with a modest `-Xmx` this matters — context thrashing and `OutOfMemoryError` in the test JVM are two sides of the same problem.

## Reducing the number of contexts

### 1. Keep the mock set uniform within a slice

```java
// base class, one mock set, one context for every subclass
@ExtendWith(SpringExtension.class)
@WebMvcTest(OrderController.class)
abstract class OrderControllerTestBase {
  @Autowired protected MockMvc mvc;
  @MockBean protected OrderService orderService;
  @MockBean protected AuditService auditService;
}

class OrderControllerReadTest extends OrderControllerTestBase { }
class OrderControllerWriteTest extends OrderControllerTestBase { }
```

An unused `@MockBean` in the base class costs nothing at runtime and saves a whole context load. Strict-stub complaints do not apply to `@MockBean` — it is reset, not verified.

### 2. Minimise `@TestPropertySource` variations

```java
// bad: two contexts
@TestPropertySource(properties = "app.feature-x=true")
class FeatureXTest { }

@TestPropertySource(properties = "app.feature-y=true")
class FeatureYTest { }

// better: one context
@TestPropertySource(properties = {"app.feature-x=true", "app.feature-y=true"})
class FeaturesTest { }
```

Better still, put test configuration in a single `application-test.properties` plus `@ActiveProfiles("test")` and stop annotating individual classes.

### 3. Group tests by configuration

```
src/test/java/
  unit/           # no Spring context at all — the bulk of the suite
  web/            # @WebMvcTest, one shared mock set per controller
  repository/     # @DataJpaTest
  integration/    # @SpringBootTest, as few as possible
```

### 4. Use `@DirtiesContext` only when the context is genuinely damaged

```java
@Test
@DirtiesContext        // forces a rebuild for every later class with this key
void testThatMutatesSingletonState() { }
```

Each `@DirtiesContext` evicts a context that other classes would have reused. The usual fix is to clean up the data or reset the bean in `@AfterEach` instead.

### 5. Prefer the narrowest slice, or no context

The cheapest context is the one that is never created. A service test that runs with `@ExtendWith(MockitoExtension.class)` costs microseconds; the same test as `@SpringBootTest` costs seconds. See [mockito-unit-tests.md](mockito-unit-tests.md).

### 6. Watch `webEnvironment`

```java
@SpringBootTest                                            // MOCK — no server
@SpringBootTest(webEnvironment = WebEnvironment.RANDOM_PORT) // starts a server
```

These are different keys **and** the second one starts Tomcat. Use `RANDOM_PORT` only when the test really goes over HTTP via `TestRestTemplate`.

## Static state that survives context reuse

A reused context means singletons and static state survive between test classes. Two consequences:

- Data written by one test class is visible to the next unless the test is transactional (`@DataJpaTest` rolls back; `@SpringBootTest` does not, unless you add `@Transactional`)
- A `static` field in a test class or a mutated singleton bean leaks across classes and produces order-dependent failures

Run the suite in a different order occasionally to catch these:

```xml
<plugin>
    <artifactId>maven-surefire-plugin</artifactId>
    <version>2.22.2</version>
    <configuration>
        <runOrder>random</runOrder>
    </configuration>
</plugin>
```

## Parallel execution — not on this stack

JUnit 5 parallel execution (`junit.jupiter.execution.parallel.enabled`) exists in 5.9.3, but combining it with a shared Spring context, a shared H2 instance and `@MockBean` reset semantics is a reliable way to get flaky tests. Keep the suite sequential and spend the effort on having fewer contexts instead.

## Checklist

- [ ] `logging.level.org.springframework.test.context.cache=DEBUG` enabled once, `missCount` inspected
- [ ] Number of distinct contexts is a small fraction of the number of test classes
- [ ] Mock sets identical within each slice (base classes where it helps)
- [ ] No `@DirtiesContext` without a written reason
- [ ] `@TestPropertySource` replaced by a test profile where possible
- [ ] `RANDOM_PORT` used only by tests that make real HTTP calls
- [ ] Unit tests (no context) are the majority of the suite
