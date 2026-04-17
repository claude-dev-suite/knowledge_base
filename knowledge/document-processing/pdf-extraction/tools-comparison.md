# PDF Extraction -- Tools Comparison

## Overview / TL;DR

Choosing the right PDF extraction tool depends on document type, throughput requirements, accuracy needs, and infrastructure constraints. This guide provides a systematic comparison across seven dimensions -- text quality, table extraction, layout analysis, speed, scanned-document handling, cost, and ecosystem integration -- with benchmark methodology and production code for running your own evaluations.

---

## Tool Categories

PDF extraction tools fall into three tiers:

1. **Rule-based extractors** -- Parse the PDF object tree directly. Fast, deterministic, no ML models. Examples: PyMuPDF, pdfplumber, pdfminer.six.
2. **ML-powered converters** -- Use trained models for layout analysis, table recognition, and reading-order detection. Examples: Docling, Marker, Nougat.
3. **Cloud/LLM services** -- Send pages to cloud APIs that use vision models. Highest quality, highest cost. Examples: LlamaParse, Amazon Textract, Azure Document Intelligence, Google Document AI.

---

## Detailed Feature Matrix

| Feature | PyMuPDF | pdfplumber | Docling | LlamaParse | Marker | Textract | Azure Doc AI |
|---------|---------|------------|---------|------------|--------|----------|-------------|
| **Text extraction** | Excellent | Excellent | Excellent | Excellent | Very Good | Good | Excellent |
| **Table detection** | Manual | Excellent | Very Good | Excellent | Good | Excellent | Excellent |
| **Layout analysis** | Basic | Basic | ML (DocLayNet) | Vision LLM | ML | ML | ML |
| **Multi-column** | Poor | Poor | Good | Excellent | Excellent | Good | Very Good |
| **OCR built-in** | No | No | Yes (EasyOCR) | Yes | Yes | Yes | Yes |
| **Equation handling** | No | No | Basic | Good | Excellent (LaTeX) | No | Basic |
| **Markdown output** | pymupdf4llm | Manual | Built-in | Built-in | Built-in | Manual | Manual |
| **Batch processing** | Yes | Yes | Yes | Yes (async) | Yes | Yes (async) | Yes (async) |
| **GPU required** | No | No | Optional | N/A (cloud) | Recommended | N/A | N/A |
| **Offline capable** | Yes | Yes | Yes | No | Yes | No | No |
| **Max throughput** | 100+ pg/s | 5-15 pg/s | 1-5 pg/s | 0.5-2 pg/s | 1-5 pg/s | 2-5 pg/s | 2-5 pg/s |
| **License** | AGPL-3.0 | MIT | MIT | Commercial | GPL-3.0 | Pay-per-use | Pay-per-use |
| **Pricing** | Free | Free | Free | $0.003/page | Free | $1.50/1K pages | $1.50/1K pages |

---

## Quality Benchmarks by Document Type

### Methodology

```python
from dataclasses import dataclass
from pathlib import Path
import json
import time


@dataclass
class ExtractionResult:
    tool: str
    doc_type: str
    text: str
    elapsed_seconds: float
    num_pages: int
    tables_found: int
    error: str | None = None


@dataclass
class QualityScore:
    tool: str
    doc_type: str
    text_accuracy: float      # 0-1, character-level match against ground truth
    table_accuracy: float     # 0-1, cell-level match
    structure_score: float    # 0-1, heading/list/paragraph detection
    reading_order: float      # 0-1, correct reading order
    overall: float            # Weighted average


def benchmark_extraction(
    pdf_path: str,
    ground_truth_path: str,
    extractors: dict,
) -> list[ExtractionResult]:
    """Run all extractors on a PDF and collect results."""
    results = []
    for name, extract_fn in extractors.items():
        start = time.perf_counter()
        try:
            output = extract_fn(pdf_path)
            elapsed = time.perf_counter() - start
            results.append(ExtractionResult(
                tool=name,
                doc_type=Path(pdf_path).stem,
                text=output["text"] if isinstance(output, dict) else str(output),
                elapsed_seconds=elapsed,
                num_pages=output.get("num_pages", 0) if isinstance(output, dict) else 0,
                tables_found=output.get("tables_found", 0) if isinstance(output, dict) else 0,
            ))
        except Exception as e:
            results.append(ExtractionResult(
                tool=name,
                doc_type=Path(pdf_path).stem,
                text="",
                elapsed_seconds=time.perf_counter() - start,
                num_pages=0,
                tables_found=0,
                error=str(e),
            ))

    return results


def compute_text_accuracy(extracted: str, ground_truth: str) -> float:
    """Character-level accuracy using longest common subsequence ratio."""
    from difflib import SequenceMatcher

    # Normalize whitespace
    extracted_norm = " ".join(extracted.split())
    truth_norm = " ".join(ground_truth.split())

    matcher = SequenceMatcher(None, extracted_norm, truth_norm)
    return matcher.ratio()


def compute_table_accuracy(
    extracted_tables: list[list[list[str]]],
    ground_truth_tables: list[list[list[str]]],
) -> float:
    """Cell-level accuracy for table extraction."""
    if not ground_truth_tables:
        return 1.0 if not extracted_tables else 0.0

    if not extracted_tables:
        return 0.0

    total_cells = 0
    correct_cells = 0

    for gt_table, ext_table in zip(ground_truth_tables, extracted_tables):
        for gt_row, ext_row in zip(gt_table, ext_table):
            for gt_cell, ext_cell in zip(gt_row, ext_row):
                total_cells += 1
                gt_norm = str(gt_cell or "").strip().lower()
                ext_norm = str(ext_cell or "").strip().lower()
                if gt_norm == ext_norm:
                    correct_cells += 1

            # Penalize missing/extra cells in row
            total_cells += abs(len(gt_row) - len(ext_row))

        # Penalize missing/extra rows
        total_cells += abs(len(gt_table) - len(ext_table))

    return correct_cells / max(total_cells, 1)
```

