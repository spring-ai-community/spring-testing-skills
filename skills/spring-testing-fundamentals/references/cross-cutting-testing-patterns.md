# Cross-Cutting Testing Patterns

Strategic guidance that applies across all Spring test types: the testing pyramid, context caching, behavior-over-implementation, Boot 4 readiness, and universal anti-patterns.

Source: Andy Wilkinson's "Testing Spring Boot Applications" (SpringOne 2019), Spring Boot team guidance.

---

## Testing Pyramid — Slice Selection

Always use the narrowest slice that covers your test objective. Only escalate when you're testing cross-cutting behavior (security + service + persistence together).

| Priority | Annotation | Scope | Typical Startup |
|---|---|---|---|
| 1 | Unit test (no Spring) | Single class, no context | < 100ms |
| 2 | `@WebMvcTest` | MVC controllers only | 1–3s |
| 3 | `@DataJpaTest` | JPA entities + repositories only | 2–5s |
| 4 | `@WebFluxTest` | Reactive controllers only | 1–3s |
| 5 | `@SpringBootTest` | Full context | 5–30s |
| 6 | `@SpringBootTest(RANDOM_PORT)` | Full context + real server | 10–30s+ |

**Decision rules**:
- `@WebMvcTest` → any REST controller test (security filter chain auto-configured)
- `@DataJpaTest` → custom JPQL/SQL queries, projections, auditing, constraints
- `@WebFluxTest` → reactive controller tests
- `@SpringBootTest` → cross-layer integration: service + security + persistence
- `@SpringBootTest(RANDOM_PORT)` → WebSocket (requires real TCP), or HTTP client integration tests

---

## Context Caching — The #1 Performance Pitfall

Spring Test caches `ApplicationContext` instances across test classes. The cache is keyed by the exact combination of configuration. Any difference creates a new context.

**Context cache killers**:

```java
// Each unique @MockitoBean combination creates a new context cache entry

// Context A (class 1):
@WebMvcTest(UserController.class)
class UserControllerTest {
    @MockitoBean UserService userService;
}

// Context B (class 2 — different mock set):
@WebMvcTest(OrderController.class)
class OrderControllerTest {
    @MockitoBean OrderService orderService;
    @MockitoBean UserService userService;   // same mock, different combination → new context
}
```

**Other cache invalidators**:
- `@DirtiesContext` — evicts the context after the test class (use only when test genuinely corrupts state)
- `@ActiveProfiles` — different profiles = different contexts
- `@Import` with different `@TestConfiguration` classes = different contexts
- `@TestPropertySource` with different properties = different contexts

**Best practice**: Standardize your mock sets. If multiple test classes all test controllers that depend on `UserService`, make them use the same `@MockitoBean` set so they share a context.

Default cache limit: 32 contexts. Beyond that, LRU eviction + recreation causes catastrophic slowdown in large test suites.

---

## @DirtiesContext — Use Sparingly

`org.springframework.test.annotation.DirtiesContext` (`@DirtiesContext`) evicts the application context after the annotated test class or method. It forces a full context rebuild for subsequent tests.

When to use:
- Test modifies `@Configuration` beans or shared static state
- Test starts background threads that pollute the context

When NOT to use:
- Test fails and you want a fresh context — fix the test instead
- Routine cleanup — use `@Transactional` rollback or `@BeforeEach` cleanup

---

## Behavior > Implementation

Across all domains, assert **observable outcomes**, not implementation details.

| Domain | Assert This (Behavior) | Not This (Implementation) |
|---|---|---|
| MVC | HTTP status code + response body fields | `verify(service).findById(1L)` |
| JPA | Re-fetched entity state after flush+clear | In-memory entity state before flush |
| Security | 401/403 status codes for unauthorized requests | Internal filter chain invocations |
| WebFlux | StepVerifier signals + emitted values | `.block()` result |
| WebSocket | Message received in `BlockingQueue` | Internal broker routing |
| Service | Return value, exception type, side-effect (email sent) | Number of repository calls |

---

## Universal Anti-Patterns

| Anti-Pattern | Why It's Bad | Fix |
|---|---|---|
| `@SpringBootTest` for every test | Slow startup, hides what you're testing | Use narrowest slice |
| Verifying mock call counts instead of behavior | Brittle — couples to implementation, breaks on refactor | Assert observable outcomes |
| String-matching JSON responses (`content().string(...)`) | Breaks on field ordering and whitespace | Use `jsonPath()` |
| Disabling security in tests (`addFilters = false`) | False confidence — you're not testing what runs in production | Test security explicitly |
| Using H2 for dialect-sensitive queries | H2 ≠ PostgreSQL — tests pass locally, fail in production | Use Testcontainers with production DB |
| `.block()` in reactive tests | Hides reactive errors, breaks scheduler assumptions | Use `StepVerifier` or `WebTestClient` |
| `Thread.sleep()` in async tests | Flaky, non-deterministic, slow | Use `Awaitility`, `BlockingQueue.poll(timeout)`, or `CountDownLatch` |
| Over-mocking value objects and DTOs | Maintenance burden, no value | Use real instances |
| `@DirtiesContext` to paper over test failures | Hides the real problem | Fix the test isolation issue instead |
| Unused stubs in `@BeforeEach` | `UnnecessaryStubbingException`, signal-to-noise ratio | Move stubs into the tests that use them |

