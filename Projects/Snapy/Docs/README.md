# Snapy

**A spaced-repetition flashcard app for Sri Lankan students, aligned to the national school syllabus.**

Snapy helps O/L (Grade 10-11) and A/L (Grade 12-13) students prepare for exams using scientifically-proven memory techniques. Built as two native mobile apps (iOS + Android) with a shared backend, Snapy brings Duolingo-style engagement to Sri Lankan education.

---

## Features

- **Flashcard Study Sessions** — Classic recall and multiple-choice (MCQ) modes, mixable within sessions
- **Spaced Repetition (FSRS)** — Algorithm runs server-side, schedules reviews at optimal intervals for long-term retention
- **Syllabus-Aligned Content** — Organized by grade, subject, term, and unit following the Sri Lankan curriculum
- **Study Plans** — Goal-based plans (exam prep, catch-up) with daily task scheduling
- **Analytics Dashboard** — Streak heatmap, accuracy trends, weak area identification
- **Leaderboards** — Weekly rankings per grade and subject
- **Quick Review** — Time-boxed sessions for fast memory refreshment
- **Passwordless Login** — SMS OTP via ShoutOUT API
- **Offline Support** — Study without internet, sync when connected

---

## Tech Stack

| Layer | Technology |
|-------|-----------|
| iOS | Swift, SwiftUI, Combine, SwiftData |
| Android | Kotlin, Jetpack Compose, Room, Hilt |
| Backend | Go, Chi router, sqlc |
| Database | PostgreSQL 16, Redis 7 |
| Infrastructure | VPS + Coolify v4, Docker, Traefik |
| CI/CD | GitHub Actions |

---

## Documentation

Read the docs in order for a complete understanding, or jump to specific topics.

### Product & Architecture
| # | Document | Description |
|---|----------|-------------|
| 1 | [App Overview & User Flow](docs/01-app-overview.md) | Features, user flows, Sri Lankan syllabus mapping, content hierarchy |
| 2 | [System Architecture](docs/02-system-architecture.md) | High-level architecture, frontend/backend responsibilities, offline strategy |
| 3 | [Technology Stack](docs/03-tech-stack.md) | Complete tech stack with justifications for each choice |
| 4 | [Implementation Plan](docs/04-implementation-plan.md) | 7-phase roadmap (~20 weeks) with deliverables and exit criteria |

### System Deep Dives
| # | Document | Description |
|---|----------|-------------|
| 5 | [Backend System](docs/05-backend-system.md) | API routes, auth flow, middleware, study plan algorithm, leaderboard |
| 6 | [iOS App](docs/06-ios-app.md) | SwiftUI screens, MVVM architecture, animations, offline support |
| 7 | [Android App](docs/07-android-app.md) | Jetpack Compose screens, MVVM architecture, animations, offline support |
| 8 | [Database Design](docs/08-database-design.md) | PostgreSQL schema, Redis data structures, indexes, migrations |
| 9 | [Spaced Repetition Engine](docs/09-spaced-repetition-engine.md) | FSRS algorithm, server-side implementation, integration with app features |

---

## Glossary

| Term | Definition |
|------|-----------|
| **FSRS** | Free Spaced Repetition Scheduler — the algorithm that determines when to show each flashcard |
| **SRS** | Spaced Repetition System — the general technique of reviewing material at increasing intervals |
| **SM-2** | SuperMemo 2 — the older spaced repetition algorithm that FSRS replaces |
| **sqlc** | SQL code generator — generates type-safe Go code from SQL queries |
| **OTP** | One-Time Password — the verification code sent via SMS for login |
| **MCQ** | Multiple Choice Question — flashcard mode with 4 answer options |
| **O/L** | GCE Ordinary Level — Sri Lankan national exam for Grade 10-11 students |
| **A/L** | GCE Advanced Level — Sri Lankan national exam for Grade 12-13 students |
| **JWT** | JSON Web Token — the authentication token format used for API access |
| **Coolify** | Self-hosted PaaS for deploying and managing Docker containers on a VPS |

---

## Project Structure

```
Repositories:
├── snapy-api/              # Go backend API
│   ├── cmd/api/            # Application entry point
│   ├── internal/           # Handlers, services, repositories
│   └── db/                 # Migrations, queries (sqlc), seed data
├── snapy-ios/              # iOS app (Swift/SwiftUI)
└── snapy-android/          # Android app (Kotlin/Compose)

Documentation:
└── snpaydocs/              # This repository
    ├── README.md           # You are here
    └── docs/               # All documentation
```
