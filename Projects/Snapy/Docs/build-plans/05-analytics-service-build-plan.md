# Build Plan: Analytics Service

> **Agent Instructions:** Build the Analytics microservice for Snapy. Provides dashboard summary stats, streak tracking with 365-day heatmap data, and weak area identification. Runs on port **8085** behind Traefik path prefix `/api/analytics`.

---

## Prerequisites

Before starting, confirm these exist:

- [ ] `pkg/common/` — shared library compiles
- [ ] `db/migrations/` — all migrations applied (including `user_streaks`, `daily_activity`, `card_srs_state`, `card_reviews`)
- [ ] `docker-compose.dev.yml` — PostgreSQL running on `localhost:5432`

If any prerequisite is missing, **stop and report back**.

---

## Reference Docs

- `10-api-build-plan.md` — Phase 5 (Steps 5.1–5.6)
- `08-database-design.md` — Tables: `user_streaks`, `daily_activity`, `card_srs_state`, `card_reviews`
- `05-backend-system.md` — Analytics endpoints, streak logic, weak area queries
- `09-spaced-repetition-engine.md` — SRS states (new, learning, review, relearning)

---

## Directory Structure

```
services/analytics-service/
├── go.mod
├── go.sum
├── cmd/
│   └── api/
│       └── main.go
├── internal/
│   ├── handler/
│   │   └── analytics.go
│   ├── service/
│   │   └── analytics.go
│   ├── repository/          ← sqlc generated
│   │   ├── db.go
│   │   ├── models.go
│   │   ├── analytics.sql.go
│   │   ├── streaks.sql.go
│   │   └── daily_activity.sql.go
│   └── model/
│       └── response.go
├── db/
│   └── queries/
│       ├── analytics.sql
│       ├── streaks.sql
│       └── daily_activity.sql
├── sqlc.yaml
├── Dockerfile
└── .env.example
```

---

## Step 1: Initialize Module

```bash
cd services/analytics-service
go mod init github.com/yourusername/snapy-api/services/analytics-service
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
```

Run `go mod tidy`.

---

## Step 2: Configure sqlc

Create `services/analytics-service/sqlc.yaml`:

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
          - db_type: "date"
            go_type: "time.Time"
```

---

## Step 3: Write sqlc Queries

### `db/queries/analytics.sql`

```sql
-- name: GetDashboardSummary :one
SELECT
    (SELECT COUNT(*) FROM card_reviews WHERE user_id = $1) AS total_reviews,
    (SELECT COUNT(*) FROM card_srs_state WHERE user_id = $1 AND state = 'review') AS cards_mastered,
    (SELECT COUNT(*) FROM card_srs_state WHERE user_id = $1) AS cards_studied,
    (SELECT COALESCE(current_streak, 0) FROM user_streaks WHERE user_id = $1) AS current_streak;

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

-- name: GetSubjectProgress :many
SELECT s.id AS subject_id, s.name AS subject_name,
       COUNT(DISTINCT u.id) AS total_units,
       COUNT(DISTINCT CASE WHEN cs.state = 'review' THEN u.id END) AS mastered_units,
       COUNT(DISTINCT cs.card_id) AS cards_studied,
       COUNT(c.id) AS total_cards
FROM subjects s
JOIN terms t ON t.subject_id = s.id
JOIN units u ON u.term_id = t.id
LEFT JOIN cards c ON c.unit_id = u.id
LEFT JOIN card_srs_state cs ON cs.card_id = c.id AND cs.user_id = $1
WHERE s.grade_id = $2
GROUP BY s.id, s.name
ORDER BY s.display_order;

-- name: GetWeeklyReviewCount :many
SELECT DATE_TRUNC('week', reviewed_at) AS week_start,
       COUNT(*) AS review_count
FROM card_reviews
WHERE user_id = $1
  AND reviewed_at >= NOW() - INTERVAL '12 weeks'
GROUP BY DATE_TRUNC('week', reviewed_at)
ORDER BY week_start DESC;
```

### `db/queries/streaks.sql`

```sql
-- name: GetUserStreak :one
SELECT user_id, current_streak, longest_streak, last_active_date
FROM user_streaks
WHERE user_id = $1;

-- name: UpsertUserStreak :one
INSERT INTO user_streaks (user_id, current_streak, longest_streak, last_active_date)
VALUES ($1, $2, $3, $4)
ON CONFLICT (user_id) DO UPDATE SET
    current_streak = EXCLUDED.current_streak,
    longest_streak = EXCLUDED.longest_streak,
    last_active_date = EXCLUDED.last_active_date
RETURNING user_id, current_streak, longest_streak, last_active_date;

-- name: InitializeStreak :one
INSERT INTO user_streaks (user_id, current_streak, longest_streak, last_active_date)
VALUES ($1, 1, 1, CURRENT_DATE)
ON CONFLICT (user_id) DO UPDATE SET
    current_streak = 1,
    longest_streak = GREATEST(user_streaks.longest_streak, 1),
    last_active_date = CURRENT_DATE
