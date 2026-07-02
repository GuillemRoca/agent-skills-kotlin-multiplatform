---
name: observability-and-instrumentation
description: >-
  Use when adding logging, crash reporting, or analytics to a Kotlin
  Multiplatform app, or before shipping a release that must be watched across
  Android, iOS, desktop, and web. Covers Kermit as the shared logging facade,
  Crashlytics with Kotlin-aware iOS crash bridging, a common AnalyticsTracker
  wired by DI, and per-store release-health monitoring.
---

# Observability and Instrumentation

## Overview

You cannot fix what you cannot see, and in KMP the same shared code runs on four platforms with four different visibility stories: Android has Crashlytics and Play Vitals, iOS needs Kotlin-aware crash bridging to be readable at all, and desktop and web report nothing unless you build the reporting path yourself. Instrument from `commonMain` before launch — a logging facade, a crash-reporter interface, and an analytics interface, each with platform implementations wired by DI.

## When to Use

- Adding a feature whose failures would be invisible without instrumentation (payments, sync, background work)
- Setting up or auditing crash reporting or analytics on any KMP target
- Before a release that will be watched during staged rollout (see `shipping-and-launch`)
- Investigating field-only issues where local reproduction failed (see `debugging-and-error-recovery`)
- iOS crash reports show mangled Kotlin/Native frames nobody can read

**Skip when:** prototypes or internal builds that will never reach users — but wire observability in before the first external release, not after.

## Core Process

### Step 1: Kermit as the Shared Logging Facade

One logging API in `commonMain`; platform-appropriate writers underneath — Logcat on Android, OSLog on iOS, console on desktop and web. Kermit's `platformLogWriter()` picks these for you.

```kotlin
// commonMain — tags + severity, structured key=value messages
private val log = Logger.withTag("sync")

// GOOD: one event, one line, greppable, no PII
log.w { "sync_failed attempt=$attempt reason=${reason.name}" }

// BAD: PII — log output leaks via logcat, consoles, and bug reports
log.d { "sync failed for $email token=$token" }
```

Configure severity and writers once at startup; in release builds drop Debug/Verbose and mirror Warn/Error into the crash reporter as breadcrumbs and non-fatals:

```kotlin
// commonMain
fun configureLogging(isRelease: Boolean, crashReporter: CrashReporter) {
    Logger.setMinSeverity(if (isRelease) Severity.Info else Severity.Verbose)
    Logger.setLogWriters(buildList {
        add(platformLogWriter())
        if (isRelease) add(CrashReporterLogWriter(crashReporter))
    })
}

class CrashReporterLogWriter(private val reporter: CrashReporter) : LogWriter() {
    override fun isLoggable(tag: String, severity: Severity) = severity >= Severity.Warn
    override fun log(severity: Severity, message: String, tag: String, throwable: Throwable?) {
        reporter.logBreadcrumb("$tag: $message")
        throwable?.let(reporter::recordNonFatal)
    }
}
```

Rules: no PII, tokens, or request bodies at any severity; stable `event key=value` shapes over prose; `println` and platform log calls (`Log.d`, `NSLog`, `console.log`) never appear in `commonMain`.

### Step 2: Crash Reporting on Android and iOS — Crashlytics with Kotlin-Aware Bridging

Define the reporter as an interface in `commonMain` — interface plus DI beats `expect`/`actual` here because it is fakeable in `commonTest` (see `platform-interop`):

```kotlin
// commonMain
interface CrashReporter {
    fun setKey(key: String, value: String)
    fun setUserId(pseudonymousId: String)      // NEVER an email or real identifier
    fun logBreadcrumb(message: String)
    fun recordNonFatal(throwable: Throwable)
}
```

```kotlin
// androidMain — Metro binds it into the shared graph
@ContributesBinding(AppScope::class)
@SingleIn(AppScope::class)
@Inject
class CrashlyticsCrashReporter : CrashReporter {
    private val crashlytics = FirebaseCrashlytics.getInstance()
    override fun setKey(key: String, value: String) = crashlytics.setCustomKey(key, value)
    override fun setUserId(pseudonymousId: String) = crashlytics.setUserId(pseudonymousId)
    override fun logBreadcrumb(message: String) = crashlytics.log(message)
    override fun recordNonFatal(throwable: Throwable) = crashlytics.recordException(throwable)
}
```

