# Video RAG -- Comprehensive Guide

## Overview / TL;DR

Video RAG combines multiple modalities -- audio transcription, visual keyframe extraction, and text-based metadata -- to make video content searchable and retrievable by LLMs. A one-hour video contains a transcript, hundreds of visual scenes, on-screen text, slides, and diagrams that together convey information no single modality captures alone. This guide covers keyframe extraction (PySceneDetect, ffmpeg), transcript generation (Whisper), visual embeddings (CLIP), on-screen text (OCR), and production-ready code for building unified multimodal video indexes.

---

## Why Video RAG Requires Multiple Modalities

1. **Audio carries the narrative.** Lectures, meetings, and podcasts communicate primarily through speech.
2. **Visuals carry structure.** Slides, diagrams, code demos, and whiteboard drawings convey information that speech references but does not fully describe.
3. **On-screen text is not spoken.** Slide titles, code, URLs, and annotations appear visually but may never be mentioned aloud.
4. **Temporal alignment matters.** A query like "What was shown when the speaker discussed authentication?" requires linking transcript timestamps to visual frames.

---

## Keyframe Extraction

### PySceneDetect

Detects scene changes using content-aware algorithms to extract representative frames.

```python
from scenedetect import open_video, SceneManager, ContentDetector
from scenedetect.scene_manager import save_images
from pathlib import Path


def extract_keyframes_scenedetect(
    video_path: str,
    output_dir: str,
    threshold: float = 27.0,
    min_scene_len: int = 30,
) -> list[dict]:
    """Extract keyframes at scene boundaries using PySceneDetect."""
    output = Path(output_dir)
    output.mkdir(parents=True, exist_ok=True)

    video = open_video(video_path)
    scene_manager = SceneManager()
    scene_manager.add_detector(
        ContentDetector(
            threshold=threshold,
            min_scene_len=min_scene_len,
        )
    )

    scene_manager.detect_scenes(video)
    scene_list = scene_manager.get_scene_list()

    # Save keyframe images
    save_images(
        scene_list,
        video,
        num_images=1,         # One frame per scene
        output_dir=str(output),
        image_name_template="$SCENE_NUMBER",
    )

    keyframes = []
    for i, (start, end) in enumerate(scene_list):
        keyframes.append({
            "scene_index": i,
            "start_time": start.get_seconds(),
            "end_time": end.get_seconds(),
            "duration": (end - start).get_seconds(),
            "start_timecode": str(start),
            "end_timecode": str(end),
            "image_path": str(output / f"{i + 1:03d}-001.jpg"),
        })

    return keyframes
```

### ffmpeg-based Extraction

```python
import subprocess
import json
from pathlib import Path


def extract_keyframes_ffmpeg(
    video_path: str,
    output_dir: str,
    interval_seconds: int = 30,
) -> list[dict]:
    """Extract frames at regular intervals using ffmpeg."""
    output = Path(output_dir)
    output.mkdir(parents=True, exist_ok=True)

    # Extract frames at interval
    cmd = [
        "ffmpeg", "-i", video_path,
        "-vf", f"fps=1/{interval_seconds}",
        "-q:v", "2",
        str(output / "frame_%04d.jpg"),
        "-y",
    ]

    subprocess.run(cmd, capture_output=True, check=True)

    frames = sorted(output.glob("frame_*.jpg"))
    keyframes = []

    for i, frame_path in enumerate(frames):
        keyframes.append({
            "frame_index": i,
            "timestamp": i * interval_seconds,
            "image_path": str(frame_path),
        })

    return keyframes


def extract_scene_changes_ffmpeg(
    video_path: str,
    output_dir: str,
    scene_threshold: float = 0.3,
) -> list[dict]:
    """Extract frames at scene changes detected by ffmpeg."""
    output = Path(output_dir)
    output.mkdir(parents=True, exist_ok=True)

    # Detect scenes
    cmd = [
        "ffmpeg", "-i", video_path,
        "-vf", f"select='gt(scene,{scene_threshold})',showinfo",
        "-vsync", "vfr",
        "-q:v", "2",
        str(output / "scene_%04d.jpg"),
        "-y",
    ]

    result = subprocess.run(cmd, capture_output=True, text=True)

    frames = sorted(output.glob("scene_*.jpg"))
    return [
        {
            "frame_index": i,
            "image_path": str(f),
        }
        for i, f in enumerate(frames)
    ]
```

