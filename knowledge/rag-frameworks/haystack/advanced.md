# Haystack 2.x -- Advanced: Custom Components, Hybrid Search, and Evaluation

## Overview

This guide covers advanced Haystack 2.x patterns for production RAG systems: building custom components, implementing hybrid retrieval with reranking, evaluation with RAGAS-style metrics, and optimization strategies. These patterns build on the basics covered in `getting-started.md` and the architecture in `overview.md`.

---

## Custom Components

### The Full Component Protocol

A production-quality custom component handles initialization, serialization, warm-up, and error recovery:

```python
from haystack import component, Document, default_from_dict, default_to_dict
from typing import List, Optional, Any
import logging

logger = logging.getLogger(__name__)


@component
class CrossEncoderReranker:
    """Reranks documents using a cross-encoder model for more accurate relevance scoring."""

    def __init__(
        self,
        model_name: str = "cross-encoder/ms-marco-MiniLM-L-6-v2",
        top_k: int = 5,
        batch_size: int = 32,
        device: Optional[str] = None,
        score_threshold: Optional[float] = None,
    ):
        self.model_name = model_name
        self.top_k = top_k
        self.batch_size = batch_size
        self.device = device
        self.score_threshold = score_threshold
        self.model = None  # loaded in warm_up

    def warm_up(self):
        """Load the cross-encoder model. Called once before first run()."""
        if self.model is not None:
            return
        from sentence_transformers import CrossEncoder
        self.model = CrossEncoder(
            self.model_name,
            device=self.device,
            max_length=512,
        )
        logger.info(f"Loaded cross-encoder: {self.model_name}")

    @component.output_types(documents=List[Document])
    def run(
        self,
        query: str,
        documents: List[Document],
        top_k: Optional[int] = None,
    ) -> dict:
        """Rerank documents by relevance to the query."""
        if not documents:
            return {"documents": []}

        if self.model is None:
            raise RuntimeError("Component not warmed up. Call warm_up() first.")

        top_k = top_k or self.top_k

        # Create query-document pairs for the cross-encoder
        pairs = [(query, doc.content) for doc in documents]

        # Score all pairs
        scores = self.model.predict(
            pairs,
            batch_size=self.batch_size,
            show_progress_bar=False,
        )

        # Attach scores and sort
        scored_docs = []
        for doc, score in zip(documents, scores):
            doc_copy = Document(
                content=doc.content,
                meta={**doc.meta, "rerank_score": float(score)},
                embedding=doc.embedding,
                score=float(score),
            )
            scored_docs.append(doc_copy)

        scored_docs.sort(key=lambda d: d.score, reverse=True)

        # Apply threshold filter if set
        if self.score_threshold is not None:
            scored_docs = [d for d in scored_docs if d.score >= self.score_threshold]

        return {"documents": scored_docs[:top_k]}

    def to_dict(self) -> dict:
        """Serialize component for pipeline YAML export."""
        return default_to_dict(
            self,
            model_name=self.model_name,
            top_k=self.top_k,
            batch_size=self.batch_size,
            device=self.device,
            score_threshold=self.score_threshold,
        )

    @classmethod
    def from_dict(cls, data: dict) -> "CrossEncoderReranker":
        """Deserialize component from pipeline YAML."""
        return default_from_dict(cls, data)
```

### Custom Document Processor

