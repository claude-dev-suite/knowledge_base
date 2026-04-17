# Giskard RAGET -- Testset Generation Deep Dive

## TL;DR

RAGET's testset generation engine analyzes your knowledge base documents to automatically produce evaluation questions across four complexity levels: simple (single-fact retrieval), complex (multi-document reasoning), distracting (questions with plausible but wrong contexts nearby), and conversational (multi-turn follow-ups). This guide covers the internal mechanics of each question type, how RAGET's knowledge graph analysis determines what to ask, quality filtering strategies to remove bad questions before they pollute your evaluation, customization options for domain-specific generation, and scaling strategies for large knowledge bases. The goal is to move from "we need to manually write 200 test questions" to "RAGET generates 200 questions in 10 minutes, we review 20 and approve the batch."

---

## How RAGET Analyzes Your Knowledge Base

### The Knowledge Graph Construction

Before generating any questions, RAGET builds an internal representation of your documents:

```
Step 1: Document Ingestion
  Raw documents --> Split into semantic units (paragraphs, sections)
  --> Each unit becomes a node in the knowledge graph

Step 2: Fact Extraction
  Each unit --> LLM extracts key facts, entities, and relationships
  --> Facts become edges between entity nodes

Step 3: Cross-Document Linking
  Entities mentioned in multiple documents --> Linked
  --> Enables multi-hop question generation

Step 4: Information Density Scoring
  Each node scored by: number of facts, uniqueness, complexity
  --> High-density nodes are preferred for question generation
```

```python
"""
Understanding RAGET's knowledge base analysis.
"""
import pandas as pd
from giskard.rag import KnowledgeBase


def analyze_knowledge_base(docs_df: pd.DataFrame) -> dict:
    """
    Analyze a knowledge base to understand what RAGET will work with.

    This helps predict question quality and identify gaps.
    """
    kb = KnowledgeBase.from_pandas(docs_df)

    # Analyze document statistics
    analysis = {
        "num_documents": len(docs_df),
        "avg_doc_length": docs_df["text"].str.len().mean(),
        "min_doc_length": docs_df["text"].str.len().min(),
        "max_doc_length": docs_df["text"].str.len().max(),
        "total_chars": docs_df["text"].str.len().sum(),
    }

    # Estimate question generation capacity
    # Rule of thumb: ~1 question per 500 chars of content
    est_capacity = analysis["total_chars"] // 500
    analysis["estimated_question_capacity"] = est_capacity

    # Check for potential issues
    issues = []
    if analysis["num_documents"] < 10:
        issues.append(
            "Few documents: limited question diversity. "
            "Complex and distracting questions may be poor quality."
        )
    if analysis["min_doc_length"] < 50:
        issues.append(
            "Very short documents found. RAGET may skip these or "
            "generate trivial questions from them."
        )
    if analysis["avg_doc_length"] > 10000:
        issues.append(
            "Long documents. Consider splitting into sections "
            "for better question targeting."
        )

    # Check content overlap (rough)
    all_words = set()
    per_doc_words = []
    for text in docs_df["text"]:
        words = set(text.lower().split())
        per_doc_words.append(words)
        all_words.update(words)

    # Vocabulary overlap between documents
    if len(per_doc_words) >= 2:
        overlaps = []
        for i in range(min(len(per_doc_words), 20)):
            for j in range(i + 1, min(len(per_doc_words), 20)):
                overlap = len(per_doc_words[i] & per_doc_words[j])
                union = len(per_doc_words[i] | per_doc_words[j])
                if union > 0:
                    overlaps.append(overlap / union)
        avg_overlap = sum(overlaps) / max(len(overlaps), 1)
        analysis["avg_document_overlap"] = avg_overlap

        if avg_overlap > 0.7:
            issues.append(
                "High document overlap. Documents may be too similar, "
                "limiting distracting question quality."
            )
        elif avg_overlap < 0.1:
            issues.append(
                "Low document overlap. Complex multi-hop questions may "
                "feel artificial since documents are unrelated."
            )

    analysis["issues"] = issues

    print(f"Knowledge Base Analysis:")
    for key, value in analysis.items():
        if key != "issues":
            print(f"  {key}: {value}")
    if issues:
        print(f"\nPotential Issues:")
        for issue in issues:
            print(f"  - {issue}")

    return analysis
```

---

## Question Type Deep Dive

### Simple Questions

Simple questions test single-fact retrieval from a single passage. They are the baseline: if your pipeline cannot answer these, nothing else matters.

**Generation strategy**: RAGET identifies a key fact in a single passage and generates a question whose answer is entirely contained in that passage.

