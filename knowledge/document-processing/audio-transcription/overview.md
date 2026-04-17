# Audio Transcription -- Comprehensive Guide

## Overview / TL;DR

Audio transcription converts spoken language into text, enabling RAG systems to index podcasts, meetings, lectures, call center recordings, and any other audio content. Modern transcription engines range from open-source models (Whisper, faster-whisper) to commercial APIs (AssemblyAI, Deepgram) that offer speaker diarization, word-level timestamps, and real-time streaming. This guide covers every major engine, explains preprocessing techniques, diarization approaches, and provides production-ready Python code for building audio-to-searchable-chunks pipelines.

---

## Why Audio Transcription for RAG

1. **Massive untapped content.** Organizations record meetings, calls, and presentations daily. This content is inaccessible to search and LLMs without transcription.
2. **Speaker-aware retrieval.** Knowing who said what enables queries like "What did the CTO say about the Q3 roadmap?"
3. **Temporal context.** Timestamps allow linking retrieved chunks back to specific moments in the recording.
4. **Multimodal RAG.** Combined with video keyframes, transcripts enable unified audio-visual search.

---

## Engine Landscape

### OpenAI Whisper (Open Source)

The foundational model for modern speech recognition. Available in multiple sizes (tiny, base, small, medium, large-v3, turbo).

```python
import whisper


def transcribe_whisper(
    audio_path: str,
    model_size: str = "large-v3",
    language: str | None = None,
) -> dict:
    """Transcribe audio using OpenAI Whisper."""
    model = whisper.load_model(model_size)

    result = model.transcribe(
        audio_path,
        language=language,
        task="transcribe",
        verbose=False,
        word_timestamps=True,
        condition_on_previous_text=True,
    )

    segments = []
    for seg in result["segments"]:
        segments.append({
            "start": round(seg["start"], 2),
            "end": round(seg["end"], 2),
            "text": seg["text"].strip(),
            "words": seg.get("words", []),
        })

    return {
        "text": result["text"],
        "language": result.get("language", ""),
        "segments": segments,
    }
```

### faster-whisper (CTranslate2)

Optimized Whisper implementation using CTranslate2 for 4x faster inference with lower memory usage.

```python
from faster_whisper import WhisperModel


def transcribe_faster_whisper(
    audio_path: str,
    model_size: str = "large-v3",
    device: str = "cuda",
    compute_type: str = "float16",
) -> dict:
    """Transcribe using faster-whisper (CTranslate2 backend)."""
    model = WhisperModel(
        model_size,
        device=device,
        compute_type=compute_type,
    )

    segments_iter, info = model.transcribe(
        audio_path,
        beam_size=5,
        language=None,           # Auto-detect
        word_timestamps=True,
        vad_filter=True,         # Filter silence
        vad_parameters={
            "min_silence_duration_ms": 500,
            "speech_pad_ms": 200,
        },
    )

    segments = []
    full_text = []

    for segment in segments_iter:
        seg_data = {
            "start": round(segment.start, 2),
            "end": round(segment.end, 2),
            "text": segment.text.strip(),
        }

        if segment.words:
            seg_data["words"] = [
                {
                    "word": w.word,
                    "start": round(w.start, 2),
                    "end": round(w.end, 2),
                    "probability": round(w.probability, 3),
                }
                for w in segment.words
            ]

        segments.append(seg_data)
        full_text.append(segment.text.strip())

    return {
        "text": " ".join(full_text),
        "language": info.language,
        "language_probability": round(info.language_probability, 3),
        "duration": round(info.duration, 2),
        "segments": segments,
    }
```

### AssemblyAI

Cloud API with built-in speaker diarization, sentiment analysis, and content moderation.

```python
import assemblyai as aai


def transcribe_assemblyai(
    audio_path: str,
    api_key: str,
    speaker_labels: bool = True,
) -> dict:
    """Transcribe with AssemblyAI including speaker diarization."""
    aai.settings.api_key = api_key

    config = aai.TranscriptionConfig(
        speaker_labels=speaker_labels,
        language_detection=True,
        punctuate=True,
        format_text=True,
        word_boost=["technical", "terms", "here"],
    )

    transcriber = aai.Transcriber()
    transcript = transcriber.transcribe(audio_path, config=config)

    if transcript.status == aai.TranscriptStatus.error:
        return {"error": transcript.error}

    # Extract utterances (speaker-labeled segments)
    utterances = []
    if transcript.utterances:
        for utt in transcript.utterances:
            utterances.append({
                "speaker": utt.speaker,
                "text": utt.text,
                "start": utt.start / 1000.0,
                "end": utt.end / 1000.0,
                "confidence": utt.confidence,
            })

    # Extract words with timestamps
    words = []
    if transcript.words:
        for w in transcript.words:
            words.append({
                "text": w.text,
                "start": w.start / 1000.0,
                "end": w.end / 1000.0,
                "confidence": w.confidence,
                "speaker": w.speaker,
            })

    return {
        "text": transcript.text,
        "utterances": utterances,
        "words": words,
        "audio_duration": transcript.audio_duration,
        "language": transcript.language_code,
    }
```

