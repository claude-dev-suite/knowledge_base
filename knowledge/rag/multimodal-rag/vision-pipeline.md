# Multimodal RAG -- Vision Pipeline: Claude/GPT-4o Vision + VoyageAI Multimodal Embeddings

## TL;DR

The vision pipeline handles images, diagrams, charts, and screenshots in RAG systems. It has two stages: (1) understanding -- using vision LLMs (Claude, GPT-4o) to interpret visual content and extract structured information, and (2) embedding -- using multimodal embedding models (VoyageAI multimodal-3, Cohere embed-v4) to place images in the same vector space as text for cross-modal retrieval. This article covers production implementations of both stages, including diagram-specific extraction, chart data recovery, screenshot OCR, and optimized embedding strategies for different image types.

---

## Vision LLM Integration

### Claude Vision for Document Understanding

```python
import base64
from pathlib import Path
from anthropic import Anthropic
from dataclasses import dataclass


@dataclass
class VisionExtractionResult:
    description: str
    extracted_text: str
    structured_data: dict | None
    confidence: float
    image_type: str  # diagram, chart, screenshot, photo, formula


class ClaudeVisionPipeline:
    """Production vision pipeline using Claude for image understanding.

    Claude excels at:
    - Architecture diagrams (extracting components and connections)
    - Charts/graphs (recovering underlying data)
    - Screenshots (OCR + UI element identification)
    - Technical diagrams (flowcharts, sequence diagrams, ERDs)
    """

    def __init__(self, client: Anthropic | None = None, model: str = "claude-sonnet-4-20250514"):
        self.client = client or Anthropic()
        self.model = model

    def classify_image(self, image_path: str) -> str:
        """Classify the type of image for specialized extraction."""
        image_b64 = self._encode_image(image_path)
        media_type = self._get_media_type(image_path)

        response = self.client.messages.create(
            model=self.model,
            max_tokens=100,
            messages=[{
                "role": "user",
                "content": [
                    {"type": "image", "source": {
                        "type": "base64",
                        "media_type": media_type,
                        "data": image_b64,
                    }},
                    {"type": "text", "text": (
                        "Classify this image into exactly one category:\n"
                        "- DIAGRAM: architecture, flowchart, sequence, ERD, network\n"
                        "- CHART: bar, line, pie, scatter, histogram\n"
                        "- SCREENSHOT: UI, application window, terminal output\n"
                        "- PHOTO: photograph of real-world object or scene\n"
                        "- FORMULA: mathematical equation or formula\n"
                        "- TABLE: visual table (not HTML/markdown)\n"
                        "Respond with exactly one word."
                    )},
                ],
            }],
        )
        return response.content[0].text.strip().upper()

    def extract_diagram(self, image_path: str) -> VisionExtractionResult:
        """Extract structured information from architecture/flow diagrams."""
        image_b64 = self._encode_image(image_path)
        media_type = self._get_media_type(image_path)

        response = self.client.messages.create(
            model=self.model,
            max_tokens=2048,
            messages=[{
                "role": "user",
                "content": [
                    {"type": "image", "source": {
                        "type": "base64",
                        "media_type": media_type,
                        "data": image_b64,
                    }},
                    {"type": "text", "text": (
                        "Analyze this diagram and extract:\n\n"
                        "1. **Components**: List every named component/box/node\n"
                        "2. **Connections**: List every arrow/line with source, target, and label\n"
                        "3. **Flow**: Describe the overall flow or architecture\n"
                        "4. **Labels**: Extract all visible text labels\n"
                        "5. **Description**: Write a comprehensive paragraph describing "
                        "what this diagram shows, suitable for a knowledge base.\n\n"
                        "Format the components and connections as structured lists."
                    )},
                ],
            }],
        )

        text = response.content[0].text
        components = self._extract_section(text, "Components")
        connections = self._extract_section(text, "Connections")

        return VisionExtractionResult(
            description=text,
            extracted_text=self._extract_section(text, "Labels"),
            structured_data={
                "components": components,
                "connections": connections,
            },
            confidence=0.85,
            image_type="diagram",
        )

    def extract_chart(self, image_path: str) -> VisionExtractionResult:
        """Extract data and insights from charts/graphs."""
        image_b64 = self._encode_image(image_path)
        media_type = self._get_media_type(image_path)

        response = self.client.messages.create(
            model=self.model,
            max_tokens=2048,
            messages=[{
                "role": "user",
                "content": [
                    {"type": "image", "source": {
                        "type": "base64",
                        "media_type": media_type,
                        "data": image_b64,
                    }},
                    {"type": "text", "text": (
                        "Analyze this chart/graph and extract:\n\n"
                        "1. **Chart Type**: What kind of chart is this?\n"
                        "2. **Title**: The chart title if visible\n"
                        "3. **Axes**: X-axis label and Y-axis label\n"
                        "4. **Data Series**: Name each data series\n"
                        "5. **Data Points**: Extract approximate values as a table.\n"
                        "   Format: | X Value | Series 1 | Series 2 | ...\n"
                        "6. **Key Insights**: What trends or patterns are visible?\n"
                        "7. **Description**: Write a comprehensive paragraph describing "
                        "the chart and its findings.\n\n"
                        "Be as precise as possible with numerical values."
                    )},
                ],
            }],
        )

        text = response.content[0].text

        return VisionExtractionResult(
            description=text,
            extracted_text=self._extract_section(text, "Data Points"),
            structured_data={
                "chart_type": self._extract_section(text, "Chart Type"),
                "title": self._extract_section(text, "Title"),
                "insights": self._extract_section(text, "Key Insights"),
            },
            confidence=0.75,  # data point extraction is approximate
            image_type="chart",
        )

    def extract_screenshot(self, image_path: str) -> VisionExtractionResult:
        """Extract text and UI structure from screenshots."""
        image_b64 = self._encode_image(image_path)
        media_type = self._get_media_type(image_path)

        response = self.client.messages.create(
            model=self.model,
            max_tokens=2048,
            messages=[{
                "role": "user",
                "content": [
                    {"type": "image", "source": {
                        "type": "base64",
                        "media_type": media_type,
                        "data": image_b64,
                    }},
                    {"type": "text", "text": (
                        "Extract all information from this screenshot:\n\n"
                        "1. **All visible text**: Transcribe every piece of text verbatim\n"
                        "2. **UI elements**: Describe buttons, menus, panels, tabs\n"
                        "3. **State**: What is the application showing? Any errors, "
                        "notifications, or status indicators?\n"
                        "4. **Context**: What application is this? What task is being performed?\n"
                        "5. **Description**: Write a detailed paragraph suitable for "
                        "a knowledge base.\n\n"
                        "Preserve the exact text as shown, including any code, "
                        "error messages, or configuration values."
                    )},
                ],
            }],
        )

        text = response.content[0].text

        return VisionExtractionResult(
            description=text,
            extracted_text=self._extract_section(text, "All visible text"),
            structured_data={
                "ui_elements": self._extract_section(text, "UI elements"),
                "state": self._extract_section(text, "State"),
            },
            confidence=0.90,
            image_type="screenshot",
        )

    def auto_extract(self, image_path: str) -> VisionExtractionResult:
        """Classify and extract using the appropriate specialized method."""
        image_type = self.classify_image(image_path)

        extractors = {
            "DIAGRAM": self.extract_diagram,
            "CHART": self.extract_chart,
            "SCREENSHOT": self.extract_screenshot,
        }

        extractor = extractors.get(image_type)
        if extractor:
            return extractor(image_path)

        # Fallback: generic description
        return self._generic_extract(image_path, image_type)

    def _generic_extract(
        self, image_path: str, image_type: str
    ) -> VisionExtractionResult:
        """Generic extraction for unspecialized image types."""
        image_b64 = self._encode_image(image_path)
        media_type = self._get_media_type(image_path)

        response = self.client.messages.create(
            model=self.model,
            max_tokens=1024,
            messages=[{
                "role": "user",
                "content": [
                    {"type": "image", "source": {
                        "type": "base64",
                        "media_type": media_type,
                        "data": image_b64,
                    }},
                    {"type": "text", "text": (
                        "Describe this image in comprehensive detail for a "
                        "knowledge base. Include all text, numbers, labels, "
                        "and important visual elements."
                    )},
                ],
            }],
        )

        return VisionExtractionResult(
            description=response.content[0].text,
            extracted_text="",
            structured_data=None,
            confidence=0.80,
            image_type=image_type.lower(),
        )

    @staticmethod
    def _encode_image(path: str) -> str:
        return base64.standard_b64encode(Path(path).read_bytes()).decode()

    @staticmethod
    def _get_media_type(path: str) -> str:
        ext = Path(path).suffix.lower()
        return {
            ".jpg": "image/jpeg", ".jpeg": "image/jpeg",
            ".png": "image/png", ".gif": "image/gif",
            ".webp": "image/webp",
        }.get(ext, "image/png")

    @staticmethod
    def _extract_section(text: str, section_name: str) -> str:
        """Extract content under a markdown section heading."""
        lines = text.split("\n")
        capture = False
        section_lines = []

        for line in lines:
            if section_name.lower() in line.lower() and (
                line.strip().startswith("#") or line.strip().startswith("**")
            ):
                capture = True
                continue
            elif capture and line.strip().startswith(("**", "#")):
                break
            elif capture:
                section_lines.append(line)

        return "\n".join(section_lines).strip()
```

