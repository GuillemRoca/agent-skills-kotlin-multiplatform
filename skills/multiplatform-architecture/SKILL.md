---
name: multiplatform-architecture
description: >-
  Use when designing module structure, deciding what goes in commonMain versus
  platform source sets, setting up Metro dependency injection, or defining
  layer boundaries in a Kotlin Multiplatform + Compose Multiplatform project.
---

# Multiplatform Architecture

## Overview

Good architecture makes a Kotlin Multiplatform app testable, maintainable, and portable across Android, iOS, desktop, and web. This skill guides the structural decisions: the shared-module-plus-entry-points layout, source-set layering, layer separation (UI ← Domain ← Data) entirely in commonMain, ViewModels with StateFlow, and compile-time dependency injection with Metro.

## When to Use

- Starting a new KMP/CMP project or adding a target to an existing one
- Deciding whether code belongs in commonMain or a platform source set
- Setting up or refactoring dependency injection across targets
- Defining boundaries between UI, domain, and data layers
- Debugging state management that behaves differently per target

**Skip when:** Making a small change inside an already-architected feature. For the common/platform boundary itself (expect/actual mechanics, iOS framework integration), see `platform-interop`.

## Core Process

### Step 1: Module Structure

1. **One shared KMP module, thin per-platform entry points:**

```
project/
├── shared/          # KMP module: business logic + shared Compose UI (App() in commonMain)
├── androidApp/      # entry point: com.android.application, MainActivity + AndroidManifest.xml
├── desktopApp/      # entry point: org.jetbrains.kotlin.jvm, main.kt, Compose Hot Reload
├── webApp/          # entry point: KMP module with js { browser() } AND wasmJs { browser() },
│                    #   entry code in a webMain source set
├── iosApp/          # Xcode project (NOT a Gradle module)
└── gradle/libs.versions.toml
```

When some platforms keep native UI, split `shared` into `sharedLogic` (no Compose dependency) and `sharedUI` (`App()`, `composeResources/`, produces the iOS framework).

2. **Dependency rules:**
   - Entry points (`androidApp`, `desktopApp`, `webApp`, `iosApp`) depend on `shared`
   - `shared` **never** depends on an entry-point module
   - All `expect`/`actual` declarations live in `shared`, never in entry points
   - Entry points contain launch plumbing only: `MainActivity`, `main.kt`, `index.html`, the Xcode project

3. **`shared/build.gradle.kts` — the AGP 9+ plugin set:**

```kotlin
plugins {
    alias(libs.plugins.kotlinMultiplatform)
    // com.android.kotlin.multiplatform.library — the KMP-native Android plugin
    alias(libs.plugins.androidKotlinMultiplatformLibrary)
    alias(libs.plugins.composeMultiplatform)
    alias(libs.plugins.composeCompiler)
    id("dev.zacsweers.metro") version "1.3.0"
}

kotlin {
    androidLibrary {
        namespace = "com.example.shared"
        compileSdk = 36
        androidResources { enable = true }
    }
    listOf(iosArm64(), iosSimulatorArm64()).forEach { iosTarget ->
        iosTarget.binaries.framework {
            baseName = "shared"
            isStatic = true
        }
    }
    jvm()
    js { browser() }
    @OptIn(ExperimentalWasmDsl::class)
    wasmJs { browser() }
}
```

The classic `com.android.library` plugin with an `androidTarget {}` block is legacy and incompatible with KMP modules under AGP 9 — treat it as migration context only.

### Step 2: Source-Set Layering

4. **commonMain first.** Write every declaration in `commonMain` until the compiler forces a platform API. `shared` source sets: `commonMain`, `androidMain`, `iosMain`, `jvmMain`, `jsMain`, `wasmJsMain`, plus matching `*Test`.

5. **Decision table:**

| Code | Where it goes |
|------|---------------|
| Domain models, use cases, repository interfaces, ViewModels, Compose UI | `commonMain` |
| Platform SDK integration that needs fakes or multiple implementations | Interface in `commonMain` + platform impls wired via DI |
| Obvious per-platform one-liner (OS name, UUID, locale) | `expect fun` / `actual fun` in `shared` |
| Class that must inherit a platform type | `expect class` / `actual class` in `shared` |

