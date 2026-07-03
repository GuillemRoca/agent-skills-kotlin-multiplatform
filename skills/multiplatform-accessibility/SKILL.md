---
name: multiplatform-accessibility
description: >-
  Use when building or reviewing Compose Multiplatform UI for accessibility.
  Semantics written once in commonMain drive both TalkBack (Android) and
  VoiceOver (iOS). Covers content descriptions, touch targets, semantics
  modifiers, focus order, keyboard navigation on desktop, color contrast,
  and testing with runComposeUiTest plus manual screen-reader passes on
  both mobile platforms.
---

# Multiplatform Accessibility

## Overview

Accessibility is not optional — it is a requirement for reaching all users. In Compose Multiplatform the semantics you write once in `shared/src/commonMain` feed the accessibility trees of every target: TalkBack on Android and VoiceOver on iOS (CMP iOS accessibility is stable), plus keyboard navigation on desktop. Web (js/wasm) is Beta and needs extra manual verification.

Core standards apply on every target:

| Requirement | Standard |
|------------|----------|
| Touch targets | Minimum 48dp x 48dp |
| Color contrast (text) | >= 4.5:1 (normal), >= 3:1 (large 18sp+) |
| Color contrast (UI components) | >= 3:1 against adjacent colors |
| Content descriptions | All meaningful visual elements |
| Focus order | Logical reading order; full keyboard reachability on desktop |
| No color-only information | Additional indicators required |

## When to Use

- Building new shared UI screens or components
- Reviewing existing shared UI for accessibility compliance
- Fixing issues reported by TalkBack or VoiceOver users
- Before shipping any user-facing feature on any target

**Skip when:** Working on non-UI code (repositories, use cases, networking).

## Core Process

### Step 1: Content Descriptions in commonMain

Every meaningful visual element needs a description. Written once, the same description is announced by TalkBack and VoiceOver:

```kotlin
// Icons that convey information
Icon(
    imageVector = Icons.Default.Delete,
    contentDescription = "Delete task", // Descriptive, not "delete icon"
)

// Decorative elements — explicitly null, skipped by both screen readers
Icon(
    imageVector = Icons.Default.Circle,
    contentDescription = null,
)

// Images — load through Res, never an Android R class
Image(
    painter = painterResource(Res.drawable.user_avatar),
    contentDescription = "Profile photo of ${user.name}",
)

// IconButtons — the description sits on the Icon
IconButton(onClick = onDelete) {
    Icon(
        imageVector = Icons.Default.Delete,
        contentDescription = "Delete task",
    )
}
```

Description rules:
- Describe the **action or meaning**, not the visual appearance: "Delete task" not "red trash can icon"
- Use `null` for purely decorative elements — never `""`
- Include dynamic content: "Profile photo of John" not just "profile photo"
- Localize descriptions like any other string: `stringResource(Res.string.delete_task)` (see `compose-multiplatform-ui`)

### Step 2: Touch Targets

Enforce 48dp minimum targets on every platform — including desktop, where touchscreens and motor-impaired mouse users exist:

```kotlin
IconButton(
    onClick = onToggle,
    modifier = Modifier.size(48.dp), // At least 48dp
) {
    Icon(
        imageVector = Icons.Default.Check,
        contentDescription = "Mark complete",
        modifier = Modifier.size(24.dp), // The icon itself can be smaller
    )
}

// Custom clickable elements: guarantee the minimum interaction area
@Composable
fun SmallChip(
    label: String,
    onClick: () -> Unit,
    modifier: Modifier = Modifier,
) {
    Surface(
        onClick = onClick,
        modifier = modifier.defaultMinSize(minHeight = 48.dp, minWidth = 48.dp),
    ) {
        Text(label, modifier = Modifier.padding(horizontal = 12.dp, vertical = 4.dp))
    }
}
```

Hover affordances on desktop do not replace size requirements — a hover-revealed 20dp button is still a 20dp target.

### Step 3: Semantics Modifiers

