# Domain Templates -- Legal, Medical, and Financial RAG: Compliance, PII, and Citations

## TL;DR

Regulated domains (legal, medical, financial) impose strict requirements on RAG systems that go beyond accuracy. Legal RAG must provide exact citations to statutes and clauses. Medical RAG must never provide diagnostic advice and must cite evidence grades. Financial RAG must include regulatory disclaimers and handle material non-public information carefully. This article provides domain-specific configurations, compliance guardrails, citation mechanisms, and PII handling strategies for each of these regulated domains.

---

## Legal RAG

### Configuration

```python
from dataclasses import dataclass, field


@dataclass
class LegalRAGConfig:
    """Configuration for legal document RAG."""
    # Chunking: preserve clause boundaries
    chunk_size: int = 1500
    chunk_overlap: int = 300
    chunking_strategy: str = "paragraph"
    preserve_section_headers: bool = True

    # Retrieval: high recall, then precise reranking
    top_k: int = 15
    rerank_top_k: int = 8
    similarity_threshold: float = 0.60
    retrieval_strategy: str = "hybrid"

    # Citations: mandatory
    citation_format: str = "inline"  # inline, footnote, appendix
    require_section_reference: bool = True
    require_document_reference: bool = True

    # Guardrails
    disclaimer: str = (
        "This information is for research purposes only and does not "
        "constitute legal advice. Consult a qualified attorney for "
        "advice on your specific situation."
    )
    prohibit_advice: bool = True
    flag_jurisdictional_differences: bool = True


class LegalCitationGenerator:
    """Generate precise legal citations from retrieved documents."""

    def format_citation(self, doc_metadata: dict) -> str:
        """Format a citation based on document type."""
        doc_type = doc_metadata.get("type", "unknown")

        if doc_type == "statute":
            return self._format_statute(doc_metadata)
        elif doc_type == "case_law":
            return self._format_case(doc_metadata)
        elif doc_type == "contract":
            return self._format_contract(doc_metadata)
        elif doc_type == "regulation":
            return self._format_regulation(doc_metadata)
        else:
            return self._format_generic(doc_metadata)

    @staticmethod
    def _format_statute(meta: dict) -> str:
        title = meta.get("title", "")
        section = meta.get("section", "")
        subsection = meta.get("subsection", "")
        return f"{title}, Section {section}" + (f"({subsection})" if subsection else "")

    @staticmethod
    def _format_case(meta: dict) -> str:
        case_name = meta.get("case_name", "")
        citation = meta.get("citation", "")
        year = meta.get("year", "")
        return f"{case_name}, {citation} ({year})"

    @staticmethod
    def _format_contract(meta: dict) -> str:
        doc_name = meta.get("document_name", "Agreement")
        section = meta.get("section", "")
        clause = meta.get("clause", "")
        return f"{doc_name}, Section {section}" + (f", Clause {clause}" if clause else "")

    @staticmethod
    def _format_regulation(meta: dict) -> str:
        body = meta.get("regulatory_body", "")
        reg_number = meta.get("regulation_number", "")
        section = meta.get("section", "")
        return f"{body} {reg_number}" + (f", Section {section}" if section else "")

    @staticmethod
    def _format_generic(meta: dict) -> str:
        source = meta.get("source", meta.get("document_name", "Unknown"))
        page = meta.get("page", "")
        return f"{source}" + (f", p. {page}" if page else "")
```

---

## Medical RAG

### Configuration

