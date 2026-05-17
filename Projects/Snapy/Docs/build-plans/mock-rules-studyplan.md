# Study Plan Service — Beeceptor Mock Rules

**Mock Server URL:** `https://studyplan-snapy.free.beeceptor.com`

**Free plan limit:** 3 custom rules. 7 total endpoints — 4 can be mocked client-side.

---

## Rule 1: GET /plans [selected]

Returns all study plans for the user. Powers the **Study Plans screen** showing plan cards with progress bars.

**Path:** `/plans`
**Method:** GET
**Status Code:** 200

```json
{
  "plans": [
    {
      "id": "sp100001-0000-0000-0000-000000000001",
      "name": "Term 1 Exam Prep",
      "goalType": "term_exam",
      "targetTermId": "t1000001-0000-0000-0000-000000000001",
      "deadline": "2026-08-15",
      "dailyMinutes": 30,
      "isActive": true,
      "createdAt": "2026-05-10T08:00:00Z",
      "progress": {
        "totalTasks": 42,
        "completedTasks": 18,
        "percentage": 43
      }
    },
    {
      "id": "sp100001-0000-0000-0000-000000000002",
      "name": "Catch Up — Algebra",
      "goalType": "catch_up",
      "targetTermId": null,
      "deadline": "2026-06-30",
      "dailyMinutes": 20,
      "isActive": false,
      "createdAt": "2026-04-20T14:30:00Z",
      "progress": {
        "totalTasks": 24,
        "completedTasks": 24,
        "percentage": 100
      }
    }
  ]
}
```

### Notes
- Frontend should send: `Authorization: Bearer <accessToken>` header.
- Only one plan can be `isActive: true` at a time. Creating a new plan deactivates the previous one.
- `progress.percentage` = `(completedTasks / totalTasks) * 100`, rounded.
- `goalType` values: `"term_exam"`, `"catch_up"`, `"custom"`.
- `"catch_up"` plans have `targetTermId: null` (they target weak areas across terms).
- Completed plans (`percentage: 100`) can be shown with a checkmark or moved to an archive section.
- `deadline` is `YYYY-MM-DD` format — use this to compute "X days remaining" on the UI.
- This response also serves as a substitute for `GET /plans/{planId}` — the frontend can find the plan by ID from this list without a separate API call.

---

## Rule 2: GET /plans/{planId}/today [selected]

Returns today's scheduled tasks. Powers the **Today's Study** screen — the main daily interaction.

**Path:** `/plans/sp100001-0000-0000-0000-000000000001/today`
**Method:** GET
**Status Code:** 200

```json
{
  "date": "2026-05-17",
  "tasks": [
    {
      "id": "t0001001-0000-0000-0000-000000000001",
      "planId": "sp100001-0000-0000-0000-000000000001",
      "unitId": "u1000001-0000-0000-0000-000000000001",
      "unitName": "Real Numbers",
      "scheduledDate": "2026-05-17",
      "taskType": "new",
      "estimatedMinutes": 10,
      "isCompleted": false,
      "completedAt": null
    },
    {
      "id": "t0001001-0000-0000-0000-000000000002",
      "planId": "sp100001-0000-0000-0000-000000000001",
      "unitId": "u1000001-0000-0000-0000-000000000003",
      "unitName": "Algebraic Expressions",
      "scheduledDate": "2026-05-17",
      "taskType": "new",
      "estimatedMinutes": 12,
      "isCompleted": false,
      "completedAt": null
    },
    {
      "id": "t0001001-0000-0000-0000-000000000003",
      "planId": "sp100001-0000-0000-0000-000000000001",
      "unitId": "u1000001-0000-0000-0000-000000000002",
      "unitName": "Indices & Logarithms",
      "scheduledDate": "2026-05-17",
      "taskType": "review",
      "estimatedMinutes": 8,
      "isCompleted": true,
      "completedAt": "2026-05-17T09:15:00Z"
    }
  ]
}
```

### Notes
- Path uses the mock plan UUID: `sp100001-0000-0000-0000-000000000001`
- `date` is today's date in `YYYY-MM-DD` — the frontend can compare with current date.
- `taskType` values:
  - `"new"` — studying unseen content (60% of daily budget)
  - `"review"` — reviewing previously studied content (40% of daily budget)
