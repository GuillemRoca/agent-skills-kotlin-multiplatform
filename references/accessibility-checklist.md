# Accessibility Checklist — Compose Multiplatform Reference

Semantics declared in `commonMain` feed TalkBack (Android) AND VoiceOver (iOS) from the same code — write once, then verify with each platform's assistive technology.

## Content Descriptions

- [ ] **All meaningful images/icons** have `contentDescription`
- [ ] **Decorative elements** use `contentDescription = null`
- [ ] **Descriptions are functional**, not visual ("Delete task", not "red trash can")
- [ ] **Dynamic descriptions** include relevant context ("Profile photo of John")
- [ ] **No empty strings** for content descriptions (use `null` for decorative)
- [ ] **IconButtons** — description on the Icon child, not the button

```kotlin
// Meaningful icon
Icon(Icons.Default.Delete, contentDescription = "Delete task")

// Decorative icon
Icon(Icons.Default.Circle, contentDescription = null)

// Dynamic description (compose resources)
Image(
    painter = painterResource(Res.drawable.avatar),
    contentDescription = "Profile photo of ${user.name}",
)
```

## Touch Targets

- [ ] **All interactive elements** >= 48dp x 48dp (also satisfies the iOS HIG 44pt minimum)
- [ ] **Visual size can differ** from touch target (icon 24dp, target 48dp)
- [ ] **Spacing between targets** sufficient to prevent mis-taps

```kotlin
IconButton(
    onClick = onAction,
    modifier = Modifier.size(48.dp), // Touch target
) {
    Icon(
        imageVector = Icons.Default.Star,
        contentDescription = "Favorite",
        modifier = Modifier.size(24.dp), // Visual size
    )
}

// For custom clickable elements:
Modifier.defaultMinSize(minHeight = 48.dp, minWidth = 48.dp)
```

## Color & Contrast

- [ ] **Text contrast** >= 4.5:1 (normal text) or >= 3:1 (large text, 18sp+)
- [ ] **UI component contrast** >= 3:1 against adjacent colors
- [ ] **No color-only information** — use icons, text, or patterns as additional cues
- [ ] **Material theme tokens** used (they guarantee contrast compliance)
- [ ] **Both light and dark themes** tested for contrast — on every target, not just Android

```kotlin
// BAD: color-only status
Text(color = if (error) Color.Red else Color.Black)

// GOOD: color + icon + text
Row {
    if (error) Icon(Icons.Default.Error, contentDescription = null)
    Text(
        text = if (error) "Error: $message" else message,
        color = if (error) MaterialTheme.colorScheme.error
                else MaterialTheme.colorScheme.onSurface,
    )
}
```

## Compose Semantics

These modifiers live in `commonMain` and map to both TalkBack and VoiceOver.

- [ ] **Section headings** use `semantics { heading() }`
- [ ] **Compound components** use `semantics(mergeDescendants = true)`
- [ ] **Dynamic content** uses `liveRegion = LiveRegionMode.Polite`
- [ ] **Custom states** use `stateDescription` (not just checked/unchecked)
- [ ] **Custom actions** provided via `customActions` where appropriate

```kotlin
// Heading — enables heading navigation in TalkBack and the VoiceOver rotor
Text(
    "Tasks",
    modifier = Modifier.semantics { heading() },
)

// Merged component — announced as one element by both screen readers
Row(modifier = Modifier.semantics(mergeDescendants = true) { }) {
    Icon(Icons.Default.Task, contentDescription = null)
    Column {
        Text("Task title")
        Text("Due tomorrow")
    }
}

// Live region
Text(
    "$count items remaining",
    modifier = Modifier.semantics { liveRegion = LiveRegionMode.Polite },
)
```

## Focus & Navigation

- [ ] **Focus order** matches logical reading order (top-to-bottom, start-to-end)
- [ ] **No focus traps** — user can navigate away from every element
- [ ] **Dialog focus** — focus moves to dialog when opened, returns on dismiss
- [ ] **Bottom sheet focus** — focus contained within sheet when open
- [ ] **Scrollable content** reachable via screen-reader swipe navigation (TalkBack and VoiceOver)
- [ ] **Desktop keyboard focus** — every interactive element reachable and activatable without a mouse

```kotlin
// Focus management for dialogs
val focusRequester = remember { FocusRequester() }
LaunchedEffect(Unit) { focusRequester.requestFocus() }

AlertDialog(
    title = {
        Text("Confirm", modifier = Modifier.focusRequester(focusRequester))
    },
    // ...
)
```

## Forms & Input

