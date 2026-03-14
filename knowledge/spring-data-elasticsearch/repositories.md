# Spring Data Elasticsearch - Repositories

## Overview

Spring Data Elasticsearch repositories provide a powerful abstraction for Elasticsearch operations, supporting derived queries, custom queries, and pagination.

## Enable Repositories

```java
@Configuration
@EnableElasticsearchRepositories(basePackages = "com.example.repository")
public class ElasticsearchRepositoryConfig extends ElasticsearchConfiguration {

    @Override
    public ClientConfiguration clientConfiguration() {
        return ClientConfiguration.builder()
            .connectedTo("localhost:9200")
            .build();
    }
}
```

## Basic Repository

```java
public interface ProductRepository extends ElasticsearchRepository<Product, String> {

    // Derived queries
    List<Product> findByCategory(String category);

    List<Product> findByCategoryAndActive(String category, Boolean active);

    List<Product> findByPriceBetween(BigDecimal min, BigDecimal max);

    List<Product> findByNameContaining(String name);

    List<Product> findByActiveTrue();

    List<Product> findByCategoryOrderByPriceDesc(String category);

    // Count queries
    long countByCategory(String category);

    // Exists queries
    boolean existsByName(String name);

    // Delete queries
    void deleteByCategory(String category);
}
```

## Derived Query Keywords

```java
public interface ProductRepository extends ElasticsearchRepository<Product, String> {

    // Equality
    List<Product> findByName(String name);
    List<Product> findByNameIs(String name);
    List<Product> findByNameEquals(String name);

    // Not equal
    List<Product> findByNameNot(String name);
    List<Product> findByNameIsNot(String name);

    // Null checks
    List<Product> findByDescriptionIsNull();
    List<Product> findByDescriptionIsNotNull();

    // Boolean
    List<Product> findByActiveTrue();
    List<Product> findByActiveFalse();

    // Range
    List<Product> findByPriceLessThan(BigDecimal price);
    List<Product> findByPriceLessThanEqual(BigDecimal price);
    List<Product> findByPriceGreaterThan(BigDecimal price);
    List<Product> findByPriceGreaterThanEqual(BigDecimal price);
    List<Product> findByPriceBetween(BigDecimal min, BigDecimal max);

    // String matching
    List<Product> findByNameContaining(String text);
    List<Product> findByNameStartingWith(String prefix);
    List<Product> findByNameEndingWith(String suffix);
    List<Product> findByNameLike(String pattern);

    // In clause
    List<Product> findByCategoryIn(Collection<String> categories);
    List<Product> findByCategoryNotIn(Collection<String> categories);

    // Ordering
    List<Product> findByCategoryOrderByPriceAsc(String category);
    List<Product> findByCategoryOrderByPriceDesc(String category);
    List<Product> findByCategoryOrderByNameAscPriceDesc(String category);

    // Limiting
    List<Product> findTop10ByCategory(String category);
    List<Product> findFirst5ByCategoryOrderByPriceDesc(String category);

    // Combined conditions
    List<Product> findByNameAndCategory(String name, String category);
    List<Product> findByNameOrCategory(String name, String category);
    List<Product> findByCategoryAndPriceGreaterThan(String category, BigDecimal price);
}
```

## Pagination and Sorting

```java
public interface ProductRepository extends ElasticsearchRepository<Product, String> {

    // Pageable
    Page<Product> findByCategory(String category, Pageable pageable);

    // Slice (doesn't count total)
    Slice<Product> findSliceByCategory(String category, Pageable pageable);

    // Sort
    List<Product> findByCategory(String category, Sort sort);
}

@Service
@RequiredArgsConstructor
public class ProductService {

    private final ProductRepository repository;

    public Page<Product> findProducts(String category, int page, int size) {
        Pageable pageable = PageRequest.of(page, size,
            Sort.by(Sort.Direction.DESC, "createdAt"));

        return repository.findByCategory(category, pageable);
    }

    public List<Product> findSorted(String category) {
        Sort sort = Sort.by(
            Sort.Order.asc("category"),
            Sort.Order.desc("price")
        );

        return repository.findByCategory(category, sort);
    }
}
```

