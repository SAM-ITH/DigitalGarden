# Snapy - Technology Stack

## Stack Overview

| Layer            | Technology               | Version          |
| ---------------- | ------------------------ | ---------------- |
| iOS App          | Swift + SwiftUI          | Swift 5.9+       |
| Android App      | Kotlin + Jetpack Compose | Kotlin 2.0+      |
| Backend Runtime  | Go                       | 1.22+            |
| Backend Router   | Chi / Gin                | Latest           |
| SQL Code Gen     | sqlc                     | Latest           |
| Validation       | go-playground/validator  | v10              |
| Primary Database | PostgreSQL               | 16               |
| Cache / Realtime | Redis                    | 7                |
| SMS OTP          | ShoutOUT API             | -                |
| Deployment       | Coolify v4 on VPS        | 4.x              |
| Containerization | Docker                   | Latest           |
| Reverse Proxy    | Traefik                  | v3 (via Coolify) |
| CI/CD            | GitHub Actions           | -                |

---

## iOS App

### Core
| Technology | Purpose | Justification |
|-----------|---------|---------------|
| **Swift 5.9+** | Programming language | Native iOS language, best performance, full platform API access, modern concurrency with async/await |
| **SwiftUI** | UI framework | Declarative UI, built-in animations, live previews, the modern standard for iOS development |
| **Combine** | Reactive framework | Native reactive streams for data binding, async event handling, pairs naturally with SwiftUI |

### Architecture & State
| Technology | Purpose | Justification |
|-----------|---------|---------------|
| **MVVM Pattern** | Architecture | Clean separation of views and business logic, native fit for SwiftUI's data binding |
| **@Observable / ObservableObject** | State management | SwiftUI's native state management, minimal boilerplate, automatic UI updates |
| **Swift Concurrency (async/await)** | Async operations | Modern structured concurrency, replaces callback chains, built into the language |

### Data & Networking
| Technology | Purpose | Justification |
|-----------|---------|---------------|
| **SwiftData** | Local database | Apple's modern persistence framework, seamless SwiftUI integration, replaces CoreData with cleaner API. Used for caching flashcard content and queuing offline reviews. |
| **URLSession** | HTTP client | Native networking, no third-party dependency needed. Supports interceptors via URLProtocol for JWT refresh logic. |
| **JSONDecoder/Codable** | JSON parsing | Native Swift protocol, zero-overhead serialization |

### Platform Services
| Technology | Purpose | Justification |
|-----------|---------|---------------|
| **Keychain Services** | Secure storage | iOS secure enclave for JWT tokens, industry standard |
| **APNs (Apple Push Notification service)** | Push notifications | Required for iOS push — daily reminders, streak alerts |
| **SwiftUI Navigation (NavigationStack)** | Routing | Native navigation with type-safe paths, deep link support |

### FSRS Integration
| Technology | Purpose | Justification |
|-----------|---------|---------------|
| **swift-fsrs** (or native implementation) | Spaced repetition | Native Swift FSRS implementation. The algorithm is ~200 lines of deterministic math — simple to implement or use an existing Swift package. No cross-platform bridge needed. |

---

## Android App

### Core
| Technology | Purpose | Justification |
|-----------|---------|---------------|
| **Kotlin 2.0+** | Programming language | Official Android language, null safety, coroutines |
| **Jetpack Compose** | UI framework | Modern declarative UI, powerful animation APIs, Google's recommended approach for new Android apps |
| **Material 3** | Design system | Latest Material Design components, dynamic theming, consistent Android look and feel |

### Architecture & State
| Technology | Purpose | Justification |
|-----------|---------|---------------|
| **MVVM Pattern** | Architecture | Matches the ViewModel-centric Jetpack architecture, consistent with iOS app pattern |
| **Kotlin Coroutines + Flow** | Async & reactive | First-class async support, Flow for reactive streams, structured concurrency |
| **Hilt** | Dependency injection | Android standard DI, compile-time safety, clean ViewModel injection |

### Data & Networking
| Technology | Purpose | Justification |
|-----------|---------|---------------|
| **Room** | Local database | Jetpack's SQLite abstraction, compile-time query verification, Flow integration for reactive queries. Used for caching flashcard content and queuing offline reviews. |
| **Ktor Client** | HTTP client | Kotlin-native HTTP client, built-in serialization support, lightweight |
| **Kotlinx Serialization** | JSON parsing | Kotlin-native, compile-time safe, works seamlessly with Ktor |

### Platform Services
| Technology | Purpose | Justification |
|-----------|---------|---------------|
| **EncryptedSharedPreferences** | Secure storage | Android Jetpack Security library for JWT token storage |
| **Firebase Cloud Messaging (FCM)** | Push notifications | Standard Android push notification service |
| **Compose Navigation** | Routing | Type-safe navigation for Compose, deep link support |
| **WorkManager** | Background sync | Jetpack library for reliable background work — syncing offline reviews when connectivity returns |

### FSRS Integration
| Technology | Purpose | Justification |
|-----------|---------|---------------|
| **fsrs-kt** (or native implementation) | Spaced repetition | Native Kotlin FSRS implementation. Same ~200 lines of deterministic math. Unit tests verify parity with the Swift and Go implementations. |

