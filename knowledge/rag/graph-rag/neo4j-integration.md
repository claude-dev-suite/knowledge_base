# Graph RAG -- Neo4j Integration with LangChain and LlamaIndex

## TL;DR

Neo4j is the most widely used graph database for Graph RAG, providing native Cypher queries, vector indexes, and full-text search in a single system. LangChain's `GraphCypherQAChain` converts natural language to Cypher queries via an LLM, enabling text-to-graph-query pipelines. LlamaIndex's `PropertyGraphIndex` provides a higher-level abstraction that handles extraction, storage, and retrieval as a unified index. This article covers production Neo4j setup, both framework integrations, hybrid vector+graph retrieval, and performance optimization patterns.

---

## Neo4j Setup for Graph RAG

### Schema Design

A well-designed Neo4j schema for Graph RAG needs to support entity storage, relationship traversal, community grouping, and vector similarity search.

```python
from neo4j import GraphDatabase
from typing import Any


class GraphRAGNeo4jSchema:
    """Production Neo4j schema for Graph RAG workloads.

    Node types:
    - Entity: extracted named entities with embeddings
    - Document: source documents
    - Chunk: document chunks linked to entities
    - Community: Leiden communities with summaries

    Relationship types:
    - RELATED_TO: entity-to-entity relationships
    - MENTIONED_IN: entity-to-chunk provenance
    - PART_OF: chunk-to-document hierarchy
    - BELONGS_TO: entity-to-community membership
    """

    SCHEMA_QUERIES = [
        # Constraints (unique identifiers)
        "CREATE CONSTRAINT entity_name IF NOT EXISTS FOR (e:Entity) REQUIRE e.name IS UNIQUE",
        "CREATE CONSTRAINT doc_id IF NOT EXISTS FOR (d:Document) REQUIRE d.id IS UNIQUE",
        "CREATE CONSTRAINT chunk_id IF NOT EXISTS FOR (c:Chunk) REQUIRE c.id IS UNIQUE",
        "CREATE CONSTRAINT community_id IF NOT EXISTS FOR (cm:Community) REQUIRE cm.id IS UNIQUE",

        # Indexes for common lookups
        "CREATE INDEX entity_type IF NOT EXISTS FOR (e:Entity) ON (e.type)",
        "CREATE INDEX chunk_doc IF NOT EXISTS FOR (c:Chunk) ON (c.document_id)",
        "CREATE INDEX community_level IF NOT EXISTS FOR (cm:Community) ON (cm.level)",

        # Vector index for entity embeddings
        """CREATE VECTOR INDEX entity_embedding IF NOT EXISTS
        FOR (e:Entity) ON (e.embedding)
        OPTIONS {indexConfig: {
            `vector.dimensions`: 1536,
            `vector.similarity_function`: 'cosine'
        }}""",

        # Vector index for chunk embeddings (for hybrid retrieval)
        """CREATE VECTOR INDEX chunk_embedding IF NOT EXISTS
        FOR (c:Chunk) ON (c.embedding)
        OPTIONS {indexConfig: {
            `vector.dimensions`: 1536,
            `vector.similarity_function`: 'cosine'
        }}""",

        # Full-text index for entity search
        """CREATE FULLTEXT INDEX entity_fulltext IF NOT EXISTS
        FOR (e:Entity) ON EACH [e.name, e.description]""",
    ]

    def __init__(self, uri: str, user: str, password: str, database: str = "neo4j"):
        self.driver = GraphDatabase.driver(uri, auth=(user, password))
        self.database = database

    def initialize_schema(self) -> None:
        """Create all constraints, indexes, and vector indexes."""
        with self.driver.session(database=self.database) as session:
            for query in self.SCHEMA_QUERIES:
                try:
                    session.run(query)
                except Exception as e:
                    # Some constraints may already exist
                    print(f"Schema query note: {e}")
        print("Schema initialized successfully.")

    def get_schema_summary(self) -> dict:
        """Return current schema statistics."""
        queries = {
            "entity_count": "MATCH (e:Entity) RETURN count(e) AS count",
            "relationship_count": "MATCH ()-[r:RELATED_TO]->() RETURN count(r) AS count",
            "chunk_count": "MATCH (c:Chunk) RETURN count(c) AS count",
            "document_count": "MATCH (d:Document) RETURN count(d) AS count",
            "community_count": "MATCH (cm:Community) RETURN count(cm) AS count",
            "avg_degree": """
                MATCH (e:Entity)-[r]-()
                WITH e, count(r) AS degree
                RETURN avg(degree) AS avg_degree
            """,
        }
        stats = {}
        with self.driver.session(database=self.database) as session:
            for key, query in queries.items():
                result = session.run(query).single()
                stats[key] = result[0] if result else 0
        return stats

    def close(self) -> None:
        self.driver.close()
```

