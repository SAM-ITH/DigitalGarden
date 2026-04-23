# Snapy - Android App: Detailed Design

## Overview

The Snapy Android app is built with **Kotlin** and **Jetpack Compose**, following the **MVVM** architecture pattern with **Hilt** for dependency injection. It uses a **native Kotlin FSRS implementation** (`fsrs-kt` package or ~200 lines of native code) for the spaced repetition algorithm.

---

## Architecture

```
┌──────────────────────────────────────────────┐
│              UI Layer (Jetpack Compose)        │
│  Screens, Components, Animations              │
└──────────────────┬───────────────────────────┘
                   │ Collects StateFlow
┌──────────────────▼───────────────────────────┐
│              ViewModels (Hilt-injected)        │
│  Business logic, UI state, coroutines         │
└──────────┬─────────────────┬─────────────────┘
           │                 │
┌──────────▼──────┐ ┌───────▼─────────────────┐
│   Repositories  │ │   FSRS Engine            │
│  (Data access   │ │   (fsrs-kt, native       │
│   abstraction)  │ │    Kotlin implementation)│
└──────────┬──────┘ └─────────────────────────┘
           │
┌──────────▼──────────────────────────────────┐
│          Data Sources                        │
│  Ktor Client (API) │  Room (Local DB)        │
│  EncryptedSharedPrefs (Tokens)               │
└──────────────────────────────────────────────┘
```

### Key Architecture Principles
- **MVVM with Repository pattern**: ViewModels access data through repositories, not directly
- **Unidirectional data flow**: UI state exposed as `StateFlow`, events flow up via callbacks
- **Hilt DI**: All dependencies injected at compile time
- **Coroutines + Flow**: All async operations use structured concurrency

---

## Screen Inventory

### Onboarding Flow
| Screen | Composable | Description |
|--------|-----------|-------------|
| Splash | `SplashScreen` | App logo, auto-login check |
| Welcome | `WelcomeScreen` | App introduction, "Get Started" button |
| Phone Input | `PhoneInputScreen` | Phone number entry with +94 prefix |
| OTP Verify | `OTPVerifyScreen` | 6-digit OTP input, auto-advance, resend timer |
| Name Input | `NameInputScreen` | User's name entry |
| Grade Select | `GradeSelectScreen` | Grade picker (10, 11, 12, 13) |
| Stream Select | `StreamSelectScreen` | A/L stream picker (shown only for Grade 12-13) |

### Main App (Bottom Navigation)
| Tab | Screen | Composable | Description |
|-----|--------|-----------|-------------|
| Study | Home | `HomeScreen` | Subject grid, streak indicator, quick review button |
| Study | Subject Detail | `SubjectDetailScreen` | Term tabs, unit list with progress |
| Study | Flashcard Session | `FlashcardSessionScreen` | Active study session |
| Study | Session Summary | `SessionSummaryScreen` | Post-session results |
| Plan | Plan List | `StudyPlanListScreen` | User's plans, create new |
| Plan | Create Plan | `CreatePlanScreen` | Goal, subjects, time, deadline |
| Plan | Plan Detail | `PlanDetailScreen` | Calendar, daily tasks |
| Stats | Dashboard | `AnalyticsDashboardScreen` | Stats, heatmap, charts |
| Ranks | Rankings | `LeaderboardScreen` | Grade and subject tabs |
| Profile | Settings | `ProfileScreen` | User info, preferences, logout |

### Navigation Structure
```kotlin
@Composable
fun SnapyNavHost(navController: NavHostController) {
    NavHost(navController, startDestination = "splash") {
        // Onboarding
        composable("splash") { SplashScreen(navController) }
        composable("welcome") { WelcomeScreen(navController) }
        composable("phone_input") { PhoneInputScreen(navController) }
        composable("otp_verify/{phone}") { OTPVerifyScreen(navController) }
        composable("name_input") { NameInputScreen(navController) }
        composable("grade_select") { GradeSelectScreen(navController) }
        composable("stream_select") { StreamSelectScreen(navController) }
        
        // Main app with bottom nav
        composable("main") {
            MainScreen()  // Contains Scaffold with BottomNavigation + nested NavHost
        }
        
        // Nested: Study flow
        composable("subject/{subjectId}") { SubjectDetailScreen(navController) }
        composable("session/{unitId}") { FlashcardSessionScreen(navController) }
        composable("summary") { SessionSummaryScreen(navController) }
        
        // Nested: Study Plan flow
        composable("create_plan") { CreatePlanScreen(navController) }
        composable("plan/{planId}") { PlanDetailScreen(navController) }
    }
}
```