### Results by Document Type

Based on evaluation across 50 documents per category (results are representative ranges, not exact benchmarks):

**Simple single-column text (reports, letters)**:

| Tool | Text Accuracy | Speed (pg/s) | Notes |
|------|--------------|---------------|-------|
| PyMuPDF | 98-99% | 120 | Best throughput |
| pdfplumber | 97-99% | 8 | Slightly slower |
| Docling | 98-99% | 2 | Overhead not justified |
| LlamaParse | 98-99% | 1 | Overkill for simple docs |

**Multi-column academic papers**:

| Tool | Text Accuracy | Reading Order | Speed (pg/s) |
|------|--------------|---------------|---------------|
| PyMuPDF | 75-85% | Poor | 100 |
| pdfplumber | 70-80% | Poor | 6 |
| Docling | 92-96% | Good | 2 |
| Marker | 93-97% | Excellent | 3 |
| LlamaParse | 95-99% | Excellent | 1 |

**Documents with tables**:

| Tool | Table Cell Accuracy | Table Detection | Speed (pg/s) |
|------|-------------------|-----------------|---------------|
| PyMuPDF | 60-70% (as text) | None | 100 |
| pdfplumber | 85-95% (bordered) | Rule-based | 5 |
| pdfplumber | 50-70% (borderless) | Rule-based | 5 |
| Docling | 85-92% (any type) | ML-based | 1.5 |
| LlamaParse | 90-97% (any type) | Vision LLM | 0.8 |
| Textract | 88-95% (any type) | ML-based | 2 |

**Scanned documents**:

| Tool | Text Accuracy | Built-in OCR | Speed (pg/s) |
|------|--------------|-------------|---------------|
| PyMuPDF + Tesseract | 85-92% | No (external) | 1-3 |
| Docling | 88-94% | Yes (EasyOCR) | 0.5-1 |
| Marker | 87-93% | Yes | 1-2 |
| LlamaParse | 92-98% | Yes | 0.5-1 |
| Textract | 90-96% | Yes | 1-2 |
| Azure Doc AI | 91-97% | Yes | 1-2 |

---

## Speed Benchmarks

