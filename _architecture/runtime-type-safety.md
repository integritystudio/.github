# Runtime Type Safety with Zod Schemas

This document describes the runtime validation strategy using Zod schemas across the observability-toolkit MCP server.

## Overview

**60+ Zod schemas** validate all critical inputs at system boundaries, ensuring type safety and preventing corruption from malformed data.

## Schema Inventory

### File I/O & Persistence (3 files, 8 schemas)

#### `src/lib/observability/indexer.ts`
- **`telemetryTypeSchema`**: Enum for 5 telemetry types (`traces`, `logs`, `metrics`, `llm-events`, `evaluations`)
- **`indexEntrySchema`**: Index entry with optional OTel fields (traceId, spanName, serviceName, agentId, toolName, operationName, evaluationName)
- **`fileIndexSchema`**: FileIndex with literal version=1, sourceFile, sourceMtime, entries array
- **Integration**: `readIndex()` validates with `fileIndexSchema.safeParse()` — prevents corruption on file-read

#### `src/lib/audit/audit-record.ts`
- **`auditRecordCategorySchema`**: Enum for 4 audit categories (`trace`, `evaluation`, `verification`, `system`)
- **`auditRecordSchema`**: Full audit record with UUID, timestamp, SHA-256 hashes (`/^[a-f0-9]{64}$/`), and correlations
- **`auditRecordInputSchema`**: Input without auto-generated fields (id, timestamp, hashes)
- **`auditQueryOptionsSchema`**: Query filter options (category, sessionId, traceId, evaluationName, date range, limit)
- **`chainIntegrityResultSchema`**: Integrity check result (valid, recordCount, brokenAt, error)
- **Integration**:
  - `createAuditRecord()`: uses `auditRecordInputSchema.safeParse()` — replaces manual sessionId/category guards
  - `queryAuditRecords()`: validates each record with `auditRecordSchema.safeParse()` on deserialization
  - `getLastRecordHash()`: uses `auditRecordSchema.safeParse()` — prevents unsafe `as` cast on file-read

### Configuration Objects (2 files, 6 schemas)

#### `src/lib/cost/cost-estimation.ts`
- **`tokenUsageSchema`**: Input tokens, output tokens with cache variants
- **`budgetConfigSchema`**: Daily/session/user limits, alert/critical thresholds, max attributions, retention days
- **`costBreakdownSchema`**: Per-request cost (input/output/total), model, fallback flag
- **`alertLevelSchema`**: Enum (`ok`, `warning`, `critical`, `exceeded`)
- **`budgetCheckResultSchema`**: Check result with level, message, currentSpend, limit, percentUsed, remainingBudget
- **`costAttributionSchema`**: Session/user cost record with timestamp, tokens, model, metadata
- **`costSummarySchema`**: Aggregate summary (totalCost, sessionCount, userCount, attributions)
- **Integration**: `createBudgetTracker()` validates config with `budgetConfigSchema.safeParse()` — replaces manual field extraction

#### `src/lib/judge/llm-judge-config.ts`
- **`llmJudgeLoggerSchema`**: Function properties (debug, info, warn, error) with passthrough
- **`llmJudgeConfigSchema`**: Timeout (max 120000ms), maxRetries (max 10), logger, evaluator string, evaluatorType enum
- **`guardrailResultSchema`**: Score 0-1, boolean blocked, optional reason
- **`hallucinationGuardrailOptionsSchema`**: Optional threshold, lazy self-reference to judge config
- **`configuredEvaluationResultSchema`**: Score, optional reason, metadata with evaluator/evaluatorType/timeoutMs/retries
- **`metaEvalGuardSchema`**: Boolean isMetaEval flag
- **Integration**: `resolveConfig()` validates config with `llmJudgeConfigSchema.safeParse()` — type-safe EvaluatorType enum validation

### Evaluation Inputs (2 files, 7 schemas)

#### `src/lib/judge/llm-as-judge.ts`
- **`evaluationEventSchema`**: OTel evaluation event (timestamp, evaluationName, scoreValue OR scoreLabel via refinement, error handling fields)
- **`testCaseSchema`**: Evaluation test case (input, output strings, optional context array, expectedOutput)
- **`evalResultSchema`**: Judge output (score 0-1, reason, optional rawResponse, optional confidence 0-1)
- **`gEvalConfigSchema`**: G-Eval criteria (name, criteria, evaluationParams array, optional temperature)
- **`pairwiseResultSchema`**: Pairwise comparison (winner A/B/tie, explanation, confidence 0-1)
- **Integration**: `gEval()` in `llm-judge-geval.ts` validates both config and testCase at function entry point

### State Persistence (2 files, 4 schemas)

#### `src/lib/quality/quality-feature-engineering.ts`
- **`percentileDistributionSchema`**: Percentile values (p10, p25, p50, p75, p90)
- **`degradationStateSchema`**: Degradation state (lastRun, breaches Record<string, number>)
- **`calibrationStateSchema`**: Calibration state (lastCalibrated, distributions nested object with distribution/sampleSize/windowStart/windowEnd, optional psiValues, optional rawScores)
- **`psiResultSchema`**: PSI result (psi value, drifted boolean)
- **Integration**:
  - `loadDegradationState()`: uses `degradationStateSchema.safeParse()` — replaces manual type guards
  - `loadCalibrationState()`: uses `calibrationStateSchema.safeParse()` — validates nested structure safely

