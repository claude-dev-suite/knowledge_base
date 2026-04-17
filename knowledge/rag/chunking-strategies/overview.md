# Chunking Strategies -- Comprehensive Guide

## Overview / TL;DR

Chunking is the process of splitting documents into smaller segments for embedding and retrieval. It is the single most impactful decision in a RAG pipeline because no amount of re-ranking, query transformation, or model quality can recover information that was destroyed by poor chunking. This guide covers every major chunking strategy with production-ready code, explains when to use each one, and provides a decision framework for choosing the right approach based on document type and use case.

---

## Why Chunking Matters

Embedding models encode meaning into fixed-dimensional vectors. When a chunk mixes unrelated topics, the embedding becomes a blurry average that matches nothing well. When a chunk is too small, it lacks context and produces noisy matches. The goal is to create chunks that are:

1. **Semantically coherent** -- each chunk covers one idea or topic.
2. **Self-contained** -- a reader (or model) can understand the chunk without surrounding text.
3. **Appropriately sized** -- large enough for meaningful embeddings, small enough for precise retrieval.
4. **Metadata-rich** -- source, section, position, and hierarchy are preserved.

---

## Strategy 1: Fixed-Size Chunking

Split text into chunks of a fixed number of characters or tokens, optionally with overlap.

**How it works**: Walk through the text in steps of `chunk_size - overlap`, slicing at character/token boundaries.

**Best for**: Unstructured text with no clear structure, or as a baseline.

**Avoid when**: Documents have headings, sections, or any structural cues.

```python
from langchain_text_splitters import CharacterTextSplitter

splitter = CharacterTextSplitter(
    separator="\n\n",      # Try paragraph boundaries first
    chunk_size=1000,        # Max characters per chunk
    chunk_overlap=200,      # Characters shared between consecutive chunks
    length_function=len,
)
chunks = splitter.split_text(document_text)
```

**Token-based fixed-size** (more accurate for LLM context budgets):

```python
import tiktoken
from langchain_text_splitters import RecursiveCharacterTextSplitter

tokenizer = tiktoken.encoding_for_model("gpt-4o")

splitter = RecursiveCharacterTextSplitter.from_tiktoken_encoder(
    encoding_name="cl100k_base",
    chunk_size=512,        # Tokens, not characters
    chunk_overlap=50,
)
chunks = splitter.split_text(document_text)
```

**Limitations**: Chunks can break mid-sentence, mid-paragraph, or mid-concept. Two closely related sentences may end up in different chunks with no semantic boundary between them.

---

## Strategy 2: Recursive Character Splitting

Tries a hierarchy of separators in order of preference, falling back to smaller separators only when a chunk would exceed the target size.

**Default separator hierarchy**: `["\n\n", "\n", ". ", " ", ""]`

**How it works**: Split on `\n\n` (paragraphs). If any resulting piece exceeds `chunk_size`, split that piece on `\n` (lines). If still too large, split on `. ` (sentences). Continue down the hierarchy.

**Best for**: General prose, articles, documentation. This is the recommended default for most text.

```python
from langchain_text_splitters import RecursiveCharacterTextSplitter

splitter = RecursiveCharacterTextSplitter(
    chunk_size=1000,
    chunk_overlap=200,
    separators=["\n\n", "\n", ". ", " ", ""],
    length_function=len,
    is_separator_regex=False,
)

# Split raw text
chunks = splitter.split_text(document_text)

# Split LangChain Documents (preserves metadata)
from langchain_core.documents import Document
docs = [Document(page_content=text, metadata={"source": "guide.md"})]
chunk_docs = splitter.split_documents(docs)
```

**Custom separators for specific formats**:

