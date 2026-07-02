# /spec — Spec-Driven Development

Write a structured specification before writing any code.

## Instructions

Load and follow the `spec-driven-development` skill from `skills/spec-driven-development/SKILL.md`.

### Clarifying Questions

Before writing the spec, ask:
1. **What problem** does this solve? Who is the user?
2. **Which features** are in scope? Which are explicitly excluded?
3. **What stack?** (which targets: Android, iOS, desktop, web? shared UI or shared logic only? Metro vs Koin, Room KMP vs SQLDelight, Ktor engines per platform)
4. **What boundaries?** (offline support? native UI islands on some platforms? accessibility? deep links / universal links? web js/wasm compatibility?)

### Spec Structure

Document these sections in `SPEC.md`:

1. **Objective** — what and why (1 paragraph)
2. **Commands** — key Gradle tasks (`./gradlew :shared:allTests`, `./gradlew :androidApp:assembleDebug`, `./gradlew :desktopApp:run`, `./gradlew :webApp:wasmJsBrowserTest`, etc.)
3. **Project Structure** — `shared` (commonMain + platform source sets) plus entry points: `androidApp`, `iosApp` (Xcode project), `desktopApp`, `webApp`
4. **Code Style** — Kotlin, Compose Multiplatform, MVVM with lifecycle-viewmodel-compose, Metro DI, Coroutines; expect/actual only where interface + DI does not fit
5. **Testing Strategy** — kotlin.test in commonTest with hand-written fakes, `runComposeUiTest` for shared UI, per-target test tasks, Maestro flows on Android and iOS
6. **Boundaries** — what is explicitly NOT in scope (including targets you will not ship)

### Output

Save as `SPEC.md` in the project root. This becomes the development contract — review with the human before implementation begins.
