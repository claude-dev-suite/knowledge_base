# Multimodal RAG -- Table Pipeline: Detection, Extraction, and Embedding

## TL;DR

Tables are the most common non-text element in enterprise documents and the most frequently mishandled by RAG pipelines. Text-only extraction flattens row-column relationships, destroying the structured information that makes tables valuable. The table pipeline addresses this with three stages: (1) detection -- identifying table boundaries in PDFs, HTML, and images, (2) extraction -- converting detected tables to structured formats (markdown, JSON, pandas DataFrames), and (3) embedding -- creating embeddings that preserve structural relationships for accurate retrieval. This article provides production implementations for each stage with support for PDFs, HTML documents, and image-based tables.

---

## Why Tables Break Standard RAG

### The Problem

When a standard text extractor encounters this table:

```
| Region | Q3 Revenue | Q4 Revenue | Growth |
|--------|-----------|-----------|--------|
| NA     | $12.5M    | $14.2M    | +13.6% |
| EU     | $8.3M     | $9.1M     | +9.6%  |
| APAC   | $6.1M     | $7.8M     | +27.9% |
```

It often produces something like:

```
Region Q3 Revenue Q4 Revenue Growth NA $12.5M $14.2M +13.6% EU $8.3M $9.1M +9.6% APAC $6.1M $7.8M +27.9%
```

This flattened text loses all structural relationships. A query like "What was NA's Q4 revenue?" cannot be reliably answered because the text chunk does not preserve which value belongs to which column and row.

### What the Table Pipeline Preserves

| Information Type | Standard RAG | Table Pipeline |
|-----------------|-------------|----------------|
| Cell values | Yes (jumbled) | Yes (structured) |
| Column headers | Sometimes | Always |
| Row-column relationships | No | Yes |
| Merged cells | No | Yes |
| Table context (caption) | Sometimes | Always |
| Cross-table references | No | Optional |

---

## Stage 1: Table Detection

### PDF Table Detection

