# LlamaIndex IngestionPipeline Deep Dive

## Overview

The IngestionPipeline is LlamaIndex's production-grade data processing pipeline. It applies a sequence of transformations (splitting, metadata extraction, embedding) to documents, with built-in support for deduplication via docstore, caching to skip unchanged documents, parallel processing, and progress monitoring. For any workload beyond a quick prototype, IngestionPipeline replaces the simpler `VectorStoreIndex.from_documents()` approach.

---

## Basic Pipeline

```python
from llama_index.core.ingestion import IngestionPipeline
from llama_index.core.node_parser import SentenceSplitter
from llama_index.embeddings.openai import OpenAIEmbedding

pipeline = IngestionPipeline(
    transformations=[
        SentenceSplitter(chunk_size=512, chunk_overlap=50),
        OpenAIEmbedding(model="text-embedding-3-small"),
    ],
)

nodes = pipeline.run(documents=documents, show_progress=True)
```

The pipeline applies transformations in order:
1. **SentenceSplitter**: Document -> Nodes (chunks)
2. **OpenAIEmbedding**: Node -> Node with embedding attached

The output is a list of Nodes ready to be inserted into a VectorStoreIndex or vector store.

---

## Transformations

Transformations are the building blocks of the pipeline. Each transformation takes a list of nodes and returns a list of nodes.

### Node Parsers (Splitters)

```python
from llama_index.core.node_parser import (
    SentenceSplitter,           # Split by sentences, respecting chunk_size
    TokenTextSplitter,          # Split by token count
    SentenceWindowNodeParser,   # Each node = 1 sentence, with surrounding window as metadata
    SemanticSplitterNodeParser, # Split when semantic similarity drops
    MarkdownNodeParser,         # Split by Markdown headings
    HTMLNodeParser,             # Split by HTML tags
    CodeSplitter,               # Split code by functions/classes
)

# Most common: SentenceSplitter
splitter = SentenceSplitter(
    chunk_size=512,          # max tokens per chunk
    chunk_overlap=50,        # tokens shared between consecutive chunks
    paragraph_separator="\n\n",  # prefer splitting on paragraph boundaries
    separator=" ",           # fallback separator
)

# For RAG with sentence-level retrieval + surrounding context
window_parser = SentenceWindowNodeParser.from_defaults(
    window_size=3,           # 3 sentences before and after
    window_metadata_key="window",
    original_text_metadata_key="original_text",
)

# For code
code_splitter = CodeSplitter(
    language="python",
    chunk_lines=40,          # max lines per chunk
    chunk_lines_overlap=5,   # lines shared between chunks
    max_chars=1500,
)
```

### Metadata Extractors

Automatically extract metadata from text content using an LLM:

```python
from llama_index.core.extractors import (
    TitleExtractor,
    SummaryExtractor,
    QuestionsAnsweredExtractor,
    KeywordExtractor,
    EntityExtractor,
)
from llama_index.llms.openai import OpenAI

llm = OpenAI(model="gpt-4o-mini", temperature=0)

# Title extraction: generates a title for each chunk
title_extractor = TitleExtractor(
    llm=llm,
    nodes=5,              # number of nodes to consider for title
)

# Summary extraction: generates a 1-2 sentence summary
summary_extractor = SummaryExtractor(
    llm=llm,
    summaries=["self"],   # summarize the chunk itself
)

# Questions extraction: generates questions this chunk can answer
qa_extractor = QuestionsAnsweredExtractor(
    llm=llm,
    questions=3,          # generate 3 questions per chunk
)

# Keyword extraction
keyword_extractor = KeywordExtractor(
    llm=llm,
    keywords=5,           # extract 5 keywords per chunk
)
```

### Embedding Models

```python
from llama_index.embeddings.openai import OpenAIEmbedding
from llama_index.embeddings.huggingface import HuggingFaceEmbedding

# OpenAI (cloud)
embed_model = OpenAIEmbedding(
    model="text-embedding-3-small",  # 1536 dims, best cost/quality
    dimensions=1024,                  # Matryoshka: reduce to save storage
)

# HuggingFace (local)
embed_model = HuggingFaceEmbedding(
    model_name="BAAI/bge-small-en-v1.5",
    device="cuda",
)
```

---

## Full Pipeline with Metadata Extraction