## Custom Query Methods

### @Query Annotation
```java
public interface ProductRepository extends ElasticsearchRepository<Product, String> {

    @Query("{\"match\": {\"name\": \"?0\"}}")
    List<Product> findByNameMatch(String name);

    @Query("{\"bool\": {\"must\": [{\"match\": {\"name\": \"?0\"}}, {\"term\": {\"category\": \"?1\"}}]}}")
    List<Product> findByNameAndCategoryQuery(String name, String category);

    @Query("""
        {
          "bool": {
            "must": [
              {"range": {"price": {"gte": ?0, "lte": ?1}}}
            ],
            "filter": [
              {"term": {"active": true}}
            ]
          }
        }
        """)
    List<Product> findByPriceRangeActive(BigDecimal minPrice, BigDecimal maxPrice);

    @Query("{\"match_phrase\": {\"description\": \"?0\"}}")
    List<Product> findByDescriptionPhrase(String phrase);

    @Query("{\"wildcard\": {\"name\": \"?0\"}}")
    List<Product> findByNameWildcard(String pattern);

    @Query("{\"fuzzy\": {\"name\": {\"value\": \"?0\", \"fuzziness\": \"AUTO\"}}}")
    List<Product> findByNameFuzzy(String name);
}
```

### Highlight Results
```java
public interface ProductRepository extends ElasticsearchRepository<Product, String> {

    @Highlight(fields = {
        @HighlightField(name = "name"),
        @HighlightField(name = "description")
    })
    @Query("{\"multi_match\": {\"query\": \"?0\", \"fields\": [\"name\", \"description\"]}}")
    SearchHits<Product> findByTextHighlighted(String text);
}

@Service
public class SearchService {

    @Autowired
    private ProductRepository repository;

    public List<HighlightedProduct> searchWithHighlights(String text) {
        SearchHits<Product> hits = repository.findByTextHighlighted(text);

        return hits.getSearchHits().stream()
            .map(hit -> {
                Product product = hit.getContent();
                Map<String, List<String>> highlights = hit.getHighlightFields();
                return new HighlightedProduct(product, highlights);
            })
            .collect(Collectors.toList());
    }
}
```

## SearchHits Results

```java
public interface ProductRepository extends ElasticsearchRepository<Product, String> {

    SearchHits<Product> findSearchHitsByCategory(String category);

    SearchPage<Product> findSearchPageByCategory(String category, Pageable pageable);
}

@Service
public class SearchHitsService {

    @Autowired
    private ProductRepository repository;

    public SearchResult searchByCategory(String category, Pageable pageable) {
        SearchPage<Product> searchPage = repository
            .findSearchPageByCategory(category, pageable);

        List<ProductWithScore> products = searchPage.getSearchHits().stream()
            .map(hit -> new ProductWithScore(
                hit.getContent(),
                hit.getScore(),
                hit.getSortValues()
            ))
            .collect(Collectors.toList());

        return new SearchResult(
            products,
            searchPage.getTotalElements(),
            searchPage.getTotalPages(),
            searchPage.getNumber()
        );
    }
}
```

## Custom Repository Implementation

