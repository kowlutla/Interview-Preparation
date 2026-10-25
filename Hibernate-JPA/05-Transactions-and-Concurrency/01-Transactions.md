# Transactions and Concurrency

## Q1. How does @Transactional interact with JPA?
**Must Know**

Spring transaction interception starts or joins a transaction and Spring's JPA integration associates a persistence context with it. Only entities loaded or persisted in that context are managed. Proxy-based interception can be bypassed by self-invocation. By default, Spring rolls back for unchecked exceptions and Errors, not every checked exception.

## Q2. Optimistic versus pessimistic locking?
**Must Know**

Optimistic locking detects conflicting writes, often through a version column, and suits relatively low contention. Pessimistic locking asks the database to lock rows and can block or deadlock. Choose based on contention, transaction duration, database behavior, and retry requirements.

## Q3. How does @Version prevent lost updates?
**Must Know**

The update checks the version observed when the entity was read and increments it on success. If another transaction updated the row first, the update affects no matching version and the provider raises an optimistic-lock failure. Retry only if the operation is safe to repeat and business rules still hold.

## Q4. What is a lost update?
**High Value**

Two transactions read the same value and both write replacements; the later write can overwrite the earlier one. Use versioned locking, suitable database locks, atomic SQL updates, or redesign the invariant. Reproduce with concurrent tests, as sequential unit tests often miss it.

## Q5. Does flush commit a transaction?
**Must Know**

No. Flush executes pending SQL within the current transaction. Commit completes it; rollback can undo flushed changes. Flush can surface constraint or lock errors early.

## Q6. What do isolation levels mean for JPA applications?
**High Value**

The database primarily enforces isolation, and exact anomaly behavior varies by engine. READ COMMITTED, REPEATABLE READ, and SERIALIZABLE are not identical across all databases. Test the actual invariant and use constraints/locking where correctness requires it.

## Q7. What does Spring Data JPA @Lock do?
**High Value**

It declares a JPA lock mode for a repository query. Lock duration, blocking, and supported modes depend on provider and database. Use it within a suitable transaction and inspect real lock behavior.

## Q8. Why can a transaction become rollback-only?
**High Value**

A persistence or runtime failure may mark the transaction rollback-only. Catching the exception does not necessarily restore commit eligibility, so the outer method can later get an unexpected rollback. Roll back and isolate independent work in a new transaction when that is the intended semantic.

## Q9. What are risks of bulk JPQL updates/deletes?
**Must Know**

Bulk operations act directly on database rows and bypass normal per-entity dirty checking and lifecycle callbacks. Managed entities can become stale. Flush pending changes when needed, execute the operation, then clear or refresh affected state deliberately.

## Q10. How do long transactions hurt performance?
**High Value**

They can hold connections and locks, increase contention, retain managed entities, and extend conflict windows. Avoid remote calls and lengthy computation inside a database transaction. Use transaction boundaries around the atomic database unit of work.

## Q11. Should an optimistic-lock exception always be retried?
**Advanced**

No. Retrying can repeat non-idempotent side effects or collide again. Reload current state, re-evaluate business rules, and use bounded retries only when safe. For user-edited data, returning a conflict may be correct; atomic SQL may suit commutative updates.

## Q12. How does transaction propagation affect persistence?
**High Value**

REQUIRED commonly joins or creates a transaction. REQUIRES_NEW suspends the outer transaction and uses an independent transaction that can commit even if the outer one later rolls back. Use it only when that behavior is intentional. Proxy interception and transaction-manager configuration matter.

## Official references
- [Spring transaction management](https://docs.spring.io/spring-framework/reference/data-access/transaction.html)
- [Jakarta Persistence](https://jakarta.ee/specifications/persistence/)
