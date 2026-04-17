# Table Extraction -- Comprehensive Guide

## Overview / TL;DR

Tables are one of the most information-dense structures in documents, yet they are among the hardest to extract programmatically. A PDF table is just lines and text at coordinates -- there is no "table" object. An HTML table may use nested divs instead of `<table>` tags. A scanned document's table is just pixels. This guide covers every major table extraction tool (Camelot, Tabula, pdfplumber, TableTransformer, Amazon Textract, Azure Document Intelligence, Docling), explains when each excels, and provides production-ready Python code for extracting, normalizing, and chunking tabular data.

---

## Why Table Extraction Is Hard

1. **No semantic markup in PDFs.** Tables in PDFs are constructed from drawing commands (lines, rectangles) and positioned text. The PDF format has no concept of rows, columns, or cells.
2. **Bordered vs. borderless tables.** Bordered tables have visible grid lines that tools can detect. Borderless tables rely on whitespace alignment, which is ambiguous.
3. **Merged cells.** Row and column spans create irregular cell grids that most tools handle poorly.
4. **Multi-page tables.** Tables that span multiple pages require stitching across page boundaries.
5. **Nested tables.** Tables within tables (common in HTML reports) require recursive extraction.
6. **Scanned documents.** Tables in images require OCR before structure recognition.

---

## Tool Landscape

### Camelot

Rule-based PDF table extractor with two modes: lattice (for bordered tables) and stream (for borderless tables).

```python
import camelot

def extract_tables_camelot(
    pdf_path: str,
    pages: str = "all",
    flavor: str = "lattice",
) -> list[dict]:
    """Extract tables using Camelot."""
    tables = camelot.read_pdf(
        pdf_path,
        pages=pages,
        flavor=flavor,           # "lattice" (bordered) or "stream" (borderless)
        strip_text="\n",
        line_scale=40,           # Lattice: line detection sensitivity
        # Stream-specific options:
        # edge_tol=50,           # Edge tolerance for cell detection
        # row_tol=2,             # Row merge tolerance
    )

    results = []
    for i, table in enumerate(tables):
        df = table.df
        accuracy = table.accuracy     # 0-100 confidence score
        whitespace = table.whitespace  # Whitespace percentage (lower = better)

        results.append({
            "table_index": i,
            "page": table.page,
            "rows": len(df),
            "cols": len(df.columns),
            "accuracy": round(accuracy, 1),
            "whitespace": round(whitespace, 1),
            "dataframe": df,
            "data": df.values.tolist(),
        })

    return results


def camelot_lattice_tuned(pdf_path: str, pages: str = "1") -> list:
    """Lattice extraction with tuned parameters for complex bordered tables."""
    tables = camelot.read_pdf(
        pdf_path,
        pages=pages,
        flavor="lattice",
        line_scale=40,
        copy_text=["v"],          # Copy text in vertical cells
        shift_text=[""],          # Shift text to cell boundaries
        line_tol=2,               # Tolerance for line detection
        joint_tol=2,              # Tolerance for line intersections
        threshold_blocksize=15,   # Adaptive threshold block size
        threshold_constant=-2,    # Adaptive threshold constant
    )
    return tables


def camelot_stream_tuned(pdf_path: str, pages: str = "1") -> list:
    """Stream extraction for borderless tables."""
    tables = camelot.read_pdf(
        pdf_path,
        pages=pages,
        flavor="stream",
        edge_tol=50,              # Edge tolerance for text alignment
        row_tol=2,                # Row grouping tolerance
        columns=["72,200,350,500"],  # Manual column boundaries (optional)
    )
    return tables
```

**Strengths**: High accuracy on bordered tables (lattice mode), manual column specification for borderless, accuracy/whitespace scores. **Weaknesses**: Requires Ghostscript, no OCR, stream mode struggles with complex borderless layouts.

### Tabula (tabula-py)

Java-based extraction wrapped for Python. Similar approach to Camelot with lattice/stream modes.

