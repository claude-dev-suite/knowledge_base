# Late Chunking -- Comprehensive Guide

## Overview / TL;DR

Late chunking is an embedding technique introduced by Jina AI that reverses the traditional "chunk then embed" order. Instead of splitting a document into chunks and embedding each chunk independently, late chunking passes the entire document through a long-context embedding model to get token-level embeddings, then applies mean pooling per chunk boundary after the transformer has processed the full context. This preserves cross-chunk context that traditional chunking destroys. A pronoun like "it" in chunk 5 retains the context from its antecedent in chunk 1 because the transformer attended to the full document before pooling. Late chunking improves retrieval quality by 5-12% on long documents with cross-references, at the cost of higher memory usage and longer processing time.

---

## The Problem: Traditional Chunk-Then-Embed

In traditional RAG pipelines:

1. Split a document into chunks (e.g., 512 tokens each).
2. Embed each chunk independently through the embedding model.
3. Store embeddings in a vector index.

This works well when each chunk is self-contained. But many documents have cross-references:

- **Pronouns**: "The system uses a message queue. **It** processes 10K messages/second." -- If "It processes..." is in a different chunk, the embedding for "It" has no idea what "it" refers to.
- **Section references**: "As described in Section 2.1, the architecture..." -- The embedding cannot capture what Section 2.1 said.
- **Acronym definitions**: "We use Kubernetes (K8s) for orchestration. [... 3 paragraphs later ...] K8s autoscaling adjusts..." -- The later chunk does not know K8s = Kubernetes.

### Concrete Example

```
Chunk 1: "FastAPI is a modern Python web framework built on Starlette
          and Pydantic. It provides automatic API documentation,
          type validation, and async support."

Chunk 2: "The framework supports dependency injection natively.
          It can handle thousands of concurrent requests with
          minimal memory overhead."
```

In chunk 2, "The framework" and "It" refer to FastAPI, but when embedded independently, the model has no way to resolve these references. The embedding for chunk 2 is a generic statement about some unnamed framework -- it will match queries about any framework, not specifically FastAPI.

---

## How Late Chunking Works

### Step-by-Step Process

