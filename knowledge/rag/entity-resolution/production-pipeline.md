# Entity Resolution -- Production Pipeline: Incremental Resolution and Canonical ID Mapping

## TL;DR

Production entity resolution requires more than a one-shot batch process. As new documents are ingested into a RAG system, new entity mentions arrive continuously and must be resolved against the existing canonical entity set. This article covers incremental resolution (resolving new entities without re-processing the entire corpus), canonical ID mapping (maintaining a stable identity layer across graph updates), alias management (preserving all surface forms for search), conflict resolution (handling contradictory merges), and monitoring (tracking resolution quality over time).

---

## Incremental Resolution Architecture

### Why Batch ER Is Not Enough

Batch entity resolution processes all entities at once during initial graph construction. This fails in production because:

1. New documents arrive continuously (daily, hourly, or real-time)
2. Re-running full ER on every ingestion is prohibitively expensive
3. Existing canonical IDs must remain stable (downstream systems reference them)
4. New entities may merge with existing ones, requiring graph updates

```
Initial Batch ER:
  10,000 mentions -> 7,500 canonical entities -> Knowledge Graph

Day 2: 500 new mentions arrive
  Option A (bad):  Re-run ER on 10,500 mentions -> expensive, IDs change
  Option B (good): Resolve 500 new mentions against 7,500 existing canonicals
```

### Incremental Resolution Pipeline

