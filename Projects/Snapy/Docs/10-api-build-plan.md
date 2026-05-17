# Snapy API — Microservice Build Plan

> Step-by-step instructions for an LLM agent to build the Snapy CRUD API as separate microservice containers, hosted on Coolify with a shared PostgreSQL database.

---

## Architecture Overview

```
                    ┌─────────────┐
                    │   Traefik   │  (Coolify managed)
                    │  (Gateway)  │
                    └──────┬──────┘
                           │
          ┌────────────────┼────────────────┬────────────────┬────────────────┐
          │                │                │                │                │
   ┌──────▼──────┐  ┌──────▼──────┐  ┌──────▼──────┐  ┌──────▼──────┐  ┌──────▼──────┐
   │ Auth Service │  │  Content    │  │  Review     │  │  Study Plan │  │  Analytics  │
   │  :8081       │  │  Service    │  │  Service    │  │  Service    │  │  Service    │
   │             │  │  :8082       │  │  :8083       │  │  :8084       │  │  :8085       │
   └──────┬──────┘  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘
          │                │                │                │                │
          └────────────────┴────────────────┴────────────────┴────────────────┘
                                    │
                            ┌───────▼───────┐
                            │  PostgreSQL   │
                            │  (Coolify)    │
                            └───────────────┘
```

**Traefik routing rules:**
| Path Prefix | Service | Port |
|---|---|---|
| `/api/auth/*` | auth-service | 8081 |
| `/api/content/*` | content-service | 8082 |
| `/api/reviews/*` | review-service | 8083 |
| `/api/study-plans/*` | studyplan-service | 8084 |
| `/api/analytics/*` | analytics-service | 8085 |

**Phase 1 scope:** PostgreSQL only. Simple JWT auth. No Redis, no OTP, no push notifications.

---

## Monorepo Structure

```
snapy-api/
├── go.work
├── go.work.sum
├── pkg/
│   └── common/
│       ├── go.mod
│       ├── go.sum
│       ├── config/
│       │   └── config.go
│       ├── database/
│       │   └── postgres.go
│       ├── jwt/
│       │   └── jwt.go
│       ├── middleware/
│       │   ├── auth.go
│       │   ├── admin.go
│       │   ├── cors.go
│       │   ├── logger.go
│       │   └── recovery.go
│       ├── response/
│       │   └── response.go
│       └── validation/
│           └── validation.go
├── services/
│   ├── auth-service/
│   │   ├── go.mod
│   │   ├── go.sum
│   │   ├── cmd/
│   │   │   └── api/
│   │   │       └── main.go
│   │   ├── internal/
│   │   │   ├── handler/
│   │   │   │   └── auth.go
│   │   │   ├── service/
│   │   │   │   └── auth.go
│   │   │   ├── repository/
│   │   │   │   ├── db.go          (sqlc generated)
│   │   │   │   ├── models.go      (sqlc generated)
│   │   │   │   └── users.sql.go   (sqlc generated)
│   │   │   └── model/
│   │   │       ├── request.go
│   │   │       └── response.go
│   │   ├── db/
│   │   │   ├── migrations/
│   │   │   └── queries/
│   │   │       └── users.sql
│   │   ├── sqlc.yaml
│   │   ├── Dockerfile
│   │   └── .env.example
│   ├── content-service/
│   │   ├── (same structure)
│   │   ├── db/queries/
│   │   │   ├── grades.sql
│   │   │   ├── subjects.sql
│   │   │   ├── terms.sql
│   │   │   ├── units.sql
│   │   │   └── cards.sql
│   │   ├── sqlc.yaml
│   │   ├── Dockerfile
│   │   └── .env.example
│   ├── review-service/
│   │   ├── (same structure)
│   │   ├── db/queries/
│   │   │   ├── reviews.sql
│   │   │   └── srs_state.sql
│   │   ├── sqlc.yaml
│   │   ├── Dockerfile
│   │   └── .env.example
│   ├── studyplan-service/
│   │   ├── (same structure)
│   │   ├── db/queries/
│   │   │   ├── study_plans.sql
│   │   │   └── study_plan_tasks.sql
│   │   ├── sqlc.yaml
│   │   ├── Dockerfile
│   │   └── .env.example
│   └── analytics-service/
│       ├── (same structure)
│       ├── db/queries/
│       │   ├── analytics.sql
│       │   ├── streaks.sql
│       │   └── daily_activity.sql
│       ├── sqlc.yaml
│       ├── Dockerfile
│       └── .env.example
├── db/
│   ├── migrations/
│   │   ├── 000001_init_schema.up.sql
│   │   └── 000001_init_schema.down.sql
│   └── seed/
│       └── seed.go
├── docker-compose.yml
├── docker-compose.dev.yml
└── Makefile
```

---

## Phase 0: Project Foundation

### Step 0.1 — Initialize the monorepo

```bash
mkdir snapy-api && cd snapy-api
git init
go work init ./pkg/common ./services/auth-service ./services/content-service ./services/review-service ./services/studyplan-service ./services/analytics-service
```

### Step 0.2 — Create shared library (`pkg/common/`)

```bash
mkdir -p pkg/common/{config,database,jwt,middleware,response,validation}
cd pkg/common
go mod init github.com/yourusername/snapy-api/pkg/common
```

**Install shared dependencies:**
```bash
go get github.com/go-chi/chi/v5
go get github.com/go-chi/cors
go get github.com/golang-jwt/jwt/v5
go get github.com/jackc/pgx/v5
go get github.com/jackc/pgx/v5/pgxpool
go get github.com/rs/zerolog
go get github.com/go-playground/validator/v10
```

**Files to create in `pkg/common/`:**

#### `pkg/common/config/config.go`
- `Config` struct with fields: `Port string`, `DatabaseURL string`, `JWTSecret string`, `JWTAccessExpiry string`, `Environment string`
- `func Load() (*Config, error)` — reads from env vars with defaults
- Use `os.Getenv` with fallback defaults (port 8080, 15m access expiry)

