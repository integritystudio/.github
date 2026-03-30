# R7: Token Budget Observability

**Version**: 1.0
**Date**: 2026-03-02
**Source**: [otel-v3-research-directions.md](otel-v3-research-directions.md) R7, white paper Section 8.3
**Status**: Done (v3.0, v2.32) — `obs_token_budget` MCP tool shipped
**Verification Date**: 2026-03-02 — Anthropic, OpenAI, Gemini docs; OTel semconv v1.40.0; Helicone, Portkey, Datadog verified

---

## 1. Introduction

### 1.1 Context Window Management Landscape (2025-2026)

Context windows have grown ~1000x in 4 years (4K tokens in 2022 to 2M tokens in 2026), but effective utilization remains a critical observability gap:

1. **Quantity vs. quality** — Models claiming 200K context typically become unreliable around 130K. Chroma Research (2025) demonstrated that performance degrades even with perfect retrieval.
2. **Prompt caching as cost lever** — 90% cost reduction on cache reads (Anthropic) has reshaped context budget architecture, making static prefix optimization a first-class concern.
3. **OTel spec gaps** — `gen_ai.usage.input_tokens` exists but no context window capacity, utilization percentage, or per-component token breakdown attributes are standardized.

### 1.2 Scope

This document covers: context window sizes, OTel token conventions, budget allocation patterns, monitoring tools, cost implications, overflow handling, caching mechanics, and research on context quality degradation.

---

## 2. Context Window Sizes

### 2.1 Current Models (Q1 2026)

| Provider | Model | Context Window | Output Limit | Notes |
|----------|-------|----------------|-------------|-------|
| Anthropic | Claude Opus 4.6 | 200K (1M beta) | — | Beta: `context-1m-2025-08-07` header; Tier 4+ |
| Anthropic | Claude Sonnet 4.6 | 200K (1M beta) | — | Same beta access |
| Anthropic | Claude Sonnet 4.5 | 200K (1M beta) | 64K | |
| Anthropic | Claude Haiku 4.5 | 200K | 64K | No 1M beta |
| OpenAI | GPT-4.1 | ~1M | — | API only; ChatGPT UI capped at 32K |
| OpenAI | o3 | 200K | 100K | |
| Google | Gemini 2.0 Flash | 1M | 8,192 | |
| Google | Gemini 2.0 Pro | 2M | — | Largest generally available |

