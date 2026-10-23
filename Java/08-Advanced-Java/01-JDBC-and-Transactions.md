# JDBC and Transactions

**Target level:** Intermediate → Senior  
**Scope:** Java 8–21

JDBC is Java's standard API for relational database access. Production database code must handle parameterization, resource ownership, transaction boundaries, isolation, connection pools, timeouts, and failure semantics. ORM frameworks build on these foundations but do not remove the need to understand them.

## 1. JDBC basics

### Q1. What is JDBC? *(Beginner)*

JDBC provides standard interfaces for obtaining connections, executing SQL, processing results, and managing transactions. A database driver implements the vendor-specific protocol behind those interfaces.

JDBC standardizes the API, not every database's SQL dialect or operational behavior. Verify driver compatibility, transaction semantics, type mappings, and performance for the target database.

### Q2. What is the difference between Statement and PreparedStatement? *(Intermediate)*

Statement executes SQL text directly. PreparedStatement supports parameterized SQL values and commonly allows the driver/database to reuse parsed or prepared execution plans where supported. Parameters separate values from SQL syntax and help prevent SQL injection.

Use PreparedStatement for user or application values. Parameter markers generally cannot replace table names, column names, or arbitrary SQL keywords; those require allowlisted query construction.

### Q3. What is ResultSet? *(Beginner)*

ResultSet represents tabular data returned by a query and normally advances through rows with next(). Column accessors read values by index or label. Many ResultSets are tied to the statement and connection that created them.

Read values while resources are open and close resources deterministically. Avoid returning a ResultSet from a method that closes its connection before the caller can use it.

### Q4. Why use try-with-resources in JDBC? *(Intermediate)*

Connection, Statement, and ResultSet implement AutoCloseable, so try-with-resources ensures they are closed on normal and exceptional paths. Closing a statement/result set and returning a connection to a pool are lifecycle operations that should not be left to garbage collection.

Use nested resource declarations or one resource block. In pooled applications, closing a logical Connection usually returns it to the pool rather than physically closing the database socket.

### Q5. What is connection pooling? *(Intermediate)*

A connection pool reuses physical database connections to avoid repeated setup costs and limit concurrent database sessions. It provides borrowed logical connections and returns them to the pool on close.

Pool size must align with database capacity, workload, and request concurrency. An oversized pool can overwhelm the database; an undersized pool can create long acquisition waits. Monitor active, idle, pending, and timeout metrics.

## 2. Transactions and isolation

### Q6. What is a database transaction? *(Beginner)*

A transaction groups operations into a unit with defined atomicity and consistency behavior, with isolation and durability guarantees provided by the database and configuration. JDBC controls transaction boundaries through Connection methods such as setAutoCommit, commit, and rollback.

Do not assume a sequence of statements is transactional when auto-commit is enabled. Define where the business operation begins and ends and how failures trigger rollback.

### Q7. What is auto-commit? *(Intermediate)*

With auto-commit enabled, each statement is generally committed as its own transaction according to the driver/database behavior. Setting auto-commit false allows multiple statements to be grouped until commit or rollback.

Always restore connection state if your code changes it before returning the connection to a pool. Pools may reset state, but application code should understand the pool's documented behavior and avoid leaking transaction settings between borrowers.

### Q8. What are transaction isolation levels? *(Advanced)*

JDBC defines READ_UNCOMMITTED, READ_COMMITTED, REPEATABLE_READ, and SERIALIZABLE, with database-specific behavior and support differences. Isolation determines which concurrent effects a transaction may observe, including dirty reads, non-repeatable reads, and phantoms under the database's implementation.

Do not assume identical behavior across database engines. Select isolation based on the invariant, performance, and concurrency requirements, then test under realistic contention.

### Q9. What is a lost update? *(Advanced)*

A lost update occurs when two transactions read the same value, compute independent updates, and one write overwrites the other's change. Isolation level alone may not prevent every application-level lost update depending on the database and access pattern.

