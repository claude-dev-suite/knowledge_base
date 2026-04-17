# Ontology-Guided Retrieval -- Domain Ontologies for Smarter RAG

## TL;DR

Ontology-guided retrieval uses formal domain ontologies (structured vocabularies of concepts and their relationships) to improve RAG retrieval beyond keyword and vector similarity. When a user asks about "myocardial infarction," an ontology-aware system knows to also retrieve documents about "heart attack," "acute coronary syndrome," and "STEMI" because the ontology encodes these synonymy and hierarchical relationships. This overview covers why ontologies matter for domain-specific RAG, the main ontology formats (OWL, SKOS, RDF), key domain ontologies (SNOMED CT for healthcare, FIBO for finance, Schema.org for general), and how to integrate ontological knowledge into retrieval pipelines.

---

## Why Ontologies Improve RAG Retrieval

### The Vocabulary Gap Problem

Domain-specific RAG suffers from vocabulary mismatch: users and documents use different terms for the same concept.

```python
VOCABULARY_GAP_EXAMPLES = {
    "healthcare": {
        "user_query": "heart attack symptoms",
        "document_terms": [
            "myocardial infarction",
            "acute coronary syndrome",
            "STEMI",
            "NSTEMI",
            "troponin elevation",
        ],
        "vector_similarity": 0.65,  # partial match
        "ontology_expansion": [
            "heart attack = myocardial infarction (synonym)",
            "myocardial infarction IS-A acute coronary syndrome",
            "STEMI IS-A myocardial infarction",
        ],
        "expanded_recall": 0.92,
    },
    "finance": {
        "user_query": "stock option vesting",
        "document_terms": [
            "equity compensation",
            "RSU",
            "restricted stock unit",
            "cliff vesting",
            "graded vesting schedule",
        ],
        "vector_similarity": 0.60,
        "ontology_expansion": [
            "stock option IS-A equity compensation",
            "RSU IS-A equity compensation",
            "vesting HAS-PART cliff vesting, graded vesting",
        ],
        "expanded_recall": 0.88,
    },
    "legal": {
        "user_query": "breach of contract remedies",
        "document_terms": [
            "contractual default",
            "specific performance",
            "compensatory damages",
            "liquidated damages",
            "anticipatory repudiation",
        ],
        "vector_similarity": 0.58,
        "ontology_expansion": [
            "breach of contract = contractual default (synonym)",
            "specific performance IS-A remedy",
            "compensatory damages IS-A remedy",
        ],
        "expanded_recall": 0.85,
    },
}
```

### How Ontologies Help

An ontology encodes three types of relationships that embeddings capture poorly:

| Relationship | Example | Embedding Similarity |
|-------------|---------|---------------------|
| Synonymy | heart attack = myocardial infarction | 0.75 (moderate) |
| Hierarchy | STEMI is-a myocardial infarction | 0.60 (weak) |
| Part-whole | left ventricle part-of heart | 0.45 (very weak) |
| Association | troponin related-to myocardial infarction | 0.50 (weak) |

Embeddings capture synonymy reasonably well but are poor at hierarchy and part-whole relationships. Ontologies make these explicit.

