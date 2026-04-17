# PDF Extraction -- Comprehensive Guide

## Overview / TL;DR

PDF extraction is the foundational step for any document-intelligence or RAG pipeline that ingests business documents, academic papers, legal contracts, or financial reports. PDFs are the most common document format in enterprise settings, yet they are notoriously hostile to text extraction -- they store rendering instructions (glyphs at coordinates), not semantic structure. This guide covers every major extraction library and service (PyMuPDF, pdfplumber, Docling, LlamaParse, Marker, Camelot, and cloud APIs), explains when each one excels, and provides production-ready Python code for building reliable extraction pipelines.

---

## Why PDF Extraction Is Hard

PDFs encode visual layout, not logical structure. A two-column academic paper stores characters in drawing order, which may interleave columns. A table is just lines and text positioned on a canvas -- there is no "table" object. Headers, footers, footnotes, and page numbers are indistinguishable from body text without heuristics.

Key challenges:

1. **No semantic structure** -- headings, paragraphs, and lists are not tagged in most PDFs.
2. **Multiple content types** -- text, tables, images, equations, and forms can coexist on a single page.
3. **Scanned documents** -- image-only PDFs require OCR before any text extraction.
4. **Encoding issues** -- custom fonts, ligatures, and character mappings can produce garbled text.
5. **Multi-column layouts** -- reading order is ambiguous without layout analysis.
6. **Embedded tables** -- table structure must be inferred from line positions or whitespace patterns.

---

## Library Landscape (2024-2025)

### PyMuPDF (fitz)

The fastest pure-extraction library. Wraps the MuPDF C library for direct access to the PDF rendering tree.

```python
import fitz  # PyMuPDF 1.25+

def extract_with_pymupdf(pdf_path: str) -> list[dict]:
    """Extract text and metadata page-by-page using PyMuPDF."""
    doc = fitz.open(pdf_path)
    pages = []

    for page_num, page in enumerate(doc):
        # Extract text preserving layout
        text = page.get_text("text")

        # Extract structured blocks (text + images)
        blocks = page.get_text("dict")["blocks"]

        # Separate text blocks from image blocks
        text_blocks = []
        image_blocks = []
        for block in blocks:
            if block["type"] == 0:  # Text block
                text_blocks.append({
                    "text": " ".join(
                        span["text"]
                        for line in block["lines"]
                        for span in line["spans"]
                    ),
                    "bbox": block["bbox"],
                    "font_sizes": list({
                        span["size"]
                        for line in block["lines"]
                        for span in line["spans"]
                    }),
                })
            elif block["type"] == 1:  # Image block
                image_blocks.append({
                    "bbox": block["bbox"],
                    "width": block.get("width", 0),
                    "height": block.get("height", 0),
                })

        pages.append({
            "page_num": page_num + 1,
            "text": text,
            "text_blocks": text_blocks,
            "image_blocks": image_blocks,
            "width": page.rect.width,
            "height": page.rect.height,
        })

    doc.close()
    return pages


def extract_markdown_pymupdf(pdf_path: str) -> str:
    """Extract PDF content as markdown using PyMuPDF4LLM."""
    import pymupdf4llm

    md_text = pymupdf4llm.to_markdown(
        pdf_path,
        page_chunks=False,    # Single string output
        write_images=False,   # Skip image extraction
        show_progress=False,
    )
    return md_text


def extract_page_chunks_pymupdf(pdf_path: str) -> list[dict]:
    """Extract per-page markdown chunks with metadata."""
    import pymupdf4llm

    chunks = pymupdf4llm.to_markdown(
        pdf_path,
        page_chunks=True,     # List of per-page dicts
        write_images=False,
    )
    # Each chunk: {"metadata": {...}, "text": "...markdown..."}
    return chunks
```

**Strengths**: Extremely fast (C-native), excellent text fidelity, rich block-level metadata (bounding boxes, font info), built-in markdown conversion via pymupdf4llm. **Weaknesses**: Table extraction requires post-processing, no built-in OCR (use with Tesseract or cloud OCR for scanned pages).

