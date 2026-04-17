# RAGAS Metrics Deep Dive

## TL;DR

RAGAS provides four core metrics for RAG evaluation: faithfulness (is the answer grounded in context?), answer relevancy (does it address the query?), context precision (are relevant docs ranked high?), and context recall (were all relevant docs retrieved?). This article dissects each metric's internal algorithm, provides mathematical formulas, shows full runnable code with real datasets, explains score interpretation, and identifies cases where RAGAS metrics can be misleading.

---

## Metric 1: Faithfulness

### What It Measures

Faithfulness quantifies whether the generated answer is factually grounded in the retrieved context. An answer with faithfulness=1.0 means every claim can be traced to the provided context. An answer with faithfulness=0.5 means half the claims are hallucinated or from parametric knowledge.

### Algorithm (Step by Step)

1. **Claim decomposition**: break the answer into atomic claims/statements
2. **NLI verification**: for each claim, determine if it is supported by the context using Natural Language Inference
3. **Score**: faithfulness = number of supported claims / total claims

### Mathematical Formula

```
faithfulness = |{c in claims(answer) : NLI(context, c) = "entailed"}| / |claims(answer)|
```

Where:
- `claims(answer)` = set of atomic factual statements extracted from the answer
- `NLI(context, c)` = Natural Language Inference check (entailed, contradicted, or neutral)
- Only "entailed" claims count as supported

### Internal Implementation Detail

RAGAS uses an LLM (default: GPT-4 or Claude) for both claim decomposition and NLI verification. The prompts are approximately:

**Claim decomposition prompt**:
```
Given the following answer, break it into individual factual claims.
Each claim should be a single sentence that states one fact.

Answer: {answer}

Claims:
```

**NLI verification prompt**:
```
Given the context and a claim, determine if the claim is:
- "supported" if the context provides evidence for the claim
- "not supported" if the context does not provide evidence

Context: {context}
Claim: {claim}
Verdict:
```

### Full Code Example

```python
from ragas import evaluate
from ragas.metrics import faithfulness
from datasets import Dataset
import pandas as pd


# Prepare a realistic evaluation dataset
eval_data = {
    "question": [
        "What are the benefits of connection pooling in PostgreSQL?",
        "How does Kafka handle message ordering?",
        "What is the default isolation level in PostgreSQL?",
    ],
    "answer": [
        # High faithfulness: all claims from context
        "Connection pooling in PostgreSQL reduces connection overhead by reusing "
        "existing connections. Tools like PgBouncer can handle thousands of "
        "concurrent connections while maintaining only a small pool of actual "
        "database connections.",

        # Medium faithfulness: some claims from context, some from parametric knowledge
        "Kafka guarantees message ordering within a single partition. Messages "
        "are assigned an offset that monotonically increases. Kafka also uses "
        "the Raft consensus protocol for leader election.",
        # Note: Raft claim may not be in context (KRaft is recent, context may say ZooKeeper)

        # Low faithfulness: answer adds information not in context
        "PostgreSQL uses Read Committed as the default isolation level. It also "
        "supports Serializable isolation which prevents all anomalies. "
        "PostgreSQL is faster than MySQL for complex analytical queries.",
        # "faster than MySQL" is not in the context
    ],
    "contexts": [
        [
            "PostgreSQL connection pooling reduces the overhead of establishing "
            "new connections. PgBouncer is a popular pooler that maintains a small "
            "pool of database connections while serving thousands of client connections."
        ],
        [
            "Apache Kafka guarantees ordering of messages within a partition. "
            "Each message in a partition is assigned a sequential offset number. "
            "Kafka traditionally uses ZooKeeper for cluster coordination."
        ],
        [
            "PostgreSQL's default transaction isolation level is Read Committed. "
            "PostgreSQL supports four isolation levels: Read Uncommitted, Read Committed, "
            "Repeatable Read, and Serializable."
        ],
    ],
}

dataset = Dataset.from_dict(eval_data)

# Run evaluation
result = evaluate(
    dataset,
    metrics=[faithfulness],
)

# Print per-query results
df = result.to_pandas()
for _, row in df.iterrows():
    print(f"Q: {row['question'][:60]}...")
    print(f"  Faithfulness: {row['faithfulness']:.4f}")
    print()

# Expected approximate scores:
# Q1: ~1.0 (all claims supported)
# Q2: ~0.67 (2/3 claims supported, Raft claim not in context)
# Q3: ~0.67 (2/3 claims supported, MySQL comparison not in context)
```

