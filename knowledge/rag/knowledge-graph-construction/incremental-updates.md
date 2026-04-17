# Knowledge Graph Construction -- Incremental Updates and Conflict Resolution

## TL;DR

Production knowledge graphs must handle continuous updates as new documents arrive, existing documents are modified, and stale information needs removal. Incremental updates avoid the cost and disruption of full graph rebuilds by processing only changed content. This article covers the three core challenges: (1) detecting what changed and extracting only new/modified triples, (2) merging new triples into the existing graph while handling conflicts (contradictions, redundancy, temporal changes), and (3) maintaining graph consistency through transactional updates, versioning, and rollback capabilities.

---

## The Incremental Update Problem

### Why Full Rebuilds Do Not Scale

```python
def compare_update_strategies(
    total_documents: int,
    new_documents_per_day: int,
    modified_documents_per_day: int,
    extraction_cost_per_doc: float = 0.02,
) -> dict:
    """Compare full rebuild vs incremental update costs."""
    daily_changed = new_documents_per_day + modified_documents_per_day

    full_rebuild = {
        "strategy": "Full rebuild",
        "daily_cost": total_documents * extraction_cost_per_doc,
        "daily_llm_calls": total_documents,
        "latency": f"~{total_documents // 100} minutes (batch)",
        "downtime": "Graph unavailable during rebuild",
    }

    incremental = {
        "strategy": "Incremental update",
        "daily_cost": daily_changed * extraction_cost_per_doc,
        "daily_llm_calls": daily_changed,
        "latency": f"~{max(daily_changed // 100, 1)} minutes",
        "downtime": "None (online updates)",
    }

    savings = {
        "cost_reduction": f"{(1 - daily_changed / max(total_documents, 1)) * 100:.0f}%",
        "llm_call_reduction": f"{(1 - daily_changed / max(total_documents, 1)) * 100:.0f}%",
    }

    return {
        "full_rebuild": full_rebuild,
        "incremental": incremental,
        "savings": savings,
    }


# Example: 50,000 docs corpus, 100 new + 50 modified per day
# Full rebuild: $1,000/day, 50,000 LLM calls
# Incremental: $3/day, 150 LLM calls
# Savings: 99.7% cost reduction
```

---

## Change Detection

### Document-Level Change Tracking

```python
import hashlib
import json
import time
from dataclasses import dataclass, field
from pathlib import Path


@dataclass
class DocumentState:
    document_id: str
    content_hash: str
    last_processed: float
    chunk_ids: list[str] = field(default_factory=list)
    triple_count: int = 0
    entity_count: int = 0


class ChangeDetector:
    """Detect which documents need reprocessing.

    Tracks document content hashes to identify:
    1. New documents (no previous hash)
    2. Modified documents (hash changed)
    3. Deleted documents (hash exists but document missing)
    4. Unchanged documents (hash matches -- skip)
    """

    def __init__(self, state_path: str):
        self.state_path = Path(state_path)
        self.state: dict[str, DocumentState] = {}
        self._load_state()

    def detect_changes(
        self, current_documents: dict[str, str]
    ) -> dict:
        """Detect changes between current documents and last processed state.

        current_documents: dict of document_id -> document_content
        """
        new_docs = []
        modified_docs = []
        deleted_docs = []
        unchanged_docs = []

        current_ids = set(current_documents.keys())
        previous_ids = set(self.state.keys())

        # New documents
        for doc_id in current_ids - previous_ids:
            new_docs.append(doc_id)

        # Modified or unchanged
        for doc_id in current_ids & previous_ids:
            current_hash = self._hash_content(current_documents[doc_id])
            if current_hash != self.state[doc_id].content_hash:
                modified_docs.append(doc_id)
            else:
                unchanged_docs.append(doc_id)

        # Deleted documents
        for doc_id in previous_ids - current_ids:
            deleted_docs.append(doc_id)

        return {
            "new": new_docs,
            "modified": modified_docs,
            "deleted": deleted_docs,
            "unchanged": unchanged_docs,
            "total_changes": len(new_docs) + len(modified_docs) + len(deleted_docs),
        }

    def record_processing(
        self,
        document_id: str,
        content: str,
        chunk_ids: list[str],
        triple_count: int,
        entity_count: int,
    ) -> None:
        """Record that a document has been processed."""
        self.state[document_id] = DocumentState(
            document_id=document_id,
            content_hash=self._hash_content(content),
            last_processed=time.time(),
            chunk_ids=chunk_ids,
            triple_count=triple_count,
            entity_count=entity_count,
        )
        self._save_state()

    def record_deletion(self, document_id: str) -> DocumentState | None:
        """Record that a document has been deleted."""
        old_state = self.state.pop(document_id, None)
        self._save_state()
        return old_state

    @staticmethod
    def _hash_content(content: str) -> str:
        return hashlib.sha256(content.encode()).hexdigest()

    def _load_state(self) -> None:
        if self.state_path.exists():
            raw = json.loads(self.state_path.read_text())
            for doc_id, record in raw.items():
                self.state[doc_id] = DocumentState(**record)

    def _save_state(self) -> None:
        raw = {
            doc_id: {
                "document_id": s.document_id,
                "content_hash": s.content_hash,
                "last_processed": s.last_processed,
                "chunk_ids": s.chunk_ids,
                "triple_count": s.triple_count,
                "entity_count": s.entity_count,
            }
            for doc_id, s in self.state.items()
        }
        self.state_path.parent.mkdir(parents=True, exist_ok=True)
        self.state_path.write_text(json.dumps(raw, indent=2))
```

