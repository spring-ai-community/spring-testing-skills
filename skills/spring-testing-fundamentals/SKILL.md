---
name: spring-testing-fundamentals
description: "Boot 4 renamed @MockBean to @MockitoBean, moved test slice annotations to new packages, and changed context caching behavior. Read for correct AssertJ/BDDMockito idioms and Boot 4 migration. Default fallback when no specific slice annotation is present. Triggers: assertThat, BDDMockito, ArgumentCaptor, @MockitoBean, @MockitoSpyBean, @DirtiesContext, UnnecessaryStubbingException, testing pyramid, context cache."
version: 0.1.0
license: Apache-2.0
---

# Spring Testing Fundamentals

**Signals**: `assertThat`, `assertThatThrownBy`, `given/willReturn`, `then(mock).should`, `ArgumentCaptor`, `@MockitoBean`, `@MockitoSpyBean`, `@DirtiesContext`, context caching, testing pyramid

## Tested With

- Spring Boot 3.2+ / Spring Boot 4.x
- Spring Framework 6.x / Spring Framework 7.x
- AssertJ 3.24+
- Mockito 5.x
- JUnit 5

## Do NOT Use This Skill When

You need domain-specific patterns:
- Testing JPA repositories, Testcontainers, `@DataJpaTest` → use `spring-jpa-testing`
- Testing REST controllers, MockMvc, `@WebMvcTest` → use `spring-mvc-testing`
- Testing security, `@WithMockUser`, JWT, OAuth2 → use `spring-security-testing`
- Testing reactive controllers, `@WebFluxTest`, `StepVerifier` → use `spring-webflux-testing`
- Testing WebSocket/STOMP → use `spring-websocket-testing`

## When to Read References

| Situation | Read |
|-----------|------|
| AssertJ scalars, optionals, BigDecimal, collections, exceptions | `references/assertj-mockito-idioms.md` |
| BDDMockito `given/willReturn`, `then(mock).should`, argument matchers | `references/assertj-mockito-idioms.md` |
| `ArgumentCaptor` usage and verification patterns | `references/assertj-mockito-idioms.md` |
| `UnnecessaryStubbingException` and strict stubbing fixes | `references/assertj-mockito-idioms.md` |
| `@MockitoBean` vs `@MockitoSpyBean` in Boot 4 | `references/assertj-mockito-idioms.md` |
| Testing pyramid: when to use each slice annotation | `references/cross-cutting-testing-patterns.md` |
| Context caching: why tests are slow, cache key, `@DirtiesContext` impact | `references/cross-cutting-testing-patterns.md` |
| Universal anti-patterns across all Spring test types | `references/cross-cutting-testing-patterns.md` |
| Boot 4 annotation migration checklist | `references/cross-cutting-testing-patterns.md` |
