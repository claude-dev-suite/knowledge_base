# RAG Guardrails -- Hallucination Detection, Groundedness, and Citations

## TL;DR

RAG systems hallucinate when the LLM generates claims not supported by the retrieved documents, ignores retrieved evidence in favor of parametric knowledge, or fabricates citations to non-existent sources. Guardrails are runtime checks that intercept these failures before they reach the user. The three main guardrail categories are: hallucination detection (comparing generated claims against retrieved context), groundedness scoring (measuring what fraction of the response is attributable to source documents), and citation enforcement (requiring inline references and verifying they point to actual retrieved passages). This overview covers the threat landscape, guardrail architecture, evaluation metrics (faithfulness, answer relevance, context precision), and integration patterns. Subsequent articles cover NeMo Guardrails and Guardrails AI implementation, and LLM-as-judge evaluation.

---

## The Hallucination Problem in RAG

### Types of RAG Hallucinations

| Type | Description | Example |
|------|-------------|---------|
| Intrinsic | Contradicts the retrieved documents | Doc says "Limit is 1000/min"; LLM says "Limit is 10,000/min" |
| Extrinsic | Adds information not in the documents | Doc says nothing about pricing; LLM invents a price |
| Fabricated citation | Cites a document section that does not exist | "According to Section 4.2..." (no such section) |
| Parametric leakage | Uses training data instead of retrieved docs | Answers with outdated info when current docs are provided |
| Partial grounding | Some claims are grounded, others are not | First paragraph from docs, second paragraph invented |

### Why RAG Does Not Eliminate Hallucination

A common misconception is that providing context documents prevents hallucination entirely. In practice:

1. The LLM may not attend to the context (especially with long contexts)
2. The retrieved documents may not contain the answer, and the LLM fills the gap
3. The LLM may merge information from multiple documents incorrectly
4. Instruction-following pressure ("be helpful") overrides groundedness

---

## Guardrail Architecture

### Where Guardrails Sit in the Pipeline

```
User Query
    |
    v
[Input Guardrails]           <-- Block prompt injection, PII, off-topic
    |
    v
[Retrieval]
    |
    v
[Context Guardrails]         <-- Verify context relevance, sufficiency
    |
    v
[LLM Generation]
    |
    v
[Output Guardrails]          <-- Hallucination check, groundedness, citations
    |
    pass / fail
    |
    v
Return answer (or fallback)
```

### Guardrail Decision Flow

