# Ontology-Guided Retrieval -- Query Expansion via Synonyms, Hyponyms, and Broader Concepts

## TL;DR

Query expansion is the core technique of ontology-guided retrieval: before sending a query to the vector store, expand it with synonyms, hyponyms (narrower terms), hypernyms (broader terms), and related concepts from a domain ontology. This article provides production implementations for three expansion strategies: (1) synonym expansion using preferred labels and alternate labels from SKOS/OWL ontologies, (2) hierarchical expansion walking the broader/narrower concept tree, and (3) weighted multi-strategy expansion that combines all relationships with configurable weights and depth limits.

---

## Expansion Strategy 1: Synonym Expansion

### Why Synonyms Matter

The same concept often has multiple names in domain literature. Without synonym expansion, retrieval misses documents using alternative terminology:

```python
from dataclasses import dataclass, field


@dataclass
class SynonymSet:
    canonical: str
    synonyms: list[str]
    abbreviations: list[str] = field(default_factory=list)
    misspellings: list[str] = field(default_factory=list)


# Examples of domain-specific synonym sets
MEDICAL_SYNONYMS = [
    SynonymSet(
        canonical="myocardial infarction",
        synonyms=["heart attack", "MI", "acute myocardial infarction", "AMI"],
        abbreviations=["MI", "AMI", "STEMI", "NSTEMI"],
    ),
    SynonymSet(
        canonical="hypertension",
        synonyms=["high blood pressure", "elevated blood pressure", "HTN"],
        abbreviations=["HTN", "HBP"],
    ),
    SynonymSet(
        canonical="cerebrovascular accident",
        synonyms=["stroke", "brain attack", "CVA"],
        abbreviations=["CVA", "TIA"],
    ),
]


class SynonymExpander:
    """Expand query terms using ontology synonym relationships.

    Supports multiple synonym sources:
    - SKOS prefLabel / altLabel / hiddenLabel
    - OWL equivalentClass labels
    - Custom synonym dictionaries
    """

    def __init__(self, ontology_index):
        self.ontology = ontology_index
        self.custom_synonyms: dict[str, list[str]] = {}

    def add_custom_synonyms(self, term: str, synonyms: list[str]) -> None:
        """Add custom synonyms not in the ontology."""
        key = term.lower().strip()
        self.custom_synonyms.setdefault(key, []).extend(synonyms)

    def expand(self, term: str) -> list[str]:
        """Get all synonyms for a term.

        Priority order:
        1. Ontology preferred label
        2. Ontology alternative labels
        3. Custom synonyms
        4. Abbreviation variants
        """
        expanded = set()

        # Ontology lookup
        concept = self.ontology.lookup(term)
        if concept:
            expanded.add(concept.preferred_label)
            expanded.update(concept.synonyms)

        # Custom synonyms
        custom = self.custom_synonyms.get(term.lower().strip(), [])
        expanded.update(custom)

        # Always include original term
        expanded.add(term)

        # Remove the query term itself to avoid redundancy
        return list(expanded)

    def expand_query(self, query: str) -> str:
        """Expand an entire query by replacing recognized terms with expansions.

        Returns the expanded query as a single string for embedding.
        """
        # Tokenize and look up each term and bigram
        words = query.split()
        expansions = []
        used_indices = set()

        # Check bigrams first (more specific)
        for i in range(len(words) - 1):
            bigram = f"{words[i]} {words[i + 1]}"
            syns = self.expand(bigram)
            if len(syns) > 1:  # found synonyms beyond the original
                expansions.append(" OR ".join(syns[:5]))
                used_indices.add(i)
                used_indices.add(i + 1)

        # Check individual words
        for i, word in enumerate(words):
            if i in used_indices:
                continue
            syns = self.expand(word)
            if len(syns) > 1:
                expansions.append(" OR ".join(syns[:5]))
            else:
                expansions.append(word)

        return " ".join(expansions)

    def get_expansion_stats(self, queries: list[str]) -> dict:
        """Analyze expansion coverage across a set of queries."""
        total_terms = 0
        expanded_terms = 0
        expansion_sizes = []

        for query in queries:
            words = query.split()
            for word in words:
                total_terms += 1
                syns = self.expand(word)
                if len(syns) > 1:
                    expanded_terms += 1
                    expansion_sizes.append(len(syns))

        return {
            "total_terms": total_terms,
            "expanded_terms": expanded_terms,
            "coverage": expanded_terms / max(total_terms, 1),
            "avg_expansion_size": (
                sum(expansion_sizes) / max(len(expansion_sizes), 1)
            ),
        }
```

---

## Expansion Strategy 2: Hierarchical Expansion

### Walking the Concept Tree

