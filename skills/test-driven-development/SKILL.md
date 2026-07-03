---
name: test-driven-development
description: >-
  Use when implementing new features or fixing bugs in a Kotlin Multiplatform
  module. Write tests before implementation following Red-Green-Refactor with
  kotlin.test in commonTest, hand-written fakes, Turbine for Flow assertions,
  and the Prove-It pattern for bugs.
---

# Test-Driven Development

## Overview

Write tests **before** implementation. The Red-Green-Refactor cycle — write a failing test (RED), write minimal code to pass (GREEN), improve the code without changing behavior (REFACTOR) — produces code that is correct by construction. In KMP the payoff doubles: a test in `commonTest` written with `kotlin.test` runs on every target, so one failing test protects Android, iOS, desktop, and web at once.

## When to Use

- Implementing any new feature or behavior in `shared`
- Fixing any bug (use the Prove-It pattern)
- Adding business logic to ViewModels, use cases, or repositories in `commonMain`
- Before refactoring (establish a test safety net first)

**Skip when:** Purely cosmetic UI changes with no behavioral logic — though shared UI behavior still deserves `runComposeUiTest` coverage (see `multiplatform-testing`).

## Core Process

### Step 1: Set Up the Common Test Toolchain

Tests live in `commonTest` and must compile for every target — no JUnit5, no MockK, no `java.*` or `android.*` APIs.

```kotlin
// shared/build.gradle.kts
kotlin {
    sourceSets {
        commonTest.dependencies {
            implementation(kotlin("test"))               // @Test, assertEquals, assertIs
            implementation(libs.kotlinx.coroutines.test) // runTest, StandardTestDispatcher
            implementation(libs.turbine)                 // Flow assertions, KMP-capable
        }
    }
}
```

Name tests in camelCase. Backtick names with spaces do not survive every target (Android instrumented execution rejects them).

### Step 2: RED — Write a Failing Test

Describe what the code should do before it exists. Dependencies are hand-written fakes implementing the production interface:

```kotlin
// shared/src/commonTest/kotlin/com/example/tasks/FakeTaskRepository.kt
class FakeTaskRepository : TaskRepository {
    private val tasks = MutableStateFlow<List<Task>>(emptyList())
    var failWith: Throwable? = null

    override fun getTasks(): Flow<List<Task>> =
        failWith?.let { error -> flow { throw error } } ?: tasks

    fun emit(newTasks: List<Task>) { tasks.value = newTasks }
}
```

```kotlin
// shared/src/commonTest/kotlin/com/example/tasks/TaskListViewModelTest.kt
@OptIn(ExperimentalCoroutinesApi::class)
class TaskListViewModelTest {
    private val repository = FakeTaskRepository()

    @BeforeTest
    fun setUp() {
        // Common main-dispatcher swap — replaces Android's JUnit MainDispatcherRule
        Dispatchers.setMain(StandardTestDispatcher())
    }

    @AfterTest
    fun tearDown() {
        Dispatchers.resetMain()
    }

    @Test
    fun initialStateIsLoading() {
        val viewModel = TaskListViewModel(GetTasksUseCase(repository))

        assertEquals(TaskListUiState.Loading, viewModel.uiState.value)
    }

    @Test
    fun emitsSuccessWithTasksWhenRepositoryReturnsData() = runTest {
        repository.emit(listOf(Task("1", "Buy groceries", completed = false)))
        val viewModel = TaskListViewModel(GetTasksUseCase(repository))

        viewModel.uiState.test {                       // Turbine
            assertEquals(TaskListUiState.Loading, awaitItem())
            val state = assertIs<TaskListUiState.Success>(awaitItem())
            assertEquals("Buy groceries", state.tasks.single().title)
        }
    }

    @Test
    fun emitsErrorWhenRepositoryThrows() = runTest {
        repository.failWith = IllegalStateException("Network error")
        val viewModel = TaskListViewModel(GetTasksUseCase(repository))

        viewModel.uiState.test {
            assertEquals(TaskListUiState.Loading, awaitItem())
            val state = assertIs<TaskListUiState.Error>(awaitItem())
            assertEquals("Network error", state.message)
        }
    }
}
```

