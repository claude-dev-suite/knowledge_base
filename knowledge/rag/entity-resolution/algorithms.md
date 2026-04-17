# Entity Resolution -- Algorithms: Splink Probabilistic, Embedding-Based, and LLM-Assisted

## TL;DR

This article provides production-ready implementations of three entity resolution algorithms for RAG knowledge graphs: (1) Splink probabilistic linkage using the Fellegi-Sunter model with blocking strategies for scalability, (2) embedding-based resolution using contextual entity embeddings with approximate nearest neighbor search, and (3) LLM-assisted resolution for high-accuracy disambiguation of ambiguous cases. Each algorithm is implemented as a standalone module and then combined into a tiered pipeline that optimizes for both accuracy and cost.

---

## Algorithm 1: Splink Probabilistic Record Linkage

### How Splink Works

Splink implements the Fellegi-Sunter model of probabilistic record linkage. For each pair of records, it computes a match weight based on how fields agree or disagree, accounting for the rarity of each value.

```python
import splink.duckdb.linker as splink_linker
from splink.duckdb.linker import DuckDBLinker
import splink.duckdb.comparison_library as cl
import splink.duckdb.comparison_template_library as ctl
import pandas as pd
from dataclasses import dataclass


@dataclass
class EntityRecord:
    """Structured entity record for Splink resolution."""
    unique_id: str
    entity_name: str
    entity_type: str
    description: str
    source_document: str
    context: str


def prepare_splink_dataframe(entities: list[EntityRecord]) -> pd.DataFrame:
    """Convert entity records to a DataFrame suitable for Splink.

    Splink requires a DataFrame with a unique_id column and
    comparison columns. We create derived columns for better matching:
    - name_clean: normalized entity name
    - name_tokens: sorted tokens for bag-of-words comparison
    - first_word: first word of the name (useful for blocking)
    """
    records = []
    for entity in entities:
        name_clean = entity.entity_name.lower().strip()
        name_tokens = " ".join(sorted(name_clean.split()))
        first_word = name_clean.split()[0] if name_clean.split() else ""

        records.append({
            "unique_id": entity.unique_id,
            "entity_name": entity.entity_name,
            "name_clean": name_clean,
            "name_tokens": name_tokens,
            "first_word": first_word,
            "entity_type": entity.entity_type,
            "description": entity.description[:200],
            "source_document": entity.source_document,
        })

    return pd.DataFrame(records)


def configure_splink_model(df: pd.DataFrame) -> dict:
    """Configure the Splink linkage model.

    The settings define:
    - link_type: dedupe_only (finding duplicates within one dataset)
    - blocking_rules: which record pairs to compare (limits quadratic explosion)
    - comparisons: how to compare each field
    - retention: what columns to keep in output
    """
    settings = {
        "link_type": "dedupe_only",
        "unique_id_column_name": "unique_id",

        # Blocking rules reduce comparisons from O(n^2) to O(n)
        # Only compare records that share at least one blocking criterion
        "blocking_rules_to_generate_predictions": [
            # Block 1: Same first word of name
            "l.first_word = r.first_word",
            # Block 2: Same entity type
            "l.entity_type = r.entity_type",
        ],

        "comparisons": [
            # Name comparison: exact, Jaro-Winkler, and Levenshtein
            ctl.name_comparison("entity_name", term_frequency_adjustments=True),

            # Cleaned name: exact and fuzzy
            cl.levenshtein_at_thresholds("name_clean", [1, 2, 3]),

            # Token comparison (bag-of-words)
            cl.jaccard_at_thresholds("name_tokens", [0.7, 0.5]),

            # Entity type: exact match only
            cl.exact_match("entity_type"),

            # Description: fuzzy comparison
            cl.jaro_winkler_at_thresholds("description", [0.8, 0.7]),
        ],

        "retain_matching_columns": True,
        "retain_intermediate_calculation_columns": False,
        "max_iterations": 20,
        "em_convergence": 0.001,
    }

    return settings


def run_splink_resolution(
    entities: list[EntityRecord],
    match_weight_threshold: float = 5.0,
) -> list[dict]:
    """Run Splink entity resolution and return match pairs.

    The match_weight_threshold controls precision/recall tradeoff:
    - Higher (e.g., 10): fewer matches, higher precision
    - Lower (e.g., 2): more matches, higher recall
    - Default 5.0 is a balanced starting point

    Returns list of dicts with:
    - entity_a_id, entity_b_id: the matched pair
    - match_weight: log-likelihood ratio
    - match_probability: probability they are the same entity
    """
    # Prepare data
    df = prepare_splink_dataframe(entities)
    settings = configure_splink_model(df)

    # Initialize linker
    linker = DuckDBLinker(df, settings)

    # Estimate model parameters
    # Step 1: Estimate probability two random records match
    linker.estimate_probability_two_random_records_match(
        "l.entity_type = r.entity_type AND l.first_word = r.first_word",
        recall=0.7,
    )

    # Step 2: Estimate u (non-match) parameters using random sampling
    linker.estimate_u_using_random_sampling(max_pairs=1e6)

    # Step 3: Estimate m (match) parameters using EM algorithm
    # Use blocking rules to create training pairs
    training_blocking_rule = "l.name_clean = r.name_clean"
    linker.estimate_parameters_using_expectation_maximisation(
        training_blocking_rule
    )

    # Additional EM pass with different blocking
    linker.estimate_parameters_using_expectation_maximisation(
        "l.entity_type = r.entity_type"
    )

    # Predict matches
    predictions = linker.predict(threshold_match_weight=match_weight_threshold)
    results_df = predictions.as_pandas_dataframe()

    # Convert to match pairs
    matches = []
    for _, row in results_df.iterrows():
        matches.append({
            "entity_a_id": row["unique_id_l"],
            "entity_b_id": row["unique_id_r"],
            "match_weight": row["match_weight"],
            "match_probability": row["match_probability"],
        })

    return sorted(matches, key=lambda x: x["match_probability"], reverse=True)


def cluster_splink_matches(
    matches: list[dict],
    entities: list[EntityRecord],
    probability_threshold: float = 0.85,
) -> dict[str, list[str]]:
    """Cluster matched entity pairs into resolution groups.

    Uses connected components: if A matches B and B matches C,
    then {A, B, C} form a cluster even if A and C were not compared.
    """
    # Filter by probability
    strong_matches = [
        m for m in matches if m["match_probability"] >= probability_threshold
    ]

    # Union-Find for clustering
    parent: dict[str, str] = {}
    for entity in entities:
        parent[entity.unique_id] = entity.unique_id

    def find(x: str) -> str:
        while parent[x] != x:
            parent[x] = parent[parent[x]]
            x = parent[x]
        return x

    def union(x: str, y: str) -> None:
        rx, ry = find(x), find(y)
        if rx != ry:
            parent[ry] = rx

    for match in strong_matches:
        union(match["entity_a_id"], match["entity_b_id"])

    # Group by cluster root
    clusters: dict[str, list[str]] = {}
    for entity in entities:
        root = find(entity.unique_id)
        clusters.setdefault(root, []).append(entity.unique_id)

    # Return only clusters with more than one entity
    return {k: v for k, v in clusters.items() if len(v) > 1}
```

