# Self-RAG: Self-Reflective Retrieval-Augmented Generation

## TL;DR

Self-RAG (Asai et al., ICLR 2024) trains an LLM to emit special "reflection tokens" that control the retrieval and generation process. Four token types -- Retrieve, IsRel, IsSup, IsUse -- allow the model to decide when to retrieve, whether retrieved passages are relevant, whether the generation is supported by the passages, and whether the generation is useful. Unlike standard RAG (always retrieve) or standard LLMs (never retrieve), Self-RAG dynamically decides on a per-query, per-segment basis. The approach achieves state-of-the-art results on knowledge-intensive tasks while maintaining citation accuracy.

---

## The Self-Reflection Token Framework

### Four Special Tokens

Self-RAG introduces four types of reflection tokens, each serving a distinct purpose in the generation process:

| Token | Values | When Emitted | Purpose |
|-------|--------|-------------|---------|
| `[Retrieve]` | `Yes`, `No`, `Continue` | Before generating each segment | Should the model retrieve passages? |
| `[IsRel]` | `Relevant`, `Irrelevant` | After retrieval, for each passage | Is this passage relevant to the query? |
| `[IsSup]` | `Fully Supported`, `Partially Supported`, `No Support` | After generation with a passage | Is the generation grounded in the passage? |
| `[IsUse]` | `5`, `4`, `3`, `2`, `1` | After generation | How useful is the overall response? (5 = best) |

### Inference Flow

```
Step 1: Query arrives
  "What are the health benefits of green tea?"

Step 2: Model decides whether to retrieve
  Output: [Retrieve] = Yes
  (Model determines it needs external knowledge)

Step 3: Retrieval system fetches top-K passages
  Passage 1: "Green tea contains catechins, powerful antioxidants..."
  Passage 2: "Black tea production involves full oxidation..."
  Passage 3: "Studies show green tea may reduce cardiovascular risk..."

Step 4: Model evaluates each passage
  Passage 1: [IsRel] = Relevant
  Passage 2: [IsRel] = Irrelevant  (discarded)
  Passage 3: [IsRel] = Relevant

Step 5: Model generates with relevant passages
  Generation: "Green tea offers several health benefits, including
  antioxidant properties from catechins and potential cardiovascular
  risk reduction."

Step 6: Model evaluates support
  [IsSup] = Fully Supported (claims match passages)

Step 7: Model evaluates utility
  [IsUse] = 5 (response is comprehensive and accurate)
```

### Multi-Segment Generation

For longer responses, the process repeats per segment:

```
Query: "Compare the environmental impact of electric vs. gas vehicles"

Segment 1:
  [Retrieve] = Yes
  -> Retrieve passages about EV environmental impact
  -> [IsRel] filtering
  -> Generate: "Electric vehicles produce zero tailpipe emissions..."
  -> [IsSup] = Fully Supported

Segment 2:
  [Retrieve] = Yes
  -> Retrieve passages about gas vehicle emissions
  -> [IsRel] filtering
  -> Generate: "Gasoline vehicles emit an average of 4.6 metric tons..."
  -> [IsSup] = Fully Supported

Segment 3:
  [Retrieve] = No  (synthesis does not need new facts)
  -> Generate: "Overall, EVs have a lower lifetime carbon footprint..."
  -> [IsSup] = Partially Supported (synthesis goes beyond individual passages)
  -> [IsUse] = 4
```

---

## Training Methodology

### Overview

Self-RAG training has two phases:

1. **Critic model training**: train a separate model to generate reflection tokens
2. **Generator training**: train the main LLM to both generate text AND emit reflection tokens

### Phase 1: Critic Model

The critic model is trained to classify (query, passage, generation) tuples:

```python
# Critic training data format
critic_examples = [
    {
        "query": "What causes earthquakes?",
        "passage": "Earthquakes occur when tectonic plates shift...",
        "generation": "Earthquakes are caused by tectonic plate movement.",
        "labels": {
            "IsRel": "Relevant",
            "IsSup": "Fully Supported",
            "IsUse": 5,
        }
    },
    {
        "query": "What causes earthquakes?",
        "passage": "The weather forecast for Tuesday shows...",
        "generation": "Earthquakes happen due to weather patterns.",
        "labels": {
            "IsRel": "Irrelevant",
            "IsSup": "No Support",
            "IsUse": 1,
        }
    },
]
```

