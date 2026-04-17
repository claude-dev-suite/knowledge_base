# Giskard RAGET -- Automatic Testset Generation and Component-Level Evaluation

## TL;DR

Giskard RAGET (RAG Evaluation Toolkit) is a Python framework that automatically generates evaluation test sets from your knowledge base and evaluates RAG pipelines at the component level (retriever and generator independently). Unlike RAGAS which evaluates the pipeline as a whole, RAGET generates four types of test questions -- simple, complex, distracting, and conversational -- by analyzing your actual documents, then scores the retriever and generator separately to pinpoint exactly where quality issues originate. This means you get actionable diagnostics ("your retriever fails on multi-hop questions") rather than aggregate scores that leave you guessing. The key differentiator is that you do not need to manually create golden datasets: RAGET builds them from your corpus automatically.

---

## The Problem RAGET Solves

### Manual Test Set Creation Is a Bottleneck

Every RAG evaluation framework needs a test set of query-answer pairs. Creating these manually is:

- **Expensive**: 50-200 high-quality question-answer pairs take 8-20 hours of domain expert time
- **Biased**: human-created questions tend to be simpler and more direct than real user queries
- **Incomplete**: manually written test sets rarely cover edge cases like distracting information or multi-hop reasoning
- **Static**: once created, test sets do not evolve as the corpus changes

RAGET solves this by automatically scanning your knowledge base documents, identifying key information, and generating diverse questions that test different aspects of retrieval and generation.

### Component-Level vs Pipeline-Level Evaluation

```
Pipeline-level evaluation (RAGAS, DeepEval):
  Query --> [RAG Pipeline] --> Answer --> Score
  
  Problem: if the score is low, is it the retriever or the generator?
  You cannot tell from aggregate metrics.

Component-level evaluation (RAGET):
  Query --> [Retriever] --> Contexts --> Retriever Score
  Query + Contexts --> [Generator] --> Answer --> Generator Score
  
  Benefit: you know exactly which component to fix.
```

| Metric | What It Tells You | Component |
|---|---|---|
| Context Precision | Are retrieved docs relevant? | Retriever |
| Context Recall | Are all needed docs retrieved? | Retriever |
| Faithfulness | Is the answer grounded in context? | Generator |
| Answer Relevancy | Does the answer address the query? | Generator |
| Correctness | Is the answer factually correct? | End-to-end |

---

## How RAGET Works

### The Three-Phase Pipeline

```
Phase 1: Knowledge Base Analysis
  Documents --> Extract facts, entities, relationships
  --> Build a knowledge graph of your corpus
  --> Identify information density per document

Phase 2: Question Generation
  Knowledge graph --> Generate 4 question types:
    - Simple: single-fact retrieval
    - Complex: multi-hop reasoning across documents
    - Distracting: questions with plausible but wrong contexts
    - Conversational: follow-up questions requiring context
  --> Each question has: query, reference answer, reference contexts

Phase 3: Component-Level Evaluation
  Test set --> Run retriever --> Score retrieval quality
  Test set --> Run generator with retrieved contexts --> Score generation quality
  Test set --> Run generator with reference contexts --> Isolate generator quality
  --> Component-level scores + failure analysis
```

### Question Types Explained

#### Simple Questions

Test basic single-fact retrieval. The answer is contained in a single passage.

```
Document: "PostgreSQL supports three types of indexes: B-tree (default),
Hash, and GiST. B-tree indexes are most suitable for equality and
range queries."

Generated question: "What is the default index type in PostgreSQL?"
Expected answer: "B-tree is the default index type in PostgreSQL."
```

#### Complex Questions

Require information from multiple documents or multiple sections. Test multi-hop reasoning.

```
Document A: "Redis Cluster partitions data across multiple nodes using
hash slots. There are 16384 hash slots distributed among nodes."

Document B: "When a Redis Cluster node fails, its replica takes over
the hash slots. This failover process takes approximately 1-2 seconds."

Generated question: "How many hash slots need to be reassigned when a
Redis Cluster node fails, and how long does the failover take?"
Expected answer: "The number of hash slots depends on the cluster
distribution (out of 16384 total), and failover takes 1-2 seconds."
```

#### Distracting Questions

