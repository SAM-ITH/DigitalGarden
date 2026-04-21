# BlinkBreak vs LookAway - Missing Features Analysis

> Comparison based on LookAway v0.9.0 through v2.0.1 release notes (all 9 pages of changelogs) against the current BlinkBreak codebase.

---

## 1. Statistics & Analytics

### 1.1 Stats Dashboard
LookAway provides a full statistics dashboard showing total screen time, focus time breakdown, app usage during screen sessions, time spent on different activities (games, meetings, etc.), and detailed break statistics. BlinkBreak has no statistics or tracking dashboard of any kind. The only telemetry currently sent is anonymous session-start events to TelemetryDeck.

### 1.2 Screen Score
LookAway calculates a daily "Screen Score" based on your screen time habits and break discipline. It includes a calendar heat-map view to track your score over the month, and an optional menu bar indicator showing the score. BlinkBreak has no scoring or gamification system to motivate healthy habits.

### 1.3 Time Since Last Break Indicators
LookAway displays "minutes without a break" in the menu bar, shows how long you've been working in break reminders, and displays cursor nudges when you've been overworking. BlinkBreak shows a session timer but does not track or display continuous work duration between breaks.

---

## 2. Break Management

### 2.1 Long Breaks
LookAway supports configurable long breaks (default: 3 minutes after every 3 short breaks). Users can customize the long break duration, frequency, and even trigger long breaks after every short break. BlinkBreak only has a single repeating short break cycle with no concept of long breaks.

### 2.2 Snooze / Postpone System
LookAway offers +1 min, +5 min, and +15 min snooze options directly in the break screen and notifications (replacing the old "Skip Break" button). Users can also configure keyboard shortcuts for snoozing. BlinkBreak only has a "Skip Break" (stop) button that immediately ends the break.

### 2.3 Postpone Limits
LookAway lets users cap how many times breaks can be postponed/snoozed per day, preventing mindless snoozing. BlinkBreak has no limit on how many times a user can skip breaks.

### 2.4 Skip Break Warning
LookAway displays a small warning when users skip a break after skipping multiple breaks in a row, encouraging healthier behavior. BlinkBreak has no such warning system.

### 2.5 Break Countdown Before Activation
LookAway shows a 5-second countdown near the cursor just before the break overlay activates, so users are never caught off-guard. The countdown duration is configurable (5s or 10s). BlinkBreak shows no pre-break warning and the full-screen overlay appears abruptly.

### 2.6 End Break Option (Long Breaks)
LookAway replaces "Skip Break" with "End Break" on long breaks when sufficient rest time has been detected, providing a more encouraging framing. BlinkBreak has no concept of smart break-end detection.

### 2.7 Break Reminder Notification Duration
LookAway allows configuring how long the break reminder notification remains visible before the full break overlay appears. BlinkBreak has no pre-break notification phase; it goes straight to the full-screen overlay.

### 2.8 Compact Break Notification
LookAway offers a compact notification design showing essential info with full details on hover, as an alternative to the full notification. BlinkBreak does not have a notification-based break system.

---

## 3. Smart Pause / Focus Detection

### 3.1 Meeting Detection
LookAway automatically detects when you're in a meeting or call by monitoring microphone usage. It auto-pauses breaks during meetings, shows a notification when a meeting is detected (with option to ignore), and deducts meeting duration from focus time. Users can exclude specific microphones, cameras, and apps from triggering detection. BlinkBreak has no meeting detection.

### 3.2 Video Playback Detection
LookAway detects when you're watching a video (YouTube, VLC, etc.) and pauses breaks automatically. It can work even when the video app is in the background (configurable), and excludes video editing apps (Final Cut Pro, DaVinci Resolve) and Spotify. BlinkBreak has no video detection.

### 3.3 Fullscreen Game Detection
LookAway automatically detects when a fullscreen game is running and pauses breaks. BlinkBreak has no game detection.

### 3.4 Deep Focus Apps Detection
LookAway lets users select specific apps (e.g., Xcode, Figma) where breaks should be paused automatically. This is user-configurable. BlinkBreak has no per-app focus detection.