Run it — it must fail:

```bash
./gradlew :shared:jvmTest    # fastest loop; the same test later runs on all targets
```

If it passes immediately, the test does not test what you think.

### Step 3: GREEN — Write Minimal Code to Pass

```kotlin
// shared/src/commonMain/kotlin/com/example/tasks/TaskListViewModel.kt
@Inject // Metro — the platform entry point owns the graph
class TaskListViewModel(getTasks: GetTasksUseCase) : ViewModel() {

    val uiState: StateFlow<TaskListUiState> = getTasks()
        .map<List<Task>, TaskListUiState> { TaskListUiState.Success(it) }
        .catch { emit(TaskListUiState.Error(it.message ?: "Unknown error")) }
        .stateIn(viewModelScope, SharingStarted.WhileSubscribed(5_000), TaskListUiState.Loading)
}
```

Run the test — it must pass. Do not add code "you'll need later"; only what the test demands.

### Step 4: REFACTOR — Improve Without Changing Behavior

- Extract common patterns, improve naming, reduce duplication
- Re-run `./gradlew :shared:jvmTest` after every change
- Before pushing, run the full matrix: `./gradlew :shared:allTests` — code green on JVM can still fail on Kotlin/Native or wasm (threading, time APIs, platform actuals)

### Step 5: Prove-It Pattern for Bug Fixes

For every bug, prove it exists before fixing it:

```kotlin
// Step 1: Write a test that demonstrates the bug
@Test
fun completedTasksDoNotAppearInActiveFilter() = runTest {
    repository.emit(
        listOf(
            Task("1", "Active task", completed = false),
            Task("2", "Done task", completed = true),
        )
    )
    val viewModel = TaskListViewModel(GetTasksUseCase(repository))

    viewModel.uiState.test {
        assertEquals(TaskListUiState.Loading, awaitItem())
        skipItems(1)                              // unfiltered Success
        viewModel.setFilter(TaskFilter.ACTIVE)
        val state = assertIs<TaskListUiState.Success>(awaitItem())
        assertEquals(listOf("Active task"), state.tasks.map { it.title })
    }
}

// Step 2: Run it — confirm it FAILS (proves the bug exists)
// Step 3: Fix the code
// Step 4: Run it — confirm it PASSES (proves the fix works)
// Step 5: Run `./gradlew :shared:allTests` — no regressions on any target
```

### Step 6: Test Pyramid

```
    /‾‾‾‾‾‾‾‾‾\
   / UI tests   \        ~5%  — runComposeUiTest in commonTest, Maestro flows
  / (per target) \             see: multiplatform-testing, e2e-verification
 /‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾\
/ Integration tests \     ~15% — Room KMP in-memory DAO (jvmTest),
| (mostly jvmTest)  |           repository + fakes, Ktor MockEngine
|‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾|
|    Unit tests       |   ~80% — kotlin.test + hand-written fakes in commonTest
| (fast, all targets) |          ViewModels, use cases, mappers
 ‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾
```

### Step 7: Repository Tests Against Fakes

Fake the boundaries (API, DAO), run the real repository, assert on resulting state — not call counts:

```kotlin
class FakeTaskApi : TaskApi {
    var response: List<TaskResponse> = emptyList()
    override suspend fun getTasks(): List<TaskResponse> = response
}

class FakeTaskDao : TaskDao {
    private val stored = MutableStateFlow<List<TaskEntity>>(emptyList())
    override suspend fun upsertAll(entities: List<TaskEntity>) {
        stored.value = (stored.value + entities).distinctBy { it.id }
    }
    override fun observeAll(): Flow<List<TaskEntity>> = stored
}

class TaskRepositoryTest {
    private val api = FakeTaskApi()
    private val dao = FakeTaskDao()
    private val repository = DefaultTaskRepository(api, dao, TaskMapper())

    @Test
    fun syncTasksFetchesFromApiAndSavesToDatabase() = runTest {
        api.response = listOf(TaskResponse(id = "1", title = "Test"))

        repository.syncTasks()

        val saved = dao.observeAll().first()
        assertEquals("1", saved.single().id)
    }
}
```

