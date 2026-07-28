# 🏛️ LLM Council: Comprehensive Guide & Technical Deep Dive

This document provides a complete technical summary of the **LLM Council** deliberation engine, its 5-stage workflow, peer-review mechanics, configuration model, and how it connects to agentic planning patterns documented in `planning_spec_drive-agents.pdf`. It uses Workflow Pattern (Compile-Time Fixed DAG).

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
└──────────────────────────────┬──────────────────────────────┘
```

| Stage | Who Executes It? | Input Received | Purpose |
| :--- | :--- | :--- | :--- |
| **Stage 1: Initial Responses** | All Council Models | User Query | Generate initial independent answers in parallel. Outputs are anonymized as `Response A`, `Response B`, `Response C`, etc. |
| **Stage 2: Peer Ranking** | All Council Models | Anonymized Responses A, B, C... | Rank & critique all answers blindly. **Kendall’s W Concordance Coefficient** is computed to measure statistical agreement. |
| **Stage 3: Agreement Analysis** | All Council Models | Anonymized Responses A, B, C... | Find common ground across answers, aggregated by topic and confidence level. |
| **Stage 4: Disagreement Analysis** | All Council Models | Anonymized Responses A, B, C... | Identify major conflicts & assign severity ratings (*Low*, *Medium*, *High*). |
| **Stage 5: Final Synthesis** | Designated **Chairman Model** | Full Deliberation Context (Stages 1–4) | Synthesize a single authoritative, peer-reviewed final answer. |

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

## 🎯 4. Planning Patterns & Detailed Stage Breakdown (`planning_spec_drive-agents.pdf`)

The document `planning_spec_drive-agents.pdf` (*"Plan before you build: Deterministic Planning Patterns for AI Agents"* by Dan Dobrin, Google Cloud App Architect) uses **LLM Council** as its primary benchmark domain to solve **agentic non-determinism**.

```
Deterministic (Zero Planning Cost) ──────────────────────────────► Bounded Flexibility (LLM-Driven)
   [Workflow] ───► [HTN] ───► [GOAP] ───► [Utility AI] ───► [LLM Compiler] ───► [SupervisorLLM]
