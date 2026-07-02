# Testing Patterns — Kotlin Multiplatform Reference

## Toolchain Notes

- **kotlin.test is the common framework.** `commonTest` uses kotlin.test (`@Test`, `assertEquals`, `assertFailsWith`) plus `runTest` from kotlinx-coroutines-test. JUnit5 and MockK are JVM-only — scope them to `jvmTest` or `androidUnitTest`, never `commonTest`.
- **Ktor MockEngine is the common-code default for API tests.** MockWebServer only runs on the JVM; keep it in `jvmTest`/`androidUnitTest` for exercising the real OkHttp engine.
- **Shared UI tests use `runComposeUiTest`** (`org.jetbrains.compose.ui:ui-test`, `@OptIn(ExperimentalTestApi::class)`) in `commonTest` — no JUnit rule needed.
- **Turbine works in `commonTest`** for Flow assertions.
- **Backtick test names with spaces do not compile on every target.** Use plain camelCase names in `commonTest`; backtick names are fine in JVM-only test source sets.
- **Per-target test tasks:**

| Task | Runs | Use |
|------|------|-----|
| `./gradlew :shared:allTests` | every configured target | pre-merge gate |
| `./gradlew :shared:jvmTest` | JVM only | fastest inner loop |
| `./gradlew :shared:iosSimulatorArm64Test` | iOS simulator | needs macOS + Xcode |
| `./gradlew :shared:wasmJsBrowserTest` | wasm in a headless browser | web regressions |

## Core Concepts

### Arrange-Act-Assert (AAA)

```kotlin
@Test
fun repositoryReturnsCachedTasksWhenNetworkUnavailable() = runTest {
    // Arrange
    val dao = FakeTaskDao(initial = listOf(TaskEntity("1", "Cached task", false)))
    val api = FakeTaskApi().apply { failWith = TaskApiException("No network") }
    val repository = DefaultTaskRepository(dao, api, TaskMapper())

    // Act
    val result = repository.getTasks().first()

    // Assert
    assertEquals(1, result.size)
    assertEquals("Cached task", result[0].title)
}
```

### Naming Convention

```
[unit] [expected behavior] [condition]
```

```kotlin
viewModelEmitsLoadingInitially
repositorySyncsFromApiWhenNetworkAvailable
daoReturnsEmptyListWhenNoTasksExist
mapperConvertsEntityToDomainCorrectly
```

## Common Assertions

```kotlin
// Equality
assertEquals(expected, actual)
assertNotEquals(unexpected, actual)

// Truthiness
assertTrue(condition)
assertFalse(condition)
assertNull(value)
assertNotNull(value)

// Type checking
assertIs<TaskListUiState.Success>(state)
assertIsNot<TaskListUiState.Loading>(state)

// Collections
assertEquals(3, list.size)
assertTrue(list.isEmpty())
assertTrue(list.contains(item))

// Exceptions (kotlin.test — works on all targets)
assertFailsWith<IllegalArgumentException> {
    repository.createTask("")
}

// Coroutine-specific
advanceUntilIdle() // Process all pending coroutines
advanceTimeBy(1000) // Advance virtual time
```

## Testing by Layer

### ViewModel Tests (Unit, commonTest)

```kotlin
@OptIn(ExperimentalCoroutinesApi::class)
class TaskListViewModelTest {

    @BeforeTest
    fun setup() {
        // kotlinx-coroutines-test: works on all targets; runTest reuses Main's scheduler
        Dispatchers.setMain(StandardTestDispatcher())
    }

    @AfterTest
    fun teardown() {
        Dispatchers.resetMain()
    }

    @Test
    fun toggleTaskUpdatesTaskCompletionStatus() = runTest {
        val repository = FakeTaskRepository(
            initialTasks = listOf(Task("1", "Test", completed = false)),
        )

        val viewModel = TaskListViewModel(GetTasksUseCase(repository))
        viewModel.onEvent(TaskListEvent.ToggleTask("1"))
        advanceUntilIdle()

        val updated = repository.updatedTasks.single()
        assertEquals("1", updated.id)
        assertTrue(updated.completed)
    }
}
```

There is no JUnit `TestWatcher`/`MainDispatcherRule` in `commonTest` — `@BeforeTest`/`@AfterTest` with `Dispatchers.setMain` is the multiplatform equivalent.

### Repository Tests (Integration, commonTest)

