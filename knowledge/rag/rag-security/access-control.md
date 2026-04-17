# ACL-Aware Retrieval -- Per-User Filtering, PII Redaction with Presidio, and GDPR Erasure

## TL;DR

In multi-user RAG systems, the retriever must respect access control lists (ACLs) so users only see documents they are authorized to access. This requires embedding ACL metadata alongside document vectors, filtering during retrieval (not after generation), and handling the full lifecycle of sensitive data: PII detection and redaction before indexing or after retrieval, per-user document permissions that scale to thousands of users, and GDPR-compliant right-to-erasure that removes a user's data from the vector store, caches, and logs. This article covers ACL-aware retrieval patterns, Microsoft Presidio integration for PII detection and redaction, and GDPR erasure workflows for RAG systems.

---

## ACL-Aware Retrieval

### The Problem

Without ACL filtering, a vector similarity search returns the most semantically relevant documents regardless of who is asking:

```
User A (engineer, team-backend):
  Query: "What is the database password?"
  Retrieved: internal-ops/db-credentials.md    <- SECURITY VIOLATION
             deployment/db-config.md           <- Authorized

User B (public, no team):
  Query: "What is the database password?"
  Retrieved: internal-ops/db-credentials.md    <- SECURITY VIOLATION
             public-docs/db-overview.md        <- Authorized
```

### ACL Metadata in Vectors

```python
from dataclasses import dataclass


@dataclass
class DocumentACL:
    """Access control metadata for a document."""

    doc_id: str
    visibility: str  # "public", "internal", "confidential", "restricted"
    allowed_teams: list[str]  # Empty = all teams
    allowed_users: list[str]  # Empty = all users in allowed teams
    allowed_roles: list[str]  # e.g., ["admin", "engineer"]
    owner: str  # Document owner
    classification: str  # "none", "pii", "phi", "financial"


class ACLAwareIndexer:
    """Index documents with ACL metadata for filtered retrieval."""

    def __init__(self, vector_store, embedding_fn):
        self.store = vector_store
        self.embed = embedding_fn

    def index_document(
        self,
        doc_id: str,
        content: str,
        acl: DocumentACL,
        extra_metadata: dict | None = None,
    ) -> None:
        """Index a document with its ACL metadata."""
        metadata = {
            "doc_id": doc_id,
            "visibility": acl.visibility,
            "allowed_teams": acl.allowed_teams,
            "allowed_users": acl.allowed_users,
            "allowed_roles": acl.allowed_roles,
            "owner": acl.owner,
            "classification": acl.classification,
        }
        if extra_metadata:
            metadata.update(extra_metadata)

        self.store.add_texts(
            texts=[content],
            metadatas=[metadata],
        )
```

### Retrieval with ACL Filtering