1. **Tokenize the full document** (up to the model's context limit, e.g., 8192 tokens).
2. **Pass all tokens through the transformer encoder** to get token-level embeddings. Each token's embedding has attended to every other token in the document via self-attention.
3. **Define chunk boundaries** at the token level (e.g., token 0-127 = chunk 1, tokens 128-255 = chunk 2, etc.).
4. **Mean-pool the token embeddings within each chunk boundary** to produce one embedding per chunk.

The key insight: in step 2, the transformer's self-attention mechanism allows every token to "see" the entire document. When we then pool tokens 128-255 into a chunk embedding (step 4), the embedding for "It" in that range already encodes the knowledge that "It" refers to FastAPI from tokens 0-127.

### Architecture Diagram

```
Traditional (chunk-then-embed):
  Document -> [Chunk 1] -> Embed -> Vector 1
              [Chunk 2] -> Embed -> Vector 2
              [Chunk 3] -> Embed -> Vector 3

Late chunking (embed-then-chunk):
  Document -> Full Transformer Pass -> Token embeddings for ALL tokens
                                        |
                                        +-> Pool tokens [0:128]    -> Vector 1
                                        +-> Pool tokens [128:256]  -> Vector 2
                                        +-> Pool tokens [256:384]  -> Vector 3
```

---

## Why It Works: Self-Attention and Context Propagation

In a transformer encoder, self-attention at each layer allows every token to gather information from every other token. After 12-24 layers of attention, each token's representation is a function of the entire input sequence.

When token 150 represents "It" (referring to FastAPI defined at token 20), the self-attention mechanism has already propagated the concept of "FastAPI" into the representation of token 150. By the time we pool tokens 128-255, the "It" token carries the meaning "FastAPI", not just "some unresolved pronoun."

This is why late chunking requires a long-context model -- the model must be able to process the entire document in a single forward pass. Models with 512-token limits cannot do late chunking on multi-page documents.

---

## Comparison with Traditional Chunking

| Aspect | Traditional Chunking | Late Chunking |
|--------|---------------------|---------------|
| Context preservation | None between chunks | Full document context |
| Pronoun resolution | Lost | Preserved |
| Cross-reference quality | Poor | Good |
| Processing speed | Fast (parallel chunks) | Slower (sequential full doc) |
| Memory usage | Low (one chunk at a time) | High (full document at once) |
| Model requirement | Any embedding model | Long-context model (4K+ tokens) |
| Best for | Short, self-contained texts | Long docs with cross-references |
| Overhead vs traditional | -- | 1.5-3x slower, 2-4x more memory |

---

## Compatible Models

Late chunking requires a model that:
1. Supports long context (ideally 8K+ tokens).
2. Exposes token-level embeddings (not just sentence-level).
3. Has been trained on long documents.

| Model | Max Tokens | Late Chunking Support | Notes |
|-------|-----------|----------------------|-------|
| jina-embeddings-v3 | 8192 | Native (designed for it) | Best model for late chunking |
| jina-embeddings-v2 | 8192 | Yes | Previous generation |
| nomic-embed-text-v1.5 | 8192 | Possible (manual) | Requires custom implementation |
| BGE-M3 | 8192 | Possible (manual) | Works but not optimized for it |
| text-embedding-3-large | 8191 | No (API, no token access) | Cannot access token-level embeddings |
| Voyage-3 | 32000 | No (API, no token access) | Cannot access token-level embeddings |

**Note**: API-based models (OpenAI, Voyage, Cohere) cannot do late chunking because they only return the final pooled embedding, not token-level embeddings. Late chunking requires access to the raw transformer output.

---

## Implementation with Jina Embeddings v3

```python
import torch
from transformers import AutoModel, AutoTokenizer
import numpy as np


class LateChunker:
    """Late chunking implementation using Jina embeddings v3.

    Embeds the full document through the transformer, then pools
    per chunk boundary to produce chunk-level embeddings with
    full document context.
    """

    def __init__(
        self,
        model_name: str = "jinaai/jina-embeddings-v3",
        device: str = "cuda" if torch.cuda.is_available() else "cpu",
        max_length: int = 8192,
    ):
        self.tokenizer = AutoTokenizer.from_pretrained(model_name, trust_remote_code=True)
        self.model = AutoModel.from_pretrained(model_name, trust_remote_code=True)
        self.model = self.model.to(device)
        self.model.eval()
        self.device = device
        self.max_length = max_length

    def embed_late_chunked(
        self,
        document: str,
        chunk_size_tokens: int = 256,
        chunk_overlap_tokens: int = 32,
    ) -> list[dict]:
        """Embed a document using late chunking.

        Args:
            document: Full document text.
            chunk_size_tokens: Target chunk size in tokens.
            chunk_overlap_tokens: Overlap between adjacent chunks.

        Returns:
            List of dicts with 'embedding', 'text', 'start_token', 'end_token'.
        """
        # Step 1: Tokenize the full document
        encoded = self.tokenizer(
            document,
            return_tensors="pt",
            max_length=self.max_length,
            truncation=True,
            return_offsets_mapping=True,
        )
        input_ids = encoded["input_ids"].to(self.device)
        attention_mask = encoded["attention_mask"].to(self.device)
        offsets = encoded["offset_mapping"][0]  # (seq_len, 2)
        total_tokens = input_ids.shape[1]

        # Step 2: Get token-level embeddings from the transformer
        with torch.no_grad():
            outputs = self.model(
                input_ids=input_ids,
                attention_mask=attention_mask,
                output_hidden_states=True,
            )
            # Use last hidden state as token embeddings
            token_embeddings = outputs.last_hidden_state[0]  # (seq_len, hidden_dim)

        # Step 3: Define chunk boundaries
        chunk_boundaries = self._compute_boundaries(
            total_tokens=total_tokens,
            chunk_size=chunk_size_tokens,
            overlap=chunk_overlap_tokens,
        )

        # Step 4: Mean-pool per chunk boundary
        chunks = []
        for start, end in chunk_boundaries:
            # Mean pool the token embeddings in this range
            chunk_emb = token_embeddings[start:end].mean(dim=0)
            # L2 normalize
            chunk_emb = chunk_emb / chunk_emb.norm()
            chunk_emb = chunk_emb.cpu().numpy()

            # Extract the text for this chunk using offset mapping
            if start < len(offsets) and end <= len(offsets):
                char_start = offsets[start][0].item()
                char_end = offsets[min(end - 1, len(offsets) - 1)][1].item()
                chunk_text = document[char_start:char_end]
            else:
                chunk_text = ""

            chunks.append({
                "embedding": chunk_emb,
                "text": chunk_text,
                "start_token": start,
                "end_token": end,
                "n_tokens": end - start,
            })

        return chunks

    def _compute_boundaries(
        self,
        total_tokens: int,
        chunk_size: int,
        overlap: int,
    ) -> list[tuple[int, int]]:
        """Compute chunk boundaries as (start, end) token index pairs."""
        boundaries = []
        step = chunk_size - overlap
        start = 1  # Skip [CLS] token

        while start < total_tokens - 1:  # Skip [SEP] token
            end = min(start + chunk_size, total_tokens - 1)
            boundaries.append((start, end))
            start += step

        return boundaries


# Usage
chunker = LateChunker()

document = """
FastAPI is a modern Python web framework built on Starlette and Pydantic.
It provides automatic API documentation, type validation, and async support.

The framework supports dependency injection natively. It can handle thousands
of concurrent requests with minimal memory overhead. This makes it ideal for
microservices architectures.

For deployment, the framework works seamlessly with Docker and Kubernetes.
Auto-scaling can be configured to handle variable load patterns.
"""

chunks = chunker.embed_late_chunked(
    document,
    chunk_size_tokens=64,
    chunk_overlap_tokens=8,
)

for i, chunk in enumerate(chunks):
    print(f"Chunk {i}: {chunk['n_tokens']} tokens, text: {chunk['text'][:80]}...")
    print(f"  Embedding shape: {chunk['embedding'].shape}")
```

---

## Handling Documents Longer Than Model Context

When documents exceed the model's context window (e.g., 8192 tokens), you need a strategy to handle the overflow.

### Strategy 1: Sliding Window with Overlap

Process the document in overlapping windows, each within the model's context limit. Tokens in overlapping regions get embeddings from both windows -- average them.

```python
def late_chunk_long_document(
    chunker: LateChunker,
    document: str,
    window_size_tokens: int = 7680,  # Leave room for special tokens
    window_overlap_tokens: int = 512,
    chunk_size_tokens: int = 256,
) -> list[dict]:
    """Late chunking for documents longer than model context.

    Uses sliding windows with overlap. Token embeddings in overlapping
    regions are averaged across windows.
    """
    # Tokenize full document to get total length
    full_tokens = chunker.tokenizer.encode(document, add_special_tokens=False)
    total_tokens = len(full_tokens)

    if total_tokens <= window_size_tokens:
        # Fits in one pass
        return chunker.embed_late_chunked(document, chunk_size_tokens)

    # Process in windows
    step = window_size_tokens - window_overlap_tokens
    all_token_embeddings = np.zeros((total_tokens, chunker.model.config.hidden_size))
    token_counts = np.zeros(total_tokens)  # Track how many times each token was embedded

    for win_start in range(0, total_tokens, step):
        win_end = min(win_start + window_size_tokens, total_tokens)
        window_token_ids = full_tokens[win_start:win_end]

        # Decode back to text for this window
        window_text = chunker.tokenizer.decode(window_token_ids)

        # Get token embeddings for this window
        encoded = chunker.tokenizer(
            window_text,
            return_tensors="pt",
            max_length=chunker.max_length,
            truncation=True,
        )

        with torch.no_grad():
            outputs = chunker.model(
                input_ids=encoded["input_ids"].to(chunker.device),
                attention_mask=encoded["attention_mask"].to(chunker.device),
                output_hidden_states=True,
            )
            window_embs = outputs.last_hidden_state[0].cpu().numpy()

        # Skip [CLS] and [SEP] tokens
        content_embs = window_embs[1:-1]
        n_content = min(len(content_embs), win_end - win_start)

        # Accumulate (for averaging in overlap regions)
        all_token_embeddings[win_start:win_start + n_content] += content_embs[:n_content]
        token_counts[win_start:win_start + n_content] += 1

        if win_end >= total_tokens:
            break

    # Average overlapping regions
    mask = token_counts > 0
    all_token_embeddings[mask] /= token_counts[mask, np.newaxis]

    # Now pool into chunks
    chunks = []
    for start in range(0, total_tokens, chunk_size_tokens):
        end = min(start + chunk_size_tokens, total_tokens)
        chunk_emb = all_token_embeddings[start:end].mean(axis=0)
        norm = np.linalg.norm(chunk_emb)
        if norm > 0:
            chunk_emb /= norm

        chunk_text = chunker.tokenizer.decode(full_tokens[start:end])
        chunks.append({
            "embedding": chunk_emb,
            "text": chunk_text,
            "start_token": start,
            "end_token": end,
        })

    return chunks
```

### Strategy 2: Hierarchical Late Chunking

For very long documents, use a hierarchical approach: split into sections that fit the context window, late-chunk each section, and add a section-level embedding for broader context.

```python
def hierarchical_late_chunk(
    chunker: LateChunker,
    document: str,
    section_delimiter: str = "\n## ",
    chunk_size_tokens: int = 256,
) -> list[dict]:
    """Hierarchical late chunking: section-level then chunk-level.

    Splits document by headings, applies late chunking within each section.
    Each chunk gets section-level context (from the section heading and intro)
    plus intra-section context (from late chunking).
    """
    # Split into sections
    sections = document.split(section_delimiter)
    all_chunks = []

    for section_idx, section_text in enumerate(sections):
        if not section_text.strip():
            continue

        # Restore delimiter for proper text
        if section_idx > 0:
            section_text = section_delimiter.lstrip() + section_text

        # Late chunk this section
        section_chunks = chunker.embed_late_chunked(
            section_text,
            chunk_size_tokens=chunk_size_tokens,
        )

        for chunk in section_chunks:
            chunk["section_index"] = section_idx
            all_chunks.append(chunk)

    return all_chunks
```

---

## Batch Processing

For production ingestion pipelines, process documents in batches.

```python
import torch
from typing import Generator


def batch_late_chunk(
    chunker: LateChunker,
    documents: list[str],
    chunk_size_tokens: int = 256,
    show_progress: bool = True,
) -> list[list[dict]]:
    """Late chunk multiple documents.

    Each document is processed independently (no cross-document attention).
    Returns a list of chunk lists, one per document.
    """
    results = []
    total = len(documents)

    for i, doc in enumerate(documents):
        if show_progress and i % 100 == 0:
            print(f"Processing document {i}/{total}")

        chunks = chunker.embed_late_chunked(doc, chunk_size_tokens=chunk_size_tokens)
        results.append(chunks)

        # Free GPU memory periodically
        if i % 50 == 0:
            torch.cuda.empty_cache()

    return results
```

---

## Common Pitfalls

1. **Using API-based models.** Late chunking requires token-level embeddings, which are not available through OpenAI, Voyage, or Cohere APIs. You need a locally-loaded transformer model.
2. **Ignoring memory limits.** A full 8192-token forward pass uses significantly more GPU memory than embedding a 256-token chunk. Budget for 4-8x more memory than traditional embedding.
3. **Applying late chunking to short documents.** For documents under 512 tokens, traditional chunking and late chunking produce nearly identical results. The benefit is proportional to document length and cross-reference density.
4. **Not handling documents longer than the context window.** Documents that exceed 8192 tokens require windowing. Silently truncating loses content.
5. **Using models not trained on long contexts.** A model trained with 512-token max length will produce poor embeddings at positions 4000+ even if the architecture supports it. Use models explicitly trained on long contexts.

---

## References

- Jina Late Chunking Blog -- https://jina.ai/news/late-chunking-in-long-context-embedding-models/
- Jina Embeddings v3 -- https://jina.ai/embeddings/
- Late Chunking Paper -- https://arxiv.org/abs/2409.04701
- Long-Context Embedding Models -- https://arxiv.org/abs/2404.12096
