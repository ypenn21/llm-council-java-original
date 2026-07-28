# ⚡ LLM Compiler Pattern in LLM Council

This document provides a detailed technical guide to the **LLM Compiler** planning and orchestration pattern for AI agents, referencing [llm-council-explained.md](file:///home/user/llm-council-java-original/llm-council-explained.md) and `planning_spec_drive-agents.pdf`.

---

## 📌 1. What is the LLM Compiler Pattern?

Inspired by classical software compilers (like LLVM), the **LLM Compiler** pattern decouples **Compilation** from **Execution**.

Instead of making an expensive LLM supervisor call at every single step, a single planning LLM call generates the complete execution graph upfront as a structured JSON DAG: `{label, tasks[{id, agent, dependsOn}]}`.

### Key Concept
* **1 Planning LLM Call**: Replaces 5–10 iterative supervisor LLM calls with a single upfront compilation call.
* **Static Java Validation**: Before execution, a static `PlanValidator` runs 5 deterministic checks on the generated DAG.
* **GOAP Fallback Circuit Breaker**: If the compiled plan fails validation, the runtime automatically falls back to **GOAP A\* Graph Search** to guarantee 100% execution reliability.

---

## ⚙️ 2. Architecture & Pipeline

```
                       ┌─────────────────────────┐
                       │ User Instruction/Query  │
                       └────────────┬────────────┘
                                    │
                                    ▼
                       ┌─────────────────────────┐
                       │      PlanCompiler       │ (1 Planning LLM Call ──► JSON DAG)
                       └────────────┬────────────┘
                                    │ JSON DAG
                                    ▼
                       ┌─────────────────────────┐
                       │      PlanValidator      │ (Static Java Guardrail — 5 Passes)
                       └────────────┬────────────┘
                                    │
                       ┌────────────┴────────────┐
                       ▼                         ▼
                  [ Valid? ]                [ Invalid? ]
                       │                         │
                       ▼                         ▼
             ┌──────────────────┐      ┌──────────────────┐
             │    DagLeveler    │      │ Fallback: GOAP   │
             │ Level Grouping   │      │ A* Graph Search  │
             └─────────┬────────┘      └──────────────────┘
                       │ 3 Parallel Step Levels
                       ▼
             ┌──────────────────┐
             │ LlmCompilerPlanner│
             │   ADK Execution  │ (Executes leveled steps concurrently)
             └──────────────────┘
```

### Core Components:

1. **PlanCompiler (1 LLM Call)**:
   * Uses structured JSON schema output to compile the user query into a JSON plan:
     ```json
     {
       "label": "Council Deliberation",
       "tasks": [
         { "id": "t1", "agent": "GenerateResponses", "dependsOn": [] },
         { "id": "t2", "agent": "EvaluateRankings", "dependsOn": ["t1"] },
         { "id": "t3", "agent": "AnalyzeAgreements", "dependsOn": ["t1"] },
         { "id": "t4", "agent": "AnalyzeDisagreements", "dependsOn": ["t1"] },
         { "id": "t5", "agent": "AggregateRankings", "dependsOn": ["t2"] },
         { "id": "t6", "agent": "AggregateAgreements", "dependsOn": ["t3"] },
         { "id": "t7", "agent": "AggregateDisagreements", "dependsOn": ["t4"] },
         { "id": "t8", "agent": "SynthesizeFinal", "dependsOn": ["t5", "t6", "t7"] }
       ]
     }
     ```

2. **PlanValidator (Static Deterministic Java Guardrail — 0 LLM Cost)**:
   Executes 5 strict validation passes in Java before any agent executes:
   1. *Agent Registry Verification*: All referenced `agent` strings exist in `AgentRegistry`.
   2. *Unique ID Check*: Every task has a unique `id`.
   3. *Graph Acyclicity Verification*: Kahn's topological sort / DFS cycle detection guarantees no circular dependencies.
   4. *Precondition Closure*: All task IDs listed in `dependsOn` exist in the task array.
   5. *Mandatory Chain Verification*: Aggregations correctly follow corresponding stage outputs.

3. **Fallback Circuit Breaker**:
   If `PlanValidator` rejects the compiled DAG, the execution engine seamlessly switches to **GOAP A\* Search**, preventing system crashes and eliminating invalid LLM outputs.

4. **DagLeveler (Auto-Parallel Leveling)**:
   Groups tasks into parallel levels by calculating dependency depth:
   
   $$\text{level}(T) = 1 + \max_{d \in \text{dependsOn}(T)} \text{level}(d)$$
   
   * **Level 1**: `GenerateResponses`
   * **Level 2**: `EvaluateRankings`, `AnalyzeAgreements`, `AnalyzeDisagreements` *(Parallel)*
   * **Level 3**: `AggregateRankings`, `AggregateAgreements`, `AggregateDisagreements`, `SynthesizeFinal`

---

## 📊 3. Stage-by-Stage Breakdown for LLM Compiler

| Stage / Step | Executing Engine | Inputs Received | Preconditions & Validation Rules | Purpose |
| :--- | :--- | :--- | :--- | :--- |
| **0. Plan Compilation Stage** | PlanCompiler | User Instruction | **Output**: Structured JSON DAG (`{tasks:[{id, agent, dependsOn}]}`). | Compile user prompt into a complete task DAG in 1 LLM call. |
| **0. Static Validation Stage** | PlanValidator (Java) | Compiled JSON DAG | **5 Passes**: Agent existence, unique IDs, graph acyclicity, precondition closure, chain rules. | Validate graph integrity in Java (0 LLM cost); triggers GOAP fallback if invalid. |
| **0. DAG Leveling Stage** | DagLeveler (Java) | Validated JSON DAG | **Formula**: $\text{level}(T) = 1 + \max(\text{dependsOn.level})$. | Level tasks by dependency depth into parallel execution steps. |
| **Level 1: Response Generation** | All Council Models | User Query | **Level 1 Tasks**: `GenerateResponses`. | Execute initial parallel model outputs. |
| **Level 2: Evaluation & Analysis** | All Council Models (Parallel) | Initial Responses | **Level 2 Tasks**: `EvaluateRankings`, `AnalyzeAgreements`, `AnalyzeDisagreements`. | **Parallel Execution**: Concurrently execute peer ranking, agreement analysis, and disagreement analysis. |
| **Level 3: Aggregation & Synthesis** | Chairman Model & Java Actions | Stage Outputs | **Level 3 Tasks**: Aggregations + `SynthesizeFinal`. | Perform Java aggregations and synthesize final answer. |
| **Coarse Replanning Check** | LlmCompilerPlanner | Kendall's W Metric | **Condition**: Kendall's W consensus threshold check. | Between-run replanning fires at most once if consensus metric requires altering the DAG. |

---

## ⚖️ 4. Pros & Cons of LLM Compiler Pattern

### ✅ Advantages
* **Low Planning Latency & Cost**: Uses exactly 1 planning LLM call upfront, compared to 5–10 calls for supervisor loops.
* **Deterministic Safety Guardrails**: Static Java `PlanValidator` catches malformed plans before execution begins.
* **Bulletproof Reliability**: Fallback circuit breaker routes invalid plans to **GOAP A\***, guaranteeing 0% failure rate.
* **Automatic Parallel Leveling**: `DagLeveler` maximizes throughput by grouping independent tasks into parallel levels.
* **Inspectable & Auditable**: The compiled JSON plan can be logged, visualized, and stored for compliance.

### ❌ Trade-Offs & Limitations
* **Upfront Schema Sensitivity**: Planning LLM must strictly adhere to the expected JSON schema.
* **Coarse Replanning Only**: Mid-run adaptation is limited; replanning requires re-compiling the DAG between whole runs.
* **Static Execution within Levels**: Tasks within a level cannot dynamically trigger new unplanned tasks mid-flight.

---

## 🔗 Related Resources
* [llm-council-explained.md](file:///home/user/llm-council-java-original/llm-council-explained.md) — Main LLM Council Architecture & 6 Planning Patterns Overview
* [htn-planning-explained.md](file:///home/user/llm-council-java-original/htn-planning-explained.md) — HTN Planning Pattern Deep Dive
* [goap-planning-explained.md](file:///home/user/llm-council-java-original/goap-planning-explained.md) — GOAP Planning Pattern Deep Dive
* [utility-ai-explained.md](file:///home/user/llm-council-java-original/utility-ai-explained.md) — Utility AI Pattern Deep Dive
* `planning_spec_drive-agents.pdf` — Specification slide deck by Dan Dobrin (Google Cloud)
