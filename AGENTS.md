# AGENTS.md — AI Agent Guidance for LLM Council (Java)

This document provides instructions, architectural context, development workflows, and guardrails for AI coding agents (`agy`, Claude, Antigravity, etc.) operating on the `llm-council-java-original` repository.

---

## 1. Project Overview & Architecture

**LLM Council** is a Java implementation of the Language Model Council deliberation process ([arXiv:2406.08598](https://arxiv.org/pdf/2406.08598)). Multiple LLMs deliberate together in a 5-stage workflow:

1. **Stage 1: Individual Responses** — Parallel execution across configured council models.
2. **Stage 2: Peer Ranking** — Anonymized cross-evaluation and Kendall's W consensus scoring.
3. **Stage 3: Agreement Analysis** — Topic-grouped consensus point extraction.
4. **Stage 4: Disagreement Analysis** — Divergence analysis and severity scoring.
5. **Stage 5: Final Synthesis** — Designated chairman model synthesizes final answer.

*Quick Consult mode*: A 3-stage variant (`POST /api/council/quick-consult`) skipping Stages 3 and 4.

### Tech Stack

- **Java Version**: Java 25 (with `--enable-preview` enabled in Maven compiler configuration)
- **Framework**: Spring Boot 4.0.5 & Spring AI 2.0.0-M8
- **UI & RPC Layer**: Vaadin Hilla 25.1.6 (React frontend + Spring Boot `@BrowserCallable` type-safe endpoints)
- **Reactive Engine**: Project Reactor (`Flux` / `Mono`) for async parallel model invocation
- **Caching & Storage**: Caffeine in-memory session cache + optional GCS / local file persistence
- **Model Integrations**: Anthropic API (`claude-opus-4-6`, `claude-sonnet-4-6`, `claude-haiku-4-5`), Google GenAI API (`gemini-3-5-flash`, `gemini-3-1-flash-lite`, `gemini-3-1-pro`), and Anthropic Vertex AI fallback (`anthropic-java-vertex`)

---

## 2. Developer Workflows & Commands

Agents **must** use the Maven Wrapper (`./mvnw`) and npm for project operations.

### Build & Compilation Commands
- **Compile Backend**:
  ```bash
  ./mvnw clean compile
  ```
- **Build Full Application (JAR with Hilla Frontend Bundle)**:
  ```bash
  ./mvnw clean package -Pproduction
  ```

### Run Commands
- **Development Mode**:
  ```bash
  ./mvnw spring-boot:run
  ```
- **Local Profile Execution**:
  ```bash
  ./mvnw spring-boot:run -Dspring-boot.run.profiles=local
  ```

### Mandatory Testing & Verification Rule
> [!IMPORTANT]
> Before concluding any task or reporting completion, agents **must** run unit and integration tests and ensure **0 compilation or test failures**:
> ```bash
> ./mvnw test
> ```
> If test or compilation errors occur, you must resolve them before finishing.

To run a specific test class:
```bash
./mvnw test -Dtest=CouncilServiceTest
```

---

## 3. Architecture & Code Conventions

### Project Directory Structure
```
src/main/java/dev/council/
├── LlmCouncilApplication.java         # Spring Boot entry point
├── client/                            # Spring AI ChatClient implementations & registry
├── config/                            # Spring Beans, Observability & Properties
├── endpoint/                          # Hilla @BrowserCallable endpoints for React UI
├── model/                             # Domain models, records, and stage DTOs
├── rest/                              # REST API controllers
├── service/                           # Core Reactive CouncilService logic & stage processors
└── support/                           # Utility & formatting helpers
```

### Coding Rules & Best Practices

1. **Reactive First (Project Reactor)**:
   - Use `Mono` and `Flux` for non-blocking asynchronous operations.
   - Do **not** block reactive threads (`.block()` or `.blockOptional()`) inside non-blocking pipelines unless explicitly required at boundary entry points.

2. **Modern Java 25 Idioms**:
   - Use `record` classes for immutable DTOs and value objects.
   - Leverage pattern matching (`switch`, `instanceof`) and modern Java constructs.
   - Keep preview features compatibility intact (`<enablePreview>true</enablePreview>`).

3. **Spring AI & ChatClient Abstraction**:
   - Route model requests through `ChatClientRegistry`.
   - Extend `AbstractChatClient` or implement `ChatClientProvider` for adding new model clients.
   - Preserve prompt templates and observation interceptors.

4. **Vaadin Hilla Endpoints**:
   - Annotate frontend-facing RPC services with `@BrowserCallable` in `dev.council.endpoint`.
   - Maintain type safety between TypeScript components and Java backend models.

---

## 5. Environment Variables & Security Guardrails

### Required Environment Variables
Ensure required API keys and project settings are configured via environment variables:
- `ANTHROPIC_API_KEY`: API key for Anthropic Claude direct access.
- `GEMINI_API_KEY`: API key for Google GenAI / Gemini access.
- `GOOGLE_CLOUD_PROJECT` / `GOOGLE_APPLICATION_CREDENTIALS`: GCP project ID and ADC credentials for Vertex AI / Cloud Storage / Cloud Trace.

### Security Rules
> [!CAUTION]
> - **Never hardcode API keys**, tokens, or GCP service account credentials in code, `application.properties`, or `council.yaml`.
> - Do not commit local credential files (`.env`, `*.json` key files, etc.).
> - Always read credentials from environment properties or Spring Boot configuration properties (`@ConfigurationProperties`).

---

## 6. Agent Operational Checklist

When implementing changes in this repository:
1. Locate target classes in `src/main/java/dev/council/`.
2. Ensure changes align with Spring AI and Reactive Project Reactor standards.
3. If changing Hilla endpoints, ensure TypeScript bindings compile cleanly.
4. Execute `./mvnw test` and confirm all tests pass.
5. Verify no sensitive credentials or temporary files are left in the repository.
