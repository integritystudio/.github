# R4: Real-Time Hallucination Detection

**Version**: 1.0
**Date**: 2026-03-02
**Source**: [otel-v3-research-directions.md](otel-v3-research-directions.md) R4, white paper Section 8.2
**Status**: Done — `obs_hallucination_detection` MCP tool shipped
**Verification Date**: 2026-03-02 — HaluGate, arXiv papers, provider logprob APIs, guardrail frameworks verified

---

## 1. Introduction

### 1.1 Hallucination Detection Landscape (2025-2026)

Real-time hallucination detection has evolved from research prototypes to production-viable pipelines. The field has converged on four architectural approaches:

1. **Pre-classification gating** — Binary classifiers determine whether a query requires factual verification before investing in detection (HaluGate Sentinel). Achieves 72.2% efficiency gain by bypassing non-factual queries.
2. **Log-probability analysis** — Lightweight sequence models classify hallucinations from token-level log-prob time series (HALT). 30x smaller and 60x faster than encoder-based approaches.
3. **Hidden state probing** — Linear probes on intermediate model activations detect hallucinated entities during the same forward pass (arXiv 2509.03531). 0.90 AUC on 70B-parameter models.
4. **Streaming CoT monitoring** — Treat hallucination as an evolving latent state over reasoning steps (arXiv 2601.02170). 87%+ accuracy with zero additional inference cost.

### 1.2 Scope

This document covers: production pipelines, research approaches, provider logprob APIs, guardrail frameworks, OTel integration patterns, and latency/accuracy tradeoffs. It does not propose a specific implementation — it provides reference material for commitment decisions.

---

## 2. Production Pipelines

### 2.1 HaluGate (vLLM Semantic Router)

HaluGate is a conditional, token-level hallucination detection pipeline integrated into vLLM Semantic Router v0.1 "Iris" (Jan 5, 2026).