```python
import hashlib
import json
import time
from dataclasses import dataclass, field
from typing import Protocol


class EmbeddingModel(Protocol):
    def embed(self, texts: list[str]) -> list[list[float]]: ...


class VectorIndex(Protocol):
    def add(self, ids: list[str], embeddings: list[list[float]]) -> None: ...
    def search(self, embedding: list[float], top_k: int) -> list[tuple[str, float]]: ...
    def delete(self, ids: list[str]) -> None: ...


@dataclass
class CanonicalEntity:
    """A resolved canonical entity in the knowledge graph."""
    canonical_id: str
    canonical_name: str
    entity_type: str
    description: str
    aliases: list[str] = field(default_factory=list)
    source_mentions: list[str] = field(default_factory=list)
    embedding: list[float] = field(default_factory=list)
    created_at: float = 0.0
    updated_at: float = 0.0
    merge_count: int = 0


@dataclass
class EntityMention:
    """A new entity mention to be resolved."""
    mention_id: str
    name: str
    entity_type: str
    description: str
    context: str
    source_document: str


class IncrementalEntityResolver:
    """Resolve new entity mentions against existing canonical entities.

    Resolution flow for each new mention:
    1. Embed the new mention with context
    2. Search the canonical entity index for nearest neighbors
    3. Apply matching criteria (similarity + type + LLM if needed)
    4. If match found: merge into existing canonical entity
    5. If no match: create a new canonical entity

    The canonical entity index is updated after each resolution
    so subsequent mentions see the latest state.
    """

    def __init__(
        self,
        embedding_model: EmbeddingModel,
        vector_index: VectorIndex,
        entity_store: "CanonicalEntityStore",
        llm=None,
        similarity_threshold: float = 0.88,
        llm_threshold: float = 0.78,
    ):
        self.embedding_model = embedding_model
        self.vector_index = vector_index
        self.store = entity_store
        self.llm = llm
        self.sim_threshold = similarity_threshold
        self.llm_threshold = llm_threshold

    def resolve_batch(
        self, mentions: list[EntityMention]
    ) -> dict:
        """Resolve a batch of new entity mentions.

        Returns resolution statistics and the mapping of
        mention_id -> canonical_id.
        """
        stats = {
            "total_mentions": len(mentions),
            "merged": 0,
            "new_entities": 0,
            "llm_calls": 0,
        }
        mapping: dict[str, str] = {}

        for mention in mentions:
            result = self._resolve_single(mention)
            mapping[mention.mention_id] = result["canonical_id"]

            if result["action"] == "merged":
                stats["merged"] += 1
            elif result["action"] == "created":
                stats["new_entities"] += 1
            if result.get("llm_used"):
                stats["llm_calls"] += 1

        return {"mapping": mapping, "stats": stats}

    def _resolve_single(self, mention: EntityMention) -> dict:
        """Resolve a single entity mention."""
        # Step 1: Create contextual embedding
        embed_text = (
            f"{mention.name}. Type: {mention.entity_type}. "
            f"{mention.description} {mention.context[:200]}"
        )
        embedding = self.embedding_model.embed([embed_text])[0]

        # Step 2: Search for candidates in the canonical index
        candidates = self.vector_index.search(embedding, top_k=10)

        # Step 3: Filter by type and similarity
        best_match = None
        best_score = 0.0

        for canonical_id, similarity in candidates:
            if similarity < self.llm_threshold:
                continue

            canonical = self.store.get(canonical_id)
            if not canonical:
                continue

            # Type compatibility check
            if (
                canonical.entity_type
                and mention.entity_type
                and canonical.entity_type != mention.entity_type
            ):
                continue

            if similarity >= self.sim_threshold:
                if similarity > best_score:
                    best_match = canonical
                    best_score = similarity

            elif self.llm and similarity >= self.llm_threshold:
                # Borderline: use LLM to decide
                is_same = self._llm_judge(mention, canonical)
                if is_same:
                    if similarity > best_score:
                        best_match = canonical
                        best_score = similarity

        # Step 4: Merge or create
        if best_match:
            self._merge_into_canonical(mention, best_match, embedding)
            return {
                "canonical_id": best_match.canonical_id,
                "action": "merged",
                "similarity": best_score,
                "llm_used": best_score < self.sim_threshold,
            }
        else:
            canonical_id = self._create_canonical(mention, embedding)
            return {
                "canonical_id": canonical_id,
                "action": "created",
                "similarity": 0.0,
                "llm_used": False,
            }

    def _merge_into_canonical(
        self,
        mention: EntityMention,
        canonical: CanonicalEntity,
        new_embedding: list[float],
    ) -> None:
        """Merge a mention into an existing canonical entity."""
        # Add alias if the name is new
        if mention.name not in canonical.aliases and mention.name != canonical.canonical_name:
            canonical.aliases.append(mention.name)

        # Track source
        canonical.source_mentions.append(mention.mention_id)

        # Update description if the new one is more detailed
        if len(mention.description) > len(canonical.description):
            canonical.description = mention.description

        # Update embedding: weighted average (gives more weight to existing)
        old_weight = canonical.merge_count + 1
        new_weight = 1
        total_weight = old_weight + new_weight

        import numpy as np
        old_emb = np.array(canonical.embedding)
        new_emb = np.array(new_embedding)
        merged_emb = (old_emb * old_weight + new_emb * new_weight) / total_weight
        # Re-normalize
        merged_emb = merged_emb / max(np.linalg.norm(merged_emb), 1e-10)
        canonical.embedding = merged_emb.tolist()

        canonical.merge_count += 1
        canonical.updated_at = time.time()

        # Update stores
        self.store.update(canonical)
        self.vector_index.delete([canonical.canonical_id])
        self.vector_index.add([canonical.canonical_id], [canonical.embedding])

    def _create_canonical(
        self, mention: EntityMention, embedding: list[float]
    ) -> str:
        """Create a new canonical entity from a mention."""
        canonical_id = self._generate_id(mention.name, mention.entity_type)

        canonical = CanonicalEntity(
            canonical_id=canonical_id,
            canonical_name=mention.name,
            entity_type=mention.entity_type,
            description=mention.description,
            aliases=[],
            source_mentions=[mention.mention_id],
            embedding=embedding,
            created_at=time.time(),
            updated_at=time.time(),
            merge_count=0,
        )

        self.store.create(canonical)
        self.vector_index.add([canonical_id], [embedding])

        return canonical_id

    def _llm_judge(
        self, mention: EntityMention, canonical: CanonicalEntity
    ) -> bool:
        """Use LLM to judge whether a mention matches a canonical entity."""
        prompt = (
            "Do these refer to the same entity?\n\n"
            f"New mention: \"{mention.name}\" ({mention.entity_type})\n"
            f"  Context: {mention.context[:200]}\n\n"
            f"Existing entity: \"{canonical.canonical_name}\" ({canonical.entity_type})\n"
            f"  Description: {canonical.description[:200]}\n"
            f"  Also known as: {', '.join(canonical.aliases[:5])}\n\n"
            "Respond with YES or NO only."
        )
        response = self.llm.invoke(prompt)
        return "YES" in response.content.upper()

    @staticmethod
    def _generate_id(name: str, entity_type: str) -> str:
        """Generate a stable canonical ID from name and type."""
        content = f"{entity_type}::{name.lower().strip()}"
        return hashlib.sha256(content.encode()).hexdigest()[:16]
```

