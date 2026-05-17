# Snapy - Spaced Repetition Engine (FSRS)

## Overview

The spaced repetition engine is the core algorithm that powers Snapy's learning effectiveness. It determines **when** each flashcard should be shown again to maximize long-term retention with minimum review effort. Snapy uses the **FSRS (Free Spaced Repetition Scheduler)** algorithm — the most advanced open-source spaced repetition algorithm available.

**Architecture decision:** FSRS runs **exclusively on the Go backend**. Mobile apps send raw review results (card ID + rating + timestamp) and the server computes all scheduling. This keeps one authoritative implementation, eliminates cross-platform consistency concerns, and simplifies the mobile apps.

---

## Why FSRS Over SM-2

SM-2 (SuperMemo 2) was the standard spaced repetition algorithm since 1987, used by Anki and many other apps. FSRS is its modern replacement. Here's why Snapy uses FSRS:

| Factor | SM-2 (1987) | FSRS (2022+) |
|--------|-------------|-------------|
| **Efficiency** | Baseline | 20-30% fewer reviews for same retention |
| **Adaptivity** | Fixed parameters, manual tuning needed | Self-optimizing based on user behavior |
| **Scientific basis** | Empirical (trial and error) | Based on Three Component Model of Memory (DSR theory) |
| **Training data** | None | Default parameters trained on 700M+ reviews from 20K+ users |
| **Adoption** | Legacy standard | Anki adopted FSRS in v23.10; industry moving to FSRS |
| **Maintenance** | No active development | Active open-source project with ongoing research |

### What This Means for Students
- Students spend **less time reviewing** and **more time learning new material**
- Cards they know well disappear for longer (weeks/months instead of being shown too soon)
- Cards they struggle with appear at exactly the right interval to strengthen memory
- The algorithm gets better at predicting their memory over time

---

## How FSRS Works

### The Three Component Model of Memory

FSRS is based on the DSR (Difficulty, Stability, Retrievability) model:

1. **Difficulty (D)**: How inherently hard the card is for this student. Range: 1 (easiest) to 10 (hardest). Higher difficulty means the card needs more frequent review.

2. **Stability (S)**: How strong the memory is, measured in days. Specifically, stability is the number of days after which the probability of recall drops to 90%. Higher stability = longer intervals between reviews.

3. **Retrievability (R)**: The current probability that the student can recall the answer right now. Starts at 1.0 (100%) right after a review and decays exponentially over time following the forgetting curve.

```
Retrievability = (1 + elapsed_days / (9 × stability))^(-1)

When elapsed_days = stability × 9 × (retention^(-1) - 1):
  → R drops to the desired retention rate (default 90%)
  → This is when the card should be reviewed
```

### Card States

Each card progresses through states:

```
                  ┌─────────┐
                  │   New    │  Card has never been studied
                  └────┬────┘
                       │ First review
                  ┌────▼────┐
            ┌─────│Learning │  Card is being initially learned
            │     └────┬────┘
            │          │ Graduate (after Good/Easy ratings)
            │     ┌────▼────┐
            │     │ Review   │  Card is in long-term review cycle
            │     └────┬────┘
            │          │ Lapse (rated "Again")
            │   ┌──────▼──────┐
            │   │ Relearning  │  Card was forgotten, being relearned
            │   └──────┬──────┘
            │          │ Re-graduate
            │          └──────→ Review
            │
            └──── Lapse ──→ Relearning
```

### Rating System

After each review, the student rates their recall:

| Rating | Value | Meaning | When to Use |
|--------|-------|---------|-------------|
| **Again** | 1 | Complete failure to recall | Student couldn't remember at all |
| **Hard** | 2 | Recalled with significant difficulty | Took a long time or was partially wrong |
| **Good** | 3 | Recalled with moderate effort | Normal successful recall |
| **Easy** | 4 | Recalled effortlessly | Instant, confident recall |

**Snapy's simplified mapping:**
- Classic mode: "Got it" → Good (3), "Missed it" → Again (1)
- MCQ mode: Correct answer → Good (3), Wrong answer → Again (1)

This simplified 2-option rating (instead of 4) is more appropriate for a mobile app where speed matters. Students don't need to deliberate over granular difficulty ratings.

### Algorithm Computation

On each review, FSRS computes:

1. **New Difficulty**:
   ```
   D' = D - w6 × (rating - 3)
   // Difficulty decreases when rated Good/Easy
   // Difficulty increases when rated Again/Hard
   // Clamped to [1, 10]
   ```

