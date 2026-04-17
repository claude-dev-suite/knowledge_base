# Unstructured.io -- Comprehensive Guide

## Overview / TL;DR

Unstructured is an open-source library and hosted API for ingesting, partitioning, and preparing documents of any format for downstream NLP, RAG, and LLM pipelines. It provides a unified interface across PDFs, DOCX, PPTX, HTML, images, emails, and dozens of other formats, handling layout detection, OCR, table extraction, and element classification in a single pipeline. This guide covers the core partitioning engine, strategy selection, element types, configuration, and integration patterns with production-ready Python code targeting the 2024-2025 API.

---

## Why Unstructured Exists

Most document processing libraries handle one format well (PyMuPDF for PDFs, python-docx for DOCX, BeautifulSoup for HTML). Building a production ingestion pipeline means wiring together dozens of format-specific parsers, each with different APIs, output schemas, and failure modes. Unstructured solves this by providing:

1. **Unified element schema** -- Every document, regardless of format, is decomposed into typed elements (Title, NarrativeText, Table, ListItem, Image, etc.) with consistent metadata.
2. **Strategy-based partitioning** -- Choose between fast (rule-based), hi_res (ML layout analysis), OCR-only, or auto strategies depending on quality/speed requirements.
3. **Built-in connectors** -- Read from and write to S3, Azure Blob, GCS, Elasticsearch, Pinecone, Weaviate, and many other sources/destinations.
4. **Chunking integration** -- Built-in chunking that respects element boundaries, so tables and titles are never split mid-element.

---

## Installation

```python
# Core library (local processing)
# pip install unstructured

# With all format dependencies
# pip install "unstructured[all-docs]"

# Specific formats only
# pip install "unstructured[pdf,docx,pptx]"

# For hi_res strategy (layout analysis)
# pip install "unstructured[pdf]" unstructured-inference

# For the hosted API client
# pip install unstructured-client
```

---

## Core Concepts

### Partitioning

Partitioning is the process of decomposing a document into structured elements. Each element has a type, text content, and metadata.

```python
from unstructured.partition.auto import partition

# Auto-detect format and partition
elements = partition(filename="report.pdf")

for element in elements:
    print(f"Type: {type(element).__name__}")
    print(f"Text: {element.text[:100]}")
    print(f"Metadata: {element.metadata.to_dict()}")
    print("---")
```

### Element Types

Unstructured classifies content into semantic element types:

| Element Type | Description | Example |
|-------------|-------------|---------|
| `Title` | Section headings | `## Architecture Overview` |
| `NarrativeText` | Body paragraphs | Regular prose content |
| `ListItem` | Bullet/numbered items | `- First item` |
| `Table` | Tabular data | HTML table representation |
| `Image` | Embedded images | Image with optional OCR text |
| `FigureCaption` | Image/figure captions | `Figure 3: System diagram` |
| `Header` | Page headers | Running header text |
| `Footer` | Page footers | Page numbers, footer text |
| `Address` | Mailing addresses | Physical address blocks |
| `EmailAddress` | Email addresses | `user@example.com` |
| `PageBreak` | Page boundaries | Separator between pages |
| `Formula` | Mathematical formulas | Equation content |
| `CodeSnippet` | Code blocks | Source code |
| `UncategorizedText` | Unclassified content | Fallback type |

```python
from unstructured.documents.elements import (
    Title,
    NarrativeText,
    Table,
    ListItem,
    Image,
    Header,
    Footer,
)

# Filter elements by type
titles = [e for e in elements if isinstance(e, Title)]
tables = [e for e in elements if isinstance(e, Table)]
body_text = [e for e in elements if isinstance(e, NarrativeText)]

# Access element metadata
for element in elements:
    meta = element.metadata
    print(f"Page: {meta.page_number}")
    print(f"Filename: {meta.filename}")
    print(f"Filetype: {meta.filetype}")
    print(f"Languages: {meta.languages}")
    print(f"Coordinates: {meta.coordinates}")
```

---

## Partitioning Strategies

### Strategy Overview

