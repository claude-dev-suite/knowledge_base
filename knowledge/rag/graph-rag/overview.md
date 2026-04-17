# Graph RAG -- Microsoft GraphRAG, HippoRAG, and Neo4j Knowledge Graphs

## TL;DR

Graph RAG augments traditional vector-similarity retrieval with knowledge graph structures, enabling multi-hop reasoning, community-level summarization, and entity-relationship awareness that flat chunk retrieval cannot achieve. Microsoft GraphRAG builds a hierarchical community graph from documents and answers global questions by summarizing communities rather than retrieving individual chunks. HippoRAG mirrors human hippocampal memory indexing by pairing a knowledge graph with a Personalized PageRank retrieval algorithm. Neo4j provides the production graph database layer where extracted triples are stored and queried via Cypher. This overview maps the landscape, compares approaches, and provides production-ready Python implementations.

---

## Why Vector-Only Retrieval Falls Short for Complex Questions

### The Fundamental Limitation

Standard RAG embeds document chunks into a vector space and retrieves the top-K most similar chunks for a given query. This works well for factual, single-hop questions ("What is the default port for PostgreSQL?") but fails systematically for:

1. **Global sensemaking questions**: "What are the main themes across all customer feedback from Q4?" requires synthesizing information scattered across hundreds of documents. No single chunk contains the answer.

2. **Multi-hop reasoning**: "Which engineers worked on both the authentication refactor and the payment migration?" requires connecting entities across multiple documents.

3. **Implicit relationship queries**: "How does the pricing team's work affect the infrastructure team's roadmap?" requires understanding entity relationships not stated in any single passage.

4. **Comparative questions**: "How do the European and Asian market strategies differ?" requires pulling and comparing information from disparate sources.

### What Graph RAG Adds

| Capability | Vector RAG | Graph RAG |
|-----------|-----------|-----------|
| Single-hop factual | Excellent | Good |
| Multi-hop reasoning | Poor | Excellent |
| Global summarization | Not possible | Native (community summaries) |
| Entity disambiguation | Poor | Built-in (entity resolution) |
| Relationship traversal | Not possible | Native (graph queries) |
| Explainability | Chunk citations | Entity-relation paths |
| Cost per query | Low ($0.001-0.01) | Medium ($0.01-0.10) |
| Index build cost | Low | High (LLM extraction) |

---

## Microsoft GraphRAG

### Architecture Overview

Microsoft GraphRAG (open-sourced 2024) converts unstructured documents into a hierarchical knowledge graph with community structure, then uses community summaries to answer both local (entity-specific) and global (corpus-wide) questions.

```
Documents
    |
    v
[LLM Entity/Relationship Extraction]
    |
    v
[Knowledge Graph (entities + relationships)]
    |
    v
[Leiden Community Detection]
    |
    v
[Hierarchical Community Summaries]
    |
    v
[Query Engine: Local Search | Global Search | DRIFT Search]
```

### Installation and Setup

```python
# Install Microsoft GraphRAG
# pip install graphrag

import asyncio
import os
from pathlib import Path

# GraphRAG requires an input directory with text files
# and a settings.yaml configuration

# Minimal settings.yaml structure:
SETTINGS_YAML = """
llm:
  api_key: ${GRAPHRAG_API_KEY}
  type: openai_chat
  model: gpt-4o
  max_tokens: 4096
  temperature: 0

embeddings:
  llm:
    api_key: ${GRAPHRAG_API_KEY}
    type: openai_embedding
    model: text-embedding-3-small

chunks:
  size: 1200
  overlap: 100

entity_extraction:
  max_gleanings: 1
  prompt: null  # uses default extraction prompt

community_reports:
  max_length: 2000
  prompt: null  # uses default summarization prompt

claim_extraction:
  enabled: false

storage:
  type: file
  base_dir: "output"

reporting:
  type: file
  base_dir: "logs"
"""


def setup_graphrag_project(project_dir: str, input_dir: str) -> None:
    """Initialize a GraphRAG project with configuration."""
    project_path = Path(project_dir)
    project_path.mkdir(parents=True, exist_ok=True)

    # Write settings
    settings_path = project_path / "settings.yaml"
    settings_path.write_text(SETTINGS_YAML)

    # Ensure input directory exists
    input_path = project_path / input_dir
    input_path.mkdir(parents=True, exist_ok=True)

    print(f"GraphRAG project initialized at {project_path}")
    print(f"Place your .txt files in {input_path}")
    print("Run: graphrag index --root {project_dir}")
```

