# Streaming RAG -- Streaming LLM Responses with Citations

## TL;DR

Streaming RAG delivers generated tokens to the user as they are produced rather than waiting for the entire response to complete. This reduces perceived latency from 3-5 seconds (time to complete response) to 200-500 milliseconds (time to first token, TTFT). The challenge unique to RAG: citations and source references must be rendered alongside the streaming text, but the LLM may not emit citation markers until mid-response. This requires client-side buffering, progressive citation rendering, and careful handling of the retrieval-to-generation handoff. This overview covers the streaming architecture, TTFT optimization, citation handling patterns, and protocol choices (SSE vs WebSocket). Subsequent articles cover server-side implementation (SSE, async generators, Vercel AI SDK, LangChain astream) and client-side patterns (progressive disclosure, citation rendering).

---

## Why Streaming Matters for RAG

### The Latency Problem

```
Non-streaming RAG:
  Query -> Retrieve (200ms) -> Generate (3000ms) -> Display
  User waits: 3200ms (staring at spinner)

Streaming RAG:
  Query -> Retrieve (200ms) -> Start streaming (200ms) -> Tokens arrive continuously
  User sees first token: 400ms (TTFT)
  Full response: 3200ms (same total, but user reads along)
```

### TTFT (Time to First Token) Breakdown

| Stage | Latency | Optimization |
|-------|---------|-------------|
| Query processing | 50-100ms | Cache rewrites |
| Embedding | 50-100ms | Cache embeddings |
| Vector search | 20-50ms | Optimize index |
| Reranking | 100-300ms | Skip or async |
| LLM TTFT | 200-500ms | Prompt caching, smaller model |
| **Total TTFT** | **420-1050ms** | **Target: < 500ms** |

---

## Streaming Architecture

### Server-Sent Events (SSE) vs WebSocket

| Feature | SSE | WebSocket |
|---------|-----|-----------|
| Direction | Server -> Client only | Bidirectional |
| Protocol | HTTP/1.1 or HTTP/2 | WebSocket protocol |
| Reconnection | Built-in auto-reconnect | Manual |
| Browser support | Native EventSource API | Native WebSocket API |
| Proxy compatibility | Excellent | May need configuration |
| **Best for RAG** | **Yes (unidirectional streaming)** | Overkill unless chat |

### Basic Streaming Pipeline

```python
import asyncio
import json
import time
from typing import AsyncIterator
from dataclasses import dataclass


@dataclass
class StreamChunk:
    """A single chunk in the RAG stream."""

    type: str  # "retrieval_start", "retrieval_done", "token", "citation", "done"
    data: dict


class StreamingRAGPipeline:
    """RAG pipeline that yields results as a stream.

    The stream emits events for each pipeline stage,
    allowing the client to show progressive updates.
    """

    def __init__(self, retriever, llm, reranker=None):
        self.retriever = retriever
        self.llm = llm
        self.reranker = reranker

    async def stream(
        self, query: str
    ) -> AsyncIterator[StreamChunk]:
        """Stream RAG response as a sequence of chunks."""
        stream_start = time.perf_counter()

        # Phase 1: Signal retrieval start
        yield StreamChunk(
            type="retrieval_start",
            data={"query": query},
        )

        # Phase 2: Retrieve documents
        docs = await asyncio.to_thread(
            self.retriever.invoke, query
        )

        # Phase 3: Rerank if available
        if self.reranker:
            docs = await asyncio.to_thread(
                self.reranker.rerank, query, docs
            )

        sources = [
            {
                "id": d.metadata.get("source", f"doc_{i}"),
                "title": d.metadata.get("title", ""),
                "score": d.metadata.get("score", 0),
            }
            for i, d in enumerate(docs[:5])
        ]

        # Phase 4: Signal retrieval complete with sources
        retrieval_time = (time.perf_counter() - stream_start) * 1000
        yield StreamChunk(
            type="retrieval_done",
            data={
                "sources": sources,
                "retrieval_time_ms": retrieval_time,
            },
        )

        # Phase 5: Stream LLM generation
        context = "\n\n".join(d.page_content for d in docs[:5])
        prompt = f"Context:\n{context}\n\nQuestion: {query}"

        async for token in self._stream_llm(prompt):
            yield StreamChunk(type="token", data={"text": token})

        # Phase 6: Signal completion
        total_time = (time.perf_counter() - stream_start) * 1000
        yield StreamChunk(
            type="done",
            data={"total_time_ms": total_time},
        )

    async def _stream_llm(self, prompt: str) -> AsyncIterator[str]:
        """Stream tokens from the LLM."""
        from anthropic import AsyncAnthropic

        client = AsyncAnthropic()

        async with client.messages.stream(
            model="claude-sonnet-4-20250514",
            max_tokens=2048,
            messages=[{"role": "user", "content": prompt}],
        ) as stream:
            async for text in stream.text_stream:
                yield text
```

### SSE Endpoint (FastAPI)

```python
from fastapi import FastAPI, Request
from fastapi.responses import StreamingResponse

app = FastAPI()
pipeline = StreamingRAGPipeline(retriever, llm)


@app.get("/api/rag/stream")
async def stream_rag(query: str):
    """SSE endpoint for streaming RAG responses."""

    async def event_generator():
        async for chunk in pipeline.stream(query):
            data = json.dumps({
                "type": chunk.type,
                **chunk.data,
            })
            yield f"data: {data}\n\n"

    return StreamingResponse(
        event_generator(),
        media_type="text/event-stream",
        headers={
            "Cache-Control": "no-cache",
            "Connection": "keep-alive",
            "X-Accel-Buffering": "no",  # Disable nginx buffering
        },
    )
```

---

## Citation Handling in Streams

### The Citation Problem

