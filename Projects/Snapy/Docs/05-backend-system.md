# Snapy - Backend System: Detailed Design

## Overview

The Snapy backend is a REST API built with **Go**, using the **Chi** router for HTTP handling and **sqlc** for type-safe database access. It handles authentication, content serving, progress tracking, study plan generation, analytics, and leaderboard computation. It connects to **PostgreSQL** for persistent storage and **Redis** for caching and real-time data structures.

The API compiles to a single static binary (~15MB), runs in a minimal Docker container, and uses ~20-30MB of RAM — leaving maximum headroom on the VPS for PostgreSQL and Redis.

---

## API Route Table

### Authentication
| Method | Route | Auth | Description |
|--------|-------|------|-------------|
| `POST` | `/auth/otp/request` | Public | Send OTP to phone number |
| `POST` | `/auth/otp/verify` | Public | Verify OTP, return JWT tokens |
| `POST` | `/auth/token/refresh` | Public | Refresh access token |

### User Profile
| Method | Route | Auth | Description |
|--------|-------|------|-------------|
| `GET` | `/users/me` | Required | Get current user profile |
| `PATCH` | `/users/me` | Required | Update name, grade, stream |

### Content
| Method | Route | Auth | Description |
|--------|-------|------|-------------|
| `GET` | `/grades` | Required | List all grades |
| `GET` | `/grades/{id}/subjects` | Required | List subjects for a grade |
| `GET` | `/subjects/{id}/units` | Required | List units organized by term |
| `GET` | `/units/{id}/cards` | Required | Get all flashcards in a unit |

### Reviews & SRS
| Method | Route | Auth | Description |
|--------|-------|------|-------------|
| `POST` | `/reviews` | Required | Submit batch of card review results |
| `GET` | `/reviews/due` | Required | Get cards due for review today |

### Study Plans
| Method | Route | Auth | Description |
|--------|-------|------|-------------|
| `POST` | `/study-plans` | Required | Create a new study plan |
| `GET` | `/study-plans` | Required | List user's study plans |
| `GET` | `/study-plans/{id}` | Required | Get study plan details |
| `GET` | `/study-plans/{id}/today` | Required | Get today's tasks |
| `PATCH` | `/study-plan-tasks/{id}` | Required | Mark task as completed |

### Analytics
| Method | Route | Auth | Description |
|--------|-------|------|-------------|
| `GET` | `/analytics/summary` | Required | Dashboard summary stats |
| `GET` | `/analytics/streak` | Required | Streak data + 365-day activity |
| `GET` | `/analytics/weak-areas` | Required | Subjects/units needing attention |

### Leaderboard
| Method | Route | Auth | Description |
|--------|-------|------|-------------|
| `GET` | `/leaderboard/grade` | Required | Grade-level leaderboard |
| `GET` | `/leaderboard/subject/{id}` | Required | Subject-level leaderboard |

### Admin (Protected)
| Method | Route | Auth | Description |
|--------|-------|------|-------------|
| `POST` | `/admin/grades` | Admin | Create grade |
| `POST` | `/admin/subjects` | Admin | Create subject |
| `POST` | `/admin/terms` | Admin | Create term |
| `POST` | `/admin/units` | Admin | Create unit |
| `POST` | `/admin/cards` | Admin | Create flashcard |
| `PUT` | `/admin/cards/{id}` | Admin | Update flashcard |
| `DELETE` | `/admin/cards/{id}` | Admin | Delete flashcard |
| `POST` | `/admin/cards/bulk` | Admin | Bulk import cards (CSV) |

---

## Authentication Flow

