# Unstructured.io -- Production Pipeline

## Overview / TL;DR

This guide provides a complete production pipeline for batch document processing using Unstructured, covering multi-format file routing, strategy selection, parallel processing, chunking, error recovery, and output to vector stores. The pipeline handles PDFs, DOCX, PPTX, HTML, images, and emails through a unified interface with configurable quality/speed tradeoffs.

---

## Architecture

```
File Source (S3 / local / API upload)
    |
    v
[1] File Router (detect format, classify complexity)
    |
    +---> PDF --> Strategy selector (fast/hi_res/ocr_only)
    +---> DOCX/PPTX --> Direct partition (fast)
    +---> HTML --> HTML partition
    +---> Image --> OCR partition
    +---> Email --> Email partition + attachment recursion
    |
    v
[2] Partitioning (parallel workers)
    |
    v
[3] Element Post-Processing
    |-- filter headers/footers
    |-- merge small elements
    |-- normalize metadata
    |
    v
[4] Chunking (by_title)
    |
    v
[5] Output (vector store / JSON / queue)
```

---

## Pipeline Configuration

```python
from dataclasses import dataclass, field
from enum import Enum


class OutputFormat(Enum):
    JSON = "json"
    LANGCHAIN = "langchain"
    LLAMAINDEX = "llamaindex"


@dataclass
class PipelineConfig:
    # Processing
    max_workers: int = 4
    default_strategy: str = "auto"
    pdf_strategy: str = "hi_res"
    hi_res_model: str = "yolox"
    infer_tables: bool = True
    ocr_languages: list[str] = field(default_factory=lambda: ["eng"])

    # File handling
    max_file_size_mb: float = 200.0
    supported_formats: set[str] = field(default_factory=lambda: {
        ".pdf", ".docx", ".doc", ".pptx", ".ppt", ".xlsx", ".xls",
        ".html", ".htm", ".md", ".txt", ".csv", ".eml", ".msg",
        ".png", ".jpg", ".jpeg", ".tiff", ".bmp", ".epub", ".rst",
    })
    process_attachments: bool = True

    # Chunking
    chunk_strategy: str = "by_title"
    max_chunk_chars: int = 1500
    new_after_n_chars: int = 1000
    overlap_chars: int = 150
    combine_under_n_chars: int = 200

    # Output
    output_format: OutputFormat = OutputFormat.JSON
    output_dir: str = "./output"

    # Error handling
    timeout_seconds: int = 300
    max_retries: int = 2
    fallback_strategy: str = "fast"
```

---

## File Router

```python
import logging
from pathlib import Path
from dataclasses import dataclass

logger = logging.getLogger(__name__)

PARTITION_MAP = {
    ".pdf": "unstructured.partition.pdf:partition_pdf",
    ".docx": "unstructured.partition.docx:partition_docx",
    ".doc": "unstructured.partition.doc:partition_doc",
    ".pptx": "unstructured.partition.pptx:partition_pptx",
    ".xlsx": "unstructured.partition.xlsx:partition_xlsx",
    ".html": "unstructured.partition.html:partition_html",
    ".htm": "unstructured.partition.html:partition_html",
    ".md": "unstructured.partition.md:partition_md",
    ".txt": "unstructured.partition.text:partition_text",
    ".csv": "unstructured.partition.csv:partition_csv",
    ".eml": "unstructured.partition.email:partition_email",
    ".msg": "unstructured.partition.msg:partition_msg",
    ".png": "unstructured.partition.image:partition_image",
    ".jpg": "unstructured.partition.image:partition_image",
    ".jpeg": "unstructured.partition.image:partition_image",
    ".tiff": "unstructured.partition.image:partition_image",
    ".epub": "unstructured.partition.epub:partition_epub",
    ".rst": "unstructured.partition.rst:partition_rst",
}


@dataclass
class FileInfo:
    path: str
    format: str
    size_mb: float
    partition_module: str
    partition_function: str
    strategy: str
    needs_ocr: bool


def route_file(file_path: str, config: PipelineConfig) -> FileInfo:
    """Determine how to process a file based on its format and characteristics."""
    path = Path(file_path)
    suffix = path.suffix.lower()
    size_mb = path.stat().st_size / (1024 * 1024)

    if suffix not in config.supported_formats:
        raise ValueError(f"Unsupported format: {suffix}")

    if size_mb > config.max_file_size_mb:
        raise ValueError(f"File too large: {size_mb:.1f}MB > {config.max_file_size_mb}MB")

    partition_ref = PARTITION_MAP.get(suffix)
    if not partition_ref:
        partition_ref = "unstructured.partition.auto:partition"

    module_path, func_name = partition_ref.rsplit(":", 1)

    # Determine strategy
    if suffix == ".pdf":
        strategy = config.pdf_strategy
    elif suffix in {".png", ".jpg", ".jpeg", ".tiff", ".bmp"}:
        strategy = "hi_res"
        needs_ocr = True
    else:
        strategy = "fast"

    needs_ocr = suffix in {".png", ".jpg", ".jpeg", ".tiff", ".bmp"}

    return FileInfo(
        path=file_path,
        format=suffix,
        size_mb=size_mb,
        partition_module=module_path,
        partition_function=func_name,
        strategy=strategy,
        needs_ocr=needs_ocr,
    )
```

