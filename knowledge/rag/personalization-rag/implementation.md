# Personalization RAG -- Implementation Patterns

## TL;DR

This guide provides production-ready implementation patterns for personalized RAG: per-user namespace filtering in vector stores (Qdrant, Pinecone, ChromaDB), preference-aware reranking that boosts results matching user context, privacy-preserving personalization techniques that work under GDPR and CCPA constraints, and a complete end-to-end pipeline that ties retrieval filtering, memory injection, and generation adaptation together. Each pattern includes working Python code with inline explanations of the design decisions. The goal is to take the concepts from the overview and memory-systems guides and show exactly how to wire them into a production pipeline.

---

## Per-User Namespace Filtering

### Why Namespace Filtering Matters

The simplest personalization technique is filtering retrieved documents to match the user's context. This eliminates obviously irrelevant results before they consume context window budget.

```
Without namespace filtering:
  Query: "How to set up CI?"
  Retrieved: [Jenkins guide, GitHub Actions guide, GitLab CI guide,
              CircleCI guide, Azure Pipelines guide]
  --> User uses GitHub Actions; 4 of 5 results are noise

With namespace filtering:
  Query: "How to set up CI?" + filter: technology=github-actions
  Retrieved: [GH Actions basics, GH Actions caching, GH Actions matrix,
              GH Actions secrets, GH Actions reusable workflows]
  --> All 5 results are directly relevant
```

### Qdrant Implementation

