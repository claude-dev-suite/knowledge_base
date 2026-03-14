# Node.js Cluster

> Source: https://nodejs.org/api/cluster.html

## Overview

The cluster module allows you to create child processes (workers) that share server ports. This enables you to take advantage of multi-core systems by running multiple instances of Node.js.

## Basic Usage

```javascript
const cluster = require('cluster');
const http = require('http');
const numCPUs = require('os').cpus().length;

if (cluster.isPrimary) {
  console.log(`Primary ${process.pid} is running`);

  // Fork workers
  for (let i = 0; i < numCPUs; i++) {
    cluster.fork();
  }

  cluster.on('exit', (worker, code, signal) => {
    console.log(`Worker ${worker.process.pid} died`);
  });
} else {
  // Workers share the TCP connection
  http.createServer((req, res) => {
    res.writeHead(200);
    res.end('Hello World\n');
  }).listen(8000);

  console.log(`Worker ${process.pid} started`);
}
```

## Auto-Restart Workers

```javascript
const cluster = require('cluster');
const http = require('http');
const numCPUs = require('os').cpus().length;

if (cluster.isPrimary) {
  console.log(`Primary ${process.pid} is running`);

  for (let i = 0; i < numCPUs; i++) {
    cluster.fork();
  }

  // Restart dead workers
  cluster.on('exit', (worker, code, signal) => {
    console.log(`Worker ${worker.process.pid} died (${signal || code}). Restarting...`);
    cluster.fork();
  });

} else {
  http.createServer((req, res) => {
    res.writeHead(200);
    res.end(`Worker ${process.pid}\n`);
  }).listen(8000);
}
```

## Communication Between Primary and Workers

### Primary to Worker

```javascript
if (cluster.isPrimary) {
  const worker = cluster.fork();

  worker.send({ type: 'config', data: { port: 8000 } });
} else {
  process.on('message', (msg) => {
    console.log('Worker received:', msg);
  });
}
```

### Worker to Primary

```javascript
if (cluster.isPrimary) {
  cluster.on('message', (worker, message) => {
    console.log(`Message from worker ${worker.id}:`, message);
  });
} else {
  process.send({ type: 'ready', workerId: cluster.worker.id });
}
```

### Broadcast to All Workers

```javascript
if (cluster.isPrimary) {
  function broadcast(message) {
    for (const id in cluster.workers) {
      cluster.workers[id].send(message);
    }
  }

  // Broadcast config update
  broadcast({ type: 'config-update', data: newConfig });
}
```

## Graceful Shutdown

```javascript
const cluster = require('cluster');
const http = require('http');
const numCPUs = require('os').cpus().length;

if (cluster.isPrimary) {
  for (let i = 0; i < numCPUs; i++) {
    cluster.fork();
  }

  // Handle shutdown signal
  process.on('SIGTERM', () => {
    console.log('Primary received SIGTERM, shutting down...');

    for (const id in cluster.workers) {
      cluster.workers[id].send('shutdown');
    }

    // Force exit after timeout
    setTimeout(() => {
      console.log('Force exit');
      process.exit(0);
    }, 10000);
  });

  cluster.on('exit', (worker) => {
    console.log(`Worker ${worker.process.pid} exited`);

    // Check if all workers exited
    if (Object.keys(cluster.workers).length === 0) {
      console.log('All workers exited, shutting down primary');
      process.exit(0);
    }
  });

} else {
  const server = http.createServer((req, res) => {
    res.writeHead(200);
    res.end('Hello\n');
  }).listen(8000);

  process.on('message', (msg) => {
    if (msg === 'shutdown') {
      console.log(`Worker ${process.pid} shutting down...`);

      // Stop accepting new connections
      server.close(() => {
        console.log(`Worker ${process.pid} closed`);
        process.exit(0);
      });
    }
  });
}
```

## Zero-Downtime Restart

```javascript
const cluster = require('cluster');
const numCPUs = require('os').cpus().length;

if (cluster.isPrimary) {
  for (let i = 0; i < numCPUs; i++) {
    cluster.fork();
  }

  // Rolling restart
  process.on('SIGUSR2', () => {
    const workers = Object.values(cluster.workers);

    function restartWorker(index) {
      if (index >= workers.length) return;

      const worker = workers[index];
      console.log(`Restarting worker ${worker.process.pid}`);

      // Fork new worker
      const newWorker = cluster.fork();

      newWorker.on('listening', () => {
        // New worker is ready, kill old one
        worker.disconnect();

        worker.on('disconnect', () => {
          console.log(`Worker ${worker.process.pid} disconnected`);
          // Restart next worker
          restartWorker(index + 1);
        });
      });
    }

    restartWorker(0);
  });
}
```

## Scheduling Policy

```javascript
// Round-robin (default on all platforms except Windows)
cluster.schedulingPolicy = cluster.SCHED_RR;

// OS-based scheduling
cluster.schedulingPolicy = cluster.SCHED_NONE;
```

## Cluster Events

| Event | Description |
|-------|-------------|
| `fork` | New worker is forked |
| `online` | Worker is running |
| `listening` | Worker is listening |
| `disconnect` | Worker IPC channel disconnected |
| `exit` | Worker process exited |
| `message` | Message received from worker |

## PM2 Alternative

For production, consider using PM2 which handles clustering, logging, and monitoring.

```bash
# Install PM2
npm install -g pm2

# Start with cluster mode
pm2 start app.js -i max

# Or specific number of instances
pm2 start app.js -i 4
```

## Best Practices

1. **Match CPU cores** - Fork one worker per CPU core
2. **Auto-restart** - Restart crashed workers
3. **Graceful shutdown** - Close connections before exiting
4. **Health checks** - Monitor worker health
5. **Use PM2 in production** - Better process management
6. **Sticky sessions** - Required for WebSockets (use nginx or PM2)
