# Office Documents -- Comprehensive Guide

## Overview / TL;DR

Office documents (Word, PowerPoint, Excel) are among the most common document types in enterprise environments. Unlike PDFs, these formats store structured data with explicit semantic markup -- headings, lists, tables, slides, and worksheets are encoded as first-class objects. This makes extraction more reliable than PDF parsing, but each format has its own library, quirks, and edge cases. This guide covers python-docx, mammoth, python-pptx, and openpyxl with production-ready code for extracting, converting to markdown, and preparing office documents for RAG pipelines.

---

## Format Landscape

| Format | Extensions | Structure | Primary Library | Alternative |
|--------|-----------|-----------|----------------|-------------|
| Word | .docx | Paragraphs, headings, tables, images | python-docx | mammoth |
| PowerPoint | .pptx | Slides, text boxes, shapes, tables | python-pptx | Unstructured |
| Excel | .xlsx | Worksheets, cells, formulas, charts | openpyxl | pandas |
| Legacy Word | .doc | Binary format | antiword/libreoffice | textract |
| Legacy Excel | .xls | Binary format | xlrd | pandas |
| Legacy PPT | .ppt | Binary format | libreoffice | python-pptx (no) |

---

## Word Documents (python-docx)

### Basic Extraction

```python
from docx import Document
from docx.enum.text import WD_ALIGN_PARAGRAPH


def extract_docx(file_path: str) -> dict:
    """Extract text, headings, tables, and metadata from a DOCX file."""
    doc = Document(file_path)

    # Core properties (metadata)
    props = doc.core_properties
    metadata = {
        "title": props.title or "",
        "author": props.author or "",
        "created": str(props.created) if props.created else "",
        "modified": str(props.modified) if props.modified else "",
        "subject": props.subject or "",
        "category": props.category or "",
    }

    # Extract paragraphs with style information
    paragraphs = []
    for para in doc.paragraphs:
        style_name = para.style.name if para.style else ""
        text = para.text.strip()

        if not text:
            continue

        paragraphs.append({
            "text": text,
            "style": style_name,
            "is_heading": style_name.startswith("Heading"),
            "heading_level": _get_heading_level(style_name),
            "alignment": str(para.alignment) if para.alignment else "left",
            "is_list": style_name.startswith("List"),
        })

    # Extract tables
    tables = []
    for table_idx, table in enumerate(doc.tables):
        rows = []
        for row in table.rows:
            cells = [cell.text.strip() for cell in row.cells]
            rows.append(cells)

        tables.append({
            "index": table_idx,
            "rows": len(rows),
            "cols": len(rows[0]) if rows else 0,
            "data": rows,
        })

    return {
        "metadata": metadata,
        "paragraphs": paragraphs,
        "tables": tables,
        "total_paragraphs": len(paragraphs),
        "total_tables": len(tables),
    }


def _get_heading_level(style_name: str) -> int | None:
    """Extract heading level from style name."""
    import re
    match = re.match(r'Heading (\d+)', style_name)
    return int(match.group(1)) if match else None
```

### DOCX to Markdown Conversion

```python
from docx import Document
import re


def docx_to_markdown(file_path: str) -> str:
    """Convert a DOCX file to clean markdown."""
    doc = Document(file_path)
    md_parts = []

    for para in doc.paragraphs:
        text = para.text.strip()
        if not text:
            md_parts.append("")
            continue

        style = para.style.name if para.style else ""

        # Headings
        level = _get_heading_level(style)
        if level:
            md_parts.append(f"{'#' * level} {text}")
            continue

        # List items
        if style.startswith("List"):
            if "Number" in style or "Ordered" in style:
                md_parts.append(f"1. {text}")
            else:
                md_parts.append(f"- {text}")
            continue

        # Bold/italic inline formatting
        formatted = _format_runs(para)
        md_parts.append(formatted)

    # Add tables
    for table in doc.tables:
        md_parts.append("")
        md_parts.append(_table_to_markdown(table))
        md_parts.append("")

    # Clean up excessive blank lines
    markdown = "\n".join(md_parts)
    markdown = re.sub(r'\n{4,}', '\n\n\n', markdown)

    return markdown.strip()


def _format_runs(paragraph) -> str:
    """Convert paragraph runs to markdown with bold/italic."""
    parts = []
    for run in paragraph.runs:
        text = run.text
        if not text:
            continue
        if run.bold and run.italic:
            text = f"***{text}***"
        elif run.bold:
            text = f"**{text}**"
        elif run.italic:
            text = f"*{text}*"
        parts.append(text)
    return "".join(parts) or paragraph.text


def _table_to_markdown(table) -> str:
    """Convert a DOCX table to markdown."""
    rows = []
    for row in table.rows:
        cells = [cell.text.strip() for cell in row.cells]
        rows.append(cells)

    if not rows:
        return ""

    md = "| " + " | ".join(rows[0]) + " |\n"
    md += "| " + " | ".join("---" for _ in rows[0]) + " |\n"
    for row in rows[1:]:
        md += "| " + " | ".join(row) + " |\n"

    return md
```

