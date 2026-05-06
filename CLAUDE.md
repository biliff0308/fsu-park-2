# CLAUDE.md — FSU Park v2

## Project

FSU Park is the Florida State University parking availability app for Android. This is the v2 greenfield rebuild, replacing a legacy Java/View-system app that has been feature-frozen. The legacy app remains in production until v2 ships under the same Play Store listing.

The motivation is craftsmanship, not deliverables. Bill is the sole developer. The goal is to produce a great Android product using modern tools and patterns, not to ship the fastest possible replacement. When choices arise between expedient and correct, choose correct.

Bill plans to retire in approximately two years. The codebase should be legible and maintainable by someone who inherits it.

## Stack

- **Language:** Kotlin
- **UI:** Jetpack Compose, Material 3
- **Architecture:** Single activity, MVVM with unidirectional data flow
- **State:** ViewModel + StateFlow, immutable UI state classes
- **Async:** Coroutines + Flow
- **DI:** Hilt
- **Networking:** Retrofit + OkHttp + kotlinx.serialization
- **Maps:** Google Maps SDK with maps-compose
- **Notifications:** Firebase Cloud Messaging (FCM)
- **Build:** Gradle Kotlin DSL, JVM 17

## SDK / Build

- `compileSdk` 36
- `targetSdk` 36
- `minSdk` 26 (Android 8.0)
- `applicationId` and `namespace`: `edu.fsu.transportation.fsugaragespotcounter`
- `versionCode` starts at 56 for v2 release (legacy is at 55)
- `versionName` follows semver: `2.0.0` for first v2 release
- App label: "FSU Park"
- Orientation: unlocked (legacy was portrait-only)

## Signing

The v2 app must be signed with the existing keystore to retain the Play Store listing.

- Keystore: `C:\Development\obsmbapp_release.keystore`
- Alias: `obsmbapp`
- **Passwords must not be in `build.gradle.kts` or version control.** Read from `local.properties` (gitignored) or environment variables.

## Architecture rules

These are non-negotiable conventions for this project. Apply them on every new screen, feature, and refactor.

### Layering

- **UI layer (Compose + ViewModel):** Composables observe `StateFlow` from ViewModels. No business logic in composables. No direct repository access from composables.
- **Domain/ViewModel layer:** ViewModels expose immutable UI state and accept user intents. No Android framework dependencies in ViewModels (no `Context`, no `Resources`, no Android imports). ViewModels are unit-testable with plain JUnit.
- **Data layer:** Repositories own data sources. Network DTOs are kept internal to the data layer; mapping to domain models happens at the repository boundary. UI never sees a Retrofit response type.

### State

- One immutable `data class` per screen representing complete UI state.
- ViewModels expose `StateFlow<UiState>`, never mutable state.
- User actions are modeled as sealed classes or sealed interfaces (intents/events) sent to the ViewModel.

### Navigation

- Compose Navigation with type-safe routes (Kotlin serialization-based routes preferred).
- Single `NavHost` at the top level. Screens are composables, not activities or fragments.

### Dependency injection

- Hilt for all dependency wiring. No service locators, no manual factories outside Hilt modules.
- ViewModels use `@HiltViewModel` and `hiltViewModel()` in composables.
- Networking, repositories, and platform services (notification manager, location provider, etc.) are provided via Hilt modules.

### Testing

- Unit tests for ViewModels and repositories. Mock the layer below, not the ViewModel itself.
- Compose previews for every screen and every meaningful state variant (loading, success, error, empty).
- Integration/UI tests are nice-to-have, not required for every feature, but the architecture should not preclude them.

### Networking

- All HTTPS. Standard certificate validation. No custom `TrustManager`, no custom `HostnameVerifier`. (The legacy hostname-mismatch workaround is being eliminated by fixing the server cert before v2 ships.)
- Retrofit interfaces with suspend functions returning domain models (after mapping in the repository).
- kotlinx.serialization for JSON parsing, with `@Serializable` data classes for DTOs.
- OkHttp interceptors for logging (debug builds only) and any cross-cutting concerns.
- Errors surface as sealed `Result` types (success/error/loading), not exceptions thrown at the UI.

### Resources

- No hardcoded strings in composables. All user-facing text in `strings.xml`.
- Theme via Material 3 `MaterialTheme`. Colors, typography, shapes defined centrally and referenced by token, not by literal value.
- Vector drawables for icons. Adaptive icon for the launcher.

## Working style

Bill prefers paired programming with AI, not autonomous code generation. Optimize for understanding and ownership, not speed.

- **Smaller diffs over larger ones.** Prefer one focused change at a time. When a task spans multiple files, walk through them one at a time unless the user asks for a sweep.
- **Explain before changing.** When making a non-trivial change, explain what you're about to do and why before doing it. The user wants to follow the reasoning, not just see the result.
- **Ask before significant decisions.** If a task can be implemented multiple ways with real trade-offs, surface the options before picking one. Do not silently choose.
- **Disagree when you disagree.** If the user proposes something that's incorrect, suboptimal, or inconsistent with the conventions in this file, push back with reasoning. Do not capitulate to be agreeable.
- **No affirmations or flattery.** Skip "great question," "excellent point," "you're absolutely right." Match the register of a senior colleague who respects the user's time.
- **No hype.** Do not call ideas brilliant or insightful. Do not perform enthusiasm. The work is the work.
- **Honest uncertainty.** When unsure, say so. "I'd want to verify this" is more useful than a confident wrong answer.
- **Brevity.** Get to the point. Bill reads carefully and doesn't need padding.

