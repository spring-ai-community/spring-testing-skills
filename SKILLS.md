# Skills Catalog

Agent-facing catalog of all skills in this repository. Use the routing table below to identify which skill to load.

## Skills

| Skill | Covers | Load When |
|-------|--------|-----------|
| `spring-testing-router` | Meta-skill: routes to the right domain skill | User says "help me test my Spring app" without specifying a domain |
| `spring-jpa-testing` | `@DataJpaTest`, Testcontainers, `@ServiceConnection`, `@Transactional` traps, flush/clear, lazy loading, N+1, SINGLE_TABLE inheritance, Hibernate 6/7 migration | Testing JPA repositories, Spring Data queries, database persistence |
| `spring-mvc-testing` | `@WebMvcTest`, `MockMvc`, `jsonPath`, request validation, `@RestControllerAdvice`, multipart, Boot 4 `RestTestClient` | Testing REST controllers, HTTP endpoints |
| `spring-security-testing` | `@WithMockUser`, `@WithUserDetails`, CSRF, JWT (`jwt()` post-processor), OAuth2, `@PreAuthorize` method security | Testing authentication, authorization, security rules |
| `spring-webflux-testing` | `@WebFluxTest`, `WebTestClient`, `StepVerifier`, virtual time, SSE, reactive security (`mutateWith`) | Testing reactive controllers, Mono/Flux, Server-Sent Events |
| `spring-websocket-testing` | `@SpringBootTest(RANDOM_PORT)`, `WebSocketStompClient`, `StompSession`, `BlockingQueue` pattern, `@SendToUser` | Testing WebSocket/STOMP messaging |
| `spring-testing-fundamentals` | AssertJ idioms, BDDMockito, `ArgumentCaptor`, `@MockitoBean`, context caching, testing pyramid, anti-patterns, Boot 4 migration checklist | General testing patterns, or when no slice annotation is present |

## Routing Notes

- If no slice annotation (`@DataJpaTest`, `@WebMvcTest`, `@WebFluxTest`, `@SpringBootTest` with STOMP) is visible, load `spring-testing-fundamentals`
- Security annotations (`@WithMockUser`, `@WithUserDetails`) → `spring-security-testing` even when combined with `@WebMvcTest`
- STOMP/WebSocket → `spring-websocket-testing` even though it uses `@SpringBootTest`
