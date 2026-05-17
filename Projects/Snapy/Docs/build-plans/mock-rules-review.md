# Review Service — Beeceptor Mock Rules

**Mock Server URL:** `https://snapy-review.free.beeceptor.com`

**Free plan limit:** 3 custom rules. All 3 review service endpoints fit within the limit.

---

## Rule 1: GET /due [selected]

Returns due cards for Quick Review. Powers the **Quick Review session** and the **home screen due badge**.

**Path:** `/due`
**Method:** GET
**Status Code:** 200

```json
{
  "dueCards": [
    {
      "cardId": "c1000001-0000-0000-0000-000000000001",
      "unitId": "u1000001-0000-0000-0000-000000000001",
      "type": "classic",
      "question": "What is the square root of 144?",
      "answer": "12",
      "choices": null,
      "explanation": null,
      "unitName": "Real Numbers",
      "srsState": {
        "stability": 2.40,
        "difficulty": 5.0,
        "elapsedDays": 3,
        "scheduledDays": 2,
        "state": "learning",
        "dueAt": "2026-05-13T10:30:00Z",
        "reps": 1,
        "lapses": 0
      }
    },
    {
      "cardId": "c1000001-0000-0000-0000-000000000002",
      "unitId": "u1000001-0000-0000-0000-000000000001",
      "type": "mcq",
      "question": "Which of the following is an irrational number?",
      "answer": null,
      "choices": [
        { "text": "3.14", "isCorrect": false },
        { "text": "√2", "isCorrect": true },
        { "text": "22/7", "isCorrect": false },
        { "text": "0.5", "isCorrect": false }
      ],
      "explanation": "√2 cannot be expressed as a fraction of two integers, making it irrational.",
      "unitName": "Real Numbers",
      "srsState": {
        "stability": 18.50,
        "difficulty": 4.2,
        "elapsedDays": 21,
        "scheduledDays": 19,
        "state": "review",
        "dueAt": "2026-05-14T08:00:00Z",
        "reps": 5,
        "lapses": 0
      }
    },
    {
      "cardId": "c1000001-0000-0000-0000-000000000005",
      "unitId": "u1000001-0000-0000-0000-000000000001",
      "type": "classic",
      "question": "Convert 0.75 to a fraction in simplest form.",
      "answer": "3/4",
      "choices": null,
      "explanation": null,
      "unitName": "Real Numbers",
      "srsState": {
        "stability": 1.20,
        "difficulty": 6.8,
        "elapsedDays": 1,
        "scheduledDays": 1,
        "state": "relearning",
        "dueAt": "2026-05-15T14:00:00Z",
        "reps": 4,
        "lapses": 1
      }
    },
    {
      "cardId": "c1000001-0000-0000-0000-000000000004",
      "unitId": "u1000001-0000-0000-0000-000000000001",
      "type": "mcq",
      "question": "Which number is a rational number?",
      "answer": null,
      "choices": [
        { "text": "π", "isCorrect": false },
        { "text": "√5", "isCorrect": false },
        { "text": "-7/3", "isCorrect": true },
        { "text": "√3", "isCorrect": false }
      ],
      "explanation": "A rational number can be written as p/q where p and q are integers. -7/3 fits this definition.",
      "unitName": "Real Numbers",
      "srsState": {
        "stability": 6.80,
        "difficulty": 5.5,
        "elapsedDays": 8,
        "scheduledDays": 7,
        "state": "review",
        "dueAt": "2026-05-12T09:00:00Z",
        "reps": 3,
        "lapses": 0
      }
    },
    {
      "cardId": "c1000001-0000-0000-0000-000000000003",
      "unitId": "u1000001-0000-0000-0000-000000000001",
      "type": "classic",
      "question": "Is 0.333... a rational number? Explain why.",
      "answer": "Yes, because 0.333... = 1/3, which is a ratio of two integers.",
      "choices": null,
      "explanation": null,
      "unitName": "Real Numbers",
      "srsState": {
        "stability": 3.50,
        "difficulty": 4.8,
        "elapsedDays": 4,
        "scheduledDays": 4,
        "state": "learning",
        "dueAt": "2026-05-15T10:30:00Z",
        "reps": 2,
        "lapses": 0
      }
    }
  ],
  "total": 5
}
```

### Notes
- Cards are sorted by `dueAt` ASC (most overdue first) — the order the student should review them.
- Includes a mix of all SRS states to test UI variations:
  - `"learning"` — card being initially learned (low stability, few reps)
  - `"review"` — card in long-term cycle (high stability, many reps)
  - `"relearning"` — card that was forgotten (lapses > 0, low stability)
- Includes both `classic` and `mcq` card types.
- The `total` field powers the home screen badge (e.g., "5 cards due").
- The `?limit=20` query param is the same route — Beeceptor can't differentiate by query string, so this single rule covers both `GET /due` and `GET /due?limit=N`.
- Frontend should send: `Authorization: Bearer <accessToken>` header.
- `srsState` is included so the frontend can optionally show metadata (e.g., "Reviewing for the 5th time" or a difficulty indicator).

---

## Rule 2: POST /batch [selected]

Submits a batch of review results. Called after completing a **study session** or **Quick Review session**.

**Path:** `/batch`
**Method:** POST
**Status Code:** 200

```json
{
  "accepted": 3,
  "rejected": 0
}
```