- **Blog**: [Token-Level Truth: Real-Time Hallucination Detection](https://blog.vllm.ai/2025/12/14/halugate.html) (Dec 14, 2025)
- **GitHub**: [vllm-project/semantic-router](https://github.com/vllm-project/semantic-router)

#### Three-Stage Pipeline

| Stage | Component | Function | Overhead |
|-------|-----------|----------|----------|
| 1 | **Sentinel** | Binary prompt classification: `NO_FACT_CHECK_NEEDED` vs `FACT_CHECK_NEEDED` | ~76-162ms |
| 2 | **Detector** | Token-level: which response tokens are unsupported by context | Scales with context |
| 3 | **Explainer** | NLI-based classification explaining why flagged spans are problematic | Included in pipeline |

**Sentinel model**: ModernBERT + LoRA binary classifier. HuggingFace: [llm-semantic-router/halugate-sentinel](https://huggingface.co/llm-semantic-router/halugate-sentinel). Training data: SQuAD, TriviaQA, HotpotQA, TruthfulQA, CoQA, Dolly, Alpaca, WritingPrompts, HaluEval.

**Efficiency**: ~35% of production queries are non-factual (creative writing, code, opinion). Pre-classification achieves 72.2% efficiency gain by routing these directly to generation.

#### HTTP Header Propagation

Detection results flow downstream via HTTP response headers:

| Header | Description |
|--------|-------------|
| `x-vsr-fact-check-needed` | Whether the query required factual verification |
| `x-vsr-unverified-factual-response` | Response contains unverified factual claims |
| `x-vsr-verification-context-missing` | Fact-checking warranted but no RAG/tool context provided |

Enables downstream gateways, proxies, or application layers to implement policies (quarantine, re-query, user warning) without modifying generation logic.

#### Performance

| Context Size | Pipeline Overhead |
|-------------|-------------------|
| 4K tokens | ~125ms |
| 16K tokens | ~365ms |
| Sentinel only | ~76-162ms |
| vs. LLM generation | 5-30 seconds typical |

---

## 3. Research Approaches

### 3.1 Streaming CoT Hallucination Detection

- **Paper**: [arXiv:2601.02170](https://arxiv.org/abs/2601.02170) (Jan 5, 2026)
- **Authors**: Haolang Lu, Minghui Pan, Ripeng Li et al.

**Core insight**: Hallucination in long CoT is not a one-off event — it is an evolving latent state that propagates across reasoning steps. The paper frames it as a Hidden Markov-style problem.

**Approach**:
- Step-level hallucination probing on intermediate reasoning steps as they stream
- Prefix-level signal aggregates step signals into a trajectory-wide confidence score
- Probes trained on hidden states already computed during normal forward passes
- Detection runs in parallel with token generation

| Level | Accuracy |
|-------|----------|
| Step-level | 87%+ |
| Prefix-level | 87%+ |
| CoT instance identification (streaming) | 78% |

**Zero additional inference cost**: All computation reuses activations from the primary generation pass. Directly applicable to o1/R1-style reasoning models.

---

### 3.2 HALT: Log-Prob Time Series

- **Paper**: [arXiv:2602.02888](https://arxiv.org/abs/2602.02888) (Feb 2, 2026)
- **Authors**: Ahmad Shapiro, Karan Taneja, Ashok Goel

**Architecture**: GRU (Gated Recurrent Unit) + entropy features over top-20 token log-probabilities. Learns model-specific calibration bias over the token sequence.

| Metric | Value |
|--------|-------|
| Size vs. Lettuce (fine-tuned ModernBERT-base) | 30x smaller |
| Speedup on HUB benchmark | 60x |
| Variants | HALT-L (Llama 3.1-8B), HALT-Q (Qwen 2.5-7B) |

**Design advantages**:
- No access to hidden states or attention maps required — only output log-probs
- Works with any model exposing logprobs via API
- Single-pass classification from already-generated log-prob sequence
- Negligible inference cost relative to primary LLM call

**Open question**: Cross-model generalization — HALT-L and HALT-Q are trained on specific model families.

---

### 3.3 Token-Level Entity Detection

- **Paper**: [arXiv:2509.03531](https://arxiv.org/abs/2509.03531) (Sep 2025)
- **Project**: [hallucination-probes.com](https://www.hallucination-probes.com/)
- **GitHub**: [obalcells/hallucination_probes](https://github.com/obalcells/hallucination_probes)

**Approach**: Lightweight linear classifiers trained on hidden activations detect fabricated entities (names, dates, citations) during the same forward pass.

| Model | AUC (probes) | AUC (semantic entropy baseline) |
|-------|-------------|-------------------------------|
| Llama-3.3-70B | 0.90 | 0.71 |
| 4 other families | Consistent outperformance | — |

**Scaling**: Probes add negligible FLOPs relative to 70B forward pass. Cross-model transfer demonstrated — annotated responses from one model train classifiers effective on others.

---

### 3.4 Additional Approaches

| Approach | Paper | Key Insight |
|----------|-------|-------------|
| **Semantic Entropy** | [Nature 2024](https://www.nature.com/articles/s41586-024-07421-0) | Uncertainty over meaning of multiple generations. AUROC 0.7-0.95 but 5-10x inference cost. |
| **Semantic Entropy Probes (SEPs)** | [arXiv:2406.15927](https://arxiv.org/abs/2406.15927) | Approximates SE from single-generation hidden states. Near-zero overhead. |
| **Semantic Energy** | [arXiv:2508.14496](https://arxiv.org/abs/2508.14496) (Aug 2025) | Boltzmann energy distribution over logits. Captures cases where SE fails. |
| **OOD-to-Hallucination** | [arXiv:2602.07253](https://arxiv.org/abs/2602.07253) (Feb 2026) | Geometric view — applies OOD detection to next-token prediction. Training-free, single-sample. |
| **UQLM** (CVS Health) | [GitHub](https://github.com/cvs-health/uqlm), [arXiv:2507.06196](https://arxiv.org/html/2507.06196) | Suite of scorers: black-box UQ, white-box UQ, LLM-as-judge. Healthcare AI focus. |

---

## 4. Provider Logprob APIs

### 4.1 Availability Matrix

| Provider | Logprobs GA | Top-N Alternatives | Notes |
|----------|-------------|-------------------|-------|
| **OpenAI** | Yes | `top_logprobs: 0-5` | Well documented, stable. [Cookbook](https://cookbook.openai.com/examples/using_logprobs) |
| **Gemini (Vertex AI)** | Yes (1.5/2.0) | Yes | **Removed from Gemini 2.5** ([forum](https://discuss.ai.google.dev/t/logprobs-is-not-enabled-for-gemini-models/107989)) |
| **Anthropic** | No | N/A | Returns `null`; no native support. Third-party: [anthropic-logprobs](https://github.com/anerli/anthropic-logprobs) |
| **Together AI** | Yes (open models) | Yes | Supports HALT-style use. [Docs](https://docs.together.ai/docs/logprobs) |
| **vLLM (self-hosted)** | Yes | Configurable | Full control |

### 4.2 Hallucination Signal from Logprobs

Per-token log probability serves as an uncertainty signal: `perplexity = exp(-mean(logprobs))`. Higher perplexity indicates higher uncertainty and potential hallucination risk.

**HALT compatibility**: Requires top-20 log-prob sequences. OpenAI provides top-5; vLLM provides configurable N. Anthropic's lack of logprob support is a significant gap for Claude-centric stacks.

---

## 5. Guardrail Frameworks

### 5.1 Guardrails AI

- **GitHub**: [guardrails-ai/guardrails](https://github.com/guardrails-ai/guardrails)
- Validator framework with `on_fail` policies (`fix`, `reask`, `noop`)
- Provenance validators check factual grounding against source documents
- [Guardrails Index](https://www.guardrailsai.com/blog/reduce-ai-hallucinations-provenance-guardrails): 24 guardrails benchmarked across 6 risk categories

### 5.2 NeMo Guardrails (NVIDIA)

- **GitHub**: [NVIDIA-NeMo/Guardrails](https://github.com/NVIDIA-NeMo/Guardrails)
- Patronus AI Lynx (70B and 8B) integrated as hallucination rail
- Detection rate: 70% (text-davinci-003), 95% (gpt-3.5-turbo)
- Pattern-match rails <1ms; LLM-based rails 50-300ms each
- Colang DSL for dialogue-level guardrail flows

### 5.3 SelfCheckGPT

- **Paper**: [arXiv:2303.08896](https://arxiv.org/abs/2303.08896)
- Black-box, zero-resource: samples N responses, consistent facts = grounded
- Original paper uses N=20 (20x cost); N=3-5 with NLI variant is practical
- Best suited for async batch evaluation, not streaming

---

## 6. OTel Integration Patterns

### 6.1 Current Spec Status

OTel GenAI semconv (v1.37+) includes `gen_ai.evaluation.score.*` events and hallucination indicator attributes. No dedicated `gen_ai.hallucination.*` span attribute is standardized.

### 6.2 Recommended Span Pattern

Emit detection results as span events on the `gen_ai.system` span:

```
gen_ai.evaluation.score.value = 0.85
gen_ai.evaluation.score.label = "hallucination_risk"
gen_ai.evaluation.score.method = "halugate_sentinel"
```

Source: [OTel GenAI Events Spec](https://opentelemetry.io/docs/specs/semconv/gen-ai/gen-ai-events/)

### 6.3 Platform Support

| Platform | Integration |
|----------|-------------|
| **Traceloop OpenLLMetry** | Built-in Faithfulness and QA Relevancy monitors; auto-emits quality spans |
| **Datadog** | [Native OTel GenAI semconv v1.37+](https://www.datadoghq.com/blog/llm-otel-semantic-convention/) — hallucination scores as custom span attributes |
| **OneUptime** | [OTel hallucination detection guide](https://oneuptime.com/blog/post/2026-01-30-hallucination-detection/view) (Jan 2026) |

---

## 7. Latency Budget

| Approach | Overhead | Notes |
|----------|----------|-------|
| Linear probes (same forward pass) | <5ms | Hidden state tap; no extra model call |
| HALT (GRU on log-probs) | ~2ms | Negligible post-generation inference |
| HaluGate Sentinel only | ~76-162ms | ModernBERT binary classification |
| HaluGate full pipeline (4K ctx) | ~125ms | Sentinel + Detector + Explainer |
| HaluGate full pipeline (16K ctx) | ~365ms | Scales with context length |
| NeMo Guardrails (pattern match) | <1ms | Rule-based only |
| NeMo Guardrails (LLM rail) | 50-300ms | External model call per check |
| SelfCheckGPT (N=5 samples) | 5x generation | Async/batch only |
| LLM-judge approaches | 300ms-2s+ | Not suitable for synchronous path |

### Production Monitoring Stack Pattern (2026)

From [dasroot.net](https://dasroot.net/posts/2026/03/measuring-hallucination-rates-production-ai/) (Mar 2026):

1. **Continuous baseline**: White-box logprob perplexity on all responses (no extra cost)
2. **Sampled deep eval**: SelfCheckGPT or LLM-judge on % of traffic (async queue)
3. **Entity-level real-time**: Linear probes or HaluGate on factual traffic (gated by Sentinel)
4. **Leaderboard benchmarking**: Vectara HHEM-2.3 for model-level grounded summarization
5. **Observability**: Emit scores as OTel span attributes; alert on rolling hallucination rate thresholds

---

## 8. Toolkit Relevance

| Toolkit Component | Relationship |
|-------------------|-------------|
| `buildEvalPrompt()` at `src/lib/llm-as-judge.ts:803` | G-Eval prompt construction — HaluGate Explainer output could augment evaluation context |
| `normalizeWithLogprobs()` at `src/lib/llm-as-judge.ts:841` | Score normalization with logprob weighting — directly compatible with HALT-style confidence scores |
| `GEvalConfig` at `src/lib/llm-as-judge.ts:327` | Custom criteria — hallucination detection criteria can be expressed as G-Eval configs |
| `QAG faithfulness` in `src/lib/llm-as-judge.ts` | QAG-based faithfulness scoring — complementary to real-time detection (batch vs. streaming) |

HaluGate's HTTP header approach is the most natural integration point: detection results propagated via response headers can be captured as span attributes without modifying the evaluation pipeline.

---

## 9. Open Questions

- **Anthropic logprobs roadmap**: No public timeline for native logprob API support — significant gap for Claude-centric stacks
- **Cross-model HALT generalization**: HALT-L/HALT-Q trained on specific families; accuracy on other models not rigorously benchmarked
- **HaluGate detector model**: Stage 2 `hallucination_model` artifact not publicly confirmed beyond Sentinel
- **OTel hallucination semconv**: GenAI SIG has signaled intent but not published finalized `gen_ai.hallucination.*` attributes
- **Gemini 2.5 logprobs**: Silently removed with no explanation or restoration timeline
- **CoT probe tooling**: arXiv 2601.02170 lacks published code release as of March 2026