### mammoth (Alternative DOCX Converter)

```python
import mammoth


def docx_to_html_mammoth(file_path: str) -> str:
    """Convert DOCX to HTML using mammoth (better style mapping)."""
    with open(file_path, "rb") as f:
        result = mammoth.convert_to_html(f)

    return result.value


def docx_to_markdown_mammoth(file_path: str) -> str:
    """Convert DOCX to markdown using mammoth."""
    with open(file_path, "rb") as f:
        result = mammoth.convert_to_markdown(f)

    return result.value


def docx_to_html_custom_styles(file_path: str) -> str:
    """Convert with custom style mapping."""
    style_map = """
    p[style-name='Heading 1'] => h1:fresh
    p[style-name='Heading 2'] => h2:fresh
    p[style-name='Heading 3'] => h3:fresh
    p[style-name='Code'] => pre > code:fresh
    p[style-name='Quote'] => blockquote > p:fresh
    r[style-name='Code Char'] => code
    """

    with open(file_path, "rb") as f:
        result = mammoth.convert_to_html(f, style_map=style_map)

    return result.value
```

---

## PowerPoint (python-pptx)

```python
from pptx import Presentation
from pptx.util import Inches, Pt
from pptx.enum.shapes import MSO_SHAPE_TYPE


def extract_pptx(file_path: str) -> dict:
    """Extract all text content from a PowerPoint file."""
    prs = Presentation(file_path)
    slides = []

    for slide_num, slide in enumerate(prs.slides):
        slide_data = {
            "slide_number": slide_num + 1,
            "title": "",
            "texts": [],
            "notes": "",
            "tables": [],
        }

        for shape in slide.shapes:
            if shape.has_text_frame:
                for paragraph in shape.text_frame.paragraphs:
                    text = paragraph.text.strip()
                    if text:
                        slide_data["texts"].append(text)

            if shape.shape_type == MSO_SHAPE_TYPE.TABLE:
                table = shape.table
                rows = []
                for row in table.rows:
                    cells = [cell.text.strip() for cell in row.cells]
                    rows.append(cells)
                slide_data["tables"].append(rows)

            # Title detection
            if shape.has_text_frame and hasattr(shape, "placeholder_format"):
                if shape.placeholder_format and shape.placeholder_format.idx == 0:
                    slide_data["title"] = shape.text_frame.text.strip()

        # Slide notes
        if slide.has_notes_slide:
            notes = slide.notes_slide.notes_text_frame.text.strip()
            slide_data["notes"] = notes

        slides.append(slide_data)

    return {
        "total_slides": len(slides),
        "slides": slides,
    }


def pptx_to_markdown(file_path: str) -> str:
    """Convert PowerPoint to markdown with slide structure."""
    data = extract_pptx(file_path)
    md_parts = []

    for slide in data["slides"]:
        # Slide header
        title = slide["title"] or f"Slide {slide['slide_number']}"
        md_parts.append(f"## Slide {slide['slide_number']}: {title}")
        md_parts.append("")

        # Slide content
        for text in slide["texts"]:
            if text != slide["title"]:
                md_parts.append(text)

        # Tables
        for table in slide["tables"]:
            if table and table[0]:
                md_parts.append("")
                headers = table[0]
                md_parts.append("| " + " | ".join(headers) + " |")
                md_parts.append("| " + " | ".join("---" for _ in headers) + " |")
                for row in table[1:]:
                    md_parts.append("| " + " | ".join(row) + " |")
                md_parts.append("")

        # Notes
        if slide["notes"]:
            md_parts.append(f"\n> Speaker notes: {slide['notes']}")

        md_parts.append("")
        md_parts.append("---")
        md_parts.append("")

    return "\n".join(md_parts)
```

---

## Excel (openpyxl)

```python
from openpyxl import load_workbook


def extract_xlsx(
    file_path: str,
    include_formulas: bool = False,
    max_rows: int = 10000,
) -> dict:
    """Extract data from all worksheets in an Excel file."""
    wb = load_workbook(
        file_path,
        data_only=not include_formulas,
        read_only=True,
    )

    sheets = []
    for sheet_name in wb.sheetnames:
        ws = wb[sheet_name]
        rows = []
        row_count = 0

        for row in ws.iter_rows(values_only=True):
            if row_count >= max_rows:
                break
            cells = [str(cell) if cell is not None else "" for cell in row]
            if any(c.strip() for c in cells):
                rows.append(cells)
            row_count += 1

        sheets.append({
            "name": sheet_name,
            "rows": len(rows),
            "cols": max(len(r) for r in rows) if rows else 0,
            "data": rows,
        })

    wb.close()
    return {"sheets": sheets, "total_sheets": len(sheets)}


def xlsx_to_markdown(file_path: str) -> str:
    """Convert Excel file to markdown tables."""
    data = extract_xlsx(file_path)
    md_parts = []

    for sheet in data["sheets"]:
        md_parts.append(f"## Sheet: {sheet['name']}")
        md_parts.append("")

        if not sheet["data"]:
            md_parts.append("*(empty sheet)*")
            continue

        # First row as headers
        headers = sheet["data"][0]
        md_parts.append("| " + " | ".join(headers) + " |")
        md_parts.append("| " + " | ".join("---" for _ in headers) + " |")

        for row in sheet["data"][1:]:
            # Pad row if shorter than headers
            padded = row + [""] * (len(headers) - len(row))
            md_parts.append("| " + " | ".join(padded[:len(headers)]) + " |")

        md_parts.append("")

    return "\n".join(md_parts)
```

