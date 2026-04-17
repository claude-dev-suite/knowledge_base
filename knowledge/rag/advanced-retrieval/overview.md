# Advanced Retrieval Patterns

## Overview / TL;DR

Standard vector retrieval (embed query, find nearest chunks, pass to LLM) leaves significant quality on the table. Advanced retrieval patterns address specific failure modes: RAPTOR handles multi-level abstraction queries by building a hierarchical summary tree; parent-document retrieval solves the precision-context trade-off by retrieving small chunks but returning their parent documents; auto-merging retrieval dynamically combines leaf chunks into parent chunks when enough evidence accumulates. These techniques can improve retrieval recall by 15-30% over standard approaches, particularly for complex queries that require reasoning across multiple granularity levels. This document provides a landscape overview of all major advanced retrieval patterns, with deep dives available in the sibling articles.

---

## The Landscape

```
Advanced Retrieval Patterns
|
+--- Hierarchical Patterns
|    +--- Parent-Document Retrieval (small search, big context)
|    +--- Auto-Merging / Hierarchical Retrieval (dynamic leaf-to-parent merge)
|    +--- RAPTOR (recursive summarization tree)
|
+--- Multi-Step Patterns
|    +--- Iterative Retrieval (retrieve -> evaluate -> retrieve again)
|    +--- Self-Reflective RAG (SELF-RAG, CRAG)
|    +--- Corrective RAG (evaluate retrieval, web-search fallback)
|
+--- Multi-Index Patterns
|    +--- Multi-Vector Retrieval (multiple embeddings per document)
|    +--- Knowledge Graph + Vector hybrid
|    +--- Multi-Index Fusion (RRF across specialized indexes)
|
+--- Efficiency Patterns
|    +--- Sentence Window Retrieval (small embed, big context)
|    +--- Contextual Compression (compress retrieved context before generation)
|    +--- ColBERT / Late Interaction (token-level similarity)
```

---

## Pattern 1: Parent-Document Retrieval

**Problem**: Small chunks are good for precise retrieval (the embedding is focused) but bad for generation (the LLM needs more context). Large chunks are bad for retrieval (the embedding is diluted) but good for generation.

**Solution**: Index small chunks for retrieval, but return the parent chunk (or full document) for generation.

```
Document -> Split into large "parent" chunks (2000 tokens)
         -> Split parents into small "child" chunks (200 tokens)
         -> Index child chunks in the vector store
         -> Store parent chunks in a document store

At query time:
  1. Search finds relevant child chunks
  2. Look up the parent chunk for each child
  3. Pass parent chunks to the LLM
```

```python
from llama_index.core import VectorStoreIndex, StorageContext
from llama_index.core.node_parser import SentenceSplitter, HierarchicalNodeParser
from llama_index.core.storage.docstore import SimpleDocumentStore
from llama_index.core.retrievers import AutoMergingRetriever


# Create hierarchical chunks
hierarchical_parser = HierarchicalNodeParser.from_defaults(
    chunk_sizes=[2048, 512, 128],  # parent, child, leaf
)
nodes = hierarchical_parser.get_nodes_from_documents(documents)

# Store all nodes in a docstore (for parent lookup)
docstore = SimpleDocumentStore()
docstore.add_documents(nodes)

# Index only leaf nodes in the vector store
leaf_nodes = [n for n in nodes if len(n.child_nodes) == 0]
storage_context = StorageContext.from_defaults(docstore=docstore)
index = VectorStoreIndex(leaf_nodes, storage_context=storage_context)
```

See `parent-document.md` for the complete deep dive.

---

## Pattern 2: Auto-Merging Retrieval

**Problem**: When multiple leaf chunks from the same parent are retrieved, it often means the entire parent section is relevant. Returning individual leaf chunks loses the coherent structure of the parent.

**Solution**: After retrieval, check if enough leaf chunks from the same parent were retrieved. If so, merge them into the parent chunk automatically.

```
Query retrieves: [Leaf_1a, Leaf_1c, Leaf_1d, Leaf_2b, Leaf_3a]
                  ^^^^^^^^^^^^^^^^^^^^^^^^^^^
                  3 out of 4 leaves from Parent_1

Auto-merge triggered for Parent_1 (threshold: 60% of leaves)
Final context: [Parent_1 (merged), Leaf_2b, Leaf_3a]
```

