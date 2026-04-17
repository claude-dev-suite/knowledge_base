# Knowledge Graph Construction -- LLM Triple Extraction for RAG

## TL;DR

Knowledge graph construction transforms unstructured text into structured (subject, predicate, object) triples that form a graph of entities and relationships. LLMs have dramatically improved extraction quality over traditional NLP pipelines, enabling schema-guided extraction with Pydantic validation, zero-shot triple extraction, and hybrid approaches combining LLMs with specialized models like REBEL. This overview covers the full construction pipeline: from text to triples, from triples to a validated knowledge graph, with production considerations for scale, cost, and quality.

---

## Why Build a Knowledge Graph for RAG

### Vector RAG vs Knowledge Graph RAG

Standard vector RAG treats documents as bags of text chunks. Knowledge graph construction extracts the semantic structure hidden in those chunks:

| Aspect | Vector Chunks | Knowledge Graph |
|--------|--------------|-----------------|
| Storage | "Alice works at Acme Corp on the Auth project" | (Alice, works_at, Acme Corp), (Alice, works_on, Auth project) |
| Query: "Who works at Acme?" | Requires chunk retrieval + LLM extraction | Direct graph query: MATCH (p)-[:works_at]->(Acme Corp) |
| Query: "What connects Alice to Bob?" | Cannot answer reliably | Path query through shared entities |
| Update: Alice leaves Acme | Re-embed entire chunk | Delete one triple |
| Explainability | "Found in chunk #47" | "Alice --works_at--> Acme Corp (source: doc.pdf, p.3)" |

### The Triple Extraction Problem

The core challenge is converting free text into structured triples:

```
Input:  "Dr. Sarah Chen leads the machine learning team at Anthropic,
         where she oversees the development of Claude."

Output: (Dr. Sarah Chen, leads, machine learning team)
        (Dr. Sarah Chen, works_at, Anthropic)
        (Anthropic, develops, Claude)
        (Dr. Sarah Chen, oversees_development_of, Claude)
```

This requires:
1. **Entity recognition**: identifying "Dr. Sarah Chen", "Anthropic", "Claude"
2. **Relation extraction**: identifying "leads", "works_at", "develops"
3. **Triple formation**: correctly pairing subjects with objects via predicates
4. **Coreference resolution**: knowing "she" refers to "Dr. Sarah Chen"

---

## LLM-Based Triple Extraction

### Zero-Shot Extraction

The simplest approach: prompt the LLM to extract triples directly.

```python
from dataclasses import dataclass


@dataclass
class Triple:
    subject: str
    predicate: str
    object: str
    confidence: float = 1.0
    source_text: str = ""


class ZeroShotExtractor:
    """Extract triples from text using LLM zero-shot prompting.

    Pros:
    - No training data required
    - Handles any domain
    - Good coreference resolution (LLM understands pronouns)

    Cons:
    - Expensive at scale (LLM call per chunk)
    - Inconsistent predicate naming (synonymous predicates)
    - May hallucinate relationships not in the text
    """

    EXTRACTION_PROMPT = """Extract all factual relationships from this text as structured triples.

Rules:
1. Each triple must be (subject, predicate, object)
2. Only extract relationships explicitly stated or strongly implied
3. Resolve pronouns to their antecedents
4. Use consistent, lowercase predicate names (e.g., "works_at", "founded_by")
5. Entity names should be complete (include titles, qualifiers)
6. Do not infer relationships not supported by the text

Text:
{text}

Output each triple on its own line in this exact format:
(subject, predicate, object)

Triples:"""

    def __init__(self, llm):
        self.llm = llm

    def extract(self, text: str) -> list[Triple]:
        """Extract triples from a text chunk."""
        prompt = self.EXTRACTION_PROMPT.format(text=text)
        response = self.llm.invoke(prompt)
        return self._parse_triples(response.content, text)

    def extract_batch(self, texts: list[str]) -> list[list[Triple]]:
        """Extract triples from multiple text chunks."""
        results = []
        for text in texts:
            triples = self.extract(text)
            results.append(triples)
        return results

    @staticmethod
    def _parse_triples(output: str, source_text: str) -> list[Triple]:
        """Parse LLM output into Triple objects."""
        triples = []
        for line in output.strip().split("\n"):
            line = line.strip()
            if not line.startswith("("):
                continue
            # Remove parentheses
            line = line.strip("()")
            parts = [p.strip().strip("'\"") for p in line.split(",", 2)]
            if len(parts) == 3 and all(parts):
                triples.append(Triple(
                    subject=parts[0],
                    predicate=parts[1],
                    object=parts[2],
                    source_text=source_text[:200],
                ))
        return triples
```

