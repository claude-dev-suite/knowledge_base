# OCR -- Comprehensive Guide

## Overview / TL;DR

Optical Character Recognition (OCR) converts images of text into machine-readable characters. In document processing pipelines, OCR is the critical bridge between scanned documents, photographs, and screenshots and the text-based processing that downstream NLP, RAG, and LLM systems require. This guide covers every major OCR engine (Tesseract, EasyOCR, PaddleOCR, Amazon Textract, Azure Document Intelligence, Google Document AI, and Claude vision), explains preprocessing techniques that dramatically improve accuracy, and provides production-ready Python code for building reliable OCR pipelines.

---

## Why OCR Accuracy Varies

OCR accuracy depends on the interaction of four factors:

1. **Image quality** -- Resolution, contrast, skew, noise, and blur directly impact character recognition. A 300 DPI clean scan yields 95%+ accuracy; a 72 DPI photo of a crumpled document may yield 60%.
2. **Document type** -- Printed text in standard fonts is easy. Handwriting, mixed fonts, dense tables, and text overlaid on images are progressively harder.
3. **Language and script** -- Latin scripts with standard character sets are well-supported. CJK, Arabic, Devanagari, and other scripts vary by engine.
4. **Engine capabilities** -- Rule-based engines (Tesseract) struggle where ML-based engines (PaddleOCR, cloud APIs) excel, and vice versa.

---

## Engine Landscape

### Tesseract

The most widely deployed open-source OCR engine. Version 5.x uses an LSTM-based recognition model.

```python
import pytesseract
from PIL import Image


def ocr_tesseract(
    image_path: str,
    lang: str = "eng",
    psm: int = 3,
    oem: int = 3,
) -> str:
    """Extract text from an image using Tesseract."""
    image = Image.open(image_path)

    custom_config = f"--oem {oem} --psm {psm}"
    # OEM: 0=Legacy, 1=LSTM, 2=Legacy+LSTM, 3=Default
    # PSM: 3=Auto, 4=Single column, 6=Single block, 7=Single line, 11=Sparse text

    text = pytesseract.image_to_string(
        image,
        lang=lang,
        config=custom_config,
    )
    return text


def ocr_tesseract_detailed(image_path: str) -> list[dict]:
    """Get word-level bounding boxes and confidence from Tesseract."""
    image = Image.open(image_path)

    data = pytesseract.image_to_data(
        image,
        output_type=pytesseract.Output.DICT,
    )

    words = []
    for i in range(len(data["text"])):
        if data["text"][i].strip():
            words.append({
                "text": data["text"][i],
                "confidence": data["conf"][i],
                "x": data["left"][i],
                "y": data["top"][i],
                "width": data["width"][i],
                "height": data["height"][i],
                "block": data["block_num"][i],
                "paragraph": data["par_num"][i],
                "line": data["line_num"][i],
                "word": data["word_num"][i],
            })

    return words


def ocr_tesseract_hocr(image_path: str) -> str:
    """Get hOCR output (HTML with bounding boxes) from Tesseract."""
    image = Image.open(image_path)
    return pytesseract.image_to_pdf_or_hocr(image, extension="hocr").decode("utf-8")
```

**Strengths**: Free, widely available, 100+ languages, well-documented, good for clean printed text. **Weaknesses**: Poor on degraded/noisy images, no built-in preprocessing, no handwriting support, requires separate installation.

### EasyOCR

PyTorch-based OCR with built-in text detection (CRAFT) and recognition for 80+ languages.

```python
import easyocr


def ocr_easyocr(
    image_path: str,
    languages: list[str] = None,
    gpu: bool = True,
) -> list[dict]:
    """Extract text with bounding boxes using EasyOCR."""
    if languages is None:
        languages = ["en"]

    reader = easyocr.Reader(languages, gpu=gpu)
    results = reader.readtext(image_path)

    words = []
    for bbox, text, confidence in results:
        # bbox is [[x1,y1], [x2,y2], [x3,y3], [x4,y4]]
        x_coords = [p[0] for p in bbox]
        y_coords = [p[1] for p in bbox]

        words.append({
            "text": text,
            "confidence": round(confidence, 3),
            "bbox": bbox,
            "x_min": min(x_coords),
            "y_min": min(y_coords),
            "x_max": max(x_coords),
            "y_max": max(y_coords),
        })

    return words


def ocr_easyocr_text(
    image_path: str,
    languages: list[str] = None,
) -> str:
    """Get plain text output from EasyOCR."""
    results = ocr_easyocr(image_path, languages)
    return "\n".join(r["text"] for r in results)
```

