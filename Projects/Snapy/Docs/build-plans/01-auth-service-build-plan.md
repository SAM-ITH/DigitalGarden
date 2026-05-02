# Build Plan: Auth Service

> **Agent Instructions:** Build the Auth microservice for Snapy. Handles user registration, login, token refresh, and profile management. Runs on port **8081** behind Traefik path prefix `/api/auth`.

---

## Prerequisites

Before starting, confirm these exist (built by the shared-library agent):

- [ ] `pkg/common/` — shared library compiles (`go build ./...`)
- [ ] `db/migrations/` — migrations 000001 + 000002 (password_hash) applied
- [ ] `docker-compose.dev.yml` — PostgreSQL running on `localhost:5432`
- [ ] Database URL: `postgresql://snapy:snapydev@localhost:5432/snapy?sslmode=disable`

If any prerequisite is missing, **stop and report back** — do not attempt to build prerequisites yourself.

---

## Reference Docs

- `10-api-build-plan.md` — Phase 1 (Steps 1.1–1.10)
- `08-database-design.md` — Tables: `users`, `refresh_tokens`
- `05-backend-system.md` — Auth flow diagrams, JWT structure, error codes

---

## Directory Structure

```
services/auth-service/
├── go.mod
├── go.sum
├── cmd/
│   └── api/
│       └── main.go
├── internal/
│   ├── handler/
│   │   └── auth.go
│   ├── service/
│   │   └── auth.go
│   ├── repository/          ← sqlc generated (do NOT edit manually)
│   │   ├── db.go
│   │   ├── models.go
│   │   ├── users.sql.go
│   │   └── tokens.sql.go
│   └── model/
│       ├── request.go
│       └── response.go
├── db/
│   └── queries/
│       ├── users.sql
│       └── tokens.sql
├── sqlc.yaml
├── Dockerfile
└── .env.example
```

---

## Step 1: Initialize Module

```bash
cd services/auth-service
go mod init github.com/yourusername/snapy-api/services/auth-service
go mod edit -replace github.com/yourusername/snapy-api/pkg/common=../../pkg/common
```

**Install dependencies:**

```bash
go get github.com/go-chi/chi/v5
go get github.com/golang-jwt/jwt/v5
go get github.com/go-playground/validator/v10
go get github.com/jackc/pgx/v5
go get github.com/jackc/pgx/v5/pgxpool
go get github.com/golang-migrate/migrate/v4
go get github.com/rs/zerolog
go get golang.org/x/crypto/bcrypt
go get github.com/google/uuid
```

Run `go mod tidy`.

---

## Step 2: Configure sqlc

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

---

## Step 3: Write sqlc Queries

### `db/queries/users.sql`

```sql
-- name: GetUserByID :one
SELECT id, phone, name, grade_id, stream, role, password_hash, created_at, updated_at
FROM users
WHERE id = $1;

-- name: GetUserByPhone :one
SELECT id, phone, name, grade_id, stream, role, password_hash, created_at, updated_at
FROM users
WHERE phone = $1;

-- name: CreateUser :one
INSERT INTO users (phone, name, password_hash, role)
VALUES ($1, $2, $3, 'user')
RETURNING id, phone, name, grade_id, stream, role, password_hash, created_at, updated_at;

-- name: UpdateUserProfile :one
UPDATE users
SET name = $2, grade_id = $3, stream = $4, updated_at = NOW()
WHERE id = $1
RETURNING id, phone, name, grade_id, stream, role, password_hash, created_at, updated_at;
```

### `db/queries/tokens.sql`

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

**Generate:** `sqlc generate`

---

## Step 4: Request/Response Models

### `internal/model/request.go`

```go
package model

import "time"

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

### `internal/model/response.go`

```go
package model

