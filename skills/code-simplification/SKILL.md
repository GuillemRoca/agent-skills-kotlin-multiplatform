---
name: code-simplification
description: >-
  Use when code is harder to understand than it needs to be. Guides
  incremental simplification that preserves behavior exactly, following
  project conventions and Kotlin idioms.
---

# Code Simplification

## Overview

Simplicity is about comprehension speed, not line count. Simple code is code another developer can read and understand quickly. This skill guides incremental simplification — making code easier to understand without changing what it does.

## When to Use

- Code is harder to understand than its logic warrants
- Functions exceed ~40 lines or have deep nesting
- After a feature is working and tested (simplify, don't rewrite)
- During code review when readability is flagged

**Skip when:**
- Code is already clean and conventional
- You don't fully understand what the code does (Chesterton's Fence)
- The code is performance-critical and structured for speed
- A rewrite is already planned

## Core Process

Ground rules for every simplification: preserve behavior exactly (change form, not function); follow project conventions; prefer clarity over cleverness; don't over- or under-abstract; simplify what was asked, nothing more.

### Step 1: Understand (Chesterton's Fence)

1. **Read the code and its tests** — understand what it does and why
2. **Check git blame** — understand the history of complex sections
3. **Identify the author's intent** — is complexity intentional (performance, platform workaround)?
4. **Respect `/* simplify-ignore-start */` blocks** — skip annotated sections

### Step 2: Identify Opportunities

5. **Scan for these patterns:**

| Pattern | Kotlin Simplification |
|---------|----------------------|
| Deep nesting | Early returns / `when` expressions |
| Long functions | Extract well-named private functions |
| Complex conditionals | `when` expression or sealed class |
| Nullable chains | `?.let { }`, `?:`, safe calls |
| Manual null checks | `requireNotNull()`, `checkNotNull()` |
| Mutable state | `val` over `var`, immutable collections |
| Callback nesting | Coroutines / Flow |
| Builder patterns | Kotlin DSL or `apply`/`also` |
| Type casting chains | `is` smart casts, sealed hierarchies |
| Manual resource management | `use { }` or `withContext` |
| Needless expect/actual pairs | Interface in `commonMain` + DI, or move shared logic to `commonMain` |

### Step 3: Apply Incrementally

6. **One change at a time:**
   - Make a single simplification
   - Run `./gradlew :shared:jvmTest` — verify tests pass
   - If tests fail, **revert immediately** and investigate
   - Commit the change
   - Repeat

7. **Common Kotlin simplifications:**

```kotlin
// Before: nested null checks
fun getDisplayName(user: User?): String {
    if (user != null) {
        if (user.displayName != null) {
            return user.displayName
        } else {
            return user.email
        }
    } else {
        return "Anonymous"
    }
}

// After: idiomatic Kotlin
fun getDisplayName(user: User?): String =
    user?.displayName ?: user?.email ?: "Anonymous"
```

```kotlin
// Before: chained if/else returns
fun categorize(score: Int): String {
    if (score >= 90) return "Excellent"
    else if (score >= 70) return "Good"
    else if (score >= 50) return "Average"
    else return "Poor"
}

// After: when expression
fun categorize(score: Int): String = when {
    score >= 90 -> "Excellent"
    score >= 70 -> "Good"
    score >= 50 -> "Average"
    else -> "Poor"
}
```

8. **KMP-specific: collapse needless expect/actual pairs.** If the `actual` implementations are identical, the logic belongs in `commonMain`. If they differ only in one platform fact, shrink the pair to that one fact — or replace it with an interface wired through DI, which is easier to fake in `commonTest`:

```kotlin
// Before: expect/actual class; every platform source set repeats the
// same formatting logic around one platform-specific call
// commonMain
expect class GreetingFormatter() {
    fun format(name: String): String
}
// androidMain, iosMain, jvmMain... each duplicate the body

// After: logic once in commonMain; only the platform fact stays expect/actual
interface Platform { val name: String }
expect fun currentPlatform(): Platform

class GreetingFormatter(private val platform: Platform = currentPlatform()) {
    fun format(name: String): String = "Hello, $name — from ${platform.name}"
}
```

Reserve `expect`/`actual` for obvious per-platform one-liners or classes that must inherit platform types; otherwise prefer common interfaces plus platform implementations behind DI.

### Step 4: Verify

9. **Run the full check:**
   - `./gradlew :shared:allTests` — all targets pass
   - `./gradlew :shared:compileKotlinIosSimulatorArm64` — collapsed expect/actual code still compiles off the JVM
10. **Review the diff** — does it only change form, not behavior?
11. **Read the simplified code fresh** — is it actually clearer?

## Common Rationalizations

| Shortcut | Why It Fails |
|----------|-------------|
| "I'll simplify and add features at the same time" | Mixed changes are impossible to review and risky to revert. |
| "This clever one-liner is simpler" | If it takes 30 seconds to parse, it's not simpler. |
| "I'll refactor the whole file while I'm here" | Scope creep. Simplify what was asked, nothing more. |
| "The tests are passing, so my refactor is safe" | Tests may not cover the behavior you changed. Check coverage first. |

## Red Flags

- Behavior changed during "simplification"
- Tests skipped or disabled
- Simplification mixed with feature changes
- New abstractions introduced for single-use code
- expect/actual pairs whose `actual` implementations are identical
- Code made "shorter" but harder to read
- `/* simplify-ignore */` blocks modified

## Verification

- [ ] Diff changes form only — not behavior
- [ ] Each simplification committed separately
- [ ] Code is genuinely easier to understand (not just shorter)
- [ ] `/* simplify-ignore */` blocks untouched
- [ ] No expect/actual pairs remain whose actuals are identical
- [ ] Non-JVM target still compiles (`./gradlew :shared:compileKotlinIosSimulatorArm64`)
- [ ] All tests pass on all targets (`./gradlew :shared:allTests`)
