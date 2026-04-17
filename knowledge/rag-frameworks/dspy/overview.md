# DSPy -- Philosophy, Signatures, and Modules

## Overview

DSPy (Declarative Self-improving Language Programs) is a framework for programming -- not prompting -- language models. Created at Stanford NLP, DSPy replaces hand-written prompts with composable modules whose behavior is optimized by compilers. Instead of writing "You are a helpful assistant that retrieves relevant passages and answers questions based on them...", you declare what a module should do (its Signature) and let DSPy figure out how to make it work well (through optimization).

The core insight: prompts are implementation details. They should be generated and tuned automatically, just as a compiler generates machine code from high-level source. DSPy 2.6+ (released early 2025) stabilized the module and optimizer API, introduced typed predictors with Pydantic, and added native async support.

This guide covers DSPy's programming model, the three foundational abstractions (Signatures, Modules, Optimizers), and how they compose into RAG pipelines.

---

## The DSPy Programming Model

Traditional LLM development:

```
Human writes prompt -> LLM executes prompt -> Human evaluates output -> Human rewrites prompt
```

DSPy development:

```
Developer defines Signature -> Developer writes Module -> Optimizer compiles Module -> Compiled Module runs
```

The key differences:

1. **Signatures** replace prompts. A Signature declares input/output fields with descriptions.
2. **Modules** replace prompt chains. A Module composes Signatures with control flow (if/else, loops, multi-hop).
3. **Optimizers** (formerly called Teleprompters) replace prompt engineering. An Optimizer takes a Module, training data, and a metric, then generates the best prompt or finetune for the task.

```python
import dspy

# Configure the LM (language model)
lm = dspy.LM("openai/gpt-4o-mini", temperature=0.0)
dspy.configure(lm=lm)

# Traditional approach: hand-written prompt
# "Given a question and context passages, answer the question accurately..."

# DSPy approach: declare what, not how
class RAGSignature(dspy.Signature):
    """Answer the question based on the provided context."""
    context: list[str] = dspy.InputField(desc="retrieved passages")
    question: str = dspy.InputField(desc="user question")
    answer: str = dspy.OutputField(desc="factual answer grounded in context")
```

---

## Signatures

A Signature is a typed declaration of a module's interface. It specifies what goes in and what comes out, along with natural-language descriptions that guide the LM.

### Inline Signatures (Quick and Simple)

For simple tasks, use string shorthand:

```python
# "input_field -> output_field"
classify = dspy.Predict("sentence -> sentiment: str")
result = classify(sentence="DSPy makes prompt engineering obsolete.")
print(result.sentiment)  # "positive"

# Multiple inputs/outputs
qa = dspy.Predict("context, question -> answer, confidence: float")
result = qa(
    context="DSPy was created at Stanford NLP by Omar Khattab.",
    question="Who created DSPy?",
)
print(result.answer)      # "Omar Khattab at Stanford NLP"
print(result.confidence)  # 0.95
```

### Class-Based Signatures (Production Use)

For real applications, define Signatures as classes with descriptions and type annotations:

```python
import dspy
from pydantic import BaseModel, Field


class Citation(BaseModel):
    text: str = Field(description="exact quote from context")
    source_index: int = Field(description="0-based index into context list")


class CitedAnswer(dspy.Signature):
    """Answer the question using ONLY information from the provided context.
    Include citations for every claim."""

    context: list[str] = dspy.InputField(
        desc="retrieved passages, each from a different source"
    )
    question: str = dspy.InputField(desc="user question to answer")
    reasoning: str = dspy.OutputField(
        desc="step-by-step reasoning about which passages are relevant"
    )
    answer: str = dspy.OutputField(
        desc="comprehensive answer grounded in context"
    )
    citations: list[Citation] = dspy.OutputField(
        desc="citations supporting each claim in the answer"
    )
```

### Signature Field Options

