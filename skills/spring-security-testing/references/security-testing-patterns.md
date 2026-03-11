# Spring Security Testing Patterns

Quick-reference for testing secured endpoints in Spring Boot applications. `org.springframework.boot.webmvc.test.autoconfigure.WebMvcTest` (`@WebMvcTest`) auto-configures the Spring Security filter chain — you do not need `@SpringBootTest` to test security rules in MVC tests.

---

## @WithMockUser

`org.springframework.security.test.context.support.WithMockUser` (`@WithMockUser`) creates a mock `Authentication` in the `SecurityContext` for the duration of the test.

```java
import org.springframework.boot.webmvc.test.autoconfigure.WebMvcTest;
import org.springframework.security.test.context.support.WithMockUser;
import org.springframework.test.context.bean.override.mockito.MockitoBean;
import org.springframework.test.web.servlet.MockMvc;

import static org.mockito.BDDMockito.*;
import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.*;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.*;

@WebMvcTest(OrderController.class)
class OrderControllerSecurityTest {

    @Autowired MockMvc mockMvc;
    @MockitoBean OrderService orderService;

    @Test
    @WithMockUser(username = "alice", roles = "USER")
    void getOrder_asUser_returns200() throws Exception {
        given(orderService.findById(1L)).willReturn(anOrder());
        mockMvc.perform(get("/orders/1"))
            .andExpect(status().isOk());
    }

    @Test
    void getOrder_noAuth_returns401() throws Exception {
        mockMvc.perform(get("/orders/1"))
            .andExpect(status().isUnauthorized());
    }

    @Test
    @WithMockUser(roles = "USER")   // not ADMIN
    void deleteOrder_asUser_returns403() throws Exception {
        mockMvc.perform(delete("/orders/1"))
            .andExpect(status().isForbidden());
    }
}
```

**Testing all three paths for every secured endpoint**: authenticated+authorized (200), authenticated+unauthorized (403), and unauthenticated (401).

---

## roles vs authorities — The ROLE_ Prefix

```java
// roles = "ADMIN"       →  authority = "ROLE_ADMIN"   (prefix added automatically)
// authorities = "ADMIN" →  authority = "ADMIN"         (no prefix added)

// Use `roles` when security config uses hasRole("ADMIN")
// Use `authorities` when security config uses hasAuthority("products:read")

@WithMockUser(authorities = {"products:read", "products:write"})
```

**Common mistake**: Writing `@WithMockUser(roles = "ROLE_ADMIN")` — the `ROLE_` prefix is added by `@WithMockUser`, resulting in `ROLE_ROLE_ADMIN`, which never matches `hasRole("ADMIN")`.

---

## @WithUserDetails

`org.springframework.security.test.context.support.WithUserDetails` (`@WithUserDetails`) loads the principal from a real `UserDetailsService` bean — useful when your `SecurityContext` must contain a real user object.

```java
import org.springframework.security.test.context.support.WithUserDetails;
import org.springframework.boot.test.context.TestConfiguration;
import org.springframework.security.core.userdetails.User;
import org.springframework.security.core.userdetails.UserDetailsService;
import org.springframework.security.provisioning.InMemoryUserDetailsManager;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Import;

@WebMvcTest(ProfileController.class)
@Import(ProfileControllerTest.TestSecurityConfig.class)
class ProfileControllerTest {

    @Test
    @WithUserDetails(value = "alice@example.com",
                     userDetailsServiceBeanName = "userDetailsService")
    void getProfile_returnsAuthenticatedUsersProfile() throws Exception {
        mockMvc.perform(get("/profile"))
            .andExpect(status().isOk())
            .andExpect(jsonPath("$.email").value("alice@example.com"));
    }

    @TestConfiguration
    static class TestSecurityConfig {
        @Bean
        UserDetailsService userDetailsService() {
            var alice = User.withUsername("alice@example.com")
                .password("{noop}password")
                .roles("USER")
                .build();
            return new InMemoryUserDetailsManager(alice);
        }
    }
}
```

Use `@WithUserDetails` when controllers access `Authentication.getPrincipal()` and need the real principal type, not a synthetic mock.

---

## SecurityMockMvcRequestPostProcessors — Per-Request Auth

`org.springframework.security.test.web.servlet.request.SecurityMockMvcRequestPostProcessors` provides request-level authentication without annotations:

```java
import static org.springframework.security.test.web.servlet.request.SecurityMockMvcRequestPostProcessors.*;

// Synthetic user
mockMvc.perform(get("/orders").with(user("alice").roles("USER")))
    .andExpect(status().isOk());

// HTTP Basic
mockMvc.perform(get("/admin").with(httpBasic("admin", "secret")))
    .andExpect(status().isOk());

// Anonymous (explicitly unauthenticated)
mockMvc.perform(get("/public").with(anonymous()))
    .andExpect(status().isOk());
```