### Deepgram

Cloud API optimized for speed, offering real-time streaming and pre-recorded transcription.

```python
from deepgram import DeepgramClient, PrerecordedOptions


def transcribe_deepgram(
    audio_path: str,
    api_key: str,
    model: str = "nova-2",
    diarize: bool = True,
) -> dict:
    """Transcribe with Deepgram."""
    client = DeepgramClient(api_key)

    with open(audio_path, "rb") as f:
        audio_data = f.read()

    options = PrerecordedOptions(
        model=model,
        smart_format=True,
        diarize=diarize,
        punctuate=True,
        paragraphs=True,
        utterances=True,
        language="en",
    )

    response = client.listen.rest.v("1").transcribe_file(
        {"buffer": audio_data}, options
    )

    result = response.to_dict()
    channel = result["results"]["channels"][0]
    alternative = channel["alternatives"][0]

    # Extract paragraphs
    paragraphs = []
    if "paragraphs" in alternative:
        for para in alternative["paragraphs"]["paragraphs"]:
            for sentence in para.get("sentences", []):
                paragraphs.append({
                    "text": sentence["text"],
                    "start": sentence["start"],
                    "end": sentence["end"],
                    "speaker": para.get("speaker"),
                })

    return {
        "text": alternative["transcript"],
        "paragraphs": paragraphs,
        "confidence": alternative.get("confidence", 0),
        "duration": result["metadata"]["duration"],
    }
```

---

## Quick Comparison

| Feature | Whisper | faster-whisper | AssemblyAI | Deepgram |
|---------|---------|---------------|------------|---------|
| Speed (1hr audio) | 15-30 min | 3-8 min | 2-5 min | 1-3 min |
| Accuracy (English) | 95-98% | 95-98% | 96-99% | 95-98% |
| Speaker diarization | No (external) | No (external) | Built-in | Built-in |
| Word timestamps | Yes | Yes | Yes | Yes |
| Streaming | No | No | Yes | Yes |
| Languages | 100+ | 100+ | ~30 | ~30 |
| Local/Cloud | Local | Local | Cloud | Cloud |
| GPU required | Recommended | Recommended | N/A | N/A |
| Cost | Free | Free | $0.25-0.65/hr | $0.25-0.75/hr |

---

## Audio Preprocessing

```python
from pydub import AudioSegment
import subprocess


def preprocess_audio(
    input_path: str,
    output_path: str,
    target_sample_rate: int = 16000,
    mono: bool = True,
    normalize: bool = True,
) -> str:
    """Preprocess audio for optimal transcription quality."""
    audio = AudioSegment.from_file(input_path)

    # Convert to mono
    if mono and audio.channels > 1:
        audio = audio.set_channels(1)

    # Set sample rate
    audio = audio.set_frame_rate(target_sample_rate)

    # Normalize volume
    if normalize:
        target_dBFS = -20
        change_in_dBFS = target_dBFS - audio.dBFS
        audio = audio.apply_gain(change_in_dBFS)

    # Export as WAV (best for Whisper)
    audio.export(output_path, format="wav")
    return output_path


def split_long_audio(
    input_path: str,
    max_duration_minutes: int = 30,
) -> list[str]:
    """Split long audio files into chunks for processing."""
    audio = AudioSegment.from_file(input_path)
    duration_ms = len(audio)
    chunk_ms = max_duration_minutes * 60 * 1000

    if duration_ms <= chunk_ms:
        return [input_path]

    chunks = []
    for i in range(0, duration_ms, chunk_ms):
        chunk = audio[i:i + chunk_ms]
        chunk_path = f"{input_path}.chunk_{i // chunk_ms}.wav"
        chunk.export(chunk_path, format="wav")
        chunks.append(chunk_path)

    return chunks
```

---

## Speaker Diarization

```python
def diarize_with_pyannote(
    audio_path: str,
    hf_token: str,
    num_speakers: int | None = None,
) -> list[dict]:
    """Speaker diarization using pyannote.audio."""
    from pyannote.audio import Pipeline

    pipeline = Pipeline.from_pretrained(
        "pyannote/speaker-diarization-3.1",
        use_auth_token=hf_token,
    )

    kwargs = {}
    if num_speakers:
        kwargs["num_speakers"] = num_speakers

    diarization = pipeline(audio_path, **kwargs)

    segments = []
    for turn, _, speaker in diarization.itertracks(yield_label=True):
        segments.append({
            "speaker": speaker,
            "start": round(turn.start, 2),
            "end": round(turn.end, 2),
        })

    return segments


def merge_transcript_with_diarization(
    transcript_segments: list[dict],
    diarization_segments: list[dict],
) -> list[dict]:
    """Merge transcription segments with speaker labels."""
    merged = []

    for t_seg in transcript_segments:
        t_mid = (t_seg["start"] + t_seg["end"]) / 2

        # Find the diarization segment that overlaps the most
        best_speaker = "UNKNOWN"
        best_overlap = 0

        for d_seg in diarization_segments:
            overlap_start = max(t_seg["start"], d_seg["start"])
            overlap_end = min(t_seg["end"], d_seg["end"])
            overlap = max(0, overlap_end - overlap_start)

            if overlap > best_overlap:
                best_overlap = overlap
                best_speaker = d_seg["speaker"]

        merged.append({
            "speaker": best_speaker,
            "text": t_seg["text"],
            "start": t_seg["start"],
            "end": t_seg["end"],
        })

    return merged
```

