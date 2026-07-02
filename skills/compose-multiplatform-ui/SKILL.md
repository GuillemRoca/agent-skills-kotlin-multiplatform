---
name: compose-multiplatform-ui
description: >-
  Use when building shared UI with Compose Multiplatform: screens and
  components in commonMain, Material 3 theming, multiplatform resources,
  type-safe navigation, adaptive layouts across form factors, and previews
  with Compose Hot Reload. Covers component architecture, state hoisting,
  recomposition discipline, and thin per-platform entry points for Android,
  iOS, desktop, and web.
---

# Compose Multiplatform UI

## Overview

Build production-quality shared UI with Compose Multiplatform. One `App()` composable in `shared/src/commonMain` renders on Android, iOS, desktop, and web (js/wasm Beta); entry-point modules stay thin shells. This skill covers component architecture, state management, Material 3 theming, multiplatform resources, navigation, adaptive layouts, recomposition discipline, and the preview/Hot Reload iteration loop.

## When to Use

- Building new screens or UI components in `shared/src/commonMain`
- Porting Android-only Compose code into shared Compose Multiplatform code
- Setting up theming, resources, or navigation for a KMP app
- Adapting layouts for phones, tablets, desktop windows, and browsers
- Debugging recomposition or performance issues in shared UI

**Skip when:** Working on non-UI code (repositories, use cases, data layer — see `multiplatform-architecture`), or writing platform-native UI in `iosApp` SwiftUI that only hosts the Compose view controller.

## Core Process

### Step 1: One App() in commonMain, Thin Platform Entry Points

All UI lives in `shared/src/commonMain`. Each platform module contains only a shell that calls `App()`:

```kotlin
// shared/src/commonMain/kotlin/App.kt — the single UI root
@Composable
fun App() {
    AppTheme {
        AppNavigation()
    }
}
```

```kotlin
// androidApp/src/main/kotlin/MainActivity.kt
class MainActivity : ComponentActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        enableEdgeToEdge()
        setContent { App() }
    }
}
```

```kotlin
// shared/src/iosMain/kotlin/MainViewController.kt
// Consumed by SwiftUI in iosApp through UIViewControllerRepresentable.
fun MainViewController(): UIViewController = ComposeUIViewController { App() }
```

```kotlin
// desktopApp/src/main/kotlin/main.kt
fun main() = application {
    Window(onCloseRequest = ::exitApplication, title = "Tasks") {
        App()
    }
}
```

```kotlin
// webApp/src/webMain/kotlin/main.kt — paired with index.html
@OptIn(ExperimentalComposeUiApi::class)
fun main() = ComposeViewport { App() }
```

Entry-point rules:
- Entry points contain zero business logic and zero screens — only platform wiring (edge-to-edge, window configuration, DI graph creation).
- Never duplicate a screen per platform. When a platform needs a native view (maps, camera preview), reach it through the `UIKitView`/`AndroidView` escape hatches in platform source sets — see `platform-interop`.

### Step 2: Component Architecture

Write stateless composables and hoist state:

```kotlin
// Stateless — reusable, testable, previewable, identical on every target
@Composable
fun TaskItem(
    task: Task,
    onToggle: (String) -> Unit,
    onDelete: (String) -> Unit,
    modifier: Modifier = Modifier,
) {
    ListItem(
        headlineContent = { Text(task.title) },
        leadingContent = {
            Checkbox(
                checked = task.completed,
                onCheckedChange = { onToggle(task.id) },
            )
        },
        trailingContent = {
            IconButton(onClick = { onDelete(task.id) }) {
                Icon(
                    imageVector = Icons.Default.Delete,
                    contentDescription = stringResource(Res.string.delete_task),
                )
            }
        },
        modifier = modifier,
    )
}
```

Wire ViewModels in common code with `org.jetbrains.androidx.lifecycle:lifecycle-viewmodel-compose:2.10.0`. Non-JVM targets have no reflection-based factory, so always pass an explicit initializer:

```kotlin
// Stateful wrapper — owns the ViewModel, exposes callbacks
@Composable
fun TaskListScreen(
    taskRepository: TaskRepository,          // provided by the DI graph
    onNavigateToDetail: (String) -> Unit,
    modifier: Modifier = Modifier,
) {
    val viewModel = viewModel { TaskListViewModel(taskRepository) }
    val uiState by viewModel.uiState.collectAsStateWithLifecycle()

    TaskListContent(
        uiState = uiState,
        onToggle = viewModel::toggleTask,
        onDelete = viewModel::deleteTask,
        onNavigateToDetail = onNavigateToDetail,
        modifier = modifier,
    )
}
```

Component rules:
- **Single responsibility** — one composable does one thing
- **Accept `Modifier` parameter** — always last with default `Modifier`
- **Hoist state** — push state up, push events down
- **Stateless by default** — only use `remember` when necessary
- **Composition over configuration** — slots and lambdas over boolean flags
- Never use `hiltViewModel()` — Hilt does not exist in commonMain; use `viewModel { ... }` plus your DI graph (Metro or Koin)

### Step 3: State Management in Compose

Collect state lifecycle-aware and handle every state:

```kotlin
// Always collectAsStateWithLifecycle (not collectAsState) — available in
// commonMain via the JetBrains lifecycle artifact
val uiState by viewModel.uiState.collectAsStateWithLifecycle()
```

```kotlin
@Composable
fun TaskListContent(
    uiState: TaskListUiState,
    onToggle: (String) -> Unit,
    onDelete: (String) -> Unit,
    onNavigateToDetail: (String) -> Unit,
    modifier: Modifier = Modifier,
) {
    when (uiState) {
        is TaskListUiState.Loading -> LoadingIndicator(modifier)
        is TaskListUiState.Success -> {
            if (uiState.tasks.isEmpty()) {
                EmptyState(
                    message = stringResource(Res.string.no_tasks_yet),
                    modifier = modifier,
                )
            } else {
                TaskList(
                    tasks = uiState.tasks,
                    onToggle = onToggle,
                    onDelete = onDelete,
                    onNavigateToDetail = onNavigateToDetail,
                    modifier = modifier,
                )
            }
        }
        is TaskListUiState.Error -> ErrorState(
            message = uiState.message,
            onRetry = uiState.retry,
            modifier = modifier,
        )
    }
}
```

### Step 4: Material 3 Theming

Define the theme in commonMain with explicit light/dark schemes. No `Build.VERSION` checks and no dynamic color here — `android.*` APIs do not compile in common code:

```kotlin
// shared/src/commonMain/kotlin/theme/Theme.kt
private val LightColors = lightColorScheme(
    primary = Color(0xFF3B5BA5),
    onPrimary = Color(0xFFFFFFFF),
    surface = Color(0xFFFDFBF7),
    onSurface = Color(0xFF1B1B1F),
)

private val DarkColors = darkColorScheme(
    primary = Color(0xFFB0C6FF),
    onPrimary = Color(0xFF10295E),
    surface = Color(0xFF121316),
    onSurface = Color(0xFFE3E2E6),
)

@Composable
fun AppTheme(
    darkTheme: Boolean = isSystemInDarkTheme(),
    content: @Composable () -> Unit,
) {
    MaterialTheme(
        colorScheme = if (darkTheme) DarkColors else LightColors,
        typography = AppTypography, // built from Font(Res.font.inter_regular)
        content = content,
    )
}
```

Android dynamic color (Material You) is platform-specific: expose a color-scheme provider interface in commonMain and bind an androidMain implementation through DI when you want it. Everywhere else the explicit schemes apply.

Use Material tokens, not hardcoded values:

```kotlin
// GOOD: theme tokens adapt to dark mode on every target
Text(
    text = stringResource(Res.string.task_list_title),
    style = MaterialTheme.typography.headlineMedium,
    color = MaterialTheme.colorScheme.onSurface,
)

// BAD: hardcoded values break dark mode on four platforms at once
Text(text = "Title", fontSize = 24.sp, color = Color(0xFF000000))
```

### Step 5: Resources with compose.components.resources

