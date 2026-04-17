# Ingestion Orchestration -- Prefect Pipeline: Full Extract-Parse-Chunk-Embed-Index DAG

## TL;DR

This article provides a complete, production-ready RAG ingestion pipeline built with Prefect. The pipeline implements the full extract-parse-chunk-embed-index flow as a Prefect flow with individual tasks for each stage, proper retry logic, result caching, dead letter queuing, and incremental processing. Each task is independently retryable and observable through the Prefect dashboard.

---

## Pipeline Architecture

```
@flow: rag_ingestion_pipeline
    |
    +-- @task: detect_changes          [retry=2, cache=5min]
    |       |
    |       v
    +-- @task: extract_documents       [retry=3, concurrent=4]
    |       |
    |       v
    +-- @task: parse_document          [retry=2, concurrent=8]
    |       |
    |       v
    +-- @task: chunk_document          [retry=1, concurrent=8]
    |       |
    |       v
    +-- @task: embed_chunks            [retry=5, concurrent=2, rate_limited]
    |       |
    |       v
    +-- @task: index_vectors           [retry=3, concurrent=2]
    |       |
    |       v
    +-- @task: validate_ingestion      [retry=1]
    |       |
    |       v
    +-- @task: report_results          [retry=1]
```

## Complete Implementation

