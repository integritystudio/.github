# Quality Metrics Dashboard

**Version**: 2.6.0
**Status**: Production
**Last Updated**: 2026-02-10

## Overview

Programmatic quality monitoring across 7 pre-defined LLM evaluation metrics with configurable alert thresholds. Now includes a React dashboard for visual monitoring.

- **Entry point**: `computeDashboardSummary()` consumes evaluation results grouped by metric name
- **Output**: `QualityDashboardSummary` with aggregated values, triggered alerts, and health status
- **Extensible**: `MetricConfigBuilder` fluent API for custom metrics
- **Integrations**: SigNoz, Grafana, Datadog
- **Dashboard**: React 19 + Vite 6 localhost UI with 3 role-based views (v2.5)

## Architecture

```
┌──────────────────────────────────────────────────────────────────────────────┐
│                     Quality Metrics Dashboard Pipeline                       │
├──────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  Data Source                                                                 │
│  ├── EvaluationResult[] from JSONL backend / obs_query_evaluations           │
│  └── Map<string, EvaluationResult[]> keyed by metric name                    │
│                                                                              │
│         │                                                                    │
│         ▼                                                                    │
│  ┌────────────────────────────────────────────────────────────────────────┐  │
│  │                    computeDashboardSummary()                            │  │
│  │                                                                        │  │
│  │  For each metric in QUALITY_METRICS + customMetrics:                    │  │
│  │  ┌───────────────┐  ┌───────────────────┐  ┌───────────────────────┐  │  │
│  │  │ Extract       │  │ Compute           │  │ Check Alert           │  │  │
│  │  │ scoreValue[]  │─▶│ Aggregations      │─▶│ Thresholds            │  │  │
│  │  │               │  │ (avg,p50,p95,...) │  │ (warning/critical)    │  │  │
│  │  └───────────────┘  └───────────────────┘  └──────────┬────────────┘  │  │
│  │                                                        │              │  │
│  │                                            ┌───────────▼───────────┐  │  │
│  │                                            │ Determine Health      │  │  │
│  │                                            │ Status                │  │  │
│  │                                            └───────────────────────┘  │  │
│  └────────────────────────────────────────────────────────────────────────┘  │
│         │                                                                    │
│         ▼                                                                    │
│  Output: QualityDashboardSummary                                             │
│  ├── overallStatus (worst of all metrics)                                    │
│  ├── metrics[] (QualityMetricResult per metric)                              │
│  ├── alerts[] (all triggered alerts with metricName)                         │
│  └── summary { totalMetrics, healthyMetrics, warningMetrics, ... }           │
│                                                                              │
└──────────────────────────────────────────────────────────────────────────────┘
```

## Built-in Metrics

7 pre-defined metrics with recommended thresholds based on industry best practices for LLM evaluation. Available as the exported constant `QUALITY_METRICS: Record<string, QualityMetricConfig>`.

| Metric | Display Name | Unit | Range | Aggregations |
|--------|-------------|------|-------|--------------|
| `relevance` | Response Relevance | score | 0-1 | avg, p50, p95, min, count |
| `task_completion` | Task Completion Rate | rate | 0-1 | avg, p50, count |
| `tool_correctness` | Tool Selection Accuracy | rate | 0-1 | avg, p50, count |
| `hallucination` | Hallucination Rate | rate | 0-1 | avg, p95, max, count |
| `evaluation_latency` | Evaluation Latency | seconds | 0-60 | avg, p50, p95, p99, max, count |
| `faithfulness` | Response Faithfulness | score | 0-1 | avg, p50, p95, count |
| `coherence` | Response Coherence | score | 0-1 | avg, p50, p95, count |

## Aggregation System

| Aggregation | Description | Algorithm |
|-------------|-------------|-----------|
| `avg` | Arithmetic mean | sum / count |
| `min` | Minimum value | First element of sorted array |
| `max` | Maximum value | Last element of sorted array |
| `count` | Sample count | Array length (always computed when data exists) |
| `p50` | 50th percentile (median) | R-7 linear interpolation |
| `p95` | 95th percentile | R-7 linear interpolation |
| `p99` | 99th percentile | R-7 linear interpolation |