---

## Backend (Go)

### Why Go Over Node.js

| Factor | Node.js | Go |
|--------|---------|-----|
| **Memory footprint** | 150-300MB baseline | 10-30MB baseline |
| **Deployment artifact** | Node runtime + node_modules | Single static binary (~10-15MB) |
| **Concurrency model** | Single-threaded event loop | Goroutines (true parallelism, millions of lightweight threads) |
| **Cold start** | ~500ms | ~10ms |
| **Type safety** | TypeScript (compile-time, not runtime) | Built into the language with compiler enforcement |
| **Docker image size** | ~200MB (node:alpine + deps) | ~15MB (scratch + binary) |
| **VPS fit** | Needs dedicated RAM headroom | Leaves more RAM for PostgreSQL and Redis |

Go is the better fit for a VPS-hosted API where resources are shared with the database. The single-binary deployment, tiny memory footprint, and true concurrency make it ideal for Coolify/Docker deployment on a 4GB VPS.

### Core
| Technology | Purpose | Justification |
|-----------|---------|---------------|
| **Go 1.22+** | Language & runtime | Compiled, garbage-collected, excellent for API servers. Built-in HTTP server, JSON handling, crypto (JWT). Minimal external dependencies. Goroutines handle thousands of concurrent requests efficiently. |
| **Chi** | HTTP router | Lightweight, idiomatic Go router. stdlib `net/http` compatible. Composable middleware. Chosen over Gin (more opinionated, larger surface area) for simplicity. |
| **sqlc** | SQL code generation | Generates type-safe Go code from SQL queries. Write SQL, get Go functions with proper types. Zero runtime overhead — it's just compiled Go code. Chosen over GORM (heavy ORM, query abstraction) because explicit SQL is simpler and faster. |
| **golang-migrate** | Database migrations | Standard Go migration tool. SQL-based migrations stored in version control. Can run programmatically or via CLI. |
| **go-playground/validator** | Request validation | Struct tag-based validation. Validate request bodies with struct tags like `validate:"required,min=6"`. |

### Authentication & Security
| Technology | Purpose | Justification |
|-----------|---------|---------------|
| **golang-jwt/jwt** | JWT library | Standard Go JWT implementation. Signs and verifies tokens. |
| **ShoutOUT API** | SMS OTP delivery | Sri Lanka-local SMS provider, simple REST API, reliable local number support. Called via Go's `net/http` client. |
| **crypto/sha256** | Token hashing | Standard library for hashing refresh tokens before storage |
| **golang.org/x/time/rate** | Rate limiting | Standard Go rate limiter. Combined with Redis counters for distributed rate limiting. |

### Infrastructure Libraries
| Technology | Purpose | Justification |
|-----------|---------|---------------|
| **go-redis/redis** | Redis client | Feature-complete Redis client for Go, connection pooling, pipeline support |
| **pgx** | PostgreSQL driver | High-performance PostgreSQL driver for Go, used by sqlc as the underlying driver |
| **zerolog** | Logging | Zero-allocation JSON logger. Fast structured logging for production. |
| **go-fsrs** | FSRS (server-side) | Go implementation of FSRS algorithm for server-side SRS state validation. |

---

## Database

### PostgreSQL 16
**Role**: Primary persistent data store for all relational data.

**Why PostgreSQL:**
- Rock-solid ACID compliance for financial-grade data integrity
- JSONB support for flexible card content (different question types, MCQ choices)
- Full-text search capability (future: search across cards)
- Excellent performance at scale with proper indexing
- 30+ years of production hardening
- Free and open source

**What it stores:**
- User accounts and profiles
- Content hierarchy (grades, subjects, terms, units, cards)
- Card review history and SRS state
- Study plans and daily tasks
- User scores, streaks, and daily activity

### Redis 7
**Role**: Cache, ephemeral data, and real-time data structures.

**Why Redis:**
- **Leaderboard sorted sets**: `ZADD`, `ZREVRANK`, `ZREVRANGE` give O(log n) rank operations — purpose-built for leaderboards
- **OTP storage**: key-value with TTL — OTP codes auto-expire after 5 minutes, no cleanup needed
- **Rate limiting**: `INCR` with TTL for sliding window rate limiting on OTP requests
- In-memory speed for high-frequency reads (leaderboard queries, session checks)

---

## Infrastructure & Deployment

### VPS
| Spec | Recommendation |
|------|---------------|
| **OS** | Ubuntu 24.04 LTS |
| **RAM** | 4GB minimum (8GB recommended for growth) |
| **CPU** | 2-4 vCPU |
| **Storage** | 40GB+ SSD |
| **Provider** | Any (DigitalOcean, Hetzner, Linode, etc.) |

### Coolify v4
**Role**: Self-hosted PaaS that manages the entire deployment stack.

**Why Coolify:**
- Open-source alternative to Heroku/Vercel for self-hosted infrastructure
- Manages Docker containers with a web UI
- Automatic SSL via Traefik + Let's Encrypt
- Zero-downtime rolling deployments
- Git-based deployments (push to main → auto-deploy)
- Built-in monitoring and log aggregation
- Database backup management
- Environment variable management with secrets

