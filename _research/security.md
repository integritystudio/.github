# Security

Security controls and hardening measures for observability-toolkit.

## Overview

The observability-toolkit implements defense-in-depth with ten layers:
1. **Query Escaping** - SQL injection prevention for ClickHouse queries
2. **Memory Limits** - DoS prevention through result capping and size estimation
3. **Input Validation** - Strict limits on user-provided parameters
4. **SSRF Protection** - URL validation for external API connections
5. **ReDoS Prevention** - Regex pattern escaping to prevent CPU exhaustion
6. **Rate Limiting** - Token bucket rate limiter for API abuse prevention
7. **Symlink Protection** - Path traversal prevention via symlink validation
8. **Error Sanitization** - Prevents information disclosure in error messages
9. **Duration Validation** - Bounds checking on time-based query parameters
10. **LLM-as-Judge Security** - Prompt injection, input limits, timeout, circuit breaker

## Query Escaping

**File**: `src/lib/query-sanitizer.ts`

### Functions

| Function | Purpose |
|----------|---------|
| `escapeClickHouseString(value)` | Escapes quotes, backslashes, null bytes, newlines |
| `escapeClickHouseLike(value)` | Escapes LIKE wildcards (%, _) |
| `sanitizeIdentifier(name)` | Allows only alphanumeric, underscore, dot |
| `containsDangerousPattern(input)` | Returns true if dangerous SQL pattern found |
| `validateQueryInput(input, fieldName)` | Throws if dangerous pattern found |
| `escapeFilterValueSafe(value, fieldName)` | Validates then escapes for WHERE clauses |
| `escapeLikeValueSafe(value, fieldName)` | Validates then escapes for LIKE clauses |

### Input Length Limit

```typescript
const MAX_QUERY_INPUT_LENGTH = 10000;  // Defense in depth
```

Inputs exceeding 10,000 characters are rejected as dangerous. This prevents:
- Performance issues from O(n) operations on very long strings
- Potential DoS via excessive memory allocation
- Edge cases in regex processing

Warning logged: `[SECURITY] Input length X exceeds maximum Y`

### Dangerous Pattern Blocklist

22 patterns are blocklisted (case-insensitive, word-boundary matched):

```
DROP, DELETE, INSERT, UPDATE, ALTER, CREATE, TRUNCATE,
--, /*, */, ;, UNION, INTO OUTFILE, INTO DUMPFILE,
LOAD_FILE, SYSTEM, ATTACH, DETACH, RENAME, OPTIMIZE, GRANT, REVOKE
```

**Word boundary detection** reduces false positives (e.g., "DROPDOWN" does not match "DROP").

### ReDoS Prevention

Regex patterns are escaped before use in `RegExp` constructor:

```typescript
function escapeRegexPattern(pattern: string): string {
  return pattern.replace(/[.*+?^${}()|[\]\\]/g, '\\$&');
}
```

This prevents catastrophic backtracking if patterns contain regex metacharacters.

### Integration

All user inputs in `src/backends/signoz-api.ts` are sanitized:
- `traceId`, `serviceName`, `severity` → `escapeFilterValueSafe()`
- `spanName`, `search` → `escapeLikeValueSafe()`
- `metricName`, `groupBy` fields → `sanitizeIdentifier()` with validation

## Memory Limits

**Files**: `src/lib/constants.ts`, `src/lib/file-utils.ts`

### Constants

| Constant | Value | Purpose |
|----------|-------|---------|
| `MAX_RESULTS_IN_MEMORY` | 10,000 | Hard cap on result count in memory |
| `MAX_RESULTS_BYTE_SIZE` | 50 MB | Hard cap on estimated result size |
| `STREAMING_THRESHOLD` | 5,000 | Switch to streaming aggregation above this |

### StreamingAggregator

