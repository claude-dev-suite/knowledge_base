# Contextual Retrieval: Anthropic's Method Step-by-Step

## Overview / TL;DR

This document provides a complete step-by-step implementation of Anthropic's contextual retrieval technique as published in their September 2024 blog post. It covers the exact prompt, the batching strategy for processing thousands of chunks efficiently, the three-layer stacking approach (contextual embeddings + contextual BM25 + reranking) that achieves a 67% reduction in retrieval failure rate, and production considerations for scaling to large corpora.

---

## Step 1: Chunk the Documents

Before contextualization, chunk your documents using any standard strategy. Anthropic's blog post does not prescribe a specific chunking method; the contextual layer works with any chunking approach.

**Recommended**: Recursive character splitting at 512 tokens with 20% overlap, or structure-aware splitting for markdown.

```python
from langchain_text_splitters import RecursiveCharacterTextSplitter

splitter = RecursiveCharacterTextSplitter(
    chunk_size=512,
    chunk_overlap=100,
    separators=["\n\n", "\n", ". ", " ", ""],
)

def chunk_document(doc_text: str, doc_id: str, source: str) -> list[dict]:
    """Chunk a document and attach metadata."""
    chunks = splitter.split_text(doc_text)
    return [
        {
            "text": chunk,
            "metadata": {
                "doc_id": doc_id,
                "source": source,
                "chunk_index": i,
                "total_chunks": len(chunks),
            },
        }
        for i, chunk in enumerate(chunks)
    ]
```

---

## Step 2: Generate Contextual Preambles

For each chunk, send the full document + the chunk to Claude Haiku and ask for a brief situating context.

### The Exact Prompt

Anthropic published this prompt in their blog post:

```python
CONTEXTUAL_RETRIEVAL_PROMPT = """\
<document>
{WHOLE_DOCUMENT}
</document>
Here is the chunk we want to situate within the whole document:
<chunk>
{CHUNK_CONTENT}
</chunk>
Please give a short succinct context to situate this chunk within \
the overall document for the purposes of improving search retrieval \
of the chunk. Answer only with the succinct context and nothing else."""
```

**What good context looks like**:

Input chunk: "Set the `wal_level` parameter to `replica` and restart the server."

Generated context: "This chunk is from the PostgreSQL 16 Administration Guide, in the section on configuring streaming replication on the primary server. It describes the WAL configuration parameters needed before replication can be established."

**What bad context looks like**:

"This chunk talks about a database parameter." (Too vague -- no useful keywords added.)

"This is from a document about PostgreSQL replication configuration. The document covers primary server setup, replica configuration, monitoring, and failover procedures. The document was last updated in 2024 and covers PostgreSQL versions 14-16." (Too long -- 50+ tokens of generic summary dilutes the embedding.)

### Implementation with Rate Limiting