---

## Algorithm 2: Embedding-Based Resolution

### Contextual Entity Embeddings

```python
import numpy as np
from typing import Protocol


class EmbeddingModel(Protocol):
    def embed(self, texts: list[str]) -> list[list[float]]: ...


class EmbeddingEntityResolver:
    """Entity resolution using contextual embeddings.

    Key insight: embed the entity name + its surrounding context,
    not just the name. "Python" in a programming context embeds
    differently from "Python" in a zoology context.

    Pipeline:
    1. Create contextual embedding for each entity mention
    2. Use approximate nearest neighbor search to find candidates
    3. Apply a similarity threshold to identify matches
    4. Cluster matches into resolution groups
    """

    def __init__(
        self,
        embedding_model: EmbeddingModel,
        similarity_threshold: float = 0.88,
        context_weight: float = 0.3,
    ):
        self.model = embedding_model
        self.threshold = similarity_threshold
        self.context_weight = context_weight

    def create_entity_embeddings(
        self, entities: list[dict]
    ) -> np.ndarray:
        """Create contextual embeddings for all entities.

        Each entity dict should have: name, type, description, context

        The embedding text combines:
        - Entity name (primary signal)
        - Entity type (disambiguation)
        - Description or context (semantic enrichment)
        """
        texts = []
        for entity in entities:
            # Weighted combination: name is primary, context is secondary
            text = (
                f"{entity['name']}. "
                f"Type: {entity.get('type', 'unknown')}. "
                f"{entity.get('description', '')} "
                f"{entity.get('context', '')[:200]}"
            )
            texts.append(text)

        # Batch embed
        embeddings = self.model.embed(texts)
        return np.array(embeddings)

    def find_candidates_brute_force(
        self, embeddings: np.ndarray
    ) -> list[tuple[int, int, float]]:
        """Find all entity pairs above similarity threshold.

        Brute-force approach: O(n^2) comparisons.
        Use for datasets under 10,000 entities.
        """
        # L2 normalize for cosine similarity
        norms = np.linalg.norm(embeddings, axis=1, keepdims=True)
        normalized = embeddings / np.maximum(norms, 1e-10)

        # Compute full similarity matrix
        sim_matrix = normalized @ normalized.T

        # Extract pairs above threshold (upper triangle only)
        matches = []
        n = len(embeddings)
        for i in range(n):
            for j in range(i + 1, n):
                if sim_matrix[i][j] >= self.threshold:
                    matches.append((i, j, float(sim_matrix[i][j])))

        return sorted(matches, key=lambda x: x[2], reverse=True)

    def find_candidates_faiss(
        self, embeddings: np.ndarray, top_k: int = 20
    ) -> list[tuple[int, int, float]]:
        """Find candidates using FAISS approximate nearest neighbors.

        For datasets over 10,000 entities, FAISS is much faster
        than brute-force comparison.
        """
        import faiss

        dimension = embeddings.shape[1]
        n = len(embeddings)

        # Normalize for cosine similarity
        norms = np.linalg.norm(embeddings, axis=1, keepdims=True)
        normalized = (embeddings / np.maximum(norms, 1e-10)).astype("float32")

        # Build FAISS index
        if n < 50000:
            index = faiss.IndexFlatIP(dimension)  # exact inner product
        else:
            # IVF index for larger datasets
            nlist = min(int(np.sqrt(n)), 256)
            quantizer = faiss.IndexFlatIP(dimension)
            index = faiss.IndexIVFFlat(quantizer, dimension, nlist)
            index.train(normalized)
            index.nprobe = min(nlist, 20)

        index.add(normalized)

        # Search: each entity finds its top-K nearest neighbors
        similarities, indices = index.search(normalized, top_k + 1)

        matches = []
        seen = set()
        for i in range(n):
            for k in range(top_k + 1):
                j = int(indices[i][k])
                sim = float(similarities[i][k])

                if j <= i or sim < self.threshold:
                    continue

                pair = (min(i, j), max(i, j))
                if pair not in seen:
                    seen.add(pair)
                    matches.append((pair[0], pair[1], sim))

        return sorted(matches, key=lambda x: x[2], reverse=True)

    def resolve(
        self,
        entities: list[dict],
        use_faiss: bool = True,
    ) -> dict[str, list[int]]:
        """Full embedding-based resolution pipeline."""
        # Step 1: Create embeddings
        embeddings = self.create_entity_embeddings(entities)

        # Step 2: Find candidate pairs
        if use_faiss and len(entities) > 5000:
            candidates = self.find_candidates_faiss(embeddings)
        else:
            candidates = self.find_candidates_brute_force(embeddings)

        # Step 3: Type-aware filtering
        typed_candidates = []
        for i, j, sim in candidates:
            type_i = entities[i].get("type", "")
            type_j = entities[j].get("type", "")
            # Only merge if types match or one is unknown
            if type_i == type_j or not type_i or not type_j:
                typed_candidates.append((i, j, sim))

        # Step 4: Cluster using connected components
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

        for i, j, _ in typed_candidates:
            union(i, j)

        clusters: dict[int, list[int]] = {}
        for idx in range(n):
            root = find(idx)
            clusters.setdefault(root, []).append(idx)

        return {str(k): v for k, v in clusters.items() if len(v) > 1}
```

