# Python Async Programming

Complete guide to asynchronous programming in Python with asyncio, TaskGroup, and anyio.

## Fundamentals

### Basic Async/Await

```python
import asyncio

async def fetch_data(url: str) -> dict:
    """Async function (coroutine)."""
    await asyncio.sleep(1)  # Simulate I/O
    return {"url": url, "data": "..."}

async def main() -> None:
    result = await fetch_data("https://api.example.com")
    print(result)

# Run the event loop
asyncio.run(main())
```

### Coroutines vs Tasks

```python
import asyncio

async def work() -> str:
    await asyncio.sleep(1)
    return "done"

async def main() -> None:
    # Coroutine - not scheduled until awaited
    coro = work()
    result = await coro

    # Task - scheduled immediately
    task = asyncio.create_task(work())
    # Task is running in background
    result = await task
```

## Concurrent Execution

### asyncio.gather

```python
import asyncio

async def fetch(id: int) -> dict:
    await asyncio.sleep(0.5)
    return {"id": id}

async def main() -> None:
    # Run concurrently, wait for all
    results = await asyncio.gather(
        fetch(1),
        fetch(2),
        fetch(3),
    )
    print(results)  # [{"id": 1}, {"id": 2}, {"id": 3}]

    # With return_exceptions=True, exceptions don't propagate
    results = await asyncio.gather(
        fetch(1),
        failing_task(),
        return_exceptions=True,
    )
    # results[1] will be the exception
```

### TaskGroup (Python 3.11+) - Recommended

```python
import asyncio

async def fetch(id: int) -> dict:
    await asyncio.sleep(0.5)
    return {"id": id}

async def main() -> None:
    async with asyncio.TaskGroup() as tg:
        task1 = tg.create_task(fetch(1))
        task2 = tg.create_task(fetch(2))
        task3 = tg.create_task(fetch(3))

    # All tasks complete when exiting context
    print(task1.result())  # {"id": 1}
    print(task2.result())  # {"id": 2}
    print(task3.result())  # {"id": 3}
```

### TaskGroup Error Handling

```python
import asyncio

async def might_fail(n: int) -> int:
    if n == 2:
        raise ValueError(f"Failed for {n}")
    await asyncio.sleep(0.1)
    return n

async def main() -> None:
    try:
        async with asyncio.TaskGroup() as tg:
            tg.create_task(might_fail(1))
            tg.create_task(might_fail(2))  # Will raise
            tg.create_task(might_fail(3))
    except* ValueError as eg:
        # ExceptionGroup - handle all ValueErrors
        for exc in eg.exceptions:
            print(f"Caught: {exc}")
    except* TypeError as eg:
        # Handle all TypeErrors
        for exc in eg.exceptions:
            print(f"Type error: {exc}")

asyncio.run(main())
```

### Exception Groups (Python 3.11+)

```python
# except* catches exception groups
try:
    async with asyncio.TaskGroup() as tg:
        tg.create_task(task_that_raises_value_error())
        tg.create_task(task_that_raises_type_error())
except* ValueError as eg:
    print(f"ValueErrors: {eg.exceptions}")
except* TypeError as eg:
    print(f"TypeErrors: {eg.exceptions}")

# Both handlers run if both exception types occur
```

## Timeouts and Cancellation

### Timeout

```python
import asyncio

async def slow_operation() -> str:
    await asyncio.sleep(10)
    return "completed"

async def main() -> None:
    try:
        # Wait max 2 seconds
        result = await asyncio.wait_for(
            slow_operation(),
            timeout=2.0
        )
    except asyncio.TimeoutError:
        print("Operation timed out")

    # Python 3.11+ - timeout context manager
    async with asyncio.timeout(2.0):
        await slow_operation()
```

### Cancellation

```python
import asyncio

async def cancellable_task() -> str:
    try:
        await asyncio.sleep(10)
        return "done"
    except asyncio.CancelledError:
        print("Task was cancelled")
        # Cleanup here
        raise  # Re-raise to signal cancellation

async def main() -> None:
    task = asyncio.create_task(cancellable_task())

    await asyncio.sleep(1)
    task.cancel()

    try:
        await task
    except asyncio.CancelledError:
        print("Confirmed cancelled")
```

