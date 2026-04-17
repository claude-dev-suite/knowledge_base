# DSPy -- Getting Started: Install and First RAG Pipeline

## Overview

This guide walks through installing DSPy, building a complete RAG pipeline from scratch, and running it against a real dataset. By the end you will have a working retrieval-augmented generation system that can be optimized with DSPy's compilers.

---

## Installation

### Basic Install

```bash
# Core DSPy package
pip install dspy

# DSPy with specific provider support
pip install dspy[anthropic]   # for Claude models
pip install dspy[all]          # all optional dependencies
```

### Recommended Project Setup

```bash
mkdir dspy-rag-project && cd dspy-rag-project
python -m venv .venv
source .venv/bin/activate  # Linux/macOS
# .venv\Scripts\activate   # Windows

pip install dspy chromadb sentence-transformers

# Verify installation
python -c "import dspy; print(dspy.__version__)"
```

### Environment Variables

```bash
# For OpenAI models
export OPENAI_API_KEY="sk-..."

# For Anthropic models
export ANTHROPIC_API_KEY="sk-ant-..."

# For local models (no key needed, but start the server)
# ollama serve
# ollama pull llama3.1:8b
```

---

## Minimal RAG Pipeline

The simplest possible DSPy RAG pipeline in under 30 lines:

```python
import dspy
from dspy.retrieve.chromadb_rm import ChromadbRM

# 1. Configure language model
lm = dspy.LM("openai/gpt-4o-mini", temperature=0.0)
dspy.configure(lm=lm)

# 2. Define the RAG signature
class GenerateAnswer(dspy.Signature):
    """Answer the question based on the given context."""
    context: list[str] = dspy.InputField(desc="retrieved passages")
    question: str = dspy.InputField()
    answer: str = dspy.OutputField(desc="concise, factual answer")

# 3. Build the RAG module
class SimpleRAG(dspy.Module):
    def __init__(self, k: int = 3):
        self.retrieve = dspy.Retrieve(k=k)
        self.generate = dspy.ChainOfThought(GenerateAnswer)

    def forward(self, question: str) -> dspy.Prediction:
        context = self.retrieve(question).passages
        answer = self.generate(context=context, question=question)
        return dspy.Prediction(answer=answer.answer, context=context)

# 4. Run it
rag = SimpleRAG(k=5)
result = rag(question="What are the benefits of DSPy over manual prompting?")
print(result.answer)
```

---

## Building the Retrieval Index

Before the RAG module can retrieve, you need documents indexed. Here is a complete example using ChromaDB:

```python
import chromadb
from chromadb.utils import embedding_functions

# Create ChromaDB client and collection
chroma_client = chromadb.PersistentClient(path="./chroma_db")

# Use a local embedding model (no API costs)
ef = embedding_functions.SentenceTransformerEmbeddingFunction(
    model_name="BAAI/bge-small-en-v1.5"
)

collection = chroma_client.get_or_create_collection(
    name="knowledge_base",
    embedding_function=ef,
    metadata={"hnsw:space": "cosine"},
)

# Index documents
documents = [
    "DSPy is a framework for programming language models with composable modules.",
    "DSPy Signatures declare the input and output fields of a module.",
    "DSPy Modules combine Signatures with control flow like loops and conditionals.",
    "Optimizers in DSPy automatically tune prompts and few-shot examples.",
    "BootstrapFewShot generates training examples by running the pipeline on data.",
    "MIPROv2 optimizes both instructions and few-shot examples jointly.",
    "DSPy supports OpenAI, Anthropic, and local models through a unified LM interface.",
    "ChainOfThought adds a reasoning step before the output in a DSPy module.",
    "dspy.Assert and dspy.Suggest add runtime constraints with automatic retries.",
    "DSPy Retrieve integrates with ChromaDB, Pinecone, Qdrant, and Weaviate.",
]

# Add documents with IDs and metadata
collection.add(
    documents=documents,
    ids=[f"doc_{i}" for i in range(len(documents))],
    metadatas=[{"source": "dspy_docs", "chunk_index": i} for i in range(len(documents))],
)

print(f"Indexed {collection.count()} documents")
```

### Connecting DSPy to the Index

