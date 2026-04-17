# Query Transformations

## Overview / TL;DR

User queries are often poorly suited for direct retrieval. They may be vague, conversational, use different vocabulary than the documents, or require information from multiple sources that no single query can surface. Query transformations are pre-retrieval techniques that modify, expand, or decompose the user's query before it hits the vector database. This document provides a taxonomy of all major query transformation techniques, explains when each is effective, and provides implementation guidance. The most impactful techniques are query rewriting (5-15% recall improvement), multi-query generation with fusion (10-20% improvement), and HyDE for abstract/conceptual queries (10-25% improvement on specific query types).

---

## Taxonomy of Query Transformation Techniques

```
Query Transformations
|
+--- Rewriting (single query -> better single query)
|    +--- LLM-based rewriting
|    +--- Keyword extraction
|    +--- De-conversationalization
|
+--- Expansion (single query -> multiple queries)
|    +--- Multi-query generation
|    +--- Sub-question decomposition
|    +--- Step-back prompting
|
+--- Hypothetical Document Generation
|    +--- HyDE (Hypothetical Document Embeddings)
|    +--- Hypothetical Answer Generation
|
+--- Routing (query -> appropriate index/tool)
|    +--- Intent classification
|    +--- Semantic routing
|    +--- Tool selection
|
+--- Compression (long conversational context -> focused query)
     +--- Conversation condensation
     +--- Context carryover
```

---

## Technique 1: Query Rewriting

Transform the user's raw query into a better search query. This is the simplest and most universally applicable transformation.

**Why it works**: Users phrase questions conversationally ("hey, can you tell me how to make the auth thing work with the new API?") but documents use formal terminology ("Authentication configuration for API v3"). Rewriting bridges this gap.

```python
import anthropic

client = anthropic.Anthropic()


def rewrite_query(question: str) -> str:
    """Rewrite a conversational question into a search-optimized query."""
    response = client.messages.create(
        model="claude-haiku-4-20250514",
        max_tokens=150,
        temperature=0,
        messages=[{
            "role": "user",
            "content": (
                "Rewrite the following question as a concise search query "
                "optimized for semantic similarity search against a technical "
                "documentation knowledge base. Focus on key technical terms. "
                "Return ONLY the rewritten query, nothing else.\n\n"
                f"Question: {question}"
            ),
        }],
    )
    return response.content[0].text.strip()


# Examples:
# "hey how do I make my app talk to the database?" -> "database connection configuration"
# "what's the deal with those timeout errors in prod?" -> "production timeout errors troubleshooting"
# "can I use SSO with the new system?" -> "SSO single sign-on integration setup"
```

**When to use**: Always. Query rewriting is cheap (single Haiku call, ~150ms, ~$0.0003) and almost never hurts.

**When it hurts**: Very specific queries with exact terms ("error code E-4021"). Rewriting may generalize away the specific term. Use the bypass heuristic:

```python
def should_rewrite(query: str) -> bool:
    """Skip rewriting for queries that are already specific."""
    # Short, specific queries don't need rewriting
    if len(query.split()) <= 4:
        return False
    # Queries with error codes or specific identifiers
    if any(char.isdigit() for char in query) and len(query.split()) <= 6:
        return False
    # Conversational queries always benefit
    if any(word in query.lower() for word in ["please", "could", "would", "hey", "can you"]):
        return True
    return len(query.split()) > 6
```

---

## Technique 2: Multi-Query Generation

Generate multiple search queries from different angles, retrieve for each, and fuse the results. This is the core of RAG-Fusion.

**Why it works**: A single query captures one perspective on the information need. "How to set up PostgreSQL replication" might miss documents that discuss "streaming replication," "logical replication," or "standby servers." Multiple queries cover more ground.

