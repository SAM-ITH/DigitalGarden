# Build Plan: Docker Compose & Deployment

> **Agent Instructions:** This plan covers the production Docker Compose configuration, Makefile updates, and Coolify deployment setup. This should be done **after** all services are built and verified individually.

---

## Prerequisites

All 5 services must be built and passing their individual acceptance criteria:

- [ ] Auth service (port 8081) — register, login, refresh, profile
- [ ] Content service (port 8082) — content hierarchy, admin CRUD
- [ ] Review service (port 8083) — batch review, due cards
- [ ] Study Plan service (port 8084) — plan CRUD, task scheduling
- [ ] Analytics service (port 8085) — summary, streak, weak areas
- [ ] All Dockerfiles build successfully
- [ ] All services share the same JWT secret

---

## Reference Docs

- `10-api-build-plan.md` — Phase 6 (Steps 6.1–6.4)
- `02-system-architecture.md` — Deployment architecture, Coolify setup, resource allocation

---

## Step 1: Production docker-compose.yml

Create `docker-compose.yml` at the repo root:

```yaml
version: "3.8"

services:
  auth-service:
    build:
      context: .
      dockerfile: services/auth-service/Dockerfile
    environment:
      PORT: "8081"
      DATABASE_URL: ${DATABASE_URL}
      JWT_SECRET: ${JWT_SECRET}
      JWT_ACCESS_EXPIRY: 15m
      ENV: production
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.auth.rule=PathPrefix(`/api/auth`)"
      - "traefik.http.routers.auth.entrypoints=web"
      - "traefik.http.services.auth.loadbalancer.server.port=8081"
      - "traefik.http.middlewares.auth-strip.stripprefix.prefixes=/api/auth"
      - "traefik.http.routers.auth.middlewares=auth-strip"
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "wget", "--spider", "-q", "http://localhost:8081/health"]
      interval: 30s
      timeout: 5s
      retries: 3

  content-service:
    build:
      context: .
      dockerfile: services/content-service/Dockerfile
    environment:
      PORT: "8082"
      DATABASE_URL: ${DATABASE_URL}
      JWT_SECRET: ${JWT_SECRET}
      ENV: production
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.content.rule=PathPrefix(`/api/content`)"
      - "traefik.http.routers.content.entrypoints=web"
      - "traefik.http.services.content.loadbalancer.server.port=8082"
      - "traefik.http.middlewares.content-strip.stripprefix.prefixes=/api/content"
      - "traefik.http.routers.content.middlewares=content-strip"
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "wget", "--spider", "-q", "http://localhost:8082/health"]
      interval: 30s
      timeout: 5s
      retries: 3

  review-service:
    build:
      context: .
      dockerfile: services/review-service/Dockerfile
    environment:
      PORT: "8083"
      DATABASE_URL: ${DATABASE_URL}
      JWT_SECRET: ${JWT_SECRET}
      ENV: production
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.review.rule=PathPrefix(`/api/reviews`)"
      - "traefik.http.routers.review.entrypoints=web"
      - "traefik.http.services.review.loadbalancer.server.port=8083"
      - "traefik.http.middlewares.review-strip.stripprefix.prefixes=/api/reviews"
      - "traefik.http.routers.review.middlewares=review-strip"
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "wget", "--spider", "-q", "http://localhost:8083/health"]
      interval: 30s
      timeout: 5s
      retries: 3

  studyplan-service:
    build:
      context: .
      dockerfile: services/studyplan-service/Dockerfile
    environment:
      PORT: "8084"
      DATABASE_URL: ${DATABASE_URL}
      JWT_SECRET: ${JWT_SECRET}
      ENV: production
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.studyplan.rule=PathPrefix(`/api/study-plans`)"
      - "traefik.http.routers.studyplan.entrypoints=web"
      - "traefik.http.services.studyplan.loadbalancer.server.port=8084"
      - "traefik.http.middlewares.studyplan-strip.stripprefix.prefixes=/api/study-plans"
      - "traefik.http.routers.studyplan.middlewares=studyplan-strip"
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "wget", "--spider", "-q", "http://localhost:8084/health"]
      interval: 30s
      timeout: 5s
      retries: 3

  analytics-service:
    build:
      context: .
      dockerfile: services/analytics-service/Dockerfile
    environment:
      PORT: "8085"
      DATABASE_URL: ${DATABASE_URL}
      JWT_SECRET: ${JWT_SECRET}
      ENV: production
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.analytics.rule=PathPrefix(`/api/analytics`)"
      - "traefik.http.routers.analytics.entrypoints=web"
      - "traefik.http.services.analytics.loadbalancer.server.port=8085"
      - "traefik.http.middlewares.analytics-strip.stripprefix.prefixes=/api/analytics"
      - "traefik.http.routers.analytics.middlewares=analytics-strip"
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "wget", "--spider", "-q", "http://localhost:8085/health"]
      interval: 30s
      timeout: 5s
      retries: 3
```