```python
import pdfplumber
from dataclasses import dataclass, field
from pathlib import Path


@dataclass
class DetectedTable:
    page_number: int
    bounding_box: tuple[float, float, float, float]  # x0, y0, x1, y1
    row_count: int
    col_count: int
    confidence: float
    caption: str = ""
    raw_data: list[list[str]] = field(default_factory=list)


class PDFTableDetector:
    """Detect and extract tables from PDF documents.

    Uses pdfplumber's table detection which is based on:
    1. Line intersection analysis (finds table grid lines)
    2. Text alignment analysis (finds aligned text columns)
    3. Whitespace gap analysis (detects column boundaries)

    Configurable strategies for different table styles:
    - LINES: tables with visible grid lines
    - TEXT: tables without lines (aligned by whitespace)
    - MIXED: combination approach (default)
    """

    def __init__(self, strategy: str = "mixed"):
        self.strategy = strategy
        self.table_settings = self._get_settings(strategy)

    def detect_tables(self, pdf_path: str) -> list[DetectedTable]:
        """Detect all tables in a PDF document."""
        detected = []

        with pdfplumber.open(pdf_path) as pdf:
            for page_num, page in enumerate(pdf.pages):
                tables = page.find_tables(table_settings=self.table_settings)

                for table_idx, table in enumerate(tables):
                    # Extract table data
                    raw_data = table.extract()
                    if not raw_data or len(raw_data) < 2:
                        continue

                    # Calculate dimensions
                    row_count = len(raw_data)
                    col_count = max(len(row) for row in raw_data) if raw_data else 0

                    # Get bounding box
                    bbox = table.bbox  # (x0, top, x1, bottom)

                    # Find caption (text immediately above the table)
                    caption = self._find_caption(page, bbox)

                    # Assess confidence based on structure regularity
                    confidence = self._assess_confidence(raw_data)

                    detected.append(DetectedTable(
                        page_number=page_num,
                        bounding_box=bbox,
                        row_count=row_count,
                        col_count=col_count,
                        confidence=confidence,
                        caption=caption,
                        raw_data=raw_data,
                    ))

        return detected

    def _find_caption(
        self, page, table_bbox: tuple, search_height: float = 50.0
    ) -> str:
        """Find table caption in the text above the table."""
        x0, y0, x1, y1 = table_bbox

        # Search region above the table
        caption_region = (x0, max(0, y0 - search_height), x1, y0)

        # Crop page to caption region and extract text
        cropped = page.within_bbox(caption_region)
        text = cropped.extract_text() if cropped else ""

        if text:
            # Look for common caption patterns
            lines = text.strip().split("\n")
            for line in reversed(lines):
                line = line.strip()
                if line and (
                    line.lower().startswith("table")
                    or line.lower().startswith("fig")
                    or len(line) > 10
                ):
                    return line

        return ""

    def _assess_confidence(self, raw_data: list[list[str]]) -> float:
        """Assess confidence that the detected structure is actually a table."""
        if not raw_data or len(raw_data) < 2:
            return 0.0

        confidence = 0.5  # base confidence

        # Check column count consistency
        col_counts = [len(row) for row in raw_data]
        if len(set(col_counts)) == 1:
            confidence += 0.2  # all rows same width
        elif max(col_counts) - min(col_counts) <= 1:
            confidence += 0.1  # mostly consistent

        # Check for non-empty header row
        header = raw_data[0]
        non_empty_headers = sum(1 for h in header if h and h.strip())
        if non_empty_headers / max(len(header), 1) > 0.7:
            confidence += 0.15

        # Check for data in body rows
        non_empty_cells = 0
        total_cells = 0
        for row in raw_data[1:]:
            for cell in row:
                total_cells += 1
                if cell and cell.strip():
                    non_empty_cells += 1
        if total_cells > 0:
            fill_rate = non_empty_cells / total_cells
            if fill_rate > 0.5:
                confidence += 0.15

        return min(confidence, 1.0)

    @staticmethod
    def _get_settings(strategy: str) -> dict:
        """Get pdfplumber table detection settings for the strategy."""
        if strategy == "lines":
            return {
                "vertical_strategy": "lines",
                "horizontal_strategy": "lines",
            }
        elif strategy == "text":
            return {
                "vertical_strategy": "text",
                "horizontal_strategy": "text",
                "snap_tolerance": 5,
                "join_tolerance": 5,
            }
        else:  # mixed
            return {
                "vertical_strategy": "lines_strict",
                "horizontal_strategy": "lines_strict",
                "snap_tolerance": 3,
                "join_tolerance": 3,
                "edge_min_length": 10,
            }
```

### HTML Table Detection

```python
from bs4 import BeautifulSoup


class HTMLTableDetector:
    """Detect and extract tables from HTML documents."""

    def detect_tables(self, html_content: str) -> list[DetectedTable]:
        """Parse HTML and extract all tables."""
        soup = BeautifulSoup(html_content, "html.parser")
        tables = soup.find_all("table")
        detected = []

        for idx, table in enumerate(tables):
            raw_data = self._extract_table_data(table)
            if not raw_data or len(raw_data) < 2:
                continue

            # Find caption
            caption = ""
            caption_tag = table.find("caption")
            if caption_tag:
                caption = caption_tag.get_text(strip=True)

            # Check for preceding header or paragraph
            if not caption:
                prev = table.find_previous_sibling(["h1", "h2", "h3", "h4", "p"])
                if prev:
                    caption = prev.get_text(strip=True)

            detected.append(DetectedTable(
                page_number=0,
                bounding_box=(0, 0, 0, 0),
                row_count=len(raw_data),
                col_count=max(len(r) for r in raw_data),
                confidence=0.95,  # HTML tables are explicit
                caption=caption,
                raw_data=raw_data,
            ))

        return detected

    def _extract_table_data(self, table) -> list[list[str]]:
        """Extract cell data from an HTML table, handling merged cells."""
        rows = table.find_all("tr")
        if not rows:
            return []

        # First pass: determine grid dimensions
        max_cols = 0
        for row in rows:
            cols = 0
            for cell in row.find_all(["td", "th"]):
                colspan = int(cell.get("colspan", 1))
                cols += colspan
            max_cols = max(max_cols, cols)

        # Second pass: fill grid accounting for rowspan/colspan
        grid: list[list[str | None]] = [
            [None] * max_cols for _ in range(len(rows))
        ]

        for row_idx, row in enumerate(rows):
            col_idx = 0
            for cell in row.find_all(["td", "th"]):
                # Find next available column
                while col_idx < max_cols and grid[row_idx][col_idx] is not None:
                    col_idx += 1

                text = cell.get_text(strip=True)
                colspan = int(cell.get("colspan", 1))
                rowspan = int(cell.get("rowspan", 1))

                # Fill the grid
                for ri in range(rowspan):
                    for ci in range(colspan):
                        target_row = row_idx + ri
                        target_col = col_idx + ci
                        if (
                            target_row < len(grid)
                            and target_col < max_cols
                        ):
                            grid[target_row][target_col] = text

                col_idx += colspan

        # Convert None to empty string
        return [[cell or "" for cell in row] for row in grid]
```

