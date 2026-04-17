# Multimodal RAG -- Vision, Tables, Audio, and Video Retrieval

## TL;DR

Multimodal RAG extends traditional text-only retrieval to handle images, tables, audio, and video. Instead of converting everything to text (lossy), multimodal RAG uses vision-language models (Claude, GPT-4o) to understand visual content natively and multimodal embedding models (VoyageAI, Cohere) to embed different modalities into a shared vector space. The three main pipelines are: (1) vision pipeline for images and diagrams, (2) table pipeline for structured data in documents, and (3) audio/video pipeline for temporal media. This overview covers architecture decisions, embedding strategies, and production implementations for each modality.

---

## Why Text-Only RAG Fails for Real Documents

### The Modality Gap

Real-world documents contain far more than plain text:

| Content Type | Frequency in Enterprise Docs | Text-Only RAG Quality |
|-------------|-----------------------------|-----------------------|
| Paragraphs of text | ~50% of content | Excellent |
| Tables | ~15% of content | Poor (structure lost) |
| Charts/graphs | ~10% of content | Fails completely |
| Diagrams/architecture | ~8% of content | Fails completely |
| Images with text | ~7% of content | Partial (OCR only) |
| Code snippets | ~5% of content | Good (treated as text) |
| Screenshots | ~3% of content | Fails completely |
| Formulas/equations | ~2% of content | Poor |

When text-only RAG encounters a table, it either skips it entirely or serializes it into a flat string that loses row-column relationships. When it encounters a diagram, the information is completely inaccessible. This means 30-40% of document content is invisible to standard RAG.

### What Multimodal RAG Adds

```
Document (PDF/HTML/DOCX)
    |
    v
[Multimodal Parser]
    |
    +---> Text chunks  ---> Text embeddings  ---> Vector DB
    +---> Table chunks  ---> Table embeddings ---> Vector DB
    +---> Image chunks  ---> Vision embeddings --> Vector DB
    +---> Audio chunks  ---> Audio embeddings  --> Vector DB
    |
Query ---> Multimodal embedding ---> Vector DB search
    |
    v
[Retrieved: text + tables + images]
    |
    v
[Vision LLM generates answer from mixed-modality context]
```

---

## Architecture Decisions

### Strategy 1: Convert Everything to Text (Simplest)

Convert all modalities to text descriptions, then use standard text RAG.

```python
import base64
from pathlib import Path
from anthropic import Anthropic


class TextConversionPipeline:
    """Convert all modalities to text for standard RAG.

    Pros:
    - Uses existing text RAG infrastructure
    - No multimodal embeddings needed
    - Simple to implement

    Cons:
    - Lossy: visual nuance is lost in text descriptions
    - Tables lose structure when linearized
    - Diagram spatial relationships are hard to describe in text
    - Image descriptions are subjective and may miss details
    """

    def __init__(self, anthropic_client: Anthropic | None = None):
        self.client = anthropic_client or Anthropic()

    def image_to_text(self, image_path: str) -> str:
        """Describe an image using Claude's vision capability."""
        image_data = Path(image_path).read_bytes()
        media_type = self._get_media_type(image_path)

        message = self.client.messages.create(
            model="claude-sonnet-4-20250514",
            max_tokens=1024,
            messages=[{
                "role": "user",
                "content": [
                    {
                        "type": "image",
                        "source": {
                            "type": "base64",
                            "media_type": media_type,
                            "data": base64.standard_b64encode(image_data).decode(),
                        },
                    },
                    {
                        "type": "text",
                        "text": (
                            "Describe this image in detail for a knowledge base. "
                            "Include all text, numbers, labels, relationships, "
                            "and spatial arrangements. Be comprehensive."
                        ),
                    },
                ],
            }],
        )
        return message.content[0].text

    def table_to_text(self, table_html: str) -> str:
        """Convert an HTML table to a structured text description."""
        message = self.client.messages.create(
            model="claude-sonnet-4-20250514",
            max_tokens=1024,
            messages=[{
                "role": "user",
                "content": (
                    "Convert this HTML table to a structured text description. "
                    "Preserve all data, column relationships, and any patterns "
                    "in the data. Format as markdown.\n\n"
                    f"Table:\n{table_html}"
                ),
            }],
        )
        return message.content[0].text

    def audio_to_text(self, audio_path: str) -> str:
        """Transcribe audio using a speech-to-text model."""
        import openai

        client = openai.OpenAI()
        with open(audio_path, "rb") as audio_file:
            transcript = client.audio.transcriptions.create(
                model="whisper-1",
                file=audio_file,
                response_format="text",
            )
        return transcript

    @staticmethod
    def _get_media_type(path: str) -> str:
        ext = Path(path).suffix.lower()
        return {
            ".jpg": "image/jpeg",
            ".jpeg": "image/jpeg",
            ".png": "image/png",
            ".gif": "image/gif",
            ".webp": "image/webp",
        }.get(ext, "image/png")
```

