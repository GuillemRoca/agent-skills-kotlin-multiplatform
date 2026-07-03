---
name: api-and-interface-design
description: >-
  Use when designing interfaces between layers (Repository, UseCase, Ktor API
  client) or defining data contracts in a Kotlin Multiplatform project. Covers
  contract-first design, Ktor client setup with per-platform engines, sealed
  result types, error modeling, and backward compatibility.
---

# API and Interface Design

## Overview

Hyrum's Law: with enough users of an API, every observable behavior will be depended on by somebody. In a KMP project the stakes are higher — a contract in commonMain is consumed by Android, iOS, desktop, and web at once. Design Repository contracts, Ktor API clients, and ViewModel-UI contracts deliberately, and keep JVM-only libraries (Retrofit, OkHttp) out of commonMain entirely.

## When to Use

- Defining a new Repository or UseCase interface in commonMain
- Building a Ktor HTTP API client for shared code
- Defining ViewModel-to-UI contracts (UiState, events)
- Changing any public interface of the `shared` module
- Modeling errors that cross layer boundaries

**Skip when:** Modifying private implementation details with no observable contract change.

## Core Process

### Step 1: Contract-First Design

Define the interface before implementing, in commonMain, using only domain and standard-library types:

```kotlin
// shared/src/commonMain — domain layer
interface TaskRepository {
    fun observeTasks(): Flow<List<Task>>
    fun observeTask(taskId: TaskId): Flow<Task?>
    suspend fun createTask(title: String, description: String?): Task
    suspend fun updateTask(task: Task)
    suspend fun deleteTask(taskId: TaskId)
    suspend fun syncWithRemote()
}
```

Rules:

- Return `Flow` for observable data; `suspend` for one-shot operations
- Accept and return domain types — no Room entities, no Ktor types
- Everything in commonMain must compile for all targets: no `java.*`, no `android.*`, no JVM-only libraries
- Retrofit and OkHttp are JVM/Android-only and must never appear in commonMain; the shared HTTP client is Ktor

### Step 2: Ktor Client in commonMain

One configured `HttpClient` with plugins for serialization, timeouts, retry, and logging:

```kotlin
fun createHttpClient(engine: HttpClientEngine): HttpClient =
    HttpClient(engine) {
        install(ContentNegotiation) {
            json(Json {
                ignoreUnknownKeys = true
                explicitNulls = false
            })
        }
        install(HttpTimeout) {
            requestTimeoutMillis = 30_000
            connectTimeoutMillis = 10_000
        }
        install(HttpRequestRetry) {
            retryOnServerErrors(maxRetries = 2)
            exponentialDelay()
        }
        install(Logging) {
            logger = object : Logger {
                override fun log(message: String) {
                    co.touchlab.kermit.Logger.d("Ktor") { message }
                }
            }
            level = LogLevel.INFO
        }
        defaultRequest {
            url("https://api.example.com/v1/")
            contentType(ContentType.Application.Json)
        }
    }
```

Wrap endpoints in a typed API class with `@Serializable` request and response models:

```kotlin
class TaskApi(private val client: HttpClient) {
    suspend fun getTasks(page: Int = 1, perPage: Int = 20): TaskListResponse =
        client.get("tasks") {
            parameter("page", page)
            parameter("per_page", perPage)
        }.body()

    suspend fun getTask(id: TaskId): TaskResponse =
        client.get("tasks/${id.value}").body()

    suspend fun createTask(request: CreateTaskRequest): TaskResponse =
        client.post("tasks") { setBody(request) }.body()

    suspend fun updateTask(id: TaskId, request: UpdateTaskRequest): TaskResponse =
        client.put("tasks/${id.value}") { setBody(request) }.body()

    suspend fun deleteTask(id: TaskId) {
        client.delete("tasks/${id.value}")
    }
}

@Serializable
data class CreateTaskRequest(val title: String, val description: String? = null)

@Serializable
data class TaskResponse(val id: String, val title: String, val completed: Boolean)
```

Contract rules: separate request and response models even when fields overlap; never return `HttpResponse` or other Ktor types from a repository.

### Step 3: Engine Selection Per Platform

`ktor-client-core` goes in commonMain; each target adds its engine artifact:

| Source set | Engine | Artifact | Notes |
|-----------|--------|----------|-------|
| androidMain | OkHttp | `ktor-client-okhttp` | OkHttp interceptors live here only |
| iosMain | Darwin | `ktor-client-darwin` | NSURLSession; App Transport Security applies |
| jvmMain | CIO (or Java) | `ktor-client-cio` | Desktop |
| jsMain / wasmJsMain | Js | `ktor-client-js` | Browser fetch; CORS applies |

Provide the engine via an expect factory (or a Metro provider per platform — same shape):

```kotlin
// commonMain
expect fun httpClientEngine(): HttpClientEngine

// androidMain
actual fun httpClientEngine(): HttpClientEngine = OkHttp.create()

// iosMain
actual fun httpClientEngine(): HttpClientEngine = Darwin.create()

// jvmMain
actual fun httpClientEngine(): HttpClientEngine = CIO.create()

// jsMain and wasmJsMain
actual fun httpClientEngine(): HttpClientEngine = Js.create()
```

### Step 4: Sealed Types and Value Classes

Make illegal states unrepresentable — the compiler enforces exhaustiveness on every platform:

```kotlin
// BAD: stringly-typed status invites typos and defeats exhaustiveness
data class Task(val id: String, val title: String, val status: String)

// GOOD: sealed hierarchy
data class Task(val id: TaskId, val title: String, val status: TaskStatus)

sealed interface TaskStatus {
    data object Pending : TaskStatus
    data object InProgress : TaskStatus
    data class Completed(val completedAt: Instant) : TaskStatus
}
```

Value classes prevent argument mix-ups at zero runtime cost on all targets:

```kotlin
@JvmInline
value class TaskId(val value: String)

@JvmInline
value class UserId(val value: String)

suspend fun getTask(taskId: TaskId): Task // cannot pass a UserId by accident
```

### Step 5: Error Modeling at the Boundary

Define domain errors once in commonMain; map Ktor exceptions to them in the repository, never above it:

```kotlin
sealed interface DataError {
    data class Network(val cause: Throwable) : DataError
    data object Timeout : DataError
    data class NotFound(val id: String) : DataError
    data object Unauthorized : DataError
    data class Server(val code: Int) : DataError
    data class Unknown(val cause: Throwable) : DataError
}

sealed interface DataResult<out T> {
    data class Success<T>(val data: T) : DataResult<T>
    data class Error(val error: DataError) : DataResult<Nothing>
}

suspend fun <T> safeCall(block: suspend () -> T): DataResult<T> =
    try {
        DataResult.Success(block())
    } catch (e: CancellationException) {
        throw e // never swallow coroutine cancellation
    } catch (e: ClientRequestException) {
        when (e.response.status) {
            HttpStatusCode.NotFound ->
                DataResult.Error(DataError.NotFound(e.response.call.request.url.toString()))
            HttpStatusCode.Unauthorized -> DataResult.Error(DataError.Unauthorized)
            else -> DataResult.Error(DataError.Unknown(e))
        }
    } catch (e: ServerResponseException) {
        DataResult.Error(DataError.Server(e.response.status.value))
    } catch (e: HttpRequestTimeoutException) {
        DataResult.Error(DataError.Timeout)
    } catch (e: IOException) {
        DataResult.Error(DataError.Network(e))
    }
```

Use `kotlinx.io.IOException` — `java.io.IOException` does not exist off the JVM.

### Step 6: Validation at Boundaries

Validate at system boundaries, trust internally:

```kotlin
class DefaultTaskRepository(
    private val api: TaskApi,
    private val dao: TaskDao,
) : TaskRepository {

    override suspend fun createTask(title: String, description: String?): Task {
        require(title.isNotBlank()) { "Task title must not be blank" }
        require(title.length <= 200) { "Task title must not exceed 200 characters" }

        val request = CreateTaskRequest(title.trim(), description?.trim())
        val response = api.createTask(request)
        dao.upsertAll(listOf(response.toEntity()))
        return response.toDomain()
    }
}
```

### Step 7: Backward Compatibility

Extend, don't modify — the `shared` API surface has five platform consumers plus Swift callers through the iOS framework:

```kotlin
// Version 2: ADD with a default implementation; do not change existing signatures
interface TaskRepository {
    fun observeTasks(): Flow<List<Task>>
    suspend fun createTask(title: String, description: String?): Task

    fun observeTasksByStatus(status: TaskStatus): Flow<List<Task>> =
        observeTasks().map { tasks -> tasks.filter { it.status == status } }
}
```

When a breaking change is unavoidable: deprecate with `@Deprecated(message, ReplaceWith(...))`, migrate all callers in the same PR, and remember Swift call sites in `iosApp` do not refactor automatically. See `deprecation-and-migration` for the full process.

### Step 8: Naming Conventions

| Element | Convention | Example |
|---------|-----------|---------|
| Repository | `observe*` for Flow; `get*`/`create*`/`update*`/`delete*` for suspend | `observeTasks()` |
| UseCase | Verb phrase, `operator fun invoke()` | `GetTasksUseCase()` |
| Ktor API class | HTTP-verb aligned method names | `getTasks()`, `createTask()` |
| Room DAO | `observe*` for Flow, `get*` for suspend — see `multiplatform-data-persistence` | `observeAll()` |
| UiState | Sealed interface per screen | `TaskListUiState.Success` |
| UI events | Past tense or imperative | `TaskClicked`, `DeleteTask` |
| Request/Response | Suffixed | `CreateTaskRequest`, `TaskResponse` |

### Step 9: Test Contracts with MockEngine

Ktor's MockEngine runs in commonTest on every target — no JVM-only mocking library needed:

```kotlin
// shared/src/commonTest
class TaskApiTest {
    @Test
    fun getTasks_parsesResponse() = runTest {
        val engine = MockEngine { request ->
            assertEquals("/v1/tasks", request.url.encodedPath)
            respond(
                content = """{"tasks":[{"id":"1","title":"Test","completed":false}],"total":1}""",
                status = HttpStatusCode.OK,
                headers = headersOf(HttpHeaders.ContentType, "application/json"),
            )
        }
        val api = TaskApi(createHttpClient(engine))

        val response = api.getTasks()

        assertEquals(1, response.total)
        assertEquals("Test", response.tasks.first().title)
    }
}
```

For repository consumers, prefer hand-written fakes of the interface — MockK is JVM-only. See `multiplatform-testing`.

## Common Rationalizations

| Shortcut | Why It Fails |
|----------|-------------|
| "I'll define the interface from the implementation" | Implementation details leak into the contract and all five platforms inherit them. Design the contract first. |
| "Retrofit is familiar — use it for shared networking" | Retrofit is JVM-only. commonMain stops compiling for iOS and web. Ktor is the multiplatform client. |
| "One model for network, database, and UI is simpler" | Coupling the API contract to the DB schema and UI makes all three brittle; one backend rename ripples everywhere. |
| "Strings are fine for status fields" | Strings allow typos, defeat exhaustiveness checking, and hide states from the compiler on every platform. |
| "We don't need backward compatibility yet" | The iOS framework and Swift call sites are consumers you cannot refactor automatically. By the time you need compatibility, you already have them. |

## Red Flags

- Retrofit, OkHttp, or `java.*` imports in commonMain
- Ktor `HttpResponse` or `HttpClient` types in a Repository interface
- Room entities or network DTOs exposed to the UI layer
- Generic `String` where a sealed type or enum belongs
- `catch (e: Exception)` that also swallows `CancellationException`
- Breaking interface changes without `@Deprecated` migration guidance
- No validation at system boundaries
- API client methods returning raw JSON strings

## Verification

- [ ] Interfaces defined before implementations, in commonMain, using domain types only
- [ ] Ktor client configured with ContentNegotiation, timeouts, retry, and logging
- [ ] Each target supplies its engine (OkHttp, Darwin, CIO, Js) via expect/actual or DI
- [ ] Sealed types model finite state sets; value classes wrap identifier strings
- [ ] Ktor exceptions mapped to domain errors at the repository boundary; cancellation rethrown
- [ ] Request and response models are separate `@Serializable` classes
- [ ] MockEngine tests cover the API client in commonTest
- [ ] Public interface changes are additive or ship with `@Deprecated` guidance
- [ ] `./gradlew :shared:allTests` passes