```python
from enum import Enum


class ExpansionDirection(Enum):
    BROADER = "broader"     # hypernyms (parent concepts)
    NARROWER = "narrower"   # hyponyms (child concepts)
    BOTH = "both"


class HierarchicalExpander:
    """Expand queries using ontology hierarchy (broader/narrower).

    Broader expansion: "STEMI" -> also search "myocardial infarction", "cardiac event"
    Narrower expansion: "cardiac event" -> also search "STEMI", "NSTEMI", "unstable angina"

    The expansion direction depends on the query intent:
    - Specific query ("What is STEMI?") -> expand broader for context
    - General query ("Tell me about cardiac events") -> expand narrower for specifics
    """

    def __init__(self, ontology_index):
        self.ontology = ontology_index

    def expand_broader(
        self, term: str, max_depth: int = 2, max_terms: int = 10
    ) -> list[dict]:
        """Expand upward in the hierarchy (hypernyms).

        Returns terms with their depth (distance from original term).
        Closer terms are more relevant.
        """
        concept = self.ontology.lookup(term)
        if not concept:
            return []

        results = []
        frontier = [(uri, 1) for uri in concept.broader]
        visited = {concept.uri}

        while frontier and len(results) < max_terms:
            uri, depth = frontier.pop(0)
            if depth > max_depth or uri in visited:
                continue
            visited.add(uri)

            parent = self.ontology.concepts.get(uri)
            if parent:
                results.append({
                    "term": parent.preferred_label,
                    "depth": depth,
                    "relationship": "broader",
                    "weight": 1.0 / (depth + 1),  # closer = higher weight
                })
                for parent_uri in parent.broader:
                    frontier.append((parent_uri, depth + 1))

        return results

    def expand_narrower(
        self, term: str, max_depth: int = 2, max_terms: int = 15
    ) -> list[dict]:
        """Expand downward in the hierarchy (hyponyms).

        Narrower expansion generates more terms because concept trees
        are typically wider at lower levels.
        """
        concept = self.ontology.lookup(term)
        if not concept:
            return []

        results = []
        frontier = [(uri, 1) for uri in concept.narrower]
        visited = {concept.uri}

        while frontier and len(results) < max_terms:
            uri, depth = frontier.pop(0)
            if depth > max_depth or uri in visited:
                continue
            visited.add(uri)

            child = self.ontology.concepts.get(uri)
            if child:
                results.append({
                    "term": child.preferred_label,
                    "depth": depth,
                    "relationship": "narrower",
                    "weight": 1.0 / (depth + 1),
                })
                for child_uri in child.narrower:
                    frontier.append((child_uri, depth + 1))

        return results

    def expand(
        self,
        term: str,
        direction: ExpansionDirection = ExpansionDirection.BOTH,
        max_depth: int = 2,
    ) -> list[dict]:
        """Expand in the specified direction(s)."""
        results = []

        if direction in (ExpansionDirection.BROADER, ExpansionDirection.BOTH):
            results.extend(self.expand_broader(term, max_depth))

        if direction in (ExpansionDirection.NARROWER, ExpansionDirection.BOTH):
            results.extend(self.expand_narrower(term, max_depth))

        # Sort by weight (relevance)
        results.sort(key=lambda x: x["weight"], reverse=True)
        return results

    def detect_expansion_direction(self, query: str, term: str) -> ExpansionDirection:
        """Heuristically determine the best expansion direction.

        Rules:
        - If the query is asking "what is" or "define": expand BROADER for context
        - If the query is asking "types of" or "examples": expand NARROWER
        - If the query is asking "related to" or "overview": expand BOTH
        """
        query_lower = query.lower()

        narrower_signals = ["types of", "examples of", "list all", "categories of", "kinds of"]
        broader_signals = ["what is", "define", "explain", "meaning of"]

        for signal in narrower_signals:
            if signal in query_lower:
                return ExpansionDirection.NARROWER

        for signal in broader_signals:
            if signal in query_lower:
                return ExpansionDirection.BROADER

        return ExpansionDirection.BOTH
```

---

## Expansion Strategy 3: Weighted Multi-Strategy Expansion

