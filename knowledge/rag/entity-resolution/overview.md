# Entity Resolution for Graph RAG -- Why It Matters in Multi-Source RAG

## TL;DR

Entity resolution (ER) is the process of identifying when different mentions across documents refer to the same real-world entity. In Graph RAG, failing to resolve entities creates fragmented knowledge graphs where "React.js", "ReactJS", "React 18", and "React" become four separate nodes instead of one, destroying multi-hop reasoning and inflating the graph with redundant information. This overview explains why ER is critical for multi-source RAG, categorizes the resolution strategies (string matching, probabilistic, embedding-based, LLM-assisted), and provides a decision framework for choosing the right approach based on corpus characteristics.

---

## The Entity Resolution Problem in RAG

### How Fragmentation Destroys Graph Quality

Consider a knowledge graph built from three sources:

- **Source A (blog post)**: "React.js is maintained by Meta."
- **Source B (docs)**: "ReactJS supports server-side rendering."
- **Source C (meeting notes)**: "We chose React 18 for the frontend."

Without entity resolution, the graph contains three disconnected nodes:

```
[React.js] --maintained_by--> [Meta]
[ReactJS]  --supports--> [server-side rendering]
[React 18] --chosen_for--> [frontend]
```

A query like "What framework did we choose and who maintains it?" requires connecting all three mentions. Without ER, the graph cannot make this connection.

With entity resolution, all three merge into a single canonical entity:

```
[React] --maintained_by--> [Meta]
[React] --supports--> [server-side rendering]
[React] --chosen_for--> [frontend]
```

Now the query traverses a connected graph and produces a complete answer.

### Quantifying the Impact

| Metric | Without ER | With ER |
|--------|-----------|---------|
| Unique entity nodes | 2-5x inflated | Correct count |
| Average node degree | Low (fragmented) | Higher (connected) |
| Multi-hop query success rate | 30-50% | 80-95% |
| Community detection quality | Poor (many singletons) | Good (coherent groups) |
| Global search relevance | Diluted across duplicates | Concentrated |
| Storage overhead | 2-5x | 1x |

### Why This Is Hard

Entity resolution is challenging because mentions vary across multiple dimensions:

```python
from dataclasses import dataclass


@dataclass
class EntityMention:
    surface_form: str      # The text as it appears in the document
    entity_type: str       # PERSON, ORG, TECHNOLOGY, etc.
    source_document: str   # Where this mention was found
    context: str           # Surrounding text


# Example: all of these refer to the same entity
MENTION_VARIATIONS = [
    # Abbreviations and short forms
    EntityMention("AWS", "ORG", "doc1", "We deployed to AWS"),
    EntityMention("Amazon Web Services", "ORG", "doc2", "Amazon Web Services provides..."),

    # Version differences
    EntityMention("Python 3.12", "TECHNOLOGY", "doc3", "Upgraded to Python 3.12"),
    EntityMention("Python", "TECHNOLOGY", "doc4", "Python is our primary language"),
    EntityMention("CPython", "TECHNOLOGY", "doc5", "CPython implementation details"),

    # Typos and OCR errors
    EntityMention("PostgreSQL", "TECHNOLOGY", "doc6", "PostgreSQL database"),
    EntityMention("PostGreSQL", "TECHNOLOGY", "doc7", "PostGreSQL cluster"),
    EntityMention("Postgres", "TECHNOLOGY", "doc8", "Postgres query optimization"),

    # Contextual ambiguity
    EntityMention("Apple", "ORG", "doc9", "Apple released new MacBooks"),
    EntityMention("Apple", "FOOD", "doc10", "Apple varieties in the orchard"),

    # Person name variations
    EntityMention("Dr. Jane Smith", "PERSON", "doc11", "Dr. Jane Smith presented"),
    EntityMention("J. Smith", "PERSON", "doc12", "J. Smith authored the paper"),
    EntityMention("Jane", "PERSON", "doc13", "Jane mentioned in the meeting"),
]
```

---

## Resolution Strategy Categories

### 1. String Matching (Simplest)

Compare entity names using string similarity metrics. Fast but brittle.