#### `pkg/common/database/postgres.go`
- `func NewPool(ctx context.Context, databaseURL string) (*pgxpool.Pool, error)`
- Configure pool: min 2, max 10 connections
- Test connection on init with `pool.Ping(ctx)`

#### `pkg/common/jwt/jwt.go`
- `type Claims struct` with `UserID string`, `GradeID string`, `Role string`, plus registered claims (`exp`, `iat`, `sub`)
- `func GenerateToken(secret string, userID, gradeID, role string, expiry time.Duration) (string, error)`
- `func ValidateToken(secret string, tokenString string) (*Claims, error)`
- `func ExtractBearerToken(r *http.Request) (string, error)` — extract from Authorization header

#### `pkg/common/middleware/auth.go`
- `func JWTAuthMiddleware(jwtSecret string) func(http.Handler) http.Handler`
- Extract token, validate, put `Claims` into request context
- Return 401 on invalid/expired tokens
- Use context key type for storing claims

#### `pkg/common/middleware/admin.go`
- `func AdminOnlyMiddleware(next http.Handler) http.Handler`
- Read claims from context, check `role == "admin"`, return 403 if not

#### `pkg/common/middleware/cors.go`
- `func CORSMiddleware() func(http.Handler) http.Handler`
- Allow all origins (mobile apps), methods: GET/POST/PUT/PATCH/DELETE, headers: Authorization, Content-Type

#### `pkg/common/middleware/logger.go`
- `func LoggerMiddleware() func(http.Handler) http.Handler`
- Use zerolog to log method, path, status, latency, request ID

#### `pkg/common/middleware/recovery.go`
- `func RecoveryMiddleware() func(http.Handler) http.Handler`
- Recover from panics, return 500, log stack trace

#### `pkg/common/response/response.go`
- `type ErrorResponse struct` with `Error ErrorDetail` nested
- `type ErrorDetail struct` with `Code string`, `Message string`, `Details interface{}`
- `func Error(w http.ResponseWriter, status int, code string, message string)`
- `func ErrorWithDetails(w http.ResponseWriter, status int, code string, message string, details interface{})`
- `func JSON(w http.ResponseWriter, status int, data interface{})`
- `func NoContent(w http.ResponseWriter, status int)`
- Error codes: `VALIDATION_ERROR`, `UNAUTHORIZED`, `TOKEN_EXPIRED`, `FORBIDDEN`, `NOT_FOUND`, `INTERNAL_ERROR`, `RATE_LIMITED`

#### `pkg/common/validation/validation.go`
- `var Validate *validator.Validate` — singleton
- `func ValidateStruct(s interface{}) error` — returns formatted error messages

**Acceptance criteria:**
- `go build ./...` passes in `pkg/common/`
- All types compile and exports are correct

---

### Step 0.3 — Write database migrations

Create `db/migrations/000001_init_schema.up.sql` with all tables from the database design doc:

```sql
-- Extensions
CREATE EXTENSION IF NOT EXISTS "pgcrypto";

-- Tables in dependency order:
-- 1. grades (no dependencies)
-- 2. users (depends on grades)
-- 3. subjects (depends on grades)
-- 4. terms (depends on subjects)
-- 5. units (depends on terms)
-- 6. cards (depends on units)
-- 7. refresh_tokens (depends on users)
-- 8. card_reviews (depends on users, cards)
-- 9. card_srs_state (depends on users, cards)
-- 10. study_plans (depends on users, terms)
-- 11. study_plan_tasks (depends on study_plans, units)
-- 12. user_scores (depends on users, grades, subjects)
-- 13. user_streaks (depends on users)
-- 14. daily_activity (depends on users)
-- 15. device_tokens (depends on users)
```

Include all columns, types, constraints, indexes, and composite primary keys exactly as specified in `08-database-design.md`.

Create `db/migrations/000001_init_schema.down.sql` with DROP TABLE statements in reverse dependency order.

**Acceptance criteria:**
- Migration runs cleanly against an empty PostgreSQL 16 database
- All indexes and constraints are created
- `down` migration drops all tables without errors

---

### Step 0.4 — Write seed data script

Create `db/seed/seed.go`:
- Connect to database using DATABASE_URL env var
- Insert 4 grades: Grade 10 (ol), Grade 11 (ol), Grade 12 (al), Grade 13 (al)
- Insert O/L compulsory subjects per grade (Mathematics, Science, English, Sinhala/Tamil, History, Religion, ICT)
- Insert A/L subjects grouped by stream (Science, Commerce, Arts, Technology)
- Insert 3 terms per subject
- Insert sample units and cards for testing
- Use parameterized queries, handle conflicts with ON CONFLICT DO NOTHING
- Print progress to stdout

**Acceptance criteria:**
- `go run db/seed/seed.go` populates the database
- Running it again does not duplicate data
- All foreign key relationships are valid

---

### Step 0.5 — Create docker-compose.dev.yml

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

**Acceptance criteria:**
- `docker compose -f docker-compose.dev.yml up -d` starts PostgreSQL
- `psql postgresql://snapy:snapydev@localhost:5432/snapy` connects successfully

---

## Phase 1: Auth Service

### Step 1.1 — Initialize auth-service module

```bash
cd services/auth-service
go mod init github.com/yourusername/snapy-api/services/auth-service
```

Add dependency on shared library:
```bash
go mod edit -replace github.com/yourusername/snapy-api/pkg/common=../../pkg/common
go mod tidy
```

**Install service-specific dependencies:**
```bash
go get github.com/go-chi/chi/v5
go get github.com/golang-jwt/jwt/v5
go get github.com/go-playground/validator/v10
go get github.com/jackc/pgx/v5
go get github.com/jackc/pgx/v5/pgxpool
go get github.com/golang-migrate/migrate/v4
go get github.com/rs/zerolog
go get golang.org/x/crypto/bcrypt
```

