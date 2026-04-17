# Code Chunking -- Comprehensive Guide

## Overview / TL;DR

Code chunking is the process of splitting source code into semantically meaningful segments for embedding and retrieval in code-aware RAG systems. Unlike prose text, code has a rigorous syntactic structure defined by its programming language grammar. Naive character-based splitting destroys this structure, producing chunks that cut functions in half, separate docstrings from their definitions, or mix code from unrelated classes. This guide covers every major code chunking approach (tree-sitter AST parsing, LangChain Language splitters, LlamaIndex CodeSplitter, and custom AST-based splitters), explains how to preserve semantic boundaries, and provides production-ready Python code for building code search and retrieval pipelines.

---

## Why Code Needs Special Chunking

1. **Syntactic structure matters.** A function definition, its docstring, its decorator, and its body form a semantic unit. Splitting mid-function produces chunks that cannot be understood in isolation.
2. **Scope and nesting.** Classes contain methods, modules contain classes, and functions contain nested functions. The chunking strategy must respect these hierarchical boundaries.
3. **Cross-references.** Imports, type annotations, and variable references create dependencies between code regions. Metadata must capture these relationships.
4. **Multiple languages.** A repository may contain Python, TypeScript, Go, Rust, SQL, and shell scripts. Each language has different structural patterns.
5. **Documentation co-location.** Docstrings and comments should stay with the code they describe, not be chunked separately.

---

## Approach 1: Tree-sitter AST Parsing

Tree-sitter is a parser generator that produces concrete syntax trees for source code. It supports 100+ languages with the same API.

```python
import tree_sitter_python as tspython
from tree_sitter import Language, Parser


def get_python_parser() -> Parser:
    """Create a tree-sitter parser for Python."""
    PY_LANGUAGE = Language(tspython.language())
    parser = Parser(PY_LANGUAGE)
    return parser


def parse_python_ast(source_code: str) -> list[dict]:
    """Parse Python source into AST nodes with metadata."""
    parser = get_python_parser()
    tree = parser.parse(bytes(source_code, "utf-8"))
    root = tree.root_node

    nodes = []
    _walk_tree(root, source_code, nodes, depth=0)
    return nodes


def _walk_tree(
    node,
    source: str,
    results: list[dict],
    depth: int,
):
    """Recursively walk the AST and collect meaningful nodes."""
    # Node types that represent semantic units
    meaningful_types = {
        "function_definition",
        "async_function_definition",
        "class_definition",
        "decorated_definition",
        "module",
    }

    if node.type in meaningful_types:
        start_byte = node.start_byte
        end_byte = node.end_byte
        text = source[start_byte:end_byte]

        # Extract name
        name = ""
        for child in node.children:
            if child.type == "identifier":
                name = source[child.start_byte:child.end_byte]
                break
            elif child.type in ("function_definition", "class_definition"):
                for grandchild in child.children:
                    if grandchild.type == "identifier":
                        name = source[grandchild.start_byte:grandchild.end_byte]
                        break

        # Extract docstring
        docstring = ""
        body = None
        for child in node.children:
            if child.type == "block":
                body = child
                break
        if body and body.children:
            first_stmt = body.children[0]
            if first_stmt.type == "expression_statement":
                expr = first_stmt.children[0] if first_stmt.children else None
                if expr and expr.type == "string":
                    docstring = source[expr.start_byte:expr.end_byte].strip("\"'")

        results.append({
            "type": node.type,
            "name": name,
            "text": text,
            "start_line": node.start_point[0] + 1,
            "end_line": node.end_point[0] + 1,
            "depth": depth,
            "docstring": docstring[:200],
            "byte_range": (start_byte, end_byte),
        })

    for child in node.children:
        _walk_tree(child, source, results, depth + 1)


def chunk_python_by_ast(
    source_code: str,
    filepath: str = "",
    max_chunk_lines: int = 100,
    include_imports: bool = True,
) -> list[dict]:
    """Chunk Python source code using tree-sitter AST boundaries."""
    parser = get_python_parser()
    tree = parser.parse(bytes(source_code, "utf-8"))
    root = tree.root_node
    lines = source_code.split("\n")

    chunks = []

    # Extract module-level imports
    import_lines = []
    if include_imports:
        for child in root.children:
            if child.type in ("import_statement", "import_from_statement"):
                import_lines.append(
                    source_code[child.start_byte:child.end_byte]
                )
    import_context = "\n".join(import_lines) + "\n\n" if import_lines else ""

    # Extract top-level definitions
    for child in root.children:
        if child.type in (
            "function_definition",
            "async_function_definition",
            "class_definition",
            "decorated_definition",
        ):
            text = source_code[child.start_byte:child.end_byte]
            num_lines = child.end_point[0] - child.start_point[0] + 1

            # If chunk is too large, split class into methods
            if num_lines > max_chunk_lines and child.type == "class_definition":
                class_chunks = _split_class(child, source_code, import_context, filepath)
                chunks.extend(class_chunks)
            else:
                # Prepend imports for context
                chunk_text = import_context + text

                # Extract name
                name = _get_node_name(child, source_code)

                chunks.append({
                    "text": chunk_text,
                    "metadata": {
                        "source": filepath,
                        "type": child.type,
                        "name": name,
                        "start_line": child.start_point[0] + 1,
                        "end_line": child.end_point[0] + 1,
                        "language": "python",
                    },
                })

    return chunks


def _split_class(node, source: str, import_context: str, filepath: str) -> list[dict]:
    """Split a large class into method-level chunks."""
    chunks = []
    class_name = _get_node_name(node, source)

    # Class header (up to first method)
    class_header = f"class {class_name}:  # (methods split into separate chunks)\n"

    for child in node.children:
        if child.type == "block":
            for stmt in child.children:
                if stmt.type in (
                    "function_definition",
                    "async_function_definition",
                    "decorated_definition",
                ):
                    method_text = source[stmt.start_byte:stmt.end_byte]
                    method_name = _get_node_name(stmt, source)

                    chunk_text = import_context + class_header + "    " + method_text.replace("\n", "\n    ")

                    chunks.append({
                        "text": chunk_text,
                        "metadata": {
                            "source": filepath,
                            "type": "method",
                            "name": f"{class_name}.{method_name}",
                            "class": class_name,
                            "start_line": stmt.start_point[0] + 1,
                            "end_line": stmt.end_point[0] + 1,
                            "language": "python",
                        },
                    })

    return chunks


def _get_node_name(node, source: str) -> str:
    """Extract the name from an AST node."""
    for child in node.children:
        if child.type == "identifier":
            return source[child.start_byte:child.end_byte]
        if child.type in ("function_definition", "class_definition"):
            return _get_node_name(child, source)
    return ""
```

