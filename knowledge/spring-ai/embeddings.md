# Spring AI - Embeddings

## Overview

Spring AI provides embedding models for converting text into vector representations, enabling semantic search, similarity comparison, and RAG applications.

## Configuration

### OpenAI Embeddings
```yaml
spring:
  ai:
    openai:
      api-key: ${OPENAI_API_KEY}
      embedding:
        options:
          model: text-embedding-3-small
```

### Azure OpenAI Embeddings
```yaml
spring:
  ai:
    azure:
      openai:
        api-key: ${AZURE_OPENAI_API_KEY}
        endpoint: ${AZURE_OPENAI_ENDPOINT}
        embedding:
          options:
            deployment-name: text-embedding-ada-002
```

### Ollama (Local)
```yaml
spring:
  ai:
    ollama:
      base-url: http://localhost:11434
      embedding:
        options:
          model: nomic-embed-text
```

## EmbeddingModel API

### Basic Usage
```java
@Service
@RequiredArgsConstructor
public class EmbeddingService {

    private final EmbeddingModel embeddingModel;

    public float[] embed(String text) {
        EmbeddingResponse response = embeddingModel.embed(text);
        return response.getResult().getOutput();
    }

    public List<float[]> embedBatch(List<String> texts) {
        EmbeddingResponse response = embeddingModel.embed(texts);
        return response.getResults().stream()
            .map(Embedding::getOutput)
            .collect(Collectors.toList());
    }
}
```

### Using EmbeddingRequest
```java
@Service
public class AdvancedEmbeddingService {

    private final EmbeddingModel embeddingModel;

    public List<Embedding> embedWithOptions(List<String> texts) {
        EmbeddingRequest request = new EmbeddingRequest(texts,
            EmbeddingOptions.builder()
                .withModel("text-embedding-3-large")
                .withDimensions(1024)
                .build());

        EmbeddingResponse response = embeddingModel.call(request);
        return response.getResults();
    }
}
```

## Document Embedding

### Embedding Documents
```java
@Service
@RequiredArgsConstructor
public class DocumentEmbeddingService {

    private final EmbeddingModel embeddingModel;

    public List<Document> embedDocuments(List<Document> documents) {
        List<String> contents = documents.stream()
            .map(Document::getContent)
            .collect(Collectors.toList());

        EmbeddingResponse response = embeddingModel.embed(contents);

        for (int i = 0; i < documents.size(); i++) {
            float[] embedding = response.getResults().get(i).getOutput();
            documents.get(i).setEmbedding(embedding);
        }

        return documents;
    }
}
```

### Document Class
```java
public class Document {

    private String id;
    private String content;
    private Map<String, Object> metadata;
    private float[] embedding;

    // Builder pattern
    public static Document from(String content) {
        return new Document(UUID.randomUUID().toString(), content, new HashMap<>());
    }

    public Document withMetadata(String key, Object value) {
        this.metadata.put(key, value);
        return this;
    }
}
```

## Similarity Computation

### Cosine Similarity
```java
@Service
public class SimilarityService {

    private final EmbeddingModel embeddingModel;

    public double computeSimilarity(String text1, String text2) {
        EmbeddingResponse response = embeddingModel.embed(List.of(text1, text2));

        float[] embedding1 = response.getResults().get(0).getOutput();
        float[] embedding2 = response.getResults().get(1).getOutput();

        return cosineSimilarity(embedding1, embedding2);
    }

    private double cosineSimilarity(float[] a, float[] b) {
        double dotProduct = 0.0;
        double normA = 0.0;
        double normB = 0.0;

        for (int i = 0; i < a.length; i++) {
            dotProduct += a[i] * b[i];
            normA += a[i] * a[i];
            normB += b[i] * b[i];
        }

        return dotProduct / (Math.sqrt(normA) * Math.sqrt(normB));
    }

    public List<SimilarityResult> findSimilar(String query, List<Document> documents, int topK) {
        float[] queryEmbedding = embeddingModel.embed(query).getResult().getOutput();

        return documents.stream()
            .map(doc -> new SimilarityResult(doc,
                cosineSimilarity(queryEmbedding, doc.getEmbedding())))
            .sorted((a, b) -> Double.compare(b.score(), a.score()))
            .limit(topK)
            .collect(Collectors.toList());
    }
}

public record SimilarityResult(Document document, double score) {}
```

## Caching Embeddings

