# LLM Evaluation Frameworks — Platform Comparison

**Date**: 2026-03-01 | **Verified**: Langfuse v2.x, Phoenix v5.x, DeepEval v2.x, Datadog LLM Obs (Feb 2026)
**Source**: [appendix-deep-dives.md](../changelog/2.28/appendix-deep-dives.md) A3

---

## Landscape

The LLM evaluation ecosystem has converged on three patterns:

1. **OTel-native instrumentation** — Langfuse and Phoenix lead; DeepEval remains Python-native
2. **LLM-as-Judge** — every major platform supports it; differentiation is in metric libraries and aggregation
3. **CI/CD integration** — evaluation moving from notebooks to pipelines; DeepEval has the strongest pytest integration

---

## Platform Profiles

### Langfuse
Open-source (MIT), self-host via Docker or Langfuse Cloud, PostgreSQL backend. Strongest for **human-in-the-loop**: immutable auto-versioning (Dec 2025), annotation queues, trace-linked scores via native OTLP ingest. LLM-as-Judge uses custom prompt templates rather than a pre-built metric library.

### Arize Phoenix
Open-source (EL2), local or Phoenix Cloud. Deepest OTel integration of any platform — `openinference` instrumentation, OTLP protobuf + JSON. Auto-generated flowcharts for agent visualization. Explicit FK-based dataset versioning. No annotation queues; CI/CD is SDK + notebooks rather than CLI.

### DeepEval
Python library + Confident AI cloud dashboard. Not OTel-native (custom telemetry, bridged via `obs_export_confident`). **Largest metric library** (14+ including G-Eval, Faithfulness, Hallucination, Task Completion, Tool Correctness). Strongest CI/CD integration via `deepeval test run` / `@deepeval.assert_test`. No formal dataset versioning.

### Datadog LLM Observability
Managed SaaS, APM integration via Datadog Agent. Best for **enterprise APM correlation** — evaluations alongside infrastructure metrics, execution flow charts, alerting. Two-phase export: LLM Obs spans (`/v1/trace/spans`) + eval metrics (`/v2/eval-metric`). No self-host; no dedicated dataset management.

---

## Feature Matrix

| Feature | Langfuse | Phoenix | DeepEval | Datadog | Toolkit |
|---------|----------|---------|----------|---------|---------|
| OTel-native ingest | Yes (JSON) | Yes (JSON + protobuf) | No | Via DD Agent | OTLP export (4 targets) |
| Trace-linked evals | Yes | Yes | No | Yes (APM) | Yes (`traceId`) |
| Dataset versioning | Immutable | Explicit FK | No | No | Yes (G4) |
| Annotation queues | Yes | No | No | No | No |
| LLM-as-Judge | Custom prompts | Built-in evaluators | 14+ metrics | Managed | G-Eval, QAG |
| Agent-as-Judge | No | No | Partial | No | **Yes** (Procedural, Reactive) |
| CI/CD | CLI + GitHub Actions | SDK (notebooks) | pytest + CLI | CI monitors | Pipeline scripts |
| Agent visualization | Planned | Auto-flowcharts | No | Flow charts | Planned (G5) |
| Self-host | Yes (Docker) | Yes (pip) | Partial | No | Yes (local JSONL) |
| Budget-managed eval | No | No | No | No | **Yes** |
| Multi-agent eval | No | No | No | No | **Yes** |

---

## Toolkit Integration

All four export tools map `EvaluationResult` to platform-specific formats, sharing a GenAI semconv core:

| Toolkit Field | OTel Attribute | Notes |
|---------------|----------------|-------|
| `evaluationName` | `gen_ai.evaluation.name` | All platforms |
| `scoreValue` | `gen_ai.evaluation.score.value` | All platforms |
| `scoreLabel` | `gen_ai.evaluation.score.label` | All platforms |
| `explanation` | `gen_ai.evaluation.explanation` | All platforms |
| `sessionId` | `session.id` | Datadog: `session.id` meta; Confident AI: `confident.trace.thread_id` |

**`obs_export_langfuse`** (`src/lib/exports/langfuse-export.ts`) — groups evals by `traceId`; each evaluation becomes a `gen_ai.evaluation.result` span event; HTTP Basic auth.

**`obs_export_phoenix`** (`src/lib/exports/phoenix-export.ts`) — dual JSON/protobuf format via `encodeTracesProtobuf()`; URL validation distinguishes Phoenix Cloud vs. localhost vs. other.

**`obs_export_datadog`** (`src/lib/exports/datadog-export.ts`) — two-phase export (spans then eval metrics); automatic metric type inference (boolean/score/categorical); 5 hardcoded Datadog sites.

**`obs_export_confident`** (`src/lib/exports/confident-export.ts`) — Confident AI-specific attributes (`confident.span.*`, `confident.trace.*`); thread tracking via `sessionId`.

---

## Toolkit Differentiators

Features not available in any comparison platform:

- **Agent-as-Judge** (`src/lib/agent-judge/`) — ProceduralJudge (fixed pipeline, early termination) and ReactiveJudge (adaptive routing, LRU state). Tool verification with weighted scoring (selection 40%, args 30%, result 30%). Trajectory efficiency analysis and redundancy detection.
- **Multi-agent handoff scoring** (`computeMultiAgentEvaluation()`) — quantified inter-agent handoff quality across turns.
- **Budget-managed evaluation** (`quality-budget.ts`, `quality-sampler.ts`) — cost-aware scheduling; high-priority evaluations always run, lower-priority sampled against remaining budget.
- **Evaluation resilience** — `JudgeCircuitBreaker`, `evaluateWithRetry()` (60s cap, exponential backoff), `runCanaryEvaluations()`.
- **Position bias mitigation** — `mitigatedPairwiseEval()` with automated A/B order swapping.

---

## Deployment Patterns

| Mode | Toolkit | Platforms |
|------|---------|-----------|
| **Batch** (post-hoc) | `derive-evaluations.ts` → `judge-evaluations.ts`, cron `0 6,18 * * *` | All platforms support batch |
| **Online** (per-request) | T1 rule-based every request; T2 LLM judge budget-sampled | Langfuse (webhooks), Phoenix (experimental), Datadog (managed) |
| **CI/CD** (pre-deploy) | Pipeline scripts with exit codes | DeepEval strongest; Langfuse CLI; Datadog monitors |

---

## Decision Framework

| Priority | Consider |
|----------|----------|
| Open-source, self-hosted | Langfuse or Phoenix |
| Largest metric library + CI/CD | DeepEval |
| Enterprise APM integration | Datadog |
| Deepest OTel pipeline | Phoenix |
| Human-in-the-loop review | Langfuse |
| Agent-specific evaluation | Toolkit + any export target |
| Cost-constrained | Toolkit (local) + Phoenix (local) |

The toolkit complements all four platforms via `obs_export_*` — it is not a replacement.
