# Testing Library Queries Reference

Comprehensive documentation for Testing Library query methods.

**Official Documentation:** https://testing-library.com/docs/queries/about

---

## Table of Contents

1. [Query Types](#query-types)
2. [Query Priority](#query-priority)
3. [ByRole](#byrole)
4. [ByLabelText](#bylabeltext)
5. [ByPlaceholderText](#byplaceholdertext)
6. [ByText](#bytext)
7. [ByDisplayValue](#bydisplayvalue)
8. [ByAltText](#byalttext)
9. [ByTitle](#bytitle)
10. [ByTestId](#bytestid)
11. [TextMatch](#textmatch)
12. [Within](#within)
13. [Configuration](#configuration)

---

## Query Types

### Variants

| Prefix | 0 Matches | 1 Match | >1 Matches | Retry | Use Case |
|--------|-----------|---------|------------|-------|----------|
| `getBy` | throw | return | throw | No | Element should exist |
| `queryBy` | null | return | throw | No | Element may not exist |
| `findBy` | throw | return | throw | Yes | Element will appear async |
| `getAllBy` | throw | array | array | No | Multiple elements |
| `queryAllBy` | [] | array | array | No | Multiple, may not exist |
| `findAllBy` | throw | array | array | Yes | Multiple async |

### Usage Examples

```typescript
import { render, screen } from '@testing-library/react';

// getBy - throws if not found
const button = screen.getByRole('button');

// queryBy - returns null if not found (good for asserting absence)
expect(screen.queryByText('Loading')).not.toBeInTheDocument();

// findBy - waits for element (returns Promise)
const asyncElement = await screen.findByText('Loaded');

// getAllBy - returns array
const listItems = screen.getAllByRole('listitem');

// queryAllBy - returns empty array if none found
const items = screen.queryAllByTestId('item');

// findAllBy - async, returns array
const asyncItems = await screen.findAllByRole('option');
```

---

## Query Priority

Use queries in this priority order for accessible, maintainable tests:

### 1. Accessible to Everyone

```typescript
// getByRole - HIGHEST PRIORITY
// Queries based on accessibility tree
screen.getByRole('button', { name: 'Submit' });
screen.getByRole('heading', { level: 1 });
screen.getByRole('textbox', { name: /email/i });
screen.getByRole('checkbox', { checked: true });

// getByLabelText - Great for form fields
screen.getByLabelText('Username');
screen.getByLabelText(/password/i);

// getByPlaceholderText - When no label exists
screen.getByPlaceholderText('Enter your email');

// getByText - For non-interactive elements
screen.getByText('Welcome to our app');
screen.getByText(/hello/i);

// getByDisplayValue - Current value of form elements
screen.getByDisplayValue('current input value');
```

### 2. Semantic Queries

```typescript
// getByAltText - For images
screen.getByAltText('Company logo');

// getByTitle - For elements with title attribute
screen.getByTitle('Close');
```

### 3. Test IDs (Last Resort)

```typescript
// getByTestId - Only when other queries don't work
screen.getByTestId('custom-element');
```

---

## ByRole

Most recommended query - uses accessibility roles.

### Basic Usage

```typescript
// Button
screen.getByRole('button');
screen.getByRole('button', { name: 'Submit' });
screen.getByRole('button', { name: /submit/i });

// Links
screen.getByRole('link', { name: 'Home' });

// Headings
screen.getByRole('heading');
screen.getByRole('heading', { level: 1 });
screen.getByRole('heading', { name: 'Welcome' });

// Form elements
screen.getByRole('textbox'); // input[type="text"], textarea
screen.getByRole('checkbox');
screen.getByRole('radio');
screen.getByRole('combobox'); // select
screen.getByRole('spinbutton'); // input[type="number"]
screen.getByRole('slider'); // input[type="range"]
```

### Options

```typescript
screen.getByRole('button', {
  // Accessible name (label)
  name: 'Submit',
  name: /submit/i,
  name: (content, element) => content.startsWith('Submit'),

  // Hidden elements (default: false)
  hidden: true,

  // For checkboxes and radio buttons
  checked: true,
  pressed: true, // toggle buttons

  // For combobox/listbox
  selected: true,
  expanded: true,

  // For headings
  level: 2,

  // Current state
  current: true,
  current: 'page', // 'page' | 'step' | 'location' | 'date' | 'time'

  // Description
  description: 'Helper text',
  description: /helper/i,

  // Busy state
  busy: true,

  // Value (for sliders, spinbuttons)
  value: {
    now: 5,
    min: 0,
    max: 10,
    text: 'medium'
  }
});
```

### Common Roles

| Role | Elements |
|------|----------|
| `button` | `<button>`, `<input type="button">`, `[role="button"]` |
| `link` | `<a href>`, `[role="link"]` |
| `textbox` | `<input type="text">`, `<textarea>`, `[role="textbox"]` |
| `checkbox` | `<input type="checkbox">`, `[role="checkbox"]` |
| `radio` | `<input type="radio">`, `[role="radio"]` |
| `heading` | `<h1>`-`<h6>`, `[role="heading"]` |
| `list` | `<ul>`, `<ol>`, `[role="list"]` |
| `listitem` | `<li>`, `[role="listitem"]` |
| `img` | `<img>`, `[role="img"]` |
| `navigation` | `<nav>`, `[role="navigation"]` |
| `main` | `<main>`, `[role="main"]` |
| `combobox` | `<select>`, `[role="combobox"]` |
| `option` | `<option>`, `[role="option"]` |
| `table` | `<table>`, `[role="table"]` |
| `row` | `<tr>`, `[role="row"]` |
| `cell` | `<td>`, `[role="cell"]` |
| `dialog` | `<dialog>`, `[role="dialog"]` |
| `alert` | `[role="alert"]` |
| `tab` | `[role="tab"]` |
| `tabpanel` | `[role="tabpanel"]` |
| `menu` | `[role="menu"]` |
| `menuitem` | `[role="menuitem"]` |

---

## ByLabelText

Finds form elements by their associated label.

```typescript
// Label with htmlFor
// <label for="username">Username</label>
// <input id="username" />
screen.getByLabelText('Username');

// Wrapping label
// <label>Username <input /></label>
screen.getByLabelText('Username');

// aria-label
// <input aria-label="Username" />
screen.getByLabelText('Username');

// aria-labelledby
// <span id="label">Username</span>
// <input aria-labelledby="label" />
screen.getByLabelText('Username');

// Options
screen.getByLabelText('Username', {
  selector: 'input', // Specify element type
  exact: false, // Allow partial match
});
```

---

## ByPlaceholderText

```typescript
// <input placeholder="Enter your email" />
screen.getByPlaceholderText('Enter your email');
screen.getByPlaceholderText(/enter/i);
```

---

## ByText

Finds elements by their text content.

```typescript
// <span>Hello World</span>
screen.getByText('Hello World');
screen.getByText(/hello/i);

// Options
screen.getByText('Hello World', {
  exact: false, // Partial match
  selector: 'span', // Limit to specific element
  ignore: 'script, style', // Ignore certain elements
  normalizer: getDefaultNormalizer({
    trim: false,
    collapseWhitespace: false,
  }),
});

// Function matcher
screen.getByText((content, element) => {
  return element?.tagName.toLowerCase() === 'span' &&
         content.startsWith('Hello');
});
```

---

## ByDisplayValue

Finds form elements by their current value.

```typescript
// <input value="John" />
screen.getByDisplayValue('John');

// <select><option selected>Apple</option></select>
screen.getByDisplayValue('Apple');

// <textarea>Some text</textarea>
screen.getByDisplayValue('Some text');

// Regex
screen.getByDisplayValue(/john/i);
```

---

## ByAltText

For images and areas with alt text.

```typescript
// <img alt="Profile picture" />
screen.getByAltText('Profile picture');
screen.getByAltText(/profile/i);

// <area alt="Click here" />
screen.getByAltText('Click here');
```

---

## ByTitle

Finds elements with title attribute or SVG title element.

```typescript
// <span title="Close">X</span>
screen.getByTitle('Close');

// <svg><title>Warning</title></svg>
screen.getByTitle('Warning');
```

---

## ByTestId

Last resort query using data-testid attribute.

```typescript
// <div data-testid="custom-element" />
screen.getByTestId('custom-element');

// Custom attribute (configure in setup)
configure({ testIdAttribute: 'data-my-test-id' });
```

---

## TextMatch

Options for text matching used across all queries.

### String

```typescript
// Exact match (default)
screen.getByText('Hello World');
```

### Regex

```typescript
// Case-insensitive
screen.getByText(/hello world/i);

// Partial match
screen.getByText(/hello/);

// Start of string
screen.getByText(/^hello/i);
```

### Function

```typescript
screen.getByText((content, element) => {
  const hasText = (node: Element) => node.textContent === 'Hello World';
  const nodeHasText = hasText(element!);
  const childrenDontHaveText = Array.from(element!.children || [])
    .every(child => !hasText(child as Element));
  return nodeHasText && childrenDontHaveText;
});
```

### Options

```typescript
screen.getByText('Hello World', {
  // Exact match (default: true)
  exact: false,

  // Custom normalizer
  normalizer: getDefaultNormalizer({
    trim: true, // Remove leading/trailing whitespace
    collapseWhitespace: true, // Collapse multiple spaces
  }),
});
```

---

## Within

Query within a specific container.

```typescript
import { within, screen } from '@testing-library/react';

// Get container
const form = screen.getByRole('form');

// Query within form
const submitButton = within(form).getByRole('button', { name: 'Submit' });
const emailInput = within(form).getByLabelText('Email');

// Useful for duplicate elements
const rows = screen.getAllByRole('row');
const firstRowButton = within(rows[0]).getByRole('button');
const secondRowButton = within(rows[1]).getByRole('button');
```

---

## Configuration

```typescript
import { configure, getDefaultNormalizer } from '@testing-library/react';

configure({
  // Custom test ID attribute
  testIdAttribute: 'data-my-test-id',

  // Async utilities timeout (default: 1000ms)
  asyncUtilTimeout: 5000,

  // Compute style for hidden check
  computedStyleSupportsPseudoElements: true,

  // Default hidden option for getByRole
  defaultHidden: false,

  // Show original stack trace
  showOriginalStackTrace: false,

  // Throw suggestions
  throwSuggestions: true,

  // Custom getElementError
  getElementError: (message, container) => {
    const error = new Error(
      [message, prettyDOM(container)].filter(Boolean).join('\n\n')
    );
    error.name = 'TestingLibraryElementError';
    return error;
  },
});
```

---

## Best Practices

### Do

```typescript
// Use getByRole when possible
screen.getByRole('button', { name: 'Submit' });

// Use semantic queries
screen.getByLabelText('Email');

// Use regex for flexibility
screen.getByText(/welcome/i);

// Use within for scoped queries
within(form).getByRole('textbox');
```

### Don't

```typescript
// Don't use container.querySelector
container.querySelector('.submit-button'); // Bad

// Don't rely on implementation details
screen.getByTestId('button-1'); // Less preferred

// Don't use getByText for form elements
screen.getByText('Email'); // Use getByLabelText instead
```

---

## Quick Reference

| Query | Use Case |
|-------|----------|
| `getByRole` | Any element with accessible role |
| `getByLabelText` | Form fields with labels |
| `getByPlaceholderText` | Inputs with placeholder |
| `getByText` | Non-interactive text content |
| `getByDisplayValue` | Current form values |
| `getByAltText` | Images |
| `getByTitle` | Elements with title |
| `getByTestId` | Last resort |
