# Build Plan: Study Plan Service

> **Agent Instructions:** Build the Study Plan microservice for Snapy. Handles study plan creation with intelligent task scheduling, daily task retrieval, and task completion tracking. Runs on port **8084** behind Traefik path prefix `/api/study-plans`.

---

## Prerequisites

Before starting, confirm these exist:

- [ ] `pkg/common/` — shared library compiles
- [ ] `db/migrations/` — all migrations applied (including `study_plans`, `study_plan_tasks`, `units`, `card_srs_state`)
- [ ] `docker-compose.dev.yml` — PostgreSQL running on `localhost:5432`

If any prerequisite is missing, **stop and report back**.

---

## Reference Docs

- `10-api-build-plan.md` — Phase 4 (Steps 4.1–4.6)
- `08-database-design.md` — Tables: `study_plans`, `study_plan_tasks`, `units`, `card_srs_state`
- `05-backend-system.md` — Study plan generation algorithm, plan adjustment logic
- `09-spaced-repetition-engine.md` — SRS state structure (stability, difficulty, state, lapses)

---

## Directory Structure

```
services/studyplan-service/
├── go.mod
├── go.sum
├── cmd/
│   └── api/
│       └── main.go
├── internal/
│   ├── handler/
│   │   └── studyplan.go
│   ├── service/
│   │   ├── studyplan.go
│   │   └── generator.go
│   ├── repository/          ← sqlc generated
│   │   ├── db.go
│   │   ├── models.go
│   │   ├── study_plans.sql.go
│   │   └── study_plan_tasks.sql.go
│   └── model/
│       ├── request.go
│       └── response.go
├── db/
│   └── queries/
│       ├── study_plans.sql
│       └── study_plan_tasks.sql
├── sqlc.yaml
├── Dockerfile
└── .env.example
```

---

## Step 1: Initialize Module

```bash
cd services/studyplan-service
go mod init github.com/yourusername/snapy-api/services/studyplan-service
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

Same pattern as other services. Create `services/studyplan-service/sqlc.yaml`:

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

### `db/queries/study_plans.sql`

```sql
-- name: CreateStudyPlan :one
INSERT INTO study_plans (user_id, name, goal_type, target_term_id, deadline, daily_minutes)
VALUES ($1, $2, $3, $4, $5, $6)
RETURNING id, user_id, name, goal_type, target_term_id, deadline, daily_minutes, is_active, created_at;

-- name: GetStudyPlanByID :one
SELECT id, user_id, name, goal_type, target_term_id, deadline, daily_minutes, is_active, created_at
FROM study_plans
WHERE id = $1;

-- name: ListStudyPlansByUser :many
SELECT id, user_id, name, goal_type, target_term_id, deadline, daily_minutes, is_active, created_at
FROM study_plans
WHERE user_id = $1
ORDER BY created_at DESC;

-- name: UpdateStudyPlanActive :one
UPDATE study_plans SET is_active = $2 WHERE id = $1
RETURNING id, user_id, name, goal_type, target_term_id, deadline, daily_minutes, is_active, created_at;

-- name: GetActivePlanByUser :one
SELECT id, user_id, name, goal_type, target_term_id, deadline, daily_minutes, is_active, created_at
FROM study_plans
WHERE user_id = $1 AND is_active = true
ORDER BY created_at DESC
LIMIT 1;
```

### `db/queries/study_plan_tasks.sql`

```sql
-- name: CreateStudyPlanTask :one
INSERT INTO study_plan_tasks (plan_id, unit_id, scheduled_date, task_type, estimated_minutes)
VALUES ($1, $2, $3, $4, $5)
RETURNING id, plan_id, unit_id, scheduled_date, task_type, estimated_minutes, is_completed, completed_at;

-- name: ListTasksByPlanAndDate :many
SELECT id, plan_id, unit_id, scheduled_date, task_type, estimated_minutes, is_completed, completed_at
FROM study_plan_tasks
WHERE plan_id = $1 AND scheduled_date = $2
ORDER BY task_type, unit_id;

-- name: ListTasksByPlan :many
SELECT id, plan_id, unit_id, scheduled_date, task_type, estimated_minutes, is_completed, completed_at
FROM study_plan_tasks
WHERE plan_id = $1
ORDER BY scheduled_date, task_type, unit_id;

-- name: GetTaskByID :one
SELECT id, plan_id, unit_id, scheduled_date, task_type, estimated_minutes, is_completed, completed_at
FROM study_plan_tasks
WHERE id = $1;

-- name: CompleteTask :one
UPDATE study_plan_tasks
SET is_completed = true, completed_at = NOW()
WHERE id = $1
RETURNING id, plan_id, unit_id, scheduled_date, task_type, estimated_minutes, is_completed, completed_at;

