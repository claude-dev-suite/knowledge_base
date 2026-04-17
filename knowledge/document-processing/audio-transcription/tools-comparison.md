# Audio Transcription -- Tools Comparison

## Overview / TL;DR

Choosing the right transcription engine depends on accuracy requirements, budget, latency needs, and whether speaker diarization is required. This guide provides accuracy/speed/cost comparisons across Whisper (all sizes), faster-whisper, AssemblyAI, and Deepgram, with benchmark methodology and a decision framework.

---

## Accuracy/Speed/Cost Matrix

| Engine | Model | WER (English) | Speed (1hr audio) | Cost/Hour | Diarization | Local |
|--------|-------|---------------|-------------------|-----------|-------------|-------|
| Whisper | tiny | 7-12% | 2-4 min (GPU) | Free | No | Yes |
| Whisper | base | 5-9% | 3-6 min (GPU) | Free | No | Yes |
| Whisper | small | 4-7% | 5-10 min (GPU) | Free | No | Yes |
| Whisper | medium | 3-6% | 10-20 min (GPU) | Free | No | Yes |
| Whisper | large-v3 | 2-5% | 15-30 min (GPU) | Free | No | Yes |
| Whisper | turbo | 3-5% | 3-6 min (GPU) | Free | No | Yes |
| faster-whisper | large-v3 | 2-5% | 3-8 min (GPU) | Free | No | Yes |
| faster-whisper | large-v3 | 2-5% | 15-40 min (CPU) | Free | No | Yes |
| AssemblyAI | best | 2-4% | 2-5 min | $0.65/hr | Yes | No |
| AssemblyAI | nano | 4-7% | 1-2 min | $0.25/hr | Yes | No |
| Deepgram | nova-2 | 3-5% | 1-3 min | $0.25/hr | Yes | No |
| Deepgram | whisper | 3-6% | 2-4 min | $0.36/hr | Yes | No |
| Google STT | latest_long | 3-5% | 2-5 min | $0.016/15s | Yes | No |

---

## Accuracy by Content Type

| Content Type | Whisper large-v3 | faster-whisper | AssemblyAI | Deepgram |
|-------------|-----------------|---------------|------------|---------|
| Podcast (clean) | 97-99% | 97-99% | 98-99% | 97-99% |
| Meeting (multiple speakers) | 93-96% | 93-96% | 95-98% | 94-97% |
| Phone call (8kHz) | 88-93% | 88-93% | 92-96% | 91-95% |
| Lecture (echoing room) | 90-95% | 90-95% | 93-97% | 92-96% |
| Interview (noisy background) | 85-92% | 85-92% | 90-95% | 89-94% |
| Technical terminology | 90-95% | 90-95% | 93-97%* | 92-96%* |
| Non-native English speakers | 85-92% | 85-92% | 88-94% | 87-93% |
| Multilingual (switching) | 80-90% | 80-90% | 75-85% | 70-82% |

*AssemblyAI and Deepgram support custom vocabulary boosting for technical terms.

---

## Diarization Comparison

| Feature | pyannote 3.1 | AssemblyAI | Deepgram | WhisperX |
|---------|-------------|------------|---------|----------|
| Speaker accuracy | 90-95% (DER) | 92-97% (DER) | 90-95% (DER) | 85-92% (DER) |
| Max speakers | Unlimited | 10+ | 10+ | Unlimited |
| Speaker count detection | Auto | Auto | Auto or manual | Auto |
| Word-level speaker | Yes | Yes | Yes | Yes |
| Overlapping speech | Partial | Good | Partial | No |
| Real-time | No | Yes (streaming) | Yes (streaming) | No |
| Local/Cloud | Local | Cloud | Cloud | Local |
| Cost | Free (Hugging Face) | Included | Included | Free |

---

## Speed Benchmarks

```python
import time
from dataclasses import dataclass


@dataclass
class TranscriptionBenchmark:
    engine: str
    model: str
    audio_duration_seconds: float
    transcription_seconds: float
    real_time_factor: float   # < 1 means faster than real-time
    word_count: int


def benchmark_speed(audio_path: str) -> list[TranscriptionBenchmark]:
    """Benchmark transcription speed across engines."""
    from pydub import AudioSegment
    audio = AudioSegment.from_file(audio_path)
    duration = len(audio) / 1000.0

    results = []

    # faster-whisper (multiple sizes)
    try:
        from faster_whisper import WhisperModel
        for size in ["tiny", "base", "small", "large-v3"]:
            model = WhisperModel(size, device="cuda", compute_type="float16")
            start = time.perf_counter()
            segments, info = model.transcribe(audio_path, beam_size=5)
            text = " ".join(s.text for s in segments)
            elapsed = time.perf_counter() - start

            results.append(TranscriptionBenchmark(
                engine="faster-whisper",
                model=size,
                audio_duration_seconds=duration,
                transcription_seconds=round(elapsed, 2),
                real_time_factor=round(elapsed / duration, 3),
                word_count=len(text.split()),
            ))
    except ImportError:
        pass

    return results
```