```python
import tabula

def extract_tables_tabula(
    pdf_path: str,
    pages: str = "all",
    method: str = "lattice",
) -> list[dict]:
    """Extract tables using Tabula."""
    dfs = tabula.read_pdf(
        pdf_path,
        pages=pages,
        lattice=(method == "lattice"),
        stream=(method == "stream"),
        multiple_tables=True,
        pandas_options={"header": None},
    )

    results = []
    for i, df in enumerate(dfs):
        if df.empty:
            continue
        results.append({
            "table_index": i,
            "rows": len(df),
            "cols": len(df.columns),
            "dataframe": df,
            "data": df.values.tolist(),
        })

    return results


def tabula_with_area(
    pdf_path: str,
    page: int = 1,
    area: list[float] = None,
) -> list:
    """Extract table from a specific page area."""
    # area = [top, left, bottom, right] in points
    dfs = tabula.read_pdf(
        pdf_path,
        pages=str(page),
        area=area,               # Crop to specific region
        lattice=True,
        multiple_tables=True,
    )
    return dfs
```

**Strengths**: Mature, well-tested, area-based extraction. **Weaknesses**: Requires Java, slower than native Python tools, less tunable than Camelot.

### pdfplumber

Character-level PDF access with built-in table detection and extraction.

```python
import pdfplumber

def extract_tables_pdfplumber(pdf_path: str) -> list[dict]:
    """Extract tables using pdfplumber with tuned settings."""
    results = []

    with pdfplumber.open(pdf_path) as pdf:
        for page_num, page in enumerate(pdf.pages):
            # Default table settings
            tables = page.extract_tables({
                "vertical_strategy": "lines",
                "horizontal_strategy": "lines",
                "snap_tolerance": 3,
                "snap_x_tolerance": 3,
                "snap_y_tolerance": 3,
                "join_tolerance": 3,
                "join_x_tolerance": 3,
                "join_y_tolerance": 3,
                "edge_min_length": 3,
                "min_words_vertical": 3,
                "min_words_horizontal": 1,
                "text_tolerance": 3,
                "text_x_tolerance": 3,
                "text_y_tolerance": 3,
                "intersection_tolerance": 3,
                "intersection_x_tolerance": 3,
                "intersection_y_tolerance": 3,
            })

            for table_idx, table in enumerate(tables):
                if not table or not table[0]:
                    continue

                results.append({
                    "page": page_num + 1,
                    "table_index": table_idx,
                    "rows": len(table),
                    "cols": len(table[0]),
                    "data": table,
                })

    return results


def extract_borderless_tables_pdfplumber(pdf_path: str) -> list[dict]:
    """Extract borderless tables using text-based detection."""
    results = []

    with pdfplumber.open(pdf_path) as pdf:
        for page_num, page in enumerate(pdf.pages):
            tables = page.extract_tables({
                "vertical_strategy": "text",
                "horizontal_strategy": "text",
                "snap_tolerance": 5,
                "min_words_vertical": 2,
                "min_words_horizontal": 1,
            })

            for table_idx, table in enumerate(tables):
                if table and len(table) > 1:
                    results.append({
                        "page": page_num + 1,
                        "table_index": table_idx,
                        "rows": len(table),
                        "cols": len(table[0]) if table[0] else 0,
                        "data": table,
                    })

    return results


def debug_table_detection(pdf_path: str, page_num: int = 0):
    """Visually debug table detection on a page."""
    with pdfplumber.open(pdf_path) as pdf:
        page = pdf.pages[page_num]

        # Get table bounding boxes
        tables = page.find_tables()

        # Draw debug image
        im = page.to_image(resolution=150)
        for table in tables:
            im.draw_rect(table.bbox, stroke="red", stroke_width=2)
        im.save(f"debug_page_{page_num + 1}.png")

        return len(tables)
```

