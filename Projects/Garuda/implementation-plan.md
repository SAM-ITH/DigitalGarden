# Garuda - Implementation Plan

## Overview

Garuda is a data pipeline that extracts structured financial data from Colombo Stock Exchange (CSE) quarterly report PDFs and serves it through a monetized REST API.

The project is divided into two systems:

| System | Name | Description |
|--------|------|-------------|
| **System 1** | [Mayura CLI](system-1-mayura-cli.md) | Bloomberg-style terminal for PDF intake, parsing, AI structuring, and data submission |
| **System 2** | [Garuda REST API](system-2-garuda-api.md) | FastAPI service with internal write endpoints and public read-only endpoints (monetized) |
| **Database** | [Database Architecture](database-architecture.md) | PostgreSQL schema designed from 9 real CSE quarterly reports across 6 sectors |
| **DeepSeek Prompt** | [DeepSeek Prompt Guide](deepseek-prompt-guide.md) | System prompt, user prompt template, API call, validation and retry strategy |
| **Pipeline Flow** | [Data Pipeline Flow](data-pipeline-flow.md) | Step-by-step: PDF → LiteParse → DeepSeek → API → PostgreSQL |

## Architecture

```
┌─────────────────────────────────────────────┐
│              System 1: Mayura CLI            │
│                                             │
│  User Input → PDF → LiteParse → DeepSeek     │
│                          ↓                  │
│                   Structured JSON            │
│                          ↓                  │
│                   POST to API ──────────────┼──┐
└─────────────────────────────────────────────┘  │
                                                  │
┌─────────────────────────────────────────────┐  │
│           System 2: Garuda REST API          │  │
│                                             │  │
│  Internal Endpoints (write) ←───────────────┼──┘
│         ↓                                   │
│    PostgreSQL (Coolify)                     │
│         ↓                                   │
│  Public Endpoints (read-only, monetized)    │
│         ↓                                   │
│    Paying Customers                         │
└─────────────────────────────────────────────┘
```

## Tech Stack

| Component | Technology |
|-----------|-----------|
| CLI Terminal | Python, Rich |
| PDF Parsing | LiteParse (local, open-source) |
| AI Structuring | DeepSeek API (via OpenAI SDK) |
| REST API | Python, FastAPI |
| Database | PostgreSQL (hosted on Coolify) |
| ORM | SQLAlchemy (async) + asyncpg |
| Migrations | Alembic |
| Deployment | Docker on Coolify |
| Auth | Self-managed API keys (SHA-256 hashed) |

## Development Order

### Recommended path:

1. **System 2, Phase 1-3** — Get the API running with database and write endpoints first
2. **System 1, Phase 0-3** — Build the CLI with parsing and structuring
3. **System 1, Phase 4** — Connect CLI to the running API
4. **System 2, Phase 4-7** — Build public endpoints, auth, rate limiting, Docker
5. **System 1, Phase 5** — Polish the CLI

This order lets you test the full pipeline early (by Phase 3 of both systems) while deferring monetization features until the core works.

## Phase Summary

### System 1: Mayura CLI (6 phases)

| Phase | Name | Goal |
|-------|------|------|
| 0 | [Project Scaffolding](system-1-mayura-cli.md#phase-0-project-scaffolding) | Runnable Python package with config |
| 1 | [Terminal UI](system-1-mayura-cli.md#phase-1-terminal-ui) | Styled interactive input collection |
| 2 | [PDF Parsing](system-1-mayura-cli.md#phase-2-pdf-parsing-liteparse) | LiteParse local parsing |
| 3 | [AI Structuring](system-1-mayura-cli.md#phase-3-ai-structuring-deepseek) | DeepSeek data extraction |
| 4 | [API Integration](system-1-mayura-cli.md#phase-4-api-integration) | Submit data to REST API |
| 5 | [Polish](system-1-mayura-cli.md#phase-5-polish-and-error-handling) | Logging, error handling, UX |

### System 2: Garuda REST API (7 phases)

| Phase | Name | Goal |
|-------|------|------|
| 1 | [Project Scaffolding](system-2-garuda-api.md#phase-1-project-scaffolding) | Runnable FastAPI app with health check |
| 2 | [Database Schema](system-2-garuda-api.md#phase-2-database-schema-and-migrations) | PostgreSQL tables via Alembic |
| 3 | [Internal Write Endpoints](system-2-garuda-api.md#phase-3-internal-write-endpoints) | CLI can POST financial data |
| 4 | [Public Read Endpoints](system-2-garuda-api.md#phase-4-public-read-endpoints) | Customer-facing query API |
| 5 | [Authentication](system-2-garuda-api.md#phase-5-authentication-and-api-key-system) | API key system |
| 6 | [Rate Limiting](system-2-garuda-api.md#phase-6-rate-limiting-and-usage-tracking) | Per-key rate limits and usage tracking |
| 7 | [Docker & Deployment](system-2-garuda-api.md#phase-7-docker-and-deployment) | Production-ready Docker image for Coolify |

## Verification

### End-to-end integration test:
1. Start System 2 locally (`docker compose up`)
2. Run Mayura CLI with a real CSE quarterly report PDF
3. Verify: PDF → LiteParse → DeepSeek → structured JSON → API → database
4. Query the public endpoint with a customer API key → get the same data back
