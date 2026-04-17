# Code Chunking -- Production Pipeline

## Overview / TL;DR

A production code chunking pipeline must crawl repositories, detect languages, parse source files into AST-based chunks, enrich metadata, embed chunks, and index them for code search and retrieval. This guide provides a complete pipeline from Git repository to vector store, handling multi-language repos, large files, syntax errors, and incremental updates.

---

## Architecture

```
Git Repository
    |
    v
[1] File Discovery
    |-- walk directory tree
    |-- filter by extension / .gitignore
    |-- detect language per file
    |
    v
[2] Parsing & Chunking
    |-- route to language-specific parser
    |-- extract functions, classes, methods
    |-- attach imports as context
    |-- handle syntax errors gracefully
    |
    v
[3] Metadata Enrichment
    |-- source file, line range, name, type
    |-- content hash for deduplication
    |-- git blame (last modified, author)
    |-- dependency analysis
    |
    v
[4] Embedding
    |-- batch embed chunks
    |-- code-optimized embedding model
    |
    v
[5] Indexing
    |-- upsert to vector store
    |-- maintain file -> chunk mapping
    |-- incremental updates on git diff
```

---

## File Discovery and Language Detection

```python
from pathlib import Path
from dataclasses import dataclass

LANGUAGE_MAP = {
    ".py": "python",
    ".js": "javascript",
    ".jsx": "javascript",
    ".ts": "typescript",
    ".tsx": "typescript",
    ".go": "go",
    ".rs": "rust",
    ".java": "java",
    ".c": "c",
    ".h": "c",
    ".cpp": "cpp",
    ".hpp": "cpp",
    ".rb": "ruby",
    ".php": "php",
    ".scala": "scala",
    ".swift": "swift",
    ".kt": "kotlin",
    ".sh": "bash",
    ".sql": "sql",
    ".lua": "lua",
}

# Files/directories to skip
SKIP_PATTERNS = {
    "node_modules", ".git", "__pycache__", ".venv", "venv",
    "dist", "build", ".next", ".nuxt", "target",
    "vendor", ".tox", ".mypy_cache", ".pytest_cache",
}

# Max file size to process (skip generated/minified files)
MAX_FILE_SIZE_KB = 500


@dataclass
class SourceFile:
    path: str
    language: str
    size_bytes: int
    relative_path: str


def discover_files(
    repo_path: str,
    languages: set[str] | None = None,
) -> list[SourceFile]:
    """Discover source files in a repository."""
    repo = Path(repo_path)
    files = []

    for path in repo.rglob("*"):
        # Skip directories
        if path.is_dir():
            continue

        # Skip ignored directories
        parts = path.relative_to(repo).parts
        if any(p in SKIP_PATTERNS for p in parts):
            continue

        # Check extension
        ext = path.suffix.lower()
        language = LANGUAGE_MAP.get(ext)
        if language is None:
            continue

        if languages and language not in languages:
            continue

        # Skip large files
        size = path.stat().st_size
        if size > MAX_FILE_SIZE_KB * 1024:
            continue

        # Skip empty files
        if size == 0:
            continue

        files.append(SourceFile(
            path=str(path),
            language=language,
            size_bytes=size,
            relative_path=str(path.relative_to(repo)),
        ))

    return sorted(files, key=lambda f: f.relative_path)
```

---

## Multi-Language Chunking Engine

