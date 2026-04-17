# Markdown Structured Parsing -- Tools Comparison

## Overview / TL;DR

Markdown parsing tools range from simple regex patterns to full AST parsers. The right choice depends on whether you need element-level access, handling of edge cases (nested lists, mixed content), and integration with existing frameworks. This guide compares markdown-it-py, tree-sitter-markdown, regex-based parsing, LangChain's MarkdownHeaderTextSplitter, and Python-Markdown across parsing accuracy, speed, feature coverage, and chunking quality.

---

## Tool Comparison Matrix

| Feature | LangChain MHTS | markdown-it-py | tree-sitter-md | Python-Markdown | Regex |
|---------|---------------|---------------|----------------|----------------|-------|
| **Heading splitting** | Built-in | Token-based | AST nodes | Extensions | Pattern match |
| **Heading hierarchy** | Metadata propagation | Token nesting | Parent-child | Not tracked | Manual |
| **Code block handling** | Pass-through | Fence tokens | Code nodes | Extensions | Pattern match |
| **Table parsing** | Pass-through | Plugin-based | Table nodes | Extension | Pattern match |
| **List parsing** | Pass-through | Token-based | List nodes | Built-in | Pattern match |
| **Frontmatter** | Not handled | Plugin | Not built-in | Extension | Regex |
| **Wiki links** | Not handled | Plugin | Not built-in | Extension | Regex |
| **HTML in markdown** | Pass-through | Parsed | HTML nodes | Parsed | Ignored |
| **GFM support** | N/A | Plugins | Partial | Extension | Partial |
| **Chunking** | Built-in | Manual | Manual | Manual | Manual |
| **Metadata output** | h1/h2/h3/h4 | Token stream | AST | HTML | Manual |
| **Edge case handling** | Good | Excellent | Excellent | Good | Poor |
| **Speed** | Fast | Fast | Fast | Medium | Fastest |
| **Dependencies** | langchain | markdown-it-py | tree-sitter | markdown | None |

---

## Parsing Quality Comparison

### Heading Detection

```python
test_cases = [
    ("# Simple heading", True),
    ("## Heading with `code`", True),
    ("### Heading with **bold**", True),
    ("Not a heading", False),
    ("#NotAHeading", False),           # No space after #
    ("    # Indented heading", False),  # Indented = code block
    ("# ", False),                      # Empty heading
]


def test_heading_detection():
    """Compare heading detection across tools."""
    import re

    results = {}

    # Regex approach
    regex_results = []
    for text, expected in test_cases:
        match = bool(re.match(r'^#{1,6}\s+\S', text))
        regex_results.append(match == expected)
    results["regex"] = sum(regex_results) / len(regex_results)

    # markdown-it-py approach
    from markdown_it import MarkdownIt
    md = MarkdownIt()
    mit_results = []
    for text, expected in test_cases:
        tokens = md.parse(text)
        has_heading = any(t.type == "heading_open" for t in tokens)
        mit_results.append(has_heading == expected)
    results["markdown_it"] = sum(mit_results) / len(mit_results)

    return results
```

### Code Block Preservation

