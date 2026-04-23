# Snapy - Phased Implementation Plan

## Overview

The implementation is divided into **7 phases** spanning approximately **20 weeks**. Each phase has clear deliverables and exit criteria. The approach prioritizes getting a working end-to-end flow early (auth → content → study) and layering features incrementally.

---

## Phase 0: Foundation (Weeks 1-2)

### Goal
Set up all infrastructure, project scaffolding, database schema, and CI/CD. By the end, a "hello world" API is deployed and both mobile apps connect to it.

### Tasks

**Infrastructure**
- [ ] Provision VPS (Ubuntu 24.04 LTS, 4GB+ RAM)
- [ ] Install and configure Coolify v4
- [ ] Configure domain and DNS (e.g., `api.snapy.lk`)
- [ ] Verify Traefik auto-SSL is working (HTTPS on domain)
- [ ] Configure firewall (allow ports 22, 80, 443, 8000 for Coolify UI)

**Backend**
- [ ] Initialize Go project (`snapy-api`) with `go mod init`
- [ ] Set up Chi router with middleware stack
- [ ] Configure golangci-lint
- [ ] Create Docker Compose for local development (PostgreSQL + Redis)
- [ ] Create multi-stage production Dockerfile (builder → scratch)
- [ ] Set up sqlc with pgx driver for PostgreSQL
- [ ] Design and create initial database schema (all tables from `08-database-design.md`)
- [ ] Write SQL queries in `db/queries/` and run `sqlc generate`
- [ ] Run initial migration with golang-migrate
- [ ] Create health check endpoint (`GET /health`)
- [ ] Deploy API to Coolify — verify health check responds

**Database**
- [ ] Deploy PostgreSQL 16 container via Coolify with persistent volume
- [ ] Deploy Redis 7 container via Coolify with persistent volume
- [ ] Configure database backups (pg_dump daily cron)
- [ ] Set memory limits on both containers

**Mobile**
- [ ] Initialize iOS project (Xcode, SwiftUI, Swift 5.9+)
- [ ] Initialize Android project (Android Studio, Jetpack Compose, Kotlin 2.0+)
- [ ] Set up basic networking layer in both apps (URLSession / Ktor Client)
- [ ] Both apps successfully call `GET /health` and display response

**CI/CD**
- [ ] Set up GitHub Actions for backend (golangci-lint, go test, sqlc verify on PR)
- [ ] Set up Coolify webhook for auto-deploy on merge to main
- [ ] Set up GitHub Actions for iOS (build + test)
- [ ] Set up GitHub Actions for Android (build + test)

### Exit Criteria
- API deployed and responding at `https://api.snapy.lk/health`
- PostgreSQL and Redis running in Coolify containers
- Both mobile apps display "Connected to Snapy API" on launch
- CI pipeline runs on every PR
- Merge to main triggers auto-deploy

---

## Phase 1: Authentication & Onboarding (Weeks 3-4)

### Goal
Users can sign up and log in via SMS OTP. New users complete onboarding (name, grade selection). Returning users land on home screen.

### Tasks

**Backend - Auth**
- [ ] Integrate ShoutOUT API SDK/REST client
- [ ] `POST /auth/otp/request` — generate 6-digit OTP, store in Redis (5-min TTL), call ShoutOUT to send SMS
- [ ] `POST /auth/otp/verify` — validate OTP, create user if new, issue JWT access + refresh tokens
- [ ] `POST /auth/token/refresh` — validate refresh token, issue new access token
- [ ] Rate limiting middleware: max 3 OTP requests per phone per 10 minutes (Redis counter)
- [ ] JWT middleware: verify access token on protected routes
- [ ] Store refresh tokens (hashed) in PostgreSQL

**Backend - User Profile**
- [ ] `GET /users/me` — return current user profile
- [ ] `PATCH /users/me` — update name, grade, stream
- [ ] Seed grades table (Grade 10, 11, 12, 13)

**iOS - Onboarding**
- [ ] Splash screen
- [ ] Phone number input screen (with Sri Lankan phone format validation: +94...)
- [ ] OTP verification screen (6-digit input, auto-advance, resend button with 30s countdown)
- [ ] Name input screen
- [ ] Grade selection screen (10, 11, 12, 13)
- [ ] Stream selection screen (Science, Commerce, Arts, Technology — shown only for Grade 12-13)
- [ ] JWT token storage in Keychain
- [ ] Auto-login on app launch if valid refresh token exists
- [ ] Token refresh interceptor on 401 responses

