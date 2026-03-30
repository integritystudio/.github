# Internal Observability Architecture

**Version**: 1.8.6
**Last Updated**: 2026-02-01

---

## Overview

The observability-toolkit provides comprehensive self-monitoring capabilities through internal observability. This enables visibility into the toolkit's own operations including query performance, cache efficiency, backend health, and resource utilization.

## Architecture

```
┌──────────────────────────────────────────────────────────────────────────────┐
│                          observability-toolkit                                │
├──────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  ┌────────────────┐  ┌────────────────┐  ┌────────────────────────────────┐  │
│  │    Server      │  │    Backends    │  │           Tools                │  │
│  │                │  │                │  │                                │  │
│  │  Rate Limiter  │  │  LocalJsonl    │  │  obs_health_check              │  │
│  │  Error Handler │  │  SigNozApi     │  │  obs_query_*                   │  │
│  └───────┬────────┘  └───────┬────────┘  └───────────────┬────────────────┘  │
│          │                   │                           │                   │
│          ▼                   ▼                           ▼                   │
│  ┌───────────────────────────────────────────────────────────────────────┐   │
│  │                    Internal Observability Layer                        │   │
│  ├─────────────┬─────────────┬─────────────┬─────────────┬───────────────┤   │
│  │ Instrumen-  │   Metrics   │   Cache     │  Parse      │   Histogram   │   │
│  │ tation      │   Export    │   Stats     │  Stats      │   Tracking    │   │
│  │             │             │             │             │               │   │
│  │ OTel spans  │ OTel meters │ hit/miss/   │ success/    │ p50/p95/p99   │   │
│  │ withSpan()  │ counters    │ eviction    │ failure     │ latency       │   │
│  └─────────────┴─────────────┴─────────────┴─────────────┴───────────────┘   │
│          │                                                                   │
│          ▼                                                                   │
│  ┌───────────────────────────────────────────────────────────────────────┐   │
│  │                         OTLP Export (opt-in)                           │   │
│  │                    → SigNoz / any OTLP backend                         │   │
│  └───────────────────────────────────────────────────────────────────────┘   │
└──────────────────────────────────────────────────────────────────────────────┘
```

---

## Core Components

### 1. OTel Self-Instrumentation

**Location**: `src/lib/instrumentation.ts`

Opt-in OpenTelemetry SDK integration for tracing toolkit operations.

#### InstrumentationManager

Encapsulates OTel SDK state with singleton pattern for production, isolated instances for testing.

```typescript
import { withSpan, initializeInstrumentation } from './lib/instrumentation.js';

// Initialize on server start
await initializeInstrumentation();

// Wrap operations with spans
const result = await withSpan('queryTraces', { backend: 'local' }, async (span) => {
  // operation logic
  return results;
});
```

#### Features

| Feature | Description |
|---------|-------------|
| Singleton pattern | Production singleton, test isolation (A1) |
| Retry with backoff | Exponential backoff on init failure (M1) |
| Status tracking | `waitForInitialization()` for async status (M1-B4) |
| Attribute truncation | 128 char keys, 1024 char values per OTel spec (H4) |
| SDK timeout | 5s default to prevent blocking (H3) |

#### Status API

```typescript
import { getInitializationStatus, waitForInitialization } from './lib/instrumentation.js';

const status = getInitializationStatus();
// { status: 'ready' | 'pending' | 'initializing' | 'disabled' | 'failed', error?: string }

const result = await waitForInitialization();
```

---

### 2. Metrics Export

**Location**: `src/lib/metrics.ts`

OTel metrics for internal observability when `OTEL_ENABLED=true`.

#### Exported Metrics

| Metric | Type | Labels | Description |
|--------|------|--------|-------------|
| `obs_toolkit.cache.hits` | Counter | `cache_type` | Cache hit count |
| `obs_toolkit.cache.misses` | Counter | `cache_type` | Cache miss count |
| `obs_toolkit.cache.evictions` | Counter | `cache_type` | LRU eviction count |
| `obs_toolkit.query.duration` | Histogram | `query_type`, `backend` | Query latency (ms) |
| `obs_toolkit.rate_limit.rejections` | Counter | `limiter` | Rate-limited requests |
| `obs_toolkit.circuit_breaker.state` | Histogram | `circuit_breaker`, `state` | State transitions |

#### Usage

```typescript
import { recordCacheHit, recordQueryDuration, recordCircuitBreakerState } from './lib/metrics.js';

recordCacheHit('traces');
recordQueryDuration('traces', 123.5, 'local');
recordCircuitBreakerState('signoz-api', 'open');
```

All functions are no-ops when metrics disabled.

---

### 3. Query Cache

**Location**: `src/lib/cache.ts`

LRU cache with TTL for query results.

#### CacheStats Interface

```typescript
interface CacheStats {
  hits: number;
  misses: number;
  evictions: number;
  size: number;
  hitRate: number;  // hits / (hits + misses)
}
```

#### Tracked Caches

| Cache | Contents |
|-------|----------|
| `traceCache` | Trace query results |
| `logCache` | Log query results |
| `metricCache` | Metric query results |
| `llmEventCache` | LLM event query results |

#### API

```typescript
class QueryCache<T> {
  constructor(maxSize?: number, ttlMs?: number);
  get(key: string): T | undefined;
  set(key: string, data: T): void;
  has(key: string): boolean;
  clear(): void;
  get size(): number;
  getStats(): CacheStats;
}
```

---

### 4. Parse Statistics

**Location**: `src/lib/parse-stats.ts`

Tracks JSONL parse success/failure/empty counts per file with LRU eviction.