Semantics in commonMain map to both the Android and iOS accessibility trees:

```kotlin
// Merge children into a single TalkBack/VoiceOver announcement
@Composable
fun TaskItem(
    task: Task,
    onToggle: () -> Unit,
    modifier: Modifier = Modifier,
) {
    Row(
        modifier = modifier
            .semantics(mergeDescendants = true) {
                stateDescription = if (task.completed) "Completed" else "Not completed"
                customActions = listOf(
                    CustomAccessibilityAction("Toggle completion") {
                        onToggle()
                        true
                    }
                )
            }
            .clickable(onClick = onToggle),
    ) {
        Checkbox(checked = task.completed, onCheckedChange = null)
        Text(task.title)
    }
}

// Headings enable heading navigation (TalkBack reading controls, VoiceOver rotor)
Text(
    text = stringResource(Res.string.my_tasks),
    style = MaterialTheme.typography.headlineMedium,
    modifier = Modifier.semantics { heading() },
)

// Live regions announce dynamic content changes
Text(
    text = "3 items remaining",
    modifier = Modifier.semantics { liveRegion = LiveRegionMode.Polite },
)
```

Semantics rules:
- `mergeDescendants = true` for compound components (card with title + subtitle)
- `heading()` for section titles
- `liveRegion` for content that updates dynamically (counters, status)
- `stateDescription` for custom state (not just "checked/unchecked")
- `Role` for custom interactive elements

### Step 4: Focus Order and Keyboard Navigation

Default traversal follows visual order. Manage focus explicitly for dialogs, and treat desktop as keyboard-first:

```kotlin
@Composable
fun TaskDialog(onDismiss: () -> Unit) {
    val focusRequester = remember { FocusRequester() }

    AlertDialog(
        onDismissRequest = onDismiss,
        title = {
            Text(
                text = stringResource(Res.string.delete_task_title),
                modifier = Modifier.focusRequester(focusRequester),
            )
        },
        confirmButton = { /* ... */ },
    )

    LaunchedEffect(Unit) {
        focusRequester.requestFocus()
    }
}
```

Desktop keyboard rules:
- Every interactive element must be reachable with Tab/Shift+Tab and activatable with Enter/Space — Material components support this; custom clickables need `Modifier.focusable()` and visible focus indication
- Verify traversal order matches reading order after layout changes; fix with `Modifier.focusProperties { next = ... }` when needed
- Provide shortcuts for frequent actions via `Modifier.onKeyEvent` in the window scope

Web caveat: js/wasm targets are Beta and browser screen-reader integration is still limited. Do not assume semantics reach the browser accessibility tree — verify manually in the shipped browsers and keep critical flows keyboard-operable.

### Step 5: Color and Contrast

Never rely on color alone; use theme tokens that guarantee contrast:

```kotlin
// BAD: only color distinguishes status
Text(
    text = task.title,
    color = if (task.overdue) Color.Red else Color.Black,
)

// GOOD: color + icon + text label
Row {
    if (task.overdue) {
        Icon(
            imageVector = Icons.Default.Warning,
            contentDescription = null, // read as part of merged semantics
            tint = MaterialTheme.colorScheme.error,
        )
    }
    Text(
        text = task.title,
        color = if (task.overdue) MaterialTheme.colorScheme.error
                else MaterialTheme.colorScheme.onSurface,
    )
    if (task.overdue) {
        Text(
            text = stringResource(Res.string.overdue),
            color = MaterialTheme.colorScheme.error,
            style = MaterialTheme.typography.labelSmall,
        )
    }
}
```

Material scheme pairs (`onSurface`/`surface`, `onPrimary`/`primary`, `error`/`onError`) are designed for compliance — validate any custom colors in your light AND dark schemes against the 4.5:1 / 3:1 ratios (see `references/accessibility-checklist.md`).

### Step 6: Testing

