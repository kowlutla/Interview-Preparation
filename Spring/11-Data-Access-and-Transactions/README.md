# Spring Data Access and Transactions

## Q&A

### Q1. What does @Transactional do?

@Transactional declares transaction semantics for a method or class when transaction management is configured. In common Spring proxy mode, calls passing through the proxy are intercepted and a transaction manager coordinates the relevant resource transaction. The annotation does not make arbitrary remote calls atomic with a database write.

### Q2. What is transaction propagation?

Propagation determines how a method participates when a transaction already exists. REQUIRED joins an existing transaction or starts one; REQUIRES_NEW suspends an existing transaction and starts a separate one where supported; SUPPORTS can run with or without a transaction. Choose based on business consistency, locking, and failure semantics rather than treating propagation as a performance toggle.

### Q3. What is transaction isolation?

Isolation controls which effects of concurrent transactions are visible, subject to the database and transaction manager. Standard levels describe anomalies such as dirty reads, non-repeatable reads, and phantom reads, but exact behavior depends on the database implementation. Validate assumptions against the production database, not only an in-memory test database.

### Q4. Does Spring roll back for every exception?

Traditional declarative defaults generally roll back for unchecked exceptions and Errors, not all checked exceptions. Explicit rollback rules can change this. A caught exception that never escapes the transactional boundary may not trigger rollback automatically, and rollback-only status can lead to an unexpected rollback at commit time.

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

## Official references

- [Spring transaction management](https://docs.spring.io/spring-framework/reference/data-access/transaction.html)
- [JDBC abstraction](https://docs.spring.io/spring-framework/reference/data-access/jdbc.html)