```python
"""
Per-user namespace filtering with Qdrant.
"""
from dataclasses import dataclass, field

from qdrant_client import QdrantClient
from qdrant_client.models import (
    Distance,
    FieldCondition,
    Filter,
    MatchAny,
    MatchValue,
    PointStruct,
    VectorParams,
)


@dataclass
class UserContext:
    """User context for filtering retrieval."""
    user_id: str
    team: str = ""
    tech_stack: list[str] = field(default_factory=list)
    expertise_level: str = "intermediate"
    product_tier: str = ""  # For SaaS: "free", "pro", "enterprise"
    access_groups: list[str] = field(default_factory=list)


class QdrantPersonalizedRetriever:
    """
    Personalized retrieval using Qdrant metadata filtering.

    Documents are indexed with metadata (technology, team, difficulty,
    access_level) that enables per-user filtering at query time.
    """

    def __init__(
        self,
        client: QdrantClient,
        collection_name: str,
        embedding_fn,
    ):
        self.client = client
        self.collection = collection_name
        self.embed = embedding_fn

    def index_document(
        self,
        doc_id: str,
        text: str,
        metadata: dict,
    ) -> None:
        """
        Index a document with personalization-relevant metadata.

        Required metadata fields:
        - technology: str (e.g., "python", "react")
        - difficulty: str ("beginner", "intermediate", "advanced")
        - team: str (optional, for team-specific docs)
        - access_level: str ("public", "internal", "restricted")
        """
        embedding = self.embed(text)

        self.client.upsert(
            collection_name=self.collection,
            points=[PointStruct(
                id=doc_id,
                vector=embedding.tolist(),
                payload={
                    "text": text,
                    **metadata,
                },
            )],
        )

    def retrieve(
        self,
        query: str,
        user: UserContext,
        k: int = 10,
        strategy: str = "soft",
    ) -> list[dict]:
        """
        Retrieve documents with personalized filtering.

        Strategies:
        - "hard": strict filtering, only matching docs returned
        - "soft": prefer matching docs, fall back to all docs
        - "boost": retrieve all, boost matching docs in scoring
        """
        query_vector = self.embed(query).tolist()

        if strategy == "hard":
            return self._hard_filter_retrieve(query_vector, user, k)
        elif strategy == "soft":
            return self._soft_filter_retrieve(query_vector, user, k)
        else:
            return self._boost_retrieve(query_vector, user, k)

    def _hard_filter_retrieve(
        self,
        query_vector: list[float],
        user: UserContext,
        k: int,
    ) -> list[dict]:
        """Strict filtering: only return docs matching user context."""
        conditions = []

        # Access control (always enforce)
        allowed_levels = ["public"]
        if user.access_groups:
            allowed_levels.append("internal")
        conditions.append(
            FieldCondition(
                key="access_level",
                match=MatchAny(any=allowed_levels),
            )
        )

        # Tech stack filter (if user has a defined stack)
        if user.tech_stack:
            conditions.append(
                FieldCondition(
                    key="technology",
                    match=MatchAny(any=user.tech_stack),
                )
            )

        results = self.client.search(
            collection_name=self.collection,
            query_vector=query_vector,
            query_filter=Filter(must=conditions),
            limit=k,
        )

        return [
            {
                "text": r.payload["text"],
                "score": r.score,
                "metadata": {
                    k: v for k, v in r.payload.items() if k != "text"
                },
            }
            for r in results
        ]

    def _soft_filter_retrieve(
        self,
        query_vector: list[float],
        user: UserContext,
        k: int,
    ) -> list[dict]:
        """
        Soft filtering: try filtered first, fall back to unfiltered.

        This prevents empty results when the filter is too restrictive.
        """
        # Try with filters
        filtered_results = self._hard_filter_retrieve(
            query_vector, user, k
        )

        if len(filtered_results) >= k // 2:
            return filtered_results

        # Fall back to unfiltered (but still respect access control)
        access_filter = Filter(must=[
            FieldCondition(
                key="access_level",
                match=MatchAny(any=["public", "internal"]),
            ),
        ])

        results = self.client.search(
            collection_name=self.collection,
            query_vector=query_vector,
            query_filter=access_filter,
            limit=k,
        )

        # Merge: filtered results first, then unfiltered
        seen_ids = {r.get("id") for r in filtered_results}
        for r in results:
            if len(filtered_results) >= k:
                break
            if r.id not in seen_ids:
                filtered_results.append({
                    "text": r.payload["text"],
                    "score": r.score,
                    "metadata": {
                        k_: v for k_, v in r.payload.items() if k_ != "text"
                    },
                })

        return filtered_results[:k]

    def _boost_retrieve(
        self,
        query_vector: list[float],
        user: UserContext,
        k: int,
    ) -> list[dict]:
        """
        Boost strategy: retrieve all, then rerank with user preference boost.

        This gives the best recall while still preferring personalized results.
        """
        # Retrieve more than needed (we will rerank)
        results = self.client.search(
            collection_name=self.collection,
            query_vector=query_vector,
            limit=k * 3,
        )

        # Apply personalization boost
        boosted = []
        for r in results:
            base_score = r.score
            boost = 0.0

            # Tech stack match
            doc_tech = r.payload.get("technology", "")
            if doc_tech in user.tech_stack:
                boost += 0.15

            # Expertise level match
            doc_diff = r.payload.get("difficulty", "intermediate")
            if doc_diff == user.expertise_level:
                boost += 0.08

            # Team match
            doc_team = r.payload.get("team", "")
            if doc_team == user.team:
                boost += 0.10

            boosted.append({
                "text": r.payload["text"],
                "score": base_score + boost,
                "base_score": base_score,
                "boost": boost,
                "metadata": {
                    k_: v for k_, v in r.payload.items() if k_ != "text"
                },
            })

        boosted.sort(key=lambda x: x["score"], reverse=True)
        return boosted[:k]
```

### ChromaDB Implementation