---

## Canonical Entity Store

```python
from pathlib import Path


class CanonicalEntityStore:
    """Persistent storage for canonical entities.

    In production, this would be backed by a database (PostgreSQL,
    Neo4j, or DynamoDB). This implementation uses a JSON file
    for simplicity but exposes the same interface.
    """

    def __init__(self, store_path: str):
        self.path = Path(store_path)
        self._data: dict[str, CanonicalEntity] = {}
        self._load()

    def _load(self) -> None:
        """Load from disk."""
        if self.path.exists():
            raw = json.loads(self.path.read_text())
            for cid, record in raw.items():
                self._data[cid] = CanonicalEntity(**record)

    def _save(self) -> None:
        """Persist to disk."""
        raw = {}
        for cid, entity in self._data.items():
            raw[cid] = {
                "canonical_id": entity.canonical_id,
                "canonical_name": entity.canonical_name,
                "entity_type": entity.entity_type,
                "description": entity.description,
                "aliases": entity.aliases,
                "source_mentions": entity.source_mentions,
                "embedding": entity.embedding,
                "created_at": entity.created_at,
                "updated_at": entity.updated_at,
                "merge_count": entity.merge_count,
            }
        self.path.write_text(json.dumps(raw, indent=2))

    def get(self, canonical_id: str) -> CanonicalEntity | None:
        return self._data.get(canonical_id)

    def create(self, entity: CanonicalEntity) -> None:
        self._data[entity.canonical_id] = entity
        self._save()

    def update(self, entity: CanonicalEntity) -> None:
        self._data[entity.canonical_id] = entity
        self._save()

    def delete(self, canonical_id: str) -> None:
        self._data.pop(canonical_id, None)
        self._save()

    def list_all(self) -> list[CanonicalEntity]:
        return list(self._data.values())

    def search_by_alias(self, alias: str) -> CanonicalEntity | None:
        """Find a canonical entity by any of its aliases."""
        alias_lower = alias.lower().strip()
        for entity in self._data.values():
            if entity.canonical_name.lower().strip() == alias_lower:
                return entity
            if any(a.lower().strip() == alias_lower for a in entity.aliases):
                return entity
        return None

    def get_stats(self) -> dict:
        """Return store statistics."""
        entities = list(self._data.values())
        return {
            "total_canonical_entities": len(entities),
            "total_aliases": sum(len(e.aliases) for e in entities),
            "avg_merge_count": (
                sum(e.merge_count for e in entities) / max(len(entities), 1)
            ),
            "entity_types": dict(
                sorted(
                    {
                        t: sum(1 for e in entities if e.entity_type == t)
                        for t in set(e.entity_type for e in entities)
                    }.items(),
                    key=lambda x: x[1],
                    reverse=True,
                )
            ),
        }
```

---

## Conflict Resolution

### Handling Contradictory Merges

