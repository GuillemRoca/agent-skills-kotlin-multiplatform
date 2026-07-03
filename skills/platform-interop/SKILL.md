---
name: platform-interop
description: >-
  Use when crossing the common/platform boundary in a Kotlin Multiplatform
  project — choosing between expect/actual and interface + DI, integrating the
  shared framework into an iOS app, calling platform-native APIs from shared
  code, or reasoning about Swift/ObjC interop and Kotlin/Native memory.
---

# Platform Interop

## Overview

Every KMP feature eventually touches an API that exists on only one platform. Crossing that boundary carelessly — expect/actual sprawl, platform types leaking into common signatures, Xcode integration held together by copied build phases — works on the author's machine and breaks the next target you add. This skill picks the right boundary mechanism, integrates the shared framework into iOS correctly, and keeps platform code testable from commonTest.

## When to Use

- Deciding between expect/actual and interface + DI for a platform capability
- Integrating `shared` into the iosApp Xcode project (framework, SwiftPM, CocoaPods)
- Exposing Kotlin APIs to Swift, or calling Swift/ObjC/platform APIs from Kotlin
- Wrapping per-platform behavior (scheduling, notifications, sensors) behind a common interface
- Debugging Kotlin/Native memory or interop issues

**Skip when:** The code is pure common Kotlin with no platform API in sight — see `multiplatform-architecture` for layering. Platform-adaptive UI belongs to `compose-multiplatform-ui`.

## Core Process

### Step 1: Choose the Boundary Mechanism

1. **Decision table — default to interface + DI** (JetBrains guidance: prefer standard language constructs over expect/actual):

| Situation | Mechanism |
|-----------|-----------|
| Platform capability with real logic; tests need a fake | Interface in commonMain + platform impls via DI |
| More than one implementation may exist on a platform | Interface + DI |
| Obvious per-platform one-liner (OS name, UUID) | `expect fun` / `actual fun` |
| Class must inherit a platform type | `expect class` / `actual class` |

2. **Interface + DI — the default.** Each target compiles commonMain plus its own source set, so a `@ContributesBinding` in androidMain lands only in the Android graph:

```kotlin
// shared/commonMain
interface DeviceInfo { val model: String }

// shared/androidMain
@ContributesBinding(AppScope::class)
@Inject
class AndroidDeviceInfo : DeviceInfo {
    override val model = "${Build.MANUFACTURER} ${Build.MODEL}"
}

// shared/iosMain
@ContributesBinding(AppScope::class)
@Inject
class IosDeviceInfo : DeviceInfo {
    override val model = UIDevice.currentDevice.model
}
```

Common code injects `DeviceInfo` and fakes it in commonTest with a three-line class — no mocking framework needed (see `multiplatform-testing`).

3. **expect/actual — for one-liners** where an interface adds nothing:

```kotlin
// commonMain
interface Platform { val name: String }
expect fun currentPlatform(): Platform

// androidMain
actual fun currentPlatform(): Platform = object : Platform {
    override val name = "Android ${Build.VERSION.SDK_INT}"
}
```

Every `expect` must have an `actual` in every target the module compiles for — adding a target means implementing all of them before anything builds. Keep the expect surface small.

### Step 2: Integrate the Shared Framework into iOS

4. **Three routes — pick by dependency needs, not tutorial age:**

| Route | When to use |
|-------|-------------|
| Direct integration: Xcode build phase runs `./gradlew :shared:embedAndSignAppleFrameworkForXcode` | Default — what the KMP IDE plugin generates |
| SwiftPM local package wrapping the framework | Modern alternative for teams standardized on SwiftPM |
| CocoaPods Gradle plugin | Only when consuming Pod dependencies from Kotlin |

5. **Swift imports the framework by its `baseName`, and Compose crosses the boundary in exactly one place:**

```kotlin
// shared/iosMain — MainViewController.kt
fun MainViewController(): UIViewController = ComposeUIViewController { App() }
```

```swift
// iosApp — ContentView.swift
import SwiftUI
import shared   // matches binaries.framework { baseName = "shared" }

struct ComposeView: UIViewControllerRepresentable {
    func makeUIViewController(context: Context) -> UIViewController {
        MainViewControllerKt.MainViewController()
    }
    func updateUIViewController(_ uiViewController: UIViewController, context: Context) {}
}
```

