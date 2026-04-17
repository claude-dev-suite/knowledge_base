# RAG Cost Math

## Overview / TL;DR

RAG costs decompose into four categories: embedding (ingestion + query-time), vector database storage and queries, re-ranking, and LLM generation. A naive RAG pipeline costs $0.001-0.005 per query; an advanced pipeline with re-ranking costs $0.005-0.02; an agentic pipeline costs $0.02-0.10. This document provides exact formulas, comparison tables for cheap vs quality configurations, and strategies for reducing cost without sacrificing retrieval quality.

---

## Cost Formula

The total cost per query is:

```
Cost/query = C_embed_query + C_retrieve + C_rerank + C_llm_input + C_llm_output
```

Where:
- `C_embed_query` = cost to embed the user query
- `C_retrieve` = vector DB query cost (often free for self-hosted, metered for managed)
- `C_rerank` = re-ranker cost per candidate batch
- `C_llm_input` = (system prompt + retrieved context + query) * price per input token
- `C_llm_output` = generated answer * price per output token

For pipelines with query rewriting or multi-query:
```
Cost/query += C_rewrite_llm + (N_queries - 1) * C_embed_query
```

---

## Component Costs (2025 Pricing)

### Embedding Models

| Model | Price | Dimensions | Quality (MTEB) | Notes |
|-------|-------|-----------|-----------------|-------|
| OpenAI text-embedding-3-small | $0.02/M tokens | 1536 | 62.3 | Cheapest API option |
| OpenAI text-embedding-3-large | $0.13/M tokens | 3072 | 64.6 | Higher quality |
| Voyage AI voyage-3 | $0.06/M tokens | 1024 | 67.5 | Good quality/price ratio |
| Cohere embed-v4 | $0.10/M tokens | 1024 | 66.2 | Supports search types |
| BAAI/bge-base-en-v1.5 (local) | ~$0.00/M tokens | 768 | 63.6 | Free but needs GPU |
| BAAI/bge-large-en-v1.5 (local) | ~$0.00/M tokens | 1024 | 64.2 | Free but needs GPU |

Local model "cost" is the GPU instance cost amortized over queries. A $0.50/hr GPU instance processing 100K queries/hr costs $0.000005/query -- effectively free.

### Vector Databases

| Service | Free Tier | Paid Pricing | Storage | Notes |
|---------|-----------|-------------|---------|-------|
| Pinecone | 100K vectors | $0.096/hr (s1.x1) | ~$70/mo for 1M vectors | Managed, serverless available |
| Qdrant Cloud | 1GB free | $0.044/GB/mo | ~$45/mo for 1M vectors | Good price/performance |
| Weaviate Cloud | 100K objects | $0.175/1M objects/mo | ~$25/mo for 1M vectors | Hybrid search built-in |
| Chroma (self-hosted) | Unlimited | Server cost only | $0/mo (plus server) | Dev/small-scale |
| pgvector (PostgreSQL) | N/A | DB cost only | Existing DB cost | Good if you already have PG |

**Cost per 1M vectors** (768-dim, approximate monthly):

| Service | Storage | Query Cost (1M queries) | Total Monthly |
|---------|---------|------------------------|---------------|
| Pinecone Serverless | ~$8 | ~$8 (read units) | ~$16 |
| Qdrant Cloud | ~$45 | Included | ~$45 |
| Self-hosted (t3.medium) | ~$30 (instance) | Included | ~$30 |
| pgvector (RDS db.t3.medium) | ~$30 (instance) | Included | ~$30 |

### Re-rankers

| Service | Pricing | Per Query (20 candidates) | Quality |
|---------|---------|--------------------------|---------|
| Cohere Rerank v3.5 | $2.00/1K searches | $0.002 | Best |
| Jina Reranker v2 | $0.02/1K documents | $0.0004 | Good |
| Voyage AI reranker | $0.05/1K documents | $0.001 | Good |
| bge-reranker-v2-m3 (local) | GPU cost only | ~$0.00001 | Good |
| LLM-based (Haiku) | Token pricing | ~$0.001-0.003 | Variable |

### LLM Generation

| Model | Input ($/M tokens) | Output ($/M tokens) | Notes |
|-------|-------------------|---------------------|-------|
| Claude Haiku 3.5 | $0.80 | $4.00 | Fast, cheap |
| Claude Sonnet 4 | $3.00 | $15.00 | Best quality/speed balance |
| Claude Opus 4 | $15.00 | $75.00 | Highest quality |
| GPT-4o-mini | $0.15 | $0.60 | Very cheap |
| GPT-4o | $2.50 | $10.00 | Good quality |
| Llama 3.1 70B (self-hosted) | ~$0.50 (amortized) | ~$0.50 | Requires GPU infra |

---

## Worked Examples

### Example 1: Budget RAG ($0.001/query)