**Strengths**: No external dependencies (pure Python), character-level access, visual debugging, fine-grained control over detection parameters. **Weaknesses**: Slower than Camelot for large documents, borderless table detection is less reliable.

### TableTransformer (Microsoft)

Deep learning model for table detection and structure recognition, trained on PubTables-1M and FinTabNet.

```python
from transformers import AutoModelForObjectDetection, TableTransformerForObjectDetection
from transformers import AutoImageProcessor
from PIL import Image
import torch


def detect_tables_transformer(image_path: str) -> list[dict]:
    """Detect tables in a page image using TableTransformer."""
    # Load detection model
    processor = AutoImageProcessor.from_pretrained(
        "microsoft/table-transformer-detection"
    )
    model = AutoModelForObjectDetection.from_pretrained(
        "microsoft/table-transformer-detection"
    )

    image = Image.open(image_path).convert("RGB")
    inputs = processor(images=image, return_tensors="pt")

    with torch.no_grad():
        outputs = model(**inputs)

    # Post-process detections
    target_sizes = torch.tensor([image.size[::-1]])
    results = processor.post_process_object_detection(
        outputs, threshold=0.7, target_sizes=target_sizes
    )[0]

    tables = []
    for score, label, box in zip(
        results["scores"], results["labels"], results["boxes"]
    ):
        tables.append({
            "confidence": round(score.item(), 3),
            "label": model.config.id2label[label.item()],
            "bbox": [round(c.item(), 1) for c in box],
        })

    return tables


def recognize_table_structure(image_path: str, table_bbox: list) -> dict:
    """Recognize rows, columns, and cells within a detected table."""
    processor = AutoImageProcessor.from_pretrained(
        "microsoft/table-transformer-structure-recognition-v1.1-all"
    )
    model = TableTransformerForObjectDetection.from_pretrained(
        "microsoft/table-transformer-structure-recognition-v1.1-all"
    )

    image = Image.open(image_path).convert("RGB")

    # Crop to table region
    table_image = image.crop(table_bbox)

    inputs = processor(images=table_image, return_tensors="pt")

    with torch.no_grad():
        outputs = model(**inputs)

    target_sizes = torch.tensor([table_image.size[::-1]])
    results = processor.post_process_object_detection(
        outputs, threshold=0.5, target_sizes=target_sizes
    )[0]

    structure = {"rows": [], "columns": [], "cells": []}
    for score, label, box in zip(
        results["scores"], results["labels"], results["boxes"]
    ):
        label_name = model.config.id2label[label.item()]
        entry = {
            "confidence": round(score.item(), 3),
            "bbox": [round(c.item(), 1) for c in box],
        }

        if "row" in label_name:
            structure["rows"].append(entry)
        elif "column" in label_name:
            structure["columns"].append(entry)
        else:
            structure["cells"].append(entry)

    return structure
```

**Strengths**: Handles bordered and borderless tables, works on images (scanned docs), ML-based structure recognition. **Weaknesses**: Requires PyTorch/GPU, needs page-to-image conversion, more complex pipeline.

### Amazon Textract

Cloud service for document analysis with built-in table extraction.