### Bulk Data Loading

For large Graph RAG indexes, use batch operations rather than individual queries.

```python
from dataclasses import dataclass


@dataclass
class EntityRecord:
    name: str
    type: str
    description: str
    embedding: list[float]
    source_chunk_ids: list[str]


@dataclass
class RelationshipRecord:
    source_name: str
    target_name: str
    relation_type: str
    description: str
    weight: float


class Neo4jBulkLoader:
    """Efficient bulk loading for Graph RAG data into Neo4j.

    Uses UNWIND for batch operations, which is 10-50x faster than
    individual MERGE statements for large datasets.
    """

    def __init__(self, driver, database: str = "neo4j", batch_size: int = 500):
        self.driver = driver
        self.database = database
        self.batch_size = batch_size

    def load_entities(self, entities: list[EntityRecord]) -> int:
        """Bulk upsert entities with embeddings."""
        query = """
        UNWIND $batch AS entity
        MERGE (e:Entity {name: entity.name})
        SET e.type = entity.type,
            e.description = entity.description,
            e.embedding = entity.embedding,
            e.updated_at = datetime()
        WITH e, entity
        UNWIND entity.source_chunk_ids AS chunk_id
        MATCH (c:Chunk {id: chunk_id})
        MERGE (e)-[:MENTIONED_IN]->(c)
        """
        total = 0
        for i in range(0, len(entities), self.batch_size):
            batch = [
                {
                    "name": e.name,
                    "type": e.type,
                    "description": e.description,
                    "embedding": e.embedding,
                    "source_chunk_ids": e.source_chunk_ids,
                }
                for e in entities[i : i + self.batch_size]
            ]
            with self.driver.session(database=self.database) as session:
                session.run(query, batch=batch)
            total += len(batch)
        return total

    def load_relationships(self, relationships: list[RelationshipRecord]) -> int:
        """Bulk upsert relationships between entities."""
        query = """
        UNWIND $batch AS rel
        MATCH (source:Entity {name: rel.source_name})
        MATCH (target:Entity {name: rel.target_name})
        MERGE (source)-[r:RELATED_TO {type: rel.relation_type}]->(target)
        SET r.description = rel.description,
            r.weight = rel.weight,
            r.updated_at = datetime()
        """
        total = 0
        for i in range(0, len(relationships), self.batch_size):
            batch = [
                {
                    "source_name": r.source_name,
                    "target_name": r.target_name,
                    "relation_type": r.relation_type,
                    "description": r.description,
                    "weight": r.weight,
                }
                for r in relationships[i : i + self.batch_size]
            ]
            with self.driver.session(database=self.database) as session:
                session.run(query, batch=batch)
            total += len(batch)
        return total

    def load_communities(
        self, communities: list[dict], entity_memberships: list[tuple[str, int]]
    ) -> int:
        """Bulk load community nodes and entity memberships."""
        # Create community nodes
        community_query = """
        UNWIND $batch AS cm
        MERGE (c:Community {id: cm.id})
        SET c.level = cm.level,
            c.title = cm.title,
            c.summary = cm.summary,
            c.entity_count = cm.entity_count
        """
        with self.driver.session(database=self.database) as session:
            session.run(community_query, batch=communities)

        # Create BELONGS_TO relationships
        membership_query = """
        UNWIND $batch AS mem
        MATCH (e:Entity {name: mem.entity_name})
        MATCH (c:Community {id: mem.community_id})
        MERGE (e)-[:BELONGS_TO]->(c)
        """
        for i in range(0, len(entity_memberships), self.batch_size):
            batch = [
                {"entity_name": name, "community_id": cid}
                for name, cid in entity_memberships[i : i + self.batch_size]
            ]
            with self.driver.session(database=self.database) as session:
                session.run(membership_query, batch=batch)

        return len(communities)
```