```python
from haystack import component, Document
from typing import List
import re


@component
class MetadataEnricher:
    """Enriches documents with computed metadata fields."""

    def __init__(self, extract_entities: bool = True, compute_readability: bool = True):
        self.extract_entities = extract_entities
        self.compute_readability = compute_readability

    @component.output_types(documents=List[Document])
    def run(self, documents: List[Document]) -> dict:
        enriched = []
        for doc in documents:
            meta = {**doc.meta}

            # Word count
            words = doc.content.split()
            meta["word_count"] = len(words)

            # Sentence count
            sentences = re.split(r'[.!?]+', doc.content)
            meta["sentence_count"] = len([s for s in sentences if s.strip()])

            # Average sentence length (readability proxy)
            if self.compute_readability and meta["sentence_count"] > 0:
                meta["avg_sentence_length"] = meta["word_count"] / meta["sentence_count"]

            # Simple entity extraction (URLs, emails)
            if self.extract_entities:
                meta["urls"] = re.findall(r'https?://\S+', doc.content)
                meta["emails"] = re.findall(r'\b[\w.+-]+@[\w-]+\.[\w.-]+\b', doc.content)
                meta["code_blocks"] = len(re.findall(r'```', doc.content)) // 2

            enriched.append(Document(
                content=doc.content,
                meta=meta,
                embedding=doc.embedding,
                id=doc.id,
            ))

        return {"documents": enriched}
```

### Custom Retriever Wrapping an External API

```python
from haystack import component, Document
from typing import List, Optional
import httpx


@component
class ElasticSearchRetriever:
    """Custom retriever that wraps an Elasticsearch instance with custom scoring."""

    def __init__(
        self,
        es_url: str = "http://localhost:9200",
        index_name: str = "documents",
        top_k: int = 10,
        timeout: float = 30.0,
    ):
        self.es_url = es_url
        self.index_name = index_name
        self.top_k = top_k
        self.timeout = timeout
        self.client = None

    def warm_up(self):
        self.client = httpx.Client(base_url=self.es_url, timeout=self.timeout)

    @component.output_types(documents=List[Document])
    def run(self, query: str, top_k: Optional[int] = None) -> dict:
        if self.client is None:
            raise RuntimeError("Component not warmed up")

        k = top_k or self.top_k

        # Elasticsearch query with BM25 + phrase boosting
        body = {
            "size": k,
            "query": {
                "bool": {
                    "should": [
                        {"match": {"content": {"query": query, "boost": 1.0}}},
                        {"match_phrase": {"content": {"query": query, "boost": 3.0, "slop": 2}}},
                    ]
                }
            },
            "_source": ["content", "meta"],
        }

        response = self.client.post(
            f"/{self.index_name}/_search",
            json=body,
        )
        response.raise_for_status()
        data = response.json()

        documents = []
        for hit in data["hits"]["hits"]:
            doc = Document(
                content=hit["_source"]["content"],
                meta=hit["_source"].get("meta", {}),
                score=hit["_score"],
                id=hit["_id"],
            )
            documents.append(doc)

        return {"documents": documents}
```

---

## Hybrid Retrieval with Reranking

The highest-quality RAG pipelines combine multiple retrieval strategies and rerank the merged results:

```python
from haystack import Pipeline
from haystack.components.embedders import SentenceTransformersTextEmbedder
from haystack.components.retrievers.in_memory import (
    InMemoryBM25Retriever,
    InMemoryEmbeddingRetriever,
)
from haystack.components.joiners import DocumentJoiner
from haystack.components.builders import PromptBuilder
from haystack.components.generators import OpenAIGenerator

# Assume document_store is already populated
# from previous indexing pipeline

pipe = Pipeline()

# Query embedding
pipe.add_component(
    "embedder",
    SentenceTransformersTextEmbedder(model="BAAI/bge-small-en-v1.5"),
)

# BM25 retrieval (keyword matching)
pipe.add_component(
    "bm25",
    InMemoryBM25Retriever(document_store=document_store, top_k=10),
)

# Embedding retrieval (semantic matching)
pipe.add_component(
    "embedding",
    InMemoryEmbeddingRetriever(document_store=document_store, top_k=10),
)

# Merge with reciprocal rank fusion
pipe.add_component(
    "joiner",
    DocumentJoiner(join_mode="reciprocal_rank_fusion", top_k=15),
)

# Rerank with cross-encoder
pipe.add_component(
    "reranker",
    CrossEncoderReranker(
        model_name="cross-encoder/ms-marco-MiniLM-L-6-v2",
        top_k=5,
    ),
)

