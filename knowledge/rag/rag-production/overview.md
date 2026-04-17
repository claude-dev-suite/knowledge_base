# RAG Production -- Production-Readiness Patterns

## TL;DR

Moving a RAG pipeline from prototype to production requires solving problems that do not exist in development: index staleness (documents change but the index does not), scaling to thousands of concurrent users across multiple tenants, handling partial failures gracefully, and maintaining quality SLAs while keeping costs predictable. This overview covers the production-readiness checklist, the three pillars of production RAG (reliable indexing, horizontal scaling, and operational excellence), and the architectural patterns that separate toy RAG from production RAG. Subsequent articles cover incremental and blue-green indexing strategies, and horizontal scaling with multi-tenant isolation.

---

## The Production Gap

### What Works in Development but Fails in Production

| Development | Production | What Breaks |
|-------------|-----------|-------------|
| Reindex everything on every change | Documents change continuously | Reindexing takes hours, stale data served |
| Single-process, in-memory vector store | Thousands of concurrent queries | Memory exhaustion, no fault tolerance |
| One user, one namespace | Multiple tenants sharing infrastructure | Data leakage between tenants |
| Ignore errors | Partial failures (embedding API down) | Silent data loss, corrupt index |
| No monitoring | Quality degradation over weeks | Users complain before you notice |
| Fixed configuration | Different SLAs per customer tier | Premium customers get the same slow response |

---

## Production-Readiness Checklist

```python
from dataclasses import dataclass
from enum import Enum


class ReadinessLevel(Enum):
    NOT_READY = "not_ready"
    PARTIALLY_READY = "partially_ready"
    PRODUCTION_READY = "production_ready"


@dataclass
class ReadinessCheck:
    name: str
    category: str
    passed: bool
    detail: str


class ProductionReadinessChecker:
    """Evaluate whether a RAG system is production-ready."""

    def check_all(self, config: dict) -> dict:
        """Run all production-readiness checks."""
        checks = [
            # Indexing
            ReadinessCheck(
                "incremental_indexing",
                "indexing",
                config.get("incremental_indexing_enabled", False),
                "Incremental indexing avoids full reindex on every change",
            ),
            ReadinessCheck(
                "index_versioning",
                "indexing",
                config.get("index_alias_enabled", False),
                "Index aliases enable zero-downtime reindexing",
            ),
            ReadinessCheck(
                "document_deduplication",
                "indexing",
                config.get("dedup_enabled", False),
                "Hash-based dedup prevents duplicate chunks",
            ),

            # Scaling
            ReadinessCheck(
                "horizontal_scaling",
                "scaling",
                config.get("replica_count", 1) > 1,
                "Multiple replicas for fault tolerance",
            ),
            ReadinessCheck(
                "tenant_isolation",
                "scaling",
                config.get("multi_tenant", False),
                "Namespace or partition isolation per tenant",
            ),
            ReadinessCheck(
                "rate_limiting",
                "scaling",
                config.get("rate_limiting_enabled", False),
                "Per-tenant rate limits to prevent noisy neighbor",
            ),

            # Reliability
            ReadinessCheck(
                "retry_policy",
                "reliability",
                config.get("retry_enabled", False),
                "Retries with exponential backoff on transient failures",
            ),
            ReadinessCheck(
                "circuit_breaker",
                "reliability",
                config.get("circuit_breaker_enabled", False),
                "Circuit breaker prevents cascade failures",
            ),
            ReadinessCheck(
                "fallback_response",
                "reliability",
                config.get("fallback_enabled", False),
                "Graceful degradation when pipeline fails",
            ),

            # Observability
            ReadinessCheck(
                "tracing",
                "observability",
                config.get("tracing_enabled", False),
                "Full trace per request with stage-level spans",
            ),
            ReadinessCheck(
                "alerting",
                "observability",
                config.get("alerting_enabled", False),
                "Alerts on latency, error rate, and quality degradation",
            ),
            ReadinessCheck(
                "quality_monitoring",
                "observability",
                config.get("eval_pipeline_enabled", False),
                "Automated quality evaluation on production traffic",
            ),
        ]

        passed = sum(1 for c in checks if c.passed)
        total = len(checks)

        if passed == total:
            level = ReadinessLevel.PRODUCTION_READY
        elif passed >= total * 0.7:
            level = ReadinessLevel.PARTIALLY_READY
        else:
            level = ReadinessLevel.NOT_READY

        return {
            "readiness_level": level.value,
            "passed": passed,
            "total": total,
            "checks": [
                {
                    "name": c.name,
                    "category": c.category,
                    "passed": c.passed,
                    "detail": c.detail,
                }
                for c in checks
            ],
            "failed_checks": [
                c.name for c in checks if not c.passed
            ],
        }
```

