# Spring Data Elasticsearch - Mapping

## Overview

Elasticsearch mappings define how documents and their fields are stored and indexed. Proper mapping is crucial for search performance and accuracy.

## Field Types

### Basic Field Types
```java
@Document(indexName = "products")
public class Product {

    @Id
    private String id;

    // Text - full-text search, analyzed
    @Field(type = FieldType.Text)
    private String name;

    // Keyword - exact match, not analyzed
    @Field(type = FieldType.Keyword)
    private String sku;

    // Numbers
    @Field(type = FieldType.Integer)
    private Integer quantity;

    @Field(type = FieldType.Long)
    private Long viewCount;

    @Field(type = FieldType.Double)
    private Double price;

    @Field(type = FieldType.Float)
    private Float rating;

    // Boolean
    @Field(type = FieldType.Boolean)
    private Boolean active;

    // Date
    @Field(type = FieldType.Date, format = DateFormat.date_hour_minute_second)
    private LocalDateTime createdAt;

    // Binary
    @Field(type = FieldType.Binary)
    private byte[] thumbnail;

    // IP
    @Field(type = FieldType.Ip)
    private String ipAddress;
}
```

### Multi-Field Mapping
```java
@Document(indexName = "products")
public class Product {

    // Text with keyword subfield
    @MultiField(
        mainField = @Field(type = FieldType.Text, analyzer = "standard"),
        otherFields = {
            @InnerField(suffix = "keyword", type = FieldType.Keyword),
            @InnerField(suffix = "autocomplete", type = FieldType.Text,
                       analyzer = "autocomplete")
        }
    )
    private String name;
}

// Usage:
// name - full-text search
// name.keyword - exact match, sorting, aggregations
// name.autocomplete - autocomplete search
```

### Nested Objects
```java
@Document(indexName = "products")
public class Product {

    @Id
    private String id;

    // Nested - separate documents, maintains relationships
    @Field(type = FieldType.Nested)
    private List<Variant> variants;

    // Object - flattened, loses relationships
    @Field(type = FieldType.Object)
    private Manufacturer manufacturer;
}

public class Variant {
    @Field(type = FieldType.Keyword)
    private String color;

    @Field(type = FieldType.Keyword)
    private String size;

    @Field(type = FieldType.Double)
    private Double price;

    @Field(type = FieldType.Integer)
    private Integer stock;
}

public class Manufacturer {
    @Field(type = FieldType.Keyword)
    private String name;

    @Field(type = FieldType.Keyword)
    private String country;
}
```

## Analyzers

### Built-in Analyzers
```java
@Document(indexName = "products")
public class Product {

    // Standard analyzer (default)
    @Field(type = FieldType.Text, analyzer = "standard")
    private String name;

    // Language-specific
    @Field(type = FieldType.Text, analyzer = "english")
    private String description;

    // Simple - lowercase only
    @Field(type = FieldType.Text, analyzer = "simple")
    private String tags;

    // Whitespace - splits on whitespace only
    @Field(type = FieldType.Text, analyzer = "whitespace")
    private String codes;

    // Keyword - no analysis
    @Field(type = FieldType.Text, analyzer = "keyword")
    private String exactField;
}
```

### Custom Analyzers
```json
// resources/elasticsearch/settings/custom-analyzers.json
{
  "analysis": {
    "analyzer": {
      "autocomplete": {
        "type": "custom",
        "tokenizer": "standard",
        "filter": ["lowercase", "autocomplete_filter"]
      },
      "autocomplete_search": {
        "type": "custom",
        "tokenizer": "standard",
        "filter": ["lowercase"]
      },
      "html_strip": {
        "type": "custom",
        "char_filter": ["html_strip"],
        "tokenizer": "standard",
        "filter": ["lowercase", "stop"]
      },
      "path_analyzer": {
        "type": "custom",
        "tokenizer": "path_tokenizer"
      }
    },
    "filter": {
      "autocomplete_filter": {
        "type": "edge_ngram",
        "min_gram": 2,
        "max_gram": 20
      }
    },
    "tokenizer": {
      "path_tokenizer": {
        "type": "path_hierarchy",
        "delimiter": "/"
      }
    }
  }
}
```

```java
@Document(indexName = "products")
@Setting(settingPath = "/elasticsearch/settings/custom-analyzers.json")
public class Product {

    @Field(type = FieldType.Text,
           analyzer = "autocomplete",
           searchAnalyzer = "autocomplete_search")
    private String name;

    @Field(type = FieldType.Text, analyzer = "html_strip")
    private String htmlContent;

    @Field(type = FieldType.Text, analyzer = "path_analyzer")
    private String categoryPath;
}
```

## Index Settings

### Programmatic Settings
```java
@Document(indexName = "products")
@Setting(
    shards = 3,
    replicas = 1,
    refreshInterval = "1s",
    indexStoreType = "fs"
)
public class Product {
    // ...
}
```

### JSON Settings
```java
@Document(indexName = "products")
@Setting(settingPath = "/elasticsearch/settings/products.json")
@Mapping(mappingPath = "/elasticsearch/mappings/products.json")
public class Product {
    // ...
}
```

```json
// resources/elasticsearch/settings/products.json
{
  "index": {
    "number_of_shards": 3,
    "number_of_replicas": 1,
    "refresh_interval": "1s",
    "max_result_window": 50000,
    "analysis": {
      "analyzer": {
        "my_analyzer": {
          "type": "custom",
          "tokenizer": "standard",
          "filter": ["lowercase", "asciifolding"]
        }
      }
    }
  }
}
```

