---
title: "Observability Toolkit"
excerpt: "MCP server for querying traces, metrics, logs, and LLM events from agentic coding tools."
layout: single
author_profile: true
---

MCP server for observability tooling — query traces, metrics, logs, and LLM events from agentic coding tools. Works with any agent emitting OTel GenAI semantic conventions. Ingest via OTLP to Cloudflare R2 or read from local JSONL.

## Tools

| Tool | Description |
|------|-------------|
| `obs_query_traces` | Query spans with filtering, regex, numeric operators |
| `obs_query_metrics` | Aggregations (sum, avg, p50, p95, p99, rate), time buckets |
| `obs_query_logs` | Boolean search, field extraction, negation |
| `obs_query_llm_events` | Token usage, duration, provider/model filters |
| `obs_query_evaluations` | Evaluation events with aggregations and groupBy |
| `obs_query_verifications` | Human verification events for EU AI Act compliance |
| `obs_query_regressions` | EWMA drift and consecutive breach tracking |
| `obs_query_metric_histograms` | OTLP histogram bucket distributions |
| `obs_health_check` | System health with cache statistics |
| `obs_token_budget` | Context utilization, cache hit rate, headroom per model/session |
| `obs_hallucination_detection` | Risk rates, scores, model/method breakdowns |
| `obs_multi_agent_coordination` | Delegation depth, fan-out ratio, handoff latency |
| `obs_routing_telemetry` | Model distribution, cost savings, fallback rate |
| `obs_estimate_cost` | Token cost estimation across models |
| `obs_audit_trail` | Audit trail events (SHA-256 hash chain) |
| `obs_manage_datasets` | Create, list, get, delete evaluation datasets |
| `obs_export_langfuse` | Export evaluations to Langfuse via OTLP HTTP |
| `obs_export_phoenix` | Export evaluations to Arize Phoenix via OTLP HTTP |
| `obs_export_datadog` | Export evaluations to Datadog LLM Observability |
| `obs_export_confident` | Export evaluations to Confident AI |
| `obs_ingest_spans` | Ingest spans to cloud backend via OTLP protobuf |
| `obs_ingest_traces` | Push complete OTel traces with service metadata |

## Data Sources

**Local JSONL** — Scans `~/.claude/telemetry/` and project-local directories. Supports gzip. Compatible with Claude Code and any OTel file exporter.

**Cloud Backend** — All query tools accept `backend: 'local' | 'cloud' | 'auto'`. Cloud queries D1/R2 via authenticated API. Circuit breaker protects against cascading failures.

### Data Pipelines

1. **Local**: hooks -> telemetry JSONL -> derive -> judge -> sync-to-kv -> Cloudflare KV
2. **Cloud**: hooks -> OTLP HTTP -> ingest.integritystudio.ai -> R2 -> D1 -> api.integritystudio.ai

## Services

**Ingest Worker** — Cloudflare Worker (Hono) at `ingest.integritystudio.ai`. Receives OTLP protobuf telemetry, stores to R2 with SHA-256 bearer auth and KV idempotency.

**API Worker** — Cloudflare Worker (Hono) at `api.integritystudio.ai`. Queries D1/R2 with cursor-based pagination and per-key rate limiting.

**API Provisioning** — Two-worker system for API key lifecycle. HMAC-signed requests, Auth0 JWT validation, Supabase org management, tiered access (starter/growth/enterprise).

## Evaluation Libraries

**LLM-as-Judge** — G-Eval (chain-of-thought + logprob normalization), QAG faithfulness, position bias mitigation, panel evaluation, circuit breaker + retry.

**Agent-as-Judge** — ProceduralJudge (fixed pipeline, early termination), ReactiveJudge (adaptive routing), tool verification, trajectory efficiency analysis, multi-agent handoff scoring.

**Quality Pipeline** — Rule-based metrics (every invocation, zero cost), LLM judge metrics (sampled, budget-controlled), entropy-based divergence detection, EWMA regression detection.

## Dashboard

React 19 + Vite dashboard with Hono API, Auth0 Universal Login, role-based access via Supabase. Deployed as Cloudflare Pages + Worker.

## Integrations

| Platform | Method | Status |
|----------|--------|--------|
| Claude Code | Native MCP | Full |
| Cursor / Windsurf / Continue.dev / Cline | MCP config | Full |
| Any OTel agent | OTLP -> local JSONL | Full |
| Langfuse / Phoenix / Datadog / Confident AI | OTLP / HTTP export | Export only |

OTel GenAI semconv v1.40.0 compliance. 15 LLM providers supported.