```python
import anthropic
import asyncio
import time
from dataclasses import dataclass, field


@dataclass
class ContextResult:
    original_text: str
    context: str
    contextualized_text: str
    metadata: dict
    input_tokens: int = 0
    output_tokens: int = 0
    cached_tokens: int = 0


class ContextGenerator:
    """Generate contextual preambles with rate limiting and error handling."""

    def __init__(
        self,
        model: str = "claude-haiku-4-20250514",
        max_concurrent: int = 10,
        max_retries: int = 3,
    ):
        self.client = anthropic.AsyncAnthropic()
        self.model = model
        self.semaphore = asyncio.Semaphore(max_concurrent)
        self.max_retries = max_retries
        self.stats = {"total": 0, "cached": 0, "errors": 0, "tokens_in": 0, "tokens_out": 0}

    async def generate_context(
        self,
        document_text: str,
        chunk_text: str,
        use_caching: bool = True,
    ) -> str:
        """Generate context for a single chunk with retry logic."""
        for attempt in range(self.max_retries):
            try:
                async with self.semaphore:
                    if use_caching:
                        response = await self.client.messages.create(
                            model=self.model,
                            max_tokens=200,
                            temperature=0,
                            system=[{
                                "type": "text",
                                "text": f"<document>\n{document_text}\n</document>",
                                "cache_control": {"type": "ephemeral"},
                            }],
                            messages=[{
                                "role": "user",
                                "content": (
                                    f"Here is the chunk we want to situate within the "
                                    f"whole document:\n<chunk>\n{chunk_text}\n</chunk>\n"
                                    f"Please give a short succinct context to situate "
                                    f"this chunk within the overall document for the "
                                    f"purposes of improving search retrieval of the chunk. "
                                    f"Answer only with the succinct context and nothing else."
                                ),
                            }],
                        )
                    else:
                        response = await self.client.messages.create(
                            model=self.model,
                            max_tokens=200,
                            temperature=0,
                            messages=[{
                                "role": "user",
                                "content": CONTEXTUAL_RETRIEVAL_PROMPT.format(
                                    WHOLE_DOCUMENT=document_text,
                                    CHUNK_CONTENT=chunk_text,
                                ),
                            }],
                        )

                    # Track stats
                    self.stats["total"] += 1
                    self.stats["tokens_in"] += response.usage.input_tokens
                    self.stats["tokens_out"] += response.usage.output_tokens
                    if hasattr(response.usage, "cache_read_input_tokens"):
                        if response.usage.cache_read_input_tokens > 0:
                            self.stats["cached"] += 1

                    return response.content[0].text.strip()

            except anthropic.RateLimitError:
                wait = 2 ** attempt
                await asyncio.sleep(wait)
            except anthropic.APIError as e:
                if attempt == self.max_retries - 1:
                    self.stats["errors"] += 1
                    return ""  # Return empty context on final failure
                await asyncio.sleep(1)

        return ""

    async def process_document(
        self,
        document_text: str,
        chunks: list[dict],
    ) -> list[ContextResult]:
        """Process all chunks from a single document.

        IMPORTANT: Process all chunks from one document before moving to the next.
        This maximizes prompt cache hits (the document is cached on the first call).
        """
        tasks = []
        for chunk in chunks:
            task = self._process_single_chunk(document_text, chunk)
            tasks.append(task)

        return await asyncio.gather(*tasks)

    async def _process_single_chunk(
        self,
        document_text: str,
        chunk: dict,
    ) -> ContextResult:
        context = await self.generate_context(document_text, chunk["text"])
        return ContextResult(
            original_text=chunk["text"],
            context=context,
            contextualized_text=f"{context}\n\n{chunk['text']}",
            metadata=chunk.get("metadata", {}),
        )
```

---

## Step 3: Build Dual Index (Embeddings + BM25)

Index the contextualized text in both a vector store and a BM25 index.