---

## Visual Embeddings with CLIP

```python
import torch
from PIL import Image
from transformers import CLIPProcessor, CLIPModel


class VideoVisualEncoder:
    """Encode video keyframes using CLIP for visual search."""

    def __init__(self, model_name: str = "openai/clip-vit-base-patch32"):
        self.model = CLIPModel.from_pretrained(model_name)
        self.processor = CLIPProcessor.from_pretrained(model_name)
        self.model.eval()

    def encode_images(self, image_paths: list[str]) -> list[list[float]]:
        """Encode a batch of images into CLIP embeddings."""
        images = [Image.open(p).convert("RGB") for p in image_paths]

        inputs = self.processor(
            images=images,
            return_tensors="pt",
            padding=True,
        )

        with torch.no_grad():
            outputs = self.model.get_image_features(**inputs)

        # Normalize embeddings
        embeddings = outputs / outputs.norm(dim=-1, keepdim=True)
        return embeddings.cpu().numpy().tolist()

    def encode_text(self, queries: list[str]) -> list[list[float]]:
        """Encode text queries for visual search."""
        inputs = self.processor(
            text=queries,
            return_tensors="pt",
            padding=True,
            truncation=True,
        )

        with torch.no_grad():
            outputs = self.model.get_text_features(**inputs)

        embeddings = outputs / outputs.norm(dim=-1, keepdim=True)
        return embeddings.cpu().numpy().tolist()

    def search_frames(
        self,
        query: str,
        frame_embeddings: list[list[float]],
        frame_metadata: list[dict],
        top_k: int = 5,
    ) -> list[dict]:
        """Search keyframes by text query using CLIP."""
        import numpy as np

        query_emb = np.array(self.encode_text([query])[0])
        frame_embs = np.array(frame_embeddings)

        similarities = np.dot(frame_embs, query_emb)
        top_indices = np.argsort(similarities)[-top_k:][::-1]

        results = []
        for idx in top_indices:
            results.append({
                **frame_metadata[idx],
                "similarity": float(similarities[idx]),
            })

        return results
```

---

## On-Screen Text Extraction (OCR on Keyframes)

```python
import pytesseract
from PIL import Image


def extract_onscreen_text(
    keyframes: list[dict],
    min_confidence: int = 60,
) -> list[dict]:
    """Extract on-screen text from keyframes using OCR."""
    results = []

    for kf in keyframes:
        image = Image.open(kf["image_path"])
        data = pytesseract.image_to_data(
            image, output_type=pytesseract.Output.DICT,
        )

        words = []
        for i in range(len(data["text"])):
            text = data["text"][i].strip()
            conf = int(data["conf"][i])
            if text and conf >= min_confidence:
                words.append(text)

        onscreen_text = " ".join(words)

        results.append({
            **kf,
            "onscreen_text": onscreen_text,
            "word_count": len(words),
        })

    return results
```

---

## Unified Multimodal Chunks