Percentile calculation: `rank = (percentile / 100) * (n - 1)`, interpolate between `sorted[floor(rank)]` and `sorted[ceil(rank)]`. All values except `count` are rounded to 4 decimal places.

## Alert Thresholds

### Warning Thresholds

| Metric | Aggregation | Direction | Threshold | Message Template |
|--------|-------------|-----------|-----------|-----------------|
| `relevance` | p50 | below | 0.7 | Relevance p50 ({value}) below 0.7 threshold |
| `task_completion` | avg | below | 0.85 | Task completion rate ({value}) below 85% target |
| `tool_correctness` | avg | below | 0.95 | Tool correctness ({value}) below 95% target |
| `hallucination` | avg | above | 0.1 | Hallucination rate ({value}) above 10% threshold |
| `evaluation_latency` | p95 | above | 5.0 | Evaluation latency p95 ({value}s) exceeds 5s target |
| `faithfulness` | p50 | below | 0.8 | Faithfulness p50 ({value}) below 0.8 threshold |
| `coherence` | p50 | below | 0.75 | Coherence p50 ({value}) below 0.75 threshold |

### Critical Thresholds

| Metric | Aggregation | Direction | Threshold | Message Template |
|--------|-------------|-----------|-----------|-----------------|
| `relevance` | p50 | below | 0.5 | Relevance p50 ({value}) critically low |
| `task_completion` | avg | below | 0.70 | Task completion rate ({value}) critically low |
| `tool_correctness` | avg | below | 0.85 | Tool correctness ({value}) critically low |
| `hallucination` | avg | above | 0.2 | Hallucination rate ({value}) critically high |
| `evaluation_latency` | p95 | above | 10.0 | Evaluation latency p95 ({value}s) critically high |
| `faithfulness` | p50 | below | 0.6 | Faithfulness p50 ({value}) critically low |

Coherence has no critical threshold by default; add one via `MetricConfigBuilder` if needed.

### Alert Direction

- **below**: Fires when the computed value is less than the threshold. Used for quality metrics where higher is better (relevance, faithfulness, task completion, tool correctness, coherence).
- **above**: Fires when the computed value exceeds the threshold. Used for metrics where lower is better (hallucination rate, evaluation latency).

The `{value}` placeholder in message templates is replaced with the actual value formatted to 4 decimal places.

Triggered alerts within each metric are sorted by severity: critical first, then warning, then info.

### Health Status Hierarchy

After evaluating all thresholds, each metric receives a health status:

| Priority | Status | Condition |
|----------|--------|-----------|
| 1 | `no_data` | No evaluation scores available |
| 2 | `critical` | Any triggered alert has severity `critical` |
| 3 | `warning` | Any triggered alert has severity `warning` |
| 4 | `healthy` | No alerts triggered, data present |

The dashboard `overallStatus` is the worst status across all metrics. If any metric is `critical`, the dashboard is `critical`. If all metrics have no data, the dashboard is `no_data`.

## Core Interfaces

