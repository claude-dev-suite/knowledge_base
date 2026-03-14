# Jest Mocking Deep Dive

Comprehensive guide to mocking in Jest.

**Official Documentation:** https://jestjs.io/docs/mock-functions

---

## Table of Contents

1. [Mock Functions](#mock-functions)
2. [Spying](#spying)
3. [Module Mocking](#module-mocking)
4. [Manual Mocks](#manual-mocks)
5. [Timer Mocks](#timer-mocks)
6. [ES6 Class Mocking](#es6-class-mocking)
7. [Common Patterns](#common-patterns)
8. [Best Practices](#best-practices)

---

## Mock Functions

### Creating Mock Functions

```javascript
// Basic mock
const mockFn = jest.fn();

// Mock with implementation
const mockAdd = jest.fn((a, b) => a + b);

// Mock returning specific value
const mockGetter = jest.fn().mockReturnValue(42);

// Mock returning different values on each call
const mockSequence = jest.fn()
  .mockReturnValueOnce('first')
  .mockReturnValueOnce('second')
  .mockReturnValue('default');

console.log(mockSequence()); // 'first'
console.log(mockSequence()); // 'second'
console.log(mockSequence()); // 'default'
console.log(mockSequence()); // 'default'
```

### Async Mock Functions

```javascript
// Mock resolved value
const mockFetch = jest.fn().mockResolvedValue({ data: [] });

// Mock resolved values in sequence
const mockApi = jest.fn()
  .mockResolvedValueOnce({ data: 'first' })
  .mockResolvedValueOnce({ data: 'second' });

// Mock rejection
const mockFailingFetch = jest.fn().mockRejectedValue(new Error('Network error'));

// Mix resolved and rejected
const mockUnstable = jest.fn()
  .mockResolvedValueOnce({ data: 'success' })
  .mockRejectedValueOnce(new Error('failed'))
  .mockResolvedValue({ data: 'recovered' });
```

### Mock Implementation

```javascript
// Set implementation
const mockCalculate = jest.fn().mockImplementation((a, b) => a * b);

// One-time implementation
const mock = jest.fn()
  .mockImplementationOnce(() => 'first call')
  .mockImplementationOnce(() => 'second call')
  .mockImplementation(() => 'default');

// Access to this and arguments
const mockCallback = jest.fn(function(x) {
  return this.value + x;
});
```

### Mock Properties and Methods

```javascript
const mock = jest.fn();

// mock.calls - Array of call arguments
mock('first', 'second');
mock('third');
console.log(mock.mock.calls);
// [['first', 'second'], ['third']]
console.log(mock.mock.calls[0][0]); // 'first'

// mock.results - Array of return values
const mockAdd = jest.fn(x => x + 1);
mockAdd(1);
mockAdd(2);
console.log(mockAdd.mock.results);
// [{ type: 'return', value: 2 }, { type: 'return', value: 3 }]

// mock.instances - Array of instances (when used with new)
const MockClass = jest.fn();
const instance1 = new MockClass();
const instance2 = new MockClass();
console.log(MockClass.mock.instances);
// [instance1, instance2]

// mock.contexts - Array of this values
const mock = jest.fn();
const obj = { method: mock };
obj.method();
console.log(mock.mock.contexts[0]); // obj

// mock.lastCall - Last call arguments
mock('a', 'b');
console.log(mock.mock.lastCall); // ['a', 'b']
```

### Mock Assertions

```javascript
const mockFn = jest.fn();
mockFn('arg1', 'arg2');
mockFn('arg3');

// Call assertions
expect(mockFn).toHaveBeenCalled();
expect(mockFn).toHaveBeenCalledTimes(2);
expect(mockFn).toHaveBeenCalledWith('arg1', 'arg2');
expect(mockFn).toHaveBeenLastCalledWith('arg3');
expect(mockFn).toHaveBeenNthCalledWith(1, 'arg1', 'arg2');

// Return assertions
const mockReturn = jest.fn(() => 42);
mockReturn();
expect(mockReturn).toHaveReturned();
expect(mockReturn).toHaveReturnedTimes(1);
expect(mockReturn).toHaveReturnedWith(42);
expect(mockReturn).toHaveLastReturnedWith(42);
expect(mockReturn).toHaveNthReturnedWith(1, 42);
```

### Clearing and Resetting

```javascript
const mock = jest.fn(() => 'value');
mock('test');

// Clear - removes call history, keeps implementation
mock.mockClear();
expect(mock).not.toHaveBeenCalled();
expect(mock()).toBe('value'); // Implementation preserved

// Reset - clears + removes implementation
mock.mockReset();
expect(mock()).toBeUndefined(); // No implementation

// Restore - for spies, restores original
const obj = { method: () => 'original' };
const spy = jest.spyOn(obj, 'method').mockReturnValue('mocked');
spy.mockRestore();
expect(obj.method()).toBe('original');

// Clear all mocks
jest.clearAllMocks();
jest.resetAllMocks();
jest.restoreAllMocks();
```

---

## Spying

### Basic Spying

```javascript
const calculator = {
  add: (a, b) => a + b,
  subtract: (a, b) => a - b,
};

// Spy on method (preserves implementation)
const addSpy = jest.spyOn(calculator, 'add');

calculator.add(1, 2); // Returns 3 (original)

expect(addSpy).toHaveBeenCalled();
expect(addSpy).toHaveBeenCalledWith(1, 2);

// Restore
addSpy.mockRestore();
```

### Spy with Mock Implementation

```javascript
const api = {
  fetchUsers: async () => {
    // Real API call
    const response = await fetch('/api/users');
    return response.json();
  },
};

// Override implementation
jest.spyOn(api, 'fetchUsers').mockResolvedValue([{ id: 1, name: 'John' }]);

// Now returns mock data
const users = await api.fetchUsers();
expect(users).toEqual([{ id: 1, name: 'John' }]);
```

### Spy on Getters/Setters

```javascript
const video = {
  _playing: false,
  get playing() {
    return this._playing;
  },
  set playing(value) {
    this._playing = value;
  },
};

// Spy on getter
const getSpy = jest.spyOn(video, 'playing', 'get').mockReturnValue(true);
expect(video.playing).toBe(true);

// Spy on setter
const setSpy = jest.spyOn(video, 'playing', 'set');
video.playing = true;
expect(setSpy).toHaveBeenCalledWith(true);
```

### Spy on Built-in Methods

```javascript
// Console
const consoleSpy = jest.spyOn(console, 'log').mockImplementation();
doSomething();
expect(consoleSpy).toHaveBeenCalledWith('expected message');
consoleSpy.mockRestore();

// Math
const randomSpy = jest.spyOn(Math, 'random').mockReturnValue(0.5);
expect(Math.random()).toBe(0.5);
randomSpy.mockRestore();

// Date
const dateSpy = jest.spyOn(global, 'Date').mockImplementation(() => ({
  getTime: () => 1234567890,
}));
```

---

## Module Mocking

### Basic Module Mock

```javascript
// api.js
export const fetchUsers = async () => {
  const response = await fetch('/api/users');
  return response.json();
};

// test.js
jest.mock('./api');
import { fetchUsers } from './api';

test('uses mocked module', async () => {
  fetchUsers.mockResolvedValue([{ id: 1 }]);
  const users = await fetchUsers();
  expect(users).toEqual([{ id: 1 }]);
});
```

### Mock with Factory

```javascript
jest.mock('./api', () => ({
  fetchUsers: jest.fn().mockResolvedValue([]),
  createUser: jest.fn().mockResolvedValue({ id: 1 }),
  __esModule: true, // For ES modules
}));
```

### Partial Mock

```javascript
jest.mock('./utils', () => {
  const actual = jest.requireActual('./utils');
  return {
    ...actual,
    formatDate: jest.fn().mockReturnValue('2024-01-01'),
  };
});

// Or using spyOn
import * as utils from './utils';
jest.spyOn(utils, 'formatDate').mockReturnValue('2024-01-01');
```

### Mock Node Modules

```javascript
// Mock axios
jest.mock('axios');
import axios from 'axios';

test('fetches data', async () => {
  axios.get.mockResolvedValue({ data: { users: [] } });
  const result = await fetchData();
  expect(axios.get).toHaveBeenCalledWith('/api/data');
});
```

### Dynamic Mocks

```javascript
// Different mock per test
import { fetchData } from './api';

jest.mock('./api');

describe('with success', () => {
  beforeEach(() => {
    fetchData.mockResolvedValue({ data: 'success' });
  });

  test('handles success', async () => {
    const result = await fetchData();
    expect(result.data).toBe('success');
  });
});

describe('with failure', () => {
  beforeEach(() => {
    fetchData.mockRejectedValue(new Error('failed'));
  });

  test('handles failure', async () => {
    await expect(fetchData()).rejects.toThrow('failed');
  });
});
```

---

## Manual Mocks

### Creating Manual Mocks

```
project/
├── __mocks__/
│   └── axios.js        # Mocks node_modules/axios
├── src/
│   ├── __mocks__/
│   │   └── api.js      # Mocks src/api.js
│   └── api.js
```

```javascript
// __mocks__/axios.js
export default {
  get: jest.fn().mockResolvedValue({ data: {} }),
  post: jest.fn().mockResolvedValue({ data: {} }),
  create: jest.fn(() => ({
    get: jest.fn().mockResolvedValue({ data: {} }),
    post: jest.fn().mockResolvedValue({ data: {} }),
  })),
};

// src/__mocks__/api.js
export const fetchUsers = jest.fn().mockResolvedValue([]);
export const createUser = jest.fn().mockResolvedValue({ id: 1 });
```

### Using Manual Mocks

```javascript
// For user modules, must call jest.mock()
jest.mock('../api');

// For node_modules, automatic if __mocks__ exists
// But you can still call jest.mock() explicitly
jest.mock('axios');

// Use unmocked version
jest.unmock('axios');
```

---

## Timer Mocks

### Basic Timer Mocking

```javascript
jest.useFakeTimers();

test('calls callback after 1 second', () => {
  const callback = jest.fn();

  setTimeout(callback, 1000);

  expect(callback).not.toHaveBeenCalled();

  jest.advanceTimersByTime(1000);

  expect(callback).toHaveBeenCalledTimes(1);
});

test('with intervals', () => {
  const callback = jest.fn();

  setInterval(callback, 1000);

  jest.advanceTimersByTime(3000);

  expect(callback).toHaveBeenCalledTimes(3);
});
```

### Timer Methods

```javascript
// Advance by time
jest.advanceTimersByTime(1000);

// Run all timers
jest.runAllTimers();

// Run only pending timers
jest.runOnlyPendingTimers();

// Advance to next timer
jest.advanceTimersToNextTimer();
jest.advanceTimersToNextTimer(3); // Advance 3 timers

// Clear all timers
jest.clearAllTimers();

// Get pending timers count
jest.getTimerCount();
```

### Modern Fake Timers

```javascript
// Use modern implementation (recommended)
jest.useFakeTimers('modern');

// Or in config
module.exports = {
  timers: 'modern',
};

// Mock Date
jest.setSystemTime(new Date('2024-01-01'));

// Get real system time
const realNow = jest.getRealSystemTime();

// Restore
jest.useRealTimers();
```

### Async Timer Testing

```javascript
jest.useFakeTimers();

test('async timer', async () => {
  const callback = jest.fn();

  const promise = new Promise((resolve) => {
    setTimeout(() => {
      callback();
      resolve('done');
    }, 1000);
  });

  jest.advanceTimersByTime(1000);

  await promise;
  expect(callback).toHaveBeenCalled();
});

// Or with runAllTimersAsync
test('async timer v2', async () => {
  const callback = jest.fn();

  setTimeout(async () => {
    await asyncOperation();
    callback();
  }, 1000);

  await jest.runAllTimersAsync();
  expect(callback).toHaveBeenCalled();
});
```

---

## ES6 Class Mocking

### Basic Class Mock

```javascript
// sound-player.js
export default class SoundPlayer {
  constructor() {
    this.foo = 'bar';
  }
  playSoundFile(fileName) {
    console.log('Playing sound file ' + fileName);
  }
}

// test.js
jest.mock('./sound-player');
import SoundPlayer from './sound-player';
import SoundPlayerConsumer from './sound-player-consumer';

test('mocks class', () => {
  const consumer = new SoundPlayerConsumer();
  consumer.playSomethingCool();

  expect(SoundPlayer).toHaveBeenCalledTimes(1);

  const mockInstance = SoundPlayer.mock.instances[0];
  expect(mockInstance.playSoundFile).toHaveBeenCalledWith('cool.mp3');
});
```

### Manual Class Mock

```javascript
// __mocks__/sound-player.js
export const mockPlaySoundFile = jest.fn();

const mock = jest.fn().mockImplementation(() => ({
  playSoundFile: mockPlaySoundFile,
}));

export default mock;
```

### Mock Class Methods

```javascript
jest.mock('./sound-player', () => {
  return jest.fn().mockImplementation(() => ({
    playSoundFile: jest.fn(),
    stop: jest.fn(),
  }));
});

// Or mock specific method
SoundPlayer.prototype.playSoundFile = jest.fn();
```

---

## Common Patterns

### Testing API Calls

```javascript
import axios from 'axios';
import { fetchUsers } from './api';

jest.mock('axios');

describe('fetchUsers', () => {
  afterEach(() => {
    jest.clearAllMocks();
  });

  test('returns users on success', async () => {
    const users = [{ id: 1, name: 'John' }];
    axios.get.mockResolvedValue({ data: users });

    const result = await fetchUsers();

    expect(axios.get).toHaveBeenCalledWith('/api/users');
    expect(result).toEqual(users);
  });

  test('throws on failure', async () => {
    axios.get.mockRejectedValue(new Error('Network error'));

    await expect(fetchUsers()).rejects.toThrow('Network error');
  });
});
```

### Testing with Context/Providers

```javascript
import { render } from '@testing-library/react';
import { UserContext } from './context';

const mockContextValue = {
  user: { id: 1, name: 'John' },
  setUser: jest.fn(),
};

test('uses context', () => {
  render(
    <UserContext.Provider value={mockContextValue}>
      <Component />
    </UserContext.Provider>
  );

  expect(mockContextValue.setUser).toHaveBeenCalled();
});
```

### Testing Event Handlers

```javascript
test('calls onClick when clicked', () => {
  const handleClick = jest.fn();
  render(<Button onClick={handleClick}>Click me</Button>);

  fireEvent.click(screen.getByText('Click me'));

  expect(handleClick).toHaveBeenCalledTimes(1);
  expect(handleClick).toHaveBeenCalledWith(expect.any(Object)); // event
});
```

### Testing Debounced Functions

```javascript
jest.useFakeTimers();

test('debounced search', () => {
  const onSearch = jest.fn();
  render(<SearchInput onSearch={onSearch} debounceMs={300} />);

  fireEvent.change(screen.getByRole('textbox'), { target: { value: 'test' } });

  expect(onSearch).not.toHaveBeenCalled();

  jest.advanceTimersByTime(300);

  expect(onSearch).toHaveBeenCalledWith('test');
});
```

---

## Best Practices

### Do

```javascript
// Clear mocks between tests
afterEach(() => {
  jest.clearAllMocks();
});

// Use specific assertions
expect(mock).toHaveBeenCalledWith(expect.objectContaining({ id: 1 }));

// Mock at the right level
jest.spyOn(api, 'fetchUsers'); // Instead of mocking fetch globally

// Restore mocks after tests
afterEach(() => {
  jest.restoreAllMocks();
});
```

### Don't

```javascript
// Don't forget to await async mocks
const result = mockAsync(); // Missing await!

// Don't over-mock
jest.mock('./module'); // Only mock what you need

// Don't mock what you don't own (usually)
jest.mock('react'); // Avoid unless necessary
```

---

## Quick Reference

| Method | Description |
|--------|-------------|
| `jest.fn()` | Create mock function |
| `jest.spyOn(obj, 'method')` | Spy on method |
| `jest.mock('module')` | Mock module |
| `jest.unmock('module')` | Remove mock |
| `jest.requireActual('module')` | Get real module |
| `mockFn.mockReturnValue(val)` | Set return value |
| `mockFn.mockResolvedValue(val)` | Set async return |
| `mockFn.mockRejectedValue(err)` | Set async error |
| `mockFn.mockImplementation(fn)` | Set implementation |
| `mockFn.mockClear()` | Clear history |
| `mockFn.mockReset()` | Reset mock |
| `mockFn.mockRestore()` | Restore original |
| `jest.useFakeTimers()` | Enable timer mocks |
| `jest.useRealTimers()` | Restore real timers |
| `jest.advanceTimersByTime(ms)` | Advance fake timers |
| `jest.runAllTimers()` | Run all timers |
| `jest.setSystemTime(date)` | Mock Date |
