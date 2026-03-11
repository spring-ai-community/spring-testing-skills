# JPA Testing: @DataJpaTest Patterns, @Transactional Traps, and Test Data

Core patterns for writing `org.springframework.boot.data.jpa.test.autoconfigure.DataJpaTest` (`@DataJpaTest`) tests using `org.springframework.boot.jpa.test.autoconfigure.TestEntityManager` (`TestEntityManager`). Covers flush/clear cycles, the `@Transactional` trap, test data setup, `@Modifying` bulk updates, and what not to test.

---

## The Flush+Clear Pattern

The most common mistake in `@DataJpaTest` tests: saving data and reading it back without flushing and clearing.

```java
import jakarta.persistence.EntityManager;
import org.springframework.boot.data.jpa.test.autoconfigure.DataJpaTest;
import org.springframework.boot.jpa.test.autoconfigure.TestEntityManager;
import org.springframework.beans.factory.annotation.Autowired;

@DataJpaTest
class OrderRepositoryTest {

    @Autowired TestEntityManager em;
    @Autowired OrderRepository orderRepository;

    @Test
    void findByCustomerId_returnsMatchingOrders() {
        Customer customer = em.persist(new Customer("alice@example.com"));
        em.persist(new Order(customer, OrderStatus.PENDING));
        em.persist(new Order(customer, OrderStatus.SHIPPED));
        em.flush();   // Send SQL to the database
        em.clear();   // Evict L1 cache — forces repository read to hit the DB

        List<Order> orders = orderRepository.findByCustomerId(customer.getId());

        assertThat(orders).hasSize(2);
    }
}
```

**Why this matters**: Without `flush()`, SQL may never reach the database. Without `clear()`, `findByCustomerId` returns the entity from the first-level cache and never hits the database — your query is never exercised.

Use `persistAndFlush()` as a shorthand when you only need the flush:

```java
User saved = em.persistAndFlush(new User("alice@example.com"));
em.clear(); // Still need clear() to evict from L1 cache
```

---

## The @Transactional-in-Tests Trap

Multiple authoritative sources warn against wrapping integration tests in `@Transactional`:

### Why It Causes Problems

1. **Lazy loading works in tests, fails in production**: The test transaction keeps the `EntityManager` open, allowing lazy collections to load — masking `LazyInitializationException` that production code would hit.

2. **Hibernate may not flush**: Since the transaction is rolled back, `FlushModeType.AUTO` may never trigger. Constraint violations and unique index violations go undetected.

3. **Auto-dirty-checking false positives**: Entities modified within a `@Transactional` test are automatically persisted at flush time without calling `save()`. This would not happen in production if the service layer doesn't explicitly save.

4. **Different transaction boundaries**: Production commits at service method boundaries. Test-level `@Transactional` merges everything into one transaction, hiding multi-transaction bugs.

### When @Transactional IS Appropriate in Tests

- `@DataJpaTest` for isolated query testing (rollback is convenient, transactions are scoped)
- Tests with explicit `flush()` calls — you force SQL execution and detect constraint violations
- Quick verification of derived query methods where production transaction behavior is not the concern

### When to AVOID @Transactional in Tests

- Full integration tests verifying service-layer transaction boundaries
- Tests for `@Modifying` bulk update queries
- Tests that verify lazy loading behavior (see `references/lazy-loading-tests.md`)
- Tests for optimistic locking (`@Version`)

