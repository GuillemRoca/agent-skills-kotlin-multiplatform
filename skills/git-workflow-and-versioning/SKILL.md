---
name: git-workflow-and-versioning
description: >-
  Use when managing branches, commits, and release versioning for a Kotlin
  Multiplatform + Compose Multiplatform project. Covers trunk-based development,
  atomic commits, and a single source-of-truth app version fanned out to
  Android, iOS, desktop, and web.
---

# Git Workflow and Versioning

## Overview

Trunk-based development keeps `main` always deployable, uses short-lived feature branches (1-3 days), and makes atomic commits that address one logical concern. KMP versioning has a twist: five artifacts (Android AAB, iOS IPA, desktop installers, and js/wasm web bundles) all ship from one repository, so define the version once and fan it out — never let platforms drift.

## When to Use

- Starting a new feature branch
- Making commits during development
- Preparing a coordinated multi-platform release
- Bumping the app version across all targets
- Reviewing branch strategy or merge approach
- Setting up per-platform version wiring

**Skip when:** The project has an established, documented git workflow and version-fanout that already satisfies the mapping table below.

## Core Process

### Step 1: Trunk-Based Development

1. **Branch strategy:**

```
main (always deployable — CI is green across all targets)
  ├── feature/task-sharing      (1-3 days, then merge)
  ├── feature/offline-sync      (1-3 days, then merge)
  ├── fix/crash-on-empty-list   (hours, then merge)
  └── release/1.2.0             (cut from main, hotfixes only)
```

2. **Branch rules:**
   - `main` is always green: `./gradlew :shared:allTests` passes (plus iOS sim tests on a macOS runner)
   - Feature branches are short-lived (1-3 days max)
   - Delete branches after merge
   - No long-lived feature branches — use feature flags in `shared` instead (see `shipping-and-launch`)
   - Release branches are cut from `main`, not from feature branches

### Step 2: Atomic Commits

3. **Each commit addresses one logical concern:**

```bash
# GOOD: atomic commits
git commit -m "$(cat <<'EOF'
Add TaskDao with Room KMP CRUD operations

Room DAO in commonMain for the tasks table: observe (Flow), upsert,
and delete. Reactive observation drives the shared Compose UI.
EOF
)"

git commit -m "$(cat <<'EOF'
Add TaskRepository with offline-first sync

Local Room database is the source of truth. Remote sync via the Ktor
client behind the TaskApi interface, wired through Metro DI. Network
failures fall back to cached data.
EOF
)"

# BAD: kitchen sink commit
git commit -m "Add task feature with database, API, UI, and tests"
```

4. **Commit message format:**
   - First line: imperative, under 72 characters ("Add", "Fix", "Update", "Remove")
   - Blank line, then a body explaining *why*, not *what* (the diff shows what)
   - Reference issues: `Fixes #42`

### Step 3: Change Sizing

5. **Target ~100 lines per commit:**

| Size | Lines | Review Time | Action |
|------|-------|-------------|--------|
| Small | < 50 | Minutes | Merge quickly |
| Medium | 50-200 | ~30 min | Standard review |
| Large | 200-500 | Hours | Consider splitting |
| Too Large | > 500 | Days | **Must split** |

6. **Split strategies:**
   - Refactoring separate from feature work
   - `commonMain` logic separate from platform `actual` implementations
   - Shared Compose UI separate from data layer
   - Tests in the same commit as the code they test

### Step 4: Save-Point Pattern

7. **Commits as checkpoints — use the fastest test loop:**

```bash
# Before risky changes (jvmTest is the fastest KMP loop):
./gradlew :shared:jvmTest && git add -A && git commit -m "Checkpoint: working state before refactor"

# Try the change... if it breaks:
git revert HEAD

# If it works, continue to the next increment
```

### Step 5: Single Source of Truth for the App Version

8. **Declare the version once in `gradle/libs.versions.toml`:**

```toml
[versions]
# Marketing version, semantic. Bump here and only here.
appVersion = "1.2.0"
# Monotonic build number, driven by CI (e.g. run number). Local default 0.
appBuild = "0"
```

9. **Semantics:**

```
appVersion: MAJOR.MINOR.PATCH
  MAJOR: breaking changes / major redesign
  MINOR: new features, backward compatible
  PATCH: bug fixes
appBuild: monotonically increasing integer (CI run number)
  Used where a store demands a strictly increasing counter.
```

Read the catalog values once in a convention script or root `build.gradle.kts`:

```kotlin
// helper available to every module (e.g. in buildSrc or a settings ext)
val appVersion = libs.versions.appVersion.get()          // "1.2.0"
val appBuild = libs.versions.appBuild.get().toInt()       // CI run number
val (maj, min, patch) = appVersion.split(".").map(String::toInt)
val versionCodeValue = maj * 10000 + min * 100 + patch    // 10200
```

### Step 6: Fan the Version Out to Every Platform

10. **Mapping table — one version, five destinations:**