6. **Prefer interface + DI over expect/actual** (JetBrains guidance: use standard language constructs first). Interfaces are trivial to fake in `commonTest` and allow multiple implementations:

```kotlin
// commonMain
interface Platform { val name: String }

// androidMain
class AndroidPlatform : Platform {
    override val name = "Android ${Build.VERSION.SDK_INT}"
}
```

Reserve `expect`/`actual` for one-liners where an interface adds nothing:

```kotlin
// commonMain
expect fun currentPlatform(): Platform

// androidMain
actual fun currentPlatform(): Platform = AndroidPlatform()
```

See `platform-interop` for the full decision framework and worked examples.

### Step 3: Layer Separation

7. **Three layers, all in commonMain, strict boundaries:**

| Layer | Contains | Depends On |
|-------|----------|-----------|
| **UI** | Composables, ViewModels, UiState | Domain |
| **Domain** | Use cases, domain models, repository interfaces | Nothing platform-specific |
| **Data** | Repository implementations, Ktor/Room data sources, mappers | Domain (interfaces) |

8. **Data flows one direction: UI ← Domain ← Data.** The domain layer is plain Kotlin:

```kotlin
// commonMain — domain layer, zero platform imports
interface TaskRepository {
    fun getTasks(): Flow<List<Task>>
    suspend fun addTask(task: Task)
}

data class Task(val id: String, val title: String, val completed: Boolean)

@Inject
class GetTasksUseCase(private val repository: TaskRepository) {
    operator fun invoke(): Flow<List<Task>> = repository.getTasks()
}
```

Persistence and networking implementations (Room KMP, Ktor client) also live in commonMain, with platform drivers injected — see `multiplatform-data-persistence`.

### Step 4: ViewModel and State

9. **MVVM with a sealed UiState.** `org.jetbrains.androidx.lifecycle:lifecycle-viewmodel-compose:2.10.0` works in commonMain:

```kotlin
sealed interface TaskListUiState {
    data object Loading : TaskListUiState
    data class Success(val tasks: List<Task>) : TaskListUiState
    data class Error(val message: String) : TaskListUiState
}

@Inject
class TaskListViewModel(getTasks: GetTasksUseCase) : ViewModel() {
    val uiState: StateFlow<TaskListUiState> = getTasks()
        .map<List<Task>, TaskListUiState> { TaskListUiState.Success(it) }
        .catch { emit(TaskListUiState.Error(it.message ?: "Unknown error")) }
        .stateIn(
            scope = viewModelScope,
            started = SharingStarted.WhileSubscribed(5_000),
            initialValue = TaskListUiState.Loading,
        )
}
```

10. **Non-JVM targets need an explicit initializer.** Reflection-based `viewModel()` creation only exists on Android/JVM; on iOS, js, and wasmJs pass the factory lambda:

```kotlin
@Composable
fun TaskListScreen(graph: AppGraph) {
    val viewModel = viewModel { TaskListViewModel(graph.getTasksUseCase) }
    val state by viewModel.uiState.collectAsStateWithLifecycle()
    // render state — see compose-multiplatform-ui
}
```

### Step 5: Dependency Injection with Metro

11. **Metro (`dev.zacsweers.metro`) is compile-time DI that works on all KMP targets.** Hilt does not work in commonMain — it is Android-only legacy. Koin is the runtime-resolution alternative if the team prefers it. Define the graph in `shared`:

```kotlin
// commonMain
@DependencyGraph(AppScope::class)
interface AppGraph {
    val getTasksUseCase: GetTasksUseCase
}

@ContributesBinding(AppScope::class)
@SingleIn(AppScope::class)
@Inject
class DefaultTaskRepository(
    private val api: TaskApi,
    private val dao: TaskDao,
) : TaskRepository
```