```python
import hashlib
import json
import time
from pathlib import Path
from dataclasses import dataclass, field
from prefect import flow, task, get_run_logger
from prefect.tasks import task_input_hash
from prefect.concurrency.sync import concurrency
from datetime import timedelta


# --- Data Models ---

@dataclass
class DocumentSource:
    document_id: str
    source_path: str
    content_hash: str = ""
    metadata: dict = field(default_factory=dict)


@dataclass
class ParsedDocument:
    document_id: str
    text: str
    tables: list[str] = field(default_factory=list)
    images: list[str] = field(default_factory=list)
    metadata: dict = field(default_factory=dict)


@dataclass
class Chunk:
    chunk_id: str
    document_id: str
    text: str
    metadata: dict = field(default_factory=dict)


@dataclass
class EmbeddedChunk:
    chunk_id: str
    document_id: str
    text: str
    embedding: list[float]
    metadata: dict = field(default_factory=dict)


@dataclass
class IngestionResult:
    total_documents: int
    documents_processed: int
    documents_skipped: int
    documents_failed: int
    chunks_created: int
    vectors_indexed: int
    duration_seconds: float
    errors: list[dict] = field(default_factory=list)


# --- Tasks ---

@task(
    retries=2,
    retry_delay_seconds=10,
    cache_key_fn=task_input_hash,
    cache_expiration=timedelta(minutes=5),
    name="detect-changes",
)
def detect_changes(
    source_dir: str,
    state_file: str,
) -> list[DocumentSource]:
    """Scan source directory for new or modified documents."""
    logger = get_run_logger()
    source_path = Path(source_dir)
    state_path = Path(state_file)

    # Load previous state
    previous_state: dict[str, str] = {}
    if state_path.exists():
        previous_state = json.loads(state_path.read_text())

    # Scan current documents
    documents = []
    supported_extensions = {".pdf", ".txt", ".md", ".html", ".docx"}

    for file_path in source_path.rglob("*"):
        if file_path.suffix.lower() not in supported_extensions:
            continue

        content_hash = hashlib.sha256(file_path.read_bytes()).hexdigest()
        doc_id = str(file_path.relative_to(source_path))

        # Skip unchanged documents
        if previous_state.get(doc_id) == content_hash:
            continue

        documents.append(DocumentSource(
            document_id=doc_id,
            source_path=str(file_path),
            content_hash=content_hash,
            metadata={"extension": file_path.suffix, "size": file_path.stat().st_size},
        ))

    logger.info(f"Found {len(documents)} new/modified documents")
    return documents


@task(
    retries=3,
    retry_delay_seconds=30,
    name="extract-document",
)
def extract_document(source: DocumentSource) -> str:
    """Extract raw text content from a document."""
    logger = get_run_logger()
    path = Path(source.source_path)
    ext = path.suffix.lower()

    try:
        if ext == ".txt" or ext == ".md":
            return path.read_text(encoding="utf-8")

        elif ext == ".pdf":
            import fitz
            doc = fitz.open(str(path))
            text_parts = []
            for page in doc:
                text_parts.append(page.get_text())
            doc.close()
            return "\n\n".join(text_parts)

        elif ext == ".html":
            from bs4 import BeautifulSoup
            html = path.read_text(encoding="utf-8")
            soup = BeautifulSoup(html, "html.parser")
            return soup.get_text(separator="\n\n")

        elif ext == ".docx":
            import docx
            document = docx.Document(str(path))
            return "\n\n".join(p.text for p in document.paragraphs if p.text.strip())

        else:
            raise ValueError(f"Unsupported format: {ext}")

    except Exception as e:
        logger.error(f"Failed to extract {source.document_id}: {e}")
        raise


@task(
    retries=2,
    retry_delay_seconds=5,
    name="parse-document",
)
def parse_document(
    document_id: str, raw_text: str, metadata: dict
) -> ParsedDocument:
    """Clean and structure extracted text."""
    import re

    # Clean text
    text = raw_text.strip()
    text = re.sub(r"\n{3,}", "\n\n", text)  # collapse excessive newlines
    text = re.sub(r"[ \t]+", " ", text)       # collapse whitespace
    text = re.sub(r"\x00", "", text)           # remove null bytes

    # Extract metadata from content
    if text[:100].count("\n") < 3:
        first_line = text.split("\n")[0].strip()
        if len(first_line) < 200:
            metadata["title"] = first_line

    metadata["char_count"] = len(text)
    metadata["word_count"] = len(text.split())

    return ParsedDocument(
        document_id=document_id,
        text=text,
        metadata=metadata,
    )


@task(
    retries=1,
    name="chunk-document",
)
def chunk_document(
    parsed: ParsedDocument,
    chunk_size: int = 1000,
    overlap: int = 200,
) -> list[Chunk]:
    """Split document into overlapping chunks."""
    text = parsed.text
    words = text.split()
    chunks = []

    if len(words) <= chunk_size:
        chunks.append(Chunk(
            chunk_id=f"{parsed.document_id}:0",
            document_id=parsed.document_id,
            text=text,
            metadata={**parsed.metadata, "chunk_index": 0, "total_chunks": 1},
        ))
        return chunks

    start = 0
    chunk_idx = 0

    while start < len(words):
        end = start + chunk_size
        chunk_words = words[start:end]
        chunk_text = " ".join(chunk_words)

        chunks.append(Chunk(
            chunk_id=f"{parsed.document_id}:{chunk_idx}",
            document_id=parsed.document_id,
            text=chunk_text,
            metadata={
                **parsed.metadata,
                "chunk_index": chunk_idx,
                "start_word": start,
                "end_word": min(end, len(words)),
            },
        ))

        start += chunk_size - overlap
        chunk_idx += 1

    # Set total_chunks in metadata
    for chunk in chunks:
        chunk.metadata["total_chunks"] = len(chunks)

    return chunks


@task(
    retries=5,
    retry_delay_seconds=30,
    name="embed-chunks",
    tags=["api-call", "rate-limited"],
)
def embed_chunks(
    chunks: list[Chunk],
    model: str = "text-embedding-3-small",
    batch_size: int = 100,
) -> list[EmbeddedChunk]:
    """Generate embeddings for chunks using OpenAI API."""
    logger = get_run_logger()
    import openai

    client = openai.OpenAI()
    embedded = []

    for i in range(0, len(chunks), batch_size):
        batch = chunks[i : i + batch_size]
        texts = [c.text for c in batch]

        # Use Prefect concurrency limits for rate limiting
        with concurrency("embedding-api", occupy=1):
            response = client.embeddings.create(
                model=model,
                input=texts,
            )

        for chunk, embedding_data in zip(batch, response.data):
            embedded.append(EmbeddedChunk(
                chunk_id=chunk.chunk_id,
                document_id=chunk.document_id,
                text=chunk.text,
                embedding=embedding_data.embedding,
                metadata=chunk.metadata,
            ))

        logger.info(f"Embedded batch {i // batch_size + 1}, total: {len(embedded)}")

    return embedded


@task(
    retries=3,
    retry_delay_seconds=10,
    name="index-vectors",
)
def index_vectors(
    embedded_chunks: list[EmbeddedChunk],
    collection_name: str = "rag_documents",
    batch_size: int = 100,
) -> int:
    """Upsert embeddings into the vector database."""
    logger = get_run_logger()

    # Example using Qdrant (replace with your vector DB client)
    from qdrant_client import QdrantClient
    from qdrant_client.models import PointStruct

    client = QdrantClient(url="http://localhost:6333")
    total_indexed = 0

    for i in range(0, len(embedded_chunks), batch_size):
        batch = embedded_chunks[i : i + batch_size]
        points = [
            PointStruct(
                id=hashlib.md5(ec.chunk_id.encode()).hexdigest(),
                vector=ec.embedding,
                payload={
                    "chunk_id": ec.chunk_id,
                    "document_id": ec.document_id,
                    "text": ec.text,
                    **ec.metadata,
                },
            )
            for ec in batch
        ]
        client.upsert(collection_name=collection_name, points=points)
        total_indexed += len(points)

    logger.info(f"Indexed {total_indexed} vectors into {collection_name}")
    return total_indexed


@task(
    retries=1,
    name="validate-ingestion",
)
def validate_ingestion(
    document_ids: list[str],
    collection_name: str = "rag_documents",
    sample_size: int = 5,
) -> dict:
    """Validate that ingested documents are searchable."""
    logger = get_run_logger()
    from qdrant_client import QdrantClient

    client = QdrantClient(url="http://localhost:6333")
    issues = []

    # Check collection exists and has points
    collection_info = client.get_collection(collection_name)
    if collection_info.points_count == 0:
        issues.append("Collection is empty after ingestion")

    # Sample search to verify vectors work
    import random
    sample_docs = random.sample(document_ids, min(sample_size, len(document_ids)))

    for doc_id in sample_docs:
        results = client.scroll(
            collection_name=collection_name,
            scroll_filter={"must": [{"key": "document_id", "match": {"value": doc_id}}]},
            limit=1,
        )
        if not results[0]:
            issues.append(f"Document {doc_id} not found in index")

    validation = {
        "valid": len(issues) == 0,
        "issues": issues,
        "collection_points": collection_info.points_count,
        "documents_checked": len(sample_docs),
    }

    if issues:
        logger.warning(f"Validation issues: {issues}")
    else:
        logger.info("Validation passed")

    return validation


@task(name="save-state")
def save_state(
    documents: list[DocumentSource],
    state_file: str,
) -> None:
    """Save processing state for incremental runs."""
    state_path = Path(state_file)

    # Load existing state
    existing = {}
    if state_path.exists():
        existing = json.loads(state_path.read_text())

    # Update with processed documents
    for doc in documents:
        existing[doc.document_id] = doc.content_hash

    state_path.write_text(json.dumps(existing, indent=2))


# --- Main Flow ---

@flow(
    name="rag-ingestion-pipeline",
    description="Full RAG ingestion: detect -> extract -> parse -> chunk -> embed -> index",
    retries=0,  # Flow-level retry not needed; tasks handle retries
    log_prints=True,
)
def rag_ingestion_pipeline(
    source_dir: str,
    state_file: str = "./ingestion_state.json",
    collection_name: str = "rag_documents",
    chunk_size: int = 1000,
    chunk_overlap: int = 200,
    embedding_model: str = "text-embedding-3-small",
) -> IngestionResult:
    """Full RAG ingestion pipeline."""
    logger = get_run_logger()
    start_time = time.time()

    # Stage 1: Detect changes
    changed_docs = detect_changes(source_dir, state_file)

    if not changed_docs:
        logger.info("No changes detected. Pipeline complete.")
        return IngestionResult(
            total_documents=0, documents_processed=0,
            documents_skipped=0, documents_failed=0,
            chunks_created=0, vectors_indexed=0,
            duration_seconds=time.time() - start_time,
        )

    logger.info(f"Processing {len(changed_docs)} documents")

    # Stages 2-4: Extract, parse, chunk (per document)
    all_chunks: list[Chunk] = []
    processed_docs: list[DocumentSource] = []
    failed_docs: list[dict] = []

    for doc_source in changed_docs:
        try:
            # Extract
            raw_text = extract_document(doc_source)

            # Parse
            parsed = parse_document(
                doc_source.document_id, raw_text, doc_source.metadata
            )

            # Chunk
            chunks = chunk_document(parsed, chunk_size, chunk_overlap)
            all_chunks.extend(chunks)
            processed_docs.append(doc_source)

        except Exception as e:
            logger.error(f"Failed to process {doc_source.document_id}: {e}")
            failed_docs.append({
                "document_id": doc_source.document_id,
                "error": str(e),
            })

    if not all_chunks:
        logger.warning("No chunks produced. Check document processing.")
        return IngestionResult(
            total_documents=len(changed_docs),
            documents_processed=0,
            documents_skipped=0,
            documents_failed=len(failed_docs),
            chunks_created=0,
            vectors_indexed=0,
            duration_seconds=time.time() - start_time,
            errors=failed_docs,
        )

    # Stage 5: Embed
    logger.info(f"Embedding {len(all_chunks)} chunks")
    embedded = embed_chunks(all_chunks, model=embedding_model)

    # Stage 6: Index
    vectors_indexed = index_vectors(embedded, collection_name=collection_name)

    # Stage 7: Validate
    doc_ids = [d.document_id for d in processed_docs]
    validation = validate_ingestion(doc_ids, collection_name=collection_name)

    # Save state for incremental runs
    save_state(processed_docs, state_file)

    duration = time.time() - start_time
    logger.info(
        f"Pipeline complete in {duration:.1f}s. "
        f"Processed: {len(processed_docs)}, "
        f"Chunks: {len(all_chunks)}, "
        f"Vectors: {vectors_indexed}, "
        f"Failed: {len(failed_docs)}"
    )

    return IngestionResult(
        total_documents=len(changed_docs),
        documents_processed=len(processed_docs),
        documents_skipped=0,
        documents_failed=len(failed_docs),
        chunks_created=len(all_chunks),
        vectors_indexed=vectors_indexed,
        duration_seconds=duration,
        errors=failed_docs,
    )


# --- Deployment ---

if __name__ == "__main__":
    # Local run
    result = rag_ingestion_pipeline(
        source_dir="./documents",
        state_file="./ingestion_state.json",
    )
    print(f"Result: {result}")

# Prefect deployment (schedule):
# prefect deployment build ./pipeline.py:rag_ingestion_pipeline \
#   --name "daily-ingestion" \
#   --cron "0 2 * * *" \
#   --pool default-agent-pool
```

---

## Common Pitfalls

1. **Not using Prefect concurrency limits for API calls.** Without `concurrency("embedding-api", occupy=1)`, parallel tasks overwhelm the embedding API with rate-limit errors.

2. **Storing embeddings in task results.** Prefect persists task results for caching. Large embedding arrays bloat the result store. Disable caching for the embed task or use a separate storage backend.

3. **Not saving state after each document.** If the pipeline crashes after processing 500 of 1000 documents, the state file should reflect the 500 that succeeded. Save state incrementally, not just at the end.

4. **Running the entire pipeline as a single task.** Each stage should be a separate task so failures are isolated and retried independently.

5. **Ignoring the validation stage.** Index a batch of documents, then immediately run a sample search. Catching issues during ingestion is much cheaper than discovering them when users report bad answers.

---

## References

- Prefect documentation: https://docs.prefect.io/
- Prefect tasks and flows: https://docs.prefect.io/concepts/tasks/
- Prefect concurrency: https://docs.prefect.io/concepts/concurrency-limits/
- Prefect deployments: https://docs.prefect.io/concepts/deployments/
