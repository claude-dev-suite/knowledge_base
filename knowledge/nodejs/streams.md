# Node.js Streams

> Source: https://nodejs.org/api/stream.html

## Overview

Streams are collections of data that might not be available all at once and don't have to fit in memory. They are especially useful for processing large files or data from external sources.

## Stream Types

| Type | Description | Example |
|------|-------------|---------|
| **Readable** | Source of data | `fs.createReadStream()`, `http.IncomingMessage` |
| **Writable** | Destination for data | `fs.createWriteStream()`, `http.ServerResponse` |
| **Duplex** | Both readable and writable | `net.Socket`, `zlib` streams |
| **Transform** | Duplex that modifies data | `zlib.createGzip()`, `crypto.createCipheriv()` |

## Using Streams

### Reading from Streams

```javascript
const fs = require('fs');

const readable = fs.createReadStream('file.txt');

// Event-based
readable.on('data', (chunk) => {
  console.log(`Received ${chunk.length} bytes`);
});

readable.on('end', () => {
  console.log('No more data');
});

readable.on('error', (err) => {
  console.error('Error:', err);
});
```

### Async Iteration (Recommended)

```javascript
const fs = require('fs');

async function processFile() {
  const readable = fs.createReadStream('file.txt');

  for await (const chunk of readable) {
    console.log(`Received ${chunk.length} bytes`);
  }
}
```

### Writing to Streams

```javascript
const fs = require('fs');

const writable = fs.createWriteStream('output.txt');

writable.write('Hello ');
writable.write('World!');
writable.end(); // Signals no more data

writable.on('finish', () => {
  console.log('Write completed');
});
```

## Pipeline (Recommended)

The `pipeline()` function handles errors and cleanup automatically.

```javascript
const { pipeline } = require('stream/promises');
const fs = require('fs');
const zlib = require('zlib');

// Compress a file
async function compress() {
  await pipeline(
    fs.createReadStream('input.txt'),
    zlib.createGzip(),
    fs.createWriteStream('input.txt.gz')
  );
  console.log('Compression complete');
}

// Decompress a file
async function decompress() {
  await pipeline(
    fs.createReadStream('input.txt.gz'),
    zlib.createGunzip(),
    fs.createWriteStream('output.txt')
  );
}
```

### With Transform

```javascript
const { Transform } = require('stream');

const uppercase = new Transform({
  transform(chunk, encoding, callback) {
    this.push(chunk.toString().toUpperCase());
    callback();
  }
});

await pipeline(
  fs.createReadStream('input.txt'),
  uppercase,
  fs.createWriteStream('output.txt')
);
```

## Backpressure

Backpressure occurs when data is written faster than it can be consumed. Handle it to prevent memory issues.

### Manual Backpressure Handling

```javascript
const readable = fs.createReadStream('large-file.txt');
const writable = fs.createWriteStream('output.txt');

readable.on('data', (chunk) => {
  const canContinue = writable.write(chunk);

  if (!canContinue) {
    // Pause until drain event
    readable.pause();
    writable.once('drain', () => readable.resume());
  }
});

readable.on('end', () => writable.end());
```

### Pipeline Handles It Automatically

```javascript
// pipeline() handles backpressure automatically
await pipeline(readable, transform, writable);
```

## Creating Custom Streams

### Readable Stream

```javascript
const { Readable } = require('stream');

class Counter extends Readable {
  constructor(max) {
    super();
    this.max = max;
    this.current = 0;
  }

  _read() {
    if (this.current <= this.max) {
      this.push(String(this.current++));
    } else {
      this.push(null); // Signal end of stream
    }
  }
}

const counter = new Counter(10);
for await (const num of counter) {
  console.log(num);
}
```

### Writable Stream

```javascript
const { Writable } = require('stream');

class Logger extends Writable {
  _write(chunk, encoding, callback) {
    console.log(`[LOG] ${chunk.toString()}`);
    callback();
  }
}

const logger = new Logger();
logger.write('Hello');
logger.write('World');
logger.end();
```

### Transform Stream

```javascript
const { Transform } = require('stream');

class JSONParser extends Transform {
  constructor() {
    super({ objectMode: true });
  }

  _transform(chunk, encoding, callback) {
    try {
      const obj = JSON.parse(chunk.toString());
      this.push(obj);
      callback();
    } catch (err) {
      callback(err);
    }
  }
}
```

## Object Mode

Streams can work with JavaScript objects instead of buffers.

```javascript
const { Transform } = require('stream');

const objectStream = new Transform({
  objectMode: true,
  transform(obj, encoding, callback) {
    obj.processed = true;
    this.push(obj);
    callback();
  }
});

objectStream.write({ id: 1, name: 'Item 1' });
objectStream.on('data', (obj) => console.log(obj));
// { id: 1, name: 'Item 1', processed: true }
```

## Reading Lines (readline)

```javascript
const fs = require('fs');
const readline = require('readline');

async function processLines(filepath) {
  const rl = readline.createInterface({
    input: fs.createReadStream(filepath),
    crlfDelay: Infinity
  });

  for await (const line of rl) {
    console.log(`Line: ${line}`);
  }
}
```

## Best Practices

1. **Use pipeline()** - Handles errors and cleanup automatically
2. **Handle backpressure** - Don't ignore `write()` return value
3. **Use async iteration** - Cleaner code with `for await...of`
4. **Always handle errors** - Listen for 'error' events or use pipeline
5. **Prefer streams for large data** - Avoid loading entire files in memory