```python
"""
Per-user namespace filtering with ChromaDB.
"""
import chromadb


class ChromaPersonalizedRetriever:
    """Personalized retrieval using ChromaDB where filters."""

    def __init__(
        self,
        collection_name: str,
        persist_dir: str = "./chroma_db",
    ):
        self.client = chromadb.PersistentClient(path=persist_dir)
        self.collection = self.client.get_or_create_collection(
            name=collection_name,
            metadata={"hnsw:space": "cosine"},
        )

    def retrieve(
        self,
        query: str,
        user: "UserContext",
        k: int = 10,
    ) -> list[dict]:
        """Retrieve with ChromaDB metadata filtering."""
        where_filter = self._build_where_filter(user)

        try:
            results = self.collection.query(
                query_texts=[query],
                n_results=k,
                where=where_filter if where_filter else None,
            )
        except Exception:
            # Fallback: no filter if the filter causes errors
            results = self.collection.query(
                query_texts=[query],
                n_results=k,
            )

        documents = results.get("documents", [[]])[0]
        metadatas = results.get("metadatas", [[]])[0]
        distances = results.get("distances", [[]])[0]

        return [
            {
                "text": doc,
                "score": 1 - dist,  # ChromaDB returns distances
                "metadata": meta,
            }
            for doc, meta, dist in zip(documents, metadatas, distances)
        ]

    def _build_where_filter(self, user: "UserContext") -> dict | None:
        """Build ChromaDB where filter from user context."""
        conditions = []

        if user.tech_stack:
            conditions.append({
                "technology": {"$in": user.tech_stack},
            })

        if user.expertise_level == "beginner":
            conditions.append({
                "difficulty": {"$in": ["beginner", "intermediate"]},
            })

        if len(conditions) == 0:
            return None
        elif len(conditions) == 1:
            return conditions[0]
        else:
            return {"$and": conditions}
```

---

## Preference-Aware Reranking

### Cross-Encoder Reranking with User Signals

```python
"""
Preference-aware reranking: combine relevance scoring with user preferences.
"""
import numpy as np
from sentence_transformers import CrossEncoder


class PreferenceAwareReranker:
    """
    Rerank retrieved documents using both relevance and user preferences.

    Combines cross-encoder relevance scores with preference signals
    to produce a personalized ranking.
    """

    def __init__(
        self,
        cross_encoder_model: str = "cross-encoder/ms-marco-MiniLM-L-12-v2",
        relevance_weight: float = 0.70,
        preference_weight: float = 0.30,
    ):
        self.cross_encoder = CrossEncoder(cross_encoder_model)
        self.relevance_weight = relevance_weight
        self.preference_weight = preference_weight

    def rerank(
        self,
        query: str,
        documents: list[dict],
        user: "UserContext",
        user_memories: list[dict] | None = None,
        top_k: int = 5,
    ) -> list[dict]:
        """
        Rerank documents with preference-aware scoring.

        Final score = relevance_weight * relevance_score
                    + preference_weight * preference_score
        """
        if not documents:
            return []

        # Cross-encoder relevance scores
        pairs = [(query, doc["text"]) for doc in documents]
        relevance_scores = self.cross_encoder.predict(pairs)

        # Normalize relevance scores to [0, 1]
        if len(relevance_scores) > 1:
            r_min = min(relevance_scores)
            r_max = max(relevance_scores)
            r_range = r_max - r_min if r_max > r_min else 1.0
            relevance_normalized = [
                (s - r_min) / r_range for s in relevance_scores
            ]
        else:
            relevance_normalized = [0.5]

        # Preference scores
        preference_scores = [
            self._compute_preference_score(doc, user, user_memories)
            for doc in documents
        ]

        # Combined scoring
        combined = []
        for i, doc in enumerate(documents):
            final_score = (
                self.relevance_weight * relevance_normalized[i]
                + self.preference_weight * preference_scores[i]
            )

            combined.append({
                **doc,
                "final_score": float(final_score),
                "relevance_score": float(relevance_normalized[i]),
                "preference_score": float(preference_scores[i]),
            })

        combined.sort(key=lambda x: x["final_score"], reverse=True)
        return combined[:top_k]

    def _compute_preference_score(
        self,
        doc: dict,
        user: "UserContext",
        user_memories: list[dict] | None = None,
    ) -> float:
        """
        Compute a preference score for a document given user context.

        Factors:
        1. Tech stack alignment (does the doc match user's tech?)
        2. Difficulty alignment (does the doc match user's level?)
        3. Team relevance (is this doc from user's team?)
        4. Historical preference (has the user liked similar docs?)
        """
        score = 0.0
        factors = 0

        metadata = doc.get("metadata", {})

        # Tech stack alignment (0 or 1)
        doc_tech = metadata.get("technology", "")
        if doc_tech and user.tech_stack:
            if doc_tech in user.tech_stack:
                score += 1.0
            factors += 1

        # Difficulty alignment (0 to 1)
        doc_diff = metadata.get("difficulty", "")
        if doc_diff and user.expertise_level:
            diff_map = {"beginner": 0, "intermediate": 1, "advanced": 2}
            user_level = diff_map.get(user.expertise_level, 1)
            doc_level = diff_map.get(doc_diff, 1)
            distance = abs(user_level - doc_level)
            score += max(0, 1 - distance * 0.5)
            factors += 1

        # Team relevance
        doc_team = metadata.get("team", "")
        if doc_team and user.team:
            if doc_team == user.team:
                score += 1.0
            factors += 1

        # Memory-based preference
        if user_memories:
            memory_boost = self._memory_preference_score(doc, user_memories)
            score += memory_boost
            factors += 1

        return score / max(factors, 1)

    def _memory_preference_score(
        self,
        doc: dict,
        user_memories: list[dict],
    ) -> float:
        """
        Score based on user's past interactions.

        If the user has previously engaged with similar content
        (same topic, same source), boost this document.
        """
        doc_text = doc.get("text", "").lower()
        doc_words = set(doc_text.split()[:50])

        max_overlap = 0
        for mem in user_memories:
            mem_text = mem.get("content", "").lower()
            mem_words = set(mem_text.split())
            if mem_words:
                overlap = len(doc_words & mem_words) / max(len(mem_words), 1)
                max_overlap = max(max_overlap, overlap)

        return min(max_overlap, 1.0)
```