```python
import numpy as np


class WeightedQueryExpander:
    """Combine synonym, hierarchical, and related-concept expansion.

    Each expansion type gets a configurable weight:
    - Synonyms: high weight (0.9-1.0) -- same concept, different words
    - Narrower: medium weight (0.5-0.7) -- specific instances
    - Broader: medium-low weight (0.3-0.5) -- general context
    - Related: low weight (0.2-0.4) -- associated concepts

    The weights control how much each expansion type influences retrieval.
    """

    def __init__(
        self,
        synonym_expander: SynonymExpander,
        hierarchy_expander: HierarchicalExpander,
        weights: dict | None = None,
    ):
        self.synonym = synonym_expander
        self.hierarchy = hierarchy_expander
        self.weights = weights or {
            "synonym": 0.95,
            "narrower": 0.60,
            "broader": 0.40,
            "related": 0.30,
        }

    def expand(
        self,
        query: str,
        max_expansion_terms: int = 20,
    ) -> dict:
        """Full weighted expansion of a query.

        Returns:
        - expanded_terms: list of (term, weight, source) tuples
        - expanded_query: single string for embedding
        - expansion_metadata: details for debugging
        """
        # Extract key terms
        key_terms = self._extract_domain_terms(query)

        all_expansions: list[tuple[str, float, str]] = []
        expansion_details = {}

        for term in key_terms:
            term_expansions = []

            # Synonyms
            synonyms = self.synonym.expand(term)
            for syn in synonyms:
                if syn.lower() != term.lower():
                    term_expansions.append(
                        (syn, self.weights["synonym"], "synonym")
                    )

            # Hierarchy
            direction = self.hierarchy.detect_expansion_direction(query, term)
            hierarchy_terms = self.hierarchy.expand(term, direction, max_depth=2)
            for ht in hierarchy_terms:
                base_weight = self.weights.get(ht["relationship"], 0.3)
                term_expansions.append(
                    (ht["term"], base_weight * ht["weight"], ht["relationship"])
                )

            # Related concepts
            concept = self.synonym.ontology.lookup(term)
            if concept:
                for related_uri in concept.related[:5]:
                    related = self.synonym.ontology.concepts.get(related_uri)
                    if related:
                        term_expansions.append(
                            (related.preferred_label, self.weights["related"], "related")
                        )

            expansion_details[term] = term_expansions
            all_expansions.extend(term_expansions)

        # Deduplicate and sort by weight
        seen = set()
        unique_expansions = []
        for exp_term, weight, source in all_expansions:
            key = exp_term.lower()
            if key not in seen:
                seen.add(key)
                unique_expansions.append((exp_term, weight, source))

        unique_expansions.sort(key=lambda x: x[1], reverse=True)
        unique_expansions = unique_expansions[:max_expansion_terms]

        # Build expanded query string
        weighted_terms = []
        for exp_term, weight, _ in unique_expansions:
            if weight >= 0.7:
                weighted_terms.append(exp_term)
            elif weight >= 0.4:
                weighted_terms.append(exp_term)

        expanded_query = query + " " + " ".join(weighted_terms)

        return {
            "original_query": query,
            "expanded_query": expanded_query,
            "expanded_terms": unique_expansions,
            "key_terms_found": key_terms,
            "expansion_details": expansion_details,
        }

    def create_multiple_queries(
        self, query: str, max_queries: int = 4
    ) -> list[dict]:
        """Create multiple query variants for multi-query retrieval.

        Instead of one expanded query, create several focused queries:
        1. Original query (highest weight)
        2. Query with synonyms only
        3. Query focused on narrower concepts
        4. Query focused on broader context
        """
        key_terms = self._extract_domain_terms(query)
        queries = [
            {"query": query, "weight": 1.0, "strategy": "original"},
        ]

        # Synonym-expanded query
        synonym_terms = []
        for term in key_terms:
            syns = self.synonym.expand(term)
            synonym_terms.extend(s for s in syns if s.lower() != term.lower())

        if synonym_terms:
            queries.append({
                "query": query + " " + " ".join(synonym_terms[:10]),
                "weight": 0.8,
                "strategy": "synonym",
            })

        # Narrower-focused query
        narrower_terms = []
        for term in key_terms:
            children = self.hierarchy.expand_narrower(term, max_depth=1, max_terms=5)
            narrower_terms.extend(c["term"] for c in children)

        if narrower_terms:
            queries.append({
                "query": query + " " + " ".join(narrower_terms[:8]),
                "weight": 0.6,
                "strategy": "narrower",
            })

        # Broader-focused query
        broader_terms = []
        for term in key_terms:
            parents = self.hierarchy.expand_broader(term, max_depth=1, max_terms=3)
            broader_terms.extend(p["term"] for p in parents)

        if broader_terms:
            queries.append({
                "query": query + " " + " ".join(broader_terms[:5]),
                "weight": 0.4,
                "strategy": "broader",
            })

        return queries[:max_queries]

    def _extract_domain_terms(self, query: str) -> list[str]:
        """Extract terms that exist in the ontology."""
        words = query.lower().split()
        found_terms = []

        # Check bigrams first
        for i in range(len(words) - 1):
            bigram = f"{words[i]} {words[i + 1]}"
            if self.synonym.ontology.lookup(bigram):
                found_terms.append(bigram)

        # Check individual words
        for word in words:
            if self.synonym.ontology.lookup(word):
                found_terms.append(word)

        return found_terms if found_terms else words[:3]
```