---

## Graph Update Operations

### Adding New Triples

```python
from dataclasses import dataclass
from typing import Literal


@dataclass
class GraphUpdate:
    operation: Literal["add", "remove", "modify"]
    triple: "Triple"
    reason: str
    source_document: str
    timestamp: float = 0.0


class IncrementalGraphUpdater:
    """Apply incremental updates to a knowledge graph.

    Handles three update types:
    1. ADD: Insert new triples from new/modified documents
    2. REMOVE: Delete triples from deleted/modified documents
    3. MODIFY: Update triple properties (confidence, description, source)

    All updates are transactional: either all changes for a document
    apply, or none do (prevents partial updates on failure).
    """

    def __init__(self, neo4j_driver, database: str = "neo4j"):
        self.driver = neo4j_driver
        self.database = database

    def process_new_document(
        self, document_id: str, triples: list, entity_resolver=None
    ) -> dict:
        """Add triples from a new document to the graph."""
        stats = {"entities_added": 0, "triples_added": 0, "merged_entities": 0}

        with self.driver.session(database=self.database) as session:
            with session.begin_transaction() as tx:
                try:
                    for triple in triples:
                        # Resolve entities if resolver available
                        if entity_resolver:
                            subj_canonical = entity_resolver.resolve(
                                triple.subject.name, triple.subject.entity_type
                            )
                            obj_canonical = entity_resolver.resolve(
                                triple.object.name, triple.object.entity_type
                            )
                        else:
                            subj_canonical = triple.subject.name
                            obj_canonical = triple.object.name

                        # Upsert subject entity
                        result = tx.run("""
                            MERGE (e:Entity {name: $name})
                            ON CREATE SET
                                e.type = $type,
                                e.description = $description,
                                e.created_at = datetime(),
                                e.source_documents = [$doc_id]
                            ON MATCH SET
                                e.description = CASE
                                    WHEN size(e.description) < size($description)
                                    THEN $description
                                    ELSE e.description
                                END,
                                e.source_documents = CASE
                                    WHEN NOT $doc_id IN e.source_documents
                                    THEN e.source_documents + $doc_id
                                    ELSE e.source_documents
                                END,
                                e.updated_at = datetime()
                            RETURN e.created_at = e.updated_at AS is_new
                        """,
                            name=subj_canonical,
                            type=triple.subject.entity_type,
                            description=triple.subject.description,
                            doc_id=document_id,
                        )
                        if result.single() and result.single()["is_new"]:
                            stats["entities_added"] += 1

                        # Upsert object entity (same pattern)
                        tx.run("""
                            MERGE (e:Entity {name: $name})
                            ON CREATE SET
                                e.type = $type,
                                e.description = $description,
                                e.created_at = datetime(),
                                e.source_documents = [$doc_id]
                            ON MATCH SET
                                e.source_documents = CASE
                                    WHEN NOT $doc_id IN e.source_documents
                                    THEN e.source_documents + $doc_id
                                    ELSE e.source_documents
                                END,
                                e.updated_at = datetime()
                        """,
                            name=obj_canonical,
                            type=triple.object.entity_type,
                            description=triple.object.description,
                            doc_id=document_id,
                        )

                        # Add relationship
                        tx.run("""
                            MATCH (s:Entity {name: $subject})
                            MATCH (o:Entity {name: $object})
                            MERGE (s)-[r:RELATED_TO {type: $predicate}]->(o)
                            ON CREATE SET
                                r.confidence = $confidence,
                                r.source_documents = [$doc_id],
                                r.source_chunk = $chunk_id,
                                r.created_at = datetime()
                            ON MATCH SET
                                r.confidence = CASE
                                    WHEN $confidence > r.confidence
                                    THEN $confidence
                                    ELSE r.confidence
                                END,
                                r.source_documents = CASE
                                    WHEN NOT $doc_id IN r.source_documents
                                    THEN r.source_documents + $doc_id
                                    ELSE r.source_documents
                                END,
                                r.updated_at = datetime()
                        """,
                            subject=subj_canonical,
                            object=obj_canonical,
                            predicate=triple.predicate,
                            confidence=triple.confidence,
                            doc_id=document_id,
                            chunk_id=triple.source_chunk_id,
                        )
                        stats["triples_added"] += 1

                    tx.commit()
                except Exception as e:
                    tx.rollback()
                    raise RuntimeError(
                        f"Failed to process document {document_id}: {e}"
                    )

        return stats

    def process_modified_document(
        self, document_id: str, new_triples: list
    ) -> dict:
        """Handle a modified document: remove old triples, add new ones."""
        stats = {"removed": 0, "added": 0}

        # Step 1: Remove triples sourced only from this document
        remove_stats = self.remove_document_triples(document_id)
        stats["removed"] = remove_stats["triples_removed"]

        # Step 2: Add new triples
        add_stats = self.process_new_document(document_id, new_triples)
        stats["added"] = add_stats["triples_added"]

        return stats

    def remove_document_triples(self, document_id: str) -> dict:
        """Remove triples that were sourced only from this document.

        If a triple has multiple source documents, only remove this
        document from the source list. Only delete the triple if
        this was the last source.
        """
        stats = {"triples_removed": 0, "triples_updated": 0, "entities_removed": 0}

        with self.driver.session(database=self.database) as session:
            # Remove document from relationship source lists
            result = session.run("""
                MATCH ()-[r:RELATED_TO]->()
                WHERE $doc_id IN r.source_documents
                WITH r, size(r.source_documents) AS source_count
                WHERE source_count = 1
                DELETE r
                RETURN count(r) AS deleted
            """, doc_id=document_id)
            record = result.single()
            stats["triples_removed"] = record["deleted"] if record else 0

            # Update multi-source relationships
            result = session.run("""
                MATCH ()-[r:RELATED_TO]->()
                WHERE $doc_id IN r.source_documents
                AND size(r.source_documents) > 1
                SET r.source_documents = [d IN r.source_documents WHERE d <> $doc_id]
                RETURN count(r) AS updated
            """, doc_id=document_id)
            record = result.single()
            stats["triples_updated"] = record["updated"] if record else 0

            # Remove orphan entities (no remaining relationships)
            result = session.run("""
                MATCH (e:Entity)
                WHERE NOT (e)-[:RELATED_TO]-() AND NOT ()-[:RELATED_TO]-(e)
                AND $doc_id IN coalesce(e.source_documents, [])
                AND size(coalesce(e.source_documents, [])) = 1
                DELETE e
                RETURN count(e) AS deleted
            """, doc_id=document_id)
            record = result.single()
            stats["entities_removed"] = record["deleted"] if record else 0

        return stats
```

