# Synthetic Evaluation Data Generation for RAG

## TL;DR

Synthetic evaluation data is generated programmatically from your knowledge base documents, producing (question, answer, context) triples without manual annotation. Tools like RAGAS TestsetGenerator, Giskard RAGET, and LlamaIndex's synthetic generation module can bootstrap an evaluation set in minutes. Synthetic data is useful for rapid iteration and initial pipeline benchmarking, but has known biases: it overrepresents straightforward factual queries and underrepresents the edge cases that matter most. The recommended workflow is: generate synthetic data first for quick iteration, then refine with manual annotation for production evaluation.

---

## When to Use Synthetic Data

### Good Fit

1. **Early development**: no production traffic yet, need something to evaluate against
2. **Rapid iteration**: testing many configurations quickly before investing in manual annotation
3. **Coverage expansion**: generating questions about newly added documents
4. **Regression testing**: quick sanity checks after pipeline changes
5. **Training rerankers/classifiers**: generating large training sets for downstream models

### Poor Fit

1. **Final production evaluation**: synthetic data has blind spots; use a golden set
2. **Measuring user satisfaction**: synthetic questions do not reflect real user behavior
3. **Testing edge cases**: synthetic generators produce "clean" questions, not the messy queries users actually type
4. **Comparing vendor solutions**: synthetic bias may favor one architecture over another

---

## RAGAS TestsetGenerator

### Overview

RAGAS provides a `TestsetGenerator` that creates evaluation datasets from documents by:
1. Analyzing document structure and content
2. Generating questions of varying complexity (simple, reasoning, multi-context)
3. Producing corresponding reference answers
4. Tracking which documents each question was derived from

### Basic Usage

```python
from ragas.testset import TestsetGenerator
from ragas.llms import LangchainLLMWrapper
from ragas.embeddings import LangchainEmbeddingsWrapper
from langchain_anthropic import ChatAnthropic
from langchain_openai import OpenAIEmbeddings
from langchain_community.document_loaders import DirectoryLoader


# Load your knowledge base documents
loader = DirectoryLoader("./knowledge_base/", glob="**/*.md")
documents = loader.load()

# Configure generator
generator_llm = LangchainLLMWrapper(
    ChatAnthropic(model="claude-3-5-sonnet-20241022")
)
generator_embeddings = LangchainEmbeddingsWrapper(
    OpenAIEmbeddings(model="text-embedding-3-small")
)

generator = TestsetGenerator(
    llm=generator_llm,
    embedding_model=generator_embeddings,
)

# Generate test set
testset = generator.generate_with_langchain_docs(
    documents=documents,
    testset_size=100,
)

# Convert to dataset
dataset = testset.to_dataset()
print(dataset)
# Dataset({
#     features: ['question', 'ground_truth', 'contexts', 'evolution_type'],
#     num_rows: 100
# })
```

### Controlling Question Types

RAGAS generates three types of questions:

| Type | Description | Example | Proportion |
|------|-------------|---------|-----------|
| Simple | Direct factual extraction | "What is the default port for PostgreSQL?" | ~40% |
| Reasoning | Requires inference from context | "Why does connection pooling improve performance?" | ~30% |
| Multi-context | Needs information from multiple chunks | "How do RLS and RBAC complement each other?" | ~30% |

```python
from ragas.testset.evolutions import (
    simple,
    reasoning,
    multi_context,
)

# Custom distribution
testset = generator.generate_with_langchain_docs(
    documents=documents,
    testset_size=100,
    distributions={
        simple: 0.3,
        reasoning: 0.4,
        multi_context: 0.3,
    },
)
```

### Filtering Low-Quality Generations

Not all synthetic questions are good. Filter by:

```python
def filter_synthetic_questions(
    testset_df,
    min_question_length: int = 10,
    min_answer_length: int = 20,
    max_answer_length: int = 2000,
) -> list[dict]:
    """Filter out low-quality synthetic questions."""
    filtered = []
    for _, row in testset_df.iterrows():
        question = row["question"]
        answer = row["ground_truth"]

        # Skip very short questions (likely malformed)
        if len(question) < min_question_length:
            continue

        # Skip very short or very long answers
        if len(answer) < min_answer_length or len(answer) > max_answer_length:
            continue

        # Skip questions that are too similar to the answer (parroting)
        from difflib import SequenceMatcher
        similarity = SequenceMatcher(None, question.lower(), answer.lower()).ratio()
        if similarity > 0.7:
            continue

        # Skip yes/no questions (hard to evaluate)
        if answer.strip().lower() in ("yes", "no", "true", "false"):
            continue

        filtered.append(row.to_dict())

    print(f"Filtered: {len(testset_df)} -> {len(filtered)} questions")
    return filtered
```

---

## Giskard RAGET

### Overview

Giskard's RAGET (RAG Evaluation Toolkit) generates question-answer pairs with a focus on identifying failure modes. It creates questions specifically designed to test:
- Hallucination resistance
- Off-topic handling
- Factual accuracy

### Usage

```python
from giskard.rag import KnowledgeBase, generate_testset, QATestset
from giskard.llm.client import set_default_client
import pandas as pd

# Configure LLM
set_default_client("anthropic/claude-3-5-sonnet-20241022")

# Load knowledge base
df = pd.DataFrame({
    "text": [doc.page_content for doc in documents],
    "source": [doc.metadata.get("source", "") for doc in documents],
})
knowledge_base = KnowledgeBase(df, columns=["text"])

# Generate test set
testset = generate_testset(
    knowledge_base=knowledge_base,
    num_questions=100,
    agent_description="A RAG system for PostgreSQL documentation",
)

# Access generated questions
for item in testset.to_pandas().itertuples():
    print(f"Q: {item.question}")
    print(f"A: {item.reference_answer}")
    print(f"Type: {item.question_type}")
    print()
```

### Giskard Question Types

| Type | Purpose | Example |
|------|---------|---------|
| Simple | Direct factual | "What is the default value of work_mem?" |
| Complex | Multi-step reasoning | "How does shared_buffers interact with work_mem for query performance?" |
| Distracting | Tests noise resistance | "Is PostgreSQL faster than MongoDB for JSON operations?" |
| Situational | Context-dependent | "When should I choose SERIALIZABLE over READ COMMITTED?" |
| Double | Multiple sub-questions | "What are the pros and cons of connection pooling and when should you use PgBouncer vs pgpool?" |
| Conversational | Follow-up questions | "Tell me about RLS." / "How do I configure it for multi-tenant apps?" |

---

## LlamaIndex Synthetic Generation

### Overview

LlamaIndex generates (question, answer) pairs from indexed documents using its query engine.

### Usage

```python
from llama_index.core import VectorStoreIndex, SimpleDirectoryReader
from llama_index.core.evaluation import DatasetGenerator


# Load and index documents
documents = SimpleDirectoryReader("./knowledge_base").load_data()
index = VectorStoreIndex.from_documents(documents)

# Create dataset generator
generator = DatasetGenerator.from_documents(
    documents,
    num_questions_per_chunk=2,
)

# Generate questions
questions = generator.generate_questions()
print(f"Generated {len(questions)} questions")

# Generate question + reference answer pairs
qa_pairs = generator.generate_questions_from_nodes()
for qa in qa_pairs[:5]:
    print(f"Q: {qa.query}")
    print(f"A: {qa.reference_answer}")
    print()
```

### Custom Question Generation with LlamaIndex

```python
from llama_index.core.evaluation import generate_question_context_pairs
from llama_index.core.node_parser import SentenceSplitter

# Parse documents into nodes
parser = SentenceSplitter(chunk_size=1024, chunk_overlap=128)
nodes = parser.get_nodes_from_documents(documents)

# Generate QA pairs with custom prompts
qa_dataset = generate_question_context_pairs(
    nodes=nodes,
    llm=llm,
    num_questions_per_chunk=2,
    qa_generate_prompt_tmpl=(
        "Context:\n{context_str}\n\n"
        "Given the context above, generate a specific technical question "
        "that can be answered using ONLY the information in the context. "
        "The question should require understanding, not just keyword matching.\n\n"
        "Question:"
    ),
)

# Save for evaluation
qa_dataset.save_json("synthetic_eval_set.json")
```

---