---

## Multi-Query Retrieval with Expansion

```python
class MultiQueryOntologyRetriever:
    """Execute multiple expanded queries and merge results.

    Instead of embedding one expanded query (which dilutes specificity),
    run multiple focused queries and merge results with weighted scoring.
    """

    def __init__(self, expander: WeightedQueryExpander, vector_store, embedding_model):
        self.expander = expander
        self.store = vector_store
        self.embedder = embedding_model

    def retrieve(self, query: str, top_k: int = 10) -> list[dict]:
        """Retrieve using multi-query expansion."""
        # Generate query variants
        query_variants = self.expander.create_multiple_queries(query)

        # Run each query
        all_results: dict[str, dict] = {}

        for variant in query_variants:
            embedding = self.embedder.embed([variant["query"]])[0]
            results = self.store.query(embedding, top_k=top_k)

            for result in results:
                doc_id = result.get("id", "")
                score = result.get("score", 0) * variant["weight"]

                if doc_id in all_results:
                    all_results[doc_id]["score"] = max(
                        all_results[doc_id]["score"], score
                    )
                    all_results[doc_id]["matched_strategies"].append(
                        variant["strategy"]
                    )
                else:
                    all_results[doc_id] = {
                        **result,
                        "score": score,
                        "matched_strategies": [variant["strategy"]],
                    }

        # Sort and return top-K
        sorted_results = sorted(
            all_results.values(),
            key=lambda x: x["score"],
            reverse=True,
        )
        return sorted_results[:top_k]
```

---

## Evaluation: Measuring Expansion Quality

```python
def evaluate_expansion(
    test_queries: list[dict],
    retriever_without_expansion,
    retriever_with_expansion,
    k: int = 10,
) -> dict:
    """Compare retrieval with and without ontology expansion.

    test_queries: list of {"query": str, "relevant_docs": list[str]}
    """
    metrics = {
        "without_expansion": {"recall": [], "precision": [], "mrr": []},
        "with_expansion": {"recall": [], "precision": [], "mrr": []},
    }

    for test in test_queries:
        query = test["query"]
        relevant = set(test["relevant_docs"])

        # Without expansion
        results_no_exp = retriever_without_expansion.retrieve(query, top_k=k)
        retrieved_no_exp = set(r.get("id", "") for r in results_no_exp)

        # With expansion
        results_exp = retriever_with_expansion.retrieve(query, top_k=k)
        retrieved_exp = set(r.get("id", "") for r in results_exp)

        # Recall
        metrics["without_expansion"]["recall"].append(
            len(relevant & retrieved_no_exp) / max(len(relevant), 1)
        )
        metrics["with_expansion"]["recall"].append(
            len(relevant & retrieved_exp) / max(len(relevant), 1)
        )

        # Precision
        metrics["without_expansion"]["precision"].append(
            len(relevant & retrieved_no_exp) / max(len(retrieved_no_exp), 1)
        )
        metrics["with_expansion"]["precision"].append(
            len(relevant & retrieved_exp) / max(len(retrieved_exp), 1)
        )

    # Average metrics
    summary = {}
    for method in ["without_expansion", "with_expansion"]:
        summary[method] = {
            metric: round(sum(values) / max(len(values), 1), 4)
            for metric, values in metrics[method].items()
        }

    summary["recall_improvement"] = round(
        summary["with_expansion"]["recall"] - summary["without_expansion"]["recall"], 4
    )

    return summary
```

---

## Common Pitfalls

1. **Expanding every word in the query.** Only expand domain-specific terms that exist in the ontology. Expanding "what" or "the" adds noise.

2. **Equal weighting for all expansion types.** Synonyms should weight nearly 1.0; broader concepts at 2 levels deep should weight 0.2. Use decaying weights.

3. **Unbounded narrower expansion.** A broad concept like "disease" may have 10,000 narrower terms. Always cap max_terms and max_depth.

4. **Not measuring the precision/recall tradeoff.** Expansion typically increases recall but may decrease precision. Measure both and tune weights accordingly.

5. **Using expansion with already-good embeddings.** Modern embedding models already capture many synonym relationships. Test whether expansion actually improves your specific use case before adding complexity.

---

## References

- Carpineto & Romano "A Survey of Automatic Query Expansion in Information Retrieval" (ACM Computing Surveys, 2012)
- SKOS Reference: https://www.w3.org/TR/skos-reference/
- UMLS (Unified Medical Language System): https://www.nlm.nih.gov/research/umls/