### Schema-Guided Extraction with Pydantic

Constrain extraction to a predefined schema for consistent, validated output.

```python
from pydantic import BaseModel, Field, field_validator
from typing import Literal
from enum import Enum


class EntityType(str, Enum):
    PERSON = "person"
    ORGANIZATION = "organization"
    TECHNOLOGY = "technology"
    PROJECT = "project"
    CONCEPT = "concept"
    LOCATION = "location"
    EVENT = "event"


class RelationType(str, Enum):
    WORKS_AT = "works_at"
    FOUNDED = "founded"
    LEADS = "leads"
    DEVELOPS = "develops"
    USES = "uses"
    DEPENDS_ON = "depends_on"
    PART_OF = "part_of"
    LOCATED_IN = "located_in"
    COLLABORATES_WITH = "collaborates_with"
    ACQUIRED = "acquired"
    COMPETES_WITH = "competes_with"
    IMPLEMENTS = "implements"
    RELATED_TO = "related_to"


class ExtractedEntity(BaseModel):
    name: str = Field(description="The canonical name of the entity")
    entity_type: EntityType = Field(description="The type of entity")
    description: str = Field(
        default="",
        description="A one-sentence description of this entity",
    )

    @field_validator("name")
    @classmethod
    def name_not_empty(cls, v: str) -> str:
        if not v.strip():
            raise ValueError("Entity name cannot be empty")
        return v.strip()


class ExtractedTriple(BaseModel):
    subject: ExtractedEntity
    predicate: RelationType
    object: ExtractedEntity
    confidence: float = Field(
        default=1.0, ge=0.0, le=1.0,
        description="Confidence that this triple is correct",
    )


class ExtractionResult(BaseModel):
    triples: list[ExtractedTriple]
    entities: list[ExtractedEntity]


class SchemaGuidedExtractor:
    """Extract triples constrained to a Pydantic schema.

    The schema provides:
    1. Fixed entity types (prevents "miscellaneous" entities)
    2. Fixed relation types (prevents synonym proliferation)
    3. Validation (rejects malformed triples)
    4. Structured output (JSON, not free text)
    """

    SCHEMA_PROMPT = """Extract entities and relationships from the following text.

## Entity Types
{entity_types}

## Relation Types
{relation_types}

## Text
{text}

## Instructions
1. Identify all named entities and classify them using the entity types above
2. Extract relationships between entities using ONLY the relation types above
3. If a relationship does not fit any predefined type, use "related_to"
4. Rate your confidence for each triple (0.0 to 1.0)
5. Resolve all pronouns and coreferences

Respond with valid JSON matching this schema:
{{
    "entities": [
        {{"name": "...", "entity_type": "...", "description": "..."}}
    ],
    "triples": [
        {{
            "subject": {{"name": "...", "entity_type": "..."}},
            "predicate": "...",
            "object": {{"name": "...", "entity_type": "..."}},
            "confidence": 0.95
        }}
    ]
}}"""

    def __init__(self, llm, min_confidence: float = 0.5):
        self.llm = llm
        self.min_confidence = min_confidence

    def extract(self, text: str) -> ExtractionResult:
        """Extract schema-validated triples from text."""
        entity_types = "\n".join(
            f"- {t.value}: {t.name}" for t in EntityType
        )
        relation_types = "\n".join(
            f"- {t.value}" for t in RelationType
        )

        prompt = self.SCHEMA_PROMPT.format(
            entity_types=entity_types,
            relation_types=relation_types,
            text=text,
        )

        response = self.llm.invoke(prompt)

        # Parse and validate with Pydantic
        import json
        try:
            # Extract JSON from response (handle markdown code blocks)
            content = response.content
            if "```json" in content:
                content = content.split("```json")[1].split("```")[0]
            elif "```" in content:
                content = content.split("```")[1].split("```")[0]

            raw = json.loads(content)
            result = ExtractionResult(**raw)

            # Filter by confidence
            result.triples = [
                t for t in result.triples
                if t.confidence >= self.min_confidence
            ]

            return result

        except (json.JSONDecodeError, Exception) as e:
            # Fallback: return empty result
            return ExtractionResult(triples=[], entities=[])