Computes aggregations incrementally without holding all values:
- Supported: sum, avg, min, max, count, p50, p95, p99
- Uses **reservoir sampling** (1000 samples) for percentile approximation
- Trade-off: ~1-2% accuracy loss for unlimited data size

### Memory Enforcement

```typescript
enforceMemoryLimit<T>(results: T[], limit?: number): {
  results: T[];
  truncated: boolean;
  originalCount: number;
  truncationReason?: 'count' | 'size';
}
```

**Dual protection**:
1. **Count limit** - Truncates if array length exceeds `MAX_RESULTS_IN_MEMORY`
2. **Size limit** - Truncates if estimated byte size exceeds `MAX_RESULTS_BYTE_SIZE`

### Byte Size Estimation

```typescript
function estimateByteSize<T>(results: T[], sampleSize = 100): number
```

- For arrays ≤100 items: measures directly via `JSON.stringify()`
- For larger arrays: samples every Nth item and extrapolates
- Trade-off: ~5% accuracy loss for O(sample_size) vs O(n) performance
- Binary search finds safe truncation point when size limit exceeded

When truncation occurs:
- Response includes `truncated: true`, `originalCount`, and `truncationReason`
- Warning logged: `[MEMORY] Results truncated from X to Y items`

### Integration

All 5 query tools enforce memory limits:
- `query-traces.ts`
- `query-logs.ts`
- `query-metrics.ts` (+ streaming aggregation)
- `query-llm-events.ts`
- `query-evaluations.ts` (+ streaming aggregation)

## Input Validation

**File**: `src/lib/input-validator.ts`

### Limits

| Constant | Value | Purpose |
|----------|-------|---------|
| `MAX_LIMIT` | 1,000 | Maximum results per query |
| `MAX_DATE_RANGE_DAYS` | 365 | Maximum date range span |
| `MAX_REGEX_LENGTH` | 200 | Maximum regex pattern length |
| `MAX_REGEX_GROUPS` | 10 | Maximum regex capture groups |
| `DEFAULT_LIMIT` | 50 | Default when limit not specified |

### Validators

| Function | Behavior |
|----------|----------|
| `validateLimit(limit)` | Clamps to 1-1000, defaults to 50 |
| `validateDateRange(start?, end?)` | Validates YYYY-MM-DD format, enforces max range |
| `validateRegexPattern(pattern)` | Checks length, group count, compilation |

### Error Handling

```typescript
class InputValidationError extends Error {
  field: string;      // Which input field failed
  constraint: string; // What constraint was violated
}
```

### Integration

All 5 query tools validate inputs before processing:
- Limit validation on all queries
- Date range validation when dates provided
- Regex validation for pattern/search inputs

## Data Validation

Additional validation implemented throughout:

### Numeric Safety
- `extractValidNumber()` - Rejects NaN, Infinity, non-numeric types
- `extractValidString()` - Rejects non-string types
- Score range: Infinity values rejected in evaluations
- min/max: Returns `undefined` when no valid scores (not Infinity)

### Type Coercion Prevention
- String literal unions for type safety
- Explicit type assertions on all response fields
- `normalizeFinishReasons()` handles edge cases (single string, empty array)
- `normalizeScoreLabel()` treats empty string as undefined

## SSRF Protection

**File**: `src/backends/signoz-api.ts`

### URL Validation

```typescript
function validateAndSanitizeUrl(url: string): string
```

The `SIGNOZ_QUERY_URL` environment variable and constructor `baseUrl` parameter are validated:

| Check | Blocked Values |
|-------|----------------|
| Protocol | `http://`, `file://`, `ftp://` (only HTTPS allowed) |
| IPv4 Localhost | `localhost`, `127.0.0.1`, `0.0.0.0` |
| IPv6 Localhost | `::1`, `0:0:0:0:0:0:0:1`, `::ffff:127.0.0.1`, `::ffff:7f00:1`, `::ffff:127.*` |
| IPv4 Private | `10.x.x.x`, `172.16-31.x.x`, `192.168.x.x` |
| IPv6 Private | `fc00::*`, `fd00::*` (ULA), `fe80::*` (link-local) |
| IPv4-mapped Private | `::ffff:10.*`, `::ffff:172.16-31.*`, `::ffff:192.168.*` |
| Reserved Domains | `*.local`, `*.internal`, `*.localdomain`, `*.home.arpa`, `*.localhost` |