Shared resources live under `composeResources`, not Android `res/`. There is no `R` class and no Android XML in shared UI — the Gradle plugin generates a type-safe `Res` accessor:

```
shared/src/commonMain/composeResources/
├── drawable/            logo.xml, ic_empty.png
├── drawable-dark/       logo.xml            (dark-theme qualifier)
├── font/                inter_regular.ttf
├── values/              strings.xml
└── values-es/           strings.xml         (locale qualifier)
```

```xml
<!-- values/strings.xml -->
<resources>
    <string name="task_list_title">My Tasks</string>
    <string name="delete_task">Delete task</string>
    <string name="no_tasks_yet">No tasks yet</string>
</resources>
```

```kotlin
Text(stringResource(Res.string.task_list_title))
Image(
    painter = painterResource(Res.drawable.logo),
    contentDescription = null, // decorative
)
val appFont = FontFamily(Font(Res.font.inter_regular))
```

Resource rules:
- Qualifiers work like Android's: locale (`values-es`), theme (`-dark`), density (`-xxhdpi`)
- Every user-visible string goes through `Res.string` from day one — hardcoded literals never get localized
- Raw files go in `composeResources/files/` and load via `Res.readBytes("files/...")`

### Step 6: Navigation

Use `org.jetbrains.androidx.navigation:navigation-compose:2.9.2` with type-safe `@Serializable` routes:

```kotlin
@Serializable data object TaskList
@Serializable data class TaskDetail(val taskId: String)

@Composable
fun AppNavigation(modifier: Modifier = Modifier) {
    val navController = rememberNavController()

    NavHost(
        navController = navController,
        startDestination = TaskList,
        modifier = modifier,
    ) {
        composable<TaskList> {
            TaskListScreen(
                taskRepository = LocalAppGraph.current.taskRepository,
                onNavigateToDetail = { id -> navController.navigate(TaskDetail(id)) },
            )
        }
        composable<TaskDetail> { backStackEntry ->
            val route: TaskDetail = backStackEntry.toRoute()
            TaskDetailScreen(
                taskId = route.taskId,
                onNavigateBack = { navController.popBackStack() },
            )
        }
    }
}
```

Navigation rules:
- Screens receive navigation callbacks (`onNavigateToDetail`), never the `NavController`
- Routes are `@Serializable` types, not strings — arguments are constructor parameters
- Navigation 3 has been available for CMP since 1.10 but is still rc and its UI artifact does not yet cover all targets — adopt it only when your target set is supported (see `deprecation-and-migration` for the switch)

### Step 7: Adaptive Layouts and Input Differences

The same `App()` runs in a phone window, a resizable desktop window, and a browser tab. Branch on window size class, never on platform name or orientation:

```kotlin
// org.jetbrains.compose.material3.adaptive:adaptive
@Composable
fun TaskListRoute(modifier: Modifier = Modifier) {
    val windowSizeClass = currentWindowAdaptiveInfo().windowSizeClass
    val useTwoPane =
        windowSizeClass.windowWidthSizeClass == WindowWidthSizeClass.EXPANDED

    if (useTwoPane) {
        TaskListDetailPane(modifier)   // list + detail side by side
    } else {
        TaskListSinglePane(modifier)   // navigate list -> detail
    }
}
```

Input rules:
- Desktop users resize windows continuously — test intermediate widths, not just presets; set a sane minimum via `Window(state = rememberWindowState(...))`
- Mouse and keyboard exist on desktop and web: provide hover feedback (Material components do this automatically), keyboard focus traversal, and shortcuts via `Modifier.onKeyEvent`
- Mouse users expect visible scrollbars — pair lazy lists with a scrollbar on desktop:

```kotlin
// jvmMain (desktop-only wrapper around a common list)
Box(modifier) {
    LazyColumn(state = listState) { /* items */ }
    VerticalScrollbar(
        adapter = rememberScrollbarAdapter(listState),
        modifier = Modifier.align(Alignment.CenterEnd).fillMaxHeight(),
    )
}
```

- Touch targets stay at 48dp minimum on all targets even when a mouse is present — see `multiplatform-accessibility`

### Step 8: Recomposition Discipline

