# Caching and Optimization

## Q1. What is the difference between first-level and second-level cache?
**Must Know**

The first-level cache is the persistence context that Hibernate uses to track managed entities and preserve identity within that context. It is part of normal entity management. The second-level cache is optional and can share cached entity data across persistence contexts, depending on configuration.

So if I load the same entity twice in one context, Hibernate can reuse the managed instance. A second-level cache can reduce database reads across transactions, but only if the data's update frequency and consistency requirements make caching worthwhile.

## Q2. What is Hibernate's query cache?
**Advanced**

Hibernate's query cache can cache query result information according to its configured strategy, but it isn't a universal cache for every SQL result. Its usefulness depends on the query pattern, cache configuration, and how the underlying data changes; it is often considered alongside second-level caching.

I don't enable it across the application just because it exists. I measure hit rates and database load, and I verify invalidation behavior for the actual access pattern.

## Q3. When does second-level caching help?
**High Value**

I consider it for data that is read frequently, changes relatively infrequently, and has a clear acceptable-consistency model—for example, reference data shared across many requests. It is less attractive for highly mutable records or data that must always be fresh.

I choose a suitable cache provider and concurrency strategy, and I remember that the database remains the source of truth. A cache is useful only if the reduction in database work outweighs invalidation and operational complexity.

## Q4. What can go wrong with cache invalidation?
**High Value**

Stale data can appear when updates happen through direct SQL, another application, or a code path that doesn't coordinate with the same cache. The outcome depends on the cache strategy, transaction boundaries, and whether all writers participate in invalidation.

I test those out-of-band update paths explicitly. If the application cannot guarantee the required freshness, I either change the caching design or avoid caching that data.

## Q5. What is JDBC batching in Hibernate?
**High Value**

JDBC batching groups compatible statements so the driver can send them in fewer round trips. This can improve high-volume insert or update workloads, but it depends on configuration, driver support, statement shape, and identifier generation. Identity-generated IDs can constrain insert batching because the generated key may be needed immediately.

For a batch job, I also consider flushing and clearing in chunks so the persistence context doesn't grow without bound. Then I verify actual batching rather than assuming a configuration property is enough.

## Q6. Why might configured batching not improve performance?
**Advanced**

The statements may not actually be batched, the identifier strategy may prevent it, or the SQL statements may not be compatible for batching. Even when batching works, the main cost may be indexes, constraints, lock contention, or database execution rather than network round trips.

I check the executed batch behavior and compare throughput under realistic load. Configuration is only a hypothesis; measured improvement is the evidence.

## Q7. What is Hibernate StatelessSession?
**Advanced**

StatelessSession is a Hibernate-specific API intended for certain workloads that need reduced state management. It does not provide the usual persistence-context behavior and automatic dirty checking that I rely on with a normal Session.

It can be useful for specialized bulk processing, but I wouldn't substitute it casually into ordinary domain logic. I first confirm that the application doesn't depend on the lifecycle and identity semantics that a stateful session provides.

## Q8. How do you approach optimizing entity loading?
**Must Know**

I start by removing unnecessary work: reduce repeated queries, fetch only the associations the use case needs, use projections when I don't need full entities, and paginate appropriately. Then I measure whether batch fetching, indexes, JDBC batching, or caching would address the remaining bottleneck.

Before tuning, I record a baseline such as latency, SQL count, rows transferred, pool wait, and memory. Afterward, I compare the same metrics under realistic conditions and verify correctness.

## Q9. Do read-only hints guarantee that writes cannot happen?
**High Value**

No. An ORM read-only hint can influence tracking or optimization, but its enforcement depends on the provider and configuration. A database-level read-only transaction may have stronger behavior, but I don't assume every layer interprets the hint identically.

If preventing writes is a correctness or security requirement, I enforce it through appropriate permissions and application design, not just an ORM annotation.

## Q10. How would you optimize a bulk import?
**High Value**

I first determine the required atomicity: must the whole import succeed or fail as one unit, or are chunk-level commits acceptable? Then I validate input in bounded chunks, use JDBC batching when supported, avoid loading large graphs, and flush/clear at safe points if I use a stateful persistence context.

For very large imports, JDBC or database-native bulk loading may be more appropriate. I measure throughput and memory, and I account for the fact that committing each chunk changes rollback semantics.

## Official references
- [Hibernate User Guide](https://docs.jboss.org/hibernate/orm/current/userguide/html_single/Hibernate_User_Guide.html)