```typescript
// src/lib/quality/quality-metrics.ts

type AlertSeverity = 'info' | 'warning' | 'critical';
type ThresholdDirection = 'above' | 'below';

interface QualityMetricConfig {
  name: string;                    // Metric name (matches gen_ai.evaluation.name)
  displayName: string;             // Human-readable display name
  description: string;             // Description for dashboard tooltips
  aggregations: EvaluationAggregation[];  // Functions to compute
  alerts: AlertThreshold[];        // Alert thresholds
  range: { min: number; max: number };    // Expected score range
  unit: 'score' | 'rate' | 'seconds' | 'percentage';
}

interface AlertThreshold {
  aggregation: EvaluationAggregation;  // Which aggregation to monitor
  value: number;                        // Threshold value
  direction: 'above' | 'below';        // Alert condition direction
  severity: 'info' | 'warning' | 'critical';
  message: string;                      // Template with {value} placeholder
}

interface QualityMetricResult {
  name: string;
  displayName: string;
  values: Record<EvaluationAggregation, number | null>;  // Computed aggregations
  sampleCount: number;             // Number of evaluations used
  alerts: TriggeredAlert[];        // Alerts that fired
  status: 'healthy' | 'warning' | 'critical' | 'no_data';
  period?: { start: string; end: string };
}

interface TriggeredAlert {
  severity: 'info' | 'warning' | 'critical';
  message: string;                 // Formatted message with actual value
  aggregation: EvaluationAggregation;
  threshold: number;               // Configured threshold
  actualValue: number;             // Current value
  direction: 'above' | 'below';
}

interface QualityDashboardSummary {
  overallStatus: 'healthy' | 'warning' | 'critical' | 'no_data';
  metrics: QualityMetricResult[];
  alerts: Array<TriggeredAlert & { metricName: string }>;
  summary: {
    totalMetrics: number;
    healthyMetrics: number;
    warningMetrics: number;
    criticalMetrics: number;
    noDataMetrics: number;
  };
  timestamp: string;               // ISO timestamp when computed
}
```

## Usage Examples

### Computing a Dashboard Summary

```typescript
import { computeDashboardSummary } from './lib/quality-metrics.js';
import type { EvaluationResult } from './backends/index.js';

const evaluationsByMetric = new Map<string, EvaluationResult[]>();

evaluationsByMetric.set('relevance', [
  { timestamp: '2026-02-06T10:00:00Z', evaluationName: 'relevance', scoreValue: 0.85 },
  { timestamp: '2026-02-06T10:01:00Z', evaluationName: 'relevance', scoreValue: 0.92 },
  { timestamp: '2026-02-06T10:02:00Z', evaluationName: 'relevance', scoreValue: 0.78 },
]);

evaluationsByMetric.set('hallucination', [
  { timestamp: '2026-02-06T10:00:00Z', evaluationName: 'hallucination', scoreValue: 0.05 },
  { timestamp: '2026-02-06T10:01:00Z', evaluationName: 'hallucination', scoreValue: 0.08 },
]);

const dashboard = computeDashboardSummary(evaluationsByMetric);
// dashboard.overallStatus: 'healthy'
// dashboard.summary: { totalMetrics: 7, healthyMetrics: 2, noDataMetrics: 5, ... }
// dashboard.alerts: []

// With optional parameters:
const period = { start: '2026-02-06T00:00:00Z', end: '2026-02-06T23:59:59Z' };
const dashboardWithPeriod = computeDashboardSummary(evaluationsByMetric, undefined, period);
```

### Creating a Custom Metric

```typescript
import {
  createMetricConfig,
  registerQualityMetric,
  getAllQualityMetrics,
  computeDashboardSummary,
} from './lib/quality-metrics.js';

const toxicity = createMetricConfig('toxicity')
  .displayName('Toxicity Score')
  .description('Measures harmful or toxic content in responses')
  .aggregations('avg', 'p50', 'p95', 'max', 'count')
  .range(0, 1)
  .unit('score')
  .alertAbove('avg', 0.1, 'warning', 'Toxicity avg ({value}) above 10% threshold')
  .alertAbove('avg', 0.25, 'critical', 'Toxicity avg ({value}) critically high')
  .build();

// Option 1: Register in module-scoped registry (for getQualityMetric / getAllQualityMetrics)
registerQualityMetric(toxicity);

// Option 2: Pass custom metrics directly to computeDashboardSummary
const dashboard = computeDashboardSummary(evaluationsByMetric, { toxicity });

// Bridge: use the registry to feed custom metrics into the dashboard
const allMetrics = getAllQualityMetrics();
const dashboardFromRegistry = computeDashboardSummary(evaluationsByMetric, allMetrics);
```

