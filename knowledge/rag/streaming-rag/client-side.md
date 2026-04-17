# Client-Side Streaming RAG -- Progressive Disclosure, Citation Rendering, and TTFT Optimization

## TL;DR

The client side of streaming RAG must handle a sequence of asynchronous events (retrieval status, source metadata, token stream, citations, errors) and render them progressively so the user sees useful information as early as possible. This article covers client-side patterns: progressive disclosure UX (showing retrieval status, then sources, then streaming text), inline citation rendering (matching citation markers to source documents as tokens arrive), TTFT optimization on the client (prefetching, skeleton states, optimistic updates), and production-quality EventSource handling (reconnection, error recovery, and cleanup).

---

## EventSource Client Setup

### Basic SSE Client (Browser JavaScript)

```python
# This article includes both Python (for testing/automation)
# and JavaScript (for browser clients) examples.

# Python SSE client for testing streaming endpoints
import json
import httpx


class SSEClient:
    """Python client for consuming SSE streams from RAG endpoints.

    Useful for testing, monitoring, and backend-to-backend streaming.
    """

    def __init__(self, base_url: str):
        self.base_url = base_url

    def stream_query(self, query: str) -> list[dict]:
        """Consume an SSE stream and return all events."""
        events = []

        with httpx.stream(
            "GET",
            f"{self.base_url}/api/stream",
            params={"query": query},
            timeout=30.0,
        ) as response:
            buffer = ""
            for chunk in response.iter_text():
                buffer += chunk
                while "\n\n" in buffer:
                    event_text, buffer = buffer.split("\n\n", 1)
                    event = self._parse_sse_event(event_text)
                    if event:
                        events.append(event)

        return events

    def stream_query_live(self, query: str):
        """Stream events as they arrive (generator)."""
        with httpx.stream(
            "GET",
            f"{self.base_url}/api/stream",
            params={"query": query},
            timeout=30.0,
        ) as response:
            buffer = ""
            for chunk in response.iter_text():
                buffer += chunk
                while "\n\n" in buffer:
                    event_text, buffer = buffer.split("\n\n", 1)
                    event = self._parse_sse_event(event_text)
                    if event:
                        yield event

    @staticmethod
    def _parse_sse_event(text: str) -> dict | None:
        """Parse an SSE event into a dict."""
        event_type = None
        data = None

        for line in text.strip().split("\n"):
            if line.startswith("event: "):
                event_type = line[7:]
            elif line.startswith("data: "):
                data = line[6:]
            elif line.startswith(":"):
                # Comment (keepalive), ignore
                continue

        if data:
            try:
                parsed = json.loads(data)
                parsed["_event_type"] = event_type
                return parsed
            except json.JSONDecodeError:
                return {"_raw": data, "_event_type": event_type}
        return None
```

### Async Python Client

```python
import asyncio
import json
import httpx


class AsyncSSEClient:
    """Async Python SSE client for streaming RAG responses."""

    def __init__(self, base_url: str):
        self.base_url = base_url

    async def stream_query(self, query: str):
        """Async generator yielding parsed SSE events."""
        async with httpx.AsyncClient() as client:
            async with client.stream(
                "GET",
                f"{self.base_url}/api/stream",
                params={"query": query},
                timeout=30.0,
            ) as response:
                buffer = ""
                async for chunk in response.aiter_text():
                    buffer += chunk
                    while "\n\n" in buffer:
                        event_text, buffer = buffer.split("\n\n", 1)
                        event = self._parse_event(event_text)
                        if event:
                            yield event

    @staticmethod
    def _parse_event(text: str) -> dict | None:
        data = None
        for line in text.strip().split("\n"):
            if line.startswith("data: "):
                data = line[6:]
        if data:
            try:
                return json.loads(data)
            except json.JSONDecodeError:
                return None
        return None
```

---

## Progressive Disclosure Pattern

### State Machine for RAG Streaming UI

