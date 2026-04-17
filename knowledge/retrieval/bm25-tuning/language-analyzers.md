# Non-English BM25: Language Analyzers and Tokenization

## Overview

BM25 operates on tokens, not raw text. The choice of language analyzer -- tokenizer, stemmer, stop-word filter, and normalization -- can affect retrieval quality more than tuning k1 and b. English analysis is well-understood (standard tokenizer + Porter stemmer + stop words), but other languages present unique challenges: compound words in German, inflection-rich morphology in Italian, no word boundaries in Chinese and Japanese, and diacritics in French and Spanish. This guide covers practical analyzer configuration for the six most commonly needed non-English languages, plus patterns for custom analyzers.

---

## How Analyzers Interact with BM25

BM25 scores are computed over analyzed (tokenized) terms. The analyzer determines:
1. **What counts as a token**: affects term frequency (f) and document frequency (df)
2. **Which tokens are indexed**: stop-word removal changes IDF values
3. **How tokens are normalized**: stemming conflates forms, affecting both TF and IDF
4. **Document length calculation**: |D| counts analyzed tokens, not raw characters

A poor analyzer can destroy retrieval quality even with perfect k1/b tuning.

---

## Italian

### Challenges
- Rich inflectional morphology: verbs have ~50 forms (parlo, parli, parla, parliamo, parlate, parlano, parlavo, ...)
- Articles, prepositions, and pronouns are often contracted: "dell'informazione" (of the information)
- Elision: "l'azienda" should match "azienda"

### Elasticsearch Built-In Analyzer

```json
PUT /italian_index
{
  "settings": {
    "analysis": {
      "analyzer": {
        "italian_custom": {
          "type": "custom",
          "tokenizer": "standard",
          "filter": [
            "lowercase",
            "italian_elision",
            "italian_stop",
            "italian_stemmer"
          ]
        }
      },
      "filter": {
        "italian_elision": {
          "type": "elision",
          "articles": ["c", "l", "all", "dall", "dell", "nell", "sull",
                       "coll", "pell", "gl", "agl", "dagl", "degl",
                       "negl", "sugl", "un", "m", "t", "s", "v", "d"]
        },
        "italian_stop": {
          "type": "stop",
          "stopwords": "_italian_"
        },
        "italian_stemmer": {
          "type": "stemmer",
          "language": "light_italian"
        }
      }
    }
  },
  "mappings": {
    "properties": {
      "content": {
        "type": "text",
        "analyzer": "italian_custom"
      }
    }
  }
}
```

### Stemming Options

| Stemmer | Aggressiveness | Example: "informazioni" | Notes |
|---|---|---|---|
| `light_italian` | Conservative | "informazion" | Recommended -- fewer false conflations |
| `italian` (Snowball) | Moderate | "inform" | May over-stem: "informare" (to inform) = "informazione" (information) |

### Python (NLTK)

```python
from nltk.stem.snowball import ItalianStemmer
from nltk.corpus import stopwords
import nltk

nltk.download("stopwords")

stemmer = ItalianStemmer()
stop_words = set(stopwords.words("italian"))

def analyze_italian(text):
    tokens = text.lower().split()
    # Handle elision
    cleaned = []
    for t in tokens:
        # Remove common elision patterns
        for prefix in ["l'", "d'", "un'", "all'", "dell'", "nell'", "sull'"]:
            if t.startswith(prefix):
                t = t[len(prefix):]
                break
        cleaned.append(t)
    # Remove stop words and stem
    return [stemmer.stem(t) for t in cleaned if t not in stop_words and len(t) > 1]

print(analyze_italian("l'informazione dell'azienda e la sua struttura"))
# ['informazion', 'aziend', 'struttur']
```

---

## French

### Challenges
- Elision: "l'homme", "j'ai", "qu'est-ce"
- Accented characters: e, e with acute, e with grave, e with circumflex should match intelligently
- Compound tenses and past participles: "mange, mangeait, mange, mangeant"

### Elasticsearch Analyzer