```python
from dataclasses import dataclass
from enum import Enum


class GuardrailAction(Enum):
    PASS = "pass"
    WARN = "warn"
    BLOCK = "block"
    RETRY = "retry"


@dataclass
class GuardrailResult:
    action: GuardrailAction
    score: float              # 0.0 = completely fails, 1.0 = fully passes
    reason: str
    details: dict | None = None


class RAGGuardrailPipeline:
    """Orchestrate multiple guardrails on RAG output."""

    def __init__(
        self,
        groundedness_threshold: float = 0.7,
        relevance_threshold: float = 0.6,
        max_retries: int = 1,
    ):
        self.groundedness_threshold = groundedness_threshold
        self.relevance_threshold = relevance_threshold
        self.max_retries = max_retries

    def check_groundedness(
        self, answer: str, context: str
    ) -> GuardrailResult:
        """Check what fraction of claims in the answer are
        supported by the retrieved context."""
        # Extract claims from the answer
        claims = self._extract_claims(answer)
        if not claims:
            return GuardrailResult(
                action=GuardrailAction.PASS,
                score=1.0,
                reason="No factual claims detected",
            )

        supported = 0
        unsupported_claims = []
        for claim in claims:
            if self._claim_supported_by_context(claim, context):
                supported += 1
            else:
                unsupported_claims.append(claim)

        score = supported / len(claims)

        if score >= self.groundedness_threshold:
            action = GuardrailAction.PASS
        elif score >= self.groundedness_threshold * 0.7:
            action = GuardrailAction.WARN
        else:
            action = GuardrailAction.BLOCK

        return GuardrailResult(
            action=action,
            score=score,
            reason=(
                f"{supported}/{len(claims)} claims supported"
            ),
            details={"unsupported_claims": unsupported_claims},
        )

    def check_relevance(
        self, answer: str, query: str
    ) -> GuardrailResult:
        """Check whether the answer addresses the user's question."""
        score = self._compute_relevance(answer, query)

        if score >= self.relevance_threshold:
            action = GuardrailAction.PASS
        else:
            action = GuardrailAction.RETRY

        return GuardrailResult(
            action=action,
            score=score,
            reason=f"Relevance score: {score:.2f}",
        )

    def check_citations(
        self, answer: str, documents: list[dict]
    ) -> GuardrailResult:
        """Verify that citations in the answer point to real documents."""
        citations = self._extract_citations(answer)
        if not citations:
            return GuardrailResult(
                action=GuardrailAction.WARN,
                score=0.5,
                reason="No citations found in answer",
            )

        doc_ids = {d.get("doc_id", d.get("source", "")) for d in documents}
        valid = sum(1 for c in citations if c in doc_ids)
        score = valid / len(citations) if citations else 0.0

        if score >= 0.9:
            action = GuardrailAction.PASS
        elif score >= 0.5:
            action = GuardrailAction.WARN
        else:
            action = GuardrailAction.BLOCK

        return GuardrailResult(
            action=action,
            score=score,
            reason=f"{valid}/{len(citations)} citations valid",
        )

    def run_all(
        self,
        query: str,
        answer: str,
        context: str,
        documents: list[dict],
    ) -> dict:
        """Run all guardrails and return combined result."""
        results = {
            "groundedness": self.check_groundedness(answer, context),
            "relevance": self.check_relevance(answer, query),
            "citations": self.check_citations(answer, documents),
        }

        # Overall action is the most severe individual action
        severity = {
            GuardrailAction.PASS: 0,
            GuardrailAction.WARN: 1,
            GuardrailAction.RETRY: 2,
            GuardrailAction.BLOCK: 3,
        }
        overall_action = max(
            (r.action for r in results.values()),
            key=lambda a: severity[a],
        )

        return {
            "overall_action": overall_action.value,
            "checks": {
                name: {
                    "action": r.action.value,
                    "score": r.score,
                    "reason": r.reason,
                }
                for name, r in results.items()
            },
        }

    def _extract_claims(self, text: str) -> list[str]:
        """Extract factual claims from text.
        In production, use an LLM or NLI model for this."""
        # Simplified: split into sentences
        import re
        sentences = re.split(r"[.!?]+", text)
        return [s.strip() for s in sentences if len(s.strip()) > 20]

    def _claim_supported_by_context(
        self, claim: str, context: str
    ) -> bool:
        """Check if a claim is supported by context.
        In production, use NLI or LLM-as-judge."""
        # Simplified: keyword overlap
        claim_words = set(claim.lower().split())
        context_words = set(context.lower().split())
        overlap = len(claim_words & context_words) / max(
            len(claim_words), 1
        )
        return overlap > 0.4

    def _compute_relevance(
        self, answer: str, query: str
    ) -> float:
        """Compute answer relevance to query.
        In production, use embedding similarity or LLM judge."""
        query_words = set(query.lower().split())
        answer_words = set(answer.lower().split())
        if not query_words:
            return 0.0
        return len(query_words & answer_words) / len(query_words)

    def _extract_citations(self, text: str) -> list[str]:
        """Extract citation references from text."""
        import re
        # Match patterns like [1], [doc-123], [Source: filename.md]
        return re.findall(r"\[([^\]]+)\]", text)
```

---

## Groundedness Scoring

### Claim-Level Decomposition

The most accurate groundedness checks decompose the answer into individual claims and verify each:

```python
from anthropic import Anthropic


class ClaimDecomposer:
    """Decompose an answer into individual verifiable claims."""

    def __init__(self, model: str = "claude-sonnet-4-20250514"):
        self.client = Anthropic()
        self.model = model

    def decompose(self, answer: str) -> list[str]:
        """Break an answer into atomic factual claims."""
        response = self.client.messages.create(
            model=self.model,
            max_tokens=1024,
            messages=[
                {
                    "role": "user",
                    "content": (
                        "Decompose the following text into a list of "
                        "individual factual claims. Each claim should be "
                        "a single, self-contained statement that can be "
                        "independently verified. Return one claim per line, "
                        "prefixed with '- '.\n\n"
                        f"Text:\n{answer}"
                    ),
                },
            ],
        )
        lines = response.content[0].text.strip().split("\n")
        return [
            line.lstrip("- ").strip()
            for line in lines
            if line.strip().startswith("- ")
        ]


class NLIGroundednessChecker:
    """Check claim groundedness using Natural Language Inference.

    Uses a cross-encoder NLI model to classify each claim as
    ENTAILMENT (supported), CONTRADICTION, or NEUTRAL (not supported).
    """

    def __init__(self, model_name: str = "cross-encoder/nli-deberta-v3-base"):
        from sentence_transformers import CrossEncoder

        self.model = CrossEncoder(model_name)
        self.label_map = {0: "contradiction", 1: "entailment", 2: "neutral"}

    def check_claim(self, claim: str, context: str) -> dict:
        """Check if a single claim is entailed by the context."""
        score = self.model.predict([(context, claim)])[0]

        # score is a 3-element array: [contradiction, entailment, neutral]
        label_idx = score.argmax()
        label = self.label_map[label_idx]
        confidence = float(score[label_idx])

        return {
            "claim": claim,
            "label": label,
            "confidence": confidence,
            "supported": label == "entailment" and confidence > 0.7,
        }

    def check_answer(
        self, answer: str, context: str, decomposer: ClaimDecomposer
    ) -> dict:
        """Check groundedness of an entire answer."""
        claims = decomposer.decompose(answer)
        results = [self.check_claim(c, context) for c in claims]

        supported = sum(1 for r in results if r["supported"])
        total = len(results)

        return {
            "groundedness_score": supported / total if total > 0 else 0.0,
            "total_claims": total,
            "supported_claims": supported,
            "unsupported_claims": [
                r["claim"] for r in results if not r["supported"]
            ],
            "claim_details": results,
        }
```

---

## Citation Enforcement

### Inline Citation Pattern

Force the LLM to cite sources inline and verify them:

```python
CITATION_SYSTEM_PROMPT = """Answer the question based on the provided documents.

CITATION RULES (mandatory):
1. Every factual claim MUST include an inline citation in the format [doc_id].
2. Only cite documents that are provided below. Do NOT invent citations.
3. If no document supports a claim, write "I could not find this in the provided documents."
4. Direct quotes must use quotation marks with the citation: "exact text" [doc_id].

Documents:
{context}
"""


def format_documents_for_citation(
    documents: list[dict],
) -> str:
    """Format documents with clear IDs for citation."""
    parts = []
    for i, doc in enumerate(documents):
        doc_id = doc.get("metadata", {}).get("source", f"doc_{i}")
        parts.append(f"[{doc_id}]\n{doc['content']}\n")
    return "\n---\n".join(parts)


class CitationVerifier:
    """Verify that all citations in an answer reference real documents."""

    def verify(
        self, answer: str, document_ids: set[str]
    ) -> dict:
        """Check every citation in the answer against available docs."""
        import re

        citations = re.findall(r"\[([^\]]+)\]", answer)
        if not citations:
            return {
                "has_citations": False,
                "valid_ratio": 0.0,
                "missing_citations": [],
                "verdict": "NO_CITATIONS",
            }

        valid = []
        invalid = []
        for cite in citations:
            if cite in document_ids:
                valid.append(cite)
            else:
                invalid.append(cite)

        ratio = len(valid) / len(citations)
        return {
            "has_citations": True,
            "total_citations": len(citations),
            "valid_citations": len(valid),
            "invalid_citations": invalid,
            "valid_ratio": ratio,
            "verdict": "PASS" if ratio >= 0.9 else "FAIL",
        }
```

---

## Evaluation Metrics

### RAGAS Framework Metrics

| Metric | What It Measures | Score Range | Threshold |
|--------|-----------------|-------------|-----------|
| Faithfulness | Fraction of claims supported by context | 0-1 | > 0.8 |
| Answer Relevancy | How well the answer addresses the query | 0-1 | > 0.7 |
| Context Precision | Fraction of retrieved docs that are relevant | 0-1 | > 0.6 |
| Context Recall | Fraction of required info present in context | 0-1 | > 0.7 |

### Computing Faithfulness with RAGAS

```python
from ragas import evaluate
from ragas.metrics import faithfulness, answer_relevancy
from datasets import Dataset


def evaluate_rag_faithfulness(
    questions: list[str],
    answers: list[str],
    contexts: list[list[str]],
    ground_truths: list[str] | None = None,
) -> dict:
    """Evaluate RAG responses for faithfulness using RAGAS."""
    data = {
        "question": questions,
        "answer": answers,
        "contexts": contexts,
    }
    if ground_truths:
        data["ground_truth"] = ground_truths

    dataset = Dataset.from_dict(data)

    metrics = [faithfulness, answer_relevancy]
    results = evaluate(dataset, metrics=metrics)

    return {
        "faithfulness": float(results["faithfulness"]),
        "answer_relevancy": float(results["answer_relevancy"]),
        "per_sample": results.to_pandas().to_dict("records"),
    }
```