```python
from difflib import SequenceMatcher


class StringMatchResolver:
    """Resolve entities using string similarity.

    Good for: typos, capitalization differences, simple abbreviations
    Bad for: completely different names for the same entity (AWS vs Amazon Web Services)
    """

    def __init__(self, threshold: float = 0.85):
        self.threshold = threshold

    def are_same_entity(self, mention_a: str, mention_b: str) -> tuple[bool, float]:
        """Check if two mentions refer to the same entity."""
        # Normalize
        a = mention_a.lower().strip()
        b = mention_b.lower().strip()

        # Exact match after normalization
        if a == b:
            return True, 1.0

        # Substring match (one is contained in the other)
        if a in b or b in a:
            shorter = min(len(a), len(b))
            longer = max(len(a), len(b))
            return True, shorter / longer

        # Sequence similarity
        ratio = SequenceMatcher(None, a, b).ratio()
        return ratio >= self.threshold, ratio

    def resolve_cluster(self, mentions: list[str]) -> dict[str, str]:
        """Cluster mentions and assign canonical names.

        Returns a mapping from each mention to its canonical form.
        """
        # Build similarity graph
        n = len(mentions)
        clusters: dict[int, list[int]] = {i: [i] for i in range(n)}
        parent: dict[int, int] = {i: i for i in range(n)}

        def find(x: int) -> int:
            while parent[x] != x:
                parent[x] = parent[parent[x]]
                x = parent[x]
            return x

        def union(x: int, y: int) -> None:
            rx, ry = find(x), find(y)
            if rx != ry:
                parent[ry] = rx

        for i in range(n):
            for j in range(i + 1, n):
                is_same, _ = self.are_same_entity(mentions[i], mentions[j])
                if is_same:
                    union(i, j)

        # Group by cluster
        groups: dict[int, list[str]] = {}
        for i in range(n):
            root = find(i)
            groups.setdefault(root, []).append(mentions[i])

        # Choose canonical name (longest non-abbreviated form)
        mapping = {}
        for group in groups.values():
            canonical = max(group, key=len)
            for mention in group:
                mapping[mention] = canonical

        return mapping
```

### 2. Probabilistic (Splink)

Model-based resolution using probabilistic record linkage.

```python
def explain_probabilistic_er() -> dict:
    """Explain the probabilistic approach to entity resolution.

    Probabilistic ER (Fellegi-Sunter model) estimates the probability
    that two records refer to the same entity based on:
    - Agreement on fields (name, type, context)
    - Disagreement on fields
    - Prior probability of a match
    - Field-specific comparison functions (exact, fuzzy, phonetic)

    The key insight is that rare agreements (e.g., matching on an
    unusual name) provide more evidence of a match than common
    agreements (e.g., both entities are type "TECHNOLOGY").
    """
    return {
        "model": "Fellegi-Sunter probabilistic record linkage",
        "library": "Splink (open source, Python)",
        "strengths": [
            "Handles uncertainty quantitatively (match probability)",
            "Accounts for field-specific comparison metrics",
            "Scales well with blocking strategies",
            "Produces interpretable match weights",
        ],
        "weaknesses": [
            "Requires training data or manual parameter tuning",
            "Limited to structured record comparison",
            "Cannot understand semantic meaning",
        ],
        "best_for": "Large-scale ER with structured entity records",
        "see_also": "algorithms.md for full Splink implementation",
    }
```

### 3. Embedding-Based

Use vector similarity between entity descriptions to find matches.

