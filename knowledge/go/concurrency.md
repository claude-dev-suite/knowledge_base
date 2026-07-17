# Go Concurrency

> Official Documentation: https://go.dev/doc/effective_go#concurrency

## Overview

Concurrency is a first-class part of Go. Lightweight **goroutines** run
functions concurrently, and **channels** let them communicate safely. The Go
proverb captures the philosophy: *"Do not communicate by sharing memory;
instead, share memory by communicating."* Examples target **Go 1.26**.

## Goroutines

A goroutine is a function executing concurrently, multiplexed onto OS threads by
the runtime. Starting one costs a few kilobytes of stack.

```go
go doWork()            // start a goroutine
go func() {            // anonymous goroutine
	fmt.Println("async")
}()
```

Goroutines run independently; `main` returning terminates them all. Use a
`sync.WaitGroup` or channels to wait for completion — never `time.Sleep`.

## Channels

Channels are typed conduits. Send with `ch <- v`, receive with `v := <-ch`.

```go
ch := make(chan int)      // unbuffered: send blocks until a receiver is ready
buf := make(chan int, 3)  // buffered: holds up to 3 values

ch <- 42                  // send
v := <-ch                 // receive
v, ok := <-ch             // ok is false once the channel is closed and drained

close(ch)                 // only the sender should close
for v := range ch {       // ranges until the channel is closed
	fmt.Println(v)
}
```

Guidelines:

- Sending on a closed channel panics; receiving returns the zero value + `false`.
- Only close a channel from the sending side, and only once.
- A `nil` channel blocks forever (useful to disable a `select` case).

### WaitGroup with channels

```go
func fanOut(jobs []int) []int {
	results := make(chan int, len(jobs))
	var wg sync.WaitGroup
	for _, j := range jobs {
		wg.Add(1)
		go func(n int) {
			defer wg.Done()
			results <- n * n
		}(j)
	}
	wg.Wait()
	close(results)

	var out []int
	for r := range results {
		out = append(out, r)
	}
	return out
}
```

Note: from Go 1.22 onward each loop iteration has its own `j`, so capturing the
loop variable directly is safe; passing it as an argument still works and is
explicit.

## select

`select` waits on multiple channel operations, choosing one that is ready (at
random if several are). It is the core primitive for multiplexing.

```go
select {
case v := <-in:
	fmt.Println("received", v)
case out <- next:
	fmt.Println("sent")
case <-time.After(time.Second):
	fmt.Println("timeout")
default:
	fmt.Println("nothing ready") // non-blocking
}
```

## The sync package

Channels are not always the right tool. For protecting shared state, `sync`
provides classic primitives.

| Type / func | Purpose |
| --- | --- |
| `sync.Mutex` | Mutual exclusion lock |
| `sync.RWMutex` | Multiple readers or one writer |
| `sync.WaitGroup` | Wait for a set of goroutines to finish |
| `sync.Once` | Run initialization exactly once |
| `sync.Map` | Concurrent map for specific access patterns |
| `sync.Pool` | Reuse temporary objects to cut allocations |
| `sync/atomic` | Lock-free atomic operations |

```go
type Counter struct {
	mu sync.Mutex
	n  int
}

func (c *Counter) Inc() {
	c.mu.Lock()
	defer c.mu.Unlock()
	c.n++
}

var once sync.Once
once.Do(func() { fmt.Println("init") }) // runs once even across goroutines
```

Typed atomics avoid manual size handling:

```go
var hits atomic.Int64
hits.Add(1)
fmt.Println(hits.Load())
```

Always run tests with the **race detector** to catch data races:

```bash
go test -race ./...
go run -race .
```

## Context

The `context` package carries deadlines, cancellation signals, and
request-scoped values across API boundaries and goroutines.

```go
func fetch(ctx context.Context, url string) error {
	ctx, cancel := context.WithTimeout(ctx, 2*time.Second)
	defer cancel() // always call cancel to release resources

	req, _ := http.NewRequestWithContext(ctx, http.MethodGet, url, nil)
	resp, err := http.DefaultClient.Do(req)
	if err != nil {
		return err
	}
	defer resp.Body.Close()
	return nil
}
```

Common constructors:

| Function | Use |
| --- | --- |
| `context.Background()` | Root context (main, init, tests) |
| `context.TODO()` | Placeholder when unsure which context to use |
| `context.WithCancel(parent)` | Manual cancellation |
| `context.WithTimeout(parent, d)` | Cancel after a duration |
| `context.WithDeadline(parent, t)` | Cancel at a specific time |
| `context.WithValue(parent, k, v)` | Attach a request-scoped value |

Select on `ctx.Done()` to react to cancellation:

```go
select {
case <-ctx.Done():
	return ctx.Err() // context.Canceled or context.DeadlineExceeded
case res := <-work:
	return handle(res)
}
```

Pass `ctx` as the first parameter of a function; never store it in a struct.

## Worker pools

A worker pool bounds concurrency by fanning a fixed number of goroutines out
over a jobs channel.

```go
func workerPool(ctx context.Context, jobs <-chan int, workers int) <-chan int {
	results := make(chan int)
	var wg sync.WaitGroup

	for i := 0; i < workers; i++ {
		wg.Add(1)
		go func() {
			defer wg.Done()
			for j := range jobs {
				select {
				case <-ctx.Done():
					return
				case results <- j * 2:
				}
			}
		}()
	}

	go func() {
		wg.Wait()
		close(results)
	}()
	return results
}
```

For coordinating a group of goroutines that can fail, prefer
`golang.org/x/sync/errgroup`, which combines a `WaitGroup` with error
propagation and context cancellation.

## Pitfalls

- **Goroutine leaks**: a goroutine blocked on a channel that never receives
  never exits. Ensure every goroutine has a guaranteed way to finish (closed
  channel or context cancellation).
- **Deadlocks**: the runtime panics if *all* goroutines are blocked.
- **Unsynchronized access**: shared variables need a mutex or channel; rely on
  `-race` to find violations.