## Claude for Synthetic Q/A Pair Generation

### Custom Generation Pipeline

For maximum control over question types and difficulty, use Claude directly:

```python
import anthropic
import json
from typing import Literal

client = anthropic.Anthropic()


def generate_qa_pairs(
    document: str,
    source: str,
    question_types: list[Literal[
        "factual", "procedural", "reasoning", "comparison",
        "unanswerable", "multi_hop"
    ]],
    num_per_type: int = 2,
) -> list[dict]:
    """Generate diverse Q/A pairs from a document using Claude."""
    type_instructions = {
        "factual": "Generate a straightforward factual question that can be answered with a specific fact from the document.",
        "procedural": "Generate a how-to question about a process described in the document.",
        "reasoning": "Generate a 'why' or 'when' question that requires understanding the reasoning behind concepts in the document.",
        "comparison": "Generate a question comparing two concepts mentioned in the document.",
        "unanswerable": "Generate a plausible-sounding question about the topic that CANNOT be answered from the document. The answer should indicate the information is not available.",
        "multi_hop": "Generate a question that requires combining information from different parts of the document.",
    }

    all_pairs = []
    for qtype in question_types:
        instruction = type_instructions[qtype]
        response = client.messages.create(
            model="claude-3-5-sonnet-20241022",
            max_tokens=1500,
            messages=[{
                "role": "user",
                "content": (
                    f"Document (source: {source}):\n{document[:3000]}\n\n"
                    f"Task: {instruction}\n\n"
                    f"Generate exactly {num_per_type} question-answer pairs.\n\n"
                    f"Format each as:\n"
                    f"Q: <question>\n"
                    f"A: <complete answer based only on the document>\n"
                    f"DIFFICULTY: <easy|medium|hard>\n\n"
                    f"Rules:\n"
                    f"- Answers must be based ONLY on information in the document\n"
                    f"- Questions should be natural (how a real user would ask)\n"
                    f"- Answers should be complete (2-4 sentences)\n"
                    f"- Vary the difficulty level\n"
                ),
            }],
        )

        # Parse response
        text = response.content[0].text
        pairs = parse_qa_pairs(text, qtype, source)
        all_pairs.extend(pairs)

    return all_pairs


def parse_qa_pairs(
    text: str, qtype: str, source: str
) -> list[dict]:
    """Parse Q/A pairs from LLM response."""
    pairs = []
    current_q = None
    current_a = None
    current_d = "medium"

    for line in text.strip().split("\n"):
        line = line.strip()
        if line.startswith("Q:"):
            if current_q and current_a:
                pairs.append({
                    "query": current_q,
                    "ground_truth": current_a,
                    "question_type": qtype,
                    "difficulty": current_d,
                    "source_document": source,
                    "synthetic": True,
                })
            current_q = line[2:].strip()
            current_a = None
            current_d = "medium"
        elif line.startswith("A:"):
            current_a = line[2:].strip()
        elif line.startswith("DIFFICULTY:"):
            current_d = line[11:].strip().lower()

    if current_q and current_a:
        pairs.append({
            "query": current_q,
            "ground_truth": current_a,
            "question_type": qtype,
            "difficulty": current_d,
            "source_document": source,
            "synthetic": True,
        })

    return pairs


# Generate from all documents
def generate_synthetic_eval_set(
    documents: list[dict],  # [{"content": str, "source": str}]
    questions_per_doc: int = 8,
) -> list[dict]:
    """Generate a full synthetic evaluation set from documents."""
    all_pairs = []
    for doc in documents:
        pairs = generate_qa_pairs(
            document=doc["content"],
            source=doc["source"],
            question_types=["factual", "procedural", "reasoning", "unanswerable"],
            num_per_type=questions_per_doc // 4,
        )
        all_pairs.extend(pairs)
    return all_pairs


# Usage
docs = [
    {"content": "PostgreSQL RLS uses CREATE POLICY...", "source": "pg-security.md"},
    {"content": "Kafka partitions handle message ordering...", "source": "kafka-config.md"},
]
synthetic_set = generate_synthetic_eval_set(docs)
print(f"Generated {len(synthetic_set)} synthetic Q/A pairs")
```

