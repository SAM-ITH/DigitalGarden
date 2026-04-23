# Snapy - iOS App: Detailed Design

## Overview

The Snapy iOS app is built with **Swift** and **SwiftUI**, following the **MVVM** architecture pattern. It uses a **native Swift FSRS implementation** (`swift-fsrs` package or ~200 lines of native code) for the spaced repetition algorithm.

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
└──────────┬─────────────────┬─────────────────┘
           │                 │
┌──────────▼──────┐ ┌───────▼─────────────────┐
│    Services     │ │   FSRS Engine            │
│  (Network, DB,  │ │   (swift-fsrs, native    │
│   Auth, Push)   │ │    Swift implementation) │
└──────────┬──────┘ └─────────────────────────┘
           │
┌──────────▼──────────────────────────────────┐
│          Data Layer                          │
│  URLSession (API)  │  SwiftData (Local DB)   │
│  Keychain (Tokens) │                         │
└──────────────────────────────────────────────┘
```

### Key Architecture Principles
- **MVVM**: Views observe ViewModels, ViewModels coordinate services
- **@Observable macro** (Swift 5.9+): Simpler than ObservableObject, automatic change tracking
- **Dependency Injection**: Services injected via environment or init parameters
- **Async/Await**: All async operations use Swift Concurrency (no Combine for data flow, Combine only where needed for legacy APIs)

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
  ├── Shuffle or order by FSRS priority
  ├── Display progress bar (card X of N)
  │
  ├── For each card:
  │   ├── [Classic Card]
  │   │   ├── Show question
  │   │   ├── "Show Answer" button
  │   │   ├── Tap → 3D flip animation → reveal answer
  │   │   ├── "Got it" / "Missed it" buttons
  │   │   ├── Run FSRS: "Got it" → rating 3 (Good), "Missed it" → rating 1 (Again)
  │   │   └── Record review, advance to next card
  │   │
  │   └── [MCQ Card]
  │       ├── Show question + 4 choice buttons
  │       ├── User taps choice
  │       ├── Correct → choice turns green, success haptic
  │       ├── Incorrect → choice turns red, correct answer highlighted green
  │       ├── Show explanation (if available)
  │       ├── Run FSRS: correct → rating 3 (Good), incorrect → rating 1 (Again)
  │       ├── 1.5s delay, then advance
  │       └── Record review
  │
  └── All cards done → SessionSummaryView
      ├── Cards reviewed count
      ├── Accuracy percentage
      ├── XP earned
      ├── "Continue" or "Back to Units" buttons
      └── Sync reviews to server (POST /reviews)
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
    var sessionResults: [ReviewResult] = []
    
    var currentCard: Card? { 
        cards.indices.contains(currentIndex) ? cards[currentIndex] : nil 
    }
    var progress: Double { 
        cards.isEmpty ? 0 : Double(currentIndex) / Double(cards.count) 
    }
    var isSessionComplete: Bool { currentIndex >= cards.count }
    
    private let fsrs: FSRS  // Native Swift FSRS
    private let reviewService: ReviewService
    
    // Classic mode: reveal answer
    func revealAnswer() {
        isAnswerRevealed = true
    }
    
    // Classic mode: mark correct/incorrect
    func markAnswer(correct: Bool) {
        let rating: Rating = correct ? .good : .again
        recordReview(rating: rating)
        advanceToNext()
    }
    
    // MCQ mode: select choice
    func selectChoice(index: Int) {
        selectedChoice = index
        let card = cards[currentIndex]
        isCorrect = card.choices?[index].isCorrect ?? false
        let rating: Rating = isCorrect == true ? .good : .again
        recordReview(rating: rating)
        
        // Auto-advance after delay
        Task {
            try? await Task.sleep(for: .seconds(1.5))
            advanceToNext()
        }
    }
    
    private func recordReview(rating: Rating) {
        let card = cards[currentIndex]
        let result = fsrs.review(cardId: card.id, rating: rating)
        sessionResults.append(result)
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
    var srsStateJSON: String  // Serialized SRS state from FSRS
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
  ├── FSRS runs locally → instant scheduling
  ├── Review results stored as PendingReview in SwiftData
  │
  After session:
  ├── If online: POST /reviews → mark PendingReviews as synced → delete
  └── If offline: PendingReviews remain queued
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

## FSRS Integration (Native Swift)

### Approach
The FSRS algorithm is implemented natively in Swift — either using the `swift-fsrs` open-source package or a ~200-line native implementation. No cross-platform bridge needed.

### Using FSRS

```swift
import SwiftFSRS  // or local FSRS module

// Create FSRS instance with default parameters
let fsrs = FSRS(parameters: .default)

// Review a card
let result = fsrs.review(
    state: .new,
    stability: 0,
    difficulty: 5.0,
    elapsedDays: 0,
    rating: .good
)
// result.nextState — updated card state
// result.stability — new stability value
// result.scheduledDays — days until next review
// result.dueAt — exact due date
```

### Native Types
```swift
enum Rating: Int {
    case again = 1, hard = 2, good = 3, easy = 4
}

enum CardState {
    case new, learning, review, relearning
}

struct ReviewResult {
    let cardId: String
    let rating: Rating
    let nextState: CardState
    let stability: Double
    let difficulty: Double
    let scheduledDays: Int
    let dueAt: Date
}
```

### Consistency Verification
The same test vectors are used across iOS (Swift), Android (Kotlin), and backend (Go) to ensure all three implementations produce identical scheduling results.

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
├── FSRS/
│   ├── FSRS.swift                    # FSRS algorithm implementation
│   ├── FSRSParameters.swift          # Algorithm parameters (w0-w18)
│   ├── CardState.swift               # State enum
│   ├── Rating.swift                  # Rating enum
│   └── ReviewResult.swift            # Review output model
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
| Sync Manager | Queue management, conflict resolution |
| FSRS Algorithm | FSRS scheduling produces expected intervals (shared test vectors) |

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
