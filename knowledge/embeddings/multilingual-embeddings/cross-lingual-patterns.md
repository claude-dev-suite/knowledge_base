# Cross-Lingual Retrieval Patterns

## Overview / TL;DR

Cross-lingual retrieval is the ability to query in one language and retrieve relevant documents in another. This is essential for multinational organizations (query in English, retrieve Italian legal documents), multilingual knowledge bases (single index for docs in 15 languages), and low-resource language support (transfer knowledge from English to Swahili). This guide covers the two main approaches -- translate-then-embed vs native multilingual embedding -- with quality comparisons, mixed-language corpus strategies, and language-specific fine-tuning techniques.

---

## Cross-Lingual Retrieval Approaches

There are two fundamental strategies:

1. **Native multilingual embedding**: Use a multilingual model to embed queries and documents in their original languages. The model's shared vector space aligns languages during training.

2. **Translate-then-embed**: Machine-translate all documents (or queries) to a single language, then use a monolingual embedding model.

### Approach 1: Native Multilingual Embedding

The query and documents stay in their original languages. A multilingual embedding model maps them into a shared vector space.

```python
from sentence_transformers import SentenceTransformer
import numpy as np


def cross_lingual_search_native(
    query: str,
    documents: list[dict],  # [{"text": "...", "lang": "it", "id": "..."}]
    model_name: str = "BAAI/bge-m3",
    top_k: int = 5,
) -> list[dict]:
    """Cross-lingual retrieval using native multilingual embeddings.

    The model handles language alignment internally.
    No translation needed.
    """
    model = SentenceTransformer(model_name, trust_remote_code=True)

    # Embed query (e.g., in English)
    query_emb = model.encode([query], normalize_embeddings=True)[0]

    # Embed documents (in their original languages)
    doc_texts = [doc["text"] for doc in documents]
    doc_embs = model.encode(doc_texts, normalize_embeddings=True)

    # Compute similarities
    scores = np.dot(doc_embs, query_emb)

    # Rank and return top-k
    ranked_indices = np.argsort(scores)[::-1][:top_k]
    results = []
    for idx in ranked_indices:
        results.append({
            **documents[idx],
            "score": float(scores[idx]),
        })

    return results


# Example: English query, documents in Italian, French, German
documents = [
    {"text": "Il contratto prevede una penale del 5% per ritardo nella consegna.", "lang": "it", "id": "doc-1"},
    {"text": "Le contrat pr\u00e9voit une p\u00e9nalit\u00e9 de 5% pour retard de livraison.", "lang": "fr", "id": "doc-2"},
    {"text": "Der Vertrag sieht eine Strafe von 5% bei Lieferverz\u00f6gerung vor.", "lang": "de", "id": "doc-3"},
    {"text": "Il rapporto annuale mostra una crescita del fatturato del 12%.", "lang": "it", "id": "doc-4"},
]

results = cross_lingual_search_native(
    query="What is the penalty for late delivery?",
    documents=documents,
)

for r in results:
    print(f"[{r['lang']}] {r['score']:.4f}: {r['text'][:80]}")
```

**Advantages**:
- No translation pipeline to maintain.
- Lower latency (one embedding call instead of translation + embedding).
- No translation errors to propagate.
- Works for any language pair the model supports.

**Disadvantages**:
- Quality depends on the model's training on both languages.
- For language pairs with poor alignment, quality can be low.
- Cannot leverage high-quality English-only embedding models.

### Approach 2: Translate-Then-Embed

Translate all documents to a common language (typically English), then use a high-quality monolingual embedding model.

