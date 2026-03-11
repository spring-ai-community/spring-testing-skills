---
name: spring-testing-fundamentals
description: "Use this skill when the user asks about testing patterns that apply across all Spring test types, AssertJ assertion idioms, Mockito or BDDMockito setup, argument matchers, ArgumentCaptor, UnnecessaryStubbingException, strict stubbing, @MockitoBean vs @MockitoSpyBean, context caching, the testing pyramid, choosing between @WebMvcTest/@DataJpaTest/@SpringBootTest, Boot 4 annotation migration, or universal test anti-patterns. Also use as the default fallback when no specific slice annotation (@DataJpaTest, @WebMvcTest, @WebFluxTest) is present in the question. Triggers on: org.assertj.core.api.Assertions, assertThat, isEqualTo, isInstanceOf, extracting, assertThatThrownBy, org.mockito.BDDMockito, given/willReturn, then(mock).should, ArgumentCaptor, @Captor, org.springframework.test.context.bean.override.mockito.MockitoBean, @MockitoSpyBean, @DirtiesContext, context cache, testing pyramid, Andy Wilkinson testing."
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
