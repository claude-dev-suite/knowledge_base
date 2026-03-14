# Spring Data Neo4j - Repositories

## Overview

Spring Data Neo4j repositories provide a familiar abstraction for graph database operations, supporting derived queries, custom Cypher queries, and pagination.

## Enable Repositories

```java
@Configuration
@EnableNeo4jRepositories(basePackages = "com.example.repository")
public class Neo4jRepositoryConfig {
    // Configuration
}
```

## Basic Repository

```java
public interface PersonRepository extends Neo4jRepository<Person, Long> {

    // Derived queries
    List<Person> findByName(String name);

    Optional<Person> findByEmail(String email);

    List<Person> findByNameContaining(String name);

    List<Person> findByBirthYearGreaterThan(Integer year);

    List<Person> findByBirthYearBetween(Integer start, Integer end);

    boolean existsByEmail(String email);

    long countByBirthYearGreaterThan(Integer year);

    void deleteByEmail(String email);
}
```

## Derived Query Keywords

```java
public interface PersonRepository extends Neo4jRepository<Person, Long> {

    // Equality
    List<Person> findByName(String name);
    List<Person> findByNameIs(String name);
    List<Person> findByNameEquals(String name);

    // Not equal
    List<Person> findByNameNot(String name);
    List<Person> findByNameIsNot(String name);

    // Null checks
    List<Person> findByEmailIsNull();
    List<Person> findByEmailIsNotNull();

    // Like patterns
    List<Person> findByNameLike(String pattern);
    List<Person> findByNameNotLike(String pattern);
    List<Person> findByNameStartingWith(String prefix);
    List<Person> findByNameEndingWith(String suffix);
    List<Person> findByNameContaining(String text);

    // Comparison
    List<Person> findByBirthYearLessThan(Integer year);
    List<Person> findByBirthYearLessThanEqual(Integer year);
    List<Person> findByBirthYearGreaterThan(Integer year);
    List<Person> findByBirthYearGreaterThanEqual(Integer year);
    List<Person> findByBirthYearBetween(Integer start, Integer end);

    // Boolean
    List<Person> findByActiveTrue();
    List<Person> findByActiveFalse();

    // In clause
    List<Person> findByNameIn(Collection<String> names);
    List<Person> findByNameNotIn(Collection<String> names);

    // Ordering
    List<Person> findByNameOrderByBirthYearAsc(String name);
    List<Person> findByNameOrderByBirthYearDesc(String name);

    // Limiting
    List<Person> findTop10ByBirthYearGreaterThan(Integer year);
    Optional<Person> findFirstByNameOrderByBirthYearDesc(String name);

    // Combined
    List<Person> findByNameAndBirthYear(String name, Integer year);
    List<Person> findByNameOrEmail(String name, String email);
}
```

## Custom Cypher Queries

```java
public interface PersonRepository extends Neo4jRepository<Person, Long> {

    @Query("MATCH (p:Person) WHERE p.name = $name RETURN p")
    List<Person> findByNameCustom(String name);

    @Query("MATCH (p:Person) WHERE p.name CONTAINS $text RETURN p ORDER BY p.name")
    List<Person> searchByName(String text);

    @Query("""
        MATCH (p:Person)-[:ACTED_IN]->(m:Movie)
        WHERE m.title = $movieTitle
        RETURN p
        """)
    List<Person> findActorsByMovie(String movieTitle);

    @Query("""
        MATCH (p:Person)-[:DIRECTED]->(m:Movie)
        WHERE p.name = $directorName
        RETURN m
        """)
    List<Movie> findMoviesByDirector(String directorName);

    @Query("""
        MATCH (p1:Person)-[:ACTED_IN]->(m:Movie)<-[:ACTED_IN]-(p2:Person)
        WHERE p1.name = $personName AND p1 <> p2
        RETURN DISTINCT p2
        """)
    List<Person> findCoActors(String personName);

    @Query("MATCH (p:Person) WHERE p.born > $year RETURN count(p)")
    long countBornAfter(Integer year);

    @Query("MATCH (p:Person) RETURN avg(p.born)")
    Double getAverageBirthYear();
}
```

## Pagination and Sorting

```java
public interface MovieRepository extends Neo4jRepository<Movie, Long> {

    Page<Movie> findByReleasedGreaterThan(Integer year, Pageable pageable);

    Slice<Movie> findSliceByReleasedBetween(Integer start, Integer end, Pageable pageable);

    List<Movie> findByReleasedGreaterThan(Integer year, Sort sort);

    @Query(value = """
        MATCH (m:Movie)
        WHERE m.released > $year
        RETURN m
        ORDER BY m.released DESC
        SKIP $skip
        LIMIT $limit
        """,
        countQuery = """
        MATCH (m:Movie)
        WHERE m.released > $year
        RETURN count(m)
        """)
    Page<Movie> findRecentMovies(Integer year, Pageable pageable);
}

@Service
@RequiredArgsConstructor
public class MovieService {

    private final MovieRepository movieRepository;

    public Page<Movie> findMovies(int page, int size) {
        Pageable pageable = PageRequest.of(page, size,
            Sort.by(Sort.Direction.DESC, "released"));

        return movieRepository.findByReleasedGreaterThan(2000, pageable);
    }
}
```

## Projections