```python
from llama_index.core.retrievers import AutoMergingRetriever

# Configure auto-merging
retriever = AutoMergingRetriever(
    vector_retriever=index.as_retriever(similarity_top_k=12),
    storage_context=storage_context,
    simple_ratio_thresh=0.4,  # Merge when 40%+ of leaves are retrieved
)

nodes = retriever.retrieve("How does the authentication system work?")
```

See `auto-merging.md` for the complete deep dive.

---

## Pattern 3: RAPTOR (Recursive Abstractive Processing for Tree-Organized Retrieval)

**Problem**: Some queries require information at different levels of abstraction. "What is the main theme of this report?" needs a high-level summary, while "What specific metric improved in Q3?" needs a precise detail. Standard flat indexes handle one level well but not both.

**Solution**: Build a tree of summaries. Cluster chunks, summarize each cluster, embed the summaries, then cluster and summarize again. This creates a multi-level tree where leaf nodes are original chunks and internal/root nodes are progressively more abstract summaries.

```
Level 0 (leaves):    C1  C2  C3  C4  C5  C6  C7  C8  C9  C10
                      \  /      \  |  /      \   |   /
Level 1 (summaries):  S1         S2          S3
                        \         |          /
Level 2 (root):            Root Summary

All nodes (C1-C10, S1-S3, Root) are embedded and indexed.
Queries match at the appropriate level of abstraction.
```

See `raptor-paper.md` for the complete deep dive.

---

## Pattern 4: Sentence Window Retrieval

**Problem**: Parent-document retrieval requires a separate document store. Sentence window is a simpler alternative that achieves a similar effect.

**Solution**: Index individual sentences, but store surrounding sentences as metadata. At retrieval time, expand the matched sentence to include its window.

```python
from llama_index.core.node_parser import SentenceWindowNodeParser
from llama_index.core.postprocessor import MetadataReplacementPostProcessor

# Parse with sentence windows
parser = SentenceWindowNodeParser.from_defaults(
    window_size=3,  # 3 sentences on each side
    window_metadata_key="window",
    original_text_metadata_key="original_text",
)
nodes = parser.get_nodes_from_documents(documents)

# Index the nodes (single sentences are indexed)
index = VectorStoreIndex(nodes)

# At query time, replace the node text with the window
query_engine = index.as_query_engine(
    node_postprocessors=[
        MetadataReplacementPostProcessor(target_metadata_key="window"),
    ],
)
```

**Trade-off vs parent-document**: Simpler to implement (no separate docstore) but the context window is a fixed-size window around the sentence, not a semantically coherent parent chunk.

---

## Pattern 5: Multi-Vector Retrieval

**Problem**: A single embedding per chunk captures one aspect of its meaning. Chunks that contain both a question and an answer, or both a concept and an example, may not match queries for either aspect optimally.

**Solution**: Generate multiple embeddings per chunk from different representations (summary, hypothetical questions, key concepts).

```python
import anthropic

client = anthropic.Anthropic()


def create_multi_vector_chunk(chunk_text: str, metadata: dict) -> list[dict]:
    """Create multiple indexed representations for one chunk."""
    entries = []

    # Representation 1: Original text embedding
    entries.append({
        "text_to_embed": chunk_text,
        "text_for_context": chunk_text,
        "metadata": {**metadata, "representation": "original"},
    })

    # Representation 2: Summary embedding
    summary = generate_summary(chunk_text)
    entries.append({
        "text_to_embed": summary,
        "text_for_context": chunk_text,  # Still return original for generation
        "metadata": {**metadata, "representation": "summary"},
    })

    # Representation 3: Hypothetical questions this chunk answers
    questions = generate_questions(chunk_text)
    for q in questions:
        entries.append({
            "text_to_embed": q,
            "text_for_context": chunk_text,
            "metadata": {**metadata, "representation": "question"},
        })

    return entries


def generate_summary(text: str) -> str:
    response = client.messages.create(
        model="claude-haiku-4-20250514",
        max_tokens=100,
        messages=[{
            "role": "user",
            "content": f"Summarize in 1-2 sentences:\n{text}",
        }],
    )
    return response.content[0].text.strip()


def generate_questions(text: str, n: int = 3) -> list[str]:
    response = client.messages.create(
        model="claude-haiku-4-20250514",
        max_tokens=200,
        messages=[{
            "role": "user",
            "content": (
                f"Generate {n} questions that this text answers. "
                f"One per line, nothing else.\n\n{text}"
            ),
        }],
    )
    return [q.strip() for q in response.content[0].text.strip().split("\n") if q.strip()]
```