```python
from unstructured.partition.pdf import partition_pdf

# Fast strategy: rule-based, no ML models
elements_fast = partition_pdf(
    filename="report.pdf",
    strategy="fast",
)

# Hi-res strategy: ML layout analysis + OCR
elements_hires = partition_pdf(
    filename="report.pdf",
    strategy="hi_res",
    hi_res_model_name="yolox",      # Layout detection model
    infer_table_structure=True,      # Extract table HTML
)

# OCR-only strategy: for scanned documents
elements_ocr = partition_pdf(
    filename="report.pdf",
    strategy="ocr_only",
    ocr_languages=["eng"],
)

# Auto strategy: choose best based on document characteristics
elements_auto = partition_pdf(
    filename="report.pdf",
    strategy="auto",
)
```

### Strategy Comparison

| Strategy | Speed | Quality | Tables | OCR | GPU Needed |
|----------|-------|---------|--------|-----|-----------|
| `fast` | Very Fast | Good | Basic | No | No |
| `hi_res` | Slow | Excellent | ML-based | Yes | Recommended |
| `ocr_only` | Medium | Good (scanned) | No | Yes | No |
| `auto` | Varies | Adaptive | Varies | If needed | Varies |

### Fast Strategy Deep Dive

The fast strategy uses pdfminer.six for text extraction and rule-based heuristics for element classification. No ML models are loaded.

```python
from unstructured.partition.pdf import partition_pdf

elements = partition_pdf(
    filename="report.pdf",
    strategy="fast",
    # Text extraction parameters
    include_page_breaks=True,
    languages=["eng"],
    # Metadata control
    include_metadata=True,
    metadata_filename="custom_name.pdf",
    metadata_last_modified="2025-01-15",
)

# Fast strategy element classification rules:
# - Short text in larger font -> Title
# - Paragraphs -> NarrativeText
# - Lines starting with bullets/numbers -> ListItem
# - Detected table patterns -> Table (basic)
```

### Hi-Res Strategy Deep Dive

The hi_res strategy uses a layout detection model (YOLOX or Detectron2) to identify regions on each page, then classifies and extracts content from each region.

```python
from unstructured.partition.pdf import partition_pdf

elements = partition_pdf(
    filename="report.pdf",
    strategy="hi_res",

    # Layout model selection
    hi_res_model_name="yolox",          # "yolox" (fast) or "detectron2_onnx"

    # Table extraction
    infer_table_structure=True,          # Extract table as HTML
    # Table structure model is used when infer_table_structure=True

    # OCR settings (used for image regions)
    ocr_languages=["eng"],

    # Image extraction
    extract_images_in_pdf=True,
    extract_image_block_output_dir="./images",
    extract_image_block_types=["Image", "Table"],

    # Coordinates (bounding boxes for each element)
    include_page_breaks=True,
)

# Hi-res elements have richer metadata
for element in elements:
    if isinstance(element, Table):
        # Table elements include HTML representation
        print(element.metadata.text_as_html)
        # <table><tr><td>Header 1</td><td>Header 2</td></tr>...</table>
```

---

## Format-Specific Partitioning

```python
from unstructured.partition.docx import partition_docx
from unstructured.partition.pptx import partition_pptx
from unstructured.partition.html import partition_html
from unstructured.partition.email import partition_email
from unstructured.partition.xlsx import partition_xlsx
from unstructured.partition.md import partition_md
from unstructured.partition.text import partition_text
from unstructured.partition.image import partition_image
from unstructured.partition.csv import partition_csv
from unstructured.partition.epub import partition_epub

# Word documents -- preserves headings, lists, tables
docx_elements = partition_docx(filename="document.docx")

# PowerPoint -- extracts text from slides
pptx_elements = partition_pptx(filename="presentation.pptx")

# HTML -- structure-aware parsing
html_elements = partition_html(filename="page.html")
html_elements_from_text = partition_html(text="<h1>Title</h1><p>Body</p>")

# Email (EML/MSG)
email_elements = partition_email(filename="message.eml")
for e in email_elements:
    print(e.metadata.sent_from)  # Sender
    print(e.metadata.sent_to)    # Recipients
    print(e.metadata.subject)    # Subject line

# Excel spreadsheets
xlsx_elements = partition_xlsx(filename="data.xlsx")

# Images (OCR)
image_elements = partition_image(
    filename="scan.png",
    strategy="hi_res",
    ocr_languages=["eng"],
)

# Markdown
md_elements = partition_md(filename="README.md")
```

