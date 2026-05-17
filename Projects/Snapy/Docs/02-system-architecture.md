# Snapy - System Architecture

## High-Level Architecture

```
┌──────────────────┐     ┌──────────────────┐
│   iOS App         │     │  Android App      │
│   (Swift/SwiftUI) │     │  (Kotlin/Compose) │
│   (UI + cache     │     │  (UI + cache      │
│    only)           │     │   only)           │
└────────┬──────────┘     └────────┬──────────┘
         │           HTTPS          │
         └──────────┬───────────────┘
                    │
          ┌─────────▼──────────┐
          │   Traefik Reverse  │
          │   Proxy (SSL)      │
          └─────────┬──────────┘
                    │
          ┌─────────▼──────────┐
          │   Backend API       │
          │   (Go / Chi)        │
          │   go-fsrs           │  ← FSRS runs ONLY here
          └───┬─────────┬──────┘
              │         │
    ┌─────────▼───┐ ┌───▼─────────┐
    │ PostgreSQL  │ │   Redis     │
    │ 16          │ │   7         │
    │ (Primary DB)│ │ (Cache/     │
    │             │ │  Leaderboard│
    └─────────────┘ │  /OTP)     │
                    └─────────────┘

          External Services:
          ┌─────────────────┐
          │ ShoutOUT SMS API│ ← OTP delivery
          │ APNs            │ ← iOS push notifications
          │ FCM             │ ← Android push notifications
          └─────────────────┘
```

All services run on a single VPS managed by **Coolify v4** (self-hosted PaaS).

---

## Frontend Responsibilities (iOS & Android Apps)

Both native apps handle:

### UI & Presentation
- All screen rendering and navigation
- Flashcard animations (3D flip, slide transitions, feedback pulses)
- Streak heatmap rendering (custom canvas/painter)
- Progress bars, charts, and data visualizations
- Theme and responsive layout management

### Local State & Caching
- Cache flashcard content locally for offline access (SwiftData on iOS, Room on Android)
- Store JWT access and refresh tokens securely (Keychain on iOS, EncryptedSharedPreferences on Android)
- Maintain active session state (current card index, answers, timer)

### Study Session Management
- Manage active session state (current card index, answers, timer)
- Display cards and collect user ratings (correct/incorrect)
- Queue review results for batch upload to server
- FSRS scheduling is handled entirely server-side — no client-side algorithm needed

### Offline Queue
- Queue card review results when offline
- Sync queued reviews to server when connectivity returns
- Display sync status indicator to user

### Input Validation
- Validate user input before sending to API (phone format, OTP length, etc.)
- Provide immediate feedback for invalid input

---

## Backend Responsibilities (Go API)

### Authentication
- Accept OTP requests, generate codes, store in Redis with TTL
- Call ShoutOUT API to deliver SMS
- Verify OTP codes and issue JWT access + refresh token pairs
- Handle token refresh flow
- Rate limit OTP requests per phone number

### Content API
- Serve grade, subject, term, and unit listings
- Serve flashcard content (questions, answers, choices) for requested units
- Support content versioning for cache invalidation
- Content is read-only for regular users

### FSRS Scheduling (Server-Side Only)
- Run the FSRS spaced repetition algorithm on every review batch sync
- Compute new stability, difficulty, and next due date for each reviewed card
- Store authoritative SRS state in PostgreSQL (`card_srs_state` table)
- Serve "due cards" lists based on server-computed due dates
- Single source of truth — no client-side FSRS implementations needed
- Eliminates cross-platform consistency concerns and conflict resolution

### Study Plan Engine
- Generate study plans based on user goals, subjects, timeline, and daily availability
- Compute daily task assignments weighted by weak areas and unseen content
- Track plan completion progress
- Adjust remaining plan if student falls behind

### Analytics Aggregation
- Compute streak data (current streak, longest streak, daily activity heatmap)
- Calculate per-subject accuracy trends
- Identify weak areas (units with high lapse rate or low accuracy)
- Aggregate total statistics (cards mastered, reviews completed, study hours)

### Leaderboard
- Compute and cache leaderboard rankings using Redis sorted sets
- Support grade-level and subject-level leaderboards
- Weekly reset cycle
- Serve user rank, top N, and nearby ranks

### Admin API
- CRUD operations for grades, subjects, terms, units, and cards
- Protected by admin role check
- Used by content management panel

---

## Database Responsibilities

### PostgreSQL (Primary Data Store)
All persistent, relational data:
- User accounts and profiles
- Authentication tokens (refresh tokens)
- Content hierarchy (grades, subjects, terms, units, cards)
- Card review history
- SRS state per user per card
- Study plans and daily tasks
- User scores and streaks
- Daily activity records

### Redis (Cache & Real-Time Data)
Ephemeral and high-frequency access data:
- **OTP codes**: stored with 5-minute TTL, auto-expire
- **Rate limiting**: counters per phone number for OTP requests
- **Leaderboard sorted sets**: per grade and per subject, updated on each review sync
- **Session cache**: optional JWT session metadata for quick validation

---

## Communication Patterns