---

## Awaitility for Async Assertions

For asynchronous operations that don't fit `BlockingQueue` or `StepVerifier`:

```java
import org.awaitility.Awaitility;
import static org.awaitility.Awaitility.*;

// Poll until condition is true, with timeout
await()
    .atMost(Duration.ofSeconds(10))
    .pollInterval(Duration.ofMillis(100))
    .untilAsserted(() -> {
        assertThat(emailSentCount.get()).isEqualTo(1);
    });

// Simple predicate poll
await()
    .atMost(Duration.ofSeconds(5))
    .until(() -> auditLog.contains("ORDER_CREATED"));
```

Awaitility is far more reliable than `Thread.sleep()` and gives clear timeout messages when the condition isn't met.

---

## Boot 4 Readiness Checklist

### Stack Requirements
- Java 21+
- Jakarta namespaces only (no `javax.*`)
- No deprecated Boot 2/3 APIs

### Annotation Migration

| Boot 3.x | Boot 4.x | New Package |
|---|---|---|
| `@MockBean` | `@MockitoBean` | `org.springframework.test.context.bean.override.mockito` |
| `@SpyBean` | `@MockitoSpyBean` | `org.springframework.test.context.bean.override.mockito` |
| `@WebMvcTest` import | Same name, new package | `org.springframework.boot.webmvc.test.autoconfigure` |
| `@DataJpaTest` import | Same name, new package | `org.springframework.boot.data.jpa.test.autoconfigure` |
| `TestEntityManager` import | Same name, new package | `org.springframework.boot.jpa.test.autoconfigure` |
| `@WebFluxTest` import | Same name, new package | `org.springframework.boot.webflux.test.autoconfigure` |

### Quick Migration Script

```bash
# Find all imports that need updating
grep -r "org.springframework.boot.test.autoconfigure.orm.jpa" src/test/
grep -r "org.springframework.boot.test.autoconfigure.web.servlet" src/test/
grep -r "org.springframework.boot.test.autoconfigure.web.reactive" src/test/
grep -r "org.springframework.boot.test.mock.mockito.MockBean" src/test/
grep -r "org.springframework.boot.test.mock.mockito.SpyBean" src/test/
```

### AOT Safety Rules

Spring Boot 4 supports AOT compilation (GraalVM native image). Test code that uses reflection hacks or classloader manipulation may not work with AOT.

- Prefer constructor injection over field injection
- Prefer explicit bean wiring over `@Autowired` field injection in test configs
- Prefer framework-provided test annotations over custom classloader manipulation
- Avoid `ReflectionTestUtils` for production code fields — fix the visibility instead

---

## Context Caching Optimization Patterns

### Pattern 1: Shared Base Test Class

```java
// Base class that all @WebMvcTest tests extend — same mock set = shared context
@WebMvcTest
abstract class AbstractWebMvcTest {
    @MockitoBean UserService userService;
    @MockitoBean OrderService orderService;
    @MockitoBean ProductService productService;
}

// Subclass narrows to specific controller — no new mocks → shares context
@WebMvcTest(UserController.class)
class UserControllerTest extends AbstractWebMvcTest { ... }

@WebMvcTest(OrderController.class)
class OrderControllerTest extends AbstractWebMvcTest { ... }
```

Trade-off: base class includes mocks that specific tests don't need. Acceptable for small suites; may not scale.

### Pattern 2: @TestConfiguration Shared Config

```java
// Shared test infrastructure in one place
@TestConfiguration
public class SharedTestConfig {
    @Bean @Primary
    public Clock testClock() { return Clock.fixed(Instant.parse("2026-01-01T00:00:00Z"), ZoneOffset.UTC); }
}

// Import everywhere — same config class = same cache key contribution
@WebMvcTest(UserController.class)
@Import(SharedTestConfig.class)
class UserControllerTest { ... }
```

---

## Test Naming Conventions

Consistent naming makes failures readable:

```
methodUnderTest_scenarioDescription_expectedBehavior

findById_withValidId_returnsUser()
findById_withUnknownId_throwsNotFoundException()
createOrder_whenStockInsufficient_throwsOutOfStockException()
getOrder_withoutAuth_returns401()
getOrder_asUser_withDifferentOwner_returns403()
```

The pattern: what you're testing (`findById`), the condition (`withValidId`), the expectation (`returnsUser`).
