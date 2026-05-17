# Analytics Service — Beeceptor Mock Rules

**Mock Server URL:** `https://analytics-snapy.free.beeceptor.com`

**Free plan limit:** 3 custom rules. 5 total endpoints — 2 can be mocked client-side.

---

## Rule 1: GET /summary [selected]

Returns dashboard summary stats. Powers the **Home Screen** stats cards and the **Profile/Stats** header.

**Path:** `/summary`
**Method:** GET
**Status Code:** 200

```json
{
  "totalReviews": 342,
  "cardsMastered": 87,
  "cardsStudied": 156,
  "currentStreak": 7
}
```

### Notes
- Frontend should send: `Authorization: Bearer <accessToken>` header.
- All fields are integers — no decimals.
- **Field meanings:**
  - `totalReviews` — total number of card reviews ever submitted (from `card_reviews` table)
  - `cardsMastered` — cards in `"review"` SRS state (long-term retention achieved)
  - `cardsStudied` — total cards with any SRS state (has been seen at least once)
  - `currentStreak` — consecutive days of studying
- **Derived stats the frontend can compute:**
  - Mastery percentage: `cardsMastered / cardsStudied * 100` → 87/156 = 55.8%
  - Accuracy not included here (use `correctCount / cardsReviewed` from heatmap data)
- **For a new user** (no reviews yet), the response would be:
  ```json
  {
    "totalReviews": 0,
    "cardsMastered": 0,
    "cardsStudied": 0,
    "currentStreak": 0
  }
  ```
- The home screen can show these as stat cards: "342 Reviews", "87 Mastered", "7 Day Streak 🔥".
- To toggle between "new user" and "active user" states during development, edit the JSON in Beeceptor.

---

## Rule 2: GET /streak [selected]

Returns streak data with a 365-day activity heatmap. Powers the **Stats/Profile** screen streak display and heatmap calendar.

**Path:** `/streak`
**Method:** GET
**Status Code:** 200

