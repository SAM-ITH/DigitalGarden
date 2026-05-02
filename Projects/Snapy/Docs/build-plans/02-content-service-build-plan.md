# Build Plan: Content Service

> **Agent Instructions:** Build the Content microservice for Snapy. Serves the educational content hierarchy (grades → subjects → terms → units → cards) and provides admin CRUD for content management. Runs on port **8082** behind Traefik path prefix `/api/content`.

---

## Prerequisites

Before starting, confirm these exist (built by the shared-library agent):

- [ ] `pkg/common/` — shared library compiles (`go build ./...`)
- [ ] `db/migrations/` — all migrations applied
- [ ] `db/seed/seed.go` — seed data populated
- [ ] `docker-compose.dev.yml` — PostgreSQL running on `localhost:5432`

If any prerequisite is missing, **stop and report back**.

---

## Reference Docs

- `10-api-build-plan.md` — Phase 2 (Steps 2.1–2.8)
- `08-database-design.md` — Tables: `grades`, `subjects`, `terms`, `units`, `cards`
- `05-backend-system.md` — Content API routes, admin routes, content versioning

---

## Directory Structure

```
services/content-service/
├── go.mod
├── go.sum
├── cmd/
│   └── api/
│       └── main.go
├── internal/
│   ├── handler/
│   │   ├── content.go
│   │   └── admin.go
│   ├── service/
│   │   └── content.go
│   ├── repository/          ← sqlc generated
│   │   ├── db.go
│   │   ├── models.go
│   │   ├── grades.sql.go
│   │   ├── subjects.sql.go
│   │   ├── terms.sql.go
│   │   ├── units.sql.go
│   │   └── cards.sql.go
│   └── model/
│       ├── request.go
│       └── response.go
├── db/
│   └── queries/
│       ├── grades.sql
│       ├── subjects.sql
│       ├── terms.sql
│       ├── units.sql
│       └── cards.sql
├── sqlc.yaml
├── Dockerfile
└── .env.example
```

---

## Step 1: Initialize Module

```bash
cd services/content-service
go mod init github.com/yourusername/snapy-api/services/content-service
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
go get encoding/json
```

Run `go mod tidy`.

---

## Step 2: Configure sqlc

Create `services/content-service/sqlc.yaml`:

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

### `db/queries/grades.sql`

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

### `db/queries/subjects.sql`

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
RETURNING id, grade_id, name, name_si, name_ta, icon, color, is_compulsory, stream, display_order, created_at;
```

### `db/queries/terms.sql`

```sql
-- name: ListTermsBySubject :many
SELECT id, subject_id, name, display_order
FROM terms
WHERE subject_id = $1
ORDER BY display_order;

-- name: GetTermByID :one
SELECT id, subject_id, name, display_order
FROM terms
WHERE id = $1;

-- name: CreateTerm :one
INSERT INTO terms (subject_id, name, display_order)
VALUES ($1, $2, $3)
RETURNING id, subject_id, name, display_order;
```

### `db/queries/units.sql`

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
RETURNING id, term_id, name, description, card_count, version, display_order, created_at, updated_at;

-- name: UpdateUnitCardCount :exec
UPDATE units
SET card_count = $2, version = version + 1, updated_at = NOW()
WHERE id = $1;
```

### `db/queries/cards.sql`

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
RETURNING id, unit_id, type, question, answer, choices, explanation, display_order, created_at, updated_at;

-- name: UpdateCard :one
UPDATE cards
SET type = $2, question = $3, answer = $4, choices = $5, explanation = $6, display_order = $7, updated_at = NOW()
WHERE id = $1
RETURNING id, unit_id, type, question, answer, choices, explanation, display_order, created_at, updated_at;

-- name: DeleteCard :exec
DELETE FROM cards WHERE id = $1;

-- name: BulkCreateCards :copyfrom
INSERT INTO cards (unit_id, type, question, answer, choices, explanation, display_order)
VALUES ($1, $2, $3, $4, $5, $6, $7);

-- name: CountCardsByUnit :one
SELECT COUNT(*) FROM cards WHERE unit_id = $1;
```

**Generate:** `sqlc generate`

**Note:** `BulkCreateCards` uses `:copyfrom` — this uses pgx's `CopyFrom` for efficient bulk inserts. The generated method signature will use `pgx.BatchResults` or a slice-based approach. Check the generated code and adapt the service layer accordingly.

---

## Step 4: Request/Response Models

### `internal/model/request.go`

```go
package model

