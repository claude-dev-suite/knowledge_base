# BM25 Fundamentals

## Overview

BM25 (Best Matching 25) is a probabilistic ranking function that scores documents against a query based on term frequency, inverse document frequency, and document length normalization. Despite being proposed in 1994, BM25 remains the default scoring function in Elasticsearch, OpenSearch, Apache Solr, and Lucene. It is often the baseline that neural retrieval models must beat, and it frequently wins in scenarios involving exact term matching, specialized notation, or domains where training data for neural models is scarce.

Understanding BM25's formula and its two tuning parameters (k1 and b) is essential for anyone building search systems, whether you use BM25 alone, as a first-stage retriever, or in hybrid combination with vector search.

---

## The Full BM25 Formula

For a query Q containing terms q1, q2, ..., qn, the BM25 score of a document D is:

```
Score(Q, D) = SUM_{i=1}^{n}  IDF(qi) * [ f(qi, D) * (k1 + 1) ] / [ f(qi, D) + k1 * (1 - b + b * |D| / avgdl) ]
```

Where:
- `f(qi, D)` = frequency of term qi in document D (raw count)
- `|D|` = length of document D (in tokens/words)
- `avgdl` = average document length across the corpus
- `k1` = term frequency saturation parameter (default: 1.2)
- `b` = document length normalization parameter (default: 0.75)
- `IDF(qi)` = inverse document frequency of term qi

---

## Component-by-Component Breakdown

### Component 1: IDF (Inverse Document Frequency)

IDF measures how rare or informative a term is across the corpus. Rare terms are more discriminating:

```
IDF(qi) = ln( (N - n(qi) + 0.5) / (n(qi) + 0.5) + 1 )
```

Where:
- `N` = total number of documents in the corpus
- `n(qi)` = number of documents containing term qi

**Intuition**: "the" appears in nearly every document, so its IDF is close to 0. "pgvector" appears in very few documents, so its IDF is high.

**Example** (corpus of 10,000 documents):

| Term | Documents containing | IDF |
|---|---|---|
| "the" | 9,800 | 0.02 |
| "database" | 3,000 | 1.25 |
| "pgvector" | 50 | 5.29 |
| "HNSW" | 20 | 6.22 |

The Lucene/Elasticsearch variant uses a slightly different IDF formula:

```
IDF_lucene(qi) = ln(1 + (N - n(qi) + 0.5) / (n(qi) + 0.5))
```

This avoids negative IDF values for very common terms.

### Component 2: Term Frequency Saturation (k1)

The numerator and denominator together implement a saturating TF function:

```
TF_component = f(qi, D) * (k1 + 1) / (f(qi, D) + k1 * normalization_factor)
```

This is a key innovation over raw TF (term frequency). In raw TF, a document mentioning "database" 100 times scores 100x higher than one mentioning it once. This is rarely useful -- after a few mentions, additional occurrences add diminishing information.

The k1 parameter controls the saturation curve:

| f(qi, D) (occurrences) | k1=0.5 | k1=1.2 (default) | k1=2.0 | k1=100 (approaches raw TF) |
|---|---|---|---|---|
| 1 | 0.67 | 0.91 | 1.00 | 1.00 |
| 2 | 0.80 | 1.25 | 1.50 | 1.98 |
| 5 | 0.91 | 1.56 | 2.14 | 4.81 |
| 10 | 0.95 | 1.70 | 2.50 | 9.17 |
| 50 | 0.99 | 1.93 | 2.94 | 33.78 |
| 100 | 1.00 | 1.97 | 2.99 | 50.25 |

**Low k1 (0.5-0.8)**: aggressive saturation -- a second occurrence barely matters. Good for short, precise queries.

**Default k1 (1.2)**: moderate saturation -- a few occurrences matter, many do not. Works well for most corpora.

**High k1 (1.5-2.0)**: less saturation -- term frequency continues to matter. Better for long documents where term frequency is more informative.

### Component 3: Document Length Normalization (b)

The normalization factor in the denominator:

```
normalization = 1 - b + b * (|D| / avgdl)
```