```java
public interface ProductRepositoryCustom {
    List<Product> searchProducts(ProductSearchCriteria criteria);
    SearchPage<Product> advancedSearch(String query, Pageable pageable);
}

@Repository
@RequiredArgsConstructor
public class ProductRepositoryImpl implements ProductRepositoryCustom {

    private final ElasticsearchOperations operations;

    @Override
    public List<Product> searchProducts(ProductSearchCriteria criteria) {
        BoolQuery.Builder boolQuery = new BoolQuery.Builder();

        if (criteria.getName() != null) {
            boolQuery.must(MatchQuery.of(m -> m
                .field("name")
                .query(criteria.getName())
            )._toQuery());
        }

        if (criteria.getCategory() != null) {
            boolQuery.filter(TermQuery.of(t -> t
                .field("category")
                .value(criteria.getCategory())
            )._toQuery());
        }

        if (criteria.getMinPrice() != null || criteria.getMaxPrice() != null) {
            RangeQuery.Builder rangeQuery = new RangeQuery.Builder().field("price");
            if (criteria.getMinPrice() != null) {
                rangeQuery.gte(JsonData.of(criteria.getMinPrice()));
            }
            if (criteria.getMaxPrice() != null) {
                rangeQuery.lte(JsonData.of(criteria.getMaxPrice()));
            }
            boolQuery.filter(rangeQuery.build()._toQuery());
        }

        NativeQuery query = NativeQuery.builder()
            .withQuery(boolQuery.build()._toQuery())
            .withSort(Sort.by(Sort.Direction.DESC, "createdAt"))
            .build();

        return operations.search(query, Product.class)
            .stream()
            .map(SearchHit::getContent)
            .collect(Collectors.toList());
    }

    @Override
    public SearchPage<Product> advancedSearch(String queryText, Pageable pageable) {
        NativeQuery query = NativeQuery.builder()
            .withQuery(MultiMatchQuery.of(m -> m
                .query(queryText)
                .fields("name^2", "description", "category")
                .type(TextQueryType.BestFields)
                .fuzziness("AUTO")
            )._toQuery())
            .withPageable(pageable)
            .withHighlightQuery(new HighlightQuery(
                Highlight.of(h -> h
                    .fields("name", f -> f)
                    .fields("description", f -> f)
                    .preTags("<em>")
                    .postTags("</em>")
                ),
                Product.class
            ))
            .build();

        SearchHits<Product> searchHits = operations.search(query, Product.class);
        return SearchHitSupport.searchPageFor(searchHits, pageable);
    }
}

public interface ProductRepository extends
        ElasticsearchRepository<Product, String>,
        ProductRepositoryCustom {
    // Standard methods
}
```

## Aggregation Support

```java
@Repository
public class ProductAggregationRepository {

    @Autowired
    private ElasticsearchOperations operations;

    public Map<String, Long> getCategoryCount() {
        NativeQuery query = NativeQuery.builder()
            .withAggregation("categories",
                Aggregation.of(a -> a.terms(t -> t.field("category"))))
            .withMaxResults(0)
            .build();

        SearchHits<Product> searchHits = operations.search(query, Product.class);

        ElasticsearchAggregations aggregations =
            (ElasticsearchAggregations) searchHits.getAggregations();

        return aggregations.aggregationsAsMap().get("categories")
            .aggregation().getAggregate().sterms().buckets().array()
            .stream()
            .collect(Collectors.toMap(
                StringTermsBucket::key,
                StringTermsBucket::docCount
            ));
    }

    public PriceStats getPriceStatsByCategory(String category) {
        NativeQuery query = NativeQuery.builder()
            .withQuery(TermQuery.of(t -> t.field("category").value(category))._toQuery())
            .withAggregation("price_stats",
                Aggregation.of(a -> a.stats(s -> s.field("price"))))
            .withMaxResults(0)
            .build();

        SearchHits<Product> searchHits = operations.search(query, Product.class);

        ElasticsearchAggregations aggregations =
            (ElasticsearchAggregations) searchHits.getAggregations();

        StatsAggregate stats = aggregations.aggregationsAsMap()
            .get("price_stats").aggregation().getAggregate().stats();

        return new PriceStats(
            stats.min(),
            stats.max(),
            stats.avg(),
            stats.count()
        );
    }
}
```

## Best Practices

| Do | Don't |
|----|-------|
| Use appropriate query types | Match query for exact fields |
| Implement pagination | Return unbounded results |
| Use SearchHits for scores | Ignore relevance scores |
| Define custom repository for complex queries | Overcomplicate derived queries |
| Use filters for non-scoring | Match queries for filtering |
| Optimize aggregations | Run heavy aggregations frequently |

## Production Checklist

- [ ] Pagination implemented
- [ ] Custom queries optimized
- [ ] Aggregations tested
- [ ] Highlighting configured
- [ ] Sort options validated
- [ ] Query performance monitored
