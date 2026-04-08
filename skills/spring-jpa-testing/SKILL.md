---
name: spring-jpa-testing
description: "@DataJpaTest requires TestEntityManager + flush()/clear() before assertions — without this, tests read from Hibernate L1 cache and pass falsely. Read before writing any JPA test. Triggers: @DataJpaTest, JpaRepository, TestEntityManager, @ServiceConnection, Testcontainers, @Modifying, Hibernate 6/7, @AutoConfigureTestDatabase, flush and clear in tests, @Transactional in tests, jakarta.persistence.EntityManager."
version: 0.1.0
license: Apache-2.0
---

# Spring JPA Testing

CRITICAL: `@DataJpaTest` requires `TestEntityManager` for test data setup and `flush()`+`clear()` before read assertions. Tests that skip this read from the Hibernate L1 cache and produce false-passing tests — the query is never executed against the database. See "Critical Rules" below.

**Signals**: `@DataJpaTest`, `JpaRepository`, `EntityManager`, `TestEntityManager`, `@ServiceConnection`, `PostgreSQLContainer`, `@AutoConfigureTestDatabase`, `@Modifying`, `clearAutomatically`, Testcontainers, Hibernate 6, Hibernate 7

## Tested With

- Spring Boot 3.2+ / Spring Boot 4.x
- Spring Framework 6.x / Spring Framework 7.x
- Hibernate 6.x / Hibernate 7.x
- JUnit 5, Testcontainers 1.19+

## Critical Rules — Always Apply in @DataJpaTest

**1. Use TestEntityManager for test data setup — never Mockito mocks.**
`@DataJpaTest` provides a real in-memory database. Mocking the repository defeats the purpose.
Use `@Autowired TestEntityManager em` and `em.persist()` / `em.persistAndFlush()` to insert test data.
Do NOT use the repository under test for setup — that creates circular logic where a bug in the repository masks both setup and assertion failures.

**2. Always flush() and clear() before read assertions.**
After persisting test data, call `em.flush()` then `em.clear()` before querying through the repository.
Without flush, SQL may never reach the database. Without clear, the repository returns entities from
the Hibernate L1 cache — your query is never actually executed against the database.

```java
@DataJpaTest
class OrderRepositoryTest {
    @Autowired TestEntityManager em;
    @Autowired OrderRepository orders;

    @Test
    void findByStatus_returnsMatching() {
        em.persist(new Order("pending"));
        em.persist(new Order("shipped"));
        em.flush();   // SQL hits the database
        em.clear();   // L1 cache evicted — forces real DB read

        List<Order> result = orders.findByStatus("pending");
        assertThat(result).hasSize(1);
    }
}
```

See `references/transactional-tests.md` for full patterns including @Modifying bulk updates, constraint violation testing, and the @Transactional trap.

## Do NOT Use This Skill When

- Testing REST controllers or HTTP endpoints → use `spring-mvc-testing`
- Testing authentication, authorization, CSRF, JWT, or OAuth2 → use `spring-security-testing`
- Testing reactive repositories (R2DBC) or WebClient → use `spring-webflux-testing`
- Writing pure unit tests without a Spring context → use `spring-testing-fundamentals`
- Setting up AssertJ assertions or Mockito stubs → use `spring-testing-fundamentals`

## When to Read References

| Situation | Read |
|-----------|------|
| Setting up `@DataJpaTest` with Testcontainers and `@ServiceConnection` | `references/testcontainers.md` |
| Choosing between `@DataJpaTest` and `@SpringBootTest` | `references/testcontainers.md` |
| Debugging `@Transactional` rollback issues, flush/clear problems, test isolation | `references/transactional-tests.md` |
| Writing test patterns: derived queries, `@Query`, pagination, specs, projections, auditing | `references/transactional-tests.md` |
| Lazy loading works in test but fails in production, N+1 issues | `references/lazy-loading-tests.md` |
| Testing `SINGLE_TABLE` inheritance, read-only repositories, `@QueryHint` pitfalls | `references/lazy-loading-tests.md` |
| Hibernate 6.x or 7.x migration failures, Jakarta packages, sequence naming, `ClassCastException` | `references/hibernate-6-migration.md` |
| Boot 3.x → 4.x test package import changes | `references/hibernate-6-migration.md` |