---

## LangChain GraphCypherQAChain

LangChain's `GraphCypherQAChain` converts natural language questions into Cypher queries, executes them against Neo4j, and generates natural language answers from the results.

### Basic Setup

```python
from langchain_community.graphs import Neo4jGraph
from langchain_openai import ChatOpenAI
from langchain.chains import GraphCypherQAChain


def create_cypher_qa_chain(
    neo4j_uri: str,
    neo4j_user: str,
    neo4j_password: str,
    model: str = "gpt-4o",
    temperature: float = 0,
    verbose: bool = True,
) -> GraphCypherQAChain:
    """Create a LangChain GraphCypherQAChain for natural language graph queries.

    The chain works in three steps:
    1. LLM generates a Cypher query from the natural language question
    2. Cypher query is executed against Neo4j
    3. LLM generates a natural language answer from the query results
    """
    # Connect to Neo4j
    graph = Neo4jGraph(
        url=neo4j_uri,
        username=neo4j_user,
        password=neo4j_password,
    )

    # Refresh schema (LLM needs this to generate valid Cypher)
    graph.refresh_schema()

    # Create the chain
    llm = ChatOpenAI(model=model, temperature=temperature)

    chain = GraphCypherQAChain.from_llm(
        llm=llm,
        graph=graph,
        verbose=verbose,
        return_intermediate_steps=True,
        validate_cypher=True,  # validates syntax before execution
        top_k=20,  # limit results returned from Cypher
    )

    return chain


# Usage
# chain = create_cypher_qa_chain("bolt://localhost:7687", "neo4j", "password")
# result = chain.invoke({"query": "What entities are connected to the auth service?"})
# print(result["result"])
# print(result["intermediate_steps"])  # shows generated Cypher
```

### Custom Cypher Generation Prompt

The default prompt often generates invalid Cypher for Graph RAG schemas. Customize it with your schema details and example queries.

```python
from langchain_core.prompts import PromptTemplate

CUSTOM_CYPHER_PROMPT = PromptTemplate(
    input_variables=["schema", "question"],
    template="""You are an expert Neo4j Cypher query generator for a Graph RAG knowledge graph.

## Graph Schema
{schema}

## Node Types and Properties
- Entity: name (string, unique), type (string), description (string), embedding (vector)
- Chunk: id (string, unique), text (string), document_id (string), embedding (vector)
- Document: id (string, unique), title (string), source (string)
- Community: id (integer, unique), level (integer), title (string), summary (string)

## Relationship Types
- (Entity)-[:RELATED_TO {{type: string, description: string, weight: float}}]->(Entity)
- (Entity)-[:MENTIONED_IN]->(Chunk)
- (Chunk)-[:PART_OF]->(Document)
- (Entity)-[:BELONGS_TO]->(Community)

## Cypher Query Rules
1. Always use LIMIT to cap results (default 20)
2. For entity lookups, use case-insensitive matching: toLower(e.name) CONTAINS toLower($search)
3. For relationship queries, traverse with variable-length paths: -[:RELATED_TO*1..3]-
4. Return structured data: entity names, types, descriptions, and relationship details
5. Do not use OPTIONAL MATCH unless specifically needed
6. For community queries, filter by level for the desired granularity

## Example Queries
Question: "What is connected to React?"
Cypher: MATCH (e:Entity)-[r:RELATED_TO]-(neighbor:Entity)
WHERE toLower(e.name) CONTAINS 'react'
RETURN e.name AS entity, type(r) AS rel_type, r.description AS description, neighbor.name AS connected_to
LIMIT 20

Question: "What are the main communities?"
Cypher: MATCH (cm:Community)
WHERE cm.level = 2
RETURN cm.title AS community, cm.summary AS summary, cm.entity_count AS size
ORDER BY cm.entity_count DESC
LIMIT 10

Question: "How are authentication and payments related?"
Cypher: MATCH path = shortestPath(
  (a:Entity {{name: 'authentication'}})-[:RELATED_TO*..5]-(b:Entity {{name: 'payments'}})
)
RETURN [n IN nodes(path) | n.name] AS path_entities,
       [r IN relationships(path) | r.description] AS path_descriptions

## Task
Generate a Cypher query for this question. Return ONLY the Cypher query, no explanation.

Question: {question}
Cypher:""",
)


CUSTOM_QA_PROMPT = PromptTemplate(
    input_variables=["context", "question"],
    template="""You are a helpful assistant answering questions about a knowledge graph.

## Graph Query Results
{context}

## Question
{question}

Answer based ONLY on the graph query results above. If the results are empty or
insufficient, say so. Include specific entity names and relationships from the results.

Answer:""",
)


def create_custom_cypher_chain(
    neo4j_uri: str,
    neo4j_user: str,
    neo4j_password: str,
) -> GraphCypherQAChain:
    """Create a chain with customized prompts for Graph RAG schema."""
    graph = Neo4jGraph(
        url=neo4j_uri,
        username=neo4j_user,
        password=neo4j_password,
    )
    graph.refresh_schema()

    llm = ChatOpenAI(model="gpt-4o", temperature=0)

    chain = GraphCypherQAChain.from_llm(
        llm=llm,
        graph=graph,
        cypher_prompt=CUSTOM_CYPHER_PROMPT,
        qa_prompt=CUSTOM_QA_PROMPT,
        validate_cypher=True,
        return_intermediate_steps=True,
        top_k=20,
    )

    return chain
```

