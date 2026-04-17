# Markdown Structured Parsing -- Production Pipeline

## Overview / TL;DR

A production markdown ingestion pipeline must recursively discover files in vaults and repositories, extract frontmatter metadata, parse heading structure, protect code blocks and tables from splitting, resolve cross-references, and produce chunks with rich hierarchical metadata for RAG retrieval. This guide provides a complete pipeline from vault/repo to indexed chunks.

---

## Architecture

```
Markdown Vault / Repository
    |
    v
[1] File Discovery
    |-- recursive .md scan
    |-- .gitignore / .obsidianignore filtering
    |-- file metadata (path, size, modified date)
    |
    v
[2] Parsing
    |-- frontmatter extraction (YAML)
    |-- heading hierarchy detection
    |-- code block identification
    |-- table identification
    |-- wiki link resolution
    |
    v
[3] Chunking
    |-- split by heading boundaries
    |-- sub-split long sections by size
    |-- protect code blocks and tables
    |-- propagate heading hierarchy as metadata
    |
    v
[4] Metadata Enrichment
    |-- frontmatter fields (title, tags, date, author)
    |-- file path and directory
    |-- heading breadcrumb
    |-- link targets
    |
    v
[5] Output (JSON, vector store)
```

---

## Pipeline Implementation

```python
import logging
import re
import yaml
import time
from dataclasses import dataclass, field
from pathlib import Path

logger = logging.getLogger(__name__)


@dataclass
class MarkdownConfig:
    chunk_size: int = 1000
    chunk_overlap: int = 200
    min_chunk_size: int = 50
    heading_levels: list[int] = field(default_factory=lambda: [1, 2, 3, 4])
    extract_frontmatter: bool = True
    resolve_wiki_links: bool = True
    protect_code_blocks: bool = True
    protect_tables: bool = True
    ignore_patterns: list[str] = field(default_factory=lambda: [
        ".obsidian", "node_modules", ".git", "__pycache__",
        ".trash", ".stversions",
    ])


@dataclass
class MarkdownChunk:
    text: str
    metadata: dict


@dataclass
class PipelineResult:
    total_files: int = 0
    processed: int = 0
    failed: int = 0
    total_chunks: int = 0
    chunks: list[MarkdownChunk] = field(default_factory=list)
    errors: list[dict] = field(default_factory=list)
    processing_seconds: float = 0.0


class MarkdownPipeline:
    """Production pipeline for markdown vault/repository ingestion."""

    def __init__(self, config: MarkdownConfig):
        self.config = config

    def process_vault(self, vault_path: str) -> PipelineResult:
        """Process all markdown files in a vault."""
        start = time.perf_counter()
        vault = Path(vault_path)
        result = PipelineResult()

        # Discover files
        md_files = self._discover_files(vault)
        result.total_files = len(md_files)
        logger.info(f"Found {len(md_files)} markdown files")

        for md_file in md_files:
            try:
                chunks = self._process_file(md_file, vault)
                result.chunks.extend(chunks)
                result.processed += 1
            except Exception as e:
                result.failed += 1
                result.errors.append({
                    "file": str(md_file),
                    "error": str(e),
                })
                logger.warning(f"Failed: {md_file}: {e}")

        result.total_chunks = len(result.chunks)
        result.processing_seconds = time.perf_counter() - start

        logger.info(
            f"Processed {result.processed}/{result.total_files} files, "
            f"{result.total_chunks} chunks in {result.processing_seconds:.1f}s"
        )

        return result

    def _discover_files(self, vault: Path) -> list[Path]:
        """Discover markdown files, respecting ignore patterns."""
        files = []
        for path in sorted(vault.rglob("*.md")):
            parts = path.relative_to(vault).parts
            if any(
                ignore in parts
                for ignore in self.config.ignore_patterns
            ):
                continue
            files.append(path)
        return files

    def _process_file(self, md_file: Path, vault: Path) -> list[MarkdownChunk]:
        """Process a single markdown file into chunks."""
        text = md_file.read_text(encoding="utf-8", errors="replace")
        relative = str(md_file.relative_to(vault))

        # Extract frontmatter
        frontmatter = {}
        body = text
        if self.config.extract_frontmatter:
            frontmatter, body = self._extract_frontmatter(text)

        # Resolve wiki links
        if self.config.resolve_wiki_links:
            body = self._resolve_links(body, md_file, vault)

        # Protect code blocks and tables
        protected_body, restorations = self._protect_elements(body)

        # Split by headings
        sections = self._split_by_headings(protected_body)

        # Create chunks
        chunks = []
        for section in sections:
            section_text = self._restore_elements(section["text"], restorations)

            if len(section_text.strip()) < self.config.min_chunk_size:
                continue

            # Sub-split if too long
            if len(section_text) > self.config.chunk_size:
                sub_chunks = self._size_split(section_text)
            else:
                sub_chunks = [section_text]

            for i, chunk_text in enumerate(sub_chunks):
                if len(chunk_text.strip()) < self.config.min_chunk_size:
                    continue

                metadata = {
                    "source": relative,
                    "directory": str(md_file.parent.relative_to(vault)),
                    **section["heading_hierarchy"],
                    "chunk_index": i,
                }

                # Add frontmatter
                if frontmatter:
                    for key in ("title", "tags", "date", "author", "category", "description"):
                        if key in frontmatter:
                            metadata[f"fm_{key}"] = frontmatter[key]

                chunks.append(MarkdownChunk(
                    text=chunk_text,
                    metadata=metadata,
                ))

        return chunks

    def _extract_frontmatter(self, text: str) -> tuple[dict, str]:
        """Extract YAML frontmatter."""
        match = re.match(r'^---\n(.*?)\n---\n?', text, re.DOTALL)
        if match:
            try:
                fm = yaml.safe_load(match.group(1)) or {}
            except yaml.YAMLError:
                fm = {}
            return fm, text[match.end():]
        return {}, text

    def _resolve_links(self, text: str, current_file: Path, vault: Path) -> str:
        """Resolve wiki-style [[links]]."""
        def _replace(match):
            link = match.group(1)
            if "|" in link:
                target, display = link.split("|", 1)
            else:
                target = display = link

            # Find target file
            target_name = target if target.endswith(".md") else target + ".md"
            for candidate in vault.rglob(target_name):
                rel = candidate.relative_to(vault)
                return f"[{display}]({rel})"

            return display

        return re.sub(r'\[\[([^\]]+)\]\]', _replace, text)

    def _protect_elements(self, text: str) -> tuple[str, dict]:
        """Replace code blocks and tables with placeholders."""
        restorations = {}
        counter = 0

        # Protect fenced code blocks
        if self.config.protect_code_blocks:
            def _replace_code(match):
                nonlocal counter
                placeholder = f"__CODE_BLOCK_{counter}__"
                restorations[placeholder] = match.group(0)
                counter += 1
                return placeholder

            text = re.sub(r'```[\s\S]*?```', _replace_code, text)

        # Protect tables
        if self.config.protect_tables:
            def _replace_table(match):
                nonlocal counter
                placeholder = f"__TABLE_{counter}__"
                restorations[placeholder] = match.group(0)
                counter += 1
                return placeholder

            table_pattern = r'(\|[^\n]+\|\n\|[\s\-:]+\|\n(?:\|[^\n]+\|\n)*)'
            text = re.sub(table_pattern, _replace_table, text)

        return text, restorations

    def _restore_elements(self, text: str, restorations: dict) -> str:
        """Restore protected elements from placeholders."""
        for placeholder, original in restorations.items():
            text = text.replace(placeholder, original)
        return text

    def _split_by_headings(self, text: str) -> list[dict]:
        """Split text at heading boundaries with hierarchy tracking."""
        lines = text.split("\n")
        sections = []
        current_lines = []
        heading_stack = {}

        for line in lines:
            match = re.match(r'^(#{1,6})\s+(.+)$', line)
            if match:
                # Save previous section
                if current_lines:
                    content = "\n".join(current_lines).strip()
                    if content:
                        sections.append({
                            "text": content,
                            "heading_hierarchy": {**heading_stack},
                        })
                    current_lines = []

                level = len(match.group(1))
                heading = match.group(2).strip()

                # Update stack
                heading_stack = {
                    k: v for k, v in heading_stack.items()
                    if int(k[1:]) < level
                }
                heading_stack[f"h{level}"] = heading

            current_lines.append(line)

        # Last section
        if current_lines:
            content = "\n".join(current_lines).strip()
            if content:
                sections.append({
                    "text": content,
                    "heading_hierarchy": {**heading_stack},
                })

        return sections

    def _size_split(self, text: str) -> list[str]:
        """Split long text by size with overlap."""
        from langchain_text_splitters import RecursiveCharacterTextSplitter

        splitter = RecursiveCharacterTextSplitter(
            chunk_size=self.config.chunk_size,
            chunk_overlap=self.config.chunk_overlap,
            separators=["\n\n", "\n", ". ", " ", ""],
        )
        return splitter.split_text(text)
```