```json
PUT /french_index
{
  "settings": {
    "analysis": {
      "analyzer": {
        "french_custom": {
          "type": "custom",
          "tokenizer": "standard",
          "filter": [
            "lowercase",
            "french_elision",
            "french_stop",
            "french_stemmer"
          ]
        }
      },
      "filter": {
        "french_elision": {
          "type": "elision",
          "articles_case": true,
          "articles": ["l", "m", "t", "qu", "n", "s", "j", "d", "c",
                       "jusqu", "quoiqu", "lorsqu", "puisqu"]
        },
        "french_stop": {
          "type": "stop",
          "stopwords": "_french_"
        },
        "french_stemmer": {
          "type": "stemmer",
          "language": "light_french"
        }
      }
    }
  }
}
```

### Accent Handling

```json
{
  "filter": {
    "french_asciifolding": {
      "type": "asciifolding",
      "preserve_original": true
    }
  }
}
```

Setting `preserve_original: true` indexes both "resume" and "resume" (with accent), so a query without accents still matches accented text.

---

## German

### Challenges
- **Compound words**: German forms compound nouns that are single tokens: "Datenbankverbindungspooling" (database connection pooling). BM25 treats this as a single term, so a query for "Datenbank" will not match.
- Noun capitalization: "Datenbank" vs "datenbank"
- Umlauts: a with umlaut, o with umlaut, u with umlaut, sz ligature

### Compound Word Handling

This is the critical issue for German BM25. Without decompounding, recall drops significantly.

```json
PUT /german_index
{
  "settings": {
    "analysis": {
      "analyzer": {
        "german_custom": {
          "type": "custom",
          "tokenizer": "standard",
          "filter": [
            "lowercase",
            "german_normalization",
            "german_decompounder",
            "german_stop",
            "german_stemmer"
          ]
        }
      },
      "filter": {
        "german_normalization": {
          "type": "german_normalization"
        },
        "german_decompounder": {
          "type": "hyphenation_decompounder",
          "hyphenation_patterns_path": "analysis/de_DR.xml",
          "word_list": ["daten", "bank", "verbindung", "pool", "server",
                        "anwendung", "konfiguration", "verwaltung",
                        "sicherheit", "benutzer", "schnittstelle"],
          "min_word_size": 5,
          "min_subword_size": 3,
          "max_subword_size": 25
        },
        "german_stop": {
          "type": "stop",
          "stopwords": "_german_"
        },
        "german_stemmer": {
          "type": "stemmer",
          "language": "light_german"
        }
      }
    }
  }
}
```

**Dictionary-based decompounder** (more reliable):

```json
{
  "german_decompounder": {
    "type": "dictionary_decompounder",
    "word_list_path": "analysis/german_dictionary.txt",
    "min_word_size": 5,
    "min_subword_size": 3,
    "only_longest_match": false
  }
}
```

### German Normalization

The `german_normalization` filter handles:
- a-umlaut -> ae (alternative representation)
- o-umlaut -> oe
- u-umlaut -> ue
- sz-ligature -> ss

---

## Spanish

### Challenges
- Verb conjugations: 6 persons * multiple tenses = many forms
- Diminutives/augmentatives: "perro" -> "perrito" -> "perrazo"
- Accented vowels: a/e/i/o/u with acute accents

### Elasticsearch Analyzer

```json
PUT /spanish_index
{
  "settings": {
    "analysis": {
      "analyzer": {
        "spanish_custom": {
          "type": "custom",
          "tokenizer": "standard",
          "filter": [
            "lowercase",
            "spanish_stop",
            "spanish_stemmer"
          ]
        }
      },
      "filter": {
        "spanish_stop": {
          "type": "stop",
          "stopwords": "_spanish_"
        },
        "spanish_stemmer": {
          "type": "stemmer",
          "language": "light_spanish"
        }
      }
    }
  }
}
```

---

## Japanese

### Challenges
- **No word boundaries**: "東京大学に行きました" has no spaces between words
- Mixed scripts: kanji, hiragana, katakana, Latin characters
- Multiple readings for the same kanji

### Tokenization with Kuromoji