```python
from enum import Enum
from dataclasses import dataclass, field


class StreamState(Enum):
    IDLE = "idle"
    RETRIEVING = "retrieving"
    SOURCES_LOADED = "sources_loaded"
    GENERATING = "generating"
    COMPLETE = "complete"
    ERROR = "error"


@dataclass
class StreamUIState:
    """State model for the streaming RAG UI.

    The client transitions through states as events arrive:
    IDLE -> RETRIEVING -> SOURCES_LOADED -> GENERATING -> COMPLETE
    Any state can transition to ERROR.
    """

    state: StreamState = StreamState.IDLE
    query: str = ""
    sources: list[dict] = field(default_factory=list)
    text_buffer: str = ""
    citations: list[dict] = field(default_factory=list)
    error_message: str = ""
    ttft_ms: float = 0.0
    total_ms: float = 0.0

    def process_event(self, event: dict) -> "StreamUIState":
        """Process an SSE event and return updated state."""
        event_type = event.get("type", "")

        if event_type == "stream_start":
            self.state = StreamState.RETRIEVING
            self.query = event.get("query", "")

        elif event_type == "retrieval_start":
            self.state = StreamState.RETRIEVING

        elif event_type == "sources":
            self.sources = event.get("sources", [])
            self.state = StreamState.SOURCES_LOADED

        elif event_type == "retrieval_done":
            self.sources = event.get("sources", [])
            self.state = StreamState.SOURCES_LOADED

        elif event_type == "token":
            if self.state != StreamState.GENERATING:
                self.state = StreamState.GENERATING
                self.ttft_ms = event.get("elapsed_ms", 0)
            self.text_buffer += event.get("text", "")

        elif event_type == "citation":
            self.citations.append({
                "id": event.get("id"),
                "source": event.get("source", {}),
            })

        elif event_type == "done":
            self.state = StreamState.COMPLETE
            self.total_ms = event.get("total_time_ms", 0)

        elif event_type == "error":
            self.state = StreamState.ERROR
            self.error_message = event.get("message", "Unknown error")

        return self

    def get_render_data(self) -> dict:
        """Get data needed to render the current UI state."""
        return {
            "state": self.state.value,
            "show_spinner": self.state == StreamState.RETRIEVING,
            "show_sources": self.state in (
                StreamState.SOURCES_LOADED,
                StreamState.GENERATING,
                StreamState.COMPLETE,
            ),
            "show_text": self.state in (
                StreamState.GENERATING,
                StreamState.COMPLETE,
            ),
            "show_cursor": self.state == StreamState.GENERATING,
            "show_error": self.state == StreamState.ERROR,
            "sources": self.sources,
            "text": self.text_buffer,
            "citations": self.citations,
            "error": self.error_message,
        }
```

### Python Terminal Renderer

```python
import sys


class TerminalStreamRenderer:
    """Render streaming RAG in a terminal.

    Shows progressive disclosure:
    1. "Searching..." during retrieval
    2. Source list when retrieved
    3. Streaming text with cursor
    4. Final summary
    """

    def __init__(self):
        self.state = StreamUIState()

    def render_event(self, event: dict) -> None:
        """Process and render a single event."""
        self.state.process_event(event)
        render = self.state.get_render_data()

        if render["show_spinner"] and self.state.state == StreamState.RETRIEVING:
            sys.stdout.write("\rSearching...")
            sys.stdout.flush()

        elif event.get("type") == "sources":
            sys.stdout.write("\r" + " " * 20 + "\r")  # Clear line
            print(f"\nSources ({len(render['sources'])}):")
            for s in render["sources"]:
                print(f"  [{s.get('index', '?')}] {s.get('title', s.get('id', '?'))}")
            print("\nAnswer:")

        elif event.get("type") == "token":
            sys.stdout.write(event.get("text", ""))
            sys.stdout.flush()

        elif event.get("type") == "done":
            print(f"\n\n--- Done ({self.state.total_ms:.0f}ms) ---")

        elif event.get("type") == "error":
            print(f"\nError: {render['error']}")


# Usage
async def main():
    renderer = TerminalStreamRenderer()
    client = AsyncSSEClient("http://localhost:8000")

    async for event in client.stream_query("What are the rate limits?"):
        renderer.render_event(event)
```

---

## Citation Rendering

### Client-Side Citation Parser

