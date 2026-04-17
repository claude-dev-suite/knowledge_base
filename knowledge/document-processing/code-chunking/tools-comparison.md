# Code Chunking -- Tools Comparison

## Overview / TL;DR

Code chunking tools range from simple regex-based splitters to full AST parsers. The right choice depends on language coverage requirements, boundary quality needs, and integration complexity. This guide provides systematic comparisons across tree-sitter, LangChain Language splitters, LlamaIndex CodeSplitter, and Python's built-in ast module, with benchmark code for measuring boundary quality and retrieval accuracy.

---

## Tool Comparison Matrix

| Feature | tree-sitter (custom) | LangChain Language | LlamaIndex CodeSplitter | Python ast | Regex-based |
|---------|---------------------|-------------------|------------------------|-----------|-------------|
| **Parsing method** | Full AST | Regex separators | tree-sitter AST | Python AST | Pattern match |
| **Languages** | 100+ | 15 | 100+ | Python only | Any |
| **Boundary quality** | Excellent | Good | Excellent | Excellent | Poor |
| **Nesting handling** | Full | None | Full | Full | None |
| **Docstring preservation** | Yes | Unreliable | Yes | Yes | No |
| **Decorator handling** | Yes | No | Yes | Yes | No |
| **Line number tracking** | Yes | No | Yes | Yes | No |
| **Type/name extraction** | Yes | No | Yes | Yes | No |
| **Overlap support** | Custom | Built-in | Built-in | Custom | Custom |
| **Setup complexity** | Medium | Low | Medium | None | Low |
| **Dependencies** | tree-sitter + grammars | langchain | llama-index + tree-sitter | stdlib | None |
| **Error tolerance** | Partial parse | Graceful | Partial parse | Fails on syntax errors | Graceful |

---

## Boundary Quality Analysis

```python
import ast
from dataclasses import dataclass


@dataclass
class BoundaryQuality:
    tool: str
    total_chunks: int
    clean_boundaries: int        # Chunk starts/ends at a definition
    broken_functions: int        # Functions split across chunks
    broken_classes: int          # Classes split mid-body
    orphaned_decorators: int     # Decorators separated from definition
    orphaned_docstrings: int     # Docstrings separated from definition
    score: float                 # 0-1, higher = better


def measure_boundary_quality(
    source_code: str,
    chunks: list[str],
) -> BoundaryQuality:
    """Measure how well chunk boundaries align with code structure."""
    try:
        tree = ast.parse(source_code)
    except SyntaxError:
        return BoundaryQuality(
            tool="unknown", total_chunks=len(chunks),
            clean_boundaries=0, broken_functions=0,
            broken_classes=0, orphaned_decorators=0,
            orphaned_docstrings=0, score=0.0,
        )

    # Find all definition boundaries
    definitions = []
    for node in ast.walk(tree):
        if isinstance(node, (ast.FunctionDef, ast.AsyncFunctionDef, ast.ClassDef)):
            start = node.lineno
            end = node.end_lineno
            definitions.append((start, end, node.name, type(node).__name__))

    # Map chunk text to line ranges
    lines = source_code.split("\n")
    chunk_ranges = []
    search_start = 0
    for chunk in chunks:
        chunk_lines = chunk.strip().split("\n")
        if not chunk_lines:
            continue
        first_line = chunk_lines[0].strip()

        for i in range(search_start, len(lines)):
            if lines[i].strip() == first_line:
                start = i + 1
                end = start + len(chunk_lines) - 1
                chunk_ranges.append((start, end))
                search_start = i + 1
                break

    # Analyze boundaries
    clean = 0
    broken_funcs = 0
    broken_classes = 0

    for def_start, def_end, name, kind in definitions:
        # Check if any chunk boundary falls inside this definition
        for chunk_start, chunk_end in chunk_ranges:
            if chunk_start > def_start and chunk_start < def_end:
                if kind in ("FunctionDef", "AsyncFunctionDef"):
                    broken_funcs += 1
                elif kind == "ClassDef":
                    broken_classes += 1
                break
        else:
            clean += 1

    total_defs = len(definitions)
    score = clean / max(total_defs, 1)

    return BoundaryQuality(
        tool="unknown",
        total_chunks=len(chunks),
        clean_boundaries=clean,
        broken_functions=broken_funcs,
        broken_classes=broken_classes,
        orphaned_decorators=0,
        orphaned_docstrings=0,
        score=round(score, 3),
    )
```

