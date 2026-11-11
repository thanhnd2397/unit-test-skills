# @WebMvcTest (Boot 2.0.3)

Testing Spring MVC controllers with a focused slice. The assertion API is classic `MockMvc` — `MockMvcTester` arrived in Boot 3.2 and does not exist here.

## Basic structure

```java
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.ExtendWith;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.autoconfigure.web.servlet.WebMvcTest;
import org.springframework.boot.test.mock.mockito.MockBean;
import org.springframework.test.context.junit.jupiter.SpringExtension;
import org.springframework.test.web.servlet.MockMvc;

import static org.mockito.ArgumentMatchers.any;
import static org.mockito.BDDMockito.given;
import static org.mockito.Mockito.verify;
import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.*;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.*;

@ExtendWith(SpringExtension.class)      // REQUIRED on Boot 2.0.3
@WebMvcTest(OrderController.class)
class OrderControllerTest {

  @Autowired
  private MockMvc mvc;

  @MockBean
  private OrderService orderService;

  @MockBean
  private UserService userService;
}
```

Narrow the slice to the controller under test (`@WebMvcTest(OrderController.class)`). Without the argument, every controller in the application is loaded and each of their collaborators needs a `@MockBean`.

## What gets loaded

- The specified controller(s)
- Spring MVC infrastructure (`HandlerMapping`, `HandlerAdapter`, message converters)
- Jackson `ObjectMapper`
- `@ControllerAdvice` exception handlers
- Bean validation (if a provider is on the classpath)
- Spring Security filters (if Spring Security is on the classpath)

Not loaded: `@Service`, `@Component`, `@Repository` beans, the DataSource, JPA.

## Testing GET endpoints

```java
@Test
void shouldReturnOrder() throws Exception {
  given(orderService.findById(1L)).willReturn(OrderFixtures.pendingOrder());

  mvc.perform(get("/orders/1"))
     .andExpect(status().isOk())
     .andExpect(content().contentType(MediaType.APPLICATION_JSON))
     .andExpect(jsonPath("$.id").value(1))
     .andExpect(jsonPath("$.status").value("PENDING"));
}
```

Every test method needs `throws Exception` — `perform()` declares it.

## Testing POST with a request body

```java
@Test
void shouldCreateOrder() throws Exception {
  given(orderService.create(any(OrderRequest.class))).willReturn(1L);

  mvc.perform(post("/orders")
         .contentType(MediaType.APPLICATION_JSON)
         .content("{\"product\":\"Laptop\",\"quantity\":2}"))
     .andExpect(status().isCreated())
     .andExpect(header().string("Location", "/orders/1"));
}
```

For a payload long enough to hurt readability as an escaped string, put it in `src/test/resources` and read it:

```java
private String payload(String name) throws IOException {
  return StreamUtils.copyToString(
      new ClassPathResource("orders/" + name).getInputStream(), StandardCharsets.UTF_8);
}

@Test
void shouldCreateOrderFromFixtureFile() throws Exception {
  mvc.perform(post("/orders")
         .contentType(MediaType.APPLICATION_JSON)
         .content(payload("create-request.json")))
     .andExpect(status().isCreated());
}
```

Or build the JSON from an object with `JacksonTester` (`@AutoConfigureJsonTesters` is on by default in `@WebMvcTest`):

```java
@Autowired
private JacksonTester<OrderRequest> json;

@Test
void shouldCreateOrderFromObject() throws Exception {
  OrderRequest request = new OrderRequest("Laptop", 2);

  mvc.perform(post("/orders")
         .contentType(MediaType.APPLICATION_JSON)
         .content(json.write(request).getJson()))
     .andExpect(status().isCreated());
}
```

## Testing validation errors

```java
@Test
void shouldRejectInvalidOrder() throws Exception {
  mvc.perform(post("/orders")
         .contentType(MediaType.APPLICATION_JSON)
         .content("{\"product\":\"\",\"quantity\":-1}"))
     .andExpect(status().isBadRequest())
     .andExpect(jsonPath("$.errors").exists());

  verify(orderService, never()).create(any(OrderRequest.class));
}
```

The exact error body depends on the project's `@ControllerAdvice`; assert against what that handler actually produces, not against the default Spring Boot error shape.

## Query parameters and path variables

```java
@Test
void shouldFilterOrdersByStatus() throws Exception {
  mvc.perform(get("/orders").param("status", "PENDING").param("page", "0"))
     .andExpect(status().isOk());

  verify(orderService).findByStatus("PENDING");
}

@Test
void shouldCancelOrder() throws Exception {
  mvc.perform(put("/orders/{id}/cancel", 123L))
     .andExpect(status().isOk());

  verify(orderService).cancel(123L);
}
```

## Testing with security

Needs `spring-security-test` on the test classpath (managed by the Boot BOM, not part of the baseline set — add it deliberately):

```java
@Test
@WithMockUser(roles = "ADMIN")
void adminShouldDeleteOrder() throws Exception {
  mvc.perform(delete("/orders/1"))
     .andExpect(status().isNoContent());
}

@Test
void anonymousShouldBeUnauthorized() throws Exception {
  mvc.perform(delete("/orders/1"))
     .andExpect(status().isUnauthorized());
}
```

CSRF applies to state-changing requests when Spring Security is active:

```java
mvc.perform(post("/orders").with(csrf())
       .contentType(MediaType.APPLICATION_JSON)
       .content("{\"product\":\"Laptop\",\"quantity\":1}"))
   .andExpect(status().isCreated());
```

To take security out of the picture entirely:

```java
@ExtendWith(SpringExtension.class)
@WebMvcTest(OrderController.class)
@AutoConfigureMockMvc(addFilters = false)   // no servlet filters, including security
class OrderControllerWithoutSecurityTest { }
```

`addFilters = false` also removes any other filter the slice registers — if the behavior under test depends on a filter, keep them on and authenticate instead.

## Multiple controllers in one slice

```java
@ExtendWith(SpringExtension.class)
@WebMvcTest({OrderController.class, ProductController.class})
class WebLayerTest { }
```

Only do this when the test genuinely spans both; otherwise it widens the context and the set of required `@MockBean`s.

## Testing an exception handler

```java
@Test
void shouldReturn404WhenOrderMissing() throws Exception {
  given(orderService.findById(999L)).willThrow(new OrderNotFoundException(999L));

  mvc.perform(get("/orders/999"))
     .andExpect(status().isNotFound())
     .andExpect(jsonPath("$.message").value("Order 999 not found"));
}
```

## Key points

1. **`@ExtendWith(SpringExtension.class)` first** — without it nothing is auto-configured
2. **`@MockBean` for every collaborator** of the loaded controller(s)
3. **`throws Exception` on every test method**
4. **Assert HTTP semantics here** — status, headers, content type, JSON shape
5. **Do not assert business rules here** — that belongs in a unit test
6. **`verify(...)` the service call** when the side effect is the point of the endpoint
7. **Java 8**: escaped JSON strings, resource files or `JacksonTester`, never text blocks

Full MockMvc API for Spring 5.0.7: [mockmvc-classic.md](mockmvc-classic.md).
