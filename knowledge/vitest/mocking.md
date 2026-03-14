# Vitest Mocking - Comprehensive Guide

This comprehensive guide covers all aspects of mocking in Vitest, from basic function
mocks to advanced module mocking patterns. Mocking is essential for isolating units
of code during testing and controlling external dependencies.

---

## Table of Contents

1. [vi.fn() - Creating Mock Functions](#1-vifn---creating-mock-functions)
2. [Mock Function Properties](#2-mock-function-properties)
3. [Mock Return Values](#3-mock-return-values)
4. [Mock Implementations](#4-mock-implementations)
5. [vi.spyOn() - Spying on Methods](#5-vispyon---spying-on-methods)
6. [vi.mock() - Module Mocking](#6-vimock---module-mocking)
7. [vi.doMock() - Inline Module Mocking](#7-vidomock---inline-module-mocking)
8. [vi.importActual() and vi.importMock()](#8-viimportactual-and-viimportmock)
9. [__mocks__ Directory](#9-__mocks__-directory)
10. [Mocking ES Modules](#10-mocking-es-modules)
11. [Mocking Classes](#11-mocking-classes)
12. [Mocking Timers](#12-mocking-timers)
13. [Mocking Global Objects](#13-mocking-global-objects)
14. [vi.stubGlobal() and vi.stubEnv()](#14-vistubglobal-and-vistubenv)
15. [Resetting Mocks](#15-resetting-mocks)
16. [Partial Mocking](#16-partial-mocking)
17. [Testing React Components with Mocks](#17-testing-react-components-with-mocks)
18. [Best Practices](#18-best-practices)

---

## 1. vi.fn() - Creating Mock Functions

The `vi.fn()` function creates a mock function that tracks calls, arguments, and
return values. It's the foundation of mocking in Vitest.

### Basic Mock Function Creation

```typescript
import { vi, expect, it, describe } from 'vitest';

describe('vi.fn() basics', () => {
  it('creates a mock function with no implementation', () => {
    const mockFn = vi.fn();

    // Call the mock
    mockFn();
    mockFn('arg1', 'arg2');

    // By default, returns undefined
    expect(mockFn()).toBeUndefined();

    // Track that it was called
    expect(mockFn).toHaveBeenCalled();
    expect(mockFn).toHaveBeenCalledTimes(3);
  });

  it('creates a mock function with an implementation', () => {
    const mockFn = vi.fn((a: number, b: number) => a + b);

    expect(mockFn(2, 3)).toBe(5);
    expect(mockFn(10, 20)).toBe(30);
    expect(mockFn).toHaveBeenCalledTimes(2);
  });

  it('creates a typed mock function', () => {
    // Define the function type
    type Calculator = (a: number, b: number) => number;

    // Create typed mock
    const mockCalculator = vi.fn<Calculator>();
    mockCalculator.mockReturnValue(42);

    expect(mockCalculator(1, 2)).toBe(42);
  });
});
```

### Mock Function with Generic Types

```typescript
import { vi, expect, it } from 'vitest';

interface User {
  id: number;
  name: string;
  email: string;
}

// Mock function returning a User
const getUser = vi.fn<(id: number) => User>();

getUser.mockReturnValue({
  id: 1,
  name: 'John Doe',
  email: 'john@example.com',
});

it('returns typed user data', () => {
  const user = getUser(1);
  expect(user.name).toBe('John Doe');
  expect(getUser).toHaveBeenCalledWith(1);
});
```

### Creating Mock Functions for Callbacks

```typescript
import { vi, expect, it } from 'vitest';

// Simulating an event handler
const onClick = vi.fn<(event: { target: HTMLElement }) => void>();

// Simulating an async callback
const onDataLoaded = vi.fn<(data: string[]) => Promise<void>>();

it('tracks callback invocations', () => {
  const button = { tagName: 'BUTTON' } as HTMLElement;
  onClick({ target: button });

  expect(onClick).toHaveBeenCalledWith({ target: button });
});

it('handles async callbacks', async () => {
  onDataLoaded.mockResolvedValue(undefined);

  await onDataLoaded(['item1', 'item2']);
  expect(onDataLoaded).toHaveBeenCalledWith(['item1', 'item2']);
});
```

---

## 2. Mock Function Properties

Mock functions in Vitest have several properties that track how they were called
and what they returned.

### The mock Property

Every mock function has a `mock` property containing call information:

```typescript
import { vi, expect, it, describe, beforeEach } from 'vitest';

describe('mock function properties', () => {
  const mockFn = vi.fn((x: number) => x * 2);

  beforeEach(() => {
    mockFn.mockClear();
  });

  it('tracks calls in mock.calls', () => {
    mockFn(1);
    mockFn(2, 'extra');
    mockFn(3);

    // mock.calls is an array of argument arrays
    expect(mockFn.mock.calls).toEqual([
      [1],
      [2, 'extra'],
      [3],
    ]);

    // Access specific call arguments
    expect(mockFn.mock.calls[0][0]).toBe(1);
    expect(mockFn.mock.calls[1]).toEqual([2, 'extra']);
  });

  it('tracks results in mock.results', () => {
    mockFn(5);
    mockFn(10);

    // mock.results contains return values and types
    expect(mockFn.mock.results).toEqual([
      { type: 'return', value: 10 },
      { type: 'return', value: 20 },
    ]);
  });

  it('tracks thrown errors in mock.results', () => {
    const throwingMock = vi.fn(() => {
      throw new Error('Test error');
    });

    expect(() => throwingMock()).toThrow('Test error');

    expect(throwingMock.mock.results[0]).toEqual({
      type: 'throw',
      value: expect.any(Error),
    });
  });
});
```

### mock.calls Property

```typescript
import { vi, expect, it } from 'vitest';

const mockFn = vi.fn();

it('provides detailed call information', () => {
  mockFn('first');
  mockFn('second', { data: true });
  mockFn();

  // Number of calls
  expect(mockFn.mock.calls.length).toBe(3);

  // First call arguments
  expect(mockFn.mock.calls[0]).toEqual(['first']);

  // Second call arguments
  expect(mockFn.mock.calls[1]).toEqual(['second', { data: true }]);

  // Third call (no arguments)
  expect(mockFn.mock.calls[2]).toEqual([]);

  // Last call shorthand
  expect(mockFn.mock.lastCall).toEqual([]);
});
```

### mock.results Property

```typescript
import { vi, expect, it } from 'vitest';

it('tracks all return values', () => {
  const mockFn = vi.fn()
    .mockReturnValueOnce('first')
    .mockReturnValueOnce('second')
    .mockReturnValue('default');

  mockFn();
  mockFn();
  mockFn();
  mockFn();

  expect(mockFn.mock.results).toEqual([
    { type: 'return', value: 'first' },
    { type: 'return', value: 'second' },
    { type: 'return', value: 'default' },
    { type: 'return', value: 'default' },
  ]);
});
```

### mock.instances Property

When a mock function is used as a constructor, `mock.instances` tracks the
created instances:

```typescript
import { vi, expect, it } from 'vitest';

it('tracks constructor instances', () => {
  const MockClass = vi.fn(function(this: { value: number }, value: number) {
    this.value = value;
  });

  const instance1 = new MockClass(10);
  const instance2 = new MockClass(20);

  // Track instances
  expect(MockClass.mock.instances.length).toBe(2);
  expect(MockClass.mock.instances[0].value).toBe(10);
  expect(MockClass.mock.instances[1].value).toBe(20);
});
```

### mock.contexts Property

The `mock.contexts` property tracks the `this` context for each call:

```typescript
import { vi, expect, it } from 'vitest';

it('tracks this context', () => {
  const mockFn = vi.fn();

  const obj1 = { name: 'obj1' };
  const obj2 = { name: 'obj2' };

  mockFn.call(obj1, 'arg1');
  mockFn.call(obj2, 'arg2');

  expect(mockFn.mock.contexts[0]).toBe(obj1);
  expect(mockFn.mock.contexts[1]).toBe(obj2);
});
```

---

## 3. Mock Return Values

Control what mock functions return with various methods.

### mockReturnValue()

Sets a default return value for all calls:

```typescript
import { vi, expect, it } from 'vitest';

it('returns the same value for all calls', () => {
  const mockFn = vi.fn();
  mockFn.mockReturnValue(42);

  expect(mockFn()).toBe(42);
  expect(mockFn()).toBe(42);
  expect(mockFn()).toBe(42);
});

it('can return complex objects', () => {
  const mockFn = vi.fn();
  mockFn.mockReturnValue({
    status: 'success',
    data: [1, 2, 3],
  });

  const result = mockFn();
  expect(result.status).toBe('success');
  expect(result.data).toHaveLength(3);
});
```

### mockReturnValueOnce()

Returns a value only for the next call, then falls back to the default:

```typescript
import { vi, expect, it } from 'vitest';

it('returns different values sequentially', () => {
  const mockFn = vi.fn();

  mockFn
    .mockReturnValueOnce('first')
    .mockReturnValueOnce('second')
    .mockReturnValueOnce('third')
    .mockReturnValue('default');

  expect(mockFn()).toBe('first');
  expect(mockFn()).toBe('second');
  expect(mockFn()).toBe('third');
  expect(mockFn()).toBe('default');
  expect(mockFn()).toBe('default');
});

it('simulates different states', () => {
  const checkConnection = vi.fn()
    .mockReturnValueOnce(false)  // First check: disconnected
    .mockReturnValueOnce(false)  // Second check: still disconnected
    .mockReturnValue(true);       // Subsequent checks: connected

  expect(checkConnection()).toBe(false);
  expect(checkConnection()).toBe(false);
  expect(checkConnection()).toBe(true);
  expect(checkConnection()).toBe(true);
});
```

### mockResolvedValue() for Async Functions

```typescript
import { vi, expect, it } from 'vitest';

it('mocks async function that resolves', async () => {
  const fetchUser = vi.fn();

  fetchUser.mockResolvedValue({
    id: 1,
    name: 'John Doe',
  });

  const user = await fetchUser(1);
  expect(user.name).toBe('John Doe');
});

it('chains resolved values', async () => {
  const fetchData = vi.fn();

  fetchData
    .mockResolvedValueOnce({ page: 1, items: ['a', 'b'] })
    .mockResolvedValueOnce({ page: 2, items: ['c', 'd'] })
    .mockResolvedValue({ page: 3, items: [] });

  expect(await fetchData()).toEqual({ page: 1, items: ['a', 'b'] });
  expect(await fetchData()).toEqual({ page: 2, items: ['c', 'd'] });
  expect(await fetchData()).toEqual({ page: 3, items: [] });
});
```

### mockRejectedValue() for Async Errors

```typescript
import { vi, expect, it } from 'vitest';

it('mocks async function that rejects', async () => {
  const fetchUser = vi.fn();

  fetchUser.mockRejectedValue(new Error('User not found'));

  await expect(fetchUser(999)).rejects.toThrow('User not found');
});

it('simulates retry with eventual success', async () => {
  const unstableApi = vi.fn();

  unstableApi
    .mockRejectedValueOnce(new Error('Network error'))
    .mockRejectedValueOnce(new Error('Timeout'))
    .mockResolvedValue({ success: true });

  await expect(unstableApi()).rejects.toThrow('Network error');
  await expect(unstableApi()).rejects.toThrow('Timeout');
  expect(await unstableApi()).toEqual({ success: true });
});

it('tests error handling', async () => {
  const apiCall = vi.fn().mockRejectedValue(new Error('API Error'));

  const handleApiCall = async () => {
    try {
      return await apiCall();
    } catch (error) {
      return { error: (error as Error).message };
    }
  };

  const result = await handleApiCall();
  expect(result).toEqual({ error: 'API Error' });
});
```

---

## 4. Mock Implementations

For more complex mocking scenarios, use implementation functions.

### mockImplementation()

Replaces the mock's implementation entirely:

```typescript
import { vi, expect, it } from 'vitest';

it('provides custom implementation', () => {
  const mockFn = vi.fn();

  mockFn.mockImplementation((a: number, b: number) => {
    return a * b;
  });

  expect(mockFn(3, 4)).toBe(12);
  expect(mockFn(5, 6)).toBe(30);
});

it('can access arguments and return computed values', () => {
  const processItem = vi.fn();

  processItem.mockImplementation((item: { id: number; name: string }) => {
    return {
      ...item,
      processed: true,
      processedAt: '2024-01-01',
    };
  });

  const result = processItem({ id: 1, name: 'Test' });
  expect(result).toEqual({
    id: 1,
    name: 'Test',
    processed: true,
    processedAt: '2024-01-01',
  });
});

it('can simulate async operations', async () => {
  const fetchWithDelay = vi.fn();

  fetchWithDelay.mockImplementation(async (id: number) => {
    // Simulate async processing
    return { id, fetched: true };
  });

  const result = await fetchWithDelay(123);
  expect(result).toEqual({ id: 123, fetched: true });
});
```

### mockImplementationOnce()

Provides implementation for a single call:

```typescript
import { vi, expect, it } from 'vitest';

it('uses different implementations per call', () => {
  const mockFn = vi.fn();

  mockFn
    .mockImplementationOnce(() => 'first implementation')
    .mockImplementationOnce(() => 'second implementation')
    .mockImplementation(() => 'default implementation');

  expect(mockFn()).toBe('first implementation');
  expect(mockFn()).toBe('second implementation');
  expect(mockFn()).toBe('default implementation');
  expect(mockFn()).toBe('default implementation');
});

it('simulates state changes', () => {
  let connectionAttempts = 0;

  const connect = vi.fn()
    .mockImplementationOnce(() => {
      connectionAttempts++;
      throw new Error('Connection refused');
    })
    .mockImplementationOnce(() => {
      connectionAttempts++;
      throw new Error('Timeout');
    })
    .mockImplementation(() => {
      connectionAttempts++;
      return { connected: true, attempts: connectionAttempts };
    });

  expect(() => connect()).toThrow('Connection refused');
  expect(() => connect()).toThrow('Timeout');
  expect(connect()).toEqual({ connected: true, attempts: 3 });
});
```

### Combining Return Values and Implementations

```typescript
import { vi, expect, it } from 'vitest';

it('falls back to mockReturnValue after mockImplementationOnce', () => {
  const mockFn = vi.fn()
    .mockImplementationOnce((x: number) => x * 2)
    .mockReturnValue(100);

  expect(mockFn(5)).toBe(10);  // Uses implementation
  expect(mockFn(5)).toBe(100); // Falls back to return value
});
```

---

## 5. vi.spyOn() - Spying on Methods

`vi.spyOn()` creates a spy on an existing object method, allowing you to track
calls while optionally preserving or replacing the original behavior.

### Basic Spying

```typescript
import { vi, expect, it, describe, afterEach } from 'vitest';

describe('vi.spyOn()', () => {
  const calculator = {
    add: (a: number, b: number) => a + b,
    multiply: (a: number, b: number) => a * b,
  };

  afterEach(() => {
    vi.restoreAllMocks();
  });

  it('spies on method while preserving behavior', () => {
    const spy = vi.spyOn(calculator, 'add');

    // Original behavior is preserved
    expect(calculator.add(2, 3)).toBe(5);

    // But calls are tracked
    expect(spy).toHaveBeenCalledWith(2, 3);
    expect(spy).toHaveBeenCalledTimes(1);
  });

  it('can replace the implementation', () => {
    const spy = vi.spyOn(calculator, 'add').mockReturnValue(100);

    expect(calculator.add(2, 3)).toBe(100);
    expect(spy).toHaveBeenCalledWith(2, 3);
  });

  it('can provide custom implementation', () => {
    vi.spyOn(calculator, 'multiply').mockImplementation((a, b) => {
      console.log(`Multiplying ${a} by ${b}`);
      return a * b * 2; // Double the result
    });

    expect(calculator.multiply(3, 4)).toBe(24);
  });
});
```

### Spying on Object Getters and Setters

```typescript
import { vi, expect, it, afterEach } from 'vitest';

const config = {
  _apiUrl: 'https://api.example.com',
  get apiUrl() {
    return this._apiUrl;
  },
  set apiUrl(value: string) {
    this._apiUrl = value;
  },
};

afterEach(() => {
  vi.restoreAllMocks();
});

it('spies on getter', () => {
  const spy = vi.spyOn(config, 'apiUrl', 'get').mockReturnValue('https://mock.api.com');

  expect(config.apiUrl).toBe('https://mock.api.com');
  expect(spy).toHaveBeenCalled();
});

it('spies on setter', () => {
  const spy = vi.spyOn(config, 'apiUrl', 'set');

  config.apiUrl = 'https://new.api.com';

  expect(spy).toHaveBeenCalledWith('https://new.api.com');
});
```

### Spying on Class Methods

```typescript
import { vi, expect, it, describe, afterEach } from 'vitest';

class UserService {
  async getUser(id: number) {
    // Real implementation would fetch from API
    return { id, name: 'Real User' };
  }

  async saveUser(user: { id: number; name: string }) {
    // Real implementation would save to database
    return { success: true };
  }
}

describe('spying on class methods', () => {
  afterEach(() => {
    vi.restoreAllMocks();
  });

  it('spies on instance methods', async () => {
    const service = new UserService();
    const spy = vi.spyOn(service, 'getUser').mockResolvedValue({
      id: 1,
      name: 'Mocked User',
    });

    const user = await service.getUser(1);

    expect(user.name).toBe('Mocked User');
    expect(spy).toHaveBeenCalledWith(1);
  });

  it('spies on prototype methods', async () => {
    const spy = vi.spyOn(UserService.prototype, 'getUser').mockResolvedValue({
      id: 1,
      name: 'Prototype Mock',
    });

    const service1 = new UserService();
    const service2 = new UserService();

    expect(await service1.getUser(1)).toEqual({ id: 1, name: 'Prototype Mock' });
    expect(await service2.getUser(2)).toEqual({ id: 1, name: 'Prototype Mock' });
    expect(spy).toHaveBeenCalledTimes(2);
  });
});
```

### Spying on Built-in Objects

```typescript
import { vi, expect, it, afterEach } from 'vitest';

afterEach(() => {
  vi.restoreAllMocks();
});

it('spies on console.log', () => {
  const spy = vi.spyOn(console, 'log').mockImplementation(() => {});

  console.log('Test message');
  console.log('Another message', { data: true });

  expect(spy).toHaveBeenCalledTimes(2);
  expect(spy).toHaveBeenCalledWith('Test message');
  expect(spy).toHaveBeenCalledWith('Another message', { data: true });
});

it('spies on Math.random', () => {
  vi.spyOn(Math, 'random').mockReturnValue(0.5);

  expect(Math.random()).toBe(0.5);
  expect(Math.random()).toBe(0.5);
});

it('spies on Date.now', () => {
  vi.spyOn(Date, 'now').mockReturnValue(1704067200000); // 2024-01-01

  expect(Date.now()).toBe(1704067200000);
});
```

---

## 6. vi.mock() - Module Mocking

`vi.mock()` replaces entire modules with mock implementations. The mock
is hoisted to the top of the file.

### Basic Module Mocking

```typescript
import { vi, expect, it, describe } from 'vitest';

// This is hoisted to the top of the file
vi.mock('./userService', () => ({
  getUser: vi.fn().mockResolvedValue({ id: 1, name: 'Mock User' }),
  createUser: vi.fn().mockResolvedValue({ id: 2, name: 'New User' }),
  deleteUser: vi.fn().mockResolvedValue({ success: true }),
}));

// Import after vi.mock() - but vi.mock is hoisted so this works
import { getUser, createUser, deleteUser } from './userService';

describe('module mocking', () => {
  it('uses mocked getUser', async () => {
    const user = await getUser(1);
    expect(user).toEqual({ id: 1, name: 'Mock User' });
  });

  it('uses mocked createUser', async () => {
    const newUser = await createUser({ name: 'Test' });
    expect(newUser).toEqual({ id: 2, name: 'New User' });
  });
});
```

### Auto-Mocking Modules

When no factory function is provided, Vitest auto-mocks all exports:

```typescript
import { vi, expect, it } from 'vitest';

// Auto-mock - all exports become vi.fn()
vi.mock('./database');

import { query, connect, disconnect } from './database';

it('auto-mocks all exports', () => {
  // All functions are now vi.fn()
  expect(vi.isMockFunction(query)).toBe(true);
  expect(vi.isMockFunction(connect)).toBe(true);
  expect(vi.isMockFunction(disconnect)).toBe(true);

  // Can set return values
  (query as ReturnType<typeof vi.fn>).mockResolvedValue([{ id: 1 }]);
});
```

### Mocking Named and Default Exports

```typescript
import { vi, expect, it } from 'vitest';

// Module with both named and default exports
vi.mock('./api', () => ({
  default: vi.fn().mockReturnValue('default export'),
  namedExport1: vi.fn().mockReturnValue('named 1'),
  namedExport2: vi.fn().mockReturnValue('named 2'),
}));

import api, { namedExport1, namedExport2 } from './api';

it('mocks default export', () => {
  expect(api()).toBe('default export');
});

it('mocks named exports', () => {
  expect(namedExport1()).toBe('named 1');
  expect(namedExport2()).toBe('named 2');
});
```

### Mocking Node.js Built-in Modules

```typescript
import { vi, expect, it } from 'vitest';

vi.mock('fs', () => ({
  readFileSync: vi.fn().mockReturnValue('mocked file content'),
  writeFileSync: vi.fn(),
  existsSync: vi.fn().mockReturnValue(true),
}));

vi.mock('path', () => ({
  join: vi.fn((...args: string[]) => args.join('/')),
  resolve: vi.fn((...args: string[]) => '/' + args.join('/')),
}));

import { readFileSync, existsSync } from 'fs';
import { join } from 'path';

it('uses mocked fs module', () => {
  const content = readFileSync('/test/file.txt', 'utf-8');
  expect(content).toBe('mocked file content');
  expect(existsSync('/any/path')).toBe(true);
});

it('uses mocked path module', () => {
  expect(join('a', 'b', 'c')).toBe('a/b/c');
});
```

### Mocking Third-Party Libraries

```typescript
import { vi, expect, it } from 'vitest';

// Mock axios
vi.mock('axios', () => ({
  default: {
    get: vi.fn().mockResolvedValue({ data: { success: true } }),
    post: vi.fn().mockResolvedValue({ data: { id: 1 } }),
    create: vi.fn().mockReturnValue({
      get: vi.fn().mockResolvedValue({ data: [] }),
      post: vi.fn().mockResolvedValue({ data: {} }),
    }),
  },
}));

import axios from 'axios';

it('mocks axios.get', async () => {
  const response = await axios.get('/api/users');
  expect(response.data).toEqual({ success: true });
});

it('mocks axios instance', async () => {
  const instance = axios.create({ baseURL: 'http://api.example.com' });
  const response = await instance.get('/users');
  expect(response.data).toEqual([]);
});
```

---

## 7. vi.doMock() - Inline Module Mocking

Unlike `vi.mock()`, `vi.doMock()` is not hoisted. Use it when you need to
mock modules conditionally or with different implementations per test.

### Basic vi.doMock() Usage

```typescript
import { vi, expect, it, describe, beforeEach } from 'vitest';

describe('vi.doMock()', () => {
  beforeEach(() => {
    vi.resetModules();
  });

  it('mocks module for this specific test', async () => {
    vi.doMock('./config', () => ({
      apiUrl: 'http://test.api.com',
      timeout: 1000,
    }));

    // Must use dynamic import after doMock
    const { apiUrl, timeout } = await import('./config');

    expect(apiUrl).toBe('http://test.api.com');
    expect(timeout).toBe(1000);
  });

  it('can use different mock in another test', async () => {
    vi.doMock('./config', () => ({
      apiUrl: 'http://staging.api.com',
      timeout: 5000,
    }));

    const { apiUrl, timeout } = await import('./config');

    expect(apiUrl).toBe('http://staging.api.com');
    expect(timeout).toBe(5000);
  });
});
```

### Conditional Mocking

```typescript
import { vi, expect, it, describe, beforeEach } from 'vitest';

describe('conditional module mocking', () => {
  beforeEach(() => {
    vi.resetModules();
  });

  it('mocks based on environment', async () => {
    const isProduction = false;

    if (!isProduction) {
      vi.doMock('./logger', () => ({
        log: vi.fn(),
        error: vi.fn(),
        debug: vi.fn().mockImplementation(console.log),
      }));
    }

    const { debug } = await import('./logger');
    debug('Test message');

    expect(debug).toHaveBeenCalledWith('Test message');
  });
});
```

### vi.doUnmock()

Removes a module mock:

```typescript
import { vi, expect, it, describe, beforeEach } from 'vitest';

describe('vi.doUnmock()', () => {
  beforeEach(() => {
    vi.resetModules();
  });

  it('removes mock to use real module', async () => {
    // First, mock the module
    vi.doMock('./math', () => ({
      add: vi.fn().mockReturnValue(100),
    }));

    const mockedMath = await import('./math');
    expect(mockedMath.add(1, 2)).toBe(100);

    // Reset and unmock
    vi.resetModules();
    vi.doUnmock('./math');

    // Now get real module
    const realMath = await import('./math');
    expect(realMath.add(1, 2)).toBe(3);
  });
});
```

---

## 8. vi.importActual() and vi.importMock()

These functions allow you to access the real or auto-mocked version of a module
within a mock factory.

### vi.importActual()

Import the actual module implementation:

```typescript
import { vi, expect, it } from 'vitest';

vi.mock('./utils', async () => {
  // Get the real module
  const actual = await vi.importActual<typeof import('./utils')>('./utils');

  return {
    ...actual,
    // Only mock specific functions
    fetchData: vi.fn().mockResolvedValue({ mocked: true }),
  };
});

import { formatDate, parseDate, fetchData } from './utils';

it('uses real formatDate', () => {
  // Real implementation
  expect(formatDate(new Date('2024-01-01'))).toBe('2024-01-01');
});

it('uses mocked fetchData', async () => {
  // Mocked implementation
  const data = await fetchData();
  expect(data).toEqual({ mocked: true });
});
```

### vi.importMock()

Import the auto-mocked version of a module:

```typescript
import { vi, expect, it, describe, beforeEach } from 'vitest';

describe('vi.importMock()', () => {
  beforeEach(() => {
    vi.resetModules();
  });

  it('gets auto-mocked module', async () => {
    const mockedModule = await vi.importMock<typeof import('./database')>('./database');

    // All exports are auto-mocked
    expect(vi.isMockFunction(mockedModule.query)).toBe(true);
    expect(vi.isMockFunction(mockedModule.connect)).toBe(true);

    // Can configure mocks
    mockedModule.query.mockResolvedValue([{ id: 1 }]);

    const result = await mockedModule.query('SELECT * FROM users');
    expect(result).toEqual([{ id: 1 }]);
  });
});
```

### Combining importActual with Custom Logic

```typescript
import { vi, expect, it } from 'vitest';

vi.mock('./api', async () => {
  const actual = await vi.importActual<typeof import('./api')>('./api');

  return {
    ...actual,
    // Wrap the real function with additional logic
    fetchUser: vi.fn(async (id: number) => {
      if (id === 999) {
        throw new Error('User not found');
      }
      return actual.fetchUser(id);
    }),
  };
});

import { fetchUser } from './api';

it('throws for special ID', async () => {
  await expect(fetchUser(999)).rejects.toThrow('User not found');
});

it('uses real implementation for other IDs', async () => {
  const user = await fetchUser(1);
  expect(user).toBeDefined();
});
```

---

## 9. __mocks__ Directory

Vitest supports a `__mocks__` directory for manual mocks that can be reused
across tests.

### Directory Structure

```
src/
  __mocks__/
    axios.ts           # Mock for node_modules/axios
    fs.ts              # Mock for Node.js fs module
  services/
    __mocks__/
      userService.ts   # Mock for ./userService
    userService.ts
    userService.test.ts
```

### Creating a Manual Mock

```typescript
// src/__mocks__/axios.ts
import { vi } from 'vitest';

export default {
  get: vi.fn().mockResolvedValue({ data: {} }),
  post: vi.fn().mockResolvedValue({ data: {} }),
  put: vi.fn().mockResolvedValue({ data: {} }),
  delete: vi.fn().mockResolvedValue({ data: {} }),
  create: vi.fn().mockReturnThis(),
  interceptors: {
    request: { use: vi.fn() },
    response: { use: vi.fn() },
  },
};
```

### Using Manual Mocks

```typescript
// src/services/userService.test.ts
import { vi, expect, it, describe, beforeEach } from 'vitest';

// This will use src/__mocks__/axios.ts
vi.mock('axios');

import axios from 'axios';
import { UserService } from './userService';

describe('UserService with manual mock', () => {
  const service = new UserService();

  beforeEach(() => {
    vi.mocked(axios.get).mockResolvedValue({
      data: { id: 1, name: 'Test User' },
    });
  });

  it('fetches user using mocked axios', async () => {
    const user = await service.getUser(1);

    expect(axios.get).toHaveBeenCalledWith('/api/users/1');
    expect(user.name).toBe('Test User');
  });
});
```

### Manual Mock for Local Module

```typescript
// src/services/__mocks__/userService.ts
import { vi } from 'vitest';

export const getUser = vi.fn().mockResolvedValue({
  id: 1,
  name: 'Mocked User',
  email: 'mock@example.com',
});

export const createUser = vi.fn().mockResolvedValue({
  id: 2,
  name: 'New Mocked User',
});

export const deleteUser = vi.fn().mockResolvedValue({ success: true });
```

```typescript
// Another test file
import { vi, expect, it } from 'vitest';

// Uses the __mocks__/userService.ts file
vi.mock('./userService');

import { getUser } from './userService';

it('uses manual mock from __mocks__ directory', async () => {
  const user = await getUser(1);
  expect(user.name).toBe('Mocked User');
});
```

---

## 10. Mocking ES Modules

ES Modules have specific characteristics that require special handling when mocking.

### Understanding ES Module Exports

```typescript
// myModule.ts - ES Module with various export types
export const constant = 42;
export let mutableVar = 'initial';
export function namedFunction() {
  return 'named';
}
export class MyClass {
  getValue() {
    return 'class value';
  }
}
export default function defaultExport() {
  return 'default';
}
```

### Mocking ES Module Exports

```typescript
import { vi, expect, it, describe } from 'vitest';

// Mock the entire module
vi.mock('./myModule', () => ({
  constant: 100,
  mutableVar: 'mocked',
  namedFunction: vi.fn().mockReturnValue('mocked named'),
  MyClass: vi.fn().mockImplementation(() => ({
    getValue: vi.fn().mockReturnValue('mocked class value'),
  })),
  default: vi.fn().mockReturnValue('mocked default'),
}));

import myDefault, { constant, namedFunction, MyClass } from './myModule';

describe('ES Module mocking', () => {
  it('mocks constant exports', () => {
    expect(constant).toBe(100);
  });

  it('mocks named function exports', () => {
    expect(namedFunction()).toBe('mocked named');
  });

  it('mocks class exports', () => {
    const instance = new MyClass();
    expect(instance.getValue()).toBe('mocked class value');
  });

  it('mocks default exports', () => {
    expect(myDefault()).toBe('mocked default');
  });
});
```

### Re-exporting Modules

```typescript
import { vi, expect, it } from 'vitest';

// When a module re-exports from another module
// barrel.ts: export * from './moduleA'; export * from './moduleB';

vi.mock('./barrel', async () => {
  const actualA = await vi.importActual<typeof import('./moduleA')>('./moduleA');

  return {
    ...actualA,
    // Mock specific exports from the barrel
    specificFunction: vi.fn(),
  };
});
```

### Mocking ESM-only Packages

Some packages are ESM-only and require special handling:

```typescript
import { vi, expect, it, describe, beforeEach } from 'vitest';

// For ESM-only packages, you might need to mock at a deeper level
vi.mock('node-fetch', () => ({
  default: vi.fn().mockResolvedValue({
    ok: true,
    json: vi.fn().mockResolvedValue({ data: 'mocked' }),
    text: vi.fn().mockResolvedValue('mocked text'),
  }),
}));

import fetch from 'node-fetch';

describe('mocking ESM-only packages', () => {
  it('mocks node-fetch', async () => {
    const response = await fetch('https://api.example.com');
    const data = await response.json();

    expect(data).toEqual({ data: 'mocked' });
  });
});
```

---

## 11. Mocking Classes

Vitest provides multiple approaches for mocking classes and their methods.

### Mocking Class Constructors

```typescript
import { vi, expect, it, describe, beforeEach } from 'vitest';

// Original class
class DatabaseConnection {
  private connectionString: string;

  constructor(connectionString: string) {
    this.connectionString = connectionString;
  }

  async connect() {
    // Real connection logic
    return { connected: true };
  }

  async query(sql: string) {
    // Real query logic
    return [];
  }
}

describe('mocking class constructors', () => {
  it('mocks entire class', () => {
    const MockDatabaseConnection = vi.fn().mockImplementation((connectionString: string) => ({
      connectionString,
      connect: vi.fn().mockResolvedValue({ connected: true, mocked: true }),
      query: vi.fn().mockResolvedValue([{ id: 1 }]),
    }));

    const db = new MockDatabaseConnection('mock://connection');

    expect(db.connectionString).toBe('mock://connection');
    expect(db.connect()).resolves.toEqual({ connected: true, mocked: true });
  });
});
```

### Mocking Class with vi.mock()

```typescript
import { vi, expect, it, describe } from 'vitest';

vi.mock('./DatabaseConnection', () => {
  const MockDatabaseConnection = vi.fn().mockImplementation(() => ({
    connect: vi.fn().mockResolvedValue({ connected: true }),
    query: vi.fn().mockResolvedValue([]),
    disconnect: vi.fn().mockResolvedValue(undefined),
  }));

  return { DatabaseConnection: MockDatabaseConnection };
});

import { DatabaseConnection } from './DatabaseConnection';

describe('mocked DatabaseConnection', () => {
  it('creates mocked instance', async () => {
    const db = new DatabaseConnection('test://db');

    await expect(db.connect()).resolves.toEqual({ connected: true });
    expect(db.query).toBeDefined();
  });
});
```

### Mocking Specific Class Methods

```typescript
import { vi, expect, it, describe, beforeEach, afterEach } from 'vitest';

class UserRepository {
  async findById(id: number) {
    // Real database call
    return { id, name: 'Real User' };
  }

  async findAll() {
    // Real database call
    return [];
  }

  async save(user: { name: string }) {
    // Real database call
    return { id: 1, ...user };
  }
}

describe('mocking specific class methods', () => {
  let repo: UserRepository;

  beforeEach(() => {
    repo = new UserRepository();
  });

  afterEach(() => {
    vi.restoreAllMocks();
  });

  it('mocks findById while keeping other methods real', async () => {
    vi.spyOn(repo, 'findById').mockResolvedValue({
      id: 1,
      name: 'Mocked User',
    });

    const user = await repo.findById(1);
    expect(user.name).toBe('Mocked User');

    // Other methods remain real (or can be separately mocked)
  });

  it('mocks all methods of instance', async () => {
    vi.spyOn(repo, 'findById').mockResolvedValue({ id: 1, name: 'Mock' });
    vi.spyOn(repo, 'findAll').mockResolvedValue([{ id: 1 }, { id: 2 }]);
    vi.spyOn(repo, 'save').mockResolvedValue({ id: 99, name: 'Saved' });

    expect(await repo.findAll()).toHaveLength(2);
    expect((await repo.save({ name: 'Test' })).id).toBe(99);
  });
});
```

### Mocking Abstract Classes

```typescript
import { vi, expect, it, describe } from 'vitest';

abstract class BaseService {
  abstract fetchData(): Promise<unknown>;

  async processData() {
    const data = await this.fetchData();
    return { processed: true, data };
  }
}

describe('mocking abstract classes', () => {
  it('creates concrete mock implementation', async () => {
    const MockService = class extends BaseService {
      fetchData = vi.fn().mockResolvedValue({ items: [1, 2, 3] });
    };

    const service = new MockService();
    const result = await service.processData();

    expect(result).toEqual({
      processed: true,
      data: { items: [1, 2, 3] },
    });
  });
});
```

### Mocking Static Methods

```typescript
import { vi, expect, it, describe, afterEach } from 'vitest';

class Analytics {
  static track(event: string, data: Record<string, unknown>) {
    // Real analytics call
    console.log('Tracking:', event, data);
  }

  static identify(userId: string) {
    // Real identify call
    console.log('Identifying:', userId);
  }
}

describe('mocking static methods', () => {
  afterEach(() => {
    vi.restoreAllMocks();
  });

  it('mocks static methods', () => {
    const trackSpy = vi.spyOn(Analytics, 'track').mockImplementation(() => {});
    const identifySpy = vi.spyOn(Analytics, 'identify').mockImplementation(() => {});

    Analytics.track('page_view', { page: '/home' });
    Analytics.identify('user-123');

    expect(trackSpy).toHaveBeenCalledWith('page_view', { page: '/home' });
    expect(identifySpy).toHaveBeenCalledWith('user-123');
  });
});
```

---

## 12. Mocking Timers

Vitest provides comprehensive timer mocking through `vi.useFakeTimers()`.

### Basic Timer Mocking

```typescript
import { vi, expect, it, describe, beforeEach, afterEach } from 'vitest';

describe('timer mocking', () => {
  beforeEach(() => {
    vi.useFakeTimers();
  });

  afterEach(() => {
    vi.useRealTimers();
  });

  it('mocks setTimeout', () => {
    const callback = vi.fn();

    setTimeout(callback, 1000);

    expect(callback).not.toHaveBeenCalled();

    vi.advanceTimersByTime(1000);

    expect(callback).toHaveBeenCalledTimes(1);
  });

  it('mocks setInterval', () => {
    const callback = vi.fn();

    setInterval(callback, 500);

    vi.advanceTimersByTime(500);
    expect(callback).toHaveBeenCalledTimes(1);

    vi.advanceTimersByTime(500);
    expect(callback).toHaveBeenCalledTimes(2);

    vi.advanceTimersByTime(1500);
    expect(callback).toHaveBeenCalledTimes(5);
  });

  it('runs all timers', () => {
    const callback1 = vi.fn();
    const callback2 = vi.fn();

    setTimeout(callback1, 1000);
    setTimeout(callback2, 5000);

    vi.runAllTimers();

    expect(callback1).toHaveBeenCalled();
    expect(callback2).toHaveBeenCalled();
  });
});
```

### Advanced Timer Control

```typescript
import { vi, expect, it, describe, beforeEach, afterEach } from 'vitest';

describe('advanced timer control', () => {
  beforeEach(() => {
    vi.useFakeTimers();
  });

  afterEach(() => {
    vi.useRealTimers();
  });

  it('runs only pending timers', () => {
    const callback = vi.fn(() => {
      // This schedules another timer
      setTimeout(callback, 1000);
    });

    setTimeout(callback, 1000);

    // Only run the first timer, not recursive ones
    vi.runOnlyPendingTimers();

    expect(callback).toHaveBeenCalledTimes(1);

    // Run the next pending timer
    vi.runOnlyPendingTimers();

    expect(callback).toHaveBeenCalledTimes(2);
  });

  it('advances timers to next timer', () => {
    const callback1 = vi.fn();
    const callback2 = vi.fn();

    setTimeout(callback1, 1000);
    setTimeout(callback2, 5000);

    // Jump to and execute the next timer
    vi.advanceTimersToNextTimer();
    expect(callback1).toHaveBeenCalled();
    expect(callback2).not.toHaveBeenCalled();

    vi.advanceTimersToNextTimer();
    expect(callback2).toHaveBeenCalled();
  });

  it('gets pending timers count', () => {
    setTimeout(() => {}, 1000);
    setTimeout(() => {}, 2000);
    setInterval(() => {}, 500);

    expect(vi.getTimerCount()).toBe(3);

    vi.advanceTimersByTime(1000);

    // setTimeout at 1000 executed, interval fired twice
    expect(vi.getTimerCount()).toBe(2); // One setTimeout + interval remaining
  });
});
```

### Mocking Date and System Time

```typescript
import { vi, expect, it, describe, beforeEach, afterEach } from 'vitest';

describe('mocking system time', () => {
  beforeEach(() => {
    vi.useFakeTimers();
  });

  afterEach(() => {
    vi.useRealTimers();
  });

  it('sets specific system time', () => {
    vi.setSystemTime(new Date('2024-06-15T10:30:00Z'));

    expect(new Date().toISOString()).toBe('2024-06-15T10:30:00.000Z');
    expect(Date.now()).toBe(1718444400000);
  });

  it('advances system time with timers', () => {
    vi.setSystemTime(new Date('2024-01-01T00:00:00Z'));

    const startTime = Date.now();

    vi.advanceTimersByTime(3600000); // 1 hour

    expect(Date.now() - startTime).toBe(3600000);
    expect(new Date().toISOString()).toBe('2024-01-01T01:00:00.000Z');
  });

  it('tests time-dependent code', () => {
    vi.setSystemTime(new Date('2024-12-25'));

    function isChristmas() {
      const now = new Date();
      return now.getMonth() === 11 && now.getDate() === 25;
    }

    expect(isChristmas()).toBe(true);

    vi.setSystemTime(new Date('2024-07-04'));
    expect(isChristmas()).toBe(false);
  });
});
```

### Timer Configuration Options

```typescript
import { vi, expect, it, describe, beforeEach, afterEach } from 'vitest';

describe('timer configuration', () => {
  afterEach(() => {
    vi.useRealTimers();
  });

  it('configures which timers to fake', () => {
    vi.useFakeTimers({
      toFake: ['setTimeout', 'setInterval'],
      // Don't fake Date, performance, etc.
    });

    // setTimeout is faked
    const callback = vi.fn();
    setTimeout(callback, 1000);
    vi.advanceTimersByTime(1000);
    expect(callback).toHaveBeenCalled();

    // Date.now() is real
    const realNow = Date.now();
    expect(realNow).toBeGreaterThan(0);
  });

  it('sets initial system time', () => {
    vi.useFakeTimers({
      now: new Date('2024-01-01'),
    });

    expect(new Date().getFullYear()).toBe(2024);
  });

  it('limits timer iterations to prevent infinite loops', () => {
    vi.useFakeTimers({
      loopLimit: 100,
    });

    let count = 0;
    const interval = setInterval(() => {
      count++;
      if (count > 1000) clearInterval(interval);
    }, 10);

    // This would normally run forever, but loopLimit prevents it
    expect(() => vi.runAllTimers()).toThrow();
  });
});
```

### Testing Debounce and Throttle

```typescript
import { vi, expect, it, describe, beforeEach, afterEach } from 'vitest';

// Simple debounce implementation
function debounce<T extends (...args: unknown[]) => unknown>(
  fn: T,
  delay: number
): T {
  let timeoutId: ReturnType<typeof setTimeout>;
  return ((...args: Parameters<T>) => {
    clearTimeout(timeoutId);
    timeoutId = setTimeout(() => fn(...args), delay);
  }) as T;
}

describe('testing debounce', () => {
  beforeEach(() => {
    vi.useFakeTimers();
  });

  afterEach(() => {
    vi.useRealTimers();
  });

  it('debounces function calls', () => {
    const callback = vi.fn();
    const debouncedCallback = debounce(callback, 300);

    // Call multiple times rapidly
    debouncedCallback();
    debouncedCallback();
    debouncedCallback();

    expect(callback).not.toHaveBeenCalled();

    // Advance time partially
    vi.advanceTimersByTime(200);
    expect(callback).not.toHaveBeenCalled();

    // Advance past debounce delay
    vi.advanceTimersByTime(100);
    expect(callback).toHaveBeenCalledTimes(1);
  });

  it('resets debounce on new calls', () => {
    const callback = vi.fn();
    const debouncedCallback = debounce(callback, 300);

    debouncedCallback();
    vi.advanceTimersByTime(200);

    debouncedCallback(); // Reset the timer
    vi.advanceTimersByTime(200);

    expect(callback).not.toHaveBeenCalled();

    vi.advanceTimersByTime(100);
    expect(callback).toHaveBeenCalledTimes(1);
  });
});
```

---

## 13. Mocking Global Objects

Vitest allows mocking of global objects like `window`, `document`, `navigator`, etc.

### Mocking Window Properties

```typescript
import { vi, expect, it, describe, beforeEach, afterEach } from 'vitest';

describe('mocking window', () => {
  const originalWindow = global.window;

  beforeEach(() => {
    // Create a mock window object
    global.window = {
      ...originalWindow,
      location: {
        href: 'http://localhost:3000',
        pathname: '/test',
        search: '?query=value',
        hash: '#section',
        assign: vi.fn(),
        replace: vi.fn(),
        reload: vi.fn(),
      },
      localStorage: {
        getItem: vi.fn(),
        setItem: vi.fn(),
        removeItem: vi.fn(),
        clear: vi.fn(),
        length: 0,
        key: vi.fn(),
      },
    } as unknown as Window & typeof globalThis;
  });

  afterEach(() => {
    global.window = originalWindow;
  });

  it('mocks window.location', () => {
    expect(window.location.pathname).toBe('/test');

    window.location.assign('http://example.com');
    expect(window.location.assign).toHaveBeenCalledWith('http://example.com');
  });

  it('mocks localStorage', () => {
    (window.localStorage.getItem as ReturnType<typeof vi.fn>).mockReturnValue('stored-value');

    const value = window.localStorage.getItem('key');

    expect(value).toBe('stored-value');
    expect(window.localStorage.getItem).toHaveBeenCalledWith('key');
  });
});
```

### Mocking fetch

```typescript
import { vi, expect, it, describe, beforeEach, afterEach } from 'vitest';

describe('mocking fetch', () => {
  const originalFetch = global.fetch;

  beforeEach(() => {
    global.fetch = vi.fn();
  });

  afterEach(() => {
    global.fetch = originalFetch;
  });

  it('mocks successful fetch response', async () => {
    const mockResponse = {
      ok: true,
      status: 200,
      json: vi.fn().mockResolvedValue({ data: 'test' }),
      text: vi.fn().mockResolvedValue('text response'),
    };

    (global.fetch as ReturnType<typeof vi.fn>).mockResolvedValue(mockResponse);

    const response = await fetch('https://api.example.com/data');
    const data = await response.json();

    expect(global.fetch).toHaveBeenCalledWith('https://api.example.com/data');
    expect(data).toEqual({ data: 'test' });
  });

  it('mocks fetch error', async () => {
    (global.fetch as ReturnType<typeof vi.fn>).mockRejectedValue(new Error('Network error'));

    await expect(fetch('https://api.example.com')).rejects.toThrow('Network error');
  });

  it('mocks different responses based on URL', async () => {
    (global.fetch as ReturnType<typeof vi.fn>).mockImplementation((url: string) => {
      if (url.includes('/users')) {
        return Promise.resolve({
          ok: true,
          json: () => Promise.resolve([{ id: 1, name: 'User' }]),
        });
      }
      if (url.includes('/products')) {
        return Promise.resolve({
          ok: true,
          json: () => Promise.resolve([{ id: 1, name: 'Product' }]),
        });
      }
      return Promise.resolve({
        ok: false,
        status: 404,
      });
    });

    const usersResponse = await fetch('/api/users');
    const users = await usersResponse.json();
    expect(users).toEqual([{ id: 1, name: 'User' }]);

    const productsResponse = await fetch('/api/products');
    const products = await productsResponse.json();
    expect(products).toEqual([{ id: 1, name: 'Product' }]);
  });
});
```

### Mocking Navigator

```typescript
import { vi, expect, it, describe, beforeEach, afterEach } from 'vitest';

describe('mocking navigator', () => {
  const originalNavigator = global.navigator;

  beforeEach(() => {
    Object.defineProperty(global, 'navigator', {
      value: {
        userAgent: 'Mozilla/5.0 (Test Browser)',
        language: 'en-US',
        languages: ['en-US', 'en'],
        onLine: true,
        geolocation: {
          getCurrentPosition: vi.fn(),
          watchPosition: vi.fn(),
          clearWatch: vi.fn(),
        },
        clipboard: {
          writeText: vi.fn().mockResolvedValue(undefined),
          readText: vi.fn().mockResolvedValue('clipboard content'),
        },
      },
      writable: true,
      configurable: true,
    });
  });

  afterEach(() => {
    Object.defineProperty(global, 'navigator', {
      value: originalNavigator,
      writable: true,
      configurable: true,
    });
  });

  it('mocks navigator.onLine', () => {
    expect(navigator.onLine).toBe(true);

    Object.defineProperty(navigator, 'onLine', { value: false });
    expect(navigator.onLine).toBe(false);
  });

  it('mocks clipboard API', async () => {
    await navigator.clipboard.writeText('test text');
    expect(navigator.clipboard.writeText).toHaveBeenCalledWith('test text');

    const text = await navigator.clipboard.readText();
    expect(text).toBe('clipboard content');
  });

  it('mocks geolocation', () => {
    const successCallback = vi.fn();
    const errorCallback = vi.fn();

    navigator.geolocation.getCurrentPosition(successCallback, errorCallback);

    expect(navigator.geolocation.getCurrentPosition).toHaveBeenCalledWith(
      successCallback,
      errorCallback
    );
  });
});
```

---

## 14. vi.stubGlobal() and vi.stubEnv()

Vitest provides convenient methods for stubbing global variables and
environment variables.

### vi.stubGlobal()

```typescript
import { vi, expect, it, describe, beforeEach, afterEach } from 'vitest';

describe('vi.stubGlobal()', () => {
  afterEach(() => {
    vi.unstubAllGlobals();
  });

  it('stubs global fetch', async () => {
    vi.stubGlobal('fetch', vi.fn().mockResolvedValue({
      ok: true,
      json: () => Promise.resolve({ stubbed: true }),
    }));

    const response = await fetch('/api/data');
    const data = await response.json();

    expect(data).toEqual({ stubbed: true });
  });

  it('stubs custom global', () => {
    vi.stubGlobal('myGlobalConfig', {
      apiUrl: 'http://test.api.com',
      debug: true,
    });

    expect((globalThis as Record<string, unknown>).myGlobalConfig).toEqual({
      apiUrl: 'http://test.api.com',
      debug: true,
    });
  });

  it('stubs window.matchMedia', () => {
    vi.stubGlobal('matchMedia', vi.fn().mockImplementation((query: string) => ({
      matches: query === '(prefers-color-scheme: dark)',
      media: query,
      onchange: null,
      addListener: vi.fn(),
      removeListener: vi.fn(),
      addEventListener: vi.fn(),
      removeEventListener: vi.fn(),
      dispatchEvent: vi.fn(),
    })));

    const darkModeQuery = window.matchMedia('(prefers-color-scheme: dark)');
    expect(darkModeQuery.matches).toBe(true);

    const lightModeQuery = window.matchMedia('(prefers-color-scheme: light)');
    expect(lightModeQuery.matches).toBe(false);
  });

  it('stubs IntersectionObserver', () => {
    const mockIntersectionObserver = vi.fn().mockImplementation(() => ({
      observe: vi.fn(),
      unobserve: vi.fn(),
      disconnect: vi.fn(),
    }));

    vi.stubGlobal('IntersectionObserver', mockIntersectionObserver);

    const observer = new IntersectionObserver(() => {});
    observer.observe(document.createElement('div'));

    expect(mockIntersectionObserver).toHaveBeenCalled();
  });
});
```

### vi.stubEnv()

```typescript
import { vi, expect, it, describe, afterEach } from 'vitest';

describe('vi.stubEnv()', () => {
  afterEach(() => {
    vi.unstubAllEnvs();
  });

  it('stubs environment variables', () => {
    vi.stubEnv('API_KEY', 'test-api-key-123');
    vi.stubEnv('NODE_ENV', 'test');
    vi.stubEnv('DEBUG', 'true');

    expect(process.env.API_KEY).toBe('test-api-key-123');
    expect(process.env.NODE_ENV).toBe('test');
    expect(process.env.DEBUG).toBe('true');
  });

  it('tests code that depends on env vars', () => {
    vi.stubEnv('DATABASE_URL', 'postgres://test:test@localhost/testdb');

    function getDbConfig() {
      return {
        url: process.env.DATABASE_URL,
        ssl: process.env.NODE_ENV === 'production',
      };
    }

    const config = getDbConfig();
    expect(config.url).toBe('postgres://test:test@localhost/testdb');
  });

  it('unstubs specific env var', () => {
    const originalValue = process.env.PATH;

    vi.stubEnv('PATH', '/mocked/path');
    expect(process.env.PATH).toBe('/mocked/path');

    vi.unstubAllEnvs();
    expect(process.env.PATH).toBe(originalValue);
  });
});
```

### Combining stubGlobal and stubEnv

```typescript
import { vi, expect, it, describe, afterEach } from 'vitest';

describe('combined global and env stubbing', () => {
  afterEach(() => {
    vi.unstubAllGlobals();
    vi.unstubAllEnvs();
  });

  it('creates complete test environment', async () => {
    // Stub environment
    vi.stubEnv('API_BASE_URL', 'http://test.api.com');
    vi.stubEnv('API_KEY', 'test-key');

    // Stub fetch
    vi.stubGlobal('fetch', vi.fn().mockResolvedValue({
      ok: true,
      json: () => Promise.resolve({ users: [] }),
    }));

    // Function under test
    async function fetchUsers() {
      const response = await fetch(
        `${process.env.API_BASE_URL}/users`,
        {
          headers: { 'X-API-Key': process.env.API_KEY! },
        }
      );
      return response.json();
    }

    const result = await fetchUsers();

    expect(fetch).toHaveBeenCalledWith(
      'http://test.api.com/users',
      { headers: { 'X-API-Key': 'test-key' } }
    );
    expect(result).toEqual({ users: [] });
  });
});
```

---

## 15. Resetting Mocks

Vitest provides several methods to reset mock state between tests.

### mockClear()

Clears call history but keeps the mock implementation:

```typescript
import { vi, expect, it, describe, beforeEach } from 'vitest';

describe('mockClear()', () => {
  const mockFn = vi.fn().mockReturnValue('mocked');

  beforeEach(() => {
    mockFn.mockClear();
  });

  it('first test', () => {
    mockFn('arg1');
    mockFn('arg2');

    expect(mockFn).toHaveBeenCalledTimes(2);
    expect(mockFn.mock.calls).toEqual([['arg1'], ['arg2']]);
  });

  it('second test starts fresh', () => {
    // Call history is cleared
    expect(mockFn).not.toHaveBeenCalled();
    expect(mockFn.mock.calls).toEqual([]);

    // But implementation is preserved
    expect(mockFn()).toBe('mocked');
  });
});
```

### mockReset()

Clears call history AND resets return value/implementation to undefined:

```typescript
import { vi, expect, it, describe, beforeEach } from 'vitest';

describe('mockReset()', () => {
  const mockFn = vi.fn().mockReturnValue('mocked');

  it('has mocked return value', () => {
    expect(mockFn()).toBe('mocked');
  });

  it('after reset returns undefined', () => {
    mockFn.mockReset();

    // Call history cleared
    expect(mockFn).not.toHaveBeenCalled();

    // Return value also reset
    expect(mockFn()).toBeUndefined();
  });
});
```

### mockRestore()

Restores the original implementation (only works with vi.spyOn()):

```typescript
import { vi, expect, it, describe, afterEach } from 'vitest';

const calculator = {
  add: (a: number, b: number) => a + b,
  multiply: (a: number, b: number) => a * b,
};

describe('mockRestore()', () => {
  afterEach(() => {
    vi.restoreAllMocks();
  });

  it('restores original implementation', () => {
    const spy = vi.spyOn(calculator, 'add').mockReturnValue(100);

    expect(calculator.add(2, 3)).toBe(100);

    spy.mockRestore();

    // Original behavior restored
    expect(calculator.add(2, 3)).toBe(5);
  });
});
```

### Global Reset Methods

```typescript
import { vi, expect, it, describe, beforeEach, afterEach } from 'vitest';

describe('global reset methods', () => {
  const mock1 = vi.fn().mockReturnValue('mock1');
  const mock2 = vi.fn().mockReturnValue('mock2');

  // vi.clearAllMocks() - clears all mock call history
  beforeEach(() => {
    vi.clearAllMocks();
  });

  it('clears all mocks', () => {
    mock1();
    mock2();

    vi.clearAllMocks();

    expect(mock1).not.toHaveBeenCalled();
    expect(mock2).not.toHaveBeenCalled();

    // Implementations preserved
    expect(mock1()).toBe('mock1');
    expect(mock2()).toBe('mock2');
  });
});

describe('resetAllMocks', () => {
  const mock1 = vi.fn().mockReturnValue('mock1');
  const mock2 = vi.fn().mockReturnValue('mock2');

  it('resets all mocks', () => {
    mock1();
    mock2();

    vi.resetAllMocks();

    expect(mock1).not.toHaveBeenCalled();
    expect(mock2).not.toHaveBeenCalled();

    // Implementations also reset
    expect(mock1()).toBeUndefined();
    expect(mock2()).toBeUndefined();
  });
});

describe('restoreAllMocks', () => {
  const obj = {
    method: () => 'original',
  };

  afterEach(() => {
    vi.restoreAllMocks();
  });

  it('restores all spied methods', () => {
    vi.spyOn(obj, 'method').mockReturnValue('mocked');

    expect(obj.method()).toBe('mocked');

    vi.restoreAllMocks();

    expect(obj.method()).toBe('original');
  });
});
```

### Configuration in vitest.config.ts

```typescript
// vitest.config.ts
import { defineConfig } from 'vitest/config';

export default defineConfig({
  test: {
    // Automatically clear mocks before each test
    clearMocks: true,

    // Automatically reset mocks before each test
    resetMocks: true,

    // Automatically restore mocks before each test
    restoreMocks: true,

    // Reset modules before each test
    resetModules: true,
  },
});
```

---

## 16. Partial Mocking

Partial mocking allows you to mock specific parts of a module while keeping
the rest of the real implementation.

### Using importOriginal (Recommended)

```typescript
import { vi, expect, it, describe } from 'vitest';

vi.mock('./utils', async (importOriginal) => {
  const actual = await importOriginal<typeof import('./utils')>();

  return {
    ...actual,
    // Only mock fetchData, keep everything else real
    fetchData: vi.fn().mockResolvedValue({ mocked: true }),
  };
});

import { formatDate, parseDate, fetchData, validateEmail } from './utils';

describe('partial mocking with importOriginal', () => {
  it('uses real formatDate', () => {
    const result = formatDate(new Date('2024-01-15'));
    expect(result).toBe('2024-01-15'); // Real implementation
  });

  it('uses mocked fetchData', async () => {
    const data = await fetchData();
    expect(data).toEqual({ mocked: true }); // Mocked
  });

  it('uses real validateEmail', () => {
    expect(validateEmail('test@example.com')).toBe(true); // Real
    expect(validateEmail('invalid')).toBe(false); // Real
  });
});
```

### Partial Mock with Spy

```typescript
import { vi, expect, it, describe, beforeEach, afterEach } from 'vitest';
import * as utils from './utils';

describe('partial mocking with spyOn', () => {
  afterEach(() => {
    vi.restoreAllMocks();
  });

  it('mocks single function while keeping others real', () => {
    // Only mock the expensive function
    vi.spyOn(utils, 'expensiveCalculation').mockReturnValue(42);

    // Real implementation
    expect(utils.simpleAdd(1, 2)).toBe(3);

    // Mocked
    expect(utils.expensiveCalculation()).toBe(42);
  });
});
```

### Partial Mock with Dynamic Behavior

```typescript
import { vi, expect, it, describe } from 'vitest';

vi.mock('./api', async (importOriginal) => {
  const actual = await importOriginal<typeof import('./api')>();

  return {
    ...actual,
    fetchUser: vi.fn().mockImplementation(async (id: number) => {
      // Use mock for specific IDs, real implementation for others
      if (id === 999) {
        return { id: 999, name: 'Mock User', isTest: true };
      }
      return actual.fetchUser(id);
    }),
  };
});

import { fetchUser, fetchProducts } from './api';

describe('partial mock with conditional behavior', () => {
  it('returns mock for specific ID', async () => {
    const user = await fetchUser(999);
    expect(user.isTest).toBe(true);
  });

  it('uses real implementation for other IDs', async () => {
    const user = await fetchUser(1);
    expect(user.isTest).toBeUndefined();
  });

  it('fetchProducts uses real implementation', async () => {
    const products = await fetchProducts();
    expect(products).toBeDefined(); // Real data
  });
});
```

### Partial Mock of Default Export

```typescript
import { vi, expect, it, describe } from 'vitest';

vi.mock('./apiClient', async (importOriginal) => {
  const actual = await importOriginal<typeof import('./apiClient')>();
  const actualDefault = actual.default;

  return {
    ...actual,
    default: {
      ...actualDefault,
      // Only mock the post method
      post: vi.fn().mockResolvedValue({ success: true }),
    },
  };
});

import apiClient from './apiClient';

describe('partial mock of default export', () => {
  it('uses mocked post method', async () => {
    const result = await apiClient.post('/data', { test: true });
    expect(result).toEqual({ success: true });
  });

  it('uses real get method', async () => {
    const result = await apiClient.get('/users');
    // Uses real implementation
    expect(result).toBeDefined();
  });
});
```

---

## 17. Testing React Components with Mocks

This section covers patterns for testing React components using Vitest mocks.

### Mocking React Hooks

```typescript
import { vi, expect, it, describe, beforeEach } from 'vitest';
import { render, screen } from '@testing-library/react';

// Mock the custom hook
vi.mock('./useUser', () => ({
  useUser: vi.fn(),
}));

import { useUser } from './useUser';
import { UserProfile } from './UserProfile';

describe('UserProfile component', () => {
  beforeEach(() => {
    vi.mocked(useUser).mockReturnValue({
      user: null,
      loading: false,
      error: null,
    });
  });

  it('shows loading state', () => {
    vi.mocked(useUser).mockReturnValue({
      user: null,
      loading: true,
      error: null,
    });

    render(<UserProfile userId={1} />);

    expect(screen.getByText('Loading...')).toBeInTheDocument();
  });

  it('shows user data', () => {
    vi.mocked(useUser).mockReturnValue({
      user: { id: 1, name: 'John Doe', email: 'john@example.com' },
      loading: false,
      error: null,
    });

    render(<UserProfile userId={1} />);

    expect(screen.getByText('John Doe')).toBeInTheDocument();
    expect(screen.getByText('john@example.com')).toBeInTheDocument();
  });

  it('shows error state', () => {
    vi.mocked(useUser).mockReturnValue({
      user: null,
      loading: false,
      error: new Error('Failed to fetch user'),
    });

    render(<UserProfile userId={1} />);

    expect(screen.getByText('Error: Failed to fetch user')).toBeInTheDocument();
  });
});
```

### Mocking API Calls in Components

```typescript
import { vi, expect, it, describe, beforeEach } from 'vitest';
import { render, screen, waitFor } from '@testing-library/react';
import userEvent from '@testing-library/user-event';

vi.mock('./api', () => ({
  fetchUsers: vi.fn(),
  createUser: vi.fn(),
  deleteUser: vi.fn(),
}));

import { fetchUsers, createUser, deleteUser } from './api';
import { UserList } from './UserList';

describe('UserList component', () => {
  beforeEach(() => {
    vi.clearAllMocks();
  });

  it('fetches and displays users', async () => {
    vi.mocked(fetchUsers).mockResolvedValue([
      { id: 1, name: 'Alice' },
      { id: 2, name: 'Bob' },
    ]);

    render(<UserList />);

    await waitFor(() => {
      expect(screen.getByText('Alice')).toBeInTheDocument();
      expect(screen.getByText('Bob')).toBeInTheDocument();
    });

    expect(fetchUsers).toHaveBeenCalledTimes(1);
  });

  it('handles user creation', async () => {
    const user = userEvent.setup();

    vi.mocked(fetchUsers).mockResolvedValue([]);
    vi.mocked(createUser).mockResolvedValue({ id: 3, name: 'Charlie' });

    render(<UserList />);

    await user.type(screen.getByPlaceholderText('Enter name'), 'Charlie');
    await user.click(screen.getByText('Add User'));

    await waitFor(() => {
      expect(createUser).toHaveBeenCalledWith({ name: 'Charlie' });
    });
  });

  it('handles API errors gracefully', async () => {
    vi.mocked(fetchUsers).mockRejectedValue(new Error('Network error'));

    render(<UserList />);

    await waitFor(() => {
      expect(screen.getByText('Failed to load users')).toBeInTheDocument();
    });
  });
});
```

### Mocking Context Providers

```typescript
import { vi, expect, it, describe } from 'vitest';
import { render, screen } from '@testing-library/react';
import { AuthContext } from './AuthContext';
import { ProtectedComponent } from './ProtectedComponent';

describe('ProtectedComponent', () => {
  const mockAuthContext = {
    user: null as { id: number; name: string } | null,
    isAuthenticated: false,
    login: vi.fn(),
    logout: vi.fn(),
  };

  const renderWithAuth = (authValue = mockAuthContext) => {
    return render(
      <AuthContext.Provider value={authValue}>
        <ProtectedComponent />
      </AuthContext.Provider>
    );
  };

  it('shows login prompt when not authenticated', () => {
    renderWithAuth();

    expect(screen.getByText('Please log in')).toBeInTheDocument();
  });

  it('shows content when authenticated', () => {
    renderWithAuth({
      ...mockAuthContext,
      user: { id: 1, name: 'John' },
      isAuthenticated: true,
    });

    expect(screen.getByText('Welcome, John!')).toBeInTheDocument();
  });

  it('calls logout when button clicked', async () => {
    const logout = vi.fn();
    const user = userEvent.setup();

    renderWithAuth({
      ...mockAuthContext,
      user: { id: 1, name: 'John' },
      isAuthenticated: true,
      logout,
    });

    await user.click(screen.getByText('Logout'));

    expect(logout).toHaveBeenCalled();
  });
});
```

### Mocking React Router

```typescript
import { vi, expect, it, describe, beforeEach } from 'vitest';
import { render, screen } from '@testing-library/react';
import { MemoryRouter, useNavigate, useParams } from 'react-router-dom';

vi.mock('react-router-dom', async () => {
  const actual = await vi.importActual('react-router-dom');
  return {
    ...actual,
    useNavigate: vi.fn(),
    useParams: vi.fn(),
  };
});

import { UserPage } from './UserPage';

describe('UserPage', () => {
  const mockNavigate = vi.fn();

  beforeEach(() => {
    vi.mocked(useNavigate).mockReturnValue(mockNavigate);
    vi.mocked(useParams).mockReturnValue({ userId: '123' });
  });

  it('uses route params', () => {
    render(
      <MemoryRouter>
        <UserPage />
      </MemoryRouter>
    );

    expect(screen.getByText('User ID: 123')).toBeInTheDocument();
  });

  it('navigates on button click', async () => {
    const user = userEvent.setup();

    render(
      <MemoryRouter>
        <UserPage />
      </MemoryRouter>
    );

    await user.click(screen.getByText('Go Back'));

    expect(mockNavigate).toHaveBeenCalledWith(-1);
  });
});
```

### Mocking Child Components

```typescript
import { vi, expect, it, describe } from 'vitest';
import { render, screen } from '@testing-library/react';

// Mock heavy child component
vi.mock('./HeavyChart', () => ({
  HeavyChart: vi.fn(({ data, title }) => (
    <div data-testid="mock-chart">
      Chart: {title} ({data.length} items)
    </div>
  )),
}));

import { Dashboard } from './Dashboard';
import { HeavyChart } from './HeavyChart';

describe('Dashboard', () => {
  it('passes correct props to chart', () => {
    render(<Dashboard />);

    expect(HeavyChart).toHaveBeenCalledWith(
      expect.objectContaining({
        title: 'Sales Data',
        data: expect.any(Array),
      }),
      expect.anything()
    );

    expect(screen.getByTestId('mock-chart')).toBeInTheDocument();
  });
});
```

---

## 18. Best Practices

### 1. Always Clean Up Mocks

```typescript
import { vi, describe, beforeEach, afterEach } from 'vitest';

describe('proper cleanup', () => {
  beforeEach(() => {
    vi.clearAllMocks();
  });

  afterEach(() => {
    vi.restoreAllMocks();
    vi.unstubAllGlobals();
    vi.unstubAllEnvs();
    vi.useRealTimers();
  });

  // Tests...
});
```

### 2. Use Type-Safe Mocks

```typescript
import { vi, expect, it } from 'vitest';

// Define types for better IDE support and type safety
interface UserService {
  getUser(id: number): Promise<User>;
  createUser(data: CreateUserData): Promise<User>;
}

const mockUserService: UserService = {
  getUser: vi.fn(),
  createUser: vi.fn(),
};

// TypeScript will catch errors if mock doesn't match interface
vi.mocked(mockUserService.getUser).mockResolvedValue({
  id: 1,
  name: 'Test',
  email: 'test@example.com',
});
```

### 3. Prefer spyOn for Existing Objects

```typescript
import { vi, expect, it, afterEach } from 'vitest';

// Good: Spy on existing method
afterEach(() => {
  vi.restoreAllMocks();
});

it('tracks console.log calls', () => {
  const spy = vi.spyOn(console, 'log').mockImplementation(() => {});

  // Code under test
  console.log('test message');

  expect(spy).toHaveBeenCalledWith('test message');
});
```

### 4. Mock at the Right Level

```typescript
import { vi, expect, it } from 'vitest';

// Bad: Mocking too low-level (implementation details)
vi.mock('./database/connection');
vi.mock('./database/query-builder');
vi.mock('./database/transaction');

// Good: Mock at the boundary
vi.mock('./database', () => ({
  query: vi.fn(),
  transaction: vi.fn(),
}));
```

### 5. Keep Mock Data Realistic

```typescript
import { vi, expect, it } from 'vitest';

// Bad: Unrealistic mock data
const mockUser = { id: 1 };

// Good: Realistic mock data that matches production shapes
const mockUser = {
  id: 1,
  name: 'John Doe',
  email: 'john@example.com',
  createdAt: '2024-01-15T10:30:00Z',
  updatedAt: '2024-01-15T10:30:00Z',
  roles: ['user'],
  preferences: {
    theme: 'dark',
    notifications: true,
  },
};
```

### 6. Use Factory Functions for Mock Data

```typescript
import { vi } from 'vitest';

// Factory function for creating test users
function createMockUser(overrides: Partial<User> = {}): User {
  return {
    id: Math.floor(Math.random() * 1000),
    name: 'Test User',
    email: 'test@example.com',
    createdAt: new Date().toISOString(),
    ...overrides,
  };
}

// Usage in tests
const admin = createMockUser({ roles: ['admin'] });
const newUser = createMockUser({ createdAt: new Date().toISOString() });
```

### 7. Avoid Over-Mocking

```typescript
import { vi, expect, it } from 'vitest';

// Bad: Mocking everything, tests become meaningless
vi.mock('./userService');
vi.mock('./validation');
vi.mock('./utils');
vi.mock('./constants');

// Good: Only mock external dependencies
vi.mock('./api'); // External API calls
// Let validation, utils, constants use real implementations
```

### 8. Test Mock Interactions

```typescript
import { vi, expect, it } from 'vitest';

it('calls API with correct parameters', async () => {
  const mockFetch = vi.fn().mockResolvedValue({ data: [] });

  await fetchUsersWithFilters(mockFetch, {
    role: 'admin',
    active: true,
  });

  // Verify HOW the mock was called, not just that it was called
  expect(mockFetch).toHaveBeenCalledWith('/api/users', {
    method: 'GET',
    headers: expect.objectContaining({
      'Content-Type': 'application/json',
    }),
    params: {
      role: 'admin',
      active: 'true',
    },
  });
});
```

### 9. Document Complex Mocks

```typescript
import { vi } from 'vitest';

/**
 * Mock for the PaymentService that simulates:
 * - Successful payments for amounts < $1000
 * - Declined payments for amounts >= $1000
 * - Network errors for amount === $666
 */
vi.mock('./paymentService', () => ({
  processPayment: vi.fn().mockImplementation(async (amount: number) => {
    if (amount === 666) {
      throw new Error('Network error');
    }
    if (amount >= 1000) {
      return { success: false, error: 'Payment declined' };
    }
    return { success: true, transactionId: `txn_${Date.now()}` };
  }),
}));
```

### 10. Use Mock Matchers Effectively

```typescript
import { vi, expect, it } from 'vitest';

it('uses appropriate matchers', () => {
  const mockFn = vi.fn();

  mockFn({ id: 1, timestamp: Date.now() });

  // Use expect.any() for dynamic values
  expect(mockFn).toHaveBeenCalledWith({
    id: 1,
    timestamp: expect.any(Number),
  });

  // Use expect.objectContaining() for partial matches
  expect(mockFn).toHaveBeenCalledWith(
    expect.objectContaining({ id: 1 })
  );

  // Use expect.stringContaining() for partial strings
  mockFn('Error: Something went wrong at line 42');
  expect(mockFn).toHaveBeenCalledWith(
    expect.stringContaining('Something went wrong')
  );
});
```

---

## Quick Reference

### Mock Creation

| Method | Description |
|--------|-------------|
| `vi.fn()` | Create a mock function |
| `vi.fn(impl)` | Create mock with implementation |
| `vi.spyOn(obj, 'method')` | Spy on object method |
| `vi.spyOn(obj, 'prop', 'get')` | Spy on getter |
| `vi.spyOn(obj, 'prop', 'set')` | Spy on setter |

### Mock Configuration

| Method | Description |
|--------|-------------|
| `mockReturnValue(val)` | Always return value |
| `mockReturnValueOnce(val)` | Return value once |
| `mockResolvedValue(val)` | Return resolved promise |
| `mockResolvedValueOnce(val)` | Return resolved promise once |
| `mockRejectedValue(err)` | Return rejected promise |
| `mockRejectedValueOnce(err)` | Return rejected promise once |
| `mockImplementation(fn)` | Set implementation |
| `mockImplementationOnce(fn)` | Set implementation once |

### Mock Properties

| Property | Description |
|----------|-------------|
| `mock.calls` | Array of call arguments |
| `mock.results` | Array of return values |
| `mock.instances` | Array of `this` values (constructors) |
| `mock.contexts` | Array of `this` contexts |
| `mock.lastCall` | Arguments of last call |

### Mock Reset

| Method | Description |
|--------|-------------|
| `mockClear()` | Clear call history |
| `mockReset()` | Clear history + reset return value |
| `mockRestore()` | Restore original (spyOn only) |
| `vi.clearAllMocks()` | Clear all mocks |
| `vi.resetAllMocks()` | Reset all mocks |
| `vi.restoreAllMocks()` | Restore all mocks |

### Module Mocking

| Method | Description |
|--------|-------------|
| `vi.mock(path)` | Auto-mock module (hoisted) |
| `vi.mock(path, factory)` | Mock with factory (hoisted) |
| `vi.doMock(path)` | Mock module (not hoisted) |
| `vi.doUnmock(path)` | Remove module mock |
| `vi.importActual(path)` | Import real module |
| `vi.importMock(path)` | Import auto-mocked module |
| `vi.resetModules()` | Reset module registry |

### Timer Mocking

| Method | Description |
|--------|-------------|
| `vi.useFakeTimers()` | Enable fake timers |
| `vi.useRealTimers()` | Restore real timers |
| `vi.advanceTimersByTime(ms)` | Advance time |
| `vi.advanceTimersToNextTimer()` | Advance to next timer |
| `vi.runAllTimers()` | Run all timers |
| `vi.runOnlyPendingTimers()` | Run pending timers |
| `vi.setSystemTime(date)` | Set system time |
| `vi.getTimerCount()` | Get pending timer count |

### Global Stubbing

| Method | Description |
|--------|-------------|
| `vi.stubGlobal(name, value)` | Stub global variable |
| `vi.stubEnv(name, value)` | Stub environment variable |
| `vi.unstubAllGlobals()` | Restore all globals |
| `vi.unstubAllEnvs()` | Restore all env vars |

---

## Additional Resources

- [Vitest Documentation](https://vitest.dev/)
- [Vitest Mocking Guide](https://vitest.dev/guide/mocking.html)
- [Vitest API Reference](https://vitest.dev/api/vi.html)
- [Testing Library](https://testing-library.com/)