### pdfplumber

Built on pdfminer.six, pdfplumber excels at table detection and character-level access.

```python
import pdfplumber

def extract_with_pdfplumber(pdf_path: str) -> list[dict]:
    """Extract text and tables using pdfplumber."""
    pages = []

    with pdfplumber.open(pdf_path) as pdf:
        for page_num, page in enumerate(pdf.pages):
            # Full page text
            text = page.extract_text() or ""

            # Table extraction with custom settings
            table_settings = {
                "vertical_strategy": "lines",
                "horizontal_strategy": "lines",
                "snap_tolerance": 3,
                "join_tolerance": 3,
                "edge_min_length": 3,
                "min_words_vertical": 3,
                "min_words_horizontal": 1,
            }
            tables = page.extract_tables(table_settings)

            # Character-level data for custom parsing
            chars = page.chars  # List of dicts with x0, y0, x1, y1, text, fontname, size

            pages.append({
                "page_num": page_num + 1,
                "text": text,
                "tables": tables,  # List of list-of-lists
                "char_count": len(chars),
                "width": float(page.width),
                "height": float(page.height),
            })

    return pages


def tables_to_markdown(tables: list[list[list[str]]]) -> list[str]:
    """Convert pdfplumber tables to markdown format."""
    md_tables = []
    for table in tables:
        if not table or not table[0]:
            continue

        headers = table[0]
        md = "| " + " | ".join(str(h or "") for h in headers) + " |\n"
        md += "| " + " | ".join("---" for _ in headers) + " |\n"

        for row in table[1:]:
            md += "| " + " | ".join(str(cell or "") for cell in row) + " |\n"

        md_tables.append(md)

    return md_tables
```

**Strengths**: Best-in-class table extraction for line-bordered tables, character-level access, visual debugging (`page.to_image()`). **Weaknesses**: Slower than PyMuPDF (pure Python), struggles with borderless tables, no built-in markdown output.

### Docling (IBM)

Document understanding library that combines layout analysis (DocLayNet model) with structure recognition for high-fidelity conversion.

```python
from docling.document_converter import DocumentConverter
from docling.datamodel.pipeline_options import PdfPipelineOptions
from docling.datamodel.base_models import InputFormat

def extract_with_docling(pdf_path: str) -> dict:
    """Extract structured content using Docling's ML-based pipeline."""
    pipeline_options = PdfPipelineOptions()
    pipeline_options.do_ocr = True           # Enable OCR for scanned pages
    pipeline_options.do_table_structure = True # ML-based table recognition

    converter = DocumentConverter(
        allowed_formats=[InputFormat.PDF],
        format_options={
            InputFormat.PDF: {"pipeline_options": pipeline_options}
        },
    )

    result = converter.convert(pdf_path)
    doc = result.document

    # Export as markdown
    markdown = doc.export_to_markdown()

    # Export as structured JSON (DoclingDocument schema)
    doc_dict = doc.export_to_dict()

    # Access individual elements
    elements = []
    for item in doc.iterate_items():
        element, level = item
        elements.append({
            "type": type(element).__name__,
            "text": getattr(element, "text", ""),
            "level": level,
        })

    return {
        "markdown": markdown,
        "elements": elements,
        "num_pages": doc.num_pages(),
        "metadata": doc_dict.get("metadata", {}),
    }


def docling_chunking(pdf_path: str, max_tokens: int = 512) -> list[dict]:
    """Use Docling's built-in hierarchical chunker."""
    from docling.chunking import HierarchicalChunker

    converter = DocumentConverter()
    result = converter.convert(pdf_path)
    doc = result.document

    chunker = HierarchicalChunker(
        max_tokens=max_tokens,
        tokenizer="cl100k_base",
    )

    chunks = list(chunker.chunk(doc))
    return [
        {
            "text": chunk.text,
            "metadata": chunk.meta.export_json_dict(),
            "headings": chunk.meta.headings,
        }
        for chunk in chunks
    ]
```