```python
import re
from dataclasses import dataclass


@dataclass
class TextSegment:
    """A segment of text with optional citation."""

    text: str
    citation_id: str | None = None


class ClientCitationParser:
    """Parse streaming text into segments with inline citations.

    As text streams in, the parser identifies citation markers
    [1], [2] etc. and splits the text into segments for rendering.
    """

    CITATION_RE = re.compile(r"\[(\d+)\]")

    def __init__(self, sources: list[dict]):
        self.sources = {
            str(s.get("index", i + 1)): s
            for i, s in enumerate(sources)
        }
        self.segments: list[TextSegment] = []

    def parse_text(self, full_text: str) -> list[TextSegment]:
        """Parse the full accumulated text into segments."""
        segments = []
        last_end = 0

        for match in self.CITATION_RE.finditer(full_text):
            # Text before citation
            if match.start() > last_end:
                text_before = full_text[last_end : match.start()]
                if text_before.strip():
                    segments.append(TextSegment(text=text_before))

            # Citation
            cite_id = match.group(1)
            segments.append(
                TextSegment(text=f"[{cite_id}]", citation_id=cite_id)
            )
            last_end = match.end()

        # Remaining text after last citation
        if last_end < len(full_text):
            remaining = full_text[last_end:]
            if remaining.strip():
                segments.append(TextSegment(text=remaining))

        self.segments = segments
        return segments

    def get_cited_sources(self) -> list[dict]:
        """Get the list of sources that were actually cited."""
        cited_ids = {
            s.citation_id
            for s in self.segments
            if s.citation_id
        }
        return [
            self.sources[cid]
            for cid in sorted(cited_ids)
            if cid in self.sources
        ]
```

---

## TTFT Optimization on the Client

### Optimistic UI Updates

```python
class OptimisticRAGClient:
    """Client that shows optimistic UI states to reduce perceived latency.

    Techniques:
    1. Show cached answer preview while retrieving
    2. Display probable sources immediately
    3. Start rendering as soon as the first token arrives
    """

    def __init__(self, cache: dict | None = None):
        self.cache = cache or {}
        self.prefetch_sources: dict = {}

    def get_optimistic_state(self, query: str) -> dict:
        """Get an immediate optimistic state for the query."""
        # Check if we have a cached response
        cache_key = query.lower().strip()
        if cache_key in self.cache:
            return {
                "show_cached_preview": True,
                "cached_answer": self.cache[cache_key]["answer"][:200] + "...",
                "cached_sources": self.cache[cache_key].get("sources", []),
                "message": "Showing cached response, updating...",
            }

        return {
            "show_cached_preview": False,
            "show_skeleton": True,
            "message": "Searching documentation...",
        }

    def update_cache(
        self, query: str, answer: str, sources: list
    ) -> None:
        """Update the client cache after a complete response."""
        self.cache[query.lower().strip()] = {
            "answer": answer,
            "sources": sources,
        }
```

### Connection Management

```python
class ManagedSSEConnection:
    """Managed SSE connection with auto-reconnect and cleanup.

    Handles:
    - Automatic reconnection on connection drop
    - Clean cancellation when user navigates away
    - Timeout on stale connections
    """

    def __init__(
        self,
        base_url: str,
        max_retries: int = 3,
        timeout: float = 30.0,
    ):
        self.base_url = base_url
        self.max_retries = max_retries
        self.timeout = timeout
        self._cancel = False

    async def stream(
        self, query: str
    ) -> list[dict]:
        """Stream with retry logic."""
        all_events = []

        for attempt in range(self.max_retries + 1):
            if self._cancel:
                break

            try:
                async for event in self._connect(query):
                    if self._cancel:
                        break
                    all_events.append(event)
                    if event.get("type") == "done":
                        return all_events

                # Stream ended without "done" event
                break

            except (httpx.ReadTimeout, httpx.ConnectError) as e:
                if attempt < self.max_retries:
                    wait = (attempt + 1) * 2
                    await asyncio.sleep(wait)
                else:
                    raise

        return all_events

    async def _connect(self, query: str):
        """Single connection attempt."""
        async with httpx.AsyncClient() as client:
            async with client.stream(
                "GET",
                f"{self.base_url}/api/stream",
                params={"query": query},
                timeout=self.timeout,
            ) as response:
                buffer = ""
                async for chunk in response.aiter_text():
                    buffer += chunk
                    while "\n\n" in buffer:
                        raw, buffer = buffer.split("\n\n", 1)
                        event = self._parse(raw)
                        if event:
                            yield event

    def cancel(self) -> None:
        """Cancel the active stream."""
        self._cancel = True

    @staticmethod
    def _parse(text: str) -> dict | None:
        for line in text.strip().split("\n"):
            if line.startswith("data: "):
                try:
                    return json.loads(line[6:])
                except json.JSONDecodeError:
                    return None
        return None
```