type CreateGradeRequest struct {
    Name            string `json:"name" validate:"required"`
    Level           int    `json:"level" validate:"required,min=10,max=13"`
    EducationStage  string `json:"educationStage" validate:"required,oneof=ol al"`
    DisplayOrder    int    `json:"displayOrder" validate:"required,min=1"`
}

type CreateSubjectRequest struct {
    GradeID       string  `json:"gradeId" validate:"required,uuid"`
    Name          string  `json:"name" validate:"required"`
    NameSi        *string `json:"nameSi"`
    NameTa        *string `json:"nameTa"`
    Icon          *string `json:"icon"`
    Color         *string `json:"color"`
    IsCompulsory  bool    `json:"isCompulsory"`
    Stream        *string `json:"stream" validate:"omitempty,oneof=science commerce arts technology"`
    DisplayOrder  int     `json:"displayOrder" validate:"required,min=1"`
}

type CreateTermRequest struct {
    SubjectID    string `json:"subjectId" validate:"required,uuid"`
    Name         string `json:"name" validate:"required"`
    DisplayOrder int    `json:"displayOrder" validate:"required,min=1"`
}

type CreateUnitRequest struct {
    TermID       string  `json:"termId" validate:"required,uuid"`
    Name         string  `json:"name" validate:"required"`
    Description  *string `json:"description"`
    DisplayOrder int     `json:"displayOrder" validate:"required,min=1"`
}

type CreateCardRequest struct {
    UnitID       string          `json:"unitId" validate:"required,uuid"`
    Type         string          `json:"type" validate:"required,oneof=classic mcq"`
    Question     string          `json:"question" validate:"required"`
    Answer       *string         `json:"answer"`
    Choices      json.RawMessage `json:"choices"`
    Explanation  *string         `json:"explanation"`
    DisplayOrder int             `json:"displayOrder" validate:"required,min=0"`
}

type UpdateCardRequest struct {
    Type         string          `json:"type" validate:"required,oneof=classic mcq"`
    Question     string          `json:"question" validate:"required"`
    Answer       *string         `json:"answer"`
    Choices      json.RawMessage `json:"choices"`
    Explanation  *string         `json:"explanation"`
    DisplayOrder int             `json:"displayOrder" validate:"required,min=0"`
}

type BulkCardImportRequest struct {
    UnitID string             `json:"unitId" validate:"required,uuid"`
    Cards  []CreateCardItem   `json:"cards" validate:"required,min=1,max=500"`
}

type CreateCardItem struct {
    Type         string          `json:"type" validate:"required,oneof=classic mcq"`
    Question     string          `json:"question" validate:"required"`
    Answer       *string         `json:"answer"`
    Choices      json.RawMessage `json:"choices"`
    Explanation  *string         `json:"explanation"`
    DisplayOrder int             `json:"displayOrder" validate:"min=0"`
}
```

### `internal/model/response.go`

```go
package model

type GradeResponse struct {
    ID              string `json:"id"`
    Name            string `json:"name"`
    Level           int    `json:"level"`
    EducationStage  string `json:"educationStage"`
    DisplayOrder    int    `json:"displayOrder"`
}

type SubjectResponse struct {
    ID            string  `json:"id"`
    GradeID       string  `json:"gradeId"`
    Name          string  `json:"name"`
    NameSi        *string `json:"nameSi"`
    NameTa        *string `json:"nameTa"`
    Icon          *string `json:"icon"`
    Color         *string `json:"color"`
    IsCompulsory  bool    `json:"isCompulsory"`
    Stream        *string `json:"stream"`
    DisplayOrder  int     `json:"displayOrder"`
}

type TermResponse struct {
    ID           string `json:"id"`
    SubjectID    string `json:"subjectId"`
    Name         string `json:"name"`
    DisplayOrder int    `json:"displayOrder"`
}

type UnitResponse struct {
    ID           string  `json:"id"`
    TermID       string  `json:"termId"`
    Name         string  `json:"name"`
    Description  *string `json:"description"`
    CardCount    int     `json:"cardCount"`
    Version      int     `json:"version"`
    DisplayOrder int     `json:"displayOrder"`
}