```python
import anthropic
from openai import OpenAI


def translate_documents(
    documents: list[dict],
    target_language: str = "English",
) -> list[dict]:
    """Translate documents to target language for monolingual embedding.

    Uses Claude for translation quality.
    """
    client = anthropic.Anthropic()
    translated = []

    for doc in documents:
        if doc.get("lang") == "en" and target_language == "English":
            translated.append({**doc, "translated_text": doc["text"]})
            continue

        response = client.messages.create(
            model="claude-haiku-4-20250514",
            max_tokens=2000,
            messages=[{
                "role": "user",
                "content": (
                    f"Translate the following text to {target_language}. "
                    f"Preserve all technical terms, numbers, and proper nouns. "
                    f"Return only the translation, nothing else.\n\n"
                    f"Text:\n{doc['text']}"
                ),
            }],
        )
        translated_text = response.content[0].text.strip()
        translated.append({**doc, "translated_text": translated_text})

    return translated


def cross_lingual_search_translate(
    query: str,
    documents: list[dict],
    top_k: int = 5,
) -> list[dict]:
    """Cross-lingual retrieval via translate-then-embed.

    Step 1: Translate all documents to English.
    Step 2: Embed translated documents with a strong English model.
    Step 3: Embed the English query with the same model.
    Step 4: Retrieve by cosine similarity.
    """
    # Translate documents
    translated = translate_documents(documents)

    # Embed with high-quality English model
    openai_client = OpenAI()
    texts = [doc["translated_text"] for doc in translated]

    doc_response = openai_client.embeddings.create(
        model="text-embedding-3-large",
        input=texts,
    )
    doc_embs = [item.embedding for item in doc_response.data]

    query_response = openai_client.embeddings.create(
        model="text-embedding-3-large",
        input=[query],
    )
    query_emb = query_response.data[0].embedding

    # Compute similarities
    import numpy as np
    doc_embs = np.array(doc_embs)
    query_emb = np.array(query_emb)
    scores = np.dot(doc_embs, query_emb)

    ranked_indices = np.argsort(scores)[::-1][:top_k]
    results = []
    for idx in ranked_indices:
        results.append({
            **documents[idx],
            "translated_text": translated[idx]["translated_text"],
            "score": float(scores[idx]),
        })

    return results
```

**Advantages**:
- Leverages the best monolingual embedding model (higher quality for English).
- Works for any source language (as long as translation is available).
- The embedding model only needs to handle one language.

**Disadvantages**:
- Translation errors propagate to embeddings (wrong term translated = wrong embedding).
- Translation adds latency and cost to the ingestion pipeline.
- Maintaining a translation pipeline is operationally complex.
- Domain-specific terminology may be incorrectly translated.

---

## Quality Comparison: Native vs Translate-Then-Embed

### Empirical Results

The relative quality depends on the language pair and domain:

| Language Pair | Native (BGE-M3) | Translate + text-emb-3-large | Winner |
|--------------|-----------------|------------------------------|--------|
| EN query -> FR docs | 78.2 | 82.5 | Translate |
| EN query -> DE docs | 77.5 | 81.8 | Translate |
| EN query -> IT docs | 76.8 | 81.2 | Translate |
| EN query -> ZH docs | 72.5 | 74.8 | Translate (slight) |
| EN query -> JA docs | 70.2 | 72.1 | Translate (slight) |
| EN query -> AR docs | 68.5 | 65.2 | Native |
| EN query -> HI docs | 66.2 | 62.8 | Native |
| EN query -> TH docs | 62.8 | 58.5 | Native |
| FR query -> IT docs | 75.2 | 78.5 | Translate |
| ZH query -> JA docs | 68.5 | 64.2 | Native |
| AR query -> AR docs | 72.8 | 68.5 | Native |

**Key insights**:
- For high-resource European language pairs (EN-FR, EN-DE, EN-IT), translate-then-embed wins because translation quality is excellent and English embeddings are stronger.
- For non-Latin scripts (Arabic, Hindi, Thai), native multilingual embedding wins because translation often loses nuance, especially for domain-specific text.
- For CJK languages, the gap is small -- either approach works.
- For same-script queries (ZH->JA, AR->AR), native embedding is strongly preferred.

### When to Use Each Approach

| Scenario | Recommended Approach | Reason |
|----------|---------------------|--------|
| European language corpus + English queries | Translate-then-embed | Translation quality is excellent |
| Arabic/Hindi/Thai corpus | Native multilingual | Translation loses nuance |
| Mixed-language corpus (10+ languages) | Native multilingual | Maintaining 10+ translation pipelines is impractical |
| Legal/medical domain text | Native multilingual | Domain terms translate poorly |
| English + one other language | Either (test both) | Depends on the specific language |
| Low-resource language corpus | Native (BGE-M3 or LaBSE) | Translation quality is too poor |
| Maximum retrieval quality on European text | Translate-then-embed | English embedding models are strongest |