```python
import logging
from dataclasses import dataclass, field

logger = logging.getLogger(__name__)


@dataclass
class CodeChunk:
    text: str
    metadata: dict = field(default_factory=dict)


@dataclass
class ChunkingResult:
    file_path: str
    language: str
    chunks: list[CodeChunk] = field(default_factory=list)
    error: str | None = None


class CodeChunkingPipeline:
    """Multi-language code chunking pipeline."""

    def __init__(
        self,
        max_chunk_lines: int = 60,
        max_chunk_chars: int = 3000,
        include_imports: bool = True,
        split_large_classes: bool = True,
    ):
        self.max_chunk_lines = max_chunk_lines
        self.max_chunk_chars = max_chunk_chars
        self.include_imports = include_imports
        self.split_large_classes = split_large_classes
        self._parsers = {}

    def chunk_file(self, source_file: SourceFile) -> ChunkingResult:
        """Chunk a single source file."""
        try:
            source = Path(source_file.path).read_text(encoding="utf-8", errors="replace")
        except Exception as e:
            return ChunkingResult(
                file_path=source_file.path,
                language=source_file.language,
                error=f"Cannot read file: {e}",
            )

        if not source.strip():
            return ChunkingResult(
                file_path=source_file.path,
                language=source_file.language,
            )

        try:
            if source_file.language == "python":
                chunks = self._chunk_python(source, source_file)
            else:
                chunks = self._chunk_with_treesitter(source, source_file)
        except Exception as e:
            logger.warning(f"AST parsing failed for {source_file.path}: {e}")
            chunks = self._chunk_fallback(source, source_file)

        # Ensure all chunks have enriched metadata
        for chunk in chunks:
            chunk.metadata["source"] = source_file.relative_path
            chunk.metadata["language"] = source_file.language
            chunk.metadata.setdefault("chunk_type", "code")

        return ChunkingResult(
            file_path=source_file.path,
            language=source_file.language,
            chunks=chunks,
        )

    def _chunk_python(self, source: str, sf: SourceFile) -> list[CodeChunk]:
        """Python-specific chunking using ast module."""
        import ast

        tree = ast.parse(source)
        lines = source.split("\n")
        chunks = []

        # Extract imports
        import_lines = []
        for node in ast.iter_child_nodes(tree):
            if isinstance(node, (ast.Import, ast.ImportFrom)):
                import_lines.extend(lines[node.lineno - 1:node.end_lineno])
        import_block = "\n".join(import_lines) + "\n\n" if import_lines else ""

        for node in ast.iter_child_nodes(tree):
            if isinstance(node, (ast.FunctionDef, ast.AsyncFunctionDef)):
                start = node.lineno - 1
                if node.decorator_list:
                    start = node.decorator_list[0].lineno - 1
                end = node.end_lineno

                chunk_text = "\n".join(lines[start:end])
                if self.include_imports:
                    chunk_text = import_block + chunk_text

                chunks.append(CodeChunk(
                    text=chunk_text,
                    metadata={
                        "type": "function",
                        "name": node.name,
                        "start_line": start + 1,
                        "end_line": end,
                        "docstring": (ast.get_docstring(node) or "")[:200],
                    },
                ))

            elif isinstance(node, ast.ClassDef):
                start = node.lineno - 1
                if node.decorator_list:
                    start = node.decorator_list[0].lineno - 1
                end = node.end_lineno
                num_lines = end - start

                if num_lines > self.max_chunk_lines and self.split_large_classes:
                    # Split class into methods
                    class_header = f"class {node.name}:  # methods chunked separately\n"

                    for item in node.body:
                        if isinstance(item, (ast.FunctionDef, ast.AsyncFunctionDef)):
                            m_start = item.lineno - 1
                            if item.decorator_list:
                                m_start = item.decorator_list[0].lineno - 1
                            m_end = item.end_lineno

                            method_text = "\n".join(lines[m_start:m_end])
                            full_text = import_block + class_header + "    " + method_text.replace("\n", "\n    ")

                            chunks.append(CodeChunk(
                                text=full_text,
                                metadata={
                                    "type": "method",
                                    "name": f"{node.name}.{item.name}",
                                    "class_name": node.name,
                                    "start_line": m_start + 1,
                                    "end_line": m_end,
                                },
                            ))
                else:
                    chunk_text = "\n".join(lines[start:end])
                    if self.include_imports:
                        chunk_text = import_block + chunk_text

                    chunks.append(CodeChunk(
                        text=chunk_text,
                        metadata={
                            "type": "class",
                            "name": node.name,
                            "start_line": start + 1,
                            "end_line": end,
                            "docstring": (ast.get_docstring(node) or "")[:200],
                        },
                    ))

        return chunks

    def _chunk_with_treesitter(self, source: str, sf: SourceFile) -> list[CodeChunk]:
        """Generic tree-sitter chunking for non-Python languages."""
        from llama_index.core.node_parser import CodeSplitter
        from llama_index.core import Document

        splitter = CodeSplitter(
            language=sf.language,
            chunk_lines=self.max_chunk_lines,
            chunk_lines_overlap=5,
            max_chars=self.max_chunk_chars,
        )

        doc = Document(text=source, metadata={"source": sf.relative_path})
        nodes = splitter.get_nodes_from_documents([doc])

        return [
            CodeChunk(
                text=node.text,
                metadata={
                    "type": "code_block",
                    "start_line": node.metadata.get("start_line", 0),
                    "end_line": node.metadata.get("end_line", 0),
                },
            )
            for node in nodes
        ]

    def _chunk_fallback(self, source: str, sf: SourceFile) -> list[CodeChunk]:
        """Fallback: LangChain language-aware splitter."""
        from langchain_text_splitters import RecursiveCharacterTextSplitter, Language

        lang_map = {
            "python": Language.PYTHON,
            "javascript": Language.JS,
            "typescript": Language.TS,
            "go": Language.GO,
            "rust": Language.RUST,
            "java": Language.JAVA,
            "ruby": Language.RUBY,
            "scala": Language.SCALA,
            "swift": Language.SWIFT,
        }

        lang = lang_map.get(sf.language)
        if lang:
            splitter = RecursiveCharacterTextSplitter.from_language(
                language=lang,
                chunk_size=self.max_chunk_chars,
                chunk_overlap=200,
            )
        else:
            splitter = RecursiveCharacterTextSplitter(
                chunk_size=self.max_chunk_chars,
                chunk_overlap=200,
            )

        texts = splitter.split_text(source)
        return [
            CodeChunk(
                text=t,
                metadata={"type": "fallback_chunk"},
            )
            for t in texts
        ]

    def chunk_repository(
        self,
        repo_path: str,
        languages: set[str] | None = None,
        progress_callback=None,
    ) -> list[ChunkingResult]:
        """Chunk all source files in a repository."""
        files = discover_files(repo_path, languages)
        logger.info(f"Found {len(files)} source files")

        results = []
        for i, sf in enumerate(files):
            result = self.chunk_file(sf)
            results.append(result)

            if progress_callback:
                progress_callback(i + 1, len(files), sf.relative_path)

        total_chunks = sum(len(r.chunks) for r in results)
        errors = sum(1 for r in results if r.error)
        logger.info(f"Chunked {len(files)} files into {total_chunks} chunks ({errors} errors)")

        return results
```

