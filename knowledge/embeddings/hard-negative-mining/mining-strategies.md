# Hard Negative Mining Strategies -- Detailed Implementation

## Overview / TL;DR

This guide provides complete implementations for every major hard negative mining strategy: BM25-mined negatives (fast, keyword-overlap), dense-mined negatives (similar embeddings but wrong), cross-encoder-scored negatives (most accurate, expensive), TAS-B distillation method, MarginMSE loss, and curriculum mining (easy to hard). Each strategy includes runnable code, performance characteristics, and guidance on when to use it. For most use cases, dense-mined negatives with cross-encoder filtering provide the best quality/cost trade-off.

---

## Strategy 1: BM25-Mined Negatives

Use sparse retrieval (BM25) to find documents with high keyword overlap but different meaning. These are "lexically hard" negatives.

```python
from rank_bm25 import BM25Okapi
import numpy as np


def mine_bm25_negatives(
    queries: list[str],
    positives: list[str],
    corpus: list[str],
    n_negatives: int = 10,
) -> list[dict]:
    """Mine hard negatives using BM25 sparse retrieval.

    BM25 finds documents with high keyword overlap.
    These are "lexically confusing" -- they share terms with the query
    but answer a different question.

    Fast but lower quality than dense mining.
    """
    # Tokenize corpus for BM25
    tokenized_corpus = [doc.lower().split() for doc in corpus]
    bm25 = BM25Okapi(tokenized_corpus)

    positive_set = set(positives)
    training_data = []

    for i, (query, positive) in enumerate(zip(queries, positives)):
        if i % 500 == 0:
            print(f"BM25 mining: {i}/{len(queries)}")

        # Get BM25 scores
        tokenized_query = query.lower().split()
        scores = bm25.get_scores(tokenized_query)
        ranked = np.argsort(scores)[::-1]

        # Collect negatives (exclude positive)
        negatives = []
        for idx in ranked:
            if corpus[idx] not in positive_set and len(negatives) < n_negatives:
                negatives.append(corpus[idx])

        training_data.append({
            "query": query,
            "positive": positive,
            "negatives": negatives,
        })

    return training_data


# Cost: ~$0 (runs locally, ~10K queries/minute on CPU)
# Quality: Medium (lexical overlap, misses semantic similarity)
```

---

## Strategy 2: Dense-Mined Negatives

Use a pre-trained embedding model to find semantically similar but irrelevant documents. These are the most common and effective hard negatives.

```python
import numpy as np
from sentence_transformers import SentenceTransformer
import faiss


def mine_dense_negatives(
    queries: list[str],
    positives: list[str],
    corpus: list[str],
    model_name: str = "BAAI/bge-base-en-v1.5",
    n_negatives: int = 10,
    n_candidates: int = 100,
    exclude_top_k: int = 5,
) -> list[dict]:
    """Mine hard negatives using dense retrieval with FAISS.

    Uses a pre-trained embedding model to find the most similar
    corpus documents. The top-K most similar (likely relevant) are
    excluded to avoid false negatives.

    FAISS enables efficient similarity search over large corpora.
    """
    model = SentenceTransformer(model_name)

    # Embed corpus and build FAISS index
    print(f"Embedding {len(corpus)} documents...")
    corpus_embs = model.encode(
        corpus,
        normalize_embeddings=True,
        batch_size=256,
        show_progress_bar=True,
    ).astype(np.float32)

    dim = corpus_embs.shape[1]
    index = faiss.IndexFlatIP(dim)  # Inner product (cosine similarity for normalized vecs)
    index.add(corpus_embs)

    # Build positive lookup for exclusion
    positive_indices = {}
    for i, pos in enumerate(positives):
        for j, doc in enumerate(corpus):
            if doc == pos:
                positive_indices[i] = j
                break

    # Mine negatives for each query
    print(f"Mining negatives for {len(queries)} queries...")
    query_embs = model.encode(
        queries,
        normalize_embeddings=True,
        batch_size=256,
        show_progress_bar=True,
    ).astype(np.float32)

    # Batch FAISS search
    scores, indices = index.search(query_embs, n_candidates)

    training_data = []
    for i in range(len(queries)):
        negatives = []
        for rank, (idx, score) in enumerate(zip(indices[i], scores[i])):
            if idx == -1:
                continue
            # Skip the positive document
            if i in positive_indices and idx == positive_indices[i]:
                continue
            # Skip top-K (likely relevant, false negative risk)
            if rank < exclude_top_k:
                continue
            negatives.append(corpus[idx])
            if len(negatives) >= n_negatives:
                break

        training_data.append({
            "query": queries[i],
            "positive": positives[i],
            "negatives": negatives,
        })

    return training_data


# Cost: ~$0 (runs locally, depends on GPU for embedding)
# Quality: High (semantic similarity, good hard negatives)
```

