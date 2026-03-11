# JPA Testing: Lazy Loading, Entity Lifecycle, Inheritance, and Read-Only Repositories

Covers lazy loading behavior in `org.springframework.boot.data.jpa.test.autoconfigure.DataJpaTest` (`@DataJpaTest`) tests vs production, N+1 detection, testing `SINGLE_TABLE` inheritance, and the `@QueryHint(HINT_READ_ONLY)` ClassCastException pitfall with read-only repositories.

---

## Lazy Loading in @DataJpaTest: The Hidden Trap

```java
import org.springframework.boot.data.jpa.test.autoconfigure.DataJpaTest;
import org.springframework.boot.jpa.test.autoconfigure.TestEntityManager;

@DataJpaTest
class OrderRepositoryTest {

    @Autowired TestEntityManager em;
    @Autowired OrderRepository orderRepository;

    @Test
    void findOrderWithItems_lazyLoadingInTransaction() {
        Customer customer = em.persist(new Customer("alice@example.com"));
        Order order = em.persist(new Order(customer));
        em.persist(new OrderItem(order, "PROD-1", 2));
        em.persist(new OrderItem(order, "PROD-2", 1));
        em.flush();
        em.clear();

        Order loaded = orderRepository.findById(order.getId()).orElseThrow();

        // This WORKS because @DataJpaTest wraps the test in @Transactional,
        // keeping the EntityManager open for the entire test.
        // In production without a transaction, this would throw LazyInitializationException.
        assertThat(loaded.getItems()).hasSize(2);
    }
}
```

**The problem**: `@OneToMany(fetch = LAZY)` collections load successfully in a `@DataJpaTest` because the test-level `@Transactional` keeps the `EntityManager` open. This masks `LazyInitializationException` that production code would encounter when accessed outside a transaction.

### Testing That LazyInitializationException IS Thrown in Production

To verify production-like behavior, use `@SpringBootTest` without `@Transactional`:

```java
import org.springframework.boot.test.context.SpringBootTest;

@SpringBootTest
class OrderLazyLoadingTest {

    @Autowired OrderRepository orderRepository;

    @Test
    void findOrder_accessingItemsOutsideTransaction_throwsLazyInitializationException() {
        // Setup: save order with items (done in a service or @Transactional helper)
        UUID savedId = createOrderWithItems();

        // Loading in @SpringBootTest does NOT wrap in @Transactional
        Order order = orderRepository.findById(savedId).orElseThrow();

        // Accessing lazy collection outside transaction should throw
        assertThatThrownBy(() -> order.getItems().size())
            .isInstanceOf(org.hibernate.LazyInitializationException.class);
    }
}
```

---

## N+1 Detection in Tests

N+1 problems are hard to see in `@DataJpaTest` because the test has one entity. Use a test with enough data to make the SQL count visible, combined with a query counter.

### Detecting N+1 with DataSource-Proxy

A SQL counter using DataSource-Proxy can assert the number of queries:

```java
import net.ttddyy.dsproxy.asserts.ProxyTestDataSource;

// In @BeforeEach, wrap the DataSource with DataSource-Proxy
// Then in the test:
@Test
void findOrders_noNPlusOne() {
    // Given: 10 orders with customers
    for (int i = 0; i < 10; i++) {
        Customer c = em.persist(new Customer("user" + i + "@example.com"));
        em.persist(new Order(c, OrderStatus.PENDING));
    }
    em.flush();
    em.clear();

    proxyDataSource.reset();
    List<Order> orders = orderRepository.findAllWithCustomer();

    // Should be 1 query (join fetch), not 11 (1 + N)
    assertThat(proxyDataSource.getQueryExecutions()).hasSize(1);
}
```

### Using JOIN FETCH to Solve N+1

```java
// Repository method using JOIN FETCH to prevent N+1:
@Query("SELECT o FROM Order o JOIN FETCH o.customer WHERE o.status = :status")
List<Order> findAllWithCustomer(@Param("status") OrderStatus status);
```

**Caveat**: Avoid `JOIN FETCH` with `SINGLE_TABLE` inheritance across multiple branches — see Hibernate 6.x `ClassCastException` in `references/hibernate-6-migration.md`.

---

## Testing Entity Relationships

### @OneToMany: Cascade and Orphan Removal