### Indexing Pipeline

The indexing pipeline is the most expensive part of GraphRAG. It makes LLM calls for every chunk to extract entities and relationships, then builds communities.

```python
import pandas as pd
from dataclasses import dataclass


@dataclass
class Entity:
    name: str
    type: str
    description: str
    source_chunks: list[str]


@dataclass
class Relationship:
    source: str
    target: str
    description: str
    weight: float
    source_chunks: list[str]


@dataclass
class Community:
    id: int
    level: int
    title: str
    summary: str
    entities: list[str]
    relationships: list[tuple[str, str]]


def estimate_indexing_cost(
    num_documents: int,
    avg_tokens_per_doc: int,
    chunk_size: int = 1200,
    cost_per_1k_input: float = 0.0025,  # GPT-4o input
    cost_per_1k_output: float = 0.01,   # GPT-4o output
) -> dict:
    """Estimate the LLM cost of GraphRAG indexing.

    GraphRAG makes approximately:
    - 1 extraction call per chunk (entity/relationship extraction)
    - 1 gleaning call per chunk (optional second pass)
    - 1 summary call per community
    - 1 embedding call per entity description
    """
    total_tokens = num_documents * avg_tokens_per_doc
    num_chunks = total_tokens / chunk_size

    # Extraction: each chunk sent as input, ~500 tokens output
    extraction_input_cost = (num_chunks * chunk_size / 1000) * cost_per_1k_input
    extraction_output_cost = (num_chunks * 500 / 1000) * cost_per_1k_output

    # Community summaries: ~20% of chunks become communities
    num_communities = num_chunks * 0.2
    summary_input_cost = (num_communities * 2000 / 1000) * cost_per_1k_input
    summary_output_cost = (num_communities * 1000 / 1000) * cost_per_1k_output

    total = (
        extraction_input_cost
        + extraction_output_cost
        + summary_input_cost
        + summary_output_cost
    )

    return {
        "num_chunks": int(num_chunks),
        "estimated_communities": int(num_communities),
        "extraction_cost": round(extraction_input_cost + extraction_output_cost, 2),
        "summary_cost": round(summary_input_cost + summary_output_cost, 2),
        "total_estimated_cost": round(total, 2),
        "note": "Does not include embedding costs (~10% of total)",
    }


# Example: 500 documents, ~2000 tokens each
cost = estimate_indexing_cost(500, 2000)
# {'num_chunks': 833, 'estimated_communities': 166,
#  'extraction_cost': 6.24, 'summary_cost': 2.49,
#  'total_estimated_cost': 8.73, ...}
```

### Query Modes

GraphRAG provides three distinct query strategies:

