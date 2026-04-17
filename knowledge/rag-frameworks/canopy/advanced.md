# Canopy -- Advanced: Custom Chunkers, Context Engine Tuning, and Production Patterns

## Overview

This guide covers advanced Canopy patterns for production RAG systems: building custom chunkers for domain-specific content, tuning the context engine for optimal retrieval quality, implementing reranking, monitoring and evaluation, and production deployment strategies.

---

## Custom Chunkers

### Why Custom Chunking Matters

The default `RecursiveCharacterChunker` works well for general text, but domain-specific content often has structure that generic chunkers miss:

- **Code documentation**: Functions and classes should not be split mid-definition
- **Legal documents**: Sections and clauses should be kept together
- **API references**: Endpoint descriptions should be atomic units
- **Tables and structured data**: Rows should not be split across chunks

### Building a Code-Aware Chunker

```python
from canopy.knowledge_base.chunker import Chunker
from canopy.models.data_models import Document
from canopy.knowledge_base.models import KBDocChunk
import re


class CodeAwareChunker(Chunker):
    """Chunks documents preserving code blocks and function definitions."""

    def __init__(
        self,
        chunk_size: int = 512,
        chunk_overlap: int = 50,
        code_block_pattern: str = r"```[\s\S]*?```",
    ):
        self.chunk_size = chunk_size
        self.chunk_overlap = chunk_overlap
        self.code_block_pattern = code_block_pattern

    def chunk_documents(self, documents: list[Document]) -> list[KBDocChunk]:
        all_chunks = []

        for doc in documents:
            # Split into code blocks and text blocks
            blocks = self._split_preserving_code(doc.text)
            current_chunk = ""
            chunk_index = 0

            for block in blocks:
                block_tokens = len(block.split())

                # If adding this block exceeds chunk_size, finalize current chunk
                if (
                    current_chunk
                    and len(current_chunk.split()) + block_tokens > self.chunk_size
                ):
                    all_chunks.append(self._make_chunk(
                        doc, current_chunk.strip(), chunk_index,
                    ))
                    chunk_index += 1

                    # Overlap: keep last few sentences
                    overlap_text = self._get_overlap(current_chunk)
                    current_chunk = overlap_text + "\n\n" + block
                else:
                    current_chunk += "\n\n" + block if current_chunk else block

                # If a single block exceeds chunk_size (large code block), emit it alone
                if block_tokens > self.chunk_size and not current_chunk.strip() == block:
                    pass  # already handled above

            # Emit remaining text
            if current_chunk.strip():
                all_chunks.append(self._make_chunk(
                    doc, current_chunk.strip(), chunk_index,
                ))

        return all_chunks

    def _split_preserving_code(self, text: str) -> list[str]:
        """Split text into blocks, keeping code blocks intact."""
        blocks = []
        last_end = 0

        for match in re.finditer(self.code_block_pattern, text):
            # Text before code block
            pre_text = text[last_end:match.start()].strip()
            if pre_text:
                # Split pre-text by paragraphs
                paragraphs = re.split(r"\n\n+", pre_text)
                blocks.extend(p.strip() for p in paragraphs if p.strip())

            # Code block as a single unit
            blocks.append(match.group())
            last_end = match.end()

        # Remaining text after last code block
        remaining = text[last_end:].strip()
        if remaining:
            paragraphs = re.split(r"\n\n+", remaining)
            blocks.extend(p.strip() for p in paragraphs if p.strip())

        return blocks

    def _get_overlap(self, text: str) -> str:
        """Get the last N tokens for overlap."""
        words = text.split()
        overlap_words = words[-self.chunk_overlap:] if len(words) > self.chunk_overlap else words
        return " ".join(overlap_words)

    def _make_chunk(self, doc: Document, text: str, index: int) -> KBDocChunk:
        return KBDocChunk(
            id=f"{doc.id}_{index}",
            document_id=doc.id,
            text=text,
            source=doc.source,
            metadata={**doc.metadata, "chunk_index": index},
        )
```

### Heading-Aware Chunker for Documentation