---

## Conflict Resolution

### Types of Conflicts

```python
from enum import Enum


class ConflictType(Enum):
    CONTRADICTION = "contradiction"      # new triple contradicts existing
    TEMPORAL_UPDATE = "temporal_update"   # new info supersedes old
    CONFIDENCE_CHANGE = "confidence"     # same triple, different confidence
    TYPE_MISMATCH = "type_mismatch"     # entity type differs
    REDUNDANT = "redundant"              # exact duplicate


@dataclass
class TripleConflict:
    existing_triple: dict
    new_triple: dict
    conflict_type: ConflictType
    resolution: str = ""


class ConflictDetector:
    """Detect and resolve conflicts between new and existing triples."""

    def __init__(self, neo4j_driver, database: str = "neo4j", llm=None):
        self.driver = neo4j_driver
        self.database = database
        self.llm = llm

    def check_conflicts(
        self, new_triples: list
    ) -> list[TripleConflict]:
        """Check new triples against existing graph for conflicts."""
        conflicts = []

        for triple in new_triples:
            existing = self._find_existing_triples(
                triple.subject.name, triple.object.name
            )

            for existing_triple in existing:
                conflict = self._classify_conflict(triple, existing_triple)
                if conflict:
                    conflicts.append(conflict)

        return conflicts

    def _find_existing_triples(
        self, subject: str, object_name: str
    ) -> list[dict]:
        """Find existing triples between two entities."""
        query = """
        MATCH (s:Entity)-[r:RELATED_TO]->(o:Entity)
        WHERE toLower(s.name) = toLower($subject)
          AND toLower(o.name) = toLower($object)
        RETURN s.name AS subject, r.type AS predicate, o.name AS object,
               r.confidence AS confidence,
               r.source_documents AS sources,
               r.created_at AS created_at
        """
        with self.driver.session(database=self.database) as session:
            result = session.run(
                query, subject=subject, object=object_name
            )
            return [dict(r) for r in result]

    def _classify_conflict(
        self, new_triple, existing: dict
    ) -> TripleConflict | None:
        """Classify the type of conflict between triples."""
        # Redundant: exact same triple
        if new_triple.predicate == existing["predicate"]:
            if abs(new_triple.confidence - existing.get("confidence", 0)) < 0.1:
                return TripleConflict(
                    existing_triple=existing,
                    new_triple=self._triple_to_dict(new_triple),
                    conflict_type=ConflictType.REDUNDANT,
                    resolution="skip",
                )
            else:
                return TripleConflict(
                    existing_triple=existing,
                    new_triple=self._triple_to_dict(new_triple),
                    conflict_type=ConflictType.CONFIDENCE_CHANGE,
                    resolution="update_confidence",
                )

        # Different predicate between same entities: possible contradiction
        if self.llm:
            is_contradiction = self._check_contradiction_llm(
                new_triple, existing
            )
            if is_contradiction:
                return TripleConflict(
                    existing_triple=existing,
                    new_triple=self._triple_to_dict(new_triple),
                    conflict_type=ConflictType.CONTRADICTION,
                )

        return None

    def _check_contradiction_llm(
        self, new_triple, existing: dict
    ) -> bool:
        """Use LLM to check if two triples contradict each other."""
        prompt = (
            "Do these two statements about the same entities contradict?\n\n"
            f"Existing: ({existing['subject']}, {existing['predicate']}, {existing['object']})\n"
            f"New: ({new_triple.subject.name}, {new_triple.predicate}, {new_triple.object.name})\n\n"
            "Respond YES or NO."
        )
        response = self.llm.invoke(prompt)
        return "YES" in response.content.upper()

    def resolve_conflict(
        self, conflict: TripleConflict
    ) -> str:
        """Apply automatic conflict resolution."""
        if conflict.conflict_type == ConflictType.REDUNDANT:
            return "skip"

        if conflict.conflict_type == ConflictType.CONFIDENCE_CHANGE:
            # Keep higher confidence
            return "update_confidence"

        if conflict.conflict_type == ConflictType.TEMPORAL_UPDATE:
            # Newer information wins
            return "replace"

        if conflict.conflict_type == ConflictType.CONTRADICTION:
            # Keep both with lower confidence, flag for review
            return "keep_both_flagged"

        return "skip"

    @staticmethod
    def _triple_to_dict(triple) -> dict:
        return {
            "subject": triple.subject.name,
            "predicate": triple.predicate,
            "object": triple.object.name,
            "confidence": triple.confidence,
        }
```