### Strategy 2: Multimodal Embeddings (Recommended)

Embed all modalities into a shared vector space using multimodal embedding models.

```python
import httpx
import numpy as np
from dataclasses import dataclass


@dataclass
class MultimodalChunk:
    content: str | bytes
    modality: str  # "text", "image", "table", "audio"
    embedding: list[float] | None = None
    metadata: dict | None = None
    source_document: str = ""
    page_number: int = 0


class MultimodalEmbeddingPipeline:
    """Embed multiple modalities into a shared vector space.

    Uses VoyageAI's multimodal-3 model which natively supports
    text, images, and interleaved text+image inputs.
    """

    def __init__(self, voyage_api_key: str):
        self.api_key = voyage_api_key
        self.base_url = "https://api.voyageai.com/v1"
        self.model = "voyage-multimodal-3"
        self.dimension = 1024

    def embed_text(self, texts: list[str]) -> list[list[float]]:
        """Embed text chunks."""
        response = httpx.post(
            f"{self.base_url}/embeddings",
            headers={"Authorization": f"Bearer {self.api_key}"},
            json={
                "model": self.model,
                "input": texts,
                "input_type": "document",
            },
            timeout=60,
        )
        response.raise_for_status()
        data = response.json()
        return [item["embedding"] for item in data["data"]]

    def embed_images(self, image_paths: list[str]) -> list[list[float]]:
        """Embed images directly (not text descriptions).

        VoyageAI multimodal-3 accepts base64 images and produces
        embeddings in the same vector space as text.
        """
        inputs = []
        for path in image_paths:
            image_data = Path(path).read_bytes()
            media_type = self._get_media_type(path)
            b64 = base64.standard_b64encode(image_data).decode()
            inputs.append([{
                "type": "image_base64",
                "image_base64": b64,
                "media_type": media_type,
            }])

        response = httpx.post(
            f"{self.base_url}/multimodalembeddings",
            headers={"Authorization": f"Bearer {self.api_key}"},
            json={
                "model": self.model,
                "inputs": inputs,
                "input_type": "document",
            },
            timeout=120,
        )
        response.raise_for_status()
        data = response.json()
        return [item["embedding"] for item in data["data"]]

    def embed_image_with_context(
        self, image_path: str, surrounding_text: str
    ) -> list[float]:
        """Embed an image alongside its surrounding text.

        Interleaved inputs produce better embeddings than image-only
        because the text provides semantic context for the image.
        """
        image_data = Path(image_path).read_bytes()
        media_type = self._get_media_type(image_path)
        b64 = base64.standard_b64encode(image_data).decode()

        input_content = [
            {"type": "text", "text": surrounding_text},
            {
                "type": "image_base64",
                "image_base64": b64,
                "media_type": media_type,
            },
        ]

        response = httpx.post(
            f"{self.base_url}/multimodalembeddings",
            headers={"Authorization": f"Bearer {self.api_key}"},
            json={
                "model": self.model,
                "inputs": [input_content],
                "input_type": "document",
            },
            timeout=120,
        )
        response.raise_for_status()
        data = response.json()
        return data["data"][0]["embedding"]

    def embed_query(self, query: str) -> list[float]:
        """Embed a query for retrieval (uses query input_type)."""
        response = httpx.post(
            f"{self.base_url}/embeddings",
            headers={"Authorization": f"Bearer {self.api_key}"},
            json={
                "model": self.model,
                "input": [query],
                "input_type": "query",
            },
            timeout=30,
        )
        response.raise_for_status()
        data = response.json()
        return data["data"][0]["embedding"]

    @staticmethod
    def _get_media_type(path: str) -> str:
        ext = Path(path).suffix.lower()
        return {
            ".jpg": "image/jpeg",
            ".jpeg": "image/jpeg",
            ".png": "image/png",
            ".gif": "image/gif",
            ".webp": "image/webp",
        }.get(ext, "image/png")
```