```python
def generate_multi_queries(question: str, n: int = 4) -> list[str]:
    """Generate N diverse search queries from different angles."""
    response = client.messages.create(
        model="claude-haiku-4-20250514",
        max_tokens=400,
        temperature=0.7,  # Higher temperature for diversity
        messages=[{
            "role": "user",
            "content": (
                f"You are an AI assistant helping to improve search retrieval. "
                f"Given the following question, generate {n} different search "
                f"queries that would help find relevant information from different "
                f"angles. Each query should approach the topic differently.\n\n"
                f"Original question: {question}\n\n"
                f"Return exactly {n} queries, one per line, nothing else."
            ),
        }],
    )
    queries = response.content[0].text.strip().split("\n")
    return [q.strip().lstrip("0123456789.-) ") for q in queries if q.strip()][:n]


# Example:
# Input: "How do I set up PostgreSQL replication?"
# Output:
#   1. "PostgreSQL streaming replication primary standby setup"
#   2. "configure pg_hba.conf and postgresql.conf for replication"
#   3. "PostgreSQL WAL archiving and replication slots"
#   4. "high availability PostgreSQL read replica configuration"
```

**Optimal query count**: 3-5 queries. Below 3, coverage is insufficient. Above 5, queries become redundant and fusion quality degrades.

---

## Technique 3: Reciprocal Rank Fusion (RRF)

Combine results from multiple queries (or multiple retrieval methods) into a single ranked list.

**The RRF Formula**:

```
RRF_score(document) = SUM over all rankings: 1 / (k + rank_in_ranking)
```

Where `k` is a constant (default 60) that controls how much top ranks are weighted over lower ranks.

```python
def reciprocal_rank_fusion(
    result_lists: list[list[dict]],
    k: int = 60,
) -> list[dict]:
    """Fuse multiple ranked result lists using RRF.

    Args:
        result_lists: List of ranked result lists. Each result must have an 'id' key.
        k: Ranking constant (default 60). Higher k = less emphasis on top ranks.

    Returns:
        Fused and re-ranked list of results.
    """
    scores = {}  # document_id -> cumulative RRF score
    doc_map = {}  # document_id -> document dict

    for results in result_lists:
        for rank, doc in enumerate(results, 1):
            doc_id = doc["id"]
            if doc_id not in scores:
                scores[doc_id] = 0.0
                doc_map[doc_id] = doc
            scores[doc_id] += 1.0 / (k + rank)

    # Sort by fused score
    fused = sorted(
        [(doc_map[doc_id], score) for doc_id, score in scores.items()],
        key=lambda x: x[1],
        reverse=True,
    )

    return [
        {**doc, "rrf_score": score}
        for doc, score in fused
    ]
```

**Why k=60**: The original RRF paper (Cormack et al., 2009) found k=60 to be robust across many datasets. Lower values (k=1-10) heavily favor top-ranked results. Higher values (k=100+) treat all ranks more equally.

```
Rank 1: score = 1/(60+1)  = 0.01639
Rank 2: score = 1/(60+2)  = 0.01613
Rank 5: score = 1/(60+5)  = 0.01538
Rank 10: score = 1/(60+10) = 0.01429
Rank 20: score = 1/(60+20) = 0.01250

The difference between rank 1 and rank 20 is only 24%.
This means a document ranked #20 by one query and #1 by another
scores higher than a document ranked #5 by all queries.
```

---

## Technique 4: Sub-Question Decomposition

Break complex questions into simpler sub-questions, retrieve for each, and synthesize.

**Why it works**: Complex questions like "Compare the authentication methods in API v2 vs v3 and recommend the best approach for a microservices architecture" require information from at least three different topics. No single retrieval can find all of it.

```python
def decompose_question(question: str) -> list[str]:
    """Break a complex question into retrievable sub-questions."""
    response = client.messages.create(
        model="claude-haiku-4-20250514",
        max_tokens=400,
        temperature=0,
        messages=[{
            "role": "user",
            "content": (
                "Break the following complex question into 2-4 simpler "
                "sub-questions that can each be answered independently. "
                "Each sub-question should target a specific piece of "
                "information needed to answer the original question.\n\n"
                f"Question: {question}\n\n"
                "Return one sub-question per line, nothing else."
            ),
        }],
    )
    questions = response.content[0].text.strip().split("\n")
    return [q.strip().lstrip("0123456789.-) ") for q in questions if q.strip()]


# Example:
# Input: "Compare auth methods in API v2 vs v3 for microservices"
# Output:
#   1. "What authentication methods does API v2 support?"
#   2. "What authentication methods does API v3 support?"
#   3. "Best authentication approach for microservices architecture"
```

