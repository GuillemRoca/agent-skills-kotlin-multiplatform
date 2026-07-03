---
name: multiplatform-data-persistence
description: >-
  Use when implementing local data storage in a Kotlin Multiplatform project
  with Room KMP, DataStore, or SQLDelight. Covers entities and DAOs in shared
  code, platform-specific database builders, migrations and migration tests,
  DataStore preferences, and the repository pattern that lets web targets
  fall back to in-memory storage.
---

# Multiplatform Data Persistence

## Overview

Room KMP (2.8.x) is the default database for shared code: entities, DAOs, and the database class compile for Android, iOS, and desktop with the `BundledSQLiteDriver`. DataStore handles key-value preferences the same way. Keep every persistence mechanism behind a repository interface — js/wasmJs targets have no Room support and need a fallback implementation.

## When to Use

- Setting up or modifying a Room KMP database in the `shared` module
- Writing or testing database migrations
- Storing user preferences with DataStore across platforms
- Choosing between Room KMP, SQLDelight, and DataStore
- Implementing offline-first data access behind a repository
- Deciding how web (js/wasmJs) targets persist data

**Skip when:** Data is purely in-memory for the life of the process, or comes only from a remote API with no caching requirement.

## Core Process

### Step 1: Choose the Storage Mechanism

| Requirement | Solution |
|------------|----------|
| Structured, relational, queryable data | Room KMP (default) |
| User preferences, flags, simple settings | DataStore (Preferences) |
| SQL-first workflow, hand-written SQL | SQLDelight 2.x |
| Persistence on js/wasmJs | Repository fallback: in-memory or localStorage-backed |
| Secrets and tokens | Platform keystores — see `security-and-hardening` |
| **Never in shared code** | Direct SharedPreferences or NSUserDefaults calls |

Room KMP vs SQLDelight:

| Aspect | Room KMP 2.8 | SQLDelight 2.2 |
|--------|--------------|----------------|
| Schema definition | Kotlin annotations (`@Entity`, `@Dao`) | `.sq` files; Kotlin API generated from SQL |
| Code generation | KSP, configured per target | Gradle plugin, per database |
| Driver | `BundledSQLiteDriver` on all supported targets | Per-platform drivers (Android, native, JDBC, web worker) |
| Web support | None on js/wasmJs | js via web worker driver; wasmJs limited |
| Migrations | Auto-migrations + `Migration` classes, exported schemas | `.sqm` files verified by the Gradle plugin |
| Fits best | Teams coming from Android Room; annotation-driven | SQL-first teams; projects needing js persistence |

Default to Room KMP unless the team prefers writing SQL directly or needs SQL on the web target.

### Step 2: Gradle Setup (Room KMP)

Configure KSP per target — Room generates platform actuals for each one:

```kotlin
// shared/build.gradle.kts
plugins {
    alias(libs.plugins.kotlinMultiplatform)
    alias(libs.plugins.androidKotlinMultiplatformLibrary)
    alias(libs.plugins.ksp)
    alias(libs.plugins.room)
}

kotlin {
    sourceSets {
        commonMain.dependencies {
            implementation(libs.androidx.room.runtime)
            implementation(libs.androidx.sqlite.bundled)
        }
    }
}

dependencies {
    add("kspAndroid", libs.androidx.room.compiler)
    add("kspIosArm64", libs.androidx.room.compiler)
    add("kspIosSimulatorArm64", libs.androidx.room.compiler)
    add("kspJvm", libs.androidx.room.compiler)
}

room {
    schemaDirectory("$projectDir/schemas")
}
```

```toml
# gradle/libs.versions.toml (excerpt)
[versions]
room = "2.8.4"
sqlite = "2.6.2"

[libraries]
androidx-room-runtime = { module = "androidx.room:room-runtime", version.ref = "room" }
androidx-room-compiler = { module = "androidx.room:room-compiler", version.ref = "room" }
androidx-sqlite-bundled = { module = "androidx.sqlite:sqlite-bundled", version.ref = "sqlite" }

[plugins]
room = { id = "androidx.room", version.ref = "room" }
```

If `shared` also targets js/wasmJs, these dependencies cannot sit in commonMain — see Step 6.

### Step 3: Entities, DAOs, and Database in Shared Code

