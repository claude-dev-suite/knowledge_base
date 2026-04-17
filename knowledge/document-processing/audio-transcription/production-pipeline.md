# Audio Transcription -- Production Pipeline

## Overview / TL;DR

A production audio transcription pipeline must handle diverse audio formats, preprocess for quality, transcribe with the optimal engine, add speaker labels, produce time-stamped chunks for RAG, and support batch processing with error recovery. This guide provides a complete async pipeline from raw audio files to speaker-aware, time-coded chunks ready for embedding and retrieval.

---

## Architecture

```
Audio Files (mp3, wav, m4a, ogg, flac, webm)
    |
    v
[1] Audio Preprocessing
    |-- format conversion (to 16kHz mono WAV)
    |-- volume normalization
    |-- noise reduction (optional)
    |-- silence splitting for long files
    |
    v
[2] Transcription
    |-- engine selection (local/cloud)
    |-- word-level timestamps
    |-- language detection
    |
    v
[3] Speaker Diarization
    |-- pyannote (local) or API (cloud)
    |-- merge with transcript
    |
    v
[4] Post-Processing
    |-- punctuation correction
    |-- paragraph segmentation
    |-- speaker label normalization
    |
    v
[5] Chunking
    |-- speaker-aware chunks
    |-- time-range metadata
    |-- topic-based boundaries
    |
    v
[6] Output (JSON, vector store)
```

---

## Pipeline Implementation