### Hybrid Vector + Cypher Retrieval

Combine vector similarity search with graph traversal for more comprehensive retrieval.

```python
from langchain_core.runnables import RunnablePassthrough, RunnableLambda
from langchain_openai import OpenAIEmbeddings


class HybridGraphRetriever:
    """Combine Neo4j vector search with Cypher graph traversal.

    Strategy:
    1. Vector search finds semantically similar entities
    2. Cypher traversal expands to related entities (1-2 hops)
    3. Results are merged and deduplicated
    4. LLM generates answer from combined context
    """

    def __init__(
        self,
        neo4j_uri: str,
        neo4j_user: str,
        neo4j_password: str,
        embedding_model=None,
    ):
        self.driver = GraphDatabase.driver(
            neo4j_uri, auth=(neo4j_user, neo4j_password)
        )
        self.embedding_model = embedding_model or OpenAIEmbeddings(
            model="text-embedding-3-small"
        )

    def retrieve(self, query: str, top_k: int = 10) -> list[dict]:
        """Hybrid retrieval: vector similarity + graph traversal."""
        # Step 1: Vector search for seed entities
        query_embedding = self.embedding_model.embed_query(query)
        seed_entities = self._vector_search(query_embedding, top_k=5)

        # Step 2: Graph traversal from seed entities
        expanded_entities = []
        for entity in seed_entities:
            neighbors = self._traverse_neighborhood(entity["name"], hops=2)
            expanded_entities.extend(neighbors)

        # Step 3: Merge and deduplicate
        seen = set()
        merged = []
        for entity in seed_entities + expanded_entities:
            if entity["name"] not in seen:
                seen.add(entity["name"])
                merged.append(entity)

        # Step 4: Get source chunks for provenance
        for entity in merged:
            entity["source_chunks"] = self._get_source_chunks(entity["name"])

        return merged[:top_k]

    def _vector_search(
        self, embedding: list[float], top_k: int = 5
    ) -> list[dict]:
        """Find entities by vector similarity."""
        query = """
        CALL db.index.vector.queryNodes('entity_embedding', $top_k, $embedding)
        YIELD node, score
        RETURN node.name AS name,
               node.type AS type,
               node.description AS description,
               score
        ORDER BY score DESC
        """
        with self.driver.session() as session:
            result = session.run(query, top_k=top_k, embedding=embedding)
            return [dict(record) for record in result]

    def _traverse_neighborhood(
        self, entity_name: str, hops: int = 2
    ) -> list[dict]:
        """Traverse entity neighborhood via Cypher."""
        query = f"""
        MATCH (start:Entity {{name: $name}})
        MATCH (start)-[:RELATED_TO*1..{hops}]-(neighbor:Entity)
        WITH DISTINCT neighbor, min(length(
            shortestPath((start)-[:RELATED_TO*]-(neighbor))
        )) AS distance
        RETURN neighbor.name AS name,
               neighbor.type AS type,
               neighbor.description AS description,
               distance
        ORDER BY distance
        LIMIT 20
        """
        with self.driver.session() as session:
            result = session.run(query, name=entity_name)
            return [dict(record) for record in result]

    def _get_source_chunks(self, entity_name: str, limit: int = 3) -> list[str]:
        """Get source text chunks for an entity (provenance)."""
        query = """
        MATCH (e:Entity {name: $name})-[:MENTIONED_IN]->(c:Chunk)
        RETURN c.text AS text
        LIMIT $limit
        """
        with self.driver.session() as session:
            result = session.run(query, name=entity_name, limit=limit)
            return [record["text"] for record in result]

    def close(self) -> None:
        self.driver.close()
```

