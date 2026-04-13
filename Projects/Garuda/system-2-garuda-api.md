# System 2: Garuda REST API

Python FastAPI service for storing and serving CSE (Colombo Stock Exchange) quarterly financial data. Deployed as a Docker container on Coolify with PostgreSQL.

**Tech stack:** Python, FastAPI, SQLAlchemy (async) + asyncpg, Alembic, PostgreSQL, Pydantic, Docker

## Architecture

```
┌─────────────────────────────────────────────────────┐
│                   Garuda REST API                    │
│                                                     │
│  ┌──────────────┐  ┌─────────────┐  ┌───────────┐  │
│  │   Internal    │  │   Public    │  │   Admin   │  │
│  │   Endpoints   │  │  Endpoints  │  │ Endpoints │  │
│  │  (write)      │  │ (read-only) │  │ (keys)    │  │
│  └──────┬───────┘  └──────┬──────┘  └─────┬─────┘  │
│         │                 │               │         │
│         ▼                 ▼               ▼         │
│  ┌─────────────────────────────────────────────┐    │
│  │           Auth + Rate Limiting               │    │
│  └──────────────────┬──────────────────────────┘    │
│                     ▼                               │
│  ┌─────────────────────────────────────────────┐    │
│  │              PostgreSQL (Coolify)            │    │
│  └─────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────┘
```

## Final Directory Structure

```
garuda-api/
├── Dockerfile
├── docker-compose.yml          # Local dev (app + PostgreSQL)
├── docker-compose.prod.yml     # Production (app only, external DB)
├── pyproject.toml
├── .env.example
├── .gitignore
├── .dockerignore
├── alembic.ini
├── alembic/
│   ├── env.py
│   ├── script.py.mako
│   └── versions/
├── scripts/
│   ├── entrypoint.sh
│   └── create_initial_api_key.py
├── app/
│   ├── __init__.py
│   ├── main.py                 # FastAPI app factory
│   ├── core/
│   │   ├── __init__.py
│   │   ├── config.py           # pydantic-settings
│   │   ├── security.py         # API key hashing/generation
│   │   ├── exceptions.py       # Custom exceptions + handlers
│   │   └── logging.py          # Structured JSON logging
│   ├── api/v1/
│   │   ├── __init__.py
│   │   ├── router.py           # Aggregates all v1 routers
│   │   ├── dependencies.py     # Auth dependencies
│   │   ├── internal.py         # Write endpoints (Mayura CLI)
│   │   ├── public.py           # Read endpoints (customers)
│   │   └── admin.py            # API key management
│   ├── models/
│   │   ├── __init__.py
│   │   ├── company.py
│   │   ├── financial_report.py
│   │   ├── api_key.py
│   │   └── usage_log.py
│   ├── schemas/
│   │   ├── __init__.py
│   │   ├── company.py
│   │   ├── financial_report.py
│   │   ├── api_key.py
│   │   ├── usage.py
│   │   └── pagination.py
│   ├── crud/
│   │   ├── __init__.py
│   │   ├── company.py
│   │   ├── financial_report.py
│   │   ├── api_key.py
│   │   └── usage_log.py
│   ├── db/
│   │   ├── __init__.py
│   │   ├── base.py             # Declarative base
│   │   └── session.py          # Async engine + session
│   └── middleware/
│       ├── __init__.py
│       ├── rate_limiter.py
│       ├── usage_logger.py
│       └── request_id.py
└── tests/
    ├── __init__.py
    ├── conftest.py
    ├── test_health.py
    ├── test_internal_endpoints.py
    ├── test_public_endpoints.py
    ├── test_auth.py
    └── test_rate_limiting.py
```

## Phase Dependencies

```
Phase 1 → Phase 2 → Phase 3 → Phase 4 → Phase 5 → Phase 6 → Phase 7
(strictly sequential — each phase builds on the previous)
```

---

## Phase 1: Project Scaffolding

**Goal:** A runnable FastAPI application with project structure, config management, and health check endpoint.

### Tasks

1. Create `pyproject.toml` (Python 3.11+) with dependencies:
   - `fastapi`, `uvicorn[standard]`, `sqlalchemy[asyncio]`, `asyncpg`, `alembic`
   - `pydantic-settings`, `python-dotenv`
   - `httpx`, `pytest`, `pytest-asyncio` (dev dependencies)

