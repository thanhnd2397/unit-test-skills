# @RestClientTest (Boot 2.0.3)

Testing code that calls an external HTTP API, with `MockRestServiceServer` standing in for the remote service. No network, no WireMock.

## What it auto-configures

- `RestTemplateBuilder` wired to a mock request factory
- `MockRestServiceServer` bound to that builder
- Jackson `ObjectMapper` and the standard message converters
- The classes listed in the annotation (`@RestClientTest(WeatherService.class)`)

On this stack the client is always `RestTemplate` — `RestClient` is Spring Framework 6.1.

## Requirement on the class under test

The service must build its `RestTemplate` from an injected `RestTemplateBuilder`, otherwise the mock server has nothing to intercept:

```java
@Service
public class WeatherService {

  private final RestTemplate restTemplate;

  public WeatherService(RestTemplateBuilder builder) {
    this.restTemplate = builder.rootUri("https://api.weather.com").build();
  }

  public Weather getCurrentWeather(String city) {
    return restTemplate.getForObject("/v1/current?city={city}", Weather.class, city);
  }
}
```

A service that does `new RestTemplate()` internally cannot be tested this way — either refactor it to take the builder, or inject a `RestTemplate` bean and bind the mock server manually:

```java
RestTemplate restTemplate = new RestTemplate();
MockRestServiceServer server = MockRestServiceServer.bindTo(restTemplate).build();
WeatherService service = new WeatherService(restTemplate);
```

That form needs no Spring context at all and is often the better test.

## Basic test

```java
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.ExtendWith;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.autoconfigure.web.client.RestClientTest;
import org.springframework.test.context.junit.jupiter.SpringExtension;
import org.springframework.test.web.client.MockRestServiceServer;

import static org.assertj.core.api.Assertions.assertThat;
import static org.springframework.test.web.client.match.MockRestRequestMatchers.*;
import static org.springframework.test.web.client.response.MockRestResponseCreators.*;

@ExtendWith(SpringExtension.class)      // REQUIRED on Boot 2.0.3
@RestClientTest(WeatherService.class)
class WeatherServiceTest {

  @Autowired
  private WeatherService weatherService;

  @Autowired
  private MockRestServiceServer server;

  @Test
  void shouldFetchWeather() {
    server.expect(requestTo("https://api.weather.com/v1/current?city=Hanoi"))
          .andExpect(method(HttpMethod.GET))
          .andRespond(withSuccess("{\"temperature\":22,\"condition\":\"Sunny\"}",
                                  MediaType.APPLICATION_JSON));

    Weather weather = weatherService.getCurrentWeather("Hanoi");

    assertThat(weather.getTemperature()).isEqualTo(22);
    assertThat(weather.getCondition()).isEqualTo("Sunny");
    server.verify();
  }
}
```

## Request matching

```java
// exact URI
server.expect(requestTo("https://api.example.com/users/1")).andRespond(withSuccess());

// URI pattern (Hamcrest)
server.expect(requestTo(matchesPattern("https://api.example.com/users/\\d+")))
      .andRespond(withSuccess());

// method + query parameter
server.expect(requestTo(containsString("/users")))
      .andExpect(method(HttpMethod.GET))
      .andExpect(queryParam("page", "0"))
      .andRespond(withSuccess());

// expected call count
server.expect(ExpectedCount.once(), requestTo("https://api.example.com/users"))
      .andExpect(method(HttpMethod.POST))
      .andRespond(withCreatedEntity(URI.create("/users/1")));

server.expect(ExpectedCount.manyTimes(), requestTo(matchesPattern(".*/api/.*")))
      .andRespond(withSuccess());

// request body
server.expect(requestTo("https://api.example.com/users"))
      .andExpect(content().contentType(MediaType.APPLICATION_JSON))
      .andExpect(content().json("{\"name\":\"John\"}"))
      .andRespond(withSuccess());

// request body by JSON path
server.expect(requestTo("https://api.example.com/users"))
      .andExpect(jsonPath("$.name").value("John"))
      .andRespond(withSuccess());

// headers
server.expect(requestTo("https://api.example.com/users"))
      .andExpect(header("Authorization", "Bearer token123"))
      .andExpect(header("X-Api-Key", "secret"))
      .andRespond(withSuccess());
```