```python
@dataclass
class UserContext:
    """Security context for the requesting user."""

    user_id: str
    teams: list[str]
    roles: list[str]
    clearance_level: str  # "public", "internal", "confidential", "restricted"


CLEARANCE_HIERARCHY = {
    "public": 0,
    "internal": 1,
    "confidential": 2,
    "restricted": 3,
}


class ACLFilteredRetriever:
    """Retriever that enforces ACL filtering at the vector store level.

    Filters are applied BEFORE results are returned, not after.
    This ensures unauthorized documents never enter the pipeline.
    """

    def __init__(self, vector_store, embedding_fn):
        self.store = vector_store
        self.embed = embedding_fn

    def build_acl_filter(self, user: UserContext) -> dict:
        """Build a vector store filter from user context.

        The filter ensures only authorized documents are returned.
        Logic: document is visible if:
          (visibility <= user clearance) AND
          (allowed_teams is empty OR user's team is in allowed_teams) AND
          (allowed_users is empty OR user_id is in allowed_users)
        """
        user_level = CLEARANCE_HIERARCHY.get(
            user.clearance_level, 0
        )

        # Build visibility filter (documents at or below user's level)
        allowed_visibilities = [
            level
            for level, num in CLEARANCE_HIERARCHY.items()
            if num <= user_level
        ]

        acl_filter = {
            "$and": [
                {"visibility": {"$in": allowed_visibilities}},
                {
                    "$or": [
                        # Document allows all teams
                        {"allowed_teams": {"$eq": []}},
                        # User's team is in allowed list
                        *[
                            {"allowed_teams": {"$contains": team}}
                            for team in user.teams
                        ],
                    ],
                },
            ],
        }

        return acl_filter

    def retrieve(
        self,
        query: str,
        user: UserContext,
        top_k: int = 10,
    ) -> list:
        """Retrieve documents with ACL filtering."""
        acl_filter = self.build_acl_filter(user)

        results = self.store.similarity_search(
            query,
            k=top_k,
            filter=acl_filter,
        )

        # Double-check: verify each result is authorized
        # (defense in depth -- do not rely solely on filter)
        verified = []
        for doc in results:
            if self._is_authorized(doc.metadata, user):
                verified.append(doc)

        return verified

    def _is_authorized(
        self, metadata: dict, user: UserContext
    ) -> bool:
        """Verify authorization (defense in depth)."""
        doc_level = CLEARANCE_HIERARCHY.get(
            metadata.get("visibility", "restricted"), 3
        )
        user_level = CLEARANCE_HIERARCHY.get(
            user.clearance_level, 0
        )

        if doc_level > user_level:
            return False

        allowed_teams = metadata.get("allowed_teams", [])
        if allowed_teams and not any(
            t in allowed_teams for t in user.teams
        ):
            return False

        allowed_users = metadata.get("allowed_users", [])
        if allowed_users and user.user_id not in allowed_users:
            return False

        return True
```

### Role-Based Access Control (RBAC)

```python
class RBACRetriever:
    """Retriever with role-based access control.

    Roles are hierarchical: admin > manager > engineer > viewer.
    Higher roles can access documents intended for lower roles.
    """

    ROLE_HIERARCHY = {
        "viewer": 0,
        "engineer": 1,
        "manager": 2,
        "admin": 3,
    }

    def __init__(self, vector_store):
        self.store = vector_store

    def retrieve(
        self,
        query: str,
        user_roles: list[str],
        top_k: int = 10,
    ) -> list:
        """Retrieve with RBAC filtering."""
        max_role_level = max(
            self.ROLE_HIERARCHY.get(r, 0) for r in user_roles
        )

        # Get all roles at or below user's level
        allowed_roles = [
            role
            for role, level in self.ROLE_HIERARCHY.items()
            if level <= max_role_level
        ]

        return self.store.similarity_search(
            query,
            k=top_k,
            filter={"allowed_roles": {"$in": allowed_roles}},
        )
```

---

## PII Redaction with Presidio

### Setup