The critic is typically a smaller model (7B) fine-tuned on ~50K labeled examples. GPT-4 is used to generate the initial labels.

### Phase 2: Generator Training

The generator is trained on text that includes reflection tokens inline:

```
Training input:
"[Retrieve] Yes [IsRel] Relevant Green tea contains catechins,
powerful antioxidants that protect cells. [IsSup] Fully Supported
[IsUse] 5"
```

The generator learns to:
1. Predict when to emit `[Retrieve]` tokens
2. Predict relevance and support labels for retrieved passages
3. Generate text conditioned on these assessments

Training data is constructed by:
1. Running the critic model on (query, passage, generation) triples
2. Inserting the critic's labels as inline tokens
3. Training the generator to reproduce both text and tokens

### Training Data Pipeline

```python
# Pseudocode for training data construction
def create_training_data(queries, corpus, retriever, critic_model, generator_model):
    training_examples = []

    for query in queries:
        # Retrieve passages
        passages = retriever.retrieve(query, k=10)

        # Generate without retrieval (for [Retrieve] = No examples)
        no_retrieve_output = generator_model.generate(query)

        # Generate with each passage
        for passage in passages:
            generation = generator_model.generate(query, context=passage)

            # Critic evaluates
            is_rel = critic_model.predict_relevance(query, passage)
            is_sup = critic_model.predict_support(query, passage, generation)
            is_use = critic_model.predict_utility(query, generation)

            # Create training example with inline tokens
            training_examples.append(
                format_with_reflection_tokens(
                    query, passage, generation,
                    retrieve=True, is_rel=is_rel, is_sup=is_sup, is_use=is_use
                )
            )

        # Also create "no retrieval needed" examples
        is_use = critic_model.predict_utility(query, no_retrieve_output)
        training_examples.append(
            format_with_reflection_tokens(
                query, None, no_retrieve_output,
                retrieve=False, is_rel=None, is_sup=None, is_use=is_use
            )
        )

    return training_examples
```

---

## Implementation Sketch

### Minimal Self-RAG with an Existing LLM

You can approximate Self-RAG's behavior without training a custom model by using prompt engineering with a strong LLM:

