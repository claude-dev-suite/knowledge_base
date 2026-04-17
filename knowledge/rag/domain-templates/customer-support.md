# Domain Templates -- Customer Support RAG: Corpus Shape, Chunking, and Guardrails

## TL;DR

Customer support RAG is the most common production RAG deployment. It serves FAQ answers, troubleshooting guides, and product documentation to both human agents and customer-facing chatbots. The key challenges are: handling short, direct questions against a FAQ-style corpus, maintaining conversation context across multi-turn interactions, enforcing answer boundaries (no off-topic responses), and achieving sub-2-second response times. This article provides the complete template with chunking strategy, retrieval configuration, conversation handling, guardrails, and evaluation setup.

---

## Corpus Shape

```python
from dataclasses import dataclass, field


@dataclass
class SupportCorpusProfile:
    """Typical customer support knowledge base profile."""
    total_documents: int = 5000
    document_types: dict = field(default_factory=lambda: {
        "faq_entries": {"count": 2000, "avg_tokens": 200, "structure": "question-answer pairs"},
        "product_docs": {"count": 1500, "avg_tokens": 2000, "structure": "structured sections"},
        "troubleshooting": {"count": 800, "avg_tokens": 1500, "structure": "problem-steps-resolution"},
        "policy_docs": {"count": 500, "avg_tokens": 3000, "structure": "formal sections"},
        "release_notes": {"count": 200, "avg_tokens": 1000, "structure": "version-feature lists"},
    })
    update_frequency: str = "daily (FAQ), weekly (docs), monthly (policies)"
    languages: list[str] = field(default_factory=lambda: ["en"])
    avg_queries_per_day: int = 10000
    avg_query_length_tokens: int = 15


def configure_support_chunking(corpus: SupportCorpusProfile) -> dict:
    """Configure chunking strategy for customer support corpus.

    Key insight: FAQ entries should NOT be chunked (they are already
    the right size). Product docs need recursive chunking. Policy docs
    need paragraph-level chunking to preserve context.
    """
    return {
        "faq_entries": {
            "strategy": "none",
            "reason": "FAQ entries are already atomic Q&A pairs (200 tokens avg)",
            "chunk_size": None,
        },
        "product_docs": {
            "strategy": "recursive",
            "chunk_size": 500,
            "overlap": 100,
            "reason": "Section-aware splitting for product documentation",
        },
        "troubleshooting": {
            "strategy": "semantic",
            "chunk_size": 800,
            "overlap": 150,
            "reason": "Keep problem-solution pairs together",
        },
        "policy_docs": {
            "strategy": "paragraph",
            "chunk_size": 1000,
            "overlap": 200,
            "reason": "Preserve policy clause boundaries",
        },
    }
```

---

## Retrieval Configuration

```python
class SupportRetriever:
    """Optimized retriever for customer support queries.

    Strategy: Hybrid search (vector + keyword) with cross-encoder reranking.

    Why hybrid: Customer queries often contain exact product names,
    error codes, and version numbers that keyword search handles better
    than vector similarity alone.
    """

    def __init__(self, vector_store, keyword_index, reranker, llm):
        self.vector_store = vector_store
        self.keyword_index = keyword_index
        self.reranker = reranker
        self.llm = llm

    def retrieve(
        self, query: str, conversation_history: list[dict] | None = None, top_k: int = 5
    ) -> list[dict]:
        """Retrieve support documents with conversation awareness."""
        # Step 1: Rewrite query with conversation context
        search_query = query
        if conversation_history:
            search_query = self._contextualize_query(query, conversation_history)

        # Step 2: Hybrid search
        vector_results = self.vector_store.query(
            self._embed(search_query), top_k=top_k * 3
        )
        keyword_results = self.keyword_index.search(search_query, top_k=top_k * 3)

        # Step 3: Merge with reciprocal rank fusion
        merged = self._reciprocal_rank_fusion(vector_results, keyword_results)

        # Step 4: Rerank
        reranked = self.reranker.rerank(search_query, merged[:top_k * 2])

        return reranked[:top_k]

    def _contextualize_query(
        self, query: str, history: list[dict]
    ) -> str:
        """Rewrite query using conversation history for better retrieval."""
        recent = history[-3:]  # last 3 turns
        history_text = "\n".join(
            f"{msg['role']}: {msg['content']}" for msg in recent
        )
        response = self.llm.invoke(
            f"Given this conversation:\n{history_text}\n\n"
            f"And this follow-up question: {query}\n\n"
            f"Rewrite as a standalone search query. Only output the query."
        )
        return response.content.strip()

    @staticmethod
    def _reciprocal_rank_fusion(
        vector_results: list[dict],
        keyword_results: list[dict],
        k: int = 60,
    ) -> list[dict]:
        """Merge results using Reciprocal Rank Fusion."""
        scores: dict[str, float] = {}
        doc_map: dict[str, dict] = {}

        for rank, doc in enumerate(vector_results):
            doc_id = doc.get("id", "")
            scores[doc_id] = scores.get(doc_id, 0) + 1.0 / (k + rank + 1)
            doc_map[doc_id] = doc

        for rank, doc in enumerate(keyword_results):
            doc_id = doc.get("id", "")
            scores[doc_id] = scores.get(doc_id, 0) + 1.0 / (k + rank + 1)
            doc_map[doc_id] = doc

        sorted_ids = sorted(scores.keys(), key=lambda x: scores[x], reverse=True)
        return [doc_map[doc_id] for doc_id in sorted_ids if doc_id in doc_map]

    def _embed(self, text: str) -> list[float]:
        return self.vector_store.embed_query(text)
```