```json
PUT /japanese_index
{
  "settings": {
    "analysis": {
      "analyzer": {
        "japanese_custom": {
          "type": "custom",
          "tokenizer": "kuromoji_tokenizer",
          "filter": [
            "kuromoji_baseform",
            "kuromoji_part_of_speech",
            "ja_stop",
            "kuromoji_stemmer",
            "lowercase"
          ]
        }
      },
      "filter": {
        "kuromoji_part_of_speech": {
          "type": "kuromoji_part_of_speech",
          "stoptags": [
            "助詞-格助詞-一般",
            "助詞-終助詞"
          ]
        },
        "ja_stop": {
          "type": "stop",
          "stopwords": "_japanese_"
        }
      }
    }
  }
}
```

**Kuromoji tokenizer modes:**

| Mode | Behavior | Example: "関西国際空港" |
|---|---|---|
| `normal` | Standard segmentation | "関西国際空港" (one token) |
| `search` | Decompound for search | "関西", "国際", "空港", "関西国際空港" |
| `extended` | Unknown words split into unigrams | Same as search + fallback splitting |

For BM25, `search` mode is recommended because it generates both the compound and sub-compound tokens, improving recall for partial matches.

```json
{
  "tokenizer": {
    "kuromoji_search": {
      "type": "kuromoji_tokenizer",
      "mode": "search"
    }
  }
}
```

### Python (fugashi)

```python
import fugashi

tagger = fugashi.Tagger()

def analyze_japanese(text):
    tokens = []
    for word in tagger(text):
        # word.feature contains POS tags and base form
        base_form = word.feature.lemma or str(word)
        # Filter particles and auxiliaries
        if word.feature.pos1 not in ("助詞", "助動詞", "記号"):
            tokens.append(base_form)
    return tokens

print(analyze_japanese("東京大学でデータベースを研究しています"))
# ['東京大学', 'データベース', '研究']
```

---

## Chinese

### Challenges
- No word boundaries (like Japanese)
- No inflectional morphology (unlike European languages)
- Traditional vs simplified characters
- Single-character vs multi-character words: "人" (person) vs "人工智能" (artificial intelligence)

### Tokenization with IK Analyzer

```json
PUT /chinese_index
{
  "settings": {
    "analysis": {
      "analyzer": {
        "chinese_ik_smart": {
          "type": "custom",
          "tokenizer": "ik_smart"
        },
        "chinese_ik_max": {
          "type": "custom",
          "tokenizer": "ik_max_word"
        }
      }
    }
  },
  "mappings": {
    "properties": {
      "content": {
        "type": "text",
        "analyzer": "chinese_ik_max",
        "search_analyzer": "chinese_ik_smart"
      }
    }
  }
}
```

**IK tokenizer modes:**

| Mode | Behavior | Example: "中华人民共和国" |
|---|---|---|
| `ik_smart` | Coarsest segmentation | "中华人民共和国" |
| `ik_max_word` | Finest segmentation (all possible splits) | "中华人民共和国", "中华人民", "中华", "华人", "人民共和国", "人民", "共和国", "共和", "国" |

**Best practice**: use `ik_max_word` for indexing (maximum recall) and `ik_smart` for search queries (precision).

### Python (jieba)

```python
import jieba
import jieba.analyse

# Standard segmentation
def analyze_chinese(text):
    # jieba.cut returns a generator of tokens
    tokens = list(jieba.cut(text, cut_all=False))
    # Filter single-character tokens and punctuation
    stop_chars = set("的了在是我有和人这中大为上个国")
    return [t for t in tokens if len(t) > 1 and t not in stop_chars]

print(analyze_chinese("数据库连接池是管理数据库连接的重要组件"))
# ['数据库', '连接池', '管理', '数据库', '连接', '重要', '组件']

# Keyword extraction (useful for query analysis)
keywords = jieba.analyse.extract_tags("数据库连接池是管理数据库连接的重要组件", topK=5)
print(keywords)
# ['连接池', '数据库', '组件', '管理', '连接']
```

---

## Stemming vs Lemmatization

| Approach | Method | "running" | "better" | Pros | Cons |
|---|---|---|---|---|---|
| Stemming | Rule-based suffix stripping | "run" | "better" | Fast, no dictionary needed | Over-stems, does not handle irregulars |
| Lemmatization | Dictionary/morphological | "run" | "good" | Accurate, handles irregulars | Slower, needs language model |