- One task is shown as already completed (`isCompleted: true`) to test the "done" state in the UI.
- `estimatedMinutes` tells the student how long each task should take.
- Total daily minutes for these 3 tasks = 30 min (matches the plan's `dailyMinutes`).
- Tapping a task should navigate to the unit's flashcard session (content service → review service flow).
- After completing a task, the frontend calls `PATCH /tasks/{taskId}` (mocked client-side).

---

## Rule 3: POST /plans [selected]

Creates a new study plan. Called from the **Create Plan** flow.

**Path:** `/plans`
**Method:** POST
**Status Code:** 201

```json
{
  "id": "sp100001-0000-0000-0000-000000000003",
  "name": "Mid-Year Review",
  "goalType": "term_exam",
  "targetTermId": "t1000001-0000-0000-0000-000000000002",
  "deadline": "2026-09-30",
  "dailyMinutes": 25,
  "isActive": true,
  "createdAt": "2026-05-17T16:00:00Z",
  "progress": {
    "totalTasks": 36,
    "completedTasks": 0,
    "percentage": 0
  }
}
```

### Notes
- Request body the frontend sends:
  ```json
  {
    "name": "Mid-Year Review",
    "goalType": "term_exam",
    "targetTermId": "t1000001-0000-0000-0000-000000000002",
    "deadline": "2026-09-30",
    "dailyMinutes": 25
  }
  ```
- Frontend should send: `Authorization: Bearer <accessToken>` header.
- The server generates all tasks immediately on plan creation. The response includes `progress` with `totalTasks` already populated.
- `goalType` validation: must be one of `"term_exam"`, `"catch_up"`, `"custom"`.
- `targetTermId` is required for `"term_exam"` plans, optional for `"catch_up"` and `"custom"`.
- `deadline` must be a future date (server validates). Format: `YYYY-MM-DD`.
- `dailyMinutes` range: 5–300 (server validates).
- When a new plan is created, any previously active plan is automatically deactivated (`isActive: false`).
- The mock echoes back a fresh plan. The real server would run the plan generator algorithm to distribute tasks across days.

---

## Skipped: PATCH /tasks/{taskId} — Mock client-side

Completes a task. Simple state toggle — mock it in the app.

### iOS (Swift)
```swift
func completeTask(taskId: String) async throws -> TaskResponse {
    #if MOCK
    try await Task.sleep(for: .seconds(0.5))
    return TaskResponse(
        id: taskId,
        planId: "sp100001-0000-0000-0000-000000000001",
        unitId: "u1000001-0000-0000-0000-000000000001",
        unitName: "Real Numbers",
        scheduledDate: "2026-05-17",
        taskType: "new",
        estimatedMinutes: 10,
        isCompleted: true,
        completedAt: ISO8601DateFormatter().string(from: Date())
    )
    #else
    return try await apiClient.request(.completeTask(taskId: taskId))
    #endif
}
```

### Android (Kotlin)
```kotlin
suspend fun completeTask(taskId: String): TaskResponse {
    if (BuildConfig.MOCK) {
        delay(500)
        return TaskResponse(
            id = taskId,
            planId = "sp100001-0000-0000-0000-000000000001",
            unitId = "u1000001-0000-0000-0000-000000000001",
            unitName = "Real Numbers",
            scheduledDate = LocalDate.now().toString(),
            taskType = "new",
            estimatedMinutes = 10,
            isCompleted = true,
            completedAt = Instant.now().toString()
        )
    }
    return apiClient.completeTask(taskId)
}
```

### Expected response from real API:
```json
{
  "id": "t0001001-0000-0000-0000-000000000001",
  "planId": "sp100001-0000-0000-0000-000000000001",
  "unitId": "u1000001-0000-0000-0000-000000000001",
  "unitName": "Real Numbers",
  "scheduledDate": "2026-05-17",
  "taskType": "new",
  "estimatedMinutes": 10,
  "isCompleted": true,
  "completedAt": "2026-05-17T09:30:00Z"
}
```

---

## Skipped: GET /plans/{planId} — Use list response

The plan details (including progress) are already included in the `GET /plans` list response (Rule 1). The frontend can filter by ID from the cached list instead of making a separate call.

If needed during development, the response shape is identical to a single item from the plans array:
```json
{
  "id": "sp100001-0000-0000-0000-000000000001",
  "name": "Term 1 Exam Prep",
  "goalType": "term_exam",
  "targetTermId": "t1000001-0000-0000-0000-000000000001",
  "deadline": "2026-08-15",
  "dailyMinutes": 30,
  "isActive": true,
  "createdAt": "2026-05-10T08:00:00Z",
  "progress": {
    "totalTasks": 42,
    "completedTasks": 18,
    "percentage": 43
  }
}
```

---

## Skipped: DELETE /plans/{planId} — Mock client-side

Deactivates a plan (soft delete — sets `isActive: false`). No complex response needed.

### Expected response:
```json
{
  "id": "sp100001-0000-0000-0000-000000000001",
  "name": "Term 1 Exam Prep",
  "goalType": "term_exam",
  "targetTermId": "t1000001-0000-0000-0000-000000000001",
  "deadline": "2026-08-15",
  "dailyMinutes": 30,
  "isActive": false,
  "createdAt": "2026-05-10T08:00:00Z",
  "progress": {
    "totalTasks": 42,
    "completedTasks": 18,
    "percentage": 43
  }
}
```

Mock client-side: just update the local state to set `isActive: false`.

---

## Skipped: GET /health — Mock client-side

Trivial health check. Returns `{ "status": "ok", "database": "connected" }`. Same as all other services.

---

## Testing the Full Study Plan Flow

```
┌──────────────────────────────────────────────────────────┐
│                 STUDY PLAN FLOW                           │
├──────────────────────────────────────────────────────────┤
│                                                          │
│  1. Study Plans Screen                                   │
│     → GET /plans  ←── Rule 1                            │
│     → Display plan cards with progress bars              │
│       ├─ "Term 1 Exam Prep" — 43% complete, active      │
│       └─ "Catch Up — Algebra" — 100% complete, inactive  │
│     → User taps "Create New Plan"                        │
│                                                          │
│  2. Create Plan Flow                                     │
│     → Step 1: Name → "Mid-Year Review"                   │
│     → Step 2: Goal type → "Term Exam"                    │
│     → Step 3: Select term → "Term 2"                     │
│     → Step 4: Deadline → "2026-09-30"                    │
│     → Step 5: Daily minutes → 25                         │
│     → POST /plans  ←── Rule 3                           │
│     → Previous active plan auto-deactivated              │
│     → Navigate to Today's Tasks                          │
│                                                          │
│  3. Today's Tasks Screen                                 │
│     → GET /plans/{planId}/today  ←── Rule 2             │
│     → Display tasks for today:                           │
│       ☐ Real Numbers (new) — 10 min                      │
│       ☐ Algebraic Expressions (new) — 12 min             │
│       ☑ Indices & Logarithms (review) — 8 min ✓         │
│     → Total: 30 min today                                │
│                                                          │
│  4. Start Task                                           │
│     → User taps "Real Numbers"                           │
│     → Navigate to unit study (content service)           │
│     → Study cards, submit reviews (review service)       │
│     → Return to tasks screen                             │
│                                                          │
│  5. Complete Task                                        │
│     → User taps "Done" on "Real Numbers"                 │
│     → PATCH /tasks/{taskId}  ←── Mock client-side       │
│     → Update local state: isCompleted = true             │
│     → Animate checkbox/progress update                   │
│                                                          │
│  6. Progress Update                                      │
│     → Recalculate from GET /plans list                   │
│     → Progress bar updates: 43% → 45%                   │
│                                                          │
└──────────────────────────────────────────────────────────┘
```

---

## Mock UUIDs Reference

Consistent IDs used across all mock responses:

| Entity | UUID | Description |
|--------|------|-------------|
| **Study Plans** | | |
| Term 1 Exam Prep | `sp100001-0000-0000-0000-000000000001` | Active plan |
| Catch Up — Algebra | `sp100001-0000-0000-0000-000000000002` | Completed plan |
| Mid-Year Review | `sp100001-0000-0000-0000-000000000003` | Created in Rule 3 |
| **Tasks** | | |
| Task — Real Numbers (new) | `t0001001-0000-0000-0000-000000000001` | Pending |
| Task — Algebra (new) | `t0001001-0000-0000-0000-000000000002` | Pending |
| Task — Indices (review) | `t0001001-0000-0000-0000-000000000003` | Completed |
| **Terms** | | |
| Term 1 | `t1000001-0000-0000-0000-000000000001` | From content mock |
| Term 2 | `t1000001-0000-0000-0000-000000000002` | From content mock |
| **Units** | | |
| Real Numbers | `u1000001-0000-0000-0000-000000000001` | From content mock |
| Indices & Logarithms | `u1000001-0000-0000-0000-000000000002` | From content mock |
| Algebraic Expressions | `u1000001-0000-0000-0000-000000000003` | From content mock |

---

## Goal Types Reference (for UI development)

| goalType | Label | Description | targetTermId |
|----------|-------|-------------|-------------|
| `term_exam` | "Term Exam Prep" | Study all units in a term for an upcoming exam | **Required** |
| `catch_up` | "Catch Up" | Focus on weak/overdue areas across subjects | Optional (null) |
| `custom` | "Custom Plan" | User-defined focus areas | Optional |

## Task Types Reference

| taskType | Label | Color Hint | Description |
|----------|-------|-----------|-------------|
| `new` | "New" | Blue/primary | Studying unseen content |
| `review` | "Review" | Orange/secondary | Reviewing previously studied material |
