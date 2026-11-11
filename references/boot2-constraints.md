# Boot 2.0.3 / Java 8 Constraints

The complete list of modern testing APIs that are **not** available on this stack, with the replacement to use instead. Consult this before copying any example from current Spring Boot documentation, blog posts or newer projects — most of it will not compile here.

## Spring Boot / Spring Framework

| Modern API | Available from | Replacement on Boot 2.0.3 |
|---|---|---|
| `@MockitoBean`, `@MockitoSpyBean` | Spring Framework 6.2 / Boot 3.4 | `@MockBean`, `@SpyBean` from `org.springframework.boot.test.mock.mockito` |
| `MockMvcTester` + AssertJ web assertions | Boot 3.2 | `MockMvc.perform(...).andExpect(...)` |
| `RestTestClient` | Boot 4 | `TestRestTemplate` with `@SpringBootTest(webEnvironment = RANDOM_PORT)` |
| `WebTestClient` for MVC | Spring Framework 5.3 (`bindToController`) | `MockMvc` |
| `RestClient` and `RestClient.Builder` | Spring Framework 6.1 | `RestTemplate` / `RestTemplateBuilder` |
| `@ServiceConnection` | Boot 3.1 | not applicable; this stack has no Testcontainers |
| `@DynamicPropertySource` | Spring Framework 5.2.5 | `ApplicationContextInitializer` + `TestPropertyValues.of(...).applyTo(ctx)`, or `@TestPropertySource` |
| `@DataJdbcTest` | Boot 2.1 | `@JdbcTest` |
| `@WebServiceClientTest` | Boot 2.1 | plain unit test around the client |
| `@MockBean` on a prototype-scoped bean | Spring Framework 7 | redesign the test; not supported |
| `@SpringBootTest` meta-annotated with `@ExtendWith(SpringExtension.class)` | Boot 2.1 | declare `@ExtendWith(SpringExtension.class)` yourself on every Spring test |
| `ApplicationContextRunner` with `withPropertyValues` fluency for slices | 2.0 has it | available, fine to use |
| Context pausing / automatic context eviction | Spring Framework 7 | only `@DirtiesContext` and cache-key discipline |

### The extension is not optional

```java
// Boot 2.0.3 — correct
@ExtendWith(SpringExtension.class)
@SpringBootTest
class OrderIntegrationTest { }

// Boot 2.0.3 — silently broken: no context is loaded, @Autowired stays null
@SpringBootTest
class OrderIntegrationTest { }

// JUnit 4 only — no effect under Jupiter
@RunWith(SpringRunner.class)
@SpringBootTest
class OrderIntegrationTest { }
```

Slices that need it just the same: `@WebMvcTest`, `@DataJpaTest`, `@JdbcTest`, `@JsonTest`, `@RestClientTest`, `@WebFluxTest`, `@DataMongoTest`, `@DataRedisTest`, `@DataLdapTest`, `@DataNeo4jTest`, `@JooqTest`.

## Mockito 3.12.4

| Not available | Why | Replacement |
|---|---|---|
| `mockStatic(...)` | requires `mockito-inline`, which is not in the dependency set | extract the static call into an injectable collaborator |
| `mockConstruction(...)` | same | inject a factory |
| `@MockitoSettings(strictness = ...)` on a class without `MockitoExtension` | needs the extension | add `@ExtendWith(MockitoExtension.class)` |
| `Mockito.mock(Foo.class, withSettings().defaultAnswer(...))` on final classes | inline mock maker absent | make the class non-final or extract an interface |

Available and preferred: `@Mock`, `@Spy`, `@Captor`, `@InjectMocks`, `MockitoExtension`, `BDDMockito.given(...).willReturn(...)`, `ArgumentCaptor`, `verify(mock, times(n))`, `lenient()`, `MockitoAnnotations.openMocks(this)`.

## AssertJ 3.9.1

The project is pinned to the version Boot 2.0.3 manages, which predates a lot of commonly used API.

