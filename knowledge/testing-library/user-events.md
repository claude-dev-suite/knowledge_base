# Testing Library User Events

Comprehensive documentation for user-event library.

**Official Documentation:** https://testing-library.com/docs/user-event/intro

---

## Table of Contents

1. [Setup](#setup)
2. [Pointer Events](#pointer-events)
3. [Keyboard Events](#keyboard-events)
4. [Clipboard Events](#clipboard-events)
5. [Utility Methods](#utility-methods)
6. [Options](#options)
7. [Common Patterns](#common-patterns)

---

## Setup

```typescript
import { render, screen } from '@testing-library/react';
import userEvent from '@testing-library/user-event';

describe('Component', () => {
  it('handles user interaction', async () => {
    // Always create a new user instance
    const user = userEvent.setup();

    render(<MyComponent />);

    // All user methods are async
    await user.click(screen.getByRole('button'));
  });
});
```

### Setup Options

```typescript
const user = userEvent.setup({
  // Delay between actions (default: 0)
  delay: null, // null = real delay, 0 = no delay

  // Custom document
  document: window.document,

  // Keyboard layout
  keyboardMap: keyboardKey,

  // Pointer events configuration
  pointerEventsCheck: PointerEventsCheckLevel.EachTarget,

  // Skip auto-close for select
  skipAutoClose: false,

  // Skip click on focus
  skipClick: false,

  // Skip hover
  skipHover: false,

  // Write to clipboard
  writeToClipboard: true,
});
```

---

## Pointer Events

### click

```typescript
const user = userEvent.setup();

// Basic click
await user.click(screen.getByRole('button'));

// Click with options
await user.click(element, {
  // Keyboard modifiers
  shiftKey: true,
  ctrlKey: true,
  altKey: true,
  metaKey: true,

  // Mouse button (0=left, 1=middle, 2=right)
  button: 0,

  // Pointer type
  pointerType: 'mouse', // 'mouse' | 'pen' | 'touch'

  // Skip hover (default: false)
  skipHover: true,
});
```

### dblClick

```typescript
// Double click
await user.dblClick(screen.getByRole('button'));
```

### tripleClick

```typescript
// Triple click (selects paragraph)
await user.tripleClick(screen.getByRole('textbox'));
```

### hover / unhover

```typescript
// Hover over element
await user.hover(screen.getByText('Hover me'));

// Move away from element
await user.unhover(screen.getByText('Hover me'));
```

### pointer

```typescript
// Low-level pointer API
await user.pointer({
  keys: '[MouseLeft]',
  target: element,
});

// Multiple pointer actions
await user.pointer([
  { keys: '[MouseLeft>]', target: element }, // Press down
  { coords: { x: 100, y: 100 } }, // Move
  { keys: '[/MouseLeft]' }, // Release
]);

// Drag and drop
await user.pointer([
  { keys: '[MouseLeft>]', target: dragSource },
  { target: dropTarget },
  { keys: '[/MouseLeft]' },
]);
```

---

## Keyboard Events

### type

Types into an element.

```typescript
const user = userEvent.setup();
const input = screen.getByRole('textbox');

// Basic typing
await user.type(input, 'Hello World');

// Type with special keys
await user.type(input, 'Hello{Enter}World');
await user.type(input, '{Backspace}{Backspace}');
await user.type(input, '{selectall}{backspace}'); // Clear

// Skip click (element already focused)
await user.type(input, 'Hello', { skipClick: true });

// Initial selection
await user.type(input, 'replacement', {
  initialSelectionStart: 0,
  initialSelectionEnd: 5, // Replaces first 5 chars
});
```

### Special Keys in type()

```typescript
// Modifier keys
await user.type(input, '{Shift}abc'); // ABC
await user.type(input, '{Control}a'); // Select all
await user.type(input, '{Alt}'); // Alt key

// Navigation
await user.type(input, '{ArrowLeft}');
await user.type(input, '{ArrowRight}');
await user.type(input, '{ArrowUp}');
await user.type(input, '{ArrowDown}');
await user.type(input, '{Home}');
await user.type(input, '{End}');
await user.type(input, '{PageUp}');
await user.type(input, '{PageDown}');

// Editing
await user.type(input, '{Backspace}');
await user.type(input, '{Delete}');
await user.type(input, '{Enter}');
await user.type(input, '{Tab}');
await user.type(input, '{Escape}');
await user.type(input, '{Space}');

// Selection
await user.type(input, '{selectall}');
await user.type(input, '{Shift>}{ArrowRight}{/Shift}'); // Select char

// Clipboard
await user.type(input, '{Control>}c{/Control}'); // Copy
await user.type(input, '{Control>}v{/Control}'); // Paste
await user.type(input, '{Control>}x{/Control}'); // Cut
```

### keyboard

Low-level keyboard API for complex sequences.

```typescript
const user = userEvent.setup();

// Press and release
await user.keyboard('abc'); // Types "abc"

// Special keys
await user.keyboard('{Enter}');
await user.keyboard('{Tab}');
await user.keyboard('{Escape}');

// Key down/up
await user.keyboard('{Shift>}'); // Press Shift down
await user.keyboard('{/Shift}'); // Release Shift
await user.keyboard('{Shift>}ABC{/Shift}'); // Shift + ABC

// Modifiers
await user.keyboard('{Control>}a{/Control}'); // Ctrl+A
await user.keyboard('{Alt>}{Tab}{/Alt}'); // Alt+Tab
await user.keyboard('{Meta>}s{/Meta}'); // Cmd+S (Mac)

// Complex combinations
await user.keyboard('{Control>}{Shift>}p{/Shift}{/Control}'); // Ctrl+Shift+P
```

### clear

Clears an input/textarea.

```typescript
const user = userEvent.setup();
const input = screen.getByRole('textbox');

await user.clear(input);
```

---

## Clipboard Events

### copy

```typescript
const user = userEvent.setup();

// Select and copy
await user.tripleClick(element); // Select all
await user.copy();

// Returns DataTransfer
const dataTransfer = await user.copy();
expect(dataTransfer.getData('text/plain')).toBe('copied text');
```

### cut

```typescript
const user = userEvent.setup();

await user.tripleClick(input);
const dataTransfer = await user.cut();
expect(input).toHaveValue('');
```

### paste

```typescript
const user = userEvent.setup();

// Paste string
await user.click(input);
await user.paste('pasted text');

// Paste DataTransfer
const clipboardData = new DataTransfer();
clipboardData.setData('text/plain', 'custom paste');
await user.paste(clipboardData);
```

---

## Utility Methods

### selectOptions

Select options in a `<select>` element.

```typescript
const user = userEvent.setup();
const select = screen.getByRole('combobox');

// Select by value
await user.selectOptions(select, 'option-value');

// Select by display text
await user.selectOptions(select, 'Option Text');

// Select multiple (for multi-select)
await user.selectOptions(select, ['option1', 'option2']);

// Select by element
const option = screen.getByRole('option', { name: 'Apple' });
await user.selectOptions(select, option);
```

### deselectOptions

```typescript
const user = userEvent.setup();

// Deselect in multi-select
await user.deselectOptions(multiSelect, 'option1');
```

### upload

Upload files to file input.

```typescript
const user = userEvent.setup();
const input = screen.getByLabelText('Upload');

// Single file
const file = new File(['content'], 'file.txt', { type: 'text/plain' });
await user.upload(input, file);

expect(input.files[0]).toBe(file);
expect(input.files).toHaveLength(1);

// Multiple files
const files = [
  new File(['content1'], 'file1.txt', { type: 'text/plain' }),
  new File(['content2'], 'file2.txt', { type: 'text/plain' }),
];
await user.upload(input, files);
```

### tab

Navigate with Tab key.

```typescript
const user = userEvent.setup();

// Tab forward
await user.tab();

// Tab backward
await user.tab({ shift: true });

// Verify focus
expect(screen.getByRole('button')).toHaveFocus();
```

---

## Options

### Global Options

```typescript
const user = userEvent.setup({
  // Delay between characters when typing
  delay: 100,

  // Pointer events check level
  pointerEventsCheck: PointerEventsCheckLevel.Never,
  // PointerEventsCheckLevel.Never - Skip all checks
  // PointerEventsCheckLevel.EachTarget - Check each target (default)

  // Skip initial hover before click
  skipHover: true,

  // Skip click before typing
  skipClick: true,

  // Enable clipboard API
  writeToClipboard: true,
});
```

### Per-Action Options

```typescript
// Click options
await user.click(element, {
  skipHover: true,
  skipPointerEventsCheck: true,
});

// Type options
await user.type(input, 'text', {
  skipClick: true,
  initialSelectionStart: 0,
  initialSelectionEnd: 0,
});
```

---

## Common Patterns

### Form Submission

```typescript
it('submits form with user data', async () => {
  const user = userEvent.setup();
  const onSubmit = vi.fn();

  render(<LoginForm onSubmit={onSubmit} />);

  await user.type(screen.getByLabelText('Email'), 'test@example.com');
  await user.type(screen.getByLabelText('Password'), 'password123');
  await user.click(screen.getByRole('button', { name: 'Login' }));

  expect(onSubmit).toHaveBeenCalledWith({
    email: 'test@example.com',
    password: 'password123',
  });
});
```

### Dropdown Selection

```typescript
it('selects option from dropdown', async () => {
  const user = userEvent.setup();

  render(<CountrySelect />);

  await user.click(screen.getByRole('combobox'));
  await user.selectOptions(
    screen.getByRole('combobox'),
    screen.getByRole('option', { name: 'United States' })
  );

  expect(screen.getByRole('combobox')).toHaveValue('US');
});
```

### Checkbox Toggle

```typescript
it('toggles checkbox', async () => {
  const user = userEvent.setup();

  render(<Checkbox label="Accept terms" />);

  const checkbox = screen.getByRole('checkbox');

  expect(checkbox).not.toBeChecked();
  await user.click(checkbox);
  expect(checkbox).toBeChecked();
  await user.click(checkbox);
  expect(checkbox).not.toBeChecked();
});
```

### Keyboard Navigation

```typescript
it('navigates with keyboard', async () => {
  const user = userEvent.setup();

  render(
    <>
      <input data-testid="first" />
      <input data-testid="second" />
      <button data-testid="third">Submit</button>
    </>
  );

  await user.tab();
  expect(screen.getByTestId('first')).toHaveFocus();

  await user.tab();
  expect(screen.getByTestId('second')).toHaveFocus();

  await user.tab();
  expect(screen.getByTestId('third')).toHaveFocus();
});
```

### Drag and Drop

```typescript
it('drags item to new position', async () => {
  const user = userEvent.setup();

  render(<DraggableList items={['A', 'B', 'C']} />);

  const itemA = screen.getByText('A');
  const itemC = screen.getByText('C');

  await user.pointer([
    { keys: '[MouseLeft>]', target: itemA },
    { target: itemC },
    { keys: '[/MouseLeft]' },
  ]);
});
```

### Autocomplete/Search

```typescript
it('searches and selects from autocomplete', async () => {
  const user = userEvent.setup();

  render(<Autocomplete />);

  await user.type(screen.getByRole('combobox'), 'react');

  // Wait for suggestions
  await screen.findByRole('option', { name: 'React.js' });

  await user.click(screen.getByRole('option', { name: 'React.js' }));

  expect(screen.getByRole('combobox')).toHaveValue('React.js');
});
```

### Modal Interactions

```typescript
it('opens modal and interacts', async () => {
  const user = userEvent.setup();

  render(<ModalTrigger />);

  // Open modal
  await user.click(screen.getByRole('button', { name: 'Open' }));

  // Wait for modal
  const modal = await screen.findByRole('dialog');

  // Interact within modal
  await user.type(within(modal).getByLabelText('Name'), 'John');
  await user.click(within(modal).getByRole('button', { name: 'Save' }));

  // Modal closed
  expect(screen.queryByRole('dialog')).not.toBeInTheDocument();
});
```

### Hover States

```typescript
it('shows tooltip on hover', async () => {
  const user = userEvent.setup();

  render(<ButtonWithTooltip />);

  const button = screen.getByRole('button');

  // Tooltip not visible
  expect(screen.queryByRole('tooltip')).not.toBeInTheDocument();

  // Hover to show
  await user.hover(button);
  expect(await screen.findByRole('tooltip')).toBeInTheDocument();

  // Unhover to hide
  await user.unhover(button);
  await waitForElementToBeRemoved(() => screen.queryByRole('tooltip'));
});
```

---

## Best Practices

### Do

```typescript
// Use userEvent.setup() for each test
const user = userEvent.setup();

// Always await user actions
await user.click(button);

// Use realistic interactions
await user.type(input, 'text'); // Types character by character
```

### Don't

```typescript
// Don't use fireEvent for user interactions
fireEvent.click(button); // Less realistic

// Don't forget await
user.click(button); // Missing await!

// Don't use userEvent directly without setup
await userEvent.click(button); // Use setup() instead
```

---

## Quick Reference

| Method | Description |
|--------|-------------|
| `click(element)` | Click element |
| `dblClick(element)` | Double click |
| `tripleClick(element)` | Triple click |
| `hover(element)` | Hover over |
| `unhover(element)` | Move away |
| `type(element, text)` | Type text |
| `keyboard(text)` | Press keys |
| `clear(element)` | Clear input |
| `selectOptions(select, values)` | Select option |
| `deselectOptions(select, values)` | Deselect option |
| `upload(input, files)` | Upload files |
| `tab()` | Tab navigation |
| `copy()` | Copy selection |
| `cut()` | Cut selection |
| `paste(text)` | Paste content |
