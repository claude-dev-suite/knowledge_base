# Playwright Locators

Locators are the primary way to find elements on a page in Playwright. They represent a way to find element(s) on the page at any moment and are used to perform actions on elements such as click, fill, etc.

## Table of Contents

1. [Role-based Locators](#role-based-locators)
2. [Text Locators](#text-locators)
3. [Label and Placeholder Locators](#label-and-placeholder-locators)
4. [Test ID Locators](#test-id-locators)
5. [CSS Selectors](#css-selectors)
6. [XPath Selectors](#xpath-selectors)
7. [Locator Chaining](#locator-chaining)
8. [Filtering Locators](#filtering-locators)
9. [Nth Locators](#nth-locators)
10. [Frame Locators](#frame-locators)
11. [Locator Actions](#locator-actions)
12. [Locator State](#locator-state)
13. [Getting Values](#getting-values)
14. [Strict Mode and Multiple Matches](#strict-mode-and-multiple-matches)
15. [Locator Assertions](#locator-assertions)
16. [Best Practices](#best-practices)
17. [Debugging Locators](#debugging-locators)

---

## Role-based Locators

Role-based locators are the recommended way to locate elements. They use ARIA roles and accessible names to find elements, making tests more resilient and accessible.

### Basic Role Locators

```typescript
// Button
page.getByRole('button', { name: 'Submit' });
page.getByRole('button', { name: /submit/i }); // regex for case-insensitive

// Link
page.getByRole('link', { name: 'Sign up' });

// Heading
page.getByRole('heading', { name: 'Welcome' });
page.getByRole('heading', { level: 1 }); // h1 specifically
page.getByRole('heading', { name: 'Welcome', level: 2 }); // h2 with specific name

// Textbox (input[type=text], textarea)
page.getByRole('textbox', { name: 'Email' });

// Checkbox
page.getByRole('checkbox', { name: 'Accept terms' });
page.getByRole('checkbox', { name: 'Accept terms', checked: true });

// Radio
page.getByRole('radio', { name: 'Option A' });
page.getByRole('radio', { name: 'Option A', checked: true });

// Combobox (select, autocomplete)
page.getByRole('combobox', { name: 'Country' });

// Listbox
page.getByRole('listbox', { name: 'Options' });

// Option (within listbox or combobox)
page.getByRole('option', { name: 'United States' });
page.getByRole('option', { name: 'United States', selected: true });
```

### Complete ARIA Roles Reference

```typescript
// Document Structure Roles
page.getByRole('article');                    // <article> or role="article"
page.getByRole('banner');                     // <header> (top-level) or role="banner"
page.getByRole('complementary');              // <aside> or role="complementary"
page.getByRole('contentinfo');                // <footer> (top-level) or role="contentinfo"
page.getByRole('figure');                     // <figure> or role="figure"
page.getByRole('main');                       // <main> or role="main"
page.getByRole('navigation');                 // <nav> or role="navigation"
page.getByRole('region', { name: 'Sidebar' }); // <section> with accessible name
page.getByRole('search');                     // role="search"

// Widget Roles
page.getByRole('button');                     // <button> or role="button"
page.getByRole('checkbox');                   // <input type="checkbox"> or role="checkbox"
page.getByRole('combobox');                   // <select> or role="combobox"
page.getByRole('grid');                       // <table> (interactive) or role="grid"
page.getByRole('gridcell');                   // <td> in grid or role="gridcell"
page.getByRole('link');                       // <a href> or role="link"
page.getByRole('listbox');                    // <select> or role="listbox"
page.getByRole('menu');                       // role="menu"
page.getByRole('menubar');                    // role="menubar"
page.getByRole('menuitem');                   // role="menuitem"
page.getByRole('menuitemcheckbox');           // role="menuitemcheckbox"
page.getByRole('menuitemradio');              // role="menuitemradio"
page.getByRole('option');                     // <option> or role="option"
page.getByRole('progressbar');                // <progress> or role="progressbar"
page.getByRole('radio');                      // <input type="radio"> or role="radio"
page.getByRole('scrollbar');                  // role="scrollbar"
page.getByRole('searchbox');                  // <input type="search"> or role="searchbox"
page.getByRole('slider');                     // <input type="range"> or role="slider"
page.getByRole('spinbutton');                 // <input type="number"> or role="spinbutton"
page.getByRole('switch');                     // role="switch"
page.getByRole('tab');                        // role="tab"
page.getByRole('tablist');                    // role="tablist"
page.getByRole('tabpanel');                   // role="tabpanel"
page.getByRole('textbox');                    // <input type="text">, <textarea> or role="textbox"
page.getByRole('treeitem');                   // role="treeitem"

// Table Roles
page.getByRole('table');                      // <table> or role="table"
page.getByRole('row');                        // <tr> or role="row"
page.getByRole('rowgroup');                   // <tbody>, <thead>, <tfoot> or role="rowgroup"
page.getByRole('rowheader');                  // <th scope="row"> or role="rowheader"
page.getByRole('columnheader');               // <th> or role="columnheader"
page.getByRole('cell');                       // <td> or role="cell"

// List Roles
page.getByRole('list');                       // <ul>, <ol> or role="list"
page.getByRole('listitem');                   // <li> or role="listitem"

// Dialog Roles
page.getByRole('dialog');                     // <dialog> or role="dialog"
page.getByRole('alertdialog');                // role="alertdialog"

// Alert Roles
page.getByRole('alert');                      // role="alert"
page.getByRole('status');                     // role="status"
page.getByRole('log');                        // role="log"
page.getByRole('marquee');                    // role="marquee"
page.getByRole('timer');                      // role="timer"

// Form Roles
page.getByRole('form');                       // <form> with accessible name or role="form"
page.getByRole('group');                      // <fieldset> or role="group"

// Landmark Roles
page.getByRole('application');                // role="application"
page.getByRole('document');                   // role="document"

// Image Roles
page.getByRole('img');                        // <img>, role="img", or SVG with role
page.getByRole('figure');                     // <figure> or role="figure"

// Separator
page.getByRole('separator');                  // <hr> or role="separator"

// Definition
page.getByRole('definition');                 // <dd> or role="definition"
page.getByRole('term');                       // <dt> or role="term"

// Tooltip
page.getByRole('tooltip');                    // role="tooltip"

// Tree
page.getByRole('tree');                       // role="tree"
page.getByRole('treeitem');                   // role="treeitem"
```

### Role Options

```typescript
// name - filter by accessible name
page.getByRole('button', { name: 'Submit' });
page.getByRole('button', { name: /submit/i }); // regex

// exact - exact string match (default: true for string, false for regex)
page.getByRole('button', { name: 'Submit', exact: true });
page.getByRole('button', { name: 'sub', exact: false }); // partial match

// checked - for checkbox/radio
page.getByRole('checkbox', { checked: true });
page.getByRole('checkbox', { checked: false });

// disabled - filter by disabled state
page.getByRole('button', { disabled: true });
page.getByRole('textbox', { disabled: false });

// expanded - for expandable elements
page.getByRole('button', { expanded: true });
page.getByRole('treeitem', { expanded: false });

// includeHidden - include hidden elements (default: false)
page.getByRole('button', { includeHidden: true });

// level - for headings
page.getByRole('heading', { level: 1 }); // h1
page.getByRole('heading', { level: 2 }); // h2

// pressed - for toggle buttons
page.getByRole('button', { pressed: true });
page.getByRole('button', { pressed: false });

// selected - for options
page.getByRole('option', { selected: true });
page.getByRole('tab', { selected: true });
```

### Complex Role Examples

```typescript
// Navigation menu items
const nav = page.getByRole('navigation');
const menuItems = nav.getByRole('link');

// Form with labeled inputs
const form = page.getByRole('form', { name: 'Contact Form' });
const emailInput = form.getByRole('textbox', { name: 'Email' });
const submitBtn = form.getByRole('button', { name: 'Send Message' });

// Table with specific cell
const table = page.getByRole('table', { name: 'Users' });
const row = table.getByRole('row', { name: 'John Doe' });
const editBtn = row.getByRole('button', { name: 'Edit' });

// Dialog with form
const dialog = page.getByRole('dialog', { name: 'Edit Profile' });
const nameInput = dialog.getByRole('textbox', { name: 'Full Name' });
const saveBtn = dialog.getByRole('button', { name: 'Save' });
```

---

## Text Locators

Text locators find elements by their text content. They are useful when you need to find elements by visible text.

### Basic Text Matching

```typescript
// Exact match (default)
page.getByText('Hello World');

// Partial match
page.getByText('Hello', { exact: false });

// Case-insensitive with regex
page.getByText(/hello world/i);

// Match text that starts with
page.getByText(/^Hello/);

// Match text that ends with
page.getByText(/World$/);

// Match text containing pattern
page.getByText(/Hello.*World/);
```

### Text Matching Options

```typescript
// exact - exact string match vs substring
page.getByText('Hello', { exact: true });  // matches only "Hello"
page.getByText('Hello', { exact: false }); // matches "Hello World", "Say Hello", etc.

// Note: regex patterns ignore the exact option
page.getByText(/Hello/);  // always partial match
page.getByText(/^Hello$/); // exact match with regex
```

### Advanced Text Patterns

```typescript
// Match numbers
page.getByText(/\d+ items/);              // "5 items", "100 items"
page.getByText(/\$[\d,]+\.\d{2}/);        // "$1,234.56"

// Match dates
page.getByText(/\d{1,2}\/\d{1,2}\/\d{4}/); // "12/25/2024"

// Match email patterns in text
page.getByText(/[\w.-]+@[\w.-]+\.\w+/);

// Match with word boundaries
page.getByText(/\bError\b/);              // matches "Error" but not "Errors"

// Match multiple possibilities
page.getByText(/Submit|Send|Save/);

// Match whitespace variations
page.getByText(/Hello\s+World/);          // any amount of whitespace
```

### Text Locator Scope

```typescript
// getByText searches within all descendant text nodes
// It finds the smallest element containing the text

// HTML: <div><span>Hello</span> World</div>
page.getByText('Hello');       // matches the <span>
page.getByText('Hello World'); // matches the <div>

// Be specific with role when text is ambiguous
page.getByRole('button').getByText('Submit');
page.getByRole('heading').getByText('Welcome');
```

---

## Label and Placeholder Locators

### Label Locators

Label locators find form controls by their associated label text. This is the recommended way to locate form inputs.

```typescript
// Basic label matching
page.getByLabel('Email');
page.getByLabel('Password');
page.getByLabel('Remember me');

// Partial match
page.getByLabel('Email', { exact: false }); // matches "Email Address"

// Regex match
page.getByLabel(/email/i);

// Labels work with various form controls
page.getByLabel('Username');     // <input>
page.getByLabel('Message');      // <textarea>
page.getByLabel('Country');      // <select>
page.getByLabel('Accept terms'); // <input type="checkbox">
page.getByLabel('Gender');       // <input type="radio">
```

### How Labels Are Associated

```typescript
// Explicit label with for attribute
// <label for="email">Email</label>
// <input id="email" type="text">
page.getByLabel('Email');

// Wrapped input
// <label>Email <input type="text"></label>
page.getByLabel('Email');

// aria-labelledby
// <span id="label-id">Email</span>
// <input aria-labelledby="label-id">
page.getByLabel('Email');

// aria-label
// <input aria-label="Email address">
page.getByLabel('Email address');

// Multiple labels
// <label for="field">First Label</label>
// <label for="field">Second Label</label>
// <input id="field">
page.getByLabel('First Label');
page.getByLabel('Second Label'); // Both work
```

### Placeholder Locators

```typescript
// Basic placeholder matching
page.getByPlaceholder('Enter your email');
page.getByPlaceholder('Search...');

// Partial match
page.getByPlaceholder('email', { exact: false });

// Regex match
page.getByPlaceholder(/enter.*email/i);

// Works with input and textarea
page.getByPlaceholder('Type your message');
```

### Alt Text Locators

```typescript
// For images
page.getByAltText('Company Logo');
page.getByAltText('User avatar');

// Partial match
page.getByAltText('Logo', { exact: false });

// Regex
page.getByAltText(/logo/i);

// Works with:
// <img alt="Company Logo">
// <input type="image" alt="Submit">
// <area alt="Link area">
```

### Title Locators

```typescript
// Match title attribute
page.getByTitle('Close dialog');
page.getByTitle('More information');

// Partial match
page.getByTitle('Close', { exact: false });

// Regex
page.getByTitle(/close/i);

// Works with any element with title attribute
// <button title="Close dialog">X</button>
// <abbr title="Cascading Style Sheets">CSS</abbr>
// <a title="View documentation" href="/docs">Docs</a>
```

---

## Test ID Locators

Test ID locators provide a reliable way to locate elements using data attributes, making tests resilient to text and structure changes.

### Basic Usage

```typescript
// Default attribute: data-testid
page.getByTestId('submit-button');
page.getByTestId('user-email-input');
page.getByTestId('login-form');

// HTML:
// <button data-testid="submit-button">Submit</button>
// <input data-testid="user-email-input">
// <form data-testid="login-form">
```

### Configuring Test ID Attribute

```typescript
// playwright.config.ts
import { defineConfig } from '@playwright/test';

export default defineConfig({
  use: {
    // Use a different attribute
    testIdAttribute: 'data-test-id',
  },
});

// Now matches: <button data-test-id="submit">

// Other common configurations:
// testIdAttribute: 'data-cy'        // For Cypress migration
// testIdAttribute: 'data-qa'        // QA-specific IDs
// testIdAttribute: 'data-automation-id'
```

### Setting Test ID at Runtime

```typescript
// In test file - override for specific tests
import { test } from '@playwright/test';

test.use({ testIdAttribute: 'data-custom-id' });

test('uses custom test id', async ({ page }) => {
  await page.getByTestId('my-button').click();
});
```

### Test ID Best Practices

```typescript
// Use descriptive, hierarchical names
page.getByTestId('header-nav-home-link');
page.getByTestId('checkout-form-submit-btn');
page.getByTestId('product-card-add-to-cart');

// Use consistent naming conventions
// kebab-case: data-testid="user-profile-avatar"
// camelCase: data-testid="userProfileAvatar"
// BEM-style: data-testid="header__nav--mobile"

// Combine with other locators for specificity
const card = page.getByTestId('product-card').first();
const addBtn = card.getByRole('button', { name: 'Add to Cart' });
```

---

## CSS Selectors

CSS selectors provide powerful element selection using standard CSS syntax.

### Basic CSS Selectors

```typescript
// Tag name
page.locator('button');
page.locator('input');
page.locator('div');

// Class
page.locator('.primary');
page.locator('.btn.btn-primary');
page.locator('.card.featured');

// ID
page.locator('#submit-btn');
page.locator('#login-form');

// Attribute
page.locator('[type="submit"]');
page.locator('[data-testid="login"]');
page.locator('[href="/about"]');
```

### Attribute Selectors

```typescript
// Exact match
page.locator('[type="text"]');

// Attribute presence
page.locator('[disabled]');
page.locator('[required]');

// Starts with
page.locator('[href^="https://"]');
page.locator('[class^="btn-"]');

// Ends with
page.locator('[href$=".pdf"]');
page.locator('[src$=".png"]');

// Contains
page.locator('[href*="example"]');
page.locator('[class*="active"]');

// Contains word (space-separated)
page.locator('[class~="featured"]');

// Starts with (hyphen-separated)
page.locator('[lang|="en"]'); // matches en, en-US, en-GB

// Case-insensitive (CSS4)
page.locator('[type="submit" i]');
```

### Combinators

```typescript
// Descendant (any level)
page.locator('form input');
page.locator('.container .card .title');

// Direct child
page.locator('ul > li');
page.locator('.menu > .item');

// Adjacent sibling (immediately after)
page.locator('h1 + p');
page.locator('label + input');

// General sibling (any sibling after)
page.locator('h1 ~ p');
page.locator('.header ~ .content');
```

### Pseudo-classes

```typescript
// State pseudo-classes
page.locator('input:focus');
page.locator('button:hover');
page.locator('input:disabled');
page.locator('input:enabled');
page.locator('input:checked');
page.locator('option:selected');
page.locator(':required');
page.locator(':optional');
page.locator(':valid');
page.locator(':invalid');

// Structural pseudo-classes
page.locator('li:first-child');
page.locator('li:last-child');
page.locator('li:nth-child(2)');
page.locator('li:nth-child(odd)');
page.locator('li:nth-child(even)');
page.locator('li:nth-child(3n)');      // every 3rd
page.locator('li:nth-child(3n+1)');    // 1st, 4th, 7th...
page.locator('li:nth-last-child(2)');  // 2nd from end
page.locator('p:first-of-type');
page.locator('p:last-of-type');
page.locator('p:nth-of-type(2)');
page.locator(':only-child');
page.locator(':only-of-type');
page.locator(':empty');

// Negation
page.locator('button:not(.disabled)');
page.locator('input:not([type="hidden"])');
page.locator('li:not(:first-child)');

// Matches any of the selectors
page.locator(':is(h1, h2, h3)');
page.locator('.btn:is(.primary, .secondary)');

// Has (contains element)
page.locator('div:has(> img)');
page.locator('form:has(input[type="email"])');
```

### Playwright-specific CSS Extensions

```typescript
// Visible elements only
page.locator('button:visible');

// Text matching
page.locator('button:text("Submit")');
page.locator('button:text-is("Submit")'); // exact match
page.locator('button:text-matches("sub", "i")'); // regex

// Has text (includes descendants)
page.locator('div:has-text("Hello")');

// Right/left of (visual position)
page.locator('button:right-of(#sidebar)');
page.locator('input:left-of(button)');

// Above/below
page.locator('input:above(#footer)');
page.locator('button:below(#header)');

// Near (within 50px by default)
page.locator('button:near(#target)');
page.locator('button:near(#target, 100)'); // within 100px
```

### Complex CSS Examples

```typescript
// Form inputs with validation states
page.locator('input:invalid:not(:placeholder-shown)');

// Table rows with specific content
page.locator('tr:has(td:text("Active"))');

// Navigation items excluding current
page.locator('nav a:not([aria-current="page"])');

// Cards with images
page.locator('.card:has(img)');

// Buttons in specific containers
page.locator('.modal.open button.primary');

// Multiple attribute conditions
page.locator('input[type="text"][required]:not([disabled])');
```

---

## XPath Selectors

XPath provides powerful element selection using path expressions. Use when CSS selectors are insufficient.

### Basic XPath

```typescript
// Explicit XPath prefix
page.locator('xpath=//button');
page.locator('xpath=//input[@type="text"]');

// Shorthand (starts with // or /)
page.locator('//button[@type="submit"]');
page.locator('/html/body/div/form');
```

### XPath Axes

```typescript
// Descendant (default with //)
page.locator('//div//span');

// Direct child
page.locator('//div/span');

// Parent
page.locator('//span/parent::div');
page.locator('//input/parent::*');

// Ancestor
page.locator('//span/ancestor::div');
page.locator('//input/ancestor::form');

// Following sibling
page.locator('//h1/following-sibling::p');

// Preceding sibling
page.locator('//p/preceding-sibling::h1');

// Following (all nodes after)
page.locator('//h1/following::p');

// Preceding (all nodes before)
page.locator('//footer/preceding::header');
```

### XPath Predicates

```typescript
// Position
page.locator('//li[1]');        // first
page.locator('//li[last()]');   // last
page.locator('//li[position()<=3]'); // first 3

// Attribute
page.locator('//input[@type="text"]');
page.locator('//a[@href="/about"]');
page.locator('//button[@disabled]');

// Text content
page.locator('//button[text()="Submit"]');
page.locator('//button[contains(text(), "Sub")]');
page.locator('//h1[normalize-space()="Welcome"]');

// Multiple conditions
page.locator('//input[@type="text" and @required]');
page.locator('//button[@type="submit" or @type="button"]');
page.locator('//input[not(@disabled)]');
```

### XPath Functions

```typescript
// Text functions
page.locator('//button[contains(text(), "Submit")]');
page.locator('//p[starts-with(text(), "Error")]');
page.locator('//p[normalize-space()="Clean text"]');
page.locator('//p[string-length(text()) > 100]');

// Attribute functions
page.locator('//a[contains(@href, "example")]');
page.locator('//img[starts-with(@src, "/images")]');
page.locator('//*[contains(@class, "active")]');

// Node functions
page.locator('//div[count(child::p) > 2]');
page.locator('//ul[count(li) = 5]');
page.locator('//*[local-name()="button"]');
```

### Complex XPath Examples

```typescript
// Find element by partial class
page.locator('//*[contains(@class, "btn") and contains(@class, "primary")]');

// Find by data attribute
page.locator('//button[@data-testid="submit"]');

// Table cell by row content
page.locator('//tr[td[text()="John"]]/td[3]');

// Find input by label
page.locator('//label[text()="Email"]/following-sibling::input');

// Find within specific context
page.locator('//div[@id="modal"]//button[text()="Close"]');

// Find by multiple text conditions
page.locator('//p[contains(text(), "Error") and contains(text(), "invalid")]');

// Find elements with specific child
page.locator('//div[child::img and child::h2]');
```

---

## Locator Chaining

Locator chaining allows you to narrow down element selection by combining multiple locators.

### Basic Chaining

```typescript
// Find within a container
const dialog = page.getByRole('dialog');
const submitBtn = dialog.getByRole('button', { name: 'Submit' });
const cancelBtn = dialog.getByRole('button', { name: 'Cancel' });

// Chain multiple levels
const nav = page.getByRole('navigation');
const menu = nav.getByRole('list');
const items = menu.getByRole('listitem');

// Use with CSS/XPath
const container = page.locator('#main-content');
const cards = container.locator('.card');
const firstCardTitle = cards.first().locator('.title');
```

### Parent/Child Relationships

```typescript
// locator() creates child selector
const parent = page.locator('.parent');
const child = parent.locator('.child');

// getBy* methods also chain
const form = page.locator('form');
const emailInput = form.getByLabel('Email');
const passwordInput = form.getByLabel('Password');

// Combining different locator types
const article = page.getByRole('article');
const header = article.locator('header');
const title = header.getByRole('heading');
```

### Sibling Selection

```typescript
// Using XPath for siblings
const input = page.locator('//label[text()="Email"]/following-sibling::input');

// Using CSS adjacent sibling
const nextParagraph = page.locator('h1 + p');

// Using filter with has to find siblings indirectly
const row = page.getByRole('row').filter({ hasText: 'John Doe' });
const editBtn = row.getByRole('button', { name: 'Edit' });
```

### Locator Method Chaining

```typescript
// and() - both conditions must match
const publishedArticle = page
  .getByRole('article')
  .and(page.locator('.published'));

// or() - either condition matches
const interactiveElement = page
  .getByRole('button')
  .or(page.getByRole('link'));

// Combining and/or
const targetElement = page
  .locator('.card')
  .and(page.locator(':visible'))
  .or(page.locator('.featured'));
```

---

## Filtering Locators

Filtering allows you to narrow down a set of elements based on various criteria.

### Filter by Text

```typescript
// hasText - contains text (partial match)
page.getByRole('listitem').filter({ hasText: 'Product' });
page.locator('.card').filter({ hasText: 'Featured' });

// hasText with regex
page.getByRole('listitem').filter({ hasText: /\$\d+/ });

// hasNotText - excludes elements with text
page.getByRole('listitem').filter({ hasNotText: 'Out of stock' });
page.locator('.item').filter({ hasNotText: /sold out/i });
```

### Filter by Child Locator

```typescript
// has - must contain matching element
page.getByRole('listitem').filter({
  has: page.getByRole('button', { name: 'Add to cart' })
});

page.locator('.card').filter({
  has: page.locator('img')
});

// Multiple conditions
page.getByRole('row').filter({
  has: page.getByRole('cell', { name: 'Active' }),
  hasText: 'John'
});
```

### Filter by Absence

```typescript
// hasNot - must NOT contain matching element
page.getByRole('listitem').filter({
  hasNot: page.getByText('Out of stock')
});

page.locator('.card').filter({
  hasNot: page.locator('.sold-out-badge')
});

// hasNotText - must NOT contain text
page.getByRole('listitem').filter({
  hasNotText: 'Unavailable'
});
```

### Complex Filtering

```typescript
// Combine multiple filter criteria
const availableProducts = page
  .getByRole('listitem')
  .filter({ hasText: 'In Stock' })
  .filter({ hasNot: page.locator('.discontinued') });

// Filter by multiple conditions
const activeUsers = page.getByRole('row').filter({
  has: page.getByRole('cell', { name: 'Active' }),
  hasNot: page.getByRole('cell', { name: 'Admin' })
});

// Chained filtering
const items = page
  .locator('.product')
  .filter({ has: page.locator('.in-stock') })
  .filter({ hasText: /^\$[0-9]+$/ })
  .filter({ hasNot: page.locator('.members-only') });
```

### Filter Examples

```typescript
// Find table row by multiple cells
const userRow = page.getByRole('row').filter({
  has: page.getByRole('cell', { name: 'john@example.com' })
});

// Find card with specific content
const productCard = page.locator('.product-card').filter({
  hasText: 'Laptop',
  has: page.getByRole('button', { name: 'Buy Now' })
});

// Find available items excluding premium
const standardItems = page
  .getByRole('listitem')
  .filter({ hasNotText: 'Premium' })
  .filter({ hasNot: page.locator('.locked') });
```

---

## Nth Locators

Nth locators allow you to select specific elements from a set of matching elements.

### Basic Nth Selection

```typescript
// first() - first matching element
page.getByRole('listitem').first();
page.locator('.card').first();

// last() - last matching element
page.getByRole('listitem').last();
page.locator('.card').last();

// nth() - specific index (0-based)
page.getByRole('listitem').nth(0);  // same as first()
page.getByRole('listitem').nth(1);  // second item
page.getByRole('listitem').nth(2);  // third item

// Negative index (from end)
page.getByRole('listitem').nth(-1); // last
page.getByRole('listitem').nth(-2); // second to last
```

### Combining Nth with Other Locators

```typescript
// Get first matching within container
const dialog = page.getByRole('dialog');
const firstButton = dialog.getByRole('button').first();

// Chain nth selections
const table = page.getByRole('table');
const firstRow = table.getByRole('row').nth(1); // skip header
const firstCell = firstRow.getByRole('cell').first();

// Filter then select nth
const activeItems = page
  .getByRole('listitem')
  .filter({ hasText: 'Active' })
  .nth(0);
```

### Getting Count

```typescript
// Count matching elements
const count = await page.getByRole('listitem').count();
console.log(`Found ${count} items`);

// Use count in logic
const buttonCount = await page.getByRole('button').count();
if (buttonCount > 0) {
  await page.getByRole('button').first().click();
}

// Iterate over elements
const items = page.getByRole('listitem');
const count = await items.count();
for (let i = 0; i < count; i++) {
  const text = await items.nth(i).textContent();
  console.log(text);
}
```

### All Elements

```typescript
// Get all elements as array
const allButtons = await page.getByRole('button').all();
for (const button of allButtons) {
  await button.click();
}

// Get all text contents
const allTexts = await page.getByRole('listitem').allTextContents();
console.log(allTexts); // ['Item 1', 'Item 2', ...]

// Get all inner texts
const innerTexts = await page.getByRole('listitem').allInnerTexts();
```

---

## Frame Locators

Frame locators allow you to interact with elements inside iframes.

### Basic Frame Access

```typescript
// By name or id attribute
const frame = page.frameLocator('iframe[name="content"]');
const frame2 = page.frameLocator('#my-iframe');

// By src attribute
const frame3 = page.frameLocator('iframe[src*="external.com"]');

// Access elements within frame
await frame.getByRole('button', { name: 'Submit' }).click();
await frame.getByLabel('Email').fill('test@example.com');
```

### Multiple Frames

```typescript
// First iframe
const firstFrame = page.frameLocator('iframe').first();

// Nth iframe
const secondFrame = page.frameLocator('iframe').nth(1);

// Last iframe
const lastFrame = page.frameLocator('iframe').last();
```

### Nested Frames

```typescript
// Frame inside frame
const outerFrame = page.frameLocator('#outer-frame');
const innerFrame = outerFrame.frameLocator('#inner-frame');
await innerFrame.getByRole('button').click();
```

### Frame Owner

```typescript
// Get the iframe element itself (not contents)
const frameElement = page.locator('iframe[name="content"]');
await frameElement.getAttribute('src');

// Check iframe attributes
const frameSrc = await page.locator('#my-iframe').getAttribute('src');
```

### Working with Frame Content

```typescript
// Fill form inside iframe
const frame = page.frameLocator('#payment-frame');
await frame.getByLabel('Card Number').fill('4111111111111111');
await frame.getByLabel('Expiry').fill('12/25');
await frame.getByLabel('CVV').fill('123');
await frame.getByRole('button', { name: 'Pay' }).click();

// Wait for element in frame
await frame.getByText('Payment Successful').waitFor();

// Assert within frame
await expect(frame.getByRole('heading')).toHaveText('Checkout');
```

---

## Locator Actions

Locator actions allow you to interact with elements. All actions auto-wait for the element to be actionable.

### Click Actions

```typescript
// Basic click
await locator.click();

// Double click
await locator.dblclick();

// Right click (context menu)
await locator.click({ button: 'right' });

// Middle click
await locator.click({ button: 'middle' });

// Click at position (relative to element)
await locator.click({ position: { x: 10, y: 20 } });

// Click with modifiers
await locator.click({ modifiers: ['Shift'] });
await locator.click({ modifiers: ['Control'] });
await locator.click({ modifiers: ['Alt'] });
await locator.click({ modifiers: ['Meta'] }); // Command on Mac
await locator.click({ modifiers: ['Control', 'Shift'] });

// Force click (bypass actionability checks)
await locator.click({ force: true });

// Click with delay between mousedown and mouseup
await locator.click({ delay: 100 });

// No wait after click
await locator.click({ noWaitAfter: true });

// Click with custom timeout
await locator.click({ timeout: 5000 });

// Click count (triple click to select line)
await locator.click({ clickCount: 3 });

// Trial run (check actionability without clicking)
await locator.click({ trial: true });
```

### Input Actions

```typescript
// Fill input (clears first)
await locator.fill('Hello World');

// Clear input
await locator.clear();

// Type (key by key, with events)
await locator.pressSequentially('Hello', { delay: 100 });

// Press single key
await locator.press('Enter');
await locator.press('Tab');
await locator.press('Escape');
await locator.press('Backspace');

// Key combinations
await locator.press('Control+a');
await locator.press('Control+c');
await locator.press('Control+v');
await locator.press('Shift+Tab');
await locator.press('Meta+Enter'); // Command+Enter on Mac

// Special keys
await locator.press('ArrowDown');
await locator.press('ArrowUp');
await locator.press('ArrowLeft');
await locator.press('ArrowRight');
await locator.press('PageDown');
await locator.press('PageUp');
await locator.press('Home');
await locator.press('End');
await locator.press('F1');

// Input with timeout
await locator.fill('text', { timeout: 5000 });
```

### Select Actions

```typescript
// Select by value
await locator.selectOption('value');

// Select by label
await locator.selectOption({ label: 'Option 1' });

// Select by index
await locator.selectOption({ index: 2 });

// Select multiple (for multi-select)
await locator.selectOption(['value1', 'value2']);
await locator.selectOption([
  { label: 'Option 1' },
  { label: 'Option 2' }
]);

// Clear selection
await locator.selectOption([]);
```

### Checkbox and Radio Actions

```typescript
// Check checkbox
await locator.check();

// Uncheck checkbox
await locator.uncheck();

// Set checked state
await locator.setChecked(true);
await locator.setChecked(false);

// Check with force
await locator.check({ force: true });

// Check specific radio button
await page.getByRole('radio', { name: 'Option A' }).check();
```

### Hover and Focus

```typescript
// Hover over element
await locator.hover();

// Hover at position
await locator.hover({ position: { x: 0, y: 0 } });

// Hover with modifiers
await locator.hover({ modifiers: ['Shift'] });

// Focus element
await locator.focus();

// Blur element (remove focus)
await locator.blur();
```

### Drag and Drop

```typescript
// Drag to another element
await source.dragTo(target);

// Drag with options
await source.dragTo(target, {
  sourcePosition: { x: 10, y: 10 },
  targetPosition: { x: 20, y: 20 }
});

// Manual drag operations
await locator.hover();
await page.mouse.down();
await page.mouse.move(100, 200);
await page.mouse.up();
```

### File Upload

```typescript
// Set single file
await locator.setInputFiles('/path/to/file.pdf');

// Set multiple files
await locator.setInputFiles(['/path/to/file1.pdf', '/path/to/file2.pdf']);

// Clear file input
await locator.setInputFiles([]);

// Set file with buffer
await locator.setInputFiles({
  name: 'file.txt',
  mimeType: 'text/plain',
  buffer: Buffer.from('file content')
});

// Multiple files with buffers
await locator.setInputFiles([
  {
    name: 'file1.txt',
    mimeType: 'text/plain',
    buffer: Buffer.from('content 1')
  },
  {
    name: 'file2.txt',
    mimeType: 'text/plain',
    buffer: Buffer.from('content 2')
  }
]);
```

### Scroll Actions

```typescript
// Scroll element into view
await locator.scrollIntoViewIfNeeded();

// Scroll with wheel
await locator.hover();
await page.mouse.wheel(0, 100); // scroll down
await page.mouse.wheel(0, -100); // scroll up
```

### Screenshot

```typescript
// Screenshot element
await locator.screenshot({ path: 'element.png' });

// Screenshot with options
await locator.screenshot({
  path: 'element.png',
  type: 'png', // or 'jpeg'
  quality: 80, // jpeg only
  omitBackground: true,
  animations: 'disabled',
  scale: 'device' // or 'css'
});
```

---

## Locator State

Methods for waiting and checking element state.

### Wait For Element

```typescript
// Wait for element to be visible (default)
await locator.waitFor();
await locator.waitFor({ state: 'visible' });

// Wait for element to be hidden
await locator.waitFor({ state: 'hidden' });

// Wait for element to be attached to DOM
await locator.waitFor({ state: 'attached' });

// Wait for element to be detached from DOM
await locator.waitFor({ state: 'detached' });

// Wait with timeout
await locator.waitFor({ timeout: 10000 });
await locator.waitFor({ state: 'visible', timeout: 5000 });
```

### State Checks

```typescript
// Check visibility
const isVisible = await locator.isVisible();

// Check if hidden
const isHidden = await locator.isHidden();

// Check if enabled
const isEnabled = await locator.isEnabled();

// Check if disabled
const isDisabled = await locator.isDisabled();

// Check if editable
const isEditable = await locator.isEditable();

// Check if checked (checkbox/radio)
const isChecked = await locator.isChecked();
```

### State Check Options

```typescript
// With timeout (waits until state or timeout)
const isVisible = await locator.isVisible({ timeout: 5000 });
const isEnabled = await locator.isEnabled({ timeout: 3000 });

// Note: State checks with timeout return boolean, don't throw
```

### Element State in Conditions

```typescript
// Conditional actions based on state
if (await page.getByRole('dialog').isVisible()) {
  await page.getByRole('button', { name: 'Close' }).click();
}

// Wait then check
await locator.waitFor({ state: 'visible' });
const enabled = await locator.isEnabled();

// Alternative: use assertions for waiting
await expect(locator).toBeVisible();
await expect(locator).toBeEnabled();
```

---

## Getting Values

Methods for extracting information from elements.

### Text Content

```typescript
// Get text content (includes hidden text)
const text = await locator.textContent();

// Get inner text (visible text only)
const innerText = await locator.innerText();

// Get all text contents (multiple elements)
const allTexts = await locator.allTextContents();

// Get all inner texts
const allInnerTexts = await locator.allInnerTexts();
```

### Input Values

```typescript
// Get input value
const value = await locator.inputValue();

// Works with:
// <input>, <textarea>, <select>
```

### Attributes

```typescript
// Get attribute value
const href = await locator.getAttribute('href');
const src = await locator.getAttribute('src');
const dataId = await locator.getAttribute('data-id');
const className = await locator.getAttribute('class');

// Returns null if attribute doesn't exist
const value = await locator.getAttribute('data-custom');
if (value !== null) {
  console.log(value);
}
```

### HTML Content

```typescript
// Get inner HTML
const html = await locator.innerHTML();

// Get outer HTML (includes element itself)
const outerHtml = await locator.evaluate(el => el.outerHTML);
```

### Bounding Box

```typescript
// Get element position and size
const box = await locator.boundingBox();
if (box) {
  console.log(box.x, box.y, box.width, box.height);
}

// Returns null if element is not visible
```

### Element Count

```typescript
// Count matching elements
const count = await locator.count();

// Use in assertions
expect(await locator.count()).toBe(5);
```

### Evaluate on Element

```typescript
// Run JavaScript on element
const tagName = await locator.evaluate(el => el.tagName);

// Get computed style
const color = await locator.evaluate(el =>
  getComputedStyle(el).color
);

// Get custom property
const data = await locator.evaluate(el => el.dataset.custom);

// Pass arguments
const result = await locator.evaluate(
  (el, suffix) => el.textContent + suffix,
  '!'
);

// Evaluate on all matching elements
const allIds = await locator.evaluateAll(
  elements => elements.map(el => el.id)
);
```

---

## Strict Mode and Multiple Matches

Playwright's strict mode ensures actions target exactly one element.

### Strict Mode Behavior

```typescript
// Throws error if multiple elements match
await page.getByRole('button').click();
// Error: locator.click: Error: strict mode violation:
// getByRole('button') resolved to 3 elements

// Solutions:

// 1. Be more specific
await page.getByRole('button', { name: 'Submit' }).click();

// 2. Use nth selectors
await page.getByRole('button').first().click();
await page.getByRole('button').nth(0).click();

// 3. Filter
await page.getByRole('button').filter({ hasText: 'Submit' }).click();

// 4. Chain from container
await page.getByRole('dialog').getByRole('button').click();
```

### When Strict Mode Applies

```typescript
// Strict mode applies to single-element actions:
await locator.click();        // strict
await locator.fill('text');   // strict
await locator.check();        // strict
await locator.textContent();  // strict
await locator.getAttribute(); // strict

// Non-strict (works with multiple):
await locator.count();            // returns count
await locator.all();              // returns all
await locator.allTextContents();  // returns all texts
await locator.first();            // returns first
await locator.nth(0);             // returns specific one
```

### Handling Multiple Elements

```typescript
// Iterate over all matching elements
const buttons = page.getByRole('button');
const count = await buttons.count();
for (let i = 0; i < count; i++) {
  await buttons.nth(i).click();
}

// Or use all()
for (const button of await buttons.all()) {
  await button.click();
}

// Collect all values
const texts = await page.getByRole('listitem').allTextContents();
```

---

## Locator Assertions

Playwright Test provides assertions that auto-wait for conditions.

### Visibility Assertions

```typescript
// Assert visible
await expect(locator).toBeVisible();

// Assert hidden
await expect(locator).toBeHidden();

// Assert not visible (alias)
await expect(locator).not.toBeVisible();

// With timeout
await expect(locator).toBeVisible({ timeout: 10000 });
```

### State Assertions

```typescript
// Enabled/disabled
await expect(locator).toBeEnabled();
await expect(locator).toBeDisabled();

// Editable
await expect(locator).toBeEditable();

// Checked (checkbox/radio)
await expect(locator).toBeChecked();
await expect(locator).not.toBeChecked();

// Focused
await expect(locator).toBeFocused();

// Attached to DOM
await expect(locator).toBeAttached();

// Empty
await expect(locator).toBeEmpty();
```

### Text Assertions

```typescript
// Exact text
await expect(locator).toHaveText('Hello World');

// Partial text
await expect(locator).toContainText('Hello');

// Regex text
await expect(locator).toHaveText(/hello/i);

// Multiple elements text
await expect(locator).toHaveText(['Item 1', 'Item 2', 'Item 3']);

// Ignore whitespace
await expect(locator).toHaveText('Hello', { ignoreCase: true });

// Use inner text (visible only)
await expect(locator).toHaveText('Hello', { useInnerText: true });
```

### Value Assertions

```typescript
// Input value
await expect(locator).toHaveValue('test@example.com');
await expect(locator).toHaveValue(/\w+@\w+/);

// Multiple values (multi-select)
await expect(locator).toHaveValues(['option1', 'option2']);
```

### Attribute Assertions

```typescript
// Has attribute
await expect(locator).toHaveAttribute('href', '/about');
await expect(locator).toHaveAttribute('data-testid', 'submit');

// Regex value
await expect(locator).toHaveAttribute('href', /\/about/);

// Attribute exists (any value)
await expect(locator).toHaveAttribute('disabled');

// Has class
await expect(locator).toHaveClass('btn-primary');
await expect(locator).toHaveClass(/active/);

// Multiple classes
await expect(locator).toHaveClass(['btn', 'btn-primary']);

// Has ID
await expect(locator).toHaveId('submit-button');
```

### CSS Assertions

```typescript
// CSS property value
await expect(locator).toHaveCSS('color', 'rgb(255, 0, 0)');
await expect(locator).toHaveCSS('display', 'flex');
await expect(locator).toHaveCSS('font-size', '16px');
```

### Count Assertions

```typescript
// Element count
await expect(locator).toHaveCount(5);
await expect(page.getByRole('listitem')).toHaveCount(3);
```

### Screenshot Assertions

```typescript
// Visual comparison
await expect(locator).toHaveScreenshot();
await expect(locator).toHaveScreenshot('button.png');

// With options
await expect(locator).toHaveScreenshot({
  maxDiffPixels: 100,
  threshold: 0.2
});
```

### Negating Assertions

```typescript
// Use .not for negation
await expect(locator).not.toBeVisible();
await expect(locator).not.toHaveText('Error');
await expect(locator).not.toBeChecked();
await expect(locator).not.toHaveClass('disabled');
```

### Soft Assertions

```typescript
// Soft assertions don't stop test execution
await expect.soft(locator).toHaveText('Hello');
await expect.soft(locator).toBeVisible();
// Test continues even if assertions fail
// Failures are reported at the end
```

### Assertion Options

```typescript
// Custom timeout
await expect(locator).toBeVisible({ timeout: 10000 });

// Custom error message
await expect(locator, 'Submit button should be visible').toBeVisible();

// Polling interval
await expect.poll(async () => {
  return await locator.count();
}, { timeout: 10000, intervals: [250, 500, 1000] }).toBe(5);
```

---

## Best Practices

### Locator Priority (Most to Least Recommended)

```typescript
// 1. Role-based locators (BEST)
page.getByRole('button', { name: 'Submit' });

// 2. Label-based locators
page.getByLabel('Email');

// 3. Placeholder locators
page.getByPlaceholder('Enter email');

// 4. Text locators
page.getByText('Welcome');

// 5. Test ID locators
page.getByTestId('submit-btn');

// 6. CSS/XPath (AVOID when possible)
page.locator('#submit-btn');
```

### Make Locators Resilient

```typescript
// BAD: Brittle selectors
page.locator('.sc-AxjAm.bMPJWb'); // generated class names
page.locator('body > div > div > form > button'); // positional
page.locator('[style*="color: red"]'); // style-based

// GOOD: Semantic selectors
page.getByRole('button', { name: 'Submit' });
page.getByLabel('Email address');
page.getByTestId('checkout-submit');
```

### Use Accessible Names

```typescript
// Buttons with accessible names
page.getByRole('button', { name: 'Close dialog' }); // aria-label
page.getByRole('button', { name: 'Submit form' }); // button text

// Images with alt text
page.getByAltText('Company logo');

// Form inputs with labels
page.getByLabel('Password');
```

### Scoped Locators

```typescript
// Scope to container first
const modal = page.getByRole('dialog');
const form = modal.getByRole('form');
const submitBtn = form.getByRole('button', { name: 'Submit' });

// Avoid global searches when context is known
// BAD
await page.getByRole('button', { name: 'Submit' }).click(); // which submit?

// GOOD
const checkoutForm = page.getByTestId('checkout-form');
await checkoutForm.getByRole('button', { name: 'Submit' }).click();
```

### Avoid Index-Based Selectors

```typescript
// BAD: Fragile position-based
page.locator('button').nth(3);
page.locator('li:nth-child(5)');

// GOOD: Content-based selection
page.getByRole('button', { name: 'Delete' });
page.getByRole('listitem').filter({ hasText: 'Product Name' });
```

### Handle Dynamic Content

```typescript
// Use regex for dynamic text
page.getByText(/Order #\d+/);
page.getByRole('heading', { name: /Welcome, .+/ });

// Filter for dynamic lists
page.getByRole('row').filter({ hasText: 'user@example.com' });

// Use test IDs for truly dynamic elements
page.getByTestId('dynamic-content');
```

### Wait Appropriately

```typescript
// Let auto-waiting work
await page.getByRole('button').click(); // auto-waits

// Use assertions for complex waits
await expect(page.getByText('Success')).toBeVisible();

// Explicit wait only when necessary
await page.getByRole('dialog').waitFor({ state: 'visible' });
```

---

## Debugging Locators

### Using Codegen

```bash
# Generate tests with locators
npx playwright codegen http://localhost:3000

# Generate tests for specific viewport
npx playwright codegen --viewport-size=800,600 http://localhost:3000

# Generate tests for mobile
npx playwright codegen --device="iPhone 13" http://localhost:3000

# Save to file
npx playwright codegen -o tests/generated.spec.ts http://localhost:3000
```

### Using the Inspector

```bash
# Run with inspector
npx playwright test --debug

# Or with UI mode
npx playwright test --ui
```

```typescript
// Pause in test
await page.pause();

// After pause, use Inspector to:
// - Pick elements and see suggested locators
// - Step through test
// - Edit locators live
```

### Debugging in Test

```typescript
// Log locator info
const locator = page.getByRole('button');
console.log('Count:', await locator.count());
console.log('Texts:', await locator.allTextContents());

// Highlight element
await locator.highlight();

// Screenshot for debugging
await locator.screenshot({ path: 'debug-element.png' });

// Evaluate selector match
const elements = await locator.evaluateAll(els => els.length);
console.log('Matched elements:', elements);
```

### Locator Testing in Console

```typescript
// In browser console (via page.evaluate)
await page.evaluate(() => {
  // Test CSS selector
  console.log(document.querySelectorAll('button.primary'));

  // Test XPath
  console.log(document.evaluate(
    '//button[@type="submit"]',
    document,
    null,
    XPathResult.ORDERED_NODE_SNAPSHOT_TYPE,
    null
  ).snapshotLength);
});
```

### Common Debugging Commands

```bash
# Show test trace
npx playwright show-trace trace.zip

# Run single test with debug
npx playwright test -g "test name" --debug

# Run with headed browser
npx playwright test --headed

# Run with slow motion
npx playwright test --headed --slowMo=1000
```

### Playwright Trace Viewer

```typescript
// Enable tracing in config
// playwright.config.ts
export default defineConfig({
  use: {
    trace: 'on-first-retry', // or 'on', 'off', 'retain-on-failure'
  },
});

// View trace
// npx playwright show-trace test-results/trace.zip
```

### Testing Locator Resolution

```typescript
// Check what a locator matches
test('debug locator', async ({ page }) => {
  await page.goto('/');

  const locator = page.getByRole('button');

  // See how many elements match
  const count = await locator.count();
  console.log(`Found ${count} buttons`);

  // See all texts
  const texts = await locator.allTextContents();
  console.log('Button texts:', texts);

  // Verify specific locator
  const submitBtn = page.getByRole('button', { name: 'Submit' });
  await expect(submitBtn).toHaveCount(1);
});
```

### Using Test Generator for Locators

```typescript
// In test, use locator suggestions
test('find best locator', async ({ page }) => {
  await page.goto('/');

  // Pause and use picker
  await page.pause();

  // In Inspector:
  // 1. Click "Pick locator" tool
  // 2. Click element on page
  // 3. See suggested locator
  // 4. Copy to clipboard
});
```

---

## Quick Reference

### Most Common Locators

```typescript
// Interactive elements
page.getByRole('button', { name: 'Submit' });
page.getByRole('link', { name: 'Home' });
page.getByRole('textbox', { name: 'Email' });
page.getByRole('checkbox', { name: 'Remember me' });
page.getByRole('combobox', { name: 'Country' });

// By content
page.getByText('Welcome');
page.getByLabel('Password');
page.getByPlaceholder('Search');
page.getByTestId('submit-btn');

// CSS fallback
page.locator('#element-id');
page.locator('.class-name');
```

### Most Common Actions

```typescript
await locator.click();
await locator.fill('text');
await locator.check();
await locator.selectOption('value');
await locator.press('Enter');
await locator.hover();
```

### Most Common Assertions

```typescript
await expect(locator).toBeVisible();
await expect(locator).toHaveText('Expected text');
await expect(locator).toHaveValue('expected value');
await expect(locator).toBeEnabled();
await expect(locator).toHaveCount(5);
```

### Most Common Filters

```typescript
locator.filter({ hasText: 'text' });
locator.filter({ has: page.getByRole('button') });
locator.first();
locator.nth(0);
```
