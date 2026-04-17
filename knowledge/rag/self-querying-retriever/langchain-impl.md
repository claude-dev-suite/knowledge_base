# LangChain SelfQueryRetriever -- Setup, Attribute Descriptions, and Comparators

## TL;DR

LangChain's `SelfQueryRetriever` is a ready-to-use implementation that uses an LLM to automatically decompose user queries into semantic search terms and structured metadata filters. It handles the LLM prompting, output parsing, filter translation to vector-store-native syntax, and query execution in a single chain. This article covers complete setup with Chroma and Pinecone backends, attribute info configuration (which controls what the LLM can filter on), comparator types, custom filter functions, and integration with existing RAG pipelines.

---

## Basic Setup

### Installation

```python
# pip install langchain langchain-anthropic langchain-chroma langchain-openai

from langchain.retrievers.self_query.base import SelfQueryRetriever
from langchain.chains.query_constructor.schema import AttributeInfo
from langchain_anthropic import ChatAnthropic
from langchain_openai import OpenAIEmbeddings
from langchain_chroma import Chroma
```

### Document Preparation

```python
from langchain_core.documents import Document

# Documents with rich metadata for self-querying
documents = [
    Document(
        page_content="Complete guide to building REST APIs with FastAPI",
        metadata={
            "language": "python",
            "framework": "fastapi",
            "year": 2024,
            "difficulty": "intermediate",
            "doc_type": "tutorial",
            "author": "Jane Smith",
            "word_count": 3500,
        },
    ),
    Document(
        page_content="Introduction to React hooks and state management",
        metadata={
            "language": "javascript",
            "framework": "react",
            "year": 2024,
            "difficulty": "beginner",
            "doc_type": "tutorial",
            "author": "John Doe",
            "word_count": 2200,
        },
    ),
    Document(
        page_content="Advanced Rust ownership and borrowing patterns",
        metadata={
            "language": "rust",
            "framework": "none",
            "year": 2023,
            "difficulty": "advanced",
            "doc_type": "guide",
            "author": "Alice Johnson",
            "word_count": 5000,
        },
    ),
    Document(
        page_content="PostgreSQL query optimization and indexing strategies",
        metadata={
            "language": "sql",
            "framework": "postgresql",
            "year": 2024,
            "difficulty": "advanced",
            "doc_type": "reference",
            "author": "Bob Williams",
            "word_count": 4200,
        },
    ),
    Document(
        page_content="Getting started with Django REST framework",
        metadata={
            "language": "python",
            "framework": "django",
            "year": 2023,
            "difficulty": "beginner",
            "doc_type": "tutorial",
            "author": "Jane Smith",
            "word_count": 1800,
        },
    ),
]
```

### Attribute Info Configuration

```python
# AttributeInfo tells the LLM what metadata fields exist
# and what values they can take. This is critical for
# accurate filter generation.

metadata_field_info = [
    AttributeInfo(
        name="language",
        description=(
            "The programming language of the document. "
            "One of: 'python', 'javascript', 'typescript', "
            "'rust', 'go', 'java', 'sql'"
        ),
        type="string",
    ),
    AttributeInfo(
        name="framework",
        description=(
            "The framework or technology covered. "
            "Examples: 'fastapi', 'django', 'react', 'vue', "
            "'postgresql', 'none'"
        ),
        type="string",
    ),
    AttributeInfo(
        name="year",
        description="The publication year as an integer (e.g., 2023, 2024)",
        type="integer",
    ),
    AttributeInfo(
        name="difficulty",
        description=(
            "The difficulty level. "
            "One of: 'beginner', 'intermediate', 'advanced'"
        ),
        type="string",
    ),
    AttributeInfo(
        name="doc_type",
        description=(
            "The type of document. "
            "One of: 'tutorial', 'reference', 'guide', "
            "'api-docs', 'changelog'"
        ),
        type="string",
    ),
    AttributeInfo(
        name="author",
        description="The author's full name",
        type="string",
    ),
    AttributeInfo(
        name="word_count",
        description="The approximate word count of the document",
        type="integer",
    ),
]

document_content_description = (
    "Technical documentation including tutorials, guides, "
    "and reference materials for software development"
)
```

### Creating the SelfQueryRetriever