---

## Head-to-Head Comparison

### Test Setup

```python
def compare_chunkers(source_code: str, filepath: str = "test.py") -> dict:
    """Run all chunkers on the same source and compare results."""
    results = {}

    # 1. LangChain Language splitter
    try:
        from langchain_text_splitters import RecursiveCharacterTextSplitter, Language
        lc_splitter = RecursiveCharacterTextSplitter.from_language(
            language=Language.PYTHON,
            chunk_size=2000,
            chunk_overlap=200,
        )
        lc_chunks = lc_splitter.split_text(source_code)
        lc_quality = measure_boundary_quality(source_code, lc_chunks)
        lc_quality.tool = "langchain"
        results["langchain"] = {
            "chunks": len(lc_chunks),
            "avg_size": sum(len(c) for c in lc_chunks) / max(len(lc_chunks), 1),
            "boundary_score": lc_quality.score,
            "broken_functions": lc_quality.broken_functions,
        }
    except ImportError:
        pass

    # 2. LlamaIndex CodeSplitter
    try:
        from llama_index.core.node_parser import CodeSplitter
        from llama_index.core import Document
        li_splitter = CodeSplitter(
            language="python",
            chunk_lines=40,
            chunk_lines_overlap=5,
            max_chars=2000,
        )
        li_docs = [Document(text=source_code, metadata={"source": filepath})]
        li_nodes = li_splitter.get_nodes_from_documents(li_docs)
        li_texts = [n.text for n in li_nodes]
        li_quality = measure_boundary_quality(source_code, li_texts)
        li_quality.tool = "llamaindex"
        results["llamaindex"] = {
            "chunks": len(li_texts),
            "avg_size": sum(len(c) for c in li_texts) / max(len(li_texts), 1),
            "boundary_score": li_quality.score,
            "broken_functions": li_quality.broken_functions,
        }
    except ImportError:
        pass

    # 3. Python ast (custom)
    try:
        ast_chunks = chunk_python_stdlib(source_code, filepath)
        ast_texts = [c["text"] for c in ast_chunks]
        ast_quality = measure_boundary_quality(source_code, ast_texts)
        ast_quality.tool = "python_ast"
        results["python_ast"] = {
            "chunks": len(ast_texts),
            "avg_size": sum(len(c) for c in ast_texts) / max(len(ast_texts), 1),
            "boundary_score": ast_quality.score,
            "broken_functions": ast_quality.broken_functions,
        }
    except Exception:
        pass

    # 4. Naive character splitter (baseline)
    from langchain_text_splitters import CharacterTextSplitter
    naive_splitter = CharacterTextSplitter(
        chunk_size=2000, chunk_overlap=200,
    )
    naive_chunks = naive_splitter.split_text(source_code)
    naive_quality = measure_boundary_quality(source_code, naive_chunks)
    naive_quality.tool = "naive"
    results["naive_baseline"] = {
        "chunks": len(naive_chunks),
        "avg_size": sum(len(c) for c in naive_chunks) / max(len(naive_chunks), 1),
        "boundary_score": naive_quality.score,
        "broken_functions": naive_quality.broken_functions,
    }

    return results
```

### Expected Results

| Metric | tree-sitter | LangChain | LlamaIndex | Python ast | Naive |
|--------|------------|-----------|------------|-----------|-------|
| Boundary score | 0.95-1.0 | 0.70-0.85 | 0.90-1.0 | 0.95-1.0 | 0.30-0.50 |
| Broken functions | 0-1 | 3-8 | 0-2 | 0 | 5-15 |
| Metadata quality | Rich | Minimal | Good | Rich | None |
| Processing speed | Fast | Very Fast | Fast | Very Fast | Very Fast |
| Setup effort | Medium | Minimal | Medium | None | None |

---

## Language Coverage Comparison

| Language | tree-sitter | LangChain | LlamaIndex | Notes |
|----------|------------|-----------|------------|-------|
| Python | Yes | Yes | Yes | All tools support Python |
| JavaScript | Yes | Yes | Yes | |
| TypeScript | Yes | Yes | Yes | |
| Go | Yes | Yes | Yes | |
| Rust | Yes | Yes | Yes | |
| Java | Yes | Yes | Yes | |
| C/C++ | Yes | Yes | Yes | |
| Ruby | Yes | Yes | Yes | |
| PHP | Yes | No | Yes | LangChain lacks PHP |
| Scala | Yes | Yes | Yes | |
| Swift | Yes | Yes | Yes | |
| Kotlin | Yes | No | Yes | |
| Haskell | Yes | No | Yes | |
| Lua | Yes | No | Yes | |
| SQL | Yes | No | Yes (partial) | |
| Shell/Bash | Yes | No | Yes | |
| YAML/TOML | Limited | No | No | Not code per se |

