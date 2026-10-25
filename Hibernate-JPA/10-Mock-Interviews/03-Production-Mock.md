# Mock Interview 3 — Production Troubleshooting and Spring Data JPA

## Q1. save() triggers a SELECT. Explain.
Spring Data may select merge semantics based on new-entity detection; merge can load state before copying changes.

**Follow-up:** How can a nullable @Version property affect new-entity detection?

## Q2. saveAndFlush() returns, but the row is not durable. Why?
Flush synchronizes SQL but does not commit. The surrounding transaction may still roll back.

**Follow-up:** How would you test this?

## Q3. No UPDATE appears after a setter. What do you inspect?
Managed state, value equality, transaction interception, read-only settings, flush/commit, detached state, and SQL logging.

**Follow-up:** Why can self-invocation affect @Transactional?

## Q4. LazyInitializationException occurs during JSON serialization. Fix it.
Define the fetch plan and map to a DTO inside the service transaction rather than loading arbitrary associations during serialization.

**Follow-up:** What risks arise from OSIV?

## Q5. A child row is not linked to its parent. Why?
The owning side may not be set; cascade may also be missing for a new child. Inspect mappings, helper methods, and SQL.

**Follow-up:** Does cascade set the owning side automatically?

## Q6. A paginated query returns duplicate roots. Investigate.
Inspect collection joins, row multiplication, DISTINCT semantics, sort order, and provider behavior. Prefer ID-first pagination or DTO projections.

**Follow-up:** Why doesn't DISTINCT always solve pagination?

## Q7. Bulk delete succeeds but a later read returns the deleted entity.
A managed entity or cache may be stale. Clear/refresh the persistence context and verify cache and transaction behavior.

**Follow-up:** What are the implications of clearing the entire context?

## Q8. An optimistic-lock exception occurs only in production.
Concurrent writes may be rare in tests. Inspect version values, request overlap, retry policy, and business invariants; reproduce with concurrent transactions.

**Follow-up:** What is a safe retry policy?

## Q9. Database time is low but request latency is high.
Inspect query count, connection acquisition, rows transferred, serialization, application work, and time inside transactions.

**Follow-up:** Which metrics distinguish pool wait from SQL time?

## Q10. Repository Page queries are slow.
Check the count query, indexes, offset depth, deterministic sorting, and whether Slice or keyset pagination meets requirements.

**Follow-up:** When is total count unnecessary?

## Q11. How do you troubleshoot SQL safely in production?
Use sampling, traces, and statement metrics; avoid exposing credentials or sensitive bind values. Capture explain plans with representative parameters where safe.

**Follow-up:** Why can SQL logging itself affect latency?

## Q12. Design a regression test for an ORM bug.
Use the actual provider and, where practical, the production database engine or a compatible test database. Assert query/result correctness and add transaction/concurrency tests when relevant.

**Follow-up:** Why might an in-memory database miss production behavior?