```python
@dataclass
class MergeConflict:
    """A conflict detected during entity resolution."""
    entity_a_id: str
    entity_b_id: str
    conflict_type: str  # "type_mismatch", "description_contradiction", "cluster_size"
    details: str
    resolution: str = ""  # how it was resolved
    resolved: bool = False


class ConflictResolver:
    """Handle conflicts that arise during entity resolution.

    Common conflicts:
    1. Type mismatch: Two entities with same name but different types
    2. Description contradiction: Merged entities have contradictory info
    3. Cluster explosion: Too many entities merging into one canonical
    4. Circular merge: A previously split entity tries to re-merge
    """

    def __init__(self, max_cluster_size: int = 20, llm=None):
        self.max_cluster_size = max_cluster_size
        self.llm = llm
        self.conflicts: list[MergeConflict] = []

    def check_merge(
        self,
        mention: EntityMention,
        canonical: CanonicalEntity,
    ) -> MergeConflict | None:
        """Check for conflicts before merging."""
        # Check 1: Type mismatch
        if (
            mention.entity_type
            and canonical.entity_type
            and mention.entity_type != canonical.entity_type
        ):
            conflict = MergeConflict(
                entity_a_id=mention.mention_id,
                entity_b_id=canonical.canonical_id,
                conflict_type="type_mismatch",
                details=(
                    f"Mention type '{mention.entity_type}' differs from "
                    f"canonical type '{canonical.entity_type}'"
                ),
            )
            self.conflicts.append(conflict)
            return conflict

        # Check 2: Cluster size explosion
        if canonical.merge_count >= self.max_cluster_size:
            conflict = MergeConflict(
                entity_a_id=mention.mention_id,
                entity_b_id=canonical.canonical_id,
                conflict_type="cluster_size",
                details=(
                    f"Canonical entity already has {canonical.merge_count} merges, "
                    f"exceeds max of {self.max_cluster_size}"
                ),
            )
            self.conflicts.append(conflict)
            return conflict

        # Check 3: Description contradiction (if LLM available)
        if self.llm and mention.description and canonical.description:
            if self._descriptions_contradict(
                mention.description, canonical.description
            ):
                conflict = MergeConflict(
                    entity_a_id=mention.mention_id,
                    entity_b_id=canonical.canonical_id,
                    conflict_type="description_contradiction",
                    details="Entity descriptions contain contradictory information",
                )
                self.conflicts.append(conflict)
                return conflict

        return None

    def _descriptions_contradict(self, desc_a: str, desc_b: str) -> bool:
        """Use LLM to check if two descriptions are contradictory."""
        response = self.llm.invoke(
            "Do these two descriptions contradict each other?\n\n"
            f"Description A: {desc_a[:300]}\n"
            f"Description B: {desc_b[:300]}\n\n"
            "Respond YES if they contain contradictory facts, NO otherwise."
        )
        return "YES" in response.content.upper()

    def auto_resolve(self, conflict: MergeConflict) -> str:
        """Attempt to automatically resolve a conflict."""
        if conflict.conflict_type == "type_mismatch":
            conflict.resolution = "SKIP: Do not merge entities with different types"
            conflict.resolved = True
            return "skip"

        if conflict.conflict_type == "cluster_size":
            conflict.resolution = (
                "SKIP: Create new canonical entity instead of merging into "
                "oversized cluster"
            )
            conflict.resolved = True
            return "create_new"

        if conflict.conflict_type == "description_contradiction":
            conflict.resolution = (
                "FLAG: Merged but flagged for human review. "
                "Both descriptions retained."
            )
            conflict.resolved = True
            return "merge_with_flag"

        return "unresolved"

    def get_unresolved(self) -> list[MergeConflict]:
        """Get all unresolved conflicts for manual review."""
        return [c for c in self.conflicts if not c.resolved]

    def get_conflict_stats(self) -> dict:
        """Return conflict statistics."""
        total = len(self.conflicts)
        resolved = sum(1 for c in self.conflicts if c.resolved)
        by_type: dict[str, int] = {}
        for c in self.conflicts:
            by_type[c.conflict_type] = by_type.get(c.conflict_type, 0) + 1

        return {
            "total_conflicts": total,
            "resolved": resolved,
            "unresolved": total - resolved,
            "by_type": by_type,
        }
```

---

## Monitoring and Quality Tracking