**Strengths**: ML-based layout analysis (DocLayNet), excellent table structure recognition, built-in chunker that respects document hierarchy, open-source (MIT). **Weaknesses**: Heavier dependencies (PyTorch), slower than rule-based tools, relatively new (fewer battle-tested edge cases).

### LlamaParse

Cloud API from LlamaIndex that uses multimodal LLMs for document understanding.

```python
from llama_parse import LlamaParse

def extract_with_llamaparse(
    pdf_path: str,
    api_key: str,
    parsing_instruction: str = "",
) -> dict:
    """Extract content using LlamaParse cloud API."""
    parser = LlamaParse(
        api_key=api_key,
        result_type="markdown",           # "markdown" or "text"
        num_workers=4,                     # Parallel page processing
        verbose=False,
        language="en",
        parsing_instruction=parsing_instruction,
    )

    documents = parser.load_data(pdf_path)

    return {
        "pages": [
            {
                "text": doc.text,
                "metadata": doc.metadata,
            }
            for doc in documents
        ],
        "total_pages": len(documents),
    }


def llamaparse_with_images(pdf_path: str, api_key: str) -> dict:
    """Extract with image extraction enabled."""
    parser = LlamaParse(
        api_key=api_key,
        result_type="markdown",
        take_screenshot=True,          # Page screenshots for multimodal
        is_formatting_instruction=True,
        parsing_instruction=(
            "Extract all text preserving structure. "
            "Convert tables to markdown tables. "
            "Describe charts and figures in detail."
        ),
    )

    documents = parser.load_data(pdf_path)
    return {
        "pages": [{"text": doc.text, "metadata": doc.metadata} for doc in documents],
    }
```

**Strengths**: Highest quality for complex layouts (uses vision LLMs), handles scanned docs natively, excellent table extraction, supports custom parsing instructions. **Weaknesses**: Cloud-only (data leaves your infrastructure), per-page pricing, latency (seconds per page), rate limits.

### Marker

Open-source tool optimized for converting PDFs to clean markdown, particularly strong on academic papers.

```python
from marker.converters.pdf import PdfConverter
from marker.models import create_model_dict
from marker.config.parser import ConfigParser

def extract_with_marker(pdf_path: str) -> dict:
    """Convert PDF to markdown using Marker's ML pipeline."""
    config_parser = ConfigParser(
        {
            "output_format": "markdown",
            "force_ocr": False,
            "paginate_output": True,
        }
    )

    models = create_model_dict()
    converter = PdfConverter(
        config=config_parser.generate_config_dict(),
        artifact_dict=models,
    )

    rendered = converter(pdf_path)
    markdown = rendered.markdown

    # Access page-level data
    metadata = {
        "num_pages": len(rendered.children) if hasattr(rendered, "children") else 0,
    }

    return {
        "markdown": markdown,
        "metadata": metadata,
    }
```

**Strengths**: Excellent markdown output quality (especially for academic papers), handles equations (LaTeX), open-source with local execution, multi-column support. **Weaknesses**: GPU recommended for speed, large model downloads, less configurable than library approaches.

---

## Quick Comparison Matrix

| Feature | PyMuPDF | pdfplumber | Docling | LlamaParse | Marker |
|---------|---------|------------|---------|------------|--------|
| Speed (pages/sec) | 100+ | 5-15 | 1-3 | 0.5-2 | 1-5 |
| Text quality | High | High | Very High | Highest | Very High |
| Table extraction | Basic | Excellent | Very Good | Excellent | Good |
| Layout analysis | Rule-based | Rule-based | ML (DocLayNet) | Vision LLM | ML |
| OCR support | External | No | Built-in | Built-in | Built-in |
| Markdown output | pymupdf4llm | Manual | Built-in | Built-in | Built-in |
| Scanned PDFs | With OCR | No | Yes | Yes | Yes |
| Multi-column | Basic | Basic | Good | Excellent | Excellent |
| Local / Cloud | Local | Local | Local | Cloud | Local |
| License | AGPL-3.0 | MIT | MIT | Commercial | GPL-3.0 |