```python
import dspy
from dspy.retrieve.chromadb_rm import ChromadbRM

# Create the retriever
retriever = ChromadbRM(
    collection_name="knowledge_base",
    persist_directory="./chroma_db",
    embedding_function="BAAI/bge-small-en-v1.5",
    k=5,
)

# Configure DSPy with LM and retriever
lm = dspy.LM("openai/gpt-4o-mini", temperature=0.0)
dspy.configure(lm=lm, rm=retriever)

# Now dspy.Retrieve() will use ChromaDB
r = dspy.Retrieve(k=3)
results = r("How do DSPy optimizers work?")
for passage in results.passages:
    print(f"  - {passage}")
```

---

## Complete RAG Pipeline with Dataset

Here is a production-style pipeline with training data, evaluation, and basic optimization:

### Step 1: Define the Dataset

```python
import dspy

# Create training examples
# Each example has inputs (question) and labels (answer)
trainset = [
    dspy.Example(
        question="What is a DSPy Signature?",
        answer="A Signature declares the input and output fields of a DSPy module, "
               "replacing hand-written prompts with typed declarations.",
    ).with_inputs("question"),
    dspy.Example(
        question="How does ChainOfThought work in DSPy?",
        answer="ChainOfThought adds an automatic reasoning step before generating "
               "the output, encouraging the language model to think step by step.",
    ).with_inputs("question"),
    dspy.Example(
        question="What is the purpose of DSPy optimizers?",
        answer="Optimizers automatically tune prompts, instructions, and few-shot "
               "examples to maximize a user-defined metric on training data.",
    ).with_inputs("question"),
    dspy.Example(
        question="How does dspy.Retrieve work?",
        answer="dspy.Retrieve is a module that queries the configured retrieval model "
               "to fetch the top-k most relevant passages for a given query.",
    ).with_inputs("question"),
    dspy.Example(
        question="What backends does DSPy support for retrieval?",
        answer="DSPy supports ChromaDB, Pinecone, Qdrant, Weaviate, and ColBERTv2 "
               "as retrieval backends through dedicated retriever modules.",
    ).with_inputs("question"),
    dspy.Example(
        question="What is DSPy Assert?",
        answer="dspy.Assert adds hard runtime constraints to module outputs. If a "
               "constraint fails, DSPy automatically retries with feedback.",
    ).with_inputs("question"),
    dspy.Example(
        question="How do you configure the language model in DSPy?",
        answer="Use dspy.LM() to create a language model instance and dspy.configure(lm=lm) "
               "to set it as the default for all modules.",
    ).with_inputs("question"),
    dspy.Example(
        question="Can DSPy use local models?",
        answer="Yes, DSPy supports local models through Ollama and any OpenAI-compatible "
               "API server like vLLM or llama.cpp.",
    ).with_inputs("question"),
]

# Split into train and dev
devset = trainset[:3]
trainset = trainset[3:]
```

### Step 2: Define the Pipeline

```python
class RAGPipeline(dspy.Module):
    """Production RAG pipeline with retrieval and chain-of-thought answer generation."""

    def __init__(self, k: int = 5):
        self.k = k
        self.retrieve = dspy.Retrieve(k=k)
        self.generate = dspy.ChainOfThought(
            "context, question -> answer"
        )

    def forward(self, question: str) -> dspy.Prediction:
        # Retrieve relevant passages
        retrieved = self.retrieve(question)
        context = retrieved.passages

        # Generate answer with chain-of-thought reasoning
        result = self.generate(
            context=context,
            question=question,
        )

        return dspy.Prediction(
            answer=result.answer,
            context=context,
            reasoning=result.reasoning,
        )
```

### Step 3: Define the Metric

```python
import dspy


def rag_metric(example, prediction, trace=None):
    """Evaluate RAG output: checks answer relevance using an LM judge."""

    # Use an LM-as-judge approach
    judge = dspy.Predict("question, gold_answer, predicted_answer -> score: float")

    result = judge(
        question=example.question,
        gold_answer=example.answer,
        predicted_answer=prediction.answer,
    )

    score = float(result.score)

    # During optimization, return binary pass/fail
    if trace is not None:
        return score >= 0.7

    # During evaluation, return the numeric score
    return score


# Simpler alternative: exact match metric
def exact_match(example, prediction, trace=None):
    gold = example.answer.lower().strip()
    pred = prediction.answer.lower().strip()
    return gold in pred or pred in gold
```

### Step 4: Evaluate the Unoptimized Pipeline