Note: `registerQualityMetric` populates a module-scoped registry for `getQualityMetric` / `getAllQualityMetrics`. It does **not** automatically feed into `computeDashboardSummary`. Pass custom metrics explicitly via the second parameter, or bridge with `getAllQualityMetrics()`.

### Interpreting Alerts

```typescript
import {
  computeDashboardSummary,
  formatMetricValue,
  getQualityMetric,
} from './lib/quality-metrics.js';

const dashboard = computeDashboardSummary(evaluationsByMetric);

for (const alert of dashboard.alerts) {
  console.log(`[${alert.severity.toUpperCase()}] ${alert.metricName}: ${alert.message}`);
  // [CRITICAL] relevance: Relevance p50 (0.4500) critically low
}

for (const metric of dashboard.metrics) {
  const config = getQualityMetric(metric.name);
  if (config && metric.values.avg !== null) {
    console.log(`${metric.displayName}: ${formatMetricValue(metric.values.avg, config.unit)}`);
  }
}
```

## Edge Cases

- **Empty data**: Metrics with no evaluations return `no_data` status and null aggregation values
- **Null scores**: Evaluations where `scoreValue` is undefined or null are filtered out before aggregation
- **Empty Map**: Calling `computeDashboardSummary(new Map())` returns all 7 built-in metrics with `no_data` status
- **Duplicate registration**: `registerQualityMetric()` throws if metric name matches a built-in or already-registered custom metric
- **Unregister built-in**: `unregisterQualityMetric('relevance')` returns `false` because built-in metrics are not stored in the custom registry
- **Registry scope**: The custom metric registry is module-scoped. In multi-worker environments, each worker maintains its own registry

## Value Formatting

`formatMetricValue()` produces display strings based on unit type:

| Unit | Format | Example Input | Example Output |
|------|--------|---------------|----------------|
| `score` | 4 decimal places | 0.8567 | `0.8567` |
| `rate` | Percentage, 1 decimal | 0.95 | `95.0%` |
| `percentage` | Percentage, 1 decimal | 0.85 | `85.0%` |
| `seconds` | 2 decimals + "s" | 3.456 | `3.46s` |
| (null) | N/A literal | null | `N/A` |

## MetricConfigBuilder API

Fluent builder for creating custom `QualityMetricConfig` objects. Defaults: `aggregations: ['avg', 'count']`, `range: { min: 0, max: 1 }`, `unit: 'score'`, `alerts: []`.

| Method | Signature | Description |
|--------|-----------|-------------|
| `displayName()` | `(name: string): this` | Set display name |
| `description()` | `(desc: string): this` | Set description |
| `aggregations()` | `(...aggs: EvaluationAggregation[]): this` | Set aggregations to compute |
| `range()` | `(min: number, max: number): this` | Set expected value range |
| `unit()` | `(unit: 'score' \| 'rate' \| 'seconds' \| 'percentage'): this` | Set measurement unit |
| `alertBelow()` | `(agg, value, severity, message?): this` | Alert when aggregation falls below value |
| `alertAbove()` | `(agg, value, severity, message?): this` | Alert when aggregation exceeds value |
| `build()` | `(): QualityMetricConfig` | Finalize and return config |

## Computation Functions