Include contexts that are topically related but do not contain the answer. Test the retriever's ability to distinguish relevant from merely related content.

```
Document (relevant): "Python's asyncio.gather() runs multiple
coroutines concurrently and returns results in order."

Document (distractor): "Python's threading module provides Thread
and Lock primitives for concurrent execution using OS threads."

Generated question: "How do you run multiple async operations
concurrently in Python and get ordered results?"
Expected answer: "Use asyncio.gather() which runs coroutines
concurrently and returns results in the order they were passed."
```

#### Conversational Questions

Simulate multi-turn conversations where follow-up questions depend on previous context.

```
Turn 1: "What is Kafka's default retention period?"
Answer 1: "Kafka retains messages for 7 days by default."

Turn 2: "How do I change it to 30 days?"
Answer 2: "Set retention.ms to 2592000000 in the topic
configuration or broker defaults."
```

---

## Installation and Setup

```bash
pip install "giskard[llm]>=2.0"
```

### Basic Configuration

```python
"""
Giskard RAGET basic setup and configuration.
"""
import os

import giskard
import pandas as pd


# Configure the LLM used for test generation and evaluation
# RAGET uses an LLM to generate questions and judge answers
giskard.llm.set_default_client("anthropic")
os.environ["ANTHROPIC_API_KEY"] = "your-key"

# Alternatively, use OpenAI
# giskard.llm.set_default_client("openai")
# os.environ["OPENAI_API_KEY"] = "your-key"
```

### Preparing Your Knowledge Base

```python
"""
Prepare documents for RAGET testset generation.
"""
import pandas as pd
from pathlib import Path


def load_knowledge_base(docs_dir: str) -> pd.DataFrame:
    """
    Load documents into a DataFrame for RAGET.

    RAGET expects a DataFrame with at minimum a text column.
    Additional columns (source, title) improve question quality.
    """
    records = []

    for md_file in Path(docs_dir).rglob("*.md"):
        content = md_file.read_text(encoding="utf-8")

        # Skip very short documents
        if len(content) < 100:
            continue

        records.append({
            "text": content,
            "source": str(md_file.relative_to(docs_dir)),
            "title": md_file.stem.replace("-", " ").title(),
        })

    df = pd.DataFrame(records)
    print(f"Loaded {len(df)} documents")
    print(f"Avg length: {df['text'].str.len().mean():.0f} chars")

    return df


def load_from_vector_store(
    collection_name: str,
    limit: int = 500,
) -> pd.DataFrame:
    """
    Load documents from a vector store for RAGET.

    Use this when you want to generate tests from the same
    chunks your pipeline actually retrieves.
    """
    # Example with ChromaDB
    import chromadb
    client = chromadb.PersistentClient(path="./chroma_db")
    collection = client.get_collection(collection_name)

    results = collection.get(
        limit=limit,
        include=["documents", "metadatas"],
    )

    records = []
    for doc, meta in zip(results["documents"], results["metadatas"]):
        records.append({
            "text": doc,
            "source": meta.get("source", "unknown"),
        })

    return pd.DataFrame(records)
```

---

## Generating Test Sets

### Full Testset Generation

