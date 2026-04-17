# RAG Production Indexing -- Incremental, Blue-Green Re-Indexing, and Alias Swapping

## TL;DR

Production RAG systems must keep their vector index synchronized with changing source documents without downtime or serving stale data. Three patterns address this: incremental indexing (processing only changed documents using content hashes and change detection), blue-green re-indexing (building a complete new index alongside the live one, then swapping atomically), and alias swapping (using index aliases so the application always points to a named alias while the underlying index is replaced). This article covers implementation of all three patterns with content-hash-based change detection, Elasticsearch/OpenSearch and Pinecone alias management, and production scheduling strategies.

---

## Incremental Indexing

### Concept

Instead of reindexing the entire corpus, detect which documents have changed since the last indexing run and process only those.

### Change Detection with Content Hashing

```python
import hashlib
import json
import time
from dataclasses import dataclass, field
from pathlib import Path

import redis


@dataclass
class DocumentChange:
    doc_id: str
    change_type: str  # "added", "modified", "deleted"
    content_hash: str
    old_hash: str | None = None


class ContentHashTracker:
    """Track document content hashes to detect changes.

    Stores SHA-256 hashes of document content in Redis.
    On each indexing run, compares current hashes to stored
    hashes to determine which documents need reprocessing.
    """

    def __init__(
        self,
        redis_client: redis.Redis,
        prefix: str = "doc_hash:",
    ):
        self.redis = redis_client
        self.prefix = prefix

    def compute_hash(self, content: str) -> str:
        """Compute SHA-256 hash of document content."""
        return hashlib.sha256(content.encode("utf-8")).hexdigest()

    def detect_changes(
        self, documents: list[dict]
    ) -> list[DocumentChange]:
        """Compare current documents against stored hashes.

        Args:
            documents: list of {"doc_id": str, "content": str}

        Returns:
            List of changes (added, modified, deleted)
        """
        changes = []
        current_ids = set()

        for doc in documents:
            doc_id = doc["doc_id"]
            current_ids.add(doc_id)
            new_hash = self.compute_hash(doc["content"])

            key = f"{self.prefix}{doc_id}"
            old_hash = self.redis.get(key)

            if old_hash is None:
                changes.append(DocumentChange(
                    doc_id=doc_id,
                    change_type="added",
                    content_hash=new_hash,
                ))
            elif old_hash.decode() != new_hash:
                changes.append(DocumentChange(
                    doc_id=doc_id,
                    change_type="modified",
                    content_hash=new_hash,
                    old_hash=old_hash.decode(),
                ))
            # else: unchanged, skip

        # Detect deletions
        stored_keys = self.redis.keys(f"{self.prefix}*")
        for key in stored_keys:
            doc_id = key.decode().removeprefix(self.prefix)
            if doc_id not in current_ids:
                changes.append(DocumentChange(
                    doc_id=doc_id,
                    change_type="deleted",
                    content_hash="",
                ))

        return changes

    def update_hashes(self, changes: list[DocumentChange]) -> None:
        """Update stored hashes after successful indexing."""
        pipe = self.redis.pipeline()
        for change in changes:
            key = f"{self.prefix}{change.doc_id}"
            if change.change_type in ("added", "modified"):
                pipe.set(key, change.content_hash)
            elif change.change_type == "deleted":
                pipe.delete(key)
        pipe.execute()
```

### Incremental Indexing Pipeline