```

### REBEL Model for Cost-Efficient Extraction

```python
class REBELExtractor:
    """Extract triples using the REBEL model (Babelscape).

    REBEL is a seq2seq model fine-tuned for relation extraction.
    It runs locally, costs nothing per extraction, and is fast.

    Pros:
    - Free (runs on GPU or CPU)
    - Fast (~100ms per chunk on GPU)
    - Trained on large relation extraction datasets

    Cons:
    - Limited to relations it was trained on
    - No schema customization
    - Lower accuracy than GPT-4/Claude on complex text
    - Does not handle coreference well
    """

    def __init__(self, device: str = "cpu"):
        from transformers import pipeline

        self.pipe = pipeline(
            "text2text-generation",
            model="Babelscape/rebel-large",
            device=0 if device == "cuda" else -1,
            max_length=512,
        )

    def extract(self, text: str) -> list[Triple]:
        """Extract triples using REBEL."""
        # REBEL expects text and outputs linearized triples
        outputs = self.pipe(
            text,
            max_length=512,
            num_beams=5,
            num_return_sequences=1,
        )

        raw_output = outputs[0]["generated_text"]
        return self._parse_rebel_output(raw_output, text)

    @staticmethod
    def _parse_rebel_output(output: str, source_text: str) -> list[Triple]:
        """Parse REBEL's linearized output format.

        REBEL outputs: <triplet> subject <subj> predicate <obj> object
        """
        triples = []
        current = {"subject": "", "predicate": "", "object": ""}
        state = "none"

        tokens = output.replace("<s>", "").replace("</s>", "").strip().split()

        for token in tokens:
            if token == "<triplet>":
                if current["subject"] and current["object"]:
                    triples.append(Triple(
                        subject=current["subject"].strip(),
                        predicate=current["predicate"].strip(),
                        object=current["object"].strip(),
                        source_text=source_text[:200],
                    ))
                current = {"subject": "", "predicate": "", "object": ""}
                state = "subject"
            elif token == "<subj>":
                state = "predicate"
            elif token == "<obj>":
                state = "object"
            else:
                if state in current:
                    current[state] += " " + token

        # Don't forget the last triple
        if current["subject"] and current["object"]:
            triples.append(Triple(
                subject=current["subject"].strip(),
                predicate=current["predicate"].strip(),
                object=current["object"].strip(),
                source_text=source_text[:200],
            ))

        return triples
```

---

## Triple Validation and Cleaning

```python
import re
from collections import Counter


class TripleValidator:
    """Validate and clean extracted triples before graph insertion.

    Validation checks:
    1. Entity names are non-empty and reasonable length
    2. Predicates are normalized (lowercase, underscored)
    3. No self-loops (subject == object)
    4. No duplicate triples
    5. Confidence meets minimum threshold
    6. Entity types are consistent across mentions
    """

    def __init__(
        self,
        min_entity_length: int = 2,
        max_entity_length: int = 100,
        min_confidence: float = 0.5,
    ):
        self.min_entity_len = min_entity_length
        self.max_entity_len = max_entity_length
        self.min_confidence = min_confidence

    def validate_and_clean(self, triples: list[Triple]) -> list[Triple]:
        """Run all validation checks and return clean triples."""
        cleaned = []
        seen = set()

        for triple in triples:
            # Normalize
            triple.subject = self._normalize_entity(triple.subject)
            triple.object = self._normalize_entity(triple.object)
            triple.predicate = self._normalize_predicate(triple.predicate)

            # Validate
            if not self._is_valid(triple):
                continue

            # Deduplicate
            key = (triple.subject, triple.predicate, triple.object)
            if key in seen:
                continue
            seen.add(key)

            cleaned.append(triple)

        return cleaned

    def _normalize_entity(self, name: str) -> str:
        """Normalize entity name."""
        # Strip whitespace and quotes
        name = name.strip().strip("'\"")
        # Collapse multiple spaces
        name = re.sub(r"\s+", " ", name)
        # Title case for proper nouns
        if name.islower() and len(name.split()) <= 3:
            name = name.title()
        return name

    def _normalize_predicate(self, predicate: str) -> str:
        """Normalize predicate to snake_case."""
        predicate = predicate.strip().lower()
        predicate = re.sub(r"[^a-z0-9\s_]", "", predicate)
        predicate = re.sub(r"\s+", "_", predicate)
        return predicate

    def _is_valid(self, triple: Triple) -> bool:
        """Check if a triple passes all validation rules."""
        # Non-empty
        if not triple.subject or not triple.predicate or not triple.object:
            return False

        # Length bounds
        if (
            len(triple.subject) < self.min_entity_len
            or len(triple.subject) > self.max_entity_len
        ):
            return False
        if (
            len(triple.object) < self.min_entity_len
            or len(triple.object) > self.max_entity_len
        ):
            return False

        # No self-loops
        if triple.subject.lower() == triple.object.lower():
            return False

        # Confidence threshold
        if triple.confidence < self.min_confidence:
            return False

        return True

    def get_predicate_stats(self, triples: list[Triple]) -> dict:
        """Analyze predicate usage for schema refinement."""
        predicates = [t.predicate for t in triples]
        counter = Counter(predicates)
        return {
            "total_unique_predicates": len(counter),
            "most_common": counter.most_common(20),
            "singleton_predicates": sum(1 for c in counter.values() if c == 1),
        }
