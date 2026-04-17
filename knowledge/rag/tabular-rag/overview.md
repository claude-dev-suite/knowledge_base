# Tabular RAG -- NL2SQL and Vector Hybrid for Structured Data

## TL;DR

Most RAG systems only handle unstructured text, but enterprises store critical data in SQL databases, spreadsheets, and data warehouses. Tabular RAG bridges this gap by combining two retrieval strategies: NL2SQL (translating natural language questions into SQL queries for precise numeric and aggregation answers) and vector search over text descriptions of tables and columns (for semantic understanding of what data is available). When a user asks "What were our top 5 customers by revenue last quarter?", NL2SQL generates the SQL; when they ask "How do we calculate churn rate?", vector search finds documentation about the metric definition. This overview covers the architecture, when to use each approach, and hybrid strategies. Subsequent articles cover LangChain SQLDatabaseChain implementation and hybrid text-SQL patterns with semantic layers.

---

## The Structured Data Problem

### Why Vector Search Alone Fails on Structured Data

```
User: "What were total sales in Q3 2024?"

Vector search approach:
  -> Embeds query
  -> Retrieves document: "Our Q2 sales exceeded $10M..."
  -> Wrong quarter, approximate number
  -> Answer: "Sales were approximately $10M" (WRONG)

NL2SQL approach:
  -> Generates: SELECT SUM(amount) FROM sales WHERE quarter = 'Q3' AND year = 2024
  -> Executes against database
  -> Answer: "Total sales in Q3 2024 were $12,345,678.90" (EXACT)
```

### Query Types and Best Approach

| Query Type | Example | Best Approach |
|------------|---------|--------------|
| Aggregation | "Total revenue last month" | NL2SQL |
| Lookup | "What is customer X's email?" | NL2SQL |
| Ranking | "Top 10 products by sales" | NL2SQL |
| Filtering | "Orders over $1000 from France" | NL2SQL |
| Definition | "How is churn rate calculated?" | Vector search |
| Process | "How do I create a refund?" | Vector search |
| Hybrid | "Why did Q3 revenue drop compared to Q2?" | Both |

---

## Architecture

### Hybrid Router

```python
from anthropic import Anthropic
from enum import Enum


class QueryRoute(Enum):
    SQL = "sql"
    VECTOR = "vector"
    HYBRID = "hybrid"


class TabularRAGRouter:
    """Route queries to SQL, vector search, or both.

    Uses an LLM to classify the query type and determine
    which retrieval strategy will produce the best answer.
    """

    ROUTING_PROMPT = """Classify this question into one of three categories:

SQL: The question asks for specific data, numbers, aggregations, rankings,
     or lookups that require querying a database.
     Examples: "total revenue", "top 10 customers", "how many orders"

VECTOR: The question asks about concepts, definitions, processes, or
        documentation that is stored as text.
        Examples: "how does churn work", "explain the refund process"

HYBRID: The question requires both data from the database AND context
        from documentation to answer fully.
        Examples: "why did revenue drop", "compare Q3 to our targets"

Question: {question}

Respond with ONLY one word: SQL, VECTOR, or HYBRID"""

    def __init__(self, model: str = "claude-haiku-4-20250514"):
        self.client = Anthropic()
        self.model = model

    def route(self, question: str) -> QueryRoute:
        """Determine the best retrieval strategy."""
        response = self.client.messages.create(
            model=self.model,
            max_tokens=10,
            messages=[{
                "role": "user",
                "content": self.ROUTING_PROMPT.format(question=question),
            }],
        )

        route_text = response.content[0].text.strip().upper()

        if "SQL" in route_text and "HYBRID" not in route_text:
            return QueryRoute.SQL
        elif "VECTOR" in route_text:
            return QueryRoute.VECTOR
        else:
            return QueryRoute.HYBRID


class TabularRAGPipeline:
    """Complete tabular RAG pipeline with routing."""

    def __init__(
        self,
        sql_chain,
        vector_retriever,
        llm,
        router: TabularRAGRouter,
    ):
        self.sql_chain = sql_chain
        self.retriever = vector_retriever
        self.llm = llm
        self.router = router

    def query(self, question: str) -> dict:
        """Route and execute the query."""
        route = self.router.route(question)

        if route == QueryRoute.SQL:
            return self._sql_query(question)
        elif route == QueryRoute.VECTOR:
            return self._vector_query(question)
        else:
            return self._hybrid_query(question)

    def _sql_query(self, question: str) -> dict:
        """Execute via NL2SQL."""
        result = self.sql_chain.invoke({"query": question})
        return {
            "answer": result.get("result", ""),
            "sql": result.get("intermediate_steps", []),
            "route": "sql",
        }

    def _vector_query(self, question: str) -> dict:
        """Execute via vector search."""
        docs = self.retriever.invoke(question)
        context = "\n\n".join(d.page_content for d in docs[:5])
        answer = self.llm.invoke(
            f"Context:\n{context}\n\nQuestion: {question}"
        ).content
        return {
            "answer": answer,
            "sources": [d.metadata for d in docs[:5]],
            "route": "vector",
        }

    def _hybrid_query(self, question: str) -> dict:
        """Execute both SQL and vector, combine results."""
        sql_result = self._sql_query(question)
        vector_result = self._vector_query(question)

        combined_context = (
            f"Database query result:\n{sql_result['answer']}\n\n"
            f"Documentation context:\n{vector_result['answer']}"
        )

        answer = self.llm.invoke(
            f"Using both the database result and documentation context, "
            f"answer the question comprehensively.\n\n"
            f"{combined_context}\n\n"
            f"Question: {question}"
        ).content

        return {
            "answer": answer,
            "sql_result": sql_result["answer"],
            "doc_context": vector_result["answer"],
            "route": "hybrid",
        }
```

