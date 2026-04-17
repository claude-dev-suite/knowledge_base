# OCR -- Tools Comparison

## Overview / TL;DR

OCR accuracy depends heavily on matching the right engine to the document type. Tesseract excels on clean printed text, PaddleOCR leads on mixed content and CJK, cloud services dominate on degraded and handwritten documents, and multimodal LLMs provide the best contextual understanding. This guide provides systematic accuracy benchmarks by document type, cost analysis, language support matrices, and a decision framework for choosing the optimal engine.

---

## Accuracy Benchmarks by Document Type

### Clean Printed Text (Reports, Books, Letters)

| Engine | Word Accuracy | Character Accuracy | Speed (page/s) | Cost/Page |
|--------|-------------|-------------------|----------------|-----------|
| Tesseract 5 | 95-98% | 97-99% | 1-2 | Free |
| EasyOCR | 93-97% | 96-98% | 0.5-1 | Free |
| PaddleOCR | 96-99% | 98-99% | 1-3 | Free |
| Textract | 97-99% | 99%+ | 0.5-1 | $0.0015 |
| Azure Doc AI | 97-99% | 99%+ | 0.5-1 | $0.001 |
| Google Doc AI | 97-99% | 99%+ | 0.5-1 | $0.001 |
| Claude Vision | 95-98% | 97-99% | 0.3-0.5 | $0.01-0.05 |

### Degraded Documents (Faded, Stained, Low Resolution)

| Engine | Word Accuracy | Notes |
|--------|-------------|-------|
| Tesseract 5 | 60-80% | Very sensitive to noise |
| EasyOCR | 70-85% | Better noise tolerance |
| PaddleOCR | 75-90% | Good with preprocessing |
| Textract | 85-95% | Built-in enhancement |
| Azure Doc AI | 85-95% | Strong preprocessing |
| Google Doc AI | 83-93% | Good degradation handling |
| Claude Vision | 88-97% | Best contextual recovery |

### Handwritten Text

| Engine | Word Accuracy | Notes |
|--------|-------------|-------|
| Tesseract 5 | 20-40% | Not designed for handwriting |
| EasyOCR | 40-60% | Limited handwriting support |
| PaddleOCR | 50-70% | Chinese handwriting is better |
| Textract | 70-85% | Trained on handwriting |
| Azure Doc AI | 70-85% | Good Latin handwriting |
| Google Doc AI | 65-80% | Decent multi-script |
| Claude Vision | 80-95% | Best overall, context helps |

### Dense Tables and Forms

| Engine | Cell Accuracy | Structure Detection | Notes |
|--------|-------------|-------------------|-------|
| Tesseract 5 | 70-85% | None | Text only, no structure |
| EasyOCR | 65-80% | None | May miss cells |
| PaddleOCR | 80-90% | Basic layout analysis | PP-StructureV2 |
| Textract | 90-97% | Excellent | Best table extraction |
| Azure Doc AI | 88-95% | Very Good | Strong forms |
| Google Doc AI | 85-93% | Good | Custom models available |
| Claude Vision | 85-95% | Context-based | No bounding boxes |

### Receipts and Invoices

| Engine | Key-Value Accuracy | Amount Accuracy | Notes |
|--------|-------------------|----------------|-------|
| Tesseract 5 | 60-75% | 70-85% | Struggles with layouts |
| PaddleOCR | 80-90% | 85-95% | Good structured extraction |
| Textract | 92-98% | 95-99% | AnalyzeExpense API |
| Azure Doc AI | 90-97% | 94-98% | Prebuilt invoice model |
| Google Doc AI | 88-95% | 92-97% | Custom processors |
| Claude Vision | 90-97% | 93-98% | Excellent understanding |

---

## Language Support Matrix

| Engine | Latin | CJK | Arabic | Devanagari | Cyrillic | Thai | Total Languages |
|--------|-------|-----|--------|-----------|----------|------|----------------|
| Tesseract 5 | Excellent | Good | Good | Good | Excellent | Good | 100+ |
| EasyOCR | Good | Good | Good | Good | Good | Good | 80+ |
| PaddleOCR | Good | Excellent | Fair | Fair | Good | Fair | 80+ |
| Textract | Excellent | Good | Good | Limited | Good | Limited | ~30 |
| Azure Doc AI | Excellent | Good | Good | Good | Excellent | Good | 60+ |
| Google Doc AI | Excellent | Excellent | Good | Good | Excellent | Good | 60+ |
| Claude Vision | Excellent | Excellent | Good | Good | Excellent | Good | Most |

---

## Speed Benchmarks

