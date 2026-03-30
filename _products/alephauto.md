---
title: "AlephAuto"
excerpt: "Job queue framework with real-time dashboard for automation pipelines."
layout: single
author_profile: true
---

Job queue framework with real-time dashboard for automation pipelines. Node.js + TypeScript, event-driven lifecycle with concurrency control, auto-retry, and Sentry integration.

## Pipelines

| Pipeline | Schedule | Output |
|----------|----------|--------|
| Duplicate Detection | 2 AM daily | HTML/MD/JSON reports + PRs |
| Schema Enhancement | 3 AM daily | Modified READMEs + JSON |
| Git Activity Reporter | Sunday 8 PM | Jekyll MD + SVG |
| Repository Cleanup | Sunday 3 AM | Cleanup logs |
| Repomix | 2 AM daily | Compressed XML repo snapshots |
| Claude Health | 8 AM daily | MD/JSON reports |
| Dashboard Populate | 6 AM/6 PM | Cloudflare KV + reports |
| Bugfix Audit | Recurring | Audit reports |
| Gitignore Update | Scheduled | Updated .gitignore files |
| Plugin Management | Monday 9 AM | Audit reports |
| Test Refactor | Manual | Refactored test files |

## Architecture

- **SidequestServer** — Event-driven job lifecycle (created -> queued -> running -> completed/failed) with concurrency control, auto-retry with error classification, and PostgreSQL persistence
- **REST API** — 23 routes with Zod-validated types, auth middleware, rate limiting, WebSocket support
- **React Dashboard** — Real-time pipeline monitoring (Vite + TypeScript)
- **Pipeline Core** — Scan orchestrator, structural similarity engine, error classifier, branch manager (branch/commit/PR automation)
- **Shared Packages** — `@shared/logging` (Pino), `@shared/process-io` (child process utilities)
- **Edge Worker** — Cloudflare Worker proxy (n0ai-proxy)

## Key Capabilities

- **Duplicate Detection** — Multi-stage pipeline: repo scanning, pattern detection, extraction, annotation, similarity analysis, grouping, and report generation
- **Schema Enhancement** — Automated Schema.org metadata injection into repository documentation
- **Quality Metrics Integration** — Populates the Quality Metrics Dashboard via derive/judge/sync pipeline
- **Git Activity Reporting** — Weekly cross-repo activity summaries published as Jekyll posts with SVG visualizations
- **Job Execution Control** — Disable/enable job creation for CI/CD deployments via environment flag

## Deployment

PM2 process management with Doppler secrets. Traditional server deployment with automated update scripts. Job execution can be paused during deployments.