---

## State Management

### UI State Pattern
Each screen has a dedicated UI state data class and ViewModel.

```kotlin
// UI State
data class HomeUiState(
    val subjects: List<Subject> = emptyList(),
    val dueCardsCount: Int = 0,
    val currentStreak: Int = 0,
    val isLoading: Boolean = false,
    val error: String? = null
)

// ViewModel
@HiltViewModel
class HomeViewModel @Inject constructor(
    private val contentRepository: ContentRepository,
    private val reviewRepository: ReviewRepository,
    private val analyticsRepository: AnalyticsRepository
) : ViewModel() {
    
    private val _uiState = MutableStateFlow(HomeUiState())
    val uiState: StateFlow<HomeUiState> = _uiState.asStateFlow()
    
    init {
        loadHome()
    }
    
    private fun loadHome() {
        viewModelScope.launch {
            _uiState.update { it.copy(isLoading = true) }
            
            try {
                // Parallel fetch with coroutines
                coroutineScope {
                    val subjects = async { contentRepository.getSubjects() }
                    val dueCount = async { reviewRepository.getDueCardsCount() }
                    val streak = async { analyticsRepository.getCurrentStreak() }
                    
                    _uiState.update {
                        it.copy(
                            subjects = subjects.await(),
                            dueCardsCount = dueCount.await(),
                            currentStreak = streak.await(),
                            isLoading = false
                        )
                    }
                }
            } catch (e: Exception) {
                _uiState.update { it.copy(error = e.message, isLoading = false) }
            }
        }
    }
}

// Screen
@Composable
fun HomeScreen(
    viewModel: HomeViewModel = hiltViewModel(),
    onSubjectClick: (Subject) -> Unit,
    onQuickReviewClick: () -> Unit
) {
    val uiState by viewModel.uiState.collectAsStateWithLifecycle()
    
    // Render based on uiState...
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
Identical to iOS (see `01-app-overview.md` for user flow). The session ViewModel manages the same state machine.

### FlashcardSessionViewModel
```kotlin
@HiltViewModel
class FlashcardSessionViewModel @Inject constructor(
    private val reviewRepository: ReviewRepository,
    private val fsrs: FSRS,  // Native Kotlin FSRS
    savedStateHandle: SavedStateHandle
) : ViewModel() {
    
    private val unitId: String = savedStateHandle["unitId"]!!
    
    data class SessionUiState(
        val cards: List<Card> = emptyList(),
        val currentIndex: Int = 0,
        val isAnswerRevealed: Boolean = false,
        val selectedChoice: Int? = null,
        val isCorrect: Boolean? = null,
        val sessionResults: List<ReviewResult> = emptyList(),
        val isSessionComplete: Boolean = false
    ) {
        val currentCard: Card? get() = cards.getOrNull(currentIndex)
        val progress: Float get() = if (cards.isEmpty()) 0f else currentIndex.toFloat() / cards.size
    }
    
    private val _uiState = MutableStateFlow(SessionUiState())
    val uiState: StateFlow<SessionUiState> = _uiState.asStateFlow()
    
    init {
        loadCards()
    }
    
    private fun loadCards() {
        viewModelScope.launch {
            val cards = reviewRepository.getCardsForUnit(unitId)
            _uiState.update { it.copy(cards = cards) }
        }
    }
    
    fun revealAnswer() {
        _uiState.update { it.copy(isAnswerRevealed = true) }
    }
    
    fun markAnswer(correct: Boolean) {
        val rating = if (correct) Rating.GOOD else Rating.AGAIN
        recordReview(rating)
        advanceToNext()
    }
    
    fun selectChoice(index: Int) {
        val card = _uiState.value.currentCard ?: return
        val isCorrect = card.choices?.get(index)?.isCorrect ?: false
        val rating = if (isCorrect) Rating.GOOD else Rating.AGAIN
        
        _uiState.update { it.copy(selectedChoice = index, isCorrect = isCorrect) }
        recordReview(rating)
        
        viewModelScope.launch {
            delay(1500)
            advanceToNext()
        }
    }
    
    private fun recordReview(rating: Rating) {
        val card = _uiState.value.currentCard ?: return
        val result = fsrs.review(cardId = card.id, rating = rating)
        _uiState.update { it.copy(sessionResults = it.sessionResults + result) }
    }
    
    private fun advanceToNext() {
        _uiState.update {
            val nextIndex = it.currentIndex + 1
            it.copy(
                currentIndex = nextIndex,
                isAnswerRevealed = false,
                selectedChoice = null,
                isCorrect = null,
                isSessionComplete = nextIndex >= it.cards.size
            )
        }
    }
    
    fun syncResults() {
        viewModelScope.launch {
            reviewRepository.submitReviews(_uiState.value.sessionResults)
        }
    }
}
```

---

## Animations

### Card Flip (Classic Mode)
3D rotation using Compose animation APIs.

```kotlin
@Composable
fun FlashcardView(
    card: Card,
    isFlipped: Boolean,
    onFlip: () -> Unit
) {
    val rotation by animateFloatAsState(
        targetValue = if (isFlipped) 180f else 0f,
        animationSpec = tween(durationMillis = 300, easing = EaseInOut),
        label = "cardFlip"
    )
    
    Box(
        modifier = Modifier
            .fillMaxWidth()
            .height(300.dp)
            .graphicsLayer {
                rotationY = rotation
                cameraDistance = 12f * density
            }
            .clickable { onFlip() }
    ) {
        if (rotation <= 90f) {
            // Front face (question)
            CardFace(
                text = card.question,
                color = MaterialTheme.colorScheme.primaryContainer
            )
        } else {
            // Back face (answer) — counter-rotate so text reads correctly
            CardFace(
                text = card.answer ?: "",
                color = MaterialTheme.colorScheme.tertiaryContainer,
                modifier = Modifier.graphicsLayer { rotationY = 180f }
            )
        }
    }
}
```

### MCQ Feedback
```kotlin
@Composable
fun ChoiceButton(
    choice: Choice,
    isSelected: Boolean,
    showResult: Boolean,
    onClick: () -> Unit
) {
    val backgroundColor by animateColorAsState(
        targetValue = when {
            showResult && choice.isCorrect -> Color(0xFF4CAF50).copy(alpha = 0.3f)
            showResult && isSelected && !choice.isCorrect -> Color(0xFFF44336).copy(alpha = 0.3f)
            else -> MaterialTheme.colorScheme.surfaceVariant
        },
        animationSpec = tween(200),
        label = "choiceColor"
    )
    
    OutlinedButton(
        onClick = onClick,
        enabled = !showResult,
        colors = ButtonDefaults.outlinedButtonColors(containerColor = backgroundColor),
        modifier = Modifier.fillMaxWidth()
    ) {
        Text(choice.text)
    }
}
```

### Streak Heatmap
Custom rendering using Compose `Canvas`.

```kotlin
@Composable
fun StreakHeatmap(
    activities: List<DailyActivity>,  // 365 days
    modifier: Modifier = Modifier
) {
    Canvas(
        modifier = modifier
            .fillMaxWidth()
            .height(with(LocalDensity.current) { (7 * 14).dp })
    ) {
        val cellSize = 12.dp.toPx()
        val gap = 2.dp.toPx()
        
        activities.forEachIndexed { index, activity ->
            val col = index / 7
            val row = index % 7
            val x = col * (cellSize + gap)
            val y = row * (cellSize + gap)
            
            val intensity = (activity.cardsReviewed / 30f).coerceIn(0f, 1f)
            val color = if (activity.cardsReviewed == 0) {
                Color.Gray.copy(alpha = 0.15f)
            } else {
                Color(0xFF4CAF50).copy(alpha = 0.1f + intensity * 0.9f)
            }
            
            drawRoundRect(
                color = color,
                topLeft = Offset(x, y),
                size = Size(cellSize, cellSize),
                cornerRadius = CornerRadius(2.dp.toPx())
            )
        }
    }
}
```

### Progress Bar
```kotlin
@Composable
fun SessionProgressBar(progress: Float) {
    LinearProgressIndicator(
        progress = { animateFloatAsState(progress, tween(300)).value },
        modifier = Modifier
            .fillMaxWidth()
            .height(8.dp)
            .clip(RoundedCornerShape(4.dp)),
        color = MaterialTheme.colorScheme.primary,
        trackColor = MaterialTheme.colorScheme.surfaceVariant
    )
}
```

---

## Offline Architecture

### Local Storage (Room)

```kotlin
// Room entities for local caching