```python
import logging
from datetime import datetime

logger = logging.getLogger(__name__)


class IncrementalIndexer:
    """Process only changed documents during each indexing run.

    Steps:
    1. Scan source documents for changes (hash comparison)
    2. Chunk only changed documents
    3. Delete old vectors for modified/deleted documents
    4. Insert new vectors for added/modified documents
    5. Update hash tracker
    """

    def __init__(
        self,
        vector_store,
        embedding_fn,
        chunker,
        hash_tracker: ContentHashTracker,
    ):
        self.vector_store = vector_store
        self.embed = embedding_fn
        self.chunker = chunker
        self.tracker = hash_tracker

    def run(self, documents: list[dict]) -> dict:
        """Execute an incremental indexing run."""
        start_time = time.time()

        # Step 1: Detect changes
        changes = self.tracker.detect_changes(documents)

        if not changes:
            logger.info("No changes detected, skipping indexing")
            return {
                "status": "no_changes",
                "duration_seconds": 0,
            }

        logger.info(
            "Detected %d changes: %d added, %d modified, %d deleted",
            len(changes),
            sum(1 for c in changes if c.change_type == "added"),
            sum(1 for c in changes if c.change_type == "modified"),
            sum(1 for c in changes if c.change_type == "deleted"),
        )

        stats = {"added": 0, "modified": 0, "deleted": 0, "chunks_created": 0}

        # Step 2: Process deletions and modifications (remove old vectors)
        ids_to_delete = [
            c.doc_id
            for c in changes
            if c.change_type in ("deleted", "modified")
        ]
        if ids_to_delete:
            self._delete_vectors(ids_to_delete)
            stats["deleted"] = sum(
                1 for c in changes if c.change_type == "deleted"
            )

        # Step 3: Process additions and modifications (add new vectors)
        docs_to_index = []
        doc_map = {d["doc_id"]: d for d in documents}
        for change in changes:
            if change.change_type in ("added", "modified"):
                doc = doc_map.get(change.doc_id)
                if doc:
                    docs_to_index.append(doc)

        if docs_to_index:
            chunks_created = self._index_documents(docs_to_index)
            stats["added"] = sum(
                1 for c in changes if c.change_type == "added"
            )
            stats["modified"] = sum(
                1 for c in changes if c.change_type == "modified"
            )
            stats["chunks_created"] = chunks_created

        # Step 4: Update hash tracker
        self.tracker.update_hashes(changes)

        duration = time.time() - start_time
        logger.info(
            "Incremental indexing complete in %.1fs: %s",
            duration,
            stats,
        )

        return {
            "status": "completed",
            "duration_seconds": duration,
            "stats": stats,
        }

    def _delete_vectors(self, doc_ids: list[str]) -> None:
        """Delete all vectors associated with given document IDs."""
        for doc_id in doc_ids:
            # Delete by metadata filter (doc_id field)
            self.vector_store.delete(
                filter={"doc_id": {"$eq": doc_id}}
            )

    def _index_documents(self, documents: list[dict]) -> int:
        """Chunk, embed, and insert documents."""
        all_chunks = []
        for doc in documents:
            chunks = self.chunker.split(doc["content"])
            for i, chunk in enumerate(chunks):
                all_chunks.append({
                    "content": chunk,
                    "metadata": {
                        "doc_id": doc["doc_id"],
                        "chunk_index": i,
                        "source": doc.get("source", ""),
                        "indexed_at": datetime.utcnow().isoformat(),
                    },
                })

        if all_chunks:
            texts = [c["content"] for c in all_chunks]
            metadatas = [c["metadata"] for c in all_chunks]
            self.vector_store.add_texts(texts, metadatas=metadatas)

        return len(all_chunks)
```

---

## Blue-Green Re-Indexing

### Concept

Maintain two index instances: "blue" (live, serving traffic) and "green" (standby, being rebuilt). When green is ready, swap traffic atomically. This provides zero-downtime reindexing and instant rollback.

```
Time 0:  [blue: LIVE]  [green: idle]
Time 1:  [blue: LIVE]  [green: rebuilding...]
Time 2:  [blue: LIVE]  [green: ready, validated]
Time 3:  [blue: idle]  [green: LIVE]  <- swap
Rollback: [blue: LIVE]  [green: idle]  <- swap back
```

### Implementation

