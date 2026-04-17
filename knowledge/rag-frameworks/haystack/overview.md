# Haystack 2.x -- Pipeline DAG Architecture

## Overview

Haystack is an open-source framework by deepset for building production-grade LLM applications. Haystack 2.x (released late 2023, stabilized in 2024-2025) is a complete rewrite that replaces the linear pipeline model of Haystack 1.x with a directed acyclic graph (DAG) architecture. Every component in a Haystack 2.x pipeline is a node in a graph, connected by typed input/output sockets. This enables branching, merging, and conditional routing that was impossible in the linear model.

This guide covers the core architecture, the component protocol, pipeline construction, and the design patterns that make Haystack 2.x effective for RAG systems.

---

## Core Concepts

### Components

A Component is any Python class that implements the Haystack component protocol. The protocol requires two things:

1. A `run()` method decorated with `@component.output_types()` that declares output types
2. Input parameters declared via the `run()` method's type hints

```python
from haystack import component, Document
from typing import List, Optional


@component
class TextCleaner:
    """Cleans and normalizes text in documents."""

    def __init__(self, lowercase: bool = True, strip_whitespace: bool = True):
        self.lowercase = lowercase
        self.strip_whitespace = strip_whitespace

    @component.output_types(documents=List[Document])
    def run(self, documents: List[Document]) -> dict:
        cleaned = []
        for doc in documents:
            text = doc.content
            if self.strip_whitespace:
                text = " ".join(text.split())
            if self.lowercase:
                text = text.lower()
            cleaned.append(Document(content=text, meta=doc.meta))
        return {"documents": cleaned}
```

### The Component Protocol in Detail

```python
from haystack import component


@component
class MyComponent:
    """
    The @component decorator does three things:
    1. Registers the class as a Haystack component
    2. Enables serialization/deserialization
    3. Enables pipeline connection validation
    """

    def __init__(self, param: str = "default"):
        # __init__ parameters become serializable config
        self.param = param

    @component.output_types(
        result=str,           # named output "result" of type str
        score=float,          # named output "score" of type float
    )
    def run(
        self,
        text: str,            # required input "text"
        threshold: float = 0.5,  # optional input "threshold" with default
    ) -> dict:
        # run() MUST return a dict with keys matching output_types
        processed = text.upper() if self.param == "upper" else text
        return {
            "result": processed,
            "score": 0.95,
        }
```

### Pipeline

A Pipeline is a DAG of connected components. You add components, connect their outputs to inputs, and run the pipeline:

```python
from haystack import Pipeline
from haystack.components.builders import PromptBuilder
from haystack.components.generators import OpenAIGenerator
from haystack.components.retrievers.in_memory import InMemoryBM25Retriever
from haystack.document_stores.in_memory import InMemoryDocumentStore

# Create document store and populate it
store = InMemoryDocumentStore()
store.write_documents([
    Document(content="Haystack 2.x uses a DAG pipeline architecture."),
    Document(content="Components in Haystack are connected via typed sockets."),
    Document(content="Haystack supports OpenAI, Anthropic, and local models."),
])

# Create pipeline
pipe = Pipeline()

# Add components
pipe.add_component("retriever", InMemoryBM25Retriever(document_store=store))
pipe.add_component("prompt", PromptBuilder(
    template="""
    Given these documents:
    {% for doc in documents %}
    - {{ doc.content }}
    {% endfor %}

    Answer: {{ question }}
    """
))
pipe.add_component("llm", OpenAIGenerator(model="gpt-4o-mini"))

# Connect components (output_name -> input_name)
pipe.connect("retriever.documents", "prompt.documents")
pipe.connect("prompt.prompt", "llm.prompt")

# Run the pipeline
result = pipe.run({
    "retriever": {"query": "What architecture does Haystack use?"},
    "prompt": {"question": "What architecture does Haystack use?"},
})

print(result["llm"]["replies"][0])
```

---

## Pipeline DAG Architecture

### How the DAG Works

Unlike linear pipelines where data flows A -> B -> C, Haystack 2.x pipelines form a graph:

```
                    +-------------+
                    |  Retriever  |
         +-------->|  (BM25)     |--------+
         |         +-------------+        |
         |                                |
+--------+---+                     +------v------+
|   Query    |                     |   Prompt    |
|   Input    |-------------------->|   Builder   |
+--------+---+                     +------+------+
         |                                |
         |         +-------------+  +-----v------+
         +-------->|  Retriever  |  |    LLM     |
                   |  (Embed)    |--+  Generator  |
                   +-------------+  +-----+------+
                                          |
                                    +-----v------+
                                    |   Output   |
                                    +------------+
```

### Connection Rules

1. **Type safety**: Output types must match input types. Connecting `str` output to `List[Document]` input raises a `PipelineConnectError`.
2. **Fan-out**: One output can connect to multiple inputs (broadcasting).
3. **Fan-in**: Multiple outputs can connect to the same input if the input type is compatible (e.g., `List[Document]` can receive from multiple retrievers via a `DocumentJoiner`).
4. **No cycles**: The graph must be acyclic. Cycles raise a `PipelineValidationError`.

```python
from haystack import Pipeline
from haystack.components.joiners import DocumentJoiner
from haystack.components.retrievers.in_memory import (
    InMemoryBM25Retriever,
    InMemoryEmbeddingRetriever,
)
from haystack.components.embedders import SentenceTransformersTextEmbedder

pipe = Pipeline()

# Two retrievers (hybrid search)
pipe.add_component("text_embedder", SentenceTransformersTextEmbedder(
    model="BAAI/bge-small-en-v1.5"
))
pipe.add_component("bm25_retriever", InMemoryBM25Retriever(document_store=store, top_k=5))
pipe.add_component("embed_retriever", InMemoryEmbeddingRetriever(document_store=store, top_k=5))

# Joiner merges results from both retrievers
pipe.add_component("joiner", DocumentJoiner(join_mode="reciprocal_rank_fusion"))

# Connect both retrievers to the joiner (fan-in)
pipe.connect("text_embedder.embedding", "embed_retriever.query_embedding")
pipe.connect("bm25_retriever.documents", "joiner.documents")
pipe.connect("embed_retriever.documents", "joiner.documents")
```

### Pipeline Serialization

Haystack pipelines serialize to YAML for version control and deployment:

```python
# Save pipeline to YAML
pipe.dump(open("pipeline.yaml", "w"))

# Load pipeline from YAML
from haystack import Pipeline
loaded_pipe = Pipeline.load(open("pipeline.yaml"))

# Run the loaded pipeline
result = loaded_pipe.run({"retriever": {"query": "test"}})
```

The YAML format:

```yaml
components:
  retriever:
    type: haystack.components.retrievers.in_memory.InMemoryBM25Retriever
    init_parameters:
      document_store:
        type: haystack.document_stores.in_memory.InMemoryDocumentStore
      top_k: 5
  prompt:
    type: haystack.components.builders.PromptBuilder
    init_parameters:
      template: "Given: {{ documents }}\nAnswer: {{ question }}"
  llm:
    type: haystack.components.generators.OpenAIGenerator
    init_parameters:
      model: gpt-4o-mini
connections:
  - sender: retriever.documents
    receiver: prompt.documents
  - sender: prompt.prompt
    receiver: llm.prompt
```

---

## Document Stores

Document stores are the persistence layer for Haystack. They store documents and provide retrieval interfaces:

```python
from haystack.document_stores.in_memory import InMemoryDocumentStore
from haystack import Document

# In-memory (for development/testing)
store = InMemoryDocumentStore(
    bm25_tokenization_regex=r"(?u)\b\w\w+\b",
    bm25_algorithm="BM25Plus",
    embedding_similarity_function="cosine",
)

# Write documents
documents = [
    Document(
        content="Haystack 2.x uses a DAG pipeline architecture.",
        meta={"source": "docs", "section": "architecture"},
    ),
    Document(
        content="Components communicate through typed input/output sockets.",
        meta={"source": "docs", "section": "components"},
    ),
]
store.write_documents(documents, policy="overwrite")  # or "skip", "error"
print(f"Documents in store: {store.count_documents()}")
```

