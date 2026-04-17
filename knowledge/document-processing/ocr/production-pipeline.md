# OCR -- Production Pipeline

## Overview / TL;DR

A production OCR pipeline must handle diverse image qualities, route documents to the optimal engine, preprocess images for maximum accuracy, validate output, and produce clean text ready for downstream processing. This guide provides a complete pipeline implementation with image classification, adaptive preprocessing, multi-engine routing, confidence-based validation, post-OCR correction, and batch processing with error recovery.

---

## Architecture

```
Input Images (scan / photo / PDF page render)
    |
    v
[1] Image Classifier
    |-- clean print --> lightweight OCR (Tesseract/PaddleOCR)
    |-- degraded --> heavy preprocessing + cloud OCR
    |-- handwritten --> Claude Vision / cloud API
    |-- mixed --> zone segmentation + per-zone routing
    |
    v
[2] Preprocessing
    |-- grayscale conversion
    |-- DPI normalization (target 300)
    |-- deskew
    |-- denoise
    |-- contrast enhancement
    |-- binarization
    |
    v
[3] OCR Engine (with fallback)
    |-- primary engine
    |-- confidence check
    |-- fallback engine if confidence < threshold
    |
    v
[4] Post-Processing
    |-- common error correction
    |-- hyphenation repair
    |-- artifact removal
    |-- layout reconstruction
    |
    v
[5] Validation & Output
    |-- confidence scoring
    |-- quality metrics
    |-- structured output (text + metadata)
```

---

## Image Classifier

```python
import cv2
import numpy as np
from dataclasses import dataclass
from enum import Enum


class ImageQuality(Enum):
    CLEAN_PRINT = "clean_print"
    DEGRADED = "degraded"
    HANDWRITTEN = "handwritten"
    MIXED = "mixed"
    PHOTO = "photo"
    LOW_RES = "low_resolution"


@dataclass
class ImageClassification:
    quality: ImageQuality
    dpi_estimate: int
    skew_angle: float
    noise_level: float      # 0-1, higher = more noise
    contrast_ratio: float   # higher = better contrast
    is_color: bool
    has_text_regions: bool
    recommended_engine: str
    recommended_preprocessing: list[str]


def classify_image(image_path: str) -> ImageClassification:
    """Classify image quality to determine optimal processing strategy."""
    image = cv2.imread(image_path)
    if image is None:
        raise ValueError(f"Cannot load image: {image_path}")

    h, w = image.shape[:2]
    is_color = len(image.shape) == 3 and image.shape[2] == 3

    # Convert to grayscale for analysis
    gray = cv2.cvtColor(image, cv2.COLOR_BGR2GRAY) if is_color else image

    # Estimate DPI (heuristic based on image dimensions for A4)
    # A4 at 300 DPI = ~2480x3508 pixels
    dpi_estimate = int(max(w, h) / 11.7 * 1)  # Rough estimate

    # Measure noise level using Laplacian variance
    laplacian_var = cv2.Laplacian(gray, cv2.CV_64F).var()
    noise_level = min(1.0, max(0.0, 1.0 - laplacian_var / 1000))

    # Measure contrast
    contrast_ratio = float(gray.max() - gray.min()) / 255

    # Estimate skew
    skew_angle = _estimate_skew(gray)

    # Determine quality category
    preprocessing = []
    if abs(skew_angle) > 1.0:
        preprocessing.append("deskew")
    if noise_level > 0.5:
        preprocessing.append("denoise")
    if contrast_ratio < 0.5:
        preprocessing.append("enhance_contrast")
    if dpi_estimate < 200:
        preprocessing.append("upscale")

    preprocessing.append("binarize")

    # Classify
    if noise_level > 0.7 or contrast_ratio < 0.3:
        quality = ImageQuality.DEGRADED
        engine = "cloud"  # Textract or Azure
    elif dpi_estimate < 150:
        quality = ImageQuality.LOW_RES
        engine = "paddleocr"  # Better at low res
    elif noise_level < 0.3 and contrast_ratio > 0.6:
        quality = ImageQuality.CLEAN_PRINT
        engine = "tesseract"  # Fast and accurate
    else:
        quality = ImageQuality.MIXED
        engine = "paddleocr"

    return ImageClassification(
        quality=quality,
        dpi_estimate=dpi_estimate,
        skew_angle=round(skew_angle, 2),
        noise_level=round(noise_level, 3),
        contrast_ratio=round(contrast_ratio, 3),
        is_color=is_color,
        has_text_regions=True,
        recommended_engine=engine,
        recommended_preprocessing=preprocessing,
    )


def _estimate_skew(gray: np.ndarray) -> float:
    """Estimate image skew angle."""
    edges = cv2.Canny(gray, 50, 150, apertureSize=3)
    lines = cv2.HoughLinesP(
        edges, 1, np.pi / 180,
        threshold=100,
        minLineLength=100,
        maxLineGap=10,
    )

    if lines is None or len(lines) == 0:
        return 0.0

    angles = []
    for line in lines:
        x1, y1, x2, y2 = line[0]
        angle = np.degrees(np.arctan2(y2 - y1, x2 - x1))
        if abs(angle) < 45:
            angles.append(angle)

    if not angles:
        return 0.0

    return float(np.median(angles))
```