```python
"""
Generate evaluation test sets from your knowledge base.
"""
from giskard.rag import KnowledgeBase, generate_testset, QATestset


def generate_rag_testset(
    knowledge_df: pd.DataFrame,
    num_questions: int = 100,
    question_distribution: dict | None = None,
) -> QATestset:
    """
    Generate a test set from the knowledge base.

    Args:
        knowledge_df: DataFrame with 'text' column (and optionally 'source')
        num_questions: total number of questions to generate
        question_distribution: dict mapping question type to proportion
            Default: {"simple": 0.4, "complex": 0.3,
                      "distracting": 0.2, "conversational": 0.1}

    Returns:
        QATestset with generated questions, answers, and metadata
    """
    # Create knowledge base object
    knowledge_base = KnowledgeBase.from_pandas(knowledge_df)

    # Default distribution
    if question_distribution is None:
        question_distribution = {
            "simple": 0.40,
            "complex": 0.30,
            "distracting": 0.20,
            "conversational": 0.10,
        }

    print(f"Generating {num_questions} questions...")
    print(f"Distribution: {question_distribution}")

    testset = generate_testset(
        knowledge_base,
        num_questions=num_questions,
        language="en",
        agent_description=(
            "A RAG system that answers technical questions "
            "based on internal documentation."
        ),
    )

    # Report statistics
    print(f"\nGenerated {len(testset)} questions:")
    for q_type, count in testset.question_type_counts().items():
        print(f"  {q_type}: {count}")

    return testset


def inspect_generated_questions(testset: QATestset) -> None:
    """
    Inspect generated questions for quality before using them.

    Always review a sample of generated questions to ensure they
    are well-formed and answerable.
    """
    df = testset.to_pandas()

    print(f"\nTotal questions: {len(df)}")
    print(f"\nColumns: {list(df.columns)}")

    # Show examples of each type
    for q_type in df["question_type"].unique():
        subset = df[df["question_type"] == q_type]
        print(f"\n{'='*60}")
        print(f"Question type: {q_type} ({len(subset)} questions)")
        print(f"{'='*60}")

        for _, row in subset.head(2).iterrows():
            print(f"\n  Q: {row['question']}")
            print(f"  A: {row['reference_answer'][:200]}")
            if "reference_context" in row:
                ctx = str(row["reference_context"])[:150]
                print(f"  Context: {ctx}...")


def save_testset(testset: QATestset, output_path: str) -> None:
    """Save testset for reuse in CI/CD."""
    testset.save(output_path)
    print(f"Saved testset to {output_path}")


def load_testset(path: str) -> QATestset:
    """Load a previously generated testset."""
    return QATestset.load(path)
```

---

## Component-Level Evaluation

### Wrapping Your Pipeline for RAGET

```python
"""
Wrap your RAG pipeline components for RAGET evaluation.
"""
from giskard.rag import evaluate as raget_evaluate


def wrap_rag_pipeline(retriever_fn, generator_fn):
    """
    Wrap your RAG pipeline for RAGET component-level evaluation.

    RAGET needs to call the retriever and generator independently
    to measure each component separately.
    """

    def answer_fn(question: str, history: list | None = None) -> str:
        """The full pipeline: retrieve then generate."""
        contexts = retriever_fn(question)
        context_text = "\n\n".join(contexts)
        answer = generator_fn(question, context_text)
        return answer

    return answer_fn


def run_component_evaluation(
    answer_fn,
    testset: "QATestset",
    knowledge_base: "KnowledgeBase",
) -> dict:
    """
    Run RAGET component-level evaluation.

    This evaluates both the retriever and generator independently,
    providing per-component scores and failure analysis.
    """
    report = raget_evaluate(
        answer_fn=answer_fn,
        testset=testset,
        knowledge_base=knowledge_base,
    )

    # Extract component-level metrics
    print("\n=== RAGET Evaluation Report ===\n")

    # Retriever metrics
    print("Retriever Performance:")
    for metric_name, score in report.retriever_metrics.items():
        print(f"  {metric_name}: {score:.4f}")

    # Generator metrics
    print("\nGenerator Performance:")
    for metric_name, score in report.generator_metrics.items():
        print(f"  {metric_name}: {score:.4f}")

    # Per question-type breakdown
    print("\nPerformance by Question Type:")
    for q_type, metrics in report.metrics_by_question_type.items():
        print(f"\n  {q_type}:")
        for metric_name, score in metrics.items():
            print(f"    {metric_name}: {score:.4f}")

    # Failure analysis
    print("\nFailure Analysis:")
    failures = report.get_failures()
    for failure_type, examples in failures.items():
        print(f"\n  {failure_type}: {len(examples)} failures")
        for ex in examples[:2]:
            print(f"    Q: {ex['question'][:80]}...")

    return {
        "retriever": report.retriever_metrics,
        "generator": report.generator_metrics,
        "by_question_type": report.metrics_by_question_type,
        "num_failures": sum(len(v) for v in failures.values()),
    }
```

### Understanding Component Scores

