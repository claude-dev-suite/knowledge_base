# Golden Evaluation Dataset Construction

## TL;DR

A golden evaluation dataset is a manually curated set of (query, expected_answer, relevant_documents) triples used to measure RAG pipeline quality. Without one, you are optimizing blind. This article covers annotation guidelines, inter-annotator agreement, minimum set sizes, question taxonomy, format schema, version control, and augmentation strategies. The minimum viable golden set is 50 queries; 100-200 is recommended for statistically significant comparisons between pipeline configurations.

---

## Why You Need a Golden Set

### The Problem with Vibes-Based Evaluation

Without a golden set, teams evaluate RAG quality by:
1. Trying a few queries manually and checking if the answer "looks right"
2. Asking teammates if the output seems good
3. Relying on automated metrics without ground truth (meaningless for context precision/recall)

This fails because:
- Small samples are not representative of the query distribution
- Subjective assessment is inconsistent across evaluators
- Changes that improve one query type may degrade another (undetected)
- There is no way to measure regression when the pipeline changes

### What a Golden Set Enables

- **Automated regression testing**: run after every pipeline change
- **A/B testing**: statistically compare two configurations
- **Root cause analysis**: identify which query types are failing
- **Benchmarking**: track quality over time with a consistent baseline

---

## Minimum Set Size

### Statistical Power Analysis

To detect a 5% difference in faithfulness between two pipeline configurations with 80% statistical power:

```python
import math

def min_sample_size(
    effect_size: float = 0.05,
    alpha: float = 0.05,
    power: float = 0.80,
    variance: float = 0.04,
) -> int:
    """Minimum sample size for paired t-test."""
    from scipy.stats import norm
    z_alpha = norm.ppf(1 - alpha / 2)
    z_beta = norm.ppf(power)
    n = ((z_alpha + z_beta) ** 2 * variance) / (effect_size ** 2)
    return math.ceil(n)

# Typical RAG evaluation variance
n = min_sample_size(effect_size=0.05, variance=0.04)
# n ≈ 62 queries
```

### Practical Guidelines

| Set Size | What It Supports | When to Use |
|----------|-----------------|-------------|
| 20-30 | Quick sanity check, no stat significance | Rapid prototyping |
| 50-75 | Detect large differences (>8% metric change) | MVP evaluation |
| 100-150 | Detect moderate differences (>5%) | Production evaluation |
| 200-500 | Detect small differences (>2-3%), per-type analysis | Mature systems |
| 500+ | Detailed per-type, per-topic analysis | Enterprise/research |

**Recommendation**: start with 50, grow to 100-200 as your system matures.

---

## Question Taxonomy

### Why Taxonomy Matters

Different question types stress different parts of the pipeline:

| Question Type | Tests | Example |
|--------------|-------|---------|
| Factual | Basic retrieval + extraction | "What is the default port for PostgreSQL?" |
| Procedural | Multi-step retrieval + instruction following | "How to configure RLS in PostgreSQL?" |
| Reasoning | Retrieval + inference/analysis | "Why does connection pooling improve performance?" |
| Comparison | Multi-document retrieval + synthesis | "What are the differences between RLS and RBAC?" |
| Unanswerable | Retrieval failure handling | "What is PostgreSQL's built-in support for GraphQL?" |
| Ambiguous | Query understanding + clarification | "How to set up security?" (which kind?) |
| Multi-hop | Iterative retrieval + chaining | "Which index type is best for the query pattern used in RLS policies?" |
| Temporal | Time-aware retrieval | "What changed in PostgreSQL 17 regarding RLS?" |

### Recommended Distribution

For a 100-query golden set:

```python
taxonomy_distribution = {
    "factual": 25,        # 25% - Baseline
    "procedural": 20,     # 20% - Common in technical docs
    "reasoning": 15,      # 15% - Tests comprehension
    "comparison": 10,     # 10% - Tests multi-doc synthesis
    "unanswerable": 10,   # 10% - Tests graceful failure
    "ambiguous": 5,       # 5%  - Tests query understanding
    "multi_hop": 10,      # 10% - Tests complex retrieval
    "temporal": 5,        # 5%  - Tests time awareness
}
```