@Entity(tableName = "cached_units")
data class CachedUnitEntity(
    @PrimaryKey val unitId: String,
    val version: Int,
    val lastFetched: Long  // epoch millis
)

@Entity(tableName = "cached_cards")
data class CachedCardEntity(
    @PrimaryKey val cardId: String,
    val unitId: String,
    val type: String,          // "classic" or "mcq"
    val question: String,
    val answer: String?,
    val choicesJson: String?,   // Serialized JSON
    val explanation: String?,
    val displayOrder: Int
)

@Entity(tableName = "pending_reviews")
data class PendingReviewEntity(
    @PrimaryKey(autoGenerate = true) val id: Long = 0,
    val cardId: String,
    val rating: Int,
    val reviewedAt: Long,       // epoch millis
    val srsStateJson: String,   // Serialized SRS state
    val isSynced: Boolean = false
)
```

### Room DAOs
```kotlin
@Dao
interface CardCacheDao {
    @Query("SELECT * FROM cached_cards WHERE unitId = :unitId ORDER BY displayOrder")
    fun getCardsForUnit(unitId: String): Flow<List<CachedCardEntity>>
    
    @Insert(onConflict = OnConflictStrategy.REPLACE)
    suspend fun insertCards(cards: List<CachedCardEntity>)
    
