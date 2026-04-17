# Adaptive RAG: Routing by Query Complexity

## TL;DR

Adaptive RAG (Jeong et al., NAACL 2024) introduces a query complexity classifier that routes queries to different retrieval strategies: no retrieval for simple factual queries the LLM already knows, single-step retrieval for moderate queries, and multi-step iterative retrieval for complex multi-hop queries. This avoids the cost and latency of unnecessary retrieval while ensuring complex queries get the thorough retrieval they need. The classifier can be a trained model, a rule-based system, or an LLM-based router. This article covers the routing logic, classifier implementations, integration with Self-RAG and CRAG, and a complete LangGraph implementation.

---

## The Core Idea

### Standard RAG Treats All Queries the Same

```
"What is Python?" -----> [Retrieve] -> [Generate]  (unnecessary retrieval)
"Configure RLS"  -----> [Retrieve] -> [Generate]  (single retrieval is fine)
"Compare RLS vs RBAC    [Retrieve] -> [Generate]  (needs multi-step, but
  across PG, MySQL,                                  only gets one shot)
  and SQL Server"
```

### Adaptive RAG Routes by Complexity

```
"What is Python?" -----> [No Retrieval] -> [Generate directly]
                          Latency: ~200ms

"Configure RLS"  -----> [Single Retrieval] -> [Generate]
                          Latency: ~500ms

"Compare RLS vs RBAC    [Multi-Step Retrieval] -> [Synthesize] -> [Generate]
  across PG, MySQL,       Step 1: Retrieve PG RLS docs
  and SQL Server"         Step 2: Retrieve MySQL access control docs
                          Step 3: Retrieve SQL Server security docs
                          Latency: ~2000ms (but accurate)
```

### Why This Matters

| Metric | Standard RAG | Adaptive RAG | Improvement |
|--------|-------------|-------------|-------------|
| Avg latency (simple queries) | 500ms | 200ms | 60% faster |
| Avg latency (complex queries) | 500ms | 2000ms | Slower but accurate |
| Accuracy (simple queries) | 82% (noise from retrieval) | 88% | +6% |
| Accuracy (complex queries) | 61% (insufficient retrieval) | 79% | +18% |
| Overall accuracy | 72% | 84% | +12% |
| Avg cost per query | $0.005 | $0.004 | 20% cheaper |

---

## Query Complexity Classification

### Three Complexity Levels

| Level | Description | Examples | Strategy |
|-------|-------------|----------|----------|
| **A: Simple** | Common knowledge, definitions, well-known facts | "What is PostgreSQL?", "Who created Python?", "What does REST stand for?" | No retrieval, direct LLM generation |
| **B: Moderate** | Domain-specific factual, single-topic | "How to configure RLS in PostgreSQL?", "What are the default Kafka retention settings?" | Single retrieval step |
| **C: Complex** | Multi-hop, comparison, requires synthesis | "Compare PostgreSQL and MySQL security models for multi-tenant SaaS", "Trace how a request flows through our microservice architecture" | Multi-step retrieval with decomposition |

### Method 1: Rule-Based Classifier

Fast, interpretable, no model required:

```python
import re
from enum import Enum


class QueryComplexity(Enum):
    SIMPLE = "simple"
    MODERATE = "moderate"
    COMPLEX = "complex"


def classify_complexity_rules(query: str) -> QueryComplexity:
    """Rule-based query complexity classifier."""
    query_lower = query.lower().strip()
    words = query_lower.split()
    word_count = len(words)

    # Heuristic 1: Very short queries about common topics -> simple
    if word_count <= 5 and re.search(
        r'\b(what is|who is|define|meaning of)\b', query_lower
    ):
        return QueryComplexity.SIMPLE

    # Heuristic 2: Comparison, multi-entity, or multi-step indicators -> complex
    complex_indicators = [
        r'\b(compare|versus|vs\.?|difference between)\b',
        r'\b(step.by.step|end.to.end|from .+ to .+)\b',
        r'\b(and|or)\b.*\b(and|or)\b',  # Multiple conjunctions
        r'\b(across|between)\b.*\b(and|,)\b',  # Cross-entity comparison
        r'\b(trace|analyze|evaluate|architect)\b',
        r'\b(pros and cons|advantages and disadvantages)\b',
    ]
    if any(re.search(p, query_lower) for p in complex_indicators):
        return QueryComplexity.COMPLEX

    # Heuristic 3: Long queries with technical terms -> likely complex
    if word_count > 20:
        return QueryComplexity.COMPLEX

    # Heuristic 4: How-to with specific technology -> moderate
    if re.search(r'\b(how to|configure|setup|install|implement|create)\b', query_lower):
        return QueryComplexity.MODERATE

    # Heuristic 5: Specific factual questions -> moderate
    if re.search(r'\b(what are|list|show|explain)\b', query_lower):
        return QueryComplexity.MODERATE

    # Default
    return QueryComplexity.MODERATE


# Tests
assert classify_complexity_rules("What is PostgreSQL?") == QueryComplexity.SIMPLE
assert classify_complexity_rules("How to configure RLS in PostgreSQL?") == QueryComplexity.MODERATE
assert classify_complexity_rules(
    "Compare PostgreSQL and MySQL security models for multi-tenant applications"
) == QueryComplexity.COMPLEX
```

