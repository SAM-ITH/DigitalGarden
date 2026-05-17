# Content Service — Beeceptor Mock Rules

**Mock Server URL:** `https://content-snapy.free.beeceptor.com`

**Free plan limit:** 3 custom rules. `GET /grades` is skipped — only 4 static records, hardcode client-side.

---

## Rule 1: GET /grades/{gradeId}/subjects

Returns the subject list for a grade. Powers the **Home Screen** subject grid.

**Path:** `/grades/c1d2e3f4-5b67-8901-abcd-ef1234567890/subjects`
**Method:** GET
**Status Code:** 200

```json
{
  "gradeId": "c1d2e3f4-5b67-8901-abcd-ef1234567890",
  "gradeName": "Grade 11",
  "subjects": [
    {
      "id": "s1000001-0000-0000-0000-000000000001",
      "name": "Mathematics",
      "nameSi": "ගණිතය",
      "nameTa": "கணிதம்",
      "icon": "calculator",
      "color": "#4F46E5",
      "isCompulsory": true,
      "stream": null,
      "displayOrder": 1
    },
    {
      "id": "s1000001-0000-0000-0000-000000000002",
      "name": "Science",
      "nameSi": "විද්‍යාව",
      "nameTa": "அறிவியல்",
      "icon": "flask",
      "color": "#059669",
      "isCompulsory": true,
      "stream": null,
      "displayOrder": 2
    },
    {
      "id": "s1000001-0000-0000-0000-000000000003",
      "name": "English",
      "nameSi": "English",
      "nameTa": "English",
      "icon": "book",
      "color": "#DC2626",
      "isCompulsory": true,
      "stream": null,
      "displayOrder": 3
    },
    {
      "id": "s1000001-0000-0000-0000-000000000004",
      "name": "History",
      "nameSi": "ඉතිහාසය",
      "nameTa": "வரலாறு",
      "icon": "landmark",
      "color": "#B45309",
      "isCompulsory": true,
      "stream": null,
      "displayOrder": 4
    },
    {
      "id": "s1000001-0000-0000-0000-000000000005",
      "name": "ICT",
      "nameSi": "සත්‍ය සහ සන්නිවේදන තාක්ෂණය",
      "nameTa": "தகவல் மற்றும் தகவல்தொடர்பு தொழில்நுட்பம்",
      "icon": "laptop",
      "color": "#7C3AED",
      "isCompulsory": false,
      "stream": null,
      "displayOrder": 5
    }
  ]
}
```

### Notes
- Path uses Grade 11 UUID: `c1d2e3f4-5b67-8901-abcd-ef1234567890`
- Each subject has `icon` and `color` for the home screen grid cards.
- `nameSi` / `nameTa` are for Sinhala/Tamil localization (Phase 5+).
- `isCompulsory` can be used to sort or visually separate subjects.
- `stream` is `null` for O/L subjects. A/L subjects would have `"science"`, `"commerce"`, etc.

---

## Rule 2: GET /subjects/{subjectId}/units

Returns terms and units for a subject. Powers the **Subject Detail Screen** with term tabs and unit lists.

**Path:** `/subjects/s1000001-0000-0000-0000-000000000001/units`
**Method:** GET
**Status Code:** 200

```json
{
  "subjectId": "s1000001-0000-0000-0000-000000000001",
  "subjectName": "Mathematics",
  "terms": [
    {
      "id": "t1000001-0000-0000-0000-000000000001",
      "name": "Term 1",
      "displayOrder": 1,
      "units": [
        {
          "id": "u1000001-0000-0000-0000-000000000001",
          "name": "Real Numbers",
          "description": "Rational and irrational numbers, number line",
          "cardCount": 20,
          "version": 1,
          "displayOrder": 1
        },
        {
          "id": "u1000001-0000-0000-0000-000000000002",
          "name": "Indices & Logarithms",
          "description": "Laws of indices, logarithmic expressions",
          "cardCount": 18,
          "version": 1,
          "displayOrder": 2
        },
        {
          "id": "u1000001-0000-0000-0000-000000000003",
          "name": "Algebraic Expressions",
          "description": "Simplifying and factorizing expressions",
          "cardCount": 22,
          "version": 1,
          "displayOrder": 3
        }
      ]
    },
    {
      "id": "t1000001-0000-0000-0000-000000000002",
      "name": "Term 2",
      "displayOrder": 2,
      "units": [
        {
          "id": "u1000001-0000-0000-0000-000000000004",
          "name": "Simultaneous Equations",
          "description": "Solving systems of linear equations",
          "cardCount": 16,
          "version": 1,
          "displayOrder": 1
        },
        {
          "id": "u1000001-0000-0000-0000-000000000005",
          "name": "Triangles & Geometry",
          "description": "Congruence, similarity, Pythagoras theorem",
          "cardCount": 24,
          "version": 1,
          "displayOrder": 2
        }
      ]
    },
    {
      "id": "t1000001-0000-0000-0000-000000000003",
      "name": "Term 3",
      "displayOrder": 3,
      "units": [
        {
          "id": "u1000001-0000-0000-0000-000000000006",
          "name": "Sets",
          "description": "Set operations, Venn diagrams",
          "cardCount": 15,
          "version": 1,
          "displayOrder": 1
        },
        {
          "id": "u1000001-0000-0000-0000-000000000007",
          "name": "Probability",
          "description": "Basic probability, tree diagrams",
          "cardCount": 14,
          "version": 1,
          "displayOrder": 2
        }
      ]
    }
  ]
}
```