    @Query("SELECT version FROM cached_units WHERE unitId = :unitId")
    suspend fun getUnitVersion(unitId: String): Int?
}

@Dao
interface PendingReviewDao {
    @Insert
    suspend fun insert(review: PendingReviewEntity)
    
    @Query("SELECT * FROM pending_reviews WHERE isSynced = 0")
    suspend fun getPendingReviews(): List<PendingReviewEntity>
    
    @Query("UPDATE pending_reviews SET isSynced = 1 WHERE id IN (:ids)")
    suspend fun markAsSynced(ids: List<Long>)
    
    @Query("DELETE FROM pending_reviews WHERE isSynced = 1")
    suspend fun deleteSynced()
}
```

### Sync with WorkManager
```kotlin
@HiltWorker
class ReviewSyncWorker @AssistedInject constructor(
    @Assisted context: Context,
    @Assisted params: WorkerParameters,
    private val pendingReviewDao: PendingReviewDao,
    private val apiClient: APIClient
) : CoroutineWorker(context, params) {
    
    override suspend fun doWork(): Result {
        val pending = pendingReviewDao.getPendingReviews()
        if (pending.isEmpty()) return Result.success()
        
        return try {
            val reviews = pending.map { it.toReviewRequest() }
            apiClient.submitReviews(reviews)
            pendingReviewDao.markAsSynced(pending.map { it.id })
            pendingReviewDao.deleteSynced()
            Result.success()
        } catch (e: Exception) {
            Result.retry()  // WorkManager handles exponential backoff
        }
    }
}

// Schedule sync
fun scheduleSyncWork(context: Context) {
    val constraints = Constraints.Builder()
        .setRequiredNetworkType(NetworkType.CONNECTED)
        .build()
    
    val syncRequest = OneTimeWorkRequestBuilder<ReviewSyncWorker>()
        .setConstraints(constraints)
        .setBackoffCriteria(
            BackoffPolicy.EXPONENTIAL,
            WorkRequest.MIN_BACKOFF_MILLIS,
            TimeUnit.MILLISECONDS
        )
        .build()
    
    WorkManager.getInstance(context)
        .enqueueUniqueWork("review_sync", ExistingWorkPolicy.REPLACE, syncRequest)
}
```

---

## Networking

### Ktor Client Setup
```kotlin
@Module
@InstallIn(SingletonComponent::class)
object NetworkModule {
    
