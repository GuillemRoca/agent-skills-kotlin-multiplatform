# Agent Skills for Kotlin Multiplatform

**Production-grade Kotlin Multiplatform engineering skills for AI coding agents.**

29 specialized workflows covering the full development lifecycle from spec to the app stores — built for Kotlin Multiplatform and Compose Multiplatform: shared logic **and** shared UI across Android, iOS, desktop, and web.

## What Are Skills?

Skills are structured Markdown files that teach AI coding agents **how** to work, not just what to build. Each skill provides:

- **Step-by-step workflows** — not vague advice, but numbered processes
- **Kotlin code examples** — real KMP/CMP patterns, not pseudocode
- **Anti-rationalizations** — rebuttals for common shortcuts agents attempt
- **Verification checklists** — tangible evidence, not "looks correct"

## Development Lifecycle

```
  DEFINE         PLAN          BUILD        VERIFY        REVIEW         SHIP
 ┌──────┐      ┌──────┐      ┌──────┐      ┌──────┐      ┌──────┐      ┌──────┐
 │ Idea │ ───▶ │ Spec │ ───▶ │ Code │ ───▶ │ Test │ ───▶ │  QA  │ ───▶ │  Go  │
 │Refine│      │  PRD │      │ Impl │      │Debug │      │ Gate │      │ Live │
 └──────┘      └──────┘      └──────┘      └──────┘      └──────┘      └──────┘
  /spec          /plan        /build        /test         /review       /ship
```

Seven slash commands provide quick access:

| Phase | Command | What It Does |
|-------|---------|-------------|
| DEFINE | `/spec` | Write a structured specification |
| PLAN | `/plan` | Break work into ordered tasks |
| BUILD | `/build` | Implement incrementally with TDD |
| BUILD | `/code-simplify` | Simplify code preserving behavior |
| VERIFY | `/test` | Test-driven development workflow |
| REVIEW | `/review` | Five-axis code review |
| SHIP | `/ship` | Pre-launch checklist and rollout |

## All 29 Skills

### DEFINE Phase
| Skill | Description |
|-------|-------------|
| `interview-me` | Iterative questioning that turns vague requests into ~95%-confident requirements |
| `idea-refine` | Sharpen vague ideas into focused, actionable directions |
| `spec-driven-development` | Write structured specs before coding, with targets and source-set boundaries |
| `context-engineering` | Set up project context for AI-assisted development |

### PLAN Phase
| Skill | Description |
|-------|-------------|
| `planning-and-task-breakdown` | Break work into vertical slices that compile for every target |
| `multiplatform-architecture` | Module structure, source sets, expect/actual vs interfaces, Metro DI, KMP ViewModel |

### BUILD Phase
| Skill | Description |
|-------|-------------|
| `incremental-implementation` | Build in small, verifiable increments across targets |
| `test-driven-development` | Red-Green-Refactor with kotlin.test in commonTest |
| `compose-multiplatform-ui` | Shared Compose UI, Material 3, resources, navigation, per-platform entry points |
| `multiplatform-data-persistence` | Room KMP, DataStore, SQLDelight, offline-first, repository pattern |
| `platform-interop` | expect/actual, iOS framework integration, Swift interop, per-platform capabilities |
| `api-and-interface-design` | Ktor client, sealed types, contract-first design |
| `source-driven-development` | Every framework decision backed by official docs |
| `doubt-driven-development` | Adversarial self-review for hard-to-reverse decisions |
| `code-simplification` | Simplify code without changing behavior |
| `documentation-and-adrs` | Architecture Decision Records and documentation |

### VERIFY Phase
| Skill | Description |
|-------|-------------|
| `multiplatform-testing` | commonTest, runComposeUiTest, per-target test tasks, instrumented and simulator tests |
| `e2e-verification` | Maestro flows on Android and iOS: acceptance criteria as executable checks |
| `multiplatform-accessibility` | Compose semantics for TalkBack and VoiceOver, touch targets, keyboard navigation |
| `debugging-and-error-recovery` | Systematic debugging across Logcat, Xcode, JVM debuggers, and browser devtools |
| `performance-optimization` | Recomposition discipline plus per-platform profiling and release-build measurement |