---

## Cost Analysis

### Open-Source (Infrastructure Only)

| Engine + Model | GPU Instance | Cost per 100 Hours | Real-Time Factor |
|---------------|-------------|-------------------|-----------------|
| faster-whisper tiny | g4dn.xlarge | $2.63 | 0.05x |
| faster-whisper base | g4dn.xlarge | $4.38 | 0.08x |
| faster-whisper small | g4dn.xlarge | $8.77 | 0.17x |
| faster-whisper large-v3 | g4dn.xlarge | $21.93 | 0.42x |
| Whisper large-v3 | g4dn.xlarge | $78.90 | 1.5x |
| faster-whisper large-v3 | CPU (c5.2xlarge) | $58.30 | 3.4x |

### Cloud Services

| Service | Model | Per Hour | 100 Hours | Free Tier |
|---------|-------|----------|-----------|-----------|
| AssemblyAI | best | $0.65 | $65 | None |
| AssemblyAI | nano | $0.25 | $25 | None |
| Deepgram | nova-2 | $0.25 | $25 | $200 credit |
| Deepgram | whisper | $0.36 | $36 | $200 credit |
| Google STT | latest_long | ~$3.84 | ~$384 | 60 min/mo |
| AWS Transcribe | standard | $1.44 | $144 | 60 min/mo |

---

## Feature Comparison

| Feature | Whisper | faster-whisper | AssemblyAI | Deepgram |
|---------|---------|---------------|------------|---------|
| Word timestamps | Yes | Yes | Yes | Yes |
| Sentence timestamps | Manual | Manual | Yes | Yes |
| Speaker diarization | External | External | Built-in | Built-in |
| Punctuation | Yes | Yes | Yes | Yes |
| Custom vocabulary | No | No | Yes (word_boost) | Yes (keywords) |
| Profanity filter | No | No | Yes | Yes |
| Topic detection | No | No | Yes | No |
| Sentiment analysis | No | No | Yes | No |
| PII redaction | No | No | Yes | Yes |
| Streaming/real-time | No | No | Yes | Yes |
| Batch processing | Local | Local | API | API |
| Language detection | Yes | Yes | Yes | Yes |
| Translation | Yes (to English) | Yes (to English) | No | No |

---

## Decision Framework

```
What are your requirements?
  |
  +---> Data must stay local
  |     |
  |     +---> GPU available
  |     |     |
  |     |     +---> Need highest accuracy
  |     |     |     --> faster-whisper large-v3 + pyannote
  |     |     |
  |     |     +---> Need speed
  |     |           --> faster-whisper turbo + pyannote
  |     |
  |     +---> CPU only
  |           --> faster-whisper small (slower but works)
  |
  +---> Cloud OK
        |
        +---> Need speaker diarization
        |     |
        |     +---> Budget sensitive
        |     |     --> Deepgram nova-2 ($0.25/hr)
        |     |
        |     +---> Need best accuracy
        |           --> AssemblyAI best ($0.65/hr)
        |
        +---> Just need text, no diarization
        |     --> faster-whisper large-v3 locally (free)
        |
        +---> Need real-time/streaming
              --> Deepgram or AssemblyAI streaming API
```

---

## Common Pitfalls

1. **Comparing WER without controlling for content type.** A model with 3% WER on podcasts may have 15% WER on phone calls.
2. **Not accounting for diarization cost.** pyannote is free but needs GPU. Cloud APIs include diarization but charge per hour.
3. **Using Whisper large-v3 when turbo is sufficient.** The turbo model is 8x faster with minimal accuracy loss on clean audio.
4. **Ignoring the real-time factor.** A model that processes 1 hour of audio in 2 hours is impractical for batch workloads.
5. **Not testing with your actual audio.** Background noise, accents, and technical vocabulary vary enormously. Always benchmark on representative samples.
6. **Choosing the cheapest cloud API without testing accuracy.** Cost per hour means nothing if the transcription quality requires manual correction.

---

## References

- OpenAI Whisper -- https://github.com/openai/whisper
- faster-whisper -- https://github.com/SYSTRAN/faster-whisper
- AssemblyAI -- https://www.assemblyai.com/docs
- Deepgram -- https://developers.deepgram.com/docs
- pyannote.audio -- https://github.com/pyannote/pyannote-audio