---

## LlamaIndex PropertyGraphIndex

LlamaIndex's `PropertyGraphIndex` provides a higher-level abstraction that handles the full Graph RAG pipeline: extraction, storage, and retrieval as a single index.

### Basic PropertyGraphIndex Setup

```python
from llama_index.core import (
    PropertyGraphIndex,
    SimpleDirectoryReader,
    Settings,
)
from llama_index.llms.openai import OpenAI
from llama_index.embeddings.openai import OpenAIEmbedding
from llama_index.graph_stores.neo4j import Neo4jPropertyGraphStore


def create_property_graph_index(
    documents_dir: str,
    neo4j_uri: str,
    neo4j_user: str,
    neo4j_password: str,
    neo4j_database: str = "neo4j",
) -> PropertyGraphIndex:
    """Create a LlamaIndex PropertyGraphIndex backed by Neo4j.

    PropertyGraphIndex handles:
    1. Document chunking
    2. LLM-based entity and relationship extraction
    3. Storage in Neo4j as a labeled property graph
    4. Multiple retrieval strategies (keyword, vector, Cypher, hybrid)
    """
    # Configure LLM and embedding model
    Settings.llm = OpenAI(model="gpt-4o", temperature=0)
    Settings.embed_model = OpenAIEmbedding(model="text-embedding-3-small")

    # Load documents
    documents = SimpleDirectoryReader(documents_dir).load_data()

    # Create Neo4j graph store
    graph_store = Neo4jPropertyGraphStore(
        url=neo4j_uri,
        username=neo4j_user,
        password=neo4j_password,
        database=neo4j_database,
    )

    # Build index (this triggers extraction and storage)
    index = PropertyGraphIndex.from_documents(
        documents,
        property_graph_store=graph_store,
        show_progress=True,
    )

    return index


# Usage:
# index = create_property_graph_index(
#     "./docs", "bolt://localhost:7687", "neo4j", "password"
# )
# query_engine = index.as_query_engine()
# response = query_engine.query("How are the auth and payment services related?")
```

### Custom Extraction with SchemaLLMPathExtractor

Control what entities and relationships are extracted by defining a schema.

```python
from llama_index.core.indices.property_graph import SchemaLLMPathExtractor
from typing import Literal


def create_schema_guided_index(
    documents_dir: str,
    graph_store,
) -> PropertyGraphIndex:
    """Create a PropertyGraphIndex with schema-guided extraction.

    Schema-guided extraction improves quality by:
    1. Restricting entity types to a known set
    2. Defining valid relationship types
    3. Reducing hallucinated entities and relationships
    """
    # Define the schema
    entities = Literal["PERSON", "ORGANIZATION", "TECHNOLOGY", "PROJECT", "CONCEPT"]
    relations = Literal[
        "WORKS_ON", "USES", "DEPENDS_ON", "PART_OF",
        "CREATED_BY", "RELATED_TO", "IMPLEMENTS"
    ]

    # Create the extractor
    kg_extractor = SchemaLLMPathExtractor(
        llm=OpenAI(model="gpt-4o", temperature=0),
        possible_entities=entities,
        possible_relations=relations,
        strict=False,  # allow entities outside the schema if clearly valid
        num_workers=4,  # parallel extraction
        max_triplets_per_chunk=15,
    )

    documents = SimpleDirectoryReader(documents_dir).load_data()

    index = PropertyGraphIndex.from_documents(
        documents,
        property_graph_store=graph_store,
        kg_extractors=[kg_extractor],
        show_progress=True,
    )

    return index
```

### Multiple Retrieval Strategies

LlamaIndex PropertyGraphIndex supports several retrieval strategies that can be combined.

