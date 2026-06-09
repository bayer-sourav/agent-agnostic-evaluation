# Agent-Agnostic Evaluation Pipeline - Deep Research Notes & Proposal

> **Goal:** Design an evaluation pipeline that can assess *any* AI agent (LangGraph, CrewAI, AutoGen, custom ReAct loops, OpenAI Assistants, vertical SaaS agents, etc.) using a **standardized, framework-independent** interface - producing insights useful to **engineers, researchers, product managers, and business stakeholders** alike.

---

## 1. Why "Agent-Agnostic" Matters

Today's agent ecosystem is fragmented:
- Frameworks differ wildly (LangGraph, CrewAI, AutoGen, LlamaIndex, Semantic Kernel, DSPy, custom).
- Each has its own tracing format, tool-calling schema, memory model, and event log.
- Evaluation is often baked into the framework (e.g., LangSmith ↔ LangChain), creating **vendor lock-in** and making **apples-to-apples comparison impossible**.
- Business users want outcomes (cost, success rate, ROI); engineers want traces (token usage, tool errors); researchers want benchmarks (reasoning quality, robustness).

An **agnostic pipeline** decouples *what* is being measured from *how* the agent is implemented.

---

## 2. Landscape — What's Out There

### 2.1 Open-source / Framework-tied evaluators
| Tool | Focus | Agnostic? |
|---|---|---|
| **LangSmith / LangChain Evals** | Tracing + LLM-as-judge | Partial (best with LC) |
| **Ragas** | RAG quality (faithfulness, context recall) | Yes for RAG |
| **DeepEval** | Pytest-style LLM/agent tests | Mostly yes |
| **TruLens** | Feedback functions, groundedness | Yes |
| **Phoenix (Arize)** | OTel-based tracing & eval | Yes (OpenTelemetry) |
| **OpenAI Evals** | Static benchmarks | Model-level |
| **Promptfoo** | Prompt regression testing | Yes |
| **Inspect AI (UK AISI)** | Safety-grade agent evals | Yes |
| **AgentBench / AgentBoard / τ-bench / SWE-bench / WebArena / GAIA / OSWorld** | Benchmark suites | Yes, but task-specific |
| **MLflow LLM Evaluate** | Lifecycle eval | Yes |
| **Galileo, Braintrust, Langfuse, Helicone, Patronus** | Commercial observability + eval | Mostly yes |

### 2.2 Standards emerging
- **OpenTelemetry GenAI semantic conventions** (spans for LLM calls, tool calls, agent steps).
- **OpenInference** (Arize) — instrumentation spec across frameworks.
- **MCP (Model Context Protocol)** — standardizing tool interfaces; useful boundary for evaluation.
- **AgentOps schema** for agent runs.

### 2.3 Research benchmarks worth tracking
- **τ-bench** (tool-use + user simulation)
- **GAIA** (general assistant)
- **SWE-bench / SWE-bench Verified** (software agents)
- **WebArena / VisualWebArena / WorkArena** (web agents)
- **OSWorld** (computer-use)
- **MLE-bench** (ML engineering)
- **AgentBench** (multi-environment)
- **HAL / HELM-Agent** (holistic)

**Gap:** No single pipeline ingests agents from arbitrary frameworks, normalizes their traces, runs both **offline benchmarks** and **online production evals**, and reports to **multiple audiences**.

---

## 3. Metrics — A Layered Taxonomy

### 3.1 Outcome / Task metrics (what the agent *achieved*)
- **Task success rate** (binary / graded)
- **Goal completion %** (sub-goal decomposition)
- **Exact match / F1 / BLEU / ROUGE** (for structured outputs)
- **Pass@k**, **Resolved@k** (for code / multi-attempt tasks)

### 3.2 Trajectory / Process metrics (how it got there)
- **Step count / efficiency** (vs. optimal trajectory)
- **Tool-call correctness** (right tool, right args)
- **Tool-call precision/recall** vs. ground-truth trajectory
- **Plan quality** (LLM-judge on intermediate plans)
- **Redundancy / loop detection**
- **Recovery from errors** (does it retry intelligently?)

### 3.3 Reasoning quality
- **Faithfulness** (output grounded in retrieved context / tool outputs)
- **Coherence / logical consistency**
- **Hallucination rate**
- **Self-consistency** across seeds

### 3.4 Safety & alignment
- **Refusal correctness** (appropriate refusals)
- **Prompt-injection resistance**
- **PII / secret leakage**
- **Toxicity / bias**
- **Jailbreak robustness**
- **Over-refusal rate**

### 3.5 Operational / system metrics
- **Latency** (p50, p95, p99 end-to-end and per step)
- **Token usage** (input/output/cached)
- **Cost per task** ($)
- **Throughput**
- **Tool error rate / API failure rate**
- **Context window utilization**

### 3.6 Business / product metrics
- **Cost per resolved ticket / per workflow**
- **Human handoff rate** / **deflection rate**
- **User satisfaction (CSAT, thumbs up/down)**
- **Time saved vs. baseline**
- **ROI** = (value generated − cost) / cost
- **SLA compliance**

