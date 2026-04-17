# PDF Extraction -- Production Pipeline

## Overview / TL;DR

A production PDF ingestion pipeline must handle diverse document types (native text, scanned, mixed), route each document to the optimal extractor, handle failures gracefully, and produce clean, chunked output ready for embedding and indexing. This guide provides a complete async pipeline implementation with document classification, multi-tool routing, table extraction, error recovery, progress tracking, and output validation.

---

## Architecture

```
Input Queue (S3/local/API)
    |
    v
[1] Document Classifier
    |-- native text --> PyMuPDF fast path
    |-- scanned --> OCR path (Docling/Tesseract)
    |-- mixed --> Hybrid path (per-page routing)
    |-- complex layout --> ML path (Docling/Marker)
    |
    v
[2] Extraction Engine (async workers)
    |-- text extraction
    |-- table extraction (separate pass)
    |-- image extraction (optional)
    |-- metadata extraction
    |
    v
[3] Post-Processing
    |-- merge text + tables
    |-- clean/normalize markdown
    |-- validate output quality
    |
    v
[4] Chunking & Output
    |-- structure-aware chunking
    |-- metadata enrichment
    |-- write to vector store / output queue
```

---

## Document Classifier

```python
import fitz
from dataclasses import dataclass
from enum import Enum
from pathlib import Path


class DocType(Enum):
    NATIVE_SIMPLE = "native_simple"        # Single column, no tables
    NATIVE_TABLES = "native_tables"        # Has tables
    NATIVE_COMPLEX = "native_complex"      # Multi-column or complex layout
    SCANNED = "scanned"                    # Image-only pages
    MIXED = "mixed"                        # Mix of native and scanned
    ENCRYPTED = "encrypted"
    CORRUPTED = "corrupted"


@dataclass
class ClassificationResult:
    doc_type: DocType
    num_pages: int
    native_pages: list[int]
    scanned_pages: list[int]
    has_tables: bool
    has_multi_column: bool
    avg_text_density: float
    file_size_mb: float


def classify_document(pdf_path: str) -> ClassificationResult:
    """Classify a PDF to determine optimal extraction strategy."""
    path = Path(pdf_path)
    file_size_mb = path.stat().st_size / (1024 * 1024)

    try:
        doc = fitz.open(pdf_path)
    except Exception:
        return ClassificationResult(
            doc_type=DocType.CORRUPTED,
            num_pages=0,
            native_pages=[],
            scanned_pages=[],
            has_tables=False,
            has_multi_column=False,
            avg_text_density=0.0,
            file_size_mb=file_size_mb,
        )

    if doc.is_encrypted:
        doc.close()
        return ClassificationResult(
            doc_type=DocType.ENCRYPTED,
            num_pages=len(doc),
            native_pages=[],
            scanned_pages=[],
            has_tables=False,
            has_multi_column=False,
            avg_text_density=0.0,
            file_size_mb=file_size_mb,
        )

    native_pages = []
    scanned_pages = []
    text_densities = []
    has_tables = False
    has_multi_column = False

    for page_num, page in enumerate(doc):
        text = page.get_text("text").strip()
        images = page.get_images()
        text_len = len(text)

        # Text density: characters per page area
        area = page.rect.width * page.rect.height
        density = text_len / max(area, 1)
        text_densities.append(density)

        if text_len > 50:
            native_pages.append(page_num)
        elif images and text_len < 20:
            scanned_pages.append(page_num)

        # Table detection heuristic: look for grid-like line patterns
        drawings = page.get_drawings()
        horizontal_lines = sum(
            1 for d in drawings
            for item in d.get("items", [])
            if item[0] == "l" and abs(item[1].y - item[2].y) < 2
        )
        vertical_lines = sum(
            1 for d in drawings
            for item in d.get("items", [])
            if item[0] == "l" and abs(item[1].x - item[2].x) < 2
        )
        if horizontal_lines > 3 and vertical_lines > 3:
            has_tables = True

        # Multi-column detection: check text block X positions
        blocks = page.get_text("dict")["blocks"]
        text_blocks = [b for b in blocks if b["type"] == 0]
        if len(text_blocks) > 3:
            x_positions = sorted(set(
                round(b["bbox"][0] / 50) * 50
                for b in text_blocks
            ))
            if len(x_positions) >= 2:
                gap = x_positions[1] - x_positions[0]
                if gap > 100:
                    has_multi_column = True

    doc.close()

    avg_density = sum(text_densities) / max(len(text_densities), 1)

    # Determine overall type
    if scanned_pages and not native_pages:
        doc_type = DocType.SCANNED
    elif scanned_pages and native_pages:
        doc_type = DocType.MIXED
    elif has_multi_column:
        doc_type = DocType.NATIVE_COMPLEX
    elif has_tables:
        doc_type = DocType.NATIVE_TABLES
    else:
        doc_type = DocType.NATIVE_SIMPLE

    return ClassificationResult(
        doc_type=doc_type,
        num_pages=len(native_pages) + len(scanned_pages),
        native_pages=native_pages,
        scanned_pages=scanned_pages,
        has_tables=has_tables,
        has_multi_column=has_multi_column,
        avg_text_density=avg_density,
        file_size_mb=file_size_mb,
    )
```