```python
from llama_index.core.indices.property_graph import (
    LLMSynonymRetriever,
    VectorContextRetriever,
    TextToCypherRetriever,
)


def create_multi_strategy_query_engine(index: PropertyGraphIndex):
    """Create a query engine that combines multiple retrieval strategies.

    Strategies:
    1. LLMSynonymRetriever: LLM generates synonyms/related terms for the query,
       then does keyword matching against entity names
    2. VectorContextRetriever: uses embedding similarity to find relevant entities
       and their neighborhoods
    3. TextToCypherRetriever: converts query to Cypher for precise graph queries
    """
    # Strategy 1: Synonym-based keyword retrieval
    synonym_retriever = LLMSynonymRetriever(
        index.property_graph_store,
        llm=Settings.llm,
        include_text=True,  # include source chunk text
        synonym_prompt=(
            "Given the query '{query_str}', generate a list of synonyms, "
            "related terms, and alternative phrasings that might match "
            "entity names in a knowledge graph. Return one term per line."
        ),
    )

    # Strategy 2: Vector similarity retrieval
    vector_retriever = VectorContextRetriever(
        index.property_graph_store,
        embed_model=Settings.embed_model,
        include_text=True,
        similarity_top_k=10,
        path_depth=2,  # traverse 2 hops from matched entities
    )

    # Strategy 3: Text-to-Cypher retrieval
    cypher_retriever = TextToCypherRetriever(
        index.property_graph_store,
        llm=Settings.llm,
        # Custom Cypher generation prompt
        cypher_prompt=(
            "Generate a Cypher query to answer this question about "
            "a knowledge graph. Use LIMIT 20. "
            "Available node labels: Entity, Chunk, Document, Community. "
            "Available relationship types: RELATED_TO, MENTIONED_IN, PART_OF. "
            "\nQuestion: {query_str}\nCypher:"
        ),
    )

    # Combine retrievers
    query_engine = index.as_query_engine(
        sub_retrievers=[
            synonym_retriever,
            vector_retriever,
        ],
        # Use vector + synonym by default; add cypher_retriever for
        # questions that need precise graph traversal
    )

    return query_engine


def create_cypher_query_engine(index: PropertyGraphIndex):
    """Create a query engine focused on Cypher-based retrieval.

    Best for questions that explicitly involve graph structure:
    - "What is the shortest path between X and Y?"
    - "Which entities have the most connections?"
    - "List all entities of type TECHNOLOGY"
    """
    cypher_retriever = TextToCypherRetriever(
        index.property_graph_store,
        llm=Settings.llm,
    )

    return index.as_query_engine(
        sub_retrievers=[cypher_retriever],
    )
```

---

## Production Patterns

### Connection Pooling and Error Handling

```python
import time
from contextlib import contextmanager
from neo4j import GraphDatabase
from neo4j.exceptions import ServiceUnavailable, SessionExpired, TransientError


class ProductionNeo4jClient:
    """Production-grade Neo4j client with connection pooling and retry logic.

    Neo4j Python driver already pools connections, but we add:
    - Retry logic for transient failures
    - Health checking
    - Query timeout
    - Metrics collection
    """

    def __init__(
        self,
        uri: str,
        user: str,
        password: str,
        database: str = "neo4j",
        max_connection_pool_size: int = 50,
        connection_acquisition_timeout: float = 60.0,
        max_transaction_retry_time: float = 30.0,
    ):
        self.driver = GraphDatabase.driver(
            uri,
            auth=(user, password),
            max_connection_pool_size=max_connection_pool_size,
            connection_acquisition_timeout=connection_acquisition_timeout,
            max_transaction_retry_time=max_transaction_retry_time,
        )
        self.database = database
        self._query_count = 0
        self._error_count = 0

    def health_check(self) -> bool:
        """Verify Neo4j connectivity."""
        try:
            with self.driver.session(database=self.database) as session:
                result = session.run("RETURN 1 AS ok")
                return result.single()["ok"] == 1
        except Exception:
            return False

    def execute_with_retry(
        self,
        query: str,
        parameters: dict | None = None,
        max_retries: int = 3,
        retry_delay: float = 1.0,
    ) -> list[dict]:
        """Execute a Cypher query with retry logic for transient errors."""
        parameters = parameters or {}

        for attempt in range(max_retries):
            try:
                with self.driver.session(database=self.database) as session:
                    result = session.run(query, parameters)
                    records = [dict(record) for record in result]
                    self._query_count += 1
                    return records
            except (ServiceUnavailable, SessionExpired, TransientError) as e:
                self._error_count += 1
                if attempt < max_retries - 1:
                    time.sleep(retry_delay * (2 ** attempt))
                    continue
                raise
            except Exception:
                self._error_count += 1
                raise

        return []

    def get_metrics(self) -> dict:
        """Return client metrics."""
        return {
            "total_queries": self._query_count,
            "total_errors": self._error_count,
            "error_rate": (
                self._error_count / self._query_count
                if self._query_count > 0
                else 0
            ),
        }

    def close(self) -> None:
        self.driver.close()
```