---

## Retrieval Quality Impact

```python
from dataclasses import dataclass


@dataclass
class RetrievalBenchmark:
    chunker: str
    precision_at_5: float
    recall_at_5: float
    mrr: float          # Mean Reciprocal Rank


def benchmark_retrieval(
    source_files: dict[str, str],
    queries: list[dict],
    chunker_fn,
    embed_fn,
    k: int = 5,
) -> RetrievalBenchmark:
    """Measure retrieval quality for a chunking strategy."""
    import numpy as np

    # Chunk all files
    all_chunks = []
    for filepath, source in source_files.items():
        chunks = chunker_fn(source, filepath)
        all_chunks.extend(chunks)

    # Embed all chunks
    chunk_texts = [c["text"] for c in all_chunks]
    chunk_embeddings = embed_fn(chunk_texts)

    # Evaluate each query
    precisions = []
    recalls = []
    reciprocal_ranks = []

    for query in queries:
        query_text = query["query"]
        expected_files = set(query["expected_files"])
        expected_functions = set(query.get("expected_functions", []))

        query_embedding = embed_fn([query_text])[0]

        # Compute similarities
        similarities = np.dot(chunk_embeddings, query_embedding)
        top_k_indices = np.argsort(similarities)[-k:][::-1]

        # Check results
        retrieved_files = set()
        retrieved_functions = set()
        first_relevant_rank = None

        for rank, idx in enumerate(top_k_indices):
            meta = all_chunks[idx].get("metadata", {})
            source = meta.get("source", "")
            name = meta.get("name", "")

            retrieved_files.add(source)
            if name:
                retrieved_functions.add(name)

            if source in expected_files and first_relevant_rank is None:
                first_relevant_rank = rank + 1

        precision = len(retrieved_files & expected_files) / k
        recall = len(retrieved_files & expected_files) / max(len(expected_files), 1)
        mrr = 1.0 / first_relevant_rank if first_relevant_rank else 0.0

        precisions.append(precision)
        recalls.append(recall)
        reciprocal_ranks.append(mrr)

    return RetrievalBenchmark(
        chunker="unknown",
        precision_at_5=round(np.mean(precisions), 3),
        recall_at_5=round(np.mean(recalls), 3),
        mrr=round(np.mean(reciprocal_ranks), 3),
    )
```

---

## Decision Framework

```
What are your requirements?
  |
  +---> Python only
  |     --> Python ast module (zero dependencies, best metadata)
  |
  +---> Multi-language (2-5 languages)
  |     |
  |     +---> Need rich metadata (line numbers, names, docstrings)
  |     |     --> tree-sitter custom + LlamaIndex CodeSplitter
  |     |
  |     +---> Quick integration, metadata not critical
  |           --> LangChain Language splitters
  |
  +---> Many languages (10+)
  |     --> tree-sitter custom (widest coverage)
  |
  +---> Just need "good enough" splitting
        --> LangChain RecursiveCharacterTextSplitter.from_language()
```

---

## Common Pitfalls

1. **Evaluating chunkers by chunk count only.** More chunks does not mean better retrieval. Boundary quality and semantic coherence matter more.
2. **Not testing retrieval quality.** A chunker with perfect boundaries but wrong granularity may perform worse than a good-enough splitter.
3. **Ignoring syntax errors.** Real codebases contain files with syntax errors. AST-based tools must handle parse failures gracefully.
4. **Using the same chunk size for prose and code.** Code chunks need larger sizes (1500-3000 chars) because functions are denser than prose paragraphs.
5. **Not accounting for imports.** A function chunk without its imports is missing critical context for understanding and embeddings.

---

## References

- Tree-sitter documentation -- https://tree-sitter.github.io/tree-sitter/
- LangChain code splitters -- https://python.langchain.com/docs/how_to/code_splitter/
- LlamaIndex CodeSplitter -- https://docs.llamaindex.ai/en/stable/module_guides/loading/node_parsers/
- Python ast module -- https://docs.python.org/3/library/ast.html