# Prompt and generation
pipe.add_component(
    "prompt",
    PromptBuilder(
        template="""You are a precise technical assistant. Answer the question using ONLY
the provided context passages. Cite passage numbers in square brackets.

{% for doc in documents %}
[{{ loop.index }}] {{ doc.content }}
{% endfor %}

Question: {{ question }}

Answer (with citations):"""
    ),
)
pipe.add_component(
    "llm",
    OpenAIGenerator(
        model="gpt-4o-mini",
        generation_kwargs={"temperature": 0.0, "max_tokens": 500},
    ),
)

# Connect the graph
pipe.connect("embedder.embedding", "embedding.query_embedding")
pipe.connect("bm25.documents", "joiner.documents")
pipe.connect("embedding.documents", "joiner.documents")
pipe.connect("joiner.documents", "reranker.documents")
pipe.connect("reranker.documents", "prompt.documents")
pipe.connect("prompt.prompt", "llm.prompt")

# Run
result = pipe.run(
    {
        "embedder": {"text": "How does hybrid search work?"},
        "bm25": {"query": "How does hybrid search work?"},
        "reranker": {"query": "How does hybrid search work?"},
        "prompt": {"question": "How does hybrid search work?"},
    }
)
```

### Reciprocal Rank Fusion Explained

RRF merges ranked lists by assigning scores based on rank position:

```
RRF_score(doc) = sum(1 / (k + rank_i)) for each list i containing doc
```

where `k` is a constant (default 60). This naturally balances results from different retrieval methods without needing to normalize scores.

```python
# Haystack's DocumentJoiner supports three modes:
joiner_concatenate = DocumentJoiner(join_mode="concatenate")       # append all, deduplicate
joiner_merge = DocumentJoiner(join_mode="merge")                    # average scores
joiner_rrf = DocumentJoiner(join_mode="reciprocal_rank_fusion")     # RRF (recommended)
```

---

## Evaluation

### Building an Evaluation Pipeline

Haystack integrates with evaluation frameworks. Here is a complete evaluation setup:

```python
from haystack import Pipeline
from haystack.components.evaluators import (
    ContextRelevanceEvaluator,
    FaithfulnessEvaluator,
    SASEvaluator,
)


def evaluate_rag_pipeline(
    rag_pipeline: Pipeline,
    questions: list[str],
    ground_truth_answers: list[str],
    ground_truth_contexts: list[list[str]],
) -> dict:
    """Evaluate a RAG pipeline on a test dataset."""

    # Run RAG pipeline on all questions
    predictions = []
    retrieved_contexts = []

    for question in questions:
        result = rag_pipeline.run(
            {
                "embedder": {"text": question},
                "prompt": {"question": question},
            }
        )
        predictions.append(result["llm"]["replies"][0])
        retrieved_contexts.append(
            [doc.content for doc in result["retriever"]["documents"]]
        )

    # Evaluate faithfulness: is the answer grounded in the retrieved context?
    faithfulness_eval = FaithfulnessEvaluator()
    faithfulness_results = faithfulness_eval.run(
        questions=questions,
        contexts=retrieved_contexts,
        predicted_answers=predictions,
    )

    # Evaluate context relevance: are retrieved passages relevant to the question?
    context_eval = ContextRelevanceEvaluator()
    context_results = context_eval.run(
        questions=questions,
        contexts=retrieved_contexts,
    )

    # Evaluate answer similarity: does the answer match the ground truth?
    sas_eval = SASEvaluator(model="cross-encoder/stsb-roberta-large")
    sas_eval.warm_up()
    sas_results = sas_eval.run(
        predicted_answers=predictions,
        ground_truth_answers=ground_truth_answers,
    )

    return {
        "faithfulness": faithfulness_results,
        "context_relevance": context_results,
        "answer_similarity": sas_results,
        "predictions": predictions,
        "contexts": retrieved_contexts,
    }
```

### Custom Evaluation Metrics

```python
from haystack import component
from typing import List


