---
name: shipping-and-launch
description: >-
  Use when preparing to release a Kotlin Multiplatform + Compose Multiplatform
  app across Google Play, the Apple App Store, desktop installers, and the web.
  Covers the pre-launch checklist, per-storefront staged rollout, feature-flag
  kill switches in shared code, rollback, and coordinated multi-platform launch.
---

# Shipping and Launch

## Overview

Ship with confidence: verify production readiness, stage the rollout per storefront, monitor, and keep a rollback plan ready. A KMP app ships to four storefronts with very different mechanics — Google Play staged rollout, App Store phased release, desktop direct-download installers, and instant-redeploy web — from one shared codebase. Coordinate them so one kill switch in `shared` covers every platform.

## When to Use

- Before any release to Google Play (internal, alpha, beta, production)
- Before submitting an iOS build to TestFlight or the App Store
- Before publishing desktop installers (Dmg/Msi/Deb)
- Before deploying the web bundle to static hosting
- When coordinating a multi-platform launch
- After a hotfix that needs expedited release

**Skip when:** Not releasing (local development only).

## Core Process

### Step 1: Pre-Launch Checklist

1. **Code quality (module-qualified Gradle tasks):**

```bash
./gradlew :shared:allTests          # all KMP targets (iOS sim needs macOS)
./gradlew :shared:jvmTest           # fastest loop
./gradlew :androidApp:bundleRelease # release AAB
./gradlew :desktopApp:packageDistributionForCurrentOS
./gradlew :webApp:composeCompatibilityBrowserDistribution
```

2. **Manual code checks:**
   - [ ] No `TODO`/`FIXME` without issue references
   - [ ] No debug logging left in production paths (gate Kermit severity)
   - [ ] No hardcoded UI strings — use `stringResource(Res.string.x)`
   - [ ] Feature flags for incomplete features are OFF (see Step 6)

3. **Security** (see `security-and-hardening`):
   - [ ] No secrets in source (`commonMain` or platform source sets)
   - [ ] Ktor client enforces HTTPS; certificate pinning where required
   - [ ] R8/ProGuard enabled for the Android release build
   - [ ] Dependency vulnerability scan clean

4. **Performance** (see `performance-optimization`):
   - [ ] Startup and jank targets met per platform
   - [ ] Artifact sizes within budget (AAB, IPA, installers, wasm bundle)
   - [ ] Baseline Profiles included in `androidApp`
   - [ ] No main-thread blocking in shared coroutine code

5. **Accessibility** (see `multiplatform-accessibility`):
   - [ ] Screen readers tested (TalkBack, VoiceOver, desktop, web)
   - [ ] Touch/click targets and contrast meet standards
   - [ ] `contentDescription`/semantics on meaningful composables

6. **Infrastructure & observability** (see `observability-and-instrumentation`):
   - [ ] Signing correct for every platform (see `git-workflow-and-versioning`)
   - [ ] Crash/error monitoring wired for each target
   - [ ] Analytics events verified
   - [ ] Release notes written

### Step 2: Google Play (Android)

7. **Track progression and staged rollout:**

```
Internal → Closed Alpha → Open Beta → Production (staged)
Production %: 1% -> 5% -> 25% -> 50% -> 100%
Day 0: 1%   (monitor crashes, ANR, feedback)
Day 1: 5%   (check Android Vitals)
Day 3: 25%  (review ratings)
Day 5: 50%
Day 7: 100%
```

8. **Decision thresholds:**

| Metric | Action |
|--------|--------|
| Crash rate > 2x previous version | **Halt rollout**, investigate |
| ANR rate > 0.47% | **Halt rollout**, investigate |
| Negative reviews spike > 2x | **Halt rollout**, investigate |
| New error rate > 0.1% | Investigate, consider halt |
| Startup regression > 20% | Investigate, consider halt |

9. **Requirements:** target the current required API level, upload the AAB from `:androidApp:bundleRelease`, upload the R8 mapping file, complete Data Safety and privacy policy.

### Step 3: Apple App Store (iOS)

10. **Progression — build lead time in, not just percentages:**

```
Xcode archive (from iosApp) -> App Store Connect
  -> TestFlight internal testers (instant)
    -> TestFlight external testers (needs Beta App Review, ~1 day)
      -> Submit for App Review  (budget 1-3 days; expedited review is exceptional)
        -> Phased release (automatic, 7 days):
           1% -> 2% -> 5% -> 10% -> 20% -> 50% -> 100%
```

11. **Rollback reality:** you cannot un-ship a build. During phased release you can **pause** distribution (holds the current percentage) or expedite an approved hotfix build. Plan the review lead time into the launch calendar; the kill switch (Step 6) is your fastest lever.

### Step 4: Desktop Distribution

12. **Package per OS — each installer must be built on its own OS (or a CI matrix):**

```bash
# macOS runner -> Dmg
./gradlew :desktopApp:packageDistributionForCurrentOS
# Windows runner -> Msi
./gradlew :desktopApp:packageDistributionForCurrentOS
# Linux runner -> Deb
./gradlew :desktopApp:packageDistributionForCurrentOS
```

13. **macOS signing and notarization** (unsigned apps are blocked by Gatekeeper):