2. **New Stability** (depends on current state):
   - For **first review** (New → Learning):
     ```
     S = w[rating-1]  // Initial stability from FSRS parameters
     // w0=0.40, w1=0.60, w2=2.40, w3=5.80 (defaults)
     ```
   - For **successful review** (Good/Easy on Review card):
     ```
     S' = S × (1 + exp(w8) × (11 - D) × S^(-w9) × (exp(w10 × (1 - R)) - 1))
     // Stability grows — more for: easy cards, low difficulty, high retrievability at review time
     ```
   - For **lapse** (Again on Review card):
     ```
     S' = w11 × D^(-w12) × ((S+1)^w13 - 1) × exp(w14 × (1 - R))
     // Stability drops dramatically but retains some memory benefit
     ```

3. **Next Interval**:
   ```
   interval = S' × 9 × (1/desired_retention - 1)
   // For 90% retention: interval ≈ S' × 1.0
   // The card is scheduled for review after 'interval' days
   ```

### FSRS Parameters

The algorithm uses 19 trainable parameters (w0 through w18). Default values are trained on 700M+ reviews:

```
w = [0.40, 0.60, 2.40, 5.80, 4.93, 0.94, 0.86, 0.01, 1.49, 0.14,
     0.94, 2.18, 0.05, 0.34, 1.26, 0.29, 2.61, 0.00, 0.00]
```

These defaults work well out of the box for most students. Personalized parameters can be computed after a student accumulates 500+ reviews (see Future Enhancements).

---

## Implementation: Server-Side Only (Go)

### Architecture

FSRS runs exclusively in the Go backend. The server is the single source of truth for all SRS state.

```
Mobile App                          Go Backend (Review Service)
──────────                          ────────────────────────────
Student answers card
  │
  ├── Records: { cardId, rating, timestamp }
  │
  └── Queues for sync
       │
       ▼
POST /reviews (batch)
  [{ cardId, rating, reviewedAt }, ...]
                                     │
                                     ▼
                                   For each review:
                                     1. Get current SRS state from DB
                                        (or create new if first review)
                                     2. Run FSRS:
                                        input: current state + rating
                                        output: new stability, difficulty, due date
                                     3. Upsert card_srs_state
                                     4. Insert card_reviews (history)
                                     5. Update analytics (streak, activity)
                                     │
                                     ▼
                                   Response: { accepted, rejected }
  
GET /reviews/due                     │
  ◄──────────────────────────────────┘
  Returns cards where due_at <= NOW()
  (server-computed due dates)
```

### Why Server-Only

| Factor | Client-Side FSRS | Server-Only FSRS |
|--------|------------------|-------------------|
| Implementations needed | 3 (Swift, Kotlin, Go) | **1 (Go only)** |
| Cross-platform parity testing | Required (shared test vectors) | **Not needed** |
| Conflict resolution | Needed (two devices) | **Not needed** |
| Offline Quick Review | Works | **Requires connectivity** |
| Offline unit study | Works | Works (cards cached, results queued) |
| Code complexity on mobile | High (algorithm + local state) | **Low (just send ratings)** |
| Network payload per review | Heavy (cardId + rating + full SRS state) | **Light (cardId + rating only)** |
| Single source of truth | No (client + server) | **Yes (server only)** |
| Easy to update algorithm | Must update 3 platforms | **Update 1 place** |

**The tradeoff:** Quick Review (studying due cards across all subjects) requires internet because only the server knows which cards are due. This is an acceptable tradeoff because:
- Sri Lankan students typically have mobile data available
- Offline unit study (the main use case) still works fine
- The architecture is significantly simpler

### Go Implementation

The backend uses the `go-fsrs` package:

```go
// internal/service/fsrs.go
package service

import (
    "time"
    gofsrs "github.com/open-spaced-repetition/go-fsrs"
)

type FSRSService struct {
    scheduler gofsrs.FSRS
}

func NewFSRSService() *FSRSService {
    params := gofsrs.DefaultParameters()
    return &FSRSService{
        scheduler: gofsrs.NewFSRS(params),
    }
}

type SRSOutput struct {
    Stability     float64
    Difficulty    float64
    State         string
    DueAt         time.Time
    ScheduledDays int
    ElapsedDays   int
    Reps          int
    Lapses        int
}

// ProcessReview runs FSRS for a single card review.
// If the card has no existing state (new card), initialState is used.
func (s *FSRSService) ProcessReview(
    currentStability float64,
    currentDifficulty float64,
    currentState string,
    currentReps int,
    currentLapses int,
    currentDue time.Time,
    rating int,
    reviewedAt time.Time,
) SRSOutput {
    card := gofsrs.Card{
        Stability:  currentStability,
        Difficulty: currentDifficulty,
        Reps:       currentReps,
        Lapses:     currentLapses,
        Due:        currentDue,
    }

    // Map state string to go-fsrs state
    switch currentState {
    case "new":
        card.State = gofsrs.New
    case "learning":
        card.State = gofsrs.Learning
    case "review":
        card.State = gofsrs.Review
    case "relearning":
        card.State = gofsrs.Relearning
    }

    results := s.scheduler.Repeat(card, reviewedAt)
    result := results[gofsrs.Rating(rating)]

    return SRSOutput{
        Stability:     result.Card.Stability,
        Difficulty:    result.Card.Difficulty,
        State:         stateToString(result.Card.State),
        DueAt:         result.Card.Due,
        ScheduledDays: result.Card.ScheduledDays,
        ElapsedDays:   result.Card.ElapsedDays,
        Reps:          result.Card.Reps,
        Lapses:        result.Card.Lapses,
    }
}

func stateToString(state gofsrs.State) string {
    switch state {
    case gofsrs.New:
        return "new"
    case gofsrs.Learning:
        return "learning"
    case gofsrs.Review:
        return "review"
    case gofsrs.Relearning:
        return "relearning"
    default:
        return "new"
    }
}
```

### Integration with Review Service

When the Review Service receives a batch of reviews:

```go
// Pseudocode for batch processing
for each review in batch:
    // 1. Get existing SRS state (if any)
    existingState := db.GetSRSState(userID, review.CardID)

    var output SRSOutput
    if existingState == nil {
        // New card — use default initial values
        output = fsrs.ProcessReview(
            stability:  0,
            difficulty: 0,
            state:      "new",
            reps:       0,
            lapses:     0,
            due:        time.Now(),
            rating:     review.Rating,
            reviewedAt: review.ReviewedAt,
        )
    } else {
        // Existing card — use current state
        output = fsrs.ProcessReview(
            stability:  existingState.Stability,
            difficulty: existingState.Difficulty,
            state:      existingState.State,
            reps:       existingState.Reps,
            lapses:     existingState.Lapses,
            due:        existingState.DueAt,
            rating:     review.Rating,
            reviewedAt: review.ReviewedAt,
        )
    }

    // 2. Insert review history
    db.CreateReview(userID, review.CardID, review.Rating, review.ReviewedAt)

    // 3. Upsert SRS state with FSRS output
    db.UpsertSRSState(userID, review.CardID, output)

    // 4. Update analytics (daily activity, streak)
    analytics.RecordActivity(userID, review)
```

---

## Integration with App Features

### 1. Due Cards Query
Cards are "due" when their scheduled review time has passed.

```sql
-- Backend query for due cards
SELECT * FROM card_srs_state
WHERE user_id = $1 AND due_at <= NOW()
ORDER BY due_at ASC;  -- Most overdue first
```

The number of due cards is shown on the home screen as a badge. **Requires connectivity** — the phone fetches this from the server.

### 2. Quick Review Mode
Quick Review pulls due cards from **all subjects** and presents them in a time-limited session.

```
User taps "Quick Review" → Select duration (5/10 min)
  → App calls GET /reviews/due (requires internet)
  → Server returns cards where due_at <= NOW()
  → Present cards until time runs out or all reviewed
  → Send ratings batch to server
  → Server runs FSRS for each card
```

**Note:** Quick Review requires internet. This is the main tradeoff of server-only FSRS.

### 3. Weak Area Detection
A card/unit is "weak" if:
- It has a **high lapse count** (frequently forgotten and moved to Relearning)
- It has **low stability** (memory decays quickly)
- It has **high difficulty** (inherently hard for this student)

```sql
-- Weak units query
SELECT u.id, u.name, 
       AVG(cs.stability) as avg_stability,
       SUM(cs.lapses) as total_lapses,
       AVG(cs.difficulty) as avg_difficulty
FROM card_srs_state cs
JOIN cards c ON c.id = cs.card_id
JOIN units u ON u.id = c.unit_id
WHERE cs.user_id = $1
GROUP BY u.id, u.name
ORDER BY avg_stability ASC, total_lapses DESC
LIMIT 5;
```

