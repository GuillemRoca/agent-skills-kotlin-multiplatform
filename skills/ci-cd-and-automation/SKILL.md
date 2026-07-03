---
name: ci-cd-and-automation
description: >-
  Use when setting up or improving CI/CD pipelines for a Kotlin Multiplatform +
  Compose Multiplatform project. Covers sequential quality gates split across
  Linux and macOS runners, Gradle and configuration caching, fail-fast
  ordering, and per-platform release lanes for Play, TestFlight, desktop
  installers, and static web hosting.
---

# CI/CD and Automation

## Overview

Shift left: catch problems early through automated checks. A KMP pipeline has one twist single-platform pipelines lack: targets need different machines. Linux runners build and test Android, JVM, and js/wasm; iOS requires a macOS runner with Xcode. Structure CI as sequential quality gates — fast, cheap checks on Linux first, the expensive macOS gate only after they pass — so feedback arrives in minutes and macOS minutes are never burned on code that fails lint.

## When to Use

- Setting up CI for a new KMP project
- Adding quality gates or an iOS gate to an existing pipeline
- A target (usually iOS or wasm) keeps breaking on main because no PR compiles it
- Automating releases for any target: Play, TestFlight, desktop installers, web hosting
- Pipeline exceeds 15 minutes or macOS runner costs are climbing

**Skip when:** the project has a single Android target and no shared module — a plain Android pipeline is simpler. Skip the release lanes (Step 4 only) while the app has no external users yet.

## Core Process

### Step 1: Order Gates Fastest to Slowest

Run everything except iOS on `ubuntu-latest`. Order gates so cheap failures kill the run before expensive work starts:

1. Static analysis: `detekt` (plus Android Lint if configured) — ~1 min
2. Fastest tests: `:shared:jvmTest` — runs `commonTest` at JVM speed
3. Android unit tests: `:shared:testDebugUnitTest`
4. Android app compiles: `:androidApp:assembleDebug`
5. Web tests in a headless browser: `:webApp:wasmJsBrowserTest`

No gate skipping — fix the code, not the gate:

```kotlin
// BAD: suppressing the rule to get past CI
@Suppress("MagicNumber")
val timeout = 5000

// GOOD: fix what the rule is pointing at
private const val NETWORK_TIMEOUT_MS = 5_000L
```

### Step 2: The Workflow — Linux Checks Plus a Gated iOS Job

```yaml
# .github/workflows/kmp-ci.yml
name: KMP CI

on:
  pull_request:
    branches: [main]
  push:
    branches: [main]

concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true

jobs:
  checks:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-java@v4
        with:
          distribution: temurin
          java-version: 17

      - name: Setup Gradle (dependency + build caching)
        uses: gradle/actions/setup-gradle@v4
        with:
          cache-read-only: ${{ github.event_name == 'pull_request' }}

      # Gate 1: static analysis (~1 min)
      - name: Detekt
        run: ./gradlew detekt

      # Gate 2: commonTest at JVM speed (~2 min)
      - name: JVM tests
        run: ./gradlew :shared:jvmTest

      # Gate 3: Android unit tests (~3 min)
      - name: Android unit tests
        run: ./gradlew :shared:testDebugUnitTest

      # Gate 4: Android app compiles (~3 min)
      - name: Assemble Android debug
        run: ./gradlew :androidApp:assembleDebug

      # Gate 5: web tests in a headless browser (~4 min)
      - name: Wasm browser tests
        run: ./gradlew :webApp:wasmJsBrowserTest

      - name: Upload test reports
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: test-reports
          path: '**/build/reports/tests/'

  ios:
    needs: checks   # macOS minutes are expensive; only run on code that passed Linux gates
    runs-on: macos-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-java@v4
        with:
          distribution: temurin
          java-version: 17
      - uses: gradle/actions/setup-gradle@v4
        with:
          cache-read-only: ${{ github.event_name == 'pull_request' }}

      - name: Select Xcode
        run: sudo xcode-select -s /Applications/Xcode_16.4.app

      # Gate 6: Kotlin/Native tests on the simulator
      - name: iOS simulator tests
        run: ./gradlew :shared:iosSimulatorArm64Test

      # Gate 7: the framework Xcode embeds actually links
      - name: Link iOS framework
        run: ./gradlew :shared:linkDebugFrameworkIosSimulatorArm64
```

