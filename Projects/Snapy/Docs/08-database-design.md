# Snapy - Database Schema & Design

## Overview

Snapy uses **PostgreSQL 16** as the primary data store for all persistent, relational data, and **Redis 7** for caching, ephemeral data, and real-time data structures (leaderboards, OTP codes, rate limiting).

---

## PostgreSQL Schema

### Entity Relationship Overview

```
users ──────────────┬──── refresh_tokens
  │                 │
  │                 ├──── card_reviews
  │                 │
  │                 ├──── card_srs_state
  │                 │
  │                 ├──── study_plans ──── study_plan_tasks
  │                 │
  │                 ├──── user_scores
  │                 │
  │                 ├──── user_streaks
  │                 │
  │                 └──── daily_activity
  │
grades ──── subjects ──── terms ──── units ──── cards
```

---

### Table: `users`
Stores all registered user accounts.

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `id` | `UUID` | PK, DEFAULT gen_random_uuid() | Unique user identifier |
| `phone` | `VARCHAR(15)` | UNIQUE, NOT NULL | Phone number with country code (+94...) |
| `name` | `VARCHAR(100)` | NULL | User's display name (set during onboarding) |
| `grade_id` | `UUID` | FK → grades.id, NULL | Current grade |
| `stream` | `VARCHAR(20)` | NULL | A/L stream (science/commerce/arts/technology), null for O/L |
| `role` | `VARCHAR(10)` | NOT NULL, DEFAULT 'user' | 'user' or 'admin' |
| `created_at` | `TIMESTAMPTZ` | NOT NULL, DEFAULT now() | Account creation timestamp |
| `updated_at` | `TIMESTAMPTZ` | NOT NULL, DEFAULT now() | Last profile update |

**Indexes:**
- `users_phone_idx` UNIQUE on `phone`

---

### Table: `refresh_tokens`
Stores hashed refresh tokens for JWT authentication.

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `id` | `UUID` | PK, DEFAULT gen_random_uuid() | Token record ID |
| `user_id` | `UUID` | FK → users.id, NOT NULL | Token owner |
| `token_hash` | `VARCHAR(64)` | NOT NULL | SHA-256 hash of the refresh token |
| `expires_at` | `TIMESTAMPTZ` | NOT NULL | Token expiration (30 days from creation) |
| `created_at` | `TIMESTAMPTZ` | NOT NULL, DEFAULT now() | Token creation timestamp |

**Indexes:**
- `refresh_tokens_token_hash_idx` on `token_hash`
- `refresh_tokens_user_id_idx` on `user_id`

**Cleanup:** Cron job deletes rows where `expires_at < now()` daily.

---

### Table: `grades`
Sri Lankan school grades supported by the app.

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `id` | `UUID` | PK, DEFAULT gen_random_uuid() | Grade ID |
| `name` | `VARCHAR(20)` | NOT NULL | Display name (e.g., "Grade 10") |
| `level` | `INTEGER` | NOT NULL, UNIQUE | Numeric level (10, 11, 12, 13) |
| `education_stage` | `VARCHAR(20)` | NOT NULL | 'ol' (Grade 10-11) or 'al' (Grade 12-13) |
| `display_order` | `INTEGER` | NOT NULL | Sort order for UI |

**Seed data:** 4 rows (Grade 10, 11, 12, 13).

---

### Table: `subjects`
Subjects available per grade.

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `id` | `UUID` | PK, DEFAULT gen_random_uuid() | Subject ID |
| `grade_id` | `UUID` | FK → grades.id, NOT NULL | Which grade this subject belongs to |
| `name` | `VARCHAR(100)` | NOT NULL | Subject name (e.g., "Mathematics") |
| `name_si` | `VARCHAR(100)` | NULL | Sinhala name |
| `name_ta` | `VARCHAR(100)` | NULL | Tamil name |
| `icon` | `VARCHAR(50)` | NULL | Icon identifier (e.g., "math", "science") |
| `color` | `VARCHAR(7)` | NULL | Hex color for UI (e.g., "#4CAF50") |
| `is_compulsory` | `BOOLEAN` | NOT NULL, DEFAULT false | Whether compulsory for the grade |
| `stream` | `VARCHAR(20)` | NULL | A/L stream (null if applicable to all or O/L) |
| `display_order` | `INTEGER` | NOT NULL | Sort order within grade |
| `created_at` | `TIMESTAMPTZ` | NOT NULL, DEFAULT now() | |