**Configuration**: Local embeddings, Chroma, no re-ranking, GPT-4o-mini.

```
Query embedding (local):                    $0.000000
Vector retrieval (self-hosted Chroma):       $0.000000
Re-ranking:                                  $0.000000  (skipped)
LLM input (500 system + 2000 context + 100 query = 2600 tokens):
    2600 * $0.15 / 1M =                     $0.000390
LLM output (~500 tokens):
    500 * $0.60 / 1M =                      $0.000300
---------------------------------------------------------
Total per query:                             $0.000690

At 10,000 queries/month:                     $6.90
Plus server costs (~$30/mo):                 $36.90/month
```

### Example 2: Production RAG ($0.008/query)

**Configuration**: OpenAI embeddings, Pinecone serverless, Cohere reranking, Claude Sonnet.

```
Query embedding (text-embedding-3-small, ~50 tokens):
    50 * $0.02 / 1M =                       $0.000001
Vector retrieval (Pinecone serverless):      $0.000008  (read units)
Cohere re-ranking (20 candidates):           $0.002000
LLM input (500 system + 3000 context + 100 query = 3600 tokens):
    3600 * $3.00 / 1M =                     $0.010800
LLM output (~500 tokens):
    500 * $15.00 / 1M =                     $0.007500
---------------------------------------------------------
Total per query:                             $0.020309

Wait -- that's $0.02, not $0.008. Let's optimize.
```

**Optimized production**: Use Haiku for query rewriting, smaller context window.

```
Query rewriting (Haiku, ~200 tokens in + 50 out):
    200 * $0.80 / 1M + 50 * $4.00 / 1M =   $0.000360
Query embedding (local bge-base):            $0.000000
Vector retrieval (Qdrant Cloud):             $0.000005
Local re-ranking (bge-reranker):             $0.000001
LLM input (300 system + 2000 context + 100 query = 2400 tokens):
    2400 * $3.00 / 1M =                     $0.007200
LLM output (~400 tokens):
    400 * $15.00 / 1M =                     $0.006000
---------------------------------------------------------
Total per query:                             $0.013566

At 10,000 queries/month:                     $135.66
Plus infra (~$75/mo):                        $210.66/month
```

### Example 3: High-Quality Agentic RAG ($0.05/query)

**Configuration**: Claude Sonnet for agent, 3 retrieval iterations, Cohere reranking.

```
Agent planning (Sonnet, ~500 in + 200 out):
    500 * $3.00/1M + 200 * $15.00/1M =      $0.004500

Iteration 1:
    Query embed (local):                     $0.000000
    Vector retrieval:                        $0.000005
    Cohere rerank:                           $0.002000
    Agent evaluation (Sonnet, ~1500 in + 100 out):
        1500 * $3.00/1M + 100 * $15.00/1M = $0.006000

Iteration 2:
    Query embed (local):                     $0.000000
    Vector retrieval:                        $0.000005
    Cohere rerank:                           $0.002000
    Agent evaluation (Sonnet, ~2500 in + 100 out):
        2500 * $3.00/1M + 100 * $15.00/1M = $0.009000

Final generation (Sonnet, ~4000 in + 800 out):
    4000 * $3.00/1M + 800 * $15.00/1M =     $0.024000
---------------------------------------------------------
Total per query:                             $0.047510

At 10,000 queries/month:                     $475.10
Plus infra (~$100/mo):                       $575.10/month
```

---

## Configuration Comparison Table

| Config | Embed | Vector DB | Reranker | LLM | $/query | $/10K queries | Quality |
|--------|-------|-----------|----------|-----|---------|---------------|---------|
| Ultra-cheap | Local bge | Chroma | None | GPT-4o-mini | $0.0007 | $7 | Low |
| Budget | Local bge | pgvector | Local bge-reranker | GPT-4o-mini | $0.001 | $10 | Medium |
| Standard | text-emb-3-small | Pinecone | Cohere | Claude Sonnet | $0.02 | $200 | High |
| Optimized | Local bge | Qdrant | Local bge-reranker | Claude Sonnet | $0.013 | $130 | High |
| Premium | voyage-3 | Pinecone | Cohere | Claude Opus | $0.10 | $1,000 | Highest |
| Agentic | Local bge | Qdrant | Cohere | Claude Sonnet x3 | $0.05 | $500 | Very High |

---

## Ingestion Costs (One-Time + Incremental)

Ingestion costs are often overlooked but can be significant for large corpora.

### Embedding Cost at Ingestion

```
Total ingestion cost = total_tokens * embedding_price_per_token
```