type CardResponse struct {
    ID           string          `json:"id"`
    UnitID       string          `json:"unitId"`
    Type         string          `json:"type"`
    Question     string          `json:"question"`
    Answer       *string         `json:"answer,omitempty"`
    Choices      json.RawMessage `json:"choices,omitempty"`
    Explanation  *string         `json:"explanation,omitempty"`
    DisplayOrder int             `json:"displayOrder"`
}

type SubjectUnitsResponse struct {
    Terms []TermWithUnits `json:"terms"`
}

type TermWithUnits struct {
    Term  TermResponse   `json:"term"`
    Units []UnitResponse `json:"units"`
}

type BulkImportResponse struct {
    Count int `json:"count"`
}
```

---

## Step 5: Service Layer

### `internal/service/content.go`

```go
package service

type ContentService interface {
    // Public (authenticated user)
    ListGrades(ctx context.Context) ([]model.GradeResponse, error)
    ListSubjects(ctx context.Context, gradeID string) ([]model.SubjectResponse, error)
    ListUnitsBySubject(ctx context.Context, subjectID string) (*model.SubjectUnitsResponse, error)
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

type contentService struct {
    queries *repository.Queries
    pool    *pgxpool.Pool
}

func NewContentService(queries *repository.Queries, pool *pgxpool.Pool) ContentService
```

### Key Logic

**ListUnitsBySubject:**
1. Query `ListUnitsBySubject(subjectID)` — returns flat list of units
2. Get distinct term IDs from the result
3. Query `ListTermsBySubject` to get term metadata
4. Group units by `term_id`
5. Build `SubjectUnitsResponse` with `TermWithUnits` nested structure
6. Terms and units sorted by `display_order`

**CreateCard:**
1. Validate that card type is correct:
   - `classic` → `answer` must be non-nil
   - `mcq` → `choices` must be non-nil and valid JSON
2. Insert via `CreateCard`
3. After insert, call `UpdateUnitCardCount` with `CountCardsByUnit` result

**DeleteCard:**
1. Delete via `DeleteCard`
2. After delete, call `UpdateUnitCardCount` with updated count
3. This also increments the unit `version` (triggers cache invalidation on mobile)

**BulkImportCards:**
1. Validate unit exists via `GetUnitByID`
2. For each card in the batch, set `unitID` from the request
3. Use `BulkCreateCards` (pgx CopyFrom) for efficient insert
4. Update `UpdateUnitCardCount` with total count after import
5. Return `BulkImportResponse{Count: len(cards)}`

---

## Step 6: Handler Layer

### `internal/handler/content.go`

Public content handlers:

```go
func (h *ContentHandler) ListGrades(w http.ResponseWriter, r *http.Request)
func (h *ContentHandler) ListSubjects(w http.ResponseWriter, r *http.Request)
func (h *ContentHandler) ListUnitsBySubject(w http.ResponseWriter, r *http.Request)
func (h *ContentHandler) ListCardsByUnit(w http.ResponseWriter, r *http.Request)
```

URL parameter extraction: `chi.URLParam(r, "gradeId")`, etc.

### `internal/handler/admin.go`

Admin CRUD handlers:

```go
func (h *ContentHandler) CreateGrade(w http.ResponseWriter, r *http.Request)
func (h *ContentHandler) CreateSubject(w http.ResponseWriter, r *http.Request)
func (h *ContentHandler) CreateTerm(w http.ResponseWriter, r *http.Request)
func (h *ContentHandler) CreateUnit(w http.ResponseWriter, r *http.Request)
func (h *ContentHandler) CreateCard(w http.ResponseWriter, r *http.Request)
func (h *ContentHandler) UpdateCard(w http.ResponseWriter, r *http.Request)
func (h *ContentHandler) DeleteCard(w http.ResponseWriter, r *http.Request)
func (h *ContentHandler) BulkImportCards(w http.ResponseWriter, r *http.Request)
```

---

## Step 7: Main Entry Point

### `cmd/api/main.go`

```go
func main() {
    // 1. Load config
    // 2. Connect to PostgreSQL
    // 3. Run migrations
    // 4. Create repository + pool
    // 5. Create service
    // 6. Create handler
    // 7. Create chi router with global middleware:
    //    - RequestID, Logger, Recovery, CORS
    // 8. Public + JWT routes:
    //    GET    /grades                     → ListGrades         (JWT)
    //    GET    /grades/{gradeId}/subjects   → ListSubjects       (JWT)
    //    GET    /subjects/{subjectId}/units  → ListUnitsBySubject (JWT)
    //    GET    /units/{unitId}/cards        → ListCardsByUnit    (JWT)
    //    GET    /health                      → Health check       (public)
    // 9. Admin routes (JWT + AdminOnly):
    //    POST   /admin/grades               → CreateGrade
    //    POST   /admin/subjects             → CreateSubject
    //    POST   /admin/terms                → CreateTerm
    //    POST   /admin/units                → CreateUnit
    //    POST   /admin/cards                → CreateCard
    //    PUT    /admin/cards/{cardId}        → UpdateCard
    //    DELETE /admin/cards/{cardId}        → DeleteCard
    //    POST   /admin/cards/bulk            → BulkImportCards
    // 10. Start server on port 8082
}
```

**Middleware chain for admin routes:**
```go
r.Group(func(r chi.Router) {
    r.Use(middleware.JWTAuthMiddleware(config.JWTSecret))
    r.Use(middleware.AdminOnlyMiddleware)
    // admin routes here
})
```

---

## Step 8: Dockerfile

```dockerfile
FROM golang:1.22-alpine AS builder
WORKDIR /app
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 GOOS=linux go build -ldflags="-s -w" -o /content-service ./cmd/api

FROM scratch
COPY --from=builder /content-service /content-service
COPY --from=builder /etc/ssl/certs/ca-certificates.crt /etc/ssl/certs/
EXPOSE 8082
ENTRYPOINT ["/content-service"]
```

---

## Step 9: .env.example

```
PORT=8082
DATABASE_URL=postgresql://snapy:snapydev@localhost:5432/snapy?sslmode=disable
JWT_SECRET=change-me-to-a-64-char-random-string
JWT_ACCESS_EXPIRY=15m
ENV=development
```

---

## Step 10: Verification

```bash
cd services/content-service
go mod tidy
go build ./...
sqlc generate

# Start service
go run cmd/api/main.go

# Public routes (need JWT from auth-service)
TOKEN="<jwt-from-auth-service>"

curl -s http://localhost:8082/grades -H "Authorization: Bearer $TOKEN"
# Expected: Array of 4 grades

curl -s http://localhost:8082/grades/<gradeId>/subjects -H "Authorization: Bearer $TOKEN"
# Expected: Array of subjects for that grade

curl -s http://localhost:8082/subjects/<subjectId>/units -H "Authorization: Bearer $TOKEN"
# Expected: Nested terms → units structure

curl -s http://localhost:8082/units/<unitId>/cards -H "Authorization: Bearer $TOKEN"
# Expected: Array of flashcards

# Admin routes (need admin JWT)
ADMIN_TOKEN="<admin-jwt>"

curl -s -X POST http://localhost:8082/admin/cards \
  -H "Authorization: Bearer $ADMIN_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"unitId":"<uuid>","type":"classic","question":"What is 2+2?","answer":"4","displayOrder":1}'
# Expected: Created card

# Non-admin should get 403
curl -s -X POST http://localhost:8082/admin/cards \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"unitId":"<uuid>","type":"classic","question":"test","answer":"test","displayOrder":1}'
# Expected: 403 FORBIDDEN
```

---

## Acceptance Criteria

- [ ] Service compiles and starts on port 8082
- [ ] `GET /grades` returns all grades sorted by display_order
- [ ] `GET /grades/{id}/subjects` returns subjects for a grade
- [ ] `GET /subjects/{id}/units` returns terms with nested units
- [ ] `GET /units/{id}/cards` returns all cards for a unit
- [ ] Admin `POST /admin/grades` creates a new grade
- [ ] Admin `POST /admin/subjects` creates a new subject
- [ ] Admin `POST /admin/terms` creates a new term
- [ ] Admin `POST /admin/units` creates a new unit
- [ ] Admin `POST /admin/cards` creates a card and updates unit card_count
- [ ] Admin `PUT /admin/cards/{id}` updates a card and increments unit version
- [ ] Admin `DELETE /admin/cards/{id}` deletes a card and updates card_count
- [ ] Admin `POST /admin/cards/bulk` imports multiple cards efficiently
- [ ] Non-admin users get 403 on admin routes
- [ ] Unauthenticated requests get 401
- [ ] Health check returns 200
- [ ] Dockerfile builds successfully
- [ ] MCQ cards validate choices JSON structure
- [ ] Unit version increments on card create/update/delete
