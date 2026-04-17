# LLM-as-Judge -- Self-Check Prompts, NLI Models, and the TRUE Metric

## TL;DR

LLM-as-judge uses a language model (the same one that generated the answer, or a separate "judge" model) to evaluate whether a RAG response is faithful to the retrieved context. This is the most flexible and accurate guardrail approach, achieving 80-90% agreement with human annotators on hallucination detection. The three main implementations are: self-check prompts (asking the LLM to verify its own output), dedicated NLI (Natural Language Inference) models fine-tuned for textual entailment, and the TRUE metric (a benchmark-calibrated approach using multiple NLI signals). This article covers prompt engineering for judges, NLI model selection and integration, threshold calibration, and production patterns for running judge evaluations at scale.

---

## Self-Check Prompting

### Concept

The simplest LLM-as-judge approach: after generating a RAG answer, send a second prompt asking an LLM to verify whether each claim in the answer is supported by the context.

### Basic Self-Check

```python
from anthropic import Anthropic


class SelfCheckJudge:
    """Use an LLM to verify its own (or another model's) output
    against the provided context.

    The judge model evaluates each claim independently and produces
    a groundedness verdict with explanation.
    """

    JUDGE_PROMPT = """You are a fact-checking judge. Your task is to determine
whether the ANSWER is fully supported by the CONTEXT.

CONTEXT:
{context}

ANSWER:
{answer}

Evaluate the answer by checking each factual claim against the context.
For each claim, determine:
- SUPPORTED: The context contains information that directly supports this claim
- NOT SUPPORTED: The context does not contain information supporting this claim
- CONTRADICTED: The context contains information that contradicts this claim

Respond in this exact format:

CLAIMS:
1. [claim text] -> [SUPPORTED/NOT SUPPORTED/CONTRADICTED] -- [brief explanation]
2. [claim text] -> [SUPPORTED/NOT SUPPORTED/CONTRADICTED] -- [brief explanation]
...

VERDICT: [GROUNDED/PARTIALLY GROUNDED/NOT GROUNDED]
GROUNDEDNESS_SCORE: [0.0 to 1.0]
"""

    def __init__(
        self,
        judge_model: str = "claude-sonnet-4-20250514",
    ):
        self.client = Anthropic()
        self.judge_model = judge_model

    def evaluate(
        self, answer: str, context: str
    ) -> dict:
        """Evaluate an answer for groundedness."""
        response = self.client.messages.create(
            model=self.judge_model,
            max_tokens=2048,
            messages=[
                {
                    "role": "user",
                    "content": self.JUDGE_PROMPT.format(
                        context=context, answer=answer
                    ),
                },
            ],
        )

        result_text = response.content[0].text
        return self._parse_verdict(result_text)

    def _parse_verdict(self, text: str) -> dict:
        """Parse the structured verdict from the judge response."""
        import re

        claims = []
        claim_pattern = re.compile(
            r"\d+\.\s*(.+?)\s*->\s*(SUPPORTED|NOT SUPPORTED|CONTRADICTED)"
            r"\s*--\s*(.+)"
        )
        for match in claim_pattern.finditer(text):
            claims.append({
                "claim": match.group(1).strip(),
                "verdict": match.group(2).strip(),
                "explanation": match.group(3).strip(),
            })

        # Extract overall verdict
        verdict_match = re.search(
            r"VERDICT:\s*(GROUNDED|PARTIALLY GROUNDED|NOT GROUNDED)",
            text,
        )
        score_match = re.search(
            r"GROUNDEDNESS_SCORE:\s*([\d.]+)", text
        )

        supported = sum(
            1 for c in claims if c["verdict"] == "SUPPORTED"
        )

        return {
            "verdict": (
                verdict_match.group(1) if verdict_match else "UNKNOWN"
            ),
            "score": (
                float(score_match.group(1))
                if score_match
                else supported / max(len(claims), 1)
            ),
            "total_claims": len(claims),
            "supported_claims": supported,
            "unsupported_claims": [
                c for c in claims if c["verdict"] != "SUPPORTED"
            ],
            "all_claims": claims,
        }
```

### Structured Self-Check with Tool Use

Using tool use forces the judge to produce parseable output:

```python
from anthropic import Anthropic
from pydantic import BaseModel, Field


class ClaimVerdict(BaseModel):
    claim: str
    verdict: str = Field(description="SUPPORTED, NOT_SUPPORTED, or CONTRADICTED")
    explanation: str
    supporting_quote: str | None = Field(
        default=None,
        description="Direct quote from context that supports or contradicts",
    )


class GroundednessJudgment(BaseModel):
    claims: list[ClaimVerdict]
    overall_verdict: str = Field(
        description="GROUNDED, PARTIALLY_GROUNDED, or NOT_GROUNDED"
    )
    groundedness_score: float = Field(ge=0.0, le=1.0)
    reasoning: str


class StructuredSelfCheck:
    """Self-check judge using structured output for reliable parsing."""

    def __init__(self, judge_model: str = "claude-sonnet-4-20250514"):
        self.client = Anthropic()
        self.model = judge_model

    def evaluate(self, answer: str, context: str) -> GroundednessJudgment:
        response = self.client.messages.create(
            model=self.model,
            max_tokens=2048,
            system=(
                "You are a groundedness evaluator. Decompose the answer "
                "into claims and check each against the context. "
                "Be strict: if the context does not explicitly support "
                "a claim, mark it as NOT_SUPPORTED."
            ),
            messages=[
                {
                    "role": "user",
                    "content": (
                        f"CONTEXT:\n{context}\n\n"
                        f"ANSWER:\n{answer}\n\n"
                        "Evaluate the groundedness of this answer."
                    ),
                },
            ],
            tools=[
                {
                    "name": "submit_judgment",
                    "description": "Submit the groundedness judgment",
                    "input_schema": GroundednessJudgment.model_json_schema(),
                },
            ],
            tool_choice={"type": "tool", "name": "submit_judgment"},
        )

        tool_block = next(
            b for b in response.content if b.type == "tool_use"
        )
        return GroundednessJudgment.model_validate(tool_block.input)
```

### Multi-Judge Consensus

Using multiple judge instances (different models or prompts) and taking a majority vote:

```python
from concurrent.futures import ThreadPoolExecutor


class MultiJudgeEvaluator:
    """Run multiple judge evaluations and use consensus voting.

    Different models or prompt variants may catch different types
    of hallucinations. Consensus reduces false positives and
    false negatives.
    """

    def __init__(self, judges: list[SelfCheckJudge]):
        self.judges = judges

    def evaluate(
        self, answer: str, context: str
    ) -> dict:
        """Run all judges in parallel and aggregate results."""
        with ThreadPoolExecutor(max_workers=len(self.judges)) as pool:
            futures = [
                pool.submit(judge.evaluate, answer, context)
                for judge in self.judges
            ]
            results = [f.result() for f in futures]

        scores = [r["score"] for r in results]
        verdicts = [r["verdict"] for r in results]

        # Majority vote on verdict
        from collections import Counter
        verdict_counts = Counter(verdicts)
        consensus_verdict = verdict_counts.most_common(1)[0][0]

        return {
            "consensus_verdict": consensus_verdict,
            "mean_score": sum(scores) / len(scores),
            "min_score": min(scores),
            "max_score": max(scores),
            "agreement_rate": (
                verdict_counts.most_common(1)[0][1] / len(verdicts)
            ),
            "individual_results": results,
        }


# Setup with different models as judges
judges = [
    SelfCheckJudge(judge_model="claude-sonnet-4-20250514"),
    SelfCheckJudge(judge_model="claude-haiku-4-20250514"),
]
evaluator = MultiJudgeEvaluator(judges)
```

---

## NLI-Based Groundedness Checking

### Why NLI Models

Natural Language Inference (NLI) models are specifically trained to classify whether a hypothesis is entailed by, contradicted by, or neutral to a premise. This maps directly to groundedness checking:

- Premise = Retrieved context
- Hypothesis = A claim from the RAG answer
- Entailment = Claim is grounded
- Contradiction = Claim contradicts the context
- Neutral = Claim is not supported (possible hallucination)

### Using Cross-Encoder NLI Models

