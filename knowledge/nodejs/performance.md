# Node.js Performance

> Source: https://nodejs.org/en/learn/getting-started/profiling

## CPU Profiling

### V8 Profiler (Built-in)

```bash
# Generate profile
node --prof app.js

# Process the log file
node --prof-process isolate-*.log > processed.txt
```

### Chrome DevTools

```bash
# Start with inspector
node --inspect app.js

# Open Chrome and navigate to
chrome://inspect
```

### Using Inspector API

```javascript
const inspector = require('inspector');
const fs = require('fs');

const session = new inspector.Session();
session.connect();

// Start profiling
session.post('Profiler.enable', () => {
  session.post('Profiler.start', () => {
    // Run your code here
    doWork();

    // Stop profiling
    session.post('Profiler.stop', (err, { profile }) => {
      fs.writeFileSync('profile.cpuprofile', JSON.stringify(profile));
    });
  });
});
```

## Memory Profiling

### Memory Usage

```javascript
const used = process.memoryUsage();

console.log({
  rss: `${Math.round(used.rss / 1024 / 1024)} MB`,      // Resident Set Size
  heapTotal: `${Math.round(used.heapTotal / 1024 / 1024)} MB`,
  heapUsed: `${Math.round(used.heapUsed / 1024 / 1024)} MB`,
  external: `${Math.round(used.external / 1024 / 1024)} MB`,
  arrayBuffers: `${Math.round(used.arrayBuffers / 1024 / 1024)} MB`
});
```

### V8 Heap Statistics

```javascript
const v8 = require('v8');

const heapStats = v8.getHeapStatistics();
console.log({
  totalHeapSize: `${Math.round(heapStats.total_heap_size / 1024 / 1024)} MB`,
  usedHeapSize: `${Math.round(heapStats.used_heap_size / 1024 / 1024)} MB`,
  heapSizeLimit: `${Math.round(heapStats.heap_size_limit / 1024 / 1024)} MB`
});
```

### Heap Snapshots

```javascript
const v8 = require('v8');
const fs = require('fs');

// Take heap snapshot
const snapshotFile = `heap-${Date.now()}.heapsnapshot`;
const stream = fs.createWriteStream(snapshotFile);
v8.writeHeapSnapshot(snapshotFile);

// Or via signal
process.on('SIGUSR2', () => {
  v8.writeHeapSnapshot();
});
```

## Common Performance Issues

### Memory Leaks

```javascript
// LEAK: Unbounded cache
const cache = {};
function setCache(key, value) {
  cache[key] = value; // Never cleaned up!
}

// FIX: Use LRU cache
const LRU = require('lru-cache');
const cache = new LRU({ max: 1000 });

// LEAK: Event listeners not removed
class MyClass extends EventEmitter {
  start() {
    process.on('data', this.handleData); // Added every call!
  }
}

// FIX: Remove listeners
class MyClass extends EventEmitter {
  constructor() {
    this.handleData = this.handleData.bind(this);
  }
  start() {
    process.on('data', this.handleData);
  }
  stop() {
    process.off('data', this.handleData);
  }
}

// LEAK: Closures holding references
function createHandler(largeData) {
  return () => {
    // largeData captured forever
    console.log('handler');
  };
}

// FIX: Extract only what's needed
function createHandler(dataId) {
  return () => {
    console.log(`handler for ${dataId}`);
  };
}
```

### Event Loop Blocking

```javascript
// BAD: Synchronous operations
const data = fs.readFileSync('large-file.txt');
const hash = crypto.pbkdf2Sync(password, salt, 100000, 64, 'sha512');

// GOOD: Async operations
const data = await fs.promises.readFile('large-file.txt');
const hash = await promisify(crypto.pbkdf2)(password, salt, 100000, 64, 'sha512');

// GOOD: Offload CPU work to worker threads
const { Worker } = require('worker_threads');
```

### N+1 Queries

```javascript
// BAD: N+1 queries
const users = await User.findAll();
for (const user of users) {
  user.posts = await Post.findAll({ where: { userId: user.id } });
}

// GOOD: Eager loading
const users = await User.findAll({
  include: [{ model: Post }]
});
```

## Optimization Techniques

### Connection Pooling

```javascript
const { Pool } = require('pg');

const pool = new Pool({
  max: 20,
  idleTimeoutMillis: 30000,
  connectionTimeoutMillis: 2000
});

// Use pool for queries
const result = await pool.query('SELECT * FROM users');
```

### Caching

```javascript
const Redis = require('ioredis');
const redis = new Redis();

async function getUser(id) {
  // Check cache first
  const cached = await redis.get(`user:${id}`);
  if (cached) {
    return JSON.parse(cached);
  }

  // Fetch from database
  const user = await db.query('SELECT * FROM users WHERE id = $1', [id]);

  // Cache result
  await redis.set(`user:${id}`, JSON.stringify(user), 'EX', 3600);

  return user;
}
```

### Streaming Large Data

```javascript
// BAD: Load entire file
const data = await fs.promises.readFile('large-file.json');
const parsed = JSON.parse(data);

// GOOD: Stream and parse incrementally
const JSONStream = require('JSONStream');

const stream = fs.createReadStream('large-file.json')
  .pipe(JSONStream.parse('*'));

for await (const item of stream) {
  await processItem(item);
}
```

## Monitoring

### Event Loop Lag

```javascript
const start = process.hrtime.bigint();

setImmediate(() => {
  const lag = Number(process.hrtime.bigint() - start) / 1e6;
  if (lag > 100) {
    console.warn(`Event loop lag: ${lag}ms`);
  }
});
```

### Metrics Export

```javascript
const prometheus = require('prom-client');

// Enable default metrics
prometheus.collectDefaultMetrics();

// Custom metrics
const httpRequestDuration = new prometheus.Histogram({
  name: 'http_request_duration_seconds',
  help: 'Duration of HTTP requests in seconds',
  labelNames: ['method', 'route', 'status'],
  buckets: [0.1, 0.5, 1, 2, 5]
});

// Metrics endpoint
app.get('/metrics', async (req, res) => {
  res.set('Content-Type', prometheus.register.contentType);
  res.end(await prometheus.register.metrics());
});
```

## Node.js Flags

```bash
# Increase heap size
node --max-old-space-size=4096 app.js

# Optimize for memory
node --optimize-for-size app.js

# Enable GC logging
node --trace-gc app.js

# Expose GC for manual triggering
node --expose-gc app.js

# Performance timeline
node --perf-basic-prof app.js
```

## Performance Targets

| Metric | Target |
|--------|--------|
| Event loop lag | < 100ms |
| Response time P99 | < 500ms |
| Heap usage | < 70% of limit |
| GC pause | < 100ms |
| CPU usage | < 70% per core |
