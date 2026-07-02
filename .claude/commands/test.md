# /test — Test-Driven Development

Write tests before implementation using Red-Green-Refactor.

## Instructions

Load and follow these skills:
- `test-driven-development` from `skills/test-driven-development/SKILL.md`
- `multiplatform-testing` from `skills/multiplatform-testing/SKILL.md` (for per-target and device tests)
- `e2e-verification` from `skills/e2e-verification/SKILL.md` (for end-to-end acceptance flows)

### New Feature Testing

1. **Write failing test** in `commonTest` describing the expected behavior
2. **Run test** — confirm it FAILS (RED)
3. **Write minimal code** to make it pass (GREEN)
4. **Run full suite** — `./gradlew :shared:jvmTest` (fast loop), `:shared:allTests` before done
5. **Refactor** while tests stay green
6. **Repeat** for next behavior

### Bug Fix Testing (Prove-It Pattern)

1. **Write a test** that demonstrates the bug
2. **Run test** — confirm it FAILS (proves bug exists)
3. **Fix the code**
4. **Run test** — confirm it PASSES (proves fix works)
5. **Run full suite** — `./gradlew :shared:allTests` (no regressions)

### Test Types

| Type | Framework | Command |
|------|-----------|---------|
| Common unit tests | kotlin.test + fakes + Turbine | `./gradlew :shared:jvmTest` (fast) / `:shared:allTests` (all targets) |
| Android unit tests | kotlin.test | `./gradlew :shared:testDebugUnitTest` |
| iOS simulator tests | kotlin.test | `./gradlew :shared:iosSimulatorArm64Test` (macOS) |
| Shared Compose UI tests | `runComposeUiTest` | runs via `:shared:jvmTest` / `:shared:allTests` |
| Web tests | kotlin.test | `./gradlew :webApp:wasmJsBrowserTest` |
| E2E acceptance flows | Maestro | `maestro test .maestro/` (Android AND iOS) |

### Rules

- Tests BEFORE implementation (always)
- Every test must be seen failing before passing
- No `Thread.sleep` — use `runTest` with `advanceUntilIdle()` or `waitUntil`
- Fake at boundaries only (DAO, API) — hand-written fakes in commonTest, no MockK in common code
- Descriptive test names that read like behavior specs