---

## Core Processing Engine

```python
import importlib
import logging
import time
from dataclasses import dataclass, field
from concurrent.futures import ProcessPoolExecutor, as_completed
from pathlib import Path

logger = logging.getLogger(__name__)


@dataclass
class ProcessingResult:
    file_path: str
    elements: list = field(default_factory=list)
    chunks: list = field(default_factory=list)
    element_count: int = 0
    chunk_count: int = 0
    processing_seconds: float = 0.0
    strategy_used: str = ""
    error: str | None = None
    warnings: list[str] = field(default_factory=list)


class UnstructuredPipeline:
    """Production pipeline for batch document processing with Unstructured."""

    def __init__(self, config: PipelineConfig):
        self.config = config

    def process_file(self, file_path: str) -> ProcessingResult:
        """Process a single file through the full pipeline."""
        start = time.perf_counter()

        try:
            # Route
            file_info = route_file(file_path, self.config)

            # Partition
            elements = self._partition(file_info)
            if not elements:
                return ProcessingResult(
                    file_path=file_path,
                    processing_seconds=time.perf_counter() - start,
                    strategy_used=file_info.strategy,
                    warnings=["Partitioning returned zero elements"],
                )

            # Post-process
            elements = self._post_process(elements)

            # Chunk
            chunks = self._chunk(elements)

            # Validate
            warnings = self._validate(elements, chunks, file_path)

            return ProcessingResult(
                file_path=file_path,
                elements=elements,
                chunks=chunks,
                element_count=len(elements),
                chunk_count=len(chunks),
                processing_seconds=time.perf_counter() - start,
                strategy_used=file_info.strategy,
                warnings=warnings,
            )

        except Exception as e:
            logger.exception(f"Error processing {file_path}")
            return ProcessingResult(
                file_path=file_path,
                processing_seconds=time.perf_counter() - start,
                error=str(e),
            )

    def _partition(self, file_info: FileInfo) -> list:
        """Partition file with strategy fallback."""
        module = importlib.import_module(file_info.partition_module)
        partition_fn = getattr(module, file_info.partition_function)

        kwargs = {
            "filename": file_info.path,
            "include_page_breaks": True,
            "include_metadata": True,
        }

        if file_info.format == ".pdf":
            kwargs["strategy"] = file_info.strategy
            if file_info.strategy == "hi_res":
                kwargs["hi_res_model_name"] = self.config.hi_res_model
                kwargs["infer_table_structure"] = self.config.infer_tables
            kwargs["ocr_languages"] = self.config.ocr_languages

        elif file_info.needs_ocr:
            kwargs["strategy"] = "hi_res"
            kwargs["ocr_languages"] = self.config.ocr_languages

        try:
            elements = partition_fn(**kwargs)
            if elements:
                return elements
        except Exception as e:
            logger.warning(
                f"Primary strategy {file_info.strategy} failed for "
                f"{file_info.path}: {e}"
            )

        # Fallback
        if file_info.strategy != self.config.fallback_strategy:
            logger.info(f"Falling back to {self.config.fallback_strategy}")
            kwargs["strategy"] = self.config.fallback_strategy
            if "hi_res_model_name" in kwargs:
                del kwargs["hi_res_model_name"]
            if "infer_table_structure" in kwargs:
                del kwargs["infer_table_structure"]
            return partition_fn(**kwargs)

        return []

    def _post_process(self, elements: list) -> list:
        """Filter and clean elements."""
        from unstructured.documents.elements import (
            Header, Footer, PageBreak, UncategorizedText,
        )

        # Remove non-content elements
        filtered = [
            e for e in elements
            if not isinstance(e, (Header, Footer, PageBreak))
        ]

        # Remove very short uncategorized text (likely artifacts)
        filtered = [
            e for e in filtered
            if not (isinstance(e, UncategorizedText) and len(e.text.strip()) < 10)
        ]

        # Clean text content
        for element in filtered:
            # Normalize whitespace
            element.text = " ".join(element.text.split())
            # Remove null bytes
            element.text = element.text.replace("\x00", "")

        # Remove empty elements
        filtered = [e for e in filtered if e.text.strip()]

        return filtered

    def _chunk(self, elements: list) -> list:
        """Chunk elements using configured strategy."""
        from unstructured.chunking.title import chunk_by_title
        from unstructured.chunking.basic import chunk_elements

        if self.config.chunk_strategy == "by_title":
            return chunk_by_title(
                elements,
                max_characters=self.config.max_chunk_chars,
                new_after_n_chars=self.config.new_after_n_chars,
                combine_text_under_n_chars=self.config.combine_under_n_chars,
                overlap=self.config.overlap_chars,
            )
        else:
            return chunk_elements(
                elements,
                max_characters=self.config.max_chunk_chars,
                new_after_n_chars=self.config.new_after_n_chars,
                overlap=self.config.overlap_chars,
            )

    def _validate(
        self,
        elements: list,
        chunks: list,
        file_path: str,
    ) -> list[str]:
        """Validate processing output."""
        warnings = []

        total_chars = sum(len(e.text) for e in elements)
        if total_chars < 50:
            warnings.append(f"Very little text extracted: {total_chars} chars")

        avg_chunk_size = (
            sum(len(c.text) for c in chunks) / len(chunks)
            if chunks else 0
        )
        if avg_chunk_size < 50:
            warnings.append(f"Average chunk is very small: {avg_chunk_size:.0f} chars")
        if avg_chunk_size > self.config.max_chunk_chars:
            warnings.append(f"Chunks exceed max size: {avg_chunk_size:.0f} chars")

        return warnings

    def process_batch(
        self,
        file_paths: list[str],
        progress_callback=None,
    ) -> list[ProcessingResult]:
        """Process multiple files with parallel workers."""
        results = []
        completed = 0

        with ProcessPoolExecutor(max_workers=self.config.max_workers) as executor:
            futures = {
                executor.submit(self.process_file, path): path
                for path in file_paths
            }

            for future in as_completed(futures):
                path = futures[future]
                completed += 1

                try:
                    result = future.result(timeout=self.config.timeout_seconds)
                    results.append(result)
                except Exception as e:
                    results.append(ProcessingResult(
                        file_path=path,
                        error=str(e),
                    ))

                if progress_callback:
                    progress_callback(completed, len(file_paths), path)

        return results
```

