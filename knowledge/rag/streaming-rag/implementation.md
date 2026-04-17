# Streaming RAG Implementation -- SSE, WebSocket, Async Generators, Vercel AI SDK, and LangChain astream

## TL;DR

This article covers server-side streaming implementation for RAG systems across five approaches: raw SSE with FastAPI async generators, WebSocket streaming for bidirectional chat, the Vercel AI SDK pattern (streaming with tool calls for retrieval), LangChain's astream and astream_events APIs, and LlamaIndex's streaming chat engine. Each approach has different tradeoffs in complexity, framework coupling, and feature support. The article includes complete working code for each, including proper error handling, heartbeat keepalives, and backpressure management.

---

## SSE with FastAPI

### Complete SSE Implementation

```python
import asyncio
import json
import time
import uuid
import logging
from typing import AsyncIterator

from fastapi import FastAPI, Query, HTTPException
from fastapi.responses import StreamingResponse
from anthropic import AsyncAnthropic

logger = logging.getLogger(__name__)

app = FastAPI()


class SSEStreamingRAG:
    """Server-Sent Events streaming RAG implementation."""

    def __init__(self, retriever, model: str = "claude-sonnet-4-20250514"):
        self.retriever = retriever
        self.client = AsyncAnthropic()
        self.model = model

    async def stream_response(
        self, query: str, session_id: str | None = None
    ) -> AsyncIterator[str]:
        """Generate SSE events for a RAG query."""
        request_id = str(uuid.uuid4())

        # Event: stream start
        yield self._sse_event("stream_start", {
            "request_id": request_id,
            "query": query,
        })

        # Phase 1: Retrieval
        try:
            docs = await asyncio.to_thread(
                self.retriever.invoke, query
            )
        except Exception as e:
            yield self._sse_event("error", {
                "message": f"Retrieval failed: {str(e)}",
            })
            return

        sources = [
            {
                "index": i + 1,
                "id": d.metadata.get("source", f"doc_{i}"),
                "title": d.metadata.get("title", "Untitled"),
                "snippet": d.page_content[:200],
            }
            for i, d in enumerate(docs[:5])
        ]

        yield self._sse_event("sources", {"sources": sources})

        # Phase 2: Streaming generation
        context = "\n\n".join(
            f"[{i+1}] {d.page_content}" for i, d in enumerate(docs[:5])
        )

        try:
            token_count = 0
            async with self.client.messages.stream(
                model=self.model,
                max_tokens=2048,
                system=(
                    "Answer based on the provided context documents. "
                    "Cite sources using [1], [2], etc."
                ),
                messages=[{
                    "role": "user",
                    "content": f"Context:\n{context}\n\nQuestion: {query}",
                }],
            ) as stream:
                buffer = ""
                last_emit = time.perf_counter()

                async for text in stream.text_stream:
                    buffer += text
                    token_count += 1

                    # Batch tokens: emit every 50ms or 10 tokens
                    now = time.perf_counter()
                    if (
                        now - last_emit > 0.05
                        or len(buffer) > 50
                    ):
                        yield self._sse_event("token", {
                            "text": buffer,
                        })
                        buffer = ""
                        last_emit = now

                # Flush remaining buffer
                if buffer:
                    yield self._sse_event("token", {"text": buffer})

            # Get final usage
            message = await stream.get_final_message()
            usage = {
                "input_tokens": message.usage.input_tokens,
                "output_tokens": message.usage.output_tokens,
            }

        except Exception as e:
            yield self._sse_event("error", {
                "message": f"Generation failed: {str(e)}",
            })
            return

        # Event: stream complete
        yield self._sse_event("done", {
            "request_id": request_id,
            "token_count": token_count,
            "usage": usage,
        })

    @staticmethod
    def _sse_event(event_type: str, data: dict) -> str:
        """Format a Server-Sent Event."""
        payload = json.dumps({"type": event_type, **data})
        return f"event: {event_type}\ndata: {payload}\n\n"


rag = SSEStreamingRAG(retriever=None)  # Replace with actual retriever


@app.get("/api/stream")
async def stream_endpoint(
    query: str = Query(..., min_length=1, max_length=2000),
    session_id: str = Query(default=None),
):
    """SSE streaming endpoint for RAG queries."""

    async def generate():
        async for event in rag.stream_response(query, session_id):
            yield event

        # Send keepalive comment to prevent timeout
        yield ": keepalive\n\n"

    return StreamingResponse(
        generate(),
        media_type="text/event-stream",
        headers={
            "Cache-Control": "no-cache, no-store, must-revalidate",
            "Connection": "keep-alive",
            "X-Accel-Buffering": "no",
        },
    )
```

---

## WebSocket Streaming

### For Conversational RAG