```python
import numpy as np


class EmbeddingResolver:
    """Resolve entities using embedding similarity.

    Instead of comparing surface forms, embed the entity name + context
    and compare in vector space. This handles:
    - Synonyms (AWS = Amazon Web Services)
    - Descriptions (same concept, different words)
    - Multilingual entities

    Limitation: embedding similarity alone can produce false positives
    for related-but-different entities (React vs Vue.js).
    """

    def __init__(self, embedding_model, threshold: float = 0.88):
        self.model = embedding_model
        self.threshold = threshold

    def build_entity_embeddings(
        self, entities: list[dict]
    ) -> list[dict]:
        """Embed all entities for comparison.

        Each entity dict should have: name, type, description, context
        """
        texts = []
        for entity in entities:
            # Combine name, type, and context for richer embedding
            text = (
                f"{entity['name']} ({entity['type']}): "
                f"{entity.get('description', '')} "
                f"Context: {entity.get('context', '')}"
            )
            texts.append(text)

        embeddings = self.model.embed(texts)

        for entity, emb in zip(entities, embeddings):
            entity["embedding"] = emb

        return entities

    def find_matches(
        self, entities: list[dict], batch_size: int = 100
    ) -> list[tuple[int, int, float]]:
        """Find all entity pairs above the similarity threshold.

        Returns list of (idx_a, idx_b, similarity) tuples.
        """
        embeddings = np.array([e["embedding"] for e in entities])

        # Normalize for cosine similarity
        norms = np.linalg.norm(embeddings, axis=1, keepdims=True)
        normalized = embeddings / np.maximum(norms, 1e-10)

        matches = []
        n = len(entities)

        # Process in blocks to manage memory
        for i in range(0, n, batch_size):
            block = normalized[i : i + batch_size]
            # Compare block against all entities after it
            for j in range(i, n, batch_size):
                if j == i:
                    # Within the same block: upper triangle only
                    sim_matrix = block @ block.T
                    for bi in range(len(block)):
                        for bj in range(bi + 1, len(block)):
                            if sim_matrix[bi][bj] >= self.threshold:
                                matches.append(
                                    (i + bi, i + bj, float(sim_matrix[bi][bj]))
                                )
                else:
                    other_block = normalized[j : j + batch_size]
                    sim_matrix = block @ other_block.T
                    for bi in range(len(block)):
                        for bj in range(len(other_block)):
                            if sim_matrix[bi][bj] >= self.threshold:
                                matches.append(
                                    (i + bi, j + bj, float(sim_matrix[bi][bj]))
                                )

        return sorted(matches, key=lambda x: x[2], reverse=True)

    def cluster_entities(
        self, entities: list[dict], matches: list[tuple[int, int, float]]
    ) -> dict[int, list[int]]:
        """Cluster matched entities using connected components."""
        n = len(entities)
        parent = list(range(n))

        def find(x: int) -> int:
            while parent[x] != x:
                parent[x] = parent[parent[x]]
                x = parent[x]
            return x

        def union(x: int, y: int) -> None:
            rx, ry = find(x), find(y)
            if rx != ry:
                parent[ry] = rx

        for idx_a, idx_b, _ in matches:
            # Additional type check: only merge if types are compatible
            if entities[idx_a]["type"] == entities[idx_b]["type"]:
                union(idx_a, idx_b)

        clusters: dict[int, list[int]] = {}
        for i in range(n):
            root = find(i)
            clusters.setdefault(root, []).append(i)

        return {k: v for k, v in clusters.items() if len(v) > 1}
```

### 4. LLM-Assisted

Use an LLM to judge whether two entity mentions refer to the same entity.

```python
class LLMResolver:
    """Use an LLM to resolve ambiguous entity matches.

    The LLM is used as a final judge for cases where string matching
    and embedding similarity are inconclusive. This is the most
    accurate but also the most expensive approach.

    Recommended: use string + embedding as first pass, then LLM
    only for borderline cases (similarity between 0.70 and 0.90).
    """

    def __init__(self, llm, max_comparisons_per_batch: int = 20):
        self.llm = llm
        self.max_batch = max_comparisons_per_batch

    def judge_pair(
        self,
        entity_a: dict,
        entity_b: dict,
    ) -> dict:
        """Ask the LLM whether two entities refer to the same thing."""
        prompt = (
            "Do these two entity mentions refer to the same real-world entity?\n\n"
            f"Entity A:\n"
            f"  Name: {entity_a['name']}\n"
            f"  Type: {entity_a['type']}\n"
            f"  Context: {entity_a.get('context', 'N/A')}\n\n"
            f"Entity B:\n"
            f"  Name: {entity_b['name']}\n"
            f"  Type: {entity_b['type']}\n"
            f"  Context: {entity_b.get('context', 'N/A')}\n\n"
            "Respond in this exact format:\n"
            "SAME: yes/no\n"
            "CONFIDENCE: 1-10\n"
            "CANONICAL_NAME: <the best name to use for this entity>\n"
            "REASONING: <brief explanation>"
        )

        response = self.llm.invoke(prompt)
        return self._parse_response(response.content)

    def judge_batch(
        self,
        pairs: list[tuple[dict, dict]],
    ) -> list[dict]:
        """Judge multiple entity pairs in a single LLM call.

        More efficient than individual calls: one LLM call for up to
        20 comparisons instead of 20 separate calls.
        """
        if len(pairs) > self.max_batch:
            pairs = pairs[: self.max_batch]

        comparisons = []
        for i, (a, b) in enumerate(pairs, 1):
            comparisons.append(
                f"Comparison {i}:\n"
                f"  A: {a['name']} ({a['type']}) - Context: {a.get('context', 'N/A')[:100]}\n"
                f"  B: {b['name']} ({b['type']}) - Context: {b.get('context', 'N/A')[:100]}"
            )

        prompt = (
            "For each comparison, determine if the two entities refer "
            "to the same real-world entity.\n\n"
            + "\n\n".join(comparisons)
            + "\n\nFor each comparison, respond:\n"
            "COMPARISON N: SAME=yes/no, CONFIDENCE=1-10, CANONICAL=<name>\n"
        )

        response = self.llm.invoke(prompt)
        return self._parse_batch_response(response.content, len(pairs))

    @staticmethod
    def _parse_response(text: str) -> dict:
        result = {"same": False, "confidence": 0, "canonical_name": "", "reasoning": ""}
        for line in text.strip().split("\n"):
            line = line.strip()
            if line.upper().startswith("SAME:"):
                result["same"] = "yes" in line.lower()
            elif line.upper().startswith("CONFIDENCE:"):
                try:
                    result["confidence"] = int(line.split(":")[1].strip())
                except ValueError:
                    result["confidence"] = 5
            elif line.upper().startswith("CANONICAL_NAME:"):
                result["canonical_name"] = line.split(":", 1)[1].strip()
            elif line.upper().startswith("REASONING:"):
                result["reasoning"] = line.split(":", 1)[1].strip()
        return result

    @staticmethod
    def _parse_batch_response(text: str, expected: int) -> list[dict]:
        results = []
        for line in text.strip().split("\n"):
            if "COMPARISON" in line.upper() and "SAME" in line.upper():
                same = "SAME=yes" in line.lower() or "same=yes" in line.lower()
                conf = 5
                canonical = ""
                parts = line.split(",")
                for part in parts:
                    if "CONFIDENCE" in part.upper():
                        try:
                            conf = int(part.split("=")[1].strip())
                        except (ValueError, IndexError):
                            pass
                    if "CANONICAL" in part.upper():
                        try:
                            canonical = part.split("=")[1].strip()
                        except IndexError:
                            pass
                results.append({
                    "same": same,
                    "confidence": conf,
                    "canonical_name": canonical,
                })
        # Pad if parsing missed some
        while len(results) < expected:
            results.append({"same": False, "confidence": 0, "canonical_name": ""})
        return results
```

