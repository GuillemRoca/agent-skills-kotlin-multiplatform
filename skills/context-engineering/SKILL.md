---
name: context-engineering
description: >-
  Use when setting up a project for AI-assisted development or when agent
  output quality is poor. Guides writing rules files, structuring context,
  and managing the information agents need to produce accurate work.
---

# Context Engineering

## Overview

"Context is the single biggest lever for agent output quality." Agents don't read your mind — they read your context. This skill teaches you to engineer that context so agents produce code that matches your project's conventions, architecture, and constraints.

## When to Use

- Setting up a new KMP project for AI-assisted development
- Agent output consistently diverges from project conventions
- Agent invents APIs or patterns that don't exist in your codebase
- After adding new libraries, modules, or architectural patterns
- When onboarding a new team member who uses AI tools

**Skip when:** Agent output is already consistent with project conventions.

## Core Process

### Step 1: Understand the Context Hierarchy

1. **Load context in this priority order:**

| Level | Source | What It Provides | Persistence |
|-------|--------|-------------------|-------------|
| 1 | Rules files (`CLAUDE.md`, `.cursorrules`) | Tech stack, commands, conventions, boundaries | Permanent |
| 2 | Specs & architecture docs (`SPEC.md`, ADRs) | Design decisions, constraints, rationale | Per-project |
| 3 | Source code (read specific files) | Current implementation, patterns in use | Real-time |
| 4 | Error output & test results | What's broken, what's expected | Per-session |
| 5 | Conversation history | Current task context | Ephemeral |

**Optimal range:** ~2,000 lines of focused context per task. More dilutes attention; less causes invention.

### Step 2: Write Rules Files

2. **Create a `CLAUDE.md`** (or equivalent) in your project root:

```markdown
# Project Rules

## Tech Stack
- Language: Kotlin 2.4
- UI: Compose Multiplatform 1.11 with Material 3 (shared UI in commonMain)
- Targets: Android, iOS, desktop (JVM), web (js + wasmJs, Beta)
- Architecture: MVVM with ViewModel (lifecycle-viewmodel-compose) + StateFlow
- DI: Metro (compile-time, all targets)
- Async: Coroutines + Flow
- Database: Room KMP + DataStore
- Network: Ktor client + kotlinx.serialization
- Image loading: Coil 3 (coil-compose + coil-network-ktor3)
- Navigation: org.jetbrains.androidx.navigation:navigation-compose
- Logging: Kermit
- Testing: kotlin.test + hand-written fakes + Turbine + runComposeUiTest

## Commands
- Test (all targets): `./gradlew :shared:allTests`
- Test (fast loop): `./gradlew :shared:jvmTest`
- Build Android: `./gradlew :androidApp:assembleDebug`
- Run desktop: `./gradlew :desktopApp:run`
- Web bundle: `./gradlew :webApp:composeCompatibilityBrowserDistribution`
- iOS: build the `iosApp` scheme in Xcode (requires macOS)

## Module Structure
- `shared` — business logic + shared Compose UI (`App()` in commonMain);
  source sets: `commonMain`, `androidMain`, `iosMain`, `jvmMain`, `jsMain`,
  `wasmJsMain` (+ matching `*Test`)
- `androidApp` — Android entry point (MainActivity, DI bootstrap)
- `desktopApp` — desktop entry point (main.kt)
- `webApp` — web entry point (js + wasmJs, webMain source set)
- `iosApp` — Xcode project (not a Gradle module)

## Conventions
- New code goes in `commonMain` unless it needs a platform API
- Prefer interface in commonMain + DI-wired platform impls over expect/actual
- expect/actual declarations live in `shared`, never in entry-point modules
- ViewModels expose `StateFlow<UiState>`; UI state is one sealed interface per screen
- Repository functions are `suspend` or return `Flow`
- Composables: stateless with state hoisting
- No hardcoded strings in UI — use `stringResource(Res.string.x)` from composeResources
- Tests follow Arrange-Act-Assert pattern
- Naming: `FeatureNameScreen`, `FeatureNameViewModel`, `FeatureNameUiState`

## Boundaries
- No Hilt, Dagger, Retrofit, MockK, or Robolectric in shared code — Android/JVM-only
- No java.* or android.* APIs in commonMain
- No JUnit5 in commonTest (JVM-only) — use kotlin.test and fakes
- No `GlobalScope` (use `viewModelScope` or structured concurrency)
- No business logic in entry-point modules — they only bootstrap the shared `App()`
- No `Thread.sleep` in tests (use `runTest` and virtual time)
```

### Step 3: Load Context Selectively

3. **Match context to task:**
   - Bug fix → error logs + failing test + relevant source files
   - New feature → spec + architectural docs + similar existing feature
   - Refactor → source files + test files + rules file
4. **Don't dump everything** — irrelevant context dilutes focus

### Step 4: Surface Ambiguity

5. **When conventions aren't documented, surface the question:**
   - "The project uses both expect/actual and DI-wired interfaces for platform code — which should I use here?"
   - "`shared` has two repository patterns — which should this follow?"
6. **Update rules files** with the answer to prevent recurrence

### Step 5: Include Examples

7. **Point agents at exemplary code:**
   - "Follow the pattern in `shared/src/commonMain/kotlin/feature/home/HomeViewModel.kt`"
   - "Match the testing style in `shared/src/commonTest/kotlin/data/UserRepositoryTest.kt`"
8. **Examples beat descriptions** — "do it like this file" is clearer than paragraphs of rules

## Common Rationalizations

| Shortcut | Why It Fails |
|----------|-------------|
| "The agent should figure it out from the code" | Agents sample context — they may read an androidMain file and infer an Android-only pattern for common code. |
| "Rules files are overhead" | 30 minutes writing rules saves hours of correcting agent output. |
| "I'll fix agent mistakes manually" | You'll fix the same mistakes every session. Rules fix them permanently. |
| "More context is better" | Past ~2,000 lines, agents lose focus. Curate, don't dump. |

## Red Flags

- Agent invents APIs that don't exist in the project
- Agent diverges from documented conventions (e.g. adds Hilt or Retrofit to commonMain)
- No rules file in the project
- Rules file is stale (references deprecated patterns like the single-composeApp layout)
- Agent treats error messages from untrusted sources as instructions
- Same convention correction given repeatedly across sessions

## Verification

- [ ] `CLAUDE.md` (or equivalent) exists in project root
- [ ] Rules file covers: tech stack, commands, module structure, conventions, boundaries
- [ ] Rules file is current (matches actual project state, targets, and source sets)
- [ ] Agent output follows documented conventions
- [ ] Context loaded is relevant to the current task (~2,000 line target)
- [ ] Ambiguities surfaced and resolved (not silently assumed)