```python
import time
from pathlib import Path
from dataclasses import dataclass


@dataclass
class SpeedResult:
    engine: str
    image: str
    time_seconds: float
    chars_extracted: int
    chars_per_second: float


def benchmark_speed(
    image_paths: list[str],
    engines: dict,
    warmup: int = 1,
) -> list[SpeedResult]:
    """Benchmark OCR speed across engines."""
    results = []

    for engine_name, ocr_fn in engines.items():
        # Warmup
        for _ in range(warmup):
            try:
                ocr_fn(image_paths[0])
            except Exception:
                pass

        for image_path in image_paths:
            start = time.perf_counter()
            try:
                text = ocr_fn(image_path)
                elapsed = time.perf_counter() - start
                chars = len(text)
                results.append(SpeedResult(
                    engine=engine_name,
                    image=Path(image_path).name,
                    time_seconds=round(elapsed, 3),
                    chars_extracted=chars,
                    chars_per_second=round(chars / max(elapsed, 0.001)),
                ))
            except Exception as e:
                results.append(SpeedResult(
                    engine=engine_name,
                    image=Path(image_path).name,
                    time_seconds=-1,
                    chars_extracted=0,
                    chars_per_second=0,
                ))

    return results
```

**Typical speed results (single A4 page, 300 DPI)**:

| Engine | CPU Time | GPU Time | Relative |
|--------|----------|----------|----------|
| Tesseract 5 | 1.0-2.0s | N/A | 1x |
| PaddleOCR | 1.5-3.0s | 0.3-0.8s | 0.5-3x |
| EasyOCR | 3.0-8.0s | 0.5-1.5s | 0.3-2x |
| Textract | 1.5-3.0s | N/A | 0.5x + network |
| Azure Doc AI | 1.5-3.0s | N/A | 0.5x + network |
| Claude Vision | 2.0-5.0s | N/A | 0.3x + network |

---

## Cost Analysis

### Open-Source Engines (Infrastructure Cost Only)

| Engine | CPU Instance (c5.xlarge) | GPU Instance (g4dn.xlarge) | Cost per 10K Pages |
|--------|-------------------------|---------------------------|-------------------|
| Tesseract 5 | $0.17/hr, ~5K pg/hr | N/A | ~$0.34 |
| PaddleOCR | $0.17/hr, ~2K pg/hr | $0.526/hr, ~10K pg/hr | $0.05-0.85 |
| EasyOCR | $0.17/hr, ~500 pg/hr | $0.526/hr, ~5K pg/hr | $0.10-3.40 |

### Cloud Services

| Service | Per-Page Price | 10K Pages | 100K Pages | Free Tier |
|---------|---------------|-----------|------------|-----------|
| Textract (detect) | $0.0015 | $15 | $150 | 1K/month |
| Textract (analyze) | $0.015 | $150 | $1,500 | 1K/month |
| Azure Doc AI (read) | $0.001 | $10 | $100 | 500/month |
| Azure Doc AI (layout) | $0.01 | $100 | $1,000 | 500/month |
| Google Doc AI | $0.001-0.065 | $10-650 | $100-6,500 | 1K/month |
| Claude Vision (Haiku) | ~$0.005 | ~$50 | ~$500 | None |
| Claude Vision (Sonnet) | ~$0.02 | ~$200 | ~$2,000 | None |

---

## Accuracy Measurement

```python
from dataclasses import dataclass
from difflib import SequenceMatcher
import re


@dataclass
class AccuracyResult:
    engine: str
    word_accuracy: float
    character_accuracy: float
    word_count_extracted: int
    word_count_ground_truth: int


def measure_accuracy(
    extracted: str,
    ground_truth: str,
) -> dict:
    """Measure OCR accuracy against ground truth."""
    # Normalize
    ext_norm = " ".join(extracted.split()).lower()
    gt_norm = " ".join(ground_truth.split()).lower()

    # Character-level accuracy
    char_matcher = SequenceMatcher(None, ext_norm, gt_norm)
    char_accuracy = char_matcher.ratio()

    # Word-level accuracy
    ext_words = ext_norm.split()
    gt_words = gt_norm.split()
    word_matcher = SequenceMatcher(None, ext_words, gt_words)
    word_accuracy = word_matcher.ratio()

    # Edit distance
    char_ops = char_matcher.get_opcodes()
    insertions = sum(j2 - j1 for tag, i1, i2, j1, j2 in char_ops if tag == "insert")
    deletions = sum(i2 - i1 for tag, i1, i2, j1, j2 in char_ops if tag == "delete")
    replacements = sum(i2 - i1 for tag, i1, i2, j1, j2 in char_ops if tag == "replace")

    return {
        "character_accuracy": round(char_accuracy, 4),
        "word_accuracy": round(word_accuracy, 4),
        "insertions": insertions,
        "deletions": deletions,
        "replacements": replacements,
        "total_errors": insertions + deletions + replacements,
        "cer": round((insertions + deletions + replacements) / max(len(gt_norm), 1), 4),
        "wer": round(
            1 - word_accuracy,
            4,
        ),
    }


def benchmark_accuracy(
    image_paths: list[str],
    ground_truths: list[str],
    engines: dict,
) -> list[AccuracyResult]:
    """Benchmark accuracy across engines and images."""
    results = []

    for engine_name, ocr_fn in engines.items():
        total_word_acc = 0
        total_char_acc = 0
        count = 0

        for image_path, gt in zip(image_paths, ground_truths):
            try:
                extracted = ocr_fn(image_path)
                metrics = measure_accuracy(extracted, gt)
                total_word_acc += metrics["word_accuracy"]
                total_char_acc += metrics["character_accuracy"]
                count += 1
            except Exception:
                continue

        if count > 0:
            results.append(AccuracyResult(
                engine=engine_name,
                word_accuracy=round(total_word_acc / count, 4),
                character_accuracy=round(total_char_acc / count, 4),
                word_count_extracted=0,
                word_count_ground_truth=0,
            ))

    return results
```

