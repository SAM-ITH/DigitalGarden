# Snapy - App Overview & User Flow

## Problem Statement

Sri Lankan students preparing for GCE Ordinary Level (O/L) and Advanced Level (A/L) examinations lack a modern, mobile-first study tool aligned to the national school syllabus. Existing options are either generic flashcard apps with no local curriculum support, or outdated paper-based revision methods. Students need a tool that:

- Organizes content by their exact grade, subject, and school term
- Uses scientifically-proven memory techniques (spaced repetition) to maximize retention
- Adapts to their individual learning pace and weak areas
- Makes studying engaging through gamification (streaks, leaderboards, study plans)
- Works offline for students with intermittent connectivity

Snapy fills this gap as a Duolingo-inspired study companion specifically built for Sri Lankan students.

---

## Target Audience

| Segment | Grades | Description |
|---------|--------|-------------|
| **O/L Students** | Grade 10-11 | Preparing for GCE Ordinary Level examination. Study 6 compulsory + 3 optional subjects. High exam pressure — results determine A/L stream eligibility. |
| **A/L Students** | Grade 12-13 | Preparing for GCE Advanced Level examination. Stream-based study (Science, Commerce, Arts, Technology). Extremely competitive — results determine university admission. |

---

## Core Features

### 1. Passwordless SMS Login
- Users sign up and log in using their phone number
- OTP (One-Time Password) sent via SMS using ShoutOUT API
- No passwords to remember — frictionless authentication

### 2. Onboarding Flow
- Step 1: Enter phone number
- Step 2: Verify OTP code
- Step 3: Enter name
- Step 4: Select grade (10, 11, 12, or 13)
- Step 5: (A/L only) Select stream (Science, Commerce, Arts, Technology)
- Step 6: Land on home screen with personalized content

### 3. Content Browsing
- Content organized in a clear hierarchy:
  ```
  Grade → Subject → Term → Unit → Flashcards
  ```
- Users see only the subjects relevant to their grade
- Units organized by school term (Term 1, Term 2, Term 3)
- Progress indicators on each subject and unit

### 4. Flashcard Study Sessions
Two modes of study, which can be mixed within a single session:

**Mode 1: Classic Recall**
- A question is displayed on the flashcard
- Student thinks of the answer mentally
- Taps the card to reveal the correct answer
- Self-evaluates: marks "Got it" (correct) or "Missed it" (incorrect)
- Moves to the next card

**Mode 2: Multiple Choice (MCQ)**
- A question is displayed with 4 answer options
- Student selects one option
- Instant feedback: selected answer turns green (correct) or red (incorrect)
- If incorrect, the correct answer is highlighted with an explanation
- Moves to the next card

**Mixed Mode:**
- Within a unit study session, cards alternate between Classic and MCQ types
- The mode is determined per-card based on the card type set by content creators
- Provides variety and tests different cognitive skills (recall vs. recognition)

### 5. Spaced Repetition Engine (FSRS)
- Uses the FSRS (Free Spaced Repetition Scheduler) algorithm
- Cards the student struggles with appear more frequently
- Cards the student knows well are scheduled further into the future
- Scientifically optimized to achieve ~90% retention with minimum review effort
- Adapts to each student's individual memory patterns over time

### 6. Quick Review Mode
- For students who want a fast memory refreshment
- Surfaces only cards that are "due" for review across all subjects
- Time-boxed sessions (5 or 10 minutes)
- Ideal for bus rides, breaks, or before class

### 7. Study Plans
- Student sets a goal (e.g., "Prepare for Term 1 exam")
- Selects target subjects
- Specifies available study time per day and deadline
- App generates a day-by-day study plan:
  - Distributes units across available days
  - Prioritizes weak areas and unseen content
  - Balances new learning with review sessions
- Calendar view shows daily tasks
- Tracks progress through the plan

### 8. Analytics Dashboard
- **Streak Heatmap**: GitHub-contribution-style grid showing daily study activity over the past year. Color intensity reflects amount of study that day.
- **Accuracy Trends**: Per-subject accuracy over time (charts)
- **Weak Area Identification**: Subjects and units where the student scores lowest or lapses most frequently
- **Total Stats**: Cards mastered, total reviews completed, study hours logged
- **Improvement Tracking**: How accuracy and retention have improved over time

### 9. Leaderboard
- **Grade-level leaderboard**: Compete with all students in the same grade
- **Subject-level leaderboard**: Compete within specific subjects
- Points earned from: cards reviewed, accuracy bonuses, streak multipliers
- Weekly reset to keep competition fresh and give new students a chance
- View your rank, top students, and nearby ranks

### 10. Profile & Settings
- View and edit name, grade, stream
- Notification preferences (daily reminder time)
- Study statistics summary
- Logout

---

## User Flows

### Flow 1: First Launch & Onboarding
```
App Launch
  → Splash Screen
  → Welcome Screen (app intro)
  → Phone Number Input
  → OTP Verification
  → Name Input
  → Grade Selection (10, 11, 12, 13)
  → [If A/L] Stream Selection (Science, Commerce, Arts, Technology)
  → Home Screen (personalized for grade/stream)
```

