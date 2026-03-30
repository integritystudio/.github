# R8: Model Routing Telemetry

**Version**: 1.0
**Date**: 2026-03-02
**Source**: [otel-v3-research-directions.md](otel-v3-research-directions.md) R8, white paper Section 8.3
**Status**: Done (v3.0.7) — `obs_routing_telemetry` MCP tool shipped
**Verification Date**: 2026-03-02 — vLLM SR, Not Diamond, OpenRouter, RouteLLM, OTel semconv, gateway docs verified

---

## 1. Introduction

### 1.1 Model Routing Landscape (2025-2026)

Model routing — selecting the optimal LLM for each query based on complexity, cost, or latency — has evolved from research concept to production infrastructure:

1. **Open-source routing** — vLLM Semantic Router v0.1 "Iris" (Jan 2026) implements a full OTel distributed tracing pipeline with named spans per routing stage and 96.4% classification accuracy.
2. **Commercial routing** — Not Diamond (powering OpenRouter's Auto Router), Martian, and Unify deliver 2-10x cost reduction with <1% quality degradation.
3. **Research advances** — RouteLLM (ICLR 2025), BEST-Route (ICML 2025), and Router-R1 (NeurIPS 2025) demonstrate complexity-based, test-time compute, and RL-based routing strategies.
4. **OTel gap** — No `gen_ai.operation.name = "route"` or dedicated routing span attributes exist. The `gen_ai.request.model` / `gen_ai.response.model` delta serves as an implicit audit trail.

### 1.2 Scope

This document covers: vLLM Semantic Router, commercial platforms, OTel conventions, routing algorithms, cost-quality measurement, LLM gateways, observability patterns, and research. It does not prescribe a routing implementation.

---

## 2. vLLM Semantic Router "Iris"

### 2.1 Overview

| Property | Detail |
|----------|--------|
| **Release** | v0.1, codename "Iris" (Jan 5, 2026) |
| **Architecture** | Envoy External Processor between users and vLLM backends |
| **Dev velocity** | 600+ PRs, 300+ issues, 50+ contributors since Sep 2025 |
| **Partners** | AMD co-development (Dec 2025); Red Hat integration |
| **GitHub** | [vllm-project/semantic-router](https://github.com/vllm-project/semantic-router) |
| **Blog** | [Iris release](https://blog.vllm.ai/2026/01/05/vllm-sr-iris.html) |

This is a Mixture-of-Models (MoM) system — selects among distinct deployed models per query. Not to be confused with Mixture-of-Experts (MoE), which is intra-model routing.

### 2.2 Signal-Decision Plugin Chain

Architecture blog: [Signal-Decision](https://blog.vllm.ai/2025/11/19/signal-decision.html) (Nov 2025)

**Signal types**: `keyword` (regex), `embedding` (neural similarity), `domain` (MMLU classification + custom LoRA), `factual` (fact-check trigger), `feedback` (satisfaction), `preference` (personalization)

**Decision engine**: Combines signals via AND/OR logic with priority ordering. Previously static features (jailbreak, PII, semantic caching) are now configurable per-decision plugins.

### 2.3 Classification

- **Model**: ModernBERT + LoRA adapters — 96.4% validation accuracy, ~12ms via native Rust/Candle
- **Baseline**: 14 MMLU domain categories; extensible via custom LoRA without full retraining
- **HF org**: [llm-semantic-router](https://huggingface.co/llm-semantic-router)

### 2.4 OTel Distributed Tracing

Docs: [Distributed Tracing](https://vllm-semantic-router.com/docs/tutorials/observability/distributed-tracing/)

Named spans in pipeline order:

| Span | Key Attributes |
|------|---------------|
| `semantic_router.classification` | `category.name`, `classification.time_ms` |
| `semantic_router.security.pii_detection` | detection result |
| `semantic_router.security.jailbreak_detection` | detection result |
| `semantic_router.cache.lookup` | `cache.hit` (bool) |
| `semantic_router.routing.decision` | strategy, confidence |
| `semantic_router.backend.selection` | selected model |
| `semantic_router.system_prompt.injection` | prompt modification |
| `semantic_router.upstream.request` | model call |

Config: `OTEL_SERVICE_NAME`, `OTEL_EXPORTER_OTLP_ENDPOINT`, `OTEL_RESOURCE_ATTRIBUTES`. Sampling: `always_on`, `probabilistic`. Exporters: `stdout`, `otlp`.

**Important**: These are router-specific span extensions, not part of the official GenAI semconv.

---

## 3. Commercial Routing Platforms

### 3.1 Not Diamond

| Property | Detail |
|----------|--------|
| **URL** | [notdiamond.ai](https://www.notdiamond.ai/) |
| **Approach** | Meta-model analyzing query characteristics; routes to best-fit model |
| **Performance** | Up to 25% accuracy improvement; up to 10x cost reduction |
| **Routing latency** | 10-100ms per recommendation |
| **Differentiator** | Custom router training on user evaluation data |
| **Compliance** | SOC-2 + ISO 27001 |

### 3.2 OpenRouter Auto Router

| Property | Detail |
|----------|--------|
| **Model** | `openrouter/auto` — powered by Not Diamond |
| **Pool** | 19 curated high-quality models |
| **Cost** | No routing surcharge; standard model rates |
| **Transparency** | Response includes `model` attribute showing which model handled the request |
| **Customization** | `plugins` parameter restricts to model subset |
| **Source** | [Announcement](https://openrouter.ai/announcements/happy-new-year-introducing-a-new-auto-router) (Jan 2025) |

Non-determinism caveat: Same prompt may route to different models as availability/pricing change.

### 3.3 Martian

| Property | Detail |
|----------|--------|
| **Funding** | $9M Series A; Accenture strategic investment |
| **Approach** | Per-query expert orchestration; enterprise compliance layer |
| **Framing** | Cost-quality Pareto frontier ([Up and to the Left](https://withmartian.com/post/up-and-to-the-left)) |
| **Benchmark** | Published RouterBench (see Section 6) |

### 3.4 Unify.ai

| Property | Detail |
|----------|--------|
| **Differentiator** | Real-time benchmarks updated every 10 minutes |
| **Router** | Neural network trained on dynamic benchmark data |
| **Control** | Three sliders — quality, cost, latency |
| **Custom** | Train custom routers on own data |

---

## 4. OTel Conventions for Routing

### 4.1 Current Routing-Relevant Attributes

| Attribute | Status | Routing Relevance |
|-----------|--------|-------------------|
| `gen_ai.request.model` | Stable | Pre-routing intent |
| `gen_ai.response.model` | Stable | Post-routing result |
| `gen_ai.provider.name` | Stable | Multi-provider discriminator |
| `gen_ai.operation.name` | Stable | Operation type (`chat`, `embeddings`) |

**The `gen_ai.request.model` vs `gen_ai.response.model` delta is the de facto routing audit trail**: if these differ, a routing decision occurred. This is implicit.

### 4.2 What is Missing

| Gap | Impact |
|-----|--------|
| No `gen_ai.operation.name = "route"` | No explicit routing span type |
| No `gen_ai.routing.strategy` | Cannot distinguish routing algorithms |
| No `gen_ai.routing.confidence` | Cannot assess routing decision quality |
| No `gen_ai.routing.candidate_models` | Cannot audit model pool |
| No routing span hierarchy | Gateway -> classifier -> model call not standardized |

### 4.3 Proposed Routing Span Structure

Derived from vLLM SR practice and semconv principles (not standardized):

```
# Parent routing span
gen_ai.operation.name = "route"
gen_ai.request.model = "auto"
gen_ai.routing.strategy = "meta_model"

  # Child: classifier span
  gen_ai.operation.name = "classify"
  gen_ai.routing.classifier = "ModernBERT-LoRA"
  gen_ai.routing.domain = "medical"
  gen_ai.routing.confidence = 0.94

  # Child: model call span
  gen_ai.operation.name = "chat"
  gen_ai.request.model = "gpt-4o"
  gen_ai.response.model = "gpt-4o"
```

---

## 5. Routing Strategies and Algorithms

### 5.1 Complexity-Based Binary Routing

**RouteLLM** (UC Berkeley / Anyscale, [arXiv:2406.18665](https://arxiv.org/abs/2406.18665), ICLR 2025):
- Four router types: weighted Elo, BERT classifier, LLM-based, matrix factorization
- Training: human preference data (LMSYS Chatbot Arena) + augmentation
- Results: 85% cost reduction on MT Bench, 45% on MMLU, 35% on GSM8K at comparable quality
- Transfer learning: generalizes when model pair changes at test time
- GitHub: [lm-sys/RouteLLM](https://github.com/lm-sys/RouteLLM)

### 5.2 Test-Time Compute Routing

**BEST-Route** (Microsoft Research, [arXiv:2506.22716](https://arxiv.org/abs/2506.22716), ICML 2025):
- Insight: For borderline queries, sampling N responses from a cheaper model and picking the best can match expensive model quality
- Jointly decides: which model AND how many samples
- Results: 60% cost reduction with <1% performance degradation
- GitHub: [microsoft/best-route-llm](https://github.com/microsoft/best-route-llm)

### 5.3 Cascade Routing

**Unified Routing and Cascading** (ETH Zurich, [arXiv:2410.10347](https://arxiv.org/abs/2410.10347)):
- Generalizes routing + cascading: skip models, reorder, stop early
- Results: SWE-Bench +14% over baselines; RouterBench +1-4%
- GitHub: [eth-sri/cascade-routing](https://github.com/eth-sri/cascade-routing)

### 5.4 RL-Based Multi-Round Routing

**Router-R1** (UIUC, [arXiv:2506.09033](https://arxiv.org/abs/2506.09033), NeurIPS 2025):
- Router is an LLM; interleaves "think" and "route" actions
- Integrates sub-model responses into evolving context
- Generalizes to unseen models using pricing/latency/performance descriptors
- GitHub: [ulab-uiuc/Router-R1](https://github.com/ulab-uiuc/Router-R1)

### 5.5 Selective Reasoning Routing

**When to Reason** (IBM Research, [arXiv:2510.08731](https://arxiv.org/abs/2510.08731), NeurIPS 2025 MLForSys):
- Routes between reasoning-augmented and direct inference
- Results: +10.2pp MMLU-Pro accuracy, -47.1% latency, -48.5% token consumption
- Integrated with Envoy ext_proc for production

### 5.6 Strategy Comparison

| Strategy | Cost Savings | Quality Impact | Key Paper |
|----------|-------------|----------------|-----------|
| Binary (RouteLLM) | 35-85% | Comparable | ICLR 2025 |
| Test-time compute (BEST-Route) | 60% | <1% degradation | ICML 2025 |
| Cascade (ETH) | 13-80% relative | +1-14% | arXiv 2410.10347 |
| RL multi-round (Router-R1) | Variable | Pareto-optimal | NeurIPS 2025 |
| Reasoning selective | 48.5% tokens | +10.2pp accuracy | NeurIPS 2025 |

---

## 6. Cost-Quality Measurement

### 6.1 FrugalGPT

- **Paper**: [arXiv:2305.05176](https://arxiv.org/abs/2305.05176) (Stanford; TMLR Dec 2024)
- LLM cascade learning optimal model combinations per query class
- Peak: Match GPT-4 with **98% cost reduction**; or +4% accuracy at same cost
- GitHub: [stanford-futuredata/FrugalGPT](https://github.com/stanford-futuredata/FrugalGPT)

### 6.2 RouterBench

- **Paper**: [arXiv:2403.12031](https://arxiv.org/abs/2403.12031) (Martian; ICML 2024)
- 405,000+ inference outcomes, 11 LLMs, 8 datasets, 64 tasks
- De facto standard benchmark — virtually every 2025 routing paper evaluates on it
- GitHub: [withmartian/routerbench](https://github.com/withmartian/routerbench)

### 6.3 Routing Effectiveness Metrics

| Metric | Definition |
|--------|-----------|
| Cost reduction % | `(cost_single - cost_router) / cost_single` |
| Quality delta (AIQ) | `accuracy_router - accuracy_single` |
| Routing accuracy | % of queries routed to objectively best model |
| Cache hit rate | % served from semantic cache |
| Fallback rate | % rerouted from primary model |
| P50/P99 routing latency | Routing overhead in ms |

### 6.4 Pareto Frontier

The canonical visualization: X-axis = inference cost, Y-axis = quality metric. A router "pushes" the frontier up-and-left by matching query difficulty to model capability. RouterBench uses convex hull operations to compute the Pareto-optimal frontier.

---

## 7. LLM Gateway/Proxy Routing

### 7.1 Gateway Comparison

| Gateway | Routing Strategies | OTel | Key Feature |
|---------|-------------------|------|-------------|
| **LiteLLM** | simple-shuffle, least-busy, usage, latency, semantic | OTLP HTTP/gRPC | Broad model coverage; open source |
| **Portkey** | latency, cost, weighted, canary, circuit breaker | OTLP ingest | Gartner Cool Vendor; agent obs |
| **Helicone** | latency P2C+PeakEWMA, weighted, cost | Built-in logging | Rust; 8ms P50 |
| **Bifrost** (Maxim) | Adaptive load balancer, fallback | — | Go; 11us overhead at 5K RPS |
| **Kong AI Gateway** | Semantic load balancing, provider-agnostic | Native OTel plugin | Enterprise API gateway |
| **vLLM SR Iris** | Signal-Decision Plugin Chain | Full span pipeline | Deepest OTel integration |
| **OpenRouter** | Meta-model (Not Diamond) | `model` in response | 19-model marketplace |
| **aurelio-labs/semantic-router** | Embedding similarity | — | Sub-ms deterministic; Python lib |

### 7.2 LiteLLM OTel Integration

Single-line setup from v1.81.0. Config: `OTEL_TRACER_NAME` (default `litellm`), `OTEL_SERVICE_NAME`. Metadata dict fields exported as span attributes. Supports OTLP HTTP and gRPC. Parent span: "Received Proxy Server Request".

Source: [LiteLLM OTel](https://docs.litellm.ai/docs/observability/opentelemetry_integration)

---

## 8. Observability for Routing Decisions

### 8.1 Recommended Span Attributes

**Request-time** (on router span):
- `gen_ai.request.model` — client intent
- `routing.strategy` — meta_model | complexity | latency | cost | cascade
- `routing.candidate_models` — models considered
- `routing.selected_model` — chosen model
- `routing.classifier` — classifier model/method
- `routing.classification.confidence` — decision confidence
- `routing.classification.time_ms` — routing latency

**Result** (on model call span):
- `gen_ai.response.model` — actual responding model
- `routing.fallback_triggered` — bool
- `routing.fallback_reason` — rate_limit | timeout | server_error

### 8.2 Dashboard Panels

1. **Model distribution** — Stacked area chart per model over time; detect drift
2. **Cost per routed query** — Rolling average vs. single-model baseline
3. **Quality delta by domain** — AIQ per category; detect underperforming segments
4. **Routing latency** — P50/P99 overhead; alert >100ms
5. **Cache hit rate** — Semantic cache effectiveness over time
6. **Fallback rate by provider** — Detect provider degradation
7. **Pareto scatter** — Per-query cost vs. quality; live frontier visualization

### 8.3 Alert Patterns

| Alert | Condition | Severity |
|-------|-----------|----------|
| Routing latency spike | P99 >500ms | Critical |
| High fallback rate | >10% over 5-min window | High |
| Model distribution skew | Single model >80% traffic | Medium |
| Classification confidence drop | Mean <0.7 | Medium |
| Quality regression | AIQ drops >5% vs. prior 24h | High |
| Cache hit rate drop | >20pp decline | Low |
| Cost budget breach | Rolling cost exceeds budget | High |

---

## 9. MoE vs. MoM Distinction

| Dimension | Mixture of Experts (MoE) | Mixture of Models (MoM) |
|-----------|--------------------------|-------------------------|
| Granularity | Sub-model (expert FFN layers) | Whole models |
| Decision point | Inside model at inference | At gateway before model |
| Examples | DeepSeek-R1, Kimi K2 | vLLM SR, Not Diamond, RouteLLM |
| Visibility | Opaque to caller | Observable via OTel spans |
| OTel integration | Not applicable | Full span hierarchy possible |

2025 frontier models (DeepSeek-R1, GPT-OSS-120B) use MoE internally; MoM routing at the system level is orthogonal and additive.

---

## 10. Toolkit Relevance

| Toolkit Component | Relationship |
|-------------------|-------------|
| `MODEL_PRICING` at `src/lib/constants.ts:362` | Per-model pricing — routing telemetry correlates model selection with cost outcomes |
| `calculateCost()` at `src/lib/cost-estimation.ts:214` | Cost calculation — routing-aware cost includes classifier overhead |
| `obs_estimate_cost` tool | Cost estimation — could incorporate routing savings projections |
| `computeTrend()` at `src/lib/quality/quality-trends.ts:284` | Trend computation — quality delta tracking across routing strategy changes |

A routing telemetry extension would correlate model selection decisions with cost and quality outcomes, enabling data-driven routing strategy optimization.

---

## 11. Open Questions

- **OTel routing semconv**: Has any SIG working group proposed routing-specific GenAI attributes? No published proposal found.
- **OpenRouter accuracy**: No published routing accuracy numbers for the 19-model Auto Router pool
- **Gateway namespacing**: Each gateway uses proprietary span attributes; no `gen_ai.routing.*` namespace standardized
- **Multi-turn routing**: How should optimal routing shift between conversation turns?
- **Events vs. attributes**: Should routing decisions be serialized as OTel events or span attributes?
- **Routing for reasoning**: IBM's "When to Reason" shows reasoning-mode routing is as impactful as model selection — should this be a distinct routing dimension?