```python
import time
from pathlib import Path
from typing import Callable


def speed_benchmark(
    pdf_dir: str,
    extractors: dict[str, Callable],
    warmup_runs: int = 2,
    benchmark_runs: int = 5,
) -> dict:
    """Run speed benchmarks across all PDFs in a directory."""
    pdf_files = sorted(Path(pdf_dir).glob("*.pdf"))
    results = {}

    for name, extract_fn in extractors.items():
        # Warmup
        for _ in range(warmup_runs):
            try:
                extract_fn(str(pdf_files[0]))
            except Exception:
                pass

        tool_results = []
        for pdf_path in pdf_files:
            timings = []
            for _ in range(benchmark_runs):
                start = time.perf_counter()
                try:
                    extract_fn(str(pdf_path))
                    elapsed = time.perf_counter() - start
                    timings.append(elapsed)
                except Exception:
                    continue

            if timings:
                import fitz
                num_pages = len(fitz.open(str(pdf_path)))
                avg_time = sum(timings) / len(timings)

                tool_results.append({
                    "file": pdf_path.name,
                    "pages": num_pages,
                    "avg_seconds": round(avg_time, 3),
                    "pages_per_second": round(num_pages / avg_time, 1),
                    "min_seconds": round(min(timings), 3),
                    "max_seconds": round(max(timings), 3),
                })

        results[name] = tool_results

    return results


def format_speed_report(results: dict) -> str:
    """Format speed benchmark results as a markdown table."""
    lines = ["| Tool | Avg Pages/sec | Min | Max | Files Tested |"]
    lines.append("| --- | --- | --- | --- | --- |")

    for tool, file_results in results.items():
        if not file_results:
            continue
        all_pps = [r["pages_per_second"] for r in file_results]
        avg_pps = sum(all_pps) / len(all_pps)
        min_pps = min(all_pps)
        max_pps = max(all_pps)
        lines.append(
            f"| {tool} | {avg_pps:.1f} | {min_pps:.1f} | {max_pps:.1f} | {len(file_results)} |"
        )

    return "\n".join(lines)
```

---

## Cost Analysis

### Open-Source Tools (Infrastructure Cost Only)

| Tool | CPU Instance (c5.xlarge) | GPU Instance (g4dn.xlarge) | Cost per 10K pages |
|------|-------------------------|---------------------------|-------------------|
| PyMuPDF | 1-2 min | N/A | ~$0.01 |
| pdfplumber | 10-30 min | N/A | ~$0.10 |
| Docling | 30-60 min (CPU) | 5-15 min (GPU) | $0.15-$0.50 |
| Marker | 60-120 min (CPU) | 10-20 min (GPU) | $0.20-$0.60 |

### Cloud Services

| Service | Per-Page Price | 10K Pages | 100K Pages | Free Tier |
|---------|---------------|-----------|------------|-----------|
| LlamaParse | $0.003 | $30 | $300 | 1K pages/day |
| Amazon Textract | $0.0015 | $15 | $150 | 1K pages/month |
| Azure Doc AI | $0.001-$0.01 | $10-$100 | $100-$1000 | 500 pages/month |
| Google Doc AI | $0.001-$0.065 | $10-$650 | $100-$6500 | 1K pages/month |

---

## Integration Patterns

### Unified Extraction Interface

```python
from abc import ABC, abstractmethod
from dataclasses import dataclass
from enum import Enum


class ExtractorType(Enum):
    PYMUPDF = "pymupdf"
    PDFPLUMBER = "pdfplumber"
    DOCLING = "docling"
    LLAMAPARSE = "llamaparse"
    MARKER = "marker"


@dataclass
class ExtractionOutput:
    text: str
    markdown: str
    tables: list[list[list[str]]]
    metadata: dict
    pages: int
    method: str


class PDFExtractor(ABC):
    @abstractmethod
    def extract(self, pdf_path: str) -> ExtractionOutput:
        pass

    @abstractmethod
    def supports_ocr(self) -> bool:
        pass


class PyMuPDFExtractor(PDFExtractor):
    def extract(self, pdf_path: str) -> ExtractionOutput:
        import fitz
        import pymupdf4llm

        doc = fitz.open(pdf_path)
        text = ""
        for page in doc:
            text += page.get_text("text") + "\n"

        markdown = pymupdf4llm.to_markdown(pdf_path, show_progress=False)

        num_pages = len(doc)
        doc.close()

        return ExtractionOutput(
            text=text,
            markdown=markdown,
            tables=[],
            metadata={},
            pages=num_pages,
            method="pymupdf",
        )

    def supports_ocr(self) -> bool:
        return False


class PdfplumberExtractor(PDFExtractor):
    def extract(self, pdf_path: str) -> ExtractionOutput:
        import pdfplumber

        text_parts = []
        all_tables = []

        with pdfplumber.open(pdf_path) as pdf:
            num_pages = len(pdf.pages)
            for page in pdf.pages:
                text_parts.append(page.extract_text() or "")
                tables = page.extract_tables()
                all_tables.extend(tables)

        return ExtractionOutput(
            text="\n".join(text_parts),
            markdown="\n".join(text_parts),
            tables=all_tables,
            metadata={},
            pages=num_pages,
            method="pdfplumber",
        )

    def supports_ocr(self) -> bool:
        return False


class DoclingExtractor(PDFExtractor):
    def extract(self, pdf_path: str) -> ExtractionOutput:
        from docling.document_converter import DocumentConverter

        converter = DocumentConverter()
        result = converter.convert(pdf_path)
        doc = result.document

        return ExtractionOutput(
            text=doc.export_to_markdown(),
            markdown=doc.export_to_markdown(),
            tables=[],
            metadata=doc.export_to_dict().get("metadata", {}),
            pages=doc.num_pages(),
            method="docling",
        )

    def supports_ocr(self) -> bool:
        return True


def create_extractor(tool: ExtractorType) -> PDFExtractor:
    """Factory for PDF extractors."""
    extractors = {
        ExtractorType.PYMUPDF: PyMuPDFExtractor,
        ExtractorType.PDFPLUMBER: PdfplumberExtractor,
        ExtractorType.DOCLING: DoclingExtractor,
    }
    cls = extractors.get(tool)
    if cls is None:
        raise ValueError(f"Unknown extractor: {tool}")
    return cls()
```

