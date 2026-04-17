# Office Documents -- Production Pipeline

## Overview / TL;DR

A production office document pipeline must detect file formats, route to appropriate extractors, convert to clean markdown, handle legacy formats, and produce chunked output for RAG systems. This guide provides a complete multi-format router with DOCX, PPTX, XLSX extraction, legacy format conversion, markdown normalization, structure-aware chunking, and batch processing.

---

## Architecture

```
Input Files (.docx, .pptx, .xlsx, .doc, .xls, .ppt)
    |
    v
[1] Format Detection & Validation
    |-- detect by extension and magic bytes
    |-- check file integrity
    |-- convert legacy formats (.doc -> .docx)
    |
    v
[2] Format-Specific Extraction
    |-- DOCX: python-docx -> paragraphs + tables
    |-- PPTX: python-pptx -> slides + notes
    |-- XLSX: openpyxl -> sheets + data
    |
    v
[3] Markdown Conversion
    |-- headings from styles
    |-- tables to markdown tables
    |-- inline formatting preserved
    |
    v
[4] Chunking
    |-- heading-aware for DOCX
    |-- slide-based for PPTX
    |-- sheet/table-based for XLSX
    |
    v
[5] Output (JSON, vector store)
```

---

## Format Detection and Validation

```python
import struct
from pathlib import Path
from dataclasses import dataclass

MAGIC_BYTES = {
    b"PK\x03\x04": "ooxml",  # ZIP-based (docx, pptx, xlsx)
    b"\xd0\xcf\x11\xe0": "ole2",  # Legacy Office (doc, xls, ppt)
}


@dataclass
class FileValidation:
    path: str
    extension: str
    detected_format: str
    is_valid: bool
    is_legacy: bool
    error: str | None = None


def validate_office_file(file_path: str) -> FileValidation:
    """Validate an office document file."""
    path = Path(file_path)
    ext = path.suffix.lower()

    if not path.exists():
        return FileValidation(
            path=file_path, extension=ext, detected_format="unknown",
            is_valid=False, is_legacy=False, error="File not found",
        )

    # Read magic bytes
    try:
        with open(file_path, "rb") as f:
            header = f.read(8)
    except Exception as e:
        return FileValidation(
            path=file_path, extension=ext, detected_format="unknown",
            is_valid=False, is_legacy=False, error=str(e),
        )

    detected = "unknown"
    for magic, fmt in MAGIC_BYTES.items():
        if header[:len(magic)] == magic:
            detected = fmt
            break

    is_legacy = detected == "ole2"
    is_valid = detected in ("ooxml", "ole2")

    return FileValidation(
        path=file_path,
        extension=ext,
        detected_format=detected,
        is_valid=is_valid,
        is_legacy=is_legacy,
    )
```

---

## Core Pipeline

