# Production Troubleshooting Scenarios

## Q1. An endpoint triggers hundreds of SQL statements. How do you investigate?
**Must Know**

I would start with a representative request and capture its trace and SQL statement count. Then I'd group statements by shape and look for repeated queries that load associations inside a loop. ORM statistics and application traces help connect the SQL to the code path; I avoid logging sensitive bind values in production.

Once I identify the access pattern, I'd compare a projection, fetch join, entity graph, or batch fetching. I'd re-test with realistic result sizes to make sure I reduced round trips without creating an enormous join result.

## Q2. A request fails with LazyInitializationException. What is your approach?
**Must Know**

First, I'd locate which lazy association is accessed and where the persistence context ends. A common cause is returning an entity from a service and having JSON serialization touch an uninitialized collection afterward.

I'd define the data the endpoint actually needs, fetch it inside the intended transaction, and map it to a DTO before returning. I wouldn't make every association EAGER as a quick fix, because that can introduce N+1 queries and over-fetching elsewhere.

## Q3. A field changes in Java, but no UPDATE appears. What do you check?
**Must Know**

I'd check whether the entity is managed, whether the new value differs from the old value, and whether the operation runs inside the expected transaction. I'd also verify that the transactional method is actually intercepted, that the entity wasn't detached, and that SQL logging is enabled for the correct datasource.

If the field didn't really change, no UPDATE may be correct. If it did, I'd follow the operation through flush and commit rather than adding a save call without understanding the cause.

## Q4. A bidirectional relationship is missing after save. Why?
**Must Know**

I'd inspect which side owns the relationship and whether that side was set. In a common parent-child mapping, adding the child to the parent's collection is not enough if the child's foreign-key reference remains null. I'd also check cascade settings if the child is new.

Then I'd flush and inspect the generated SQL and foreign-key value. Helper methods that update both sides can prevent this class of bug from recurring.

## Q5. A paginated endpoint becomes slow after a collection fetch join. Why?
**Must Know**

The join can produce several SQL rows per root entity, so the database's limit and offset no longer correspond cleanly to logical root entities. Depending on Hibernate version and configuration, the provider may warn or paginate in memory, which can load far more data than expected.

I'd consider paging root IDs first and fetching their associations in a second query, or using a DTO projection. I'd verify both the result correctness and the execution plan after the change.

## Q6. A bulk update runs, but later reads show old values. Why?
**Must Know**

Bulk JPQL updates bypass the normal synchronization of each managed entity. If the persistence context already contains an entity affected by the update, that Java object can still hold the old value.

I'd flush pending changes if needed, run the bulk query, and then clear or refresh the affected state when later code needs current values. If stale data persists across contexts, I'd also check transaction isolation and second-level cache behavior.

## Q7. Optimistic locking fails intermittently under load. What would you do?
**High Value**

I'd treat the exception as evidence that another transaction changed the row, then identify which business operations overlap and inspect the version values. The exception isn't automatically a Hibernate defect; tests with sequential requests may never reproduce it.

Next I'd decide whether the operation can safely retry after reloading, should return a conflict to the caller, or needs a different update/locking strategy. Any retry must be bounded and safe to repeat.

## Q8. save() executes an unexpected SELECT. Is that a bug?
**High Value**

Not necessarily. Spring Data may decide the entity is not new and call merge; merge can load existing state before copying the supplied state into the managed instance. This can happen with application-assigned IDs or a new-entity detection strategy that doesn't match the entity's lifecycle.

I'd inspect the ID/version fields and the returned instance before changing the implementation. The goal is to correct the lifecycle semantics, not to eliminate a SELECT at any cost.

## Q9. Database connections remain occupied for too long. What do you investigate?
**High Value**

I'd inspect connection-pool wait time, transaction duration, slow queries, thread dumps, and whether the code holds a transaction open while making remote calls or doing expensive work. Streaming queries and resources that aren't closed can also keep connections occupied.

I'd fix the actual cause before increasing pool size, because a larger pool can increase database contention without reducing the time each connection is held.

## Q10. SQL statements are fast, but the service is slow. Why?
**High Value**

The service may be executing too many individually fast statements, waiting for a connection, transferring too many rows, serializing a large graph, or doing application work inside a transaction. The slowest single SQL statement doesn't necessarily explain total request latency.

I'd compare end-to-end traces, total statement count, rows transferred, connection wait, and serialization time. Then I'd optimize the dominant measured cost.

## Q11. An entity appears stale after another transaction updates it. Why?
**High Value**

A managed entity isn't automatically refreshed every time another transaction changes the database. The current persistence context may still hold the previous state, and isolation or second-level caching can also affect what is visible.

I'd check the context and transaction lifetime first, then reload or refresh as appropriate. If another service updates the same table directly, I'd also review how that update interacts with the cache strategy.

## Q12. A constraint violation appears at commit rather than save. Why?
**High Value**

The save operation may only have changed managed state, with SQL deferred until flush. Some databases also support constraints that are checked at transaction end. So the point where the exception appears doesn't necessarily mean the invalid value was introduced at commit.

I'd inspect the deepest database exception and reproduce with a controlled flush to determine when the constraint is checked. After a persistence failure, I'd roll back instead of continuing in the same failed transaction.

## Q13. A bulk import causes memory growth. How would you address it?
**High Value**

I'd check whether one persistence context is retaining every entity and its tracking state. For large imports, I'd process bounded chunks and flush/clear at safe intervals, or consider JDBC/database-native loading if ORM lifecycle behavior isn't needed.

Before introducing chunk commits, I'd confirm whether the import must be atomic. Smaller transactions can reduce memory and lock duration, but they change what happens if a later chunk fails.

## Q14. EAGER associations still generate many queries. Why?
**High Value**

EAGER means the association must be available; it doesn't require Hibernate to fetch everything through one join. Depending on the query and mapping, secondary selects can still occur and create an N+1 pattern.

I'd inspect the actual SQL and replace the global expectation with a use-case-specific fetch plan, such as a fetch join, entity graph, or projection.

## Q15. How would you handle a Hibernate performance incident end to end?
**Must Know**

I'd first establish impact and capture a representative trace. Then I'd measure total latency, query count, rows transferred, connection-pool waits, lock waits, and memory, and identify the dominant query or access pattern. I'd inspect the database execution plan and form a specific hypothesis.

I'd change one variable at a time, compare before and after under realistic load, and verify that data correctness hasn't regressed. Finally, I'd add a regression test or monitoring signal so the same problem is easier to detect next time.

## Official references
- [Hibernate User Guide](https://docs.jboss.org/hibernate/orm/current/userguide/html_single/Hibernate_User_Guide.html)
- [Spring Data JPA](https://docs.spring.io/spring-data/jpa/reference/)