    @Provides
    @Singleton
    fun provideHttpClient(tokenStore: TokenStore): HttpClient {
        return HttpClient(OkHttp) {
            install(ContentNegotiation) {
                json(Json {
                    ignoreUnknownKeys = true
                    isLenient = true
                })
            }
            
            install(Auth) {
                bearer {
                    loadTokens {
                        BearerTokens(
                            accessToken = tokenStore.accessToken ?: "",
                            refreshToken = tokenStore.refreshToken ?: ""
                        )
                    }
                    refreshTokens {
                        val response = client.post("${BASE_URL}/auth/token/refresh") {
                            setBody(RefreshRequest(tokenStore.refreshToken ?: ""))
                            contentType(ContentType.Application.Json)
                        }
                        val tokens = response.body<TokenResponse>()
                        tokenStore.saveTokens(tokens.accessToken, tokens.refreshToken)
                        BearerTokens(tokens.accessToken, tokens.refreshToken)
                    }
                }
            }
            
            install(Logging) {
                level = LogLevel.BODY
            }
            
            defaultRequest {
                url(BASE_URL)
                contentType(ContentType.Application.Json)
            }
        }
    }
}
```

### Token Storage
```kotlin
class TokenStore @Inject constructor(
    @ApplicationContext context: Context
) {
    private val prefs = EncryptedSharedPreferences.create(
        "snapy_secure_prefs",
        MasterKeys.getOrCreate(MasterKeys.AES256_GCM_SPEC),
        context,
        EncryptedSharedPreferences.PrefKeyEncryptionScheme.AES256_SIV,
        EncryptedSharedPreferences.PrefValueEncryptionScheme.AES256_GCM
    )
    
    var accessToken: String?
        get() = prefs.getString("access_token", null)
        set(value) = prefs.edit().putString("access_token", value).apply()
    
    var refreshToken: String?
        get() = prefs.getString("refresh_token", null)
        set(value) = prefs.edit().putString("refresh_token", value).apply()
    
    fun saveTokens(access: String, refresh: String) {
        prefs.edit()
            .putString("access_token", access)
            .putString("refresh_token", refresh)
            .apply()
    }
    
    fun clear() {
        prefs.edit().clear().apply()
    }
}
```

---

## Push Notifications (FCM)

### Service
```kotlin
class SnapyFirebaseMessagingService : FirebaseMessagingService() {
    
    override fun onNewToken(token: String) {
        // Send token to backend
        CoroutineScope(Dispatchers.IO).launch {
            try {
                apiClient.registerDeviceToken(token = token, platform = "android")
            } catch (e: Exception) {
                // Store locally, retry later
            }
        }
    }
    
    override fun onMessageReceived(message: RemoteMessage) {
        message.notification?.let {
            showNotification(it.title ?: "Snapy", it.body ?: "")
        }
    }
    
    private fun showNotification(title: String, body: String) {
        val intent = Intent(this, MainActivity::class.java).apply {
            flags = Intent.FLAG_ACTIVITY_NEW_TASK or Intent.FLAG_ACTIVITY_CLEAR_TASK
        }
        val pendingIntent = PendingIntent.getActivity(
            this, 0, intent, PendingIntent.FLAG_IMMUTABLE
        )
        
        val notification = NotificationCompat.Builder(this, "snapy_reminders")
            .setSmallIcon(R.drawable.ic_notification)
            .setContentTitle(title)
            .setContentText(body)
            .setAutoCancel(true)
            .setContentIntent(pendingIntent)
            .build()
        
        NotificationManagerCompat.from(this).notify(0, notification)
    }
}
```

---

## FSRS Integration (Native Kotlin)

### Approach
The FSRS algorithm is implemented natively in Kotlin — either using the `fsrs-kt` open-source package or a ~200-line native implementation. No cross-platform bridge needed.

### Gradle Setup
```kotlin
// app/build.gradle.kts
dependencies {
    implementation("com.snapy:fsrs-kt:1.0.0")  // or local module
}
```

### Using FSRS from Kotlin
```kotlin
import com.snapy.fsrs.FSRS
import com.snapy.fsrs.FSRSParameters
import com.snapy.fsrs.Rating
import com.snapy.fsrs.CardState

val fsrs = FSRS(parameters = FSRSParameters.default)

