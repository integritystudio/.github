# OTel GenAI Attribute Reference

**Semconv**: v1.40.0 (all GenAI conventions are Development stability)
**Toolkit**: v2.28 | **Date**: 2026-03-01
**Source**: [appendix-deep-dives.md](../changelog/2.28/appendix-deep-dives.md) A1

---

## Stability and Opt-In

All GenAI semantic conventions are **Development** stability as of v1.40.0 — not covered by OTel stability guarantees. To opt in to the latest experimental GenAI attributes:

```
OTEL_SEMCONV_STABILITY_OPT_IN=gen_ai_latest_experimental
```

---

## Coverage Summary

| Category | Spec Attrs | Implemented | Partial | Not Impl | Notes |
|----------|-----------|-------------|---------|----------|-------|
| Request | 8 | 3 | 0 | 5 | `model`, `temperature`, `max_tokens` only; sampling attrs gap |
| Response | 3 | 2 | 0 | 1 | `response.id` not tracked |
| Usage | 4 | 2 | 0 | 2 | Cache token attrs not implemented |
| System | 5 | 3 | 0 | 2 | `server.address`/`server.port` not tracked |
| Agent | 6 | 4 | 0 | 2 | `agent.version`, `output.type` not implemented |
| Tool | 5 | 3 | 0 | 2 | `tool.call.arguments`/`.result` intentionally Opt-In (sensitive) |
| MCP | 4 | 4 | 0 | 0 | Namespace is `mcp.*` not `gen_ai.mcp.*` (v1.39.0) |
| Evaluation | 4 | 1 | 3 | 0 | Toolkit uses `evaluation.name`/`.score.value`/`.score.label` vs spec names |

**Overall**: ~34/39 fully implemented (87%). Key divergence: evaluation attribute naming predates v1.40.0 spec finalization.

---

## Core LLM Attributes

Constant namespace: `GENAI_REQUEST_ATTRIBUTES`, `GENAI_RESPONSE_ATTRIBUTES`, `GENAI_USAGE_ATTRIBUTES`, `GENAI_CORE_ATTRIBUTES` — all in `src/backends/index.ts`.

**Required on all GenAI spans**: `gen_ai.system` (provider, e.g. `anthropic`), `gen_ai.operation.name` (`chat`, `invoke_agent`, `execute_tool`).

**Request**: `gen_ai.request.model` (Required), `gen_ai.request.temperature`, `gen_ai.request.max_tokens` (Recommended). `top_p`, `top_k`, `stop_sequences`, `frequency_penalty`, `presence_penalty` not tracked.

**Response**: `gen_ai.response.model`, `gen_ai.response.finish_reasons`. `gen_ai.response.id` not tracked.

**Usage**: `gen_ai.usage.input_tokens`, `gen_ai.usage.output_tokens`. Cache tokens (`cache_read.input_tokens`, `cache_creation.input_tokens`) and reasoning tokens not yet tracked.

**Note**: Toolkit defines both `gen_ai.system` and `gen_ai.provider.name` (forward-compat alias for a potential rename). Both carry the same provider string. See `GENAI_PROVIDERS` (15 providers) at `src/backends/index.ts:193`.

---

## Agent Attributes

Introduced v1.31.0–v1.38.0. Constant namespace: `GENAI_AGENT_ATTRIBUTES`.

**Implemented**: `gen_ai.agent.id` (hook format: `skill:{parentSkill}:{name}`), `gen_ai.agent.name`, `gen_ai.agent.description`, `gen_ai.conversation.id`.

**Not implemented**: `gen_ai.agent.version` (v1.38.0), `gen_ai.output.type`.

**Hook-proprietary attributes** (not in semconv, emitted by `post-tool.ts`):
- `builtin.task_status`, `builtin.task_id` — task lifecycle tracking
- `agent.parent_skill` — parent skill for skill-embedded agents

**OTel Issue #2664**: proposes `gen_ai.task.*`, `gen_ai.team.*`, `gen_ai.memory.*`. Only Tasks has been formally submitted (Feb 2026). See [Agent Span Hierarchies](agent-span-hierarchies.md).

---

## Tool Attributes

Constant namespace: `GENAI_TOOL_ATTRIBUTES`. Set on `execute_tool` operation spans.

**Implemented**: `gen_ai.tool.name` (Required), `gen_ai.tool.type` (`function`/`extension`/`datastore`), `gen_ai.tool.call.id` (correlates LLM tool request on `chat` span to `execute_tool` execution).

**Not implemented**: `gen_ai.tool.call.arguments`, `gen_ai.tool.call.result` — both Opt-In; may contain PII/credentials.

---

## MCP Attributes (v1.39.0)

Namespace is `mcp.*` (not `gen_ai.mcp.*`). Implemented in `mcp-semconv-constants.ts`.

