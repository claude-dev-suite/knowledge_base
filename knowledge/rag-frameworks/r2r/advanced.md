# R2R -- Advanced: Multi-Tenant, Agentic Workflows, and Custom Pipelines

## Overview

This guide covers advanced R2R patterns for production systems: multi-tenant isolation with collections and users, agentic RAG workflows where the LLM controls retrieval, custom pipeline configuration, evaluation strategies, and performance optimization. These patterns build on the basics in `getting-started.md` and the architecture in `overview.md`.

---

## Multi-Tenant Architecture

### User and Collection Model

R2R provides two levels of isolation:

1. **Users**: Individual accounts with authentication and document ownership
2. **Collections**: Groups of documents that can be shared across users or restricted

```python
from r2r import R2RClient

client = R2RClient("http://localhost:7272")

# Create tenant organizations as collections
org_a = client.collections.create(
    name="Acme Corp",
    description="Acme Corp knowledge base",
)
org_a_id = org_a["results"]["id"]

org_b = client.collections.create(
    name="Beta Inc",
    description="Beta Inc knowledge base",
)
org_b_id = org_b["results"]["id"]

# Create users for each tenant
user_a = client.users.create(
    email="alice@acme.com",
    password="secure-password-a",
)

user_b = client.users.create(
    email="bob@beta.com",
    password="secure-password-b",
)

# Add users to their respective collections
client.collections.add_user(
    collection_id=org_a_id,
    user_id=user_a["results"]["id"],
)

client.collections.add_user(
    collection_id=org_b_id,
    user_id=user_b["results"]["id"],
)
```

### Tenant-Scoped Ingestion

```python
# Ingest documents for Acme Corp
acme_doc = client.documents.create(
    file_path="./acme/playbook.pdf",
    metadata={
        "tenant": "acme",
        "department": "engineering",
    },
)
client.collections.add_document(
    collection_id=org_a_id,
    document_id=acme_doc["results"]["document_id"],
)

# Ingest documents for Beta Inc
beta_doc = client.documents.create(
    file_path="./beta/handbook.pdf",
    metadata={
        "tenant": "beta",
        "department": "operations",
    },
)
client.collections.add_document(
    collection_id=org_b_id,
    document_id=beta_doc["results"]["document_id"],
)
```

### Tenant-Scoped Search and RAG

```python
def tenant_search(client, collection_id: str, query: str, k: int = 5):
    """Search within a specific tenant's collection."""
    return client.retrieval.search(
        query=query,
        search_settings={
            "use_vector_search": True,
            "use_fulltext_search": True,
            "search_limit": k,
            "filters": {
                "collection_ids": [collection_id],
            },
        },
    )


def tenant_rag(client, collection_id: str, query: str):
    """RAG query scoped to a tenant's collection."""
    return client.retrieval.rag(
        query=query,
        rag_generation_config={
            "model": "openai/gpt-4o-mini",
            "temperature": 0.0,
        },
        search_settings={
            "use_vector_search": True,
            "use_fulltext_search": True,
            "search_limit": 5,
            "filters": {
                "collection_ids": [collection_id],
            },
        },
    )


# Acme Corp query -- only sees Acme documents
acme_results = tenant_search(client, org_a_id, "deployment procedures")

# Beta Inc query -- only sees Beta documents
beta_results = tenant_search(client, org_b_id, "deployment procedures")
```

### Access Control Middleware

