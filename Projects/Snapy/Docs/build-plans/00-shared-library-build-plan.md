# Build Plan: Shared Library & Project Foundation

> **Agent Instructions:** This plan covers the monorepo initialization, shared Go library (`pkg/common/`), database migrations, seed data, and dev docker-compose. **All other services depend on this being complete first.**

---

## Context

Snapy API is a Go microservice monorepo. Every service shares a common library for config, database, JWT, middleware, response formatting, and validation. All services connect to a single PostgreSQL 16 database with a shared migration history.

**Reference docs:**
- `10-api-build-plan.md` — Phase 0 (Steps 0.1–0.5)
- `08-database-design.md` — Full schema with all tables, columns, types, indexes
- `05-backend-system.md` — Error codes, middleware stack, response format

---

## Step 1: Initialize Go Workspace

```bash
mkdir snapy-api && cd snapy-api
git init
go work init ./pkg/common ./services/auth-service ./services/content-service ./services/review-service ./services/studyplan-service ./services/analytics-service
```

Create the top-level directory structure:

```
snapy-api/
├── go.work
├── pkg/common/
├── services/
│   ├── auth-service/
│   ├── content-service/
│   ├── review-service/
│   ├── studyplan-service/
│   └── analytics-service/
├── db/
│   ├── migrations/
│   └── seed/
├── docker-compose.yml
├── docker-compose.dev.yml
└── Makefile
```

---

## Step 2: Create Shared Library (`pkg/common/`)

```bash
mkdir -p pkg/common/{config,database,jwt,middleware,response,validation}
cd pkg/common
go mod init github.com/yourusername/snapy-api/pkg/common
```

### Dependencies

```bash
go get github.com/go-chi/chi/v5
go get github.com/go-chi/cors
go get github.com/golang-jwt/jwt/v5
go get github.com/jackc/pgx/v5
go get github.com/jackc/pgx/v5/pgxpool
go get github.com/rs/zerolog
go get github.com/go-playground/validator/v10
```

### 2.1 `pkg/common/config/config.go`

```go
package config

type Config struct {
    Port            string
    DatabaseURL     string
    JWTSecret       string
    JWTAccessExpiry string
    Environment     string
}

func Load() (*Config, error)
```

- `Port` — default `8080`
- `DatabaseURL` — required, from `DATABASE_URL` env var
- `JWTSecret` — required, from `JWT_SECRET` env var
- `JWTAccessExpiry` — default `15m`
- `Environment` — default `development`
- Parse `JWTAccessExpiry` into `time.Duration` with a helper method `GetAccessExpiry() time.Duration`

### 2.2 `pkg/common/database/postgres.go`

```go
package database

func NewPool(ctx context.Context, databaseURL string) (*pgxpool.Pool, error)
```

- Configure pool: `MinConns: 2`, `MaxConns: 10`
- Call `pool.Ping(ctx)` to verify connection
- Log connection success with zerolog
- Return pool or descriptive error

### 2.3 `pkg/common/jwt/jwt.go`

```go
package jwt

type Claims struct {
    UserID string `json:"sub"`
    GradeID string `json:"grade_id"`
    Role    string `json:"role"`
    jwt.RegisteredClaims
}

func GenerateToken(secret string, userID, gradeID, role string, expiry time.Duration) (string, error)
func ValidateToken(secret string, tokenString string) (*Claims, error)
func ExtractBearerToken(r *http.Request) (string, error)
```

- `GenerateToken` — create signed JWT with HS256, set `iat`, `exp`, `sub`
- `ValidateToken` — parse and verify signature + expiry, return `Claims`
- `ExtractBearerToken` — strip "Bearer " prefix from Authorization header, return error if missing/malformed

### 2.4 `pkg/common/middleware/auth.go`

```go
package middleware

type contextKey string
const ClaimsKey contextKey = "claims"

func JWTAuthMiddleware(jwtSecret string) func(http.Handler) http.Handler
func GetClaimsFromContext(ctx context.Context) (*jwt.Claims, error)
```

- Extract token via `jwt.ExtractBearerToken(r)`
- Validate via `jwt.ValidateToken(jwtSecret, token)`
- Store `*Claims` in request context using `ClaimsKey`
- Return 401 with `UNAUTHORIZED` error code on failure
- `GetClaimsFromContext` — helper for handlers to retrieve claims

### 2.5 `pkg/common/middleware/admin.go`

```go
package middleware

func AdminOnlyMiddleware(next http.Handler) http.Handler
```

- Read claims from context via `GetClaimsFromContext`
- Check `claims.Role == "admin"`
- Return 403 with `FORBIDDEN` error code if not admin

### 2.6 `pkg/common/middleware/cors.go`

```go
package middleware

func CORSMiddleware() func(http.Handler) http.Handler
```

- Allow all origins (`"*"` — mobile apps)
- Allow methods: GET, POST, PUT, PATCH, DELETE, OPTIONS
- Allow headers: Authorization, Content-Type
- Use `github.com/go-chi/cors` package

### 2.7 `pkg/common/middleware/logger.go`

