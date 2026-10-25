# Fetching and Performance

## Q1. What is the N+1 query problem?
**Must Know**

One query loads root entities and additional queries load associations for each root, producing approximately 1 + N statements. It often appears when lazy associations are traversed in a loop, but eager mappings can also cause secondary selects. Diagnose with SQL logs, query counts, traces, and realistic data sizes.

## Q2. How do you fix N+1 queries?
**Must Know**

Choose a fetch join, entity graph, batch fetching, or DTO projection based on the endpoint and relationship cardinality. A giant join can multiply rows, so verify statement count and result volume after changing the fetch plan. Avoid globally switching associations to EAGER.

## Q3. What is a fetch join and what are its limitations?
**Must Know**

A JPQL fetch join loads an association as part of the query. It is useful for required relationships but joining multiple collections can multiply rows. Collection fetch joins with pagination are particularly problematic because SQL row limits do not correspond to unique roots. Hibernate behavior can vary by version/configuration; inspect warnings and never ignore in-memory pagination.

## Q4. What are entity graphs?
**High Value**

Entity graphs specify attributes to fetch for a query or find operation without embedding every fetch choice in JPQL. Named and dynamic graphs are useful for reusable use-case fetch plans. Providers may implement them with joins or secondary queries, so inspect generated SQL.

## Q5. Why not make all associations EAGER?
**Must Know**

EAGER does not guarantee one join and may load unnecessary data or trigger extra selects. Prefer LAZY mappings with explicit fetch plans for each use case. Make required data available inside the intended service boundary.

## Q6. What causes LazyInitializationException?
**Must Know**

Hibernate cannot initialize an unloaded proxy or collection after the persistence context/session is unavailable. Fetch required data inside the service transaction or map to a DTO before returning. Do not use EAGER everywhere or OSIV simply to hide missing fetch plans.

## Q7. What are Open Session in View trade-offs?
**High Value**

OSIV keeps a persistence context open through web request processing, allowing lazy loads during serialization. This can hide fetch-plan problems and cause unpredictable queries outside the service transaction. Whether it is enabled by default depends on framework configuration/version. Plan DTOs and fetches before disabling it.

## Q8. Why can collection fetch joins break pagination?
**Must Know**

A collection join creates multiple SQL rows per root, so limit/offset may cut across a root's rows. Providers may fetch many rows and paginate in memory, which is dangerous for large datasets. Page root IDs first and fetch associations in a second query, or use a paginated DTO projection.

## Q9. When should DTO projections be preferred?
**High Value**

Use projections when the use case needs selected fields rather than entity lifecycle tracking or relationship navigation. They can reduce transferred columns and avoid graph loading. Validate generated SQL, joins, and count queries instead of assuming every projection is optimal.

## Q10. What is batch fetching?
**High Value**

Batch fetching loads several eligible lazy associations/entities in one query using an IN-style predicate, reducing round trips without joining every collection into one result. Hibernate supports provider-specific configuration. Results depend on access patterns, dialect, and batch size.

## Q11. How do you diagnose slow Hibernate queries?
**Must Know**

Separate ORM overhead from database execution time. Inspect SQL and query counts, traces, pool metrics, execution plans, indexes, row estimates, lock waits, and transferred rows. Look for N+1 patterns, over-fetching, missing indexes, and long transactions. Optimize based on measured evidence.

## Q12. Why can fast SQL still produce a slow endpoint?
**High Value**

Hundreds of fast statements, connection-pool waits, large result sets, JSON serialization, and application work inside a transaction can dominate latency. Measure end-to-end timing, statement count, rows, pool wait, and serialization time.

## Official references
- [Hibernate User Guide](https://docs.jboss.org/hibernate/orm/current/userguide/html_single/Hibernate_User_Guide.html)
- [Spring Data JPA](https://docs.spring.io/spring-data/jpa/reference/)
