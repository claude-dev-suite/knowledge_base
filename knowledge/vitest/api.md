# Vitest API Reference

Comprehensive documentation for the Vitest testing framework API.

---

## Table of Contents

1. [Test Functions](#test-functions)
2. [Assertions and Matchers](#assertions-and-matchers)
3. [Setup and Teardown](#setup-and-teardown)
4. [Test Modifiers](#test-modifiers)
5. [Async Testing](#async-testing)
6. [Snapshot Testing](#snapshot-testing)
7. [Mocking](#mocking)
8. [Test Context and Fixtures](#test-context-and-fixtures)
9. [Test Filtering](#test-filtering)
10. [Type Testing](#type-testing)
11. [Configuration Options](#configuration-options)
12. [CLI Options](#cli-options)
13. [Watch Mode](#watch-mode)
14. [Best Practices](#best-practices)
15. [Common Patterns](#common-patterns)

---

## Test Functions

### Basic Test Functions

```typescript
import { describe, it, test, expect, vi } from 'vitest';

// Basic test - defines a set of related expectations
test('adds numbers correctly', () => {
  expect(1 + 1).toBe(2);
});

// `it` is an alias for `test`
it('adds numbers correctly', () => {
  expect(1 + 1).toBe(2);
});

// Test with timeout (in milliseconds)
test('long running operation', async () => {
  await longOperation();
}, 10000); // 10 second timeout
```

### Grouping Tests with describe

```typescript
// Group related tests
describe('Calculator', () => {
  it('adds two numbers', () => {
    expect(add(1, 2)).toBe(3);
  });

  it('subtracts two numbers', () => {
    expect(subtract(5, 3)).toBe(2);
  });

  // Nested describe blocks
  describe('advanced operations', () => {
    it('calculates square root', () => {
      expect(Math.sqrt(4)).toBe(2);
    });
  });
});
```

### Benchmark Testing

```typescript
import { bench, describe } from 'vitest';

// Basic benchmark
bench('sort array', () => {
  const arr = [3, 1, 2];
  arr.sort();
});

// Benchmark with options
bench('complex operation', () => {
  heavyComputation();
}, { time: 1000, iterations: 100 });

// Grouped benchmarks
describe('sorting algorithms', () => {
  bench('quicksort', () => quickSort(data));
  bench('mergesort', () => mergeSort(data));
  bench('bubblesort', () => bubbleSort(data));
});
```

---

## Assertions and Matchers

### Equality Matchers

```typescript
// Strict equality (===) using Object.is()
expect(2 + 2).toBe(4);
expect('hello').toBe('hello');

// Deep equality for objects and arrays
expect({ name: 'John' }).toEqual({ name: 'John' });
expect([1, 2, 3]).toEqual([1, 2, 3]);

// Strict deep equality (checks types, undefined properties, array sparseness)
expect(new Stock('apples')).not.toStrictEqual({ type: 'apples' });
```

### Truthiness Matchers

```typescript
// Truthy/Falsy
expect(1).toBeTruthy();
expect(0).toBeFalsy();
expect('').toBeFalsy();
expect('hello').toBeTruthy();

// Null and Undefined
expect(null).toBeNull();
expect(undefined).toBeUndefined();
expect('defined').toBeDefined();

// Nullable (null or undefined)
expect(null).toBeNullable();
expect(undefined).toBeNullable();

// NaN
expect(NaN).toBeNaN();
expect(0 / 0).toBeNaN();
```

### Number Matchers

```typescript
// Comparisons
expect(10).toBeGreaterThan(5);
expect(10).toBeGreaterThanOrEqual(10);
expect(5).toBeLessThan(10);
expect(5).toBeLessThanOrEqual(5);

// Floating point comparisons (handles precision issues)
expect(0.1 + 0.2).toBeCloseTo(0.3, 5); // 5 decimal places
expect(Math.PI).toBeCloseTo(3.14159, 4);
```

### String Matchers

```typescript
// Pattern matching with regex
expect('hello world').toMatch(/world/);
expect('hello world').toMatch(/^hello/);

// Substring matching
expect('hello world').toContain('world');
expect('pineapple').toContain('apple');

// String with length
expect('hello').toHaveLength(5);
```

### Array and Iterable Matchers

```typescript
// Contains item
expect(['apple', 'banana', 'orange']).toContain('banana');

// Contains structurally equal item
expect([{ id: 1 }, { id: 2 }]).toContainEqual({ id: 1 });

// Length
expect([1, 2, 3]).toHaveLength(3);

// Array containing subset
expect(['apple', 'banana', 'orange']).toEqual(
  expect.arrayContaining(['apple', 'orange'])
);
```

### Object Matchers

```typescript
const user = {
  name: 'John',
  address: {
    city: 'New York',
    country: 'USA'
  }
};

// Property existence
expect(user).toHaveProperty('name');
expect(user).toHaveProperty('address.city');

// Property with specific value
expect(user).toHaveProperty('name', 'John');
expect(user).toHaveProperty('address.city', 'New York');

// Partial object matching
expect(user).toMatchObject({
  name: 'John',
  address: { city: 'New York' }
});

// Object containing subset
expect(user).toEqual(expect.objectContaining({
  name: 'John'
}));
```

### Type Matchers

```typescript
// typeof check
expect('hello').toBeTypeOf('string');
expect(42).toBeTypeOf('number');
expect({}).toBeTypeOf('object');
expect(() => {}).toBeTypeOf('function');

// instanceof check
expect(new Date()).toBeInstanceOf(Date);
expect([]).toBeInstanceOf(Array);
expect(new Error()).toBeInstanceOf(Error);

// One of multiple values
expect('apple').toBeOneOf(['apple', 'banana', 'orange']);
expect(status).toBeOneOf([200, 201, 204]);
```

### Exception Matchers

```typescript
// Basic throw check
expect(() => {
  throw new Error('Something went wrong');
}).toThrow();

// Throw with specific message
expect(() => {
  throw new Error('Invalid input');
}).toThrow('Invalid input');

// Throw with regex pattern
expect(() => {
  throw new Error('User not found: 123');
}).toThrow(/not found/);

// Throw specific error type
expect(() => {
  throw new TypeError('Expected string');
}).toThrow(TypeError);

// Throw exact error
expect(() => {
  throw new Error('Exact message');
}).toThrow(new Error('Exact message'));
```

### Asymmetric Matchers

```typescript
// Match any value (except null/undefined)
expect({ id: 1, name: 'John' }).toEqual({
  id: expect.anything(),
  name: 'John'
});

// Match any instance of type
expect({ id: generateId(), createdAt: new Date() }).toEqual({
  id: expect.any(Number),
  createdAt: expect.any(Date)
});

// String containing substring
expect({ message: 'Hello, World!' }).toEqual({
  message: expect.stringContaining('World')
});

// String matching pattern
expect({ email: 'user@example.com' }).toEqual({
  email: expect.stringMatching(/@example\.com$/)
});

// Array containing items
expect({ tags: ['a', 'b', 'c'] }).toEqual({
  tags: expect.arrayContaining(['a', 'c'])
});

// Object containing properties
expect({ user: { id: 1, name: 'John', age: 30 } }).toEqual({
  user: expect.objectContaining({ id: 1, name: 'John' })
});

// Floating point in objects
expect({ total: 0.1 + 0.2 }).toEqual({
  total: expect.closeTo(0.3, 5)
});
```

### Custom Predicates

```typescript
// Custom satisfaction check
const isEven = (n: number) => n % 2 === 0;
expect(4).toSatisfy(isEven);
expect(5).not.toSatisfy(isEven);

// Complex custom validation
expect(user).toSatisfy((u) =>
  u.age >= 18 && u.email.includes('@')
);
```

### Negation

```typescript
// Use .not to negate any matcher
expect(5).not.toBe(10);
expect([1, 2]).not.toContain(3);
expect({ a: 1 }).not.toHaveProperty('b');
expect(() => safeFn()).not.toThrow();
```

---

## Setup and Teardown

### Basic Hooks

```typescript
import { beforeAll, afterAll, beforeEach, afterEach } from 'vitest';

// Run once before all tests in the file/suite
beforeAll(() => {
  console.log('Setting up test environment');
});

// Run once after all tests complete
afterAll(() => {
  console.log('Cleaning up test environment');
});

// Run before each individual test
beforeEach(() => {
  console.log('Preparing for test');
});

// Run after each individual test
afterEach(() => {
  console.log('Cleaning up after test');
});
```

### Async Hooks

```typescript
beforeAll(async () => {
  await database.connect();
  await database.seed();
});

afterAll(async () => {
  await database.clear();
  await database.disconnect();
});

beforeEach(async () => {
  await database.beginTransaction();
});

afterEach(async () => {
  await database.rollbackTransaction();
});
```

### Hooks with Timeout

```typescript
// Hook with custom timeout (default is 5000ms)
beforeAll(async () => {
  await slowSetup();
}, 30000); // 30 second timeout

afterAll(async () => {
  await slowCleanup();
}, 30000);
```

### Scoped Hooks

```typescript
describe('Database Tests', () => {
  beforeAll(async () => {
    // Only runs before tests in this describe block
    await connectToTestDatabase();
  });

  afterAll(async () => {
    await disconnectFromTestDatabase();
  });

  describe('User operations', () => {
    beforeEach(() => {
      // Runs before each test in this nested block
    });

    it('creates a user', () => {});
    it('updates a user', () => {});
  });
});
```

### Test-Level Hooks

```typescript
import { onTestFinished, onTestFailed } from 'vitest';

it('test with cleanup', async ({ onTestFinished }) => {
  const resource = await acquireResource();

  // Guaranteed to run after test completes
  onTestFinished(async () => {
    await resource.release();
  });

  // Test logic here
});

it('test with failure handler', async ({ onTestFailed }) => {
  onTestFailed((error) => {
    console.log('Test failed with:', error.message);
    // Useful for debugging, screenshots, etc.
  });

  // Test logic here
});
```

---

## Test Modifiers

### skip - Skip Tests

```typescript
// Skip individual test
it.skip('skipped test', () => {
  // This test will not run
});

// Skip entire suite
describe.skip('skipped suite', () => {
  it('test 1', () => {});
  it('test 2', () => {});
});

// Conditional skip
it.skipIf(process.env.CI)('skip in CI', () => {
  // Only runs locally
});

// Skip programmatically in test
it('conditional skip', ({ skip }) => {
  if (someCondition) {
    skip(); // Skip remaining test execution
  }
  // Test continues if not skipped
});
```

### only - Focus Tests

```typescript
// Run only this test
it.only('focused test', () => {
  // Only this test will run
});

// Run only this suite
describe.only('focused suite', () => {
  it('test 1', () => {}); // Runs
  it('test 2', () => {}); // Runs
});
```

### todo - Placeholder Tests

```typescript
// Mark test as todo (shows in reports)
it.todo('implement user authentication');
it.todo('add input validation');

describe.todo('Payment Processing');
```

### concurrent - Parallel Execution

```typescript
// Run tests concurrently
describe('Concurrent Tests', () => {
  it.concurrent('test 1', async () => {
    await delay(1000);
  });

  it.concurrent('test 2', async () => {
    await delay(1000);
  });

  it.concurrent('test 3', async () => {
    await delay(1000);
  });
});

// All tests in suite run concurrently
describe.concurrent('All Concurrent', () => {
  it('test 1', async () => {});
  it('test 2', async () => {});
});
```

### sequential - Sequential Execution

```typescript
// Force sequential execution within concurrent suite
describe.concurrent('Mixed Execution', () => {
  it('concurrent 1', async () => {});
  it('concurrent 2', async () => {});

  describe.sequential('Must be sequential', () => {
    it('step 1', async () => {}); // Runs first
    it('step 2', async () => {}); // Runs after step 1
  });
});

it.sequential('sequential test', async () => {});
```

### each - Parameterized Tests

```typescript
// Array of arrays
it.each([
  [1, 1, 2],
  [1, 2, 3],
  [2, 2, 4],
])('add(%i, %i) = %i', (a, b, expected) => {
  expect(add(a, b)).toBe(expected);
});

// Array of objects
it.each([
  { input: 'hello', expected: 5 },
  { input: 'world', expected: 5 },
  { input: 'hi', expected: 2 },
])('length of "$input" is $expected', ({ input, expected }) => {
  expect(input.length).toBe(expected);
});

// Template literal syntax
it.each`
  a    | b    | expected
  ${1} | ${1} | ${2}
  ${1} | ${2} | ${3}
  ${2} | ${2} | ${4}
`('add($a, $b) = $expected', ({ a, b, expected }) => {
  expect(add(a, b)).toBe(expected);
});

// Printf formatting
// %s - string, %d/%i - integer, %f - float
// %j - JSON, %o - object, %# - index, %% - literal %
it.each([
  ['apple', 5],
  ['banana', 6],
])('%s has %d characters', (fruit, length) => {
  expect(fruit.length).toBe(length);
});

// Describe with each
describe.each([
  { name: 'Chrome', version: 90 },
  { name: 'Firefox', version: 88 },
  { name: 'Safari', version: 14 },
])('Browser: $name v$version', ({ name, version }) => {
  it('should load page', () => {});
  it('should execute JavaScript', () => {});
});
```

### for - Alternative to each

```typescript
// test.for provides TestContext and doesn't spread arrays
test.for([
  [1, 2, 3],
  [4, 5, 9],
])('add(%d, %d) = %d', ([a, b, expected], { expect }) => {
  expect(add(a, b)).toBe(expected);
});
```

### fails - Expected Failures

```typescript
// Mark test as expected to fail
it.fails('this assertion is wrong', () => {
  expect(1).toBe(2); // Test passes because it fails as expected
});

// Useful for documenting known bugs
it.fails('known bug #123', () => {
  expect(buggyFunction()).toBe(correctResult);
});
```

### runIf - Conditional Execution

```typescript
// Run only if condition is true
it.runIf(process.env.NODE_ENV === 'development')('dev only test', () => {
  // Only runs in development
});

it.runIf(hasFeatureFlag('newFeature'))('new feature test', () => {
  // Only runs when feature flag is enabled
});
```

### shuffle - Random Order

```typescript
// Run tests in random order
describe.shuffle('Randomized Tests', () => {
  it('test 1', () => {});
  it('test 2', () => {});
  it('test 3', () => {});
});
```

---

## Async Testing

### Promises

```typescript
// Return promise
it('returns data', () => {
  return fetchData().then(data => {
    expect(data).toBe('expected data');
  });
});

// resolves helper
it('resolves with data', async () => {
  await expect(fetchData()).resolves.toBe('expected data');
  await expect(fetchUser(1)).resolves.toEqual({ id: 1, name: 'John' });
});

// rejects helper
it('rejects with error', async () => {
  await expect(fetchInvalid()).rejects.toThrow('Not found');
  await expect(fetchInvalid()).rejects.toBeInstanceOf(NotFoundError);
});
```

### Async/Await

```typescript
it('async test', async () => {
  const result = await fetchData();
  expect(result).toBe('expected data');
});

it('multiple async operations', async () => {
  const user = await createUser({ name: 'John' });
  const updatedUser = await updateUser(user.id, { name: 'Jane' });
  expect(updatedUser.name).toBe('Jane');
});

it('parallel async operations', async () => {
  const [users, posts] = await Promise.all([
    fetchUsers(),
    fetchPosts()
  ]);
  expect(users).toHaveLength(10);
  expect(posts).toHaveLength(20);
});
```

### Callbacks

```typescript
// Use done callback for callback-based async
it('callback test', (done) => {
  fetchDataWithCallback((error, result) => {
    try {
      expect(error).toBeNull();
      expect(result).toBe('data');
      done();
    } catch (e) {
      done(e);
    }
  });
});

// Better: promisify and use async/await
it('promisified callback', async () => {
  const result = await new Promise((resolve, reject) => {
    fetchDataWithCallback((error, data) => {
      if (error) reject(error);
      else resolve(data);
    });
  });
  expect(result).toBe('data');
});
```

### Fake Timers

```typescript
import { vi } from 'vitest';

beforeEach(() => {
  vi.useFakeTimers();
});

afterEach(() => {
  vi.useRealTimers();
});

it('handles setTimeout', () => {
  const callback = vi.fn();
  setTimeout(callback, 1000);

  expect(callback).not.toHaveBeenCalled();

  vi.advanceTimersByTime(1000);
  expect(callback).toHaveBeenCalledTimes(1);
});

it('handles setInterval', () => {
  const callback = vi.fn();
  setInterval(callback, 1000);

  vi.advanceTimersByTime(3000);
  expect(callback).toHaveBeenCalledTimes(3);
});

it('runs all timers', () => {
  const callback = vi.fn();
  setTimeout(callback, 1000);
  setTimeout(callback, 2000);
  setTimeout(callback, 3000);

  vi.runAllTimers();
  expect(callback).toHaveBeenCalledTimes(3);
});

it('advances to next timer', () => {
  const callback1 = vi.fn();
  const callback2 = vi.fn();

  setTimeout(callback1, 1000);
  setTimeout(callback2, 2000);

  vi.advanceTimersToNextTimer();
  expect(callback1).toHaveBeenCalled();
  expect(callback2).not.toHaveBeenCalled();
});

it('mocks Date', () => {
  vi.setSystemTime(new Date('2024-01-01'));

  expect(new Date().getFullYear()).toBe(2024);
  expect(Date.now()).toBe(new Date('2024-01-01').getTime());
});
```

### Polling with expect.poll

```typescript
// Retry assertion until it passes
it('waits for element', async () => {
  triggerAsyncUpdate();

  await expect.poll(() => {
    return document.querySelector('.updated');
  }).toBeTruthy();
});

// With custom options
await expect.poll(
  () => fetchStatus(),
  {
    timeout: 10000, // 10 seconds
    interval: 500,  // Check every 500ms
  }
).toBe('complete');
```

### vi.waitFor

```typescript
import { vi } from 'vitest';

it('waits for condition', async () => {
  let ready = false;
  setTimeout(() => { ready = true; }, 100);

  await vi.waitFor(() => {
    expect(ready).toBe(true);
  });
});

// With options
await vi.waitFor(
  async () => {
    const response = await fetch('/api/status');
    expect(response.ok).toBe(true);
  },
  {
    timeout: 5000,
    interval: 100
  }
);
```

---

## Snapshot Testing

### Basic Snapshots

```typescript
import { expect, it } from 'vitest';

it('matches snapshot', () => {
  const user = {
    id: 1,
    name: 'John',
    createdAt: new Date('2024-01-01')
  };

  expect(user).toMatchSnapshot();
});

// With custom name
it('matches named snapshot', () => {
  expect(data).toMatchSnapshot('user data');
});
```

### Inline Snapshots

```typescript
it('matches inline snapshot', () => {
  const result = formatUser({ name: 'John', age: 30 });

  // Snapshot stored directly in code (auto-updated)
  expect(result).toMatchInlineSnapshot(`
    {
      "age": 30,
      "name": "John",
    }
  `);
});
```

### File Snapshots

```typescript
it('matches file snapshot', async () => {
  const html = renderComponent(<UserCard user={user} />);

  // Compares against external file
  await expect(html).toMatchFileSnapshot('./snapshots/user-card.html');
});
```

### Error Snapshots

```typescript
it('error matches snapshot', () => {
  expect(() => {
    throw new Error('Invalid input: expected number');
  }).toThrowErrorMatchingSnapshot();
});

it('error matches inline snapshot', () => {
  expect(() => {
    throw new Error('User not found');
  }).toThrowErrorMatchingInlineSnapshot(`"User not found"`);
});
```

### Property Matchers in Snapshots

```typescript
it('snapshot with property matchers', () => {
  const user = createUser('John');

  expect(user).toMatchSnapshot({
    id: expect.any(String),
    createdAt: expect.any(Date),
    name: 'John'
  });
});
```

### Updating Snapshots

```bash
# Update all snapshots
vitest -u
vitest --update

# In watch mode, press 'u' to update
```

### Custom Serializers

```typescript
// vitest.config.ts
export default defineConfig({
  test: {
    snapshotSerializers: ['./custom-serializer.ts']
  }
});

// Or in test file
expect.addSnapshotSerializer({
  serialize(val, config, indentation, depth, refs, printer) {
    return `User: ${val.name}`;
  },
  test(val) {
    return val && val.hasOwnProperty('name');
  }
});
```

---

## Mocking

### Function Mocks

```typescript
import { vi } from 'vitest';

// Create mock function
const mockFn = vi.fn();

// Mock with implementation
const mockAdd = vi.fn((a, b) => a + b);

// Mock with return value
const mockGetUser = vi.fn().mockReturnValue({ id: 1, name: 'John' });

// Mock with resolved value (for async)
const mockFetchUser = vi.fn().mockResolvedValue({ id: 1, name: 'John' });

// Mock with rejected value
const mockFailedFetch = vi.fn().mockRejectedValue(new Error('Network error'));

// Mock implementation once
const mockOnce = vi.fn()
  .mockReturnValueOnce('first')
  .mockReturnValueOnce('second')
  .mockReturnValue('default');

expect(mockOnce()).toBe('first');
expect(mockOnce()).toBe('second');
expect(mockOnce()).toBe('default');
```

### Spying on Methods

```typescript
const calculator = {
  add: (a: number, b: number) => a + b,
  multiply: (a: number, b: number) => a * b
};

// Spy on method (preserves original implementation)
const spy = vi.spyOn(calculator, 'add');

calculator.add(1, 2); // Calls original

expect(spy).toHaveBeenCalledWith(1, 2);
expect(spy).toHaveReturnedWith(3);

// Spy with mock implementation
vi.spyOn(calculator, 'multiply').mockImplementation(() => 0);
expect(calculator.multiply(2, 3)).toBe(0);

// Restore original
spy.mockRestore();
```

### Module Mocking

```typescript
import { vi } from 'vitest';

// Mock entire module (hoisted to top)
vi.mock('./utils', () => ({
  formatDate: vi.fn(() => '2024-01-01'),
  parseDate: vi.fn()
}));

// Mock with auto-mocking
vi.mock('./api'); // All exports become vi.fn()

// Partial mock (keep some original)
vi.mock('./utils', async (importOriginal) => {
  const actual = await importOriginal();
  return {
    ...actual,
    formatDate: vi.fn(() => 'mocked date')
  };
});

// Import actual in mock
import { formatDate } from './utils';
vi.mocked(formatDate).mockReturnValue('custom date');
```

### Dynamic Mocking

```typescript
// vi.doMock is not hoisted (for dynamic imports)
vi.doMock('./config', () => ({
  apiUrl: 'http://test-api.com'
}));

const { apiUrl } = await import('./config');
expect(apiUrl).toBe('http://test-api.com');

// Unmock
vi.doUnmock('./config');
```

### Global Mocking

```typescript
// Mock global variable
vi.stubGlobal('fetch', vi.fn().mockResolvedValue({
  json: () => Promise.resolve({ data: 'mocked' })
}));

// Mock environment variable
vi.stubEnv('API_KEY', 'test-key');

// Restore all
afterEach(() => {
  vi.unstubAllGlobals();
  vi.unstubAllEnvs();
});
```

### Mock Assertions

```typescript
const mock = vi.fn();

mock('a', 'b');
mock('c');
mock('d', 'e', 'f');

// Call assertions
expect(mock).toHaveBeenCalled();
expect(mock).toHaveBeenCalledTimes(3);
expect(mock).toHaveBeenCalledWith('a', 'b');
expect(mock).toHaveBeenLastCalledWith('d', 'e', 'f');
expect(mock).toHaveBeenNthCalledWith(2, 'c');

// Return value assertions
const mockReturn = vi.fn(() => 'result');
mockReturn();

expect(mockReturn).toHaveReturned();
expect(mockReturn).toHaveReturnedTimes(1);
expect(mockReturn).toHaveReturnedWith('result');

// Call order
const mock1 = vi.fn();
const mock2 = vi.fn();
mock1();
mock2();

expect(mock1).toHaveBeenCalledBefore(mock2);
expect(mock2).toHaveBeenCalledAfter(mock1);
```

### Resetting Mocks

```typescript
const mock = vi.fn(() => 'value');

// Clear call history (keeps implementation)
mock.mockClear();
// Or for all mocks
vi.clearAllMocks();

// Reset to empty function
mock.mockReset();
// Or for all mocks
vi.resetAllMocks();

// Restore original (for spies)
mock.mockRestore();
// Or for all mocks
vi.restoreAllMocks();
```

---

## Test Context and Fixtures

### Built-in Context

```typescript
it('has test context', ({ task, expect, skip }) => {
  // task - test metadata
  console.log(task.name); // test name

  // expect - bound to current test
  expect(1).toBe(1);

  // skip - skip test conditionally
  if (condition) skip();
});
```

### Custom Fixtures with test.extend

```typescript
import { test as baseTest } from 'vitest';

// Define custom test with fixtures
const test = baseTest.extend({
  // Simple fixture
  page: async ({}, use) => {
    const page = await createPage();
    await use(page);
    await page.close();
  },

  // Fixture depending on another
  authenticatedPage: async ({ page }, use) => {
    await page.login('user', 'pass');
    await use(page);
    await page.logout();
  },

  // Fixture with cleanup
  database: async ({}, use) => {
    const db = await connectDatabase();
    await db.seed();
    await use(db);
    await db.clear();
    await db.disconnect();
  }
});

// Use custom test
test('uses fixtures', async ({ page, database }) => {
  const users = await database.getUsers();
  await page.displayUsers(users);
});

// Fixtures are automatically set up and torn down
```

### Automatic Fixtures

```typescript
const test = baseTest.extend({
  // Auto fixture runs for every test
  logger: [
    async ({}, use) => {
      console.log('Test starting');
      await use(createLogger());
      console.log('Test complete');
    },
    { auto: true }
  ]
});

test('logger is automatically available', () => {
  // logger fixture runs even if not used
});
```

### Scoped Fixtures

```typescript
const test = baseTest.extend({
  // Worker scope - shared across tests in same worker
  workerDb: [
    async ({}, use) => {
      const db = await connectDatabase();
      await use(db);
      await db.disconnect();
    },
    { scope: 'worker' }
  ],

  // File scope - shared across tests in same file
  fileSetup: [
    async ({}, use) => {
      await setup();
      await use(null);
      await teardown();
    },
    { scope: 'file' }
  ]
});
```

### Suite-Scoped Context

```typescript
const test = baseTest.extend({
  items: async ({}, use) => {
    await use([]);
  }
});

describe('Shopping Cart', () => {
  // Override fixture for this suite
  test.scoped({ items: ['apple', 'banana'] });

  test('has pre-populated items', ({ items }) => {
    expect(items).toHaveLength(2);
  });
});
```

### Injected Fixtures

```typescript
// vitest.config.ts
export default defineConfig({
  test: {
    provide: {
      apiUrl: 'http://localhost:3000'
    }
  }
});

// test file
const test = baseTest.extend({
  apiUrl: [
    async ({ inject }, use) => {
      await use(inject('apiUrl'));
    },
    { injected: true }
  ]
});
```

---

## Test Filtering

### CLI Filtering

```bash
# Filter by file name
vitest basic              # runs files containing "basic"
vitest user.test.ts       # runs specific file

# Filter by test name pattern
vitest -t "should create"
vitest --testNamePattern "user.*login"

# Filter by line number (Vitest 3+)
vitest ./tests/user.test.ts:25

# Run related tests
vitest related src/utils.ts
```

### Programmatic Filtering

```typescript
// Skip based on condition
it.skipIf(process.env.CI)('local only test', () => {});
it.skipIf(!hasDatabase)('requires database', () => {});

// Run based on condition
it.runIf(process.env.INTEGRATION)('integration test', () => {});
it.runIf(isLinux)('linux specific test', () => {});
```

### Using skip in Context

```typescript
it('conditional execution', ({ skip }) => {
  const result = checkPrerequisites();

  if (!result.ready) {
    skip(); // Skip remaining test
    return;
  }

  // Continue with test
  expect(result.data).toBeDefined();
});

// Skip with message (v3.1+)
it('with skip message', ({ skip }) => {
  if (condition) {
    skip('Prerequisite not met');
  }
});
```

---

## Type Testing

### expectTypeOf API

```typescript
import { expectTypeOf } from 'vitest';

// Basic type checks
expectTypeOf('hello').toBeString();
expectTypeOf(42).toBeNumber();
expectTypeOf(true).toBeBoolean();
expectTypeOf(null).toBeNull();
expectTypeOf(undefined).toBeUndefined();
expectTypeOf([]).toBeArray();
expectTypeOf({}).toBeObject();
expectTypeOf(() => {}).toBeFunction();

// Type equality
expectTypeOf<string>().toEqualTypeOf<string>();
expectTypeOf({ a: 1 }).toEqualTypeOf<{ a: number }>();

// Type extension
expectTypeOf<{ a: 1; b: 2 }>().toExtend<{ a: number }>();
expectTypeOf<number>().toExtend<string | number>();

// Function types
expectTypeOf<(a: string) => number>().toBeFunction();
expectTypeOf<(a: string) => number>().parameters.toEqualTypeOf<[string]>();
expectTypeOf<(a: string) => number>().returns.toEqualTypeOf<number>();
expectTypeOf<(a: string) => number>().parameter(0).toBeString();

// Callable checks
expectTypeOf<(a: string) => void>().toBeCallableWith('hello');

// Constructor types
expectTypeOf(Date).toBeConstructibleWith(new Date());
expectTypeOf(Date).constructorParameters.toEqualTypeOf<[string | number | Date] | []>();

// Instance types
expectTypeOf(Date).instance.toHaveProperty('toISOString');

// Promise types
expectTypeOf(Promise.resolve('hello')).resolves.toBeString();

// Array element types
expectTypeOf([1, 2, 3]).items.toEqualTypeOf<number>();

// Object properties
expectTypeOf({ a: 1, b: 'hello' }).toHaveProperty('a');
expectTypeOf({ a: 1, b: 'hello' }).toHaveProperty('a').toBeNumber();

// Negation
expectTypeOf<string>().not.toEqualTypeOf<number>();

// Union extraction
type Response = string | number | { error: string };
expectTypeOf<Response>().extract<string>().toEqualTypeOf<string>();
expectTypeOf<Response>().exclude<string>().toEqualTypeOf<number | { error: string }>();
```

### assertType API

```typescript
import { assertType } from 'vitest';

// Direct type assertion
const value = 42;
assertType<number>(value);

// With error expectation
// @ts-expect-error - string is not assignable to number
assertType<number>('hello');
```

### Type Test Files

```typescript
// user.test-d.ts (automatically treated as type test)
import { expectTypeOf } from 'vitest';
import { createUser, User } from './user';

it('createUser returns User type', () => {
  expectTypeOf(createUser).returns.toEqualTypeOf<User>();
});

it('User has required properties', () => {
  expectTypeOf<User>().toHaveProperty('id');
  expectTypeOf<User>().toHaveProperty('name');
  expectTypeOf<User>().toHaveProperty('email');
});
```

### Running Type Tests

```bash
# Run with type checking
vitest --typecheck

# Or configure in vitest.config.ts
export default defineConfig({
  test: {
    typecheck: {
      enabled: true,
      include: ['**/*.test-d.ts']
    }
  }
});
```

---

## Configuration Options

### Basic Configuration

```typescript
// vitest.config.ts
import { defineConfig } from 'vitest/config';

export default defineConfig({
  test: {
    // Test file patterns
    include: ['**/*.{test,spec}.{js,mjs,cjs,ts,mts,cts,jsx,tsx}'],
    exclude: ['**/node_modules/**', '**/dist/**'],

    // Global test timeout (default: 5000ms)
    testTimeout: 10000,
    hookTimeout: 10000,

    // Enable globals (describe, it, expect without import)
    globals: true,

    // Test environment
    environment: 'node', // 'node' | 'jsdom' | 'happy-dom' | 'edge-runtime'

    // Setup files (run before each test file)
    setupFiles: ['./tests/setup.ts'],

    // Global setup (run once before all tests)
    globalSetup: './tests/global-setup.ts'
  }
});
```

### Reporter Configuration

```typescript
export default defineConfig({
  test: {
    // Single reporter
    reporters: 'verbose',

    // Multiple reporters
    reporters: ['default', 'json', 'html'],

    // Reporter with options
    reporters: [
      'default',
      ['json', { outputFile: './test-results.json' }],
      ['html', { outputFile: './test-report.html' }]
    ],

    // Output file for reporters
    outputFile: {
      json: './results/json-report.json',
      junit: './results/junit-report.xml'
    }
  }
});
```

### Coverage Configuration

```typescript
export default defineConfig({
  test: {
    coverage: {
      enabled: true,
      provider: 'v8', // 'v8' | 'istanbul'
      reporter: ['text', 'json', 'html'],
      reportsDirectory: './coverage',

      // Coverage thresholds
      thresholds: {
        lines: 80,
        functions: 80,
        branches: 80,
        statements: 80
      },

      // Include/exclude patterns
      include: ['src/**/*.ts'],
      exclude: ['src/**/*.test.ts', 'src/**/*.d.ts']
    }
  }
});
```

### Pool Configuration

```typescript
export default defineConfig({
  test: {
    // Process pool type
    pool: 'forks', // 'forks' | 'threads' | 'vmForks' | 'vmThreads'

    // Pool options
    poolOptions: {
      forks: {
        singleFork: false,
        isolate: true
      },
      threads: {
        singleThread: false,
        isolate: true
      }
    },

    // Max parallel workers
    maxWorkers: 4,

    // Min parallel workers
    minWorkers: 1,

    // Run files in parallel
    fileParallelism: true,

    // Max concurrent tests
    maxConcurrency: 5
  }
});
```

### Mock Configuration

```typescript
export default defineConfig({
  test: {
    // Clear mocks between tests
    clearMocks: true,

    // Reset mocks between tests
    mockReset: true,

    // Restore mocks between tests
    restoreMocks: true,

    // Unstub env variables
    unstubEnvs: true,

    // Unstub global variables
    unstubGlobals: true
  }
});
```

### Snapshot Configuration

```typescript
export default defineConfig({
  test: {
    // Snapshot format options
    snapshotFormat: {
      printBasicPrototype: false,
      escapeString: false
    },

    // Custom snapshot serializers
    snapshotSerializers: ['./tests/serializers/custom.ts'],

    // Custom snapshot path resolver
    resolveSnapshotPath: (testPath, snapExtension) => {
      return testPath.replace('tests', '__snapshots__') + snapExtension;
    }
  }
});
```

### Watch Mode Configuration

```typescript
export default defineConfig({
  test: {
    // Watch for changes
    watch: true,

    // Files that trigger full rerun
    forceRerunTriggers: [
      '**/package.json',
      '**/vitest.config.ts'
    ],

    // Additional watch patterns
    watchTriggerPatterns: [
      '**/fixtures/**'
    ]
  }
});
```

### Sequence Configuration

```typescript
export default defineConfig({
  test: {
    sequence: {
      // Shuffle test order
      shuffle: true,

      // Seed for shuffle (for reproducibility)
      seed: 12345,

      // Hook execution order
      hooks: 'list' // 'list' | 'stack' | 'parallel'
    }
  }
});
```

### TypeScript Configuration

```typescript
export default defineConfig({
  test: {
    typecheck: {
      enabled: true,
      checker: 'tsc', // 'tsc' | 'vue-tsc'
      include: ['**/*.test-d.ts'],
      tsconfig: './tsconfig.test.json'
    }
  }
});
```

---

## CLI Options

### Common Commands

```bash
# Start in watch mode (development)
vitest

# Single run (CI)
vitest run

# Watch mode explicitly
vitest watch
vitest dev

# Run benchmarks
vitest bench

# Initialize project
vitest init
vitest init browser

# List matching tests
vitest list
vitest list --json
```

### Filtering Options

```bash
# Filter by file name
vitest user
vitest "**/*.integration.test.ts"

# Filter by test name
vitest -t "should create user"
vitest --testNamePattern "login.*success"

# Run tests related to changed files
vitest --changed
vitest --changed HEAD~1

# Run related tests
vitest related src/utils.ts
```

### Execution Options

```bash
# Specify config file
vitest -c ./config/vitest.config.ts
vitest --config ./custom.config.ts

# Set root directory
vitest -r ./src
vitest --root ./packages/core

# Set environment
vitest --environment jsdom
vitest --environment happy-dom

# Specify pool
vitest --pool threads
vitest --pool forks

# Limit workers
vitest --maxWorkers 4
```

### Output Options

```bash
# Reporters
vitest --reporter verbose
vitest --reporter json --outputFile results.json
vitest --reporter junit --outputFile junit.xml

# Enable UI
vitest --ui
vitest --ui --open

# Disable colors
vitest --no-color

# Silent mode
vitest --silent
```

### Coverage Options

```bash
# Enable coverage
vitest --coverage
vitest --coverage.enabled

# Coverage provider
vitest --coverage.provider v8
vitest --coverage.provider istanbul

# Coverage reporters
vitest --coverage.reporter text
vitest --coverage.reporter html
```

### Snapshot Options

```bash
# Update snapshots
vitest -u
vitest --update

# Update specific snapshot
vitest -t "snapshot test" -u
```

### Debugging Options

```bash
# Run specific shard
vitest --shard 1/3
vitest --shard 2/3
vitest --shard 3/3

# Bail on first failure
vitest --bail 1
vitest --bail 5

# Retry failed tests
vitest --retry 3

# Inspect for debugging
vitest --inspect
vitest --inspect-brk

# Disable file parallelism
vitest --no-file-parallelism

# Single thread/fork
vitest --pool threads --poolOptions.threads.singleThread
```

### Type Checking Options

```bash
# Enable type checking
vitest --typecheck

# Type checker
vitest --typecheck.checker tsc
vitest --typecheck.checker vue-tsc
```

---

## Watch Mode

### Interactive Commands

```
Press h to show help
Press f to filter by filename
Press t to filter by test name
Press q to quit
Press u to update snapshots
Press a to run all tests
Press Enter to trigger a test run
```

### Automatic Rerun

Watch mode automatically reruns tests when:

- Source files change
- Test files change
- Dependencies change

### Filtering in Watch Mode

```bash
# Start with filter
vitest --watch user

# Filter interactively
# Press 'f' to filter by filename
# Press 't' to filter by test name
```

### Watch Configuration

```typescript
// vitest.config.ts
export default defineConfig({
  test: {
    watch: true,

    // Files that trigger full rerun
    forceRerunTriggers: [
      '**/package.json',
      '**/vitest.config.ts',
      '**/tsconfig.json'
    ]
  }
});
```

---

## Best Practices

### Test Organization

```typescript
// Group related tests
describe('UserService', () => {
  describe('createUser', () => {
    it('creates user with valid data', () => {});
    it('throws error for invalid email', () => {});
    it('hashes password before saving', () => {});
  });

  describe('deleteUser', () => {
    it('deletes existing user', () => {});
    it('throws error for non-existent user', () => {});
  });
});
```

### Test Naming

```typescript
// Use descriptive names that explain expected behavior
it('should return empty array when no users exist', () => {});
it('should throw ValidationError when email is invalid', () => {});
it('should increment counter by 1 when increment button clicked', () => {});

// Avoid vague names
// Bad: it('works', () => {});
// Bad: it('test user', () => {});
```

### Arrange-Act-Assert Pattern

```typescript
it('calculates total with discount', () => {
  // Arrange
  const cart = new ShoppingCart();
  cart.addItem({ price: 100 });
  cart.addItem({ price: 50 });
  const discount = 0.1;

  // Act
  const total = cart.calculateTotal(discount);

  // Assert
  expect(total).toBe(135);
});
```

### Test Isolation

```typescript
describe('Counter', () => {
  let counter: Counter;

  // Fresh instance for each test
  beforeEach(() => {
    counter = new Counter();
  });

  it('starts at zero', () => {
    expect(counter.value).toBe(0);
  });

  it('increments by one', () => {
    counter.increment();
    expect(counter.value).toBe(1);
  });

  // Tests are independent - order doesn't matter
});
```

### Mock Management

```typescript
// Clear/reset mocks properly
afterEach(() => {
  vi.clearAllMocks();
});

// Or configure globally
// vitest.config.ts
export default defineConfig({
  test: {
    clearMocks: true,
    restoreMocks: true
  }
});
```

### Async Test Best Practices

```typescript
// Always await async operations
it('fetches user data', async () => {
  const user = await fetchUser(1);
  expect(user).toBeDefined();
});

// Use resolves/rejects for clarity
it('resolves with user', async () => {
  await expect(fetchUser(1)).resolves.toMatchObject({ id: 1 });
});

// Set appropriate timeouts
it('long operation', async () => {
  await expect(longOperation()).resolves.toBeDefined();
}, 30000);
```

### Snapshot Best Practices

```typescript
// Use inline snapshots for small values
it('formats date correctly', () => {
  expect(formatDate(new Date('2024-01-01'))).toMatchInlineSnapshot(
    `"January 1, 2024"`
  );
});

// Use property matchers for dynamic values
it('creates user with id', () => {
  expect(createUser('John')).toMatchSnapshot({
    id: expect.any(String),
    createdAt: expect.any(Date)
  });
});

// Review snapshot changes carefully
```

---

## Common Patterns

### Testing React Components

```typescript
import { render, screen, fireEvent } from '@testing-library/react';
import { describe, it, expect, vi } from 'vitest';
import Button from './Button';

describe('Button', () => {
  it('renders with text', () => {
    render(<Button>Click me</Button>);
    expect(screen.getByText('Click me')).toBeInTheDocument();
  });

  it('calls onClick when clicked', async () => {
    const handleClick = vi.fn();
    render(<Button onClick={handleClick}>Click</Button>);

    await fireEvent.click(screen.getByText('Click'));
    expect(handleClick).toHaveBeenCalledTimes(1);
  });

  it('is disabled when disabled prop is true', () => {
    render(<Button disabled>Click</Button>);
    expect(screen.getByText('Click')).toBeDisabled();
  });
});
```

### Testing Vue Components

```typescript
import { mount } from '@vue/test-utils';
import { describe, it, expect, vi } from 'vitest';
import Counter from './Counter.vue';

describe('Counter', () => {
  it('renders initial count', () => {
    const wrapper = mount(Counter, {
      props: { initialCount: 5 }
    });
    expect(wrapper.text()).toContain('5');
  });

  it('increments count on button click', async () => {
    const wrapper = mount(Counter);
    await wrapper.find('button').trigger('click');
    expect(wrapper.text()).toContain('1');
  });

  it('emits update event', async () => {
    const wrapper = mount(Counter);
    await wrapper.find('button').trigger('click');
    expect(wrapper.emitted('update')).toHaveLength(1);
  });
});
```

### Testing Custom Hooks

```typescript
import { renderHook, act } from '@testing-library/react';
import { describe, it, expect } from 'vitest';
import { useCounter } from './useCounter';

describe('useCounter', () => {
  it('initializes with default value', () => {
    const { result } = renderHook(() => useCounter());
    expect(result.current.count).toBe(0);
  });

  it('initializes with provided value', () => {
    const { result } = renderHook(() => useCounter(10));
    expect(result.current.count).toBe(10);
  });

  it('increments counter', () => {
    const { result } = renderHook(() => useCounter());

    act(() => {
      result.current.increment();
    });

    expect(result.current.count).toBe(1);
  });

  it('decrements counter', () => {
    const { result } = renderHook(() => useCounter(5));

    act(() => {
      result.current.decrement();
    });

    expect(result.current.count).toBe(4);
  });
});
```

### Testing API Calls

```typescript
import { describe, it, expect, vi, beforeEach } from 'vitest';
import { fetchUsers, createUser } from './api';

// Mock fetch globally
const mockFetch = vi.fn();
vi.stubGlobal('fetch', mockFetch);

describe('API', () => {
  beforeEach(() => {
    mockFetch.mockReset();
  });

  describe('fetchUsers', () => {
    it('returns users on success', async () => {
      const users = [{ id: 1, name: 'John' }];
      mockFetch.mockResolvedValue({
        ok: true,
        json: () => Promise.resolve(users)
      });

      const result = await fetchUsers();
      expect(result).toEqual(users);
      expect(mockFetch).toHaveBeenCalledWith('/api/users');
    });

    it('throws error on failure', async () => {
      mockFetch.mockResolvedValue({
        ok: false,
        status: 500
      });

      await expect(fetchUsers()).rejects.toThrow('Failed to fetch users');
    });
  });

  describe('createUser', () => {
    it('sends POST request with user data', async () => {
      const newUser = { name: 'Jane' };
      const createdUser = { id: 2, ...newUser };

      mockFetch.mockResolvedValue({
        ok: true,
        json: () => Promise.resolve(createdUser)
      });

      const result = await createUser(newUser);

      expect(mockFetch).toHaveBeenCalledWith('/api/users', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify(newUser)
      });
      expect(result).toEqual(createdUser);
    });
  });
});
```

### Testing Error Handling

```typescript
import { describe, it, expect } from 'vitest';
import { validateEmail, validateAge } from './validators';

describe('Validators', () => {
  describe('validateEmail', () => {
    it('accepts valid email', () => {
      expect(() => validateEmail('user@example.com')).not.toThrow();
    });

    it('throws for invalid email', () => {
      expect(() => validateEmail('invalid')).toThrow('Invalid email format');
    });

    it('throws specific error type', () => {
      expect(() => validateEmail('invalid')).toThrow(ValidationError);
    });
  });

  describe('validateAge', () => {
    it('accepts valid age', () => {
      expect(() => validateAge(25)).not.toThrow();
    });

    it.each([
      [-1, 'Age cannot be negative'],
      [150, 'Age must be less than 120'],
      [NaN, 'Age must be a number']
    ])('throws for age %s: %s', (age, message) => {
      expect(() => validateAge(age)).toThrow(message);
    });
  });
});
```

### Testing with Fixtures

```typescript
import { test as baseTest } from 'vitest';
import { createTestDatabase, createTestUser } from './test-utils';

const test = baseTest.extend({
  db: async ({}, use) => {
    const db = await createTestDatabase();
    await use(db);
    await db.cleanup();
  },

  user: async ({ db }, use) => {
    const user = await createTestUser(db);
    await use(user);
  },

  authenticatedClient: async ({ user }, use) => {
    const client = createApiClient();
    await client.authenticate(user);
    await use(client);
  }
});

test('authenticated user can access profile', async ({ authenticatedClient }) => {
  const profile = await authenticatedClient.getProfile();
  expect(profile).toBeDefined();
  expect(profile.email).toBeDefined();
});

test('user can update settings', async ({ authenticatedClient, db }) => {
  await authenticatedClient.updateSettings({ theme: 'dark' });

  const settings = await db.getUserSettings(authenticatedClient.userId);
  expect(settings.theme).toBe('dark');
});
```

### Testing Events and Callbacks

```typescript
import { describe, it, expect, vi } from 'vitest';
import { EventEmitter } from './EventEmitter';

describe('EventEmitter', () => {
  it('calls listener when event is emitted', () => {
    const emitter = new EventEmitter();
    const listener = vi.fn();

    emitter.on('test', listener);
    emitter.emit('test', 'data');

    expect(listener).toHaveBeenCalledWith('data');
  });

  it('calls multiple listeners', () => {
    const emitter = new EventEmitter();
    const listener1 = vi.fn();
    const listener2 = vi.fn();

    emitter.on('test', listener1);
    emitter.on('test', listener2);
    emitter.emit('test');

    expect(listener1).toHaveBeenCalled();
    expect(listener2).toHaveBeenCalled();
  });

  it('removes listener with off', () => {
    const emitter = new EventEmitter();
    const listener = vi.fn();

    emitter.on('test', listener);
    emitter.off('test', listener);
    emitter.emit('test');

    expect(listener).not.toHaveBeenCalled();
  });

  it('once listener fires only once', () => {
    const emitter = new EventEmitter();
    const listener = vi.fn();

    emitter.once('test', listener);
    emitter.emit('test');
    emitter.emit('test');

    expect(listener).toHaveBeenCalledTimes(1);
  });
});
```

### Testing Classes

```typescript
import { describe, it, expect, vi, beforeEach } from 'vitest';
import { ShoppingCart, Product } from './ShoppingCart';

describe('ShoppingCart', () => {
  let cart: ShoppingCart;

  beforeEach(() => {
    cart = new ShoppingCart();
  });

  describe('addItem', () => {
    it('adds product to cart', () => {
      const product: Product = { id: '1', name: 'Apple', price: 1.5 };
      cart.addItem(product, 2);

      expect(cart.items).toHaveLength(1);
      expect(cart.items[0]).toEqual({ product, quantity: 2 });
    });

    it('increases quantity if product already in cart', () => {
      const product: Product = { id: '1', name: 'Apple', price: 1.5 };
      cart.addItem(product, 2);
      cart.addItem(product, 3);

      expect(cart.items).toHaveLength(1);
      expect(cart.items[0].quantity).toBe(5);
    });
  });

  describe('removeItem', () => {
    it('removes product from cart', () => {
      const product: Product = { id: '1', name: 'Apple', price: 1.5 };
      cart.addItem(product, 2);
      cart.removeItem('1');

      expect(cart.items).toHaveLength(0);
    });
  });

  describe('getTotal', () => {
    it('calculates total correctly', () => {
      cart.addItem({ id: '1', name: 'Apple', price: 1.5 }, 2);
      cart.addItem({ id: '2', name: 'Banana', price: 0.75 }, 4);

      expect(cart.getTotal()).toBe(6); // (1.5 * 2) + (0.75 * 4)
    });

    it('applies discount correctly', () => {
      cart.addItem({ id: '1', name: 'Apple', price: 10 }, 1);

      expect(cart.getTotal(0.1)).toBe(9); // 10 - 10%
    });
  });
});
```

---

## Quick Reference

### Import Statement

```typescript
import {
  describe,
  it,
  test,
  expect,
  vi,
  beforeAll,
  afterAll,
  beforeEach,
  afterEach
} from 'vitest';
```

### Common Matchers

| Matcher | Description |
|---------|-------------|
| `toBe(value)` | Strict equality |
| `toEqual(value)` | Deep equality |
| `toBeTruthy()` | Truthy value |
| `toBeFalsy()` | Falsy value |
| `toBeNull()` | Is null |
| `toBeUndefined()` | Is undefined |
| `toBeDefined()` | Is defined |
| `toContain(item)` | Array/string contains |
| `toHaveLength(n)` | Length check |
| `toHaveProperty(key)` | Object has property |
| `toThrow()` | Function throws |
| `toMatch(regex)` | String matches pattern |

### Test Modifiers

| Modifier | Description |
|----------|-------------|
| `.skip` | Skip test |
| `.only` | Run only this test |
| `.todo` | Placeholder test |
| `.concurrent` | Run in parallel |
| `.sequential` | Run sequentially |
| `.each([])` | Parameterized test |
| `.fails` | Expected to fail |
| `.skipIf(cond)` | Skip if condition |
| `.runIf(cond)` | Run if condition |

### Mock Functions

| Method | Description |
|--------|-------------|
| `vi.fn()` | Create mock function |
| `vi.spyOn(obj, 'method')` | Spy on method |
| `vi.mock('module')` | Mock module |
| `mockFn.mockReturnValue(val)` | Set return value |
| `mockFn.mockResolvedValue(val)` | Set resolved value |
| `mockFn.mockImplementation(fn)` | Set implementation |
| `mockFn.mockClear()` | Clear call history |
| `mockFn.mockReset()` | Reset to empty function |
| `mockFn.mockRestore()` | Restore original |