### Notes
- Path uses Mathematics UUID: `s1000001-0000-0000-0000-000000000001`
- Response is nested: `terms → units` so the frontend can render term tabs.
- `cardCount` is denormalized — used for progress display (e.g., "12/20 cards reviewed").
- `version` is for cache invalidation — compare local cached version with server version.

---

## Rule 3: GET /units/{unitId}/cards

Returns all flashcards in a unit. Powers the **Flashcard Study Session**. Contains a mix of classic and MCQ cards.

**Path:** `/units/u1000001-0000-0000-0000-000000000001/cards`
**Method:** GET
**Status Code:** 200

```json
{
  "unitId": "u1000001-0000-0000-0000-000000000001",
  "unitName": "Real Numbers",
  "version": 1,
  "cards": [
    {
      "id": "c1000001-0000-0000-0000-000000000001",
      "type": "classic",
      "question": "What is the square root of 144?",
      "answer": "12",
      "choices": null,
      "explanation": null,
      "displayOrder": 1
    },
    {
      "id": "c1000001-0000-0000-0000-000000000002",
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
      "displayOrder": 2
    },
    {
      "id": "c1000001-0000-0000-0000-000000000003",
      "type": "classic",
      "question": "Is 0.333... a rational number? Explain why.",
      "answer": "Yes, because 0.333... = 1/3, which is a ratio of two integers.",
      "choices": null,
      "explanation": null,
      "displayOrder": 3
    },
    {
      "id": "c1000001-0000-0000-0000-000000000004",
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
      "displayOrder": 4
    },
    {
      "id": "c1000001-0000-0000-0000-000000000005",
      "type": "classic",
      "question": "Convert 0.75 to a fraction in simplest form.",
      "answer": "3/4",
      "choices": null,
      "explanation": null,
      "displayOrder": 5
    },
    {
      "id": "c1000001-0000-0000-0000-000000000006",
      "type": "mcq",
      "question": "Where does √7 lie on the number line?",
      "answer": null,
      "choices": [
        { "text": "Between 1 and 2", "isCorrect": false },
        { "text": "Between 2 and 3", "isCorrect": true },
        { "text": "Between 3 and 4", "isCorrect": false },
        { "text": "Between 4 and 5", "isCorrect": false }
      ],
      "explanation": "√7 ≈ 2.646, so it lies between 2 and 3.",
      "displayOrder": 6
    },
    {
      "id": "c1000001-0000-0000-0000-000000000007",
      "type": "classic",
      "question": "What is the value of √0?",
      "answer": "0",
      "choices": null,
      "explanation": null,
      "displayOrder": 7
    },
    {
      "id": "c1000001-0000-0000-0000-000000000008",
      "type": "mcq",
      "question": "Which statement about real numbers is correct?",
      "answer": null,
      "choices": [
        { "text": "All real numbers are rational", "isCorrect": false },
        { "text": "Irrational numbers include √4", "isCorrect": false },
        { "text": "Real numbers = rational + irrational", "isCorrect": true },
        { "text": "0 is an irrational number", "isCorrect": false }
      ],
      "explanation": "The set of real numbers is the union of rational and irrational numbers.",
      "displayOrder": 8
    }
  ]
}
```

### Notes
- Path uses "Real Numbers" unit UUID: `u1000001-0000-0000-0000-000000000001`
- Card types alternate between `classic` and `mcq` to test mixed mode sessions.
- **Classic cards**: have `answer` (non-null), `choices` is null.
- **MCQ cards**: have `choices` array (4 options with `isCorrect`), `answer` is null.
- `explanation` is shown after answering (especially for wrong MCQ answers).
- 8 cards included — enough to test a full session flow without being too long.

---

## Skipped: GET /grades — Mock client-side

Only 4 rows that never change. Hardcode in the app.

### iOS (Swift)
```swift
// Services/MockData.swift
#if MOCK
let mockGrades = [
    Grade(id: UUID(uuidString: "f1e2d3c4-b5a6-7890-1234-567890abcdef")!, name: "Grade 10", level: 10, educationStage: "ol", displayOrder: 1),
    Grade(id: UUID(uuidString: "c1d2e3f4-5b67-8901-abcd-ef1234567890")!, name: "Grade 11", level: 11, educationStage: "ol", displayOrder: 2),
    Grade(id: UUID(uuidString: "b2c3d4e5-6f78-9012-abcd-ef1234567890")!, name: "Grade 12", level: 12, educationStage: "al", displayOrder: 3),
    Grade(id: UUID(uuidString: "d3e4f5a6-7b89-0123-abcd-ef1234567890")!, name: "Grade 13", level: 13, educationStage: "al", displayOrder: 4)
]
#endif
```