---

## Choosing the Right Approach

```python
def recommend_er_strategy(
    entity_count: int,
    source_count: int,
    entity_types: list[str],
    budget_per_entity_usd: float,
    accuracy_requirement: str,  # "low", "medium", "high"
) -> dict:
    """Recommend an entity resolution strategy."""

    if entity_count < 500 and accuracy_requirement == "high":
        return {
            "strategy": "LLM-assisted (all pairs)",
            "reasoning": "Small enough for LLM to judge all pairs directly",
            "estimated_cost": entity_count * (entity_count - 1) / 2 * 0.001,
        }

    if accuracy_requirement == "low" or budget_per_entity_usd < 0.001:
        return {
            "strategy": "String matching only",
            "reasoning": "Cheapest option, handles typos and capitalization",
            "estimated_cost": 0,
        }

    # Default: tiered approach
    return {
        "strategy": "Tiered: string -> embedding -> LLM",
        "reasoning": (
            "String matching resolves obvious duplicates (free). "
            "Embedding similarity catches semantic matches ($0.0001/entity). "
            "LLM judges only borderline cases ($0.001/comparison)."
        ),
        "estimated_cost": entity_count * 0.001,
        "tier_details": {
            "tier_1_string": "Resolves ~40% of duplicates (free)",
            "tier_2_embedding": "Resolves ~40% of remaining (cheap)",
            "tier_3_llm": "Resolves ~15% borderline cases (expensive)",
            "remaining_5_percent": "Manual review or accept as separate",
        },
    }
```

---

## Common Pitfalls

1. **Merging entities of different types.** "Apple" (company) and "Apple" (fruit) should not merge. Always check entity type compatibility before merging.

2. **Losing provenance after resolution.** After merging "React.js" into "React", preserve the original surface forms as aliases. Queries for "React.js" should still work.

3. **Running ER as a one-time step.** New documents introduce new mentions. ER must be incremental -- resolve new entities against the existing canonical set.

4. **Setting thresholds too aggressively.** A similarity threshold of 0.95 misses valid matches ("PostgreSQL" vs "Postgres"). Start at 0.80 and tune on evaluation data.

5. **Ignoring context in resolution.** Two entities named "Mercury" in a tech corpus (the database) and a science corpus (the planet) need context to disambiguate.

6. **Not evaluating resolution quality.** Measure precision (how many merges were correct) and recall (how many true duplicates were found) on a labeled sample.

---

## References

- Fellegi & Sunter "A Theory for Record Linkage" (1969)
- Splink: https://moj-analytical-services.github.io/splink/
- Entity Resolution survey: Christophides et al. "An Overview of End-to-End Entity Resolution for Big Data" (ACM Computing Surveys, 2021)
- Dedupe: https://docs.dedupe.io/
