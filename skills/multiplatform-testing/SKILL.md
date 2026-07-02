---
name: multiplatform-testing
description: >-
  Use when deciding which KMP test source set a test belongs in, writing shared
  Compose UI tests with runComposeUiTest, or running and debugging tests per
  target (JVM, Android, iOS simulator, wasm). Maps the full multiplatform test
  matrix, its Gradle tasks, and the platform-native harnesses around it.
---

# Multiplatform Testing

## Overview

One KMP module, six test source sets, five execution environments. This skill maps where each kind of test belongs, how to write shared Compose UI tests that run on every target from a single `commonTest` file, and which Gradle task executes what. The guiding rule: write once in `commonTest`, execute fast on the JVM, and reserve per-platform source sets for code that genuinely lives in a platform source set.

## When to Use

- Deciding where a new test file belongs in the `shared` module
- Writing UI tests for shared Compose Multiplatform screens
- Running or debugging tests on a specific target (iOS simulator, wasm browser, Android device)
- Setting up the per-target test matrix locally or in CI
- Testing `expect`/`actual` platform implementations

**Skip when:** Driving the Red-Green-Refactor loop for pure logic — `test-driven-development` owns the cycle; this skill owns the matrix. Full user journeys on installed apps belong to `e2e-verification`.

## Core Process

### Step 1: Map the Test Source Sets

| Source set | Executes on | What belongs here |
|---|---|---|
| `commonTest` | Every enabled target | Logic tests (kotlin.test + fakes + Turbine) and shared Compose UI tests (`runComposeUiTest`). Default home for all tests. |
| `jvmTest` | Desktop JVM, no device | JVM-only test setup (Room in-memory DAO tests, Ktor Java engine); also the fastest executor of everything inherited from `commonTest` |
| `androidUnitTest` | Local JVM with Android stubs | Tests of `androidMain` actuals that need no device |
| `androidInstrumentedTest` | Android device/emulator | Tests of `androidMain` actuals needing the real framework (Context, content resolvers, permissions) |
| `iosSimulatorArm64Test` | iOS simulator (Kotlin/Native) | Tests of `iosMain` actuals (NSUserDefaults, Keychain, Darwin APIs) |
| `wasmJsTest` | Headless browser | Tests of `wasmJsMain` actuals (browser APIs, JS interop) |

Placement rule: a test goes in `commonTest` unless the code under test lives in a platform source set. A logic test placed in `androidUnitTest` silently loses iOS, desktop, and web coverage.

### Step 2: Wire Shared UI Test Dependencies

Compose Multiplatform ships a common UI-test API — no JUnit rule, no test manifest, no device required:

```kotlin
// shared/build.gradle.kts
kotlin {
    sourceSets {
        commonTest.dependencies {
            implementation(kotlin("test"))
            @OptIn(ExperimentalComposeLibrary::class)
            implementation(compose.uiTest)            // org.jetbrains.compose.ui:ui-test
        }
        jvmTest.dependencies {
            implementation(compose.desktop.currentOs) // UI tests execute on desktop, fast
        }
    }
}
```

With `compose.desktop.currentOs` in `jvmTest`, every shared UI test runs headlessly on the desktop JVM in seconds — that is the default feedback loop. The same tests also run on the iOS simulator and in the browser when you execute those targets.

### Step 3: Write Shared Compose UI Tests in commonTest

```kotlin
// shared/src/commonTest/kotlin/com/example/tasks/AppUiTest.kt
@OptIn(ExperimentalTestApi::class)
class AppUiTest {

    @Test
    fun createTaskShowsItInTheList() = runComposeUiTest {
        setContent { App() }

        onNodeWithTag("add_task_fab").performClick()
        onNodeWithTag("task_title_input").performTextInput("Buy groceries")
        onNodeWithTag("save_button").performClick()

        onNodeWithText("Buy groceries").assertExists()
    }

    @Test
    fun emptyStateVisibleOnFirstLaunch() = runComposeUiTest {
        setContent { App() }

        onNodeWithText("No tasks yet").assertIsDisplayed()
    }

    @Test
    fun syncCompletesAsynchronously() = runComposeUiTest {
        setContent { App() }

        onNodeWithTag("sync_button").performClick()
        waitUntilExactlyOneExists(hasText("Synced"))   // never sleep
    }
}
```

The core vocabulary:

| Finders | Actions | Assertions |
|---|---|---|
| `onNodeWithText` | `performClick` | `assertExists` |
| `onNodeWithTag` | `performTextInput` | `assertIsDisplayed` |
| `onNodeWithContentDescription` | `performScrollTo` | `assertTextEquals` |
| `onAllNodesWithTag` | `performTouchInput` | `assertIsEnabled` / `assertIsNotEnabled` |

Selector preference: user-visible text first, content description for icon-only elements, `testTag` for structural nodes with dynamic text. Avoid index-based selection and parent traversal.

`testTag` discipline:

```kotlin
// commonMain — single source of truth for tags
object TestTags {
    const val ADD_TASK_FAB = "add_task_fab"
    const val SAVE_BUTTON = "save_button"
}

FloatingActionButton(
    onClick = onAddTask,
    modifier = Modifier.testTag(TestTags.ADD_TASK_FAB),
) { Icon(Icons.Default.Add, contentDescription = "Add task") }
```