### Interpreting Faithfulness Scores

| Score Range | Interpretation | Action |
|------------|---------------|--------|
| 0.90-1.00 | Excellent grounding | System is working well |
| 0.75-0.90 | Good, minor hallucinations | Acceptable for most use cases |
| 0.50-0.75 | Significant hallucination | Improve grounding prompt, add verification |
| <0.50 | Severe hallucination | Fundamental pipeline issue |

### When Faithfulness Misleads

1. **Verbatim copying scores 1.0 but is not useful**: an answer that copies context verbatim without synthesizing gets perfect faithfulness but low relevancy/helpfulness.
2. **Correct parametric knowledge penalized**: if the LLM adds a correct fact from its training data that is not in the context, faithfulness drops even though the answer is correct.
3. **Claim decomposition errors**: the LLM may merge two claims or split incorrectly, affecting the denominator.
4. **Context too short**: if context is a single sentence, the bar for "supported" is very narrow.

---

## Metric 2: Answer Relevancy

### What It Measures

Answer relevancy measures whether the generated answer actually addresses the user's question. High relevancy means the answer is on-topic and responsive; low relevancy means it is off-topic or generic.

### Algorithm

1. **Reverse question generation**: given the answer, generate N questions that the answer could be responding to
2. **Similarity**: compute embedding similarity between each generated question and the original question
3. **Score**: average similarity across generated questions

### Mathematical Formula

```
answer_relevancy = (1/N) * SUM_{i=1}^{N} cosine(embed(q_original), embed(q_generated_i))
```

Where:
- `q_generated_i` = question generated from the answer
- `N` = number of generated questions (default: 3)
- `embed()` = embedding model

### Intuition

If the answer is truly relevant to the question, then questions reverse-engineered from the answer should be similar to the original question. If the answer is off-topic, the reverse questions will be different.

**Example**:
```
Original query: "How does PostgreSQL handle connection pooling?"
Answer: "PgBouncer manages a pool of database connections..."

Generated questions from the answer:
  Q1: "What tool manages PostgreSQL connection pools?"
  Q2: "How does PgBouncer handle database connections?"
  Q3: "What is PgBouncer used for?"

cosine(original, Q1) = 0.89
cosine(original, Q2) = 0.92
cosine(original, Q3) = 0.78

Answer relevancy = (0.89 + 0.92 + 0.78) / 3 = 0.863
```

### Code

```python
from ragas.metrics import answer_relevancy

result = evaluate(dataset, metrics=[answer_relevancy])
df = result.to_pandas()

for _, row in df.iterrows():
    print(f"Q: {row['question'][:60]}...")
    print(f"  Answer Relevancy: {row['answer_relevancy']:.4f}")
```

### When Answer Relevancy Misleads

1. **Overly generic answers score moderately**: "That is a great question about databases" -- generic enough to generate similar-ish questions, but not actually useful.
2. **Partial answers score well**: if the answer addresses one part of a multi-part question, the reverse questions still relate to the original.
3. **Embedding model quality matters**: weak embedding models produce noisy similarity scores.

---

## Metric 3: Context Precision

### What It Measures

Context precision evaluates whether the relevant documents in the retrieved context are ranked higher than irrelevant ones. High precision means the retriever puts the good documents first.

### Algorithm

1. For each retrieved document, determine if it is relevant to the ground truth answer
2. Compute precision at each position K where a relevant document appears
3. Average these precisions (Average Precision)

### Mathematical Formula

```
context_precision = (1/|relevant|) * SUM_{k=1}^{K} (precision@k * is_relevant(k))

where:
  precision@k = (relevant docs in top k) / k
  is_relevant(k) = 1 if document at position k is relevant, 0 otherwise
```

This is essentially Average Precision (AP), a standard IR metric.

### Example

```
Retrieved context (5 documents):
  Position 1: "PostgreSQL RLS uses CREATE POLICY..." [RELEVANT]
  Position 2: "PostgreSQL supports multiple index types" [IRRELEVANT]
  Position 3: "Row Level Security filters rows by user" [RELEVANT]
  Position 4: "PostgreSQL has excellent JSON support"   [IRRELEVANT]
  Position 5: "CREATE POLICY syntax for RLS..."         [RELEVANT]

Precision at relevant positions:
  Position 1: precision@1 = 1/1 = 1.0
  Position 3: precision@3 = 2/3 = 0.667
  Position 5: precision@5 = 3/5 = 0.6

Context Precision = (1/3) * (1.0 + 0.667 + 0.6) = 0.756
```