**Sources**:
- [Nurkiewicz — Spring Pitfalls: Transactional Tests Considered Harmful](https://nurkiewicz.com/2011/11/spring-pitfalls-transactional-tests.html)
- [rieckpil — Spring Boot Testing Pitfall: Transaction Rollback](https://rieckpil.de/spring-boot-testing-pitfall-transaction-rollback-in-tests/)
- [Marco Behler — Should My Tests Be @Transactional?](https://www.marcobehler.com/2014/06/25/should-my-tests-be-transactional)

---

## Core Test Patterns

### Testing Derived Query Methods

```java
@Test
void findByEmailAndActiveTrue_returnsOnlyActiveUsers() {
    em.persist(new User("bob@example.com", true));
    em.persist(new User("bob@example.com", false));
    em.flush();
    em.clear();

    List<User> results = userRepository.findByEmailAndActiveTrue("bob@example.com");

    assertThat(results).hasSize(1);
    assertThat(results.get(0).isActive()).isTrue();
}

@Test
void findTop3ByOrderByCreatedAtDesc_returnsLatestThree() {
    for (int i = 0; i < 5; i++) {
        em.persist(new Product("P" + i, BigDecimal.TEN));
    }
    em.flush();
    em.clear();

    List<Product> top3 = productRepository.findTop3ByOrderByCreatedAtDesc();
    assertThat(top3).hasSize(3);
}
```

### Testing Custom @Query Methods

```java
// Repository method:
// @Query("SELECT o FROM Order o WHERE o.status = :status AND o.total >= :minTotal")
// List<Order> findByStatusAndMinTotal(@Param("status") OrderStatus status,
//                                     @Param("minTotal") BigDecimal minTotal);

@Test
void findByStatusAndMinTotal_filtersCorrectly() {
    Customer c = em.persist(new Customer("alice@example.com"));
    em.persist(orderWith(c, OrderStatus.PENDING, new BigDecimal("50.00")));
    em.persist(orderWith(c, OrderStatus.PENDING, new BigDecimal("150.00")));
    em.persist(orderWith(c, OrderStatus.SHIPPED, new BigDecimal("200.00")));
    em.flush();
    em.clear();

    List<Order> results = orderRepository
        .findByStatusAndMinTotal(OrderStatus.PENDING, new BigDecimal("100.00"));

    assertThat(results).hasSize(1);
    assertThat(results.get(0).getTotal()).isEqualByComparingTo("150.00");
}
```

### Testing Pagination

```java
@Test
void findAll_paginated_returnsCorrectPage() {
    for (int i = 0; i < 10; i++) {
        em.persist(new Product("P" + i, BigDecimal.TEN));
    }
    em.flush();
    em.clear();

    Pageable pageable = PageRequest.of(0, 3, Sort.by("name").ascending());
    Page<Product> page = productRepository.findAll(pageable);

    assertThat(page.getContent()).hasSize(3);
    assertThat(page.getTotalElements()).isEqualTo(10);
    assertThat(page.getTotalPages()).isEqualTo(4);
    assertThat(page.getContent().get(0).getName()).isEqualTo("P0");
}
```

### Testing Specifications

```java
@Test
void spec_filtersByCategory() {
    em.persist(new Product("Widget", "ELECTRONICS", BigDecimal.TEN));
    em.persist(new Product("Screw", "HARDWARE", BigDecimal.ONE));
    em.flush();
    em.clear();

    List<Product> results = productRepository
        .findAll(ProductSpecs.hasCategory("ELECTRONICS"));

    assertThat(results).hasSize(1);
    assertThat(results.get(0).getName()).isEqualTo("Widget");
}
```

### Testing Interface Projections

```java
// interface UserSummary { String getEmail(); String getDisplayName(); }
// List<UserSummary> findByActiveTrue();

@Test
void findByActiveTrue_returnsProjection() {
    em.persist(new User("alice@example.com", "Alice Smith", true));
    em.persist(new User("bob@example.com", "Bob Jones", false));
    em.flush();
    em.clear();

    List<UserSummary> summaries = userRepository.findByActiveTrue();

    assertThat(summaries).hasSize(1);
    assertThat(summaries.get(0).getEmail()).isEqualTo("alice@example.com");
    assertThat(summaries.get(0).getDisplayName()).isEqualTo("Alice Smith");
}
```

### Testing Entity Auditing

```java
import org.springframework.boot.data.jpa.test.autoconfigure.DataJpaTest;
import org.springframework.boot.test.context.TestConfiguration;
import org.springframework.context.annotation.Import;
import org.springframework.data.jpa.repository.config.EnableJpaAuditing;

@DataJpaTest
@Import(AuditedEntityTest.AuditingConfig.class)
class AuditedEntityTest {

    @TestConfiguration
    @EnableJpaAuditing
    static class AuditingConfig {}

    @Autowired TestEntityManager em;
    @Autowired ProductRepository productRepository;

    @Test
    void save_populatesAuditDates() {
        Product p = productRepository.save(new Product("Widget", BigDecimal.TEN));
        em.flush();
        em.clear();

        Product loaded = productRepository.findById(p.getId()).orElseThrow();
        assertThat(loaded.getCreatedAt()).isNotNull();
        assertThat(loaded.getUpdatedAt()).isNotNull();
    }
}
```

**Note**: `@EnableJpaAuditing` is not loaded by `@DataJpaTest` by default — you must import a `@TestConfiguration` that enables it.

### Constraint Violation Testing

```java
@Test
void save_duplicateEmail_throwsConstraintViolation() {
    em.persist(new User("alice@example.com"));
    em.flush();

    assertThatThrownBy(() -> {
        em.persist(new User("alice@example.com"));
        em.flush(); // flush() triggers the database constraint check
    }).isInstanceOf(DataIntegrityViolationException.class);
}
```

**Key**: The second `em.flush()` is required — without it, Hibernate may batch the insert and you may not see the violation.

### Transaction Rollback Control

```java
// @DataJpaTest wraps each test in @Transactional → rollback after each test.
// Override when you need a real commit:

@Test
@Commit   // Commits the transaction — use sparingly, clean up manually
void saveOrder_commitsSuccessfully() { /* ... */ }

// Or disable the wrapping transaction entirely:
import org.springframework.transaction.annotation.Propagation;
import org.springframework.transaction.annotation.Transactional;

@DataJpaTest
@Transactional(propagation = Propagation.NOT_SUPPORTED)
class OrderRepositoryCommitTest {
    // Must clean up test data manually in @BeforeEach
    @BeforeEach
    void cleanUp() {
        orderRepository.deleteAll();
    }
}
```

---

## Test Data Setup and Cleanup

### Vlad Mihalcea's Approach: Truncate Before Each Test

> "Execute cleanup **before** each test, not after. Cleanup in `@AfterEach` may not run during test failures or debugging sessions."

Since Hibernate 6.2, use the `SchemaManager` API:

```java
import org.hibernate.engine.spi.SessionFactoryImplementor;

@Autowired EntityManagerFactory entityManagerFactory;

@BeforeEach
void cleanUp() {
    entityManagerFactory
        .unwrap(SessionFactoryImplementor.class)
        .getSchemaManager()
        .truncateMappedObjects();
}
```

Benefits: truncates all tables mapped to JPA entities, respects foreign key order automatically, works with any database.

**Source**: [Vlad Mihalcea — The best way to clean up test data](https://vladmihalcea.com/clean-up-test-data-spring/)

### @Sql for Complex Data Setup

For complex test data, use `@Sql` to load SQL scripts:

```java
import org.springframework.test.context.jdbc.Sql;

@Test
@Sql("/test-data/users.sql")
void complexQuery_shouldReturnExpectedResults() { ... }
```

Caveat: SQL scripts need maintenance when schemas change. Prefer programmatic setup for resilience.

### Do NOT Use the Repository Under Test for Setup

```java
// WRONG: circular logic — repository bug masks both setup and assertion failure
userRepository.save(new User("alice@example.com"));
List<User> users = userRepository.findByActiveTrue();

// CORRECT: use TestEntityManager for setup
em.persistAndFlush(new User("alice@example.com", true));
em.clear();
List<User> users = userRepository.findByActiveTrue();
```

Using the same repository for both setup and assertion creates circular logic that masks bugs in the method under test.

**Source**: [Arho Huttunen — Testing the Persistence Layer](https://www.arhohuttunen.com/spring-boot-datajpatest/)

---

## Testing @Modifying Bulk Updates

### The Pattern

```java
import org.springframework.data.jpa.repository.Modifying;
import org.springframework.data.jpa.repository.Query;
import org.springframework.transaction.annotation.Transactional;

@Modifying
@Transactional  // REQUIRED when extending Repository (not JpaRepository)
@Query("UPDATE User u SET u.status = :status WHERE u.id = :id")
int updateStatus(@Param("id") Long id, @Param("status") String status);
```

When extending `org.springframework.data.repository.Repository` (not `JpaRepository`), `@Modifying` queries do NOT inherit `@Transactional`. Without it, you get:

```
InvalidDataAccessApiUsageException: Executing an update/delete query
```

### Testing Bulk Updates

```java
@Test
void updateStatus_shouldModifyEntity() {
    User user = em.persistAndFlush(new User("Alice", "ACTIVE"));
    em.clear();

    int updated = userRepository.updateStatus(user.getId(), "INACTIVE");

    assertThat(updated).isEqualTo(1);
    em.clear(); // Clear stale cached state after bulk update
    User reloaded = em.find(User.class, user.getId());
    assertThat(reloaded.getStatus()).isEqualTo("INACTIVE");
}
```

Steps:
1. Insert test data via `TestEntityManager`
2. Execute the bulk update
3. `em.clear()` — bulk updates bypass Hibernate dirty-checking; the L1 cache holds stale state
4. Re-read and assert

### clearAutomatically for Safety

`@Modifying(clearAutomatically = true)` clears the persistence context automatically after the bulk update:

```java
@Modifying(clearAutomatically = true)
@Transactional
@Query("UPDATE StepExecution s SET s.itemCount = :count, s.version = s.version + 1 WHERE s.id = :id")
int updateItemCount(@Param("id") UUID id, @Param("count") long count);
```

### Testing Version Increment in Bulk Updates

JPQL bulk updates bypass Hibernate's dirty-checking and optimistic lock version increment. If you manage the version field manually in the query, verify it:

```java
@Test
void bulkUpdate_shouldIncrementVersion() {
    StepExecution step = createAndSaveStep();
    int initialVersion = step.getVersion();

    repository.updateItemCount(step.getId(), 42);
    em.clear();

    StepExecution reloaded = em.find(StepExecution.class, step.getId());
    assertThat(reloaded.getVersion()).isEqualTo(initialVersion + 1);
    assertThat(reloaded.getItemCount()).isEqualTo(42);
}
```

**Sources**:
- [Thorben Janssen — Implementing Bulk Updates with Spring Data JPA](https://thorben-janssen.com/implementing-bulk-updates-with-spring-data-jpa/)
- [Baeldung — Spring Data JPA @Modifying Annotation](https://www.baeldung.com/spring-data-jpa-modifying-annotation)

---

## What NOT to Test

### Skip These (Framework-Validated)

| What | Why |
|------|-----|
| Inherited CRUD: `save()`, `findById()`, `delete()` | Tested by the Spring Data JPA team |
| Derived query compilation: `findByName()`, `findByStatusAndType()` | Spring Data validates at application startup |
| Schema generation | Use Flyway migrations instead; test migration execution as startup validation |

### DO Test These

- **Custom `@Query` methods** — both JPQL and native SQL; native queries are especially fragile
- **`@Modifying` queries** — bypass dirty-checking, must be verified
- **Projection interfaces/DTOs** — field mapping errors are common
- **Database constraints** — unique constraints, foreign keys, check constraints
- **Aggregate persistence round-trips** — save + clear + reload + verify all fields
- **Optimistic locking** — concurrent modification detection via `@Version`
- **Inheritance discriminator values** — correct subtype storage and retrieval

**Sources**:
- [Arho Huttunen — Testing the Persistence Layer](https://www.arhohuttunen.com/spring-boot-datajpatest/)
- [Reflectoring.io — Testing JPA Queries with @DataJpaTest](https://reflectoring.io/spring-boot-data-jpa-test/)

---

## Anti-Patterns Summary

| Anti-Pattern | Fix |
|---|---|
| `save()` then `findById()` without `flush()+clear()` | Always flush+clear before read assertions |
| Testing `save()` and `findById()` only | Test YOUR query methods, not Spring Data plumbing |
| Loading `@SpringBootTest` for isolated repository tests | Use `@DataJpaTest` — 90% less infrastructure |
| Using H2 for native queries with PostgreSQL syntax | Use Testcontainers with production DB image |
| Using the repository under test to set up test data | Use `TestEntityManager` for setup |
| Assuming `@Transactional` test behavior matches production | It doesn't — lazy loading, flush timing, and transaction boundaries differ |