```python
# Create vector store
embeddings = OpenAIEmbeddings(model="text-embedding-3-small")
vectorstore = Chroma.from_documents(
    documents, embeddings, collection_name="docs"
)

# Create LLM for query parsing
llm = ChatAnthropic(model="claude-haiku-4-20250514", temperature=0)

# Create self-querying retriever
retriever = SelfQueryRetriever.from_llm(
    llm=llm,
    vectorstore=vectorstore,
    document_contents=document_content_description,
    metadata_field_info=metadata_field_info,
    search_kwargs={"k": 5},
    verbose=True,  # Log parsed queries for debugging
)

# Usage examples
# Query with implicit filters
results = retriever.invoke("Python tutorials for beginners")
# Parsed: query="tutorials for beginners", filter: language=="python" AND difficulty=="beginner"

results = retriever.invoke("Advanced guides published after 2023")
# Parsed: query="guides", filter: difficulty=="advanced" AND year>2023

results = retriever.invoke("What did Jane Smith write about APIs?")
# Parsed: query="APIs", filter: author=="Jane Smith"

results = retriever.invoke("Short tutorials under 2000 words")
# Parsed: query="tutorials", filter: doc_type=="tutorial" AND word_count<2000
```

---

## Comparator Types

### Available Comparators

| Comparator | Symbol | Example | Use Case |
|------------|--------|---------|----------|
| eq | == | `language == "python"` | Exact match |
| ne | != | `language != "sql"` | Exclusion |
| gt | > | `year > 2023` | Numeric range |
| gte | >= | `word_count >= 1000` | Minimum threshold |
| lt | < | `year < 2024` | Numeric range |
| lte | <= | `word_count <= 5000` | Maximum threshold |
| contain | contains | `framework contains "api"` | Substring match |
| like | like | `author like "%Smith%"` | Pattern match |
| in | in | `language in ["python", "javascript"]` | Multiple values |
| nin | not in | `difficulty nin ["beginner"]` | Exclusion list |

### Restricting Comparators

```python
from langchain.chains.query_constructor.schema import AttributeInfo
from langchain.chains.query_constructor.base import (
    StructuredQueryOutputParser,
    get_query_constructor_prompt,
)


# Restrict which comparators are available for each attribute
# This prevents the LLM from generating unsupported filters

restricted_attributes = [
    AttributeInfo(
        name="language",
        description="Programming language",
        type="string",
        # Only allow exact match and in-list for strings
    ),
    AttributeInfo(
        name="year",
        description="Publication year",
        type="integer",
        # Numeric attributes support all comparators
    ),
]

# You can also restrict globally by passing allowed_comparators
# to the query constructor
from langchain.chains.query_constructor.ir import (
    Comparator,
    Operator,
)

retriever = SelfQueryRetriever.from_llm(
    llm=llm,
    vectorstore=vectorstore,
    document_contents=document_content_description,
    metadata_field_info=restricted_attributes,
    allowed_comparators=[
        Comparator.EQ,
        Comparator.NE,
        Comparator.GT,
        Comparator.GTE,
        Comparator.LT,
        Comparator.LTE,
        Comparator.IN,
    ],
    allowed_operators=[
        Operator.AND,
        Operator.OR,
    ],
)
```

---

## Vector Store Backends

### Chroma

```python
from langchain_chroma import Chroma

vectorstore = Chroma.from_documents(
    documents,
    embeddings,
    collection_name="self_query_demo",
)

retriever = SelfQueryRetriever.from_llm(
    llm=llm,
    vectorstore=vectorstore,
    document_contents=document_content_description,
    metadata_field_info=metadata_field_info,
)

# Chroma filter format (generated automatically):
# {"$and": [{"language": {"$eq": "python"}}, {"year": {"$gt": 2023}}]}
```

### Pinecone

```python
from langchain_pinecone import PineconeVectorStore
from pinecone import Pinecone

pc = Pinecone(api_key="...")
index = pc.Index("self-query-demo")

vectorstore = PineconeVectorStore(
    index=index,
    embedding=embeddings,
    text_key="text",
)

retriever = SelfQueryRetriever.from_llm(
    llm=llm,
    vectorstore=vectorstore,
    document_contents=document_content_description,
    metadata_field_info=metadata_field_info,
)

# Pinecone filter format (generated automatically):
# {"$and": [{"language": {"$eq": "python"}}, {"year": {"$gt": 2023}}]}
```

### Weaviate

```python
from langchain_weaviate import WeaviateVectorStore
import weaviate

client = weaviate.connect_to_local()

vectorstore = WeaviateVectorStore(
    client=client,
    index_name="Documents",
    text_key="text",
    embedding=embeddings,
)

retriever = SelfQueryRetriever.from_llm(
    llm=llm,
    vectorstore=vectorstore,
    document_contents=document_content_description,
    metadata_field_info=metadata_field_info,
)
```

---

## Debugging and Logging

### Inspecting Parsed Queries

