# Spring AI - RAG (Retrieval-Augmented Generation)

## Overview

RAG combines retrieval of relevant documents with language model generation to provide accurate, grounded responses. Spring AI provides comprehensive support for building RAG applications.

## Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                        RAG Pipeline                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  1. Document Loading    2. Chunking      3. Embedding           │
│  ┌─────────────┐       ┌──────────┐     ┌────────────┐         │
│  │   PDFs      │──────▶│  Split   │────▶│  Embed     │         │
│  │   Text      │       │  Chunks  │     │  Vectors   │         │
│  │   Markdown  │       └──────────┘     └────────────┘         │
│  └─────────────┘                              │                 │
│                                               ▼                 │
│  4. Storage              5. Retrieval    ┌────────────┐        │
│  ┌─────────────┐       ┌──────────┐     │  Vector    │        │
│  │   Vector    │◀──────│  Query   │◀────│  Store     │        │
│  │   Store     │       │  Embed   │     └────────────┘        │
│  └─────────────┘       └──────────┘                            │
│        │                                                        │
│        ▼                                                        │
│  6. Generation                                                  │
│  ┌─────────────────────────────────────────────────────┐       │
│  │  Query + Retrieved Context ──▶ LLM ──▶ Response     │       │
│  └─────────────────────────────────────────────────────┘       │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

## Dependencies

```xml
<dependency>
    <groupId>org.springframework.ai</groupId>
    <artifactId>spring-ai-openai-spring-boot-starter</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.ai</groupId>
    <artifactId>spring-ai-pgvector-store-spring-boot-starter</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.ai</groupId>
    <artifactId>spring-ai-pdf-document-reader</artifactId>
</dependency>
```

## Document Loading

### PDF Reader
```java
@Service
public class DocumentLoaderService {

    public List<Document> loadPdf(Resource pdfResource) {
        PdfDocumentReader reader = new PdfDocumentReader(pdfResource);
        return reader.read();
    }

    public List<Document> loadPdfWithMetadata(Resource pdfResource) {
        PdfDocumentReader reader = new PdfDocumentReader(
            pdfResource,
            PdfDocumentReaderConfig.builder()
                .withPageExtractedTextFormatter(ExtractedTextFormatter.defaults())
                .withPagesPerDocument(1)
                .build()
        );

        return reader.read().stream()
            .map(doc -> doc.mutate()
                .withMetadata("source", pdfResource.getFilename())
                .build())
            .collect(Collectors.toList());
    }
}
```

### Text and Markdown
```java
@Service
public class TextLoaderService {

    public List<Document> loadTextFile(Resource resource) {
        TextReader reader = new TextReader(resource);
        return reader.read();
    }

    public List<Document> loadMarkdown(Resource resource) {
        return new TextReader(resource).read();
    }

    public List<Document> loadFromDirectory(String path) throws IOException {
        List<Document> documents = new ArrayList<>();

        Files.walk(Path.of(path))
            .filter(Files::isRegularFile)
            .filter(p -> p.toString().endsWith(".txt") || p.toString().endsWith(".md"))
            .forEach(file -> {
                Resource resource = new FileSystemResource(file.toFile());
                documents.addAll(new TextReader(resource).read());
            });

        return documents;
    }
}
```

## Document Chunking

### Token-based Chunking
```java
@Service
public class ChunkingService {

    public List<Document> chunkDocuments(List<Document> documents) {
        TokenTextSplitter splitter = new TokenTextSplitter();
        return splitter.apply(documents);
    }

    public List<Document> chunkWithConfig(List<Document> documents) {
        TokenTextSplitter splitter = new TokenTextSplitter(
            800,    // Default chunk size
            350,    // Min chunk size
            5,      // Min chunk length chars
            10000,  // Max tokens per document
            true    // Keep separator
        );
        return splitter.apply(documents);
    }
}
```

### Recursive Character Splitting
```java
@Service
public class RecursiveChunkingService {

    public List<Document> chunk(List<Document> documents) {
        // Split by paragraphs, then sentences, then words
        List<String> separators = List.of("\n\n", "\n", ". ", " ");

        return documents.stream()
            .flatMap(doc -> splitRecursively(doc.getContent(), separators, 1000, 200).stream())
            .map(Document::from)
            .collect(Collectors.toList());
    }

    private List<String> splitRecursively(String text, List<String> separators,
                                          int chunkSize, int overlap) {
        // Implementation of recursive splitting
        List<String> chunks = new ArrayList<>();
        // ... splitting logic
        return chunks;
    }
}
```

