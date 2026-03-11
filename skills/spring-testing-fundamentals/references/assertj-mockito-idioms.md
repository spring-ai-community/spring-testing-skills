# AssertJ + Mockito / BDDMockito Idioms

Quick-reference for assertion and mocking patterns used across all Spring Boot test types. Prefer `org.mockito.BDDMockito` (`given/willReturn`) over classic Mockito (`when/thenReturn`) for readability and BDD alignment.

---

## Static Import Conventions

```java
// AssertJ
import static org.assertj.core.api.Assertions.*;

// BDDMockito (preferred over classic Mockito)
import static org.mockito.BDDMockito.*;

// Hamcrest (for MockMvc jsonPath — not AssertJ)
import static org.hamcrest.Matchers.*;
```

---

## AssertJ — Scalar Assertions

```java
import static org.assertj.core.api.Assertions.*;

assertThat(user.getEmail()).isEqualTo("alice@example.com");
assertThat(user.getAge()).isGreaterThan(18).isLessThan(100);
assertThat(user.getName()).startsWith("Al").endsWith("ce").hasSize(5);
assertThat(user.isActive()).isTrue();
assertThat(user.getDeletedAt()).isNull();
assertThat(optional).isPresent().hasValue("expected");
assertThat(optional).isEmpty();

// BigDecimal — ALWAYS use isEqualByComparingTo (not isEqualTo — scale-aware comparison)
assertThat(price).isEqualByComparingTo(BigDecimal.valueOf(9.99));
assertThat(price).isEqualByComparingTo("9.99");
```

---

## AssertJ — Collection Assertions

```java
// Ordering matters
assertThat(list).containsExactly("alpha", "beta", "gamma");

// Ordering doesn't matter
assertThat(list).containsExactlyInAnyOrder("gamma", "alpha", "beta");

// Subset check
assertThat(list).contains("alpha", "gamma");
assertThat(list).hasSize(3).isNotEmpty();

// Filtering
assertThat(orders)
    .filteredOn(o -> o.getStatus() == OrderStatus.PENDING)
    .hasSize(2);

// Extracting single field
assertThat(users)
    .extracting(User::getEmail)
    .containsExactlyInAnyOrder("alice@example.com", "bob@example.com");

// Multi-field extraction (tuple)
import static org.assertj.core.groups.Tuple.tuple;

assertThat(orders)
    .extracting(Order::getStatus, Order::getTotal)
    .containsExactlyInAnyOrder(
        tuple(OrderStatus.PENDING, new BigDecimal("50.00")),
        tuple(OrderStatus.SHIPPED, new BigDecimal("100.00"))
    );

// allSatisfy / anySatisfy
assertThat(users).allSatisfy(user -> {
    assertThat(user.getEmail()).contains("@");
    assertThat(user.isActive()).isTrue();
});
assertThat(users).anySatisfy(user ->
    assertThat(user.getRoles()).contains("ADMIN"));
```

---

## AssertJ — Exception Assertions

```java
// Preferred pattern
assertThatThrownBy(() -> userService.findById(99L))
    .isInstanceOf(UserNotFoundException.class)
    .hasMessage("User not found: 99")
    .hasMessageContaining("99");

// assertThatExceptionOfType (fluent alternative)
assertThatExceptionOfType(UserNotFoundException.class)
    .isThrownBy(() -> userService.findById(99L))
    .withMessage("User not found: 99");

// catchThrowable — when you need to inspect the exception further
Throwable thrown = catchThrowable(() -> orderService.cancel(1L));
assertThat(thrown)
    .isInstanceOf(IllegalStateException.class)
    .hasMessageContaining("already cancelled");

// assertThatNoException — verify no exception is thrown
assertThatNoException().isThrownBy(() -> validator.validate(validRequest));
```

---

## BDDMockito — Stubbing

```java
import static org.mockito.BDDMockito.*;

// given / willReturn
given(userRepository.findById(1L))
    .willReturn(Optional.of(new User(1L, "alice@example.com")));

// willThrow
given(userRepository.findById(99L))
    .willThrow(new UserNotFoundException(99L));

// willAnswer — dynamic return value
given(idGenerator.nextId())
    .willAnswer(inv -> UUID.randomUUID().toString());

// Void methods
willDoNothing().given(emailService).sendWelcome(any(User.class));
willThrow(new EmailException("SMTP down")).given(emailService).sendWelcome(any(User.class));
```

