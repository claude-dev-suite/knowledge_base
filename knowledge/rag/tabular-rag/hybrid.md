# Hybrid Text-SQL RAG -- Text Description Embeddings, SQL Integration, and Semantic Layers

## TL;DR

Pure NL2SQL fails when questions require context that lives outside the database (business definitions, calculation rules, domain knowledge), and pure vector search fails when questions require precise numeric answers from structured data. Hybrid text-SQL RAG combines both: vector search over documentation (including table descriptions, metric definitions, and business rules) provides the LLM with domain context, while NL2SQL generates and executes precise queries. Semantic layers (Cube, dbt Metrics) add a third dimension by pre-defining trusted metric calculations that the LLM can reference instead of inventing SQL aggregations. This article covers hybrid architecture, text description embedding strategies, Cube and dbt semantic layer integration, and evaluation.

---

## Hybrid Architecture

### Three-Source Context Assembly

```python
from dataclasses import dataclass
from typing import Any


@dataclass
class HybridContext:
    """Context assembled from multiple sources for answer generation."""

    sql_result: dict | None = None
    documents: list[dict] | None = None
    metric_definitions: list[dict] | None = None


class HybridTextSQLRAG:
    """RAG pipeline that combines vector search, NL2SQL, and
    semantic layer queries for comprehensive answers.

    For a question like "Why did Q3 churn increase?", the pipeline:
    1. Retrieves the definition of "churn" from documentation
    2. Queries the database for Q3 churn numbers
    3. Retrieves business context about Q3 events
    4. Synthesizes everything into a complete answer
    """

    def __init__(
        self,
        vector_retriever,
        sql_executor,
        semantic_layer=None,
        llm=None,
        router=None,
    ):
        self.retriever = vector_retriever
        self.sql = sql_executor
        self.semantic = semantic_layer
        self.llm = llm
        self.router = router

    def query(self, question: str) -> dict:
        """Execute a hybrid query."""
        context = HybridContext()

        # Step 1: Always retrieve documentation context
        docs = self.retriever.invoke(question)
        context.documents = [
            {"content": d.page_content, "source": d.metadata.get("source", "")}
            for d in docs[:5]
        ]

        # Step 2: Check if SQL is needed
        route = self.router.route(question) if self.router else "hybrid"

        if route in ("sql", "hybrid"):
            # Check semantic layer first for pre-defined metrics
            if self.semantic:
                metrics = self.semantic.find_metrics(question)
                if metrics:
                    context.metric_definitions = metrics
                    # Use semantic layer query if available
                    sl_result = self.semantic.query(question, metrics)
                    if sl_result:
                        context.sql_result = sl_result

            # Fall back to raw SQL if no semantic layer match
            if not context.sql_result:
                sql_result = self.sql.execute_with_schema(question)
                if sql_result.get("success"):
                    context.sql_result = sql_result

        # Step 3: Generate answer from combined context
        answer = self._generate_answer(question, context)

        return {
            "answer": answer,
            "context": {
                "has_sql": context.sql_result is not None,
                "has_docs": context.documents is not None,
                "has_metrics": context.metric_definitions is not None,
            },
        }

    def _generate_answer(
        self, question: str, context: HybridContext
    ) -> str:
        """Generate answer from hybrid context."""
        parts = []

        if context.metric_definitions:
            metrics_text = "\n".join(
                f"- {m['name']}: {m['definition']}"
                for m in context.metric_definitions
            )
            parts.append(f"Metric Definitions:\n{metrics_text}")

        if context.sql_result:
            sql_text = (
                f"Database Query: {context.sql_result.get('sql', 'N/A')}\n"
                f"Result: {context.sql_result.get('rows', [])}"
            )
            parts.append(f"Data:\n{sql_text}")

        if context.documents:
            doc_text = "\n\n".join(
                d["content"] for d in context.documents
            )
            parts.append(f"Documentation:\n{doc_text}")

        combined = "\n\n---\n\n".join(parts)

        response = self.llm.invoke(
            f"Using the following context (data + documentation), "
            f"answer the question comprehensively. Include specific "
            f"numbers from the data and reference definitions when relevant.\n\n"
            f"{combined}\n\n"
            f"Question: {question}"
        )
        return response.content
```

