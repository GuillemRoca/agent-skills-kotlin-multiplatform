---
name: e2e-verification
description: >-
  Use when a feature slice needs proof it works end-to-end on the real app
  targets, or when closing the implement-run-assert loop for any user-facing
  change. One Maestro YAML flow runs the same acceptance checks on both the
  Android emulator and the iOS simulator; desktop and web get lighter-weight
  verification.
---

# End-to-End Verification (Maestro)

## Overview

"It compiles and `:shared:allTests` passes" is not proof that a feature works. Every feature slice gets a Maestro flow — a YAML file of user actions and assertions derived from the acceptance criteria — that runs black-box against the installed apps. Because the UI is shared Compose Multiplatform, one flow verifies both Android and iOS: same steps, same user-visible strings, two platforms. The flow is committed next to the code, so the acceptance criterion stays executable forever.

## When to Use

- Completing any feature slice with a user-visible flow (see `incremental-implementation`)
- Turning acceptance criteria into executable checks (see `spec-driven-development`)
- Verifying a bug fix changes user-facing behavior, not just the unit under test
- Regression-protecting critical journeys (login, checkout, sync) in CI
- Smoke-testing release candidates on both mobile platforms (see `shipping-and-launch`)

**Skip when:** The change has no runtime UI surface (pure data-layer refactor, build config). Logic permutations belong in ViewModel and repository tests (`test-driven-development`); single-screen state coverage belongs in `runComposeUiTest` (`multiplatform-testing`).

## Core Process

### Step 1: Install and Probe

Maestro is a single binary (Java 17+) that drives Android over adb and iOS over the simulator:

```bash
# Pin the version so CI and local runs agree
export MAESTRO_VERSION=2.6.1
curl -fsSL "https://get.maestro.mobile.dev" | bash
maestro --version                 # probe availability

adb devices                       # Android emulator visible?
xcrun simctl list devices booted  # iOS simulator booted? (macOS only)
```

If Maestro is unavailable, say so explicitly and fall back to `runComposeUiTest` coverage plus platform screenshots (`adb exec-out screencap` / `xcrun simctl io booted screenshot`) — never silently skip E2E verification.

### Step 2: Align App Identifiers Across Platforms

Maestro selects the app by `appId`: the Android `applicationId` on Android, the bundle identifier on iOS. Make them identical (e.g. `com.example.tasks` in `androidApp/build.gradle.kts` and the Xcode target) so one flow drives both. If they must differ, parametrize:

```yaml
appId: ${APP_ID}
```

```bash
maestro test -e APP_ID=com.example.tasks.android .maestro/
maestro test -e APP_ID=com.example.tasks.ios .maestro/
```

### Step 3: Write the Flow From Acceptance Criteria — Before Implementing

Translate each acceptance criterion into a flow under `.maestro/`, named after the slice:

```yaml
# .maestro/create-task.yaml
# Acceptance: "User can create a task and sees it in the list"
appId: com.example.tasks   # android applicationId == iOS bundle id
---
- launchApp:
    clearState: true
- tapOn: "Add task"
- inputText: "Buy groceries"
- tapOn: "Save"
- assertVisible: "Buy groceries"
```

Flow rules:

- One flow per acceptance criterion; compose shared steps with `runFlow`:

  ```yaml
  - runFlow: subflows/login.yaml
  ```
- `clearState: true` on `launchApp` for deterministic starts
- Assert on user-visible text — with shared Compose UI the copy comes from `composeResources`, so the same assertion holds on both platforms
- Wrap genuinely async steps in `retry` blocks instead of sprinkling waits:

  ```yaml
  - retry:
      maxRetries: 3
      commands:
        - tapOn: "Sync"
        - assertVisible: "Synced"
  ```
- Isolate genuine platform divergence (system dialogs, permissions) behind conditions:

  ```yaml
  - runFlow:
      when:
        platform: iOS
      file: subflows/ios-allow-notifications.yaml
  ```
- Deterministic selectors first; `assertWithAI` only where a selector cannot express the assertion (visual patterns, third-party content) — AI assertions cost more and can flake

### Step 4: Run the Loop on Both Mobile Targets

Implement → build/install → run → fix → re-run, on each platform:

```bash
# Android (emulator running)
./gradlew :androidApp:installDebug
maestro test .maestro/create-task.yaml

# iOS (macOS, simulator booted)
xcodebuild -project iosApp/iosApp.xcodeproj -scheme iosApp \
  -configuration Debug -sdk iphonesimulator -derivedDataPath build/ios build
xcrun simctl install booted build/ios/Build/Products/Debug-iphonesimulator/iosApp.app
maestro test .maestro/create-task.yaml
```

With both an emulator and a simulator connected, target one explicitly: `maestro --device <id> test .maestro/`. A slice is done when the flow passes on **both** platforms — shared UI does not mean shared integration; the Xcode framework embed, the Android manifest, keyboards, and safe areas all live in the gaps. Capture evidence for the PR:

```bash
maestro record .maestro/create-task.yaml   # video of the run
```

### Step 5: Verify Desktop and Web (Lighter Weight)

Maestro does not drive desktop or web; match the effort to the risk:

- **Desktop**: the same `App()` is already covered by `runComposeUiTest` integration tests executing on the desktop JVM (see `multiplatform-testing`). For a release-shaped check, run the app via the `desktopApp [hot]` run configuration or `./gradlew :desktopApp:run` and walk the acceptance flow manually.
- **Web (Beta)**: run `./gradlew :webApp:wasmJsBrowserDevelopmentRun` (run config `webApp[wasmJs]`), walk the flow in the browser, and watch the devtools console for errors and failed requests. Keep this toolchain-neutral — no browser-automation framework is mandated. State in the PR exactly what was walked and observed.

### Step 6: Wire Into CI

Two jobs, same flows (see `ci-cd-and-automation`):

```yaml
e2e-android:
  runs-on: ubuntu-latest
  steps:
    - uses: actions/checkout@v4
    - uses: gradle/actions/setup-gradle@v4
    - name: Install Maestro
      run: curl -fsSL "https://get.maestro.mobile.dev" | bash
    - name: E2E flows (Android)
      uses: reactivecircus/android-emulator-runner@v2
      with:
        api-level: 36
        arch: x86_64
        script: |
          ./gradlew :androidApp:installDebug
          $HOME/.maestro/bin/maestro test .maestro/

e2e-ios:
  runs-on: macos-15
  steps:
    - uses: actions/checkout@v4
    - uses: gradle/actions/setup-gradle@v4
    - name: Install Maestro
      run: curl -fsSL "https://get.maestro.mobile.dev" | bash
    - name: Build, install, run flows (iOS)
      run: |
        xcrun simctl boot "iPhone 16"
        xcodebuild -project iosApp/iosApp.xcodeproj -scheme iosApp \
          -configuration Debug -sdk iphonesimulator -derivedDataPath build/ios build
        xcrun simctl install booted build/ios/Build/Products/Debug-iphonesimulator/iosApp.app
        $HOME/.maestro/bin/maestro test .maestro/
```

`maestro test .maestro/` runs every committed flow — the acceptance criteria of all shipped slices become the cross-platform regression suite. For device-farm scale, `maestro cloud` runs the same flows on hosted devices.

## Common Rationalizations

| Shortcut | Why It Fails |
|----------|-------------|
| "The unit tests pass, the feature works" | Unit tests prove the pieces in isolation. The user experiences the assembled flow — DI wiring, navigation, the iOS framework embed, and manifest bugs live in the gaps. |
| "It passes on Android; iOS is the same Compose code" | Shared UI is not shared integration. Keyboards, safe areas, permission dialogs, lifecycle, and the Xcode embed differ. Running the same flow on iOS costs one command. |
| "I'll write the flow after the feature is done" | Written after, the flow describes what you built, not what was asked. Written first, it is the acceptance criterion made executable. |
| "Maestro isn't installed, I'll just say it works" | "It works" without a run is an assertion, not evidence. State the tool is unavailable and verify another way — never claim an unrun check. |
| "I'll use assertWithAI everywhere, it's easier" | AI assertions are slower, cost money, and can flake on ambiguity. Selectors on shared copy are free and deterministic — AI is the escape hatch, not the default. |
| "E2E tests are flaky, not worth it" | Flakiness comes from timing hacks and shared state. `clearState`, `retry` blocks, and user-visible assertions make flows boringly stable. |

## Red Flags

- A completed feature slice with no flow under `.maestro/`
- Flows run against only one mobile platform ("iOS later")
- Flows that only `launchApp` and assert nothing
- Sleep-style waits instead of `retry` blocks
- Assertions on platform-specific strings instead of shared `composeResources` copy
- `assertWithAI` used where `assertVisible` would do
- Flows passing locally but absent from the CI emulator and simulator jobs
- "Verified manually" in a PR with no recorded run or committed flow

## Verification

- [ ] Every acceptance criterion of the slice has a flow in `.maestro/`
- [ ] Flows start from `clearState: true` (or document why not)
- [ ] Assertions target user-visible shared copy; `assertWithAI` only where unavoidable
- [ ] Android and iOS app identifiers aligned (or the flow is parametrized)
- [ ] Desktop/web verification stated where those targets ship (jvmTest UI tests, manual wasm walkthrough)
- [ ] CI runs `maestro test .maestro/` on both the emulator and the simulator jobs
- [ ] `maestro test .maestro/<slice>.yaml` passes against the Android emulator (paste output)
- [ ] `maestro test .maestro/<slice>.yaml` passes against the iOS simulator (paste output)