```python
from fastapi import FastAPI, Depends, HTTPException, Header
from r2r import R2RClient

app = FastAPI()
r2r_client = R2RClient("http://localhost:7272")

# Tenant mapping (in production, use a proper auth service)
TENANT_MAP = {
    "api-key-acme-123": {"tenant": "acme", "collection_id": "uuid-acme"},
    "api-key-beta-456": {"tenant": "beta", "collection_id": "uuid-beta"},
}


def get_tenant(x_api_key: str = Header(...)):
    tenant = TENANT_MAP.get(x_api_key)
    if not tenant:
        raise HTTPException(status_code=401, detail="Invalid API key")
    return tenant


@app.post("/search")
async def search(query: str, tenant: dict = Depends(get_tenant)):
    results = r2r_client.retrieval.search(
        query=query,
        search_settings={
            "use_vector_search": True,
            "use_fulltext_search": True,
            "search_limit": 5,
            "filters": {
                "collection_ids": [tenant["collection_id"]],
            },
        },
    )
    return results


@app.post("/ask")
async def ask(query: str, tenant: dict = Depends(get_tenant)):
    response = r2r_client.retrieval.rag(
        query=query,
        rag_generation_config={"model": "openai/gpt-4o-mini"},
        search_settings={
            "filters": {
                "collection_ids": [tenant["collection_id"]],
            },
        },
    )
    return response
```

---

## Agentic RAG Workflows

### Basic Agent

R2R's agent mode lets the LLM decide when to search and what to search for:

```python
# Single-turn agent query
response = client.retrieval.agent(
    message={
        "role": "user",
        "content": "I need to understand how our authentication system works. "
                   "What are the main components and how do they interact?",
    },
    rag_generation_config={
        "model": "openai/gpt-4o",
        "temperature": 0.0,
        "max_tokens": 2000,
    },
    search_settings={
        "use_vector_search": True,
        "use_fulltext_search": True,
        "use_kg_search": True,
        "search_limit": 10,
    },
)

answer = response["results"]["completion"]["choices"][0]["message"]["content"]
print(answer)
```

### Multi-Turn Agent with Conversation History

```python
class RAGAgent:
    """Multi-turn RAG agent with conversation memory."""

    def __init__(
        self,
        client: R2RClient,
        model: str = "openai/gpt-4o-mini",
        collection_id: str = None,
    ):
        self.client = client
        self.model = model
        self.collection_id = collection_id
        self.conversation_id = None

    def start_conversation(self) -> str:
        """Start a new conversation."""
        result = self.client.conversations.create()
        self.conversation_id = result["results"]["id"]
        return self.conversation_id

    def ask(self, question: str) -> str:
        """Ask a question in the current conversation."""
        if not self.conversation_id:
            self.start_conversation()

        search_settings = {
            "use_vector_search": True,
            "use_fulltext_search": True,
            "search_limit": 5,
        }

        if self.collection_id:
            search_settings["filters"] = {
                "collection_ids": [self.collection_id],
            }

        response = self.client.retrieval.agent(
            message={"role": "user", "content": question},
            conversation_id=self.conversation_id,
            rag_generation_config={
                "model": self.model,
                "temperature": 0.0,
            },
            search_settings=search_settings,
        )

        return response["results"]["completion"]["choices"][0]["message"]["content"]

    def get_history(self) -> list:
        """Retrieve conversation history."""
        if not self.conversation_id:
            return []
        result = self.client.conversations.retrieve(
            conversation_id=self.conversation_id,
        )
        return result["results"]["messages"]


# Usage
agent = RAGAgent(client, model="openai/gpt-4o-mini")
agent.start_conversation()

answer1 = agent.ask("What authentication methods does the system support?")
print(f"A1: {answer1}\n")

answer2 = agent.ask("How is the OAuth2 flow implemented specifically?")
print(f"A2: {answer2}\n")

answer3 = agent.ask("What are the known security considerations for this approach?")
print(f"A3: {answer3}\n")

# Review conversation
history = agent.get_history()
print(f"Conversation has {len(history)} messages")
```

### Research Agent with Iterative Retrieval