```python
from sentence_transformers import CrossEncoder
import numpy as np


class NLIGroundednessChecker:
    """Check RAG answer groundedness using NLI cross-encoder models.

    Advantages over LLM-as-judge:
    - Much faster (50ms vs 2s per claim)
    - Much cheaper (local inference, no API cost)
    - Deterministic (no temperature variation)

    Disadvantages:
    - Less nuanced (3-class classification vs rich explanation)
    - Shorter context window (512 tokens typically)
    """

    MODELS = {
        "deberta-v3": "cross-encoder/nli-deberta-v3-base",
        "deberta-v3-large": "cross-encoder/nli-deberta-v3-large",
        "roberta": "cross-encoder/nli-roberta-base",
    }

    def __init__(
        self,
        model_name: str = "deberta-v3",
        entailment_threshold: float = 0.7,
    ):
        model_path = self.MODELS.get(model_name, model_name)
        self.model = CrossEncoder(model_path)
        self.threshold = entailment_threshold
        # Label mapping: depends on model
        # DeBERTa: 0=contradiction, 1=entailment, 2=neutral
        self.label_names = ["contradiction", "entailment", "neutral"]

    def check_claim(
        self, claim: str, context: str
    ) -> dict:
        """Check a single claim against the context."""
        # CrossEncoder expects (premise, hypothesis) pairs
        scores = self.model.predict(
            [(context, claim)], apply_softmax=True
        )[0]

        label_idx = int(np.argmax(scores))
        label = self.label_names[label_idx]
        confidence = float(scores[label_idx])

        return {
            "claim": claim,
            "label": label,
            "confidence": confidence,
            "scores": {
                name: float(scores[i])
                for i, name in enumerate(self.label_names)
            },
            "grounded": (
                label == "entailment"
                and confidence >= self.threshold
            ),
        }

    def check_answer(
        self,
        answer: str,
        context: str,
    ) -> dict:
        """Check all claims in an answer."""
        claims = self._split_into_claims(answer)

        results = []
        for claim in claims:
            # NLI models have limited context windows (~512 tokens)
            # Chunk the context and check against each chunk
            context_chunks = self._chunk_context(context, max_chars=1500)
            best_result = None
            best_entailment_score = 0.0

            for chunk in context_chunks:
                result = self.check_claim(claim, chunk)
                if result["scores"]["entailment"] > best_entailment_score:
                    best_entailment_score = result["scores"]["entailment"]
                    best_result = result

            results.append(best_result)

        grounded = sum(1 for r in results if r["grounded"])
        total = len(results)

        return {
            "groundedness_score": grounded / total if total > 0 else 0.0,
            "total_claims": total,
            "grounded_claims": grounded,
            "ungrounded_claims": [
                r["claim"] for r in results if not r["grounded"]
            ],
            "details": results,
        }

    def _split_into_claims(self, text: str) -> list[str]:
        """Split text into individual claims (sentences)."""
        import re
        sentences = re.split(r"(?<=[.!?])\s+", text)
        return [s.strip() for s in sentences if len(s.strip()) > 10]

    def _chunk_context(
        self, context: str, max_chars: int = 1500
    ) -> list[str]:
        """Chunk context to fit NLI model's context window."""
        if len(context) <= max_chars:
            return [context]

        chunks = []
        sentences = context.split(". ")
        current_chunk = ""
        for sentence in sentences:
            if len(current_chunk) + len(sentence) > max_chars:
                if current_chunk:
                    chunks.append(current_chunk)
                current_chunk = sentence
            else:
                current_chunk = (
                    f"{current_chunk}. {sentence}"
                    if current_chunk
                    else sentence
                )
        if current_chunk:
            chunks.append(current_chunk)
        return chunks
```

### Specialized NLI Models for Fact Verification

```python
class AlignScoreChecker:
    """Use AlignScore for fact verification.

    AlignScore is specifically designed for factual consistency
    checking and outperforms general NLI models on this task.
    """

    def __init__(
        self,
        model_name: str = "roberta-large-mnli",
        device: str = "cpu",
    ):
        from transformers import (
            AutoModelForSequenceClassification,
            AutoTokenizer,
        )
        import torch

        self.tokenizer = AutoTokenizer.from_pretrained(model_name)
        self.model = AutoModelForSequenceClassification.from_pretrained(
            model_name
        )
        self.device = torch.device(device)
        self.model.to(self.device)
        self.model.eval()

    def score(self, claim: str, context: str) -> float:
        """Score how well a claim is supported by the context.

        Returns a float from 0.0 (not supported) to 1.0 (fully supported).
        """
        import torch

        inputs = self.tokenizer(
            context,
            claim,
            return_tensors="pt",
            truncation=True,
            max_length=512,
            padding=True,
        ).to(self.device)

        with torch.no_grad():
            outputs = self.model(**inputs)
            probs = torch.softmax(outputs.logits, dim=-1)

        # Index 2 = entailment for MNLI models
        entailment_score = float(probs[0][2])
        return entailment_score

    def check_claims(
        self,
        claims: list[str],
        context: str,
        threshold: float = 0.7,
    ) -> dict:
        """Check multiple claims against context."""
        results = []
        for claim in claims:
            score = self.score(claim, context)
            results.append({
                "claim": claim,
                "score": score,
                "grounded": score >= threshold,
            })

        grounded_count = sum(1 for r in results if r["grounded"])
        return {
            "overall_score": (
                grounded_count / len(results) if results else 0.0
            ),
            "claims": results,
        }
```

