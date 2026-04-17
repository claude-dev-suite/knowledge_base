# RAG Guardrails Implementation -- NeMo Guardrails, Guardrails AI, and Pydantic Structured Output

## TL;DR

This article covers three production frameworks for implementing RAG guardrails: NVIDIA NeMo Guardrails (Colang-based conversational rails with topical, safety, and fact-checking controls), Guardrails AI (validator-based approach with a hub of reusable validators and structured output enforcement), and Pydantic-based structured output (using constrained generation to force the LLM to produce grounded, cited, schema-conformant responses). Each framework takes a different approach to the same problem -- preventing the LLM from generating ungrounded or harmful content -- and they can be composed for defense in depth. The article includes full setup, configuration, and integration code for each.

---

## NVIDIA NeMo Guardrails

### Architecture

NeMo Guardrails uses Colang, a domain-specific language for defining conversational rails (constraints). Rails intercept the conversation at three points: input (before LLM), dialog (during turn management), and output (after LLM).

```
User message
    |
    v
[Input Rails]        <- Topic filtering, PII detection, jailbreak detection
    |
    v
[Dialog Rails]       <- Turn management, canonical forms
    |
    v
[LLM Generation]
    |
    v
[Output Rails]       <- Fact-checking, hallucination detection, moderation
    |
    v
Response to user
```

### Setup and Configuration

```python
# Directory structure for NeMo Guardrails config
# config/
#   config.yml           <- Main configuration
#   rails/
#     input.co           <- Input rails (Colang)
#     output.co          <- Output rails (Colang)
#     fact_check.co      <- Fact-checking rail
#   prompts/
#     fact_check.yml     <- Prompt templates
```

### config.yml

```yaml
# config/config.yml
models:
  - type: main
    engine: anthropic
    model: claude-sonnet-4-20250514

rails:
  input:
    flows:
      - check topic allowed
      - check jailbreak

  output:
    flows:
      - check facts
      - check hallucination

  config:
    fact_checking:
      provider: llm
      fallback_response: >
        I could not verify this information against the source documents.
        Please check the original sources directly.
```

### Input Rails (Colang)

```python
# config/rails/input.co -- Colang syntax

define user ask about allowed topic
  "How do I configure authentication?"
  "What are the API rate limits?"
  "Explain the deployment process"

define user ask about disallowed topic
  "What is the meaning of life?"
  "Write me a poem"
  "Tell me a joke"

define flow check topic allowed
  user ask about disallowed topic
  bot inform topic not supported
  stop

define bot inform topic not supported
  "I can only help with questions about our product documentation.
  Please ask a question related to the product."
```

### Fact-Checking Output Rail

```python
# config/rails/fact_check.co

define subflow check facts
  """Check that the bot response is grounded in the context."""
  $is_grounded = execute check_facts_action(
    response=$bot_message,
    context=$relevant_chunks
  )

  if not $is_grounded
    bot provide unverified disclaimer
    stop

define bot provide unverified disclaimer
  "I was unable to fully verify this response against the source
  documents. The information may be incomplete or inaccurate.
  Please refer to the original documentation."
```

### Python Integration