### Docker
Each service runs in its own container:
```
snapy-api       → Go binary (scratch/alpine image, ~15MB)
postgres        → PostgreSQL 16 with persistent volume
redis           → Redis 7 with persistent volume
```

### Domain & SSL
- Domain pointed to VPS IP
- Traefik (managed by Coolify) handles:
  - SSL certificate provisioning (Let's Encrypt)
  - Automatic renewal
  - HTTPS redirect
  - Reverse proxy routing to API container

---

## CI/CD Pipeline

### GitHub Actions (on every push/PR)
```
Trigger: Push to any branch / PR opened
  → Checkout code
  → Set up Go toolchain
  → Run linter (golangci-lint)
  → Run tests (go test ./...)
  → Run sqlc verify (check SQL/Go type consistency)
  → Report results
```

### Deployment (on merge to main)
```
Trigger: Merge to main branch
  → GitHub Actions: full test suite passes
  → Webhook notifies Coolify
  → Coolify:
    → Pulls latest code
    → Builds Docker image (multi-stage: build Go binary, copy to scratch)
    → Runs database migrations (golang-migrate)
    → Rolling deployment (new container starts, health check passes, old container stops)
```

### Mobile App CI
```
iOS:
  → GitHub Actions with macOS runner
  → Build with Xcode
  → Run XCTest suite
  → (Release) Archive and upload to App Store Connect via Fastlane

Android:
  → GitHub Actions with Linux runner
  → Build with Gradle
  → Run instrumented tests
  → (Release) Build AAB and upload to Google Play via Fastlane
```

---

## FSRS Strategy: Native Per Platform

Instead of sharing FSRS via a cross-platform layer (like KMP), each platform implements FSRS natively:

| Platform | FSRS Library | Language |
|----------|-------------|---------|
| iOS | `swift-fsrs` or native implementation | Swift |
| Android | `fsrs-kt` or native implementation | Kotlin |
| Backend | `go-fsrs` | Go |

**Why this approach:**
- FSRS is a well-defined mathematical algorithm (~200 lines of core logic)
- The algorithm is deterministic — same inputs always produce same outputs
- Existing open-source implementations are available in all three languages
- Each implementation can be verified with the same unit test cases
- Eliminates the complexity of KMP (build setup, SKIE bridging, separate repo, CI complexity)
- No cross-platform dependency management

**How to ensure consistency:**
- Define a shared set of test vectors (input state + rating → expected output state)
- Run the same test cases on all three implementations
- If all tests pass, the implementations are functionally identical

---

## Development Environment

### Repository Structure
```
Repositories:
├── snapy-api/                # Go backend
│   ├── cmd/api/              # Application entry point
│   ├── internal/             # Private application code
│   │   ├── handler/          # HTTP handlers
│   │   ├── middleware/       # Middleware (auth, rate-limit, logging)
│   │   ├── service/          # Business logic
│   │   ├── repository/       # Database queries (sqlc generated)
│   │   └── model/            # Domain types
│   ├── db/
│   │   ├── migrations/       # SQL migration files
│   │   ├── queries/          # SQL queries for sqlc
│   │   └── seed/             # Seed data
│   ├── sqlc.yaml             # sqlc configuration
│   ├── Dockerfile
│   ├── go.mod
│   └── go.sum
│
├── snapy-ios/                # iOS app (Swift/SwiftUI)
│   ├── Snapy/
│   │   ├── Views/
│   │   ├── ViewModels/
│   │   ├── Models/
│   │   ├── Services/
│   │   ├── FSRS/             # Native Swift FSRS implementation
│   │   └── SnapyApp.swift
│   ├── Snapy.xcodeproj
│   └── SnapyTests/
│
└── snapy-android/            # Android app (Kotlin/Compose)
    ├── app/
    │   ├── src/main/
    │   │   ├── java/.../snapy/
    │   │   │   ├── ui/
    │   │   │   ├── viewmodel/
    │   │   │   ├── data/
    │   │   │   ├── fsrs/     # Native Kotlin FSRS implementation
    │   │   │   └── di/
    │   │   └── res/
    │   └── build.gradle.kts
    ├── build.gradle.kts
    └── settings.gradle.kts
```

### Why Separate Repos
- Each project has a fundamentally different build system (Go, Xcode, Gradle)
- Separate CI/CD pipelines per platform
- Team members can work on one platform without pulling the others
- Simpler than monorepo for 3 different languages

### Development Tools
| Tool | Purpose |
|------|---------|
| **golangci-lint** | Go linting (aggregates multiple linters) |
| **sqlc** | SQL → Go code generation |
| **golang-migrate** | Database migration management |
| **go test** | Backend unit and integration tests |
| **XCTest** | iOS unit and UI tests |
| **JUnit + Compose Test** | Android unit and UI tests |
| **SwiftLint** | iOS code style enforcement |
| **Detekt** | Kotlin static analysis |
| **Docker Compose** | Local development environment (PostgreSQL + Redis) |
| **Air** | Go hot-reload for development (watches files, rebuilds on change) |