### Compliance & Reporting (5 schemas)

#### `src/lib/audit/compliance-report.ts`
- **`complianceStatusSchema`**: Enum (`compliant`, `partial`, `gap`)
- **`complianceCheckItemSchema`**: Per-article check (requirement, article, status, detail)
- **`coverageSummarySchema`**: Coverage counts (traceRecords, evaluationRecords, verificationRecords, systemRecords)
- **`complianceReportSchema`**: Full report (generatedAt, reportingPeriod, retention, auditRecordCount, chainIntegrity, coverage, checks[], overallStatus)
- **`complianceReportOptionsSchema`**: Report options (startDate, endDate, telemetryDir)
- **Integration**: Input validation in `generateComplianceReport()` — ensures EU AI Act compliance audit trail integrity

## Validation Patterns

### Pattern 1: File I/O Boundaries (safeParse)

Use `safeParse()` for all external data:

```typescript
const parsed = JSON.parse(content);
const result = fileIndexSchema.safeParse(parsed);
if (!result.success) {
  return null; // or throw error
}
return result.data; // Fully typed, guaranteed valid
```

### Pattern 2: Constructor/Entry Points (safeParse)

Use `safeParse()` at function entry points for external inputs:

```typescript
export function createBudgetTracker(config: BudgetConfig = {}) {
  const parseResult = budgetConfigSchema.safeParse(config);
  if (!parseResult.success) {
    throw new CostEstimationError(`Invalid budget config: ${parseResult.error.message}`);
  }
  const validatedConfig = parseResult.data;
  // Use validatedConfig exclusively downstream
}
```

### Pattern 3: Internal/Trusted Data (parse)

Use `parse()` only for data you fully control:

```typescript
// In tests: data is constructed locally, always valid
const record = auditRecordSchema.parse(testData);
```

### Pattern 4: Type Inference

Always infer types from schemas, never duplicate:

```typescript
// ✓ Good: Single source of truth
export type AuditRecordInput = z.infer<typeof auditRecordInputSchema>;

// ✗ Bad: Manual type definition alongside schema
export interface AuditRecordInput { ... }
export const auditRecordInputSchema = z.object({ ... });
```

## Benefits

| Aspect | Before | After |
|--------|--------|-------|
| Manual guards | 16+ lines per function | 2-3 lines with schema validation |
| Type safety | Runtime `as` casts, silent corruption | Full type validation, errors surfaced |
| Error messages | Generic "invalid input" | Zod's structured error details |
| Maintenance | Manual sync between interface + guards | Single schema as source of truth |
| File I/O | No validation on read | All reads validated via safeParse |

## Files Using Zod Validation (v3.0.14)

| Module | Files | Schemas | Coverage |
|--------|-------|---------|----------|
| Indexing | 1 | 3 | readIndex, queryIndex, readLinesByNumber |
| Audit | 2 | 9 | createAuditRecord, queryAuditRecords, getLastRecordHash, generateComplianceReport |
| Cost | 1 | 7 | createBudgetTracker, calculateCost, estimateBatchCost |
| Judge Config | 1 | 6 | resolveConfig, LLMJudge constructor |
| Evaluation | 2 | 7 | gEval, qagEvaluate, mitigatedPairwiseEval |
| Quality (feature engineering) | 1 | 4+ | loadDegradationState, loadCalibrationState, ParameterStability, sweepWithCrossValidation (v3.0.14) |
| Local JSONL backend | 1 | 3 | datasetRecordSchema, datasetTraceRecordSchema, datasetRunRecordSchema — dataset record parsing (v3.0.14) |
| Evaluation hooks | 1 | 10 | webhookConfigSchema, hookStatsSchema, hookResultSchema, scoreNormalizationConfigSchema, etc. (v3.0.14, extracted to evaluation-hooks-schemas.ts) |
| **Total** | **10** | **49+** | **System boundaries** |

Additional schemas in `src/lib/validation/` (see [VALIDATION_INTEGRATION.md](VALIDATION_INTEGRATION.md)):
- `dashboard-schemas.ts` — 10 schemas (trace spans, OTel logs, evaluation records, KV sync)
- `api-schemas.ts` — 21 schemas (OTLP ingestion, dataset records, session entries)

## Future Work

**Remaining gaps (LOWER priority):**
- `quality-views.ts` — RoleView discriminated unions
- `langfuse-export.ts` — LangfuseConfig (replace manual validation)
- Resilience modules — CacheStats, CircuitBreakerOptions
- Error handling — ErrorCode, ErrorCategory

See [BACKLOG.md](BACKLOG.md) for detailed item tracking.

## References

- **Zod Documentation**: https://zod.dev
- **Pattern**: Validate at system boundaries (files, API endpoints, config constructors)
- **Test coverage**: All 1553 tests pass with schema validation integrated
