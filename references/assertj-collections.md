# AssertJ Collections (3.9.1)

Assertions for `List`, `Set`, `Map`, arrays and streams, limited to the API available in AssertJ 3.9.1.

## When to use this reference

- The value under test is a `List`, `Set`, `Map`, array or `Stream`
- You need to assert on several elements, their order, or fields inside them
- You are reaching for `extracting()`, `filteredOn()`, `containsExactly()`
- A single scalar or object → [assertj-basics.md](assertj-basics.md)

## Size and emptiness

```java
List<Order> orders = orderService.findAll();

assertThat(orders).isNotEmpty();
assertThat(orders).isEmpty();
assertThat(orders).hasSize(3);
assertThat(orders).hasSameSizeAs(expectedOrders);
```

`hasSizeGreaterThan(n)`, `hasSizeLessThan(n)` and `hasSizeBetween(a, b)` were added in 3.12 and are **not available**. Assert on the size directly:

```java
assertThat(orders.size()).isGreaterThan(0);
assertThat(orders.size()).isLessThanOrEqualTo(10);
```

## Containment

```java
// contains these, any order, extras allowed
assertThat(orders).contains(order1, order2);

// exactly these, in this order, no extras
assertThat(statuses).containsExactly("NEW", "PENDING", "COMPLETED");

// exactly these, any order, no extras
assertThat(statuses).containsExactlyInAnyOrder("COMPLETED", "NEW", "PENDING");

// at least one of them
assertThat(statuses).containsAnyOf("NEW", "CANCELLED");

// all of them, any order (extras allowed) — same as contains(...)
assertThat(statuses).containsAll(Arrays.asList("NEW", "PENDING"));

// subsequence / sequence
assertThat(statuses).containsSequence("NEW", "PENDING");       // consecutive
assertThat(statuses).containsSubsequence("NEW", "COMPLETED");  // in order, gaps allowed

// absence
assertThat(statuses).doesNotContain("DELETED");
assertThat(statuses).doesNotHaveDuplicates();

// single element
assertThat(orders).containsOnlyOnce(order1);
assertThat(orders).hasSize(1).first().isEqualTo(order1);
```

`containsExactlyInAnyOrderElementsOf(collection)` and `containsExactlyElementsOf(collection)` take an `Iterable` instead of varargs and are available in 3.9.1 — useful when the expected values come from a list.

## Extracting fields

```java
// single field
assertThat(orders)
    .extracting(Order::getStatus)
    .containsExactly("NEW", "PENDING", "COMPLETED");

// by name (works without a getter reference)
assertThat(orders)
    .extracting("status")
    .containsExactly("NEW", "PENDING", "COMPLETED");

// several fields as tuples
assertThat(orders)
    .extracting(Order::getId, Order::getStatus)
    .containsExactly(
        tuple(1L, "NEW"),
        tuple(2L, "PENDING"),
        tuple(3L, "COMPLETED"));

// nested path by name
assertThat(orders)
    .extracting("customer.name")
    .contains("John Doe");
```

`extracting` on a collection avoids having to implement `equals()` on domain objects, and gives a readable failure message naming the actual values.

## Filtering

```java
assertThat(orders)
    .filteredOn(order -> "PENDING".equals(order.getStatus()))
    .hasSize(2)
    .extracting(Order::getId)
    .containsExactlyInAnyOrder(1L, 3L);

// by field value
assertThat(orders).filteredOn("status", "PENDING").hasSize(2);

// null / not-null
assertThat(orders).filteredOn("cancelledAt", null).hasSize(3);
assertThat(orders).filteredOnNull("cancelledAt").hasSize(3);
```

With a lambda, `filteredOn` takes a `Predicate` — Java 8 friendly.

## Predicates and per-element assertions

```java
assertThat(orders).allMatch(o -> o.getTotal().compareTo(BigDecimal.ZERO) > 0);
assertThat(orders).anyMatch(o -> "COMPLETED".equals(o.getStatus()));
assertThat(orders).noneMatch(o -> "DELETED".equals(o.getStatus()));

// assertions (not just a boolean) on every element — better failure messages
assertThat(orders).allSatisfy(o -> {
  assertThat(o.getId()).isPositive();
  assertThat(o.getStatus()).isNotEmpty();
});
```

`satisfiesExactly(...)` (3.19) is **not available**. When each element needs different assertions, assert by index:

```java
assertThat(orders).hasSize(3);
assertThat(orders.get(0).getStatus()).isEqualTo("NEW");
assertThat(orders.get(1).getStatus()).isEqualTo("PENDING");
assertThat(orders.get(2).getStatus()).isEqualTo("COMPLETED");
```

Or navigate with the element accessors AssertJ 3.9.1 does provide:

```java
assertThat(orders).first().hasFieldOrPropertyWithValue("status", "NEW");
assertThat(orders).last().hasFieldOrPropertyWithValue("status", "COMPLETED");
assertThat(orders).element(1).hasFieldOrPropertyWithValue("status", "PENDING");
```

## Nested collections

```java
assertThat(orders)
    .flatExtracting(Order::getItems)
    .extracting(OrderItem::getProduct)
    .contains("Laptop", "Mouse");

// by field name
assertThat(orders).flatExtracting("items").hasSize(5);
```

## Comparing elements by fields

