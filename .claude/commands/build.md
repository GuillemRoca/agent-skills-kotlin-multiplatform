# /build — Incremental Implementation with TDD

Build the next task incrementally using test-driven development.

## Modes

- `/build` — interactive: work task by task, checking in between tasks.
- `/build auto` — autonomous: plan the remaining tasks first, present the full
  plan for ONE approval, then implement every task without further check-ins.
  Stop and ask only on: failing tests you cannot fix within the task's scope,
  a needed deviation from the approved plan, or destructive operations.
  Everything else follows the same per-task cycle below.

## Instructions

Load and follow these skills:
- `incremental-implementation` from `skills/incremental-implementation/SKILL.md`
- `test-driven-development` from `skills/test-driven-development/SKILL.md`

### Per-Task Cycle

For each task in `tasks/todo.md`:

1. **Review** the task's acceptance criteria
2. **Gather examples** — find similar patterns in the codebase
3. **Write a failing test** (RED) — in `commonTest` with kotlin.test, describing the expected behavior
4. **Write minimal code** to pass the test (GREEN) — in commonMain, or a platform source set only when genuinely platform-specific
5. **Run `./gradlew :shared:jvmTest`** — the fastest feedback loop; verify all tests pass
6. **Compile a non-JVM target early** — `./gradlew :shared:compileKotlinIosSimulatorArm64` catches JVM-only APIs leaking into commonMain before they accumulate
7. **Run `./gradlew :androidApp:assembleDebug`** — verify the app build succeeds
8. **Verify the slice end-to-end** when it is user-visible — run its Maestro flow per `e2e-verification` (`maestro test .maestro/<slice>.yaml`) on Android and iOS
9. **Commit** the increment
10. **Repeat** for the next acceptance criterion

### Error Handling

If a test fails unexpectedly or the build breaks:
- Load `debugging-and-error-recovery` skill
- Follow the six-step triage process
- Fix the root cause before proceeding

### Rules

- Each increment leaves the codebase in a working state on every configured target
- Never skip the build check or the non-JVM compile check between increments
- Commits are small and focused (~100 lines)
- Incomplete features go behind feature flags
- Don't mix refactoring with feature work
