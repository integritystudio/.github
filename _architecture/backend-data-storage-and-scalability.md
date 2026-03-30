# Backend Data Storage, Messaging, and Scalability

**Version**: 1.0
**Date**: 2026-02-08
**Scope**: observability-toolkit v2.0.1
**Purpose**: Architecture reference for production-layer data storage, long-term retention, messaging system design, and scalability/efficiency/reliability considerations.

---

## 1. Production-Layer Data Storage

### Dual-Backend Architecture

The toolkit operates with two production backends, selectable per query.

```
MCP Tool Request
    ↓
Backend Selection (auto | local | signoz)
    ↓
┌─────────────────────┐     ┌──────────────────────┐
│  Local JSONL Backend │     │  SigNoz Cloud Backend │
│  (MultiDirectory)    │     │  (OTLP + Query API)   │
├─────────────────────┤     ├──────────────────────┤
│  ~/.claude/telemetry │     │  HTTPS ingest endpoint │
│  Flat JSONL files    │     │  ClickHouse storage    │
│  .idx sidecar index  │     │  SSRF-safe client      │
└─────────────────────┘     └──────────────────────┘
```

### Local JSONL Backend

**Implementation**: `src/backends/local-jsonl.ts` (1800+ lines)

The local backend stores telemetry as flat JSONL (one JSON object per line) in `~/.claude/telemetry/` (configurable via `TELEMETRY_DIR`).

**File Layout**:

| File | Format | Contents |
|------|--------|----------|
| `traces.jsonl` | FlatSpan | Span records with traceId, spanId, attributes, events, links |
| `logs.jsonl` | FlatLog | Log records with severity, body, traceId correlation |
| `metrics.jsonl` | FlatMetric | Counter, histogram, gauge data points with exemplars |
| `llm-events.jsonl` | FlatLLMEvent | LLM operation events (chat, embeddings, tool calls) |
| `evaluations.jsonl` | FlatEvaluationResult | Evaluation scores, explanations, tool verifications |

**Compression**: Gzip support (`.jsonl.gz`) with transparent decompression at read time. Both compressed and uncompressed files are discovered and merged.

**Multi-Directory Discovery**: The `MultiDirectoryBackend` searches multiple locations:
- `~/.claude/telemetry/` (global)
- `.claude/telemetry/` (project-local)
- `telemetry/` and `.telemetry/` (project root)

**Schemas** (defined in `local-jsonl.ts:226-301`):

```typescript
// FlatSpan: OTel span with nanosecond-precision timestamps
interface FlatSpan {
  traceId: string;
  spanId: string;
  parentSpanId?: string;
  name: string;
  kind: number;
  startTime: [number, number]; // [seconds, nanoseconds]
  endTime: [number, number];
  status: { code: number; message?: string };
  attributes: Record<string, unknown>;
  events: SpanEvent[];
  links: SpanLink[];
  resource: Record<string, unknown>;
  instrumentationScope: { name: string; version?: string };
}
```

**Indexing** (`src/lib/indexer.ts`): Sidecar `.idx` files store per-file metadata (line offsets, timestamp ranges, attribute value sets) enabling fast filtering without full file scans.

### SigNoz Cloud Backend

**Implementation**: `src/backends/signoz-api.ts` (50KB)

**Endpoints**:
- **Ingest**: `https://ingest.{region}.signoz.cloud/v1/{traces,metrics,logs}` (OTLP HTTP)
- **Query**: `https://{tenant}.signoz.io` or `SIGNOZ_QUERY_URL` (ClickHouse-backed API)

**Security**:
- HTTPS-only enforcement
- SSRF validation blocks localhost, private networks (10.x, 172.16-31.x, 192.168.x), and cloud metadata endpoints (169.254.169.254)
- API key passed via `SIGNOZ-ACCESS-TOKEN` header

### Backend Selection Logic

```
auto mode:
  1. Check SIGNOZ_ACCESS_TOKEN env var
  2. If present → use SigNoz backend
  3. If absent → use local JSONL backend

Explicit override via `backend` parameter on every query tool.
```

---

## 2. Long-Term Data Storage

### Data Lifecycle

```
Write Phase          Active Phase           Archive Phase
─────────────       ─────────────          ─────────────
OTLP ingest    →    JSONL on disk    →     Gzip compression
Hook emission  →    .idx sidecar     →     Rotated files
                    Query-ready             Reduced scan cost
```

### Retention and Limits

| Constraint | Value | Location |
|-----------|-------|----------|
| Max results in memory | 10,000 | `constants.ts:MAX_RESULTS_IN_MEMORY` |
| Streaming threshold | 5,000 | `constants.ts:STREAMING_THRESHOLD` |
| Worst files limit | Configurable | `constants.ts:WORST_FILES_LIMIT` |
| Cache TTL | 60s default | `CACHE_TTL_MS` env var |
| Rate limit window | 60s / 100 requests | `server-utils.ts` |