val result = fsrs.review(
    state = CardState.New,
    stability = 0.0,
    difficulty = 5.0,
    elapsedDays = 0,
    rating = Rating.Good
)
// result.nextState
// result.stability
// result.scheduledDays
// result.dueAt
```

### Consistency Verification
The same test vectors are used across iOS (Swift), Android (Kotlin), and backend (Go) to ensure all three implementations produce identical scheduling results.

---

## Dependency Injection (Hilt)

```kotlin
@Module
@InstallIn(SingletonComponent::class)
object AppModule {
    
    @Provides
    @Singleton
    fun provideDatabase(@ApplicationContext context: Context): SnapyDatabase {
        return Room.databaseBuilder(context, SnapyDatabase::class.java, "snapy_db")
            .build()
    }
    
    @Provides
    fun provideCardCacheDao(db: SnapyDatabase): CardCacheDao = db.cardCacheDao()
    
    @Provides
    fun providePendingReviewDao(db: SnapyDatabase): PendingReviewDao = db.pendingReviewDao()
    
    @Provides
    @Singleton
    fun provideFSRS(): FSRS = FSRS(FSRSParameters.default)
    
    @Provides
    @Singleton
    fun provideTokenStore(@ApplicationContext context: Context): TokenStore = TokenStore(context)
}

@Module
@InstallIn(SingletonComponent::class)
abstract class RepositoryModule {
    
    @Binds
    abstract fun bindContentRepository(impl: ContentRepositoryImpl): ContentRepository
    
    @Binds
    abstract fun bindReviewRepository(impl: ReviewRepositoryImpl): ReviewRepository
    