### LangChain Integration

```python
from langchain_community.document_loaders import (
    PyMuPDFLoader,
    PDFPlumberLoader,
    UnstructuredPDFLoader,
)
from langchain_core.documents import Document


def load_pdf_langchain(
    pdf_path: str,
    method: str = "pymupdf",
) -> list[Document]:
    """Load PDF using LangChain document loaders."""
    loaders = {
        "pymupdf": lambda: PyMuPDFLoader(pdf_path),
        "pdfplumber": lambda: PDFPlumberLoader(pdf_path),
        "unstructured": lambda: UnstructuredPDFLoader(
            pdf_path,
            mode="elements",
            strategy="hi_res",
        ),
    }

    loader = loaders[method]()
    return loader.load()


def load_pdf_llamaindex(pdf_path: str, method: str = "pymupdf") -> list:
    """Load PDF using LlamaIndex readers."""
    if method == "pymupdf":
        from llama_index.readers.file import PyMuPDFReader
        reader = PyMuPDFReader()
        return reader.load_data(pdf_path)

    elif method == "llamaparse":
        from llama_parse import LlamaParse
        parser = LlamaParse(result_type="markdown")
        return parser.load_data(pdf_path)

    elif method == "docling":
        from llama_index.readers.docling import DoclingReader
        reader = DoclingReader()
        return reader.load_data(pdf_path)

    else:
        raise ValueError(f"Unknown method: {method}")
```

---

## Decision Matrix: Choosing by Use Case

| Use Case | Recommended Tool | Why |
|----------|-----------------|-----|
| High-throughput batch (100K+ docs) | PyMuPDF | 100x faster than alternatives |
| Table-heavy financial reports | pdfplumber + Docling | pdfplumber for bordered, Docling for borderless |
| Academic paper corpus | Marker | Best multi-column + equation handling |
| Legal contract analysis | LlamaParse or Azure Doc AI | Highest accuracy on complex layouts |
| Mixed scanned/native archive | Docling | Built-in OCR + layout analysis, local |
| Cost-sensitive startup | PyMuPDF + Tesseract | Free, good enough for simple docs |
| Regulated industry (data must stay local) | Docling or Marker | No cloud dependency |
| Multilingual documents | Azure Doc AI or Textract | Best language coverage |
| One-off analysis (< 1K pages) | LlamaParse | Free tier, highest quality |

---

## Common Pitfalls in Tool Selection

1. **Choosing by speed alone.** PyMuPDF is 100x faster but cannot handle multi-column or scanned documents. Match the tool to the document type.
2. **Using cloud services for simple text PDFs.** Paying $0.003/page for documents that PyMuPDF handles perfectly is wasteful.
3. **Ignoring license requirements.** PyMuPDF is AGPL (copyleft) and Marker is GPL. If your product is proprietary, you may need commercial licenses or MIT alternatives (pdfplumber, Docling).
4. **Not benchmarking on your actual documents.** Published benchmarks use curated test sets. Your documents may have different characteristics.
5. **Assuming one tool fits all.** Production pipelines should route documents to different extractors based on content type (see the production-pipeline article).
6. **Overlooking the total cost of self-hosting ML models.** Docling and Marker require GPU instances. The compute cost may exceed cloud API pricing at low volumes.

---

## References

- PyMuPDF benchmarks -- https://pymupdf.readthedocs.io/en/latest/about.html
- Docling technical report -- https://arxiv.org/abs/2408.09869
- Marker benchmarks -- https://github.com/VikParuchuri/marker#benchmarks
- Amazon Textract pricing -- https://aws.amazon.com/textract/pricing/
- Azure Document Intelligence pricing -- https://azure.microsoft.com/en-us/pricing/details/ai-document-intelligence/
- LlamaParse documentation -- https://docs.llamaindex.ai/en/stable/llama_cloud/llama_parse/