| Corpus Size | Avg Tokens/Doc | Total Tokens | text-emb-3-small | voyage-3 | Local (free) |
|-------------|---------------|--------------|------------------|----------|-------------|
| 1,000 docs | 2,000 | 2M | $0.04 | $0.12 | $0.00 |
| 10,000 docs | 2,000 | 20M | $0.40 | $1.20 | $0.00 |
| 100,000 docs | 2,000 | 200M | $4.00 | $12.00 | $0.00 |
| 1,000,000 docs | 2,000 | 2B | $40.00 | $120.00 | $0.00 |

Important: Re-chunking (changing chunk size or strategy) requires re-embedding the entire corpus.

### Contextual Retrieval Ingestion Cost

If using Anthropic's contextual retrieval (prepending context via Claude), there is an additional LLM cost per chunk:

```
Context generation cost = num_chunks * avg_tokens_per_call * price_per_token
```

With prompt caching (90% cost reduction):

| Chunks | Without Caching | With Caching | Savings |
|--------|----------------|-------------|---------|
| 10,000 | ~$15 | ~$1.50 | 90% |
| 100,000 | ~$150 | ~$15 | 90% |
| 1,000,000 | ~$1,500 | ~$150 | 90% |

See the contextual-retrieval KB entry for detailed implementation.

---

## Token Budget Optimization

The largest cost driver is usually LLM input tokens (context). Optimizing the token budget directly reduces cost.

### Strategy 1: Compress Context

```python
def compress_chunks(chunks: list[str], max_tokens: int = 2000) -> str:
    """Truncate each chunk to essential content, fit within budget."""
    import tiktoken
    enc = tiktoken.encoding_for_model("gpt-4")

    budget_per_chunk = max_tokens // len(chunks)
    compressed = []

    for chunk in chunks:
        tokens = enc.encode(chunk)
        if len(tokens) > budget_per_chunk:
            truncated = enc.decode(tokens[:budget_per_chunk])
            compressed.append(truncated + "...")
        else:
            compressed.append(chunk)

    return "\n\n---\n\n".join(compressed)
```

### Strategy 2: Use Fewer, Better Chunks

Re-ranking lets you retrieve many candidates but pass only the top 3-5 to the LLM. Fewer chunks = fewer input tokens = lower cost.

| Chunks in Context | Avg Input Tokens | Sonnet Cost (input) | Quality Impact |
|------------------|-----------------|--------------------|----|
| 3 chunks | 1,500 | $0.0045 | May miss relevant info |
| 5 chunks | 2,500 | $0.0075 | Good balance |
| 10 chunks | 5,000 | $0.0150 | Diminishing returns |
| 20 chunks | 10,000 | $0.0300 | Usually wasteful |

### Strategy 3: Tiered Model Selection

Use the cheapest model that meets quality requirements for each task:

```python
def select_model(query_complexity: str) -> str:
    """Route to appropriate model based on query complexity."""
    if query_complexity == "simple":
        return "claude-haiku-4-20250514"   # $0.80/$4.00 per M tokens
    elif query_complexity == "moderate":
        return "claude-sonnet-4-20250514"  # $3.00/$15.00 per M tokens
    else:
        return "claude-opus-4-20250514"    # $15.00/$75.00 per M tokens


def classify_complexity(question: str, context_chunks: list[str]) -> str:
    """Simple heuristic for routing. Replace with LLM classifier if needed."""
    if len(question.split()) < 10 and len(context_chunks) <= 3:
        return "simple"
    elif any(word in question.lower() for word in ["compare", "analyze", "explain why", "tradeoffs"]):
        return "complex"
    else:
        return "moderate"
```

**Cost impact of tiered routing**:

| Query Mix | All-Sonnet | Tiered (60% Haiku / 30% Sonnet / 10% Opus) |
|-----------|-----------|---------------------------------------------|
| 10K queries | $135 | $68 |
| Savings | Baseline | ~50% |

### Strategy 4: Prompt Caching

If your system prompt + few-shot examples are large and identical across queries, use Anthropic's prompt caching:

```python
import anthropic

client = anthropic.Anthropic()

# The system prompt is cached across requests (90% discount on cache hits)
response = client.messages.create(
    model="claude-sonnet-4-20250514",
    max_tokens=1000,
    system=[
        {
            "type": "text",
            "text": "You are an expert assistant. [... long system prompt ...]",
            "cache_control": {"type": "ephemeral"},  # Cache this block
        }
    ],
    messages=[{
        "role": "user",
        "content": f"Context:\n{context}\n\nQuestion: {question}",
    }],
)
```

| System Prompt Size | Without Caching | With Caching (90% hits) | Monthly Savings (10K queries) |
|-------------------|----------------|------------------------|------------------------------|
| 500 tokens | $0.0015/query | $0.00015/query | $13.50 |
| 2,000 tokens | $0.006/query | $0.0006/query | $54.00 |
| 5,000 tokens | $0.015/query | $0.0015/query | $135.00 |

---

## Monthly Cost Projections

