# Domain Templates -- Per-Domain RAG Configuration Templates

## TL;DR

Different domains require fundamentally different RAG configurations. A customer support RAG needs fast retrieval with low latency and conversation-aware context, while a legal RAG needs high-precision retrieval with citation tracking and compliance guardrails. Domain templates encode these domain-specific configurations (chunking strategy, retrieval parameters, guardrails, evaluation criteria) as reusable blueprints. This overview covers the template architecture, provides a framework for creating new domain templates, and identifies the key configuration axes that vary across domains.

---

## Why One-Size-Fits-All RAG Fails

### Configuration Axes That Vary by Domain

```python
from dataclasses import dataclass, field
from typing import Literal


@dataclass
class RAGDomainTemplate:
    """Domain-specific RAG configuration template."""
    domain: str
    description: str

    # Chunking
    chunk_size: int = 1000
    chunk_overlap: int = 200
    chunking_strategy: Literal["fixed", "semantic", "recursive", "paragraph"] = "recursive"

    # Retrieval
    top_k: int = 5
    similarity_threshold: float = 0.7
    reranker: str | None = None
    retrieval_strategy: Literal["vector", "hybrid", "graph", "ontology"] = "vector"

    # Generation
    system_prompt_template: str = ""
    max_response_tokens: int = 1024
    temperature: float = 0.0
    require_citations: bool = False

    # Guardrails
    pii_detection: bool = False
    hallucination_check: bool = True
    domain_boundary_enforcement: bool = False
    max_sources_per_answer: int = 5

    # Evaluation
    eval_metrics: list[str] = field(default_factory=lambda: ["faithfulness", "relevancy"])

    # Corpus
    expected_corpus_size: str = "medium"  # small (<1K), medium (1K-100K), large (>100K)
    update_frequency: str = "weekly"
    document_types: list[str] = field(default_factory=lambda: ["text"])


def list_domain_templates() -> dict[str, RAGDomainTemplate]:
    """Return all available domain templates."""
    return {
        "customer_support": RAGDomainTemplate(
            domain="customer_support",
            description="FAQ, product docs, troubleshooting guides",
            chunk_size=500,
            chunk_overlap=100,
            chunking_strategy="recursive",
            top_k=5,
            similarity_threshold=0.75,
            reranker="cross-encoder",
            retrieval_strategy="hybrid",
            system_prompt_template=(
                "You are a helpful customer support agent. Answer based "
                "ONLY on the provided context. If unsure, say so and "
                "suggest contacting support. Be concise and friendly."
            ),
            max_response_tokens=512,
            temperature=0.1,
            require_citations=False,
            pii_detection=True,
            hallucination_check=True,
            domain_boundary_enforcement=True,
            eval_metrics=["faithfulness", "relevancy", "response_time"],
            expected_corpus_size="medium",
            update_frequency="daily",
            document_types=["faq", "product_docs", "troubleshooting"],
        ),
        "legal": RAGDomainTemplate(
            domain="legal",
            description="Contracts, regulations, case law, compliance",
            chunk_size=1500,
            chunk_overlap=300,
            chunking_strategy="paragraph",
            top_k=10,
            similarity_threshold=0.65,
            reranker="cross-encoder",
            retrieval_strategy="hybrid",
            system_prompt_template=(
                "You are a legal research assistant. Provide accurate "
                "information with exact citations (document, section, "
                "paragraph). Flag any areas of legal uncertainty. "
                "Do NOT provide legal advice."
            ),
            max_response_tokens=2048,
            temperature=0.0,
            require_citations=True,
            pii_detection=True,
            hallucination_check=True,
            domain_boundary_enforcement=True,
            max_sources_per_answer=10,
            eval_metrics=["faithfulness", "citation_accuracy", "completeness"],
            expected_corpus_size="large",
            update_frequency="monthly",
            document_types=["contracts", "regulations", "case_law"],
        ),
        "medical": RAGDomainTemplate(
            domain="medical",
            description="Clinical guidelines, drug information, research",
            chunk_size=1000,
            chunk_overlap=200,
            chunking_strategy="semantic",
            top_k=8,
            similarity_threshold=0.70,
            reranker="cross-encoder",
            retrieval_strategy="ontology",
            system_prompt_template=(
                "You are a medical information assistant. Provide "
                "evidence-based information with citations to sources. "
                "Include confidence levels. Always recommend consulting "
                "a healthcare professional. Never diagnose."
            ),
            max_response_tokens=1500,
            temperature=0.0,
            require_citations=True,
            pii_detection=True,
            hallucination_check=True,
            domain_boundary_enforcement=True,
            eval_metrics=["faithfulness", "citation_accuracy", "medical_accuracy"],
            expected_corpus_size="large",
            update_frequency="quarterly",
            document_types=["clinical_guidelines", "drug_info", "research_papers"],
        ),
        "financial": RAGDomainTemplate(
            domain="financial",
            description="Reports, regulatory filings, market analysis",
            chunk_size=800,
            chunk_overlap=150,
            chunking_strategy="recursive",
            top_k=8,
            similarity_threshold=0.70,
            reranker="cross-encoder",
            retrieval_strategy="hybrid",
            system_prompt_template=(
                "You are a financial research assistant. Provide factual "
                "information from the documents with precise numbers and "
                "dates. Cite sources. Note that past performance does "
                "not indicate future results."
            ),
            max_response_tokens=1500,
            temperature=0.0,
            require_citations=True,
            pii_detection=True,
            hallucination_check=True,
            domain_boundary_enforcement=True,
            eval_metrics=["faithfulness", "numerical_accuracy", "citation_accuracy"],
            expected_corpus_size="large",
            update_frequency="daily",
            document_types=["10k_filings", "earnings_reports", "market_analysis"],
        ),
    }
```

