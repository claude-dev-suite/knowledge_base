# Table Extraction -- Production Pipeline

## Overview / TL;DR

A production table extraction pipeline must detect tables across document types, extract cell content accurately, handle edge cases (merged cells, multi-page tables, borderless layouts), normalize output, and produce chunks optimized for embedding and retrieval. This guide provides a complete pipeline with cascading extractors, quality validation, markdown and natural-language output, and integration with downstream RAG systems.

---

## Architecture

```
Input Document
    |
    v
[1] Table Detection
    |-- Rule-based (pdfplumber find_tables)
    |-- ML-based (TableTransformer) for borderless
    |
    v
[2] Table Classification
    |-- bordered / borderless / mixed
    |-- merged cells detected
    |
    v
[3] Extraction (cascading)
    |-- Tier 1: Camelot lattice / pdfplumber lines
    |-- Tier 2: Camelot stream / pdfplumber text
    |-- Tier 3: ML extractor (Docling/TableTransformer)
    |
    v
[4] Normalization
    |-- Cell cleaning
    |-- Header detection
    |-- Merged cell handling
    |-- Type inference (numeric, date, text)
    |
    v
[5] Chunking
    |-- Whole table (small tables)
    |-- Row-grouped (large tables)
    |-- NL description (for embedding)
    |
    v
[6] Output (markdown, JSON, vector store)
```

---

## Table Detection and Classification

```python
import pdfplumber
import fitz
from dataclasses import dataclass, field
from enum import Enum


class TableType(Enum):
    BORDERED = "bordered"
    PARTIAL_BORDER = "partial_border"
    BORDERLESS = "borderless"
    UNKNOWN = "unknown"


@dataclass
class DetectedTable:
    page: int
    bbox: tuple[float, float, float, float]  # x0, y0, x1, y1
    table_type: TableType
    confidence: float
    rows_estimate: int
    cols_estimate: int


def detect_tables(pdf_path: str) -> list[DetectedTable]:
    """Detect all tables in a PDF with type classification."""
    detected = []

    with pdfplumber.open(pdf_path) as pdf:
        for page_num, page in enumerate(pdf.pages):
            # Find tables using pdfplumber
            tables = page.find_tables()

            for table in tables:
                bbox = table.bbox
                # Classify table type based on line analysis
                table_type = _classify_table_type(page, bbox)

                detected.append(DetectedTable(
                    page=page_num + 1,
                    bbox=bbox,
                    table_type=table_type,
                    confidence=0.9 if table_type == TableType.BORDERED else 0.6,
                    rows_estimate=len(table.rows) if hasattr(table, "rows") else 0,
                    cols_estimate=len(table.cells[0]) if table.cells else 0,
                ))

    return detected


def _classify_table_type(page, bbox: tuple) -> TableType:
    """Classify table type based on line patterns within the bounding box."""
    x0, y0, x1, y1 = bbox

    # Count lines within the table area
    lines = page.lines if hasattr(page, "lines") else []
    h_lines = 0
    v_lines = 0

    for line in lines:
        lx0 = line.get("x0", 0)
        ly0 = line.get("top", 0)
        lx1 = line.get("x1", 0)
        ly1 = line.get("bottom", 0)

        # Check if line is within table bbox
        if lx0 >= x0 - 5 and lx1 <= x1 + 5 and ly0 >= y0 - 5 and ly1 <= y1 + 5:
            if abs(ly0 - ly1) < 2:  # Horizontal
                h_lines += 1
            elif abs(lx0 - lx1) < 2:  # Vertical
                v_lines += 1

    if h_lines > 2 and v_lines > 2:
        return TableType.BORDERED
    elif h_lines > 2:
        return TableType.PARTIAL_BORDER
    elif h_lines == 0 and v_lines == 0:
        return TableType.BORDERLESS
    else:
        return TableType.UNKNOWN
```

---

## Cascading Extraction Engine

