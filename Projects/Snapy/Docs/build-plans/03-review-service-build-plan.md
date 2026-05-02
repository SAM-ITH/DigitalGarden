# Build Plan: Review Service

> **Agent Instructions:** Build the Review microservice for Snapy. Handles batch review submission with SRS state management, due card queries, and server-side FSRS validation. Runs on port **8083** behind Traefik path prefix `/api/reviews`.

---

## Prerequisites

Before starting, confirm these exist:

- [ ] `pkg/common/` — shared library compiles
- [ ] `db/migrations/` — all migrations applied (including `card_reviews`, `card_srs_state`, `cards` tables)
- [ ] `docker-compose.dev.yml` — PostgreSQL running on `localhost:5432`

If any prerequisite is missing, **stop and report back**.

---

## Reference Docs

- `10-api-build-plan.md` — Phase 3 (Steps 3.1–3.7)
- `08-database-design.md` — Tables: `card_reviews`, `card_srs_state`, `cards`, `units`
- `09-spaced-repetition-engine.md` — FSRS algorithm, card states, rating system, server-side validation
- `05-backend-system.md` — Review batch sync flow, SRS conflict resolution, FSRS validation

---

## Directory Structure

```
services/review-service/
├── go.mod
├── go.sum
├── cmd/
│   └── api/
│       └── main.go
├── internal/
│   ├── handler/
│   │   └── review.go
│   ├── service/
│   │   ├── review.go
│   │   └── fsrs.go
│   ├── repository/          ← sqlc generated
│   │   ├── db.go
│   │   ├── models.go
│   │   ├── reviews.sql.go
│   │   └── srs_state.sql.go
│   └── model/
│       ├── request.go
│       └── response.go
├── db/
│   └── queries/
│       ├── reviews.sql
│       └── srs_state.sql
├── sqlc.yaml
├── Dockerfile
└── .env.example
```

---

## Step 1: Initialize Module

```bash
cd services/review-service
go mod init github.com/yourusername/snapy-api/services/review-service
go mod edit -replace github.com/yourusername/snapy-api/pkg/common=../../pkg/common
```

**Install dependencies:**

```bash
go get github.com/go-chi/chi/v5
go get github.com/go-playground/validator/v10
go get github.com/jackc/pgx/v5
go get github.com/jackc/pgx/v5/pgxpool
go get github.com/golang-migrate/migrate/v4
go get github.com/rs/zerolog
go get github.com/google/uuid
go get github.com/open-spaced-repetition/go-fsrs
```

Run `go mod tidy`.

---

## Step 2: Configure sqlc

Same pattern as other services. Create `services/review-service/sqlc.yaml`:

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
          - db_type: "jsonb"
            go_type: "encoding/json.RawMessage"
```

---

## Step 3: Write sqlc Queries

### `db/queries/reviews.sql`

```sql
-- name: CreateReview :one
INSERT INTO card_reviews (user_id, card_id, rating, reviewed_at)
VALUES ($1, $2, $3, $4)
RETURNING id, user_id, card_id, rating, reviewed_at, created_at;

-- name: ListReviewsByUser :many
SELECT id, user_id, card_id, rating, reviewed_at, created_at
FROM card_reviews
WHERE user_id = $1
ORDER BY reviewed_at DESC
LIMIT $2 OFFSET $3;

-- name: GetReviewCountByUser :one
SELECT COUNT(*) FROM card_reviews WHERE user_id = $1;
```

### `db/queries/srs_state.sql`

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
RETURNING user_id, card_id, stability, difficulty, elapsed_days, scheduled_days,
          state, due_at, last_reviewed_at, reps, lapses;

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

-- name: DeleteSRSState :exec
DELETE FROM card_srs_state WHERE user_id = $1 AND card_id = $2;
```

**Generate:** `sqlc generate`

---

## Step 4: Request/Response Models

### `internal/model/request.go`

```go
package model

type BatchReviewRequest struct {
    Reviews []ReviewItem `json:"reviews" validate:"required,min=1,max=100"`
}

type ReviewItem struct {
    CardID     string        `json:"cardId" validate:"required,uuid"`
    Rating     int16         `json:"rating" validate:"required,min=1,max=4"`
    ReviewedAt time.Time     `json:"reviewedAt" validate:"required"`
    SRSState   *SRSStateInput `json:"srsState"`
}

type SRSStateInput struct {
    Stability     float32 `json:"stability"`
    Difficulty    float32 `json:"difficulty"`
    ElapsedDays   int32   `json:"elapsedDays"`
    ScheduledDays int32   `json:"scheduledDays"`
    State         string  `json:"state"`
    DueAt         time.Time `json:"dueAt"`
}
```

### `internal/model/response.go`