Use post-processors when you need different authentication per test method within the same test class, or when `@WithMockUser` doesn't provide enough control.

---

## CSRF — Required for Mutating Requests

Spring Security enables CSRF protection by default for session-based applications. Without `csrf()` on POST/PUT/DELETE, MockMvc returns `403 Forbidden`.

```java
import static org.springframework.security.test.web.servlet.request.SecurityMockMvcRequestPostProcessors.*;

mockMvc.perform(post("/orders")
        .with(csrf())
        .with(user("alice").roles("USER"))
        .contentType(MediaType.APPLICATION_JSON)
        .content(orderJson))
    .andExpect(status().isCreated());
```

### Disabling CSRF for Stateless/JWT Apps

For REST APIs using JWT (no session), disable CSRF in a `@TestConfiguration`:

```java
import org.springframework.boot.test.context.TestConfiguration;
import org.springframework.context.annotation.Bean;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.web.SecurityFilterChain;
import org.springframework.context.annotation.Import;

@WebMvcTest(OrderController.class)
@Import(OrderControllerTest.DisableCsrfConfig.class)
class OrderControllerTest {

    @TestConfiguration
    static class DisableCsrfConfig {
        @Bean
        SecurityFilterChain testChain(HttpSecurity http) throws Exception {
            http.csrf(csrf -> csrf.disable())
                .authorizeHttpRequests(auth -> auth.anyRequest().authenticated());
            return http.build();
        }
    }
}
```

---

## JWT Resource Server Testing with jwt()

The `jwt()` post-processor from Spring Security Test is the canonical way to test JWT-secured endpoints without running a real authorization server.

```java
import static org.springframework.security.test.web.servlet.request.SecurityMockMvcRequestPostProcessors.*;
import org.springframework.context.annotation.Import;

@WebMvcTest(OAuth2ResourceServerController.class)
@Import(OAuth2ResourceServerSecurityConfiguration.class)
class OAuth2ResourceServerControllerTests {

    @Autowired MockMvc mockMvc;

    @Test
    void indexGreetsAuthenticatedUser() throws Exception {
        mockMvc.perform(get("/").with(jwt().jwt(j -> j.subject("ch4mpy"))))
            .andExpect(content().string(is("Hello, ch4mpy!")));
    }

    @Test
    void messageRequiresScopeReadAuthority() throws Exception {
        // Via scope claim — Jwt extracts to SCOPE_* authorities automatically
        mockMvc.perform(get("/message")
                .with(jwt().jwt(j -> j.claim("scope", "message:read"))))
            .andExpect(content().string(is("secret message")));

        // Or via explicit authorities
        mockMvc.perform(get("/message")
                .with(jwt().authorities(new SimpleGrantedAuthority("SCOPE_message:read"))))
            .andExpect(content().string(is("secret message")));
    }

    @Test
    void messageWithoutScope_returns403() throws Exception {
        mockMvc.perform(get("/message").with(jwt()))
            .andExpect(status().isForbidden());
    }
}
```

`jwt()` builds a `JwtAuthenticationToken` without contacting any authorization server. The lambda `j -> j.subject("ch4mpy")` configures JWT claims.

---

## OAuth2 Login Testing

```java
import static org.springframework.security.test.web.servlet.request.SecurityMockMvcRequestPostProcessors.*;

// oauth2Login() — OAuth2 resource owner password
mockMvc.perform(get("/profile")
        .with(oauth2Login()
            .attributes(attrs -> attrs.put("email", "alice@example.com"))))
    .andExpect(status().isOk());

// mockOidcLogin() — OIDC with ID token
mockMvc.perform(get("/profile")
        .with(mockOidcLogin()
            .idToken(t -> t.subject("user-123").claim("email", "alice@example.com"))))
    .andExpect(status().isOk());
```

---

## Custom @WithSecurityContext for Complex JWT Scenarios

When the same JWT structure is used repeatedly, create a custom annotation backed by `org.springframework.security.test.context.support.WithSecurityContext`:

```java
import org.springframework.security.test.context.support.WithSecurityContext;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;

// Step 1: Define the annotation
@Retention(RetentionPolicy.RUNTIME)
@WithSecurityContext(factory = JwtFactory.class)
public @interface WithMockJwt {
    String subject() default "test-user";
    String[] authorities() default {};
    String[] scopes() default {};
}

// Step 2: Implement the factory
import org.springframework.security.core.authority.SimpleGrantedAuthority;
import org.springframework.security.core.context.SecurityContext;
import org.springframework.security.core.context.SecurityContextImpl;
import org.springframework.security.oauth2.jwt.Jwt;
import org.springframework.security.oauth2.server.resource.authentication.JwtAuthenticationToken;
import org.springframework.security.test.context.support.WithSecurityContextFactory;
import java.util.stream.Stream;
import java.util.Collection;
import java.util.stream.Collectors;

class JwtFactory implements WithSecurityContextFactory<WithMockJwt> {
    @Override
    public SecurityContext createSecurityContext(WithMockJwt annotation) {
        Jwt jwt = Jwt.withTokenValue("test-token")
            .header("alg", "none")
            .subject(annotation.subject())
            .claim("scope", String.join(" ", annotation.scopes()))
            .build();

        Collection<SimpleGrantedAuthority> authorities = Stream
            .concat(
                Stream.of(annotation.authorities()).map(SimpleGrantedAuthority::new),
                Stream.of(annotation.scopes()).map(s -> new SimpleGrantedAuthority("SCOPE_" + s))
            )
            .collect(Collectors.toList());

        return new SecurityContextImpl(new JwtAuthenticationToken(jwt, authorities));
    }
}

// Step 3: Use in tests
@Test
@WithMockJwt(subject = "alice", scopes = "message:read")
void messageCanBeRead() throws Exception {
    mockMvc.perform(get("/message")).andExpect(status().isOk());
}
```

---

## Testing @PreAuthorize Method Security

Method security is NOT active in `@WebMvcTest` by default — it applies at the service layer, not the HTTP layer. Use `@SpringBootTest` with a minimal context:

```java
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.security.access.AccessDeniedException;
import org.springframework.security.test.context.support.WithMockUser;
import org.springframework.test.context.bean.override.mockito.MockitoBean;

@SpringBootTest(classes = {OrderService.class, MethodSecurityConfig.class})
class OrderServiceMethodSecurityTest {

    @Autowired OrderService orderService;
    @MockitoBean OrderRepository orderRepository;

    @Test
    @WithMockUser(roles = "USER")
    void cancelOrder_asUser_throwsAccessDenied() {
        assertThatThrownBy(() -> orderService.cancelOrder(1L))
            .isInstanceOf(AccessDeniedException.class);
    }

    @Test
    @WithMockUser(roles = "ADMIN")
    void cancelOrder_asAdmin_succeeds() {
        given(orderRepository.findById(1L)).willReturn(Optional.of(anOrder()));
        assertThatNoException().isThrownBy(() -> orderService.cancelOrder(1L));
    }
}
```

---

## SecurityMockMvcResultMatchers

```java
import static org.springframework.security.test.web.servlet.response.SecurityMockMvcResultMatchers.*;

mockMvc.perform(get("/profile").with(user("alice")))
    .andExpect(authenticated().withUsername("alice"))
    .andExpect(authenticated().withRoles("USER"));

mockMvc.perform(get("/public"))
    .andExpect(unauthenticated());
```

---

## Boot 3.x → 4.x Security Testing Changes

| Area | Boot 3.x / Security 6.x | Boot 4.x / Security 7.x |
|---|---|---|
| Security DSL | Lambda DSL (`.and()` deprecated since Security 6) | Lambda DSL only — `.and()` removed |
| `WebSecurityConfigurerAdapter` | Removed in Security 6 | Same — use `SecurityFilterChain` beans |
| `jwt()` post-processor | `spring-security-test` 6.x | `spring-security-test` 7.x (same API) |
| `AuthorizationManager` | Optional | Everywhere — `FilterSecurityInterceptor` removed |
| `@MockBean` in test classes | `org.springframework.boot.test.mock.mockito` | `@MockitoBean` from `org.springframework.test.context.bean.override.mockito` |

---

## Anti-Patterns

| Anti-Pattern | Fix |
|---|---|
| Using `@SpringBootTest` for all security tests | `@WebMvcTest` loads the security filter chain — use it for HTTP security tests |
| Only testing the happy path (200) | Always add unauthenticated (401) + wrong-role (403) tests |
| `@WithMockUser(roles = "ROLE_ADMIN")` | Use `roles = "ADMIN"` — the `ROLE_` prefix is added automatically |
| Forgetting `.with(csrf())` on POST/PUT/DELETE | Add `csrf()` or disable CSRF in a `@TestConfiguration` for stateless APIs |
| Disabling security with `addFilters = false` | You are not testing security — false confidence |
| Setting `SecurityContextHolder` manually in tests | Use `@WithMockUser`, `@WithUserDetails`, or post-processors |