```java
@Service
@RequiredArgsConstructor
public class CachedEmbeddingService {

    private final EmbeddingModel embeddingModel;
    private final Cache<String, float[]> embeddingCache;

    public float[] getEmbedding(String text) {
        String cacheKey = generateCacheKey(text);

        return embeddingCache.get(cacheKey, key -> {
            EmbeddingResponse response = embeddingModel.embed(text);
            return response.getResult().getOutput();
        });
    }

    private String generateCacheKey(String text) {
        return DigestUtils.sha256Hex(text);
    }
}

@Configuration
public class EmbeddingCacheConfig {

    @Bean
    public Cache<String, float[]> embeddingCache() {
        return Caffeine.newBuilder()
            .maximumSize(10_000)
            .expireAfterWrite(Duration.ofHours(24))
            .build();
    }
}
```

## Batch Processing

```java
@Service
@RequiredArgsConstructor
@Slf4j
public class BatchEmbeddingService {

    private final EmbeddingModel embeddingModel;

    private static final int BATCH_SIZE = 100;

    public List<Document> embedDocumentsInBatches(List<Document> documents) {
        List<Document> result = new ArrayList<>();

        for (int i = 0; i < documents.size(); i += BATCH_SIZE) {
            int end = Math.min(i + BATCH_SIZE, documents.size());
            List<Document> batch = documents.subList(i, end);

            log.info("Processing batch {}/{}", (i / BATCH_SIZE) + 1,
                (documents.size() / BATCH_SIZE) + 1);

            List<Document> embeddedBatch = embedBatch(batch);
            result.addAll(embeddedBatch);

            // Rate limiting
            if (end < documents.size()) {
                sleep(100);
            }
        }

        return result;
    }

    private List<Document> embedBatch(List<Document> batch) {
        List<String> contents = batch.stream()
            .map(Document::getContent)
            .collect(Collectors.toList());

        EmbeddingResponse response = embeddingModel.embed(contents);

        for (int i = 0; i < batch.size(); i++) {
            batch.get(i).setEmbedding(response.getResults().get(i).getOutput());
        }

        return batch;
    }

    private void sleep(long millis) {
        try {
            Thread.sleep(millis);
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }
    }
}
```

## Integration with Vector Stores

```java
@Service
@RequiredArgsConstructor
public class VectorStoreService {

    private final EmbeddingModel embeddingModel;
    private final VectorStore vectorStore;

    public void indexDocuments(List<Document> documents) {
        // Embeddings are created automatically by VectorStore
        vectorStore.add(documents);
    }

    public List<Document> search(String query, int topK) {
        return vectorStore.similaritySearch(
            SearchRequest.query(query).withTopK(topK)
        );
    }

    public List<Document> searchWithThreshold(String query, int topK, double threshold) {
        return vectorStore.similaritySearch(
            SearchRequest.query(query)
                .withTopK(topK)
                .withSimilarityThreshold(threshold)
        );
    }

    public List<Document> searchWithFilter(String query, Map<String, Object> filter) {
        return vectorStore.similaritySearch(
            SearchRequest.query(query)
                .withTopK(10)
                .withFilterExpression(buildFilterExpression(filter))
        );
    }

    private String buildFilterExpression(Map<String, Object> filter) {
        return filter.entrySet().stream()
            .map(e -> e.getKey() + " == '" + e.getValue() + "'")
            .collect(Collectors.joining(" && "));
    }
}
```

## Embedding Dimensions

```java
@Service
public class DimensionAwareEmbeddingService {

    private final EmbeddingModel embeddingModel;

    public int getEmbeddingDimension() {
        return embeddingModel.dimensions();
    }

    public float[] embedWithDimensions(String text, int targetDimensions) {
        EmbeddingRequest request = new EmbeddingRequest(List.of(text),
            EmbeddingOptions.builder()
                .withDimensions(targetDimensions)
                .build());

        return embeddingModel.call(request).getResult().getOutput();
    }
}
```

## Best Practices

| Do | Don't |
|----|-------|
| Batch embed for efficiency | Embed one at a time in loops |
| Cache embeddings when possible | Re-compute same embeddings |
| Choose model size for use case | Always use largest model |
| Handle rate limits gracefully | Ignore API limits |
| Normalize embeddings if needed | Assume normalization |
| Monitor embedding costs | Ignore usage metrics |

## Production Checklist

- [ ] Appropriate embedding model selected
- [ ] Batch processing implemented
- [ ] Caching strategy defined
- [ ] Rate limiting handled
- [ ] Cost monitoring enabled
- [ ] Dimension consistency verified
- [ ] Error handling implemented