---

## NL2SQL Basics

### Safe SQL Generation

```python
from anthropic import Anthropic


class SafeNL2SQL:
    """Generate SQL from natural language with safety constraints.

    Critical: generated SQL must be read-only (SELECT only),
    validated before execution, and run with limited permissions.
    """

    SYSTEM_PROMPT = """You are a SQL query generator. Generate a SELECT query
to answer the user's question based on the database schema below.

RULES:
1. Generate ONLY SELECT queries. Never generate INSERT, UPDATE, DELETE, DROP, or ALTER.
2. Use only tables and columns from the schema below.
3. Use appropriate aggregations (SUM, COUNT, AVG) for numeric questions.
4. Include WHERE clauses for filtering.
5. Use ORDER BY and LIMIT for ranking questions.
6. Output ONLY the SQL query, nothing else.

DATABASE SCHEMA:
{schema}"""

    def __init__(
        self,
        db_connection,
        schema: str,
        model: str = "claude-sonnet-4-20250514",
    ):
        self.client = Anthropic()
        self.db = db_connection
        self.schema = schema
        self.model = model

    def generate_sql(self, question: str) -> str:
        """Generate SQL from natural language question."""
        response = self.client.messages.create(
            model=self.model,
            max_tokens=500,
            system=self.SYSTEM_PROMPT.format(schema=self.schema),
            messages=[{
                "role": "user",
                "content": question,
            }],
        )
        sql = response.content[0].text.strip()
        sql = sql.strip("```sql").strip("```").strip()
        return sql

    def validate_sql(self, sql: str) -> dict:
        """Validate generated SQL for safety."""
        sql_upper = sql.upper().strip()
        dangerous_keywords = [
            "INSERT", "UPDATE", "DELETE", "DROP", "ALTER",
            "TRUNCATE", "CREATE", "GRANT", "REVOKE", "EXEC",
        ]

        issues = []
        for keyword in dangerous_keywords:
            if keyword in sql_upper:
                issues.append(
                    f"Dangerous keyword detected: {keyword}"
                )

        if not sql_upper.startswith("SELECT"):
            issues.append("Query does not start with SELECT")

        # Check for multiple statements
        if ";" in sql[:-1]:  # Allow trailing semicolon
            issues.append("Multiple statements detected")

        return {
            "valid": len(issues) == 0,
            "issues": issues,
        }

    def execute(self, question: str) -> dict:
        """Generate, validate, and execute SQL."""
        sql = self.generate_sql(question)
        validation = self.validate_sql(sql)

        if not validation["valid"]:
            return {
                "error": f"SQL validation failed: {validation['issues']}",
                "sql": sql,
            }

        try:
            results = self.db.execute(sql)
            rows = results.fetchall()
            columns = list(results.keys()) if hasattr(results, 'keys') else []

            return {
                "sql": sql,
                "columns": columns,
                "rows": [list(row) for row in rows[:100]],
                "row_count": len(rows),
            }
        except Exception as e:
            return {
                "error": f"SQL execution failed: {str(e)}",
                "sql": sql,
            }
```