```python
class AdvancedSig(dspy.Signature):
    """Classify support tickets by urgency and route to department."""

    ticket_text: str = dspy.InputField(desc="raw support ticket content")
    customer_tier: str = dspy.InputField(
        desc="customer tier: free, pro, or enterprise",
        prefix="Customer Tier:",  # customize how this appears in the prompt
    )
    urgency: str = dspy.OutputField(
        desc="urgency level",
        prefix="Urgency Level:",
        # Constrain output values
    )
    department: str = dspy.OutputField(desc="routing department")
    summary: str = dspy.OutputField(desc="one-line summary for the agent")
```

---

## Modules

Modules are the workhorses of DSPy. A Module takes a Signature and adds behavior: how many times to call the LM, whether to retrieve information, how to chain reasoning steps.

### Built-in Modules

| Module | Purpose | When to Use |
|--------|---------|-------------|
| `dspy.Predict` | Single LM call | Simple input->output tasks |
| `dspy.ChainOfThought` | Adds reasoning step before output | Tasks requiring multi-step logic |
| `dspy.ReAct` | Interleaves reasoning with tool calls | Agentic tasks with external tools |
| `dspy.ProgramOfThought` | Generates and executes code | Math, data analysis, computation |
| `dspy.MultiChainComparison` | Runs multiple chains, picks best | High-stakes answers needing consensus |
| `dspy.Retry` | Retries with feedback on failure | When outputs must pass validation |

### dspy.Predict

The simplest module. One call to the LM:

```python
predictor = dspy.Predict(CitedAnswer)

result = predictor(
    context=["DSPy was released in 2023 by Stanford NLP."],
    question="When was DSPy released?",
)
print(result.answer)
print(result.citations)
```

### dspy.ChainOfThought

Automatically inserts a `reasoning` field before the output, encouraging the LM to think step-by-step:

```python
cot = dspy.ChainOfThought("question -> answer")
result = cot(question="If a RAG system retrieves 10 passages at 500 tokens each, "
                       "and the LLM context window is 4096 tokens, how many passages fit?")
print(result.reasoning)  # "Each passage is 500 tokens. 4096 / 500 = 8.19, so 8 passages fit."
print(result.answer)     # "8 passages"
```

### Custom Modules

Custom modules compose built-in modules with Python control flow:

```python
import dspy


class MultiHopRAG(dspy.Module):
    """Two-hop retrieval: retrieve, read, generate follow-up query, retrieve again, answer."""

    def __init__(self, num_passages: int = 3):
        self.num_passages = num_passages
        self.retrieve = dspy.Retrieve(k=num_passages)
        self.generate_query = dspy.ChainOfThought(
            "context, question -> follow_up_query"
        )
        self.generate_answer = dspy.ChainOfThought(
            "context, question -> answer"
        )

    def forward(self, question: str) -> dspy.Prediction:
        # First hop
        passages_1 = self.retrieve(question).passages

        # Generate a follow-up query based on what we learned
        follow_up = self.generate_query(
            context=passages_1,
            question=question,
        )

        # Second hop with refined query
        passages_2 = self.retrieve(follow_up.follow_up_query).passages

        # Combine all passages and answer
        all_passages = passages_1 + passages_2
        # Deduplicate
        seen = set()
        unique_passages = []
        for p in all_passages:
            if p not in seen:
                seen.add(p)
                unique_passages.append(p)

        prediction = self.generate_answer(
            context=unique_passages,
            question=question,
        )
        return dspy.Prediction(
            answer=prediction.answer,
            passages=unique_passages,
        )


# Use the module
multi_hop = MultiHopRAG(num_passages=5)
result = multi_hop(question="What optimizer should I use for a RAG pipeline in DSPy?")
print(result.answer)
```

### Module Composition Patterns

```python
class EnsembleRAG(dspy.Module):
    """Run multiple retrieval strategies and merge results."""

    def __init__(self):
        self.keyword_retrieve = dspy.Retrieve(k=5)
        self.semantic_retrieve = dspy.Retrieve(k=5)
        self.rerank = dspy.ChainOfThought(
            "question, passages -> ranked_passages: list[str]"
        )
        self.answer = dspy.ChainOfThought(
            "context, question -> answer"
        )

    def forward(self, question: str) -> dspy.Prediction:
        # Parallel retrieval (in practice, you would use different indexes)
        kw_results = self.keyword_retrieve(question).passages
        sem_results = self.semantic_retrieve(question).passages

        # Merge and deduplicate
        all_passages = list(dict.fromkeys(kw_results + sem_results))

        # LM-based reranking
        reranked = self.rerank(
            question=question,
            passages=all_passages,
        )

        return self.answer(
            context=reranked.ranked_passages[:5],
            question=question,
        )
```