### GPT-4o Vision as Alternative

```python
from openai import OpenAI


class GPT4oVisionPipeline:
    """Alternative vision pipeline using GPT-4o.

    GPT-4o handles vision similarly to Claude but with different strengths:
    - Better at reading handwritten text
    - Better at complex mathematical formulas
    - Lower cost per image token

    Claude strengths:
    - Better at architecture diagram interpretation
    - Better at long-form image description
    - More consistent structured extraction
    """

    def __init__(self, client: OpenAI | None = None):
        self.client = client or OpenAI()
        self.model = "gpt-4o"

    def extract(self, image_path: str, extraction_prompt: str) -> str:
        """Generic vision extraction with GPT-4o."""
        image_b64 = base64.standard_b64encode(
            Path(image_path).read_bytes()
        ).decode()
        media_type = self._get_media_type(image_path)

        response = self.client.chat.completions.create(
            model=self.model,
            max_tokens=2048,
            messages=[{
                "role": "user",
                "content": [
                    {
                        "type": "image_url",
                        "image_url": {
                            "url": f"data:{media_type};base64,{image_b64}",
                            "detail": "high",  # high resolution analysis
                        },
                    },
                    {"type": "text", "text": extraction_prompt},
                ],
            }],
        )
        return response.choices[0].message.content

    def batch_extract(
        self, image_paths: list[str], prompt: str, max_concurrent: int = 5
    ) -> list[str]:
        """Extract from multiple images with rate limiting.

        GPT-4o has a tokens-per-minute limit for vision.
        Each image consumes approximately:
        - Low detail: 85 tokens
        - High detail: 85 + 170 * (tiles), where tiles depend on resolution
        - 1024x1024 image = ~765 tokens
        """
        import concurrent.futures
        import time

        results = [None] * len(image_paths)

        with concurrent.futures.ThreadPoolExecutor(
            max_workers=max_concurrent
        ) as executor:
            futures = {}
            for i, path in enumerate(image_paths):
                future = executor.submit(self.extract, path, prompt)
                futures[future] = i

            for future in concurrent.futures.as_completed(futures):
                idx = futures[future]
                try:
                    results[idx] = future.result()
                except Exception as e:
                    results[idx] = f"Extraction failed: {e}"

        return results

    @staticmethod
    def _get_media_type(path: str) -> str:
        ext = Path(path).suffix.lower()
        return {
            ".jpg": "image/jpeg", ".jpeg": "image/jpeg",
            ".png": "image/png", ".gif": "image/gif",
            ".webp": "image/webp",
        }.get(ext, "image/png")
```