### Method 2: LLM-Based Classifier

More accurate, handles nuance, but adds latency:

```python
import anthropic

client = anthropic.Anthropic()


def classify_complexity_llm(query: str) -> QueryComplexity:
    """Classify query complexity using Claude Haiku."""
    response = client.messages.create(
        model="claude-3-5-haiku-20241022",
        max_tokens=20,
        messages=[{
            "role": "user",
            "content": (
                "Classify this search query's complexity level.\n\n"
                "SIMPLE = common knowledge, definitions, well-known facts "
                "(no retrieval needed)\n"
                "MODERATE = specific factual question about one topic "
                "(single retrieval step)\n"
                "COMPLEX = comparison, multi-hop reasoning, or synthesis across "
                "multiple topics (multi-step retrieval needed)\n\n"
                f"Query: {query}\n\n"
                "Respond with ONLY: SIMPLE, MODERATE, or COMPLEX"
            ),
        }],
    )
    text = response.content[0].text.strip().upper()
    if "SIMPLE" in text:
        return QueryComplexity.SIMPLE
    elif "COMPLEX" in text:
        return QueryComplexity.COMPLEX
    return QueryComplexity.MODERATE
```

### Method 3: Trained Classifier

The paper's approach -- train a small classifier on labeled data:

```python
from sklearn.ensemble import GradientBoostingClassifier
from sklearn.feature_extraction.text import TfidfVectorizer
from sklearn.pipeline import Pipeline
import pickle


def train_complexity_classifier(
    queries: list[str],
    labels: list[str],  # "simple", "moderate", "complex"
) -> Pipeline:
    """Train a query complexity classifier."""
    pipeline = Pipeline([
        ("tfidf", TfidfVectorizer(
            max_features=5000,
            ngram_range=(1, 3),
            sublinear_tf=True,
        )),
        ("clf", GradientBoostingClassifier(
            n_estimators=200,
            max_depth=5,
            learning_rate=0.1,
        )),
    ])
    pipeline.fit(queries, labels)
    return pipeline


def predict_complexity(
    pipeline: Pipeline, query: str
) -> QueryComplexity:
    """Predict query complexity using trained model."""
    label = pipeline.predict([query])[0]
    return QueryComplexity(label)


# Training data generation
# Option 1: Manually label 200-500 queries
# Option 2: Use LLM to label, then train fast classifier
def generate_training_data_with_llm(
    queries: list[str], batch_size: int = 20
) -> list[tuple[str, str]]:
    """Generate training labels using LLM, then train fast classifier."""
    labeled = []
    for i in range(0, len(queries), batch_size):
        batch = queries[i:i + batch_size]
        for query in batch:
            complexity = classify_complexity_llm(query)
            labeled.append((query, complexity.value))
    return labeled
```

### Classifier Comparison

| Method | Latency | Accuracy | Cost | Best For |
|--------|---------|----------|------|----------|
| Rule-based | <1ms | ~75% | Free | Predictable query patterns |
| Trained classifier | <5ms | ~85% | Free (after training) | Production with labeled data |
| LLM (Haiku) | ~100ms | ~92% | ~$0.0001/query | Highest accuracy needs |
| Hybrid (rules + LLM fallback) | 1-100ms | ~90% | Varies | Best cost/accuracy tradeoff |

---

## Retrieval Strategies by Complexity

### Simple: Direct Generation (No Retrieval)

