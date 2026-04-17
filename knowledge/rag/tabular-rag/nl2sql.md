# NL2SQL -- LangChain SQLDatabaseChain, Safe Execution, and Schema-Aware Retrieval

## TL;DR

NL2SQL (Natural Language to SQL) translates user questions into database queries, enabling RAG systems to answer questions that require precise numeric data, aggregations, and joins. LangChain provides `SQLDatabaseChain` and `create_sql_query_chain` for end-to-end NL2SQL with schema introspection, query generation, execution, and answer synthesis. This article covers LangChain SQL chain setup, schema introspection strategies (full schema vs relevant tables), safe execution patterns (read-only connections, query validation, row limits), error recovery (retry with error feedback), and few-shot prompting for complex queries.

---

## LangChain SQL Chain Setup

### Basic Setup with create_sql_query_chain

```python
from langchain_anthropic import ChatAnthropic
from langchain_community.utilities import SQLDatabase
from langchain.chains import create_sql_query_chain
from langchain_community.tools.sql_database.tool import QuerySQLDataBaseTool

# Connect to database (read-only recommended)
db = SQLDatabase.from_uri(
    "postgresql://readonly_user:password@localhost:5432/analytics",
    include_tables=[
        "orders", "customers", "products", "order_items",
    ],
    sample_rows_in_table_info=3,  # Include sample rows in schema
)

# Create the LLM
llm = ChatAnthropic(model="claude-sonnet-4-20250514", temperature=0)

# Create the SQL generation chain
sql_chain = create_sql_query_chain(llm, db)

# Generate SQL (does NOT execute)
generated_sql = sql_chain.invoke({
    "question": "What were total sales last month?"
})
print(f"Generated SQL: {generated_sql}")
```

### Full Query-Execute-Answer Chain

```python
from langchain_core.output_parsers import StrOutputParser
from langchain_core.prompts import ChatPromptTemplate
from langchain_core.runnables import RunnablePassthrough
from operator import itemgetter


def create_full_sql_rag_chain(
    db: SQLDatabase,
    llm,
):
    """Create a complete NL2SQL chain: question -> SQL -> execute -> answer."""

    # Step 1: Generate SQL
    sql_chain = create_sql_query_chain(llm, db)

    # Step 2: Execute SQL
    execute_tool = QuerySQLDataBaseTool(db=db)

    # Step 3: Generate natural language answer
    answer_prompt = ChatPromptTemplate.from_messages([
        (
            "system",
            "Given a user question, the SQL query that was executed, "
            "and the SQL result, write a natural language answer. "
            "Include specific numbers from the result. "
            "If the result is empty, say so explicitly.",
        ),
        (
            "human",
            "Question: {question}\n"
            "SQL Query: {query}\n"
            "SQL Result: {result}\n"
            "Answer:",
        ),
    ])

    # Compose the chain
    chain = (
        RunnablePassthrough.assign(query=sql_chain)
        .assign(result=itemgetter("query") | execute_tool)
        | answer_prompt
        | llm
        | StrOutputParser()
    )

    return chain


# Usage
chain = create_full_sql_rag_chain(db, llm)
answer = chain.invoke({"question": "Top 5 customers by total order value"})
print(answer)
```

---

## Safe Execution Patterns

### Read-Only Database Connection

