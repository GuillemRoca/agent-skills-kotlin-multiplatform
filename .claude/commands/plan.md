# /plan — Planning and Task Breakdown

Break work into ordered, verifiable tasks before writing code.

## Instructions

Load and follow the `planning-and-task-breakdown` skill from `skills/planning-and-task-breakdown/SKILL.md`.

### Process (Read-Only)

**Do not modify any code.** This command produces a plan, not implementation.

1. **Read the spec** (`SPEC.md`) or requirements
2. **Map the codebase** — which modules and source sets are involved? (`shared` commonMain vs platform source sets, entry-point modules)
3. **Map dependencies** — what must exist before other work can start?
4. **Slice vertically** — each task delivers observable, testable functionality across commonMain and each target entry point
5. **Write structured tasks** with acceptance criteria and verification steps
6. **Order by dependencies** with checkpoints every 2–3 tasks

### Output

Create two files:
- `tasks/plan.md` — dependency graph and approach
- `tasks/todo.md` — ordered task list with acceptance criteria

### Key Rule

Slice vertically (end-to-end per feature, from commonMain through each entry point), not horizontally (all DAOs, then all repositories, then all ViewModels). Each slice must compile on all configured targets.

Present the plan to the human for review before implementation begins.