### OTP Request
```
Client                        Server (Go)                     Redis              ShoutOUT
  │                             │                               │                   │
  │  POST /auth/otp/request     │                               │                   │
  │  { phone: "+94771234567" }  │                               │                   │
  │ ──────────────────────────> │                               │                   │
  │                             │  Check rate limit             │                   │
  │                             │  GET rate:otp:+94771234567    │                   │
  │                             │ ─────────────────────────────>│                   │
  │                             │  count < 3? OK                │                   │
  │                             │ <─────────────────────────────│                   │
  │                             │                               │                   │
  │                             │  Generate 6-digit OTP         │                   │
  │                             │  SET otp:+94771234567 "482901"│                   │
  │                             │  EXPIRE 300 (5 min)           │                   │
  │                             │ ─────────────────────────────>│                   │
  │                             │                               │                   │
  │                             │  INCR rate:otp:+94771234567   │                   │
  │                             │  EXPIRE 600 (10 min)          │                   │
  │                             │ ─────────────────────────────>│                   │
  │                             │                               │                   │
  │                             │  Send SMS via HTTP client     │                   │
  │                             │ ─────────────────────────────────────────────────>│
  │                             │                               │  SMS delivered    │
  │                             │ <─────────────────────────────────────────────────│
  │                             │                               │                   │
  │  { success: true,           │                               │                   │
  │    message: "OTP sent" }    │                               │                   │
  │ <────────────────────────── │                               │                   │
```

### OTP Verification
```
Client                        Server (Go)                     Redis           PostgreSQL
  │                             │                               │                  │
  │  POST /auth/otp/verify      │                               │                  │
  │  { phone: "+94771234567",   │                               │                  │
  │    otp: "482901" }          │                               │                  │
  │ ──────────────────────────> │                               │                  │
  │                             │  GET otp:+94771234567         │                  │
  │                             │ ─────────────────────────────>│                  │
  │                             │  "482901" (matches!)          │                  │
  │                             │ <─────────────────────────────│                  │
  │                             │                               │                  │
  │                             │  DEL otp:+94771234567         │                  │
  │                             │ ─────────────────────────────>│                  │
  │                             │                               │                  │
  │                             │  Find or create user          │                  │
  │                             │ ────────────────────────────────────────────────>│
  │                             │  user record                  │                  │
  │                             │ <────────────────────────────────────────────────│
  │                             │                               │                  │
  │                             │  Generate JWT access token    │                  │
  │                             │  Generate refresh token       │                  │
  │                             │  Store refresh token hash     │                  │
  │                             │ ────────────────────────────────────────────────>│
  │                             │                               │                  │
  │  { accessToken: "eyJ...",   │                               │                  │
  │    refreshToken: "abc...",  │                               │                  │
  │    isNewUser: true,         │                               │                  │
  │    user: { id, phone,       │                               │                  │
  │      name, gradeId } }      │                               │                  │
  │ <────────────────────────── │                               │                  │
```

### JWT Token Structure
```json
// Access Token Payload (15-minute expiry)
{
  "sub": "user-uuid-here",
  "grade_id": "grade-uuid",
  "role": "user",
  "iat": 1700000000,
  "exp": 1700000900
}

// Refresh Token: opaque string, stored hashed in DB
// 30-day expiry, rotated on each use
```

---

## Middleware Stack

Chi middleware chain, applied in order:

```go
r := chi.NewRouter()

// 1. Request ID + Logger
r.Use(middleware.RequestID)
r.Use(zerolog.Logger)      // Structured JSON logging

// 2. Recovery (panic handler)
r.Use(middleware.Recoverer)

// 3. CORS
r.Use(cors.Handler(cors.Options{
    AllowedOrigins: []string{"*"},  // Mobile apps
    AllowedMethods: []string{"GET", "POST", "PUT", "PATCH", "DELETE"},
    AllowedHeaders: []string{"Authorization", "Content-Type"},
}))

// 4. Rate Limiter (global)
r.Use(RateLimitMiddleware(redisClient))

// Public routes
r.Post("/auth/otp/request", handler.RequestOTP)
r.Post("/auth/otp/verify", handler.VerifyOTP)
r.Post("/auth/token/refresh", handler.RefreshToken)
r.Get("/health", handler.HealthCheck)

// Protected routes (JWT required)
r.Group(func(r chi.Router) {
    r.Use(JWTAuthMiddleware(jwtSecret))
    
    r.Get("/users/me", handler.GetProfile)
    r.Patch("/users/me", handler.UpdateProfile)
    r.Get("/grades", handler.ListGrades)
    // ... all authenticated routes
    
    // Admin routes (admin role required)
    r.Group(func(r chi.Router) {
        r.Use(AdminOnlyMiddleware)
        r.Post("/admin/cards", handler.CreateCard)
        // ... all admin routes
    })
})
```