### Local Retention Strategy

The local backend has no built-in rotation or purge. Long-term storage considerations:

1. **File Growth**: JSONL files grow unbounded. Production deployments should implement external rotation (logrotate, cron-based archival).
2. **Gzip Archival**: Move older `.jsonl` files to `.jsonl.gz`. The backend reads both transparently.
3. **Date-Based Filtering**: All query tools accept `startDate`/`endDate` parameters. The indexer skips files whose timestamp range falls entirely outside the query window.
4. **Disk Budget**: A typical Claude Code session produces ~50-200KB of telemetry per hour. At sustained use, plan for ~5-10GB/year without compression, ~1-2GB/year with gzip.

### SigNoz Retention

SigNoz Cloud manages retention via ClickHouse TTL policies. Default retention varies by plan (15 days to unlimited). The toolkit delegates all retention decisions to the remote backend.

---

## 3. Messaging System Design

### MCP Request/Response Model

The toolkit communicates exclusively via the Model Context Protocol (MCP) over stdio transport. There is no HTTP server, WebSocket, or message queue.

```
Claude Host (LLM)
    │
    │  JSON-RPC over stdio
    │
    ▼
MCP Server (this toolkit)
    │
    ├── tools/call handler
    │       ↓
    │   Rate Limiter → Tool Lookup → Zod Validation → Backend Query → Response
    │
    └── Graceful shutdown (SIGTERM/SIGINT)
```

**Request Flow** (`src/server.ts:165-217`):

1. **Rate limiting**: Sliding window counter (100 req/60s, 1-second granularity). Returns `isError: true` with retry-after guidance when exceeded.
2. **Tool lookup**: Map-based O(1) lookup from 15 registered tools.
3. **Schema validation**: Zod schema parsing with structured error messages on failure.
4. **OTel span wrapping**: Each tool invocation is wrapped in a trace span for self-instrumentation.
5. **Error sanitization**: All errors pass through `sanitizeErrorForResponse()` before reaching the client.

### Message Boundaries

- **Input**: JSON-RPC `tools/call` with tool name + arguments object
- **Output**: MCP content array (`text` type) with JSON-serialized results
- **Errors**: MCP `isError: true` with sanitized message (no stack traces in production)
- **No streaming**: All responses are complete before delivery. Large result sets are truncated to `limit` parameter (default 50, max 1000).

### Tool Schema Contract

Each tool defines its interface via Zod schemas converted to JSON Schema at server startup (`zod-to-json-schema` with `$refStrategy: 'none'` for MCP compatibility). This serves as the messaging contract between host and server.

```typescript
// Tool definition pattern (src/tools/index.ts)
export const toolDefinitions: ToolDefinition[] = [
  {
    name: 'obs_query_traces',
    description: '...',
    inputSchema: traceQuerySchema,    // Zod schema
    handler: queryTracesHandler,       // async (args) => result
  },
  // ... 14 more tools
];
```

---

## 4. Scalability Considerations

### Memory Efficiency

| Mechanism | Implementation | Bound |
|-----------|---------------|-------|
| LRU query cache | `src/lib/cache.ts` | 100 entries per cache, 500 total (5 caches) |
| LRU regex cache | `local-jsonl.ts:89-118` | 100 compiled patterns |
| Result streaming | `file-utils.ts` | Switches at 5,000 results |
| Result cap | All query tools | 10,000 max in memory |
| Rate limiter buckets | `server-utils.ts` | Fixed 60 buckets (O(1) memory) |

**Key principle**: No unbounded data structures. Every Map, array, and cache has a maximum size.

### Query Performance

1. **Index-first filtering**: `.idx` sidecar files enable timestamp and attribute pre-filtering without reading full JSONL files.
2. **Streaming for large datasets**: Above `STREAMING_THRESHOLD` (5,000), the backend switches from in-memory array accumulation to streaming line-by-line processing.
3. **Regex caching**: Frequently-used span name patterns are compiled once and cached (LRU, max 100).
4. **Date range pruning**: Files whose indexed timestamp range doesn't overlap the query window are skipped entirely.

### Reliability Patterns

#### Circuit Breaker (`src/lib/circuit-breaker.ts`)

Protects against cascading failures when the SigNoz backend is unavailable.

```
States: CLOSED → OPEN → HALF_OPEN → CLOSED

CLOSED:    Normal operation. Track failure count.
           3 consecutive failures → transition to OPEN.

OPEN:      All requests fail fast (no network call).
           After 60-second cooldown → transition to HALF_OPEN.

HALF_OPEN: Allow one probe request.
           Success → CLOSED. Failure → OPEN.
```