### Step 1.2 — Configure sqlc for auth-service

Create `services/auth-service/sqlc.yaml`:
```yaml
version: "2"
sql:
  - engine: "postgresql"
    queries: "db/queries"
    schema: "../../db/migrations"
    gen:
      go:
        package: "repository"
        out: "internal/repository"
        sql_package: "pgx/v5"
        emit_json_tags: true
        emit_empty_slices: true
        overrides:
          - db_type: "uuid"
            go_type: "github.com/google/uuid.UUID"
          - db_type: "timestamptz"
            go_type: "time.Time"
```

### Step 1.3 — Write sqlc queries

Create `services/auth-service/db/queries/users.sql`:

```sql
-- name: GetUserByID :one
SELECT id, phone, name, grade_id, stream, role, created_at, updated_at
FROM users
WHERE id = $1;

-- name: GetUserByPhone :one
SELECT id, phone, name, grade_id, stream, role, created_at, updated_at
FROM users
WHERE phone = $1;

-- name: CreateUser :one
INSERT INTO users (phone, name, role)
VALUES ($1, $2, 'user')
RETURNING id, phone, name, grade_id, stream, role, created_at, updated_at;

-- name: UpdateUserProfile :one
UPDATE users
SET name = $2, grade_id = $3, stream = $4, updated_at = NOW()
WHERE id = $1
RETURNING id, phone, name, grade_id, stream, role, created_at, updated_at;

-- name: UpdateUserPassword :one
UPDATE users
SET password_hash = $2, updated_at = NOW()
WHERE id = $1
RETURNING id, phone, name, grade_id, stream, role, created_at, updated_at;
```

> **NOTE:** Add a `password_hash` column to the users table in a new migration `000002_add_password_hash.up.sql`:
> ```sql
> ALTER TABLE users ADD COLUMN password_hash VARCHAR(255) NULL;
> ```

Create `services/auth-service/db/queries/tokens.sql`:

```sql
-- name: CreateRefreshToken :one
INSERT INTO refresh_tokens (user_id, token_hash, expires_at)
VALUES ($1, $2, $3)
RETURNING id, user_id, token_hash, expires_at, created_at;

-- name: GetRefreshTokenByHash :one
SELECT id, user_id, token_hash, expires_at, created_at
FROM refresh_tokens
WHERE token_hash = $1;

-- name: DeleteRefreshToken :exec
DELETE FROM refresh_tokens WHERE id = $1;

-- name: DeleteRefreshTokensByUser :exec
DELETE FROM refresh_tokens WHERE user_id = $1;
```

Generate code:
```bash
cd services/auth-service
sqlc generate
```

### Step 1.4 — Create request/response models

Create `services/auth-service/internal/model/request.go`:

```go
type RegisterRequest struct {
    Phone    string `json:"phone" validate:"required,min=10,max=15"`
    Name     string `json:"name" validate:"required,min=2,max=100"`
    Password string `json:"password" validate:"required,min=6,max=72"`
}

type LoginRequest struct {
    Phone    string `json:"phone" validate:"required"`
    Password string `json:"password" validate:"required"`
}

type RefreshTokenRequest struct {
    RefreshToken string `json:"refreshToken" validate:"required"`
}

type UpdateProfileRequest struct {
    Name    *string `json:"name" validate:"omitempty,min=2,max=100"`
    GradeID *string `json:"gradeId" validate:"omitempty,uuid"`
    Stream  *string `json:"stream" validate:"omitempty,oneof=science commerce arts technology"`
}
```

Create `services/auth-service/internal/model/response.go`:

```go
type AuthResponse struct {
    AccessToken  string          `json:"accessToken"`
    RefreshToken string          `json:"refreshToken"`
    IsNewUser    bool            `json:"isNewUser"`
    User         UserProfileResponse `json:"user"`
}

type UserProfileResponse struct {
    ID        string  `json:"id"`
    Phone     string  `json:"phone"`
    Name      *string `json:"name"`
    GradeID   *string `json:"gradeId"`
    Stream    *string `json:"stream"`
    Role      string  `json:"role"`
    CreatedAt string  `json:"createdAt"`
}
```

### Step 1.5 — Create service layer

Create `services/auth-service/internal/service/auth.go`:

```go
type AuthService interface {
    Register(ctx context.Context, req model.RegisterRequest) (*model.AuthResponse, error)
    Login(ctx context.Context, req model.LoginRequest) (*model.AuthResponse, error)
    RefreshToken(ctx context.Context, refreshToken string) (*model.AuthResponse, error)
    GetProfile(ctx context.Context, userID string) (*model.UserProfileResponse, error)
    UpdateProfile(ctx context.Context, userID string, req model.UpdateProfileRequest) (*model.UserProfileResponse, error)
}

type authService struct {
    queries  *repository.Queries
    jwtSecret string
    accessExpiry time.Duration
    refreshExpiry time.Duration
}
```

**Register logic:**
1. Check if user with phone already exists → return error if yes
2. Hash password with bcrypt (cost 12)
3. Insert user with phone, name, password_hash
4. Generate JWT access token (15 min)
5. Generate opaque refresh token, hash with SHA-256, store in `refresh_tokens`
6. Return `AuthResponse`

**Login logic:**
1. Find user by phone → return 401 if not found
2. Compare bcrypt hash → return 401 if mismatch
3. Generate access + refresh tokens
4. Return `AuthResponse`

**RefreshToken logic:**
1. Hash the incoming refresh token with SHA-256
2. Look up in `refresh_tokens` by hash → return 401 if not found or expired
3. Delete old refresh token (rotation)
4. Generate new access + refresh token pair
5. Return `AuthResponse`

**GetProfile logic:**
1. `GetUserByID` from database
2. Map to `UserProfileResponse`