### 4. Study Plan Weighting
When generating a study plan, the algorithm considers FSRS data:
- Units with more **overdue cards** get higher priority
- Units with more **Relearning-state cards** are scheduled more frequently
- Units with higher average **difficulty** get more daily time allocated

### 5. Unit Study (Offline Compatible)
When a student opens a cached unit:
- All cards in the unit are shown regardless of SRS state
- Student studies all cards in order
- Results are queued locally: `{ cardId, rating, timestamp }`
- When connectivity returns, batch is sent to server
- Server runs FSRS for each queued review

---

## Mobile App Responsibilities (Regarding FSRS)

Mobile apps do **NOT** run FSRS. Their responsibilities are limited to:

### During a Study Session
1. Display cards (from local cache or server)
2. Collect the student's rating for each card:
   - Classic: "Got it" (rating 3) or "Missed it" (rating 1)
   - MCQ: Correct (rating 3) or Incorrect (rating 1)
3. Record: `{ cardId, rating, reviewedAt }`
4. Queue result for sync

### Syncing Results
1. When online, batch-upload queued reviews to `POST /reviews`
2. Request body is simple — just ratings, no SRS state:
   ```json
   {
     "reviews": [
       { "cardId": "uuid-1", "rating": 3, "reviewedAt": "2026-05-01T10:30:00Z" },
       { "cardId": "uuid-2", "rating": 1, "reviewedAt": "2026-05-01T10:31:00Z" }
     ]
   }
   ```
3. Server responds with `{ accepted, rejected }`

### Fetching Due Cards
1. Call `GET /reviews/due` when online
2. Server returns cards with server-computed due dates
3. Used for Quick Review mode and home screen badge

---

## Example: A Student's Journey with FSRS

```
Day 1: Student first sees "What is photosynthesis?"
  → Sends: { cardId, rating: 3 (Good), reviewedAt }
  → Server FSRS: New → Learning, stability = 2.40 days
  → Due: Day 3

Day 3: Card appears for review (server says it's due)
  → Student answers correctly: rating 3 (Good)
  → Server FSRS: Review, stability grows to 6.8 days
  → Due: Day 10

Day 10: Card appears again
  → Student answers correctly: rating 3 (Good)
  → Server FSRS: stability grows to 18.5 days
  → Due: Day 29

Day 29: Card appears
  → Student forgets! rating 1 (Again)
  → Server FSRS: Review → Relearning, stability drops to 1.2 days
  → Due: Day 30

Day 30: Card appears (quick relearn)
  → Student answers correctly: rating 3 (Good)
  → Server FSRS: Relearning → Review, stability = 3.5 days
  → Due: Day 34

Day 34: Card appears
  → Student answers correctly: rating 3 (Good)
  → Server FSRS: stability = 9.1 days
  → Continues growing from here...
```

Notice how:
- Intervals grow exponentially when the student answers correctly (2 → 7 → 19 days)
- A lapse resets stability but not to zero (the brain retains partial memory)
- Recovery after a lapse is faster than the initial learning (1.2 → 3.5 → 9.1 days)

---

## Desired Retention Configuration

The `desiredRetention` parameter controls the tradeoff between review frequency and recall probability:

| Retention | Effect | Use Case |
|-----------|--------|----------|
| **0.85 (85%)** | Longer intervals, fewer reviews | Students comfortable with occasional forgetting |
| **0.90 (90%)** | Balanced (default) | Most students — good retention without excessive reviews |
| **0.95 (95%)** | Shorter intervals, more reviews | Exam preparation — minimize forgetting |

**Snapy default**: 0.90 (90%)

**Future feature**: Allow students to adjust retention per subject or when an exam is approaching (temporarily increase to 0.95).

---

## Unit Testing the Algorithm (Server-Side Only)

Since FSRS runs only in Go, testing is straightforward — no cross-platform parity testing needed.