```python
from nemoguardrails import RailsConfig, LLMRails


class NeMoGuardedRAG:
    """RAG pipeline with NeMo Guardrails for input and output rails."""

    def __init__(
        self,
        config_path: str,
        retriever,
    ):
        self.config = RailsConfig.from_path(config_path)
        self.rails = LLMRails(self.config)
        self.retriever = retriever

        # Register custom actions
        self.rails.register_action(
            self._retrieve_context, name="retrieve_context"
        )
        self.rails.register_action(
            self._check_facts_action, name="check_facts_action"
        )

    async def _retrieve_context(self, query: str) -> list[str]:
        """Custom action: retrieve documents for the query."""
        docs = self.retriever.invoke(query)
        return [doc.page_content for doc in docs]

    async def _check_facts_action(
        self, response: str, context: list[str]
    ) -> bool:
        """Custom action: verify response is grounded in context."""
        # Use the LLM to check groundedness
        check_prompt = (
            "You are a fact-checker. Determine whether the following "
            "response is fully supported by the provided context.\n\n"
            f"Context:\n{chr(10).join(context)}\n\n"
            f"Response:\n{response}\n\n"
            "Answer with ONLY 'supported' or 'not supported'."
        )
        result = await self.rails.llm.agenerate(
            prompts=[check_prompt]
        )
        return "supported" in result.generations[0][0].text.lower()

    async def query(self, user_message: str) -> dict:
        """Process a query through guardrails."""
        # NeMo Guardrails handles the full flow:
        # input rails -> retrieval -> generation -> output rails
        response = await self.rails.generate_async(
            messages=[{"role": "user", "content": user_message}]
        )
        return {
            "answer": response["content"],
            "guardrails_applied": True,
        }


# Usage
import asyncio

async def main():
    rag = NeMoGuardedRAG(
        config_path="./config",
        retriever=my_retriever,
    )

    # Allowed topic -- passes input rails
    result = await rag.query("How do I configure API authentication?")
    print(result["answer"])

    # Disallowed topic -- blocked by input rails
    result = await rag.query("Tell me a joke about databases")
    print(result["answer"])
    # -> "I can only help with questions about our product documentation."

asyncio.run(main())
```

---

## Guardrails AI

### Architecture

Guardrails AI uses a validator-based approach. You define a schema (using RAIL XML or Pydantic) with validators attached to each field. The framework wraps the LLM call and validates the output, optionally re-prompting on failure.

### Setup

```python
# pip install guardrails-ai
# guardrails hub install hub://guardrails/detect_pii
# guardrails hub install hub://guardrails/provenance_llm
# guardrails hub install hub://guardrails/toxic_language

import guardrails as gd
from guardrails.hub import DetectPII, ProvenanceLLM, ToxicLanguage
from pydantic import BaseModel, Field
```

### Pydantic Schema with Validators

```python
from pydantic import BaseModel, Field
import guardrails as gd
from guardrails.hub import ProvenanceLLM, ToxicLanguage


class RAGResponse(BaseModel):
    """Schema for a guardrailed RAG response.

    Validators are attached to fields and run automatically
    after LLM generation. Failed validators trigger re-prompting
    or raise a ValidationError.
    """

    answer: str = Field(
        description="The answer to the user's question",
        json_schema_extra={
            "validators": [
                ProvenanceLLM(
                    validation_method="sentence",
                    llm_callable="gpt-4o-mini",
                    top_k=3,
                    on_fail="reask",
                ),
                ToxicLanguage(
                    validation_method="sentence",
                    on_fail="filter",
                ),
            ],
        },
    )

    confidence: float = Field(
        description="Confidence score from 0 to 1",
        ge=0.0,
        le=1.0,
    )

    sources: list[str] = Field(
        description="List of source document IDs cited",
        min_length=1,
    )

    caveats: str | None = Field(
        default=None,
        description="Any caveats or limitations of the answer",
    )


def guarded_rag_query(
    query: str,
    context: str,
    document_ids: list[str],
) -> dict:
    """Execute a guardrailed RAG query using Guardrails AI."""
    guard = gd.Guard.from_pydantic(RAGResponse)

    prompt = (
        "You are a documentation assistant. Answer the question "
        "based on the provided context.\n\n"
        f"Context:\n{context}\n\n"
        f"Available document IDs: {document_ids}\n\n"
        f"Question: {query}\n\n"
        "Respond with a JSON object matching the required schema."
    )

    result = guard(
        model="claude-sonnet-4-20250514",
        messages=[{"role": "user", "content": prompt}],
        metadata={
            "sources": context,  # Needed by ProvenanceLLM
        },
        num_reasks=2,  # Retry up to 2 times on validation failure
    )

    if result.validation_passed:
        return {
            "answer": result.validated_output["answer"],
            "confidence": result.validated_output["confidence"],
            "sources": result.validated_output["sources"],
            "validation_passed": True,
        }
    else:
        return {
            "answer": "Could not generate a verified response.",
            "validation_passed": False,
            "errors": [
                str(e) for e in result.validation_output
            ],
        }
```

