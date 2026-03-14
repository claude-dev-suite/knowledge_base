# Vitest Test Coverage

Comprehensive guide to configuring and using test coverage in Vitest projects.

## Overview

Test coverage measures how much of your source code is executed during testing. Vitest provides built-in coverage support through two providers: **v8** (native V8 coverage) and **istanbul** (instrumentation-based coverage). Coverage helps identify untested code paths and ensures code quality.

---

## 1. Setup and Installation

### Installing Coverage Dependencies

Vitest supports coverage out of the box, but you need to install the coverage provider package.

#### For v8 Coverage (Recommended for Speed)

```bash
# npm
npm install -D @vitest/coverage-v8

# yarn
yarn add -D @vitest/coverage-v8

# pnpm
pnpm add -D @vitest/coverage-v8

# bun
bun add -D @vitest/coverage-v8
```

#### For Istanbul Coverage (More Features)

```bash
# npm
npm install -D @vitest/coverage-istanbul

# yarn
yarn add -D @vitest/coverage-istanbul

# pnpm
pnpm add -D @vitest/coverage-istanbul

# bun
bun add -D @vitest/coverage-istanbul
```

### Basic Configuration

Add coverage configuration to your `vitest.config.ts`:

```typescript
import { defineConfig } from 'vitest/config'

export default defineConfig({
  test: {
    coverage: {
      enabled: true,
      provider: 'v8', // or 'istanbul'
    },
  },
})
```

### Enabling Coverage via CLI

You can enable coverage without configuration:

```bash
# Enable coverage for a single run
vitest run --coverage

# Enable coverage in watch mode
vitest --coverage

# Specify provider via CLI
vitest run --coverage.provider=istanbul
```

### Package.json Scripts

Add convenient scripts for coverage:

```json
{
  "scripts": {
    "test": "vitest",
    "test:coverage": "vitest run --coverage",
    "test:coverage:watch": "vitest --coverage",
    "test:coverage:ui": "vitest --coverage --ui"
  }
}
```

---

## 2. Coverage Providers (v8, istanbul)

### Understanding Coverage Providers

Vitest supports two coverage providers, each with distinct characteristics:

| Feature | v8 | Istanbul |
|---------|-----|----------|
| Speed | Faster | Slower |
| Accuracy | Good | Excellent |
| Source Maps | Native | Instrumentation |
| Browser Support | Limited | Full |
| Branch Coverage | Basic | Advanced |

### v8 Provider

The v8 provider uses V8's built-in code coverage feature:

```typescript
import { defineConfig } from 'vitest/config'

export default defineConfig({
  test: {
    coverage: {
      provider: 'v8',
    },
  },
})
```

**Advantages of v8:**
- Faster execution (no instrumentation overhead)
- Native to Node.js
- Lower memory usage
- Better for large codebases

**Limitations of v8:**
- May have less accurate branch coverage
- Limited browser environment support
- Some edge cases with source maps

### Istanbul Provider

Istanbul provides instrumentation-based coverage:

```typescript
import { defineConfig } from 'vitest/config'

export default defineConfig({
  test: {
    coverage: {
      provider: 'istanbul',
    },
  },
})
```

**Advantages of Istanbul:**
- More accurate coverage metrics
- Better branch coverage detection
- Full browser environment support
- Extensive configuration options
- Industry standard tooling

**Limitations of Istanbul:**
- Slower due to code instrumentation
- Higher memory usage
- May affect test performance

### Choosing a Provider

```typescript
// For most projects - use v8 for speed
export default defineConfig({
  test: {
    coverage: {
      provider: 'v8',
    },
  },
})

// For projects requiring accurate branch coverage
export default defineConfig({
  test: {
    coverage: {
      provider: 'istanbul',
    },
  },
})

// For browser testing
export default defineConfig({
  test: {
    coverage: {
      provider: 'istanbul', // Better browser support
    },
  },
})
```

---

## 3. Coverage Configuration

### Complete Configuration Reference

```typescript
import { defineConfig } from 'vitest/config'

export default defineConfig({
  test: {
    coverage: {
      // Enable coverage collection
      enabled: true,

      // Coverage provider
      provider: 'v8', // 'v8' | 'istanbul'

      // Output directory for coverage reports
      reportsDirectory: './coverage',

      // Reporter types
      reporter: ['text', 'json', 'html', 'lcov'],

      // Files to include in coverage
      include: ['src/**/*.{js,ts,jsx,tsx}'],

      // Files to exclude from coverage
      exclude: [
        'node_modules/',
        'test/',
        '**/*.d.ts',
        '**/*.test.{js,ts}',
        '**/*.spec.{js,ts}',
        '**/index.{js,ts}',
      ],

      // Coverage thresholds
      thresholds: {
        lines: 80,
        functions: 80,
        branches: 80,
        statements: 80,
      },

      // Report all files, not just tested ones
      all: true,

      // Clean coverage results before running
      clean: true,

      // Skip full coverage report
      skipFull: false,

      // Extension to use for coverage files
      extension: ['.js', '.ts', '.jsx', '.tsx', '.vue', '.svelte'],

      // Enable source maps
      sourcemap: true,

      // Watermarks for coverage percentages
      watermarks: {
        statements: [50, 80],
        functions: [50, 80],
        branches: [50, 80],
        lines: [50, 80],
      },

      // Process sources for coverage
      processingConcurrency: 20,
    },
  },
})
```

### Configuration by Environment

```typescript
import { defineConfig } from 'vitest/config'

export default defineConfig({
  test: {
    coverage: {
      provider: 'v8',
      reporter: process.env.CI ? ['lcov', 'json'] : ['text', 'html'],
      reportsDirectory: process.env.CI ? './coverage-ci' : './coverage',
      thresholds: process.env.CI ? {
        lines: 80,
        functions: 80,
        branches: 70,
        statements: 80,
      } : undefined,
    },
  },
})
```

### Workspace Configuration

For monorepo setups with Vitest workspaces:

```typescript
// vitest.workspace.ts
import { defineWorkspace } from 'vitest/config'

export default defineWorkspace([
  {
    extends: './vitest.config.ts',
    test: {
      name: 'unit',
      include: ['packages/*/src/**/*.test.ts'],
      coverage: {
        include: ['packages/*/src/**/*.ts'],
        exclude: ['packages/*/src/**/*.test.ts'],
      },
    },
  },
  {
    extends: './vitest.config.ts',
    test: {
      name: 'integration',
      include: ['tests/integration/**/*.test.ts'],
      coverage: {
        include: ['src/**/*.ts'],
      },
    },
  },
])
```

---

## 4. Running Coverage

### Basic Coverage Commands

```bash
# Run tests with coverage once
vitest run --coverage

# Run tests with coverage in watch mode
vitest --coverage

# Run specific test files with coverage
vitest run --coverage src/utils.test.ts

# Run tests matching a pattern with coverage
vitest run --coverage --testNamePattern="should validate"
```

### CLI Coverage Options

```bash
# Specify coverage provider
vitest run --coverage --coverage.provider=istanbul

# Specify reporters
vitest run --coverage --coverage.reporter=text --coverage.reporter=html

# Set custom output directory
vitest run --coverage --coverage.reportsDirectory=./my-coverage

# Include all files (not just tested)
vitest run --coverage --coverage.all

# Set thresholds via CLI
vitest run --coverage --coverage.thresholds.lines=80

# Skip full coverage files in report
vitest run --coverage --coverage.skipFull
```

### Running Coverage for Specific Directories

```bash
# Coverage for specific source directory
vitest run --coverage --coverage.include=src/utils/**

# Coverage excluding certain paths
vitest run --coverage --coverage.exclude=src/legacy/**

# Multiple includes
vitest run --coverage \
  --coverage.include=src/components/** \
  --coverage.include=src/hooks/**
```

### Coverage with Different Reporters

```bash
# Text report only (console output)
vitest run --coverage --coverage.reporter=text

# HTML report for visual inspection
vitest run --coverage --coverage.reporter=html

# Multiple reporters
vitest run --coverage \
  --coverage.reporter=text \
  --coverage.reporter=html \
  --coverage.reporter=lcov

# JSON for programmatic access
vitest run --coverage --coverage.reporter=json
```

### Parallel Coverage Collection

```bash
# Run with multiple threads (default)
vitest run --coverage

# Single-threaded for debugging coverage issues
vitest run --coverage --pool=forks --poolOptions.forks.singleFork

# Limit concurrency
vitest run --coverage --maxConcurrency=4
```

---

## 5. Coverage Reporters (text, html, lcov, json)

### Text Reporter

Console-based coverage output:

```typescript
export default defineConfig({
  test: {
    coverage: {
      reporter: ['text'],
    },
  },
})
```

**Output Example:**
```
--------------------|---------|----------|---------|---------|-------------------
File                | % Stmts | % Branch | % Funcs | % Lines | Uncovered Line #s
--------------------|---------|----------|---------|---------|-------------------
All files           |   85.71 |    83.33 |   88.89 |   85.71 |
 src                |   85.71 |    83.33 |   88.89 |   85.71 |
  utils.ts          |   90.00 |    85.00 |   92.00 |   90.00 | 45-48,102
  helpers.ts        |   80.00 |    80.00 |   85.00 |   80.00 | 23-30,67-72
--------------------|---------|----------|---------|---------|-------------------
```

**Text Reporter Variants:**

```typescript
export default defineConfig({
  test: {
    coverage: {
      reporter: [
        'text',           // Standard table output
        'text-summary',   // Summary only
        'text-lcov',      // LCOV format text
      ],
    },
  },
})
```

### HTML Reporter

Interactive HTML coverage report:

```typescript
export default defineConfig({
  test: {
    coverage: {
      reporter: ['html'],
      reportsDirectory: './coverage',
    },
  },
})
```

**Features:**
- Interactive file browser
- Line-by-line coverage highlighting
- Branch coverage visualization
- Sortable columns
- Filter and search capabilities

**Customizing HTML Reporter:**

```typescript
export default defineConfig({
  test: {
    coverage: {
      reporter: [
        ['html', {
          subdir: 'html-report',
        }],
      ],
    },
  },
})
```

### LCOV Reporter

Standard format for CI/CD integration:

```typescript
export default defineConfig({
  test: {
    coverage: {
      reporter: ['lcov'],
      reportsDirectory: './coverage',
    },
  },
})
```

**Output:** `coverage/lcov.info`

**LCOV Format Structure:**
```
TN:
SF:/path/to/file.ts
FN:10,functionName
FNDA:5,functionName
FNF:1
FNH:1
DA:10,5
DA:11,5
DA:12,0
LF:3
LH:2
BRF:2
BRH:1
end_of_record
```

**Use Cases:**
- SonarQube integration
- Codecov uploads
- Coveralls integration
- GitHub Actions coverage

### JSON Reporter

Machine-readable coverage data:

```typescript
export default defineConfig({
  test: {
    coverage: {
      reporter: ['json'],
      reportsDirectory: './coverage',
    },
  },
})
```

**Output:** `coverage/coverage-final.json`

**JSON Structure:**
```json
{
  "/path/to/file.ts": {
    "path": "/path/to/file.ts",
    "statementMap": {},
    "fnMap": {},
    "branchMap": {},
    "s": {},
    "f": {},
    "b": {}
  }
}
```

**JSON Summary Reporter:**

```typescript
export default defineConfig({
  test: {
    coverage: {
      reporter: ['json-summary'],
    },
  },
})
```

**Output:** `coverage/coverage-summary.json`

```json
{
  "total": {
    "lines": { "total": 100, "covered": 85, "skipped": 0, "pct": 85 },
    "statements": { "total": 120, "covered": 102, "skipped": 0, "pct": 85 },
    "functions": { "total": 25, "covered": 22, "skipped": 0, "pct": 88 },
    "branches": { "total": 40, "covered": 32, "skipped": 0, "pct": 80 }
  }
}
```