```python
import logging
from dataclasses import dataclass, field

logger = logging.getLogger(__name__)


@dataclass
class ExtractedTable:
    page: int
    table_index: int
    data: list[list[str]]
    headers: list[str]
    rows: int
    cols: int
    method: str
    confidence: float
    table_type: str
    warnings: list[str] = field(default_factory=list)


class TableExtractionPipeline:
    """Cascading table extraction with multiple fallback strategies."""

    def __init__(self, enable_ml: bool = True):
        self.enable_ml = enable_ml

    def extract_all_tables(self, pdf_path: str) -> list[ExtractedTable]:
        """Extract all tables from a PDF using cascading strategies."""
        # Detect tables first
        detected = detect_tables(pdf_path)
        if not detected:
            return []

        results = []

        for det in detected:
            table = self._extract_single_table(pdf_path, det)
            if table:
                results.append(table)

        return results

    def _extract_single_table(
        self,
        pdf_path: str,
        detection: DetectedTable,
    ) -> ExtractedTable | None:
        """Try extraction strategies in order of speed/reliability."""

        strategies = self._get_strategies(detection.table_type)

        for strategy_name, strategy_fn in strategies:
            try:
                data = strategy_fn(pdf_path, detection)
                if data and len(data) > 1:
                    quality = self._assess_quality(data)
                    if quality > 0.5:
                        headers = data[0] if data else []
                        return ExtractedTable(
                            page=detection.page,
                            table_index=0,
                            data=data,
                            headers=headers,
                            rows=len(data),
                            cols=len(headers),
                            method=strategy_name,
                            confidence=quality,
                            table_type=detection.table_type.value,
                        )
                    else:
                        logger.info(
                            f"Strategy {strategy_name} quality too low: {quality:.2f}"
                        )
            except Exception as e:
                logger.debug(f"Strategy {strategy_name} failed: {e}")
                continue

        logger.warning(f"All strategies failed for table on page {detection.page}")
        return None

    def _get_strategies(self, table_type: TableType) -> list:
        """Order strategies based on table type."""
        if table_type == TableType.BORDERED:
            strategies = [
                ("camelot_lattice", self._extract_camelot_lattice),
                ("pdfplumber_lines", self._extract_pdfplumber_lines),
                ("camelot_stream", self._extract_camelot_stream),
            ]
        elif table_type == TableType.BORDERLESS:
            strategies = [
                ("pdfplumber_text", self._extract_pdfplumber_text),
                ("camelot_stream", self._extract_camelot_stream),
            ]
            if self.enable_ml:
                strategies.append(("docling", self._extract_docling))
        else:
            strategies = [
                ("pdfplumber_lines", self._extract_pdfplumber_lines),
                ("camelot_lattice", self._extract_camelot_lattice),
                ("pdfplumber_text", self._extract_pdfplumber_text),
                ("camelot_stream", self._extract_camelot_stream),
            ]

        return strategies

    def _extract_camelot_lattice(
        self,
        pdf_path: str,
        detection: DetectedTable,
    ) -> list[list[str]]:
        """Extract using Camelot lattice mode."""
        import camelot

        tables = camelot.read_pdf(
            pdf_path,
            pages=str(detection.page),
            flavor="lattice",
            line_scale=40,
        )

        if tables:
            best = max(tables, key=lambda t: t.accuracy)
            return best.df.values.tolist()
        return []

    def _extract_camelot_stream(
        self,
        pdf_path: str,
        detection: DetectedTable,
    ) -> list[list[str]]:
        """Extract using Camelot stream mode."""
        import camelot

        tables = camelot.read_pdf(
            pdf_path,
            pages=str(detection.page),
            flavor="stream",
            edge_tol=50,
            row_tol=2,
        )

        if tables:
            best = max(tables, key=lambda t: t.accuracy)
            return best.df.values.tolist()
        return []

    def _extract_pdfplumber_lines(
        self,
        pdf_path: str,
        detection: DetectedTable,
    ) -> list[list[str]]:
        """Extract using pdfplumber with line-based detection."""
        import pdfplumber

        with pdfplumber.open(pdf_path) as pdf:
            page = pdf.pages[detection.page - 1]
            tables = page.extract_tables({
                "vertical_strategy": "lines",
                "horizontal_strategy": "lines",
                "snap_tolerance": 3,
            })

            if tables:
                # Return the table closest to the detected bbox
                return tables[0]
        return []

    def _extract_pdfplumber_text(
        self,
        pdf_path: str,
        detection: DetectedTable,
    ) -> list[list[str]]:
        """Extract using pdfplumber with text-based detection."""
        import pdfplumber

        with pdfplumber.open(pdf_path) as pdf:
            page = pdf.pages[detection.page - 1]
            tables = page.extract_tables({
                "vertical_strategy": "text",
                "horizontal_strategy": "text",
                "snap_tolerance": 5,
                "min_words_vertical": 2,
            })

            if tables:
                return tables[0]
        return []

    def _extract_docling(
        self,
        pdf_path: str,
        detection: DetectedTable,
    ) -> list[list[str]]:
        """Extract using Docling ML pipeline."""
        from docling.document_converter import DocumentConverter

        converter = DocumentConverter()
        result = converter.convert(pdf_path)
        doc = result.document

        # Find tables in the document
        for item in doc.iterate_items():
            element, level = item
            if hasattr(element, "data") and hasattr(element.data, "table_cells"):
                # Convert Docling table to list of lists
                return self._docling_table_to_list(element)

        return []

    def _docling_table_to_list(self, table_element) -> list[list[str]]:
        """Convert a Docling table element to list of lists."""
        if not hasattr(table_element, "data"):
            return []
        # Simplified extraction from Docling's table structure
        data = table_element.data
        if hasattr(data, "grid"):
            return [[str(cell) for cell in row] for row in data.grid]
        return []

    def _assess_quality(self, data: list[list[str]]) -> float:
        """Assess extraction quality heuristically."""
        if not data or len(data) < 2:
            return 0.0

        total_cells = sum(len(row) for row in data)
        empty_cells = sum(
            1 for row in data
            for cell in row
            if not str(cell or "").strip()
        )

        if total_cells == 0:
            return 0.0

        # Penalize high empty-cell ratio
        empty_ratio = empty_cells / total_cells
        if empty_ratio > 0.7:
            return 0.2

        # Check column consistency
        col_counts = [len(row) for row in data]
        most_common_cols = max(set(col_counts), key=col_counts.count)
        consistency = sum(1 for c in col_counts if c == most_common_cols) / len(col_counts)

        # Combined score
        fill_score = 1.0 - empty_ratio
        return (fill_score * 0.6 + consistency * 0.4)
```

