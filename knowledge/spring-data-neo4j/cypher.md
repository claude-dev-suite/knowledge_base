# Spring Data Neo4j - Cypher Queries

## Overview

Cypher is Neo4j's declarative query language for querying and updating graph data. Spring Data Neo4j provides multiple ways to execute Cypher queries.

## Basic Cypher Patterns

### Node Patterns
```cypher
-- Match all Person nodes
MATCH (p:Person)
RETURN p

-- Match with property filter
MATCH (p:Person {name: 'Tom Hanks'})
RETURN p

-- Match with WHERE clause
MATCH (p:Person)
WHERE p.born > 1960 AND p.born < 1970
RETURN p

-- Match multiple labels
MATCH (e:Person:Employee)
RETURN e
```

### Relationship Patterns
```cypher
-- Direct relationship
MATCH (p:Person)-[:ACTED_IN]->(m:Movie)
RETURN p, m

-- Relationship with properties
MATCH (p:Person)-[r:ACTED_IN]->(m:Movie)
WHERE r.role = 'Forrest Gump'
RETURN p, m, r

-- Variable length paths
MATCH (p1:Person)-[:KNOWS*1..3]-(p2:Person)
WHERE p1.name = 'Alice'
RETURN p2

-- Any relationship
MATCH (p:Person)-->(m:Movie)
RETURN p, m
```

## Spring Data Integration

### @Query Annotation
```java
public interface MovieRepository extends Neo4jRepository<Movie, Long> {

    @Query("MATCH (m:Movie) WHERE m.title = $title RETURN m")
    Optional<Movie> findByTitle(String title);

    @Query("""
        MATCH (p:Person)-[r:ACTED_IN]->(m:Movie)
        WHERE m.title = $title
        RETURN p, r, m
        """)
    List<Person> findActorsByMovieTitle(String title);

    @Query("""
        MATCH (m:Movie)
        WHERE m.released >= $startYear AND m.released <= $endYear
        RETURN m
        ORDER BY m.released DESC
        """)
    List<Movie> findByYearRange(int startYear, int endYear);
}
```

### Neo4jClient Queries
```java
@Service
@RequiredArgsConstructor
public class CypherService {

    private final Neo4jClient client;

    public List<Movie> findMoviesByActor(String actorName) {
        return client.query("""
            MATCH (p:Person)-[:ACTED_IN]->(m:Movie)
            WHERE p.name = $name
            RETURN m
            """)
            .bind(actorName).to("name")
            .fetchAs(Movie.class)
            .all();
    }

    public Map<String, Object> executeAndGetMap(String cypher, Map<String, Object> params) {
        return client.query(cypher)
            .bindAll(params)
            .fetch()
            .one()
            .orElse(Collections.emptyMap());
    }
}
```

## CRUD Operations

### Create
```java
@Service
public class PersonService {

    @Autowired
    private Neo4jClient client;

    public void createPerson(String name, int birthYear) {
        client.query("""
            CREATE (p:Person {name: $name, born: $born})
            RETURN p
            """)
            .bind(name).to("name")
            .bind(birthYear).to("born")
            .run();
    }

    public void createPersonIfNotExists(String name) {
        client.query("""
            MERGE (p:Person {name: $name})
            ON CREATE SET p.created = datetime()
            ON MATCH SET p.lastSeen = datetime()
            RETURN p
            """)
            .bind(name).to("name")
            .run();
    }

    public void createRelationship(String person1, String person2) {
        client.query("""
            MATCH (p1:Person {name: $name1}), (p2:Person {name: $name2})
            CREATE (p1)-[:KNOWS]->(p2)
            """)
            .bind(person1).to("name1")
            .bind(person2).to("name2")
            .run();
    }
}
```

### Update
```java
public void updatePerson(String name, String newEmail) {
    client.query("""
        MATCH (p:Person {name: $name})
        SET p.email = $email, p.updated = datetime()
        RETURN p
        """)
        .bind(name).to("name")
        .bind(newEmail).to("email")
        .run();
}

public void incrementViewCount(Long movieId) {
    client.query("""
        MATCH (m:Movie)
        WHERE id(m) = $id
        SET m.viewCount = coalesce(m.viewCount, 0) + 1
        RETURN m
        """)
        .bind(movieId).to("id")
        .run();
}
```

### Delete
```java
public void deletePerson(String name) {
    // Delete node only (fails if relationships exist)
    client.query("MATCH (p:Person {name: $name}) DELETE p")
        .bind(name).to("name")
        .run();
}

public void deletePersonWithRelationships(String name) {
    // DETACH DELETE removes node and all relationships
    client.query("MATCH (p:Person {name: $name}) DETACH DELETE p")
        .bind(name).to("name")
        .run();
}

public void deleteRelationship(String person1, String person2) {
    client.query("""
        MATCH (p1:Person {name: $name1})-[r:KNOWS]-(p2:Person {name: $name2})
        DELETE r
        """)
        .bind(person1).to("name1")
        .bind(person2).to("name2")
        .run();
}
```

## Aggregations