-- name: GetPlanProgress :one
SELECT
    COUNT(*) AS total_tasks,
    COUNT(*) FILTER (WHERE is_completed) AS completed_tasks
FROM study_plan_tasks
WHERE plan_id = $1;

-- name: GetIncompleteTasksByPlan :many
SELECT id, plan_id, unit_id, scheduled_date, task_type, estimated_minutes, is_completed, completed_at
FROM study_plan_tasks
WHERE plan_id = $1 AND is_completed = false
ORDER BY scheduled_date;

-- name: DeleteTasksByPlan :exec
DELETE FROM study_plan_tasks WHERE plan_id = $1;
```

### Additional queries needed for plan generation

### `db/queries/units.sql`

```sql
-- name: GetUnitsByTerm :many
SELECT id, term_id, name, description, card_count, version, display_order
FROM units
WHERE term_id = $1
ORDER BY display_order;

-- name: GetUnitByID :one
SELECT id, term_id, name, description, card_count, version, display_order
FROM units
WHERE id = $1;
```

### `db/queries/srs_state.sql`

```sql
-- name: GetUnitProgressForUser :many
SELECT u.id AS unit_id, u.name AS unit_name, u.card_count,
       COUNT(cs.card_id) AS cards_studied,
       COUNT(CASE WHEN cs.state = 'review' THEN 1 END) AS cards_mastered,
       COALESCE(SUM(cs.lapses), 0) AS total_lapses,
       COUNT(CASE WHEN cs.due_at <= NOW() THEN 1 END) AS overdue_cards
FROM units u
LEFT JOIN cards c ON c.unit_id = u.id
LEFT JOIN card_srs_state cs ON cs.card_id = c.id AND cs.user_id = $1
WHERE u.id = ANY($2::uuid[])
GROUP BY u.id, u.name, u.card_count
ORDER BY u.display_order;
```

**Generate:** `sqlc generate`

---

## Step 4: Request/Response Models

### `internal/model/request.go`

```go
package model

type CreateStudyPlanRequest struct {
    Name          string  `json:"name" validate:"required,min=2,max=100"`
    GoalType      string  `json:"goalType" validate:"required,oneof=term_exam catch_up custom"`
    TargetTermID  *string `json:"targetTermId" validate:"omitempty,uuid"`
    Deadline      string  `json:"deadline" validate:"required"` // YYYY-MM-DD
    DailyMinutes  int     `json:"dailyMinutes" validate:"required,min=5,max=300"`
}
```

### `internal/model/response.go`

```go
package model

type StudyPlanResponse struct {
    ID              string              `json:"id"`
    Name            string              `json:"name"`
    GoalType        string              `json:"goalType"`
    TargetTermID    *string             `json:"targetTermId,omitempty"`
    Deadline        string              `json:"deadline"`
    DailyMinutes    int                 `json:"dailyMinutes"`
    IsActive        bool                `json:"isActive"`
    CreatedAt       string              `json:"createdAt"`
    Progress        *PlanProgress       `json:"progress,omitempty"`
}

type PlanProgress struct {
    TotalTasks     int `json:"totalTasks"`
    CompletedTasks int `json:"completedTasks"`
    Percentage     int `json:"percentage"`
}

type TaskResponse struct {
    ID               string  `json:"id"`
    PlanID           string  `json:"planId"`
    UnitID           string  `json:"unitId"`
    UnitName         string  `json:"unitName"`
    ScheduledDate    string  `json:"scheduledDate"`
    TaskType         string  `json:"taskType"`
    EstimatedMinutes int     `json:"estimatedMinutes"`
    IsCompleted      bool    `json:"isCompleted"`
    CompletedAt      *string `json:"completedAt,omitempty"`
}

type TaskListResponse struct {
    Date  string         `json:"date"`
    Tasks []TaskResponse `json:"tasks"`
}

type StudyPlanListResponse struct {
    Plans []StudyPlanResponse `json:"plans"`
}
```

---

## Step 5: Plan Generator

### `internal/service/generator.go`

This is the core plan generation algorithm. It creates daily tasks based on user's progress and deadline.

```go
package service

type PlanGenerator struct {
    queries *repository.Queries
    pool    *pgxpool.Pool
}