2. Create `app/core/config.py` using `pydantic-settings`:
   ```python
   class Settings(BaseSettings):
       model_config = SettingsConfigDict(env_file=".env")

       database_url: str
       internal_api_key: str
       cors_origins: list[str] = ["http://localhost:3000"]
       env: str = "development"
   ```

3. Create `app/main.py` — FastAPI app factory with:
   - CORS middleware (configured origins)
   - API versioning: mount v1 router at `/api/v1`
   - `GET /health` endpoint returning `{"status": "ok"}`

4. Create `.env.example`, `.gitignore`

### Files to Create

- `garuda-api/pyproject.toml`
- `garuda-api/.env.example`
- `garuda-api/.gitignore`
- `garuda-api/app/__init__.py`
- `garuda-api/app/main.py`
- `garuda-api/app/core/__init__.py`
- `garuda-api/app/core/config.py`
- `garuda-api/app/api/__init__.py`
- `garuda-api/app/api/v1/__init__.py`
- `garuda-api/app/api/v1/router.py`

### Done When

`uvicorn app.main:app --reload` starts without errors. `GET /health` returns `{"status": "ok"}`. Config loads from `.env` with validation.

---

## Phase 2: Database Schema and Migrations

**Goal:** PostgreSQL schema with all tables defined in SQLAlchemy ORM, managed by Alembic async migrations.

### Database Schema

#### `companies` table

| Column | Type | Notes |
|--------|------|-------|
| id | UUID | PK, `gen_random_uuid()` |
| ticker | VARCHAR(20) | UNIQUE, NOT NULL, indexed |
| name | VARCHAR(255) | NOT NULL |
| sector | VARCHAR(100) | nullable |
| market | VARCHAR(50) | default "CSE" |
| is_active | BOOLEAN | default true |
| created_at | TIMESTAMP WITH TZ | default now() |
| updated_at | TIMESTAMP WITH TZ | auto-update |

#### `quarterly_financial_reports` table

| Column | Type | Notes |
|--------|------|-------|
| id | UUID | PK |
| company_id | UUID | FK → companies.id, NOT NULL |
| year | INTEGER | NOT NULL |
| quarter | SMALLINT | 1-4, NOT NULL |
| revenue | NUMERIC(20,2) | nullable |
| cost_of_sales | NUMERIC(20,2) | nullable |
| gross_profit | NUMERIC(20,2) | nullable |
| operating_profit | NUMERIC(20,2) | nullable |
| net_profit | NUMERIC(20,2) | nullable |
| eps | NUMERIC(12,4) | nullable |
| total_assets | NUMERIC(20,2) | nullable |
| total_liabilities | NUMERIC(20,2) | nullable |
| total_equity | NUMERIC(20,2) | nullable |
| operating_cash_flow | NUMERIC(20,2) | nullable |
| investing_cash_flow | NUMERIC(20,2) | nullable |
| financing_cash_flow | NUMERIC(20,2) | nullable |
| net_cash_flow | NUMERIC(20,2) | nullable |
| dividends_per_share | NUMERIC(12,4) | nullable |
| shares_outstanding | BIGINT | nullable |
| report_date | DATE | nullable (filing date) |
| currency | VARCHAR(3) | default "LKR" |
| source | VARCHAR(255) | nullable |
| raw_data | JSONB | nullable (original payload) |
| created_at | TIMESTAMP WITH TZ | default now() |
| updated_at | TIMESTAMP WITH TZ | auto-update |

**Unique constraint:** `(company_id, year, quarter)` — one report per company per quarter.

#### `api_keys` table

| Column | Type | Notes |
|--------|------|-------|
| id | UUID | PK |
| key_hash | VARCHAR(64) | UNIQUE, NOT NULL, indexed (SHA-256) |
| key_prefix | VARCHAR(8) | NOT NULL (first 8 chars, for display) |
| customer_name | VARCHAR(255) | NOT NULL |
| customer_email | VARCHAR(255) | nullable |
| plan | VARCHAR(50) | NOT NULL, default "free" |
| rate_limit_per_minute | INTEGER | default 60 |
| is_active | BOOLEAN | default true |
| is_internal | BOOLEAN | default false |
| expires_at | TIMESTAMP WITH TZ | nullable |
| created_at | TIMESTAMP WITH TZ | default now() |