```python
from llama_index.core.ingestion import IngestionPipeline
from llama_index.core.node_parser import SentenceSplitter
from llama_index.core.extractors import (
    TitleExtractor,
    KeywordExtractor,
    QuestionsAnsweredExtractor,
)
from llama_index.embeddings.openai import OpenAIEmbedding
from llama_index.llms.openai import OpenAI

llm = OpenAI(model="gpt-4o-mini", temperature=0)

pipeline = IngestionPipeline(
    transformations=[
        # Step 1: Split into chunks
        SentenceSplitter(chunk_size=512, chunk_overlap=50),

        # Step 2: Extract metadata (LLM calls)
        TitleExtractor(llm=llm),
        KeywordExtractor(llm=llm, keywords=5),
        QuestionsAnsweredExtractor(llm=llm, questions=3),

        # Step 3: Embed (must be last)
        OpenAIEmbedding(model="text-embedding-3-small"),
    ],
)

nodes = pipeline.run(documents=documents, show_progress=True)

# Each node now has:
# - node.text: the chunk text
# - node.embedding: the vector
# - node.metadata["document_title"]: extracted title
# - node.metadata["excerpt_keywords"]: extracted keywords
# - node.metadata["questions_this_excerpt_can_answer"]: generated questions
```

---

## Docstore for Deduplication

The docstore tracks which documents have been processed. When you re-run the pipeline with updated documents, only new or changed documents are processed:

```python
from llama_index.core.ingestion import IngestionPipeline, IngestionCache
from llama_index.core.storage.docstore import SimpleDocumentStore

# Create pipeline with docstore
docstore = SimpleDocumentStore()

pipeline = IngestionPipeline(
    transformations=[
        SentenceSplitter(chunk_size=512),
        OpenAIEmbedding(model="text-embedding-3-small"),
    ],
    docstore=docstore,
    docstore_strategy="upserts",  # "upserts" | "upserts_and_delete" | "duplicates_only"
)

# First run: processes all documents
nodes = pipeline.run(documents=documents)
print(f"Processed {len(nodes)} nodes")

# Second run with same documents: skips already-processed
nodes = pipeline.run(documents=documents)
print(f"Processed {len(nodes)} nodes")  # 0 (all already in docstore)

# Third run with new documents added
documents.append(Document(text="New content..."))
nodes = pipeline.run(documents=documents)
print(f"Processed {len(nodes)} nodes")  # only nodes from new document
```

### Docstore Strategies

| Strategy | Behavior |
|---|---|
| `upserts` | Skip docs already in docstore. Update if content changed (by doc_id). |
| `upserts_and_delete` | Same as upserts, plus remove docs no longer in the input list. |
| `duplicates_only` | Skip docs with identical content hash. Does not track by doc_id. |

### Persistent Docstore

```python
from llama_index.storage.docstore.redis import RedisDocumentStore

# Redis-backed docstore (persists across pipeline runs)
docstore = RedisDocumentStore.from_host_and_port(
    host="localhost",
    port=6379,
    namespace="my_pipeline",
)

pipeline = IngestionPipeline(
    transformations=[...],
    docstore=docstore,
    docstore_strategy="upserts_and_delete",
)
```

---

## Cache for Skip-Unchanged

The IngestionCache stores the output of each transformation step. If a node's input has not changed, the cached output is reused:

```python
from llama_index.core.ingestion import IngestionPipeline, IngestionCache
from llama_index.core.storage.kvstore import SimpleKVStore

# In-memory cache
cache = IngestionCache(cache=SimpleKVStore())

pipeline = IngestionPipeline(
    transformations=[
        SentenceSplitter(chunk_size=512),
        KeywordExtractor(llm=llm),      # expensive LLM call
        OpenAIEmbedding(),               # expensive API call
    ],
    cache=cache,
)

# First run: all transformations run
nodes = pipeline.run(documents=documents)

# Second run: if documents unchanged, outputs are served from cache
# Saves LLM and embedding API costs
nodes = pipeline.run(documents=documents)
```

### Redis-Backed Cache

```python
from llama_index.storage.kvstore.redis import RedisKVStore
from llama_index.core.ingestion import IngestionCache

cache = IngestionCache(
    cache=RedisKVStore.from_host_and_port(host="localhost", port=6379)
)

pipeline = IngestionPipeline(
    transformations=[...],
    cache=cache,
)
```

---

## Custom Transformations

Create your own transformation by subclassing `TransformComponent`:

```python
from llama_index.core.schema import TransformComponent, BaseNode
from typing import Sequence

class MetadataCleanerTransform(TransformComponent):
    """Remove PII and normalize metadata."""

    def __call__(self, nodes: list[BaseNode], **kwargs) -> list[BaseNode]:
        for node in nodes:
            # Remove email addresses from text
            import re
            node.text = re.sub(r'\b[\w.+-]+@[\w-]+\.[\w.-]+\b', '[EMAIL]', node.text)

            # Normalize metadata keys
            if "Author" in node.metadata:
                node.metadata["author"] = node.metadata.pop("Author")

            # Add processing timestamp
            from datetime import datetime
            node.metadata["processed_at"] = datetime.now().isoformat()

        return nodes

class ChunkFilterTransform(TransformComponent):
    """Filter out very short or very long chunks."""

    def __init__(self, min_length: int = 50, max_length: int = 5000):
        self.min_length = min_length
        self.max_length = max_length

    def __call__(self, nodes: list[BaseNode], **kwargs) -> list[BaseNode]:
        return [
            node for node in nodes
            if self.min_length <= len(node.text) <= self.max_length
        ]


# Use in pipeline
pipeline = IngestionPipeline(
    transformations=[
        SentenceSplitter(chunk_size=512),
        MetadataCleanerTransform(),
        ChunkFilterTransform(min_length=100),
        OpenAIEmbedding(),
    ],
)
```

---

## Parallel Processing

### Parallel Embedding

For large corpora, parallelize the embedding step:

```python
from llama_index.core.ingestion import IngestionPipeline
from llama_index.embeddings.openai import OpenAIEmbedding

pipeline = IngestionPipeline(
    transformations=[
        SentenceSplitter(chunk_size=512),
        OpenAIEmbedding(
            model="text-embedding-3-small",
            num_workers=8,               # parallel API calls
        ),
    ],
)

# Process in batches
nodes = pipeline.run(
    documents=documents,
    show_progress=True,
    num_workers=4,                       # parallel transformation workers
)
```

### Batch Processing for Large Datasets

```python
import math

def process_in_batches(documents, pipeline, batch_size=500):
    """Process documents in batches to manage memory."""
    all_nodes = []
    num_batches = math.ceil(len(documents) / batch_size)

    for i in range(num_batches):
        batch = documents[i * batch_size : (i + 1) * batch_size]
        print(f"Processing batch {i+1}/{num_batches} ({len(batch)} documents)")

        nodes = pipeline.run(documents=batch, show_progress=True)
        all_nodes.extend(nodes)

        print(f"  Generated {len(nodes)} nodes (total: {len(all_nodes)})")

    return all_nodes

# Usage
all_nodes = process_in_batches(documents, pipeline, batch_size=500)
```

### Async Processing

```python
import asyncio

async def process_async(documents, pipeline):
    """Process documents asynchronously."""
    nodes = await pipeline.arun(documents=documents, show_progress=True)
    return nodes

# Usage
nodes = asyncio.run(process_async(documents, pipeline))
```

---

## Monitoring Progress

### Built-in Progress

```python
# show_progress=True enables tqdm progress bars
nodes = pipeline.run(documents=documents, show_progress=True)
```

### Custom Callback Handler

```python
from llama_index.core.callbacks import CallbackManager, LlamaDebugHandler

# Debug handler logs all events
debug_handler = LlamaDebugHandler(print_trace_on_end=True)
callback_manager = CallbackManager([debug_handler])

pipeline = IngestionPipeline(
    transformations=[
        SentenceSplitter(chunk_size=512),
        OpenAIEmbedding(),
    ],
)

# Attach callback manager to Settings
from llama_index.core import Settings
Settings.callback_manager = callback_manager

nodes = pipeline.run(documents=documents)

# Print event trace
debug_handler.flush_event_logs()
```

### Token Usage Tracking

```python
from llama_index.core.callbacks import TokenCountingHandler
import tiktoken

token_counter = TokenCountingHandler(
    tokenizer=tiktoken.encoding_for_model("gpt-4o-mini").encode,
)

callback_manager = CallbackManager([token_counter])
Settings.callback_manager = callback_manager

# After pipeline run:
print(f"Embedding tokens: {token_counter.total_embedding_token_count}")
print(f"LLM prompt tokens: {token_counter.prompt_llm_token_count}")
print(f"LLM completion tokens: {token_counter.completion_llm_token_count}")
print(f"Estimated cost: ${token_counter.total_embedding_token_count * 0.00002:.4f}")
```

---

## Direct Vector Store Insertion

For production, pipe nodes directly into a vector store without building an in-memory index:

```python
from llama_index.core.ingestion import IngestionPipeline
from llama_index.vector_stores.qdrant import QdrantVectorStore
from qdrant_client import QdrantClient

# Connect to Qdrant
client = QdrantClient(url="http://localhost:6333")
vector_store = QdrantVectorStore(client=client, collection_name="documents")

# Pipeline that writes directly to vector store
pipeline = IngestionPipeline(
    transformations=[
        SentenceSplitter(chunk_size=512),
        OpenAIEmbedding(model="text-embedding-3-small"),
    ],
    vector_store=vector_store,
)

# Nodes are automatically inserted into Qdrant after embedding
pipeline.run(documents=documents, show_progress=True)

# No need to manually insert -- the pipeline handles it
```

### With pgvector

```python
from llama_index.vector_stores.postgres import PGVectorStore

vector_store = PGVectorStore.from_params(
    database="vectordb",
    host="localhost",
    password="secret",
    port=5432,
    user="app",
    table_name="documents",
    embed_dim=1536,
)

pipeline = IngestionPipeline(
    transformations=[
        SentenceSplitter(chunk_size=512),
        OpenAIEmbedding(model="text-embedding-3-small"),
    ],
    vector_store=vector_store,
    docstore=SimpleDocumentStore(),        # deduplication
    docstore_strategy="upserts_and_delete",
)

pipeline.run(documents=documents, show_progress=True)
```

---

## Production Pipeline Pattern

```python
from llama_index.core.ingestion import IngestionPipeline, IngestionCache
from llama_index.core.node_parser import SentenceSplitter
from llama_index.core.extractors import KeywordExtractor
from llama_index.core.storage.docstore import SimpleDocumentStore
from llama_index.embeddings.openai import OpenAIEmbedding
from llama_index.vector_stores.postgres import PGVectorStore
from llama_index.llms.openai import OpenAI
import logging

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

def create_production_pipeline(vector_store, docstore=None, cache=None):
    """Create a production ingestion pipeline with all features."""

    return IngestionPipeline(
        transformations=[
            # 1. Chunk documents
            SentenceSplitter(
                chunk_size=512,
                chunk_overlap=50,
                paragraph_separator="\n\n",
            ),

            # 2. Extract searchable keywords (improves hybrid search)
            KeywordExtractor(
                llm=OpenAI(model="gpt-4o-mini", temperature=0),
                keywords=5,
            ),

            # 3. Embed (must be last)
            OpenAIEmbedding(
                model="text-embedding-3-small",
                num_workers=4,
            ),
        ],
        vector_store=vector_store,
        docstore=docstore,
        docstore_strategy="upserts_and_delete",
        cache=cache,
    )


# Initialize
vector_store = PGVectorStore.from_params(
    database="vectordb", host="localhost",
    password="secret", port=5432, user="app",
    table_name="documents", embed_dim=1536,
)

docstore = SimpleDocumentStore()

pipeline = create_production_pipeline(vector_store, docstore)

# Run
from llama_index.core import SimpleDirectoryReader

documents = SimpleDirectoryReader("./data", recursive=True).load_data()
logger.info(f"Loaded {len(documents)} documents")

pipeline.run(documents=documents, show_progress=True)
logger.info("Ingestion complete")
```

---

## Common Pitfalls

1. **Putting embedding before metadata extraction**: metadata extractors may use the text to generate metadata (keywords, summaries). If you embed first, the metadata will not be included in the embedding. Always put embedding last.

2. **Not using a docstore for incremental updates**: without a docstore, every pipeline run reprocesses all documents. For a corpus that grows incrementally, this wastes time and API costs.

3. **Setting chunk_overlap too high**: overlap > 25% of chunk_size creates nearly-duplicate nodes that waste storage and confuse retrieval. Keep overlap at 5-15% of chunk_size.

4. **Using metadata extractors on all chunks**: metadata extraction requires an LLM call per chunk. For 10K chunks at $0.001/call, that is $10 per run. Use extractors selectively or only for important documents.

5. **Not persisting cache between runs**: the in-memory cache is lost when the process exits. Use Redis-backed cache for persistent caching across runs.

6. **Ignoring token limits**: metadata extractors concatenate the chunk text with a prompt. If your chunks are large and the extractor prompt is long, you may exceed the LLM's context window.

---

## References

- LlamaIndex IngestionPipeline: https://docs.llamaindex.ai/en/stable/module_guides/loading/ingestion_pipeline/
- LlamaIndex Transformations: https://docs.llamaindex.ai/en/stable/module_guides/loading/ingestion_pipeline/transformations/
- LlamaIndex Node Parsers: https://docs.llamaindex.ai/en/stable/module_guides/loading/node_parsers/