### Difficulty Levels

Within each type, vary difficulty:

```python
difficulty_distribution = {
    "easy": 30,     # 30% - Single document, obvious answer
    "medium": 45,   # 45% - Requires 2-3 documents or synthesis
    "hard": 25,     # 25% - Requires reasoning, multiple hops, or domain expertise
}
```

---

## Annotation Guidelines

### For Annotators

Provide annotators with clear, written guidelines:

```markdown
# RAG Evaluation Annotation Guidelines

## Task
For each query, provide:
1. The expected answer (ground truth)
2. The minimum set of documents needed to answer
3. The question type (factual/procedural/reasoning/comparison/unanswerable)
4. The difficulty level (easy/medium/hard)

## Ground Truth Answer Rules
- Write the answer as a COMPLETE response, not just keywords
- Include all relevant facts that should appear in a good answer
- Do NOT include information that is not in the knowledge base
- For unanswerable queries, write: "This information is not available
  in the knowledge base."
- Be specific: "PostgreSQL uses port 5432 by default" not "the default
  port is a well-known number"

## Relevant Documents
- List ALL documents that contain information needed for the answer
- Include partial matches (document covers topic but not the specific fact)
- Mark each as "essential" (must retrieve) or "helpful" (improves answer)

## Quality Criteria
- Would a domain expert agree with this ground truth?
- Is the answer complete enough to evaluate against?
- Are the relevant documents correctly identified?
```

### Example Annotation

```json
{
    "id": "pg-rls-001",
    "query": "How do I enable Row Level Security on a PostgreSQL table?",
    "question_type": "procedural",
    "difficulty": "easy",
    "ground_truth": "To enable Row Level Security on a PostgreSQL table: 1) Run ALTER TABLE tablename ENABLE ROW LEVEL SECURITY to activate RLS on the table. 2) Create policies using CREATE POLICY to define access rules. Without policies, RLS blocks all access by default (except for table owners who bypass RLS). 3) Optionally, use ALTER TABLE tablename FORCE ROW LEVEL SECURITY to apply policies even to the table owner.",
    "relevant_documents": [
        {
            "doc_id": "pg-security-003",
            "title": "PostgreSQL Row Level Security",
            "relevance": "essential",
            "relevant_section": "Enabling RLS"
        },
        {
            "doc_id": "pg-security-004",
            "title": "PostgreSQL CREATE POLICY Reference",
            "relevance": "helpful",
            "relevant_section": "Policy syntax"
        }
    ],
    "annotator": "expert_1",
    "annotation_date": "2025-01-15",
    "notes": ""
}
```

---

## Inter-Annotator Agreement (IAA)

### Why IAA Matters

If two annotators write different ground truth answers for the same query, your evaluation is unreliable. Measure IAA to ensure annotation quality.

### Measuring IAA for RAG Ground Truth

