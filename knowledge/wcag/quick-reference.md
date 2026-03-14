# WCAG 2.2 Quick Reference

> Source: https://www.w3.org/WAI/WCAG22/quickref/

## Level A Requirements

### Perceivable

| Criterion | Requirement |
|-----------|-------------|
| 1.1.1 Non-text Content | All images have alt text |
| 1.2.1 Audio-only/Video-only | Provide transcript or description |
| 1.2.2 Captions | Provide captions for video |
| 1.2.3 Audio Description | Describe visual content in audio |
| 1.3.1 Info and Relationships | Structure conveyed programmatically |
| 1.3.2 Meaningful Sequence | Reading order makes sense |
| 1.3.3 Sensory Characteristics | Don't rely only on shape/color |
| 1.4.1 Use of Color | Color not only visual means |
| 1.4.2 Audio Control | Can pause/stop audio |

### Operable

| Criterion | Requirement |
|-----------|-------------|
| 2.1.1 Keyboard | All functionality via keyboard |
| 2.1.2 No Keyboard Trap | Can navigate away with keyboard |
| 2.1.4 Character Key Shortcuts | Can turn off single-key shortcuts |
| 2.2.1 Timing Adjustable | Time limits can be extended |
| 2.2.2 Pause, Stop, Hide | Can control moving content |
| 2.3.1 Three Flashes | No content flashes > 3 times/sec |
| 2.4.1 Bypass Blocks | Skip navigation links |
| 2.4.2 Page Titled | Descriptive page titles |
| 2.4.3 Focus Order | Logical tab order |
| 2.4.4 Link Purpose | Link text describes destination |
| 2.5.1 Pointer Gestures | Single pointer alternative |
| 2.5.2 Pointer Cancellation | Can abort/undo pointer actions |
| 2.5.3 Label in Name | Visible label in accessible name |
| 2.5.4 Motion Actuation | Motion can be disabled |

### Understandable

| Criterion | Requirement |
|-----------|-------------|
| 3.1.1 Language of Page | HTML lang attribute set |
| 3.2.1 On Focus | No unexpected changes on focus |
| 3.2.2 On Input | No unexpected changes on input |
| 3.3.1 Error Identification | Errors clearly identified |
| 3.3.2 Labels or Instructions | Form inputs have labels |

### Robust

| Criterion | Requirement |
|-----------|-------------|
| 4.1.2 Name, Role, Value | Custom controls have ARIA |

## Level AA Requirements

### Perceivable

| Criterion | Requirement |
|-----------|-------------|
| 1.2.4 Captions (Live) | Live captions for video |
| 1.2.5 Audio Description | Audio descriptions for video |
| 1.3.4 Orientation | Works in any orientation |
| 1.3.5 Identify Input Purpose | Input autocomplete attributes |
| 1.4.3 Contrast (Minimum) | 4.5:1 text, 3:1 large text |
| 1.4.4 Resize Text | 200% zoom without loss |
| 1.4.5 Images of Text | Use real text, not images |
| 1.4.10 Reflow | No horizontal scroll at 320px |
| 1.4.11 Non-text Contrast | 3:1 for UI and graphics |
| 1.4.12 Text Spacing | Works with increased spacing |
| 1.4.13 Content on Hover/Focus | Dismissible, hoverable, persistent |

### Operable

| Criterion | Requirement |
|-----------|-------------|
| 2.4.5 Multiple Ways | Multiple ways to find pages |
| 2.4.6 Headings and Labels | Descriptive headings/labels |
| 2.4.7 Focus Visible | Visible focus indicator |
| 2.4.11 Focus Not Obscured | Focus partially visible |
| 2.4.13 Focus Appearance | Adequate focus indicator |
| 2.5.7 Dragging Movements | Single pointer alternative to drag |
| 2.5.8 Target Size (Minimum) | 24×24px targets |

### Understandable

| Criterion | Requirement |
|-----------|-------------|
| 3.1.2 Language of Parts | Language changes marked |
| 3.2.3 Consistent Navigation | Same order across pages |
| 3.2.4 Consistent Identification | Same functionality, same name |
| 3.2.6 Consistent Help | Help in same location |
| 3.3.3 Error Suggestion | Suggest corrections |
| 3.3.4 Error Prevention (Legal) | Confirm important actions |
| 3.3.7 Redundant Entry | Don't re-ask for info |

### Robust

| Criterion | Requirement |
|-----------|-------------|
| 4.1.3 Status Messages | Announce status to AT |

## Common Code Patterns

### Images

```html
<!-- Informative image -->
<img src="chart.png" alt="Sales increased 25% in Q3" />

<!-- Decorative image -->
<img src="divider.png" alt="" role="presentation" />

<!-- Complex image -->
<img src="flowchart.png" alt="Process flowchart" aria-describedby="flowchart-desc" />
<div id="flowchart-desc">Step 1: Submit form. Step 2: Review...</div>
```

### Form Labels

```html
<!-- Explicit label -->
<label for="email">Email</label>
<input type="email" id="email" />

<!-- With hint text -->
<label for="password">Password</label>
<input type="password" id="password" aria-describedby="pwd-hint" />
<span id="pwd-hint">Must be 8+ characters</span>

<!-- Error state -->
<label for="name">Name</label>
<input type="text" id="name" aria-invalid="true" aria-describedby="name-error" />
<span id="name-error" role="alert">Name is required</span>
```

### Skip Link

```html
<a href="#main-content" class="skip-link">Skip to main content</a>
<!-- Navigation here -->
<main id="main-content">
  <!-- Content -->
</main>
```

### Focus Visible CSS

```css
:focus-visible {
  outline: 3px solid #005fcc;
  outline-offset: 2px;
}

/* Remove outline for mouse users */
:focus:not(:focus-visible) {
  outline: none;
}
```

### Color Contrast

| Type | Minimum Ratio |
|------|---------------|
| Normal text | 4.5:1 |
| Large text (18pt+) | 3:1 |
| UI components | 3:1 |
| Graphics | 3:1 |

### Target Size

```css
.button, .link {
  min-width: 44px;  /* Recommended */
  min-height: 44px;
  /* Or 24×24px for WCAG 2.2 AA minimum */
}
```
