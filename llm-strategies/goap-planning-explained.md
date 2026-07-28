# 🎮 GOAP (Goal-Oriented Action Planning) in LLM Council

This document provides a comprehensive guide to the **Goal-Oriented Action Planning (GOAP)** strategy for AI agents, referencing the architecture from [llm-council-explained.md](file:///home/user/llm-council-java-original/llm-council-explained.md) and `planning_spec_drive-agents.pdf`.

---

## 📌 1. What is GOAP Planning?

**Goal-Oriented Action Planning (GOAP)** is an AI planning technique originally invented for game AI (*F.E.A.R.*, 2005) and adapted for agentic orchestration. 

Instead of hard-coding execution DAGs or calling an expensive supervisor LLM at every step, GOAP uses **A\* Graph Search** in pure Java to find the globally cheapest sequence of actions that transforms the **Current World State** into a desired **Goal State**.

### Key Advantages in Agent Orchestration:
* **Zero Planning LLM Cost**: Planning completes in microseconds in Java without calling an LLM.
* **Auto-Discovered Parallelism**: `PlanOptimizer` automatically groups independent actions into parallel execution layers based on precondition dependency depth.
* **100% Deterministic & Auditable**: Given the same initial state and goals, A\* search always produces the exact same optimal plan.

---

## ⚙️ 2. The GOAP Architecture & Pipeline

```
                       ┌─────────────────────────┐
                       │ User Instruction/Query  │
                       └────────────┬────────────┘
                                    │
                                    ▼
                       ┌─────────────────────────┐
                       │    GoapQueryAnalyzer    │ (Rule-based keyword matching, 0 LLM calls)
                       └────────────┬────────────┘
                                    │ Target World-State Goals
                                    ▼
                       ┌─────────────────────────┐
                       │     A* Search Engine    │ (Cost-optimal action search, 0 LLM calls)
                       └────────────┬────────────┘
                                    │ 8 Sequential Actions
                                    ▼
                       ┌─────────────────────────┐
                       │      PlanOptimizer      │ (Computes dependency depth)
                       └────────────┬────────────┘
                                    │ Collapses 8 Sequential Actions ──► 3 Parallel Steps
                                    ▼
                       ┌─────────────────────────┐
                       │       GoapPlanner       │
                       │  Execution + Replanning │ (Executes steps + consensus-driven replanning)
                       └─────────────────────────┘
```

### Core Components:

1. **10-Variable World State**:
   A boolean state map tracking available data products:
   * `{QUERY_AVAILABLE}`
   * `{RESPONSES_AVAILABLE}`
   * `{RANKINGS_AVAILABLE}`
   * `{AGREEMENTS_AVAILABLE}`
   * `{DISAGREEMENTS_AVAILABLE}`
   * `{SYNTHESIS_AVAILABLE}`
   * `{AGGREGATE_RANKINGS_AVAILABLE}`
   * `{AGGREGATE_AGREEMENTS_AVAILABLE}`
   * `{AGGREGATE_DISAGREEMENTS_AVAILABLE}`
   * `{FINAL_SYNTHESIS_COMPLETED}`

2. **GoapQueryAnalyzer (0 LLM Calls)**:
   * Uses rule-based keyword matching on the user instruction to select target goals:
     * *Full Mode*: `{FINAL_SYNTHESIS_COMPLETED: true}`
     * *Quick Mode*: `{RANKINGS_AVAILABLE: true, SYNTHESIS_AVAILABLE: true}`
     * *Ranking Mode*: `{AGGREGATE_RANKINGS_AVAILABLE: true}`

3. **Action System (Preconditions + Effects + Costs)**:
   Declared in `GoapActionRegistry` with numeric costs:
   * **5 LLM Actions** (Cost: **10** each): `GenerateResponses`, `EvaluateRankings`, `AnalyzeAgreements`, `AnalyzeDisagreements`, `SynthesizeFinal`.
   * **3 Java Aggregation Actions** (Cost: **1** each): `AggregateRankings`, `AggregateAgreements`, `AggregateDisagreements`.

4. **PlanOptimizer (Auto-Parallelism)**:
   * Computes the dependency depth of each action based on its preconditions and effects.
   * Actions at the same dependency level run concurrently as a single parallel step.
   * Collapses an **8-step sequential A\* plan into 3 parallel execution layers**.

---

## 🧮 3. The A\* Search Process

The GOAP planner runs an **A\* Graph Search** over the state space:

$$f(n) = g(n) + h(n)$$

* $g(n)$: Total accumulated action cost so far (LLM actions cost 10, Java actions cost 1).
* $h(n)$: Admissible heuristic estimate of remaining cost to reach goal state (never overestimates).

```
[Start Node: {QUERY_AVAILABLE}] 
            │
            ├─► Action: GenerateResponses (Cost 10) ──► [{RESPONSES_AVAILABLE}]
            │                                                 │
            │          ┌──────────────────────────────────────┴──────────────────────────────────────┐
            │          ▼                                      ▼                                      ▼
            │   Action: EvaluateRankings               Action: AnalyzeAgreements              Action: AnalyzeDisagreements
            │   (Cost 10)                              (Cost 10)                              (Cost 10)
            │          │                                      │                                      │
            │          ▼                                      ▼                                      ▼
            │   [{RANKINGS_AVAILABLE}]                 [{AGREEMENTS_AVAILABLE}]               [{DISAGREEMENTS_AVAILABLE}]
            │          │                                      │                                      │
            │          ▼                                      ▼                                      ▼
            │   Action: AggregateRankings              Action: AggregateAgreements            Action: AggregateDisagreements
            │   (Cost 1)                               (Cost 1)                               (Cost 1)
            │          │                                      │                                      │
            └──────────┴──────────────────────────────────────┼──────────────────────────────────────┘
                                                              ▼
                                                   Action: SynthesizeFinal (Cost 10)
                                                              │
                                                              ▼
                                               [Goal Node: {FINAL_SYNTHESIS_COMPLETED}]
```

Because LLM actions cost 10 and Java actions cost 1, A\* guarantees finding the globally minimum-cost plan.

---

## 📊 4. Stage-by-Stage Breakdown for GOAP Planning

| Stage | Executing Agent / Engine | Inputs Received | Preconditions & Effects | Purpose |
| :--- | :--- | :--- | :--- | :--- |
| **0. Goal Analysis Stage** | GoapQueryAnalyzer (Rules) | Natural Language User Instruction | **Precondition**: User prompt.<br>**Effect**: Sets target state goals (e.g. `{FINAL_SYNTHESIS_COMPLETED: true}`). | Map user instruction to target state variables without calling an LLM (0 LLM cost). |
| **0. A\* Search Planning Stage** | A\* Graph Search Planner | Current World State + Target Goals | **Precondition**: Action registry.<br>**Effect**: Returns cost-optimal action sequence. | Microsecond search for the globally cheapest path to the goal state. |
| **0. Plan Optimization Stage** | PlanOptimizer (Java) | A\* Action Sequence | **Precondition**: Preconditions/effects of plan.<br>**Effect**: Parallel step levels. | Compute dependency depth per action to group independent actions into parallel execution layers (8 steps $\rightarrow$ 3 parallel steps). |
| **Step 1: Response Generation** | All Council Models | `{QUERY_AVAILABLE}` | **Precondition**: `{QUERY_AVAILABLE}`<br>**Effect**: `{RESPONSES_AVAILABLE}` | Execute initial parallel model generation (Cost 10). |
| **Step 2 (Parallel Layer): Peer Evaluation & Analysis** | All Council Models | `{RESPONSES_AVAILABLE}` | **Precondition**: `{RESPONSES_AVAILABLE}`<br>**Effect**: `{RANKINGS_AVAILABLE}`, `{AGREEMENTS_AVAILABLE}`, `{DISAGREEMENTS_AVAILABLE}` | Concurrently execute Stage 2 peer ranking, Stage 3 agreement analysis, and Stage 4 disagreement analysis. |
| **Step 2 (Parallel Layer): Aggregation** | Java Aggregation Actions | `{RANKINGS_AVAILABLE}`, etc. | **Precondition**: Stage outputs.<br>**Effect**: Set aggregate state variables. | Perform pure-computation Java data aggregations (Cost 1 each). |
| **Step 3: Adaptive Replanning Check** | GoapPlanner Engine | Kendall's W Metric | **Effect**: Skips steps if state threshold met. | If Kendall's W shows high consensus ($W > 0.8$), updates world state to skip disagreement analysis. |
| **Step 3: Final Synthesis** | Chairman Model | Full Deliberation Context | **Precondition**: All required aggregates.<br>**Effect**: `{FINAL_SYNTHESIS_COMPLETED}` | Generate final synthesized answer (Cost 10). |

---

## ⚖️ 5. Pros & Cons of GOAP Planning

### ✅ Advantages
* **Zero LLM Planning Cost**: A\* search runs entirely in Java — planning completes in microseconds vs 100ms–2s per step for LLM supervisors.
* **100% Deterministic & Auditable**: Same initial state and goals always produce the exact same plan. Fully reproducible.
* **Auto-Discovered Parallelism**: `PlanOptimizer` automatically groups independent actions into parallel layers without hand-coding DAGs.
* **Cost-Optimal Paths**: Admissible heuristic guarantees globally cheapest execution path.
* **Adaptive Replanning**: Can skip downstream steps at runtime based on deterministic metric thresholds (e.g. high Kendall's W).

### ❌ Trade-Offs & Limitations
* **Keyword-Based Goal Selection**: `GoapQueryAnalyzer` uses rule-based keyword matching — cannot understand novel natural language prompts.
* **Fixed Action Set**: All actions must be statically declared in `GoapActionRegistry`.
* **Exponential Search Growth**: A\* state-space search grows exponentially with larger action inventories (though lightweight for 8–15 actions).
* **No Learning / Feedback Loop**: The planner does not learn from past runs; same keywords always produce the same plan.

---

## 🔗 Related Resources
* [llm-council-explained.md](file:///home/user/llm-council-java-original/llm-council-explained.md) — Main LLM Council Architecture & 6 Planning Patterns Overview
* [htn-planning-explained.md](file:///home/user/llm-council-java-original/htn-planning-explained.md) — HTN Planning Pattern Deep Dive
* `planning_spec_drive-agents.pdf` — Specification slide deck by Dan Dobrin (Google Cloud)