- [ ] **Text fields** have visible labels (not just placeholder text)
- [ ] **Error messages** associated with the field they describe
- [ ] **Required fields** indicated (not by color alone)
- [ ] **Input types** set correctly (`keyboardType`, `imeAction`)
- [ ] **Auto-fill** supported where appropriate

```kotlin
OutlinedTextField(
    value = title,
    onValueChange = onTitleChange,
    label = { Text("Task title") }, // Accessible label
    isError = titleError != null,
    supportingText = titleError?.let { { Text(it) } },
    keyboardOptions = KeyboardOptions(
        keyboardType = KeyboardType.Text,
        imeAction = ImeAction.Done,
    ),
)
```

## Testing

### TalkBack (Android, Manual)

- [ ] **Enable** TalkBack: Settings → Accessibility → TalkBack
- [ ] **Navigate** through every screen by swiping right
- [ ] **Verify** all announcements make sense without seeing the screen
- [ ] **Check** all interactive elements are reachable
- [ ] **Verify** no elements are skipped or read in wrong order
- [ ] **Test** custom actions (double-tap, long-press)

### VoiceOver (iOS, Manual)

- [ ] **Enable** VoiceOver: Settings → Accessibility → VoiceOver (or triple-click Accessibility Shortcut)
- [ ] **Navigate** through every screen by swiping right — the same `commonMain` semantics must announce correctly here too
- [ ] **Verify** heading navigation via the rotor works (requires `heading()` semantics)
- [ ] **Check** merged components are announced as single elements
- [ ] **Test** custom actions via the rotor

### Desktop Keyboard Navigation (Manual)

- [ ] **Traverse** every screen with Tab/Shift+Tab — order matches reading order
- [ ] **Activate** every interactive element with Enter/Space
- [ ] **Verify** a visible focus indicator on the focused element
- [ ] **Dismiss** dialogs and sheets with Esc
- [ ] **Confirm** no mouse-only interactions exist

### Web (Caveats)

- [ ] **Know the limitation** — Compose Multiplatform for web renders to a canvas; screen-reader support is limited compared to DOM-based apps and evolving between releases
- [ ] **Verify** the current CMP release notes for the accessibility status of the web target before making claims
- [ ] **Test** with a browser screen reader on the deployed js/wasm bundle; treat web accessibility as an explicit launch checklist item, not an assumption

### Accessibility Scanner (Android, Automated)

- [ ] **Install** Accessibility Scanner from Play Store
- [ ] **Run** on every screen
- [ ] **Fix** all "Error" findings
- [ ] **Review** "Warning" findings

### Compose UI Tests (commonTest)

```kotlin
@OptIn(ExperimentalTestApi::class)
class AccessibilityTest {

    @Test
    fun deleteButtonHasAccessibleDescription() = runComposeUiTest {
        setContent {
            TaskItem(task = sampleTask, onToggle = {}, onDelete = {})
        }

        onNodeWithContentDescription("Delete task")
            .assertExists()
            .assertHasClickAction()
    }

    @Test
    fun headingHasSemanticsRole() = runComposeUiTest {
        setContent {
            Text("Tasks", modifier = Modifier.semantics { heading() })
        }

        onNodeWithText("Tasks")
            .assert(SemanticsMatcher.keyIsDefined(SemanticsProperties.Heading))
    }

    @Test
    fun touchTargetsMeetMinimumSize() = runComposeUiTest {
        setContent {
            IconButton(onClick = {}) {
                Icon(Icons.Default.Add, contentDescription = "Add task")
            }
        }

        onNodeWithContentDescription("Add task")
            .assertTouchHeightIsAtLeast(48.dp)
            .assertTouchWidthIsAtLeast(48.dp)
    }
}
```

## Common Violations

| Violation | Impact | Fix |
|-----------|--------|-----|
| Missing content description | Screen reader skips element (TalkBack and VoiceOver) | Add descriptive `contentDescription` |
| Touch target < 48dp | Hard to tap for motor disabilities | `Modifier.defaultMinSize(48.dp)` |
| Color-only information | Invisible to color-blind users | Add icon, text, or pattern |
| Missing heading semantics | Can't navigate by headings (TalkBack) or rotor (VoiceOver) | `semantics { heading() }` |
| No live region | Dynamic changes not announced | `liveRegion = LiveRegionMode.Polite` |
| Focus trap in overlay | Can't navigate away | Proper dismiss handling |
| Placeholder as label | Label disappears on input | Use `label` parameter |
| Low contrast text | Unreadable for low vision | Use Material theme tokens |
| Mouse-only desktop interactions | Unusable without a pointer | Keyboard focus + Enter/Space activation |