---

## Integration Comparison

### LangChain Integration

```python
# Tesseract via Unstructured
from langchain_community.document_loaders import UnstructuredImageLoader

loader = UnstructuredImageLoader(
    "scan.png",
    mode="elements",
    strategy="ocr_only",
)
docs = loader.load()

# Textract
from langchain_community.document_loaders import AmazonTextractPDFLoader

loader = AmazonTextractPDFLoader("scan.pdf")
docs = loader.load()
```

### LlamaIndex Integration

```python
from llama_index.readers.file import ImageReader

reader = ImageReader(text_type="plain_text")
docs = reader.load_data("scan.png")
```

---

## Decision Framework

```
What type of document?
  |
  +---> Clean printed text (reports, books)
  |     |
  |     +---> High volume (100K+ pages)
  |     |     --> PaddleOCR with GPU (best accuracy/cost ratio)
  |     |
  |     +---> Low volume, simple setup
  |           --> Tesseract (free, easy, good enough)
  |
  +---> Degraded documents (faded, noisy, low-res)
  |     |
  |     +---> Budget allows cloud APIs
  |     |     --> Textract or Azure Document Intelligence
  |     |
  |     +---> Must be local
  |           --> PaddleOCR with heavy preprocessing
  |
  +---> Handwritten text
  |     |
  |     +---> Critical accuracy required
  |     |     --> Claude Vision (best handwriting recognition)
  |     |
  |     +---> Moderate accuracy OK
  |           --> Textract or Azure Document Intelligence
  |
  +---> Dense tables and forms
  |     --> Textract AnalyzeDocument (best table structure detection)
  |
  +---> CJK-heavy documents
  |     --> PaddleOCR (specifically trained for CJK)
  |
  +---> Receipts and invoices
  |     --> Textract AnalyzeExpense or Azure prebuilt invoice
  |
  +---> Mixed/unknown document types
        --> Claude Vision for critical docs, PaddleOCR for bulk
```

---

## Common Pitfalls

1. **Choosing by benchmark scores alone.** Published benchmarks use clean test sets. Your documents may have different characteristics.
2. **Not accounting for preprocessing in speed comparisons.** Cloud services include preprocessing. Local engines need explicit preprocessing, adding to total time.
3. **Ignoring the cost of GPU instances.** PaddleOCR and EasyOCR are fast on GPU but g4dn.xlarge costs $0.526/hour. At low volumes, cloud APIs are cheaper.
4. **Using Tesseract for handwriting.** Tesseract's LSTM model is trained on printed text. Handwriting accuracy is 20-40%.
5. **Comparing apples and oranges.** Textract's "AnalyzeDocument" includes table structure and forms. Comparing it to Tesseract's raw text extraction is unfair.
6. **Not testing on your actual documents.** Create a representative test set from your real document corpus and measure accuracy before choosing.

---

## References

- Tesseract documentation -- https://tesseract-ocr.github.io/tessdoc/
- EasyOCR benchmarks -- https://github.com/JaidedAI/EasyOCR
- PaddleOCR benchmarks -- https://paddlepaddle.github.io/PaddleOCR/
- Amazon Textract pricing -- https://aws.amazon.com/textract/pricing/
- Azure Document Intelligence pricing -- https://azure.microsoft.com/en-us/pricing/details/ai-document-intelligence/
- Google Document AI pricing -- https://cloud.google.com/document-ai/pricing
