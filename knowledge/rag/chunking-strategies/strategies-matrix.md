# Chunking Strategies Matrix

## Overview / TL;DR

This document provides a single-reference comparison of all major chunking strategies. Use the matrix to quickly identify which strategy fits your document type, then refer to the code snippets for implementation. Each strategy is rated on complexity, retrieval quality, cost, and appropriate use cases.

---

## Master Comparison Matrix

| Strategy | Best For | Chunk Size Range | Overlap | Complexity | Retrieval Quality | Ingestion Cost | Key Advantage | Key Disadvantage |
|----------|---------|-----------------|---------|-----------|------------------|----------------|---------------|-----------------|
| Fixed-size | Baseline / prototypes | 500-1500 chars | 10-20% | Very Low | Low | Negligible | Simple, fast | Breaks mid-concept |
| Recursive character | General prose | 500-1500 chars | 10-20% | Low | Medium | Negligible | Good default | No semantic awareness |
| Sentence-based | Narratives, transcripts | 3-10 sentences | 1-2 sentences | Low | Medium | Negligible | Never breaks mid-sentence | Uneven chunk sizes |
| Markdown header | Documentation, wikis | Section-length | None (per-section) | Low | High | Negligible | Preserves doc structure | Requires markdown format |
| HTML header | Web pages | Section-length | None | Low | High | Negligible | Preserves heading hierarchy | Requires HTML |
| Code (language-aware) | Source code | Function/class | 5-10 lines | Medium | High | Negligible | Respects AST boundaries | Language-specific setup |
| Semantic | Topic-shifting text | Variable (50-500 sent.) | None (natural breaks) | Medium | High | Medium (embedding calls) | Adapts to content | Unpredictable sizes |
| Hierarchical | Multi-granularity retrieval | Multiple sizes | Parent-child overlap | Medium | Very High | Negligible | Precision + context | Complex storage |
| Table-aware | Documents with tables | Per-table or per-row | Context window | Medium | High (for tabular data) | Negligible | Preserves table structure | Requires table detection |
| Proposition-based | Fact-dense knowledge bases | 5-15 propositions | None | High | Very High | High (LLM per passage) | Maximum precision | Expensive ingestion |

---

## Detailed Strategy Cards

### Fixed-Size Chunking

```
Complexity:       [*----]  Very Low
Retrieval Quality: [**---]  Low
Ingestion Cost:    [*----]  Negligible
Maintenance:       [*----]  None
```

**Implementation**:
```python
from langchain_text_splitters import CharacterTextSplitter

splitter = CharacterTextSplitter(
    separator="\n\n",
    chunk_size=1000,
    chunk_overlap=200,
)
```

**When to use**: Prototyping, baseline comparison, unstructured text with no format.
**When to avoid**: Any document with headings, code, tables, or topical structure.

---

### Recursive Character Splitting

```
Complexity:       [*----]  Low
Retrieval Quality: [***--]  Medium
Ingestion Cost:    [*----]  Negligible
Maintenance:       [*----]  None
```

**Implementation**:
```python
from langchain_text_splitters import RecursiveCharacterTextSplitter

splitter = RecursiveCharacterTextSplitter(
    chunk_size=1000,
    chunk_overlap=200,
    separators=["\n\n", "\n", ". ", " ", ""],
)
```

**When to use**: Default for any text content. Start here and switch only if evaluation shows problems.
**When to avoid**: Code, tables, documents where structure carries meaning.

---

### Sentence-Based Splitting

```
Complexity:       [**---]  Low-Medium
Retrieval Quality: [***--]  Medium
Ingestion Cost:    [*----]  Negligible
Maintenance:       [*----]  None
```

**Implementation**:
```python
import nltk
from nltk.tokenize import sent_tokenize

sentences = sent_tokenize(text)
# Group into chunks of N sentences with M overlap
```

**When to use**: Transcripts, interview text, essays, any text where sentence integrity matters.
**When to avoid**: Highly structured documents (markdown, code).

---

### Markdown Header Splitting

```
Complexity:       [**---]  Low
Retrieval Quality: [****-]  High
Ingestion Cost:    [*----]  Negligible
Maintenance:       [*----]  None
```

