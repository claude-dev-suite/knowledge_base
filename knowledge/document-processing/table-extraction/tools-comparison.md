# Table Extraction -- Tools Comparison

## Overview / TL;DR

Choosing the right table extraction tool requires matching the tool's strengths to your table types. Bordered tables with visible grid lines are easy for rule-based tools. Borderless tables with whitespace alignment need ML models or careful tuning. Scanned tables require OCR plus structure recognition. This guide provides a systematic accuracy comparison across table types, extraction speed benchmarks, feature matrices, and a decision framework with production benchmark code.

---

## Accuracy Matrix by Table Type

### Table Type Taxonomy

| Table Type | Description | Difficulty | Best Tool(s) |
|-----------|-------------|------------|-------------|
| Bordered (full grid) | All cells have visible borders | Easy | Camelot lattice, pdfplumber |
| Partial borders | Horizontal lines only, no vertical | Medium | pdfplumber, Camelot stream |
| Borderless | Whitespace-aligned columns | Hard | Textract, TableTransformer, Docling |
| Merged cells | Row/column spans | Hard | Textract, Azure Doc AI |
| Multi-page | Table spans page breaks | Hard | Custom stitching required |
| Nested | Tables within tables | Very Hard | Custom parsing |
| Rotated/skewed | Non-axis-aligned tables | Very Hard | OCR preprocessing required |
| Colored backgrounds | Cell backgrounds instead of borders | Medium | ML-based tools |

### Detailed Accuracy Comparison

Based on evaluation across representative document sets:

**Bordered tables (full grid lines)**:

| Tool | Cell Accuracy | Structure Accuracy | Speed | Notes |
|------|-------------|-------------------|-------|-------|
| Camelot (lattice) | 95-99% | 97-99% | Fast | Best for clean borders |
| pdfplumber (lines) | 93-98% | 95-98% | Medium | Good default |
| Tabula (lattice) | 90-96% | 93-97% | Medium | Requires Java |
| Docling | 92-97% | 94-97% | Slow | ML overhead unnecessary |
| Textract | 93-98% | 95-98% | Medium | Cloud cost |
| TableTransformer | 88-95% | 90-95% | Slow | GPU needed |

**Borderless tables (whitespace-aligned)**:

| Tool | Cell Accuracy | Structure Accuracy | Speed | Notes |
|------|-------------|-------------------|-------|-------|
| Camelot (stream) | 60-80% | 65-82% | Fast | Needs manual columns |
| pdfplumber (text) | 55-75% | 60-78% | Medium | Unreliable |
| Tabula (stream) | 58-78% | 62-80% | Medium | Slightly better than Camelot |
| Docling | 80-92% | 82-93% | Slow | ML layout analysis helps |
| Textract | 85-95% | 87-96% | Medium | Best cloud option |
| Azure Doc AI | 83-94% | 85-95% | Medium | Close to Textract |
| TableTransformer | 78-90% | 80-92% | Slow | Good open-source option |

**Tables with merged cells**:

| Tool | Cell Accuracy | Merge Detection | Speed |
|------|-------------|----------------|-------|
| Camelot | 40-60% | None | Fast |
| pdfplumber | 45-65% | None | Medium |
| Docling | 70-85% | Partial | Slow |
| Textract | 80-92% | Yes | Medium |
| Azure Doc AI | 78-90% | Yes | Medium |
| TableTransformer | 65-80% | Partial | Slow |

---

## Benchmark Framework

