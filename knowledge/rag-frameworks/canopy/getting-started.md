# Canopy -- Getting Started: CLI Server, Chat, and Customization

## Overview

This guide walks through installing Canopy, setting up a Pinecone-backed RAG server, uploading documents, and querying through the OpenAI-compatible API. By the end you will have a running RAG server that can be used as a drop-in replacement for the OpenAI chat completions endpoint.

---

## Installation

```bash
# Install Canopy
pip install canopy-sdk

# Verify
python -c "import canopy; print('Canopy installed')"
```

### Environment Setup

```bash
# Required: Pinecone API key (from https://app.pinecone.io/)
export PINECONE_API_KEY="your-pinecone-api-key"

# Required: OpenAI API key (for embeddings and generation)
export OPENAI_API_KEY="sk-..."

# Optional: customize the index name
export CANOPY_INDEX_NAME="my-knowledge-base"
```

---

## Step 1: Create the Pinecone Index

```bash
# Create a new Canopy-compatible index
canopy new --index-name my-knowledge-base
```

Or programmatically:

```python
from canopy.knowledge_base import KnowledgeBase

kb = KnowledgeBase(index_name="my-knowledge-base")

# Create a serverless index
kb.create_canopy_index(
    spec={
        "serverless": {
            "cloud": "aws",
            "region": "us-east-1",
        }
    }
)

print("Index created successfully")
```

### Verify the Index

```python
from pinecone import Pinecone

pc = Pinecone()
index = pc.Index("my-knowledge-base")
stats = index.describe_index_stats()
print(f"Index stats: {stats}")
```

---

## Step 2: Upload Documents

### Using the CLI

```bash
# Upload a directory of documents
canopy upsert --index-name my-knowledge-base --data-path ./docs/

# Upload a specific file
canopy upsert --index-name my-knowledge-base --data-path ./docs/architecture.md

# Upload from a JSONL file (each line: {"id": "...", "text": "...", "source": "...", "metadata": {}})
canopy upsert --index-name my-knowledge-base --data-path ./documents.jsonl
```

### Using Python

```python
from canopy.knowledge_base import KnowledgeBase
from canopy.models.data_models import Document

kb = KnowledgeBase(index_name="my-knowledge-base")

# Upload individual documents
documents = [
    Document(
        id="doc-001",
        text="""Pinecone is a managed vector database designed for machine learning applications.
It supports real-time vector similarity search with sub-100ms latency at any scale.
Key features include serverless deployment, metadata filtering, and sparse-dense hybrid search.""",
        source="pinecone-docs/overview",
        metadata={"category": "product", "version": "2024"},
    ),
    Document(
        id="doc-002",
        text="""Canopy is Pinecone's open-source RAG framework. It provides a production-ready
RAG server with an OpenAI-compatible API. Canopy handles document chunking, embedding,
retrieval, and LLM generation out of the box.""",
        source="canopy-docs/overview",
        metadata={"category": "framework"},
    ),
    Document(
        id="doc-003",
        text="""To configure Canopy, set environment variables for PINECONE_API_KEY and
OPENAI_API_KEY. Then create a Pinecone index using canopy new, upload documents
with canopy upsert, and start the server with canopy start.""",
        source="canopy-docs/quickstart",
        metadata={"category": "setup"},
    ),
]

kb.upsert(documents)
print(f"Uploaded {len(documents)} documents")
```

### Uploading from Files

```python
from canopy.knowledge_base import KnowledgeBase
from canopy.models.data_models import Document
from pathlib import Path

kb = KnowledgeBase(index_name="my-knowledge-base")


def load_documents_from_directory(directory: str) -> list[Document]:
    """Load markdown and text files as Canopy documents."""
    docs = []
    for path in Path(directory).glob("**/*"):
        if path.suffix not in {".md", ".txt", ".rst"}:
            continue
        text = path.read_text(encoding="utf-8")
        if not text.strip():
            continue
        docs.append(Document(
            id=f"file-{path.stem}",
            text=text,
            source=str(path.relative_to(directory)),
            metadata={
                "filename": path.name,
                "category": path.parent.name,
                "extension": path.suffix,
            },
        ))
    return docs


documents = load_documents_from_directory("./knowledge_base")
print(f"Loaded {len(documents)} documents")

# Upload in batches
batch_size = 50
for i in range(0, len(documents), batch_size):
    batch = documents[i : i + batch_size]
    kb.upsert(batch)
    print(f"Uploaded {min(i + batch_size, len(documents))}/{len(documents)}")
```

---

## Step 3: Start the Server

### Using the CLI