---

## Reliability Patterns

### Retry with Exponential Backoff

```python
import asyncio
import logging
import random
from functools import wraps

logger = logging.getLogger(__name__)


def retry_with_backoff(
    max_retries: int = 3,
    base_delay: float = 1.0,
    max_delay: float = 30.0,
    retryable_exceptions: tuple = (Exception,),
):
    """Decorator for retrying RAG pipeline stages on transient failures.

    Uses exponential backoff with jitter to avoid thundering herd.
    """

    def decorator(func):
        @wraps(func)
        def wrapper(*args, **kwargs):
            last_exception = None
            for attempt in range(max_retries + 1):
                try:
                    return func(*args, **kwargs)
                except retryable_exceptions as e:
                    last_exception = e
                    if attempt < max_retries:
                        delay = min(
                            base_delay * (2 ** attempt) + random.uniform(0, 1),
                            max_delay,
                        )
                        logger.warning(
                            "Retry %d/%d for %s after %.1fs: %s",
                            attempt + 1,
                            max_retries,
                            func.__name__,
                            delay,
                            str(e),
                        )
                        import time
                        time.sleep(delay)
            raise last_exception

        return wrapper

    return decorator


class RetryableEmbeddingService:
    """Embedding service with retry logic for API failures."""

    def __init__(self, primary_client, fallback_client=None):
        self.primary = primary_client
        self.fallback = fallback_client

    @retry_with_backoff(
        max_retries=3,
        retryable_exceptions=(ConnectionError, TimeoutError),
    )
    def embed(self, text: str) -> list[float]:
        """Embed with retry. Falls back to secondary on repeated failure."""
        try:
            return self.primary.embed_query(text)
        except Exception:
            if self.fallback:
                logger.warning("Primary embedding failed, using fallback")
                return self.fallback.embed_query(text)
            raise
```

### Circuit Breaker

```python
import time
from enum import Enum


class CircuitState(Enum):
    CLOSED = "closed"      # Normal operation
    OPEN = "open"          # Failing, reject requests
    HALF_OPEN = "half_open"  # Testing recovery


class CircuitBreaker:
    """Circuit breaker for RAG pipeline external dependencies.

    When a dependency (embedding API, vector store, LLM) fails
    repeatedly, the circuit opens to prevent cascade failures
    and allow recovery time.
    """

    def __init__(
        self,
        failure_threshold: int = 5,
        recovery_timeout: float = 30.0,
        half_open_max_calls: int = 3,
    ):
        self.failure_threshold = failure_threshold
        self.recovery_timeout = recovery_timeout
        self.half_open_max_calls = half_open_max_calls

        self.state = CircuitState.CLOSED
        self.failure_count = 0
        self.last_failure_time = 0.0
        self.half_open_calls = 0

    def call(self, func, *args, **kwargs):
        """Execute function through circuit breaker."""
        if self.state == CircuitState.OPEN:
            if time.time() - self.last_failure_time > self.recovery_timeout:
                self.state = CircuitState.HALF_OPEN
                self.half_open_calls = 0
            else:
                raise CircuitOpenError(
                    f"Circuit is open, retry after "
                    f"{self.recovery_timeout}s"
                )

        if self.state == CircuitState.HALF_OPEN:
            if self.half_open_calls >= self.half_open_max_calls:
                self.state = CircuitState.OPEN
                self.last_failure_time = time.time()
                raise CircuitOpenError("Half-open test calls exhausted")
            self.half_open_calls += 1

        try:
            result = func(*args, **kwargs)
            self._on_success()
            return result
        except Exception as e:
            self._on_failure()
            raise

    def _on_success(self) -> None:
        if self.state == CircuitState.HALF_OPEN:
            self.state = CircuitState.CLOSED
        self.failure_count = 0

    def _on_failure(self) -> None:
        self.failure_count += 1
        self.last_failure_time = time.time()
        if self.failure_count >= self.failure_threshold:
            self.state = CircuitState.OPEN


class CircuitOpenError(Exception):
    pass
```