```python
import asyncio
import logging
import time
from dataclasses import dataclass, field
from pathlib import Path
from concurrent.futures import ProcessPoolExecutor

logger = logging.getLogger(__name__)


@dataclass
class TranscriptionConfig:
    engine: str = "faster-whisper"       # "faster-whisper", "assemblyai", "deepgram"
    model_size: str = "large-v3"
    device: str = "cuda"
    compute_type: str = "float16"
    diarize: bool = True
    diarization_method: str = "pyannote"  # "pyannote", "api"
    language: str | None = None
    max_audio_duration_minutes: int = 60
    target_sample_rate: int = 16000
    normalize_volume: bool = True
    vad_filter: bool = True
    max_workers: int = 2
    # Cloud API keys
    assemblyai_key: str = ""
    deepgram_key: str = ""
    hf_token: str = ""


@dataclass
class TranscriptionResult:
    file_path: str
    text: str
    segments: list[dict] = field(default_factory=list)
    utterances: list[dict] = field(default_factory=list)
    chunks: list[dict] = field(default_factory=list)
    language: str = ""
    duration_seconds: float = 0.0
    processing_seconds: float = 0.0
    engine_used: str = ""
    error: str | None = None
    warnings: list[str] = field(default_factory=list)


class AudioTranscriptionPipeline:
    """Production pipeline for audio transcription and chunking."""

    def __init__(self, config: TranscriptionConfig):
        self.config = config

    async def process_file(self, audio_path: str) -> TranscriptionResult:
        """Process a single audio file through the full pipeline."""
        start = time.perf_counter()

        try:
            # Step 1: Preprocess
            preprocessed = await self._preprocess(audio_path)

            # Step 2: Transcribe
            transcript = await self._transcribe(preprocessed)
            if transcript.error:
                return transcript

            # Step 3: Diarize
            if self.config.diarize:
                transcript = await self._diarize(preprocessed, transcript)

            # Step 4: Post-process
            transcript = self._post_process(transcript)

            # Step 5: Chunk
            transcript.chunks = self._chunk(transcript)

            transcript.processing_seconds = time.perf_counter() - start
            return transcript

        except Exception as e:
            logger.exception(f"Error processing {audio_path}")
            return TranscriptionResult(
                file_path=audio_path,
                text="",
                error=str(e),
                processing_seconds=time.perf_counter() - start,
            )

    async def _preprocess(self, audio_path: str) -> str:
        """Preprocess audio file."""
        from pydub import AudioSegment

        loop = asyncio.get_event_loop()

        def _do_preprocess():
            audio = AudioSegment.from_file(audio_path)

            # Convert to mono
            if audio.channels > 1:
                audio = audio.set_channels(1)

            # Set sample rate
            audio = audio.set_frame_rate(self.config.target_sample_rate)

            # Normalize
            if self.config.normalize_volume:
                target_dbfs = -20
                change = target_dbfs - audio.dBFS
                audio = audio.apply_gain(change)

            # Export
            output = f"{audio_path}.preprocessed.wav"
            audio.export(output, format="wav")
            return output

        return await loop.run_in_executor(None, _do_preprocess)

    async def _transcribe(self, audio_path: str) -> TranscriptionResult:
        """Run transcription with configured engine."""
        loop = asyncio.get_event_loop()

        if self.config.engine == "faster-whisper":
            return await loop.run_in_executor(
                None, self._transcribe_faster_whisper, audio_path,
            )
        elif self.config.engine == "assemblyai":
            return await loop.run_in_executor(
                None, self._transcribe_assemblyai, audio_path,
            )
        elif self.config.engine == "deepgram":
            return await loop.run_in_executor(
                None, self._transcribe_deepgram, audio_path,
            )
        else:
            return TranscriptionResult(
                file_path=audio_path, text="",
                error=f"Unknown engine: {self.config.engine}",
            )

    def _transcribe_faster_whisper(self, audio_path: str) -> TranscriptionResult:
        """Transcribe with faster-whisper."""
        from faster_whisper import WhisperModel

        model = WhisperModel(
            self.config.model_size,
            device=self.config.device,
            compute_type=self.config.compute_type,
        )

        segments_iter, info = model.transcribe(
            audio_path,
            beam_size=5,
            language=self.config.language,
            word_timestamps=True,
            vad_filter=self.config.vad_filter,
        )

        segments = []
        full_text = []
        for seg in segments_iter:
            segments.append({
                "start": round(seg.start, 2),
                "end": round(seg.end, 2),
                "text": seg.text.strip(),
                "words": [
                    {"word": w.word, "start": round(w.start, 2),
                     "end": round(w.end, 2), "probability": round(w.probability, 3)}
                    for w in (seg.words or [])
                ],
            })
            full_text.append(seg.text.strip())

        return TranscriptionResult(
            file_path=audio_path,
            text=" ".join(full_text),
            segments=segments,
            language=info.language,
            duration_seconds=round(info.duration, 2),
            engine_used="faster-whisper",
        )

    def _transcribe_assemblyai(self, audio_path: str) -> TranscriptionResult:
        """Transcribe with AssemblyAI."""
        import assemblyai as aai
        aai.settings.api_key = self.config.assemblyai_key

        config = aai.TranscriptionConfig(
            speaker_labels=self.config.diarize,
            language_detection=True,
        )

        transcript = aai.Transcriber().transcribe(audio_path, config=config)

        if transcript.status == aai.TranscriptStatus.error:
            return TranscriptionResult(
                file_path=audio_path, text="",
                error=transcript.error, engine_used="assemblyai",
            )

        utterances = []
        if transcript.utterances:
            for u in transcript.utterances:
                utterances.append({
                    "speaker": u.speaker,
                    "text": u.text,
                    "start": u.start / 1000.0,
                    "end": u.end / 1000.0,
                })

        return TranscriptionResult(
            file_path=audio_path,
            text=transcript.text,
            utterances=utterances,
            language=transcript.language_code or "",
            duration_seconds=transcript.audio_duration or 0,
            engine_used="assemblyai",
        )

    def _transcribe_deepgram(self, audio_path: str) -> TranscriptionResult:
        """Transcribe with Deepgram."""
        from deepgram import DeepgramClient, PrerecordedOptions

        client = DeepgramClient(self.config.deepgram_key)

        with open(audio_path, "rb") as f:
            audio_data = f.read()

        options = PrerecordedOptions(
            model="nova-2",
            smart_format=True,
            diarize=self.config.diarize,
            utterances=True,
        )

        response = client.listen.rest.v("1").transcribe_file(
            {"buffer": audio_data}, options,
        )
        result = response.to_dict()
        channel = result["results"]["channels"][0]
        alt = channel["alternatives"][0]

        utterances = []
        if "paragraphs" in alt:
            for para in alt["paragraphs"]["paragraphs"]:
                for sent in para.get("sentences", []):
                    utterances.append({
                        "speaker": f"SPEAKER_{para.get('speaker', 0)}",
                        "text": sent["text"],
                        "start": sent["start"],
                        "end": sent["end"],
                    })

        return TranscriptionResult(
            file_path=audio_path,
            text=alt["transcript"],
            utterances=utterances,
            duration_seconds=result["metadata"]["duration"],
            engine_used="deepgram",
        )

    async def _diarize(
        self,
        audio_path: str,
        transcript: TranscriptionResult,
    ) -> TranscriptionResult:
        """Add speaker diarization to transcript."""
        if transcript.utterances:
            return transcript  # Already has speaker labels from API

        if not transcript.segments:
            return transcript

        loop = asyncio.get_event_loop()

        try:
            diarization = await loop.run_in_executor(
                None, self._run_pyannote, audio_path,
            )

            merged = []
            for seg in transcript.segments:
                mid = (seg["start"] + seg["end"]) / 2
                speaker = "UNKNOWN"
                for d in diarization:
                    if d["start"] <= mid <= d["end"]:
                        speaker = d["speaker"]
                        break

                merged.append({
                    "speaker": speaker,
                    "text": seg["text"],
                    "start": seg["start"],
                    "end": seg["end"],
                })

            transcript.utterances = merged

        except Exception as e:
            transcript.warnings.append(f"Diarization failed: {e}")

        return transcript

    def _run_pyannote(self, audio_path: str) -> list[dict]:
        """Run pyannote diarization."""
        from pyannote.audio import Pipeline

        pipeline = Pipeline.from_pretrained(
            "pyannote/speaker-diarization-3.1",
            use_auth_token=self.config.hf_token,
        )
        diarization = pipeline(audio_path)

        return [
            {"speaker": speaker, "start": round(turn.start, 2), "end": round(turn.end, 2)}
            for turn, _, speaker in diarization.itertracks(yield_label=True)
        ]

    def _post_process(self, transcript: TranscriptionResult) -> TranscriptionResult:
        """Clean and normalize transcript."""
        import re

        text = transcript.text
        # Fix common transcription artifacts
        text = re.sub(r'\s+', ' ', text)
        text = text.strip()
        transcript.text = text

        # Normalize speaker labels
        if transcript.utterances:
            speaker_map = {}
            counter = 1
            for utt in transcript.utterances:
                orig = utt["speaker"]
                if orig not in speaker_map:
                    speaker_map[orig] = f"Speaker {counter}"
                    counter += 1
                utt["speaker"] = speaker_map[orig]

        return transcript

    def _chunk(self, transcript: TranscriptionResult) -> list[dict]:
        """Create speaker-aware chunks with time metadata."""
        source = transcript.file_path

        if transcript.utterances:
            chunks = chunk_transcript_by_speaker(
                transcript.utterances,
                source=source,
                max_chunk_chars=1500,
            )
            return [{"text": c.text, "metadata": c.metadata} for c in chunks]

        # Fallback: chunk by segments
        from langchain_text_splitters import RecursiveCharacterTextSplitter
        splitter = RecursiveCharacterTextSplitter(
            chunk_size=1500, chunk_overlap=200,
        )
        texts = splitter.split_text(transcript.text)
        return [
            {
                "text": t,
                "metadata": {
                    "source": source,
                    "duration": transcript.duration_seconds,
                    "chunk_type": "transcript",
                },
            }
            for t in texts
        ]

    async def process_batch(
        self,
        audio_paths: list[str],
        progress_callback=None,
    ) -> list[TranscriptionResult]:
        """Process multiple audio files."""
        results = []
        for i, path in enumerate(audio_paths):
            result = await self.process_file(path)
            results.append(result)
            if progress_callback:
                progress_callback(i + 1, len(audio_paths), path)
        return results
```

