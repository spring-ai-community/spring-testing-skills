# MVC / REST Controller Testing Patterns

Quick-reference for testing Spring MVC controllers with `org.springframework.boot.webmvc.test.autoconfigure.WebMvcTest` (`@WebMvcTest`) and `org.springframework.test.web.servlet.MockMvc` (`MockMvc`).

Spring Security is auto-configured in `@WebMvcTest` — you do not need `@SpringBootTest` to test secured endpoints. For security-specific testing patterns (`@WithMockUser`, JWT, OAuth2), see `spring-security-testing`.

---

## @WebMvcTest Basics

`org.springframework.boot.webmvc.test.autoconfigure.WebMvcTest` (`@WebMvcTest`) creates a Spring MVC test slice:

- **Loads**: Controllers, `@ControllerAdvice`, `JsonComponent`, `Filter`, `WebMvcConfigurer`, Spring Security filter chain
- **Does NOT load**: `@Service`, `@Repository`, `@Component` — mock these with `@MockitoBean`

```java
import org.springframework.boot.webmvc.test.autoconfigure.WebMvcTest;
import org.springframework.test.context.bean.override.mockito.MockitoBean;
import org.springframework.test.web.servlet.MockMvc;
import org.springframework.beans.factory.annotation.Autowired;
import com.fasterxml.jackson.databind.ObjectMapper;

import static org.mockito.BDDMockito.*;
import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.*;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.*;

@WebMvcTest(UserController.class)
class UserControllerTest {

    @Autowired MockMvc mockMvc;
    @MockitoBean UserService userService;     // Boot 4: @MockitoBean (was @MockBean in Boot 3)
    @Autowired ObjectMapper objectMapper;

    @Test
    void getUser_returnsUser() throws Exception {
        given(userService.findById(1L))
            .willReturn(new UserDto(1L, "alice@example.com", "Alice"));

        mockMvc.perform(get("/users/1")
                .accept(MediaType.APPLICATION_JSON))
            .andExpect(status().isOk())
            .andExpect(jsonPath("$.id").value(1))
            .andExpect(jsonPath("$.email").value("alice@example.com"))
            .andExpect(jsonPath("$.name").value("Alice"));
    }
}
```

**Always scope `@WebMvcTest`** to the specific controller class under test. Without scoping, all controllers in the application context are loaded — slower and noisier.

---

## Static Imports

Include these at the top of every MockMvc test class:

```java
import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.*;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.*;
import static org.springframework.test.web.servlet.result.MockMvcResultHandlers.*;
import static org.mockito.BDDMockito.*;
import static org.hamcrest.Matchers.*;
```

---

## GET / POST / PUT / DELETE

```java
// GET with path variable and query param
mockMvc.perform(get("/orders/{id}", 42L)
        .param("includeItems", "true")
        .accept(MediaType.APPLICATION_JSON))
    .andExpect(status().isOk())
    .andExpect(jsonPath("$.id").value(42));

// POST with JSON body
OrderRequest req = new OrderRequest("PROD-1", 3);
mockMvc.perform(post("/orders")
        .contentType(MediaType.APPLICATION_JSON)
        .content(objectMapper.writeValueAsString(req)))
    .andExpect(status().isCreated())
    .andExpect(header().string("Location", containsString("/orders/")));

// PUT
mockMvc.perform(put("/orders/{id}", 42L)
        .contentType(MediaType.APPLICATION_JSON)
        .content(objectMapper.writeValueAsString(req)))
    .andExpect(status().isOk());

// DELETE
mockMvc.perform(delete("/orders/{id}", 42L))
    .andExpect(status().isNoContent());
```

---

## jsonPath Assertions

```java
// Scalar values
.andExpect(jsonPath("$.name").value("Alice"))
.andExpect(jsonPath("$.active").value(true))
.andExpect(jsonPath("$.age").value(greaterThan(18)))   // Hamcrest matcher

// Arrays
.andExpect(jsonPath("$.items").isArray())
.andExpect(jsonPath("$.items", hasSize(3)))
.andExpect(jsonPath("$.items[0].sku").value("PROD-1"))

// Existence checks
.andExpect(jsonPath("$.id").exists())
.andExpect(jsonPath("$.internalField").doesNotExist())

// String matching
.andExpect(jsonPath("$.email").value(containsString("@example.com")))
```

Prefer `jsonPath()` over `content().string(...)` — JSON field ordering is not guaranteed and string comparison is brittle.

---

## Request Validation (400 Bad Request)

```java
// Controller uses @Valid on @RequestBody

@Test
void createOrder_invalidRequest_returns400() throws Exception {
    OrderRequest invalid = new OrderRequest("", 0);

    mockMvc.perform(post("/orders")
            .contentType(MediaType.APPLICATION_JSON)
            .content(objectMapper.writeValueAsString(invalid)))
        .andExpect(status().isBadRequest());
}

// With structured error response from @RestControllerAdvice:
@Test
void createOrder_invalidRequest_returnsFieldErrors() throws Exception {
    mockMvc.perform(post("/orders")
            .contentType(MediaType.APPLICATION_JSON)
            .content("{\"sku\":\"\",\"quantity\":0}"))
        .andExpect(status().isBadRequest())
        .andExpect(jsonPath("$.errors", hasSize(greaterThanOrEqualTo(2))))
        .andExpect(jsonPath("$.errors[*].field", hasItems("sku", "quantity")));
}
```

