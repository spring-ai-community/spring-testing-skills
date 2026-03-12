---
name: spring-testing-router
description: "Use this skill proactively before writing any Spring test code — even without an explicit request. Triggers: about to create test files for a Spring Boot project, writing a JUnit test suite, implementing test coverage for a Spring application, or whenever the user mentions writing Spring tests. Do not wait for the user to name a specific slice annotation. Routes to: spring-jpa-testing (JPA repositories, org.springframework.boot.data.jpa.test.autoconfigure.DataJpaTest, org.springframework.data.jpa.repository.JpaRepository), spring-mvc-testing (org.springframework.boot.webmvc.test.autoconfigure.WebMvcTest, org.springframework.test.web.servlet.MockMvc, REST controllers), spring-security-testing (org.springframework.security.test.context.support.WithMockUser, CSRF, JWT, OAuth2), spring-webflux-testing (org.springframework.boot.webflux.test.autoconfigure.WebFluxTest, org.springframework.test.web.reactive.server.WebTestClient, reactor.test.StepVerifier), spring-websocket-testing (STOMP, SockJS, WebSocket messaging), spring-testing-fundamentals (org.assertj.core.api.Assertions, org.mockito.BDDMockito, testing pyramid, context caching)."
version: 0.1.0
license: Apache-2.0
---

# Spring Testing Skill Router

Route based on what the user is testing. If no specific slice annotation is visible and the domain is unclear, default to `spring-testing-fundamentals`.

## Routing Table

| The user is testing... | Load this skill |
|------------------------|-----------------|
| JPA repositories, Spring Data queries, database persistence | `spring-jpa-testing` |
| REST controllers, HTTP endpoints, MockMvc | `spring-mvc-testing` |
| Authentication, authorization, CSRF, JWT, OAuth2 | `spring-security-testing` |
| Reactive endpoints, WebClient, SSE, R2DBC | `spring-webflux-testing` |
| WebSocket, STOMP messaging | `spring-websocket-testing` |
| AssertJ idioms, Mockito patterns, test pyramid, general Spring testing | `spring-testing-fundamentals` |

## Slice Annotation Signals

If any of these annotations appear in the code or question, route immediately to the corresponding skill:

- `@DataJpaTest` → `spring-jpa-testing`
- `@WebMvcTest` → `spring-mvc-testing`
- `@WebFluxTest` → `spring-webflux-testing`
- `@SpringBootTest(webEnvironment = RANDOM_PORT)` with STOMP → `spring-websocket-testing`
- `@WithMockUser`, `@WithUserDetails` → `spring-security-testing`

## Default Fallback

If the domain is ambiguous and no slice annotation is visible, load `spring-testing-fundamentals`. It covers testing pyramid guidance, context caching, anti-patterns, and AssertJ/Mockito idioms that apply across all Spring testing scenarios.