**LangChain MultiVectorRetriever**:

```python
from langchain.retrievers.multi_vector import MultiVectorRetriever
from langchain.storage import InMemoryByteStore
from langchain_community.vectorstores import Chroma
from langchain_openai import OpenAIEmbeddings

# Vector store for the multiple embeddings
vectorstore = Chroma(
    collection_name="multi_vector",
    embedding_function=OpenAIEmbeddings(),
)

# Byte store for the original documents
docstore = InMemoryByteStore()

retriever = MultiVectorRetriever(
    vectorstore=vectorstore,
    byte_store=docstore,
    id_key="doc_id",
)

# Add original documents to the docstore
# Add summaries/questions to the vectorstore pointing back to doc_id
```

---

## Pattern 6: Self-Reflective RAG (SELF-RAG)

**Problem**: Standard RAG retrieves context for every query, even when retrieval is unnecessary or the retrieved context is irrelevant.

**Solution**: The LLM decides whether retrieval is needed, evaluates the relevance of retrieved documents, and checks its own answer for hallucination.

```python
class SelfReflectiveRAG:
    """Implements the SELF-RAG pattern."""

    def __init__(self, retriever, client):
        self.retriever = retriever
        self.client = client

    def query(self, question: str) -> dict:
        # Step 1: Decide if retrieval is needed
        needs_retrieval = self._check_retrieval_need(question)

        if not needs_retrieval:
            return self._direct_answer(question)

        # Step 2: Retrieve
        chunks = self.retriever.search(question, top_k=5)

        # Step 3: Evaluate relevance of each chunk
        relevant_chunks = [
            c for c in chunks if self._is_relevant(question, c["text"])
        ]

        if not relevant_chunks:
            # No relevant chunks found -- try different approach
            return self._direct_answer(question)

        # Step 4: Generate answer
        answer = self._generate(question, relevant_chunks)

        # Step 5: Check for hallucination
        if not self._is_grounded(answer, relevant_chunks):
            # Regenerate with stricter prompt
            answer = self._generate_strict(question, relevant_chunks)

        return {"answer": answer, "sources": relevant_chunks}

    def _check_retrieval_need(self, question: str) -> bool:
        """LLM decides if retrieval is necessary."""
        response = self.client.messages.create(
            model="claude-haiku-4-20250514",
            max_tokens=10,
            messages=[{
                "role": "user",
                "content": (
                    f"Does this question require looking up external information, "
                    f"or can it be answered from general knowledge? "
                    f"Answer 'retrieve' or 'direct'.\n\nQuestion: {question}"
                ),
            }],
        )
        return "retrieve" in response.content[0].text.lower()

    def _is_relevant(self, question: str, chunk_text: str) -> bool:
        """LLM evaluates chunk relevance."""
        response = self.client.messages.create(
            model="claude-haiku-4-20250514",
            max_tokens=10,
            messages=[{
                "role": "user",
                "content": (
                    f"Is this passage relevant to answering the question? "
                    f"Answer 'yes' or 'no'.\n\n"
                    f"Question: {question}\nPassage: {chunk_text[:500]}"
                ),
            }],
        )
        return "yes" in response.content[0].text.lower()

    def _is_grounded(self, answer: str, chunks: list[dict]) -> bool:
        """Check if the answer is grounded in the retrieved context."""
        context = "\n".join(c["text"] for c in chunks)
        response = self.client.messages.create(
            model="claude-haiku-4-20250514",
            max_tokens=10,
            messages=[{
                "role": "user",
                "content": (
                    f"Is every claim in this answer supported by the context? "
                    f"Answer 'yes' or 'no'.\n\n"
                    f"Context: {context[:2000]}\n\nAnswer: {answer}"
                ),
            }],
        )
        return "yes" in response.content[0].text.lower()
```