**Traefik routing summary:**

| Path Prefix | Service | Internal Port | Strip Prefix |
|---|---|---|---|
| `/api/auth/*` | auth-service | 8081 | `/api/auth` |
| `/api/content/*` | content-service | 8082 | `/api/content` |
| `/api/reviews/*` | review-service | 8083 | `/api/reviews` |
| `/api/study-plans/*` | studyplan-service | 8084 | `/api/study-plans` |
| `/api/analytics/*` | analytics-service | 8085 | `/api/analytics` |

**Important:** Each service's internal routes do NOT include the `/api/<service>` prefix. Traefik strips the prefix before forwarding. For example, `GET /api/auth/me` → Traefik strips `/api/auth` → forwards `GET /me` to auth-service.

---

## Step 2: Update Makefile

Add production targets to the existing `Makefile`:

```makefile
# Existing targets (dev, migrate, seed, generate, build, test) remain

# Production build
build-prod:
	docker compose build

# Production up (requires .env with DATABASE_URL, JWT_SECRET)
up:
	docker compose up -d

# Production down
down:
	docker compose down

# Full test suite across all services
test-all:
	cd services/auth-service && go test ./...
	cd services/content-service && go test ./...
	cd services/review-service && go test ./...
	cd services/studyplan-service && go test ./...
	cd services/analytics-service && go test ./...

# Lint all services
lint:
	cd pkg/common && go vet ./...
	cd services/auth-service && go vet ./...
	cd services/content-service && go vet ./...
	cd services/review-service && go vet ./...
	cd services/studyplan-service && go vet ./...
	cd services/analytics-service && go vet ./...

# Generate all sqlc code
generate:
	cd services/auth-service && sqlc generate
	cd services/content-service && sqlc generate
	cd services/review-service && sqlc generate
	cd services/studyplan-service && sqlc generate
	cd services/analytics-service && sqlc generate

# Health check all services
health:
	@echo "Auth:" && curl -s http://localhost:8081/health | jq . || echo "FAILED"
	@echo "Content:" && curl -s http://localhost:8082/health | jq . || echo "FAILED"
	@echo "Review:" && curl -s http://localhost:8083/health | jq . || echo "FAILED"
	@echo "StudyPlan:" && curl -s http://localhost:8084/health | jq . || echo "FAILED"
	@echo "Analytics:" && curl -s http://localhost:8085/health | jq . || echo "FAILED"
```

---

## Step 3: .env File (Production)

Create `.env` (gitignored!) at the repo root:

```
DATABASE_URL=postgresql://snapy:<secure-password>@postgres:5432/snapy?sslmode=disable
JWT_SECRET=<generate-with-openssl-rand-hex-32>
```

**Generate JWT secret:**
```bash
openssl rand -hex 32
```

Add `.env` to `.gitignore`.

---

## Step 4: Dockerfile Build Context

Each service's Dockerfile needs to handle the monorepo build context. The build context is the repo root (`snapy-api/`).

**Update each service's Dockerfile** to copy from the correct paths:

```dockerfile
FROM golang:1.22-alpine AS builder
WORKDIR /app

# Copy go.work files
COPY go.work go.work.sum ./

# Copy common library
COPY pkg/common/ pkg/common/

# Copy service go.mod/go.sum first (layer caching)
COPY services/auth-service/go.mod services/auth-service/go.sum ./services/auth-service/

# Download dependencies
WORKDIR /app/services/auth-service
RUN go mod download

# Copy service source code
COPY services/auth-service/ /app/services/auth-service/

# Build binary
RUN CGO_ENABLED=0 GOOS=linux go build -ldflags="-s -w" -o /app/bin/auth-service ./cmd/api

FROM scratch
COPY --from=builder /app/bin/auth-service /auth-service
COPY --from=builder /etc/ssl/certs/ca-certificates.crt /etc/ssl/certs/
EXPOSE 8081
ENTRYPOINT ["/auth-service"]
```

**Repeat this pattern for each service**, adjusting:
- Service directory name (`auth-service` → `content-service`, etc.)
- Binary output name (`auth-service` → `content-service`, etc.)
- EXPOSE port (`8081` → `8082`, etc.)