**Indexes:**
- `subjects_grade_id_order_idx` on `(grade_id, display_order)`

---

### Table: `terms`
Academic terms within a subject.

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `id` | `UUID` | PK, DEFAULT gen_random_uuid() | Term ID |
| `subject_id` | `UUID` | FK → subjects.id, NOT NULL | Parent subject |
| `name` | `VARCHAR(20)` | NOT NULL | "Term 1", "Term 2", "Term 3" |
| `display_order` | `INTEGER` | NOT NULL | Sort order (1, 2, 3) |

**Indexes:**
- `terms_subject_id_order_idx` on `(subject_id, display_order)`

---

### Table: `units`
Study units within a term. Each unit contains a set of flashcards.

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `id` | `UUID` | PK, DEFAULT gen_random_uuid() | Unit ID |
| `term_id` | `UUID` | FK → terms.id, NOT NULL | Parent term |
| `name` | `VARCHAR(200)` | NOT NULL | Unit name (e.g., "Algebra") |
| `description` | `TEXT` | NULL | Brief description of the unit |
| `card_count` | `INTEGER` | NOT NULL, DEFAULT 0 | Denormalized count of cards (updated on card add/delete) |
| `version` | `INTEGER` | NOT NULL, DEFAULT 1 | Incremented on any card change (for cache invalidation) |
| `display_order` | `INTEGER` | NOT NULL | Sort order within term |
| `created_at` | `TIMESTAMPTZ` | NOT NULL, DEFAULT now() | |
| `updated_at` | `TIMESTAMPTZ` | NOT NULL, DEFAULT now() | |

**Indexes:**
- `units_term_id_order_idx` on `(term_id, display_order)`

---

### Table: `cards`
Individual flashcards within a unit. Supports both Classic (recall) and MCQ types.

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `id` | `UUID` | PK, DEFAULT gen_random_uuid() | Card ID |
| `unit_id` | `UUID` | FK → units.id, NOT NULL | Parent unit |
| `type` | `VARCHAR(10)` | NOT NULL | 'classic' or 'mcq' |
| `question` | `TEXT` | NOT NULL | Question text |
| `answer` | `TEXT` | NULL | Answer text (for classic cards) |
| `choices` | `JSONB` | NULL | MCQ choices (for mcq cards) |
| `explanation` | `TEXT` | NULL | Explanation shown after answering (especially for MCQ incorrect) |
| `display_order` | `INTEGER` | NOT NULL | Order within the unit |
| `created_at` | `TIMESTAMPTZ` | NOT NULL, DEFAULT now() | |
| `updated_at` | `TIMESTAMPTZ` | NOT NULL, DEFAULT now() | |

**JSONB `choices` format (for MCQ cards):**
```json
[
  { "text": "Mitochondria", "isCorrect": true },
  { "text": "Nucleus", "isCorrect": false },
  { "text": "Ribosome", "isCorrect": false },
  { "text": "Golgi apparatus", "isCorrect": false }
]
```

**Indexes:**
- `cards_unit_id_order_idx` on `(unit_id, display_order)`

**Constraint:** CHECK that either `answer` is NOT NULL (for classic) or `choices` is NOT NULL (for mcq).

---

### Table: `card_reviews`
Historical log of every card review. Immutable append-only table for analytics.

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `id` | `UUID` | PK, DEFAULT gen_random_uuid() | Review ID |
| `user_id` | `UUID` | FK → users.id, NOT NULL | Who reviewed |
| `card_id` | `UUID` | FK → cards.id, NOT NULL | Which card |
| `rating` | `SMALLINT` | NOT NULL | FSRS rating: 1=Again, 2=Hard, 3=Good, 4=Easy |
| `reviewed_at` | `TIMESTAMPTZ` | NOT NULL | When the review happened (client timestamp) |
| `created_at` | `TIMESTAMPTZ` | NOT NULL, DEFAULT now() | When stored on server |

**Indexes:**
- `card_reviews_user_id_reviewed_at_idx` on `(user_id, reviewed_at)` — for analytics queries
- `card_reviews_user_id_card_id_idx` on `(user_id, card_id)` — for per-card history

**Partitioning consideration:** If table grows very large (millions of rows), partition by `reviewed_at` month. Not needed initially.