**Android - Onboarding**
- [ ] Same screens as iOS (matching feature parity)
- [ ] JWT token storage in EncryptedSharedPreferences
- [ ] Auto-login and token refresh interceptor

### Exit Criteria
- New user can: enter phone → receive OTP → verify → enter name → select grade → land on home screen
- Returning user launches app → auto-logged in → home screen
- Invalid OTP shows error
- Rate limiting blocks excessive OTP requests
- Works on both iOS and Android

---

## Phase 2: Content System & Classic Flashcards (Weeks 5-7)

### Goal
Content hierarchy is populated. Users can browse subjects and units. Classic Recall flashcard mode is fully functional with session flow.

### Tasks

**Backend - Content API**
- [ ] `GET /grades` — list all grades
- [ ] `GET /grades/:id/subjects` — list subjects for a grade
- [ ] `GET /subjects/:id/units` — list units organized by term
- [ ] `GET /units/:id/cards` — get all flashcards in a unit
- [ ] Seed subjects for O/L (Grade 10-11) — all compulsory + common optional subjects
- [ ] Seed subjects for A/L streams (Grade 12-13) — subjects per stream
- [ ] Seed terms (Term 1, 2, 3) for each subject
- [ ] Seed sample units and flashcards for 2-3 subjects (enough for testing)

**Backend - Admin API**
- [ ] `POST /admin/cards` — create flashcard
- [ ] `PUT /admin/cards/:id` — update flashcard
- [ ] `DELETE /admin/cards/:id` — delete flashcard
- [ ] Same CRUD for subjects, units, terms
- [ ] Admin role check middleware (admin flag on user record)

**Backend - Review Tracking**
- [ ] `POST /reviews` — accept batch of card review results (card_id, rating, reviewed_at)
- [ ] Store review history in `card_reviews` table
- [ ] Store/update SRS state in `card_srs_state` table

**iOS - Content Browsing**
- [ ] Home screen with subject grid (icons, names, progress indicators)
- [ ] Subject detail screen with term tabs and unit list
- [ ] Unit progress indicator (percentage of cards reviewed)
- [ ] Pull-to-refresh on content lists

**iOS - Classic Flashcard Mode**
- [ ] Flashcard session screen
- [ ] Card displays question text
- [ ] "Show Answer" button → card flips with 3D rotation animation (300ms)
- [ ] Answer revealed → "Got it" / "Missed it" buttons
- [ ] Progress bar showing current card / total cards
- [ ] Session auto-advances to next card after marking
- [ ] Session summary screen (cards reviewed, correct count, accuracy percentage)
- [ ] Review results sent to `POST /reviews`

**Android - Content Browsing & Classic Flashcards**
- [ ] Same screens and functionality as iOS (feature parity)
- [ ] Card flip animation using Compose animation APIs

**Offline - Initial**
- [ ] Cache unit cards locally when first loaded (SwiftData / Room)
- [ ] Flashcard session works without network (reads from local cache)

### Exit Criteria
- User selects grade → sees relevant subjects → selects subject → browses terms/units
- User opens unit → classic flashcard session begins → reviews all cards → sees summary
- Card flip animation is smooth (60fps)
- Review results are stored on server
- Session works offline if cards were previously loaded
- Admin can create/edit/delete cards via API

---

## Phase 3: MCQ Mode & Spaced Repetition (Weeks 8-10)

### Goal
MCQ flashcard mode is functional. FSRS spaced repetition algorithm is integrated. Cards are scheduled based on memory strength. Quick review mode is available.

### Tasks

**FSRS - Native Per Platform**
- [ ] Implement or integrate FSRS in Swift for iOS (`swift-fsrs` package or ~200 lines native)
- [ ] Implement or integrate FSRS in Kotlin for Android (`fsrs-kt` package or ~200 lines native)
- [ ] Implement or integrate FSRS in Go for backend (`go-fsrs` package)
- [ ] Define shared test vectors (input state + rating → expected output)
- [ ] Run same test cases on all three implementations to verify parity
- [ ] Map Classic mode results: "Got it" → Good (3), "Missed it" → Again (1)
- [ ] Map MCQ mode results: Correct → Good (3), Incorrect → Again (1)

**Backend - Spaced Repetition**
- [ ] Integrate `go-fsrs` for server-side SRS validation
- [ ] `GET /reviews/due` — return cards due for review (where `due_at <= now()`)
  - Ordered by: most overdue first
  - Filterable by subject
  - Include unit and subject info for each card
- [ ] Update `POST /reviews` to validate and store FSRS state from client