### Controlling Difficulty

```python
def generate_by_difficulty(
    document: str,
    source: str,
    difficulty: Literal["easy", "medium", "hard"],
    num_questions: int = 5,
) -> list[dict]:
    """Generate questions at a specific difficulty level."""
    difficulty_prompts = {
        "easy": (
            "Generate easy questions that can be answered by finding "
            "a single explicit fact in the document. The answer should "
            "be directly stated in one sentence of the document."
        ),
        "medium": (
            "Generate medium-difficulty questions that require "
            "combining 2-3 facts from the document or understanding "
            "the implications of what is stated."
        ),
        "hard": (
            "Generate hard questions that require deep understanding, "
            "inference beyond what is explicitly stated, or combining "
            "concepts from different parts of the document. Include "
            "questions that have nuanced answers."
        ),
    }

    response = client.messages.create(
        model="claude-3-5-sonnet-20241022",
        max_tokens=2000,
        messages=[{
            "role": "user",
            "content": (
                f"Document:\n{document[:3000]}\n\n"
                f"{difficulty_prompts[difficulty]}\n\n"
                f"Generate {num_questions} questions with answers.\n"
                f"Format: Q: <question>\\nA: <answer>\\n\\n"
            ),
        }],
    )

    return parse_qa_pairs(response.content[0].text, "mixed", source)
```

---

## Quality Filtering

Synthetic data has noise. Filter aggressively:

```python
def quality_filter_pipeline(
    qa_pairs: list[dict],
    knowledge_base_texts: list[str],
    embedding_model,
) -> list[dict]:
    """Multi-stage quality filtering for synthetic Q/A pairs."""

    # Stage 1: Basic length and format filters
    stage1 = []
    for pair in qa_pairs:
        q = pair["query"]
        a = pair["ground_truth"]

        if len(q) < 15 or len(q) > 500:
            continue
        if len(a) < 30 or len(a) > 2000:
            continue
        if "?" not in q and not q.strip().endswith("."):
            continue  # Not a question or statement
        if a.lower().startswith(("i don't know", "i cannot", "sorry")):
            if pair["question_type"] != "unanswerable":
                continue  # Evasive answer for an answerable question

        stage1.append(pair)

    print(f"Stage 1 (format): {len(qa_pairs)} -> {len(stage1)}")

    # Stage 2: Answerability check -- is the answer actually in the KB?
    kb_text = "\n\n".join(knowledge_base_texts)
    stage2 = []
    for pair in stage1:
        if pair["question_type"] == "unanswerable":
            stage2.append(pair)
            continue

        # Check if answer content appears in KB
        response = client.messages.create(
            model="claude-3-5-haiku-20241022",
            max_tokens=10,
            messages=[{
                "role": "user",
                "content": (
                    f"Can this answer be verified from the document?\n\n"
                    f"Document excerpt: {pair.get('source_content', kb_text[:2000])}\n\n"
                    f"Answer: {pair['ground_truth']}\n\n"
                    f"Respond: VERIFIABLE or NOT_VERIFIABLE"
                ),
            }],
        )
        if "VERIFIABLE" in response.content[0].text:
            stage2.append(pair)

    print(f"Stage 2 (answerability): {len(stage1)} -> {len(stage2)}")

    # Stage 3: Deduplication
    questions = [p["query"] for p in stage2]
    embeddings = embedding_model.embed_documents(questions)
    stage3 = []
    seen_indices = set()

    for i in range(len(stage2)):
        if i in seen_indices:
            continue
        stage3.append(stage2[i])
        # Mark near-duplicates
        for j in range(i + 1, len(stage2)):
            if j in seen_indices:
                continue
            import numpy as np
            sim = np.dot(embeddings[i], embeddings[j]) / (
                np.linalg.norm(embeddings[i]) * np.linalg.norm(embeddings[j])
            )
            if sim > 0.9:  # Near-duplicate
                seen_indices.add(j)

    print(f"Stage 3 (dedup): {len(stage2)} -> {len(stage3)}")
    return stage3
```

---

## The Bootstrap-Then-Refine Workflow

The recommended approach for building evaluation datasets:

### Phase 1: Bootstrap with Synthetic Data (Day 1)