### Custom Validators

```python
from guardrails.validators import (
    Validator,
    ValidationResult,
    PassResult,
    FailResult,
    register_validator,
)


@register_validator(
    name="rag/grounded-in-context", data_type="string"
)
class GroundedInContext(Validator):
    """Validate that every sentence in the answer is grounded
    in the provided context documents.

    Uses a lightweight NLI check or keyword overlap to verify
    groundedness without requiring an additional LLM call.
    """

    def __init__(
        self,
        min_overlap: float = 0.3,
        on_fail: str = "reask",
        **kwargs,
    ):
        super().__init__(on_fail=on_fail, **kwargs)
        self.min_overlap = min_overlap

    def validate(
        self, value: str, metadata: dict
    ) -> ValidationResult:
        context = metadata.get("sources", "")
        if not context:
            return FailResult(
                error_message="No context provided for grounding check",
            )

        sentences = [
            s.strip()
            for s in value.replace("!", ".").replace("?", ".").split(".")
            if len(s.strip()) > 15
        ]

        context_words = set(context.lower().split())
        ungrounded = []

        for sentence in sentences:
            sentence_words = set(sentence.lower().split())
            if not sentence_words:
                continue
            overlap = len(sentence_words & context_words) / len(
                sentence_words
            )
            if overlap < self.min_overlap:
                ungrounded.append(sentence)

        if ungrounded:
            return FailResult(
                error_message=(
                    f"The following sentences appear ungrounded: "
                    f"{'; '.join(ungrounded[:3])}"
                ),
                fix_value=value,  # Trigger reask with error context
            )

        return PassResult()


@register_validator(
    name="rag/valid-citations", data_type="list"
)
class ValidCitations(Validator):
    """Validate that cited document IDs exist in the retrieval results."""

    def __init__(self, on_fail: str = "reask", **kwargs):
        super().__init__(on_fail=on_fail, **kwargs)

    def validate(
        self, value: list, metadata: dict
    ) -> ValidationResult:
        available_ids = set(metadata.get("document_ids", []))
        if not available_ids:
            return PassResult()

        invalid = [v for v in value if v not in available_ids]
        if invalid:
            return FailResult(
                error_message=(
                    f"Citations reference non-existent documents: {invalid}. "
                    f"Available: {sorted(available_ids)}"
                ),
            )
        return PassResult()
```

### Combining Multiple Validators

```python
class StrictRAGResponse(BaseModel):
    """RAG response with multiple layered validators."""

    answer: str = Field(
        description="Answer grounded in provided documents",
        json_schema_extra={
            "validators": [
                GroundedInContext(min_overlap=0.3, on_fail="reask"),
                ToxicLanguage(on_fail="filter"),
            ],
        },
    )

    sources: list[str] = Field(
        description="Document IDs cited in the answer",
        json_schema_extra={
            "validators": [
                ValidCitations(on_fail="reask"),
            ],
        },
    )

    is_answerable: bool = Field(
        description="Whether the context contains enough info to answer",
    )
```

---

## Pydantic Structured Output

### Direct Structured Output with Anthropic

Anthropic and OpenAI support structured output natively, which can enforce citation and groundedness at the schema level:

```python
from anthropic import Anthropic
from pydantic import BaseModel, Field


class CitedClaim(BaseModel):
    """A single claim with its supporting citation."""

    claim: str = Field(description="A factual statement")
    source_doc_id: str = Field(
        description="ID of the document supporting this claim"
    )
    quote: str = Field(
        description="Direct quote from the source document"
    )
    confidence: float = Field(
        description="Confidence that the source supports this claim",
        ge=0.0,
        le=1.0,
    )


class StructuredRAGAnswer(BaseModel):
    """RAG answer with enforced citation structure."""

    claims: list[CitedClaim] = Field(
        description="List of claims, each with a citation",
        min_length=1,
    )
    summary: str = Field(
        description="Brief summary synthesizing the claims"
    )
    answerable: bool = Field(
        description="Whether the documents contain sufficient info"
    )
    unanswered_aspects: list[str] = Field(
        default_factory=list,
        description="Parts of the question not covered by documents",
    )


def structured_rag_query(
    query: str,
    documents: list[dict],
) -> StructuredRAGAnswer:
    """Generate a structured RAG response with enforced citations.

    By using Pydantic structured output, the LLM is constrained
    to produce a response that matches the schema -- every claim
    must have a source_doc_id and a direct quote.
    """
    client = Anthropic()

    context = "\n\n".join(
        f"[{doc['id']}]\n{doc['content']}" for doc in documents
    )
    doc_ids = [doc["id"] for doc in documents]

    response = client.messages.create(
        model="claude-sonnet-4-20250514",
        max_tokens=2048,
        system=(
            "You are a documentation assistant. Answer questions using "
            "ONLY the provided documents. Every claim must cite a specific "
            "document and include a direct quote from that document. "
            f"Available document IDs: {doc_ids}"
        ),
        messages=[
            {
                "role": "user",
                "content": (
                    f"Documents:\n{context}\n\n"
                    f"Question: {query}\n\n"
                    "Respond with a structured answer."
                ),
            },
        ],
        # Note: tool_use-based structured output
        tools=[
            {
                "name": "provide_answer",
                "description": "Provide a structured answer with citations",
                "input_schema": StructuredRAGAnswer.model_json_schema(),
            },
        ],
        tool_choice={"type": "tool", "name": "provide_answer"},
    )

    # Extract the tool call result
    tool_block = next(
        b for b in response.content if b.type == "tool_use"
    )
    return StructuredRAGAnswer.model_validate(tool_block.input)


def verify_structured_answer(
    answer: StructuredRAGAnswer,
    documents: dict[str, str],
) -> dict:
    """Post-generation verification of structured output.

    Even with structured output, verify that:
    1. Cited doc IDs actually exist
    2. Quoted text actually appears in the cited document
    """
    results = []
    for claim in answer.claims:
        doc_exists = claim.source_doc_id in documents
        quote_found = False
        if doc_exists:
            # Check if the quote appears in the document
            doc_text = documents[claim.source_doc_id].lower()
            quote_lower = claim.quote.lower().strip()
            # Allow fuzzy matching (quotes may be slightly paraphrased)
            quote_words = set(quote_lower.split())
            doc_words = set(doc_text.split())
            word_overlap = len(quote_words & doc_words) / max(
                len(quote_words), 1
            )
            quote_found = word_overlap > 0.7

        results.append({
            "claim": claim.claim,
            "doc_exists": doc_exists,
            "quote_verified": quote_found,
            "verified": doc_exists and quote_found,
        })

    verified_count = sum(1 for r in results if r["verified"])
    return {
        "total_claims": len(results),
        "verified_claims": verified_count,
        "verification_rate": (
            verified_count / len(results) if results else 0
        ),
        "details": results,
    }
```

---

## Composing Frameworks for Defense in Depth

### Layered Guardrail Architecture