```python
"""
Interpreting RAGET component-level scores.
"""


def diagnose_pipeline(
    retriever_metrics: dict,
    generator_metrics: dict,
) -> list[str]:
    """
    Interpret RAGET metrics and produce actionable recommendations.

    Returns a list of recommendations ordered by priority.
    """
    recommendations = []

    # Retriever diagnostics
    ctx_precision = retriever_metrics.get("context_precision", 0)
    ctx_recall = retriever_metrics.get("context_recall", 0)

    if ctx_recall < 0.60:
        recommendations.append(
            "RETRIEVER - LOW RECALL ({:.2f}): The retriever is missing relevant "
            "documents. Consider:\n"
            "  1. Increasing K (retrieve more documents)\n"
            "  2. Using hybrid search (BM25 + dense) for better coverage\n"
            "  3. Checking if relevant documents are properly indexed\n"
            "  4. Reducing chunk size to prevent relevant info from being "
            "buried in large chunks".format(ctx_recall)
        )

    if ctx_precision < 0.50:
        recommendations.append(
            "RETRIEVER - LOW PRECISION ({:.2f}): The retriever returns too many "
            "irrelevant documents. Consider:\n"
            "  1. Adding a reranker (cross-encoder) to filter retrieved docs\n"
            "  2. Using metadata filtering to narrow the search space\n"
            "  3. Improving embedding quality (fine-tune or use a better model)\n"
            "  4. Reducing K to return fewer, more focused results".format(ctx_precision)
        )

    # Generator diagnostics
    faithfulness = generator_metrics.get("faithfulness", 0)
    relevancy = generator_metrics.get("answer_relevancy", 0)
    correctness = generator_metrics.get("correctness", 0)

    if faithfulness < 0.70:
        recommendations.append(
            "GENERATOR - LOW FAITHFULNESS ({:.2f}): The LLM is not grounding "
            "its answers in the retrieved context. Consider:\n"
            "  1. Adding explicit grounding instructions to the prompt\n"
            "  2. Using citation-required prompting (ask the LLM to cite sources)\n"
            "  3. Switching to a model with better instruction following\n"
            "  4. Implementing Self-RAG to verify grounding".format(faithfulness)
        )

    if relevancy < 0.65:
        recommendations.append(
            "GENERATOR - LOW RELEVANCY ({:.2f}): Answers do not address the "
            "questions. Consider:\n"
            "  1. Improving the system prompt to focus on answering the question\n"
            "  2. Checking if the context is confusing the LLM\n"
            "  3. Adding query reformulation before generation".format(relevancy)
        )

    if faithfulness >= 0.85 and correctness < 0.60:
        recommendations.append(
            "RETRIEVER-GENERATOR MISMATCH: High faithfulness ({:.2f}) but low "
            "correctness ({:.2f}). The LLM faithfully uses the context, but "
            "the context is wrong or incomplete. Focus on retriever "
            "improvements.".format(faithfulness, correctness)
        )

    if not recommendations:
        recommendations.append(
            "All metrics are within acceptable ranges. "
            "Consider running with more test questions for higher confidence."
        )

    return recommendations
```

---

## Customizing Question Generation

### Controlling Question Distribution

```python
"""
Customize RAGET question generation for your use case.
"""
from giskard.rag import KnowledgeBase, generate_testset


def generate_retrieval_focused_testset(
    knowledge_base: KnowledgeBase,
    num_questions: int = 100,
) -> "QATestset":
    """
    Generate a testset focused on retrieval challenges.

    Emphasizes distracting and complex questions that stress-test
    the retriever.
    """
    return generate_testset(
        knowledge_base,
        num_questions=num_questions,
        language="en",
        # Heavy on retrieval-challenging question types
        distribution={
            "simple": 0.20,
            "complex": 0.30,
            "distracting": 0.40,
            "conversational": 0.10,
        },
    )


def generate_generation_focused_testset(
    knowledge_base: KnowledgeBase,
    num_questions: int = 100,
) -> "QATestset":
    """
    Generate a testset focused on generation challenges.

    Emphasizes questions that require reasoning, synthesis, and
    accurate grounding.
    """
    return generate_testset(
        knowledge_base,
        num_questions=num_questions,
        language="en",
        # Heavy on questions that challenge the generator
        distribution={
            "simple": 0.15,
            "complex": 0.45,
            "distracting": 0.10,
            "conversational": 0.30,
        },
    )


def generate_domain_specific_testset(
    knowledge_base: KnowledgeBase,
    domain_description: str,
    user_personas: list[str],
    num_questions: int = 100,
) -> "QATestset":
    """
    Generate questions tailored to your domain and user base.
    """
    # Build a rich agent description for better question quality
    persona_text = "\n".join(f"- {p}" for p in user_personas)
    agent_desc = (
        f"{domain_description}\n\n"
        f"Typical users include:\n{persona_text}\n\n"
        "Generate questions that these users would realistically ask."
    )

    return generate_testset(
        knowledge_base,
        num_questions=num_questions,
        language="en",
        agent_description=agent_desc,
    )


# Example usage
# kb = KnowledgeBase.from_pandas(docs_df)
# testset = generate_domain_specific_testset(
#     kb,
#     domain_description="Internal engineering documentation for a fintech company",
#     user_personas=[
#         "Backend engineers looking for API documentation",
#         "DevOps engineers troubleshooting infrastructure",
#         "New hires onboarding to the codebase",
#     ],
#     num_questions=150,
# )
```

