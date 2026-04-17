# Batch Inference -- Implementation: Batch Embedding and Generation Pipelines

## Overview

This guide covers implementing production batch inference pipelines for RAG systems. It includes batch embedding for document indexing, batch generation for content processing, evaluation pipelines, error handling, and integration with workflow orchestration.

---

## Batch Embedding Pipeline

### Complete Document Indexing Pipeline

```python
import openai
import json
import time
from pathlib import Path
from dataclasses import dataclass


@dataclass
class BatchJob:
    batch_id: str
    input_file: str
    status: str
    total_requests: int


class BatchEmbeddingPipeline:
    """Production batch embedding pipeline using OpenAI Batch API."""

    def __init__(
        self,
        model: str = "text-embedding-3-small",
        max_batch_size: int = 50000,
        max_file_size_mb: int = 190,
    ):
        self.client = openai.OpenAI()
        self.model = model
        self.max_batch_size = max_batch_size
        self.max_file_size_mb = max_file_size_mb

    def embed_documents(
        self,
        documents: list[dict],  # [{"id": "...", "text": "..."}]
        output_dir: str = "./batch_results",
    ) -> dict[str, list[float]]:
        """Embed all documents using batch API. Returns {doc_id: embedding}."""
        Path(output_dir).mkdir(parents=True, exist_ok=True)

        # Split into batches
        batches = self._split_into_batches(documents)
        print(f"Split {len(documents)} documents into {len(batches)} batches")

        # Submit all batches
        jobs = []
        for i, batch in enumerate(batches):
            input_file = f"{output_dir}/batch_input_{i}.jsonl"
            self._write_batch_file(batch, input_file)
            batch_id = self._submit_batch(input_file)
            jobs.append(BatchJob(
                batch_id=batch_id,
                input_file=input_file,
                status="submitted",
                total_requests=len(batch),
            ))

        # Wait for all batches to complete
        completed = self._wait_for_all(jobs)

        # Collect results
        all_embeddings = {}
        for job in completed:
            results = self._download_results(job.batch_id)
            for result in results:
                doc_id = result["custom_id"]
                if "embedding" in result:
                    all_embeddings[doc_id] = result["embedding"]
                else:
                    print(f"Failed embedding for {doc_id}: {result.get('error')}")

        print(f"Successfully embedded {len(all_embeddings)}/{len(documents)} documents")
        return all_embeddings

    def _split_into_batches(self, documents: list[dict]) -> list[list[dict]]:
        """Split documents into batches respecting size limits."""
        batches = []
        current_batch = []
        current_size = 0

        for doc in documents:
            # Estimate JSONL line size
            line = json.dumps({
                "custom_id": doc["id"],
                "method": "POST",
                "url": "/v1/embeddings",
                "body": {"model": self.model, "input": doc["text"]},
            })
            line_size = len(line.encode("utf-8"))

            if (
                len(current_batch) >= self.max_batch_size
                or (current_size + line_size) > self.max_file_size_mb * 1024 * 1024
            ):
                batches.append(current_batch)
                current_batch = []
                current_size = 0

            current_batch.append(doc)
            current_size += line_size

        if current_batch:
            batches.append(current_batch)

        return batches

    def _write_batch_file(self, documents: list[dict], output_file: str):
        """Write documents to a JSONL batch file."""
        with open(output_file, "w") as f:
            for doc in documents:
                request = {
                    "custom_id": doc["id"],
                    "method": "POST",
                    "url": "/v1/embeddings",
                    "body": {
                        "model": self.model,
                        "input": doc["text"],
                    },
                }
                f.write(json.dumps(request) + "\n")

    def _submit_batch(self, input_file: str) -> str:
        """Upload file and create batch."""
        with open(input_file, "rb") as f:
            uploaded = self.client.files.create(file=f, purpose="batch")

        batch = self.client.batches.create(
            input_file_id=uploaded.id,
            endpoint="/v1/embeddings",
            completion_window="24h",
        )
        return batch.id

    def _wait_for_all(
        self,
        jobs: list[BatchJob],
        poll_interval: int = 60,
        timeout_hours: int = 24,
    ) -> list[BatchJob]:
        """Wait for all batch jobs to complete."""
        timeout_seconds = timeout_hours * 3600
        start = time.time()

        while time.time() - start < timeout_seconds:
            all_done = True
            for job in jobs:
                if job.status in ("completed", "failed", "expired"):
                    continue

                batch = self.client.batches.retrieve(job.batch_id)
                job.status = batch.status

                if batch.status not in ("completed", "failed", "expired"):
                    all_done = False

            if all_done:
                return jobs

            pending = sum(1 for j in jobs if j.status not in ("completed", "failed", "expired"))
            print(f"Waiting... {pending} batches still processing")
            time.sleep(poll_interval)

        raise TimeoutError(f"Batches not completed within {timeout_hours} hours")

    def _download_results(self, batch_id: str) -> list[dict]:
        """Download and parse batch results."""
        batch = self.client.batches.retrieve(batch_id)

        if batch.status != "completed":
            return []

        content = self.client.files.content(batch.output_file_id)
        results = []
        for line in content.text.strip().split("\n"):
            data = json.loads(line)
            if data["response"]["status_code"] == 200:
                results.append({
                    "custom_id": data["custom_id"],
                    "embedding": data["response"]["body"]["data"][0]["embedding"],
                })
            else:
                results.append({
                    "custom_id": data["custom_id"],
                    "error": data["response"]["body"],
                })

        return results


# Usage
pipeline = BatchEmbeddingPipeline(model="text-embedding-3-small")

documents = [
    {"id": f"doc-{i}", "text": f"Document content about topic {i}..."}
    for i in range(10000)
]

embeddings = pipeline.embed_documents(documents)
print(f"Got {len(embeddings)} embeddings")
```

