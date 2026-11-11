# @DataJpaTest (Boot 2.0.3)

Testing JPA repositories with an isolated data-layer slice and an embedded database.

## Basic structure

```java
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.ExtendWith;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.autoconfigure.orm.jpa.DataJpaTest;
import org.springframework.boot.test.autoconfigure.orm.jpa.TestEntityManager;
import org.springframework.test.context.junit.jupiter.SpringExtension;

import static org.assertj.core.api.Assertions.assertThat;

@ExtendWith(SpringExtension.class)      // REQUIRED on Boot 2.0.3
@DataJpaTest
class OrderRepositoryTest {

  @Autowired
  private OrderRepository orderRepository;

  @Autowired
  private TestEntityManager entityManager;
}
```

## What gets loaded

- Spring Data repository beans
- `EntityManager` and `TestEntityManager`
- An **embedded DataSource** replacing the configured one (H2 by default — `com.h2database:h2` on the test classpath)
- Transaction manager; each test runs in a transaction that **rolls back** afterwards

Not loaded: services, controllers, the web layer, the production DataSource.

### The database here is H2, not the production DB

This stack has no Testcontainers (see [boot2-constraints.md](boot2-constraints.md)), so repository tests run on embedded H2. That is fast and hermetic, but it will not catch vendor-specific behavior: native queries with MySQL/Oracle syntax, dialect-specific functions, collation and case sensitivity, lock semantics, or column-type edge cases.

Practical split:

- **Derived queries, mappings, cascades, pagination** → `@DataJpaTest` on H2, high value
- **Native queries with vendor SQL** → keep the SQL in a unit-tested method where possible, and verify against the real database manually or in a separate, manually-run test
- Add H2's compatibility mode when the schema is generated from vendor DDL:

```properties
# src/test/resources/application-test.properties
spring.datasource.url=jdbc:h2:mem:testdb;MODE=MySQL;DB_CLOSE_DELAY=-1
spring.jpa.hibernate.ddl-auto=create-drop
```

```java
@ExtendWith(SpringExtension.class)
@DataJpaTest
@ActiveProfiles("test")
class OrderRepositoryTest { }
```

Compatibility mode narrows the gap; it does not close it. Say so in review rather than implying H2 proves the query works in production.

## Keeping the real database

When a test must run against the configured DataSource (a shared dev schema, for example):

```java
@ExtendWith(SpringExtension.class)
@DataJpaTest
@AutoConfigureTestDatabase(replace = AutoConfigureTestDatabase.Replace.NONE)
class OrderRepositoryRealDbTest { }
```

This couples the suite to an external database — it will fail on a machine without access. Use it deliberately, keep such tests tagged (`@Tag("db")`) and excluded from the default build.

## Testing derived and @Query methods

```java
@Test
void shouldFindOrdersByStatus() {
  Order pending = new Order("PENDING");
  Order completed = new Order("COMPLETED");
  entityManager.persist(pending);
  entityManager.persist(completed);
  entityManager.flush();

  List<Order> pendingOrders = orderRepository.findByStatus("PENDING");

  assertThat(pendingOrders).hasSize(1);
  assertThat(pendingOrders.get(0).getStatus()).isEqualTo("PENDING");
}
```

```java
@Query("SELECT o FROM Order o WHERE o.createdAt > :date AND o.status = :status")
List<Order> findRecentByStatus(@Param("date") LocalDateTime date, @Param("status") String status);
```

```java
@Test
void shouldFindRecentOrders() {
  Order old = new Order("PENDING");
  old.setCreatedAt(LocalDateTime.now().minusDays(10));
  Order recent = new Order("PENDING");
  recent.setCreatedAt(LocalDateTime.now().minusHours(1));
  entityManager.persist(old);
  entityManager.persist(recent);
  entityManager.flush();

  List<Order> found = orderRepository.findRecentByStatus(
      LocalDateTime.now().minusDays(1), "PENDING");

  assertThat(found).hasSize(1);
  assertThat(found.get(0).getId()).isEqualTo(recent.getId());
}
```

## Testing pagination