```python
from typing import Literal


def explain_query_modes() -> dict[str, dict]:
    """Document the three GraphRAG query modes and when to use each."""
    return {
        "local_search": {
            "description": (
                "Starts from specific entities mentioned in the query, "
                "traverses their neighborhood in the graph, and generates "
                "an answer from the local subgraph context."
            ),
            "best_for": [
                "Entity-specific questions",
                "Questions about relationships between known entities",
                "Factual lookups with entity names in the query",
            ],
            "example_queries": [
                "What projects has Alice worked on?",
                "How are the auth service and payment service connected?",
                "What are the key features of product X?",
            ],
            "cost_per_query": "Low (1-2 LLM calls)",
            "latency": "1-3 seconds",
        },
        "global_search": {
            "description": (
                "Uses pre-computed community summaries to answer questions "
                "that require understanding the entire corpus. Maps the query "
                "to all community summaries, generates partial answers from "
                "each relevant community, then reduces to a final answer."
            ),
            "best_for": [
                "Corpus-wide summarization",
                "Theme/trend identification",
                "Questions without specific entity references",
            ],
            "example_queries": [
                "What are the main themes in customer feedback?",
                "Summarize the key technical decisions made this quarter",
                "What are the biggest risks across all projects?",
            ],
            "cost_per_query": "High (many LLM calls: map + reduce)",
            "latency": "5-30 seconds depending on community count",
        },
        "drift_search": {
            "description": (
                "Dynamic Reasoning and Inference with Flexible Traversal. "
                "Starts with local search, then iteratively expands the "
                "search frontier based on intermediate findings. Combines "
                "local precision with global coverage."
            ),
            "best_for": [
                "Multi-hop questions",
                "Exploratory queries where the full scope is unknown",
                "Questions that span multiple communities",
            ],
            "example_queries": [
                "How did the database migration affect downstream services?",
                "Trace the evolution of the pricing strategy",
                "What dependencies exist between the frontend and ML teams?",
            ],
            "cost_per_query": "Medium-High (iterative LLM calls)",
            "latency": "3-15 seconds",
        },
    }
```

---

## HippoRAG

### How It Works

HippoRAG (2024) models the human hippocampal memory system. It separates knowledge into a neocortex-like LLM (parametric memory) and a hippocampus-like knowledge graph (non-parametric memory), connected through Personalized PageRank for context-sensitive retrieval.

```
Query
  |
  v
[LLM extracts query entities (pattern separation)]
  |
  v
[Match query entities to KG nodes (pattern completion)]
  |
  v
[Run Personalized PageRank from matched nodes]
  |
  v
[Retrieve passages linked to high-PPR nodes]
  |
  v
[Generate answer from retrieved passages]
```

### Implementation