### Production Document Stores

```python
# Elasticsearch
# pip install elasticsearch-haystack
from haystack_integrations.document_stores.elasticsearch import ElasticsearchDocumentStore

es_store = ElasticsearchDocumentStore(
    hosts="http://localhost:9200",
    index="my_documents",
)

# Qdrant
# pip install qdrant-haystack
from haystack_integrations.document_stores.qdrant import QdrantDocumentStore

qdrant_store = QdrantDocumentStore(
    url="http://localhost:6333",
    index="my_documents",
    embedding_dim=384,
    recreate_index=False,
    hnsw_config={"m": 16, "ef_construct": 128},
)

# pgvector
# pip install pgvector-haystack
from haystack_integrations.document_stores.pgvector import PgvectorDocumentStore

pg_store = PgvectorDocumentStore(
    connection_string="postgresql://user:pass@localhost:5432/vectordb",
    table_name="documents",
    embedding_dimension=384,
    vector_function="cosine_similarity",
    recreate_table=False,
    hnsw_recreate=False,
    hnsw_index_creation_kwargs={"m": 16, "ef_construction": 128},
)

# Pinecone
# pip install pinecone-haystack
from haystack_integrations.document_stores.pinecone import PineconeDocumentStore

pinecone_store = PineconeDocumentStore(
    api_key="your-key",
    index="my-index",
    namespace="production",
    dimension=384,
    metric="cosine",
)
```

---

## Embedders

Haystack separates document embedding (indexing time) from text embedding (query time):

```python
from haystack.components.embedders import (
    SentenceTransformersDocumentEmbedder,
    SentenceTransformersTextEmbedder,
)

# Document embedder (for indexing)
doc_embedder = SentenceTransformersDocumentEmbedder(
    model="BAAI/bge-small-en-v1.5",
    prefix="",          # some models need "passage: " prefix
    batch_size=64,
    progress_bar=True,
    normalize_embeddings=True,
)
doc_embedder.warm_up()  # load model into memory

# Embed documents
result = doc_embedder.run(documents=documents)
embedded_docs = result["documents"]
# Each document now has doc.embedding (List[float])

# Text embedder (for queries)
text_embedder = SentenceTransformersTextEmbedder(
    model="BAAI/bge-small-en-v1.5",
    prefix="",          # some models need "query: " prefix
)
text_embedder.warm_up()

result = text_embedder.run(text="What is Haystack?")
query_embedding = result["embedding"]  # List[float]
```

### OpenAI Embeddings

```python
from haystack.components.embedders import (
    OpenAIDocumentEmbedder,
    OpenAITextEmbedder,
)

doc_embedder = OpenAIDocumentEmbedder(
    model="text-embedding-3-small",
    dimensions=512,     # optional dimensionality reduction
)

text_embedder = OpenAITextEmbedder(
    model="text-embedding-3-small",
    dimensions=512,
)
```

---

## Generators (LLMs)

Generators wrap LLM API calls:

```python
from haystack.components.generators import OpenAIGenerator

# OpenAI
generator = OpenAIGenerator(
    model="gpt-4o-mini",
    generation_kwargs={
        "temperature": 0.0,
        "max_tokens": 1000,
    },
    system_prompt="You are a helpful assistant that answers questions based on provided context.",
)

result = generator.run(prompt="What is Haystack?")
print(result["replies"][0])   # the generated text
print(result["meta"][0])      # token usage, model info

# Anthropic
# pip install anthropic-haystack
from haystack_integrations.components.generators.anthropic import AnthropicGenerator

anthropic_gen = AnthropicGenerator(
    model="claude-sonnet-4-20250514",
    generation_kwargs={
        "temperature": 0.0,
        "max_tokens": 1000,
    },
)

# Local models via Ollama
# pip install ollama-haystack
from haystack_integrations.components.generators.ollama import OllamaGenerator

ollama_gen = OllamaGenerator(
    model="llama3.1:8b",
    url="http://localhost:11434",
)
```

