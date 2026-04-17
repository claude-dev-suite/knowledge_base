# Practical BM25 k1 and b Tuning Guide

## Overview

BM25 has two primary tuning parameters: k1 (term frequency saturation, default 1.2) and b (document length normalization, default 0.75). The defaults work reasonably well for general web search, but per-collection tuning can improve MRR@10 by 5-20% on specialized corpora. This guide covers what each parameter controls, how to tune them systematically, and how to set them in Elasticsearch, OpenSearch, and Lucene.

---

## What k1 Controls

k1 determines how quickly term frequency saturates. At the extremes:
- **k1 = 0**: term frequency is completely ignored. Only term presence/absence matters (binary model).
- **k1 -> infinity**: no saturation. The score scales linearly with term frequency (bag-of-words TF).

### Effect on Ranking

```
TF_component(f, k1, norm) = f * (k1 + 1) / (f + k1 * norm)
```

For a document with norm=1 (average length):

| TF (occurrences) | k1=0 | k1=0.5 | k1=1.2 | k1=2.0 | k1=5.0 |
|---|---|---|---|---|---|
| 0 | 0 | 0 | 0 | 0 | 0 |
| 1 | 1.0 | 0.67 | 0.91 | 1.0 | 1.0 |
| 2 | 1.0 | 0.80 | 1.25 | 1.50 | 1.67 |
| 3 | 1.0 | 0.86 | 1.41 | 1.80 | 2.25 |
| 5 | 1.0 | 0.91 | 1.56 | 2.14 | 3.00 |
| 10 | 1.0 | 0.95 | 1.70 | 2.50 | 4.00 |
| 20 | 1.0 | 0.98 | 1.83 | 2.73 | 4.80 |

### When to Increase k1 (> 1.2)

- **Long documents**: when documents are thousands of words, a term appearing 10 times may genuinely indicate higher relevance than appearing once
- **Verbose queries**: multi-sentence queries where term frequency in the document correlates with topical depth
- **Technical documentation**: API docs where a function name appearing frequently may indicate the primary topic

### When to Decrease k1 (< 1.2)

- **Short documents**: tweets, product titles, metadata fields -- term frequency rarely exceeds 2-3
- **Precision-oriented tasks**: when you want exact-match behavior (presence matters, count does not)
- **Navigational queries**: "python documentation" -- the user wants a specific page, not one that mentions "python" 50 times

---

## What b Controls

b determines how much document length affects scoring. It interpolates between two extremes:
- **b = 0**: no length normalization. A 10,000-word document has the same baseline TF scoring as a 100-word document.
- **b = 1.0**: full normalization. TF is divided by the ratio of document length to average length.

### Effect on Ranking

The normalization factor: `norm = 1 - b + b * (dl / avgdl)`

With avgdl = 500:

| Doc length | b=0 | b=0.25 | b=0.5 | b=0.75 | b=1.0 |
|---|---|---|---|---|---|
| 100 (0.2x avg) | 1.0 | 0.80 | 0.60 | 0.40 | 0.20 |
| 250 (0.5x avg) | 1.0 | 0.875 | 0.75 | 0.625 | 0.50 |
| 500 (1x avg) | 1.0 | 1.0 | 1.0 | 1.0 | 1.0 |
| 1000 (2x avg) | 1.0 | 1.25 | 1.50 | 1.75 | 2.0 |
| 2500 (5x avg) | 1.0 | 2.0 | 3.0 | 4.0 | 5.0 |

Higher norm values penalize long documents more (they appear in the denominator of the TF component).

### When to Increase b (> 0.75)

- **Heterogeneous document lengths**: when your corpus has a wide range of lengths (100-word abstracts mixed with 10,000-word papers), strong normalization prevents long documents from dominating
- **When long documents are not more relevant**: in web search, a page that is 10x longer is not necessarily 10x more relevant

### When to Decrease b (< 0.75)

- **Homogeneous document lengths**: when all documents are roughly the same length (e.g., product descriptions), length normalization adds noise
- **When long documents ARE more relevant**: comprehensive API reference pages, detailed technical guides
- **Fixed-length chunks**: if you have pre-chunked documents to ~500 tokens each, document length variation is minimal, so b should be near 0

---

## Grid Search Methodology

### Step 1: Prepare Evaluation Data

You need a set of queries with relevance judgments:

```python
# Minimum: 50-100 queries with at least one relevant document each
# Format: list of (query, list of relevant doc IDs)
eval_data = [
    ("python database connection", ["doc_42", "doc_187", "doc_531"]),
    ("HNSW index parameters", ["doc_89", "doc_204"]),
    ("kubernetes pod scheduling", ["doc_1023", "doc_1024", "doc_1025"]),
    # ... more queries
]
```

If you do not have relevance judgments, you can create them by:
1. Running BM25 with defaults and having humans judge the top-20 results
2. Using click-through data as weak labels
3. Using LLM-as-judge to generate relevance labels from query-document pairs

### Step 2: Define the Grid

```python
# Standard grid for BM25 tuning
k1_values = [0.4, 0.6, 0.8, 1.0, 1.2, 1.4, 1.6, 1.8, 2.0, 2.5, 3.0]
b_values = [0.0, 0.1, 0.2, 0.3, 0.4, 0.5, 0.6, 0.7, 0.75, 0.8, 0.9, 1.0]

# Total: 11 * 12 = 132 combinations
```

### Step 3: Run the Grid Search

```python
import math
from collections import Counter
from itertools import product

class BM25Evaluator:
    def __init__(self, corpus: list[dict], tokenizer):
        """
        corpus: list of {"id": str, "text": str}
        tokenizer: function that takes text and returns list of tokens
        """
        self.corpus = corpus
        self.tokenize = tokenizer
        self.tokenized = [self.tokenize(doc["text"]) for doc in corpus]
        self.N = len(corpus)
        self.avgdl = sum(len(doc) for doc in self.tokenized) / self.N
        self.doc_lens = [len(doc) for doc in self.tokenized]
        self.doc_tfs = [Counter(doc) for doc in self.tokenized]

        # Precompute document frequencies
        self.df = {}
        for doc in self.tokenized:
            for term in set(doc):
                self.df[term] = self.df.get(term, 0) + 1

    def idf(self, term):
        df = self.df.get(term, 0)
        return math.log(1 + (self.N - df + 0.5) / (df + 0.5))

    def score(self, query_tokens, doc_idx, k1, b):
        dl = self.doc_lens[doc_idx]
        tf_dict = self.doc_tfs[doc_idx]
        s = 0.0
        for term in query_tokens:
            tf = tf_dict.get(term, 0)
            if tf == 0:
                continue
            idf = self.idf(term)
            norm = 1 - b + b * (dl / self.avgdl)
            tf_component = (tf * (k1 + 1)) / (tf + k1 * norm)
            s += idf * tf_component
        return s

    def search(self, query_tokens, k1, b, top_k=20):
        scores = []
        for i in range(self.N):
            s = self.score(query_tokens, i, k1, b)
            if s > 0:
                scores.append((self.corpus[i]["id"], s))
        scores.sort(key=lambda x: x[1], reverse=True)
        return scores[:top_k]

    def evaluate_mrr(self, eval_data, k1, b, top_k=10):
        """Mean Reciprocal Rank at top_k."""
        reciprocal_ranks = []
        for query_text, relevant_ids in eval_data:
            query_tokens = self.tokenize(query_text)
            results = self.search(query_tokens, k1, b, top_k)
            rr = 0.0
            for rank, (doc_id, _) in enumerate(results, 1):
                if doc_id in relevant_ids:
                    rr = 1.0 / rank
                    break
            reciprocal_ranks.append(rr)
        return sum(reciprocal_ranks) / len(reciprocal_ranks)

    def evaluate_recall(self, eval_data, k1, b, top_k=20):
        """Recall at top_k."""
        recalls = []
        for query_text, relevant_ids in eval_data:
            query_tokens = self.tokenize(query_text)
            results = self.search(query_tokens, k1, b, top_k)
            retrieved_ids = {doc_id for doc_id, _ in results}
            recall = len(retrieved_ids & set(relevant_ids)) / len(relevant_ids)
            recalls.append(recall)
        return sum(recalls) / len(recalls)

    def grid_search(self, eval_data, k1_range, b_range, metric="mrr"):
        best_score = -1
        best_params = (1.2, 0.75)
        results = []

        for k1, b in product(k1_range, b_range):
            if metric == "mrr":
                score = self.evaluate_mrr(eval_data, k1, b)
            else:
                score = self.evaluate_recall(eval_data, k1, b)

            results.append({"k1": k1, "b": b, "score": score})

            if score > best_score:
                best_score = score
                best_params = (k1, b)

        return best_params, best_score, results


# Usage
import re

def simple_tokenizer(text):
    return re.findall(r'\w+', text.lower())

evaluator = BM25Evaluator(corpus, simple_tokenizer)

k1_range = [0.4, 0.6, 0.8, 1.0, 1.2, 1.4, 1.6, 1.8, 2.0]
b_range = [0.0, 0.1, 0.2, 0.3, 0.4, 0.5, 0.6, 0.7, 0.75, 0.8, 0.9, 1.0]

best_params, best_score, all_results = evaluator.grid_search(
    eval_data, k1_range, b_range, metric="mrr"
)

print(f"Best k1={best_params[0]}, b={best_params[1]}, MRR@10={best_score:.4f}")
```