---

## Privacy-Preserving Personalization

### Differential Privacy for User Profiles

```python
"""
Privacy-preserving personalization techniques.
"""
import hashlib
import json
import os
from datetime import datetime, timedelta
from pathlib import Path

import numpy as np


class DifferentiallyPrivateProfileAggregator:
    """
    Aggregate user preferences with differential privacy.

    When multiple users' preferences are combined (e.g., for
    team-level personalization), add noise to prevent individual
    user identification from aggregate statistics.
    """

    def __init__(self, epsilon: float = 1.0):
        """
        Args:
            epsilon: privacy budget (lower = more private, noisier)
        """
        self.epsilon = epsilon

    def aggregate_preferences(
        self,
        user_preferences: list[dict[str, float]],
    ) -> dict[str, float]:
        """
        Aggregate preferences across users with DP noise.

        Returns noisy aggregate that preserves privacy of
        individual contributions.
        """
        if not user_preferences:
            return {}

        # Collect all preference keys
        all_keys = set()
        for prefs in user_preferences:
            all_keys.update(prefs.keys())

        # Aggregate with Laplace noise
        aggregated = {}
        n = len(user_preferences)

        for key in all_keys:
            values = [p.get(key, 0.0) for p in user_preferences]
            true_mean = np.mean(values)

            # Laplace mechanism: add noise proportional to sensitivity/epsilon
            sensitivity = 1.0 / n  # Max impact of one user on the mean
            noise = np.random.laplace(0, sensitivity / self.epsilon)

            noisy_mean = true_mean + noise
            aggregated[key] = float(np.clip(noisy_mean, 0, 1))

        return aggregated


class OnDevicePersonalization:
    """
    Keep personalization data on the user's device.

    Instead of storing preferences server-side, compute
    personalization signals client-side and send only the
    filtering parameters (not the raw preferences).
    """

    @staticmethod
    def compute_client_side_filters(
        user_preferences: dict,
        available_technologies: list[str],
        threshold: float = 0.5,
    ) -> dict:
        """
        Compute retrieval filters client-side.

        The server receives only filter parameters, not raw preferences.
        This is the most privacy-preserving approach for personalization.
        """
        # Client-side: determine which technologies to filter for
        tech_filter = [
            tech for tech in available_technologies
            if user_preferences.get(f"interest_{tech}", 0) >= threshold
        ]

        # Client-side: determine expertise level
        expertise_scores = {
            "beginner": user_preferences.get("skill_beginner", 0),
            "intermediate": user_preferences.get("skill_intermediate", 0),
            "advanced": user_preferences.get("skill_advanced", 0),
        }
        expertise = max(expertise_scores, key=expertise_scores.get)

        # Send to server: only filter parameters, not raw scores
        return {
            "technologies": tech_filter,
            "expertise_level": expertise,
        }


class GDPRCompliantPipeline:
    """
    End-to-end GDPR-compliant personalization pipeline.

    Implements all GDPR requirements:
    - Consent management (Article 6)
    - Right to access (Article 15)
    - Right to rectification (Article 16)
    - Right to erasure (Article 17)
    - Right to portability (Article 20)
    - Data minimization (Article 5)
    - Purpose limitation (Article 5)
    """

    def __init__(self, storage_dir: str):
        self.storage_dir = Path(storage_dir)
        self.storage_dir.mkdir(parents=True, exist_ok=True)

    def record_consent(
        self,
        user_id: str,
        consented_categories: list[str],
        consent_text: str,
    ) -> None:
        """
        Record user consent for data processing.

        Categories: "profile", "query_history", "preferences",
                    "entity_memory", "behavioral_signals"
        """
        consent_record = {
            "user_id": self._hash_id(user_id),
            "categories": consented_categories,
            "consent_text": consent_text,
            "timestamp": datetime.now().isoformat(),
            "ip_hash": None,  # Optionally record hashed IP for audit
        }

        path = self._user_dir(user_id) / "consent.json"
        path.parent.mkdir(parents=True, exist_ok=True)
        with open(path, "w") as f:
            json.dump(consent_record, f, indent=2)

    def store_data(
        self,
        user_id: str,
        category: str,
        data: dict,
    ) -> bool:
        """
        Store user data only if consent exists for this category.

        Returns True if stored, False if consent is missing.
        """
        consent = self._load_consent(user_id)
        if not consent or category not in consent.get("categories", []):
            return False

        # Data minimization: only store necessary fields
        minimized = self._minimize_data(data, category)

        data_path = self._user_dir(user_id) / f"{category}.json"
        existing = []
        if data_path.exists():
            with open(data_path) as f:
                existing = json.load(f)

        existing.append({
            "data": minimized,
            "timestamp": datetime.now().isoformat(),
        })

        with open(data_path, "w") as f:
            json.dump(existing, f, indent=2)

        return True

    def access_data(self, user_id: str) -> dict:
        """
        Right to access: return all stored data for a user.

        Article 15: users can request a copy of their data.
        """
        user_dir = self._user_dir(user_id)
        if not user_dir.exists():
            return {"user_id": user_id, "data": {}}

        all_data = {}
        for data_file in user_dir.glob("*.json"):
            with open(data_file) as f:
                all_data[data_file.stem] = json.load(f)

        return {"user_id": user_id, "data": all_data}

    def delete_data(self, user_id: str) -> bool:
        """
        Right to erasure: delete all user data.

        Article 17: users can request complete data deletion.
        """
        import shutil
        user_dir = self._user_dir(user_id)
        if user_dir.exists():
            shutil.rmtree(user_dir)
            return True
        return False

    def export_data(self, user_id: str) -> str:
        """
        Right to portability: export data in machine-readable format.

        Article 20: provide data in JSON format.
        """
        data = self.access_data(user_id)
        return json.dumps(data, indent=2)

    def _minimize_data(self, data: dict, category: str) -> dict:
        """
        Data minimization: only keep fields necessary for the purpose.
        """
        allowed_fields = {
            "profile": {"role", "expertise_level", "team"},
            "query_history": {"query", "timestamp"},
            "preferences": {"preference_key", "preference_value"},
            "entity_memory": {"entity", "relation", "value"},
        }

        allowed = allowed_fields.get(category, set())
        if not allowed:
            return data

        return {k: v for k, v in data.items() if k in allowed}

    def _hash_id(self, user_id: str) -> str:
        return hashlib.sha256(
            f"personal:{user_id}".encode()
        ).hexdigest()[:16]

    def _user_dir(self, user_id: str) -> Path:
        return self.storage_dir / self._hash_id(user_id)

    def _load_consent(self, user_id: str) -> dict | None:
        path = self._user_dir(user_id) / "consent.json"
        if not path.exists():
            return None
        with open(path) as f:
            return json.load(f)
```