func NewPlanGenerator(queries *repository.Queries, pool *pgxpool.Pool) *PlanGenerator
```

### GenerateTasks Method

**Input:** `planID`, `userID`, `targetTermID`, `deadline`, `dailyMinutes`

**Algorithm:**

1. **Gather units:**
   - If `targetTermID` is set: `GetUnitsByTerm(targetTermID)`
   - Else: fetch all units for user's enrolled subjects (need user's grade from context)

2. **Assess current state per unit** using `GetUnitProgressForUser(userID, unitIDs)`:
   - `cards_studied` / `card_count` = seen ratio
   - `cards_mastered` / `card_count` = mastery ratio
   - `total_lapses` = weakness indicator
   - `overdue_cards` = urgency indicator

3. **Compute priority score per unit:**
   ```
   unseen_ratio = (card_count - cards_studied) / card_count
   accuracy = cards_mastered / max(cards_studied, 1)
   overdue_ratio = overdue_cards / card_count

   priority = (3.0 * unseen_ratio)
           + (2.0 * (1.0 - accuracy))
           + (1.0 * overdue_ratio)
   ```

4. **Sort units by priority** (highest first).

5. **Calculate available days:**
   ```
   available_days = days_between(today, deadline)
   if available_days < 1: return error "deadline must be in the future"
   ```

6. **Distribute across days:**
   - For each unit, estimate time: `card_count * 0.5` minutes per card
   - Daily capacity: `dailyMinutes`
   - Split: 60% new content, 40% review per day
   - For each day:
     ```
     new_budget = daily_minutes * 0.6
     review_budget = daily_minutes * 0.4

     assign new-type tasks until new_budget exhausted
     assign review-type tasks for units with overdue cards until review_budget exhausted
     ```

7. **Insert tasks:** For each day/unit assignment:
   - `CreateStudyPlanTask(planID, unitID, scheduledDate, taskType, estimatedMinutes)`
   - `taskType` = `"new"` for unseen units, `"review"` for units with existing progress

**All task inserts should be in a single transaction.**

---

## Step 6: Service Layer

### `internal/service/studyplan.go`

```go
package service

type StudyPlanService interface {
    CreatePlan(ctx context.Context, userID string, req model.CreateStudyPlanRequest) (*model.StudyPlanResponse, error)
    ListPlans(ctx context.Context, userID string) (*model.StudyPlanListResponse, error)
    GetPlan(ctx context.Context, planID string) (*model.StudyPlanResponse, error)
    GetTodayTasks(ctx context.Context, planID string) (*model.TaskListResponse, error)
    CompleteTask(ctx context.Context, taskID string) (*model.TaskResponse, error)
    DeactivatePlan(ctx context.Context, planID string) (*model.StudyPlanResponse, error)
}

type studyplanService struct {
    queries   *repository.Queries
    pool      *pgxpool.Pool
    generator *PlanGenerator
}

func NewStudyPlanService(queries *repository.Queries, pool *pgxpool.Pool) StudyPlanService
```

### CreatePlan Logic

1. Parse `deadline` string to `time.Time`
2. Validate deadline is in the future
3. If user has an active plan, deactivate it (`UpdateStudyPlanActive(existingPlanID, false)`)
4. Insert plan: `CreateStudyPlan(userID, name, goalType, targetTermID, deadline, dailyMinutes)`
5. Run plan generator: `generator.GenerateTasks(ctx, planID, userID, targetTermID, deadline, dailyMinutes)`
6. Get plan progress: `GetPlanProgress(planID)`
7. Map to `StudyPlanResponse` and return

### ListPlans Logic

1. `ListStudyPlansByUser(userID)`
2. For each plan, get progress stats
3. Map to `StudyPlanListResponse`

### GetPlan Logic

1. `GetStudyPlanByID(planID)`
2. Verify plan belongs to requesting user (get userID from context)
3. Get progress: `GetPlanProgress(planID)`
4. Map to `StudyPlanResponse` with progress

### GetTodayTasks Logic

1. Verify plan belongs to user
2. Get today's date: `time.Now().Format("2006-01-02")`
3. `ListTasksByPlanAndDate(planID, today)`
4. For each task, get unit name via `GetUnitByID`
5. Map to `TaskListResponse`

### CompleteTask Logic

1. `GetTaskByID(taskID)` — verify task exists
2. Verify the task's plan belongs to requesting user
3. Check task is not already completed
4. `CompleteTask(taskID)` — sets `is_completed=true`, `completed_at=NOW()`
5. Map to `TaskResponse`

### DeactivatePlan Logic

1. Verify plan belongs to user
2. `UpdateStudyPlanActive(planID, false)`
3. Map to `StudyPlanResponse`

---

## Step 7: Handler Layer

### `internal/handler/studyplan.go`

```go
type StudyPlanHandler struct {
    service service.StudyPlanService
}

func NewStudyPlanHandler(service service.StudyPlanService) *StudyPlanHandler