```json
// resources/elasticsearch/mappings/products.json
{
  "properties": {
    "name": {
      "type": "text",
      "analyzer": "my_analyzer",
      "fields": {
        "keyword": {
          "type": "keyword"
        }
      }
    },
    "description": {
      "type": "text",
      "analyzer": "english"
    },
    "price": {
      "type": "scaled_float",
      "scaling_factor": 100
    },
    "createdAt": {
      "type": "date",
      "format": "strict_date_optional_time||epoch_millis"
    }
  }
}
```

## Special Mappings

### Geo Point
```java
@Document(indexName = "stores")
public class Store {

    @Id
    private String id;

    @GeoPointField
    private GeoPoint location;

    // Alternative: use lat/lon
    @Field(type = FieldType.Double)
    private Double lat;

    @Field(type = FieldType.Double)
    private Double lon;
}

// GeoPoint class
public class GeoPoint {
    private double lat;
    private double lon;

    public GeoPoint(double lat, double lon) {
        this.lat = lat;
        this.lon = lon;
    }
}
```

### Completion Field (Suggestions)
```java
@Document(indexName = "products")
@Setting(settingPath = "/elasticsearch/settings/completion.json")
public class Product {

    @Id
    private String id;

    @CompletionField(
        maxInputLength = 100,
        preserveSeparators = true,
        preservePositionIncrements = true
    )
    private Completion suggest;

    public void initSuggestion(String name, String... contexts) {
        Completion.ContextQuery[] contextQueries = Arrays.stream(contexts)
            .map(Completion.ContextQuery::new)
            .toArray(Completion.ContextQuery[]::new);

        this.suggest = new Completion(new String[]{name});
    }
}
```

### Join Field (Parent-Child)
```java
@Document(indexName = "qa")
public class Question {

    @Id
    private String id;

    @Field(type = FieldType.Text)
    private String title;

    @JoinTypeRelations(
        relations = {
            @JoinTypeRelation(parent = "question", children = {"answer"})
        }
    )
    private JoinField<String> relation;

    public Question() {
        this.relation = new JoinField<>("question");
    }
}

@Document(indexName = "qa")
public class Answer {

    @Id
    private String id;

    @Field(type = FieldType.Text)
    private String content;

    @JoinTypeRelations(
        relations = {
            @JoinTypeRelation(parent = "question", children = {"answer"})
        }
    )
    private JoinField<String> relation;

    public Answer(String parentId) {
        this.relation = new JoinField<>("answer", parentId);
    }
}
```

### Dense Vector (ML Embeddings)
```java
@Document(indexName = "documents")
public class Document {

    @Id
    private String id;

    @Field(type = FieldType.Text)
    private String content;

    @Field(type = FieldType.Dense_Vector, dims = 768)
    private float[] embedding;
}

// Vector similarity search
public List<Document> similaritySearch(float[] queryVector) {
    NativeQuery query = NativeQuery.builder()
        .withQuery(ScriptScoreQuery.of(s -> s
            .query(MatchAllQuery.of(m -> m)._toQuery())
            .script(Script.of(sc -> sc
                .inline(InlineScript.of(i -> i
                    .source("cosineSimilarity(params.query_vector, 'embedding') + 1.0")
                    .params(Map.of("query_vector", JsonData.of(queryVector)))))))
        )._toQuery())
        .build();

    return operations.search(query, Document.class)
        .stream()
        .map(SearchHit::getContent)
        .collect(Collectors.toList());
}
```

## Dynamic Mapping

```java
@Document(indexName = "logs")
@DynamicMapping(DynamicMappingValue.Strict)
public class Log {
    // Only mapped fields allowed
}

@Document(indexName = "events")
@DynamicMapping(DynamicMappingValue.True)  // Default
public class Event {
    // New fields auto-mapped
}

@Document(indexName = "metrics")
@DynamicMapping(DynamicMappingValue.False)
public class Metric {
    // New fields stored but not indexed
}

// Dynamic templates via JSON mapping
```

## Index Operations

```java
@Service
@RequiredArgsConstructor
public class MappingService {

    private final ElasticsearchOperations operations;

    public void createIndexWithMapping(Class<?> entityClass) {
        IndexOperations indexOps = operations.indexOps(entityClass);

        if (!indexOps.exists()) {
            indexOps.create();
            indexOps.putMapping(indexOps.createMapping());
        }
    }

    public void updateMapping(Class<?> entityClass) {
        IndexOperations indexOps = operations.indexOps(entityClass);
        indexOps.putMapping(indexOps.createMapping());
    }

    public Map<String, Object> getMapping(Class<?> entityClass) {
        return operations.indexOps(entityClass).getMapping();
    }

    public void reindex(Class<?> sourceClass, String targetIndex) {
        // Create new index
        IndexOperations indexOps = operations.indexOps(sourceClass);
        indexOps.create();
        indexOps.putMapping();

        // Use Elasticsearch reindex API
        // ...
    }
}
```

## Best Practices

| Do | Don't |
|----|-------|
| Define explicit mappings | Rely on dynamic mapping |
| Use keyword for exact match | Use text for aggregations |
| Use multi-fields for flexibility | Create duplicate fields |
| Choose appropriate analyzers | Use standard for everything |
| Set doc_values: false for non-aggregating | Keep all doc_values |
| Use nested for object arrays | Use object if order matters |

## Production Checklist

- [ ] All fields explicitly mapped
- [ ] Analyzers appropriate for data
- [ ] Multi-fields configured
- [ ] Shard count planned for growth
- [ ] Dynamic mapping disabled or strict
- [ ] Index templates defined
- [ ] Mapping migration strategy planned