### Notes
- This is called when the user finishes a study session. The frontend sends all card ratings at once.
- Request body the frontend sends:
  ```json
  {
    "reviews": [
      {
        "cardId": "c1000001-0000-0000-0000-000000000001",
        "rating": 3,
        "reviewedAt": "2026-05-16T10:30:00Z"
      },
      {
        "cardId": "c1000001-0000-0000-0000-000000000002",
        "rating": 1,
        "reviewedAt": "2026-05-16T10:31:00Z"
      },
      {
        "cardId": "c1000001-0000-0000-0000-000000000005",
        "rating": 3,
        "reviewedAt": "2026-05-16T10:32:00Z"
      }
    ]
  }
  ```
- **Rating values:**
  - `1` = Again (missed it — classic "Missed it", or wrong MCQ answer)
  - `2` = Hard (not used in Snapy's simplified mode)
  - `3` = Good (got it — classic "Got it", or correct MCQ answer)
  - `4` = Easy (not used in Snapy's simplified mode)
- The mock always returns `accepted` matching the batch size and `rejected: 0`. In production, `rejected` would be > 0 if a card doesn't exist.
- Frontend should send: `Authorization: Bearer <accessToken>` header.
- Batch size is capped at 100 reviews (frontend should validate before sending).
- The response is intentionally minimal — the server handles all FSRS computation. No SRS state is returned.

---

## Rule 3: GET /health [selected]

Health check endpoint. Not needed for frontend development but useful for verifying the mock server is reachable.

**Path:** `/health`
**Method:** GET
**Status Code:** 200

```json
{
  "status": "ok",
  "database": "connected"
}
```

### Notes
- This is a standard health check. Use it to verify Beeceptor connectivity during development.
- In production, this is used by Traefik/Coolify for health checks.

---

## Testing the Full Review Flow

```
┌──────────────────────────────────────────────────────────┐
│                    REVIEW FLOW                            │
├──────────────────────────────────────────────────────────┤
│                                                          │
│  1. Home Screen                                          │
│     → GET /due  ←── Rule 1                              │
│     → Display badge: "5 cards due" (from total field)    │
│     → User taps "Quick Review"                           │
│                                                          │
│  2. Quick Review Session                                 │
│     → GET /due  ←── Rule 1                              │
│     → Present cards one by one:                          │
│       Card 1: Classic (learning) → "What is √144?"      │
│         → "Got it" (rating 3) or "Missed it" (rating 1) │
│       Card 2: MCQ (review) → "Which is irrational?"      │
│         → Select answer → correct (3) or wrong (1)      │
│       Card 3: Classic (relearning) → "Convert 0.75"     │
│         → "Got it" or "Missed it"                        │
│       Card 4: MCQ (review) → "Which is rational?"        │
│         → Select answer                                  │
│       Card 5: Classic (learning) → "Is 0.333 rational?"  │
│         → "Got it" or "Missed it"                        │
│     → Queue ratings locally: [{cardId, rating, timestamp}]│
│                                                          │
│  3. Session Summary                                      │
│     → Show: 5 cards reviewed, 4 correct, 1 wrong        │
│     → POST /batch  ←── Rule 2                           │
│       Body: { "reviews": [...all 5 ratings...] }          │
│       Response: { "accepted": 5, "rejected": 0 }         │
│     → Navigate back to Home Screen                        │
│                                                          │
│  4. Unit Study Session (alternative flow)                 │
│     → User taps "Real Numbers" unit                      │
│     → GET /units/{unitId}/cards  ←── Content Service     │
│     → Study all 8 cards (regardless of SRS state)        │
│     → Queue ratings locally                              │
│     → POST /batch  ←── Rule 2                           │
│     → Show session summary                               │
│                                                          │
└──────────────────────────────────────────────────────────┘
```

---

## Mock UUIDs Reference

Consistent IDs used across all mock responses (same as Content Service):

| Entity | UUID | Description |
|--------|------|-------------|
| **User** | | |
| Test Student | `a1b2c3d4-e5f6-7890-abcd-ef1234567890` | From auth mock |
| **Units** | | |
| Real Numbers | `u1000001-0000-0000-0000-000000000001` | Term 1 Mathematics |
| **Cards (Real Numbers)** | | |
| Card 1 | `c1000001-0000-0000-0000-000000000001` | Classic: √144 |
| Card 2 | `c1000001-0000-0000-0000-000000000002` | MCQ: irrational number |
| Card 3 | `c1000001-0000-0000-0000-000000000003` | Classic: 0.333 rational |
| Card 4 | `c1000001-0000-0000-0000-000000000004` | MCQ: rational number |
| Card 5 | `c1000001-0000-0000-0000-000000000005` | Classic: 0.75 to fraction |

---

## SRS State Reference (for UI development)

These are the 4 possible `state` values in `srsState`:

| State | Meaning | UI Hint |
|-------|---------|---------|
| `new` | Never studied | Could show "New" badge |
| `learning` | Initially learning (1-2 reviews) | Could show "Learning" badge |
| `review` | In long-term review cycle | Normal display |
| `relearning` | Forgot and relearning | Could highlight with warning color |

### Rating → State Transitions (for understanding the flow)

| Current State | Rating 1 (Again) | Rating 3 (Good) |
|---|---|---|
| `new` | → `learning` | → `learning` |
| `learning` | → `learning` (stays) | → `review` |
| `review` | → `relearning` (lapse) | → `review` (interval grows) |
| `relearning` | → `relearning` (stays) | → `review` |