### Code

```python
from ragas.metrics import context_precision

# Requires ground_truth
eval_data_with_gt = {
    "question": ["How does PostgreSQL RLS work?"],
    "answer": ["PostgreSQL RLS uses CREATE POLICY..."],
    "contexts": [[
        "PostgreSQL RLS uses CREATE POLICY statements for row-level access.",
        "PostgreSQL supports GIN, GiST, and B-tree indexes.",
        "Row Level Security in PostgreSQL filters rows based on current user.",
    ]],
    "ground_truth": [
        "PostgreSQL Row Level Security uses CREATE POLICY to define "
        "per-row access control that filters based on the current user."
    ],
}

dataset = Dataset.from_dict(eval_data_with_gt)
result = evaluate(dataset, metrics=[context_precision])
print(f"Context Precision: {result['context_precision']:.4f}")
```

---

## Metric 4: Context Recall

### What It Measures

Context recall measures whether all the information needed to answer the question was present in the retrieved context. High recall means the retriever found everything; low recall means important information was missed.

### Algorithm

1. Decompose the ground truth answer into individual statements
2. For each statement, check if it can be attributed to any document in the context
3. Context recall = attributable statements / total statements

### Mathematical Formula

```
context_recall = |{s in statements(ground_truth) : attributable(s, context)}| / |statements(ground_truth)|
```

### Example

```
Ground truth: "PostgreSQL RLS uses CREATE POLICY to define row access.
  Policies can be PERMISSIVE or RESTRICTIVE. RLS must be enabled with
  ALTER TABLE ... ENABLE ROW LEVEL SECURITY."

Statements:
  S1: "PostgreSQL RLS uses CREATE POLICY" -> Found in context -> attributable
  S2: "Policies can be PERMISSIVE or RESTRICTIVE" -> Not in context -> NOT attributable
  S3: "RLS must be enabled with ALTER TABLE" -> Found in context -> attributable

Context Recall = 2/3 = 0.667
```

### Code

```python
from ragas.metrics import context_recall

result = evaluate(dataset, metrics=[context_recall])
print(f"Context Recall: {result['context_recall']:.4f}")
```

---

## Running All Metrics Together

### Complete Evaluation Script

```python
from ragas import evaluate
from ragas.metrics import (
    faithfulness,
    answer_relevancy,
    context_precision,
    context_recall,
)
from ragas.llms import LangchainLLMWrapper
from ragas.embeddings import LangchainEmbeddingsWrapper
from langchain_anthropic import ChatAnthropic
from langchain_openai import OpenAIEmbeddings
from datasets import Dataset
import json


def load_eval_dataset(path: str) -> Dataset:
    """Load evaluation dataset from JSON file."""
    with open(path) as f:
        data = json.load(f)

    return Dataset.from_dict({
        "question": [d["question"] for d in data],
        "answer": [d["answer"] for d in data],
        "contexts": [d["contexts"] for d in data],
        "ground_truth": [d["ground_truth"] for d in data],
    })


def run_evaluation(
    dataset: Dataset,
    llm_model: str = "claude-3-5-sonnet-20241022",
    embedding_model: str = "text-embedding-3-small",
) -> dict:
    """Run full RAGAS evaluation with specified models."""

    # Configure RAGAS to use Claude for evaluation
    evaluator_llm = LangchainLLMWrapper(ChatAnthropic(model=llm_model))
    evaluator_embeddings = LangchainEmbeddingsWrapper(
        OpenAIEmbeddings(model=embedding_model)
    )

    metrics = [faithfulness, answer_relevancy, context_precision, context_recall]

    result = evaluate(
        dataset,
        metrics=metrics,
        llm=evaluator_llm,
        embeddings=evaluator_embeddings,
    )

    # Summary
    print("\n" + "=" * 50)
    print("RAGAS EVALUATION SUMMARY")
    print("=" * 50)
    for metric_name in ["faithfulness", "answer_relevancy", "context_precision", "context_recall"]:
        score = result[metric_name]
        status = "GOOD" if score >= 0.75 else "OK" if score >= 0.50 else "POOR"
        print(f"  {metric_name:25s}: {score:.4f}  [{status}]")

    # Per-query analysis
    df = result.to_pandas()
    print(f"\nPer-query results ({len(df)} queries):")

    # Find worst-performing queries
    for metric in ["faithfulness", "answer_relevancy"]:
        worst = df.nsmallest(3, metric)
        print(f"\n  Worst {metric}:")
        for _, row in worst.iterrows():
            print(f"    {row['question'][:60]}... = {row[metric]:.4f}")

    return result


# Usage
dataset = load_eval_dataset("eval_dataset.json")
result = run_evaluation(dataset)
```