```python
from langchain.chains.query_constructor.base import (
    StructuredQueryOutputParser,
)


class DebugSelfQueryRetriever:
    """Wrapper that logs the parsed query for debugging."""

    def __init__(self, retriever: SelfQueryRetriever):
        self.retriever = retriever

    def invoke(self, query: str) -> list:
        """Retrieve with detailed logging of parsed components."""
        # Access the query constructor chain
        constructor = self.retriever.query_constructor
        structured_query = constructor.invoke({"query": query})

        print(f"Original query: {query}")
        print(f"Semantic query: {structured_query.query}")
        print(f"Filter: {structured_query.filter}")

        if structured_query.filter:
            print(f"Filter dict: {structured_query.filter.to_dict()}")

        results = self.retriever.invoke(query)

        print(f"Results: {len(results)} documents")
        for i, doc in enumerate(results):
            print(f"  [{i}] {doc.page_content[:80]}...")
            print(f"      Metadata: {doc.metadata}")

        return results


# Usage
debug_retriever = DebugSelfQueryRetriever(retriever)
results = debug_retriever.invoke("Python tutorials from 2024")
# Output:
# Original query: Python tutorials from 2024
# Semantic query: Python tutorials
# Filter: eq(language, "python") and eq(year, 2024)
# Results: 1 documents
#   [0] Complete guide to building REST APIs with FastAPI...
#       Metadata: {'language': 'python', 'framework': 'fastapi', ...}
```

---

## Integration with RAG Pipeline

```python
from langchain.chains import create_retrieval_chain
from langchain.chains.combine_documents import create_stuff_documents_chain
from langchain_core.prompts import ChatPromptTemplate


def create_self_querying_rag_chain(
    vectorstore,
    metadata_field_info: list[AttributeInfo],
    document_contents: str,
    llm_model: str = "claude-sonnet-4-20250514",
):
    """Create a full RAG chain with self-querying retrieval."""
    # Parser LLM (fast, cheap -- used for query parsing)
    parser_llm = ChatAnthropic(
        model="claude-haiku-4-20250514", temperature=0
    )

    # Generator LLM (powerful -- used for answer generation)
    generator_llm = ChatAnthropic(
        model=llm_model, temperature=0
    )

    # Self-querying retriever
    retriever = SelfQueryRetriever.from_llm(
        llm=parser_llm,
        vectorstore=vectorstore,
        document_contents=document_contents,
        metadata_field_info=metadata_field_info,
        search_kwargs={"k": 5},
    )

    # Answer generation
    prompt = ChatPromptTemplate.from_messages([
        (
            "system",
            "Answer the question based on the provided context. "
            "Cite the source documents by their metadata.\n\n"
            "Context:\n{context}",
        ),
        ("human", "{input}"),
    ])

    question_answer_chain = create_stuff_documents_chain(
        generator_llm, prompt
    )

    return create_retrieval_chain(retriever, question_answer_chain)


# Usage
chain = create_self_querying_rag_chain(
    vectorstore=vectorstore,
    metadata_field_info=metadata_field_info,
    document_contents=document_content_description,
)

result = chain.invoke({
    "input": "What Python tutorials are available for beginners?"
})
print(result["answer"])
```

---

## Common Pitfalls

1. **Vague attribute descriptions.** If you describe `year` as "the year" without specifying it is an integer, the LLM may generate string comparisons like `year == "2024"` instead of `year == 2024`. Be specific about types and valid values.
2. **Not testing with queries that have no filters.** "Explain recursion" has no filter component, but the LLM may hallucinate a filter. Verify that the parser handles pure semantic queries correctly.
3. **Using an expensive model for query parsing.** Query parsing is a simple extraction task. Claude Haiku or GPT-4o-mini is sufficient and 10x cheaper than using Claude Sonnet or GPT-4o.
4. **Not restricting allowed comparators.** If your vector store does not support `LIKE` or `CONTAINS`, but the LLM generates those comparators, the query fails. Restrict to comparators your backend supports.
5. **Metadata fields with inconsistent values.** If some documents have `language: "Python"` and others have `language: "python"`, the filter `language == "python"` misses half the results. Normalize metadata during ingestion.
6. **Not providing a document_contents description.** This description helps the LLM understand what the documents are about, improving both semantic query extraction and filter accuracy.

---

## References

- LangChain SelfQueryRetriever: https://python.langchain.com/docs/how_to/self_query/
- LangChain query constructor: https://python.langchain.com/docs/how_to/query_constructors/
- Chroma metadata filtering: https://docs.trychroma.com/docs/collections/filter
- Pinecone metadata filtering: https://docs.pinecone.io/guides/data/filter-with-metadata
