# Spring Data Elasticsearch - Basics

## Overview

Spring Data Elasticsearch provides integration with Elasticsearch, offering repository-based access, template operations, and object mapping for search-optimized data access.

## Dependencies

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-elasticsearch</artifactId>
</dependency>
```

## Configuration

### application.yml
```yaml
spring:
  elasticsearch:
    uris: http://localhost:9200
    username: ${ELASTICSEARCH_USERNAME:}
    password: ${ELASTICSEARCH_PASSWORD:}
    connection-timeout: 5s
    socket-timeout: 30s
```

### Java Configuration
```java
@Configuration
public class ElasticsearchConfig extends ElasticsearchConfiguration {

    @Override
    public ClientConfiguration clientConfiguration() {
        return ClientConfiguration.builder()
            .connectedTo("localhost:9200")
            .usingSsl()
            .withBasicAuth("elastic", "password")
            .withConnectTimeout(Duration.ofSeconds(5))
            .withSocketTimeout(Duration.ofSeconds(30))
            .build();
    }
}
```

### Cluster Configuration
```java
@Configuration
public class ElasticsearchClusterConfig extends ElasticsearchConfiguration {

    @Override
    public ClientConfiguration clientConfiguration() {
        return ClientConfiguration.builder()
            .connectedTo("node1:9200", "node2:9200", "node3:9200")
            .withBasicAuth("elastic", "password")
            .withClientConfigurer(
                ElasticsearchClients.ElasticsearchHttpClientConfigurationCallback.from(
                    httpClientBuilder -> {
                        httpClientBuilder.setDefaultRequestConfig(
                            RequestConfig.custom()
                                .setConnectionRequestTimeout(5000)
                                .setSocketTimeout(30000)
                                .build()
                        );
                        return httpClientBuilder;
                    }
                )
            )
            .build();
    }
}
```

## Document Mapping

### Basic Entity
```java
@Document(indexName = "products")
public class Product {

    @Id
    private String id;

    @Field(type = FieldType.Text, analyzer = "standard")
    private String name;

    @Field(type = FieldType.Text, analyzer = "english")
    private String description;

    @Field(type = FieldType.Keyword)
    private String category;

    @Field(type = FieldType.Double)
    private BigDecimal price;

    @Field(type = FieldType.Integer)
    private Integer stock;

    @Field(type = FieldType.Boolean)
    private Boolean active;

    @Field(type = FieldType.Date, format = DateFormat.date_hour_minute_second)
    private LocalDateTime createdAt;

    @Field(type = FieldType.Nested)
    private List<ProductAttribute> attributes;

    // Getters and setters
}

public class ProductAttribute {
    @Field(type = FieldType.Keyword)
    private String name;

    @Field(type = FieldType.Keyword)
    private String value;
}
```

### Index Settings
```java
@Document(indexName = "products")
@Setting(settingPath = "/elasticsearch/settings/products.json")
@Mapping(mappingPath = "/elasticsearch/mappings/products.json")
public class Product {
    // ...
}

// Or programmatic settings
@Document(indexName = "products")
@Setting(
    shards = 3,
    replicas = 1,
    refreshInterval = "1s"
)
public class Product {
    // ...
}
```

### settings.json
```json
{
  "index": {
    "number_of_shards": 3,
    "number_of_replicas": 1,
    "analysis": {
      "analyzer": {
        "autocomplete": {
          "type": "custom",
          "tokenizer": "standard",
          "filter": ["lowercase", "autocomplete_filter"]
        }
      },
      "filter": {
        "autocomplete_filter": {
          "type": "edge_ngram",
          "min_gram": 2,
          "max_gram": 20
        }
      }
    }
  }
}
```

## ElasticsearchOperations

```java
@Service
@RequiredArgsConstructor
public class ProductSearchService {

    private final ElasticsearchOperations elasticsearchOperations;

    public Product save(Product product) {
        return elasticsearchOperations.save(product);
    }

    public List<Product> saveAll(List<Product> products) {
        return StreamSupport
            .stream(elasticsearchOperations.save(products).spliterator(), false)
            .collect(Collectors.toList());
    }

    public Optional<Product> findById(String id) {
        return Optional.ofNullable(
            elasticsearchOperations.get(id, Product.class)
        );
    }

    public void delete(String id) {
        elasticsearchOperations.delete(id, Product.class);
    }

    public long count() {
        return elasticsearchOperations.count(
            Query.findAll(), Product.class
        );
    }

