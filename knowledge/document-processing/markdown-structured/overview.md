# Markdown Structured Parsing -- Comprehensive Guide

## Overview / TL;DR

Markdown is the most common format for documentation, knowledge bases, READMEs, wikis, and developer notes. Unlike PDFs or Word documents, markdown has explicit structural markers -- headings, lists, code blocks, tables, and links -- that can be leveraged for high-quality, structure-aware chunking. This guide covers header-aware splitting (LangChain MarkdownHeaderTextSplitter), markdown parsing libraries (markdown-it, tree-sitter-markdown, regex), frontmatter extraction, cross-reference resolution, and production-ready code for building documentation-to-searchable-chunks pipelines.

---

## Why Markdown Structure Matters for RAG

1. **Headings define topic boundaries.** A `## Authentication` section contains all authentication-related content. Splitting at heading boundaries produces semantically coherent chunks.
2. **Heading hierarchy provides context.** A chunk under `# User Guide > ## Authentication > ### API Keys` carries rich positional metadata.
3. **Code blocks are atomic.** A code block split mid-function is useless. Markdown fences (```) clearly delineate code.
4. **Lists and tables are structured.** Splitting a bulleted list mid-item destroys meaning. Markdown syntax makes list boundaries explicit.
5. **Frontmatter carries metadata.** YAML frontmatter in markdown files (title, tags, date, author) provides filterable metadata for retrieval.

---

## Approach 1: LangChain MarkdownHeaderTextSplitter

The most popular approach for heading-aware markdown chunking. Splits on heading boundaries and propagates heading hierarchy as metadata.

```python
from langchain_text_splitters import MarkdownHeaderTextSplitter, RecursiveCharacterTextSplitter


def split_markdown_by_headers(
    markdown_text: str,
    max_chunk_size: int = 1000,
    chunk_overlap: int = 200,
) -> list[dict]:
    """Split markdown by heading hierarchy, then by size."""
    headers_to_split_on = [
        ("#", "h1"),
        ("##", "h2"),
        ("###", "h3"),
        ("####", "h4"),
    ]

    # Step 1: Split by headers
    md_splitter = MarkdownHeaderTextSplitter(
        headers_to_split_on=headers_to_split_on,
        strip_headers=False,     # Keep headers in chunk text
    )
    header_chunks = md_splitter.split_text(markdown_text)

    # Step 2: Split long sections by size
    size_splitter = RecursiveCharacterTextSplitter(
        chunk_size=max_chunk_size,
        chunk_overlap=chunk_overlap,
        separators=["\n\n", "\n", ". ", " ", ""],
    )
    final_chunks = size_splitter.split_documents(header_chunks)

    # Convert to dicts
    results = []
    for i, chunk in enumerate(final_chunks):
        results.append({
            "text": chunk.page_content,
            "metadata": {
                **chunk.metadata,   # h1, h2, h3, h4 from header splitter
                "chunk_index": i,
            },
        })

    return results
```

**How it works**: The splitter scans for heading patterns (`#`, `##`, `###`, etc.) and creates chunk boundaries at each heading. Each chunk carries metadata with the full heading hierarchy (e.g., `{"h1": "User Guide", "h2": "Authentication", "h3": "API Keys"}`). Long sections are then split by the size splitter, which preserves the heading metadata through the second split.

---

## Approach 2: Custom Markdown Parser

For more control over code blocks, tables, lists, and frontmatter, use a custom parser.

```python
import re
import yaml
from dataclasses import dataclass, field


@dataclass
class MarkdownSection:
    heading: str
    heading_level: int
    content: str
    metadata: dict = field(default_factory=dict)
    code_blocks: list[dict] = field(default_factory=list)
    tables: list[str] = field(default_factory=list)
    lists: list[str] = field(default_factory=list)


def parse_markdown_structured(text: str) -> dict:
    """Parse markdown into structured sections with element types."""
    # Extract frontmatter
    frontmatter = {}
    content = text
    fm_match = re.match(r'^---\n(.*?)\n---\n', text, re.DOTALL)
    if fm_match:
        try:
            frontmatter = yaml.safe_load(fm_match.group(1)) or {}
        except yaml.YAMLError:
            pass
        content = text[fm_match.end():]

    # Parse into sections
    sections = _split_by_headings(content)

    # Extract elements from each section
    for section in sections:
        section.code_blocks = _extract_code_blocks(section.content)
        section.tables = _extract_tables(section.content)
        section.lists = _extract_lists(section.content)

    return {
        "frontmatter": frontmatter,
        "sections": sections,
    }


def _split_by_headings(text: str) -> list[MarkdownSection]:
    """Split markdown text into sections by heading boundaries."""
    lines = text.split("\n")
    sections = []
    current_heading = ""
    current_level = 0
    current_lines = []
    heading_stack = {}

    for line in lines:
        match = re.match(r'^(#{1,6})\s+(.+)$', line)
        if match:
            # Save previous section
            if current_lines:
                content = "\n".join(current_lines).strip()
                if content:
                    sections.append(MarkdownSection(
                        heading=current_heading,
                        heading_level=current_level,
                        content=content,
                        metadata={**heading_stack},
                    ))
                current_lines = []

            level = len(match.group(1))
            heading = match.group(2).strip()

            # Update heading stack
            heading_stack = {
                k: v for k, v in heading_stack.items()
                if int(k[1:]) < level
            }
            heading_stack[f"h{level}"] = heading

            current_heading = heading
            current_level = level

        current_lines.append(line)

    # Last section
    if current_lines:
        content = "\n".join(current_lines).strip()
        if content:
            sections.append(MarkdownSection(
                heading=current_heading,
                heading_level=current_level,
                content=content,
                metadata={**heading_stack},
            ))

    return sections


def _extract_code_blocks(text: str) -> list[dict]:
    """Extract fenced code blocks with language tags."""
    pattern = r'```(\w*)\n(.*?)```'
    blocks = []
    for match in re.finditer(pattern, text, re.DOTALL):
        blocks.append({
            "language": match.group(1) or "text",
            "code": match.group(2).strip(),
            "start": match.start(),
            "end": match.end(),
        })
    return blocks


def _extract_tables(text: str) -> list[str]:
    """Extract markdown tables."""
    pattern = r'(\|[^\n]+\|\n\|[\s\-:]+\|\n(?:\|[^\n]+\|\n)*)'
    return re.findall(pattern, text)


def _extract_lists(text: str) -> list[str]:
    """Extract contiguous list blocks."""
    pattern = r'((?:^\s*[-*+]\s+.+\n?)+)'
    return re.findall(pattern, text, re.MULTILINE)
```

---

## Approach 3: markdown-it-py Parser

markdown-it-py provides a token-based parser that produces an AST, enabling precise element-level access.

```python
from markdown_it import MarkdownIt


def parse_with_markdown_it(text: str) -> list[dict]:
    """Parse markdown into tokens using markdown-it-py."""
    md = MarkdownIt()
    tokens = md.parse(text)

    elements = []
    for token in tokens:
        if token.type == "heading_open":
            level = int(token.tag[1])  # h1 -> 1, h2 -> 2
            elements.append({
                "type": "heading",
                "level": level,
                "tag": token.tag,
            })
        elif token.type == "inline" and elements and elements[-1]["type"] == "heading":
            elements[-1]["text"] = token.content
        elif token.type == "paragraph_open":
            elements.append({"type": "paragraph"})
        elif token.type == "inline" and elements and elements[-1]["type"] == "paragraph":
            elements[-1]["text"] = token.content
        elif token.type == "fence":
            elements.append({
                "type": "code_block",
                "language": token.info,
                "code": token.content,
            })
        elif token.type == "bullet_list_open":
            elements.append({"type": "list", "items": []})
        elif token.type == "list_item_open":
            pass
        elif token.type == "inline" and elements and elements[-1]["type"] == "list":
            elements[-1]["items"].append(token.content)

    return elements
```

---

## Frontmatter Extraction

```python
import yaml
import re


def extract_frontmatter(text: str) -> tuple[dict, str]:
    """Extract YAML frontmatter from markdown."""
    match = re.match(r'^---\n(.*?)\n---\n?', text, re.DOTALL)
    if match:
        try:
            fm = yaml.safe_load(match.group(1)) or {}
        except yaml.YAMLError:
            fm = {}
        body = text[match.end():]
        return fm, body

    return {}, text


def enrich_chunks_with_frontmatter(
    chunks: list[dict],
    frontmatter: dict,
) -> list[dict]:
    """Add frontmatter fields to chunk metadata."""
    for chunk in chunks:
        meta = chunk.get("metadata", {})
        # Add selected frontmatter fields
        if "title" in frontmatter:
            meta["doc_title"] = frontmatter["title"]
        if "tags" in frontmatter:
            meta["tags"] = frontmatter["tags"]
        if "date" in frontmatter:
            meta["date"] = str(frontmatter["date"])
        if "author" in frontmatter:
            meta["author"] = frontmatter["author"]
        if "category" in frontmatter:
            meta["category"] = frontmatter["category"]
        chunk["metadata"] = meta

    return chunks
```

---

## Vault/Repository Processing

```python
from pathlib import Path
import logging

logger = logging.getLogger(__name__)


def process_markdown_vault(
    vault_path: str,
    max_chunk_size: int = 1000,
    chunk_overlap: int = 200,
) -> list[dict]:
    """Process all markdown files in a vault/repository."""
    vault = Path(vault_path)
    all_chunks = []

    md_files = sorted(vault.rglob("*.md"))
    logger.info(f"Found {len(md_files)} markdown files in {vault_path}")

    for md_file in md_files:
        try:
            text = md_file.read_text(encoding="utf-8", errors="replace")
            relative = str(md_file.relative_to(vault))

            # Extract frontmatter
            frontmatter, body = extract_frontmatter(text)

            # Split by headers then size
            chunks = split_markdown_by_headers(
                body,
                max_chunk_size=max_chunk_size,
                chunk_overlap=chunk_overlap,
            )

            # Add file-level metadata
            for chunk in chunks:
                chunk["metadata"]["source"] = relative
                chunk["metadata"]["directory"] = str(md_file.parent.relative_to(vault))

            # Add frontmatter
            chunks = enrich_chunks_with_frontmatter(chunks, frontmatter)

            all_chunks.extend(chunks)

        except Exception as e:
            logger.warning(f"Failed to process {md_file}: {e}")

    logger.info(f"Produced {len(all_chunks)} chunks from {len(md_files)} files")
    return all_chunks
```

---

## Cross-Reference Resolution

```python
import re
from pathlib import Path


def resolve_wiki_links(
    text: str,
    current_file: str,
    vault_path: str,
) -> str:
    """Resolve [[wiki-style links]] to relative paths."""
    vault = Path(vault_path)

    def _resolve(match):
        link_text = match.group(1)
        # Handle alias: [[target|display]]
        if "|" in link_text:
            target, display = link_text.split("|", 1)
        else:
            target = link_text
            display = link_text

        # Find the target file
        target_file = _find_file(vault, target)
        if target_file:
            rel_path = target_file.relative_to(vault)
            return f"[{display}]({rel_path})"
        return display

    return re.sub(r'\[\[([^\]]+)\]\]', _resolve, text)


def _find_file(vault: Path, name: str) -> Path | None:
    """Find a markdown file by name in the vault."""
    # Try exact match first
    if not name.endswith(".md"):
        name += ".md"

    for path in vault.rglob(name):
        return path

    # Try case-insensitive
    name_lower = name.lower()
    for path in vault.rglob("*.md"):
        if path.name.lower() == name_lower:
            return path

    return None


def extract_links(text: str) -> list[dict]:
    """Extract all links from markdown text."""
    links = []

    # Standard markdown links: [text](url)
    for match in re.finditer(r'\[([^\]]+)\]\(([^)]+)\)', text):
        links.append({
            "type": "markdown",
            "display": match.group(1),
            "target": match.group(2),
        })

    # Wiki links: [[target]] or [[target|display]]
    for match in re.finditer(r'\[\[([^\]]+)\]\]', text):
        link_text = match.group(1)
        if "|" in link_text:
            target, display = link_text.split("|", 1)
        else:
            target = display = link_text
        links.append({
            "type": "wiki",
            "display": display,
            "target": target,
        })

    return links
```

---

## Common Pitfalls

1. **Splitting inside code blocks.** A `RecursiveCharacterTextSplitter` applied after `MarkdownHeaderTextSplitter` can split inside fenced code blocks. Protect code blocks by extracting them first.
2. **Stripping headers from chunks.** Without headers, the chunk loses its topic context. Always keep `strip_headers=False` or prepend the heading hierarchy.
3. **Ignoring frontmatter.** YAML frontmatter contains title, tags, date, and category metadata that enables filtering and improves retrieval.
4. **Not preserving heading hierarchy.** A chunk under `### API Keys` without knowing it belongs to `## Authentication > # User Guide` loses context.
5. **Using fixed-size splitting on markdown.** Character-based splitting ignores headings, code blocks, and tables. Always use heading-aware splitting first.
6. **Not handling wiki-style links.** Obsidian, Notion exports, and some wikis use `[[link]]` syntax. Resolve or convert them.
7. **Chunking tables mid-row.** Markdown tables should be kept intact or split at row boundaries.
8. **Not normalizing link formats.** Some markdown files use relative paths, absolute paths, or wiki links. Normalize for consistent metadata.

---

## Production Checklist

- [ ] Splitting respects heading boundaries (MarkdownHeaderTextSplitter or custom)
- [ ] Heading hierarchy is preserved as metadata on each chunk
- [ ] Long sections are sub-split by RecursiveCharacterTextSplitter
- [ ] Code blocks are kept intact (not split mid-block)
- [ ] Tables are kept intact (not split mid-row)
- [ ] YAML frontmatter is extracted and added to chunk metadata
- [ ] File path and directory are included in metadata
- [ ] Wiki-style links are resolved or converted
- [ ] Vault/repo processing handles all .md files recursively
- [ ] Empty and very short chunks are filtered out

---

## References

- LangChain MarkdownHeaderTextSplitter -- https://python.langchain.com/docs/how_to/markdown_header_metadata_splitter/
- markdown-it-py -- https://github.com/executablebooks/markdown-it-py
- tree-sitter-markdown -- https://github.com/tree-sitter-grammars/tree-sitter-markdown
- Python-Markdown -- https://python-markdown.github.io/
- PyYAML -- https://pyyaml.org/
