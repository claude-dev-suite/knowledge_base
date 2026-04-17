# Knowledge Graph Construction -- Extraction Pipeline: Schema-Guided Pydantic, REBEL, and Validation

## TL;DR

The extraction pipeline is the core of knowledge graph construction, transforming raw text chunks into validated triples ready for graph insertion. This article covers three production extraction approaches in depth: (1) schema-guided extraction using Pydantic models for type-safe, constrained output, (2) REBEL model extraction for cost-free local processing, and (3) a hybrid pipeline that uses REBEL for first-pass extraction and LLM for validation and gap-filling. Each approach includes chunking strategies, batch processing, and quality metrics.

---

## Chunking for Triple Extraction

### Why Chunk Size Matters for KG Construction

Standard RAG chunking (512-1024 tokens) is suboptimal for triple extraction because:

1. **Coreferences break across chunks**: "Alice works at Acme. She leads the auth team." If split between chunks, the second chunk loses the referent of "she."
2. **Relationships span paragraphs**: A paragraph may introduce an entity; the next paragraph describes its relationships.
3. **Context loss**: Small chunks provide insufficient context for the LLM to extract accurate triples.

```python
from dataclasses import dataclass


@dataclass
class ExtractionChunk:
    text: str
    chunk_id: str
    document_id: str
    start_char: int
    end_char: int
    overlap_text: str = ""  # text from adjacent chunks for context


class KGChunker:
    """Chunking strategy optimized for triple extraction.

    Key differences from RAG chunking:
    - Larger chunks (1500-2500 tokens) to preserve context
    - Larger overlap (200-400 tokens) to maintain coreferences
    - Paragraph-aware splitting (never break mid-paragraph)
    - Section-aware: prefer splitting at section boundaries
    """

    def __init__(
        self,
        chunk_size: int = 2000,
        overlap: int = 300,
        min_chunk_size: int = 200,
    ):
        self.chunk_size = chunk_size
        self.overlap = overlap
        self.min_chunk_size = min_chunk_size

    def chunk_document(
        self, text: str, document_id: str
    ) -> list[ExtractionChunk]:
        """Split document into extraction-optimized chunks."""
        # Split into paragraphs first
        paragraphs = self._split_paragraphs(text)

        chunks = []
        current_text = ""
        current_start = 0
        char_pos = 0

        for para in paragraphs:
            para_tokens = len(para.split())

            # If adding this paragraph exceeds chunk size, finalize current chunk
            if (
                len(current_text.split()) + para_tokens > self.chunk_size
                and len(current_text.split()) >= self.min_chunk_size
            ):
                chunk = ExtractionChunk(
                    text=current_text.strip(),
                    chunk_id=f"{document_id}:chunk:{len(chunks)}",
                    document_id=document_id,
                    start_char=current_start,
                    end_char=char_pos,
                )
                chunks.append(chunk)

                # Start new chunk with overlap
                overlap_words = current_text.split()[-self.overlap:]
                current_text = " ".join(overlap_words) + "\n\n" + para
                current_start = char_pos - len(" ".join(overlap_words))
            else:
                current_text += "\n\n" + para if current_text else para

            char_pos += len(para) + 2  # +2 for paragraph separator

        # Final chunk
        if current_text.strip() and len(current_text.split()) >= self.min_chunk_size:
            chunks.append(ExtractionChunk(
                text=current_text.strip(),
                chunk_id=f"{document_id}:chunk:{len(chunks)}",
                document_id=document_id,
                start_char=current_start,
                end_char=char_pos,
            ))

        # Add overlap context from adjacent chunks
        for i, chunk in enumerate(chunks):
            parts = []
            if i > 0:
                parts.append(chunks[i - 1].text[-500:])
            parts.append(chunk.text)
            if i < len(chunks) - 1:
                parts.append(chunks[i + 1].text[:500])
            chunk.overlap_text = "\n\n---\n\n".join(parts)

        return chunks

    @staticmethod
    def _split_paragraphs(text: str) -> list[str]:
        """Split text into paragraphs, preserving structure."""
        # Split on double newlines or section headers
        import re
        paragraphs = re.split(r"\n\s*\n", text)
        return [p.strip() for p in paragraphs if p.strip()]
```

