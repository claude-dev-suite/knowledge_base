# axe-core

> Source: https://github.com/dequelabs/axe-core

## Overview

axe-core is an open-source accessibility testing engine by Deque Systems. It's designed for zero false positives and tests against WCAG 2.0, 2.1, and 2.2.

## Installation

```bash
npm install axe-core
```

## Browser Usage

```html
<script src="node_modules/axe-core/axe.min.js"></script>
<script>
  axe.run().then(results => {
    console.log(results.violations);
  });
</script>
```

## Node.js Usage

```javascript
const axe = require('axe-core');
const { JSDOM } = require('jsdom');

const dom = new JSDOM('<!DOCTYPE html><html><body><img src="test.png"></body></html>');
const document = dom.window.document;

axe.run(document).then(results => {
  console.log(results.violations);
});
```

## Testing Frameworks

### Playwright

```bash
npm install @axe-core/playwright
```

```javascript
import { test, expect } from '@playwright/test';
import AxeBuilder from '@axe-core/playwright';

test('page is accessible', async ({ page }) => {
  await page.goto('https://example.com');

  const results = await new AxeBuilder({ page }).analyze();

  expect(results.violations).toEqual([]);
});

// With specific tags
test('WCAG 2.2 AA compliance', async ({ page }) => {
  await page.goto('/');

  const results = await new AxeBuilder({ page })
    .withTags(['wcag2a', 'wcag2aa', 'wcag22aa'])
    .analyze();

  expect(results.violations).toEqual([]);
});

// Exclude elements
test('accessible except third-party widget', async ({ page }) => {
  await page.goto('/');

  const results = await new AxeBuilder({ page })
    .exclude('#third-party-widget')
    .analyze();

  expect(results.violations).toEqual([]);
});
```

### Jest / Vitest

```bash
npm install jest-axe
```

```javascript
import { axe, toHaveNoViolations } from 'jest-axe';
import { render } from '@testing-library/react';

expect.extend(toHaveNoViolations);

test('Button is accessible', async () => {
  const { container } = render(<Button>Click me</Button>);
  const results = await axe(container);
  expect(results).toHaveNoViolations();
});

// With options
test('Form is accessible', async () => {
  const { container } = render(<ContactForm />);
  const results = await axe(container, {
    rules: {
      'color-contrast': { enabled: false }
    }
  });
  expect(results).toHaveNoViolations();
});
```

### Cypress

```bash
npm install cypress-axe
```

```javascript
// cypress/support/e2e.js
import 'cypress-axe';

// In test file
describe('Accessibility', () => {
  beforeEach(() => {
    cy.visit('/');
    cy.injectAxe();
  });

  it('has no accessibility violations', () => {
    cy.checkA11y();
  });

  it('main content is accessible', () => {
    cy.checkA11y('#main-content');
  });

  it('passes WCAG 2.2 AA', () => {
    cy.checkA11y(null, {
      runOnly: {
        type: 'tag',
        values: ['wcag2a', 'wcag2aa', 'wcag22aa']
      }
    });
  });
});
```

## Configuration Options

### Tags

| Tag | Description |
|-----|-------------|
| `wcag2a` | WCAG 2.0 Level A |
| `wcag2aa` | WCAG 2.0 Level AA |
| `wcag2aaa` | WCAG 2.0 Level AAA |
| `wcag21a` | WCAG 2.1 Level A |
| `wcag21aa` | WCAG 2.1 Level AA |
| `wcag22aa` | WCAG 2.2 Level AA |
| `best-practice` | Best practices |
| `experimental` | Experimental rules |

### Run Options

```javascript
axe.run(document, {
  // Specific rules
  runOnly: {
    type: 'rule',
    values: ['color-contrast', 'image-alt']
  },

  // By tags
  runOnly: {
    type: 'tag',
    values: ['wcag2aa', 'best-practice']
  },

  // Disable specific rules
  rules: {
    'color-contrast': { enabled: false },
    'region': { enabled: false }
  },

  // Include/exclude elements
  include: [['#main-content']],
  exclude: [['.third-party'], ['#ads']],

  // Result types to return
  resultTypes: ['violations', 'incomplete']
}).then(results => {
  console.log(results);
});
```

## Results Structure

```javascript
{
  violations: [
    {
      id: 'image-alt',
      impact: 'critical', // critical, serious, moderate, minor
      tags: ['wcag2a', 'wcag111'],
      description: 'Images must have alternate text',
      help: 'Images must have alternate text',
      helpUrl: 'https://dequeuniversity.com/rules/axe/4.8/image-alt',
      nodes: [
        {
          html: '<img src="photo.jpg">',
          target: ['img'],
          failureSummary: 'Fix any of the following...'
        }
      ]
    }
  ],
  passes: [...],
  incomplete: [...],
  inapplicable: [...]
}
```

## CI Integration

### GitHub Actions

```yaml
- name: Run accessibility tests
  run: npx playwright test --grep accessibility

- name: Upload results
  if: failure()
  uses: actions/upload-artifact@v4
  with:
    name: axe-results
    path: axe-results.json
```

### Reporting

```javascript
// Custom reporter
const results = await axe.run();

if (results.violations.length > 0) {
  console.error('Accessibility violations found:');
  results.violations.forEach(violation => {
    console.error(`\n${violation.id}: ${violation.help}`);
    console.error(`Impact: ${violation.impact}`);
    violation.nodes.forEach(node => {
      console.error(`  - ${node.target}`);
    });
  });
  process.exit(1);
}
```

## Common Rules

| Rule ID | Description | Impact |
|---------|-------------|--------|
| `image-alt` | Images must have alt text | Critical |
| `color-contrast` | Sufficient color contrast | Serious |
| `label` | Form elements need labels | Critical |
| `button-name` | Buttons need accessible names | Critical |
| `link-name` | Links need accessible names | Serious |
| `region` | Content in landmarks | Moderate |
| `heading-order` | Headings in order | Moderate |
| `duplicate-id` | Unique IDs | Minor |