### Combining Multiple Reporters

```typescript
export default defineConfig({
  test: {
    coverage: {
      reporter: [
        'text',           // Console output
        'text-summary',   // Brief summary
        'html',           // Visual report
        'lcov',           // CI integration
        'json',           // Programmatic access
        'json-summary',   // Summary JSON
        'cobertura',      // Jenkins/Azure DevOps
        'clover',         // Atlassian tools
      ],
    },
  },
})
```

### Reporter with Options

```typescript
export default defineConfig({
  test: {
    coverage: {
      reporter: [
        ['text', { file: 'coverage.txt' }],
        ['html', { subdir: 'html-report' }],
        ['lcov', { file: 'lcov.info' }],
        ['json', { file: 'coverage.json' }],
      ],
    },
  },
})
```

---

## 6. Coverage Thresholds

### Basic Threshold Configuration

```typescript
export default defineConfig({
  test: {
    coverage: {
      thresholds: {
        lines: 80,
        functions: 80,
        branches: 75,
        statements: 80,
      },
    },
  },
})
```

### Understanding Threshold Types

| Threshold | Description |
|-----------|-------------|
| `lines` | Percentage of executed lines |
| `functions` | Percentage of called functions |
| `branches` | Percentage of taken branches |
| `statements` | Percentage of executed statements |

### Global vs Per-File Thresholds

```typescript
export default defineConfig({
  test: {
    coverage: {
      thresholds: {
        // Global thresholds
        lines: 80,
        functions: 80,
        branches: 75,
        statements: 80,

        // Per-file thresholds
        perFile: true, // Enforce thresholds per file
      },
    },
  },
})
```

### Custom Thresholds by Glob Pattern

```typescript
export default defineConfig({
  test: {
    coverage: {
      thresholds: {
        lines: 80,
        functions: 80,
        branches: 75,
        statements: 80,

        // Override for specific paths
        'src/utils/**': {
          lines: 90,
          functions: 90,
          branches: 85,
          statements: 90,
        },
        'src/legacy/**': {
          lines: 50,
          functions: 50,
          branches: 40,
          statements: 50,
        },
      },
    },
  },
})
```

### Threshold Failure Behavior

```typescript
export default defineConfig({
  test: {
    coverage: {
      thresholds: {
        lines: 80,
        functions: 80,
        branches: 75,
        statements: 80,

        // Auto-update thresholds in config (useful during development)
        autoUpdate: false,
      },
    },
  },
})
```

### 100% Coverage Enforcement

```typescript
export default defineConfig({
  test: {
    coverage: {
      thresholds: {
        '100': true, // Require 100% coverage for all metrics
      },
    },
  },
})
```

### Threshold with Tolerance

```typescript
// Custom script to handle threshold tolerance
// vitest.config.ts
export default defineConfig({
  test: {
    coverage: {
      thresholds: {
        lines: 78,      // Allow 2% tolerance from 80% target
        functions: 78,
        branches: 73,
        statements: 78,
      },
    },
  },
})
```

### CI/CD Threshold Strategy

```typescript
const isCI = process.env.CI === 'true'

export default defineConfig({
  test: {
    coverage: {
      thresholds: isCI ? {
        lines: 80,
        functions: 80,
        branches: 75,
        statements: 80,
        perFile: true,
      } : undefined, // No thresholds locally
    },
  },
})
```

---

## 7. Including/Excluding Files

### Include Patterns

```typescript
export default defineConfig({
  test: {
    coverage: {
      include: [
        'src/**/*.ts',
        'src/**/*.tsx',
        'src/**/*.js',
        'src/**/*.jsx',
        'lib/**/*.ts',
      ],
    },
  },
})
```

### Exclude Patterns

```typescript
export default defineConfig({
  test: {
    coverage: {
      exclude: [
        // Test files
        '**/*.test.ts',
        '**/*.spec.ts',
        '**/__tests__/**',
        '**/tests/**',

        // Type definitions
        '**/*.d.ts',

        // Configuration files
        '**/vitest.config.ts',
        '**/vite.config.ts',
        '**/jest.config.*',

        // Build outputs
        '**/dist/**',
        '**/build/**',
        '**/coverage/**',

        // Node modules
        '**/node_modules/**',

        // Storybook
        '**/*.stories.{ts,tsx,js,jsx}',

        // Mock files
        '**/__mocks__/**',
        '**/mocks/**',

        // Index files (re-exports)
        '**/index.{ts,js}',

        // Generated files
        '**/generated/**',
        '**/*.generated.ts',

        // Scripts
        '**/scripts/**',
      ],
    },
  },
})
```

### Default Excludes

Vitest automatically excludes these patterns:

```typescript
const defaultExcludes = [
  'coverage/**',
  'dist/**',
  '**/[.]**',
  'packages/*/test?(s)/**',
  '**/*.d.ts',
  '**/virtual:*',
  '**/__x00__*',
  '**/\x00*',
  'cypress/**',
  'test?(s)/**',
  'test?(-*).?(c|m)[jt]s?(x)',
  '**/*{.,-}{test,spec}.?(c|m)[jt]s?(x)',
  '**/__tests__/**',
  '**/{karma,rollup,webpack,vite,vitest,jest,ava,babel,nyc,cypress,tsup,build}.config.*',
  '**/vitest.{workspace,projects}.[jt]s?(on)',
  '**/.{eslint,mocha,prettier}rc.{?(c|m)js,yml}',
]
```

### Including All Files

```typescript
export default defineConfig({
  test: {
    coverage: {
      all: true, // Include files not imported by tests
      include: ['src/**/*.ts'],
      exclude: ['**/*.test.ts', '**/*.d.ts'],
    },
  },
})
```

### Extension Filtering

```typescript
export default defineConfig({
  test: {
    coverage: {
      extension: ['.ts', '.tsx', '.js', '.jsx', '.vue', '.svelte'],
    },
  },
})
```

### Complex Include/Exclude Example