---

## Guardrails

```python
class SupportGuardrails:
    """Guardrails specific to customer support RAG.

    Rules:
    1. Never provide information outside the knowledge base
    2. Detect and mask PII in both queries and responses
    3. Escalate to human agent when confidence is low
    4. Never make promises about refunds/compensation without policy backing
    5. Maintain professional, empathetic tone
    """

    def __init__(self, llm):
        self.llm = llm
        self.escalation_phrases = [
            "speak to a human", "talk to someone", "manager",
            "complaint", "lawyer", "sue", "legal action",
        ]

    def check_query(self, query: str) -> dict:
        """Pre-retrieval guardrail checks on the user query."""
        checks = {"passed": True, "actions": []}

        # PII detection
        pii = self._detect_pii(query)
        if pii:
            checks["actions"].append({
                "type": "pii_detected",
                "fields": pii,
                "action": "mask_before_logging",
            })

        # Escalation detection
        query_lower = query.lower()
        for phrase in self.escalation_phrases:
            if phrase in query_lower:
                checks["actions"].append({
                    "type": "escalation_requested",
                    "action": "route_to_human_agent",
                })
                break

        return checks

    def check_response(
        self, response: str, retrieved_docs: list[dict], query: str
    ) -> dict:
        """Post-generation guardrail checks on the response."""
        checks = {"passed": True, "modifications": []}

        # Check for unsupported claims
        if not retrieved_docs:
            checks["passed"] = False
            checks["modifications"].append({
                "type": "no_sources",
                "action": "replace_with_fallback",
                "fallback": (
                    "I don't have specific information about that in my "
                    "knowledge base. Let me connect you with a support agent "
                    "who can help."
                ),
            })
            return checks

        # Check for promises about refunds/compensation
        promise_keywords = ["refund", "compensation", "credit", "free", "discount"]
        for keyword in promise_keywords:
            if keyword in response.lower():
                # Verify it's backed by a source
                source_texts = " ".join(
                    d.get("content", d.get("text", "")) for d in retrieved_docs
                )
                if keyword not in source_texts.lower():
                    checks["modifications"].append({
                        "type": "unsupported_promise",
                        "keyword": keyword,
                        "action": "add_disclaimer",
                    })

        # PII in response
        pii = self._detect_pii(response)
        if pii:
            checks["modifications"].append({
                "type": "pii_in_response",
                "action": "mask_pii",
            })

        return checks

    @staticmethod
    def _detect_pii(text: str) -> list[str]:
        """Simple PII detection (use a proper NER model in production)."""
        import re
        pii_found = []
        if re.search(r"\b\d{3}[-.]?\d{3}[-.]?\d{4}\b", text):
            pii_found.append("phone_number")
        if re.search(r"\b[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Z|a-z]{2,}\b", text):
            pii_found.append("email")
        if re.search(r"\b\d{3}-?\d{2}-?\d{4}\b", text):
            pii_found.append("ssn")
        if re.search(r"\b\d{4}[\s-]?\d{4}[\s-]?\d{4}[\s-]?\d{4}\b", text):
            pii_found.append("credit_card")
        return pii_found
```

---

## Evaluation

```python
def configure_support_evaluation() -> dict:
    """Configure evaluation metrics for customer support RAG."""
    return {
        "automated_metrics": {
            "faithfulness": {
                "description": "Is the answer supported by retrieved documents?",
                "target": 0.95,
                "tool": "RAGAS or custom LLM judge",
            },
            "answer_relevancy": {
                "description": "Does the answer address the user's question?",
                "target": 0.90,
                "tool": "RAGAS",
            },
            "response_time_ms": {
                "description": "End-to-end latency",
                "target": 2000,
                "tool": "Application metrics",
            },
            "retrieval_precision": {
                "description": "Are retrieved docs relevant?",
                "target": 0.80,
                "tool": "LLM judge on retrieved docs",
            },
        },
        "human_metrics": {
            "csat": {
                "description": "Customer satisfaction score",
                "target": 4.0,
                "scale": "1-5",
            },
            "resolution_rate": {
                "description": "Queries resolved without human escalation",
                "target": 0.70,
            },
        },
    }
```

---

## Common Pitfalls

1. **Chunking FAQ entries.** FAQ Q&A pairs are already atomic. Splitting them destroys the question-answer association.

2. **Not handling conversation context.** "What about the Pro plan?" requires knowing the previous question was about pricing. Without context rewriting, retrieval fails.

3. **Allowing the LLM to make promises.** Without guardrails, the LLM may promise refunds or features that do not exist. Verify promise-related statements against source documents.

4. **Optimizing only for accuracy, ignoring latency.** Support chatbots need sub-2-second responses. Heavy reranking or large context windows may push latency above acceptable limits.

---

## References

- Conversational RAG: LangChain chat with retrieval guide
- PII detection: Microsoft Presidio, Anthropic guardrails
- Customer support RAG evaluation: Zendesk ML blog