---

## Argument Matchers

```java
// Exact value (no matcher needed)
given(repo.findById(1L)).willReturn(Optional.of(user));

// any() / any(Class) — type-safe
given(repo.save(any(User.class))).willReturn(savedUser);

// RULE: If ANY argument uses a matcher, ALL arguments must use matchers
// WRONG (mixes matcher with literal):
given(repo.findByEmailAndActive(eq("alice@example.com"), true)).willReturn(Optional.of(user));

// CORRECT:
given(repo.findByEmailAndActive(eq("alice@example.com"), eq(true))).willReturn(Optional.of(user));

// argThat — inline predicate
given(repo.save(argThat(u -> "alice@example.com".equals(u.getEmail()))))
    .willReturn(savedUser);

// Common matchers:
// any()               — any non-null value
// any(Class)          — any instance of class
// anyString()         — any String (including null via Mockito 4+)
// anyLong() / anyInt() — any primitive wrapper
// eq(value)           — exact equality
// isNull() / notNull() — null checks
// argThat(predicate)  — custom predicate
```

---

## BDDMockito — Verification

```java
// then(mock).should() replaces Mockito.verify(mock)
then(emailService).should().sendWelcome(any(User.class));       // called once
then(retryService).should(times(3)).attempt(any());             // called N times
then(emailService).should(never()).sendAlert(any());            // never called
then(cache).should(atLeastOnce()).evict(anyString());           // at least once
then(emailService).shouldHaveNoMoreInteractions();              // use sparingly — brittle
```

---

## ArgumentCaptor

Capture the argument passed to a mock method and assert on it:

```java
import org.mockito.ArgumentCaptor;
import org.mockito.Captor;

@Captor
ArgumentCaptor<EmailNotification> captor;

@Test
void registerUser_sendsWelcomeEmail() {
    userService.register(new RegisterRequest("alice@example.com", "Alice"));

    then(emailService).should().send(captor.capture());

    EmailNotification notification = captor.getValue();
    assertThat(notification.getTo()).isEqualTo("alice@example.com");
    assertThat(notification.getSubject()).contains("Welcome");
}

// Multiple invocations
then(auditLog).should(times(2)).record(captor.capture());
List<AuditEvent> events = captor.getAllValues();
assertThat(events).extracting(AuditEvent::getAction)
    .containsExactly("USER_CREATED", "EMAIL_SENT");
```

`@Captor` requires `@ExtendWith(MockitoExtension.class)` or Spring's test support to inject. Alternatively: `ArgumentCaptor<EmailNotification> captor = ArgumentCaptor.forClass(EmailNotification.class);`

---

## @MockitoBean vs @MockitoSpyBean (Boot 4)

```java
import org.springframework.test.context.bean.override.mockito.MockitoBean;
import org.springframework.test.context.bean.override.mockito.MockitoSpyBean;

// @MockitoBean — full mock, all methods return defaults unless stubbed
// Replaces @MockBean from org.springframework.boot.test.mock.mockito (removed in Boot 4)
@MockitoBean
OrderService orderService;

// @MockitoSpyBean — wraps the REAL bean, calls real implementation unless stubbed
// Replaces @SpyBean (removed in Boot 4)
@MockitoSpyBean
AuditService auditService;

// With spy, only override what you need:
willDoNothing().given(auditService).record(any(AuditEvent.class));
// All other AuditService methods run the real code
```

**Warning on `@MockitoSpyBean`**: The real bean must be loadable in the test slice context. In `@WebMvcTest`, only web-layer beans load — `@MockitoSpyBean` on a service will fail if all of that service's dependencies aren't satisfied.

---

## Strict Stubbing and UnnecessaryStubbingException

Mockito 2+ uses strict stubbing by default (via `MockitoExtension`): if you define a stub that is never called in the test, Mockito throws `UnnecessaryStubbingException`.