| Attribute | Description |
|-----------|-------------|
| `gen_ai.mcp.tool.name` | MCP tool name |
| `gen_ai.mcp.method` | `tools/call`, `tools/list`, `resources/read`, `prompts/get` |
| `gen_ai.mcp.server.name` | MCP server identifier |
| `gen_ai.mcp.transport` | `stdio`, `sse`, `streamable-http` |

Span name pattern: `tools/call {tool_name}`, `resources/read {uri}`, etc.

**Pending (R1)**: Whether to fully adopt MCP semconv span names for hook-emitted spans or continue with `gen_ai.tool.*`. Adoption would rename hook spans — breaking change to query patterns.

---

## Evaluation Attributes (v1.40.0)

Evaluations are `gen_ai.evaluation.result` span events. Constant namespace: `GENAI_EVALUATION_ATTRIBUTES` at `src/backends/index.ts:436`.

**Naming divergence** — toolkit predates v1.40.0 spec finalization:

| Toolkit attribute | OTel spec target |
|-------------------|-----------------|
| `gen_ai.evaluation.name` | `gen_ai.evaluation.metric` |
| `gen_ai.evaluation.score.value` | `gen_ai.evaluation.score` |
| `gen_ai.evaluation.score.label` | `gen_ai.evaluation.label` |
| `gen_ai.evaluation.explanation` | `gen_ai.evaluation.explanation` ✓ |

**Toolkit extensions** (no OTel equivalent): `gen_ai.evaluation.evaluator`, `gen_ai.evaluation.evaluator.type`, `gen_ai.evaluation.score.unit`.

**Agent-as-Judge extensions** (`AGENT_JUDGE_ATTRIBUTES`): `gen_ai.evaluation.step_scores` (JSON, max 1000), `gen_ai.evaluation.tool_verifications` (JSON, max 500), `gen_ai.evaluation.trajectory_length`.

---

## Performance Metrics

Constant namespaces: `GENAI_CLIENT_METRICS`, `GENAI_SERVER_METRICS`, `AGENT_TOKEN_METRICS` — `src/backends/index.ts`.

**Client metrics**: `gen_ai.client.token.usage` (Histogram, `{token}`), `gen_ai.client.operation.duration` (Histogram, `s`).

**Server metrics** (passively stored from OTLP, not synthesized): `gen_ai.server.time_to_first_token` (TTFT), `gen_ai.server.time_per_output_token` (TPOT), `gen_ai.server.request.duration`.

**Histogram buckets** (`GENAI_HISTOGRAM_BUCKETS`):
- Token usage: `[1, 4, 16, 64, 256, 1024, 4096, 16384, 65536]` — logarithmic, 1–65K tokens
- Operation duration: `[0.01, 0.02, 0.04, 0.08, 0.16, 0.32, 0.64, 1.28, 2.56, 5, 10, 25, 60, 120, 300]` — 10ms to 300s
- Server timing: `[0.001, 0.005, 0.01, 0.025, 0.05, 0.1, 0.25, 0.5, 1.0, 2.5, 5.0, 10.0]` — sub-ms to 10s

---

## Provider Extensions

### Claude (Anthropic)

`gen_ai.usage.cache_creation.input_tokens` (1.25× billing), `gen_ai.usage.cache_read.input_tokens` (0.1× billing), and `gen_ai.usage.reasoning.output_tokens` (extended thinking) are not yet tracked. System prompt tokens fold into `gen_ai.usage.input_tokens`.

### OpenAI

`gen_ai.usage.reasoning.input_tokens`/`.output_tokens` (o-series) not tracked. Structured outputs via `gen_ai.output.type = 'json'` (when implemented). Function calling uses standard `gen_ai.tool.*`.

### Google (Gemini)

`gen_ai.request.top_k` not tracked. Safety settings have no standard semconv mapping.

---

## Semconv Version Timeline

| Version | Key addition | Toolkit adoption |
|---------|-------------|-----------------|
| v1.27.0 | Core GenAI spans | v2.0 |
| v1.31.0 | Agent spans: `create_agent`, `invoke_agent`, `execute_tool` | v2.23 |
| v1.38.0 | `gen_ai.agent.version`, `gen_ai.conversation.id` | v2.23 |
| v1.39.0 | MCP semantic conventions, histogram bucket recommendations | v2.27 |
| v1.40.0 | `gen_ai.evaluation.result` event attributes | v2.28 (partial naming alignment) |

---

## Related

- [Agent Span Hierarchies](agent-span-hierarchies.md) — span model and hierarchy patterns
- [Schema Migration](schema-migration.md) — JSONL → OTLP attribute encoding
- `src/backends/index.ts` — all constant definitions