---

## Template Application

```python
class TemplateApplicator:
    """Apply a domain template to configure a RAG pipeline."""

    def __init__(self, template: RAGDomainTemplate):
        self.template = template

    def configure_chunker(self) -> dict:
        """Return chunker configuration from template."""
        return {
            "strategy": self.template.chunking_strategy,
            "chunk_size": self.template.chunk_size,
            "overlap": self.template.chunk_overlap,
        }

    def configure_retriever(self) -> dict:
        """Return retriever configuration from template."""
        config = {
            "strategy": self.template.retrieval_strategy,
            "top_k": self.template.top_k,
            "similarity_threshold": self.template.similarity_threshold,
        }
        if self.template.reranker:
            config["reranker"] = self.template.reranker
        return config

    def configure_generator(self) -> dict:
        """Return generation configuration from template."""
        return {
            "system_prompt": self.template.system_prompt_template,
            "max_tokens": self.template.max_response_tokens,
            "temperature": self.template.temperature,
            "require_citations": self.template.require_citations,
        }

    def configure_guardrails(self) -> dict:
        """Return guardrail configuration from template."""
        return {
            "pii_detection": self.template.pii_detection,
            "hallucination_check": self.template.hallucination_check,
            "domain_boundary": self.template.domain_boundary_enforcement,
            "max_sources": self.template.max_sources_per_answer,
        }

    def get_full_config(self) -> dict:
        """Return the complete pipeline configuration."""
        return {
            "domain": self.template.domain,
            "chunker": self.configure_chunker(),
            "retriever": self.configure_retriever(),
            "generator": self.configure_generator(),
            "guardrails": self.configure_guardrails(),
            "evaluation": {"metrics": self.template.eval_metrics},
        }
```

---

## Common Pitfalls

1. **Using the same chunk size for all domains.** Legal documents need larger chunks (1500+ tokens) to preserve clause context. FAQ entries need smaller chunks (300-500 tokens) for precision.

2. **Skipping domain-specific guardrails.** Medical and legal RAG must include disclaimers and citation requirements. A generic template misses these.

3. **Not tuning similarity thresholds per domain.** Technical documentation has higher semantic density (threshold 0.75+), while legal text uses varied vocabulary (threshold 0.60-0.65).

4. **Applying the same evaluation metrics everywhere.** Customer support cares about response time. Legal cares about citation accuracy. Use domain-appropriate metrics.

---

## References

- LangChain templates: https://github.com/langchain-ai/langchain/tree/master/templates
- Domain-specific RAG patterns: Gao et al. "RAG for LLMs: A Survey" (2024)
