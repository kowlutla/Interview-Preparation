# Production Troubleshooting Scenarios

## Q1. An endpoint triggers hundreds of SQL statements. How do you investigate?
**Must Know**

Capture a representative trace and query count, group statements by SQL shape, and look for repeated association loads or queries in loops. Correlate SQL logs with ORM statistics and application traces; avoid sensitive bind-value logging in production. Fix the fetch plan and re-test with realistic cardinality to ensure the fix did not create a huge join result.

## Q2. A request fails with LazyInitializationException. What is the right fix?
**Must Know**

Find where the unloaded association is accessed and where the persistence context ends. Load the required data inside the service transaction and map to a DTO before returning. Avoid making every relationship EAGER or relying on OSIV to hide the design issue.

## Q3. A field changed but no UPDATE appears. What do you check?
**Must Know**

Verify the entity is managed, the value differs, a transaction is active, and flush/commit occurs. Check proxy interception, self-invocation, detached state, read-only settings, and SQL logging for the correct datasource. No SQL may be correct when the value did not change.

## Q4. A bidirectional relationship is missing after save. Why?
**Must Know**

The owning side may not have been set. Update both sides using helper methods, ensure cascade when persisting a new child, flush, and inspect generated SQL and the foreign-key row.

## Q5. A paged endpoint becomes slow after adding a collection fetch join. Why?
**Must Know**

The join multiplies root rows, so database pagination may not match root-entity pagination. Some configurations may paginate in memory. Page root IDs first, then fetch associations, or use a paginated DTO projection.

## Q6. A bulk update runs, but subsequent reads show old values. Why?
**Must Know**

Bulk operations bypass normal synchronization of managed entities. Flush pending changes if necessary, run the bulk query, then clear or refresh affected state. Consider transaction isolation and second-level cache behavior.

## Q7. Optimistic locking fails intermittently under load. What now?
**High Value**

Treat it as a concurrent modification signal. Identify the conflicting row and business operation, inspect version increments, and decide whether bounded retry, conflict response, atomic SQL, or different locking fits the invariant. Retries must be repeat-safe.

## Q8. save() executes an unexpected SELECT. Is it a bug?
**High Value**

Not necessarily. Spring Data may choose merge based on new-entity detection, and merge may load state before copying values. Inspect identifier/version fields and the returned managed instance.

## Q9. Database connections remain occupied too long. What might be wrong?
**High Value**

Transactions may include remote calls or expensive work, streams may not close, slow queries may hold connections, or pool settings may be wrong. Inspect connection acquisition wait, transaction duration, query latency, and thread dumps. Shorten transaction scope before blindly increasing pool size.

## Q10. SQL statements are fast, but the service is slow.
**High Value**

The endpoint may issue too many statements, transfer too many rows, wait for a connection, serialize large graphs, or do application work inside a transaction. Measure query count, pool wait, rows, serialization, and end-to-end traces.

## Q11. An entity looks stale after another transaction updates it.
**High Value**

A managed entity is not refreshed automatically on every database change. Check persistence-context lifetime, isolation, second-level cache, and updates that bypass the ORM. Reload/refresh or start a new transaction as appropriate.

## Q12. A constraint violation occurs at commit rather than save.
**High Value**

SQL may be deferred until flush, and some database constraints are deferred until transaction end. Force a flush in a controlled diagnostic test, inspect the deepest database exception, and roll back rather than continuing after failure.

## Q13. A bulk import causes memory growth.
**High Value**

The persistence context may retain entities and snapshots. Process bounded chunks, flush/clear safely, consider JDBC/database-native loading, and preserve the required transaction atomicity.

## Q14. EAGER associations still generate many queries. Why?
**High Value**

EAGER requires availability, not a single SQL join. Hibernate may issue secondary selects depending on query shape and mapping. Define a use-case-specific fetch plan and inspect SQL.

## Q15. How do you approach a Hibernate performance incident?
**Must Know**

Establish impact; capture traces and query counts; measure latency, rows, pool waits, locks, and heap; inspect the dominant SQL and execution plan; change one variable; compare before/after under realistic load; add a regression test or monitoring signal.

## Official references
- [Hibernate User Guide](https://docs.jboss.org/hibernate/orm/current/userguide/html_single/Hibernate_User_Guide.html)
- [Spring Data JPA](https://docs.spring.io/spring-data/jpa/reference/)
