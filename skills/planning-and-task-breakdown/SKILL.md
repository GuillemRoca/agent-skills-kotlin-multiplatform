---
name: planning-and-task-breakdown
description: >-
  Use when starting implementation of a feature or change that involves
  multiple files or steps. Produces a structured task list with vertical
  slices, acceptance criteria, and verification steps.
---

# Planning and Task Breakdown

## Overview

"The plan *is* the task — 10 minutes upfront prevents hours of rework." Break work into small, ordered, verifiable tasks before writing code. Each task is a vertical slice that leaves the system in a working state — on every target, not just one.

## When to Use

- Before implementing any feature spanning more than 2–3 files
- After writing a spec (follows `spec-driven-development`)
- When a task feels "too big to start"
- When multiple developers will work on related code

**Skip when:** The change is a single-file fix with obvious scope.

## Core Process

### Step 1: Read-Only Analysis

1. **Read the spec** (SPEC.md) or requirements — do not modify any code
2. **Map the codebase:**
   - Which modules are involved? (`shared`, `androidApp`, `desktopApp`, `webApp`, `iosApp`)
   - Which layers? (UI in commonMain → ViewModel → Repository → DataSource)
   - What existing code can be reused?
3. **Identify constraints:**
   - Which targets does this feature ship to?
   - Which pieces need platform implementations (expect/actual or DI-wired interfaces)?
   - Library versions and KMP support across targets (js/wasmJs artifacts available?)

### Step 2: Map Dependencies

4. **Draw the dependency graph:**
   - Data models → Repository → UseCase → ViewModel → UI (all in commonMain)
   - Room KMP entities → DAOs → migrations → per-platform database builders
   - Navigation routes (`@Serializable`) → screen composables → ViewModels
5. **Identify the critical path** — what must exist before other work can start

### Step 3: Vertical Slicing

6. **Slice vertically, not horizontally:**

   **Wrong (horizontal):**
   - Task 1: Create all Room entities
   - Task 2: Create all DAOs
   - Task 3: Create all repositories
   - Task 4: Create all ViewModels
   - Task 5: Create all screens

   **Right (vertical):**
   - Task 1: User can view item list (Entity + DAO + Repo + ViewModel + Screen in commonMain, plus expect/actual database builder per platform)
   - Task 2: User can create new item (AddScreen + ViewModel + Repo insert)
   - Task 3: User can edit existing item (EditScreen + ViewModel + Repo update)
   - Task 4: User can delete item (swipe-to-delete + Repo delete + undo)

   A slice is also NOT "implement Android first, port to the other targets later." Every slice spans commonMain logic + shared Compose UI + any per-platform `actual`s, and shared code compiles for all targets from the first task.

7. **Each slice must:**
   - Deliver observable functionality
   - Be testable in isolation
   - Leave `./gradlew :shared:allTests` passing and at least one entry point building

### Step 4: Write Structured Tasks

8. **Create `tasks/plan.md`** with the dependency graph and approach
9. **Create `tasks/todo.md`** with tasks in execution order:

```markdown
## Tasks

### Task 1: [Short description]
**Files:** `shared/src/commonMain/kotlin/.../ItemListScreen.kt`,
`shared/src/commonMain/kotlin/.../ItemDao.kt`
**Acceptance criteria:**
- Item list loads from Room database
- Empty state shown when no items exist
- Loading state shown during fetch
**Verification:**
- [ ] `./gradlew :shared:allTests` passes
- [ ] `./gradlew :androidApp:assembleDebug` (or `:desktopApp:run`) succeeds
- [ ] `@Preview` in commonMain renders correctly
```

### Step 5: Order by Dependencies

10. **Sequence tasks** so each builds on the previous
11. **Add checkpoints** every 2–3 tasks: run `./gradlew :shared:allTests` plus at least one entry-point build, review with human
12. **Flag risks** on tasks with uncertainty — mark as "spike" if investigation needed

## Task Sizing Guide

| Size | Files | Duration | Action |
|------|-------|----------|--------|
| Small | 1–2 | < 30 min | Execute directly |
| Medium | 3–5 | 30–60 min | Execute with checkpoint |
| Large | 6+ | > 60 min | **Split further** |

Split when: task touches >2 independent subsystems, or acceptance criteria exceed 5 items.

## Common Rationalizations

| Shortcut | Why It Fails |
|----------|-------------|
| "I'll plan as I go" | Without a task list, you lose track of scope and dependencies. Rework multiplies. |
| "The spec is the plan" | Specs describe *what*. Plans describe *how* and *in what order*. |
| "Planning takes too long" | A 10-minute plan prevents hours of backtracking. |
| "I know this codebase, I don't need a plan" | Plans catch dependency gaps that familiarity masks. |
| "Ship Android first, port the rest later" | "Porting" later means re-testing every layer twice. Common code compiles for all targets from day one — plan it that way. |

## Red Flags

- Implementing without a written task list
- Tasks missing acceptance criteria
- No verification steps on tasks
- Tasks spanning >5 files without splitting
- No checkpoints between tasks
- Horizontal slicing (all DAOs, then all repos, then all VMs)
- Platform-per-task slicing (Android now, iOS/desktop/web "later")

## Verification

- [ ] `tasks/plan.md` exists with dependency graph
- [ ] `tasks/todo.md` exists with ordered tasks
- [ ] Every task has acceptance criteria and verification steps
- [ ] Tasks are vertically sliced (each delivers observable functionality across targets)
- [ ] No task exceeds "Large" sizing without justification
- [ ] Checkpoints (`./gradlew :shared:allTests` + one entry-point build) placed every 2–3 tasks
- [ ] Human has reviewed the plan before implementation starts
