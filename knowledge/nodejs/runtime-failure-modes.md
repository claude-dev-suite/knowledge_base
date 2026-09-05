# Node.js Failure Modes With No Diagnostic

> No upstream. Each mechanism is documented on its own page; the set of
> failures that produce *no error at all* is not collected anywhere, because it
> is defined by absence rather than by a feature.

## Why this page exists

Node reports a great deal loudly. An unhandled rejection terminates the process
from Node 15; a thrown `error` event with no listener is an uncaught exception;
a missing module fails at load.

The expensive mistakes are the ones that produce **nothing**: no exception, no
warning, no failed test. They pass review, pass CI, and appear as latency,
memory growth, or a silently truncated result in production.

The organising fact is that your code gets **one process and one thread**.
Almost everything below is a way of occupying that thread, or of losing track
of work scheduled on it.

## The four with no signal whatsoever

### Blocking the event loop

`fs.readFileSync`, `crypto.pbkdf2Sync`, `execSync`, or a CPU-bound loop stops
*every* concurrent request, not just the one that called it. The latency lands
on unrelated endpoints — a health check times out while a report is generated —
which is why the symptom rarely points at the cause.

Nothing reports it. There is no warning for "this took 200 ms of CPU".

**The distinction that matters**: on the **startup** path, `*Sync` is often the
right choice and deserves no comment. On the **request** path it is a defect.
Establish which before commenting; it removes most false positives.

### A stream written faster than it drains

```js
source.on('data', chunk => dest.write(chunk));   // ignores the return value
```

`write()` returning `false` means "the buffer is full, stop". Ignoring it grows
the buffer in memory until the process dies — with no error until the
allocation fails.

`pipeline()` handles backpressure *and* propagates errors *and* destroys the
streams on failure, which a bare `pipe()` does not.

### A listener added per request

```js
app.get('/x', (req, res) => {
  emitter.on('update', handler);   // never removed
});
```

Listeners accumulate, each holding its closure and everything captured. The
only signal is `MaxListenersExceededWarning` at eleven — a warning about the
leak's *size*, not its existence, and one that is routinely raised in code
where eleven listeners are legitimate.

### A callback called twice, or never

Neither is detectable by the runtime. A callback invoked twice runs the
continuation twice — double-charging, double-inserting. One never invoked
leaves a request hanging until a timeout somewhere else fires, with the handle
still open.

Promises make this class structurally impossible, which is the strongest
argument for converting a callback API rather than reviewing it carefully.

## The ones that only fail outside development

These are worse than the four above, because local testing actively confirms
they work.

### `process.exit()` truncating output

```js
console.log(report);
process.exit(0);
```

Whether this loses the output depends on what stdout **is**:

| stdout target | Write behaviour | Result |
|---|---|---|
| TTY (a terminal) | synchronous | output appears — every manual test passes |
| pipe (`\| less`, `> file`, CI) | **asynchronous** | `exit` truncates it |

So a CLI loses its output exactly when it is used inside a script, which is the
case that matters and the one nobody tests interactively.

Set `process.exitCode` and return, letting the loop drain, rather than calling
`exit`.

### Module-scope state under multiple processes

```js
let cache = {};
let requestCounts = {};        // a rate limiter
```

Module state is **per process**. Under `cluster`, PM2, or any horizontally
scaled deployment there are N copies, so:

- a rate limiter allows N times its configured limit;
- a lock locks nothing;
- a cache has an N-times-lower hit rate and can serve N different answers;
- an in-memory session store logs users out at random as they hit other workers.

Every one of these works perfectly on one developer machine, and the failure in
production looks like a data problem rather than an architectural one.

### An open handle keeping the process alive

```js
const fh = await fs.promises.open(p);
const data = await parse(fh);      // throws: never closed
await fh.close();
```

The symptom is usually **a process that will not exit** rather than an error —
a CI job that hangs after tests pass, or a container that ignores SIGTERM. An
un-`unref`'d `setInterval` does the same thing.

`try/finally`, or `using` where the target supports explicit resource
management, is the fix.

## What to check, in order

1. **Is this the startup path or the request path?** Settles every synchronous
   call in the diff.
2. **How many processes run this?** Settles every piece of module state.
3. **Is stdout a pipe here?** Settles every `process.exit`.
4. **Does every path release what it acquired?** Handles, timers, listeners,
   locks.
5. **Can this fan-out be unbounded?** `Promise.all` over a data-sized array
   opens every connection at once; the limit should be a decision, not an
   accident.

## Version boundaries worth knowing

| Behaviour | From |
|---|---|
| Unhandled rejection terminates the process | Node 15 |
| Global `fetch` | Node 18 |
| Stable test runner, `--watch` | Node 20 |
| `require()` of ESM | Node 22 (behind a flag earlier) |

Read `engines` in `package.json` and the Node version CI actually uses — they
disagree more often than not.
