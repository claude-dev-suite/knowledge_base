# ESLint Rules

> Source: https://eslint.org/docs/latest/rules/

## Rule Severity

| Value | Meaning |
|-------|---------|
| `"off"` or `0` | Disable rule |
| `"warn"` or `1` | Warning (doesn't affect exit code) |
| `"error"` or `2` | Error (exit code 1) |

## Essential Rules

### Possible Problems

```javascript
rules: {
  // Disallow await inside loops (use Promise.all)
  'no-await-in-loop': 'error',

  // Disallow duplicate conditions
  'no-duplicate-case': 'error',

  // Disallow duplicate keys
  'no-dupe-keys': 'error',

  // Disallow unreachable code
  'no-unreachable': 'error',

  // Require === and !==
  'eqeqeq': ['error', 'always'],

  // Disallow unused variables
  'no-unused-vars': ['error', { argsIgnorePattern: '^_' }],
}
```

### Suggestions

```javascript
rules: {
  // Require const for variables never reassigned
  'prefer-const': 'error',

  // Prefer arrow functions for callbacks
  'prefer-arrow-callback': 'error',

  // Prefer template literals
  'prefer-template': 'error',

  // Prefer destructuring
  'prefer-destructuring': ['error', { array: false, object: true }],

  // No var, use let/const
  'no-var': 'error',

  // Object shorthand
  'object-shorthand': 'error',
}
```

### Complexity

```javascript
rules: {
  // Cyclomatic complexity
  'complexity': ['warn', 10],

  // Maximum nesting depth
  'max-depth': ['warn', 4],

  // Maximum lines per function
  'max-lines-per-function': ['warn', 50],

  // Maximum parameters
  'max-params': ['warn', 4],
}
```

## Recommended Configs

### ESLint Recommended

```javascript
import eslint from '@eslint/js';

export default [
  eslint.configs.recommended,
];
```

Includes rules like:
- `no-undef`
- `no-unused-vars`
- `no-unreachable`
- `no-dupe-keys`
- `no-duplicate-case`
- `use-isnan`

## Rule Options

Many rules accept options:

```javascript
rules: {
  // Simple on/off
  'no-console': 'warn',

  // With options array
  'no-unused-vars': ['error', {
    vars: 'all',
    args: 'after-used',
    argsIgnorePattern: '^_',
    ignoreRestSiblings: true
  }],

  // Complexity with threshold
  'complexity': ['error', { max: 10 }],

  // Quotes with options
  'quotes': ['error', 'single', { avoidEscape: true }],
}
```

## Disabling Rules

### Inline Comments

```javascript
// Disable for next line
// eslint-disable-next-line no-console
console.log('debug');

// Disable for specific rules
/* eslint-disable no-console, no-alert */
console.log('allowed');
alert('allowed');
/* eslint-enable no-console, no-alert */

// Disable for entire file (at top)
/* eslint-disable */
```

### In Config

```javascript
export default [
  {
    files: ['**/*.test.js'],
    rules: {
      'no-unused-expressions': 'off'
    }
  }
];
```

## Popular Rule Sets

| Package | Description |
|---------|-------------|
| `@eslint/js` | ESLint's recommended rules |
| `typescript-eslint` | TypeScript-specific rules |
| `eslint-plugin-react` | React-specific rules |
| `eslint-plugin-react-hooks` | React Hooks rules |
| `eslint-config-prettier` | Disables formatting rules |

## Rule Categories

| Category | Purpose |
|----------|---------|
| Possible Problems | Catch likely bugs |
| Suggestions | Better patterns |
| Layout & Formatting | Code style (prefer Prettier) |

## Deprecated Rules

Formatting rules are deprecated in ESLint 9. Use Prettier instead:

- `indent` → Prettier
- `quotes` → Prettier
- `semi` → Prettier
- `comma-dangle` → Prettier