### Step 4: Visualize Results

```python
import numpy as np
import matplotlib.pyplot as plt

# Create a heatmap of MRR@10 for each (k1, b) combination
k1_vals = sorted(set(r["k1"] for r in all_results))
b_vals = sorted(set(r["b"] for r in all_results))

score_matrix = np.zeros((len(k1_vals), len(b_vals)))
for r in all_results:
    i = k1_vals.index(r["k1"])
    j = b_vals.index(r["b"])
    score_matrix[i, j] = r["score"]

fig, ax = plt.subplots(figsize=(10, 8))
im = ax.imshow(score_matrix, cmap="YlOrRd", aspect="auto")
ax.set_xticks(range(len(b_vals)))
ax.set_xticklabels([f"{v:.2f}" for v in b_vals])
ax.set_yticks(range(len(k1_vals)))
ax.set_yticklabels([f"{v:.1f}" for v in k1_vals])
ax.set_xlabel("b (length normalization)")
ax.set_ylabel("k1 (TF saturation)")
ax.set_title("BM25 MRR@10 Grid Search")
plt.colorbar(im)
plt.tight_layout()
plt.savefig("bm25_grid_search.png", dpi=150)
```

---

## Optimal Values by Collection Type

Research across multiple IR test collections shows consistent patterns:

| Collection Type | Optimal k1 | Optimal b | Rationale |
|---|---|---|---|
| Web pages (general) | 1.2 | 0.75 | Default -- balanced for varied content |
| Short texts (tweets, titles) | 0.4-0.8 | 0.1-0.3 | Low TF, uniform length |
| Long documents (papers, books) | 1.5-2.5 | 0.5-0.8 | TF is more informative |
| Pre-chunked passages (~200 tokens) | 0.8-1.2 | 0.1-0.4 | Uniform length, moderate TF |
| Product descriptions | 0.6-1.0 | 0.2-0.5 | Short, structured, moderate variation |
| Code files | 1.2-2.0 | 0.3-0.6 | Repetitive identifiers, length varies |
| Q&A pairs (FAQ) | 0.4-0.8 | 0.0-0.3 | Very short, uniform length |
| Medical literature | 1.5-2.0 | 0.5-0.8 | Technical vocabulary, longer documents |
| Legal documents | 1.8-3.0 | 0.4-0.7 | Very long, term repetition matters |

---

## Setting Parameters in Search Engines

### Elasticsearch / OpenSearch

```json
// Index-level similarity setting
PUT /my_index
{
  "settings": {
    "index": {
      "similarity": {
        "custom_bm25": {
          "type": "BM25",
          "k1": 1.5,
          "b": 0.5,
          "discount_overlaps": true
        }
      }
    }
  },
  "mappings": {
    "properties": {
      "content": {
        "type": "text",
        "similarity": "custom_bm25"
      },
      "title": {
        "type": "text",
        "similarity": "custom_bm25"
      }
    }
  }
}
```

**Per-field similarity** (different k1/b for title vs body):

```json
PUT /my_index
{
  "settings": {
    "index": {
      "similarity": {
        "title_bm25": {
          "type": "BM25",
          "k1": 0.6,
          "b": 0.2
        },
        "body_bm25": {
          "type": "BM25",
          "k1": 1.5,
          "b": 0.75
        }
      }
    }
  },
  "mappings": {
    "properties": {
      "title": {
        "type": "text",
        "similarity": "title_bm25"
      },
      "body": {
        "type": "text",
        "similarity": "body_bm25"
      }
    }
  }
}
```

### Elasticsearch Python Client

```python
from elasticsearch import Elasticsearch

es = Elasticsearch("http://localhost:9200")

# Create index with custom BM25
es.indices.create(
    index="my_index",
    body={
        "settings": {
            "index": {
                "similarity": {
                    "custom_bm25": {
                        "type": "BM25",
                        "k1": 1.5,
                        "b": 0.5,
                    }
                }
            }
        },
        "mappings": {
            "properties": {
                "content": {
                    "type": "text",
                    "similarity": "custom_bm25",
                }
            }
        },
    },
)
```

