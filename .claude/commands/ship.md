---
description: Run the Kotlin Multiplatform pre-launch checklist via parallel fan-out to specialist personas, then synthesize a go/no-go decision across Play Store, App Store, desktop, and web with rollback and staged rollout plan.
---

# /ship — Shipping and Launch (Kotlin Multiplatform)

Invoke the `shipping-and-launch` skill from `skills/shipping-and-launch/SKILL.md`.

`/ship` is a **fan-out orchestrator**. It runs three specialist personas in parallel against the current change, then merges their reports into a single go/no-go decision with a rollback plan and staged rollout across all four distribution channels (Play Store, App Store, desktop, web). The personas operate independently — no shared state, no ordering — which is what makes parallel execution safe and useful here.

## Phase A — Parallel fan-out

Spawn three subagents concurrently using the Agent tool. **Issue all three Agent tool calls in a single assistant turn so they execute in parallel** — sequential calls defeat the purpose of this command.

In Claude Code, each call passes `subagent_type` matching the persona's `name` field:

1. **`code-reviewer`** — Run a five-axis review (correctness, readability, architecture, security, performance) on the staged changes or recent commits. Check KMP items: code in the most common source set, no JVM-only APIs in commonMain, no needless expect/actual, Metro graph wiring, StateFlow exposure, CMP recomposition/stability. Output the standard review template.
2. **`security-auditor`** — Run a vulnerability and threat-model pass against OWASP Mobile Top 10 across Android, iOS, desktop, and web surfaces. Check secrets in commonMain constants and the wasm bundle, platform keystore usage (Android Keystore / iOS Keychain), Ktor TLS and cert pinning, deep links / universal links, dependency CVEs, R8 config for Android release. Output the standard audit report.
3. **`test-engineer`** — Analyze test coverage for the change. Identify gaps in happy path, edge cases, error paths, and concurrency (coroutines, Flow). Cover ViewModel state transitions, Repository boundaries, Room KMP DAO behavior, shared Compose UI states, and per-target test tasks. Output the standard coverage analysis.

In other harnesses without an Agent tool, invoke each persona's system prompt sequentially and treat their outputs as if returned in parallel — the merge phase still works.