---

## Preprocessing Pipeline

```python
import cv2
import numpy as np


class OCRPreprocessor:
    """Configurable image preprocessing for OCR."""

    def __init__(self, target_dpi: int = 300):
        self.target_dpi = target_dpi

    def preprocess(
        self,
        image: np.ndarray,
        steps: list[str],
    ) -> np.ndarray:
        """Apply preprocessing steps in order."""
        processed = image.copy()

        for step in steps:
            method = getattr(self, f"_step_{step}", None)
            if method:
                processed = method(processed)
            else:
                raise ValueError(f"Unknown preprocessing step: {step}")

        return processed

    def _step_grayscale(self, image: np.ndarray) -> np.ndarray:
        if len(image.shape) == 3:
            return cv2.cvtColor(image, cv2.COLOR_BGR2GRAY)
        return image

    def _step_upscale(self, image: np.ndarray) -> np.ndarray:
        h, w = image.shape[:2]
        if max(h, w) < 2000:
            scale = 2.0
            return cv2.resize(
                image, None, fx=scale, fy=scale,
                interpolation=cv2.INTER_CUBIC,
            )
        return image

    def _step_deskew(self, image: np.ndarray) -> np.ndarray:
        gray = image if len(image.shape) == 2 else cv2.cvtColor(image, cv2.COLOR_BGR2GRAY)
        coords = np.column_stack(np.where(gray < 128))
        if len(coords) < 100:
            return image

        angle = cv2.minAreaRect(coords)[-1]
        if angle < -45:
            angle = 90 + angle
        elif angle > 45:
            angle = angle - 90

        if abs(angle) < 0.5:
            return image

        h, w = image.shape[:2]
        center = (w // 2, h // 2)
        matrix = cv2.getRotationMatrix2D(center, angle, 1.0)
        return cv2.warpAffine(
            image, matrix, (w, h),
            flags=cv2.INTER_CUBIC,
            borderMode=cv2.BORDER_REPLICATE,
        )

    def _step_denoise(self, image: np.ndarray) -> np.ndarray:
        if len(image.shape) == 3:
            return cv2.fastNlMeansDenoisingColored(image, h=10)
        return cv2.fastNlMeansDenoising(image, h=10)

    def _step_enhance_contrast(self, image: np.ndarray) -> np.ndarray:
        gray = image if len(image.shape) == 2 else cv2.cvtColor(image, cv2.COLOR_BGR2GRAY)
        clahe = cv2.createCLAHE(clipLimit=2.0, tileGridSize=(8, 8))
        return clahe.apply(gray)

    def _step_binarize(self, image: np.ndarray) -> np.ndarray:
        gray = image if len(image.shape) == 2 else cv2.cvtColor(image, cv2.COLOR_BGR2GRAY)
        return cv2.adaptiveThreshold(
            gray, 255,
            cv2.ADAPTIVE_THRESH_GAUSSIAN_C,
            cv2.THRESH_BINARY,
            blockSize=11,
            C=2,
        )

    def _step_remove_borders(self, image: np.ndarray) -> np.ndarray:
        gray = image if len(image.shape) == 2 else cv2.cvtColor(image, cv2.COLOR_BGR2GRAY)
        _, binary = cv2.threshold(gray, 200, 255, cv2.THRESH_BINARY)
        contours, _ = cv2.findContours(binary, cv2.RETR_EXTERNAL, cv2.CHAIN_APPROX_SIMPLE)

        if contours:
            all_pts = np.concatenate(contours)
            x, y, cw, ch = cv2.boundingRect(all_pts)
            pad = 10
            x = max(0, x - pad)
            y = max(0, y - pad)
            return image[y:y + ch + 2 * pad, x:x + cw + 2 * pad]

        return image
```