```python
# For markdown-heavy text
md_splitter = RecursiveCharacterTextSplitter(
    chunk_size=1000,
    chunk_overlap=200,
    separators=[
        "\n## ",     # H2 headers
        "\n### ",    # H3 headers
        "\n\n",      # Paragraphs
        "\n",        # Lines
        ". ",        # Sentences
        " ",         # Words
        "",          # Characters
    ],
)

# For code
code_splitter = RecursiveCharacterTextSplitter(
    chunk_size=2000,
    chunk_overlap=200,
    separators=[
        "\nclass ",
        "\ndef ",
        "\n\n",
        "\n",
        " ",
        "",
    ],
)
```

---

## Strategy 3: Sentence-Based Splitting

Split on sentence boundaries to ensure chunks never break mid-sentence. Often combined with a grouping step to merge short sentences into chunks that meet a minimum size.

```python
# Using NLTK
import nltk
nltk.download('punkt_tab', quiet=True)
from nltk.tokenize import sent_tokenize

def sentence_chunk(text: str, max_chunk_tokens: int = 512, overlap_sentences: int = 2):
    """Group sentences into chunks, respecting a token budget."""
    import tiktoken
    enc = tiktoken.encoding_for_model("gpt-4o")
    sentences = sent_tokenize(text)

    chunks = []
    current_chunk = []
    current_tokens = 0

    for sentence in sentences:
        sent_tokens = len(enc.encode(sentence))
        if current_tokens + sent_tokens > max_chunk_tokens and current_chunk:
            chunks.append(" ".join(current_chunk))
            # Keep last N sentences for overlap
            current_chunk = current_chunk[-overlap_sentences:] if overlap_sentences else []
            current_tokens = sum(len(enc.encode(s)) for s in current_chunk)
        current_chunk.append(sentence)
        current_tokens += sent_tokens

    if current_chunk:
        chunks.append(" ".join(current_chunk))

    return chunks
```

**Using spaCy** (better sentence boundary detection for complex text):

```python
import spacy

nlp = spacy.load("en_core_web_sm")

def spacy_sentence_chunk(text: str, max_tokens: int = 512):
    doc = nlp(text)
    sentences = [sent.text.strip() for sent in doc.sents]
    # Group sentences into chunks (same logic as above)
    return group_sentences(sentences, max_tokens)
```

**Best for**: Narrative text, transcripts, essays where sentence integrity matters more than structural boundaries.

---

## Strategy 4: Markdown/HTML Structure-Aware Splitting

Parse the document's structural markup and create chunks that respect heading hierarchies.

### Markdown Splitting

```python
from langchain_text_splitters import MarkdownHeaderTextSplitter

headers_to_split_on = [
    ("#", "h1"),
    ("##", "h2"),
    ("###", "h3"),
    ("####", "h4"),
]

md_splitter = MarkdownHeaderTextSplitter(
    headers_to_split_on=headers_to_split_on,
    strip_headers=False,  # Keep headers in chunk text
)
md_chunks = md_splitter.split_text(markdown_text)

# Each chunk has metadata like:
# {"h1": "User Guide", "h2": "Authentication", "h3": "API Keys"}

# Chain with recursive splitting for long sections
from langchain_text_splitters import RecursiveCharacterTextSplitter
size_splitter = RecursiveCharacterTextSplitter(chunk_size=1000, chunk_overlap=200)
final_chunks = size_splitter.split_documents(md_chunks)
# Heading metadata is preserved through the second split
```

### HTML Splitting

```python
from langchain_text_splitters import HTMLHeaderTextSplitter

headers = [
    ("h1", "Header 1"),
    ("h2", "Header 2"),
    ("h3", "Header 3"),
]

html_splitter = HTMLHeaderTextSplitter(headers_to_split_on=headers)
chunks = html_splitter.split_text(html_content)
```

### Custom Markdown Splitter (more control)

