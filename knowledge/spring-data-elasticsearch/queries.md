# Spring Data Elasticsearch - Queries

## Overview

Elasticsearch provides powerful query capabilities including full-text search, structured queries, aggregations, and suggestions. This guide covers advanced query patterns.

## Query Types

### Match Query
```java
// Basic match
Query matchQuery = MatchQuery.of(m -> m
    .field("name")
    .query("laptop")
)._toQuery();

// With options
Query matchWithOptions = MatchQuery.of(m -> m
    .field("name")
    .query("laptops computers")
    .operator(Operator.And)        // All terms must match
    .fuzziness("AUTO")             // Fuzzy matching
    .prefixLength(2)               // Min prefix before fuzzy
    .maxExpansions(50)             // Max fuzzy variations
    .minimumShouldMatch("75%")     // Min matching terms
)._toQuery();
```

### Multi-Match Query
```java
// Search multiple fields
Query multiMatch = MultiMatchQuery.of(m -> m
    .query("gaming laptop")
    .fields("name^3", "description^1", "category^2")  // Field boosting
    .type(TextQueryType.BestFields)
    .tieBreaker(0.3)
)._toQuery();

// Different types
// BestFields - max score from any field
// MostFields - sum of all field scores
// CrossFields - treats fields as one
// Phrase - phrase matching
// PhrasePrefix - phrase with prefix on last term
```

### Term Queries
```java
// Exact match (no analysis)
Query termQuery = TermQuery.of(t -> t
    .field("status")
    .value("active")
)._toQuery();

// Multiple terms
Query termsQuery = TermsQuery.of(t -> t
    .field("category")
    .terms(TermsQueryField.of(f -> f
        .value(List.of(
            FieldValue.of("electronics"),
            FieldValue.of("computers")
        ))))
)._toQuery();

// Terms set (min match)
Query termsSetQuery = TermsSetQuery.of(t -> t
    .field("tags")
    .terms(List.of("premium", "sale", "new"))
    .minimumShouldMatchScript(Script.of(s -> s
        .inline(InlineScript.of(i -> i.source("2")))))
)._toQuery();
```

### Range Queries
```java
// Numeric range
Query numericRange = RangeQuery.of(r -> r
    .field("price")
    .gte(JsonData.of(100))
    .lte(JsonData.of(500))
)._toQuery();

// Date range
Query dateRange = RangeQuery.of(r -> r
    .field("createdAt")
    .gte(JsonData.of("2024-01-01"))
    .lte(JsonData.of("now"))
    .format("yyyy-MM-dd||epoch_millis")
)._toQuery();

// Relative dates
Query recentRange = RangeQuery.of(r -> r
    .field("createdAt")
    .gte(JsonData.of("now-7d/d"))  // Last 7 days
)._toQuery();
```

### Wildcard and Prefix
```java
// Wildcard
Query wildcardQuery = WildcardQuery.of(w -> w
    .field("sku")
    .value("PROD-*-2024")
    .caseInsensitive(true)
)._toQuery();

// Prefix
Query prefixQuery = PrefixQuery.of(p -> p
    .field("name")
    .value("lap")
)._toQuery();

// Regexp
Query regexpQuery = RegexpQuery.of(r -> r
    .field("productCode")
    .value("[A-Z]{2}-[0-9]{4}")
)._toQuery();
```

### Fuzzy Query
```java
Query fuzzyQuery = FuzzyQuery.of(f -> f
    .field("name")
    .value("laptpo")  // Typo in "laptop"
    .fuzziness("AUTO")
    .prefixLength(2)
    .maxExpansions(50)
    .transpositions(true)
)._toQuery();
```

## Bool Query Combinations