**iOS - MCQ Mode**
- [ ] MCQ card layout: question text + 4 answer option buttons
- [ ] Tap answer → instant feedback:
  - Correct: selected option turns green, brief success animation
  - Incorrect: selected option turns red, correct answer highlighted green
- [ ] Optional explanation text shown after answering
- [ ] Auto-advance to next card after 1.5s delay

**iOS - Mixed Mode & FSRS**
- [ ] Mixed sessions: interleave Classic and MCQ cards based on card type
- [ ] Integrate native FSRS (swift-fsrs):
  - After each card review, run FSRS to compute next review date
  - Store updated SRS state locally and sync to server
- [ ] Due cards badge on home screen (number of cards due today)

**iOS - Quick Review**
- [ ] Quick Review button on home screen
- [ ] Duration picker: 5 minutes or 10 minutes
- [ ] Session loads due cards across all subjects
- [ ] Timer displayed during session
- [ ] Session ends when time runs out or all due cards reviewed

**Android - MCQ, Mixed Mode, FSRS, Quick Review**
- [ ] Same functionality as iOS (feature parity)
- [ ] Integrate native FSRS (fsrs-kt) directly as Kotlin dependency

**Offline Enhancement**
- [ ] Queue review results with FSRS state when offline
- [ ] Sync queue to server on reconnect
- [ ] Sync status indicator in UI (synced / pending / syncing)

### Exit Criteria
- MCQ cards display 4 choices with correct/incorrect feedback
- Mixed sessions alternate Classic and MCQ cards within one session
- FSRS schedules cards: easy cards appear later, missed cards appear sooner
- Quick Review surfaces due cards across subjects, respects time limit
- Due cards count visible on home screen
- Offline reviews queue and sync correctly
- FSRS unit tests pass on all three platforms (Swift, Kotlin, Go)

---

## Phase 4: Study Plans & Analytics (Weeks 11-13)

### Goal
Students can create goal-based study plans. Analytics dashboard shows comprehensive study statistics including streak heatmap.

### Tasks

**Backend - Study Plans**
- [ ] `POST /study-plans` — create plan:
  - Input: goal type, target subjects, deadline, daily available minutes
  - Algorithm distributes remaining units across available days
  - Weights: unseen units first, then weak units (low accuracy), then review-heavy units
  - Output: day-by-day task list stored in `study_plan_tasks`
- [ ] `GET /study-plans` — list user's plans
- [ ] `GET /study-plans/:id/today` — get today's tasks with unit details
- [ ] `PATCH /study-plan-tasks/:id` — mark task as completed
- [ ] Plan adjustment: if student falls behind, redistribute remaining tasks

**Backend - Analytics**
- [ ] `GET /analytics/summary` — return:
  - Total cards reviewed (all time)
  - Total study sessions
  - Total study time (estimated from session lengths)
  - Cards mastered (in "Review" state with high stability)
  - Overall accuracy percentage
- [ ] `GET /analytics/streak` — return:
  - Current streak (consecutive active days)
  - Longest streak
  - Daily activity for past 365 days (array of { date, cardsReviewed, accuracy })
  - Used by heatmap and streak display
- [ ] `GET /analytics/weak-areas` — return:
  - Subjects/units ranked by weakness score
  - Weakness score = combination of: low accuracy, high lapse count, many overdue cards
  - Include card counts and accuracy per weak area
- [ ] Update daily_activity table on each review batch sync
- [ ] Update user_streaks table on each review batch sync

**iOS - Study Plans**
- [ ] "Create Study Plan" flow:
  - Select goal (e.g., "Prepare for Term 1 Exam", "Catch up on weak areas")
  - Select subjects to include
  - Set daily study time (15 / 30 / 45 / 60 min)
  - Set deadline (date picker)
  - Plan generated → show summary
- [ ] Study Plan view:
  - Calendar showing planned study days
  - Today's tasks (units to study, estimated time)
  - Tap task → starts flashcard session for that unit
  - Mark completed → progress updates
  - Overall plan progress bar

**iOS - Analytics Dashboard**
- [ ] Streak heatmap:
  - 52-column x 7-row grid (one year of daily activity)
  - Color gradient: grey (no activity) → light green → dark green (high activity)
  - Custom `Canvas` / `Path` rendering for performance
  - Tap a day to see details (cards reviewed, accuracy)
- [ ] Stats summary cards:
  - Current streak (with flame icon)
  - Total cards mastered
  - Overall accuracy
  - Study hours