    public boolean exists(String id) {
        return elasticsearchOperations.exists(id, Product.class);
    }
}
```

## Index Management

```java
@Service
@RequiredArgsConstructor
public class IndexManagementService {

    private final ElasticsearchOperations operations;
    private final IndexOperations indexOperations;

    @PostConstruct
    public void init() {
        IndexOperations ops = operations.indexOps(Product.class);

        // Create index if not exists
        if (!ops.exists()) {
            ops.create();
            ops.putMapping(ops.createMapping());
        }
    }

    public void recreateIndex(Class<?> entityClass) {
        IndexOperations ops = operations.indexOps(entityClass);

        if (ops.exists()) {
            ops.delete();
        }

        ops.create();
        ops.putMapping(ops.createMapping());
    }

    public void refresh(Class<?> entityClass) {
        operations.indexOps(entityClass).refresh();
    }

    public Map<String, Object> getMapping(Class<?> entityClass) {
        return operations.indexOps(entityClass).getMapping();
    }

    public Settings getSettings(Class<?> entityClass) {
        return operations.indexOps(entityClass).getSettings();
    }
}
```

## Bulk Operations

```java
@Service
@RequiredArgsConstructor
public class BulkOperationsService {

    private final ElasticsearchOperations operations;

    public void bulkIndex(List<Product> products) {
        List<IndexQuery> queries = products.stream()
            .map(product -> new IndexQueryBuilder()
                .withId(product.getId())
                .withObject(product)
                .build())
            .collect(Collectors.toList());

        operations.bulkIndex(queries, Product.class);
    }

    public void bulkUpdate(List<Product> products) {
        List<UpdateQuery> queries = products.stream()
            .map(product -> UpdateQuery.builder(product.getId())
                .withDocument(Document.from(
                    Map.of("price", product.getPrice())))
                .build())
            .collect(Collectors.toList());

        operations.bulkUpdate(queries, IndexCoordinates.of("products"));
    }

    public void bulkDelete(List<String> ids) {
        ids.forEach(id -> operations.delete(id, Product.class));
    }
}
```

## Reactive Elasticsearch

```java
@Configuration
@EnableReactiveElasticsearchRepositories
public class ReactiveElasticsearchConfig extends ReactiveElasticsearchConfiguration {

    @Override
    public ReactiveElasticsearchClient reactiveElasticsearchClient() {
        ClientConfiguration config = ClientConfiguration.builder()
            .connectedTo("localhost:9200")
            .build();
        return ReactiveRestClients.create(config);
    }
}

@Service
@RequiredArgsConstructor
public class ReactiveProductService {

    private final ReactiveElasticsearchOperations operations;

    public Mono<Product> save(Product product) {
        return operations.save(product);
    }

    public Flux<Product> saveAll(List<Product> products) {
        return operations.saveAll(Mono.just(products), Product.class);
    }

    public Mono<Product> findById(String id) {
        return operations.get(id, Product.class);
    }

    public Flux<Product> search(Query query) {
        return operations.search(query, Product.class)
            .map(SearchHit::getContent);
    }

    public Mono<Long> count() {
        return operations.count(Query.findAll(), Product.class);
    }
}
```

## Error Handling

```java
@Service
@Slf4j
public class SafeElasticsearchService {

    @Autowired
    private ElasticsearchOperations operations;

    public Optional<Product> safeFindById(String id) {
        try {
            return Optional.ofNullable(operations.get(id, Product.class));
        } catch (ElasticsearchException e) {
            log.error("Elasticsearch error finding product {}: {}", id, e.getMessage());
            return Optional.empty();
        }
    }

    public boolean safeIndex(Product product) {
        try {
            operations.save(product);
            return true;
        } catch (ElasticsearchException e) {
            log.error("Failed to index product: {}", e.getMessage());
            return false;
        }
    }
}
```

## Best Practices

| Do | Don't |
|----|-------|
| Define proper mappings | Rely on dynamic mapping |
| Use appropriate field types | Use Text for exact matches |
| Configure proper analyzers | Use default for all fields |
| Implement bulk operations | Index documents one by one |
| Set refresh interval appropriately | Refresh after every write |
| Monitor cluster health | Ignore yellow/red status |

## Production Checklist

- [ ] Proper index mappings defined
- [ ] Analyzers configured for text fields
- [ ] Shard and replica count set
- [ ] Connection pooling configured
- [ ] Timeout values appropriate
- [ ] Authentication enabled
- [ ] SSL/TLS configured
- [ ] Index lifecycle management set up