iOS needs one extra move. When a Kotlin exception escapes to the Swift/Objective-C boundary, the app dies as a native crash and Crashlytics records the Kotlin/Native runtime's termination path: mangled `kfun:` symbols with no Kotlin file or line numbers, and every Kotlin crash collapsing into the same unreadable signature. Unsymbolicated K/N traces are useless — you cannot tell a JSON parsing crash from a coroutine cancellation bug. Bridge Kotlin stack traces with CrashKiOS (Touchlab), which installs Kotlin's unhandled-exception hook and reports real Kotlin frames to Crashlytics:

```kotlin
// iosMain — call at app start, before any other shared code runs
fun setupCrashReporting() {
    enableCrashlytics()                        // CrashKiOS: Kotlin frames, symbolicated
    setCrashlyticsUnhandledExceptionHook()
}
```

Complete the pipeline in the release lanes (see `ci-cd-and-automation`): upload dSYMs on iOS (Crashlytics run-script phase) and the R8 mapping file on Android (Crashlytics Gradle plugin) — obfuscated traces are noise on both platforms.

Record non-fatals for caught-but-abnormal paths — a swallowed exception is an invisible bug:

```kotlin
suspend fun sync(): SyncResult = try {
    api.sync()
} catch (e: SyncConflictException) {
    crashReporter.recordNonFatal(e)            // handled, but visible from the field
    SyncResult.Conflict
}
```

### Step 3: Crash Visibility on Desktop and Web

No store, no crash SDK — build the reporting path yourself: capture uncaught exceptions and POST them to your backend.

```kotlin
// commonMain — shared reporting path over Ktor
class BackendErrorReporter(private val client: HttpClient) {
    suspend fun report(throwable: Throwable, context: Map<String, String> = emptyMap()) {
        runCatching {                          // never crash while reporting a crash
            client.post("$BASE_URL/client-errors") {
                contentType(ContentType.Application.Json)
                setBody(ErrorReport(
                    message = throwable.message ?: "unknown",
                    stackTrace = throwable.stackTraceToString(),
                    platform = currentPlatform().name,
                    context = context,
                ))
            }
        }
    }
}
```

Catch coroutine failures in the shared application scope, and platform-level uncaught exceptions in each entry point:

```kotlin
// commonMain — application-level coroutine scope
val appScope = CoroutineScope(
    SupervisorJob() + Dispatchers.Default + CoroutineExceptionHandler { _, e ->
        Logger.e(e) { "uncaught_coroutine_exception" }
        errorReporter.reportAsync(e)           // fire-and-forget wrapper around report()
    }
)
```

```kotlin
// desktopApp/src/main/kotlin/main.kt
fun main() {
    Thread.setDefaultUncaughtExceptionHandler { _, e ->
        runBlocking { errorReporter.report(e, mapOf("origin" to "uncaught_thread")) }
    }
    application { Window(onCloseRequest = ::exitApplication) { App() } }
}
```

```kotlin
// webApp/src/webMain/kotlin/main.kt — without this, a web crash is a silent blank page
fun main() {
    window.onerror = { message, _, _, _, _ ->
        errorReporter.reportAsync(RuntimeException(message.toString()))
        false
    }
    ComposeViewport { App() }
}
```

### Step 4: Analytics Behind a Common Interface

Analytics and crash reporting stay separate pipelines: different consumers (product vs engineering), different retention, different consent and PII rules. Define typed events and one interface in `commonMain`; wire platform implementations with Metro DI.

```kotlin
// commonMain
sealed interface AnalyticsEvent {
    val name: String
    val params: Map<String, String> get() = emptyMap()

    data class ScreenView(val screen: String) : AnalyticsEvent {
        override val name get() = "screen_view"
        override val params get() = mapOf("screen" to screen)
    }
    data class TaskCompleted(val source: String) : AnalyticsEvent {
        override val name get() = "task_completed"
        override val params get() = mapOf("source" to source)
    }
}

interface AnalyticsTracker {
    fun track(event: AnalyticsEvent)
}
```

```kotlin
// androidMain (iosMain mirrors this against the Firebase iOS SDK)
@ContributesBinding(AppScope::class)
@SingleIn(AppScope::class)
@Inject
class FirebaseAnalyticsTracker : AnalyticsTracker {
    override fun track(event: AnalyticsEvent) {
        Firebase.analytics.logEvent(event.name) {
            event.params.forEach { (key, value) -> param(key, value) }
        }
    }
}

// jvmMain (desktop) and webMain: batch and POST to your analytics endpoint
@ContributesBinding(AppScope::class)
@SingleIn(AppScope::class)
@Inject
class BackendAnalyticsTracker(
    private val client: HttpClient,
    private val scope: CoroutineScope,
) : AnalyticsTracker {
    override fun track(event: AnalyticsEvent) {
        scope.launch { client.post("$BASE_URL/analytics") { setBody(event.toDto()) } }
    }
}
```

