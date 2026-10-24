# Spring Data JPA Integration

## Q&A

### Q1. What does Spring Data JPA add on top of JPA?

It provides repository abstractions, derived query parsing, pagination and sorting integration, projections, auditing support, and extension points. JPA still defines persistence semantics and the provider implements them; Spring Data does not eliminate the need to understand entity state, query plans, or transaction boundaries.

### Q2. How do repository interfaces work without a handwritten implementation?

Spring Data creates repository proxy implementations based on repository metadata and configured infrastructure. Method names, annotations, and fragments describe query behavior. When a derived query becomes hard to understand, prefer an explicit `@Query` or a custom repository fragment rather than an opaque method name.

### Q3. What is the purpose of Page, Slice, and List return types?

A Page commonly includes total-count metadata and may require an additional count query. A Slice indicates whether more data exists without necessarily calculating the full total. A List returns results without pagination metadata. Choose based on product needs and query cost.

### Q4. What is a projection?

A projection retrieves a subset or alternate view of data instead of exposing the full entity model. Interface and DTO projections can reduce unnecessary data transfer, but actual SQL and optimization depend on query form and provider behavior. Inspect generated SQL and query plans for critical paths.

### Q5. How should dynamic filtering be implemented?

Use a supported approach such as Specifications, query-by-example where suitable, or a custom repository query. Validate allowed filter and sort fields, combine predicates deliberately, and avoid string-concatenating untrusted SQL. Query design should remain understandable and testable.

### Q6. How do N+1 queries arise?

An initial query loads a collection of entities, then accessing relationships triggers additional queries per entity. Detect it with SQL logs, tracing, query-count assertions, and database metrics. Fix with an appropriate fetch strategy, projection, entity graph, or query redesign; indiscriminately eager-loading everything can create oversized joins.

### Q7. How do optimistic and pessimistic locking differ?

Optimistic locking detects conflicting updates, commonly using a version attribute, and fails one competing update rather than holding a long-lived lock throughout work. Pessimistic locking requests database locks for selected records. Choose based on contention, transaction duration, retry policy, and database capabilities.

### Q8. What repository abstractions should you use?

The available interfaces vary by Spring Data version. Common options include `CrudRepository`, `ListCrudRepository`, `PagingAndSortingRepository`, and `JpaRepository`; do not assume inheritance relationships from one version apply unchanged to another. Select the smallest useful contract and consult the reference for the chosen release.

## Official references

- [Spring Data JPA reference](https://docs.spring.io/spring-data/jpa/reference/)
- [Query methods](https://docs.spring.io/spring-data/jpa/reference/jpa/query-methods.html)

### Q9. Why can repository pagination return inconsistent results during concurrent writes?

Offset pages are positional, so inserts/deletes between requests can shift records across page boundaries. A deterministic sort reduces ambiguity but does not create a consistent snapshot across independent requests. For large changing datasets, consider cursor/keyset pagination and document its consistency model.

### Q10. How do Specifications help with dynamic queries?

Specifications compose predicates in a type-aware way for supported Spring Data JPA setups. They are useful when filters are optional, but complex joins and fetches can become difficult to reason about. Keep sorting, filtering, and fetch strategy deliberate and inspect generated SQL.

### Q11. Why can @Modifying queries leave managed entities stale?

Bulk JPQL/native updates can bypass ordinary entity-by-entity state tracking. The persistence context may still hold old values. Clear or refresh state when appropriate and understand flush/clear options; test subsequent reads in the same transaction.

### Q12. What is the risk of using native queries everywhere?

Native SQL can be necessary for database-specific features and performance, but it couples code to dialect/schema details and may reduce portability. Keep critical native queries documented and tested against the production database engine.

### Q13. How should auditing timestamps be handled?

Choose a consistent time basis, usually UTC instants for persisted event timestamps, and configure auditing infrastructure explicitly. Do not rely on an undocumented host default time zone; test serialization and display separately.

### Q14. How do you diagnose a query that is fast locally but slow in production?

Compare data volume/distribution, indexes, database statistics, execution plan, lock contention, parameter selectivity, and network/connection wait. A unit test cannot substitute for an execution plan on representative production-like data.