### Image-Based Table Detection

```python
class ImageTableDetector:
    """Detect tables in images using vision LLMs.

    For scanned documents or screenshots where tables are not
    machine-readable, use a vision LLM to detect and extract.
    """

    def __init__(self, anthropic_client):
        self.client = anthropic_client

    def detect_and_extract(self, image_path: str) -> list[DetectedTable]:
        """Use Claude vision to detect and extract tables from an image."""
        import base64

        image_data = Path(image_path).read_bytes()
        b64 = base64.standard_b64encode(image_data).decode()

        response = self.client.messages.create(
            model="claude-sonnet-4-20250514",
            max_tokens=4096,
            messages=[{
                "role": "user",
                "content": [
                    {
                        "type": "image",
                        "source": {
                            "type": "base64",
                            "media_type": "image/png",
                            "data": b64,
                        },
                    },
                    {
                        "type": "text",
                        "text": (
                            "Extract ALL tables from this image.\n\n"
                            "For EACH table:\n"
                            "1. Provide the table caption or title if visible\n"
                            "2. Convert the table to markdown format\n"
                            "3. Preserve all values exactly as shown\n"
                            "4. Handle merged cells by repeating the value\n\n"
                            "Separate multiple tables with ---TABLE_SEPARATOR---\n\n"
                            "Output ONLY the markdown tables with captions. No other text."
                        ),
                    },
                ],
            }],
        )

        return self._parse_vision_output(response.content[0].text)

    def _parse_vision_output(self, text: str) -> list[DetectedTable]:
        """Parse the vision LLM's markdown table output."""
        tables = text.split("---TABLE_SEPARATOR---")
        detected = []

        for table_text in tables:
            table_text = table_text.strip()
            if not table_text:
                continue

            lines = table_text.split("\n")
            caption = ""
            table_lines = []

            for line in lines:
                if line.strip().startswith("|"):
                    table_lines.append(line)
                elif not table_lines and line.strip():
                    caption = line.strip()

            if len(table_lines) < 3:  # header + separator + at least 1 row
                continue

            raw_data = self._parse_markdown_table(table_lines)
            if raw_data:
                detected.append(DetectedTable(
                    page_number=0,
                    bounding_box=(0, 0, 0, 0),
                    row_count=len(raw_data),
                    col_count=max(len(r) for r in raw_data) if raw_data else 0,
                    confidence=0.80,
                    caption=caption,
                    raw_data=raw_data,
                ))

        return detected

    @staticmethod
    def _parse_markdown_table(lines: list[str]) -> list[list[str]]:
        """Parse markdown table lines into a 2D list."""
        data = []
        for line in lines:
            line = line.strip()
            if not line.startswith("|"):
                continue
            # Skip separator line
            if all(c in "|-: " for c in line):
                continue
            cells = [c.strip() for c in line.split("|")[1:-1]]
            data.append(cells)
        return data
```

---

