# Late Chunking -- Step-by-Step Implementation

## Overview / TL;DR

This guide provides the complete, runnable implementation of late chunking: loading jina-embeddings-v3 with 8192-token context, tokenizing documents, extracting token-level embeddings from the transformer, defining chunk boundaries (by sentence, paragraph, or fixed size), mean-pooling per boundary, and producing final chunk embeddings. It includes handling for documents longer than model context, batch processing for production ingestion, and integration patterns with vector databases.

---

## Prerequisites

```bash
pip install transformers torch sentence-transformers numpy
# For Jina v3 specifically:
pip install einops
```

Hardware requirements:
- **GPU recommended**: A10G (24GB) or better for jina-embeddings-v3 at full 8192 context.
- **CPU possible**: For documents under 2048 tokens, CPU is feasible but slow (5-10x slower).
- **Memory**: 4-8 GB GPU VRAM for typical usage, 16 GB for max context.

---

## Step 1: Load the Model

```python
import torch
from transformers import AutoModel, AutoTokenizer


def load_late_chunking_model(
    model_name: str = "jinaai/jina-embeddings-v3",
    device: str | None = None,
    use_fp16: bool = True,
) -> tuple:
    """Load a model suitable for late chunking.

    Args:
        model_name: HuggingFace model name. Must support long context
                    and expose token-level embeddings.
        device: 'cuda', 'cpu', or None for auto-detect.
        use_fp16: Use half precision for lower memory usage.

    Returns:
        Tuple of (model, tokenizer, device).
    """
    if device is None:
        device = "cuda" if torch.cuda.is_available() else "cpu"

    tokenizer = AutoTokenizer.from_pretrained(model_name, trust_remote_code=True)

    model = AutoModel.from_pretrained(
        model_name,
        trust_remote_code=True,
        torch_dtype=torch.float16 if (use_fp16 and device == "cuda") else torch.float32,
    )
    model = model.to(device)
    model.eval()

    print(f"Model loaded on {device}")
    print(f"Max position embeddings: {model.config.max_position_embeddings}")
    print(f"Hidden size: {model.config.hidden_size}")

    return model, tokenizer, device


model, tokenizer, device = load_late_chunking_model()
```

---

## Step 2: Tokenize the Full Document

```python
def tokenize_document(
    text: str,
    tokenizer: AutoTokenizer,
    max_length: int = 8192,
) -> dict:
    """Tokenize a document, returning token IDs, attention mask, and offset mapping.

    The offset mapping is critical for mapping token positions back to
    character positions in the original text.
    """
    encoded = tokenizer(
        text,
        return_tensors="pt",
        max_length=max_length,
        truncation=True,
        padding=False,
        return_offsets_mapping=True,
        return_special_tokens_mask=True,
    )

    total_tokens = encoded["input_ids"].shape[1]
    was_truncated = len(tokenizer.encode(text, add_special_tokens=False)) > max_length - 2

    return {
        "input_ids": encoded["input_ids"],
        "attention_mask": encoded["attention_mask"],
        "offset_mapping": encoded["offset_mapping"][0],  # (seq_len, 2)
        "special_tokens_mask": encoded["special_tokens_mask"][0],
        "total_tokens": total_tokens,
        "was_truncated": was_truncated,
        "original_text": text,
    }


# Example
doc = "FastAPI is a modern Python web framework. It supports async operations."
result = tokenize_document(doc, tokenizer)
print(f"Total tokens: {result['total_tokens']}")
print(f"Truncated: {result['was_truncated']}")
```

---

## Step 3: Get Token-Level Embeddings

```python
import numpy as np


def get_token_embeddings(
    model: AutoModel,
    input_ids: torch.Tensor,
    attention_mask: torch.Tensor,
    device: str,
) -> np.ndarray:
    """Run the full document through the transformer and extract token embeddings.

    Each token's embedding has attended to ALL other tokens via self-attention.
    This is what preserves cross-chunk context.

    Returns:
        numpy array of shape (seq_len, hidden_dim).
    """
    input_ids = input_ids.to(device)
    attention_mask = attention_mask.to(device)

    with torch.no_grad():
        outputs = model(
            input_ids=input_ids,
            attention_mask=attention_mask,
            output_hidden_states=True,
        )

    # Use last hidden state (most common choice)
    token_embeddings = outputs.last_hidden_state[0]  # (seq_len, hidden_dim)

    return token_embeddings.cpu().float().numpy()


# Example
token_encoded = tokenize_document(doc, tokenizer)
token_embs = get_token_embeddings(
    model,
    token_encoded["input_ids"],
    token_encoded["attention_mask"],
    device,
)
print(f"Token embeddings shape: {token_embs.shape}")
# e.g., (24, 1024) -- 24 tokens, 1024 hidden dims
```