| Not available | Added in | Use instead |
|---|---|---|
| `usingRecursiveComparison()` | 3.12 | `isEqualToComparingFieldByFieldRecursively(expected)` |
| `usingRecursiveFieldByFieldElementComparatorIgnoringFields(...)` | 3.17+ | `usingElementComparatorIgnoringFields("id", "createdAt")` |
| `isNotBlank()` / `isBlank()` | 3.11 | `isNotEmpty()`, or `assertThat(s.trim()).isNotEmpty()` |
| `hasSizeGreaterThan(n)`, `hasSizeLessThan(n)`, `hasSizeBetween(a, b)` | 3.12 | `assertThat(list.size()).isGreaterThan(n)` |
| `satisfiesExactly(...)`, `satisfiesExactlyInAnyOrder(...)` | 3.19 | `allSatisfy(...)`, or assert elements by index |
| `asInstanceOf(InstanceOfAssertFactories.LIST)` | 3.13 | cast, or `extracting(...)` |
| `assertThatNoException()` | 3.17 | `assertThatCode(...).doesNotThrowAnyException()` |
| `extracting(Function...)` on a single object | 3.11+ | `extracting("field1", "field2")` with field names |
| `containsExactlyInAnyOrderElementsOf` on `Map` keys | varies | `assertThat(map.keySet()).containsExactlyInAnyOrder(...)` |

Available in 3.9.1 and used throughout this skill: `assertThat`, `assertThatThrownBy`, `assertThatExceptionOfType`, `assertThatCode`, `satisfies(Consumer)`, `allSatisfy`, `allMatch` / `anyMatch` / `noneMatch`, `extracting` on iterables (lambda or field name), `flatExtracting`, `filteredOn`, `tuple`, `entry`, `hasEntrySatisfying`, `usingRecursiveFieldByFieldElementComparator`, `usingElementComparatorIgnoringFields`, `SoftAssertions`, `isCloseTo(temporal, within(...))`, `hasValueSatisfying`, custom `AbstractAssert` subclasses.

## JUnit 5.9.3

Everything the suite needs exists, but these are newer than 5.9.3:

| Not available | Added in |
|---|---|
| `@AutoClose` | 5.11 |
| `@FieldSource` for `@ParameterizedTest` | 5.11 |
| `@ParameterizedClass` / `@ClassTemplate` | 5.13 |
| `TempDirFactory` | 5.10 |
| JUnit 6 unified versioning / Kotlin 2.2 baseline | 6.0 |

Available: `@Test`, `@Nested`, `@DisplayName`, `@DisplayNameGeneration`, `@Tag`, `@TempDir` (with `cleanup` mode), `@ParameterizedTest` with `@ValueSource` / `@CsvSource` / `@MethodSource` / `@EnumSource`, `@RepeatedTest`, `assertAll`, `assertThrows`, `assertThrowsExactly`, `assertInstanceOf`, `assertTimeout`, `@EnabledOnJre`, `@Disabled`, `Assumptions`.

## Java 8 language level

Every snippet must compile under `-source 1.8`. Common things to rewrite when adapting modern examples:

| Java 9+ | Java 8 |
|---|---|
| `var order = new Order()` | `Order order = new Order();` |
| text blocks `"""{ "a": 1 }"""` | escaped string, or a `src/test/resources/*.json` file read with `ClassPathResource` |
| `record OrderRequest(String p, int q) {}` | a class with constructor, getters, `equals`/`hashCode` |
| `List.of(a, b)` / `Map.of(k, v)` | `Arrays.asList(a, b)`, `Collections.singletonMap(k, v)` |
| `list.getFirst()` / `getLast()` / `reversed()` | `list.get(0)`, `list.get(list.size() - 1)`, `Collections.reverse(copy)` |
| `switch` pattern matching, sealed types | `if (x instanceof Foo)` with a cast |
| `Stream.toList()` | `stream.collect(Collectors.toList())` |
| `Optional.isEmpty()` | `!optional.isPresent()` |
| `"x".repeat(3)`, `String.strip()` | `StringUtils` helpers or manual code |

Java 8 **does** have: lambdas, method references, `Optional`, streams, `java.time`, default methods, `@FunctionalInterface`, repeatable annotations — all of which JUnit 5 and AssertJ rely on.

## When an example cannot be ported

If a technique genuinely requires a newer version — mocking a static utility, container-backed database tests, recursive comparison on deep graphs — do not fake it and do not add a dependency outside [dependencies.md](dependencies.md). Say which version it needs, then offer the stack-compatible alternative:

- static utility → extract an interface and inject it
- real-database fidelity → `@JdbcTest` / `@DataJpaTest` on H2 plus a manual SQL script, and accept the parity gap
- deep object comparison → `isEqualToComparingFieldByFieldRecursively`, or assert the fields that carry meaning
