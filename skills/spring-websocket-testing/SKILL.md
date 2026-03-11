---
name: spring-websocket-testing
description: "Use this skill when the user asks to test a WebSocket endpoint, test a STOMP endpoint, test @MessageMapping handlers, test a SockJS connection, test real-time messaging, test @SendToUser delivery, use WebSocketStompClient in tests, or debug WebSocket connection timeouts in tests. Also triggers on: org.springframework.web.socket.messaging.WebSocketStompClient, org.springframework.messaging.simp.stomp.StompSession, org.springframework.messaging.simp.stomp.StompFrameHandler, org.springframework.web.socket.client.standard.StandardWebSocketClient, @SpringBootTest(webEnvironment = RANDOM_PORT) with WebSocket, BlockingQueue for STOMP, CountDownLatch in WebSocket tests, session.subscribe, session.send, /topic/, /queue/, /app/, @SendToUser, @MessageMapping."
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