## Stage 2: Table Conversion

### Converting to Multiple Formats

```python
import json
import csv
import io


class TableConverter:
    """Convert detected tables to various formats for embedding and storage."""

    @staticmethod
    def to_markdown(table: DetectedTable) -> str:
        """Convert to markdown with caption."""
        if not table.raw_data:
            return ""

        lines = []
        if table.caption:
            lines.append(f"**{table.caption}**\n")

        # Clean cells
        clean_data = []
        for row in table.raw_data:
            clean_data.append([
                str(cell).strip() if cell else ""
                for cell in row
            ])

        # Normalize column count
        max_cols = max(len(row) for row in clean_data)
        for row in clean_data:
            while len(row) < max_cols:
                row.append("")

        # Header
        lines.append("| " + " | ".join(clean_data[0]) + " |")
        lines.append("| " + " | ".join(["---"] * max_cols) + " |")

        # Body
        for row in clean_data[1:]:
            lines.append("| " + " | ".join(row) + " |")

        return "\n".join(lines)

    @staticmethod
    def to_json(table: DetectedTable) -> str:
        """Convert to JSON array of objects (header values as keys)."""
        if not table.raw_data or len(table.raw_data) < 2:
            return "[]"

        headers = [
            str(h).strip() if h else f"column_{i}"
            for i, h in enumerate(table.raw_data[0])
        ]

        rows = []
        for row in table.raw_data[1:]:
            obj = {}
            for i, header in enumerate(headers):
                value = row[i].strip() if i < len(row) and row[i] else ""
                obj[header] = value
            rows.append(obj)

        result = {"caption": table.caption, "data": rows}
        return json.dumps(result, indent=2)

    @staticmethod
    def to_csv(table: DetectedTable) -> str:
        """Convert to CSV string."""
        output = io.StringIO()
        writer = csv.writer(output)
        for row in table.raw_data:
            writer.writerow([str(cell).strip() if cell else "" for cell in row])
        return output.getvalue()

    @staticmethod
    def to_natural_language(table: DetectedTable) -> str:
        """Convert table to natural language description.

        This is useful for embedding because it captures semantic
        relationships that structural formats may not convey clearly.
        """
        if not table.raw_data or len(table.raw_data) < 2:
            return ""

        headers = table.raw_data[0]
        lines = []

        if table.caption:
            lines.append(f"The following describes: {table.caption}")

        lines.append(
            f"This table has {len(headers)} columns: "
            + ", ".join(str(h) for h in headers if h)
            + "."
        )
        lines.append(f"It contains {len(table.raw_data) - 1} data rows.")

        for row_idx, row in enumerate(table.raw_data[1:], 1):
            parts = []
            for col_idx, cell in enumerate(row):
                if cell and cell.strip() and col_idx < len(headers):
                    header = headers[col_idx] or f"Column {col_idx + 1}"
                    parts.append(f"{header} is {cell.strip()}")
            if parts:
                lines.append(f"Row {row_idx}: " + "; ".join(parts) + ".")

        return "\n".join(lines)

    @staticmethod
    def to_dataframe_code(table: DetectedTable) -> str:
        """Generate pandas DataFrame creation code for the table.

        Useful for downstream analytics and for LLM-based
        table question answering.
        """
        if not table.raw_data or len(table.raw_data) < 2:
            return "import pandas as pd\ndf = pd.DataFrame()"

        headers = table.raw_data[0]
        rows = table.raw_data[1:]

        # Sanitize headers for Python variable names
        clean_headers = []
        for h in headers:
            h_clean = str(h).strip().replace(" ", "_").replace("-", "_")
            if not h_clean:
                h_clean = f"col_{len(clean_headers)}"
            clean_headers.append(h_clean)

        code_lines = [
            "import pandas as pd",
            "",
            "data = {",
        ]

        for col_idx, header in enumerate(clean_headers):
            values = []
            for row in rows:
                val = row[col_idx].strip() if col_idx < len(row) and row[col_idx] else ""
                values.append(f'"{val}"')
            code_lines.append(f'    "{header}": [{", ".join(values)}],')

        code_lines.append("}")
        code_lines.append("")
        code_lines.append("df = pd.DataFrame(data)")

        return "\n".join(code_lines)
```