```python
import boto3
from pathlib import Path


def extract_tables_textract(pdf_path: str) -> list[dict]:
    """Extract tables using Amazon Textract."""
    client = boto3.client("textract")

    with open(pdf_path, "rb") as f:
        document_bytes = f.read()

    response = client.analyze_document(
        Document={"Bytes": document_bytes},
        FeatureTypes=["TABLES"],
    )

    # Parse Textract response into table structures
    blocks = response["Blocks"]
    block_map = {b["Id"]: b for b in blocks}

    tables = []
    for block in blocks:
        if block["BlockType"] != "TABLE":
            continue

        table_data = []
        if "Relationships" not in block:
            continue

        cells = []
        for rel in block["Relationships"]:
            if rel["Type"] == "CHILD":
                for cell_id in rel["Ids"]:
                    cell_block = block_map[cell_id]
                    if cell_block["BlockType"] == "CELL":
                        cells.append(cell_block)

        # Organize cells into rows
        max_row = max(c["RowIndex"] for c in cells)
        max_col = max(c["ColumnIndex"] for c in cells)

        table_grid = [["" for _ in range(max_col)] for _ in range(max_row)]
        for cell in cells:
            row_idx = cell["RowIndex"] - 1
            col_idx = cell["ColumnIndex"] - 1
            cell_text = _get_cell_text(cell, block_map)
            table_grid[row_idx][col_idx] = cell_text

        tables.append({
            "rows": max_row,
            "cols": max_col,
            "data": table_grid,
            "confidence": block.get("Confidence", 0),
        })

    return tables


def _get_cell_text(cell_block: dict, block_map: dict) -> str:
    """Extract text content from a Textract cell block."""
    text_parts = []
    if "Relationships" in cell_block:
        for rel in cell_block["Relationships"]:
            if rel["Type"] == "CHILD":
                for word_id in rel["Ids"]:
                    word_block = block_map.get(word_id, {})
                    if word_block.get("BlockType") == "WORD":
                        text_parts.append(word_block.get("Text", ""))
    return " ".join(text_parts)
```

---

## Table Output Normalization

```python
import pandas as pd
import re


def table_to_dataframe(
    table_data: list[list[str]],
    first_row_header: bool = True,
) -> pd.DataFrame:
    """Convert raw table data to a clean DataFrame."""
    if not table_data:
        return pd.DataFrame()

    # Clean cell values
    cleaned = []
    for row in table_data:
        cleaned_row = []
        for cell in row:
            value = str(cell or "").strip()
            value = re.sub(r'\s+', ' ', value)
            cleaned_row.append(value)
        cleaned.append(cleaned_row)

    if first_row_header and len(cleaned) > 1:
        headers = cleaned[0]
        # Ensure unique column names
        seen = {}
        unique_headers = []
        for h in headers:
            if h in seen:
                seen[h] += 1
                unique_headers.append(f"{h}_{seen[h]}")
            else:
                seen[h] = 0
                unique_headers.append(h)

        df = pd.DataFrame(cleaned[1:], columns=unique_headers)
    else:
        df = pd.DataFrame(cleaned)

    return df


def table_to_markdown(table_data: list[list[str]]) -> str:
    """Convert table data to markdown format."""
    if not table_data or not table_data[0]:
        return ""

    headers = [str(h or "").strip() for h in table_data[0]]
    md = "| " + " | ".join(headers) + " |\n"
    md += "| " + " | ".join("---" for _ in headers) + " |\n"

    for row in table_data[1:]:
        cells = [str(c or "").strip() for c in row]
        # Pad if row has fewer cells than header
        while len(cells) < len(headers):
            cells.append("")
        md += "| " + " | ".join(cells[:len(headers)]) + " |\n"

    return md


def table_to_csv(table_data: list[list[str]], output_path: str):
    """Write table data to CSV."""
    import csv
    with open(output_path, "w", newline="", encoding="utf-8") as f:
        writer = csv.writer(f)
        for row in table_data:
            writer.writerow([str(c or "").strip() for c in row])
```

---

## Chunking Strategies for Tables