```python
import json


class ResearchAgent:
    """Agent that iteratively searches and synthesizes information."""

    def __init__(self, client: R2RClient, model: str = "openai/gpt-4o"):
        self.client = client
        self.model = model

    def research(self, topic: str, max_iterations: int = 3) -> dict:
        """
        Iteratively research a topic:
        1. Generate initial search queries from the topic
        2. Search and collect evidence
        3. Identify knowledge gaps
        4. Generate follow-up queries
        5. Repeat until sufficient or max iterations reached
        """
        all_evidence = []
        queries_used = []

        # Initial queries
        initial_response = self.client.retrieval.rag(
            query=f"Generate 3 specific search queries to research this topic: {topic}. "
                  f"Return only the queries, one per line.",
            rag_generation_config={
                "model": self.model,
                "temperature": 0.3,
            },
            search_settings={"search_limit": 0},  # no retrieval for query generation
        )

        queries = initial_response["results"]["completion"]["choices"][0]["message"]["content"].strip().split("\n")
        queries = [q.strip().strip("0123456789.-) ") for q in queries if q.strip()]

        for iteration in range(max_iterations):
            for query in queries[:3]:
                if query in queries_used:
                    continue
                queries_used.append(query)

                results = self.client.retrieval.search(
                    query=query,
                    search_settings={
                        "use_vector_search": True,
                        "use_fulltext_search": True,
                        "search_limit": 5,
                    },
                )

                for chunk in results["results"]["chunk_search_results"]:
                    all_evidence.append({
                        "query": query,
                        "text": chunk["text"],
                        "score": chunk["score"],
                        "iteration": iteration,
                    })

            # Check if we have enough evidence
            if len(all_evidence) >= 15:
                break

            # Generate follow-up queries based on collected evidence
            evidence_summary = "\n".join(
                e["text"][:200] for e in sorted(
                    all_evidence, key=lambda x: x["score"], reverse=True
                )[:5]
            )

            followup_response = self.client.retrieval.rag(
                query=f"Based on this evidence about '{topic}':\n{evidence_summary}\n\n"
                      f"What questions remain unanswered? Generate 2 follow-up search queries.",
                rag_generation_config={"model": self.model, "temperature": 0.3},
                search_settings={"search_limit": 0},
            )

            queries = followup_response["results"]["completion"]["choices"][0]["message"]["content"].strip().split("\n")
            queries = [q.strip().strip("0123456789.-) ") for q in queries if q.strip()]

        # Final synthesis
        top_evidence = sorted(all_evidence, key=lambda x: x["score"], reverse=True)[:10]
        evidence_text = "\n\n".join(
            f"[{i+1}] {e['text']}" for i, e in enumerate(top_evidence)
        )

        synthesis = self.client.retrieval.rag(
            query=f"Based on the following evidence, write a comprehensive summary about: {topic}\n\n"
                  f"Evidence:\n{evidence_text}",
            rag_generation_config={
                "model": self.model,
                "temperature": 0.0,
                "max_tokens": 2000,
            },
            search_settings={"search_limit": 0},
        )

        return {
            "topic": topic,
            "summary": synthesis["results"]["completion"]["choices"][0]["message"]["content"],
            "evidence_count": len(all_evidence),
            "queries_used": queries_used,
            "iterations": iteration + 1,
        }


# Usage
researcher = ResearchAgent(client, model="openai/gpt-4o")
result = researcher.research("How to optimize vector search performance at scale")
print(result["summary"])
print(f"\nUsed {len(result['queries_used'])} queries over {result['iterations']} iterations")
```

---

## Custom Pipeline Configuration

### Embedding Model Selection

```toml
# r2r.toml -- using local models
[embedding]
provider = "ollama"
model = "nomic-embed-text"
dimension = 768
base_url = "http://localhost:11434"
batch_size = 64

# Or using Voyage AI
[embedding]
provider = "litellm"
model = "voyage/voyage-3"
dimension = 1024
```

```python
# Override embedding at ingestion time
result = client.documents.create(
    file_path="technical-doc.pdf",
    ingestion_config={
        "embedding_config": {
            "model": "text-embedding-3-large",
            "dimension": 1024,   # higher dimension for technical content
        },
    },
)
```

### Custom Chunking Strategies