```python
from fastapi import WebSocket, WebSocketDisconnect
import json


class WebSocketRAGHandler:
    """WebSocket handler for conversational streaming RAG.

    WebSockets are appropriate when:
    - The client needs to send multiple messages (chat)
    - Bidirectional communication is needed (cancel, feedback)
    - The connection is long-lived (chat session)
    """

    def __init__(self, retriever, model="claude-sonnet-4-20250514"):
        self.retriever = retriever
        self.client = AsyncAnthropic()
        self.model = model

    async def handle_connection(self, websocket: WebSocket):
        """Handle a single WebSocket connection."""
        await websocket.accept()
        chat_history = []

        try:
            while True:
                # Receive message from client
                data = await websocket.receive_json()
                query = data.get("query", "")

                if not query:
                    await websocket.send_json({
                        "type": "error",
                        "message": "Empty query",
                    })
                    continue

                # Stream response
                await self._stream_response(
                    websocket, query, chat_history
                )

        except WebSocketDisconnect:
            logger.info("WebSocket disconnected")

    async def _stream_response(
        self,
        websocket: WebSocket,
        query: str,
        chat_history: list,
    ):
        """Stream a RAG response over WebSocket."""
        # Retrieve
        docs = await asyncio.to_thread(
            self.retriever.invoke, query
        )

        sources = [d.metadata for d in docs[:5]]
        await websocket.send_json({
            "type": "sources",
            "sources": sources,
        })

        # Stream generation
        context = "\n\n".join(d.page_content for d in docs[:5])
        messages = list(chat_history) + [
            {"role": "user", "content": f"Context:\n{context}\n\nQ: {query}"},
        ]

        full_response = ""
        async with self.client.messages.stream(
            model=self.model,
            max_tokens=2048,
            messages=messages,
        ) as stream:
            async for text in stream.text_stream:
                full_response += text
                await websocket.send_json({
                    "type": "token",
                    "text": text,
                })

        # Update history
        chat_history.append({"role": "user", "content": query})
        chat_history.append({"role": "assistant", "content": full_response})

        await websocket.send_json({"type": "done"})


ws_handler = WebSocketRAGHandler(retriever=None)


@app.websocket("/ws/chat")
async def websocket_chat(websocket: WebSocket):
    await ws_handler.handle_connection(websocket)
```

---

## LangChain astream and astream_events

### astream for Simple Streaming

```python
from langchain_anthropic import ChatAnthropic
from langchain_core.prompts import ChatPromptTemplate
from langchain_core.output_parsers import StrOutputParser


async def langchain_stream_rag(
    query: str, retriever, model: str = "claude-sonnet-4-20250514"
) -> AsyncIterator[str]:
    """Stream RAG using LangChain's astream."""
    llm = ChatAnthropic(model=model, streaming=True)

    # Retrieve first (not streamed)
    docs = await asyncio.to_thread(retriever.invoke, query)
    context = "\n\n".join(d.page_content for d in docs[:5])

    prompt = ChatPromptTemplate.from_messages([
        ("system", "Answer based on context:\n{context}"),
        ("human", "{query}"),
    ])

    chain = prompt | llm | StrOutputParser()

    # astream yields string chunks
    async for chunk in chain.astream({
        "context": context,
        "query": query,
    }):
        yield chunk
```

### astream_events for Detailed Pipeline Events

```python
from langchain.chains import create_retrieval_chain
from langchain.chains.combine_documents import create_stuff_documents_chain


async def langchain_stream_events(
    query: str,
    rag_chain,
) -> AsyncIterator[dict]:
    """Stream detailed events from LangChain RAG chain.

    astream_events provides visibility into every chain step:
    retriever, LLM, parser, etc.
    """
    async for event in rag_chain.astream_events(
        {"input": query},
        version="v2",
    ):
        kind = event["event"]

        if kind == "on_retriever_start":
            yield {
                "type": "retrieval_start",
                "query": event["data"].get("input", {}).get("query", ""),
            }

        elif kind == "on_retriever_end":
            docs = event["data"].get("output", [])
            yield {
                "type": "retrieval_done",
                "num_docs": len(docs),
                "sources": [
                    d.metadata.get("source", "unknown")
                    for d in docs[:5]
                ],
            }

        elif kind == "on_chat_model_stream":
            chunk = event["data"].get("chunk", "")
            if hasattr(chunk, "content") and chunk.content:
                yield {
                    "type": "token",
                    "text": chunk.content,
                }

        elif kind == "on_chain_end":
            if event["name"] == "RunnableSequence":
                yield {"type": "done"}
```

---

## Vercel AI SDK Pattern

### Streaming with useChat (Server)

