# Mock Interview 2 — Senior Backend: Fetching, Transactions, Concurrency

## Q1. An endpoint has an N+1 issue. How do you fix it?
Measure statement count, identify association access, then compare projection, entity graph, fetch join, and batch fetching against realistic cardinality.

**Follow-up:** What changes when pagination and a collection are involved?

## Q2. Why can collection fetch join pagination be unsafe?
SQL rows multiply per root, so limits apply to rows rather than logical root entities. Page IDs first or use a suitable projection.

**Follow-up:** How would you detect in-memory pagination?

## Q3. When is a DTO projection preferable to entities?
When the endpoint needs a read shape, not lifecycle tracking or relationship navigation. It can reduce columns and avoid graph loading.

**Follow-up:** How do you verify generated SQL?

## Q4. Explain OSIV trade-offs.
It allows lazy loading during response serialization but can hide fetch-plan gaps and cause queries outside service transaction boundaries.

**Follow-up:** What refactoring helps when disabling it?

## Q5. Explain optimistic locking with @Version.
An update checks the previously read version and increments it on success; concurrent updates cause a conflict.

**Follow-up:** When should a conflict be retried versus returned?

## Q6. How do you prevent lost updates?
Use versioned optimistic locking, suitable pessimistic locking, atomic SQL, or redesign the invariant based on contention and correctness needs.

**Follow-up:** What are the costs of pessimistic locking?

## Q7. What happens when a bulk update affects a managed entity?
The persistence context may become stale because bulk operations bypass dirty checking. Flush as needed, then clear or refresh.

**Follow-up:** What about second-level cache invalidation?

## Q8. Why can a transaction be rollback-only after an exception was caught?
A participant may already have marked it rollback-only; catching the exception does not restore it.

**Follow-up:** How should independent work be isolated?

## Q9. The connection pool has high wait time. What do you inspect?
Transaction duration, slow SQL, remote calls inside transactions, leaked resources, pool metrics, and thread dumps.

**Follow-up:** Why is simply increasing pool size risky?

## Q10. How would you optimize high-volume inserts?
Measure batch execution, choose an identifier strategy compatible with batching, flush/clear in chunks, avoid large managed graphs, and preserve atomicity requirements.

**Follow-up:** When might JDBC or native loading be better?

## Q11. When is second-level caching worthwhile?
Stable, frequently reused data with an appropriate consistency model. Measure hit rates and test invalidation for out-of-band updates.

**Follow-up:** Why can query cache underperform?

## Q12. Walk through a persistence performance incident.
Establish baseline, capture traces and query counts, inspect plans and pool waits, test a measurable hypothesis, and validate correctness and improvement under realistic load.

**Follow-up:** What metric or test prevents recurrence?
