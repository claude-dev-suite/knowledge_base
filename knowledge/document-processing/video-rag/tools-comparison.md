# Video RAG -- Tools Comparison

## Overview / TL;DR

Building a video RAG pipeline requires combining tools for scene detection, transcription, visual encoding, and OCR. This guide compares the key tools at each stage: PySceneDetect vs ffmpeg for keyframe extraction, CLIP vs other visual encoders, Whisper variants for audio, and OCR options for on-screen text, with benchmarks and a decision framework.

---

## Keyframe Extraction Tools

| Feature | PySceneDetect | ffmpeg (scene filter) | ffmpeg (interval) | Katna | decord |
|---------|-------------|---------------------|-------------------|-------|--------|
| **Method** | Content-aware | Scene change detection | Fixed interval | ML-based | Frame seeking |
| **Scene detection** | ContentDetector, AdaptiveDetector, HashDetector | `select='gt(scene,X)'` | N/A | Smart keyframe | N/A |
| **Quality** | Excellent | Good | Depends on interval | Good | N/A |
| **Speed** | Fast | Very Fast | Very Fast | Medium | Very Fast |
| **Frame saving** | Built-in | Built-in | Built-in | Built-in | Manual |
| **Python API** | Yes | subprocess | subprocess | Yes | Yes |
| **Dependencies** | OpenCV | ffmpeg binary | ffmpeg binary | OpenCV, scipy | Minimal |
| **Customization** | High (thresholds, min_scene_len) | Limited | Interval only | Limited | Full control |

### Benchmark

```python
import time
from pathlib import Path


def benchmark_keyframe_extraction(video_path: str) -> dict:
    """Compare keyframe extraction methods."""
    results = {}

    # PySceneDetect
    try:
        from scenedetect import open_video, SceneManager, ContentDetector
        start = time.perf_counter()
        video = open_video(video_path)
        manager = SceneManager()
        manager.add_detector(ContentDetector(threshold=27.0))
        manager.detect_scenes(video)
        scenes = manager.get_scene_list()
        elapsed = time.perf_counter() - start

        results["pyscenedetect"] = {
            "scenes": len(scenes),
            "time_seconds": round(elapsed, 2),
        }
    except ImportError:
        pass

    # ffmpeg scene detection
    import subprocess
    start = time.perf_counter()
    cmd = [
        "ffmpeg", "-i", video_path,
        "-vf", "select='gt(scene,0.3)',showinfo",
        "-vsync", "vfr",
        "-f", "null", "-",
    ]
    result = subprocess.run(cmd, capture_output=True, text=True)
    elapsed = time.perf_counter() - start
    scene_count = result.stderr.count("pts_time:")
    results["ffmpeg_scene"] = {
        "scenes": scene_count,
        "time_seconds": round(elapsed, 2),
    }

    return results
```

**Typical results (1-hour video)**:

| Tool | Scenes Detected | Time (seconds) | Memory |
|------|----------------|----------------|--------|
| PySceneDetect (content) | 80-150 | 30-60 | ~500MB |
| PySceneDetect (adaptive) | 60-120 | 35-70 | ~500MB |
| ffmpeg (scene=0.3) | 100-200 | 15-30 | ~100MB |
| ffmpeg (30s interval) | 120 (fixed) | 10-20 | ~50MB |

---

## Visual Embedding Models

| Model | Dimensions | Image Size | Speed (img/s GPU) | Text-Image Search | License |
|-------|-----------|-----------|-------------------|-------------------|---------|
| CLIP ViT-B/32 | 512 | 224x224 | 200+ | Yes | MIT |
| CLIP ViT-L/14 | 768 | 224x224 | 80+ | Yes | MIT |
| SigLIP | 768 | 384x384 | 60+ | Yes | Apache 2.0 |
| OpenCLIP ViT-G/14 | 1024 | 224x224 | 30+ | Yes | MIT |
| BLIP-2 | 768 | 224x224 | 40+ | Yes + captioning | BSD |
| Voyage Multimodal | 1024 | Variable | API | Yes | Commercial |
| Amazon Titan Multimodal | 1024 | Variable | API | Yes | Pay-per-use |

```python
def compare_visual_encoders(image_paths: list[str]) -> dict:
    """Compare visual encoding speed and embedding dimensions."""
    import time
    results = {}

    # CLIP ViT-B/32
    try:
        from transformers import CLIPModel, CLIPProcessor
        from PIL import Image
        import torch

        model = CLIPModel.from_pretrained("openai/clip-vit-base-patch32")
        processor = CLIPProcessor.from_pretrained("openai/clip-vit-base-patch32")
        model.eval()

        images = [Image.open(p).convert("RGB") for p in image_paths]
        start = time.perf_counter()

        with torch.no_grad():
            inputs = processor(images=images, return_tensors="pt")
            outputs = model.get_image_features(**inputs)

        elapsed = time.perf_counter() - start
        results["clip_vit_b32"] = {
            "dimensions": outputs.shape[-1],
            "images": len(image_paths),
            "time_seconds": round(elapsed, 3),
            "images_per_second": round(len(image_paths) / elapsed, 1),
        }
    except ImportError:
        pass

    return results
```