```python
import numpy as np
from dataclasses import dataclass, field


@dataclass
class KGNode:
    name: str
    node_type: str
    embedding: np.ndarray | None = None
    linked_passages: list[int] = field(default_factory=list)


@dataclass
class KGEdge:
    source: str
    target: str
    relation: str
    weight: float = 1.0


class HippoRAGIndex:
    """Simplified HippoRAG implementation demonstrating the core algorithm.

    The key insight is that Personalized PageRank (PPR) from query-matched
    nodes naturally performs multi-hop retrieval: nodes 2-3 hops away from
    the query entities receive non-zero PPR scores, enabling multi-hop
    reasoning without explicit chain-of-thought decomposition.
    """

    def __init__(self, llm, embedding_model, damping: float = 0.5):
        self.llm = llm
        self.embedding_model = embedding_model
        self.damping = damping
        self.nodes: dict[str, KGNode] = {}
        self.edges: list[KGEdge] = []
        self.passages: list[str] = []
        self.adjacency: dict[str, list[str]] = {}

    def add_passage(self, passage: str, passage_id: int) -> None:
        """Extract entities and relations from a passage and add to KG."""
        # Step 1: LLM extracts (subject, predicate, object) triples
        triples = self._extract_triples(passage)

        for subj, pred, obj in triples:
            # Add nodes
            for entity_name in [subj, obj]:
                if entity_name not in self.nodes:
                    embedding = self.embedding_model.embed(entity_name)
                    self.nodes[entity_name] = KGNode(
                        name=entity_name,
                        node_type="entity",
                        embedding=embedding,
                        linked_passages=[passage_id],
                    )
                else:
                    if passage_id not in self.nodes[entity_name].linked_passages:
                        self.nodes[entity_name].linked_passages.append(passage_id)

            # Add edge
            self.edges.append(KGEdge(subj, obj, pred))

            # Update adjacency
            self.adjacency.setdefault(subj, []).append(obj)
            self.adjacency.setdefault(obj, []).append(subj)

        self.passages.append(passage)

    def retrieve(self, query: str, top_k: int = 5) -> list[str]:
        """Retrieve passages using HippoRAG's PPR-based retrieval."""
        # Step 1: Pattern separation -- extract query entities
        query_entities = self._extract_query_entities(query)

        # Step 2: Pattern completion -- match to KG nodes
        matched_nodes = self._match_to_kg_nodes(query_entities)

        if not matched_nodes:
            return []

        # Step 3: Run Personalized PageRank
        ppr_scores = self._personalized_pagerank(matched_nodes)

        # Step 4: Collect passages from high-PPR nodes
        passage_scores: dict[int, float] = {}
        for node_name, score in ppr_scores.items():
            if node_name in self.nodes:
                for pid in self.nodes[node_name].linked_passages:
                    passage_scores[pid] = passage_scores.get(pid, 0) + score

        # Step 5: Return top-K passages
        sorted_passages = sorted(
            passage_scores.items(), key=lambda x: x[1], reverse=True
        )
        return [self.passages[pid] for pid, _ in sorted_passages[:top_k]]

    def _personalized_pagerank(
        self,
        seed_nodes: list[str],
        max_iterations: int = 50,
        tolerance: float = 1e-6,
    ) -> dict[str, float]:
        """Compute Personalized PageRank from seed nodes.

        PPR naturally performs multi-hop traversal: nodes far from seed
        nodes receive diminishing but non-zero scores, capturing indirect
        relationships.
        """
        all_nodes = list(self.nodes.keys())
        n = len(all_nodes)
        if n == 0:
            return {}

        node_to_idx = {name: i for i, name in enumerate(all_nodes)}

        # Personalization vector: uniform over seed nodes
        personalization = np.zeros(n)
        for node in seed_nodes:
            if node in node_to_idx:
                personalization[node_to_idx[node]] = 1.0 / len(seed_nodes)

        # Transition matrix
        transition = np.zeros((n, n))
        for node in all_nodes:
            neighbors = self.adjacency.get(node, [])
            if neighbors:
                for neighbor in neighbors:
                    if neighbor in node_to_idx:
                        transition[node_to_idx[neighbor]][node_to_idx[node]] = (
                            1.0 / len(neighbors)
                        )

        # Power iteration
        scores = personalization.copy()
        for _ in range(max_iterations):
            new_scores = (
                (1 - self.damping) * personalization
                + self.damping * transition @ scores
            )
            if np.linalg.norm(new_scores - scores) < tolerance:
                break
            scores = new_scores

        return {all_nodes[i]: float(scores[i]) for i in range(n)}

    def _extract_triples(self, passage: str) -> list[tuple[str, str, str]]:
        """Use LLM to extract (subject, predicate, object) triples."""
        prompt = (
            "Extract all factual relationships from this passage as triples.\n"
            "Format each as: (subject, predicate, object)\n"
            "Only extract explicitly stated relationships.\n\n"
            f"Passage: {passage}\n\n"
            "Triples:"
        )
        response = self.llm.invoke(prompt)
        return self._parse_triples(response.content)

    def _extract_query_entities(self, query: str) -> list[str]:
        """Extract named entities from the query."""
        prompt = (
            "Extract the key named entities from this query.\n"
            "Return one entity per line.\n\n"
            f"Query: {query}\n\nEntities:"
        )
        response = self.llm.invoke(prompt)
        return [
            line.strip().strip("- ")
            for line in response.content.strip().split("\n")
            if line.strip()
        ]

    def _match_to_kg_nodes(
        self, query_entities: list[str], threshold: float = 0.75
    ) -> list[str]:
        """Match query entities to KG nodes using embedding similarity."""
        matched = []
        for entity in query_entities:
            entity_emb = self.embedding_model.embed(entity)
            best_match = None
            best_score = -1.0

            for node_name, node in self.nodes.items():
                if node.embedding is not None:
                    score = float(
                        np.dot(entity_emb, node.embedding)
                        / (np.linalg.norm(entity_emb) * np.linalg.norm(node.embedding))
                    )
                    if score > best_score:
                        best_score = score
                        best_match = node_name

            if best_match and best_score >= threshold:
                matched.append(best_match)

        return matched

    @staticmethod
    def _parse_triples(text: str) -> list[tuple[str, str, str]]:
        """Parse LLM output into structured triples."""
        triples = []
        for line in text.strip().split("\n"):
            line = line.strip().strip("()").strip("-").strip()
            parts = [p.strip().strip("'\"") for p in line.split(",")]
            if len(parts) == 3:
                triples.append((parts[0], parts[1], parts[2]))
        return triples
```