---

## VoyageAI Multimodal Embedding

### Production Embedding Pipeline

```python
import httpx
import time
from pathlib import Path
from dataclasses import dataclass, field


@dataclass
class EmbeddingResult:
    embedding: list[float]
    input_type: str
    token_usage: int
    model: str


@dataclass
class BatchEmbeddingResult:
    embeddings: list[list[float]]
    total_tokens: int
    failed_indices: list[int] = field(default_factory=list)


class VoyageMultimodalEmbedder:
    """Production VoyageAI multimodal embedding pipeline.

    VoyageAI multimodal-3 supports:
    - Pure text inputs
    - Pure image inputs (base64)
    - Interleaved text + image inputs (best quality)

    Embedding dimension: 1024
    Max input tokens: 32000 per request
    Max images per request: 1 (multimodal endpoint)
    Max texts per batch: 128 (text endpoint)

    Pricing (as of 2025):
    - Text: $0.06 per 1M tokens
    - Multimodal: $0.12 per 1M tokens
    """

    def __init__(
        self,
        api_key: str,
        max_retries: int = 3,
        retry_delay: float = 1.0,
    ):
        self.api_key = api_key
        self.base_url = "https://api.voyageai.com/v1"
        self.model = "voyage-multimodal-3"
        self.max_retries = max_retries
        self.retry_delay = retry_delay

    def embed_text_batch(
        self, texts: list[str], input_type: str = "document"
    ) -> BatchEmbeddingResult:
        """Embed a batch of text strings.

        input_type should be "document" for indexing, "query" for retrieval.
        """
        all_embeddings = []
        total_tokens = 0

        # Process in batches of 128
        for i in range(0, len(texts), 128):
            batch = texts[i : i + 128]

            response = self._request_with_retry(
                f"{self.base_url}/embeddings",
                json={
                    "model": self.model,
                    "input": batch,
                    "input_type": input_type,
                },
            )

            data = response.json()
            batch_embeddings = [item["embedding"] for item in data["data"]]
            all_embeddings.extend(batch_embeddings)
            total_tokens += data.get("usage", {}).get("total_tokens", 0)

        return BatchEmbeddingResult(
            embeddings=all_embeddings,
            total_tokens=total_tokens,
        )

    def embed_image(
        self,
        image_path: str,
        context_text: str | None = None,
        input_type: str = "document",
    ) -> EmbeddingResult:
        """Embed a single image, optionally with surrounding text context.

        Including context_text significantly improves embedding quality
        because the model understands what the image represents in context.
        """
        image_data = Path(image_path).read_bytes()
        media_type = self._get_media_type(image_path)
        b64 = base64.standard_b64encode(image_data).decode()

        # Build interleaved input
        input_content = []
        if context_text:
            input_content.append({"type": "text", "text": context_text})
        input_content.append({
            "type": "image_base64",
            "image_base64": b64,
            "media_type": media_type,
        })

        response = self._request_with_retry(
            f"{self.base_url}/multimodalembeddings",
            json={
                "model": self.model,
                "inputs": [input_content],
                "input_type": input_type,
            },
        )

        data = response.json()
        return EmbeddingResult(
            embedding=data["data"][0]["embedding"],
            input_type=input_type,
            token_usage=data.get("usage", {}).get("total_tokens", 0),
            model=self.model,
        )

    def embed_images_batch(
        self, image_configs: list[dict], input_type: str = "document"
    ) -> BatchEmbeddingResult:
        """Embed multiple images sequentially (multimodal endpoint is 1-at-a-time).

        image_configs: list of dicts with "path" and optional "context" keys.

        For high throughput, run this with ThreadPoolExecutor.
        """
        embeddings = []
        total_tokens = 0
        failed = []

        for i, config in enumerate(image_configs):
            try:
                result = self.embed_image(
                    config["path"],
                    context_text=config.get("context"),
                    input_type=input_type,
                )
                embeddings.append(result.embedding)
                total_tokens += result.token_usage
            except Exception as e:
                print(f"Failed to embed image {i} ({config['path']}): {e}")
                embeddings.append([0.0] * 1024)  # zero vector placeholder
                failed.append(i)

        return BatchEmbeddingResult(
            embeddings=embeddings,
            total_tokens=total_tokens,
            failed_indices=failed,
        )

    def _request_with_retry(self, url: str, **kwargs) -> httpx.Response:
        """HTTP request with exponential backoff retry."""
        kwargs.setdefault("headers", {})
        kwargs["headers"]["Authorization"] = f"Bearer {self.api_key}"
        kwargs["timeout"] = 120

        for attempt in range(self.max_retries):
            try:
                response = httpx.post(url, **kwargs)
                if response.status_code == 429:
                    # Rate limited -- back off
                    retry_after = float(
                        response.headers.get("retry-after", self.retry_delay)
                    )
                    time.sleep(retry_after)
                    continue
                response.raise_for_status()
                return response
            except httpx.HTTPStatusError:
                if attempt < self.max_retries - 1:
                    time.sleep(self.retry_delay * (2 ** attempt))
                    continue
                raise
            except httpx.TimeoutException:
                if attempt < self.max_retries - 1:
                    time.sleep(self.retry_delay)
                    continue
                raise

        raise RuntimeError(f"Failed after {self.max_retries} retries")

    @staticmethod
    def _get_media_type(path: str) -> str:
        ext = Path(path).suffix.lower()
        return {
            ".jpg": "image/jpeg", ".jpeg": "image/jpeg",
            ".png": "image/png", ".gif": "image/gif",
            ".webp": "image/webp",
        }.get(ext, "image/png")
```