```java
@Service
public class AdvancedSearchService {

    public List<Product> advancedSearch(SearchCriteria criteria) {
        BoolQuery.Builder boolQuery = new BoolQuery.Builder();

        // Must: Required matching with scoring
        if (criteria.getSearchText() != null) {
            boolQuery.must(MultiMatchQuery.of(m -> m
                .query(criteria.getSearchText())
                .fields("name^3", "description")
                .type(TextQueryType.BestFields)
            )._toQuery());
        }

        // Should: Optional matching, boosts score
        if (criteria.getPreferredBrands() != null) {
            for (String brand : criteria.getPreferredBrands()) {
                boolQuery.should(TermQuery.of(t -> t
                    .field("brand")
                    .value(brand)
                    .boost(2.0f)
                )._toQuery());
            }
        }

        // Filter: Required but no scoring (faster)
        if (criteria.getCategory() != null) {
            boolQuery.filter(TermQuery.of(t -> t
                .field("category")
                .value(criteria.getCategory())
            )._toQuery());
        }

        if (criteria.getMinPrice() != null || criteria.getMaxPrice() != null) {
            RangeQuery.Builder range = new RangeQuery.Builder().field("price");
            if (criteria.getMinPrice() != null) {
                range.gte(JsonData.of(criteria.getMinPrice()));
            }
            if (criteria.getMaxPrice() != null) {
                range.lte(JsonData.of(criteria.getMaxPrice()));
            }
            boolQuery.filter(range.build()._toQuery());
        }

        if (criteria.isInStockOnly()) {
            boolQuery.filter(RangeQuery.of(r -> r
                .field("stock")
                .gt(JsonData.of(0))
            )._toQuery());
        }

        // Must not: Exclusion
        if (criteria.getExcludedCategories() != null) {
            boolQuery.mustNot(TermsQuery.of(t -> t
                .field("category")
                .terms(TermsQueryField.of(f -> f
                    .value(criteria.getExcludedCategories().stream()
                        .map(FieldValue::of)
                        .collect(Collectors.toList()))))
            )._toQuery());
        }

        NativeQuery query = NativeQuery.builder()
            .withQuery(boolQuery.build()._toQuery())
            .withPageable(PageRequest.of(0, 20))
            .build();

        return operations.search(query, Product.class)
            .stream()
            .map(SearchHit::getContent)
            .collect(Collectors.toList());
    }
}
```

## Nested Queries

```java
// Entity with nested objects
@Document(indexName = "products")
public class Product {
    @Id
    private String id;
    private String name;

    @Field(type = FieldType.Nested)
    private List<Specification> specifications;
}

public class Specification {
    private String name;
    private String value;
}

// Query nested objects
public List<Product> findBySpecification(String specName, String specValue) {
    NativeQuery query = NativeQuery.builder()
        .withQuery(NestedQuery.of(n -> n
            .path("specifications")
            .query(BoolQuery.of(b -> b
                .must(TermQuery.of(t -> t
                    .field("specifications.name")
                    .value(specName)
                )._toQuery())
                .must(TermQuery.of(t -> t
                    .field("specifications.value")
                    .value(specValue)
                )._toQuery())
            )._toQuery())
            .scoreMode(ChildScoreMode.Avg)
        )._toQuery())
        .build();

    return operations.search(query, Product.class)
        .stream()
        .map(SearchHit::getContent)
        .collect(Collectors.toList());
}
```

## Function Score Queries

```java
// Boost by popularity and recency
public List<Product> searchWithBoosts(String text) {
    NativeQuery query = NativeQuery.builder()
        .withQuery(FunctionScoreQuery.of(fs -> fs
            .query(MatchQuery.of(m -> m
                .field("name")
                .query(text)
            )._toQuery())
            .functions(List.of(
                // Boost by sales
                FunctionScore.of(f -> f
                    .fieldValueFactor(fvf -> fvf
                        .field("salesCount")
                        .factor(1.2)
                        .modifier(FieldValueFactorModifier.Log1p)
                        .missing(1.0))
                    .weight(2.0)),
                // Boost recent products
                FunctionScore.of(f -> f
                    .exp(DecayFunction.of(d -> d
                        .field("createdAt")
                        .placement(DecayPlacement.of(p -> p
                            .origin(JsonData.of("now"))
                            .scale(JsonData.of("30d"))
                            .decay(0.5)))))
                    .weight(1.5)),
                // Random score for variety
                FunctionScore.of(f -> f
                    .randomScore(rs -> rs.seed("user123").field("_seq_no"))
                    .weight(0.5))
            ))
            .scoreMode(FunctionScoreMode.Sum)
            .boostMode(FunctionBoostMode.Multiply)
        )._toQuery())
        .build();

    return operations.search(query, Product.class)
        .stream()
        .map(SearchHit::getContent)
        .collect(Collectors.toList());
}
```