```bash
# Start the Canopy server
canopy start --index-name my-knowledge-base

# With custom port
canopy start --index-name my-knowledge-base --port 8000

# With custom configuration
canopy start --config canopy_config.yaml
```

### Using Python

```python
from canopy.chat_engine import ChatEngine
from canopy.knowledge_base import KnowledgeBase
from canopy.context_engine import ContextEngine
from canopy.llm import OpenAILLM

# Build the engine
kb = KnowledgeBase(index_name="my-knowledge-base")
context_engine = ContextEngine(knowledge_base=kb)
llm = OpenAILLM(model_name="gpt-4o-mini", temperature=0.0)

chat_engine = ChatEngine(
    context_engine=context_engine,
    llm=llm,
    system_prompt="""You are a knowledgeable assistant. Answer questions using the provided context.
If the context does not contain enough information, say "I don't have enough information."
Always mention the source of information when available.""",
)
```

---

## Step 4: Query the Server

### Using curl

```bash
# Health check
curl http://localhost:8000/v1/health

# Chat completion (OpenAI-compatible)
curl http://localhost:8000/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "gpt-4o-mini",
    "messages": [
      {"role": "user", "content": "What is Pinecone?"}
    ]
  }'

# Streaming
curl http://localhost:8000/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "gpt-4o-mini",
    "messages": [
      {"role": "user", "content": "How do I set up Canopy?"}
    ],
    "stream": true
  }'
```

### Using the OpenAI SDK

```python
from openai import OpenAI

# Point at Canopy instead of OpenAI
client = OpenAI(
    base_url="http://localhost:8000/v1",
    api_key="not-needed",
)

# Regular chat completion (with RAG)
response = client.chat.completions.create(
    model="gpt-4o-mini",
    messages=[
        {"role": "user", "content": "What features does Pinecone offer?"},
    ],
)
print(response.choices[0].message.content)

# Streaming
stream = client.chat.completions.create(
    model="gpt-4o-mini",
    messages=[
        {"role": "user", "content": "How do I configure Canopy for production?"},
    ],
    stream=True,
)

for chunk in stream:
    content = chunk.choices[0].delta.content
    if content:
        print(content, end="", flush=True)
print()
```

### Using Python Directly (No Server)

```python
from canopy.chat_engine import ChatEngine
from canopy.models.data_models import MessageBase

chat_engine = ChatEngine()

# Single query
messages = [
    MessageBase(role="user", content="What is Canopy?"),
]

response = chat_engine.chat(messages=messages)
print(response.choices[0].message.content)

# Multi-turn conversation
messages = [
    MessageBase(role="user", content="What is Pinecone?"),
]
response1 = chat_engine.chat(messages=messages)
print(f"A1: {response1.choices[0].message.content}\n")

messages.append(
    MessageBase(role="assistant", content=response1.choices[0].message.content)
)
messages.append(
    MessageBase(role="user", content="How does its serverless offering work?")
)
response2 = chat_engine.chat(messages=messages)
print(f"A2: {response2.choices[0].message.content}")
```

---

## Step 5: Context-Only Queries

Retrieve context without generation (useful for debugging or custom pipelines):

```python
from canopy.context_engine import ContextEngine
from canopy.knowledge_base import KnowledgeBase

kb = KnowledgeBase(index_name="my-knowledge-base")
context_engine = ContextEngine(knowledge_base=kb)

# Get raw context
context = context_engine.query(
    queries=[{"text": "Pinecone serverless architecture"}],
    max_context_tokens=2000,
)

print(f"Retrieved {len(context.content)} snippets ({context.num_tokens} tokens)")
for snippet in context.content:
    print(f"\n--- Source: {snippet.source} ---")
    print(snippet.text[:300])
```

### Via the API

```bash
curl http://localhost:8000/v1/context/query \
  -H "Content-Type: application/json" \
  -d '{
    "queries": [{"text": "How does Canopy work?"}],
    "max_tokens": 2000
  }'
```

---

## Step 6: Document Management

### Deleting Documents

```python
from canopy.knowledge_base import KnowledgeBase

kb = KnowledgeBase(index_name="my-knowledge-base")

# Delete by document ID
kb.delete(document_ids=["doc-001", "doc-002"])

# Delete all documents (recreate the index)
kb.delete(delete_all=True)
```

### Updating Documents

```python
# Canopy uses upsert semantics: re-uploading with the same ID overwrites
updated_doc = Document(
    id="doc-001",  # same ID
    text="Updated content for Pinecone documentation...",
    source="pinecone-docs/overview-v2",
    metadata={"category": "product", "version": "2025"},
)

kb.upsert([updated_doc])
```

### Checking Index Status