```python
import anthropic
from typing import Literal

client = anthropic.Anthropic()


def self_rag_retrieve_decision(query: str) -> bool:
    """Decide whether retrieval is needed (approximates [Retrieve] token)."""
    response = client.messages.create(
        model="claude-3-5-sonnet-20241022",
        max_tokens=10,
        messages=[{
            "role": "user",
            "content": (
                f"Does answering this query require external knowledge retrieval, "
                f"or can it be answered from general knowledge?\n"
                f"Query: {query}\n"
                f"Respond with ONLY 'retrieve' or 'no_retrieve'."
            ),
        }],
    )
    return "retrieve" in response.content[0].text.lower()


def self_rag_relevance_check(
    query: str, passage: str
) -> Literal["relevant", "irrelevant"]:
    """Evaluate passage relevance (approximates [IsRel] token)."""
    response = client.messages.create(
        model="claude-3-5-haiku-20241022",
        max_tokens=10,
        messages=[{
            "role": "user",
            "content": (
                f"Is this passage relevant to answering the query?\n"
                f"Query: {query}\n"
                f"Passage: {passage[:500]}\n"
                f"Respond with ONLY 'relevant' or 'irrelevant'."
            ),
        }],
    )
    return "relevant" if "relevant" in response.content[0].text.lower() else "irrelevant"


def self_rag_support_check(
    query: str, passage: str, generation: str
) -> Literal["fully_supported", "partially_supported", "no_support"]:
    """Evaluate answer grounding (approximates [IsSup] token)."""
    response = client.messages.create(
        model="claude-3-5-haiku-20241022",
        max_tokens=20,
        messages=[{
            "role": "user",
            "content": (
                f"Is this answer fully supported by the passage?\n"
                f"Passage: {passage[:500]}\n"
                f"Answer: {generation}\n"
                f"Respond with ONLY: 'fully_supported', 'partially_supported', or 'no_support'."
            ),
        }],
    )
    text = response.content[0].text.lower()
    if "fully" in text:
        return "fully_supported"
    elif "partially" in text:
        return "partially_supported"
    return "no_support"


def self_rag_pipeline(query: str, retriever, max_retries: int = 2) -> dict:
    """Full Self-RAG pipeline using prompt-based reflection."""

    # Step 1: Retrieve decision
    needs_retrieval = self_rag_retrieve_decision(query)

    if not needs_retrieval:
        # Direct generation
        response = client.messages.create(
            model="claude-3-5-sonnet-20241022",
            max_tokens=1000,
            messages=[{"role": "user", "content": query}],
        )
        return {
            "answer": response.content[0].text,
            "retrieved": False,
            "sources": [],
        }

    # Step 2: Retrieve and filter
    for attempt in range(max_retries):
        passages = retriever.invoke(query)

        relevant_passages = []
        for passage in passages:
            relevance = self_rag_relevance_check(query, passage.page_content)
            if relevance == "relevant":
                relevant_passages.append(passage)

        if not relevant_passages:
            # Rewrite query and try again
            query = rewrite_query(query)
            continue

        # Step 3: Generate with relevant passages
        context = "\n\n".join([p.page_content for p in relevant_passages[:5]])
        response = client.messages.create(
            model="claude-3-5-sonnet-20241022",
            max_tokens=1000,
            messages=[{
                "role": "user",
                "content": (
                    f"Answer the query using ONLY the provided context.\n\n"
                    f"Context:\n{context}\n\nQuery: {query}"
                ),
            }],
        )
        generation = response.content[0].text

        # Step 4: Verify support
        support = self_rag_support_check(
            query, relevant_passages[0].page_content, generation
        )

        if support == "no_support" and attempt < max_retries - 1:
            query = rewrite_query(query)
            continue

        return {
            "answer": generation,
            "retrieved": True,
            "sources": [p.metadata for p in relevant_passages],
            "support_level": support,
        }

    # Fallback: generate with whatever we have
    return {
        "answer": generation,
        "retrieved": True,
        "sources": [],
        "support_level": "uncertain",
    }


def rewrite_query(query: str) -> str:
    """Rewrite query to improve retrieval."""
    response = client.messages.create(
        model="claude-3-5-haiku-20241022",
        max_tokens=100,
        messages=[{
            "role": "user",
            "content": (
                f"Rewrite this search query to be more specific and likely to "
                f"find relevant documents:\n{query}"
            ),
        }],
    )
    return response.content[0].text.strip()
```

### LangGraph Implementation

```python
from langgraph.graph import StateGraph, END
from typing import TypedDict


class SelfRAGState(TypedDict):
    query: str
    original_query: str
    passages: list[dict]
    relevant_passages: list[dict]
    generation: str
    needs_retrieval: bool
    support_level: str
    attempt: int


def decide_retrieval(state: SelfRAGState) -> SelfRAGState:
    needs = self_rag_retrieve_decision(state["query"])
    return {**state, "needs_retrieval": needs}


def retrieve_passages(state: SelfRAGState) -> SelfRAGState:
    passages = retriever.invoke(state["query"])
    return {
        **state,
        "passages": [{"content": p.page_content, "metadata": p.metadata} for p in passages],
    }


def filter_relevant(state: SelfRAGState) -> SelfRAGState:
    relevant = []
    for p in state["passages"]:
        if self_rag_relevance_check(state["query"], p["content"]) == "relevant":
            relevant.append(p)
    return {**state, "relevant_passages": relevant}


def generate_answer(state: SelfRAGState) -> SelfRAGState:
    if state["relevant_passages"]:
        context = "\n\n".join([p["content"] for p in state["relevant_passages"][:5]])
        prompt = f"Context:\n{context}\n\nQuery: {state['original_query']}"
    else:
        prompt = state["original_query"]

    response = llm.invoke(prompt)
    return {**state, "generation": response.content}


def verify_support(state: SelfRAGState) -> SelfRAGState:
    if not state["relevant_passages"]:
        return {**state, "support_level": "no_context"}

    support = self_rag_support_check(
        state["original_query"],
        state["relevant_passages"][0]["content"],
        state["generation"],
    )
    return {**state, "support_level": support, "attempt": state["attempt"] + 1}


# Routing
def route_retrieval(state):
    return "retrieve" if state["needs_retrieval"] else "generate"

def route_relevance(state):
    return "generate" if state["relevant_passages"] else "retry"

def route_support(state):
    if state["support_level"] == "fully_supported" or state["attempt"] >= 3:
        return END
    return "retry"

def retry_with_rewrite(state: SelfRAGState) -> SelfRAGState:
    new_query = rewrite_query(state["query"])
    return {**state, "query": new_query}


# Build graph
graph = StateGraph(SelfRAGState)
graph.add_node("decide", decide_retrieval)
graph.add_node("retrieve", retrieve_passages)
graph.add_node("filter", filter_relevant)
graph.add_node("generate", generate_answer)
graph.add_node("verify", verify_support)
graph.add_node("retry", retry_with_rewrite)

graph.set_entry_point("decide")
graph.add_conditional_edges("decide", route_retrieval, {
    "retrieve": "retrieve", "generate": "generate",
})
graph.add_edge("retrieve", "filter")
graph.add_conditional_edges("filter", route_relevance, {
    "generate": "generate", "retry": "retry",
})
graph.add_edge("generate", "verify")
graph.add_conditional_edges("verify", route_support, {
    END: END, "retry": "retry",
})
graph.add_edge("retry", "retrieve")

self_rag_app = graph.compile()
```