@component
class RetrievalRecallEvaluator:
    """Measures what fraction of ground-truth context passages were retrieved."""

    @component.output_types(
        individual_scores=List[float],
        score=float,
    )
    def run(
        self,
        retrieved_contexts: List[List[str]],
        ground_truth_contexts: List[List[str]],
    ) -> dict:
        scores = []

        for retrieved, ground_truth in zip(retrieved_contexts, ground_truth_contexts):
            if not ground_truth:
                scores.append(1.0)
                continue

            # Check how many ground truth passages were retrieved
            found = 0
            for gt in ground_truth:
                # Fuzzy match: check if the ground truth content appears in any retrieved doc
                for ret in retrieved:
                    if gt.lower().strip() in ret.lower() or ret.lower().strip() in gt.lower():
                        found += 1
                        break

            scores.append(found / len(ground_truth))

        return {
            "individual_scores": scores,
            "score": sum(scores) / len(scores) if scores else 0.0,
        }


@component
class AnswerCompletenessEvaluator:
    """Uses an LLM to judge if the answer addresses all parts of the question."""

    def __init__(self, model: str = "gpt-4o-mini"):
        self.model = model
        self.generator = None

    def warm_up(self):
        from haystack.components.generators import OpenAIGenerator
        self.generator = OpenAIGenerator(
            model=self.model,
            generation_kwargs={"temperature": 0.0, "max_tokens": 100},
        )

    @component.output_types(
        individual_scores=List[float],
        score=float,
    )
    def run(
        self,
        questions: List[str],
        predicted_answers: List[str],
    ) -> dict:
        scores = []

        for question, answer in zip(questions, predicted_answers):
            prompt = f"""Rate from 0.0 to 1.0 how completely this answer addresses the question.
Return ONLY a number between 0.0 and 1.0.

Question: {question}
Answer: {answer}

Score:"""
            result = self.generator.run(prompt=prompt)
            try:
                score = float(result["replies"][0].strip())
                score = max(0.0, min(1.0, score))
            except (ValueError, IndexError):
                score = 0.0
            scores.append(score)

        return {
            "individual_scores": scores,
            "score": sum(scores) / len(scores) if scores else 0.0,
        }
```

### Running a Full Evaluation

```python
# Test dataset
test_questions = [
    "What is Haystack's pipeline architecture?",
    "How do you create a custom component?",
    "What document stores does Haystack support?",
]

test_answers = [
    "Haystack 2.x uses a directed acyclic graph (DAG) architecture where components are nodes connected by typed input/output sockets.",
    "Create a class decorated with @component that implements a run() method with @component.output_types() declaring its outputs.",
    "Haystack supports InMemoryDocumentStore, Elasticsearch, Qdrant, pgvector, Pinecone, and Weaviate.",
]

test_contexts = [
    ["Haystack 2.x uses a DAG pipeline architecture with typed sockets."],
    ["The @component decorator and run() method define a Haystack component."],
    ["Haystack integrates with Elasticsearch, Qdrant, pgvector, and Pinecone."],
]

results = evaluate_rag_pipeline(
    rag_pipeline=rag_pipeline,
    questions=test_questions,
    ground_truth_answers=test_answers,
    ground_truth_contexts=test_contexts,
)

print(f"Faithfulness: {results['faithfulness']['score']:.3f}")
print(f"Context relevance: {results['context_relevance']['score']:.3f}")
print(f"Answer similarity: {results['answer_similarity']['score']:.3f}")
```

---

## Advanced Pipeline Patterns

### Conditional Routing Based on Query Type

```python
from haystack import component, Pipeline
from typing import List


@component
class QueryClassifier:
    """Classifies queries into types and routes to appropriate handlers."""

    def __init__(self, model: str = "gpt-4o-mini"):
        self.model = model
        self.generator = None

    def warm_up(self):
        from haystack.components.generators import OpenAIGenerator
        self.generator = OpenAIGenerator(
            model=self.model,
            generation_kwargs={"temperature": 0.0, "max_tokens": 20},
        )

    @component.output_types(
        factual=str,
        analytical=str,
        procedural=str,
    )
    def run(self, query: str) -> dict:
        prompt = f"""Classify this query into exactly one category.
Reply with ONLY the category name.

Categories:
- factual: questions about facts, definitions, or specific information
- analytical: questions requiring comparison, analysis, or reasoning
- procedural: questions about how to do something, step-by-step instructions

Query: {query}
Category:"""

        result = self.generator.run(prompt=prompt)
        category = result["replies"][0].strip().lower()

        # Route to the correct output
        output = {}
        if "procedural" in category:
            output["procedural"] = query
        elif "analytical" in category:
            output["analytical"] = query
        else:
            output["factual"] = query

        return output