## Communication for code changes

When proposing or making a change:

1. State what's changing and why in plain prose.
2. Show the diff or the new code.
3. Note anything that changed elsewhere as a consequence.
4. Flag anything that should be tested or verified before moving on.

## Data feeds

The app consumes two JSON feeds from FSU infrastructure. The feeds are public but unpublicized.

### Availability feed

- **URL:** `https://obs-web05.its.fsu.edu/count.json` (post-cert-fix; currently has hostname mismatch issue being resolved before v2 ships)
- **Update frequency:** Server-side updates every 5 minutes
- **Client polling:** Every 5 minutes while app is in foreground; refresh on resume; pull-to-refresh on user demand
- **Content:** Per-lot space counts. Will be enhanced with LPR (license plate recognition) data to include faculty spaces and improve accuracy. The LPR-merged feed will retain the same JSON shape and keys as the current feed (an XML-to-JSON converter handles the merge server-side).

### Announcements feed

- **URL:** `https://obs-web05.its.fsu.edu/_announce.json` (post-cert-fix)
- **Update frequency:** Server-side updates when TAPS submits a new announcement via the TAPS web app
- **Client polling:** On app open, on resume, and every 30 minutes while in foreground. No more aggressive than that. (FCM push handles the time-sensitive notification path; polling exists as a backstop and to keep the announcements list current when the app is opened.)
- **Content:** Parking-related announcements (closures, events, policy notices).

### Feed handling principles

- Decouple the two feeds' refresh cadences. They have different update needs.
- Cache last-known-good response. If a refresh fails, show stale data with a clear staleness indicator, not an error screen.
- Network errors should not blank the UI. Surface them inline (banner, snackbar) while keeping previously-loaded content visible.

## Notifications

The app sends one type of notification: an alert when TAPS posts a new announcement.

**Trigger and delivery:**
The TAPS announcement web app, on submit, writes the new entry to `_announce.json` and fires an FCM topic broadcast to the "announcements" topic. All installed app instances subscribed to the topic receive the message.

**Quiet hours:**
8:00 PM to 7:00 AM Eastern time. Enforced server-side: during quiet hours, the FCM send is skipped. The announcement still appears in the file and is visible in the app on next refresh — only the push notification is suppressed. No client-side quiet-hours logic.

**Content:**
- Title: "FSU Park announcement"
- Body: First ~80 characters of the announcement text, ellipsized if longer
- Tap action: Open app to announcements view, scroll to the new entry
- Notification channel: Single "Announcements" channel

**User control:**
None beyond the OS-level POST_NOTIFICATIONS permission flow (Android 13+) and system notification channel settings. No in-app toggle.

**Default state:**
Enabled on first launch, subject to the user granting POST_NOTIFICATIONS permission. The app subscribes to the "announcements" FCM topic on first launch and on token refresh.

**Out of scope for the app, but required for notifications to work in production:**
- Firebase project setup (separate Firebase project for FSU Park, `google-services.json` added to the app, server key obtained for the TAPS web app to send with).
- Modification of the TAPS announcement web app's submit handler to fire the FCM topic broadcast and enforce EST quiet-hours suppression.

## Features in v2 scope

1. **Map-first UI replacing the tab layout.** Map is the primary screen. Lots displayed as markers with availability counts and color coding. Announcements accessible via a bottom sheet or top-bar entry, not a tab.
2. **Map integration.** Google Maps with custom markers per lot. Marker design conveys availability state at a glance.
3. **LPR-enhanced data.** Once the merged feed is live server-side, the client consumes it transparently — no client-side merging.
4. **Announcement notifications.** FCM push when TAPS posts a new announcement. See Notifications section for full design.

## Features explicitly out of scope for v2

- Android Auto integration
- Voice / Google Assistant integration
- Wear OS / watch integration
- User-contributed occupancy data / predictive analytics (ParkZen handles that space)
- 3D wayfinding inside garages
- Per-lot availability threshold notifications
- In-app notification settings beyond OS-level controls

These may return in future iterations. Not now.

## Build and run

(To be filled in once project is scaffolded.)

- Build debug: `./gradlew assembleDebug`
- Build release: `./gradlew assembleRelease`
- Run unit tests: `./gradlew test`
- Run lint: `./gradlew lint`

## Known constraints

- The legacy app is feature-frozen except for critical bug fixes and the planned cert-cleanup release.
- v2 must reuse the existing Play Store listing, which requires the same `applicationId` and the same signing key.
- The feeds are FSU infrastructure managed by Bill. Server-side changes are possible and preferred over client-side workarounds when the choice arises.
- The TAPS announcement web app is also under Bill's control and will be modified to drive FCM sends for v2 notifications.

## Out-of-scope conversations

This file is for project conventions and shared context. Conversations about Bill's broader career, retirement timeline, or non-FSU-Park projects belong elsewhere. Stay on the work.