### Request Processing Flow
```
Request
  │
  ├── 1. Request ID + Structured Logger (zerolog)
  │     → Assign unique request ID, log method/path/timestamp
  │
  ├── 2. Recovery
  │     → Catch panics, return 500 instead of crashing
  │
  ├── 3. CORS
  │     → Allow mobile app origins
  │
  ├── 4. Rate Limiter
  │     → Check Redis counter for client IP / phone
  │     → Reject with 429 if over limit
  │
  ├── 5. JWT Verification (on protected routes)
  │     → Extract Bearer token from Authorization header
  │     → Verify signature and expiry with golang-jwt
  │     → Attach user context to request context
  │     → Return 401 if invalid/expired
  │
  ├── 6. Admin Check (on /admin routes)
  │     → Verify user.Role == "admin" from context
  │     → Return 403 if not admin
  │
  ├── 7. Handler
  │     → Decode request body (json.Decoder)
  │     → Validate with go-playground/validator
  │     → Execute business logic via service layer
  │     → Query database via sqlc-generated repository
  │     → Encode response (json.Encoder)
  │
  └── 8. Error Handler
        → Consistent error response format
        → Log errors with context
```

---

## Error Response Format

All error responses follow a consistent structure:

```go
type ErrorResponse struct {
    Error ErrorDetail `json:"error"`
}

type ErrorDetail struct {
    Code    string      `json:"code"`
    Message string      `json:"message"`
    Details interface{} `json:"details,omitempty"`
}
```

```json
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Phone number must start with +94",
    "details": {
      "field": "phone",
      "received": "0771234567"
    }
  }
}
```

### Error Codes
| Code | HTTP Status | Description |
|------|-------------|-------------|
| `VALIDATION_ERROR` | 400 | Request body/params failed validation |
| `INVALID_OTP` | 400 | OTP code doesn't match |
| `OTP_EXPIRED` | 400 | OTP code has expired |
| `UNAUTHORIZED` | 401 | Missing or invalid access token |
| `TOKEN_EXPIRED` | 401 | Access token has expired |
| `REFRESH_TOKEN_INVALID` | 401 | Refresh token invalid or expired |
| `FORBIDDEN` | 403 | User doesn't have required role |
| `NOT_FOUND` | 404 | Resource not found |
| `RATE_LIMITED` | 429 | Too many requests |
| `INTERNAL_ERROR` | 500 | Unexpected server error |

---

## Rate Limiting Strategy

| Endpoint | Limit | Window | Key |
|----------|-------|--------|-----|
| `POST /auth/otp/request` | 3 requests | 10 minutes | Phone number |
| `POST /auth/otp/verify` | 5 attempts | 5 minutes | Phone number |
| `POST /auth/token/refresh` | 10 requests | 1 minute | User ID |
| All other authenticated endpoints | 100 requests | 1 minute | User ID |
| All other public endpoints | 30 requests | 1 minute | IP address |

Implementation: Redis `INCR` with `EXPIRE` for sliding window.

```go
func rateLimitCheck(ctx context.Context, rdb *redis.Client, key string, limit int, window time.Duration) error {
    count, err := rdb.Incr(ctx, key).Result()
    if err != nil {
        return err
    }
    if count == 1 {
        rdb.Expire(ctx, key, window)
    }
    if count > int64(limit) {
        return ErrRateLimited
    }
    return nil
}
```

---

