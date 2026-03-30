# Caching Design

**Version**: 1.0
**Date**: 2026-02-08
**Scope**: observability-toolkit v2.0.1
**Purpose**: Caching architecture, LRU implementation, cache key strategies, and performance characteristics.

---

## 1. Cache Architecture Overview

The toolkit uses two LRU caches at different layers: a query result cache and a regex pattern cache.

```
MCP Tool Request
    ↓
Zod Validation
    ↓
┌──────────────────────────────────────────┐
│           Query Cache (LRU)              │
│  5 instances × 100 entries = 500 max     │
│  TTL: 60s (configurable)                 │
├──────────────────────────────────────────┤
│  traceCache   │ logCache   │ metricCache │
│  llmEventCache│ evalCache  │             │
└───────┬──────────────────────────────────┘
        │ miss
        ▼
┌──────────────────────────────────────────┐
│         Regex Pattern Cache (LRU)        │
│  1 instance × 100 compiled patterns      │
│  No TTL (patterns don't change)          │
└───────┬──────────────────────────────────┘
        │
        ▼
    Backend Query (JSONL scan or SigNoz API)
```

**Key design decision**: In-memory LRU only, no Redis. The toolkit is a single-process MCP server with no shared state requirement. LRU provides bounded memory with zero external dependencies.

---

## 2. Query Cache

### Implementation

**File**: `src/lib/cache.ts` (170 lines)

```typescript
class QueryCache<T> {
  private cache: Map<string, CacheEntry<T>>;
  private maxSize: number;      // Default: 100
  private ttlMs: number;        // Default: 60,000 (CACHE_TTL_MS env var)
  private hits: number;
  private misses: number;
  private evictions: number;
}
```

### LRU Eviction Strategy (`cache.ts:110-129`)

The cache exploits JavaScript `Map` insertion order for O(1) LRU behavior:

1. **On cache hit**: Delete entry, re-insert at end (moves to most-recently-used position)
2. **On capacity exceeded**: Delete first entry from Map iterator (least-recently-used)
3. **On TTL expiry**: Checked at read time; expired entries return as misses

```typescript
// Hit: refresh position
get(key: string): T | undefined {
  const entry = this.cache.get(key);
  if (!entry) { this.misses++; return undefined; }
  if (Date.now() - entry.timestamp > this.ttlMs) {
    this.cache.delete(key);
    this.misses++;
    return undefined;
  }
  // Move to end (most recently used)
  this.cache.delete(key);
  this.cache.set(key, entry);
  this.hits++;
  return entry.data;
}

// Evict: remove oldest on overflow
set(key: string, data: T): void {
  if (this.cache.has(key)) {
    this.cache.delete(key);
  } else if (this.cache.size >= this.maxSize) {
    const firstKey = this.cache.keys().next().value;
    if (firstKey) {
      this.cache.delete(firstKey);
      this.evictions++;
    }
  }
  this.cache.set(key, { data, timestamp: Date.now() });
}
```

### Cache Instances

The `MultiDirectoryBackend` creates 5 independent cache instances:

| Cache | Stores | Max Entries |
|-------|--------|-------------|
| `traceCache` | `TraceSpan[]` query results | 100 |
| `logCache` | `LogRecord[]` query results | 100 |
| `metricCache` | `MetricDataPoint[]` query results | 100 |
| `llmEventCache` | `LLMEvent[]` query results | 100 |
| `evaluationCache` | `EvaluationResult[]` query results | 100 |

**Total memory bound**: 500 cached result sets. Each entry is the full result array for a query, so actual memory depends on result sizes (bounded by `MAX_RESULTS_IN_MEMORY` = 10,000 per result set).

### Cache Key Generation (`cache.ts:167-169`)

```typescript
function makeCacheKey(prefix: string, options: object): string {
  return `${prefix}:${JSON.stringify(options)}`;
}

// Examples:
// "traces:{"startDate":"2026-02-08","limit":50}"
// "evaluations:{"evaluationName":"relevance","scoreMin":0.5}"
```

**Properties**:
- Deterministic: identical query parameters always produce the same key
- Prefix-scoped: different data types never collide
- JSON serialization handles nested objects and arrays

**Limitation**: Parameter order matters. `{a:1, b:2}` and `{b:2, a:1}` produce different keys. This is acceptable because query parameters come from Zod-validated objects with consistent property order.

---

## 3. Regex Pattern Cache

### Implementation

**File**: `src/backends/local-jsonl.ts:89-118`

A separate LRU cache for compiled regular expressions, preventing repeated compilation of the same patterns (e.g., span name filters used across multiple queries).

| Property | Value |
|----------|-------|
| Max entries | 100 |
| TTL | None (patterns are immutable) |
| Key | Raw regex string |
| Value | Compiled `RegExp` object |

**Error handling**: Invalid regex patterns are caught, logged, and return `null` (no match) rather than throwing. This prevents a malformed `spanNameRegex` parameter from crashing the query.

---

## 4. Cache Observability

### OTel Metrics (`src/lib/metrics.ts`)