```json
{
  "currentStreak": 7,
  "longestStreak": 23,
  "heatmap": [
    { "date": "2025-06-12", "sessionsCount": 1, "cardsReviewed": 15, "correctCount": 12, "studyMinutes": 10 },
    { "date": "2025-06-15", "sessionsCount": 2, "cardsReviewed": 30, "correctCount": 24, "studyMinutes": 22 },
    { "date": "2025-06-16", "sessionsCount": 1, "cardsReviewed": 20, "correctCount": 16, "studyMinutes": 15 },
    { "date": "2025-07-02", "sessionsCount": 1, "cardsReviewed": 10, "correctCount": 8, "studyMinutes": 8 },
    { "date": "2025-07-03", "sessionsCount": 1, "cardsReviewed": 25, "correctCount": 20, "studyMinutes": 18 },
    { "date": "2025-07-04", "sessionsCount": 1, "cardsReviewed": 18, "correctCount": 14, "studyMinutes": 12 },
    { "date": "2025-07-05", "sessionsCount": 1, "cardsReviewed": 22, "correctCount": 19, "studyMinutes": 16 },
    { "date": "2025-07-06", "sessionsCount": 1, "cardsReviewed": 12, "correctCount": 10, "studyMinutes": 9 },
    { "date": "2025-07-07", "sessionsCount": 1, "cardsReviewed": 20, "correctCount": 15, "studyMinutes": 14 },
    { "date": "2025-07-08", "sessionsCount": 1, "cardsReviewed": 16, "correctCount": 13, "studyMinutes": 11 },
    { "date": "2025-07-09", "sessionsCount": 1, "cardsReviewed": 24, "correctCount": 20, "studyMinutes": 17 },
    { "date": "2025-07-10", "sessionsCount": 1, "cardsReviewed": 18, "correctCount": 14, "studyMinutes": 13 },
    { "date": "2025-07-11", "sessionsCount": 1, "cardsReviewed": 22, "correctCount": 18, "studyMinutes": 15 },
    { "date": "2025-07-12", "sessionsCount": 1, "cardsReviewed": 14, "correctCount": 11, "studyMinutes": 10 },
    { "date": "2025-07-13", "sessionsCount": 1, "cardsReviewed": 20, "correctCount": 17, "studyMinutes": 14 },
    { "date": "2025-07-14", "sessionsCount": 1, "cardsReviewed": 26, "correctCount": 22, "studyMinutes": 19 },
    { "date": "2025-07-15", "sessionsCount": 1, "cardsReviewed": 18, "correctCount": 15, "studyMinutes": 13 },
    { "date": "2025-07-16", "sessionsCount": 1, "cardsReviewed": 22, "correctCount": 19, "studyMinutes": 16 },
    { "date": "2025-07-17", "sessionsCount": 1, "cardsReviewed": 15, "correctCount": 12, "studyMinutes": 11 },
    { "date": "2025-07-18", "sessionsCount": 1, "cardsReviewed": 20, "correctCount": 16, "studyMinutes": 14 },
    { "date": "2025-07-19", "sessionsCount": 1, "cardsReviewed": 28, "correctCount": 24, "studyMinutes": 20 },
    { "date": "2025-07-20", "sessionsCount": 1, "cardsReviewed": 16, "correctCount": 13, "studyMinutes": 12 },
    { "date": "2025-07-21", "sessionsCount": 1, "cardsReviewed": 24, "correctCount": 20, "studyMinutes": 17 },
    { "date": "2025-07-22", "sessionsCount": 1, "cardsReviewed": 20, "correctCount": 16, "studyMinutes": 14 },
    { "date": "2025-07-23", "sessionsCount": 1, "cardsReviewed": 18, "correctCount": 15, "studyMinutes": 13 },
    { "date": "2025-08-10", "sessionsCount": 2, "cardsReviewed": 35, "correctCount": 28, "studyMinutes": 25 },
    { "date": "2025-08-11", "sessionsCount": 1, "cardsReviewed": 20, "correctCount": 17, "studyMinutes": 14 },
    { "date": "2025-08-12", "sessionsCount": 1, "cardsReviewed": 22, "correctCount": 18, "studyMinutes": 16 },
    { "date": "2025-09-05", "sessionsCount": 1, "cardsReviewed": 18, "correctCount": 14, "studyMinutes": 12 },
    { "date": "2025-09-06", "sessionsCount": 1, "cardsReviewed": 24, "correctCount": 20, "studyMinutes": 17 },
    { "date": "2025-09-07", "sessionsCount": 1, "cardsReviewed": 16, "correctCount": 13, "studyMinutes": 11 },
    { "date": "2025-09-08", "sessionsCount": 1, "cardsReviewed": 20, "correctCount": 16, "studyMinutes": 14 },
    { "date": "2025-10-15", "sessionsCount": 1, "cardsReviewed": 22, "correctCount": 18, "studyMinutes": 15 },
    { "date": "2025-11-20", "sessionsCount": 1, "cardsReviewed": 14, "correctCount": 11, "studyMinutes": 10 },
    { "date": "2025-11-21", "sessionsCount": 1, "cardsReviewed": 20, "correctCount": 16, "studyMinutes": 14 },
    { "date": "2025-11-22", "sessionsCount": 1, "cardsReviewed": 18, "correctCount": 15, "studyMinutes": 13 },
    { "date": "2025-12-10", "sessionsCount": 1, "cardsReviewed": 25, "correctCount": 21, "studyMinutes": 18 },
    { "date": "2026-01-05", "sessionsCount": 1, "cardsReviewed": 20, "correctCount": 16, "studyMinutes": 14 },
    { "date": "2026-01-06", "sessionsCount": 1, "cardsReviewed": 22, "correctCount": 18, "studyMinutes": 15 },
    { "date": "2026-01-07", "sessionsCount": 1, "cardsReviewed": 18, "correctCount": 15, "studyMinutes": 13 },
    { "date": "2026-02-14", "sessionsCount": 2, "cardsReviewed": 32, "correctCount": 26, "studyMinutes": 23 },
    { "date": "2026-02-15", "sessionsCount": 1, "cardsReviewed": 20, "correctCount": 16, "studyMinutes": 14 },
    { "date": "2026-03-01", "sessionsCount": 1, "cardsReviewed": 16, "correctCount": 13, "studyMinutes": 11 },
    { "date": "2026-03-02", "sessionsCount": 1, "cardsReviewed": 22, "correctCount": 18, "studyMinutes": 16 },
    { "date": "2026-03-03", "sessionsCount": 1, "cardsReviewed": 18, "correctCount": 15, "studyMinutes": 13 },
    { "date": "2026-04-20", "sessionsCount": 1, "cardsReviewed": 24, "correctCount": 20, "studyMinutes": 17 },
    { "date": "2026-04-21", "sessionsCount": 1, "cardsReviewed": 20, "correctCount": 17, "studyMinutes": 14 },
    { "date": "2026-04-22", "sessionsCount": 1, "cardsReviewed": 28, "correctCount": 23, "studyMinutes": 20 },
    { "date": "2026-04-23", "sessionsCount": 1, "cardsReviewed": 16, "correctCount": 13, "studyMinutes": 11 },
    { "date": "2026-04-24", "sessionsCount": 1, "cardsReviewed": 22, "correctCount": 18, "studyMinutes": 16 },
    { "date": "2026-04-25", "sessionsCount": 1, "cardsReviewed": 18, "correctCount": 15, "studyMinutes": 13 },
    { "date": "2026-05-11", "sessionsCount": 1, "cardsReviewed": 20, "correctCount": 16, "studyMinutes": 14 },
    { "date": "2026-05-12", "sessionsCount": 1, "cardsReviewed": 24, "correctCount": 20, "studyMinutes": 17 },
    { "date": "2026-05-13", "sessionsCount": 1, "cardsReviewed": 18, "correctCount": 15, "studyMinutes": 13 },
    { "date": "2026-05-14", "sessionsCount": 1, "cardsReviewed": 22, "correctCount": 18, "studyMinutes": 16 },
    { "date": "2026-05-15", "sessionsCount": 1, "cardsReviewed": 26, "correctCount": 22, "studyMinutes": 19 },
    { "date": "2026-05-16", "sessionsCount": 1, "cardsReviewed": 20, "correctCount": 17, "studyMinutes": 14 },
    { "date": "2026-05-17", "sessionsCount": 1, "cardsReviewed": 18, "correctCount": 15, "studyMinutes": 13 }
  ]
}
```