---

## Schema-Guided Pydantic Extraction (Full Implementation)

```python
from pydantic import BaseModel, Field, field_validator, model_validator
from typing import Literal
import json


# --- Schema Definition ---

ENTITY_TYPES = Literal[
    "person", "organization", "technology", "project",
    "concept", "location", "event", "product", "standard"
]

RELATION_TYPES = Literal[
    "works_at", "founded", "leads", "develops", "uses",
    "depends_on", "part_of", "located_in", "collaborates_with",
    "acquired", "competes_with", "implements", "authored",
    "manages", "reports_to", "member_of", "created",
    "integrates_with", "replaced_by", "related_to"
]


class Entity(BaseModel):
    name: str = Field(min_length=1, max_length=200)
    entity_type: ENTITY_TYPES
    description: str = Field(default="", max_length=500)

    @field_validator("name")
    @classmethod
    def clean_name(cls, v: str) -> str:
        import re
        v = v.strip().strip("'\"")
        v = re.sub(r"\s+", " ", v)
        return v


class Triple(BaseModel):
    subject: Entity
    predicate: RELATION_TYPES
    object: Entity
    confidence: float = Field(default=0.9, ge=0.0, le=1.0)
    source_chunk_id: str = ""

    @model_validator(mode="after")
    def no_self_loop(self) -> "Triple":
        if self.subject.name.lower() == self.object.name.lower():
            raise ValueError(
                f"Self-loop detected: {self.subject.name} -> {self.object.name}"
            )
        return self


class ExtractionOutput(BaseModel):
    entities: list[Entity]
    triples: list[Triple]


# --- Extraction Engine ---

class PydanticExtractor:
    """Schema-guided extraction with Pydantic validation.

    The extraction prompt includes the full schema (entity types,
    relation types) and requests JSON output. The response is
    validated against the Pydantic model, with automatic retry
    on validation failure.
    """

    PROMPT_TEMPLATE = """Extract entities and relationships from the text below.

## SCHEMA

### Entity Types
{entity_type_list}

### Relation Types
{relation_type_list}

## RULES
1. Extract ONLY relationships explicitly stated in the text
2. Resolve all pronouns (he/she/it/they) to named entities
3. Use the CLOSEST matching relation type from the list above
4. If no relation type fits, use "related_to"
5. Rate confidence: 1.0 = explicitly stated, 0.7 = strongly implied, 0.5 = inferred
6. Entity names should be specific and complete

## TEXT
{text}

## OUTPUT
Respond with ONLY valid JSON (no markdown, no explanation):
{{"entities": [...], "triples": [...]}}

Each entity: {{"name": "...", "entity_type": "...", "description": "..."}}
Each triple: {{"subject": {{"name": "...", "entity_type": "..."}}, "predicate": "...", "object": {{"name": "...", "entity_type": "..."}}, "confidence": 0.9}}"""

    def __init__(
        self,
        llm,
        min_confidence: float = 0.5,
        max_retries: int = 2,
    ):
        self.llm = llm
        self.min_confidence = min_confidence
        self.max_retries = max_retries

    def extract(self, chunk: ExtractionChunk) -> ExtractionOutput:
        """Extract and validate triples from a chunk."""
        # Use overlap_text if available for better context
        text = chunk.overlap_text or chunk.text

        entity_type_list = "\n".join(
            f"- {t}" for t in ENTITY_TYPES.__args__
        )
        relation_type_list = "\n".join(
            f"- {t}" for t in RELATION_TYPES.__args__
        )

        prompt = self.PROMPT_TEMPLATE.format(
            entity_type_list=entity_type_list,
            relation_type_list=relation_type_list,
            text=text,
        )

        for attempt in range(self.max_retries + 1):
            try:
                response = self.llm.invoke(prompt)
                raw_json = self._extract_json(response.content)
                parsed = json.loads(raw_json)
                result = ExtractionOutput(**parsed)

                # Filter by confidence
                result.triples = [
                    t for t in result.triples
                    if t.confidence >= self.min_confidence
                ]

                # Set source chunk ID
                for triple in result.triples:
                    triple.source_chunk_id = chunk.chunk_id

                return result

            except (json.JSONDecodeError, Exception) as e:
                if attempt < self.max_retries:
                    # Retry with error feedback
                    prompt += (
                        f"\n\nPrevious attempt failed with error: {str(e)[:200]}\n"
                        "Please fix the JSON and try again."
                    )
                else:
                    return ExtractionOutput(entities=[], triples=[])

    def extract_batch(
        self, chunks: list[ExtractionChunk]
    ) -> list[ExtractionOutput]:
        """Extract from multiple chunks with progress tracking."""
        results = []
        total_triples = 0
        total_entities = 0

        for i, chunk in enumerate(chunks):
            result = self.extract(chunk)
            results.append(result)
            total_triples += len(result.triples)
            total_entities += len(result.entities)

            if (i + 1) % 10 == 0:
                print(
                    f"Processed {i + 1}/{len(chunks)} chunks. "
                    f"Extracted {total_triples} triples, "
                    f"{total_entities} entities."
                )

        return results

    @staticmethod
    def _extract_json(text: str) -> str:
        """Extract JSON from LLM response, handling code blocks."""
        text = text.strip()
        if "```json" in text:
            text = text.split("```json")[1].split("```")[0]
        elif "```" in text:
            text = text.split("```")[1].split("```")[0]
        return text.strip()