---

## Async Extraction Engine

```python
import asyncio
import logging
from dataclasses import dataclass, field
from pathlib import Path
from typing import Callable
from concurrent.futures import ProcessPoolExecutor

logger = logging.getLogger(__name__)


@dataclass
class ExtractionResult:
    file_path: str
    doc_type: str
    method_used: str
    markdown: str
    tables: list[dict] = field(default_factory=list)
    metadata: dict = field(default_factory=dict)
    page_count: int = 0
    error: str | None = None
    processing_seconds: float = 0.0


@dataclass
class PipelineConfig:
    max_workers: int = 4
    max_file_size_mb: float = 200.0
    ocr_dpi: int = 300
    table_extraction: bool = True
    image_extraction: bool = False
    timeout_seconds: int = 300
    fallback_to_ocr: bool = True
    output_format: str = "markdown"


class PDFExtractionPipeline:
    """Async pipeline that classifies and extracts PDFs using optimal tools."""

    def __init__(self, config: PipelineConfig | None = None):
        self.config = config or PipelineConfig()
        self._executor = ProcessPoolExecutor(max_workers=self.config.max_workers)

    async def process_file(self, pdf_path: str) -> ExtractionResult:
        """Process a single PDF file through the pipeline."""
        import time
        start = time.perf_counter()
        path = Path(pdf_path)

        # Validate file
        if not path.exists():
            return ExtractionResult(
                file_path=pdf_path,
                doc_type="unknown",
                method_used="none",
                markdown="",
                error=f"File not found: {pdf_path}",
            )

        file_size_mb = path.stat().st_size / (1024 * 1024)
        if file_size_mb > self.config.max_file_size_mb:
            return ExtractionResult(
                file_path=pdf_path,
                doc_type="unknown",
                method_used="none",
                markdown="",
                error=f"File too large: {file_size_mb:.1f}MB > {self.config.max_file_size_mb}MB",
            )

        try:
            # Step 1: Classify
            classification = await asyncio.get_event_loop().run_in_executor(
                self._executor,
                classify_document,
                pdf_path,
            )

            # Step 2: Route to extractor
            result = await self._extract_by_type(pdf_path, classification)
            result.processing_seconds = time.perf_counter() - start
            return result

        except asyncio.TimeoutError:
            return ExtractionResult(
                file_path=pdf_path,
                doc_type="unknown",
                method_used="none",
                markdown="",
                error=f"Processing timed out after {self.config.timeout_seconds}s",
                processing_seconds=time.perf_counter() - start,
            )
        except Exception as e:
            logger.exception(f"Error processing {pdf_path}")
            return ExtractionResult(
                file_path=pdf_path,
                doc_type="unknown",
                method_used="none",
                markdown="",
                error=str(e),
                processing_seconds=time.perf_counter() - start,
            )

    async def _extract_by_type(
        self,
        pdf_path: str,
        classification: ClassificationResult,
    ) -> ExtractionResult:
        """Route to the best extractor based on document classification."""
        loop = asyncio.get_event_loop()

        if classification.doc_type == DocType.CORRUPTED:
            return ExtractionResult(
                file_path=pdf_path,
                doc_type="corrupted",
                method_used="none",
                markdown="",
                error="PDF is corrupted and cannot be opened",
            )

        if classification.doc_type == DocType.ENCRYPTED:
            return ExtractionResult(
                file_path=pdf_path,
                doc_type="encrypted",
                method_used="none",
                markdown="",
                error="PDF is encrypted; provide password",
            )

        if classification.doc_type == DocType.NATIVE_SIMPLE:
            return await loop.run_in_executor(
                self._executor,
                _extract_pymupdf,
                pdf_path,
                classification,
            )

        if classification.doc_type == DocType.NATIVE_TABLES:
            return await loop.run_in_executor(
                self._executor,
                _extract_with_tables,
                pdf_path,
                classification,
            )

        if classification.doc_type == DocType.NATIVE_COMPLEX:
            return await loop.run_in_executor(
                self._executor,
                _extract_complex,
                pdf_path,
                classification,
            )

        if classification.doc_type == DocType.SCANNED:
            return await loop.run_in_executor(
                self._executor,
                _extract_scanned,
                pdf_path,
                classification,
            )

        if classification.doc_type == DocType.MIXED:
            return await loop.run_in_executor(
                self._executor,
                _extract_mixed,
                pdf_path,
                classification,
            )

        return ExtractionResult(
            file_path=pdf_path,
            doc_type=classification.doc_type.value,
            method_used="none",
            markdown="",
            error=f"Unhandled document type: {classification.doc_type}",
        )

    async def process_batch(
        self,
        pdf_paths: list[str],
        progress_callback: Callable | None = None,
    ) -> list[ExtractionResult]:
        """Process multiple PDFs concurrently."""
        semaphore = asyncio.Semaphore(self.config.max_workers)
        results = []
        completed = 0

        async def _process_one(path: str) -> ExtractionResult:
            nonlocal completed
            async with semaphore:
                result = await asyncio.wait_for(
                    self.process_file(path),
                    timeout=self.config.timeout_seconds,
                )
                completed += 1
                if progress_callback:
                    progress_callback(completed, len(pdf_paths), path)
                return result

        tasks = [_process_one(path) for path in pdf_paths]
        results = await asyncio.gather(*tasks, return_exceptions=True)

        # Convert exceptions to error results
        final_results = []
        for i, result in enumerate(results):
            if isinstance(result, Exception):
                final_results.append(ExtractionResult(
                    file_path=pdf_paths[i],
                    doc_type="unknown",
                    method_used="none",
                    markdown="",
                    error=str(result),
                ))
            else:
                final_results.append(result)

        return final_results

    def close(self):
        self._executor.shutdown(wait=False)
```

