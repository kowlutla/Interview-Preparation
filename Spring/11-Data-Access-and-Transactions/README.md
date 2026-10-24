# Spring Data Access and Transactions

## Q&A

### Q1. What does @Transactional do?

@Transactional declares transaction semantics for a method or class when transaction management is configured. In common Spring proxy mode, calls passing through the proxy are intercepted and a transaction manager coordinates the relevant resource transaction. The annotation does not make arbitrary remote calls atomic with a database write.

### Q2. What is transaction propagation?

Propagation determines how a method participates when a transaction already exists. REQUIRED joins an existing transaction or starts one; REQUIRES_NEW suspends an existing transaction and starts a separate one where supported; SUPPORTS can run with or without a transaction. Choose based on business consistency, locking, and failure semantics rather than treating propagation as a performance toggle.

### Q3. What is transaction isolation?

Isolation controls which effects of concurrent transactions are visible, subject to the database and transaction manager. Standard levels describe anomalies such as dirty reads, non-repeatable reads, and phantom reads, but exact behavior depends on the database implementation. Validate assumptions against the production database, not only an in-memory test database.

### Q4. Does Spring roll back for every exception?

Traditional declarative defaults generally roll back for unchecked exceptions and Errors, not all checked exceptions. Explicit rollback rules can change this. Since Spring Framework 6.2, transaction management can also be configured with a global `rollbackOn = ALL_EXCEPTIONS` policy; version and transaction-specific rollback rules still matter. A caught exception that never escapes the transactional boundary may not trigger rollback automatically, and rollback-only status can lead to an unexpected rollback at commit time.

### Q5. Why might @Transactional appear not to work?

Common causes include self-invocation bypassing the proxy, a method not being intercepted, transaction management not being enabled/configured, an unexpected transaction manager, or exception handling that prevents rollback rules from applying. Inspect transaction logs, proxy boundaries, propagation, and the actual exception path.

### Q6. What is the difference between JdbcTemplate and NamedParameterJdbcTemplate?

JdbcTemplate provides JDBC operations with resource handling and exception translation, using positional placeholders. NamedParameterJdbcTemplate supports named parameters that are expanded into JDBC placeholders. Both reduce repetitive JDBC plumbing but do not eliminate the need for correct SQL, transaction boundaries, or query performance analysis.

### Q7. What does readOnly=true guarantee?

It communicates that a transaction is intended to be read-only and may allow transaction-manager or persistence-provider optimizations. It is not a universal guarantee that the database will reject writes; behavior depends on the transaction manager, driver, and database. Do not use it as a security control.

### Q8. Why is a database transaction not enough for reliable Kafka publishing?

A local database transaction and a broker publish usually involve separate resource managers. One can succeed while the other fails, creating inconsistent state. Depending on the use case, use an outbox pattern, idempotent consumer, retry strategy, or a deliberately designed transaction integration; do not assume @Transactional alone creates distributed atomicity.

### Q9. What causes connection pool exhaustion?

Requests may hold connections too long, transactions may be long-running, queries may be slow, pool size may be inappropriate, or downstream work may occur while a transaction remains open. Diagnose with pool active/idle/pending metrics, connection acquisition latency, transaction duration, query traces, and thread dumps. Increasing pool size without checking database capacity can worsen the incident.

### Q10. What resource-pool risk does REQUIRES_NEW introduce?

When an outer transaction holds a connection and a proxied inner call starts a REQUIRES_NEW transaction, the outer resources may remain bound while the inner transaction needs another connection. Under high concurrency, a small pool can become exhausted or cause threads to wait for connections held by their own suspended outer transactions.

Use REQUIRES_NEW only when independent commit semantics are required. Size and test the connection pool against nested transactional call patterns, keep transaction scopes short, and remember that self-invocation may prevent the intended propagation from being applied.

### Q11. When is NESTED propagation useful?

Where the transaction manager and resource support it, NESTED can use a savepoint inside an existing transaction. It is not universally supported and should not be assumed equivalent to REQUIRES_NEW, which creates a separate transaction.

### Q12. Why can a caught exception still lead to rollback failure at commit?

An inner operation may mark the transaction rollback-only before the exception is caught. The outer method then returns normally, but commit cannot succeed and may result in an unexpected rollback exception. Trace where rollback-only was set rather than focusing only on the final return path.

### Q13. What is the difference between a local transaction and a distributed workflow?

A local transaction coordinates supported resources through one transaction manager. A workflow spanning HTTP services, databases, and a broker needs explicit reliability semantics such as outbox, idempotency, saga compensation, or a supported distributed transaction protocol where appropriate.

### Q14. How can transaction boundaries affect connection pool usage?

A transaction may hold a connection until completion. Long computation or remote calls inside the boundary increase hold time and pool pressure. Measure transaction duration and connection acquisition latency, and keep the critical section focused on required database work.

### Q15. What is transaction-aware caching?

It coordinates cache operations with transaction completion so changes are not made visible at an inappropriate phase. Exact behavior depends on the cache abstraction/provider configuration. Verify rollback and after-commit behavior rather than assuming cache eviction is automatically atomic with the database.

### Q16. How should a service handle an optimistic-lock conflict?

Treat an optimistic-lock failure as evidence that another transaction changed the row after it was read, not as a generic transient database outage. Reload the current state and either re-evaluate the business command, return a conflict response, or ask the caller to reconcile. Retry only if the operation is safe and its rules can be recomputed against fresh data; do not blindly replay external side effects or overwrite the competing update.

### Q17. Why can an in-memory database hide transaction defects?

An in-memory database may differ from production in SQL dialect, implicit casts, constraint timing, lock acquisition, isolation semantics, sequence behavior, and query planning. A test can pass while a production query fails or concurrent updates violate assumptions. Use fast in-memory tests for focused logic, but run migrations and critical transaction/concurrency tests against the production database engine and a compatible version.

### Q18. How do you investigate a deadlock?

Collect database deadlock diagnostics and transaction/query logs, map lock acquisition order, and identify cycles. Reduce transaction scope, access rows in consistent order where possible, and implement safe retry for database-detected transient deadlocks only when the business operation is repeatable.

## Official references

- [Spring transaction management](https://docs.spring.io/spring-framework/reference/data-access/transaction.html)
- [JDBC abstraction](https://docs.spring.io/spring-framework/reference/data-access/jdbc.html)