```python
from datetime import datetime


class ERMonitor:
    """Monitor entity resolution quality over time.

    Tracks:
    - Resolution rate: what fraction of new mentions merge vs create
    - Cluster growth: are clusters growing too large?
    - Conflict rate: how many conflicts per batch?
    - Embedding drift: are entity embeddings shifting over time?
    """

    def __init__(self, store: CanonicalEntityStore):
        self.store = store
        self.history: list[dict] = []

    def record_batch(self, batch_stats: dict) -> None:
        """Record statistics from a resolution batch."""
        entry = {
            "timestamp": datetime.utcnow().isoformat(),
            "total_mentions": batch_stats["total_mentions"],
            "merged": batch_stats["merged"],
            "new_entities": batch_stats["new_entities"],
            "llm_calls": batch_stats.get("llm_calls", 0),
            "merge_rate": (
                batch_stats["merged"]
                / max(batch_stats["total_mentions"], 1)
            ),
        }
        self.history.append(entry)

    def check_health(self) -> dict:
        """Run health checks on the entity resolution system."""
        store_stats = self.store.get_stats()
        entities = self.store.list_all()

        issues = []

        # Check 1: Oversized clusters
        large_clusters = [
            e for e in entities if e.merge_count > 50
        ]
        if large_clusters:
            issues.append({
                "severity": "warning",
                "type": "large_clusters",
                "details": (
                    f"{len(large_clusters)} entities have >50 merges. "
                    f"Largest: '{large_clusters[0].canonical_name}' "
                    f"with {large_clusters[0].merge_count} merges."
                ),
            })

        # Check 2: Merge rate trend
        if len(self.history) >= 5:
            recent_rates = [h["merge_rate"] for h in self.history[-5:]]
            avg_rate = sum(recent_rates) / len(recent_rates)
            if avg_rate > 0.9:
                issues.append({
                    "severity": "warning",
                    "type": "high_merge_rate",
                    "details": (
                        f"Merge rate is {avg_rate:.1%} over last 5 batches. "
                        "This may indicate the similarity threshold is too low."
                    ),
                })
            elif avg_rate < 0.1:
                issues.append({
                    "severity": "info",
                    "type": "low_merge_rate",
                    "details": (
                        f"Merge rate is {avg_rate:.1%} over last 5 batches. "
                        "Most mentions are creating new entities."
                    ),
                })

        # Check 3: Orphan entities (created but never merged into)
        orphans = [e for e in entities if e.merge_count == 0]
        orphan_rate = len(orphans) / max(len(entities), 1)
        if orphan_rate > 0.7:
            issues.append({
                "severity": "info",
                "type": "many_orphans",
                "details": (
                    f"{orphan_rate:.0%} of canonical entities have no merges. "
                    "Consider lowering the similarity threshold."
                ),
            })

        return {
            "status": "unhealthy" if any(i["severity"] == "error" for i in issues) else "healthy",
            "store_stats": store_stats,
            "issues": issues,
            "history_length": len(self.history),
        }

    def generate_report(self) -> str:
        """Generate a human-readable resolution report."""
        store_stats = self.store.get_stats()
        health = self.check_health()

        lines = [
            "Entity Resolution Report",
            "=" * 40,
            f"Total canonical entities: {store_stats['total_canonical_entities']}",
            f"Total aliases: {store_stats['total_aliases']}",
            f"Average merge count: {store_stats['avg_merge_count']:.1f}",
            "",
            "Entity types:",
        ]

        for etype, count in store_stats["entity_types"].items():
            lines.append(f"  {etype}: {count}")

        lines.append("")
        lines.append(f"Health status: {health['status']}")

        if health["issues"]:
            lines.append("Issues:")
            for issue in health["issues"]:
                lines.append(f"  [{issue['severity']}] {issue['details']}")

        if self.history:
            lines.append("")
            lines.append("Recent batches:")
            for entry in self.history[-5:]:
                lines.append(
                    f"  {entry['timestamp']}: "
                    f"{entry['total_mentions']} mentions, "
                    f"{entry['merged']} merged, "
                    f"{entry['new_entities']} new "
                    f"(merge rate: {entry['merge_rate']:.0%})"
                )

        return "\n".join(lines)
```

---

## Graph Update After Resolution