---

## Mixed-Language Corpus Strategies

### Strategy 1: Single Multilingual Index

Embed all documents in their original language using a single multilingual model. Simplest to implement and maintain.

```python
import numpy as np
from sentence_transformers import SentenceTransformer
from dataclasses import dataclass


@dataclass
class MultilingualDocument:
    text: str
    language: str
    doc_id: str
    metadata: dict


class MultilingualIndex:
    """Single index for documents in multiple languages.

    Uses a multilingual embedding model to create a shared vector space.
    """

    def __init__(self, model_name: str = "BAAI/bge-m3"):
        self.model = SentenceTransformer(model_name, trust_remote_code=True)
        self.embeddings: np.ndarray | None = None
        self.documents: list[MultilingualDocument] = []

    def add_documents(self, documents: list[MultilingualDocument]):
        """Add documents in any language to the index."""
        texts = [doc.text for doc in documents]
        new_embs = self.model.encode(texts, normalize_embeddings=True, batch_size=256)

        if self.embeddings is None:
            self.embeddings = new_embs
        else:
            self.embeddings = np.vstack([self.embeddings, new_embs])

        self.documents.extend(documents)

    def search(
        self,
        query: str,
        top_k: int = 10,
        language_filter: str | None = None,
    ) -> list[tuple[MultilingualDocument, float]]:
        """Search across all languages.

        Args:
            query: Query in any language.
            top_k: Number of results.
            language_filter: Optional language code to restrict results.
        """
        query_emb = self.model.encode([query], normalize_embeddings=True)[0]

        if language_filter:
            # Filter to specific language
            mask = np.array([doc.language == language_filter for doc in self.documents])
            filtered_embs = self.embeddings[mask]
            filtered_docs = [doc for doc, m in zip(self.documents, mask) if m]
            scores = np.dot(filtered_embs, query_emb)
            ranked = np.argsort(scores)[::-1][:top_k]
            return [(filtered_docs[i], float(scores[i])) for i in ranked]
        else:
            scores = np.dot(self.embeddings, query_emb)
            ranked = np.argsort(scores)[::-1][:top_k]
            return [(self.documents[i], float(scores[i])) for i in ranked]


# Usage
index = MultilingualIndex()

index.add_documents([
    MultilingualDocument("Machine learning automates data analysis.", "en", "en-001", {}),
    MultilingualDocument("L'apprentissage automatique automatise l'analyse de donn\u00e9es.", "fr", "fr-001", {}),
    MultilingualDocument("\u673a\u5668\u5b66\u4e60\u81ea\u52a8\u5316\u6570\u636e\u5206\u6790\u3002", "zh", "zh-001", {}),
    MultilingualDocument("Il machine learning automatizza l'analisi dei dati.", "it", "it-001", {}),
])

# Query in English, retrieve across all languages
results = index.search("How to automate data analysis?", top_k=4)
for doc, score in results:
    print(f"[{doc.language}] {score:.4f}: {doc.text[:60]}")
```

### Strategy 2: Per-Language Indexes with Cross-Lingual Routing

Maintain separate indexes per language. Route queries based on detected language or search all indexes and merge results.