### IPv6 Normalization

Hostnames are normalized before validation:
- Brackets stripped: `[::1]` → `::1`
- Case normalized: lowercase comparison
- All IPv6 formats checked (compact, full, IPv4-mapped)

### Sanitization

- Preserves **whitelisted path prefixes** for backward compatibility: `/v1/`, `/api/`, `/signoz/`, `/query/`
- Strips query parameters and fragments (always)
- Non-whitelisted paths: returns only origin (protocol + host + port)
- Invalid URLs return empty string (fail closed)

### Security Logging

```
[SECURITY] Invalid URL protocol - only HTTPS allowed
[SECURITY] Internal network addresses not allowed for SigNoz URL
[SECURITY] Invalid SIGNOZ_QUERY_URL format
```

## Rate Limiting

**File**: `src/backends/signoz-api.ts`

### Token Bucket Rate Limiter

```typescript
class TokenBucketRateLimiter {
  constructor(maxTokens?: number, refillRate?: number)
  tryConsume(): boolean      // Consume 1 token, returns false if none available
  getAvailableTokens(): number
  refund(): void             // Return 1 token (used when circuit breaker blocks)
  reset(): void              // Restore to full capacity (testing only)
}
```

| Parameter | Default | Environment Variable |
|-----------|---------|---------------------|
| `maxTokens` | 60 | `RATE_LIMIT_MAX_TOKENS` |
| `refillRate` | 1/sec | `RATE_LIMIT_REFILL_RATE` |

### Behavior

- Allows **burst traffic** up to `maxTokens` requests
- Refills tokens at `refillRate` per second
- Integrated with `ApiCircuitBreaker.canRequest()`
- Returns `false` when rate limited (no tokens available)

### Circuit Breaker Integration

Combined with circuit breaker for dual protection:
- **Rate limit exceeded**: Too many requests in short window
- **Circuit breaker open**: Backend unavailable/failing

When circuit breaker is open and rejects a request:
1. Token was already consumed by `tryConsume()`
2. `refund()` returns the single token (not `reset()` which would restore all)
3. Prevents attackers from using circuit breaker state to bypass rate limiting

## Symlink Protection

**File**: `src/lib/constants.ts`

### Path Validation

```typescript
function isSymlink(path: string): boolean
function isPathWithinAllowedBases(resolvedPath: string, allowedBasePaths: string[]): boolean
```

### Allowed Base Directories

