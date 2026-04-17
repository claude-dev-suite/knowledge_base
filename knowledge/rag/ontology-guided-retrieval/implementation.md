# Ontology-Guided Retrieval -- SPARQL + Vector Hybrid with SNOMED CT and FIBO Examples

## TL;DR

This article provides production implementations of ontology-guided retrieval combining SPARQL triplestore queries with vector similarity search. SPARQL provides precise, structured queries against the ontology (finding all narrower concepts, traversing hierarchies), while vector search provides fuzzy semantic matching. The two are combined in a hybrid retriever that uses SPARQL for ontology traversal and vector search for document retrieval. Includes complete working examples for healthcare (SNOMED CT) and finance (FIBO) domains.

---

## SPARQL Triplestore Integration

### Setting Up a Triplestore

```python
from SPARQLWrapper import SPARQLWrapper, JSON
from dataclasses import dataclass


@dataclass
class TriplestoreConfig:
    endpoint: str          # SPARQL endpoint URL
    update_endpoint: str   # SPARQL update endpoint (optional)
    graph_uri: str         # Named graph URI
    username: str = ""
    password: str = ""


class SPARQLOntologyClient:
    """Query domain ontologies via SPARQL.

    Supports any SPARQL 1.1 endpoint:
    - Apache Jena Fuseki (open source, recommended)
    - GraphDB (Ontotext)
    - Stardog
    - Amazon Neptune (AWS)

    Setup with Fuseki:
    1. Download Apache Jena Fuseki
    2. Upload ontology: fuseki-server --file=ontology.owl /dataset
    3. SPARQL endpoint: http://localhost:3030/dataset/sparql
    """

    def __init__(self, config: TriplestoreConfig):
        self.sparql = SPARQLWrapper(config.endpoint)
        self.sparql.setReturnFormat(JSON)
        self.config = config

        if config.username:
            self.sparql.setCredentials(config.username, config.password)

    def query(self, sparql_query: str) -> list[dict]:
        """Execute a SPARQL query and return results as dicts."""
        self.sparql.setQuery(sparql_query)
        results = self.sparql.query().convert()

        rows = []
        for binding in results["results"]["bindings"]:
            row = {}
            for var in binding:
                row[var] = binding[var]["value"]
            rows.append(row)

        return rows

    def get_concept_by_label(self, label: str) -> list[dict]:
        """Find concepts matching a label (exact or partial)."""
        query = f"""
        PREFIX skos: <http://www.w3.org/2004/02/skos/core#>
        PREFIX rdfs: <http://www.w3.org/2000/01/rdf-schema#>

        SELECT DISTINCT ?concept ?prefLabel ?definition WHERE {{
            {{
                ?concept skos:prefLabel ?prefLabel .
                FILTER(LCASE(STR(?prefLabel)) = LCASE("{label}"))
            }}
            UNION
            {{
                ?concept skos:altLabel ?altLabel .
                FILTER(LCASE(STR(?altLabel)) = LCASE("{label}"))
                ?concept skos:prefLabel ?prefLabel .
            }}
            OPTIONAL {{ ?concept skos:definition ?definition }}
        }}
        LIMIT 10
        """
        return self.query(query)

    def get_synonyms(self, concept_uri: str) -> list[str]:
        """Get all labels (synonyms) for a concept."""
        query = f"""
        PREFIX skos: <http://www.w3.org/2004/02/skos/core#>

        SELECT ?label ?labelType WHERE {{
            {{
                <{concept_uri}> skos:prefLabel ?label .
                BIND("preferred" AS ?labelType)
            }}
            UNION
            {{
                <{concept_uri}> skos:altLabel ?label .
                BIND("alternative" AS ?labelType)
            }}
            UNION
            {{
                <{concept_uri}> skos:hiddenLabel ?label .
                BIND("hidden" AS ?labelType)
            }}
        }}
        """
        results = self.query(query)
        return [r["label"] for r in results]

    def get_broader_concepts(
        self, concept_uri: str, max_depth: int = 3
    ) -> list[dict]:
        """Get broader (parent) concepts up to max_depth levels."""
        query = f"""
        PREFIX skos: <http://www.w3.org/2004/02/skos/core#>

        SELECT ?ancestor ?label ?depth WHERE {{
            <{concept_uri}> skos:broader{{1,{max_depth}}} ?ancestor .
            ?ancestor skos:prefLabel ?label .
            {{
                SELECT ?ancestor (COUNT(?mid) AS ?depth) WHERE {{
                    <{concept_uri}> skos:broader+ ?mid .
                    ?mid skos:broader* ?ancestor .
                    ?ancestor skos:prefLabel [] .
                }}
                GROUP BY ?ancestor
            }}
        }}
        ORDER BY ?depth
        LIMIT 20
        """
        return self.query(query)

    def get_narrower_concepts(
        self, concept_uri: str, max_depth: int = 2
    ) -> list[dict]:
        """Get narrower (child) concepts."""
        query = f"""
        PREFIX skos: <http://www.w3.org/2004/02/skos/core#>

        SELECT ?descendant ?label WHERE {{
            ?descendant skos:broader{{1,{max_depth}}} <{concept_uri}> .
            ?descendant skos:prefLabel ?label .
        }}
        LIMIT 50
        """
        return self.query(query)

    def get_related_concepts(self, concept_uri: str) -> list[dict]:
        """Get related concepts (associative relationships)."""
        query = f"""
        PREFIX skos: <http://www.w3.org/2004/02/skos/core#>

        SELECT ?related ?label WHERE {{
            {{
                <{concept_uri}> skos:related ?related .
            }}
            UNION
            {{
                ?related skos:related <{concept_uri}> .
            }}
            ?related skos:prefLabel ?label .
        }}
        LIMIT 20
        """
        return self.query(query)

    def full_expansion(
        self, label: str, broader_depth: int = 2, narrower_depth: int = 2
    ) -> dict:
        """Complete ontological expansion for a term."""
        # Find the concept
        concepts = self.get_concept_by_label(label)
        if not concepts:
            return {"found": False, "original": label}

        concept_uri = concepts[0]["concept"]
        pref_label = concepts[0]["prefLabel"]

        return {
            "found": True,
            "original": label,
            "concept_uri": concept_uri,
            "preferred_label": pref_label,
            "synonyms": self.get_synonyms(concept_uri),
            "broader": [
                r["label"] for r in self.get_broader_concepts(concept_uri, broader_depth)
            ],
            "narrower": [
                r["label"] for r in self.get_narrower_concepts(concept_uri, narrower_depth)
            ],
            "related": [
                r["label"] for r in self.get_related_concepts(concept_uri)
            ],
        }
```