| Platform | Field(s) | Where | Derived from |
|----------|----------|-------|--------------|
| Android | `versionName` / `versionCode` | `androidApp/build.gradle.kts` | name = `appVersion`; code = `maj*10000+min*100+patch` |
| iOS | `CFBundleShortVersionString` / `CFBundleVersion` | generated `Version.xcconfig` or `agvtool` | short = `appVersion`; bundle = `appBuild` |
| Desktop | `packageVersion` | `desktopApp` `nativeDistributions {}` | = `appVersion` (MAJOR >= 1 required for Msi/Dmg) |
| Web | deploy label | bundle metadata / hosting path | = `appVersion+appBuild` |

11. **Android** (`androidApp/build.gradle.kts`, `com.android.application`):

```kotlin
android {
    defaultConfig {
        versionName = appVersion
        versionCode = versionCodeValue
    }
}
```

12. **iOS** — generate an xcconfig from Gradle so Xcode reads the same source:

```kotlin
// androidApp/desktopApp are Gradle; iosApp is an Xcode project. Emit an
// xcconfig the Xcode project includes, so the version stays single-sourced.
tasks.register("generateIosVersionConfig") {
    val out = rootProject.file("iosApp/Configuration/Version.xcconfig")
    doLast {
        out.parentFile.mkdirs()
        out.writeText(
            "MARKETING_VERSION=$appVersion\n" +
            "CURRENT_PROJECT_VERSION=$appBuild\n"
        )
    }
}
```

```bash
# CI alternative without a generated xcconfig: agvtool, run in iosApp/
cd iosApp
agvtool new-marketing-version "1.2.0"          # CFBundleShortVersionString
agvtool new-version -all "$GITHUB_RUN_NUMBER"  # CFBundleVersion
```

13. **Desktop** (`desktopApp/build.gradle.kts`):

```kotlin
compose.desktop {
    application {
        nativeDistributions {
            packageName = "TaskApp"
            packageVersion = appVersion   // Dmg/Msi/Deb version
        }
    }
}
```

14. **Web** — the `composeCompatibilityBrowserDistribution` bundle is deploy-versioned; stamp `appVersion+appBuild` into the deploy path or a `version.json` served beside `index.html` so rollbacks and cache-busting can reference it (see `shipping-and-launch`).

### Step 7: Tag the Release Once

15. **One annotated tag per release covers all platforms — never tag per platform:**

```bash
# After main is green and the version bump is merged:
git tag -a v1.2.0 -m "Release 1.2.0"
git push origin v1.2.0
# CI builds every artifact from this single tag.
```

### Step 8: Signing and Secrets

16. **Signing is per-platform; keep credentials out of git:**
    - Android: release keystore in CI secrets; use Play App Signing. Keystore file NEVER in git.
    - iOS: signing certs/profiles live in Xcode / CI keychain, not the repo.
    - Desktop macOS: Developer ID cert for codesign/notarization in CI secrets (see `shipping-and-launch`).
    - Store credentials in `local.properties` (gitignored) or CI secrets. See `ci-cd-and-automation`.

```bash
# Pre-commit secret scan
grep -rn "password\|secret\|api_key\|token" --include="*.kt" --include="*.properties" \
  | grep -v "local.properties" | grep -v "Test"
```

### Step 9: Git Worktrees for Parallel Work

```bash
git worktree add ../taskapp-dark-mode feature/dark-mode
cd ../taskapp-dark-mode   # independent build, own Gradle daemon
git worktree remove ../taskapp-dark-mode
```

## Common Rationalizations

| Shortcut | Why It Fails |
|----------|-------------|
| "I'll set each platform's version by hand at release" | Platforms drift — a 1.2.0 Android and 1.1.9 iOS ship together. Fan out from one source. |
| "The feature branch will only take a week" | Week-long branches drift from main and conflict. Use feature flags in `shared`. |
| "I'll tag each platform separately" | Per-platform tags fracture the release history. One commit ships everywhere; tag once. |
| "versionCode is just an Android detail" | It must increase monotonically or Play rejects the upload. Derive it deterministically from `appVersion`. |
| "I'll squash it all at the end" | Squashed commits lose the reasoning. Atomic commits stay reviewable and revertable. |

## Red Flags

- Platform versions set independently (skew between Android/iOS/desktop/web)
- The same `appVersion` hardcoded in more than one file
- Per-platform release tags instead of one tag per release
- Long-lived feature branches (> 3 days)
- Kitchen-sink commits (> 500 lines, multiple concerns)
- Commit messages that only say "fix" or "update"
- Keystore or signing credentials committed to git
- `versionCode` not monotonically increasing
- Force-push to `main`

## Verification

- [ ] `appVersion` is declared in exactly one place (`gradle/libs.versions.toml` or `gradle.properties`)
- [ ] Android `versionName`/`versionCode`, iOS `CFBundle*`, desktop `packageVersion`, and the web deploy label all derive from that one value
- [ ] `versionCode` increases with every release
- [ ] Feature branches are short-lived (1-3 days)
- [ ] Commits are atomic and messages explain *why*
- [ ] Exactly one annotated tag per release (`vMAJOR.MINOR.PATCH`)
- [ ] Signing credentials are not in git
- [ ] `main` is green: `./gradlew :shared:allTests` passes