```python
from dataclasses import dataclass


@dataclass
class TableChunk:
    text: str
    metadata: dict


def chunk_table_whole(
    table_data: list[list[str]],
    source: str,
    page: int,
    table_index: int,
) -> TableChunk:
    """Keep the entire table as a single chunk."""
    md = table_to_markdown(table_data)
    return TableChunk(
        text=md,
        metadata={
            "source": source,
            "page": page,
            "table_index": table_index,
            "chunk_type": "table",
            "rows": len(table_data),
            "cols": len(table_data[0]) if table_data else 0,
        },
    )


def chunk_table_by_rows(
    table_data: list[list[str]],
    source: str,
    page: int,
    table_index: int,
    rows_per_chunk: int = 10,
) -> list[TableChunk]:
    """Split table into chunks of N rows, repeating headers."""
    if len(table_data) < 2:
        return [chunk_table_whole(table_data, source, page, table_index)]

    headers = table_data[0]
    data_rows = table_data[1:]
    chunks = []

    for i in range(0, len(data_rows), rows_per_chunk):
        batch = data_rows[i:i + rows_per_chunk]
        chunk_table = [headers] + batch
        md = table_to_markdown(chunk_table)

        chunks.append(TableChunk(
            text=md,
            metadata={
                "source": source,
                "page": page,
                "table_index": table_index,
                "chunk_type": "table_rows",
                "row_start": i + 1,
                "row_end": i + len(batch),
                "total_rows": len(data_rows),
            },
        ))

    return chunks


def chunk_table_by_columns(
    table_data: list[list[str]],
    source: str,
    page: int,
    table_index: int,
    key_columns: list[int] = None,
) -> list[TableChunk]:
    """Create per-row natural language descriptions for embedding."""
    if len(table_data) < 2:
        return [chunk_table_whole(table_data, source, page, table_index)]

    headers = table_data[0]
    chunks = []

    for row_idx, row in enumerate(table_data[1:]):
        # Create natural language representation
        parts = []
        for col_idx, (header, value) in enumerate(zip(headers, row)):
            if value and str(value).strip():
                parts.append(f"{header}: {str(value).strip()}")

        text = "; ".join(parts)

        chunks.append(TableChunk(
            text=text,
            metadata={
                "source": source,
                "page": page,
                "table_index": table_index,
                "chunk_type": "table_row_nl",
                "row_index": row_idx + 1,
            },
        ))

    return chunks
```

---

## Common Pitfalls

1. **Using stream/text mode on bordered tables.** Always try lattice/lines mode first for bordered tables. Stream mode is a fallback for borderless.
2. **Not handling merged cells.** Merged cells appear as empty cells in adjacent positions. Detect and fill them.
3. **Assuming all tables have headers.** Some tables use the first row as data, not headers. Check before treating row 0 as column names.
4. **Splitting tables mid-row during chunking.** Row integrity is critical. Always repeat headers and chunk at row boundaries.
5. **Ignoring table detection confidence scores.** Camelot and Textract provide confidence scores. Filter or flag low-confidence extractions.
6. **Not normalizing cell whitespace.** PDF extraction often produces cells with extra spaces, newlines, or invisible characters.
7. **OCR tables without preprocessing.** Scanned tables need deskewing, binarization, and denoising before OCR for good results.
8. **Using only markdown format for table chunks.** Natural language descriptions of rows embed better than markdown for semantic search.

---

## Production Checklist

- [ ] Table detection uses the right mode (lattice for bordered, stream for borderless)
- [ ] Multiple extractors are available for fallback (e.g., pdfplumber then Camelot then ML)
- [ ] Merged cells are detected and handled
- [ ] Headers are identified and repeated in row-chunked output
- [ ] Cell content is normalized (whitespace, encoding)
- [ ] Confidence scores are tracked and low-confidence tables are flagged
- [ ] Multi-page tables are stitched across page boundaries
- [ ] Tables are chunked appropriately for the use case (whole, by-rows, or NL description)
- [ ] Table metadata (page, index, dimensions) is preserved in chunks
- [ ] Scanned tables route through OCR preprocessing pipeline

---

## References

- Camelot documentation -- https://camelot-py.readthedocs.io/en/master/
- Tabula-py documentation -- https://tabula-py.readthedocs.io/en/latest/
- pdfplumber documentation -- https://github.com/jsvine/pdfplumber
- TableTransformer -- https://github.com/microsoft/table-transformer
- PubTables-1M dataset -- https://arxiv.org/abs/2110.00061
- Amazon Textract tables -- https://docs.aws.amazon.com/textract/latest/dg/how-it-works-tables.html