---

## SNOMED CT Example: Healthcare RAG

```python
class SNOMEDCTRetriever:
    """Ontology-guided retrieval using SNOMED CT for healthcare RAG.

    SNOMED CT contains 350,000+ clinical concepts with:
    - IS-A hierarchy (e.g., STEMI IS-A myocardial infarction)
    - FINDING-SITE relationships (e.g., MI -> heart structure)
    - ASSOCIATED-MORPHOLOGY (e.g., MI -> infarct)
    - Synonyms in 40+ languages

    Access: SNOMED CT requires a license (free for many countries via NLM).
    Load into a triplestore using the RF2 release files + OWL export.
    """

    SNOMED_PREFIXES = """
    PREFIX sct: <http://snomed.info/id/>
    PREFIX skos: <http://www.w3.org/2004/02/skos/core#>
    PREFIX rdfs: <http://www.w3.org/2000/01/rdf-schema#>
    PREFIX owl: <http://www.w3.org/2002/07/owl#>
    """

    def __init__(
        self,
        sparql_client: SPARQLOntologyClient,
        vector_store,
        embedding_model,
    ):
        self.sparql = sparql_client
        self.store = vector_store
        self.embedder = embedding_model

    def retrieve(self, query: str, top_k: int = 10) -> list[dict]:
        """Retrieve medical documents using SNOMED CT expansion."""
        # Step 1: Extract clinical terms from query
        clinical_terms = self._extract_clinical_terms(query)

        # Step 2: Expand each term using SNOMED CT
        all_expansion_terms = set()
        for term in clinical_terms:
            expansion = self._expand_snomed_term(term)
            all_expansion_terms.update(expansion)

        # Step 3: Multi-query retrieval
        queries_to_run = [
            {"text": query, "weight": 1.0},
        ]

        if all_expansion_terms:
            expanded = query + " " + " ".join(all_expansion_terms)
            queries_to_run.append({"text": expanded, "weight": 0.7})

            # Synonym-only query
            synonym_terms = [t for t in all_expansion_terms if len(t.split()) <= 3]
            if synonym_terms:
                syn_query = query + " " + " ".join(synonym_terms[:10])
                queries_to_run.append({"text": syn_query, "weight": 0.85})

        # Step 4: Execute and merge
        merged: dict[str, dict] = {}
        for q in queries_to_run:
            emb = self.embedder.embed([q["text"]])[0]
            results = self.store.query(emb, top_k=top_k)
            for r in results:
                doc_id = r.get("id", "")
                score = r.get("score", 0) * q["weight"]
                if doc_id in merged:
                    merged[doc_id]["score"] = max(merged[doc_id]["score"], score)
                else:
                    merged[doc_id] = {**r, "score": score}

        sorted_results = sorted(
            merged.values(), key=lambda x: x["score"], reverse=True
        )
        return sorted_results[:top_k]

    def _expand_snomed_term(self, term: str) -> set[str]:
        """Expand a clinical term using SNOMED CT."""
        expansion = self.sparql.full_expansion(
            term, broader_depth=2, narrower_depth=1
        )

        terms = set()
        if expansion.get("found"):
            terms.update(expansion.get("synonyms", []))
            terms.update(expansion.get("broader", [])[:5])
            terms.update(expansion.get("narrower", [])[:8])
            terms.update(expansion.get("related", [])[:3])

        terms.discard(term)  # remove original to avoid duplication
        return terms

    def _extract_clinical_terms(self, query: str) -> list[str]:
        """Extract clinical terms from a query.

        In production, use a medical NER model (SciSpacy, BioBERT)
        or the SNOMED CT concept recognition API.
        """
        # Simple approach: check each word and bigram against SNOMED
        words = query.lower().split()
        found = []

        for i in range(len(words)):
            # Check trigrams, bigrams, then unigrams
            for n in [3, 2, 1]:
                if i + n <= len(words):
                    ngram = " ".join(words[i : i + n])
                    results = self.sparql.get_concept_by_label(ngram)
                    if results:
                        found.append(ngram)
                        break

        return found


# Usage example:
# config = TriplestoreConfig(
#     endpoint="http://localhost:3030/snomed/sparql",
#     update_endpoint="",
#     graph_uri="http://snomed.info/sct",
# )
# sparql = SPARQLOntologyClient(config)
# retriever = SNOMEDCTRetriever(sparql, vector_store, embedder)
# results = retriever.retrieve("What are the symptoms of a heart attack?")
```