macOS runners cost roughly 10x Linux minutes on GitHub-hosted runners. Keep the iOS job honest but cheap:

- `needs: checks` — never start macOS work on code that fails lint or JVM tests.
- Trigger the workflow only for PRs targeting `main` (as above). If cost is still too high, add a `paths` filter (`shared/**`, `iosApp/**`) to the `ios` job — but never remove it entirely: an iOS target nobody compiles is a broken release waiting.
- Do not move lint or JVM-test work onto the macOS runner; it does only what Linux cannot: Kotlin/Native compilation and simulator tests.

### Step 3: Gradle Caching and the Configuration Cache

`gradle/actions/setup-gradle@v4` caches `~/.gradle/caches`, the wrapper, and build outputs automatically; `cache-read-only` on pull requests restricts cache writes to trusted branches.

Enable build optimizations in the repo — the configuration cache matters more in KMP than in single-platform projects because configuring many targets is expensive:

```properties
# gradle.properties
org.gradle.caching=true
org.gradle.parallel=true
org.gradle.configuration-cache=true
org.gradle.jvmargs=-Xmx4g -XX:+UseParallelGC
```

If a plugin breaks the configuration cache, update or replace the plugin; do not switch the cache off for everyone.

### Step 4: Per-Platform Release Lanes

Gate deployment on green checks and an explicit trigger (a version tag). Each target has its own lane and its own versioning scheme — Android `versionCode`/`versionName`, iOS `CFBundleShortVersionString`/`CFBundleVersion`, desktop `packageVersion`, web deploy-versioned; see `git-workflow-and-versioning` and `shipping-and-launch`.

```yaml
# .github/workflows/kmp-release.yml
name: KMP Release
on:
  push:
    tags: ['v*']

jobs:
  android:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-java@v4
        with: { distribution: temurin, java-version: 17 }
      - uses: gradle/actions/setup-gradle@v4
      - name: Decode signing keystore
        run: echo "${{ secrets.RELEASE_KEYSTORE_B64 }}" | base64 -d > androidApp/release.keystore
      - name: Build app bundle
        run: ./gradlew :androidApp:bundleRelease
      - name: Upload to Play (internal track first)
        uses: r0adkll/upload-google-play@v1
        with:
          serviceAccountJsonPlainText: ${{ secrets.PLAY_STORE_CREDENTIALS }}
          packageName: com.example.app
          releaseFiles: androidApp/build/outputs/bundle/release/androidApp-release.aab
          track: internal

  ios:
    runs-on: macos-latest
    steps:
      - uses: actions/checkout@v4
      - uses: gradle/actions/setup-gradle@v4
      - name: Archive (Xcode build phase embeds the shared framework)
        run: |
          xcodebuild archive \
            -project iosApp/iosApp.xcodeproj -scheme iosApp \
            -archivePath build/iosApp.xcarchive \
            -destination 'generic/platform=iOS'
      - name: Export and upload to TestFlight
        run: |
          xcodebuild -exportArchive -archivePath build/iosApp.xcarchive \
            -exportOptionsPlist iosApp/ExportOptions.plist -exportPath build/export
          xcrun altool --upload-app --type ios --file build/export/iosApp.ipa \
            --apiKey "${{ secrets.ASC_KEY_ID }}" --apiIssuer "${{ secrets.ASC_ISSUER_ID }}"

  desktop:
    strategy:
      matrix:
        os: [ubuntu-latest, macos-latest, windows-latest]   # Deb / Dmg / Msi
    runs-on: ${{ matrix.os }}
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-java@v4
        with: { distribution: temurin, java-version: 17 }
      - uses: gradle/actions/setup-gradle@v4
      - name: Package installer for this OS
        run: ./gradlew :desktopApp:packageDistributionForCurrentOS
      - uses: actions/upload-artifact@v4
        with:
          name: desktop-${{ matrix.os }}
          path: desktopApp/build/compose/binaries/**

  web:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-java@v4
        with: { distribution: temurin, java-version: 17 }
      - uses: gradle/actions/setup-gradle@v4
      - name: Build cross-compatible bundle (wasm with js fallback)
        run: ./gradlew :webApp:composeCompatibilityBrowserDistribution
      - name: Publish to static hosting
        uses: actions/upload-pages-artifact@v3
        with:
          path: webApp/build/dist/composeWebCompatibility/productionExecutable  # adjust to your dist output
```

