---
name: deprecation-and-migration
description: >-
  Use when deprecating APIs, raising per-target platform floors (minSdk, iOS
  deployment target, browser wasm-GC floor), migrating libraries, or moving a
  Kotlin Multiplatform project to the AGP 9 shared + entry-module layout.
  Covers the strangler-fig pattern, deprecation windows, and incremental
  migration.
---

# Deprecation and Migration

## Overview

Code is a liability, not an asset — every line carries ongoing maintenance cost. Deprecation and migration manage that liability: remove what is no longer needed, upgrade what must evolve, and do it incrementally. In KMP each target has its own floor (minSdk, iOS deployment target, browser capability) and its own raise procedure, and the biggest migration many projects face is the AGP 9 move from the legacy single-`composeApp` layout to the shared + entry-module structure.

## When to Use

- Raising `minSdk`, the iOS deployment target, or the browser/wasm floor
- Migrating a library (Room, Ktor, Compose, Kotlin versions)
- Replacing an Android-only library with a KMP-capable one
- Restructuring to the recommended `shared` + entry-module layout (AGP 9)
- Removing legacy feature flags or dead code

**Skip when:** The code has no callers and can be deleted outright.

## Core Process

### Step 1: Decision Framework

1. **Assess before deprecating:**

| Question | Impact |
|----------|--------|
| How many callers exist? | grep/find usages across all source sets |
| Is a replacement ready? | Never deprecate without an alternative |
| Which targets are affected? | A floor bump may hit only one platform |
| What's the migration cost? | Test effort across targets, rollback risk |
| Is there a deadline? | Compulsory (AGP/store/security) vs advisory |

2. **Deprecation types:** *advisory* (recommended, no timeline) vs *compulsory* (hard deadline: AGP requirement, store policy, security fix).

### Step 2: Deprecate with Guidance

3. **Use Kotlin `@Deprecated` with a replacement — works in `commonMain`:**

```kotlin
@Deprecated(
    message = "Use TaskRepository.observeTasks() (Flow) instead",
    replaceWith = ReplaceWith("observeTasks()"),
    level = DeprecationLevel.WARNING  // WARNING -> ERROR -> HIDDEN
)
suspend fun getTaskList(): List<Task> = observeTasks().first()
```

4. **Progression:**

```
Phase 1: @Deprecated(WARNING) + replacement guidance
Phase 2: Migrate all internal callers (every source set)
Phase 3: Escalate to @Deprecated(ERROR)
Phase 4: Remove (or HIDDEN for published library binary compatibility)
```

### Step 3: Per-Target Platform Floors

5. **Each target has its own floor and raise procedure — audit before bumping:**

| Target | Floor | Declared in | Raise procedure |
|--------|-------|-------------|-----------------|
| Android | `minSdk` | `androidLibrary {}` (shared) + `androidApp` | lint for `NewApi`, drop `SDK_INT` guards, test on new min emulator |
| iOS | deployment target | Xcode project + framework build settings | audit `available` checks in `iosMain`/Swift, test on oldest supported simulator |
| Web | wasm-GC capability | `wasmJs` target / hosting | confirm target browsers support WasmGC; decide whether the `js` fallback can be dropped |

6. **Android minSdk bump:**

```bash
./gradlew lint 2>&1 | grep -i "NewApi\|ObsoleteSdkInt"
grep -rn "Build.VERSION.SDK_INT" shared/src/androidMain
```
Remove now-unnecessary `SDK_INT` branches, update `@RequiresApi`, test on the new minimum API emulator.

7. **iOS deployment-target bump:** search `iosMain` and Swift for `if #available` / `UIDevice` checks that the new floor makes dead; raise `IPHONEOS_DEPLOYMENT_TARGET` in the Xcode project and the framework settings together; run `./gradlew :shared:iosSimulatorArm64Test` on the oldest supported simulator.

