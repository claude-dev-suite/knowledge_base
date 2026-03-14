# Spring Data Elasticsearch - ElasticsearchOperations

## Overview

ElasticsearchOperations provides a template-based approach for Elasticsearch operations, offering fine-grained control over queries, indexing, and search functionality.

## Configuration

```java
@Configuration
public class ElasticsearchTemplateConfig extends ElasticsearchConfiguration {

    @Override
    public ClientConfiguration clientConfiguration() {
        return ClientConfiguration.builder()
            .connectedTo("localhost:9200")
            .build();
    }
}
```

## Basic CRUD Operations

```java
@Service
@RequiredArgsConstructor
public class ProductService {

    private final ElasticsearchOperations operations;

    // Create / Update
    public Product save(Product product) {
        return operations.save(product);
    }

    public Iterable<Product> saveAll(List<Product> products) {
        return operations.save(products);
    }

    // Read
    public Product findById(String id) {
        return operations.get(id, Product.class);
    }

    public List<Product> findByIds(List<String> ids) {
        NativeQuery query = NativeQuery.builder()
            .withIds(ids)
            .build();

        return operations.multiGet(query, Product.class, IndexCoordinates.of("products"))
            .stream()
            .filter(MultiGetItem::hasItem)
            .map(item -> item.getItem())
            .collect(Collectors.toList());
    }

    // Delete
    public void deleteById(String id) {
        operations.delete(id, Product.class);
    }

    public void delete(Product product) {
        operations.delete(product);
    }

    public void deleteAll() {
        NativeQuery query = NativeQuery.builder()
            .withQuery(QueryBuilders.matchAll())
            .build();

        operations.delete(query, Product.class);
    }

    // Exists
    public boolean exists(String id) {
        return operations.exists(id, Product.class);
    }

    // Count
    public long count() {
        return operations.count(Query.findAll(), Product.class);
    }
}
```

## Search Operations

### Simple Search
```java
@Service
public class SearchService {

    @Autowired
    private ElasticsearchOperations operations;

    public List<Product> search(String text) {
        NativeQuery query = NativeQuery.builder()
            .withQuery(MultiMatchQuery.of(m -> m
                .query(text)
                .fields("name", "description")
            )._toQuery())
            .build();

        return operations.search(query, Product.class)
            .stream()
            .map(SearchHit::getContent)
            .collect(Collectors.toList());
    }

    public SearchHits<Product> searchWithHits(String text) {
        NativeQuery query = NativeQuery.builder()
            .withQuery(MatchQuery.of(m -> m
                .field("name")
                .query(text)
            )._toQuery())
            .build();

        return operations.search(query, Product.class);
    }
}
```

### Paginated Search
```java
public SearchPage<Product> searchPaginated(String text, int page, int size) {
    NativeQuery query = NativeQuery.builder()
        .withQuery(MatchQuery.of(m -> m
            .field("name")
            .query(text)
        )._toQuery())
        .withPageable(PageRequest.of(page, size))
        .build();

    SearchHits<Product> hits = operations.search(query, Product.class);
    return SearchHitSupport.searchPageFor(hits, PageRequest.of(page, size));
}
```

### Streaming Results
```java
public Stream<Product> searchStream(String category) {
    NativeQuery query = NativeQuery.builder()
        .withQuery(TermQuery.of(t -> t
            .field("category")
            .value(category)
        )._toQuery())
        .build();

    return operations.searchForStream(query, Product.class)
        .map(SearchHit::getContent);
}
```

## Query Building