---

## Extraction Strategies

```python
import fitz


def _extract_pymupdf(
    pdf_path: str,
    classification: ClassificationResult,
) -> ExtractionResult:
    """Fast extraction for simple native-text PDFs."""
    import pymupdf4llm

    try:
        markdown = pymupdf4llm.to_markdown(
            pdf_path,
            page_chunks=False,
            write_images=False,
            show_progress=False,
        )

        doc = fitz.open(pdf_path)
        metadata = doc.metadata
        page_count = len(doc)
        doc.close()

        return ExtractionResult(
            file_path=pdf_path,
            doc_type=classification.doc_type.value,
            method_used="pymupdf",
            markdown=markdown,
            metadata=metadata,
            page_count=page_count,
        )
    except Exception as e:
        return ExtractionResult(
            file_path=pdf_path,
            doc_type=classification.doc_type.value,
            method_used="pymupdf",
            markdown="",
            error=str(e),
        )


def _extract_with_tables(
    pdf_path: str,
    classification: ClassificationResult,
) -> ExtractionResult:
    """Extract text with PyMuPDF and tables with pdfplumber."""
    import pymupdf4llm
    import pdfplumber

    try:
        # Get base markdown
        markdown = pymupdf4llm.to_markdown(
            pdf_path,
            page_chunks=False,
            write_images=False,
            show_progress=False,
        )

        # Extract tables separately
        tables = []
        with pdfplumber.open(pdf_path) as pdf:
            for page_num, page in enumerate(pdf.pages):
                page_tables = page.extract_tables()
                for table_idx, table in enumerate(page_tables):
                    if table and len(table) > 1:
                        md_table = _table_to_markdown(table)
                        tables.append({
                            "page": page_num + 1,
                            "table_index": table_idx,
                            "markdown": md_table,
                            "rows": len(table),
                            "cols": len(table[0]) if table[0] else 0,
                        })

        doc = fitz.open(pdf_path)
        page_count = len(doc)
        doc.close()

        return ExtractionResult(
            file_path=pdf_path,
            doc_type=classification.doc_type.value,
            method_used="pymupdf+pdfplumber",
            markdown=markdown,
            tables=tables,
            page_count=page_count,
        )
    except Exception as e:
        return ExtractionResult(
            file_path=pdf_path,
            doc_type=classification.doc_type.value,
            method_used="pymupdf+pdfplumber",
            markdown="",
            error=str(e),
        )


def _extract_complex(
    pdf_path: str,
    classification: ClassificationResult,
) -> ExtractionResult:
    """ML-based extraction for complex layouts."""
    try:
        from docling.document_converter import DocumentConverter

        converter = DocumentConverter()
        result = converter.convert(pdf_path)
        doc = result.document

        markdown = doc.export_to_markdown()
        metadata = doc.export_to_dict().get("metadata", {})

        return ExtractionResult(
            file_path=pdf_path,
            doc_type=classification.doc_type.value,
            method_used="docling",
            markdown=markdown,
            metadata=metadata,
            page_count=doc.num_pages(),
        )
    except ImportError:
        # Fallback to PyMuPDF if Docling not installed
        return _extract_pymupdf(pdf_path, classification)
    except Exception as e:
        return ExtractionResult(
            file_path=pdf_path,
            doc_type=classification.doc_type.value,
            method_used="docling",
            markdown="",
            error=str(e),
        )


def _extract_scanned(
    pdf_path: str,
    classification: ClassificationResult,
) -> ExtractionResult:
    """OCR-based extraction for scanned PDFs."""
    try:
        from docling.document_converter import DocumentConverter
        from docling.datamodel.pipeline_options import PdfPipelineOptions

        options = PdfPipelineOptions()
        options.do_ocr = True
        options.do_table_structure = True

        converter = DocumentConverter()
        result = converter.convert(pdf_path)
        doc = result.document

        return ExtractionResult(
            file_path=pdf_path,
            doc_type=classification.doc_type.value,
            method_used="docling_ocr",
            markdown=doc.export_to_markdown(),
            page_count=doc.num_pages(),
        )
    except ImportError:
        # Fallback to PyMuPDF + Tesseract
        return _extract_with_tesseract(pdf_path, classification)
    except Exception as e:
        return ExtractionResult(
            file_path=pdf_path,
            doc_type=classification.doc_type.value,
            method_used="docling_ocr",
            markdown="",
            error=str(e),
        )


def _extract_mixed(
    pdf_path: str,
    classification: ClassificationResult,
) -> ExtractionResult:
    """Per-page routing for mixed native/scanned PDFs."""
    doc = fitz.open(pdf_path)
    all_text = []

    for page_num in range(len(doc)):
        page = doc[page_num]

        if page_num in classification.native_pages:
            text = page.get_text("text")
        else:
            # OCR this page
            pix = page.get_pixmap(dpi=300)
            img_bytes = pix.tobytes("png")
            try:
                from PIL import Image
                import pytesseract
                import io

                image = Image.open(io.BytesIO(img_bytes))
                text = pytesseract.image_to_string(image)
            except ImportError:
                text = "[OCR unavailable for scanned page]"

        all_text.append(f"<!-- Page {page_num + 1} -->\n\n{text}")

    page_count = len(doc)
    doc.close()

    return ExtractionResult(
        file_path=pdf_path,
        doc_type=classification.doc_type.value,
        method_used="pymupdf+tesseract_mixed",
        markdown="\n\n---\n\n".join(all_text),
        page_count=page_count,
    )


def _extract_with_tesseract(
    pdf_path: str,
    classification: ClassificationResult,
) -> ExtractionResult:
    """Fallback OCR using PyMuPDF + Tesseract."""
    doc = fitz.open(pdf_path)
    all_text = []

    for page in doc:
        pix = page.get_pixmap(dpi=300)
        img_bytes = pix.tobytes("png")

        try:
            from PIL import Image
            import pytesseract
            import io

            image = Image.open(io.BytesIO(img_bytes))
            text = pytesseract.image_to_string(image)
            all_text.append(text)
        except Exception as e:
            all_text.append(f"[OCR error: {e}]")

    page_count = len(doc)
    doc.close()

    return ExtractionResult(
        file_path=pdf_path,
        doc_type=classification.doc_type.value,
        method_used="pymupdf+tesseract",
        markdown="\n\n".join(all_text),
        page_count=page_count,
    )


def _table_to_markdown(table: list[list]) -> str:
    """Convert a list-of-lists table to markdown."""
    if not table or not table[0]:
        return ""
    headers = table[0]
    md = "| " + " | ".join(str(h or "") for h in headers) + " |\n"
    md += "| " + " | ".join("---" for _ in headers) + " |\n"
    for row in table[1:]:
        md += "| " + " | ".join(str(cell or "") for cell in row) + " |\n"
    return md
```