## Study Plan Generation Algorithm

### Input
```json
{
  "goalType": "term_exam",
  "targetTermId": "term-uuid",
  "subjectIds": ["subj-1", "subj-2", "subj-3"],
  "deadline": "2026-07-15",
  "dailyMinutes": 30
}
```

### Algorithm Steps

1. **Gather units**: Fetch all units for the selected subjects (filtered by term if `term_exam`).

2. **Assess current state per unit**:
   - Cards total, cards reviewed, cards due
   - Average accuracy on reviewed cards
   - Estimated minutes to complete (cards x average time per card)

3. **Compute priority score per unit**:
   ```
   priority = (unseen_weight × unseen_ratio)
            + (weakness_weight × (1 - accuracy))
            + (overdue_weight × overdue_ratio)
   
   Where:
     unseen_ratio  = unreviewed_cards / total_cards
     accuracy      = correct_reviews / total_reviews (0 if no reviews)
     overdue_ratio = overdue_cards / total_cards
     
     unseen_weight  = 3.0  (prioritize new content)
     weakness_weight = 2.0  (then weak areas)
     overdue_weight  = 1.0  (then overdue reviews)
   ```

4. **Sort units by priority** (highest first).

5. **Distribute across available days**:
   ```
   available_days = business_days between today and deadline
   daily_capacity = dailyMinutes
   
   For each day:
     remaining_minutes = daily_capacity
     While remaining_minutes > 0 AND unassigned units remain:
       Pick next highest-priority unassigned unit
       estimated_time = unit.card_count × 0.5 min
       Assign unit to this day
       remaining_minutes -= estimated_time
       If unit has many cards, split across multiple days
   ```

6. **Balance review and new**: Ensure each day has a mix:
   - ~60% time on new/unseen units
   - ~40% time on review of previously studied units (due cards)

7. **Store plan**: Create `study_plan` record and `study_plan_tasks` for each day/unit pair.

### Plan Adjustment
If a student misses days, the plan auto-adjusts:
- Uncompleted tasks are redistributed across remaining days
- Priority recalculated (missed units get higher priority)
- If deadline is too close, daily load increases (with a warning to the student)

---

## Leaderboard System

### Scoring
Points are awarded on each review batch sync:

```
For each card review in the batch:
  base_points = 1
  accuracy_bonus = 2 if correct, 0 if incorrect
  card_points = base_points + accuracy_bonus

total_points = sum(card_points) × streak_multiplier

Streak multiplier:
  0-6 days   → 1.0x
  7-29 days  → 1.5x
  30+ days   → 2.0x
```

### Redis Operations

```go
// Update scores (on each review sync)
func (s *LeaderboardService) AddPoints(ctx context.Context, userID, gradeID, subjectID string, points float64) {
    weekStart := currentWeekStart()
    gradeKey := fmt.Sprintf("leaderboard:grade:%s:week:%s", gradeID, weekStart)
    subjectKey := fmt.Sprintf("leaderboard:subject:%s:week:%s", subjectID, weekStart)
    
    s.rdb.ZIncrBy(ctx, gradeKey, points, userID)
    s.rdb.ZIncrBy(ctx, subjectKey, points, userID)
}

// Get user's rank (0-based)
func (s *LeaderboardService) GetRank(ctx context.Context, key, userID string) (int64, error) {
    return s.rdb.ZRevRank(ctx, key, userID).Result()
}

// Get top N with scores
func (s *LeaderboardService) GetTopN(ctx context.Context, key string, n int64) ([]redis.Z, error) {
    return s.rdb.ZRevRangeWithScores(ctx, key, 0, n-1).Result()
}
```

### Weekly Reset (Cron Job)
Runs every Monday at 00:00 IST (UTC+5:30) using a goroutine with a ticker or external cron:

1. For each active leaderboard key:
   - Read all members and scores
   - Insert into `user_scores` table (archive for historical analytics)