### Bool Query
```java
public List<Product> complexSearch(ProductSearchCriteria criteria) {
    BoolQuery.Builder boolQuery = new BoolQuery.Builder();

    // Must clauses (AND with scoring)
    if (criteria.getName() != null) {
        boolQuery.must(MatchQuery.of(m -> m
            .field("name")
            .query(criteria.getName())
            .fuzziness("AUTO")
        )._toQuery());
    }

    // Should clauses (OR with scoring boost)
    if (criteria.getKeywords() != null) {
        for (String keyword : criteria.getKeywords()) {
            boolQuery.should(MatchQuery.of(m -> m
                .field("description")
                .query(keyword)
            )._toQuery());
        }
        boolQuery.minimumShouldMatch("1");
    }

    // Filter clauses (no scoring, faster)
    if (criteria.getCategory() != null) {
        boolQuery.filter(TermQuery.of(t -> t
            .field("category")
            .value(criteria.getCategory())
        )._toQuery());
    }

    if (criteria.isActiveOnly()) {
        boolQuery.filter(TermQuery.of(t -> t
            .field("active")
            .value(true)
        )._toQuery());
    }

    // Must not clauses (exclusion)
    if (criteria.getExcludeCategories() != null) {
        boolQuery.mustNot(TermsQuery.of(t -> t
            .field("category")
            .terms(TermsQueryField.of(f -> f
                .value(criteria.getExcludeCategories().stream()
                    .map(FieldValue::of)
                    .collect(Collectors.toList()))))
        )._toQuery());
    }

    NativeQuery query = NativeQuery.builder()
        .withQuery(boolQuery.build()._toQuery())
        .build();

    return operations.search(query, Product.class)
        .stream()
        .map(SearchHit::getContent)
        .collect(Collectors.toList());
}
```

### Range Query
```java
public List<Product> findByPriceRange(BigDecimal min, BigDecimal max) {
    NativeQuery query = NativeQuery.builder()
        .withQuery(RangeQuery.of(r -> r
            .field("price")
            .gte(JsonData.of(min))
            .lte(JsonData.of(max))
        )._toQuery())
        .build();

    return operations.search(query, Product.class)
        .stream()
        .map(SearchHit::getContent)
        .collect(Collectors.toList());
}

public List<Product> findRecentProducts(int days) {
    NativeQuery query = NativeQuery.builder()
        .withQuery(RangeQuery.of(r -> r
            .field("createdAt")
            .gte(JsonData.of("now-" + days + "d/d"))
        )._toQuery())
        .build();

    return operations.search(query, Product.class)
        .stream()
        .map(SearchHit::getContent)
        .collect(Collectors.toList());
}
```

### Full-text Queries
```java
// Match query
public List<Product> matchSearch(String text) {
    NativeQuery query = NativeQuery.builder()
        .withQuery(MatchQuery.of(m -> m
            .field("name")
            .query(text)
            .operator(Operator.And)
            .fuzziness("AUTO")
        )._toQuery())
        .build();

    return executeSearch(query);
}

// Multi-match query
public List<Product> multiMatchSearch(String text) {
    NativeQuery query = NativeQuery.builder()
        .withQuery(MultiMatchQuery.of(m -> m
            .query(text)
            .fields("name^3", "description^1", "category^2")
            .type(TextQueryType.BestFields)
            .tieBreaker(0.3)
        )._toQuery())
        .build();

    return executeSearch(query);
}

// Match phrase query
public List<Product> phraseSearch(String phrase) {
    NativeQuery query = NativeQuery.builder()
        .withQuery(MatchPhraseQuery.of(m -> m
            .field("description")
            .query(phrase)
            .slop(2)
        )._toQuery())
        .build();

    return executeSearch(query);
}

// Query string (user-facing)
public List<Product> queryStringSearch(String queryString) {
    NativeQuery query = NativeQuery.builder()
        .withQuery(QueryStringQuery.of(q -> q
            .query(queryString)
            .defaultField("name")
            .defaultOperator(Operator.And)
        )._toQuery())
        .build();

    return executeSearch(query);
}
```

## Sorting

```java
public List<Product> searchSorted(String category) {
    NativeQuery query = NativeQuery.builder()
        .withQuery(TermQuery.of(t -> t
            .field("category")
            .value(category)
        )._toQuery())
        .withSort(Sort.by(
            Sort.Order.desc("_score"),
            Sort.Order.desc("createdAt"),
            Sort.Order.asc("name")
        ))
        .build();

    return operations.search(query, Product.class)
        .stream()
        .map(SearchHit::getContent)
        .collect(Collectors.toList());
}

// Field-based sorting with missing values
public List<Product> searchWithMissingSort(String category) {
    NativeQuery query = NativeQuery.builder()
        .withQuery(TermQuery.of(t -> t
            .field("category")
            .value(category)
        )._toQuery())
        .withSort(SortOptions.of(s -> s
            .field(f -> f
                .field("price")
                .order(SortOrder.Asc)
                .missing("_last")
            )))
        .build();

    return operations.search(query, Product.class)
        .stream()
        .map(SearchHit::getContent)
        .collect(Collectors.toList());
}
```