```python
# pip install presidio-analyzer presidio-anonymizer

from presidio_analyzer import AnalyzerEngine, RecognizerResult
from presidio_anonymizer import AnonymizerEngine
from presidio_anonymizer.entities import OperatorConfig


class PIIRedactor:
    """Detect and redact PII using Microsoft Presidio.

    Can be applied at two points in the RAG pipeline:
    1. During ingestion (before indexing) -- permanent redaction
    2. During retrieval (before LLM context) -- dynamic redaction
    """

    def __init__(
        self,
        entities_to_redact: list[str] | None = None,
        language: str = "en",
    ):
        self.analyzer = AnalyzerEngine()
        self.anonymizer = AnonymizerEngine()
        self.language = language
        self.entities = entities_to_redact or [
            "PERSON",
            "EMAIL_ADDRESS",
            "PHONE_NUMBER",
            "CREDIT_CARD",
            "US_SSN",
            "IP_ADDRESS",
            "IBAN_CODE",
            "US_BANK_NUMBER",
            "LOCATION",
        ]

    def detect(self, text: str) -> list[dict]:
        """Detect PII entities in text."""
        results = self.analyzer.analyze(
            text=text,
            entities=self.entities,
            language=self.language,
        )

        return [
            {
                "entity_type": r.entity_type,
                "start": r.start,
                "end": r.end,
                "score": r.score,
                "text": text[r.start : r.end],
            }
            for r in results
            if r.score >= 0.7
        ]

    def redact(
        self,
        text: str,
        mode: str = "replace",
    ) -> tuple[str, list[dict]]:
        """Redact PII from text.

        Modes:
        - "replace": Replace with entity type label, e.g., <PERSON>
        - "mask": Replace with asterisks
        - "hash": Replace with hash of the value
        """
        results = self.analyzer.analyze(
            text=text,
            entities=self.entities,
            language=self.language,
        )

        if not results:
            return text, []

        operators = {}
        if mode == "replace":
            for entity in self.entities:
                operators[entity] = OperatorConfig(
                    "replace", {"new_value": f"<{entity}>"}
                )
        elif mode == "mask":
            for entity in self.entities:
                operators[entity] = OperatorConfig(
                    "mask",
                    {
                        "type": "mask",
                        "masking_char": "*",
                        "chars_to_mask": 100,
                        "from_end": False,
                    },
                )
        elif mode == "hash":
            for entity in self.entities:
                operators[entity] = OperatorConfig(
                    "hash", {"hash_type": "sha256"}
                )

        anonymized = self.anonymizer.anonymize(
            text=text,
            analyzer_results=results,
            operators=operators,
        )

        redacted_entities = [
            {
                "entity_type": r.entity_type,
                "original": text[r.start : r.end],
                "score": r.score,
            }
            for r in results
            if r.score >= 0.7
        ]

        return anonymized.text, redacted_entities

    def redact_documents(
        self, documents: list[dict], mode: str = "replace"
    ) -> list[dict]:
        """Redact PII from a list of retrieved documents."""
        redacted = []
        for doc in documents:
            clean_text, entities = self.redact(
                doc["content"], mode
            )
            redacted.append({
                **doc,
                "content": clean_text,
                "pii_redacted": len(entities) > 0,
                "redacted_entities": entities,
            })
        return redacted
```

### Integration with RAG Pipeline

```python
class PIIAwareRAGPipeline:
    """RAG pipeline with PII detection and redaction.

    PII is redacted at two points:
    1. Retrieved documents are redacted before passing to the LLM
    2. Generated output is scanned for PII leakage
    """

    def __init__(self, retriever, llm, pii_redactor: PIIRedactor):
        self.retriever = retriever
        self.llm = llm
        self.redactor = pii_redactor

    def query(
        self, user_query: str, user: UserContext
    ) -> dict:
        # Step 1: Retrieve
        docs = self.retriever.retrieve(user_query, user)

        # Step 2: Redact PII from documents before LLM sees them
        doc_dicts = [
            {"content": d.page_content, "metadata": d.metadata}
            for d in docs
        ]
        redacted_docs = self.redactor.redact_documents(doc_dicts)

        # Step 3: Generate from redacted context
        context = "\n\n".join(d["content"] for d in redacted_docs)
        answer = self.llm.invoke(
            f"Context:\n{context}\n\nQuestion: {user_query}"
        ).content

        # Step 4: Scan output for PII leakage
        output_pii = self.redactor.detect(answer)
        if output_pii:
            # Redact PII from the response too
            answer, _ = self.redactor.redact(answer)

        return {
            "answer": answer,
            "pii_in_context": sum(
                len(d["redacted_entities"]) for d in redacted_docs
            ),
            "pii_in_output": len(output_pii),
        }
```

---

## GDPR Right-to-Erasure

### The Challenge for RAG

GDPR Article 17 requires that personal data be erased upon request. In a RAG system, personal data may exist in:

1. Source documents in the knowledge base
2. Chunked and embedded vectors in the vector store
3. Semantic cache entries
4. Trace logs and evaluation datasets
5. LLM fine-tuning data (if applicable)

### Erasure Workflow