Sources: [Claude Context Windows](https://platform.claude.com/docs/en/build-with-claude/context-windows), [GPT-4.1](https://openai.com/index/gpt-4-1/), [Gemini 2.0 Flash](https://docs.cloud.google.com/vertex-ai/generative-ai/docs/models/gemini/2-0-flash)

### 2.2 Growth Timeline

| Year | Mainstream Max | Milestone |
|------|---------------|-----------|
| 2022 | 4K | GPT-3.5/ChatGPT launch |
| 2023 H1 | 32K | GPT-4 8K/32K |
| 2023 H2 | 200K | Claude 2.1; GPT-4 Turbo 128K |
| 2024 | 1M | Gemini 1.5 Pro |
| 2025-2026 | 2M | Gemini 2.0 Pro GA; GPT-4.1 1M |

Source: [Epoch AI — Context Windows](https://epoch.ai/data-insights/context-windows)

### 2.3 Effective Utilization

- **RULER benchmark**: Effective utilization at 50-65% of advertised window
- **Degradation**: Non-linear — sharp drop-offs, not gradual decline
- **Context awareness**: Claude Sonnet 4.5/4.6 and Haiku 4.5 receive real-time budget updates via system messages

---

## 3. OTel GenAI Semantic Conventions

### 3.1 Token Attributes (semconv v1.40.0, Development)

| Attribute | Type | Description |
|-----------|------|-------------|
| `gen_ai.usage.input_tokens` | int | All input tokens including cached |
| `gen_ai.usage.output_tokens` | int | Response/completion tokens |
| `gen_ai.usage.cache_creation.input_tokens` | int | Tokens written to provider-managed cache |
| `gen_ai.usage.cache_read.input_tokens` | int | Tokens served from cache |
| `gen_ai.request.max_tokens` | int | The `max_tokens` parameter |
| `gen_ai.token.type` | enum | `"input"` or `"output"` — metric dimension |

**Deprecated**: `gen_ai.usage.prompt_tokens` -> `input_tokens`, `gen_ai.usage.completion_tokens` -> `output_tokens`.

### 3.2 Token Metric

| Metric | Instrument | Unit |
|--------|-----------|------|
| `gen_ai.client.token.usage` | Histogram | `{token}` |

Bucket boundaries: `[1, 4, 16, 64, 256, 1024, 4096, 16384, 65536, 262144, 1048576, 4194304, 16777216, 67108864]`

Source: [OTel GenAI Metrics](https://opentelemetry.io/docs/specs/semconv/gen-ai/gen-ai-metrics/)

### 3.3 What is NOT in the Spec

| Gap | Impact |
|-----|--------|
| No `gen_ai.usage.context_window_size` | Cannot compute utilization percentage |
| No `gen_ai.usage.context_utilization_pct` | Must derive manually |
| No per-component breakdown | System/history/RAG/tool tokens not distinguished |
| No per-turn rolling total | Cumulative tracking requires custom instrumentation |
| No cache hit rate metric | Derivable but not named |
| No position metadata | Cannot track where in context window content sits |

---

## 4. Context Budget Allocation

### 4.1 Typical Composition

Context windows are consumed by competing components in priority order:

| Component | Typical Range | Characteristics |
|-----------|--------------|-----------------|
| System prompt / instructions | 2K-20K | Repeated every request; static prefix |
| Tool definitions / schemas | 5K-15K | JSON schemas for all available tools |
| Retrieved context (RAG) | 10K-50K+ | Variable; primary source of "RAG bloat" |
| Conversation history | Grows linearly | Dominant driver of context overflow |
| Tool call results | 1K-10K+ per call | Raw JSON; accumulates rapidly |
| Reserved output headroom | `max_tokens` | Must subtract from available input |

### 4.2 Allocation Strategies

| Strategy | Mechanism | Trade-off |
|----------|-----------|-----------|
| **Fixed** | Static slots per component type | Simple; inflexible |
| **Dynamic** | Adjust per-query (factual -> more RAG, reasoning -> more history) | Complex; higher quality |
| **Priority-based** | Must-retain vs. droppable content tagging | Requires metadata |
| **Trigger-based compaction** | Compact at 70-80% capacity | Anthropic-recommended |

### 4.3 Custom Instrumentation Required

The OTel spec emits total `gen_ai.usage.input_tokens` but provides no breakdown. Custom attributes needed:

```
context.system_prompt_tokens
context.history_tokens
context.rag_tokens
context.tool_result_tokens
context.utilization_pct = input_tokens / context_window_size * 100
```

Sources: [Context Management — Maxim](https://www.getmaxim.ai/articles/context-window-management-strategies-for-long-context-ai-agents-and-chatbots/), [Context Engineering — Weaviate](https://weaviate.io/blog/context-engineering)

---

## 5. Token Budget Monitoring Tools

### 5.1 Platform Comparison

| Platform | Token Tracking | Budget Controls | Context Viz | OTel |
|----------|---------------|-----------------|-------------|------|
| **Helicone** | Per-request input/output, cost | Threshold alerts (50%/80%/95%) | 1M+ support, histograms | Partial |
| **Portkey** | 40+ attributes/request | Hard/soft limits; per-key/team/org | Full trace lifecycle | Yes |
| **Datadog LLM Obs** | `gen_ai.usage.*`, cost | Dashboard alerting | Cross-layer APM correlation | Native v1.37+ |
| **LangSmith** | Per-step token counts | Budget via evaluators | Visual trace view | Partial |
| **Arize/Phoenix** | Token usage per span | Drift/threshold alerts | Span-level breakdown | Yes |

### 5.2 Provider Token Counting APIs

| Provider | Method | Notes |
|----------|--------|-------|
| **Anthropic** | `POST /v1/messages/count_tokens` | Free endpoint; exact count before sending; supports tools, images, docs |
| **OpenAI** | `/v1/usage` endpoint | Per-model aggregate; dashboard at platform.openai.com |
| **Google** | `countTokens` API | Per-request |

Source: [Claude Token Counting API](https://platform.claude.com/docs/en/build-with-claude/token-counting)

---

## 6. Cost Implications

### 6.1 Token Pricing (Q1 2026)

| Model | Input ($/M) | Output ($/M) | Cache Read ($/M) | Cache Write ($/M) |
|-------|-------------|-------------|-----------------|-------------------|
| Claude Opus 4.6 | ~$15 | ~$75 | ~$1.50 (10%) | ~$18.75 (125%) |
| Claude Sonnet 4.5 | $5 | $25 | $0.50 (10%) | $6.25 (125%) |
| GPT-4o | $5 | $20 | $2.50 (50%) | automatic |
| Gemini 2.0 Flash | $0.08-$0.30 | $0.30 | varies | — |

**Long-context premium**: Claude requests exceeding 200K tokens charged at 2x input / 1.5x output.

### 6.2 Cost Optimization Levers

| Lever | Impact | Mechanism |
|-------|--------|-----------|
| **Prompt caching** | 90% cost reduction (Anthropic) | Static prefix > cache threshold |
| **Context truncation** | Reclaims thousands of tokens/turn | Drop old tool results |
| **Output capping** | Bounds output cost directly | `max_tokens` parameter |
| **Model routing** | Reduce per-request spend | Route low-complexity to cheaper models |
| **Input compression** | 5-20x compression | Summarize retrieved documents |

### 6.3 Budget Monitoring Patterns

- **Per-request**: `total_cost = (input * price) + (output * price) - (cache_read * savings)`
- **Per-session**: Rolling cumulative cost; alert at budget thresholds
- **Per-user/team**: Virtual key scoping (Portkey, Helicone)
- **Enforcement**: Hard-cap (block) vs. soft-cap (alert + route to cheaper model)

---

## 7. Context Overflow and Truncation

### 7.1 Provider Overflow Behavior

| Provider | Behavior |
|----------|----------|
| **Claude (Sonnet 3.7+)** | Returns validation error when prompt + max_tokens > window. Does NOT silently truncate. |
| **OpenAI GPT-4.1** | Strict enforcement; client-side truncation required |
| **Legacy (Claude 3, older OpenAI)** | Silent FIFO truncation — least observable failure mode |

### 7.2 Truncation Strategies

| Strategy | Mechanism | Quality |
|----------|-----------|---------|
| **FIFO** | Drop oldest messages first | Simple; loses continuity |
| **Priority retention** | Tag must-keep vs. droppable | Requires metadata |
| **Semantic truncation** | Embed all turns; drop lowest similarity | Expensive; high quality |
| **Hierarchical summarization** | Summarize old batches; keep recent verbatim | Optimum: summarize batches of 21, keep 10 |
| **Tool result clearing** | Strip `tool_result` blocks from history | Claude-specific |
| **Sliding window** | Fixed buffer advancing forward | Simple; loses early context |

### 7.3 Detection and Alerting

- Pre-flight check via token counting API before sending
- Track `context_utilization_pct`; alert at 70%, 80%, 95%
- Monitor `headroom = context_window_size - (input_tokens + max_tokens)`; negative = overflow
- AutoGen roadmap: [GitHub Issue #156](https://github.com/microsoft/autogen/issues/156)

---

## 8. Prompt Caching

### 8.1 Anthropic Prompt Caching

| Property | Detail |
|----------|--------|
| **Mechanism** | KV-cache for previously processed tokens; matching prefix skips recomputation |
| **Marking** | `cache_control: {"type": "ephemeral"}` on content blocks |
| **Hierarchy** | Cache breakpoints: tools -> system -> messages |
| **TTL (default)** | 5 minutes; refreshed free on each use |
| **TTL (extended)** | 1 hour via `"ttl": "1h"` |
| **Cache write** | 1.25x (5 min) or 2.0x (1 hour) of base input price |
| **Cache read** | 0.1x (90% discount) |
| **Rate limits** | Cache reads do NOT count against ITPM for Sonnet 3.7+ |
| **Invalidation** | Any content change, image/tool/tool_choice change |

Source: [Claude Prompt Caching](https://platform.claude.com/docs/en/build-with-claude/prompt-caching)

### 8.2 OpenAI Prompt Caching

| Property | Detail |
|----------|--------|
| **Mechanism** | Automatic prefix caching — no API changes required |
| **Minimum** | 1,024 tokens |
| **Granularity** | 128-token block matching |
| **Pricing** | 50% of input price for cached reads |
| **Routing** | Routes to servers with recent matching prompts |

Source: [OpenAI Prompt Caching](https://platform.openai.com/docs/guides/prompt-caching)

### 8.3 Cache Budget Planning

- Static prefix (system prompt + tool defs) should be maximized and placed first
- Dynamic content (user messages, RAG) appended after cache breakpoint
- Anthropic 100K system prompt: 25% extra on first request, 90% less on subsequent; break-even at ~1.4 reads
- Monitor `cache_creation_tokens` and `cache_read_tokens` separately for caching effectiveness

---

## 9. Research on Context Quality

### 9.1 "Lost in the Middle" (Liu et al., TACL 2024)

- **Paper**: [arXiv:2307.03172](https://arxiv.org/abs/2307.03172)
- Performance highest when relevant information at beginning or end; degrades for middle positions
- Root cause: RoPE introduces long-term decay — beginning/end tokens have stronger positional signal
- **Observability implication**: Token count alone does not indicate where critical information sits

### 9.2 Context Rot (Chroma Research, 2025)

- **Source**: [research.trychroma.com/context-rot](https://research.trychroma.com/context-rot)
- 18 models tested (GPT-4.1, Claude 4, Gemini 2.5, Qwen3)
- Even with perfect retrieval (100% exact token match), performance degrades as context grows
- Distractors compound: semantically confusable content causes steeper drops
- Shuffled haystack structure worsens degradation vs. coherent structure

### 9.3 ACON: Context Compression for Long-Horizon Agents

- **Paper**: [arXiv:2510.00615](https://arxiv.org/html/2510.00615v2)
- Adaptive compression reduced peak memory by 26-54% while maintaining task performance
- Unified framework: summarization + key-value masking + observation abstraction

### 9.4 Context Length Beyond Retrieval (ACL 2025)

- **Paper**: [arXiv:2510.05381](https://arxiv.org/html/2510.05381v1)
- Context length itself is a confounding variable even when retrieval recall is perfect

---

## 10. Toolkit Relevance

| Toolkit Component | Relationship |
|-------------------|-------------|
| `calculateCost()` at `src/lib/cost-estimation.ts:214` | Processes token counts — budget tracking via context utilization adds complementary view |
| `createBudgetTracker()` at `src/lib/cost-estimation.ts:375` | Monitors spend — extends naturally to context utilization budgets |
| `MODEL_PRICING` at `src/lib/constants.ts:362` | Per-model pricing — cache tier pricing needed for accurate budget tracking |
| `obs_estimate_cost` tool | Cost estimation — context utilization percentage could augment cost predictions |

Context utilization observability would complement existing cost tracking by adding a resource utilization dimension: how much of the available context window is consumed, and by which components.

---

## 11. Open Questions

- **No standardized context utilization metric**: `gen_ai.usage.input_tokens` exists but no `gen_ai.context.window_size` or utilization percentage — open gap for semconv proposal
- **Per-component breakdown**: No standard way to distinguish system/history/RAG/tool tokens within total
- **Position metadata**: No standard for recording where retrieved content sits in context (relevant to "lost in the middle")
- **Cache hit rate**: Derivable from `cache_read.input_tokens / input_tokens` but not a named metric
- **Context quality vs. quantity**: No telemetry schema captures distractor density, semantic relevance, or position distribution
- **1M token cost modeling**: Claude's 2x input / 1.5x output premium over 200K not surfaced in standard observability tooling