**What they test**:
- Can the retriever find the right passage for a direct query?
- Can the generator extract and present a single fact correctly?

**Typical failure modes**:
- Retriever returns a related but wrong passage
- Generator paraphrases the fact incorrectly
- Query terms do not match the passage vocabulary (lexical gap)

```python
"""
Simple question generation and analysis.
"""


def analyze_simple_questions(testset_df: pd.DataFrame) -> dict:
    """
    Analyze the quality of generated simple questions.

    Simple questions should have:
    - Clear, unambiguous answers
    - Answers contained in a single context passage
    - No multi-hop reasoning required
    """
    simple = testset_df[testset_df["question_type"] == "simple"]

    analysis = {
        "count": len(simple),
        "avg_question_length": simple["question"].str.len().mean(),
        "avg_answer_length": simple["reference_answer"].str.len().mean(),
    }

    # Check for quality issues
    issues = []

    # Questions that are too short (likely yes/no)
    short_q = simple[simple["question"].str.len() < 20]
    if len(short_q) > 0:
        issues.append(f"{len(short_q)} very short questions (< 20 chars)")
        for _, row in short_q.head(3).iterrows():
            issues.append(f"  Example: {row['question']}")

    # Questions that start with "Is" or "Does" (yes/no -- less useful)
    yes_no = simple[
        simple["question"].str.lower().str.startswith(("is ", "does ", "can ", "are "))
    ]
    if len(yes_no) > len(simple) * 0.3:
        issues.append(
            f"{len(yes_no)}/{len(simple)} are yes/no questions. "
            "Consider regenerating with stronger generation prompts."
        )

    # Answers that are too long (might indicate complex question mislabeled)
    long_a = simple[simple["reference_answer"].str.len() > 500]
    if len(long_a) > 0:
        issues.append(
            f"{len(long_a)} simple questions have long answers (> 500 chars). "
            "These may actually be complex questions."
        )

    analysis["issues"] = issues
    return analysis
```

### Complex Questions

Complex questions require synthesizing information from multiple passages or performing multi-step reasoning. They are the hardest to answer correctly and the most valuable for evaluation.

**Generation strategy**: RAGET identifies entities or topics that appear in multiple documents, then generates questions that require information from at least two passages.

**What they test**:
- Can the retriever find multiple relevant passages across documents?
- Can the generator synthesize information from multiple contexts?
- Does the pipeline handle multi-hop reasoning?

```python
"""
Complex question analysis and quality control.
"""


def analyze_complex_questions(testset_df: pd.DataFrame) -> dict:
    """
    Analyze complex questions for multi-hop quality.

    Complex questions should require information from 2+ passages.
    If a complex question can be answered from a single passage,
    it is actually a simple question and should be reclassified.
    """
    complex_q = testset_df[testset_df["question_type"] == "complex"]

    if "reference_context" not in complex_q.columns:
        return {"count": len(complex_q), "warning": "No reference context column"}

    analysis = {"count": len(complex_q)}

    # Check how many reference contexts each question uses
    context_counts = []
    for _, row in complex_q.iterrows():
        ctx = row.get("reference_context", "")
        if isinstance(ctx, list):
            context_counts.append(len(ctx))
        elif isinstance(ctx, str):
            # Rough heuristic: count distinct source passages
            # (RAGET may concatenate them with separators)
            segments = ctx.split("\n\n---\n\n")
            context_counts.append(len(segments))
        else:
            context_counts.append(1)

    import numpy as np
    analysis["avg_contexts_per_question"] = float(np.mean(context_counts))
    analysis["single_context_count"] = sum(1 for c in context_counts if c <= 1)

    if analysis["single_context_count"] > len(complex_q) * 0.3:
        analysis["warning"] = (
            f"{analysis['single_context_count']}/{len(complex_q)} 'complex' questions "
            "appear to use only 1 context. These may be mislabeled simple questions."
        )

    return analysis
```

### Distracting Questions

Distracting questions are paired with contexts that are topically related but do not contain the answer. They test whether the retriever can distinguish relevant from merely similar content.

**Generation strategy**: RAGET identifies passages that share vocabulary or topic with the answer passage but contain different information. It then generates a question where the answer is in passage A, but passage B is a plausible but incorrect alternative.

