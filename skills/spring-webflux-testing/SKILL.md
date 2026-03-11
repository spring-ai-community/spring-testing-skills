---
name: spring-webflux-testing
description: "Use this skill when the user asks to test a reactive controller, test a WebFlux endpoint, set up @WebFluxTest, use WebTestClient, use StepVerifier, test a Flux or Mono, test SSE (Server-Sent Events), test reactive security, test R2DBC reactive repositories, use virtual time in tests, or test WebClient responses. Also triggers on: org.springframework.boot.webflux.test.autoconfigure.WebFluxTest, org.springframework.test.web.reactive.server.WebTestClient, reactor.test.StepVerifier, Mono, Flux, reactor.core.publisher, @AutoConfigureWebTestClient, mockUser() from SecurityMockServerConfigurers, mockJwt(), mutateWith, StepVerifier.withVirtualTime, Flux.interval, thenCancel, verifyComplete, expectBody, expectBodyList, expectStatus."
version: 0.1.0
license: Apache-2.0
---

# Spring WebFlux Testing

**Signals**: `@WebFluxTest`, `WebTestClient`, `StepVerifier`, `Mono`, `Flux`, `mutateWith`, `mockUser()`, `mockJwt()`, `thenCancel`, `verifyComplete`, `withVirtualTime`

## Tested With

- Spring Boot 3.2+ / Spring Boot 4.x
- Spring Framework 6.x / Spring Framework 7.x
- Reactor Core 3.5+ / 3.7+
- Spring Security 6.x / 7.x (auto-configured in `@WebFluxTest`)
- JUnit 5

## Do NOT Use This Skill When

- Testing Spring MVC (servlet-based) controllers → use `spring-mvc-testing`
- Testing Spring Security rules in a servlet-based application → use `spring-security-testing`
- Testing JPA repositories → use `spring-jpa-testing`
- Testing WebSocket or STOMP messaging → use `spring-websocket-testing`
- Writing pure unit tests for reactive service logic → use `spring-testing-fundamentals` for AssertJ/Mockito patterns, combine with StepVerifier for reactive

## When to Read References

| Situation | Read |
|-----------|------|
| Setting up `@WebFluxTest`, `WebTestClient` configuration | `references/webflux-testing-patterns.md` |
| GET, POST, PUT, DELETE with `WebTestClient` | `references/webflux-testing-patterns.md` |
| `expectBody`, `expectBodyList`, `jsonPath` assertions | `references/webflux-testing-patterns.md` |
| `StepVerifier` for Flux, Mono, error signals | `references/webflux-testing-patterns.md` |
| `StepVerifier.withVirtualTime` for time-based operators | `references/webflux-testing-patterns.md` |
| Testing reactive security with `mutateWith(mockUser())`, `mockJwt()` | `references/webflux-testing-patterns.md` |
| Testing SSE streams | `references/webflux-testing-patterns.md` |
| Boot 3.x → 4.x WebFlux testing changes | `references/webflux-testing-patterns.md` |