### Flow 2: Study Session
```
Home Screen
  → Select Subject (e.g., "Mathematics")
  → Subject Detail (terms listed)
  → Select Term (e.g., "Term 1")
  → Select Unit (e.g., "Algebra")
  → Flashcard Session Begins
    → Card 1 (Classic or MCQ)
    → Card 2 ...
    → Card N
  → Session Summary (score, accuracy, XP earned)
  → Return to Unit List or Start New Session
```

### Flow 3: Quick Review
```
Home Screen
  → Tap "Quick Review" button
  → Select Duration (5 min / 10 min)
  → Due cards from all subjects appear
    → Review Card 1
    → Review Card 2 ...
  → Time's up or all due cards reviewed
  → Summary Screen
```

### Flow 4: Study Plan
```
Home Screen
  → Tap "Study Plan"
  → [If no plan] Create Plan:
    → Select Goal (e.g., "Term 1 Exam Prep")
    → Select Subjects to Include
    → Set Daily Available Time (e.g., 30 min/day)
    → Set Deadline (e.g., exam date)
    → Plan Generated
  → [If plan exists] View Plan:
    → Calendar View with daily tasks
    → Tap today's tasks
    → Start recommended study session
    → Complete task → mark as done
```

### Flow 5: Analytics
```
Home Screen
  → Tap "Profile" / Analytics tab
  → Dashboard:
    → Streak Heatmap (365 days)
    → Overall Stats (total cards, accuracy, hours)
    → Accuracy by Subject (bar/line chart)
    → Weak Areas (highlighted subjects/units)
    → Improvement Over Time
```

### Flow 6: Leaderboard
```
Home Screen
  → Tap "Leaderboard" tab
  → Grade Leaderboard (default view)
    → Your rank highlighted
    → Top 50 students
  → Switch to Subject tab
    → Select subject
    → Subject-specific rankings
```

---

## Content Structure - Sri Lankan Syllabus

### O/L Subjects (Grade 10-11)

**Compulsory Subjects (6):**
1. First Language (Sinhala or Tamil)
2. English
3. Mathematics
4. Science
5. History
6. Religion (Buddhism / Hinduism / Islam / Christianity)

**Optional Subjects (students choose 3):**
- Second Language (Sinhala / Tamil)
- Commerce & Accounting
- Geography
- Civic Education
- Health & Physical Education
- Art / Music / Dancing
- Information & Communication Technology (ICT)
- Agriculture & Food Technology
- Home Economics
- Drama & Theatre

### A/L Streams & Subjects (Grade 12-13)

**Science Stream:**
- Combined Mathematics
- Physics
- Chemistry
- Biology (Bio Science track)
- Higher Mathematics (Physical Science track)

**Commerce Stream:**
- Business Studies
- Accounting
- Economics
- Business Statistics

**Arts Stream:**
- Political Science
- Geography
- Economics
- Sinhala / Tamil / English Literature
- History
- Logic
- Buddhist Civilization / Hindu Civilization

**Technology Stream:**
- Engineering Technology
- Bio Systems Technology
- Science for Technology
- Information & Communication Technology

**Common for all A/L streams:**
- General English (compulsory)
- Common General Test

### Term Structure

Each academic year is divided into **3 terms**:
- **Term 1**: January - April
- **Term 2**: May - August
- **Term 3**: September - December

Units within each subject are organized by the term in which they are taught, following the national curriculum timeline.

---

## Content Hierarchy Example

```
Grade 11 (O/L)
├── Mathematics
│   ├── Term 1
│   │   ├── Unit: Real Numbers
│   │   ├── Unit: Indices & Logarithms
│   │   ├── Unit: Algebraic Expressions
│   │   └── Unit: Linear Equations
│   ├── Term 2
│   │   ├── Unit: Simultaneous Equations
│   │   ├── Unit: Triangles & Geometry
│   │   ├── Unit: Trigonometry
│   │   └── Unit: Statistics
│   └── Term 3
│       ├── Unit: Sets
│       ├── Unit: Probability
│       ├── Unit: Mensuration
│       └── Unit: Graphs
├── Science
│   ├── Term 1
│   │   ├── Unit: Measurement
│   │   ├── Unit: Force & Motion
│   │   └── ...
│   └── ...
└── ...
```

---

## Gamification Elements (Duolingo-Inspired)

| Element | Description |
|---------|-------------|
| **Daily Streak** | Consecutive days with at least one study session. Visible on home screen. |
| **XP Points** | Earned per card reviewed. Bonus for accuracy and streaks. |
| **Streak Heatmap** | Visual 365-day grid showing activity intensity. |
| **Leaderboard Rank** | Weekly competition within grade and subject. |
| **Session Summary** | Post-session screen showing score, accuracy, XP earned. |
| **Study Plan Progress** | Visual progress bar through the study plan. |
| **Streak Freeze** | (Future) Protect streak with earned rewards. |
| **Achievements** | (Future) Badges for milestones (100 cards, 7-day streak, etc.). |
