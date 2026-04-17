# Unstructured.io -- Tools Comparison (hi_res vs fast, Local vs API)

## Overview / TL;DR

Unstructured offers multiple processing modes that trade off speed, quality, cost, and infrastructure complexity. This guide provides a systematic comparison of partitioning strategies (fast, hi_res, ocr_only, auto), deployment modes (local open-source vs hosted API), and layout detection models (YOLOX vs Detectron2). It includes benchmark code, cost modeling, and decision frameworks for choosing the right configuration for your pipeline.

---

## Strategy Comparison

### fast vs hi_res -- The Core Tradeoff

The most important decision in an Unstructured pipeline is choosing between the `fast` and `hi_res` strategies. They use fundamentally different approaches.

**fast strategy**:
- Uses pdfminer.six for text extraction (PDF) or format-native parsers (DOCX, HTML)
- Element classification via rule-based heuristics (font size, position, patterns)
- No ML models loaded, no GPU needed
- Cannot handle scanned documents
- Table detection is basic (pattern matching, not structural analysis)

**hi_res strategy**:
- Renders each page as an image
- Runs a layout detection model (YOLOX or Detectron2) to identify regions
- Classifies regions into element types (title, text, table, figure, etc.)
- Runs OCR on image regions
- Uses a table structure model to extract table HTML
- Requires ML model downloads (several GB), benefits from GPU

```python
from unstructured.partition.pdf import partition_pdf
import time


def compare_strategies(pdf_path: str) -> dict:
    """Run both strategies and compare output."""

    # Fast strategy
    start = time.perf_counter()
    fast_elements = partition_pdf(
        filename=pdf_path,
        strategy="fast",
        include_page_breaks=True,
    )
    fast_time = time.perf_counter() - start

    # Hi-res strategy
    start = time.perf_counter()
    hires_elements = partition_pdf(
        filename=pdf_path,
        strategy="hi_res",
        hi_res_model_name="yolox",
        infer_table_structure=True,
        include_page_breaks=True,
    )
    hires_time = time.perf_counter() - start

    return {
        "fast": {
            "elements": len(fast_elements),
            "time_seconds": round(fast_time, 2),
            "types": _count_types(fast_elements),
            "total_chars": sum(len(e.text) for e in fast_elements),
        },
        "hi_res": {
            "elements": len(hires_elements),
            "time_seconds": round(hires_time, 2),
            "types": _count_types(hires_elements),
            "total_chars": sum(len(e.text) for e in hires_elements),
        },
        "speedup": round(hires_time / max(fast_time, 0.01), 1),
    }


def _count_types(elements: list) -> dict:
    """Count elements by type."""
    counts = {}
    for e in elements:
        type_name = type(e).__name__
        counts[type_name] = counts.get(type_name, 0) + 1
    return counts
```

### Detailed Strategy Matrix

| Dimension | fast | hi_res | ocr_only | auto |
|-----------|------|--------|----------|------|
| **Speed** (10-page PDF) | 0.5-2s | 15-60s | 5-20s | Varies |
| **Element classification** | Heuristic | ML model | OCR + heuristic | Adaptive |
| **Table detection** | Pattern match | Layout model | None | Adaptive |
| **Table structure** | Basic text | HTML with cells | None | Adaptive |
| **OCR capability** | None | Built-in | Primary method | If needed |
| **Scanned PDFs** | Fails | Handles well | Primary use case | Auto-detects |
| **Multi-column** | Often wrong | Usually correct | Depends on OCR | Adaptive |
| **Memory usage** | ~100MB | 2-8GB | 500MB-2GB | Varies |
| **GPU benefit** | None | 5-10x speedup | Moderate | Varies |
| **Model downloads** | None | 2-4GB | 500MB | Varies |
| **Accuracy (simple)** | 95%+ | 97%+ | 85-90% | 95%+ |
| **Accuracy (complex)** | 70-85% | 90-97% | 75-85% | 85-95% |

---

## Layout Detection Models

### YOLOX vs Detectron2