### REST API (JSON over HTTPS)
- All client-server communication uses REST endpoints
- JSON request/response bodies
- HTTPS enforced via Traefik auto-SSL (Let's Encrypt)

### Authentication
- **Access Token**: JWT, 15-minute expiry, contains `userId` and `gradeId`
- **Refresh Token**: opaque token, 30-day expiry, stored hashed in PostgreSQL
- Access token sent in `Authorization: Bearer <token>` header
- On 401 response, client uses refresh token to get new access token
- If refresh fails, client redirects to login

### Request Flow
```
Client → HTTPS → Traefik (SSL termination) → Go API → PostgreSQL/Redis
```

### Optimistic Updates
- After a card review, the client records the result locally (card ID + rating + timestamp)
- Review results are queued and sent to server in batches
- Server runs FSRS and stores the authoritative SRS state
- Client does not compute any scheduling — it relies on the server for due dates
- When the client needs to know which cards are due, it fetches from the server

---

## Offline Strategy

### Card Content Caching
- When a student opens a unit for the first time, all cards in that unit are cached locally
- Cached content includes: question text, answer text, MCQ choices, explanations
- Cache is invalidated when server reports a newer content version

### Offline Study Sessions
- Students can study any cached unit without connectivity
- The app shows all cards in the unit regardless of SRS state (since the server isn't reachable)
- Review results (card ID + rating + timestamp) are stored in a local queue table

### Sync on Reconnect
```
App detects connectivity restored
  → Read all queued review results from local DB
  → POST /reviews (batch upload with raw ratings only)
  → Server runs FSRS for each review, computes and stores SRS state
  → Server responds with updated due card counts
  → Client clears queue
  → Sync indicator changes to "synced"
```

### Quick Review Offline Limitation
- Quick Review (reviewing due cards across all subjects) **requires connectivity**
- The server determines which cards are due — the phone cannot compute this locally
- Offline study is limited to browsing and studying cached units
- This is an intentional tradeoff for simpler architecture

---

## FSRS Strategy: Server-Side Only

### Approach
The FSRS algorithm runs exclusively on the Go backend. Mobile apps send raw review results (card ID + rating + timestamp) and the server computes all scheduling.

```
┌──────────────┐  ┌──────────────┐  ┌──────────────┐
│  iOS App     │  │ Android App  │  │  Go Backend  │
│              │  │              │  │              │
│  (UI only,   │  │  (UI only,   │  │  go-fsrs     │
│   sends raw  │  │   sends raw  │  │  (FSRS runs  │
│   ratings)   │  │   ratings)   │  │   here ONLY) │
└──────┬───────┘  └──────┬───────┘  └──────────────┘
       │                  │                 ▲
       └──────────────────┘                 │
              POST /reviews                 │
         (cardId, rating, timestamp) ───────┘
              Server computes:
              - new stability
              - new difficulty
              - next due date
```

### Why Server-Only
- **Single implementation**: One FSRS codebase in Go, no need for Swift or Kotlin versions
- **No cross-platform consistency concerns**: No shared test vectors, no parity verification
- **Single source of truth**: Server is the authority for all SRS state — no conflict resolution needed
- **Simpler mobile code**: Apps just collect ratings and send them, no algorithm logic
- **Easier to update**: Algorithm changes only need to be made in one place
- **Reduced network payload**: No SRS state fields sent from client to server

### Tradeoff
- **Quick Review requires connectivity**: The phone cannot determine which cards are due without asking the server
- **Offline study**: Works for browsing cached units, but due-date-based review requires internet
- **Acceptable tradeoff**: Sri Lankan students typically have mobile data available most of the time

### What's Platform-Specific (Everything Else)
- UI layer (SwiftUI vs Jetpack Compose)
- Local database (SwiftData vs Room)
- Network layer (URLSession vs Ktor)
- Push notifications (APNs vs FCM)
- Secure storage (Keychain vs EncryptedSharedPreferences)
- Platform-specific animations

---

## Deployment Architecture (Coolify on VPS)

```
VPS (Ubuntu 24.04 LTS, 4GB+ RAM)
├── Coolify v4 (management UI on port 8000)
├── Traefik (reverse proxy, ports 80/443, auto-SSL)
├── Docker Container: snapy-api (Go binary, ~15MB image)
│   └── Port 8080 (internal)
├── Docker Container: PostgreSQL 16
│   └── Port 5432 (internal)
├── Docker Container: Redis 7
│   └── Port 6379 (internal)
└── Docker volumes for persistent data
    ├── postgres-data
    └── redis-data
```

### Deployment Flow
```
Developer pushes to main branch
  → GitHub Actions: lint + test
  → On success: webhook triggers Coolify
  → Coolify pulls latest code
  → Builds Docker image
  → Rolling deployment (zero downtime)
  → Health check confirms new container is ready
  → Old container removed
```

### Resource Allocation (Recommended)
| Container | RAM Limit | CPU |
|-----------|-----------|-----|
| snapy-api (Go) | 128MB | 0.5 core |
| PostgreSQL | 2GB | 1 core |
| Redis | 256MB | 0.5 core |
| Coolify + Traefik | 1GB | 0.5 core |
| OS overhead | 512MB | - |
| **Total** | **~4GB** | **2.5 cores** |

Go's tiny memory footprint (~20-30MB actual usage) means more RAM for PostgreSQL, which benefits most from it.

---

## Security Considerations

- All traffic encrypted via HTTPS (Traefik + Let's Encrypt)
- JWT tokens have short expiry (15 min access, 30 day refresh)
- Refresh tokens stored hashed in database
- OTP codes auto-expire in Redis (5 min TTL)
- Rate limiting on OTP requests (3 per phone per 10 min)
- All SQL queries parameterized via sqlc (SQL injection prevention)
- Input validation on all API endpoints via go-playground/validator
- Admin routes protected by role-based access control
- Mobile: JWT stored in Keychain (iOS) / EncryptedSharedPreferences (Android)
- CORS configured to reject unauthorized origins
- Docker containers run with minimal privileges and memory limits