---

## Output Validation

```python
import re


@dataclass
class ValidationResult:
    is_valid: bool
    warnings: list[str]
    metrics: dict


def validate_extraction(result: ExtractionResult) -> ValidationResult:
    """Validate extraction output quality."""
    warnings = []
    text = result.markdown

    # Check for empty output
    if not text.strip():
        return ValidationResult(
            is_valid=False,
            warnings=["Extraction produced empty output"],
            metrics={"char_count": 0},
        )

    char_count = len(text)
    word_count = len(text.split())

    # Check text density (characters per page)
    if result.page_count > 0:
        chars_per_page = char_count / result.page_count
        if chars_per_page < 100:
            warnings.append(
                f"Low text density: {chars_per_page:.0f} chars/page "
                f"(expected >500 for text-heavy docs)"
            )

    # Check for encoding garbage
    garbage_chars = sum(1 for c in text if 0xE000 <= ord(c) <= 0xF8FF)
    garbage_ratio = garbage_chars / max(char_count, 1)
    if garbage_ratio > 0.01:
        warnings.append(
            f"Encoding issues detected: {garbage_ratio:.1%} private-use characters"
        )

    # Check for repeated characters (extraction artifact)
    repeated = re.findall(r'(.)\1{20,}', text)
    if repeated:
        warnings.append(
            f"Repeated character sequences detected ({len(repeated)} occurrences)"
        )

    # Check for excessive whitespace
    whitespace_ratio = sum(1 for c in text if c.isspace()) / max(char_count, 1)
    if whitespace_ratio > 0.5:
        warnings.append(
            f"Excessive whitespace: {whitespace_ratio:.1%} of content"
        )

    is_valid = len(warnings) == 0 or all(
        "Low text density" not in w and "Encoding issues" not in w
        for w in warnings
    )

    return ValidationResult(
        is_valid=is_valid,
        warnings=warnings,
        metrics={
            "char_count": char_count,
            "word_count": word_count,
            "page_count": result.page_count,
            "chars_per_page": char_count / max(result.page_count, 1),
            "garbage_ratio": garbage_ratio,
            "whitespace_ratio": whitespace_ratio,
            "tables_found": len(result.tables),
        },
    )
```