---

## Fallback Strategies

### What to Do When Guardrails Fail

```python
class GuardrailFallbackHandler:
    """Handle guardrail failures with graceful degradation."""

    def __init__(self, llm, retriever, max_retries: int = 2):
        self.llm = llm
        self.retriever = retriever
        self.max_retries = max_retries

    def handle_failure(
        self,
        query: str,
        failed_answer: str,
        guardrail_result: dict,
        attempt: int = 0,
    ) -> dict:
        """Handle a guardrail failure based on the failure type."""
        action = guardrail_result["overall_action"]

        if action == "warn":
            # Add a disclaimer but return the answer
            disclaimer = (
                "\n\n[Note: This response may contain claims that "
                "could not be fully verified against the source "
                "documents. Please verify critical information.]"
            )
            return {
                "answer": failed_answer + disclaimer,
                "guardrail_status": "passed_with_warning",
            }

        if action == "retry" and attempt < self.max_retries:
            # Re-generate with stricter grounding instructions
            return self._retry_with_stricter_prompt(
                query, guardrail_result, attempt
            )

        if action == "block":
            # Return a safe fallback
            return {
                "answer": (
                    "I found some relevant documents but could not "
                    "generate a reliable answer. Here are the most "
                    "relevant sources for you to review directly."
                ),
                "guardrail_status": "blocked",
                "sources": guardrail_result.get("sources", []),
            }

        return {"answer": failed_answer, "guardrail_status": action}

    def _retry_with_stricter_prompt(
        self, query: str, result: dict, attempt: int
    ) -> dict:
        """Retry generation with explicit grounding constraints."""
        unsupported = result.get("checks", {}).get(
            "groundedness", {}
        ).get("details", {}).get("unsupported_claims", [])

        retry_prompt = (
            "Your previous answer contained claims not supported by "
            "the provided documents. Specifically, these claims could "
            "not be verified:\n"
            + "\n".join(f"- {c}" for c in unsupported[:3])
            + "\n\nPlease answer again, using ONLY information from "
            "the provided documents. If the documents do not contain "
            "the answer, say so explicitly."
        )

        docs = self.retriever.invoke(query)
        context = "\n".join(d.page_content for d in docs)

        answer = self.llm.invoke(
            f"Context:\n{context}\n\n{retry_prompt}\n\nQuestion: {query}"
        ).content

        return {
            "answer": answer,
            "guardrail_status": f"retried_{attempt + 1}",
        }
```

---

## Common Pitfalls

1. **Relying solely on the LLM to self-police hallucinations.** The same model that hallucinated cannot reliably detect its own hallucinations. Use external checks (NLI models, separate judge LLMs, or rule-based verification).
2. **Setting groundedness thresholds too high in production.** A 1.0 threshold rejects most answers because even factually correct responses may phrase things slightly differently from the source documents. Start at 0.7 and calibrate.
3. **Not decomposing answers into claims.** Checking the entire answer as a single unit produces noisy scores. Decompose into atomic claims and check each independently for actionable results.
4. **Ignoring context sufficiency.** If the retrieved documents do not contain the answer, no amount of guardrailing will make the LLM ground its response. Check context relevance before generation.
5. **Blocking without fallback.** If your guardrails block every response that scores below threshold, and your threshold is aggressive, users get no answer at all. Implement tiered fallbacks (warn, retry, block with sources).
6. **Not logging guardrail decisions.** Without logs, you cannot measure false positive rates (good answers blocked) or false negative rates (hallucinations passed). Log every guardrail decision with scores for offline analysis.

---

## References

- RAGAS evaluation framework: https://docs.ragas.io/
- NeMo Guardrails: https://docs.nvidia.com/nemo/guardrails/
- Guardrails AI: https://docs.guardrailsai.com/
- Min et al. "FActScore: Fine-grained Atomic Evaluation of Factual Precision" (2023)
- Honovich et al. "TRUE: Re-evaluating Factual Consistency Evaluation" (NAACL 2022)