---

## Choosing the Right Tool

```
What type of PDF?
  |
  +---> Native text PDF (digital-born)
  |     |
  |     +---> Simple layout (single column, no tables)
  |     |     --> PyMuPDF (fastest, highest throughput)
  |     |
  |     +---> Has tables with visible borders
  |     |     --> pdfplumber (best rule-based table extraction)
  |     |
  |     +---> Complex layout (multi-column, mixed content)
  |           --> Docling or Marker (ML layout analysis)
  |
  +---> Scanned PDF (image-only)
  |     |
  |     +---> Clean scans, printed text
  |     |     --> Docling (built-in OCR + layout analysis)
  |     |
  |     +---> Poor quality scans, handwriting
  |           --> LlamaParse (vision LLM handles degraded input)
  |
  +---> Academic papers
  |     --> Marker (optimized for papers, handles equations)
  |
  +---> High-value documents (contracts, compliance)
        --> LlamaParse (highest accuracy, worth per-page cost)
```

---

## Font and Encoding Handling

```python
import fitz

def diagnose_encoding(pdf_path: str) -> list[dict]:
    """Detect encoding issues and custom fonts in a PDF."""
    doc = fitz.open(pdf_path)
    font_issues = []

    for page_num, page in enumerate(doc):
        blocks = page.get_text("dict")["blocks"]
        for block in blocks:
            if block["type"] != 0:
                continue
            for line in block["lines"]:
                for span in line["spans"]:
                    font = span["font"]
                    text = span["text"]

                    # Detect common encoding issues
                    has_replacement = "\ufffd" in text
                    has_control = any(ord(c) < 32 and c not in "\n\r\t" for c in text)
                    has_private_use = any(0xE000 <= ord(c) <= 0xF8FF for c in text)

                    if has_replacement or has_control or has_private_use:
                        font_issues.append({
                            "page": page_num + 1,
                            "font": font,
                            "text_preview": text[:50],
                            "replacement_chars": has_replacement,
                            "control_chars": has_control,
                            "private_use_area": has_private_use,
                        })

    doc.close()
    return font_issues


def extract_with_fallback(pdf_path: str) -> str:
    """Try multiple extraction methods, falling back on encoding issues."""
    import fitz

    doc = fitz.open(pdf_path)
    all_text = []

    for page in doc:
        # Method 1: Standard text extraction
        text = page.get_text("text")

        # Check for encoding issues
        garbled_ratio = sum(1 for c in text if ord(c) > 0xE000) / max(len(text), 1)

        if garbled_ratio > 0.05:
            # Method 2: Try rawdict for more control
            blocks = page.get_text("rawdict")["blocks"]
            text = " ".join(
                span["text"]
                for block in blocks if block["type"] == 0
                for line in block["lines"]
                for span in line["spans"]
            )

        if garbled_ratio > 0.1:
            # Method 3: OCR the page as image
            pix = page.get_pixmap(dpi=300)
            img_bytes = pix.tobytes("png")
            text = _ocr_image_bytes(img_bytes)

        all_text.append(text)

    doc.close()
    return "\n\n".join(all_text)


def _ocr_image_bytes(img_bytes: bytes) -> str:
    """OCR image bytes using Tesseract."""
    from PIL import Image
    import pytesseract
    import io

    image = Image.open(io.BytesIO(img_bytes))
    return pytesseract.image_to_string(image)
```

---

## Metadata Extraction