---

## Multi-Engine OCR with Fallback

```python
import logging
from dataclasses import dataclass, field
from typing import Callable

logger = logging.getLogger(__name__)


@dataclass
class OCRResult:
    text: str
    engine: str
    confidence: float
    word_count: int
    processing_seconds: float
    warnings: list[str] = field(default_factory=list)
    word_details: list[dict] = field(default_factory=list)
    error: str | None = None


class MultiEngineOCR:
    """OCR with automatic engine selection and fallback."""

    def __init__(
        self,
        confidence_threshold: float = 0.7,
        enable_cloud: bool = False,
        cloud_api_key: str | None = None,
    ):
        self.confidence_threshold = confidence_threshold
        self.enable_cloud = enable_cloud
        self.cloud_api_key = cloud_api_key
        self.preprocessor = OCRPreprocessor()
        self._engines = self._init_engines()

    def _init_engines(self) -> dict[str, Callable]:
        """Initialize available engines."""
        engines = {}

        try:
            import pytesseract
            engines["tesseract"] = self._ocr_tesseract
        except ImportError:
            pass

        try:
            from paddleocr import PaddleOCR
            self._paddle = PaddleOCR(use_angle_cls=True, lang="en", show_log=False)
            engines["paddleocr"] = self._ocr_paddleocr
        except ImportError:
            pass

        try:
            import easyocr
            self._easyocr_reader = easyocr.Reader(["en"], gpu=True)
            engines["easyocr"] = self._ocr_easyocr
        except ImportError:
            pass

        if self.enable_cloud:
            engines["cloud"] = self._ocr_cloud

        return engines

    def process(
        self,
        image_path: str,
        preferred_engine: str | None = None,
    ) -> OCRResult:
        """Process an image with automatic engine selection and fallback."""
        import time

        start = time.perf_counter()

        # Classify image
        classification = classify_image(image_path)

        # Preprocess
        image = cv2.imread(image_path)
        steps = ["grayscale"] + classification.recommended_preprocessing
        preprocessed = self.preprocessor.preprocess(image, steps)

        # Select engine
        engine_name = preferred_engine or classification.recommended_engine
        if engine_name not in self._engines:
            # Fall back to first available
            engine_name = next(iter(self._engines), None)
            if engine_name is None:
                return OCRResult(
                    text="",
                    engine="none",
                    confidence=0.0,
                    word_count=0,
                    processing_seconds=time.perf_counter() - start,
                    error="No OCR engines available",
                )

        # Primary OCR
        result = self._engines[engine_name](preprocessed)

        # Check confidence and fallback if needed
        if result.confidence < self.confidence_threshold:
            logger.info(
                f"Low confidence ({result.confidence:.2f}) from {engine_name}, "
                f"trying fallback"
            )
            for fallback_name, fallback_fn in self._engines.items():
                if fallback_name != engine_name:
                    fallback_result = fallback_fn(preprocessed)
                    if fallback_result.confidence > result.confidence:
                        result = fallback_result
                        result.warnings.append(
                            f"Fallback from {engine_name} to {fallback_name}"
                        )
                    if result.confidence >= self.confidence_threshold:
                        break

        result.processing_seconds = time.perf_counter() - start
        return result

    def _ocr_tesseract(self, image: np.ndarray) -> OCRResult:
        """Tesseract OCR."""
        import pytesseract

        data = pytesseract.image_to_data(
            image, output_type=pytesseract.Output.DICT,
        )

        words = []
        confidences = []
        for i in range(len(data["text"])):
            text = data["text"][i].strip()
            conf = int(data["conf"][i])
            if text and conf > 0:
                words.append(text)
                confidences.append(conf / 100.0)

        avg_confidence = sum(confidences) / max(len(confidences), 1)
        full_text = " ".join(words)

        return OCRResult(
            text=full_text,
            engine="tesseract",
            confidence=round(avg_confidence, 3),
            word_count=len(words),
            processing_seconds=0.0,
        )

    def _ocr_paddleocr(self, image: np.ndarray) -> OCRResult:
        """PaddleOCR."""
        results = self._paddle.ocr(image, cls=True)

        words = []
        confidences = []
        if results and results[0]:
            for line in results[0]:
                text = line[1][0]
                conf = line[1][1]
                words.append(text)
                confidences.append(conf)

        avg_confidence = sum(confidences) / max(len(confidences), 1)
        full_text = "\n".join(words)

        return OCRResult(
            text=full_text,
            engine="paddleocr",
            confidence=round(avg_confidence, 3),
            word_count=len(full_text.split()),
            processing_seconds=0.0,
        )

    def _ocr_easyocr(self, image: np.ndarray) -> OCRResult:
        """EasyOCR."""
        results = self._easyocr_reader.readtext(image)

        words = []
        confidences = []
        for bbox, text, conf in results:
            words.append(text)
            confidences.append(conf)

        avg_confidence = sum(confidences) / max(len(confidences), 1)
        full_text = "\n".join(words)

        return OCRResult(
            text=full_text,
            engine="easyocr",
            confidence=round(avg_confidence, 3),
            word_count=len(full_text.split()),
            processing_seconds=0.0,
        )

    def _ocr_cloud(self, image: np.ndarray) -> OCRResult:
        """Cloud OCR (Textract)."""
        import boto3

        _, buffer = cv2.imencode(".png", image)
        image_bytes = buffer.tobytes()

        client = boto3.client("textract")
        response = client.detect_text(
            Document={"Bytes": image_bytes},
        )

        lines = []
        confidences = []
        for block in response["Blocks"]:
            if block["BlockType"] == "LINE":
                lines.append(block["Text"])
                confidences.append(block["Confidence"] / 100.0)

        avg_confidence = sum(confidences) / max(len(confidences), 1)
        full_text = "\n".join(lines)

        return OCRResult(
            text=full_text,
            engine="textract",
            confidence=round(avg_confidence, 3),
            word_count=len(full_text.split()),
            processing_seconds=0.0,
        )
```