---

## FIBO Example: Financial RAG

```python
class FIBORetriever:
    """Ontology-guided retrieval using FIBO for financial RAG.

    FIBO (Financial Industry Business Ontology) covers:
    - Financial instruments (bonds, equities, derivatives)
    - Business entities (corporations, trusts, funds)
    - Contract terms (interest rates, maturity, collateral)
    - Regulatory concepts (Basel III, MiFID II)

    FIBO is freely available from the EDM Council.
    Load the OWL files into a triplestore.
    """

    FIBO_PREFIXES = """
    PREFIX fibo-fnd: <https://spec.edmcouncil.org/fibo/ontology/FND/>
    PREFIX fibo-sec: <https://spec.edmcouncil.org/fibo/ontology/SEC/>
    PREFIX fibo-be: <https://spec.edmcouncil.org/fibo/ontology/BE/>
    PREFIX rdfs: <http://www.w3.org/2000/01/rdf-schema#>
    PREFIX owl: <http://www.w3.org/2002/07/owl#>
    PREFIX skos: <http://www.w3.org/2004/02/skos/core#>
    """

    def __init__(
        self,
        sparql_client: SPARQLOntologyClient,
        vector_store,
        embedding_model,
    ):
        self.sparql = sparql_client
        self.store = vector_store
        self.embedder = embedding_model

    def retrieve(self, query: str, top_k: int = 10) -> list[dict]:
        """Retrieve financial documents using FIBO expansion."""
        financial_terms = self._extract_financial_terms(query)

        expansion_terms = set()
        for term in financial_terms:
            expanded = self._expand_fibo_term(term)
            expansion_terms.update(expanded)

        # Build queries
        queries = [{"text": query, "weight": 1.0}]
        if expansion_terms:
            queries.append({
                "text": query + " " + " ".join(expansion_terms),
                "weight": 0.7,
            })

        # Execute
        merged: dict[str, dict] = {}
        for q in queries:
            emb = self.embedder.embed([q["text"]])[0]
            results = self.store.query(emb, top_k=top_k)
            for r in results:
                doc_id = r.get("id", "")
                score = r.get("score", 0) * q["weight"]
                if doc_id in merged:
                    merged[doc_id]["score"] = max(merged[doc_id]["score"], score)
                else:
                    merged[doc_id] = {**r, "score": score}

        return sorted(
            merged.values(), key=lambda x: x["score"], reverse=True
        )[:top_k]

    def _expand_fibo_term(self, term: str) -> set[str]:
        """Expand a financial term using FIBO ontology."""
        # FIBO uses rdfs:label and rdfs:subClassOf instead of SKOS
        query = f"""
        {self.FIBO_PREFIXES}

        SELECT DISTINCT ?label WHERE {{
            ?concept rdfs:label ?matchLabel .
            FILTER(LCASE(STR(?matchLabel)) = LCASE("{term}"))

            {{
                ?concept rdfs:label ?label .
            }}
            UNION
            {{
                ?concept rdfs:subClassOf ?parent .
                ?parent rdfs:label ?label .
            }}
            UNION
            {{
                ?child rdfs:subClassOf ?concept .
                ?child rdfs:label ?label .
            }}
        }}
        LIMIT 20
        """
        results = self.sparql.query(query)
        return {r["label"] for r in results} - {term}

    def _extract_financial_terms(self, query: str) -> list[str]:
        """Extract financial domain terms from a query."""
        financial_keywords = {
            "bond", "equity", "derivative", "option", "swap",
            "interest rate", "maturity", "collateral", "credit",
            "loan", "mortgage", "securitization", "hedge",
            "portfolio", "risk", "compliance", "regulatory",
            "capital", "liquidity", "leverage", "margin",
        }
        words = query.lower().split()
        found = []

        for i in range(len(words)):
            for n in [2, 1]:
                if i + n <= len(words):
                    ngram = " ".join(words[i : i + n])
                    if ngram in financial_keywords:
                        found.append(ngram)
                    else:
                        # Check FIBO
                        results = self.sparql.get_concept_by_label(ngram)
                        if results:
                            found.append(ngram)

        return found
```