---

## End-to-End Pipeline

### Complete Personalized RAG Pipeline

```python
"""
Complete personalized RAG pipeline tying all components together.
"""
import anthropic


class PersonalizedRAGPipeline:
    """
    Production-ready personalized RAG pipeline.

    Components:
    1. User profile manager (load/update user context)
    2. Memory system (retrieve relevant memories)
    3. Personalized retriever (filter + rerank with preferences)
    4. Context augmenter (inject memories into prompt)
    5. Personalized generator (adapt output to user)
    """

    def __init__(
        self,
        retriever: "QdrantPersonalizedRetriever",
        reranker: "PreferenceAwareReranker",
        memory: "CustomVectorMemory",
        model: str = "claude-sonnet-4-20250514",
    ):
        self.retriever = retriever
        self.reranker = reranker
        self.memory = memory
        self.client = anthropic.Anthropic()
        self.model = model

    def invoke(
        self,
        query: str,
        user: "UserContext",
    ) -> dict:
        """
        Process a personalized query end-to-end.
        """
        # 1. Retrieve relevant memories
        memory_results = self.memory.search(user.user_id, query, limit=5)
        memory_context = self.memory.format_for_prompt(
            user.user_id, query, limit=5,
        )
        user_memories = [
            {"content": mem.content, "type": mem.memory_type}
            for mem, _ in memory_results
        ]

        # 2. Retrieve documents with user filtering
        candidates = self.retriever.retrieve(
            query=query,
            user=user,
            k=20,
            strategy="soft",
        )

        # 3. Rerank with preference awareness
        reranked = self.reranker.rerank(
            query=query,
            documents=candidates,
            user=user,
            user_memories=user_memories,
            top_k=5,
        )

        # 4. Build personalized prompt
        system_prompt = self._build_system_prompt(user)
        context = self._build_context(reranked, memory_context)

        # 5. Generate personalized answer
        response = self.client.messages.create(
            model=self.model,
            max_tokens=1000,
            system=system_prompt,
            messages=[{
                "role": "user",
                "content": (
                    f"{context}\n\n---\n\n"
                    f"Question: {query}\n\n"
                    "Answer the question using the documentation above. "
                    "Adapt your response to the user's context and preferences."
                ),
            }],
        )

        answer = response.content[0].text

        # 6. Store interaction in memory
        self.memory.extract_and_store(user.user_id, query, answer)

        return {
            "answer": answer,
            "sources": [
                {
                    "text": doc["text"][:200],
                    "score": doc.get("final_score", doc.get("score", 0)),
                    "preference_boost": doc.get("preference_score", 0),
                }
                for doc in reranked
            ],
            "memories_used": len(memory_results),
            "personalization_applied": True,
        }

    def _build_system_prompt(self, user: "UserContext") -> str:
        """Build a system prompt adapted to the user."""
        base = (
            "You are a technical assistant. Answer questions using "
            "the provided documentation. Only include information "
            "supported by the documentation."
        )

        if user.expertise_level == "beginner":
            base += (
                "\n\nThe user is a beginner. Explain concepts clearly, "
                "define technical terms, and provide step-by-step guidance."
            )
        elif user.expertise_level == "advanced":
            base += (
                "\n\nThe user is an advanced practitioner. Be concise. "
                "Focus on specifics, edge cases, and performance considerations."
            )

        if user.tech_stack:
            stack = ", ".join(user.tech_stack[:5])
            base += f"\n\nThe user works with: {stack}."

        return base

    def _build_context(
        self,
        documents: list[dict],
        memory_context: str,
    ) -> str:
        """Assemble the context from documents and memory."""
        sections = []

        if memory_context:
            sections.append(memory_context)

        doc_texts = []
        for i, doc in enumerate(documents, 1):
            doc_texts.append(f"[Document {i}]\n{doc['text']}")

        sections.append("\n\n".join(doc_texts))

        return "\n\n---\n\n".join(sections)
```