#### `usage_logs` table

| Column | Type | Notes |
|--------|------|-------|
| id | UUID | PK |
| api_key_id | UUID | FK → api_keys.id, NOT NULL |
| endpoint | VARCHAR(255) | NOT NULL |
| method | VARCHAR(10) | NOT NULL |
| status_code | SMALLINT | NOT NULL |
| response_time_ms | INTEGER | nullable |
| ip_address | VARCHAR(45) | nullable |
| created_at | TIMESTAMP WITH TZ | default now(), indexed |

**Index:** `(api_key_id, created_at)` for rate limit lookups and usage analytics.

### Tasks

1. Create `app/db/session.py` — async engine + `async_sessionmaker` + `get_db` dependency
2. Create `app/db/base.py` — declarative base class
3. Create SQLAlchemy ORM models in `app/models/`
4. Initialize Alembic for async operation (`alembic init alembic`)
5. Configure `alembic/env.py` for async + import all models
6. Generate initial migration
7. Create `scripts/create_initial_api_key.py` for bootstrapping admin key

### Files to Create

- `garuda-api/app/db/__init__.py`
- `garuda-api/app/db/base.py`
- `garuda-api/app/db/session.py`
- `garuda-api/app/models/__init__.py`
- `garuda-api/app/models/company.py`
- `garuda-api/app/models/financial_report.py`
- `garuda-api/app/models/api_key.py`
- `garuda-api/app/models/usage_log.py`
- `garuda-api/alembic.ini`
- `garuda-api/alembic/env.py`
- `garuda-api/alembic/script.py.mako`
- `garuda-api/scripts/create_initial_api_key.py`

### Done When

`alembic upgrade head` creates all four tables with correct columns, constraints, and indexes. `alembic downgrade base` drops everything cleanly. Bootstrap script creates an admin API key.

---

## Phase 3: Internal Write Endpoints

**Goal:** Mayura CLI can POST financial data to the API with full validation and upsert behavior.

### Endpoints

| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/v1/internal/companies` | Create a company |
| POST | `/api/v1/internal/reports` | Upsert single quarterly report |
| POST | `/api/v1/internal/reports/bulk` | Upsert multiple reports |

All protected by `X-Internal-Key` header.

### Tasks

1. Create Pydantic schemas in `app/schemas/`:
   - `CompanyCreate`, `CompanyResponse`
   - `FinancialReportCreate`, `FinancialReportResponse`

2. Create CRUD modules in `app/crud/`:
   - `get_or_create_company(ticker, name, sector)`
   - `upsert_financial_report(...)` — if same (ticker, year, quarter) exists, update all fields

3. Create `app/api/v1/internal.py` with endpoints

4. Create `app/api/v1/dependencies.py` with internal key check:
   ```python
   async def verify_internal_key(x_internal_key: str = Header()):
       if x_internal_key != settings.internal_api_key:
           raise HTTPException(status_code=401)
   ```

### Pydantic Schema

```python
class FinancialReportCreate(BaseModel):
    ticker: str
    year: int
    quarter: int                        # 1-4
    revenue: Decimal | None = None
    cost_of_sales: Decimal | None = None
    gross_profit: Decimal | None = None
    operating_profit: Decimal | None = None
    net_profit: Decimal | None = None
    eps: Decimal | None = None
    total_assets: Decimal | None = None
    total_liabilities: Decimal | None = None
    total_equity: Decimal | None = None
    operating_cash_flow: Decimal | None = None
    investing_cash_flow: Decimal | None = None
    financing_cash_flow: Decimal | None = None
    net_cash_flow: Decimal | None = None
    dividends_per_share: Decimal | None = None
    shares_outstanding: int | None = None
    report_date: date | None = None
    currency: str = "LKR"
    source: str | None = None
    raw_data: dict | None = None
