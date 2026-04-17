# Feedback Loops -- Collection: Thumbs Up/Down, CTR, Dwell Time, Langfuse/LangSmith APIs

## TL;DR

This article provides production implementations for collecting RAG feedback signals using two approaches: (1) custom collection with a lightweight API that captures explicit signals (thumbs up/down, corrections) and implicit signals (click-through rate, dwell time, copy actions), and (2) integration with observability platforms (Langfuse and LangSmith) that provide built-in feedback APIs, trace correlation, and analytics dashboards. Both approaches store feedback correlated with the full RAG trace (query, retrieved documents, generated response) for downstream improvement.

---

## Custom Feedback API

```python
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel, Field
from typing import Literal
import time
import uuid


app = FastAPI(title="RAG Feedback API")


class FeedbackRequest(BaseModel):
    trace_id: str = Field(description="RAG trace ID correlating query->retrieval->generation")
    signal_type: Literal["thumbs_up", "thumbs_down", "correction", "copy", "regenerate"]
    signal_value: float = Field(default=1.0, ge=-1.0, le=1.0)
    user_id: str = ""
    session_id: str = ""
    correction_text: str | None = None
    metadata: dict | None = None


class ImplicitSignalRequest(BaseModel):
    trace_id: str
    signal_type: Literal["dwell_time", "scroll_depth", "click_source"]
    signal_value: float
    user_id: str = ""
    session_id: str = ""


class FeedbackStore:
    """In-memory store (use PostgreSQL/DynamoDB in production)."""
    def __init__(self):
        self.feedback: list[dict] = []

    def add(self, record: dict) -> str:
        feedback_id = str(uuid.uuid4())[:8]
        record["feedback_id"] = feedback_id
        record["created_at"] = time.time()
        self.feedback.append(record)
        return feedback_id

    def get_by_trace(self, trace_id: str) -> list[dict]:
        return [f for f in self.feedback if f.get("trace_id") == trace_id]

    def get_negative(self, limit: int = 100) -> list[dict]:
        negative = [f for f in self.feedback if f.get("signal_value", 0) < 0]
        return sorted(negative, key=lambda x: x["created_at"], reverse=True)[:limit]

    def get_stats(self) -> dict:
        total = len(self.feedback)
        positive = sum(1 for f in self.feedback if f.get("signal_value", 0) > 0)
        negative = sum(1 for f in self.feedback if f.get("signal_value", 0) < 0)
        return {"total": total, "positive": positive, "negative": negative,
                "positive_rate": positive / max(total, 1)}


store = FeedbackStore()


@app.post("/feedback/explicit")
def submit_explicit_feedback(req: FeedbackRequest) -> dict:
    """Submit explicit feedback (thumbs up/down, correction)."""
    record = {
        "trace_id": req.trace_id,
        "signal_type": req.signal_type,
        "signal_value": req.signal_value if req.signal_type == "thumbs_up" else -abs(req.signal_value),
        "user_id": req.user_id,
        "session_id": req.session_id,
        "correction_text": req.correction_text,
        "metadata": req.metadata,
        "category": "explicit",
    }
    feedback_id = store.add(record)
    return {"feedback_id": feedback_id, "status": "recorded"}


@app.post("/feedback/implicit")
def submit_implicit_feedback(req: ImplicitSignalRequest) -> dict:
    """Submit implicit feedback (dwell time, scroll depth, click)."""
    # Normalize dwell time to a -1 to 1 scale
    normalized_value = req.signal_value
    if req.signal_type == "dwell_time":
        # Dwell time heuristic: <5s = negative, 5-60s = neutral, >60s = positive
        if req.signal_value < 5:
            normalized_value = -0.5
        elif req.signal_value < 60:
            normalized_value = 0.0
        else:
            normalized_value = 0.5

    record = {
        "trace_id": req.trace_id,
        "signal_type": req.signal_type,
        "signal_value": normalized_value,
        "raw_value": req.signal_value,
        "user_id": req.user_id,
        "session_id": req.session_id,
        "category": "implicit",
    }
    feedback_id = store.add(record)
    return {"feedback_id": feedback_id, "status": "recorded"}


@app.get("/feedback/stats")
def get_feedback_stats() -> dict:
    return store.get_stats()


@app.get("/feedback/negative")
def get_negative_feedback(limit: int = 50) -> list[dict]:
    return store.get_negative(limit)
```

---

## Langfuse Integration