```python
from sklearn.metrics import cohen_kappa_score
import numpy as np


def measure_iaa_binary(
    annotator_a: list[dict],
    annotator_b: list[dict],
) -> float:
    """Measure IAA for relevant document identification (binary: relevant/not)."""
    all_docs = set()
    for a in annotator_a:
        all_docs.update(d["doc_id"] for d in a["relevant_documents"])
    for b in annotator_b:
        all_docs.update(d["doc_id"] for d in b["relevant_documents"])

    labels_a = []
    labels_b = []
    for query_idx in range(len(annotator_a)):
        a_docs = {d["doc_id"] for d in annotator_a[query_idx]["relevant_documents"]}
        b_docs = {d["doc_id"] for d in annotator_b[query_idx]["relevant_documents"]}
        for doc_id in all_docs:
            labels_a.append(1 if doc_id in a_docs else 0)
            labels_b.append(1 if doc_id in b_docs else 0)

    kappa = cohen_kappa_score(labels_a, labels_b)
    return kappa


def measure_iaa_answer_similarity(
    answers_a: list[str],
    answers_b: list[str],
    embedding_model,
) -> float:
    """Measure IAA for ground truth answers via embedding similarity."""
    embeddings_a = embedding_model.embed_documents(answers_a)
    embeddings_b = embedding_model.embed_documents(answers_b)

    similarities = []
    for ea, eb in zip(embeddings_a, embeddings_b):
        cos_sim = np.dot(ea, eb) / (np.linalg.norm(ea) * np.linalg.norm(eb))
        similarities.append(cos_sim)

    return np.mean(similarities)


# Interpretation:
# Kappa > 0.8: Almost perfect agreement
# Kappa 0.6-0.8: Substantial agreement (acceptable)
# Kappa 0.4-0.6: Moderate agreement (needs guideline refinement)
# Kappa < 0.4: Poor agreement (rewrite guidelines, retrain annotators)
```

### IAA Process