---

## Strategy 3: Cross-Encoder-Scored Negatives

Use a cross-encoder to score candidate negatives and select the hardest ones. This is the most accurate method but the most expensive.

```python
import numpy as np
from sentence_transformers import SentenceTransformer, CrossEncoder


def mine_cross_encoder_negatives(
    queries: list[str],
    positives: list[str],
    corpus: list[str],
    bi_encoder_name: str = "BAAI/bge-base-en-v1.5",
    cross_encoder_name: str = "cross-encoder/ms-marco-MiniLM-L-12-v2",
    n_negatives: int = 7,
    n_candidates: int = 50,
    max_ce_score: float = 0.3,
) -> list[dict]:
    """Mine negatives scored by a cross-encoder.

    Two-stage process:
    1. Use bi-encoder to retrieve top-50 candidates (fast).
    2. Score candidates with cross-encoder (slow but accurate).
    3. Select negatives with high bi-encoder score but low cross-encoder score.

    These are the hardest, most informative negatives: the bi-encoder
    thinks they are relevant, but the cross-encoder (which sees query
    and document together) knows they are not.
    """
    bi_encoder = SentenceTransformer(bi_encoder_name)
    cross_enc = CrossEncoder(cross_encoder_name)

    # Stage 1: Dense retrieval for candidates
    print("Embedding corpus...")
    corpus_embs = bi_encoder.encode(
        corpus, normalize_embeddings=True, batch_size=256, show_progress_bar=True
    ).astype(np.float32)

    training_data = []

    for i, (query, positive) in enumerate(zip(queries, positives)):
        if i % 100 == 0:
            print(f"Cross-encoder scoring: {i}/{len(queries)}")

        # Get top candidates from bi-encoder
        query_emb = bi_encoder.encode([query], normalize_embeddings=True)[0]
        scores = np.dot(corpus_embs, query_emb)
        top_indices = np.argsort(scores)[::-1][:n_candidates]

        # Filter out the positive
        candidates = [(corpus[idx], float(scores[idx])) for idx in top_indices if corpus[idx] != positive]

        # Stage 2: Score with cross-encoder
        pairs = [(query, cand[0]) for cand in candidates]
        ce_scores = cross_enc.predict(pairs)

        # Select: high bi-encoder score, low cross-encoder score
        scored_candidates = []
        for (text, bi_score), ce_score in zip(candidates, ce_scores):
            if ce_score < max_ce_score:  # Cross-encoder says NOT relevant
                scored_candidates.append({
                    "text": text,
                    "bi_score": bi_score,
                    "ce_score": float(ce_score),
                    "hardness": bi_score - ce_score,  # Higher = harder
                })

        # Sort by hardness (most confusing to bi-encoder)
        scored_candidates.sort(key=lambda x: -x["hardness"])
        negatives = [c["text"] for c in scored_candidates[:n_negatives]]

        training_data.append({
            "query": query,
            "positive": positive,
            "negatives": negatives,
        })

    return training_data


# Cost: ~$5-20 per 10K queries (cross-encoder inference)
# Quality: Highest (verified irrelevant, maximally confusing)
```

---

## Strategy 4: TAS-B Distillation Method

Topic Aware Sampling with Balanced training. Uses a teacher model (cross-encoder) to provide soft relevance labels, then trains the student (bi-encoder) with these labels.