```python
from enum import Enum


class IndexColor(Enum):
    BLUE = "blue"
    GREEN = "green"


class BlueGreenIndexManager:
    """Manage blue-green index deployments for zero-downtime reindexing.

    The application always reads from the 'active' index.
    Reindexing builds the 'standby' index, validates it,
    then swaps the active pointer.
    """

    def __init__(
        self,
        create_index_fn,
        delete_index_fn,
        redis_client: redis.Redis,
        index_prefix: str = "rag",
    ):
        self.create_index = create_index_fn
        self.delete_index = delete_index_fn
        self.redis = redis_client
        self.prefix = index_prefix

    def get_active_color(self) -> IndexColor:
        """Get the currently active index color."""
        color = self.redis.get(f"{self.prefix}:active_color")
        if color is None:
            self.redis.set(
                f"{self.prefix}:active_color", IndexColor.BLUE.value
            )
            return IndexColor.BLUE
        return IndexColor(color.decode())

    def get_standby_color(self) -> IndexColor:
        """Get the standby (not active) index color."""
        active = self.get_active_color()
        return (
            IndexColor.GREEN
            if active == IndexColor.BLUE
            else IndexColor.BLUE
        )

    def get_index_name(self, color: IndexColor) -> str:
        """Get the full index name for a color."""
        return f"{self.prefix}_{color.value}"

    def get_active_index_name(self) -> str:
        return self.get_index_name(self.get_active_color())

    def rebuild_standby(
        self,
        documents: list[dict],
        indexer_fn,
        validator_fn,
    ) -> dict:
        """Rebuild the standby index with fresh data.

        Steps:
        1. Drop the standby index
        2. Create a fresh standby index
        3. Index all documents into standby
        4. Validate the standby index
        5. Swap if validation passes
        """
        standby = self.get_standby_color()
        standby_name = self.get_index_name(standby)

        logger.info("Rebuilding standby index: %s", standby_name)

        # Step 1: Drop old standby
        try:
            self.delete_index(standby_name)
        except Exception:
            pass  # Index may not exist

        # Step 2: Create fresh index
        self.create_index(standby_name)

        # Step 3: Index all documents
        start = time.time()
        indexer_fn(standby_name, documents)
        index_duration = time.time() - start

        # Step 4: Validate
        validation = validator_fn(standby_name)

        result = {
            "standby_index": standby_name,
            "index_duration_seconds": index_duration,
            "validation": validation,
            "swapped": False,
        }

        if validation["passed"]:
            # Step 5: Swap
            self.swap()
            result["swapped"] = True
            logger.info("Swapped active index to %s", standby_name)
        else:
            logger.error(
                "Validation failed, not swapping: %s",
                validation,
            )

        return result

    def swap(self) -> dict:
        """Swap active and standby indexes."""
        old_active = self.get_active_color()
        new_active = self.get_standby_color()

        self.redis.set(
            f"{self.prefix}:active_color", new_active.value
        )
        self.redis.set(
            f"{self.prefix}:last_swap_time", str(time.time())
        )
        self.redis.set(
            f"{self.prefix}:previous_active", old_active.value
        )

        return {
            "previous_active": old_active.value,
            "new_active": new_active.value,
        }

    def rollback(self) -> dict:
        """Rollback to the previous active index."""
        previous = self.redis.get(f"{self.prefix}:previous_active")
        if previous:
            self.redis.set(
                f"{self.prefix}:active_color", previous.decode()
            )
            return {"rolled_back_to": previous.decode()}
        return {"error": "No previous active index to rollback to"}
```

### Index Validation

```python
class IndexValidator:
    """Validate a newly built index before swapping it to active."""

    def __init__(
        self,
        min_doc_count: int = 100,
        test_queries: list[dict] | None = None,
        max_latency_ms: float = 500.0,
    ):
        self.min_doc_count = min_doc_count
        self.test_queries = test_queries or []
        self.max_latency_ms = max_latency_ms

    def validate(
        self, index_name: str, query_fn
    ) -> dict:
        """Run validation checks on a new index."""
        checks = {}

        # Check 1: Document count
        doc_count = query_fn(index_name, "count")
        checks["doc_count"] = {
            "passed": doc_count >= self.min_doc_count,
            "actual": doc_count,
            "minimum": self.min_doc_count,
        }

        # Check 2: Test queries return results
        query_checks = []
        for tq in self.test_queries:
            start = time.perf_counter()
            results = query_fn(
                index_name, "search", tq["query"]
            )
            latency = (time.perf_counter() - start) * 1000

            passed = (
                len(results) > 0
                and latency < self.max_latency_ms
            )
            query_checks.append({
                "query": tq["query"],
                "result_count": len(results),
                "latency_ms": latency,
                "passed": passed,
            })

        checks["test_queries"] = {
            "passed": all(q["passed"] for q in query_checks),
            "details": query_checks,
        }

        all_passed = all(c["passed"] for c in checks.values())
        return {"passed": all_passed, "checks": checks}
```

---

## Alias Swapping

### Elasticsearch/OpenSearch Alias Pattern

```python
from opensearchpy import OpenSearch


class ElasticsearchAliasManager:
    """Manage Elasticsearch index aliases for zero-downtime swapping.

    The application always queries through the alias name.
    During reindexing, a new index is built and the alias
    is atomically swapped to point to it.
    """

    def __init__(
        self,
        client: OpenSearch,
        alias_name: str = "rag_index",
    ):
        self.client = client
        self.alias = alias_name

    def get_current_index(self) -> str | None:
        """Get the index currently pointed to by the alias."""
        try:
            alias_info = self.client.indices.get_alias(name=self.alias)
            return list(alias_info.keys())[0] if alias_info else None
        except Exception:
            return None

    def create_versioned_index(
        self, version: str, settings: dict | None = None
    ) -> str:
        """Create a new versioned index."""
        index_name = f"{self.alias}_{version}"
        body = settings or {
            "settings": {
                "number_of_shards": 2,
                "number_of_replicas": 1,
            },
        }
        self.client.indices.create(index=index_name, body=body)
        return index_name

    def swap_alias(
        self, new_index: str, delete_old: bool = False
    ) -> dict:
        """Atomically swap the alias to point to a new index.

        This is an atomic operation -- queries never see an
        intermediate state.
        """
        old_index = self.get_current_index()

        actions = []
        if old_index:
            actions.append({
                "remove": {"index": old_index, "alias": self.alias}
            })
        actions.append({
            "add": {"index": new_index, "alias": self.alias}
        })

        self.client.indices.update_aliases(
            body={"actions": actions}
        )

        result = {
            "old_index": old_index,
            "new_index": new_index,
            "alias": self.alias,
        }

        if delete_old and old_index:
            self.client.indices.delete(index=old_index)
            result["old_index_deleted"] = True

        return result
```