---

## Normalization and Post-Processing

```python
import re
from typing import Optional


def normalize_table(
    data: list[list[str]],
    detect_headers: bool = True,
) -> dict:
    """Normalize extracted table data."""
    if not data:
        return {"headers": [], "rows": [], "warnings": []}

    warnings = []

    # Clean all cells
    cleaned = []
    for row in data:
        cleaned_row = []
        for cell in row:
            value = str(cell or "").strip()
            value = re.sub(r'\s+', ' ', value)
            value = value.replace('\x00', '')
            cleaned_row.append(value)
        cleaned.append(cleaned_row)

    # Ensure consistent column count
    max_cols = max(len(row) for row in cleaned)
    for row in cleaned:
        while len(row) < max_cols:
            row.append("")

    # Detect and separate headers
    headers = []
    rows = cleaned
    if detect_headers and len(cleaned) > 1:
        potential_headers = cleaned[0]
        # Heuristic: headers are non-numeric, non-empty
        non_empty = sum(1 for h in potential_headers if h)
        if non_empty > len(potential_headers) * 0.5:
            headers = potential_headers
            rows = cleaned[1:]
        else:
            warnings.append("Could not detect headers; treating first row as data")

    # Remove fully empty rows
    rows = [row for row in rows if any(cell.strip() for cell in row)]

    # Detect merged cells (empty cells that should inherit from above)
    rows = _fill_merged_cells(rows, headers)

    return {
        "headers": headers,
        "rows": rows,
        "col_count": max_cols,
        "row_count": len(rows),
        "warnings": warnings,
    }


def _fill_merged_cells(
    rows: list[list[str]],
    headers: list[str],
) -> list[list[str]]:
    """Fill in empty cells that appear to be vertically merged."""
    if len(rows) < 2:
        return rows

    filled = [row[:] for row in rows]

    for col_idx in range(len(filled[0])):
        last_value = ""
        empty_streak = 0

        for row_idx in range(len(filled)):
            cell = filled[row_idx][col_idx]
            if cell.strip():
                last_value = cell
                empty_streak = 0
            else:
                empty_streak += 1
                # Only fill if the column is a "label" column (first 1-2 columns)
                # and streak is short (likely merged, not missing data)
                if col_idx < 2 and empty_streak <= 3 and last_value:
                    filled[row_idx][col_idx] = last_value

    return filled


def infer_column_types(
    headers: list[str],
    rows: list[list[str]],
) -> list[str]:
    """Infer column data types from content."""
    num_cols = len(headers) if headers else (len(rows[0]) if rows else 0)
    types = []

    for col_idx in range(num_cols):
        values = [
            rows[r][col_idx]
            for r in range(len(rows))
            if col_idx < len(rows[r]) and rows[r][col_idx].strip()
        ]

        if not values:
            types.append("empty")
            continue

        # Check if mostly numeric
        numeric_count = sum(
            1 for v in values
            if re.match(r'^[\-\$\d,.\s%]+$', v.strip())
        )
        if numeric_count > len(values) * 0.7:
            types.append("numeric")
            continue

        # Check if date-like
        date_count = sum(
            1 for v in values
            if re.match(r'\d{1,4}[-/]\d{1,2}[-/]\d{1,4}', v.strip())
        )
        if date_count > len(values) * 0.5:
            types.append("date")
            continue

        types.append("text")

    return types
```

