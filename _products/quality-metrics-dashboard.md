---
title: "Quality Metrics Dashboard"
excerpt: "React dashboard for AI quality metrics with role-based access, trace inspection, and compliance views."
layout: single
author_profile: true
---

React 19 + Vite 8 dashboard displaying quality metrics derived from Claude Code session telemetry. Auth0 Universal Login with role-based access control backed by Supabase. Deployed as Cloudflare Pages (frontend) + Worker (API).

## Authentication

Auth0 Universal Login with PKCE flow. JWT verified via Auth0 JWKS on the worker. User lookup and permissions loaded from Supabase (database-driven RBAC enriched into JWT via Auth0 Post-Login Action).

### Permissions

| Permission | Access |
|---|---|
| `dashboard.read` | Base read access |
| `dashboard.executive` | Executive view |
| `dashboard.operator` | Operator view |
| `dashboard.auditor` | Auditor view |
| `dashboard.traces.read` | Trace detail |
| `dashboard.sessions.read` | Session detail |
| `dashboard.agents.read` | Agent detail |
| `dashboard.pipeline.read` | Pipeline status |
| `dashboard.compliance.read` | Compliance pages |
| `dashboard.admin` | Admin (bypasses all checks) |

## Data Pipeline

Three-step pipeline populates the dashboard from local telemetry:

1. **Derive** — Rule-based metrics: tool correctness, evaluation latency, task completion
2. **Judge** — LLM-based metrics: relevance, coherence, faithfulness, hallucination
3. **Sync** — Delta sync aggregates to Cloudflare KV (budget-based, priority: meta/agent > metrics > trends > traces)

Auto-detects missing API keys and falls back to synthetic scoring. Also available as an AlephAuto cron job running twice daily.

## API Routes

All routes require Auth0 JWT except health check.

| Route | Description |
|-------|-------------|
| `GET /api/me` | Current user session, roles, permissions |
| `GET /api/dashboard` | Dashboard summary by period and role |
| `GET /api/metrics/:name` | Metric detail and evaluations |
| `GET /api/trends/:name` | Metric trend data |
| `GET /api/traces/:traceId` | Trace spans + evaluations |
| `GET /api/sessions/:sessionId` | Session detail |
| `GET /api/agents` | Cross-session agent list |
| `GET /api/agents/detail/:agentId` | Agent stats (RED metrics, output quality) |
| `GET /api/correlations` | Metric correlation matrix |
| `GET /api/degradation-signals` | Quality degradation signals |
| `GET /api/coverage` | Evaluation coverage heatmap |
| `GET /api/pipeline` | Populate pipeline status |
| `GET /api/compliance/sla` | SLA compliance |
| `GET /api/compliance/verifications` | Human verifications |
| `GET /api/calibration` | Score calibration metadata |
| `GET /api/routing-telemetry` | Agent routing telemetry |
| `GET /api/admin/users` | List users with roles (admin) |
| `POST /api/admin/users/:userId/roles` | Assign role (admin) |

## Frontend Structure

- **Pages**: Overview, metric detail, role views (executive/operator/auditor), trace detail, session detail, agent activity, correlations, coverage, compliance, admin
- **Components**: Trend charts, evaluation tables, workflow graphs/timelines, agent activity panels
- **Contexts**: Auth, role-based access, keyboard navigation, score calibration
- **Hooks**: API queries, metric evaluations, session detail, agent stats, trace inspection
- **Validation**: Zod schemas for auth and dashboard request/response types