```

---

## Cost Optimization

```python
def estimate_construction_cost(
    num_chunks: int,
    avg_chunk_tokens: int = 500,
    strategy: str = "llm_only",
    llm_cost_per_1k_input: float = 0.003,
    llm_cost_per_1k_output: float = 0.015,
) -> dict:
    """Estimate the cost of knowledge graph construction.

    Strategies:
    - llm_only: LLM extracts all triples (highest quality, highest cost)
    - rebel_only: REBEL model only (free, lower quality)
    - hybrid: REBEL first pass, LLM validates uncertain triples
    """
    if strategy == "llm_only":
        input_tokens = num_chunks * (avg_chunk_tokens + 200)  # chunk + prompt
        output_tokens = num_chunks * 300  # estimated output
        input_cost = (input_tokens / 1000) * llm_cost_per_1k_input
        output_cost = (output_tokens / 1000) * llm_cost_per_1k_output
        return {
            "strategy": "LLM only",
            "total_cost": round(input_cost + output_cost, 2),
            "llm_calls": num_chunks,
            "quality": "high",
        }

    elif strategy == "rebel_only":
        return {
            "strategy": "REBEL only",
            "total_cost": 0.0,
            "llm_calls": 0,
            "quality": "medium",
            "note": "Requires GPU for reasonable speed",
        }

    elif strategy == "hybrid":
        # REBEL extracts all, LLM validates ~30% uncertain ones
        llm_chunks = int(num_chunks * 0.3)
        input_tokens = llm_chunks * (avg_chunk_tokens + 400)
        output_tokens = llm_chunks * 200
        input_cost = (input_tokens / 1000) * llm_cost_per_1k_input
        output_cost = (output_tokens / 1000) * llm_cost_per_1k_output
        return {
            "strategy": "Hybrid (REBEL + LLM validation)",
            "total_cost": round(input_cost + output_cost, 2),
            "llm_calls": llm_chunks,
            "quality": "high (REBEL coverage + LLM accuracy)",
        }

    return {"error": f"Unknown strategy: {strategy}"}
```

---

## Common Pitfalls

1. **Predicate proliferation.** Without a schema, LLMs generate dozens of synonymous predicates ("works_at", "employed_by", "is_employee_of"). Use schema-guided extraction or post-hoc predicate normalization.

2. **Hallucinated triples.** LLMs sometimes infer relationships not stated in the text. Always include "Only extract explicitly stated relationships" in the prompt, and validate against source text.

3. **Coreference failures.** "She joined the team in 2023" requires knowing who "she" refers to. LLMs handle this well; REBEL does not. For REBEL pipelines, run coreference resolution first.

4. **Ignoring extraction cost at scale.** At $0.01/chunk, extracting from 100,000 chunks costs $1,000. Use the hybrid strategy (REBEL + LLM validation) to reduce costs by 70%.

5. **Not validating before insertion.** Self-loops, empty entities, and duplicate triples degrade graph quality. Always run validation before inserting into the graph database.

6. **Static schemas that do not evolve.** If your schema defines 10 relation types but the corpus uses 50 distinct relationships, you lose information. Periodically analyze predicate statistics and expand the schema.

---

## References

- Huguet Cabot & Navigli "REBEL: Relation Extraction By End-to-end Language generation" (EMNLP 2021)
- Pydantic: https://docs.pydantic.dev/
- LlamaIndex KG construction: https://docs.llamaindex.ai/en/stable/module_guides/indexing/property_graph_index/
- Edge et al. "From Local to Global: A Graph RAG Approach" (2024) -- entity extraction methodology