## Geo Queries

```java
@Document(indexName = "stores")
public class Store {
    @Id
    private String id;
    private String name;

    @GeoPointField
    private GeoPoint location;
}

// Find stores near a point
public List<Store> findNearby(double lat, double lon, String distance) {
    NativeQuery query = NativeQuery.builder()
        .withQuery(GeoDistanceQuery.of(g -> g
            .field("location")
            .location(GeoLocation.of(l -> l.latlon(LatLonGeoLocation.of(ll -> ll
                .lat(lat)
                .lon(lon)))))
            .distance(distance)
        )._toQuery())
        .withSort(SortOptions.of(s -> s
            .geoDistance(GeoDistanceSort.of(gd -> gd
                .field("location")
                .location(List.of(GeoLocation.of(l -> l.latlon(LatLonGeoLocation.of(ll -> ll
                    .lat(lat)
                    .lon(lon))))))
                .order(SortOrder.Asc)
                .unit(DistanceUnit.Kilometers)))))
        .build();

    return operations.search(query, Store.class)
        .stream()
        .map(SearchHit::getContent)
        .collect(Collectors.toList());
}

// Find stores within a bounding box
public List<Store> findInBoundingBox(double topLat, double leftLon,
                                      double bottomLat, double rightLon) {
    NativeQuery query = NativeQuery.builder()
        .withQuery(GeoBoundingBoxQuery.of(g -> g
            .field("location")
            .boundingBox(GeoBounds.of(b -> b
                .tlbr(TopLeftBottomRightGeoBounds.of(tlbr -> tlbr
                    .topLeft(GeoLocation.of(l -> l.latlon(LatLonGeoLocation.of(ll -> ll
                        .lat(topLat).lon(leftLon)))))
                    .bottomRight(GeoLocation.of(l -> l.latlon(LatLonGeoLocation.of(ll -> ll
                        .lat(bottomLat).lon(rightLon)))))))))
        )._toQuery())
        .build();

    return operations.search(query, Store.class)
        .stream()
        .map(SearchHit::getContent)
        .collect(Collectors.toList());
}
```

## Suggestions

```java
@Document(indexName = "products")
@Setting(settingPath = "/elasticsearch/settings/autocomplete.json")
public class Product {
    @Id
    private String id;

    @CompletionField(maxInputLength = 100)
    private Completion suggest;

    // Populate completion field
    public void setSuggestFromName(String name) {
        this.suggest = new Completion(new String[]{name});
    }
}

// Get suggestions
public List<String> getSuggestions(String prefix) {
    NativeQuery query = NativeQuery.builder()
        .withSuggester(Suggester.of(s -> s
            .suggesters("product-suggest", FieldSuggester.of(fs -> fs
                .completion(CompletionSuggester.of(c -> c
                    .field("suggest")
                    .prefix(prefix)
                    .size(10)
                    .skipDuplicates(true)
                    .fuzzy(SuggestFuzziness.of(f -> f
                        .fuzziness("AUTO")))))))))
        .withMaxResults(0)
        .build();

    SearchHits<Product> hits = operations.search(query, Product.class);

    return hits.getSuggest()
        .getSuggestion("product-suggest")
        .getEntries()
        .stream()
        .flatMap(entry -> entry.getOptions().stream())
        .map(option -> option.getText())
        .collect(Collectors.toList());
}
```

## Best Practices

| Do | Don't |
|----|-------|
| Use filters for non-scoring clauses | Score everything |
| Specify field types correctly | Rely on dynamic mapping |
| Use completion suggester for autocomplete | Query on every keystroke |
| Test fuzzy settings | Use high fuzziness values |
| Limit expansions | Allow unlimited wildcards |
| Use geo queries for location data | Calculate distances manually |

## Production Checklist

- [ ] Query types chosen appropriately
- [ ] Bool queries optimized with filters
- [ ] Fuzzy settings tuned
- [ ] Geo queries use proper field types
- [ ] Suggestions configured
- [ ] Query performance tested
