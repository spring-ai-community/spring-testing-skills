---
name: spring-websocket-testing
description: "WebSocket tests require @SpringBootTest(RANDOM_PORT) and async message handling — @WebMvcTest won't work. Timeout and session leak traps are common. Read before testing STOMP or WebSocket. Triggers: WebSocketStompClient, StompSession, @MessageMapping, @SendToUser, BlockingQueue, /topic/, /queue/, CountDownLatch, session.subscribe, session.send."
version: 0.1.0
license: Apache-2.0
---

# Spring WebSocket Testing

**Signals**: `WebSocketStompClient`, `StompSession`, `StompFrameHandler`, `BlockingQueue`, `RANDOM_PORT`, `session.subscribe`, `session.send`, `/topic/`, `/queue/`, `/app/`, `@MessageMapping`, `@SendToUser`

## Tested With

- Spring Boot 3.2+ / Spring Boot 4.x
- Spring Framework 6.x / Spring Framework 7.x
- Spring WebSocket (bundled with Spring Framework)
- JUnit 5

## Do NOT Use This Skill When

- Testing reactive WebFlux endpoints without STOMP → use `spring-webflux-testing`
- Testing REST controllers → use `spring-mvc-testing`
- Testing authentication/authorization flows → use `spring-security-testing`
- Writing pure unit tests for `@MessageMapping` handler logic → use `spring-testing-fundamentals` (plain unit test, no infrastructure)

## When to Read References

| Situation | Read |
|-----------|------|
| Why `@SpringBootTest(webEnvironment = RANDOM_PORT)` is required for WebSocket | `references/websocket-stomp-testing-patterns.md` |
| `WebSocketStompClient` setup, `StompSession`, connect/subscribe/send pattern | `references/websocket-stomp-testing-patterns.md` |
| `BlockingQueue` pattern for receiving STOMP messages | `references/websocket-stomp-testing-patterns.md` |
| Unit testing `@MessageMapping` handlers directly (fast path) | `references/websocket-stomp-testing-patterns.md` |
| `@SendToUser` (user-targeted message) testing | `references/websocket-stomp-testing-patterns.md` |
| `CountDownLatch` pattern for single-event assertions | `references/websocket-stomp-testing-patterns.md` |
| Testing error frames and STOMP error handling | `references/websocket-stomp-testing-patterns.md` |
| WebSocket with Spring Security (session cookie auth) | `references/websocket-stomp-testing-patterns.md` |
| Anti-patterns: `Thread.sleep()`, timeout too short, session leak | `references/websocket-stomp-testing-patterns.md` |