```typescript
export default defineConfig({
  test: {
    coverage: {
      provider: 'v8',
      all: true,

      include: [
        'src/**/*.{ts,tsx}',
        'lib/**/*.{ts,tsx}',
        'packages/*/src/**/*.{ts,tsx}',
      ],

      exclude: [
        // Test-related
        '**/*.test.{ts,tsx}',
        '**/*.spec.{ts,tsx}',
        '**/__tests__/**',
        '**/__fixtures__/**',
        '**/__mocks__/**',
        '**/test-utils/**',

        // Types
        '**/*.d.ts',
        '**/types/**',

        // Configuration
        '**/config/**',
        '**/*.config.*',

        // Generated
        '**/generated/**',
        '**/*.generated.*',

        // Legacy code with low coverage expectations
        '**/legacy/**',
        '**/deprecated/**',

        // Entry points
        '**/main.{ts,tsx}',
        '**/index.{ts,tsx}',

        // Third-party integrations
        '**/vendor/**',
      ],
    },
  },
})
```

---

## 8. Coverage for Specific Files

### Running Coverage for Single File

```bash
# Coverage for specific source file
vitest run --coverage --coverage.include=src/utils/validator.ts

# Coverage for tests of specific file
vitest run --coverage src/utils/validator.test.ts
```

### File-Specific Configuration

```typescript
export default defineConfig({
  test: {
    coverage: {
      // Different thresholds for different files
      thresholds: {
        // Global defaults
        lines: 80,

        // Critical files need higher coverage
        'src/auth/**/*.ts': {
          lines: 95,
          functions: 95,
          branches: 90,
        },

        // Utils can have slightly lower
        'src/utils/**/*.ts': {
          lines: 85,
          functions: 85,
          branches: 75,
        },

        // Legacy code has lower expectations
        'src/legacy/**/*.ts': {
          lines: 50,
          functions: 50,
          branches: 40,
        },
      },
    },
  },
})
```

### Ignoring Lines in Source Code

```typescript
// Ignore next line
/* v8 ignore next */
const debugCode = process.env.DEBUG ? console.log : () => {}

// Ignore next N lines
/* v8 ignore next 3 */
if (process.env.NODE_ENV === 'development') {
  console.log('Debug mode')
  enableDevTools()
}

// Ignore specific block
/* v8 ignore start */
function legacyFunction() {
  // This entire function is ignored
  return oldBehavior()
}
/* v8 ignore stop */
```

### Istanbul Ignore Comments

```typescript
// Ignore next line
/* istanbul ignore next */
const fallback = process.env.FALLBACK || 'default'

// Ignore if branch
/* istanbul ignore if */
if (process.env.SKIP_VALIDATION) {
  return true
}

// Ignore else branch
if (condition) {
  doSomething()
} /* istanbul ignore else */ else {
  handleEdgeCase()
}

// Ignore entire file
/* istanbul ignore file */
```

### Coverage Annotations

```typescript
// Mark code as covered even if not executed
// @ts-expect-error - coverage annotation
/* c8 ignore next */
export function emergencyHandler() {
  // Emergency code path
}
```

---

## 9. Branch Coverage

### Understanding Branch Coverage

Branch coverage measures whether both true and false paths of conditional statements are tested.

```typescript
// This function has 2 branches
function isAdult(age: number): boolean {
  if (age >= 18) {  // Branch 1: true
    return true
  }
  return false      // Branch 2: false (else)
}

// Tests needed for 100% branch coverage:
test('returns true for adults', () => {
  expect(isAdult(18)).toBe(true)  // Covers branch 1
})

test('returns false for minors', () => {
  expect(isAdult(17)).toBe(false) // Covers branch 2
})
```

### Complex Branch Scenarios

```typescript
function calculateDiscount(
  price: number,
  isMember: boolean,
  couponCode?: string
): number {
  let discount = 0

  // Branch 1: membership check
  if (isMember) {
    discount += 10
  }

  // Branch 2: coupon check
  if (couponCode) {
    // Branch 3: coupon type
    if (couponCode.startsWith('SAVE')) {
      discount += 15
    } else if (couponCode.startsWith('VIP')) {
      discount += 25
    }
  }

  // Branch 4: max discount
  return Math.min(discount, 30)
}

// Tests for full branch coverage
describe('calculateDiscount', () => {
  test('no discount for non-member without coupon', () => {
    expect(calculateDiscount(100, false)).toBe(0)
  })

  test('member discount only', () => {
    expect(calculateDiscount(100, true)).toBe(10)
  })

  test('SAVE coupon for non-member', () => {
    expect(calculateDiscount(100, false, 'SAVE10')).toBe(15)
  })

  test('VIP coupon for non-member', () => {
    expect(calculateDiscount(100, false, 'VIP2023')).toBe(25)
  })

  test('member with SAVE coupon hits max', () => {
    expect(calculateDiscount(100, true, 'SAVE10')).toBe(25)
  })

  test('member with VIP coupon hits max', () => {
    expect(calculateDiscount(100, true, 'VIP2023')).toBe(30)
  })
})
```

### Ternary Operator Branches

```typescript
// Ternary operators create branches too
const status = isActive ? 'active' : 'inactive'

// Short-circuit evaluations
const name = user?.name || 'Anonymous'
const result = condition && performAction()
```

### Branch Coverage Thresholds

```typescript
export default defineConfig({
  test: {
    coverage: {
      thresholds: {
        branches: 80, // Require 80% branch coverage
      },
    },
  },
})
```

### Improving Branch Coverage

```typescript
// Before: Single test
test('validates email', () => {
  expect(validateEmail('test@example.com')).toBe(true)
})

// After: Full branch coverage
describe('validateEmail', () => {
  test('returns true for valid email', () => {
    expect(validateEmail('test@example.com')).toBe(true)
  })

  test('returns false for email without @', () => {
    expect(validateEmail('testexample.com')).toBe(false)
  })

  test('returns false for email without domain', () => {
    expect(validateEmail('test@')).toBe(false)
  })

  test('returns false for empty string', () => {
    expect(validateEmail('')).toBe(false)
  })

  test('returns false for null input', () => {
    expect(validateEmail(null)).toBe(false)
  })
})
```