RETURNING user_id, current_streak, longest_streak, last_active_date;
```

### `db/queries/daily_activity.sql`

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
RETURNING user_id, activity_date, sessions_count, cards_reviewed, correct_count, study_minutes;

-- name: GetTodayActivity :one
SELECT user_id, activity_date, sessions_count, cards_reviewed, correct_count, study_minutes
FROM daily_activity
WHERE user_id = $1 AND activity_date = CURRENT_DATE;
```

**Generate:** `sqlc generate`

---

## Step 4: Response Models

### `internal/model/response.go`

```go
package model

type DashboardSummaryResponse struct {
    TotalReviews int `json:"totalReviews"`
    CardsMastered int `json:"cardsMastered"`
    CardsStudied  int `json:"cardsStudied"`
    CurrentStreak int `json:"currentStreak"`
}

type StreakResponse struct {
    CurrentStreak int           `json:"currentStreak"`
    LongestStreak int           `json:"longestStreak"`
    Heatmap       []DayActivity `json:"heatmap"`
}

type DayActivity struct {
    Date           string `json:"date"`
    SessionsCount  int    `json:"sessionsCount"`
    CardsReviewed  int    `json:"cardsReviewed"`
    CorrectCount   int    `json:"correctCount"`
    StudyMinutes   int    `json:"studyMinutes"`
}

type WeakAreaResponse struct {
    WeakAreas []WeakArea `json:"weakAreas"`
}

type WeakArea struct {
    SubjectID    string  `json:"subjectId"`
    SubjectName  string  `json:"subjectName"`
    UnitID       string  `json:"unitId"`
    UnitName     string  `json:"unitName"`
    TotalCards   int     `json:"totalCards"`
    MasteryRatio float64 `json:"masteryRatio"`
    TotalLapses  int     `json:"totalLapses"`
    OverdueCards int     `json:"overdueCards"`
}

type RecordActivityRequest struct {
    SessionsCount int `json:"sessionsCount" validate:"min=0"`
    CardsReviewed int `json:"cardsReviewed" validate:"min=0"`
    CorrectCount  int `json:"correctCount" validate:"min=0"`
    StudyMinutes  int `json:"studyMinutes" validate:"min=0"`
}
```

---

## Step 5: Service Layer

### `internal/service/analytics.go`

```go
package service

type AnalyticsService interface {
    GetSummary(ctx context.Context, userID string) (*model.DashboardSummaryResponse, error)
    GetStreak(ctx context.Context, userID string) (*model.StreakResponse, error)
    GetWeakAreas(ctx context.Context, userID string, limit int) (*model.WeakAreaResponse, error)
    RecordActivity(ctx context.Context, userID string, req model.RecordActivityRequest) error
    UpdateStreak(ctx context.Context, userID string) error
}

type analyticsService struct {
    queries *repository.Queries
    pool    *pgxpool.Pool
}

func NewAnalyticsService(queries *repository.Queries, pool *pgxpool.Pool) AnalyticsService
```

### GetSummary Logic

1. Call `GetDashboardSummary(userID)`
2. Map to `DashboardSummaryResponse`
3. `current_streak` defaults to 0 if user has no streak record
4. All counts are integers

### GetStreak Logic

1. Call `GetUserStreak(userID)` — may return "not found" for new users
2. If no streak record: return `currentStreak: 0`, `longestStreak: 0`, empty heatmap
3. Call `GetDailyActivity(userID)` — returns 365 days of activity data
4. Build heatmap array from activity records:
   - Each `DayActivity` has date as `"YYYY-MM-DD"` string
   - Only include days with actual activity (sparse array)
5. Map to `StreakResponse`

### GetWeakAreas Logic

1. Parse optional `limit` (default: 10, max: 50)
2. Call `GetWeakAreas(userID, limit)`
3. Map to `WeakAreaResponse` with array of `WeakArea` items
4. Each weak area has subject + unit info, mastery ratio, lapse count, overdue count
5. Sorted by `mastery_ratio ASC` (lowest mastery first), then `total_lapses DESC`

### RecordActivity Logic

1. Call `UpsertDailyActivity(userID, today, sessionsCount, cardsReviewed, correctCount, studyMinutes)`
2. The UPSERT increments existing values (SQL handles the accumulation)
3. After recording activity, call `UpdateStreak(userID)`

### UpdateStreak Logic

This is the streak calculation engine:

1. Get current streak: `GetUserStreak(userID)`
2. **If no streak record exists** (new user):
   - `InitializeStreak(userID)` → sets `current_streak=1`, `longest_streak=1`, `last_active_date=today`
   - Return
3. **If `last_active_date` is today:**
   - No change needed, streak already updated today
   - Return
4. **If `last_active_date` is yesterday:**
   - Increment: `current_streak + 1`
   - Update `longest_streak = max(longest_streak, current_streak + 1)`
   - `UpsertUserStreak(userID, new_current, new_longest, today)`
