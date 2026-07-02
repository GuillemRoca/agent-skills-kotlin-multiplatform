# /review — Code Review

Review code across five axes with categorized findings.

## Instructions

Load and follow these skills:
- `code-review-and-quality` from `skills/code-review-and-quality/SKILL.md`
- `security-and-hardening` from `skills/security-and-hardening/SKILL.md`
- `performance-optimization` from `skills/performance-optimization/SKILL.md`

### Five-Axis Review

#### 1. Correctness
- All UI states handled? (loading, success, error, empty)
- Nulls handled safely? (no `!!`, proper safe calls)
- Coroutines structured correctly? (scope, cancellation)
- Room KMP queries match schema? Migrations correct?
- Code compiles on all configured targets? No JVM-only APIs in commonMain?

#### 2. Readability
- Kotlin idioms used? (when, sealed classes, extension functions)
- Clear naming? Functions < 40 lines?
- Consistent with existing codebase patterns?

#### 3. Architecture
- Layer boundaries respected? (UI → Domain → Data)
- Logic in the most common source set possible? Entry-point modules stay thin?
- No needless `expect`/`actual` (prefer interface + DI)?
- Metro graph wiring correct? (`@Inject`, `@ContributesBinding`, proper `@SingleIn` scoping)
- StateFlow exposed (not MutableStateFlow)?
- No business logic in Composables?

#### 4. Security
- No hardcoded secrets? (commonMain constants ship in every binary, including the wasm bundle)
- Input validated at boundaries?
- Deep links / universal links validated?
- No sensitive data in logs? (Kermit)

#### 5. Performance
- No N+1 queries?
- Heavy work off the main thread?
- Compose recompositions optimized? (`remember`/stability discipline, stable types, `key` params)
- Large datasets paginated? Per-target payload (web bundle size) reasonable?

### Finding Categories

| Category | Action |
|----------|--------|
| **Critical** | Must fix before merge |
| **Important** | Should fix before merge |
| **Suggestion** | Optional improvement |
| **Nit** | Formatting preference |
| **FYI** | Context, no action needed |

### Verification

Run before approving:
```bash
./gradlew :shared:allTests
./gradlew :androidApp:assembleDebug
./gradlew :shared:compileKotlinIosSimulatorArm64
./gradlew detekt
./gradlew spotlessCheck
```