### OpenSearch Python Client

```python
from opensearchpy import OpenSearch

client = OpenSearch(
    hosts=[{"host": "localhost", "port": 9200}],
    use_ssl=False,
)

client.indices.create(
    index="my_index",
    body={
        "settings": {
            "index": {
                "similarity": {
                    "custom_bm25": {
                        "type": "BM25",
                        "k1": 1.5,
                        "b": 0.5,
                    }
                }
            }
        },
        "mappings": {
            "properties": {
                "content": {
                    "type": "text",
                    "similarity": "custom_bm25",
                }
            }
        },
    },
)
```

### Apache Lucene (Java)

```java
import org.apache.lucene.search.similarities.BM25Similarity;
import org.apache.lucene.index.IndexWriterConfig;

// Set custom BM25 parameters
BM25Similarity similarity = new BM25Similarity(1.5f, 0.5f);

IndexWriterConfig config = new IndexWriterConfig(analyzer);
config.setSimilarity(similarity);

// Per-field similarity
PerFieldSimilarityWrapper perFieldSimilarity = new PerFieldSimilarityWrapper() {
    @Override
    public Similarity get(String fieldName) {
        if ("title".equals(fieldName)) {
            return new BM25Similarity(0.6f, 0.2f);
        }
        return new BM25Similarity(1.5f, 0.75f);  // default for other fields
    }
};
```

### Apache Solr

```xml
<!-- schema.xml or managed-schema -->
<similarity class="solr.SchemaSimilarityFactory">
  <str name="defaultSimFromFieldType">text_general</str>
</similarity>

<fieldType name="text_custom_bm25" class="solr.TextField">
  <analyzer>
    <tokenizer class="solr.StandardTokenizerFactory"/>
    <filter class="solr.LowerCaseFilterFactory"/>
  </analyzer>
  <similarity class="solr.BM25SimilarityFactory">
    <float name="k1">1.5</float>
    <float name="b">0.5</float>
  </similarity>
</fieldType>
```

---

## Tuning Workflow Summary

```
1. Index corpus with default BM25 (k1=1.2, b=0.75)

2. Collect/create evaluation queries with relevance judgments
   - Minimum: 50 queries
   - Ideal: 200+ queries covering diverse information needs

3. Compute baseline MRR@10 and Recall@20 with defaults

4. Run grid search over k1 and b
   - k1: [0.4, 0.6, 0.8, 1.0, 1.2, 1.4, 1.6, 1.8, 2.0]
   - b: [0.0, 0.1, 0.2, 0.3, 0.5, 0.75, 1.0]

5. Pick the (k1, b) that maximizes your primary metric
   - MRR@10 for precision (finding the best result first)
   - Recall@20 for RAG (finding all relevant passages)

6. Validate on held-out queries (split eval data 80/20)

7. Apply to production index
   - Elasticsearch: index-level similarity setting (requires reindex)
   - Lucene: IndexWriterConfig similarity
```

---

## Common Pitfalls

1. **Changing k1/b without reindexing**: in Elasticsearch, changing similarity settings requires creating a new index and reindexing. The similarity is baked into the index at write time.

2. **Tuning on too few queries**: with <20 queries, the grid search will overfit to the specific queries. Use at least 50, preferably 200+.

3. **Ignoring field-level tuning**: title fields should have lower k1 and b than body fields. A title is short and precise; the body is long and verbose.

4. **Not considering the analyzer**: k1 and b interact with the analyzer. If you change the analyzer (e.g., add stemming), re-tune k1 and b.

5. **Optimizing for the wrong metric**: MRR@10 and Recall@20 may favor different parameters. Choose the metric that matches your use case.

---

## References

- Robertson, S. and Zaragoza, H. "The Probabilistic Relevance Framework: BM25 and Beyond." 2009.
- Trotman, A. et al. "Improvements to BM25 and Language Models Examined." ADCS 2014.
- Elasticsearch Similarity Module: https://www.elastic.co/guide/en/elasticsearch/reference/current/index-modules-similarity.html
- Lipani, A. et al. "A Systematic Approach to Building Effective BM25 Indexes." SIGIR 2015.
- Yang, P. et al. "Critically Examining the 'Neural Hype': Weak Baselines and the Additivity of Effectiveness Gains from Neural Ranking Models." SIGIR 2019.