---

## Table Chunking for RAG

```python
from dataclasses import dataclass


@dataclass
class TableRAGChunk:
    text: str
    metadata: dict


def chunk_tables_for_rag(
    tables: list[ExtractedTable],
    source_file: str,
    strategy: str = "adaptive",
    max_chunk_chars: int = 1500,
) -> list[TableRAGChunk]:
    """Chunk tables optimally for RAG retrieval."""
    chunks = []

    for table in tables:
        normalized = normalize_table(table.data)
        headers = normalized["headers"]
        rows = normalized["rows"]

        # Estimate markdown size
        md = _build_markdown(headers, rows)
        md_size = len(md)

        if strategy == "adaptive":
            if md_size <= max_chunk_chars:
                # Small table: keep whole
                chunks.append(TableRAGChunk(
                    text=md,
                    metadata=_table_metadata(table, source_file, "whole"),
                ))
            elif len(rows) <= 50:
                # Medium table: row groups with headers
                row_chunks = _chunk_by_rows(headers, rows, max_chunk_chars)
                for i, rc in enumerate(row_chunks):
                    chunks.append(TableRAGChunk(
                        text=rc,
                        metadata=_table_metadata(
                            table, source_file, f"rows_{i}",
                        ),
                    ))
            else:
                # Large table: NL descriptions per row
                for row_idx, row in enumerate(rows):
                    nl = _row_to_natural_language(headers, row)
                    chunks.append(TableRAGChunk(
                        text=nl,
                        metadata=_table_metadata(
                            table, source_file, f"row_nl_{row_idx}",
                        ),
                    ))
        elif strategy == "whole":
            chunks.append(TableRAGChunk(
                text=md,
                metadata=_table_metadata(table, source_file, "whole"),
            ))
        elif strategy == "rows":
            row_chunks = _chunk_by_rows(headers, rows, max_chunk_chars)
            for i, rc in enumerate(row_chunks):
                chunks.append(TableRAGChunk(
                    text=rc,
                    metadata=_table_metadata(table, source_file, f"rows_{i}"),
                ))
        elif strategy == "natural_language":
            for row_idx, row in enumerate(rows):
                nl = _row_to_natural_language(headers, row)
                chunks.append(TableRAGChunk(
                    text=nl,
                    metadata=_table_metadata(
                        table, source_file, f"row_nl_{row_idx}",
                    ),
                ))

    return chunks


def _build_markdown(headers: list[str], rows: list[list[str]]) -> str:
    """Build markdown table from headers and rows."""
    if not headers and not rows:
        return ""

    if headers:
        md = "| " + " | ".join(headers) + " |\n"
        md += "| " + " | ".join("---" for _ in headers) + " |\n"
    else:
        cols = len(rows[0]) if rows else 0
        md = ""

    for row in rows:
        md += "| " + " | ".join(str(c or "") for c in row) + " |\n"

    return md


def _chunk_by_rows(
    headers: list[str],
    rows: list[list[str]],
    max_chars: int,
) -> list[str]:
    """Split table into row groups, repeating headers."""
    chunks = []
    current_rows = []
    current_size = 0
    header_md = ""

    if headers:
        header_md = "| " + " | ".join(headers) + " |\n"
        header_md += "| " + " | ".join("---" for _ in headers) + " |\n"
        current_size = len(header_md)

    for row in rows:
        row_md = "| " + " | ".join(str(c or "") for c in row) + " |\n"
        if current_size + len(row_md) > max_chars and current_rows:
            chunks.append(header_md + "".join(current_rows))
            current_rows = []
            current_size = len(header_md)

        current_rows.append(row_md)
        current_size += len(row_md)

    if current_rows:
        chunks.append(header_md + "".join(current_rows))

    return chunks


def _row_to_natural_language(headers: list[str], row: list[str]) -> str:
    """Convert a table row to natural language description."""
    parts = []
    for header, value in zip(headers, row):
        if value and str(value).strip():
            parts.append(f"{header}: {str(value).strip()}")
    return "; ".join(parts)


def _table_metadata(
    table: ExtractedTable,
    source_file: str,
    chunk_type: str,
) -> dict:
    return {
        "source": source_file,
        "page": table.page,
        "table_index": table.table_index,
        "extraction_method": table.method,
        "confidence": table.confidence,
        "table_type": table.table_type,
        "chunk_type": chunk_type,
        "total_rows": table.rows,
        "total_cols": table.cols,
    }
```