---

## Hybrid SPARQL + Vector Architecture

```python
class HybridOntologyRetriever:
    """Production hybrid retriever combining SPARQL and vector search.

    Architecture:
    1. SPARQL triplestore holds the ontology (SNOMED, FIBO, custom)
    2. Vector store holds document embeddings
    3. At query time:
       a. SPARQL expands the query using ontological relationships
       b. Multiple vector searches run with expanded queries
       c. Results are merged with weighted scoring
       d. LLM generates answer from retrieved context

    This separates concerns:
    - Ontology management (SPARQL) from document management (vector store)
    - Structured knowledge (ontology) from unstructured knowledge (documents)
    """

    def __init__(
        self,
        sparql_client: SPARQLOntologyClient,
        vector_store,
        embedding_model,
        llm,
        domain: str = "general",
    ):
        self.sparql = sparql_client
        self.store = vector_store
        self.embedder = embedding_model
        self.llm = llm
        self.domain = domain

    def query(self, question: str, top_k: int = 10) -> dict:
        """Answer a question using hybrid ontology + vector retrieval."""
        # Step 1: Extract and expand terms
        terms = self._extract_domain_terms(question)
        expansions = {}
        for term in terms:
            exp = self.sparql.full_expansion(term)
            if exp.get("found"):
                expansions[term] = exp

        # Step 2: Generate query variants
        query_variants = self._generate_variants(question, expansions)

        # Step 3: Multi-query vector search
        all_docs = {}
        for variant in query_variants:
            emb = self.embedder.embed([variant["query"]])[0]
            results = self.store.query(emb, top_k=top_k)
            for r in results:
                doc_id = r.get("id", "")
                score = r.get("score", 0) * variant["weight"]
                if doc_id not in all_docs or score > all_docs[doc_id]["score"]:
                    all_docs[doc_id] = {**r, "score": score}

        # Step 4: Sort and get top results
        top_docs = sorted(
            all_docs.values(), key=lambda x: x["score"], reverse=True
        )[:top_k]

        # Step 5: Generate answer
        context = "\n\n---\n\n".join(
            d.get("metadata", {}).get("content", str(d))[:500]
            for d in top_docs
        )

        # Include ontological context
        onto_context = self._format_ontology_context(expansions)

        answer = self.llm.invoke(
            f"Answer the question using the retrieved documents and "
            f"domain ontology context below.\n\n"
            f"## Domain Ontology Context\n{onto_context}\n\n"
            f"## Retrieved Documents\n{context}\n\n"
            f"## Question\n{question}"
        )

        return {
            "answer": answer.content,
            "documents": top_docs,
            "expansions": expansions,
            "query_variants": len(query_variants),
        }

    def _extract_domain_terms(self, question: str) -> list[str]:
        """Extract domain terms from the question."""
        response = self.llm.invoke(
            f"Extract key {self.domain} domain terms from this question.\n"
            f"Return one term per line.\n\n"
            f"Question: {question}\n\nTerms:"
        )
        return [
            line.strip().strip("- ")
            for line in response.content.strip().split("\n")
            if line.strip()
        ][:5]

    def _generate_variants(
        self, question: str, expansions: dict
    ) -> list[dict]:
        """Generate query variants from ontological expansions."""
        variants = [{"query": question, "weight": 1.0}]

        all_synonyms = []
        all_broader = []
        all_narrower = []

        for term, exp in expansions.items():
            all_synonyms.extend(exp.get("synonyms", []))
            all_broader.extend(exp.get("broader", [])[:3])
            all_narrower.extend(exp.get("narrower", [])[:5])

        if all_synonyms:
            variants.append({
                "query": question + " " + " ".join(set(all_synonyms)),
                "weight": 0.85,
            })

        if all_narrower:
            variants.append({
                "query": question + " " + " ".join(set(all_narrower)),
                "weight": 0.6,
            })

        if all_broader:
            variants.append({
                "query": question + " " + " ".join(set(all_broader)),
                "weight": 0.4,
            })

        return variants

    @staticmethod
    def _format_ontology_context(expansions: dict) -> str:
        """Format ontological knowledge as context for the LLM."""
        parts = []
        for term, exp in expansions.items():
            part = f"**{term}** (preferred: {exp.get('preferred_label', term)})\n"
            if exp.get("synonyms"):
                part += f"  Also known as: {', '.join(exp['synonyms'][:5])}\n"
            if exp.get("broader"):
                part += f"  Broader concepts: {', '.join(exp['broader'][:3])}\n"
            if exp.get("narrower"):
                part += f"  Specific types: {', '.join(exp['narrower'][:5])}\n"
            parts.append(part)
        return "\n".join(parts) if parts else "No ontological context available."
```