---

## Post-OCR Correction

```python
import re


class OCRPostProcessor:
    """Post-processing pipeline for OCR output."""

    def __init__(self, custom_dictionary: set[str] | None = None):
        self.dictionary = custom_dictionary or set()

    def process(self, text: str) -> str:
        """Apply all post-processing steps."""
        text = self.fix_common_errors(text)
        text = self.merge_hyphenated(text)
        text = self.remove_artifacts(text)
        text = self.normalize_whitespace(text)
        return text

    def fix_common_errors(self, text: str) -> str:
        """Fix common OCR character confusions."""
        # Context-sensitive replacements
        replacements = [
            (r'(?<=[a-z])0(?=[a-z])', 'o'),   # zero in word -> o
            (r'(?<=[a-z])1(?=[a-z])', 'l'),   # one in word -> l
            (r'(?<=[A-Z])0(?=[A-Z])', 'O'),   # zero in caps -> O
            (r'\bI-Iave\b', 'Have'),
            (r'\brn(?=[aeiouy])', 'm'),         # rn before vowel -> m
        ]

        for pattern, replacement in replacements:
            text = re.sub(pattern, replacement, text)

        return text

    def merge_hyphenated(self, text: str) -> str:
        """Rejoin hyphenated words at line breaks."""
        return re.sub(r'(\w)-\s*\n\s*(\w)', r'\1\2', text)

    def remove_artifacts(self, text: str) -> str:
        """Remove common OCR artifacts."""
        # Remove isolated punctuation
        text = re.sub(r'(?<=\s)[^\w\s]{1,2}(?=\s)', '', text)

        # Remove repeated characters (likely noise)
        text = re.sub(r'(.)\1{5,}', r'\1\1', text)

        # Remove lines with very few alphanumeric chars
        lines = text.split('\n')
        cleaned = []
        for line in lines:
            alpha_count = sum(1 for c in line if c.isalnum())
            if alpha_count > 2 or not line.strip():
                cleaned.append(line)

        return '\n'.join(cleaned)

    def normalize_whitespace(self, text: str) -> str:
        """Normalize whitespace without destroying structure."""
        # Multiple spaces to single (within lines)
        lines = text.split('\n')
        normalized = []
        for line in lines:
            normalized.append(re.sub(r' {2,}', ' ', line))

        # Remove excessive blank lines
        text = '\n'.join(normalized)
        text = re.sub(r'\n{4,}', '\n\n\n', text)

        return text.strip()
```

---

## Batch Processing