---

## Optimized Image Preprocessing

### Image Quality and Size Optimization

```python
from PIL import Image
from io import BytesIO


class ImagePreprocessor:
    """Preprocess images before embedding or vision LLM analysis.

    Optimizations:
    1. Resize large images to reduce token consumption
    2. Convert to optimal format (JPEG for photos, PNG for diagrams)
    3. Enhance contrast for low-quality scans
    4. Crop whitespace borders
    5. Split panoramic images into sections
    """

    def __init__(
        self,
        max_dimension: int = 2048,
        quality: int = 85,
        min_dimension: int = 100,
    ):
        self.max_dimension = max_dimension
        self.quality = quality
        self.min_dimension = min_dimension

    def preprocess(self, image_path: str, output_path: str | None = None) -> str:
        """Full preprocessing pipeline for an image."""
        img = Image.open(image_path)

        # Skip very small images
        if img.width < self.min_dimension or img.height < self.min_dimension:
            raise ValueError(
                f"Image too small ({img.width}x{img.height}), "
                f"minimum is {self.min_dimension}x{self.min_dimension}"
            )

        # Crop whitespace
        img = self._crop_whitespace(img)

        # Resize if needed
        img = self._resize(img)

        # Convert RGBA to RGB (for JPEG compatibility)
        if img.mode == "RGBA":
            background = Image.new("RGB", img.size, (255, 255, 255))
            background.paste(img, mask=img.split()[3])
            img = background

        # Save
        if output_path is None:
            output_path = image_path
        img.save(output_path, quality=self.quality, optimize=True)
        return output_path

    def _resize(self, img: Image.Image) -> Image.Image:
        """Resize to max dimension while preserving aspect ratio."""
        if max(img.width, img.height) <= self.max_dimension:
            return img

        ratio = self.max_dimension / max(img.width, img.height)
        new_size = (int(img.width * ratio), int(img.height * ratio))
        return img.resize(new_size, Image.LANCZOS)

    def _crop_whitespace(
        self, img: Image.Image, threshold: int = 240
    ) -> Image.Image:
        """Remove whitespace borders from an image."""
        import numpy as np

        arr = np.array(img.convert("L"))
        mask = arr < threshold
        if not mask.any():
            return img

        rows = mask.any(axis=1)
        cols = mask.any(axis=0)

        rmin, rmax = rows.argmax(), len(rows) - rows[::-1].argmax()
        cmin, cmax = cols.argmax(), len(cols) - cols[::-1].argmax()

        # Add small padding
        padding = 10
        rmin = max(0, rmin - padding)
        rmax = min(img.height, rmax + padding)
        cmin = max(0, cmin - padding)
        cmax = min(img.width, cmax + padding)

        return img.crop((cmin, rmin, cmax, rmax))

    def estimate_token_cost(self, image_path: str) -> dict:
        """Estimate token consumption for vision LLMs.

        Claude token estimation:
        - Images are resized to fit within 1568x1568
        - Token cost = (width * height) / 750

        GPT-4o token estimation (high detail):
        - Image split into 512x512 tiles
        - 170 tokens per tile + 85 base tokens
        """
        img = Image.open(image_path)
        w, h = img.width, img.height

        # Claude estimation
        claude_scale = min(1, 1568 / max(w, h))
        claude_w = int(w * claude_scale)
        claude_h = int(h * claude_scale)
        claude_tokens = (claude_w * claude_h) // 750

        # GPT-4o estimation (high detail)
        gpt_scale = min(1, 2048 / max(w, h))
        gpt_w = int(w * gpt_scale)
        gpt_h = int(h * gpt_scale)
        # Short side scaled to 768
        short_scale = min(1, 768 / min(gpt_w, gpt_h))
        final_w = int(gpt_w * short_scale)
        final_h = int(gpt_h * short_scale)
        tiles = ((final_w + 511) // 512) * ((final_h + 511) // 512)
        gpt_tokens = 85 + 170 * tiles

        return {
            "original_size": (w, h),
            "claude_tokens": claude_tokens,
            "gpt4o_tokens": gpt_tokens,
            "claude_cost_usd": claude_tokens * 3e-6,  # $3/M input tokens
            "gpt4o_cost_usd": gpt_tokens * 2.5e-6,  # $2.5/M input tokens
        }
```