```python
import fitz
from datetime import datetime


def extract_pdf_metadata(pdf_path: str) -> dict:
    """Extract comprehensive PDF metadata."""
    doc = fitz.open(pdf_path)

    metadata = doc.metadata  # Standard PDF metadata fields

    # Parse dates
    for date_field in ["creationDate", "modDate"]:
        raw = metadata.get(date_field, "")
        if raw and raw.startswith("D:"):
            try:
                # PDF date format: D:YYYYMMDDHHmmSS+HH'mm'
                clean = raw[2:16]  # Strip D: prefix, take first 14 chars
                parsed = datetime.strptime(clean, "%Y%m%d%H%M%S")
                metadata[date_field + "_parsed"] = parsed.isoformat()
            except ValueError:
                metadata[date_field + "_parsed"] = None

    # Page-level metadata
    page_info = []
    for page in doc:
        page_info.append({
            "width": round(page.rect.width, 2),
            "height": round(page.rect.height, 2),
            "rotation": page.rotation,
            "has_text": bool(page.get_text("text").strip()),
            "has_images": len(page.get_images()) > 0,
            "num_links": len(page.get_links()),
        })

    result = {
        "title": metadata.get("title", ""),
        "author": metadata.get("author", ""),
        "subject": metadata.get("subject", ""),
        "creator": metadata.get("creator", ""),
        "producer": metadata.get("producer", ""),
        "creation_date": metadata.get("creationDate_parsed"),
        "mod_date": metadata.get("modDate_parsed"),
        "num_pages": len(doc),
        "is_encrypted": doc.is_encrypted,
        "page_info": page_info,
        "has_toc": bool(doc.get_toc()),
        "toc": doc.get_toc(),  # [[level, title, page_num], ...]
    }

    doc.close()
    return result
```

---

## Image Extraction from PDFs

```python
import fitz
from pathlib import Path


def extract_images(
    pdf_path: str,
    output_dir: str,
    min_width: int = 100,
    min_height: int = 100,
) -> list[dict]:
    """Extract images from PDF, filtering by minimum dimensions."""
    output = Path(output_dir)
    output.mkdir(parents=True, exist_ok=True)

    doc = fitz.open(pdf_path)
    extracted = []

    for page_num, page in enumerate(doc):
        image_list = page.get_images(full=True)

        for img_idx, img_info in enumerate(image_list):
            xref = img_info[0]

            try:
                base_image = doc.extract_image(xref)
            except Exception:
                continue

            width = base_image["width"]
            height = base_image["height"]

            if width < min_width or height < min_height:
                continue

            ext = base_image["ext"]
            img_bytes = base_image["image"]

            filename = f"page{page_num + 1}_img{img_idx + 1}.{ext}"
            filepath = output / filename

            filepath.write_bytes(img_bytes)

            extracted.append({
                "page": page_num + 1,
                "filename": filename,
                "width": width,
                "height": height,
                "colorspace": base_image.get("colorspace", 0),
                "size_bytes": len(img_bytes),
            })

    doc.close()
    return extracted
```

---

## Handling Large PDFs

```python
import fitz
from typing import Generator


def stream_pages(pdf_path: str, batch_size: int = 10) -> Generator[list[dict], None, None]:
    """Process large PDFs in page batches to control memory usage."""
    doc = fitz.open(pdf_path)
    total_pages = len(doc)

    for start in range(0, total_pages, batch_size):
        end = min(start + batch_size, total_pages)
        batch = []

        for page_num in range(start, end):
            page = doc[page_num]
            text = page.get_text("text")
            batch.append({
                "page_num": page_num + 1,
                "text": text,
                "char_count": len(text),
            })

        yield batch

    doc.close()


def extract_page_range(pdf_path: str, start: int, end: int) -> list[dict]:
    """Extract specific page range (1-indexed, inclusive)."""
    doc = fitz.open(pdf_path)
    pages = []

    for page_num in range(max(0, start - 1), min(end, len(doc))):
        page = doc[page_num]
        pages.append({
            "page_num": page_num + 1,
            "text": page.get_text("text"),
        })

    doc.close()
    return pages
```