### Query Performance Optimization

```python
class Neo4jQueryOptimizer:
    """Query optimization patterns for Graph RAG workloads."""

    @staticmethod
    def explain_query(driver, query: str, parameters: dict | None = None) -> str:
        """Get the query execution plan (EXPLAIN) for optimization."""
        explain_query = f"EXPLAIN {query}"
        with driver.session() as session:
            result = session.run(explain_query, parameters or {})
            summary = result.consume()
            plan = summary.plan
            return Neo4jQueryOptimizer._format_plan(plan)

    @staticmethod
    def profile_query(driver, query: str, parameters: dict | None = None) -> dict:
        """Profile a query to get actual execution statistics."""
        profile_query = f"PROFILE {query}"
        with driver.session() as session:
            result = session.run(profile_query, parameters or {})
            records = [dict(r) for r in result]
            summary = result.consume()
            profile = summary.profile

            return {
                "records_returned": len(records),
                "db_hits": profile.get("dbHits", 0) if profile else 0,
                "rows": profile.get("rows", 0) if profile else 0,
                "execution_time_ms": summary.result_available_after,
            }

    @staticmethod
    def _format_plan(plan, depth: int = 0) -> str:
        """Recursively format the query plan for display."""
        if plan is None:
            return ""
        indent = "  " * depth
        line = f"{indent}{plan.operator_type}"
        if hasattr(plan, "identifiers"):
            line += f" [{', '.join(plan.identifiers)}]"
        result = line + "\n"
        for child in getattr(plan, "children", []):
            result += Neo4jQueryOptimizer._format_plan(child, depth + 1)
        return result

    @staticmethod
    def optimized_neighborhood_query(entity_name: str, hops: int = 2) -> str:
        """Generate an optimized neighborhood traversal query.

        Key optimizations:
        1. Use index hint for the starting entity
        2. Limit path length explicitly
        3. Use DISTINCT to avoid duplicate neighbor processing
        4. Return only needed properties (avoid fetching embeddings)
        """
        return f"""
        MATCH (start:Entity {{name: $name}})
        USING INDEX start:Entity(name)
        MATCH (start)-[:RELATED_TO*1..{hops}]-(neighbor:Entity)
        WITH DISTINCT neighbor
        RETURN neighbor.name AS name,
               neighbor.type AS type,
               neighbor.description AS description
        ORDER BY neighbor.name
        LIMIT 50
        """

    @staticmethod
    def optimized_community_query(level: int = 2) -> str:
        """Optimized community summary retrieval."""
        return """
        MATCH (cm:Community {level: $level})
        RETURN cm.id AS id,
               cm.title AS title,
               cm.summary AS summary,
               cm.entity_count AS entity_count
        ORDER BY cm.entity_count DESC
        """

    @staticmethod
    def optimized_path_query(
        source_name: str, target_name: str, max_hops: int = 5
    ) -> str:
        """Find shortest path between two entities."""
        return f"""
        MATCH (source:Entity {{name: $source_name}})
        MATCH (target:Entity {{name: $target_name}})
        MATCH path = shortestPath((source)-[:RELATED_TO*..{max_hops}]-(target))
        RETURN [n IN nodes(path) | n.name] AS entities,
               [r IN relationships(path) | r.description] AS descriptions,
               length(path) AS hops
        """
```

### Combining LangChain and LlamaIndex