---

## Output Formatters

```python
import json
from pathlib import Path


def results_to_json(
    results: list[ProcessingResult],
    output_dir: str,
):
    """Write processing results as JSON files."""
    out = Path(output_dir)
    out.mkdir(parents=True, exist_ok=True)

    for result in results:
        if result.error:
            continue

        stem = Path(result.file_path).stem
        chunks_data = []

        for i, chunk in enumerate(result.chunks):
            chunks_data.append({
                "chunk_id": f"{stem}_{i}",
                "text": chunk.text,
                "metadata": {
                    "source": result.file_path,
                    "chunk_index": i,
                    "total_chunks": len(result.chunks),
                    "element_type": type(chunk).__name__,
                    **(chunk.metadata.to_dict() if hasattr(chunk.metadata, "to_dict") else {}),
                },
            })

        output_path = out / f"{stem}.json"
        output_path.write_text(
            json.dumps(chunks_data, indent=2, default=str),
            encoding="utf-8",
        )


def results_to_langchain(results: list[ProcessingResult]) -> list:
    """Convert results to LangChain Document objects."""
    from langchain_core.documents import Document

    documents = []
    for result in results:
        if result.error:
            continue

        for i, chunk in enumerate(result.chunks):
            documents.append(Document(
                page_content=chunk.text,
                metadata={
                    "source": result.file_path,
                    "chunk_index": i,
                    "strategy": result.strategy_used,
                    "element_type": type(chunk).__name__,
                    "page_number": getattr(chunk.metadata, "page_number", None),
                },
            ))

    return documents


def results_to_llamaindex(results: list[ProcessingResult]) -> list:
    """Convert results to LlamaIndex Document objects."""
    from llama_index.core import Document

    documents = []
    for result in results:
        if result.error:
            continue

        for i, chunk in enumerate(result.chunks):
            documents.append(Document(
                text=chunk.text,
                metadata={
                    "source": result.file_path,
                    "chunk_index": i,
                    "strategy": result.strategy_used,
                },
            ))

    return documents
```