```python
import re
from dataclasses import dataclass


@dataclass
class StructuredChunk:
    text: str
    metadata: dict


def split_markdown_hierarchical(text: str, max_chunk_size: int = 1000) -> list[StructuredChunk]:
    """Split markdown by heading hierarchy with full metadata."""
    lines = text.split("\n")
    chunks = []
    current_text = []
    heading_stack = {}  # {level: heading_text}

    for line in lines:
        heading_match = re.match(r'^(#{1,6})\s+(.+)$', line)
        if heading_match:
            # Save current chunk
            if current_text:
                chunk_text = "\n".join(current_text).strip()
                if chunk_text:
                    chunks.append(StructuredChunk(
                        text=chunk_text,
                        metadata={**heading_stack},
                    ))
                current_text = []

            level = len(heading_match.group(1))
            heading = heading_match.group(2).strip()

            # Update heading stack: clear all deeper levels
            heading_stack = {
                k: v for k, v in heading_stack.items()
                if int(k[1:]) < level
            }
            heading_stack[f"h{level}"] = heading

        current_text.append(line)

    # Don't forget the last chunk
    if current_text:
        chunk_text = "\n".join(current_text).strip()
        if chunk_text:
            chunks.append(StructuredChunk(
                text=chunk_text,
                metadata={**heading_stack},
            ))

    return chunks
```

**Best for**: Documentation, wikis, README files, any content with heading-based structure.

---

## Strategy 5: Code Splitting

Split source code by language-specific boundaries (classes, functions, methods) rather than arbitrary character counts.

```python
from langchain_text_splitters import RecursiveCharacterTextSplitter, Language

# Python
python_splitter = RecursiveCharacterTextSplitter.from_language(
    language=Language.PYTHON,
    chunk_size=2000,
    chunk_overlap=200,
)

# JavaScript / TypeScript
js_splitter = RecursiveCharacterTextSplitter.from_language(
    language=Language.JS,
    chunk_size=2000,
    chunk_overlap=200,
)

# Go
go_splitter = RecursiveCharacterTextSplitter.from_language(
    language=Language.GO,
    chunk_size=2000,
    chunk_overlap=200,
)

python_chunks = python_splitter.split_text(python_code)
```

**LlamaIndex CodeSplitter** (uses tree-sitter for precise AST-based splitting):

```python
from llama_index.core.node_parser import CodeSplitter

splitter = CodeSplitter(
    language="python",
    chunk_lines=40,       # Target lines per chunk
    chunk_lines_overlap=5,
    max_chars=1500,
)
nodes = splitter.get_nodes_from_documents(documents)
```

**Custom AST-based splitter** (for precise function-level chunks):

```python
import ast


def split_python_by_functions(source_code: str, filepath: str = "") -> list[dict]:
    """Split Python code into function/class-level chunks with metadata."""
    tree = ast.parse(source_code)
    lines = source_code.split("\n")
    chunks = []

    for node in ast.walk(tree):
        if isinstance(node, (ast.FunctionDef, ast.AsyncFunctionDef, ast.ClassDef)):
            start_line = node.lineno - 1
            end_line = node.end_lineno
            chunk_lines = lines[start_line:end_line]
            chunk_text = "\n".join(chunk_lines)

            # Extract docstring if present
            docstring = ast.get_docstring(node) or ""

            chunks.append({
                "text": chunk_text,
                "metadata": {
                    "source": filepath,
                    "type": type(node).__name__,
                    "name": node.name,
                    "start_line": node.lineno,
                    "end_line": end_line,
                    "docstring": docstring[:200],
                    "decorators": [
                        ast.dump(d) for d in node.decorator_list
                    ],
                },
            })

    return chunks
```

**Best for**: Codebases, API references, any content where logical units are functions/classes.

---

## Strategy 6: Semantic Chunking

Use embedding similarity between adjacent sentences to detect topic shifts. Sentences with similar embeddings are grouped together; a drop in similarity creates a chunk boundary.

```python
from langchain_experimental.text_splitter import SemanticChunker
from langchain_openai import OpenAIEmbeddings

embeddings = OpenAIEmbeddings(model="text-embedding-3-small")

# Percentile-based threshold: split when similarity drops below the Nth percentile
splitter = SemanticChunker(
    embeddings=embeddings,
    breakpoint_threshold_type="percentile",
    breakpoint_threshold_amount=95,  # Split at top 5% of dissimilarity
)
chunks = splitter.split_text(document_text)
```

