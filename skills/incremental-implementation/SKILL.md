---
name: incremental-implementation
description: >-
  Use when building features incrementally. Each increment leaves the system
  in a working, testable state. Covers vertical slicing, feature flags,
  and rollback-friendly development for Kotlin Multiplatform.
---

# Incremental Implementation

## Overview

"Each increment should leave the system in a working, testable state." Build features in small, verifiable steps — each one compiles, passes tests, and can be committed. Never leave the codebase in a broken state between increments.

## When to Use

- Implementing any feature from a task breakdown (follows `planning-and-task-breakdown`)
- Building any change that spans more than one file
- Any time you're tempted to "get it all working, then commit"

**Skip when:** A true single-file, single-function change.

## Core Process

### Step 1: Review the Task

1. **Read the task's acceptance criteria** from `tasks/todo.md`
2. **Identify the vertical slice** — what observable behavior does this increment deliver?
3. **Gather existing examples** — find similar patterns already in the codebase

### Step 2: Implement with TDD

4. **For each increment, follow this cycle:**

```
┌──────────────────────────────────────────────────────────┐
│  1. Review acceptance criteria                           │
│  2. Read existing patterns in codebase                   │
│  3. Write failing test in commonTest (RED)               │
│  4. Write minimal code to pass (GREEN)                   │
│  5. Run ./gradlew :shared:jvmTest                        │
│  6. Run ./gradlew :shared:compileKotlinIosSimulatorArm64 │
│  7. Commit                                               │
│  8. Repeat for next acceptance criterion                 │
└──────────────────────────────────────────────────────────┘
```

5. **Never skip the non-JVM compile check.** `:shared:jvmTest` is the fast loop, but after every increment that touches `commonMain`, compile at least one non-JVM target early (e.g. `./gradlew :shared:compileKotlinIosSimulatorArm64`). It catches JVM-only API leaks — `java.time`, `java.io`, JVM-only libraries — before they pile up into an afternoon of untangling.

6. **At checkpoints (every few increments), run the full matrix:** `./gradlew :shared:allTests`.

### Step 3: Primary Slicing Strategies

7. **Vertical slice (preferred):**
   - End-to-end in `commonMain`: Entity → DAO → Repository → ViewModel → Screen (Room KMP, lifecycle-viewmodel-compose, shared Compose UI)
   - Each slice delivers user-visible functionality on every target
   - Example: "User can view task list" before "User can add task"

8. **Contract-first:**
   - Define the interface first (Repository interface, API contract)
   - Implement against the contract
   - Useful for parallel development (one dev does UI, another does data layer)

9. **Risk-first:**
   - Build the most uncertain piece first
   - If the risk materializes, you've spent minimal effort
   - Example: "Does this library actually support wasmJs?" before building the feature that assumes it

### Step 4: Feature Flags

10. **Use feature flags for incomplete features:**

```kotlin
// Compile-time flag: plain Kotlin object in commonMain
object FeatureFlags {
    const val TASK_SHARING = false
}

if (FeatureFlags.TASK_SHARING) {
    ShareButton(onShare = { viewModel.shareTask(task) })
}

// Runtime flag: interface in commonMain, wired through Metro DI
interface FeatureFlagService {
    fun isEnabled(key: String): Flow<Boolean>
}

@Composable
fun TaskListScreen(viewModel: TaskListViewModel) {
    val showSharing by viewModel.isFeatureEnabled("task_sharing")
        .collectAsStateWithLifecycle(initialValue = false)

    TaskListContent(
        showShareButton = showSharing,
        // ...
    )
}
```

11. **Feature flag rules:**
    - Flags have an owner and a removal date
    - Dead flags are tech debt — remove after rollout
    - Test both paths (flag on and flag off) — in `commonTest`, so every target inherits the coverage

### Step 5: Keep It Compilable

12. **The codebase must compile — on all targets — after every commit:**
    - No commented-out code as "TODO" placeholders
    - No unimplemented interfaces throwing `NotImplementedError` (unless behind a feature flag)
    - No `actual` declarations stubbed with `TODO()` on targets you aren't watching

13. **If you're stuck, revert to last green state:**
    ```bash
    # Stash current work
    git stash

    # Verify last commit is green
    ./gradlew :shared:jvmTest :shared:compileKotlinIosSimulatorArm64

    # Try a different approach
    git stash pop
    ```

### Step 6: Artifact Size Checkpoints

14. **Periodically build release artifacts and check their size:**
    ```bash
    ./gradlew :androidApp:bundleRelease
    ./gradlew :desktopApp:packageDistributionForCurrentOS
    ./gradlew :webApp:composeCompatibilityBrowserDistribution
    ```

15. **Watch for size regressions per platform** — a new `commonMain` dependency lands in every artifact, and the web bundle feels it first.

## Common Rationalizations

| Shortcut | Why It Fails |
|----------|-------------|
| "I'll commit when it all works" | Large commits are unreviewable, unrevertable, and hide bugs. |
| "The build is temporarily broken, I'll fix it" | Temporary broken builds block the team and compound errors. |
| "Feature flags are overhead" | Shipping incomplete features to production is worse overhead. |
| "I need to refactor first" | Refactoring is a separate increment. Don't mix it with feature work. |
| "100 lines is too small for a commit" | 100-line commits are reviewable in minutes. 1000-line commits take hours. |

## Red Flags

- 100+ lines of code without running tests
- `commonMain` increments verified only on the JVM — iOS/web breakage discovered days later
- Mixing unrelated changes in one increment
- Expanding scope mid-increment ("while I'm here...")
- Build broken between commits on any target
- No feature flag for partially-complete user-facing features
- Premature abstractions before the second use case
- No verification step between increments

## Verification

- [ ] Each increment has passing tests (`./gradlew :shared:jvmTest`)
- [ ] Each `commonMain` increment compiles for a non-JVM target (`./gradlew :shared:compileKotlinIosSimulatorArm64`)
- [ ] Each increment is committed separately
- [ ] Commits are small and focused (~100 lines)
- [ ] Incomplete features behind feature flags
- [ ] No mixed refactoring + feature changes in one increment
- [ ] Acceptance criteria from task checked off after each increment
- [ ] Checkpoint: full test matrix green (`./gradlew :shared:allTests`)