type AuthResponse struct {
    AccessToken  string               `json:"accessToken"`
    RefreshToken string               `json:"refreshToken"`
    IsNewUser    bool                 `json:"isNewUser"`
    User         UserProfileResponse  `json:"user"`
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

---

## Step 5: Service Layer

### `internal/service/auth.go`

```go
package service

type AuthService interface {
    Register(ctx context.Context, req model.RegisterRequest) (*model.AuthResponse, error)
    Login(ctx context.Context, req model.LoginRequest) (*model.AuthResponse, error)
    RefreshToken(ctx context.Context, refreshToken string) (*model.AuthResponse, error)
    GetProfile(ctx context.Context, userID string) (*model.UserProfileResponse, error)
    UpdateProfile(ctx context.Context, userID string, req model.UpdateProfileRequest) (*model.UserProfileResponse, error)
}

type authService struct {
    queries       *repository.Queries
    jwtSecret     string
    accessExpiry  time.Duration
    refreshExpiry time.Duration
}

func NewAuthService(queries *repository.Queries, jwtSecret string, accessExpiry time.Duration) AuthService
```

**Constructor:** `NewAuthService` creates the service. `refreshExpiry` is hardcoded to 30 days.

#### Register Logic

1. Check if user with `req.Phone` exists via `GetUserByPhone` → return `CONFLICT` error if found
2. Hash password with `bcrypt.GenerateFromPassword([]byte(req.Password), 12)`
3. Insert user via `CreateUser(phone, name, passwordHash)`
4. Generate JWT access token using `jwt.GenerateToken(secret, userID, "", "user", accessExpiry)`
5. Generate opaque refresh token: `uuid.New().String()` + hash with `SHA-256`
6. Store refresh token hash via `CreateRefreshToken(userID, sha256Hex, time.Now().Add(30*24*time.Hour))`
7. Map user to `UserProfileResponse`
8. Return `AuthResponse{AccessToken, RefreshToken(plain), IsNewUser: true, User}`

#### Login Logic

1. Find user by phone via `GetUserByPhone` → return `UNAUTHORIZED` if not found
2. Compare password: `bcrypt.CompareHashAndPassword(storedHash, []byte(req.Password))` → return `UNAUTHORIZED` on mismatch
3. Generate access + refresh tokens (same as Register steps 4-6)
4. Return `AuthResponse{IsNewUser: false, ...}`

#### RefreshToken Logic

1. SHA-256 hash the incoming `refreshToken` string
2. Look up via `GetRefreshTokenByHash` → return `UNAUTHORIZED` if not found
3. Check `expires_at > now()` → return `TOKEN_EXPIRED` if expired
4. Delete old token via `DeleteRefreshToken` (rotation)
5. Generate new access + refresh token pair
6. Store new refresh token hash
7. Return `AuthResponse{IsNewUser: false, ...}`

#### GetProfile Logic

1. `GetUserByID(userID)` from database
2. Map to `UserProfileResponse` (UUIDs as strings, timestamps as ISO 8601)

#### UpdateProfile Logic

1. Validate that at least one field is non-nil in `req`
2. Get current user via `GetUserByID(userID)`
3. Build update params: use req fields if non-nil, else keep current values
4. Call `UpdateUserProfile(userID, name, gradeID, stream)`
5. Map result to `UserProfileResponse`

---

## Step 6: Handler Layer

### `internal/handler/auth.go`

```go
package handler

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

**Each handler pattern:**

1. Decode JSON body into request model (for POST/PATCH)
2. Validate with `validation.ValidateStruct(req)` → return `VALIDATION_ERROR` on failure
3. Extract `userID` from context (for protected routes): `middleware.GetClaimsFromContext(r.Context())`
4. Call service method
5. On success: `response.JSON(w, status, data)`
6. On error: map service error to HTTP status + error code

**Error mapping in handlers:**

| Service Error | HTTP Status | Error Code |
|---|---|---|
| User not found | 404 | `NOT_FOUND` |
| Invalid credentials | 401 | `UNAUTHORIZED` |
| Phone already exists | 409 | `CONFLICT` |
| Token expired/invalid | 401 | `TOKEN_EXPIRED` |
| Validation failure | 400 | `VALIDATION_ERROR` |

---

## Step 7: Main Entry Point

### `cmd/api/main.go`

```go
func main() {
    // 1. Load config (pkg/common/config)
    // 2. Initialize zerolog logger
    // 3. Connect to PostgreSQL (pkg/common/database)
    // 4. Run migrations (golang-migrate, path "../../db/migrations" relative to working dir)
    // 5. Create sqlc repository: repository.New(pool)
    // 6. Create service: service.NewAuthService(queries, config.JWTSecret, accessExpiry)
    // 7. Create handler: handler.NewAuthHandler(svc)
    // 8. Create chi router with middleware chain:
    //    - middleware.RequestID
    //    - pkg/common/middleware.LoggerMiddleware
    //    - pkg/common/middleware.RecoveryMiddleware
    //    - pkg/common/middleware.CORSMiddleware
    // 9. Register routes:
    //    POST   /register       → handler.Register       (public)
    //    POST   /login          → handler.Login           (public)
    //    POST   /token/refresh  → handler.RefreshToken    (public)
    //    GET    /me             → handler.GetProfile      (JWT protected)
    //    PATCH  /me             → handler.UpdateProfile   (JWT protected)
    //    GET    /health         → health check            (public)
    // 10. Start HTTP server on config.Port (default 8081)
    // 11. Graceful shutdown on SIGINT/SIGTERM
}
```

**Protected routes** use `r.Group()` with `middleware.JWTAuthMiddleware(config.JWTSecret)`.

**Health check** returns `{ "status": "ok", "database": "connected" }` and pings DB.

---

## Step 8: Dockerfile

### `Dockerfile`

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

**Important:** Build context for Docker is the repo root (`snapy-api/`), not the service directory. The Dockerfile copies the service directory contents. Adjust COPY paths if building from repo root.

---

## Step 9: .env.example

```
PORT=8081
DATABASE_URL=postgresql://snapy:snapydev@localhost:5432/snapy?sslmode=disable
JWT_SECRET=change-me-to-a-64-char-random-string
JWT_ACCESS_EXPIRY=15m
ENV=development
```

---

## Step 10: Verification

```bash
cd services/auth-service

# Compile check
go mod tidy
go build ./...
sqlc generate

# Start dev database
docker compose -f ../../docker-compose.dev.yml up -d

# Run migrations
migrate -path ../../db/migrations -database "postgresql://snapy:snapydev@localhost:5432/snapy?sslmode=disable" up

# Run service
go run cmd/api/main.go

# Test endpoints
curl -s -X POST http://localhost:8081/register \
  -H "Content-Type: application/json" \
  -d '{"phone":"+94771234567","name":"Test User","password":"password123"}'
# Expected: { "accessToken": "eyJ...", "refreshToken": "...", "isNewUser": true, "user": {...} }

curl -s -X POST http://localhost:8081/login \
  -H "Content-Type: application/json" \
  -d '{"phone":"+94771234567","password":"password123"}'
# Expected: { "accessToken": "eyJ...", "refreshToken": "...", "isNewUser": false, "user": {...} }

curl -s -X POST http://localhost:8081/token/refresh \
  -H "Content-Type: application/json" \
  -d '{"refreshToken":"<token-from-above>"}'
# Expected: New token pair, old refresh token invalidated

curl -s http://localhost:8081/me \
  -H "Authorization: Bearer <access-token>"
# Expected: { "id": "...", "phone": "+94771234567", "name": "Test User", ... }

curl -s -X PATCH http://localhost:8081/me \
  -H "Authorization: Bearer <access-token>" \
  -H "Content-Type: application/json" \
  -d '{"name":"Updated Name"}'
# Expected: Updated profile

# Verify protected route rejects without token
curl -s http://localhost:8081/me
# Expected: 401 UNAUTHORIZED

# Verify duplicate registration fails
curl -s -X POST http://localhost:8081/register \
  -H "Content-Type: application/json" \
  -d '{"phone":"+94771234567","name":"Test","password":"password123"}'
# Expected: 409 CONFLICT

# Health check
curl -s http://localhost:8081/health
# Expected: { "status": "ok", "database": "connected" }
```

---

## Acceptance Criteria

- [ ] Service compiles: `go build ./...` passes
- [ ] sqlc generates: `sqlc generate` produces repository code
- [ ] Service starts on port 8081 without errors
- [ ] Register creates user and returns JWT + refresh token
- [ ] Register with duplicate phone returns 409
- [ ] Login returns JWT for existing user with correct password
- [ ] Login with wrong password returns 401
- [ ] Login with non-existent phone returns 401
- [ ] Refresh token returns new token pair (rotation)
- [ ] Old refresh token is invalidated after rotation
- [ ] Expired refresh token returns 401
- [ ] GET /me returns user profile with valid JWT
- [ ] PATCH /me updates name/grade/stream
- [ ] Protected endpoints return 401 without JWT
- [ ] Invalid JWT returns 401
- [ ] Health check returns 200 with DB status
- [ ] Dockerfile builds successfully
- [ ] Password is stored as bcrypt hash (not plaintext)
