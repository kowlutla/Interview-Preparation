# JPQL, Criteria, and Querying

## Q1. JPQL versus SQL?
**Must Know**

JPQL queries entities, attributes, and relationships; SQL queries tables and columns. Native SQL supports database-specific features but couples queries more tightly to schema and dialect. Choose native SQL deliberately and map results clearly.

## Q2. Why bind query parameters?
**Must Know**

Named/positional parameters separate values from query structure and prevent values from changing the query, reducing injection risk. Dynamic identifiers such as sort columns generally need allowlist validation because bind parameters represent values, not column names.

## Q3. When should native SQL be used?
**High Value**

Use it for database-specific features, specialized tuning, reporting, or operations not well expressed in JPQL. The trade-off is portability and schema coupling. Measure a real limitation before replacing a portable query.

## Q4. How should pagination be implemented?
**Must Know**

Spring Data Pageable applies pagination and may run a count query for Page. Use Slice when total count is unnecessary. Ensure deterministic ordering with a unique tie-breaker. Deep offset pagination can be expensive and unstable under concurrent writes; keyset pagination can scale better for suitable access patterns.

## Q5. Page versus Slice?
**High Value**

Page includes total-count metadata and commonly requires a count query. Slice focuses on whether more data exists and can avoid calculating the total. Use the least expensive abstraction that meets the product requirement.

## Q6. What are DTO projections?
**High Value**

Projections retrieve only the values needed rather than materializing complete managed entities. Spring Data supports interface and class-based projections. They suit read endpoints and reports, but generated SQL depends on query/projection shape and should be inspected.

## Q7. What is the Criteria API?
**Advanced**

Criteria builds queries programmatically with a typed object model. It can help with dynamic optional filters but is verbose compared with JPQL. Use it when dynamic composition materially helps, not for every static query.

## Q8. What are Spring Data JPA Specifications?
**High Value**

Specifications encapsulate composable Criteria predicates for dynamic filters. Watch for joins that duplicate root rows and for count-query costs. Complex reporting may be clearer with a dedicated query or SQL.

## Q9. Derived query methods versus @Query?
**Must Know**

Derived methods are convenient for simple predicates. @Query makes complex joins, projections, and fetch plans explicit. Very long method names are difficult to maintain; choose the form that best communicates intent and exposes query behavior for review.

## Q10. How should modifying queries be handled?
**High Value**

Mark repository update/delete queries with @Modifying and run them in an appropriate transaction. Consider automatic flush/clear options only when needed: bulk queries bypass managed state and can make entities stale.

## Q11. Why can queries return duplicate roots?
**High Value**

Collection joins multiply SQL rows. Provider identity handling and DISTINCT semantics affect Java results, but DISTINCT does not automatically make collection-fetch pagination safe. Inspect SQL and result shape.

## Q12. What should you check when a query is slow?
**Must Know**

Inspect generated SQL, bind types, query count, cardinality, execution plan, indexes, sort operations, lock waits, transferred columns, and count-query cost. Tune using realistic data and database measurements.

## Official references
- [Jakarta Persistence](https://jakarta.ee/specifications/persistence/)
- [Spring Data JPA](https://docs.spring.io/spring-data/jpa/reference/)