```python
from dspy.evaluate import Evaluate

# Create the unoptimized pipeline
rag = RAGPipeline(k=5)

# Evaluate on dev set
evaluator = Evaluate(
    devset=devset,
    metric=exact_match,
    num_threads=2,
    display_progress=True,
    display_table=3,
)

baseline_score = evaluator(rag)
print(f"Baseline accuracy: {baseline_score:.1f}%")
```

### Step 5: Optimize with BootstrapFewShot

```python
from dspy.teleprompt import BootstrapFewShot

# Create the optimizer
optimizer = BootstrapFewShot(
    metric=exact_match,
    max_bootstrapped_demos=4,  # max few-shot examples to generate
    max_labeled_demos=4,       # max labeled examples to use
    max_rounds=1,              # optimization rounds
)

# Compile (optimize) the pipeline
compiled_rag = optimizer.compile(
    student=RAGPipeline(k=5),
    trainset=trainset,
)

# Evaluate the optimized pipeline
optimized_score = evaluator(compiled_rag)
print(f"Optimized accuracy: {optimized_score:.1f}%")
print(f"Improvement: {optimized_score - baseline_score:+.1f}%")
```

### Step 6: Save and Load the Compiled Pipeline

```python
# Save the optimized pipeline
compiled_rag.save("./optimized_rag.json")

# Load it later
loaded_rag = RAGPipeline(k=5)
loaded_rag.load("./optimized_rag.json")

# Use the loaded pipeline
result = loaded_rag(question="What is a DSPy Signature?")
print(result.answer)
```

---

## Using Different Retrieval Backends

### Qdrant

```python
from qdrant_client import QdrantClient
from dspy.retrieve.qdrant_rm import QdrantRM

# Connect to Qdrant
client = QdrantClient(host="localhost", port=6333)

# Create DSPy retriever
qdrant_rm = QdrantRM(
    qdrant_collection_name="my_documents",
    qdrant_client=client,
    k=5,
)

dspy.configure(lm=lm, rm=qdrant_rm)
```

### Pinecone

```python
from dspy.retrieve.pinecone_rm import PineconeRM

pinecone_rm = PineconeRM(
    pinecone_index_name="my-index",
    pinecone_api_key="your-key",
    pinecone_environment="us-east-1",
    k=5,
)

dspy.configure(lm=lm, rm=pinecone_rm)
```

### Custom Retriever

```python
import dspy


class CustomRetriever(dspy.Retrieve):
    """Custom retriever wrapping any search backend."""

    def __init__(self, search_fn, k: int = 5):
        super().__init__(k=k)
        self.search_fn = search_fn

    def forward(self, query_or_queries, k=None, **kwargs):
        k = k or self.k

        if isinstance(query_or_queries, str):
            queries = [query_or_queries]
        else:
            queries = query_or_queries

        all_passages = []
        for query in queries:
            results = self.search_fn(query, top_k=k)
            passages = [r["text"] for r in results]
            all_passages.extend(passages)

        return dspy.Prediction(passages=all_passages)


# Usage
def my_search(query: str, top_k: int) -> list[dict]:
    """Replace with your actual search logic."""
    # Could be Elasticsearch, Meilisearch, PostgreSQL FTS, etc.
    return [{"text": f"Result for: {query}", "score": 0.9}]

custom_rm = CustomRetriever(search_fn=my_search, k=5)
dspy.configure(lm=lm, rm=custom_rm)
```

---

## Async Support

DSPy 2.6+ supports async modules for concurrent processing:

```python
import dspy
import asyncio


class AsyncRAG(dspy.Module):
    def __init__(self, k: int = 5):
        self.retrieve = dspy.Retrieve(k=k)
        self.generate = dspy.ChainOfThought("context, question -> answer")

    async def aforward(self, question: str) -> dspy.Prediction:
        # Retrieval
        retrieved = self.retrieve(question)

        # Generation
        result = self.generate(
            context=retrieved.passages,
            question=question,
        )

        return dspy.Prediction(answer=result.answer, context=retrieved.passages)


async def batch_process(questions: list[str]) -> list[dspy.Prediction]:
    rag = AsyncRAG(k=5)
    tasks = [rag.aforward(q) for q in questions]
    return await asyncio.gather(*tasks)


# Run batch
questions = [
    "What is DSPy?",
    "How do optimizers work?",
    "What retrieval backends are supported?",
]
results = asyncio.run(batch_process(questions))
for q, r in zip(questions, results):
    print(f"Q: {q}")
    print(f"A: {r.answer}\n")
```