```java
// Interface projection
public interface PersonSummary {
    String getName();
    Integer getBirthYear();
}

// DTO projection
public record PersonDto(String name, Integer birthYear) {}

public interface PersonRepository extends Neo4jRepository<Person, Long> {

    // Interface projection
    List<PersonSummary> findSummaryByBirthYearGreaterThan(Integer year);

    // DTO projection via Cypher
    @Query("""
        MATCH (p:Person)
        WHERE p.born > $year
        RETURN p.name as name, p.born as birthYear
        """)
    List<PersonDto> findDtoByBirthYearGreaterThan(Integer year);

    // Nested projection
    List<MovieWithActors> findMovieWithActorsByTitle(String title);
}

public interface MovieWithActors {
    String getTitle();
    Integer getReleased();
    List<PersonSummary> getActors();
}
```

## Modifying Queries

```java
public interface PersonRepository extends Neo4jRepository<Person, Long> {

    @Query("MATCH (p:Person) WHERE p.name = $oldName SET p.name = $newName")
    void updateName(String oldName, String newName);

    @Query("""
        MATCH (p:Person)
        WHERE p.born < $year
        SET p.veteran = true
        RETURN count(p)
        """)
    long markVeterans(Integer year);

    @Query("MATCH (p:Person) WHERE p.name = $name DETACH DELETE p")
    void deleteByNameWithRelationships(String name);

    @Query("""
        MATCH (p:Person), (m:Movie)
        WHERE p.name = $personName AND m.title = $movieTitle
        CREATE (p)-[:ACTED_IN {role: $role}]->(m)
        """)
    void createActedInRelationship(String personName, String movieTitle, String role);
}
```

## Relationship Queries

```java
public interface PersonRepository extends Neo4jRepository<Person, Long> {

    @Query("""
        MATCH (p:Person)-[r:ACTED_IN]->(m:Movie)
        WHERE p.name = $name
        RETURN p, collect(r), collect(m)
        """)
    Person findWithMovies(String name);

    @Query("""
        MATCH path = shortestPath((p1:Person)-[*]-(p2:Person))
        WHERE p1.name = $name1 AND p2.name = $name2
        RETURN path
        """)
    List<Object> findShortestPath(String name1, String name2);

    @Query("""
        MATCH (p:Person)-[:ACTED_IN]->(m:Movie)
        RETURN p.name as actor, count(m) as movieCount
        ORDER BY movieCount DESC
        LIMIT 10
        """)
    List<ActorMovieCount> findTopActors();

    @Query("""
        MATCH (p:Person)-[r:ACTED_IN]->(m:Movie)
        WHERE m.title = $title
        RETURN p.name as name, r.role as role
        """)
    List<CastMember> findCastByMovie(String title);
}

public interface ActorMovieCount {
    String getActor();
    Long getMovieCount();
}

public interface CastMember {
    String getName();
    String getRole();
}
```

## Custom Repository Implementation

```java
public interface PersonRepositoryCustom {
    List<Person> findByComplexCriteria(PersonSearchCriteria criteria);
    void bulkUpdateStatus(List<Long> ids, String status);
}

public class PersonRepositoryImpl implements PersonRepositoryCustom {

    private final Neo4jClient client;
    private final Neo4jTemplate template;

    public PersonRepositoryImpl(Neo4jClient client, Neo4jTemplate template) {
        this.client = client;
        this.template = template;
    }

    @Override
    public List<Person> findByComplexCriteria(PersonSearchCriteria criteria) {
        StringBuilder cypher = new StringBuilder("MATCH (p:Person) WHERE 1=1");
        Map<String, Object> params = new HashMap<>();

        if (criteria.getName() != null) {
            cypher.append(" AND p.name CONTAINS $name");
            params.put("name", criteria.getName());
        }

        if (criteria.getMinBirthYear() != null) {
            cypher.append(" AND p.born >= $minYear");
            params.put("minYear", criteria.getMinBirthYear());
        }

        if (criteria.getMaxBirthYear() != null) {
            cypher.append(" AND p.born <= $maxYear");
            params.put("maxYear", criteria.getMaxBirthYear());
        }

        cypher.append(" RETURN p");

        return client.query(cypher.toString())
            .bindAll(params)
            .fetchAs(Person.class)
            .all();
    }

    @Override
    @Transactional
    public void bulkUpdateStatus(List<Long> ids, String status) {
        client.query("""
            MATCH (p:Person)
            WHERE id(p) IN $ids
            SET p.status = $status
            """)
            .bind(ids).to("ids")
            .bind(status).to("status")
            .run();
    }
}

public interface PersonRepository extends
        Neo4jRepository<Person, Long>,
        PersonRepositoryCustom {
    // Standard methods
}
```

## Query by Example

```java
@Service
@RequiredArgsConstructor
public class PersonSearchService {

    private final PersonRepository repository;

    public List<Person> searchByExample(Person probe) {
        ExampleMatcher matcher = ExampleMatcher.matching()
            .withIgnoreCase()
            .withStringMatcher(ExampleMatcher.StringMatcher.CONTAINING)
            .withIgnoreNullValues();

        Example<Person> example = Example.of(probe, matcher);
        return repository.findAll(example);
    }

    public List<Person> searchExact(Person probe) {
        Example<Person> example = Example.of(probe);
        return repository.findAll(example);
    }
}
```

## Best Practices

| Do | Don't |
|----|-------|
| Use @Query for complex queries | Overcomplicate derived queries |
| Include count queries for pagination | Skip count with large datasets |
| Use projections for partial data | Fetch entire nodes always |
| Implement custom repository for complex logic | Put everything in service |
| Use proper parameter binding | String concatenation in queries |
| Test queries with Neo4j Browser | Deploy untested queries |

## Production Checklist

- [ ] Derived queries validated
- [ ] Custom queries tested
- [ ] Pagination implemented correctly
- [ ] Projections used appropriately
- [ ] Modifying queries transactional
- [ ] Relationship queries optimized