**Implementation**:
```python
from langchain_text_splitters import MarkdownHeaderTextSplitter

splitter = MarkdownHeaderTextSplitter(
    headers_to_split_on=[("#", "h1"), ("##", "h2"), ("###", "h3")],
)
```

**When to use**: Any markdown documentation, README files, wikis.
**When to avoid**: Unstructured text without headers. Documents where headers are decorative rather than semantic.

---

### Code Splitting (Language-Aware)

```
Complexity:       [***--]  Medium
Retrieval Quality: [****-]  High
Ingestion Cost:    [*----]  Negligible
Maintenance:       [**---]  Per-language setup
```

**Implementation**:
```python
from langchain_text_splitters import RecursiveCharacterTextSplitter, Language

splitter = RecursiveCharacterTextSplitter.from_language(
    language=Language.PYTHON,
    chunk_size=2000,
    chunk_overlap=200,
)
```

**When to use**: Source code repositories, API references, code tutorials.
**When to avoid**: Natural language text.

---

### Semantic Chunking

```
Complexity:       [***--]  Medium
Retrieval Quality: [****-]  High
Ingestion Cost:    [***--]  Medium (embedding per sentence)
Maintenance:       [**---]  Threshold tuning
```

**Implementation**:
```python
from langchain_experimental.text_splitter import SemanticChunker
from langchain_openai import OpenAIEmbeddings

splitter = SemanticChunker(
    embeddings=OpenAIEmbeddings(model="text-embedding-3-small"),
    breakpoint_threshold_type="percentile",
    breakpoint_threshold_amount=95,
)
```

**When to use**: Long documents with topic shifts not aligned to structural markers (transcripts, essays, reports).
**When to avoid**: Short documents (< 1 page). Documents with clear structural markers (use those instead).

**Cost estimate**: For a 10,000-token document with ~200 sentences:
- Using text-embedding-3-small: ~$0.0004 per document
- Using local model (bge-base): effectively free

---

### Hierarchical Chunking

```
Complexity:       [***--]  Medium
Retrieval Quality: [*****]  Very High
Ingestion Cost:    [*----]  Negligible
Maintenance:       [***--]  Storage complexity
```

**Implementation**:
```python
from llama_index.core.node_parser import HierarchicalNodeParser

parser = HierarchicalNodeParser.from_defaults(
    chunk_sizes=[2048, 512, 128],
)
```

**When to use**: When you need both precise retrieval (small chunks) and rich context for generation (parent chunks). The foundation for parent-document and auto-merging retrieval.
**When to avoid**: Simple Q&A where single-level chunking suffices.

---

### Proposition-Based Chunking

```
Complexity:       [*****]  High
Retrieval Quality: [*****]  Very High
Ingestion Cost:    [*****]  High (LLM per passage)
Maintenance:       [***--]  Prompt tuning
```

**Implementation**:
```python
# Requires LLM call per passage during ingestion
response = client.messages.create(
    model="claude-haiku-4-20250514",
    messages=[{"role": "user", "content": f"Extract atomic propositions from:\n{passage}"}],
)
```

**When to use**: High-value knowledge bases where retrieval precision justifies ingestion cost. Fact-dense documents (encyclopedias, product specs, regulatory text).
**When to avoid**: Large corpora (cost), frequently-updated content (must re-extract), conversational/narrative text.

**Cost estimate**: For a 10,000-token document split into 20 passages:
- Claude Haiku: ~20 API calls, ~$0.10-0.20 per document
- At 10,000 documents: $1,000-2,000

---

## Strategy Selection by Document Type

| Document Type | Primary Strategy | Secondary Strategy | Notes |
|--------------|-----------------|-------------------|-------|
| Technical documentation | Markdown header | Recursive (for long sections) | Chain both: split by heading, then by size |
| API reference | Markdown header | Code splitter (for examples) | Separate code blocks from prose |
| Source code | Code splitter (AST) | Recursive character | AST for functions, recursive for comments |
| Legal contracts | Sentence-based | Table-aware | Clause-level precision is critical |
| Research papers (PDF) | Recursive character | Table-aware | After PDF-to-text extraction |
| Transcripts | Semantic chunking | Sentence-based | Topic shifts are unmarked |
| Product catalogs | Table-aware | Proposition-based | Row-level retrieval for specs |
| Knowledge base articles | Markdown header | Semantic (for long sections) | Structure-first, semantic as fallback |
| Chat logs / emails | Sentence-based | Fixed-size | Short messages, natural boundaries |
| Books / long-form | Semantic chunking | Hierarchical | Multi-level abstraction |