**UpdateProfile logic:**
1. Validate non-nil fields
2. `UpdateUserProfile` in database
3. Map to `UserProfileResponse`

### Step 1.6 — Create handler layer

Create `services/auth-service/internal/handler/auth.go`:

```go
type AuthHandler struct {
    service service.AuthService
}

func NewAuthHandler(service service.AuthService) *AuthHandler

func (h *AuthHandler) Register(w http.ResponseWriter, r *http.Request)
func (h *AuthHandler) Login(w http.ResponseWriter, r *http.Request)
func (h *AuthHandler) RefreshToken(w http.ResponseWriter, r *http.Request)
func (h *AuthHandler) GetProfile(w http.ResponseWriter, r *http.Request)
func (h *AuthHandler) UpdateProfile(w http.ResponseWriter, r *http.Request)
```

Each handler:
1. Decode JSON body (for POST/PATCH)
2. Validate with `validation.ValidateStruct()`
3. Call service method
4. Return JSON response or error

### Step 1.7 — Create main.go

Create `services/auth-service/cmd/api/main.go`:

```go
func main() {
    // 1. Load config
    // 2. Connect to PostgreSQL
    // 3. Run migrations (golang-migrate, path ../../db/migrations)
    // 4. Create repository (sqlc Queries)
    // 5. Create service
    // 6. Create handler
    // 7. Create chi router with middleware chain:
    //    - RequestID, Logger, Recovery, CORS
    // 8. Register routes:
    //    POST /register          → handler.Register
    //    POST /login             → handler.Login
    //    POST /token/refresh     → handler.RefreshToken
    //    GET  /me                → handler.GetProfile      (JWT protected)
    //    PATCH /me               → handler.UpdateProfile   (JWT protected)
    // 9. Start HTTP server on config.Port
}
```

### Step 1.8 — Create Dockerfile

Create `services/auth-service/Dockerfile`:

```dockerfile
FROM golang:1.22-alpine AS builder
WORKDIR /app
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 GOOS=linux go build -ldflags="-s -w" -o /auth-service ./cmd/api

FROM scratch
COPY --from=builder /auth-service /auth-service
COPY --from=builder /etc/ssl/certs/ca-certificates.crt /etc/ssl/certs/
EXPOSE 8081
ENTRYPOINT ["/auth-service"]
```

### Step 1.9 — Create .env.example

```
PORT=8081
DATABASE_URL=postgresql://snapy:snapydev@localhost:5432/snapy?sslmode=disable
JWT_SECRET=change-me-to-a-64-char-random-string
JWT_ACCESS_EXPIRY=15m
ENV=development
```

### Step 1.10 — Verification

```bash
cd services/auth-service
go mod tidy
go build ./...
sqlc generate
go test ./...

# Manual test with docker-compose
docker compose -f docker-compose.dev.yml up -d postgres
# Run migrations
# Start service
# Test endpoints:
curl -X POST http://localhost:8081/register -d '{"phone":"+94771234567","name":"Test User","password":"password123"}'
curl -X POST http://localhost:8081/login -d '{"phone":"+94771234567","password":"password123"}'
curl -X GET http://localhost:8081/me -H "Authorization: Bearer <token>"
```

**Acceptance criteria:**
- Service compiles and starts
- Register creates user and returns JWT
- Login returns JWT for existing user
- Refresh token rotates correctly
- Protected endpoints reject requests without valid JWT
- Dockerfile builds successfully

---

## Phase 2: Content Service

### Step 2.1 — Initialize content-service

```bash
cd services/content-service
go mod init github.com/yourusername/snapy-api/services/content-service
go mod edit -replace github.com/yourusername/snapy-api/pkg/common=../../pkg/common
```

### Step 2.2 — Configure sqlc

Create `services/content-service/sqlc.yaml` (same pattern as auth, different output).

### Step 2.3 — Write sqlc queries

Create these query files under `services/content-service/db/queries/`:

**grades.sql:**
```sql
-- name: ListGrades :many
SELECT id, name, level, education_stage, display_order
FROM grades
ORDER BY display_order;

-- name: GetGradeByID :one
SELECT id, name, level, education_stage, display_order
FROM grades
WHERE id = $1;

-- name: CreateGrade :one
INSERT INTO grades (name, level, education_stage, display_order)
VALUES ($1, $2, $3, $4)
RETURNING id, name, level, education_stage, display_order;
```

**subjects.sql:**
```sql
-- name: ListSubjectsByGrade :many
SELECT id, grade_id, name, name_si, name_ta, icon, color, is_compulsory, stream, display_order
FROM subjects
WHERE grade_id = $1
ORDER BY display_order;

-- name: GetSubjectByID :one
SELECT id, grade_id, name, name_si, name_ta, icon, color, is_compulsory, stream, display_order
FROM subjects
WHERE id = $1;

-- name: CreateSubject :one
INSERT INTO subjects (grade_id, name, name_si, name_ta, icon, color, is_compulsory, stream, display_order)
VALUES ($1, $2, $3, $4, $5, $6, $7, $8, $9)
RETURNING *;
```

**terms.sql:**
```sql
-- name: ListTermsBySubject :many
SELECT id, subject_id, name, display_order
FROM terms
WHERE subject_id = $1
ORDER BY display_order;

-- name: CreateTerm :one
INSERT INTO terms (subject_id, name, display_order)
VALUES ($1, $2, $3)
RETURNING *;
```

**units.sql:**
```sql
-- name: ListUnitsByTerm :many
SELECT id, term_id, name, description, card_count, version, display_order
FROM units
WHERE term_id = $1
ORDER BY display_order;

-- name: ListUnitsBySubject :many
SELECT u.id, u.term_id, u.name, u.description, u.card_count, u.version, u.display_order
FROM units u
JOIN terms t ON t.id = u.term_id
WHERE t.subject_id = $1
ORDER BY t.display_order, u.display_order;

-- name: GetUnitByID :one
SELECT id, term_id, name, description, card_count, version, display_order
FROM units
WHERE id = $1;

-- name: CreateUnit :one
INSERT INTO units (term_id, name, description, display_order)
VALUES ($1, $2, $3, $4)
RETURNING *;
```