### Android (Kotlin)
```kotlin
// data/mock/MockGrades.kt
val mockGrades = listOf(
    Grade(id = UUID.fromString("f1e2d3c4-b5a6-7890-1234-567890abcdef"), name = "Grade 10", level = 10, educationStage = "ol", displayOrder = 1),
    Grade(id = UUID.fromString("c1d2e3f4-5b67-8901-abcd-ef1234567890"), name = "Grade 11", level = 11, educationStage = "ol", displayOrder = 2),
    Grade(id = UUID.fromString("b2c3d4e5-6f78-9012-abcd-ef1234567890"), name = "Grade 12", level = 12, educationStage = "al", displayOrder = 3),
    Grade(id = UUID.fromString("d3e4f5a6-7b89-0123-abcd-ef1234567890"), name = "Grade 13", level = 13, educationStage = "al", displayOrder = 4)
)
```

---

## Testing the Full Content Browsing Flow

```
┌──────────────────────────────────────────────────────────┐
│                  CONTENT BROWSING FLOW                    │
├──────────────────────────────────────────────────────────┤
│                                                          │
│  1. Home Screen (after login/onboarding)                 │
│     → Load grades (mocked client-side)                   │
│     → Find user's gradeId from profile                   │
│     → GET /grades/{gradeId}/subjects  ←── Rule 1        │
│     → Display subject grid with icons + colors           │
│                                                          │
│  2. Subject Detail Screen                                │
│     → User taps "Mathematics"                            │
│     → GET /subjects/{subjectId}/units  ←── Rule 2       │
│     → Display term tabs (Term 1, Term 2, Term 3)        │
│     → List units under each term with card counts        │
│                                                          │
│  3. Flashcard Study Session                              │
│     → User taps "Real Numbers"                           │
│     → GET /units/{unitId}/cards  ←── Rule 3             │
│     → Cache cards locally                                │
│     → Begin mixed session:                               │
│       Card 1: Classic → "What is √144?" → flip → "12"   │
│       Card 2: MCQ → "Which is irrational?" → 4 options  │
│       Card 3: Classic → ...                              │
│       Card 4: MCQ → ...                                  │
│       ... through all 8 cards                            │
│     → Session Summary: 8 cards, accuracy, XP             │
│                                                          │
└──────────────────────────────────────────────────────────┘
```

---

## Mock UUIDs Reference

Consistent IDs used across all mock responses:

| Entity | UUID | Description |
|--------|------|-------------|
| **Grades** | | |
| Grade 10 | `f1e2d3c4-b5a6-7890-1234-567890abcdef` | O/L |
| Grade 11 | `c1d2e3f4-5b67-8901-abcd-ef1234567890` | O/L |
| Grade 12 | `b2c3d4e5-6f78-9012-abcd-ef1234567890` | A/L |
| Grade 13 | `d3e4f5a6-7b89-0123-abcd-ef1234567890` | A/L |
| **Subjects (Grade 11)** | | |
| Mathematics | `s1000001-0000-0000-0000-000000000001` | Compulsory |
| Science | `s1000001-0000-0000-0000-000000000002` | Compulsory |
| English | `s1000001-0000-0000-0000-000000000003` | Compulsory |
| History | `s1000001-0000-0000-0000-000000000004` | Compulsory |
| ICT | `s1000001-0000-0000-0000-000000000005` | Optional |
| **Terms (Mathematics)** | | |
| Term 1 | `t1000001-0000-0000-0000-000000000001` | Jan-Apr |
| Term 2 | `t1000001-0000-0000-0000-000000000002` | May-Aug |
| Term 3 | `t1000001-0000-0000-0000-000000000003` | Sep-Dec |
| **Units (Mathematics)** | | |
| Real Numbers | `u1000001-0000-0000-0000-000000000001` | Term 1 |
| Indices & Logarithms | `u1000001-0000-0000-0000-000000000002` | Term 1 |
| Algebraic Expressions | `u1000001-0000-0000-0000-000000000003` | Term 1 |
| Simultaneous Equations | `u1000001-0000-0000-0000-000000000004` | Term 2 |
| Triangles & Geometry | `u1000001-0000-0000-0000-000000000005` | Term 2 |
| Sets | `u1000001-0000-0000-0000-000000000006` | Term 3 |
| Probability | `u1000001-0000-0000-0000-000000000007` | Term 3 |
| **Cards (Real Numbers)** | | |
| Card 1–8 | `c1000001-0000-0000-0000-000000000001` to `...000000000008` | Mixed classic/mcq |