---

## Chunk Size Recommendations by Embedding Model

| Embedding Model | Max Input | Sweet Spot | Why |
|----------------|-----------|-----------|-----|
| text-embedding-3-small | 8,191 tokens | 256-512 tokens | Quality plateaus above 512 |
| text-embedding-3-large | 8,191 tokens | 256-512 tokens | Same family, same sweet spot |
| BAAI/bge-base-en-v1.5 | 512 tokens | 128-384 tokens | Hard limit forces smaller chunks |
| BAAI/bge-large-en-v1.5 | 512 tokens | 128-384 tokens | Same tokenizer |
| voyage-3 | 16,000 tokens | 256-1024 tokens | Handles longer inputs well |
| Cohere embed-v4 | 512 tokens | 128-384 tokens | Optimized for search |
| nomic-embed-text-v1.5 | 8,192 tokens | 256-512 tokens | Open-source, good quality |
| jina-embeddings-v3 | 8,192 tokens | 256-1024 tokens | Long-context capable |

---

## Overlap Recommendations

| Chunk Size (tokens) | Recommended Overlap | Overlap % | Rationale |
|--------------------|-------------------|-----------|-----------|
| 128 | 20 | 15.6% | Small chunks need proportionally more overlap |
| 256 | 30-50 | 12-20% | Standard range |
| 512 | 50-100 | 10-20% | Good balance |
| 1024 | 100-200 | 10-20% | Standard range |
| 2048 | 200-300 | 10-15% | Less overlap needed for large chunks |

**When to use zero overlap**:
- Self-contained records (JSON objects, table rows, log entries)
- Proposition-based chunks (each proposition is independent)
- Hierarchical chunks (parent-child relationship provides overlap)

---

## Performance Impact Summary

Based on typical retrieval benchmarks (BEIR, custom eval sets):

| Strategy Change | Context Precision Impact | Context Recall Impact | Notes |
|----------------|------------------------|---------------------|-------|
| Fixed -> Recursive | +5-10% | +3-5% | Better boundary selection |
| Recursive -> Structure-aware | +10-20% | +5-10% | For structured documents |
| No reranking -> With reranking | +15-25% | +5-10% | Orthogonal to chunking |
| 1024 tokens -> 512 tokens | +5-15% precision | -5-10% recall | Precision/recall tradeoff |
| No overlap -> 20% overlap | +0-3% | +5-15% | Mainly helps recall |
| Text splitting -> Semantic | +5-15% | +3-8% | For topic-shifting documents |
| Single-level -> Hierarchical | +10-20% precision | +5-10% recall | Best of both granularities |

---

## Common Pitfalls

1. **Choosing semantic chunking for structured documents.** If your documents have headers and sections, structure-aware splitting is cheaper and often more effective than semantic chunking.
2. **Using the same strategy for all document types.** A mixed corpus (docs + code + tables) needs a multi-strategy pipeline. Route each document type to the appropriate splitter.
3. **Optimizing chunk size without measuring.** The optimal chunk size depends on your specific queries, embedding model, and documents. Always run an evaluation comparing at least 3-4 sizes.
4. **Ignoring the embedding model's input limit.** Chunks longer than the model's max input are silently truncated, losing information. Always enforce the model's limit.
5. **Over-investing in proposition chunking.** The LLM cost for proposition extraction is significant. Only use it for high-value, stable corpora where the cost is justified by retrieval quality gains.

---

## References

- LangChain text splitters API -- https://python.langchain.com/docs/how_to/#text-splitters
- LlamaIndex node parsers API -- https://docs.llamaindex.ai/en/stable/module_guides/loading/node_parsers/
- MTEB Leaderboard (embedding model input limits and quality) -- https://huggingface.co/spaces/mteb/leaderboard
- Pinecone chunking guide -- https://www.pinecone.io/learn/chunking-strategies/