8. **Browser/wasm-GC floor:** the `composeCompatibilityBrowserDistribution` bundle ships both js and wasmJs and selects at runtime. `wasmJs` requires WasmGC-capable browsers; the `js` target is the fallback for older ones. Raising the floor means dropping the `js` fallback — gate that decision on usage analytics, then remove the `js { browser() }` target and its `jsMain` code.

### Step 4: Migrate to the AGP 9 Shared + Entry-Module Layout

9. **Why:** AGP 9 makes the legacy single-`composeApp` module with `com.android.library` + `androidTarget {}` incompatible with KMP modules. Move to `shared` (logic + shared Compose UI) plus thin entry modules. Do it one boundary per commit (strangler-fig; see `incremental-implementation`).

10. **Swap the Android Gradle plugin in the shared module** — before and after:

```kotlin
// BEFORE (legacy composeApp/build.gradle.kts) — incompatible under AGP 9
plugins {
    id("com.android.library")
    kotlin("multiplatform")
}
kotlin {
    androidTarget()
    // iOS, jvm, js targets...
}
android {
    namespace = "com.example.app"
    compileSdk = 36
}
```

```kotlin
// AFTER (shared/build.gradle.kts) — AGP 9 KMP library plugin
plugins {
    kotlin("multiplatform")
    id("com.android.kotlin.multiplatform.library")
    id("org.jetbrains.compose")
    id("org.jetbrains.kotlin.plugin.compose")
}
kotlin {
    androidLibrary {
        namespace = "com.example.shared"
        compileSdk = 36
        androidResources { enable = true }
    }
    listOf(iosArm64(), iosSimulatorArm64()).forEach {
        it.binaries.framework { baseName = "shared"; isStatic = true }
    }
    jvm()
    js { browser() }
    @OptIn(ExperimentalWasmDsl::class) wasmJs { browser() }
}
```

11. **Move code — what goes where:**

| From (legacy) | To | Notes |
|---------------|-----|-------|
| `commonMain` logic + `App()` | `shared/commonMain` | shared Compose UI stays here |
| `androidMain` (except `actual`s) | `androidApp/src/main` | MainActivity + `AndroidManifest.xml` |
| `androidMain` expect/actual `actual`s | stay in `shared/androidMain` | all `actual`s live in `shared` |
| `jvmMain` / desktop `main.kt` | `desktopApp` | `org.jetbrains.kotlin.jvm`, `application {}` |
| `webMain` / js / wasm entry | `webApp` | js + wasmJs `browser()`, `index.html` |
| `iosMain` (`MainViewController.kt`) | stays in `shared/iosMain` | framework consumed by `iosApp` |

12. **Create the entry modules:**
    - `androidApp` (`com.android.application`): `MainActivity` calling `setContent { App() }`, plus the manifest.
    - `desktopApp` (`org.jetbrains.kotlin.jvm`): `main.kt` with `application { Window(onCloseRequest = ::exitApplication) { App() } }`.
    - `webApp` (KMP module, js + wasmJs `browser()`): `main.kt` in `src/webMain` with `ComposeViewport { App() }` + `index.html`.

13. **Update the Xcode build phase** — the framework now comes from `shared`, not `composeApp`:

```bash
# OLD Xcode "Run Script" build phase
./gradlew :composeApp:embedAndSignAppleFrameworkForXcode
# NEW
./gradlew :shared:embedAndSignAppleFrameworkForXcode
```
The framework `baseName` stays `shared`, so `import shared` in Swift is unchanged.

14. **Verify each step** as you go: `./gradlew :shared:allTests`, then run each platform's run configuration (`androidApp`, `iosApp`, `desktopApp [hot]`, `webApp[wasmJs]`). Migrate and commit one module boundary at a time.

### Step 5: Library Migration (Android-only -> KMP behind an interface)

15. **Strangler-fig, per feature.** Replace an Android-only library (e.g. Retrofit) with a KMP-capable one (Ktor) behind an interface in `commonMain`:

```kotlin
// Phase 1: interface in commonMain — the seam
interface TaskApi {
    suspend fun fetchTasks(): List<TaskDto>
}
```

```kotlin
// Phase 2: legacy Retrofit impl stays in androidMain during migration
class RetrofitTaskApi(private val service: TaskService) : TaskApi {
    override suspend fun fetchTasks() = service.getTasks()
}

// Phase 2: new Ktor impl in commonMain — works on every target
class KtorTaskApi(private val client: HttpClient) : TaskApi {
    override suspend fun fetchTasks(): List<TaskDto> =
        client.get("tasks").body()
}
```

```kotlin
// Phase 3: switch the DI binding per feature (Metro)
@ContributesBinding(AppScope::class)
@Inject
class KtorTaskApiBinding(client: HttpClient) : TaskApi by KtorTaskApi(client)
// Phase 4: delete RetrofitTaskApi, the Retrofit dependency, and the adapter
```

16. Migrate one feature at a time; keep both implementations behind the interface until the last caller is switched, then delete the legacy path.

### Step 6: Version Bumps and Cleanup

17. **Kotlin/Compose version bump** (they move together since Kotlin 2.0):

```markdown
- [ ] Bump `kotlin` and the Compose compiler plugin to the same version in libs.versions.toml
- [ ] ./gradlew build — fix K2 diagnostics
- [ ] ./gradlew :shared:allTests — verify all targets
- [ ] Review the Kotlin/Compose migration notes for breaking changes
```

18. **After migration:** remove deprecated code, delete migration feature flags, remove strangler adapters, update ADRs (see `documentation-and-adrs`), and confirm no references remain: `grep -rn "OldClassName" --include="*.kt"`.

## Common Rationalizations

| Shortcut | Why It Fails |
|----------|-------------|
| "Migrate the whole module layout in one commit" | Big-bang restructures are unreviewable and unrevertable. Move one boundary per commit. |
| "One floor covers all platforms" | minSdk, iOS deployment target, and the wasm-GC floor are independent — each needs its own audit and test. |
| "Keep using androidTarget {}, it still builds today" | AGP 9 makes `com.android.library` + `androidTarget {}` incompatible with KMP modules. It is compulsory, not advisory. |
| "Use Retrofit in commonMain, it's familiar" | Retrofit is JVM/Android-only. Put a `TaskApi` interface in commonMain and back it with Ktor. |
| "Delete the old library, no one calls it" | Check every source set first — Hyrum's Law. Deprecate through the interface, then remove. |

## Red Flags

- `@Deprecated` without `replaceWith` guidance
- A single floor bumped for "all platforms" without per-target audit
- `com.android.library` + `androidTarget {}` in a KMP module under AGP 9
- expect/actual `actual`s living in an entry module instead of `shared`
- Android-only libraries (Retrofit, Hilt) referenced from `commonMain`
- Big-bang module restructure in one commit
- Xcode build phase still pointing at `:composeApp:embedAndSign...`
- Migration feature flags older than a couple of sprints

## Verification

- [ ] Deprecated APIs carry `@Deprecated` with `replaceWith`; level progresses WARNING -> ERROR -> removal
- [ ] Each platform floor (minSdk, iOS deployment target, wasm-GC) raised via its own procedure and tested on its minimum
- [ ] Shared module uses `com.android.kotlin.multiplatform.library` + `androidLibrary {}` (no `androidTarget {}`)
- [ ] All expect/actual `actual`s live in `shared`; entry modules hold only platform entry points
- [ ] Xcode build phase runs `:shared:embedAndSignAppleFrameworkForXcode`
- [ ] Library migrations go through a `commonMain` interface, switched per feature, legacy path deleted last
- [ ] Migration feature flags cleaned up; no leftover references (`grep`)
- [ ] ADR written for significant migration decisions
- [ ] `./gradlew :shared:allTests` and each platform run configuration pass after migration
