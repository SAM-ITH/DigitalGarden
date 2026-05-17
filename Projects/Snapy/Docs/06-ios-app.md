# Snapy - iOS App: Detailed Design

## Overview

The Snapy iOS app is built with **Swift** and **SwiftUI**, following the **MVVM** architecture pattern. FSRS runs server-side only — the app sends raw ratings and the server handles all scheduling.

---

## Architecture

```
┌──────────────────────────────────────────────┐
│                   Views (SwiftUI)             │
│  Screens, Components, Animations              │
└──────────────────┬───────────────────────────┘
                   │ Observes
┌──────────────────▼───────────────────────────┐
│              ViewModels (@Observable)          │
│  Business logic, state management             │
└──────────────────┬───────────────────────────┘
                   │
┌──────────────────▼───────────────────────────┐
│               Services                        │
│  (Network, DB, Auth, Push)                    │
└──────────────────┬───────────────────────────┘
                   │
┌──────────────────▼───────────────────────────┐
│          Data Layer                           │
│  URLSession (API)  │  SwiftData (Local DB)    │
│  Keychain (Tokens) │                          │
└──────────────────────────────────────────────┘
```

### Key Architecture Principles
- **MVVM**: Views observe ViewModels, ViewModels coordinate services
- **@Observable macro** (Swift 5.9+): Simpler than ObservableObject, automatic change tracking
- **Dependency Injection**: Services injected via environment or init parameters
- **Async/Await**: All async operations use Swift Concurrency (no Combine for data flow, Combine only where needed for legacy APIs)
- **Server-side FSRS**: The server handles all spaced repetition scheduling. The app only collects ratings and sends them to the server.

---

## Screen Inventory

### Onboarding Flow
| Screen | SwiftUI View | Description |
|--------|-------------|-------------|
| Splash | `SplashView` | App logo, auto-login check |
| Welcome | `WelcomeView` | App introduction, "Get Started" button |
| Phone Input | `PhoneInputView` | Phone number entry with +94 prefix |
| OTP Verify | `OTPVerifyView` | 6-digit OTP input, auto-advance, resend timer |
| Name Input | `NameInputView` | User's name entry |
| Grade Select | `GradeSelectView` | Grade picker (10, 11, 12, 13) |
| Stream Select | `StreamSelectView` | A/L stream picker (shown only for Grade 12-13) |

### Main App (Tab-Based)
| Tab | Screen | SwiftUI View | Description |
|-----|--------|-------------|-------------|
| Study | Home | `HomeView` | Subject grid, streak indicator, quick review button |
| Study | Subject Detail | `SubjectDetailView` | Term tabs, unit list with progress |
| Study | Flashcard Session | `FlashcardSessionView` | Active study session |
| Study | Session Summary | `SessionSummaryView` | Post-session results |
| Study Plan | Plan List | `StudyPlanListView` | User's plans, create new |
| Study Plan | Create Plan | `CreatePlanView` | Goal, subjects, time, deadline |
| Study Plan | Plan Detail | `PlanDetailView` | Calendar, daily tasks |
| Analytics | Dashboard | `AnalyticsDashboardView` | Stats, heatmap, charts |
| Leaderboard | Rankings | `LeaderboardView` | Grade and subject tabs |
| Profile | Settings | `ProfileView` | User info, preferences, logout |

### Navigation Structure
```swift
TabView {
    NavigationStack {
        HomeView()
            → SubjectDetailView(subject)
                → FlashcardSessionView(unit)
                    → SessionSummaryView(results)
    }
    .tabItem { Label("Study", systemImage: "book.fill") }
    
    NavigationStack {
        StudyPlanListView()
            → CreatePlanView()
            → PlanDetailView(plan)
    }
    .tabItem { Label("Plan", systemImage: "calendar") }
    
    NavigationStack {
        AnalyticsDashboardView()
    }
    .tabItem { Label("Stats", systemImage: "chart.bar.fill") }
    
    NavigationStack {
        LeaderboardView()
    }
    .tabItem { Label("Ranks", systemImage: "trophy.fill") }
    
    NavigationStack {
        ProfileView()
    }
    .tabItem { Label("Profile", systemImage: "person.fill") }
}
```

---

## State Management

### ViewModels