```java
@Test
void saveOrderWithItems_cascadesPersist() {
    Customer customer = em.persistAndFlush(new Customer("alice@example.com"));
    Order order = new Order(customer);
    order.addItem(new OrderItem("PROD-1", 2));
    order.addItem(new OrderItem("PROD-2", 1));

    orderRepository.save(order);
    em.flush();
    em.clear();

    Order loaded = orderRepository.findById(order.getId()).orElseThrow();
    // Access within transaction — safe in @DataJpaTest
    assertThat(loaded.getItems()).hasSize(2);
}

@Test
void deleteOrder_cascadesOrphanRemoval() {
    // Setup: order with items
    UUID orderId = createOrderWithItems(2);
    em.flush();
    em.clear();

    orderRepository.deleteById(orderId);
    em.flush();
    em.clear();

    // Verify items were also removed (CascadeType.REMOVE / orphanRemoval=true)
    assertThat(orderItemRepository.findByOrderId(orderId)).isEmpty();
}
```

### @ManyToOne: Foreign Key Integrity

```java
@Test
void persist_withNullMandatoryRelationship_throwsConstraintViolation() {
    Order order = new Order(null, OrderStatus.PENDING); // null customer is NOT NULL constraint

    assertThatThrownBy(() -> {
        em.persist(order);
        em.flush();
    }).isInstanceOf(org.springframework.dao.DataIntegrityViolationException.class);
}
```

---

## Testing SINGLE_TABLE Inheritance

### How SINGLE_TABLE Works

```java
import jakarta.persistence.DiscriminatorColumn;
import jakarta.persistence.DiscriminatorValue;
import jakarta.persistence.Entity;
import jakarta.persistence.Inheritance;
import jakarta.persistence.InheritanceType;

@Entity
@Inheritance(strategy = InheritanceType.SINGLE_TABLE)
@DiscriminatorColumn(name = "type")
public abstract class JobExecution { ... }

@Entity
@DiscriminatorValue("COLLECTION")
public class CollectionExecution extends JobExecution { ... }
```

All entities in the hierarchy share one table. The `type` column differentiates rows.

### Testing SINGLE_TABLE Strategies

1. **Test polymorphic queries**: `findAll()` on a base repository returns both base and subclass instances.
2. **Test discriminator values**: Insert a subclass entity, query the raw table, verify the discriminator column.
3. **Test subclass-specific fields**: Subclass-specific columns are null for base class instances.
4. **Test subclass repository filtering**: A repository typed to a subclass only returns matching discriminator rows.
5. **Avoid join fetch across inheritance branches**: Hibernate 6 may throw `ClassCastException` — see `references/hibernate-6-migration.md`.

### Example Test Pattern

```java
import org.springframework.boot.data.jpa.test.autoconfigure.DataJpaTest;
import org.springframework.boot.jpa.test.autoconfigure.TestEntityManager;

@DataJpaTest
class JobExecutionRepositoryTest {

    @Autowired TestEntityManager em;
    @Autowired JobExecutionRepository jobExecutionRepository;

    @Test
    void findAll_shouldReturnSubtypeInstances() {
        CollectionExecution exec = new CollectionExecution();
        exec.setStatus(BatchStatus.STARTING);
        em.persistAndFlush(exec);
        em.clear();

        List<JobExecution> all = jobExecutionRepository.findAll();
        assertThat(all).hasSize(1);
        assertThat(all.get(0)).isInstanceOf(CollectionExecution.class);
    }

    @Test
    void collectionExecutionRepository_onlyReturnsCollectionType() {
        CollectionExecution collExec = new CollectionExecution();
        ReportExecution reportExec = new ReportExecution();
        em.persist(collExec);
        em.persist(reportExec);
        em.flush();
        em.clear();

        List<CollectionExecution> results = collectionExecutionRepository.findAll();
        assertThat(results).hasSize(1);
        assertThat(results.get(0)).isInstanceOf(CollectionExecution.class);
    }
}
```