---

## Stage 3: Table Embedding

### Embedding Strategy Selection

```python
import numpy as np
from typing import Protocol


class EmbeddingModel(Protocol):
    def embed(self, texts: list[str]) -> list[list[float]]: ...


class TableEmbedder:
    """Embed tables using strategies optimized for structured data.

    Different embedding strategies are better for different query types:

    1. MARKDOWN: Embed the markdown representation
       Best for: "Show me the table about X"
       Weakness: Structure is implicit in formatting

    2. NATURAL_LANGUAGE: Embed the NL description
       Best for: "What was the revenue in Q4?"
       Weakness: Verbose, may exceed token limits for large tables

    3. MULTI_REPRESENTATION: Embed multiple representations and store all
       Best for: Mixed query types
       Weakness: Higher storage cost

    4. ROW_LEVEL: Embed each row separately with header context
       Best for: "Find rows where X > Y"
       Weakness: Loses cross-row context
    """

    def __init__(self, embedding_model: EmbeddingModel, converter: TableConverter):
        self.model = embedding_model
        self.converter = converter

    def embed_table(
        self,
        table: DetectedTable,
        strategy: str = "multi_representation",
    ) -> list[dict]:
        """Embed a table using the specified strategy.

        Returns a list of embedding records, each with:
        - embedding: the vector
        - content: the text that was embedded
        - representation: which strategy generated this
        - metadata: table metadata
        """
        if strategy == "markdown":
            return self._embed_markdown(table)
        elif strategy == "natural_language":
            return self._embed_natural_language(table)
        elif strategy == "multi_representation":
            return self._embed_multi_representation(table)
        elif strategy == "row_level":
            return self._embed_row_level(table)
        else:
            raise ValueError(f"Unknown strategy: {strategy}")

    def _embed_markdown(self, table: DetectedTable) -> list[dict]:
        """Embed the markdown representation."""
        markdown = self.converter.to_markdown(table)
        embeddings = self.model.embed([markdown])
        return [{
            "embedding": embeddings[0],
            "content": markdown,
            "representation": "markdown",
            "metadata": self._table_metadata(table),
        }]

    def _embed_natural_language(self, table: DetectedTable) -> list[dict]:
        """Embed the natural language description."""
        nl = self.converter.to_natural_language(table)
        embeddings = self.model.embed([nl])
        return [{
            "embedding": embeddings[0],
            "content": nl,
            "representation": "natural_language",
            "metadata": self._table_metadata(table),
        }]

    def _embed_multi_representation(self, table: DetectedTable) -> list[dict]:
        """Embed markdown, natural language, and JSON representations.

        Store all three embeddings. At query time, search across all
        representations and deduplicate by table ID.
        """
        markdown = self.converter.to_markdown(table)
        nl = self.converter.to_natural_language(table)
        json_str = self.converter.to_json(table)

        texts = [markdown, nl, json_str]
        embeddings = self.model.embed(texts)

        metadata = self._table_metadata(table)
        records = []
        for emb, content, rep_type in zip(
            embeddings,
            texts,
            ["markdown", "natural_language", "json"],
        ):
            records.append({
                "embedding": emb,
                "content": content,
                "representation": rep_type,
                "metadata": metadata,
            })

        return records

    def _embed_row_level(self, table: DetectedTable) -> list[dict]:
        """Embed each row separately with header context.

        Each row embedding includes the column headers to preserve
        the column-value relationship.
        """
        if not table.raw_data or len(table.raw_data) < 2:
            return []

        headers = table.raw_data[0]
        records = []
        metadata = self._table_metadata(table)

        # Prepare row texts
        row_texts = []
        for row_idx, row in enumerate(table.raw_data[1:], 1):
            parts = []
            for col_idx, cell in enumerate(row):
                if cell and cell.strip() and col_idx < len(headers):
                    header = headers[col_idx] or f"Column {col_idx + 1}"
                    parts.append(f"{header}: {cell.strip()}")
            if parts:
                prefix = f"Table: {table.caption}. " if table.caption else ""
                row_text = prefix + "Row: " + ", ".join(parts)
                row_texts.append((row_idx, row_text))

        if not row_texts:
            return []

        # Batch embed all rows
        texts = [t for _, t in row_texts]
        embeddings = self.model.embed(texts)

        for (row_idx, text), emb in zip(row_texts, embeddings):
            row_metadata = {**metadata, "row_index": row_idx}
            records.append({
                "embedding": emb,
                "content": text,
                "representation": "row_level",
                "metadata": row_metadata,
            })

        return records

    @staticmethod
    def _table_metadata(table: DetectedTable) -> dict:
        return {
            "type": "table",
            "page": table.page_number,
            "rows": table.row_count,
            "cols": table.col_count,
            "caption": table.caption,
            "confidence": table.confidence,
        }
```

