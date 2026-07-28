# 👑 SupervisorLLM Pattern in LLM Council

This document provides a detailed technical guide to the **SupervisorLLM** planning and orchestration pattern for AI agents, referencing [llm-council-explained.md](file:///home/user/llm-council-java-original/llm-council-explained.md) and `planning_spec_drive-agents.pdf`.

---

## 📌 1. What is the SupervisorLLM Pattern?

The **SupervisorLLM** pattern represents the most flexible, LLM-driven end of the agentic planning spectrum.

Instead of pre-authoring execution graphs or state tables, a central **LLM Supervisor** operates in an iterative step-by-step loop. Every iteration, the supervisor inspects the current world state, consults an available worker agent catalog, and dynamically selects which agent(s) to execute next (or emits a `DONE` signal when finished).

### Key Concept
* **Maximum Flexibility**: Adapts dynamically to novel prompts and unpredictable user requests.
* **Iterative Step-by-Step Decisions**: Re-evaluates state after every agent execution step.
* **3-Layer Safety Net Guardrails**: Deterministic Java validators enforce workflow rules, chain integrity, and iteration caps to prevent supervisor hallucinations or infinite loops.

---

## ⚙️ 2. Architecture & Pipeline

```
                               ┌─────────────────────────┐
                               │ User Instruction/Query  │
                               └────────────┬────────────┘
                                            │
                                            ▼
   ┌────────────────────────────────────────────────────────────────────────┐
   │                        SUPERVISOR ITERATIVE LOOP                       │
   │                                                                        │
   │   ┌───────────────────┐               ┌──────────────────────────┐    │
   │   │  1. Observe State │               │ 2. Supervisor LLM Call   │    │
   │   │                   ├──────────────►│                          │    │
   │   │  State Map +      │               │  Structured JSON Choice  │    │
   │   │  Agent Catalog    │               │  selectedAgents / isDone │    │
   │   └───────────────────┘               └────────────┬─────────────┘    │
   │             ▲                                      │                  │
   │             │                                      ▼                  │
   │   ┌─────────┴─────────┐               ┌──────────────────────────┐    │
   │   │  3. Agent Dispatch│               │ 4. Safety Validation     │    │
   │   │                   │               │                          │    │
   │   │  Concurrent Flux  │◄──────────────┤ Check 3 Safety Layers    │    │
   │   │  Execution        │   If NOT Done │ If isDone=true           │    │
   │   └───────────────────┘               └────────────┬─────────────┘    │
   └────────────────────────────────────────────────────┼───────────────────┘
                                                        │
                                                        ▼
                                           [ Approved Final Output ]
```

### Core Components:

1. **Supervisor Prompt & Catalog Context**:
   Every iteration, the supervisor receives a prompt containing:
   * **Agent Catalog**: Available worker agents (`GenerateResponses`, `EvaluateRankings`, `AnalyzeAgreements`, `AnalyzeDisagreements`, `SynthesizeFinal`, `AggregateRankings`, `AggregateAgreements`, `AggregateDisagreements`).
   * **State Map**: Completed task keys, outputs, and metrics (e.g. Kendall's W).
   * **User Instruction & Guidelines**: Goal requirements.

2. **Structured JSON Decision Schema**:
   The supervisor returns a structured JSON payload:
   ```json
   {
     "thought": "Initial responses complete. Dispatching peer evaluation and agreement analysis concurrently.",
     "selectedAgents": ["EvaluateRankings", "AnalyzeAgreements", "AnalyzeDisagreements"],
     "isDone": false
   }
   ```

3. **Concurrent Agent Dispatcher**:
   Dispatches all `selectedAgents` simultaneously using reactive streams (`Flux.merge()`), updating `stateMap` as outputs complete.

4. **3-Layer Safety Net Validator (Deterministic Guardrails)**:
   Because LLMs can hallucinate or terminate prematurely, the engine enforces 3 safety layers before accepting `isDone = true`:
   1. *Workflow Completeness Rules*: Mandatory core agents (`GenerateResponses` and `SynthesizeFinal`) must exist in `stateMap`.
   2. *Chain Integrity Rules*: Aggregations and evaluation steps must precede synthesis.
   3. *Scope Filter*: Blocks attempts to invoke non-existent or out-of-scope agents.
   4. *Hard Iteration Cap*: Forces completion after max **10 iterations** to prevent infinite loops.

---

## 📊 3. Stage-by-Stage Breakdown for SupervisorLLM

| Iteration / Phase | Executing Agent / Engine | Inputs Received | Output & Safety Validation Rules | Purpose |
| :--- | :--- | :--- | :--- | :--- |
| **Iter 1: Observation & Planning** | Supervisor LLM | User Instruction + Catalog + `{}` | **Choice**: `selectedAgents: ["GenerateResponses"]`, `isDone: false`. | Supervisor reviews empty state and dispatches initial model generation. |
| **Iter 1: Execution** | All Council Models | User Query | **State Map Update**: `{RESPONSES_AVAILABLE: true}`. | Execute initial parallel model outputs. |
| **Iter 2: Observation & Planning** | Supervisor LLM | Updated State Map + Catalog | **Choice**: `selectedAgents: ["EvaluateRankings", "AnalyzeAgreements", "AnalyzeDisagreements"]`. | Supervisor detects responses and dispatches Stage 2, 3, and 4 concurrently. |
| **Iter 2: Execution** | Council Models (`Flux.merge`) | Anonymized Responses | **State Map Update**: Rankings, agreements, and disagreements added to state. | Concurrently execute peer review and analytical stages. |
| **Iter 3: Observation & Planning** | Supervisor LLM | State Map + Stage Outputs | **Choice**: `selectedAgents: ["SynthesizeFinal"]`. | Supervisor detects completed analysis and dispatches final synthesis. |
| **Iter 3: Execution** | Chairman Model | Full Deliberation Context | **State Map Update**: `{FINAL_SYNTHESIS: "..."}`. | Chairman synthesizes authoritative answer. |
| **Iter 4: Completion & Safety Net** | 3-Layer Safety Validator | State Map + `isDone: true` | **Safety Check**: Verify workflow rules, chain integrity, and scope filters. | Approve completion or force missing mandatory steps (hard cap @ 10 iterations). |

---

## ⚖️ 4. Pros & Cons of SupervisorLLM Pattern

### ✅ Advantages
* **Maximum Flexibility & Adaptability**: Dynamic reasoning loop handles arbitrary, highly non-standard user prompts.
* **No Pre-Authored Graph Required**: Zero need to code DAGs, state machines, or A\* tables upfront.
* **Human-Like Reasoning Trace**: The `thought` field in each JSON decision provides a natural language audit trail of agent reasoning.
* **Dynamic Concurrency**: Supervisor can choose to run 1, 2, or 3 agents concurrently depending on context.

### ❌ Trade-Offs & Limitations
* **Highest Planning Latency & Cost**: Requires 3–5 additional LLM calls per deliberation run just for orchestration.
* **Non-Deterministic Plan Generation**: Different runs on the same prompt may choose different agent execution orders.
* **Hallucination Risk**: Requires strict **3-layer safety net guardrails** to prevent premature termination or out-of-scope tool invocation.
* **Higher Token Overhead**: Supervisor prompt accumulates state and output history over multiple iterations.

---

## 🔗 Related Resources
* [llm-council-explained.md](file:///home/user/llm-council-java-original/llm-council-explained.md) — Main LLM Council Architecture & 6 Planning Patterns Overview
* [htn-planning-explained.md](file:///home/user/llm-council-java-original/htn-planning-explained.md) — HTN Planning Pattern Deep Dive
* [goap-planning-explained.md](file:///home/user/llm-council-java-original/goap-planning-explained.md) — GOAP Planning Pattern Deep Dive
* [utility-ai-explained.md](file:///home/user/llm-council-java-original/utility-ai-explained.md) — Utility AI Pattern Deep Dive
* [llm-compiler-explained.md](file:///home/user/llm-council-java-original/llm-compiler-explained.md) — LLM Compiler Pattern Deep Dive
* `planning_spec_drive-agents.pdf` — Specification slide deck by Dan Dobrin (Google Cloud)
