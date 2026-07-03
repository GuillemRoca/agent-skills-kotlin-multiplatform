# AGENTS.md — Agent Skills for Kotlin Multiplatform

## Skill-First Execution Model

When working on a Kotlin Multiplatform project with agent skills installed, **always check for a matching skill before implementing directly**. Skills provide battle-tested workflows that prevent common mistakes.

## Structured Lifecycle

Work follows a six-phase lifecycle. Each phase has dedicated skills:

```
DEFINE → PLAN → BUILD → VERIFY → REVIEW → SHIP
```

| Phase      | Skills                                                                                                                                                                                                                  | Purpose                                            |
|------------|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------|
| **DEFINE** | `interview-me`, `idea-refine`, `spec-driven-development`, `context-engineering`                                                                                                                                                                                            | Sharpen ideas, write specs, set up AI context      |
| **PLAN**   | `planning-and-task-breakdown`, `multiplatform-architecture`                                                                                                                                                                                                               | Break work into tasks, make architecture decisions |
| **BUILD**  | `incremental-implementation`, `test-driven-development`, `compose-multiplatform-ui`, `multiplatform-data-persistence`, `platform-interop`, `api-and-interface-design`, `source-driven-development`, `doubt-driven-development`, `code-simplification`, `documentation-and-adrs` | Implement incrementally with TDD                   |
| **VERIFY** | `multiplatform-testing`, `e2e-verification`, `multiplatform-accessibility`, `debugging-and-error-recovery`, `performance-optimization`                                                                                                                                    | Test, debug, and validate                          |
| **REVIEW** | `code-review-and-quality`, `security-and-hardening`                                                                                                                                                                                                                       | Review quality and security                        |
| **SHIP**   | `ci-cd-and-automation`, `git-workflow-and-versioning`, `shipping-and-launch`, `observability-and-instrumentation`, `deprecation-and-migration`                                                                                                                            | Automate, version, observe, and release            |

## Skill Directory Structure

Skills live in `skills/{skill-name}/SKILL.md`:

```
skills/
├── interview-me/SKILL.md
├── idea-refine/SKILL.md
├── spec-driven-development/SKILL.md
├── context-engineering/SKILL.md
├── planning-and-task-breakdown/SKILL.md
├── multiplatform-architecture/SKILL.md
├── incremental-implementation/SKILL.md
├── test-driven-development/SKILL.md
├── compose-multiplatform-ui/SKILL.md
├── multiplatform-data-persistence/SKILL.md
├── platform-interop/SKILL.md
├── api-and-interface-design/SKILL.md
├── source-driven-development/SKILL.md
├── doubt-driven-development/SKILL.md
├── code-simplification/SKILL.md
├── documentation-and-adrs/SKILL.md
├── multiplatform-testing/SKILL.md
├── e2e-verification/SKILL.md
├── multiplatform-accessibility/SKILL.md
├── debugging-and-error-recovery/SKILL.md
├── performance-optimization/SKILL.md
├── code-review-and-quality/SKILL.md
├── security-and-hardening/SKILL.md
├── ci-cd-and-automation/SKILL.md
├── git-workflow-and-versioning/SKILL.md
├── shipping-and-launch/SKILL.md
├── observability-and-instrumentation/SKILL.md
├── deprecation-and-migration/SKILL.md
└── using-agent-skills/SKILL.md
```

## SKILL.md Anatomy

Each skill follows this structure:

1. **Overview** — what and why
2. **When to Use** — trigger conditions and exclusions
3. **Core Process** — numbered steps with Kotlin code examples
4. **Common Rationalizations** — excuses agents use to skip steps + rebuttals
5. **Red Flags** — observable violations during review
6. **Verification** — exit checklist requiring tangible evidence

Target: each SKILL.md stays under 500 lines.

## Agent Personas

Three reusable agent personas in `agents/`:

| Agent                 | Role                                                                                  | Use When                                     |
|-----------------------|---------------------------------------------------------------------------------------|----------------------------------------------|
| `code-reviewer.md`    | Five-axis code review (Correctness, Readability, Architecture, Security, Performance) | Reviewing PRs or self-reviewing              |
| `test-engineer.md`    | Test design, coverage analysis, quality evaluation                                    | Designing test suites, finding coverage gaps |
| `security-auditor.md` | Cross-platform security audit, vulnerability assessment                               | Security review before release               |

## Orchestration: Personas and Skills

Three composable layers, each with a distinct job:

- **Skills** (`skills/<name>/SKILL.md`) — workflows with steps and exit criteria. The *how*. Mandatory steps when an intent matches.
- **Personas** (`agents/<role>.md`) — roles with a perspective and an output format. The *who*. See [docs/agent-personas.md](docs/agent-personas.md) for the full catalogue and orchestration rules.
- **Slash Commands** — user-facing entry points (e.g. `/spec`, `/build`, `/ship`) that compose skills and personas.

**Composition rule:** The user — or a slash command acting on their behalf — is the orchestrator. Personas do not invoke other personas. A persona may invoke skills.

The endorsed multi-persona pattern is **parallel fan-out with a merge step** — used during shipping to run `code-reviewer`, `security-auditor`, and `test-engineer` concurrently on the same diff and synthesize their reports into a single go/no-go decision.

## Command Mappings

Instructions map to development phases:

| Phase               | Skills Used                                                                       |
|---------------------|-----------------------------------------------------------------------------------|
| Specification       | `interview-me` (when vague) + `spec-driven-development`                           |
| Planning            | `planning-and-task-breakdown`                                                     |
| Building            | `incremental-implementation` + `test-driven-development`                          |
| Testing             | `test-driven-development` + `multiplatform-testing` + `e2e-verification`          |
| Reviewing           | `code-review-and-quality` + `security-and-hardening` + `performance-optimization` |
| Code Simplification | `code-simplification` + `code-review-and-quality`                                 |
| Shipping            | `shipping-and-launch` + `observability-and-instrumentation`                       |

## Reference Checklists

Detailed checklists in `references/`:

- `testing-patterns.md` — kotlin.test, fakes, runComposeUiTest, Turbine, Ktor MockEngine
- `security-checklist.md` — Secrets storage, Keystore/Keychain, Ktor TLS and pinning
- `performance-checklist.md` — Recomposition, startup, per-platform profiling, bundle size
- `accessibility-checklist.md` — TalkBack, VoiceOver, touch targets, contrast, semantics

## Core Principles

1. **Skills are workflows, not suggestions** — follow steps in order, never skip verification
2. **Anti-rationalizations are real** — check the "Common Rationalizations" table before skipping steps
3. **Evidence over assertions** — "it looks right" is not verification
4. **Assumption surfacing** — state assumptions explicitly before implementing
5. **Scope discipline** — only modify what's requested
6. **Common code first** — new code lands in the most-common source set it can compile in; platform code needs a stated reason
