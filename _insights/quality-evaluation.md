# Quality Evaluation Architecture

**Version**: 2.1.0
**Status**: Production
**Last Updated**: 2026-03-26

## Overview

Quality evaluation addresses the "invisible failure" problem where LLM systems appear operational but produce low-quality outputs. This layer provides evaluation event storage, multi-platform export, and LLM-as-Judge patterns.

## Architecture

```
┌──────────────────────────────────────────────────────────────────────────────┐
│                        Quality Evaluation Layer                               │
├──────────────────────────────────────────────────────────────────────────────┤
│  Storage & Query                                                              │
│  ├── EvaluationResult schema (OTel GenAI semantic conventions)               │
│  ├── queryEvaluations() - JSONL backend with streaming aggregation           │
│  └── obs_query_evaluations - MCP tool with filters + aggregations            │
├──────────────────────────────────────────────────────────────────────────────┤
│  Platform Exports                                                             │
│  ├── Langfuse        obs_export_langfuse    OTLP + basic auth               │
│  ├── Confident AI    obs_export_confident   API key + env tagging           │
│  ├── Arize Phoenix   obs_export_phoenix     Bearer auth + project org       │
│  └── Datadog         obs_export_datadog     Two-phase spans + eval metrics  │
├──────────────────────────────────────────────────────────────────────────────┤
│  LLM-as-Judge (see quality/llm-as-judge.md)                                  │
│  ├── G-Eval (CoT + logprobs)    ├── Circuit breaker                         │
│  ├── QAG (faithfulness)         ├── Retry with backoff                       │
│  ├── Position bias mitigation   └── Canary evaluations                       │
│  └── Multi-judge panels                                                       │
├──────────────────────────────────────────────────────────────────────────────┤
│  Compliance                                                                   │
│  └── obs_query_verifications - EU AI Act human verification tracking         │
└──────────────────────────────────────────────────────────────────────────────┘
```

## Core Types

```typescript
// src/backends/index.ts

interface EvaluationResult {
  timestamp: string;
  evaluationName: string;       // gen_ai.evaluation.name (required)
  scoreValue?: number;          // gen_ai.evaluation.score.value
  scoreLabel?: string;          // gen_ai.evaluation.score.label
  scoreUnit?: string;           // "ratio_0_1", "percentage"
  explanation?: string;         // gen_ai.evaluation.explanation
  evaluator?: string;           // Model, human, system identity
  evaluatorType?: 'llm' | 'human' | 'rule' | 'classifier';
  responseId?: string;          // gen_ai.response.id (correlation)
  traceId?: string;
  spanId?: string;
  sessionId?: string;
}

type EvaluatorType = 'llm' | 'human' | 'rule' | 'classifier';
```

## MCP Tools

### obs_query_evaluations

Query evaluation events with filtering and aggregation.

| Parameter | Type | Description |
|-----------|------|-------------|
| `evaluationName` | string | Substring match |
| `scoreMin/scoreMax` | number | Score range filter |
| `scoreLabel` | string | Exact match |
| `evaluatorType` | enum | llm, human, rule, classifier |
| `aggregation` | enum | avg, min, max, count, p50, p95, p99 |
| `groupBy` | array | evaluationName, scoreLabel, evaluator |

### Export Tools

All export tools share common filters plus platform-specific options:

| Tool | Auth | Key Features |
|------|------|--------------|
| `obs_export_langfuse` | Basic auth (pk:sk) | OTLP /v1/traces, retry with backoff |
| `obs_export_confident` | API key | Environment tagging, metric collections |
| `obs_export_phoenix` | Bearer token | Project organization, legacy auth support |
| `obs_export_datadog` | DD_API_KEY | Two-phase export, auto metric type detection |

### obs_query_verifications

EU AI Act compliance - query human verification events.

| Parameter | Type | Description |
|-----------|------|-------------|
| `sessionId` | string | Session filter |
| `verificationType` | enum | approval, rejection, override, review |

## Environment Variables

### Langfuse
| Variable | Required | Description |
|----------|----------|-------------|
| `LANGFUSE_ENDPOINT` | Yes | API endpoint URL |
| `LANGFUSE_PUBLIC_KEY` | Yes | Public key (pk-lf-...) |
| `LANGFUSE_SECRET_KEY` | Yes | Secret key (sk-lf-...) |

### Confident AI
| Variable | Required | Description |
|----------|----------|-------------|
| `CONFIDENT_ENDPOINT` | No | Custom endpoint |
| `CONFIDENT_API_KEY` | Yes | API key |

### Arize Phoenix
| Variable | Required | Description |
|----------|----------|-------------|
| `PHOENIX_COLLECTOR_ENDPOINT` | Yes | Collector URL |
| `PHOENIX_API_KEY` | Yes | Bearer token |

### Datadog
| Variable | Required | Description |
|----------|----------|-------------|
| `DD_API_KEY` | Yes | Datadog API key |
| `DD_SITE` | No | Site (datadoghq.com, eu, us3, us5, ap1) |
| `DD_LLMOBS_ML_APP` | No | LLM application name |

## OTel Event Structure

```
Trace: Customer Support Query
├── Span: invoke_agent CustomerSupportBot
│   ├── Span: chat claude-3-opus
│   │   └── Event: gen_ai.evaluation.result
│   │       ├── gen_ai.evaluation.name: "Relevance"
│   │       ├── gen_ai.evaluation.score.value: 0.92
│   │       ├── gen_ai.evaluation.score.label: "relevant"
│   │       └── gen_ai.evaluation.explanation: "Response addresses query"
│   │
│   └── Span: execute_tool lookup_customer
│       └── Event: gen_ai.evaluation.result
│           ├── gen_ai.evaluation.name: "ToolCorrectness"
│           └── gen_ai.evaluation.score.label: "pass"
```