---

## The TRUE Metric

### What Is TRUE

TRUE (Transfer and Reasoning over Unsupported Evidence) is a meta-evaluation framework from Google Research that benchmarks factual consistency metrics against human judgments. The key insight: no single metric is best for all tasks. TRUE recommends combining multiple signals.

### Implementing TRUE-Style Evaluation

```python
from dataclasses import dataclass


@dataclass
class TRUEMetricResult:
    """Result from a TRUE-style multi-signal evaluation."""
    nli_score: float         # NLI entailment probability
    qa_score: float          # QA consistency score
    llm_judge_score: float   # LLM-as-judge score
    combined_score: float    # Weighted combination
    is_faithful: bool        # Thresholded decision


class TRUEStyleEvaluator:
    """Combine multiple factual consistency signals following
    the TRUE framework methodology.

    TRUE showed that combining NLI, QA-based, and LLM-based
    metrics gives the best correlation with human judgments.
    The weights below are calibrated on the TRUE benchmark.
    """

    def __init__(
        self,
        nli_checker: NLIGroundednessChecker,
        llm_judge: SelfCheckJudge,
        nli_weight: float = 0.35,
        qa_weight: float = 0.30,
        llm_weight: float = 0.35,
        threshold: float = 0.70,
    ):
        self.nli_checker = nli_checker
        self.llm_judge = llm_judge
        self.nli_weight = nli_weight
        self.qa_weight = qa_weight
        self.llm_weight = llm_weight
        self.threshold = threshold

    def evaluate(
        self,
        answer: str,
        context: str,
        question: str,
    ) -> TRUEMetricResult:
        """Run all three evaluation signals and combine."""
        # Signal 1: NLI score
        nli_result = self.nli_checker.check_answer(answer, context)
        nli_score = nli_result["groundedness_score"]

        # Signal 2: QA consistency score
        qa_score = self._qa_consistency(answer, context, question)

        # Signal 3: LLM judge score
        llm_result = self.llm_judge.evaluate(answer, context)
        llm_score = llm_result["score"]

        # Combine
        combined = (
            self.nli_weight * nli_score
            + self.qa_weight * qa_score
            + self.llm_weight * llm_score
        )

        return TRUEMetricResult(
            nli_score=nli_score,
            qa_score=qa_score,
            llm_judge_score=llm_score,
            combined_score=combined,
            is_faithful=combined >= self.threshold,
        )

    def _qa_consistency(
        self, answer: str, context: str, question: str
    ) -> float:
        """QA-based consistency: generate questions from the answer,
        then check if the context can answer them.

        If the answer contains claims not in the context, the QA model
        will fail to answer the generated questions from context alone.
        """
        from anthropic import Anthropic

        client = Anthropic()

        # Step 1: Generate questions from the answer
        qa_gen_response = client.messages.create(
            model="claude-haiku-4-20250514",
            max_tokens=512,
            messages=[
                {
                    "role": "user",
                    "content": (
                        "Generate 3 yes/no questions that can be answered "
                        "from the following text. Return one question per line.\n\n"
                        f"Text:\n{answer}"
                    ),
                },
            ],
        )
        questions = [
            q.strip()
            for q in qa_gen_response.content[0].text.strip().split("\n")
            if q.strip() and "?" in q
        ][:3]

        if not questions:
            return 0.5  # Cannot evaluate

        # Step 2: Answer those questions using only the context
        consistent = 0
        for q in questions:
            ctx_response = client.messages.create(
                model="claude-haiku-4-20250514",
                max_tokens=100,
                messages=[
                    {
                        "role": "user",
                        "content": (
                            f"Based on this context, answer yes or no:\n\n"
                            f"Context:\n{context}\n\n"
                            f"Question: {q}\n\n"
                            "Answer with ONLY 'yes' or 'no'."
                        ),
                    },
                ],
            )

            # Also answer from the original answer
            ans_response = client.messages.create(
                model="claude-haiku-4-20250514",
                max_tokens=100,
                messages=[
                    {
                        "role": "user",
                        "content": (
                            f"Based on this text, answer yes or no:\n\n"
                            f"Text:\n{answer}\n\n"
                            f"Question: {q}\n\n"
                            "Answer with ONLY 'yes' or 'no'."
                        ),
                    },
                ],
            )

            ctx_answer = ctx_response.content[0].text.strip().lower()
            ans_answer = ans_response.content[0].text.strip().lower()

            if ctx_answer == ans_answer:
                consistent += 1

        return consistent / len(questions)
```