Use optimistic version columns, atomic SQL updates, row locking, or an appropriate isolation/transaction design. Test concurrent updates rather than only sequential unit tests.

### Q10. What is optimistic locking? *(Advanced)*

Optimistic locking detects conflicting updates rather than locking a row for the entire user workflow. A version column is checked and incremented during update; if another transaction changed the row first, the update affects zero rows or the ORM raises an optimistic-lock exception.

It works well when conflicts are relatively rare. Callers must decide whether to retry, refresh, or report a conflict; blindly retrying can overwrite intentional user decisions.

### Q11. What is pessimistic locking? *(Advanced)*

Pessimistic locking acquires database locks to prevent conflicting operations while a transaction holds them, often using a SELECT ... FOR UPDATE form supported by the database. It can be appropriate for high-contention invariants.

Locks increase contention and deadlock risk, and long transactions hold resources. Keep transaction scope short and define lock ordering when operations touch multiple records.

## 3. Failure handling and production

### Q12. Why should transactions not span slow remote calls? *(Senior)*

A database transaction held open during a slow HTTP call can retain locks and connections, increase contention, and create long rollback/recovery paths. The remote call may succeed even if the database transaction later rolls back, creating a distributed consistency problem.

Use short database transactions and explicit orchestration patterns such as outbox, saga, idempotency, or reconciliation where multiple systems must coordinate. A local JDBC transaction cannot atomically commit a remote HTTP side effect.

### Q13. How should JDBC retries be designed? *(Senior)*

Classify errors by transient/permanent nature, preserve SQLState/vendor codes, and retry only operations that are safe to repeat. A connection failure during commit can leave the outcome uncertain: the database may have committed even if the client did not receive confirmation.

Use idempotency keys, unique constraints, transaction state checks, or reconciliation for uncertain writes. Do not blindly retry every SQLException.

### Q14. What is batch processing in JDBC? *(Intermediate)*

Batching groups multiple parameterized statements for execution through addBatch and executeBatch. It can reduce round trips and improve throughput, depending on the driver and database.

Handle partial batch results and failures according to the driver contract. Choose batch sizes based on memory, transaction duration, lock pressure, and database limits rather than maximizing the batch unconditionally.

### Q15. How should SQL resources be monitored in a service? *(Senior)*

Track connection acquisition latency, active/idle connections, pending borrowers, query duration, transaction duration, timeout counts, pool exhaustion, and database-side waits. Correlate slow endpoints with query plans and database metrics.

A large connection pool does not automatically increase throughput. When the database is saturated, more concurrent queries can increase queueing and tail latency.

### Q16. What are common JDBC security and correctness mistakes? *(Senior)*

Common mistakes include concatenating user values into SQL, leaking connections, swallowing rollback failures, relying on auto-commit accidentally, failing to close results, ignoring transaction outcome uncertainty, and logging sensitive query values.

Use parameterized queries, try-with-resources, explicit transaction boundaries, safe structured diagnostics, and tests for rollback and concurrent updates.

## Official references

1. [Java SE 21 API — java.sql](https://docs.oracle.com/en/java/javase/21/docs/api/java.sql/java/sql/package-summary.html)
2. [Java SE 21 API — Connection](https://docs.oracle.com/en/java/javase/21/docs/api/java.sql/java/sql/Connection.html)
3. [Java SE 21 API — PreparedStatement](https://docs.oracle.com/en/java/javase/21/docs/api/java.sql/java/sql/PreparedStatement.html)
4. [Java SE 21 API — SQLException](https://docs.oracle.com/en/java/javase/21/docs/api/java.sql/java/sql/SQLException.html)

## Chapter checklist

- [ ] Use parameterized SQL and deterministic resource cleanup.
- [ ] Understand auto-commit and transaction boundaries.
- [ ] Choose isolation/locking based on business invariants.
- [ ] Handle uncertain commit outcomes and retries safely.
- [ ] Monitor connection pools and database contention.