### Multi-Language Support with Tree-sitter

```python
def get_parser(language: str) -> Parser:
    """Get a tree-sitter parser for the specified language."""
    language_modules = {
        "python": "tree_sitter_python",
        "javascript": "tree_sitter_javascript",
        "typescript": "tree_sitter_typescript",
        "go": "tree_sitter_go",
        "rust": "tree_sitter_rust",
        "java": "tree_sitter_java",
        "c": "tree_sitter_c",
        "cpp": "tree_sitter_cpp",
        "ruby": "tree_sitter_ruby",
    }

    module_name = language_modules.get(language)
    if not module_name:
        raise ValueError(f"Unsupported language: {language}")

    import importlib
    mod = importlib.import_module(module_name)
    lang = Language(mod.language())
    parser = Parser(lang)
    return parser


# Node types that represent function/class boundaries per language
BOUNDARY_TYPES = {
    "python": {
        "function_definition", "async_function_definition",
        "class_definition", "decorated_definition",
    },
    "javascript": {
        "function_declaration", "class_declaration",
        "arrow_function", "method_definition",
        "export_statement",
    },
    "typescript": {
        "function_declaration", "class_declaration",
        "arrow_function", "method_definition",
        "interface_declaration", "type_alias_declaration",
        "export_statement",
    },
    "go": {
        "function_declaration", "method_declaration",
        "type_declaration",
    },
    "rust": {
        "function_item", "impl_item", "struct_item",
        "enum_item", "trait_item",
    },
    "java": {
        "method_declaration", "class_declaration",
        "interface_declaration", "enum_declaration",
    },
}
```

---

## Approach 2: LangChain Language Splitters

LangChain provides language-aware recursive splitters that use language-specific separator hierarchies.