### Step 5: Feedback Loop on Failure

Read the failing task name — in KMP it names the target and therefore where to look:

| Failing task | Look at |
|---|---|
| `:shared:jvmTest` | `commonMain`/`commonTest` logic, `jvmMain` actuals |
| `:shared:testDebugUnitTest` | `androidMain` actuals, Android-only dependencies |
| `:shared:iosSimulatorArm64Test` | `iosMain` actuals, Kotlin/Native-only behavior, accidental `java.*` in common code |
| `:webApp:wasmJsBrowserTest` | `wasmJsMain` actuals, browser API use, wasm restrictions |

Then: fix the specific issue (never disable the gate), re-run, and if the fix breaks another target, revert and investigate — see `debugging-and-error-recovery` and `multiplatform-testing`.

## Common Rationalizations

| Shortcut | Why It Fails |
|----------|-------------|
| "iOS CI is too expensive; developers have Macs" | Local Xcode versions drift and nobody runs simulator tests before merging. One gated macOS job per PR is cheaper than a week of broken main. |
| "`jvmTest` passed, so the shared code works" | commonTest on the JVM misses Kotlin/Native and wasm differences: no `java.*`, different concurrency, browser API limits. CI must exercise the other targets. |
| "Web is Beta, skip its gates" | Beta describes the tooling maturity, not your users. A broken wasm bundle ships all the same; the headless browser test costs minutes. |
| "We'll wire release lanes at launch" | Launch week is the worst time to debug signing, provisioning, and store credentials. Ship to internal tracks from the first sprint. |
| "The configuration cache breaks plugin X, turn it off" | You pay the multi-target configuration cost on every build forever. Update or replace the plugin instead. |

## Red Flags

- No macOS job anywhere — the iOS target is never compiled before merge
- Unqualified `./gradlew test` or `./gradlew build` in CI instead of module-qualified tasks
- Release artifacts built and uploaded from a developer machine
- Gates disabled to merge (`-x detekt`, suppressions added the same day as the PR)
- macOS runners doing lint or JVM-test work a Linux runner could do
- Keystores or store credentials committed to the repo instead of CI secrets
- Cold Gradle on every run: no `gradle/actions/setup-gradle@v4`, no configuration cache

## Verification

- [ ] CI runs on every PR and push to main
- [ ] Linux gates ordered fastest to slowest: detekt, `:shared:jvmTest`, `:shared:testDebugUnitTest`, `:androidApp:assembleDebug`, `:webApp:wasmJsBrowserTest`
- [ ] iOS job on `macos-latest` runs `:shared:iosSimulatorArm64Test` and links the framework, gated with `needs: checks`
- [ ] `gradle/actions/setup-gradle@v4` present in every job; configuration cache enabled in `gradle.properties`
- [ ] A release lane exists for every shipping target, triggered by tags only
- [ ] All credentials live in CI secrets, none in the repo
- [ ] No gate suppressed or skipped in the merged diff
- [ ] The Linux gate sequence passes locally before pushing:

```bash
./gradlew detekt :shared:jvmTest :shared:testDebugUnitTest :androidApp:assembleDebug
```