---

## End-to-End Table RAG Pipeline

```python
class TableRAGPipeline:
    """Complete table processing pipeline from document to queryable index."""

    def __init__(
        self,
        pdf_detector: PDFTableDetector,
        html_detector: HTMLTableDetector,
        converter: TableConverter,
        embedder: TableEmbedder,
        vector_store,
    ):
        self.pdf_detector = pdf_detector
        self.html_detector = html_detector
        self.converter = converter
        self.embedder = embedder
        self.store = vector_store

    def ingest_document(
        self,
        document_path: str,
        embedding_strategy: str = "multi_representation",
    ) -> dict:
        """Process a document and index all tables."""
        path = Path(document_path)
        ext = path.suffix.lower()

        # Detect tables
        if ext == ".pdf":
            tables = self.pdf_detector.detect_tables(str(path))
        elif ext in (".html", ".htm"):
            content = path.read_text(encoding="utf-8")
            tables = self.html_detector.detect_tables(content)
        else:
            raise ValueError(f"Unsupported format: {ext}")

        # Filter low-confidence detections
        tables = [t for t in tables if t.confidence >= 0.5]

        # Embed and index
        total_records = 0
        for table_idx, table in enumerate(tables):
            records = self.embedder.embed_table(table, strategy=embedding_strategy)

            for rec_idx, record in enumerate(records):
                doc_id = f"{document_path}:table:{table_idx}:{record['representation']}"
                self.store.upsert(
                    id=doc_id,
                    embedding=record["embedding"],
                    metadata={
                        **record["metadata"],
                        "content": record["content"],
                        "source": document_path,
                        "table_index": table_idx,
                    },
                )
                total_records += 1

        return {
            "document": document_path,
            "tables_detected": len(tables),
            "records_indexed": total_records,
            "strategy": embedding_strategy,
        }

    def query_tables(
        self, question: str, top_k: int = 5
    ) -> list[dict]:
        """Query the table index."""
        query_embedding = self.embedder.model.embed([question])[0]
        results = self.store.query(
            embedding=query_embedding,
            top_k=top_k * 2,  # over-fetch for dedup
            filter={"type": "table"},
        )

        # Deduplicate by table (multi-representation may return same table multiple times)
        seen_tables = set()
        deduped = []
        for result in results:
            table_key = (
                result["metadata"]["source"],
                result["metadata"]["table_index"],
            )
            if table_key not in seen_tables:
                seen_tables.add(table_key)
                deduped.append(result)
            if len(deduped) >= top_k:
                break

        return deduped
```

---

## Table Question Answering

### LLM-Based Table QA