```swift
// Example: HomeViewModel
@Observable
class HomeViewModel {
    var subjects: [Subject] = []
    var dueCardsCount: Int = 0
    var currentStreak: Int = 0
    var isLoading: Bool = false
    var error: AppError?
    
    private let contentService: ContentService
    private let reviewService: ReviewService
    private let analyticsService: AnalyticsService
    
    func loadHome() async {
        isLoading = true
        defer { isLoading = false }
        
        async let subjectsResult = contentService.getSubjects(gradeId: userGradeId)
        async let dueResult = reviewService.getDueCardsCount()
        async let streakResult = analyticsService.getCurrentStreak()
        
        // Parallel fetch
        subjects = (try? await subjectsResult) ?? []
        dueCardsCount = (try? await dueResult) ?? 0
        currentStreak = (try? await streakResult) ?? 0
    }
}
```

### Key ViewModels
| ViewModel | Responsibilities |
|-----------|-----------------|
| `AuthViewModel` | Login flow, OTP handling, token management, auto-login |
| `HomeViewModel` | Subject list, due cards count, streak, quick review entry |
| `SubjectDetailViewModel` | Terms, units, progress per unit |
| `FlashcardSessionViewModel` | Active session state, card queue, FSRS integration, review recording |
| `StudyPlanViewModel` | Plan creation, daily tasks, completion tracking |
| `AnalyticsViewModel` | Stats loading, heatmap data, weak areas |
| `LeaderboardViewModel` | Rankings data, tab switching (grade/subject) |
| `ProfileViewModel` | User info, settings, logout |

---

## Flashcard Session UX

### Session Flow
```
User taps unit → FlashcardSessionView
  │
  ├── Load cards from local cache (or fetch from API if not cached)
  ├── Shuffle or display in order
  ├── Display progress bar (card X of N)
  │
  ├── For each card:
  │   ├── [Classic Card]
  │   │   ├── Show question
  │   │   ├── "Show Answer" button
  │   │   ├── Tap → 3D flip animation → reveal answer
  │   │   ├── "Got it" / "Missed it" buttons
  │   │   ├── Record: cardId + rating (Got it=3, Missed it=1)
  │   │   └── Advance to next card
  │   │
  │   └── [MCQ Card]
  │       ├── Show question + 4 choice buttons
  │       ├── User taps choice
  │       ├── Correct → choice turns green, success haptic
  │       ├── Incorrect → choice turns red, correct answer highlighted green
  │       ├── Show explanation (if available)
  │       ├── Record: cardId + rating (correct=3, incorrect=1)
  │       ├── 1.5s delay, then advance
  │       └── Record review
  │
  └── All cards done → SessionSummaryView
      ├── Cards reviewed count
      ├── Accuracy percentage
      ├── XP earned
      ├── "Continue" or "Back to Units" buttons
      └── Send ratings to server (POST /reviews) — server runs FSRS
```

### FlashcardSessionViewModel
```swift
@Observable
class FlashcardSessionViewModel {
    var cards: [Card] = []
    var currentIndex: Int = 0
    var isAnswerRevealed: Bool = false
    var selectedChoice: Int? = nil
    var isCorrect: Bool? = nil
    var sessionResults: [PendingReview] = []
    
    var currentCard: Card? { 
        cards.indices.contains(currentIndex) ? cards[currentIndex] : nil 
    }
    var progress: Double { 
        cards.isEmpty ? 0 : Double(currentIndex) / Double(cards.count) 
    }
    var isSessionComplete: Bool { currentIndex >= cards.count }
    
    private let reviewService: ReviewService
    
    // Classic mode: reveal answer
    func revealAnswer() {
        isAnswerRevealed = true
    }
    
    // Classic mode: mark correct/incorrect
    func markAnswer(correct: Bool) {
        let rating: Int = correct ? 3 : 1  // Good=3, Again=1
        recordReview(rating: rating)
        advanceToNext()
    }
    
    // MCQ mode: select choice
    func selectChoice(index: Int) {
        selectedChoice = index
        let card = cards[currentIndex]
        isCorrect = card.choices?[index].isCorrect ?? false
        let rating: Int = (isCorrect == true) ? 3 : 1
        recordReview(rating: rating)
        
        // Auto-advance after delay
        Task {
            try? await Task.sleep(for: .seconds(1.5))
            advanceToNext()
        }
    }
    
    private func recordReview(rating: Int) {
        let card = cards[currentIndex]
        let review = PendingReview(
            cardId: card.id,
            rating: rating,
            reviewedAt: Date()
        )
        sessionResults.append(review)
    }
    
    private func advanceToNext() {
        currentIndex += 1
        isAnswerRevealed = false
        selectedChoice = nil
        isCorrect = nil
    }
    
    func syncResults() async {
        await reviewService.submitReviews(sessionResults)
    }
}
```

