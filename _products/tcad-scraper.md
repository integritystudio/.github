---
title: "TCAD Scraper"
excerpt: "Automated property tax data collection and AI-powered search for Travis County, TX."
layout: single
author_profile: true
---

Production web scraping system for automated collection of property tax information from the Travis Central Appraisal District (TCAD). Built with TypeScript, Cloudflare Workers (Hono), Prisma, and PostgreSQL.

## Data Collection

- **API-Direct Scraping**: High-volume scraping via TCAD API with automatic token refresh
- **Continuous Batch Scraping**: 24/7 automated scraping with weighted search term generation
- **Workflow-Based Processing**: Cloudflare Workflows with 5-step pipeline (token, fetch, dedup, upsert, analytics)
- **Smart Search Strategies**: 200+ first names, 500+ last names, 150+ Austin streets with dynamic coverage adjustment

### Data Extracted

Owner name, property type, city, full address, assessed and appraised values, property ID, geographic ID, legal descriptions, and discovery metadata.

### Coverage (March 2026)

- 365,371 unique property records
- 313 unique search terms with zero overlap among top 30
- Tiered strategy: 15 terms = 19.6% coverage, 50 terms = 45.1%, 200 terms = 92.1%
- Scraping rate: ~42,000 properties/hour

## Architecture

```
React Frontend  -->  CF Workers (Hono)  -->  Hyperdrive  -->  PostgreSQL (Render)
(GitHub Pages)              |
                        CF Queue
                            |
                      ScraperWorkflow (5 steps)
                            |
                     TCAD API + KV Cache
```

### Scraping Pipeline

1. **Enqueue** via API or CLI scripts to Cloudflare Queue
2. **ScraperWorkflow** — Token acquisition, paginated API fetch (1000/page), deduplication, bulk upsert (chunks of 500), analytics update
3. **Batch Generation** — 5-tier priority term generation, 18 batch type configurations, multi-phase tail term optimizer
4. **Cron Triggers** — Token refresh (4 min), stale job cleanup (hourly), monitored search execution (6 hours)

## API and Frontend

- **REST API**: Cloudflare Workers with CORS, security headers, API key and JWT auth
- **AI-Powered Search**: Natural language property queries via Claude AI with OpenAI fallback
- **React Frontend**: Expandable property cards, financial analysis (appraised vs assessed), data freshness indicators, mobile responsive, WCAG compliant

### Key Endpoints

| Endpoint | Description |
|----------|-------------|
| `GET /api/properties` | Paginated property listing with city, type, and value filters |
| `POST /api/properties/search` | AI-powered natural language search |
| `POST /api/properties/scrape` | Enqueue scrape job |
| `POST /api/properties/scrape/batch` | Batch enqueue multiple search terms |
| `GET /health` | Health check with property count |

## Infrastructure

- **Cloudflare Workers** — Production API at `api.alephatx.info`
- **Cloudflare Hyperdrive** — Connection pooling to PostgreSQL
- **Cloudflare KV** — Token cache + response cache
- **Cloudflare Queues + Workflows** — Distributed scrape job processing
- **Render PostgreSQL** — Database hosting
- **GitHub Pages** — Frontend at `alephatx.info`
- **Sentry** — Error tracking
- **GA4 + Meta Pixel** — User behavior analytics

## Database Schema

Three core models:

- **Property** — Scraped records with composite unique constraint on (propertyId, year), indexed on city, type, value, and search term
- **ScrapeJob** — Job tracking with status lifecycle (pending, processing, completed, failed) and result analytics
- **MonitoredSearch** — Recurring automated scrapes with configurable frequency (daily, weekly, monthly)