**cards.sql:**
```sql
-- name: ListCardsByUnit :many
SELECT id, unit_id, type, question, answer, choices, explanation, display_order
FROM cards
WHERE unit_id = $1
ORDER BY display_order;

-- name: GetCardByID :one
SELECT id, unit_id, type, question, answer, choices, explanation, display_order
FROM cards
WHERE id = $1;

-- name: CreateCard :one
INSERT INTO cards (unit_id, type, question, answer, choices, explanation, display_order)
VALUES ($1, $2, $3, $4, $5, $6, $7)
RETURNING *;

-- name: UpdateCard :one
UPDATE cards
SET type = $2, question = $3, answer = $4, choices = $5, explanation = $6, display_order = $7, updated_at = NOW()
WHERE id = $1
RETURNING *;

-- name: DeleteCard :exec
DELETE FROM cards WHERE id = $1;

-- name: BulkCreateCards :copyfrom
INSERT INTO cards (unit_id, type, question, answer, choices, explanation, display_order)
VALUES ($1, $2, $3, $4, $5, $6, $7);
```

Generate: `sqlc generate`

### Step 2.4 — Create models

**Request DTOs:**
- `CreateGradeRequest` — name, level, educationStage, displayOrder
- `CreateSubjectRequest` — gradeId, name, nameSi, nameTa, icon, color, isCompulsory, stream, displayOrder
- `CreateTermRequest` — subjectId, name, displayOrder
- `CreateUnitRequest` — termId, name, description, displayOrder
- `CreateCardRequest` — unitId, type, question, answer, choices, explanation, displayOrder
- `UpdateCardRequest` — type, question, answer, choices, explanation, displayOrder
- `BulkCardImportRequest` — unitId, cards []CreateCardRequest

**Response DTOs:**
- `GradeResponse`, `SubjectResponse`, `TermResponse`, `UnitResponse`, `CardResponse`
- `SubjectUnitsResponse` — terms with nested units
- `BulkImportResponse` — count of imported cards

### Step 2.5 — Create service layer

```go
type ContentService interface {
    ListGrades(ctx context.Context) ([]model.GradeResponse, error)
    ListSubjects(ctx context.Context, gradeID string) ([]model.SubjectResponse, error)
    ListUnitsBySubject(ctx context.Context, subjectID string) ([]model.SubjectUnitsResponse, error)
    ListCardsByUnit(ctx context.Context, unitID string) ([]model.CardResponse, error)

    // Admin
    CreateGrade(ctx context.Context, req model.CreateGradeRequest) (*model.GradeResponse, error)
    CreateSubject(ctx context.Context, req model.CreateSubjectRequest) (*model.SubjectResponse, error)
    CreateTerm(ctx context.Context, req model.CreateTermRequest) (*model.TermResponse, error)
    CreateUnit(ctx context.Context, req model.CreateUnitRequest) (*model.UnitResponse, error)
    CreateCard(ctx context.Context, req model.CreateCardRequest) (*model.CardResponse, error)
    UpdateCard(ctx context.Context, cardID string, req model.UpdateCardRequest) (*model.CardResponse, error)
    DeleteCard(ctx context.Context, cardID string) error
    BulkImportCards(ctx context.Context, req model.BulkCardImportRequest) (*model.BulkImportResponse, error)
}
```

### Step 2.6 — Create handler layer

```go
type ContentHandler struct {
    service service.ContentService
}
```

**Routes:**
```
GET    /grades                        → ListGrades
GET    /grades/{gradeId}/subjects     → ListSubjects
GET    /subjects/{subjectId}/units    → ListUnitsBySubject
GET    /units/{unitId}/cards          → ListCardsByUnit

POST   /admin/grades                 → CreateGrade        (admin only)
POST   /admin/subjects               → CreateSubject      (admin only)
POST   /admin/terms                  → CreateTerm         (admin only)
POST   /admin/units                  → CreateUnit         (admin only)
POST   /admin/cards                  → CreateCard         (admin only)
PUT    /admin/cards/{cardId}         → UpdateCard         (admin only)
DELETE /admin/cards/{cardId}         → DeleteCard         (admin only)
POST   /admin/cards/bulk             → BulkImportCards    (admin only)
```

### Step 2.7 — Dockerfile & main.go

Same pattern as auth-service but port 8082.

### Step 2.8 — Verification

```bash
curl http://localhost:8082/grades -H "Authorization: Bearer <token>"
curl http://localhost:8082/grades/<gradeId>/subjects -H "Authorization: Bearer <token>"
curl http://localhost:8082/subjects/<subjectId>/units -H "Authorization: Bearer <token>"
curl http://localhost:8082/units/<unitId>/cards -H "Authorization: Bearer <token>"

# Admin
curl -X POST http://localhost:8082/admin/cards -H "Authorization: Bearer <admin-token>" \
  -d '{"unitId":"...","type":"classic","question":"What is 2+2?","answer":"4"}'
```

**Acceptance criteria:**
- Content hierarchy returns correctly nested data
- Admin CRUD creates/updates/deletes cards
- Non-admin users get 403 on admin routes
- Bulk import handles multiple cards

---

## Phase 3: Review Service

### Step 3.1 — Initialize review-service

Same module init pattern. Port 8083.

### Step 3.2 — Write sqlc queries

**reviews.sql:**
```sql
-- name: CreateReview :one
INSERT INTO card_reviews (user_id, card_id, rating, reviewed_at)
VALUES ($1, $2, $3, $4)
RETURNING *;

-- name: ListReviewsByUser :many
SELECT id, user_id, card_id, rating, reviewed_at, created_at
FROM card_reviews
WHERE user_id = $1
ORDER BY reviewed_at DESC
LIMIT $2 OFFSET $3;
```