---

## Email Processing with Attachments

```python
import logging
import tempfile
from pathlib import Path

logger = logging.getLogger(__name__)


def process_email_with_attachments(
    email_path: str,
    config: PipelineConfig,
    pipeline: UnstructuredPipeline,
) -> list[ProcessingResult]:
    """Process an email and all its attachments."""
    from unstructured.partition.email import partition_email

    results = []

    # Partition the email body
    try:
        elements = partition_email(
            filename=email_path,
            include_metadata=True,
            process_attachments=False,  # Handle attachments manually
        )

        email_result = ProcessingResult(
            file_path=email_path,
            elements=elements,
            element_count=len(elements),
            strategy_used="email",
        )

        # Add email-specific metadata
        if elements:
            meta = elements[0].metadata
            email_result.warnings.append(
                f"From: {getattr(meta, 'sent_from', 'unknown')}, "
                f"Subject: {getattr(meta, 'subject', 'unknown')}"
            )

        results.append(email_result)
    except Exception as e:
        results.append(ProcessingResult(
            file_path=email_path,
            error=f"Email parsing failed: {e}",
        ))
        return results

    # Process attachments
    if config.process_attachments:
        try:
            import email
            from email import policy

            with open(email_path, "rb") as f:
                msg = email.message_from_binary_file(f, policy=policy.default)

            for part in msg.walk():
                if part.get_content_disposition() == "attachment":
                    filename = part.get_filename()
                    if not filename:
                        continue

                    suffix = Path(filename).suffix.lower()
                    if suffix not in config.supported_formats:
                        logger.info(f"Skipping unsupported attachment: {filename}")
                        continue

                    # Save attachment to temp file
                    with tempfile.NamedTemporaryFile(
                        suffix=suffix, delete=False
                    ) as tmp:
                        tmp.write(part.get_payload(decode=True))
                        tmp_path = tmp.name

                    try:
                        att_result = pipeline.process_file(tmp_path)
                        att_result.file_path = f"{email_path}::{filename}"
                        results.append(att_result)
                    finally:
                        Path(tmp_path).unlink(missing_ok=True)

        except Exception as e:
            logger.warning(f"Attachment processing failed for {email_path}: {e}")

    return results
```

---

## Monitoring and Reporting

```python
import json
from dataclasses import dataclass, field, asdict
from pathlib import Path


@dataclass
class PipelineReport:
    total_files: int = 0
    successful: int = 0
    failed: int = 0
    total_elements: int = 0
    total_chunks: int = 0
    total_seconds: float = 0.0
    by_format: dict = field(default_factory=dict)
    by_strategy: dict = field(default_factory=dict)
    errors: list[dict] = field(default_factory=list)
    warnings: list[dict] = field(default_factory=list)


def generate_report(results: list[ProcessingResult]) -> PipelineReport:
    """Generate a summary report from processing results."""
    report = PipelineReport(total_files=len(results))

    for result in results:
        if result.error:
            report.failed += 1
            report.errors.append({
                "file": result.file_path,
                "error": result.error,
            })
            continue

        report.successful += 1
        report.total_elements += result.element_count
        report.total_chunks += result.chunk_count
        report.total_seconds += result.processing_seconds

        # Track by format
        fmt = Path(result.file_path).suffix.lower()
        if fmt not in report.by_format:
            report.by_format[fmt] = {"count": 0, "elements": 0, "chunks": 0}
        report.by_format[fmt]["count"] += 1
        report.by_format[fmt]["elements"] += result.element_count
        report.by_format[fmt]["chunks"] += result.chunk_count

        # Track by strategy
        strat = result.strategy_used
        if strat not in report.by_strategy:
            report.by_strategy[strat] = {"count": 0, "seconds": 0.0}
        report.by_strategy[strat]["count"] += 1
        report.by_strategy[strat]["seconds"] += result.processing_seconds

        # Collect warnings
        if result.warnings:
            report.warnings.append({
                "file": result.file_path,
                "warnings": result.warnings,
            })

    return report


def save_report(report: PipelineReport, output_path: str):
    """Save report to JSON."""
    Path(output_path).write_text(
        json.dumps(asdict(report), indent=2, default=str),
        encoding="utf-8",
    )
```