---

## Schema-Aware Retrieval

### Embedding Table and Column Descriptions

```python
class SchemaEmbedder:
    """Embed database schema descriptions for semantic retrieval.

    When the user asks about data concepts, we need to find
    which tables and columns are relevant. This embeds natural
    language descriptions of each table and column.
    """

    def __init__(self, vector_store, embedding_fn):
        self.store = vector_store
        self.embed = embedding_fn

    def index_schema(self, schema_info: list[dict]) -> int:
        """Index table and column descriptions for retrieval.

        schema_info format:
        [
            {
                "table": "orders",
                "description": "Customer purchase orders",
                "columns": [
                    {"name": "id", "type": "int", "description": "Order ID"},
                    {"name": "amount", "type": "decimal", "description": "Order total"},
                    ...
                ]
            },
            ...
        ]
        """
        texts = []
        metadatas = []

        for table in schema_info:
            # Index table-level description
            table_text = (
                f"Table: {table['table']}\n"
                f"Description: {table['description']}\n"
                f"Columns: {', '.join(c['name'] for c in table['columns'])}"
            )
            texts.append(table_text)
            metadatas.append({
                "type": "table",
                "table_name": table["table"],
            })

            # Index each column
            for col in table["columns"]:
                col_text = (
                    f"Column {col['name']} in table {table['table']}: "
                    f"{col['description']} (type: {col['type']})"
                )
                texts.append(col_text)
                metadatas.append({
                    "type": "column",
                    "table_name": table["table"],
                    "column_name": col["name"],
                })

        self.store.add_texts(texts, metadatas=metadatas)
        return len(texts)

    def find_relevant_tables(
        self, question: str, top_k: int = 5
    ) -> list[dict]:
        """Find tables relevant to a question."""
        results = self.store.similarity_search(
            question,
            k=top_k,
            filter={"type": {"$eq": "table"}},
        )
        return [
            {
                "table": r.metadata["table_name"],
                "description": r.page_content,
                "relevance_score": r.metadata.get("score", 0),
            }
            for r in results
        ]
```

---

## Common Pitfalls

1. **Executing generated SQL without validation.** LLMs can generate DELETE, DROP, or other destructive SQL. Always validate that generated SQL is read-only before execution.
2. **Embedding raw SQL schema instead of descriptions.** `CREATE TABLE orders (id INT, amt DECIMAL)` is not semantically rich. Embed natural language descriptions like "orders table contains customer purchase records with order amounts."
3. **Not handling SQL generation failures.** The LLM may generate syntactically invalid SQL. Catch execution errors and either retry with the error message or fall back to vector search.
4. **Using NL2SQL for questions about processes.** "How do I create a refund?" should not generate SQL. Use a router to direct definitional and process questions to vector search.
5. **Not limiting SQL result sets.** Without LIMIT, a generated query might return millions of rows. Always inject a safety LIMIT (e.g., 1000 rows) on generated queries.
6. **Ignoring data access permissions.** NL2SQL must respect the same access controls as direct database access. Use a read-only database user with permissions limited to the tables the user should access.

---

## References

- LangChain SQL: https://python.langchain.com/docs/tutorials/sql_qa/
- Text-to-SQL survey: Katsogiannis-Meimarakis & Koutrika "A survey on deep learning approaches for text-to-SQL" (2023)
- Cube semantic layer: https://cube.dev/docs