---

## Multi-Page Table Stitching

```python
def stitch_multi_page_tables(
    tables: list[ExtractedTable],
) -> list[ExtractedTable]:
    """Detect and stitch tables that span multiple pages."""
    if len(tables) < 2:
        return tables

    stitched = []
    i = 0

    while i < len(tables):
        current = tables[i]

        # Look ahead for continuation on next page
        while i + 1 < len(tables):
            next_table = tables[i + 1]

            if _is_continuation(current, next_table):
                current = _merge_tables(current, next_table)
                i += 1
            else:
                break

        stitched.append(current)
        i += 1

    return stitched


def _is_continuation(
    table_a: ExtractedTable,
    table_b: ExtractedTable,
) -> bool:
    """Heuristic: is table_b a continuation of table_a?"""
    # Must be on consecutive pages
    if table_b.page != table_a.page + 1:
        return False

    # Column count should match
    if table_a.cols != table_b.cols:
        return False

    # table_b headers should match table_a headers (or table_b has no headers)
    if table_a.headers and table_b.data:
        first_row_b = table_b.data[0]
        headers_match = all(
            str(a).strip().lower() == str(b).strip().lower()
            for a, b in zip(table_a.headers, first_row_b)
        )
        if headers_match:
            return True  # Repeated headers = continuation

    return False


def _merge_tables(
    table_a: ExtractedTable,
    table_b: ExtractedTable,
) -> ExtractedTable:
    """Merge two tables, removing duplicate headers from table_b."""
    data_a = table_a.data
    data_b = table_b.data

    # Remove header row from table_b if it matches table_a headers
    if table_a.headers and data_b:
        first_row = data_b[0]
        if all(
            str(a).strip().lower() == str(b).strip().lower()
            for a, b in zip(table_a.headers, first_row)
        ):
            data_b = data_b[1:]

    merged_data = data_a + data_b

    return ExtractedTable(
        page=table_a.page,
        table_index=table_a.table_index,
        data=merged_data,
        headers=table_a.headers,
        rows=len(merged_data),
        cols=table_a.cols,
        method=table_a.method,
        confidence=min(table_a.confidence, table_b.confidence),
        table_type=table_a.table_type,
        warnings=table_a.warnings + [f"Stitched with page {table_b.page}"],
    )
```

---

## Common Pitfalls

1. **Using a single extraction strategy for all tables.** Bordered and borderless tables need different tools. A cascade approach handles mixed documents.
2. **Not detecting multi-page tables.** Tables split across pages produce two separate extractions with duplicate headers. Always check for continuations.
3. **Ignoring extraction quality assessment.** Low-quality extractions (high empty-cell ratio, inconsistent columns) should trigger fallback strategies.
4. **Chunking tables as plain text.** Tables chunked as raw text lose their structure. Use markdown format with repeated headers for row-group chunks.
5. **Not producing natural-language row descriptions.** Markdown tables do not embed well for semantic search. NL descriptions per row improve retrieval significantly.
6. **Forgetting to normalize cell content.** Extra whitespace, newlines, and null bytes in cells degrade both display and embedding quality.

---

## Production Checklist

- [ ] Table detection identifies all tables including borderless ones
- [ ] Cascading extraction tries fast tools first, falls back to ML
- [ ] Quality assessment rejects low-confidence extractions and triggers fallback
- [ ] Multi-page tables are detected and stitched
- [ ] Cell content is normalized (whitespace, encoding, merged cells)
- [ ] Headers are detected and preserved in all output formats
- [ ] Chunking strategy adapts to table size (whole, row-groups, NL)
- [ ] Metadata tracks extraction method, confidence, page, and table dimensions
- [ ] Column types are inferred for downstream processing
- [ ] Output validation checks for empty tables and inconsistent columns

---

## References

- Camelot documentation -- https://camelot-py.readthedocs.io/en/master/
- pdfplumber table extraction -- https://github.com/jsvine/pdfplumber#extracting-tables
- TableTransformer -- https://github.com/microsoft/table-transformer
- Docling tables -- https://ds4sd.github.io/docling/
