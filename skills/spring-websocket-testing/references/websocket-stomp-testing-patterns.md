# WebSocket / STOMP Testing Patterns

Quick-reference for testing Spring WebSocket + STOMP endpoints. WebSocket tests require a **real server** — `MockMvc` and `WebTestClient` cannot complete a WebSocket handshake.

---

## Why RANDOM_PORT is Required

```java
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.boot.test.web.server.LocalServerPort;

// MockMvc operates on a mock DispatcherServlet — no real TCP socket.
// WebTestClient operates on a mock web filter chain — no real TCP socket.
// WebSocket upgrade requires a real HTTP connection that can upgrade to WS.

@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
class OrderWebSocketTest {

    @LocalServerPort
    int port;
}
```

A real server is started on a random port. `@LocalServerPort` injects the port number so tests can build the WebSocket URL.

---

## StompClient Setup

```java
import org.springframework.messaging.converter.MappingJackson2MessageConverter;
import org.springframework.web.socket.client.standard.StandardWebSocketClient;
import org.springframework.web.socket.messaging.WebSocketStompClient;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.boot.test.web.server.LocalServerPort;

@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
class ChatWebSocketTest {

    @LocalServerPort int port;
    WebSocketStompClient stompClient;

    @BeforeEach
    void setUp() {
        stompClient = new WebSocketStompClient(new StandardWebSocketClient());
        stompClient.setMessageConverter(new MappingJackson2MessageConverter());
        stompClient.setDefaultHeartbeat(new long[]{0, 0});  // Disable heartbeat in tests — prevents flakiness
    }

    @AfterEach
    void tearDown() {
        stompClient.stop();  // REQUIRED: prevents executor thread leaks across tests
    }
}
```

**Always call `stompClient.stop()` in `@AfterEach`** — `WebSocketStompClient` spawns executor threads that outlive the test if not stopped, causing flaky failures in subsequent tests.

---

## BlockingQueue Pattern — Send + Receive

The canonical pattern for asserting received STOMP messages. Use `BlockingQueue.poll()` with a timeout instead of `Thread.sleep()`:

```java
import java.util.concurrent.BlockingQueue;
import java.util.concurrent.LinkedBlockingQueue;
import java.util.concurrent.TimeUnit;
import org.springframework.messaging.simp.stomp.StompFrameHandler;
import org.springframework.messaging.simp.stomp.StompHeaders;
import org.springframework.messaging.simp.stomp.StompSession;
import org.springframework.messaging.simp.stomp.StompSessionHandlerAdapter;

@Test
void sendMessage_broadcastsToSubscribers() throws Exception {
    BlockingQueue<ChatMessage> received = new LinkedBlockingQueue<>();

    String url = "ws://localhost:" + port + "/ws";
    StompSession session = stompClient
        .connectAsync(url, new StompSessionHandlerAdapter() {})
        .get(5, TimeUnit.SECONDS);

    session.subscribe("/topic/chat", new StompFrameHandler() {
        @Override
        public java.lang.reflect.Type getPayloadType(StompHeaders headers) {
            return ChatMessage.class;
        }

        @Override
        public void handleFrame(StompHeaders headers, Object payload) {
            received.offer((ChatMessage) payload);
        }
    });

    session.send("/app/chat", new SendMessageRequest("alice", "Hello!"));

    ChatMessage msg = received.poll(5, TimeUnit.SECONDS);  // Wait up to 5s
    assertThat(msg).isNotNull();
    assertThat(msg.getSender()).isEqualTo("alice");
    assertThat(msg.getContent()).isEqualTo("Hello!");

    session.disconnect();
}
```

**Timeout guidance**: Use 5–10 seconds in `poll()` / `await()` calls. WebSocket tests have real startup overhead — 1-second timeouts cause flaky failures in CI.

---

## Unit Testing @MessageMapping Handlers Directly

For testing handler logic in isolation — fast, no infrastructure, no network:

```java
// No Spring context needed — plain unit test
class ChatControllerUnitTest {

    ChatController controller = new ChatController();

    @Test
    void handleChat_returnsChatMessage() {
        SendMessageRequest req = new SendMessageRequest("alice", "Hello!");

        ChatMessage result = controller.handleChat(req);

        assertThat(result.getSender()).isEqualTo("alice");
        assertThat(result.getContent()).isEqualTo("Hello!");
        assertThat(result.getTimestamp()).isNotNull();
    }
}
```

**Test strategy**: unit-test `@MessageMapping` handler logic directly. Use `@SpringBootTest(RANDOM_PORT)` + `StompClient` only for integration tests that verify STOMP routing, broadcast, and subscription delivery.

---

## Testing @SendToUser (User-Targeted Messages)

```java
import org.springframework.messaging.simp.stomp.StompHeaders;
import org.springframework.web.socket.WebSocketHttpHeaders;

@Test
void sendPrivateMessage_deliveredToTargetUser() throws Exception {
    BlockingQueue<PrivateMessage> aliceReceived = new LinkedBlockingQueue<>();

    StompHeaders connectHeaders = new StompHeaders();
    connectHeaders.add("login", "alice");
    connectHeaders.add("passcode", "password");

    StompSession aliceSession = stompClient
        .connectAsync("ws://localhost:" + port + "/ws",
                       new WebSocketHttpHeaders(), connectHeaders,
                       new StompSessionHandlerAdapter() {})
        .get(5, TimeUnit.SECONDS);

    aliceSession.subscribe("/user/queue/messages", new StompFrameHandler() {
        @Override
        public java.lang.reflect.Type getPayloadType(StompHeaders headers) {
            return PrivateMessage.class;
        }

        @Override
        public void handleFrame(StompHeaders headers, Object payload) {
            aliceReceived.offer((PrivateMessage) payload);
        }
    });

    StompSession bobSession = connectAs("bob");
    bobSession.send("/app/message/alice", new SendPrivateMessageRequest("Hi Alice!"));

    PrivateMessage msg = aliceReceived.poll(5, TimeUnit.SECONDS);
    assertThat(msg).isNotNull();
    assertThat(msg.getContent()).isEqualTo("Hi Alice!");

    aliceSession.disconnect();
    bobSession.disconnect();
}
```