---

## Batch Generation Pipeline

### RAG Evaluation Pipeline

```python
import anthropic
import json
import time


class BatchEvaluationPipeline:
    """Evaluate RAG responses using Anthropic Message Batches."""

    def __init__(self, model: str = "claude-sonnet-4-20250514"):
        self.client = anthropic.Anthropic()
        self.model = model

    def evaluate_responses(
        self,
        evaluations: list[dict],  # [{"question": ..., "context": ..., "answer": ..., "reference": ...}]
    ) -> list[dict]:
        """Evaluate a batch of RAG responses."""
        # Build batch requests
        batch_requests = []
        for i, eval_item in enumerate(evaluations):
            batch_requests.append({
                "custom_id": f"eval-{i}",
                "params": {
                    "model": self.model,
                    "max_tokens": 200,
                    "temperature": 0.0,
                    "messages": [
                        {
                            "role": "user",
                            "content": self._build_eval_prompt(eval_item),
                        },
                    ],
                },
            })

        # Submit batch
        batch = self.client.messages.batches.create(requests=batch_requests)
        print(f"Evaluation batch submitted: {batch.id}")

        # Wait for completion
        while True:
            batch = self.client.messages.batches.retrieve(batch.id)
            if batch.processing_status == "ended":
                break
            print(f"Processing... {batch.request_counts.succeeded}/{batch.request_counts.total}")
            time.sleep(30)

        # Collect results
        results = []
        for result in self.client.messages.batches.results(batch.id):
            idx = int(result.custom_id.split("-")[1])
            original = evaluations[idx]

            if result.result.type == "succeeded":
                eval_text = result.result.message.content[0].text
                score = self._parse_score(eval_text)
                results.append({
                    **original,
                    "score": score,
                    "evaluation": eval_text,
                    "status": "success",
                })
            else:
                results.append({
                    **original,
                    "score": None,
                    "evaluation": None,
                    "status": "error",
                    "error": str(result.result),
                })

        return results

    def _build_eval_prompt(self, eval_item: dict) -> str:
        return f"""Evaluate this RAG system response. Score from 1-5 on each dimension.

Question: {eval_item['question']}

Retrieved Context:
{eval_item['context']}

Generated Answer:
{eval_item['answer']}

Reference Answer:
{eval_item.get('reference', 'N/A')}

Rate each dimension (1-5):
1. Faithfulness: Is the answer grounded in the context?
2. Relevance: Does the answer address the question?
3. Completeness: Does the answer cover all relevant information?
4. Conciseness: Is the answer appropriately concise?

Return scores as JSON: {{"faithfulness": N, "relevance": N, "completeness": N, "conciseness": N, "overall": N}}"""

    def _parse_score(self, eval_text: str) -> dict:
        """Extract scores from evaluation text."""
        try:
            # Find JSON in the response
            start = eval_text.index("{")
            end = eval_text.rindex("}") + 1
            return json.loads(eval_text[start:end])
        except (ValueError, json.JSONDecodeError):
            return {"overall": 3, "parse_error": True}


# Usage
pipeline = BatchEvaluationPipeline()

evaluations = [
    {
        "question": "What is vector search?",
        "context": "Vector search finds similar items by comparing embedding vectors...",
        "answer": "Vector search is a technique for finding similar items using embeddings.",
        "reference": "Vector search uses mathematical similarity between embedding vectors.",
    },
    # ... hundreds more
]

results = pipeline.evaluate_responses(evaluations)

# Aggregate scores
avg_scores = {}
for r in results:
    if r["status"] == "success" and isinstance(r["score"], dict):
        for key, value in r["score"].items():
            if isinstance(value, (int, float)):
                avg_scores.setdefault(key, []).append(value)

print("Average evaluation scores:")
for metric, scores in avg_scores.items():
    print(f"  {metric}: {sum(scores)/len(scores):.2f}")
```

---

## Batch Processing with Error Handling and Retries

