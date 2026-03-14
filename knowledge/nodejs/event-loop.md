# Node.js Event Loop

> Source: https://nodejs.org/en/learn/asynchronous-work/event-loop-timers-and-nexttick

## Overview

The event loop is what allows Node.js to perform non-blocking I/O operations despite JavaScript being single-threaded. It offloads operations to the system kernel whenever possible.

## Event Loop Phases

```
   ┌───────────────────────────┐
┌─>│           timers          │  ← setTimeout, setInterval callbacks
│  └─────────────┬─────────────┘
│  ┌─────────────┴─────────────┐
│  │     pending callbacks     │  ← I/O callbacks deferred to next iteration
│  └─────────────┬─────────────┘
│  ┌─────────────┴─────────────┐
│  │       idle, prepare       │  ← internal use only
│  └─────────────┬─────────────┘
│  ┌─────────────┴─────────────┐
│  │           poll            │  ← retrieve new I/O events; execute I/O callbacks
│  └─────────────┬─────────────┘
│  ┌─────────────┴─────────────┐
│  │           check           │  ← setImmediate callbacks
│  └─────────────┬─────────────┘
│  ┌─────────────┴─────────────┐
└──┤      close callbacks      │  ← socket.on('close'), etc.
   └───────────────────────────┘
```

### Timers Phase

Executes callbacks scheduled by `setTimeout()` and `setInterval()`. A timer specifies the threshold after which a callback may be executed, not the exact time.

### Pending Callbacks Phase

Executes I/O callbacks deferred to the next loop iteration (e.g., TCP errors).

### Poll Phase

- Retrieves new I/O events
- Executes I/O related callbacks (almost all except close callbacks, timers, and setImmediate)
- Node.js will block here when appropriate

### Check Phase

`setImmediate()` callbacks are invoked here. Designed to execute callbacks after the poll phase.

### Close Callbacks Phase

Close callbacks (e.g., `socket.on('close', ...)`) are executed here.

## process.nextTick()

`process.nextTick()` is not technically part of the event loop. The `nextTickQueue` is processed after the current operation completes, regardless of the current phase.

```javascript
// nextTick runs before any I/O
process.nextTick(() => {
  console.log('nextTick');
});

setImmediate(() => {
  console.log('setImmediate');
});

// Output: nextTick, setImmediate
```

### Warning

Recursive `process.nextTick()` calls can starve I/O by preventing the event loop from reaching the poll phase.

```javascript
// BAD - starves I/O
function badRecursive() {
  process.nextTick(badRecursive);
}

// GOOD - use setImmediate for recursive operations
function goodRecursive() {
  setImmediate(goodRecursive);
}
```

## Microtasks vs Macrotasks

### Priority Order

```
1. sync code
2. process.nextTick (nextTick queue)
3. Promise callbacks (microtask queue)
4. setTimeout/setInterval (timers)
5. setImmediate (check phase)
6. I/O callbacks
```

### Example

```javascript
console.log('1 - sync');

process.nextTick(() => console.log('2 - nextTick'));

Promise.resolve().then(() => console.log('3 - Promise'));

setTimeout(() => console.log('4 - setTimeout'), 0);

setImmediate(() => console.log('5 - setImmediate'));

// Output: 1, 2, 3, 4, 5 (4 and 5 may swap depending on timing)
```

## Best Practices

### Don't Block the Event Loop

```javascript
// BAD - blocking
const hash = crypto.pbkdf2Sync(password, salt, 100000, 64, 'sha512');

// GOOD - async
crypto.pbkdf2(password, salt, 100000, 64, 'sha512', (err, hash) => {
  // use hash
});

// GOOD - promisified
const { promisify } = require('util');
const pbkdf2 = promisify(crypto.pbkdf2);
const hash = await pbkdf2(password, salt, 100000, 64, 'sha512');
```

### Use setImmediate for Recursive Operations

```javascript
// Process large array without blocking
function processArray(array, index = 0) {
  if (index >= array.length) return;

  // Process item
  processItem(array[index]);

  // Yield to event loop
  setImmediate(() => processArray(array, index + 1));
}
```

## Monitoring Event Loop Lag

```javascript
const start = process.hrtime.bigint();

setImmediate(() => {
  const lag = Number(process.hrtime.bigint() - start) / 1e6;
  console.log(`Event loop lag: ${lag}ms`);
});
```

## Node.js 20+ Changes

Starting with libuv 1.45.0 (Node.js 20), timers only run after the poll phase, not before and after. This can affect timing between `setImmediate()` and timers.