```kotlin
class DefaultTaskRepositoryTest {
    private val dao = FakeTaskDao()
    private val api = FakeTaskApi(tasks = listOf(TaskResponse("1", "Remote task")))
    private val repository = DefaultTaskRepository(dao, api, TaskMapper())

    @Test
    fun syncTasksSavesApiResponseToLocalDatabase() = runTest {
        repository.syncTasks()

        val stored = dao.observeAll().first()
        assertEquals(1, stored.size)
        assertEquals("Remote task", stored[0].title)
    }

    @Test
    fun syncTasksPropagatesNetworkError() = runTest {
        api.failWith = TaskApiException("timeout")

        assertFailsWith<TaskApiException> {
            repository.syncTasks()
        }
    }
}
```

### Room KMP DAO Tests (In-Memory)

Room KMP DAO tests run against an in-memory database with the bundled SQLite driver — no device or emulator required. Room supports the Android, iOS, and JVM targets (not js/wasm), so place these in `jvmTest` for the fast loop or in a shared test source set covering the supported targets.

```kotlin
class TaskDaoTest {
    private lateinit var db: AppDatabase
    private lateinit var dao: TaskDao

    @BeforeTest
    fun setup() {
        db = Room.inMemoryDatabaseBuilder<AppDatabase>()
            .setDriver(BundledSQLiteDriver())
            .setQueryCoroutineContext(Dispatchers.Default)
            .build()
        dao = db.taskDao()
    }

    @AfterTest
    fun teardown() = db.close()

    @Test
    fun observeAllEmitsUpdatesWhenTaskInserted() = runTest {
        val task = TaskEntity("1", "Test", null, false)

        dao.upsert(task)

        val result = dao.observeAll().first()
        assertEquals(1, result.size)
        assertEquals("Test", result[0].title)
    }

    @Test
    fun deleteByIdRemovesCorrectTask() = runTest {
        dao.upsert(TaskEntity("1", "Keep", null, false))
        dao.upsert(TaskEntity("2", "Delete", null, false))

        dao.deleteById("2")

        val result = dao.observeAll().first()
        assertEquals(1, result.size)
        assertEquals("Keep", result[0].title)
    }
}
```

### Compose UI Tests (commonTest)

```kotlin
@OptIn(ExperimentalTestApi::class)
class TaskListScreenTest {

    @Test
    fun showsLoadingIndicatorWhenStateIsLoading() = runComposeUiTest {
        setContent {
            AppTheme {
                TaskListContent(
                    uiState = TaskListUiState.Loading,
                    onToggle = {},
                    onDelete = {},
                )
            }
        }

        onNodeWithTag("loading_indicator").assertIsDisplayed()
    }

    @Test
    fun showsTasksWhenStateIsSuccess() = runComposeUiTest {
        val tasks = listOf(Task("1", "Buy groceries", false))

        setContent {
            AppTheme {
                TaskListContent(
                    uiState = TaskListUiState.Success(tasks),
                    onToggle = {},
                    onDelete = {},
                )
            }
        }

        onNodeWithText("Buy groceries").assertIsDisplayed()
    }
}
```

Dependency: `org.jetbrains.compose.ui:ui-test` in `commonTest`. The same test verifies the shared UI on Android, iOS, desktop, and web via the per-target test tasks — no `createComposeRule`, no JUnit rule.

### Ktor MockEngine for API Tests (commonTest)

```kotlin
class TaskApiTest {

    private fun apiWith(handler: MockRequestHandler): TaskApi {
        val client = HttpClient(MockEngine(handler)) {
            install(ContentNegotiation) { json() }
        }
        return TaskApi(client, baseUrl = "https://example.test")
    }

    @Test
    fun getTasksReturnsParsedResponse() = runTest {
        val api = apiWith {
            respond(
                content = """[{"id": "1", "title": "Test"}]""",
                status = HttpStatusCode.OK,
                headers = headersOf(HttpHeaders.ContentType, "application/json"),
            )
        }

        val result = api.getTasks()

        assertEquals(1, result.size)
        assertEquals("Test", result[0].title)
    }
}
```

MockWebServer remains useful in `jvmTest`/`androidUnitTest` when you need to exercise the real OkHttp engine (interceptors, connection behavior). MockEngine is the default because it runs on every target.

### Flow Testing with Turbine

```kotlin
// Turbine (app.cash.turbine:turbine) makes multi-emission Flow tests readable
// and runs in commonTest. Prefer it over collecting into lists or chaining .first() calls.
@Test
fun uiStateMovesLoadingToSuccessWhenTasksLoad() = runTest {
    val repository = FakeTaskRepository(initialTasks = listOf(task))
    val viewModel = TaskListViewModel(GetTasksUseCase(repository))

    viewModel.uiState.test {
        assertEquals(TaskListUiState.Loading, awaitItem())
        assertIs<TaskListUiState.Success>(awaitItem())
        cancelAndIgnoreRemainingEvents()
    }
}
```