---

## Running the Pipeline

```python
import asyncio


async def main():
    config = TranscriptionConfig(
        engine="faster-whisper",
        model_size="large-v3",
        diarize=True,
        diarization_method="pyannote",
        hf_token="hf_...",
    )

    pipeline = AudioTranscriptionPipeline(config)

    audio_files = [
        "/data/meetings/standup_2025-01-15.mp3",
        "/data/meetings/planning_2025-01-16.m4a",
    ]

    results = await pipeline.process_batch(audio_files)

    for result in results:
        if result.error:
            print(f"FAILED: {result.file_path} - {result.error}")
        else:
            print(
                f"OK: {result.file_path} - {result.duration_seconds:.0f}s, "
                f"{len(result.chunks)} chunks, {result.engine_used}"
            )


if __name__ == "__main__":
    asyncio.run(main())
```

---

## Common Pitfalls

1. **Not preprocessing audio format.** Whisper expects 16kHz WAV. Other formats add conversion overhead and may cause errors.
2. **Processing 3-hour recordings as one file.** Split long recordings into segments. Memory usage scales with duration.
3. **Skipping diarization for meetings.** Without speaker labels, meeting chunks are ambiguous and less useful for retrieval.
4. **Not normalizing speaker labels.** Raw diarization outputs labels like "SPEAKER_00". Normalize to "Speaker 1" for readability.
5. **Ignoring VAD for silence filtering.** Whisper hallucinates on silent segments. Always enable VAD.
6. **Not cleaning up temporary files.** Preprocessed WAV files can be large. Delete them after processing.

---

## Production Checklist

- [ ] Audio preprocessing converts to 16kHz mono WAV with volume normalization
- [ ] Long files are split before processing
- [ ] VAD filtering removes silence segments
- [ ] Transcription engine is matched to requirements (accuracy vs cost vs speed)
- [ ] Speaker diarization labels utterances
- [ ] Diarization is merged with transcript at word level
- [ ] Speaker labels are normalized and readable
- [ ] Chunks are speaker-aware with time-range metadata
- [ ] Batch processing handles multiple files with progress tracking
- [ ] Temporary files are cleaned up after processing

---

## References

- faster-whisper -- https://github.com/SYSTRAN/faster-whisper
- pyannote.audio -- https://github.com/pyannote/pyannote-audio
- AssemblyAI -- https://www.assemblyai.com/docs
- Deepgram -- https://developers.deepgram.com/docs
- pydub -- https://github.com/jiaaro/pydub
