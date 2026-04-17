# Video RAG -- Production Pipeline

## Overview / TL;DR

A production video RAG pipeline must extract keyframes, transcribe audio with speaker labels, OCR on-screen text, compute visual embeddings, align all modalities temporally, and produce unified chunks for a multimodal index. This guide provides a complete async pipeline from video files to indexed multimodal chunks.

---

## Architecture

```
Video File (mp4, mkv, webm, avi)
    |
    v
[1] Video Analysis
    |-- extract metadata (duration, resolution, fps)
    |-- extract audio track
    |-- detect scenes / extract keyframes
    |
    v
[2] Parallel Processing
    |-- Audio: transcribe + diarize
    |-- Visual: keyframe OCR + CLIP embeddings
    |
    v
[3] Temporal Alignment
    |-- align transcript segments to keyframes
    |-- merge on-screen text with transcript
    |
    v
[4] Multimodal Chunking
    |-- time-window chunks (transcript + visual)
    |-- visual-only chunks (CLIP embeddings)
    |
    v
[5] Indexing
    |-- text chunks -> text embedding index
    |-- visual chunks -> CLIP embedding index
    |-- unified search across both
```

---

## Pipeline Implementation

```python
import asyncio
import logging
import subprocess
import time
import tempfile
from dataclasses import dataclass, field
from pathlib import Path

logger = logging.getLogger(__name__)


@dataclass
class VideoConfig:
    # Keyframe extraction
    scene_threshold: float = 27.0
    min_scene_length: int = 30      # frames
    max_keyframes: int = 500

    # Transcription
    whisper_model: str = "large-v3"
    whisper_device: str = "cuda"
    diarize: bool = True

    # Visual
    clip_model: str = "openai/clip-vit-base-patch32"
    ocr_keyframes: bool = True

    # Chunking
    chunk_duration_seconds: int = 60
    min_chunk_chars: int = 50


@dataclass
class VideoMetadata:
    path: str
    duration_seconds: float
    width: int
    height: int
    fps: float
    audio_codec: str
    has_audio: bool


@dataclass
class ProcessedVideo:
    metadata: VideoMetadata
    keyframes: list[dict] = field(default_factory=list)
    transcript_segments: list[dict] = field(default_factory=list)
    text_chunks: list[dict] = field(default_factory=list)
    visual_chunks: list[dict] = field(default_factory=list)
    processing_seconds: float = 0.0
    error: str | None = None


class VideoRAGPipeline:
    """Production pipeline for video RAG processing."""

    def __init__(self, config: VideoConfig):
        self.config = config

    async def process_video(self, video_path: str) -> ProcessedVideo:
        """Process a video through the full pipeline."""
        start = time.perf_counter()

        try:
            # Step 1: Analyze video
            metadata = self._get_metadata(video_path)

            # Step 2: Extract audio and keyframes in parallel
            audio_path, keyframes = await asyncio.gather(
                self._extract_audio(video_path),
                self._extract_keyframes(video_path),
            )

            # Step 3: Transcribe and process visuals in parallel
            transcript_task = self._transcribe(audio_path) if metadata.has_audio else asyncio.coroutine(lambda: [])()
            visual_task = self._process_visuals(keyframes)

            transcript_segments, processed_keyframes = await asyncio.gather(
                transcript_task,
                visual_task,
            )

            # Step 4: Create multimodal chunks
            text_chunks = self._create_text_chunks(
                transcript_segments, processed_keyframes, video_path,
            )

            visual_chunks = self._create_visual_chunks(
                processed_keyframes, video_path,
            )

            return ProcessedVideo(
                metadata=metadata,
                keyframes=processed_keyframes,
                transcript_segments=transcript_segments,
                text_chunks=text_chunks,
                visual_chunks=visual_chunks,
                processing_seconds=time.perf_counter() - start,
            )

        except Exception as e:
            logger.exception(f"Error processing {video_path}")
            return ProcessedVideo(
                metadata=VideoMetadata(
                    path=video_path, duration_seconds=0,
                    width=0, height=0, fps=0, audio_codec="", has_audio=False,
                ),
                error=str(e),
                processing_seconds=time.perf_counter() - start,
            )

    def _get_metadata(self, video_path: str) -> VideoMetadata:
        """Extract video metadata using ffprobe."""
        import json

        cmd = [
            "ffprobe", "-v", "quiet",
            "-print_format", "json",
            "-show_format", "-show_streams",
            video_path,
        ]

        result = subprocess.run(cmd, capture_output=True, text=True, check=True)
        data = json.loads(result.stdout)

        video_stream = next(
            (s for s in data.get("streams", []) if s["codec_type"] == "video"),
            {},
        )
        audio_stream = next(
            (s for s in data.get("streams", []) if s["codec_type"] == "audio"),
            None,
        )

        fps_parts = video_stream.get("r_frame_rate", "30/1").split("/")
        fps = float(fps_parts[0]) / float(fps_parts[1]) if len(fps_parts) == 2 else 30.0

        return VideoMetadata(
            path=video_path,
            duration_seconds=float(data.get("format", {}).get("duration", 0)),
            width=int(video_stream.get("width", 0)),
            height=int(video_stream.get("height", 0)),
            fps=fps,
            audio_codec=audio_stream.get("codec_name", "") if audio_stream else "",
            has_audio=audio_stream is not None,
        )

    async def _extract_audio(self, video_path: str) -> str:
        """Extract audio track as WAV."""
        output = f"{video_path}.audio.wav"

        cmd = [
            "ffmpeg", "-i", video_path,
            "-vn", "-acodec", "pcm_s16le",
            "-ar", "16000", "-ac", "1",
            output, "-y",
        ]

        loop = asyncio.get_event_loop()
        await loop.run_in_executor(
            None,
            lambda: subprocess.run(cmd, capture_output=True, check=True),
        )

        return output

    async def _extract_keyframes(self, video_path: str) -> list[dict]:
        """Extract keyframes using scene detection."""
        loop = asyncio.get_event_loop()
        output_dir = f"{video_path}.keyframes"

        def _extract():
            from scenedetect import open_video, SceneManager, ContentDetector
            from scenedetect.scene_manager import save_images

            Path(output_dir).mkdir(parents=True, exist_ok=True)

            video = open_video(video_path)
            manager = SceneManager()
            manager.add_detector(ContentDetector(
                threshold=self.config.scene_threshold,
                min_scene_len=self.config.min_scene_length,
            ))
            manager.detect_scenes(video)
            scenes = manager.get_scene_list()

            # Limit keyframes
            if len(scenes) > self.config.max_keyframes:
                step = len(scenes) // self.config.max_keyframes
                scenes = scenes[::step][:self.config.max_keyframes]

            save_images(scenes, video, num_images=1, output_dir=output_dir)

            keyframes = []
            for i, (s, e) in enumerate(scenes):
                img_path = Path(output_dir) / f"{i + 1:03d}-001.jpg"
                keyframes.append({
                    "scene_index": i,
                    "start_time": s.get_seconds(),
                    "end_time": e.get_seconds(),
                    "image_path": str(img_path),
                })

            return keyframes

        return await loop.run_in_executor(None, _extract)

    async def _transcribe(self, audio_path: str) -> list[dict]:
        """Transcribe audio with timestamps."""
        loop = asyncio.get_event_loop()

        def _do_transcribe():
            from faster_whisper import WhisperModel

            model = WhisperModel(
                self.config.whisper_model,
                device=self.config.whisper_device,
                compute_type="float16",
            )

            segments, info = model.transcribe(
                audio_path,
                beam_size=5,
                word_timestamps=True,
                vad_filter=True,
            )

            result = []
            for seg in segments:
                result.append({
                    "start": round(seg.start, 2),
                    "end": round(seg.end, 2),
                    "text": seg.text.strip(),
                })

            return result

        return await loop.run_in_executor(None, _do_transcribe)

    async def _process_visuals(self, keyframes: list[dict]) -> list[dict]:
        """OCR and CLIP encode keyframes."""
        loop = asyncio.get_event_loop()

        def _process():
            processed = []
            for kf in keyframes:
                img_path = kf.get("image_path", "")
                if not img_path or not Path(img_path).exists():
                    processed.append(kf)
                    continue

                entry = {**kf}

                # OCR
                if self.config.ocr_keyframes:
                    try:
                        from PIL import Image
                        import pytesseract
                        image = Image.open(img_path)
                        text = pytesseract.image_to_string(image).strip()
                        entry["onscreen_text"] = text
                    except Exception:
                        entry["onscreen_text"] = ""

                processed.append(entry)

            return processed

        return await loop.run_in_executor(None, _process)

    def _create_text_chunks(
        self,
        segments: list[dict],
        keyframes: list[dict],
        video_path: str,
    ) -> list[dict]:
        """Create time-window text chunks."""
        chunks = create_multimodal_chunks(
            segments, keyframes, video_path,
            self.config.chunk_duration_seconds,
        )

        return [
            {
                "text": c.text,
                "metadata": c.metadata,
                "embedding_type": "text",
            }
            for c in chunks
            if len(c.text) > self.config.min_chunk_chars
        ]

    def _create_visual_chunks(
        self,
        keyframes: list[dict],
        video_path: str,
    ) -> list[dict]:
        """Create visual chunks with CLIP embedding metadata."""
        return [
            {
                "image_path": kf["image_path"],
                "metadata": {
                    "source": video_path,
                    "start_time": kf.get("start_time", kf.get("timestamp", 0)),
                    "onscreen_text": kf.get("onscreen_text", ""),
                    "chunk_type": "video_visual",
                },
                "embedding_type": "visual",
            }
            for kf in keyframes
            if kf.get("image_path") and Path(kf["image_path"]).exists()
        ]
```