```python
@dataclass
class MedicalRAGConfig:
    """Configuration for medical/clinical RAG."""
    chunk_size: int = 1000
    chunk_overlap: int = 200
    chunking_strategy: str = "semantic"

    top_k: int = 10
    similarity_threshold: float = 0.70
    retrieval_strategy: str = "ontology"  # use SNOMED CT
    ontology: str = "snomed_ct"

    # Evidence grading
    require_evidence_level: bool = True
    evidence_hierarchy: list[str] = field(default_factory=lambda: [
        "systematic_review_meta_analysis",
        "randomized_controlled_trial",
        "cohort_study",
        "case_control_study",
        "case_series",
        "expert_opinion",
    ])

    # Safety guardrails
    disclaimer: str = (
        "This information is for educational purposes only and is not "
        "a substitute for professional medical advice, diagnosis, or "
        "treatment. Always consult a qualified healthcare provider."
    )
    prohibit_diagnosis: bool = True
    prohibit_treatment_recommendation: bool = True
    flag_drug_interactions: bool = True
    require_recency_check: bool = True  # flag outdated guidelines
    max_guideline_age_years: int = 5


class MedicalSafetyChecker:
    """Safety checks specific to medical RAG."""

    PROHIBITED_PATTERNS = [
        "you have", "you are diagnosed with", "you should take",
        "take this medication", "stop taking", "your diagnosis is",
        "you need surgery", "you do not need",
    ]

    def check_response(self, response: str, sources: list[dict]) -> dict:
        """Check medical response for safety violations."""
        issues = []

        # Check for diagnostic language
        response_lower = response.lower()
        for pattern in self.PROHIBITED_PATTERNS:
            if pattern in response_lower:
                issues.append({
                    "type": "prohibited_language",
                    "pattern": pattern,
                    "severity": "high",
                })

        # Check source recency
        import time
        current_year = 2026
        for source in sources:
            pub_year = source.get("publication_year", current_year)
            age = current_year - pub_year
            if age > 5:
                issues.append({
                    "type": "outdated_source",
                    "source": source.get("title", "Unknown"),
                    "age_years": age,
                    "severity": "medium",
                })

        # Check evidence level
        evidence_levels = [s.get("evidence_level", "unknown") for s in sources]
        if all(level in ("expert_opinion", "case_series", "unknown") for level in evidence_levels):
            issues.append({
                "type": "weak_evidence",
                "severity": "medium",
                "message": "All sources are low-evidence-level",
            })

        return {
            "safe": len([i for i in issues if i["severity"] == "high"]) == 0,
            "issues": issues,
        }
```

---

## Financial RAG

### Configuration

```python
@dataclass
class FinancialRAGConfig:
    """Configuration for financial document RAG."""
    chunk_size: int = 800
    chunk_overlap: int = 150
    chunking_strategy: str = "recursive"

    top_k: int = 10
    similarity_threshold: float = 0.70
    retrieval_strategy: str = "hybrid"

    # Numerical accuracy
    require_numerical_verification: bool = True
    currency_formatting: str = "USD"
    decimal_precision: int = 2

    # Regulatory
    disclaimer: str = (
        "This information is for informational purposes only and does "
        "not constitute investment advice. Past performance is not "
        "indicative of future results."
    )
    include_forward_looking_disclaimer: bool = True
    detect_material_nonpublic_info: bool = True

    # Temporal
    require_date_attribution: bool = True
    flag_stale_financial_data_days: int = 90


class FinancialGuardrails:
    """Guardrails for financial RAG systems."""

    INVESTMENT_ADVICE_PATTERNS = [
        "you should buy", "you should sell", "invest in",
        "guaranteed return", "risk-free", "can't lose",
        "hot tip", "insider",
    ]

    FORWARD_LOOKING_PATTERNS = [
        "will increase", "will decrease", "expected to",
        "projected", "forecast", "anticipated",
    ]

    def check_response(self, response: str, sources: list[dict]) -> dict:
        """Check financial response for compliance issues."""
        issues = []
        response_lower = response.lower()

        # Investment advice detection
        for pattern in self.INVESTMENT_ADVICE_PATTERNS:
            if pattern in response_lower:
                issues.append({
                    "type": "investment_advice",
                    "pattern": pattern,
                    "action": "remove_or_add_disclaimer",
                })

        # Forward-looking statements
        for pattern in self.FORWARD_LOOKING_PATTERNS:
            if pattern in response_lower:
                issues.append({
                    "type": "forward_looking_statement",
                    "pattern": pattern,
                    "action": "add_forward_looking_disclaimer",
                })

        # Stale data check
        for source in sources:
            date_str = source.get("date", source.get("filing_date", ""))
            if date_str:
                # Simple age check
                age_days = source.get("age_days", 0)
                if age_days > 90:
                    issues.append({
                        "type": "stale_data",
                        "source": source.get("title", ""),
                        "age_days": age_days,
                        "action": "flag_data_age",
                    })

        return {"compliant": len(issues) == 0, "issues": issues}
```