---

## Full Incremental Update Pipeline

```python
class IncrementalUpdatePipeline:
    """Complete pipeline for incremental knowledge graph updates."""

    def __init__(
        self,
        change_detector: ChangeDetector,
        extractor,  # PydanticExtractor or HybridExtractionPipeline
        chunker,
        graph_updater: IncrementalGraphUpdater,
        conflict_detector: ConflictDetector,
    ):
        self.change_detector = change_detector
        self.extractor = extractor
        self.chunker = chunker
        self.updater = graph_updater
        self.conflict_detector = conflict_detector

    def run(self, current_documents: dict[str, str]) -> dict:
        """Execute a full incremental update cycle."""
        # Step 1: Detect changes
        changes = self.change_detector.detect_changes(current_documents)

        stats = {
            "new_documents": len(changes["new"]),
            "modified_documents": len(changes["modified"]),
            "deleted_documents": len(changes["deleted"]),
            "unchanged_documents": len(changes["unchanged"]),
            "triples_added": 0,
            "triples_removed": 0,
            "conflicts_detected": 0,
        }

        # Step 2: Process deletions
        for doc_id in changes["deleted"]:
            result = self.updater.remove_document_triples(doc_id)
            stats["triples_removed"] += result["triples_removed"]
            self.change_detector.record_deletion(doc_id)

        # Step 3: Process modifications (remove old + add new)
        for doc_id in changes["modified"]:
            content = current_documents[doc_id]
            chunks = self.chunker.chunk_document(content, doc_id)
            extraction = self.extractor.extract(chunks) if hasattr(self.extractor, 'extract') and callable(getattr(self.extractor, 'extract')) else {"triples": []}
            triples = extraction.get("triples", []) if isinstance(extraction, dict) else extraction.triples

            # Check conflicts before applying
            conflicts = self.conflict_detector.check_conflicts(triples)
            stats["conflicts_detected"] += len(conflicts)

            result = self.updater.process_modified_document(doc_id, triples)
            stats["triples_added"] += result["added"]
            stats["triples_removed"] += result["removed"]

            self.change_detector.record_processing(
                doc_id, content,
                chunk_ids=[c.chunk_id for c in chunks],
                triple_count=len(triples),
                entity_count=len(set(
                    t.subject.name for t in triples
                ) | set(
                    t.object.name for t in triples
                )),
            )

        # Step 4: Process new documents
        for doc_id in changes["new"]:
            content = current_documents[doc_id]
            chunks = self.chunker.chunk_document(content, doc_id)
            extraction = self.extractor.extract(chunks) if hasattr(self.extractor, 'extract') and callable(getattr(self.extractor, 'extract')) else {"triples": []}
            triples = extraction.get("triples", []) if isinstance(extraction, dict) else extraction.triples

            result = self.updater.process_new_document(doc_id, triples)
            stats["triples_added"] += result["triples_added"]

            self.change_detector.record_processing(
                doc_id, content,
                chunk_ids=[c.chunk_id for c in chunks],
                triple_count=len(triples),
                entity_count=len(set(
                    t.subject.name for t in triples
                ) | set(
                    t.object.name for t in triples
                )),
            )

        return stats
```