### Chat Generators

For multi-turn conversations:

```python
from haystack.components.generators.chat import OpenAIChatGenerator
from haystack.dataclasses import ChatMessage

chat_gen = OpenAIChatGenerator(model="gpt-4o-mini")

messages = [
    ChatMessage.from_system("You are a helpful RAG assistant."),
    ChatMessage.from_user("What is Haystack?"),
]

result = chat_gen.run(messages=messages)
reply = result["replies"][0]
print(reply.text)
```

---

## Prompt Building

The `PromptBuilder` uses Jinja2 templates:

```python
from haystack.components.builders import PromptBuilder

builder = PromptBuilder(
    template="""
You are an expert assistant. Answer the question using ONLY the provided context.
If the context does not contain the answer, say "I don't have enough information."

Context:
{% for doc in documents %}
[{{ loop.index }}] {{ doc.content }}
Source: {{ doc.meta.get("source", "unknown") }}
---
{% endfor %}

Question: {{ question }}

Answer:
"""
)

# Run the builder
result = builder.run(
    documents=documents,
    question="What architecture does Haystack use?",
)
print(result["prompt"])  # the formatted prompt string
```

---

## Document Processing

### Splitters

```python
from haystack.components.preprocessors import DocumentSplitter, DocumentCleaner

# Clean documents
cleaner = DocumentCleaner(
    remove_empty_lines=True,
    remove_extra_whitespaces=True,
    remove_regex=r"\[\d+\]",  # remove citation brackets
)

# Split documents
splitter = DocumentSplitter(
    split_by="sentence",         # "word", "sentence", "passage", "page"
    split_length=5,              # 5 sentences per chunk
    split_overlap=1,             # 1 sentence overlap
    split_threshold=3,           # min sentences for a valid chunk
)

# In a pipeline
pipe = Pipeline()
pipe.add_component("cleaner", cleaner)
pipe.add_component("splitter", splitter)
pipe.connect("cleaner.documents", "splitter.documents")

result = pipe.run({"cleaner": {"documents": documents}})
chunks = result["splitter"]["documents"]
```

### File Converters

```python
from haystack.components.converters import (
    TextFileToDocument,
    PyPDFToDocument,
    MarkdownToDocument,
    HTMLToDocument,
)

# PDF to documents
pdf_converter = PyPDFToDocument()
result = pdf_converter.run(sources=["report.pdf"])
docs = result["documents"]

# HTML to documents
html_converter = HTMLToDocument()
result = html_converter.run(
    sources=["page.html"],
    meta={"source": "web"},
)
```

---

## Common Pitfalls

1. **Forgetting `warm_up()` for local models**: SentenceTransformers embedders need `warm_up()` before first use. In a pipeline, this is called automatically, but when using components standalone, you must call it explicitly.

2. **Mismatched embedding dimensions**: If you index with `bge-small-en-v1.5` (384 dims) but query with `text-embedding-3-small` (1536 dims), retrieval fails. Always use the same model for indexing and querying.

3. **Not using `policy="overwrite"` for re-indexing**: By default, `write_documents` raises an error on duplicate IDs. Use `policy="overwrite"` for incremental updates or `policy="skip"` to ignore duplicates.

4. **Connecting wrong socket names**: Socket names must exactly match. `retriever.documents` is different from `retriever.document`. Use `pipe.show()` to visualize the graph and verify connections.

5. **Forgetting to pass all required inputs**: When running a pipeline, you must provide inputs for every component that has unconnected required inputs. Missing inputs cause confusing runtime errors.

6. **Using Haystack 1.x patterns in 2.x**: The `Pipeline.add_node()` API, `BaseComponent`, and `@node` decorator are all gone. Use `@component`, `Pipeline.add_component()`, and `Pipeline.connect()`.

---

## References

- Haystack documentation: https://docs.haystack.deepset.ai/docs/intro
- Haystack GitHub: https://github.com/deepset-ai/haystack
- Haystack integrations: https://haystack.deepset.ai/integrations
- deepset blog: https://www.deepset.ai/blog