### Strategy 3: Late Fusion (Pass Raw Content to Vision LLM)

Store raw images and pass them directly to a vision LLM at generation time.

```python
class LateFusionRAG:
    """Store raw multimodal content and pass to vision LLM at query time.

    Instead of converting images to text or embedding them, store
    the raw content and let the vision LLM interpret it during
    answer generation.

    Pros:
    - No information loss from conversion
    - Vision LLM sees the original content
    - Simple storage (just keep the files)

    Cons:
    - Retrieval is text-only (images are not searchable)
    - Higher generation cost (vision tokens are expensive)
    - Cannot retrieve based on image content alone
    """

    def __init__(self, text_retriever, anthropic_client: Anthropic):
        self.text_retriever = text_retriever
        self.client = anthropic_client

    def query(self, question: str) -> str:
        """Retrieve text chunks, collect associated images, generate with vision."""
        # Step 1: Retrieve text chunks (standard text retrieval)
        text_chunks = self.text_retriever.invoke(question)

        # Step 2: Collect associated images from retrieved chunks
        images = []
        for chunk in text_chunks:
            if hasattr(chunk, "metadata") and "images" in chunk.metadata:
                for img_path in chunk.metadata["images"]:
                    images.append(self._load_image(img_path))

        # Step 3: Build multimodal message for vision LLM
        content = []

        # Add text context
        text_context = "\n\n---\n\n".join(
            [chunk.page_content for chunk in text_chunks]
        )
        content.append({"type": "text", "text": f"Context:\n{text_context}"})

        # Add images (up to 5 to manage token budget)
        for img in images[:5]:
            content.append({
                "type": "image",
                "source": {
                    "type": "base64",
                    "media_type": img["media_type"],
                    "data": img["data"],
                },
            })

        content.append({
            "type": "text",
            "text": f"\nQuestion: {question}\n\nAnswer based on the context and images above.",
        })

        # Step 4: Generate with vision LLM
        message = self.client.messages.create(
            model="claude-sonnet-4-20250514",
            max_tokens=2048,
            messages=[{"role": "user", "content": content}],
        )
        return message.content[0].text

    @staticmethod
    def _load_image(path: str) -> dict:
        image_data = Path(path).read_bytes()
        ext = Path(path).suffix.lower()
        media_type = {
            ".jpg": "image/jpeg", ".jpeg": "image/jpeg",
            ".png": "image/png", ".webp": "image/webp",
        }.get(ext, "image/png")
        return {
            "media_type": media_type,
            "data": base64.standard_b64encode(image_data).decode(),
        }
```

---

## Document Parsing for Multimodal Content

### PDF Parsing with Image and Table Extraction