- Tags are constants in `commonMain`, never derived from dynamic data
- Tags are stable API: renaming copy must not break tests; renaming tags is a refactor
- Composable-level patterns and semantics: see `compose-multiplatform-ui`

### Step 4: Run Tests Per Target — Gradle Cheat Sheet

```bash
./gradlew :shared:jvmTest                  # fast default: logic + UI tests on desktop JVM
./gradlew :shared:testDebugUnitTest        # androidUnitTest on the local JVM
./gradlew :shared:connectedAndroidTest     # androidInstrumentedTest — device/emulator required
./gradlew :shared:iosSimulatorArm64Test    # Kotlin/Native tests — requires macOS + Xcode
./gradlew :shared:wasmJsBrowserTest        # wasmJsTest in a headless browser
./gradlew :shared:allTests                 # every enabled target — pre-merge / CI gate

# Filter to one class on any target
./gradlew :shared:jvmTest --tests "com.example.tasks.AppUiTest"
```

Practical loop: `:shared:jvmTest` on every change; `:shared:allTests` before pushing. In CI, Linux runners cover JVM, Android unit, and wasm; a macOS runner covers `iosSimulatorArm64Test` (see `ci-cd-and-automation`).

### Step 5: Platform-Native E2E Harnesses (Brief)

Shared UI tests run `App()` in-process. Two thin native harnesses verify the real installed apps:

- **Android**: instrumented tests in `androidApp/src/androidTest` launch the real `MainActivity` — reserve for platform integration (intents, notifications, process death), not screen logic.
- **iOS**: an XCUITest target in the `iosApp` Xcode project drives the real app:

```swift
// iosApp/iosAppUITests/SmokeTest.swift
func testAppLaunchesAndShowsTaskList() {
    let app = XCUIApplication()
    app.launch()
    XCTAssert(app.staticTexts["No tasks yet"].waitForExistence(timeout: 5))
}
```

Assert on user-visible text — it is identical across platforms because the copy comes from shared `composeResources`. For full cross-platform user journeys, prefer one Maestro flow over two native suites — see `e2e-verification`.

### Step 6: adb / simctl Corner — Platform Debugging Only

These commands debug the platform app around the shared code (state, logs, screenshots). They are not a test strategy:

```bash
# Android
adb devices
adb install -r androidApp/build/outputs/apk/debug/androidApp-debug.apk
adb shell pm clear com.example.tasks                 # reset app state
adb logcat -s Kermit:D                               # shared-code logs (Kermit tag)
adb exec-out screencap -p > android-screen.png

# iOS simulator
xcrun simctl list devices available
xcrun simctl boot "iPhone 16"
xcrun simctl launch booted com.example.tasks
xcrun simctl io booted screenshot ios-screen.png
xcrun simctl spawn booted log stream --predicate 'processImagePath CONTAINS "iosApp"'
```

If a bug only reproduces here and not in `commonTest`, suspect an `actual` implementation or entry-point wiring — see `debugging-and-error-recovery`.

## Common Rationalizations

| Shortcut | Why It Fails |
|----------|-------------|
| "UI tests need an emulator" | `runComposeUiTest` in commonTest executes on the desktop JVM in seconds. The emulator is for `androidMain` actuals and E2E, not for screen logic. |
| "jvmTest passed, so it works everywhere" | The JVM is one of five environments. Kotlin/Native and wasm differ in concurrency, time, and actuals — `:shared:allTests` is the real gate. |
| "I'll put the test in androidUnitTest, it's where tests go" | That silently drops iOS, desktop, and web coverage. commonTest is the default; platform source sets are the exception. |
| "testTags clutter production code" | Tags are inert metadata centralized in one object. They decouple tests from copy changes — cheaper than rewriting selectors every sprint. |
| "Compose tests are flaky" | Flakiness is timing hacks. `waitUntilExactlyOneExists` and semantic assertions make them deterministic. |

## Red Flags

- Logic tests sitting in `androidUnitTest` or `jvmTest` that would compile in `commonTest`
- UI coverage exists only as Android instrumented tests
- Sleep-based waits instead of `waitUntil*` APIs
- `testTag` strings duplicated inline across screens and tests
- CI runs `:shared:jvmTest` only — no `allTests`, no iOS job
- Tests asserting platform-specific strings instead of shared `composeResources` copy
- Backtick test names with spaces in `commonTest`

## Verification

- [ ] Every test lives in the source set matching the Step 1 table
- [ ] Shared screens have `runComposeUiTest` coverage in `commonTest`
- [ ] `compose.uiTest` in commonTest and `compose.desktop.currentOs` in jvmTest are wired
- [ ] Tags come from a shared `TestTags` object; selectors are semantic
- [ ] No sleeps — async waits use `waitUntil*`
- [ ] `./gradlew :shared:jvmTest` passes locally (fast loop)
- [ ] iOS target tested where a Mac is available: `./gradlew :shared:iosSimulatorArm64Test`
- [ ] Full matrix green: `./gradlew :shared:allTests`
