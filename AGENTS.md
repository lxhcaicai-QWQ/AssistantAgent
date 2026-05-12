# AGENTS.md

## Build & Run

```bash
mvn clean install -DskipTests    # Full build (skip tests)
mvn test                          # Run all tests
mvn test -Dtest=ClassName        # Run single test class
```

Start the app (requires `DASHSCOPE_API_KEY` env var):
```bash
cd assistant-agent-start && mvn spring-boot:run
```

Main class: `com.alibaba.assistant.agent.start.AssistantAgentApplication`

## Architecture

Multi-module Maven project (Java 17, Spring Boot 3.4, Spring AI 1.1.0).

**Module dependency chain:**
`common` -> `core` -> `evaluation` / `prompt-builder` -> `extensions` -> `autoconfigure` -> `start`

| Module | Purpose |
|---|---|
| `assistant-agent-common` | Shared enums, constants, interfaces, utilities |
| `assistant-agent-core` | GraalVM sandbox executor, tool registry, CodeAct engine |
| `assistant-agent-evaluation` | Evaluation graph engine for multi-dimensional intent recognition |
| `assistant-agent-prompt-builder` | Dynamic prompt assembly from evaluation results & runtime context |
| `assistant-agent-extensions` | Extension modules (experience, learning, search, reply, trigger, dynamic tools) |
| `assistant-agent-management` | Experience CRUD APIs + SKILL import/export controllers |
| `assistant-agent-autoconfigure` | Spring Boot auto-configuration, wires everything together |
| `assistant-agent-start` | Runnable Spring Boot app with demo config |

**Extension sub-modules** live inside `assistant-agent-extensions` under `com.alibaba.assistant.agent.extension.*`: dynamic, experience, learning, search, reply, trigger, evaluation integration.

## Key Tech & Gotchas

- **Lombok** is used in `common` and `core` -- enable annotation processing in IDE
- **GraalVM Polyglot** sandbox executes AI-generated code (JS/Python) at runtime; optional dependency but pulled in by `start` module
- **Spring AI Alibaba** (`spring-ai-alibaba-starter-dashscope`) provides the LLM backend; default model is `qwen-max`
- **MCP client** is included via `spring-ai-starter-mcp-client` in autoconfigure; disabled by default (`spring.ai.mcp.client.enabled=false`)
- **SPI pattern**: Search providers (`SearchProvider`), reply channels, and other extensions use Spring auto-detection via `@Component`
- Config prefix for all extensions: `spring.ai.alibaba.codeact.extension.*`
- Full config reference: `assistant-agent-start/src/main/resources/application-reference.yml`

## Code Style

- Google Java Style (4-space indent, 120 char line limit)
- Apache 2.0 license header on all source files
- Javadoc required on all public APIs
- Logging format: `ClassName#methodName - description: key={value}`
- Test naming: `shouldReturnXWhenY` pattern
- Commit messages: Conventional Commits (`feat:`, `fix:`, `docs:`, `refactor:`, `test:`)

## Testing

- JUnit 5 (`junit-jupiter`)
- Spring Boot Test (`spring-boot-starter-test`) in extensions and evaluation modules
- Target coverage: >60% on new code
- Tests do NOT require `DASHSCOPE_API_KEY` (unit tests only; no integration tests against LLM)

## CI

GitHub Actions workflow (`.github/workflows/build.yml`): `mvn clean install -B -V` then `mvn test -B` on JDK 17 (temurin).