Platform inputs (such as Android `Context`) enter through `@Provides` members on a graph factory; test doubles enter through `createDynamicGraph<AppGraph>(overrides)` — see `multiplatform-testing`.

12. **Each platform entry point creates the graph exactly once:**

```kotlin
// androidApp — MainActivity.kt
class MainActivity : ComponentActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        val graph = createGraph<AppGraph>()
        setContent { App(graph) }
    }
}

// shared/iosMain — MainViewController.kt
fun MainViewController() = ComposeUIViewController { App(createGraph<AppGraph>()) }

// desktopApp — main.kt
fun main() = application {
    val graph = createGraph<AppGraph>()
    Window(onCloseRequest = ::exitApplication) { App(graph) }
}

// webApp/src/webMain — main.kt
fun main() {
    val graph = createGraph<AppGraph>()
    ComposeViewport { App(graph) }
}
```

### Step 6: Coroutines and Flow Across Targets

13. **Structured concurrency with per-target dispatcher facts:**
   - Use `viewModelScope` — never `GlobalScope`
   - `Dispatchers.Main` exists out of the box on Android and iOS; desktop requires `kotlinx-coroutines-swing` on the classpath; js/wasmJs are single-threaded, so `Main` and `Default` share the browser event loop
   - Expose `Flow` from repositories; `catch` in the ViewModel, not the repository
   - Move heavy work off `Main` with `withContext(Dispatchers.Default)`; `Dispatchers.IO` exists on JVM and Native but not on js/wasmJs

```kotlin
// desktopApp/build.gradle.kts — provides Dispatchers.Main for Compose desktop
dependencies {
    implementation(libs.kotlinx.coroutinesSwing)
}
```

## Common Rationalizations

| Shortcut | Why It Fails |
|----------|-------------|
| "This app is Android-first — keep logic in androidApp for now" | Code in an entry point is invisible to every other target. Moving it later means untangling Android imports from business logic; writing it in commonMain first costs nothing. |
| "expect/actual everything — that's the KMP way" | Every `actual` is a hard compile-time dependency you cannot fake from commonTest. JetBrains guidance: interfaces + DI first, expect/actual for one-liners. |
| "Keep Hilt, it already works" | Hilt generates Android-only code and cannot provide anything to commonMain. You end up running two DI systems with duplicated wiring. Metro covers all targets at compile time. |
| "I'll put the logic in the ViewModel" | Fat ViewModels are untestable and cannot be composed. Domain logic belongs in use cases that work — and test — identically on every target. |
| "GlobalScope is fine for fire-and-forget" | It outlives the screen on every platform and leaks work with no cancellation. Use `viewModelScope` or an injected application scope. |
| "Skip the explicit viewModel initializer — it ran fine on Android" | The reflection factory exists only on JVM targets. The same screen crashes at first composition on iOS and web. |

## Red Flags

- `android.*`, `platform.*`, or `java.*` imports inside commonMain
- `shared/build.gradle.kts` declares a dependency on an entry-point module
- `expect`/`actual` declarations in `androidApp`, `desktopApp`, or `webApp`
- `com.android.library` + `androidTarget {}` on the shared module under AGP 9
- ViewModel exposes `MutableStateFlow` publicly, or talks to a Ktor client/DAO directly
- `viewModel<Foo>()` with no initializer lambda in shared Compose code
- `GlobalScope.launch` anywhere in production code

## Verification

- [ ] Module graph: every entry point depends on `shared`; `shared` depends on no entry point
- [ ] `shared/build.gradle.kts` uses `com.android.kotlin.multiplatform.library` with an `androidLibrary {}` block
- [ ] Domain layer in commonMain has zero platform imports
- [ ] ViewModels expose `StateFlow` via `stateIn(SharingStarted.WhileSubscribed(5_000))`
- [ ] Shared composables create ViewModels with an explicit `viewModel { ... }` initializer
- [ ] Exactly one `createGraph<AppGraph>()` per platform entry point; no service locators
- [ ] `kotlinx-coroutines-swing` present in desktopApp dependencies
- [ ] `./gradlew :shared:allTests` passes on every configured target