2. Delete all `leaderboard:*:week:{currentWeekStart}` keys
3. New week starts with clean scores

---

## Review Batch Sync

### Request
The client sends **raw review results only** — no SRS state. The server runs FSRS and computes all scheduling.
```json
{
  "reviews": [
    {
      "cardId": "card-uuid-1",
      "rating": 3,
      "reviewedAt": "2026-05-01T10:30:00Z"
    }
  ]
}
```

### Server Processing
For each review in the batch:

1. **Store review log**: Insert into `card_reviews` table
2. **Get existing SRS state**: Query `card_srs_state` for this user-card pair
3. **Run FSRS**: Compute new scheduling using `go-fsrs`:
   - If new card (no existing state): use initial values (stability=0, difficulty=0, state="new")
   - If existing card: use current state from database
   - Input: current state + rating → Output: new stability, difficulty, state, due date, reps, lapses
4. **Upsert SRS state**: Store the FSRS-computed state in `card_srs_state`
5. **Update daily activity**: Increment `sessions_count`, `cards_reviewed`, recalculate `accuracy` in `daily_activity`
6. **Update streak**: Check if this is the first review today, if so increment `current_streak` (or reset if missed a day)
7. **Update leaderboard**: Calculate points and `ZINCRBY` Redis sorted sets

### Response
```json
{
  "accepted": 2,
  "rejected": 0
}
```

**Note:** Rejections can happen if a card doesn't exist or the user doesn't have access. There is no conflict resolution needed since the server is the single source of truth — no client-submitted SRS state to conflict with.

---

## Content Versioning

Each unit has a `version` field (integer, incremented on any card change).

**Mobile app flow:**
1. App stores `unitId → version` mapping locally
2. On `GET /subjects/{id}/units`, response includes version per unit
3. If local version < server version, refetch cards for that unit
4. This avoids re-downloading unchanged content

---

## Deployment Configuration

### Dockerfile (Multi-Stage Build)
```dockerfile
# Build stage
FROM golang:1.22-alpine AS builder
WORKDIR /app
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 GOOS=linux go build -ldflags="-s -w" -o /snapy-api ./cmd/api

# Run stage
FROM scratch
COPY --from=builder /snapy-api /snapy-api
COPY --from=builder /etc/ssl/certs/ca-certificates.crt /etc/ssl/certs/
EXPOSE 8080
ENTRYPOINT ["/snapy-api"]
```

Final image: ~15MB (just the binary + CA certs for HTTPS calls to ShoutOUT).

### Environment Variables
```env
# Server
PORT=8080
ENV=production

# Database
DATABASE_URL=postgresql://snapy:password@postgres:5432/snapy?sslmode=disable

# Redis
REDIS_URL=redis://redis:6379

# JWT
JWT_SECRET=<random-64-char-string>
JWT_ACCESS_EXPIRY=15m
JWT_REFRESH_EXPIRY=720h

# ShoutOUT
SHOUTOUT_API_KEY=<your-api-key>
SHOUTOUT_API_SECRET=<your-api-secret>
SHOUTOUT_SENDER_ID=Snapy

# Push Notifications
APNS_KEY_ID=<key-id>
APNS_TEAM_ID=<team-id>
FCM_PROJECT_ID=<project-id>
```

### Health Check Endpoint
```go
func HealthCheck(w http.ResponseWriter, r *http.Request) {
    status := map[string]interface{}{
        "status":   "ok",
        "version":  version,
        "uptime":   time.Since(startTime).Seconds(),
        "database": checkDB(),   // "connected" or "error"
        "redis":    checkRedis(), // "connected" or "error"
    }
    json.NewEncoder(w).Encode(status)
}
```

---

## Project Structure