```kotlin
@Entity(tableName = "tasks", indices = [Index(value = ["created_at"])])
data class TaskEntity(
    @PrimaryKey val id: String,
    @ColumnInfo(name = "title") val title: String,
    @ColumnInfo(name = "completed") val completed: Boolean = false,
    @ColumnInfo(name = "created_at") val createdAt: Long,
    @ColumnInfo(name = "updated_at") val updatedAt: Long,
)
```

Use `kotlin.time.Clock.System.now().toEpochMilliseconds()` for timestamps — never `System.currentTimeMillis()`, which is JVM-only.

```kotlin
@Dao
interface TaskDao {
    @Query("SELECT * FROM tasks ORDER BY created_at DESC")
    fun observeAll(): Flow<List<TaskEntity>>

    @Query("SELECT * FROM tasks WHERE id = :taskId")
    suspend fun getById(taskId: String): TaskEntity?

    @Upsert
    suspend fun upsertAll(tasks: List<TaskEntity>)

    @Query("DELETE FROM tasks WHERE id = :taskId")
    suspend fun deleteById(taskId: String)

    @Query("DELETE FROM tasks")
    suspend fun deleteAll()

    @Transaction
    suspend fun replaceAll(tasks: List<TaskEntity>) {
        deleteAll()
        upsertAll(tasks)
    }
}
```

```kotlin
@Database(entities = [TaskEntity::class], version = 1, exportSchema = true)
@ConstructedBy(AppDatabaseConstructor::class)
abstract class AppDatabase : RoomDatabase() {
    abstract fun taskDao(): TaskDao
}

// Room generates the actual object for every KSP-configured target
@Suppress("NO_ACTUAL_FOR_EXPECT")
expect object AppDatabaseConstructor : RoomDatabaseConstructor<AppDatabase> {
    override fun initialize(): AppDatabase
}
```

### Step 4: Platform Database Builders

Each platform supplies a `RoomDatabase.Builder` pointing at its own file location; shared code finishes construction with the bundled driver:

```kotlin
// Shared (commonMain, or the intermediate source set from Step 6)
fun buildDatabase(builder: RoomDatabase.Builder<AppDatabase>): AppDatabase =
    builder
        .setDriver(BundledSQLiteDriver())
        .setQueryCoroutineContext(Dispatchers.IO)
        .build()
```

```kotlin
// androidMain
fun databaseBuilder(context: Context): RoomDatabase.Builder<AppDatabase> =
    Room.databaseBuilder<AppDatabase>(
        context = context,
        name = context.getDatabasePath("app.db").absolutePath,
    )
```

```kotlin
// iosMain
fun databaseBuilder(): RoomDatabase.Builder<AppDatabase> =
    Room.databaseBuilder<AppDatabase>(name = documentDirectory() + "/app.db")

@OptIn(ExperimentalForeignApi::class)
private fun documentDirectory(): String {
    val url = NSFileManager.defaultManager.URLForDirectory(
        directory = NSDocumentDirectory,
        inDomain = NSUserDomainMask,
        appropriateForURL = null,
        create = false,
        error = null,
    )
    return requireNotNull(url?.path)
}
```

```kotlin
// jvmMain
fun databaseBuilder(): RoomDatabase.Builder<AppDatabase> {
    val dbFile = File(System.getProperty("user.home"), ".myapp/app.db")
    dbFile.parentFile.mkdirs()
    return Room.databaseBuilder<AppDatabase>(name = dbFile.absolutePath)
}
```

Wire through Metro so only the platform graph knows the concrete builder:

```kotlin
// androidMain
@ContributesTo(AppScope::class)
@BindingContainer
object DatabaseBindings {
    @Provides
    @SingleIn(AppScope::class)
    fun provideDatabase(context: Context): AppDatabase =
        buildDatabase(databaseBuilder(context))

    @Provides
    fun provideTaskDao(db: AppDatabase): TaskDao = db.taskDao()
}
```

### Step 5: DataStore Preferences

`androidx.datastore` core and preferences artifacts work in shared code; only the file path is platform-specific:

```kotlin
// commonMain
fun createDataStore(producePath: () -> String): DataStore<Preferences> =
    PreferenceDataStoreFactory.createWithPath(produceFile = { producePath().toPath() })

internal const val DATA_STORE_FILE = "settings.preferences_pb"

object SettingsKeys {
    val DARK_MODE = booleanPreferencesKey("dark_mode")
    val SORT_ORDER = stringPreferencesKey("sort_order")
}

class SettingsRepository(private val dataStore: DataStore<Preferences>) {
    val darkMode: Flow<Boolean> =
        dataStore.data.map { it[SettingsKeys.DARK_MODE] ?: false }

    suspend fun setDarkMode(enabled: Boolean) {
        dataStore.edit { it[SettingsKeys.DARK_MODE] = enabled }
    }
}
```

