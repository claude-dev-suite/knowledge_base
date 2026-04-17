# Canopy -- Pinecone's RAG Framework

## Overview

Canopy is an open-source RAG framework built by Pinecone that provides a production-ready retrieval-augmented generation server. It wraps Pinecone's vector database with document processing, chunking, embedding, retrieval, and LLM generation into a single deployable service. Canopy exposes an OpenAI-compatible chat API, so existing applications using the OpenAI SDK can switch to RAG-powered responses by changing one endpoint URL.

Canopy's philosophy is opinionated defaults with escape hatches. It comes with sensible default chunking, embedding, and generation settings that work well for most use cases, while allowing customization of every component through configuration and subclassing.

This guide covers Canopy's architecture, core abstractions, and how it compares to building RAG from scratch with Pinecone.

---

## Architecture

```
                    +-------------------+
                    | OpenAI-compatible |
                    |    Chat API       |
                    +--------+----------+
                             |
                    +--------v----------+
                    |   Chat Engine     |
                    |  (orchestrates    |
                    |   retrieval +     |
                    |   generation)     |
                    +--------+----------+
                             |
              +--------------+--------------+
              |                             |
    +---------v---------+       +-----------v-----------+
    |   Context Engine  |       |    LLM (generation)   |
    | (retrieval +      |       | (OpenAI, Anthropic,   |
    |  context building)|       |  or any LiteLLM model)|
    +---------+---------+       +-----------------------+
              |
    +---------v---------+
    |   Knowledge Base  |
    | (Pinecone index + |
    |  chunker +        |
    |  embedding model) |
    +-------------------+
```

### Core Components

| Component | Purpose |
|-----------|---------|
| **ChatEngine** | Top-level orchestrator: takes user messages, retrieves context, generates response |
| **ContextEngine** | Manages retrieval from the knowledge base and formats context for the LLM |
| **KnowledgeBase** | Wraps a Pinecone index with document upload, chunking, and embedding |
| **Chunker** | Splits documents into chunks (default: `MarkdownChunker` for markdown, `RecursiveCharacterChunker` for text) |
| **ContextBuilder** | Formats retrieved chunks into a prompt context (with token budget management) |
| **LLM** | Abstraction over language models (default: OpenAI, supports any model via LiteLLM) |

### Data Flow

```
User message
  -> ChatEngine.chat()
    -> ContextEngine.query()
      -> KnowledgeBase.query()  [vector search in Pinecone]
      -> ContextBuilder.build()  [format + truncate to token budget]
    -> LLM.generate()  [with retrieved context as system message]
  -> Response (streaming or complete)
```

---

## Core Abstractions

### ChatEngine

The ChatEngine is the main entry point. It combines the ContextEngine and LLM:

```python
from canopy.chat_engine import ChatEngine
from canopy.models.data_models import MessageBase

# Create the engine (uses defaults from environment)
chat_engine = ChatEngine()

# Simple query
messages = [
    MessageBase(role="user", content="What is Pinecone's serverless architecture?")
]

response = chat_engine.chat(messages=messages)
print(response.choices[0].message.content)

# Streaming
for chunk in chat_engine.chat(messages=messages, stream=True):
    if chunk.choices[0].delta.content:
        print(chunk.choices[0].delta.content, end="", flush=True)
```

### ContextEngine

The ContextEngine retrieves and formats context:

```python
from canopy.context_engine import ContextEngine
from canopy.knowledge_base import KnowledgeBase

# Create knowledge base (connects to Pinecone)
kb = KnowledgeBase(index_name="my-canopy-index")

# Create context engine
context_engine = ContextEngine(knowledge_base=kb)

# Query for context
context = context_engine.query(
    queries=[
        {"text": "How does serverless Pinecone work?"},
    ],
    max_context_tokens=2000,
)

print(f"Context ({context.num_tokens} tokens):")
for snippet in context.content:
    print(f"  Source: {snippet.source}")
    print(f"  Text: {snippet.text[:200]}...")
```

