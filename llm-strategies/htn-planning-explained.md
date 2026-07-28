# 🌲 HTN (Hierarchical Task Network) Planning in LLM Council

This document provides an in-depth guide to the **Hierarchical Task Network (HTN)** planning pattern for AI agents, referencing the concepts and architecture from [llm-council-explained.md](file:///home/user/llm-council-java-original/llm-council-explained.md) and `planning_spec_drive-agents.pdf`.

---

## 📌 1. What is HTN Planning?

**Hierarchical Task Network (HTN)** is a classical AI planning paradigm based on **top-down recursive decomposition**. 

Instead of letting an LLM arbitrarily guess what to do at every step (like a Supervisor) or using a rigid fixed pipeline (like a basic Workflow), HTN treats domain expertise as a collection of **recipes (methods)**.

### Key Concept
* **Compound Tasks**: High-level abstract goals (e.g. `COUNCIL_DELIBERATION`).
* **Primitive Tasks**: Atomic, executable agent steps (e.g. `RunStage1_InitialResponses`, `RunStage2_PeerRanking`, `RunStage5_FinalSynthesis`).
* **Methods**: Pre-authored decomposition rules with preconditions and priority scores that break compound tasks into ordered subtasks.

---

## ⚙️ 2. The HTN Decomposition Algorithm

The HTN planner operates deterministically in pure Java via the `DECOMPOSE` algorithm:

```
                            ┌─────────────────────────────────┐
                            │   DECOMPOSE( task, worldState ) │
                            └────────────────┬────────────────┘
                                             │
                                   Identify Task Type
                                    ┌────────┴────────┐
                                    ▼                 ▼
                          [ PRIMITIVE TASK ]   [ COMPOUND TASK ]
                                    │                 │
                         1. Verify Preconditions    1. Select Method (Priority Order)
                                    │                 │
                         2. Execute & Append        2. Decompose into Subtasks
                                    │                 │
                         3. Update World State      3. Recurse on Each Subtask
                                    │                 │
                                    └────────┬────────┘
                                             ▼
                             Return Ordered Primitive Plan
```

### Algorithm Steps:
1. **Identify Task Type**:
   * **Primitive Task**:
     1. Verify preconditions against current `worldState`.
     2. Append primitive task to the output plan sequence.
     3. Apply task effects to update `worldState` for subsequent tasks.
   * **Compound Task**:
     1. **Method Selection**: Try available methods in priority order. The first method whose precondition holds in `worldState` is selected (*greedy selection, no backtracking*).
     2. **Decomposition**: The selected method expands the compound task into an ordered list of subtasks (which may be primitive or compound).
     3. **Recursion**: Recursively invoke `DECOMPOSE` on each subtask, updating state along the way.
2. **Result**: An ordered execution plan consisting entirely of primitive tasks.

---

## 🏛️ 3. HTN Architecture in LLM Council

In the context of the **LLM Council** deliberation engine, HTN provides **adaptive pipelines**: simple factual queries use a fast 3-stage pipeline, while complex debate queries trigger the full 5-stage pipeline.

```
                               ┌─────────────────────────┐
                               │   User Instruction/Query│
                               └────────────┬────────────┘
                                            │
                                            ▼
                               ┌─────────────────────────┐
                               │  Query Classifier LLM   │ (1 Fast LLM Call)
                               └────────────┬────────────┘
                                            │ Category (e.g. Factual vs Debate)
                                            ▼
                               ┌─────────────────────────┐
                               │       HTN Planner       │ (Pure Java, 0 LLM Calls)
                               │  COUNCIL_DELIBERATION   │
                               └────────────┬────────────┘
                                            │
               ┌────────────────────────────┼────────────────────────────┐
               ▼                            ▼                            ▼
       [ QuickFactual ]             [ ConsensusOnly ]             [ FullDelib ]
         (Priority 10)                (Priority 20)               (Priority 100)
               │                            │                            │
               ▼                            ▼                            ▼
    Primitive Plan (3 Stages)    Primitive Plan (4 Stages)    Primitive Plan (5 Stages)
    - Stage 1: Responses         - Stage 1: Responses         - Stage 1: Responses
    - Stage 2: Peer Ranking      - Stage 2: Peer Ranking      - Stage 2: Peer Ranking
    - Stage 5: Synthesis         - Stage 3: Agreement         - Stage 3: Agreement
                                 - Stage 5: Synthesis         - Stage 4: Disagreement
                                                              - Stage 5: Synthesis
```

### Core Components:

1. **Query Classifier (1 LLM Call)**:
   * Classifies user queries into 6 categories: *Factual*, *Informational*, *Opinion*, *Debate*, *Analytical*, *Creative*.
   * Cost: Exactly 1 fast LLM call (e.g., using Gemini Flash Lite).

2. **HTN Methods (Spring `@Component` Beans)**:
   * **`QuickFactualMethod` (Priority: 10)**:
     * *Precondition*: Query category is `FACTUAL` or `INFORMATIONAL`.
     * *Decomposition*: Expands `COUNCIL_DELIBERATION` $\rightarrow$ `[Stage 1, Stage 2, Stage 5]`.
   * **`ConsensusOnlyMethod` (Priority: 20)**:
     * *Precondition*: Query category is `OPINION` or `CREATIVE`.
     * *Decomposition*: Expands `COUNCIL_DELIBERATION` $\rightarrow$ `[Stage 1, Stage 2, Stage 3, Stage 5]`.
   * **`FullDelibMethod` (Priority: 100 - Default Fallback)**:
     * *Precondition*: Query category is `DEBATE` or `ANALYTICAL` (or wildcard fallback).
     * *Decomposition*: Expands `COUNCIL_DELIBERATION` $\rightarrow$ `[Stage 1, Stage 2, Stage 3, Stage 4, Stage 5]`.

3. **HTN Executor**:
   * Sequentially dispatches primitive task stages via reactive streams (`concatMap`). Each stage completes and updates session state before the next begins.

---

## 📊 4. Stage-by-Stage Breakdown for HTN Planning

| Stage | Executing Agent | Input Received | Purpose |
| :--- | :--- | :--- | :--- |
| **0. Query Classification Stage** | Query Classifier LLM | User Query | Categorize query type (*Factual*, *Debate*, etc.) in 1 fast LLM call. |
| **0. HTN Decomposition Stage** | HTN Planner (Java) | Query Category + World State | Select matching `HtnMethod` by priority and decompose compound task into primitive tasks. |
| **Stage 1: Initial Responses** | All Council Models | User Query | Execute primitive initial response generation in parallel. |
| **Stage 2: Peer Ranking** | All Council Models | Anonymized Responses A, B, C... | Execute primitive blind peer review & Kendall's W calculation. |
| **Stage 3: Agreement Analysis** | All Council Models | Anonymized Responses | **Dynamically Included or Skipped**: Included for *Opinion*, *Debate*, *Analytical*; skipped for *Factual*. |
| **Stage 4: Disagreement Analysis** | All Council Models | Anonymized Responses | **Dynamically Included or Skipped**: Included ONLY for *Debate* and *Analytical* queries. |
| **Stage 5: Final Synthesis** | Chairman Model | Accumulated Primitive Outputs | Synthesize final authoritative response based on executed stages. |

---

## ⚖️ 5. Pros & Cons of HTN Planning

### ✅ Advantages
* **Adaptive Pipelines**: Queries automatically get tailored execution paths (3, 4, or 5 stages) based on query complexity.
* **Pure-Addition Extensibility**: Adding a new orchestration strategy is as simple as creating a new `@Component` class implementing `HtnMethod`. Zero modifications to existing code.
* **Compile-Time Safety**: Sealed Java interfaces and pattern matching guarantee exhaustive handling of all task types at compile time.
* **Graceful Degradation**: If classification fails or returns low confidence, defaults to `FullDelibMethod` (full 5-stage pipeline) as a safe fallback.
* **Ultra-Low Planning Overhead**: Planning is pure Java data structure manipulation — only 1 fast LLM call is required for query classification.

### ❌ Trade-Offs & Limitations
* **No Backtracking**: Greedy first-match selection means if a method produces a suboptimal plan, the planner cannot revise it mid-execution.
* **Sequential Execution Only**: Primitive tasks run sequentially via `concatMap` (no parallel execution of independent compound stages).
* **Classification Dependency**: Pipeline quality depends entirely on the accuracy of the query classifier LLM.
* **Fixed Method Set**: Methods are defined at compile time via Spring beans — no dynamic method generation at runtime.

---

## 🔗 Related Resources
* [llm-council-explained.md](file:///home/user/llm-council-java-original/llm-council-explained.md) — Main LLM Council Architecture & 6 Planning Patterns Overview
* `planning_spec_drive-agents.pdf` — Specification slide deck by Dan Dobrin (Google Cloud)
* [CouncilService.java](file:///home/user/llm-council-java-original/src/main/java/dev/council/service/CouncilService.java) — Core execution engine
