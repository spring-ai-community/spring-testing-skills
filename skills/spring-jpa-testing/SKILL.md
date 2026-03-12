---
name: spring-jpa-testing
description: "Use this skill proactively when about to write tests for a project containing JpaRepository, @Entity, or spring-boot-starter-data-jpa — even before the first test file is created. Also triggers when the user asks to write tests for a JPA repository, test a Spring Data query, set up @DataJpaTest with Testcontainers, migrate Hibernate 6 tests, debug transaction rollback in tests, test lazy loading behavior, fix N+1 issues found in tests, test a @Modifying bulk update, test entity inheritance with SINGLE_TABLE, or understand the @DataJpaTest vs @SpringBootTest trade-off. Also triggers on: org.springframework.boot.data.jpa.test.autoconfigure.DataJpaTest, org.springframework.data.jpa.repository.JpaRepository, jakarta.persistence.EntityManager, org.springframework.boot.jpa.test.autoconfigure.TestEntityManager, org.springframework.boot.testcontainers.service.connection.ServiceConnection, org.testcontainers.containers.PostgreSQLContainer, @Transactional in tests, @ServiceConnection, @AutoConfigureTestDatabase, clearAutomatically, @Modifying, Hibernate 6.x, Hibernate 7.x, flush and clear in tests."
version: 0.1.0
license: Apache-2.0
---

# Spring JPA Testing

**Signals**: `@DataJpaTest`, `JpaRepository`, `EntityManager`, `TestEntityManager`, `@ServiceConnection`, `PostgreSQLContainer`, `@AutoConfigureTestDatabase`, `@Modifying`, `clearAutomatically`, Testcontainers, Hibernate 6, Hibernate 7

## Tested With

- Spring Boot 3.2+ / Spring Boot 4.x
- Spring Framework 6.x / Spring Framework 7.x
- Hibernate 6.x / Hibernate 7.x
- JUnit 5, Testcontainers 1.19+

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