```python
"""
Distracting question generation and validation.
"""


def validate_distracting_questions(
    testset_df: pd.DataFrame,
    knowledge_df: pd.DataFrame,
) -> dict:
    """
    Validate that distracting questions have genuine distractors.

    A good distracting question has:
    1. A clear correct answer in the reference context
    2. At least one distractor context that is topically similar
       but does NOT contain the answer
    3. The distractor should be hard to distinguish from the
       correct context
    """
    distracting = testset_df[testset_df["question_type"] == "distracting"]

    analysis = {
        "count": len(distracting),
        "quality_scores": [],
    }

    for _, row in distracting.iterrows():
        question = row["question"]
        ref_answer = row.get("reference_answer", "")
        ref_context = row.get("reference_context", "")

        # A good distracting question has a clear answer
        answer_clarity = min(len(ref_answer) / 50, 1.0)  # Longer answers are clearer

        # Check if the question contains disambiguating keywords
        question_words = set(question.lower().split())
        specificity = len(question_words) / 20  # More words = more specific

        quality = (answer_clarity + specificity) / 2
        analysis["quality_scores"].append(quality)

    import numpy as np
    if analysis["quality_scores"]:
        analysis["avg_quality"] = float(np.mean(analysis["quality_scores"]))
        analysis["low_quality_count"] = sum(
            1 for s in analysis["quality_scores"] if s < 0.3
        )

    return analysis
```

### Conversational Questions

Conversational questions simulate multi-turn interactions where follow-up questions depend on previous answers.

**Generation strategy**: RAGET generates a sequence of 2-3 related questions where each follow-up references or depends on the previous answer.

```python
"""
Conversational question handling.
"""


def validate_conversational_questions(testset_df: pd.DataFrame) -> dict:
    """
    Validate conversational questions for coherence.

    Good conversational questions should:
    1. Have clear dependencies between turns
    2. Each follow-up should be unanswerable without prior context
    3. The conversation should follow a natural flow
    """
    conv = testset_df[testset_df["question_type"] == "conversational"]

    analysis = {
        "count": len(conv),
        "has_history": 0,
        "standalone_followups": 0,
    }

    for _, row in conv.iterrows():
        question = row["question"]
        history = row.get("conversation_history", [])

        if history:
            analysis["has_history"] += 1

            # Check if the question makes sense without history
            # (if it does, it is not really conversational)
            pronouns = {"it", "this", "that", "they", "them", "its", "those"}
            q_words = set(question.lower().split())
            has_reference = bool(q_words & pronouns)

            if not has_reference:
                analysis["standalone_followups"] += 1

    if analysis["count"] > 0:
        analysis["dependency_rate"] = (
            1 - analysis["standalone_followups"] / analysis["count"]
        )

    return analysis
```

---

## Quality Filtering

### Automated Quality Checks