### 3.5 Screen Recording Detection
LookAway pauses breaks when screen recording or screen sharing is detected. BlinkBreak has no screen recording detection.

### 3.6 Typing Detection
LookAway can postpone a break if you're in the middle of typing, waiting until you finish before showing the break. This is opt-in via settings. BlinkBreak has no typing detection.

### 3.7 Dragging Detection
LookAway waits for you to finish dragging operations before showing a break overlay. BlinkBreak has no dragging detection.

---

## 4. Scheduling & Office Hours

### 4.1 Work Schedule / Office Hours
LookAway supports scheduling so the app is only active during configured hours. Users can set start and end times, with end times supporting past-midnight (for night owls). Each day of the week can be configured independently with custom hours. BlinkBreak has no scheduling feature; it runs 24/7 when active.

### 4.2 Pause for Fixed Time
LookAway offers a menu bar option to pause the timer for a specific duration (e.g., 15 min, 30 min, 1 hour), after which it automatically resumes. BlinkBreak has no timed-pause feature; pausing is indefinite until manually resumed.

---

## 5. Wellness Reminders

### 5.1 Posture Reminders
LookAway provides periodic posture reminders as separate overlays (independent of break reminders). They are configurable in frequency (as low as 10 seconds), size, position, and volume. BlinkBreak has no posture reminders.

### 5.2 Blink Reminders
LookAway provides blink reminders with animation synced to a gentle sound. They offer configurable size, position, and frequency. BlinkBreak's core mission is about blink breaks but it doesn't have periodic blink reminders separate from the full break cycle.

### 5.3 Wellness Reminder Customization
LookAway lets users configure wellness reminders to only show on the main screen, during pauses, or to disable screen dimming when they appear. Frequency can be set in hh:mm:ss format. BlinkBreak has no wellness reminder system at all.

---

## 6. Customization

### 6.1 Custom Break Screen Backgrounds
LookAway supports custom background images and curated gradients for the break screen. In v2.0, this expanded to 7 animated background options with a preview button. BlinkBreak uses a single static background image with a blur effect applied via the Glur library.

### 6.2 Custom Break Screen Messages
LookAway lets users add unlimited custom text messages for short and long breaks, with a 256-character limit per message. Messages are randomly rotated. BlinkBreak has 5 hardcoded messages in `ReminderMessages.swift` with no user customization.

### 6.3 Custom Sounds
LookAway lets users customize sounds for break start and end, choose from included options (Tibetan bells, flute, etc.), or upload custom audio files. Volume is configurable per sound. BlinkBreak has only 2 hardcoded WAV files (`endReminder.wav` and `pause.wav`) with a simple on/off toggle.

### 6.4 Break Screen Customization Panel
LookAway has a dedicated "Customize break screen" settings section where users can hide/show the lock screen button, hide the current time display, and preview break backgrounds without starting a break. BlinkBreak has no break screen customization beyond what's hardcoded.

### 6.5 Menu Bar Customization
LookAway offers extensive menu bar options: show detailed info (time till next break, minutes without break), hide the icon entirely, switch between icon/text/both styles, and show paused/stopped states. BlinkBreak has a fixed menu bar popover with no customization options.

### 6.6 Notification Positioning
LookAway allows configuring the position of break notifications on screen. BlinkBreak does not have configurable notification positions.

---

## 7. Cross-Device Sync

### 7.1 iPhone Sync (LookAway Mirror)
LookAway syncs with an iPhone companion app called "LookAway Mirror." When the Mac takes a break, the iPhone automatically blocks distracting websites and non-essential apps via a pairing code system. BlinkBreak has no cross-device functionality.

### 7.2 iPad Sync
LookAway v2.0.1 added support for iPad sync in addition to iPhone. BlinkBreak has no iOS/iPadOS companion.

---

## 8. Integrations & Automation

