# Evaluation Export Integrations

**Status**: All integrations complete | **Last Updated**: 2026-03-10

## Overview

| Platform | Tool | Protocol | Auth Header |
|----------|------|----------|-------------|
| Langfuse | `obs_export_langfuse` | OTLP HTTP | `Authorization: Basic base64(pub:secret)` |
| Confident AI | `obs_export_confident` | OTLP HTTP | `x-confident-api-key` |
| Arize Phoenix | `obs_export_phoenix` | OTLP HTTP | `Authorization: Bearer` (or legacy `api_key`) |
| Datadog | `obs_export_datadog` | Proprietary | `DD-API-KEY` |
| obtool.cloud | Claude Code hooks | OTLP HTTP protobuf | `Authorization: Bearer` |

All platforms: batch size 100, max export 10,000, timeout 30s.

## File Structure

```
src/
├── lib/
│   ├── core/
│   │   ├── constants.ts          # *_ENDPOINT, *_API_KEY, etc.
│   │   ├── shared-schemas.ts     # exportFilterSchema, exportOptionsSchema
│   │   └── input-validator.ts    # validateScoreRange, wrapValidationErrors
│   └── exports/
│       ├── export-utils.ts       # fetchWithRetry, sanitizeErrorText, checkMemoryUsage, validateExternalUrl
│       ├── langfuse-export.ts
│       ├── confident-export.ts
│       ├── phoenix-export.ts
│       ├── datadog-export.ts
│       ├── otlp-export.ts        # Shared OTLP logic
│       └── otlp-proto-encode.ts  # Protobuf wire format (Phoenix)
├── tools/
│   ├── export-base.ts            # createExportHandler factory (shared by all 4)
│   ├── export-langfuse.ts
│   ├── export-confident.ts
│   ├── export-phoenix.ts
│   └── export-datadog.ts
services/
└── obtool-ingest/                # Cloudflare Worker at ingest.integritystudio.ai
    └── src/                      # index.ts, auth.ts, decode.ts, r2-writer.ts
```

Data flow: Local JSONL evaluations → `*-export.ts` (convert + batch) → Platform API `/v1/traces`

## Common Tool Parameters

All export tools accept these shared parameters:

```typescript
{
  // Filters
  evaluationName?, scoreMin?, scoreMax?, scoreLabel?,
  evaluatorType?, traceId?, sessionId?, startDate?, endDate?,
  // Export options
  limit?, batchSize?, dryRun?,
}
```

## Platform Configuration

### Langfuse

Env: `LANGFUSE_ENDPOINT` (required), `LANGFUSE_PUBLIC_KEY`, `LANGFUSE_SECRET_KEY`
Additional params: `endpoint?`, `publicKey?`, `secretKey?`

### Confident AI

Env: `CONFIDENT_API_KEY` (required), `CONFIDENT_ENDPOINT` (default: `https://otel.confident-ai.com`), `CONFIDENT_ENVIRONMENT` (default: `development`)
Additional params: `metricCollection?`, `environment?`, `endpoint?`, `apiKey?`

| Evaluation Field | Confident Attribute |
|------------------|---------------------|
| `sessionId` | `confident.trace.thread_id` |
| `evaluationName` | `confident.span.name` |

### Arize Phoenix

Env: `PHOENIX_COLLECTOR_ENDPOINT` (default: `http://localhost:6006`), `PHOENIX_API_KEY` (cloud only), `PHOENIX_PROJECT_NAME` (default: `default`)
Endpoints: local `:6006`, self-hosted `http://<host>:6006`, cloud `https://app.phoenix.arize.com/s/<space>`
Additional params: `projectName?`, `legacyAuth?`, `endpoint?`, `apiKey?`

| Evaluation Field | Phoenix Attribute |
|------------------|-------------------|
| `sessionId` | `session.id` |
| `evaluationName` | `span.name` |

### Datadog

Env: `DD_API_KEY` (required), `DD_SITE` (default: `datadoghq.com`), `DD_LLMOBS_ML_APP` (required)
Sites: US1 `datadoghq.com`, US3/US5/AP1 `{region}.datadoghq.com`, EU `datadoghq.eu`
Additional params: `mlApp?`, `site?`, `exportSpans?`, `exportEvals?`, `metricType?`, `apiKey?`

Two-phase export: spans via `POST .../v1/trace/spans`, then evals via `POST .../v2/eval-metric`.

Metric type inference: `0|1` → boolean, `(0,1)` → score, `scoreLabel` present → categorical.

| Evaluation Field | Datadog Field |
|------------------|---------------|
| `evaluationName` | `label` |
| `scoreValue` | `score_value` |
| `scoreLabel` | `categorical_value` |
| `explanation` | `reasoning` |
| `traceId` | `join_on.span.trace_id` |

## GenAI Semantic Conventions (OTLP)

All OTLP-based integrations map: `evaluationName` → `gen_ai.evaluation.name`, `scoreValue` → `gen_ai.evaluation.score.value`, `scoreLabel` → `gen_ai.evaluation.score.label`, `explanation` → `gen_ai.evaluation.explanation`, `evaluatorType` → `gen_ai.evaluation.evaluator.type`.

## Security

- **SSRF**: Langfuse/Confident use `validateExternalUrl()`; Phoenix allows localhost, requires HTTPS for cloud; Datadog uses pre-defined endpoints
- **Blocked**: `127.x.x.x`, `::1`, `0.0.0.0`, `169.254.x.x`, private networks (`10.x`, `172.16-31.x`, `192.168.x`), reserved TLDs (`.local`, `.internal`)
- **DNS rebinding**: endpoints re-validated before each batch
- **Credentials**: all errors pass through `sanitizeErrorText()`
- **Memory**: warn at 400MB, abort at 600MB

## obtool.cloud Ingest

Cloudflare Worker at `ingest.integritystudio.ai` — replaced SigNoz Cloud in v2.10 (`dcd4996`).
Bearer token auth (KV-backed SHA-256), R2 JSONL storage, gzip decompression, request dedup via `X-Idempotency-Key` (5-min TTL), `/health` probes.

```
OTEL_EXPORTER_OTLP_ENDPOINT=https://ingest.integritystudio.ai
OTEL_EXPORTER_OTLP_PROTOCOL=http/protobuf
```

## References

- [Langfuse](https://langfuse.com/docs) | [Confident AI](https://www.confident-ai.com/docs) | [Phoenix](https://docs.arize.com/phoenix) | [Datadog LLM Obs](https://docs.datadoghq.com/llm_observability/)
- [OTel GenAI SemConv](https://opentelemetry.io/docs/specs/semconv/gen-ai/) | [OTLP HTTP Spec](https://opentelemetry.io/docs/specs/otlp/)
- [obtool-cloud exporter docs](../changelog/2.2/impl-obtool-cloud-exporter.md)