```python
from sentence_transformers import SentenceTransformer
from rank_bm25 import BM25Okapi
import numpy as np
import pickle
from pathlib import Path


class DualIndex:
    """Combined vector + BM25 index for contextual retrieval."""

    def __init__(self, embed_model_name: str = "BAAI/bge-base-en-v1.5"):
        self.embed_model = SentenceTransformer(embed_model_name)
        self.chunks: list[dict] = []
        self.embeddings: np.ndarray = None
        self.bm25: BM25Okapi = None

    def add_chunks(self, contextualized_chunks: list[ContextResult]):
        """Add contextualized chunks to both indexes."""
        new_chunks = [
            {
                "text": c.contextualized_text,
                "original": c.original_text,
                "context": c.context,
                "metadata": c.metadata,
            }
            for c in contextualized_chunks
        ]
        self.chunks.extend(new_chunks)

        # Rebuild indexes (in production, use incremental updates)
        self._rebuild_vector_index()
        self._rebuild_bm25_index()

    def _rebuild_vector_index(self):
        texts = [c["text"] for c in self.chunks]
        self.embeddings = self.embed_model.encode(
            texts,
            normalize_embeddings=True,
            batch_size=64,
            show_progress_bar=True,
        )

    def _rebuild_bm25_index(self):
        tokenized = [c["text"].lower().split() for c in self.chunks]
        self.bm25 = BM25Okapi(tokenized)

    def search(
        self,
        query: str,
        top_k: int = 25,
        bm25_weight: float = 0.3,
    ) -> list[dict]:
        """Hybrid search combining vector + BM25 scores."""
        # Vector search
        query_emb = self.embed_model.encode(query, normalize_embeddings=True)
        vector_scores = np.dot(self.embeddings, query_emb)

        # BM25 search
        bm25_scores = self.bm25.get_scores(query.lower().split())

        # Min-max normalize
        def normalize(arr):
            mn, mx = arr.min(), arr.max()
            return (arr - mn) / (mx - mn + 1e-8)

        v_norm = normalize(vector_scores)
        b_norm = normalize(bm25_scores)

        # Fuse
        fused = (1 - bm25_weight) * v_norm + bm25_weight * b_norm
        top_indices = np.argsort(fused)[::-1][:top_k]

        results = []
        for idx in top_indices:
            chunk = self.chunks[idx].copy()
            chunk["vector_score"] = float(vector_scores[idx])
            chunk["bm25_score"] = float(bm25_scores[idx])
            chunk["fused_score"] = float(fused[idx])
            results.append(chunk)

        return results

    def save(self, path: str):
        """Persist indexes to disk."""
        data = {
            "chunks": self.chunks,
            "embeddings": self.embeddings,
        }
        Path(path).parent.mkdir(parents=True, exist_ok=True)
        with open(path, "wb") as f:
            pickle.dump(data, f)

    def load(self, path: str):
        """Load indexes from disk."""
        with open(path, "rb") as f:
            data = pickle.load(f)
        self.chunks = data["chunks"]
        self.embeddings = data["embeddings"]
        self._rebuild_bm25_index()
```

---

## Step 4: Rerank Retrieved Results

Apply a cross-encoder reranker to the top-25 hybrid results, then select the top-5 for generation.

```python
from sentence_transformers import CrossEncoder


class ContextualReranker:
    def __init__(self, model_name: str = "BAAI/bge-reranker-v2-m3"):
        self.model = CrossEncoder(model_name)

    def rerank(
        self,
        query: str,
        candidates: list[dict],
        top_k: int = 5,
    ) -> list[dict]:
        """Rerank candidates using cross-encoder scoring."""
        # Use the ORIGINAL chunk text for reranking, not the contextualized version.
        # The reranker should evaluate relevance to the query based on actual content.
        pairs = [(query, c["original"]) for c in candidates]
        scores = self.model.predict(pairs)

        for i, c in enumerate(candidates):
            c["rerank_score"] = float(scores[i])

        candidates.sort(key=lambda x: x["rerank_score"], reverse=True)
        return candidates[:top_k]
```

---

## Step 5: Generate Answer

Pass the original chunk text (not contextualized) to the generation model.

```python
import anthropic


def generate_answer(
    question: str,
    top_chunks: list[dict],
    model: str = "claude-sonnet-4-20250514",
) -> dict:
    """Generate answer from retrieved chunks.

    IMPORTANT: Use the ORIGINAL chunk text for generation, not the
    contextualized version. The contextual preamble is optimized for
    search, not for answer generation. Including it in the prompt
    can confuse the model.
    """
    client = anthropic.Anthropic()

    context_parts = []
    for i, chunk in enumerate(top_chunks):
        source = chunk["metadata"].get("source", "unknown")
        # Use original text, not contextualized text
        context_parts.append(f"[Source {i+1}: {source}]\n{chunk['original']}")

    context = "\n\n---\n\n".join(context_parts)

    response = client.messages.create(
        model=model,
        max_tokens=1500,
        system=(
            "Answer the question using ONLY the provided context. "
            "Cite sources as [Source N] for each claim. "
            "If the context is insufficient, say so explicitly."
        ),
        messages=[{
            "role": "user",
            "content": f"Context:\n{context}\n\nQuestion: {question}",
        }],
    )

    return {
        "answer": response.content[0].text,
        "sources": [c["metadata"] for c in top_chunks],
    }
```

---

## Complete End-to-End Pipeline