```go
package model

type BatchReviewResponse struct {
    Accepted    int              `json:"accepted"`
    Rejected    int              `json:"rejected"`
    Corrections []CorrectionItem `json:"corrections,omitempty"`
}

type CorrectionItem struct {
    CardID string `json:"cardId"`
    Reason string `json:"reason"`
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

type SRSStateOutput struct {
    Stability     float32   `json:"stability"`
    Difficulty    float32   `json:"difficulty"`
    ElapsedDays   int32     `json:"elapsedDays"`
    ScheduledDays int32     `json:"scheduledDays"`
    State         string    `json:"state"`
    DueAt         time.Time `json:"dueAt"`
    Reps          int32     `json:"reps"`
    Lapses        int32     `json:"lapses"`
}
```

---

## Step 5: FSRS Service (Server-Side Validation)

### `internal/service/fsrs.go`

```go
package service

import (
    gofsrs "github.com/open-spaced-repetition/go-fsrs"
)

type FSRSService struct {
    scheduler gofsrs.FSRS
}

func NewFSRSService() *FSRSService
```

**Purpose:** Server-side FSRS recomputation to validate client-submitted SRS state.

**ValidateSRSState method:**

1. Reconstruct a `gofsrs.Card` from the client's current SRS state (before this review)
2. Call `scheduler.Repeat(card, now)` to get all rating outcomes
3. Select the result matching the client's rating
4. Compare server-computed values with client-submitted values:
   - `stability` — tolerance ±0.5
   - `difficulty` — tolerance ±0.5
   - `scheduledDays` — must match exactly
   - `state` — must match string
5. If within tolerance: accept client state as-is (client is correct)
6. If outside tolerance: use server-computed state instead (log warning)

This catches mobile app bugs or tampered data without rejecting legitimate reviews.

**FSRS default parameters (from `09-spaced-repetition-engine.md`):**
```
w = [0.40, 0.60, 2.40, 5.80, 4.93, 0.94, 0.86, 0.01, 1.49, 0.14,
     0.94, 2.18, 0.05, 0.34, 1.26, 0.29, 2.61, 0.00, 0.00]
desiredRetention = 0.9
```

---

## Step 6: Review Service Layer

### `internal/service/review.go`

```go
package service

type ReviewService interface {
    SubmitBatch(ctx context.Context, userID string, req model.BatchReviewRequest) (*model.BatchReviewResponse, error)
    GetDueCards(ctx context.Context, userID string, limit int) (*model.DueCardsResponse, error)
}

type reviewService struct {
    queries     *repository.Queries
    fsrsService *FSRSService
}

func NewReviewService(queries *repository.Queries, fsrsService *FSRSService) ReviewService
```

### SubmitBatch Logic

For each `ReviewItem` in the batch (process in a **single database transaction**):

1. **Insert review log:**
   - `CreateReview(userID, cardID, rating, reviewedAt)`

2. **Handle SRS state:**
   - **If `SRSState` is provided** (client computed FSRS):
     a. Get current server state: `GetSRSState(userID, cardID)`
     b. **Conflict resolution:**
        - If server `last_reviewed_at` is nil OR `server.last_reviewed_at < client.reviewedAt`: **accept client state**
        - If `server.last_reviewed_at >= client.reviewedAt`: **reject** (stale), add to corrections list, skip upsert
     c. **Server-side validation** (if accepting):
        - Run `FSRSService.ValidateSRSState()` to recompute
        - If server disagrees significantly, use server-computed values
     d. `UpsertSRSState(userID, cardID, ...)` with accepted values
   - **If `SRSState` is nil** (new card, first review):
     a. Compute initial SRS state using FSRS for the given rating
     b. For rating 1 (Again): state="learning", stability=w[0], scheduled_days=1
     c. For rating 2 (Hard): state="learning", stability=w[1], scheduled_days=1
     d. For rating 3 (Good): state="review", stability=w[2], scheduled_days from interval calc
     e. For rating 4 (Easy): state="review", stability=w[3], scheduled_days from interval calc
     f. `UpsertSRSState(userID, cardID, initial_state...)`

3. **Return response:**
   - `accepted` = count of accepted reviews
   - `rejected` = count of rejected (stale) reviews
   - `corrections` = list of rejected card IDs with reasons

**Important:** Use `pgx` transaction (`pool.Begin(ctx)`) to ensure atomicity of the batch.

### GetDueCards Logic

1. Parse optional `limit` query param (default: 50, max: 200)
2. Query `GetDueCards(userID, limit)` — returns cards with `due_at <= NOW()`
3. Map to `DueCardsResponse` with `DueCardItem` slice
4. For each due card, include the current SRS state
5. Return response with `total` count

---

## Step 7: Handler Layer

### `internal/handler/review.go`

