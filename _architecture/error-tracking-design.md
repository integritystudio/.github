# Error Tracking Design

**Version**: 1.0
**Date**: 2026-02-08
**Scope**: observability-toolkit v2.0.1
**Purpose**: Error classification, sanitization, and tracking architecture for the MCP server.

---

## 1. Error Classification System

### Error Categories

**Implementation**: `src/lib/error-types.ts` (200 lines)

All errors are classified into four categories that determine recovery behavior and client-facing messaging.

| Category | Retryable | Cause | Client Action |
|----------|-----------|-------|---------------|
| `user` | No | Bad input from the LLM host | Fix query parameters and retry |
| `system` | No | Internal toolkit failure | Report bug; do not retry |
| `network` | Yes | Remote backend unavailable | Retry after backoff; check circuit breaker |
| `config` | No | Missing or invalid configuration | Fix environment variables |

### Error Codes

```typescript
// User errors
INVALID_INPUT       // Malformed query parameters
INVALID_DATE_RANGE  // startDate > endDate or unparseable dates
INVALID_REGEX       // Bad regex in spanNameRegex or search
LIMIT_EXCEEDED      // Requested limit > MAX (1000)

// System errors
INTERNAL_ERROR      // Unexpected runtime failure
PARSE_ERROR         // JSONL line parse failure (malformed telemetry data)
MEMORY_ERROR        // Result set exceeds MAX_RESULTS_IN_MEMORY (10,000)
INIT_FAILED         // Server initialization failure

// Network errors
TIMEOUT             // SigNoz API response timeout
CONNECTION_FAILED   // TCP connection failure to remote backend
API_ERROR           // SigNoz returned non-2xx status
CIRCUIT_OPEN        // Circuit breaker is in OPEN state

// Config errors
MISSING_CONFIG      // Required env var not set (e.g., SIGNOZ_ACCESS_TOKEN)
INVALID_CONFIG      // Env var present but malformed
SSRF_BLOCKED        // URL failed SSRF validation
```

### CategorizedError Interface

```typescript
interface CategorizedError {
  category: ErrorCategory;   // 'user' | 'system' | 'network' | 'config'
  code: ErrorCode;           // Specific error code
  message: string;           // Human-readable, sanitized message
  retryable: boolean;        // Whether retry may succeed
  cause?: Error;             // Original error (internal only, never sent to client)
}
```

### Retryability Logic (`error-types.ts:110-142`)

```
user errors    → never retryable (fix input first)
system errors  → never retryable (internal bug)
network errors → always retryable (transient by nature)
config errors  → never retryable (fix config first)
```

---

## 2. Error Sanitization

### Production Safety

**Implementation**: `src/lib/error-sanitizer.ts` (267 lines)

Error messages are sanitized before reaching the MCP client to prevent information disclosure.

**What gets stripped**:
- Absolute file paths (Unix `/home/...`, Windows `C:\...`, Node internal `node:...`)
- Stack traces (in production mode)
- Internal module names and line numbers

**What passes through**:
- Error codes (e.g., `INVALID_INPUT`)
- Sanitized message text
- Retryability flag

### Environment Detection (`error-sanitizer.ts:33-54`)

```
Production mode (sanitize aggressively):
  - NODE_ENV unset (default-safe)
  - NODE_ENV=production
  - Any cloud environment detected (AWS Lambda, GCP, Azure, Kubernetes, Railway, Render)

Development mode (include more detail):
  - NODE_ENV=development
  - NODE_ENV=test
  - NOT in a cloud environment
```

**Cloud detection checks**: `AWS_LAMBDA_FUNCTION_NAME`, `GOOGLE_CLOUD_PROJECT`, `AZURE_FUNCTIONS_ENVIRONMENT`, `KUBERNETES_SERVICE_HOST`, `RAILWAY_ENVIRONMENT`, `RENDER`.

If `NODE_ENV=development` but cloud environment variables are present, the sanitizer logs a warning and enforces production mode.

### Path Sanitization (`error-sanitizer.ts:60-87`)

```typescript
// Single regex with alternation for O(n) pass
const SENSITIVE_PATH_PATTERN = /(\/.+?\/[^\s:)]+)|(\\[A-Za-z]:.+?)|(node:[^\s:)]+)/g;

// Input truncated to 100KB before regex to prevent ReDoS
// Paths replaced with [path] or [internal]
```

### Response Function (`error-sanitizer.ts:213-233`)

```typescript
sanitizeErrorForResponse(error: unknown, includeStack?: boolean): {
  message: string;      // Sanitized message
  code?: string;        // Error code if CategorizedError
  retryable?: boolean;  // Retry hint if CategorizedError
  stack?: string;       // Only in development mode, sanitized
}
```

---

## 3. Error Handling Layers

### Layer 1: Zod Validation (Input Boundary)

Every tool request is validated against its Zod schema before execution. Validation failures produce `user` category errors with specific field-level messages.

```
MCP Request → Zod parse → Success: proceed to handler
                        → Failure: INVALID_INPUT with field details
```

### Layer 2: Rate Limiter (Resource Protection)

Requests exceeding 100/minute are rejected before reaching any tool handler.

```
Request → Rate check → Under limit: proceed
                     → Over limit: isError=true with retry guidance
```

### Layer 3: Tool Handler (Business Logic)

Each tool handler wraps its logic in try/catch and classifies errors:

```typescript
// Pattern from src/server.ts:187-217
const result = await Sentry.startSpan(
  { name: `tool.${toolName}`, op: 'mcp.tool' },
  async () => {
    try {
      return await tool.handler(validatedArgs);
    } catch (error) {
      // Classify → sanitize → return as MCP error
      const categorized = categorizeError(error);
      return {
        content: [{ type: 'text', text: sanitizeErrorForResponse(categorized) }],
        isError: true,
      };
    }
  }
);
```

### Layer 4: Backend (Data Access)

**Local JSONL**: Parse errors on individual JSONL lines are logged and skipped (degraded, not failed). A malformed line does not abort the entire query.

**SigNoz API**: Network errors are caught by the circuit breaker. After 3 consecutive failures, all requests fail fast for 60 seconds without making network calls.

### Layer 5: Server Init (Startup)

**Implementation**: `src/lib/server-utils.ts:209-218`

`ServerInitError` tracks which initialization step failed:

| Step | What Fails |
|------|-----------|
| `tool-validation` | Duplicate tool names, missing schemas |
| `server-creation` | MCP SDK instantiation failure |
| `handler-registration` | Failed to register tools/call handler |
| `transport-connection` | Stdio transport connection failure |

Init errors are fatal -- the server exits with a non-zero code.

---

## 4. Self-Instrumentation

### OTel Spans for Error Tracking

Every tool invocation is wrapped in an OTel span (`src/server.ts:187-217`). Error spans include:

- `error: true` attribute
- `error.type`: Error class name
- `error.message`: Sanitized message
- `error.code`: CategorizedError code (if applicable)

### OTel Metrics

**Implementation**: `src/lib/metrics.ts`

| Metric | Type | Description |
|--------|------|-------------|
| `obs_toolkit.tool.duration` | Histogram | Tool execution time (ms) |
| `obs_toolkit.tool.errors` | Counter | Error count by tool and error code |
| `obs_toolkit.cache.hits` | Counter | Cache hit count |
| `obs_toolkit.cache.misses` | Counter | Cache miss count |
| `obs_toolkit.cache.evictions` | Counter | LRU eviction count |

### Structured Logging

**Implementation**: `src/lib/logger.ts`

Internal errors are logged with structured context (timestamp, level, error code, category) rather than plain `console.error`. In production, logs are emitted as OTel log records for correlation with traces.

---

## 5. Error Tracking vs. Sentry

The observability-toolkit does **not** use Sentry. Error tracking is handled through:

1. **OTel self-instrumentation**: Errors are recorded as span events and exported to SigNoz Cloud (or local JSONL) via the same telemetry pipeline the toolkit monitors.
2. **Structured error types**: `CategorizedError` provides the classification that Sentry tags/contexts would provide.
3. **Error sanitization**: Replaces Sentry's data scrubbing with a purpose-built sanitizer tuned for MCP server constraints.

This "eat your own dog food" approach means the toolkit's own errors appear in the same observability data it queries, enabling self-monitoring through its own MCP tools.

### Comparison with Sentry Patterns

| Sentry Pattern | Toolkit Equivalent |
|----------------|-------------------|
| `Sentry.captureException(error)` | OTel span event with error attributes |
| `Sentry.withScope(scope => ...)` | OTel span attributes and resource context |
| `scope.setTag('route', ...)` | Span attribute `mcp.tool.name` |
| `scope.setUser({ id })` | Resource attribute `service.instance.id` |
| `Sentry.startSpan(...)` | OTel `trace.startSpan(...)` via instrumentation.ts |
| Error levels (fatal/error/warn) | Error category (system/network/user/config) |
| Breadcrumbs | OTel span events with timestamps |
| Performance monitoring | OTel histograms (`obs_toolkit.tool.duration`) |

---

## 6. Graceful Shutdown

**Implementation**: `src/server.ts:239-263`

Signal handlers for SIGTERM and SIGINT:
1. Call `shutdownInstrumentation()` to flush pending OTel spans/metrics
2. Close stdio transport
3. Exit with code 0

This ensures in-flight telemetry (including error spans) is exported before the process terminates.

---

## Related Documents

| Document | Relevance |
|----------|-----------|
| [backend-data-storage-and-scalability.md](backend-data-storage-and-scalability.md) | Circuit breaker, rate limiter, error recovery |
| [caching-design.md](caching-design.md) | Cache miss handling and error propagation |
| [quality-metrics-dashboard.md](quality-metrics-dashboard.md) | Alert thresholds and health status |
| [../quality/quality-evaluation.md](../quality/quality-evaluation.md) | Evaluation error handling |

---

## Implementation Files

| File | Description |
|------|-------------|
| `src/lib/error-types.ts` | Error categories, codes, retryability (200 lines) |
| `src/lib/error-sanitizer.ts` | Path stripping, cloud detection, response sanitization (267 lines) |
| `src/lib/server-utils.ts` | Rate limiter, ServerInitError (218 lines) |
| `src/lib/circuit-breaker.ts` | Circuit breaker for SigNoz backend (181 lines) |
| `src/lib/metrics.ts` | OTel error metrics counters |
| `src/lib/logger.ts` | Structured logging |
| `src/lib/instrumentation.ts` | OTel SDK setup, shutdown |
| `src/server.ts` | Request error handling pipeline, graceful shutdown |