```python
import logging
import time
import subprocess
import tempfile
from dataclasses import dataclass, field
from pathlib import Path
from concurrent.futures import ProcessPoolExecutor, as_completed

logger = logging.getLogger(__name__)


@dataclass
class ProcessingResult:
    file_path: str
    format: str
    markdown: str
    metadata: dict = field(default_factory=dict)
    chunks: list[dict] = field(default_factory=list)
    processing_seconds: float = 0.0
    error: str | None = None
    warnings: list[str] = field(default_factory=list)


class OfficePipeline:
    """Production pipeline for office document processing."""

    def __init__(
        self,
        chunk_size: int = 1000,
        chunk_overlap: int = 200,
        convert_legacy: bool = True,
        max_workers: int = 4,
    ):
        self.chunk_size = chunk_size
        self.chunk_overlap = chunk_overlap
        self.convert_legacy = convert_legacy
        self.max_workers = max_workers

    def process_file(self, file_path: str) -> ProcessingResult:
        """Process a single office document."""
        start = time.perf_counter()

        validation = validate_office_file(file_path)
        if not validation.is_valid:
            return ProcessingResult(
                file_path=file_path,
                format=validation.extension,
                markdown="",
                error=validation.error or "Invalid file",
                processing_seconds=time.perf_counter() - start,
            )

        ext = validation.extension

        # Convert legacy formats
        actual_path = file_path
        if validation.is_legacy and self.convert_legacy:
            try:
                actual_path = self._convert_legacy(file_path)
                ext = Path(actual_path).suffix.lower()
            except Exception as e:
                return ProcessingResult(
                    file_path=file_path,
                    format=ext,
                    markdown="",
                    error=f"Legacy conversion failed: {e}",
                    processing_seconds=time.perf_counter() - start,
                )

        try:
            if ext == ".docx":
                result = self._process_docx(actual_path, file_path)
            elif ext == ".pptx":
                result = self._process_pptx(actual_path, file_path)
            elif ext in (".xlsx", ".xls"):
                result = self._process_xlsx(actual_path, file_path)
            else:
                result = ProcessingResult(
                    file_path=file_path,
                    format=ext,
                    markdown="",
                    error=f"Unsupported format: {ext}",
                )

            result.processing_seconds = time.perf_counter() - start
            return result

        except Exception as e:
            logger.exception(f"Error processing {file_path}")
            return ProcessingResult(
                file_path=file_path,
                format=ext,
                markdown="",
                error=str(e),
                processing_seconds=time.perf_counter() - start,
            )

    def _process_docx(self, file_path: str, original_path: str) -> ProcessingResult:
        """Process a Word document."""
        from docx import Document

        doc = Document(file_path)
        markdown = docx_to_markdown(file_path)

        # Extract metadata
        props = doc.core_properties
        metadata = {
            "title": props.title or "",
            "author": props.author or "",
            "created": str(props.created) if props.created else "",
            "modified": str(props.modified) if props.modified else "",
        }

        # Chunk with heading awareness
        chunks = self._chunk_markdown(markdown, original_path)

        return ProcessingResult(
            file_path=original_path,
            format="docx",
            markdown=markdown,
            metadata=metadata,
            chunks=chunks,
        )

    def _process_pptx(self, file_path: str, original_path: str) -> ProcessingResult:
        """Process a PowerPoint presentation."""
        markdown = pptx_to_markdown(file_path)
        data = extract_pptx(file_path)

        # Chunk by slide
        chunks = []
        for slide in data["slides"]:
            slide_text = f"## Slide {slide['slide_number']}: {slide['title']}\n\n"
            slide_text += "\n".join(slide["texts"])
            if slide["notes"]:
                slide_text += f"\n\nSpeaker notes: {slide['notes']}"

            if len(slide_text.strip()) > 50:
                chunks.append({
                    "text": slide_text,
                    "metadata": {
                        "source": original_path,
                        "slide_number": slide["slide_number"],
                        "title": slide["title"],
                        "chunk_type": "slide",
                    },
                })

        return ProcessingResult(
            file_path=original_path,
            format="pptx",
            markdown=markdown,
            chunks=chunks,
        )

    def _process_xlsx(self, file_path: str, original_path: str) -> ProcessingResult:
        """Process an Excel spreadsheet."""
        markdown = xlsx_to_markdown(file_path)
        data = extract_xlsx(file_path)

        # Chunk by sheet
        chunks = []
        for sheet in data["sheets"]:
            if not sheet["data"]:
                continue

            sheet_md = f"## Sheet: {sheet['name']}\n\n"
            headers = sheet["data"][0]
            sheet_md += "| " + " | ".join(headers) + " |\n"
            sheet_md += "| " + " | ".join("---" for _ in headers) + " |\n"

            # Chunk large sheets into row groups
            rows = sheet["data"][1:]
            rows_per_chunk = max(1, self.chunk_size // 100)

            for i in range(0, len(rows), rows_per_chunk):
                batch = rows[i:i + rows_per_chunk]
                chunk_text = sheet_md
                for row in batch:
                    padded = row + [""] * (len(headers) - len(row))
                    chunk_text += "| " + " | ".join(padded[:len(headers)]) + " |\n"

                chunks.append({
                    "text": chunk_text,
                    "metadata": {
                        "source": original_path,
                        "sheet": sheet["name"],
                        "row_start": i + 1,
                        "row_end": i + len(batch),
                        "chunk_type": "sheet_rows",
                    },
                })

        return ProcessingResult(
            file_path=original_path,
            format="xlsx",
            markdown=markdown,
            chunks=chunks,
        )

    def _chunk_markdown(self, markdown: str, source: str) -> list[dict]:
        """Chunk markdown with heading awareness."""
        from langchain_text_splitters import (
            MarkdownHeaderTextSplitter,
            RecursiveCharacterTextSplitter,
        )

        headers = [("#", "h1"), ("##", "h2"), ("###", "h3")]
        md_splitter = MarkdownHeaderTextSplitter(
            headers_to_split_on=headers, strip_headers=False,
        )
        header_chunks = md_splitter.split_text(markdown)

        size_splitter = RecursiveCharacterTextSplitter(
            chunk_size=self.chunk_size,
            chunk_overlap=self.chunk_overlap,
        )
        final_chunks = size_splitter.split_documents(header_chunks)

        return [
            {
                "text": chunk.page_content,
                "metadata": {
                    "source": source,
                    **chunk.metadata,
                    "chunk_type": "docx_section",
                },
            }
            for chunk in final_chunks
            if len(chunk.page_content.strip()) > 50
        ]

    def _convert_legacy(self, file_path: str) -> str:
        """Convert legacy Office format using LibreOffice."""
        ext = Path(file_path).suffix.lower()
        target_ext = {
            ".doc": "docx",
            ".xls": "xlsx",
            ".ppt": "pptx",
        }.get(ext, "docx")

        tmp_dir = tempfile.mkdtemp()
        result = subprocess.run(
            [
                "libreoffice", "--headless",
                "--convert-to", target_ext,
                "--outdir", tmp_dir,
                file_path,
            ],
            capture_output=True, text=True, timeout=120,
        )

        if result.returncode != 0:
            raise RuntimeError(f"LibreOffice failed: {result.stderr}")

        converted = Path(tmp_dir) / f"{Path(file_path).stem}.{target_ext}"
        if not converted.exists():
            raise RuntimeError("Converted file not found")

        return str(converted)

    def process_batch(
        self,
        file_paths: list[str],
        progress_callback=None,
    ) -> list[ProcessingResult]:
        """Process multiple files with parallel workers."""
        results = []
        completed = 0

        with ProcessPoolExecutor(max_workers=self.max_workers) as executor:
            futures = {
                executor.submit(self.process_file, p): p
                for p in file_paths
            }

            for future in as_completed(futures):
                completed += 1
                try:
                    result = future.result(timeout=120)
                    results.append(result)
                except Exception as e:
                    path = futures[future]
                    results.append(ProcessingResult(
                        file_path=path, format="unknown",
                        markdown="", error=str(e),
                    ))

                if progress_callback:
                    progress_callback(completed, len(file_paths))

        return results
```

