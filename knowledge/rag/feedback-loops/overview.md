# Feedback Loops -- User Signals to RAG Improvement

## TL;DR

RAG systems improve through feedback loops that capture user signals (thumbs up/down, click-through rates, dwell time, explicit corrections) and use them to improve retrieval quality, reranking models, and embedding representations. Without feedback loops, RAG systems stagnate: the same bad retrievals keep producing the same bad answers. This overview covers the feedback loop architecture, signal types and their reliability, collection strategies using Langfuse and LangSmith, and how feedback data feeds back into the system through embedding fine-tuning, reranker adaptation, and prompt optimization.

---

## The Feedback Loop Architecture

```
User Query -> Retrieve -> Generate -> Present Answer
                                          |
                                    [User Signal]
                                          |
                              +-----------+-----------+
                              |           |           |
                         [Thumbs]    [Click]     [Dwell]
                              |           |           |
                              v           v           v
                         [Feedback Store (Langfuse/LangSmith)]
                              |
                    +---------+---------+
                    |         |         |
               [Retrain]  [Tune]   [Update]
               Embeddings  Reranker  Prompts
                    |         |         |
                    v         v         v
              [Improved RAG Pipeline]
```

## Signal Types and Reliability

```python
from dataclasses import dataclass
from enum import Enum


class SignalType(Enum):
    EXPLICIT_POSITIVE = "thumbs_up"
    EXPLICIT_NEGATIVE = "thumbs_down"
    CORRECTION = "user_correction"
    CLICK_THROUGH = "ctr"
    DWELL_TIME = "dwell_time"
    COPY_ACTION = "copy"
    REGENERATE = "regenerate"
    FOLLOW_UP = "follow_up_question"
    ESCALATION = "escalation"


@dataclass
class FeedbackSignal:
    query: str
    response: str
    retrieved_doc_ids: list[str]
    signal_type: SignalType
    signal_value: float  # -1.0 to 1.0
    user_id: str = ""
    session_id: str = ""
    timestamp: float = 0.0
    metadata: dict | None = None


SIGNAL_RELIABILITY = {
    SignalType.EXPLICIT_POSITIVE: {
        "reliability": 0.85,
        "description": "User clicked thumbs up",
        "bias": "Users rarely rate good answers, selection bias toward memorable responses",
        "volume": "low (5-15% of interactions)",
    },
    SignalType.EXPLICIT_NEGATIVE: {
        "reliability": 0.90,
        "description": "User clicked thumbs down",
        "bias": "High reliability - users rarely give false negatives",
        "volume": "low (2-8% of interactions)",
    },
    SignalType.CORRECTION: {
        "reliability": 0.95,
        "description": "User provided the correct answer",
        "bias": "Highest reliability signal, but very rare",
        "volume": "very low (0.1-1% of interactions)",
    },
    SignalType.DWELL_TIME: {
        "reliability": 0.50,
        "description": "Time spent reading the response",
        "bias": "Long dwell may mean engaged OR confused",
        "volume": "high (100% of interactions)",
    },
    SignalType.REGENERATE: {
        "reliability": 0.75,
        "description": "User clicked regenerate",
        "bias": "Strong negative signal - response was unsatisfactory",
        "volume": "medium (5-20% of interactions)",
    },
    SignalType.COPY_ACTION: {
        "reliability": 0.70,
        "description": "User copied response text",
        "bias": "Positive signal - response was useful enough to copy",
        "volume": "medium (10-30% of interactions)",
    },
}
```

---

## Feedback Collection