```python
class HeadingChunker(Chunker):
    """Chunks markdown by heading hierarchy, keeping sections together."""

    def __init__(self, max_chunk_size: int = 1024, min_chunk_size: int = 100):
        self.max_chunk_size = max_chunk_size
        self.min_chunk_size = min_chunk_size

    def chunk_documents(self, documents: list[Document]) -> list[KBDocChunk]:
        all_chunks = []

        for doc in documents:
            sections = self._split_by_headings(doc.text)

            for i, section in enumerate(sections):
                heading, content = section
                text = f"{heading}\n\n{content}" if heading else content
                word_count = len(text.split())

                if word_count > self.max_chunk_size:
                    # Split large sections into sub-chunks
                    sub_chunks = self._split_large_section(text, self.max_chunk_size)
                    for j, sub in enumerate(sub_chunks):
                        all_chunks.append(KBDocChunk(
                            id=f"{doc.id}_{i}_{j}",
                            document_id=doc.id,
                            text=sub,
                            source=doc.source,
                            metadata={
                                **doc.metadata,
                                "heading": heading or "untitled",
                                "chunk_index": i,
                                "sub_chunk": j,
                            },
                        ))
                elif word_count >= self.min_chunk_size:
                    all_chunks.append(KBDocChunk(
                        id=f"{doc.id}_{i}",
                        document_id=doc.id,
                        text=text,
                        source=doc.source,
                        metadata={
                            **doc.metadata,
                            "heading": heading or "untitled",
                            "chunk_index": i,
                        },
                    ))
                # Skip sections below min_chunk_size (merge with next)

        return all_chunks

    def _split_by_headings(self, text: str) -> list[tuple[str, str]]:
        """Split markdown text by headings. Returns (heading, content) pairs."""
        sections = []
        current_heading = ""
        current_content = []

        for line in text.split("\n"):
            if re.match(r"^#{1,4}\s+", line):
                if current_content:
                    sections.append(
                        (current_heading, "\n".join(current_content).strip())
                    )
                current_heading = line.strip()
                current_content = []
            else:
                current_content.append(line)

        if current_content:
            sections.append(
                (current_heading, "\n".join(current_content).strip())
            )

        return sections

    def _split_large_section(self, text: str, max_size: int) -> list[str]:
        """Split a large section by paragraphs."""
        paragraphs = text.split("\n\n")
        chunks = []
        current = ""

        for para in paragraphs:
            if len((current + "\n\n" + para).split()) > max_size and current:
                chunks.append(current.strip())
                current = para
            else:
                current = current + "\n\n" + para if current else para

        if current.strip():
            chunks.append(current.strip())

        return chunks
```

### Registering Custom Chunkers

```python
from canopy.knowledge_base import KnowledgeBase

# Use your custom chunker
kb = KnowledgeBase(
    index_name="my-knowledge-base",
    chunker=CodeAwareChunker(chunk_size=512, chunk_overlap=50),
)

# Now all kb.upsert() calls use the custom chunker
kb.upsert(documents)
```

---

## Context Engine Tuning

### Adjusting Retrieval Parameters

```python
from canopy.knowledge_base import KnowledgeBase
from canopy.context_engine import ContextEngine

# More documents, larger context window
kb = KnowledgeBase(
    index_name="my-knowledge-base",
    default_top_k=15,         # retrieve more candidates
)

context_engine = ContextEngine(
    knowledge_base=kb,
    max_context_tokens=6000,  # larger context budget
)
```

### Multi-Query Context Retrieval

For complex questions, generate multiple search queries and merge results:

```python
from canopy.context_engine import ContextEngine
from canopy.knowledge_base import KnowledgeBase
import openai


def multi_query_context(
    context_engine: ContextEngine,
    question: str,
    num_queries: int = 3,
    max_context_tokens: int = 4000,
) -> dict:
    """Generate multiple search queries for better recall."""
    # Use LLM to generate query variations
    oai = openai.OpenAI()
    response = oai.chat.completions.create(
        model="gpt-4o-mini",
        temperature=0.5,
        messages=[
            {
                "role": "system",
                "content": "Generate search queries to find information that answers the user's question. "
                           "Return one query per line, no numbering.",
            },
            {
                "role": "user",
                "content": f"Generate {num_queries} different search queries for: {question}",
            },
        ],
    )

    queries = [
        {"text": q.strip()}
        for q in response.choices[0].message.content.strip().split("\n")
        if q.strip()
    ]

    # Also include the original question
    queries.insert(0, {"text": question})

    # Retrieve context for all queries
    context = context_engine.query(
        queries=queries[:num_queries + 1],
        max_context_tokens=max_context_tokens,
    )

    return context
```

### Context with Metadata Filtering

```python
from canopy.knowledge_base import KnowledgeBase

kb = KnowledgeBase(index_name="my-knowledge-base")

# Filter by metadata during retrieval
results = kb.query(
    queries=[{"text": "deployment configuration"}],
    top_k=10,
    metadata_filter={
        "category": {"$eq": "devops"},
        "version": {"$gte": "2.0"},
    },
)
```

---

## Adding Reranking

Canopy does not include built-in reranking, but you can add it with a custom context engine:

```python
from canopy.context_engine import ContextEngine
from canopy.knowledge_base import KnowledgeBase
from canopy.knowledge_base.models import KBQueryResult
from sentence_transformers import CrossEncoder


class RerankingContextEngine(ContextEngine):
    """Context engine with cross-encoder reranking."""

    def __init__(
        self,
        knowledge_base: KnowledgeBase,
        reranker_model: str = "cross-encoder/ms-marco-MiniLM-L-6-v2",
        rerank_top_k: int = 5,
        **kwargs,
    ):
        super().__init__(knowledge_base=knowledge_base, **kwargs)
        self.reranker = CrossEncoder(reranker_model)
        self.rerank_top_k = rerank_top_k

    def query(self, queries, max_context_tokens=None):
        # Retrieve more candidates than needed
        original_top_k = self._knowledge_base.default_top_k

        # Get raw results from knowledge base
        raw_results = self._knowledge_base.query(
            queries=queries,
            top_k=original_top_k * 2,  # retrieve 2x for reranking
        )

        # Rerank each query's results
        reranked_results = []
        for query_data, results in zip(queries, raw_results):
            query_text = query_data["text"]

            if not results:
                reranked_results.append([])
                continue

            # Score with cross-encoder
            pairs = [(query_text, r.text) for r in results]
            scores = self.reranker.predict(pairs, batch_size=32)

            # Sort by reranker score
            scored = sorted(
                zip(results, scores),
                key=lambda x: x[1],
                reverse=True,
            )

            # Take top-k reranked results
            reranked = [r for r, _ in scored[:self.rerank_top_k]]
            reranked_results.append(reranked)

        # Build context from reranked results
        return self._build_context(reranked_results, max_context_tokens)


# Usage
kb = KnowledgeBase(index_name="my-knowledge-base", default_top_k=20)
context_engine = RerankingContextEngine(
    knowledge_base=kb,
    reranker_model="cross-encoder/ms-marco-MiniLM-L-6-v2",
    rerank_top_k=5,
    max_context_tokens=4000,
)
```

---

## Evaluation

### Retrieval Quality Metrics

```python
from canopy.knowledge_base import KnowledgeBase
import numpy as np


def evaluate_retrieval(
    kb: KnowledgeBase,
    test_set: list[dict],  # [{"query": "...", "relevant_chunk_texts": ["..."]}]
    top_k_values: list[int] = [1, 3, 5, 10],
) -> dict:
    """Evaluate retrieval quality with MRR and Recall@K."""
    metrics = {f"recall@{k}": [] for k in top_k_values}
    metrics["mrr"] = []

    max_k = max(top_k_values)

    for test in test_set:
        results = kb.query(
            queries=[{"text": test["query"]}],
            top_k=max_k,
        )

        if not results or not results[0]:
            for k in top_k_values:
                metrics[f"recall@{k}"].append(0.0)
            metrics["mrr"].append(0.0)
            continue

        retrieved_texts = [r.text for r in results[0]]
        relevant = test["relevant_chunk_texts"]

        # MRR
        rr = 0.0
        for rank, text in enumerate(retrieved_texts, 1):
            if any(rel in text or text in rel for rel in relevant):
                rr = 1.0 / rank
                break
        metrics["mrr"].append(rr)

        # Recall@K
        for k in top_k_values:
            top_k = retrieved_texts[:k]
            found = sum(
                1 for rel in relevant
                if any(rel in t or t in rel for t in top_k)
            )
            metrics[f"recall@{k}"].append(found / len(relevant))

    return {k: float(np.mean(v)) for k, v in metrics.items()}


# Example evaluation
test_set = [
    {
        "query": "How does Pinecone serverless work?",
        "relevant_chunk_texts": [
            "Pinecone serverless automatically scales based on usage",
        ],
    },
    {
        "query": "What is Canopy?",
        "relevant_chunk_texts": [
            "Canopy is Pinecone's open-source RAG framework",
        ],
    },
]

kb = KnowledgeBase(index_name="my-knowledge-base")
metrics = evaluate_retrieval(kb, test_set)
print("Retrieval metrics:")
for metric, score in metrics.items():
    print(f"  {metric}: {score:.4f}")
```

### End-to-End RAG Evaluation

```python
from canopy.chat_engine import ChatEngine
from canopy.models.data_models import MessageBase
import openai


def evaluate_rag_quality(
    chat_engine: ChatEngine,
    test_set: list[dict],  # [{"question": "...", "expected_answer": "..."}]
) -> dict:
    """Evaluate RAG answer quality using LLM-as-judge."""
    oai = openai.OpenAI()
    scores = {"correctness": [], "completeness": [], "faithfulness": []}

    for test in test_set:
        messages = [MessageBase(role="user", content=test["question"])]
        response = chat_engine.chat(messages=messages)
        predicted = response.choices[0].message.content

        # LLM judge for correctness
        judge_prompt = f"""Rate the predicted answer against the reference on a scale of 1-5.

Question: {test["question"]}
Reference answer: {test["expected_answer"]}
Predicted answer: {predicted}

Return ONLY a JSON object: {{"correctness": N, "completeness": N}}"""

        judge_response = oai.chat.completions.create(
            model="gpt-4o-mini",
            temperature=0,
            messages=[{"role": "user", "content": judge_prompt}],
        )

        try:
            import json
            judge_scores = json.loads(judge_response.choices[0].message.content)
            scores["correctness"].append(judge_scores.get("correctness", 3) / 5.0)
            scores["completeness"].append(judge_scores.get("completeness", 3) / 5.0)
        except (json.JSONDecodeError, KeyError):
            scores["correctness"].append(0.5)
            scores["completeness"].append(0.5)

    return {k: sum(v) / len(v) for k, v in scores.items()}
```