```python
import json
import time
from dataclasses import dataclass, field
from pathlib import Path
from typing import Callable


@dataclass
class TableBenchmarkResult:
    tool: str
    file: str
    table_type: str
    tables_detected: int
    tables_expected: int
    cell_accuracy: float
    structure_accuracy: float
    time_seconds: float
    error: str | None = None


def compute_cell_accuracy(
    extracted: list[list[str]],
    ground_truth: list[list[str]],
) -> float:
    """Compute cell-level accuracy between extracted and ground truth tables."""
    if not ground_truth:
        return 1.0 if not extracted else 0.0
    if not extracted:
        return 0.0

    total_cells = 0
    correct = 0

    max_rows = max(len(extracted), len(ground_truth))
    max_cols = max(
        max((len(r) for r in ground_truth), default=0),
        max((len(r) for r in extracted), default=0),
    )

    for row_idx in range(max_rows):
        for col_idx in range(max_cols):
            total_cells += 1

            ext_val = ""
            if row_idx < len(extracted) and col_idx < len(extracted[row_idx]):
                ext_val = str(extracted[row_idx][col_idx] or "").strip().lower()

            gt_val = ""
            if row_idx < len(ground_truth) and col_idx < len(ground_truth[row_idx]):
                gt_val = str(ground_truth[row_idx][col_idx] or "").strip().lower()

            if ext_val == gt_val:
                correct += 1

    return correct / max(total_cells, 1)


def compute_structure_accuracy(
    extracted: list[list[str]],
    ground_truth: list[list[str]],
) -> float:
    """Check if row/column counts match and cells are in correct positions."""
    if not ground_truth:
        return 1.0 if not extracted else 0.0

    gt_rows = len(ground_truth)
    gt_cols = len(ground_truth[0]) if ground_truth else 0
    ext_rows = len(extracted)
    ext_cols = len(extracted[0]) if extracted else 0

    row_score = 1.0 - abs(gt_rows - ext_rows) / max(gt_rows, 1)
    col_score = 1.0 - abs(gt_cols - ext_cols) / max(gt_cols, 1)

    # Penalize but do not zero out for dimension mismatches
    return max(0.0, (row_score + col_score) / 2)


def benchmark_tool(
    tool_name: str,
    extract_fn: Callable,
    test_files: list[dict],
) -> list[TableBenchmarkResult]:
    """Run a single tool against all test files."""
    results = []

    for test in test_files:
        pdf_path = test["pdf_path"]
        ground_truth = test["ground_truth"]  # list of list[list[str]]
        table_type = test["table_type"]
        expected_count = len(ground_truth)

        start = time.perf_counter()
        try:
            extracted_tables = extract_fn(pdf_path)
            elapsed = time.perf_counter() - start

            # Match extracted tables to ground truth (order-based)
            cell_accuracies = []
            struct_accuracies = []

            for i, gt in enumerate(ground_truth):
                if i < len(extracted_tables):
                    ext = extracted_tables[i]
                    cell_accuracies.append(compute_cell_accuracy(ext, gt))
                    struct_accuracies.append(compute_structure_accuracy(ext, gt))
                else:
                    cell_accuracies.append(0.0)
                    struct_accuracies.append(0.0)

            results.append(TableBenchmarkResult(
                tool=tool_name,
                file=Path(pdf_path).name,
                table_type=table_type,
                tables_detected=len(extracted_tables),
                tables_expected=expected_count,
                cell_accuracy=sum(cell_accuracies) / max(len(cell_accuracies), 1),
                structure_accuracy=sum(struct_accuracies) / max(len(struct_accuracies), 1),
                time_seconds=round(elapsed, 3),
            ))
        except Exception as e:
            results.append(TableBenchmarkResult(
                tool=tool_name,
                file=Path(pdf_path).name,
                table_type=table_type,
                tables_detected=0,
                tables_expected=expected_count,
                cell_accuracy=0.0,
                structure_accuracy=0.0,
                time_seconds=time.perf_counter() - start,
                error=str(e),
            ))

    return results
```

---

## Feature Comparison Matrix

| Feature | Camelot | Tabula | pdfplumber | TableTransformer | Textract | Azure Doc AI | Docling |
|---------|---------|--------|------------|-----------------|----------|-------------|---------|
| Bordered tables | Excellent | Good | Very Good | Good | Very Good | Very Good | Very Good |
| Borderless tables | Fair | Fair | Fair | Good | Excellent | Very Good | Good |
| Merged cells | None | None | None | Partial | Yes | Yes | Partial |
| Scanned docs | No | No | No | Yes (images) | Yes | Yes | Yes |
| Multi-page | No | No | No | No | No | Partial | No |
| Confidence score | Yes | No | No | Yes | Yes | Yes | No |
| HTML output | No | No | No | No | No | Yes | Yes |
| Visual debug | Yes | No | Yes | No | No | No | No |
| Area selection | No | Yes | Yes | Yes (bbox) | No | No | No |
| GPU required | No | No | No | Recommended | No | No | Optional |
| Cloud required | No | No | No | No | Yes | Yes | No |
| License | MIT | MIT | MIT | MIT | Pay-per-use | Pay-per-use | MIT |
| Python-native | Yes | No (Java) | Yes | Yes | Yes (boto3) | Yes | Yes |

---

## Speed Benchmarks