---

## CountDownLatch Pattern (Alternative for Single Events)

Use `CountDownLatch` when you expect exactly one event and want to assert the value:

```java
import java.util.concurrent.CountDownLatch;
import java.util.concurrent.atomic.AtomicReference;

@Test
void onConnect_serverSendsWelcomeMessage() throws Exception {
    CountDownLatch latch = new CountDownLatch(1);
    AtomicReference<WelcomeMessage> received = new AtomicReference<>();

    StompSession session = stompClient
        .connectAsync("ws://localhost:" + port + "/ws", new StompSessionHandlerAdapter() {})
        .get(5, TimeUnit.SECONDS);

    session.subscribe("/user/queue/welcome", new StompFrameHandler() {
        @Override
        public java.lang.reflect.Type getPayloadType(StompHeaders headers) {
            return WelcomeMessage.class;
        }

        @Override
        public void handleFrame(StompHeaders headers, Object payload) {
            received.set((WelcomeMessage) payload);
            latch.countDown();
        }
    });

    session.send("/app/connect", new ConnectRequest());

    assertThat(latch.await(5, TimeUnit.SECONDS)).isTrue();
    assertThat(received.get().getText()).contains("Welcome");

    session.disconnect();
}
```

Use `BlockingQueue` for multiple messages; use `CountDownLatch + AtomicReference` for a single expected event.

---

## Testing Error Frames

```java
import org.springframework.messaging.simp.stomp.StompCommand;

@Test
void sendInvalidMessage_serverSendsErrorFrame() throws Exception {
    CountDownLatch errorLatch = new CountDownLatch(1);
    AtomicReference<String> errorMessage = new AtomicReference<>();

    StompSession session = stompClient
        .connectAsync("ws://localhost:" + port + "/ws", new StompSessionHandlerAdapter() {
            @Override
            public void handleException(StompSession s, StompCommand cmd,
                                         StompHeaders headers, byte[] payload, Throwable ex) {
                errorMessage.set(ex.getMessage());
                errorLatch.countDown();
            }
        })
        .get(5, TimeUnit.SECONDS);

    session.send("/app/chat", new InvalidRequest(null, null));

    assertThat(errorLatch.await(5, TimeUnit.SECONDS)).isTrue();
    // Optionally assert errorMessage.get() contains expected error text
}
```

---

## WebSocket with Spring Security

For full integration tests with authentication:

```java
import org.springframework.http.HttpHeaders;
import org.springframework.web.socket.WebSocketHttpHeaders;

// Step 1: Obtain a session cookie via HTTP login
// Step 2: Pass the cookie in WebSocketHttpHeaders

WebSocketHttpHeaders wsHeaders = new WebSocketHttpHeaders();
wsHeaders.add(HttpHeaders.COOKIE, "SESSION=" + sessionId);

StompSession session = stompClient
    .connectAsync("ws://localhost:" + port + "/ws",
                   wsHeaders,
                   new StompHeaders(),
                   new StompSessionHandlerAdapter() {})
    .get(5, TimeUnit.SECONDS);
```

Obtain the `sessionId` by performing an HTTP POST to `/login` with `RestTemplate` or `WebTestClient` before the WebSocket test.

---

## What to Test Where

| Test Type | What It Covers | Infrastructure |
|---|---|---|
| Unit test on `@MessageMapping` method | Handler logic, return value, exceptions | None — plain JUnit |
| `@SpringBootTest(RANDOM_PORT)` + StompClient | Full WS handshake, STOMP routing, broadcast, `@SendToUser` | Real server on random port |
| Hard to test reliably | SockJS fallback reliability, broker relay (ActiveMQ/RabbitMQ) in CI | Requires external broker |

---

## Boot 3.x → 4.x WebSocket Testing Changes

| Area | Boot 3.x | Boot 4.x |
|---|---|---|
| WebSocket namespace | `jakarta.websocket.*` | Same |
| `StandardWebSocketClient` | `org.springframework.web.socket.client.standard` | Same |
| `WebSocketStompClient` | `org.springframework.web.socket.messaging` | Same |
| Security WebSocket config | `AbstractSecurityWebSocketMessageBrokerConfigurer` | Deprecated; use component-based `SecurityFilterChain` in Security 7 |

---

## Anti-Patterns

| Anti-Pattern | Fix |
|---|---|
| Using `MockMvc` for WebSocket tests | Use `RANDOM_PORT` + `StompClient` — `MockMvc` cannot upgrade to WebSocket |
| Timeout too short (1s) in `poll()` / `await()` | Use 5–10s — startup overhead is real, especially in CI |
| Not disconnecting sessions in `@AfterEach` | Always call `session.disconnect()` for each connected session |
| Not stopping `stompClient` in `@AfterEach` | Leaks executor threads, causes flaky failures in subsequent tests |
| Assuming message order in multi-subscriber tests | Message ordering is not guaranteed; use sets or sort before asserting |
| `Thread.sleep()` for synchronization | Use `BlockingQueue.poll(timeout)` or `CountDownLatch.await(timeout)` |