## Highlighting

```java
public List<HighlightedProduct> searchWithHighlighting(String text) {
    NativeQuery query = NativeQuery.builder()
        .withQuery(MultiMatchQuery.of(m -> m
            .query(text)
            .fields("name", "description")
        )._toQuery())
        .withHighlightQuery(new HighlightQuery(
            Highlight.of(h -> h
                .fields("name", f -> f
                    .preTags("<strong>")
                    .postTags("</strong>")
                    .numberOfFragments(0))
                .fields("description", f -> f
                    .preTags("<em>")
                    .postTags("</em>")
                    .fragmentSize(150)
                    .numberOfFragments(3))
            ),
            Product.class
        ))
        .build();

    return operations.search(query, Product.class)
        .stream()
        .map(hit -> new HighlightedProduct(
            hit.getContent(),
            hit.getHighlightFields()
        ))
        .collect(Collectors.toList());
}
```

## Source Filtering

```java
public List<ProductSummary> searchSummary(String category) {
    NativeQuery query = NativeQuery.builder()
        .withQuery(TermQuery.of(t -> t
            .field("category")
            .value(category)
        )._toQuery())
        .withSourceFilter(new FetchSourceFilter(
            new String[]{"id", "name", "price"},  // includes
            new String[]{}  // excludes
        ))
        .build();

    return operations.search(query, Product.class)
        .stream()
        .map(hit -> new ProductSummary(
            hit.getContent().getId(),
            hit.getContent().getName(),
            hit.getContent().getPrice()
        ))
        .collect(Collectors.toList());
}
```

## Script Queries

```java
public List<Product> searchWithScript() {
    NativeQuery query = NativeQuery.builder()
        .withQuery(ScriptQuery.of(s -> s
            .script(Script.of(sc -> sc
                .inline(InlineScript.of(i -> i
                    .source("doc['price'].value > params.minPrice && doc['stock'].value > 0")
                    .params(Map.of("minPrice", JsonData.of(100)))
                ))
            ))
        )._toQuery())
        .build();

    return operations.search(query, Product.class)
        .stream()
        .map(SearchHit::getContent)
        .collect(Collectors.toList());
}
```

## Update Operations

```java
@Service
public class ProductUpdateService {

    @Autowired
    private ElasticsearchOperations operations;

    public void updatePrice(String id, BigDecimal newPrice) {
        UpdateQuery query = UpdateQuery.builder(id)
            .withDocument(Document.from(Map.of("price", newPrice)))
            .build();

        operations.update(query, IndexCoordinates.of("products"));
    }

    public void updateByScript(String id, int quantityChange) {
        UpdateQuery query = UpdateQuery.builder(id)
            .withScript("ctx._source.stock += params.change")
            .withParams(Map.of("change", quantityChange))
            .build();

        operations.update(query, IndexCoordinates.of("products"));
    }

    public void updateByQuery(String category, boolean active) {
        UpdateQuery query = UpdateQuery.builder(
                NativeQuery.builder()
                    .withQuery(TermQuery.of(t -> t
                        .field("category")
                        .value(category)
                    )._toQuery())
                    .build()
            )
            .withScript("ctx._source.active = params.active")
            .withParams(Map.of("active", active))
            .build();

        operations.updateByQuery(query, IndexCoordinates.of("products"));
    }
}
```

## Best Practices

| Do | Don't |
|----|-------|
| Use filters for non-scoring clauses | Score everything |
| Implement pagination | Return all results |
| Use source filtering | Fetch entire documents |
| Build queries programmatically | String concatenation |
| Cache frequent queries | Rebuild queries repeatedly |
| Use bulk operations | Update one by one |

## Production Checklist

- [ ] Query optimization validated
- [ ] Pagination implemented correctly
- [ ] Source filtering used
- [ ] Highlighting configured
- [ ] Sort options tested
- [ ] Error handling in place
