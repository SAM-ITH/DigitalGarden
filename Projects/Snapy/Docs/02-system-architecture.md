# Snapy - System Architecture

## High-Level Architecture

```
┌──────────────────┐     ┌──────────────────┐
│   iOS App         │     │  Android App      │
│   (Swift/SwiftUI) │     │  (Kotlin/Compose) │
│                   │     │                   │
│  ┌──────────────┐ │     │  ┌──────────────┐ │
│  │ swift-fsrs   │ │     │  │ fsrs-kt      │ │
│  │ (native FSRS)│ │     │  │ (native FSRS)│ │
│  └──────────────┘ │     │  └──────────────┘ │
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
          │   go-fsrs           │
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

### FSRS Algorithm (Native Per Platform)
- Execute the FSRS spaced repetition algorithm on-device for instant scheduling feedback
- iOS uses `swift-fsrs` (native Swift), Android uses `fsrs-kt` (native Kotlin)
- Compute next review dates immediately after each card review
- No network round-trip needed for scheduling decisions

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

### User Progress Sync
- Receive batched card review results from mobile apps
- Store authoritative SRS state (stability, difficulty, due dates) in PostgreSQL
- Resolve conflicts when offline reviews sync (server timestamp wins)
- Serve "due cards" lists for quick review mode

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
- After a card review, the client immediately updates local SRS state (via FSRS)
- Review results are queued and sent to server in batches
- Server stores the authoritative state
- If server rejects (rare), client re-syncs from server

---

## Offline Strategy

### Card Content Caching
- When a student opens a unit for the first time, all cards in that unit are cached locally
- Cached content includes: question text, answer text, MCQ choices, explanations
- Cache is invalidated when server reports a newer content version

### Offline Study Sessions
- Students can complete full study sessions without connectivity
- FSRS runs locally — scheduling decisions don't require the server
- Review results are stored in a local queue table

### Sync on Reconnect
```
App detects connectivity restored
  → Read all queued review results from local DB
  → POST /reviews (batch upload)
  → Server processes and stores
  → Server responds with any state corrections
  → Client clears queue and applies corrections
  → Sync indicator changes to "synced"
```

### Conflict Resolution
- Each card SRS state has a `lastReviewedAt` timestamp
- On sync, if server has a newer timestamp than the queued review, server state wins
- This handles the edge case where a user studies on two devices

---

## FSRS Strategy: Native Per Platform

### Approach
Each platform implements FSRS natively using its own language:

```
┌──────────────┐  ┌──────────────┐  ┌──────────────┐
│  iOS App     │  │ Android App  │  │  Go Backend  │
│              │  │              │  │              │
│ swift-fsrs   │  │  fsrs-kt     │  │  go-fsrs     │
│ (Swift)      │  │  (Kotlin)    │  │  (Go)        │
└──────────────┘  └──────────────┘  └──────────────┘
       │                 │                 │
       └─────────────────┼─────────────────┘
                         │
              Shared Test Vectors
         (same inputs → same outputs)
```

- **iOS**: Uses `swift-fsrs` package or ~200 lines of native Swift implementation
- **Android**: Uses `fsrs-kt` package or ~200 lines of native Kotlin implementation
- **Backend**: Uses `go-fsrs` for server-side SRS state validation

### Why Not a Shared Library (KMP)?
- FSRS is a well-defined mathematical algorithm (~200 lines of core logic)
- The algorithm is deterministic — same inputs always produce same outputs
- Existing open-source implementations are available in all three languages
- KMP would add significant complexity (build setup, SKIE bridging for iOS, separate repo, CI pipeline) for minimal benefit
- Each implementation is verified with the same test vectors to ensure parity

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