**Strengths**: Easy setup (pip install), built-in text detection, GPU acceleration, good multilingual support. **Weaknesses**: Slower than Tesseract on CPU, higher memory usage (PyTorch), less accurate on dense documents.

### PaddleOCR

PaddlePaddle-based OCR from Baidu with state-of-the-art accuracy, especially for CJK languages.

```python
from paddleocr import PaddleOCR


def ocr_paddleocr(
    image_path: str,
    lang: str = "en",
    use_gpu: bool = True,
) -> list[dict]:
    """Extract text using PaddleOCR."""
    ocr = PaddleOCR(
        use_angle_cls=True,   # Detect text rotation
        lang=lang,
        use_gpu=use_gpu,
        show_log=False,
    )

    results = ocr.ocr(image_path, cls=True)

    words = []
    if results and results[0]:
        for line in results[0]:
            bbox = line[0]
            text = line[1][0]
            confidence = line[1][1]

            words.append({
                "text": text,
                "confidence": round(confidence, 3),
                "bbox": bbox,
            })

    return words


def ocr_paddleocr_text(image_path: str, lang: str = "en") -> str:
    """Get plain text from PaddleOCR."""
    results = ocr_paddleocr(image_path, lang)
    return "\n".join(r["text"] for r in results)
```

**Strengths**: Highest accuracy among open-source engines (especially CJK), built-in text detection, angle classification, lightweight models, fast inference. **Weaknesses**: PaddlePaddle dependency (less common than PyTorch), documentation primarily in Chinese.

### Amazon Textract

Cloud OCR service with document analysis, table extraction, and form detection.

```python
import boto3


def ocr_textract(image_path: str) -> dict:
    """Extract text using Amazon Textract."""
    client = boto3.client("textract")

    with open(image_path, "rb") as f:
        image_bytes = f.read()

    response = client.detect_text(
        Document={"Bytes": image_bytes},
    )

    lines = []
    words = []
    for block in response["Blocks"]:
        if block["BlockType"] == "LINE":
            lines.append({
                "text": block["Text"],
                "confidence": block["Confidence"],
                "bbox": block["Geometry"]["BoundingBox"],
            })
        elif block["BlockType"] == "WORD":
            words.append({
                "text": block["Text"],
                "confidence": block["Confidence"],
                "bbox": block["Geometry"]["BoundingBox"],
            })

    return {
        "full_text": "\n".join(l["text"] for l in lines),
        "lines": lines,
        "words": words,
    }


def ocr_textract_document(image_path: str) -> dict:
    """Full document analysis with tables and forms."""
    client = boto3.client("textract")

    with open(image_path, "rb") as f:
        image_bytes = f.read()

    response = client.analyze_document(
        Document={"Bytes": image_bytes},
        FeatureTypes=["TABLES", "FORMS"],
    )

    return response
```

### Azure Document Intelligence

```python
from azure.ai.documentintelligence import DocumentIntelligenceClient
from azure.core.credentials import AzureKeyCredential


def ocr_azure(
    image_path: str,
    endpoint: str,
    api_key: str,
) -> dict:
    """Extract text using Azure Document Intelligence."""
    client = DocumentIntelligenceClient(
        endpoint=endpoint,
        credential=AzureKeyCredential(api_key),
    )

    with open(image_path, "rb") as f:
        poller = client.begin_analyze_document(
            "prebuilt-read",
            body=f,
        )

    result = poller.result()

    pages = []
    for page in result.pages:
        lines = []
        for line in page.lines:
            lines.append({
                "text": line.content,
                "polygon": line.polygon,
            })
        pages.append({
            "page_number": page.page_number,
            "lines": lines,
        })

    return {
        "full_text": result.content,
        "pages": pages,
        "language": result.languages[0] if result.languages else None,
    }
```

### Claude Vision (Multimodal LLM)

```python
import anthropic
import base64


def ocr_claude_vision(
    image_path: str,
    instruction: str = "Extract all text from this image exactly as it appears.",
) -> str:
    """Use Claude's vision capability for OCR."""
    client = anthropic.Anthropic()

    with open(image_path, "rb") as f:
        image_data = base64.standard_b64encode(f.read()).decode("utf-8")

    # Determine media type
    ext = image_path.lower().rsplit(".", 1)[-1]
    media_types = {
        "png": "image/png",
        "jpg": "image/jpeg",
        "jpeg": "image/jpeg",
        "gif": "image/gif",
        "webp": "image/webp",
    }
    media_type = media_types.get(ext, "image/png")

    response = client.messages.create(
        model="claude-sonnet-4-20250514",
        max_tokens=4096,
        messages=[{
            "role": "user",
            "content": [
                {
                    "type": "image",
                    "source": {
                        "type": "base64",
                        "media_type": media_type,
                        "data": image_data,
                    },
                },
                {
                    "type": "text",
                    "text": instruction,
                },
            ],
        }],
    )

    return response.content[0].text
```