| Scenario | LangChain | markdown-it | tree-sitter | Regex |
|----------|-----------|------------|-------------|-------|
| Simple fenced block | Preserved | Preserved | Preserved | Preserved |
| Nested backticks (`````) | Preserved | Preserved | Preserved | May break |
| Indented code block | Preserved | Preserved | Preserved | Missed |
| Code with heading-like content | Preserved | Preserved | Preserved | May split |
| Code block spanning > chunk_size | May split | N/A | N/A | N/A |

### Table Handling

| Scenario | LangChain | markdown-it | tree-sitter | Regex |
|----------|-----------|------------|-------------|-------|
| Simple table | Preserved | Parsed | Parsed | Captured |
| Table with pipes in cells | Preserved | Parsed | Parsed | May break |
| Table without header separator | Ignored | Plugin | Partial | Missed |
| Multi-line cells | Passed through | Plugin | Partial | Missed |

---

## Speed Benchmarks

```python
import time
import re


def benchmark_parsing(markdown_text: str, iterations: int = 100) -> dict:
    """Benchmark parsing speed across tools."""
    results = {}

    # Regex-based splitting
    start = time.perf_counter()
    for _ in range(iterations):
        sections = re.split(r'\n(?=#{1,6}\s)', markdown_text)
    elapsed = time.perf_counter() - start
    results["regex"] = {
        "total_seconds": round(elapsed, 3),
        "per_iteration_ms": round(elapsed / iterations * 1000, 2),
    }

    # LangChain MarkdownHeaderTextSplitter
    try:
        from langchain_text_splitters import MarkdownHeaderTextSplitter
        headers = [("#", "h1"), ("##", "h2"), ("###", "h3")]
        splitter = MarkdownHeaderTextSplitter(headers_to_split_on=headers)

        start = time.perf_counter()
        for _ in range(iterations):
            splitter.split_text(markdown_text)
        elapsed = time.perf_counter() - start
        results["langchain"] = {
            "total_seconds": round(elapsed, 3),
            "per_iteration_ms": round(elapsed / iterations * 1000, 2),
        }
    except ImportError:
        pass

    # markdown-it-py
    try:
        from markdown_it import MarkdownIt
        md = MarkdownIt()

        start = time.perf_counter()
        for _ in range(iterations):
            md.parse(markdown_text)
        elapsed = time.perf_counter() - start
        results["markdown_it"] = {
            "total_seconds": round(elapsed, 3),
            "per_iteration_ms": round(elapsed / iterations * 1000, 2),
        }
    except ImportError:
        pass

    return results
```

**Typical results (10KB markdown document, 100 iterations)**:

| Tool | Per-iteration (ms) | Relative Speed |
|------|-------------------|---------------|
| Regex | 0.1-0.3 | 1x (fastest) |
| LangChain MHTS | 0.5-1.5 | 3-5x slower |
| markdown-it-py | 0.3-0.8 | 2-3x slower |
| tree-sitter-md | 0.2-0.5 | 1.5-2x slower |
| Python-Markdown | 1.0-3.0 | 5-10x slower |

---

## Chunking Quality Comparison

```python
from dataclasses import dataclass


@dataclass
class ChunkQuality:
    tool: str
    total_chunks: int
    avg_chunk_size: int
    heading_metadata_present: bool
    code_blocks_intact: bool
    tables_intact: bool
    empty_chunks: int


def evaluate_chunking_quality(
    markdown_text: str,
    tool_name: str,
    chunks: list[dict],
) -> ChunkQuality:
    """Evaluate chunking quality metrics."""
    sizes = [len(c.get("text", "")) for c in chunks]
    avg_size = sum(sizes) / max(len(sizes), 1)

    # Check heading metadata
    has_heading_meta = any(
        "h1" in c.get("metadata", {}) or "h2" in c.get("metadata", {})
        for c in chunks
    )

    # Check code block integrity
    original_code_blocks = len(re.findall(r'```\w*\n.*?```', markdown_text, re.DOTALL))
    chunk_code_blocks = sum(
        len(re.findall(r'```\w*\n.*?```', c.get("text", ""), re.DOTALL))
        for c in chunks
    )
    code_intact = chunk_code_blocks >= original_code_blocks * 0.9

    # Check table integrity
    original_tables = len(re.findall(r'\|.+\|\n\|[\s\-:]+\|', markdown_text))
    chunk_tables = sum(
        len(re.findall(r'\|.+\|\n\|[\s\-:]+\|', c.get("text", "")))
        for c in chunks
    )
    tables_intact = chunk_tables >= original_tables * 0.9

    empty = sum(1 for c in chunks if len(c.get("text", "").strip()) < 20)

    return ChunkQuality(
        tool=tool_name,
        total_chunks=len(chunks),
        avg_chunk_size=int(avg_size),
        heading_metadata_present=has_heading_meta,
        code_blocks_intact=code_intact,
        tables_intact=tables_intact,
        empty_chunks=empty,
    )
```

### Expected Results

| Metric | LangChain MHTS + RCS | Custom (heading-aware) | Naive RCS | Regex split |
|--------|---------------------|----------------------|-----------|-------------|
| Heading metadata | Yes | Yes | No | Manual |
| Code blocks intact | ~90% | ~95% | ~70% | ~80% |
| Tables intact | ~85% | ~95% | ~60% | ~75% |
| Empty chunks | 1-3 | 0-2 | 0-1 | 2-5 |
| Avg chunk quality | Good | Excellent | Fair | Fair |

---

## Integration Patterns

### LlamaIndex

```python
from llama_index.core.node_parser import MarkdownNodeParser
from llama_index.core import Document


def chunk_with_llamaindex(markdown_text: str, source: str) -> list:
    """Chunk markdown using LlamaIndex MarkdownNodeParser."""
    parser = MarkdownNodeParser()
    doc = Document(text=markdown_text, metadata={"source": source})
    nodes = parser.get_nodes_from_documents([doc])
    return nodes
```

### Docling

```python
def chunk_with_docling(markdown_text: str) -> list:
    """Chunk markdown using Docling's hierarchical chunker."""
    from docling_core.types import DoclingDocument
    from docling.chunking import HierarchicalChunker

    doc = DoclingDocument.from_markdown(markdown_text)
    chunker = HierarchicalChunker(max_tokens=512)
    chunks = list(chunker.chunk(doc))
    return chunks
```

---

## Decision Framework

```
What is your markdown source?
  |
  +---> Simple documentation (headings + prose)
  |     --> LangChain MarkdownHeaderTextSplitter (easiest, good metadata)
  |
  +---> Code-heavy docs (many code blocks)
  |     --> Custom parser with code block protection
  |
  +---> Obsidian vault (frontmatter + wiki links)
  |     --> Custom parser with frontmatter extraction + link resolution
  |
  +---> Very large corpus (100K+ files)
  |     --> Regex-based splitting (fastest) + manual metadata
  |
  +---> Need full AST (complex nested structures)
  |     --> markdown-it-py or tree-sitter-markdown
  |
  +---> LlamaIndex integration
        --> LlamaIndex MarkdownNodeParser
```

---

## Common Pitfalls

1. **Using LangChain MHTS alone on long sections.** MHTS splits by headings but does not limit chunk size. Always chain with RecursiveCharacterTextSplitter.
2. **Regex splitting inside code blocks.** A heading-like pattern (`# comment`) inside a code block triggers a false split. Use a parser that understands fences.
3. **Not handling ATX vs Setext headings.** Some markdown uses underline-style headings (`====` for H1, `----` for H2). Most tools only handle ATX (`#`) by default.
4. **Ignoring GFM extensions.** GitHub Flavored Markdown adds task lists, autolinks, and strikethrough. Standard parsers may miss these.
5. **Not benchmarking on your actual content.** Documentation, blog posts, and Obsidian notes have different structure patterns. Test on representative samples.

---

## References

- LangChain MarkdownHeaderTextSplitter -- https://python.langchain.com/docs/how_to/markdown_header_metadata_splitter/
- markdown-it-py -- https://github.com/executablebooks/markdown-it-py
- tree-sitter-markdown -- https://github.com/tree-sitter-grammars/tree-sitter-markdown
- Python-Markdown -- https://python-markdown.github.io/
- LlamaIndex MarkdownNodeParser -- https://docs.llamaindex.ai/en/stable/module_guides/loading/node_parsers/
