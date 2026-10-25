# Transactions and Concurrency

## Q1. How does @Transactional interact with JPA?
**Must Know**

In a typical Spring application, `@Transactional` tells Spring's transaction infrastructure to start or join a transaction around the method. Spring's JPA integration coordinates the EntityManager so persistence operations participate in that transaction. Managed changes are synchronized when the context flushes, and the transaction manager commits or rolls back the database transaction.

I check how the method is invoked, because proxy-based transaction advice can be bypassed by self-invocation. I also remember that Spring's default rollback rules generally apply to unchecked exceptions and Errors, not every checked exception.

## Q2. What is the difference between optimistic and pessimistic locking?
**Must Know**

I choose optimistic locking when conflicts are expected to be relatively rare. The application reads a row and later checks whether it has changed before writing. Pessimistic locking asks the database to lock the row or rows so conflicting work may have to wait.

Optimistic locking avoids holding database locks for the entire business operation, but callers need a conflict strategy. Pessimistic locking can be useful under high contention, but it can increase blocking and deadlock risk. The right choice depends on the invariant, contention, transaction length, and database behavior.

## Q3. How does @Version prevent lost updates?
**Must Know**

With a versioned entity, Hibernate includes the version value it read in the update condition and increments the version when the update succeeds. If another transaction has already changed that row, the stored version no longer matches. Hibernate detects that the update did not affect the expected row and raises an optimistic-lock exception.

For example, two users may open the same account settings page. If one saves first, the second user's stale version should not silently overwrite the first user's changes. I then decide whether the application should retry after reloading, ask the user to reconcile, or use a different update model.

## Q4. What is a lost update?
**High Value**

A lost update happens when two transactions read the same starting value, calculate new values independently, and then overwrite one another. The later write can erase the earlier change even though both requests appeared successful.

I prevent it using version-based optimistic locking, suitable pessimistic locking, an atomic database update, or a domain model that avoids the read-modify-write race. I choose based on the business invariant and test with genuinely concurrent transactions.

## Q5. Does flush commit a transaction?
**Must Know**

No. Flush sends pending SQL to the database within the current transaction. Commit completes the transaction, while rollback can undo the flushed changes. Flush is useful because it can reveal constraint violations or locking problems earlier, but it does not make the operation durable on its own.

## Q6. How do isolation levels affect JPA applications?
**High Value**

JPA applications inherit most isolation behavior from the database transaction. READ COMMITTED, REPEATABLE READ, and SERIALIZABLE can prevent different anomalies, and their precise behavior varies between database engines.

I don't assume that selecting an isolation name makes every database behave identically. I identify the business invariant, understand the actual database semantics, and use constraints or locking when correctness depends on preventing a specific race.

## Q7. What does @Lock in Spring Data JPA do?
**High Value**

I can annotate a repository query with `@Lock` to request a JPA lock mode, such as a pessimistic write lock where supported. The annotation expresses the request; it doesn't guarantee identical blocking behavior on every database. The transaction boundary, provider, dialect, and database all matter.

For a concurrency issue, I verify the SQL and lock behavior on the actual database rather than relying only on the annotation in the code.

## Q8. Why can a transaction be rollback-only after an exception was caught?
**High Value**

Catching an exception doesn't necessarily make a transaction healthy again. A persistence failure or participating transactional method may already have marked the transaction rollback-only. If the outer method continues and tries to commit, Spring can report an unexpected rollback.

I don't continue writing in a transaction after a serious persistence failure. I let it roll back and, if the work is genuinely independent, run that work in a separately designed transaction.

## Q9. What are the risks of bulk JPQL updates and deletes?
**Must Know**

Bulk JPQL operations update database rows directly; they don't update every already-managed entity through normal dirty checking or lifecycle callbacks. That means the persistence context can contain stale values after the query.

I flush pending changes first when required, execute the bulk operation, and then clear or refresh affected entities when subsequent code needs current state. I also consider cache invalidation and transaction semantics. Bulk operations are efficient, but their behavior is different from changing entities one by one.

## Q10. How can long transactions hurt performance?
**High Value**

Long transactions can hold connections and locks, increase contention, keep many entities managed, and extend the period in which concurrent updates can conflict. They become especially problematic when the code performs remote calls or lengthy calculations while the transaction is open.

I keep the transaction around the database work that needs to be atomic. External side effects may need an outbox or another explicit coordination pattern rather than simply extending the database transaction across network calls.

## Q11. Should every optimistic-lock exception be retried?
**Advanced**

No. A retry is only useful if repeating the operation is safe and the business rules still hold after reloading current state. A retry that blindly repeats a payment or another non-idempotent side effect can create a more serious bug.

For some operations, a bounded retry is reasonable. For user-edited records, returning a conflict may be the correct behavior. For simple increments, an atomic database update may be more appropriate than loading and rewriting the whole entity.

## Q12. How does transaction propagation affect persistence?
**High Value**

Propagation determines whether a method joins an existing transaction or uses a different one. REQUIRED commonly joins the current transaction or creates one if needed. REQUIRES_NEW suspends the outer transaction and creates an independent transaction, so its changes can commit even if the outer operation later rolls back.

I use REQUIRES_NEW only when that independence is intentional. Otherwise, it can leave partial business outcomes that callers did not expect. Proxy interception and the configured transaction manager also affect the actual behavior.

## Official references
- [Spring transaction management](https://docs.spring.io/spring-framework/reference/data-access/transaction.html)
- [Jakarta Persistence](https://jakarta.ee/specifications/persistence/)
