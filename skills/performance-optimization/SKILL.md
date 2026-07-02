---
name: performance-optimization
description: >-
  Use when measuring or improving performance of a Kotlin Multiplatform +
  Compose Multiplatform app. Measure-first process: cross-platform
  recomposition and allocation wins in shared code, then scoped platform
  profiling — Macrobenchmark and Baseline Profiles on Android, Instruments
  on iOS, JVM profilers on desktop, bundle size and initial load on web.
---

# Performance Optimization

## Overview

Performance optimization without measurement is guessing. In a KMP + CMP app the biggest wins live in shared code — recomposition hygiene and allocation discipline pay on every target at once — while profiling is strictly per platform. Measure the release build, fix with targeted changes, verify the improvement with the same measurement, and guard against regressions.

## When to Use

- Startup, scrolling, or interaction feels slow on any target
- Android Vitals thresholds exceeded (cold start > 500ms, slow frames > 5%, ANR > 0.47%)
- App or bundle size over budget (APK, wasm binary, desktop installer)
- Before a release, as a performance regression check
- Users report slowness or battery drain

**Skip when:** no performance issue is observed or measured. Do not speculatively optimize — readability first (see `code-simplification`).

## Core Process

### Step 1: Measure the Release Build, Per Platform

The rule: profile release/optimized builds only. Debug builds disable R8 and Kotlin/Native optimizations and add instrumentation — their numbers are fiction.

| Platform | Optimized build to profile |
|---|---|
| Android | `./gradlew :androidApp:assembleRelease` (R8 + resource shrinking on) |
| iOS | Xcode Release scheme (Profile action or archive) |
| Desktop | `./gradlew :desktopApp:packageDistributionForCurrentOS` |
| Web | `./gradlew :webApp:composeCompatibilityBrowserDistribution` |

Record a baseline number before touching code — startup milliseconds, frame stats, or bytes. Every fix in later steps must be verified against this baseline.

### Step 2: Common Wins — Recomposition Hygiene (All Targets)

CMP recomposition behaves identically on Android, iOS, desktop, and web: a fix here pays on every target at once. Do this before any platform-specific work.

```kotlin
// Keys in lazy lists: identity survives reorders; avoids rebinding every row
LazyColumn {
    items(tasks, key = { it.id }) { task ->
        TaskRow(task = task, onToggle = onToggle)
    }
}

// derivedStateOf: recompose only when the derived value flips
val showScrollToTop by remember {
    derivedStateOf { listState.firstVisibleItemIndex > 5 }
}

// Stable parameters: an unstable List<T> forces recomposition on every parent pass
@Composable
fun TaskList(
    tasks: ImmutableList<Task>,          // kotlinx.collections.immutable
    onToggle: (String) -> Unit,
)

// Lambda stability: method references beat capturing lambdas in hot rows
TaskRow(taskId = task.id, onToggle = viewModel::toggle)
```

### Step 3: Common Wins — No Work in Composition

Composition runs on the UI path of every platform. Move computation into the ViewModel Flow chain so composition only renders:

```kotlin
// BAD: sorts on every recomposition, on every target
@Composable
fun TaskScreen(viewModel: TaskListViewModel) {
    val tasks by viewModel.tasks.collectAsStateWithLifecycle()
    val sorted = tasks.sortedBy { it.dueDate }   // work in composition
    TaskList(sorted.toImmutableList(), viewModel::toggle)
}

// GOOD: ViewModel exposes already-shaped state
class TaskListViewModel(repo: TaskRepository) : ViewModel() {
    val uiState: StateFlow<TaskListUiState> = repo.observeTasks()
        .map { tasks ->
            TaskListUiState.Loaded(tasks.sortedBy { it.dueDate }.toImmutableList())
        }
        .stateIn(viewModelScope, SharingStarted.WhileSubscribed(5_000), TaskListUiState.Loading)
}
```

Allocation discipline — no object churn in per-frame code:

```kotlin
// BAD: allocates a Stroke on every frame
Canvas(Modifier.fillMaxSize()) {
    drawCircle(color, style = Stroke(width = 4f))
}

// GOOD: allocate once in composition, reuse across frames
val stroke = remember { Stroke(width = 4f) }
Canvas(Modifier.fillMaxSize()) {
    drawCircle(color, style = stroke)
}
```

Images: request only the pixels you render (Coil 3, KMP):

```kotlin
AsyncImage(
    model = ImageRequest.Builder(LocalPlatformContext.current)
        .data(task.imageUrl)
        .size(200, 200)   // match display size; never full resolution for thumbnails
        .build(),
    contentDescription = task.title,
    modifier = Modifier.size(64.dp),
)
```

### Step 4: Android — Macrobenchmark, Baseline Profiles, Vitals (androidApp only)

Scoped to `androidApp`; these tools do not exist on other targets.

Android Vitals targets:

| Metric | Target | Critical |
|---|---|---|
| Cold startup | < 500ms | > 1s |
| Warm startup | < 200ms | > 500ms |
| Frame rendering (jank) | < 5% slow frames | > 10% slow frames |
| ANR rate | < 0.47% | > 1% |
| APK size (compressed) | < 10MB | > 50MB |
| Memory usage | < 150MB typical | > 256MB |