```python
# Semantic chunking (groups by meaning)
result = client.documents.create(
    file_path="research-paper.pdf",
    ingestion_config={
        "chunking_config": {
            "strategy": "by_title",       # split on headings
            "chunk_size": 1024,
            "chunk_overlap": 100,
            "combine_text_under_n_chars": 200,
            "max_characters": 2000,
        },
    },
)

# Small chunks for precise retrieval
result = client.documents.create(
    file_path="faq.md",
    ingestion_config={
        "chunking_config": {
            "strategy": "recursive",
            "chunk_size": 256,
            "chunk_overlap": 25,
        },
    },
)
```

### Custom RAG Prompts

```python
# Custom system prompt for RAG generation
response = client.retrieval.rag(
    query="What are the security best practices?",
    rag_generation_config={
        "model": "openai/gpt-4o-mini",
        "temperature": 0.0,
        "system_prompt": """You are a security expert assistant. Answer questions using ONLY
the provided context documents. Follow these rules:
1. Cite specific document passages using [Source N] notation
2. If the context lacks information to answer, explicitly state what is missing
3. Flag any security recommendations that may be outdated
4. Prioritize defensive security recommendations""",
    },
)
```

---

## Evaluation

### Retrieval Quality Assessment

```python
def evaluate_retrieval(
    client: R2RClient,
    test_set: list[dict],  # [{"query": "...", "relevant_doc_ids": ["..."]}]
    search_settings: dict = None,
) -> dict:
    """Evaluate retrieval quality on a test set."""
    if search_settings is None:
        search_settings = {
            "use_vector_search": True,
            "use_fulltext_search": True,
            "search_limit": 10,
        }

    metrics = {"recall@5": [], "recall@10": [], "mrr": []}

    for test_case in test_set:
        results = client.retrieval.search(
            query=test_case["query"],
            search_settings=search_settings,
        )

        retrieved_ids = [
            r["document_id"]
            for r in results["results"]["chunk_search_results"]
        ]
        relevant_ids = set(test_case["relevant_doc_ids"])

        # Recall@K
        for k in [5, 10]:
            found = len(relevant_ids & set(retrieved_ids[:k]))
            metrics[f"recall@{k}"].append(found / len(relevant_ids))

        # MRR
        rr = 0.0
        for rank, doc_id in enumerate(retrieved_ids, 1):
            if doc_id in relevant_ids:
                rr = 1.0 / rank
                break
        metrics["mrr"].append(rr)

    # Average metrics
    return {k: sum(v) / len(v) for k, v in metrics.items()}


# Run evaluation
test_set = [
    {"query": "authentication flow", "relevant_doc_ids": ["doc-001", "doc-002"]},
    {"query": "database schema design", "relevant_doc_ids": ["doc-010"]},
]

# Compare vector-only vs hybrid
vector_metrics = evaluate_retrieval(client, test_set, {
    "use_vector_search": True,
    "use_fulltext_search": False,
    "search_limit": 10,
})

hybrid_metrics = evaluate_retrieval(client, test_set, {
    "use_vector_search": True,
    "use_fulltext_search": True,
    "search_limit": 10,
})

print(f"Vector-only: {vector_metrics}")
print(f"Hybrid:      {hybrid_metrics}")
```

### RAG Answer Quality

