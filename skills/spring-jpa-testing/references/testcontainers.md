# JPA Testing: Testcontainers and @DataJpaTest vs @SpringBootTest

Covers the `@ServiceConnection` pattern for Testcontainers, why H2 is insufficient for production-like testing, and when to choose `org.springframework.boot.data.jpa.test.autoconfigure.DataJpaTest` (`@DataJpaTest`) versus `org.springframework.boot.test.context.SpringBootTest` (`@SpringBootTest`).

---

## Why Not H2

> "You should use the same database engine that you also run in production to validate the very same behavior that the users are going to experience."
> — Vlad Mihalcea

Problems with H2 as a test database:

- Cannot replicate PostgreSQL-specific features: window functions, `jsonb`, array types, advisory locks
- SQL syntax differences — tests pass on H2 but fail on PostgreSQL with native queries
- Flyway migrations often use PostgreSQL-specific DDL and fail silently on H2
- Sequence generation behavior differs between engines

**Recommendation**: Use Testcontainers with the same database image as production.

**Sources**:
- [Vlad Mihalcea — Testcontainers Database Integration Testing](https://vladmihalcea.com/testcontainers-database-integration-testing/)
- [Testcontainers — Replace H2 with a Real Database](https://testcontainers.com/guides/replace-h2-with-real-database-for-testing/)

---

## @DataJpaTest vs @SpringBootTest: When to Use Which

### What @DataJpaTest Does

`org.springframework.boot.data.jpa.test.autoconfigure.DataJpaTest` (`@DataJpaTest`) creates a slice of the Spring application context containing only JPA-related beans:

- Repositories, `EntityManager`, `DataSource`, `JdbcTemplate`
- Auto-configures `org.springframework.boot.jpa.test.autoconfigure.TestEntityManager` (`TestEntityManager`)
- Does NOT load `@Service`, `@Controller`, or non-JPA beans
- Wraps every test method in `@Transactional` with automatic rollback
- Validates entity mappings and JPQL query syntax at startup

By default, `@DataJpaTest` replaces the datasource with an embedded H2 database. Use `@AutoConfigureTestDatabase(replace = AutoConfigureTestDatabase.Replace.NONE)` (from `org.springframework.boot.data.jpa.test.autoconfigure.AutoConfigureTestDatabase`) to disable this replacement when using Testcontainers.

### What @SpringBootTest Does

`org.springframework.boot.test.context.SpringBootTest` (`@SpringBootTest`) loads the full application context. It does NOT add `@Transactional` by default — transactions commit normally unless you explicitly add `@Transactional` to the test class.

### Decision Table

| Scenario | Recommendation |
|----------|---------------|
| Testing custom JPQL/native queries in isolation | `@DataJpaTest` with Testcontainers |
| Testing repository + service transaction boundaries | `@SpringBootTest` |
| Testing Flyway migrations + entity mappings | `@DataJpaTest` with real DB |
| Testing DDD aggregate persistence through service layer | `@SpringBootTest` with Testcontainers |
| Project uses PostgreSQL-specific SQL | Must use Testcontainers (not H2) |
| Verifying lazy loading fails outside a transaction | `@SpringBootTest` (no auto-`@Transactional`) |

### Vlad Mihalcea's Case Against @DataJpaTest

Vlad Mihalcea recommends caution with `@DataJpaTest` for integration-style tests:

1. **Automatic `@Transactional` wrapper creates unrealistic conditions**: Service-layer transactions manage production boundaries. Test-level transactions change flush timing and dirty-checking behavior.
2. **Flush behavior mismatch**: `FlushModeType.AUTO` triggers flush before commit. In a rolled-back test transaction, SQL statements may never be sent to the database — constraint violations go undetected.
3. **Masks `LazyInitializationException`**: The test transaction keeps the `EntityManager` open for the entire test, allowing lazy collections to load that would fail in production.

Use `@DataJpaTest` for what it's designed for: isolated query correctness. Use `@SpringBootTest` when verifying production transaction behavior.

**Sources**:
- [Vlad Mihalcea — The best way to clean up test data](https://vladmihalcea.com/clean-up-test-data-spring/)
- [Arho Huttunen — Testing the Persistence Layer with @DataJpaTest](https://www.arhohuttunen.com/spring-boot-datajpatest/)
- [rieckpil — Spring Data JPA Persistence Layer Tests with @DataJpaTest](https://rieckpil.de/test-your-spring-boot-jpa-persistence-layer-with-datajpatest/)

---

## Modern Pattern: @ServiceConnection (Spring Boot 3.1+)

Since Spring Boot 3.1, the recommended approach uses `@TestConfiguration` with `@Bean @ServiceConnection`. No `@DynamicPropertySource` needed.

```java
import org.springframework.boot.test.context.TestConfiguration;
import org.springframework.boot.testcontainers.service.connection.ServiceConnection;
import org.springframework.context.annotation.Bean;
import org.testcontainers.containers.PostgreSQLContainer;

@TestConfiguration(proxyBeanMethods = false)
public class TestcontainersConfiguration {

    @Bean
    @ServiceConnection
    PostgreSQLContainer<?> postgresContainer() {
        return new PostgreSQLContainer<>("postgres:16-alpine")
                .withReuse(true);
    }
}
```

`@ServiceConnection` auto-configures `spring.datasource.url`, `spring.datasource.username`, and `spring.datasource.password` from the running container — no manual property wiring.

### Usage with @SpringBootTest

```java
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.context.annotation.Import;

@SpringBootTest
@Import(TestcontainersConfiguration.class)
class MyIntegrationTest {
    // Full application context, Testcontainers datasource
}
```

### Usage with @DataJpaTest (requires replace=NONE)

```java
import org.springframework.boot.data.jpa.test.autoconfigure.AutoConfigureTestDatabase;
import org.springframework.boot.data.jpa.test.autoconfigure.DataJpaTest;
import org.springframework.context.annotation.Import;

@DataJpaTest
@AutoConfigureTestDatabase(replace = AutoConfigureTestDatabase.Replace.NONE)
@Import(TestcontainersConfiguration.class)
class MyRepositoryTest {
    // JPA slice only, Testcontainers datasource
}
```

**Critical**: `@AutoConfigureTestDatabase(replace = NONE)` is required with `@DataJpaTest` when using Testcontainers. Without it, Spring replaces your Testcontainers `DataSource` with H2.

### Container Lifecycle

- Containers declared as `static` beans in `@TestConfiguration` are started once per JVM and shared across all tests in the suite — this is the recommended pattern for CI speed
- `withReuse(true)` keeps containers alive between test runs for faster local development (requires `testcontainers.reuse.enable=true` in `~/.testcontainers.properties`)
- Static containers started this way are automatically stopped by the Testcontainers `ResourceReaper` on JVM shutdown

### Older Pattern: @DynamicPropertySource (still valid)

Before `@ServiceConnection`, the standard approach used `@DynamicPropertySource`:

```java
import org.springframework.boot.data.jpa.test.autoconfigure.AutoConfigureTestDatabase;
import org.springframework.boot.data.jpa.test.autoconfigure.DataJpaTest;
import org.springframework.test.context.DynamicPropertyRegistry;
import org.springframework.test.context.DynamicPropertySource;
import org.testcontainers.containers.PostgreSQLContainer;
import org.testcontainers.junit.jupiter.Container;
import org.testcontainers.junit.jupiter.Testcontainers;

@DataJpaTest
@AutoConfigureTestDatabase(replace = AutoConfigureTestDatabase.Replace.NONE)
@Testcontainers
class OrderRepositoryTest {

    @Container
    static PostgreSQLContainer<?> postgres =
            new PostgreSQLContainer<>("postgres:16-alpine");

    @DynamicPropertySource
    static void configureProperties(DynamicPropertyRegistry registry) {
        registry.add("spring.datasource.url", postgres::getJdbcUrl);
        registry.add("spring.datasource.username", postgres::getUsername);
        registry.add("spring.datasource.password", postgres::getPassword);
    }
}
```

Prefer `@ServiceConnection` for new code — less boilerplate and works with Spring context caching.

---

## Separate @TestConfiguration for Slice vs Full Tests

Wim Deblauwe recommends creating separate `@TestConfiguration` classes:

- **Database-only configuration** for `@DataJpaTest` — starts only the database container
- **Full configuration** for `@SpringBootTest` — starts database plus any other containers (Redis, Kafka, etc.)

This avoids starting unnecessary containers for slice tests:

```java
// For @DataJpaTest only
@TestConfiguration(proxyBeanMethods = false)
public class DatabaseTestConfiguration {
    @Bean
    @ServiceConnection
    PostgreSQLContainer<?> postgresContainer() {
        return new PostgreSQLContainer<>("postgres:16-alpine");
    }
}

// For @SpringBootTest with full infrastructure
@TestConfiguration(proxyBeanMethods = false)
public class FullInfrastructureTestConfiguration {
    @Bean
    @ServiceConnection
    PostgreSQLContainer<?> postgresContainer() {
        return new PostgreSQLContainer<>("postgres:16-alpine");
    }

    @Bean
    @ServiceConnection
    RedisContainer redisContainer() {
        return new RedisContainer("redis:7-alpine");
    }
}
```

**Source**: [Wim Deblauwe — How I Test Production-Ready Spring Boot Applications](https://www.wimdeblauwe.com/blog/2025/07/30/how-i-test-production-ready-spring-boot-applications/)

---

## Context Caching with Testcontainers

Spring caches application contexts across tests. Two tests that import the same `@TestConfiguration` (e.g., `TestcontainersConfiguration`) will reuse the same container and context — this is why static beans are key.

Avoid `@DirtiesContext` unless absolutely necessary — it discards the cached context and forces a full restart. Use it only when your test genuinely modifies shared infrastructure state.

**Sources**:
- [JetBrains Blog — Testing Spring Boot Applications Using Testcontainers](https://blog.jetbrains.com/idea/2024/12/testing-spring-boot-applications-using-testcontainers/)
- [Spring Boot Issue #35121 — Document @ServiceConnection for @DataJpaTest](https://github.com/spring-projects/spring-boot/issues/35121)