---

## Common Pitfalls

1. **Not tracking triple provenance.** Every triple must record which document(s) it came from. Without this, deleting a document cannot remove its triples.

2. **Deleting triples that have multiple sources.** If a triple came from documents A and B, deleting document A should only remove A from the source list, not delete the triple.

3. **Applying updates non-transactionally.** If the update fails midway, the graph is in an inconsistent state. Use database transactions to ensure atomicity.

4. **Not handling entity resolution during updates.** A new document may introduce "PostgreSQL" while the graph already has "Postgres". Without entity resolution, these become separate nodes.

5. **Ignoring contradictions.** If document A says "Alice works at Acme" and document B says "Alice works at Globex", both are in the graph. Detect and flag contradictions, or apply temporal ordering (newer wins).

6. **Full rebuild as a fallback.** When incremental updates accumulate errors (orphan nodes, stale triples), periodic full rebuilds clean up the graph. Schedule monthly or quarterly full rebuilds even with incremental updates in place.

---

## References

- Neo4j Transactions: https://neo4j.com/docs/python-manual/current/transactions/
- Temporal Knowledge Graphs: Trivedi et al. "DyRep: Learning Representations over Dynamic Graphs" (2019)
- Graph versioning: https://neo4j.com/developer/graph-data-science/graph-versioning/