---

## HippoRAG vs GraphRAG Comparison

| Aspect | Microsoft GraphRAG | HippoRAG |
|--------|-------------------|-----------|
| Graph construction | LLM entity/relationship extraction | LLM triple extraction (similar) |
| Community detection | Leiden algorithm, hierarchical | Not used |
| Retrieval mechanism | Community summaries + entity lookup | Personalized PageRank from query entities |
| Global queries | Excellent (map-reduce over communities) | Limited (PPR is local-biased) |
| Multi-hop queries | Good (DRIFT search) | Excellent (PPR naturally multi-hop) |
| Index storage | Parquet files or database | Knowledge graph (Neo4j or in-memory) |
| Query latency | High for global, low for local | Low (single PPR computation) |
| Index build cost | Very high (many LLM calls) | High (triple extraction per chunk) |
| Incremental updates | Requires re-indexing | Add new nodes/edges incrementally |

### When to Choose Which

```python
def recommend_graph_rag_approach(
    primary_query_type: str,
    corpus_size_docs: int,
    update_frequency: str,
    budget_per_query_usd: float,
) -> dict:
    """Recommend Graph RAG approach based on requirements."""

    recommendations = {
        "approach": "",
        "reasoning": [],
        "alternatives": [],
    }

    if primary_query_type == "global_summarization":
        recommendations["approach"] = "Microsoft GraphRAG"
        recommendations["reasoning"].append(
            "Global search with community summaries is purpose-built for "
            "corpus-wide summarization questions."
        )
    elif primary_query_type == "multi_hop":
        if budget_per_query_usd < 0.05:
            recommendations["approach"] = "HippoRAG"
            recommendations["reasoning"].append(
                "PPR-based retrieval performs multi-hop traversal in a single "
                "computation, much cheaper than GraphRAG's DRIFT search."
            )
        else:
            recommendations["approach"] = "Microsoft GraphRAG (DRIFT)"
            recommendations["reasoning"].append(
                "DRIFT search provides the best multi-hop accuracy but at "
                "higher cost due to iterative LLM calls."
            )
    elif primary_query_type == "entity_lookup":
        recommendations["approach"] = "Neo4j + Cypher (direct)"
        recommendations["reasoning"].append(
            "For entity-centric queries, direct Cypher queries against a "
            "knowledge graph are faster and cheaper than either framework."
        )

    if update_frequency in ("hourly", "realtime"):
        recommendations["reasoning"].append(
            "HippoRAG supports incremental updates. GraphRAG requires "
            "periodic re-indexing."
        )
        if recommendations["approach"] == "Microsoft GraphRAG":
            recommendations["alternatives"].append(
                "Consider HippoRAG if incremental updates are critical."
            )

    if corpus_size_docs > 10000:
        recommendations["reasoning"].append(
            f"With {corpus_size_docs} documents, GraphRAG indexing will cost "
            f"~${estimate_indexing_cost(corpus_size_docs, 2000)['total_estimated_cost']}. "
            "Budget accordingly."
        )

    return recommendations
```

---

## Neo4j as the Production Graph Layer

Both GraphRAG and HippoRAG ultimately need a graph database for production workloads. Neo4j is the most common choice.

### Basic Neo4j Setup for Graph RAG