---

## Production Deployment

### Docker Deployment

```dockerfile
FROM python:3.11-slim

WORKDIR /app

RUN pip install canopy-sdk gunicorn

COPY canopy_config.yaml /app/
COPY startup.py /app/

ENV PINECONE_API_KEY=""
ENV OPENAI_API_KEY=""
ENV CANOPY_CONFIG_PATH="/app/canopy_config.yaml"

EXPOSE 8000

CMD ["gunicorn", "startup:app", "-w", "4", "-k", "uvicorn.workers.UvicornWorker", "-b", "0.0.0.0:8000"]
```

### Health Monitoring

```python
from fastapi import FastAPI
from canopy.chat_engine import ChatEngine
from canopy.models.data_models import MessageBase
import time

app = FastAPI()
engine = ChatEngine()


@app.get("/health")
async def health():
    """Deep health check: verify Pinecone and LLM connectivity."""
    checks = {}

    # Check Pinecone
    start = time.time()
    try:
        context = engine._context_engine.query(
            queries=[{"text": "health check"}],
            max_context_tokens=100,
        )
        checks["pinecone"] = {
            "status": "ok",
            "latency_ms": (time.time() - start) * 1000,
        }
    except Exception as e:
        checks["pinecone"] = {"status": "error", "message": str(e)}

    # Check LLM
    start = time.time()
    try:
        response = engine.chat(
            messages=[MessageBase(role="user", content="Reply with 'ok'")],
        )
        checks["llm"] = {
            "status": "ok",
            "latency_ms": (time.time() - start) * 1000,
        }
    except Exception as e:
        checks["llm"] = {"status": "error", "message": str(e)}

    overall = "ok" if all(c["status"] == "ok" for c in checks.values()) else "degraded"
    return {"status": overall, "checks": checks}
```

### Request Logging and Analytics

```python
import logging
import time
import json
from functools import wraps

logger = logging.getLogger("canopy.analytics")


def log_query(func):
    """Decorator to log query latency and token usage."""
    @wraps(func)
    def wrapper(*args, **kwargs):
        start = time.time()
        try:
            result = func(*args, **kwargs)
            latency = (time.time() - start) * 1000

            # Extract usage info
            usage = {}
            if hasattr(result, "usage"):
                usage = {
                    "prompt_tokens": result.usage.prompt_tokens,
                    "completion_tokens": result.usage.completion_tokens,
                    "total_tokens": result.usage.total_tokens,
                }

            logger.info(json.dumps({
                "event": "query",
                "latency_ms": round(latency, 2),
                "status": "success",
                "usage": usage,
            }))

            return result
        except Exception as e:
            latency = (time.time() - start) * 1000
            logger.error(json.dumps({
                "event": "query",
                "latency_ms": round(latency, 2),
                "status": "error",
                "error": str(e),
            }))
            raise

    return wrapper
```

---

## Common Pitfalls

1. **Custom chunker not producing unique IDs**: Each `KBDocChunk` must have a unique `id`. If two chunks share an ID, one overwrites the other in Pinecone. Always include the chunk index in the ID.

2. **Reranking without enough candidates**: If `default_top_k` is too low, the reranker has few candidates to work with. Set retrieval top_k to 2-3x the desired final count.

3. **Not testing chunker edge cases**: Empty documents, documents with only code blocks, and single-line documents can cause chunkers to produce empty chunks. Add guards for these cases.

4. **Ignoring token budget in context engine**: If `max_context_tokens` is larger than the model's context window minus the system prompt and max output tokens, the LLM call will fail with a context length error.

5. **Missing Pinecone index configuration**: Canopy-specific metadata fields require the index to be created with `kb.create_canopy_index()`. Using a manually created Pinecone index may work for search but fail for context building.

6. **Not warming up cross-encoder models**: The first reranking call loads the model and is slow. In production, warm up the model at startup with a dummy query.

---

## References

- Canopy GitHub: https://github.com/pinecone-io/canopy
- Pinecone documentation: https://docs.pinecone.io/
- Cross-encoder models: https://www.sbert.net/docs/cross_encoder/pretrained_models.html
- Canopy architecture: https://www.pinecone.io/blog/canopy-rag-framework/