---

## PII Handling Across Domains

```python
class DomainPIIHandler:
    """Domain-aware PII detection and handling.

    Different domains have different PII sensitivities:
    - Legal: client names, case numbers, addresses
    - Medical: patient IDs, diagnoses, medications (HIPAA)
    - Financial: account numbers, SSN, income (GLBA, SOX)
    """

    DOMAIN_PII_FIELDS = {
        "legal": [
            "client_name", "case_number", "address",
            "social_security_number", "date_of_birth",
        ],
        "medical": [
            "patient_id", "medical_record_number", "diagnosis",
            "medication", "date_of_birth", "address",
            "insurance_id", "provider_name",
        ],
        "financial": [
            "account_number", "routing_number", "social_security_number",
            "income", "credit_score", "tax_id",
        ],
    }

    DOMAIN_REGULATIONS = {
        "legal": ["ABA Model Rules", "Attorney-Client Privilege"],
        "medical": ["HIPAA", "HITECH Act", "42 CFR Part 2"],
        "financial": ["GLBA", "SOX", "GDPR", "CCPA", "PCI-DSS"],
    }

    def __init__(self, domain: str):
        self.domain = domain
        self.sensitive_fields = self.DOMAIN_PII_FIELDS.get(domain, [])
        self.regulations = self.DOMAIN_REGULATIONS.get(domain, [])

    def scan_document(self, text: str) -> dict:
        """Scan a document for domain-specific PII before indexing."""
        import re
        findings = []

        # Generic PII patterns
        patterns = {
            "email": r"\b[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Z|a-z]{2,}\b",
            "phone": r"\b\d{3}[-.]?\d{3}[-.]?\d{4}\b",
            "ssn": r"\b\d{3}-?\d{2}-?\d{4}\b",
            "credit_card": r"\b\d{4}[\s-]?\d{4}[\s-]?\d{4}[\s-]?\d{4}\b",
        }

        for pii_type, pattern in patterns.items():
            matches = re.findall(pattern, text)
            if matches:
                findings.append({
                    "type": pii_type,
                    "count": len(matches),
                    "action": "mask_before_indexing",
                })

        return {
            "domain": self.domain,
            "pii_found": len(findings) > 0,
            "findings": findings,
            "applicable_regulations": self.regulations,
        }

    def mask_pii(self, text: str) -> str:
        """Replace detected PII with masked values."""
        import re
        text = re.sub(r"\b\d{3}-?\d{2}-?\d{4}\b", "[SSN_REDACTED]", text)
        text = re.sub(r"\b\d{4}[\s-]?\d{4}[\s-]?\d{4}[\s-]?\d{4}\b", "[CC_REDACTED]", text)
        text = re.sub(
            r"\b[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Z|a-z]{2,}\b",
            "[EMAIL_REDACTED]", text
        )
        return text
```

---

## Common Pitfalls

1. **Treating regulated domains like general RAG.** A "hallucinated" medical diagnosis or legal interpretation can cause real harm. Guardrails are not optional.

2. **Not including disclaimers in every response.** Regulated domains require disclaimers. Append them automatically, not conditionally.

3. **Indexing PII without masking.** If patient SSNs end up in vector store metadata, you have a HIPAA violation. Scan and mask before indexing.

4. **Ignoring source recency.** A 2018 clinical guideline may be superseded by a 2024 update. Always check and flag source age.

5. **Providing investment advice through RAG.** Even a well-meaning "based on the data, stock X looks good" can violate securities regulations. Detect and remove advisory language.

---

## References

- HIPAA Privacy Rule: https://www.hhs.gov/hipaa/for-professionals/privacy/
- SEC Guidance on AI: https://www.sec.gov/
- ABA Model Rules of Professional Conduct
- GDPR: https://gdpr.eu/