**When to use**: Multi-hop questions, comparison questions, questions that span multiple topics or documents.

**Pipeline**:

```python
async def decompose_and_retrieve(question: str, index, top_k: int = 5) -> list[dict]:
    """Decompose question, retrieve for each sub-question, fuse results."""
    sub_questions = decompose_question(question)

    # Retrieve for each sub-question
    all_results = []
    for sq in sub_questions:
        results = index.search(sq, top_k=top_k * 2)
        all_results.append(results)

    # Also retrieve for the original question
    original_results = index.search(question, top_k=top_k * 2)
    all_results.append(original_results)

    # Fuse all results
    fused = reciprocal_rank_fusion(all_results)
    return fused[:top_k]
```

---

## Technique 5: Step-Back Prompting

Generate a more general, abstract version of the specific query. Useful when the specific query is too narrow to match relevant documents.

```python
def step_back_query(question: str) -> str:
    """Generate a broader, more abstract query."""
    response = client.messages.create(
        model="claude-haiku-4-20250514",
        max_tokens=150,
        temperature=0,
        messages=[{
            "role": "user",
            "content": (
                "Given the following specific question, generate a more general "
                "'step-back' question that covers the broader topic. The step-back "
                "question should help retrieve background information needed to "
                "answer the original question.\n\n"
                f"Specific question: {question}\n\n"
                "Step-back question:"
            ),
        }],
    )
    return response.content[0].text.strip()


# Examples:
# "Why does my React component re-render when I use useContext?"
# -> "How does React context trigger re-renders in the component tree?"

# "How do I fix ORA-12154 in my Spring Boot app?"
# -> "How does Oracle TNS name resolution work in Java applications?"
```

**When to use**: Debugging questions (specific error -> general mechanism), "why" questions, questions that require understanding the underlying system.

---

## Technique 6: Conversation Condensation

In multi-turn conversations, the current user message may be incomplete without prior context. Condensation creates a standalone query from the conversation history.

```python
def condense_conversation(
    chat_history: list[dict],
    current_question: str,
) -> str:
    """Create a standalone query from conversation history + current question."""
    history_text = "\n".join(
        f"{msg['role'].upper()}: {msg['content']}"
        for msg in chat_history[-6:]  # Last 3 turns
    )

    response = client.messages.create(
        model="claude-haiku-4-20250514",
        max_tokens=200,
        temperature=0,
        messages=[{
            "role": "user",
            "content": (
                "Given the following conversation history and a follow-up question, "
                "rewrite the follow-up question as a standalone search query that "
                "includes all necessary context from the conversation. "
                "Return ONLY the standalone query.\n\n"
                f"Conversation history:\n{history_text}\n\n"
                f"Follow-up question: {current_question}\n\n"
                "Standalone query:"
            ),
        }],
    )
    return response.content[0].text.strip()


# Example:
# History: "USER: Tell me about PostgreSQL replication"
#          "ASSISTANT: PostgreSQL supports streaming and logical replication..."
# Current: "How do I set that up?"
# Condensed: "How to set up PostgreSQL streaming replication"
```

**When to use**: Any RAG system with multi-turn conversations. Without condensation, "How do I set that up?" retrieves random results because "that" has no referent.

---

## Combining Techniques: The Full Pipeline