---

## Retrievers in DSPy

DSPy provides a `dspy.Retrieve` module and a retriever model (RM) abstraction:

```python
import dspy

# Configure a retriever
colbert_rm = dspy.ColBERTv2(url="http://localhost:8893/api/search")
dspy.configure(lm=lm, rm=colbert_rm)

# Or use other built-in retrievers
from dspy.retrieve.chromadb_rm import ChromadbRM
from dspy.retrieve.qdrant_rm import QdrantRM
from dspy.retrieve.pinecone_rm import PineconeRM

# ChromaDB
chroma_rm = ChromadbRM(
    collection_name="my_docs",
    persist_directory="./chroma_db",
    embedding_function=None,  # uses default
    k=5,
)

# Qdrant
qdrant_rm = QdrantRM(
    qdrant_collection_name="my_docs",
    qdrant_client=qdrant_client,
    k=5,
)

# Pinecone
pinecone_rm = PineconeRM(
    pinecone_index_name="my-index",
    pinecone_api_key="your-key",
    k=5,
)

dspy.configure(lm=lm, rm=chroma_rm)

# Now dspy.Retrieve() uses the configured RM
retriever = dspy.Retrieve(k=5)
results = retriever("What is DSPy?")
print(results.passages)  # list of retrieved text passages
```

---

## Evaluation and Metrics

Before optimizing, you need a metric. DSPy metrics are Python functions:

```python
def answer_correctness(example, prediction, trace=None):
    """Check if the predicted answer matches the gold answer."""
    gold = example.answer.lower().strip()
    pred = prediction.answer.lower().strip()
    return gold in pred or pred in gold


def answer_has_citations(example, prediction, trace=None):
    """Check that the answer includes at least one citation."""
    return len(prediction.citations) > 0


def composite_metric(example, prediction, trace=None):
    """Combine multiple metrics."""
    correct = answer_correctness(example, prediction, trace)
    cited = answer_has_citations(example, prediction, trace)
    if trace is not None:
        # During optimization, return True/False for bootstrapping
        return correct and cited
    # During evaluation, return a score
    return (correct + cited) / 2.0
```

Use `dspy.Evaluate` to measure performance:

```python
from dspy.evaluate import Evaluate

# Load your dataset
trainset = [
    dspy.Example(
        question="Who created DSPy?",
        answer="Omar Khattab",
    ).with_inputs("question"),
    # ... more examples
]

devset = trainset[:50]
testset = trainset[50:]

# Evaluate
evaluator = Evaluate(
    devset=devset,
    metric=answer_correctness,
    num_threads=4,
    display_progress=True,
    display_table=5,  # show 5 example rows
)

score = evaluator(multi_hop)
print(f"Accuracy: {score}%")
```

---

## Language Model Configuration

DSPy supports multiple LM providers through a unified interface:

```python
import dspy

# OpenAI
lm = dspy.LM("openai/gpt-4o-mini", temperature=0.0, max_tokens=1000)

# Anthropic
lm = dspy.LM("anthropic/claude-sonnet-4-20250514", temperature=0.0)

# Local models via Ollama
lm = dspy.LM("ollama_chat/llama3.1:8b", api_base="http://localhost:11434")

# Azure OpenAI
lm = dspy.LM(
    "azure/gpt-4o",
    api_base="https://your-resource.openai.azure.com/",
    api_version="2024-08-01-preview",
)

# vLLM or any OpenAI-compatible server
lm = dspy.LM(
    "openai/your-model-name",
    api_base="http://localhost:8000/v1",
    api_key="not-needed",
)

dspy.configure(lm=lm)

# Inspect the last LM call for debugging
lm.inspect_history(n=1)
```

---

## Typed Predictors with Pydantic