### KnowledgeBase

The KnowledgeBase manages documents in Pinecone:

```python
from canopy.knowledge_base import KnowledgeBase
from canopy.models.data_models import Document

# Connect to or create a Pinecone index
kb = KnowledgeBase(index_name="my-canopy-index")

# Create the underlying Pinecone index if it does not exist
kb.create_canopy_index(
    spec={
        "serverless": {
            "cloud": "aws",
            "region": "us-east-1",
        }
    }
)

# Upsert documents
documents = [
    Document(
        id="doc-001",
        text="Pinecone serverless automatically scales based on usage. "
             "You pay only for the storage and compute you use, with no "
             "need to provision or manage infrastructure.",
        source="pinecone-docs",
        metadata={"category": "architecture", "version": "2024"},
    ),
    Document(
        id="doc-002",
        text="Pinecone indexes support up to 1 billion vectors with "
             "sub-100ms query latency at any scale.",
        source="pinecone-docs",
        metadata={"category": "performance"},
    ),
]

kb.upsert(documents)

# Query the knowledge base directly
results = kb.query(
    queries=[{"text": "Pinecone serverless scaling"}],
    top_k=5,
)

for result in results:
    for match in result:
        print(f"Score: {match.score:.4f}")
        print(f"Text: {match.text[:200]}...")
```

---

## Chunking

Canopy includes built-in chunkers that split documents before embedding:

### RecursiveCharacterChunker (Default)

```python
from canopy.knowledge_base.chunker import RecursiveCharacterChunker

chunker = RecursiveCharacterChunker(
    chunk_size=256,         # max tokens per chunk
    chunk_overlap=20,       # overlap between chunks
    separators=["\n\n", "\n", ". ", " ", ""],
)

# Chunking happens automatically during kb.upsert()
# But you can also chunk manually:
from canopy.models.data_models import Document

doc = Document(
    id="test",
    text="Long document text...",
    source="test",
)

chunks = chunker.chunk_documents([doc])
for chunk in chunks:
    print(f"Chunk ID: {chunk.id}")
    print(f"Text: {chunk.text[:100]}...")
    print(f"Tokens: ~{len(chunk.text.split())}")
```

### MarkdownChunker

Preserves markdown structure (headings, code blocks, lists):

```python
from canopy.knowledge_base.chunker import MarkdownChunker

md_chunker = MarkdownChunker(
    chunk_size=512,
    chunk_overlap=50,
)

# Splits on markdown headings first, then by paragraphs
```

### Custom Chunker

```python
from canopy.knowledge_base.chunker import Chunker
from canopy.models.data_models import Document
from canopy.knowledge_base.models import KBDocChunk


class SentenceChunker(Chunker):
    """Split documents by sentences with a fixed sentence count per chunk."""

    def __init__(self, sentences_per_chunk: int = 5, overlap_sentences: int = 1):
        self.sentences_per_chunk = sentences_per_chunk
        self.overlap_sentences = overlap_sentences

    def chunk_documents(self, documents: list[Document]) -> list[KBDocChunk]:
        chunks = []
        for doc in documents:
            import re
            sentences = re.split(r'(?<=[.!?])\s+', doc.text)

            for i in range(0, len(sentences), self.sentences_per_chunk - self.overlap_sentences):
                chunk_sentences = sentences[i : i + self.sentences_per_chunk]
                if not chunk_sentences:
                    continue

                chunk_text = " ".join(chunk_sentences)
                chunk_id = f"{doc.id}_{i}"

                chunks.append(KBDocChunk(
                    id=chunk_id,
                    document_id=doc.id,
                    text=chunk_text,
                    source=doc.source,
                    metadata=doc.metadata,
                ))

        return chunks
```

---

## Embedding

Canopy uses OpenAI embeddings by default but supports custom embedding models:

```python
# Default: OpenAI text-embedding-3-small
# Configured via environment variables:
# CANOPY_EMBEDDING_MODEL=text-embedding-3-small

# Or programmatically:
from canopy.knowledge_base.record_encoder import OpenAIRecordEncoder

encoder = OpenAIRecordEncoder(
    model_name="text-embedding-3-small",
    batch_size=100,
)

# Custom encoder for local models
from canopy.knowledge_base.record_encoder import RecordEncoder
from canopy.knowledge_base.models import KBEncodedDocChunk, KBDocChunk
from sentence_transformers import SentenceTransformer


class LocalRecordEncoder(RecordEncoder):
    """Encode chunks using a local SentenceTransformer model."""

    def __init__(self, model_name: str = "BAAI/bge-small-en-v1.5"):
        self.model = SentenceTransformer(model_name)
        self.dimension = self.model.get_sentence_embedding_dimension()

    def encode_documents(self, chunks: list[KBDocChunk]) -> list[KBEncodedDocChunk]:
        texts = [chunk.text for chunk in chunks]
        embeddings = self.model.encode(texts, normalize_embeddings=True)

        encoded = []
        for chunk, embedding in zip(chunks, embeddings):
            encoded.append(KBEncodedDocChunk(
                id=chunk.id,
                document_id=chunk.document_id,
                text=chunk.text,
                source=chunk.source,
                metadata=chunk.metadata,
                values=embedding.tolist(),
            ))
        return encoded

    def encode_queries(self, queries: list[str]) -> list[list[float]]:
        embeddings = self.model.encode(queries, normalize_embeddings=True)
        return embeddings.tolist()

    @property
    def dimension(self) -> int:
        return self._dimension

    @dimension.setter
    def dimension(self, value):
        self._dimension = value
```

---

## LLM Configuration

### OpenAI (Default)

```python
# Configured via environment:
# OPENAI_API_KEY=sk-...
# CANOPY_LLM_MODEL=gpt-4o-mini

# Or programmatically:
from canopy.llm import OpenAILLM

llm = OpenAILLM(
    model_name="gpt-4o-mini",
    temperature=0.0,
    max_tokens=500,
)
```

### Anthropic

```python
from canopy.llm import AnthropicLLM

llm = AnthropicLLM(
    model_name="claude-sonnet-4-20250514",
    temperature=0.0,
    max_tokens=500,
)
```

### Any Model via LiteLLM

```python
# Canopy supports any model through LiteLLM
# Set the model name with provider prefix:
# CANOPY_LLM_MODEL=anthropic/claude-sonnet-4-20250514
# CANOPY_LLM_MODEL=ollama/llama3.1:8b
# CANOPY_LLM_MODEL=azure/gpt-4o
```

---

## Context Building

The ContextBuilder manages the token budget for retrieved context:

```python
from canopy.context_engine.context_builder import StuffingContextBuilder

builder = StuffingContextBuilder(
    max_context_tokens=3000,    # total token budget for context
)

# The builder:
# 1. Takes retrieved chunks sorted by relevance
# 2. Adds chunks until the token budget is exhausted
# 3. Formats the context with source attribution
# 4. Returns a ContextContent object with the formatted text
```

### Token Budget Management

```python
# The ChatEngine balances tokens between context and generation:
#
# Total model context window (e.g., 128K for GPT-4o)
#  - System prompt tokens
#  - Conversation history tokens
#  - Max output tokens
#  = Available context tokens
#
# Example with GPT-4o-mini (128K context):
#   128,000 - 500 (system) - 2,000 (history) - 1,000 (output) = 124,500 context tokens
#
# Canopy automatically computes this and fills the context budget
```

---

## Server Mode

Canopy runs as an OpenAI-compatible API server:

```bash
# Start the server
canopy start --config my_config.yaml

# The server exposes:
# POST /v1/chat/completions  (OpenAI-compatible)
# POST /v1/context/query     (context-only, no generation)
# POST /v1/context/upsert    (upload documents)
# DELETE /v1/context/delete   (delete documents)
# GET /v1/health              (health check)
```

### Using with the OpenAI SDK