---

## Running the Pipeline

```python
import asyncio
import json
from pathlib import Path


async def main():
    config = VideoConfig(
        whisper_model="large-v3",
        diarize=True,
        ocr_keyframes=True,
        chunk_duration_seconds=60,
    )

    pipeline = VideoRAGPipeline(config)

    result = await pipeline.process_video("/data/videos/meeting_2025-01-15.mp4")

    if result.error:
        print(f"Error: {result.error}")
        return

    print(f"Duration: {result.metadata.duration_seconds:.0f}s")
    print(f"Keyframes: {len(result.keyframes)}")
    print(f"Transcript segments: {len(result.transcript_segments)}")
    print(f"Text chunks: {len(result.text_chunks)}")
    print(f"Visual chunks: {len(result.visual_chunks)}")
    print(f"Processing time: {result.processing_seconds:.1f}s")

    # Save output
    output = {
        "text_chunks": result.text_chunks,
        "visual_chunks": [
            {k: v for k, v in vc.items() if k != "image_path"}
            for vc in result.visual_chunks
        ],
    }
    Path("/data/output/video_chunks.json").write_text(
        json.dumps(output, indent=2, default=str),
    )


if __name__ == "__main__":
    asyncio.run(main())
```

---

## Common Pitfalls

1. **Processing audio and video sequentially.** Audio transcription and keyframe extraction are independent. Run them in parallel.
2. **Not extracting audio as separate WAV.** Whisper processes audio files. Extract the audio track before transcription.
3. **Forgetting to limit keyframe count.** Scene detection on a 4-hour video may produce thousands of keyframes. Set a max limit.
4. **Not cleaning up temporary files.** Extracted audio and keyframe images can consume gigabytes. Delete after processing.
5. **Ignoring ffprobe for metadata.** Duration, resolution, and codec information helps tune processing parameters.
6. **Not aligning chunk time windows with transcript segment boundaries.** Time windows should snap to sentence boundaries when possible.

---

## Production Checklist

- [ ] Video metadata extracted (duration, resolution, audio presence)
- [ ] Audio extracted as 16kHz mono WAV
- [ ] Keyframes extracted at scene boundaries with configurable limits
- [ ] Audio and visual processing run in parallel
- [ ] Transcript includes word-level timestamps
- [ ] Speaker diarization labels utterances
- [ ] On-screen text OCR'd from keyframes
- [ ] CLIP embeddings computed for visual search
- [ ] Time-window chunks combine transcript and on-screen text
- [ ] Temporary files cleaned up after processing
- [ ] Output includes both text and visual chunk indexes

---

## References

- PySceneDetect -- https://www.scenedetect.com/docs/latest/
- ffmpeg / ffprobe -- https://ffmpeg.org/documentation.html
- faster-whisper -- https://github.com/SYSTRAN/faster-whisper
- CLIP -- https://github.com/openai/CLIP
- pyannote.audio -- https://github.com/pyannote/pyannote-audio