---

## Running the Pipeline

```python
import json
from pathlib import Path


def main():
    config = MarkdownConfig(
        chunk_size=1000,
        chunk_overlap=200,
        extract_frontmatter=True,
        resolve_wiki_links=True,
        protect_code_blocks=True,
        protect_tables=True,
    )

    pipeline = MarkdownPipeline(config)

    # Process an Obsidian vault
    result = pipeline.process_vault("/path/to/vault")

    print(f"Files: {result.processed}/{result.total_files}")
    print(f"Chunks: {result.total_chunks}")
    print(f"Time: {result.processing_seconds:.1f}s")
    print(f"Errors: {result.failed}")

    # Save output
    output = [
        {"text": c.text, "metadata": c.metadata}
        for c in result.chunks
    ]
    Path("/data/output/vault_chunks.json").write_text(
        json.dumps(output, indent=2, default=str),
        encoding="utf-8",
    )


if __name__ == "__main__":
    main()
```

---

## Incremental Updates

```python
import os
import json
from pathlib import Path


def get_changed_files(
    vault_path: str,
    state_file: str,
) -> tuple[list[Path], dict]:
    """Detect files changed since last processing."""
    vault = Path(vault_path)
    state_path = Path(state_file)

    # Load previous state
    previous = {}
    if state_path.exists():
        previous = json.loads(state_path.read_text())

    # Scan current files
    current = {}
    changed = []

    for md_file in sorted(vault.rglob("*.md")):
        rel = str(md_file.relative_to(vault))
        mtime = os.path.getmtime(md_file)
        size = md_file.stat().st_size
        current[rel] = {"mtime": mtime, "size": size}

        # Check if changed
        prev = previous.get(rel)
        if prev is None or prev["mtime"] != mtime or prev["size"] != size:
            changed.append(md_file)

    return changed, current


def save_state(state: dict, state_file: str):
    """Save file state for incremental updates."""
    Path(state_file).write_text(json.dumps(state, indent=2))
```