```kotlin
// androidMain
fun createDataStore(context: Context): DataStore<Preferences> =
    createDataStore { context.filesDir.resolve(DATA_STORE_FILE).absolutePath }

// iosMain — reuse documentDirectory() from Step 4
fun createDataStore(): DataStore<Preferences> =
    createDataStore { documentDirectory() + "/$DATA_STORE_FILE" }

// jvmMain
fun createDataStore(): DataStore<Preferences> =
    createDataStore {
        File(System.getProperty("user.home"), ".myapp/$DATA_STORE_FILE").absolutePath
    }
```

Do not put tokens or secrets in DataStore — see `security-and-hardening`.

### Step 6: Web Targets — Repository Interface Plus Fallback

Room publishes no js/wasmJs artifacts. If `shared` targets the web, keep only the repository interface in commonMain and move Room code into an intermediate source set shared by the Room-capable targets:

```kotlin
// shared/build.gradle.kts
kotlin {
    applyDefaultHierarchyTemplate()
    sourceSets {
        val roomMain by creating {
            dependsOn(commonMain.get())
            dependencies {
                implementation(libs.androidx.room.runtime)
                implementation(libs.androidx.sqlite.bundled)
            }
        }
        androidMain.get().dependsOn(roomMain)
        iosMain.get().dependsOn(roomMain)
        jvmMain.get().dependsOn(roomMain)
    }
}
```

```kotlin
// commonMain — the only persistence surface the rest of the app sees
interface TaskRepository {
    fun observeTasks(): Flow<List<Task>>
    suspend fun syncTasks()
    suspend fun addTask(task: Task)
}
```

```kotlin
// wasmJsMain (and jsMain) — in-memory fallback bound via Metro
@ContributesBinding(AppScope::class)
@SingleIn(AppScope::class)
@Inject
class InMemoryTaskRepository(private val api: TaskApi) : TaskRepository {
    private val tasks = MutableStateFlow<List<Task>>(emptyList())

    override fun observeTasks(): Flow<List<Task>> = tasks

    override suspend fun syncTasks() {
        tasks.value = api.getTasks().map { it.toDomain() }
    }

    override suspend fun addTask(task: Task) {
        tasks.update { it + task }
        api.createTask(task.toRequest())
    }
}
```

A localStorage-backed implementation (serialize with kotlinx.serialization, write via `kotlinx.browser.window.localStorage`) upgrades this to survive page reloads. If the web target genuinely needs SQL, evaluate SQLDelight's web worker driver instead.

### Step 7: Migrations

Never ship `fallbackToDestructiveMigration()`. Export schemas (`exportSchema = true` plus the `schemaDirectory` from Step 2) and commit the `schemas/` directory.

Prefer auto-migrations for additive changes:

```kotlin
@Database(
    entities = [TaskEntity::class],
    version = 2,
    exportSchema = true,
    autoMigrations = [AutoMigration(from = 1, to = 2)],
)
@ConstructedBy(AppDatabaseConstructor::class)
abstract class AppDatabase : RoomDatabase() {
    abstract fun taskDao(): TaskDao
}
```

Write manual migrations against `SQLiteConnection` when auto-migration cannot infer the change:

```kotlin
val MIGRATION_1_2 = object : Migration(1, 2) {
    override fun migrate(connection: SQLiteConnection) {
        connection.execSQL(
            "ALTER TABLE tasks ADD COLUMN priority INTEGER NOT NULL DEFAULT 0"
        )
    }
}
```

Test every migration. `androidx.room:room-testing` is KMP-capable; jvmTest is the fastest loop:

```kotlin
// shared/src/jvmTest
class MigrationTest {
    @Test
    fun migrate1To2_preservesRowsAndAddsPriority() {
        val helper = MigrationTestHelper(
            schemaDirectoryPath = Path("schemas"),
            databasePath = Path(SystemTemporaryDirectory, "migration-test.db"),
            driver = BundledSQLiteDriver(),
            databaseClass = AppDatabase::class,
        )
        helper.createDatabase(version = 1).use { connection ->
            connection.execSQL(
                "INSERT INTO tasks (id, title, completed, created_at, updated_at) " +
                    "VALUES ('1', 'Test', 0, 0, 0)"
            )
        }
        helper.runMigrationsAndValidate(version = 2, migrations = listOf(MIGRATION_1_2))
            .use { connection ->
                connection.prepare("SELECT priority FROM tasks WHERE id = '1'").use { stmt ->
                    assertTrue(stmt.step())
                    assertEquals(0L, stmt.getLong(0))
                }
            }
    }
}
```