### 8.1 Apple Calendar Integration
LookAway integrates with Apple Calendar to auto-pause during calendar events or focus blocks. Users can choose which calendars to monitor and optionally only pause for accepted events. BlinkBreak has no calendar integration.

### 8.2 AppleScript Support
LookAway supports controlling the app via AppleScript (start, stop, pause, resume, etc.) with documented commands. BlinkBreak has no AppleScript support.

### 8.3 Automations (AppleScripts & Shortcuts on Break)
LookAway lets users run AppleScripts or macOS Shortcuts automatically when a break starts or ends. Automations can be toggled off without deleting them. BlinkBreak has no automation system.

### 8.4 macOS Focus Filters
LookAway auto-pauses when a macOS Focus Mode (Do Not Disturb, etc.) is active. BlinkBreak has no integration with macOS Focus Modes.

### 8.5 Raycast Extension
LookAway can be controlled via a Raycast extension for pausing, postponing breaks, and other actions. BlinkBreak has no Raycast integration.

### 8.6 Keyboard Shortcuts (Comprehensive)
LookAway provides assignable keyboard shortcuts for: pause/resume timer, start break immediately, start long break, add 1 minute to timer, restart LookAway, 5-minute snooze, 15-minute snooze, open Quick Look, and configurable double-Escape behavior (snooze vs skip). BlinkBreak has no configurable keyboard shortcuts.

---

## 9. Session & State Management

### 9.1 Session State Restoration
LookAway restores timers and session state after the app restarts, keeping history accurate even if the app closes unexpectedly. BlinkBreak loses all session state on restart and starts fresh.

### 9.2 Cross-Midnight Session Handling
LookAway correctly handles sessions that span midnight, keeping active states and statistics accurate. BlinkBreak does not handle cross-midnight sessions explicitly.

### 9.3 Quick Look Panel
LookAway v2.0 replaced the simple menu bar dropdown with a full "Quick Look" mini-app panel showing current session info, Screen Score, and quick actions, accessible via a dedicated keyboard shortcut. BlinkBreak has a basic menu bar popover showing only the timer and play/stop button.

### 9.4 Idle Return Dialog ("Stepped Away?")
LookAway shows a dialog when you return from idle asking whether you took a break, with options dismissible via keyboard shortcuts. This helps keep stats accurate. BlinkBreak either restarts or continues the session automatically with no user prompt.

---

## 10. UI & Design

### 10.1 Onboarding Experience
LookAway has a fully revamped onboarding flow with custom timer configuration and guidance. BlinkBreak has no onboarding experience.

### 10.2 macOS Tahoe / Liquid Glass Design
LookAway v2.0 features a full redesign inspired by macOS Tahoe with Liquid Glass design language applied to buttons, notifications, wellness reminders, Quick Look, and stats views. BlinkBreak uses a custom dark glassmorphic design but does not follow the latest macOS design trends.

### 10.3 Lock Screen Button on Break Screen
LookAway has a button on the break overlay to instantly lock the Mac screen. BlinkBreak has no lock screen option on the break overlay.

### 10.4 Auto-Lock Screen During Breaks
LookAway can automatically lock the screen when a break starts. BlinkBreak has no auto-lock feature.

### 10.5 App Refocus After Break
LookAway correctly restores focus to the previous application after a break ends. BlinkBreak does not explicitly handle app refocus after the full-screen overlay dismisses.

### 10.6 Sleep Prevention During Long Breaks
LookAway prevents the Mac from automatically going to sleep during long breaks. BlinkBreak has no sleep prevention.

### 10.7 Escape Key to Skip Break
LookAway supports pressing Escape twice to skip a break (configurable to snooze instead). BlinkBreak has no keyboard dismiss for the break overlay.

---

## 11. Licensing & Updates

### 11.1 License Management Dashboard
LookAway has a full license management system with a dashboard to deactivate devices, add seats, and renew update subscriptions. BlinkBreak has no licensing system.

### 11.2 Beta Release Channel
LookAway supports switching to a beta release channel from settings to access early features. BlinkBreak has no release channel system.

