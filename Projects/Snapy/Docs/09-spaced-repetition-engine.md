# Snapy - Spaced Repetition Engine (FSRS)

## Overview

The spaced repetition engine is the core algorithm that powers Snapy's learning effectiveness. It determines **when** each flashcard should be shown again to maximize long-term retention with minimum review effort. Snapy uses the **FSRS (Free Spaced Repetition Scheduler)** algorithm — the most advanced open-source spaced repetition algorithm available.

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

## Implementation

### Native Per Platform

FSRS is implemented natively on each platform rather than shared via a cross-platform layer. The algorithm is ~200 lines of deterministic math — simple enough to implement in each language, and existing open-source libraries are available.

| Platform | Library / Approach | Language |
|----------|-------------------|---------|
| **iOS** | `swift-fsrs` package or native implementation | Swift |
| **Android** | `fsrs-kt` package or native implementation | Kotlin |
| **Backend** | `go-fsrs` package | Go |

### Why Not KMP (Kotlin Multiplatform)?
KMP would share code across platforms, but the overhead outweighs the benefit for FSRS:
- KMP adds complex build setup (Gradle multiplatform config)
- SKIE bridging layer needed for iOS (additional dependency)
- Separate repo + CI pipeline required
- Debugging KMP/iOS interop issues adds friction
- All of this to share ~200 lines of well-specified math

Instead, each platform uses the same algorithm spec, and **shared test vectors** verify that all three implementations produce identical results.

### Implementation Structure Per Platform

**iOS** (`Snapy/FSRS/`):
```
FSRS/
├── FSRS.swift              # Main algorithm
├── FSRSParameters.swift    # Algorithm parameters (w0-w18)
├── CardState.swift         # State enum
├── Rating.swift            # Rating enum
└── ReviewResult.swift      # Output of a review
```

**Android** (`app/src/main/java/.../fsrs/`):
```
fsrs/
├── FSRS.kt                 # Main algorithm
├── FSRSParameters.kt       # Algorithm parameters (w0-w18)
├── CardState.kt            # State enum
├── Rating.kt               # Rating enum
└── ReviewResult.kt         # Output of a review
```

**Backend** (`internal/service/`):
```
service/
└── fsrs.go                 # go-fsrs integration for server-side validation
```

### Core Types (Kotlin Reference — Swift and Go equivalents follow the same structure)

```kotlin
// Rating.kt
enum class Rating(val value: Int) {
    Again(1),
    Hard(2),
    Good(3),
    Easy(4)
}

// CardState.kt
enum class CardState {
    New,
    Learning,
    Review,
    Relearning
}

// ReviewResult.kt
data class ReviewResult(
    val cardId: String,
    val rating: Rating,
    val reviewedAt: Long,          // epoch millis
    val nextState: CardState,
    val stability: Double,
    val difficulty: Double,
    val elapsedDays: Int,
    val scheduledDays: Int,
    val dueAt: Long                // epoch millis
)

// FSRSParameters.kt
data class FSRSParameters(
    val w: DoubleArray,            // 19 parameters
    val desiredRetention: Double = 0.9  // 90% target recall
) {
    companion object {
        val default = FSRSParameters(
            w = doubleArrayOf(
                0.40, 0.60, 2.40, 5.80, 4.93, 0.94, 0.86, 0.01, 1.49, 0.14,
                0.94, 2.18, 0.05, 0.34, 1.26, 0.29, 2.61, 0.00, 0.00
            )
        )
    }
}
```

### FSRS Algorithm (Kotlin Reference Implementation)