### When to Use Which

- **Stemming**: good enough for most BM25 search. The errors (over-stemming, under-stemming) average out across many queries. All Elasticsearch built-in analyzers use stemming.
- **Lemmatization**: use when precision is critical (medical, legal) or when the language has highly irregular morphology. Requires spaCy, Stanza, or similar NLP library.

### Lemmatization with spaCy

```python
import spacy

# Install model: python -m spacy download it_core_news_sm
nlp = spacy.load("it_core_news_sm")

def lemmatize_italian(text):
    doc = nlp(text)
    return [
        token.lemma_.lower()
        for token in doc
        if not token.is_stop and not token.is_punct and len(token.text) > 1
    ]

print(lemmatize_italian("Le connessioni al database sono gestite dal pool"))
# ['connessione', 'database', 'gestire', 'pool']
```

---

## Custom Analyzers in Elasticsearch

### Multi-Language Index

When your corpus contains documents in multiple languages:

```json
PUT /multilingual_index
{
  "settings": {
    "analysis": {
      "analyzer": {
        "multilingual": {
          "type": "custom",
          "tokenizer": "icu_tokenizer",
          "filter": [
            "lowercase",
            "icu_folding",
            "multilingual_stop"
          ]
        }
      },
      "filter": {
        "multilingual_stop": {
          "type": "stop",
          "stopwords": ["the", "a", "an", "is", "are",
                        "il", "lo", "la", "le", "di",
                        "der", "die", "das", "ein", "eine",
                        "el", "la", "los", "las", "un",
                        "le", "la", "les", "un", "une"]
        }
      }
    }
  }
}
```

The `icu_tokenizer` handles Unicode text intelligently, including CJK characters, and the `icu_folding` filter normalizes accents and special characters.

### Per-Language Field Mapping

A better approach for multi-language corpora:

```json
{
  "mappings": {
    "properties": {
      "content_en": {
        "type": "text",
        "analyzer": "english"
      },
      "content_it": {
        "type": "text",
        "analyzer": "italian_custom"
      },
      "content_de": {
        "type": "text",
        "analyzer": "german_custom"
      },
      "content_ja": {
        "type": "text",
        "analyzer": "japanese_custom"
      }
    }
  }
}
```

Query with `multi_match` across language-specific fields:

```json
GET /multilingual_index/_search
{
  "query": {
    "multi_match": {
      "query": "database connection",
      "fields": ["content_en^2", "content_it", "content_de", "content_ja"],
      "type": "best_fields"
    }
  }
}
```

---

## Common Pitfalls

1. **Using the default analyzer for non-English text**: the `standard` analyzer only lowercases and splits on whitespace/punctuation. It does not stem, remove stop words, or handle language-specific features.

2. **Applying English stemming to non-English text**: the Porter stemmer will mangle non-English words. Always use the language-specific stemmer.

3. **Forgetting compound word handling for German**: without decompounding, German BM25 recall drops by 30-50%. This is the single most impactful analyzer choice for German.

4. **Not using a CJK-specific tokenizer**: standard tokenization produces individual characters for Chinese/Japanese text, which is useless for BM25. Use kuromoji (Japanese), IK/jieba (Chinese), or nori (Korean).

5. **Over-aggressive stop-word removal**: removing too many stop words can hurt phrase queries and reduce context for BM25's IDF calculation. Use the built-in language stop-word lists as a starting point.

6. **Mixing analyzers at index and search time**: if the index analyzer and search analyzer differ too much, terms will not match. The search analyzer should be equal to or simpler than the index analyzer.

---

## References

- Elasticsearch Language Analyzers: https://www.elastic.co/guide/en/elasticsearch/reference/current/analysis-lang-analyzer.html
- ICU Analysis Plugin: https://www.elastic.co/guide/en/elasticsearch/plugins/current/analysis-icu.html
- Kuromoji Plugin (Japanese): https://www.elastic.co/guide/en/elasticsearch/plugins/current/analysis-kuromoji.html
- IK Analysis Plugin (Chinese): https://github.com/medcl/elasticsearch-analysis-ik
- Snowball Stemmers: https://snowballstem.org/
- spaCy Language Models: https://spacy.io/models