```java
public interface MovieRepository extends Neo4jRepository<Movie, Long> {

    @Query("""
        MATCH (m:Movie)
        RETURN avg(m.released) as avgYear,
               min(m.released) as minYear,
               max(m.released) as maxYear,
               count(m) as totalMovies
        """)
    MovieStatistics getStatistics();

    @Query("""
        MATCH (p:Person)-[:ACTED_IN]->(m:Movie)
        RETURN p.name as actor, count(m) as movieCount
        ORDER BY movieCount DESC
        LIMIT 10
        """)
    List<ActorMovieCount> getTopActors();

    @Query("""
        MATCH (m:Movie)
        RETURN m.released as year, count(m) as count
        ORDER BY year
        """)
    List<YearCount> getMoviesPerYear();

    @Query("""
        MATCH (p:Person)-[:ACTED_IN]->(m:Movie)
        WITH p, collect(m.title) as movies
        RETURN p.name as name, movies, size(movies) as count
        ORDER BY count DESC
        """)
    List<ActorWithMovies> getActorsWithMovies();
}
```

## Path Queries

```java
public interface PersonRepository extends Neo4jRepository<Person, Long> {

    @Query("""
        MATCH path = shortestPath(
            (p1:Person {name: $name1})-[*]-(p2:Person {name: $name2})
        )
        RETURN path
        """)
    List<Object> findShortestPath(String name1, String name2);

    @Query("""
        MATCH path = (p1:Person {name: $name1})-[:KNOWS*1..4]-(p2:Person {name: $name2})
        RETURN path, length(path) as pathLength
        ORDER BY pathLength
        LIMIT 1
        """)
    Object findConnectionPath(String name1, String name2);

    @Query("""
        MATCH (start:Person {name: $name})
        CALL apoc.path.spanningTree(start, {
            maxLevel: 3,
            relationshipFilter: 'KNOWS>'
        })
        YIELD path
        RETURN path
        """)
    List<Object> getNetworkTree(String name);
}
```

## Subqueries

```java
public interface MovieRepository extends Neo4jRepository<Movie, Long> {

    @Query("""
        MATCH (m:Movie)
        WHERE m.released > 2000
        CALL {
            WITH m
            MATCH (p:Person)-[:ACTED_IN]->(m)
            RETURN count(p) as actorCount
        }
        RETURN m, actorCount
        ORDER BY actorCount DESC
        """)
    List<MovieWithActorCount> getMoviesWithActorCount();

    @Query("""
        MATCH (m:Movie)
        WHERE EXISTS {
            MATCH (m)<-[:DIRECTED]-(d:Person)
            WHERE d.born < 1960
        }
        RETURN m
        """)
    List<Movie> findMoviesDirectedByVeterans();
}
```

## APOC Procedures

```java
public interface UtilityRepository {

    @Query("CALL apoc.meta.stats() YIELD labels, relTypes RETURN labels, relTypes")
    Map<String, Object> getDatabaseStats();

    @Query("""
        MATCH (p:Person {name: $name})
        CALL apoc.path.subgraphNodes(p, {
            maxLevel: 2,
            relationshipFilter: 'KNOWS|ACTED_IN>'
        })
        YIELD node
        RETURN node
        """)
    List<Object> getSubgraph(String name);

    @Query("""
        CALL apoc.periodic.iterate(
            'MATCH (p:Person) WHERE p.status IS NULL RETURN p',
            'SET p.status = "active"',
            {batchSize: 1000}
        )
        YIELD batches, total
        RETURN batches, total
        """)
    Map<String, Object> batchUpdateStatus();
}
```

## Full-Text Search

```java
public interface SearchRepository {

    // Requires full-text index
    // CREATE FULLTEXT INDEX movieFullText FOR (m:Movie) ON EACH [m.title, m.tagline]

    @Query("""
        CALL db.index.fulltext.queryNodes('movieFullText', $searchTerm)
        YIELD node, score
        RETURN node, score
        ORDER BY score DESC
        LIMIT 10
        """)
    List<MovieSearchResult> fullTextSearch(String searchTerm);
}
```

## Pagination in Cypher

```java
public interface MovieRepository extends Neo4jRepository<Movie, Long> {

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
    Page<Movie> findMoviesAfterYear(int year, Pageable pageable);
}

@Service
public class MovieService {

    @Autowired
    private MovieRepository repository;

    public Page<Movie> getMovies(int year, int page, int size) {
        return repository.findMoviesAfterYear(
            year,
            PageRequest.of(page, size)
        );
    }
}
```

## Performance Tips

```java
// Use parameters instead of string concatenation
// GOOD
@Query("MATCH (p:Person) WHERE p.name = $name RETURN p")
Person findByName(String name);

// BAD - vulnerable to injection
// "MATCH (p:Person) WHERE p.name = '" + name + "' RETURN p"

// Use indexes
// CREATE INDEX person_name FOR (p:Person) ON (p.name)
// CREATE INDEX movie_title FOR (m:Movie) ON (m.title)

// Profile queries
@Query("PROFILE MATCH (p:Person)-[:ACTED_IN]->(m:Movie) RETURN p, m")
List<Object> profileQuery();

// Explain queries
@Query("EXPLAIN MATCH (p:Person)-[:ACTED_IN]->(m:Movie) RETURN p, m")
List<Object> explainQuery();
```

## Best Practices

| Do | Don't |
|----|-------|
| Use parameters for values | Concatenate strings |
| Create indexes for queried properties | Query unindexed properties |
| Use MERGE for idempotent creates | CREATE duplicates |
| Limit result sets | Return unbounded results |
| Use DETACH DELETE for nodes | Delete nodes with relationships |
| Profile complex queries | Deploy unoptimized queries |

## Production Checklist

- [ ] Indexes created for query patterns
- [ ] Parameters used for all values
- [ ] Queries profiled and optimized
- [ ] Pagination implemented
- [ ] Transactions properly scoped
- [ ] Error handling for query failures