```java
// element comparison by all fields instead of equals()
assertThat(orders)
    .usingFieldByFieldElementComparator()
    .containsExactlyInAnyOrder(expectedOrder1, expectedOrder2);

// recursive variant (nested objects by fields too)
assertThat(orders)
    .usingRecursiveFieldByFieldElementComparator()
    .containsExactly(expectedOrder1, expectedOrder2);

// ignore generated values
assertThat(orders)
    .usingElementComparatorIgnoringFields("id", "createdAt")
    .containsExactly(expectedOrder1, expectedOrder2);

// compare on selected fields only
assertThat(orders)
    .usingElementComparatorOnFields("status", "total")
    .containsExactly(expectedOrder1, expectedOrder2);

// custom comparator
assertThat(orders)
    .usingElementComparator(Comparator.comparing(Order::getId))
    .containsExactly(expectedOrder1, expectedOrder2);
```

`usingRecursiveFieldByFieldElementComparatorIgnoringFields(...)` is a later addition — use `usingElementComparatorIgnoringFields(...)` as above.

## Maps

```java
Map<String, Integer> stockByProduct = inventoryService.getStock();

assertThat(stockByProduct)
    .isNotEmpty()
    .hasSize(3)
    .containsKey("Laptop")
    .containsKeys("Laptop", "Mouse")
    .doesNotContainKey("Fax Machine")
    .containsValue(10)
    .containsEntry("Laptop", 10)
    .contains(entry("Laptop", 10), entry("Mouse", 50))
    .containsOnlyKeys("Laptop", "Mouse", "Keyboard");

assertThat(stockByProduct)
    .hasEntrySatisfying("Laptop", qty -> assertThat(qty).isGreaterThan(0));

// assert on keys or values as collections
assertThat(stockByProduct.keySet()).containsExactlyInAnyOrder("Laptop", "Mouse", "Keyboard");
assertThat(stockByProduct.values()).allMatch(qty -> qty >= 0);
```

## Arrays

```java
String[] roles = user.getRoles();

assertThat(roles).hasSize(2);
assertThat(roles).contains("ADMIN");
assertThat(roles).containsExactlyInAnyOrder("USER", "ADMIN");
assertThat(roles).doesNotContain("ROOT");
assertThat(roles).isSortedAccordingTo(Comparator.naturalOrder());
```

## Sets

```java
Set<String> tags = product.getTags();

assertThat(tags).contains("electronics", "sale");
assertThat(tags).doesNotContain("expired");
assertThat(tags).containsExactlyInAnyOrder("electronics", "sale", "new");
```

Never use `containsExactly` on a `HashSet` — iteration order is not part of the contract and the test will be flaky across JVMs.

## Streams

```java
assertThat(orderService.streamPending())
    .extracting(Order::getStatus)
    .containsOnly("PENDING");
```

AssertJ consumes the stream into a list, so the stream cannot be reused after the assertion.

## Sorting

```java
assertThat(orders)
    .extracting(Order::getCreatedAt)
    .isSortedAccordingTo(Comparator.<LocalDateTime>reverseOrder());

assertThat(statuses).isSorted();          // natural order
```

## Java 8 notes

```java
// no List.of
List<Order> expected = Arrays.asList(order1, order2);
Map<String, Integer> expectedStock = Collections.singletonMap("Laptop", 10);

// no getFirst() / getLast() / reversed()
assertThat(orders.get(0)).isEqualTo(order1);
assertThat(orders.get(orders.size() - 1)).isEqualTo(order3);

// collecting a stream
List<String> statuses = orders.stream().map(Order::getStatus).collect(Collectors.toList());
```

## Static imports

```java
import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.tuple;
import static org.assertj.core.api.Assertions.entry;
```

## Not available in 3.9.1

| Modern API | Added in | Use on 3.9.1 |
|---|---|---|
| `hasSizeGreaterThan(n)` / `hasSizeLessThan(n)` / `hasSizeBetween(a, b)` | 3.12 | `assertThat(list.size()).isGreaterThan(n)` |
| `satisfiesExactly(...)` / `satisfiesExactlyInAnyOrder(...)` | 3.19 | `allSatisfy(...)`, `element(i)`, or index assertions |
| `usingRecursiveFieldByFieldElementComparatorIgnoringFields(...)` | 3.17+ | `usingElementComparatorIgnoringFields(...)` |
| `usingRecursiveComparison()` on collections | 3.12 | `usingRecursiveFieldByFieldElementComparator()` |
| `anySatisfy(...)` / `noneSatisfy(...)` | 3.12+ | `anyMatch(...)` / `noneMatch(...)` |
| `containsAnyElementsOf(...)` | 3.12 | `containsAnyOf(...)` with varargs |
| `map(...)` as an alias for `extracting(...)` | 3.19 | `extracting(...)` |

## Key points

1. **`containsExactly` only when order is part of the contract** — never on a `HashSet`
2. **`extracting()` before containment checks** — no `equals()` needed, better messages
3. **`filteredOn()` + `extracting()`** composes into a precise subset assertion
4. **`allSatisfy` over `allMatch`** when you want to see which element failed
5. **`usingElementComparatorIgnoringFields`** for comparing against expected objects with generated IDs
6. **Size comparisons go through `list.size()`** on this version
7. **Java 8 only**: `Arrays.asList`, `get(0)`, `Collectors.toList()`