`requestTo(...)` accepts a `String`, a `URI` or a Hamcrest matcher. Match on the minimum that makes the test meaningful — asserting every header makes the test brittle.

## Responses

```java
// success with a JSON body
server.expect(requestTo("/users/1"))
      .andRespond(withSuccess("{\"id\":1,\"name\":\"John\"}", MediaType.APPLICATION_JSON));

// success from a classpath resource — preferred for large payloads
server.expect(requestTo("/users/1"))
      .andRespond(withSuccess(new ClassPathResource("stubs/user-1.json"),
                              MediaType.APPLICATION_JSON));

// created
server.expect(requestTo("/users"))
      .andExpect(method(HttpMethod.POST))
      .andRespond(withCreatedEntity(URI.create("/users/1")));

// errors
server.expect(requestTo("/users/999")).andRespond(withResourceNotFound());
server.expect(requestTo("/users")).andRespond(withServerError().body("boom"));
server.expect(requestTo("/users")).andRespond(withBadRequest().body("{\"error\":\"invalid\"}"));
server.expect(requestTo("/users")).andRespond(withStatus(HttpStatus.TOO_MANY_REQUESTS));

// transport failure
server.expect(requestTo("/users")).andRespond(withException(new SocketTimeoutException()));
```

`withException(...)` is how you test timeout and connection-failure handling on this stack. `DefaultResponseCreator.delay(...)` is a later Spring addition and is not available — do not try to simulate latency.

## Testing error handling

```java
@Test
void shouldWrapNotFoundInDomainException() {
  server.expect(requestTo("https://api.weather.com/v1/current?city=Nowhere"))
        .andRespond(withResourceNotFound());

  assertThatThrownBy(() -> weatherService.getCurrentWeather("Nowhere"))
      .isInstanceOf(WeatherUnavailableException.class)
      .hasMessageContaining("Nowhere");

  server.verify();
}

@Test
void shouldFallBackOnServerError() {
  server.expect(requestTo(containsString("/v1/current")))
        .andRespond(withServerError());

  Weather weather = weatherService.getCurrentWeatherOrDefault("Hanoi");

  assertThat(weather.getCondition()).isEqualTo("UNKNOWN");
}
```

Error paths are the reason this slice exists — a client that is only tested on 200 responses is untested.

## Verifying and resetting

```java
@Test
void shouldCallApiExactlyOnce() {
  server.expect(ExpectedCount.once(), requestTo("https://api.example.com/data"))
        .andRespond(withSuccess());

  service.fetchData();

  server.verify();          // fails if the expectation was not met
}
```

`server.verify()` is what turns "the code compiled and returned" into "the request was actually made as specified". Call it at the end of every test, or use `server.verify(Duration)`-free form as above.

Expectations are per-test; the slice gives each test method a fresh server, but when sharing setup across methods:

```java
@BeforeEach
void setUp() {
  server.reset();
}
```

## Ignoring request order

```java
server = MockRestServiceServer.bindTo(restTemplate).ignoreExpectOrder(true).build();
```

With the auto-configured server inside `@RestClientTest`, expectations are matched in declaration order by default — declare them in the order the code will call them, or build the server manually when order genuinely does not matter.

## Key points

1. **`@ExtendWith(SpringExtension.class)` first**
2. The class under test must accept `RestTemplateBuilder` (or a `RestTemplate`) — otherwise the mock cannot intercept
3. **`server.verify()`** at the end of each test
4. Stub bodies from `src/test/resources` once they exceed a line or two (no text blocks on Java 8)
5. **Test the error paths**: 404, 500, timeout via `withException`
6. Match on the minimum set of request attributes
7. No `.delay(...)`, no `RestClient`, no WireMock on this stack