| Function | Signature | Description |
|----------|-----------|-------------|
| `computeDashboardSummary` | `(evaluationsByMetric: Map<string, EvaluationResult[]>, customMetrics?: Record<string, QualityMetricConfig>, period?: { start: string; end: string }) => QualityDashboardSummary` | Compute dashboard across all built-in + custom metrics. |
| `computeQualityMetric` | `(evaluations: EvaluationResult[], config: QualityMetricConfig, period?: { start: string; end: string }) => QualityMetricResult` | Compute a single metric from evaluation results. NaN/Infinity scores are filtered before aggregation and status determination. |
| `computeAggregations` | `(scores: number[], aggregations: EvaluationAggregation[]) => Record<EvaluationAggregation, number \| null>` | Compute aggregation values from raw scores. |
| `checkAlertThresholds` | `(values: Record<EvaluationAggregation, number \| null>, thresholds: AlertThreshold[]) => TriggeredAlert[]` | Check values against thresholds, return triggered alerts (sorted by severity). |
| `determineHealthStatus` | `(alerts: TriggeredAlert[], hasData: boolean) => 'healthy' \| 'warning' \| 'critical' \| 'no_data'` | Derive health status from triggered alerts. |
| `formatMetricValue` | `(value: number \| null, unit: QualityMetricConfig['unit']) => string` | Format a metric value for display. |
| `createMetricConfig` | `(name: string) => MetricConfigBuilder` | Create a fluent builder for custom metric configs. |
| `computePipelineView` | `(evaluationsByMetric: Map<string, EvaluationResult[]>) => PipelineResult` | Compute pipeline stage progression with within-stage drop-off metrics. |
| `computeCoverageHeatmap` | `(evaluationsByMetric: Map<string, EvaluationResult[]>, options?: CoverageHeatmapOptions \| string) => CoverageHeatmap` | Compute metric coverage matrix with gap identification. Supports `traceId` or `sessionId` grouping with configurable thresholds. |

## Metric Registration API

| Function | Signature | Description |
|----------|-----------|-------------|
| `registerQualityMetric` | `(config: QualityMetricConfig) => void` | Register custom metric. Validates via Zod schema. Throws if name exists. |
| `unregisterQualityMetric` | `(name: string) => boolean` | Remove custom metric. Returns `true` if removed. Cannot remove built-in metrics. |
| `getAllQualityMetrics` | `() => Record<string, QualityMetricConfig>` | Get all metrics (built-in + custom). |
| `getQualityMetric` | `(name: string) => QualityMetricConfig \| undefined` | Get a specific metric by name. Checks built-in first, then custom. |

Zod validation via the exported `qualityMetricConfigSchema` enforces: `name` (1-100 chars), `displayName` (1-200 chars), `description` (0-1000 chars, empty string allowed), `aggregations` (min 1 entry), `message` (max 500 chars).

## Files

### Core Library

All tests in `src/lib/quality/quality-metrics.test.ts`.

| File | Lines | Description |
|------|-------|-------------|
| `src/lib/quality/quality-metrics.ts` | 1,586 | Metric configs, aggregation, alerting, builder, registration, trends, SLA, roles, pipeline, coverage |
| `src/lib/quality/quality-metrics.test.ts` | 3,365 | Test suite (207+ tests) |

### React Dashboard (v2.5)

| File | Description |
|------|-------------|
| `dashboard/src/api/server.ts` | Hono API on 127.0.0.1:3001 (dev only), CORS for localhost:5173 |
| `dashboard/src/api/data-loader.ts` | MultiDirectoryBackend singleton, evaluation grouping |
| `dashboard/src/api/routes/dashboard.ts` | GET /api/dashboard (period + role), GET /api/health |
| `dashboard/src/api/routes/metrics.ts` | GET /api/metrics/:name (topN + bucketCount) |
| `dashboard/src/hooks/useDashboard.ts` | TanStack Query with 30s refetch, stale-while-revalidate |
| `dashboard/src/hooks/useMetricDetail.ts` | Lazy metric detail query |
| `dashboard/src/components/MetricCard.tsx` | React.memo'd metric card with status, trend, confidence |
| `dashboard/src/components/MetricGrid.tsx` | CSS Grid auto-fit responsive layout |
| `dashboard/src/components/HealthOverview.tsx` | Status banner + summary counters |
| `dashboard/src/components/AlertList.tsx` | Severity-sorted alerts with remediation hints |
| `dashboard/src/components/SLATable.tsx` | SLA compliance rows with gap/margin |
| `dashboard/src/components/ScoreHistogram.tsx` | Pure CSS histogram (max 20 buckets) |
| `dashboard/src/components/EvaluationDetail.tsx` | Worst/best evaluations table |
| `dashboard/src/components/Indicators.tsx` | StatusBadge, TrendIndicator, ConfidenceBadge |
| `dashboard/src/components/views/*.tsx` | Executive, Operator, Auditor role views |
| `dashboard/src/App.tsx` | Wouter routes, period selector, loading/error/empty states |
| `dashboard/src/__tests__/components.test.tsx` | 24 component tests (Vitest + Testing Library) |
| `dashboard/scripts/derive-evaluations.ts` | Rule-based evaluation derivation from traces |