```kotlin
// benchmark module, instrumented against androidApp
@RunWith(AndroidJUnit4::class)
class StartupBenchmark {
    @get:Rule
    val rule = MacrobenchmarkRule()

    @Test
    fun coldStartup() {
        rule.measureRepeated(
            packageName = "com.example.androidApp",
            metrics = listOf(StartupTimingMetric()),
            startupMode = StartupMode.COLD,
            iterations = 5,
        ) {
            pressHome()
            startActivityAndWait()
        }
    }
}
```

```kotlin
@RunWith(AndroidJUnit4::class)
class BaselineProfileGenerator {
    @get:Rule
    val rule = BaselineProfileRule()

    @Test
    fun generate() {
        rule.collect(packageName = "com.example.androidApp") {
            pressHome()
            startActivityAndWait()
            device.findObject(By.text("Tasks")).click()
            device.waitForIdle()
        }
    }
}
```

```kotlin
// androidApp/build.gradle.kts — size and startup both depend on this
buildTypes {
    release {
        isMinifyEnabled = true
        isShrinkResources = true
    }
}
```

### Step 5: iOS — Startup and the Render Thread

- CMP 1.11 renders on a concurrent render thread by default: rendering overlaps UI work, freeing the main thread. Leave it on; if you suspect it in a regression, measure with it toggled rather than guessing.
- Profile with Instruments (Time Profiler for hot code, App Launch template for startup) against the Release scheme. Kotlin frames appear symbolicated, so shared-code hotspots are directly attributable.
- Kotlin/Native GC (Kotlin 2.4) marks concurrently; the practical discipline is the same as on the JVM — avoid allocation churn in per-frame code (Step 3).
- Framework size affects launch and download: export only the API `iosApp` needs from the `shared` framework.

### Step 6: Desktop — JVM Profilers

- Attach the IntelliJ profiler, Java Flight Recorder, or async-profiler. Profile the packaged distribution (`:desktopApp:packageDistributionForCurrentOS`), not the IDE run.
- Use Compose Hot Reload (`desktopApp [hot]`) to iterate on candidate fixes quickly, but only trust numbers from the packaged build.
- Startup on desktop is dominated by JVM start plus first composition; defer non-critical initialization off the first frame just as on mobile.

### Step 7: Web — Bundle Size and Initial Load

- The wasm binary dominates first load. Measure the production distribution output of `:webApp:composeCompatibilityBrowserDistribution`, never the dev server.

```bash
./gradlew :webApp:composeCompatibilityBrowserDistribution
# inspect the produced .wasm and .js sizes in webApp/build; compare
# transferred bytes in the browser Network tab with gzip/brotli enabled
```

- Browser devtools: Network tab for transferred bytes and cache headers, Performance tab for time-to-first-frame.
- The compatibility bundle ships both js and wasmJs; measure both paths, since browsers without wasmGC fall back to js.
- Load heavy `composeResources` (fonts, large images, files) lazily after the first frame.

### Step 8: Verify and Guard

- Re-run the exact measurement from Step 1 on the same release build; record before/after numbers next to the change.
- CI guards: Macrobenchmark on pre-release builds, APK and wasm size budgets as failing checks — see `ci-cd-and-automation`.
- Production: track startup and jank in the field — see `observability-and-instrumentation`.
- Re-test on low-end hardware (old Android device, oldest supported iPhone), not only the development machine.

## Common Rationalizations

| Shortcut | Why It Fails |
|---|---|
| "It's fast on my Pixel 8 / M3 Mac" | Flagship hardware is not your users' hardware. Test on low-end Android and older iPhones. |
| "It's fast on desktop, so the shared code is fine" | Targets differ: Kotlin/Native GC, single-threaded wasm, mobile thermal throttling. Measure every shipping target. |
| "We'll optimize later" | Performance debt compounds. Fixing later costs 10x more. |
| "The profiler shows it's fine" (on a debug build) | Debug builds disable R8 and Kotlin/Native optimizations and add instrumentation. Profile release builds only. |
| "Only 5% of users hit this" | 5% of 1M users is 50,000 people. Every percentage matters. |

## Red Flags

- Profiling done on debug builds
- No Baseline Profiles or Macrobenchmark in `androidApp`
- Lazy lists without `key`s; unstable `List<T>` parameters on hot composables
- Sorting, filtering, or formatting inside composable bodies
- Object allocation inside draw or pointer-input loops
- Wasm bundle size not tracked between releases
- Blocking work on the main thread (jank on mobile/desktop, frozen UI on wasm)
- "Optimizations" merged without before/after numbers

## Verification

- [ ] Bottleneck identified from a release-build profile, not a guess
- [ ] Baseline metric recorded before the change; after-number recorded with it
- [ ] Recomposition checked on the affected screen (Layout Inspector counts or composition traces)
- [ ] Lazy lists keyed; hot composable parameters stable or immutable
- [ ] Android: cold start < 500ms and slow frames < 5% via Macrobenchmark; Baseline Profile included
- [ ] iOS: Instruments Time Profiler run against the Release scheme
- [ ] Web: production bundle size measured and within budget
- [ ] CI guards in place (benchmark on pre-release builds, size budgets)
- [ ] `./gradlew :androidApp:assembleRelease :webApp:composeCompatibilityBrowserDistribution` succeeds
