# WebFlux / Reactive Testing Patterns

Quick-reference for testing reactive controllers with `org.springframework.boot.webflux.test.autoconfigure.WebFluxTest` (`@WebFluxTest`), `org.springframework.test.web.reactive.server.WebTestClient` (`WebTestClient`), and `reactor.test.StepVerifier` (`StepVerifier`).

---

## @WebFluxTest Basics

`org.springframework.boot.webflux.test.autoconfigure.WebFluxTest` (`@WebFluxTest`) creates a WebFlux test slice:

- **Loads**: `RouterFunction` beans, WebFlux controllers, `WebFilter`, `WebFluxConfigurer`, Spring Security WebFlux filter chain, auto-configured `WebTestClient`
- **Does NOT load**: `@Service`, `@Repository`, `@Component` — mock these with `@MockitoBean`

```java
import org.springframework.boot.webflux.test.autoconfigure.WebFluxTest;
import org.springframework.test.context.bean.override.mockito.MockitoBean;
import org.springframework.test.web.reactive.server.WebTestClient;
import org.springframework.beans.factory.annotation.Autowired;

import static org.mockito.BDDMockito.*;

@WebFluxTest(ProductController.class)
class ProductControllerTest {

    @Autowired WebTestClient webTestClient;
    @MockitoBean ProductService productService;   // Boot 4: @MockitoBean (was @MockBean in Boot 3)

    @Test
    void getProduct_returnsProduct() {
        given(productService.findById(1L))
            .willReturn(Mono.just(new ProductDto(1L, "Widget", BigDecimal.TEN)));

        webTestClient.get().uri("/products/1")
            .accept(MediaType.APPLICATION_JSON)
            .exchange()
            .expectStatus().isOk()
            .expectBody()
            .jsonPath("$.id").isEqualTo(1)
            .jsonPath("$.name").isEqualTo("Widget");
    }
}
```

---

## WebTestClient — GET / POST / PUT / DELETE

```java
// GET with query param
webTestClient.get()
    .uri(uri -> uri.path("/products").queryParam("category", "ELECTRONICS").build())
    .exchange()
    .expectStatus().isOk()
    .expectBodyList(ProductDto.class)
    .hasSize(3);

// POST with JSON body
webTestClient.post().uri("/products")
    .contentType(MediaType.APPLICATION_JSON)
    .bodyValue(new ProductRequest("Gadget", BigDecimal.valueOf(29.99)))
    .exchange()
    .expectStatus().isCreated()
    .expectHeader().valueMatches("Location", ".*/products/\\d+");

// PUT
webTestClient.put().uri("/products/1")
    .contentType(MediaType.APPLICATION_JSON)
    .bodyValue(req)
    .exchange()
    .expectStatus().isOk();

// DELETE
webTestClient.delete().uri("/products/1")
    .exchange()
    .expectStatus().isNoContent();
```

---

## expectBody Assertions

```java
// Single object with AssertJ value consumer
webTestClient.get().uri("/products/1")
    .exchange()
    .expectStatus().isOk()
    .expectBody(ProductDto.class)
    .value(dto -> {
        assertThat(dto.getName()).isEqualTo("Widget");
        assertThat(dto.getPrice()).isEqualByComparingTo("9.99");
    });

// List
webTestClient.get().uri("/products")
    .exchange()
    .expectBodyList(ProductDto.class)
    .hasSize(5)
    .value(list -> assertThat(list).extracting(ProductDto::getName)
                                   .contains("Widget", "Gadget"));

// Raw jsonPath
webTestClient.get().uri("/products/1")
    .exchange()
    .expectBody()
    .jsonPath("$.tags").isArray()
    .jsonPath("$.tags[0]").isEqualTo("featured");
```

---

## StepVerifier — Core Usage

`reactor.test.StepVerifier` (`StepVerifier`) is the primary tool for testing `Mono` and `Flux` at the service layer:

```java
import reactor.test.StepVerifier;
import reactor.core.publisher.Flux;

// Test a Flux
StepVerifier.create(Flux.just("alpha", "beta", "gamma"))
    .expectNext("alpha")
    .expectNext("beta")
    .expectNext("gamma")
    .verifyComplete();

// Shorthand
StepVerifier.create(flux)
    .expectNextCount(3)
    .verifyComplete();
```

---

## StepVerifier — Testing Mono

```java
import reactor.core.publisher.Mono;

Mono<UserDto> result = userService.findById(42L);

StepVerifier.create(result)
    .assertNext(user -> {
        assertThat(user.getId()).isEqualTo(42L);
        assertThat(user.getEmail()).isEqualTo("alice@example.com");
    })
    .verifyComplete();

// Empty Mono (no items, completes normally)
StepVerifier.create(userService.findById(99L))
    .verifyComplete();
```

---

## StepVerifier — Error Signals

```java
StepVerifier.create(orderService.findById(404L))
    .expectError(OrderNotFoundException.class)
    .verify();

// With message predicate
StepVerifier.create(orderService.findById(404L))
    .expectErrorMatches(ex ->
        ex instanceof OrderNotFoundException &&
        ex.getMessage().contains("404"))
    .verify();

// Shorthand
StepVerifier.create(mono)
    .verifyError(OrderNotFoundException.class);
```

---

## StepVerifier — Flux Sequences

```java
// expectNextMatches with predicate
StepVerifier.create(productFlux)
    .expectNextMatches(p -> p.getPrice().compareTo(BigDecimal.TEN) > 0)
    .expectNextMatches(Product::isActive)
    .verifyComplete();

// consumeNextWith for multi-field assertions
StepVerifier.create(orderFlux)
    .consumeNextWith(order -> {
        assertThat(order.getStatus()).isEqualTo(OrderStatus.PENDING);
        assertThat(order.getTotal()).isPositive();
    })
    .verifyComplete();
```