```python
from dataclasses import dataclass, field


@dataclass
class OntologyConcept:
    uri: str
    preferred_label: str
    synonyms: list[str] = field(default_factory=list)
    broader: list[str] = field(default_factory=list)   # parent concepts
    narrower: list[str] = field(default_factory=list)  # child concepts
    related: list[str] = field(default_factory=list)    # associated concepts
    definition: str = ""


class OntologyIndex:
    """In-memory ontology index for query expansion.

    Loads an ontology and provides fast lookup for:
    - Synonym expansion (all labels for a concept)
    - Hierarchical expansion (broader/narrower concepts)
    - Related concept expansion
    """

    def __init__(self):
        self.concepts: dict[str, OntologyConcept] = {}
        self.label_to_uri: dict[str, str] = {}

    def add_concept(self, concept: OntologyConcept) -> None:
        """Add a concept to the index."""
        self.concepts[concept.uri] = concept
        # Index all labels for lookup
        self.label_to_uri[concept.preferred_label.lower()] = concept.uri
        for syn in concept.synonyms:
            self.label_to_uri[syn.lower()] = concept.uri

    def lookup(self, term: str) -> OntologyConcept | None:
        """Look up a concept by any of its labels."""
        uri = self.label_to_uri.get(term.lower())
        return self.concepts.get(uri) if uri else None

    def get_synonyms(self, term: str) -> list[str]:
        """Get all synonyms for a term."""
        concept = self.lookup(term)
        if not concept:
            return []
        return [concept.preferred_label] + concept.synonyms

    def get_broader(self, term: str, max_depth: int = 2) -> list[str]:
        """Get broader (parent) concepts up to max_depth levels."""
        concept = self.lookup(term)
        if not concept:
            return []

        broader_terms = []
        frontier = concept.broader[:]
        visited = {concept.uri}
        depth = 0

        while frontier and depth < max_depth:
            next_frontier = []
            for uri in frontier:
                if uri in visited:
                    continue
                visited.add(uri)
                parent = self.concepts.get(uri)
                if parent:
                    broader_terms.append(parent.preferred_label)
                    next_frontier.extend(parent.broader)
            frontier = next_frontier
            depth += 1

        return broader_terms

    def get_narrower(self, term: str, max_depth: int = 2) -> list[str]:
        """Get narrower (child) concepts up to max_depth levels."""
        concept = self.lookup(term)
        if not concept:
            return []

        narrower_terms = []
        frontier = concept.narrower[:]
        visited = {concept.uri}
        depth = 0

        while frontier and depth < max_depth:
            next_frontier = []
            for uri in frontier:
                if uri in visited:
                    continue
                visited.add(uri)
                child = self.concepts.get(uri)
                if child:
                    narrower_terms.append(child.preferred_label)
                    next_frontier.extend(child.narrower)
            frontier = next_frontier
            depth += 1

        return narrower_terms

    def expand_query(
        self,
        term: str,
        include_synonyms: bool = True,
        include_broader: bool = True,
        include_narrower: bool = True,
        include_related: bool = True,
        max_depth: int = 2,
    ) -> dict[str, list[str]]:
        """Expand a query term using all ontological relationships."""
        expansion = {"original": term, "expanded_terms": []}

        if include_synonyms:
            syns = self.get_synonyms(term)
            expansion["synonyms"] = syns
            expansion["expanded_terms"].extend(syns)

        if include_broader:
            broader = self.get_broader(term, max_depth)
            expansion["broader"] = broader
            expansion["expanded_terms"].extend(broader)

        if include_narrower:
            narrower = self.get_narrower(term, max_depth)
            expansion["narrower"] = narrower
            expansion["expanded_terms"].extend(narrower)

        if include_related:
            concept = self.lookup(term)
            if concept:
                related_labels = []
                for uri in concept.related:
                    rel = self.concepts.get(uri)
                    if rel:
                        related_labels.append(rel.preferred_label)
                expansion["related"] = related_labels
                expansion["expanded_terms"].extend(related_labels)

        # Deduplicate
        expansion["expanded_terms"] = list(set(expansion["expanded_terms"]))
        return expansion
```

---

## Loading Standard Ontologies

### SKOS Format (Simple Knowledge Organization System)

```python
from rdflib import Graph as RDFGraph, Namespace, URIRef
from rdflib.namespace import SKOS, RDF, RDFS


class SKOSLoader:
    """Load SKOS ontologies into the OntologyIndex.

    SKOS is the most common format for domain vocabularies:
    - prefLabel: preferred term
    - altLabel: synonyms
    - broader/narrower: hierarchy
    - related: associations
    """

    def load(self, ontology_path: str) -> OntologyIndex:
        """Load a SKOS ontology from a file (RDF/XML, Turtle, JSON-LD)."""
        g = RDFGraph()
        g.parse(ontology_path)

        index = OntologyIndex()

        # Find all SKOS Concepts
        for concept_uri in g.subjects(RDF.type, SKOS.Concept):
            # Preferred label
            pref_label = ""
            for label in g.objects(concept_uri, SKOS.prefLabel):
                pref_label = str(label)
                break

            if not pref_label:
                continue

            # Alternative labels (synonyms)
            synonyms = [
                str(label) for label in g.objects(concept_uri, SKOS.altLabel)
            ]

            # Hidden labels (common misspellings, abbreviations)
            synonyms.extend(
                str(label) for label in g.objects(concept_uri, SKOS.hiddenLabel)
            )

            # Broader concepts
            broader = [
                str(uri) for uri in g.objects(concept_uri, SKOS.broader)
            ]

            # Narrower concepts
            narrower = [
                str(uri) for uri in g.objects(concept_uri, SKOS.narrower)
            ]

            # Related concepts
            related = [
                str(uri) for uri in g.objects(concept_uri, SKOS.related)
            ]

            # Definition
            definition = ""
            for note in g.objects(concept_uri, SKOS.definition):
                definition = str(note)
                break

            concept = OntologyConcept(
                uri=str(concept_uri),
                preferred_label=pref_label,
                synonyms=synonyms,
                broader=broader,
                narrower=narrower,
                related=related,
                definition=definition,
            )
            index.add_concept(concept)

        return index
```