---

## Performance Measurement

### Measuring TTFT and Streaming Metrics

```python
import time
from dataclasses import dataclass


@dataclass
class StreamingMetrics:
    """Metrics for a streaming RAG response."""

    query_sent_at: float = 0.0
    first_source_at: float = 0.0
    first_token_at: float = 0.0
    last_token_at: float = 0.0
    stream_done_at: float = 0.0
    total_tokens: int = 0
    total_chars: int = 0

    @property
    def ttft_ms(self) -> float:
        """Time to first token."""
        if self.first_token_at and self.query_sent_at:
            return (self.first_token_at - self.query_sent_at) * 1000
        return 0.0

    @property
    def retrieval_ms(self) -> float:
        if self.first_source_at and self.query_sent_at:
            return (self.first_source_at - self.query_sent_at) * 1000
        return 0.0

    @property
    def generation_ms(self) -> float:
        if self.last_token_at and self.first_token_at:
            return (self.last_token_at - self.first_token_at) * 1000
        return 0.0

    @property
    def total_ms(self) -> float:
        if self.stream_done_at and self.query_sent_at:
            return (self.stream_done_at - self.query_sent_at) * 1000
        return 0.0

    @property
    def tokens_per_second(self) -> float:
        gen_sec = self.generation_ms / 1000
        if gen_sec > 0:
            return self.total_tokens / gen_sec
        return 0.0


class StreamingMetricsCollector:
    """Collect metrics from a streaming session."""

    def __init__(self):
        self.metrics = StreamingMetrics()
        self.metrics.query_sent_at = time.perf_counter()

    def on_event(self, event: dict) -> None:
        event_type = event.get("type", "")

        if event_type in ("sources", "retrieval_done"):
            if not self.metrics.first_source_at:
                self.metrics.first_source_at = time.perf_counter()

        elif event_type == "token":
            now = time.perf_counter()
            if not self.metrics.first_token_at:
                self.metrics.first_token_at = now
            self.metrics.last_token_at = now
            text = event.get("text", "")
            self.metrics.total_tokens += 1
            self.metrics.total_chars += len(text)

        elif event_type == "done":
            self.metrics.stream_done_at = time.perf_counter()

    def report(self) -> dict:
        m = self.metrics
        return {
            "ttft_ms": round(m.ttft_ms, 1),
            "retrieval_ms": round(m.retrieval_ms, 1),
            "generation_ms": round(m.generation_ms, 1),
            "total_ms": round(m.total_ms, 1),
            "tokens_per_second": round(m.tokens_per_second, 1),
            "total_chars": m.total_chars,
        }
```

---

## Common Pitfalls

1. **Not showing any UI during retrieval.** The 200-500ms retrieval phase is dead time if the client shows nothing. Display a "Searching..." indicator immediately when the query is sent.
2. **Rendering citations only at the end.** If citations appear as [1], [2] in the streamed text but are only resolved after the stream completes, the user sees raw brackets during generation. Parse and render citations inline as the text streams.
3. **Not measuring TTFT on the client side.** Server-side TTFT measures time from request receipt to first token emission, but the user experiences network latency too. Measure TTFT from query submission to first visible text.
4. **Forgetting to cancel streams on navigation.** If the user navigates away and the EventSource is not closed, the server continues generating (wasting LLM tokens) and the client leaks connections.
5. **No fallback for browsers without EventSource.** While modern browsers support EventSource natively, some corporate environments block SSE. Implement a polling fallback or use a library like `eventsource-parser`.
6. **Not handling partial JSON in SSE data.** If a network issue splits an SSE event across two chunks, the JSON parse fails. Buffer until `\n\n` delimiter before parsing.

---

## References

- MDN EventSource: https://developer.mozilla.org/en-US/docs/Web/API/EventSource
- Vercel AI SDK useChat: https://sdk.vercel.ai/docs/ai-sdk-ui/chatbot
- httpx streaming: https://www.python-httpx.org/advanced/streaming/