ViewModels depend on `AnalyticsTracker`, never on a vendor SDK — and `commonTest` asserts events with a fake (see `multiplatform-testing`):

```kotlin
class FakeAnalyticsTracker : AnalyticsTracker {
    val events = mutableListOf<AnalyticsEvent>()
    override fun track(event: AnalyticsEvent) { events += event }
}
```

### Step 5: Release Health per Store — Know Each Scoreboard

Each platform has a different scoreboard; scope your expectations to it:

| Platform | Scoreboard | Notes |
|---|---|---|
| Android | Play Vitals | user-perceived crash-rate bad-behavior threshold 1.09%, ANR 0.47%; exceeding suppresses store visibility. ANRs appear here, not in your crash SDK. |
| iOS | App Store Connect Organizer / Xcode Metrics | crashes, hangs, launch time — only from opted-in users; needs dSYMs plus Step 2 bridging to show Kotlin frames |
| Desktop | none | your backend `client-errors` endpoint is the only signal; alert on it |
| Web | none | uncaught-error reports plus hosting/CDN metrics; without `window.onerror` a crash is a silent blank page |

Rollout doctrine (see `shipping-and-launch`):

- Write abort criteria before rolling: e.g. "halt at crash rate > 0.5% on the new version, on any platform"
- Compare version-over-version, not absolute — a new crash cluster at 5% rollout predicts the 100% disaster
- Alerting: Crashlytics velocity alerts (Android + iOS), a threshold alert on the client-errors endpoint (desktop + web)
- Triage hint: a spike on all platforms at once points at `commonMain`; a single-platform spike points at an `actual` implementation or a platform SDK

## Common Rationalizations

| Shortcut | Why It Fails |
|----------|-------------|
| "Crashlytics is set up on Android, we're covered" | Three of four platforms report nothing: iOS shows mangled K/N frames without bridging, desktop and web crash silently. |
| "`println` is fine in shared code" | No severity, no tags, no release policy, and no route to a crash reporter. Kermit costs one dependency and works on every target. |
| "We'll add the iOS crash bridging later" | Until then every Kotlin crash on iOS collapses into one unreadable native signature — on the platform where you can least reproduce locally. |
| "Analytics can double as crash reporting" | Different pipelines by design: consent, sampling, retention. Analytics SDKs legitimately drop events under exactly the conditions that surround a crash. |
| "Vitals and Organizer look fine, ship it" | Store dashboards lag by days and cover only two of four platforms. Velocity alerts and your own error endpoint are the early-warning system. |

## Red Flags

- iOS crash reports full of mangled `kfun:` Kotlin/Native frames
- `println`, `Log.d`, `NSLog`, or `console.log` calls in `commonMain`
- `catch (e: Exception) { }` with no `recordNonFatal` — swallowed failures are invisible
- Vendor analytics SDK types referenced in composables or ViewModels instead of `AnalyticsTracker`
- Desktop or web entry points with no uncaught-exception handler
- Log lines containing emails, tokens, or request bodies
- Release lanes without dSYM (iOS) or R8 mapping (Android) upload
- Staged rollout underway with no written abort criteria

## Verification

- [ ] Kermit is the only logging API in `commonMain`; release builds drop Debug/Verbose
- [ ] `CrashReporter` interface lives in `commonMain` with DI-bound implementations on all shipping targets
- [ ] Kotlin exception hook (CrashKiOS-style) enabled at iOS app start; dSYM and R8 mapping uploads automated in release lanes
- [ ] Desktop and web uncaught exceptions and coroutine failures reach the backend endpoint
- [ ] Changed flows record non-fatals with keys — grep the diff for empty `catch` blocks
- [ ] No PII, tokens, or bodies in any log or analytics call — grep the diff for log calls
- [ ] `commonTest` asserts expected analytics events through `FakeAnalyticsTracker`
- [ ] Rollout abort criteria written down with thresholds and an owner
- [ ] Shared tests pass on all targets: `./gradlew :shared:allTests`