**Strengths**: Handles any document type including handwriting, forms, diagrams with text, degraded images. Understands context. **Weaknesses**: Highest cost per page, rate limits, non-deterministic output, cannot provide bounding boxes.

---

## Quick Comparison

| Feature | Tesseract | EasyOCR | PaddleOCR | Textract | Azure Doc AI | Claude Vision |
|---------|-----------|---------|-----------|----------|-------------|---------------|
| Accuracy (clean print) | 95%+ | 93%+ | 96%+ | 97%+ | 97%+ | 95%+ |
| Accuracy (degraded) | 70-85% | 80-90% | 85-93% | 90-96% | 90-96% | 92-98% |
| Handwriting | None | Basic | Basic | Good | Good | Excellent |
| Speed (single page) | 0.5-2s | 1-5s | 0.5-3s | 1-3s | 1-3s | 2-5s |
| Languages | 100+ | 80+ | 80+ | 10+ | 60+ | Most |
| GPU benefit | None | 3-5x | 3-5x | N/A | N/A | N/A |
| Bounding boxes | Yes | Yes | Yes | Yes | Yes | No |
| Table detection | No | No | No | Yes | Yes | Contextual |
| Local/Cloud | Local | Local | Local | Cloud | Cloud | Cloud |
| Cost (per page) | Free | Free | Free | $0.0015 | $0.001 | $0.01-0.05 |

---

## Image Preprocessing

Preprocessing is the single biggest lever for improving OCR accuracy. A well-preprocessed image can improve accuracy by 10-30%.

```python
import cv2
import numpy as np
from PIL import Image


def preprocess_for_ocr(
    image_path: str,
    output_path: str | None = None,
    deskew: bool = True,
    denoise: bool = True,
    binarize: bool = True,
    upscale: bool = True,
    target_dpi: int = 300,
) -> np.ndarray:
    """Full preprocessing pipeline for OCR."""
    image = cv2.imread(image_path)

    if image is None:
        raise ValueError(f"Could not load image: {image_path}")

    # Convert to grayscale
    if len(image.shape) == 3:
        gray = cv2.cvtColor(image, cv2.COLOR_BGR2GRAY)
    else:
        gray = image

    # Upscale if resolution is low
    if upscale:
        height, width = gray.shape[:2]
        # Assume 72 DPI if image is small
        if max(height, width) < 1000:
            scale = target_dpi / 72
            gray = cv2.resize(
                gray,
                None,
                fx=scale,
                fy=scale,
                interpolation=cv2.INTER_CUBIC,
            )

    # Deskew
    if deskew:
        gray = _deskew(gray)

    # Denoise
    if denoise:
        gray = cv2.fastNlMeansDenoising(gray, h=10)

    # Binarize (adaptive thresholding)
    if binarize:
        gray = cv2.adaptiveThreshold(
            gray,
            255,
            cv2.ADAPTIVE_THRESH_GAUSSIAN_C,
            cv2.THRESH_BINARY,
            blockSize=11,
            C=2,
        )

    if output_path:
        cv2.imwrite(output_path, gray)

    return gray


def _deskew(image: np.ndarray) -> np.ndarray:
    """Correct image skew using projection profile."""
    coords = np.column_stack(np.where(image < 128))
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
    rotated = cv2.warpAffine(
        image,
        matrix,
        (w, h),
        flags=cv2.INTER_CUBIC,
        borderMode=cv2.BORDER_REPLICATE,
    )

    return rotated


def remove_borders(image: np.ndarray, border_size: int = 10) -> np.ndarray:
    """Remove dark borders from scanned images."""
    h, w = image.shape[:2]
    # Find content area by thresholding
    _, binary = cv2.threshold(image, 200, 255, cv2.THRESH_BINARY)

    # Find contours
    contours, _ = cv2.findContours(
        binary, cv2.RETR_EXTERNAL, cv2.CHAIN_APPROX_SIMPLE
    )

    if contours:
        # Get bounding box of all content
        all_points = np.concatenate(contours)
        x, y, cw, ch = cv2.boundingRect(all_points)
        # Add small padding
        x = max(0, x - border_size)
        y = max(0, y - border_size)
        cw = min(w - x, cw + 2 * border_size)
        ch = min(h - y, ch + 2 * border_size)
        return image[y:y + ch, x:x + cw]

    return image


def enhance_contrast(image: np.ndarray) -> np.ndarray:
    """Enhance contrast using CLAHE."""
    clahe = cv2.createCLAHE(clipLimit=2.0, tileGridSize=(8, 8))
    return clahe.apply(image)
```

