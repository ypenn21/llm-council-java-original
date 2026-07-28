# 🏛️ LLM Council: Comprehensive Guide & Technical Deep Dive

This document provides a complete technical summary of the **LLM Council** deliberation engine, its 5-stage workflow, peer-review mechanics, configuration model, and how it connects to agentic planning patterns documented in `planning_spec_drive-agents.pdf`.

---

## 📌 1. Project Overview & Use Case

**LLM Council** is a Java implementation of the language model deliberation process based on research from [arXiv:2406.08598](https://arxiv.org/pdf/2406.08598).

### The Problem
Single LLMs often generate confident hallucinations, suffer from single-model bias, or miss critical logical nuances when answering complex or high-stakes queries.

### The Solution
Instead of relying on a single AI model, **LLM Council** orchestrates multiple diverse LLMs (e.g., Claude, Gemini) in a structured 5-stage deliberation process where models anonymously peer-review, rank, and analyze each other's work before a designated chairman synthesizes the final answer.

### Primary Applications
* **High-Stakes Decision Making**: Comparing reasoning paths across multiple AI architectures.
* **Peer-Reviewed Output**: Minimizing hallucinations through mandatory cross-model evaluation.
* **Consensus & Divergence Mapping**: Explicitly identifying where AI models agree vs. where they disagree (along with severity ratings).

---

## 🔄 2. The 5-Stage Deliberation Workflow

```
┌─────────────────────────────────────────────────────────────┐
│ Stage 1: Individual Responses (Parallel Model Generation)   │
└──────────────────────────────┬──────────────────────────────┘
                               ▼
┌─────────────────────────────────────────────────────────────┐
│ Stage 2: Peer Ranking & Consensus (Blind Peer Review)       │
└──────────────────────────────┬──────────────────────────────┘
                               ▼
┌─────────────────────────────────────────────────────────────┐
│ Stage 3: Agreement Analysis (Extract Shared Ground)         │
└──────────────────────────────┬──────────────────────────────┘
                               ▼
┌─────────────────────────────────────────────────────────────┐
│ Stage 4: Disagreement Analysis (Identify Divergences)       │
└──────────────────────────────┬──────────────────────────────┘
                               ▼
┌─────────────────────────────────────────────────────────────┐
│ Stage 5: Final Synthesis (Chairman Synthesizes Answer)      │
└─────────────────────────────────────────────────────────────┘
```

| Stage | Executing Models | Inputs Received | What Happens |
| :--- | :--- | :--- | :--- |
| **Stage 1: Individual Responses** | All Council Models | Original User Query | Each model generates an independent answer in parallel. Outputs are anonymized as `Response A`, `Response B`, `Response C`, etc. |
| **Stage 2: Peer Ranking** | All Council Models | Anonymized Responses (A, B, C...) | Every model blindly critiques and ranks all responses. **Kendall’s W Concordance Coefficient** is computed to measure statistical agreement. |
| **Stage 3: Agreement Analysis** | All Council Models | Anonymized Responses | Models extract key points of agreement, which are aggregated by topic and confidence level. |
| **Stage 4: Disagreement Analysis** | All Council Models | Anonymized Responses | Models extract conflicting advice/assumptions and assign severity ratings (*Low*, *Medium*, *High*). |
| **Stage 5: Final Synthesis** | Designated **Chairman Model** | Full Deliberation Packet (Stages 1–4) | The Chairman reads all outputs and synthesizes a single, authoritative, high-quality final response. |

---

## 🤝 3. Peer Review & Chairman Mechanics

### How Blind Peer Review Works (Stages 2, 3, 4)
1. **Anonymization**: Before Stage 2 begins, the server creates a private mapping between model IDs and single-letter labels (`Response A`, `Response B`, etc.). Model identities are stripped from the prompt.
2. **Every Model Participates**: In Stages 2, 3, and 4, **all council models evaluate all responses** (including their own, without knowing which response is theirs).
3. **Server-Side Aggregation**: Results are aggregated on the backend in `CouncilService.java` to prevent client manipulation or trust boundary violations.

### Why Gemini 3.1 Flash Lite is Default Chairman
In `council.yaml`, `gemini-3-1-flash-lite` is set as the default chairman model because:
1. **Large Context Efficiency**: Stage 5 passes the entire cumulative context (all responses + all peer rankings + agreement & disagreement summaries). A lightweight model keeps token costs low.
2. **High Speed / Low Latency**: Faster Time-To-First-Token (TTFT) reduces user wait time at the end of the pipeline.
3. **Developer-Friendly Default**: Ideal for local testing. It can be swapped to `claude-opus-4-6` or `gemini-3-1-pro` simply by changing `council.chairman.model` in `council.yaml`.

---

## 🎯 4. Planning Patterns (`planning_spec_drive-agents.pdf`)

The document `planning_spec_drive-agents.pdf` (*"Plan before you build: Deterministic Planning Patterns for AI Agents"* by Dan Dobrin, Google Cloud App Architect) uses **LLM Council** as its primary benchmark domain to solve **agentic non-determinism**.

```
Deterministic (Zero Planning Cost) ──────────────────────────────► Bounded Flexibility (LLM-Driven)
   [Workflow] ───► [HTN] ───► [GOAP] ───► [Utility AI] ───► [LLM Compiler] ───► [SupervisorLLM]
```

### The 6 Planning Patterns Compared on LLM Council

1. **Workflow Pattern (Implemented in this Repository)**:
   * Compile-time fixed DAGs via Project Reactor (`Flux`/`Mono`).
   * **Planning Cost**: 0 extra LLM calls.
   * **Presets**: Full Consult (5 stages), Quick Consult (3 stages: 1, 2, 5), Rank Mode.

2. **HTN (Hierarchical Task Network)**:
   * 1 LLM call classifies query type (Factual vs. Debate) to select a pre-authored deliberation recipe.

3. **GOAP (Goal-Oriented Action Planning)**:
   * A* graph search over a 10-variable world state. Computes cost-optimal execution paths with auto-discovered concurrency in microseconds (0 LLM planning calls).

4. **Utility AI**:
   * OODA execution loop scoring eligible actions using weighted functions (*Readiness, Progress, Latency, Info Gain*).

5. **LLM Compiler**:
   * 1 planning LLM call outputs the entire task DAG as JSON upfront, validated by a static `PlanValidator` before execution (falls back to GOAP A* on validation failure).

6. **SupervisorLLM**:
   * An LLM dynamically plans the next agent step-by-step in an iterative loop bounded by 3-layer safety nets.

---

## 🛠️ 5. Technical Stack & Architecture

* **Java Version**: Java 25 (with `--enable-preview`)
* **Framework**: Spring Boot 4.0.5 & Spring AI 2.0.0-M8
* **UI & RPC Layer**: Vaadin Hilla 25.1.6 (React frontend + Spring Boot `@BrowserCallable` type-safe endpoints)
* **Reactive Engine**: Project Reactor (`Flux` / `Mono`) for parallel model execution and event streaming
* **Caching & Storage**: Caffeine in-memory session cache + optional GCS / local file persistence
* **Configuration**: `council.yaml` for dynamic member and chairman model selection

---

### Key Code Artifacts
* `src/main/resources/council.yaml` — Configures active council models, chairman model, and available models.
* `src/main/java/dev/council/service/CouncilService.java` — Core reactive 5-stage deliberation pipeline.
* `src/main/java/dev/council/endpoint/CouncilApiController.java` — REST API endpoints (`/consult`, `/quick-consult`).
* `src/main/java/dev/council/endpoint/CouncilEndpoint.java` — Vaadin Hilla RPC endpoints for React UI.