---

## Post-Processing and Cleaning

```python
import re


def clean_extracted_markdown(text: str) -> str:
    """Clean and normalize extracted markdown."""
    # Remove excessive blank lines (more than 2 consecutive)
    text = re.sub(r'\n{4,}', '\n\n\n', text)

    # Fix broken words from hyphenation at line breaks
    text = re.sub(r'(\w)-\n(\w)', r'\1\2', text)

    # Remove page headers/footers (common patterns)
    text = re.sub(r'\n\d+\s*\n', '\n', text)  # Standalone page numbers
    text = re.sub(r'\nPage \d+ of \d+\n', '\n', text)

    # Normalize bullet points
    text = re.sub(r'^[*+-]\s', '- ', text, flags=re.MULTILINE)

    # Fix common OCR artifacts
    text = text.replace('\u2019', "'")
    text = text.replace('\u201c', '"')
    text = text.replace('\u201d', '"')
    text = text.replace('\u2013', '--')
    text = text.replace('\u2014', '---')

    # Remove null bytes
    text = text.replace('\x00', '')

    # Normalize whitespace within lines (preserve newlines)
    lines = text.split('\n')
    cleaned_lines = []
    for line in lines:
        cleaned = ' '.join(line.split())
        cleaned_lines.append(cleaned)
    text = '\n'.join(cleaned_lines)

    return text.strip()


def merge_tables_into_markdown(
    markdown: str,
    tables: list[dict],
) -> str:
    """Insert extracted tables into the markdown at appropriate positions."""
    if not tables:
        return markdown

    lines = markdown.split('\n')
    page_markers = {}

    # Find page breaks in the markdown
    for i, line in enumerate(lines):
        match = re.match(r'<!--\s*Page\s+(\d+)\s*-->', line)
        if match:
            page_markers[int(match.group(1))] = i

    # Group tables by page
    tables_by_page = {}
    for table in tables:
        page = table["page"]
        if page not in tables_by_page:
            tables_by_page[page] = []
        tables_by_page[page].append(table["markdown"])

    # Insert tables after page markers (reverse order to preserve indices)
    for page in sorted(tables_by_page.keys(), reverse=True):
        if page in page_markers:
            insert_pos = page_markers[page] + 1
            table_text = "\n\n".join(tables_by_page[page])
            lines.insert(insert_pos, f"\n{table_text}\n")

    return '\n'.join(lines)
```