```go
// internal/service/fsrs_test.go
package service

import (
    "testing"
    "time"
)

func TestNewCardRatedGood(t *testing.T) {
    svc := NewFSRSService()
    
    result := svc.ProcessReview(
        0,          // stability (new card)
        0,          // difficulty (new card)
        "new",      // state
        0,          // reps
        0,          // lapses
        time.Now(), // due
        3,          // rating (Good)
        time.Now(), // reviewedAt
    )
    
    if result.State != "learning" {
        t.Errorf("expected state 'learning', got '%s'", result.State)
    }
    if result.Stability < 2.0 {
        t.Errorf("expected stability >= 2.0 for Good rating, got %f", result.Stability)
    }
    if result.ScheduledDays < 1 {
        t.Errorf("expected at least 1 scheduled day, got %d", result.ScheduledDays)
    }
}

func TestReviewCardRatedAgain(t *testing.T) {
    svc := NewFSRSService()
    now := time.Now()
    
    result := svc.ProcessReview(
        10.0,       // stability
        5.0,        // difficulty
        "review",   // state
        5,          // reps
        0,          // lapses
        now.AddDate(0, 0, -10), // due 10 days ago
        1,          // rating (Again)
        now,
    )
    
    if result.State != "relearning" {
        t.Errorf("expected state 'relearning', got '%s'", result.State)
    }
    if result.Stability >= 10.0 {
        t.Errorf("expected stability to decrease after Again, got %f", result.Stability)
    }
    if result.Lapses != 1 {
        t.Errorf("expected lapses=1, got %d", result.Lapses)
    }
}

func TestSuccessfulReviewIncreasesStability(t *testing.T) {
    svc := NewFSRSService()
    now := time.Now()
    
    result := svc.ProcessReview(
        5.0,        // stability
        5.0,        // difficulty
        "review",   // state
        3,          // reps
        0,          // lapses
        now.AddDate(0, 0, -5), // due 5 days ago
        3,          // rating (Good)
        now,
    )
    
    if result.State != "review" {
        t.Errorf("expected state 'review', got '%s'", result.State)
    }
    if result.Stability <= 5.0 {
        t.Errorf("expected stability to increase after Good, got %f", result.Stability)
    }
}

func TestIntervalsGrowExponentially(t *testing.T) {
    svc := NewFSRSService()
    
    stability := 0.0
    difficulty := 0.0
    state := "new"
    reps := 0
    lapses := 0
    due := time.Now()
    intervals := []int{}
    
    for i := 0; i < 5; i++ {
        now := due
        result := svc.ProcessReview(stability, difficulty, state, reps, lapses, due, 3, now)
        
        intervals = append(intervals, result.ScheduledDays)
        stability = result.Stability
        difficulty = result.Difficulty
        state = result.State
        reps = result.Reps
        lapses = result.Lapses
        due = result.DueAt
    }
    
    for i := 1; i < len(intervals); i++ {
        if intervals[i] < intervals[i-1] {
            t.Errorf("intervals should grow: %d < %d at step %d", intervals[i], intervals[i-1], i)
        }
    }
}
```

---

## Future Enhancements

### 1. Personalized Parameters (Phase 3+)
After a student accumulates **500+ reviews**, run the FSRS optimizer on their review history to compute personalized parameters.

**How it works:**
- Collect all `(card_srs_state, card_reviews)` data for the user
- Run the FSRS optimizer (gradient descent on the 19 parameters)
- Store personalized `w` parameters in the user's profile
- The server uses personalized parameters instead of defaults

**Benefits:**
- 10-20% additional reduction in reviews compared to default parameters
- Algorithm truly adapts to individual memory characteristics

**Implementation:**
- Server-side batch job (not real-time)
- Run weekly for eligible users (500+ reviews)
- Store in `users` table as JSONB `fsrs_params` column

### 2. Adjustable Retention per Subject
Allow students to set different retention targets:
- Default: 90% for regular study
- Exam mode: 95% for subjects with upcoming exams
- Low-priority: 85% for subjects the student is already strong in

### 3. Cached Due Cards for Offline Quick Review (Future)
If offline Quick Review becomes important:
- When online, fetch and cache the due cards list locally
- Allow reviewing cached due cards offline
- Queue results for sync when connectivity returns
- This is a lightweight enhancement that doesn't require running FSRS locally

### 4. FSRS v5+ Updates
The FSRS algorithm is actively developed. As new versions are released with improved parameter training or formula updates, only the Go implementation needs to be updated. Since there's only one implementation, updates are straightforward.

---

## References

- [FSRS Algorithm Specification](https://github.com/open-spaced-repetition/fsrs4anki/wiki/The-Algorithm)
- [open-spaced-repetition GitHub Organization](https://github.com/open-spaced-repetition)
- [go-fsrs (Go implementation)](https://github.com/open-spaced-repetition/go-fsrs)
- [FSRS vs SM-2 Comparison](https://github.com/open-spaced-repetition/fsrs4anki/wiki/Comparison-with-other-spaced-repetition-algorithms)
- [Three Component Model of Memory (DSR Theory)](https://supermemo.guru/wiki/Three_component_model_of_memory)