---

## Algorithm 3: LLM-Assisted Resolution

### Production LLM Resolution Pipeline

```python
import json
from dataclasses import dataclass


@dataclass
class ResolutionJudgment:
    entity_a_idx: int
    entity_b_idx: int
    is_same: bool
    confidence: int
    canonical_name: str
    reasoning: str


class LLMEntityResolver:
    """LLM-assisted entity resolution for high-accuracy disambiguation.

    The LLM is the most accurate resolver but also the most expensive.
    Optimize cost by:
    1. Only sending borderline cases (pre-filtered by string/embedding)
    2. Batching multiple comparisons per LLM call
    3. Caching judgments for repeated entity pairs
    """

    BATCH_PROMPT = """You are an entity resolution expert. For each pair below,
determine whether the two mentions refer to the same real-world entity.

Consider:
- Surface form variations (abbreviations, typos, versioning)
- Context clues (surrounding text indicates the domain)
- Type compatibility (a PERSON and an ORGANIZATION cannot be the same entity)

{comparisons}

For EACH comparison, respond on a single line:
COMP_N: SAME=yes/no CONF=1-10 CANONICAL="<best name>"

Example:
COMP_1: SAME=yes CONF=9 CANONICAL="PostgreSQL"
COMP_2: SAME=no CONF=8 CANONICAL=""
"""

    def __init__(self, llm, batch_size: int = 15, cache: dict | None = None):
        self.llm = llm
        self.batch_size = batch_size
        self.cache = cache if cache is not None else {}

    def resolve_pairs(
        self, entities: list[dict], candidate_pairs: list[tuple[int, int, float]]
    ) -> list[ResolutionJudgment]:
        """Resolve a list of candidate pairs using the LLM."""
        all_judgments = []

        # Process in batches
        for i in range(0, len(candidate_pairs), self.batch_size):
            batch = candidate_pairs[i : i + self.batch_size]

            # Check cache first
            uncached = []
            for idx_a, idx_b, sim in batch:
                cache_key = self._cache_key(entities[idx_a], entities[idx_b])
                if cache_key in self.cache:
                    all_judgments.append(self.cache[cache_key])
                else:
                    uncached.append((idx_a, idx_b, sim))

            if not uncached:
                continue

            # Build comparison text
            comparisons = []
            for comp_idx, (idx_a, idx_b, sim) in enumerate(uncached, 1):
                a = entities[idx_a]
                b = entities[idx_b]
                comparisons.append(
                    f"COMPARISON {comp_idx}:\n"
                    f"  A: \"{a['name']}\" (type={a.get('type', '?')})\n"
                    f"     Context: {a.get('context', 'N/A')[:150]}\n"
                    f"  B: \"{b['name']}\" (type={b.get('type', '?')})\n"
                    f"     Context: {b.get('context', 'N/A')[:150]}\n"
                    f"  Similarity: {sim:.3f}"
                )

            prompt = self.BATCH_PROMPT.format(
                comparisons="\n\n".join(comparisons)
            )

            response = self.llm.invoke(prompt)
            parsed = self._parse_batch_response(
                response.content, uncached, entities
            )

            for judgment in parsed:
                cache_key = self._cache_key(
                    entities[judgment.entity_a_idx],
                    entities[judgment.entity_b_idx],
                )
                self.cache[cache_key] = judgment
                all_judgments.append(judgment)

        return all_judgments

    def _parse_batch_response(
        self,
        text: str,
        pairs: list[tuple[int, int, float]],
        entities: list[dict],
    ) -> list[ResolutionJudgment]:
        """Parse batch LLM response into judgments."""
        judgments = []
        lines = text.strip().split("\n")

        comp_idx = 0
        for line in lines:
            line = line.strip()
            if not line.startswith("COMP_"):
                continue

            if comp_idx >= len(pairs):
                break

            idx_a, idx_b, _ = pairs[comp_idx]

            is_same = "SAME=yes" in line.lower()
            confidence = 5
            canonical = ""

            # Parse CONF
            if "CONF=" in line.upper():
                try:
                    conf_part = line.upper().split("CONF=")[1].split()[0]
                    confidence = int(conf_part)
                except (ValueError, IndexError):
                    pass

            # Parse CANONICAL
            if 'CANONICAL="' in line:
                try:
                    canonical = line.split('CANONICAL="')[1].split('"')[0]
                except IndexError:
                    pass

            judgments.append(ResolutionJudgment(
                entity_a_idx=idx_a,
                entity_b_idx=idx_b,
                is_same=is_same,
                confidence=confidence,
                canonical_name=canonical or entities[idx_a]["name"],
                reasoning="",
            ))

            comp_idx += 1

        # Fill missing judgments
        while comp_idx < len(pairs):
            idx_a, idx_b, _ = pairs[comp_idx]
            judgments.append(ResolutionJudgment(
                entity_a_idx=idx_a,
                entity_b_idx=idx_b,
                is_same=False,
                confidence=0,
                canonical_name="",
                reasoning="Parse failure",
            ))
            comp_idx += 1

        return judgments

    @staticmethod
    def _cache_key(entity_a: dict, entity_b: dict) -> str:
        """Create a deterministic cache key for an entity pair."""
        names = sorted([entity_a["name"], entity_b["name"]])
        return f"{names[0]}|||{names[1]}"
```