```python
import asyncio
import logging
import time
from concurrent.futures import ProcessPoolExecutor
from dataclasses import dataclass, field
from pathlib import Path

logger = logging.getLogger(__name__)


@dataclass
class BatchResult:
    total_images: int
    successful: int
    failed: int
    total_seconds: float
    results: list[OCRResult] = field(default_factory=list)
    errors: list[dict] = field(default_factory=list)


def process_batch_sync(
    image_paths: list[str],
    max_workers: int = 4,
    engine: str | None = None,
) -> BatchResult:
    """Process multiple images in parallel."""
    start = time.perf_counter()
    ocr = MultiEngineOCR()
    results = []
    errors = []

    with ProcessPoolExecutor(max_workers=max_workers) as executor:
        futures = {
            executor.submit(_process_single, path, engine): path
            for path in image_paths
        }

        from concurrent.futures import as_completed
        for future in as_completed(futures):
            path = futures[future]
            try:
                result = future.result(timeout=120)
                results.append(result)
            except Exception as e:
                errors.append({"file": path, "error": str(e)})

    return BatchResult(
        total_images=len(image_paths),
        successful=len(results),
        failed=len(errors),
        total_seconds=time.perf_counter() - start,
        results=results,
        errors=errors,
    )


def _process_single(image_path: str, engine: str | None) -> OCRResult:
    """Process a single image (worker function)."""
    ocr = MultiEngineOCR()
    return ocr.process(image_path, preferred_engine=engine)
```

---

## PDF-to-OCR Integration

```python
import fitz
import tempfile
from pathlib import Path


def ocr_pdf_pages(
    pdf_path: str,
    dpi: int = 300,
    engine: str | None = None,
) -> list[OCRResult]:
    """Render PDF pages to images and OCR them."""
    doc = fitz.open(pdf_path)
    ocr = MultiEngineOCR()
    results = []

    for page_num, page in enumerate(doc):
        # Render page to image
        pix = page.get_pixmap(dpi=dpi)

        with tempfile.NamedTemporaryFile(suffix=".png", delete=False) as tmp:
            pix.save(tmp.name)
            tmp_path = tmp.name

        try:
            result = ocr.process(tmp_path, preferred_engine=engine)
            result.warnings.append(f"Page {page_num + 1}")
            results.append(result)
        finally:
            Path(tmp_path).unlink(missing_ok=True)

    doc.close()
    return results
```

---

## Common Pitfalls

1. **Skipping image classification.** Clean documents processed with cloud APIs waste money. Degraded documents processed with Tesseract waste accuracy.
2. **Not preprocessing before OCR.** Deskewing, denoising, and contrast enhancement consistently improve accuracy by 5-20%.
3. **Using a single engine for all documents.** No engine is best at everything. Multi-engine with fallback catches failures.
4. **Not checking confidence scores.** Low-confidence results should trigger fallback, not pass silently to downstream.
5. **Processing full-color images.** Color adds noise and processing time. Convert to grayscale and binarize first.
6. **Not parallelizing batch processing.** OCR is CPU-intensive. Use process pools for throughput.
7. **Ignoring post-OCR correction.** Simple pattern-based correction recovers 2-5% of errors for free.
8. **Rendering PDFs at 72 DPI for OCR.** Always render at 300 DPI minimum. 72 DPI is unreadable for most OCR engines.

---

## Production Checklist

- [ ] Image classifier routes documents to optimal engine and preprocessing steps
- [ ] Preprocessing applies deskew, denoise, contrast enhancement, and binarization as needed
- [ ] Multiple OCR engines are available for fallback on low confidence
- [ ] Confidence scores are captured and checked against threshold
- [ ] Post-OCR correction handles common errors, hyphenation, and artifacts
- [ ] PDF pages are rendered at 300+ DPI before OCR
- [ ] Batch processing uses parallel workers with timeout
- [ ] Error handling isolates failures per-image
- [ ] Output includes text, confidence, engine used, and per-word data
- [ ] Metrics track accuracy, throughput, and engine distribution

---

## References

- Tesseract documentation -- https://tesseract-ocr.github.io/tessdoc/
- PaddleOCR -- https://paddlepaddle.github.io/PaddleOCR/
- EasyOCR -- https://github.com/JaidedAI/EasyOCR
- OpenCV preprocessing -- https://docs.opencv.org/4.x/
- Amazon Textract -- https://docs.aws.amazon.com/textract/