```python
class QueryTransformationPipeline:
    """Configurable query transformation pipeline."""

    def __init__(
        self,
        enable_rewriting: bool = True,
        enable_multi_query: bool = True,
        enable_decomposition: bool = False,  # For complex queries only
        enable_step_back: bool = False,      # For specific/narrow queries
        enable_hyde: bool = False,           # See hyde-deep.md
        num_multi_queries: int = 4,
    ):
        self.client = anthropic.Anthropic()
        self.enable_rewriting = enable_rewriting
        self.enable_multi_query = enable_multi_query
        self.enable_decomposition = enable_decomposition
        self.enable_step_back = enable_step_back
        self.enable_hyde = enable_hyde
        self.num_multi_queries = num_multi_queries

    def transform(
        self,
        question: str,
        chat_history: list[dict] = None,
    ) -> list[str]:
        """Transform query into one or more search queries."""
        queries = []

        # Step 0: Condense conversation if needed
        if chat_history:
            question = condense_conversation(chat_history, question)

        # Step 1: Rewrite for search
        if self.enable_rewriting:
            rewritten = rewrite_query(question)
            queries.append(rewritten)
        else:
            queries.append(question)

        # Step 2: Generate additional queries
        if self.enable_multi_query:
            multi = generate_multi_queries(question, n=self.num_multi_queries)
            queries.extend(multi)

        if self.enable_decomposition:
            sub_qs = decompose_question(question)
            queries.extend(sub_qs)

        if self.enable_step_back:
            step_back = step_back_query(question)
            queries.append(step_back)

        # Deduplicate while preserving order
        seen = set()
        unique_queries = []
        for q in queries:
            q_lower = q.lower().strip()
            if q_lower not in seen:
                seen.add(q_lower)
                unique_queries.append(q)

        return unique_queries

    def transform_and_retrieve(
        self,
        question: str,
        index,
        top_k: int = 5,
        per_query_k: int = 15,
        chat_history: list[dict] = None,
    ) -> list[dict]:
        """Full pipeline: transform -> multi-retrieve -> fuse."""
        queries = self.transform(question, chat_history)

        # Retrieve for each query
        all_results = []
        for q in queries:
            results = index.search(q, top_k=per_query_k)
            all_results.append(results)

        # Fuse with RRF
        fused = reciprocal_rank_fusion(all_results)
        return fused[:top_k]
```

---

## Performance Impact

Based on benchmarks across technical documentation corpora:

| Technique | Recall@10 Improvement | Latency Added | Cost per Query |
|-----------|---------------------|--------------|----------------|
| Query rewriting | +5-15% | 100-200ms | ~$0.0003 |
| Multi-query (4 queries) | +10-20% | 150-300ms | ~$0.001 |
| Sub-question decomposition | +15-25% (complex queries) | 200-400ms | ~$0.001 |
| Step-back prompting | +5-10% (narrow queries) | 100-200ms | ~$0.0003 |
| HyDE | +10-25% (conceptual queries) | 200-500ms | ~$0.001 |
| Conversation condensation | +20-40% (follow-up queries) | 100-200ms | ~$0.0003 |
| Combined (rewrite + multi-query) | +15-25% | 200-400ms | ~$0.0013 |

Note: These improvements are on top of whatever retrieval improvements you get from chunking, hybrid search, and reranking. They compound with those techniques.

---

## Common Pitfalls

1. **Applying all techniques to every query.** This adds latency and cost. Use routing to decide which transformations apply to each query type.
2. **Generating too many multi-queries.** Beyond 5 queries, results become redundant and RRF fusion quality degrades. 3-5 is optimal.
3. **Not condensing conversation history.** Without condensation, follow-up queries like "what about the other one?" produce garbage retrieval.
4. **Using an expensive model for query transformation.** Haiku is sufficient for all transformation tasks. Sonnet adds 3x cost and latency with minimal quality improvement.
5. **Rewriting very specific queries.** Error codes, product IDs, and exact identifiers should pass through to retrieval unchanged. Rewriting may generalize them away.
6. **Not measuring transformation impact.** Each technique should be A/B tested on your eval set. Some techniques may hurt performance on your specific query distribution.

---

## References

- Gao et al., "Precise Zero-Shot Dense Retrieval without Relevance Labels" (HyDE, 2022) -- https://arxiv.org/abs/2212.10496
- Raudaschl, "RAG-Fusion" (2023) -- https://github.com/Raudaschl/rag-fusion
- Cormack et al., "Reciprocal Rank Fusion outperforms Condorcet and individual Rank Learning Methods" (2009) -- https://plg.uwaterloo.ca/~gvcormac/cormacksigir09-rrf.pdf
- Zheng et al., "Take a Step Back: Evoking Reasoning via Abstraction" (2023) -- https://arxiv.org/abs/2310.06117
- LangChain query transformation docs -- https://python.langchain.com/docs/how_to/query_transformations/