1. Have 2+ annotators independently annotate the same 20-30 queries
2. Compute IAA (Cohen's Kappa for document relevance, cosine similarity for answers)
3. If Kappa < 0.6, review disagreements, refine guidelines, re-annotate
4. Once IAA > 0.6, split the remaining queries among annotators
5. Have 10% of all queries double-annotated as ongoing quality checks

---

## Dataset Format Schema

### JSON Schema

```json
{
    "$schema": "http://json-schema.org/draft-07/schema#",
    "type": "array",
    "items": {
        "type": "object",
        "required": ["id", "query", "ground_truth", "question_type", "difficulty"],
        "properties": {
            "id": {
                "type": "string",
                "description": "Unique identifier for this evaluation item"
            },
            "query": {
                "type": "string",
                "description": "The user's query"
            },
            "ground_truth": {
                "type": "string",
                "description": "Expected answer (complete, specific)"
            },
            "question_type": {
                "type": "string",
                "enum": ["factual", "procedural", "reasoning", "comparison",
                         "unanswerable", "ambiguous", "multi_hop", "temporal"]
            },
            "difficulty": {
                "type": "string",
                "enum": ["easy", "medium", "hard"]
            },
            "relevant_documents": {
                "type": "array",
                "items": {
                    "type": "object",
                    "properties": {
                        "doc_id": {"type": "string"},
                        "title": {"type": "string"},
                        "relevance": {"type": "string", "enum": ["essential", "helpful"]},
                        "relevant_section": {"type": "string"}
                    }
                }
            },
            "tags": {
                "type": "array",
                "items": {"type": "string"},
                "description": "Topic tags for filtering (e.g., 'postgresql', 'security')"
            },
            "annotator": {"type": "string"},
            "annotation_date": {"type": "string", "format": "date"},
            "notes": {"type": "string"}
        }
    }
}
```

### File Organization

```
eval/
  golden_set_v1.json              # Main golden set
  golden_set_v1_schema.json       # JSON schema
  splits/
    tuning.json                   # 30% for hyperparameter tuning
    holdout.json                  # 70% for final evaluation (never tune on this)
  annotations/
    annotator_a_batch1.json       # Raw annotations before reconciliation
    annotator_b_batch1.json
    iaa_report_batch1.json        # Inter-annotator agreement analysis
  changelog.md                    # Changes to the golden set over time
```

---

## Version Control for Golden Sets

### Why Version Control

The golden set evolves as:
- New query types emerge from production traffic
- Knowledge base content changes (new docs, removed docs)
- Annotators find and fix errors
- Coverage gaps are identified and filled

### Versioning Strategy

```python
# golden_set_metadata.json
{
    "version": "1.3.0",
    "created_date": "2025-01-01",
    "last_updated": "2025-03-15",
    "total_queries": 150,
    "annotators": ["expert_1", "expert_2"],
    "iaa_kappa": 0.78,
    "changelog": [
        {
            "version": "1.3.0",
            "date": "2025-03-15",
            "changes": "Added 25 temporal queries for PostgreSQL 17 features"
        },
        {
            "version": "1.2.0",
            "date": "2025-02-01",
            "changes": "Added unanswerable queries, fixed 8 ground truth errors"
        },
        {
            "version": "1.1.0",
            "date": "2025-01-15",
            "changes": "Added comparison and multi-hop query types"
        },
        {
            "version": "1.0.0",
            "date": "2025-01-01",
            "changes": "Initial golden set with 100 factual and procedural queries"
        }
    ]
}
```

### Rules for Updating

1. **Never modify existing entries silently.** Track all changes in the changelog.
2. **Increment version**: new queries = minor bump, error fixes = patch bump.
3. **Re-run baseline evaluation** after updating the golden set to establish a new baseline.
4. **Keep old versions** to compare evaluations across golden set versions.

---

## Augmentation Strategies

### Strategy 1: Query Paraphrasing

Generate paraphrased versions of existing queries to test retrieval robustness:

```python
def generate_paraphrases(
    query: str,
    n: int = 3,
) -> list[str]:
    """Generate paraphrased versions of a query."""
    response = client.messages.create(
        model="claude-3-5-sonnet-20241022",
        max_tokens=300,
        messages=[{
            "role": "user",
            "content": (
                f"Generate {n} paraphrased versions of this query. "
                f"Each should ask the same thing differently.\n\n"
                f"Original: {query}\n\n"
                f"Paraphrases (one per line):"
            ),
        }],
    )
    return [
        line.strip().lstrip("0123456789.) ")
        for line in response.content[0].text.strip().split("\n")
        if line.strip()
    ][:n]


# Usage: for each golden query, generate paraphrases that should
# produce the same answer
original = "How to configure RLS in PostgreSQL?"
paraphrases = generate_paraphrases(original)
# ["What steps are needed to set up Row Level Security in PostgreSQL?",
#  "PostgreSQL RLS configuration guide",
#  "Enable and configure row-level security on a PostgreSQL table"]
```

### Strategy 2: Adversarial Queries

Generate queries designed to trip up the RAG system:

```python
def generate_adversarial_queries(
    topic: str,
    knowledge_base_topics: list[str],
    n: int = 5,
) -> list[dict]:
    """Generate adversarial test queries."""
    response = client.messages.create(
        model="claude-3-5-sonnet-20241022",
        max_tokens=500,
        messages=[{
            "role": "user",
            "content": (
                f"Generate {n} tricky test queries about '{topic}' that would "
                f"challenge a RAG system. Include:\n"
                f"- Queries with negation ('without', 'not', 'except')\n"
                f"- Queries about things NOT in the knowledge base\n"
                f"- Queries that sound similar to {topic} but are actually different\n"
                f"- Queries requiring information from multiple documents\n\n"
                f"Knowledge base covers: {', '.join(knowledge_base_topics)}\n\n"
                f"Format: query | type | expected_behavior"
            ),
        }],
    )
    # Parse and return structured adversarial queries
    queries = []
    for line in response.content[0].text.strip().split("\n"):
        parts = line.split("|")
        if len(parts) >= 2:
            queries.append({
                "query": parts[0].strip(),
                "adversarial_type": parts[1].strip() if len(parts) > 1 else "unknown",
            })
    return queries
```

### Strategy 3: Production Traffic Sampling

Mine real queries from production logs to find gaps in the golden set:

```python
def identify_coverage_gaps(
    golden_set: list[dict],
    production_queries: list[str],
    embedding_model,
    similarity_threshold: float = 0.8,
) -> list[str]:
    """Find production queries not covered by the golden set."""
    golden_queries = [item["query"] for item in golden_set]
    golden_embeddings = embedding_model.embed_documents(golden_queries)
    prod_embeddings = embedding_model.embed_documents(production_queries)

    uncovered = []
    for i, prod_emb in enumerate(prod_embeddings):
        max_similarity = max(
            np.dot(prod_emb, g_emb) / (np.linalg.norm(prod_emb) * np.linalg.norm(g_emb))
            for g_emb in golden_embeddings
        )
        if max_similarity < similarity_threshold:
            uncovered.append(production_queries[i])

    return uncovered
```

---

## Building Your First Golden Set: Step-by-Step

### Step 1: Sample Queries (Day 1)

```python
# If you have production logs
queries_from_logs = sample_production_queries(n=200)

# If you don't have production logs
# Generate representative queries based on your knowledge base topics
topics = ["PostgreSQL security", "Kafka configuration", "Docker networking"]
queries = []
for topic in topics:
    queries.extend(generate_queries_for_topic(topic, n=20))
```

### Step 2: Deduplicate and Categorize (Day 1-2)

```python
# Remove near-duplicates
unique_queries = deduplicate_by_embedding(queries, threshold=0.9)

# Categorize by type and difficulty
for query in unique_queries:
    query["question_type"] = classify_question_type(query["text"])
    query["difficulty"] = estimate_difficulty(query["text"])

# Ensure balanced distribution across types
balanced = balance_by_taxonomy(unique_queries, target_size=100)
```

### Step 3: Annotate Ground Truth (Day 2-5)

```python
# Assign to annotators
batches = split_for_annotation(balanced, num_annotators=2, overlap=0.15)

# Each annotator writes ground truth answers and identifies relevant docs
# Use the annotation guidelines above
```

### Step 4: Reconcile and Measure IAA (Day 5-6)

```python
# Measure agreement on overlapping items
kappa = measure_iaa_binary(annotator_a_overlap, annotator_b_overlap)
answer_sim = measure_iaa_answer_similarity(answers_a, answers_b, embed_model)

# Reconcile disagreements through discussion
# Document resolution rationale
```

### Step 5: Split and Validate (Day 6-7)

```python
# Split into tuning (30%) and holdout (70%)
tuning_set, holdout_set = stratified_split(golden_set, test_size=0.7)

# Validate by running baseline evaluation
baseline_results = run_full_ragas_evaluation(holdout_set, current_pipeline)
print(f"Baseline scores: {baseline_results}")
# Save as the benchmark to beat
```

---

## Common Pitfalls

1. **Using only factual queries.** A golden set of "What is X?" questions does not test procedural, reasoning, or comparison capabilities. Follow the taxonomy distribution.
2. **Writing ground truth from memory.** Ground truth answers must be based on what is actually in the knowledge base, not the annotator's general knowledge. Have annotators reference the KB.
3. **Not including unanswerable queries.** 10% of your golden set should be queries the system should refuse to answer or express uncertainty about. This tests hallucination resistance.
4. **Letting the golden set go stale.** When you add new documents to the KB, update the golden set. When you find production failures, add them. Version and maintain it actively.
5. **Tuning on the holdout set.** If you use the holdout set to tune hyperparameters, it becomes contaminated. Maintain a strict split and never peek at holdout results during tuning.
6. **Skipping IAA.** Without inter-annotator agreement measurement, you do not know if your ground truth is reliable. Two annotators who disagree produce a noisy evaluation signal.
7. **Insufficient annotation depth.** Ground truth answers that say "yes" or "use RLS" are too vague to evaluate against. Write complete answers with all expected details.

---

## References

- Snow et al. "Cheap and Fast -- But is it Good? Evaluating Non-Expert Annotations for Natural Language Tasks" (EMNLP 2008)
- Rajpurkar et al. "SQuAD: 100,000+ Questions for Machine Comprehension of Text" (EMNLP 2016)
- Thakur et al. "BEIR: A Heterogeneous Benchmark for Zero-shot Evaluation of Information Retrieval Models" (NeurIPS 2021)
- RAGAS evaluation docs: https://docs.ragas.io/en/stable/concepts/evaluation_dataset/