---

## Post-OCR Correction

```python
import re


def correct_common_ocr_errors(text: str) -> str:
    """Fix common OCR misrecognitions."""
    corrections = {
        # Common character confusions
        r'\bl\b': 'I',       # lowercase L -> uppercase I (context-dependent)
        r'(?<=[a-z])0(?=[a-z])': 'o',  # zero in middle of word -> letter o
        r'(?<=[a-z])1(?=[a-z])': 'l',  # one in middle of word -> letter l
        r'rn(?=[a-z])': 'm',  # rn -> m (common)
    }

    # Apply regex corrections
    corrected = text
    for pattern, replacement in corrections.items():
        corrected = re.sub(pattern, replacement, corrected)

    # Fix common word-level errors
    word_corrections = {
        "tbe": "the",
        "arid": "and",
        "witb": "with",
        "tbat": "that",
        "frorn": "from",
    }

    words = corrected.split()
    corrected_words = [word_corrections.get(w.lower(), w) for w in words]
    corrected = " ".join(corrected_words)

    return corrected


def merge_hyphenated_words(text: str) -> str:
    """Rejoin words split by end-of-line hyphenation."""
    return re.sub(r'(\w)-\n(\w)', r'\1\2', text)


def remove_ocr_artifacts(text: str) -> str:
    """Remove common OCR artifacts."""
    # Remove isolated single characters (likely noise)
    text = re.sub(r'(?<=\s)[^\w\s](?=\s)', '', text)

    # Remove excessive spaces
    text = re.sub(r' {3,}', '  ', text)

    # Remove lines that are just punctuation/noise
    lines = text.split('\n')
    cleaned_lines = []
    for line in lines:
        stripped = line.strip()
        if stripped and len(re.sub(r'[^\w]', '', stripped)) > 0:
            cleaned_lines.append(line)

    return '\n'.join(cleaned_lines)
```

---

## Common Pitfalls

1. **Skipping preprocessing.** Raw scanned images often have skew, noise, and low contrast. Even 5 degrees of skew can drop accuracy by 20%.
2. **Using default Tesseract settings for all documents.** The PSM (page segmentation mode) must match the content layout. Single column, sparse text, and single line each need different PSM values.
3. **Not checking DPI.** OCR engines expect 300 DPI images. Low-resolution images must be upscaled before processing.
4. **OCR-ing native-text PDFs.** If a PDF has extractable text, direct extraction (PyMuPDF) is faster and more accurate than OCR.
5. **Ignoring confidence scores.** Low-confidence words should be flagged for review or re-processed with a different engine.
6. **Using cloud OCR for simple printed text.** Tesseract or PaddleOCR handles clean printed text at 95%+ accuracy for free. Cloud APIs are overkill.
7. **Not correcting common OCR errors.** Post-correction with dictionaries and pattern matching can recover 2-5% accuracy.
8. **Processing color images directly.** Always convert to grayscale and binarize before OCR for consistent results.

---

## Production Checklist

- [ ] Images are preprocessed (grayscale, deskew, denoise, binarize, upscale to 300 DPI)
- [ ] OCR engine is matched to document type (Tesseract for clean print, PaddleOCR for mixed, cloud for degraded)
- [ ] PSM/layout mode is configured for the content layout
- [ ] Confidence scores are captured and low-confidence results are flagged
- [ ] Post-OCR correction handles common character confusions and artifacts
- [ ] Hyphenated line breaks are rejoined
- [ ] Native-text PDFs bypass OCR and use direct extraction
- [ ] Multilingual documents use the correct language model
- [ ] Batch processing uses parallel workers for throughput
- [ ] Output includes bounding boxes for source tracing and validation

---

## References

- Tesseract documentation -- https://tesseract-ocr.github.io/tessdoc/
- EasyOCR repository -- https://github.com/JaidedAI/EasyOCR
- PaddleOCR documentation -- https://paddlepaddle.github.io/PaddleOCR/
- Amazon Textract -- https://docs.aws.amazon.com/textract/latest/dg/
- Azure Document Intelligence -- https://learn.microsoft.com/en-us/azure/ai-services/document-intelligence/
- OpenCV image processing -- https://docs.opencv.org/4.x/d7/dbd/group__imgproc.html