---

## Speaker-Aware Chunking for RAG

```python
from dataclasses import dataclass


@dataclass
class TranscriptChunk:
    text: str
    metadata: dict


def chunk_transcript_by_speaker(
    utterances: list[dict],
    source: str,
    max_chunk_chars: int = 1500,
    merge_same_speaker: bool = True,
) -> list[TranscriptChunk]:
    """Chunk transcript into speaker-aware segments."""
    if merge_same_speaker:
        utterances = _merge_consecutive_speaker(utterances)

    chunks = []
    current_text = []
    current_chars = 0
    current_speaker = None
    chunk_start = 0

    for utt in utterances:
        utt_text = f"[{utt['speaker']}]: {utt['text']}"

        # Start new chunk on speaker change or size limit
        if (
            (current_speaker and utt["speaker"] != current_speaker)
            or (current_chars + len(utt_text) > max_chunk_chars and current_text)
        ):
            chunks.append(TranscriptChunk(
                text="\n".join(current_text),
                metadata={
                    "source": source,
                    "start_time": chunk_start,
                    "end_time": utt["start"],
                    "speakers": list(set(
                        line.split("]")[0][1:] for line in current_text
                    )),
                    "chunk_type": "transcript",
                },
            ))
            current_text = []
            current_chars = 0
            chunk_start = utt["start"]

        current_text.append(utt_text)
        current_chars += len(utt_text)
        current_speaker = utt["speaker"]

    # Last chunk
    if current_text:
        chunks.append(TranscriptChunk(
            text="\n".join(current_text),
            metadata={
                "source": source,
                "start_time": chunk_start,
                "end_time": utterances[-1]["end"] if utterances else 0,
                "speakers": list(set(
                    line.split("]")[0][1:] for line in current_text
                )),
                "chunk_type": "transcript",
            },
        ))

    return chunks


def _merge_consecutive_speaker(utterances: list[dict]) -> list[dict]:
    """Merge consecutive utterances from the same speaker."""
    if not utterances:
        return []

    merged = [utterances[0].copy()]
    for utt in utterances[1:]:
        if utt["speaker"] == merged[-1]["speaker"]:
            merged[-1]["text"] += " " + utt["text"]
            merged[-1]["end"] = utt["end"]
        else:
            merged.append(utt.copy())

    return merged
```

---

## Common Pitfalls

1. **Not preprocessing audio.** Sample rate mismatches, stereo channels, and volume normalization affect accuracy. Always convert to 16kHz mono WAV.
2. **Processing very long files as a single unit.** Whisper's context window is 30 seconds. Long files should be split or use the chunked processing pipeline.
3. **Ignoring speaker diarization.** Without speaker labels, meeting transcripts are ambiguous. "We should do X" means different things from different speakers.
4. **Using Whisper large-v3 for everything.** The turbo model is 8x faster with only 1-2% accuracy loss. Use large-v3 only for critical content.
5. **Not filtering silence with VAD.** Voice Activity Detection removes silence segments, reducing processing time and preventing hallucinations on silent sections.
6. **Hallucination on silence.** Whisper generates plausible-sounding text for silent audio segments. Always enable VAD filtering.
7. **Ignoring word-level timestamps.** Word timestamps enable precise time-coded retrieval. Always request them.
8. **Not handling multiple languages.** Multilingual meetings need language detection per segment.

---

## Production Checklist

- [ ] Audio is preprocessed (16kHz, mono, normalized volume, WAV format)
- [ ] Long files are split into processable chunks
- [ ] VAD filtering removes silence before transcription
- [ ] Word-level timestamps are captured
- [ ] Speaker diarization labels who said what
- [ ] Transcript is merged with diarization output
- [ ] Chunks are speaker-aware with time ranges in metadata
- [ ] Confidence scores flag low-quality segments for review
- [ ] Language detection handles multilingual content
- [ ] Error handling recovers from corrupted audio segments

---

## References

- OpenAI Whisper -- https://github.com/openai/whisper
- faster-whisper -- https://github.com/SYSTRAN/faster-whisper
- AssemblyAI documentation -- https://www.assemblyai.com/docs
- Deepgram documentation -- https://developers.deepgram.com/docs
- pyannote.audio -- https://github.com/pyannote/pyannote-audio
- pydub -- https://github.com/jiaaro/pydub