```

### Pipeline with Caching

```python
from haystack import component, Document
from typing import List, Optional
import hashlib
import json
import time


@component
class CachedRetriever:
    """Wraps a retriever with an in-memory cache for repeated queries."""

    def __init__(self, retriever, cache_ttl: int = 3600):
        self.retriever = retriever
        self.cache_ttl = cache_ttl
        self._cache: dict[str, tuple[float, list]] = {}

    def warm_up(self):
        if hasattr(self.retriever, "warm_up"):
            self.retriever.warm_up()

    @component.output_types(documents=List[Document])
    def run(self, query: str, top_k: Optional[int] = None) -> dict:
        cache_key = hashlib.sha256(
            f"{query}:{top_k}".encode()
        ).hexdigest()

        # Check cache
        if cache_key in self._cache:
            timestamp, docs = self._cache[cache_key]
            if time.time() - timestamp < self.cache_ttl:
                return {"documents": docs}

        # Cache miss: run the actual retriever
        result = self.retriever.run(query=query, top_k=top_k)
        documents = result["documents"]

        # Store in cache
        self._cache[cache_key] = (time.time(), documents)

        # Evict old entries
        now = time.time()
        expired = [k for k, (ts, _) in self._cache.items() if now - ts > self.cache_ttl]
        for k in expired:
            del self._cache[k]

        return {"documents": documents}
```

### Streaming Responses

```python
from haystack.components.generators import OpenAIGenerator
from haystack.dataclasses import StreamingChunk


def stream_rag_response(pipeline, question: str):
    """Stream the RAG response token by token."""

    chunks = []

    def streaming_callback(chunk: StreamingChunk):
        print(chunk.content, end="", flush=True)
        chunks.append(chunk.content)

    # Create a generator with streaming
    streaming_llm = OpenAIGenerator(
        model="gpt-4o-mini",
        streaming_callback=streaming_callback,
        generation_kwargs={"temperature": 0.0},
    )

    # Replace the LLM in the pipeline
    pipeline.remove_component("llm")
    pipeline.add_component("llm", streaming_llm)
    pipeline.connect("prompt.prompt", "llm.prompt")

    # Run - output streams via callback
    result = pipeline.run(
        {
            "embedder": {"text": question},
            "prompt": {"question": question},
        }
    )

    full_response = "".join(chunks)
    return full_response
```

---

## Performance Optimization

### Batch Processing for Indexing

```python
from haystack import Document
from typing import List


def batch_index(
    documents: List[Document],
    pipeline,
    batch_size: int = 100,
):
    """Index documents in batches to manage memory usage."""
    total = len(documents)
    indexed = 0

    for i in range(0, total, batch_size):
        batch = documents[i : i + batch_size]
        result = pipeline.run({"cleaner": {"documents": batch}})
        indexed += result["writer"]["documents_written"]
        print(f"Indexed {indexed}/{total} documents")

    return indexed
```

### Parallel Embedding with Multiple Workers

```python
from haystack.components.embedders import SentenceTransformersDocumentEmbedder

# Use multiple workers for CPU-bound embedding
embedder = SentenceTransformersDocumentEmbedder(
    model="BAAI/bge-small-en-v1.5",
    batch_size=128,          # larger batches on GPU
    normalize_embeddings=True,
    device="cuda",           # use GPU if available
)
```

### Document Store Tuning

```python
# Qdrant: tune HNSW for your workload
from haystack_integrations.document_stores.qdrant import QdrantDocumentStore