```python
"""
Filter generated test sets for quality before use.
"""
import re

import numpy as np
import pandas as pd


class TestsetQualityFilter:
    """
    Filter RAGET-generated test sets to remove low-quality questions.

    Apply this after generation and before using the test set for evaluation.
    """

    def __init__(
        self,
        min_question_length: int = 15,
        max_question_length: int = 300,
        min_answer_length: int = 10,
        max_answer_length: int = 1000,
    ):
        self.min_q_len = min_question_length
        self.max_q_len = max_question_length
        self.min_a_len = min_answer_length
        self.max_a_len = max_answer_length

    def filter(self, testset_df: pd.DataFrame) -> pd.DataFrame:
        """
        Apply all quality filters and return the cleaned dataset.

        Returns:
            Filtered DataFrame with a 'filter_reason' column
            for rejected rows.
        """
        df = testset_df.copy()
        df["filter_reason"] = None

        initial_count = len(df)

        # Length filters
        df.loc[
            df["question"].str.len() < self.min_q_len,
            "filter_reason",
        ] = "question_too_short"

        df.loc[
            df["question"].str.len() > self.max_q_len,
            "filter_reason",
        ] = "question_too_long"

        df.loc[
            df["reference_answer"].str.len() < self.min_a_len,
            "filter_reason",
        ] = "answer_too_short"

        df.loc[
            df["reference_answer"].str.len() > self.max_a_len,
            "filter_reason",
        ] = "answer_too_long"

        # Content quality filters
        df = self._filter_meta_commentary(df)
        df = self._filter_duplicates(df)
        df = self._filter_yes_no_questions(df)
        df = self._filter_unanswerable(df)

        # Report
        filtered = df[df["filter_reason"].notna()]
        accepted = df[df["filter_reason"].isna()]

        print(f"Quality filtering: {initial_count} -> {len(accepted)} questions")
        if len(filtered) > 0:
            reasons = filtered["filter_reason"].value_counts()
            for reason, count in reasons.items():
                print(f"  Removed {count}: {reason}")

        return accepted.drop(columns=["filter_reason"])

    def _filter_meta_commentary(self, df: pd.DataFrame) -> pd.DataFrame:
        """Remove questions/answers with LLM meta-commentary."""
        meta_patterns = [
            r"^(Here is|Here's|Sure|I'll)",
            r"^(As an AI|As a language model)",
            r"(based on the (passage|document|text|context))",
            r"(according to the (given|provided))",
        ]

        for pattern in meta_patterns:
            mask = (
                df["question"].str.contains(pattern, case=False, regex=True, na=False)
                | df["reference_answer"].str.contains(pattern, case=False, regex=True, na=False)
            )
            df.loc[
                mask & df["filter_reason"].isna(),
                "filter_reason",
            ] = "meta_commentary"

        return df

    def _filter_duplicates(self, df: pd.DataFrame) -> pd.DataFrame:
        """Remove near-duplicate questions."""
        seen = set()
        for idx, row in df.iterrows():
            if df.loc[idx, "filter_reason"] is not None:
                continue

            # Normalize for comparison
            normalized = re.sub(r'\s+', ' ', row["question"].lower().strip())
            if normalized in seen:
                df.loc[idx, "filter_reason"] = "duplicate"
            else:
                seen.add(normalized)

        return df

    def _filter_yes_no_questions(self, df: pd.DataFrame) -> pd.DataFrame:
        """Flag yes/no questions (less useful for evaluation)."""
        yes_no_starts = ("is ", "does ", "can ", "are ", "was ", "were ", "did ", "has ", "have ")
        for idx, row in df.iterrows():
            if df.loc[idx, "filter_reason"] is not None:
                continue
            if row["question"].lower().startswith(yes_no_starts):
                # Only filter if the answer is also very short
                if len(row["reference_answer"]) < 30:
                    df.loc[idx, "filter_reason"] = "yes_no_trivial"

        return df

    def _filter_unanswerable(self, df: pd.DataFrame) -> pd.DataFrame:
        """Remove questions whose reference answer indicates unanswerability."""
        unanswerable_patterns = [
            r"(not (mentioned|stated|found|clear|specified))",
            r"(no information|insufficient|cannot be determined)",
            r"(the (text|passage|document) does not)",
        ]

        for pattern in unanswerable_patterns:
            mask = df["reference_answer"].str.contains(
                pattern, case=False, regex=True, na=False
            )
            df.loc[
                mask & df["filter_reason"].isna(),
                "filter_reason",
            ] = "unanswerable"

        return df
```

### Manual Review Workflow

```python
"""
Streamlined manual review for RAGET-generated test sets.
"""
import json
from pathlib import Path


class TestsetReviewer:
    """
    Review generated test sets interactively.

    Workflow:
    1. Generate test set with RAGET
    2. Apply automated quality filtering
    3. Review a sample manually (20-30 questions)
    4. If sample quality > 80%, accept the batch
    5. If sample quality < 80%, adjust generation and regenerate
    """

    def __init__(
        self,
        testset_df: "pd.DataFrame",
        review_sample_size: int = 30,
    ):
        import random
        self.df = testset_df
        self.sample_indices = random.sample(
            range(len(testset_df)),
            min(review_sample_size, len(testset_df)),
        )
        self.reviews = {}

    def review_cli(self) -> dict:
        """Interactive CLI review of sampled questions."""
        for i, idx in enumerate(self.sample_indices):
            row = self.df.iloc[idx]

            print(f"\n{'='*60}")
            print(f"Review {i+1}/{len(self.sample_indices)} (type: {row.get('question_type', '?')})")
            print(f"{'='*60}")
            print(f"Q: {row['question']}")
            print(f"A: {row['reference_answer'][:300]}")

            # Rate: accept / reject / edit
            while True:
                rating = input("\n[a]ccept / [r]eject / [s]kip / [q]uit: ").strip().lower()
                if rating in ("a", "r", "s", "q"):
                    break

            if rating == "q":
                break
            elif rating == "s":
                continue

            self.reviews[idx] = {
                "accepted": rating == "a",
                "question_type": row.get("question_type", "unknown"),
            }

        # Calculate acceptance rate
        if self.reviews:
            accepted = sum(1 for r in self.reviews.values() if r["accepted"])
            total = len(self.reviews)
            rate = accepted / total

            print(f"\nReview Results:")
            print(f"  Reviewed: {total}")
            print(f"  Accepted: {accepted} ({rate*100:.0f}%)")
            print(f"  Rejected: {total - accepted}")

            if rate >= 0.80:
                print("\nSample quality is acceptable. Batch approved.")
            else:
                print("\nSample quality is low. Consider regenerating or adjusting filters.")

            # Per-type acceptance
            by_type = {}
            for r in self.reviews.values():
                qt = r["question_type"]
                if qt not in by_type:
                    by_type[qt] = {"accepted": 0, "total": 0}
                by_type[qt]["total"] += 1
                if r["accepted"]:
                    by_type[qt]["accepted"] += 1

            print("\nBy question type:")
            for qt, counts in by_type.items():
                type_rate = counts["accepted"] / max(counts["total"], 1)
                print(f"  {qt}: {counts['accepted']}/{counts['total']} ({type_rate*100:.0f}%)")

            return {"acceptance_rate": rate, "by_type": by_type}

        return {"acceptance_rate": 0, "by_type": {}}
```