---

## 10. Function Coverage

### Understanding Function Coverage

Function coverage measures whether functions have been called at least once during testing.

```typescript
// Module with multiple functions
export function add(a: number, b: number): number {
  return a + b
}

export function subtract(a: number, b: number): number {
  return a - b
}

export function multiply(a: number, b: number): number {
  return a * b
}

export function divide(a: number, b: number): number {
  if (b === 0) throw new Error('Division by zero')
  return a / b
}

// 50% function coverage - only 2 of 4 functions tested
test('add works', () => {
  expect(add(2, 3)).toBe(5)
})

test('subtract works', () => {
  expect(subtract(5, 3)).toBe(2)
})
```

### Function Coverage Thresholds

```typescript
export default defineConfig({
  test: {
    coverage: {
      thresholds: {
        functions: 90, // Require 90% function coverage
      },
    },
  },
})
```

### Anonymous Functions

```typescript
// Named functions are easier to track
export const processItems = (items: Item[]) => {
  return items
    .filter((item) => item.active)     // Anonymous function 1
    .map((item) => item.value)         // Anonymous function 2
    .reduce((sum, val) => sum + val, 0) // Anonymous function 3
}

// Better for coverage tracking - named functions
const isActive = (item: Item) => item.active
const getValue = (item: Item) => item.value
const sumValues = (sum: number, val: number) => sum + val

export const processItemsNamed = (items: Item[]) => {
  return items.filter(isActive).map(getValue).reduce(sumValues, 0)
}
```

### Class Method Coverage

```typescript
class Calculator {
  add(a: number, b: number) { return a + b }
  subtract(a: number, b: number) { return a - b }
  multiply(a: number, b: number) { return a * b }
  divide(a: number, b: number) {
    if (b === 0) throw new Error('Division by zero')
    return a / b
  }
}

// Tests for all methods
describe('Calculator', () => {
  const calc = new Calculator()

  test('add', () => expect(calc.add(2, 3)).toBe(5))
  test('subtract', () => expect(calc.subtract(5, 3)).toBe(2))
  test('multiply', () => expect(calc.multiply(4, 3)).toBe(12))
  test('divide', () => expect(calc.divide(10, 2)).toBe(5))
  test('divide by zero throws', () => {
    expect(() => calc.divide(10, 0)).toThrow('Division by zero')
  })
})
```

---

## 11. Line Coverage

### Understanding Line Coverage

Line coverage measures the percentage of executable lines that were run during tests.

```typescript
function processUser(user: User): ProcessedUser {
  const name = user.name.trim()           // Line 1
  const email = user.email.toLowerCase()   // Line 2

  if (!email.includes('@')) {              // Line 3 (branch)
    throw new Error('Invalid email')       // Line 4
  }

  const domain = email.split('@')[1]       // Line 5

  return {                                 // Line 6
    name,                                  // Line 7
    email,                                 // Line 8
    domain,                                // Line 9
  }                                        // Line 10
}
```

### Line Coverage Thresholds

```typescript
export default defineConfig({
  test: {
    coverage: {
      thresholds: {
        lines: 85, // Require 85% line coverage
      },
    },
  },
})
```

### Improving Line Coverage

```typescript
// Identify uncovered lines in HTML report
// Then write tests targeting those specific lines

// Example: Error handling lines often uncovered
function fetchData(url: string) {
  try {
    const response = await fetch(url)

    if (!response.ok) {           // Line often uncovered
      throw new Error('Failed')   // Line often uncovered
    }

    return response.json()
  } catch (error) {
    console.error(error)          // Line often uncovered
    throw error                   // Line often uncovered
  }
}

// Tests to cover error paths
test('throws on non-ok response', async () => {
  global.fetch = vi.fn().mockResolvedValue({
    ok: false,
    status: 404,
  })

  await expect(fetchData('/api')).rejects.toThrow('Failed')
})

test('handles network error', async () => {
  global.fetch = vi.fn().mockRejectedValue(new Error('Network error'))

  await expect(fetchData('/api')).rejects.toThrow('Network error')
})
```

### Multi-Line Statements

```typescript
// Single statement across multiple lines counts as one
const result = someFunction(
  argument1,
  argument2,
  argument3
)

// Chained methods - each chain may count separately
const processed = data
  .filter(item => item.active)  // Line 1
  .map(item => item.value)      // Line 2
  .sort((a, b) => a - b)        // Line 3
```

---

## 12. Statement Coverage

### Understanding Statement Coverage

Statement coverage measures executed statements, which can differ from line coverage.

```typescript
// Multiple statements on one line
let a = 1; let b = 2; let c = 3;  // 3 statements, 1 line

// One statement across multiple lines
const config = {                   // 1 statement
  name: 'app',                     // continued
  version: '1.0',                  // continued
}                                  // continued
```

### Statement vs Line Coverage

```typescript
// Line coverage: 2 lines
// Statement coverage: 4 statements
const x = 1;                    // Statement 1
const y = 2;                    // Statement 2
const z = x + y;                // Statement 3
console.log(z);                 // Statement 4
```

### Statement Coverage Thresholds

```typescript
export default defineConfig({
  test: {
    coverage: {
      thresholds: {
        statements: 80, // Require 80% statement coverage
      },
    },
  },
})
```

### Complex Statement Patterns

```typescript
// For loops contain multiple statements
for (let i = 0; i < 10; i++) {
  // let i = 0 - initialization statement
  // i < 10 - condition (evaluated each iteration)
  // i++ - increment statement
  doSomething(i)
}

// Switch statements
switch (status) {
  case 'active':      // Statement
    activate()        // Statement
    break             // Statement
  case 'inactive':    // Statement
    deactivate()      // Statement
    break             // Statement
  default:            // Statement
    reset()           // Statement
}
```

---

## 13. Coverage in CI/CD

### GitHub Actions Configuration