### OWL Ontology Loading

```python
class OWLLoader:
    """Load OWL ontologies for class hierarchy and property relationships.

    OWL provides richer semantics than SKOS:
    - Class hierarchy (subClassOf)
    - Property restrictions (domain, range)
    - Equivalence classes
    - Disjoint classes
    """

    def load(self, ontology_path: str) -> OntologyIndex:
        """Load an OWL ontology."""
        g = RDFGraph()
        g.parse(ontology_path)

        OWL = Namespace("http://www.w3.org/2002/07/owl#")
        index = OntologyIndex()

        # Find all OWL Classes
        for class_uri in g.subjects(RDF.type, OWL.Class):
            # Get labels
            labels = list(g.objects(class_uri, RDFS.label))
            pref_label = str(labels[0]) if labels else str(class_uri).split("#")[-1].split("/")[-1]

            # SubClassOf (broader)
            broader = [
                str(parent) for parent in g.objects(class_uri, RDFS.subClassOf)
                if isinstance(parent, URIRef)
            ]

            # Find subclasses (narrower)
            narrower = [
                str(child) for child in g.subjects(RDFS.subClassOf, class_uri)
                if isinstance(child, URIRef)
            ]

            # Equivalent classes (synonyms)
            synonyms = []
            for equiv in g.objects(class_uri, OWL.equivalentClass):
                equiv_labels = list(g.objects(equiv, RDFS.label))
                synonyms.extend(str(l) for l in equiv_labels)

            # Comments as definitions
            definition = ""
            for comment in g.objects(class_uri, RDFS.comment):
                definition = str(comment)
                break

            concept = OntologyConcept(
                uri=str(class_uri),
                preferred_label=pref_label,
                synonyms=synonyms,
                broader=broader,
                narrower=narrower,
                definition=definition,
            )
            index.add_concept(concept)

        return index
```

---

## Ontology-Enhanced Retrieval Pipeline