```python
from neo4j import GraphDatabase


class Neo4jGraphStore:
    """Production graph store for Graph RAG systems."""

    def __init__(self, uri: str, user: str, password: str, database: str = "neo4j"):
        self.driver = GraphDatabase.driver(uri, auth=(user, password))
        self.database = database

    def close(self) -> None:
        self.driver.close()

    def create_indexes(self) -> None:
        """Create indexes for common Graph RAG access patterns."""
        queries = [
            "CREATE INDEX entity_name IF NOT EXISTS FOR (e:Entity) ON (e.name)",
            "CREATE INDEX entity_type IF NOT EXISTS FOR (e:Entity) ON (e.type)",
            "CREATE INDEX community_id IF NOT EXISTS FOR (c:Community) ON (c.id)",
            (
                "CREATE VECTOR INDEX entity_embedding IF NOT EXISTS "
                "FOR (e:Entity) ON (e.embedding) "
                "OPTIONS {indexConfig: {"
                "  `vector.dimensions`: 1536,"
                "  `vector.similarity_function`: 'cosine'"
                "}}"
            ),
        ]
        with self.driver.session(database=self.database) as session:
            for query in queries:
                session.run(query)

    def upsert_entity(
        self, name: str, entity_type: str, description: str, embedding: list[float]
    ) -> None:
        """Insert or update an entity node."""
        query = """
        MERGE (e:Entity {name: $name})
        SET e.type = $type,
            e.description = $description,
            e.embedding = $embedding,
            e.updated_at = datetime()
        """
        with self.driver.session(database=self.database) as session:
            session.run(
                query,
                name=name,
                type=entity_type,
                description=description,
                embedding=embedding,
            )

    def upsert_relationship(
        self, source: str, target: str, relation: str, weight: float = 1.0
    ) -> None:
        """Insert or update a relationship between entities."""
        query = """
        MATCH (s:Entity {name: $source})
        MATCH (t:Entity {name: $target})
        MERGE (s)-[r:RELATED_TO {type: $relation}]->(t)
        SET r.weight = $weight, r.updated_at = datetime()
        """
        with self.driver.session(database=self.database) as session:
            session.run(
                query, source=source, target=target, relation=relation, weight=weight
            )

    def get_entity_neighborhood(
        self, entity_name: str, hops: int = 2, limit: int = 50
    ) -> list[dict]:
        """Retrieve the local neighborhood of an entity for local search."""
        query = f"""
        MATCH path = (start:Entity {{name: $name}})-[*1..{hops}]-(neighbor:Entity)
        WITH neighbor, min(length(path)) AS distance
        RETURN neighbor.name AS name,
               neighbor.type AS type,
               neighbor.description AS description,
               distance
        ORDER BY distance, neighbor.name
        LIMIT $limit
        """
        with self.driver.session(database=self.database) as session:
            result = session.run(query, name=entity_name, limit=limit)
            return [dict(record) for record in result]

    def vector_search_entities(
        self, query_embedding: list[float], top_k: int = 10
    ) -> list[dict]:
        """Find entities by vector similarity (Neo4j 5.11+)."""
        query = """
        CALL db.index.vector.queryNodes('entity_embedding', $top_k, $embedding)
        YIELD node, score
        RETURN node.name AS name,
               node.type AS type,
               node.description AS description,
               score
        """
        with self.driver.session(database=self.database) as session:
            result = session.run(
                query, top_k=top_k, embedding=query_embedding
            )
            return [dict(record) for record in result]
```

---

## Production Architecture

### Recommended Stack

```
                        +-------------------+
                        |   Query Router    |
                        | (LangGraph agent) |
                        +--------+----------+
                                 |
              +------------------+------------------+
              |                  |                   |
    +---------v------+  +-------v--------+  +-------v--------+
    | Local Search   |  | Global Search  |  | DRIFT Search   |
    | (entity-based) |  | (community     |  | (iterative     |
    |                |  |  map-reduce)   |  |  expansion)    |
    +--------+-------+  +-------+--------+  +-------+--------+
              |                  |                   |
              +------------------+------------------+
                                 |
                    +------------v-----------+
                    |       Neo4j            |
                    |  (entities, relations, |
                    |   communities,         |
                    |   vector index)        |
                    +------------------------+
```

### End-to-End Pipeline