```python
import openai


def evaluate_rag_answer(
    question: str,
    answer: str,
    reference_answer: str,
    context: list[str],
) -> dict:
    """Evaluate RAG answer quality using LLM-as-judge."""
    oai = openai.OpenAI()

    # Faithfulness: is the answer grounded in the context?
    faithfulness_prompt = f"""Rate the faithfulness of this answer on a scale of 1-5.
A faithful answer only contains information that can be verified from the context.

Context: {' '.join(context[:3])}
Answer: {answer}

Score (1-5):"""

    faith_result = oai.chat.completions.create(
        model="gpt-4o-mini",
        messages=[{"role": "user", "content": faithfulness_prompt}],
        temperature=0,
        max_tokens=10,
    )
    faithfulness = int(faith_result.choices[0].message.content.strip()[0])

    # Correctness: does the answer match the reference?
    correctness_prompt = f"""Rate how well the predicted answer matches the reference on a scale of 1-5.

Question: {question}
Reference answer: {reference_answer}
Predicted answer: {answer}

Score (1-5):"""

    correct_result = oai.chat.completions.create(
        model="gpt-4o-mini",
        messages=[{"role": "user", "content": correctness_prompt}],
        temperature=0,
        max_tokens=10,
    )
    correctness = int(correct_result.choices[0].message.content.strip()[0])

    return {
        "faithfulness": faithfulness / 5.0,
        "correctness": correctness / 5.0,
    }
```

---

## Performance Optimization

### Embedding Batch Ingestion

```python
import time
from pathlib import Path


def batch_ingest(
    client: R2RClient,
    files: list[str],
    batch_size: int = 10,
    delay: float = 1.0,
) -> dict:
    """Ingest files in batches with rate limiting."""
    results = {"success": 0, "failed": 0, "errors": []}

    for i in range(0, len(files), batch_size):
        batch = files[i : i + batch_size]

        for file_path in batch:
            try:
                client.documents.create(
                    file_path=file_path,
                    run_with_orchestration=True,
                )
                results["success"] += 1
            except Exception as e:
                results["failed"] += 1
                results["errors"].append({"file": file_path, "error": str(e)})

        print(f"Progress: {min(i + batch_size, len(files))}/{len(files)}")
        time.sleep(delay)

    return results


files = [str(p) for p in Path("./docs").glob("**/*.md")]
result = batch_ingest(client, files, batch_size=10, delay=0.5)
print(f"Ingested: {result['success']}, Failed: {result['failed']}")
```

### Search Performance Tuning

```python
# Optimize search for speed
fast_search_settings = {
    "use_vector_search": True,
    "use_fulltext_search": False,    # disable BM25 for speed
    "use_kg_search": False,          # disable KG for speed
    "search_limit": 5,               # fewer results
    "ef_search": 50,                 # lower HNSW ef for speed
}

# Optimize search for quality
quality_search_settings = {
    "use_vector_search": True,
    "use_fulltext_search": True,
    "use_kg_search": True,
    "search_limit": 20,
    "ef_search": 200,                # higher HNSW ef for quality
    "hybrid_settings": {
        "full_text_weight": 1.0,
        "semantic_weight": 5.0,
        "rrf_k": 60,
    },
}
```

---

## Common Pitfalls

1. **Not scoping searches in multi-tenant deployments**: Without collection_id filters, a tenant's query returns results from all tenants. This is a data isolation violation.

2. **Agent mode with wrong model**: Agentic workflows need models that support tool use well. Use GPT-4o or Claude Sonnet/Opus for agents, not small models.

3. **Overloading with synchronous ingestion**: For large batches, always use `run_with_orchestration=True`. Synchronous ingestion of large PDFs can timeout.

4. **Not monitoring Hatchet jobs**: Async ingestion and KG construction run in Hatchet. Failed jobs are silent unless you check the Hatchet dashboard or API.

5. **KG extraction on noisy documents**: Auto-KG extracts entities from raw text. Poorly formatted documents (OCR artifacts, tables as text) produce noisy KG triples. Clean documents before ingestion.

6. **Ignoring hybrid search weights**: Default weights may not suit your domain. Experiment with `full_text_weight` and `semantic_weight` ratios. Technical queries often benefit from higher BM25 weight.

---

## References

- R2R documentation: https://r2r-docs.sciphi.ai/
- R2R GitHub: https://github.com/SciPhi-AI/R2R
- Multi-tenant guide: https://r2r-docs.sciphi.ai/cookbooks/multi-tenant
- Agent mode: https://r2r-docs.sciphi.ai/cookbooks/agent