### REVIEW Phase
| Skill | Description |
|-------|-------------|
| `code-review-and-quality` | Five-axis review: Correctness, Readability, Architecture, Security, Performance |
| `security-and-hardening` | Keystore/Keychain secrets, Ktor TLS and pinning, per-target hardening |

### SHIP Phase
| Skill | Description |
|-------|-------------|
| `ci-cd-and-automation` | GitHub Actions target matrix, Gradle caching, macOS runners for iOS |
| `git-workflow-and-versioning` | Trunk-based dev, atomic commits, one version fanned out to every platform |
| `shipping-and-launch` | Coordinated rollout across Play Store, App Store, desktop, and web |
| `observability-and-instrumentation` | Kermit logging, cross-platform crash reporting, release health |
| `deprecation-and-migration` | Platform floors, library migrations, strangler pattern, structure migrations |

### META
| Skill | Description |
|-------|-------------|
| `using-agent-skills` | How agents should operate: assumptions, scope, simplicity |

## Additional Resources

### Agent Personas (`agents/`)
- **code-reviewer.md** — Five-axis code review specialist
- **test-engineer.md** — Test design and coverage analysis
- **security-auditor.md** — Cross-platform security auditor

### Reference Checklists (`references/`)
- **testing-patterns.md** — kotlin.test, fakes, runComposeUiTest, Turbine, Ktor MockEngine
- **security-checklist.md** — Secrets storage, network, auth, per-target hardening
- **performance-checklist.md** — Recomposition, startup, per-platform profiling, bundle size
- **accessibility-checklist.md** — TalkBack, VoiceOver, touch targets, contrast, semantics

## Installation

<details>
<summary><b>Claude Code (recommended)</b></summary>

**Marketplace install:**

```bash
claude plugin marketplace add GuillemRoca/claude-plugins
claude plugin install agent-skills-kotlin-multiplatform@guillemroca
```

Restart Claude Code. Skills, slash commands (`/spec`, `/plan`, `/build`, `/test`, `/review`, `/ship`), and hooks are available immediately.

> **How loading works:** `.claude-plugin/plugin.json` registers only `commands` and `skills` explicitly. The `agents/` personas and `hooks/hooks.json` are picked up by Claude Code's convention-based auto-discovery of those directories — they are intentionally *not* listed in the manifest (listing them caused duplicate-load errors in the sibling Android plugin). If personas or hooks stop loading after a Claude Code update, check the auto-discovery behavior before touching the manifest.

To update later (use the fully qualified `plugin@marketplace` form — the short name returns "Plugin not found"):

```bash
claude plugin marketplace update guillemroca
claude plugin update agent-skills-kotlin-multiplatform@guillemroca
```

