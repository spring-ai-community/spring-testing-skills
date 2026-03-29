# spring-testing-skills

SkillsJars for testing Spring applications — curated patterns and annotated code examples
for JPA, REST, Security, WebFlux, WebSocket, and test fundamentals. Packaged for use with
AI coding agents (Claude Code, Kiro, etc.) via the [SkillsJars](https://skillsjars.com) format.

## Skills Included

| Skill | Covers |
|-------|--------|
| `spring-testing-router` | Meta-skill: routes "help me test my Spring app" prompts to the right domain skill |
| `spring-jpa-testing` | `@DataJpaTest`, Testcontainers + `@ServiceConnection`, Hibernate 6/7 pitfalls, entity lifecycle, lazy loading, N+1, SINGLE_TABLE inheritance |
| `spring-mvc-testing` | `@WebMvcTest`, `MockMvc`, `jsonPath`, validation, `@RestControllerAdvice`, Boot 4 `RestTestClient` |
| `spring-security-testing` | `@WithMockUser`, CSRF, JWT (`jwt()` post-processor), OAuth2, `@PreAuthorize` method security |
| `spring-webflux-testing` | `@WebFluxTest`, `WebTestClient`, `StepVerifier`, virtual time, SSE, reactive security |
| `spring-websocket-testing` | STOMP over WebSocket, `WebSocketStompClient`, `BlockingQueue` pattern, `@SendToUser` |
| `spring-testing-fundamentals` | AssertJ idioms, BDDMockito, `ArgumentCaptor`, `@MockitoBean`, context caching, testing pyramid, anti-patterns |

All skills target **Spring Boot 4.x / Spring Framework 7.x** with notes on Boot 3.x → 4.x package changes.

## For Consumers: Add Skills to Your Agent Project

### Maven

Add the dependency to your `pom.xml`:

```xml
<dependency>
  <groupId>org.springaicommunity</groupId>
  <artifactId>spring-testing-skills</artifactId>
  <version>0.9.0</version>
</dependency>
```

### Extract Skills to Your Agent

```bash
# Claude Code (global skills directory)
./mvnw skillsjars:extract -Ddir=~/.claude/skills

# Kiro (project-local)
./mvnw skillsjars:extract -Ddir=.kiro/skills
```

Skills are available to your agent on the next session start.

## For Contributors: Adding or Improving Skills

Each skill lives in `skills/<skill-name>/`:

```
skills/<skill-name>/
├── SKILL.md          — frontmatter (name, description trigger phrases, version) + routing body
└── references/
    └── *.md          — rich markdown with patterns and annotated code examples
```

**Adding a skill**: copy an existing skill directory, update `SKILL.md`, add reference files.

**Improving a skill**: edit reference files directly — no build step needed for content changes.

**Verify packaging**:

```bash
./mvnw clean package
jar tf target/spring-testing-skills-*.jar | grep META-INF/skills
```

### SKILL.md Structure

```yaml
---
name: your-skill-name
description: "Verbose description with BOTH natural language triggers AND fully qualified
  class/annotation names (e.g. org.springframework.boot.data.jpa.test.autoconfigure.DataJpaTest).
  Undertriggering is the primary failure mode — be generous."
version: 0.1.0
license: Apache-2.0
---

**Signals**: short list of key symbols

## Tested With
- Spring Boot version range

## Do NOT Use This Skill When
- Negative boundaries to prevent wrong-skill selection

## When to Read References
| Situation | Read |
```

## License

Apache License 2.0 — see [LICENSE](LICENSE).