---

## Step 4: Define Chunk Boundaries

Three strategies for defining where chunks begin and end.

### Strategy A: Fixed-Size Token Boundaries

```python
def fixed_size_boundaries(
    total_tokens: int,
    chunk_size: int = 256,
    overlap: int = 32,
    skip_special: bool = True,
) -> list[tuple[int, int]]:
    """Create fixed-size chunk boundaries at the token level.

    Args:
        total_tokens: Total number of tokens including special tokens.
        chunk_size: Number of content tokens per chunk.
        overlap: Number of overlapping tokens between chunks.
        skip_special: Skip [CLS] (index 0) and [SEP] (last index).

    Returns:
        List of (start_token_idx, end_token_idx) tuples (exclusive end).
    """
    start = 1 if skip_special else 0
    end_limit = total_tokens - 1 if skip_special else total_tokens
    step = chunk_size - overlap

    boundaries = []
    pos = start
    while pos < end_limit:
        end = min(pos + chunk_size, end_limit)
        boundaries.append((pos, end))
        pos += step

    return boundaries
```

### Strategy B: Sentence-Aligned Boundaries

```python
import nltk
nltk.download('punkt_tab', quiet=True)
from nltk.tokenize import sent_tokenize


def sentence_aligned_boundaries(
    text: str,
    tokenizer: AutoTokenizer,
    target_chunk_tokens: int = 256,
    max_chunk_tokens: int = 384,
) -> list[tuple[int, int, str]]:
    """Create chunk boundaries aligned to sentence endings.

    Sentences are grouped until reaching the target token count,
    ensuring chunks never break mid-sentence.

    Returns:
        List of (start_token_idx, end_token_idx, chunk_text) tuples.
    """
    sentences = sent_tokenize(text)
    boundaries = []
    current_sentences = []
    current_start = 1  # Skip [CLS]

    for sentence in sentences:
        current_sentences.append(sentence)
        combined = " ".join(current_sentences)
        n_tokens = len(tokenizer.encode(combined, add_special_tokens=False))

        if n_tokens >= target_chunk_tokens:
            # Calculate end position
            end_pos = current_start + n_tokens
            chunk_text = combined

            boundaries.append((current_start, end_pos, chunk_text))
            current_start = end_pos
            current_sentences = []

    # Handle remaining sentences
    if current_sentences:
        combined = " ".join(current_sentences)
        n_tokens = len(tokenizer.encode(combined, add_special_tokens=False))
        boundaries.append((current_start, current_start + n_tokens, combined))

    return boundaries
```

### Strategy C: Paragraph-Aligned Boundaries

```python
def paragraph_aligned_boundaries(
    text: str,
    tokenizer: AutoTokenizer,
    max_chunk_tokens: int = 512,
) -> list[tuple[int, int, str]]:
    """Create chunk boundaries aligned to paragraph breaks.

    Each paragraph becomes a chunk. Very long paragraphs are split
    at sentence boundaries.
    """
    paragraphs = [p.strip() for p in text.split("\n\n") if p.strip()]
    boundaries = []
    current_pos = 1  # Skip [CLS]

    for para in paragraphs:
        para_tokens = len(tokenizer.encode(para, add_special_tokens=False))

        if para_tokens <= max_chunk_tokens:
            boundaries.append((current_pos, current_pos + para_tokens, para))
            current_pos += para_tokens
        else:
            # Split long paragraph by sentences
            sents = sent_tokenize(para)
            current_sents = []
            chunk_start = current_pos

            for sent in sents:
                current_sents.append(sent)
                combined = " ".join(current_sents)
                n_tok = len(tokenizer.encode(combined, add_special_tokens=False))

                if n_tok >= max_chunk_tokens:
                    boundaries.append((chunk_start, chunk_start + n_tok, combined))
                    chunk_start += n_tok
                    current_sents = []

            if current_sents:
                combined = " ".join(current_sents)
                n_tok = len(tokenizer.encode(combined, add_special_tokens=False))
                boundaries.append((chunk_start, chunk_start + n_tok, combined))
                chunk_start += n_tok

            current_pos = chunk_start

    return boundaries
```