```python
import logging
from dataclasses import dataclass

logger = logging.getLogger(__name__)


@dataclass
class ErasureRequest:
    """GDPR data subject erasure request."""

    request_id: str
    subject_identifier: str  # email, user_id, or name
    subject_type: str  # "email", "user_id", "name"
    requested_at: str
    reason: str = "GDPR Article 17"


class GDPRErasureManager:
    """Handle GDPR right-to-erasure requests for RAG systems.

    Coordinates deletion across all data stores where the
    data subject's information may be stored.
    """

    def __init__(
        self,
        vector_store,
        cache,
        document_store,
        trace_store,
        pii_detector: PIIRedactor,
    ):
        self.vector_store = vector_store
        self.cache = cache
        self.document_store = document_store
        self.trace_store = trace_store
        self.pii_detector = pii_detector

    def process_erasure(
        self, request: ErasureRequest
    ) -> dict:
        """Process a complete erasure request."""
        logger.info(
            "Processing erasure request %s for subject %s",
            request.request_id,
            request.subject_identifier,
        )

        results = {}

        # Step 1: Find documents containing the subject's data
        results["documents"] = self._erase_from_documents(request)

        # Step 2: Delete vectors from the vector store
        results["vectors"] = self._erase_from_vectors(request)

        # Step 3: Clear cache entries
        results["cache"] = self._erase_from_cache(request)

        # Step 4: Redact trace logs
        results["traces"] = self._erase_from_traces(request)

        # Step 5: Generate audit record
        audit = self._create_audit_record(request, results)

        logger.info(
            "Erasure request %s complete: %s",
            request.request_id,
            results,
        )

        return {
            "request_id": request.request_id,
            "results": results,
            "audit": audit,
        }

    def _erase_from_documents(
        self, request: ErasureRequest
    ) -> dict:
        """Find and delete/redact documents containing subject's data."""
        # Search for documents mentioning the subject
        matching_docs = self.document_store.search(
            request.subject_identifier
        )

        deleted = 0
        redacted = 0
        for doc in matching_docs:
            # Check if the entire document is about the subject
            pii_entities = self.pii_detector.detect(doc["content"])
            subject_mentions = [
                e for e in pii_entities
                if request.subject_identifier.lower()
                in e["text"].lower()
            ]

            if len(subject_mentions) > 5:
                # Document is primarily about the subject -- delete it
                self.document_store.delete(doc["doc_id"])
                deleted += 1
            else:
                # Redact subject's data from the document
                redacted_content, _ = self.pii_detector.redact(
                    doc["content"]
                )
                self.document_store.update(
                    doc["doc_id"], redacted_content
                )
                redacted += 1

        return {"deleted": deleted, "redacted": redacted}

    def _erase_from_vectors(
        self, request: ErasureRequest
    ) -> dict:
        """Delete vectors associated with erased documents."""
        # Delete by metadata filter
        deleted = self.vector_store.delete(
            filter={
                "$or": [
                    {
                        "owner": {
                            "$eq": request.subject_identifier
                        }
                    },
                    {
                        "contains_pii_for": {
                            "$contains": request.subject_identifier
                        }
                    },
                ]
            }
        )

        return {"vectors_deleted": deleted or 0}

    def _erase_from_cache(self, request: ErasureRequest) -> dict:
        """Clear cache entries that might contain the subject's data."""
        # Conservative approach: flush all cache entries that
        # were created from documents mentioning the subject
        cleared = self.cache.invalidate_by_subject(
            request.subject_identifier
        )
        return {"cache_entries_cleared": cleared}

    def _erase_from_traces(
        self, request: ErasureRequest
    ) -> dict:
        """Redact subject's data from trace logs."""
        traces = self.trace_store.find_traces_containing(
            request.subject_identifier
        )

        redacted = 0
        for trace in traces:
            # Redact the subject's data from trace inputs/outputs
            self.trace_store.redact_trace(
                trace["trace_id"],
                request.subject_identifier,
                replacement=f"<REDACTED:{request.request_id}>",
            )
            redacted += 1

        return {"traces_redacted": redacted}

    def _create_audit_record(
        self, request: ErasureRequest, results: dict
    ) -> dict:
        """Create an audit trail for the erasure (GDPR requires this)."""
        return {
            "request_id": request.request_id,
            "subject_type": request.subject_type,
            "processed_at": str(time.time()),
            "actions_taken": results,
            "reason": request.reason,
            # Note: Do NOT store the subject_identifier in the audit
            # record, as that would re-create the data we just erased
        }
```

### Preventing Re-Ingestion