```yaml
# .github/workflows/test.yml
name: Test Coverage

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

jobs:
  coverage:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'

      - name: Install dependencies
        run: npm ci

      - name: Run tests with coverage
        run: npm run test:coverage

      - name: Upload coverage to Codecov
        uses: codecov/codecov-action@v4
        with:
          token: ${{ secrets.CODECOV_TOKEN }}
          files: ./coverage/lcov.info
          fail_ci_if_error: true

      - name: Upload coverage artifact
        uses: actions/upload-artifact@v4
        with:
          name: coverage-report
          path: coverage/
          retention-days: 30
```

### GitLab CI Configuration

```yaml
# .gitlab-ci.yml
stages:
  - test

test:coverage:
  stage: test
  image: node:20
  script:
    - npm ci
    - npm run test:coverage
  coverage: '/All files[^|]*\|[^|]*\s+([\d\.]+)/'
  artifacts:
    when: always
    reports:
      coverage_report:
        coverage_format: cobertura
        path: coverage/cobertura-coverage.xml
    paths:
      - coverage/
    expire_in: 1 week
```

### Azure DevOps Configuration

```yaml
# azure-pipelines.yml
trigger:
  branches:
    include:
      - main
      - develop

pool:
  vmImage: 'ubuntu-latest'

steps:
  - task: NodeTool@0
    inputs:
      versionSpec: '20.x'

  - script: npm ci
    displayName: 'Install dependencies'

  - script: npm run test:coverage
    displayName: 'Run tests with coverage'

  - task: PublishCodeCoverageResults@1
    inputs:
      codeCoverageTool: 'Cobertura'
      summaryFileLocation: '$(System.DefaultWorkingDirectory)/coverage/cobertura-coverage.xml'
      reportDirectory: '$(System.DefaultWorkingDirectory)/coverage'
```

### CircleCI Configuration

```yaml
# .circleci/config.yml
version: 2.1

jobs:
  test:
    docker:
      - image: cimg/node:20.0
    steps:
      - checkout
      - restore_cache:
          keys:
            - v1-deps-{{ checksum "package-lock.json" }}
      - run: npm ci
      - save_cache:
          key: v1-deps-{{ checksum "package-lock.json" }}
          paths:
            - node_modules
      - run: npm run test:coverage
      - store_artifacts:
          path: coverage
      - store_test_results:
          path: coverage

workflows:
  test:
    jobs:
      - test
```

### Vitest CI Configuration

```typescript
// vitest.config.ts
export default defineConfig({
  test: {
    coverage: {
      provider: 'v8',
      reporter: process.env.CI
        ? ['text', 'lcov', 'cobertura', 'json-summary']
        : ['text', 'html'],
      reportsDirectory: './coverage',
      thresholds: process.env.CI ? {
        lines: 80,
        functions: 80,
        branches: 75,
        statements: 80,
      } : undefined,
    },
  },
})
```

### Coverage Enforcement in CI

```bash
# package.json scripts
{
  "scripts": {
    "test:ci": "vitest run --coverage --coverage.thresholds.lines=80"
  }
}
```

---

## 14. Coverage Badges

### Codecov Badge

```markdown
[![codecov](https://codecov.io/gh/username/repo/branch/main/graph/badge.svg)](https://codecov.io/gh/username/repo)
```

### Coveralls Badge

```markdown
[![Coverage Status](https://coveralls.io/repos/github/username/repo/badge.svg?branch=main)](https://coveralls.io/github/username/repo?branch=main)
```

### Custom Badge with Shields.io

```markdown
![Coverage](https://img.shields.io/badge/coverage-85%25-green)
```

### Dynamic Badge from JSON Summary

```yaml
# GitHub Action to update badge
- name: Generate Coverage Badge
  uses: jaywcjlove/coverage-badges-cli@main
  with:
    source: coverage/coverage-summary.json
    output: coverage/badge.svg
```

### Badge Configuration Script