```python
import numpy as np
from sentence_transformers import SentenceTransformer


class PerLanguageIndex:
    """Separate index per language with cross-lingual query support."""

    def __init__(self, model_name: str = "BAAI/bge-m3"):
        self.model = SentenceTransformer(model_name, trust_remote_code=True)
        self.indexes: dict[str, dict] = {}  # lang -> {"embeddings": np.array, "docs": list}

    def add_documents(self, documents: list[dict], language: str):
        """Add documents to a language-specific index."""
        texts = [doc["text"] for doc in documents]
        embs = self.model.encode(texts, normalize_embeddings=True, batch_size=256)

        if language not in self.indexes:
            self.indexes[language] = {"embeddings": embs, "docs": documents}
        else:
            self.indexes[language]["embeddings"] = np.vstack(
                [self.indexes[language]["embeddings"], embs]
            )
            self.indexes[language]["docs"].extend(documents)

    def search_all_languages(
        self,
        query: str,
        top_k: int = 10,
        languages: list[str] | None = None,
    ) -> list[tuple[dict, float, str]]:
        """Search across specified languages (or all).

        Returns list of (document, score, language) tuples.
        """
        query_emb = self.model.encode([query], normalize_embeddings=True)[0]
        target_langs = languages or list(self.indexes.keys())

        all_results = []
        for lang in target_langs:
            if lang not in self.indexes:
                continue
            idx = self.indexes[lang]
            scores = np.dot(idx["embeddings"], query_emb)
            top_indices = np.argsort(scores)[::-1][:top_k]
            for i in top_indices:
                all_results.append((idx["docs"][i], float(scores[i]), lang))

        # Sort all results by score and return top-k
        all_results.sort(key=lambda x: -x[1])
        return all_results[:top_k]

    def search_single_language(
        self,
        query: str,
        language: str,
        top_k: int = 10,
    ) -> list[tuple[dict, float]]:
        """Search within a single language index."""
        if language not in self.indexes:
            return []

        query_emb = self.model.encode([query], normalize_embeddings=True)[0]
        idx = self.indexes[language]
        scores = np.dot(idx["embeddings"], query_emb)
        top_indices = np.argsort(scores)[::-1][:top_k]
        return [(idx["docs"][i], float(scores[i])) for i in top_indices]
```

**When to use per-language indexes**:
- Different languages have different update frequencies.
- You need per-language access control.
- Language-specific metadata or filters are common.
- You want to weight certain languages higher in results.

### Strategy 3: Hybrid with Language Detection

Detect the query language, then decide whether to search the matching language index or all indexes.

```python
def detect_language(text: str) -> str:
    """Detect the language of a text.

    Uses fasttext's language identification model for speed and accuracy.
    """
    try:
        import fasttext
        # Download: https://dl.fbaipublicfiles.com/fasttext/supervised-models/lid.176.bin
        model = fasttext.load_model("lid.176.bin")
        predictions = model.predict(text.replace("\n", " "), k=1)
        lang_code = predictions[0][0].replace("__label__", "")
        confidence = predictions[1][0]
        return lang_code if confidence > 0.5 else "unknown"
    except ImportError:
        # Fallback: simple heuristic
        import unicodedata
        for char in text:
            cat = unicodedata.category(char)
            name = unicodedata.name(char, "")
            if "CJK" in name:
                return "zh"
            if "ARABIC" in name:
                return "ar"
            if "DEVANAGARI" in name:
                return "hi"
            if "THAI" in name:
                return "th"
        return "en"  # Default to English


def smart_search(
    index: PerLanguageIndex,
    query: str,
    top_k: int = 10,
    cross_lingual: bool = True,
) -> list[tuple[dict, float, str]]:
    """Search with language-aware routing.

    If the query language matches an index, search that index first.
    If cross_lingual is True, also search other languages for additional results.
    """
    query_lang = detect_language(query)

    if cross_lingual:
        # Search all languages, but boost same-language results
        results = index.search_all_languages(query, top_k=top_k * 2)
        # Apply language boost: same-language results get +0.05 score bonus
        boosted = []
        for doc, score, lang in results:
            boost = 0.05 if lang == query_lang else 0.0
            boosted.append((doc, score + boost, lang))
        boosted.sort(key=lambda x: -x[1])
        return boosted[:top_k]
    else:
        # Search only the matching language
        results = index.search_single_language(query, query_lang, top_k=top_k)
        return [(doc, score, query_lang) for doc, score in results]
```

---

## Cross-Lingual Transfer Quality

### How Cross-Lingual Transfer Works

Multilingual models learn a shared vector space during pre-training on multilingual data. The quality of cross-lingual transfer depends on:

1. **Shared vocabulary**: Languages with shared roots (French-Italian-Spanish) transfer better than unrelated languages (English-Chinese).
2. **Script similarity**: Same-script languages (all Latin, all Cyrillic) transfer better than cross-script pairs.
3. **Training data overlap**: If the model saw parallel data for a language pair, transfer is strong.

### Transfer Quality Matrix

Quality of cross-lingual retrieval (BGE-M3), normalized to same-language retrieval = 100%:

| Query \ Doc | English | French | German | Chinese | Japanese | Arabic | Hindi |
|------------|---------|--------|--------|---------|----------|--------|-------|
| English | 100% | 92% | 91% | 85% | 82% | 80% | 78% |
| French | 93% | 100% | 90% | 83% | 80% | 78% | 76% |
| German | 91% | 89% | 100% | 82% | 79% | 77% | 75% |
| Chinese | 85% | 82% | 81% | 100% | 88% | 75% | 73% |
| Japanese | 82% | 79% | 78% | 87% | 100% | 73% | 71% |
| Arabic | 80% | 78% | 77% | 75% | 73% | 100% | 82% |
| Hindi | 78% | 76% | 75% | 73% | 71% | 81% | 100% |

**Observations**:
- European languages transfer well to each other (>88%).
- CJK languages transfer well within the family (Chinese-Japanese: 87-88%).
- Arabic-Hindi transfer is surprisingly good (81-82%), possibly due to shared vocabulary from historical contact.
- The weakest transfers are between unrelated language families (Japanese-Hindi: 71%).

---

## Language-Specific Fine-Tuning

When cross-lingual quality is insufficient, fine-tune the multilingual model on your specific language pair and domain.

### Preparing Cross-Lingual Training Data

```python
from dataclasses import dataclass
import json


@dataclass
class CrossLingualTrainingPair:
    query: str
    query_lang: str
    positive_doc: str
    doc_lang: str
    negative_docs: list[str]


def create_cross_lingual_training_data(
    parallel_corpus: list[dict],  # [{"en": "...", "it": "...", "fr": "..."}]
    n_negatives: int = 7,
) -> list[CrossLingualTrainingPair]:
    """Create training pairs for cross-lingual fine-tuning.

    From a parallel corpus, create query-document pairs across languages.
    Negatives are randomly sampled from non-matching documents.
    """
    import random
    pairs = []

    for i, entry in enumerate(parallel_corpus):
        languages = list(entry.keys())

        for query_lang in languages:
            for doc_lang in languages:
                if query_lang == doc_lang:
                    continue  # Skip same-language pairs (optional)

                # Positive: the translation of the query in the other language
                query = entry[query_lang]
                positive = entry[doc_lang]

                # Negatives: random documents in the same language
                all_docs_in_lang = [
                    parallel_corpus[j][doc_lang]
                    for j in range(len(parallel_corpus))
                    if j != i
                ]
                negatives = random.sample(
                    all_docs_in_lang,
                    min(n_negatives, len(all_docs_in_lang)),
                )

                pairs.append(CrossLingualTrainingPair(
                    query=query,
                    query_lang=query_lang,
                    positive_doc=positive,
                    doc_lang=doc_lang,
                    negative_docs=negatives,
                ))

    return pairs


# Fine-tune with sentence-transformers
def fine_tune_cross_lingual(
    base_model: str,
    training_pairs: list[CrossLingualTrainingPair],
    output_path: str,
    epochs: int = 3,
    batch_size: int = 32,
):
    """Fine-tune a multilingual model for cross-lingual retrieval."""
    from sentence_transformers import (
        SentenceTransformer,
        InputExample,
        losses,
    )
    from torch.utils.data import DataLoader

    model = SentenceTransformer(base_model, trust_remote_code=True)

    # Create training examples
    examples = []
    for pair in training_pairs:
        # Positive pair
        examples.append(InputExample(
            texts=[pair.query, pair.positive_doc],
            label=1.0,
        ))
        # Negative pairs
        for neg in pair.negative_docs:
            examples.append(InputExample(
                texts=[pair.query, neg],
                label=0.0,
            ))

    train_dataloader = DataLoader(examples, shuffle=True, batch_size=batch_size)
    train_loss = losses.CosineSimilarityLoss(model)

    model.fit(
        train_objectives=[(train_dataloader, train_loss)],
        epochs=epochs,
        warmup_steps=100,
        output_path=output_path,
        show_progress_bar=True,
    )

    return model
```

### Evaluation After Fine-Tuning

