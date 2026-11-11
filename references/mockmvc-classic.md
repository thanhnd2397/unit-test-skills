# MockMvc (Spring 5.0.7)

The complete MockMvc API available on this stack. This is the **only** MockMvc API here — `MockMvcTester` and its AssertJ-style assertions were added in Spring Boot 3.2.

## When to use this reference

- Writing or maintaining any `@WebMvcTest`
- Writing a controller test inside `@SpringBootTest` with `@AutoConfigureMockMvc`
- Looking up a matcher: status, header, content, JSON path, redirect, view, forwarded URL

## Setup

### Inside a slice

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

### Inside a full context

```java
@ExtendWith(SpringExtension.class)
@SpringBootTest
@AutoConfigureMockMvc
class OrderMvcIntegrationTest {

  @Autowired
  private MockMvc mvc;
}
```

### Standalone (no Spring context at all)

Fastest option for a controller with hand-constructible dependencies:

```java
class OrderControllerStandaloneTest {

  private final OrderService orderService = mock(OrderService.class);
  private MockMvc mvc;

  @BeforeEach
  void setUp() {
    mvc = MockMvcBuilders.standaloneSetup(new OrderController(orderService))
        .setControllerAdvice(new RestExceptionHandler())
        .build();
  }
}
```

`standaloneSetup` skips Spring Boot auto-configuration entirely: no `@ControllerAdvice` discovery, no message converters from the context, no security filters — you register what you need. Use it when the context is the only slow part of the test.

## Basic requests

```java
@Test
void shouldReturnOrder() throws Exception {
  given(orderService.findById(1L)).willReturn(new Order(1L, "PENDING", new BigDecimal("99.99")));

  mvc.perform(get("/orders/1"))
     .andExpect(status().isOk())
     .andExpect(content().contentType(MediaType.APPLICATION_JSON))
     .andExpect(jsonPath("$.id").value(1))
     .andExpect(jsonPath("$.status").value("PENDING"))
     .andExpect(jsonPath("$.total").value(99.99));
}

@Test
void shouldCreateOrder() throws Exception {
  given(orderService.create(any(OrderRequest.class))).willReturn(1L);

  mvc.perform(post("/orders")
         .contentType(MediaType.APPLICATION_JSON)
         .content("{\"product\":\"Laptop\",\"quantity\":2}"))
     .andExpect(status().isCreated())
     .andExpect(header().string("Location", "/orders/1"));
}

@Test
void shouldUpdateOrder() throws Exception {
  mvc.perform(put("/orders/1")
         .contentType(MediaType.APPLICATION_JSON)
         .content("{\"status\":\"COMPLETED\"}"))
     .andExpect(status().isOk());
}

@Test
void shouldDeleteOrder() throws Exception {
  mvc.perform(delete("/orders/1"))
     .andExpect(status().isNoContent());
}
```

`patch(...)`, `head(...)`, `options(...)` and `multipart(...)` are available too. On Spring 5.0, file upload uses `MockMvcRequestBuilders.multipart(...)` (`fileUpload(...)` is the deprecated older name):

```java
MockMultipartFile file = new MockMultipartFile(
    "file", "orders.csv", "text/csv", "id,status\n1,PENDING".getBytes(StandardCharsets.UTF_8));

mvc.perform(multipart("/orders/import").file(file))
   .andExpect(status().isAccepted());
```

## Status matchers

```java
.andExpect(status().isOk())            // 200
.andExpect(status().isCreated())       // 201
.andExpect(status().isAccepted())      // 202
.andExpect(status().isNoContent())     // 204
.andExpect(status().isBadRequest())    // 400
.andExpect(status().isUnauthorized())  // 401
.andExpect(status().isForbidden())     // 403
.andExpect(status().isNotFound())      // 404
.andExpect(status().isConflict())      // 409
.andExpect(status().is5xxServerError())
.andExpect(status().is(422))           // arbitrary code
```

## JSON path assertions

```java
// exact value
.andExpect(jsonPath("$.status").value("PENDING"))

// existence
.andExpect(jsonPath("$.id").exists())
.andExpect(jsonPath("$.deletedAt").doesNotExist())

// types
.andExpect(jsonPath("$.items").isArray())
.andExpect(jsonPath("$.total").isNumber())
.andExpect(jsonPath("$.active").isBoolean())

// arrays, with Hamcrest
.andExpect(jsonPath("$.items", hasSize(3)))
.andExpect(jsonPath("$.items[0].product").value("Laptop"))
.andExpect(jsonPath("$.items[*].product", containsInAnyOrder("Laptop", "Mouse")))

// nested
.andExpect(jsonPath("$.customer.address.city").value("Hanoi"))

// other Hamcrest matchers
.andExpect(jsonPath("$.total", greaterThan(0.0)))
.andExpect(jsonPath("$.description", containsString("order")))
```