---

## Embedding and Indexing

```python
import hashlib
from typing import Callable


def embed_and_index(
    results: list[ChunkingResult],
    embed_fn: Callable[[list[str]], list[list[float]]],
    batch_size: int = 100,
) -> list[dict]:
    """Embed code chunks and prepare for vector store upsert."""
    all_records = []

    # Collect all chunks
    chunks_with_meta = []
    for result in results:
        if result.error:
            continue
        for chunk in result.chunks:
            chunk_id = hashlib.sha256(
                f"{result.file_path}:{chunk.metadata.get('start_line', 0)}:{chunk.text[:100]}".encode()
            ).hexdigest()[:24]

            chunks_with_meta.append({
                "id": chunk_id,
                "text": chunk.text,
                "metadata": chunk.metadata,
            })

    # Batch embed
    texts = [c["text"] for c in chunks_with_meta]
    all_embeddings = []

    for i in range(0, len(texts), batch_size):
        batch = texts[i:i + batch_size]
        embeddings = embed_fn(batch)
        all_embeddings.extend(embeddings)

    # Build records for vector store
    for chunk_data, embedding in zip(chunks_with_meta, all_embeddings):
        all_records.append({
            "id": chunk_data["id"],
            "values": embedding,
            "metadata": {
                **chunk_data["metadata"],
                "text": chunk_data["text"][:1000],  # Truncate for metadata storage
            },
        })

    return all_records


def upsert_to_pinecone(records: list[dict], index_name: str):
    """Upsert code chunk embeddings to Pinecone."""
    from pinecone import Pinecone

    pc = Pinecone()
    index = pc.Index(index_name)

    # Batch upsert
    batch_size = 100
    for i in range(0, len(records), batch_size):
        batch = records[i:i + batch_size]
        vectors = [
            {
                "id": r["id"],
                "values": r["values"],
                "metadata": r["metadata"],
            }
            for r in batch
        ]
        index.upsert(vectors=vectors)
```