---

## Scaling to Large Knowledge Bases

### Chunked Generation for Large Corpora

```python
"""
Strategies for generating test sets from large knowledge bases.
"""
import random

import pandas as pd
from giskard.rag import KnowledgeBase, generate_testset


def generate_from_large_corpus(
    docs_df: pd.DataFrame,
    total_questions: int = 200,
    docs_per_batch: int = 50,
    seed: int = 42,
) -> "QATestset":
    """
    Generate test sets from large corpora by processing in batches.

    For corpora with 1000+ documents, processing all at once is
    slow and expensive. Instead, sample document batches and generate
    questions from each.
    """
    random.seed(seed)

    # Stratified sampling: ensure diversity across document sources
    if "source" in docs_df.columns:
        sources = docs_df["source"].unique()
        docs_per_source = max(1, docs_per_batch // len(sources))

        sampled_indices = []
        for source in sources:
            source_docs = docs_df[docs_df["source"] == source].index.tolist()
            sample = random.sample(
                source_docs, min(docs_per_source, len(source_docs))
            )
            sampled_indices.extend(sample)
    else:
        sampled_indices = random.sample(
            range(len(docs_df)), min(docs_per_batch, len(docs_df))
        )

    sampled_df = docs_df.iloc[sampled_indices].reset_index(drop=True)
    print(f"Sampled {len(sampled_df)} documents from {len(docs_df)} total")

    kb = KnowledgeBase.from_pandas(sampled_df)

    testset = generate_testset(
        kb,
        num_questions=total_questions,
        language="en",
    )

    print(f"Generated {len(testset)} questions from {len(sampled_df)} documents")
    return testset


def merge_testsets(testsets: list) -> pd.DataFrame:
    """Merge multiple RAGET test sets into one."""
    dfs = [ts.to_pandas() for ts in testsets]
    merged = pd.concat(dfs, ignore_index=True)

    # Deduplicate by question text
    merged = merged.drop_duplicates(subset=["question"], keep="first")

    print(f"Merged {len(dfs)} testsets into {len(merged)} unique questions")
    return merged
```

---

## Common Pitfalls

1. **Accepting all generated questions unchecked**: always apply automated quality filtering and review a sample. RAGET's LLM-generated questions can contain errors, ambiguities, or questions whose reference answers are wrong.

2. **Over-relying on simple questions**: simple questions are the easiest to generate and the easiest to answer. If your test set is 80% simple questions, you will get inflated scores that do not reflect real-world performance on harder queries.

3. **Not saving test sets**: regenerating test sets produces different questions each time, making longitudinal comparison impossible. Save your test set after generation and quality filtering, and reuse it for consistent evaluation.

4. **Ignoring the knowledge graph analysis**: RAGET's question quality depends on the knowledge base quality. Documents that are too short, too similar, or too unrelated produce poor questions. Run the analysis first and fix corpus issues before generating.

5. **Using distracting questions without validation**: distracting questions are only useful if the distractor contexts are genuinely confusing. If the distractors are obviously irrelevant (e.g., a cooking recipe as a distractor for a database question), they do not test anything meaningful.

6. **Not customizing the agent description**: RAGET uses the agent description to generate more realistic questions. A generic description like "a QA system" produces generic questions. Describe your actual system, users, and domain for better results.

---

## References

- Giskard RAGET Documentation: https://docs.giskard.ai/en/latest/open_source/testset_generation/
- Giskard GitHub: https://github.com/Giskard-AI/giskard
- Knowledge Base Analysis: https://docs.giskard.ai/en/latest/open_source/testset_generation/knowledge_base.html
- Question Type Distribution: https://docs.giskard.ai/en/latest/open_source/testset_generation/question_types.html