Rules of thumb: one `test { }` block per Flow; always end with an explicit `awaitComplete()`, `cancelAndIgnoreRemainingEvents()`, or consumed-everything state — Turbine fails the test on unconsumed events, which catches unexpected emissions.

### Screenshot Tests

Two supported approaches — both render the shared UI on the JVM (no device), so they are scoped to Android/JVM test source sets and do not run on iOS or web:

- **Compose Preview Screenshot Testing** (official, `com.android.compose.screenshot` plugin): turns existing `@Preview` composables into screenshot tests. `./gradlew :androidApp:updateDebugScreenshotTest` records goldens; `./gradlew :androidApp:validateDebugScreenshotTest` fails CI on diffs.
- **Roborazzi** (Robolectric-based, `androidUnitTest` only): `captureRoboImage()` inside any Robolectric/Compose test; more control (interactions before capture, non-Preview cases).

```kotlin
// Roborazzi example (androidUnitTest)
@Test
fun taskCard_default() {
    composeTestRule.setContent { AppTheme { TaskCard(task) } }
    composeTestRule.onRoot().captureRoboImage()
}
```

Golden images are committed; a failing diff is a review artifact, not a flaky annoyance — keep goldens deterministic (fixed locale, font scale, and date/time inputs). They cover the shared composables' rendering through the Android/JVM pipeline only — platform-specific rendering issues still need the per-target UI tests above.

## Fakes & Test Doubles in commonTest

```kotlin
// A fake is a lightweight real implementation of a commonMain interface with
// test-friendly shortcuts. Fakes compile on every target — mocks do not.
class FakeTaskRepository(
    initialTasks: List<Task> = emptyList(),
) : TaskRepository {
    private val tasks = MutableStateFlow(initialTasks)

    val updatedTasks = mutableListOf<Task>() // recording — replaces coVerify
    var failWith: Throwable? = null          // switchable failure mode

    override fun getTasks(): Flow<List<Task>> = tasks

    override suspend fun updateTask(task: Task) {
        failWith?.let { throw it }
        updatedTasks += task
        tasks.value = tasks.value.map { if (it.id == task.id) task else it }
    }
}
```

```kotlin
// Verification happens through observable state, not a call-recording DSL:
assertEquals(listOf("1"), repository.updatedTasks.map { it.id })

// Stub different outcomes by mutating the fake:
repository.failWith = TaskApiException("timeout")
```

Guidelines:

- One fake per `commonMain` interface, defined once in `commonTest` and reused across all layers' tests.
- Give fakes recording lists for verification and a `failWith` (or similar) switch for error paths.
- Wire fakes through constructor injection; with Metro, use `createDynamicGraph<TestGraph>(overrides)` to swap fakes into a real graph.
- Prefer fakes over mocks even on the JVM: they survive refactors and test behavior, not call sequences.

### MockK on the JVM (scoped)

MockK is available only in JVM test source sets (`jvmTest`, `androidUnitTest`) — it does not compile in `commonTest`. Reach for it there only when a fake is disproportionate (e.g. verifying interactions with a large third-party JVM interface):

```kotlin
// jvmTest or androidUnitTest ONLY
val repo = mockk<TaskRepository>()
coEvery { repo.syncTasks() } just Runs
every { repo.getTasks() } returns flowOf(listOf(task))

coVerify(exactly = 1) { repo.syncTasks() }
```

## Anti-Patterns

| Anti-Pattern | Problem | Fix |
|-------------|---------|-----|
| Testing implementation details | Breaks on refactor | Test behavior and outcomes |
| Shared mutable state between tests | Order-dependent failures | Fresh setup in `@BeforeTest` |
| `Thread.sleep` in tests | Slow, flaky, JVM-only | `advanceUntilIdle()` or `waitUntil` |
| Permanently ignored tests | Dead code, false confidence | Fix or delete |
| MockK or JUnit5 in `commonTest` | Does not compile on non-JVM targets | Hand-written fakes; scope mocks to `jvmTest` |
| Mocking everything | Tests nothing real | Fake at boundaries only |
| No assertions | Test always passes | Every test needs at least one assertion |
| Testing only on the JVM | Platform-specific regressions slip through | `:shared:allTests` in CI, not just `:shared:jvmTest` |