```python
from pinecone import Pinecone

pc = Pinecone()
index = pc.Index("my-knowledge-base")

stats = index.describe_index_stats()
print(f"Total vectors: {stats.total_vector_count}")
print(f"Dimension: {stats.dimension}")
for ns, ns_stats in stats.namespaces.items():
    print(f"Namespace '{ns}': {ns_stats.vector_count} vectors")
```

---

## Customization

### Custom System Prompt

```python
chat_engine = ChatEngine(
    system_prompt="""You are a technical support assistant for our product.
Rules:
1. Only answer questions related to our product documentation
2. If asked about competitors, politely redirect to our features
3. Include specific references to documentation sections when possible
4. If unsure, say "Let me connect you with a human agent"
5. Keep answers concise but thorough""",
)
```

### Custom Top-K and Token Budget

```python
from canopy.knowledge_base import KnowledgeBase
from canopy.context_engine import ContextEngine
from canopy.chat_engine import ChatEngine

# Retrieve more documents for complex queries
kb = KnowledgeBase(index_name="my-knowledge-base", default_top_k=10)

context_engine = ContextEngine(
    knowledge_base=kb,
    max_context_tokens=6000,  # larger context window
)

chat_engine = ChatEngine(
    context_engine=context_engine,
    max_generated_tokens=1500,
)
```

### Using Pinecone Namespaces for Multi-Tenancy

```python
from canopy.knowledge_base import KnowledgeBase
from canopy.models.data_models import Document

# Use namespaces to isolate tenant data
kb_tenant_a = KnowledgeBase(
    index_name="shared-index",
    namespace="tenant-a",
)

kb_tenant_b = KnowledgeBase(
    index_name="shared-index",
    namespace="tenant-b",
)

# Upload tenant-specific documents
kb_tenant_a.upsert([
    Document(id="a-001", text="Tenant A specific documentation...", source="tenant-a"),
])

kb_tenant_b.upsert([
    Document(id="b-001", text="Tenant B specific documentation...", source="tenant-b"),
])

# Queries are automatically scoped to the namespace
results_a = kb_tenant_a.query([{"text": "documentation"}], top_k=5)
results_b = kb_tenant_b.query([{"text": "documentation"}], top_k=5)
# results_a only contains tenant A documents
# results_b only contains tenant B documents
```

---

## Integration with Existing Applications

### FastAPI Wrapper

```python
from fastapi import FastAPI
from pydantic import BaseModel
from canopy.chat_engine import ChatEngine
from canopy.models.data_models import MessageBase

app = FastAPI()
engine = ChatEngine()


class ChatRequest(BaseModel):
    messages: list[dict]
    stream: bool = False


@app.post("/chat")
async def chat(request: ChatRequest):
    messages = [
        MessageBase(role=m["role"], content=m["content"])
        for m in request.messages
    ]

    if request.stream:
        from fastapi.responses import StreamingResponse
        import json

        def generate():
            for chunk in engine.chat(messages=messages, stream=True):
                data = {
                    "content": chunk.choices[0].delta.content or "",
                    "finish_reason": chunk.choices[0].finish_reason,
                }
                yield f"data: {json.dumps(data)}\n\n"
            yield "data: [DONE]\n\n"

        return StreamingResponse(generate(), media_type="text/event-stream")

    response = engine.chat(messages=messages)
    return {
        "answer": response.choices[0].message.content,
        "usage": {
            "prompt_tokens": response.usage.prompt_tokens,
            "completion_tokens": response.usage.completion_tokens,
        },
    }
```

---

## Common Pitfalls

1. **Not creating the index before upserting**: `canopy new` or `kb.create_canopy_index()` must run before any upsert. Upserting to a nonexistent index fails.

2. **Missing environment variables**: Both `PINECONE_API_KEY` and `OPENAI_API_KEY` must be set. Canopy does not give clear errors for missing keys.

3. **Document ID collisions**: If you upsert documents with the same ID, the old document is overwritten. Use unique IDs when you want to keep both versions.

4. **Server port conflicts**: Default port 8000 may conflict with other services. Use `--port` to specify a different port.

5. **Large documents without chunking config**: Very large documents may produce chunks that exceed Pinecone's metadata size limit (40KB). Reduce chunk_size if you see metadata errors.

6. **Not handling streaming correctly**: When `stream=True`, the response is a generator. Do not try to access `.choices[0].message.content` on a streaming response -- iterate over chunks instead.

---

## References

- Canopy GitHub: https://github.com/pinecone-io/canopy
- Canopy quickstart: https://docs.pinecone.io/guides/gen-ai/canopy
- Pinecone documentation: https://docs.pinecone.io/
- OpenAI API compatibility: https://platform.openai.com/docs/api-reference/chat