---

## Animations

### Card Flip (Classic Mode)
3D rotation around the Y-axis to reveal the answer.

```swift
struct FlashcardView: View {
    let card: Card
    @Binding var isFlipped: Bool
    
    var body: some View {
        ZStack {
            // Front face (question)
            CardFace(text: card.question, color: .blue)
                .opacity(isFlipped ? 0 : 1)
                .rotation3DEffect(
                    .degrees(isFlipped ? 180 : 0),
                    axis: (x: 0, y: 1, z: 0)
                )
            
            // Back face (answer)
            CardFace(text: card.answer ?? "", color: .green)
                .opacity(isFlipped ? 1 : 0)
                .rotation3DEffect(
                    .degrees(isFlipped ? 0 : -180),
                    axis: (x: 0, y: 1, z: 0)
                )
        }
        .animation(.easeInOut(duration: 0.3), value: isFlipped)
        .onTapGesture {
            isFlipped.toggle()
        }
    }
}
```

### MCQ Feedback
```swift
// Choice button changes color on selection
struct ChoiceButton: View {
    let choice: Choice
    let isSelected: Bool
    let showResult: Bool
    
    var backgroundColor: Color {
        guard showResult else { return .secondary.opacity(0.1) }
        if choice.isCorrect { return .green.opacity(0.3) }
        if isSelected && !choice.isCorrect { return .red.opacity(0.3) }
        return .secondary.opacity(0.1)
    }
    
    var body: some View {
        Text(choice.text)
            .frame(maxWidth: .infinity)
            .padding()
            .background(backgroundColor)
            .cornerRadius(12)
            .animation(.easeInOut(duration: 0.2), value: showResult)
    }
}
```

### Streak Heatmap
Custom rendering using SwiftUI `Canvas` for performance with 365 cells.

```swift
struct StreakHeatmapView: View {
    let activities: [DailyActivity]  // 365 days
    
    var body: some View {
        Canvas { context, size in
            let cellSize: CGFloat = 12
            let gap: CGFloat = 2
            let cols = 52  // weeks
            let rows = 7   // days
            
            for (index, activity) in activities.enumerated() {
                let col = index / 7
                let row = index % 7
                let x = CGFloat(col) * (cellSize + gap)
                let y = CGFloat(row) * (cellSize + gap)
                
                let intensity = min(Double(activity.cardsReviewed) / 30.0, 1.0)
                let color = Color.green.opacity(0.1 + intensity * 0.9)
                
                let rect = CGRect(x: x, y: y, width: cellSize, height: cellSize)
                context.fill(
                    RoundedRectangle(cornerRadius: 2).path(in: rect),
                    with: .color(activity.cardsReviewed == 0 ? .gray.opacity(0.15) : color)
                )
            }
        }
        .frame(height: 7 * 14) // 7 rows × (12 + 2) gap
    }
}
```

### Progress Bar
```swift
struct SessionProgressBar: View {
    let progress: Double  // 0.0 to 1.0
    
    var body: some View {
        GeometryReader { geo in
            ZStack(alignment: .leading) {
                RoundedRectangle(cornerRadius: 4)
                    .fill(.gray.opacity(0.2))
                
                RoundedRectangle(cornerRadius: 4)
                    .fill(.green)
                    .frame(width: geo.size.width * progress)
                    .animation(.easeInOut(duration: 0.3), value: progress)
            }
        }
        .frame(height: 8)
    }
}
```

---

## Offline Architecture

### Local Storage (SwiftData)

```swift
// SwiftData models for local caching

@Model
class CachedUnit {
    @Attribute(.unique) var unitId: String
    var version: Int
    var lastFetched: Date
    var cards: [CachedCard]
}

@Model
class CachedCard {
    @Attribute(.unique) var cardId: String
    var type: String        // "classic" or "mcq"
    var question: String
    var answer: String?
    var choicesJSON: String? // Serialized [Choice]
    var explanation: String?
    var displayOrder: Int
}

@Model
class PendingReview {
    var cardId: String
    var rating: Int
    var reviewedAt: Date
    var isSynced: Bool = false
}
```