```

### Files to Create

- `garuda-api/app/schemas/__init__.py`
- `garuda-api/app/schemas/company.py`
- `garuda-api/app/schemas/financial_report.py`
- `garuda-api/app/crud/__init__.py`
- `garuda-api/app/crud/company.py`
- `garuda-api/app/crud/financial_report.py`
- `garuda-api/app/api/v1/internal.py`
- `garuda-api/app/api/v1/dependencies.py`

### Done When

POST with valid key inserts data (creating company if needed). Second POST with same ticker/year/quarter updates the row. Wrong key returns 401. Bulk endpoint works. Validation rejects invalid input (quarter=5, missing ticker).

---

## Phase 4: Public Read Endpoints

**Goal:** Customers can query financial data through well-designed REST endpoints.

### Endpoints

| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/v1/companies` | List companies (filterable by sector, paginated) |
| GET | `/api/v1/companies/{ticker}` | Company detail |
| GET | `/api/v1/reports/{ticker}` | All reports for a company |
| GET | `/api/v1/reports/{ticker}/{year}` | All quarters for a company in a year |
| GET | `/api/v1/reports/{ticker}/{year}/{quarter}` | Specific quarterly report |
| GET | `/api/v1/reports` | Search/filter across companies |

### Tasks

1. Create pagination schema:
   ```python
   class PaginatedResponse(BaseModel, Generic[T]):
       data: list[T]
       total: int
       limit: int
       offset: int
   ```

2. Add read CRUD functions in `app/crud/`

3. Create `app/api/v1/public.py` with all endpoints

4. Offset pagination: `?limit=20&offset=0`

5. For now, endpoints are open (auth added in Phase 5)

### Files to Create

- `garuda-api/app/api/v1/public.py`
- `garuda-api/app/schemas/pagination.py`
- Modify: `app/crud/financial_report.py`, `app/crud/company.py`

### Done When

All GET endpoints return correct JSON with seeded data. Pagination works. `GET /api/v1/reports/JKH/2025/1` returns the report or 404. Sector filtering works. Non-existent tickers return 404 with clear error.

---

## Phase 5: Authentication and API Key System

**Goal:** All endpoints secured with API keys. Keys hashed in database. Admin can manage keys.

### How It Works

1. Customer receives a raw API key (shown once at creation): `garuda_a1b2c3d4e5f6...`
2. Key is SHA-256 hashed and stored in `api_keys` table
3. On each request, the provided key is hashed and looked up
4. Active + non-expired = authorized

### Tasks

1. `app/core/security.py`:
   ```python
   def generate_api_key() -> tuple[str, str, str]:
       """Returns (raw_key, key_hash, key_prefix)"""
       ...
   def hash_api_key(raw_key: str) -> str:
       """SHA-256 hash of the key"""
       ...
   ```

2. Auth dependency in `app/api/v1/dependencies.py`:
   - Reads `X-API-Key` header
   - Hashes and looks up in database
   - Checks `is_active` and `expires_at`
   - For internal endpoints: checks `is_internal == True`
   - Returns `ApiKey` object for usage logging

3. Admin endpoints in `app/api/v1/admin.py` (protected by internal key):

   | Method | Path | Description |
   |--------|------|-------------|
   | POST | `/api/v1/admin/api-keys` | Create key (returns raw key once) |
   | GET | `/api/v1/admin/api-keys` | List keys (prefix only) |
   | PATCH | `/api/v1/admin/api-keys/{id}` | Update plan/rate limit/deactivate |
   | DELETE | `/api/v1/admin/api-keys/{id}` | Soft delete (set `is_active=false`) |

4. Apply auth dependencies to all public and internal routes

### Files to Create

- `garuda-api/app/core/security.py`
- `garuda-api/app/api/v1/admin.py`
- `garuda-api/app/schemas/api_key.py`
- `garuda-api/app/crud/api_key.py`
- Modify: `app/api/v1/dependencies.py`, `scripts/create_initial_api_key.py`

### Done When

Public endpoints return 401 without key. Valid active key returns 200. Expired/deactivated key returns 403. Internal endpoints reject non-internal keys. Admin can create, list, update, and deactivate keys. Raw key only visible at creation.

---

## Phase 6: Rate Limiting and Usage Tracking

**Goal:** Per-key rate limiting based on plan. All requests logged for billing and analytics.

### Rate Limit Plans

| Plan | Requests/min |
|------|-------------|
| free | 30 |
| basic | 120 |
| pro | 600 |
| internal | unlimited |