---

## Common Pitfalls

1. **Not handling legacy formats.** Production pipelines receive .doc, .xls, and .ppt files. Always provide LibreOffice conversion.
2. **Processing PPTX as flat text.** Slides are discrete units. Chunk per-slide, not by character count.
3. **Not preserving heading hierarchy from DOCX.** Document styles define semantic structure. Use MarkdownHeaderTextSplitter after conversion.
4. **Loading huge Excel files fully in memory.** Use read_only mode and chunk large sheets by row groups.
5. **Ignoring password-protected files.** Check for encryption and fail gracefully with a clear error message.
6. **Not tracking original file path.** After legacy conversion, preserve the original path in metadata for citations.

---

## Production Checklist

- [ ] Format detection works by both extension and magic bytes
- [ ] Legacy formats (.doc, .xls, .ppt) are converted via LibreOffice
- [ ] DOCX extraction preserves headings, tables, and inline formatting
- [ ] PPTX extraction includes slide notes
- [ ] XLSX extraction handles multiple sheets and large files
- [ ] Chunking strategy matches format (heading-aware for DOCX, per-slide for PPTX, row-groups for XLSX)
- [ ] Metadata includes source file, format, author, and creation date
- [ ] Error handling catches corrupt and password-protected files
- [ ] Batch processing uses parallel workers with timeouts
- [ ] Output validation checks for empty results

---

## References

- python-docx -- https://python-docx.readthedocs.io/
- mammoth -- https://github.com/mwilliamson/python-mammoth
- python-pptx -- https://python-pptx.readthedocs.io/
- openpyxl -- https://openpyxl.readthedocs.io/
- LibreOffice CLI -- https://www.libreoffice.org/
