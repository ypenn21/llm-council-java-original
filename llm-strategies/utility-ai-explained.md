# 🔄 Utility AI Pattern in LLM Council

This document provides a detailed technical guide to the **Utility AI** planning and orchestration pattern for AI agents, referencing [llm-council-explained.md](file:///home/user/llm-council-java-original/llm-council-explained.md) and `planning_spec_drive-agents.pdf`.

---

## 📌 1. What is Utility AI Planning?

**Utility AI** operates on the core principle: *"No plan survives first contact with live data."*

Instead of generating a fixed plan up front (like HTN or GOAP), Utility AI evaluates every available action dynamically at runtime during each decision tick using an **OODA (Observe-Orient-Decide-Act) Execution Loop**.

### Key Concept
Execution order and concurrency are not hand-coded or pre-calculated — **parallelism emerges naturally** as a mathematical result of action scoring.

---

## ⚙️ 2. The OODA Execution Loop

Every decision tick, the Utility AI engine runs through the 4-phase OODA loop:

```
                  ┌──────────────────────────────────────────────┐
                  │                 OODA LOOP                    │
                  └──────────────────────┬───────────────────────┘
                                         │
                 ┌───────────────────────┴───────────────────────┐
                 ▼                                               ▼
         ┌───────────────┐                               ┌───────────────┐
         │  1. OBSERVE   │                               │   2. ORIENT   │
         │               │                               │               │
         │ Filter        │                               │ Score Candidates│
         │ Eligible      ├──────────────────────────────►│ via           │
         │ Actions       │                               │ CompositeScorer│
         └───────────────┘                               └───────┬───────┘
                                                                 │
                 ┌───────────────────────────────────────────────┘
                 ▼
         ┌───────────────┐                               ┌───────────────┐
         │   3. DECIDE   │                               │    4. ACT     │
         │               │                               │               │
         │ Select        │                               │ Execute in    │
         │ Candidates    ├──────────────────────────────►│ Parallel      │
         │ Score >= 0.5  │                               │ Flux.merge()  │
         └───────────────┘                               └───────┬───────┘
                 ▲                                               │
                 └───────────────────────────────────────────────┘
                   Recurse until NIRVANA Event (All work complete)
```

### The 4 OODA Phases Explained:

1. **OBSERVE**:
   * Inspects the 11 variables in `worldState`.
   * Filters actions that meet 3 eligibility checks:
     * Preconditions are satisfied?
     * Output effects are not yet available?
     * Action is not already in-progress?

2. **ORIENT**:
   * Evaluates each eligible action using a `CompositeScorer` across **4 weighted scoring dimensions**:
     * **Readiness**: How many downstream actions will executing this action unblock?
     * **Progress**: Weighting based on pipeline depth towards the final goal.
     * **Latency**: Estimated execution speed and time cost.
     * **Information Gain**: Contribution to final answer quality and confidence.

3. **DECIDE**:
   * Selects all actions with utility scores $\ge$ **Parallel Threshold (0.5)**.
   * **Emergent Dynamic Parallelism**: If 3 actions score $\ge 0.5$, all 3 are selected for concurrent execution!
   * Fallback: If no action scores $\ge 0.5$, picks the single highest-scoring action to guarantee forward progress.

4. **ACT**:
   * Dispatches selected actions concurrently via `Flux.merge()`.
   * Marks corresponding effects as `in-progress`.
   * Upon action completion, marks state variables as `available` and loops back to **OBSERVE**.
   * **Terminal Event**: Triggers a `NIRVANA` event when no eligible actions remain and nothing is in-progress.

---

## 🏛️ 3. Utility AI System Components in LLM Council

```
                               ┌─────────────────────────┐
                               │       World State       │ (11 State Variables Tracked)
                               └────────────┬────────────┘
                                            │
                                            ▼
                               ┌─────────────────────────┐
                               │      Action System      │ (9 Pre-authored Actions)
                               └────────────┬────────────┘
                                            │
                                            ▼
                               ┌─────────────────────────┐
                               │     Scoring Engine      │ (4 Weighted Scorers)
                               └────────────┬────────────┘
                                            │
               ┌────────────────────────────┼────────────────────────────┐
               ▼                            ▼                            ▼
       [ Balanced Preset ]      [ Latency-Optimized ]       [ Info-Gain Preset ]
       Equal Scorers Weight     Favors Speed (3-Stage)      Favors Quality (5-Stage)
```

### System Specs:
* **11 World State Variables**: Binary tracking of queries, initial responses, rankings, agreements, disagreements, final synthesis, and aggregate metrics.
* **9 Concrete Actions**: Each declares preconditions, effects, cost (0.05–0.9), and value (0.5–1.0).
* **3 Adaptive Profiles**:
  1. *Balanced Profile*: Equal weighting across readiness, progress, latency, and info gain.
  2. *Latency-Optimized Profile*: Heavily weights latency, causing the engine to skip agreement/disagreement analysis and jump straight to synthesis.
  3. *Information-Gain Profile*: Heavily weights quality, forcing full 5-stage execution before synthesis.

---

## 📊 4. Stage-by-Stage Breakdown for Utility AI Planning

| Phase / Stage | Executing Engine | Inputs Received | Preconditions & Scoring Dimensions | Purpose |
| :--- | :--- | :--- | :--- | :--- |
| **0. Observe Phase** | OODA Evaluator | 11 World-State Variables | **Check**: Preconditions met AND effects missing AND not in-progress. | Filter eligible candidate actions for the current tick. |
| **0. Orient Phase** | CompositeScorer | Eligible Actions + World State | **Scoring Dimensions**: *Readiness*, *Progress*, *Latency*, *Information Gain*. | Calculate scalar utility score (0.0 to 1.0) for each candidate in $O(1)$ time (0 LLM calls). |
| **0. Decide Phase** | Decision Engine | Candidate Utility Scores | **Threshold**: $\ge 0.5$ for parallel dispatch. | Select actions for execution; enables dynamic concurrent execution when multiple actions qualify. |
| **Stage 1: Response Generation** | Action Executor | `{QUERY_AVAILABLE}` | **Action**: `GenerateResponses`<br>**Score**: High readiness & progress. | Dispatch initial parallel model responses. |
| **Stage 2: Peer Evaluation & Analysis** | Action Executor (`Flux.merge`) | `{RESPONSES_AVAILABLE}` | **Actions**: `EvaluateRankings`, `AnalyzeAgreements`, `AnalyzeDisagreements`<br>**Score**: All score $\ge 0.5$ simultaneously. | **Emergent Parallelism**: Dispatches Stage 2, 3, and 4 concurrently in a single decision tick. |
| **Java Aggregations** | Action Executor | Stage Outputs | **Actions**: `AggregateRankings`, `AggregateAgreements`, `AggregateDisagreements` | Perform pure-computation Java data aggregations as inputs complete. |
| **Stage 5: Final Synthesis** | Action Executor | Aggregate State | **Action**: `SynthesizeFinal`<br>**Score**: Highest progress weight when aggregates exist. | Generate final synthesized answer. |
| **Terminal Check** | OODA Evaluator | Completed World State | **Condition**: Zero eligible actions remain. | Emit `NIRVANA` event and finish deliberation. |

---

## ⚖️ 5. Pros & Cons of Utility AI Planning

### ✅ Advantages
* **Emergent Dynamic Parallelism**: Concurrency is not manually wired — actions execute in parallel whenever multiple candidates score above threshold.
* **Lightweight Decisions**: Scorer evaluations are $O(1)$ per action in Java (0 LLM planning calls).
* **Implicit Dependency Handling**: Preconditions and effects implicitly define the execution graph without a centralized orchestrator.
* **Adaptive Strategy Swapping**: Swap between *Balanced*, *Latency-Optimized*, and *Information-Gain* behavior simply by changing config weights.
* **Full Observability**: Event stream traces every scoring decision, making reasoning transparent and debuggable.

### ❌ Trade-Offs & Limitations
* **Scorer Weight Sensitivity**: Weights require manual calibration; poorly tuned weights can serialize parallel tasks or skip important stages.
* **Fixed Action Set**: Actions are predefined at compile time — no runtime action discovery or generation.
* **Threshold Sensitivity**: The $0.5$ parallel threshold dictates concurrency — setting it too low over-parallelizes API calls, while setting it too high serializes execution.
* **No Backtracking**: Actions are fire-and-forget — failed actions clear state but cannot undo downstream side-effects.

---

## 🔗 Related Resources
* [llm-council-explained.md](file:///home/user/llm-council-java-original/llm-council-explained.md) — Main LLM Council Architecture & 6 Planning Patterns Overview
* [htn-planning-explained.md](file:///home/user/llm-council-java-original/htn-planning-explained.md) — HTN Planning Pattern Deep Dive
* [goap-planning-explained.md](file:///home/user/llm-council-java-original/goap-planning-explained.md) — GOAP Planning Pattern Deep Dive
* `planning_spec_drive-agents.pdf` — Specification slide deck by Dan Dobrin (Google Cloud)