### Tasks

1. `app/middleware/rate_limiter.py`:
   - In-memory sliding window (dict keyed by `api_key_id`, stores timestamps)
   - Check against `api_key.rate_limit_per_minute`
   - Return 429 with `Retry-After` header when exceeded
   - Internal keys bypass rate limiting
   - (Swap to Redis later if scaling to multiple instances)

2. `app/middleware/usage_logger.py`:
   - After each authenticated request, insert row into `usage_logs`
   - Fire-and-forget via FastAPI `BackgroundTasks` or `asyncio.create_task`
   - Log: endpoint, method, status_code, response_time_ms, ip_address, api_key_id

3. Admin usage endpoints:

   | Method | Path | Description |
   |--------|------|-------------|
   | GET | `/api/v1/admin/usage` | Aggregated usage (filterable by key, date range) |
   | GET | `/api/v1/admin/usage/{key_id}` | Usage for a specific key |

### Files to Create

- `garuda-api/app/middleware/__init__.py`
- `garuda-api/app/middleware/rate_limiter.py`
- `garuda-api/app/middleware/usage_logger.py`
- `garuda-api/app/schemas/usage.py`
- `garuda-api/app/crud/usage_log.py`
- Modify: `app/api/v1/admin.py`

### Done When

Key with `rate_limit_per_minute=5` gets 429 after 5 rapid requests. `Retry-After` header present. Requests succeed again after waiting. `usage_logs` table accumulates rows. Admin endpoint returns correct usage counts.

---

## Phase 7: Docker and Deployment

**Goal:** Production-ready Docker image deployable on Coolify.

### Tasks

1. `Dockerfile` — Multi-stage build:
   ```dockerfile
   # Build stage
   FROM python:3.11-slim AS builder
   # ... install dependencies

   # Runtime stage
   FROM python:3.11-slim
   # ... copy only what's needed, non-root user
   EXPOSE 8000
   ENTRYPOINT ["scripts/entrypoint.sh"]
   ```

2. `docker-compose.yml` — Local development:
   ```yaml
   services:
     api:
       build: .
       ports: ["8000:8000"]
       env_file: .env
       depends_on: [db]
     db:
       image: postgres:16
       environment:
         POSTGRES_DB: garuda
         POSTGRES_USER: garuda
         POSTGRES_PASSWORD: garuda
       ports: ["5432:5432"]
   ```

3. `docker-compose.prod.yml` — Production (app only, connects to Coolify PostgreSQL)

4. `scripts/entrypoint.sh`:
   ```bash
   #!/bin/bash
   alembic upgrade head
   uvicorn app.main:app --host 0.0.0.0 --port 8000 --workers 2
   ```

5. Production middleware in `app/main.py`:
   - Security headers: `X-Content-Type-Options`, `X-Frame-Options`, `Strict-Transport-Security`
   - Request ID middleware (UUID per request in response headers and logs)
   - Structured JSON logging via `app/core/logging.py`

6. Enhanced health check: `GET /health` also verifies database connectivity (`SELECT 1`)

7. `.dockerignore` — exclude `.env`, `__pycache__`, `.git`, `tests/`, etc.

### Files to Create

- `garuda-api/Dockerfile`
- `garuda-api/docker-compose.yml`
- `garuda-api/docker-compose.prod.yml`
- `garuda-api/scripts/entrypoint.sh`
- `garuda-api/.dockerignore`
- `garuda-api/app/core/logging.py`
- `garuda-api/app/middleware/request_id.py`
- Modify: `garuda-api/app/main.py`

### Done When

`docker compose up` starts app + local PostgreSQL, runs migrations, health check returns OK. `docker compose -f docker-compose.prod.yml up` connects to external PostgreSQL. Security headers present on all responses. Structured JSON logs on stdout. Docker image runs as non-root. Ready for Coolify deployment.

### Coolify Deployment Notes

- Coolify supports Docker-based deployments — point it at the repo or a built image
- Set environment variables in Coolify's service config: `DATABASE_URL`, `INTERNAL_API_KEY`, `CORS_ORIGINS`
- The PostgreSQL connection string from Coolify's hosted database goes into `DATABASE_URL`
- Coolify handles SSL/TLS termination and domain routing