---

## Detecting Scanned vs. Native PDFs

```python
import fitz


def classify_pdf(pdf_path: str) -> dict:
    """Classify PDF as native text, scanned, or mixed."""
    doc = fitz.open(pdf_path)
    page_classifications = []

    for page_num, page in enumerate(doc):
        text = page.get_text("text").strip()
        images = page.get_images()

        text_len = len(text)
        has_images = len(images) > 0

        # Heuristic: scanned pages have little/no text but large images
        if text_len > 50:
            classification = "native"
        elif has_images and text_len < 20:
            classification = "scanned"
        elif has_images and text_len < 50:
            classification = "mixed"
        else:
            classification = "empty"

        page_classifications.append({
            "page": page_num + 1,
            "type": classification,
            "text_length": text_len,
            "num_images": len(images),
        })

    doc.close()

    types = {p["type"] for p in page_classifications}
    if types == {"native"}:
        overall = "native"
    elif types == {"scanned"}:
        overall = "scanned"
    elif "empty" in types and len(types) == 1:
        overall = "empty"
    else:
        overall = "mixed"

    return {
        "overall_type": overall,
        "pages": page_classifications,
        "native_count": sum(1 for p in page_classifications if p["type"] == "native"),
        "scanned_count": sum(1 for p in page_classifications if p["type"] == "scanned"),
        "total_pages": len(page_classifications),
    }
```

---

## Common Pitfalls

1. **Assuming all PDFs have extractable text.** Many PDFs are scanned images. Always classify first, then route to OCR when needed.
2. **Ignoring reading order in multi-column layouts.** PyMuPDF and pdfplumber extract in drawing order, which interleaves columns. Use Docling or Marker for layout-aware extraction.
3. **Treating tables as regular text.** Tables become garbled when extracted as plain text. Use pdfplumber's table extraction or Docling's table recognition.
4. **Not handling password-protected PDFs.** Check `doc.is_encrypted` and call `doc.authenticate(password)` before extraction.
5. **Processing entire PDFs in memory.** Large PDFs (1000+ pages) can exhaust memory. Use page-range extraction or streaming.
6. **Ignoring font encoding issues.** Custom fonts with private-use-area codepoints produce garbage text. Detect and fall back to OCR.
7. **Using OCR on native-text PDFs.** OCR is slower and less accurate than direct text extraction. Classify pages first.
8. **Discarding PDF metadata.** Title, author, creation date, and table of contents provide valuable context for downstream processing.

---

## Production Checklist

- [ ] PDF type classification runs before extraction (native vs. scanned vs. mixed)
- [ ] Native pages use direct extraction (PyMuPDF or pdfplumber)
- [ ] Scanned pages route to OCR pipeline (Docling, Tesseract, or cloud OCR)
- [ ] Tables are detected and extracted separately from body text
- [ ] Encoding issues are detected and handled (fallback to OCR)
- [ ] Large PDFs are processed in page batches, not loaded entirely into memory
- [ ] Metadata (title, author, dates, TOC) is preserved as chunk metadata
- [ ] Images are extracted and optionally described for multimodal RAG
- [ ] Password-protected PDFs are handled gracefully (prompt for password or skip)
- [ ] Extraction errors are caught per-page so one bad page does not abort the entire document

---

## References

- PyMuPDF documentation -- https://pymupdf.readthedocs.io/en/latest/
- pymupdf4llm (markdown conversion) -- https://pymupdf.readthedocs.io/en/latest/pymupdf4llm/
- pdfplumber documentation -- https://github.com/jsvine/pdfplumber
- Docling documentation -- https://ds4sd.github.io/docling/
- LlamaParse documentation -- https://docs.llamaindex.ai/en/stable/llama_cloud/llama_parse/
- Marker repository -- https://github.com/VikParuchuri/marker
- PDF specification (ISO 32000) -- https://www.iso.org/standard/75839.html