See `multiplatform-testing` for source-set layout and test tasks.

### Step 8: Repository Pattern and Mappers

The local database is the single source of truth; sync writes to it and the UI observes it:

```kotlin
// roomMain (Room-capable targets)
@ContributesBinding(AppScope::class)
@SingleIn(AppScope::class)
@Inject
class RoomTaskRepository(
    private val dao: TaskDao,
    private val api: TaskApi,
) : TaskRepository {

    override fun observeTasks(): Flow<List<Task>> =
        dao.observeAll().map { entities -> entities.map { it.toDomain() } }

    override suspend fun syncTasks() {
        val remote = api.getTasks()
        dao.replaceAll(remote.map { it.toEntity() })
    }

    override suspend fun addTask(task: Task) {
        dao.upsertAll(listOf(task.toEntity()))
        try {
            api.createTask(task.toRequest())
        } catch (e: IOException) {
            // Local write already succeeded; queue or retry the remote sync
        }
    }
}
```

Keep separate models per layer and map at the boundary:

```kotlin
// Domain model — what ViewModels and use cases see
data class Task(val id: String, val title: String, val completed: Boolean)

// Network model — API contract, kotlinx.serialization
@Serializable
data class TaskResponse(val id: String, val title: String, val completed: Boolean)

fun TaskEntity.toDomain() = Task(id = id, title = title, completed = completed)

fun TaskResponse.toEntity() = TaskEntity(
    id = id,
    title = title,
    completed = completed,
    createdAt = Clock.System.now().toEpochMilliseconds(),
    updatedAt = Clock.System.now().toEpochMilliseconds(),
)
```

Room entities never reach the UI layer — see `multiplatform-architecture` for layer boundaries and `api-and-interface-design` for the repository contract itself.

## Common Rationalizations

| Shortcut | Why It Fails |
|----------|-------------|
| "fallbackToDestructiveMigration is fine for now" | Users on every platform lose their data at the first schema change. Write migrations from day one. |
| "I'll use SharedPreferences on Android and NSUserDefaults on iOS" | Two implementations, two bug surfaces, nothing testable in commonTest. DataStore works in shared code. |
| "Web can skip persistence; we'll bolt it on later" | Without a repository interface now, DAO calls spread through shared code and the web target stops compiling. |
| "One model for entity, domain, and network is simpler" | Schema, API, and UI then change in lockstep; every backend field rename becomes a database migration. |
| "Migration tests need a device, so skip them" | The KMP MigrationTestHelper runs in jvmTest in seconds. Untested migrations fail in production, where recovery is hardest. |

## Red Flags

- `fallbackToDestructiveMigration()` anywhere in production code
- Room types (`TaskEntity`, DAOs) imported by ViewModels or composables
- `android.content.SharedPreferences` or `NSUserDefaults` used directly for app preferences
- `exportSchema = false`, or no committed `schemas/` directory
- Room dependencies declared in commonMain while `shared` targets js/wasmJs
- `suspend fun getAll(): List<T>` where the UI needs to observe changes — return `Flow`
- ViewModels calling DAOs directly instead of a repository interface
- Secrets stored in DataStore or the database — see `security-and-hardening`

## Verification

- [ ] Entities, DAOs, and `@Database` compile in shared code with `@ConstructedBy` and the expect constructor
- [ ] KSP configured for every Room target (`kspAndroid`, `kspIosArm64`, `kspIosSimulatorArm64`, `kspJvm`)
- [ ] Each platform provides its `RoomDatabase.Builder`; shared code applies `BundledSQLiteDriver`
- [ ] `exportSchema = true` and `schemas/` committed to version control
- [ ] Every schema change has a migration (auto or manual) with a passing jvmTest
- [ ] Preferences go through DataStore with per-platform path providers
- [ ] All persistence sits behind a repository interface in commonMain; web target has a working fallback
- [ ] Separate entity, domain, and network models with explicit mappers
- [ ] `./gradlew :shared:allTests` passes