## Vector Store Integration

### Configuration
```yaml
spring:
  ai:
    vectorstore:
      pgvector:
        dimensions: 1536
        index-type: HNSW
        distance-type: COSINE_DISTANCE
```

### Indexing Documents
```java
@Service
@RequiredArgsConstructor
public class IndexingService {

    private final VectorStore vectorStore;
    private final ChunkingService chunkingService;
    private final DocumentLoaderService loaderService;

    public void indexPdf(Resource pdfResource) {
        // Load
        List<Document> documents = loaderService.loadPdf(pdfResource);

        // Chunk
        List<Document> chunks = chunkingService.chunkDocuments(documents);

        // Add metadata
        chunks.forEach(chunk -> chunk.getMetadata()
            .put("source", pdfResource.getFilename()));

        // Store (embeddings created automatically)
        vectorStore.add(chunks);
    }

    public void indexBatch(List<Resource> resources) {
        List<Document> allChunks = resources.stream()
            .flatMap(r -> loaderService.loadPdf(r).stream())
            .collect(Collectors.toList());

        List<Document> chunks = chunkingService.chunkDocuments(allChunks);
        vectorStore.add(chunks);
    }
}
```

## Retrieval

### Basic Retrieval
```java
@Service
@RequiredArgsConstructor
public class RetrievalService {

    private final VectorStore vectorStore;

    public List<Document> retrieve(String query, int topK) {
        return vectorStore.similaritySearch(
            SearchRequest.query(query).withTopK(topK)
        );
    }

    public List<Document> retrieveWithThreshold(String query, double threshold) {
        return vectorStore.similaritySearch(
            SearchRequest.query(query)
                .withTopK(10)
                .withSimilarityThreshold(threshold)
        );
    }

    public List<Document> retrieveWithFilter(String query, String source) {
        return vectorStore.similaritySearch(
            SearchRequest.query(query)
                .withTopK(5)
                .withFilterExpression("source == '" + source + "'")
        );
    }
}
```

### Hybrid Retrieval
```java
@Service
@RequiredArgsConstructor
public class HybridRetrievalService {

    private final VectorStore vectorStore;
    private final KeywordSearchService keywordSearch;

    public List<Document> hybridSearch(String query, int topK) {
        // Semantic search
        List<Document> semanticResults = vectorStore.similaritySearch(
            SearchRequest.query(query).withTopK(topK * 2)
        );

        // Keyword search
        List<Document> keywordResults = keywordSearch.search(query, topK * 2);

        // Merge and re-rank using Reciprocal Rank Fusion
        return reciprocalRankFusion(semanticResults, keywordResults, topK);
    }

    private List<Document> reciprocalRankFusion(
            List<Document> list1, List<Document> list2, int topK) {

        Map<String, Double> scores = new HashMap<>();
        int k = 60; // RRF constant

        for (int i = 0; i < list1.size(); i++) {
            String id = list1.get(i).getId();
            scores.merge(id, 1.0 / (k + i + 1), Double::sum);
        }

        for (int i = 0; i < list2.size(); i++) {
            String id = list2.get(i).getId();
            scores.merge(id, 1.0 / (k + i + 1), Double::sum);
        }

        Map<String, Document> docMap = Stream.concat(list1.stream(), list2.stream())
            .collect(Collectors.toMap(Document::getId, d -> d, (a, b) -> a));

        return scores.entrySet().stream()
            .sorted(Map.Entry.<String, Double>comparingByValue().reversed())
            .limit(topK)
            .map(e -> docMap.get(e.getKey()))
            .collect(Collectors.toList());
    }
}
```

## RAG with ChatClient

### Basic RAG
```java
@Service
@RequiredArgsConstructor
public class RagService {

    private final ChatClient chatClient;
    private final VectorStore vectorStore;

    public String query(String question) {
        // Retrieve relevant documents
        List<Document> relevantDocs = vectorStore.similaritySearch(
            SearchRequest.query(question).withTopK(5)
        );

        String context = relevantDocs.stream()
            .map(Document::getContent)
            .collect(Collectors.joining("\n\n"));

        // Generate response
        return chatClient.prompt()
            .system("""
                Answer the question based on the following context.
                If the answer is not in the context, say "I don't know."

                Context:
                {context}
                """)
            .user(question)
            .call()
            .content();
    }
}
```