---

## Incremental Updates

```python
import subprocess
import json
from pathlib import Path


def get_changed_files(repo_path: str, since_commit: str) -> list[str]:
    """Get files changed since a specific commit."""
    result = subprocess.run(
        ["git", "diff", "--name-only", since_commit, "HEAD"],
        cwd=repo_path,
        capture_output=True,
        text=True,
    )

    if result.returncode != 0:
        raise RuntimeError(f"git diff failed: {result.stderr}")

    changed = []
    for line in result.stdout.strip().split("\n"):
        if line.strip():
            full_path = str(Path(repo_path) / line.strip())
            changed.append(full_path)

    return changed


def incremental_update(
    repo_path: str,
    since_commit: str,
    pipeline: CodeChunkingPipeline,
    embed_fn,
    delete_fn,
    upsert_fn,
):
    """Update index with only changed files."""
    changed_files = get_changed_files(repo_path, since_commit)
    logger.info(f"Processing {len(changed_files)} changed files")

    for file_path in changed_files:
        # Delete old chunks for this file
        relative = str(Path(file_path).relative_to(repo_path))
        delete_fn(filter={"source": relative})

        # Re-chunk if file still exists
        if Path(file_path).exists():
            ext = Path(file_path).suffix.lower()
            language = LANGUAGE_MAP.get(ext)
            if language:
                sf = SourceFile(
                    path=file_path,
                    language=language,
                    size_bytes=Path(file_path).stat().st_size,
                    relative_path=relative,
                )
                result = pipeline.chunk_file(sf)
                if result.chunks:
                    records = embed_and_index([result], embed_fn)
                    upsert_fn(records)
```

---

## Common Pitfalls

1. **Not filtering generated files.** `node_modules`, `dist`, `build`, and minified files should be excluded. They add noise and waste embedding budget.
2. **Processing files over 500KB.** Very large files are usually generated (protobuf, migrations). Set a size limit.
3. **Not handling encoding errors.** Some source files use non-UTF-8 encoding. Use `errors="replace"` when reading.
4. **Embedding entire files instead of functions.** File-level chunks are too large for precise retrieval. Function/method level is the right granularity.
5. **Not using code-specific embedding models.** Models like `voyage-code-3` and `code-search-ada-002` embed code better than generic text models.
6. **Skipping incremental updates.** Re-indexing an entire repository on every commit is wasteful. Use git diff to identify changed files.
7. **Not deduplicating chunks.** Copied files and similar functions produce near-identical chunks. Deduplicate using content hashes.

---

## Production Checklist

- [ ] File discovery excludes generated/vendored directories and large files
- [ ] Language detection covers all languages in the repository
- [ ] AST-based chunking is used with fallback to regex-based splitting
- [ ] Large classes are split into method-level chunks
- [ ] Imports are prepended as context
- [ ] Metadata includes file path, language, line range, function/class name
- [ ] Content hashes enable deduplication
- [ ] Syntax errors are caught and files fall back to simpler chunking
- [ ] Code-optimized embedding model is used
- [ ] Incremental updates process only changed files
- [ ] Batch embedding respects API rate limits

---

## References

- Tree-sitter documentation -- https://tree-sitter.github.io/tree-sitter/
- LlamaIndex CodeSplitter -- https://docs.llamaindex.ai/en/stable/module_guides/loading/node_parsers/
- Voyage AI code embeddings -- https://docs.voyageai.com/docs/embeddings
- Pinecone vector database -- https://docs.pinecone.io/