---

## Running the Pipeline

```python
import asyncio
from pathlib import Path


async def main():
    """Example: process all PDFs in a directory."""
    config = PipelineConfig(
        max_workers=4,
        max_file_size_mb=100.0,
        table_extraction=True,
        timeout_seconds=300,
    )

    pipeline = PDFExtractionPipeline(config)

    # Collect PDF files
    pdf_dir = Path("/data/documents")
    pdf_files = sorted(str(p) for p in pdf_dir.glob("**/*.pdf"))
    print(f"Found {len(pdf_files)} PDFs to process")

    # Progress callback
    def on_progress(completed: int, total: int, path: str):
        print(f"[{completed}/{total}] Processed: {Path(path).name}")

    # Process batch
    results = await pipeline.process_batch(pdf_files, progress_callback=on_progress)

    # Report
    success = [r for r in results if r.error is None]
    failed = [r for r in results if r.error is not None]

    print(f"\nResults: {len(success)} succeeded, {len(failed)} failed")

    for result in failed:
        print(f"  FAILED: {result.file_path} -- {result.error}")

    # Validate successful extractions
    for result in success:
        validation = validate_extraction(result)
        if not validation.is_valid:
            print(f"  WARNING: {result.file_path}")
            for warning in validation.warnings:
                print(f"    - {warning}")

        # Clean and save output
        cleaned = clean_extracted_markdown(result.markdown)
        if result.tables:
            cleaned = merge_tables_into_markdown(cleaned, result.tables)

        output_path = Path("/data/output") / f"{Path(result.file_path).stem}.md"
        output_path.parent.mkdir(parents=True, exist_ok=True)
        output_path.write_text(cleaned, encoding="utf-8")

    pipeline.close()


if __name__ == "__main__":
    asyncio.run(main())
```

---

## Monitoring and Metrics