```python
class OntologyEnhancedRetriever:
    """Retriever that uses ontology knowledge to improve recall.

    Pipeline:
    1. Extract key terms from the query
    2. Expand terms using the ontology (synonyms, broader, narrower)
    3. Run multiple vector searches (original + expanded terms)
    4. Merge and rerank results
    5. Return top-K documents
    """

    def __init__(
        self,
        ontology_index: OntologyIndex,
        vector_store,
        embedding_model,
        llm=None,
        expansion_weight: float = 0.7,
    ):
        self.ontology = ontology_index
        self.store = vector_store
        self.embedder = embedding_model
        self.llm = llm
        self.expansion_weight = expansion_weight

    def retrieve(self, query: str, top_k: int = 10) -> list[dict]:
        """Retrieve documents using ontology-expanded query."""
        # Step 1: Extract key terms from query
        key_terms = self._extract_key_terms(query)

        # Step 2: Expand each term using ontology
        all_expanded_terms = set()
        expansion_info = {}

        for term in key_terms:
            expansion = self.ontology.expand_query(term, max_depth=2)
            all_expanded_terms.update(expansion["expanded_terms"])
            expansion_info[term] = expansion

        # Step 3: Search with original query
        original_embedding = self.embedder.embed([query])[0]
        original_results = self.store.query(original_embedding, top_k=top_k * 2)

        # Step 4: Search with expanded terms
        expanded_results = []
        if all_expanded_terms:
            expanded_query = query + " " + " ".join(all_expanded_terms)
            expanded_embedding = self.embedder.embed([expanded_query])[0]
            expanded_results = self.store.query(
                expanded_embedding, top_k=top_k * 2
            )

        # Step 5: Merge and rerank
        merged = self._merge_results(
            original_results, expanded_results, top_k
        )

        return merged

    def _extract_key_terms(self, query: str) -> list[str]:
        """Extract key domain terms from the query."""
        if self.llm:
            response = self.llm.invoke(
                "Extract the key domain-specific terms from this query.\n"
                "Return one term per line, no explanations.\n\n"
                f"Query: {query}\n\nTerms:"
            )
            terms = [
                line.strip().strip("- ")
                for line in response.content.strip().split("\n")
                if line.strip()
            ]
            return terms
        else:
            # Simple: split and look up each word/bigram in ontology
            words = query.lower().split()
            terms = []
            for i, word in enumerate(words):
                if self.ontology.lookup(word):
                    terms.append(word)
                # Check bigrams
                if i < len(words) - 1:
                    bigram = f"{word} {words[i + 1]}"
                    if self.ontology.lookup(bigram):
                        terms.append(bigram)
            return terms

    def _merge_results(
        self,
        original: list[dict],
        expanded: list[dict],
        top_k: int,
    ) -> list[dict]:
        """Merge original and expanded results with weighted scoring."""
        scored: dict[str, dict] = {}

        for result in original:
            doc_id = result.get("id", result.get("metadata", {}).get("id", ""))
            scored[doc_id] = {
                **result,
                "combined_score": result.get("score", 0) * 1.0,
            }

        for result in expanded:
            doc_id = result.get("id", result.get("metadata", {}).get("id", ""))
            if doc_id in scored:
                scored[doc_id]["combined_score"] += (
                    result.get("score", 0) * self.expansion_weight
                )
            else:
                scored[doc_id] = {
                    **result,
                    "combined_score": result.get("score", 0) * self.expansion_weight,
                }

        # Sort by combined score
        sorted_results = sorted(
            scored.values(),
            key=lambda x: x["combined_score"],
            reverse=True,
        )

        return sorted_results[:top_k]
```

---

## Key Domain Ontologies

| Ontology | Domain | Concepts | Format | License |
|----------|--------|----------|--------|---------|
| SNOMED CT | Healthcare | 350,000+ | RF2, OWL | Licensed (NLM) |
| FIBO | Finance | 15,000+ | OWL | Open (EDM Council) |
| Gene Ontology | Biology | 45,000+ | OBO, OWL | Open |
| Schema.org | General web | 800+ | RDFa, JSON-LD | Open |
| Dublin Core | Metadata | 15 core | RDF | Open |
| LOINC | Lab tests | 100,000+ | CSV, FHIR | Open (registration) |
| ICD-10 | Diagnoses | 70,000+ | Tabular, OWL | Licensed (WHO) |

---

## Common Pitfalls

1. **Over-expanding queries.** Expanding "heart" to all 500 narrower terms floods retrieval with noise. Limit expansion depth (2 levels max) and term count (20 max).

2. **Using generic ontologies for specialized domains.** Schema.org does not know that "STEMI" is a type of heart attack. Use domain-specific ontologies.

3. **Ignoring ontology updates.** Medical ontologies like SNOMED CT update biannually. Stale ontology data misses new terms and relationships.

4. **Not weighting expansion terms.** Synonyms should have nearly equal weight to the original term, but broader/narrower concepts should be down-weighted. A query about "STEMI" is primarily about STEMI, not all cardiac events.

5. **Loading large ontologies entirely into memory.** SNOMED CT has 350,000+ concepts. Use a triplestore (Fuseki, GraphDB) or database rather than in-memory dictionaries for large ontologies.

6. **Skipping ontology integration when vector similarity seems sufficient.** Benchmark on your actual query distribution. Domain-specific queries often have 20-40% lower recall without ontology expansion.

---

## References

- SNOMED CT: https://www.snomed.org/
- FIBO: https://spec.edmcouncil.org/fibo/
- SKOS Reference: https://www.w3.org/TR/skos-reference/
- OWL 2 Web Ontology Language: https://www.w3.org/TR/owl2-overview/
- RDFLib: https://rdflib.readthedocs.io/
- Schema.org: https://schema.org/