---

## When to Use Self-RAG

### Good Fit

1. **Knowledge-intensive tasks** where factual accuracy and source grounding are critical (medical, legal, compliance)
2. **Citation-required applications** where users need to verify claims against sources
3. **Mixed query types** where some queries need retrieval and others do not
4. **Long-form generation** where different sections may need different retrieval strategies

### Poor Fit

1. **Creative writing or open-ended generation** where grounding is not needed
2. **Simple chatbots** where the overhead of reflection is not justified
3. **Real-time applications** where the additional LLM calls for reflection exceed the latency budget (each reflection token approximation costs 100-300ms)

---

## Limitations

1. **Training complexity**: full Self-RAG requires training a custom model with reflection tokens, which needs significant compute and labeled data.
2. **Prompt-based approximation is imperfect**: using LLM-as-judge for reflection adds latency (multiple LLM calls per query) and is less reliable than trained reflection tokens.
3. **Calibration of reflection scores**: the IsUse scores (1-5) are not well-calibrated. A score of 4 from one query context may not mean the same as 4 from another.
4. **Retrieval decision accuracy**: the model may incorrectly skip retrieval for queries that need external knowledge, or unnecessarily retrieve for queries it can answer from parametric knowledge.
5. **Sequential processing**: reflection tokens are evaluated sequentially per segment, making parallelization difficult.

---

## Benchmarks

Self-RAG results from the original paper (Llama-2 13B backbone):

| Task | Standard RAG | Self-RAG | Improvement |
|------|-------------|----------|-------------|
| PopQA (factual) | 44.6 | 54.9 | +10.3 |
| TriviaQA | 68.5 | 73.2 | +4.7 |
| PubHealth (bio) | 72.3 | 76.2 | +3.9 |
| ARC-Challenge | 67.1 | 72.4 | +5.3 |
| ASQA (long-form) | 31.2 | 37.1 | +5.9 |
| FactScore (bio) | 32.7 | 40.2 | +7.5 |

Key finding: the improvement is largest on tasks requiring factual precision (PopQA, FactScore), where the reflection mechanism prevents hallucination.

---

## Common Pitfalls

1. **Not setting a retry limit.** Without a max-attempts cap, the pipeline can loop indefinitely between retrieval, relevance check, query rewrite, and retrieval again. Always cap at 2-3 attempts.
2. **Using heavy models for reflection.** Reflection checks (IsRel, IsSup) do not need a large model. Use Haiku-class models for reflection and Sonnet/Opus for generation.
3. **Evaluating support against only one passage.** The answer may synthesize information from multiple passages. Check support against the union of all relevant passages, not just the top one.
4. **Treating prompt-based Self-RAG as equivalent to trained Self-RAG.** Trained reflection tokens are faster, more consistent, and better calibrated than LLM-as-judge approximations. The prompt-based approach is a useful approximation, not a replacement.

---

## References

- Asai et al. "Self-RAG: Learning to Retrieve, Generate, and Critique through Self-Reflection" (ICLR 2024). https://arxiv.org/abs/2310.11511
- Self-RAG code: https://github.com/AkariAsai/self-rag
- LangGraph Self-RAG tutorial: https://langchain-ai.github.io/langgraph/tutorials/rag/langgraph_self_rag/