---

## Text Description Embeddings

### Embedding Business Glossary and Metric Definitions

```python
class BusinessKnowledgeIndexer:
    """Index business knowledge for hybrid RAG.

    Sources:
    1. Table/column descriptions
    2. Metric definitions (how KPIs are calculated)
    3. Business glossary (term definitions)
    4. Business rules and constraints
    """

    def __init__(self, vector_store, embedding_fn):
        self.store = vector_store
        self.embed = embedding_fn

    def index_metric_definitions(
        self, metrics: list[dict]
    ) -> int:
        """Index metric definitions.

        metric format:
        {
            "name": "Monthly Recurring Revenue (MRR)",
            "definition": "Sum of all active subscription amounts",
            "sql_formula": "SUM(subscriptions.amount) WHERE status='active'",
            "tables": ["subscriptions"],
            "category": "revenue",
        }
        """
        texts = []
        metadatas = []

        for metric in metrics:
            text = (
                f"Metric: {metric['name']}\n"
                f"Definition: {metric['definition']}\n"
                f"Calculation: {metric['sql_formula']}\n"
                f"Source tables: {', '.join(metric['tables'])}"
            )
            texts.append(text)
            metadatas.append({
                "type": "metric",
                "name": metric["name"],
                "category": metric.get("category", ""),
                "tables": ",".join(metric["tables"]),
            })

        self.store.add_texts(texts, metadatas=metadatas)
        return len(texts)

    def index_business_glossary(
        self, terms: list[dict]
    ) -> int:
        """Index business glossary terms.

        term format:
        {
            "term": "Churn Rate",
            "definition": "Percentage of customers who cancel...",
            "related_metrics": ["MRR Churn", "Logo Churn"],
            "department": "success",
        }
        """
        texts = []
        metadatas = []

        for term in terms:
            text = (
                f"Business Term: {term['term']}\n"
                f"Definition: {term['definition']}"
            )
            if term.get("related_metrics"):
                text += f"\nRelated metrics: {', '.join(term['related_metrics'])}"

            texts.append(text)
            metadatas.append({
                "type": "glossary",
                "term": term["term"],
                "department": term.get("department", ""),
            })

        self.store.add_texts(texts, metadatas=metadatas)
        return len(texts)

    def index_table_relationships(
        self, relationships: list[dict]
    ) -> int:
        """Index table relationships and join paths.

        relationship format:
        {
            "from_table": "orders",
            "to_table": "customers",
            "join_key": "orders.customer_id = customers.id",
            "relationship": "many-to-one",
            "description": "Each order belongs to one customer",
        }
        """
        texts = []
        metadatas = []

        for rel in relationships:
            text = (
                f"Relationship: {rel['from_table']} -> {rel['to_table']}\n"
                f"Join: {rel['join_key']}\n"
                f"Type: {rel['relationship']}\n"
                f"Description: {rel['description']}"
            )
            texts.append(text)
            metadatas.append({
                "type": "relationship",
                "from_table": rel["from_table"],
                "to_table": rel["to_table"],
            })

        self.store.add_texts(texts, metadatas=metadatas)
        return len(texts)
```

---

## Semantic Layer Integration

### Cube Semantic Layer