### Cache Flow
```
User opens unit:
  ├── Check SwiftData: is unit cached? Is version current?
  │   ├── YES: Load cards from SwiftData → start session
  │   └── NO: Fetch from API → store in SwiftData → start session
  │
  During session:
  ├── Record results: cardId + rating + timestamp
  ├── Store as PendingReview in SwiftData (no FSRS state needed)
  │
  After session:
  ├── If online: POST /reviews → server runs FSRS → mark PendingReviews as synced → delete
  └── If offline: PendingReviews remain queued (server will run FSRS when synced)
```

### Sync Manager
```swift
@Observable
class SyncManager {
    var pendingCount: Int = 0
    var isSyncing: Bool = false
    
    func syncPendingReviews() async {
        let pending = fetchPendingReviews()  // from SwiftData
        guard !pending.isEmpty else { return }
        
        isSyncing = true
        defer { isSyncing = false }
        
        // Send raw ratings only — server computes FSRS
        let reviews = pending.map { $0.toReviewRequest() }
        
        do {
            let response = try await apiClient.submitReviews(reviews)
            markAsSynced(pending)
            pendingCount = 0
        } catch {
            // Will retry on next connectivity change or app foreground
        }
    }
}
```

**Trigger sync on:**
- App comes to foreground
- Network connectivity restored (via `NWPathMonitor`)
- After each study session completes (if online)

---

## Networking

### API Client
```swift
class APIClient {
    private let baseURL: URL
    private let session: URLSession
    private let tokenStore: TokenStore  // Keychain wrapper
    
    func request<T: Decodable>(_ endpoint: Endpoint) async throws -> T {
        var request = endpoint.urlRequest(baseURL: baseURL)
        
        // Attach access token
        if let token = tokenStore.accessToken {
            request.setValue("Bearer \(token)", forHTTPHeaderField: "Authorization")
        }
        
        let (data, response) = try await session.data(for: request)
        let httpResponse = response as! HTTPURLResponse
        
        // Handle 401 — attempt token refresh
        if httpResponse.statusCode == 401 {
            try await refreshToken()
            return try await self.request(endpoint)  // Retry with new token
        }
        
        guard (200...299).contains(httpResponse.statusCode) else {
            let error = try JSONDecoder().decode(APIError.self, from: data)
            throw error
        }
        
        return try JSONDecoder().decode(T.self, from: data)
    }
    
    private func refreshToken() async throws {
        guard let refreshToken = tokenStore.refreshToken else {
            throw AuthError.noRefreshToken
        }
        
        let response: TokenResponse = try await request(.refreshToken(refreshToken))
        tokenStore.saveTokens(access: response.accessToken, refresh: response.refreshToken)
    }
}
```

### Token Storage (Keychain)
```swift
class TokenStore {
    private let keychain = KeychainHelper()
    
    var accessToken: String? {
        get { keychain.read(key: "snapy_access_token") }
        set { keychain.save(key: "snapy_access_token", value: newValue) }
    }
    
    var refreshToken: String? {
        get { keychain.read(key: "snapy_refresh_token") }
        set { keychain.save(key: "snapy_refresh_token", value: newValue) }
    }
}
```

---

## Push Notifications (APNs)

### Registration
```swift
// In AppDelegate or App init
UNUserNotificationCenter.current().requestAuthorization(options: [.alert, .badge, .sound]) { granted, _ in
    if granted {
        DispatchQueue.main.async {
            UIApplication.shared.registerForRemoteNotifications()
        }
    }
}

// When token received
func application(_ application: UIApplication, didRegisterForRemoteNotificationsWithDeviceToken deviceToken: Data) {
    let token = deviceToken.map { String(format: "%02x", $0) }.joined()
    Task {
        try? await apiClient.registerDeviceToken(token: token, platform: "ios")
    }
}
```

### Notification Types
| Type | Trigger | Content |
|------|---------|---------|
| Daily Reminder | User's configured time, if no session today | "Time to study! You have X cards due." |
| Streak at Risk | Evening, if no session and streak > 3 | "Don't lose your X-day streak! Study now." |
| Plan Reminder | Morning, if study plan has tasks today | "Today's plan: X units in Y subjects." |