```python
import json
import time
import openai
from typing import Optional


class RobustBatchProcessor:
    """Batch processor with retry logic and error handling."""

    def __init__(self, max_retries: int = 3, retry_delay: int = 300):
        self.client = openai.OpenAI()
        self.max_retries = max_retries
        self.retry_delay = retry_delay

    def process_with_retries(
        self,
        requests: list[dict],
        endpoint: str = "/v1/embeddings",
    ) -> list[dict]:
        """Process requests with automatic retry for failures."""
        all_results = {}
        pending = requests

        for attempt in range(self.max_retries + 1):
            if not pending:
                break

            if attempt > 0:
                print(f"Retry attempt {attempt}: {len(pending)} requests")
                time.sleep(self.retry_delay)

            # Submit batch
            batch_id = self._submit(pending, endpoint)
            batch = self._wait(batch_id)
            results = self._download(batch)

            # Separate successes and failures
            new_pending = []
            for result in results:
                if result.get("error"):
                    # Find original request for retry
                    original = next(
                        (r for r in pending if r["custom_id"] == result["custom_id"]),
                        None,
                    )
                    if original:
                        new_pending.append(original)
                else:
                    all_results[result["custom_id"]] = result

            pending = new_pending

        if pending:
            print(f"WARNING: {len(pending)} requests failed after {self.max_retries} retries")

        return list(all_results.values())

    def _submit(self, requests: list[dict], endpoint: str) -> str:
        """Create and submit a batch."""
        # Write to temp file
        import tempfile
        with tempfile.NamedTemporaryFile(mode="w", suffix=".jsonl", delete=False) as f:
            for req in requests:
                f.write(json.dumps(req) + "\n")
            temp_path = f.name

        with open(temp_path, "rb") as f:
            uploaded = self.client.files.create(file=f, purpose="batch")

        batch = self.client.batches.create(
            input_file_id=uploaded.id,
            endpoint=endpoint,
            completion_window="24h",
        )
        return batch.id

    def _wait(self, batch_id: str) -> dict:
        """Wait for batch completion."""
        while True:
            batch = self.client.batches.retrieve(batch_id)
            if batch.status in ("completed", "failed", "expired"):
                return batch
            time.sleep(60)

    def _download(self, batch) -> list[dict]:
        """Download results, handling both successes and failures."""
        results = []

        if batch.output_file_id:
            content = self.client.files.content(batch.output_file_id)
            for line in content.text.strip().split("\n"):
                data = json.loads(line)
                if data["response"]["status_code"] == 200:
                    results.append({
                        "custom_id": data["custom_id"],
                        "data": data["response"]["body"],
                    })
                else:
                    results.append({
                        "custom_id": data["custom_id"],
                        "error": data["response"]["body"],
                    })

        if batch.error_file_id:
            error_content = self.client.files.content(batch.error_file_id)
            for line in error_content.text.strip().split("\n"):
                data = json.loads(line)
                results.append({
                    "custom_id": data["custom_id"],
                    "error": data.get("error", "unknown error"),
                })

        return results
```

---

## Integration with Vector Databases

```python
from qdrant_client import QdrantClient
from qdrant_client.models import PointStruct, VectorParams, Distance


def batch_embed_and_index(
    documents: list[dict],
    collection_name: str = "documents",
    qdrant_url: str = "http://localhost:6333",
    embedding_model: str = "text-embedding-3-small",
    embedding_dim: int = 1536,
):
    """Batch embed documents and index in Qdrant."""
    # Step 1: Batch embed
    pipeline = BatchEmbeddingPipeline(model=embedding_model)
    embeddings = pipeline.embed_documents(documents)

    # Step 2: Create Qdrant collection
    qdrant = QdrantClient(url=qdrant_url)

    if not qdrant.collection_exists(collection_name):
        qdrant.create_collection(
            collection_name=collection_name,
            vectors_config=VectorParams(
                size=embedding_dim,
                distance=Distance.COSINE,
            ),
        )

    # Step 3: Upload to Qdrant in batches
    points = []
    for doc in documents:
        if doc["id"] in embeddings:
            points.append(PointStruct(
                id=hash(doc["id"]) % (2**63),
                vector=embeddings[doc["id"]],
                payload={
                    "doc_id": doc["id"],
                    "text": doc["text"][:1000],
                    "metadata": doc.get("metadata", {}),
                },
            ))

    batch_size = 100
    for i in range(0, len(points), batch_size):
        batch = points[i:i + batch_size]
        qdrant.upsert(collection_name=collection_name, points=batch)

    print(f"Indexed {len(points)} documents in Qdrant")
```

---

## Common Pitfalls

1. **Not tracking batch job IDs**: If your process crashes between submitting and downloading results, you lose the batch ID. Always persist batch IDs to a file or database immediately after submission.

2. **Ignoring partial failures**: Some requests in a batch may fail (rate limits, invalid input, server errors). Always check each result's status code and retry failures.

3. **Large file upload timeouts**: JSONL files over 100MB may timeout during upload. Split into smaller batches or compress.

4. **Not matching custom_ids back to documents**: Results come back unordered. Use the `custom_id` field (set to your document ID) to match embeddings back to documents.

5. **Blocking on batch completion**: Batch processing can take hours. Implement async polling or webhook callbacks rather than blocking your main process.

6. **Not validating input before batching**: Invalid inputs (empty strings, strings exceeding token limits) cause per-request failures. Validate and clean inputs before creating the batch file.

---

## References

- OpenAI Batch API: https://platform.openai.com/docs/guides/batch
- Anthropic Message Batches: https://docs.anthropic.com/en/docs/build-with-claude/message-batches
- Voyage AI embedding: https://docs.voyageai.com/