---

## Complete Pipeline Runner

```python
import logging
from pathlib import Path

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)


def run_pipeline(
    input_dir: str,
    output_dir: str,
    config: PipelineConfig | None = None,
):
    """Run the full pipeline on a directory of documents."""
    config = config or PipelineConfig()
    config.output_dir = output_dir

    pipeline = UnstructuredPipeline(config)

    # Discover files
    input_path = Path(input_dir)
    file_paths = []
    for fmt in config.supported_formats:
        file_paths.extend(str(p) for p in input_path.rglob(f"*{fmt}"))

    file_paths.sort()
    logger.info(f"Found {len(file_paths)} files to process")

    if not file_paths:
        logger.warning("No supported files found")
        return

    # Process
    def on_progress(completed: int, total: int, path: str):
        logger.info(f"[{completed}/{total}] {Path(path).name}")

    results = pipeline.process_batch(file_paths, progress_callback=on_progress)

    # Output
    results_to_json(results, output_dir)

    # Report
    report = generate_report(results)
    save_report(report, str(Path(output_dir) / "pipeline_report.json"))

    logger.info(
        f"Pipeline complete: {report.successful}/{report.total_files} files, "
        f"{report.total_chunks} chunks, {report.total_seconds:.1f}s total"
    )

    if report.errors:
        logger.warning(f"{report.failed} files failed:")
        for err in report.errors:
            logger.warning(f"  {err['file']}: {err['error']}")


if __name__ == "__main__":
    run_pipeline(
        input_dir="/data/documents",
        output_dir="/data/output",
        config=PipelineConfig(
            max_workers=4,
            pdf_strategy="hi_res",
            chunk_strategy="by_title",
            max_chunk_chars=1500,
        ),
    )
```

---

## Common Pitfalls

1. **Not filtering headers and footers before chunking.** Page headers/footers get mixed into content chunks, degrading retrieval quality.
2. **Using hi_res for all formats.** Only PDFs and images benefit from hi_res. DOCX, HTML, and Markdown should use fast (their structure is already explicit).
3. **Processing attachments inline with emails.** Email attachments should be extracted and processed separately to avoid timeout and memory issues.
4. **Not setting combine_under_n_chars.** Without it, short elements (titles, single-line paragraphs) become individual tiny chunks that produce noisy embeddings.
5. **Ignoring table HTML.** When infer_table_structure is enabled, the table's structural HTML is in metadata.text_as_html. The element's text field is a flat rendering.
6. **Running the pipeline without timeouts.** Corrupted files or extremely complex PDFs can hang workers indefinitely.
7. **Not tracking strategy usage in output metadata.** When downstream quality issues arise, you need to know which strategy produced each chunk.

---

## Production Checklist

- [ ] File router handles all expected formats with appropriate strategies
- [ ] PDF strategy selection matches document complexity
- [ ] Headers, footers, and page breaks are filtered before chunking
- [ ] Chunking uses by_title with combine_under_n_chars to prevent tiny chunks
- [ ] Email attachments are extracted and processed recursively
- [ ] Timeouts prevent hung workers on corrupted files
- [ ] Fallback strategies catch primary strategy failures
- [ ] Output includes source file, chunk index, strategy, and element type metadata
- [ ] Pipeline report tracks success/failure rates, format distribution, and processing time
- [ ] Parallel workers are configured for available hardware (CPU count for fast, GPU for hi_res)

---

## References

- Unstructured documentation -- https://docs.unstructured.io/
- Unstructured chunking -- https://docs.unstructured.io/open-source/core-functionality/chunking
- Unstructured connectors -- https://docs.unstructured.io/open-source/ingest/overview
- ProcessPoolExecutor -- https://docs.python.org/3/library/concurrent.futures.html