---

## FSRS Integration (Server-Side)

### Approach
FSRS runs exclusively on the Go backend. The iOS app does NOT implement FSRS. It only collects ratings and sends them to the server.

### App Responsibilities
1. Display cards and collect ratings:
   - Classic mode: "Got it" (rating 3) or "Missed it" (rating 1)
   - MCQ mode: Correct (rating 3) or Incorrect (rating 1)
2. Queue results as `PendingReview` with `{ cardId, rating, reviewedAt }`
3. Sync to `POST /reviews` when online — server runs FSRS
4. Fetch due cards from `GET /reviews/due` — server determines what's due

### Due Cards Badge
- App calls `GET /reviews/due` when online to get due card count
- Displayed as a badge on the home screen
- **Requires connectivity** — the phone cannot compute due dates locally

---

## Project Structure

```
Snapy/
├── SnapyApp.swift                    # App entry point, tab view setup
├── Views/
│   ├── Onboarding/
│   │   ├── SplashView.swift
│   │   ├── WelcomeView.swift
│   │   ├── PhoneInputView.swift
│   │   ├── OTPVerifyView.swift
│   │   ├── NameInputView.swift
│   │   ├── GradeSelectView.swift
│   │   └── StreamSelectView.swift
│   ├── Home/
│   │   ├── HomeView.swift
│   │   └── SubjectCardView.swift
│   ├── Study/
│   │   ├── SubjectDetailView.swift
│   │   ├── FlashcardSessionView.swift
│   │   ├── FlashcardView.swift
│   │   ├── ClassicCardView.swift
│   │   ├── MCQCardView.swift
│   │   ├── ChoiceButton.swift
│   │   └── SessionSummaryView.swift
│   ├── StudyPlan/
│   │   ├── StudyPlanListView.swift
│   │   ├── CreatePlanView.swift
│   │   └── PlanDetailView.swift
│   ├── Analytics/
│   │   ├── AnalyticsDashboardView.swift
│   │   ├── StreakHeatmapView.swift
│   │   └── WeakAreasView.swift
│   ├── Leaderboard/
│   │   └── LeaderboardView.swift
│   ├── Profile/
│   │   └── ProfileView.swift
│   └── Components/
│       ├── SessionProgressBar.swift
│       ├── StreakBadge.swift
│       └── LoadingView.swift
├── ViewModels/
│   ├── AuthViewModel.swift
│   ├── HomeViewModel.swift
│   ├── SubjectDetailViewModel.swift
│   ├── FlashcardSessionViewModel.swift
│   ├── StudyPlanViewModel.swift
│   ├── AnalyticsViewModel.swift
│   ├── LeaderboardViewModel.swift
│   └── ProfileViewModel.swift
├── Services/
│   ├── APIClient.swift
│   ├── TokenStore.swift
│   ├── ContentService.swift
│   ├── ReviewService.swift
│   ├── AnalyticsService.swift
│   ├── SyncManager.swift
│   └── PushNotificationService.swift
├── Models/
│   ├── User.swift
│   ├── Subject.swift
│   ├── Unit.swift
│   ├── Card.swift
│   ├── ReviewResult.swift
│   └── APIError.swift
├── Data/
│   ├── SwiftDataModels.swift       # CachedUnit, CachedCard, PendingReview
│   └── ModelContainer+Setup.swift
├── Extensions/
│   ├── Color+Snapy.swift           # Brand colors
│   └── Date+Formatting.swift
├── Resources/
│   ├── Assets.xcassets
│   └── Localizable.xcstrings       # English, Sinhala, Tamil
└── Info.plist
```

---

## Testing Strategy

### Unit Tests
| Area | What to Test |
|------|-------------|
| ViewModels | State transitions, data loading, error handling |
| Services | API request construction, response parsing |
| Sync Manager | Queue management, batch upload |

### UI Tests
| Flow | What to Test |
|------|-------------|
| Onboarding | Complete registration flow end-to-end |
| Flashcard Session | Classic mode full session, MCQ mode full session |
| Study Plan | Plan creation, task completion |
| Offline | Start session offline, sync when online |

### Tools
- **XCTest**: Unit and UI testing framework
- **Swift Testing** (new): Modern test syntax with `@Test` macro
- **Preview Testing**: SwiftUI previews for visual verification during development