```python
from unstructured.partition.pdf import partition_pdf
import time


def compare_models(pdf_path: str) -> dict:
    """Compare YOLOX and Detectron2 layout models."""

    results = {}
    for model in ["yolox", "detectron2_onnx"]:
        start = time.perf_counter()
        elements = partition_pdf(
            filename=pdf_path,
            strategy="hi_res",
            hi_res_model_name=model,
            infer_table_structure=True,
        )
        elapsed = time.perf_counter() - start

        results[model] = {
            "elements": len(elements),
            "time_seconds": round(elapsed, 2),
            "types": _count_types(elements),
            "tables_with_html": sum(
                1 for e in elements
                if hasattr(e.metadata, "text_as_html") and e.metadata.text_as_html
            ),
        }

    return results
```

| Feature | YOLOX | Detectron2 (ONNX) |
|---------|-------|-------------------|
| Speed | Faster (2-3x) | Slower but more precise |
| Table detection | Good | Better on complex tables |
| Figure detection | Good | Slightly better |
| Memory | Lower (~2GB) | Higher (~4GB) |
| ONNX support | Yes | Yes |
| Default | Yes (recommended) | Alternative |
| Best for | General documents | Table-heavy documents |

---

## Local vs Hosted API

### Feature Comparison

| Feature | Local (open-source) | Hosted API |
|---------|-------------------|------------|
| **Formats supported** | All | All |
| **Strategies** | All | All |
| **Page parallelism** | Manual | Built-in (split_pdf_page) |
| **Throughput** | Limited by hardware | Auto-scaling |
| **Chunking** | Library functions | API parameter |
| **Setup complexity** | High (models, deps) | API key only |
| **Data privacy** | Full control | Sent to Unstructured servers |
| **Cost (10K pages/mo)** | Infrastructure only | ~$10 (free tier: 1K pages) |
| **Cost (100K pages/mo)** | $50-200 (compute) | ~$100 |
| **Cost (1M pages/mo)** | $200-1000 (compute) | ~$1000 |
| **Latency** | No network overhead | Network + queue time |
| **Reliability** | Self-managed | 99.9% SLA (paid) |
| **Updates** | Manual | Automatic |

### Local Processing Setup

```python
import os
from pathlib import Path


def setup_local_environment():
    """Configure environment for local Unstructured processing."""
    # Model cache directory
    os.environ["UNSTRUCTURED_HI_RES_MODEL_NAME"] = "yolox"

    # Tesseract path (if not in PATH)
    # os.environ["TESSERACT_CMD"] = "/usr/bin/tesseract"

    # Verify dependencies
    checks = {
        "unstructured": _check_import("unstructured"),
        "pdf support": _check_import("pdfminer"),
        "docx support": _check_import("docx"),
        "ocr support": _check_import("pytesseract"),
        "hi_res support": _check_import("unstructured_inference"),
    }

    for dep, available in checks.items():
        status = "OK" if available else "MISSING"
        print(f"  {dep}: {status}")

    return all(checks.values())


def _check_import(module: str) -> bool:
    try:
        __import__(module)
        return True
    except ImportError:
        return False
```

### API Client Setup

```python
from unstructured_client import UnstructuredClient
from unstructured_client.models import operations, shared


def create_api_client(api_key: str) -> UnstructuredClient:
    """Create and configure the Unstructured API client."""
    return UnstructuredClient(
        api_key_auth=api_key,
        server_url="https://api.unstructuredapp.io",
    )


def partition_via_api(
    client: UnstructuredClient,
    file_path: str,
    strategy: str = "auto",
    chunk: bool = False,
) -> list:
    """Partition a document using the hosted API."""
    with open(file_path, "rb") as f:
        content = f.read()

    params = shared.PartitionParameters(
        files=shared.Files(
            content=content,
            file_name=file_path,
        ),
        strategy=shared.Strategy(strategy),
        languages=["eng"],
        split_pdf_page=True,
        split_pdf_concurrency_level=10,
    )

    if chunk:
        params.chunking_strategy = "by_title"
        params.max_characters = 1500
        params.new_after_n_chars = 1000
        params.overlap = 150

    req = operations.PartitionRequest(partition_parameters=params)
    response = client.general.partition(request=req)
    return response.elements
```

---

## Benchmark Framework