```python
import time
import json
import logging
from dataclasses import dataclass, field, asdict
from pathlib import Path

logger = logging.getLogger(__name__)


@dataclass
class PipelineMetrics:
    total_files: int = 0
    successful: int = 0
    failed: int = 0
    total_pages: int = 0
    total_tables: int = 0
    total_seconds: float = 0.0
    by_method: dict = field(default_factory=dict)
    by_doc_type: dict = field(default_factory=dict)
    errors: list[dict] = field(default_factory=list)


def compute_metrics(results: list[ExtractionResult]) -> PipelineMetrics:
    """Compute aggregate metrics from pipeline results."""
    metrics = PipelineMetrics(total_files=len(results))

    for result in results:
        if result.error:
            metrics.failed += 1
            metrics.errors.append({
                "file": result.file_path,
                "error": result.error,
                "method": result.method_used,
            })
        else:
            metrics.successful += 1
            metrics.total_pages += result.page_count
            metrics.total_tables += len(result.tables)
            metrics.total_seconds += result.processing_seconds

            # Track by method
            method = result.method_used
            if method not in metrics.by_method:
                metrics.by_method[method] = {"count": 0, "pages": 0, "seconds": 0.0}
            metrics.by_method[method]["count"] += 1
            metrics.by_method[method]["pages"] += result.page_count
            metrics.by_method[method]["seconds"] += result.processing_seconds

            # Track by doc type
            dtype = result.doc_type
            if dtype not in metrics.by_doc_type:
                metrics.by_doc_type[dtype] = {"count": 0, "pages": 0}
            metrics.by_doc_type[dtype]["count"] += 1
            metrics.by_doc_type[dtype]["pages"] += result.page_count

    return metrics


def save_metrics(metrics: PipelineMetrics, output_path: str):
    """Save metrics to JSON for monitoring dashboards."""
    Path(output_path).write_text(
        json.dumps(asdict(metrics), indent=2, default=str),
        encoding="utf-8",
    )
    logger.info(
        f"Pipeline complete: {metrics.successful}/{metrics.total_files} files, "
        f"{metrics.total_pages} pages in {metrics.total_seconds:.1f}s"
    )
```

---

## Common Pitfalls

1. **Processing all PDFs with the same extractor.** Classification-based routing gives 2-10x better quality at similar cost.
2. **Not setting timeouts.** Corrupted or extremely complex PDFs can hang extractors indefinitely. Always use timeouts.
3. **Ignoring extraction validation.** Empty output, encoding garbage, and low text density indicate extraction failures that silently corrupt downstream data.
4. **Running OCR on native-text pages.** Wastes compute and reduces accuracy. Classify per-page, not per-document.
5. **Not cleaning extracted text.** Page numbers, headers/footers, and hyphenation artifacts degrade embedding and retrieval quality.
6. **Loading entire PDFs into memory for batch processing.** Process in page ranges or use streaming for documents over 500 pages.
7. **Not logging extraction methods and metrics.** When downstream quality degrades, you need to trace which extractor produced which chunks.

---

## Production Checklist

- [ ] Document classifier routes each PDF to the optimal extractor
- [ ] Per-page routing handles mixed native/scanned documents
- [ ] Table extraction runs separately and merges into final output
- [ ] Async processing with configurable concurrency and timeouts
- [ ] Output validation catches empty results, encoding issues, and low density
- [ ] Post-processing cleans page numbers, headers, hyphenation, and whitespace
- [ ] Error handling isolates failures per-file (one bad PDF does not stop the batch)
- [ ] Metrics track success rate, throughput, method distribution, and error types
- [ ] Retry logic with fallback extractors for transient failures
- [ ] Output is written as clean markdown ready for chunking

---

## References

- PyMuPDF documentation -- https://pymupdf.readthedocs.io/en/latest/
- pdfplumber documentation -- https://github.com/jsvine/pdfplumber
- Docling documentation -- https://ds4sd.github.io/docling/
- asyncio patterns -- https://docs.python.org/3/library/asyncio.html
- ProcessPoolExecutor -- https://docs.python.org/3/library/concurrent.futures.html