store = QdrantDocumentStore(
    url="http://localhost:6333",
    index="optimized_docs",
    embedding_dim=384,
    hnsw_config={
        "m": 32,              # higher = better recall, more memory
        "ef_construct": 256,  # higher = better index quality, slower build
    },
    optimizers_config={
        "indexing_threshold": 20000,  # switch from plain to HNSW at this count
    },
    quantization_config={
        "scalar": {
            "type": "int8",           # 4x memory reduction
            "quantile": 0.99,
            "always_ram": True,
        }
    },
)
```

---

## Testing Custom Components

```python
import pytest
from haystack import Document
from your_project.components.reranker import CrossEncoderReranker


class TestCrossEncoderReranker:
    def setup_method(self):
        self.reranker = CrossEncoderReranker(
            model_name="cross-encoder/ms-marco-MiniLM-L-6-v2",
            top_k=3,
        )
        self.reranker.warm_up()

    def test_reranks_documents(self):
        docs = [
            Document(content="Completely irrelevant content about cooking"),
            Document(content="Python is a programming language for many tasks"),
            Document(content="Haystack is a framework for building RAG pipelines"),
        ]

        result = self.reranker.run(
            query="What is Haystack?",
            documents=docs,
        )

        # The Haystack document should be ranked first
        assert "Haystack" in result["documents"][0].content

    def test_empty_documents(self):
        result = self.reranker.run(query="test", documents=[])
        assert result["documents"] == []

    def test_top_k_limits_results(self):
        docs = [Document(content=f"Document {i}") for i in range(10)]
        result = self.reranker.run(query="test", documents=docs, top_k=3)
        assert len(result["documents"]) == 3

    def test_score_threshold(self):
        reranker = CrossEncoderReranker(
            model_name="cross-encoder/ms-marco-MiniLM-L-6-v2",
            top_k=10,
            score_threshold=0.5,
        )
        reranker.warm_up()

        docs = [
            Document(content="Highly relevant to Haystack pipelines"),
            Document(content="Totally unrelated content about gardening tips"),
        ]

        result = reranker.run(query="Haystack pipelines", documents=docs)
        # Low-score documents should be filtered out
        for doc in result["documents"]:
            assert doc.score >= 0.5

    def test_serialization(self):
        serialized = self.reranker.to_dict()
        assert serialized["init_parameters"]["model_name"] == "cross-encoder/ms-marco-MiniLM-L-6-v2"
        assert serialized["init_parameters"]["top_k"] == 3

        restored = CrossEncoderReranker.from_dict(serialized)
        assert restored.model_name == self.reranker.model_name
        assert restored.top_k == self.reranker.top_k
```

---

## Common Pitfalls

1. **Not implementing `to_dict`/`from_dict` on custom components**: Without serialization methods, your pipeline cannot be saved to YAML. Always implement both for production components.

2. **Blocking I/O in components**: Network calls in `run()` without timeouts can hang the pipeline. Always set timeouts on HTTP clients and database connections.

3. **Mutating input documents**: Components should create new Document objects rather than modifying inputs in place. Input mutation can cause subtle bugs in branching pipelines where the same document goes to multiple components.

4. **Not handling empty inputs**: Components should gracefully handle empty document lists and empty strings. Check `if not documents: return {"documents": []}` at the top of `run()`.

5. **Ignoring warm_up lifecycle**: Heavy resources (models, database connections) should be initialized in `warm_up()`, not `__init__()`. This allows pipeline serialization without loading models.

6. **Evaluating without a baseline**: Always measure performance before and after adding reranking, hybrid retrieval, or other optimizations. Some "improvements" actually decrease quality for certain query distributions.

---

## References

- Haystack custom components: https://docs.haystack.deepset.ai/docs/custom-components
- Haystack evaluation: https://docs.haystack.deepset.ai/docs/evaluation
- Cross-encoder reranking: https://www.sbert.net/docs/cross_encoder/pretrained_models.html
- Haystack cookbook: https://github.com/deepset-ai/haystack-cookbook