---

## Tiered Pipeline: Combining All Three Algorithms

```python
class TieredEntityResolver:
    """Three-tier entity resolution pipeline.

    Tier 1: String matching (free, fast)
        -> Resolves exact/near-exact duplicates
        -> Passes unresolved pairs to Tier 2

    Tier 2: Embedding similarity (cheap, medium speed)
        -> Resolves semantic duplicates
        -> Passes borderline pairs to Tier 3

    Tier 3: LLM judgment (expensive, slow)
        -> Resolves ambiguous cases with high accuracy

    Cost model for 10,000 entities:
    - Tier 1: ~0 cost, resolves ~40% of duplicates
    - Tier 2: ~$0.50 (embeddings), resolves ~40% of remaining
    - Tier 3: ~$2-5 (LLM calls for ~200 borderline pairs)
    - Total: ~$3-6 vs $50+ for LLM-only approach
    """

    def __init__(
        self,
        embedding_model: EmbeddingModel,
        llm,
        string_threshold: float = 0.85,
        embedding_threshold: float = 0.88,
        llm_threshold_range: tuple[float, float] = (0.75, 0.88),
        llm_confidence_threshold: int = 7,
    ):
        self.string_resolver = StringMatchResolver(threshold=string_threshold)
        self.embedding_resolver = EmbeddingEntityResolver(
            embedding_model=embedding_model,
            similarity_threshold=embedding_threshold,
        )
        self.llm_resolver = LLMEntityResolver(llm=llm)

        self.llm_range = llm_threshold_range
        self.llm_confidence = llm_confidence_threshold

    def resolve(self, entities: list[dict]) -> dict:
        """Run the full tiered resolution pipeline.

        Returns:
        - clusters: dict mapping canonical_id -> list of entity indices
        - canonical_names: dict mapping canonical_id -> chosen name
        - stats: resolution statistics per tier
        """
        n = len(entities)
        # Union-Find for global clustering
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

        stats = {"tier1_matches": 0, "tier2_matches": 0, "tier3_matches": 0}

        # --- Tier 1: String matching ---
        names = [e["name"] for e in entities]
        name_mapping = self.string_resolver.resolve_cluster(names)

        # Group by canonical name
        canonical_groups: dict[str, list[int]] = {}
        for i, name in enumerate(names):
            canonical = name_mapping.get(name, name)
            canonical_groups.setdefault(canonical, []).append(i)

        for group in canonical_groups.values():
            if len(group) > 1:
                for idx in group[1:]:
                    union(group[0], idx)
                    stats["tier1_matches"] += 1

        # --- Tier 2: Embedding similarity ---
        embeddings = self.embedding_resolver.create_entity_embeddings(entities)

        # Only compare entities NOT already resolved in Tier 1
        unresolved_indices = []
        resolved_roots = set()
        for i in range(n):
            root = find(i)
            if root not in resolved_roots:
                resolved_roots.add(root)
                unresolved_indices.append(i)

        if len(unresolved_indices) > 1:
            # Compute similarities among unresolved entities
            sub_embeddings = embeddings[unresolved_indices]
            norms = np.linalg.norm(sub_embeddings, axis=1, keepdims=True)
            normalized = sub_embeddings / np.maximum(norms, 1e-10)
            sim_matrix = normalized @ normalized.T

            borderline_pairs = []  # for Tier 3

            for si in range(len(unresolved_indices)):
                for sj in range(si + 1, len(unresolved_indices)):
                    real_i = unresolved_indices[si]
                    real_j = unresolved_indices[sj]
                    sim = float(sim_matrix[si][sj])

                    # Type check
                    if (
                        entities[real_i].get("type")
                        and entities[real_j].get("type")
                        and entities[real_i]["type"] != entities[real_j]["type"]
                    ):
                        continue

                    if sim >= self.embedding_resolver.threshold:
                        union(real_i, real_j)
                        stats["tier2_matches"] += 1
                    elif self.llm_range[0] <= sim < self.llm_range[1]:
                        borderline_pairs.append((real_i, real_j, sim))

            # --- Tier 3: LLM for borderline cases ---
            if borderline_pairs:
                judgments = self.llm_resolver.resolve_pairs(
                    entities, borderline_pairs
                )
                for judgment in judgments:
                    if (
                        judgment.is_same
                        and judgment.confidence >= self.llm_confidence
                    ):
                        union(judgment.entity_a_idx, judgment.entity_b_idx)
                        stats["tier3_matches"] += 1

        # Build final clusters
        clusters: dict[int, list[int]] = {}
        for i in range(n):
            root = find(i)
            clusters.setdefault(root, []).append(i)

        # Choose canonical names
        canonical_names = {}
        for root, members in clusters.items():
            # Prefer the longest, most descriptive name
            member_names = [entities[i]["name"] for i in members]
            canonical = max(member_names, key=len)
            canonical_names[root] = canonical

        return {
            "clusters": {k: v for k, v in clusters.items() if len(v) > 1},
            "canonical_names": canonical_names,
            "stats": stats,
            "total_entities": n,
            "unique_after_resolution": len(clusters),
        }
```

