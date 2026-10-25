# Mock Interview 2 — Senior Backend: Fetching, Transactions, Concurrency

## Q1. An endpoint has an N+1 issue. How do you fix it?
I'd capture a representative trace and query count, identify the association that triggers repeated loads, and compare a projection, entity graph, fetch join, or batch fetching. I'd verify statement count and row volume after the change.

**Follow-up:** What changes when the association is a collection and pagination is required?

## Q2. Why can collection fetch join pagination be unsafe?
A collection join produces multiple SQL rows per root, so the database limit applies to rows rather than logical root entities. I'd consider paging root IDs first and then fetching associations, or use a DTO projection.

**Follow-up:** How would you detect in-memory pagination?

## Q3. When is a DTO projection preferable to entities?
If the endpoint needs a read-only shape and doesn't need entity lifecycle tracking or navigation, a DTO projection can retrieve fewer columns and avoid loading a large graph. I'd inspect the generated SQL rather than assuming every projection is optimal.

**Follow-up:** What would you check in the query plan?

## Q4. Explain OSIV trade-offs.
OSIV can allow lazy loading during response serialization, but it may hide missing fetch plans and trigger database calls outside the service transaction. I'd define the response data explicitly and map it to a DTO within the intended service boundary.

**Follow-up:** What refactoring helps when disabling OSIV?

## Q5. Explain optimistic locking with @Version.
Hibernate includes the version it read in the update condition and increments it after a successful write. If another transaction changed the row, the update no longer matches and Hibernate reports a conflict.

**Follow-up:** When would you retry versus return a conflict to the caller?

## Q6. How do you prevent lost updates?
I'd use versioned optimistic locking, a suitable pessimistic lock, an atomic database update, or redesign the invariant. The choice depends on contention and business semantics; pessimistic locking also brings blocking and deadlock risk.

**Follow-up:** How would you test the race?

## Q7. What happens when a bulk update affects a managed entity?
Bulk updates bypass normal entity dirty checking, so an entity already in the persistence context may still show old values. I'd flush pending changes if necessary, run the bulk query, and clear or refresh the affected state.

**Follow-up:** What about second-level cache invalidation?

## Q8. Why can a transaction be rollback-only after an exception was caught?
The persistence provider or a participating operation may already have marked it rollback-only. Catching the exception doesn't restore the transaction's ability to commit. I'd let it roll back and isolate independent work in a new transaction only if that is the intended business behavior.

**Follow-up:** Why can REQUIRES_NEW create surprising outcomes?

## Q9. The connection pool has high wait time. What do you inspect?
I'd look at transaction duration, slow queries, remote calls inside transactions, leaked resources, streaming query lifetimes, pool metrics, and thread dumps. Increasing pool size without fixing long-held connections can just increase database contention.

**Follow-up:** Which metric distinguishes pool wait from SQL execution time?

## Q10. How would you optimize high-volume inserts?
I'd measure whether JDBC batching is actually happening, check whether the identifier strategy permits it, and process entities in bounded chunks with flush/clear where appropriate. I'd preserve the import's required atomicity.

**Follow-up:** When might JDBC or database-native loading be a better fit?

## Q11. When is second-level caching worthwhile?
I'd consider it for frequently reused, relatively stable data with an acceptable consistency model. I'd measure cache hit rates and test invalidation, especially if data can be changed outside Hibernate.

**Follow-up:** Why might query cache underperform?

## Q12. Walk through a persistence performance incident.
I'd establish impact, capture traces and query counts, inspect SQL plans and connection-pool waits, and form a measurable hypothesis. Then I'd change one variable at a time and compare performance and correctness under realistic load.

**Follow-up:** What metric or regression test would help prevent recurrence?
