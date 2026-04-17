# Self-Querying Retriever -- LLM-Inferred Metadata Filters

## TL;DR

A self-querying retriever uses an LLM to parse the user's natural language query into two components: a semantic search query (for vector similarity) and structured metadata filters (for exact matching on document attributes). When a user asks "What Python tutorials were published after 2023?", a standard retriever embeds the entire string and hopes the vector search handles the date constraint. A self-querying retriever extracts the semantic part ("Python tutorials") for embedding search and the structured part (published_date > 2023) as a metadata filter, combining both for precise results. This overview covers the architecture, when self-querying outperforms standard retrieval, the LLM parsing step, and integration patterns. Subsequent articles cover LangChain's SelfQueryRetriever and advanced patterns (hybrid search combination, evaluation, fallback strategies).

---

## The Problem with Pure Vector Search

### Queries with Implicit Filters

```
Query: "What are the pricing plans for enterprise customers?"

Pure vector search:
  -> Embeds the full query
  -> Returns: pricing docs for ALL customer types
  -> User sees free-tier pricing mixed with enterprise pricing

Self-querying:
  -> Semantic query: "pricing plans"
  -> Filter: customer_tier == "enterprise"
  -> Returns: ONLY enterprise pricing docs
```

### Query Types That Benefit

| Query Pattern | Semantic Part | Filter Part |
|---------------|--------------|-------------|
| "Python tutorials after 2023" | "Python tutorials" | year > 2023 |
| "API docs for version 3" | "API docs" | version == "3" |
| "Bug reports from the auth team" | "bug reports" | team == "auth" |
| "Internal policies about remote work" | "policies about remote work" | visibility == "internal" |
| "High-priority incidents last month" | "incidents" | priority == "high" AND date >= last_month |

---

## Architecture

### How Self-Querying Works

```
User: "Show me Python tutorials published in 2024 for beginners"
    |
    v
[LLM Parser]
    |
    +--> Semantic query: "Python tutorials for beginners"
    +--> Filters: {
           "language": {"$eq": "python"},
           "year": {"$eq": 2024},
           "difficulty": {"$eq": "beginner"}
         }
    |
    v
[Vector Store]
    |
    +--> Vector search on "Python tutorials for beginners"
    +--> Metadata filter on language/year/difficulty
    +--> Intersection of both result sets
    |
    v
[Filtered, relevant documents]
```

### The LLM Parser

```python
from anthropic import Anthropic
from pydantic import BaseModel, Field
import json


class StructuredQuery(BaseModel):
    """Output of the self-querying LLM parser."""

    semantic_query: str = Field(
        description="The semantic search query (what to find)"
    )
    filters: dict = Field(
        default_factory=dict,
        description="Metadata filters to apply",
    )
    limit: int | None = Field(
        default=None,
        description="Maximum number of results requested",
    )


class SelfQueryParser:
    """Parse natural language queries into semantic + filter components.

    Uses an LLM to understand the user's intent and extract
    structured filters from the query.
    """

    def __init__(
        self,
        attribute_descriptions: dict[str, str],
        model: str = "claude-haiku-4-20250514",
    ):
        self.client = Anthropic()
        self.model = model
        self.attribute_descriptions = attribute_descriptions

    def _build_system_prompt(self) -> str:
        attrs = "\n".join(
            f"  - {name}: {desc}"
            for name, desc in self.attribute_descriptions.items()
        )
        return f"""You are a query parser for a document search system.

Given a user's natural language query, extract:
1. A semantic search query (the conceptual meaning to search for)
2. Structured metadata filters (exact constraints on document attributes)

Available document attributes for filtering:
{attrs}

Filter operators:
- $eq: equals (exact match)
- $ne: not equals
- $gt: greater than
- $gte: greater than or equal
- $lt: less than
- $lte: less than or equal
- $in: in a list of values
- $contains: contains substring

Rules:
- Only create filters for attributes that are explicitly or strongly implied in the query
- The semantic query should capture the conceptual meaning WITHOUT the filter constraints
- If no filters are applicable, return an empty filters dict
- Do not invent filter values that are not in the query"""

    def parse(self, query: str) -> StructuredQuery:
        """Parse a natural language query into structured components."""
        response = self.client.messages.create(
            model=self.model,
            max_tokens=500,
            system=self._build_system_prompt(),
            messages=[{
                "role": "user",
                "content": (
                    f"Parse this query into semantic search + filters:\n\n"
                    f"Query: \"{query}\"\n\n"
                    f"Respond with a JSON object matching this schema:\n"
                    f'{{"semantic_query": "...", "filters": {{...}}, "limit": null}}'
                ),
            }],
        )

        text = response.content[0].text.strip()

        # Extract JSON from response
        try:
            # Find JSON in the response
            start = text.find("{")
            end = text.rfind("}") + 1
            parsed = json.loads(text[start:end])
            return StructuredQuery(**parsed)
        except (json.JSONDecodeError, ValueError):
            # Fallback: treat entire query as semantic
            return StructuredQuery(semantic_query=query)


# Example usage
parser = SelfQueryParser(
    attribute_descriptions={
        "language": "Programming language (python, javascript, rust, etc.)",
        "year": "Publication year (integer, e.g., 2024)",
        "difficulty": "Difficulty level (beginner, intermediate, advanced)",
        "category": "Document category (tutorial, reference, guide, api-docs)",
        "author": "Document author name",
        "version": "Software version (e.g., '3.0', '2.1')",
    }
)

result = parser.parse("Show me advanced Python tutorials from 2024")
# StructuredQuery(
#     semantic_query="Python tutorials",
#     filters={
#         "language": {"$eq": "python"},
#         "year": {"$eq": 2024},
#         "difficulty": {"$eq": "advanced"},
#     },
#     limit=None,
# )
```