- [ ] Accuracy by subject (bar chart or line chart)
- [ ] Weak areas section:
  - Cards showing weakest subjects/units
  - "Practice Now" button to start session on weak area

**Android - Study Plans & Analytics**
- [ ] Same functionality as iOS (feature parity)
- [ ] Streak heatmap using Compose `Canvas` API

### Exit Criteria
- Student creates study plan → sees daily tasks → completes tasks → progress tracked
- Analytics dashboard shows accurate streak heatmap, stats, and weak areas
- Streak heatmap renders 365 days of activity
- Weak areas correctly identify struggling subjects/units
- Study plan redistributes if student falls behind

---

## Phase 5: Social Features & Polish (Weeks 14-16)

### Goal
Leaderboard system is live. Push notifications keep students engaged. App is polished with performance optimizations and accessibility.

### Tasks

**Backend - Leaderboard**
- [ ] Scoring system:
  - +1 point per card reviewed
  - +2 bonus for correct answer
  - Streak multiplier: 1.0x (no streak) → 1.5x (7+ day streak) → 2.0x (30+ day streak)
- [ ] Redis sorted sets:
  - `leaderboard:grade:{gradeId}:week:{weekStart}` — grade-level rankings
  - `leaderboard:subject:{subjectId}:week:{weekStart}` — subject-level rankings
  - Updated via `ZINCRBY` on each review batch sync
- [ ] `GET /leaderboard/grade` — top 50 for user's grade + user's rank
- [ ] `GET /leaderboard/subject/:id` — top 50 for subject + user's rank
- [ ] Weekly reset cron job (every Monday 00:00 IST):
  - Archive current week scores to `user_scores` table
  - Clear Redis sorted sets

**Backend - Push Notifications**
- [ ] Store device tokens (APNs token for iOS, FCM token for Android)
- [ ] Daily study reminder: send at user's configured time if no session today
- [ ] Streak-at-risk warning: send in evening if no session and streak > 3 days
- [ ] Integration with APNs (iOS) and FCM (Android) server SDKs