---

### Table: `card_srs_state`
Current FSRS state for each user-card pair. Updated on every review.

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `user_id` | `UUID` | FK → users.id, NOT NULL | User |
| `card_id` | `UUID` | FK → cards.id, NOT NULL | Card |
| `stability` | `REAL` | NOT NULL, DEFAULT 0 | FSRS stability parameter |
| `difficulty` | `REAL` | NOT NULL, DEFAULT 0 | FSRS difficulty parameter |
| `elapsed_days` | `INTEGER` | NOT NULL, DEFAULT 0 | Days since last review |
| `scheduled_days` | `INTEGER` | NOT NULL, DEFAULT 0 | Days until next review |
| `state` | `VARCHAR(12)` | NOT NULL, DEFAULT 'new' | 'new', 'learning', 'review', 'relearning' |
| `due_at` | `TIMESTAMPTZ` | NOT NULL, DEFAULT now() | When this card is next due |
| `last_reviewed_at` | `TIMESTAMPTZ` | NULL | Last review timestamp (for conflict resolution) |
| `reps` | `INTEGER` | NOT NULL, DEFAULT 0 | Total number of reviews for this card |
| `lapses` | `INTEGER` | NOT NULL, DEFAULT 0 | Number of times card went to "Again" from review state |

**Primary Key:** Composite `(user_id, card_id)`

**Indexes:**
- `card_srs_state_user_due_idx` on `(user_id, due_at)` — critical for "due cards" query
- `card_srs_state_user_state_idx` on `(user_id, state)` — for analytics (count by state)

---

### Table: `study_plans`
User-created study plans with goals and parameters.

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `id` | `UUID` | PK, DEFAULT gen_random_uuid() | Plan ID |
| `user_id` | `UUID` | FK → users.id, NOT NULL | Plan owner |
| `name` | `VARCHAR(100)` | NOT NULL | Plan name (e.g., "Term 1 Exam Prep") |
| `goal_type` | `VARCHAR(20)` | NOT NULL | 'term_exam', 'catch_up', 'custom' |
| `target_term_id` | `UUID` | FK → terms.id, NULL | Target term (for term_exam goal) |
| `deadline` | `DATE` | NOT NULL | Target completion date |
| `daily_minutes` | `INTEGER` | NOT NULL | Planned daily study time |
| `is_active` | `BOOLEAN` | NOT NULL, DEFAULT true | Whether plan is active |
| `created_at` | `TIMESTAMPTZ` | NOT NULL, DEFAULT now() | |

**Indexes:**
- `study_plans_user_id_active_idx` on `(user_id, is_active)`

---

### Table: `study_plan_tasks`
Daily tasks within a study plan.

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `id` | `UUID` | PK, DEFAULT gen_random_uuid() | Task ID |
| `plan_id` | `UUID` | FK → study_plans.id, NOT NULL | Parent plan |
| `unit_id` | `UUID` | FK → units.id, NOT NULL | Unit to study |
| `scheduled_date` | `DATE` | NOT NULL | Which day this task is for |
| `task_type` | `VARCHAR(10)` | NOT NULL, DEFAULT 'new' | 'new' (first time) or 'review' |
| `estimated_minutes` | `INTEGER` | NOT NULL | Estimated time for this task |
| `is_completed` | `BOOLEAN` | NOT NULL, DEFAULT false | Whether student completed it |
| `completed_at` | `TIMESTAMPTZ` | NULL | When completed |

**Indexes:**
- `study_plan_tasks_plan_date_idx` on `(plan_id, scheduled_date)` — for "today's tasks" query
- `study_plan_tasks_plan_completed_idx` on `(plan_id, is_completed)` — for progress calculation

---

### Table: `user_scores`
Archived weekly leaderboard scores (from Redis weekly reset).

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `id` | `UUID` | PK, DEFAULT gen_random_uuid() | Score record ID |
| `user_id` | `UUID` | FK → users.id, NOT NULL | User |
| `grade_id` | `UUID` | FK → grades.id, NOT NULL | Grade at the time |
| `subject_id` | `UUID` | FK → subjects.id, NULL | Null for grade-level score |
| `points` | `INTEGER` | NOT NULL | Points earned that week |
| `week_start` | `DATE` | NOT NULL | Monday of the scored week |
| `created_at` | `TIMESTAMPTZ` | NOT NULL, DEFAULT now() | |