---

## Combining Vision Extraction with Embeddings

```python
class VisionEmbeddingPipeline:
    """Unified pipeline: extract structured info + embed for retrieval.

    For each image:
    1. Classify the image type
    2. Extract structured information using the appropriate method
    3. Embed the image (optionally with context) for vector retrieval
    4. Store both the extraction and the embedding
    """

    def __init__(
        self,
        vision: ClaudeVisionPipeline,
        embedder: VoyageMultimodalEmbedder,
        preprocessor: ImagePreprocessor,
    ):
        self.vision = vision
        self.embedder = embedder
        self.preprocessor = preprocessor

    def process_image(
        self,
        image_path: str,
        surrounding_text: str = "",
        document_source: str = "",
        page_number: int = 0,
    ) -> dict:
        """Process a single image through the full pipeline."""
        # Preprocess
        try:
            processed_path = self.preprocessor.preprocess(image_path)
        except ValueError:
            return {"skipped": True, "reason": "Image too small"}

        # Extract with vision LLM
        extraction = self.vision.auto_extract(processed_path)

        # Embed with context
        context = surrounding_text or extraction.description[:200]
        embedding_result = self.embedder.embed_image(
            processed_path, context_text=context
        )

        return {
            "skipped": False,
            "image_path": image_path,
            "image_type": extraction.image_type,
            "description": extraction.description,
            "extracted_text": extraction.extracted_text,
            "structured_data": extraction.structured_data,
            "embedding": embedding_result.embedding,
            "embedding_tokens": embedding_result.token_usage,
            "confidence": extraction.confidence,
            "metadata": {
                "source": document_source,
                "page": page_number,
                "surrounding_text": surrounding_text[:500],
            },
        }

    def process_document_images(
        self, images: list[dict], max_concurrent: int = 3
    ) -> list[dict]:
        """Process all images from a document.

        images: list of dicts with keys:
        - path: image file path
        - context: surrounding text (optional)
        - source: document name
        - page: page number

        Rate limiting: VoyageAI multimodal endpoint has per-minute limits.
        Process sequentially or with limited concurrency.
        """
        results = []
        for img_config in images:
            result = self.process_image(
                image_path=img_config["path"],
                surrounding_text=img_config.get("context", ""),
                document_source=img_config.get("source", ""),
                page_number=img_config.get("page", 0),
            )
            results.append(result)

        # Summary statistics
        processed = [r for r in results if not r.get("skipped")]
        skipped = [r for r in results if r.get("skipped")]

        print(f"Processed: {len(processed)}, Skipped: {len(skipped)}")
        if processed:
            total_tokens = sum(r["embedding_tokens"] for r in processed)
            print(f"Total embedding tokens: {total_tokens}")
            type_counts = {}
            for r in processed:
                t = r["image_type"]
                type_counts[t] = type_counts.get(t, 0) + 1
            print(f"Image types: {type_counts}")

        return results
```