```java
@Test
void shouldReturnPagedResults() {
  for (int i = 0; i < 20; i++) {
    entityManager.persist(new Order("PENDING"));
  }
  entityManager.flush();

  Page<Order> page = orderRepository.findByStatus("PENDING", PageRequest.of(0, 10));

  assertThat(page.getContent()).hasSize(10);
  assertThat(page.getTotalElements()).isEqualTo(20L);
  assertThat(page.getTotalPages()).isEqualTo(2);
}
```

Sorting (Spring Data 2.0 API): `PageRequest.of(0, 10, Sort.by(Sort.Direction.DESC, "createdAt"))`.

## Testing lazy loading

```java
@Test
void shouldLazyLoadOrderItems() {
  Order order = new Order("PENDING");
  order.addItem(new OrderItem("Laptop", 2));
  entityManager.persist(order);
  entityManager.flush();
  entityManager.clear();                 // detach everything

  Optional<Order> found = orderRepository.findById(order.getId());

  assertThat(found).isPresent();
  assertThat(found.get().getItems()).hasSize(1);          // triggers the lazy load
  assertThat(found.get().getItems().get(0).getProduct()).isEqualTo("Laptop");
}
```

The test transaction keeps the session open, so a lazy collection resolves here even when it would throw `LazyInitializationException` in production. To reproduce the production failure, read the entity in a method that is not transactional, or assert on the fetch plan of the query instead.

## Testing cascades

```java
@Test
void shouldCascadeDelete() {
  Order order = new Order("PENDING");
  order.addItem(new OrderItem("Laptop", 2));
  entityManager.persist(order);
  entityManager.flush();
  Long itemId = order.getItems().get(0).getId();

  orderRepository.delete(order);
  entityManager.flush();

  assertThat(entityManager.find(OrderItem.class, itemId)).isNull();
}
```

## Testing a native query

```java
@Test
void shouldCalculatePendingTotal() {
  entityManager.persist(new Order("PENDING", new BigDecimal("100.00")));
  entityManager.persist(new Order("PENDING", new BigDecimal("200.00")));
  entityManager.flush();

  BigDecimal total = orderRepository.calculatePendingTotal();

  assertThat(total).isEqualByComparingTo(new BigDecimal("300.00"));
}
```

Use `isEqualByComparingTo` for `BigDecimal` — `isEqualTo` compares scale, so `300.00` and `300` fail.

## Seeding with SQL

```java
@ExtendWith(SpringExtension.class)
@DataJpaTest
@Sql("/data/orders.sql")                      // before each test method
@SqlGroup({
  @Sql(scripts = "/data/orders.sql"),
  @Sql(scripts = "/data/cleanup.sql", executionPhase = Sql.ExecutionPhase.AFTER_TEST_METHOD)
})
class OrderRepositoryWithSeedTest { }
```

Prefer `TestEntityManager` for a handful of rows (it keeps the expected state next to the assertion) and SQL scripts when the fixture is large or must bypass JPA.

## Transaction behavior

Tests are `@Transactional` and roll back by default. Two consequences worth knowing:

```java
@Test
@Rollback(false)        // keep the data (rarely what you want)
void shouldPersistData() {
  orderRepository.save(new Order("PENDING"));
}
```

- `save()` inside the test transaction may not hit the database until `flush()` — always `entityManager.flush()` before asserting that a constraint or trigger fired
- `entityManager.clear()` after `flush()` is how you prove a read actually came from the database and not the first-level cache

```java
@Test
void shouldRejectDuplicateOrderNumber() {
  entityManager.persist(new Order("ORD-1"));
  entityManager.flush();

  entityManager.persist(new Order("ORD-1"));

  assertThatThrownBy(() -> entityManager.flush())
      .isInstanceOf(PersistenceException.class);
}
```

## Key points

1. **`@ExtendWith(SpringExtension.class)` first**
2. **`TestEntityManager` for setup**, the repository for the behavior under test
3. **`flush()` after `persist()`** or no SQL is sent
4. **`clear()` to detach** before testing lazy loading or a real re-read
5. **H2 is not the production database** — be explicit about what the test does and does not prove
6. **`isEqualByComparingTo` for `BigDecimal`**
7. **No `@MockBean` here** — if you are mocking the repository, this is the wrong slice
8. **Java 8**: `for` loops instead of `IntStream` is fine, `list.get(0)` not `getFirst()`, no `var`