```python
from openai import OpenAI

# Point the OpenAI client at Canopy
client = OpenAI(
    base_url="http://localhost:8000/v1",
    api_key="not-needed",  # Canopy handles auth to Pinecone and OpenAI
)

# Use exactly like OpenAI, but with RAG context
response = client.chat.completions.create(
    model="gpt-4o-mini",  # Canopy will use whatever model is configured
    messages=[
        {"role": "user", "content": "What is Pinecone's serverless architecture?"},
    ],
    stream=True,
)

for chunk in response:
    if chunk.choices[0].delta.content:
        print(chunk.choices[0].delta.content, end="")
```

---

## Configuration

### YAML Configuration

```yaml
# canopy_config.yaml
knowledge_base:
  index_name: my-canopy-index
  default_top_k: 5
  chunker:
    type: RecursiveCharacterChunker
    params:
      chunk_size: 512
      chunk_overlap: 50
  record_encoder:
    type: OpenAIRecordEncoder
    params:
      model_name: text-embedding-3-small
      batch_size: 100

context_engine:
  max_context_tokens: 4000

llm:
  type: OpenAILLM
  params:
    model_name: gpt-4o-mini
    temperature: 0.0
    max_tokens: 1000

chat_engine:
  system_prompt: |
    You are a helpful assistant that answers questions based on the provided context.
    If you cannot find the answer in the context, say "I don't have enough information."
    Always cite the source when possible.
```

### Environment Variables

```bash
# Required
export PINECONE_API_KEY="your-pinecone-api-key"
export OPENAI_API_KEY="sk-..."

# Canopy configuration
export CANOPY_INDEX_NAME="my-canopy-index"
export CANOPY_LLM_MODEL="gpt-4o-mini"
export CANOPY_EMBEDDING_MODEL="text-embedding-3-small"
export CANOPY_DEFAULT_TOP_K="5"
export CANOPY_MAX_CONTEXT_TOKENS="4000"

# Optional: for Anthropic
export ANTHROPIC_API_KEY="sk-ant-..."
```

---

## When to Use Canopy

| Scenario | Canopy | Custom Pinecone + LangChain |
|----------|--------|----------------------------|
| Already using Pinecone | Best fit | Also works |
| Need OpenAI-compatible API | Built-in | Must build |
| Want quick deployment | Minutes | Hours |
| Need custom retrieval logic | Limited | Full flexibility |
| Multi-model support | Via LiteLLM | Direct integration |
| Production monitoring | Basic | Build your own |
| Knowledge graph | Not supported | Use R2R or custom |
| Multi-tenant isolation | Via Pinecone namespaces | Via Pinecone namespaces |

---

## Common Pitfalls

1. **Not creating the Pinecone index first**: `kb.create_canopy_index()` must be called before upserting documents. Without it, Canopy fails with a "namespace not found" error.

2. **Mixing embedding dimensions**: If you change the embedding model after initial indexing, old vectors become incompatible. Delete the index and re-index everything.

3. **Exceeding Pinecone's metadata size limit**: Pinecone limits metadata to 40KB per vector. Canopy stores chunk text in metadata by default. Very long chunks may exceed this limit.

4. **Not setting PINECONE_API_KEY**: Both environment variables (`PINECONE_API_KEY` and `OPENAI_API_KEY`) must be set before starting Canopy. Missing keys cause cryptic initialization errors.

5. **Using the wrong index spec**: Pinecone serverless and pod-based indexes have different specs. Ensure `create_canopy_index()` uses the correct spec for your Pinecone plan.

6. **Ignoring token budget**: If `max_context_tokens` is too small, retrieved chunks are truncated. If too large, it leaves insufficient room for generation. Tune based on your model's context window.

---

## References

- Canopy GitHub: https://github.com/pinecone-io/canopy
- Pinecone documentation: https://docs.pinecone.io/
- Canopy quickstart: https://docs.pinecone.io/guides/gen-ai/canopy
- Pinecone blog: https://www.pinecone.io/blog/canopy-rag-framework/