**Shared automated tests in commonTest** — `runComposeUiTest` asserts on the same semantics tree that screen readers consume, and runs on every target via `./gradlew :shared:allTests` (fast loop: `:shared:jvmTest`). Dependency: `org.jetbrains.compose.ui:ui-test`; no JUnit rule needed. See `multiplatform-testing`.

```kotlin
@OptIn(ExperimentalTestApi::class)
class TaskItemAccessibilityTest {

    @Test
    fun deleteButton_hasMeaningfulDescription() = runComposeUiTest {
        setContent {
            TaskItem(
                task = Task(id = "1", title = "Buy groceries", completed = false),
                onToggle = {},
                onDelete = {},
            )
        }

        onNodeWithContentDescription("Delete task")
            .assertExists()
            .assertHasClickAction()
    }

    @Test
    fun overdueTask_isNotColorOnly() = runComposeUiTest {
        setContent { TaskItem(task = overdueTask, onToggle = {}, onDelete = {}) }

        onNodeWithText("Overdue").assertExists()
    }
}
```

**Manual screen-reader passes — run BOTH columns before shipping.** A passing TalkBack session does not prove VoiceOver works; traversal, announcements, and gesture mappings differ:

| Action | Android (TalkBack) | iOS (VoiceOver) |
|--------|--------------------|-----------------|
| Enable | Settings > Accessibility > TalkBack | Settings > Accessibility > VoiceOver (or side-button triple-click) |
| Next / previous element | Swipe right / left | Swipe right / left |
| Activate focused element | Double-tap | Double-tap |
| Navigate by headings | Reading controls (swipe up/down), then swipe | Rotor (two-finger twist) set to Headings, then swipe up/down |
| Automated audit | Accessibility Scanner app | Xcode Accessibility Inspector (works on the simulator) |

Verify on both: every announcement makes sense without seeing the screen, all interactive elements are reachable, nothing is skipped or read out of order, and state changes (toggles, live regions) are announced.

**Desktop:** complete one full keyboard-only pass per screen — no mouse. **Web:** manual browser pass with a screen reader where supported; keyboard pass always.

## Common Rationalizations

| Shortcut | Why It Fails |
|----------|-------------|
| "We'll add accessibility later" | Retrofitting is 5-10x harder than building accessible from the start. |
| "TalkBack works, so VoiceOver will too" | Same semantics, different platform mapping. Traversal and announcements diverge — test both. |
| "ContentDescription is visual boilerplate" | It is the only interface screen-reader users have to your app — on two platforms at once. |
| "Material components handle accessibility" | They provide a foundation. Descriptions, focus order, and semantic grouping are still yours. |
| "Touch targets look too big" | The visual element can be 24dp; the target must be 48dp. They are independent. |
| "Web is Beta, accessibility there can wait" | Keyboard operability and semantics cost nothing extra now; retrofitting a Beta target later compounds the debt. |

## Red Flags

- Images or icons without content descriptions (`""` instead of `null` for decorative)
- Touch targets smaller than 48dp on any target
- Color as the only differentiator
- Missing `heading()` semantics on section titles, or no `mergeDescendants` on compound components
- Hardcoded colors that skip the theme's contrast-safe pairs
- Screen-reader pass done on only one platform (TalkBack but never VoiceOver, or vice versa)
- Accessibility tests living in `androidUnitTest` instead of `commonTest`
- Desktop build never driven with keyboard only

## Verification

- [ ] All meaningful images/icons have descriptive `contentDescription`; decorative ones use `null`
- [ ] All touch targets >= 48dp x 48dp
- [ ] Color is never the only differentiator; custom colors meet 4.5:1 / 3:1
- [ ] Section headings use `semantics { heading() }`; compound components merge descendants
- [ ] Dynamic content uses `liveRegion`; custom state uses `stateDescription`
- [ ] TalkBack pass completed on Android AND VoiceOver pass completed on iOS
- [ ] Desktop screens completed a keyboard-only pass with visible focus
- [ ] commonTest asserts content descriptions and click actions via `runComposeUiTest`
- [ ] `./gradlew :shared:allTests` passes