```python
class DefenseInDepthRAG:
    """Compose multiple guardrail frameworks for maximum safety.

    Layer 1: NeMo Guardrails for input filtering (topic, jailbreak)
    Layer 2: Pydantic structured output for citation enforcement
    Layer 3: Guardrails AI validators for output verification
    """

    def __init__(
        self,
        retriever,
        nemo_config_path: str,
        groundedness_threshold: float = 0.7,
    ):
        self.retriever = retriever
        self.nemo_rails = LLMRails(
            RailsConfig.from_path(nemo_config_path)
        )
        self.client = Anthropic()
        self.threshold = groundedness_threshold

    async def query(self, user_message: str) -> dict:
        # Layer 1: NeMo input rails
        input_check = await self.nemo_rails.generate_async(
            messages=[{"role": "user", "content": user_message}]
        )
        if self._was_blocked(input_check):
            return {
                "answer": input_check["content"],
                "blocked_by": "input_rails",
            }

        # Retrieve documents
        docs = self.retriever.invoke(user_message)
        doc_map = {
            d.metadata.get("source", f"doc_{i}"): d.page_content
            for i, d in enumerate(docs)
        }

        # Layer 2: Structured output with citations
        structured = structured_rag_query(
            query=user_message,
            documents=[
                {"id": k, "content": v} for k, v in doc_map.items()
            ],
        )

        # Layer 3: Verify citations
        verification = verify_structured_answer(structured, doc_map)

        if verification["verification_rate"] < self.threshold:
            return {
                "answer": (
                    "I found relevant documents but could not produce "
                    "a fully verified answer. The most relevant sources "
                    "are listed below for your reference."
                ),
                "sources": list(doc_map.keys()),
                "verification_rate": verification["verification_rate"],
                "blocked_by": "output_verification",
            }

        return {
            "answer": structured.summary,
            "claims": [c.model_dump() for c in structured.claims],
            "verification": verification,
            "blocked_by": None,
        }

    def _was_blocked(self, response: dict) -> bool:
        """Check if NeMo rails blocked the input."""
        blocked_phrases = [
            "I can only help with",
            "I cannot assist with",
            "outside my scope",
        ]
        content = response.get("content", "")
        return any(p in content for p in blocked_phrases)
```

---

## Performance Comparison

### Framework Overhead

| Framework | Latency Overhead | Extra LLM Calls | Best For |
|-----------|-----------------|-----------------|----------|
| NeMo Guardrails | 100-500 ms | 0-2 (for fact-check) | Input filtering, topic control |
| Guardrails AI | 50-200 ms | 0-2 (for reask) | Schema validation, output enforcement |
| Pydantic structured | ~0 ms | 0 (part of generation) | Citation enforcement |
| All three combined | 200-800 ms | 0-4 | Maximum safety |

---

## Common Pitfalls

1. **Overloading NeMo Guardrails with too many flows.** Each input/output rail adds latency. Start with the 2-3 most critical rails and add more based on observed failures.
2. **Using ProvenanceLLM validator without a source context.** The ProvenanceLLM validator in Guardrails AI requires the `sources` key in metadata. Without it, the validator silently passes everything.
3. **Not handling Guardrails AI reask loops.** If the LLM consistently fails validation, the reask loop can burn through API credits. Set `num_reasks` to a small number (1-2) and handle final failures gracefully.
4. **Trusting structured output blindly.** Just because the LLM produced a JSON with `source_doc_id` and `quote` fields does not mean those values are correct. Always verify citations post-generation.
5. **Running all guardrails synchronously.** Input rails and output rails can run independently of each other in some cases. Use async where possible to minimize total latency.
6. **Not testing guardrails with adversarial inputs.** Guardrails that work on normal queries may fail on crafted inputs. Test with prompt injections, topic boundary cases, and edge cases.

---

## References

- NeMo Guardrails documentation: https://docs.nvidia.com/nemo/guardrails/
- NeMo Guardrails GitHub: https://github.com/NVIDIA/NeMo-Guardrails
- Guardrails AI documentation: https://docs.guardrailsai.com/
- Guardrails AI Hub: https://hub.guardrailsai.com/
- Pydantic documentation: https://docs.pydantic.dev/