**Indexes:**
- `user_scores_grade_week_idx` on `(grade_id, week_start, points DESC)` — historical grade leaderboard
- `user_scores_subject_week_idx` on `(subject_id, week_start, points DESC)` — historical subject leaderboard
- `user_scores_user_id_idx` on `(user_id)` — user's score history

---

### Table: `user_streaks`
Current and longest streak per user.

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `user_id` | `UUID` | PK, FK → users.id | User |
| `current_streak` | `INTEGER` | NOT NULL, DEFAULT 0 | Consecutive active days |
| `longest_streak` | `INTEGER` | NOT NULL, DEFAULT 0 | All-time longest streak |
| `last_active_date` | `DATE` | NULL | Date of last study session |

**Update logic (on each review sync):**
```sql
-- If last_active_date is yesterday: increment streak
-- If last_active_date is today: no change
-- If last_active_date is older: reset streak to 1
-- Update longest_streak if current > longest
```

---

### Table: `daily_activity`
Per-user daily activity summary. Used for streak heatmap rendering.

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `user_id` | `UUID` | FK → users.id, NOT NULL | User |
| `activity_date` | `DATE` | NOT NULL | Date of activity |
| `sessions_count` | `INTEGER` | NOT NULL, DEFAULT 0 | Number of study sessions |
| `cards_reviewed` | `INTEGER` | NOT NULL, DEFAULT 0 | Total cards reviewed |
| `correct_count` | `INTEGER` | NOT NULL, DEFAULT 0 | Correct answers count |
| `study_minutes` | `INTEGER` | NOT NULL, DEFAULT 0 | Estimated study time |

**Primary Key:** Composite `(user_id, activity_date)`

**Indexes:**
- `daily_activity_user_date_idx` on `(user_id, activity_date DESC)` — for heatmap range query (past 365 days)

**Update logic:** `INSERT ... ON CONFLICT (user_id, activity_date) DO UPDATE SET cards_reviewed = cards_reviewed + $1, ...`

---

### Table: `device_tokens`
Push notification tokens for each user's devices.

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `id` | `UUID` | PK, DEFAULT gen_random_uuid() | Token ID |
| `user_id` | `UUID` | FK → users.id, NOT NULL | Token owner |
| `platform` | `VARCHAR(10)` | NOT NULL | 'ios' or 'android' |
| `token` | `TEXT` | NOT NULL | APNs or FCM device token |
| `created_at` | `TIMESTAMPTZ` | NOT NULL, DEFAULT now() | |
| `updated_at` | `TIMESTAMPTZ` | NOT NULL, DEFAULT now() | |

**Indexes:**
- `device_tokens_user_id_idx` on `(user_id)`

**Unique constraint:** on `(user_id, platform, token)` to prevent duplicates.

---

## Key Queries

### Get Due Cards for a User
```sql
SELECT cs.*, c.question, c.answer, c.choices, c.type, c.explanation,
       u.name AS unit_name, s.name AS subject_name
FROM card_srs_state cs
JOIN cards c ON c.id = cs.card_id
JOIN units u ON u.id = c.unit_id
JOIN terms t ON t.id = u.term_id
JOIN subjects s ON s.id = t.subject_id
WHERE cs.user_id = $1
  AND cs.due_at <= NOW()
ORDER BY cs.due_at ASC
LIMIT 50;
```

### Get Streak Heatmap Data (Past 365 Days)
```sql
SELECT activity_date, cards_reviewed, correct_count, sessions_count, study_minutes
FROM daily_activity
WHERE user_id = $1
  AND activity_date >= CURRENT_DATE - INTERVAL '365 days'
ORDER BY activity_date ASC;
```

### Get Weak Areas
```sql
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
LIMIT 10;
```

### Get Unit Progress for a Subject
```sql
SELECT u.id, u.name, u.card_count, u.display_order,
       COUNT(cs.card_id) AS cards_studied,
       COUNT(CASE WHEN cs.state = 'review' THEN 1 END) AS cards_mastered
FROM units u
JOIN terms t ON t.id = u.term_id
LEFT JOIN cards c ON c.unit_id = u.id
LEFT JOIN card_srs_state cs ON cs.card_id = c.id AND cs.user_id = $1
WHERE t.subject_id = $2
GROUP BY u.id, u.name, u.card_count, u.display_order
ORDER BY u.display_order;
```