**LlamaIndex SemanticSplitterNodeParser**:

```python
from llama_index.core.node_parser import SemanticSplitterNodeParser
from llama_index.embeddings.openai import OpenAIEmbedding

embed_model = OpenAIEmbedding(model_name="text-embedding-3-small")

splitter = SemanticSplitterNodeParser(
    buffer_size=1,            # Sentences to group before comparing
    breakpoint_percentile_threshold=95,
    embed_model=embed_model,
)
nodes = splitter.get_nodes_from_documents(documents)
```

**Custom implementation** (no framework dependency):

```python
import numpy as np
from sentence_transformers import SentenceTransformer
import nltk
nltk.download('punkt_tab', quiet=True)
from nltk.tokenize import sent_tokenize


def semantic_chunk(
    text: str,
    model_name: str = "BAAI/bge-base-en-v1.5",
    threshold_type: str = "percentile",  # "percentile", "stddev", "gradient"
    threshold_value: float = 95.0,
    min_chunk_size: int = 100,
) -> list[str]:
    """Split text into semantically coherent chunks."""
    model = SentenceTransformer(model_name)
    sentences = sent_tokenize(text)

    if len(sentences) <= 1:
        return [text]

    # Embed all sentences
    embeddings = model.encode(sentences, normalize_embeddings=True)

    # Compute cosine similarities between adjacent sentences
    similarities = []
    for i in range(len(embeddings) - 1):
        sim = np.dot(embeddings[i], embeddings[i + 1])
        similarities.append(sim)

    # Determine breakpoints based on threshold type
    breakpoints = _find_breakpoints(similarities, threshold_type, threshold_value)

    # Build chunks
    chunks = []
    current_chunk = [sentences[0]]
    for i in range(1, len(sentences)):
        if (i - 1) in breakpoints and len(" ".join(current_chunk)) >= min_chunk_size:
            chunks.append(" ".join(current_chunk))
            current_chunk = []
        current_chunk.append(sentences[i])

    if current_chunk:
        chunks.append(" ".join(current_chunk))

    return chunks


def _find_breakpoints(
    similarities: list[float],
    threshold_type: str,
    threshold_value: float,
) -> set[int]:
    """Find indices where chunk boundaries should be placed."""
    if threshold_type == "percentile":
        # Split at positions where similarity is below the threshold percentile
        cutoff = np.percentile(similarities, 100 - threshold_value)
        return {i for i, s in enumerate(similarities) if s < cutoff}

    elif threshold_type == "stddev":
        # Split where similarity drops more than N standard deviations below mean
        mean = np.mean(similarities)
        std = np.std(similarities)
        cutoff = mean - threshold_value * std
        return {i for i, s in enumerate(similarities) if s < cutoff}

    elif threshold_type == "gradient":
        # Split at large negative changes in similarity (steepest drops)
        gradients = np.diff(similarities)
        cutoff = np.percentile(gradients, 100 - threshold_value)
        # Return the index after the gradient drop
        return {i + 1 for i, g in enumerate(gradients) if g < cutoff}

    else:
        raise ValueError(f"Unknown threshold type: {threshold_type}")
```

**Best for**: Transcripts, long-form essays, documents where topic shifts do not align with structural markers.

**Trade-offs**: Requires embedding every sentence during chunking (cost + latency). Chunk sizes vary unpredictably. Results are sensitive to the threshold parameter.

---

## Strategy 7: Hierarchical Chunking

Create multiple chunk sizes from the same document, forming a tree structure. Retrieve leaf nodes for precision, return parent nodes for context.

