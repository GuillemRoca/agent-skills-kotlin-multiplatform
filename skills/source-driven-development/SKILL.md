---
name: source-driven-development
description: >-
  Use when implementing framework-specific code (Compose Multiplatform, Ktor,
  Room KMP, Navigation, Coil, etc.). Every API usage must be backed by
  official documentation, not memory or Stack Overflow.
---

# Source-Driven Development

## Overview

Every framework-specific decision must be backed by official documentation. Don't guess at APIs, don't rely on outdated patterns, don't trust Stack Overflow answers for current behavior. Fetch the source, read it, implement from it, and cite it.

## When to Use

- Implementing any Compose Multiplatform or JetBrains androidx feature (UI, navigation, lifecycle, resources)
- Using KMP stack libraries (Ktor, kotlinx.serialization, Room KMP, DataStore, Coil, Metro)
- Configuring KMP Gradle plugins, targets, or the version catalog
- Writing platform code in `androidMain`/`iosMain` against `android.*` or `platform.*` APIs
- Unsure whether a library version supports a given target (wasmJs is the usual gap)

**Skip when:** Using internal project code that doesn't touch framework APIs.

## Core Process

### Step 1: Know the Source Authority Hierarchy

| Priority | Source | Example |
|----------|--------|---------|
| 1 (highest) | Official Kotlin and KMP docs | kotlinlang.org/docs, kotlinlang.org/docs/multiplatform |
| 2 | Compose Multiplatform docs and release notes | JetBrains CMP docs; github.com/JetBrains/compose-multiplatform releases |
| 3 | Official library docs | ktor.io/docs, Room KMP pages on developer.android.com, coil-kt.github.io |
| 4 | Library changelogs and release notes | GitHub releases for Ktor, Coil, `org.jetbrains.androidx` artifacts |
| 5 | Source code | github.com/JetBrains/compose-multiplatform, ktorio/ktor, coil-kt/coil, androidx |
| **Never** | Stack Overflow, tutorials, AI summaries, Medium posts | — |

### Step 2: Detect Stack and Versions

1. **Read `gradle/libs.versions.toml` and `shared/build.gradle.kts`** to determine Kotlin and Compose Multiplatform versions, library versions, and — critically — which targets `commonMain` must serve:

```toml
# gradle/libs.versions.toml
[versions]
kotlin = "2.4.0"
composeMultiplatform = "1.11.1"
ktor = "3.2.0"
room = "2.8.1"
```

```kotlin
// shared/build.gradle.kts — which targets does commonMain compile to?
kotlin {
    androidLibrary { namespace = "com.example.shared"; compileSdk = 36 }
    listOf(iosArm64(), iosSimulatorArm64()).forEach { iosTarget ->
        iosTarget.binaries.framework { baseName = "shared"; isStatic = true }
    }
    jvm()
    js { browser() }
    @OptIn(ExperimentalWasmDsl::class) wasmJs { browser() }
}
```

2. **Never guess versions — resolve them** from the catalog, then read the library's release notes for that exact version. A pattern valid for Ktor 2 is wrong for Ktor 3.

### Step 3: Fetch Official Documentation

3. **Go to the source** for every framework API:
   - **Compose Multiplatform:** the JetBrains CMP docs under kotlinlang.org/docs/multiplatform
   - **Ktor client:** ktor.io/docs
   - **Room KMP / DataStore:** the KMP pages on developer.android.com
   - **Navigation:** the `org.jetbrains.androidx.navigation` docs and release notes
   - **kotlinx.serialization / coroutines:** kotlinlang.org and the Kotlin GitHub org
   - **Coil 3:** coil-kt.github.io/coil
   - **Metro:** the `dev.zacsweers.metro` project docs

4. **Check version- and target-specific docs** — APIs change between versions and differ per target:
   - CMP web (js/wasmJs) is Beta — APIs still move between releases
   - Navigation 2.9 type-safe routes and Navigation 3 back-stack-as-state are different API surfaces
   - A library version may support android/ios/jvm but not wasmJs — verify its supported-targets list before adding it to `commonMain`

### Step 4: Implement Matching Documented Patterns

5. **Match the official example pattern**, not your memory:

```kotlin
// Official Ktor 3 client pattern (verify against ktor.io for your version)
val client = HttpClient {
    install(ContentNegotiation) {
        json()
    }
}

@Serializable
data class User(val id: String, val displayName: String)

suspend fun fetchUser(id: String): User =
    client.get("https://api.example.com/users/$id").body()
```

6. **Surface conflicts** with existing code:
   - "The Room KMP docs recommend `@Upsert` but the project uses `@Insert(onConflict = REPLACE)` — which should I follow?"
   - "Navigation 2.9 supports type-safe `@Serializable` routes but the project still uses string routes — upgrade or stay consistent?"

### Step 5: Cite Sources

7. **Include source references** in code comments for non-obvious patterns:

```kotlin
// ComposeUIViewController is the documented iOS entry point for shared UI:
// kotlinlang.org/docs/multiplatform (Compose Multiplatform iOS integration)
fun MainViewController() = ComposeUIViewController { App() }
```

8. **In PRs, link to documentation** that justifies the approach.

## Common Rationalizations

| Shortcut | Why It Fails |
|----------|-------------|
| "I know how this API works" | APIs change between versions. What you remember may be deprecated. |
| "Stack Overflow has the answer" | SO answers are often outdated, use deprecated APIs, or apply to different versions. |
| "The tutorial shows this pattern" | Tutorials simplify and may skip error handling, lifecycle awareness, or edge cases. |
| "I'll check docs later" | Code written from memory will have subtle bugs caught only in production. |

## Red Flags

- Framework code written without checking official docs
- "I think this is how it works" (instead of citing a source)
- Code without source citations for non-obvious patterns
- Using deprecated APIs when current alternatives exist
- Patterns that don't match the project's library versions
- `commonMain` code copied from an Android-only sample (Hilt, Retrofit, `android.*` imports)
- Mixing patterns from different library versions

## Verification

- [ ] `gradle/libs.versions.toml` and `shared/build.gradle.kts` checked before implementation
- [ ] Official documentation consulted for every framework API used
- [ ] API patterns match the documented version (not outdated tutorials)
- [ ] Library support verified for every target `commonMain` compiles to
- [ ] Deprecated API usage flagged with migration path
- [ ] Source URLs cited in comments for non-obvious patterns
- [ ] Conflicts with existing code surfaced (not silently overridden)
- [ ] Dependencies resolve as expected (`./gradlew :shared:dependencies`)