### Self-Querying Retriever

```python
class SelfQueryingRetriever:
    """Retriever that uses an LLM to parse queries into
    semantic search + metadata filters.
    """

    def __init__(
        self,
        vector_store,
        parser: SelfQueryParser,
        default_k: int = 10,
    ):
        self.store = vector_store
        self.parser = parser
        self.default_k = default_k

    def invoke(self, query: str) -> list:
        """Parse the query and retrieve with filters."""
        # Step 1: Parse query into components
        structured = self.parser.parse(query)

        # Step 2: Build vector store filter
        vs_filter = self._build_filter(structured.filters)

        # Step 3: Search with both semantic query and filter
        k = structured.limit or self.default_k

        results = self.store.similarity_search(
            query=structured.semantic_query,
            k=k,
            filter=vs_filter,
        )

        return results

    def _build_filter(self, filters: dict) -> dict | None:
        """Convert parsed filters to vector store filter format."""
        if not filters:
            return None

        conditions = []
        for field, constraint in filters.items():
            if isinstance(constraint, dict):
                conditions.append({field: constraint})
            else:
                # Simple equality
                conditions.append({field: {"$eq": constraint}})

        if len(conditions) == 1:
            return conditions[0]
        return {"$and": conditions}
```

---

## When to Use Self-Querying

### Decision Matrix

```
Q: Do your documents have structured metadata?
  No  -> Self-querying will not help. Use standard retrieval.
  Yes -> Continue.

Q: Do users frequently query with implicit filters?
  Rarely  -> Standard retrieval with good embeddings is sufficient.
  Often   -> Self-querying adds significant value.

Q: Is latency critical (sub-200ms)?
  Yes -> The extra LLM call adds 200-500ms. Consider pre-computed
         filter suggestions instead.
  No  -> Self-querying is a good fit.

Q: Are your metadata fields well-defined and documented?
  Yes -> Self-querying works well with clear attribute descriptions.
  No  -> The LLM parser will struggle with ambiguous attributes.
```

### Cost-Benefit Analysis

| Factor | Standard Retrieval | Self-Querying |
|--------|-------------------|---------------|
| Latency | 50-200ms | 250-700ms (+LLM parse) |
| Cost | Embedding only | Embedding + LLM parse call |
| Precision on filtered queries | Low-Medium | High |
| Precision on open queries | High | High (no filters applied) |
| Complexity | Low | Medium |

---

## Attribute Design Best Practices

### Good Attribute Descriptions

```python
# Good: specific, with examples and valid values
GOOD_ATTRIBUTES = {
    "language": (
        "Programming language of the document. "
        "Values: python, javascript, typescript, rust, go, java"
    ),
    "year": (
        "Publication year as integer. Range: 2020-2025"
    ),
    "difficulty": (
        "Content difficulty level. "
        "Values: beginner, intermediate, advanced"
    ),
    "doc_type": (
        "Type of document. "
        "Values: tutorial, reference, api-docs, guide, changelog"
    ),
}

# Bad: vague, no valid values listed
BAD_ATTRIBUTES = {
    "language": "The language",
    "year": "When it was written",
    "difficulty": "How hard it is",
    "doc_type": "The type",
}
```

---

## Common Pitfalls

1. **Not providing attribute descriptions to the parser.** Without descriptions, the LLM guesses what filters are available and produces invalid filter keys. Always list available attributes with their valid values.
2. **Applying self-querying to all queries.** Queries like "explain recursion" have no filter component. The LLM parser adds latency for no benefit. Detect filter-like queries before invoking the parser.
3. **Not handling parser failures gracefully.** When the LLM fails to parse or produces invalid JSON, fall back to standard vector search using the original query. Never let a parser error block the query.
4. **Too many filterable attributes.** More than 10-15 attributes confuses the LLM parser. Focus on the 5-8 most commonly filtered attributes.
5. **Not testing with ambiguous queries.** "Show me recent docs" is ambiguous -- does "recent" mean last month or last year? Test with edge cases and provide clear attribute descriptions that handle ambiguity.
6. **Ignoring the empty results problem.** Overly specific filters (year=2024 AND language=cobol AND difficulty=advanced) may match zero documents. Detect empty results and suggest relaxing filters.

---

## References

- LangChain SelfQueryRetriever: https://python.langchain.com/docs/how_to/self_query/
- Kor structured extraction: https://github.com/eyurtsev/kor
- Weaviate filtered search: https://weaviate.io/developers/weaviate/search/filters