### 3.7 Robustness / generalization
- **Performance under paraphrased inputs**
- **Adversarial perturbations**
- **Distribution shift** (held-out domains)
- **Long-horizon degradation**

---

## 4. How to Make It Truly Agent-Agnostic

### 4.1 Core architectural principle: **Adapt at the boundary, standardize in the middle**

```
[Any Agent Framework] → [Adapter] → [Canonical Trace Schema] → [Eval Engine] → [Reporting Layer]
```

### 4.2 Canonical Agent Run Schema (proposed)
Every agent run, regardless of framework, normalized to:

```json
{
  "run_id": "...",
  "agent_id": "...",
  "task": {"input": "...", "ground_truth": "...", "metadata": {}},
  "steps": [
    {
      "step_id": 0,
      "type": "thought | tool_call | tool_result | llm_call | message",
      "content": "...",
      "tool": {"name": "...", "args": {}, "result": "..."},
      "tokens": {"in": 0, "out": 0},
      "latency_ms": 0,
      "cost_usd": 0.0,
      "timestamp": "..."
    }
  ],
  "final_output": "...",
  "status": "success | failure | error",
  "metadata": {"framework": "langgraph", "model": "gpt-4o", ...}
}
```

Built on top of **OpenTelemetry GenAI conventions + OpenInference**, so existing instrumentation maps cleanly.

### 4.3 Adapter strategy
- **Pull adapters**: parse traces from LangSmith, Langfuse, Phoenix, AutoGen logs, CrewAI callbacks.
- **Push SDK**: thin client (`pipeline.log_step(...)`) for custom agents.
- **OTel collector**: zero-code instrumentation via standard exporters.
- **Black-box mode**: if no traces available, evaluate only `input → output` (degraded but works for any API agent).

### 4.4 Decoupling principles
1. **Interface over implementation** — evaluators operate on the canonical schema, never on framework objects.
2. **Plug-in metrics** — each metric is a function `f(run) → score`; can be code-based, LLM-judge, or human.
3. **Judge-agnostic** — LLM-judge metrics support multiple backends (GPT-4o, Claude, open models) for cost/bias control.
4. **Dataset-agnostic** — support static benchmarks, synthetic generation, replay of production traces, A/B comparisons.
5. **Environment-agnostic** — sandboxed task environments (web, code, RAG, tool-use) behind a common `Environment` interface.

---

## 5. Pipeline Architecture (Proposal)

```
┌───────────────────────────────────────────────────────────────┐
│ 1. INGESTION LAYER                                            │
│   - OTel/OpenInference collectors                             │
│   - Framework adapters (LangGraph, CrewAI, AutoGen, MCP, ...) │
│   - Push SDK (Python/TS)                                      │
│   - Black-box HTTP wrapper                                    │
└───────────────────────────────────────────────────────────────┘
                            │
                            ▼
┌───────────────────────────────────────────────────────────────┐
│ 2. NORMALIZATION LAYER                                        │
│   - Canonical Run Schema                                      │
│   - PII scrubbing, redaction                                  │
│   - Storage: object store (raw) + columnar DB (DuckDB/Parquet)│
└───────────────────────────────────────────────────────────────┘
                            │
                            ▼
┌───────────────────────────────────────────────────────────────┐
│ 3. EVALUATION ENGINE                                          │
│   - Metric registry (code, LLM-judge, human-in-loop)          │
│   - Dataset / benchmark registry                              │
│   - Task runners (offline batch + online streaming)           │
│   - Sandboxed environments (Docker, browser, REPL)            │
│   - Statistical layer (CI, significance tests, power)         │
└───────────────────────────────────────────────────────────────┘
                            │
                            ▼
┌───────────────────────────────────────────────────────────────┐
│ 4. ANALYSIS & REPORTING                                       │
│   - Per-audience dashboards                                   │
│   - Regression detection (vs. baseline / previous version)    │
│   - Failure clustering & root-cause                           │
│   - Exportable scorecards (JSON, PDF, Markdown)               │
└───────────────────────────────────────────────────────────────┘
                            │
                            ▼
┌───────────────────────────────────────────────────────────────┐
│ 5. FEEDBACK LOOP                                              │
│   - CI/CD hooks (block deploys on regressions)                │
│   - Production guardrails (online evals → alerts)             │
│   - Dataset curation from failures                            │
└───────────────────────────────────────────────────────────────┘
```

---

## 6. Multi-Audience Reporting

### 6.1 For Business / Executives
- **One-page scorecard**: Success rate, cost/task, deflection rate, ROI, trend vs. last week.
- **Risk meter**: % of unsafe outputs, compliance flags.
- **Comparative view**: Agent v1 vs. v2 vs. competitor, $/successful outcome.
- Language: outcomes, money, risk — no token counts.

### 6.2 For Product Managers
- **Funnel view**: Inputs → tasks attempted → completed → satisfied users.
- **Cohort analysis** by use-case, user segment.
- **Feature impact**: How did adding a new tool affect success/cost?
- **CSAT correlation** with internal metrics.