Constraints (from Claude Code's subagent model):
- Subagents cannot spawn other subagents — do not let one persona delegate to another.
- Each subagent gets its own context window and returns only its report to this main session.
- If you need teammates that talk to each other instead of just reporting back, use Claude Code Agent Teams and reference these personas as teammate types (see `references/orchestration-patterns.md`).

**Persona resolution.** If you've defined your own `code-reviewer`, `security-auditor`, or `test-engineer` in `.claude/agents/` or `~/.claude/agents/`, those take precedence over this plugin's versions — `/ship` picks up your customizations automatically. This is intentional: plugin subagents sit at the bottom of Claude Code's scope priority table, so user-level definitions win by design.

### Skip-fan-out threshold

Skip Phase A and run a single-pass review **only if all of the following are true:**

- The change touches **2 files or fewer**
- The diff is **under 50 lines**
- It does **not** touch any of: auth, payments, data persistence (Room KMP/DataStore/SharedPreferences), `gradle.properties`, signing config, iOS entitlements / `Info.plist`, ProGuard/R8 rules, AndroidManifest permissions or exported components

Otherwise, default to fan-out. `/ship` is designed for production-bound changes — when the blast radius is non-trivial, run the parallel review even if the diff looks small.

## Phase B — Merge in main context

Once all three reports are back, the main agent (not a sub-persona) synthesizes them. Run these checks directly — do not delegate them back to subagents.

> **Non-KMP repos.** Sections 1–6 below are scoped to Kotlin Multiplatform app releases. When running `/ship` against a docs-only PR, a Claude Code plugin, a CLI tool, or any repo without `gradlew`/`shared`/entry-point modules, skip the Gradle commands, Vitals targets, accessibility checks, and store gates — they will fail spuriously. Apply the spirit (correctness, security, no secrets, clean release) to whatever surface the repo actually ships, and report which sections were N/A.

### 1. Code Quality
- Aggregate Critical/Important findings from `code-reviewer`
- Resolve duplicates between reviewers
- Run and verify:
  ```bash
  ./gradlew :shared:allTests                              # All-target unit tests
  ./gradlew :shared:testDebugUnitTest                     # Android unit tests
  ./gradlew :shared:iosSimulatorArm64Test                 # iOS simulator tests (macOS)
  ./gradlew :androidApp:bundleRelease                     # Android release AAB
  ./gradlew :desktopApp:packageDistributionForCurrentOS   # Desktop installer (Dmg/Msi/Deb)
  ./gradlew :webApp:wasmJsBrowserTest                     # Web tests
  ./gradlew :webApp:composeCompatibilityBrowserDistribution # Web cross-compat bundle
  ./gradlew detekt                                        # Static analysis
  ./gradlew spotlessCheck                                 # Formatting
  maestro test .maestro/                                  # E2E acceptance flows, Android AND iOS (see e2e-verification)
  ```
- Check for stragglers: no `TODO`/`FIXME` without issue links, no Kermit debug/verbose logging in production code, no hardcoded strings in UI, feature flags for incomplete features OFF.

### 2. Security
- Promote any Critical/High `security-auditor` findings to launch blockers
- Cross-reference with `code-reviewer`'s security axis
- Verify directly: no secrets in source (especially commonMain constants and the wasm bundle), Ktor enforces TLS with cert pinning, R8 enabled for Android release, no debuggable release manifest, dependency vulnerability scan clean.

### 3. Performance
Verify against `references/performance-checklist.md`.

**Android — Android Vitals targets:**
- **Cold startup** p50 < 500 ms, p90 < 1200 ms (Android Vitals "Excessive startup time" thresholds)
- **Slow rendering** (jank) < 5% of frames > 16 ms
- **Frozen frames** < 0.1% of frames > 700 ms
- **ANR rate** < 0.47% of users
- **Crash-free users** > 99.5%
- **AAB size** within budget; deltas vs. previous release flagged
- **Baseline Profiles** included and current for the `androidApp` modules touched

**iOS:**
- Crash-free rate parity with Android target
- MetricKit launch time within budget; no main-thread hangs (watchdog terminations)

**Desktop / Web:**
- Web `composeCompatibilityBrowserDistribution` bundle size within budget
- Desktop startup sanity; no new main-thread blocking patterns (network, disk I/O, large allocations) on any target

### 4. Accessibility
Verify against `references/accessibility-checklist.md`:
- TalkBack (Android) AND VoiceOver (iOS) tested on all touched screens
- Touch targets ≥ 48dp
- CMP semantics / content descriptions on all meaningful elements
- Color contrast meets WCAG AA

### 5. Infrastructure
All four distribution channels:
- **Play Store**: release keystore + key alias correct, AAB signed
- **App Store**: Xcode archive + App Store Connect signing / provisioning profiles correct
- **Desktop**: installer signing / notarization (Dmg/Msi/Deb)
- **Web**: static-hosting deploy target configured
- Environment configs correct (API URLs, feature flag remote keys)
- Crash monitoring configured for the new code paths (Crashlytics on Android, crash reporting on iOS)
- Analytics events fire as expected
- Per-platform version numbers bumped: Android `versionCode`/`versionName`, iOS `CFBundleShortVersionString`/`CFBundleVersion`, desktop `packageVersion`, web deploy version

### 6. Documentation
- Release notes written (per store where required)
- Store listings updated if user-visible features changed
- ADRs written for any significant architectural decisions
- README and `AGENTS.md` updated if developer workflow changed

## Phase C — Decision and rollback

Produce a single output:

```markdown
## Ship Decision: GO | NO-GO

### Blockers (must fix before ship)
- [Source persona: Critical finding + file:line]

### Recommended fixes (should fix before ship)
- [Source persona: Important finding + file:line]

### Acknowledged risks (shipping anyway)
- [Risk + mitigation]

### Rollback plan
- Trigger conditions: [crash-free users drop below 99%, ANR spike, new-issue crash volume, iOS crash-report spike, user-review sentiment, specific Vitals threshold breach]
- Rollback procedure (in order of fastest):
  1. **Feature flag kill switch** — disable via Remote Config (seconds; preferred for any flag-gated feature; works across all platforms)
  2. **Web / desktop redeploy** — redeploy the previous web bundle / desktop installer (minutes; instant rollback on these channels)
  3. **Halt staged rollout** — Play Console → Release → Halt rollout; App Store Connect → pause phased release (minutes; stops further percentage growth but doesn't pull the build from users who already have it)
  4. **Emergency hotfix** — branch from release tag, fast-track fix through internal/TestFlight to production at higher rollout percentage (hours)
- Recovery time objective (RTO): [target time from trigger to user-visible recovery]

### Staged rollout plan
```
Web + Desktop (instant deploy, instant rollback) →
Play Store staged: 1% → 5% → 25% → 50% → 100% →
App Store phased release: 7-day automatic curve (pauseable)
```

Ship web and desktop first (instant deploy, instant rollback), then begin the Play Store staged rollout, then start the App Store phased release. After each expansion, monitor for at least 24h (longer for low-traffic apps) before promoting:
- Crash reporting: any new crash signatures on Android or iOS? Spike in existing ones?
- Android Vitals: ANR rate, slow rendering, startup time
- Store reviews: negative sentiment spike?
- Feature analytics: expected events firing at expected volumes?

Halt criteria — promote only when ALL of: crash-free users > 99.5% on Android and iOS, ANR rate within threshold, no new P0/P1 crash issues, no review sentiment regression.

### Specialist reports (full)
- [code-reviewer report]
- [security-auditor report]
- [test-engineer report]
```

## Rules

1. The three Phase A personas run in parallel — never sequentially.
2. Personas do not call each other. The main agent merges in Phase B.
3. The rollback plan is mandatory before any GO decision.
4. If any persona returns a Critical finding, the default verdict is NO-GO unless the user explicitly accepts the risk.
5. Skip the fan-out only when all three skip-fan-out conditions hold (≤2 files, <50 lines, no sensitive surfaces). Otherwise default to fan-out — `/ship` is designed for production-bound KMP releases where blast radius justifies parallel review.