```python
import numpy as np
from sentence_transformers import SentenceTransformer, CrossEncoder, InputExample, losses
from torch.utils.data import DataLoader


def prepare_tasb_training_data(
    queries: list[str],
    corpus: list[str],
    bi_encoder_name: str = "BAAI/bge-base-en-v1.5",
    cross_encoder_name: str = "cross-encoder/ms-marco-MiniLM-L-12-v2",
    n_candidates_per_query: int = 20,
) -> list[InputExample]:
    """Prepare TAS-B style training data with teacher scores.

    For each query, retrieve candidates and score them with a teacher
    cross-encoder. The student bi-encoder learns to replicate these scores.
    """
    bi_encoder = SentenceTransformer(bi_encoder_name)
    cross_enc = CrossEncoder(cross_encoder_name)

    # Dense retrieval for candidates
    corpus_embs = bi_encoder.encode(corpus, normalize_embeddings=True, batch_size=256)

    examples = []

    for i, query in enumerate(queries):
        if i % 200 == 0:
            print(f"TAS-B prep: {i}/{len(queries)}")

        query_emb = bi_encoder.encode([query], normalize_embeddings=True)[0]
        scores = np.dot(corpus_embs, query_emb)
        top_indices = np.argsort(scores)[::-1][:n_candidates_per_query]

        # Score with teacher cross-encoder
        pairs = [(query, corpus[idx]) for idx in top_indices]
        teacher_scores = cross_enc.predict(pairs)

        # Create pairwise examples from teacher scores
        for j in range(len(top_indices)):
            for k in range(j + 1, len(top_indices)):
                if abs(teacher_scores[j] - teacher_scores[k]) > 0.1:
                    # Higher-scored document is the positive
                    if teacher_scores[j] > teacher_scores[k]:
                        examples.append(InputExample(
                            texts=[query, corpus[top_indices[j]], corpus[top_indices[k]]],
                        ))
                    else:
                        examples.append(InputExample(
                            texts=[query, corpus[top_indices[k]], corpus[top_indices[j]]],
                        ))

    return examples
```

---

## Strategy 5: MarginMSE Loss

Train with soft labels from a teacher model rather than binary relevant/not-relevant. This naturally handles the difficulty spectrum.

```python
from sentence_transformers import SentenceTransformer, CrossEncoder, losses, InputExample
from torch.utils.data import DataLoader
import numpy as np


def train_with_margin_mse(
    student_model_name: str,
    teacher_model_name: str,
    queries: list[str],
    corpus: list[str],
    output_path: str,
    n_candidates: int = 20,
    epochs: int = 3,
    batch_size: int = 32,
):
    """Train with MarginMSE loss using teacher-scored examples.

    MarginMSE computes the margin between positive and negative scores
    from both student and teacher, then minimizes the MSE between them.
    This teaches the student to replicate the teacher's ranking.
    """
    student = SentenceTransformer(student_model_name)
    teacher = CrossEncoder(teacher_model_name)

    # Get candidates and teacher scores
    corpus_embs = student.encode(corpus, normalize_embeddings=True, batch_size=256)

    examples = []
    for i, query in enumerate(queries):
        if i % 200 == 0:
            print(f"Preparing MarginMSE data: {i}/{len(queries)}")

        query_emb = student.encode([query], normalize_embeddings=True)[0]
        bi_scores = np.dot(corpus_embs, query_emb)
        top_indices = np.argsort(bi_scores)[::-1][:n_candidates]

        pairs = [(query, corpus[idx]) for idx in top_indices]
        teacher_scores = teacher.predict(pairs)

        # Create margin examples: (query, doc_a, doc_b, margin)
        for j in range(len(top_indices) - 1):
            margin = teacher_scores[j] - teacher_scores[j + 1]
            if abs(margin) > 0.05:
                examples.append(InputExample(
                    texts=[query, corpus[top_indices[j]], corpus[top_indices[j + 1]]],
                    label=float(margin),
                ))

    dataloader = DataLoader(examples, shuffle=True, batch_size=batch_size)
    train_loss = losses.MarginMSELoss(model=student)

    student.fit(
        train_objectives=[(dataloader, train_loss)],
        epochs=epochs,
        warmup_steps=100,
        output_path=output_path,
        show_progress_bar=True,
    )

    return student
```

---

## Strategy 6: Curriculum Mining (Easy to Hard)

Start training with easy negatives and progressively introduce harder ones. This prevents the model from being overwhelmed by hard examples early in training.