**srs_state.sql:**
```sql
-- name: GetSRSState :one
SELECT user_id, card_id, stability, difficulty, elapsed_days, scheduled_days,
       state, due_at, last_reviewed_at, reps, lapses
FROM card_srs_state
WHERE user_id = $1 AND card_id = $2;

-- name: UpsertSRSState :one
INSERT INTO card_srs_state (user_id, card_id, stability, difficulty, elapsed_days,
    scheduled_days, state, due_at, last_reviewed_at, reps, lapses)
VALUES ($1, $2, $3, $4, $5, $6, $7, $8, $9, $10, $11)
ON CONFLICT (user_id, card_id) DO UPDATE SET
    stability = EXCLUDED.stability,
    difficulty = EXCLUDED.difficulty,
    elapsed_days = EXCLUDED.elapsed_days,
    scheduled_days = EXCLUDED.scheduled_days,
    state = EXCLUDED.state,
    due_at = EXCLUDED.due_at,
    last_reviewed_at = EXCLUDED.last_reviewed_at,
    reps = EXCLUDED.reps,
    lapses = EXCLUDED.lapses
RETURNING *;

-- name: GetDueCards :many
SELECT cs.user_id, cs.card_id, cs.stability, cs.difficulty, cs.elapsed_days,
       cs.scheduled_days, cs.state, cs.due_at, cs.last_reviewed_at, cs.reps, cs.lapses,
       c.id AS card_id, c.unit_id, c.type, c.question, c.answer, c.choices, c.explanation,
       u.name AS unit_name
FROM card_srs_state cs
JOIN cards c ON c.id = cs.card_id
JOIN units u ON u.id = c.unit_id
WHERE cs.user_id = $1
  AND cs.due_at <= NOW()
ORDER BY cs.due_at ASC
LIMIT $2;

-- name: GetNewCards :many
SELECT c.id AS card_id, c.unit_id, c.type, c.question, c.answer, c.choices, c.explanation,
       u.name AS unit_name
FROM cards c
JOIN units u ON u.id = c.unit_id
WHERE c.unit_id = ANY($1::uuid[])
  AND NOT EXISTS (
    SELECT 1 FROM card_srs_state cs
    WHERE cs.user_id = $2 AND cs.card_id = c.id
  )
ORDER BY c.display_order
LIMIT $3;
```

### Step 3.3 — Create models

**Request:**
```go
type BatchReviewRequest struct {
    Reviews []ReviewItem `json:"reviews" validate:"required,min=1,max=100"`
}

type ReviewItem struct {
    CardID     string    `json:"cardId" validate:"required,uuid"`
    Rating     int16     `json:"rating" validate:"required,min=1,max=4"`
    ReviewedAt time.Time `json:"reviewedAt" validate:"required"`
}
```

**Response:**
```go
type BatchReviewResponse struct {
    Accepted    int              `json:"accepted"`
    Rejected    int              `json:"rejected"`
}

type CorrectionItem struct {
    CardID    string `json:"cardId"`
    Reason    string `json:"reason"`
}

type DueCardsResponse struct {
    DueCards []DueCardItem `json:"dueCards"`
    Total    int           `json:"total"`
}

type DueCardItem struct {
    CardID      string          `json:"cardId"`
    UnitID      string          `json:"unitId"`
    Type        string          `json:"type"`
    Question    string          `json:"question"`
    Answer      *string         `json:"answer,omitempty"`
    Choices     json.RawMessage `json:"choices,omitempty"`
    Explanation *string         `json:"explanation,omitempty"`
    UnitName    string          `json:"unitName"`
    SRSState    *SRSStateOutput `json:"srsState,omitempty"`
}
```

### Step 3.4 — Service layer

```go
type ReviewService interface {
    SubmitBatch(ctx context.Context, userID string, req BatchReviewRequest) (*BatchReviewResponse, error)
    GetDueCards(ctx context.Context, userID string, limit int) (*DueCardsResponse, error)
}
```

**SubmitBatch logic:**
1. For each review item (process in a **single database transaction**):
   a. Insert into `card_reviews` (history log)
   b. Get existing SRS state from `card_srs_state` (if any)
   c. Run FSRS (go-fsrs) to compute new scheduling:
      - If new card: use initial values (stability=0, difficulty=0, state="new")
      - If existing card: use current state from database
      - Input: current state + rating → Output: new stability, difficulty, state, due date
   d. Upsert `card_srs_state` with FSRS-computed values
   e. Update daily activity and streak
2. Return accepted/rejected counts

**GetDueCards logic:**
1. Parse optional `limit` query param (default: 50)
2. Query `GetDueCards(userID, limit)` — returns cards with `due_at <= NOW()`
3. Map to `DueCardsResponse`

### Step 3.5 — Handler & routes

```
POST /batch         → SubmitBatch     (JWT protected)
GET  /due           → GetDueCards      (JWT protected)
GET  /due?limit=20  → GetDueCards      (JWT protected, with limit param)
```

### Step 3.6 — Dockerfile & main.go

Port 8083.

### Step 3.7 — Verification

```bash
# Submit reviews
curl -X POST http://localhost:8083/batch -H "Authorization: Bearer <token>" \
  -d '{"reviews":[{"cardId":"...","rating":3,"reviewedAt":"2026-05-01T10:30:00Z"}]}'

# Get due cards
curl http://localhost:8083/due -H "Authorization: Bearer <token>"
```

**Acceptance criteria:**
- Batch review submission stores reviews and updates SRS state
- Due cards query returns cards with due_at <= now
- Conflict resolution works correctly (stale updates rejected)

---

## Phase 4: Study Plan Service

### Step 4.1 — Initialize studyplan-service

Same pattern. Port 8084.

### Step 4.2 — Write sqlc queries