```python
import time
import json
from pathlib import Path


class FeedbackCollector:
    """Collect and store user feedback signals."""

    def __init__(self, store_path: str = "./feedback"):
        self.store_path = Path(store_path)
        self.store_path.mkdir(parents=True, exist_ok=True)
        self.buffer: list[FeedbackSignal] = []
        self.buffer_size = 100

    def record_explicit(
        self, query: str, response: str, doc_ids: list[str],
        is_positive: bool, user_id: str = "", session_id: str = "",
    ) -> None:
        """Record thumbs up/down feedback."""
        signal = FeedbackSignal(
            query=query, response=response, retrieved_doc_ids=doc_ids,
            signal_type=SignalType.EXPLICIT_POSITIVE if is_positive else SignalType.EXPLICIT_NEGATIVE,
            signal_value=1.0 if is_positive else -1.0,
            user_id=user_id, session_id=session_id,
            timestamp=time.time(),
        )
        self._store(signal)

    def record_correction(
        self, query: str, original_response: str, corrected_response: str,
        doc_ids: list[str], user_id: str = "",
    ) -> None:
        """Record a user-provided correction."""
        signal = FeedbackSignal(
            query=query, response=original_response, retrieved_doc_ids=doc_ids,
            signal_type=SignalType.CORRECTION, signal_value=-1.0,
            user_id=user_id, timestamp=time.time(),
            metadata={"corrected_response": corrected_response},
        )
        self._store(signal)

    def record_implicit(
        self, query: str, response: str, doc_ids: list[str],
        signal_type: SignalType, signal_value: float,
        user_id: str = "", session_id: str = "",
    ) -> None:
        """Record implicit feedback (dwell time, copy, etc.)."""
        signal = FeedbackSignal(
            query=query, response=response, retrieved_doc_ids=doc_ids,
            signal_type=signal_type, signal_value=signal_value,
            user_id=user_id, session_id=session_id,
            timestamp=time.time(),
        )
        self._store(signal)

    def _store(self, signal: FeedbackSignal) -> None:
        """Store a feedback signal."""
        self.buffer.append(signal)
        if len(self.buffer) >= self.buffer_size:
            self._flush()

    def _flush(self) -> None:
        """Write buffered signals to disk."""
        if not self.buffer:
            return
        timestamp = int(time.time())
        path = self.store_path / f"feedback_{timestamp}.jsonl"
        with open(path, "a") as f:
            for signal in self.buffer:
                record = {
                    "query": signal.query,
                    "response": signal.response[:500],
                    "doc_ids": signal.retrieved_doc_ids,
                    "signal_type": signal.signal_type.value,
                    "signal_value": signal.signal_value,
                    "user_id": signal.user_id,
                    "session_id": signal.session_id,
                    "timestamp": signal.timestamp,
                    "metadata": signal.metadata,
                }
                f.write(json.dumps(record) + "\n")
        self.buffer.clear()

    def get_negative_feedback(self, min_count: int = 3) -> list[dict]:
        """Get queries with repeated negative feedback."""
        from collections import Counter
        negative_queries = Counter()
        for path in self.store_path.glob("feedback_*.jsonl"):
            for line in path.read_text().split("\n"):
                if not line.strip():
                    continue
                record = json.loads(line)
                if record["signal_value"] < 0:
                    negative_queries[record["query"]] += 1

        return [
            {"query": q, "count": c}
            for q, c in negative_queries.most_common()
            if c >= min_count
        ]
```

---

## Common Pitfalls

1. **Only collecting explicit feedback.** Thumbs up/down covers 10-20% of interactions. Implicit signals (dwell time, copy, regenerate) cover 100% and are essential for statistical significance.

2. **Treating all feedback equally.** User corrections are 95% reliable; dwell time is 50% reliable. Weight signals by reliability when aggregating.

3. **Not closing the loop.** Collecting feedback without using it is waste. Schedule monthly reviews of negative feedback patterns and act on them.

4. **Feedback bias.** Power users provide disproportionate feedback. Normalize by user to avoid overfitting to a few vocal users.

---

## References

- Langfuse: https://langfuse.com/docs
- LangSmith: https://docs.smith.langchain.com/
- RLHF for RAG: Ouyang et al. "Training language models to follow instructions" (2022)