```python
# Generate 200 synthetic Q/A pairs from your KB
synthetic_set = generate_synthetic_eval_set(documents, questions_per_doc=10)

# Quality filter to ~120
filtered_set = quality_filter_pipeline(synthetic_set, kb_texts, embed_model)

# Use this for rapid pipeline iteration
baseline_results = evaluate_pipeline(pipeline, filtered_set)
```

### Phase 2: Identify Gaps and Failures (Day 2-3)

```python
# Find queries where the pipeline fails
failures = [
    item for item, result in zip(filtered_set, per_query_results)
    if result["faithfulness"] < 0.5 or result["answer_relevancy"] < 0.5
]

# Categorize failure modes
# - Missing context: retriever did not find relevant docs
# - Wrong context: retriever found irrelevant docs
# - Hallucination: LLM added unsupported claims
# - Off-topic: LLM answered a different question
```

### Phase 3: Manual Annotation on Gaps (Day 3-5)

```python
# Manually annotate 50-80 queries focusing on:
# 1. Real production queries (if available)
# 2. Edge cases identified in Phase 2
# 3. Unanswerable queries (synthetic generators rarely produce these well)
# 4. Complex multi-hop queries (synthetic are too simple)

manual_set = annotate_manually(
    production_queries=sample_production_queries(30),
    edge_case_queries=generate_edge_cases(20),
    unanswerable_queries=create_unanswerable(10),
    multi_hop_queries=create_multi_hop(15),
)
```

### Phase 4: Merge and Split (Day 5)

```python
# Combine synthetic (filtered) + manual
full_eval_set = merge_datasets(filtered_set, manual_set)

# Split into tuning and holdout
tuning_set, holdout_set = stratified_split(full_eval_set, holdout_ratio=0.7)

# Tag synthetic vs manual for analysis
for item in full_eval_set:
    item["data_source"] = "synthetic" if item.get("synthetic") else "manual"
```

---

## Benchmarks: Synthetic vs. Manual Evaluation Quality

| Aspect | Synthetic Only | Manual Only | Bootstrap + Refine |
|--------|---------------|-------------|-------------------|
| Coverage of query types | Moderate (biased toward factual) | High (controlled) | High |
| Effort to create | Minutes | Days | 1 day synthetic + 2-3 days manual |
| Reflects real users | Low | Medium-high | Medium-high |
| Tests edge cases | Low | High | High |
| Sufficient for A/B testing | Yes (directional) | Yes (precise) | Yes (precise) |
| Maintenance burden | Low (regenerate) | High (re-annotate) | Medium |

---

## Common Pitfalls

1. **Using synthetic data as the sole evaluation.** Synthetic questions are "clean" -- they do not capture typos, ambiguity, implicit assumptions, or domain jargon that real users produce. Always supplement with manual annotation or production traffic.
2. **Not filtering synthetic outputs.** Raw synthetic data includes malformed questions, overly simple queries, and factually incorrect ground truths. Apply the quality filtering pipeline.
3. **Generating from the same LLM used for RAG.** If you generate synthetic questions with Claude and your RAG pipeline uses Claude for generation, the evaluation is biased -- the generator "knows" how the synthetic questions were constructed. Use a different model for generation if possible.
4. **Ignoring unanswerable queries.** Synthetic generators almost never produce unanswerable queries (they always generate questions the document can answer). Manually add unanswerable examples.
5. **Too many easy questions.** Synthetic generators default to simple factual extraction. Explicitly request reasoning, comparison, and multi-hop questions to test the pipeline's limits.
6. **Not versioning synthetic datasets.** Even though synthetic data is "cheap" to regenerate, version it. Score improvements from pipeline changes must be measured on the same dataset to be meaningful.

---

## References

- RAGAS TestsetGenerator: https://docs.ragas.io/en/stable/concepts/testset_generation/
- Giskard RAGET: https://docs.giskard.ai/en/stable/open_source/testset_generation/
- LlamaIndex evaluation: https://docs.llamaindex.ai/en/stable/module_guides/evaluating/
- Alberti et al. "Synthetic QA Corpora Generation with Roundtrip Consistency" (ACL 2019)