---

## Testing Personalization

```python
"""
Tests for personalized RAG pipeline.
"""
import pytest


class TestPersonalizedRetrieval:
    """Test that personalization actually changes retrieval results."""

    def test_tech_stack_filtering(self, retriever, documents):
        """Documents should be filtered by user tech stack."""
        user_python = UserContext(
            user_id="test-1",
            tech_stack=["python"],
        )
        user_java = UserContext(
            user_id="test-2",
            tech_stack=["java"],
        )

        results_python = retriever.retrieve("how to connect to database", user_python, k=5)
        results_java = retriever.retrieve("how to connect to database", user_java, k=5)

        # Results should be different
        python_techs = [r["metadata"].get("technology") for r in results_python]
        java_techs = [r["metadata"].get("technology") for r in results_java]

        assert "python" in python_techs, "Python user should get Python docs"
        assert "java" in java_techs, "Java user should get Java docs"

    def test_expertise_adaptation(self, pipeline):
        """Answers should adapt to expertise level."""
        query = "What is a database index?"

        beginner = UserContext(user_id="b1", expertise_level="beginner")
        advanced = UserContext(user_id="a1", expertise_level="advanced")

        beginner_answer = pipeline.invoke(query, beginner)["answer"]
        advanced_answer = pipeline.invoke(query, advanced)["answer"]

        # Beginner answer should be longer (more explanation)
        assert len(beginner_answer) > len(advanced_answer) * 0.8

    def test_fallback_without_personalization(self, pipeline):
        """New users should still get reasonable answers."""
        new_user = UserContext(user_id="new-user-no-history")
        result = pipeline.invoke("how to deploy my app", new_user)

        assert result["answer"]
        assert len(result["answer"]) > 50

    def test_memory_accumulation(self, pipeline, memory):
        """Memories should accumulate across interactions."""
        user = UserContext(user_id="mem-test", tech_stack=["python"])

        pipeline.invoke("How do I use SQLAlchemy?", user)
        pipeline.invoke("What about Alembic migrations?", user)

        memories = memory.get_user_summary("mem-test")
        assert memories["total"] > 0
```