---

## Hosted API (Unstructured API)

The hosted API provides the same partitioning capabilities as a managed service, with higher throughput and no local model dependencies.

```python
from unstructured_client import UnstructuredClient
from unstructured_client.models import operations, shared

client = UnstructuredClient(
    api_key_auth="your-api-key",
    server_url="https://api.unstructuredapp.io",
)


def partition_via_api(file_path: str, strategy: str = "auto") -> list:
    """Partition a document using the Unstructured hosted API."""
    with open(file_path, "rb") as f:
        req = operations.PartitionRequest(
            partition_parameters=shared.PartitionParameters(
                files=shared.Files(
                    content=f.read(),
                    file_name=file_path,
                ),
                strategy=shared.Strategy(strategy),
                languages=["eng"],
                split_pdf_page=True,           # Parallel page processing
                split_pdf_concurrency_level=5,
            ),
        )

    response = client.general.partition(request=req)
    return response.elements


def partition_with_chunking(file_path: str) -> list:
    """Partition and chunk in a single API call."""
    with open(file_path, "rb") as f:
        req = operations.PartitionRequest(
            partition_parameters=shared.PartitionParameters(
                files=shared.Files(
                    content=f.read(),
                    file_name=file_path,
                ),
                strategy=shared.Strategy.HI_RES,
                chunking_strategy="by_title",
                max_characters=1500,
                new_after_n_chars=1000,
                overlap=200,
                overlap_all=False,
                combine_under_n_chars=200,
            ),
        )

    response = client.general.partition(request=req)
    return response.elements
```

---

## Built-in Chunking

```python
from unstructured.partition.auto import partition
from unstructured.chunking.title import chunk_by_title
from unstructured.chunking.basic import chunk_elements

# Partition first
elements = partition(filename="report.pdf", strategy="hi_res")

# Chunk by title: groups elements under the same heading
chunks = chunk_by_title(
    elements,
    max_characters=1500,         # Hard max per chunk
    new_after_n_chars=1000,      # Soft max -- try to break here
    combine_text_under_n_chars=200,  # Merge small elements
    overlap=150,                 # Character overlap between chunks
    overlap_all=False,           # Only overlap between same-type elements
)

# Basic chunking: simple character-based grouping
basic_chunks = chunk_elements(
    elements,
    max_characters=1500,
    new_after_n_chars=1000,
    overlap=150,
)

# Each chunk is a CompositeElement with combined text
for chunk in chunks:
    print(f"Type: {type(chunk).__name__}")
    print(f"Text length: {len(chunk.text)}")
    print(f"Metadata: {chunk.metadata.to_dict()}")
    # Metadata includes orig_elements (list of source elements)
```

---

## Metadata and Element Processing

```python
from unstructured.partition.auto import partition
from unstructured.staging.base import elements_to_dicts, elements_from_dicts
import json


def elements_to_json(elements: list, output_path: str):
    """Serialize elements to JSON for storage or transport."""
    dicts = elements_to_dicts(elements)
    with open(output_path, "w") as f:
        json.dump(dicts, f, indent=2, default=str)


def elements_from_json(input_path: str) -> list:
    """Deserialize elements from JSON."""
    with open(input_path) as f:
        dicts = json.load(f)
    return elements_from_dicts(dicts)


def filter_content_elements(elements: list) -> list:
    """Remove headers, footers, and page breaks -- keep only content."""
    from unstructured.documents.elements import Header, Footer, PageBreak

    return [
        e for e in elements
        if not isinstance(e, (Header, Footer, PageBreak))
    ]


def enrich_metadata(elements: list, source_url: str, doc_id: str) -> list:
    """Add custom metadata to all elements."""
    for element in elements:
        element.metadata.url = source_url
        element.metadata.data_source = element.metadata.data_source or {}
    return elements
```