```python
from dataclasses import dataclass, field
from pathlib import Path
from enum import Enum


class ChunkType(Enum):
    TEXT = "text"
    TABLE = "table"
    IMAGE = "image"
    FORMULA = "formula"


@dataclass
class ParsedElement:
    content: str | bytes
    chunk_type: ChunkType
    page_number: int
    bounding_box: tuple[float, float, float, float] | None = None
    metadata: dict = field(default_factory=dict)


class MultimodalPDFParser:
    """Parse PDFs into text, table, and image elements.

    Uses multiple libraries for best extraction quality:
    - PyMuPDF (fitz) for text and image extraction
    - pdfplumber for table detection and extraction
    - Unstructured for layout-aware parsing

    The parser preserves the reading order and associates
    images with their surrounding text for context.
    """

    def __init__(self, image_output_dir: str = "./extracted_images"):
        self.image_output_dir = Path(image_output_dir)
        self.image_output_dir.mkdir(parents=True, exist_ok=True)

    def parse(self, pdf_path: str) -> list[ParsedElement]:
        """Parse a PDF into typed elements."""
        elements: list[ParsedElement] = []

        # Extract images
        image_elements = self._extract_images(pdf_path)
        elements.extend(image_elements)

        # Extract tables
        table_elements = self._extract_tables(pdf_path)
        elements.extend(table_elements)

        # Extract text (excluding table regions)
        text_elements = self._extract_text(pdf_path, table_elements)
        elements.extend(text_elements)

        # Sort by page number and vertical position
        elements.sort(
            key=lambda e: (
                e.page_number,
                e.bounding_box[1] if e.bounding_box else 0,
            )
        )

        return elements

    def _extract_images(self, pdf_path: str) -> list[ParsedElement]:
        """Extract images from PDF using PyMuPDF."""
        import fitz

        elements = []
        doc = fitz.open(pdf_path)

        for page_num, page in enumerate(doc):
            image_list = page.get_images(full=True)

            for img_idx, img_info in enumerate(image_list):
                xref = img_info[0]
                base_image = doc.extract_image(xref)
                image_bytes = base_image["image"]
                image_ext = base_image["ext"]

                # Save image to disk
                image_filename = f"page{page_num}_img{img_idx}.{image_ext}"
                image_path = self.image_output_dir / image_filename
                image_path.write_bytes(image_bytes)

                # Get bounding box
                rects = page.get_image_rects(xref)
                bbox = None
                if rects:
                    r = rects[0]
                    bbox = (r.x0, r.y0, r.x1, r.y1)

                # Skip very small images (likely icons/bullets)
                if bbox:
                    width = bbox[2] - bbox[0]
                    height = bbox[3] - bbox[1]
                    if width < 50 or height < 50:
                        continue

                elements.append(ParsedElement(
                    content=str(image_path),
                    chunk_type=ChunkType.IMAGE,
                    page_number=page_num,
                    bounding_box=bbox,
                    metadata={
                        "image_path": str(image_path),
                        "format": image_ext,
                        "width": bbox[2] - bbox[0] if bbox else 0,
                        "height": bbox[3] - bbox[1] if bbox else 0,
                    },
                ))

        doc.close()
        return elements

    def _extract_tables(self, pdf_path: str) -> list[ParsedElement]:
        """Extract tables from PDF using pdfplumber."""
        import pdfplumber

        elements = []

        with pdfplumber.open(pdf_path) as pdf:
            for page_num, page in enumerate(pdf.pages):
                tables = page.extract_tables()

                for table_idx, table in enumerate(tables):
                    if not table or len(table) < 2:
                        continue

                    # Convert to markdown
                    markdown = self._table_to_markdown(table)

                    # Get table bounding box
                    table_settings = page.find_tables()
                    bbox = None
                    if table_idx < len(table_settings):
                        tb = table_settings[table_idx]
                        bbox = tb.bbox

                    elements.append(ParsedElement(
                        content=markdown,
                        chunk_type=ChunkType.TABLE,
                        page_number=page_num,
                        bounding_box=bbox,
                        metadata={
                            "rows": len(table),
                            "cols": len(table[0]) if table else 0,
                            "table_index": table_idx,
                        },
                    ))

        return elements

    def _extract_text(
        self, pdf_path: str, table_elements: list[ParsedElement]
    ) -> list[ParsedElement]:
        """Extract text from PDF, excluding regions covered by tables."""
        import fitz

        elements = []
        doc = fitz.open(pdf_path)

        # Build exclusion zones from table bounding boxes
        table_zones: dict[int, list[tuple]] = {}
        for te in table_elements:
            if te.bounding_box:
                table_zones.setdefault(te.page_number, []).append(te.bounding_box)

        for page_num, page in enumerate(doc):
            # Get all text blocks
            blocks = page.get_text("blocks")
            zones = table_zones.get(page_num, [])

            page_text_parts = []
            for block in blocks:
                x0, y0, x1, y1, text, block_no, block_type = block[:7]

                # Skip image blocks
                if block_type != 0:
                    continue

                # Skip blocks inside table zones
                if self._is_inside_table(
                    (x0, y0, x1, y1), zones
                ):
                    continue

                text = text.strip()
                if text:
                    page_text_parts.append(text)

            if page_text_parts:
                full_text = "\n\n".join(page_text_parts)
                elements.append(ParsedElement(
                    content=full_text,
                    chunk_type=ChunkType.TEXT,
                    page_number=page_num,
                    metadata={"char_count": len(full_text)},
                ))

        doc.close()
        return elements

    @staticmethod
    def _table_to_markdown(table: list[list]) -> str:
        """Convert a parsed table to markdown format."""
        if not table:
            return ""

        # Clean cells
        cleaned = []
        for row in table:
            cleaned_row = [
                str(cell).strip() if cell else "" for cell in row
            ]
            cleaned.append(cleaned_row)

        # Header
        header = "| " + " | ".join(cleaned[0]) + " |"
        separator = "| " + " | ".join(["---"] * len(cleaned[0])) + " |"

        # Rows
        rows = []
        for row in cleaned[1:]:
            # Pad or truncate row to match header width
            padded = row + [""] * (len(cleaned[0]) - len(row))
            rows.append("| " + " | ".join(padded[: len(cleaned[0])]) + " |")

        return "\n".join([header, separator] + rows)

    @staticmethod
    def _is_inside_table(
        block_bbox: tuple, table_zones: list[tuple], overlap_threshold: float = 0.5
    ) -> bool:
        """Check if a text block overlaps with a table zone."""
        bx0, by0, bx1, by1 = block_bbox
        block_area = max((bx1 - bx0) * (by1 - by0), 1)

        for tx0, ty0, tx1, ty1 in table_zones:
            # Calculate intersection
            ix0 = max(bx0, tx0)
            iy0 = max(by0, ty0)
            ix1 = min(bx1, tx1)
            iy1 = min(by1, ty1)

            if ix0 < ix1 and iy0 < iy1:
                intersection_area = (ix1 - ix0) * (iy1 - iy0)
                if intersection_area / block_area > overlap_threshold:
                    return True

        return False
```