This penalizes long documents (which naturally contain more term occurrences) and boosts short documents.

| b value | Effect |
|---|---|
| b = 0 | No length normalization. A 10,000-word document is not penalized vs a 100-word document. |
| b = 0.75 (default) | Moderate normalization. Long documents are penalized proportionally to how much longer they are than average. |
| b = 1.0 | Full normalization. A document twice the average length has its TF scores halved. |

**Example**: avgdl = 500 tokens

| Document length | b=0 | b=0.5 | b=0.75 (default) | b=1.0 |
|---|---|---|---|---|
| 100 tokens (short) | 1.0 | 0.60 | 0.40 | 0.20 |
| 500 tokens (average) | 1.0 | 1.0 | 1.0 | 1.0 |
| 1000 tokens (2x avg) | 1.0 | 1.50 | 1.75 | 2.0 |
| 2500 tokens (5x avg) | 1.0 | 3.0 | 4.0 | 5.0 |

Higher normalization values in the denominator reduce the TF score for long documents.

---

## Worked Example

Query: "python database connection"
Document: "Python provides several libraries for database connection pooling. The most popular Python database libraries include SQLAlchemy and psycopg2 for PostgreSQL database connections."

**Document stats**: 25 words, avgdl = 100 words

**Term frequencies in document**:
- "python": 2
- "database": 3
- "connection": 2

**Corpus stats** (N = 10,000):
- "python": in 1,500 docs -> IDF = ln(1 + (10000 - 1500 + 0.5)/(1500 + 0.5)) = ln(1 + 5.67) = 1.90
- "database": in 2,000 docs -> IDF = ln(1 + (10000 - 2000 + 0.5)/(2000 + 0.5)) = ln(1 + 4.00) = 1.61
- "connection": in 800 docs -> IDF = ln(1 + (10000 - 800 + 0.5)/(800 + 0.5)) = ln(1 + 11.50) = 2.53

**Using default k1=1.2, b=0.75**:

Normalization factor = 1 - 0.75 + 0.75 * (25/100) = 0.25 + 0.1875 = 0.4375

For "python" (f=2):
```
TF = 2 * (1.2 + 1) / (2 + 1.2 * 0.4375) = 4.4 / 2.525 = 1.743
Score_python = 1.90 * 1.743 = 3.31
```

For "database" (f=3):
```
TF = 3 * 2.2 / (3 + 1.2 * 0.4375) = 6.6 / 3.525 = 1.872
Score_database = 1.61 * 1.872 = 3.01
```

For "connection" (f=2):
```
TF = 2 * 2.2 / (2 + 1.2 * 0.4375) = 4.4 / 2.525 = 1.743
Score_connection = 2.53 * 1.743 = 4.41
```

**Total BM25 score = 3.31 + 3.01 + 4.41 = 10.73**

Notice "connection" contributes the most despite having the same TF as "python" -- its higher IDF (rarer term) makes it more discriminating.

---

## BM25 in Python (from Scratch)