### By Query Volume

Assuming "Optimized" configuration ($0.013/query) + $75/mo infrastructure:

| Monthly Queries | Query Cost | Infra Cost | Total | Per-Query Effective |
|-----------------|-----------|-----------|-------|-------------------|
| 1,000 | $13 | $75 | $88 | $0.088 |
| 10,000 | $130 | $75 | $205 | $0.021 |
| 100,000 | $1,300 | $150 | $1,450 | $0.015 |
| 1,000,000 | $13,000 | $500 | $13,500 | $0.014 |

Note: Infrastructure costs increase with scale (more replicas, higher-tier DB instances) but grow sublinearly relative to query volume.

### Break-Even: RAG vs Long-Context

At what query volume does RAG become cheaper than stuffing everything in context?

Assume 100-page knowledge base (~50K tokens):

```
Long-context cost/query:
    Input: 50,000 tokens * $3.00/M = $0.15
    Output: 500 tokens * $15.00/M = $0.0075
    Total: $0.1575/query

RAG cost/query (optimized):
    Total: $0.013/query

Break-even:
    RAG infra cost = $75/month
    Savings per query = $0.1575 - $0.013 = $0.1445
    Break-even queries = $75 / $0.1445 = 519 queries/month
```

If you expect more than ~500 queries/month on a 100-page corpus, RAG is cheaper. For smaller corpora or lower query volumes, long-context may be more economical.

---

## Common Pitfalls

1. **Ignoring ingestion costs.** A one-time embedding of 1M documents with a paid API costs $40-120. Re-chunking means re-embedding. Factor this into the total cost of ownership.
2. **Using the most expensive model for everything.** Query rewriting, classification, and evaluation do not need Opus or even Sonnet. Use Haiku for auxiliary LLM calls.
3. **Not measuring actual token usage.** Estimated costs based on average chunk sizes can be 2-3x off. Log actual token counts per query and compute real costs.
4. **Over-retrieving context.** Passing 20 chunks to the LLM when 5 would suffice triples input costs with minimal quality improvement.
5. **Forgetting about re-ranker costs.** Cohere Rerank at $2/1K searches adds $0.002/query. At 1M queries/month, that is $2,000 just for re-ranking. A local model is effectively free.
6. **Not using prompt caching.** If your system prompt exceeds 500 tokens and is identical across queries, you are leaving money on the table by not caching it.
7. **Scaling vector DB prematurely.** Pinecone's highest tiers cost hundreds per month. Start with a cheaper option (pgvector, self-hosted Qdrant) and migrate when you outgrow it.

---

## Cost Monitoring

```python
"""Track and alert on RAG pipeline costs."""

from dataclasses import dataclass, field
import logging

logger = logging.getLogger(__name__)


@dataclass
class CostTracker:
    """Accumulate per-query costs for monitoring and alerting."""
    costs: dict = field(default_factory=lambda: {
        "embedding": 0.0,
        "retrieval": 0.0,
        "reranking": 0.0,
        "llm_input": 0.0,
        "llm_output": 0.0,
        "query_rewrite": 0.0,
    })

    def add(self, category: str, amount: float):
        self.costs[category] = self.costs.get(category, 0.0) + amount

    def total(self) -> float:
        return sum(self.costs.values())

    def log_query(self, query_id: str):
        logger.info(
            "RAG query cost",
            extra={
                "query_id": query_id,
                "total_cost": self.total(),
                **self.costs,
            },
        )

    def check_budget(self, max_per_query: float = 0.05):
        if self.total() > max_per_query:
            logger.warning(
                f"Query cost ${self.total():.4f} exceeds budget ${max_per_query:.4f}",
                extra={"costs": self.costs},
            )


def estimate_llm_cost(
    input_tokens: int,
    output_tokens: int,
    model: str,
) -> float:
    """Estimate LLM cost based on token counts and model pricing."""
    pricing = {
        "claude-haiku-4-20250514": (0.80, 4.00),
        "claude-sonnet-4-20250514": (3.00, 15.00),
        "claude-opus-4-20250514": (15.00, 75.00),
        "gpt-4o-mini": (0.15, 0.60),
        "gpt-4o": (2.50, 10.00),
    }
    input_price, output_price = pricing.get(model, (3.00, 15.00))
    return (input_tokens * input_price / 1_000_000) + (output_tokens * output_price / 1_000_000)
```

---

## References

- Anthropic pricing -- https://www.anthropic.com/pricing
- OpenAI pricing -- https://openai.com/pricing
- Pinecone pricing -- https://www.pinecone.io/pricing/
- Qdrant pricing -- https://qdrant.tech/pricing/
- Cohere pricing -- https://cohere.com/pricing
- MTEB Leaderboard (embedding quality benchmarks) -- https://huggingface.co/spaces/mteb/leaderboard