```javascript
// scripts/generate-badge.js
import { readFileSync, writeFileSync } from 'fs'

const summary = JSON.parse(
  readFileSync('./coverage/coverage-summary.json', 'utf-8')
)

const coverage = summary.total.lines.pct
let color = 'red'

if (coverage >= 80) color = 'green'
else if (coverage >= 60) color = 'yellow'
else if (coverage >= 40) color = 'orange'

const badge = `https://img.shields.io/badge/coverage-${coverage}%25-${color}`
console.log(`Badge URL: ${badge}`)
```

### SonarQube Badge

```markdown
[![Quality Gate Status](https://sonarcloud.io/api/project_badges/measure?project=your-project&metric=coverage)](https://sonarcloud.io/dashboard?id=your-project)
```

---

## 15. Istanbul-specific Options

### Istanbul Configuration

```typescript
export default defineConfig({
  test: {
    coverage: {
      provider: 'istanbul',

      // Istanbul-specific options
      ignoreClassMethods: ['render', 'componentDidMount'],

      // Include empty files in report
      all: true,

      // Watermarks for color coding
      watermarks: {
        statements: [50, 80],
        functions: [50, 80],
        branches: [50, 80],
        lines: [50, 80],
      },
    },
  },
})
```

### Istanbul Instrumentation Options

```typescript
export default defineConfig({
  test: {
    coverage: {
      provider: 'istanbul',

      // Custom instrumenter options
      instrumenterOptions: {
        istanbul: {
          // Preserve comments in instrumented code
          preserveComments: true,

          // Compact output
          compact: true,

          // ES modules support
          esModules: true,

          // Auto-wrap in IIFE
          autoWrap: true,
        },
      },
    },
  },
})
```

### Istanbul Ignore Patterns

```typescript
// Ignore entire file
/* istanbul ignore file */

// Ignore next statement
/* istanbul ignore next */
const debugOnlyCode = process.env.DEBUG && console.log('debug')

// Ignore if branch
/* istanbul ignore if */
if (process.env.NODE_ENV === 'test') {
  mockDatabase()
}

// Ignore else branch
if (isValid) {
  process()
} /* istanbul ignore else */ else {
  handleInvalid()
}

// Ignore specific function
/* istanbul ignore next */
function legacyHandler() {
  // Legacy code
}
```

### Istanbul with Source Maps

```typescript
export default defineConfig({
  test: {
    coverage: {
      provider: 'istanbul',
      sourcemap: true,
    },
  },
  build: {
    sourcemap: true,
  },
})
```

### Istanbul Reporters

```typescript
export default defineConfig({
  test: {
    coverage: {
      provider: 'istanbul',
      reporter: [
        'text',
        'text-summary',
        'html',
        'lcov',
        'json',
        'json-summary',
        'cobertura',   // Jenkins integration
        'clover',      // Atlassian tools
        'teamcity',    // TeamCity integration
      ],
    },
  },
})
```

---

## 16. V8-specific Options

### V8 Configuration

```typescript
export default defineConfig({
  test: {
    coverage: {
      provider: 'v8',

      // V8-specific options
      // Source map support
      sourcemap: true,
    },
  },
})
```

### V8 Ignore Comments

```typescript
// Ignore next line
/* v8 ignore next */
const platformSpecific = process.platform === 'win32' ? winPath : unixPath

// Ignore next N lines
/* v8 ignore next 3 */
if (process.env.EXPERIMENTAL) {
  enableExperimentalFeature()
  logExperimentalUsage()
}

// Ignore block
/* v8 ignore start */
function experimentalFeature() {
  // Not ready for coverage
}
/* v8 ignore stop */
```

### V8 Performance Optimization

```typescript
export default defineConfig({
  test: {
    coverage: {
      provider: 'v8',

      // Process files concurrently
      processingConcurrency: 20,

      // Skip reporting for 100% covered files
      skipFull: true,
    },
  },
})
```

### V8 with TypeScript

```typescript
export default defineConfig({
  test: {
    coverage: {
      provider: 'v8',
      include: ['src/**/*.ts'],
      extension: ['.ts', '.tsx'],
    },
  },
  esbuild: {
    sourcemap: 'both',
  },
})
```

### V8 Reporters

```typescript
export default defineConfig({
  test: {
    coverage: {
      provider: 'v8',
      reporter: [
        'text',
        'html',
        'lcov',
        'json',
        'json-summary',
      ],
    },
  },
})
```

### V8 Coverage Accuracy

```typescript
// V8 may report slightly different branch coverage
// For critical accuracy, use Istanbul

// Example where V8 and Istanbul differ:
const value = condition ? 'a' : 'b'  // V8 may not track both branches accurately

// For better V8 accuracy, use explicit if-else:
let value: string
if (condition) {
  value = 'a'
} else {
  value = 'b'
}
```

---

## 17. Best Practices

### Configuration Best Practices

```typescript
// vitest.config.ts
export default defineConfig({
  test: {
    coverage: {
      // 1. Choose appropriate provider
      provider: 'v8', // Use v8 for speed, istanbul for accuracy

      // 2. Enable all files coverage
      all: true,

      // 3. Set reasonable thresholds
      thresholds: {
        lines: 80,
        functions: 80,
        branches: 75,
        statements: 80,
        perFile: true, // Enforce per-file
      },

      // 4. Configure appropriate reporters
      reporter: process.env.CI
        ? ['text', 'lcov', 'json-summary']
        : ['text', 'html'],

      // 5. Properly include/exclude files
      include: ['src/**/*.{ts,tsx}'],
      exclude: [
        '**/*.test.{ts,tsx}',
        '**/*.d.ts',
        '**/types/**',
        '**/index.{ts,tsx}',
      ],

      // 6. Clean between runs
      clean: true,
    },
  },
})
```

### Testing Strategy for Coverage

```typescript
// 1. Test public APIs, not implementation details
// Good - tests behavior
test('validates user email', () => {
  expect(validateUser({ email: 'test@example.com' })).toBe(true)
})

// Avoid - tests implementation
test('regex matches email pattern', () => {
  expect(EMAIL_REGEX.test('test@example.com')).toBe(true)
})

// 2. Test edge cases and error paths
describe('divide', () => {
  test('divides numbers correctly', () => {
    expect(divide(10, 2)).toBe(5)
  })

  test('handles negative numbers', () => {
    expect(divide(-10, 2)).toBe(-5)
  })

  test('throws on division by zero', () => {
    expect(() => divide(10, 0)).toThrow()
  })
})

// 3. Use parameterized tests for multiple scenarios
test.each([
  [1, 1, 2],
  [2, 3, 5],
  [-1, 1, 0],
  [0, 0, 0],
])('add(%i, %i) = %i', (a, b, expected) => {
  expect(add(a, b)).toBe(expected)
})
```

### Coverage Anti-Patterns to Avoid

```typescript
// AVOID: Testing private methods directly
// Instead, test through public interface

// AVOID: Adding tests just to hit coverage numbers
test('meaningless coverage test', () => {
  const result = someFunction()
  expect(result).toBeDefined() // Doesn't verify behavior
})

// AVOID: Ignoring too much code
/* v8 ignore start */
// Hundreds of lines ignored
/* v8 ignore stop */

// AVOID: Setting unrealistic thresholds
thresholds: {
  lines: 100,      // Too strict for most projects
  branches: 100,   // Often impossible with error handling
}

// AVOID: No thresholds at all
thresholds: undefined // Coverage can regress silently
```

### Incremental Coverage Improvement

```typescript
// Start with achievable thresholds
thresholds: {
  lines: 60,
  functions: 60,
  branches: 50,
  statements: 60,
}

// Gradually increase as coverage improves
// Week 2:
thresholds: {
  lines: 65,
  functions: 65,
  branches: 55,
  statements: 65,
}

// Week 4:
thresholds: {
  lines: 70,
  functions: 70,
  branches: 60,
  statements: 70,
}
```

### Coverage Review Process

```markdown
## Coverage Review Checklist

1. [ ] Overall coverage meets thresholds
2. [ ] Critical paths have high coverage (auth, payments, etc.)
3. [ ] Error handling paths are tested
4. [ ] No meaningful code is ignored without justification
5. [ ] New code maintains or improves coverage
6. [ ] Coverage report is generated in CI
7. [ ] Coverage trends are monitored over time
```

### Organizing Test Files

```
src/
├── components/
│   ├── Button/
│   │   ├── Button.tsx
│   │   ├── Button.test.tsx    # Co-located test
│   │   └── Button.styles.ts
│   └── Form/
│       ├── Form.tsx
│       └── Form.test.tsx
├── utils/
│   ├── validators.ts
│   └── validators.test.ts
└── hooks/
    ├── useAuth.ts
    └── useAuth.test.ts
```

### Coverage Monitoring Dashboard

```typescript
// scripts/coverage-report.ts
import { readFileSync } from 'fs'

interface CoverageSummary {
  total: {
    lines: { pct: number }
    statements: { pct: number }
    functions: { pct: number }
    branches: { pct: number }
  }
}

const summary: CoverageSummary = JSON.parse(
  readFileSync('./coverage/coverage-summary.json', 'utf-8')
)

console.log('Coverage Report')
console.log('================')
console.log(`Lines:      ${summary.total.lines.pct.toFixed(2)}%`)
console.log(`Statements: ${summary.total.statements.pct.toFixed(2)}%`)
console.log(`Functions:  ${summary.total.functions.pct.toFixed(2)}%`)
console.log(`Branches:   ${summary.total.branches.pct.toFixed(2)}%`)

// Check thresholds
const thresholds = { lines: 80, statements: 80, functions: 80, branches: 75 }

let failed = false
for (const [metric, threshold] of Object.entries(thresholds)) {
  const actual = summary.total[metric as keyof typeof summary.total].pct
  if (actual < threshold) {
    console.error(`FAIL: ${metric} (${actual}%) below threshold (${threshold}%)`)
    failed = true
  }
}

process.exit(failed ? 1 : 0)
```

### Integration with Code Quality Tools

```typescript
// vitest.config.ts with SonarQube
export default defineConfig({
  test: {
    coverage: {
      provider: 'v8',
      reporter: ['lcov', 'text'],
      reportsDirectory: './coverage',
    },
  },
})

// sonar-project.properties
// sonar.javascript.lcov.reportPaths=coverage/lcov.info
```

### Coverage for Monorepos

```typescript
// vitest.workspace.ts
export default defineWorkspace([
  {
    test: {
      name: 'packages/core',
      include: ['packages/core/**/*.test.ts'],
      coverage: {
        include: ['packages/core/src/**/*.ts'],
        thresholds: { lines: 90 },
      },
    },
  },
  {
    test: {
      name: 'packages/ui',
      include: ['packages/ui/**/*.test.ts'],
      coverage: {
        include: ['packages/ui/src/**/*.ts'],
        thresholds: { lines: 80 },
      },
    },
  },
  {
    test: {
      name: 'packages/utils',
      include: ['packages/utils/**/*.test.ts'],
      coverage: {
        include: ['packages/utils/src/**/*.ts'],
        thresholds: { lines: 95 },
      },
    },
  },
])
```

---

## Quick Reference

### Essential Commands

```bash
# Run tests with coverage
vitest run --coverage

# Watch mode with coverage
vitest --coverage

# Coverage with specific reporter
vitest run --coverage --coverage.reporter=html

# Coverage with thresholds
vitest run --coverage --coverage.thresholds.lines=80

# Coverage for specific files
vitest run --coverage --coverage.include=src/utils/**
```

### Configuration Template

```typescript
import { defineConfig } from 'vitest/config'

export default defineConfig({
  test: {
    coverage: {
      enabled: true,
      provider: 'v8',
      reporter: ['text', 'html', 'lcov'],
      reportsDirectory: './coverage',
      include: ['src/**/*.{ts,tsx}'],
      exclude: ['**/*.test.{ts,tsx}', '**/*.d.ts'],
      all: true,
      clean: true,
      thresholds: {
        lines: 80,
        functions: 80,
        branches: 75,
        statements: 80,
      },
    },
  },
})
```

### Ignore Comments

```typescript
// V8 provider
/* v8 ignore next */
/* v8 ignore next 3 */
/* v8 ignore start */ ... /* v8 ignore stop */