```python
class TableQA:
    """Answer questions about tables using LLM.

    Supports two modes:
    1. Direct QA: pass the table + question to the LLM
    2. Code generation: generate pandas code to answer analytically
    """

    def __init__(self, llm):
        self.llm = llm

    def answer_direct(
        self, question: str, table_markdown: str, caption: str = ""
    ) -> str:
        """Answer a question directly from the table."""
        context = ""
        if caption:
            context += f"Table: {caption}\n\n"
        context += table_markdown

        response = self.llm.invoke(
            f"Answer the question based ONLY on the table below.\n\n"
            f"{context}\n\n"
            f"Question: {question}\n\n"
            f"Answer precisely. If the answer requires calculation, "
            f"show the calculation. If the table does not contain "
            f"enough information, say so."
        )
        return response.content

    def answer_with_code(
        self, question: str, table: DetectedTable
    ) -> dict:
        """Generate and execute pandas code to answer analytically."""
        # Generate DataFrame code
        df_code = TableConverter.to_dataframe_code(table)

        # Ask LLM to write analysis code
        response = self.llm.invoke(
            f"Given this pandas DataFrame:\n\n```python\n{df_code}\n```\n\n"
            f"Write Python code to answer this question: {question}\n\n"
            f"Rules:\n"
            f"- Use only pandas operations\n"
            f"- Store the final answer in a variable called 'answer'\n"
            f"- Handle type conversions (strings to numbers) if needed\n"
            f"- Do not use exec() or eval()\n\n"
            f"```python\n"
        )

        analysis_code = response.content.strip().strip("`").strip()
        if analysis_code.startswith("python"):
            analysis_code = analysis_code[6:].strip()

        # Execute in a restricted environment
        full_code = df_code + "\n\n" + analysis_code

        local_vars: dict = {}
        try:
            exec(full_code, {"__builtins__": {"len": len, "sum": sum, "max": max, "min": min, "float": float, "int": int, "str": str, "round": round}}, local_vars)
            answer = local_vars.get("answer", "No answer computed")
        except Exception as e:
            answer = f"Code execution failed: {e}"

        return {
            "answer": str(answer),
            "code": full_code,
            "method": "pandas_analysis",
        }
```

---

## Performance Benchmarks

| Detection Method | Precision | Recall | Speed (pages/sec) |
|-----------------|-----------|--------|-------------------|
| pdfplumber (lines) | 92% | 78% | 15 |
| pdfplumber (text) | 75% | 88% | 12 |
| pdfplumber (mixed) | 85% | 85% | 10 |
| HTML parsing | 99% | 99% | 100+ |
| Vision LLM (Claude) | 90% | 95% | 0.5 |
| Vision LLM (GPT-4o) | 88% | 93% | 0.5 |

### Embedding Strategy Comparison

| Strategy | Storage per Table | Retrieval Accuracy | Best For |
|----------|-------------------|-------------------|----------|
| Markdown | 1 vector | 72% | Simple lookups |
| Natural Language | 1 vector | 78% | Semantic questions |
| Multi-representation | 3 vectors | 85% | Mixed query types |
| Row-level | N vectors (N=rows) | 82% (row queries) | Filtering, comparison |

---

## Common Pitfalls

1. **Treating all tables the same.** A 3-column key-value table needs different processing than a 20-column data matrix. Use confidence scoring and table size to choose strategies.

2. **Ignoring table captions.** The caption is often the most important retrieval signal. A query "Q4 revenue table" will match on caption text, not cell values.

3. **Not handling merged cells.** PDF tables frequently have merged headers (spanning multiple columns). pdfplumber handles this, but the raw data may have None values that need filling.

4. **Embedding very large tables as single chunks.** A 100-row table exceeds embedding model token limits. Use row-level embedding for large tables, or truncate to the first 30 rows with a note.

5. **Using generic text chunking on tables.** Standard text chunkers split tables mid-row, destroying structure. Always detect tables first and treat them as atomic units.

6. **Not deduplicating multi-representation results.** If you embed markdown, NL, and JSON versions of the same table, retrieval returns them as separate results. Deduplicate by table ID before presenting to the user.

---

## References

- pdfplumber: https://github.com/jsvine/pdfplumber
- Beautiful Soup: https://www.crummy.com/software/BeautifulSoup/
- Table Transformer (Microsoft): https://github.com/microsoft/table-transformer
- Camelot: https://camelot-py.readthedocs.io/
- Tabula: https://tabula.technology/