---

## Multi-Format Router

```python
import logging
from pathlib import Path
from dataclasses import dataclass, field

logger = logging.getLogger(__name__)


@dataclass
class OfficeDocResult:
    file_path: str
    format: str
    markdown: str
    metadata: dict = field(default_factory=dict)
    error: str | None = None


def process_office_document(file_path: str) -> OfficeDocResult:
    """Route an office document to the appropriate extractor."""
    ext = Path(file_path).suffix.lower()

    try:
        if ext == ".docx":
            markdown = docx_to_markdown(file_path)
            data = extract_docx(file_path)
            return OfficeDocResult(
                file_path=file_path,
                format="docx",
                markdown=markdown,
                metadata=data["metadata"],
            )

        elif ext == ".pptx":
            markdown = pptx_to_markdown(file_path)
            return OfficeDocResult(
                file_path=file_path,
                format="pptx",
                markdown=markdown,
            )

        elif ext in (".xlsx", ".xls"):
            markdown = xlsx_to_markdown(file_path)
            return OfficeDocResult(
                file_path=file_path,
                format="xlsx",
                markdown=markdown,
            )

        elif ext == ".doc":
            # Legacy .doc requires external tool
            markdown = _convert_legacy_doc(file_path)
            return OfficeDocResult(
                file_path=file_path,
                format="doc",
                markdown=markdown,
            )

        else:
            return OfficeDocResult(
                file_path=file_path,
                format=ext,
                markdown="",
                error=f"Unsupported format: {ext}",
            )

    except Exception as e:
        return OfficeDocResult(
            file_path=file_path,
            format=ext,
            markdown="",
            error=str(e),
        )


def _convert_legacy_doc(file_path: str) -> str:
    """Convert legacy .doc format using LibreOffice CLI."""
    import subprocess
    import tempfile

    with tempfile.TemporaryDirectory() as tmp_dir:
        result = subprocess.run(
            [
                "libreoffice", "--headless", "--convert-to", "docx",
                "--outdir", tmp_dir, file_path,
            ],
            capture_output=True,
            text=True,
            timeout=60,
        )

        if result.returncode != 0:
            raise RuntimeError(f"LibreOffice conversion failed: {result.stderr}")

        docx_path = Path(tmp_dir) / (Path(file_path).stem + ".docx")
        if docx_path.exists():
            return docx_to_markdown(str(docx_path))

    raise RuntimeError("Converted file not found")
```

---

## Common Pitfalls

1. **Ignoring document styles.** DOCX headings are defined by paragraph styles, not font size. Always use style names for heading detection.
2. **Losing table structure.** Tables in DOCX and PPTX are first-class objects. Extract them separately, not as concatenated text.
3. **Not handling merged cells in Excel.** openpyxl reports merged cells as None in non-primary positions. Detect and fill them.
4. **Skipping slide notes.** PowerPoint speaker notes often contain more detailed content than the slides themselves.
5. **Not converting legacy formats.** .doc, .xls, and .ppt require LibreOffice or specialized libraries. Always handle them in production.
6. **Processing very large Excel files in memory.** Use `read_only=True` mode and row limits for large spreadsheets.
7. **Ignoring inline formatting.** Bold, italic, and code styles in DOCX carry semantic meaning. Preserve them in markdown.
8. **Not extracting images.** Embedded images may contain text (diagrams, charts). Extract and OCR them for complete content.

---

## Production Checklist

- [ ] Format detection routes .docx, .pptx, .xlsx to appropriate extractors
- [ ] Legacy formats (.doc, .xls, .ppt) are converted via LibreOffice
- [ ] Headings are detected from document styles, not font properties
- [ ] Tables are extracted as structured data and converted to markdown
- [ ] PowerPoint notes are included in output
- [ ] Excel worksheets are individually processed with headers detected
- [ ] Inline formatting (bold, italic) is preserved in markdown
- [ ] Large files are handled with streaming/row limits
- [ ] Document metadata (title, author, dates) is preserved
- [ ] Error handling catches corrupt/password-protected files

---

## References

- python-docx documentation -- https://python-docx.readthedocs.io/en/latest/
- mammoth documentation -- https://github.com/mwilliamson/python-mammoth
- python-pptx documentation -- https://python-pptx.readthedocs.io/en/latest/
- openpyxl documentation -- https://openpyxl.readthedocs.io/en/stable/