**study_plans.sql:**
```sql
-- name: CreateStudyPlan :one
INSERT INTO study_plans (user_id, name, goal_type, target_term_id, deadline, daily_minutes)
VALUES ($1, $2, $3, $4, $5, $6)
RETURNING *;

-- name: GetStudyPlanByID :one
SELECT * FROM study_plans WHERE id = $1;

-- name: ListStudyPlansByUser :many
SELECT * FROM study_plans
WHERE user_id = $1
ORDER BY created_at DESC;

-- name: UpdateStudyPlanActive :one
UPDATE study_plans SET is_active = $2 WHERE id = $1 RETURNING *;
```

**study_plan_tasks.sql:**
```sql
-- name: CreateStudyPlanTask :one
INSERT INTO study_plan_tasks (plan_id, unit_id, scheduled_date, task_type, estimated_minutes)
VALUES ($1, $2, $3, $4, $5)
RETURNING *;

-- name: ListTasksByPlanAndDate :many
SELECT * FROM study_plan_tasks
WHERE plan_id = $1 AND scheduled_date = $2
ORDER BY task_type, unit_id;

-- name: GetTaskByID :one
SELECT * FROM study_plan_tasks WHERE id = $1;

-- name: CompleteTask :one
UPDATE study_plan_tasks
SET is_completed = true, completed_at = NOW()
WHERE id = $1
RETURNING *;

-- name: GetPlanProgress :one
SELECT
    COUNT(*) AS total_tasks,
    COUNT(*) FILTER (WHERE is_completed) AS completed_tasks
FROM study_plan_tasks
WHERE plan_id = $1;
```

### Step 4.3 — Create models

**Request:**
- `CreateStudyPlanRequest` — name, goalType, targetTermId, deadline, dailyMinutes
- `CompleteTaskRequest` — (no body needed, just task ID in URL)

**Response:**
- `StudyPlanResponse` — plan details with progress stats
- `StudyPlanListResponse` — array of plans
- `TaskListResponse` — array of tasks for a date
- `TaskResponse` — single task details

### Step 4.4 — Service layer

```go
type StudyPlanService interface {
    CreatePlan(ctx context.Context, userID string, req CreateStudyPlanRequest) (*StudyPlanResponse, error)
    ListPlans(ctx context.Context, userID string) ([]StudyPlanResponse, error)
    GetPlan(ctx context.Context, planID string) (*StudyPlanResponse, error)
    GetTodayTasks(ctx context.Context, planID string) (*TaskListResponse, error)
    CompleteTask(ctx context.Context, taskID string) (*TaskResponse, error)
}
```

**CreatePlan logic:**
1. Insert plan record
2. Run plan generation algorithm:
   a. Fetch all units for the selected subjects/term
   b. Assess current progress per unit (cards studied, accuracy)
   c. Compute priority scores (unseen weight 3.0, weakness 2.0, overdue 1.0)
   d. Sort by priority
   e. Distribute across days between today and deadline
   f. 60% new content, 40% review per day
3. Insert `study_plan_tasks` for each day/unit assignment

### Step 4.5 — Routes

```
POST   /plans              → CreatePlan
GET    /plans              → ListPlans
GET    /plans/{planId}     → GetPlan
GET    /plans/{planId}/today → GetTodayTasks
PATCH  /tasks/{taskId}     → CompleteTask
```

### Step 4.6 — Dockerfile, main.go, verification

Port 8084. Same patterns.

**Acceptance criteria:**
- Plan creation generates tasks distributed across days
- Today's tasks returns correct date's tasks
- Completing a task marks it done with timestamp

---

## Phase 5: Analytics Service

### Step 5.1 — Initialize analytics-service

Same pattern. Port 8085.

### Step 5.2 — Write sqlc queries

**analytics.sql:**
```sql
-- name: GetDashboardSummary :one
SELECT
    (SELECT COUNT(*) FROM card_reviews WHERE user_id = $1) AS total_reviews,
    (SELECT COUNT(*) FROM card_srs_state WHERE user_id = $1 AND state = 'review') AS cards_mastered,
    (SELECT COUNT(*) FROM card_srs_state WHERE user_id = $1) AS cards_studied,
    (SELECT current_streak FROM user_streaks WHERE user_id = $1) AS current_streak;

-- name: GetWeakAreas :many
SELECT s.id AS subject_id, s.name AS subject_name,
       u.id AS unit_id, u.name AS unit_name,
       COUNT(cs.card_id) AS total_cards,
       AVG(CASE WHEN cs.state = 'review' THEN 1.0 ELSE 0.0 END) AS mastery_ratio,
       SUM(cs.lapses) AS total_lapses,
       COUNT(CASE WHEN cs.due_at <= NOW() THEN 1 END) AS overdue_cards
FROM card_srs_state cs
JOIN cards c ON c.id = cs.card_id
JOIN units u ON u.id = c.unit_id
JOIN terms t ON t.id = u.term_id
JOIN subjects s ON s.id = t.subject_id
WHERE cs.user_id = $1
GROUP BY s.id, s.name, u.id, u.name
ORDER BY mastery_ratio ASC, total_lapses DESC
LIMIT $2;
```

**streaks.sql:**
```sql
-- name: GetUserStreak :one
SELECT current_streak, longest_streak, last_active_date
FROM user_streaks WHERE user_id = $1;

-- name: UpsertUserStreak :one
INSERT INTO user_streaks (user_id, current_streak, longest_streak, last_active_date)
VALUES ($1, $2, $3, $4)
ON CONFLICT (user_id) DO UPDATE SET
    current_streak = EXCLUDED.current_streak,
    longest_streak = EXCLUDED.longest_streak,
    last_active_date = EXCLUDED.last_active_date
RETURNING *;
```

