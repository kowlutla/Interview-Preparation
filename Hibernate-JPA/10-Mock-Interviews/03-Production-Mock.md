# Mock Interview 3 — Production Troubleshooting and Spring Data JPA

## Q1. save() triggers a SELECT. Explain.
I'd check Spring Data's new-entity detection first. It may choose merge rather than persist, and merge can load the existing managed state before copying values. I'd inspect identifier/version configuration and use the managed instance returned by merge.

**Follow-up:** How can a nullable @Version property affect new-entity detection?

## Q2. saveAndFlush() returns, but the row is not durable. Why?
Flush executes pending SQL, but it doesn't commit the surrounding transaction. If the transaction later rolls back, the database changes can be undone.

**Follow-up:** How would you demonstrate this in a test?

## Q3. No UPDATE appears after a setter. What do you inspect?
I'd check managed state, whether the value actually changed, the transaction boundary, proxy interception, detached state, read-only configuration, and SQL logging. I wouldn't add save() until I understood why dirty checking wasn't producing the expected result.

**Follow-up:** Why can self-invocation affect @Transactional?

## Q4. LazyInitializationException occurs during JSON serialization. Fix it.
I'd identify the association being accessed and define the response's data requirements. Then I'd load the required data inside the service transaction and map it to a DTO before returning, rather than relying on serialization to trigger database access.

**Follow-up:** What risks arise from OSIV?

## Q5. A child row is not linked to its parent. Why?
I'd check the owning side and foreign-key mapping. In a common one-to-many/many-to-one mapping, setting only the parent's collection may not set the child's owning reference. I'd also check cascade if the child is new and inspect the SQL.

**Follow-up:** Does cascade automatically set the owning side?

## Q6. A paginated query returns duplicate roots. Investigate.
I'd inspect collection joins and SQL row multiplication, then check DISTINCT behavior, ordering, and provider version. For pagination, I'd consider a two-step ID-first query or a projection instead of assuming DISTINCT fixes everything.

**Follow-up:** Why doesn't DISTINCT always solve pagination?

## Q7. Bulk delete succeeds but a later read returns the deleted entity.
I'd check whether the entity is still managed in the current persistence context or cached elsewhere. Bulk operations bypass normal entity synchronization, so I may need to clear or refresh state and verify cache and transaction behavior.

**Follow-up:** What are the trade-offs of clearing the whole context?

## Q8. An optimistic-lock exception occurs only in production.
I'd look for overlapping updates, inspect version values and the business operation, and try to reproduce with concurrent transactions. Then I'd decide whether bounded retry, conflict response, atomic SQL, or a different locking strategy fits the invariant.

**Follow-up:** What makes a retry safe?

## Q9. Database time is low but request latency is high.
I'd compare end-to-end traces, total query count, connection acquisition wait, rows transferred, serialization time, and application work inside the transaction. Fast individual statements don't rule out too many round trips or a long-held connection.

**Follow-up:** Which metrics distinguish pool wait from SQL time?

## Q10. Repository Page queries are slow.
I'd inspect the count query, indexes, offset depth, and deterministic sorting. If the caller doesn't need a total count, Slice may help; for deep pages, I'd evaluate keyset pagination.

**Follow-up:** When is total count unnecessary?

## Q11. How do you troubleshoot SQL safely in production?
I'd start with tracing, sampling, and statement metrics, avoiding credentials or sensitive bind values in logs. Where safe, I'd capture representative SQL and examine its execution plan. I also account for the overhead that verbose SQL logging can add.

**Follow-up:** Why can SQL logging itself affect latency?

## Q12. Design a regression test for an ORM bug.
I'd use the actual JPA provider and, where practical, the production database engine or a compatible test database. I'd assert both result correctness and important query behavior, and add concurrent tests for locking issues. An in-memory database may not reproduce the production engine's SQL or locking semantics.

**Follow-up:** What should be covered in a concurrency regression test?