```python
import asyncio


async def full_contextual_retrieval_pipeline():
    """Complete pipeline from documents to answers."""

    # 1. Load and chunk documents
    documents = load_documents("./data/")  # Your document loader
    all_chunks = []
    doc_chunk_map = {}  # doc_id -> chunks

    for doc in documents:
        chunks = chunk_document(doc["text"], doc["id"], doc["source"])
        all_chunks.extend(chunks)
        doc_chunk_map[doc["id"]] = {
            "text": doc["text"],
            "chunks": chunks,
        }

    print(f"Total chunks: {len(all_chunks)}")

    # 2. Contextualize chunks (process one document at a time for caching)
    generator = ContextGenerator(max_concurrent=10)
    index = DualIndex()
    reranker = ContextualReranker()

    for doc_id, doc_data in doc_chunk_map.items():
        print(f"Processing {doc_id} ({len(doc_data['chunks'])} chunks)...")
        contextualized = await generator.process_document(
            doc_data["text"],
            doc_data["chunks"],
        )
        index.add_chunks(contextualized)

    print(f"Stats: {generator.stats}")
    index.save("./index/contextual_retrieval.pkl")

    # 3. Query
    question = "How do I configure streaming replication in PostgreSQL?"

    # Hybrid search
    candidates = index.search(question, top_k=25, bm25_weight=0.3)

    # Rerank
    top_chunks = reranker.rerank(question, candidates, top_k=5)

    # Generate
    result = generate_answer(question, top_chunks)
    print(result["answer"])


if __name__ == "__main__":
    asyncio.run(full_contextual_retrieval_pipeline())
```

---

## Processing Order for Maximum Cache Efficiency

The prompt caching mechanism caches the document text. To maximize cache hits:

```python
async def process_corpus_optimally(documents: list[dict]):
    """Process documents in order that maximizes cache hits.

    Rules:
    1. Process ALL chunks from one document before moving to the next.
    2. Process larger documents first (more chunks = more cache hits per document).
    3. Never interleave chunks from different documents.
    """
    # Sort by number of chunks (descending) to maximize cache value
    docs_with_chunks = []
    for doc in documents:
        chunks = chunk_document(doc["text"], doc["id"], doc["source"])
        docs_with_chunks.append({
            "doc": doc,
            "chunks": chunks,
            "num_chunks": len(chunks),
        })

    docs_with_chunks.sort(key=lambda x: x["num_chunks"], reverse=True)

    generator = ContextGenerator(max_concurrent=10)

    for item in docs_with_chunks:
        # All chunks from this document are processed together
        # Cache is warm for the document on the first call
        results = await generator.process_document(
            item["doc"]["text"],
            item["chunks"],
        )
        yield results
```

---

## Common Pitfalls

1. **Interleaving chunks from different documents.** This breaks prompt caching. The cache has a 5-minute TTL -- if you process chunks A1, B1, A2, B2, every call is a cache miss.
2. **Using the contextualized text for generation.** The context preamble ("This chunk is from the PostgreSQL guide...") is for search, not for the final answer. Pass original chunk text to the generation model.
3. **Generating overly long context preambles.** Keep `max_tokens=200` in the API call. Context should be 50-100 tokens. Longer context dilutes the embedding.
4. **Not combining with BM25.** Contextual embeddings alone give 35% improvement. Adding BM25 (which is free to run) pushes it to 49%.
5. **Processing all documents in parallel across chunks.** Process sequentially by document, in parallel within each document. This is the caching-optimal access pattern.
6. **Not monitoring cache hit rate.** Track `cache_read_input_tokens` vs `input_tokens` in the API response to verify caching is working. Cache hit rate should be > 80% for multi-chunk documents.

---

## References

- Anthropic, "Introducing Contextual Retrieval" -- https://www.anthropic.com/news/contextual-retrieval
- Anthropic, "Prompt Caching with Claude" -- https://docs.anthropic.com/en/docs/build-with-claude/prompt-caching
- Anthropic API usage tracking -- https://docs.anthropic.com/en/api/messages
