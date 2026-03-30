# Agent Span Hierarchies

**Semconv**: v1.40.0 (agent spans landed v1.31.0–v1.38.0)
**OTel SDK**: `sdk-node` / `exporter-trace-otlp-http` v0.213.x · `sdk-trace-base` / `resources` v2.x
**Source**: [appendix-deep-dives.md](../changelog/2.28/appendix-deep-dives.md) A2

---

## Span Model

Every span has a `TraceId`, `SpanId`, optional `ParentSpanId`, `Kind`, `Status`, `Attributes`, and `Events`. Spans form a parent-child tree within a trace; `SpanLink` objects carry cross-trace relationships.

**Status precedence**: `ERROR > OK > UNSET`. Once set to `OK`, a status is final. OTel does not auto-propagate errors up the span tree — each span sets its own status.

### Five Core Agent Operations

| Operation | Span Kind | Description |
|-----------|-----------|-------------|
| `create_agent` | INTERNAL or CLIENT | Agent instantiation — precedes first `invoke_agent` |
| `invoke_agent` | INTERNAL (in-process) or CLIENT (remote) | Agent execution entry point |
| `execute_tool` | INTERNAL (function) or CLIENT (remote) | Single tool call |
| `chat` / `generate_content` | CLIENT | LLM API call |
| `retrieval` | CLIENT | Vector store or knowledge base lookup |

**Span kind rule**: use `CLIENT` only when there is an actual network call (HTTP, gRPC, external subprocess). In-process agents and local function tools are `INTERNAL`.

---

## Single-Agent Span Tree

A typical agent loop: receive user input → call LLM → execute tools → call LLM again → return response.

```
[INTERNAL] invoke_agent my-agent
  gen_ai.agent.name = "my-agent"
  gen_ai.usage.input_tokens  = 1240   ← aggregate across all child chat spans
  gen_ai.usage.output_tokens = 87
  |
  +-- [CLIENT] chat gpt-4o            (model requests tools)
  |     gen_ai.response.finish_reasons = ["tool_calls"]
  |
  +-- [INTERNAL] execute_tool search_web
  +-- [INTERNAL] execute_tool read_file
  |
  +-- [CLIENT] chat gpt-4o            (final response after tools)
        gen_ai.response.finish_reasons = ["stop"]
```

`invoke_agent` holds **aggregate** token counts; each `chat` span holds **per-call** counts. Tool spans are siblings of `chat` spans (both children of `invoke_agent`), not children of them.

---

## Multi-Agent Patterns

### Same-Process Delegation

Sub-agents within the same process share one `TraceId`. The coordinator's `invoke_agent` is the root; sub-agents' `invoke_agent` spans are INTERNAL children.

```
[INTERNAL] invoke_agent coordinator
  +-- [CLIENT] chat gpt-4o            (planning)
  +-- [INTERNAL] invoke_agent researcher
  |     +-- [CLIENT] chat gpt-4o
  |     +-- [INTERNAL] execute_tool web_search
  +-- [INTERNAL] invoke_agent writer
  |     +-- [CLIENT] chat gpt-4o
  +-- [CLIENT] chat gpt-4o            (synthesis)
```

### Remote Delegation

When an orchestrator calls a remote agent, OTel creates two traces connected by a span link. W3C `traceparent` carries context across the boundary; the remote agent creates a new `TraceId` and links back.

```
Trace T1: Orchestrator
  [CLIENT] invoke_agent remote-processor
    SpanLink → (TraceId: T2, SpanId: S2-root)

Trace T2: Processor (separate service)
  [SERVER] invoke_agent remote-processor
    gen_ai.conversation.id = "conv-xyz"   ← shared correlation key
    +-- [CLIENT] chat claude-3-5-sonnet
    +-- [INTERNAL] execute_tool process_data
```

### Group Chat (Sequential) vs. Consensus (Parallel)

Sequential agents (AG2, CrewAI speaker-selection) have non-overlapping `startTime`/`endTime` on their `invoke_agent` spans. Parallel consensus agents have **overlapping** timestamps — that overlap is how you detect concurrency.

---

## Tool Chain Patterns

- **Sequential**: tool spans have non-overlapping timestamps, each a sibling of `chat` under `invoke_agent`
- **Parallel**: overlapping timestamps; all dispatched after a single `chat` with `finish_reason: tool_calls`
- **Nested**: a tool can invoke an agent, creating `invoke_agent` inside `execute_tool` — 4-level nesting
- **RAG**: `retrieval` span under `execute_tool` (or directly under `invoke_agent` when agent retrieves directly)

---

## Error Propagation

OTel does not auto-bubble errors. Rules:

- Tool failure → set `ERROR` on that span; parent `invoke_agent` must set `ERROR` explicitly
- Retry → each attempt is a **new span**; same `gen_ai.tool.call.id` links attempt to retry
- Recovery → if retry succeeds, parent `invoke_agent` status is `OK`
- Partial failure in parallel tools → parent decides `OK` (graceful degradation) or `ERROR` (fatal)

---

## Correlation

| Mechanism | Scope | Use When |
|-----------|-------|----------|
| `TraceId` | Single trace | All spans in same-process hierarchy |
| `gen_ai.conversation.id` | Cross-trace | Multi-turn sessions, group chats, cross-service |
| Span links | Cross-trace | Async/remote agent delegation |
| `gen_ai.tool.call.id` | Within trace | Links LLM tool request (`chat` span) to `execute_tool` execution |
| W3C `traceparent` | Cross-service | Propagates trace context to remote agents |

`traceparent` format: `00-<trace-id-32hex>-<parent-span-id-16hex>-<flags>`

---

## Toolkit Mapping

Toolkit spans are primarily `INTERNAL` (local agent loop) and `CLIENT` (LLM API calls). MCP tool calls via Claude Code use stdio transport and are classified as `INTERNAL` — a deliberate toolkit choice since they run within the same logical agent session. Other OTel implementations may prefer `CLIENT` for subprocess calls.

OTel attribute constants: `GENAI_AGENT_ATTRIBUTES`, `GENAI_TOOL_ATTRIBUTES`, `GENAI_CORE_ATTRIBUTES` in `src/backends/index.ts`. Quality evaluation layer: `computeMultiAgentEvaluation()` and `evaluateHandoffs()` in `src/lib/quality/quality-multi-agent.ts`; `verifyToolCall()` and `analyzeTrajectory()` in `src/lib/agent-judge/agent-judge-verification.ts`.

---

## Proposed Extensions (OTel Issue #2664)

As of v1.40.0, only `Tasks` has been formally proposed. The `invoke_agent` / `execute_tool` model remains normative. Pending proposals would add `gen_ai.task.*`, `gen_ai.team.*`, and `gen_ai.memory.*` attributes. Until formalized, use `gen_ai.conversation.id` for session correlation.