```go
type ReviewHandler struct {
    service service.ReviewService
}

func NewReviewHandler(service service.ReviewService) *ReviewHandler

func (h *ReviewHandler) SubmitBatch(w http.ResponseWriter, r *http.Request)
func (h *ReviewHandler) GetDueCards(w http.ResponseWriter, r *http.Request)
```

**SubmitBatch handler:**
1. Decode `BatchReviewRequest` from body
2. Validate struct
3. Get `userID` from JWT claims in context
4. Call `service.SubmitBatch(ctx, userID, req)`
5. Return 200 with `BatchReviewResponse`

**GetDueCards handler:**
1. Get `userID` from JWT claims
2. Parse `limit` from query string: `r.URL.Query().Get("limit")` (default 50)
3. Call `service.GetDueCards(ctx, userID, limit)`
4. Return 200 with `DueCardsResponse`

---

## Step 8: Main Entry Point

### `cmd/api/main.go`

```go
func main() {
    // 1. Load config
    // 2. Connect to PostgreSQL
    // 3. Run migrations
    // 4. Create repository
    // 5. Create FSRS service
    // 6. Create review service
    // 7. Create handler
    // 8. Create chi router with middleware:
    //    - RequestID, Logger, Recovery, CORS
    // 9. All routes are JWT protected:
    //    POST /batch         → handler.SubmitBatch   (JWT)
    //    GET  /due           → handler.GetDueCards    (JWT)
    //    GET  /due?limit=20  → handler.GetDueCards    (JWT, with limit)
    //    GET  /health        → health check           (public)
    // 10. Start server on port 8083
}
```

---

## Step 9: Dockerfile

```dockerfile
FROM golang:1.22-alpine AS builder
WORKDIR /app
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 GOOS=linux go build -ldflags="-s -w" -o /review-service ./cmd/api

FROM scratch
COPY --from=builder /review-service /review-service
COPY --from=builder /etc/ssl/certs/ca-certificates.crt /etc/ssl/certs/
EXPOSE 8083
ENTRYPOINT ["/review-service"]
```

---

## Step 10: .env.example

```
PORT=8083
DATABASE_URL=postgresql://snapy:snapydev@localhost:5432/snapy?sslmode=disable
JWT_SECRET=change-me-to-a-64-char-random-string
JWT_ACCESS_EXPIRY=15m
ENV=development
```

---

## Step 11: Verification

```bash
cd services/review-service
go mod tidy
go build ./...
sqlc generate

# Start service
go run cmd/api/main.go

TOKEN="<jwt-from-auth-service>"

# Submit batch reviews
curl -s -X POST http://localhost:8083/batch \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "reviews": [
      {
        "cardId": "<card-uuid>",
        "rating": 3,
        "reviewedAt": "2026-05-01T10:30:00Z",
        "srsState": {
          "stability": 2.40,
          "difficulty": 5.0,
          "elapsedDays": 0,
          "scheduledDays": 2,
          "state": "learning",
          "dueAt": "2026-05-03T10:30:00Z"
        }
      }
    ]
  }'
# Expected: { "accepted": 1, "rejected": 0, "corrections": [] }

# Submit without SRS state (new card)
curl -s -X POST http://localhost:8083/batch \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "reviews": [
      {
        "cardId": "<another-card-uuid>",
        "rating": 3,
        "reviewedAt": "2026-05-01T10:30:00Z"
      }
    ]
  }'
# Expected: Initial SRS state created automatically

# Get due cards
curl -s http://localhost:8083/due \
  -H "Authorization: Bearer $TOKEN"
# Expected: { "dueCards": [...], "total": N }

# Get due cards with limit
curl -s "http://localhost:8083/due?limit=10" \
  -H "Authorization: Bearer $TOKEN"
# Expected: Max 10 due cards
```

---

## Acceptance Criteria

- [ ] Service compiles and starts on port 8083
- [ ] `POST /batch` stores reviews and updates SRS state
- [ ] Batch processing is atomic (all succeed or all fail)
- [ ] SRS state upsert works (INSERT for new, UPDATE for existing)
- [ ] Conflict resolution works: stale client state is rejected
- [ ] Corrections list returned for rejected reviews
- [ ] New cards (no SRS state) get initial FSRS state computed
- [ ] Server-side FSRS validation catches incorrect client computations
- [ ] `GET /due` returns cards with `due_at <= NOW()`
- [ ] `GET /due?limit=N` respects the limit parameter
- [ ] Due cards are sorted by `due_at` ASC (most overdue first)
- [ ] Response includes card content (question, answer, choices) + SRS state
- [ ] All routes require JWT authentication
- [ ] Health check returns 200
- [ ] Dockerfile builds successfully
- [ ] Batch size capped at 100 reviews