> **Versioning:** the plugin has no `version` field — every commit on `main` is a release, and Claude Code shows the installed version as a commit SHA. If you installed the numbered 1.0.0 release, the commands above move you to the latest commit; no reinstall needed. See [CONTRIBUTING.md](CONTRIBUTING.md#releases).

> **Update fails with an SSH / "Permission denied (publickey)" error?** This is the most common cause. The marketplace clones repos via SSH. If you don't have SSH keys set up on GitHub, either [add your SSH key](https://docs.github.com/en/authentication/connecting-to-github-with-ssh/adding-a-new-ssh-key-to-your-github-account) or use the full HTTPS URL to force the HTTPS cloning:
> ```bash
> /plugin marketplace add https://github.com/GuillemRoca/claude-plugins.git
> /plugin install agent-skills-kotlin-multiplatform@guillemroca
> ```

</details>

<details>
<summary><b>Cursor</b></summary>

Copy any `SKILL.md` into `.cursor/rules/`, or combine into a single `.cursorrules` file. See [docs/cursor-setup.md](docs/cursor-setup.md).

```bash
mkdir -p .cursor/rules
cp agent-skills-kotlin-multiplatform/skills/test-driven-development/SKILL.md .cursor/rules/
cp agent-skills-kotlin-multiplatform/skills/code-review-and-quality/SKILL.md .cursor/rules/
cp agent-skills-kotlin-multiplatform/skills/incremental-implementation/SKILL.md .cursor/rules/
```

</details>

<details>
<summary><b>Gemini CLI</b></summary>

Install skills for auto-discovery, or add to `GEMINI.md` for persistent context. See [docs/gemini-cli-setup.md](docs/gemini-cli-setup.md).

```bash
mkdir -p .gemini/skills
cp -r agent-skills-kotlin-multiplatform/skills/* .gemini/skills/
```

</details>

<details>
<summary><b>Windsurf</b></summary>

Combine 2–3 core skills into `.windsurfrules`. See [docs/windsurf-setup.md](docs/windsurf-setup.md).

</details>

<details>
<summary><b>OpenCode</b></summary>

Uses agent-driven skill execution via `AGENTS.md` and the `skill` tool. See [docs/opencode-setup.md](docs/opencode-setup.md).

</details>

<details>
<summary><b>GitHub Copilot</b></summary>

Use agent definitions from `agents/` as Copilot personas and skill content in `.github/copilot-instructions.md`. See [docs/copilot-setup.md](docs/copilot-setup.md).

</details>

<details>
<summary><b>Kiro IDE</b></summary>

Copy skills into `.kiro/skills/` at the project or global level. Kiro also supports `AGENTS.md`. See [Kiro docs](https://kiro.dev/docs/skills/).

</details>

<details>
<summary><b>Other Agents</b></summary>

Skills are plain Markdown — they work with any agent that accepts system prompts or instruction files:

```bash
git clone https://github.com/GuillemRoca/agent-skills-kotlin-multiplatform.git
```

Copy skills into your project and point your agent at them. See [docs/getting-started.md](docs/getting-started.md).

</details>

## Tech Stack

These skills are designed for:

- **Language:** Kotlin (2.4+)
- **Targets:** Android, iOS, Desktop (JVM), Web (Kotlin/JS + Kotlin/Wasm)
- **UI:** Compose Multiplatform + Material 3 (shared UI in `commonMain`)
- **Project shape:** `shared` module + `androidApp`/`desktopApp`/`webApp` entry points + `iosApp` Xcode project (JetBrains recommended structure, AGP 9 compatible)
- **Architecture:** MVVM / MVI + Clean Architecture
- **DI:** Metro (compile-time, all targets)
- **Async:** Coroutines + Flow
- **Database:** Room KMP + DataStore (SQLDelight as alternative)
- **Network:** Ktor Client + kotlinx.serialization
- **ViewModel/Navigation:** androidx lifecycle ViewModel + Navigation Compose (multiplatform)
- **Testing:** kotlin.test + Compose UI test (`runComposeUiTest`) + Turbine + Maestro
- **Build:** Gradle (Kotlin DSL) with version catalogs
- **CI:** GitHub Actions (Linux + macOS runners)
- **Distribution:** Play Store + App Store + desktop packages (Dmg/Msi/Deb) + static web hosting

## Contributing

See [CONTRIBUTING.md](CONTRIBUTING.md) for guidelines on adding or modifying skills.

## License

MIT — see [LICENSE](LICENSE).

## Credits

Based on [agent-skills](https://github.com/addyosmani/agent-skills) by Addy Osmani and the sibling [agent-skills-android](https://github.com/GuillemRoca/agent-skills-android), adapted for the Kotlin Multiplatform ecosystem following the [JetBrains Compose Multiplatform documentation](https://kotlinlang.org/docs/multiplatform/compose-multiplatform-create-first-app.html).