### Step 3: Swift/ObjC Interop Essentials

6. **Know what your Kotlin looks like from Swift** (via the generated ObjC header):
   - Classes map to classes, interfaces to protocols; top-level functions land on `<FileName>Kt`
   - Sealed hierarchies lose exhaustiveness — Swift `switch` needs a `default` branch
   - `List`/`Map` bridge to `Array`/`Dictionary`; nullable primitives box (`Int?` → `KotlinInt?`)
   - Suspend functions surface as completion handlers **and** as Swift `async`; annotate with `@Throws` or Kotlin exceptions crash the app instead of throwing

```kotlin
@ObjCName("TaskService")
class TaskFacade(private val getTasks: GetTasksUseCase) {
    @Throws(CancellationException::class)
    suspend fun loadTasks(): List<Task> = getTasks().first()
}
```

```swift
let tasks = try await TaskService(getTasks: graph.getTasksUseCase).loadTasks()
```

7. **Kotlin 2.4 directions:**
   - **Swift export (Beta)** exposes Kotlin to Swift without the ObjC intermediate — enums, overloads, and package namespaces come through cleaner; evaluate it for new API surfaces
   - Kotlin/Native can **consume Swift packages directly** — prefer that over hand-written cinterop `.def` files for Swift-only SDKs

### Step 4: Respect the Kotlin/Native Memory Model

8. **Modern GC, no freezing:**
   - Kotlin 2.4 Kotlin/Native uses a tracing GC with concurrent marking by default; objects move freely between threads. The freezing/`InvalidMutabilityException` era is legacy — delete `freeze()` calls and thread-confinement workarounds on sight.
   - One caution survives: **reference cycles across the Kotlin/ObjC boundary are collected by neither runtime.** A Kotlin object retaining a `UIViewController` that retains the Kotlin object leaks — break the cycle with a Swift `weak` reference or an explicit `dispose()`.

### Step 5: Worked Example — Background Scheduling Behind a Common Interface

9. **Background work is the canonical interop problem:** four platforms, four schedulers, one common API. Define the interface in commonMain; common code never sees a platform type:

```kotlin
// shared/commonMain
interface SyncScheduler {
    fun schedulePeriodicSync(interval: Duration)
    fun cancel()
}
```

10. **androidMain — WorkManager.** Android-only APIs are fine here; they must never appear in commonMain:

```kotlin
@ContributesBinding(AppScope::class)
@SingleIn(AppScope::class)
@Inject
class AndroidSyncScheduler(private val context: Context) : SyncScheduler {
    override fun schedulePeriodicSync(interval: Duration) {
        val request = PeriodicWorkRequestBuilder<SyncWorker>(interval.toJavaDuration())
            .setConstraints(
                Constraints.Builder().setRequiredNetworkType(NetworkType.CONNECTED).build()
            )
            .build()
        WorkManager.getInstance(context).enqueueUniquePeriodicWork(
            "sync", ExistingPeriodicWorkPolicy.KEEP, request,
        )
    }

    override fun cancel() {
        WorkManager.getInstance(context).cancelUniqueWork("sync")
    }
}
```

11. **iosMain — BGTaskScheduler** via the bundled `platform.BackgroundTasks` bindings:

```kotlin
import platform.BackgroundTasks.BGAppRefreshTaskRequest
import platform.BackgroundTasks.BGTaskScheduler
import platform.Foundation.NSDate
import platform.Foundation.dateWithTimeIntervalSinceNow

@ContributesBinding(AppScope::class)
@SingleIn(AppScope::class)
@Inject
class IosSyncScheduler : SyncScheduler {
    override fun schedulePeriodicSync(interval: Duration) {
        val request = BGAppRefreshTaskRequest(identifier = TASK_ID).apply {
            earliestBeginDate =
                NSDate.dateWithTimeIntervalSinceNow(interval.toDouble(DurationUnit.SECONDS))
        }
        BGTaskScheduler.sharedScheduler.submitTaskRequest(request, error = null)
    }

    override fun cancel() {
        BGTaskScheduler.sharedScheduler.cancelTaskRequestWithIdentifier(TASK_ID)
    }

    private companion object { const val TASK_ID = "com.example.sync" }
}
```