---

## Threshold Calibration

### Why Fixed Thresholds Are Wrong

Different use cases require different thresholds:

| Use Case | Recommended Threshold | Rationale |
|----------|----------------------|-----------|
| Medical/legal | 0.90+ | High cost of hallucination |
| Financial reports | 0.85+ | Regulatory requirements |
| Customer support | 0.70-0.80 | Balance helpfulness and accuracy |
| Internal docs | 0.60-0.70 | Lower stakes, higher tolerance |
| Creative writing | N/A | Groundedness not applicable |

### Data-Driven Threshold Selection

```python
from sklearn.metrics import precision_recall_curve
import numpy as np


def calibrate_threshold(
    human_labels: list[bool],
    model_scores: list[float],
    target_precision: float = 0.90,
) -> dict:
    """Find the optimal threshold that achieves target precision.

    Args:
        human_labels: Ground truth (True = grounded, False = hallucinated)
        model_scores: Model-predicted groundedness scores
        target_precision: Minimum acceptable precision

    Returns:
        Optimal threshold and performance metrics
    """
    precisions, recalls, thresholds = precision_recall_curve(
        human_labels, model_scores
    )

    # Find the threshold that achieves target precision
    # with maximum recall
    valid_indices = np.where(precisions >= target_precision)[0]

    if len(valid_indices) == 0:
        # Cannot achieve target precision
        best_idx = np.argmax(precisions)
        return {
            "optimal_threshold": float(thresholds[best_idx]),
            "achieved_precision": float(precisions[best_idx]),
            "achieved_recall": float(recalls[best_idx]),
            "target_met": False,
        }

    # Among thresholds meeting precision target, pick highest recall
    best_idx = valid_indices[np.argmax(recalls[valid_indices])]

    # Ensure we do not go out of bounds (thresholds is 1 shorter)
    threshold_idx = min(best_idx, len(thresholds) - 1)

    return {
        "optimal_threshold": float(thresholds[threshold_idx]),
        "achieved_precision": float(precisions[best_idx]),
        "achieved_recall": float(recalls[best_idx]),
        "target_met": True,
    }
```

---

## Production Patterns

### Async Judge Pipeline

```python
import asyncio
import logging
from typing import AsyncIterator

logger = logging.getLogger(__name__)


class AsyncJudgePipeline:
    """Run LLM-as-judge evaluation asynchronously for production throughput.

    Supports batch evaluation, concurrent judge execution,
    and streaming results.
    """

    def __init__(
        self,
        judge: SelfCheckJudge,
        max_concurrency: int = 10,
    ):
        self.judge = judge
        self.semaphore = asyncio.Semaphore(max_concurrency)

    async def evaluate_single(
        self, answer: str, context: str, request_id: str
    ) -> dict:
        """Evaluate a single response with concurrency control."""
        async with self.semaphore:
            try:
                # Run synchronous judge in executor
                loop = asyncio.get_event_loop()
                result = await loop.run_in_executor(
                    None, self.judge.evaluate, answer, context
                )
                result["request_id"] = request_id
                return result
            except Exception as e:
                logger.error(
                    "Judge evaluation failed for %s: %s",
                    request_id,
                    str(e),
                )
                return {
                    "request_id": request_id,
                    "score": 0.0,
                    "verdict": "ERROR",
                    "error": str(e),
                }

    async def evaluate_batch(
        self, items: list[dict]
    ) -> list[dict]:
        """Evaluate a batch of responses concurrently."""
        tasks = [
            self.evaluate_single(
                item["answer"],
                item["context"],
                item.get("request_id", str(i)),
            )
            for i, item in enumerate(items)
        ]
        return await asyncio.gather(*tasks)

    async def stream_evaluations(
        self, items: list[dict]
    ) -> AsyncIterator[dict]:
        """Stream evaluation results as they complete."""
        tasks = {
            asyncio.ensure_future(
                self.evaluate_single(
                    item["answer"],
                    item["context"],
                    item.get("request_id", str(i)),
                )
            ): i
            for i, item in enumerate(items)
        }

        for completed in asyncio.as_completed(tasks.keys()):
            result = await completed
            yield result
```