**iOS - Leaderboard**
- [ ] Leaderboard screen with two tabs:
  - Grade leaderboard (all students in user's grade)
  - Subject leaderboard (select subject from dropdown)
- [ ] User's rank highlighted in the list
- [ ] Top 3 with special styling (gold, silver, bronze)
- [ ] Weekly countdown timer showing time until reset
- [ ] Pull-to-refresh

**iOS - Push Notifications**
- [ ] Register for APNs, send device token to backend
- [ ] Notification permission request during onboarding
- [ ] Settings screen: configure reminder time, enable/disable notifications
- [ ] Handle notification tap → open relevant screen (study session, streak info)

**iOS - Polish**
- [ ] Loading states and skeleton screens for all data-dependent views
- [ ] Error states with retry buttons
- [ ] Empty states (no study plans, no reviews yet)
- [ ] Haptic feedback on card interactions
- [ ] Smooth transitions between screens
- [ ] Image and content lazy loading
- [ ] Memory optimization (release cached data for inactive units)

**Android - Leaderboard, Push, Polish**
- [ ] Same functionality as iOS (feature parity)
- [ ] FCM integration for push notifications
- [ ] Material 3 motion transitions

**Accessibility**
- [ ] VoiceOver support (iOS) / TalkBack support (Android) on all screens
- [ ] Dynamic Type / system font scaling respected
- [ ] Minimum touch target sizes (44x44 pt iOS, 48x48 dp Android)
- [ ] Sufficient color contrast ratios (WCAG AA)
- [ ] Screen reader labels on all interactive elements

**Localization Foundation**
- [ ] Extract all UI strings into localization files
- [ ] Sinhala (`si`) translation for all UI strings
- [ ] Tamil (`ta`) translation for all UI strings
- [ ] Language picker in settings (English, Sinhala, Tamil)
- [ ] Note: card content remains in the language it was authored in

### Exit Criteria
- Leaderboard shows real-time rankings per grade and subject
- Weekly reset works correctly
- Push notifications delivered for daily reminders and streak warnings
- App passes accessibility audit (VoiceOver/TalkBack navigation works)
- UI strings available in English, Sinhala, Tamil
- All screens have proper loading, error, and empty states

---

## Phase 6: Content Pipeline & Launch Prep (Weeks 17-20)

### Goal
Admin panel for content management is built. Content is populated for all target subjects. App is tested, optimized, and ready for app store submission.

### Tasks

**Admin Panel**
- [ ] Simple web admin panel (can be a Go-served static frontend, or a separate lightweight tool)
- [ ] Login with admin credentials
- [ ] Content management:
  - Browse/create/edit/delete: grades, subjects, terms, units, cards
  - Bulk card import (CSV upload)
  - Card preview (see how it looks in the app)
- [ ] User management: view users, set admin role
- [ ] Analytics overview: total users, active users, popular subjects

**Content Population**
- [ ] Recruit content creators (teachers, subject matter experts)
- [ ] Define content guidelines:
  - Question format standards
  - MCQ distractor quality guidelines
  - Minimum cards per unit (15-25)
- [ ] Populate O/L subjects:
  - Mathematics (all terms)
  - Science (all terms)
  - English (all terms)
  - At least 3 more compulsory subjects
- [ ] Populate A/L subjects (at least Science stream):
  - Combined Mathematics
  - Physics
  - Chemistry
- [ ] Content review and quality check

**Testing**
- [ ] End-to-end testing on both platforms:
  - Full onboarding flow
  - Content browsing and flashcard sessions (both modes)
  - Spaced repetition scheduling verification
  - Study plan creation and execution
  - Analytics accuracy
  - Leaderboard rankings
  - Offline mode and sync
- [ ] Beta testing with 20-50 real students
  - Collect feedback on UX, content quality, and bugs
  - Iterate on critical issues
- [ ] Performance testing:
  - Simulate 1,000 concurrent users on API
  - Measure response times under load
  - Identify and fix bottlenecks
- [ ] Security audit:
  - Input sanitization on all endpoints
  - JWT security (algorithm, expiry, refresh flow)
  - Rate limiting effectiveness
  - SQL injection prevention (verify sqlc parameterization)
  - No sensitive data in API responses

**Production Hardening**
- [ ] Database backup strategy: daily pg_dump to external storage (S3 or similar)
- [ ] Monitoring: Coolify built-in metrics + Sentry for error tracking
- [ ] Logging: structured JSON logs with request tracing
- [ ] API response compression (gzip)
- [ ] Database connection pooling
- [ ] Redis persistence configuration (RDB snapshots)

**App Store Preparation**
- [ ] iOS:
  - App Store screenshots (6.7" and 6.1" iPhone)
  - App description and keywords
  - Privacy policy URL
  - App Review compliance (data collection disclosure)
  - TestFlight beta distribution
- [ ] Android:
  - Google Play Store screenshots
  - Store listing description
  - Privacy policy
  - Data safety form
  - Internal/closed testing track

### Exit Criteria
- Admin panel operational, content creators can add/edit cards
- Minimum viable content: 6+ O/L subjects + 3 A/L subjects fully populated
- Beta feedback addressed, critical bugs fixed
- Performance: API p95 response time < 200ms under 1,000 concurrent users
- Security audit passed
- App store submissions ready (all assets, descriptions, policies prepared)
- Backups running and verified
- Monitoring and error tracking active

---

## Phase Summary

| Phase | Duration | Key Deliverable |
|-------|----------|----------------|
| **0: Foundation** | Weeks 1-2 | Infrastructure + scaffolding deployed |
| **1: Auth & Onboarding** | Weeks 3-4 | SMS OTP login + onboarding flow |
| **2: Content & Classic Cards** | Weeks 5-7 | Content browsing + classic flashcard mode |
| **3: MCQ & Spaced Repetition** | Weeks 8-10 | MCQ mode + FSRS + quick review |
| **4: Study Plans & Analytics** | Weeks 11-13 | Study plans + analytics dashboard + heatmap |
| **5: Social & Polish** | Weeks 14-16 | Leaderboard + push notifications + accessibility |
| **6: Content & Launch** | Weeks 17-20 | Content populated + tested + app store ready |

---

## Risk Mitigation

| Risk | Impact | Mitigation |
|------|--------|-----------|
| Content creation bottleneck | Delays launch | Start recruiting content creators in Phase 2, don't wait until Phase 6 |
| ShoutOUT SMS delivery failures | Users can't log in | Implement retry logic, consider fallback SMS provider, show "resend" with timer |
| VPS resource exhaustion | App becomes slow/unresponsive | Monitor early, set container memory limits, have vertical scaling plan ready |
| FSRS parity across platforms | Different scheduling on iOS vs Android | Define shared test vectors early, run same tests on all three implementations |
| App store rejection | Delays launch | Follow guidelines strictly, submit for review early with TestFlight/internal track |
| Scope creep | Timeline extends | Strictly follow phase boundaries, defer "nice to haves" to post-launch |