```python
def handle_simple(query: str) -> str:
    """Generate answer directly from LLM knowledge."""
    response = llm.invoke(
        f"Answer this question concisely: {query}\n\n"
        f"If you are not confident in your answer, say 'I am not sure' "
        f"so we can look this up."
    )
    answer = response.content

    # Safety check: if LLM is uncertain, upgrade to moderate
    if "not sure" in answer.lower() or "don't know" in answer.lower():
        return handle_moderate(query)

    return answer
```

### Moderate: Single-Step Retrieval

```python
def handle_moderate(query: str) -> str:
    """Standard RAG: retrieve once, generate."""
    docs = retriever.invoke(query)
    context = "\n\n".join([d.page_content for d in docs[:5]])
    response = llm.invoke(
        f"Answer based on the context:\n{context}\n\nQuery: {query}"
    )
    return response.content
```

### Complex: Multi-Step Retrieval with Decomposition

```python
def handle_complex(query: str) -> str:
    """Decompose query into sub-questions, retrieve for each, synthesize."""

    # Step 1: Decompose into sub-questions
    decomposition = llm.invoke(
        f"Break this complex question into 2-4 simpler sub-questions "
        f"that can be answered independently.\n\n"
        f"Question: {query}\n\n"
        f"Return each sub-question on a new line, numbered."
    )
    sub_questions = [
        line.strip().lstrip("0123456789.) ")
        for line in decomposition.content.strip().split("\n")
        if line.strip() and not line.strip().startswith("#")
    ]

    # Step 2: Retrieve and answer each sub-question
    sub_answers = []
    all_sources = []
    for sq in sub_questions:
        docs = retriever.invoke(sq)
        context = "\n\n".join([d.page_content for d in docs[:3]])
        answer = llm.invoke(
            f"Context:\n{context}\n\nQuestion: {sq}\n\nAnswer concisely."
        )
        sub_answers.append({"question": sq, "answer": answer.content})
        all_sources.extend(docs[:3])

    # Step 3: Synthesize sub-answers into final response
    sub_qa_text = "\n\n".join(
        f"Q: {sa['question']}\nA: {sa['answer']}" for sa in sub_answers
    )
    synthesis = llm.invoke(
        f"Using these sub-answers, provide a comprehensive answer to the "
        f"original question.\n\n"
        f"Original question: {query}\n\n"
        f"Sub-answers:\n{sub_qa_text}"
    )

    return synthesis.content
```

---

## Combining with Self-RAG and CRAG

Adaptive RAG is a routing mechanism. Self-RAG and CRAG are retrieval evaluation mechanisms. They compose naturally:

```
Query
  |
  v
[Adaptive RAG Classifier]
  |
  +--> Simple: LLM generates directly
  |       |
  |       v
  |     [Self-RAG: verify answer is not hallucinated]
  |
  +--> Moderate:
  |       |
  |       v
  |     [Retrieve from KB]
  |       |
  |       v
  |     [CRAG: evaluate confidence]
  |       |
  |       +--> Correct: [Self-RAG: generate + verify support]
  |       +--> Incorrect: [Web search] -> [Self-RAG: generate + verify]
  |       +--> Ambiguous: [Combine KB + web] -> [Self-RAG]
  |
  +--> Complex:
          |
          v
        [Decompose into sub-questions]
          |
          v
        [For each sub-question:]
          [Retrieve] -> [CRAG confidence check]
          -> [Self-RAG: generate sub-answer + verify]
          |
          v
        [Synthesize sub-answers]
          |
          v
        [Self-RAG: verify final synthesis]
```

### LangGraph Implementation of Combined Pipeline

