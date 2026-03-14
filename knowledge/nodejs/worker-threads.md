# Node.js Worker Threads

> Source: https://nodejs.org/api/worker_threads.html

## Overview

Worker threads enable parallel JavaScript execution. Unlike `child_process` or `cluster`, workers can share memory using `SharedArrayBuffer` and transfer data efficiently using `ArrayBuffer`.

## When to Use Worker Threads

**Use for:**
- CPU-intensive operations (hashing, compression, parsing)
- Image/video processing
- Complex calculations
- Any operation that would block the event loop

**Don't use for:**
- I/O-bound operations (use async I/O instead)
- Simple operations that complete quickly

## Basic Usage

### Main Thread

```javascript
const { Worker, isMainThread, parentPort, workerData } = require('worker_threads');

if (isMainThread) {
  // Main thread
  const worker = new Worker(__filename, {
    workerData: { value: 42 }
  });

  worker.on('message', (result) => {
    console.log('Result:', result);
  });

  worker.on('error', (err) => {
    console.error('Worker error:', err);
  });

  worker.on('exit', (code) => {
    if (code !== 0) {
      console.error(`Worker stopped with exit code ${code}`);
    }
  });
} else {
  // Worker thread
  const { value } = workerData;
  const result = heavyComputation(value);
  parentPort.postMessage(result);
}
```

### Separate Worker File

**main.js:**
```javascript
const { Worker } = require('worker_threads');

function runWorker(data) {
  return new Promise((resolve, reject) => {
    const worker = new Worker('./worker.js', { workerData: data });
    worker.on('message', resolve);
    worker.on('error', reject);
    worker.on('exit', (code) => {
      if (code !== 0) {
        reject(new Error(`Worker stopped with exit code ${code}`));
      }
    });
  });
}

async function main() {
  const result = await runWorker({ numbers: [1, 2, 3, 4, 5] });
  console.log('Sum:', result);
}

main();
```

**worker.js:**
```javascript
const { workerData, parentPort } = require('worker_threads');

const sum = workerData.numbers.reduce((a, b) => a + b, 0);
parentPort.postMessage(sum);
```

## Worker Pool Pattern

Reuse workers to avoid creation overhead.

```javascript
const { Worker } = require('worker_threads');
const os = require('os');

class WorkerPool {
  constructor(workerPath, poolSize = os.cpus().length) {
    this.workerPath = workerPath;
    this.poolSize = poolSize;
    this.workers = [];
    this.freeWorkers = [];
    this.taskQueue = [];

    this.initialize();
  }

  initialize() {
    for (let i = 0; i < this.poolSize; i++) {
      this.addWorker();
    }
  }

  addWorker() {
    const worker = new Worker(this.workerPath);

    worker.on('message', (result) => {
      const { resolve } = worker.currentTask;
      worker.currentTask = null;
      this.freeWorkers.push(worker);
      resolve(result);
      this.processQueue();
    });

    worker.on('error', (err) => {
      if (worker.currentTask) {
        worker.currentTask.reject(err);
        worker.currentTask = null;
      }
      // Replace dead worker
      this.workers = this.workers.filter(w => w !== worker);
      this.addWorker();
    });

    this.workers.push(worker);
    this.freeWorkers.push(worker);
  }

  execute(data) {
    return new Promise((resolve, reject) => {
      this.taskQueue.push({ data, resolve, reject });
      this.processQueue();
    });
  }

  processQueue() {
    if (this.taskQueue.length === 0 || this.freeWorkers.length === 0) {
      return;
    }

    const worker = this.freeWorkers.pop();
    const task = this.taskQueue.shift();

    worker.currentTask = task;
    worker.postMessage(task.data);
  }

  async close() {
    await Promise.all(this.workers.map(w => w.terminate()));
  }
}

// Usage
const pool = new WorkerPool('./worker.js');

async function main() {
  const results = await Promise.all([
    pool.execute({ task: 'hash', data: 'password1' }),
    pool.execute({ task: 'hash', data: 'password2' }),
    pool.execute({ task: 'hash', data: 'password3' }),
  ]);
  console.log(results);
  await pool.close();
}
```

## Sharing Memory

### SharedArrayBuffer

```javascript
// main.js
const { Worker } = require('worker_threads');

const sharedBuffer = new SharedArrayBuffer(4);
const sharedArray = new Int32Array(sharedBuffer);

const worker = new Worker('./worker.js', {
  workerData: { sharedBuffer }
});

// Wait for worker to modify
setTimeout(() => {
  console.log('Value from worker:', Atomics.load(sharedArray, 0));
}, 1000);
```

```javascript
// worker.js
const { workerData } = require('worker_threads');

const sharedArray = new Int32Array(workerData.sharedBuffer);
Atomics.store(sharedArray, 0, 42);
```

### Transferring ArrayBuffer

Transfer ownership instead of copying (zero-copy).

```javascript
// main.js
const buffer = new ArrayBuffer(1024 * 1024); // 1MB
const worker = new Worker('./worker.js');

// Transfer ownership - buffer becomes unusable in main thread
worker.postMessage({ buffer }, [buffer]);
console.log(buffer.byteLength); // 0 - transferred
```

## MessageChannel

Create custom communication channels between workers.

```javascript
const { Worker, MessageChannel } = require('worker_threads');

const { port1, port2 } = new MessageChannel();

const worker = new Worker('./worker.js', {
  workerData: { port: port1 },
  transferList: [port1]
});

port2.on('message', (msg) => {
  console.log('Received:', msg);
});

port2.postMessage('Hello from main!');
```

## Best Practices

1. **Use worker pools** - Avoid creating/destroying workers repeatedly
2. **Transfer large data** - Use `transferList` for ArrayBuffers
3. **Limit pool size** - Match CPU cores for CPU-bound work
4. **Handle errors** - Always listen for 'error' events
5. **Clean up** - Call `worker.terminate()` when done
6. **Avoid sharing** - SharedArrayBuffer adds complexity; prefer message passing