`jsonPath` takes **Hamcrest** matchers, not AssertJ. This is the one place in the suite where Hamcrest is still required — `org.hamcrest.Matchers.*` comes with `spring-boot-starter-test`.

## Content assertions

```java
.andExpect(content().contentType(MediaType.APPLICATION_JSON))
.andExpect(content().contentTypeCompatibleWith(MediaType.APPLICATION_JSON))
.andExpect(content().string(containsString("PENDING")))
.andExpect(content().json("{\"status\":\"PENDING\"}"))            // lenient JSON compare
.andExpect(content().json("{\"status\":\"PENDING\"}", true))      // strict: no extra fields
```

`content().json(...)` uses JSONassert, which `spring-boot-starter-test` provides.

## Header assertions

```java
.andExpect(header().string("Location", "/orders/1"))
.andExpect(header().string("Content-Type", containsString("application/json")))
.andExpect(header().longValue("X-Total-Count", 42L))
.andExpect(header().exists("X-Request-Id"))
.andExpect(header().doesNotExist("X-Deprecated"))
```

## Server-side MVC assertions

Useful when testing a classic MVC application with views rather than a REST API:

```java
.andExpect(view().name("orders/list"))
.andExpect(model().attributeExists("orders"))
.andExpect(model().attribute("pageTitle", "Orders"))
.andExpect(model().hasNoErrors())
.andExpect(model().attributeHasFieldErrors("orderForm", "quantity"))
.andExpect(forwardedUrl("/WEB-INF/views/orders/list.jsp"))
.andExpect(redirectedUrl("/orders/1"))
.andExpect(flash().attributeExists("message"))
```

## Request parameters, headers, session

```java
// query parameters
mvc.perform(get("/orders").param("status", "PENDING").param("page", "0"))

// path variables (placeholder form keeps the URI template readable)
mvc.perform(get("/orders/{id}", 1L))

// request headers
mvc.perform(get("/orders/1").header("X-Api-Key", "secret"))

// accept / content type
mvc.perform(get("/orders/1").accept(MediaType.APPLICATION_JSON))

// cookies and session
mvc.perform(get("/orders").cookie(new Cookie("locale", "vi")))
mvc.perform(get("/orders").sessionAttr("userId", 42L))

// form submission
mvc.perform(post("/orders")
       .contentType(MediaType.APPLICATION_FORM_URLENCODED)
       .param("product", "Laptop")
       .param("quantity", "2"))
```

## Capturing the response

```java
@Test
void shouldReturnCreatedId() throws Exception {
  given(orderService.create(any(OrderRequest.class))).willReturn(42L);

  MvcResult result = mvc.perform(post("/orders")
         .contentType(MediaType.APPLICATION_JSON)
         .content("{\"product\":\"Laptop\",\"quantity\":1}"))
     .andExpect(status().isCreated())
     .andReturn();

  String location = result.getResponse().getHeader("Location");
  assertThat(location).isEqualTo("/orders/42");

  String body = result.getResponse().getContentAsString();
  assertThat(body).contains("\"id\":42");
}
```

Combining `andReturn()` with AssertJ is the idiomatic way to get fluent assertions out of classic MockMvc — parse the body with `JacksonTester` or `ObjectMapper` and assert on the object:

```java
Order created = objectMapper.readValue(result.getResponse().getContentAsString(), Order.class);

assertThat(created.getStatus()).isEqualTo("PENDING");
assertThat(created.getTotal()).isEqualByComparingTo(new BigDecimal("99.99"));
```

Note: `getContentAsString()` uses the response character encoding; for non-ASCII content set it explicitly in the controller or use `getContentAsString(StandardCharsets.UTF_8)` where available.

## Debugging

```java
mvc.perform(get("/orders/1"))
   .andDo(print())                  // dumps request + response to stdout
   .andExpect(status().isOk());
```

`andDo(print())` is for diagnosing a failing test; remove it before committing or the build log fills up.

## Static imports

```java
import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.*;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.*;
import static org.springframework.test.web.servlet.result.MockMvcResultHandlers.print;
import static org.springframework.security.test.web.servlet.request.SecurityMockMvcRequestPostProcessors.csrf;
import static org.hamcrest.Matchers.*;
import static org.assertj.core.api.Assertions.assertThat;
```

## Key points

1. **`throws Exception` on every test method** — `perform()` throws it
2. **`jsonPath` needs Hamcrest**, everything outside MockMvc uses AssertJ
3. **`andDo(print())` only while debugging**
4. **Prefer `jsonPath()` over `content().string()`** for field-level precision
5. **`standaloneSetup` when the context is the bottleneck** — no auto-configuration, you register advice and converters yourself
6. **`andReturn()` + AssertJ** for anything the matchers cannot express
7. There is no migration target on this stack — `MockMvcTester` requires Boot 3.2