### Notes
- Frontend should send: `Authorization: Bearer <accessToken>` header.
- **Heatmap is sparse** — only includes days with actual activity. Days with no study don't appear in the array.
- The heatmap covers the last 365 days. This mock has ~55 active days out of 365.
- **Heatmap visualization:** Map `cardsReviewed` or `studyMinutes` to color intensity:
  - No entry for a date → gray/empty (didn't study)
  - `studyMinutes < 10` → light green
  - `studyMinutes 10-15` → medium green
  - `studyMinutes > 15` → dark green
- The mock includes a **23-day streak in July 2025** (July 2–24, with some gaps — actually it's a solid run from July 2–24 = 23 consecutive days). This is the `longestStreak`.
- The **current streak is 7** — the mock has consecutive entries from May 11–17 (7 days).
- `sessionsCount` can be > 1 if the student studied multiple times in a day (e.g., morning + evening).
- `correctCount / cardsReviewed` gives accuracy for that day (e.g., May 17: 15/18 = 83%).

---

## Rule 3: GET /weak-areas [selected]

Returns units with the lowest mastery. Powers the **Weak Areas** section on the Stats screen and the home screen "Focus on these" suggestions.

**Path:** `/weak-areas`
**Method:** GET
**Status Code:** 200

```json
{
  "weakAreas": [
    {
      "subjectId": "s1000001-0000-0000-0000-000000000001",
      "subjectName": "Mathematics",
      "unitId": "u1000001-0000-0000-0000-000000000004",
      "unitName": "Simultaneous Equations",
      "totalCards": 16,
      "masteryRatio": 0.13,
      "totalLapses": 12,
      "overdueCards": 8
    },
    {
      "subjectId": "s1000001-0000-0000-0000-000000000001",
      "subjectName": "Mathematics",
      "unitId": "u1000001-0000-0000-0000-000000000005",
      "unitName": "Triangles & Geometry",
      "totalCards": 24,
      "masteryRatio": 0.25,
      "totalLapses": 9,
      "overdueCards": 5
    },
    {
      "subjectId": "s1000001-0000-0000-0000-000000000001",
      "subjectName": "Mathematics",
      "unitId": "u1000001-0000-0000-0000-000000000002",
      "unitName": "Indices & Logarithms",
      "totalCards": 18,
      "masteryRatio": 0.33,
      "totalLapses": 6,
      "overdueCards": 3
    },
    {
      "subjectId": "s1000001-0000-0000-0000-000000000002",
      "subjectName": "Science",
      "unitId": "u1000002-0000-0000-0000-000000000003",
      "unitName": "Chemical Reactions",
      "totalCards": 20,
      "masteryRatio": 0.35,
      "totalLapses": 7,
      "overdueCards": 4
    },
    {
      "subjectId": "s1000001-0000-0000-0000-000000000003",
      "subjectName": "English",
      "unitId": "u1000003-0000-0000-0000-000000000002",
      "unitName": "Essay Writing",
      "totalCards": 14,
      "masteryRatio": 0.43,
      "totalLapses": 4,
      "overdueCards": 2
    }
  ]
}
```

### Notes
- Frontend should send: `Authorization: Bearer <accessToken>` header.
- Sorted by `masteryRatio` ASC (lowest mastery first) — the weakest areas appear at the top.
- `?limit=N` query param works on the same route (Beeceptor can't differentiate by query string, so this rule covers both).
- **Field meanings:**
  - `masteryRatio` — fraction of cards in `"review"` SRS state (0.0 = nothing mastered, 1.0 = all mastered)
  - `totalLapses` — how many times cards in this unit were rated "Again" (forgot)
  - `overdueCards` — cards past their scheduled review date
- **UI suggestions:**
  - Show a warning/red indicator for units with `masteryRatio < 0.3`
  - Show "8 overdue" badge for overdue cards
  - Tapping a weak area should navigate to that unit's flashcards
- The default limit is 10 (the mock returns 5 — adjust in Beeceptor to test pagination).
- Cross-subject weak areas help students prioritize across all their subjects.

---

## Skipped: POST /activity — Mock client-side

Records study session activity after each session. Fire-and-forget — returns 204 No Content.

### Why skippable
This is a write-only endpoint. The frontend sends data and gets no response body. Mock it by simply not making the API call in mock mode, or by returning success immediately.

### iOS (Swift)
```swift
func recordActivity(
    sessionsCount: Int,
    cardsReviewed: Int,
    correctCount: Int,
    studyMinutes: Int
) async throws {
    #if MOCK
    try await Task.sleep(for: .milliseconds(200))
    return
    #else
    let _: EmptyResponse = try await apiClient.request(
        .recordActivity(
            sessionsCount: sessionsCount,
            cardsReviewed: cardsReviewed,
            correctCount: correctCount,
            studyMinutes: studyMinutes
        )
    )
    #endif
}
```

### Android (Kotlin)
```kotlin
suspend fun recordActivity(
    sessionsCount: Int,
    cardsReviewed: Int,
    correctCount: Int,
    studyMinutes: Int
) {
    if (BuildConfig.MOCK) {
        delay(200)
        return
    }
    apiClient.recordActivity(
        RecordActivityRequest(
            sessionsCount = sessionsCount,
            cardsReviewed = cardsReviewed,
            correctCount = correctCount,
            studyMinutes = studyMinutes
        )
    )
}
```

### Expected request/response:
```
POST /activity
Authorization: Bearer <accessToken>
Content-Type: application/json

{
  "sessionsCount": 1,
  "cardsReviewed": 20,
  "correctCount": 16,
  "studyMinutes": 15
}

→ 204 No Content
```

The server uses this data to:
1. Upsert into `daily_activity` (increments if called multiple times per day)
2. Update the user's streak (increment or reset based on last active date)

---

## Skipped: GET /health — Mock client-side

Trivial health check. Returns `{ "status": "ok", "database": "connected" }`. Same as all other services.

---

## Testing the Full Analytics Flow

```
┌──────────────────────────────────────────────────────────┐
│                  ANALYTICS FLOW                           │
├──────────────────────────────────────────────────────────┤
│                                                          │
│  1. Home Screen                                          │
│     → GET /summary  ←── Rule 1                          │
│     → Display stat cards:                                │
│       "342 Reviews"  "87 Mastered"  "7 Day Streak 🔥"   │
│     → Quick link: "5 weak areas →"                       │
│                                                          │
│  2. Study Session (Unit or Quick Review)                  │
│     → User studies cards, submits ratings                │
│     → POST /batch (review service)                       │
│     → POST /activity  ←── Mock client-side              │
│       { sessionsCount: 1, cardsReviewed: 8,              │
│         correctCount: 6, studyMinutes: 12 }              │
│     → Server updates streak + daily activity             │
│                                                          │
│  3. Stats / Profile Screen                               │
│     → GET /summary  ←── Rule 1                          │
│       Stats header: 342 reviews, 87 mastered             │
│     → GET /streak  ←── Rule 2                           │
│       Streak display: "7 day streak 🔥"                  │
│       Longest: "23 days"                                 │
│       Heatmap calendar (365 days, ~55 active days)       │
│         May: ■■□■■■■ (7 day streak visible)              │
│         Apr: □□■■■■■□□                                   │
│         Jul: ■■■■■■■■■■■■■■■■■■■■■■ (23 day streak)     │
│                                                          │
│  4. Weak Areas Section                                   │
│     → GET /weak-areas  ←── Rule 3                       │
│     → Display ranked list:                               │
│       🔴 Simultaneous Equations — 13% mastery, 12 lapses │
│       🔴 Triangles & Geometry — 25% mastery, 9 lapses   │
│       🟡 Indices & Logarithms — 33% mastery, 6 lapses   │
│       🟡 Chemical Reactions — 35% mastery, 7 lapses     │
│       🟢 Essay Writing — 43% mastery, 4 lapses          │
│     → Tap → navigate to unit flashcards                  │
│                                                          │
└──────────────────────────────────────────────────────────┘
```

---

## Mock UUIDs Reference

Consistent IDs used across all mock responses:

| Entity | UUID | Description |
|--------|------|-------------|
| **Subjects (Grade 11)** | | |
| Mathematics | `s1000001-0000-0000-0000-000000000001` | From content mock |
| Science | `s1000001-0000-0000-0000-000000000002` | From content mock |
| English | `s1000001-0000-0000-0000-000000000003` | From content mock |
| **Units** | | |
| Real Numbers | `u1000001-0000-0000-0000-000000000001` | Term 1, Mathematics |
| Indices & Logarithms | `u1000001-0000-0000-0000-000000000002` | Term 1, Mathematics |
| Algebraic Expressions | `u1000001-0000-0000-0000-000000000003` | Term 1, Mathematics |
| Simultaneous Equations | `u1000001-0000-0000-0000-000000000004` | Term 2, Mathematics |
| Triangles & Geometry | `u1000001-0000-0000-0000-000000000005` | Term 2, Mathematics |
| Chemical Reactions | `u1000002-0000-0000-0000-000000000003` | Science |
| Essay Writing | `u1000003-0000-0000-0000-000000000002` | English |

---

## Heatmap Color Mapping (for UI implementation)

Map `studyMinutes` per day to heatmap cell color:

| studyMinutes | Color | Level |
|---|---|---|
| No data (date not in array) | `#E5E7EB` (empty gray) | 0 |
| 1–9 | `#BBF7D0` (light green) | 1 |
| 10–15 | `#4ADE80` (medium green) | 2 |
| 16–20 | `#16A34A` (green) | 3 |
| 21+ | `#166534` (dark green) | 4 |

This is the same 5-level color scale used by GitHub's contribution graph.
