---
name: spring-security-testing
description: "Use this skill when the user asks to test authentication, test authorization rules, test role-based access control, test a secured endpoint, set up @WithMockUser, test CSRF protection, test JWT resource server endpoints, test OAuth2 login, test @PreAuthorize method security, mock a user in a Spring Security test, or write tests that verify 401 Unauthorized or 403 Forbidden responses. Also triggers on: org.springframework.security.test.context.support.WithMockUser, org.springframework.security.test.context.support.WithUserDetails, org.springframework.security.test.web.servlet.request.SecurityMockMvcRequestPostProcessors, csrf(), jwt(), oauth2Login(), mockOidcLogin(), @WithSecurityContext, SecurityFilterChain, hasRole, hasAuthority, AccessDeniedException, SCOPE_, JwtAuthenticationToken."
version: 0.1.0
license: Apache-2.0
---

# Spring Security Testing

**Signals**: `@WithMockUser`, `@WithUserDetails`, `csrf()`, `jwt()`, `oauth2Login()`, `mockOidcLogin()`, `@WithSecurityContext`, `SecurityMockMvcRequestPostProcessors`, `AccessDeniedException`, `SCOPE_`

## Tested With

- Spring Boot 3.2+ / Spring Boot 4.x
- Spring Security 6.x / 7.x
- Spring Framework 6.x / Spring Framework 7.x
- JUnit 5

## Do NOT Use This Skill When

- Testing REST controller HTTP behavior (status codes, JSON body, headers) without security focus → use `spring-mvc-testing`
- Testing reactive endpoints with security → use `spring-webflux-testing`
- Testing JPA repositories → use `spring-jpa-testing`
- Writing pure unit tests for service logic without security context → use `spring-testing-fundamentals`

## When to Read References

| Situation | Read |
|-----------|------|
| `@WithMockUser` and `@WithUserDetails` setup | `references/security-testing-patterns.md` |
| `roles` vs `authorities` and the `ROLE_` prefix trap | `references/security-testing-patterns.md` |
| Per-request auth with `SecurityMockMvcRequestPostProcessors` (`user()`, `httpBasic()`) | `references/security-testing-patterns.md` |
| CSRF on POST/PUT/DELETE — preventing 403 in tests | `references/security-testing-patterns.md` |
| JWT resource server testing with `jwt()` post-processor | `references/security-testing-patterns.md` |
| OAuth2 login and OIDC testing with `oauth2Login()`, `mockOidcLogin()` | `references/security-testing-patterns.md` |
| Custom `@WithSecurityContext` for complex JWT scenarios | `references/security-testing-patterns.md` |
| `@PreAuthorize` method security testing | `references/security-testing-patterns.md` |
| Boot 3.x → 4.x security testing changes | `references/security-testing-patterns.md` |