### RAG with QuestionAnswerAdvisor
```java
@Configuration
public class RagConfig {

    @Bean
    public ChatClient ragChatClient(ChatModel model, VectorStore vectorStore) {
        return ChatClient.builder(model)
            .defaultAdvisors(
                new QuestionAnswerAdvisor(vectorStore, SearchRequest.defaults())
            )
            .build();
    }
}

@Service
@RequiredArgsConstructor
public class AdvisedRagService {

    private final ChatClient ragChatClient;

    public String query(String question) {
        // QuestionAnswerAdvisor automatically retrieves and injects context
        return ragChatClient.prompt()
            .user(question)
            .call()
            .content();
    }
}
```

## Advanced RAG Patterns

### Query Rewriting
```java
@Service
@RequiredArgsConstructor
public class QueryRewriteRagService {

    private final ChatClient chatClient;
    private final VectorStore vectorStore;

    public String query(String userQuery) {
        // Rewrite query for better retrieval
        String rewrittenQuery = chatClient.prompt()
            .system("Rewrite the following question to be more specific and searchable:")
            .user(userQuery)
            .call()
            .content();

        // Retrieve with improved query
        List<Document> docs = vectorStore.similaritySearch(
            SearchRequest.query(rewrittenQuery).withTopK(5)
        );

        // Generate answer
        return generateAnswer(userQuery, docs);
    }
}
```

### Multi-Query Retrieval
```java
@Service
public class MultiQueryRagService {

    private final ChatClient chatClient;
    private final VectorStore vectorStore;

    public String query(String userQuery) {
        // Generate multiple query variations
        List<String> queries = generateQueryVariations(userQuery);

        // Retrieve for each query
        Set<Document> allDocs = new HashSet<>();
        for (String q : queries) {
            allDocs.addAll(vectorStore.similaritySearch(
                SearchRequest.query(q).withTopK(3)
            ));
        }

        // Generate answer from all retrieved docs
        return generateAnswer(userQuery, new ArrayList<>(allDocs));
    }

    private List<String> generateQueryVariations(String query) {
        String response = chatClient.prompt()
            .system("""
                Generate 3 different versions of the following question.
                Return each version on a new line.
                """)
            .user(query)
            .call()
            .content();

        return Arrays.asList(response.split("\n"));
    }
}
```

### Conversational RAG
```java
@Service
@RequiredArgsConstructor
public class ConversationalRagService {

    private final ChatClient chatClient;
    private final VectorStore vectorStore;
    private final ChatMemory chatMemory;

    public String query(String sessionId, String question) {
        // Get conversation history
        List<Message> history = chatMemory.get(sessionId);

        // Contextualize question with history
        String standaloneQuestion = contextualizeQuestion(history, question);

        // Retrieve
        List<Document> docs = vectorStore.similaritySearch(
            SearchRequest.query(standaloneQuestion).withTopK(5)
        );

        // Generate
        String answer = generateAnswer(standaloneQuestion, docs, history);

        // Update memory
        chatMemory.add(sessionId, new UserMessage(question));
        chatMemory.add(sessionId, new AssistantMessage(answer));

        return answer;
    }

    private String contextualizeQuestion(List<Message> history, String question) {
        if (history.isEmpty()) {
            return question;
        }

        return chatClient.prompt()
            .system("""
                Given the conversation history, reformulate the follow-up question
                to be a standalone question that captures the full context.
                """)
            .messages(history)
            .user(question)
            .call()
            .content();
    }
}
```

## Best Practices

| Do | Don't |
|----|-------|
| Tune chunk size for content | One-size-fits-all chunking |
| Add meaningful metadata | Skip metadata enrichment |
| Use appropriate similarity threshold | Return irrelevant documents |
| Implement query rewriting | Use raw user queries |
| Cache embeddings | Re-embed same content |
| Monitor retrieval quality | Ignore precision/recall |

## Production Checklist

- [ ] Document loading pipeline tested
- [ ] Chunk size optimized for content
- [ ] Vector store indexed properly
- [ ] Retrieval threshold tuned
- [ ] Prompt template refined
- [ ] Conversation memory configured
- [ ] Error handling implemented
- [ ] Monitoring and logging enabled