```python
from llama_index.core.node_parser import HierarchicalNodeParser, SentenceSplitter

# Create a hierarchy: large (2048) -> medium (512) -> small (128)
hierarchical_parser = HierarchicalNodeParser.from_defaults(
    chunk_sizes=[2048, 512, 128],
)
nodes = hierarchical_parser.get_nodes_from_documents(documents)

# Each node has parent/child relationships:
# - node.parent_node (reference to larger chunk)
# - node.child_nodes (references to smaller chunks)
# - node.relationships (all relationships)
```

This strategy is the foundation for parent-document retrieval and auto-merging retrieval. See the `advanced-retrieval/` KB entries for detailed implementations.

**Best for**: When you need both precision (small chunks match specific queries) and context (the LLM needs surrounding information to generate good answers).

---

## Strategy 8: Table-Aware Chunking

Standard text splitters destroy tabular data. Tables need special handling.

```python
import re
from dataclasses import dataclass


@dataclass
class TableChunk:
    table_text: str
    caption: str
    context_before: str
    metadata: dict


def extract_and_chunk_tables(markdown_text: str) -> tuple[list[str], list[TableChunk]]:
    """Separate tables from text, chunk each independently."""
    # Match markdown tables (lines starting with |)
    table_pattern = r'(\n\|[^\n]+\|(?:\n\|[^\n]+\|)+)'

    tables = []
    text_parts = []
    last_end = 0

    for match in re.finditer(table_pattern, markdown_text):
        # Text before table
        text_before = markdown_text[last_end:match.start()].strip()
        if text_before:
            text_parts.append(text_before)

        # Find caption (usually the line right before the table)
        lines_before = markdown_text[:match.start()].rstrip().split("\n")
        caption = lines_before[-1] if lines_before else ""

        tables.append(TableChunk(
            table_text=match.group(0).strip(),
            caption=caption,
            context_before=text_before[-200:] if text_before else "",
            metadata={"type": "table", "caption": caption},
        ))
        last_end = match.end()

    # Remaining text after last table
    remaining = markdown_text[last_end:].strip()
    if remaining:
        text_parts.append(remaining)

    return text_parts, tables


def table_to_row_chunks(table: TableChunk) -> list[dict]:
    """Convert a table into one chunk per row for fine-grained retrieval."""
    lines = table.table_text.strip().split("\n")
    if len(lines) < 3:  # Need header + separator + at least one row
        return [{"text": table.table_text, "metadata": table.metadata}]

    header = lines[0]
    rows = [line for line in lines[2:] if line.strip()]  # Skip separator

    chunks = []
    for row in rows:
        chunk_text = f"{table.caption}\n{header}\n{row}"
        chunks.append({
            "text": chunk_text,
            "metadata": {**table.metadata, "row_context": table.context_before},
        })

    return chunks
```

---

## Strategy 9: Sliding Window with Propositions

Convert each passage into atomic propositions (single facts), then group propositions into chunks. Based on the "Dense X Retrieval" paper (Chen et al., 2023).

```python
import anthropic

client = anthropic.Anthropic()


def extract_propositions(text: str) -> list[str]:
    """Extract atomic propositions from a text passage."""
    response = client.messages.create(
        model="claude-haiku-4-20250514",
        max_tokens=2000,
        messages=[{
            "role": "user",
            "content": (
                "Extract all atomic propositions from the following text. "
                "Each proposition should:\n"
                "1. Express a single fact\n"
                "2. Be self-contained (understandable without the original text)\n"
                "3. Include necessary context (resolve pronouns, expand abbreviations)\n\n"
                "Return one proposition per line, nothing else.\n\n"
                f"Text:\n{text}"
            ),
        }],
    )
    props = response.content[0].text.strip().split("\n")
    return [p.strip() for p in props if p.strip()]


def proposition_chunk(
    document_text: str,
    passage_size: int = 500,
    props_per_chunk: int = 10,
):
    """Create proposition-based chunks from a document."""
    # Step 1: Split into passages
    from langchain_text_splitters import RecursiveCharacterTextSplitter
    splitter = RecursiveCharacterTextSplitter(chunk_size=passage_size, chunk_overlap=50)
    passages = splitter.split_text(document_text)

    # Step 2: Extract propositions from each passage
    all_propositions = []
    for passage in passages:
        props = extract_propositions(passage)
        all_propositions.extend(props)

    # Step 3: Group propositions into chunks
    chunks = []
    for i in range(0, len(all_propositions), props_per_chunk):
        batch = all_propositions[i:i + props_per_chunk]
        chunks.append("\n".join(batch))

    return chunks
```