```python
from dataclasses import dataclass


@dataclass
class VideoChunk:
    text: str
    metadata: dict
    embedding_type: str  # "text", "visual", or "multimodal"


def create_multimodal_chunks(
    transcript_segments: list[dict],
    keyframes: list[dict],
    video_path: str,
    chunk_duration_seconds: int = 60,
) -> list[VideoChunk]:
    """Create unified chunks combining transcript and visual data."""
    # Get video duration from last segment or keyframe
    max_time = 0
    if transcript_segments:
        max_time = max(s.get("end", 0) for s in transcript_segments)
    if keyframes:
        max_time = max(max_time, max(kf.get("start_time", kf.get("timestamp", 0)) for kf in keyframes))

    chunks = []

    for window_start in range(0, int(max_time) + 1, chunk_duration_seconds):
        window_end = window_start + chunk_duration_seconds

        # Collect transcript text in this window
        window_text = []
        for seg in transcript_segments:
            if seg["start"] >= window_start and seg["start"] < window_end:
                speaker = seg.get("speaker", "")
                if speaker:
                    window_text.append(f"[{speaker}]: {seg['text']}")
                else:
                    window_text.append(seg["text"])

        # Collect on-screen text from keyframes in this window
        onscreen = []
        frame_paths = []
        for kf in keyframes:
            kf_time = kf.get("start_time", kf.get("timestamp", 0))
            if kf_time >= window_start and kf_time < window_end:
                if kf.get("onscreen_text"):
                    onscreen.append(kf["onscreen_text"])
                frame_paths.append(kf["image_path"])

        if not window_text and not onscreen:
            continue

        # Build chunk text
        parts = []
        parts.append(f"[{_format_timestamp(window_start)} - {_format_timestamp(window_end)}]")

        if window_text:
            parts.append("\nTranscript:")
            parts.extend(window_text)

        if onscreen:
            parts.append("\nOn-screen text:")
            parts.extend(onscreen)

        chunk_text = "\n".join(parts)

        chunks.append(VideoChunk(
            text=chunk_text,
            metadata={
                "source": video_path,
                "start_time": window_start,
                "end_time": window_end,
                "has_transcript": bool(window_text),
                "has_onscreen_text": bool(onscreen),
                "keyframe_paths": frame_paths,
                "chunk_type": "video_multimodal",
            },
            embedding_type="text",
        ))

    return chunks


def _format_timestamp(seconds: float) -> str:
    """Format seconds as HH:MM:SS."""
    h = int(seconds // 3600)
    m = int((seconds % 3600) // 60)
    s = int(seconds % 60)
    return f"{h:02d}:{m:02d}:{s:02d}"
```

---

## Common Pitfalls

1. **Relying only on transcript.** Visual content (slides, diagrams, code) contains information never spoken aloud.
2. **Extracting too many keyframes.** One frame every 30 seconds creates thousands of images for a long video. Use scene detection instead.
3. **Not aligning transcript with visual frames.** Temporal alignment is essential for queries that reference what was shown during specific speech.
4. **Using generic text embeddings for visual search.** CLIP embeddings enable text-to-image search that regular text embeddings cannot.
5. **Ignoring on-screen text.** Slide titles, code, and URLs visible on screen should be OCR'd and included in chunks.
6. **Not chunking by time windows.** Fixed time-window chunks maintain temporal coherence between transcript and visual data.
7. **Processing full-resolution frames.** Resize keyframes to 720p or smaller for OCR and CLIP. Full resolution wastes compute.
8. **Missing video metadata.** Duration, title, speaker, and date should be captured from the video file or upload context.

---

## Production Checklist

- [ ] Keyframes are extracted at scene changes, not fixed intervals
- [ ] Audio transcript is generated with timestamps and speaker labels
- [ ] On-screen text is OCR'd from keyframes
- [ ] Visual embeddings (CLIP) enable text-to-image search
- [ ] Multimodal chunks combine transcript + on-screen text per time window
- [ ] Temporal metadata (start_time, end_time) enables video seeking
- [ ] Keyframe paths are stored for visual display in results
- [ ] Video metadata (title, duration, speakers) is preserved
- [ ] Long videos are processed in segments to manage memory
- [ ] Duplicate frames (static scenes) are deduplicated

---

## References

- PySceneDetect -- https://www.scenedetect.com/docs/latest/
- ffmpeg -- https://ffmpeg.org/documentation.html
- CLIP -- https://github.com/openai/CLIP
- Whisper -- https://github.com/openai/whisper
- faster-whisper -- https://github.com/SYSTRAN/faster-whisper