---

## Diagnosing Score Patterns

### Pattern: High Faithfulness + Low Relevancy

The answer is grounded in context but does not address the question.

**Common cause**: the retriever found related but off-topic documents, and the LLM faithfully answered based on them.

**Fix**: improve retrieval (better query, hybrid search, reranking) rather than generation.

### Pattern: Low Faithfulness + High Relevancy

The answer addresses the question but adds information not in the context.

**Common cause**: the LLM uses parametric knowledge to supplement retrieved context. Often correct but unverifiable.

**Fix**: strengthen grounding instructions in the prompt: "Answer ONLY based on the provided context."

### Pattern: Low Context Precision + High Context Recall

All relevant docs were retrieved, but they are buried among irrelevant ones.

**Common cause**: retriever has high recall but no ranking quality. Too many documents without reranking.

**Fix**: add a reranker (cross-encoder) or reduce K.

### Pattern: High Context Precision + Low Context Recall

Retrieved docs are relevant but incomplete -- some needed information is missing.

**Common cause**: knowledge base gaps or chunking that splits relevant information across non-adjacent chunks.

**Fix**: improve chunking (larger chunks with overlap), add more documents to the knowledge base, or use multi-step retrieval.

---

## RAGAS Limitations and When It Misleads

### 1. LLM Evaluator Bias

RAGAS uses an LLM internally for claim decomposition and NLI. This means:
- The evaluation quality depends on the evaluator LLM's capability
- Different evaluator LLMs produce different scores for the same answer
- The evaluator may have the same biases as the generator (especially if both are the same model)

**Mitigation**: use a different model family for evaluation than for generation (e.g., evaluate with Claude if generating with GPT-4, or vice versa).

### 2. Claim Decomposition Artifacts

The claim decomposition step can produce:
- Overly fine-grained claims (splitting "PostgreSQL RLS filters rows based on the current user" into 3 claims)
- Merged claims that should be separate
- Tautological claims ("PostgreSQL is a database") that inflate the denominator

### 3. Context Window Truncation

If the combined context exceeds the evaluator LLM's context window, RAGAS truncates. This can cause faithfulness scores to drop because the evaluator does not see the full context.

### 4. Score Instability

RAGAS scores can vary by 0.05-0.10 across runs due to LLM stochasticity. Always:
- Run evaluation 3 times and average
- Or set temperature=0 in the evaluator LLM
- Report confidence intervals, not point estimates

### 5. No Distinction Between Types of Unfaithfulness

RAGAS treats all unsupported claims equally. But there is a big difference between:
- Adding a correct fact from parametric knowledge (benign)
- Contradicting the context (dangerous)
- Fabricating a plausible-sounding but wrong claim (dangerous)

---

## Common Pitfalls

1. **Comparing RAGAS scores across different evaluator LLMs.** A faithfulness of 0.85 with GPT-4-as-evaluator is not comparable to 0.85 with Claude-as-evaluator. Keep the evaluator constant across comparisons.
2. **Not providing ground_truth for context metrics.** Context precision and context recall require ground truth answers. Without them, RAGAS cannot determine which retrieved documents are relevant.
3. **Evaluating with the same model used for generation.** Self-evaluation bias: the model may judge its own output more favorably. Use a different model family for evaluation.
4. **Treating 0.85 as universally "good".** Score thresholds depend on the domain and use case. In medical/legal contexts, faithfulness below 0.95 may be unacceptable. In casual Q&A, 0.75 may suffice.
5. **Ignoring per-query variance.** An average faithfulness of 0.80 could mean 80% of queries score 1.0 and 20% score 0.0 -- the latter is a serious problem hidden by the average. Always examine the distribution.

---

## References

- Es et al. "RAGAS: Automated Evaluation of Retrieval Augmented Generation" (2023). https://arxiv.org/abs/2309.15217
- RAGAS documentation: https://docs.ragas.io/
- RAGAS GitHub: https://github.com/explodinggradients/ragas
- Zheng et al. "Judging LLM-as-a-Judge with MT-Bench and Chatbot Arena" (NeurIPS 2023)