Register the handler at app launch (`BGTaskScheduler.sharedScheduler.registerForTaskWithIdentifier`), declare `TASK_ID` under `BGTaskSchedulerPermittedIdentifiers` in Info.plist, and re-submit from the handler — iOS treats every schedule as advisory.

12. **jvmMain (desktop) — coroutine ticker** on an injected application-lifetime scope:

```kotlin
@ContributesBinding(AppScope::class)
@SingleIn(AppScope::class)
@Inject
class DesktopSyncScheduler(
    private val syncTasks: SyncTasksUseCase,   // commonMain use case
    private val appScope: CoroutineScope,      // @SingleIn(AppScope::class) provided scope
) : SyncScheduler {
    private var job: Job? = null

    override fun schedulePeriodicSync(interval: Duration) {
        job?.cancel()
        job = appScope.launch {
            while (isActive) {
                delay(interval)
                syncTasks()
            }
        }
    }

    override fun cancel() { job?.cancel() }
}
```

13. **wasmJsMain — document the constraint.** Browsers grant no background execution to an unloaded page; do not fake it:

```kotlin
@ContributesBinding(AppScope::class)
@Inject
class WasmSyncScheduler : SyncScheduler {
    // No background work exists for a closed tab. Sync opportunistically while
    // the app is foregrounded; state must tolerate staleness on next launch.
    override fun schedulePeriodicSync(interval: Duration) { /* foreground refresh only */ }
    override fun cancel() {}
}
```

Each target's `createGraph<AppGraph>()` compiles commonMain plus exactly one platform source set, so exactly one binding resolves per platform. Common code depends only on `SyncScheduler` and fakes it in commonTest.

## Common Rationalizations

| Shortcut | Why It Fails |
|----------|-------------|
| "expect/actual is simpler than DI ceremony" | Every `actual` is a hard compile dependency that cannot be faked from commonTest, and adding a target forces implementing all actuals before anything builds. Interfaces stay testable and open. |
| "Use CocoaPods — all the tutorials do" | CocoaPods adds Ruby tooling and lockfile churn you only need when consuming Pods. Direct integration via `embedAndSignAppleFrameworkForXcode` is what the KMP IDE plugin generates. |
| "Call BGTaskScheduler from common code — cinterop makes it visible" | `platform.*` APIs exist only in iosMain; the module stops compiling for every other target, and platform types start leaking into shared signatures. |
| "The old blog says freeze() and DispatchQueue everything" | Freezing died with the legacy memory manager. Cargo-culted workarounds add complexity and mask real cross-boundary cycle leaks. |
| "iOS integration can wait until CI has a mac runner" | The framework boundary (baseName, static linking, `@Throws`) fails at Xcode build time. Discovering it weeks later is the expensive path — see `ci-cd-and-automation`. |

## Red Flags

- `platform.*`, `android.*`, or `java.*` imports inside commonMain
- `expect class` hierarchies carrying business logic that diverges per platform
- Swift `import` name differing from the framework `baseName`, or framework paths hardcoded in Xcode build phases
- WorkManager or BGTaskScheduler types appearing in a commonMain signature
- `freeze()`, `@SharedImmutable`, or thread-confinement workarounds in new code
- Kotlin singleton retaining a `UIViewController` with no dispose path
- Suspend function exposed to Swift without `@Throws`
- More `actual`s than interfaces — expect/actual as the default instead of the exception

## Verification

- [ ] Each platform capability is reachable from common code through an interface or a small `expect fun` — state which mechanism and why
- [ ] commonTest contains a fake for every platform interface (see `multiplatform-testing`)
- [ ] Exactly one `@ContributesBinding` implementation per target per interface; graph resolves on all targets
- [ ] Swift consumes the framework via `import shared`; the UI boundary is a single `ComposeUIViewController`
- [ ] Suspend functions exposed to Swift declare `@Throws`
- [ ] No `freeze()` or legacy memory-model workarounds in the diff
- [ ] iOS task identifiers registered at launch and declared in Info.plist
- [ ] `./gradlew :shared:allTests` passes on all configured targets
- [ ] `./gradlew :shared:embedAndSignAppleFrameworkForXcode` succeeds on macOS with Xcode