**Sources**:
- [Baeldung — Query JPA Repository with Single Table Inheritance](https://www.baeldung.com/jpa-inheritance-single-table)
- [Vlad Mihalcea — The best way to map @DiscriminatorColumn](https://vladmihalcea.com/the-best-way-to-map-the-discriminatorcolumn-with-jpa-and-hibernate/)
- [Hibernate Discourse — ClassCastException with join fetch and inheritance](https://discourse.hibernate.org/t/classcastexception-in-hibernate-6-when-join-fetch-is-used-in-a-query-with-entity-inheritance/7815)

---

## Read-Only Repositories

### The Read-Only Repository Pattern

For DDD read models, extend `org.springframework.data.repository.Repository` (not `JpaRepository`) with only query methods. This enforces the aggregate boundary at compile time — callers cannot accidentally call `save()` or `delete()`.

```java
import org.springframework.data.repository.NoRepositoryBean;
import org.springframework.data.repository.Repository;

@NoRepositoryBean
public interface ReadOnlyRepository<T, ID> extends Repository<T, ID> {
    Optional<T> findById(ID id);
    List<T> findAll();
    long count();
}
```

### @QueryHint(HINT_READ_ONLY) ClassCastException — Hibernate 6.6.x

**Symptom**: `@QueryHint(name = HINT_READ_ONLY, value = "true")` on `findById` throws `ClassCastException` in Hibernate 6.6.x.

**Root cause**: Spring Data JPA implements `findById` using `EntityManager.find()`, which passes hints as `Map<String, Object>`. The `@QueryHints` annotation always passes values as `String`, but Hibernate 6.6 expects `Boolean` for the `org.hibernate.readOnly` hint with `em.find()`.

```java
// BROKEN in Hibernate 6.6.x:
@QueryHints(@QueryHint(name = org.hibernate.jpa.QueryHints.HINT_READ_ONLY, value = "true"))
Optional<T> findById(ID id); // ClassCastException: String cannot be cast to Boolean
```

### Preferred Fix: @Transactional(readOnly = true)

Since Spring Framework 5.1, `@Transactional(readOnly = true)` propagates the read-only flag to the Hibernate `Session` via `Session.setDefaultReadOnly(true)`. This:

- Disables dirty-checking for all loaded entities
- Skips hydrated state snapshots (memory savings)
- Works consistently with all query types including `em.find()`
- Avoids the String/Boolean type mismatch entirely

```java
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

@Service
@Transactional(readOnly = true)
public class UserReadService {
    public List<User> findAll() {
        return repository.findAll();
    }
}
```

### Testing Read-Only Behavior

```java
@Test
void findById_readOnlyContext_doesNotDirtyCheck() {
    User saved = em.persistAndFlush(new User("Alice"));
    em.clear();

    // Load via read-only service
    User loaded = userReadService.findById(saved.getId()).orElseThrow();
    loaded.setEmail("modified@example.com"); // Modify in-memory

    em.flush(); // Should NOT generate an UPDATE statement (dirty-check disabled)

    em.clear();
    User reloaded = em.find(User.class, saved.getId());
    // Email should be unchanged because dirty-checking was disabled
    assertThat(reloaded.getEmail()).isEqualTo("Alice");
}
```

**Sources**:
- [Vlad Mihalcea — Spring read-only transaction Hibernate optimization](https://vladmihalcea.com/spring-read-only-transaction-hibernate-optimization/)
- [Thorben Janssen — Hibernate's Read-Only Query Hint](https://thorben-janssen.com/read-only-query-hint/)
- [Spring Data JPA #1503 — HINT_READONLY not applied to findOne()](https://github.com/spring-projects/spring-data-jpa/issues/1503)

---

## Optimistic Locking Tests

`@Version` fields protect against lost updates. Test that concurrent modification is detected:

```java
import jakarta.persistence.OptimisticLockException;

@Test
void optimisticLocking_concurrentModification_throwsOptimisticLockException() {
    Product saved = em.persistAndFlush(new Product("Widget", BigDecimal.TEN));
    UUID id = saved.getId();
    em.clear();

    // Load two separate instances (simulate two concurrent requests)
    Product instance1 = productRepository.findById(id).orElseThrow();
    Product instance2 = productRepository.findById(id).orElseThrow();

    // First save succeeds
    instance1.setPrice(new BigDecimal("15.00"));
    productRepository.saveAndFlush(instance1);
    em.clear();

    // Second save should fail — stale version
    instance2.setPrice(new BigDecimal("20.00"));
    assertThatThrownBy(() -> productRepository.saveAndFlush(instance2))
        .isInstanceOf(OptimisticLockException.class)
        .extracting(e -> ((OptimisticLockException) e).getEntity())
        .isEqualTo(instance2);
}
```

**Note**: This test requires a `@Transactional(propagation = NOT_SUPPORTED)` context or `@SpringBootTest` — optimistic lock conflicts require separate transaction commits, which the `@DataJpaTest` automatic rollback prevents.
