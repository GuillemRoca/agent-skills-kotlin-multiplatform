# Performance Checklist — Kotlin Multiplatform Reference

## Rendering & Compose Performance (All Targets)

Shared Compose UI means recomposition discipline pays off on every platform at once.

- [ ] **Stable types** for Compose parameters (data classes, `@Stable` annotation)
- [ ] **`ImmutableList`** from kotlinx.collections.immutable for list parameters
- [ ] **`key()` parameter** on `LazyColumn` / `LazyRow` items
- [ ] **`derivedStateOf`** for computed values to reduce recompositions
- [ ] **Method references** over inline lambdas in loops where possible
- [ ] **Recomposition counts** checked (Layout Inspector on Android; compiler reports on all targets)
- [ ] **No heavy computation** in composition (move to ViewModel/UseCase)

```kotlin
// commonMain — identical on Android, iOS, desktop, and web
@Composable
fun TaskList(tasks: ImmutableList<Task>) {
    LazyColumn {
        items(tasks, key = { it.id }) { task ->
            TaskItem(task = task)
        }
    }
}
```

## Compose Compiler (All Targets)

The Compose compiler runs on shared code — one fix benefits every platform.

- [ ] **Strong skipping** in effect (default in current Compose compiler) — unstable params compared by instance equality
- [ ] **Stability config** (`stabilityConfigurationFile`) for classes the compiler can't infer (external models, platform types)
- [ ] **Compiler metrics/reports** reviewed for hot screens (`metricsDestination`/`reportsDestination`) — no unexpectedly unstable/restartable-only composables
- [ ] **`@Immutable`/`@Stable`** annotations on UI model classes where inference fails

## Memory Management (All Targets)

- [ ] **Coroutine scopes** properly scoped (no GlobalScope) — leaked coroutines leak on every platform
- [ ] **Large images** loaded with size constraints via Coil 3:

```kotlin
// commonMain — Coil 3 is KMP-capable
AsyncImage(
    model = ImageRequest.Builder(LocalPlatformContext.current)
        .data(url)
        .size(Size(200, 200)) // Don't load full resolution
        .crossfade(true)
        .memoryCachePolicy(CachePolicy.ENABLED)
        .build(),
    contentDescription = "...",
)
```

- [ ] **Large datasets** paginated — never loaded into memory unbounded
- [ ] **Listeners/callbacks** unregistered in `onCleared()` or `DisposableEffect`
- [ ] **Kotlin/Native GC** — modern GC with concurrent marking (Kotlin 2.4); still avoid retaining large object graphs in long-lived singletons on iOS
- [ ] **Android memory pressure** (`androidApp`) — `onTrimMemory` handled; leaks checked with LeakCanary

## Network Efficiency (All Targets)

- [ ] **API responses** paginated (not unbounded)
- [ ] **Caching** — Ktor `HttpCache` plugin installed; server cache headers respected
- [ ] **Compression** enabled (gzip)
- [ ] **Batch requests** where possible (reduce connection count)
- [ ] **Image CDN** with resizing parameters (request device-appropriate sizes)
- [ ] **Offline-first** — read from Room KMP cache, sync in background

## Database Performance (All Targets)

- [ ] **No N+1 queries** — use `@Transaction` with `@Relation` or JOINs
- [ ] **Indices** on frequently queried columns:

```kotlin
@Entity(
    tableName = "tasks",
    indices = [
        Index(value = ["created_at"]),
        Index(value = ["completed", "created_at"]),
    ]
)
```

- [ ] **Queries return only needed columns** (no `SELECT *` for large tables)
- [ ] **Room queries** profiled (enable query logging in debug)
- [ ] **Transactions** for batch operations (`@Transaction` annotation)
- [ ] **Write-ahead logging** (WAL) enabled where the driver supports it (Room default on Android/JVM)

## Android (androidApp)

### Android Vitals Targets

| Metric | Good | Needs Work | Critical |
|--------|------|-----------|----------|
| Cold startup | < 500ms | 500ms–1s | > 1s |
| Warm startup | < 200ms | 200ms–500ms | > 500ms |
| Slow frame rate | < 5% | 5%–10% | > 10% |
| Frozen frames | < 1% | 1%–3% | > 3% |
| ANR rate | < 0.47% | 0.47%–1% | > 1% |
| Crash rate | < 1.09% | 1.09%–2% | > 2% |

### Startup

- [ ] **Baseline Profiles** generated and included in release builds
- [ ] **Non-critical initialization** deferred to after first frame:

```kotlin
// androidApp — defer analytics, pre-fetching, etc.
lifecycleScope.launch {
    lifecycle.repeatOnLifecycle(Lifecycle.State.STARTED) {
        deferredInit()
    }
}
```

- [ ] **Splash screen** using `SplashScreen` API (not custom Activity)
- [ ] **Content provider initialization** minimized (auto-init libraries)
- [ ] **Macrobenchmark** covering cold/warm startup in CI
- [ ] **Trace startup** with Perfetto

### App Size

- [ ] **R8** enabled with resource shrinking; **R8 full mode** evaluated — verify reflection/serialization paths
- [ ] **WebP** for raster images; **vector drawables** for icons
- [ ] **APK Analyzer** reviewed before release (`classes.dex`, `res/`, `lib/`)
- [ ] **App Bundle** (`./gradlew :androidApp:bundleRelease`) with ABI splits for native libraries
- [ ] **16 KB page-size support** for native libraries (required since Nov 2025 for apps targeting Android 15+; NDK r28+ / AGP 8.5.1+)

### Background Work

- [ ] **WorkManager** with constraints for deferrable work; foreground services only for user-visible tasks; no unnecessary wake locks

## iOS (iosApp + iosMain)

- [ ] **Framework linking** — `isStatic = true` for the shared framework; measure launch impact of dynamic alternatives
- [ ] **Cheap entry point** — `ComposeUIViewController { App() }` creation does no synchronous I/O or graph-wide eager initialization
- [ ] **Concurrent render thread** (on by default since CMP 1.11) not blocked by main-thread work
- [ ] **Instruments** — Time Profiler for CPU, Allocations for memory, Hitches for dropped frames
- [ ] **App size** reviewed in Xcode Organizer per release

## Desktop (desktopApp)

- [ ] **JVM startup** — lazy initialization in `main.kt`; avoid classpath bloat
- [ ] **Main dispatcher** backed by `kotlinx-coroutines-swing`
- [ ] **Packaged runtime** trimmed via `./gradlew :desktopApp:packageDistributionForCurrentOS`; installer size tracked per release
- [ ] **Compose Hot Reload** used for the dev loop only — never shipped configuration

## Web (webApp)

- [ ] **Bundle size** of `./gradlew :webApp:composeCompatibilityBrowserDistribution` output tracked per release
- [ ] **Initial load** — lightweight loading indicator in `index.html` while the wasm/js bundle downloads and instantiates
- [ ] **Both targets shipped** — wasmJs for runtime performance, js for browser reach (cross-compat distribution)
- [ ] **Static hosting compression** enabled (gzip/brotli)
- [ ] **Browser DevTools** performance and memory profiles on the deployed bundle

## Profiling Tools

| Tool | Use For | When |
|------|---------|------|
| **Layout Inspector** (Android Studio) | Recomposition counts, UI hierarchy | Compose optimization |
| **Compose compiler reports** | Stability/skippability of shared composables | All targets |
| **Android Studio CPU/Memory Profiler** | Method traces, heap dumps | Android jank, leaks |
| **LeakCanary** | Automatic leak detection | Android development builds |
| **Macrobenchmark** | Startup, scroll, animation timing | Android CI regression testing |
| **Baseline Profile Generator** | AOT compilation optimization | Android release builds |
| **Perfetto** | System-level traces | Android deep analysis |
| **Instruments** (Xcode) | Time Profiler, Allocations, Hitches | iOS profiling |
| **JVM profilers** (async-profiler, VisualVM) | Desktop CPU/heap | Desktop performance |
| **Browser DevTools** | Bundle load, JS/wasm CPU, memory | Web performance |
| **Firebase Performance** | Production monitoring | Android/iOS post-release |

## Anti-Patterns

| Anti-Pattern | Impact | Fix |
|-------------|--------|-----|
| Loading images at full resolution | High memory on every target | Coil with size constraints |
| N+1 database queries | Slow list loading | JOIN queries or `@Relation` |
| Main thread disk/network I/O | ANR (Android), hitches (iOS), frozen UI (desktop) | `withContext(Dispatchers.IO)` |
| Unbounded LazyColumn data | High memory, slow render | Pagination |
| GlobalScope.launch | Leaked coroutines on all targets | viewModelScope or structured scope |
| Missing Baseline Profiles | Slow Android cold start | Generate profiles in CI (`androidApp`) |
| Synchronous initialization in entry points | Slow startup everywhere | Defer in `Application.onCreate`, `MainViewController`, `main()`, `ComposeViewport` alike |
| Large JSON parsing on main thread | Jank on every target | Parse on a background dispatcher |