---

## Evaluation Metrics

```python
def evaluate_entity_resolution(
    predicted_clusters: dict[int, list[int]],
    gold_clusters: dict[int, list[int]],
    n_entities: int,
) -> dict:
    """Evaluate ER quality using standard metrics.

    Metrics:
    - Pairwise precision: fraction of predicted same-entity pairs that are correct
    - Pairwise recall: fraction of true same-entity pairs that were found
    - Pairwise F1: harmonic mean
    - Cluster-level accuracy: fraction of clusters that exactly match gold
    """
    # Build pairwise sets
    def pairs_from_clusters(clusters: dict) -> set[tuple[int, int]]:
        pairs = set()
        for members in clusters.values():
            for i in range(len(members)):
                for j in range(i + 1, len(members)):
                    pair = (min(members[i], members[j]), max(members[i], members[j]))
                    pairs.add(pair)
        return pairs

    predicted_pairs = pairs_from_clusters(predicted_clusters)
    gold_pairs = pairs_from_clusters(gold_clusters)

    true_positives = len(predicted_pairs & gold_pairs)
    false_positives = len(predicted_pairs - gold_pairs)
    false_negatives = len(gold_pairs - predicted_pairs)

    precision = true_positives / max(true_positives + false_positives, 1)
    recall = true_positives / max(true_positives + false_negatives, 1)
    f1 = 2 * precision * recall / max(precision + recall, 1e-10)

    return {
        "pairwise_precision": round(precision, 4),
        "pairwise_recall": round(recall, 4),
        "pairwise_f1": round(f1, 4),
        "true_positives": true_positives,
        "false_positives": false_positives,
        "false_negatives": false_negatives,
        "predicted_clusters": len(predicted_clusters),
        "gold_clusters": len(gold_clusters),
    }
```