### Graceful Degradation

```python
class ResilientRAGPipeline:
    """RAG pipeline with graceful degradation on partial failures."""

    def __init__(self, retriever, reranker, llm, cache):
        self.retriever = retriever
        self.reranker = reranker
        self.llm = llm
        self.cache = cache
        self.retriever_circuit = CircuitBreaker()
        self.llm_circuit = CircuitBreaker()

    def query(self, user_query: str) -> dict:
        """Execute query with fallbacks at each stage."""

        # Try cache first
        cached = self._try_cache(user_query)
        if cached:
            return cached

        # Retrieval with fallback
        docs = self._retrieve_with_fallback(user_query)

        # Reranking with fallback (skip if unavailable)
        docs = self._rerank_with_fallback(user_query, docs)

        # Generation with fallback
        answer = self._generate_with_fallback(user_query, docs)

        return {
            "answer": answer["text"],
            "sources": [d.metadata for d in docs[:5]],
            "degraded": answer.get("degraded", False),
        }

    def _try_cache(self, query: str) -> dict | None:
        try:
            return self.cache.get(query)
        except Exception:
            return None

    def _retrieve_with_fallback(self, query: str) -> list:
        try:
            return self.retriever_circuit.call(
                self.retriever.invoke, query
            )
        except CircuitOpenError:
            logger.warning("Retriever circuit open, using empty context")
            return []

    def _rerank_with_fallback(self, query: str, docs: list) -> list:
        if not docs:
            return docs
        try:
            return self.reranker.rerank(query, docs)
        except Exception:
            logger.warning("Reranker failed, using retrieval order")
            return docs

    def _generate_with_fallback(
        self, query: str, docs: list
    ) -> dict:
        context = "\n\n".join(
            d.page_content for d in docs[:5]
        ) if docs else ""

        try:
            answer = self.llm_circuit.call(
                self.llm.invoke,
                f"Context:\n{context}\n\nQuestion: {query}",
            )
            return {"text": answer.content, "degraded": False}
        except CircuitOpenError:
            if docs:
                return {
                    "text": (
                        "I am currently unable to generate a full answer. "
                        "Here are the most relevant document excerpts:\n\n"
                        + "\n---\n".join(
                            d.page_content[:200] for d in docs[:3]
                        )
                    ),
                    "degraded": True,
                }
            return {
                "text": (
                    "I am temporarily unable to answer. "
                    "Please try again in a few moments."
                ),
                "degraded": True,
            }
```

---

## Configuration Management