```
snapy-api/
├── cmd/
│   └── api/
│       └── main.go                    # Entry point: config, DI, server start
├── internal/
│   ├── config/
│   │   └── config.go                  # Environment variable loading
│   ├── handler/
│   │   ├── auth.go                    # OTP request, verify, token refresh
│   │   ├── user.go                    # Profile CRUD
│   │   ├── content.go                 # Grades, subjects, units, cards
│   │   ├── review.go                  # Review submission and due cards
│   │   ├── studyplan.go               # Study plan CRUD and generation
│   │   ├── analytics.go               # Stats, streaks, weak areas
│   │   ├── leaderboard.go             # Leaderboard queries
│   │   ├── admin.go                   # Admin content management
│   │   └── health.go                  # Health check
│   ├── middleware/
│   │   ├── auth.go                    # JWT verification
│   │   ├── admin.go                   # Admin role check
│   │   ├── ratelimit.go               # Redis-based rate limiting
│   │   └── logger.go                  # Request logging (zerolog)
│   ├── service/
│   │   ├── otp.go                     # OTP generation, ShoutOUT HTTP calls
│   │   ├── token.go                   # JWT creation, refresh logic
│   │   ├── fsrs.go                    # FSRS scheduling (go-fsrs, server-side only)
│   │   ├── studyplan.go               # Plan generation algorithm
│   │   ├── leaderboard.go             # Redis leaderboard operations
│   │   ├── analytics.go               # Stats computation
│   │   └── push.go                    # APNs + FCM notification sending
│   ├── repository/                    # sqlc-generated code
│   │   ├── db.go                      # Database interface (generated)
│   │   ├── models.go                  # Go structs from SQL types (generated)
│   │   ├── users.sql.go               # User queries (generated)
│   │   ├── cards.sql.go               # Card queries (generated)
│   │   ├── reviews.sql.go             # Review queries (generated)
│   │   └── ...                        # Other generated query files
│   └── model/
│       ├── request.go                 # Request DTOs with validation tags
│       └── response.go                # Response DTOs
├── db/
│   ├── migrations/
│   │   ├── 000001_init_schema.up.sql
│   │   ├── 000001_init_schema.down.sql
│   │   └── ...
│   ├── queries/
│   │   ├── users.sql                  # SQL queries for sqlc
│   │   ├── cards.sql
│   │   ├── reviews.sql
│   │   ├── studyplans.sql
│   │   ├── analytics.sql
│   │   └── leaderboard.sql
│   └── seed/
│       └── seed.go                    # Seed data (grades, subjects, terms)
├── sqlc.yaml                          # sqlc configuration
├── Dockerfile
├── docker-compose.yml                 # Local dev (PostgreSQL + Redis)
├── go.mod
├── go.sum
└── .env.example
```

### sqlc Configuration
```yaml
# sqlc.yaml
version: "2"
sql:
  - engine: "postgresql"
    queries: "db/queries"
    schema: "db/migrations"
    gen:
      go:
        package: "repository"
        out: "internal/repository"
        sql_package: "pgx/v5"
        emit_json_tags: true
        emit_empty_slices: true
```

### Example: sqlc Query → Generated Go
```sql
-- db/queries/users.sql

-- name: GetUserByPhone :one
SELECT id, phone, name, grade_id, stream, role, created_at, updated_at
FROM users
WHERE phone = $1;

-- name: CreateUser :one
INSERT INTO users (phone)
VALUES ($1)
RETURNING id, phone, name, grade_id, stream, role, created_at, updated_at;

-- name: UpdateUserProfile :one
UPDATE users
SET name = $2, grade_id = $3, stream = $4, updated_at = NOW()
WHERE id = $1
RETURNING id, phone, name, grade_id, stream, role, created_at, updated_at;
```

sqlc generates:
```go
// internal/repository/users.sql.go (auto-generated)

func (q *Queries) GetUserByPhone(ctx context.Context, phone string) (User, error) { ... }
func (q *Queries) CreateUser(ctx context.Context, phone string) (User, error) { ... }
func (q *Queries) UpdateUserProfile(ctx context.Context, arg UpdateUserProfileParams) (User, error) { ... }
```

Type-safe, no runtime reflection, no query building overhead.