---

## Step 5: Mean-Pool Per Chunk

```python
def pool_chunks(
    token_embeddings: np.ndarray,
    boundaries: list[tuple[int, int]],
    normalize: bool = True,
) -> np.ndarray:
    """Mean-pool token embeddings within each chunk boundary.

    Args:
        token_embeddings: (seq_len, hidden_dim) array from the transformer.
        boundaries: List of (start, end) token index pairs.
        normalize: L2-normalize the resulting chunk embeddings.

    Returns:
        (n_chunks, hidden_dim) array of chunk embeddings.
    """
    chunk_embeddings = []

    for start, end in boundaries:
        # Clamp indices to valid range
        start = max(0, min(start, len(token_embeddings) - 1))
        end = max(start + 1, min(end, len(token_embeddings)))

        # Mean pool
        chunk_emb = token_embeddings[start:end].mean(axis=0)

        if normalize:
            norm = np.linalg.norm(chunk_emb)
            if norm > 0:
                chunk_emb = chunk_emb / norm

        chunk_embeddings.append(chunk_emb)

    return np.array(chunk_embeddings)
```

---

## Complete Pipeline: Full Runnable Code

```python
import torch
import numpy as np
from transformers import AutoModel, AutoTokenizer
from dataclasses import dataclass


@dataclass
class LateChunk:
    """A chunk produced by late chunking."""
    text: str
    embedding: np.ndarray
    start_token: int
    end_token: int
    n_tokens: int
    document_id: str


class LateChunkingPipeline:
    """Production-ready late chunking pipeline."""

    def __init__(
        self,
        model_name: str = "jinaai/jina-embeddings-v3",
        device: str | None = None,
        max_context: int = 8192,
        chunk_size: int = 256,
        chunk_overlap: int = 32,
    ):
        self.device = device or ("cuda" if torch.cuda.is_available() else "cpu")
        self.max_context = max_context
        self.chunk_size = chunk_size
        self.chunk_overlap = chunk_overlap

        self.tokenizer = AutoTokenizer.from_pretrained(model_name, trust_remote_code=True)
        self.model = AutoModel.from_pretrained(model_name, trust_remote_code=True)
        self.model = self.model.to(self.device)
        self.model.eval()

        self.hidden_dim = self.model.config.hidden_size

    def process_document(
        self,
        text: str,
        document_id: str = "",
    ) -> list[LateChunk]:
        """Process a single document with late chunking.

        Returns a list of LateChunk objects with embeddings that
        preserve full document context.
        """
        # Tokenize
        encoded = self.tokenizer(
            text,
            return_tensors="pt",
            max_length=self.max_context,
            truncation=True,
            return_offsets_mapping=True,
        )

        total_tokens = encoded["input_ids"].shape[1]
        offsets = encoded["offset_mapping"][0]

        # Get token embeddings
        with torch.no_grad():
            outputs = self.model(
                input_ids=encoded["input_ids"].to(self.device),
                attention_mask=encoded["attention_mask"].to(self.device),
                output_hidden_states=True,
            )
            token_embs = outputs.last_hidden_state[0].cpu().float().numpy()

        # Compute boundaries
        boundaries = self._compute_boundaries(total_tokens)

        # Pool and create chunks
        chunks = []
        for start, end in boundaries:
            # Mean pool
            chunk_emb = token_embs[start:end].mean(axis=0)
            norm = np.linalg.norm(chunk_emb)
            if norm > 0:
                chunk_emb /= norm

            # Extract text via offset mapping
            char_start = offsets[start][0].item() if start < len(offsets) else 0
            char_end = offsets[min(end - 1, len(offsets) - 1)][1].item() if end <= len(offsets) else len(text)
            chunk_text = text[char_start:char_end].strip()

            chunks.append(LateChunk(
                text=chunk_text,
                embedding=chunk_emb,
                start_token=start,
                end_token=end,
                n_tokens=end - start,
                document_id=document_id,
            ))

        return chunks

    def process_batch(
        self,
        documents: list[dict],  # [{"text": "...", "id": "..."}]
        show_progress: bool = True,
    ) -> list[LateChunk]:
        """Process multiple documents in batch.

        Documents are processed sequentially (each needs a full forward pass).
        """
        all_chunks = []

        for i, doc in enumerate(documents):
            if show_progress and i % 50 == 0:
                print(f"Processing document {i}/{len(documents)}")

            chunks = self.process_document(
                text=doc["text"],
                document_id=doc.get("id", f"doc_{i}"),
            )
            all_chunks.extend(chunks)

            # Periodically free GPU memory
            if i % 25 == 0 and self.device == "cuda":
                torch.cuda.empty_cache()

        return all_chunks

    def _compute_boundaries(self, total_tokens: int) -> list[tuple[int, int]]:
        """Compute chunk boundaries, skipping special tokens."""
        boundaries = []
        step = self.chunk_size - self.chunk_overlap
        pos = 1  # Skip [CLS]
        end_limit = total_tokens - 1  # Skip [SEP]

        while pos < end_limit:
            end = min(pos + self.chunk_size, end_limit)
            boundaries.append((pos, end))
            pos += step

        return boundaries


# Usage
pipeline = LateChunkingPipeline(
    chunk_size=256,
    chunk_overlap=32,
)

document = """
FastAPI is a modern Python web framework built on Starlette and Pydantic.
It provides automatic API documentation through OpenAPI and JSON Schema.
The framework supports both synchronous and asynchronous request handling.

One of its key features is dependency injection. The DI system allows you
to declare dependencies for each endpoint, and the framework resolves them
automatically. This makes testing and code reuse straightforward.

For production deployment, the framework integrates well with Docker.
You can build a container image with a simple Dockerfile and deploy it
to any container orchestration platform.
"""

chunks = pipeline.process_document(document, document_id="fastapi-guide")

for chunk in chunks:
    print(f"[{chunk.start_token}:{chunk.end_token}] ({chunk.n_tokens} tokens)")
    print(f"  Text: {chunk.text[:100]}...")
    print(f"  Embedding: {chunk.embedding.shape}, norm={np.linalg.norm(chunk.embedding):.4f}")
```