- Current working directory (resolved)
- `~/.claude/` (user's Claude config directory)

### Behavior

1. Detects symlinks using `lstatSync()`
2. Resolves symlink target using `realpathSync()`
3. Validates resolved path is within allowed bases
4. **Rejects** symlinks pointing outside allowed directories

### Security Logging

```
[SECURITY] Rejected symlink traversal: /path/to/symlink -> /etc/passwd
```

## Error Sanitization

**File**: `src/lib/error-sanitizer.ts`

### Functions

| Function | Purpose |
|----------|---------|
| `sanitizeErrorMessage(error)` | Removes file paths from error messages |
| `sanitizeErrorForResponse(error)` | Prepares sanitized error for MCP responses |
| `sanitizePath(path)` | Converts absolute paths to relative format |

### Path Removal Patterns

- Unix paths: `/Users/`, `/home/`, `/var/`, `/opt/`, `/tmp/`
- Windows paths: `C:\`, `D:\`
- Home directory: `~/`
- Node.js internals: `node:internal/`

### Safe Patterns (Pass Through)

Validation errors are considered safe and pass through unchanged:
- "Invalid", "must be", "required", "cannot be"
- "Rate limit", "Circuit breaker"

### Integration

Applied in:
- `src/server.ts` - MCP error responses
- `src/backends/local-jsonl.ts` - Health check errors
- `src/backends/signoz-api.ts` - API error responses

## Duration Validation

**File**: `src/lib/input-validator.ts`

### Constants

| Constant | Value | Purpose |
|----------|-------|---------|
| `MAX_DURATION_MS` | 86,400,000 (24h) | Maximum duration value |

### Validator

```typescript
function validateDuration(value: number | undefined, fieldName: string): number | undefined
```

| Input | Output |
|-------|--------|
| `undefined` | `undefined` |
| Valid number (0 to max) | Same value |
| Exceeds max | Clamped to `MAX_DURATION_MS` |
| Negative | Throws `InputValidationError` |

### Integration

Applied to `query-traces.ts`:
- `minDurationMs` parameter
- `maxDurationMs` parameter

## LLM-as-Judge Security

**File**: `src/lib/llm-as-judge.ts`

### Prompt Injection Protection

14 patterns detected (case-insensitive, Unicode-normalized):
```
ignore previous instructions, system prompt, you are now, forget everything,
disregard previous/prior, new instructions:, override system/instructions,
act as if you are, pretend you are, jailbreak, DAN, developer mode,
ignore safety, bypass filter/restriction/rule
```

**Homoglyph Normalization** (Unicode TR39):
- Uses `confusables` library for full coverage
- Cyrillic "а" (U+0430) → Latin "a"
- Greek "ο" (U+03BF) → Latin "o"
- Full-width "Ａ" (U+FF21) → ASCII "A"

**Zero-Width Character Removal**:
```
U+200B-U+200D (zero-width space/joiner/non-joiner)
U+2060 (word joiner), U+180E (Mongolian vowel separator)
U+FEFF (BOM), U+034F (combining grapheme joiner)
U+FE00-U+FE0F (variation selectors)
```

### Input Size Limits

| Constant | Value | Purpose |
|----------|-------|---------|
| `MAX_INPUT_SIZE_BYTES` | 64 KB | Total test case size |
| `MAX_TEXT_LENGTH` | 10,000 | Per-field character limit |
| `MAX_CONTEXT_ITEMS` | 20 | Context array length |
| `MAX_STATEMENTS` | 20 | QAG statement extraction |
| `MAX_JSON_DEPTH` | 5 | JSON parsing nesting |

### Timeout Protection

```typescript
async function withTimeout<T>(
  fn: (signal: AbortSignal) => Promise<T>,
  timeoutMs: number = 30000
): Promise<T>
```

- Uses `AbortController` for atomic cancellation
- Prevents race conditions between completion and timeout
- Throws `LLMTimeoutError` on timeout

### Circuit Breaker

```typescript
class JudgeCircuitBreaker {
  constructor(threshold?: number, resetTimeout?: number)
  evaluate<T>(evaluate: () => Promise<T>, fallback?: () => Promise<T>): Promise<T>
}
```

| Default | Value |
|---------|-------|
| Threshold | 5 failures |
| Reset timeout | 30 seconds |

**Rate Limit Exclusion**: 429 errors not counted toward threshold:
- Error names: `RateLimitError`, `ThrottlingException`, `TooManyRequestsError`
- Status codes: 429
- AWS codes: `ThrottlingException`, `ProvisionedThroughputExceededException`

### ReDoS Prevention

All prompt injection patterns use:
- Non-capturing groups `(?:...)`
- No nested quantifiers
- Word boundary matching via `search()` (avoids `lastIndex` issues)

## Test Coverage

| Module | Tests |
|--------|-------|
| `query-sanitizer.test.ts` | 54 tests (includes max length) |
| `input-validator.test.ts` | 39 tests (includes duration) |
| `error-sanitizer.test.ts` | 47 tests |
| `constants.test.ts` (symlink) | 5 tests |
| `signoz-api.test.ts` (rate limit) | 12 tests (includes refund) |
| `signoz-api.test.ts` (SSRF/URL) | 22 tests (includes IPv6) |
| Memory limits (in `file-utils.test.ts`) | ~20 tests |
| `llm-as-judge.test.ts` | 81+ tests |

**Total security-related tests**: 280+

## Security Decisions

1. **Validate before escape** - `escapeFilterValueSafe()` validates for dangerous patterns first, then escapes, preventing injection after escaping.

2. **Word boundaries** - Dangerous pattern matching uses word boundaries to reduce false positives while maintaining security.

3. **Pass-through for missing values** - `validateDateRange()` allows open-ended queries when either date is missing (returns current date as default).

4. **Reservoir sampling trade-off** - ~1-2% accuracy loss on percentiles is acceptable for memory safety with large datasets.

5. **HTTPS-only for SigNoz** - Production SigNoz instances should always use HTTPS. HTTP blocked to prevent credential leakage over unencrypted connections.

6. **Origin-only URLs** - `validateAndSanitizeUrl()` returns only the origin, stripping path/query/fragment to prevent parameter injection attacks.

7. **Dual memory limits** - Both count (10k items) and size (50MB) limits enforced. Large objects with few items could bypass count-only limits.

8. **Sample-based sizing** - For arrays >100 items, byte size is estimated via sampling. Trade-off: ~5% accuracy for O(100) vs O(n) performance.

9. **Token bucket rate limiting** - Allows burst traffic (60 requests) with gradual refill (1/sec). Configurable via environment variables for different deployment scenarios.

10. **Symlink resolution** - All paths resolved to real paths before use. Symlinks pointing outside allowed directories (working dir, ~/.claude/) are rejected with security warnings.

11. **Error message sanitization** - File paths, stack traces, and internal details stripped from user-facing errors. Validation errors pass through unchanged for debugging.

12. **Duration clamping** - Values exceeding MAX_DURATION_MS (24h) are clamped rather than rejected, preventing DoS while maintaining usability. Negative values are rejected.

13. **URL path whitelist** - Safe path prefixes (/v1/, /api/, /signoz/, /query/) preserved for backward compatibility with custom deployments. Unknown paths stripped for security.

14. **Rate limiter refund vs reset** - When circuit breaker blocks a request, only the single consumed token is refunded. Using `reset()` (restore all tokens) would allow attackers to bypass rate limiting by triggering circuit breaker state.

15. **Input length limits** - 10KB max for query inputs (MAX_QUERY_INPUT_LENGTH). Defense in depth against very long inputs even though regex patterns are safe. O(n) operations on 100MB strings could cause performance issues.

16. **Comprehensive IPv6 SSRF protection** - All IPv6 formats checked: compact (`::1`), full (`0:0:0:0:0:0:0:1`), IPv4-mapped (`::ffff:127.0.0.1`). Brackets normalized before comparison.

17. **Prompt injection silent filtering** - `sanitizeForPrompt()` replaces patterns with `[filtered]` rather than throwing, allowing evaluation to proceed with sanitized input. Fail-fast can be implemented by callers using the exported `PromptInjectionError` class.

18. **Homoglyph detection before pattern matching** - Detection uses normalized text (Cyrillic→Latin) while preserving original text when no injection found. This catches homoglyph attacks without destroying legitimate non-Latin content.

19. **AbortController for timeout atomicity** - Uses abort signal as single source of truth to prevent race conditions between function completion and timeout firing. Both paths check `signal.aborted` before acting.

20. **Rate limit exclusion from circuit breaker** - 429 errors are transient and shouldn't open the circuit. Detection uses error name, status code, and message patterns for comprehensive provider coverage.