```kotlin
// FSRS.kt
class FSRS(private val params: FSRSParameters = FSRSParameters.default) {
    
    /**
     * Process a review and compute the next scheduling state.
     */
    fun review(
        cardId: String,
        currentState: CardState,
        currentStability: Double,
        currentDifficulty: Double,
        elapsedDays: Int,
        rating: Rating
    ): ReviewResult {
        val now = Clock.System.now().toEpochMilliseconds()
        
        val retrievability = if (currentState == CardState.Review && elapsedDays > 0) {
            calculateRetrievability(elapsedDays.toDouble(), currentStability)
        } else {
            1.0 // Learning/New cards don't have meaningful retrievability
        }
        
        val newDifficulty = nextDifficulty(currentDifficulty, rating)
        val newStability = nextStability(
            currentState, currentStability, newDifficulty, retrievability, rating
        )
        val newState = nextState(currentState, rating)
        val scheduledDays = nextInterval(newStability)
        val dueAt = now + (scheduledDays * 24L * 60 * 60 * 1000)
        
        return ReviewResult(
            cardId = cardId,
            rating = rating,
            reviewedAt = now,
            nextState = newState,
            stability = newStability,
            difficulty = newDifficulty,
            elapsedDays = elapsedDays,
            scheduledDays = scheduledDays,
            dueAt = dueAt
        )
    }
    
    private fun calculateRetrievability(elapsed: Double, stability: Double): Double {
        return (1.0 + elapsed / (9.0 * stability)).pow(-1.0)
    }
    
    private fun nextDifficulty(d: Double, rating: Rating): Double {
        val newD = d - params.w[6] * (rating.value - 3)
        return newD.coerceIn(1.0, 10.0)
    }
    
    private fun nextStability(
        state: CardState,
        s: Double,
        d: Double,
        r: Double,
        rating: Rating
    ): Double {
        return when {
            // First review (New card)
            state == CardState.New -> params.w[rating.value - 1]
            
            // Lapse (Again on a Review card)
            rating == Rating.Again -> {
                params.w[11] * d.pow(-params.w[12]) * 
                    ((s + 1).pow(params.w[13]) - 1) * 
                    exp(params.w[14] * (1 - r))
            }
            
            // Successful review (Hard/Good/Easy on Review card)
            else -> {
                s * (1 + exp(params.w[8]) * (11 - d) * 
                    s.pow(-params.w[9]) * 
                    (exp(params.w[10] * (1 - r)) - 1))
            }
        }.coerceAtLeast(0.01) // Minimum stability
    }
    
    private fun nextState(current: CardState, rating: Rating): CardState {
        return when (current) {
            CardState.New -> if (rating == Rating.Again) CardState.Learning else CardState.Learning
            CardState.Learning -> when (rating) {
                Rating.Again -> CardState.Learning
                Rating.Hard -> CardState.Learning
                Rating.Good -> CardState.Review
                Rating.Easy -> CardState.Review
            }
            CardState.Review -> when (rating) {
                Rating.Again -> CardState.Relearning
                else -> CardState.Review
            }
            CardState.Relearning -> when (rating) {
                Rating.Again -> CardState.Relearning
                Rating.Hard -> CardState.Relearning
                Rating.Good -> CardState.Review
                Rating.Easy -> CardState.Review
            }
        }
    }
    
    private fun nextInterval(stability: Double): Int {
        val interval = stability * 9.0 * (1.0 / params.desiredRetention - 1.0)
        return maxOf(1, interval.roundToInt())  // Minimum 1 day
    }
}
```

### Server-Side Validation (Go)

The backend uses the `go-fsrs` package for server-side validation. This verifies that the client's FSRS output is correct.

```go
// internal/service/fsrs.go
package service

import (
    "math"
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

func (s *FSRSService) ValidateSRSState(clientState ClientSRSState, rating int) (bool, ServerSRSState) {
    // Server recomputes to validate client's FSRS output
    // This catches bugs in mobile implementation or tampered data
    card := gofsrs.Card{
        Stability:  clientState.Stability,
        Difficulty: clientState.Difficulty,
        // ... map remaining fields
    }
    
    result := s.scheduler.Repeat(card, time.Now())
    expected := result[gofsrs.Rating(rating)]
    
    // Allow small floating point differences
    isValid := math.Abs(expected.Card.Stability-clientState.Stability) < 0.01 &&
               math.Abs(expected.Card.Difficulty-clientState.Difficulty) < 0.01
    
    return isValid, ServerSRSState{
        Stability:     expected.Card.Stability,
        Difficulty:    expected.Card.Difficulty,
        State:         expected.Card.State,
        Due:           expected.Card.Due,
        ScheduledDays: expected.Card.ScheduledDays,
        ElapsedDays:   expected.Card.ElapsedDays,
        Reps:          expected.Card.Reps,
        Lapses:        expected.Card.Lapses,
    }
}
```
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

The number of due cards is shown on the home screen as a badge.

### 2. Quick Review Mode
Quick Review pulls due cards from **all subjects** and presents them in a time-limited session.

```
User taps "Quick Review" → Select duration (5/10 min)
  → Fetch all due cards (across all subjects)
  → Sort by most overdue first
  → Present cards until time runs out or all reviewed
  → Sync results
```

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

### 5. Session Card Ordering
Within a study session, cards are ordered by FSRS priority:
1. **Relearning cards** (highest priority — recently forgotten)
2. **Due review cards** (sorted by how overdue they are)
3. **New cards** (not yet studied)
4. **Learning cards** (in initial learning phase)

---