### Offline Evaluation Pipeline

```python
import json
from pathlib import Path


class OfflineJudgeEvaluator:
    """Run judge evaluations on historical RAG responses for
    quality monitoring and threshold calibration."""

    def __init__(
        self,
        judge: SelfCheckJudge,
        output_dir: str = "./eval_results",
    ):
        self.judge = judge
        self.output_dir = Path(output_dir)
        self.output_dir.mkdir(parents=True, exist_ok=True)

    def evaluate_dataset(
        self,
        dataset_path: str,
    ) -> dict:
        """Evaluate a dataset of historical RAG interactions.

        Expected dataset format (JSONL):
        {"query": "...", "answer": "...", "context": "...", "id": "..."}
        """
        results = []
        total_score = 0.0

        with open(dataset_path) as f:
            for line in f:
                item = json.loads(line.strip())
                result = self.judge.evaluate(
                    item["answer"], item["context"]
                )
                result["id"] = item.get("id", "")
                result["query"] = item.get("query", "")
                results.append(result)
                total_score += result["score"]

        # Write results
        output_path = self.output_dir / "judge_results.jsonl"
        with open(output_path, "w") as f:
            for r in results:
                f.write(json.dumps(r) + "\n")

        n = len(results)
        return {
            "total_evaluated": n,
            "mean_score": total_score / n if n > 0 else 0,
            "grounded_pct": (
                sum(1 for r in results if r["score"] >= 0.7)
                / n
                * 100
                if n > 0
                else 0
            ),
            "output_path": str(output_path),
        }
```

---

## Comparing Approaches

| Approach | Accuracy | Latency | Cost | Context Window |
|----------|----------|---------|------|---------------|
| LLM self-check (same model) | 75-85% | 1-3s | $0.01-0.05 | 128K+ |
| LLM cross-check (different model) | 80-90% | 1-3s | $0.01-0.05 | 128K+ |
| NLI cross-encoder | 70-80% | 50ms | $0 (local) | 512 tokens |
| NLI + LLM combined (TRUE-style) | 85-92% | 1-4s | $0.01-0.05 | Hybrid |
| Multi-judge consensus | 88-93% | 2-6s | $0.02-0.10 | 128K+ |

---

## Common Pitfalls

1. **Using the same model as both generator and judge.** The generator and judge share the same biases. If the generator hallucinates a plausible-sounding claim, the same model as judge may rate it as grounded. Use a different model or add NLI as a second signal.
2. **Not chunking context for NLI models.** NLI cross-encoders typically have 512-token context windows. Passing a 5000-token context truncates silently, missing relevant information. Chunk the context and check each claim against all chunks.
3. **Treating the judge score as ground truth.** LLM judges achieve 80-90% agreement with humans, meaning 10-20% of judgments are wrong. Use judge scores for monitoring and alerting, not as hard gates in production without human calibration.
4. **Running judge evaluation synchronously in the request path.** A 2-second judge evaluation doubles the response time. Run judges asynchronously -- return the answer immediately and log the judgment for offline review.
5. **Not calibrating thresholds on your own data.** Published thresholds from papers may not transfer to your domain. Collect 100+ human-annotated examples from your production traffic and calibrate thresholds using precision-recall curves.
6. **Ignoring the cost of QA-based consistency checks.** The TRUE-style QA consistency method requires 6+ additional LLM calls per evaluation (3 question generations + 3 answer verifications from context + 3 from answer). At scale, this becomes the dominant cost.

---

## References

- Honovich et al. "TRUE: Re-evaluating Factual Consistency Evaluation" (NAACL 2022)
- Min et al. "FActScore: Fine-grained Atomic Evaluation of Factual Precision" (EMNLP 2023)
- SelfCheckGPT: Manakul et al. "SelfCheckGPT: Zero-Resource Black-Box Hallucination Detection" (2023)
- DeBERTa NLI models: https://huggingface.co/cross-encoder/nli-deberta-v3-base
- AlignScore: Zha et al. "AlignScore: Evaluating Factual Consistency with a Unified Alignment Function" (2023)