```python
from sqlalchemy import create_engine, text
from sqlalchemy.pool import StaticPool


def create_readonly_engine(
    connection_string: str,
    statement_timeout_ms: int = 5000,
    max_rows: int = 1000,
):
    """Create a read-only database connection with safety limits."""
    engine = create_engine(
        connection_string,
        connect_args={
            "options": f"-c statement_timeout={statement_timeout_ms}",
        },
        poolclass=StaticPool,
    )

    # Set session to read-only
    with engine.connect() as conn:
        conn.execute(text("SET default_transaction_read_only = ON"))
        conn.commit()

    return engine


class SafeSQLExecutor:
    """Execute SQL with multiple safety layers."""

    BLOCKED_KEYWORDS = {
        "INSERT", "UPDATE", "DELETE", "DROP", "ALTER",
        "TRUNCATE", "CREATE", "GRANT", "REVOKE",
        "EXEC", "EXECUTE", "CALL", "MERGE",
    }

    def __init__(
        self,
        engine,
        max_rows: int = 1000,
        timeout_ms: int = 5000,
    ):
        self.engine = engine
        self.max_rows = max_rows
        self.timeout_ms = timeout_ms

    def validate(self, sql: str) -> dict:
        """Validate SQL before execution."""
        sql_upper = sql.upper().strip()
        issues = []

        # Check for blocked keywords
        for keyword in self.BLOCKED_KEYWORDS:
            # Check as a word boundary to avoid false positives
            # (e.g., "UPDATED_AT" should not match "UPDATE")
            import re
            if re.search(rf"\b{keyword}\b", sql_upper):
                issues.append(f"Blocked keyword: {keyword}")

        # Must start with SELECT or WITH (for CTEs)
        if not (sql_upper.startswith("SELECT") or sql_upper.startswith("WITH")):
            issues.append("Query must start with SELECT or WITH")

        # No multiple statements
        # Remove semicolons from string literals before checking
        clean = re.sub(r"'[^']*'", "", sql)
        if clean.count(";") > 1:
            issues.append("Multiple statements not allowed")

        return {"valid": len(issues) == 0, "issues": issues}

    def execute(self, sql: str) -> dict:
        """Validate and execute SQL safely."""
        validation = self.validate(sql)
        if not validation["valid"]:
            return {
                "success": False,
                "error": f"Validation failed: {validation['issues']}",
                "sql": sql,
            }

        # Inject LIMIT if not present
        sql_with_limit = self._ensure_limit(sql)

        try:
            with self.engine.connect() as conn:
                # Set per-query timeout
                conn.execute(
                    text(f"SET statement_timeout = {self.timeout_ms}")
                )
                result = conn.execute(text(sql_with_limit))
                rows = result.fetchall()
                columns = list(result.keys())

                return {
                    "success": True,
                    "sql": sql_with_limit,
                    "columns": columns,
                    "rows": [list(row) for row in rows],
                    "row_count": len(rows),
                    "truncated": len(rows) >= self.max_rows,
                }

        except Exception as e:
            return {
                "success": False,
                "error": str(e),
                "sql": sql_with_limit,
            }

    def _ensure_limit(self, sql: str) -> str:
        """Add LIMIT if not present."""
        if "LIMIT" not in sql.upper():
            sql = sql.rstrip(";")
            return f"{sql} LIMIT {self.max_rows}"
        return sql
```

---

## Schema-Aware Retrieval

### Dynamic Schema Selection

```python
class DynamicSchemaSelector:
    """Select relevant tables based on the question.

    Instead of passing the full database schema to the LLM
    (which may be thousands of lines for large databases),
    select only the relevant tables based on the question.
    """

    def __init__(self, db: SQLDatabase, vector_store, embedding_fn):
        self.db = db
        self.store = vector_store
        self.embed = embedding_fn
        self._index_schema()

    def _index_schema(self) -> None:
        """Index table descriptions for retrieval."""
        table_info = self.db.get_table_info()
        # Parse table info into individual table descriptions
        tables = self._parse_table_info(table_info)

        texts = []
        metadatas = []
        for table_name, info in tables.items():
            texts.append(info)
            metadatas.append({"table_name": table_name})

        self.store.add_texts(texts, metadatas=metadatas)

    def select_tables(
        self, question: str, max_tables: int = 5
    ) -> list[str]:
        """Select the most relevant tables for a question."""
        results = self.store.similarity_search(
            question, k=max_tables
        )
        return [r.metadata["table_name"] for r in results]

    def get_focused_schema(self, question: str) -> str:
        """Get schema info for only the relevant tables."""
        relevant_tables = self.select_tables(question)
        return self.db.get_table_info_no_throw(
            table_names=relevant_tables
        )

    def _parse_table_info(self, info: str) -> dict[str, str]:
        """Parse the full table info string into per-table entries."""
        tables = {}
        current_table = ""
        current_info = []

        for line in info.split("\n"):
            if line.startswith("CREATE TABLE"):
                if current_table:
                    tables[current_table] = "\n".join(current_info)
                # Extract table name
                parts = line.split()
                current_table = parts[2].strip("(").strip('"')
                current_info = [line]
            else:
                current_info.append(line)

        if current_table:
            tables[current_table] = "\n".join(current_info)

        return tables
```

---

## Few-Shot Prompting for Complex Queries

