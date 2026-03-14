# Jest Advanced Patterns

Advanced testing patterns, integration testing, and best practices for Jest.

**Official Documentation:** https://jestjs.io/docs/getting-started

---

## Table of Contents

1. [Testing Async Code](#testing-async-code)
2. [Snapshot Testing](#snapshot-testing)
3. [Custom Matchers](#custom-matchers)
4. [Testing React Components](#testing-react-components)
5. [Integration Testing](#integration-testing)
6. [Test Isolation](#test-isolation)
7. [Performance Testing](#performance-testing)
8. [Configuration Patterns](#configuration-patterns)
9. [Debugging Tests](#debugging-tests)

---

## Testing Async Code

### Promises

```typescript
// Using async/await (recommended)
test('async/await pattern', async () => {
  const data = await fetchData();
  expect(data).toBe('peanut butter');
});

// Using .resolves/.rejects
test('resolves pattern', async () => {
  await expect(fetchData()).resolves.toBe('peanut butter');
  await expect(fetchWithError()).rejects.toThrow('error');
});

// Using done callback (legacy)
test('done callback', (done) => {
  fetchData().then((data) => {
    expect(data).toBe('peanut butter');
    done();
  });
});

// Error handling
test('handles errors', async () => {
  expect.assertions(1);
  try {
    await fetchWithError();
  } catch (e) {
    expect(e.message).toMatch('error');
  }
});
```

### Testing Async Iterators

```typescript
async function* generateNumbers(n: number) {
  for (let i = 0; i < n; i++) {
    yield i;
  }
}

test('async iterators', async () => {
  const results: number[] = [];

  for await (const num of generateNumbers(3)) {
    results.push(num);
  }

  expect(results).toEqual([0, 1, 2]);
});
```

### Testing Streams

```typescript
import { Readable } from 'stream';

test('readable stream', async () => {
  const chunks: string[] = [];

  const stream = Readable.from(['hello', ' ', 'world']);

  for await (const chunk of stream) {
    chunks.push(chunk);
  }

  expect(chunks.join('')).toBe('hello world');
});

// Testing event emitters
test('event emitter', (done) => {
  const emitter = new EventEmitter();

  emitter.on('data', (data) => {
    expect(data).toBe('test');
    done();
  });

  emitter.emit('data', 'test');
});
```

### Concurrent Tests

```typescript
// Run tests concurrently
test.concurrent('concurrent test 1', async () => {
  await sleep(100);
  expect(1).toBe(1);
});

test.concurrent('concurrent test 2', async () => {
  await sleep(100);
  expect(2).toBe(2);
});

// Limit concurrency
describe.concurrent('limited concurrency', () => {
  // Tests run concurrently within this describe
});
```

---

## Snapshot Testing

### Basic Snapshots

```typescript
// Component snapshot
test('renders correctly', () => {
  const tree = render(<Button label="Click me" />);
  expect(tree).toMatchSnapshot();
});

// Inline snapshot (auto-generated)
test('inline snapshot', () => {
  expect({ name: 'John', age: 30 }).toMatchInlineSnapshot(`
    {
      "age": 30,
      "name": "John",
    }
  `);
});

// Property matchers for dynamic values
test('with dynamic values', () => {
  const user = {
    id: expect.any(Number),
    name: 'John',
    createdAt: expect.any(Date),
  };

  expect(createUser('John')).toMatchSnapshot({
    id: expect.any(Number),
    createdAt: expect.any(Date),
  });
});
```

### Custom Serializers

```typescript
// jest.config.js
module.exports = {
  snapshotSerializers: ['enzyme-to-json/serializer'],
};

// Custom serializer
expect.addSnapshotSerializer({
  test: (val) => val && val.hasOwnProperty('className'),
  print: (val, serialize) => {
    return `ClassName<${val.className}>`;
  },
});
```

### Snapshot Best Practices

```typescript
// Named snapshots for clarity
test('user profile', () => {
  expect(render(<UserProfile />)).toMatchSnapshot('logged in user');
});

// Focused snapshots (avoid large snapshots)
test('button text only', () => {
  const { getByRole } = render(<Button>Submit</Button>);
  expect(getByRole('button').textContent).toMatchSnapshot();
});

// Update snapshots: jest -u or jest --updateSnapshot
```

---

## Custom Matchers

### Basic Custom Matcher

```typescript
// setupTests.ts
expect.extend({
  toBeWithinRange(received: number, floor: number, ceiling: number) {
    const pass = received >= floor && received <= ceiling;

    if (pass) {
      return {
        message: () =>
          `expected ${received} not to be within range ${floor} - ${ceiling}`,
        pass: true,
      };
    } else {
      return {
        message: () =>
          `expected ${received} to be within range ${floor} - ${ceiling}`,
        pass: false,
      };
    }
  },
});

// TypeScript declaration
declare global {
  namespace jest {
    interface Matchers<R> {
      toBeWithinRange(floor: number, ceiling: number): R;
    }
  }
}

// Usage
test('custom matcher', () => {
  expect(100).toBeWithinRange(90, 110);
  expect(101).not.toBeWithinRange(0, 100);
});
```

### Async Custom Matcher

```typescript
expect.extend({
  async toBeValidUser(received: { id: string }) {
    const user = await fetchUser(received.id);
    const pass = user !== null && user.active === true;

    return {
      message: () =>
        pass
          ? `expected user ${received.id} not to be valid`
          : `expected user ${received.id} to be valid`,
      pass,
    };
  },
});

test('async matcher', async () => {
  await expect({ id: '123' }).toBeValidUser();
});
```

### DOM Custom Matchers

```typescript
expect.extend({
  toHaveTextContent(received: HTMLElement, expected: string) {
    const textContent = received.textContent || '';
    const pass = textContent.includes(expected);

    return {
      message: () =>
        pass
          ? `expected element not to have text content "${expected}"`
          : `expected element to have text content "${expected}", but got "${textContent}"`,
      pass,
    };
  },

  toBeVisible(received: HTMLElement) {
    const style = window.getComputedStyle(received);
    const pass =
      style.display !== 'none' &&
      style.visibility !== 'hidden' &&
      style.opacity !== '0';

    return {
      message: () =>
        pass
          ? 'expected element not to be visible'
          : 'expected element to be visible',
      pass,
    };
  },
});
```

---

## Testing React Components

### Component Testing with Testing Library

```typescript
import { render, screen, fireEvent, waitFor } from '@testing-library/react';
import userEvent from '@testing-library/user-event';

describe('LoginForm', () => {
  const mockOnSubmit = jest.fn();

  beforeEach(() => {
    mockOnSubmit.mockClear();
  });

  it('submits form with valid data', async () => {
    const user = userEvent.setup();

    render(<LoginForm onSubmit={mockOnSubmit} />);

    await user.type(screen.getByLabelText(/email/i), 'test@example.com');
    await user.type(screen.getByLabelText(/password/i), 'password123');
    await user.click(screen.getByRole('button', { name: /submit/i }));

    expect(mockOnSubmit).toHaveBeenCalledWith({
      email: 'test@example.com',
      password: 'password123',
    });
  });

  it('shows validation errors', async () => {
    const user = userEvent.setup();

    render(<LoginForm onSubmit={mockOnSubmit} />);

    await user.click(screen.getByRole('button', { name: /submit/i }));

    expect(await screen.findByText(/email is required/i)).toBeInTheDocument();
    expect(mockOnSubmit).not.toHaveBeenCalled();
  });

  it('disables submit while loading', async () => {
    render(<LoginForm onSubmit={mockOnSubmit} loading />);

    expect(screen.getByRole('button', { name: /submit/i })).toBeDisabled();
  });
});
```

### Testing Hooks

```typescript
import { renderHook, act } from '@testing-library/react';

describe('useCounter', () => {
  it('initializes with default value', () => {
    const { result } = renderHook(() => useCounter());

    expect(result.current.count).toBe(0);
  });

  it('initializes with custom value', () => {
    const { result } = renderHook(() => useCounter(10));

    expect(result.current.count).toBe(10);
  });

  it('increments count', () => {
    const { result } = renderHook(() => useCounter());

    act(() => {
      result.current.increment();
    });

    expect(result.current.count).toBe(1);
  });

  it('updates when props change', () => {
    const { result, rerender } = renderHook(
      ({ initial }) => useCounter(initial),
      { initialProps: { initial: 0 } }
    );

    expect(result.current.count).toBe(0);

    rerender({ initial: 10 });

    expect(result.current.count).toBe(10);
  });
});
```

### Testing Context

```typescript
import { render, screen } from '@testing-library/react';

const mockUser = { id: '1', name: 'John' };

const wrapper = ({ children }: { children: React.ReactNode }) => (
  <UserContext.Provider value={{ user: mockUser, setUser: jest.fn() }}>
    {children}
  </UserContext.Provider>
);

describe('UserProfile', () => {
  it('displays user name from context', () => {
    render(<UserProfile />, { wrapper });

    expect(screen.getByText('John')).toBeInTheDocument();
  });
});

// Reusable wrapper factory
const createWrapper = (contextValue: Partial<UserContextType>) => {
  return ({ children }: { children: React.ReactNode }) => (
    <UserContext.Provider value={{ ...defaultContext, ...contextValue }}>
      {children}
    </UserContext.Provider>
  );
};

it('handles missing user', () => {
  render(<UserProfile />, {
    wrapper: createWrapper({ user: null }),
  });

  expect(screen.getByText('Not logged in')).toBeInTheDocument();
});
```

### Testing Portals and Modals

```typescript
describe('Modal', () => {
  beforeEach(() => {
    // Create portal root
    const portalRoot = document.createElement('div');
    portalRoot.setAttribute('id', 'modal-root');
    document.body.appendChild(portalRoot);
  });

  afterEach(() => {
    // Cleanup
    const portalRoot = document.getElementById('modal-root');
    if (portalRoot) {
      document.body.removeChild(portalRoot);
    }
  });

  it('renders in portal', () => {
    render(<Modal isOpen>Modal content</Modal>);

    const modalRoot = document.getElementById('modal-root');
    expect(modalRoot).toHaveTextContent('Modal content');
  });

  it('closes on escape key', async () => {
    const onClose = jest.fn();
    const user = userEvent.setup();

    render(
      <Modal isOpen onClose={onClose}>
        Content
      </Modal>
    );

    await user.keyboard('{Escape}');

    expect(onClose).toHaveBeenCalled();
  });
});
```

---

## Integration Testing

### API Integration Tests

```typescript
import { setupServer } from 'msw/node';
import { rest } from 'msw';

const server = setupServer(
  rest.get('/api/users', (req, res, ctx) => {
    return res(ctx.json([{ id: 1, name: 'John' }]));
  }),

  rest.post('/api/users', async (req, res, ctx) => {
    const { name } = await req.json();
    return res(ctx.json({ id: 2, name }));
  }),

  rest.get('/api/users/:id', (req, res, ctx) => {
    const { id } = req.params;
    return res(ctx.json({ id: Number(id), name: `User ${id}` }));
  })
);

beforeAll(() => server.listen());
afterEach(() => server.resetHandlers());
afterAll(() => server.close());

describe('User API', () => {
  it('fetches users', async () => {
    const users = await api.getUsers();

    expect(users).toEqual([{ id: 1, name: 'John' }]);
  });

  it('creates user', async () => {
    const user = await api.createUser({ name: 'Jane' });

    expect(user).toEqual({ id: 2, name: 'Jane' });
  });

  it('handles errors', async () => {
    server.use(
      rest.get('/api/users', (req, res, ctx) => {
        return res(ctx.status(500), ctx.json({ error: 'Server error' }));
      })
    );

    await expect(api.getUsers()).rejects.toThrow('Server error');
  });
});
```

### Database Integration Tests

```typescript
import { PrismaClient } from '@prisma/client';

const prisma = new PrismaClient();

beforeAll(async () => {
  // Connect to test database
  await prisma.$connect();
});

beforeEach(async () => {
  // Clean database before each test
  await prisma.user.deleteMany();
  await prisma.post.deleteMany();
});

afterAll(async () => {
  await prisma.$disconnect();
});

describe('UserService', () => {
  it('creates user', async () => {
    const user = await userService.create({
      email: 'test@example.com',
      name: 'Test User',
    });

    expect(user.id).toBeDefined();
    expect(user.email).toBe('test@example.com');

    // Verify in database
    const dbUser = await prisma.user.findUnique({
      where: { id: user.id },
    });
    expect(dbUser).not.toBeNull();
  });

  it('finds user with posts', async () => {
    // Setup
    const user = await prisma.user.create({
      data: {
        email: 'test@example.com',
        name: 'Test',
        posts: {
          create: [{ title: 'Post 1' }, { title: 'Post 2' }],
        },
      },
    });

    // Test
    const result = await userService.findWithPosts(user.id);

    expect(result.posts).toHaveLength(2);
  });
});
```

### Full Stack Integration

```typescript
import supertest from 'supertest';
import { app } from '../app';

const request = supertest(app);

describe('POST /api/auth/login', () => {
  beforeEach(async () => {
    // Seed test user
    await seedTestUser({
      email: 'test@example.com',
      password: 'hashedPassword',
    });
  });

  it('returns token on valid credentials', async () => {
    const response = await request
      .post('/api/auth/login')
      .send({ email: 'test@example.com', password: 'password' })
      .expect(200);

    expect(response.body.token).toBeDefined();
    expect(response.body.user.email).toBe('test@example.com');
  });

  it('returns 401 on invalid credentials', async () => {
    const response = await request
      .post('/api/auth/login')
      .send({ email: 'test@example.com', password: 'wrong' })
      .expect(401);

    expect(response.body.error).toBe('Invalid credentials');
  });

  it('validates input', async () => {
    const response = await request
      .post('/api/auth/login')
      .send({ email: 'invalid' })
      .expect(400);

    expect(response.body.errors).toBeDefined();
  });
});

describe('Protected routes', () => {
  let token: string;

  beforeEach(async () => {
    const loginResponse = await request
      .post('/api/auth/login')
      .send({ email: 'test@example.com', password: 'password' });

    token = loginResponse.body.token;
  });

  it('allows access with valid token', async () => {
    await request
      .get('/api/profile')
      .set('Authorization', `Bearer ${token}`)
      .expect(200);
  });

  it('rejects without token', async () => {
    await request.get('/api/profile').expect(401);
  });
});
```

---

## Test Isolation

### Isolated Module State

```typescript
// Reset module state between tests
beforeEach(() => {
  jest.resetModules();
});

test('module isolation', () => {
  const counter = require('./counter');
  counter.increment();
  expect(counter.getCount()).toBe(1);
});

test('fresh module state', () => {
  const counter = require('./counter');
  // Starts fresh due to resetModules
  expect(counter.getCount()).toBe(0);
});
```

### Isolated Environment

```typescript
// Use separate test environments
// In package.json or jest.config.js

// For DOM testing
testEnvironment: 'jsdom'

// For Node testing
testEnvironment: 'node'

// Custom environment per file
/**
 * @jest-environment jsdom
 */

/**
 * @jest-environment node
 */
```

### Global State Management

```typescript
// setupFilesAfterEnv.ts
import { cleanup } from '@testing-library/react';

// Cleanup after each test
afterEach(() => {
  cleanup();
  jest.clearAllMocks();
  localStorage.clear();
  sessionStorage.clear();
});

// Reset modules after each test file
afterAll(() => {
  jest.resetModules();
});

// Global test timeout
jest.setTimeout(10000);
```

---

## Performance Testing

### Test Execution Time

```typescript
test('performance', () => {
  const start = performance.now();

  // Operation to measure
  heavyComputation();

  const end = performance.now();
  const duration = end - start;

  expect(duration).toBeLessThan(100); // Should complete in 100ms
});

// Using Jest's built-in timer
test('with timeout', async () => {
  await expect(
    Promise.race([
      slowOperation(),
      new Promise((_, reject) =>
        setTimeout(() => reject(new Error('Timeout')), 1000)
      ),
    ])
  ).resolves.toBeDefined();
});
```

### Memory Testing

```typescript
test('memory usage', () => {
  const used = process.memoryUsage();

  // Operation
  createLargeDataStructure();

  const usedAfter = process.memoryUsage();
  const heapGrowth = usedAfter.heapUsed - used.heapUsed;

  expect(heapGrowth).toBeLessThan(50 * 1024 * 1024); // Less than 50MB
});
```

### Benchmark Tests

```typescript
// jest.config.js - separate config for benchmarks
module.exports = {
  testMatch: ['**/*.bench.ts'],
  testTimeout: 60000,
};

// algorithm.bench.ts
describe('sorting benchmarks', () => {
  const sizes = [100, 1000, 10000];

  sizes.forEach((size) => {
    test(`sorts ${size} items`, () => {
      const data = Array.from({ length: size }, () => Math.random());

      const iterations = 100;
      const times: number[] = [];

      for (let i = 0; i < iterations; i++) {
        const copy = [...data];
        const start = performance.now();
        copy.sort((a, b) => a - b);
        times.push(performance.now() - start);
      }

      const avg = times.reduce((a, b) => a + b) / times.length;
      console.log(`Average time for ${size} items: ${avg.toFixed(2)}ms`);

      expect(avg).toBeLessThan(size / 10); // Rough expectation
    });
  });
});
```

---

## Configuration Patterns

### Project-Specific Configs

```javascript
// jest.config.js
module.exports = {
  projects: [
    {
      displayName: 'unit',
      testMatch: ['<rootDir>/src/**/*.test.ts'],
      testEnvironment: 'node',
    },
    {
      displayName: 'integration',
      testMatch: ['<rootDir>/tests/integration/**/*.test.ts'],
      testEnvironment: 'node',
      setupFilesAfterEnv: ['<rootDir>/tests/integration/setup.ts'],
    },
    {
      displayName: 'e2e',
      testMatch: ['<rootDir>/tests/e2e/**/*.test.ts'],
      testEnvironment: 'jsdom',
      setupFilesAfterEnv: ['<rootDir>/tests/e2e/setup.ts'],
    },
  ],
};
```

### Coverage Configuration

```javascript
module.exports = {
  collectCoverage: true,
  collectCoverageFrom: [
    'src/**/*.{ts,tsx}',
    '!src/**/*.d.ts',
    '!src/**/*.stories.{ts,tsx}',
    '!src/**/index.ts',
  ],
  coverageThreshold: {
    global: {
      branches: 80,
      functions: 80,
      lines: 80,
      statements: 80,
    },
    './src/critical/': {
      branches: 100,
      functions: 100,
      lines: 100,
      statements: 100,
    },
  },
  coverageReporters: ['text', 'lcov', 'html'],
};
```

### Module Resolution

```javascript
module.exports = {
  moduleNameMapper: {
    // Path aliases
    '^@/(.*)$': '<rootDir>/src/$1',
    '^@components/(.*)$': '<rootDir>/src/components/$1',

    // Static assets
    '\\.(css|less|scss|sass)$': 'identity-obj-proxy',
    '\\.(jpg|jpeg|png|gif|svg)$': '<rootDir>/__mocks__/fileMock.js',
  },

  moduleDirectories: ['node_modules', 'src'],

  transform: {
    '^.+\\.(ts|tsx)$': 'ts-jest',
  },
};
```

### Test Reporters

```javascript
module.exports = {
  reporters: [
    'default',
    [
      'jest-junit',
      {
        outputDirectory: 'reports',
        outputName: 'junit.xml',
      },
    ],
    [
      'jest-html-reporter',
      {
        pageTitle: 'Test Report',
        outputPath: 'reports/test-report.html',
      },
    ],
  ],
};
```

---

## Debugging Tests

### Debug Mode

```bash
# Run with Node debugger
node --inspect-brk node_modules/.bin/jest --runInBand

# VS Code launch.json
{
  "type": "node",
  "request": "launch",
  "name": "Debug Jest Tests",
  "runtimeExecutable": "node",
  "runtimeArgs": [
    "--inspect-brk",
    "${workspaceRoot}/node_modules/.bin/jest",
    "--runInBand"
  ],
  "console": "integratedTerminal"
}
```

### Debugging Async Issues

```typescript
// Increase timeout for debugging
jest.setTimeout(30000);

test('debug async', async () => {
  debugger; // Breakpoint

  const result = await complexAsyncOperation();

  console.log('Result:', JSON.stringify(result, null, 2));

  expect(result).toBeDefined();
});

// Check for unhandled promises
test('unhandled promise detection', async () => {
  const unhandledPromises: Promise<unknown>[] = [];

  process.on('unhandledRejection', (promise) => {
    unhandledPromises.push(promise as Promise<unknown>);
  });

  await runCode();

  expect(unhandledPromises).toHaveLength(0);
});
```

### Test Filtering

```bash
# Run specific test file
jest path/to/file.test.ts

# Run tests matching pattern
jest --testNamePattern="should handle"

# Run only changed tests
jest --onlyChanged

# Run tests related to changed files
jest --findRelatedTests path/to/file.ts

# Run with verbose output
jest --verbose

# Show individual test results
jest --expand
```

### Logging in Tests

```typescript
// Debug specific test
test.only('debug this test', () => {
  // Only this test runs
});

// Skip test
test.skip('skip this', () => {
  // Skipped
});

// Todo test
test.todo('implement this test');

// Conditional skip
const skipCondition = process.env.CI === 'true';
(skipCondition ? test.skip : test)('conditional test', () => {
  // Runs based on condition
});
```

---

## Quick Reference

### Test Lifecycle

| Hook | Scope | When |
|------|-------|------|
| `beforeAll` | File | Once before all tests |
| `beforeEach` | File | Before each test |
| `afterEach` | File | After each test |
| `afterAll` | File | Once after all tests |

### Common Matchers

| Matcher | Use Case |
|---------|----------|
| `toBe` | Strict equality |
| `toEqual` | Deep equality |
| `toMatchObject` | Partial object match |
| `toContain` | Array/string contains |
| `toHaveBeenCalled` | Mock was called |
| `toThrow` | Function throws |
| `toMatchSnapshot` | Snapshot match |

### Test Organization

```typescript
describe('ComponentName', () => {
  describe('when condition A', () => {
    beforeEach(() => {
      // Setup for condition A
    });

    it('should behave X', () => {});
    it('should behave Y', () => {});
  });

  describe('when condition B', () => {
    beforeEach(() => {
      // Setup for condition B
    });

    it('should behave Z', () => {});
  });
});
```

### CLI Commands

```bash
# Watch mode
jest --watch

# Coverage
jest --coverage

# Update snapshots
jest -u

# Run in band (sequential)
jest --runInBand

# Clear cache
jest --clearCache

# Show config
jest --showConfig
```
