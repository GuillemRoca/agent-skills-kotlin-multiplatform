---
name: code-review-and-quality
description: >-
  Use when reviewing Kotlin Multiplatform code (own or a teammate's),
  self-reviewing before creating a PR, or running a final quality gate on a
  feature. Five-axis review framework: Correctness, Readability, Architecture,
  Security, Performance — with KMP source-set placement checks, per-target
  behavior checks, and categorized findings.
---

# Code Review and Quality

## Overview

Review code across five axes: Correctness, Readability, Architecture, Security, and Performance. The approval standard: "Approve when it definitely improves overall code health of the system." Every finding is categorized and actionable. In a KMP codebase, Architecture additionally polices where code lives (source-set placement) and Correctness polices how it behaves on every target.

## When to Use

- Reviewing a pull request (own or teammate's)
- Self-review before creating a PR
- Requested code quality check on a specific file or module
- After completing a feature (final quality gate)

**Skip when:**
- Not reviewing code (this is a review skill, not a writing skill — see `incremental-implementation`)
- Auditing a whole codebase for security only — go straight to `security-and-hardening`

## Core Process

### Step 1: Understand Context

1. **Read the PR description** — what problem does it solve?
2. **Read the spec or ticket** — does the PR match the stated goal?
3. **Check the diff size** — target ~100 lines, flag >1000 lines for splitting
4. **Note which source sets the diff touches** — a `commonMain` change ships to every target; scale review depth accordingly

### Step 2: Review Tests First

5. **Start with test files** — they reveal the intended behavior
6. **Check for:**
   - Are critical paths tested in `commonTest` with kotlin.test, not only in `jvmTest`?
   - Are edge cases covered (null, empty, error, boundary values)?
   - Do test names describe behavior?
   - Are tests independent (no shared mutable state)?
   - Are fakes hand-written? MockK, Robolectric, and JUnit5 are JVM-only and must not appear in common code — see `multiplatform-testing`

### Step 3: Five-Axis Review

#### Axis 1: Correctness

7. **Verify behavior matches intent:**
   - Does the code handle all states? (loading, success, error, empty)
   - Are nulls handled safely? (no `!!`, proper `?.` chains)
   - Are coroutines structured correctly? (proper scope, cancellation, no `GlobalScope`)
   - Do Room KMP queries match the schema, and are migrations tested?

8. **Check per-target behavior differences** for anything in `commonMain`:
   - **Dispatchers:** `Dispatchers.IO` does not exist on js/wasm — use `Dispatchers.Default` or inject a dispatcher. `Dispatchers.Main` on desktop needs `kotlinx-coroutines-swing` in `desktopApp`
   - **Blocking:** `runBlocking` does not exist on js/wasm — suspend all the way, or keep blocking calls in JVM/native source sets
   - **Time:** `java.time` is JVM/Android-only. Use `kotlin.time` and kotlinx-datetime in common code; review timezone assumptions per target
   - **Locale and formatting:** `String.format` is JVM-only, and number/date formatting differs per platform. Locale-sensitive output belongs behind an interface with platform implementations

```kotlin
// BAD in commonMain: compiles only where java.* exists; IO is JVM-specific
suspend fun today() = withContext(Dispatchers.IO) {
    java.time.LocalDate.now()
}

// GOOD in commonMain: portable APIs, injected dispatcher
class TaskLoader(
    private val backgroundDispatcher: CoroutineDispatcher = Dispatchers.Default,
) {
    suspend fun today(clock: Clock = Clock.System): LocalDate =
        withContext(backgroundDispatcher) {
            clock.todayIn(TimeZone.currentSystemDefault())
        }
}
```

#### Axis 2: Readability

9. **Kotlin-specific readability:**

```kotlin
// GOOD: idiomatic Kotlin
val activeTask = tasks.firstOrNull { !it.completed }
    ?: return TaskListUiState.Empty

// BAD: Java-style
var activeTask: Task? = null
for (task in tasks) {
    if (!task.completed) {
        activeTask = task
        break
    }
}
if (activeTask == null) return TaskListUiState.Empty
```

10. **Check for:**
    - Clear naming (functions describe actions, variables describe content)
    - Appropriate use of `when` expressions, `let`/`also`/`apply`, extension functions
    - Functions under ~40 lines
    - Sealed classes/interfaces for exhaustive state handling
    - No nested callbacks (use coroutines)

#### Axis 3: Architecture

11. **Verify layer boundaries:**
    - UI layer only calls ViewModel (never Repository/DAO directly)
    - Domain layer has no platform dependencies
    - Data layer implements domain interfaces
    - No business logic in Composables
    - Dependencies wired with Metro (`@Inject` constructors, `@ContributesBinding`), not service locators

12. **Verify KMP placement** (see `multiplatform-architecture`):
    - **Most-common source set:** is new code in the most common source set it can compile in? Logic using only stdlib + KMP libraries belongs in `commonMain`, not duplicated per platform
    - **Needless expect/actual:** would an interface in `commonMain` plus platform implementations via DI do? Reserve `expect`/`actual` for per-platform one-liners or classes that must inherit platform types
    - **JVM-only leaks:** no `java.*` or `android.*` imports in `commonMain`; platform APIs live in `androidMain`/`iosMain`/`jvmMain`
    - **Thin entry points:** `androidApp`/`desktopApp`/`webApp`/`iosApp` contain only bootstrap (activity, `main.kt`, viewport, graph creation). All expect/actual declarations and business logic live in `shared`

```kotlin
// BAD: expect/actual ceremony where an interface would do
expect class AnalyticsTracker { fun track(event: String) }

// GOOD: interface in commonMain, platform impl contributed via Metro
interface AnalyticsTracker { fun track(event: String) }

// androidMain
@ContributesBinding(AppScope::class)
@Inject
class FirebaseAnalyticsTracker : AnalyticsTracker { /* ... */ }
```

#### Axis 4: Security

13. **Check for:**
    - No hardcoded secrets in any source set — `commonMain` constants ship to every target, including the fully inspectable js/wasm bundle
    - Input validation for user-provided data (deep links, API responses)
    - No logging of sensitive data (Kermit calls with tokens or passwords)
    - Secrets at rest behind the platform `SecretStorage` implementations
    - See `security-and-hardening` for the comprehensive checklist

#### Axis 5: Performance

14. **Check for:**
    - N+1 query patterns in Room KMP
    - Unbounded data loading (page large datasets)
    - Unnecessary recompositions in Compose (unstable parameters, lambda allocations)
    - Heavy work on the UI thread (inject a background dispatcher)
    - Leaked scopes (a `CoroutineScope` outliving its owner)
    - See `performance-optimization` for the comprehensive checklist

### Step 4: Categorize Findings

15. **Use severity categories:**

| Category | Description | Action Required |
|----------|-------------|----------------|
| **Critical** | Security vulnerability, data loss, crash | Must fix before merge |
| **Important** | Missing tests, architecture violation, bug risk | Should fix before merge |
| **Suggestion** | Better Kotlin idiom, readability improvement | Optional, author's discretion |
| **Nit** | Formatting, naming preference | Optional |
| **FYI** | Context or explanation, no action needed | Informational |

16. **Format findings:**
```
**[Critical]** `shared/src/commonMain/kotlin/data/TaskApi.kt:12` — API key
hardcoded in commonMain. It ships in every binary, including the readable
wasm bundle. Move it behind the platform SecretStorage boundary.

**[Important]** `shared/src/commonMain/kotlin/ui/TaskListViewModel.kt:23` —
Uses `Dispatchers.IO`, which does not exist on js/wasm. Inject a dispatcher
and default to `Dispatchers.Default`.

**[Suggestion]** `shared/src/androidMain/kotlin/Formatters.kt:8` — No
android.* dependency here; move to commonMain so iOS and desktop reuse it.
```

### Step 5: Verify Build and Tests

17. **Before approving:**
    - `./gradlew :shared:allTests` passes (runs the test suite on every target)
    - If the diff touches `commonMain`, compile at least one Apple target — Kotlin/Native surfaces errors JVM compilation cannot: `./gradlew :shared:compileKotlinIosSimulatorArm64` (requires macOS + Xcode; delegate to CI when reviewing on Linux)
    - Entry points still build: `./gradlew :androidApp:assembleDebug :desktopApp:build`
    - Lint and detekt clean, if configured

## Common Rationalizations

| Shortcut | Why It Fails |
|----------|-------------|
| "LGTM" without reading the code | Rubber-stamp reviews miss bugs. They also train teammates to skip reviews. |
| "I'll clean it up later" | Later never comes. Fix it now or create a tracked issue. |
| "It works on Android, so it's fine" | commonMain ships to five targets. A green jvmTest says nothing about iOS dispatchers or missing wasm APIs. |
| "The author knows best" | Fresh eyes catch blind spots. That's the point of review. |
| "It's just a small change" | Small changes in the wrong source set create platform debt: duplicated logic and needless expect/actual. |

## Red Flags

- PR over 1000 lines without justification
- No tests in the PR, or tests only in `jvmTest` for a `commonMain` change
- `!!` (non-null assertion) without justification
- `GlobalScope` usage
- `java.*` or `android.*` imports in `commonMain`
- New `expect`/`actual` pairs where an interface plus DI would do
- Business logic or expect/actual declarations in entry-point modules
- Mutable state exposed from ViewModel (public `MutableStateFlow`)
- Secrets or API keys in source code
- `@Suppress` annotations without explanatory comments

## Verification

- [ ] All five axes reviewed (Correctness, Readability, Architecture, Security, Performance)
- [ ] Tests reviewed first (coverage, edge cases, naming, commonTest placement)
- [ ] Per-target behavior checked for commonMain changes (dispatchers, time, locale)
- [ ] New code sits in the most common source set it can compile in
- [ ] Findings categorized (Critical, Important, Suggestion, Nit, FYI)
- [ ] Critical findings resolved before approval
- [ ] Apple target compiled for commonMain diffs (`./gradlew :shared:compileKotlinIosSimulatorArm64`)
- [ ] PR size reasonable (~100 lines, flagged if >1000)
- [ ] `./gradlew :shared:allTests` passes