```python
FEW_SHOT_EXAMPLES = [
    {
        "question": "What were total sales by product category last quarter?",
        "sql": """SELECT p.category, SUM(oi.quantity * oi.unit_price) as total_sales
FROM order_items oi
JOIN products p ON oi.product_id = p.id
JOIN orders o ON oi.order_id = o.id
WHERE o.created_at >= DATE_TRUNC('quarter', CURRENT_DATE - INTERVAL '3 months')
  AND o.created_at < DATE_TRUNC('quarter', CURRENT_DATE)
GROUP BY p.category
ORDER BY total_sales DESC""",
    },
    {
        "question": "Which customers have not placed an order in the last 90 days?",
        "sql": """SELECT c.id, c.name, c.email, MAX(o.created_at) as last_order
FROM customers c
LEFT JOIN orders o ON c.id = o.customer_id
GROUP BY c.id, c.name, c.email
HAVING MAX(o.created_at) < CURRENT_DATE - INTERVAL '90 days'
   OR MAX(o.created_at) IS NULL
ORDER BY last_order ASC NULLS FIRST
LIMIT 100""",
    },
    {
        "question": "What is the month-over-month growth rate?",
        "sql": """WITH monthly AS (
    SELECT DATE_TRUNC('month', created_at) as month,
           SUM(total_amount) as revenue
    FROM orders
    WHERE created_at >= CURRENT_DATE - INTERVAL '12 months'
    GROUP BY DATE_TRUNC('month', created_at)
)
SELECT month,
       revenue,
       LAG(revenue) OVER (ORDER BY month) as prev_revenue,
       ROUND(
           (revenue - LAG(revenue) OVER (ORDER BY month))
           / LAG(revenue) OVER (ORDER BY month) * 100, 2
       ) as growth_pct
FROM monthly
ORDER BY month""",
    },
]


def build_few_shot_prompt(
    question: str,
    schema: str,
    examples: list[dict] = FEW_SHOT_EXAMPLES,
) -> str:
    """Build a prompt with few-shot examples for better SQL generation."""
    examples_text = "\n\n".join(
        f"Question: {ex['question']}\nSQL:\n{ex['sql']}"
        for ex in examples
    )

    return f"""Generate a SQL query for the given question.

DATABASE SCHEMA:
{schema}

EXAMPLES:
{examples_text}

Question: {question}
SQL:"""
```

---

## Error Recovery

```python
class NL2SQLWithRetry:
    """NL2SQL with error feedback retry."""

    def __init__(self, sql_generator, executor: SafeSQLExecutor, llm, max_retries: int = 2):
        self.generator = sql_generator
        self.executor = executor
        self.llm = llm
        self.max_retries = max_retries

    def query(self, question: str, schema: str) -> dict:
        """Generate and execute SQL with retry on failure."""
        errors = []

        for attempt in range(self.max_retries + 1):
            # Generate SQL (include previous errors for learning)
            if errors:
                error_context = "\n".join(
                    f"Attempt {i+1} failed:\nSQL: {e['sql']}\nError: {e['error']}"
                    for i, e in enumerate(errors)
                )
                prompt = (
                    f"Previous attempts failed. Fix the SQL.\n\n"
                    f"{error_context}\n\n"
                    f"Schema:\n{schema}\n\n"
                    f"Question: {question}"
                )
            else:
                prompt = build_few_shot_prompt(question, schema)

            response = self.llm.invoke(prompt)
            sql = response.content.strip().strip("```sql").strip("```").strip()

            # Execute
            result = self.executor.execute(sql)

            if result["success"]:
                return {
                    "success": True,
                    "sql": sql,
                    "result": result,
                    "attempts": attempt + 1,
                }

            errors.append({"sql": sql, "error": result["error"]})

        return {
            "success": False,
            "errors": errors,
            "attempts": self.max_retries + 1,
        }
```

---

## Common Pitfalls

1. **Passing the entire database schema to the LLM.** Large schemas (100+ tables) overwhelm the context window and degrade SQL quality. Use schema-aware retrieval to select only relevant tables.
2. **Not validating generated SQL before execution.** The LLM may generate UPDATE, DELETE, or other destructive queries. Always validate before execution and use a read-only database connection.
3. **No query timeout.** A poorly generated query (missing WHERE clause on a large table) can run for minutes. Set statement-level timeouts (5-10 seconds) on all generated queries.
4. **Not including sample rows in the schema.** LangChain's `sample_rows_in_table_info` parameter includes example data, helping the LLM understand data formats and typical values.
5. **Not providing few-shot examples for complex queries.** CTEs, window functions, and multi-table joins are hard for LLMs to generate correctly without examples. Provide 3-5 few-shot examples covering your common query patterns.
6. **Ignoring SQL injection in the generated query.** While the LLM generates the SQL (not the user directly), a crafted user question like "Show orders; DROP TABLE orders" could influence the generated SQL. Always validate.

---

## References

- LangChain SQL tutorial: https://python.langchain.com/docs/tutorials/sql_qa/
- LangChain SQLDatabaseChain: https://python.langchain.com/docs/integrations/tools/sql_database/
- DIN-SQL: Pourreza & Rafiei "DIN-SQL: Decomposed In-Context Learning of Text-to-SQL" (2023)