## Security

| Feature | Implementation |
|---------|----------------|
| DNS rebinding | URL validation, HTTPS-only for exports |
| Memory protection | 600MB OOM threshold, MAX_AGGREGATION_GROUPS=10,000 |
| Credential sanitization | Masked in logs, error messages |
| Timestamp validation | Year 2000-3000 range |

## File Structure

```
src/
├── backends/
│   ├── index.ts                    # EvaluationResult, EvaluatorType types
│   └── local-jsonl.ts              # queryEvaluations() method
├── lib/
│   ├── core/
│   │   └── constants.ts            # Export env vars, HttpStatus
│   ├── exports/
│   │   ├── export-utils.ts         # Shared export utilities
│   │   ├── langfuse-export.ts      # Langfuse OTLP export
│   │   ├── confident-export.ts     # Confident AI export
│   │   ├── phoenix-export.ts       # Arize Phoenix export
│   │   └── datadog-export.ts       # Datadog LLM Obs export
│   ├── judge/
│   │   └── llm-as-judge.ts         # LLM-as-Judge patterns
│   └── audit/
│       └── verification-events.ts  # EU AI Act compliance
└── tools/
    ├── query-evaluations.ts        # obs_query_evaluations
    ├── export-langfuse.ts          # obs_export_langfuse
    ├── export-confident.ts         # obs_export_confident
    ├── export-phoenix.ts           # obs_export_phoenix
    ├── export-datadog.ts           # obs_export_datadog
    └── query-verifications.ts      # obs_query_verifications
```

## Test Coverage

| Component | Tests | File |
|-----------|-------|------|
| Query evaluations | 45+ | query-evaluations.test.ts |
| Langfuse export | 71 | langfuse-export.test.ts |
| Confident AI export | 40+ | confident-export.test.ts |
| Phoenix export | 50+ | phoenix-export.test.ts |
| Datadog export | 160 | datadog-export.test.ts |
| LLM-as-Judge | 108 | llm-as-judge.test.ts |
| Verifications | 30+ | query-verifications.test.ts |

## Regression Detection: LOO Cross-Validation

`sweepWithCrossValidation()` in `src/lib/quality/quality-feature-engineering.ts` extends the backtest sweep with leave-one-out (LOO) cross-validation over labeled incident windows. It runs the full 48-config sweep grid, then re-runs it N times — each time holding out one incident — to measure how stable the best-F1 config selection is.

```typescript
import { sweepWithCrossValidation } from 'src/lib/quality/quality-feature-engineering.js';

const result: CrossValidationResult = sweepWithCrossValidation(snapshots, incidents);
// result.fullSweep      — BacktestSweepResult over all incidents
// result.folds          — CrossValidationFold[] (one per held-out incident)
// result.parameterStability — ParameterStability (std dev per config axis)
// result.isStable       — true when every LOO fold selects the same best-F1 config
```

### Key Types

```typescript
/** Std dev of each BacktestConfig parameter across LOO folds */
interface ParameterStability {
  varianceThresholdStdDev: number;
  coverageDropoutThresholdStdDev: number;
  confirmationWindowStdDev: number;
}

interface CrossValidationResult {
  fullSweep: BacktestSweepResult;
  folds: CrossValidationFold[];
  parameterStability: ParameterStability;
  /** true when every LOO fold selects the same best-by-F1 config as the full sweep */
  isStable: boolean;
}
```

`ParameterStability` (previously named `ParameterVariance` in early drafts) reports the standard deviation of the best-F1 config parameters across LOO folds. Higher values indicate the config selection is sensitive to which incidents are included. `isStable: true` means LOO selection is consistent with full-sweep selection.

## Version History

| Version | Changes |
|---------|---------|
| **2.0.0** | Initial production release — `queryEvaluations()`, Langfuse/Confident AI/Phoenix/Datadog export, `obs_query_verifications` |
| **2.1** | OTel evaluation events emitted from judge functions; `inputHash` provenance for EU AI Act Article 13 compliance |
| **2.9.3** | `export-base.ts` shared handler factory (commit 7789dc9) — eliminated ~200 lines of duplication across 4 export tools; `exportFilterSchema`, `exportOptionsSchema` shared Zod schemas (commit 0c0980b) |
| **2.1.0** | `sweepWithCrossValidation()` LOO cross-validation added; `ParameterStability` type (renamed from `ParameterVariance`) reports config stability across folds |

---

## Planned Extensions

~~Per the [Interface Research Index](../interface/README.md), v2.1 will emit `gen_ai.evaluation.result` OTel events from `gEval()`, `qagEvaluate()`, and `panelEvaluation()` (recommendation R2.1)~~ **COMPLETE** (commit 3401be3). Events include evaluation name, score, explanation, evaluator identity, duration, and `inputHash` provenance via `emitEvaluationEvent()` (commit 92b3fd4). Downstream platforms (Langfuse, Datadog, Phoenix) can display evaluation results natively within trace views. Provenance fields satisfy EU AI Act Article 13 compliance (August 2026 deadline).

## Related Documentation

- [LLM-as-Judge Architecture](llm-as-judge.md) - G-Eval, QAG, bias mitigation
- [Agent-as-Judge Architecture](agent-as-judge.md) - Multi-agent evaluation patterns
- [Interface Research Index](../interface/README.md) - Explainability roadmap and recommendation consolidation
- [Contributing Guide](../CONTRIBUTING.md) - Error handling conventions and commit format
- [Changelog](../CHANGELOG.md) - Release details