```python
import json
import time
from dataclasses import dataclass, field
from pathlib import Path


@dataclass
class BenchmarkResult:
    file: str
    strategy: str
    elements: int
    total_chars: int
    tables_found: int
    time_seconds: float
    types: dict = field(default_factory=dict)
    error: str | None = None


def run_benchmark_suite(
    pdf_dir: str,
    strategies: list[str] = None,
) -> list[BenchmarkResult]:
    """Benchmark multiple strategies across a directory of PDFs."""
    from unstructured.partition.pdf import partition_pdf
    from unstructured.documents.elements import Table

    if strategies is None:
        strategies = ["fast", "hi_res"]

    pdf_files = sorted(Path(pdf_dir).glob("*.pdf"))
    results = []

    for pdf_path in pdf_files:
        for strategy in strategies:
            try:
                start = time.perf_counter()
                elements = partition_pdf(
                    filename=str(pdf_path),
                    strategy=strategy,
                    hi_res_model_name="yolox" if strategy == "hi_res" else None,
                    infer_table_structure=(strategy == "hi_res"),
                )
                elapsed = time.perf_counter() - start

                results.append(BenchmarkResult(
                    file=pdf_path.name,
                    strategy=strategy,
                    elements=len(elements),
                    total_chars=sum(len(e.text) for e in elements),
                    tables_found=sum(1 for e in elements if isinstance(e, Table)),
                    time_seconds=round(elapsed, 2),
                    types=_count_types(elements),
                ))
            except Exception as e:
                results.append(BenchmarkResult(
                    file=pdf_path.name,
                    strategy=strategy,
                    elements=0,
                    total_chars=0,
                    tables_found=0,
                    time_seconds=0,
                    error=str(e),
                ))

    return results


def format_benchmark_table(results: list[BenchmarkResult]) -> str:
    """Format results as a markdown comparison table."""
    lines = ["| File | Strategy | Elements | Chars | Tables | Time (s) |"]
    lines.append("| --- | --- | --- | --- | --- | --- |")

    for r in sorted(results, key=lambda x: (x.file, x.strategy)):
        if r.error:
            lines.append(f"| {r.file} | {r.strategy} | ERROR | - | - | - |")
        else:
            lines.append(
                f"| {r.file} | {r.strategy} | {r.elements} | "
                f"{r.total_chars} | {r.tables_found} | {r.time_seconds} |"
            )

    return "\n".join(lines)
```

---

## Performance Optimization

### Local Processing Optimization

```python
from concurrent.futures import ProcessPoolExecutor, as_completed
from pathlib import Path
from unstructured.partition.auto import partition
import logging

logger = logging.getLogger(__name__)


def parallel_partition(
    file_paths: list[str],
    strategy: str = "fast",
    max_workers: int = 4,
) -> dict[str, list]:
    """Process multiple files in parallel using process pool."""
    results = {}

    with ProcessPoolExecutor(max_workers=max_workers) as executor:
        futures = {
            executor.submit(_partition_file, path, strategy): path
            for path in file_paths
        }

        for future in as_completed(futures):
            path = futures[future]
            try:
                elements = future.result(timeout=300)
                results[path] = elements
                logger.info(f"Processed {path}: {len(elements)} elements")
            except Exception as e:
                logger.error(f"Failed {path}: {e}")
                results[path] = []

    return results


def _partition_file(file_path: str, strategy: str) -> list:
    """Worker function for parallel processing."""
    from unstructured.partition.auto import partition
    return partition(filename=file_path, strategy=strategy)
```

### API Throughput Optimization

```python
import asyncio
import aiohttp
from pathlib import Path


async def batch_partition_api(
    file_paths: list[str],
    api_key: str,
    max_concurrent: int = 10,
    strategy: str = "auto",
) -> dict[str, list]:
    """Process files through the API with controlled concurrency."""
    from unstructured_client import UnstructuredClient
    from unstructured_client.models import operations, shared

    client = UnstructuredClient(
        api_key_auth=api_key,
        server_url="https://api.unstructuredapp.io",
    )

    semaphore = asyncio.Semaphore(max_concurrent)
    results = {}

    async def process_one(file_path: str):
        async with semaphore:
            with open(file_path, "rb") as f:
                content = f.read()

            req = operations.PartitionRequest(
                partition_parameters=shared.PartitionParameters(
                    files=shared.Files(
                        content=content,
                        file_name=file_path,
                    ),
                    strategy=shared.Strategy(strategy),
                    split_pdf_page=True,
                    split_pdf_concurrency_level=5,
                ),
            )

            loop = asyncio.get_event_loop()
            response = await loop.run_in_executor(
                None,
                lambda: client.general.partition(request=req),
            )
            return file_path, response.elements

    tasks = [process_one(fp) for fp in file_paths]
    for coro in asyncio.as_completed(tasks):
        try:
            path, elements = await coro
            results[path] = elements
        except Exception as e:
            logger.error(f"API error: {e}")

    return results
```