**daily_activity.sql:**
```sql
-- name: GetDailyActivity :many
SELECT user_id, activity_date, sessions_count, cards_reviewed, correct_count, study_minutes
FROM daily_activity
WHERE user_id = $1
  AND activity_date >= CURRENT_DATE - INTERVAL '365 days'
ORDER BY activity_date ASC;

-- name: UpsertDailyActivity :one
INSERT INTO daily_activity (user_id, activity_date, sessions_count, cards_reviewed, correct_count, study_minutes)
VALUES ($1, $2, $3, $4, $5, $6)
ON CONFLICT (user_id, activity_date) DO UPDATE SET
    sessions_count = daily_activity.sessions_count + EXCLUDED.sessions_count,
    cards_reviewed = daily_activity.cards_reviewed + EXCLUDED.cards_reviewed,
    correct_count = daily_activity.correct_count + EXCLUDED.correct_count,
    study_minutes = daily_activity.study_minutes + EXCLUDED.study_minutes
RETURNING *;
```

### Step 5.3 — Create models

**Response:**
- `DashboardSummaryResponse` — totalReviews, cardsMastered, cardsStudied, currentStreak
- `StreakResponse` — currentStreak, longestStreak, heatmap []DayActivity
- `DayActivity` — date, sessionsCount, cardsReviewed, correctCount, studyMinutes
- `WeakAreaResponse` — array of weak areas with subject, unit, mastery ratio, lapses

### Step 5.4 — Service layer

```go
type AnalyticsService interface {
    GetSummary(ctx context.Context, userID string) (*DashboardSummaryResponse, error)
    GetStreak(ctx context.Context, userID string) (*StreakResponse, error)
    GetWeakAreas(ctx context.Context, userID string, limit int) ([]WeakArea, error)
    RecordActivity(ctx context.Context, userID string, activity DailyActivityInput) error
    UpdateStreak(ctx context.Context, userID string) error
}
```

**UpdateStreak logic:**
1. Get current streak from DB
2. If `last_active_date` is yesterday → increment `current_streak`
3. If `last_active_date` is today → no change
4. If `last_active_date` is older → reset `current_streak` to 1
5. Update `longest_streak` if `current > longest`

### Step 5.5 — Routes

```
GET /summary      → GetSummary     (JWT protected)
GET /streak       → GetStreak      (JWT protected)
GET /weak-areas   → GetWeakAreas   (JWT protected)
```

### Step 5.6 — Dockerfile, main.go, verification

Port 8085.

```bash
curl http://localhost:8085/summary -H "Authorization: Bearer <token>"
curl http://localhost:8085/streak -H "Authorization: Bearer <token>"
curl http://localhost:8085/weak-areas -H "Authorization: Bearer <token>"
```

**Acceptance criteria:**
- Summary returns aggregate stats
- Streak includes 365-day heatmap data
- Weak areas identifies units with low mastery

---

## Phase 6: Docker Compose & Coolify Deployment

### Step 6.1 — Production docker-compose.yml

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
```

### Step 6.2 — Makefile

```makefile
.PHONY: dev migrate seed generate build test

# Local development
dev:
	docker compose -f docker-compose.dev.yml up -d

# Database
migrate:
	migrate -path db/migrations -database $(DATABASE_URL) up

migrate-down:
	migrate -path db/migrations -database $(DATABASE_URL) down 1

seed:
	go run db/seed/seed.go

# Code generation
generate:
	cd services/auth-service && sqlc generate
	cd services/content-service && sqlc generate
	cd services/review-service && sqlc generate
	cd services/studyplan-service && sqlc generate
	cd services/analytics-service && sqlc generate

# Build all
build:
	docker compose build

# Test
test:
	cd services/auth-service && go test ./...
	cd services/content-service && go test ./...
	cd services/review-service && go test ./...
	cd services/studyplan-service && go test ./...
	cd services/analytics-service && go test ./...
```

### Step 6.3 — Coolify deployment checklist

1. **Push to GitHub** — Coolify watches the repo
2. **In Coolify:**
   - Create 5 services (one per microservice)
   - Each service points to its Dockerfile
   - Set environment variables for each service
   - Configure Traefik routing labels
3. **Environment variables to set in Coolify:**
   - `DATABASE_URL` — internal PostgreSQL connection string
   - `JWT_SECRET` — shared across all services (generate with `openssl rand -hex 32`)
   - `ENV` — `production`
4. **Database:**
   - PostgreSQL is already hosted in Coolify
   - Run migrations: `make migrate` or add migration step to Dockerfile entrypoint
5. **Health checks:** Each service exposes a `GET /health` endpoint returning `{ "status": "ok", "database": "connected" }`
6. **Verify:** After deployment, test each route through Traefik

### Step 6.4 — Final verification checklist

- [ ] All 5 services start without errors
- [ ] Auth: register, login, token refresh, profile CRUD
- [ ] Content: grades, subjects, units, cards read; admin CRUD
- [ ] Reviews: batch submit, due cards, SRS state upsert
- [ ] Study plans: create plan, list plans, today's tasks, complete task
- [ ] Analytics: summary, streak heatmap, weak areas
- [ ] JWT auth works across all services (same secret)
- [ ] Admin routes protected (403 for non-admin)
- [ ] Traefik routes correctly to each service
- [ ] Health checks pass
- [ ] Migrations run cleanly
- [ ] Seed data loads correctly

---

## Build Order Summary

| Step | What | Depends On |
|------|------|-----------|
| 0.1 | Go workspace init | — |
| 0.2 | Shared library (`pkg/common/`) | 0.1 |
| 0.3 | Database migrations | 0.1 |
| 0.4 | Seed data script | 0.3 |
| 0.5 | Dev docker-compose | 0.3 |
| 1.x | Auth service | 0.2, 0.3 |
| 2.x | Content service | 0.2, 0.3 |
| 3.x | Review service | 0.2, 0.3 |
| 4.x | Study plan service | 0.2, 0.3 |
| 5.x | Analytics service | 0.2, 0.3 |
| 6.x | Production compose + deploy | All above |

Services 1-5 can be built in parallel since they share only the common library and database.