## Synchronization Primitives

### Semaphore (Rate Limiting)

```python
import asyncio

async def fetch(sem: asyncio.Semaphore, url: str) -> dict:
    async with sem:
        # Only N concurrent operations
        print(f"Fetching {url}")
        await asyncio.sleep(1)
        return {"url": url}

async def main() -> None:
    sem = asyncio.Semaphore(3)  # Max 3 concurrent

    urls = [f"https://api.example.com/{i}" for i in range(10)]
    tasks = [fetch(sem, url) for url in urls]
    results = await asyncio.gather(*tasks)
```

### Lock

```python
import asyncio

class Counter:
    def __init__(self) -> None:
        self.value = 0
        self._lock = asyncio.Lock()

    async def increment(self) -> None:
        async with self._lock:
            current = self.value
            await asyncio.sleep(0.01)  # Simulate work
            self.value = current + 1
```

### Event

```python
import asyncio

async def waiter(event: asyncio.Event, name: str) -> None:
    print(f"{name} waiting...")
    await event.wait()
    print(f"{name} proceeding!")

async def setter(event: asyncio.Event) -> None:
    await asyncio.sleep(1)
    print("Setting event")
    event.set()

async def main() -> None:
    event = asyncio.Event()

    async with asyncio.TaskGroup() as tg:
        tg.create_task(waiter(event, "A"))
        tg.create_task(waiter(event, "B"))
        tg.create_task(setter(event))
```

### Queue

```python
import asyncio

async def producer(queue: asyncio.Queue[int]) -> None:
    for i in range(5):
        await queue.put(i)
        print(f"Produced {i}")
    await queue.put(None)  # Sentinel

async def consumer(queue: asyncio.Queue[int]) -> None:
    while True:
        item = await queue.get()
        if item is None:
            break
        print(f"Consumed {item}")
        queue.task_done()

async def main() -> None:
    queue: asyncio.Queue[int] = asyncio.Queue(maxsize=2)

    async with asyncio.TaskGroup() as tg:
        tg.create_task(producer(queue))
        tg.create_task(consumer(queue))
```

## Async Context Managers

```python
from contextlib import asynccontextmanager
from typing import AsyncGenerator

@asynccontextmanager
async def database_connection() -> AsyncGenerator[Connection, None]:
    conn = await connect_to_db()
    try:
        yield conn
    finally:
        await conn.close()

async def main() -> None:
    async with database_connection() as conn:
        result = await conn.execute("SELECT * FROM users")
```

## Async Generators

```python
from typing import AsyncGenerator

async def async_range(count: int) -> AsyncGenerator[int, None]:
    for i in range(count):
        await asyncio.sleep(0.1)
        yield i

async def main() -> None:
    async for num in async_range(5):
        print(num)

    # Or collect all
    items = [item async for item in async_range(5)]
```

## AnyIO (Portable Async)

AnyIO provides a unified API that works with both asyncio and trio.

### Installation

```bash
uv add anyio
```

### Basic Usage

```python
import anyio

async def task(name: str) -> None:
    print(f"{name} starting")
    await anyio.sleep(1)
    print(f"{name} done")

async def main() -> None:
    async with anyio.create_task_group() as tg:
        tg.start_soon(task, "A")
        tg.start_soon(task, "B")

# Works with asyncio
anyio.run(main)

# Or with trio
anyio.run(main, backend="trio")
```

### AnyIO Semaphore

```python
import anyio

async def fetch(limiter: anyio.CapacityLimiter, url: str) -> dict:
    async with limiter:
        await anyio.sleep(1)
        return {"url": url}

async def main() -> None:
    limiter = anyio.CapacityLimiter(3)

    async with anyio.create_task_group() as tg:
        for i in range(10):
            tg.start_soon(fetch, limiter, f"https://api.example.com/{i}")

anyio.run(main)
```

### AnyIO File I/O