```go
package middleware

func LoggerMiddleware(logger zerolog.Logger) func(http.Handler) http.Handler
```

- Log: method, path, status code, latency, remote IP, request ID
- Use `zerolog` for structured JSON logging
- Wrap `chi.middleware.RequestID` to attach ID

### 2.8 `pkg/common/middleware/recovery.go`

```go
package middleware

func RecoveryMiddleware(logger zerolog.Logger) func(http.Handler) http.Handler
```

- Recover from panics
- Log stack trace and error message
- Return 500 with `INTERNAL_ERROR` error code

### 2.9 `pkg/common/response/response.go`

```go
package response

type ErrorResponse struct {
    Error ErrorDetail `json:"error"`
}

type ErrorDetail struct {
    Code    string      `json:"code"`
    Message string      `json:"message"`
    Details interface{} `json:"details,omitempty"`
}

func JSON(w http.ResponseWriter, status int, data interface{})
func Error(w http.ResponseWriter, status int, code string, message string)
func ErrorWithDetails(w http.ResponseWriter, status int, code string, message string, details interface{})
func NoContent(w http.ResponseWriter, status int)
```

**Error codes to use across all services:**

| Code | HTTP Status | When |
|------|-------------|------|
| `VALIDATION_ERROR` | 400 | Invalid input |
| `UNAUTHORIZED` | 401 | Missing/invalid token |
| `TOKEN_EXPIRED` | 401 | Token past expiry |
| `FORBIDDEN` | 403 | Insufficient role |
| `NOT_FOUND` | 404 | Resource missing |
| `CONFLICT` | 409 | Duplicate resource |
| `INTERNAL_ERROR` | 500 | Unexpected error |

### 2.10 `pkg/common/validation/validation.go`

```go
package validation

var Validate *validator.Validate

func Init()
func ValidateStruct(s interface{}) error
```

- `Validate` — singleton `validator.Validate` instance
- `Init()` — called once at startup, registers custom validators if needed
- `ValidateStruct` — returns formatted error messages mapping field → issue

---

## Step 3: Database Migrations

Create `db/migrations/000001_init_schema.up.sql`.

**Tables in dependency order** (refer to `08-database-design.md` for exact columns, types, constraints):

1. `grades` — PK: UUID, columns: name, level (UNIQUE), education_stage, display_order
2. `users` — PK: UUID, FK → grades.id (nullable), phone (UNIQUE), name, stream, role, timestamps
3. `subjects` — PK: UUID, FK → grades.id, name, name_si, name_ta, icon, color, is_compulsory, stream, display_order, timestamp
4. `terms` — PK: UUID, FK → subjects.id, name, display_order
5. `units` — PK: UUID, FK → terms.id, name, description, card_count, version, display_order, timestamps
6. `cards` — PK: UUID, FK → units.id, type, question, answer, choices (JSONB), explanation, display_order, timestamps. CHECK constraint: answer IS NOT NULL OR choices IS NOT NULL
7. `refresh_tokens` — PK: UUID, FK → users.id, token_hash, expires_at, created_at
8. `card_reviews` — PK: UUID, FK → users.id + cards.id, rating (SMALLINT), reviewed_at, created_at
9. `card_srs_state` — Composite PK (user_id, card_id), FK → users.id + cards.id, stability, difficulty, elapsed_days, scheduled_days, state, due_at, last_reviewed_at, reps, lapses
10. `study_plans` — PK: UUID, FK → users.id + terms.id (nullable), name, goal_type, deadline, daily_minutes, is_active, created_at
11. `study_plan_tasks` — PK: UUID, FK → study_plans.id + units.id, scheduled_date, task_type, estimated_minutes, is_completed, completed_at
12. `user_scores` — PK: UUID, FK → users.id + grades.id + subjects.id (nullable), points, week_start, created_at
13. `user_streaks` — PK: user_id (FK → users.id), current_streak, longest_streak, last_active_date
14. `daily_activity` — Composite PK (user_id, activity_date), sessions_count, cards_reviewed, correct_count, study_minutes
15. `device_tokens` — PK: UUID, FK → users.id, platform, token, timestamps. UNIQUE (user_id, platform, token)

**All indexes** (refer to `08-database-design.md` index section for each table):

- `users_phone_idx` UNIQUE on `phone`
- `refresh_tokens_token_hash_idx` on `token_hash`
- `refresh_tokens_user_id_idx` on `user_id`
- `subjects_grade_id_order_idx` on `(grade_id, display_order)`
- `terms_subject_id_order_idx` on `(subject_id, display_order)`
- `units_term_id_order_idx` on `(term_id, display_order)`
- `cards_unit_id_order_idx` on `(unit_id, display_order)`
- `card_reviews_user_id_reviewed_at_idx` on `(user_id, reviewed_at)`
- `card_reviews_user_id_card_id_idx` on `(user_id, card_id)`
- `card_srs_state_user_due_idx` on `(user_id, due_at)`
- `card_srs_state_user_state_idx` on `(user_id, state)`
- `study_plans_user_id_active_idx` on `(user_id, is_active)`
- `study_plan_tasks_plan_date_idx` on `(plan_id, scheduled_date)`
- `study_plan_tasks_plan_completed_idx` on `(plan_id, is_completed)`
- `user_scores_grade_week_idx` on `(grade_id, week_start, points DESC)`
- `user_scores_subject_week_idx` on `(subject_id, week_start, points DESC)`
- `user_scores_user_id_idx` on `(user_id)`
- `daily_activity_user_date_idx` on `(user_id, activity_date DESC)`
- `device_tokens_user_id_idx` on `(user_id)`

