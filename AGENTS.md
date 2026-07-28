# AGENTS.md — AI Agent Guidance for LLM Council (Java)

This document provides instructions, architectural context, development workflows, agentic planning strategies, and guardrails for AI coding agents (`agy`, Claude, Antigravity, etc.) operating on the `llm-council-java-original` repository.

---

## 1. Project Overview & Deliberation Architecture

**LLM Council** is a Java implementation of the Language Model Council deliberation process ([arXiv:2406.08598](https://arxiv.org/pdf/2406.08598)), evolving ideas popularized by Karpathy's LLM Council. Multiple LLMs deliberate together in a structured 5-stage reactive workflow:

1. **Stage 1: Individual Responses** — Parallel execution across configured council models.
2. **Stage 2: Peer Ranking** — Anonymized cross-evaluation and Kendall's W consensus scoring.
3. **Stage 3: Agreement Analysis** — Topic-grouped consensus point extraction.
4. **Stage 4: Disagreement Analysis** — Divergence analysis and severity scoring.
5. **Stage 5: Final Synthesis** — Designated chairman model synthesizes the final answer.

*Quick Consult mode*: A 3-stage variant (`POST /api/council/quick-consult`) executing Stages 1, 2, and 5 while skipping Stages 3 and 4.

### Tech Stack

- **Java Version**: Java 25 (with `--enable-preview` enabled in Maven compiler configuration)
- **Framework**: Spring Boot 4.0.5 & Spring AI 2.0.0-M8
- **UI & RPC Layer**: Vaadin Hilla 25.1.6 (React frontend + Spring Boot `@BrowserCallable` type-safe endpoints)
- **Reactive Engine**: Project Reactor (`Flux` / `Mono`) for async parallel model invocation
- **Caching & Storage**: Caffeine in-memory session cache + optional GCS / local file persistence
- **Observability**: OpenTelemetry tracing, Cloud Trace exporter, Micrometer metrics, Cloud Logging
- **Model Integrations**: Anthropic API (`claude-opus-4-6`, `claude-sonnet-4-6`, `claude-haiku-4-5`), Google GenAI API (`gemini-3-5-flash`, `gemini-3-1-flash-lite`, `gemini-3-1-pro`), and Anthropic Vertex AI fallback (`anthropic-java-vertex`)

---

## 2. Agentic Planning Strategies in LLM Council

The repository includes a comprehensive set of 6 agentic planning strategies documented in [llm-strategies.md](file:///home/user/llm-council-java-original/llm-strategies.md) and detailed in the [llm-strategies/](file:///home/user/llm-council-java-original/llm-strategies) directory:

### Master Strategy Matrix

| Strategy | Planning LLM Cost | Determinism | Accuracy | Key Differentiator | Primary Selection Reason | Deep Dive Link |
| :--- | :---: | :---: | :---: | :--- | :--- | :--- |
| **Workflow Pattern** *(Repo Default)* | **$0.00** | **100%** | **10 / 10** | Compile-time fixed Java Reactor DAGs | **Guarantees 100% deliberation coverage** with zero planner risk | [llm-council-explained.md](file:///home/user/llm-council-java-original/llm-strategies/llm-council-explained.md) |
| **HTN** *(Hierarchical Task Network)* | **~$0.0001** | **80%** | **8.0 / 10** | 1 fast classifier LLM routes queries into pre-authored task recipes | **Saves ~40% tokens on simple queries** by skipping unneeded stages | [htn-planning-explained.md](file:///home/user/llm-council-java-original/llm-strategies/htn-planning-explained.md) |
| **GOAP** *(Goal-Oriented Action Planning)* | **$0.00** | **100%** | **9.5 / 10** | Microsecond A\* graph search + adaptive Kendall's W pruning | **#1 Overall Cost-Optimized Strategy** (25%–30% token savings) | [goap-planning-explained.md](file:///home/user/llm-council-java-original/llm-strategies/goap-planning-explained.md) |
| **Utility AI** *(OODA Execution Loop)* | **$0.00** | **70%** | **7.5 / 10** | Action scoring via $O(1)$ math functions across 4 weights | **Emergent runtime parallelism** tailored by config profiles | [utility-ai-explained.md](file:///home/user/llm-council-java-original/llm-strategies/utility-ai-explained.md) |
| **LLM Compiler** | **~$0.0005** | **85%** | **8.5 / 10** | 1 upfront JSON DAG compilation + static Java validation | **Flexible DAG creation** without heavy per-step LLM overhead | [llm-compiler-explained.md](file:///home/user/llm-council-java-original/llm-strategies/llm-compiler-explained.md) |
| **SupervisorLLM** | **~$0.0050+** | **30%** | **6.0–8.0** | Step-by-step LLM decision loop bounded by 3-layer safety net | **Maximum flexibility** for unpredictable, open-ended prompts | [supervisor-llm-explained.md](file:///home/user/llm-council-java-original/llm-strategies/supervisor-llm-explained.md) |

For cost optimization and trade-off analysis, see [cost-optimization.md](file:///home/user/llm-council-java-original/llm-strategies/cost-optimization.md).

---

## 3. Developer Workflows & Commands

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
- **Build Native Image (GraalVM)**:
  ```bash
  ./mvnw clean package -Pnative,production -DskipTests
  ```

### Execution Commands
- **Development Mode (Local Profile)**:
  ```bash
  ./mvnw spring-boot:run -Dspring-boot.run.profiles=local
  ```
- **Cloud / Default Execution**:
  ```bash
  ./mvnw spring-boot:run
  ```

### REST API Verification
- **Full Deliberation**:
  ```bash
  curl -sf -X POST http://localhost:8080/api/council/consult \
    -H "Content-Type: application/json" \
    -d '{"query": "Should we adopt event sourcing?"}'
  ```
- **Quick Consult**:
  ```bash
  curl -sf -X POST http://localhost:8080/api/council/quick-consult \
    -H "Content-Type: application/json" \
    -d '{"query": "Tradeoffs of CQRS?"}'
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

## 4. Project Directory Structure & Key Files

```
/home/user/llm-council-java-original/
├── AGENTS.md                               # AI agent instructions (this document)
├── README.md                               # Project documentation & architecture overview
├── llm-strategies.md                       # Master index for agentic planning strategies
├── llm-strategies/                         # Planning strategy deep dives
│   ├── llm-council-explained.md            # Workflow Pattern (compile-time DAG)
│   ├── htn-planning-explained.md           # HTN Planning
│   ├── goap-planning-explained.md          # Goal-Oriented Action Planning
│   ├── utility-ai-explained.md             # Utility AI OODA Loop
│   ├── llm-compiler-explained.md           # LLM Compiler
│   ├── supervisor-llm-explained.md         # Supervisor LLM
│   └── cost-optimization.md                # Cost & benchmark trade-off analysis
├── src/main/java/dev/council/
│   ├── LlmCouncilApplication.java         # Spring Boot entry point
│   ├── client/                            # ChatClient implementations & registry
│   ├── config/                            # Spring Beans, Observability & Properties
│   ├── endpoint/                          # Hilla @BrowserCallable & REST endpoints
│   ├── model/                             # Domain models, records, and stage DTOs
│   ├── rest/                              # REST API controllers
│   ├── service/                           # Reactive CouncilService & stage processors
│   └── support/                           # Utility & formatting helpers
├── src/main/frontend/
│   ├── App.tsx                            # React router setup
│   ├── views/                             # MainLayout, CouncilView, TracingView
│   └── components/                        # Stage review panels & consensus visualizers
└── src/main/resources/
    ├── application.properties             # Spring & model configuration
    ├── council.yaml                       # Active council members & chairman settings
    └── system/*.st                        # StringTemplate stage prompt files
```

---

## 5. Coding Rules, Conventions & Customization

### Coding Guidelines
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

5. **Prompt Customization**:
   - Prompts for all 5 stages live in `src/main/resources/system/*.st` as StringTemplate files. Edit these directly to tune stage behaviors without Java code changes.

6. **Model & Role Configuration (`council.yaml`)**:
   - Active members, available pool, chairman, title generator, and summarizer models are configured in `council.yaml`.

---

## 6. Environment Variables & Security Guardrails

### Required Environment Variables
Configure credentials and environment settings via environment variables:
- `ANTHROPIC_API_KEY`: API key for direct Anthropic Claude access (when `VERTEX_AI_ANTHROPIC_ENABLED=false`).
- `GOOGLE_CLOUD_PROJECT`: GCP project ID for Google GenAI, Cloud Trace, Stackdriver, and GCS.
- `GOOGLE_CLOUD_LOCATION`: GCP region (e.g. `us-central1` or `global`).
- `VERTEX_AI_ANTHROPIC_ENABLED`: Set to `true` (default) to route Claude through Vertex AI using ADC.
- `CONVERSATIONS`: Set to `local` (default) or `cloud` for session persistence location.

### Security Rules
> [!CAUTION]
> - **Never hardcode API keys**, tokens, or GCP service account credentials in code, `application.properties`, or `council.yaml`.
> - Do not commit local credential files (`.env`, `*.json` key files, etc.).
> - Always read credentials from environment properties or Spring Boot configuration properties (`@ConfigurationProperties`).

---

## 7. Agent Operational Checklist

When implementing changes in this repository:
1. Locate target classes in `src/main/java/dev/council/` or frontend components in `src/main/frontend/`.
2. Ensure changes align with Spring AI and Reactive Project Reactor standards.
3. If changing Hilla endpoints, ensure TypeScript bindings compile cleanly (`./mvnw clean compile`).
4. Execute `./mvnw test` and confirm **0 test or compilation failures**.
5. Verify no sensitive credentials, API keys, or temporary files are left in the repository.

