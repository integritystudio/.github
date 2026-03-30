# LLM-as-Judge Architecture

**Version**: 2.1.0 | **Status**: Production | **Last Updated**: 2026-03-26

## Overview

LLM-as-Judge evaluates AI outputs using AI judges. This module implements G-Eval, QAG, bias mitigation, and production resilience patterns. Core types and Zod schemas are co-located in `src/lib/judge/llm-as-judge.ts` as the single source of truth; TypeScript types are derived via `z.infer<>`.

## Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                    LLM-as-Judge Components                       │
├─────────────────────────────────────────────────────────────────┤
│  Evaluation Patterns          Production Utilities              │
│  ├── gEval()                  ├── JudgeCircuitBreaker           │
│  ├── qagEvaluate()            ├── evaluateWithRetry (60s cap)   │
│  ├── mitigatedPairwiseEval()  ├── runCanaryEvaluations          │
│  └── panelEvaluation()        └── LOG_LEVEL configuration       │
├─────────────────────────────────────────────────────────────────┤
│  Security Layer                                                  │
│  ├── 14 prompt injection patterns + Unicode TR39                │
│  ├── Input validation (64KB total, 10KB/field, 20 context)      │
│  ├── Safe JSON parsing (depth=5, optimized iteration)           │
│  └── 30s default timeout on all LLM calls                       │
└─────────────────────────────────────────────────────────────────┘
```

## Evaluation Patterns

### G-Eval (Chain-of-Thought + Logprobs)

```
Input → Generate eval steps → Evaluate with CoT → Normalize via logprobs → Score
```

G-Eval generates evaluation criteria steps dynamically, then uses logprob-weighted normalization to produce calibrated scores. The `normalizeWithLogprobs()` function computes a probability-weighted average across the score distribution rather than taking the argmax, making scores more sensitive to near-ties.

### QAG (Question-Answer Generation)

```
Output → Extract statements → Generate yes/no questions → Answer from context → Score
```

QAG decomposes outputs into atomic claims, converts each into a verifiable yes/no question, then answers each question against the source context. Score is the fraction of questions answered affirmatively. Optimized for faithfulness evaluation.

### Bias Mitigation

| Strategy | Function | Mechanism |
|----------|----------|-----------|
| Position bias | `mitigatedPairwiseEval()` | Double evaluation with response order swapped; winner must be consistent |
| Multi-judge | `panelEvaluation()` | Median score from multiple models; agreement computed as inter-rater confidence |

## Security

| Threat | Mitigation |
|--------|------------|
| Prompt injection | 14 regex patterns + Unicode TR39 normalization (`sanitizeForPrompt()`) |
| ReDoS | 64KB total / 10KB per field cap before pattern matching |
| JSON depth attacks | Depth limit at 5, iterative not recursive (`safeJSONParse()`) |
| Context flooding | Hard cap at 20 context items (`validateTestCase()`) |
| Judge timeout DoS | 30s timeout on all LLM calls (`evaluateWithRetry()`) |
| Score manipulation | Logprob normalization + pairwise order swap |

Security utilities: `src/lib/judge/llm-judge-security.ts`

## Production Utilities

**Circuit breaker** (`JudgeCircuitBreaker`): configurable failure threshold and reset timeout; ignores transient 429s; supports fallback model switching.

**Retry logic** (`evaluateWithRetry`): exponential backoff capped at 60s, preserves `error.cause` chain, validates score range on each attempt.

**Canary evaluations** (`runCanaryEvaluations`): built-in test cases (perfect answer, hallucination, off-topic) used to verify judge health before production use.

## OTel Integration

`gEval()`, `qagEvaluate()`, and `panelEvaluation()` emit `gen_ai.evaluation.result` OTel events via `emitEvaluationEvent()`. Events include `inputHash` for provenance tracing (EU AI Act Article 13).

## Schemas

Exported Zod schemas (runtime validation at system boundaries):

| Schema | Validates |
|--------|-----------|
| `evaluationEventSchema` | `EvaluationEvent` — enforces `scoreValue \| scoreLabel` present |
| `testCaseSchema` | `TestCase` — input/output required, min length 1 |
| `evalResultSchema` | `EvalResult` — score in [0, 1], reason required |
| `gEvalConfigSchema` | `GEvalConfig` — evaluationParams must have at least 1 item |
| `pairwiseResultSchema` | `PairwiseResult` — winner enum + confidence in [0, 1] |

Hooks integration schemas (webhook delivery of external scores) live separately in `src/lib/judge/evaluation-hooks-schemas.ts`.

## Files

| File | Description |
|------|-------------|
| `src/lib/judge/llm-as-judge.ts` | Barrel: core types, Zod schemas, submodule re-exports |
| `src/lib/judge/evaluation-hooks-schemas.ts` | Zod schemas for webhook-based evaluation injection |
| `src/lib/judge/llm-judge-security.ts` | Sanitization, validation, safe JSON parsing |
| `src/lib/judge/llm-judge-geval.ts` | G-Eval CoT + logprobs |
| `src/lib/judge/llm-judge-qag.ts` | QAG faithfulness evaluation |
| `src/lib/judge/llm-judge-bias.ts` | Position bias mitigation, panel evaluation |
| `src/lib/judge/llm-judge-resilience.ts` | Circuit breaker, retry, canary evaluations |
| `src/lib/judge/llm-judge-dag.ts` | DAG-based structured evaluation |
| `src/lib/judge/llm-judge-config.ts` | Configuration and built-in metric definitions |
| `src/lib/judge/llm-judge-domain.ts` | Pre-built RAG and agent GEvalConfig constants; `DomainEvaluator` |
| `src/lib/judge/evaluation-hooks.ts` | Webhook hook executor implementation |

## Related

- [Quality Evaluation](quality-evaluation.md) — broader evaluation context
- [Agent-as-Judge](agent-as-judge.md) — multi-agent evaluation patterns
- [Interface Research Index](../interface/README.md) — explainability roadmap
