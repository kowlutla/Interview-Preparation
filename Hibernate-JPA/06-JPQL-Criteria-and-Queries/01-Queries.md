# JPQL, Criteria, and Querying

## Q1. What is the difference between JPQL and SQL?
**Must Know**

I use JPQL when I want to query the entity model rather than the physical schema. The query refers to entity names, Java attributes, and mapped relationships. Native SQL refers directly to tables and columns and can use database-specific features.

JPQL is often easier to keep portable, while native SQL can be the right choice for specialized reporting or database features. I choose based on the query's needs and inspect the generated SQL either way.

## Q2. Why should query parameters be bound instead of concatenated?
**Must Know**

I bind user-supplied values because parameters keep the data separate from the query structure and help prevent SQL injection. For example, I would use a named parameter for a product SKU instead of building the query string by concatenating the SKU value.

One subtle point is that parameters generally bind values, not arbitrary identifiers such as a column name or sort direction. If users can choose sorting fields, I validate those choices against an allowlist rather than concatenating unchecked input.

## Q3. When would you use native SQL?
**High Value**

I use native SQL when the query needs a database-specific feature, a specialized report, or a form of tuning that is awkward or impossible in JPQL. It can also be useful for carefully designed bulk operations.

The trade-off is tighter coupling to the schema and database dialect, plus potentially more involved result mapping. I first establish a real requirement with measurements rather than switching to native SQL just because it looks more familiar.

## Q4. How would you implement pagination safely?
**Must Know**

For a typical Spring Data query, I can pass a Pageable and receive a Page or Slice. I make the sort deterministic, ideally including a unique tie-breaker, so rows don't move unpredictably between pages when sort values are equal.

I also consider the cost of deep offsets and count queries. For very large or frequently changing datasets, keyset pagination may be more efficient. If the query fetches a collection, I avoid combining a collection fetch join with ordinary pagination without checking its behavior.

## Q5. What is the difference between Page and Slice?
**High Value**

I use Page when the caller needs total-element or total-page information. That commonly means Spring Data needs a count query in addition to fetching the requested rows. If the UI only needs to know whether another page exists, Slice can avoid the full count and may be cheaper.

I choose the return type based on what the caller actually needs rather than calculating a total count by default.

## Q6. When are DTO projections useful?
**High Value**

If an endpoint needs only a few fields—for example, an order number, status, and creation date—I usually don't need to load a full entity graph. A DTO projection can select just those values, reduce data transfer, and avoid unnecessary persistence-context tracking.

Spring Data JPA supports interface and class-based projections. I still inspect the generated SQL, because query shape and nested projections can affect which columns and joins are used.

## Q7. What is the Criteria API, and when would you use it?
**Advanced**

The Criteria API lets me construct queries programmatically using a typed object model. It can be useful when a search screen has many optional filters and I need to compose predicates dynamically. The trade-off is verbosity; a simple static query is often clearer as JPQL or a repository method.

I use Criteria when dynamic composition makes the code easier to maintain, not as the default for every query.

## Q8. What are Spring Data JPA Specifications?
**High Value**

A Specification wraps a Criteria predicate so I can compose reusable filters. For example, a product search could optionally filter by category, status, and price range without creating a separate repository method for every combination.

I watch for joins that duplicate root rows and for expensive count queries during pagination. If the use case becomes reporting-heavy or structurally complex, a dedicated query implementation may be easier to understand.

## Q9. When should I use a derived query method versus @Query?
**Must Know**

Derived methods are convenient when the predicate is simple and the method name remains readable. When a query has several joins, a projection, a fetch plan, or complicated conditions, I prefer an explicit `@Query` because the intent is easier to review.

I avoid extremely long method names that encode an entire query in a method signature. The goal is maintainable query logic, not maximizing the amount of query text hidden in a name.

## Q10. How should I handle a modifying repository query?
**High Value**

For an update or delete query declared with `@Query`, I use `@Modifying` and execute it in an appropriate transaction. These bulk operations bypass normal entity-by-entity dirty checking, so an entity already loaded in the persistence context may still show its old values.

I decide whether to flush pending changes first and whether to clear or refresh managed state afterward. I don't turn on automatic clearing without considering that it detaches all managed entities in that context.

## Q11. Why can a query return duplicate root entities?
**High Value**

When I join a root entity to a collection, the SQL result can contain multiple rows for the same root—one per matching child. How that appears in the Java result depends on query semantics and provider behavior. DISTINCT can help in some cases, but it doesn't automatically make collection-fetch pagination safe.

I inspect the SQL and result shape, then decide whether a projection, a different join, or a two-step fetch is more appropriate.

## Q12. What do you investigate when a query is slow?
**Must Know**

I start with the actual SQL and parameters, then inspect the execution plan, indexes, join cardinality, sorting, row counts, and lock waits. I also check whether the ORM is issuing too many statements or retrieving columns and relationships the endpoint doesn't need. For paged results, I look at the count query separately.

I test with representative data volume because a query that looks fine on a tiny development database can behave very differently in production.

## Official references
- [Jakarta Persistence](https://jakarta.ee/specifications/persistence/)
- [Spring Data JPA](https://docs.spring.io/spring-data/jpa/reference/)