**Parameters**:
- Failure threshold: 3 consecutive failures
- Recovery timeout: 60 seconds
- Applies to: SigNoz API backend only (local JSONL has no external dependency)

#### Rate Limiter (`src/lib/server-utils.ts:52-188`)

Sliding window counter prevents resource exhaustion from runaway LLM loops.

- **Window**: 60 seconds
- **Max requests**: 100 per window
- **Granularity**: 1-second buckets (60 total)
- **Clock**: Monotonic (`process.hrtime.bigint()`) with `Date.now()` fallback
- **Memory**: Fixed O(60) regardless of request volume

#### Error Categorization (`src/lib/error-types.ts`)

Errors are classified by category to determine recovery strategy:

| Category | Retryable | Examples |
|----------|-----------|---------|
| `user` | No | Invalid input, bad date range, regex syntax error |
| `system` | No | Parse error, memory error, init failure |
| `network` | Yes | Timeout, connection failure, API error, circuit open |
| `config` | No | Missing config, invalid config, SSRF blocked |

### Efficiency Optimizations

1. **Lazy file discovery**: JSONL files are discovered once per backend initialization, not per query.
2. **Cache hit rates**: OTel metrics (`obs_toolkit.cache.hits`, `obs_toolkit.cache.misses`) track cache effectiveness. Target hit rate: >60% for repeated dashboard queries.
3. **Zod schema compilation**: Schemas are compiled once at startup and reused for every request validation.
4. **JSON Schema conversion**: `zod-to-json-schema` runs once at server init, not per tool listing request.
5. **Monotonic clock**: Rate limiter uses `process.hrtime.bigint()` to avoid clock skew issues.

### Horizontal Scaling Limitations

The toolkit is designed as a **single-process MCP server** connected to one host via stdio. It does not support:
- Multiple concurrent host connections
- Clustered deployment
- Shared state across instances
- Message queuing or pub/sub

This is by design: MCP servers are 1:1 with their host process. Horizontal scaling happens at the SigNoz backend layer (ClickHouse cluster), not at the MCP server layer.

---

## 5. Data Flow Summary

```
                     ┌─────────────────────────────────────────────┐
                     │              MCP Server Process              │
                     │                                             │
  JSON-RPC stdin ──▶ │  Rate Limiter                               │
                     │      ↓                                      │
                     │  Tool Router (15 tools)                     │
                     │      ↓                                      │
                     │  Zod Validation                             │
                     │      ↓                                      │
                     │  ┌─────────────┐   ┌──────────────────┐    │
                     │  │ Query Cache  │──▶│ Cache Hit → Return│    │
                     │  │ (LRU, 100)  │   └──────────────────┘    │
                     │  └──────┬──────┘                            │
                     │         │ miss                               │
                     │         ▼                                    │
                     │  ┌──────────────┐   ┌────────────────────┐  │
                     │  │ Local JSONL  │   │ SigNoz Cloud       │  │
                     │  │ + Indexer    │   │ + Circuit Breaker  │  │
                     │  └──────────────┘   └────────────────────┘  │
                     │         │                    │               │
                     │         └────────┬───────────┘               │
                     │                  ▼                           │
                     │  Error Sanitization → OTel Span → Response  │
                     │                                             │
  JSON-RPC stdout ◀──│                                             │
                     └─────────────────────────────────────────────┘
```

---

## Related Documents

| Document | Relevance |
|----------|-----------|
| [quality-metrics-dashboard.md](quality-metrics-dashboard.md) | Dashboard pipeline that queries stored evaluation data |
| [caching-design.md](caching-design.md) | Detailed caching architecture |
| [error-tracking-design.md](error-tracking-design.md) | Error handling and tracking patterns |
| [../quality/quality-evaluation.md](../quality/quality-evaluation.md) | Evaluation storage and export |
| [../visibility/dashboard-configuration.md](../visibility/dashboard-configuration.md) | OTel metrics dashboards |

---

## Implementation Files

| File | Description |
|------|-------------|
| `src/backends/local-jsonl.ts` | Local JSONL backend with multi-directory discovery (1800+ lines) |
| `src/backends/signoz-api.ts` | SigNoz Cloud backend with SSRF protection (50KB) |
| `src/backends/index.ts` | TelemetryBackend interface, query option schemas, OTel semantic conventions |
| `src/server.ts` | MCP server entry point, request handling pipeline |
| `src/lib/server-utils.ts` | Rate limiter, ServerInitError |
| `src/lib/circuit-breaker.ts` | Circuit breaker for remote backend resilience |
| `src/lib/indexer.ts` | Sidecar .idx file indexing for fast filtering |
| `src/lib/file-utils.ts` | File I/O, path validation, JSONL streaming |
| `src/lib/constants.ts` | Configuration limits and thresholds |