func (h *StudyPlanHandler) CreatePlan(w http.ResponseWriter, r *http.Request)
func (h *StudyPlanHandler) ListPlans(w http.ResponseWriter, r *http.Request)
func (h *StudyPlanHandler) GetPlan(w http.ResponseWriter, r *http.Request)
func (h *StudyPlanHandler) GetTodayTasks(w http.ResponseWriter, r *http.Request)
func (h *StudyPlanHandler) CompleteTask(w http.ResponseWriter, r *http.Request)
```

**Error mapping:**

| Error | HTTP Status | Code |
|---|---|---|
| Plan not found | 404 | `NOT_FOUND` |
| Task not found | 404 | `NOT_FOUND` |
| Not owner of plan | 403 | `FORBIDDEN` |
| Deadline in past | 400 | `VALIDATION_ERROR` |
| Task already completed | 409 | `CONFLICT` |

---

## Step 8: Main Entry Point

### `cmd/api/main.go`

```go
func main() {
    // 1. Load config
    // 2. Connect to PostgreSQL
    // 3. Run migrations
    // 4. Create repository + pool
    // 5. Create plan generator
    // 6. Create study plan service
    // 7. Create handler
    // 8. Create chi router with middleware
    // 9. All routes JWT protected:
    //    POST   /plans              → CreatePlan       (JWT)
    //    GET    /plans              → ListPlans        (JWT)
    //    GET    /plans/{planId}     → GetPlan          (JWT)
    //    GET    /plans/{planId}/today → GetTodayTasks   (JWT)
    //    PATCH  /tasks/{taskId}     → CompleteTask     (JWT)
    //    DELETE /plans/{planId}     → DeactivatePlan   (JWT)
    //    GET    /health             → health check      (public)
    // 10. Start server on port 8084
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
RUN CGO_ENABLED=0 GOOS=linux go build -ldflags="-s -w" -o /studyplan-service ./cmd/api

FROM scratch
COPY --from=builder /studyplan-service /studyplan-service
COPY --from=builder /etc/ssl/certs/ca-certificates.crt /etc/ssl/certs/
EXPOSE 8084
ENTRYPOINT ["/studyplan-service"]
```

---

## Step 10: .env.example

```
PORT=8084
DATABASE_URL=postgresql://snapy:snapydev@localhost:5432/snapy?sslmode=disable
JWT_SECRET=change-me-to-a-64-char-random-string
JWT_ACCESS_EXPIRY=15m
ENV=development
```

---

## Step 11: Verification

```bash
cd services/studyplan-service
go mod tidy
go build ./...
sqlc generate

# Start service
go run cmd/api/main.go

TOKEN="<jwt-from-auth-service>"

# Create a study plan
curl -s -X POST http://localhost:8084/plans \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Term 1 Exam Prep",
    "goalType": "term_exam",
    "targetTermId": "<term-uuid>",
    "deadline": "2026-08-15",
    "dailyMinutes": 30
  }'
# Expected: Plan created with progress stats

# List plans
curl -s http://localhost:8084/plans \
  -H "Authorization: Bearer $TOKEN"
# Expected: Array of user's plans

# Get plan details
curl -s http://localhost:8084/plans/<planId> \
  -H "Authorization: Bearer $TOKEN"
# Expected: Plan with progress (totalTasks, completedTasks, percentage)

# Get today's tasks
curl -s http://localhost:8084/plans/<planId>/today \
  -H "Authorization: Bearer $TOKEN"
# Expected: Today's date with list of tasks

# Complete a task
curl -s -X PATCH http://localhost:8084/tasks/<taskId> \
  -H "Authorization: Bearer $TOKEN"
# Expected: Updated task with isCompleted=true, completedAt set
```

---

## Acceptance Criteria

- [ ] Service compiles and starts on port 8084
- [ ] `POST /plans` creates a plan and generates daily tasks
- [ ] Plan generation distributes tasks across available days
- [ ] Priority scoring prioritizes unseen content (weight 3.0) > weak areas (2.0) > overdue (1.0)
- [ ] Daily split: 60% new content, 40% review
- [ ] Creating a new plan deactivates any existing active plan
- [ ] `GET /plans` lists all user's plans
- [ ] `GET /plans/{id}` returns plan with progress stats
- [ ] `GET /plans/{id}/today` returns today's scheduled tasks
- [ ] `PATCH /tasks/{id}` marks task as completed with timestamp
- [ ] Completing already-completed task returns 409
- [ ] Users can only access their own plans (403 for others' plans)
- [ ] Deadline validation rejects past dates
- [ ] Health check returns 200
- [ ] Dockerfile builds successfully