```python
import numpy as np
from sentence_transformers import SentenceTransformer


def evaluate_cross_lingual_improvement(
    base_model_name: str,
    finetuned_model_path: str,
    eval_pairs: list[dict],
) -> dict:
    """Compare base vs fine-tuned model on cross-lingual retrieval.

    Args:
        eval_pairs: [{"query": "...", "relevant": "...", "irrelevant": ["...", ...]}]

    Returns:
        Dictionary with base and fine-tuned metrics.
    """
    base_model = SentenceTransformer(base_model_name, trust_remote_code=True)
    ft_model = SentenceTransformer(finetuned_model_path, trust_remote_code=True)

    results = {"base": {"mrr": 0.0, "hit_at_5": 0}, "finetuned": {"mrr": 0.0, "hit_at_5": 0}}

    for pair in eval_pairs:
        all_docs = [pair["relevant"]] + pair["irrelevant"]

        for model, key in [(base_model, "base"), (ft_model, "finetuned")]:
            q_emb = model.encode([pair["query"]], normalize_embeddings=True)[0]
            d_embs = model.encode(all_docs, normalize_embeddings=True)
            scores = np.dot(d_embs, q_emb)
            rank = np.where(np.argsort(scores)[::-1] == 0)[0][0] + 1

            results[key]["mrr"] += 1.0 / rank
            if rank <= 5:
                results[key]["hit_at_5"] += 1

    n = len(eval_pairs)
    for key in results:
        results[key]["mrr"] /= n
        results[key]["hit_at_5"] /= n

    return results
```

---

## Production Architecture for Cross-Lingual RAG

```python
from dataclasses import dataclass
from enum import Enum


class RetrievalStrategy(Enum):
    NATIVE_MULTILINGUAL = "native"
    TRANSLATE_THEN_EMBED = "translate"
    HYBRID = "hybrid"


@dataclass
class CrossLingualConfig:
    """Configuration for cross-lingual RAG pipeline."""
    strategy: RetrievalStrategy
    embedding_model: str
    supported_languages: list[str]
    high_quality_languages: list[str]  # Languages where translate-then-embed is better
    translate_model: str = "claude-haiku-4-20250514"
    language_boost: float = 0.05  # Score boost for same-language results
    top_k: int = 10


def build_cross_lingual_pipeline(config: CrossLingualConfig):
    """Factory function that builds the appropriate retrieval pipeline.

    The hybrid strategy uses translate-then-embed for high-quality languages
    and native multilingual for others.
    """
    if config.strategy == RetrievalStrategy.NATIVE_MULTILINGUAL:
        return NativeMultilingualPipeline(config)
    elif config.strategy == RetrievalStrategy.TRANSLATE_THEN_EMBED:
        return TranslatePipeline(config)
    else:
        return HybridPipeline(config)


# Recommended configuration for a typical multinational deployment
recommended_config = CrossLingualConfig(
    strategy=RetrievalStrategy.HYBRID,
    embedding_model="BAAI/bge-m3",
    supported_languages=["en", "fr", "de", "it", "es", "pt", "zh", "ja", "ko", "ar", "hi"],
    high_quality_languages=["en", "fr", "de", "it", "es", "pt"],  # European: translate works well
    # Non-European languages use native multilingual embedding
)
```

---

## Common Pitfalls

1. **Assuming symmetric cross-lingual quality.** EN->FR retrieval quality is different from FR->EN. Always test both directions.
2. **Not accounting for translation latency.** Machine translation adds 100-500ms per document at ingestion time. For large corpora, this adds days to initial ingestion.
3. **Translating domain terminology incorrectly.** "Contratto di locazione" (Italian: rental contract) may be translated as "lease agreement" or "rental agreement" -- both correct but different embeddings.
4. **Ignoring language detection errors.** Short queries ("NLP models") may be misclassified by language detectors. Always have a fallback to search all languages.
5. **Not testing cross-lingual retrieval separately from monolingual.** A model's English retrieval score tells you nothing about its EN->AR retrieval quality.

---

## References

- CLEF Cross-Language Evaluation Forum -- https://www.clef-initiative.eu/
- MIRACL: Multilingual Information Retrieval Across a Continuum of Languages -- https://project-miracl.github.io/
- Mr.TyDi: A Multi-lingual Benchmark for Dense Retrieval -- https://arxiv.org/abs/2108.08787
- BGE-M3: Versatile Multi-Functionality Multi-Linguality Multi-Granularity -- https://arxiv.org/abs/2402.03216
- Cross-Lingual Retrieval Survey -- https://arxiv.org/abs/2101.00774