```python
import numpy as np
from sentence_transformers import SentenceTransformer, InputExample


def curriculum_mine_negatives(
    queries: list[str],
    positives: list[str],
    corpus: list[str],
    model: SentenceTransformer,
    epoch: int,
    total_epochs: int,
    n_negatives: int = 7,
) -> list[InputExample]:
    """Mine negatives with difficulty proportional to training progress.

    Early epochs: negatives from ranks 50-100 (easy).
    Middle epochs: negatives from ranks 20-50 (medium).
    Late epochs: negatives from ranks 5-20 (hard).

    This curriculum helps the model build a foundation before
    tackling the hardest distinctions.
    """
    # Compute difficulty range based on epoch
    progress = epoch / total_epochs  # 0.0 to 1.0
    start_rank = int(50 * (1 - progress)) + 5  # 55 -> 5
    end_rank = start_rank + 30

    corpus_embs = model.encode(corpus, normalize_embeddings=True, batch_size=256)
    positive_set = set(positives)

    examples = []
    for i, (query, positive) in enumerate(zip(queries, positives)):
        query_emb = model.encode([query], normalize_embeddings=True)[0]
        scores = np.dot(corpus_embs, query_emb)
        ranked = np.argsort(scores)[::-1]

        negatives = []
        for rank, idx in enumerate(ranked):
            if rank < start_rank:
                continue
            if rank > end_rank:
                break
            if corpus[idx] not in positive_set:
                negatives.append(corpus[idx])
            if len(negatives) >= n_negatives:
                break

        for neg in negatives[:1]:
            examples.append(InputExample(
                texts=[query, positive, neg],
            ))

    print(f"Epoch {epoch}: mining ranks {start_rank}-{end_rank} "
          f"({len(examples)} examples)")

    return examples
```

---

## Avoiding False Negatives

The most critical aspect of hard negative mining. False negatives corrupt the training signal.

```python
import anthropic


def verify_negatives_with_llm(
    query: str,
    positive: str,
    candidate_negatives: list[str],
    max_negatives: int = 7,
) -> list[str]:
    """Use an LLM to verify that candidates are truly irrelevant.

    More expensive than cross-encoder but more accurate for edge cases.
    Use for high-stakes domains (medical, legal).
    """
    client = anthropic.Anthropic()

    verified = []
    for neg in candidate_negatives:
        response = client.messages.create(
            model="claude-haiku-4-20250514",
            max_tokens=10,
            messages=[{
                "role": "user",
                "content": (
                    f"Does this document answer the query? Reply only YES or NO.\n\n"
                    f"Query: {query}\n\n"
                    f"Document: {neg[:500]}"
                ),
            }],
        )

        answer = response.content[0].text.strip().upper()
        if answer == "NO":
            verified.append(neg)
        # If YES, this is a false negative -- skip it

        if len(verified) >= max_negatives:
            break

    return verified
```

---

## Strategy Comparison

| Strategy | Time (10K queries) | Quality | False Negative Risk | Best For |
|----------|-------------------|---------|--------------------|---------| 
| BM25 | 2 min | Medium | Low | Quick baseline, keyword-heavy domains |
| Dense (FAISS) | 10 min | High | Medium | Primary strategy for most use cases |
| Cross-encoder scored | 2-4 hours | Highest | Very low | High-stakes domains, final refinement |
| TAS-B | 3-5 hours | High | Low | When you want soft labels |
| Curriculum | 15 min/epoch | High | Medium | Training stability |
| LLM-verified | 8-12 hours | Highest | Lowest | Medical, legal (where errors are costly) |

---

## Common Pitfalls

1. **Mining from a different domain.** Negatives from Wikipedia are not hard for a medical corpus. Always mine from the same corpus you will deploy on.
2. **Not re-mining after training.** After fine-tuning, the model's similarity scores change. Yesterday's hard negatives may be today's easy negatives. Re-mine periodically.
3. **Using the same negatives for every epoch.** Static negatives become less informative over time. Curriculum or iterative mining keeps the training signal fresh.
4. **Skipping false negative filtering.** In specialized domains, 10-20% of dense-mined negatives may be false negatives. Always filter.
5. **Mining too many negatives.** Beyond 7-10 per query, the additional negatives provide marginal benefit but increase training time linearly.

---

## References

- ANCE (Approximate Nearest Neighbor Negative Contrastive Learning) -- https://arxiv.org/abs/2007.00808
- TAS-B -- https://arxiv.org/abs/2104.06967
- RocketQA -- https://arxiv.org/abs/2010.08191
- MarginMSE Loss -- https://www.sbert.net/docs/package_reference/losses.html#marginmseloss
- sentence-transformers Hard Negative Mining -- https://www.sbert.net/examples/training/ms_marco/README.html
