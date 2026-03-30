# Research Directions — Briefs

**Version**: 1.2
**Date**: 2026-02-14
**Last Research Update**: 2026-02-14 (audit pass: all codebase references verified)
**Source**: [Roadmap](README.md) Future Research Directions, white paper Section 8
**Items**: R1-R12
**Status**: Exploratory — no implementation commitment

---

## S8.1 Emerging Standards

### R1: MCP Observability Conventions

**Context**: As the Model Context Protocol matures beyond the current 1:1 host-server model, standardized telemetry conventions for MCP tool calls will be needed. The toolkit itself is an MCP server with a 15-tool router — making it both a consumer and potential contributor to these conventions.

**Research Questions**:
- ~~Will the MCP specification define standard span attributes for tool invocation?~~ **RESOLVED (Feb 2026):** MCP spec (2025-11-25) does not define native OTel attributes, but **OTel has published MCP-specific semantic conventions** at [opentelemetry.io/docs/specs/semconv/gen-ai/mcp/](https://opentelemetry.io/docs/specs/semconv/gen-ai/mcp/). Span name format: `{mcp.method.name} {target}` (e.g., `tools/call get-weather` where target = `gen_ai.tool.name`). `mcp.method.name` is a Required span attribute with well-known values: `tools/call`, `resources/read`, `prompts/get`, etc. Added in semconv v1.39.0 (Jan 2026).
- ~~How should MCP tool calls be represented in OTel traces?~~ **RESOLVED:** MCP tool call spans are designed to be compatible with `gen_ai` `execute_tool` spans. FastMCP ships native OTel instrumentation, auto-generating traces for tool, prompt, resource, and resource template operations.
- Should tool-level rate limiting and circuit breaker events be standardized attributes?
- **NEW**: MCP Discussion #269 proposes adding OTel trace context to the MCP protocol itself. Issue #246 proposes including OTel trace identifiers in client-to-server messages. The C# SDK has initial OTel support. Community consensus leans toward instrumenting servers to emit OTLP rather than wrapping OTLP into the MCP protocol.

**Signals to Watch**:

| Signal | Source | Current State |
|--------|--------|---------------|
| MCP specification updates | [modelcontextprotocol.io](https://modelcontextprotocol.io) | **2025-11-25 release** (Tasks, OAuth 2.1, Streamable HTTP) |
| OTel GenAI SIG discussions on MCP | OTel GitHub / CNCF Slack | **MCP semconv published in v1.39.0** |
| MCP OTel trace context proposals | [Discussion #269](https://github.com/modelcontextprotocol/modelcontextprotocol/discussions/269), [Issue #246](https://github.com/modelcontextprotocol/modelcontextprotocol/issues/246) | **Active discussion** |
| FastMCP OTel instrumentation | [gofastmcp.com/servers/telemetry](https://gofastmcp.com/servers/telemetry) | **Shipping** |
| Langfuse/Phoenix MCP integrations | Platform changelogs | DeepEval has MCP evaluation support |

**Prerequisites for Commitment**: ~~MCP specification reaches v1.0 with defined telemetry hooks; or OTel GenAI SIG publishes MCP-specific semantic conventions.~~ **Prerequisite met:** OTel MCP semconv published.

**Status**: **Committed (v2.29)** — Adopted OTel MCP semconv v1.39.0 for all tool call spans. Implemented `mcpToolCall()` instrumentation middleware with:
- Span naming: `tools/call {toolName}` (SpanKind.SERVER)
- Required: `mcp.method.name`, `gen_ai.tool.name`, `error.type` (on failure)
- Recommended: `gen_ai.operation.name` (`execute_tool`), `network.transport` (`pipe`), `mcp.session.id` (when available)
- Opt-in: `gen_ai.tool.call.arguments`
- Metric: `mcp.server.operation.duration` histogram (seconds)
- Legacy dual-emit: `obs_toolkit.tool.name`, `obs_toolkit.tool.success` for backward compatibility
- Constants in `src/lib/observability/mcp-semconv-constants.ts`, middleware in `src/lib/observability/mcp-semconv.ts`
- Telemetry-driven hardening (4d3eb48): corrected `getMeter()` comment in metrics.ts, documented `pipe` transport assumption in mcp-semconv.ts
- Code review hardening (v2.29): `MCP_OPERATION_VALUES.EXECUTE_TOOL` constant (W3), `MCP_TRANSPORT_VALUES` + configurable `transport` in `McpToolCallOptions` (W4), `SESSION_DURATION` @todo for session lifecycle tracking (W2), startTime wall-clock latency documentation (S2), dual-emit rationale comments (S3)

**Remaining open questions**:
- Should tool-level rate limiting and circuit breaker events be standardized attributes?
- MCP Discussion #269: OTel trace context in MCP protocol (community monitoring)

**Expertise Required**: MCP protocol internals, OTel semantic convention process (OTEP submission, stability levels)

---

### R2: Agentic System Semantics (OTel Issue #2664)

**Context**: OTel Issue #2664 proposes comprehensive semantic conventions for agentic systems: tasks, artifacts, memory, and multi-step reasoning chains. This would standardize how agent workflows appear in trace visualizations across all observability platforms.

**Research Questions**:
- ~~What span attributes and events does the proposal define for agent tasks, tool use, and reasoning steps?~~ **RESOLVED (Feb 2026):** The proposal defines five core primitives: Tasks (`gen_ai.task.*`), Actions (tool calls, LLM queries, API requests), Teams (dynamic agent groups), Artifacts (inputs/outputs), and Memory (persistent context). Key attributes landed incrementally across semconv v1.31.0-v1.38.0:
  - `gen_ai.agent.id`, `gen_ai.agent.name`, `gen_ai.agent.description` (v1.31.0)
  - Operations: `create_agent` (v1.31.0), `invoke_agent` (v1.33.0), `execute_tool` (v1.37.0), `generate_content`
  - Span naming: `create_agent {name}` (CLIENT), `invoke_agent {name}` (CLIENT/INTERNAL), `execute_tool {name}`
  - Full tool call attribute set: `gen_ai.tool.call.arguments`, `gen_ai.tool.call.id`, `gen_ai.tool.call.result`, `gen_ai.tool.definitions` (v1.38.0), `gen_ai.tool.name`, `gen_ai.tool.type`
  - Source: [GenAI Agent Spans Specification](https://opentelemetry.io/docs/specs/semconv/gen-ai/gen-ai-agent-spans/)
- ~~How does the proposal handle multi-agent scenarios?~~ **PARTIALLY RESOLVED:** Teams and memory primitives remain in proposal stage (Issues #2664, #2665). Single-agent and tool execution conventions are landed; multi-agent coordination is not yet standardized.
- ~~What is the timeline to reach experimental stability?~~ **RESOLVED:** Agent attributes reached **Development (experimental)** in v1.31.0 with the initial AI Agent Semantic Convention (#1732, #1739), with enhancements through v1.38.0. Opt-in via `OTEL_SEMCONV_STABILITY_OPT_IN=gen_ai_latest_experimental`. No timeline announced for Stable promotion.

**Signals to Watch**:

| Signal | Source | Current State |
|--------|--------|---------------|
| OTel Issue #2664 status | [github.com/open-telemetry](https://github.com/open-telemetry/semantic-conventions/issues/2664) | **Open — concepts landing incrementally** |
| Agent spans specification | [OTel GenAI Agent Spans](https://opentelemetry.io/docs/specs/semconv/gen-ai/gen-ai-agent-spans/) | **Published (Development since v1.31.0)** |
| OTel GenAI SIG meeting notes | OTel community calendar | Active — cross-framework standardization work (IBM Bee, wxFlow, CrewAI, AutoGen, LangGraph) |
| Vendor adoption | Platform changelogs | **Datadog and Arize Phoenix both ship agent monitoring (June 2025)** |

**Prerequisites for Commitment**: ~~Proposal reaches OTEP stage with defined attribute names and stability guarantees; or two major vendors adopt draft conventions.~~ **Both prerequisites met.** Agent attributes are in the semconv (Development since v1.31.0, enhanced through v1.38.0). Datadog and Arize Phoenix have adopted agent-specific observability. **Ready for commitment.**

**Expertise Required**: OTel semantic convention process, distributed tracing for agentic systems, span hierarchy design

**Relevance to Toolkit**: `src/lib/agent-as-judge.ts` already models agent actions (`AgentAction` at line 108), evaluation plans (`EvaluationPlan` at line 183), and trajectory analysis (`analyzeTrajectory()` at line 602). These map directly to the `invoke_agent`/`execute_tool` span conventions. The `gen_ai.agent.*` attributes provide a standardized naming scheme for the toolkit's existing agent evaluation data.

---

### R3: Multi-Agent Coordination Observability

**Context**: Agent-to-agent communication patterns (delegation, consensus, negotiation) introduce new observability requirements beyond single-agent tool use. As multi-agent orchestration frameworks mature, standardized telemetry for coordination events becomes necessary.

**Research Questions**:
- What coordination patterns exist (hierarchical delegation, peer consensus, market-based negotiation)?
- How should inter-agent messages be represented in traces — as link spans, events on a shared parent, or separate trace contexts?
- What metrics matter for coordination health (handoff latency, consensus convergence time, delegation depth)?

**Signals to Watch**:

| Signal | Source | Current State |
|--------|--------|---------------|
| Multi-agent framework telemetry | AutoGen, CrewAI, LangGraph | **All three now ship OTel instrumentation** |
| AG2 OTel tracing | [docs.ag2.ai](https://docs.ag2.ai/latest/docs/blog/2026/02/08/AG2-OpenTelemetry-Tracing/) | **Feb 2026: Built-in OTel with W3C Trace Context propagation across A2A calls** |
| CrewAI OTel instrumentation | [PyPI](https://pypi.org/project/opentelemetry-instrumentation-crewai/) | **v0.51.1 (Jan 2026): OTel + OpenInference instrumentation** |
| LangGraph/LangChain OTel | [LangSmith blog](https://blog.langchain.com/end-to-end-opentelemetry-langsmith/) | **LangSmith SDK v0.4.25+ with native end-to-end OTel support** |
| OTel messaging semconv evolution | OTel spec | Kafka/RabbitMQ focus; agent coordination not yet addressed |
| Agent-to-agent protocol proposals | Research papers, framework RFCs | **AG2 implements W3C Trace Context propagation across A2A HTTP calls** |

**Prerequisites for Commitment**: ~~One multi-agent framework ships OTel-native coordination telemetry.~~ **Prerequisite met.** AutoGen/AG2, CrewAI, and LangGraph all ship OTel-native telemetry. AG2 specifically implements W3C Trace Context propagation across agent-to-agent calls. **Ready for commitment.**

**Expertise Required**: Multi-agent system design, distributed coordination protocols, OTel trace linking

**Relevance to Toolkit**: `MultiAgentEvaluation` at `src/lib/quality/quality-multi-agent.ts:68` and `HandoffEvaluation` at `src/lib/quality/quality-multi-agent.ts:31` already model evaluation of multi-agent outputs. `ConsensusConfig` at `src/lib/agent-as-judge.ts:195` models judge consensus. The availability of OTel-instrumented frameworks means the toolkit can now consume real multi-agent traces (not just mock data) for evaluation and visualization.

---

## S8.2 Quality Measurement Evolution

### R4: Real-Time Hallucination Detection

**Context**: Current hallucination detection in the toolkit is batch-oriented (G-Eval scores computed post-hoc). Real-time detection would flag hallucinations during streaming generation, enabling early intervention before users see incorrect content.

**Research Questions**:
- Can token-level confidence scores (logprobs) serve as streaming hallucination signals?
- What is the latency budget for real-time detection without degrading user experience?
- How do faithfulness scores (NLI-based, G-Eval) correlate with token-level uncertainty?

**Signals to Watch**:

| Signal | Source | Current State |
|--------|--------|---------------|
| Provider logprob APIs | Claude, OpenAI, Gemini | Available but not standardized |
| **HaluGate (vLLM)** | [blog.vllm.ai/2025/12/14/halugate.html](https://blog.vllm.ai/2025/12/14/halugate.html) | **Production-ready: 3-stage pipeline (Sentinel → Detector → Explainer), 72.2% efficiency gain, HTTP header propagation** |
| **Streaming CoT hallucination detection** | [arXiv 2601.02170](https://arxiv.org/abs/2601.02170) (Jan 2026) | **87%+ accuracy at step/prefix levels; zero additional inference cost** |
| **HALT: Log-prob time series** | [arXiv 2602.02888](https://arxiv.org/html/2602.02888) (Feb 2026) | **Lightweight sequence models classify hallucinations from log-prob sequences** |
| Real-time entity detection | [arXiv 2509.03531](https://arxiv.org/abs/2509.03531) | **Token-level hallucinated entity detection, scales to 70B params** |
| Real-time safety filters | Platform safety APIs | Production but opaque |

**Prerequisites for Commitment**: ~~Provider logprob APIs stabilize with consistent semantics; or a lightweight streaming NLI model becomes available.~~ **Partially met.** HaluGate provides a production-ready 3-stage pipeline with HTTP header propagation for downstream policy enforcement. The Sentinel model is available on HuggingFace (`llm-semantic-router/halugate-sentinel`). However, it runs server-side (vLLM); adapting for client-side API consumers is additional work. The streaming CoT approach (arXiv 2601.02170) achieves 87%+ accuracy with zero inference overhead but requires access to internal model states.

**Expertise Required**: NLP evaluation methods, streaming inference pipelines, token-level uncertainty quantification

**Relevance to Toolkit**: `src/lib/llm-as-judge.ts` implements G-Eval (`buildEvalPrompt()` at line 803), QAG-based faithfulness, and score normalization (`normalizeWithLogprobs()` at line 841). HaluGate's HTTP header approach could be integrated as an observability signal — detection results propagated via response headers can be captured as span attributes without modifying the evaluation pipeline.

---

### R5: Automated Regression Detection

**Context**: When LLM providers update models, quality can silently degrade. Automated regression detection would compare evaluation scores across model versions and alert when statistically significant drops occur.

**Research Questions**:
- What statistical tests are appropriate for detecting quality regressions in LLM output scores (t-test, Mann-Whitney U, CUSUM)?
- How many evaluation samples are needed for sufficient statistical power?
- How should model version changes be detected from telemetry (span attribute changes, manual annotation)?

**Signals to Watch**:

| Signal | Source | Current State |
|--------|--------|---------------|
| Model version tracking in OTel | GenAI semconv `gen_ai.response.model` | Available |
| Statistical process control for ML | ML monitoring literature | Established methods |
| Langfuse/Phoenix regression features | Platform changelogs | Langfuse has experiment comparison |

**Prerequisites for Commitment**: Sufficient evaluation volume (100+ evaluations per model version) is being collected in production; or users report quality regressions that would have been caught by automated detection.

**Expertise Required**: Statistical process control, hypothesis testing for evaluation data, time-series anomaly detection

**Relevance to Toolkit**: `computeTrend()` at `src/lib/quality/quality-trends.ts:284` already computes trend direction and stability for metrics. Regression detection would extend this with cross-version comparison and statistical significance testing.

---

### R6: Domain-Specific Evaluators

**Context**: The toolkit's current evaluators (G-Eval, QAG) are domain-agnostic. Specialized evaluators for code generation, medical content, legal analysis, etc. would provide more accurate quality assessment for domain-specific applications.

**Research Questions**:
- What domain-specific evaluation criteria exist (code: syntax correctness, test pass rate; medical: clinical accuracy, evidence grading; legal: citation accuracy, jurisdictional relevance)?
- Can domain evaluators be implemented as G-Eval criteria customizations, or do they need fundamentally different evaluation approaches?
- How do domain-specific evaluators compose with the existing `panelEvaluation()` multi-evaluator pattern?

**Signals to Watch**:

| Signal | Source | Current State |
|--------|--------|---------------|
| Domain-specific LLM benchmarks | HumanEval, MedQA, LegalBench | Active research |
| DeepEval domain evaluators | DeepEval changelog | **Agent-specific evaluators shipped: Task Completion, Tool Correctness, Tool Invocation Order. DAGMetric for custom evaluation logic. MCP interaction evaluation support.** |
| DeepEval ArenaGEval | [deepeval.com/changelog](https://deepeval.com/changelog/changelog-2025) | **First comparative evaluation metric (compare test cases, choose best performer)** |
| Opik span-level metrics | [comet.com/site/products/opik/](https://www.comet.com/site/products/opik/) | **LLM-as-Judge and code-based metrics attached to individual call spans** |
| Industry-specific compliance | HIPAA, SOX, etc. | Sector-specific |

**Prerequisites for Commitment**: ~~A specific domain use case emerges from toolkit users; or a domain evaluator framework that integrates with G-Eval scoring becomes available.~~ **Partially met.** DeepEval's DAGMetric (deterministic decision trees for evaluation using LLM-as-a-judge) provides a framework for building domain-specific evaluation logic as directed acyclic graphs. This could integrate with the toolkit's existing `GEvalConfig` pattern. Agent-specific evaluators (Task Completion, Tool Correctness) represent the first domain-specialized metrics.

**Expertise Required**: Domain expertise (code, medical, legal), evaluation methodology design, `GEvalConfig` extension patterns

**Relevance to Toolkit**: `GEvalConfig` at `src/lib/llm-as-judge.ts:327` accepts custom criteria and evaluation steps. `TestCase` at line 308 is extensible with domain-specific fields. Domain evaluators would be implemented as specialized configs, not new code paths.

---

## S8.3 Cost Optimization

### R7: Token Budget Observability

**Context**: Real-time context window utilization tracking would help users understand how much of their context budget is consumed by system prompts, conversation history, tool results, and cached content.

**Research Questions**:
- Can context window utilization be computed from span attributes (`gen_ai.usage.input_tokens` vs model context limit)?
- How should context budget allocation be visualized (stacked bar by source: system, history, tools)?
- What alerting thresholds make sense for context window utilization (80% warning, 95% critical)?

**Signals to Watch**:

| Signal | Source | Current State |
|--------|--------|---------------|
| Context window sizes | Provider model cards | Growing (200K+ Claude) |
| Token counting in OTel | GenAI semconv | `gen_ai.usage.*` available |
| Context management libraries | LangChain, LlamaIndex | Application-level |

**Prerequisites for Commitment**: Users report context window overflow issues; or cost optimization becomes a primary concern (cost per session exceeds threshold).

**Expertise Required**: LLM context window mechanics, token budget allocation strategies

**Relevance to Toolkit**: `calculateCost()` at `src/lib/cost-estimation.ts:214` already processes token counts. Budget tracking via `createBudgetTracker()` at line 375 monitors spend. Context utilization would add a complementary resource utilization view.

---

### R8: Model Routing Telemetry

**Context**: Systems that route requests to different models based on complexity, cost, or latency need visibility into routing decisions and their cost/quality tradeoffs.

**Research Questions**:
- What attributes describe a routing decision (candidate models, selection criteria, confidence in selection)?
- How should routing effectiveness be measured (cost saved vs quality lost)?
- Can routing decisions be represented as OTel events on the parent span?

**Signals to Watch**:

| Signal | Source | Current State |
|--------|--------|---------------|
| Model routing frameworks | Martian, Not Diamond, OpenRouter | **Active ecosystem; Not Diamond powers OpenRouter's Auto Router** |
| **vLLM Semantic Router** | [blog.vllm.ai/2026/01/05/vllm-sr-iris.html](https://blog.vllm.ai/2026/01/05/vllm-sr-iris.html) | **v0.1 "Iris" (Jan 2026): System-level Mixture-of-Models routing with native OTel distributed tracing** |
| OTel GenAI model selection | GenAI semconv | **No dedicated routing conventions, but `gen_ai.request.model` vs `gen_ai.response.model` captures routing audit trail** |
| Cost-quality Pareto optimization | Research papers | Active area |

**Prerequisites for Commitment**: ~~The toolkit or its users adopt model routing; or OTel GenAI semconv define routing-specific attributes.~~ **Partially met.** vLLM Semantic Router ships native OTel tracing for routing decisions (classification, caching, routing, backend interactions) but uses its own attribute schema, not yet standardized. The existing `gen_ai.request.model`/`gen_ai.response.model` pair provides a basic routing audit trail.

**Expertise Required**: LLM inference economics, multi-model orchestration, decision telemetry

**Relevance to Toolkit**: `MODEL_PRICING` at `src/lib/constants.ts:362` provides per-model pricing. A routing telemetry extension would correlate model selection with cost and quality outcomes.

---

### R9: Caching Effectiveness Metrics

**Context**: Semantic caching (reusing responses for semantically similar prompts) and KV cache optimization can dramatically reduce costs. Observing cache effectiveness requires new metrics.

**Research Questions**:
- What cache metrics matter? Hit rate, semantic similarity threshold, cost savings, staleness risk.
- How should semantic cache hits be distinguished from exact cache hits in telemetry?
- Can caching effectiveness be inferred from existing span attributes (identical responses, low TTFT)?

**Signals to Watch**:

| Signal | Source | Current State |
|--------|--------|---------------|
| Provider prompt caching | Claude prompt caching, GPT caching | **Major cost factor: Anthropic 90% cache read savings; OpenAI 50-90% depending on model family** |
| Provider pricing (Feb 2026) | Platform pricing pages | **Claude: cache read = 90% off ($0.50/M for Opus 4.5 vs $5.00). OpenAI: GPT-4.1 cache = 75% off ($0.50/M vs $2.00). o3 cache = 75% off. o3-mini: $1.10/$4.40.** |
| OTel cache attributes | GenAI semconv | **No `gen_ai.*.cache*` attributes standardized. Caching remains provider-specific (Anthropic `cache_creation_input_tokens`/`cache_read_input_tokens`, OpenAI `cached_tokens`).** |
| Semantic caching libraries | GPTCache, Zep | Maturing |
| **Maxim AI Bifrost** | [getmaxim.ai](https://www.getmaxim.ai/) | **LLM gateway with semantic caching (up to 30% cost reduction), automatic failover, load balancing** |

**Prerequisites for Commitment**: ~~Provider prompt caching becomes a significant cost factor.~~ **Prerequisite met.** Cache read savings of 75-90% make caching a first-class cost optimization concern. The lack of standardized OTel cache attributes means the toolkit would need provider-specific attribute mapping.

**Expertise Required**: Semantic caching systems, cache hit rate analysis, provider caching APIs

**Relevance to Toolkit**: The toolkit's own LRU cache (`src/lib/cache.ts`, 5 instances x 100 entries) already tracks `obs_toolkit.cache.hits/misses/evictions`. LLM-level caching is a different layer but the same metric patterns apply.

---

## S8.4 Privacy and Compliance

### R10: Content Redaction Pipelines

**Context**: Telemetry containing user prompts and LLM responses may include PII, proprietary data, or sensitive content. OTel Collector processors can redact content before it reaches storage, but no standardized redaction pipeline exists for LLM telemetry.

**Research Questions**:
- What OTel Collector processors exist for content redaction (transform processor, attributes processor)?
- Can redaction be applied selectively (redact prompt content but preserve token counts and timing)?
- How do redaction rules compose with the toolkit's error sanitization patterns?

**Signals to Watch**:

| Signal | Source | Current State |
|--------|--------|---------------|
| OTel Collector redaction processor | [OTel Collector contrib](https://github.com/open-telemetry/opentelemetry-collector-contrib/tree/main/processor/redactionprocessor) | **Available: allowlist-based, regex value masking, can target `gen_ai.*` attributes** |
| **LLM-specific PII redaction processor** | [IJC paper (Dec 2025)](https://ijcjournal.org/InternationalJournalOfComputer/article/view/2458) | **"Safe Observability" — custom OTel Collector processor with hybrid regex + NER detection via gRPC microservice** |
| PII detection libraries | Presidio, AWS Comprehend | Mature |
| **OTel sensitive data handling** | [opentelemetry.io/docs/security/handling-sensitive-data/](https://opentelemetry.io/docs/security/handling-sensitive-data/) | **Updated with GenAI considerations** |
| Practical guides | OneUptime (Nov 2025, Feb 2026), Dash0, Better Stack | **Multiple walkthroughs for configuring redaction in OTel pipelines** |

**Prerequisites for Commitment**: ~~A deployment context requires PII redaction in telemetry; or regulatory compliance mandates content redaction.~~ **Approaching:** EU AI Act enforcement in August 2026 (see R11) will require data handling compliance. No official LLM-specific redaction processor exists in OTel contrib yet, but the IJC paper provides a viable custom build approach (Go, OTel Collector Builder, NER microservice).

**Expertise Required**: OTel Collector processor configuration, PII detection/redaction techniques, data loss prevention, Go (for custom processor builds via OTel Collector Builder)

**Relevance to Toolkit**: Error sanitization at `src/lib/error-sanitizer.ts` (267 lines) already strips file paths, stack traces, and internal module names. Content redaction extends this pattern to span attribute values (prompt/response content). The existing redaction processor can be configured for `gen_ai.*` attributes but lacks LLM-aware content understanding without an NER microservice. EU AI Act analysis in `docs/interface/llm-explainability-research.md` (Section 6) documents regulatory context.

---

### R11: Audit Trail Requirements — **Done (v2.29)**

**Context**: Regulatory frameworks (EU AI Act, NIST AI RMF) increasingly require audit trails for AI decision-making. This means traceable links from AI outputs back to the model version, prompt, evaluation scores, and human oversight decisions.

**Status**: **Done (v2.29, 26ebcc6).** The toolkit now provides:
- `AuditRecord` type with SHA-256 hash chain for tamper evidence (Article 13)
- `obs_audit_trail` MCP tool with query, report, and verify actions
- 180-day minimum retention enforcement per Article 19
- Compliance report generation covering Articles 12, 13, and 19
- Implementation: `src/lib/audit/audit-record.ts`, `retention-guard.ts`, `compliance-report.ts`, `src/tools/audit-trail.ts`

**Research Questions** (resolved):
- What constitutes a compliant audit trail under EU AI Act Article 13? → SHA-256 hash chain linking traces, evaluations, and verifications with tamper detection
- How should audit records reference OTel traces and evaluations? → Separate `AuditRecord` type with `traceId`, `evaluationName`, `sessionId` correlation fields
- What retention requirements apply? → Article 19 mandates 6-month (180-day) minimum; enforced via `getEffectiveRetentionDays()` floor

**Signals to Watch**:

| Signal | Source | Current State |
|--------|--------|---------------|
| EU AI Act Article 13/50 implementation | European Commission | **Confirmed: August 2, 2026 enforcement** |
| **Article 50 Code of Practice** | [EC Code of Practice page](https://digital-strategy.ec.europa.eu/en/policies/code-practice-ai-generated-content) | **First draft published Dec 17, 2025. Second draft ~mid-March 2026. Final expected June 2026. Voluntary but serves as compliance demonstration.** |
| **Article 13 record-keeping** | [artificialintelligenceact.eu/article/13/](https://artificialintelligenceact.eu/article/13/) | **Full requirements: risk management, data governance, technical documentation, record-keeping, transparency, human oversight** |
| NIST AI RMF GOVERN/MEASURE functions | NIST | Published (v1.0) |
| AI audit trail standards | ISO/IEC | In development |

**Prerequisites**: All met. Code of Practice first draft published Dec 2025. Article 13 record-keeping requirements directly map to toolkit audit trail capabilities.

**Expertise Required**: Regulatory compliance (EU AI Act Articles 13, 50), audit trail design, immutable record storage

**Relevance to Toolkit**: `AuditRecord` with hash chain provides tamper-evident audit trail. `obs_audit_trail` tool generates compliance reports. Retention guard enforces Article 19 minimum. Human verification events (`verification-events.ts`) cover Article 12(3)(d). EU AI Act analysis in `docs/reviews/eu-ai-act-observability-requirements.md` documents full regulatory mapping.

---

### R12: Differential Privacy for Aggregated Telemetry

**Context**: When sharing aggregated quality metrics (dashboards, reports), differential privacy techniques can protect individual user interactions from being reconstructed, even if the aggregates are public.

**Research Questions**:
- What differential privacy mechanisms are appropriate for LLM evaluation metrics (Laplace noise, Gaussian mechanism)?
- What epsilon values balance utility and privacy for quality score aggregations?
- Can differential privacy be applied at the OTel Collector level or must it be in the application?

**Signals to Watch**:

| Signal | Source | Current State |
|--------|--------|---------------|
| Differential privacy libraries | Google DP, OpenDP | Mature |
| Privacy-preserving ML monitoring | Research papers | Academic |
| GDPR aggregation requirements | DPA guidance | Evolving |

**Prerequisites for Commitment**: Aggregated telemetry is shared externally (public dashboards, benchmarks); or a privacy impact assessment identifies reconstruction risk.

**Expertise Required**: Differential privacy theory, privacy-preserving aggregation, statistical utility analysis

**Relevance to Toolkit**: `computeDashboardSummary()` in `src/lib/quality-metrics.ts` aggregates evaluation scores with percentiles (p50, p95, p99). Differential privacy would add noise to these aggregations before export. The `obs_export_*` tools would be the natural injection point.