#### ParseStatsTracker

```typescript
import { getGlobalParseStatsTracker } from './lib/parse-stats.js';

const tracker = getGlobalParseStatsTracker();

tracker.recordBatch('/path/to/file.jsonl', { parsed: 100, skipped: 5, empty: 10 });

const stats = tracker.getAggregateStats();
```

#### AggregateParseStats

```typescript
interface AggregateParseStats {
  totalFiles: number;
  totalLines: number;
  totalParsed: number;
  totalSkipped: number;
  parseSuccessRate: number;  // 0-1
  worstFiles: FileParseStats[];  // Top 5 by skip rate
}
```

---

### 5. Response Time Histograms

**Location**: `src/lib/histogram.ts`

Histogram with reservoir sampling for accurate percentile calculation.

#### Features

- O(1) bucket assignment via binary search
- Reservoir sampling (max 10,000 values) for percentile accuracy
- OTel-aligned default bucket boundaries
- Memory-bounded (~80KB per histogram)

#### Usage

```typescript
import { Histogram, DEFAULT_LATENCY_BUCKETS } from './lib/histogram.js';

const histogram = new Histogram(DEFAULT_LATENCY_BUCKETS);
histogram.observe(45.2);

const stats = histogram.getStats();
// { min, max, mean, p50, p95, p99, count }

const buckets = histogram.getBuckets();
// { buckets: [10, 50, ...], counts: [5, 12, ...], sum, count }
```

#### Default Buckets

```typescript
const DEFAULT_LATENCY_BUCKETS = [10, 50, 100, 250, 500, 1000, 2500, 5000, 10000];
```

---

### 6. Circuit Breaker

**Location**: `src/backends/signoz-api.ts`

Protects SigNoz API backend with circuit breaker pattern.

#### State Transitions

| Transition | Level | Message |
|------------|-------|---------|
| closed → open | WARN | `Circuit breaker OPENED after N consecutive failures` |
| open → half-open | INFO | `Circuit breaker entering HALF-OPEN state` |
| half-open → open | WARN | `Circuit breaker OPENED after N consecutive failures` |
| half-open → closed | INFO | `Circuit breaker CLOSED after successful request` |

---

## Configuration

### Environment Variables

| Variable | Description | Default |
|----------|-------------|---------|
| `OTEL_ENABLED` | Enable OTel instrumentation | `false` |
| `OTEL_EXPORTER_OTLP_ENDPOINT` | OTLP collector URL | - |
| `OTEL_SERVICE_NAME` | Service name for traces | `observability-toolkit` |
| `CACHE_TTL_MS` | Cache entry TTL | `60000` (1 min) |

### Constants

| Constant | Value | Location |
|----------|-------|----------|
| `SLOW_QUERY_THRESHOLD_MS` | 500 | backends |
| `MAX_CACHE_SIZE` | 100 | `QueryCache` |
| `CIRCUIT_MAX_FAILURES` | 3 | `CircuitBreaker` |
| `CIRCUIT_RESET_MS` | 30000 | `CircuitBreaker` |
| `MAX_RESERVOIR_SIZE` | 10000 | `Histogram` |
| `MAX_TRACKED_FILES` | 100 | `ParseStatsTracker` |

---

## Health Check Output

**Location**: `src/tools/health-check.ts`

The `obs_health_check` tool returns comprehensive status:

```json
{
  "status": "ok",
  "backends": {
    "local": { "status": "ok", "fileCount": 12 },
    "signoz": { "status": "ok", "circuitState": "closed" }
  },
  "cache": {
    "traces": { "hits": 156, "misses": 23, "hitRate": 0.871, "size": 45 },
    "logs": { "hits": 89, "misses": 34, "hitRate": 0.723, "size": 100 },
    "metrics": { "hits": 12, "misses": 8, "hitRate": 0.6, "size": 20 },
    "llmEvents": { "hits": 5, "misses": 2, "hitRate": 0.714, "size": 7 }
  },
  "today": "2026-02-01"
}
```

---

## Operations Guide

### Interpreting Cache Metrics

| Hit Rate | Status | Action |
|----------|--------|--------|
| >80% | Excellent | Cache effective |
| 50-80% | Good | Normal operation |
| 20-50% | Fair | Consider increasing TTL |
| <20% | Poor | Review query patterns |

| Eviction Rate | Status | Action |
|---------------|--------|--------|
| 0 | Normal | Cache not full |
| <10% | Normal | Healthy turnover |
| >30% | Warning | Increase max size |

### Interpreting Query Latency

| Duration | Status | Action |
|----------|--------|--------|
| <500ms | Normal | - |
| 500-1000ms | Moderate | Monitor |
| 1000-5000ms | Slow | Check file sizes |
| >5000ms | Critical | Add date filters, check regex |

### Debugging Checklist

**High cache miss rate:**
1. Check query specificity (too unique?)
2. Check TTL (too short?)
3. Check eviction rate (cache too small?)

**Slow queries:**
1. Check telemetry file sizes
2. Narrow date range filters
3. Check regex patterns for backtracking
4. Consider adding indexes

**Circuit breaker opening:**
1. Verify SigNoz connectivity
2. Validate API key
3. Check SigNoz service health
4. Review network conditions

---

## File Structure

```
src/lib/
├── instrumentation.ts    # OTel SDK management
├── metrics.ts            # OTel metrics export
├── cache.ts              # Query cache with stats
├── parse-stats.ts        # JSONL parse tracking
├── histogram.ts          # Response time histograms
├── circuit-breaker.ts    # Circuit breaker pattern
└── logger.ts             # Structured logging
```