| Metric | Type | Labels | Description |
|--------|------|--------|-------------|
| `obs_toolkit.cache.hits` | Counter | `cache_name` | Successful cache lookups |
| `obs_toolkit.cache.misses` | Counter | `cache_name` | Cache misses (not found or expired) |
| `obs_toolkit.cache.evictions` | Counter | `cache_name` | LRU evictions due to capacity |

### Statistics API (`cache.ts:152-161`)

```typescript
interface CacheStats {
  hits: number;       // Successful retrievals
  misses: number;     // Key not found or expired
  evictions: number;  // LRU evictions due to capacity
  size: number;       // Current number of entries
  hitRate: number;    // hits / (hits + misses), 0.0 - 1.0
}
```

Accessible via `getCacheStats()` on each backend instance. The `obs_health_check` tool reports aggregate cache statistics when available.

### Target Performance

| Metric | Target | Rationale |
|--------|--------|-----------|
| Hit rate | >60% | Dashboard queries repeat within TTL window |
| Eviction rate | <10% of sets | If high, increase `maxSize` or reduce query diversity |
| P99 lookup | <1ms | Map-based O(1) operations |

---

## 5. Cache Invalidation

### TTL-Based Expiry

The primary invalidation mechanism. Entries expire after `CACHE_TTL_MS` (default 60 seconds).

**Why 60 seconds**: Balances freshness with performance. Telemetry data is append-only (JSONL files grow but don't mutate existing lines), so a 60-second window means at most 60 seconds of missing recent data.

### No Explicit Invalidation via MCP

The cache has a `clear()` method internally but no MCP tool exposes cache invalidation. Rationale:
- Telemetry data is append-only; past results don't become incorrect
- New data appears on the next cache miss after TTL expiry
- Explicit invalidation via MCP tools would add complexity for a marginal freshness improvement

### Cache Bypass

The `backend` parameter on query tools forces backend selection but does **not** bypass the cache. To get uncached results, a query must either:
1. Wait for TTL expiry
2. Use different parameters (different cache key)
3. Restart the MCP server process

---

## 6. Memory Budget

### Worst-Case Calculation

```
5 caches × 100 entries × 10,000 results/entry × ~200 bytes/result
= 5 × 100 × 10,000 × 200
= 1 GB (theoretical maximum)
```

**In practice**: Most queries return 50-200 results (default limit). Realistic worst case:

```
5 × 100 × 200 × 200 bytes = 20 MB
```

This is well within the typical Node.js heap budget for a CLI-launched MCP server.

### Tuning

| Env Var | Default | Effect |
|---------|---------|--------|
| `CACHE_TTL_MS` | 60000 | Shorter = fresher data, more backend queries |
| Cache `maxSize` | 100 | Compile-time constant; increase for diverse query patterns |

---

## 7. Design Decisions

### Why LRU, Not Redis

| Factor | LRU Cache | Redis |
|--------|-----------|-------|
| Deployment | Single process (MCP server) | Requires Redis server |
| Persistence | None needed (telemetry is source of truth) | Unnecessary overhead |
| Shared state | Not needed (1:1 host-server) | Would add complexity |
| Latency | <1ms (in-process Map) | 1-5ms (network round trip) |
| Dependencies | Zero | ioredis package + running Redis |
| Memory bound | Configurable maxSize | Requires separate monitoring |

**Verdict**: LRU is the correct choice for a single-process, zero-dependency MCP server.

### Why Not Cache-Aside with Write-Through

The toolkit is read-heavy: MCP tools query telemetry data but never write it (writes come from OTel SDK exporters). Cache-aside (read: check cache, miss: query backend, fill cache) is the natural fit. Write-through would require intercepting OTel exports, which is out of scope.

### Why Separate Cache Per Data Type

Rather than one large cache, 5 small caches provide:
- **Type safety**: `QueryCache<TraceSpan[]>` vs `QueryCache<LogRecord[]>` prevents type confusion
- **Independent sizing**: Trace queries may be more frequent than metric queries
- **Isolated eviction**: Heavy trace querying doesn't evict evaluation cache entries
- **Cleaner statistics**: Per-cache hit rates identify which data types benefit most from caching

---

## Related Documents

| Document | Relevance |
|----------|-----------|
| [backend-data-storage-and-scalability.md](backend-data-storage-and-scalability.md) | Memory bounds, streaming thresholds, query performance |
| [error-tracking-design.md](error-tracking-design.md) | Cache miss error propagation |
| [quality-metrics-dashboard.md](quality-metrics-dashboard.md) | Dashboard queries that benefit from caching |

---

## Implementation Files

| File | Description |
|------|-------------|
| `src/lib/cache.ts` | QueryCache class, LRU implementation, statistics (170 lines) |
| `src/backends/local-jsonl.ts:89-118` | Regex pattern cache |
| `src/lib/metrics.ts` | Cache OTel metrics (hits, misses, evictions) |
| `src/lib/constants.ts` | CACHE_TTL_MS, MAX_RESULTS_IN_MEMORY |
| `src/backends/index.ts` | Backend interface with `getCacheStats?()` |