```python
import httpx


class CubeSemanticLayer:
    """Integration with Cube semantic layer for pre-defined metrics.

    Cube provides a centralized place to define metrics, dimensions,
    and their relationships. The LLM queries Cube instead of
    generating raw SQL, ensuring consistent metric calculations.
    """

    def __init__(
        self,
        cube_url: str = "http://localhost:4000",
        api_token: str = "",
    ):
        self.url = cube_url
        self.token = api_token
        self.client = httpx.Client(
            base_url=cube_url,
            headers={"Authorization": f"Bearer {api_token}"},
        )

    def get_available_metrics(self) -> list[dict]:
        """Get all available metrics from Cube."""
        response = self.client.get("/cubejs-api/v1/meta")
        meta = response.json()

        metrics = []
        for cube in meta.get("cubes", []):
            for measure in cube.get("measures", []):
                metrics.append({
                    "name": measure["name"],
                    "title": measure.get("title", measure["name"]),
                    "type": measure["type"],
                    "cube": cube["name"],
                    "description": measure.get("description", ""),
                })

        return metrics

    def find_metrics(self, question: str) -> list[dict]:
        """Find relevant pre-defined metrics for a question."""
        all_metrics = self.get_available_metrics()

        # Simple keyword matching (in production, use embedding similarity)
        question_lower = question.lower()
        relevant = []
        for metric in all_metrics:
            name_lower = metric["title"].lower()
            desc_lower = metric.get("description", "").lower()
            if any(
                word in question_lower
                for word in name_lower.split()
                if len(word) > 3
            ):
                relevant.append(metric)

        return relevant[:5]

    def query(
        self,
        question: str,
        metrics: list[dict],
        time_dimension: str | None = None,
    ) -> dict | None:
        """Query Cube for metric data."""
        if not metrics:
            return None

        measures = [m["name"] for m in metrics]

        cube_query = {"measures": measures}

        if time_dimension:
            cube_query["timeDimensions"] = [{
                "dimension": time_dimension,
                "granularity": "month",
            }]

        try:
            response = self.client.post(
                "/cubejs-api/v1/load",
                json={"query": cube_query},
            )
            data = response.json()

            return {
                "success": True,
                "sql": data.get("sql", {}).get("sql", [""])[0],
                "rows": data.get("data", []),
                "source": "cube_semantic_layer",
            }
        except Exception as e:
            return {"success": False, "error": str(e)}
```

### dbt Metrics Layer

```python
class DbtMetricsLayer:
    """Integration with dbt Metrics for consistent metric definitions.

    dbt Metrics defines metrics as code (YAML), ensuring
    every query uses the same calculation logic.
    """

    def __init__(
        self,
        metrics_yaml_path: str,
        db_connection,
    ):
        self.db = db_connection
        self.metrics = self._load_metrics(metrics_yaml_path)

    def _load_metrics(self, path: str) -> dict:
        """Load metric definitions from dbt metrics YAML."""
        import yaml

        with open(path) as f:
            config = yaml.safe_load(f)

        metrics = {}
        for metric in config.get("metrics", []):
            metrics[metric["name"]] = {
                "name": metric["name"],
                "label": metric.get("label", metric["name"]),
                "description": metric.get("description", ""),
                "type": metric.get("type", ""),
                "sql": metric.get("sql", ""),
                "timestamp": metric.get("timestamp", ""),
                "time_grains": metric.get("time_grains", ["day", "month"]),
                "dimensions": metric.get("dimensions", []),
                "filters": metric.get("filters", []),
            }

        return metrics

    def find_metric(self, question: str) -> dict | None:
        """Find the best matching metric for a question."""
        question_lower = question.lower()
        best_match = None
        best_score = 0

        for name, metric in self.metrics.items():
            # Score based on keyword overlap
            label_words = set(metric["label"].lower().split())
            desc_words = set(metric["description"].lower().split())
            all_words = label_words | desc_words

            question_words = set(question_lower.split())
            overlap = len(all_words & question_words)

            if overlap > best_score:
                best_score = overlap
                best_match = metric

        return best_match if best_score >= 2 else None

    def query_metric(
        self,
        metric_name: str,
        time_grain: str = "month",
        filters: dict | None = None,
    ) -> dict:
        """Query a pre-defined metric with optional filters."""
        metric = self.metrics.get(metric_name)
        if not metric:
            return {"error": f"Metric '{metric_name}' not found"}

        # Build SQL from metric definition
        sql = f"""
SELECT DATE_TRUNC('{time_grain}', {metric['timestamp']}) as period,
       {metric['sql']} as {metric_name}
FROM {metric.get('model', 'base_table')}
"""
        if filters:
            conditions = [
                f"{k} = '{v}'" for k, v in filters.items()
            ]
            sql += " WHERE " + " AND ".join(conditions)

        sql += f" GROUP BY DATE_TRUNC('{time_grain}', {metric['timestamp']})"
        sql += " ORDER BY period"

        try:
            result = self.db.execute(sql)
            rows = result.fetchall()
            return {
                "success": True,
                "metric": metric_name,
                "definition": metric["description"],
                "sql": sql,
                "rows": [list(r) for r in rows],
            }
        except Exception as e:
            return {"success": False, "error": str(e)}
```