### 6.3 For Engineers / ML
- **Trace explorer** with step-level metrics.
- **Failure mode clustering** (e.g., "tool X returned timeout 23% of failures").
- **Token & latency flamegraphs**.
- **Regression diff** between runs/commits.
- **Replay & debug** single traces.

### 6.4 For Researchers
- **Benchmark leaderboards** with confidence intervals.
- **Ablation tables** (model × prompt × tool set).
- **Reproducibility bundle**: dataset hash + seed + container.
- **Robustness curves** (perturbation strength vs. score).

### 6.5 For Compliance / Risk
- **Audit log** of every decision + safety-eval results.
- **Policy-violation reports**.
- **Data-lineage** trace (what data influenced what output).

---

## 7. Design Principles (Checklist)

- ✅ **Framework-agnostic** via adapters + canonical schema
- ✅ **Model-agnostic** judges (avoid GPT-only bias; rotate judges)
- ✅ **Deterministic where possible** (seeded, sandboxed envs, fixed datasets)
- ✅ **Composable metrics** (mix code, LLM-judge, human)
- ✅ **Offline + online parity** (same metrics in CI and prod)
- ✅ **Statistically honest** (CIs, multiple-runs, significance)
- ✅ **Cost-aware** (cache LLM-judge calls; sample intelligently)
- ✅ **Privacy-first** (redaction at ingest; on-prem option)
- ✅ **Open standards** (OTel, OpenInference, MCP)
- ✅ **Extensible** (plug-in SDK for new metrics, datasets, envs)

---

## 8. Key Challenges & Mitigations

| Challenge | Mitigation |
|---|---|
| LLM-judge bias & variance | Multi-judge ensembles, calibration against human labels, position/length bias controls |
| Non-determinism of agents | Multiple seeds, report distributions not point estimates |
| Long-horizon / multi-turn evals | Hierarchical scoring (sub-goal + final), user simulators (τ-bench style) |
| Trajectory ground-truth scarcity | Use process-reward models, pairwise preference, or outcome-only |
| Benchmark contamination | Private hold-outs, dynamic test generation, perturbation suites |
| Cost of running large evals | Smart sampling, importance weighting, cached judges |
| Cross-framework semantic drift | Strict schema validation + conformance tests for adapters |
| Online ↔ offline gap | Shadow-mode prod replays into offline eval |

---

## 9. Proposed Phased Roadmap

**Phase 0 — Foundations**
- Define canonical Run Schema (OTel-aligned).
- Build 2 reference adapters (LangGraph + black-box HTTP).
- Storage: Parquet + DuckDB.

**Phase 1 — Core Metrics**
- Implement metric registry: success, tool-call F1, latency, cost, faithfulness, safety basics.
- LLM-judge framework with multi-backend support.
- CLI + Python SDK.

**Phase 2 — Benchmarks & Environments**
- Integrate τ-bench, GAIA, SWE-bench-lite, a RAG benchmark.
- Sandboxed environments (Docker, browser via Playwright).
- Statistical reporting (CIs, paired tests).

**Phase 3 — Reporting Layer**
- Audience-specific dashboards (Business / Eng / Research).
- Regression detection + CI integration.
- Exportable scorecards.

**Phase 4 — Production Online Evals**
- Streaming eval workers.
- Guardrails + alerting.
- Failure clustering & dataset curation loop.

**Phase 5 — Ecosystem**
- More adapters (CrewAI, AutoGen, Semantic Kernel, MCP).
- Community metric/dataset plug-ins.
- Reproducibility bundles & public leaderboards.

---

## 10. Success Criteria for the Pipeline Itself

- **Coverage**: works with ≥ 5 major frameworks out-of-the-box.
- **Latency**: offline eval of 1k runs in < 10 min on commodity hardware.
- **Judge cost**: < $0.01 per evaluated trajectory at median.
- **Reproducibility**: any reported number reproducible from stored bundle.
- **Adoption proxy**: 3 different audience personas using the same data with zero re-instrumentation.

---

## 11. Open Questions to Explore Next

1. Should we adopt OpenInference *as-is* or extend it for agent-specific concepts (plans, sub-agents, memory ops)?
2. Best way to handle **multi-agent / hierarchical** runs in the schema (nested vs. flat with parent_id)?
3. How to standardize **user simulators** for online-like offline evals?
4. Can we build a **process reward model** trained on aggregated trajectories for cheap intermediate scoring?
5. Governance: who curates the public benchmark + metric registry?

---

## 12. TL;DR

Build a pipeline that:
1. **Ingests** traces from any agent via adapters + OTel.
2. **Normalizes** them into a canonical schema.
3. **Evaluates** with a pluggable, multi-audience metric library covering outcome, process, safety, cost, and business KPIs.
4. **Reports** through audience-tailored views — same source of truth, different lenses.
5. **Closes the loop** with CI gates and production guardrails.

The differentiator is not a new metric — it's **interoperability + multi-audience usefulness + statistical honesty** in one place.