```python
def run_speed_comparison(pdf_path: str) -> dict:
    """Compare extraction speed across tools for a single PDF."""
    import time

    results = {}

    # Camelot lattice
    try:
        import camelot
        start = time.perf_counter()
        camelot.read_pdf(pdf_path, flavor="lattice")
        results["camelot_lattice"] = round(time.perf_counter() - start, 3)
    except Exception as e:
        results["camelot_lattice"] = f"error: {e}"

    # pdfplumber
    try:
        import pdfplumber
        start = time.perf_counter()
        with pdfplumber.open(pdf_path) as pdf:
            for page in pdf.pages:
                page.extract_tables()
        results["pdfplumber"] = round(time.perf_counter() - start, 3)
    except Exception as e:
        results["pdfplumber"] = f"error: {e}"

    # Tabula
    try:
        import tabula
        start = time.perf_counter()
        tabula.read_pdf(pdf_path, pages="all", lattice=True, multiple_tables=True)
        results["tabula"] = round(time.perf_counter() - start, 3)
    except Exception as e:
        results["tabula"] = f"error: {e}"

    return results
```

**Typical speed results (10-page document with 5 tables)**:

| Tool | Time (seconds) | Relative Speed |
|------|---------------|---------------|
| pdfplumber | 0.8-1.5 | 1x (baseline) |
| Camelot (lattice) | 1.0-2.0 | 0.8x |
| Tabula | 2.0-4.0 | 0.4x |
| Docling | 5.0-15.0 | 0.1x |
| TableTransformer | 3.0-10.0 | 0.15x |
| Textract | 2.0-5.0 | 0.3x (+ network) |

---

## Cost Comparison

| Tool | License | Per-Page Cost | 10K Pages/Month | Notes |
|------|---------|--------------|----------------|-------|
| Camelot | MIT | $0 (compute) | ~$0.50 | CPU only |
| Tabula | MIT | $0 (compute) | ~$0.50 | Needs Java |
| pdfplumber | MIT | $0 (compute) | ~$0.50 | Pure Python |
| TableTransformer | MIT | $0 (compute) | ~$5-20 | GPU recommended |
| Docling | MIT | $0 (compute) | ~$5-20 | GPU optional |
| Textract | Pay-per-use | $0.015/page | $150 | Tables feature |
| Azure Doc AI | Pay-per-use | $0.01/page | $100 | Layout model |
| Google Doc AI | Pay-per-use | $0.01-0.065/page | $100-650 | Custom models |

---

## Decision Framework

```
What type of tables do you have?
  |
  +---> Bordered (visible grid lines)
  |     |
  |     +---> Digital PDF (not scanned)
  |     |     --> Camelot (lattice) or pdfplumber
  |     |
  |     +---> Scanned document
  |           --> Textract or Docling (with OCR)
  |
  +---> Borderless (whitespace-aligned)
  |     |
  |     +---> Budget allows cloud APIs
  |     |     --> Textract or Azure Document Intelligence
  |     |
  |     +---> Must be local
  |           --> TableTransformer or Docling
  |
  +---> Merged cells
  |     --> Textract or Azure Document Intelligence (only tools with merge detection)
  |
  +---> Mixed table types in same document
  |     --> Cascade: try Camelot lattice -> pdfplumber -> ML fallback
  |
  +---> Very high volume (100K+ pages)
        --> Camelot/pdfplumber for bordered + Textract for borderless
```

---

## Common Pitfalls

1. **Benchmarking on clean test data only.** Real documents have merged cells, spanning headers, footnotes inside tables, and partial borders.
2. **Not trying lattice before stream.** Many borderless-looking tables actually have faint borders detectable by lattice mode with adjusted thresholds.
3. **Comparing cloud and local tools without accounting for network latency.** Textract may report faster processing but total latency includes upload/download.
4. **Ignoring the cost of GPU instances for ML tools.** A g4dn.xlarge costs $0.526/hour. At low volumes, cloud APIs are cheaper than GPU self-hosting.
5. **Using a single tool for all table types.** No tool handles every table type well. Use a cascade of tools from fast+cheap to slow+accurate.
6. **Not validating against ground truth.** Extraction that looks correct visually may have wrong cells, swapped columns, or missing rows.

---

## References

- Camelot documentation -- https://camelot-py.readthedocs.io/en/master/
- Tabula-py documentation -- https://tabula-py.readthedocs.io/en/latest/
- pdfplumber documentation -- https://github.com/jsvine/pdfplumber
- TableTransformer -- https://github.com/microsoft/table-transformer
- Amazon Textract tables -- https://docs.aws.amazon.com/textract/latest/dg/how-it-works-tables.html
- Azure Document Intelligence -- https://learn.microsoft.com/en-us/azure/ai-services/document-intelligence/
- PubTables-1M benchmark -- https://arxiv.org/abs/2110.00061
