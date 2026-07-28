# 🧭 Agentic Planning Strategies in LLM Council

This document serves as the master summary index for all 6 agentic planning strategies and cost/accuracy tradeoffs in the [llm-strategies/](file:///home/user/llm-council-java-original/llm-strategies) directory. It highlights the key differentiators of each pattern and explains why specific strategies were chosen for different production scenarios.

---

## 🗺️ Master Comparison Matrix

| Strategy | Planning LLM Cost | Determinism | Accuracy | Key Differentiator | Primary Selection Reason | Deep Dive Link |
| :--- | :---: | :---: | :---: | :--- | :--- | :--- |
| **Workflow Pattern** *(Repository Default)* | **$0.00** | **100%** | **10 / 10** | Compile-time fixed Java Reactor DAGs | **Guarantees 100% deliberation coverage** with zero planner risk | [llm-council-explained.md](file:///home/user/llm-council-java-original/llm-strategies/llm-council-explained.md) |
| **HTN** *(Hierarchical Task Network)* | **~$0.0001** | **80%** | **8.0 / 10** | 1 fast classifier LLM routes queries into pre-authored task recipes | **Saves ~40% tokens on simple queries** by skipping unneeded stages | [htn-planning-explained.md](file:///home/user/llm-council-java-original/llm-strategies/htn-planning-explained.md) |
| **GOAP** *(Goal-Oriented Action Planning)* | **$0.00** | **100%** | **9.5 / 10** | Microsecond A\* graph search + adaptive Kendall's W pruning | **#1 Overall Cost-Optimized Strategy** (25%–30% token savings) | [goap-planning-explained.md](file:///home/user/llm-council-java-original/llm-strategies/goap-planning-explained.md) |
| **Utility AI** *(OODA Execution Loop)* | **$0.00** | **70%** | **7.5 / 10** | Action scoring via $O(1)$ math functions across 4 weights | **Emergent runtime parallelism** tailored by config profiles | [utility-ai-explained.md](file:///home/user/llm-council-java-original/llm-strategies/utility-ai-explained.md) |
| **LLM Compiler** | **~$0.0005** | **85%** | **8.5 / 10** | 1 upfront JSON DAG compilation + static Java validation | **Flexible DAG creation** without heavy per-step LLM overhead | [llm-compiler-explained.md](file:///home/user/llm-council-java-original/llm-strategies/llm-compiler-explained.md) |
| **SupervisorLLM** | **~$0.0050+** | **30%** | **6.0–8.0** | Step-by-step LLM decision loop bounded by 3-layer safety net | **Maximum flexibility** for unpredictable, open-ended prompts | [supervisor-llm-explained.md](file:///home/user/llm-council-java-original/llm-strategies/supervisor-llm-explained.md) |

---

## 📑 Strategy-by-Strategy Executive Summaries

### 1. Workflow Pattern (Compile-Time Fixed DAGs — Repository Default)
* **Summary**: The execution graph is hard-coded at compile time using Project Reactor (`Flux`/`Mono`) in `CouncilService.java`.
* **Key Differentiator**: 100% deterministic bytecode execution with $0.00 planning token overhead.
* **Why We Chose It as Default**:
  It guarantees 100% 5-stage deliberation accuracy, zero runtime planner deviation, and maximum predictability. It is ideal for high-stakes decision-making where skipping evaluation steps is unacceptable.
* 📖 **Deep Dive**: [llm-council-explained.md](file:///home/user/llm-council-java-original/llm-strategies/llm-council-explained.md)

---

### 2. HTN (Hierarchical Task Network)
* **Summary**: Uses 1 fast classifier LLM call to categorize query intent (*Factual*, *Opinion*, *Debate*, *Analytical*) and routes it to pre-authored Spring `@Component` method recipes (`QuickFactual`, `ConsensusOnly`, `FullDelib`).
* **Key Differentiator**: Top-down recursive decomposition tailored dynamically to query complexity.
* **Why Choose It Over Others**:
  Ideal for high-volume mixed traffic where ~50% of prompts are simple factual questions. Bypassing Stages 3 & 4 on factual queries saves ~40% of tokens while maintaining 100% accuracy for complex debate queries.
* 📖 **Deep Dive**: [htn-planning-explained.md](file:///home/user/llm-council-java-original/llm-strategies/htn-planning-explained.md)

---

### 3. GOAP (Goal-Oriented Action Planning)
* **Summary**: Microsecond A\* graph search over a 10-variable world state combined with dependency-depth parallel leveling (`PlanOptimizer`).
* **Key Differentiator**: $0.00 planning cost + adaptive runtime pruning (automatically skipping Stage 4 when Kendall's W consensus exceeds $0.8$).
* **Why Choose It Over Others**:
  Chosen as the **#1 overall cost-optimized strategy**. It combines 100% deterministic microsecond search with intelligent token pruning, reducing total deliberation token consumption by 25%–30% without sacrificing accuracy.
* 📖 **Deep Dive**: [goap-planning-explained.md](file:///home/user/llm-council-java-original/llm-strategies/goap-planning-explained.md)

---

### 4. Utility AI (OODA Execution Loop)
* **Summary**: Operates a continuous Observe-Orient-Decide-Act (OODA) loop every decision tick, scoring candidate actions using $O(1)$ math functions across 4 weighted dimensions (*Readiness, Progress, Latency, Info Gain*).
* **Key Differentiator**: Concurrency and execution order **emerge dynamically** at runtime rather than being pre-scripted.
* **Why Choose It Over Others**:
  Best when pipeline behavior needs to be dynamically re-tuned (e.g. switching between *Latency-Optimized* and *Information-Gain* profiles) via simple configuration file updates rather than code changes.
* 📖 **Deep Dive**: [utility-ai-explained.md](file:///home/user/llm-council-java-original/llm-strategies/utility-ai-explained.md)

---

### 5. LLM Compiler Pattern
* **Summary**: A single planning LLM call emits a complete JSON task DAG upfront (`{label, tasks[{id, agent, dependsOn}]}`), which is validated by a static Java guardrail (`PlanValidator`) before execution.
* **Key Differentiator**: Single upfront compilation call + GOAP A\* fallback circuit breaker if JSON validation fails.
* **Why Choose It Over Others**:
  Best when flexible natural language instructions must be compiled into custom execution DAGs without paying the heavy token and latency penalty of per-step supervisor loops.
* 📖 **Deep Dive**: [llm-compiler-explained.md](file:///home/user/llm-council-java-original/llm-strategies/llm-compiler-explained.md)

---

### 6. SupervisorLLM Pattern
* **Summary**: An LLM supervisor operates in an iterative step-by-step decision loop, bounded by 3-layer deterministic safety net guardrails and a 10-iteration hard cap.
* **Key Differentiator**: Maximum flexibility and adaptability to open-ended, non-standard user prompts.
* **Why Choose It Over Others**:
  Chosen when handling highly unpredictable user requests that cannot fit into pre-authored DAGs or fixed A\* action tables.
* 📖 **Deep Dive**: [supervisor-llm-explained.md](file:///home/user/llm-council-java-original/llm-strategies/supervisor-llm-explained.md)

---

## 📊 Cost, Accuracy & Single-Model Benchmark Summary
* **Cost Efficiency Analysis**: Ranks all 6 strategies by token spend and planning overhead.
* **Accuracy vs. Determinism**: Evaluates zero-variance bytecode workflows against dynamic LLM planners.
* **LLM Council vs. Single-Threaded Single-Model Agent**: Complete tradeoff analysis highlighting why multi-model deliberation eliminates single-model hallucinations and vendor lock-in.
* 📖 **Deep Dive**: [cost-optimization.md](file:///home/user/llm-council-java-original/llm-strategies/cost-optimization.md)