```python
from langchain_text_splitters import RecursiveCharacterTextSplitter, Language

# Python splitter
python_splitter = RecursiveCharacterTextSplitter.from_language(
    language=Language.PYTHON,
    chunk_size=2000,
    chunk_overlap=200,
)

# TypeScript splitter
ts_splitter = RecursiveCharacterTextSplitter.from_language(
    language=Language.TS,
    chunk_size=2000,
    chunk_overlap=200,
)

# Go splitter
go_splitter = RecursiveCharacterTextSplitter.from_language(
    language=Language.GO,
    chunk_size=2000,
    chunk_overlap=200,
)

# Usage
python_chunks = python_splitter.split_text(python_source)

# With LangChain Documents
from langchain_core.documents import Document

docs = [Document(page_content=source, metadata={"source": "app.py"})]
chunk_docs = python_splitter.split_documents(docs)
```

**Separator hierarchies used by LangChain**:

| Language | Primary Separators |
|----------|-------------------|
| Python | `\nclass `, `\ndef `, `\n\tdef `, `\n\n`, `\n`, `" "` |
| JavaScript | `\nfunction `, `\nconst `, `\nlet `, `\nclass `, `\n\n`, `\n` |
| TypeScript | `\ninterface `, `\ntype `, `\nfunction `, `\nclass `, `\n\n` |
| Go | `\nfunc `, `\ntype `, `\n\n`, `\n` |
| Rust | `\nfn `, `\nstruct `, `\nenum `, `\nimpl `, `\n\n` |
| Java | `\nclass `, `\npublic `, `\nprotected `, `\nprivate `, `\n\n` |

**Limitations**: LangChain splitters use regex-based separators, not AST parsing. They can split inside string literals, comments, or nested functions.

---

## Approach 3: LlamaIndex CodeSplitter

LlamaIndex's CodeSplitter uses tree-sitter for precise AST-based splitting.

```python
from llama_index.core.node_parser import CodeSplitter
from llama_index.core import Document

# Create splitter
splitter = CodeSplitter(
    language="python",
    chunk_lines=40,           # Target lines per chunk
    chunk_lines_overlap=5,    # Lines of overlap
    max_chars=1500,           # Maximum characters per chunk
)

# Split documents
documents = [Document(text=source_code, metadata={"source": "app.py"})]
nodes = splitter.get_nodes_from_documents(documents)

for node in nodes:
    print(f"Lines: {node.metadata.get('start_line')}-{node.metadata.get('end_line')}")
    print(f"Text length: {len(node.text)}")
    print("---")
```

**Supported languages**: Python, JavaScript, TypeScript, Go, Rust, Java, C, C++, Ruby, PHP, Scala, Swift, Markdown, LaTeX, HTML, and more (any language with a tree-sitter grammar).

---

## Approach 4: Python AST Module (Python Only)

For Python-only codebases, the built-in `ast` module provides reliable parsing without external dependencies.

```python
import ast
import textwrap


def chunk_python_stdlib(
    source_code: str,
    filepath: str = "",
    include_module_docstring: bool = True,
) -> list[dict]:
    """Chunk Python code using the standard library ast module."""
    tree = ast.parse(source_code)
    lines = source_code.split("\n")
    chunks = []

    # Module docstring
    module_docstring = ast.get_docstring(tree)
    if module_docstring and include_module_docstring:
        chunks.append({
            "text": f'"""{module_docstring}"""',
            "metadata": {
                "source": filepath,
                "type": "module_docstring",
                "name": filepath,
                "language": "python",
            },
        })

    # Import block
    import_lines = []
    for node in ast.iter_child_nodes(tree):
        if isinstance(node, (ast.Import, ast.ImportFrom)):
            import_lines.extend(lines[node.lineno - 1:node.end_lineno])
    import_block = "\n".join(import_lines)

    # Top-level definitions
    for node in ast.iter_child_nodes(tree):
        if isinstance(node, (ast.FunctionDef, ast.AsyncFunctionDef)):
            chunk_lines = lines[node.lineno - 1:node.end_lineno]
            # Include decorators
            if node.decorator_list:
                first_decorator = node.decorator_list[0]
                chunk_lines = lines[first_decorator.lineno - 1:node.end_lineno]

            chunk_text = "\n".join(chunk_lines)
            if import_block:
                chunk_text = import_block + "\n\n" + chunk_text

            chunks.append({
                "text": chunk_text,
                "metadata": {
                    "source": filepath,
                    "type": "function",
                    "name": node.name,
                    "start_line": node.lineno,
                    "end_line": node.end_lineno,
                    "language": "python",
                    "docstring": (ast.get_docstring(node) or "")[:200],
                    "args": [arg.arg for arg in node.args.args],
                },
            })

        elif isinstance(node, ast.ClassDef):
            chunk_lines = lines[node.lineno - 1:node.end_lineno]
            chunk_text = "\n".join(chunk_lines)
            if import_block:
                chunk_text = import_block + "\n\n" + chunk_text

            chunks.append({
                "text": chunk_text,
                "metadata": {
                    "source": filepath,
                    "type": "class",
                    "name": node.name,
                    "start_line": node.lineno,
                    "end_line": node.end_lineno,
                    "language": "python",
                    "docstring": (ast.get_docstring(node) or "")[:200],
                    "methods": [
                        m.name for m in node.body
                        if isinstance(m, (ast.FunctionDef, ast.AsyncFunctionDef))
                    ],
                    "bases": [ast.dump(b) for b in node.bases],
                },
            })

    return chunks
```