```python
class GraphEntityUpdater:
    """Update the knowledge graph after entity resolution.

    When entities are merged, the graph needs:
    1. Redirect all relationships from old entity to canonical
    2. Merge node properties (descriptions, embeddings)
    3. Preserve aliases as searchable attributes
    4. Update community memberships
    """

    def __init__(self, neo4j_driver, database: str = "neo4j"):
        self.driver = neo4j_driver
        self.database = database

    def apply_merge(
        self, old_entity_name: str, canonical_entity_name: str
    ) -> dict:
        """Merge old_entity into canonical_entity in the graph."""
        with self.driver.session(database=self.database) as session:
            # Step 1: Move all relationships
            moved = session.run("""
                MATCH (old:Entity {name: $old_name})-[r]->(target)
                WHERE NOT target.name = $canonical_name
                MATCH (canonical:Entity {name: $canonical_name})
                CREATE (canonical)-[new_r:RELATED_TO]->(target)
                SET new_r = properties(r)
                DELETE r
                RETURN count(new_r) AS moved_out
            """, old_name=old_entity_name, canonical_name=canonical_entity_name).single()

            moved_in = session.run("""
                MATCH (source)-[r]->(old:Entity {name: $old_name})
                WHERE NOT source.name = $canonical_name
                MATCH (canonical:Entity {name: $canonical_name})
                CREATE (source)-[new_r:RELATED_TO]->(canonical)
                SET new_r = properties(r)
                DELETE r
                RETURN count(new_r) AS moved_in
            """, old_name=old_entity_name, canonical_name=canonical_entity_name).single()

            # Step 2: Add alias
            session.run("""
                MATCH (canonical:Entity {name: $canonical_name})
                SET canonical.aliases = coalesce(canonical.aliases, []) + [$old_name]
            """, canonical_name=canonical_entity_name, old_name=old_entity_name)

            # Step 3: Move source chunk links
            session.run("""
                MATCH (old:Entity {name: $old_name})-[r:MENTIONED_IN]->(chunk:Chunk)
                MATCH (canonical:Entity {name: $canonical_name})
                MERGE (canonical)-[:MENTIONED_IN]->(chunk)
                DELETE r
            """, old_name=old_entity_name, canonical_name=canonical_entity_name)

            # Step 4: Delete old entity
            session.run("""
                MATCH (old:Entity {name: $old_name})
                DETACH DELETE old
            """, old_name=old_entity_name)

            return {
                "merged": f"{old_entity_name} -> {canonical_entity_name}",
                "relationships_moved": (
                    (moved["moved_out"] if moved else 0)
                    + (moved_in["moved_in"] if moved_in else 0)
                ),
            }

    def apply_batch_merges(
        self, merges: list[tuple[str, str]]
    ) -> dict:
        """Apply multiple entity merges efficiently."""
        total_moved = 0
        for old_name, canonical_name in merges:
            result = self.apply_merge(old_name, canonical_name)
            total_moved += result["relationships_moved"]

        return {
            "merges_applied": len(merges),
            "total_relationships_moved": total_moved,
        }
```

---

## Common Pitfalls

1. **Changing canonical IDs after downstream systems depend on them.** Once a canonical ID is published, it must remain stable. Use the alias system instead of renaming.

2. **Not averaging embeddings after merges.** After merging 10 mentions into one canonical entity, the embedding should reflect all mentions, not just the first one.

3. **Ignoring cluster explosions.** If one canonical entity accumulates 200+ merges, something is wrong (threshold too low, or a very common term like "system" is being over-merged). Set max_cluster_size limits.

4. **Running incremental resolution without the vector index.** Linear scanning of all canonicals for each new mention is O(n). Use a vector index (FAISS, Qdrant, Pinecone) for O(log n) lookups.

5. **Not monitoring merge rate trends.** A sudden spike in merge rate may indicate a corpus shift or threshold drift. Track and alert on merge rate changes.

6. **Forgetting to update graph relationships.** Merging entities in the canonical store without redirecting graph edges creates orphan relationships pointing to deleted nodes.

---

## References

- Entity resolution in knowledge graphs: Christophides et al. "End-to-End Entity Resolution for Big Data" (2021)
- Splink: https://moj-analytical-services.github.io/splink/
- FAISS: https://github.com/facebookresearch/faiss
- Neo4j APOC merge procedures: https://neo4j.com/labs/apoc/