## Test Coverage

| Category | Tests | Added |
|----------|-------|-------|
| Pre-defined metrics (QUALITY_METRICS) | 6 | v2.0 |
| Aggregation computation | 10 | v2.0 |
| Alert threshold checking | 6 | v2.0 |
| Health status determination | 4 | v2.0 |
| Single metric computation | 5 | v2.0 |
| Dashboard summary | 5 | v2.0 |
| Metric registration | 7 | v2.0 |
| Value formatting | 5 | v2.0 |
| MetricConfigBuilder | 2 | v2.0 |
| Enriched alerts + correlation | 32 | v2.1 |
| Trend analysis + confidence + SLA + roles + multi-agent | 65 | v2.2 |
| Configurable thresholds + glob matching + integration | 26 | v2.3 |
| Validation + agreement range + SLA status + edge cases | 14 | v2.4 |
| NaN filtering, SLA invariant, pipeline visualization, coverage heatmap | 16 | v2.6 |
| **Total** | **207** | |

### Dashboard Component Tests (v2.6)

24 tests in `dashboard/src/__tests__/components.test.tsx` using Vitest + @testing-library/react:

| Component | Tests |
|-----------|-------|
| StatusBadge | 3 |
| TrendIndicator | 4 |
| ConfidenceBadge | 2 |
| HealthOverview | 5 |
| AlertList | 5 |
| SLATable | 5 |

## React Dashboard (v2.5)

Visual quality monitoring interface at `localhost:5173` (dev) / `integritystudio.dev` (production), backed by a Hono API at `127.0.0.1:3001` in dev or the Cloudflare Worker (`quality-metrics-api`) in production.

### Architecture

```
Browser (:5173)           Hono API (:3001)          ~/.claude/telemetry/
React 19 + Vite 6   -->  @hono/node-server    -->  evaluations-*.jsonl
TanStack Query            Zod validation            (with .idx sidecars)
Wouter routing            bound to 127.0.0.1
```

### API Endpoints

| Endpoint | Params | Response |
|----------|--------|----------|
| `GET /api/dashboard` | `period=24h\|7d\|30d`, `role=executive\|operator\|auditor` | `QualityDashboardSummary` or role view |
| `GET /api/metrics/:name` | `topN=1-50`, `bucketCount=2-20` | `MetricDetailResult` with histogram |
| `GET /api/health` | none | `{ status, hasData }` |

### Data Flow

1. `computePeriodDates(period)` -> start/end ISO strings
2. `loadEvaluationsByMetric(start, end)` -> `Map<string, EvaluationResult[]>` (limit: 10,000)
3. `computeDashboardSummary(grouped, customMetrics, period)` -> `QualityDashboardSummary`
4. Optional: `computeRoleView(dashboard, role)` -> role-specific view

### Key Design Decisions

| Decision | Rationale |
|----------|-----------|
| TanStack Query | Smart caching, stale-while-revalidate, retry with backoff, background refetch |
| Wouter routing | 1.5KB vs React Router 20KB, deep-linkable URLs |
| React.memo on MetricCard | Prevents 7+ re-renders per 30s poll cycle |
| CSS Grid auto-fit | `repeat(auto-fit, minmax(300px, 1fr))` responsive without media queries |
| Hono on 127.0.0.1 | Localhost-only, prevents accidental network exposure |
| Pure CSS histograms | 10-20 buckets don't justify a charting library |
| Import from parent `dist/` | Avoids type drift, uses same types as MCP server |