### 11.3 Update Notifications in Menu Bar
LookAway shows new version availability in the menu bar dropdown. BlinkBreak has no in-app update notification system.

---

## Summary Table

| Category | LookAway | BlinkBreak | Gap |
|---|---|---|---|
| Stats Dashboard | Yes | No | **Missing** |
| Screen Score / Gamification | Yes | No | **Missing** |
| Long Breaks | Yes | No | **Missing** |
| Snooze / Postpone System | Yes | Skip only | **Partial** |
| Postpone Limits | Yes | No | **Missing** |
| Pre-Break Countdown | Yes | No | **Missing** |
| Meeting Detection | Yes | No | **Missing** |
| Video Playback Detection | Yes | No | **Missing** |
| Fullscreen Game Detection | Yes | No | **Missing** |
| Deep Focus Apps Detection | Yes | No | **Missing** |
| Screen Recording Detection | Yes | No | **Missing** |
| Typing / Dragging Detection | Yes | No | **Missing** |
| Work Scheduling / Office Hours | Yes | No | **Missing** |
| Timed Pause | Yes | No | **Missing** |
| Posture Reminders | Yes | No | **Missing** |
| Blink Reminders | Yes | No | **Missing** |
| Custom Backgrounds | Yes (7 animated + custom) | Single static image | **Missing** |
| Custom Messages | Yes (unlimited, user-defined) | 5 hardcoded | **Partial** |
| Custom Sounds | Yes (upload + curated) | 2 hardcoded WAVs | **Missing** |
| Break Screen Customization | Full settings panel | None | **Missing** |
| iPhone/iPad Sync | Yes | No | **Missing** |
| Apple Calendar Integration | Yes | No | **Missing** |
| AppleScript Support | Yes | No | **Missing** |
| Automations (Scripts/Shortcuts) | Yes | No | **Missing** |
| Focus Filters | Yes | No | **Missing** |
| Raycast Extension | Yes | No | **Missing** |
| Keyboard Shortcuts | Yes (comprehensive) | None | **Missing** |
| Session State Restoration | Yes | No | **Missing** |
| Quick Look Panel | Yes | Basic menu bar popover | **Partial** |
| Idle Return Dialog | Yes | No | **Missing** |
| Onboarding | Yes | No | **Missing** |
| Lock Screen on Break | Yes | No | **Missing** |
| App Refocus After Break | Yes | No | **Missing** |
| Sleep Prevention During Breaks | Yes | No | **Missing** |
| Escape Key to Dismiss Break | Yes | No | **Missing** |
| License Management | Yes | No | **Missing** |
| Beta Channel | Yes | No | **Missing** |
| Localization | English | EN/DE/FR/JA (partial) | **BlinkBreak ahead** |

---

## Recommended Priority for Implementation

### High Priority (Core Differentiators)
1. **Statistics Dashboard** - Users expect to see their screen time data
2. **Snooze System with Limits** - Critical UX improvement over "skip only"
3. **Keyboard Shortcuts** - Power user expectation for macOS apps
4. **Session State Restoration** - Data loss on restart is a poor experience
5. **Pre-Break Countdown** - Reduces jarring full-screen takeover
6. **Onboarding** - First-launch experience sets the tone

### Medium Priority (Smart Features)
7. **Meeting Detection** - Major pain point if breaks interrupt calls
8. **Video Playback Detection** - Prevents annoying interruptions
9. **Long Breaks** - Health benefit of extended rest periods
10. **Work Scheduling** - Essential for work-life boundary control
11. **Custom Sounds & Backgrounds** - Easy personalization wins
12. **Custom Break Messages** - User engagement feature

### Lower Priority (Nice-to-Have)
13. **iPhone/iPad Sync** - Differentiator but complex to build
14. **Calendar Integration** - Nice but not critical
15. **AppleScript/Automations** - Power user niche
16. **Wellness Reminders** - Additional value layer
17. **Screen Score/Gamification** - Engagement hook
18. **Deep Focus/Game/Screen Recording Detection** - Edge case handling