```java
// CAUSE: stub defined in @BeforeEach but only one test uses it
@BeforeEach
void setUp() {
    given(repo.findById(1L)).willReturn(Optional.of(user)); // UnnecessaryStubbingException
}

@Test
void testA() {
    repo.findById(1L); // only testA uses this stub
}

@Test
void testB() {
    // testB doesn't call findById — UnnecessaryStubbingException for testB!
}
```

**Fixes**:

```java
// Fix A (preferred): Move stubs into each test that needs them
@Test
void testA() {
    given(repo.findById(1L)).willReturn(Optional.of(user));
    repo.findById(1L);
}

// Fix B: Use lenient() for shared setup stubs
@BeforeEach
void setUp() {
    lenient().when(repo.findById(anyLong())).thenReturn(Optional.of(user));
}

// Fix C (avoid): Relax at class level
@MockitoSettings(strictness = Strictness.LENIENT)
class MyServiceTest { }
```

---

## MockMvc + Hamcrest vs AssertJ

`MockMvc.andExpect()` uses **Hamcrest** matchers (not AssertJ) in `jsonPath`:

```java
import static org.hamcrest.Matchers.*;

.andExpect(jsonPath("$.items", hasSize(3)))
.andExpect(jsonPath("$.name", containsString("Widget")))
.andExpect(jsonPath("$.price", greaterThan(0.0)))
.andExpect(jsonPath("$.tags", hasItem("featured")))
.andExpect(jsonPath("$.ids", containsInAnyOrder(1, 2, 3)))
```

For complex body assertions, extract and use AssertJ:

```java
String json = mockMvc.perform(get("/orders/1"))
    .andExpect(status().isOk())
    .andReturn().getResponse().getContentAsString();
OrderDto order = objectMapper.readValue(json, OrderDto.class);
assertThat(order.getItems()).hasSize(3);
```

---

## Common Mistakes

```java
// MISTAKE 1: Stubbing after the act — always stub BEFORE the method call
orderService.processOrder(request);       // act (wrong order)
given(repo.findById(1L)).willReturn(...); // WRONG — stub too late

// MISTAKE 2: Mixing matchers with literals
given(repo.find(eq("alice"), true)).willReturn(...);      // ERROR
given(repo.find(eq("alice"), eq(true))).willReturn(...);  // CORRECT

// MISTAKE 3: Over-verifying implementation details
verify(repo).findById(1L);
verify(mapper).toDto(any()); // noise — tests implementation, not behavior

// CORRECT: verify observable side effects only
then(emailService).should().sendConfirmation(any(Order.class));

// MISTAKE 4: Mocking DTOs or domain value objects
// Don't mock simple data carriers — use real instances instead

// MISTAKE 5: Defined stubs in @BeforeEach but not all tests use them
// Move stubs into the tests that actually need them
```

---

## Boot 3.x → 4.x Mockito/AssertJ Changes

| Area | Boot 3.x | Boot 4.x |
|---|---|---|
| `@MockBean` | `org.springframework.boot.test.mock.mockito.MockBean` | Removed — use `@MockitoBean` from `org.springframework.test.context.bean.override.mockito` |
| `@SpyBean` | `org.springframework.boot.test.mock.mockito.SpyBean` | Removed — use `@MockitoSpyBean` |
| `MockMvc` (primary) | `MockMvc` | `MockMvc` or `RestTestClient` via `@AutoConfigureRestTestClient` |
| AssertJ | 3.24.x | 3.26.x+ |
| Strict stubbing | Default via `MockitoExtension` | Same |

---

## Quick Reference

| Need | Use |
|---|---|
| Stub return value | `given(mock.method(args)).willReturn(value)` |
| Stub void to do nothing | `willDoNothing().given(mock).method(args)` |
| Stub to throw | `given(mock.method(args)).willThrow(ex)` |
| Verify call happened | `then(mock).should().method(args)` |
| Verify never called | `then(mock).should(never()).method(any())` |
| Capture argument | `then(mock).should().method(captor.capture()); captor.getValue()` |
| Assert exception | `assertThatThrownBy(() -> ...).isInstanceOf(X.class)` |
| Assert collection fields | `assertThat(list).extracting(X::getField).contains(...)` |
| Assert BigDecimal | `assertThat(val).isEqualByComparingTo(BigDecimal.valueOf(x))` |
| Hamcrest in MockMvc jsonPath | `jsonPath("$.field", is("value"))` |