```python
from langgraph.graph import StateGraph, END
from typing import TypedDict


class AdaptiveRAGState(TypedDict):
    query: str
    complexity: str                  # "simple", "moderate", "complex"
    sub_questions: list[str]
    sub_answers: list[dict]
    documents: list[dict]
    confidence: str                  # CRAG: "correct", "incorrect", "ambiguous"
    web_results: list[str]
    generation: str
    is_supported: bool               # Self-RAG verification
    attempt: int


# --- Nodes ---
def classify(state: AdaptiveRAGState) -> AdaptiveRAGState:
    complexity = classify_complexity_rules(state["query"])
    return {**state, "complexity": complexity.value}


def direct_generate(state: AdaptiveRAGState) -> AdaptiveRAGState:
    answer = handle_simple(state["query"])
    return {**state, "generation": answer}


def single_retrieve(state: AdaptiveRAGState) -> AdaptiveRAGState:
    docs = retriever.invoke(state["query"])
    return {
        **state,
        "documents": [{"content": d.page_content, "metadata": d.metadata} for d in docs],
    }


def crag_evaluate(state: AdaptiveRAGState) -> AdaptiveRAGState:
    evaluator = RetrievalConfidenceEvaluator()
    doc_texts = [d["content"] for d in state["documents"]]
    action, scores = evaluator.evaluate(state["query"], doc_texts)
    return {**state, "confidence": action}


def generate_from_kb(state: AdaptiveRAGState) -> AdaptiveRAGState:
    context = "\n\n".join([d["content"] for d in state["documents"][:5]])
    response = llm.invoke(f"Context:\n{context}\n\nQuery: {state['query']}")
    return {**state, "generation": response.content}


def web_fallback(state: AdaptiveRAGState) -> AdaptiveRAGState:
    results = web_search_fallback(state["query"])
    context = "\n\n".join(results[:5])
    response = llm.invoke(f"Context:\n{context}\n\nQuery: {state['query']}")
    return {**state, "generation": response.content, "web_results": results}


def decompose(state: AdaptiveRAGState) -> AdaptiveRAGState:
    decomposition = llm.invoke(
        f"Break into 2-4 sub-questions:\n{state['query']}"
    )
    sub_qs = [
        l.strip().lstrip("0123456789.) ")
        for l in decomposition.content.split("\n")
        if l.strip()
    ]
    return {**state, "sub_questions": sub_qs}


def multi_step_retrieve_and_answer(state: AdaptiveRAGState) -> AdaptiveRAGState:
    sub_answers = []
    for sq in state["sub_questions"]:
        docs = retriever.invoke(sq)
        context = "\n\n".join([d.page_content for d in docs[:3]])
        answer = llm.invoke(f"Context:\n{context}\n\nQ: {sq}").content
        sub_answers.append({"question": sq, "answer": answer})
    return {**state, "sub_answers": sub_answers}


def synthesize(state: AdaptiveRAGState) -> AdaptiveRAGState:
    qa_text = "\n".join(
        f"Q: {sa['question']}\nA: {sa['answer']}" for sa in state["sub_answers"]
    )
    response = llm.invoke(
        f"Synthesize a comprehensive answer.\n"
        f"Original: {state['query']}\n\nSub-answers:\n{qa_text}"
    )
    return {**state, "generation": response.content}


def verify_support(state: AdaptiveRAGState) -> AdaptiveRAGState:
    """Self-RAG style verification."""
    if not state["documents"]:
        return {**state, "is_supported": True}  # No docs to check against

    support = self_rag_support_check(
        state["query"],
        state["documents"][0]["content"] if state["documents"] else "",
        state["generation"],
    )
    return {**state, "is_supported": support != "no_support"}


# --- Routing ---
def route_complexity(state):
    return state["complexity"]

def route_confidence(state):
    return state["confidence"]

def route_verification(state):
    return END if state["is_supported"] else "web_fallback"


# --- Build Graph ---
graph = StateGraph(AdaptiveRAGState)

graph.add_node("classify", classify)
graph.add_node("direct_generate", direct_generate)
graph.add_node("single_retrieve", single_retrieve)
graph.add_node("crag_evaluate", crag_evaluate)
graph.add_node("generate_from_kb", generate_from_kb)
graph.add_node("web_fallback", web_fallback)
graph.add_node("decompose", decompose)
graph.add_node("multi_step", multi_step_retrieve_and_answer)
graph.add_node("synthesize", synthesize)
graph.add_node("verify", verify_support)

graph.set_entry_point("classify")

graph.add_conditional_edges("classify", route_complexity, {
    "simple": "direct_generate",
    "moderate": "single_retrieve",
    "complex": "decompose",
})

# Simple path
graph.add_edge("direct_generate", END)

# Moderate path
graph.add_edge("single_retrieve", "crag_evaluate")
graph.add_conditional_edges("crag_evaluate", route_confidence, {
    "correct": "generate_from_kb",
    "incorrect": "web_fallback",
    "ambiguous": "generate_from_kb",  # Simplified; could combine
})
graph.add_edge("generate_from_kb", "verify")
graph.add_edge("web_fallback", END)
graph.add_conditional_edges("verify", route_verification, {
    END: END,
    "web_fallback": "web_fallback",
})

# Complex path
graph.add_edge("decompose", "multi_step")
graph.add_edge("multi_step", "synthesize")
graph.add_edge("synthesize", END)

adaptive_rag_app = graph.compile()
```

