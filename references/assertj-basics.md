# AssertJ Basics (3.9.1)

Fluent assertions for scalars, strings, numbers, dates, `Optional`, exceptions and single objects.

**This stack is pinned to AssertJ 3.9.1** (the version Spring Boot 2.0.3 manages). Everything below exists in 3.9.1. Methods that appear in modern AssertJ documentation but were added later are listed in the [Not available](#not-available-in-391) section with their replacement — do not emit them, they will not compile.

## Object equality

```java
assertThat(order.getStatus()).isEqualTo("PENDING");
assertThat(order.getId()).isNotEqualTo(0L);
assertThat(order).isEqualTo(expectedOrder);       // needs equals()
assertThat(order).isNotNull();
assertThat(order).isSameAs(cachedOrder);
assertThat(nullOrder).isNull();
assertThat(order).isInstanceOf(PhysicalOrder.class);
```

## Field-by-field comparison (without equals)

```java
// compares declared fields, one level
assertThat(actualOrder).isEqualToComparingFieldByField(expectedOrder);

// recursive: nested objects compared by their fields too
assertThat(actualOrder).isEqualToComparingFieldByFieldRecursively(expectedOrder);

// ignore generated values
assertThat(actualOrder).isEqualToIgnoringGivenFields(expectedOrder, "id", "createdAt");

// compare only what matters
assertThat(actualOrder).isEqualToComparingOnlyGivenFields(expectedOrder, "status", "total");

// single field / property
assertThat(order).hasFieldOrPropertyWithValue("status", "PENDING");
```

`isEqualToComparingFieldByFieldRecursively` is the 3.9.1 equivalent of the modern `usingRecursiveComparison()`. It has no fluent configuration — to ignore fields, use `isEqualToIgnoringGivenFields` (non-recursive) or assert the fields individually.

## Strings

```java
assertThat(order.getDescription())
    .isEqualTo("Test Order")
    .isNotEmpty()
    .startsWith("Test")
    .endsWith("Order")
    .contains("Test")
    .doesNotContain("DELETED")
    .hasSize(10)
    .matches("[A-Za-z ]+")
    .isEqualToIgnoringCase("test order");

assertThat(order.getNotes()).isNullOrEmpty();
assertThat(code).containsOnlyDigits();
assertThat(name).isSubstringOf("Laptop Pro 16");
```

`isNotBlank()` / `isBlank()` do not exist in 3.9.1 — use `isNotEmpty()`, or `assertThat(value.trim()).isNotEmpty()` when whitespace must be rejected.

## Numbers

```java
assertThat(order.getQuantity())
    .isEqualTo(2)
    .isGreaterThan(0)
    .isLessThanOrEqualTo(10)
    .isBetween(1, 10)
    .isPositive()
    .isNotZero();

// BigDecimal: compare by value, not by scale
assertThat(order.getTotal()).isEqualByComparingTo(new BigDecimal("99.99"));
assertThat(order.getTotal()).isGreaterThan(BigDecimal.ZERO);

// doubles: always with a tolerance
assertThat(ratio).isCloseTo(0.33, within(0.01));
assertThat(ratio).isCloseTo(0.33, withinPercentage(1.0));
```

`assertThat(new BigDecimal("99.990")).isEqualTo(new BigDecimal("99.99"))` **fails** — scale is part of `equals`. Use `isEqualByComparingTo` for money.

## Booleans

```java
assertThat(order.isActive()).isTrue();
assertThat(order.isDeleted()).isFalse();
```

## Dates and times

```java
assertThat(order.getCreatedAt())
    .isEqualTo(LocalDateTime.of(2024, 1, 15, 10, 30))
    .isBefore(LocalDateTime.now())
    .isAfter(LocalDateTime.of(2024, 1, 1, 0, 0))
    .isCloseTo(LocalDateTime.now(), within(5, ChronoUnit.SECONDS));

assertThat(order.getDeliveryDate()).isEqualTo(LocalDate.parse("2024-02-01"));
assertThat(order.getCreatedAt()).isBetween(startOfDay, endOfDay);
assertThat(order.getCreatedAt().toLocalDate()).isToday();
```

Better than a tolerance: inject a fixed `Clock` (`Clock.fixed(Instant.parse("2024-01-15T10:30:00Z"), ZoneOffset.UTC)`) and assert exact equality. `within(5, ChronoUnit.SECONDS)` is a fallback for code that calls `LocalDateTime.now()` directly.

## Optional

```java
Optional<Order> maybeOrder = orderService.findById(1L);

assertThat(maybeOrder).isPresent();
assertThat(maybeOrder).contains(expectedOrder);
assertThat(maybeOrder).hasValueSatisfying(new Consumer<Order>() {
  @Override
  public void accept(Order order) {
    assertThat(order.getId()).isEqualTo(1L);
  }
});

assertThat(orderService.findById(999L)).isEmpty();
```

With a lambda (Java 8): `assertThat(maybeOrder).hasValueSatisfying(order -> assertThat(order.getId()).isEqualTo(1L));`

Note `Optional.isEmpty()` does not exist on Java 8 — in production code use `!optional.isPresent()`; the AssertJ `isEmpty()` above is an assertion, not the JDK method.

## Exceptions

```java
// most common form
assertThatThrownBy(() -> service.findById(999L))
    .isInstanceOf(OrderNotFoundException.class)
    .hasMessage("Order 999 not found")
    .hasMessageContaining("999")
    .hasNoCause();

// typed, with cause assertions
assertThatExceptionOfType(OrderProcessingException.class)
    .isThrownBy(() -> service.processOrder(invalidOrder))
    .withMessageStartingWith("Cannot process")
    .withCauseInstanceOf(ValidationException.class);

// assert nothing is thrown
assertThatCode(() -> service.processOrder(validOrder)).doesNotThrowAnyException();

// capture and assert separately
Throwable thrown = catchThrowable(() -> service.findById(999L));
assertThat(thrown).isInstanceOf(OrderNotFoundException.class);
assertThat(((OrderNotFoundException) thrown).getOrderId()).isEqualTo(999L);
```

`assertThatNoException()` (3.17) does not exist — use `assertThatCode(...).doesNotThrowAnyException()`.

JUnit's own `assertThrows` works too and returns the exception; pick one style per project and stay with it.

## satisfies

```java
assertThat(order).satisfies(o -> {
  assertThat(o.getId()).isPositive();
  assertThat(o.getStatus()).isNotEmpty();
  assertThat(o.getCreatedAt()).isNotNull();
});
```

## Soft assertions

Collect every failure instead of stopping at the first:

```java
@Test
void shouldPopulateAllOrderFields() {
  Order order = orderService.findById(1L);

  SoftAssertions.assertSoftly(softly -> {
    softly.assertThat(order.getId()).isEqualTo(1L);
    softly.assertThat(order.getStatus()).isEqualTo("PENDING");
    softly.assertThat(order.getTotal()).isEqualByComparingTo(new BigDecimal("99.99"));
    softly.assertThat(order.getItems()).isNotEmpty();
  });
}
```

Useful when asserting a mapper or DTO conversion, where knowing *all* the wrong fields at once saves a cycle. Do not use it to paper over a test that asserts too many unrelated things.

## Custom assertions

Worth writing once a domain check appears in more than three tests:

```java
public class OrderAssert extends AbstractAssert<OrderAssert, Order> {

  private OrderAssert(Order actual) {
    super(actual, OrderAssert.class);
  }

  public static OrderAssert assertThat(Order actual) {
    return new OrderAssert(actual);
  }

  public OrderAssert isPending() {
    isNotNull();
    if (!"PENDING".equals(actual.getStatus())) {
      failWithMessage("Expected order status to be PENDING but was <%s>", actual.getStatus());
    }
    return this;
  }

  public OrderAssert hasTotal(BigDecimal expected) {
    isNotNull();
    if (actual.getTotal() == null || actual.getTotal().compareTo(expected) != 0) {
      failWithMessage("Expected total <%s> but was <%s>", expected, actual.getTotal());
    }
    return this;
  }
}
```

```java
OrderAssert.assertThat(order)
    .isPending()
    .hasTotal(new BigDecimal("99.99"));
```

Use `compareTo` rather than `equals` for `BigDecimal` inside custom assertions, as above.

## Descriptions for better failure messages

```java
assertThat(order.getStatus())
    .as("status of order %d", order.getId())
    .isEqualTo("PENDING");
```

`as(...)` must come before the assertion it describes.

## Static imports

```java
import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;
import static org.assertj.core.api.Assertions.assertThatExceptionOfType;
import static org.assertj.core.api.Assertions.assertThatCode;
import static org.assertj.core.api.Assertions.catchThrowable;
import static org.assertj.core.api.Assertions.within;
import static org.assertj.core.api.Assertions.withinPercentage;
```

## Not available in 3.9.1

| Modern API | Added in | Use on 3.9.1 |
|---|---|---|
| `usingRecursiveComparison()` | 3.12 | `isEqualToComparingFieldByFieldRecursively(expected)` |
| `isNotBlank()` / `isBlank()` | 3.11 | `isNotEmpty()` / `assertThat(s.trim()).isEmpty()` |
| `assertThatNoException()` | 3.17 | `assertThatCode(...).doesNotThrowAnyException()` |
| `asInstanceOf(InstanceOfAssertFactories.*)` | 3.13 | cast explicitly |
| `extracting(Order::getId, Order::getStatus)` on a single object | 3.11+ | `extracting("id", "status")` with field names |
| `hasSizeGreaterThan(n)` on strings/collections | 3.12 | `assertThat(value.length()).isGreaterThan(n)` |
| `satisfiesAnyOf(...)` | 3.12 | `anyOf(Condition...)` or restructure the test |
| `isNotEqualByComparingTo` | 3.15 | `assertThat(a.compareTo(b)).isNotZero()` |

If a review comment or example uses one of these, port it rather than upgrading AssertJ — the version is managed by the Boot BOM and bumping it is out of scope for a test change.

## Key points

1. **3.9.1 API only** — check the table above before using anything you have seen in recent docs
2. **`isEqualByComparingTo` for `BigDecimal`**, always
3. **`isEqualToComparingFieldByFieldRecursively`** is the recursive comparison on this version
4. **`assertThatThrownBy`** for exceptions, with a message assertion that would fail if the wrong error were thrown
5. **Soft assertions for DTO mapping**, not as a general habit
6. **Custom asserts** once the same domain check repeats
7. Collections have their own reference: [assertj-collections.md](assertj-collections.md)