```python
import anyio
from anyio import Path

async def main() -> None:
    # Async file operations
    path = Path("data.txt")

    # Write
    await path.write_text("Hello, World!")

    # Read
    content = await path.read_text()

    # Check existence
    if await path.exists():
        await path.unlink()

anyio.run(main)
```

## HTTP with aiohttp

```python
import aiohttp
import asyncio

async def fetch(session: aiohttp.ClientSession, url: str) -> str:
    async with session.get(url) as response:
        return await response.text()

async def main() -> None:
    async with aiohttp.ClientSession() as session:
        html = await fetch(session, "https://example.com")
        print(len(html))

        # Concurrent requests
        urls = ["https://example.com"] * 5
        tasks = [fetch(session, url) for url in urls]
        results = await asyncio.gather(*tasks)

asyncio.run(main)
```

## HTTP with httpx (Async)

```python
import httpx
import asyncio

async def main() -> None:
    async with httpx.AsyncClient() as client:
        response = await client.get("https://api.example.com")
        data = response.json()

        # Concurrent requests
        urls = [f"https://api.example.com/{i}" for i in range(5)]
        tasks = [client.get(url) for url in urls]
        responses = await asyncio.gather(*tasks)

asyncio.run(main)
```

## Patterns

### Retry with Backoff

```python
import asyncio
from typing import TypeVar, Callable, Awaitable

T = TypeVar("T")

async def retry_with_backoff[T](
    fn: Callable[[], Awaitable[T]],
    max_retries: int = 3,
    base_delay: float = 1.0,
) -> T:
    for attempt in range(max_retries):
        try:
            return await fn()
        except Exception as e:
            if attempt == max_retries - 1:
                raise
            delay = base_delay * (2 ** attempt)
            await asyncio.sleep(delay)
    raise RuntimeError("Should not reach here")
```

### Producer-Consumer with Graceful Shutdown

```python
import asyncio
from typing import AsyncGenerator

async def producer(queue: asyncio.Queue[str | None]) -> None:
    for i in range(10):
        await queue.put(f"item-{i}")
        await asyncio.sleep(0.1)
    await queue.put(None)  # Signal end

async def consumer(queue: asyncio.Queue[str | None], name: str) -> None:
    while True:
        item = await queue.get()
        if item is None:
            await queue.put(None)  # Propagate to other consumers
            break
        print(f"{name}: {item}")
        queue.task_done()

async def main() -> None:
    queue: asyncio.Queue[str | None] = asyncio.Queue()

    async with asyncio.TaskGroup() as tg:
        tg.create_task(producer(queue))
        tg.create_task(consumer(queue, "C1"))
        tg.create_task(consumer(queue, "C2"))
```

### Async Iterator Protocol

```python
class AsyncCounter:
    def __init__(self, stop: int) -> None:
        self.current = 0
        self.stop = stop

    def __aiter__(self) -> "AsyncCounter":
        return self

    async def __anext__(self) -> int:
        if self.current >= self.stop:
            raise StopAsyncIteration
        await asyncio.sleep(0.1)
        value = self.current
        self.current += 1
        return value

async def main() -> None:
    async for num in AsyncCounter(5):
        print(num)
```

## Best Practices

1. **Use TaskGroup** over gather for better error handling (Python 3.11+)
2. **Always use timeouts** for external operations
3. **Use Semaphore** for rate limiting
4. **Handle CancelledError** properly in long-running tasks
5. **Use async context managers** for resource cleanup
6. **Prefer anyio** for library code that should work with multiple backends
7. **Don't mix sync and async** - use `asyncio.to_thread()` for sync code

## Common Pitfalls

| Pitfall | Solution |
|---------|----------|
| Blocking the event loop | Use `asyncio.to_thread()` for sync code |
| Not awaiting coroutines | Always await or create_task |
| Catching CancelledError without re-raising | Re-raise after cleanup |
| Using `time.sleep()` | Use `asyncio.sleep()` |
| Global mutable state | Use contextvars for task-local state |

## References

- [asyncio docs](https://docs.python.org/3/library/asyncio.html)
- [PEP 654 - Exception Groups](https://peps.python.org/pep-0654/)
- [anyio docs](https://anyio.readthedocs.io/)