```python
from langfuse import Langfuse
from langfuse.decorators import observe, langfuse_context


class LangfuseRAGTracker:
    """Track RAG traces and collect feedback via Langfuse.

    Langfuse provides:
    - Automatic trace correlation (query -> retrieval -> generation)
    - Score API for attaching feedback to traces
    - Dashboard for analyzing feedback patterns
    - Export API for training data generation
    """

    def __init__(
        self,
        public_key: str,
        secret_key: str,
        host: str = "https://cloud.langfuse.com",
    ):
        self.langfuse = Langfuse(
            public_key=public_key,
            secret_key=secret_key,
            host=host,
        )

    @observe(name="rag-query")
    def process_query(
        self, query: str, retriever, generator, user_id: str = ""
    ) -> dict:
        """Process a RAG query with full Langfuse tracing."""
        trace = langfuse_context.get_current_trace()

        # Retrieval span
        with trace.span(name="retrieval") as span:
            docs = retriever.retrieve(query)
            span.update(
                input=query,
                output=[d.get("id", "") for d in docs],
                metadata={"doc_count": len(docs)},
            )

        # Generation span
        with trace.span(name="generation") as span:
            context = "\n\n".join(d.get("content", "") for d in docs[:5])
            response = generator.generate(query, context)
            span.update(input={"query": query, "context_length": len(context)},
                        output=response)

        # Tag trace for feedback correlation
        trace.update(
            user_id=user_id,
            metadata={"doc_ids": [d.get("id", "") for d in docs]},
            tags=["rag", "production"],
        )

        return {
            "response": response,
            "trace_id": trace.id,
            "doc_ids": [d.get("id", "") for d in docs],
        }

    def submit_feedback(
        self,
        trace_id: str,
        name: str,
        value: float,
        comment: str = "",
    ) -> None:
        """Submit feedback score to Langfuse."""
        self.langfuse.score(
            trace_id=trace_id,
            name=name,
            value=value,
            comment=comment,
        )

    def get_low_scoring_traces(
        self, score_name: str = "user_feedback", threshold: float = 0.0, limit: int = 100
    ) -> list:
        """Fetch traces with low feedback scores for analysis."""
        # Use Langfuse API to fetch traces
        traces = self.langfuse.fetch_traces(
            limit=limit,
            order_by="created_at",
        )
        low_scoring = []
        for trace in traces.data:
            for score in trace.scores or []:
                if score.name == score_name and score.value <= threshold:
                    low_scoring.append({
                        "trace_id": trace.id,
                        "query": trace.input,
                        "score": score.value,
                        "comment": score.comment,
                    })
        return low_scoring

    def export_training_data(self, min_score: float = 0.5) -> list[dict]:
        """Export positive feedback as training data for fine-tuning."""
        traces = self.langfuse.fetch_traces(limit=1000)
        training_data = []
        for trace in traces.data:
            for score in trace.scores or []:
                if score.name == "user_feedback" and score.value >= min_score:
                    training_data.append({
                        "query": trace.input,
                        "response": trace.output,
                        "score": score.value,
                    })
        return training_data
```

---

## LangSmith Integration

```python
from langsmith import Client as LangSmithClient


class LangSmithRAGTracker:
    """Track RAG traces and collect feedback via LangSmith."""

    def __init__(self, api_key: str | None = None):
        self.client = LangSmithClient(api_key=api_key)

    def submit_feedback(
        self, run_id: str, key: str, score: float, comment: str = ""
    ) -> None:
        """Submit feedback to LangSmith."""
        self.client.create_feedback(
            run_id=run_id,
            key=key,
            score=score,
            comment=comment,
        )

    def get_feedback_dataset(
        self, project_name: str, feedback_key: str = "user_rating"
    ) -> list[dict]:
        """Export runs with feedback for analysis."""
        runs = self.client.list_runs(
            project_name=project_name,
            filter='has(feedback_key, "user_rating")',
        )
        dataset = []
        for run in runs:
            feedbacks = list(self.client.list_feedback(run_ids=[run.id]))
            for fb in feedbacks:
                if fb.key == feedback_key:
                    dataset.append({
                        "query": run.inputs.get("query", ""),
                        "response": run.outputs.get("response", ""),
                        "score": fb.score,
                        "run_id": str(run.id),
                    })
        return dataset
```

---

## Common Pitfalls

1. **Not correlating feedback with traces.** Feedback without trace context (which documents were retrieved, what prompt was used) is useless for debugging. Always attach feedback to a trace ID.

2. **Collecting feedback without analyzing it.** Schedule weekly reviews of negative feedback patterns. The data only helps if someone acts on it.

3. **Not normalizing implicit signals.** Raw dwell time (seconds) is not comparable to thumbs up (binary). Normalize all signals to a consistent scale before aggregating.

4. **Privacy violations in feedback data.** Feedback stores contain user queries that may include PII. Apply the same PII handling as the main RAG pipeline.

---

## References

- Langfuse scores: https://langfuse.com/docs/scores
- LangSmith feedback: https://docs.smith.langchain.com/evaluation/how_to_guides/annotation