---

## Audio and Video Pipeline

```python
from dataclasses import dataclass


@dataclass
class AudioSegment:
    text: str
    start_time: float
    end_time: float
    speaker: str | None = None


class AudioVideoPipeline:
    """Process audio and video for RAG.

    Audio/video processing pipeline:
    1. Extract audio from video (if video)
    2. Transcribe with timestamps using Whisper
    3. Diarize speakers (optional, using pyannote)
    4. Chunk by speaker turns or time windows
    5. Embed transcript chunks
    6. Store with temporal metadata for citation
    """

    def __init__(self, openai_client=None):
        import openai
        self.client = openai_client or openai.OpenAI()

    def transcribe_audio(
        self, audio_path: str, language: str | None = None
    ) -> list[AudioSegment]:
        """Transcribe audio with word-level timestamps."""
        with open(audio_path, "rb") as f:
            response = self.client.audio.transcriptions.create(
                model="whisper-1",
                file=f,
                response_format="verbose_json",
                timestamp_granularities=["segment"],
                language=language,
            )

        segments = []
        for seg in response.segments:
            segments.append(AudioSegment(
                text=seg["text"].strip(),
                start_time=seg["start"],
                end_time=seg["end"],
            ))

        return segments

    def extract_audio_from_video(
        self, video_path: str, output_path: str
    ) -> str:
        """Extract audio track from video file."""
        import subprocess

        subprocess.run(
            [
                "ffmpeg", "-i", video_path,
                "-vn",  # no video
                "-acodec", "pcm_s16le",
                "-ar", "16000",  # 16kHz sample rate
                "-ac", "1",  # mono
                output_path,
            ],
            check=True,
            capture_output=True,
        )
        return output_path

    def chunk_by_time_window(
        self,
        segments: list[AudioSegment],
        window_seconds: float = 60.0,
        overlap_seconds: float = 10.0,
    ) -> list[dict]:
        """Chunk transcript segments into time-windowed chunks.

        Each chunk covers a time window with overlap for context continuity.
        """
        if not segments:
            return []

        chunks = []
        total_duration = segments[-1].end_time
        window_start = 0.0

        while window_start < total_duration:
            window_end = window_start + window_seconds

            # Collect segments in this window
            window_segments = [
                s for s in segments
                if s.start_time < window_end and s.end_time > window_start
            ]

            if window_segments:
                text = " ".join(s.text for s in window_segments)
                chunks.append({
                    "text": text,
                    "start_time": window_start,
                    "end_time": min(window_end, total_duration),
                    "segment_count": len(window_segments),
                    "speakers": list(set(
                        s.speaker for s in window_segments if s.speaker
                    )),
                })

            window_start += window_seconds - overlap_seconds

        return chunks

    def extract_video_frames(
        self, video_path: str, output_dir: str, interval_seconds: float = 30.0
    ) -> list[dict]:
        """Extract key frames from video at regular intervals.

        These frames can be embedded using the vision pipeline
        and stored alongside transcript chunks for multimodal retrieval.
        """
        import subprocess
        import json

        output_path = Path(output_dir)
        output_path.mkdir(parents=True, exist_ok=True)

        # Get video duration
        probe = subprocess.run(
            [
                "ffprobe", "-v", "quiet",
                "-print_format", "json",
                "-show_format", video_path,
            ],
            capture_output=True,
            text=True,
        )
        duration = float(json.loads(probe.stdout)["format"]["duration"])

        # Extract frames at intervals
        frames = []
        timestamp = 0.0
        frame_idx = 0

        while timestamp < duration:
            frame_path = output_path / f"frame_{frame_idx:04d}.jpg"
            subprocess.run(
                [
                    "ffmpeg", "-ss", str(timestamp),
                    "-i", video_path,
                    "-vframes", "1",
                    "-q:v", "2",
                    str(frame_path),
                ],
                capture_output=True,
                check=True,
            )

            frames.append({
                "frame_path": str(frame_path),
                "timestamp": timestamp,
                "frame_index": frame_idx,
            })

            timestamp += interval_seconds
            frame_idx += 1

        return frames
```