---

## Transcription Tools for Video

| Feature | faster-whisper | Whisper (OpenAI) | AssemblyAI | Deepgram |
|---------|---------------|-----------------|------------|---------|
| Speed (1hr video) | 3-8 min | 15-30 min | 2-5 min | 1-3 min |
| Word timestamps | Yes | Yes | Yes | Yes |
| Speaker diarization | External (pyannote) | External | Built-in | Built-in |
| Language detection | Yes | Yes | Yes | Yes |
| Cost | Free (GPU compute) | Free (GPU compute) | $0.25-0.65/hr | $0.25-0.75/hr |
| Local processing | Yes | Yes | No | No |

---

## OCR Tools for Video Frames

| Tool | Speed (frame) | Accuracy (slides) | Accuracy (code) | Notes |
|------|-------------|-------------------|------------------|-------|
| Tesseract | 0.5-1s | 90-95% | 80-90% | Free, needs preprocessing |
| PaddleOCR | 0.3-0.8s | 92-97% | 85-93% | Best accuracy/speed |
| EasyOCR | 1-3s | 88-94% | 80-90% | Easy setup |
| Claude Vision | 2-5s | 95-99% | 90-98% | Best quality, expensive |

---

## Cost Analysis for 100 Hours of Video

| Component | Tool | Cost |
|-----------|------|------|
| Keyframe extraction | PySceneDetect | ~$2 (compute) |
| Transcription | faster-whisper (GPU) | ~$22 (g4dn.xlarge) |
| Transcription | AssemblyAI (best) | ~$65 |
| Diarization | pyannote (GPU) | ~$15 (g4dn.xlarge) |
| Diarization | AssemblyAI | Included |
| Visual embeddings | CLIP (GPU) | ~$5 (g4dn.xlarge) |
| OCR on keyframes | Tesseract (CPU) | ~$1 |
| **Total (self-hosted)** | | **~$45** |
| **Total (cloud APIs)** | | **~$75** |

---

## Decision Framework

```
What type of video content?
  |
  +---> Lectures/presentations (slides + speech)
  |     --> Scene detection for slides + Whisper + OCR on keyframes
  |
  +---> Meetings (talking heads, screenshare)
  |     --> Interval keyframes + Whisper with diarization + OCR on screenshares
  |
  +---> Tutorials/demos (code + narration)
  |     --> Frequent keyframes + Whisper + OCR for code
  |
  +---> Visual-heavy (product demos, design reviews)
  |     --> Scene detection + CLIP visual embeddings + Whisper
  |
  +---> Podcast (audio-only or minimal video)
        --> Whisper only (skip visual processing)

Which keyframe extraction?
  |
  +---> Content-aware scenes needed --> PySceneDetect
  +---> Simple interval sampling --> ffmpeg
  +---> Minimal dependencies --> ffmpeg

Which transcription?
  |
  +---> Need diarization + local processing --> faster-whisper + pyannote
  +---> Need managed + diarization --> AssemblyAI or Deepgram
  +---> Audio only, no diarization needed --> faster-whisper
```

---

## Common Pitfalls

1. **Processing all frames instead of keyframes.** A 1-hour video at 30fps has 108,000 frames. Scene detection reduces this to 50-200.
2. **Not aligning visual and audio timelines.** CLIP embeddings and transcript segments must share the same time axis for multimodal retrieval.
3. **Using fixed intervals for slide presentations.** Slides may stay visible for 5 minutes or change every 10 seconds. Scene detection adapts to content.
4. **Ignoring OCR for on-screen text.** Slide text, code, and annotations visible in frames are often more searchable than spoken words.
5. **Embedding keyframes without temporal metadata.** A CLIP match is useless without knowing when in the video the frame appears.
6. **Not deduplicating static scenes.** Long static shots produce identical keyframes. Deduplicate by perceptual hash.

---

## References

- PySceneDetect -- https://www.scenedetect.com/docs/latest/
- ffmpeg documentation -- https://ffmpeg.org/documentation.html
- CLIP -- https://github.com/openai/CLIP
- OpenCLIP -- https://github.com/mlfoundations/open_clip
- faster-whisper -- https://github.com/SYSTRAN/faster-whisper
- AssemblyAI -- https://www.assemblyai.com/docs
- Deepgram -- https://developers.deepgram.com/docs