```

Below is the detailed stage execution breakdown for each of the 6 planning strategies:

---

### Strategy 1: Workflow Pattern (Compile-Time Fixed DAGs — Implemented in this Repository)
* **Overview**: Execution graph is hard-coded at compile time using Google ADK's `SequentialAgent` and `ParallelAgent` (or Reactor `Flux`/`Mono` chains in `CouncilService`).
* **Planning LLM Calls**: **0** (Fixed at bean creation time).

| Stage | Who Executes It? | Input Received | Purpose |
| :--- | :--- | :--- | :--- |
| **Stage 1: Initial Response** | All Council Models (Parallel) | User Query | Generate initial independent responses. |
| **Stage 2: Peer Ranking** | All Council Models (Parallel) | Anonymized Responses (A, B, C...) | Anonymized cross-evaluation & Kendall's W consensus scoring. |
| **Stage 3: Agreement Analysis** | All Council Models (Parallel) | Anonymized Responses (A, B, C...) | Extract shared consensus points across models. *(Skipped in Quick preset)* |
| **Stage 4: Disagreement Analysis** | All Council Models (Parallel) | Anonymized Responses (A, B, C...) | Extract divergence points and severity scores. *(Skipped in Quick preset)* |
| **Stage 5: Final Synthesis** | Chairman Model Only | All Prior Stage Outputs | Synthesize final authoritative answer. |

---

### Strategy 2: HTN Pattern (Hierarchical Task Network)
* **Overview**: Uses an LLM classifier to determine query type, then decomposes compound deliberation tasks recursively into primitive tasks using priority-ordered recipes.
* **Planning LLM Calls**: **1** (Query Classifier).

| Stage | Who Executes It? | Input Received | Purpose |
| :--- | :--- | :--- | :--- |
| **Query Classification Stage** | Fast LLM Classifier | User Query | Classify query into 1 of 6 categories (e.g. *Factual*, *Opinion*, *Debate*, *Analytical*). |
| **HTN Decomposition Stage** | HTN Planner (Pure Java) | Query Category + Priority Methods | Select highest-priority method (e.g. `QuickFactual` p:10, `ConsensusOnly` p:20, `FullDelib` p:100) to emit ordered primitive tasks. |
| **Stage 1: Initial Responses** | All Council Models | User Query | Execute primitive initial response generation. |
| **Stage 2: Peer Ranking** | All Council Models | Anonymized Responses | Execute primitive peer evaluation. |
| **Stages 3 & 4: Agreement / Disagreement** | All Council Models | Anonymized Responses | **Dynamically Included or Skipped**: Included if query is *Debate* / *Analytical*; skipped if query is *Factual*. |
| **Stage 5: Final Synthesis** | Chairman Model | Accumulated Primitive Outputs | Primitive synthesis step producing final answer. |

---

### Strategy 3: GOAP Pattern (Goal-Oriented Action Planning)
* **Overview**: Uses A* graph search over a 10-variable world state with preconditions and effects. Planning completes in microseconds in Java with zero LLM calls.
* **Planning LLM Calls**: **0** (A* search in Java).

| Stage | Who Executes It? | Input Received | Purpose |
| :--- | :--- | :--- | :--- |
| **Goal Analysis Stage** | GoapQueryAnalyzer (Rule Engine) | User Instruction | Keyword rules map natural language to target world-state goals (e.g. `{FINAL_SYNTHESIS_AVAILABLE}`). |
| **A* Search Planning Stage** | A* Graph Search Planner | World State + Action Registry | Discovers cost-optimal sequence of 8 actions (5 LLM actions @ cost 10 + 3 Java aggregations @ cost 1). |
| **Plan Optimization Stage** | PlanOptimizer | A* Action Sequence | Computes dependency depth per action and collapses 8 sequential steps into 3 parallel execution steps. |
| **Execution & Adaptive Replanning Stages** | Action System | Unblocked World-State Variables | Dispatches unblocked actions concurrently; adaptive replanning skips disagreement steps if Kendall's W shows high consensus. |

---

### Strategy 4: Utility AI Pattern (OODA Execution Loop)
* **Overview**: Operates an Observe-Orient-Decide-Act (OODA) loop every decision tick, scoring eligible actions dynamically based on real-time world state.
* **Planning LLM Calls**: **0** (O(1) scoring evaluations per action).

| Stage | Who Executes It? | Input Received | Purpose |
| :--- | :--- | :--- | :--- |
| **Observe Stage** | OODA Evaluator | World State | Filter eligible actions whose preconditions are met and not in-progress. |
| **Orient Stage** | CompositeScorer | Eligible Actions + World State | Score candidates across 4 dimensions: *Readiness* (unblocked actions), *Progress* (pipeline depth), *Latency*, and *Info Gain*. |
| **Decide Stage** | Decision Engine | Scored Actions | Select all actions scoring $\ge 0.5$ threshold for emergent dynamic parallelism (or pick top scorer). |
| **Act Stage** | Executor | Selected Actions | Execute selected actions in parallel via `Flux.merge()`, update world state, and repeat until `NIRVANA` event (no eligible actions remain). |

---

### Strategy 5: LLM Compiler Pattern
* **Overview**: A single planning LLM call generates the complete task DAG as JSON upfront. A static validator verifies graph integrity before execution, falling back to GOAP A* if invalid.
* **Planning LLM Calls**: **1** (PlanCompiler upfront).

| Stage | Who Executes It? | Input Received | Purpose |
| :--- | :--- | :--- | :--- |
| **Compilation Stage** | PlanCompiler (1 LLM Call) | User Instruction | Emits entire task DAG as JSON: `{label, tasks[{id, agent, dependsOn}]}`. |
| **Validation Safety Net Stage** | PlanValidator (Static Java) | Compiled JSON DAG | Checks agent existence, unique IDs, graph acyclicity, precondition closure, and mandatory aggregate chains. (Falls back to GOAP A* if rejected). |
| **DAG Leveling Stage** | DagLeveler | Validated Task DAG | Levels DAG by dependency depth (`level = 1 + max(dependsOn.level)`) to group work into parallel steps (8 tasks $\rightarrow$ 3 parallel steps). |
| **Execution Stages** | LlmCompilerPlanner | Precompiled Leveled Steps | Walks precompiled steps in ADK runtime; replanning fires at most once between whole runs based on consensus metrics. |

---

### Strategy 6: SupervisorLLM Pattern
* **Overview**: An LLM supervisor dynamically chooses which agent(s) to run next step-by-step in an iterative loop, bounded by 3-layer safety nets.
* **Planning LLM Calls**: **3–5** (1 call per step iteration).

| Stage | Who Executes It? | Input Received | Purpose |
| :--- | :--- | :--- | :--- |
| **Observe Stage** | Supervisor Loop | State Map + Agent Catalog + Instruction | Builds complete prompt context representing completed outputs and current state. |
| **Supervisor Planning Stage** | Supervisor LLM | Prompts Context | Selects next agent(s) to run concurrently or emits `DONE` when complete. |
| **Agent Execution Stage** | Agent Dispatcher | Selected Agent(s) | Dispatches selected agents concurrently and writes structured outputs into shared state map. |
| **Safety Net Verification Stage** | 3-Layer Safety Validator | Proposed `DONE` Signal + State Map | Verifies 3 safety layers before termination: (1) Workflow rules (required agents ran), (2) Chain rules (aggregates present), (3) Scope filters (no out-of-scope agents). Hard cap of 10 iterations prevents runaway loops. |

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
