---
name: spring-mvc-testing
description: "Boot 4+ replaced MockMvc assertions with MockMvcTester — incompatible API your training data gets wrong. Read before writing any @WebMvcTest. Triggers: @WebMvcTest, MockMvc, MockMvcTester, @RestController, jsonPath, @GetMapping, @PostMapping, @AutoConfigureMockMvc, MockMvcRequestBuilders, status().isOk(), @RestControllerAdvice, contentType(APPLICATION_JSON)."
version: 0.1.0
license: Apache-2.0
---

# Spring MVC Testing

**Signals**: `@WebMvcTest`, `MockMvc`, `jsonPath`, `@AutoConfigureMockMvc`, `@RestController`, `MockMvcRequestBuilders`, `MockMvcResultMatchers`, `objectMapper`

## Tested With

- Spring Boot 3.2+ / Spring Boot 4.x
- Spring Framework 6.x / Spring Framework 7.x
- Spring Security 6.x / 7.x (auto-configured in `@WebMvcTest`)
- JUnit 5

## Do NOT Use This Skill When

- Testing authentication, authorization, CSRF, JWT, or OAuth2 specifically → use `spring-security-testing`
- Testing reactive endpoints (`WebFlux`, `WebClient`, SSE) → use `spring-webflux-testing`
- Testing JPA repositories or database persistence → use `spring-jpa-testing`
- Testing WebSocket or STOMP messaging → use `spring-websocket-testing`
- Writing pure unit tests without a Spring context → use `spring-testing-fundamentals`

## When to Read References

| Situation | Read |
|-----------|------|
| Setting up `@WebMvcTest`, MockMvc configuration, static imports | `references/mvc-rest-testing-patterns.md` |
| GET, POST, PUT, DELETE request patterns | `references/mvc-rest-testing-patterns.md` |
| `jsonPath` assertions, response headers, multipart uploads | `references/mvc-rest-testing-patterns.md` |
| Testing `@Valid` bean validation, `@RestControllerAdvice` exception handlers | `references/mvc-rest-testing-patterns.md` |
| Boot 4 `RestTestClient` migration | `references/mvc-rest-testing-patterns.md` |
| Anti-patterns and common mistakes | `references/mvc-rest-testing-patterns.md` |