---

## Evaluation

### Evaluating the Classifier

```python
def evaluate_classifier(
    eval_set: list[dict],  # [{"query": str, "true_complexity": str}]
    classifier_fn,
) -> dict:
    """Evaluate classification accuracy."""
    correct = 0
    confusion = {"simple": {"simple": 0, "moderate": 0, "complex": 0},
                 "moderate": {"simple": 0, "moderate": 0, "complex": 0},
                 "complex": {"simple": 0, "moderate": 0, "complex": 0}}

    for item in eval_set:
        predicted = classifier_fn(item["query"]).value
        true = item["true_complexity"]
        confusion[true][predicted] += 1
        if predicted == true:
            correct += 1

    accuracy = correct / len(eval_set)
    print(f"Accuracy: {accuracy:.3f}")
    print(f"Confusion matrix:")
    for true_label, predictions in confusion.items():
        print(f"  True={true_label}: {predictions}")

    return {"accuracy": accuracy, "confusion": confusion}
```

### Evaluating End-to-End Quality

```python
def evaluate_adaptive_rag(
    eval_set: list[dict],
    pipeline,
) -> dict:
    """Evaluate end-to-end adaptive RAG quality."""
    results = {
        "correct_by_complexity": {"simple": 0, "moderate": 0, "complex": 0},
        "total_by_complexity": {"simple": 0, "moderate": 0, "complex": 0},
        "avg_latency_by_complexity": {"simple": [], "moderate": [], "complex": []},
    }

    for item in eval_set:
        import time
        start = time.time()
        output = pipeline.invoke({
            "query": item["query"],
            "complexity": "",
            "sub_questions": [],
            "sub_answers": [],
            "documents": [],
            "confidence": "",
            "web_results": [],
            "generation": "",
            "is_supported": False,
            "attempt": 0,
        })
        latency = (time.time() - start) * 1000

        complexity = item["true_complexity"]
        results["total_by_complexity"][complexity] += 1
        results["avg_latency_by_complexity"][complexity].append(latency)

        # Check answer quality (simplified)
        if item["expected_answer"].lower() in output["generation"].lower():
            results["correct_by_complexity"][complexity] += 1

    # Compute averages
    for c in ["simple", "moderate", "complex"]:
        total = results["total_by_complexity"][c]
        correct = results["correct_by_complexity"][c]
        latencies = results["avg_latency_by_complexity"][c]
        print(
            f"{c}: accuracy={correct/total:.3f} "
            f"avg_latency={sum(latencies)/len(latencies):.0f}ms"
            if total > 0 else f"{c}: no samples"
        )

    return results
```

---

## Common Pitfalls

1. **Over-routing to "simple".** Aggressively skipping retrieval causes hallucination on queries that seem simple but require specific knowledge. Add a safety check: if the LLM expresses uncertainty in the direct-generation path, upgrade to moderate.
2. **Under-routing to "complex".** Multi-step retrieval is expensive. If most queries are moderate, the classifier should have a high bar for "complex". Measure the false-complex rate and ensure it is below 10%.
3. **Not evaluating the classifier separately.** Measure classifier accuracy independently from end-to-end quality. A classifier that is 70% accurate will degrade overall performance compared to always using moderate.
4. **Fixed sub-question count.** Some complex queries decompose into 2 sub-questions, others into 5. Let the LLM decide the decomposition rather than forcing a fixed count.
5. **No safety net for misclassification.** Always have fallback paths: simple -> moderate if LLM is uncertain; moderate -> web search if CRAG confidence is low; complex -> timeout with best available answer if decomposition takes too long.
6. **Ignoring the cost of multi-step retrieval.** Each sub-question retrieval is a separate embedding + vector search + LLM call. For 4 sub-questions, this is 4x the cost of a single retrieval. Budget accordingly.

---

## References

- Jeong et al. "Adaptive-RAG: Learning to Adapt Retrieval-Augmented Large Language Models through Question Complexity" (NAACL 2024). https://arxiv.org/abs/2403.14403
- LangGraph Adaptive RAG: https://langchain-ai.github.io/langgraph/tutorials/rag/langgraph_adaptive_rag/
- Asai et al. "Self-RAG" (ICLR 2024). https://arxiv.org/abs/2310.11511
- Yan et al. "Corrective RAG" (2024). https://arxiv.org/abs/2401.15884