---

## Common Pitfalls

1. **Hard filters returning empty results**: always implement a fallback to unfiltered retrieval when personalized filters return too few results. An unpersonalized answer is better than no answer.

2. **Reranking weight imbalance**: if preference_weight is too high (>0.5), irrelevant documents matching user preferences will outrank relevant documents. Start with 70/30 relevance/preference and tune from there.

3. **Leaking personalization across users**: ensure memory isolation is airtight. A memory system bug that mixes User A's preferences into User B's retrieval is both a quality failure and a privacy violation.

4. **Ignoring consent withdrawal**: GDPR allows users to withdraw consent at any time. When consent is withdrawn, you must stop using their data immediately and delete it within 30 days. Build this into your pipeline from day one.

5. **Not measuring lift**: always A/B test personalized vs non-personalized responses. If personalization does not measurably improve user satisfaction or task completion, it is adding complexity for no benefit.

6. **Over-personalizing rare queries**: for rare or novel queries, user history has little to contribute. Detect when a query is outside the user's historical pattern and fall back to non-personalized retrieval.

---

## References

- Qdrant filtering: https://qdrant.tech/documentation/concepts/filtering/
- ChromaDB metadata filtering: https://docs.trychroma.com/guides/filtering
- Pinecone metadata filtering: https://docs.pinecone.io/guides/data/filter-with-metadata
- GDPR requirements: https://gdpr-info.eu/
- Differential privacy: Dwork, C. and Roth, A. "The Algorithmic Foundations of Differential Privacy." 2014