```python
import math
from collections import Counter

class BM25:
    def __init__(self, corpus: list[list[str]], k1: float = 1.2, b: float = 0.75):
        """
        corpus: list of tokenized documents (each document is a list of tokens)
        """
        self.k1 = k1
        self.b = b
        self.corpus = corpus
        self.N = len(corpus)
        self.avgdl = sum(len(doc) for doc in corpus) / self.N

        # Precompute document frequencies
        self.df = {}  # term -> number of documents containing it
        for doc in corpus:
            for term in set(doc):
                self.df[term] = self.df.get(term, 0) + 1

        # Precompute IDF
        self.idf = {}
        for term, df in self.df.items():
            self.idf[term] = math.log(1 + (self.N - df + 0.5) / (df + 0.5))

        # Precompute term frequencies per document
        self.doc_tf = [Counter(doc) for doc in corpus]
        self.doc_lens = [len(doc) for doc in corpus]

    def score(self, query: list[str], doc_idx: int) -> float:
        """Score a single document against a query."""
        score = 0.0
        dl = self.doc_lens[doc_idx]
        tf_dict = self.doc_tf[doc_idx]

        for term in query:
            if term not in self.idf:
                continue

            tf = tf_dict.get(term, 0)
            idf = self.idf[term]

            norm = 1 - self.b + self.b * (dl / self.avgdl)
            tf_score = (tf * (self.k1 + 1)) / (tf + self.k1 * norm)

            score += idf * tf_score

        return score

    def search(self, query: list[str], k: int = 10) -> list[tuple[int, float]]:
        """Return top-k (doc_index, score) pairs."""
        scores = [(i, self.score(query, i)) for i in range(self.N)]
        scores.sort(key=lambda x: x[1], reverse=True)
        return scores[:k]


# Usage
corpus = [
    ["python", "database", "connection", "pooling", "sqlalchemy"],
    ["javascript", "react", "frontend", "component", "state"],
    ["python", "asyncio", "database", "postgres", "connection"],
    ["database", "index", "optimization", "query", "performance"],
]

bm25 = BM25(corpus, k1=1.2, b=0.75)
results = bm25.search(["python", "database", "connection"], k=3)

for doc_idx, score in results:
    print(f"Doc {doc_idx}: score={score:.3f} | {corpus[doc_idx]}")
```

---

## BM25 Variants

### BM25L

Addresses a flaw in standard BM25: very long documents are over-penalized because the length normalization can push the effective TF below useful levels. BM25L adds a floor:

```
TF_BM25L = (f(qi, D) + delta) * (k1 + 1) / (f(qi, D) + delta + k1 * norm)
```

Where delta = 0.5 (default). This ensures long documents are not excessively penalized for containing rare terms.

### BM25+

Similar motivation to BM25L but uses a different correction:

```
Score_BM25+(Q, D) = SUM IDF(qi) * [ TF_standard + delta ]
```

Adds delta (default 1.0) to the TF component, ensuring a non-zero lower bound.

### BM25F (for Fielded Documents)

Extends BM25 to documents with multiple fields (title, body, metadata):

```
f_combined(qi, D) = SUM_field  w_field * f(qi, D_field) / B_field
```

Where w_field is the field weight and B_field is a per-field length normalization. This is the standard in Elasticsearch for multi-field search.

---

## Common Pitfalls

1. **Using default parameters without evaluation**: k1=1.2 and b=0.75 are reasonable defaults, but they are not optimal for every corpus. Always evaluate on your data (see `k1-b-tuning.md`).

2. **Ignoring the analyzer**: BM25 operates on tokens, not raw text. The choice of tokenizer, stemmer, and stop-word filter dramatically affects quality (see `language-analyzers.md`).

3. **Comparing BM25 scores across different corpora**: BM25 scores are relative to a specific corpus (IDF depends on document frequencies in that corpus). A score of 15.0 in one index is not comparable to 15.0 in another.

4. **Not using BM25 as a baseline**: always establish BM25 performance before investing in neural retrieval. On some domains (exact matching, structured data), BM25 is hard to beat.

5. **Treating BM25 as obsolete**: modern hybrid search systems use BM25 alongside vector search. BM25 catches exact matches that embedding models miss (see `hybrid-patterns.md`).

6. **Applying stemming when exact matching matters**: stemming "running" to "run" helps recall but hurts precision when the exact form matters (e.g., code identifiers).

---

## References

- Robertson, S. and Zaragoza, H. "The Probabilistic Relevance Framework: BM25 and Beyond." Foundations and Trends in Information Retrieval, 2009.
- Elasticsearch BM25 documentation: https://www.elastic.co/guide/en/elasticsearch/reference/current/index-modules-similarity.html
- Lucene BM25 implementation: https://lucene.apache.org/core/9_0_0/core/org/apache/lucene/search/similarities/BM25Similarity.html
- Lv, Y. and Zhai, C. "Lower-Bounding Term Frequency Normalization." CIKM 2011 (BM25L).
- Trotman, A. et al. "Improvements to BM25 and Language Models Examined." ADCS 2014 (BM25+).
