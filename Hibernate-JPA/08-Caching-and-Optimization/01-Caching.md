# Caching and Optimization

## Q1. First-level versus second-level cache?
**Must Know**

The first-level cache is the persistence context and is mandatory for entity identity within it. The optional second-level cache is shared across contexts within a factory scope and requires configuration. The first avoids repeat loads in one context; the second can reduce reads across transactions when consistency requirements permit.

## Q2. What is Hibernate query cache?
**Advanced**

It can cache query result information according to the configured strategy, but it is not a universal cache for arbitrary SQL results. It is usually useful only with a suitable second-level cache and stable access pattern. Invalidation and frequent updates can erase its value; measure hit rates.

## Q3. When does second-level caching help?
**High Value**

Frequently read, relatively stable data with a clear consistency requirement is a good candidate. Highly mutable data or strict freshness needs can make caching ineffective or unsafe. Choose the concurrency strategy and provider carefully; the database remains the source of truth.

## Q4. What are cache invalidation risks?
**High Value**

Out-of-band updates or paths that bypass cache coordination can leave stale values. Correctness depends on update paths, transaction boundaries, cache strategy, and acceptable staleness. Test external SQL and multi-instance behavior.

## Q5. What is JDBC batching?
**High Value**

Batching groups compatible statements to reduce database round trips. Confirm identifier strategy and SQL shape allow batching; identity generation may limit insert batching. Large jobs may flush and clear in chunks to bound persistence-context memory.

## Q6. Why might configured batching not help?
**Advanced**

The driver/provider may not batch, IDs may prevent batching, statement shapes may differ, or the bottleneck may be indexes, locks, constraints, and network latency. Verify actual batch execution and benchmark under realistic load.

## Q7. What is StatelessSession?
**Advanced**

Hibernate's provider-specific StatelessSession has reduced state management and lacks the normal persistence-context/automatic dirty-checking behavior. It can suit certain bulk workloads but removes ordinary entity lifecycle conveniences. Use only when these differences are intentional.

## Q8. How do you optimize entity loading?
**Must Know**

Avoid unnecessary queries and rows first with deliberate fetch plans, DTO projections, and pagination. Then measure batch fetching, indexes, batching, caching, and transaction scope. Capture baseline latency, query count, rows, pool metrics, and memory before tuning.

## Q9. Are read-only hints guaranteed to prevent writes?
**High Value**

No. ORM read-only flags can be hints or optimizations, depending on configuration. Database-level read-only transactions may have stronger semantics. Use permissions for security; do not rely on an ORM hint as an access-control mechanism.

## Q10. How should a bulk import be optimized?
**High Value**

Validate bounded chunks, batch compatible writes, avoid large object graphs, and flush/clear safely when using a stateful context. For very large loads, JDBC or database-native bulk loading may be better. Decide whether partial chunk commits are acceptable because they change atomicity.

## Official references
- [Hibernate User Guide](https://docs.jboss.org/hibernate/orm/current/userguide/html_single/Hibernate_User_Guide.html)