---

## Testing @RestControllerAdvice Exception Handlers

`@WebMvcTest` auto-discovers `@RestControllerAdvice` beans in the same package. No extra import needed.

```java
@Test
void getUser_notFound_returns404() throws Exception {
    given(userService.findById(99L))
        .willThrow(new UserNotFoundException(99L));

    mockMvc.perform(get("/users/99"))
        .andExpect(status().isNotFound())
        .andExpect(jsonPath("$.message").value(containsString("99")));
}
```

In Spring Boot 3.2+ with `spring.mvc.problemdetails.enabled=true` (Boot 4: default on), standard exceptions return `ProblemDetail` (RFC 7807). Your exception handler tests should expect `application/problem+json`.

---

## Response Headers

```java
mockMvc.perform(post("/orders")
        .contentType(MediaType.APPLICATION_JSON)
        .content(objectMapper.writeValueAsString(req)))
    .andExpect(status().isCreated())
    .andExpect(header().exists("Location"))
    .andExpect(header().string("Location", matchesPattern(".*/orders/\\d+")));
```

---

## Multipart File Upload

```java
import org.springframework.mock.web.MockMultipartFile;

@Test
void uploadAvatar_returns200() throws Exception {
    MockMultipartFile file = new MockMultipartFile(
        "file", "avatar.png", MediaType.IMAGE_PNG_VALUE, "fake-image-bytes".getBytes());

    mockMvc.perform(multipart("/users/1/avatar").file(file))
        .andExpect(status().isOk());
}
```

---

## Full Context Integration Test (@SpringBootTest + @AutoConfigureMockMvc)

When you need the full application context (e.g., real database, security integration):

```java
import org.springframework.boot.test.autoconfigure.web.servlet.AutoConfigureMockMvc;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.security.test.context.support.WithMockUser;

@SpringBootTest
@AutoConfigureMockMvc
class UserIntegrationTest {

    @Autowired MockMvc mockMvc;

    @Test
    @WithMockUser(roles = "ADMIN")
    void listUsers_asAdmin_returnsAll() throws Exception {
        mockMvc.perform(get("/users"))
            .andExpect(status().isOk())
            .andExpect(jsonPath("$", hasSize(greaterThan(0))));
    }
}
```

Use `@SpringBootTest + @AutoConfigureMockMvc` sparingly — it's 10x slower than `@WebMvcTest`.

---

## Debugging

```java
mockMvc.perform(get("/users/1"))
    .andDo(print())    // Prints full request/response to stdout — remove before commit
    .andExpect(status().isOk());
```

---

## Boot 4.x: RestTestClient

Spring Boot 4 introduced `RestTestClient`, providing a `WebTestClient`-style fluent API for servlet-based applications:

```java
import org.springframework.boot.webmvc.test.autoconfigure.WebMvcTest;
import org.springframework.boot.test.autoconfigure.web.servlet.AutoConfigureRestTestClient;
import org.springframework.test.web.reactive.server.WebTestClient; // same client
import org.springframework.test.context.bean.override.mockito.MockitoBean;

@WebMvcTest(GreetingController.class)
@AutoConfigureRestTestClient
class GreetingControllerTest {

    @Autowired WebTestClient restTestClient;   // injected as RestTestClient alias
    @MockitoBean GreetingService service;

    @Test
    void greetingShouldReturnMessageFromService() {
        when(service.greet()).thenReturn("Hello, Mock");
        restTestClient.get().uri("/greeting")
            .exchange()
            .expectStatus().isOk()
            .expectBody(String.class)
            .isEqualTo("Hello, Mock");
    }
}
```

`RestTestClient` and `MockMvc` are both valid in Boot 4. `RestTestClient` is preferred for teams already using `WebTestClient` in WebFlux tests — same API, same assertions.

---

## Boot 3.x → 4.x Migration Table

| Area | Boot 3.x | Boot 4.x |
|---|---|---|
| `@WebMvcTest` import | `org.springframework.boot.test.autoconfigure.web.servlet` | `org.springframework.boot.webmvc.test.autoconfigure` |
| `@MockBean` | `org.springframework.boot.test.mock.mockito.MockBean` | `@MockitoBean` from `org.springframework.test.context.bean.override.mockito` |
| Test client | `MockMvc` | `MockMvc` or `RestTestClient` via `@AutoConfigureRestTestClient` |
| `ProblemDetail` | Opt-in (`spring.mvc.problemdetails.enabled=true`) | Default for standard exceptions |
| Servlet imports | `jakarta.servlet.*` | Same |

---

## Anti-Patterns

| Anti-Pattern | Fix |
|---|---|
| Using `@SpringBootTest` for every controller test | Use `@WebMvcTest` — 10x faster for controller-only tests |
| Not scoping `@WebMvcTest(controllers=…)` | Always specify the controller class to avoid loading everything |
| Asserting exact JSON with `content().string(…)` | Use `jsonPath()` — resilient to field ordering changes |
| Forgetting `contentType(APPLICATION_JSON)` on POST/PUT | MockMvc returns 415 Unsupported Media Type without it |
| Expecting `status().isOk()` without checking body | Check both status and key response fields |
| Verifying mock interactions instead of HTTP behavior | Controller tests verify the HTTP contract, not service internals |
| Using `@MockBean` (Boot 3) in Boot 4 code | Replace with `@MockitoBean` from `o.s.test.context.bean.override.mockito` |
