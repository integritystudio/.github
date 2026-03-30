# Token Optimization Guide for observability-toolkit

> Strategies to reduce MCP tool definition and response token usage

## Overview

The observability-toolkit MCP server has been optimized from ~3.1k tokens to ~2.4k tokens across 9 tool definitions (~24% reduction). This document covers implemented optimizations and future strategies.

## Current Token Footprint (Post-Optimization)

| Tool | Est. Tokens | Parameters | Status |
|------|-------------|------------|--------|
| `obs_query_traces` | ~510 | 17 | Optimized |
| `obs_query_evaluations` | ~450 | 15 | Optimized |
| `obs_query_llm_events` | ~350 | 9 | Optimized |
| `obs_query_metrics` | ~280 | 7 | Optimized |
| `obs_query_logs` | ~250 | 8 | Optimized |
| `obs_setup_claudeignore` | ~230 | 6 | Optimized |
| `obs_context_stats` | ~120 | 3 | Optimized |
| `obs_get_trace_url` | ~105 | 1 | Optimized |
| `obs_health_check` | ~85 | 1 | Optimized |
| **Total** | **~2,380** | | **~24% saved** |

---

## Implemented Optimizations

### 1. Description Shortening

All tool descriptions shortened while preserving meaning:

```typescript
// Before
backend: z.enum(['local', 'signoz', 'auto']).default('auto')
  .describe('Backend to query: local (JSONL files), signoz (API), or auto (tries both)')

// After
backend: z.enum(['local', 'signoz', 'auto']).default('auto')
  .describe('Backend (default: auto)')
```

### 2. Shared Schema Extraction

Common parameters extracted to `lib/shared-schemas.ts`:

```typescript
import {
  dateRangeSchema,    // startDate, endDate
  paginationSchema,   // limit (default: 50)
  backendSchema,      // backend enum
  sessionSchema,      // sessionId
  traceIdSchema,      // traceId
  operationNameSchema // OTel GenAI operation
} from '../lib/shared-schemas.js';

export const queryLogsSchema = z.object({
  ...backendSchema,
  ...sessionSchema,
  ...dateRangeSchema,
  ...paginationSchema,
  severity: z.string().optional().describe('Severity (ERROR/WARN/INFO/DEBUG)'),
  search: z.string().optional().describe('Search text'),
  ...traceIdSchema,
});
```

### 3. TOON Encoder Utility

Available in `lib/toon-encoder.ts` for 40-60% response token reduction:

```typescript
import { toToon, isToonBeneficial } from './lib/toon-encoder.js';

// Check if TOON would help (uniform arrays)
if (isToonBeneficial(traces)) {
  const compact = toToon(traces); // 40-60% smaller
}
```

---

## Future Optimization Strategies

### Tool Consolidation (If Needed)

Consolidate 5 query tools into unified interface:

```typescript
export const obsQuerySchema = z.object({
  type: z.enum(['traces', 'logs', 'metrics', 'llm_events', 'evaluations']),
  backend: z.enum(['local', 'signoz', 'auto']).default('auto'),
  startDate: z.string().optional(),
  endDate: z.string().optional(),
  limit: z.number().max(1000).default(50),
  filters: z.record(z.unknown()).optional(),
});
```

**Trade-offs:** Removes ~1,500 tokens but less discoverable. Only consider if adding many new tools.

### Dynamic Toolsets Pattern

For 20+ tools, implement progressive disclosure:

```typescript
const obsSearchTools = {
  name: 'obs_search_tools',
  description: 'Search available observability tools',
  inputSchema: z.object({ query: z.string() }),
};

const obsExecute = {
  name: 'obs_execute',
  description: 'Execute an observability tool',
  inputSchema: z.object({
    tool: z.string(),
    params: z.record(z.unknown()),
  }),
};
```

### Code Execution Pattern

For 100+ tools, present MCP tools as code APIs that the LLM writes against.

---

## TOON Format Reference

### When to Use TOON

| Data Type | Savings | Recommendation |
|-----------|---------|----------------|
| Uniform arrays (traces, logs) | 40-60% | Use TOON |
| Time-series metrics | 50-60% | Use TOON |
| Nested structures | 5-15% | Consider JSON compact |
| Deeply nested config | -10% | Use JSON |

### Example

```json
// JSON (100 tokens)
{
  "traces": [
    {"traceId": "abc", "name": "query", "durationMs": 150},
    {"traceId": "def", "name": "insert", "durationMs": 230}
  ]
}
```

```
// TOON (40 tokens) - 60% reduction
traces[2]{traceId,name,durationMs}:
  abc,query,150
  def,insert,230
```

---

## Implementation Status

| Phase | Status | Impact |
|-------|--------|--------|
| Description shortening | Complete | ~24% reduction |
| Shared schema extraction | Complete | Code maintainability |
| TOON utility | Available | 40-60% response reduction |
| Tool consolidation | Not needed | - |
| Dynamic toolsets | Not needed | - |

---

## References

- [SEP-1576: Mitigating Token Bloat in MCP](https://github.com/modelcontextprotocol/modelcontextprotocol/issues/1576)
- [Reducing MCP token usage by 100x - Speakeasy](https://www.speakeasy.com/blog/how-we-reduced-token-usage-by-100x-dynamic-toolsets-v2)
- [Code execution with MCP - Anthropic](https://www.anthropic.com/engineering/code-execution-with-mcp)
- [TOON Format](https://toonformat.dev/) | [GitHub](https://github.com/toon-format/toon)

---

*Last updated: 2026-01-31*