---

## Common Pitfalls

1. **SPARQL endpoint latency.** Each SPARQL query adds 50-200ms. Cache expansion results aggressively (ontologies change infrequently).

2. **Loading the entire ontology into memory.** SNOMED CT is 1.5GB+. Use a triplestore with proper indexing instead of in-memory graphs.

3. **Not handling ontology versioning.** SNOMED CT releases biannually. Track which version your expansions are based on.

4. **Assuming all domains have good ontologies.** Many domains lack comprehensive ontologies. Build a custom SKOS vocabulary from your domain's terminology.

5. **Ignoring multilingual labels.** SNOMED CT includes labels in 40+ languages. Match the ontology language to your document language.

6. **Over-relying on SPARQL for retrieval.** SPARQL provides exact structural queries but not fuzzy matching. Always combine with vector similarity for robust retrieval.

---

## References

- SPARQL 1.1 Query Language: https://www.w3.org/TR/sparql11-query/
- SPARQLWrapper: https://sparqlwrapper.readthedocs.io/
- Apache Jena Fuseki: https://jena.apache.org/documentation/fuseki2/
- SNOMED CT Browser: https://browser.ihtsdotools.org/
- FIBO: https://spec.edmcouncil.org/fibo/
- RDFLib: https://rdflib.readthedocs.io/