---

## Cost and Quality Comparison

| Model | Cost per Image | Extraction Quality | Best For |
|-------|---------------|-------------------|----------|
| Claude Sonnet | ~$0.003-0.012 | Excellent diagrams | Architecture, flowcharts |
| Claude Haiku | ~$0.001-0.003 | Good general | Bulk processing, screenshots |
| GPT-4o | ~$0.002-0.008 | Excellent charts | Data extraction, formulas |
| GPT-4o-mini | ~$0.0005-0.002 | Good general | Cost-sensitive bulk |
| VoyageAI embed | ~$0.0001 | N/A (embedding only) | Retrieval, similarity |

### Recommended Strategy by Use Case

```python
def recommend_vision_strategy(
    image_count: int,
    primary_image_type: str,
    budget_usd: float,
    latency_requirement_ms: int,
) -> dict:
    """Recommend the optimal vision pipeline configuration."""

    if budget_usd / max(image_count, 1) < 0.002:
        return {
            "extraction_model": "claude-haiku (or skip extraction)",
            "embedding_model": "voyage-multimodal-3",
            "strategy": "Embed images directly, extract on-demand at query time",
            "reasoning": "Budget too low for per-image extraction",
        }

    if primary_image_type in ("diagram", "architecture"):
        return {
            "extraction_model": "claude-sonnet",
            "embedding_model": "voyage-multimodal-3 with context",
            "strategy": "Extract structured data + embed with extraction as context",
            "reasoning": "Claude excels at diagram interpretation",
        }

    if primary_image_type in ("chart", "graph", "formula"):
        return {
            "extraction_model": "gpt-4o",
            "embedding_model": "voyage-multimodal-3",
            "strategy": "Extract data points with GPT-4o, embed with data as context",
            "reasoning": "GPT-4o better at numerical data extraction",
        }

    return {
        "extraction_model": "claude-sonnet",
        "embedding_model": "voyage-multimodal-3 with context",
        "strategy": "Standard: classify, extract with appropriate model, embed with context",
        "reasoning": "General-purpose pipeline for mixed image types",
    }
```