// Istanbul provider
/* istanbul ignore next */
/* istanbul ignore if */
/* istanbul ignore else */
/* istanbul ignore file */
```

### CI/CD Integration

```yaml
# GitHub Actions
- run: npm run test:coverage
- uses: codecov/codecov-action@v4
  with:
    files: ./coverage/lcov.info
```

---

## Troubleshooting

### Common Issues

**Coverage shows 0% for all files:**
```typescript
// Ensure files are properly included
include: ['src/**/*.ts'], // Check glob patterns match your structure
all: true, // Include untested files
```

**Source maps not working:**
```typescript
// Enable source maps
export default defineConfig({
  test: {
    coverage: {
      sourcemap: true,
    },
  },
  build: {
    sourcemap: true,
  },
})
```

**Thresholds not enforced:**
```bash
# Ensure thresholds are set and coverage is enabled
vitest run --coverage --coverage.thresholds.lines=80
```

**Coverage differs between providers:**
```typescript
// v8 and istanbul may report different numbers
// For consistent CI results, stick with one provider
provider: 'istanbul', // More consistent, slower
provider: 'v8',       // Faster, may vary slightly
```

**Files not appearing in report:**
```typescript
// Check include/exclude patterns
include: ['src/**/*.{ts,tsx,js,jsx}'],
exclude: [
  // Make sure important files aren't excluded
  '**/node_modules/**',
  '**/*.test.*',
],
```

---

## Additional Resources

- [Vitest Coverage Documentation](https://vitest.dev/guide/coverage.html)
- [Istanbul.js Documentation](https://istanbul.js.org/)
- [V8 Code Coverage](https://v8.dev/blog/javascript-code-coverage)
- [Codecov Integration](https://docs.codecov.com/docs)
- [SonarQube JavaScript Coverage](https://docs.sonarqube.org/latest/analysis/coverage/)
