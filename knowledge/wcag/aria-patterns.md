# ARIA Patterns

> Source: https://www.w3.org/WAI/ARIA/apg/patterns/

## When to Use ARIA

1. **First rule**: Don't use ARIA if native HTML works
2. **Use ARIA when**: Creating custom widgets not available in HTML
3. **Always test**: With actual screen readers

## Common Patterns

### Modal Dialog

```html
<div
  role="dialog"
  aria-modal="true"
  aria-labelledby="dialog-title"
  aria-describedby="dialog-desc"
>
  <h2 id="dialog-title">Confirm Delete</h2>
  <p id="dialog-desc">Are you sure you want to delete this item?</p>
  <button>Cancel</button>
  <button>Delete</button>
</div>
```

**JavaScript requirements:**
- Trap focus inside dialog
- Return focus to trigger when closed
- Close on Escape key

### Tab Panel

```html
<div role="tablist" aria-label="Settings">
  <button role="tab" aria-selected="true" aria-controls="panel-1" id="tab-1">
    General
  </button>
  <button role="tab" aria-selected="false" aria-controls="panel-2" id="tab-2" tabindex="-1">
    Privacy
  </button>
</div>

<div role="tabpanel" id="panel-1" aria-labelledby="tab-1">
  <!-- General content -->
</div>
<div role="tabpanel" id="panel-2" aria-labelledby="tab-2" hidden>
  <!-- Privacy content -->
</div>
```

**Keyboard navigation:**
- Arrow keys move between tabs
- Tab key moves to panel content

### Menu

```html
<button aria-haspopup="menu" aria-expanded="false" aria-controls="menu">
  Options
</button>
<ul role="menu" id="menu" hidden>
  <li role="menuitem" tabindex="-1">Edit</li>
  <li role="menuitem" tabindex="-1">Delete</li>
  <li role="separator"></li>
  <li role="menuitem" tabindex="-1">Settings</li>
</ul>
```

**Keyboard navigation:**
- Arrow keys navigate items
- Enter/Space activates item
- Escape closes menu

### Accordion

```html
<div class="accordion">
  <h3>
    <button aria-expanded="true" aria-controls="section1">
      Section 1
    </button>
  </h3>
  <div id="section1" role="region" aria-labelledby="section1-btn">
    <!-- Content -->
  </div>

  <h3>
    <button aria-expanded="false" aria-controls="section2">
      Section 2
    </button>
  </h3>
  <div id="section2" role="region" aria-labelledby="section2-btn" hidden>
    <!-- Content -->
  </div>
</div>
```

### Combobox (Autocomplete)

```html
<label for="city">City</label>
<input
  type="text"
  id="city"
  role="combobox"
  aria-autocomplete="list"
  aria-expanded="true"
  aria-controls="city-listbox"
  aria-activedescendant="option-2"
/>
<ul role="listbox" id="city-listbox">
  <li role="option" id="option-1">New York</li>
  <li role="option" id="option-2" aria-selected="true">Los Angeles</li>
  <li role="option" id="option-3">Chicago</li>
</ul>
```

### Alert

```html
<!-- Static alert -->
<div role="alert">
  Your session will expire in 5 minutes.
</div>

<!-- Dynamic alert (injected) -->
<div role="alert" aria-live="assertive">
  Form submitted successfully!
</div>
```

### Progress Bar

```html
<!-- Determinate -->
<div
  role="progressbar"
  aria-valuenow="75"
  aria-valuemin="0"
  aria-valuemax="100"
  aria-label="Upload progress"
>
  75%
</div>

<!-- Indeterminate -->
<div
  role="progressbar"
  aria-label="Loading"
>
  Loading...
</div>
```

### Slider

```html
<label id="volume-label">Volume</label>
<div
  role="slider"
  tabindex="0"
  aria-labelledby="volume-label"
  aria-valuemin="0"
  aria-valuemax="100"
  aria-valuenow="50"
  aria-valuetext="50%"
>
</div>
```

### Tooltip

```html
<button aria-describedby="tooltip">
  Settings
</button>
<div role="tooltip" id="tooltip">
  Configure application settings
</div>
```

### Breadcrumb

```html
<nav aria-label="Breadcrumb">
  <ol>
    <li><a href="/">Home</a></li>
    <li><a href="/products">Products</a></li>
    <li><a href="/products/shoes" aria-current="page">Shoes</a></li>
  </ol>
</nav>
```

## Live Regions

### aria-live

| Value | Behavior |
|-------|----------|
| `polite` | Announce when idle |
| `assertive` | Announce immediately |
| `off` | Don't announce |

### aria-atomic

```html
<!-- Announce entire region on change -->
<div aria-live="polite" aria-atomic="true">
  Cart: 3 items, $50.00
</div>
```

### Status Messages

```html
<!-- Form success -->
<div role="status" aria-live="polite">
  Form submitted successfully
</div>

<!-- Loading -->
<div role="status" aria-live="polite" aria-busy="true">
  Loading...
</div>
```

## Common ARIA Attributes

| Attribute | Purpose |
|-----------|---------|
| `aria-label` | Label for element |
| `aria-labelledby` | Reference to labeling element |
| `aria-describedby` | Reference to description |
| `aria-expanded` | Expandable state |
| `aria-pressed` | Toggle button state |
| `aria-selected` | Selection state |
| `aria-hidden` | Hide from AT |
| `aria-disabled` | Disabled state |
| `aria-current` | Current item in set |
| `aria-invalid` | Validation state |
| `aria-required` | Required field |

## Don'ts

```html
<!-- DON'T: Add ARIA to native elements -->
<button role="button">Submit</button>

<!-- DON'T: Use aria-hidden on focusable elements -->
<button aria-hidden="true">Hidden but focusable</button>

<!-- DON'T: Conflict with native semantics -->
<button role="link">Click here</button>

<!-- DON'T: Use abstract roles -->
<div role="widget">...</div>
```