---

## Integration with Vector Databases

### Qdrant

```python
from qdrant_client import QdrantClient
from qdrant_client.models import Distance, VectorParams, PointStruct


def index_late_chunks_qdrant(
    chunks: list[LateChunk],
    collection_name: str = "late_chunked_docs",
    qdrant_url: str = "localhost",
):
    """Index late-chunked embeddings in Qdrant."""
    client = QdrantClient(url=qdrant_url)

    # Create collection
    dim = chunks[0].embedding.shape[0]
    client.recreate_collection(
        collection_name=collection_name,
        vectors_config=VectorParams(
            size=dim,
            distance=Distance.COSINE,
        ),
    )

    # Upload points
    points = [
        PointStruct(
            id=i,
            vector=chunk.embedding.tolist(),
            payload={
                "text": chunk.text,
                "document_id": chunk.document_id,
                "start_token": chunk.start_token,
                "end_token": chunk.end_token,
                "n_tokens": chunk.n_tokens,
            },
        )
        for i, chunk in enumerate(chunks)
    ]

    client.upsert(
        collection_name=collection_name,
        points=points,
    )

    print(f"Indexed {len(points)} late-chunked vectors in '{collection_name}'")
```

---

## Common Pitfalls

1. **Not skipping special tokens during pooling.** [CLS] and [SEP] tokens have different semantics from content tokens. Always exclude them from chunk pooling.
2. **Forgetting to normalize.** Mean-pooled embeddings are not unit-length. Always L2-normalize for cosine similarity.
3. **Using the wrong model output.** Some models expose multiple outputs. Use `last_hidden_state` for late chunking, not `pooler_output` (which is already pooled for the full sequence).
4. **Processing documents on CPU when GPU is available.** A single 8192-token forward pass takes ~50ms on GPU but ~5000ms on CPU. Check `torch.cuda.is_available()`.
5. **Not handling tokenizer offset mapping.** Without offset mapping, you cannot map token boundaries back to character positions in the original text, making it impossible to extract the chunk text.

---

## References

- Jina Late Chunking -- https://jina.ai/news/late-chunking-in-long-context-embedding-models/
- Late Chunking Paper -- https://arxiv.org/abs/2409.04701
- Jina Embeddings v3 -- https://huggingface.co/jinaai/jina-embeddings-v3
- HuggingFace Transformers -- https://huggingface.co/docs/transformers