### Telemetry Derivation

`dashboard/scripts/derive-evaluations.ts` derives evaluation records from `traces-*.jsonl`:

| Derived Metric | Source Spans | Derivation Logic |
|----------------|-------------|------------------|
| `tool_correctness` | `hook:builtin-post-tool`, `hook:mcp-post-tool` | `success === true` -> 1.0, else 0.0 |
| `evaluation_latency` | Measurable hook spans | Duration in seconds from `span.duration` |
| `task_completion` | `TaskCreate`/`TaskUpdate` per session | `min(updates / (creates * 2), 1.0)` + agent pre/post ratio |

Output format: OTel flat records with `gen_ai.evaluation.*` attributes, compatible with `normalizeEvaluation()` in `backends/local-jsonl.ts`.

### Routes

| Path | View |
|------|------|
| `/` | Full dashboard: health overview + 7 metric cards + alerts + SLA |
| `/role/:roleName` | Executive, Operator, or Auditor view |
| `/metrics/:metricName` | Metric detail with histogram + worst/best evaluations |

## Pipeline Visualization (v2.6)

`computePipelineView()` models evaluation flow as sequential stages with within-stage drop-off metrics.

```typescript
interface PipelineStage {
  metric: string;
  entryCount: number;   // Evaluations entering this stage
  exitCount: number;    // Evaluations with finite scores
}

interface PipelineDropoff {
  fromStage: string;
  toStage: string;
  count: number;        // entryCount - exitCount (within-stage drop)
  percent: number;      // Drop percentage relative to entry
}

interface PipelineResult {
  stages: PipelineStage[];
  dropoffs: PipelineDropoff[];
  overallConversion: number;  // First stage exit / first stage entry
}
```

Stages are ordered by entry count (descending). Drop-off is computed within each stage (entry vs exit), representing evaluations that lack valid scores at that stage.

## Coverage Heatmap (v2.6)

`computeCoverageHeatmap()` generates a metric-by-input coverage matrix identifying evaluation gaps.

```typescript
type CoverageStatus = 'covered' | 'partial' | 'missing';

interface CoverageCell {
  metric: string;
  inputId: string;
  count: number;
  status: CoverageStatus;  // >= coveredThreshold: covered, >= partialThreshold: partial, else missing
}

interface CoverageGap {
  metric: string;
  missingInputs: string[];
}

interface CoverageHeatmap {
  cells: CoverageCell[];
  gaps: CoverageGap[];
  overallCoverage: number;  // Fraction of cells that are 'covered'
}

interface CoverageHeatmapOptions {
  inputKey?: 'traceId' | 'sessionId';  // Default: 'traceId'
  coveredThreshold?: number;            // Default: 1
  partialThreshold?: number;            // Default: 1 (no partial by default)
}
```

Supports backward-compatible string API: `computeCoverageHeatmap(data, 'sessionId')` is equivalent to `computeCoverageHeatmap(data, { inputKey: 'sessionId' })`.

---

## Planned Extensions

Planned interface extensions based on the [Interface Research Index](README.md) consolidation. Approach: **Hybrid (E)** for v2.1 (enriched JSON + LLM grounding), evolving to **Dashboard Data Contract (C)** for v2.2 (typed progressive disclosure).

### v2.1: Enriched Alerts and Correlation — COMPLETE