```python
from enum import Enum


class SearchMode(Enum):
    LOCAL = "local"
    GLOBAL = "global"
    DRIFT = "drift"
    AUTO = "auto"


class GraphRAGPipeline:
    """Production Graph RAG pipeline combining GraphRAG concepts with Neo4j."""

    def __init__(
        self,
        neo4j_store: Neo4jGraphStore,
        llm,
        embedding_model,
    ):
        self.store = neo4j_store
        self.llm = llm
        self.embedding_model = embedding_model

    def query(
        self, question: str, mode: SearchMode = SearchMode.AUTO
    ) -> dict:
        """Answer a question using the appropriate search strategy."""
        if mode == SearchMode.AUTO:
            mode = self._classify_query(question)

        if mode == SearchMode.LOCAL:
            return self._local_search(question)
        elif mode == SearchMode.GLOBAL:
            return self._global_search(question)
        elif mode == SearchMode.DRIFT:
            return self._drift_search(question)
        else:
            raise ValueError(f"Unknown search mode: {mode}")

    def _classify_query(self, question: str) -> SearchMode:
        """Route query to the best search mode."""
        response = self.llm.invoke(
            "Classify this question for a knowledge graph search:\n"
            f"Question: {question}\n\n"
            "Categories:\n"
            "- LOCAL: asks about specific entities or their direct relationships\n"
            "- GLOBAL: asks about themes, trends, or corpus-wide patterns\n"
            "- DRIFT: asks about chains of influence, indirect effects, or "
            "multi-step connections\n\n"
            "Respond with exactly one word: LOCAL, GLOBAL, or DRIFT"
        )
        mode_str = response.content.strip().upper()
        return SearchMode[mode_str] if mode_str in SearchMode.__members__ else SearchMode.LOCAL

    def _local_search(self, question: str) -> dict:
        """Entity-centric search using graph neighborhood."""
        # Extract entities from question
        query_embedding = self.embedding_model.embed(question)
        matched_entities = self.store.vector_search_entities(query_embedding, top_k=5)

        # Get neighborhood context
        context_parts = []
        for entity in matched_entities:
            neighbors = self.store.get_entity_neighborhood(entity["name"], hops=2)
            entity_context = f"Entity: {entity['name']} ({entity['type']})\n"
            entity_context += f"Description: {entity['description']}\n"
            entity_context += "Connected to:\n"
            for n in neighbors[:10]:
                entity_context += f"  - {n['name']} ({n['type']}): {n['description']}\n"
            context_parts.append(entity_context)

        context = "\n---\n".join(context_parts)
        answer = self.llm.invoke(
            f"Based on this knowledge graph context, answer the question.\n\n"
            f"Context:\n{context}\n\nQuestion: {question}"
        )

        return {
            "answer": answer.content,
            "mode": "local",
            "entities_used": [e["name"] for e in matched_entities],
            "context_size": len(context),
        }

    def _global_search(self, question: str) -> dict:
        """Community-summary-based search for global questions."""
        # In production, this would query pre-computed community summaries
        # stored in Neo4j Community nodes
        query = """
        MATCH (c:Community)
        RETURN c.id AS id, c.title AS title, c.summary AS summary
        ORDER BY c.level DESC, c.id
        """
        with self.store.driver.session(database=self.store.database) as session:
            communities = [dict(r) for r in session.run(query)]

        # Map phase: score each community's relevance
        relevant_summaries = []
        for community in communities:
            relevance = self.llm.invoke(
                f"Rate 0-10 how relevant this community summary is to the question.\n"
                f"Question: {question}\n"
                f"Summary: {community['summary']}\n"
                f"Respond with just the number."
            )
            score = int(relevance.content.strip())
            if score >= 5:
                relevant_summaries.append(
                    {"summary": community["summary"], "score": score}
                )

        # Reduce phase: synthesize answer from relevant communities
        summaries_text = "\n\n".join(
            f"[Relevance {s['score']}/10]: {s['summary']}"
            for s in sorted(relevant_summaries, key=lambda x: x["score"], reverse=True)
        )
        answer = self.llm.invoke(
            f"Synthesize an answer from these community summaries.\n\n"
            f"Summaries:\n{summaries_text}\n\nQuestion: {question}"
        )

        return {
            "answer": answer.content,
            "mode": "global",
            "communities_consulted": len(communities),
            "communities_relevant": len(relevant_summaries),
        }

    def _drift_search(self, question: str) -> dict:
        """Iterative graph traversal expanding from initial entities."""
        query_embedding = self.embedding_model.embed(question)
        seed_entities = self.store.vector_search_entities(query_embedding, top_k=3)

        visited = set()
        all_context = []
        frontier = [e["name"] for e in seed_entities]
        max_iterations = 3

        for iteration in range(max_iterations):
            new_frontier = []
            for entity_name in frontier:
                if entity_name in visited:
                    continue
                visited.add(entity_name)

                neighbors = self.store.get_entity_neighborhood(
                    entity_name, hops=1, limit=10
                )
                context_entry = {
                    "entity": entity_name,
                    "neighbors": [n["name"] for n in neighbors],
                    "descriptions": [n["description"] for n in neighbors],
                }
                all_context.append(context_entry)
                new_frontier.extend(
                    n["name"] for n in neighbors if n["name"] not in visited
                )

            # Check if we have enough context
            context_text = self._format_drift_context(all_context)
            sufficiency = self.llm.invoke(
                f"Given this graph context, can you fully answer the question?\n"
                f"Context:\n{context_text}\n\nQuestion: {question}\n\n"
                f"Respond YES or NO."
            )
            if "YES" in sufficiency.content.upper():
                break

            frontier = new_frontier[:5]  # limit expansion

        context_text = self._format_drift_context(all_context)
        answer = self.llm.invoke(
            f"Based on this graph traversal context, answer the question.\n\n"
            f"Context:\n{context_text}\n\nQuestion: {question}"
        )

        return {
            "answer": answer.content,
            "mode": "drift",
            "entities_visited": list(visited),
            "iterations": iteration + 1,
        }

    @staticmethod
    def _format_drift_context(context_entries: list[dict]) -> str:
        parts = []
        for entry in context_entries:
            part = f"Entity: {entry['entity']}\n"
            for name, desc in zip(entry["neighbors"], entry["descriptions"]):
                part += f"  -> {name}: {desc}\n"
            parts.append(part)
        return "\n".join(parts)
```