---

## Common Pitfalls

1. **Not including context with image embeddings.** An image of a bar chart embedded alone is far less useful than the same image embedded alongside "Figure 3: Q4 revenue by region." Always pass surrounding text.

2. **Using the same extraction prompt for all image types.** A diagram extraction prompt wastes tokens on screenshots and misses key information. Classify first, then extract.

3. **Ignoring image resolution limits.** Vision LLMs resize images internally. Sending a 10000x10000 image does not improve quality but increases cost. Preprocess to 2048px max dimension.

4. **Processing decorative images.** Company logos, horizontal rules, and background patterns consume extraction budget without adding retrieval value. Filter by size and content.

5. **Not validating chart data extraction.** Vision LLMs approximate numerical values from charts. If exact values matter, validate against source data or flag with low confidence.

6. **Sequential processing when batch is possible.** Text embeddings can be batched (128 at a time). Image embeddings currently cannot (VoyageAI processes 1 at a time), but extraction can be parallelized across multiple LLM calls.

---

## References

- Anthropic Vision docs: https://docs.anthropic.com/en/docs/build-with-claude/vision
- OpenAI Vision guide: https://platform.openai.com/docs/guides/vision
- VoyageAI Multimodal: https://docs.voyageai.com/docs/multimodal-embeddings
- Image token estimation: https://docs.anthropic.com/en/docs/build-with-claude/vision#image-costs