---

## Common Pitfalls

1. **Not protecting code blocks before heading-based splitting.** A `# comment` inside a code block triggers a false heading split.
2. **Not protecting tables.** Size-based sub-splitting can break tables mid-row.
3. **Ignoring frontmatter.** Title, tags, and date from frontmatter are essential for filtering and improving retrieval.
4. **Not tracking file path in metadata.** Chunk citations require knowing which file the content came from.
5. **Processing the same file repeatedly.** Use modification time tracking for incremental updates.
6. **Not filtering .obsidian and other config directories.** Internal Obsidian files, node_modules, and .git should be excluded.

---

## Production Checklist

- [ ] File discovery recursively scans for .md files with ignore patterns
- [ ] YAML frontmatter is extracted and added to chunk metadata
- [ ] Wiki-style links are resolved to standard markdown links
- [ ] Code blocks are protected from heading-based and size-based splitting
- [ ] Tables are protected from mid-row splitting
- [ ] Heading hierarchy is tracked and propagated as metadata
- [ ] Long sections are sub-split by RecursiveCharacterTextSplitter
- [ ] Short/empty chunks are filtered out
- [ ] File path and directory are included in metadata
- [ ] Incremental processing tracks file modification times
- [ ] Error handling skips corrupt files without stopping the pipeline

---

## References

- LangChain MarkdownHeaderTextSplitter -- https://python.langchain.com/docs/how_to/markdown_header_metadata_splitter/
- markdown-it-py -- https://github.com/executablebooks/markdown-it-py
- PyYAML -- https://pyyaml.org/
- Obsidian vault structure -- https://help.obsidian.md/