### Pinecone Index Alias Pattern

```python
from pinecone import Pinecone


class PineconeCollectionSwapper:
    """Manage Pinecone index swapping using collections.

    Pinecone does not have native aliases, so we use collections
    as snapshots and recreate indexes from them for rollback.
    """

    def __init__(
        self,
        api_key: str,
        index_name: str = "rag-production",
    ):
        self.pc = Pinecone(api_key=api_key)
        self.index_name = index_name

    def snapshot_current(self, snapshot_name: str) -> dict:
        """Create a collection from the current index (snapshot)."""
        self.pc.create_collection(
            name=snapshot_name,
            source=self.index_name,
        )
        return {
            "snapshot": snapshot_name,
            "source_index": self.index_name,
        }

    def reindex_in_place(
        self, documents: list[dict], embedding_fn
    ) -> dict:
        """Clear and reindex the current index.

        For Pinecone, this deletes all vectors and re-inserts.
        A collection snapshot provides rollback capability.
        """
        index = self.pc.Index(self.index_name)

        # Delete all existing vectors
        index.delete(delete_all=True)

        # Reindex in batches
        batch_size = 100
        total_indexed = 0

        for i in range(0, len(documents), batch_size):
            batch = documents[i : i + batch_size]
            vectors = []
            for doc in batch:
                embedding = embedding_fn(doc["content"])
                vectors.append({
                    "id": doc["doc_id"],
                    "values": embedding,
                    "metadata": doc.get("metadata", {}),
                })
            index.upsert(vectors=vectors)
            total_indexed += len(vectors)

        return {"total_indexed": total_indexed}

    def rollback_from_snapshot(self, snapshot_name: str) -> dict:
        """Restore the index from a collection snapshot."""
        # Delete current index
        self.pc.delete_index(self.index_name)

        # Recreate from collection
        self.pc.create_index(
            name=self.index_name,
            dimension=1536,
            metric="cosine",
            source_collection=snapshot_name,
        )

        return {"restored_from": snapshot_name}
```

---

## Scheduling Strategies

### Incremental vs Full Reindex Schedule

```python
from dataclasses import dataclass


@dataclass
class IndexingSchedule:
    """Define when incremental and full reindexes should run."""

    # Incremental: process only changes
    incremental_interval_minutes: int = 15
    # Full reindex: rebuild from scratch (catches drift)
    full_reindex_interval_hours: int = 24
    # Full reindex during low-traffic window
    full_reindex_preferred_hour_utc: int = 3  # 3 AM UTC


# Typical schedule:
# - Every 15 min: incremental indexing (fast, processes ~10-100 docs)
# - Daily at 3 AM: full reindex with blue-green swap (catches any drift)
# - On-demand: manual reindex via admin API
```

---

## Common Pitfalls

1. **Deleting vectors by content instead of by document ID.** When a document is modified, you must delete all its old chunk vectors before inserting new ones. Without a doc_id metadata field, you cannot identify which vectors belong to which document.
2. **Not validating the new index before swapping.** A reindex that encounters silent errors may produce an index with missing documents. Always run validation queries before swapping.
3. **Blue-green without enough resources.** During reindexing, both indexes exist simultaneously, doubling storage and memory requirements. Budget for 2x resources.
4. **No rollback mechanism.** If the new index has a subtle quality regression not caught by validation, you need to swap back immediately. Always keep the previous index available for at least 24 hours.
5. **Running full reindex during peak traffic.** Full reindexing is resource-intensive. Schedule it during low-traffic windows and use incremental indexing during peak hours.
6. **Not tracking which documents failed to index.** If 3 out of 10,000 documents fail to embed due to content issues, those failures are silently lost. Log every failure and surface it in monitoring.

---

## References

- Elasticsearch index aliases: https://www.elastic.co/guide/en/elasticsearch/reference/current/indices-aliases.html
- Pinecone collections: https://docs.pinecone.io/guides/data/understanding-collections
- Weaviate backup and restore: https://weaviate.io/developers/weaviate/configuration/backups
