# Playwright Assertions

Playwright Test includes auto-retrying assertions that remove flakiness by waiting until the expected condition is met. This comprehensive guide covers all assertion types, patterns, and best practices.

## Table of Contents

1. [Auto-Retrying Assertions Concept](#auto-retrying-assertions-concept)
2. [Visibility Assertions](#visibility-assertions)
3. [State Assertions](#state-assertions)
4. [Text Assertions](#text-assertions)
5. [Value Assertions](#value-assertions)
6. [Attribute Assertions](#attribute-assertions)
7. [Count Assertions](#count-assertions)
8. [Page Assertions](#page-assertions)
9. [Screenshot Assertions](#screenshot-assertions)
10. [Negation](#negation)
11. [Soft Assertions](#soft-assertions)
12. [Polling Assertions](#polling-assertions)
13. [Custom Timeout](#custom-timeout)
14. [toPass for Retry](#topass-for-retry)
15. [Generic Matchers](#generic-matchers)
16. [Assertion Options](#assertion-options)
17. [Best Practices](#best-practices)

---

## Auto-Retrying Assertions Concept

### Overview

Playwright assertions are designed to handle the dynamic nature of web applications. Unlike traditional assertions that check immediately and fail if the condition is not met, Playwright's auto-retrying assertions will repeatedly check the condition until it passes or times out.

```typescript
import { expect } from '@playwright/test';

// This will retry until the element is visible or timeout
await expect(locator).toBeVisible();
```

### How Auto-Retry Works

1. **Initial Check**: Playwright checks if the assertion condition is met
2. **Retry Loop**: If not met, it waits briefly and checks again
3. **Success**: Returns immediately when condition is satisfied
4. **Timeout**: Fails after the default timeout (5 seconds by default)

```typescript
// Behind the scenes, this is similar to:
// while (timeout not reached) {
//   if (element.isVisible()) return success;
//   wait(retryInterval);
// }
// throw TimeoutError;
```

### Why Auto-Retry Matters

Web applications are asynchronous. Elements may:
- Load after network requests complete
- Appear after animations finish
- Change state based on JavaScript execution
- Be rendered after React/Vue/Angular hydration

```typescript
// BAD: Manual wait - brittle and slow
await page.waitForTimeout(2000);
expect(await locator.isVisible()).toBe(true);

// GOOD: Auto-retrying assertion - reliable and fast
await expect(locator).toBeVisible();
```

### Auto-Retrying vs Non-Retrying Assertions

| Auto-Retrying (Async)                | Non-Retrying (Sync)        |
| ------------------------------------ | -------------------------- |
| `await expect(locator).toBeVisible()`| `expect(value).toBe(5)`    |
| Waits for condition                  | Checks immediately         |
| Used with locators/pages             | Used with static values    |
| Returns Promise                      | Returns void               |

```typescript
// Auto-retrying - use with locators
await expect(page.locator('.status')).toHaveText('Complete');

// Non-retrying - use with already-fetched values
const count = await page.locator('.item').count();
expect(count).toBeGreaterThan(0);
```

### Default Timeout Configuration

```typescript
// playwright.config.ts
import { defineConfig } from '@playwright/test';

export default defineConfig({
  expect: {
    // Global timeout for all expect() calls
    timeout: 10000, // 10 seconds
  },
});
```

### List of Auto-Retrying Assertions

These assertions retry until success or timeout:

**Locator Assertions:**
- `toBeAttached()`
- `toBeChecked()`
- `toBeDisabled()`
- `toBeEditable()`
- `toBeEmpty()`
- `toBeEnabled()`
- `toBeFocused()`
- `toBeHidden()`
- `toBeInViewport()`
- `toBeVisible()`
- `toContainText()`
- `toHaveAccessibleDescription()`
- `toHaveAccessibleName()`
- `toHaveAttribute()`
- `toHaveClass()`
- `toHaveCount()`
- `toHaveCSS()`
- `toHaveId()`
- `toHaveJSProperty()`
- `toHaveRole()`
- `toHaveScreenshot()`
- `toHaveText()`
- `toHaveValue()`
- `toHaveValues()`

**Page Assertions:**
- `toHaveScreenshot()`
- `toHaveTitle()`
- `toHaveURL()`

**API Response Assertions:**
- `toBeOK()`

---

## Visibility Assertions

### toBeVisible()

Ensures the element is visible to the user. An element is considered visible when:
- It has non-empty bounding box
- It does not have `visibility: hidden` CSS property
- It does not have `display: none` CSS property
- It does not have `opacity: 0` CSS property

```typescript
// Basic visibility check
await expect(page.locator('.dialog')).toBeVisible();

// Wait for element to become visible after action
await page.click('#open-modal');
await expect(page.locator('.modal')).toBeVisible();

// With custom timeout
await expect(page.locator('.slow-loading')).toBeVisible({ timeout: 15000 });
```

#### Visibility Check Details

```typescript
// Element must be in DOM and visible
const submitButton = page.getByRole('button', { name: 'Submit' });
await expect(submitButton).toBeVisible();

// Check visibility of specific element among many
await expect(page.locator('.notification').first()).toBeVisible();

// Visibility with locator chaining
await expect(
  page.locator('.card').filter({ hasText: 'Premium' }).locator('.badge')
).toBeVisible();
```

### toBeHidden()

Ensures the element is not visible to the user. This is the opposite of `toBeVisible()`.

```typescript
// Basic hidden check
await expect(page.locator('.loading-spinner')).toBeHidden();

// Wait for element to hide after action
await page.click('#close-modal');
await expect(page.locator('.modal')).toBeHidden();

// Verify error message is hidden initially
await expect(page.locator('.error-message')).toBeHidden();
await page.fill('#email', 'invalid');
await expect(page.locator('.error-message')).toBeVisible();
```

#### Hidden vs Not Attached

```typescript
// toBeHidden() passes for:
// 1. Elements not in DOM
// 2. Elements with display: none
// 3. Elements with visibility: hidden
// 4. Elements with opacity: 0
// 5. Elements with zero dimensions

// If you need to distinguish, use toBeAttached()
await expect(page.locator('.removed-element')).not.toBeAttached();
await expect(page.locator('.hidden-element')).toBeAttached();
await expect(page.locator('.hidden-element')).toBeHidden();
```

### toBeAttached() and not.toBeAttached()

Checks whether the element is attached to the DOM (present in the document).

```typescript
// Element exists in DOM (even if hidden)
await expect(page.locator('#hidden-form')).toBeAttached();

// Element is removed from DOM
await page.click('#delete-item');
await expect(page.locator('.deleted-item')).not.toBeAttached();

// With attached option
await expect(page.locator('.dynamic')).toBeAttached({ attached: true });
await expect(page.locator('.dynamic')).toBeAttached({ attached: false }); // Same as not.toBeAttached()
```

### toBeInViewport()

Ensures the element is within the visible viewport.

```typescript
// Check if element is in viewport
await expect(page.locator('#footer')).toBeInViewport();

// Check with ratio (percentage visible)
await expect(page.locator('.hero-image')).toBeInViewport({ ratio: 0.5 }); // At least 50% visible

// Scroll and verify
await page.locator('#section-5').scrollIntoViewIfNeeded();
await expect(page.locator('#section-5')).toBeInViewport();
```

---

## State Assertions

### toBeEnabled()

Ensures the element is enabled and can be interacted with.

```typescript
// Check button is enabled
await expect(page.getByRole('button', { name: 'Submit' })).toBeEnabled();

// Verify form field is enabled after condition
await page.check('#agree-terms');
await expect(page.locator('#submit-button')).toBeEnabled();

// With timeout
await expect(page.locator('button.primary')).toBeEnabled({ timeout: 5000 });
```

#### Enabled State for Different Elements

```typescript
// Button enabled
await expect(page.getByRole('button', { name: 'Save' })).toBeEnabled();

// Input enabled
await expect(page.getByLabel('Username')).toBeEnabled();

// Select enabled
await expect(page.getByRole('combobox')).toBeEnabled();

// Link (always enabled unless disabled attribute)
await expect(page.getByRole('link', { name: 'Home' })).toBeEnabled();
```

### toBeDisabled()

Ensures the element is disabled and cannot be interacted with.

```typescript
// Check button is disabled
await expect(page.getByRole('button', { name: 'Submit' })).toBeDisabled();

// Verify button disables during submission
await page.click('#submit');
await expect(page.locator('#submit')).toBeDisabled();

// Form field disabled based on selection
await page.selectOption('#country', 'other');
await expect(page.locator('#state-field')).toBeDisabled();
```

#### Disabled Detection

```typescript
// Playwright detects disabled state from:
// - disabled attribute
// - aria-disabled="true"
// - disabled property on form elements

// All these are considered disabled:
// <button disabled>Click</button>
// <button aria-disabled="true">Click</button>
// <input disabled />
// <select disabled></select>
```

### toBeChecked()

Ensures the checkbox or radio button is checked.

```typescript
// Basic checkbox check
await expect(page.getByRole('checkbox', { name: 'Remember me' })).toBeChecked();

// Radio button checked
await expect(page.getByRole('radio', { name: 'Express shipping' })).toBeChecked();

// Check after interaction
await page.getByLabel('I agree').check();
await expect(page.getByLabel('I agree')).toBeChecked();

// With checked option (explicit)
await expect(page.locator('#newsletter')).toBeChecked({ checked: true });
await expect(page.locator('#newsletter')).toBeChecked({ checked: false }); // Same as not.toBeChecked()
```

#### Indeterminate State

```typescript
// Some checkboxes have indeterminate state (tri-state)
// toBeChecked() checks for fully checked state

// For indeterminate, use attribute check
await expect(page.locator('#select-all')).toHaveJSProperty('indeterminate', true);
```

### toBeFocused()

Ensures the element has focus.

```typescript
// Check element has focus
await expect(page.getByLabel('Search')).toBeFocused();

// Verify focus after click
await page.click('#username');
await expect(page.locator('#username')).toBeFocused();

// Tab navigation focus
await page.keyboard.press('Tab');
await expect(page.locator('#password')).toBeFocused();

// Focus after page load (autofocus)
await page.goto('/login');
await expect(page.getByLabel('Email')).toBeFocused();
```

### toBeEditable()

Ensures the element is editable (can receive text input).

```typescript
// Check input is editable
await expect(page.getByLabel('Username')).toBeEditable();

// Textarea editable
await expect(page.locator('textarea#comments')).toBeEditable();

// Content-editable div
await expect(page.locator('[contenteditable="true"]')).toBeEditable();

// After enabling a previously readonly field
await page.click('#edit-mode');
await expect(page.locator('#readonly-field')).toBeEditable();
```

#### Editable vs Enabled

```typescript
// Editable checks if element can receive text input
// Enabled checks if element can be interacted with at all

// A button can be enabled but not editable
await expect(page.locator('button')).toBeEnabled();
// await expect(page.locator('button')).toBeEditable(); // Would fail

// A readonly input is enabled but not editable
await expect(page.locator('input[readonly]')).toBeEnabled();
// await expect(page.locator('input[readonly]')).toBeEditable(); // Would fail
```

### toBeEmpty()

Ensures the element has no text content or no child elements.

```typescript
// Input is empty
await expect(page.getByLabel('Search')).toBeEmpty();

// Clear and verify
await page.locator('#field').clear();
await expect(page.locator('#field')).toBeEmpty();

// Container has no children
await expect(page.locator('.cart-items')).toBeEmpty();

// After removing all items
await page.click('#clear-all');
await expect(page.locator('.notification-list')).toBeEmpty();
```

---

## Text Assertions

### toHaveText()

Ensures the element has the exact text content. By default, matching is case-sensitive and normalizes whitespace.

```typescript
// Exact text match
await expect(page.locator('.title')).toHaveText('Welcome to Our Site');

// With regex
await expect(page.locator('.message')).toHaveText(/Hello, \w+!/);

// Case insensitive
await expect(page.locator('.status')).toHaveText('success', { ignoreCase: true });

// Multiple elements (must match all in order)
await expect(page.locator('.menu-item')).toHaveText([
  'Home',
  'Products',
  'About',
  'Contact'
]);
```

#### Text Normalization

```typescript
// By default, Playwright normalizes whitespace:
// - Trims leading/trailing whitespace
// - Collapses multiple spaces into one
// - Replaces newlines with spaces

// This matches even with extra whitespace in DOM:
// <div class="greeting">   Hello    World   </div>
await expect(page.locator('.greeting')).toHaveText('Hello World');

// Disable normalization
await expect(page.locator('.code')).toHaveText('  indented  ', {
  normalizeWhitespace: false
});
```

#### Using Regex Patterns

```typescript
// Match pattern
await expect(page.locator('.date')).toHaveText(/\d{2}\/\d{2}\/\d{4}/);

// Case insensitive regex
await expect(page.locator('.greeting')).toHaveText(/welcome/i);

// Match any of multiple patterns
await expect(page.locator('.status')).toHaveText(/success|complete|done/i);

// Complex patterns
await expect(page.locator('.email')).toHaveText(/[\w.+-]+@[\w-]+\.[\w.-]+/);
```

#### Multiple Elements

```typescript
// When locator matches multiple elements, provide array of expected texts
const items = page.locator('.list-item');
await expect(items).toHaveText(['First', 'Second', 'Third']);

// With regex for each
await expect(items).toHaveText([
  /Item \d+/,
  /Item \d+/,
  /Item \d+/
]);

// Mixed string and regex
await expect(items).toHaveText([
  'Static Text',
  /Dynamic: \w+/,
  'Another Static'
]);
```

### toContainText()

Ensures the element contains the specified substring. Less strict than `toHaveText()`.

```typescript
// Contains substring
await expect(page.locator('.description')).toContainText('important');

// With regex
await expect(page.locator('.log')).toContainText(/error|warning/i);

// Verify partial message
await expect(page.locator('.notification')).toContainText('successfully saved');

// Multiple elements (each must contain text)
await expect(page.locator('.card')).toContainText(['price', 'price', 'price']);
```

#### toHaveText vs toContainText

```typescript
// toHaveText - checks entire text content
// DOM: <div>Hello World</div>
await expect(page.locator('div')).toHaveText('Hello World'); // Pass
await expect(page.locator('div')).toHaveText('Hello'); // Fail

// toContainText - checks for substring
await expect(page.locator('div')).toContainText('Hello World'); // Pass
await expect(page.locator('div')).toContainText('Hello'); // Pass
await expect(page.locator('div')).toContainText('World'); // Pass
```

#### Whitespace and Case Options

```typescript
// Case insensitive
await expect(page.locator('.message')).toContainText('SUCCESS', {
  ignoreCase: true
});

// Preserve whitespace
await expect(page.locator('.code-block')).toContainText('  if  ', {
  normalizeWhitespace: false
});

// Use inner text vs text content
await expect(page.locator('.formatted')).toContainText('visible only', {
  useInnerText: true // Respects CSS (hidden elements excluded)
});
```

---

## Value Assertions

### toHaveValue()

Ensures an input element has the specified value. Works with `<input>`, `<textarea>`, and `<select>` elements.

```typescript
// Input value
await expect(page.getByLabel('Username')).toHaveValue('john_doe');

// Textarea value
await expect(page.locator('textarea#bio')).toHaveValue('My bio text...');

// Select value
await expect(page.locator('select#country')).toHaveValue('us');

// After filling
await page.fill('#email', 'test@example.com');
await expect(page.locator('#email')).toHaveValue('test@example.com');
```

#### Value with Regex

```typescript
// Match pattern
await expect(page.locator('#phone')).toHaveValue(/\d{3}-\d{3}-\d{4}/);

// Partial match with regex
await expect(page.locator('#email')).toHaveValue(/@example\.com$/);

// UUID format
await expect(page.locator('#session-id')).toHaveValue(
  /^[0-9a-f]{8}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{12}$/i
);
```

### toHaveValues()

Ensures a `<select multiple>` element has the specified selected values.

```typescript
// Multiple selected values
await expect(page.locator('select#colors')).toHaveValues(['red', 'blue', 'green']);

// After selecting multiple options
await page.selectOption('#skills', ['javascript', 'typescript', 'python']);
await expect(page.locator('#skills')).toHaveValues(['javascript', 'typescript', 'python']);

// Order matters
await expect(page.locator('#categories')).toHaveValues(['cat1', 'cat2']); // Specific order

// With regex patterns
await expect(page.locator('#options')).toHaveValues([/opt-\d+/, /opt-\d+/]);
```

---

## Attribute Assertions

### toHaveAttribute()

Ensures the element has the specified attribute with optional value check.

```typescript
// Check attribute exists (any value)
await expect(page.locator('img')).toHaveAttribute('src');

// Check attribute with exact value
await expect(page.locator('a.home')).toHaveAttribute('href', '/');

// Check attribute with regex
await expect(page.locator('img.product')).toHaveAttribute('src', /\/images\/products\//);

// Data attributes
await expect(page.locator('.item')).toHaveAttribute('data-id', '123');
await expect(page.locator('.user')).toHaveAttribute('data-role', 'admin');
```

#### Common Attribute Checks

```typescript
// Links
await expect(page.locator('a.external')).toHaveAttribute('target', '_blank');
await expect(page.locator('a.download')).toHaveAttribute('download');

// Forms
await expect(page.locator('input')).toHaveAttribute('type', 'email');
await expect(page.locator('input')).toHaveAttribute('required');
await expect(page.locator('input')).toHaveAttribute('placeholder', 'Enter email');
await expect(page.locator('input')).toHaveAttribute('maxlength', '100');

// Images
await expect(page.locator('img')).toHaveAttribute('alt', 'Product image');
await expect(page.locator('img')).toHaveAttribute('loading', 'lazy');

// ARIA attributes
await expect(page.locator('.dialog')).toHaveAttribute('role', 'dialog');
await expect(page.locator('.dialog')).toHaveAttribute('aria-modal', 'true');
await expect(page.locator('.alert')).toHaveAttribute('aria-live', 'polite');
```

#### Attribute Value Patterns

```typescript
// URL patterns
await expect(page.locator('a')).toHaveAttribute('href', /^https:\/\//);

// ID patterns
await expect(page.locator('.item')).toHaveAttribute('data-id', /^\d+$/);

// Boolean-like attributes (present = true)
await expect(page.locator('input')).toHaveAttribute('disabled', '');
await expect(page.locator('input')).toHaveAttribute('readonly', '');
```

### toHaveClass()

Ensures the element has the specified CSS class(es).

```typescript
// Single class
await expect(page.locator('button')).toHaveClass('btn-primary');

// With regex (partial match)
await expect(page.locator('button')).toHaveClass(/btn-/);

// Multiple classes (all must be present)
await expect(page.locator('.card')).toHaveClass('card card-elevated shadow');

// Dynamic class verification
await page.click('#toggle-active');
await expect(page.locator('.item')).toHaveClass(/active/);
```

#### Class Matching Behavior

```typescript
// Element: <div class="btn btn-primary btn-lg"></div>

// Exact match (all classes in that order, space-separated)
await expect(page.locator('div')).toHaveClass('btn btn-primary btn-lg'); // Pass

// Partial match with regex
await expect(page.locator('div')).toHaveClass(/btn-primary/); // Pass
await expect(page.locator('div')).toHaveClass(/btn/); // Pass

// This would fail (missing classes):
// await expect(page.locator('div')).toHaveClass('btn'); // Fail - exact match

// Multiple elements with different classes
const buttons = page.locator('button');
await expect(buttons).toHaveClass(['btn-submit', 'btn-cancel', 'btn-reset']);
```

### toHaveId()

Ensures the element has the specified ID.

```typescript
// Check exact ID
await expect(page.locator('.main-header')).toHaveId('header');

// Verify dynamic IDs
await expect(page.locator('.modal')).toHaveId(/modal-\d+/);

// After render
await expect(page.locator('[data-testid="submit"]')).toHaveId('submit-button');
```

### toHaveCSS()

Ensures the element has the specified CSS property value.

```typescript
// Check color (must use computed format)
await expect(page.locator('.error')).toHaveCSS('color', 'rgb(255, 0, 0)');

// Check background
await expect(page.locator('.success')).toHaveCSS('background-color', 'rgb(0, 255, 0)');

// Check display
await expect(page.locator('.hidden')).toHaveCSS('display', 'none');
await expect(page.locator('.visible')).toHaveCSS('display', 'block');

// Check visibility
await expect(page.locator('.invisible')).toHaveCSS('visibility', 'hidden');

// Check dimensions
await expect(page.locator('.box')).toHaveCSS('width', '100px');
await expect(page.locator('.box')).toHaveCSS('height', '50px');
```

#### CSS Computed Values

```typescript
// CSS values are computed values (not what's in stylesheet)

// Colors are always rgb() or rgba()
await expect(page.locator('a')).toHaveCSS('color', 'rgb(0, 0, 255)'); // Not "blue"

// Shorthand properties expand
await expect(page.locator('.box')).toHaveCSS('margin-top', '10px'); // Not "margin"
await expect(page.locator('.box')).toHaveCSS('border-top-width', '1px');

// Font properties
await expect(page.locator('p')).toHaveCSS('font-size', '16px');
await expect(page.locator('h1')).toHaveCSS('font-weight', '700');
await expect(page.locator('p')).toHaveCSS('font-family', '"Helvetica Neue", sans-serif');

// Flexbox
await expect(page.locator('.container')).toHaveCSS('display', 'flex');
await expect(page.locator('.container')).toHaveCSS('justify-content', 'center');
await expect(page.locator('.container')).toHaveCSS('align-items', 'center');

// Transforms (computed matrix form)
await expect(page.locator('.rotated')).toHaveCSS('transform', 'matrix(0, 1, -1, 0, 0, 0)');
```

### toHaveJSProperty()

Ensures the element has the specified JavaScript property value.

```typescript
// Check JS property (not HTML attribute)
await expect(page.locator('input[type="checkbox"]')).toHaveJSProperty('checked', true);

// Value property
await expect(page.locator('input')).toHaveJSProperty('value', 'test');

// Custom properties set via JavaScript
await expect(page.locator('#component')).toHaveJSProperty('isInitialized', true);

// HTMLInputElement properties
await expect(page.locator('input')).toHaveJSProperty('selectionStart', 0);
await expect(page.locator('input')).toHaveJSProperty('validity', expect.objectContaining({
  valid: true
}));
```

### toHaveRole()

Ensures the element has the specified ARIA role.

```typescript
// Check role
await expect(page.locator('.dialog')).toHaveRole('dialog');
await expect(page.locator('.sidebar')).toHaveRole('navigation');
await expect(page.locator('.alert')).toHaveRole('alert');

// Implicit roles
await expect(page.locator('button')).toHaveRole('button');
await expect(page.locator('a[href]')).toHaveRole('link');
await expect(page.locator('input[type="text"]')).toHaveRole('textbox');
```

### toHaveAccessibleName()

Ensures the element has the specified accessible name.

```typescript
// Check accessible name
await expect(page.locator('button')).toHaveAccessibleName('Submit Form');
await expect(page.locator('img')).toHaveAccessibleName('Company Logo');

// With regex
await expect(page.locator('button')).toHaveAccessibleName(/submit/i);

// Accessible name from label
await expect(page.locator('input#email')).toHaveAccessibleName('Email Address');
```

### toHaveAccessibleDescription()

Ensures the element has the specified accessible description.

```typescript
// Check accessible description (from aria-describedby)
await expect(page.locator('#password')).toHaveAccessibleDescription(
  'Password must be at least 8 characters'
);

// With regex
await expect(page.locator('button')).toHaveAccessibleDescription(/confirm/i);
```

---

## Count Assertions

### toHaveCount()

Ensures the locator matches the specified number of elements.

```typescript
// Exact count
await expect(page.locator('.list-item')).toHaveCount(5);

// After adding items
await page.click('#add-item');
await page.click('#add-item');
await expect(page.locator('.item')).toHaveCount(2);

// After removing
await page.click('.delete-all');
await expect(page.locator('.item')).toHaveCount(0);

// Role-based counting
await expect(page.getByRole('listitem')).toHaveCount(10);
await expect(page.getByRole('row')).toHaveCount(25);
```

#### Count with Different Locators

```typescript
// Count table rows
await expect(page.locator('table tbody tr')).toHaveCount(20);

// Count cards
await expect(page.locator('.card')).toHaveCount(6);

// Count visible items only
const visibleItems = page.locator('.item:visible');
await expect(visibleItems).toHaveCount(5);

// Count filtered results
await page.fill('#search', 'test');
await expect(page.locator('.search-result')).toHaveCount(3);

// After pagination
await page.click('#next-page');
await expect(page.locator('.item')).toHaveCount(10);
```

---

## Page Assertions

### toHaveURL()

Ensures the page has the specified URL.

```typescript
// Exact URL match
await expect(page).toHaveURL('https://example.com/dashboard');

// With regex
await expect(page).toHaveURL(/\/dashboard$/);
await expect(page).toHaveURL(/example\.com/);

// After navigation
await page.click('a[href="/profile"]');
await expect(page).toHaveURL('/profile');

// Query parameters
await expect(page).toHaveURL(/\?tab=settings/);
await expect(page).toHaveURL('https://example.com/search?q=test');
```

#### URL Pattern Matching

```typescript
// Match path
await expect(page).toHaveURL(/\/users\/\d+/); // /users/123

// Match any query string
await expect(page).toHaveURL(/\/search\?/);

// Match specific domain
await expect(page).toHaveURL(/^https:\/\/api\.example\.com/);

// Match hash/fragment
await expect(page).toHaveURL(/#section-\d+/);

// Complex patterns
await expect(page).toHaveURL(/\/products\/[\w-]+\/reviews$/);
```

#### URL Object Matching

```typescript
// Using URL object properties
await expect(page).toHaveURL(url => {
  const parsed = new URL(url);
  return parsed.pathname === '/dashboard' && parsed.searchParams.has('tab');
});
```

### toHaveTitle()

Ensures the page has the specified title.

```typescript
// Exact title match
await expect(page).toHaveTitle('Dashboard - My App');

// With regex
await expect(page).toHaveTitle(/Dashboard/);
await expect(page).toHaveTitle(/My App$/);

// After navigation
await page.goto('/about');
await expect(page).toHaveTitle('About Us - My App');

// Dynamic title
await expect(page).toHaveTitle(/Welcome, \w+/);
```

---

## Screenshot Assertions

### toHaveScreenshot()

Performs visual comparison against a reference screenshot.

```typescript
// Element screenshot
await expect(page.locator('.chart')).toHaveScreenshot('chart.png');

// Full page screenshot
await expect(page).toHaveScreenshot('homepage.png');

// Auto-generated name (based on test name)
await expect(page).toHaveScreenshot();
await expect(page.locator('.header')).toHaveScreenshot();
```

#### Screenshot Options

```typescript
// Maximum different pixels allowed
await expect(page.locator('.component')).toHaveScreenshot('component.png', {
  maxDiffPixels: 100,
});

// Maximum different pixel ratio
await expect(page.locator('.component')).toHaveScreenshot('component.png', {
  maxDiffPixelRatio: 0.1, // 10% of pixels can differ
});

// Threshold for pixel color difference (0-1)
await expect(page.locator('.component')).toHaveScreenshot('component.png', {
  threshold: 0.2, // 20% color difference tolerance
});

// Disable animations
await expect(page).toHaveScreenshot('static.png', {
  animations: 'disabled',
});

// Mask dynamic elements
await expect(page).toHaveScreenshot('page.png', {
  mask: [page.locator('.timestamp'), page.locator('.random-ad')],
});
```

#### Advanced Screenshot Options

```typescript
// Full page screenshot
await expect(page).toHaveScreenshot('full.png', {
  fullPage: true,
});

// Specific clip area
await expect(page).toHaveScreenshot('header.png', {
  clip: { x: 0, y: 0, width: 1200, height: 100 },
});

// Scale
await expect(page).toHaveScreenshot('scaled.png', {
  scale: 'css', // or 'device'
});

// Timeout for screenshot stabilization
await expect(page).toHaveScreenshot('stable.png', {
  timeout: 10000,
});

// Omit background (transparent)
await expect(page.locator('.logo')).toHaveScreenshot('logo.png', {
  omitBackground: true,
});
```

#### Updating Screenshots

```bash
# Update all screenshots
npx playwright test --update-snapshots

# Update specific test file
npx playwright test tests/visual.spec.ts --update-snapshots
```

---

## Negation

### Using .not

All assertions can be negated using `.not` to assert the opposite condition.

```typescript
// Visibility
await expect(page.locator('.loading')).not.toBeVisible();
await expect(page.locator('.content')).not.toBeHidden();

// State
await expect(page.locator('button')).not.toBeDisabled();
await expect(page.locator('input')).not.toBeChecked();
await expect(page.locator('.field')).not.toBeEmpty();

// Text
await expect(page.locator('.message')).not.toHaveText('Error');
await expect(page.locator('.status')).not.toContainText('failed');

// Value
await expect(page.locator('input')).not.toHaveValue('');

// Attribute
await expect(page.locator('button')).not.toHaveAttribute('disabled');
await expect(page.locator('.item')).not.toHaveClass(/hidden/);

// Count
await expect(page.locator('.error')).not.toHaveCount(0); // At least one error

// Page
await expect(page).not.toHaveURL('/login');
await expect(page).not.toHaveTitle('Error');
```

#### Negation Best Practices

```typescript
// Prefer positive assertions when possible
// BAD: Double negative (confusing)
await expect(page.locator('.item')).not.not.toBeVisible();

// GOOD: Use specific assertions
await expect(page.locator('.loading')).toBeHidden(); // Instead of not.toBeVisible()
await expect(page.locator('input')).toBeEmpty(); // Instead of not.toHaveValue(/.+/)

// Negation is useful for absence checks
await expect(page.locator('.error-message')).not.toBeVisible();
await expect(page).not.toHaveURL(/error/);
```

---

## Soft Assertions

### expect.soft()

Soft assertions do not stop test execution when they fail. The test continues, and all failures are reported at the end.

```typescript
import { test, expect } from '@playwright/test';

test('multiple checks', async ({ page }) => {
  await page.goto('/dashboard');

  // All assertions run even if some fail
  await expect.soft(page.locator('.welcome')).toHaveText('Welcome');
  await expect.soft(page.locator('.user-name')).toContainText('John');
  await expect.soft(page.locator('.notification-count')).toHaveText('5');
  await expect.soft(page.locator('.status')).toHaveText('Active');

  // Test continues...
  await page.click('#logout');
  await expect.soft(page).toHaveURL('/login');
});
```

### Checking Soft Assertion Failures

```typescript
test('verify form validation', async ({ page }) => {
  await page.goto('/registration');
  await page.click('#submit');

  // Check multiple validation messages
  await expect.soft(page.locator('#name-error')).toBeVisible();
  await expect.soft(page.locator('#email-error')).toBeVisible();
  await expect.soft(page.locator('#password-error')).toBeVisible();

  // Check if any soft assertions failed
  expect(test.info().errors).toHaveLength(0);
});
```

### Soft Assertions Use Cases

```typescript
// Validation testing
test('form validation shows all errors', async ({ page }) => {
  await page.goto('/form');
  await page.click('#submit');

  await expect.soft(page.locator('.error').nth(0)).toHaveText('Name is required');
  await expect.soft(page.locator('.error').nth(1)).toHaveText('Email is required');
  await expect.soft(page.locator('.error').nth(2)).toHaveText('Password is required');
  await expect.soft(page.locator('.error').nth(3)).toHaveText('Phone is required');
});

// Dashboard verification
test('dashboard displays all widgets', async ({ page }) => {
  await page.goto('/dashboard');

  await expect.soft(page.locator('.widget-sales')).toBeVisible();
  await expect.soft(page.locator('.widget-users')).toBeVisible();
  await expect.soft(page.locator('.widget-revenue')).toBeVisible();
  await expect.soft(page.locator('.widget-orders')).toBeVisible();

  // Get comprehensive failure report
});

// Content verification
test('article has all sections', async ({ page }) => {
  await page.goto('/article/123');

  await expect.soft(page.locator('h1')).toHaveText('Article Title');
  await expect.soft(page.locator('.author')).toContainText('By');
  await expect.soft(page.locator('.date')).toContainText('2024');
  await expect.soft(page.locator('.content')).not.toBeEmpty();
  await expect.soft(page.locator('.tags')).toBeVisible();
});
```

---

## Polling Assertions

### expect.poll()

Creates a polling assertion that repeatedly calls the provided function until it returns the expected value or times out.

```typescript
// Poll API status
await expect.poll(async () => {
  const response = await page.request.get('/api/status');
  return response.json();
}).toEqual({ status: 'ready' });

// Poll element count
await expect.poll(async () => {
  return await page.locator('.item').count();
}).toBe(10);

// Poll text content
await expect.poll(async () => {
  return await page.locator('.status').textContent();
}).toBe('Complete');
```

### Polling Options

```typescript
// Custom intervals (time between checks)
await expect.poll(async () => {
  return await page.locator('.progress').getAttribute('value');
}, {
  intervals: [100, 200, 500, 1000], // Increasing intervals
}).toBe('100');

// Custom timeout
await expect.poll(async () => {
  const response = await page.request.get('/api/job/123');
  const data = await response.json();
  return data.status;
}, {
  timeout: 60000, // 60 seconds
}).toBe('completed');

// Custom message
await expect.poll(async () => {
  return await page.locator('.count').textContent();
}, {
  message: 'Waiting for count to reach 100',
  timeout: 30000,
}).toBe('100');
```

### Polling Use Cases

```typescript
// Wait for async operation
await expect.poll(async () => {
  const response = await page.request.get('/api/export/status');
  return (await response.json()).progress;
}).toBe(100);

// Wait for file processing
await expect.poll(async () => {
  const rows = await page.locator('table tbody tr').count();
  return rows;
}, {
  intervals: [500, 1000, 2000],
  timeout: 30000,
}).toBeGreaterThan(0);

// Wait for WebSocket update
await expect.poll(async () => {
  return await page.locator('.live-price').textContent();
}, {
  intervals: [100],
}).not.toBe('Loading...');

// Complex condition
await expect.poll(async () => {
  const items = await page.locator('.item').all();
  return items.length > 0 && await items[0].isVisible();
}).toBe(true);
```

---

## Custom Timeout

### Per-Assertion Timeout

Override the default timeout for individual assertions.

```typescript
// Longer timeout for slow operations
await expect(page.locator('.heavy-content')).toBeVisible({ timeout: 30000 });

// Longer timeout for navigation
await expect(page).toHaveURL('/slow-page', { timeout: 60000 });

// Shorter timeout for quick checks
await expect(page.locator('.instant-update')).toHaveText('Done', { timeout: 1000 });
```

### Global Timeout Configuration

```typescript
// playwright.config.ts
import { defineConfig } from '@playwright/test';

export default defineConfig({
  expect: {
    timeout: 10000, // 10 seconds for all assertions
  },
});
```

### Project-Level Timeout

```typescript
// playwright.config.ts
import { defineConfig } from '@playwright/test';

export default defineConfig({
  projects: [
    {
      name: 'fast-tests',
      expect: { timeout: 5000 },
    },
    {
      name: 'slow-tests',
      expect: { timeout: 30000 },
    },
  ],
});
```

### Timeout in Test Setup

```typescript
// Set timeout for specific test file
test.describe('slow page tests', () => {
  test.use({
    expect: { timeout: 20000 },
  });

  test('loads dashboard', async ({ page }) => {
    await expect(page.locator('.dashboard')).toBeVisible();
  });
});
```

---

## toPass for Retry

### Using toPass()

The `toPass()` assertion retries a block of code until it passes or times out. Useful for complex assertions that cannot be expressed with a single expect.

```typescript
// Retry until complex condition passes
await expect(async () => {
  const response = await page.request.get('/api/data');
  expect(response.status()).toBe(200);

  const data = await response.json();
  expect(data.items).toHaveLength(10);
  expect(data.status).toBe('complete');
}).toPass();

// With options
await expect(async () => {
  await page.click('#refresh');
  await expect(page.locator('.count')).toHaveText('100');
}).toPass({
  timeout: 30000,
  intervals: [1000, 2000, 5000],
});
```

### toPass Use Cases

```typescript
// Retry flaky operation
await expect(async () => {
  await page.click('#unreliable-button');
  await expect(page.locator('.success-message')).toBeVisible();
}).toPass({ timeout: 10000 });

// Wait for multiple conditions
await expect(async () => {
  const items = await page.locator('.item').all();
  expect(items.length).toBeGreaterThan(5);

  for (const item of items) {
    await expect(item).toBeVisible();
    await expect(item.locator('.price')).not.toBeEmpty();
  }
}).toPass();

// API retry with validation
await expect(async () => {
  const response = await page.request.post('/api/process', {
    data: { id: 123 }
  });

  expect(response.ok()).toBe(true);

  const result = await response.json();
  expect(result.processed).toBe(true);
  expect(result.errors).toHaveLength(0);
}).toPass({ timeout: 60000 });

// Complex DOM state
await expect(async () => {
  const modal = page.locator('.modal');
  await expect(modal).toBeVisible();
  await expect(modal.locator('.title')).toHaveText('Confirmation');
  await expect(modal.locator('.btn-confirm')).toBeEnabled();
}).toPass();
```

### toPass Options

```typescript
await expect(async () => {
  // Assertions here
}).toPass({
  // Maximum time to retry
  timeout: 30000,

  // Intervals between retries
  intervals: [100, 250, 500, 1000],

  // Custom message on failure
  message: 'Failed to complete the operation',
});
```

---

## Generic Matchers

### Non-Retrying Matchers

These matchers check immediately without retrying. Use them with already-fetched values.

```typescript
// toBe - strict equality (===)
const count = await page.locator('.item').count();
expect(count).toBe(5);

const title = await page.title();
expect(title).toBe('Dashboard');

// toEqual - deep equality
const data = await page.evaluate(() => window.appState);
expect(data).toEqual({ user: 'john', role: 'admin' });

// toContain - array/string contains
const classes = await page.locator('.btn').getAttribute('class');
expect(classes).toContain('btn-primary');

const items = await page.locator('.item').allTextContents();
expect(items).toContain('Special Item');

// toMatch - string regex match
const text = await page.locator('.message').textContent();
expect(text).toMatch(/Hello, \w+!/);
```

### Numeric Matchers

```typescript
const count = await page.locator('.item').count();

// Greater than
expect(count).toBeGreaterThan(0);
expect(count).toBeGreaterThanOrEqual(5);

// Less than
expect(count).toBeLessThan(100);
expect(count).toBeLessThanOrEqual(50);

// Close to (for floating point)
const progress = parseFloat(await page.locator('.progress').getAttribute('value'));
expect(progress).toBeCloseTo(0.75, 2); // 2 decimal places
```

### Truthiness Matchers

```typescript
const element = await page.locator('.optional').count();
expect(element > 0).toBeTruthy();

const disabled = await page.locator('button').isDisabled();
expect(disabled).toBeFalsy();

const value = await page.locator('input').inputValue();
expect(value).toBeDefined();
expect(value).not.toBeNull();
expect(value).not.toBeUndefined();
```

### Object Matchers

```typescript
// toMatchObject - partial object match
const response = await page.request.get('/api/user');
const user = await response.json();

expect(user).toMatchObject({
  name: 'John',
  role: 'admin',
});

// toHaveProperty - check property existence and value
expect(user).toHaveProperty('email');
expect(user).toHaveProperty('profile.avatar');
expect(user).toHaveProperty('settings.theme', 'dark');

// objectContaining - in expect calls
expect(user).toEqual(expect.objectContaining({
  id: expect.any(Number),
  email: expect.stringMatching(/@example\.com$/),
}));
```

### Array Matchers

```typescript
const items = await page.locator('.item').allTextContents();

// toHaveLength
expect(items).toHaveLength(5);

// toContain
expect(items).toContain('Special');

// arrayContaining
expect(items).toEqual(expect.arrayContaining(['Item 1', 'Item 2']));

// Each element
items.forEach(item => {
  expect(item).toMatch(/Item \d+/);
});
```

### String Matchers

```typescript
const text = await page.locator('.content').textContent();

// Exact match
expect(text).toBe('Hello World');

// Contains
expect(text).toContain('World');

// Regex
expect(text).toMatch(/^Hello/);

// stringContaining
expect(text).toEqual(expect.stringContaining('llo Wor'));

// stringMatching
expect(text).toEqual(expect.stringMatching(/hello/i));
```

### Type Matchers

```typescript
const data = await page.evaluate(() => window.appData);

// Type checking
expect(typeof data.id).toBe('number');
expect(Array.isArray(data.items)).toBe(true);

// Using expect helpers
expect(data.id).toEqual(expect.any(Number));
expect(data.name).toEqual(expect.any(String));
expect(data.items).toEqual(expect.any(Array));
```

---

## Assertion Options

### Common Options

Most auto-retrying assertions accept these options:

```typescript
interface AssertionOptions {
  timeout?: number;      // Max time to wait (ms)
  message?: string;      // Custom error message
}
```

```typescript
// Timeout
await expect(page.locator('.slow')).toBeVisible({ timeout: 15000 });

// Custom message
await expect(page.locator('.status')).toHaveText('Active', {
  message: 'Expected user status to be Active after login',
});

// Both
await expect(page.locator('.result')).toContainText('Success', {
  timeout: 10000,
  message: 'Operation should complete successfully',
});
```

### Text-Specific Options

```typescript
interface TextOptions {
  ignoreCase?: boolean;        // Case-insensitive matching
  normalizeWhitespace?: boolean; // Normalize whitespace (default: true)
  useInnerText?: boolean;      // Use innerText instead of textContent
  timeout?: number;
}

// Case insensitive
await expect(page.locator('.message')).toHaveText('SUCCESS', {
  ignoreCase: true
});

// Preserve whitespace
await expect(page.locator('.code')).toHaveText('  code  ', {
  normalizeWhitespace: false
});

// Use inner text (respects CSS visibility)
await expect(page.locator('.formatted')).toContainText('Visible Text', {
  useInnerText: true
});
```

### Screenshot Options

```typescript
interface ScreenshotOptions {
  animations?: 'disabled' | 'allow';
  caret?: 'hide' | 'initial';
  clip?: { x: number; y: number; width: number; height: number };
  fullPage?: boolean;
  mask?: Locator[];
  maskColor?: string;
  maxDiffPixels?: number;
  maxDiffPixelRatio?: number;
  omitBackground?: boolean;
  scale?: 'css' | 'device';
  stylePath?: string | string[];
  threshold?: number;
  timeout?: number;
}

await expect(page).toHaveScreenshot('page.png', {
  animations: 'disabled',
  mask: [page.locator('.dynamic-content')],
  maxDiffPixels: 50,
  fullPage: true,
});
```

### Polling Options

```typescript
interface PollOptions {
  intervals?: number[];  // Time between polls
  timeout?: number;      // Max total time
  message?: string;      // Custom error message
}

await expect.poll(async () => {
  return await page.locator('.count').textContent();
}, {
  intervals: [100, 200, 500, 1000],
  timeout: 30000,
  message: 'Waiting for count to update',
}).toBe('100');
```

### toPass Options

```typescript
interface ToPassOptions {
  intervals?: number[];  // Time between retries
  timeout?: number;      // Max total time
  message?: string;      // Custom error message
}

await expect(async () => {
  // Assertions
}).toPass({
  intervals: [1000, 2000, 5000],
  timeout: 60000,
  message: 'Complex operation should succeed',
});
```

---

## Best Practices

### 1. Use Auto-Retrying Assertions

```typescript
// BAD: Manual wait then immediate check
await page.waitForSelector('.element');
const isVisible = await page.locator('.element').isVisible();
expect(isVisible).toBe(true);

// GOOD: Auto-retrying assertion
await expect(page.locator('.element')).toBeVisible();
```

### 2. Avoid Explicit Waits

```typescript
// BAD: Arbitrary sleep
await page.waitForTimeout(2000);
await expect(page.locator('.status')).toHaveText('Done');

// GOOD: Let assertion handle timing
await expect(page.locator('.status')).toHaveText('Done');
```

### 3. Use Appropriate Assertion Methods

```typescript
// BAD: Checking visibility by getting attribute
const display = await page.locator('.modal').evaluate(
  el => getComputedStyle(el).display
);
expect(display).not.toBe('none');

// GOOD: Use built-in visibility assertion
await expect(page.locator('.modal')).toBeVisible();
```

### 4. Prefer Specific Assertions

```typescript
// BAD: Generic assertion
const text = await page.locator('.title').textContent();
expect(text).toBe('Welcome');

// GOOD: Specific assertion (auto-retries)
await expect(page.locator('.title')).toHaveText('Welcome');
```

### 5. Use Soft Assertions for Multiple Checks

```typescript
// BAD: Stops at first failure
await expect(page.locator('.field1')).toBeVisible();
await expect(page.locator('.field2')).toBeVisible();
await expect(page.locator('.field3')).toBeVisible();

// GOOD: Reports all failures
await expect.soft(page.locator('.field1')).toBeVisible();
await expect.soft(page.locator('.field2')).toBeVisible();
await expect.soft(page.locator('.field3')).toBeVisible();
```

### 6. Set Appropriate Timeouts

```typescript
// BAD: Using default timeout for known slow operations
await expect(page.locator('.slow-report')).toBeVisible();

// GOOD: Explicit timeout for slow operations
await expect(page.locator('.slow-report')).toBeVisible({ timeout: 30000 });

// Also configure globally for consistency
// playwright.config.ts
export default defineConfig({
  expect: { timeout: 10000 },
});
```

### 7. Use Descriptive Custom Messages

```typescript
// BAD: No context on failure
await expect(page.locator('.status')).toHaveText('Active');

// GOOD: Clear failure message
await expect(page.locator('.status')).toHaveText('Active', {
  message: 'User status should be Active after successful login',
});
```

### 8. Combine Assertions Logically

```typescript
// BAD: Testing implementation details
await expect(page.locator('.modal')).toHaveCSS('display', 'block');
await expect(page.locator('.modal')).toHaveCSS('opacity', '1');

// GOOD: Test behavior, not implementation
await expect(page.locator('.modal')).toBeVisible();
```

### 9. Use toPass for Complex Scenarios

```typescript
// BAD: Multiple separate assertions with timing issues
await page.click('#refresh');
await expect(page.locator('.items')).toHaveCount(10);
await expect(page.locator('.status')).toHaveText('Loaded');

// GOOD: Grouped assertions that must all pass together
await expect(async () => {
  await page.click('#refresh');
  await expect(page.locator('.items')).toHaveCount(10);
  await expect(page.locator('.status')).toHaveText('Loaded');
}).toPass();
```

### 10. Test User-Visible Behavior

```typescript
// BAD: Testing internal state
const state = await page.evaluate(() => window.appState.isLoggedIn);
expect(state).toBe(true);

// GOOD: Test visible outcome
await expect(page.locator('.welcome-message')).toBeVisible();
await expect(page.locator('.user-menu')).toBeVisible();
```

### 11. Use Appropriate Locator Strategies

```typescript
// BAD: Fragile selectors in assertions
await expect(page.locator('div.container > div:nth-child(2) > span')).toHaveText('Hello');

// GOOD: Semantic locators
await expect(page.getByRole('heading', { name: 'Hello' })).toBeVisible();
await expect(page.getByText('Hello')).toBeVisible();
await expect(page.getByTestId('greeting')).toHaveText('Hello');
```

### 12. Handle Dynamic Content

```typescript
// Mask dynamic content in screenshots
await expect(page).toHaveScreenshot('dashboard.png', {
  mask: [
    page.locator('.timestamp'),
    page.locator('.random-id'),
    page.locator('.ad-banner'),
  ],
});

// Use regex for dynamic text
await expect(page.locator('.order-id')).toHaveText(/Order #\d+/);

// Use polling for API-dependent content
await expect.poll(async () => {
  return await page.locator('.live-count').textContent();
}).toMatch(/\d+/);
```

### 13. Organize Assertions in Tests

```typescript
test('user registration flow', async ({ page }) => {
  // Arrange
  await page.goto('/register');

  // Act
  await page.fill('#email', 'test@example.com');
  await page.fill('#password', 'SecurePass123');
  await page.click('#submit');

  // Assert - grouped logically
  await expect(page).toHaveURL('/welcome');
  await expect(page.locator('.success-message')).toBeVisible();
  await expect(page.locator('.user-email')).toHaveText('test@example.com');
});
```

### 14. Avoid Negation When Positive Alternative Exists

```typescript
// Okay but less clear
await expect(page.locator('.error')).not.toBeVisible();

// Better - more explicit
await expect(page.locator('.error')).toBeHidden();

// Okay but less clear
await expect(page.locator('input')).not.toHaveValue('');

// Better - test for expected state
await expect(page.locator('input')).toHaveValue('expected value');
```

### 15. Use Assertions in Page Objects

```typescript
// page-objects/login-page.ts
class LoginPage {
  constructor(private page: Page) {}

  async expectToBeOnLoginPage() {
    await expect(this.page).toHaveURL('/login');
    await expect(this.page.locator('h1')).toHaveText('Sign In');
  }

  async expectLoginError(message: string) {
    await expect(this.page.locator('.error-alert')).toBeVisible();
    await expect(this.page.locator('.error-alert')).toContainText(message);
  }

  async expectLoginSuccess() {
    await expect(this.page).toHaveURL('/dashboard');
    await expect(this.page.locator('.welcome')).toBeVisible();
  }
}

// In test
test('login with invalid credentials', async ({ page }) => {
  const loginPage = new LoginPage(page);
  await page.goto('/login');

  await page.fill('#email', 'wrong@example.com');
  await page.fill('#password', 'wrongpass');
  await page.click('#submit');

  await loginPage.expectLoginError('Invalid credentials');
});
```

---

## Summary

Playwright's assertion system is designed to make tests reliable and maintainable:

1. **Auto-retrying assertions** eliminate flakiness by waiting for conditions
2. **Specific matchers** for visibility, state, text, values, attributes, and more
3. **Soft assertions** allow comprehensive validation without stopping on first failure
4. **Polling** enables custom async condition checking
5. **toPass()** groups complex assertions that must all succeed
6. **Flexible options** for timeouts, messages, and comparison parameters

Key principles:
- Use auto-retrying assertions with locators
- Avoid arbitrary waits
- Test user-visible behavior
- Set appropriate timeouts
- Use descriptive error messages
- Group related assertions logically
