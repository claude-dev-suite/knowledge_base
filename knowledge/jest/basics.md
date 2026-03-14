# Jest Testing Framework Reference

Comprehensive documentation for Jest testing framework.

**Official Documentation:** https://jestjs.io/docs/getting-started

---

## Table of Contents

1. [Setup](#setup)
2. [Test Functions](#test-functions)
3. [Matchers](#matchers)
4. [Setup and Teardown](#setup-and-teardown)
5. [Mocking](#mocking)
6. [Async Testing](#async-testing)
7. [Snapshot Testing](#snapshot-testing)
8. [Configuration](#configuration)
9. [CLI Options](#cli-options)

---

## Setup

### Installation

```bash
npm install --save-dev jest
npm install --save-dev @types/jest ts-jest # For TypeScript
```

### Basic Configuration

```javascript
// jest.config.js
module.exports = {
  testEnvironment: 'node', // or 'jsdom' for browser
  testMatch: ['**/__tests__/**/*.js', '**/*.test.js'],
  collectCoverageFrom: ['src/**/*.{js,jsx,ts,tsx}'],
  coverageThreshold: {
    global: {
      branches: 80,
      functions: 80,
      lines: 80,
      statements: 80,
    },
  },
};
```

### TypeScript Configuration

```javascript
// jest.config.js
module.exports = {
  preset: 'ts-jest',
  testEnvironment: 'node',
  moduleNameMapper: {
    '^@/(.*)$': '<rootDir>/src/$1',
  },
};
```

---

## Test Functions

### Basic Tests

```javascript
// Basic test
test('adds 1 + 2 to equal 3', () => {
  expect(1 + 2).toBe(3);
});

// Using it (alias for test)
it('adds 1 + 2 to equal 3', () => {
  expect(1 + 2).toBe(3);
});

// Grouping tests
describe('Calculator', () => {
  test('adds numbers', () => {
    expect(add(1, 2)).toBe(3);
  });

  test('subtracts numbers', () => {
    expect(subtract(5, 2)).toBe(3);
  });

  // Nested describe
  describe('division', () => {
    test('divides numbers', () => {
      expect(divide(6, 2)).toBe(3);
    });
  });
});
```

### Test Modifiers

```javascript
// Skip test
test.skip('skipped test', () => {});
it.skip('also skipped', () => {});
xtest('another way to skip', () => {});
xit('also works', () => {});

// Only run this test
test.only('focused test', () => {});
it.only('also focused', () => {});

// Placeholder test
test.todo('implement later');

// Parameterized tests
test.each([
  [1, 1, 2],
  [1, 2, 3],
  [2, 2, 4],
])('add(%i, %i) = %i', (a, b, expected) => {
  expect(add(a, b)).toBe(expected);
});

// Template literal syntax
test.each`
  a    | b    | expected
  ${1} | ${1} | ${2}
  ${1} | ${2} | ${3}
`('add($a, $b) = $expected', ({ a, b, expected }) => {
  expect(add(a, b)).toBe(expected);
});

// Concurrent tests
test.concurrent('concurrent test 1', async () => {
  await delay(100);
});
test.concurrent('concurrent test 2', async () => {
  await delay(100);
});

// Failing test marker
test.failing('known bug', () => {
  expect(buggyFunction()).toBe(expected); // Expected to fail
});
```

---

## Matchers

### Common Matchers

```javascript
// Equality
expect(2 + 2).toBe(4); // Strict equality
expect({ a: 1 }).toEqual({ a: 1 }); // Deep equality
expect({ a: 1, b: 2 }).toStrictEqual({ a: 1, b: 2 }); // Strict deep equality

// Truthiness
expect(value).toBeTruthy();
expect(value).toBeFalsy();
expect(value).toBeNull();
expect(value).toBeUndefined();
expect(value).toBeDefined();
expect(value).toBeNaN();

// Numbers
expect(value).toBeGreaterThan(3);
expect(value).toBeGreaterThanOrEqual(3.5);
expect(value).toBeLessThan(5);
expect(value).toBeLessThanOrEqual(4.5);
expect(0.1 + 0.2).toBeCloseTo(0.3, 5); // Floating point

// Strings
expect('team').toMatch(/tea/);
expect('team').toContain('tea');
expect('hello').toHaveLength(5);

// Arrays
expect([1, 2, 3]).toContain(2);
expect([{ a: 1 }, { a: 2 }]).toContainEqual({ a: 1 });
expect(['a', 'b']).toHaveLength(2);
expect(['a', 'b', 'c']).toEqual(expect.arrayContaining(['a', 'c']));

// Objects
expect(obj).toHaveProperty('name');
expect(obj).toHaveProperty('address.city', 'NYC');
expect(obj).toMatchObject({ name: 'John' });
expect(obj).toEqual(expect.objectContaining({ name: 'John' }));

// Negation
expect(value).not.toBe(5);
expect(array).not.toContain(4);
```

### Exception Matchers

```javascript
// Function throws
expect(() => throwingFn()).toThrow();
expect(() => throwingFn()).toThrow(Error);
expect(() => throwingFn()).toThrow('error message');
expect(() => throwingFn()).toThrow(/message/);

// Async throws
await expect(asyncFn()).rejects.toThrow();
await expect(asyncFn()).rejects.toThrow('message');
```

### Asymmetric Matchers

```javascript
expect(result).toEqual({
  id: expect.any(Number),
  name: expect.any(String),
  createdAt: expect.any(Date),
});

expect(result).toEqual({
  items: expect.arrayContaining([expect.any(Object)]),
});

expect(result).toEqual(expect.objectContaining({
  name: expect.stringContaining('test'),
  email: expect.stringMatching(/@example\.com$/),
}));

// Anything except null/undefined
expect(value).toEqual(expect.anything());
```

### Custom Matchers

```javascript
expect.extend({
  toBeWithinRange(received, floor, ceiling) {
    const pass = received >= floor && received <= ceiling;
    return {
      pass,
      message: () =>
        pass
          ? `expected ${received} not to be within range ${floor} - ${ceiling}`
          : `expected ${received} to be within range ${floor} - ${ceiling}`,
    };
  },
});

expect(100).toBeWithinRange(90, 110);
```

---

## Setup and Teardown

```javascript
// Before/after each test
beforeEach(() => {
  initializeDatabase();
});

afterEach(() => {
  clearDatabase();
});

// Before/after all tests in file
beforeAll(() => {
  return connectToDatabase();
});

afterAll(() => {
  return disconnectFromDatabase();
});

// Scoped to describe block
describe('Database tests', () => {
  beforeAll(() => setupTestDatabase());
  afterAll(() => teardownTestDatabase());

  beforeEach(() => seedData());
  afterEach(() => clearData());

  test('reads data', () => {});
  test('writes data', () => {});
});

// Async setup/teardown
beforeEach(async () => {
  await seedDatabase();
});

afterEach(async () => {
  await clearDatabase();
});
```

---

## Mocking

### Function Mocks

```javascript
// Create mock function
const mockFn = jest.fn();

// With implementation
const mockFn = jest.fn((x) => x * 2);

// Return values
const mock = jest.fn()
  .mockReturnValue(10)
  .mockReturnValueOnce(5);

// Async return values
const mockAsync = jest.fn()
  .mockResolvedValue({ data: [] })
  .mockResolvedValueOnce({ data: [1] })
  .mockRejectedValue(new Error('failed'));

// Implementation
const mock = jest.fn()
  .mockImplementation((x) => x * 2)
  .mockImplementationOnce((x) => x * 3);
```

### Mock Assertions

```javascript
const mockFn = jest.fn();

mockFn('arg1', 'arg2');
mockFn('arg3');

expect(mockFn).toHaveBeenCalled();
expect(mockFn).toHaveBeenCalledTimes(2);
expect(mockFn).toHaveBeenCalledWith('arg1', 'arg2');
expect(mockFn).toHaveBeenLastCalledWith('arg3');
expect(mockFn).toHaveBeenNthCalledWith(1, 'arg1', 'arg2');

// Return value assertions
expect(mockFn).toHaveReturned();
expect(mockFn).toHaveReturnedTimes(2);
expect(mockFn).toHaveReturnedWith(expectedValue);
expect(mockFn).toHaveLastReturnedWith(expectedValue);
```

### Spying

```javascript
// Spy on object method
const video = {
  play() { return true; },
  stop() { return false; }
};

const spy = jest.spyOn(video, 'play');
video.play();

expect(spy).toHaveBeenCalled();
spy.mockRestore(); // Restore original

// Spy with mock implementation
jest.spyOn(video, 'play').mockImplementation(() => 'mocked');

// Spy on module
import * as utils from './utils';
jest.spyOn(utils, 'formatDate').mockReturnValue('2024-01-01');
```

### Module Mocking

```javascript
// Mock entire module
jest.mock('./api', () => ({
  fetchUsers: jest.fn().mockResolvedValue([]),
  createUser: jest.fn().mockResolvedValue({ id: 1 }),
}));

// Partial mock
jest.mock('./utils', () => ({
  ...jest.requireActual('./utils'),
  formatDate: jest.fn().mockReturnValue('mocked'),
}));

// Auto-mock
jest.mock('./module'); // All exports become jest.fn()

// Mock with factory
jest.mock('./config', () => {
  return {
    __esModule: true,
    default: { apiUrl: 'http://test.com' },
    API_KEY: 'test-key',
  };
});
```

### Timer Mocks

```javascript
// Enable fake timers
jest.useFakeTimers();

test('timer test', () => {
  const callback = jest.fn();
  setTimeout(callback, 1000);

  expect(callback).not.toHaveBeenCalled();

  jest.advanceTimersByTime(1000);
  expect(callback).toHaveBeenCalledTimes(1);
});

// Other timer methods
jest.runAllTimers();
jest.runOnlyPendingTimers();
jest.advanceTimersToNextTimer();
jest.clearAllTimers();

// Mock Date
jest.setSystemTime(new Date('2024-01-01'));

// Restore real timers
jest.useRealTimers();
```

### Clearing Mocks

```javascript
// Clear mock call history
mockFn.mockClear();
jest.clearAllMocks();

// Reset mock (clear + remove return values)
mockFn.mockReset();
jest.resetAllMocks();

// Restore original (for spies)
mockFn.mockRestore();
jest.restoreAllMocks();

// In config
module.exports = {
  clearMocks: true, // Before each test
  resetMocks: true,
  restoreMocks: true,
};
```

---

## Async Testing

### Promises

```javascript
// Return promise
test('resolves with data', () => {
  return fetchData().then(data => {
    expect(data).toBe('data');
  });
});

// Resolves/Rejects
test('resolves', () => {
  return expect(fetchData()).resolves.toBe('data');
});

test('rejects', () => {
  return expect(fetchBad()).rejects.toThrow('error');
});
```

### Async/Await

```javascript
test('async test', async () => {
  const data = await fetchData();
  expect(data).toBe('data');
});

test('async resolves', async () => {
  await expect(fetchData()).resolves.toBe('data');
});

test('async rejects', async () => {
  await expect(fetchBad()).rejects.toThrow();
});
```

### Callbacks

```javascript
// Using done callback
test('callback test', done => {
  fetchDataWithCallback((error, data) => {
    try {
      expect(error).toBeNull();
      expect(data).toBe('data');
      done();
    } catch (e) {
      done(e);
    }
  });
});
```

### Timeouts

```javascript
// Per-test timeout
test('long test', async () => {
  await longOperation();
}, 10000); // 10 seconds

// Global timeout in config
module.exports = {
  testTimeout: 10000,
};
```

---

## Snapshot Testing

```javascript
// Basic snapshot
test('renders correctly', () => {
  const tree = renderer.create(<Button>Click</Button>).toJSON();
  expect(tree).toMatchSnapshot();
});

// Inline snapshot
test('renders correctly', () => {
  expect(render()).toMatchInlineSnapshot(`
    <div>
      <button>Click</button>
    </div>
  `);
});

// Property matchers (for dynamic values)
test('user object', () => {
  const user = createUser('John');
  expect(user).toMatchSnapshot({
    id: expect.any(String),
    createdAt: expect.any(Date),
  });
});

// Update snapshots
// jest --updateSnapshot
// jest -u
```

---

## Configuration

```javascript
// jest.config.js
module.exports = {
  // Test environment
  testEnvironment: 'jsdom', // or 'node'

  // Test file patterns
  testMatch: ['**/__tests__/**/*.[jt]s?(x)', '**/?(*.)+(spec|test).[jt]s?(x)'],
  testPathIgnorePatterns: ['/node_modules/'],

  // Module resolution
  moduleNameMapper: {
    '^@/(.*)$': '<rootDir>/src/$1',
    '\\.(css|less|scss)$': 'identity-obj-proxy',
  },
  moduleFileExtensions: ['js', 'jsx', 'ts', 'tsx', 'json'],

  // Setup
  setupFiles: ['<rootDir>/jest.setup.js'],
  setupFilesAfterEnv: ['<rootDir>/jest.setupTests.js'],
  globalSetup: '<rootDir>/jest.globalSetup.js',
  globalTeardown: '<rootDir>/jest.globalTeardown.js',

  // Transform
  transform: {
    '^.+\\.(ts|tsx)$': 'ts-jest',
  },

  // Coverage
  collectCoverage: true,
  collectCoverageFrom: ['src/**/*.{js,jsx,ts,tsx}', '!src/**/*.d.ts'],
  coverageDirectory: 'coverage',
  coverageReporters: ['text', 'lcov', 'html'],
  coverageThreshold: {
    global: {
      branches: 80,
      functions: 80,
      lines: 80,
      statements: 80,
    },
  },

  // Mocking
  clearMocks: true,
  resetMocks: true,
  restoreMocks: true,

  // Other
  verbose: true,
  testTimeout: 5000,
  maxWorkers: '50%',
};
```

---

## CLI Options

```bash
# Run all tests
jest

# Watch mode
jest --watch
jest --watchAll

# Run specific tests
jest path/to/test.js
jest --testNamePattern="pattern"
jest -t "test name"

# Coverage
jest --coverage
jest --coverageReporters="text"

# Update snapshots
jest --updateSnapshot
jest -u

# Run in band (no parallelization)
jest --runInBand

# Verbose output
jest --verbose

# Bail on first failure
jest --bail

# Configuration
jest --config=path/to/config.js
jest --showConfig

# Clear cache
jest --clearCache

# Debug
jest --debug
node --inspect-brk node_modules/.bin/jest --runInBand
```

---

## Quick Reference

### Test Functions

| Function | Description |
|----------|-------------|
| `test(name, fn)` | Define a test |
| `it(name, fn)` | Alias for test |
| `describe(name, fn)` | Group tests |
| `test.skip()` | Skip test |
| `test.only()` | Run only this test |
| `test.todo()` | Placeholder |
| `test.each()` | Parameterized tests |

### Common Matchers

| Matcher | Description |
|---------|-------------|
| `toBe(value)` | Strict equality |
| `toEqual(value)` | Deep equality |
| `toBeTruthy()` | Truthy value |
| `toBeFalsy()` | Falsy value |
| `toBeNull()` | Is null |
| `toBeUndefined()` | Is undefined |
| `toContain(item)` | Array/string contains |
| `toHaveLength(n)` | Length check |
| `toHaveProperty(key)` | Object has property |
| `toThrow()` | Function throws |
| `toMatch(regex)` | String matches pattern |

### Mock Functions

| Method | Description |
|--------|-------------|
| `jest.fn()` | Create mock function |
| `jest.spyOn(obj, 'method')` | Spy on method |
| `jest.mock('module')` | Mock module |
| `mockFn.mockReturnValue(val)` | Set return value |
| `mockFn.mockResolvedValue(val)` | Set resolved value |
| `mockFn.mockImplementation(fn)` | Set implementation |
| `mockFn.mockClear()` | Clear call history |
| `mockFn.mockReset()` | Reset mock |
| `mockFn.mockRestore()` | Restore original |