---

## Evaluation

### Measuring Hybrid RAG Quality

```python
class TabularRAGEvaluator:
    """Evaluate tabular RAG accuracy on known question-answer pairs."""

    def __init__(self, pipeline):
        self.pipeline = pipeline

    def evaluate(
        self, test_cases: list[dict]
    ) -> dict:
        """Run evaluation on test cases.

        test_case format:
        {
            "question": "Total revenue in Q3 2024?",
            "expected_answer": "$12,345,678",
            "expected_route": "sql",
            "expected_tables": ["orders"],
        }
        """
        results = []

        for tc in test_cases:
            result = self.pipeline.query(tc["question"])

            # Check route correctness
            route_correct = (
                result.get("context", {}).get("has_sql", False)
                == (tc.get("expected_route") in ("sql", "hybrid"))
            )

            # Check answer contains expected value
            expected = tc.get("expected_answer", "")
            answer_correct = expected.lower() in result["answer"].lower()

            results.append({
                "question": tc["question"],
                "route_correct": route_correct,
                "answer_correct": answer_correct,
                "answer": result["answer"][:200],
            })

        total = len(results)
        return {
            "total": total,
            "route_accuracy": sum(r["route_correct"] for r in results) / total,
            "answer_accuracy": sum(r["answer_correct"] for r in results) / total,
            "details": results,
        }
```

---

## Common Pitfalls

1. **Not embedding metric definitions.** If the LLM does not know that "MRR" means "Monthly Recurring Revenue = SUM of active subscription amounts," it will generate incorrect SQL. Index all metric definitions and business glossary terms.
2. **Bypassing the semantic layer for ad-hoc SQL.** If you have a semantic layer (Cube, dbt), always check it first. It ensures consistent calculations. Raw NL2SQL may compute metrics differently than the official definitions.
3. **Not including join path information.** LLMs struggle with multi-table joins. Embedding table relationship descriptions (join keys, relationship types) significantly improves join accuracy.
4. **Treating all questions as SQL-answerable.** Questions about processes, definitions, or reasons require documentation, not SQL. A router that correctly identifies question type is essential for hybrid RAG.
5. **Not evaluating on real business questions.** Synthetic test questions like "count all orders" are easy. Real questions like "why did Q3 churn spike for enterprise accounts" require hybrid context. Test with real questions from business users.
6. **Ignoring data freshness.** SQL queries return live data, but vector-searched documentation may be outdated. If a metric definition changed last week, the documentation and semantic layer must be in sync.

---

## References

- Cube semantic layer: https://cube.dev/docs
- dbt Metrics: https://docs.getdbt.com/docs/build/metrics
- LangChain SQL tutorial: https://python.langchain.com/docs/tutorials/sql_qa/
- BIRD-SQL benchmark: Li et al. "Can LLM Already Serve as A Database Interface?" (2023)