```python
class ErasureBlocklist:
    """Maintain a blocklist of erased identifiers to prevent re-ingestion.

    After processing an erasure request, the subject's identifier
    is added to a blocklist. During ingestion, new documents are
    checked against the blocklist and auto-redacted if they contain
    blocked identifiers.
    """

    def __init__(self, redis_client):
        self.redis = redis_client
        self.key = "gdpr:erasure_blocklist"

    def add(self, identifier: str) -> None:
        """Add an identifier to the blocklist."""
        # Store hashed identifier (we do not need the original)
        import hashlib
        hashed = hashlib.sha256(identifier.encode()).hexdigest()
        self.redis.sadd(self.key, hashed)

    def is_blocked(self, identifier: str) -> bool:
        """Check if an identifier is on the blocklist."""
        import hashlib
        hashed = hashlib.sha256(identifier.encode()).hexdigest()
        return bool(self.redis.sismember(self.key, hashed))

    def check_document(
        self,
        content: str,
        pii_detector: PIIRedactor,
    ) -> tuple[bool, list[str]]:
        """Check if a document contains any blocked identifiers."""
        pii_entities = pii_detector.detect(content)
        blocked_entities = []

        for entity in pii_entities:
            if self.is_blocked(entity["text"]):
                blocked_entities.append(entity["text"])

        return len(blocked_entities) > 0, blocked_entities
```

---

## Audit Logging

```python
import json
import logging
from datetime import datetime

audit_logger = logging.getLogger("rag.security.audit")


class SecurityAuditLogger:
    """Log all access control decisions for compliance."""

    def log_access(
        self,
        user: UserContext,
        query: str,
        docs_requested: int,
        docs_returned: int,
        docs_filtered: int,
        reason: str = "",
    ) -> None:
        """Log a retrieval access decision."""
        audit_logger.info(
            json.dumps({
                "event": "document_access",
                "timestamp": datetime.utcnow().isoformat(),
                "user_id": user.user_id,
                "user_teams": user.teams,
                "user_roles": user.roles,
                "user_clearance": user.clearance_level,
                "query_preview": query[:50],
                "docs_matched": docs_requested,
                "docs_returned": docs_returned,
                "docs_filtered_by_acl": docs_filtered,
                "reason": reason,
            })
        )

    def log_pii_redaction(
        self,
        user_id: str,
        doc_id: str,
        entities_redacted: int,
        entity_types: list[str],
    ) -> None:
        """Log PII redaction events."""
        audit_logger.info(
            json.dumps({
                "event": "pii_redaction",
                "timestamp": datetime.utcnow().isoformat(),
                "user_id": user_id,
                "doc_id": doc_id,
                "entities_redacted": entities_redacted,
                "entity_types": entity_types,
            })
        )
```

---

## Common Pitfalls

1. **Filtering after retrieval instead of during retrieval.** If ACL filtering happens after the vector search returns results, unauthorized documents are briefly loaded into memory and may leak through side channels. Filter at the vector store query level.
2. **Not handling the "no results after filtering" case.** If all top-k results are filtered out by ACLs, the user gets an empty response. Detect this and return a helpful message like "No authorized documents match your query."
3. **Storing raw PII in vector metadata.** If you store email addresses in metadata for ACL purposes, they are visible in vector store dumps. Store hashed or encrypted identifiers instead.
4. **Forgetting to erase from caches during GDPR requests.** The vector store is cleaned, but cached answers containing the subject's data still exist. Erasure must cover all data stores.
5. **Not auditing ACL decisions.** Without audit logs, you cannot prove compliance during a GDPR audit. Log every access control decision including what was filtered and why.
6. **Assuming GDPR erasure is a one-time operation.** After erasing data, new documents mentioning the same subject may be ingested. Maintain a blocklist to prevent re-ingestion.

---

## References

- Microsoft Presidio: https://microsoft.github.io/presidio/
- GDPR Article 17 (Right to Erasure): https://gdpr-info.eu/art-17-gdpr/
- OWASP LLM06 Sensitive Information Disclosure: https://owasp.org/www-project-top-10-for-large-language-model-applications/
- Pinecone access control: https://docs.pinecone.io/guides/security/access-control