```python
class UnifiedGraphRAGRetriever:
    """Unified retriever that uses LangChain for Cypher and LlamaIndex for vectors.

    LangChain excels at text-to-Cypher conversion.
    LlamaIndex excels at schema-guided extraction and vector retrieval.
    Combine both for the best of each.
    """

    def __init__(
        self,
        neo4j_uri: str,
        neo4j_user: str,
        neo4j_password: str,
        llama_index: PropertyGraphIndex,
        langchain_chain: GraphCypherQAChain,
    ):
        self.llama_index = llama_index
        self.langchain_chain = langchain_chain
        self.llm = ChatOpenAI(model="gpt-4o", temperature=0)

    def query(self, question: str) -> dict:
        """Route to the best retrieval strategy based on question type."""
        # Classify question type
        question_type = self._classify_question(question)

        if question_type == "structured":
            # Use LangChain Cypher for structured graph queries
            result = self.langchain_chain.invoke({"query": question})
            return {
                "answer": result["result"],
                "strategy": "langchain_cypher",
                "cypher": result.get("intermediate_steps", []),
            }
        elif question_type == "semantic":
            # Use LlamaIndex vector retrieval for semantic queries
            engine = self.llama_index.as_query_engine(
                sub_retrievers=[
                    VectorContextRetriever(
                        self.llama_index.property_graph_store,
                        embed_model=Settings.embed_model,
                        similarity_top_k=10,
                    )
                ]
            )
            response = engine.query(question)
            return {
                "answer": str(response),
                "strategy": "llamaindex_vector",
                "sources": [n.metadata for n in response.source_nodes],
            }
        else:
            # Hybrid: run both and combine
            cypher_result = self.langchain_chain.invoke({"query": question})
            vector_engine = self.llama_index.as_query_engine()
            vector_result = vector_engine.query(question)

            combined = self.llm.invoke(
                f"Combine these two answers to the question: {question}\n\n"
                f"Answer 1 (from graph query): {cypher_result['result']}\n\n"
                f"Answer 2 (from semantic search): {str(vector_result)}\n\n"
                f"Provide a comprehensive combined answer."
            )

            return {
                "answer": combined.content,
                "strategy": "hybrid",
            }

    def _classify_question(self, question: str) -> str:
        """Classify whether the question needs structured or semantic retrieval."""
        response = self.llm.invoke(
            "Classify this question for a knowledge graph:\n"
            f"Question: {question}\n\n"
            "- STRUCTURED: asks about specific paths, counts, or graph structure\n"
            "- SEMANTIC: asks about meaning, similarity, or themes\n"
            "- HYBRID: needs both structural and semantic understanding\n\n"
            "Respond with one word: STRUCTURED, SEMANTIC, or HYBRID"
        )
        result = response.content.strip().upper()
        if result in ("STRUCTURED", "SEMANTIC", "HYBRID"):
            return result.lower()
        return "hybrid"
```

---

## Common Pitfalls

1. **Not refreshing the schema in LangChain.** `GraphCypherQAChain` needs the current Neo4j schema to generate valid Cypher. Call `graph.refresh_schema()` after schema changes, or the LLM will generate queries against a stale schema.

2. **Returning too much data from Cypher.** Always use `LIMIT` in generated Cypher queries. Without it, a broad query like `MATCH (e:Entity) RETURN e` could return millions of nodes and crash the pipeline.

3. **Missing vector indexes.** Neo4j vector search requires explicit index creation. If you skip `CREATE VECTOR INDEX`, similarity queries will fail or fall back to brute-force scan.

4. **Not using UNWIND for bulk operations.** Individual `MERGE` statements for thousands of entities are 10-50x slower than batched `UNWIND` operations. Always batch when loading data.

5. **Ignoring query plans.** Use `EXPLAIN` and `PROFILE` to verify that queries use indexes. A missing index on a frequently queried property can make queries 100x slower.

6. **Mixing LlamaIndex and LangChain incorrectly.** Both frameworks can connect to the same Neo4j database, but they may use different node labels and relationship types. Verify schema compatibility when combining them.

---

## References

- LangChain Neo4j integration: https://python.langchain.com/docs/integrations/graphs/neo4j_cypher
- LlamaIndex PropertyGraphIndex: https://docs.llamaindex.ai/en/stable/module_guides/indexing/property_graph_index/
- Neo4j Cypher manual: https://neo4j.com/docs/cypher-manual/current/
- Neo4j Vector Index: https://neo4j.com/docs/cypher-manual/current/indexes/semantic-indexes/vector-indexes/
- Neo4j Python driver: https://neo4j.com/docs/python-manual/current/