```python
from pydantic import BaseModel, Field


class RAGProductionConfig(BaseModel):
    """Configuration for production RAG deployment."""

    # Retrieval
    retrieval_top_k: int = Field(default=10, ge=1, le=100)
    rerank_top_k: int = Field(default=5, ge=1, le=50)
    similarity_threshold: float = Field(default=0.3, ge=0.0, le=1.0)

    # Caching
    semantic_cache_enabled: bool = True
    semantic_cache_ttl: int = Field(default=1800, ge=60)
    semantic_cache_threshold: float = Field(default=0.93, ge=0.8, le=1.0)

    # Reliability
    max_retries: int = Field(default=3, ge=0, le=10)
    circuit_breaker_threshold: int = Field(default=5, ge=1)
    circuit_breaker_timeout: float = Field(default=30.0, ge=5.0)
    request_timeout: float = Field(default=30.0, ge=5.0, le=120.0)

    # LLM
    model: str = "claude-sonnet-4-20250514"
    max_tokens: int = Field(default=2048, ge=100, le=8192)
    temperature: float = Field(default=0.0, ge=0.0, le=1.0)

    # Scaling
    max_concurrent_requests: int = Field(default=50, ge=1)

    # Quality
    min_faithfulness_score: float = Field(default=0.7, ge=0.0, le=1.0)
    guardrails_enabled: bool = True
```

---

## Health Check Endpoint

```python
import time
from dataclasses import dataclass


@dataclass
class HealthStatus:
    healthy: bool
    checks: dict[str, dict]
    timestamp: float


class RAGHealthChecker:
    """Health check for production RAG system."""

    def __init__(self, retriever, llm, cache):
        self.retriever = retriever
        self.llm = llm
        self.cache = cache

    def check(self) -> HealthStatus:
        """Run all health checks."""
        checks = {}

        # Vector store health
        checks["vector_store"] = self._check_vector_store()

        # LLM health
        checks["llm"] = self._check_llm()

        # Cache health
        checks["cache"] = self._check_cache()

        all_healthy = all(c["healthy"] for c in checks.values())

        return HealthStatus(
            healthy=all_healthy,
            checks=checks,
            timestamp=time.time(),
        )

    def _check_vector_store(self) -> dict:
        try:
            start = time.perf_counter()
            results = self.retriever.invoke("health check test query")
            latency = (time.perf_counter() - start) * 1000
            return {
                "healthy": True,
                "latency_ms": latency,
                "result_count": len(results),
            }
        except Exception as e:
            return {"healthy": False, "error": str(e)}

    def _check_llm(self) -> dict:
        try:
            start = time.perf_counter()
            response = self.llm.invoke("Say 'ok'")
            latency = (time.perf_counter() - start) * 1000
            return {
                "healthy": True,
                "latency_ms": latency,
            }
        except Exception as e:
            return {"healthy": False, "error": str(e)}

    def _check_cache(self) -> dict:
        try:
            self.cache.redis.ping()
            return {"healthy": True}
        except Exception as e:
            return {"healthy": False, "error": str(e)}
```

---

## Common Pitfalls

1. **No retry logic on embedding API calls.** Embedding APIs have transient failures. Without retries, a single timeout during ingestion silently drops documents from the index.
2. **Reindexing the entire corpus on every document change.** Full reindexing takes hours for large corpora. Use incremental indexing (see indexing article) to process only changed documents.
3. **No circuit breaker on external dependencies.** When the LLM API goes down, every request blocks for the full timeout duration. Circuit breakers fail fast and allow recovery.
4. **Treating all tenants equally.** Premium tenants expect lower latency and higher quality. Implement tiered configurations with different top-k, model selection, and SLA targets per tier.
5. **No health check endpoint.** Without health checks, load balancers route traffic to unhealthy instances. Implement deep health checks that verify connectivity to the vector store, LLM, and cache.
6. **Deploying without a rollback plan.** When a new model, embedding, or retrieval configuration degrades quality, you need to roll back immediately. Use feature flags and shadow mode to validate changes before full rollout.

---

## References

- Pinecone production RAG guide: https://www.pinecone.io/learn/production-rag/
- Weaviate production deployment: https://weaviate.io/developers/weaviate/deployment
- Netflix resilience patterns: https://netflixtechblog.com/
- Microsoft retry pattern: https://learn.microsoft.com/en-us/azure/architecture/patterns/retry