---

## Debugging and Inspection

### Inspect LM Calls

```python
# After running a query, inspect what prompts were sent
lm = dspy.LM("openai/gpt-4o-mini", temperature=0.0)
dspy.configure(lm=lm)

rag = SimpleRAG(k=3)
result = rag(question="What is DSPy?")

# Show the last LM call
lm.inspect_history(n=1)

# Show all calls in the pipeline
lm.inspect_history(n=10)
```

### Trace Module Execution

```python
import dspy

# Enable tracing
with dspy.context(trace=[]):
    result = rag(question="What is DSPy?")

# The trace contains all intermediate results
# Useful for debugging multi-hop pipelines
```

### Logging Configuration

```python
import logging

# Enable DSPy debug logging
logging.basicConfig(level=logging.DEBUG)
dspy_logger = logging.getLogger("dspy")
dspy_logger.setLevel(logging.DEBUG)

# Or just warnings
dspy_logger.setLevel(logging.WARNING)
```

---

## Project Structure for Production

Recommended directory layout for a DSPy RAG project:

```
dspy-rag-project/
  pyproject.toml
  src/
    __init__.py
    config.py              # LM and RM configuration
    signatures.py          # All Signature definitions
    modules/
      __init__.py
      rag.py               # RAG module
      multi_hop.py         # Multi-hop RAG module
      reranker.py          # Reranking module
    metrics/
      __init__.py
      correctness.py       # Answer correctness metric
      faithfulness.py      # Grounding metric
    data/
      __init__.py
      loader.py            # Dataset loading utilities
  compiled/
    rag_optimized.json     # Saved compiled pipeline
  data/
    train.jsonl            # Training examples
    dev.jsonl              # Development examples
    test.jsonl             # Test examples
  scripts/
    index_documents.py     # Document indexing script
    optimize.py            # Run optimization
    evaluate.py            # Run evaluation
    serve.py               # Serve the pipeline via API
  tests/
    test_rag.py
    test_metrics.py
```

### Configuration Module

```python
# src/config.py
import os
import dspy
from dspy.retrieve.chromadb_rm import ChromadbRM


def configure(
    model: str = "openai/gpt-4o-mini",
    temperature: float = 0.0,
    collection: str = "knowledge_base",
    chroma_path: str = "./chroma_db",
    retrieval_k: int = 5,
):
    """Configure DSPy LM and retriever for the project."""
    lm = dspy.LM(
        model,
        temperature=temperature,
        max_tokens=1000,
    )

    rm = ChromadbRM(
        collection_name=collection,
        persist_directory=chroma_path,
        k=retrieval_k,
    )

    dspy.configure(lm=lm, rm=rm)
    return lm, rm


# Usage in scripts:
# from src.config import configure
# configure(model="anthropic/claude-sonnet-4-20250514")
```

---

## Common Pitfalls

1. **Not calling `.with_inputs()` on examples**: Every `dspy.Example` must call `.with_inputs("field1", "field2")` to tell DSPy which fields are inputs vs. labels. Without this, optimization fails silently.

2. **Indexing too few documents**: If your ChromaDB collection has fewer documents than `k`, retrieval returns duplicates or empty results. Always verify `collection.count() >= k`.

3. **Mixing sync and async**: Do not call `forward()` inside an async context. Use `aforward()` for async modules.

4. **Forgetting to save compiled models**: After optimization, always call `compiled_model.save()`. Re-optimizing is expensive and non-deterministic.

5. **Using the wrong embedding model**: If you indexed with `bge-small-en-v1.5`, your retriever must use the same model. Mismatched embeddings produce garbage results.

6. **Not setting `temperature=0`**: For reproducible results during development and evaluation, always use `temperature=0.0`. Random sampling makes metrics unreliable.

---

## Next Steps

- **Optimization**: See `optimization.md` for BootstrapFewShot, MIPROv2, and BootstrapFinetune
- **DSPy Overview**: See `overview.md` for Signatures, Modules, and the programming model
- **Advanced Patterns**: Multi-hop retrieval, ensemble methods, and assertion-driven pipelines

---

## References

- DSPy Quick Start: https://dspy.ai/learn/programming/overview/
- DSPy Retrieval Models: https://dspy.ai/learn/retrieval_model_clients/overview/
- DSPy GitHub Examples: https://github.com/stanfordnlp/dspy/tree/main/examples