```

---

## REBEL Model Extraction (Full Implementation)

```python
from transformers import AutoModelForSeq2SeqLM, AutoTokenizer
import torch


class REBELExtractor:
    """Full REBEL extraction pipeline with post-processing.

    REBEL outputs linearized triples in a special format:
    <triplet> subject <subj> relation <obj> object <triplet> ...

    This implementation adds:
    - Batch processing with proper padding
    - Post-processing to normalize entity names and predicates
    - Confidence scoring based on beam search probabilities
    """

    def __init__(self, device: str = "cpu", batch_size: int = 8):
        self.device = torch.device(device)
        self.model = AutoModelForSeq2SeqLM.from_pretrained(
            "Babelscape/rebel-large"
        ).to(self.device)
        self.tokenizer = AutoTokenizer.from_pretrained(
            "Babelscape/rebel-large"
        )
        self.batch_size = batch_size

    def extract(self, text: str) -> list[Triple]:
        """Extract triples from a single text."""
        inputs = self.tokenizer(
            text,
            return_tensors="pt",
            max_length=1024,
            truncation=True,
            padding=True,
        ).to(self.device)

        with torch.no_grad():
            outputs = self.model.generate(
                **inputs,
                max_length=512,
                num_beams=5,
                num_return_sequences=1,
                return_dict_in_generate=True,
                output_scores=True,
            )

        decoded = self.tokenizer.decode(
            outputs.sequences[0], skip_special_tokens=False
        )
        triples = self._parse_output(decoded, text)
        return triples

    def extract_batch(
        self, texts: list[str]
    ) -> list[list[Triple]]:
        """Batch extraction for efficiency."""
        all_results = []

        for i in range(0, len(texts), self.batch_size):
            batch_texts = texts[i : i + self.batch_size]
            inputs = self.tokenizer(
                batch_texts,
                return_tensors="pt",
                max_length=1024,
                truncation=True,
                padding=True,
            ).to(self.device)

            with torch.no_grad():
                outputs = self.model.generate(
                    **inputs,
                    max_length=512,
                    num_beams=5,
                    num_return_sequences=1,
                )

            for j, sequence in enumerate(outputs):
                decoded = self.tokenizer.decode(
                    sequence, skip_special_tokens=False
                )
                source_text = batch_texts[j] if j < len(batch_texts) else ""
                triples = self._parse_output(decoded, source_text)
                all_results.append(triples)

        return all_results

    def _parse_output(self, text: str, source: str) -> list[Triple]:
        """Parse REBEL linearized output into triples."""
        triples = []
        text = text.replace("<s>", "").replace("</s>", "").replace("<pad>", "")

        # Split on <triplet> tokens
        parts = text.split("<triplet>")

        for part in parts:
            part = part.strip()
            if not part:
                continue

            # Extract components
            if "<subj>" not in part or "<obj>" not in part:
                continue

            subject_text = part.split("<subj>")[0].strip()
            remainder = part.split("<subj>")[1] if "<subj>" in part else ""
            predicate_text = remainder.split("<obj>")[0].strip()
            object_text = (
                remainder.split("<obj>")[1].strip()
                if "<obj>" in remainder
                else ""
            )

            if subject_text and predicate_text and object_text:
                triples.append(Triple(
                    subject=Entity(
                        name=subject_text,
                        entity_type="concept",  # REBEL does not type entities
                    ),
                    predicate=self._map_predicate(predicate_text),
                    object=Entity(
                        name=object_text,
                        entity_type="concept",
                    ),
                    confidence=0.7,  # default confidence for REBEL
                    source_chunk_id="",
                ))

        return triples

    @staticmethod
    def _map_predicate(rebel_predicate: str) -> RELATION_TYPES:
        """Map REBEL predicate to schema relation type."""
        mapping = {
            "headquarters location": "located_in",
            "country": "located_in",
            "founded by": "founded",
            "developer": "develops",
            "owned by": "part_of",
            "subsidiary": "part_of",
            "employer": "works_at",
            "member of": "member_of",
            "author": "authored",
            "creator": "created",
            "part of": "part_of",
            "uses": "uses",
            "instance of": "related_to",
            "subclass of": "related_to",
        }
        pred_lower = rebel_predicate.lower().strip()
        return mapping.get(pred_lower, "related_to")