**Best for**: Knowledge bases where users ask highly specific factual questions. Each proposition chunk precisely matches single-fact queries.

**Trade-off**: Expensive (requires LLM call per passage during ingestion). Best used for high-value corpora where retrieval precision justifies the cost.

---

## Chunk Size Optimization

### Guidelines by Use Case

| Use Case | Chunk Size (tokens) | Overlap | Rationale |
|----------|-------------------|---------|-----------|
| Q&A over documentation | 256-512 | 50-100 | Precise answers need focused chunks |
| Summarization | 1024-2048 | 200-400 | Summaries need broader context |
| Code search | 512-1024 | 100-200 | Function-level granularity |
| Legal/compliance | 256-400 | 50-100 | Clause-level precision |
| Conversational chat | 400-600 | 100-150 | Balance precision and context |
| Multi-hop reasoning | 512-1024 | 150-200 | Enough context for chain-of-thought |

### Embedding Model Sweet Spots

Most embedding models have an optimal input size range:

| Model | Max Tokens | Optimal Range | Notes |
|-------|-----------|---------------|-------|
| text-embedding-3-small | 8191 | 256-512 | Quality degrades above 512 |
| text-embedding-3-large | 8191 | 256-512 | Same family |
| bge-base-en-v1.5 | 512 | 128-384 | Hard limit at 512 |
| bge-large-en-v1.5 | 512 | 128-384 | Same tokenizer |
| voyage-3 | 16000 | 256-1024 | Handles longer chunks well |
| Cohere embed-v4 | 512 | 128-384 | Search-optimized |

### Empirical Optimization

```python
from ragas.metrics import context_precision, context_recall
from ragas import evaluate
from datasets import Dataset


def optimize_chunk_size(
    documents: list,
    eval_dataset: dict,
    chunk_sizes: list[int] = [128, 256, 512, 1024, 2048],
):
    """Test multiple chunk sizes and find the optimal one."""
    from langchain_text_splitters import RecursiveCharacterTextSplitter

    results = []
    for size in chunk_sizes:
        splitter = RecursiveCharacterTextSplitter(
            chunk_size=size,
            chunk_overlap=size // 5,  # 20% overlap
        )
        chunks = splitter.split_documents(documents)

        # Build index and run queries (implementation depends on your setup)
        scores = run_eval(chunks, eval_dataset)
        results.append({
            "chunk_size": size,
            "num_chunks": len(chunks),
            "context_precision": scores["context_precision"],
            "context_recall": scores["context_recall"],
            "f1": 2 * scores["context_precision"] * scores["context_recall"]
                  / (scores["context_precision"] + scores["context_recall"] + 1e-8),
        })
        print(f"chunk_size={size}: {len(chunks)} chunks, "
              f"precision={scores['context_precision']:.3f}, "
              f"recall={scores['context_recall']:.3f}")

    return results
```

---

## Metadata Preservation

Every chunk should carry metadata that enables filtering, citation, and debugging.

```python
from langchain_core.documents import Document


def create_rich_chunk(
    text: str,
    source: str,
    section_hierarchy: dict,
    chunk_index: int,
    total_chunks: int,
) -> Document:
    """Create a chunk with comprehensive metadata."""
    import tiktoken
    enc = tiktoken.encoding_for_model("gpt-4o")

    return Document(
        page_content=text,
        metadata={
            # Source tracking
            "source": source,
            "chunk_index": chunk_index,
            "total_chunks": total_chunks,

            # Section hierarchy
            **section_hierarchy,  # {"h1": "User Guide", "h2": "Auth"}

            # Size info
            "char_count": len(text),
            "token_count": len(enc.encode(text)),

            # Content signals
            "has_code": "```" in text,
            "has_table": "|" in text and "---" in text,
            "has_list": bool(re.search(r'^\s*[-*]\s', text, re.MULTILINE)),

            # Freshness
            "last_updated": "2025-03-15",
        },
    )