---

## Integration with Other Frameworks

### Using RAGET Test Sets with RAGAS

```python
"""
Use RAGET-generated test sets with RAGAS evaluation.

Best of both worlds: RAGET's automatic test generation +
RAGAS's established metrics.
"""
from datasets import Dataset
from giskard.rag import QATestset
from ragas import evaluate
from ragas.metrics import (
    answer_relevancy,
    context_precision,
    context_recall,
    faithfulness,
)


def raget_testset_to_ragas(
    testset: QATestset,
    rag_pipeline,
) -> Dataset:
    """
    Convert a RAGET testset to RAGAS format and run the pipeline.
    """
    df = testset.to_pandas()

    questions = []
    answers = []
    contexts = []
    ground_truths = []

    for _, row in df.iterrows():
        query = row["question"]
        output = rag_pipeline.invoke(query)

        questions.append(query)
        answers.append(output["answer"])
        contexts.append(output.get("contexts", []))
        ground_truths.append(row.get("reference_answer", ""))

    return Dataset.from_dict({
        "question": questions,
        "answer": answers,
        "contexts": contexts,
        "ground_truth": ground_truths,
    })


def evaluate_with_ragas_and_raget(
    testset: QATestset,
    rag_pipeline,
) -> dict:
    """Run RAGAS evaluation on RAGET-generated questions."""
    dataset = raget_testset_to_ragas(testset, rag_pipeline)

    results = evaluate(
        dataset,
        metrics=[faithfulness, answer_relevancy, context_precision, context_recall],
    )

    print("RAGAS scores on RAGET-generated questions:")
    for metric, score in results.items():
        if isinstance(score, float):
            print(f"  {metric}: {score:.4f}")

    return results
```

---

## Common Pitfalls

1. **Corpus too small**: RAGET needs sufficient document diversity to generate varied questions. With fewer than 10 documents, question quality and diversity will be poor. Aim for 30+ documents minimum.

2. **Not reviewing generated questions**: RAGET's questions are LLM-generated and can contain errors (wrong expected answers, ambiguous questions, questions not answerable from the corpus). Always review a sample (20-30 questions) before using the test set for evaluation.

3. **Ignoring question type performance**: if your pipeline scores well on simple questions but poorly on complex ones, the overall score hides a critical weakness. Always analyze per-question-type metrics.

4. **Regenerating test sets too frequently**: each generation produces different questions, making longitudinal comparison impossible. Generate once, save, and reuse for consistent tracking. Regenerate only when the corpus changes significantly.

5. **Conflating component and pipeline metrics**: a retriever with high recall but low precision will feed noisy context to the generator. If the generator has high faithfulness, it faithfully uses irrelevant content. Diagnose components independently.

6. **Not accounting for LLM cost**: testset generation and evaluation both make LLM calls. A 200-question testset generation costs $5-15 depending on the model. Factor this into your evaluation budget.

---

## References

- Giskard RAGET: https://docs.giskard.ai/en/latest/open_source/testset_generation/
- Giskard RAG Evaluation: https://docs.giskard.ai/en/latest/open_source/testset_generation/rag_evaluation/
- Giskard GitHub: https://github.com/Giskard-AI/giskard
- RAGAS: https://docs.ragas.io/ (compatible metrics framework)
- "Automatic Test Set Generation for RAG" -- Giskard Research Blog