---

## StepVerifier.withVirtualTime — Time-Based Operators

```java
import java.time.Duration;
import reactor.core.publisher.Flux;

// CRITICAL: The Flux MUST be created inside the supplier lambda.
// Creating it outside causes real time to pass instead of virtual time.

StepVerifier.withVirtualTime(() ->
        Flux.interval(Duration.ofSeconds(1)).take(3))
    .expectSubscription()
    .thenAwait(Duration.ofSeconds(3))
    .expectNextCount(3)
    .verifyComplete();

// Testing timeout
StepVerifier.withVirtualTime(() ->
        slowMono.timeout(Duration.ofSeconds(5)))
    .expectSubscription()
    .thenAwait(Duration.ofSeconds(6))
    .expectError(java.util.concurrent.TimeoutException.class)
    .verify();
```

**Common mistake**: Creating the `Flux` before the `withVirtualTime` lambda — the operator subscription happens in real time, so virtual time never advances. The reactive source must be created inside the `Supplier<Publisher<T>>`.

---

## Reactive Security with WebTestClient

`org.springframework.security.test.web.reactive.server.SecurityMockServerConfigurers` provides `mutateWith` support for reactive security:

```java
import static org.springframework.security.test.web.reactive.server.SecurityMockServerConfigurers.*;

// Synthetic user
webTestClient.mutateWith(mockUser("alice").roles("USER"))
    .get().uri("/profile")
    .exchange()
    .expectStatus().isOk();

// JWT
webTestClient.mutateWith(
    mockJwt().jwt(j -> j.subject("user-123").claim("scope", "orders:read")))
    .get().uri("/orders")
    .exchange()
    .expectStatus().isOk();

// OIDC login
webTestClient.mutateWith(
    mockOidcLogin().idToken(t -> t.claim("email", "alice@example.com")))
    .get().uri("/profile")
    .exchange()
    .expectStatus().isOk();

// CSRF for mutating requests (session-based apps)
webTestClient.mutateWith(csrf())
    .mutateWith(mockUser("alice").roles("USER"))
    .post().uri("/orders")
    .contentType(MediaType.APPLICATION_JSON)
    .bodyValue(orderReq)
    .exchange()
    .expectStatus().isCreated();
```

`mutateWith` returns a new `WebTestClient` with the mutation applied — the original client is unchanged. Chain multiple `mutateWith` calls for compound configurations.

---

## Testing SSE (Server-Sent Events)

```java
import org.springframework.http.codec.ServerSentEvent;

webTestClient.get().uri("/events/stream")
    .accept(MediaType.TEXT_EVENT_STREAM)
    .exchange()
    .expectStatus().isOk()
    .returnResult(ServerSentEvent.class)
    .getResponseBody()
    .as(StepVerifier::create)
    .expectNextMatches(sse -> "order-update".equals(sse.event()))
    .thenCancel()          // CRITICAL: cancel — never wait for an infinite stream
    .verify(Duration.ofSeconds(5));
```

**Always use `thenCancel()`** for infinite streams (`Flux.interval`, SSE, etc.) — without it, the test waits forever for `verifyComplete()`.

---

## Full Integration with WebTestClient (@SpringBootTest)

When you need the full application context:

```java
import org.springframework.boot.test.autoconfigure.web.reactive.AutoConfigureWebTestClient;
import org.springframework.boot.test.context.SpringBootTest;

@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
@AutoConfigureWebTestClient
class OrderIntegrationTest {

    @Autowired WebTestClient webTestClient;

    @Test
    void createOrder_fullStack_returns201() {
        webTestClient.post().uri("/orders")
            .contentType(MediaType.APPLICATION_JSON)
            .bodyValue(new OrderRequest("PROD-1", 2))
            .exchange()
            .expectStatus().isCreated();
    }
}
```

---

## Boot 3.x → 4.x WebFlux Testing Changes

| Area | Boot 3.x | Boot 4.x |
|---|---|---|
| `@WebFluxTest` import | `org.springframework.boot.test.autoconfigure.web.reactive` | `org.springframework.boot.webflux.test.autoconfigure` |
| `WebTestClient` auto-config | Auto-configured in `@WebFluxTest` | Same |
| `@AutoConfigureWebTestClient` | Available | Same |
| Reactive security configurers | `o.s.security.test.web.reactive.server` | Same |
| Reactor Core | 3.5.x | 3.7.x+ (minor additions) |
| `@MockBean` | `org.springframework.boot.test.mock.mockito` | `@MockitoBean` from `org.springframework.test.context.bean.override.mockito` |

---

## Anti-Patterns

| Anti-Pattern | Fix |
|---|---|
| Calling `.block()` in tests | Use `StepVerifier` or `WebTestClient` — `.block()` hides reactive errors and breaks schedulers |
| Creating `Flux` outside `withVirtualTime` supplier | Flux must be created inside the `Supplier<Publisher<T>>` lambda |
| `verify()` with no timeout | Use `verify(Duration.ofSeconds(5))` or `verifyComplete()` — prevents test hanging forever |
| Not calling `thenCancel()` on infinite streams | Infinite `Flux.interval()` will hang the test indefinitely |
| Using `MockMvc` for WebFlux controller tests | Use `WebTestClient` — `MockMvc` is for servlet-based Spring MVC |
| Using `@SpringBootTest` for all WebFlux tests | Use `@WebFluxTest` for controller-only tests — significantly faster |
| Using `@MockBean` in Boot 4 | Replace with `@MockitoBean` from `org.springframework.test.context.bean.override.mockito` |