```

---

## Common Pitfalls

1. **Using character-based chunk sizes when token budgets matter.** Characters and tokens have a variable ratio (roughly 4 characters per token for English). A 1000-character chunk could be 200-300 tokens depending on content. Use token-based sizing for accuracy.
2. **Zero overlap on narrative text.** Without overlap, information at chunk boundaries is effectively lost. A query matching the last sentence of chunk N and the first sentence of chunk N+1 will not match either chunk well.
3. **Ignoring document structure.** Splitting a markdown file with `CharacterTextSplitter` ignores headings, creating chunks that mix content from different sections. Always use structure-aware splitting for structured documents.
4. **Chunks that are too small (under 50 tokens).** Very short chunks produce noisy embeddings and match too many unrelated queries. Set a minimum chunk size.
5. **Chunks that are too large (over 1000 tokens).** Large chunks dilute the embedding's semantic focus. The embedding becomes a blend of topics that matches nothing precisely.
6. **Not testing chunk quality empirically.** The "right" chunk size varies by document type, embedding model, and query patterns. Always measure with an evaluation dataset.
7. **Stripping all metadata during chunking.** Source, section, and position metadata is essential for citation, filtering, and re-ranking. Preserve it through every splitting step.
8. **Chunking tables as text.** Tables become gibberish when split mid-row. Extract tables separately and chunk them as structured data.

---

## Decision Framework

```
What type of document?
  |
  +---> Markdown/HTML with headings
  |     --> Structure-aware splitting + recursive for long sections
  |
  +---> Source code
  |     --> Language-specific code splitter (AST-based preferred)
  |
  +---> Unstructured prose (articles, essays)
  |     --> Is the text long with topic shifts?
  |         YES --> Semantic chunking
  |         NO  --> Recursive character splitting
  |
  +---> Transcripts / conversations
  |     --> Semantic chunking (topics shift without structural markers)
  |
  +---> Tables / structured data
  |     --> Table-aware extraction + row-level chunks
  |
  +---> Mixed (prose + tables + code)
        --> Multi-strategy: extract tables/code first, chunk prose separately
```

---

## Production Checklist

- [ ] Chunking strategy matches document type (recursive for prose, structure-aware for markdown/HTML, AST for code)
- [ ] Chunk size is tuned to the embedding model's optimal range (typically 256-512 tokens)
- [ ] Overlap is 10-20% of chunk size and aligned to sentence boundaries
- [ ] Metadata includes source, section hierarchy, chunk position, and token count
- [ ] Tables and code blocks are handled separately from prose
- [ ] Minimum chunk size prevents noisy micro-chunks
- [ ] Token-based sizing is used instead of character-based when accuracy matters
- [ ] Evaluation dataset measures retrieval quality across chunk size variations
- [ ] Chunking pipeline is idempotent (re-running produces identical chunks)
- [ ] Re-chunking pipeline exists for parameter changes (requires full re-embedding)

---

## References

- LangChain text splitters documentation -- https://python.langchain.com/docs/how_to/#text-splitters
- LlamaIndex node parsers -- https://docs.llamaindex.ai/en/stable/module_guides/loading/node_parsers/
- Chen et al., "Dense X Retrieval: What Retrieval Granularity Should We Use?" (2023) -- https://arxiv.org/abs/2312.06648
- Greg Kamradt, "5 Levels of Text Splitting" -- https://github.com/FullStackRetrieval-com/RetrievalTutorials
- Pinecone, "Chunking strategies for LLM applications" -- https://www.pinecone.io/learn/chunking-strategies/