---

## Common Pitfalls

1. **Underestimating indexing cost.** GraphRAG indexing can cost $10-100+ for large corpora. Always run `estimate_indexing_cost()` before committing. Consider indexing a representative sample first.

2. **Using global search for local questions.** Global search consults all community summaries (expensive). If the question mentions specific entities, local search is 10x cheaper and faster.

3. **Skipping entity resolution.** Both GraphRAG and HippoRAG extract entities per-chunk. Without entity resolution, "React.js", "ReactJS", and "React" become three separate nodes. See `entity-resolution/` for solutions.

4. **Ignoring graph quality metrics.** After building the graph, check: average node degree (should be 3-10), number of disconnected components (should be few), and largest community size (should not dominate). Poor graph structure produces poor retrieval.

5. **Not caching community summaries.** Global search generates partial answers per community. Cache these (keyed by community ID + summary hash) to avoid redundant LLM calls for similar queries.

6. **Choosing Graph RAG when vector RAG suffices.** If 90%+ of your queries are single-hop factual lookups, vector RAG with reranking is simpler, cheaper, and often more accurate. Graph RAG shines for multi-hop, global, and relationship queries.

---

## References

- Edge et al. "From Local to Global: A Graph RAG Approach to Query-Focused Summarization" (Microsoft Research, 2024)
- Gutierrez et al. "HippoRAG: Neurobiologically Inspired Long-Term Memory for Large Language Models" (2024)
- Microsoft GraphRAG: https://microsoft.github.io/graphrag/
- Neo4j Vector Index: https://neo4j.com/docs/cypher-manual/current/indexes/semantic-indexes/vector-indexes/
- Leiden Community Detection: Traag et al. "From Louvain to Leiden" (Scientific Reports, 2019)