---

## Metadata Enrichment

```python
import hashlib
import re


def enrich_code_chunk_metadata(chunk: dict) -> dict:
    """Add computed metadata fields to a code chunk."""
    text = chunk["text"]
    metadata = chunk.get("metadata", {})

    # Content hash for deduplication
    metadata["content_hash"] = hashlib.sha256(text.encode()).hexdigest()[:16]

    # Size metrics
    metadata["char_count"] = len(text)
    metadata["line_count"] = text.count("\n") + 1

    # Code complexity indicators
    metadata["has_docstring"] = '"""' in text or "'''" in text
    metadata["has_type_hints"] = bool(re.search(r':\s*(str|int|float|bool|list|dict|tuple|Optional|Union)', text))
    metadata["has_decorators"] = bool(re.search(r'^@\w+', text, re.MULTILINE))
    metadata["has_error_handling"] = "try:" in text or "except " in text
    metadata["has_async"] = "async " in text or "await " in text

    # Import dependencies
    imports = re.findall(r'^(?:from|import)\s+(\S+)', text, re.MULTILINE)
    metadata["imports"] = imports

    chunk["metadata"] = metadata
    return chunk
```

---

## Common Pitfalls

1. **Using character-based splitters for code.** `RecursiveCharacterTextSplitter` without language awareness splits mid-function, mid-string, and mid-comment.
2. **Not including imports as context.** A function chunk without its imports is incomplete. Prepend relevant imports.
3. **Chunking classes as a single unit.** Large classes with many methods produce oversized chunks. Split into method-level chunks with class header context.
4. **Ignoring decorators.** Decorators are semantically bound to their function/class. Always include them in the chunk.
5. **Not handling nested functions.** Tree-sitter and ast both expose nested functions. Decide whether to chunk them separately or keep them within the parent.
6. **Missing file-level metadata.** Source file path, language, and line numbers are essential for citations and navigation.
7. **Not deduplicating across file versions.** In repos with multiple branches or copied files, identical chunks should be deduplicated using content hashes.
8. **Using the same chunk size for all languages.** Python functions average 15-30 lines; Java methods average 20-50. Tune chunk sizes per language.

---

## Production Checklist

- [ ] Chunking uses AST-aware parsing (tree-sitter or language-native AST)
- [ ] Chunks respect function, class, and method boundaries
- [ ] Imports are included as context in each chunk
- [ ] Large classes are split into method-level chunks with class header
- [ ] Decorators and docstrings stay with their definitions
- [ ] Metadata includes source file, language, line range, and function/class name
- [ ] Content hashes enable deduplication
- [ ] Multi-language repos use appropriate parsers per file extension
- [ ] Chunk size is tuned per language and embedding model
- [ ] Error handling catches syntax errors gracefully (parse failures should not crash the pipeline)

---

## References

- Tree-sitter documentation -- https://tree-sitter.github.io/tree-sitter/
- Tree-sitter Python bindings -- https://github.com/tree-sitter/py-tree-sitter
- LangChain code splitters -- https://python.langchain.com/docs/how_to/code_splitter/
- LlamaIndex CodeSplitter -- https://docs.llamaindex.ai/en/stable/module_guides/loading/node_parsers/
- Python ast module -- https://docs.python.org/3/library/ast.html