---

## End-to-End Multimodal RAG Pipeline

```python
from typing import Any


class MultimodalRAGPipeline:
    """Complete multimodal RAG pipeline from document ingestion to query.

    Combines all modality-specific pipelines into a unified system
    with a shared vector store for cross-modal retrieval.
    """

    def __init__(
        self,
        embedding_pipeline: MultimodalEmbeddingPipeline,
        pdf_parser: MultimodalPDFParser,
        vision_describer: TextConversionPipeline,
        vector_store,
        llm_client: Anthropic,
    ):
        self.embedder = embedding_pipeline
        self.parser = pdf_parser
        self.describer = vision_describer
        self.vector_store = vector_store
        self.llm = llm_client

    def ingest_document(self, document_path: str) -> dict:
        """Parse, embed, and index a multimodal document."""
        ext = Path(document_path).suffix.lower()

        if ext == ".pdf":
            elements = self.parser.parse(document_path)
        else:
            raise ValueError(f"Unsupported format: {ext}")

        stats = {"text": 0, "table": 0, "image": 0}

        for element in elements:
            if element.chunk_type == ChunkType.TEXT:
                embedding = self.embedder.embed_text([element.content])[0]
                self.vector_store.upsert(
                    id=f"{document_path}:text:{element.page_number}",
                    embedding=embedding,
                    metadata={
                        "content": element.content,
                        "type": "text",
                        "source": document_path,
                        "page": element.page_number,
                    },
                )
                stats["text"] += 1

            elif element.chunk_type == ChunkType.TABLE:
                # Embed the markdown representation of the table
                embedding = self.embedder.embed_text([element.content])[0]
                self.vector_store.upsert(
                    id=f"{document_path}:table:{element.page_number}:{element.metadata.get('table_index', 0)}",
                    embedding=embedding,
                    metadata={
                        "content": element.content,
                        "type": "table",
                        "source": document_path,
                        "page": element.page_number,
                        "rows": element.metadata.get("rows", 0),
                        "cols": element.metadata.get("cols", 0),
                    },
                )
                stats["table"] += 1

            elif element.chunk_type == ChunkType.IMAGE:
                image_path = element.metadata["image_path"]
                # Generate text description for metadata
                description = self.describer.image_to_text(image_path)
                # Embed the image directly
                embedding = self.embedder.embed_images([image_path])[0]
                self.vector_store.upsert(
                    id=f"{document_path}:image:{element.page_number}",
                    embedding=embedding,
                    metadata={
                        "content": description,
                        "type": "image",
                        "image_path": image_path,
                        "source": document_path,
                        "page": element.page_number,
                    },
                )
                stats["image"] += 1

        return stats

    def query(self, question: str, top_k: int = 8) -> dict:
        """Query the multimodal index and generate an answer."""
        # Embed query
        query_embedding = self.embedder.embed_query(question)

        # Retrieve from all modalities
        results = self.vector_store.query(query_embedding, top_k=top_k)

        # Build multimodal context for the vision LLM
        message_content = []
        sources = []

        for result in results:
            meta = result["metadata"]
            result_type = meta["type"]

            if result_type in ("text", "table"):
                message_content.append({
                    "type": "text",
                    "text": f"[{result_type.upper()} from {meta['source']} p.{meta['page']}]\n{meta['content']}",
                })
            elif result_type == "image":
                # Include both the image and its description
                image_path = meta.get("image_path")
                if image_path and Path(image_path).exists():
                    image_data = Path(image_path).read_bytes()
                    message_content.append({
                        "type": "image",
                        "source": {
                            "type": "base64",
                            "media_type": "image/png",
                            "data": base64.standard_b64encode(image_data).decode(),
                        },
                    })
                message_content.append({
                    "type": "text",
                    "text": f"[IMAGE DESCRIPTION from {meta['source']} p.{meta['page']}]\n{meta['content']}",
                })

            sources.append({
                "source": meta["source"],
                "page": meta["page"],
                "type": result_type,
                "score": result.get("score", 0),
            })

        message_content.append({
            "type": "text",
            "text": f"\nQuestion: {question}\n\nAnswer comprehensively based on all the context above (text, tables, and images).",
        })

        # Generate answer with vision LLM
        response = self.llm.messages.create(
            model="claude-sonnet-4-20250514",
            max_tokens=2048,
            messages=[{"role": "user", "content": message_content}],
        )

        return {
            "answer": response.content[0].text,
            "sources": sources,
            "modalities_retrieved": list(set(s["type"] for s in sources)),
        }
```