---

## Redis Data Structures

### OTP Storage
```
Key:     otp:{phone}
Type:    String
Value:   6-digit OTP code (e.g., "482901")
TTL:     300 seconds (5 minutes)
Example: SET otp:+94771234567 "482901" EX 300
```

### OTP Rate Limiting
```
Key:     rate:otp:{phone}
Type:    String (counter)
TTL:     600 seconds (10 minutes)
Example: INCR rate:otp:+94771234567
         EXPIRE rate:otp:+94771234567 600
Logic:   Reject if GET returns >= 3
```

### General Rate Limiting
```
Key:     rate:{endpoint}:{identifier}
Type:    String (counter)
TTL:     Varies by endpoint (see rate limiting strategy in backend doc)
Example: INCR rate:api:user-uuid-here
         EXPIRE rate:api:user-uuid-here 60
```

### Leaderboard - Grade Level
```
Key:     leaderboard:grade:{gradeId}:week:{weekStart}
Type:    Sorted Set
Member:  userId (UUID string)
Score:   Points earned this week (float)
Example: ZINCRBY leaderboard:grade:abc123:week:2026-04-20 15.0 user-uuid
         ZREVRANK leaderboard:grade:abc123:week:2026-04-20 user-uuid
         ZREVRANGE leaderboard:grade:abc123:week:2026-04-20 0 49 WITHSCORES
```

### Leaderboard - Subject Level
```
Key:     leaderboard:subject:{subjectId}:week:{weekStart}
Type:    Sorted Set
Member:  userId (UUID string)
Score:   Points earned this week for this subject
Example: Same operations as grade leaderboard
```

---

## Migration Strategy

### Tool: golang-migrate

Standard Go migration tool. Migrations are plain SQL files stored in version control.

**Workflow:**
1. Developer writes SQL migration files:
   - `db/migrations/000001_init_schema.up.sql` (apply)
   - `db/migrations/000001_init_schema.down.sql` (rollback)
2. Run `migrate -path db/migrations -database $DATABASE_URL up` to apply
3. In production: migrations run automatically during deployment (before new API container starts)

**Migration naming convention:** `{number}_{description}.{up|down}.sql`

**Migration files are version-controlled** in the git repository.

### SQL Queries (sqlc)

All database queries are written as plain SQL in `db/queries/` and compiled to type-safe Go code:

```
db/queries/users.sql     → internal/repository/users.sql.go
db/queries/cards.sql     → internal/repository/cards.sql.go
db/queries/reviews.sql   → internal/repository/reviews.sql.go
```

Run `sqlc generate` to regenerate Go code after modifying SQL queries.

### Seed Data Script
A seed script (`db/seed/seed.go`) populates initial data:
- 4 grades (10, 11, 12, 13)
- O/L compulsory and optional subjects (per grade)
- A/L stream subjects (per grade and stream)
- 3 terms per subject
- Sample units and cards for testing

Run with: `go run db/seed/seed.go`

---

## Backup Strategy

### PostgreSQL
- **Daily backup**: `pg_dump` cron job at 02:00 IST
- **Retention**: Keep 7 daily backups, 4 weekly backups
- **Storage**: Compressed backups stored on VPS + optionally synced to external storage
- **Recovery**: `pg_restore` from latest backup

### Redis
- **RDB snapshots**: Configured to save every 5 minutes if 100+ keys changed
- **AOF persistence**: Disabled (leaderboard data is reconstructable from PostgreSQL `user_scores`)
- **Recovery**: If Redis data is lost, leaderboard resets (acceptable) and OTP codes expire naturally

---

## Performance Considerations

1. **Connection pooling**: Use pgx's built-in connection pool (min: 2, max: 10 connections)
2. **Denormalized `card_count`**: Avoids COUNT(*) on cards table for unit listings
3. **Content `version` field**: Enables efficient cache invalidation without fetching all cards
4. **Composite indexes**: All frequently queried combinations have dedicated indexes
5. **JSONB for MCQ choices**: Avoids separate `choices` table, simpler queries, good enough for 4 choices per card
6. **`daily_activity` UPSERT**: Single query updates daily stats (no read-modify-write cycle)
7. **Leaderboard in Redis**: O(log n) rank operations, keeps load off PostgreSQL for high-frequency leaderboard queries
