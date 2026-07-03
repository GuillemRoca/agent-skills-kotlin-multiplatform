---
name: spec-driven-development
description: >-
  Use when starting new Kotlin Multiplatform projects, features, or changes
  with unclear requirements. Guides writing a structured spec (SPEC.md) that
  becomes the shared source of truth before any code is written.
---

# Spec-Driven Development

## Overview

Write a structured specification before writing code. The spec becomes the shared source of truth — a development contract that prevents misalignment, scope creep, and wasted effort. Every implementation decision traces back to the spec.

## When to Use

- Starting a new KMP project or module
- Adding a feature that spans multiple files, layers, or targets
- Requirements are ambiguous or come from multiple stakeholders
- Before writing a `tasks/plan.md`

**Skip when:** Single-line fixes or changes that are unambiguous and self-contained.

## Core Process

### Phase 1: Specify

1. **Ask clarifying questions** before writing anything:
   - What problem does this solve? Who is the user?
   - Which features are in scope? Which are explicitly out?
   - Which platforms are in scope? Shared Compose UI everywhere, or native UI on some targets?
   - What is the tech stack? (Kotlin/CMP versions, DI framework, persistence, iOS deployment target vs minSdk)
   - What are the boundaries? (offline support? accessibility? which browsers for web?)

2. **Write SPEC.md** with these sections:

```markdown
# Feature Name — Specification

## Objective
What we're building and why. One paragraph.

## Targets
Which platforms this feature ships to: e.g. Android + iOS + desktop in v1;
web (js/wasm, Beta) in v2. Note any target where UI stays native.

## Commands
Key Gradle tasks and how to use them:
- `./gradlew :shared:allTests` — run shared tests on every target
- `./gradlew :shared:jvmTest` — fastest unit-test loop
- `./gradlew :androidApp:assembleDebug` — build Android debug APK
- `./gradlew :desktopApp:run` — launch the desktop app
- `./gradlew :webApp:composeCompatibilityBrowserDistribution` — cross-compatible web bundle
- iOS: build the `iosApp` scheme in Xcode (runs `:shared:embedAndSignAppleFrameworkForXcode`)

## Project Structure
Where new code lives:
- `shared` — KMP module: business logic + shared Compose UI (`App()` in `commonMain`)
  - source sets: `commonMain`, `androidMain`, `iosMain`, `jvmMain`, `jsMain`,
    `wasmJsMain` (+ matching `*Test`)
- `androidApp` — Android entry point (`MainActivity`, `AndroidManifest.xml`)
- `desktopApp` — JVM entry point (`main.kt`, Compose Hot Reload)
- `webApp` — web entry point (js + wasmJs browser targets, `webMain` source set)
- `iosApp` — Xcode project (not a Gradle module)

## Code Style
- Kotlin with Compose Multiplatform (Material 3) in `commonMain`
- MVVM with ViewModel (`lifecycle-viewmodel-compose`) + StateFlow in `commonMain`
- Metro for dependency injection (compile-time, all targets)
- Ktor client + kotlinx.serialization for networking
- Room KMP + DataStore for persistence
- Coroutines + Flow for async operations

## Testing Strategy
- Unit tests: kotlin.test + hand-written fakes in `commonTest` (no JUnit5/MockK — JVM-only)
- UI tests: `runComposeUiTest` in `commonTest` for shared screens
- Integration tests: Room in-memory database, Ktor MockEngine
- Target: critical paths green on `:shared:allTests`, not arbitrary coverage %

## Boundaries
What is explicitly NOT in scope:
- [ ] List exclusions here

Code placement: business logic and shared UI live in `commonMain`. Platform code
enters only through `expect`/`actual` or DI-provided interfaces in `shared`'s
platform source sets — never in entry-point modules.
```

3. **Save as `SPEC.md`** in the project or module root

### Phase 2: Plan

4. Review spec with human — get explicit approval before continuing
5. Use `planning-and-task-breakdown` to create implementation tasks from the spec

### Phase 3: Tasks

6. Break spec into vertical slices (see `planning-and-task-breakdown`)
7. Each task references the spec section it implements

### Phase 4: Implement

8. Build incrementally (see `incremental-implementation`)
9. Every PR references the spec section it addresses
10. Spec evolves with the project — update it when requirements change

## Common Rationalizations

| Shortcut | Why It Fails |
|----------|-------------|
| "The feature is simple, no spec needed" | Simple features have hidden edge cases (process death on Android, backgrounding on iOS, browser refresh on web). A brief spec still beats none. |
| "We'll figure it out as we go" | Without a spec, each developer builds a different mental model. Alignment costs compound across five platforms. |
| "The ticket/issue IS the spec" | Tickets describe what to build. Specs describe how it fits into the system, which targets it ships to, what's excluded, and how to verify. |
| "Writing specs slows us down" | Rework from misalignment costs 3–10x more than a spec. |

## Red Flags

- Implementation started without written spec
- Spec has no "Targets" line — nobody agreed which platforms are in scope
- Spec has no "Boundaries" or exclusions section
- Spec doesn't specify testing strategy
- Multiple developers have different understandings of scope
- Spec never updated after requirement changes

## Verification

- [ ] SPEC.md exists in version control
- [ ] All seven sections filled in (Objective, Targets, Commands, Structure, Style, Testing, Boundaries)
- [ ] Targets line states which platforms ship, and Boundaries states what stays in commonMain vs platform source sets
- [ ] Human has reviewed and approved the spec
- [ ] Implementation tasks reference spec sections
- [ ] Spec updated when requirements changed during development
