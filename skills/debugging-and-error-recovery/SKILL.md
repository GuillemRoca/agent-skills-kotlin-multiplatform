---
name: debugging-and-error-recovery
description: >-
  Use when encountering unexpected behavior, crashes, or test failures in a
  Kotlin Multiplatform + Compose Multiplatform project. Six-step triage from
  reproduction to regression guard that first isolates WHERE the bug lives
  (commonMain logic, a platform actual, or one platform's rendering), then
  applies the right per-platform toolbox.
---

# Debugging and Error Recovery

## Overview

Stop-the-Line Rule: when unexpected behavior occurs, halt feature work. Debugging in KMP is systematic, not guesswork, and the first question is always WHERE: shared `commonMain` logic, a platform `actual`, or a single platform's rendering. Reproduce the bug, isolate the layer, localize with platform tools, fix the root cause (not the symptom), and guard with a regression test.

## When to Use

- Unexpected crash or exception on any target
- Test failure (commonTest or a platform test source set)
- Behavior that differs between targets ("works on Android, broken on iOS")
- Behavior that does not match the spec
- Build failure for one target after seemingly unrelated changes

**Skip when:**
- The failure is already understood and has a failing test — go straight to `test-driven-development` and fix it.
- The problem is a measured performance regression — use `performance-optimization` instead.

**Stop-the-Line:** when you encounter unexpected behavior, stop feature work. Errors compound — a bug in step 3 makes steps 4-10 wrong.

## Core Process

### Step 1: Reproduce

Record which targets fail — that is the primary KMP debugging signal.

```
Targets failing: iOS (simulator + device) — Android/desktop/web OK
Build: debug (Xcode Debug scheme)
Steps: Launch → Tasks tab → pull to refresh → crash
Frequency: every time
Commit: abc123
```

Reproduce on the cheapest target that shows the bug. If shared logic is suspect, express the reproduction as a failing commonTest and run it on the JVM first — the fastest loop:

```bash
./gradlew :shared:jvmTest --tests "com.example.TaskRepositoryTest"
```

### Step 2: Isolate Where the Bug Lives

| Observation | Likely layer | First move |
|---|---|---|
| Fails identically on every target | commonMain logic | Write a failing commonTest; debug on JVM (`:shared:jvmTest`) |
| Fails on exactly one target | Platform `actual`, dispatcher difference, or platform rendering | Run the same commonTest per target (Step 4 playbook) |
| Crash with native frames in Xcode | Kotlin/Native or Swift interop boundary | Get the symbolicated crash report — Kotlin frames appear in the stack; see `platform-interop` |
| Logic tests green, UI wrong on one target | Compose rendering or entry-point wiring | `runComposeUiTest` in commonTest, then that platform's UI inspector |
| Compiles everywhere except one target | Source-set or dependency configuration | Check that target's dependencies and `actual` declarations in `shared/build.gradle.kts` |

### Step 3: Localize with the Platform Toolbox

Kermit in commonMain is the shared breadcrumb layer — one log call is visible in Logcat, the Xcode console, desktop stdout, and the browser console alike:

```kotlin
// commonMain
private val log = Logger.withTag("TaskRepository")

suspend fun sync() {
    log.d { "sync start, pending=${pending.size}" }
    // ...
    log.e(throwable) { "sync failed" }
}
```

| Platform | Tools | Notes |
|---|---|---|
| Android | Logcat (`adb logcat -s TaskRepository`), Android Studio debugger, Layout Inspector (recomposition counts), `adb` | Kermit output lands in Logcat under your tag |
| iOS | Xcode console, Instruments; Kotlin/Native crash reports are symbolicated — Kotlin function names show in the stack | Attach the Xcode/lldb debugger to Kotlin code via the KMP IDE plugin |
| Desktop | IDE JVM debugger: breakpoints, watches, expression evaluation | Iterate on candidate fixes with Compose Hot Reload (`desktopApp [hot]`) |
| Web | Browser devtools console and network tab | Wasm stack traces have limited fidelity — reproduce on JVM first whenever the code is shared |

Common KMP crash signatures:

| Signature | Likely cause |
|---|---|
| `NullPointerException` at the interop boundary | Swift passed nil into a non-null Kotlin parameter |
| Native frames / `EXC_BAD_ACCESS` in Xcode | Kotlin/Native-side crash — symbolicate before guessing |
| `IllegalStateException: Module with the Main dispatcher is missing` | Missing main-dispatcher artifact (e.g. `kotlinx-coroutines-swing` in `desktopApp`) |
| ViewModel instantiation crash on iOS/web only | Non-JVM targets need an explicit initializer: `viewModel { TaskListViewModel(...) }` |
| Works in debug, crashes in iOS release | Kotlin/Native release optimizations — reproduce with the Xcode Release scheme |
| Frozen UI on web | Blocking call on the single-threaded wasm main loop |

### Step 4: Bisect — the "Bug Only on One Target" Playbook

When one target misbehaves, the suspects in order:

1. **`actual` implementations** — date/number formatting, time zones, file paths, randomness.
2. **Dispatcher and threading differences** — `Dispatchers.IO` does not exist on wasmJs; iOS enforces main-thread UI rules.
3. **Platform rendering or entry-point wiring** — `MainViewController`, `ComposeViewport`, `MainActivity`.

Write ONE commonTest and run it per target to bisect:

```kotlin
// commonTest — identical assertions everywhere; per-target results bisect the actual
class DateFormatterTest {
    @Test
    fun formatsIsoDate() {
        assertEquals("2026-07-02", formatIsoDate(epochMillis = 1_751_414_400_000))
    }
}
```

```bash
./gradlew :shared:jvmTest --tests "*DateFormatterTest*"   # green -> JVM actual OK
./gradlew :shared:iosSimulatorArm64Test                   # red -> iosMain actual is the culprit
./gradlew :shared:wasmJsBrowserTest
```

JVM green plus iOS red means the bug is in the `iosMain` actual (or Kotlin/Native-specific behavior), not in shared logic.

For regressions, bisect history with the cheapest failing check:

```bash
git bisect start
git bisect bad
git bisect good v1.4.0
./gradlew :shared:jvmTest        # or :shared:iosSimulatorArm64Test for an iOS-only bug
git bisect good                  # or bad; repeat until the culprit commit is found
```

For shared-UI bugs, reduce inside a UI test instead of the full app:

```kotlin
@OptIn(ExperimentalTestApi::class)
@Test
fun taskRowShowsOverdueBadge() = runComposeUiTest {
    setContent { TaskRow(task = overdueTask, onToggle = {}) }
    onNodeWithTag("overdue_badge").assertExists()
}
```

### Step 5: Fix the Root Cause

```kotlin
// SYMPTOM FIX (bad): swallow and move on
try {
    repository.syncTasks()
} catch (e: Exception) {
    // silence — the bug is now invisible on every target
}

// ROOT CAUSE FIX (good): handle the specific failure and surface state
try {
    repository.syncTasks()
} catch (e: IOException) {   // kotlinx.io — multiplatform
    _uiState.update { it.copy(error = SyncError.Network) }
}
```

Common KMP fixes:

| Bug | Fix |
|---|---|
| Race on shared mutable state | `MutableStateFlow.update { }` or `Mutex` |
| Blocking work on the main thread (jank, ANR, frozen wasm) | `withContext(Dispatchers.IO)` on JVM/Android/native; on wasmJs keep the call suspending and non-blocking — `Dispatchers.IO` does not exist there |
| Divergent behavior across `actual` implementations | Prefer interface in commonMain + DI-wired platform impls (easier to fake and test) over widening `expect`/`actual` |
| ViewModel crash on non-JVM targets | Provide the explicit initializer: `viewModel { TaskListViewModel(graph.taskRepository) }` |
| Recomposition loop | `derivedStateOf`, stable parameter types, keyed `items` — see `performance-optimization` |
| UI state lost on Android configuration change | Hoist state into the shared ViewModel, not the composable |

### Step 6: Guard and Verify

Write the regression test with the Prove-It pattern: it must fail before the fix and pass after — the same discipline as `test-driven-development`. Use kotlin.test with hand-written fakes in commonTest (no MockK — it is JVM-only):

```kotlin
class FakeTaskApi : TaskApi {
    var failWith: Throwable? = null
    override suspend fun getTasks(): List<TaskDto> {
        failWith?.let { throw it }
        return emptyList()
    }
}

class TaskListViewModelTest {
    @Test
    fun syncSurfacesNetworkErrorWithoutCrashing() = runTest {
        val api = FakeTaskApi().apply { failWith = IOException("timeout") }
        val viewModel = TaskListViewModel(DefaultTaskRepository(api, FakeTaskDao()))

        viewModel.sync()

        assertIs<TaskListUiState.Error>(viewModel.uiState.value)
    }
}
```

Run the guard on every target the bug touched, then everywhere:

```bash
./gradlew :shared:allTests
```

Close the loop by re-running the original reproduction steps on the originally failing target — see `e2e-verification`.

## Common Rationalizations

| Shortcut | Why It Fails |
|---|---|
| "Just add a try-catch and move on" | You hid the bug on every target at once. It resurfaces in a worse form. |
| "It only breaks on iOS — patch it in Swift" | The root cause is almost always in the Kotlin `actual` or shared logic. A Swift patch forks behavior across platforms permanently. |
| "Debug it on the web target where it crashed" | Wasm stack traces are low-fidelity. If the code is shared, reproduce it as a commonTest on the JVM and debug there. |
| "It's a flaky test, just re-run it" | Flakiness has a root cause: dispatcher timing, shared state, or a real race. `runTest`'s virtual time usually exposes it. |
| "A clean build fixed it" | If you cannot explain why, it is not fixed. Caching bugs exist, but so do races that hide behind rebuild timing. |
| "I'll investigate later" | Errors compound. A bug in step 3 makes steps 4-10 wrong. Stop the line. |

## Red Flags

- Generic `catch (e: Exception)` swallowing errors in shared code
- `delay()` or `Thread.sleep` added to "fix" timing issues
- Bug fix merged without a regression test
- Platform-side workaround (Swift/JS) papering over a shared-logic defect
- `@Ignore` added to a failing test instead of a fix
- Fix verified on one target when several were failing
- No record of which targets reproduced the bug
- Fix bundles unrelated changes ("while I'm here...")

## Verification

- [ ] Bug reproduced reliably; failing targets recorded
- [ ] Layer identified: commonMain logic, platform `actual`, or platform rendering
- [ ] Root cause explained — not just the symptom silenced
- [ ] Regression test written (commonTest when shared; platform test set when platform-only)
- [ ] Test observed red before the fix and green after
- [ ] Original reproduction steps re-run on the originally failing target
- [ ] No unrelated changes in the fix commit
- [ ] `./gradlew :shared:allTests` passes