```kotlin
// BAD: new lambda instance on every recomposition
items(tasks) { task ->
    TaskItem(task, onToggle = { viewModel.toggle(task.id) }, onDelete = {})
}

// GOOD: stable method references, keyed items
LazyColumn {
    items(tasks, key = { it.id }) { task ->
        TaskItem(
            task = task,
            onToggle = viewModel::toggleTask,
            onDelete = viewModel::deleteTask,
        )
    }
}

// Computed values that change less often than their inputs
val showFab by remember {
    derivedStateOf { listState.firstVisibleItemIndex == 0 }
}
```

- Use immutable data classes for UI state; `kotlinx.collections.immutable` `ImmutableList` for list parameters
- Annotate stable-by-contract classes with `@Stable`
- Profile before optimizing — see `performance-optimization` for measuring on each target

### Step 9: Previews and Compose Hot Reload

Use the unified preview annotation in commonMain — it renders in IntelliJ IDEA and Android Studio without touching any platform module:

```kotlin
import org.jetbrains.compose.ui.tooling.preview.Preview

@Preview
@Composable
private fun TaskItemPreview() {
    AppTheme {
        TaskItem(
            task = Task(id = "1", title = "Buy groceries", completed = false),
            onToggle = {},
            onDelete = {},
        )
    }
}

@Preview
@Composable
private fun TaskListDarkPreview() {
    AppTheme(darkTheme = true) {
        TaskListContent(
            uiState = TaskListUiState.Success(
                tasks = listOf(
                    Task("1", "Buy groceries", completed = false),
                    Task("2", "Walk the dog", completed = true),
                ),
            ),
            onToggle = {},
            onDelete = {},
            onNavigateToDetail = {},
        )
    }
}
```

The fast iteration loop:
1. `@Preview` in commonMain for static component states (light/dark, empty/error)
2. The `desktopApp [hot]` run configuration (Compose Hot Reload, stable, bundled) for interactive iteration — edit commonMain composables and watch the running desktop window update without restart
3. Lock results in with shared UI tests (`runComposeUiTest` in commonTest) — see `multiplatform-testing`

## Common Rationalizations

| Shortcut | Why It Fails |
|----------|-------------|
| "I'll add the modifier parameter later" | Every consumer on every platform needs it later. Do it now. |
| "State hoisting is overkill for this screen" | Unhoisted state kills previews, `runComposeUiTest`, and reuse across targets. |
| "I'll skip the empty/error states" | Users see a blank screen — on four platforms simultaneously. |
| "I'll hardcode the strings and localize later" | Literals scattered across composables never get extracted. `Res.string` costs one line now. |
| "This screen is Android-first, I'll build it in androidApp" | It duplicates theme, navigation, and ViewModel wiring; moving it to commonMain later is a rewrite. |
| "Recomposition optimization can wait" | Correct — but only if you measured. Profile first, then decide. |

## Red Flags

- Composable without a `Modifier` parameter
- `collectAsState` instead of `collectAsStateWithLifecycle`
- `android.*` imports or `R.` references in `commonMain`
- Screens or `App()` copies living in entry-point modules instead of `shared/src/commonMain`
- Hardcoded colors, sizes, or user-visible strings instead of theme tokens and `Res`
- String routes, or a `NavController` passed into screen composables
- Layout branching on platform name or orientation instead of window size class
- No `@Preview` functions in commonMain

## Verification

- [ ] All composables accept a trailing `Modifier` parameter with a default
- [ ] State hoisted — content composables are stateless and previewable
- [ ] All UI states handled (loading, success, empty, error)
- [ ] `grep -rn "android\." shared/src/commonMain` and `grep -rn "\bR\." shared/src/commonMain` return nothing
- [ ] Strings, drawables, and fonts load through the generated `Res` class
- [ ] Routes are `@Serializable` types; screens receive callbacks, not the controller
- [ ] Layout verified at compact and expanded widths by resizing the desktop window (`desktopApp [hot]`)
- [ ] Previews exist in commonMain for key components in light and dark
- [ ] `./gradlew :shared:allTests :androidApp:assembleDebug` succeeds