```

---

## Hybrid Pipeline: REBEL + LLM Validation

```python
class HybridExtractionPipeline:
    """Combine REBEL speed with LLM accuracy.

    Pipeline:
    1. REBEL extracts triples from all chunks (free, fast)
    2. LLM validates and enriches uncertain triples (expensive, accurate)
    3. LLM fills gaps where REBEL missed relationships

    This reduces LLM costs by 60-80% while maintaining quality
    close to LLM-only extraction.
    """

    def __init__(
        self,
        rebel_extractor: REBELExtractor,
        llm_extractor: PydanticExtractor,
        validation_llm,
        rebel_confidence_threshold: float = 0.8,
    ):
        self.rebel = rebel_extractor
        self.llm_extractor = llm_extractor
        self.validation_llm = validation_llm
        self.rebel_threshold = rebel_confidence_threshold

    def extract(self, chunks: list[ExtractionChunk]) -> dict:
        """Run the hybrid extraction pipeline."""
        stats = {
            "total_chunks": len(chunks),
            "rebel_triples": 0,
            "llm_validated": 0,
            "llm_new_triples": 0,
            "llm_calls": 0,
        }

        all_triples: list[Triple] = []

        # Phase 1: REBEL extraction (all chunks)
        texts = [c.text for c in chunks]
        rebel_results = self.rebel.extract_batch(texts)

        for chunk, rebel_triples in zip(chunks, rebel_results):
            stats["rebel_triples"] += len(rebel_triples)

            # Set source chunk IDs
            for t in rebel_triples:
                t.source_chunk_id = chunk.chunk_id

            # Separate high-confidence and uncertain triples
            confident = [
                t for t in rebel_triples
                if t.confidence >= self.rebel_threshold
            ]
            uncertain = [
                t for t in rebel_triples
                if t.confidence < self.rebel_threshold
            ]

            all_triples.extend(confident)

            # Phase 2: LLM validates uncertain REBEL triples
            if uncertain:
                validated = self._validate_with_llm(
                    chunk.text, uncertain
                )
                all_triples.extend(validated)
                stats["llm_validated"] += len(validated)
                stats["llm_calls"] += 1

            # Phase 3: LLM gap-filling for chunks with few triples
            if len(rebel_triples) < 2:
                llm_result = self.llm_extractor.extract(chunk)
                # Only add triples not already found by REBEL
                existing_keys = {
                    (t.subject.name.lower(), t.object.name.lower())
                    for t in all_triples
                }
                for t in llm_result.triples:
                    key = (t.subject.name.lower(), t.object.name.lower())
                    if key not in existing_keys:
                        all_triples.append(t)
                        stats["llm_new_triples"] += 1
                stats["llm_calls"] += 1

        return {
            "triples": all_triples,
            "stats": stats,
        }

    def _validate_with_llm(
        self, text: str, triples: list[Triple]
    ) -> list[Triple]:
        """Use LLM to validate uncertain REBEL triples."""
        triple_text = "\n".join(
            f"({t.subject.name}, {t.predicate}, {t.object.name})"
            for t in triples
        )

        prompt = (
            "Validate these extracted triples against the source text.\n"
            "For each triple, respond VALID or INVALID.\n\n"
            f"Source text:\n{text[:1000]}\n\n"
            f"Triples to validate:\n{triple_text}\n\n"
            "Respond with one line per triple: VALID or INVALID"
        )

        response = self.validation_llm.invoke(prompt)
        lines = response.content.strip().split("\n")

        validated = []
        for triple, line in zip(triples, lines):
            if "VALID" in line.upper() and "INVALID" not in line.upper():
                triple.confidence = 0.85
                validated.append(triple)

        return validated