```python
# This pattern matches Vercel AI SDK's expected format
# for use with the `useChat` React hook

from fastapi import FastAPI, Request
from fastapi.responses import StreamingResponse


@app.post("/api/chat")
async def vercel_ai_chat(request: Request):
    """Endpoint compatible with Vercel AI SDK useChat hook.

    The Vercel AI SDK expects a specific streaming format:
    each line is a data chunk prefixed with type indicators.
    """
    body = await request.json()
    messages = body.get("messages", [])
    query = messages[-1]["content"] if messages else ""

    async def generate():
        # Retrieve documents
        docs = await asyncio.to_thread(retriever.invoke, query)
        context = "\n\n".join(d.page_content for d in docs[:5])

        # Send source annotations (Vercel AI SDK data)
        sources = [d.metadata for d in docs[:5]]
        source_annotation = json.dumps({
            "sources": sources,
        })
        yield f"8:{source_annotation}\n"

        # Stream text tokens
        client = AsyncAnthropic()
        async with client.messages.stream(
            model="claude-sonnet-4-20250514",
            max_tokens=2048,
            system=f"Answer based on context:\n{context}",
            messages=[{"role": "user", "content": query}],
        ) as stream:
            async for text in stream.text_stream:
                # Vercel AI SDK text format: 0:"text chunk"
                escaped = json.dumps(text)
                yield f"0:{escaped}\n"

        # Signal completion
        yield 'd:{"finishReason":"stop"}\n'

    return StreamingResponse(
        generate(),
        media_type="text/plain",
        headers={
            "X-Vercel-AI-Data-Stream": "v1",
        },
    )
```

---

## Error Handling in Streams

### Graceful Error Recovery

```python
class ResilientStreamingRAG:
    """Streaming RAG with error handling at each stage."""

    async def stream(self, query: str) -> AsyncIterator[str]:
        """Stream with error handling and recovery."""
        try:
            # Retrieval with timeout
            docs = await asyncio.wait_for(
                asyncio.to_thread(self.retriever.invoke, query),
                timeout=5.0,
            )
        except asyncio.TimeoutError:
            yield self._sse_event("error", {
                "stage": "retrieval",
                "message": "Search timed out. Generating without context.",
                "recoverable": True,
            })
            docs = []
        except Exception as e:
            yield self._sse_event("error", {
                "stage": "retrieval",
                "message": str(e),
                "recoverable": True,
            })
            docs = []

        # Generate (with or without context)
        context = "\n\n".join(d.page_content for d in docs[:5]) if docs else ""
        try:
            async for token in self._stream_llm(query, context):
                yield self._sse_event("token", {"text": token})
        except Exception as e:
            yield self._sse_event("error", {
                "stage": "generation",
                "message": str(e),
                "recoverable": False,
            })
            return

        yield self._sse_event("done", {})

    async def _stream_llm(
        self, query: str, context: str
    ) -> AsyncIterator[str]:
        """Stream LLM with retry on transient failures."""
        client = AsyncAnthropic()
        prompt = f"Context:\n{context}\n\nQuestion: {query}" if context else query

        async with client.messages.stream(
            model="claude-sonnet-4-20250514",
            max_tokens=2048,
            messages=[{"role": "user", "content": prompt}],
        ) as stream:
            async for text in stream.text_stream:
                yield text

    @staticmethod
    def _sse_event(event_type: str, data: dict) -> str:
        payload = json.dumps({"type": event_type, **data})
        return f"event: {event_type}\ndata: {payload}\n\n"
```

---

## Common Pitfalls

1. **Not batching token emissions.** Sending one SSE event per token creates excessive network overhead. Batch tokens every 50-100ms for smoother streaming with lower overhead.
2. **No heartbeat/keepalive.** Long retrieval phases (before generation starts) may cause proxies or load balancers to close the connection. Send periodic SSE comments (`: keepalive`) during retrieval.
3. **Not handling client disconnection.** If the user navigates away mid-stream, the server must detect the disconnection and stop generation to avoid wasting LLM tokens.
4. **Blocking the event loop during retrieval.** Vector store queries are synchronous I/O. Use `asyncio.to_thread()` to run them without blocking the event loop.
5. **Not sending error events.** If generation fails mid-stream, the client sees a hanging spinner. Always send an error event so the client can display an appropriate message.
6. **Ignoring backpressure.** If the client cannot consume tokens as fast as the server produces them, the server buffer grows. In high-concurrency scenarios, implement flow control or connection limits.

---

## References

- FastAPI StreamingResponse: https://fastapi.tiangolo.com/advanced/custom-response/#streamingresponse
- Anthropic streaming: https://docs.anthropic.com/en/api/messages-streaming
- Vercel AI SDK: https://sdk.vercel.ai/docs/ai-sdk-ui
- LangChain streaming: https://python.langchain.com/docs/how_to/streaming/