---

## Error Handling Patterns

```python
import logging
from pathlib import Path

logger = logging.getLogger(__name__)


def safe_partition(
    file_path: str,
    strategy: str = "auto",
    fallback_strategy: str = "fast",
) -> list:
    """Partition with fallback on failure."""
    from unstructured.partition.auto import partition

    try:
        elements = partition(
            filename=file_path,
            strategy=strategy,
            include_page_breaks=True,
        )
        if not elements:
            logger.warning(f"Empty result for {file_path} with {strategy}, trying {fallback_strategy}")
            elements = partition(
                filename=file_path,
                strategy=fallback_strategy,
            )
        return elements

    except Exception as e:
        logger.warning(f"Strategy {strategy} failed for {file_path}: {e}")
        if strategy != fallback_strategy:
            try:
                return partition(
                    filename=file_path,
                    strategy=fallback_strategy,
                )
            except Exception as e2:
                logger.error(f"Fallback also failed for {file_path}: {e2}")
                return []
        return []


def validate_elements(elements: list, file_path: str) -> dict:
    """Validate partitioning output quality."""
    from unstructured.documents.elements import NarrativeText, Title, Table

    total = len(elements)
    text_elements = [e for e in elements if isinstance(e, (NarrativeText, Title))]
    tables = [e for e in elements if isinstance(e, Table)]
    total_chars = sum(len(e.text) for e in elements)

    warnings = []
    if total == 0:
        warnings.append("No elements extracted")
    if total_chars < 100:
        warnings.append(f"Very little text extracted: {total_chars} chars")
    if total > 0 and total_chars / total < 10:
        warnings.append("Average element is very short (possible fragmentation)")

    return {
        "file": file_path,
        "total_elements": total,
        "text_elements": len(text_elements),
        "tables": len(tables),
        "total_chars": total_chars,
        "warnings": warnings,
        "is_valid": len(warnings) == 0,
    }
```

---

## Common Pitfalls

1. **Using hi_res for all documents.** The hi_res strategy loads ML models and is 10-50x slower than fast. Use fast for simple documents and hi_res only when layout analysis matters.
2. **Not installing format-specific dependencies.** `unstructured[all-docs]` installs everything, but in production, install only what you need to minimize image size.
3. **Ignoring table HTML.** When `infer_table_structure=True`, tables include HTML in `metadata.text_as_html`. The `text` field is a plain-text representation that loses structure.
4. **Not filtering headers and footers.** Page headers/footers pollute embeddings and retrieval. Filter them before chunking.
5. **Using the library for high-throughput without the API.** Local processing is CPU-bound and single-threaded per document. The hosted API parallelizes page processing.
6. **Not setting `combine_text_under_n_chars` when chunking.** Without it, every short element (titles, single-line paragraphs) becomes its own chunk.
7. **Assuming element types are always correct.** The fast strategy uses heuristics that can misclassify elements. Validate with hi_res for critical documents.

---

## Production Checklist

- [ ] Strategy selection matches document complexity (fast for simple, hi_res for complex/scanned)
- [ ] Format-specific dependencies are installed (avoid `all-docs` in production containers)
- [ ] Table structure extraction is enabled for documents with tables (`infer_table_structure=True`)
- [ ] Headers, footers, and page breaks are filtered before chunking
- [ ] Chunking uses `chunk_by_title` to respect document structure
- [ ] Element metadata (page number, source, coordinates) is preserved through chunking
- [ ] Error handling includes fallback strategies and per-file isolation
- [ ] Output validation checks for empty results and low text density
- [ ] API rate limits and concurrency are configured for hosted API usage
- [ ] Custom metadata (source URL, doc ID, ingestion timestamp) is attached to elements

---

## References

- Unstructured documentation -- https://docs.unstructured.io/
- Unstructured open-source repository -- https://github.com/Unstructured-IO/unstructured
- Unstructured API documentation -- https://docs.unstructured.io/api-reference/
- Element types reference -- https://docs.unstructured.io/open-source/concepts/document-elements
- Chunking strategies -- https://docs.unstructured.io/open-source/core-functionality/chunking