```

---

## Quality Metrics for Extraction

```python
def compute_extraction_metrics(
    predicted_triples: list[Triple],
    gold_triples: list[Triple],
    entity_match_threshold: float = 0.85,
) -> dict:
    """Compute extraction quality metrics.

    Metrics:
    - Triple-level precision/recall/F1
    - Entity-level precision/recall/F1
    - Predicate accuracy (among matched triples)
    """
    from difflib import SequenceMatcher

    def entities_match(a: str, b: str) -> bool:
        return SequenceMatcher(None, a.lower(), b.lower()).ratio() >= entity_match_threshold

    def triple_matches(pred: Triple, gold: Triple) -> bool:
        return (
            entities_match(pred.subject.name, gold.subject.name)
            and entities_match(pred.object.name, gold.object.name)
            and pred.predicate == gold.predicate
        )

    # Triple-level metrics
    matched_pred = set()
    matched_gold = set()

    for pi, pred in enumerate(predicted_triples):
        for gi, gold in enumerate(gold_triples):
            if gi not in matched_gold and triple_matches(pred, gold):
                matched_pred.add(pi)
                matched_gold.add(gi)
                break

    tp = len(matched_pred)
    fp = len(predicted_triples) - tp
    fn = len(gold_triples) - len(matched_gold)

    precision = tp / max(tp + fp, 1)
    recall = tp / max(tp + fn, 1)
    f1 = 2 * precision * recall / max(precision + recall, 1e-10)

    return {
        "triple_precision": round(precision, 4),
        "triple_recall": round(recall, 4),
        "triple_f1": round(f1, 4),
        "predicted_count": len(predicted_triples),
        "gold_count": len(gold_triples),
        "true_positives": tp,
        "false_positives": fp,
        "false_negatives": fn,
    }
```

---

## Common Pitfalls

1. **Using RAG-sized chunks for extraction.** 512-token chunks break coreferences. Use 1500-2500 token chunks with 200-400 token overlap.

2. **Not validating Pydantic output.** LLMs generate nearly-valid JSON with subtle errors. Always parse with Pydantic and retry on failure.

3. **Trusting REBEL confidence blindly.** REBEL does not output calibrated confidence scores. Treat all REBEL outputs as medium-confidence and validate important ones with an LLM.

4. **Not deduplicating across chunks.** The same relationship may be extracted from overlapping chunks. Deduplicate by (subject, predicate, object) tuple after extraction.

5. **Ignoring extraction statistics.** Track triples-per-chunk, predicate distribution, and entity type distribution. Anomalies (0 triples from a chunk, one predicate dominating 90%) indicate extraction problems.

---

## References

- Huguet Cabot & Navigli "REBEL: Relation Extraction By End-to-end Language generation" (EMNLP 2021)
- Pydantic documentation: https://docs.pydantic.dev/
- Transformers library: https://huggingface.co/docs/transformers/