    @Binds
    abstract fun bindAnalyticsRepository(impl: AnalyticsRepositoryImpl): AnalyticsRepository
}
```

---

## Project Structure

```
app/
├── src/main/
│   ├── java/com/snapy/app/
│   │   ├── SnapyApplication.kt           # Hilt application class
│   │   ├── MainActivity.kt               # Single activity, Compose host
│   │   ├── navigation/
│   │   │   ├── SnapyNavHost.kt            # Navigation graph
│   │   │   └── Screen.kt                 # Route definitions
│   │   ├── ui/
│   │   │   ├── onboarding/
│   │   │   │   ├── SplashScreen.kt
│   │   │   │   ├── WelcomeScreen.kt
│   │   │   │   ├── PhoneInputScreen.kt
│   │   │   │   ├── OTPVerifyScreen.kt
│   │   │   │   ├── NameInputScreen.kt
│   │   │   │   ├── GradeSelectScreen.kt
│   │   │   │   └── StreamSelectScreen.kt
│   │   │   ├── home/
│   │   │   │   ├── HomeScreen.kt
│   │   │   │   └── SubjectCard.kt
│   │   │   ├── study/
│   │   │   │   ├── SubjectDetailScreen.kt
│   │   │   │   ├── FlashcardSessionScreen.kt
│   │   │   │   ├── FlashcardView.kt
│   │   │   │   ├── ClassicCardView.kt
│   │   │   │   ├── MCQCardView.kt
│   │   │   │   ├── ChoiceButton.kt
│   │   │   │   └── SessionSummaryScreen.kt
│   │   │   ├── plan/
│   │   │   │   ├── StudyPlanListScreen.kt
│   │   │   │   ├── CreatePlanScreen.kt
│   │   │   │   └── PlanDetailScreen.kt
│   │   │   ├── analytics/
│   │   │   │   ├── AnalyticsDashboardScreen.kt
│   │   │   │   ├── StreakHeatmap.kt
│   │   │   │   └── WeakAreasSection.kt
│   │   │   ├── leaderboard/
│   │   │   │   └── LeaderboardScreen.kt
│   │   │   ├── profile/
│   │   │   │   └── ProfileScreen.kt
│   │   │   ├── components/
│   │   │   │   ├── SessionProgressBar.kt
│   │   │   │   ├── StreakBadge.kt
│   │   │   │   └── LoadingIndicator.kt
│   │   │   └── theme/
│   │   │       ├── Theme.kt
│   │   │       ├── Color.kt
│   │   │       └── Type.kt
│   │   ├── viewmodel/
│   │   │   ├── AuthViewModel.kt
│   │   │   ├── HomeViewModel.kt
│   │   │   ├── SubjectDetailViewModel.kt
│   │   │   ├── FlashcardSessionViewModel.kt
│   │   │   ├── StudyPlanViewModel.kt
│   │   │   ├── AnalyticsViewModel.kt
│   │   │   ├── LeaderboardViewModel.kt
│   │   │   └── ProfileViewModel.kt
│   │   ├── data/
│   │   │   ├── repository/
│   │   │   │   ├── ContentRepository.kt
│   │   │   │   ├── ReviewRepository.kt
│   │   │   │   └── AnalyticsRepository.kt
│   │   │   ├── remote/
│   │   │   │   ├── APIClient.kt
│   │   │   │   ├── dto/                  # API request/response models
│   │   │   │   └── TokenStore.kt
│   │   │   ├── local/
│   │   │   │   ├── SnapyDatabase.kt
│   │   │   │   ├── dao/
│   │   │   │   │   ├── CardCacheDao.kt
│   │   │   │   │   └── PendingReviewDao.kt
│   │   │   │   └── entity/               # Room entities
│   │   │   └── sync/
│   │   │       └── ReviewSyncWorker.kt
│   │   ├── fsrs/                         # Native Kotlin FSRS
│   │   │   ├── FSRS.kt
│   │   │   ├── FSRSParameters.kt
│   │   │   ├── CardState.kt
│   │   │   ├── Rating.kt
│   │   │   └── ReviewResult.kt
│   │   ├── di/
│   │   │   ├── AppModule.kt
│   │   │   ├── NetworkModule.kt
│   │   │   └── RepositoryModule.kt
│   │   ├── model/                        # Domain models
│   │   │   ├── User.kt
│   │   │   ├── Subject.kt
│   │   │   ├── Unit.kt
│   │   │   ├── Card.kt
│   │   │   └── DailyActivity.kt
│   │   └── service/
│   │       └── SnapyFirebaseMessagingService.kt
│   ├── res/
│   │   ├── values/
│   │   │   ├── strings.xml              # English strings
│   │   │   ├── colors.xml
│   │   │   └── themes.xml
│   │   ├── values-si/
│   │   │   └── strings.xml              # Sinhala strings
│   │   ├── values-ta/
│   │   │   └── strings.xml              # Tamil strings
│   │   ├── drawable/
│   │   └── mipmap/
│   └── AndroidManifest.xml
├── build.gradle.kts
└── proguard-rules.pro
```

---

## Testing Strategy

### Unit Tests
| Area | What to Test | Framework |
|------|-------------|-----------|
| ViewModels | State transitions, data loading, error handling | JUnit + Turbine (Flow testing) |
| Repositories | Data mapping, cache logic | JUnit + MockK |
| Room DAOs | Query correctness | Room testing (in-memory DB) |
| FSRS Algorithm | FSRS scheduling produces expected intervals (shared test vectors) | JUnit |

### UI Tests
| Flow | What to Test | Framework |
|------|-------------|-----------|
| Onboarding | Complete registration flow | Compose UI Test |
| Flashcard Session | Classic and MCQ modes | Compose UI Test |
| Navigation | Tab switching, deep navigation | Compose UI Test |
| Offline | Session without network | Compose UI Test + mock |

### Tools
| Tool | Purpose |
|------|---------|
| **JUnit 5** | Unit test framework |
| **MockK** | Kotlin mocking library |
| **Turbine** | Testing Kotlin Flows |
| **Compose UI Test** | UI testing for Compose |
| **Hilt Testing** | DI in tests |
| **Robolectric** | Run Android tests on JVM (faster CI) |

---

## Build Configuration

```kotlin
// app/build.gradle.kts
android {
    namespace = "com.snapy.app"
    compileSdk = 35
    
    defaultConfig {
        applicationId = "com.snapy.app"
        minSdk = 26        // Android 8.0+ (covers 95%+ of devices)
        targetSdk = 35
        versionCode = 1
        versionName = "1.0.0"
    }
    
    buildTypes {
        release {
            isMinifyEnabled = true
            isShrinkResources = true
            proguardFiles(
                getDefaultProguardFile("proguard-android-optimize.txt"),
                "proguard-rules.pro"
            )
        }
    }
    
    buildFeatures {
        compose = true
    }
}
```

### Min SDK 26 Rationale
- Android 8.0 (Oreo) and above
- Covers 95%+ of active Android devices globally
- Enables: notification channels, autofill, adaptive icons, background execution limits
- Avoids: complex backward compatibility workarounds for older APIs