---

## Cost Comparison

| Approach | Indexing Cost (1000 pages) | Query Cost | Retrieval Quality |
|----------|--------------------------|------------|-------------------|
| Text-only RAG | $0.50 (embeddings) | $0.01-0.05 | Misses 30-40% of content |
| Convert-to-text | $15-30 (vision LLM) | $0.01-0.05 | Good but lossy |
| Multimodal embeddings | $2-5 (VoyageAI) | $0.01-0.05 | Best for retrieval |
| Late fusion | $0.50 (text only) | $0.10-0.50 (vision tokens) | Best for generation |
| Hybrid (recommended) | $5-10 | $0.05-0.15 | Best overall |

---

## Common Pitfalls

1. **Ignoring table structure.** Flattening tables to "col1: val1, col2: val2" loses relationships between columns. Always convert to markdown tables or structured formats.

2. **Embedding tiny images.** Icons, bullets, and decorative images add noise. Filter by minimum size (at least 100x100 pixels) and ignore images inside table cells.

3. **Not pairing images with context.** An image of a graph without its caption and surrounding text is nearly meaningless. Always embed images with their nearby text.

4. **Exceeding vision LLM token limits.** Each image consumes 1000-4000 tokens. Limit to 5-8 images per query to stay within budget.

5. **Mixing modality-specific vector stores.** Use a single shared vector space (via multimodal embeddings) rather than separate stores per modality. Separate stores require manual result merging and cannot do cross-modal similarity.

6. **Skipping OCR for scanned documents.** If PDFs contain scanned images of text, PyMuPDF's text extraction returns nothing. Use OCR (Tesseract or cloud vision APIs) before the parsing pipeline.

---

## References

- VoyageAI Multimodal Embeddings: https://docs.voyageai.com/docs/multimodal-embeddings
- Anthropic Vision: https://docs.anthropic.com/en/docs/build-with-claude/vision
- OpenAI Whisper: https://platform.openai.com/docs/guides/speech-to-text
- PyMuPDF: https://pymupdf.readthedocs.io/
- pdfplumber: https://github.com/jsvine/pdfplumber
- Unstructured: https://docs.unstructured.io/