5. **If `last_active_date` is older than yesterday (gap):**
   - Reset: `current_streak = 1`
   - Keep `longest_streak` as-is (don't reduce it)
   - `UpsertUserStreak(userID, 1, longest_streak, today)`

**Date comparison:** Use `CURRENT_DATE` in PostgreSQL or `time.Now().Truncate(24*time.Hour)` in Go. Be careful with timezone — use the same timezone consistently.

---

## Step 6: Handler Layer

### `internal/handler/analytics.go`

```go
type AnalyticsHandler struct {
    service service.AnalyticsService
}

func NewAnalyticsHandler(service service.AnalyticsService) *AnalyticsHandler

func (h *AnalyticsHandler) GetSummary(w http.ResponseWriter, r *http.Request)
func (h *AnalyticsHandler) GetStreak(w http.ResponseWriter, r *http.Request)
func (h *AnalyticsHandler) GetWeakAreas(w http.ResponseWriter, r *http.Request)
func (h *AnalyticsHandler) RecordActivity(w http.ResponseWriter, r *http.Request)
```

**GetWeakAreas:** Parse `limit` from query string: `r.URL.Query().Get("limit")` → default 10, max 50.

**RecordActivity:**
- POST endpoint for mobile app to record study session data
- Decodes `RecordActivityRequest`
- Calls `RecordActivity` + implicitly updates streak
- Returns 204 No Content on success

---

## Step 7: Main Entry Point

### `cmd/api/main.go`

```go
func main() {
    // 1. Load config
    // 2. Connect to PostgreSQL
    // 3. Run migrations
    // 4. Create repository + pool
    // 5. Create analytics service
    // 6. Create handler
    // 7. Create chi router with middleware
    // 8. Routes:
    //    GET  /summary       → GetSummary      (JWT)
    //    GET  /streak        → GetStreak        (JWT)
    //    GET  /weak-areas    → GetWeakAreas     (JWT)
    //    GET  /weak-areas?limit=20 → GetWeakAreas (JWT, with limit)
    //    POST /activity      → RecordActivity   (JWT)
    //    GET  /health        → health check      (public)
    // 9. Start server on port 8085
}
```

---

## Step 8: Dockerfile

```dockerfile
FROM golang:1.22-alpine AS builder
WORKDIR /app
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 GOOS=linux go build -ldflags="-s -w" -o /analytics-service ./cmd/api

FROM scratch
COPY --from=builder /analytics-service /analytics-service
COPY --from=builder /etc/ssl/certs/ca-certificates.crt /etc/ssl/certs/
EXPOSE 8085
ENTRYPOINT ["/analytics-service"]
```

---

## Step 9: .env.example

```
PORT=8085
DATABASE_URL=postgresql://snapy:snapydev@localhost:5432/snapy?sslmode=disable
JWT_SECRET=change-me-to-a-64-char-random-string
JWT_ACCESS_EXPIRY=15m
ENV=development
```

---

## Step 10: Verification

```bash
cd services/analytics-service
go mod tidy
go build ./...
sqlc generate

# Start service
go run cmd/api/main.go

TOKEN="<jwt-from-auth-service>"

# Get dashboard summary
curl -s http://localhost:8085/summary \
  -H "Authorization: Bearer $TOKEN"
# Expected: { "totalReviews": N, "cardsMastered": N, "cardsStudied": N, "currentStreak": N }

# Get streak + heatmap
curl -s http://localhost:8085/streak \
  -H "Authorization: Bearer $TOKEN"
# Expected: { "currentStreak": N, "longestStreak": N, "heatmap": [...] }

# Get weak areas
curl -s http://localhost:8085/weak-areas \
  -H "Authorization: Bearer $TOKEN"
# Expected: { "weakAreas": [{ "subjectName": "...", "unitName": "...", "masteryRatio": 0.3, ... }] }

# Get weak areas with limit
curl -s "http://localhost:8085/weak-areas?limit=5" \
  -H "Authorization: Bearer $TOKEN"
# Expected: Max 5 weak areas

# Record activity
curl -s -X POST http://localhost:8085/activity \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"sessionsCount":1,"cardsReviewed":20,"correctCount":16,"studyMinutes":15}'
# Expected: 204 No Content

# Verify streak updated after activity
curl -s http://localhost:8085/streak \
  -H "Authorization: Bearer $TOKEN"
# Expected: currentStreak incremented
```

---

## Acceptance Criteria

- [ ] Service compiles and starts on port 8085
- [ ] `GET /summary` returns total reviews, cards mastered, cards studied, current streak
- [ ] Summary handles new users gracefully (all zeros)
- [ ] `GET /streak` returns current and longest streak
- [ ] `GET /streak` includes 365-day heatmap with activity data
- [ ] Heatmap only includes days with actual activity (sparse)
- [ ] `GET /weak-areas` identifies units with low mastery ratio
- [ ] Weak areas sorted by mastery ratio ASC, then lapses DESC
- [ ] `GET /weak-areas?limit=N` respects limit parameter
- [ ] `POST /activity` records daily activity (UPSERT increments)
- [ ] Recording activity updates the streak correctly
- [ ] Streak logic: consecutive days increment, gap resets to 1
- [ ] Streak logic: same-day duplicate activity doesn't double-increment
- [ ] Longest streak is never reduced
- [ ] All routes require JWT authentication
- [ ] Health check returns 200
- [ ] Dockerfile builds successfully