## Example: A Student's Journey with FSRS

```
Day 1: Student first sees "What is photosynthesis?"
  → State: New → Learning
  → Rating: Good (3)
  → Stability: 2.40 days
  → Scheduled: 2 days → Due Day 3

Day 3: Card appears for review
  → State: Review
  → Student answers correctly: Good (3)
  → Retrievability was ~90% (right on schedule)
  → New Stability: 6.8 days
  → Scheduled: 7 days → Due Day 10

Day 10: Card appears again
  → State: Review
  → Student answers correctly: Good (3)
  → New Stability: 18.5 days
  → Scheduled: 19 days → Due Day 29

Day 29: Card appears
  → State: Review
  → Student forgets! Rating: Again (1)
  → State: Review → Relearning
  → New Stability: 1.2 days (drops dramatically)
  → Scheduled: 1 day → Due Day 30

Day 30: Card appears (quick relearn)
  → State: Relearning
  → Student answers correctly: Good (3)
  → State: Relearning → Review
  → New Stability: 3.5 days
  → Scheduled: 4 days → Due Day 34

Day 34: Card appears
  → State: Review
  → Student answers correctly: Good (3)
  → New Stability: 9.1 days
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

## Unit Testing the Algorithm

```kotlin
// FSRSTest.kt
class FSRSTest {
    private val fsrs = FSRS(FSRSParameters.default)
    
    @Test
    fun `new card rated Good should move to Learning with initial stability`() {
        val result = fsrs.review(
            cardId = "test-1",
            currentState = CardState.New,
            currentStability = 0.0,
            currentDifficulty = 5.0,
            elapsedDays = 0,
            rating = Rating.Good
        )
        
        assertEquals(CardState.Learning, result.nextState)
        assertEquals(2.40, result.stability, 0.01)  // w[2] for Good
        assertTrue(result.scheduledDays >= 1)
    }
    
    @Test
    fun `review card rated Again should move to Relearning with reduced stability`() {
        val result = fsrs.review(
            cardId = "test-2",
            currentState = CardState.Review,
            currentStability = 10.0,
            currentDifficulty = 5.0,
            elapsedDays = 10,
            rating = Rating.Again
        )
        
        assertEquals(CardState.Relearning, result.nextState)
        assertTrue(result.stability < 10.0)  // Stability decreased
        assertTrue(result.stability > 0.0)   // But not zero
    }
    
    @Test
    fun `successful review should increase stability`() {
        val result = fsrs.review(
            cardId = "test-3",
            currentState = CardState.Review,
            currentStability = 5.0,
            currentDifficulty = 5.0,
            elapsedDays = 5,
            rating = Rating.Good
        )
        
        assertEquals(CardState.Review, result.nextState)
        assertTrue(result.stability > 5.0)  // Stability grew
    }
    
    @Test
    fun `intervals should grow exponentially with consistent Good ratings`() {
        var stability = 0.0
        var difficulty = 5.0
        var state = CardState.New
        var elapsedDays = 0
        val intervals = mutableListOf<Int>()
        
        // Simulate 5 consecutive Good reviews
        repeat(5) {
            val result = fsrs.review(
                cardId = "test-4",
                currentState = state,
                currentStability = stability,
                currentDifficulty = difficulty,
                elapsedDays = elapsedDays,
                rating = Rating.Good
            )
            
            intervals.add(result.scheduledDays)
            stability = result.stability
            difficulty = result.difficulty
            state = result.nextState
            elapsedDays = result.scheduledDays
        }
        
        // Each interval should be longer than the last
        for (i in 1 until intervals.size) {
            assertTrue(intervals[i] >= intervals[i - 1])
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
- The app uses personalized parameters instead of defaults

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

### 3. FSRS v5+ Updates
The FSRS algorithm is actively developed. As new versions are released with improved parameter training or formula updates, each platform's FSRS library can be updated independently. Since each implementation is just ~200 lines of well-specified math, updating all three platforms is straightforward.

---

## References

- [FSRS Algorithm Specification](https://github.com/open-spaced-repetition/fsrs4anki/wiki/The-Algorithm)
- [open-spaced-repetition GitHub Organization](https://github.com/open-spaced-repetition)
- [go-fsrs (Go implementation)](https://github.com/open-spaced-repetition/go-fsrs)
- [FSRS vs SM-2 Comparison](https://github.com/open-spaced-repetition/fsrs4anki/wiki/Comparison-with-other-spaced-repetition-algorithms)
- [Three Component Model of Memory (DSR Theory)](https://supermemo.guru/wiki/Three_component_model_of_memory)