---

## Common Pitfalls

1. **Skipping blocking in Splink.** Without blocking rules, Splink compares every pair of records (O(n^2)). For 100,000 entities, that is 5 billion comparisons. Always define blocking rules.

2. **Using entity name embeddings without context.** Embedding just "Apple" gives a generic vector. Embedding "Apple (technology company) -- maker of iPhone and Mac" is far more discriminative.

3. **Not using type constraints.** Embedding similarity cannot distinguish between a person named "Java" and the programming language "Java." Always enforce type compatibility before merging.

4. **Sending too many pairs to the LLM.** LLM resolution at $0.01/comparison adds up fast. Use the tiered approach: string matching and embeddings handle 80%+ of cases; send only borderline pairs to the LLM.

5. **Not caching LLM judgments.** The same entity pair may appear across multiple batches. Cache judgments by entity name pair to avoid redundant LLM calls.

6. **Transitive closure errors.** If A matches B and B matches C, union-find merges all three. But if B was a false positive, the entire cluster is wrong. Monitor cluster sizes and flag unusually large clusters for review.

---

## References

- Splink documentation: https://moj-analytical-services.github.io/splink/
- Fellegi & Sunter (1969): "A Theory for Record Linkage"
- FAISS: https://github.com/facebookresearch/faiss
- Dedupe library: https://docs.dedupe.io/