Create `db/migrations/000001_init_schema.down.sql` with DROP TABLE in **reverse** dependency order (15 → 1).

Create `db/migrations/000002_add_password_hash.up.sql`:
```sql
ALTER TABLE users ADD COLUMN password_hash VARCHAR(255) NULL;
```

Create `db/migrations/000002_add_password_hash.down.sql`:
```sql
ALTER TABLE users DROP COLUMN password_hash;
```

---

## Step 4: Seed Data Script

Create `db/seed/seed.go`:

- Connect via `DATABASE_URL` env var
- Use `pgxpool`
- Use parameterized queries with `ON CONFLICT DO NOTHING`
- **Grades (4 rows):**

| name | level | education_stage | display_order |
|------|-------|-----------------|---------------|
| Grade 10 | 10 | ol | 1 |
| Grade 11 | 11 | ol | 2 |
| Grade 12 | 12 | al | 3 |
| Grade 13 | 13 | al | 4 |

- **O/L Subjects (per grade 10 & 11):** Mathematics, Science, English, Sinhala, Tamil, History, Religion, ICT — all `is_compulsory=true`, `stream=NULL`
- **A/L Subjects (per grade 12 & 13):**
  - Science stream: Combined Mathematics, Physics, Chemistry, Biology
  - Commerce stream: Accounting, Business Studies, Economics
  - Arts stream: History, Political Science, Geography, Sinhala, Logic
  - Technology stream: Engineering Technology, Science for Technology, Information & Communication Technology
  - Set `stream` field accordingly, `is_compulsory=false` for optional subjects
- **Terms:** 3 terms per subject ("Term 1", "Term 2", "Term 3")
- **Sample data:** 2 sample units per term, 5 sample cards per unit (mix of classic and mcq types)
- Print progress to stdout as rows are inserted

---

## Step 5: Dev Docker Compose

Create `docker-compose.dev.yml`:

```yaml
version: "3.8"
services:
  postgres:
    image: postgres:16-alpine
    environment:
      POSTGRES_USER: snapy
      POSTGRES_PASSWORD: snapydev
      POSTGRES_DB: snapy
    ports:
      - "5432:5432"
    volumes:
      - pgdata:/var/lib/postgresql/data

volumes:
  pgdata:
```

---

## Step 6: Makefile

Create `Makefile`:

```makefile
.PHONY: dev migrate migrate-down seed generate build test

dev:
	docker compose -f docker-compose.dev.yml up -d

dev-down:
	docker compose -f docker-compose.dev.yml down

migrate:
	migrate -path db/migrations -database $(DATABASE_URL) up

migrate-down:
	migrate -path db/migrations -database $(DATABASE_URL) down 1

seed:
	go run db/seed/seed.go

generate:
	cd services/auth-service && sqlc generate
	cd services/content-service && sqlc generate
	cd services/review-service && sqlc generate
	cd services/studyplan-service && sqlc generate
	cd services/analytics-service && sqlc generate

build:
	go build ./pkg/common/...

test:
	go test ./pkg/common/...
```

---

## Acceptance Criteria

- [ ] `go work init` succeeds, `go.work` file created
- [ ] `go build ./...` passes in `pkg/common/`
- [ ] All types compile and exports are accessible
- [ ] `docker compose -f docker-compose.dev.yml up -d` starts PostgreSQL
- [ ] `make migrate` runs all migrations against empty database without errors
- [ ] `make migrate-down` rolls back cleanly
- [ ] `go run db/seed/seed.go` populates all seed data
- [ ] Running seed script again does not duplicate data (ON CONFLICT DO NOTHING)
- [ ] All foreign key relationships are valid after seeding
- [ ] All indexes are created (verify with `\di` in psql)
- [ ] `psql postgresql://snapy:snapydev@localhost:5432/snapy` connects successfully

---

## Output for Other Agents

After completing this plan, the following are available for service agents:

1. **`pkg/common/`** — importable shared library with config, DB, JWT, middleware, response, validation
2. **`db/migrations/`** — full schema with all 15 tables + indexes + constraints
3. **`db/seed/`** — populated test data
4. **`docker-compose.dev.yml`** — running PostgreSQL on port 5432
5. **`Makefile`** — common commands

Each service agent should:
- `go mod edit -replace github.com/yourusername/snapy-api/pkg/common=../../pkg/common`
- Import shared packages as needed
- Run `sqlc generate` with schema pointing to `../../db/migrations`