---

## Cost Modeling

```python
from dataclasses import dataclass


@dataclass
class CostEstimate:
    strategy: str
    deployment: str
    monthly_pages: int
    compute_cost: float
    api_cost: float
    total_cost: float
    pages_per_dollar: float


def estimate_costs(monthly_pages: int) -> list[CostEstimate]:
    """Estimate monthly costs for different configurations."""
    estimates = []

    # Local fast -- CPU instance
    cpu_hours = monthly_pages / 3600  # ~1 page/sec
    cpu_cost = cpu_hours * 0.17       # c5.xlarge on-demand
    estimates.append(CostEstimate(
        strategy="fast",
        deployment="local_cpu",
        monthly_pages=monthly_pages,
        compute_cost=round(cpu_cost, 2),
        api_cost=0,
        total_cost=round(cpu_cost, 2),
        pages_per_dollar=round(monthly_pages / max(cpu_cost, 0.01)),
    ))

    # Local hi_res -- GPU instance
    gpu_hours = monthly_pages / 360  # ~0.1 page/sec
    gpu_cost = gpu_hours * 0.526     # g4dn.xlarge on-demand
    estimates.append(CostEstimate(
        strategy="hi_res",
        deployment="local_gpu",
        monthly_pages=monthly_pages,
        compute_cost=round(gpu_cost, 2),
        api_cost=0,
        total_cost=round(gpu_cost, 2),
        pages_per_dollar=round(monthly_pages / max(gpu_cost, 0.01)),
    ))

    # Hosted API
    api_price_per_page = 0.001  # Approximate
    api_cost = monthly_pages * api_price_per_page
    estimates.append(CostEstimate(
        strategy="hi_res",
        deployment="hosted_api",
        monthly_pages=monthly_pages,
        compute_cost=0,
        api_cost=round(api_cost, 2),
        total_cost=round(api_cost, 2),
        pages_per_dollar=round(monthly_pages / max(api_cost, 0.01)),
    ))

    return estimates
```

---

## Decision Framework

```
What are your requirements?
  |
  +---> Data must stay on-premise
  |     |
  |     +---> Simple documents (text-heavy, no tables)
  |     |     --> Local fast strategy (cheapest, fastest)
  |     |
  |     +---> Complex documents (tables, multi-column, scanned)
  |     |     --> Local hi_res with GPU
  |     |
  |     +---> Mixed document types
  |           --> Local auto strategy (adaptive)
  |
  +---> Data can go to cloud
        |
        +---> Volume < 1K pages/month
        |     --> Hosted API free tier
        |
        +---> Volume 1K-100K pages/month
        |     --> Hosted API (cost-effective, no infra)
        |
        +---> Volume > 100K pages/month
              --> Cost comparison (local GPU vs API pricing)
```

---

## Common Pitfalls

1. **Running hi_res on CPU for production workloads.** Without GPU, hi_res is 10-50x slower. Use fast strategy on CPU or provision GPU instances.
2. **Not caching model downloads in CI/CD.** Layout models are 2-4GB. Cache them in Docker layers or artifact storage.
3. **Using the API without split_pdf_page.** Without page splitting, the API processes pages sequentially. Enabling it gives 3-5x throughput improvement.
4. **Comparing strategies without controlling for document type.** Fast and hi_res produce similar output on simple documents. The difference shows on complex layouts.
5. **Not setting concurrency limits for the API.** Too many concurrent requests hit rate limits. Start with 5-10 concurrent requests and scale up.
6. **Ignoring the auto strategy.** Auto inspects the document and chooses the best strategy per page. It is often the best default.

---

## References

- Unstructured partitioning strategies -- https://docs.unstructured.io/open-source/concepts/partitioning-strategies
- Unstructured API reference -- https://docs.unstructured.io/api-reference/
- YOLOX paper -- https://arxiv.org/abs/2107.08430
- Detectron2 -- https://github.com/facebookresearch/detectron2
- Unstructured pricing -- https://unstructured.io/pricing
