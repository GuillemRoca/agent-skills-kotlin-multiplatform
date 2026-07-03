# GitHub Copilot Setup

## Skills Directory

Organize skills in your repository:

```bash
# Clone the repo
git clone https://github.com/GuillemRoca/agent-skills-kotlin-multiplatform.git

# Copy skills into your project
mkdir -p your-project/.github/skills
cp -r agent-skills-kotlin-multiplatform/skills/* your-project/.github/skills/
```

## Copilot Instructions

Create `.github/copilot-instructions.md` with your project's coding standards:

```markdown
# Kotlin Multiplatform Development Standards

## Architecture
- MVVM with Clean Architecture layers (UI → Domain → Data) in the shared module
- Metro for dependency injection (compile-time, constructor injection only; no Hilt in commonMain)
- ViewModels expose StateFlow, never MutableStateFlow or LiveData
- Prefer interface + DI over expect/actual; expect/actual only for per-platform one-liners

## Testing
- Write tests before implementation (Red-Green-Refactor)
- kotlin.test in commonTest; MockK/JUnit5 only in JVM test source sets
- runComposeUiTest for shared Compose UI tests
- No Thread.sleep in tests — use advanceUntilIdle()

## Code Quality
- Kotlin idioms: when expressions, sealed classes, extension functions
- Functions under 40 lines
- Handle all UI states: loading, success, error, empty

## Build Commands
- Build (Android entry point): ./gradlew :androidApp:assembleDebug
- Test (all targets): ./gradlew :shared:allTests
- Fast test loop: ./gradlew :shared:jvmTest

## Reference
See .github/skills/ for detailed workflow skills.
```

## Agent Personas

Use agent personas for targeted reviews in Copilot Chat:

- Reference `agents/code-reviewer.md` for five-axis code review
- Reference `agents/test-engineer.md` for test design and coverage analysis
- Reference `agents/security-auditor.md` for OWASP Mobile Top 10 auditing

## Best Practices

- Keep instructions concise — focus on rules, not explanations
- Leverage agent personas for specialized review workflows
- Reference specific skill content when working on particular development phases
- Integrate Copilot reviews into your existing PR process