DSPy 2.5+ supports Pydantic models for structured output:

```python
from pydantic import BaseModel, Field
import dspy


class RetrievalResult(BaseModel):
    passage: str = Field(description="the most relevant passage")
    relevance_score: float = Field(ge=0.0, le=1.0, description="relevance 0-1")
    reasoning: str = Field(description="why this passage is relevant")


class AnswerWithSources(BaseModel):
    answer: str = Field(description="the answer to the question")
    sources: list[RetrievalResult] = Field(description="supporting passages")
    confidence: float = Field(ge=0.0, le=1.0)


class StructuredRAG(dspy.Signature):
    """Answer questions with structured, cited responses."""
    context: list[str] = dspy.InputField()
    question: str = dspy.InputField()
    response: AnswerWithSources = dspy.OutputField()


predictor = dspy.Predict(StructuredRAG)
result = predictor(
    context=["DSPy uses optimizers to compile prompts.", "MIPROv2 is the recommended optimizer."],
    question="How does DSPy optimize prompts?",
)
# result.response is an AnswerWithSources instance
print(result.response.answer)
print(result.response.confidence)
```

---

## Assertions and Constraints

DSPy Assertions let you add runtime constraints that trigger automatic retries:

```python
import dspy


class ConstrainedRAG(dspy.Module):
    def __init__(self):
        self.retrieve = dspy.Retrieve(k=5)
        self.answer = dspy.ChainOfThought("context, question -> answer, citations: list[str]")

    def forward(self, question: str) -> dspy.Prediction:
        passages = self.retrieve(question).passages
        result = self.answer(context=passages, question=question)

        # Hard constraint: must have citations
        dspy.Assert(
            len(result.citations) > 0,
            "Answer must include at least one citation from the context.",
        )

        # Soft constraint: suggest but don't fail
        dspy.Suggest(
            len(result.answer) > 50,
            "Answer should be detailed (more than 50 characters).",
        )

        # Hard constraint: citations must come from retrieved passages
        for citation in result.citations:
            dspy.Assert(
                any(citation in p for p in passages),
                f"Citation '{citation[:50]}...' not found in any retrieved passage.",
            )

        return result
```

---

## Common Pitfalls

1. **Not configuring the LM before using modules**: Always call `dspy.configure(lm=lm)` before instantiating or calling any module. Without this, you get cryptic errors.

2. **Forgetting `.with_inputs()` on examples**: When creating training examples, you must mark which fields are inputs. Without this, the optimizer cannot distinguish inputs from labels.

3. **Using temperature > 0 for deterministic tasks**: For retrieval and factual QA, set `temperature=0.0` on the LM. Non-zero temperature introduces randomness that makes optimization unstable.

4. **Writing prompts inside Signatures**: The docstring on a Signature class is the instruction. Keep it short and declarative. Do not write multi-paragraph prompts -- that defeats the purpose of DSPy.

5. **Evaluating on the training set**: Always split your data into train/dev/test. Evaluate on dev during optimization and on test for final numbers. Overfitting to the training set is the most common cause of optimization not generalizing.

6. **Not inspecting LM history**: Use `lm.inspect_history(n=3)` to see exactly what prompts DSPy generates. This is essential for debugging unexpected behavior.

---

## When to Use DSPy vs. Other Frameworks

| Scenario | Use DSPy | Use LangChain/LlamaIndex |
|----------|----------|--------------------------|
| You have labeled data and want to optimize | Yes | No |
| You need complex prompt engineering | Yes (let the optimizer do it) | Yes (manual) |
| You want quick prototyping | Moderate (need to learn Signatures) | Yes (more examples available) |
| Production RAG with metrics | Yes | Possible but manual prompt tuning |
| Multi-hop reasoning | Yes (native Module composition) | Yes (with custom chains) |
| You need many integrations | Fewer than LangChain | LangChain has more |

---

## References

- DSPy documentation: https://dspy.ai/
- DSPy GitHub: https://github.com/stanfordnlp/dspy
- Original paper: "DSPy: Compiling Declarative Language Model Calls into Self-Improving Pipelines"
- DSPy Discord: https://discord.gg/dspy