A fake is ten lines you write once and reuse in every test on every target. If a fake is painful to write, the interface is too wide — fix the design (see `api-and-interface-design`).

### Step 8: Room KMP DAO Tests in jvmTest

The Room KMP in-memory builder needs a driver; run DAO tests in `jvmTest` for the fast lane (JVM APIs are allowed there):

```kotlin
// shared/src/jvmTest/kotlin/com/example/tasks/TaskDaoTest.kt
class TaskDaoTest {
    private lateinit var db: AppDatabase
    private lateinit var dao: TaskDao

    @BeforeTest
    fun setUp() {
        db = Room.inMemoryDatabaseBuilder<AppDatabase>()
            .setDriver(BundledSQLiteDriver())
            .build()
        dao = db.taskDao()
    }

    @AfterTest
    fun tearDown() {
        db.close()
    }

    @Test
    fun upsertAndObserveReturnsTask() = runTest {
        dao.upsert(TaskEntity(id = "1", title = "Test", dueDate = null, completed = false))

        val result = dao.observeAll().first()
        assertEquals("Test", result.single().title)
    }
}
```

Schema, drivers, and per-target persistence setup: see `multiplatform-data-persistence`.

### Step 9: Best Practices

- **State over interactions** — assert on outcomes, not method calls
- **DAMP over DRY** — each test self-contained and readable
- **Prefer real > fakes > stubs** — fake only at system boundaries (API, DB, clock)
- **Descriptive camelCase names** — read like behavior specifications, valid on all targets
- **Use `runTest`** — with `advanceUntilIdle()` when the scheduler must drain; never `delay()` or platform sleeps
- **Fast loop on JVM, full matrix before push** — `:shared:jvmTest` per cycle, `:shared:allTests` per commit

## Common Rationalizations

| Shortcut | Why It Fails |
|----------|-------------|
| "I'll just use MockK like on Android" | MockK is JVM-only bytecode manipulation — it does not compile in commonTest and can never run on iOS, native, or wasm targets. Hand-written fakes work everywhere and survive refactors. |
| "I'll write tests after the code works" | You won't. Tests written after verify what you built, not what was intended. |
| "TDD is slow" | TDD prevents debugging across five targets. Net time is less. |
| "This is too simple to test" | Simple code becomes complex code. Tests catch the transition. |
| "Writing fakes is boilerplate" | A fake is ~10 reusable lines. If it is painful, the interface is too wide — TDD just surfaced a design flaw. |
| "The test passed on the first try, so it's good" | A test that never failed may not test what you think. Make it fail first. |

## Red Flags

- Code written before tests
- Tests that pass immediately (never saw RED)
- `delay()` or platform sleeps in tests instead of `runTest` + `advanceUntilIdle()`
- MockK, Robolectric, or JUnit5 dependencies declared for `commonTest`
- Tests that depend on execution order
- Faking internal classes (fake boundaries only)
- Green on `:shared:jvmTest` but `:shared:allTests` never run
- `@Ignore` tests without issue references

## Verification

- [ ] Tests written BEFORE implementation (Red-Green-Refactor)
- [ ] Every test seen failing before passing
- [ ] Bug fixes use the Prove-It pattern
- [ ] All test doubles are hand-written fakes; no mocking library in commonTest
- [ ] ViewModel tests swap the main dispatcher via `Dispatchers.setMain(StandardTestDispatcher())`
- [ ] Test names are camelCase behavior descriptions
- [ ] Edge cases and error paths covered
- [ ] Fast loop used during development: `./gradlew :shared:jvmTest`
- [ ] Full matrix green: `./gradlew :shared:allTests`