```bash
# Sign with a Developer ID Application certificate, then notarize
codesign --deep --force --options runtime \
  --sign "Developer ID Application: Example (TEAMID)" TaskApp.app
xcrun notarytool submit TaskApp.dmg \
  --apple-id "$APPLE_ID" --team-id "$TEAM_ID" --password "$APP_PW" --wait
xcrun stapler staple TaskApp.dmg
```

14. **Rollback:** desktop has no store gate — re-point the download link (and the auto-update feed, if any) to the previous signed installer. Publish SHA-256 checksums beside each artifact.

### Step 5: Web

15. **Build and deploy the cross-compatible bundle to static hosting:**

```bash
./gradlew :webApp:composeCompatibilityBrowserDistribution
# Output (js + wasmJs, runtime-selected) -> deploy to static hosting / CDN.
# Use immutable, content-hashed asset names for cache-busting.
```

16. **Rollback is instant:** redeploy the previous bundle (keep the last known-good build addressable). Web is the safest surface, so it usually leads the launch order (Step 7).

### Step 6: Feature Flags in Shared Code (one kill switch, all platforms)

17. **Put the flag interface in `commonMain` and wire it through DI so a single remote toggle disables a feature on Android, iOS, desktop, and web at once:**

```kotlin
// commonMain
enum class Flag { NEW_SYNC, SOCIAL_SHARE }

interface FeatureFlags {
    fun isEnabled(flag: Flag): Boolean
}

// consumed by shared Compose UI
@Composable
fun TaskScreen(flags: FeatureFlags) {
    if (flags.isEnabled(Flag.NEW_SYNC)) {
        NewSyncBanner()
    }
}
```

```kotlin
// Remote-backed implementation, bound via Metro for every target.
@ContributesBinding(AppScope::class)
@SingleIn(AppScope::class)
@Inject
class RemoteFeatureFlags(private val config: RemoteConfigClient) : FeatureFlags {
    override fun isEnabled(flag: Flag): Boolean =
        config.boolean(flag.name, default = false)
}
```

18. Flip a broken feature off remotely — every platform hides it on next config fetch, no new build required. This is the fastest cross-platform response and does not depend on any store.

### Step 7: Coordinated Multi-Platform Launch

19. **Ship in risk order — safest and most-reversible first:**

```
1. Web + Desktop      (instant redeploy / re-point download; lowest blast radius)
2. Google Play        (staged rollout 1% -> 100%, halt-able)
3. Apple App Store    (phased release; longest lead time, least reversible)
```

20. **Keep the shared backend compatible across app versions.** Users update at different speeds and iOS lags behind web, so the backend must serve version N and N-1 of every platform simultaneously. Version and evolve APIs additively (see `api-and-interface-design`); never ship a backend change that only the newest client tolerates.

21. **One kill switch covers all.** Because flags live in `shared` (Step 6), a single remote toggle neutralizes a bad feature everywhere while you decide whether to halt Play, pause the App Store phase, or redeploy web.

### Step 8: Post-Launch Monitoring

22. **First 24 hours** (see `observability-and-instrumentation`): new crash types per platform, ANR rate, startup/latency, store reviews, critical flows end-to-end.

23. **First 7 days:** rollout expanded on schedule, App Store phase progressing, feature-flag cleanup scheduled, retro if issues occurred.

## Common Rationalizations

| Shortcut | Why It Fails |
|----------|-------------|
| "Ship all four platforms at 100% together" | Blast radius is maximal and App Store can't be un-shipped. Stage per storefront, in risk order. |
| "The backend only needs to support the newest app" | iOS users lag for weeks; an N-only backend breaks every not-yet-updated client. Support N and N-1. |
| "We'll add a kill switch later" | Later is during the incident. Put flags in `shared` before launch so one toggle covers all platforms. |
| "Desktop doesn't need signing, it's a direct download" | Unsigned/un-notarized apps are blocked by Gatekeeper and SmartScreen. Sign and notarize. |
| "Skip staged rollout, tests pass" | Tests miss device-, OS-, and locale-specific issues. Stage and monitor. |

## Red Flags

- All platforms pushed to 100% simultaneously
- No rollback plan for at least one storefront
- Feature flags living in a platform module instead of `shared`
- Backend change that only the newest client tolerates
- Desktop installers unsigned or un-notarized
- Web bundle deployed without a retained previous build to roll back to
- No monitoring configured before launch
- Launching iOS without accounting for App Review lead time
- Launching on Friday with no weekend coverage

## Verification

- [ ] Pre-launch checklist complete (quality, security, performance, accessibility)
- [ ] Each artifact signed correctly (AAB, IPA, notarized desktop installers)
- [ ] Google Play staged-rollout plan and halt thresholds defined
- [ ] App Store review lead time built into the schedule; phased release enabled
- [ ] Desktop installers built per-OS, signed/notarized, checksummed
- [ ] Web previous build retained for instant redeploy rollback
- [ ] Feature-flag kill switch lives in `commonMain` and is wired via DI
- [ ] Shared backend verified compatible with app versions N and N-1
- [ ] Launch order is web/desktop -> Play -> App Store
- [ ] `./gradlew :shared:allTests` passes before cutting the release