**`QualityMetricResult` additions** — COMPLETE (commit 74bbada). Addresses [G2](quality-dashboard-ux-review.md#2-no-narrative-explainability-in-alerts), [R1](llm-explainability-research.md#priority-1-explanation-quality-and-display-high-impact-low-effort):

```typescript
interface QualityMetricResult {
  // ... existing fields ...

  // Representative explanation from lowest-scoring evaluation
  worstExplanation?: {
    scoreValue: number;
    explanation: string;
    traceId?: string;
    timestamp: string;
  };
}
```

**`TriggeredAlert` additions** — COMPLETE (commit b90d14d, fix 996d60a). Addresses [G2](quality-dashboard-ux-review.md#2-no-narrative-explainability-in-alerts), [G6](quality-dashboard-ux-review.md#6-no-remediation-guidance), [G10](quality-dashboard-ux-review.md#10-no-evidence-trail), [R1](llm-explainability-research.md#priority-1-explanation-quality-and-display-high-impact-low-effort), [R6](llm-explainability-research.md#priority-6-regulatory-compliance-enhancements-medium-impact-low-effort):

```typescript
interface TriggeredAlert {
  // ... existing fields ...

  affectedCount?: number;           // Evaluations contributing to this alert
  remediationHints?: string[];      // Actionable next steps
  relatedMetrics?: string[];        // Other metrics also degraded
}
```

Alert message templates will include sample count context ([R4.2](llm-explainability-research.md#priority-4-dashboard-explainability-enhancements-medium-impact-low-effort)):
```
// Before: "Relevance p50 (0.4500) critically low"
// After:  "Relevance p50 (0.4500) critically low (n=47 evaluations, last 1h)"
```

**Cross-metric correlation** — COMPLETE (commit 8009f86, fix 1396727). Addresses [G1](quality-dashboard-ux-review.md#1-no-cross-metric-correlation-toxic-combinations):

```typescript
interface MetricCorrelationRule {
  name: string;                     // e.g., "quality_crisis"
  displayName: string;              // e.g., "Compound Quality Risk"
  conditions: MetricCondition[];    // Multiple metric thresholds, ALL must match
  severity: 'warning' | 'critical';
  explanation: string;              // Why this combination matters
}
```

Post-processes in `computeDashboardSummary()` after individual metric evaluation, emitting compound alerts when all conditions match. 3 default rules: `content_quality_crisis`, `agent_reliability_failure`, `response_degradation`.

### v2.2: Progressive Disclosure and Confidence

**Progressive disclosure types** (addresses [G3](quality-dashboard-ux-review.md#3-only-2-levels-of-progressive-disclosure), [R4](llm-explainability-research.md#priority-4-dashboard-explainability-enhancements-medium-impact-low-effort)):

| Level | Type | Content |
|-------|------|---------|
| L1 (exists) | `QualityDashboardSummary` | Overall status, metric count, alert count |
| L2 (exists) | `QualityMetricResult[]` | Per-metric aggregations, alerts, status |
| L3 (new) | `MetricDetailResult` | Top failing evaluations, agent breakdown, temporal distribution |
| L4 (new) | `EvaluationDetailResult` | Full evaluation context: response, evaluator, scores, metadata |
| L5 (new) | Raw data | JSONL records, trace IDs, session IDs (queryable via `obs_query_evaluations`) |

**Confidence indicators** (addresses [R3](llm-explainability-research.md#priority-3-confidence-indicators-medium-impact-medium-effort)):
- Logprob-based confidence from `normalizeWithLogprobs()`
- Multi-judge agreement from `panelEvaluation()`

**Multi-agent metrics** (addresses [R5](llm-explainability-research.md#priority-5-multi-agent-explainability-high-impact-high-effort)):
- `handoff_correctness` added to built-in `QUALITY_METRICS`
- Turn-level evaluation support in agent-as-judge

## Related Documentation

- [Quality Evaluation Architecture](../quality/quality-evaluation.md) - Evaluation storage and export
- [LLM-as-Judge Architecture](../quality/llm-as-judge.md) - G-Eval, QAG, bias mitigation
- [Agent-as-Judge Architecture](../quality/agent-as-judge.md) - Multi-agent evaluation patterns
- [Dashboard Configuration](../visibility/dashboard-configuration.md) - OTel metrics dashboards
- [Alert Configuration](../visibility/alert-configuration.md) - External alerting rules
- [Interface Research Index](README.md) - Cross-document analysis, approach comparison, recommendation consolidation