**Key points:**
- `go.work` and `go.work.sum` must be copied for the workspace to resolve
- `pkg/common/` must be copied since services depend on it via `go.work`
- The `replace` directive in each service's `go.mod` points to the local common library

---

## Step 5: Coolify Deployment Checklist

### 5.1 Push to GitHub

```bash
git remote add origin <repo-url>
git push -u origin main
```

### 5.2 In Coolify Dashboard

1. **Create application** for each service (5 total)
2. For each service:
   - Point to the repo + branch (`main`)
   - Set build context to repo root
   - Set Dockerfile path: `services/<service-name>/Dockerfile`
   - Configure environment variables (see below)
3. **Configure Traefik** labels (already in docker-compose.yml)

### 5.3 Environment Variables per Service

Each service needs these env vars set in Coolify:

| Variable | Value |
|---|---|
| `PORT` | Service-specific (8081-8085) |
| `DATABASE_URL` | Internal PostgreSQL connection string |
| `JWT_SECRET` | Same 64-char hex string across ALL services |
| `JWT_ACCESS_EXPIRY` | `15m` |
| `ENV` | `production` |

**DATABASE_URL format (internal Docker network):**
```
postgresql://snapy:<password>@<postgres-container-name>:5432/snapy?sslmode=disable
```

### 5.4 Database Setup

1. PostgreSQL is hosted as a Coolify managed service
2. Run migrations before starting services:
   ```bash
   make migrate
   ```
3. Or add a migration init container/entrypoint script

### 5.5 Resource Allocation

| Container | RAM Limit | CPU |
|---|---|---|
| auth-service | 64MB | 0.25 |
| content-service | 64MB | 0.25 |
| review-service | 128MB | 0.5 |
| studyplan-service | 64MB | 0.25 |
| analytics-service | 64MB | 0.25 |
| PostgreSQL | 2GB | 1.0 |
| Traefik + Coolify | 1GB | 0.5 |
| **Total** | **~3.5GB** | **3.0 cores** |

Review service gets more RAM due to batch processing and FSRS computation.

---

## Step 6: End-to-End Verification

After all services are deployed and running through Traefik:

```bash
BASE_URL=https://your-domain.com

# 1. Auth - Register
curl -s -X POST $BASE_URL/api/auth/register \
  -H "Content-Type: application/json" \
  -d '{"phone":"+94771234567","name":"Test User","password":"password123"}'
# → tokens

# 2. Auth - Login
curl -s -X POST $BASE_URL/api/auth/login \
  -H "Content-Type: application/json" \
  -d '{"phone":"+94771234567","password":"password123"}'
# → tokens

# 3. Content - List grades
curl -s $BASE_URL/api/content/grades \
  -H "Authorization: Bearer $TOKEN"
# → grades array

# 4. Review - Get due cards
curl -s $BASE_URL/api/reviews/due \
  -H "Authorization: Bearer $TOKEN"
# → due cards

# 5. Study Plan - Create plan
curl -s -X POST $BASE_URL/api/study-plans/plans \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"name":"Test Plan","goalType":"custom","deadline":"2026-12-31","dailyMinutes":30}'
# → plan with tasks

# 6. Analytics - Summary
curl -s $BASE_URL/api/analytics/summary \
  -H "Authorization: Bearer $TOKEN"
# → dashboard stats

# 7. Health checks through Traefik
curl -s $BASE_URL/api/auth/health
curl -s $BASE_URL/api/content/health
curl -s $BASE_URL/api/reviews/health
curl -s $BASE_URL/api/study-plans/health
curl -s $BASE_URL/api/analytics/health
```

---

## Step 7: Final Checklist

- [ ] All 5 services start without errors in Docker
- [ ] JWT auth works across all services (same secret)
- [ ] Traefik routes correctly to each service
- [ ] Path prefix stripping works (services see clean paths)
- [ ] Health checks pass for all services
- [ ] Auth: register, login, token refresh, profile CRUD
- [ ] Content: grades, subjects, units, cards read; admin CRUD
- [ ] Reviews: batch submit, due cards, SRS state upsert
- [ ] Study plans: create plan, list plans, today's tasks, complete task
- [ ] Analytics: summary, streak heatmap, weak areas
- [ ] Admin routes protected (403 for non-admin)
- [ ] Migrations run cleanly on production database
- [ ] Seed data loads correctly
- [ ] All Docker images build from repo root context
- [ ] `.env` is gitignored
- [ ] All services restart on failure (`restart: unless-stopped`)