```
Non-streaming: LLM generates full response with citations [1][2],
               then client renders everything at once.

Streaming:     Tokens arrive one by one: "The" "rate" "limit" "is"
               "1000" "/min" "[" "1" "]" "."
               When should the client render the citation?
               What if the citation reference comes after the claim?
```

### Citation-Aware Streaming

```python
import re
from typing import AsyncIterator


class CitationAwareStreamer:
    """Buffer streaming tokens to handle citations properly.

    Buffers tokens until a citation marker is complete,
    then emits the text with the citation attached.
    """

    CITATION_PATTERN = re.compile(r"\[(\d+)\]")

    def __init__(self, sources: list[dict]):
        self.sources = {
            str(i + 1): s for i, s in enumerate(sources)
        }
        self.buffer = ""
        self.citations_seen: set[str] = set()

    async def process_stream(
        self, token_stream: AsyncIterator[str]
    ) -> AsyncIterator[dict]:
        """Process token stream and emit text + citation events."""
        async for token in token_stream:
            self.buffer += token

            # Check for complete citation markers
            while True:
                match = self.CITATION_PATTERN.search(self.buffer)
                if not match:
                    # No citation marker found
                    # Emit all buffered text except last few chars
                    # (which might be start of a citation like "[")
                    safe_end = self.buffer.rfind("[")
                    if safe_end == -1:
                        # No potential citation start, emit all
                        if self.buffer:
                            yield {
                                "type": "text",
                                "content": self.buffer,
                            }
                            self.buffer = ""
                    else:
                        # Keep potential citation start in buffer
                        if safe_end > 0:
                            yield {
                                "type": "text",
                                "content": self.buffer[:safe_end],
                            }
                            self.buffer = self.buffer[safe_end:]
                    break

                # Found a citation marker
                cite_id = match.group(1)

                # Emit text before the citation
                text_before = self.buffer[: match.start()]
                if text_before:
                    yield {"type": "text", "content": text_before}

                # Emit the citation event
                if cite_id not in self.citations_seen:
                    self.citations_seen.add(cite_id)
                    source = self.sources.get(cite_id, {})
                    yield {
                        "type": "citation",
                        "id": cite_id,
                        "source": source,
                    }
                else:
                    yield {
                        "type": "citation_ref",
                        "id": cite_id,
                    }

                # Remove processed text from buffer
                self.buffer = self.buffer[match.end() :]

        # Flush remaining buffer
        if self.buffer:
            yield {"type": "text", "content": self.buffer}
```

---

## TTFT Optimization

### Parallel Retrieval and LLM Warming

```python
class OptimizedStreamingRAG:
    """RAG pipeline optimized for minimum TTFT.

    Techniques:
    1. Start LLM generation as soon as first docs are retrieved
    2. Use prompt caching for stable system prompt
    3. Skip reranking (or run async) to reduce TTFT
    """

    def __init__(self, retriever, reranker=None):
        self.retriever = retriever
        self.reranker = reranker

    async def stream(
        self, query: str
    ) -> AsyncIterator[StreamChunk]:
        """Optimized streaming with minimal TTFT."""
        from anthropic import AsyncAnthropic

        client = AsyncAnthropic()

        # Step 1: Retrieve (blocking -- needed before generation)
        docs = await asyncio.to_thread(
            self.retriever.invoke, query
        )

        # Step 2: Start streaming immediately (skip reranking for TTFT)
        # Reranking can run in background and update citations later
        context = "\n\n".join(d.page_content for d in docs[:5])
        sources = [d.metadata for d in docs[:5]]

        yield StreamChunk(
            type="retrieval_done",
            data={"sources": sources},
        )

        # Step 3: Stream with prompt caching
        async with client.messages.stream(
            model="claude-sonnet-4-20250514",
            max_tokens=2048,
            system=[
                {
                    "type": "text",
                    "text": (
                        "Answer based on the provided context. "
                        "Cite sources using [1], [2], etc."
                    ),
                    "cache_control": {"type": "ephemeral"},
                },
                {
                    "type": "text",
                    "text": f"Context:\n{context}",
                    "cache_control": {"type": "ephemeral"},
                },
            ],
            messages=[{"role": "user", "content": query}],
        ) as stream:
            async for text in stream.text_stream:
                yield StreamChunk(
                    type="token", data={"text": text}
                )

        yield StreamChunk(type="done", data={})
```

---

## Common Pitfalls

1. **Not disabling response buffering.** Nginx, CloudFront, and other proxies buffer responses by default. Streaming requires disabling buffering with `X-Accel-Buffering: no` (nginx) or equivalent headers.
2. **Blocking on reranking before streaming starts.** Reranking adds 100-300ms to TTFT. Either skip reranking or run it in parallel with the start of generation and update the citation list after.
3. **Not handling SSE reconnection.** If the connection drops mid-stream, the client must reconnect and resume. Use event IDs so the server can replay missed events.
4. **Rendering citations only after the full response.** Users see claims without sources for seconds. Use citation-aware streaming to render citations inline as they appear in the token stream.
5. **Not measuring TTFT separately from total latency.** A 5-second response with 300ms TTFT feels fast. A 3-second response with 2-second TTFT feels slow. Track and optimize TTFT independently.
6. **Sending tokens one at a time over the network.** Individual token SSE events have HTTP overhead. Batch tokens into small groups (every 50-100ms) to reduce network overhead while maintaining perceived smoothness.

---

## References

- Vercel AI SDK: https://sdk.vercel.ai/docs
- MDN Server-Sent Events: https://developer.mozilla.org/en-US/docs/Web/API/Server-sent_events
- Anthropic streaming: https://docs.anthropic.com/en/api/messages-streaming
- LangChain astream: https://python.langchain.com/docs/how_to/streaming/