---

## Comparison Matrix

| Pattern | Best For | Recall Improvement | Latency Impact | Complexity |
|---------|---------|-------------------|----------------|-----------|
| Parent-Document | Precision + context balance | +10-15% | +0ms retrieval, more LLM tokens | Medium |
| Auto-Merging | Coherent multi-chunk answers | +10-20% | +5-10ms merge logic | Medium |
| RAPTOR | Multi-level abstraction queries | +15-25% | +0ms retrieval (tree is pre-built) | High |
| Sentence Window | Simple precision + context | +5-10% | +0ms | Low |
| Multi-Vector | Diverse query matching | +10-20% | +0ms (multiple entries in same index) | Medium |
| Self-Reflective | Reducing hallucination | +0-5% recall, -20-30% hallucination | +300-600ms (extra LLM calls) | High |

---

## When to Use Each Pattern

```
What is your primary problem?
  |
  +---> Retrieved chunks too small for good generation
  |     --> Parent-Document or Sentence Window
  |         (Parent-Document if you need semantic parent boundaries)
  |         (Sentence Window if you want simplicity)
  |
  +---> Queries at different abstraction levels (summary vs detail)
  |     --> RAPTOR
  |
  +---> Single query misses relevant chunks from different angles
  |     --> Multi-Vector Retrieval (or RAG-Fusion from query-transformations)
  |
  +---> Multiple related chunks should be returned together
  |     --> Auto-Merging Retrieval
  |
  +---> LLM hallucinates despite good retrieval
  |     --> Self-Reflective RAG (evaluate and regenerate)
  |
  +---> Retrieval is sometimes unnecessary (simple queries)
        --> Self-Reflective RAG (decide whether to retrieve)
```

---

## Stacking Patterns

These patterns are composable. Common production stacks:

**Stack 1: High Precision + Context (most common)**
```
Parent-Document + Reranking + Query Rewriting
```

**Stack 2: Maximum Recall**
```
Multi-Query (RAG-Fusion) + Hybrid Search + Reranking + Parent-Document
```

**Stack 3: Complex Reasoning**
```
RAPTOR + Self-Reflective RAG + Sub-Question Decomposition
```

**Stack 4: Production Balanced**
```
Contextual Retrieval + Hybrid Search + Reranking + Sentence Window
```

---

## Common Pitfalls

1. **Applying every pattern at once.** Each pattern adds complexity and latency. Start with the simplest pattern that addresses your primary failure mode and measure.
2. **Using hierarchical retrieval without measuring.** Parent-document and auto-merging are not always better than simple chunking with the right chunk size. Measure on your eval set.
3. **Building RAPTOR for short documents.** RAPTOR's tree construction is expensive and only pays off for corpora with multi-level abstraction needs. For simple Q&A over short docs, it adds complexity without benefit.
4. **Not tuning the auto-merge threshold.** Too low (0.2) merges too aggressively, returning overly broad parent chunks. Too high (0.8) never triggers merging. Start at 0.4 and tune with your eval set.
5. **Forgetting that advanced retrieval does not fix bad chunking.** All hierarchical patterns depend on good initial chunking. Parent-document retrieval cannot help if the parent chunks are poorly defined.

---

## References

- Sarthi et al., "RAPTOR: Recursive Abstractive Processing for Tree-Organized Retrieval" (2024) -- https://arxiv.org/abs/2401.18059
- Asai et al., "Self-RAG: Learning to Retrieve, Generate, and Critique through Self-Reflection" (2023) -- https://arxiv.org/abs/2310.11511
- Yan et al., "Corrective Retrieval Augmented Generation" (CRAG, 2024) -- https://arxiv.org/abs/2401.15884
- LlamaIndex advanced retrieval guide -- https://docs.llamaindex.ai/en/stable/optimizing/advanced_retrieval/advanced_retrieval/
- LangChain parent document retriever -- https://python.langchain.com/docs/how_to/parent_document_retriever/
