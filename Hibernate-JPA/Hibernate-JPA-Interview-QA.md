# Hibernate, JPA, and Spring Data JPA Interview Handbook

A focused guide for Java backend engineers who need to explain ORM behavior, query performance, transaction boundaries, and production failure modes. It distinguishes the Jakarta Persistence contract from Hibernate-specific behavior and Spring Data repository conventions.

**Audience:** Experienced Java backend engineers.  
**Version scope:** Modern Jakarta Persistence, Hibernate ORM, and Spring Data JPA; check the exact provider, Spring Boot, JDBC driver, and database versions before relying on version-sensitive behavior.  
**Study approach:** Learn the lifecycle and SQL implications behind each feature, not just the annotation name.

## Contents

1. [Foundations and persistence context](#1-foundations-and-persistence-context)
2. [Entity mapping and lifecycle](#2-entity-mapping-and-lifecycle)
3. [Fetching and query performance](#3-fetching-and-query-performance)
4. [Transactions and concurrency](#4-transactions-and-concurrency)
5. [Queries and Spring Data JPA](#5-queries-and-spring-data-jpa)
6. [Caching and production troubleshooting](#6-caching-and-production-troubleshooting)
7. [Mapping and write performance](#6a-mapping-and-write-performance)
8. [Mapping details and persistence operations](#7-mapping-details-and-persistence-operations)
9. [Repository design, dynamic queries, and batching](#8-repository-design-dynamic-queries-and-batching)
10. [Testing and production diagnosis](#9-testing-and-production-diagnosis)
11. [Senior-level production scenarios](#10-senior-level-production-scenarios)
12. [Last-minute revision](#11-last-minute-revision)
13. [Official references and version notes](#12-official-references-and-version-notes)
14. [Editorial and validation notes](#13-editorial-and-validation-notes)

---

## 1. Foundations and persistence context

### Q1. What is the difference between JPA, Hibernate, and Spring Data JPA?

Jakarta Persistence (formerly JPA) is a specification defining standard APIs and behavior for object-relational persistence. Hibernate ORM is a popular provider that implements that specification and also offers provider-specific features. Spring Data JPA builds on the JPA layer and provides repository abstractions, query derivation, projections, pagination, and integration with Spring's repository infrastructure.

These are different layers, not competing alternatives. Standard JPA APIs improve provider portability, but providers can generate different SQL and have different extensions and performance characteristics. Spring Data JPA reduces boilerplate, but it does not remove the need to understand SQL, transactions, entity state, or fetching.

### Q2. What is a persistence context?

A persistence context is the managed workspace in which the provider tracks entity instances and their persistent identities. Within one persistence context, the same entity type and database identity normally correspond to one managed Java instance. The provider tracks changes to managed entities and synchronizes them with the database during flushing.

The persistence context is not the database itself, and it is not synonymous with a transaction, although application frameworks commonly scope it around a transaction or request. A long-lived context can accumulate many managed entities and become expensive; clear lifecycle boundaries are important for memory usage and correctness.

### Q3. What are the entity lifecycle states?

A **new/transient** object is not managed by a persistence context. A **managed** entity is tracked, so changes can be detected and synchronized. A **detached** entity is no longer tracked by that context, although it may correspond to an existing database row. A **removed** entity is managed but marked for deletion.

These states explain why changing a managed entity may persist without an explicit repository save call, while changing a detached object usually does not update the database automatically. The transition rules depend on operations such as `persist`, `merge`, `remove`, detachment, and context lifecycle.

### Q4. What is dirty checking, and does every update require save()?

Dirty checking is the provider's mechanism for detecting changes to managed entities and synchronizing those changes to the database. In a typical transaction, loading an entity, changing a field, and committing can result in an UPDATE even if `save()` is never called, because the entity is already managed.

Calling `save()` on an already managed entity is often unnecessary for dirty checking, though repository conventions and abstraction boundaries may still make it appear in service code. The important condition is that the entity remains managed and the transaction/context lifecycle allows the changes to be flushed. Dirty checking is not automatic tracking for arbitrary detached Java objects.

### Q5. What is the difference between flush and commit?

A flush synchronizes pending persistence-context changes with the database, often by executing INSERT, UPDATE, or DELETE statements. Commit completes the database transaction. Flushing does not itself mean the transaction is committed; later work can still fail and the transaction can roll back.

The provider may flush before transaction commit and, depending on flush mode and query semantics, before certain queries. This explains why constraint violations can surface earlier than expected and why SQL may execute before the service method returns. Do not use flush as a substitute for commit or as proof that another transaction can see the changes.

### Q6. What is the difference between persist() and merge()?

`persist()` makes a new entity instance managed and schedules its insertion according to persistence-context and transaction behavior. `merge()` copies state from a new or detached object into a managed instance and returns that managed instance; the original object does not become managed merely because merge was called.

A common mistake is to ignore the value returned by `merge()` and continue changing the original detached object, expecting those changes to be tracked. In Spring Data JPA, `save()` typically chooses between persist and merge based on its entity-new detection strategy, which can depend on identifiers, version properties, and custom entity information. Do not assume every save means a simple INSERT.

---

## 2. Entity mapping and lifecycle

### Q7. What is the owning side of a bidirectional relationship, and what does mappedBy mean?

The owning side is the side whose mapping controls the relationship update in the database. In a typical bidirectional one-to-many/many-to-one relationship, the many-to-one side owns the foreign-key mapping, while the one-to-many side uses `mappedBy` to refer to the field that owns it. Updating only the inverse side can leave the in-memory object graph looking correct while the expected relationship is not persisted.

Keep both sides of a bidirectional relationship consistent in application code through helper methods where appropriate. The exact owner depends on the mapping, not on which side is called “parent” in the business domain.

### Q8. What is the difference between cascade and orphanRemoval?

Cascade propagates selected persistence operations—such as persist, merge, or remove—from one entity to related entities. `orphanRemoval = true` expresses that a privately owned child should be deleted when it is removed from the relationship, subject to the mapping's semantics.

Neither is a generic “save everything related” switch. Applying cascade remove to shared reference data can delete objects that should survive, and orphan removal is suitable only when the child is truly owned by the parent. Choose cascade options based on the aggregate/lifecycle relationship, and understand how collection changes are detected.

### Q9. How should equals() and hashCode() be implemented for entities?

Entity equality is difficult because entity instances can move between transient, managed, detached, and proxy states, and generated identifiers may not exist at construction time. A naive implementation based on a database-generated ID can change its hash code after insertion into a HashSet, making the entity hard to find.

Choose a deliberate policy that fits the entity model and provider. A stable natural/business key can work if it is truly immutable and unique; generated-ID strategies need careful handling of unsaved instances and proxy classes. Avoid blindly including every mutable association or field in equality, because that can trigger lazy loads, recurse through relationships, or make equality change over time.

### Q10. Why are lazy and eager fetching important?

Fetch strategy controls when related data is loaded. Lazy fetching defers loading until the relationship is accessed, while eager fetching expresses that the provider should fetch the association eagerly according to the persistence specification and provider behavior. Eager does not guarantee one SQL join; the provider may issue additional queries.

Lazy loading can reduce unnecessary work but may trigger extra SQL or fail when accessed after the persistence context is closed. Eager loading can load more data than a use case needs. Prefer defining the required fetch plan for the specific use case—through fetch joins, entity graphs, projections, or queries—rather than making every relationship eager to hide a symptom.

### Q11. What causes LazyInitializationException?

In Hibernate, this exception commonly occurs when code accesses an uninitialized lazy association after the persistence context/session needed to load it is no longer available. The exception reveals a mismatch between the data needed by the use case and the lifetime of the persistence context.

The durable fix is usually to load the required data within the intended transaction using a fetch join, entity graph, explicit query, or DTO projection. Relying on Open Session in View or changing everything to eager fetching can hide the problem while introducing hidden queries, excessive data loading, or unexpected database access during serialization.

---

## 3. Fetching and query performance

### Q12. What is the N+1 query problem?

N+1 occurs when an initial query loads a set of parent entities and then accessing a relationship causes one additional query per parent. For example, one query loads 100 orders, and serialization accesses each order's customer, producing many extra queries depending on the mapping and cache state.

The result can be slow even when each individual query is fast. Diagnose it by inspecting generated SQL, query counts, traces, and the actual access path—not just by reading the entity mapping. It can arise with lazy associations and sometimes with eager mappings that are fetched through secondary selects.

### Q13. How do you fix N+1 queries?

Choose the fetch plan based on the data the use case actually needs. Options include a fetch join, an entity graph, a DTO/projection query, batch fetching, or restructuring the query to load related data in a small number of calls. The best option depends on cardinality, pagination, selected columns, and how the data is used.

A fetch join is not universally safe: joining multiple to-many associations can multiply result rows and increase memory use. Batch fetching can reduce round trips without creating one enormous joined result. Verify the fix by checking SQL count, row counts, response latency, and memory under representative data sizes.

### Q14. Why can collection fetch joins be problematic with pagination?

A to-many fetch join can produce multiple SQL rows for one root entity. Applying row-level pagination to those joined rows may not correspond to pagination over distinct root entities. Depending on provider and query, the ORM may retrieve more rows and paginate in memory, or warn/fail when configured to reject that behavior.

For a paginated endpoint, a common approach is to page root IDs first, then fetch the required relationships for those IDs in a second query, preserving order as needed. DTO projections or a different query shape can also help. Always verify generated SQL and provider warnings for the exact Hibernate version.

### Q15. What is the difference between JPQL, Criteria API, and native SQL?

JPQL queries the entity model and its mapped attributes rather than directly naming database tables and columns. Criteria API builds a query using a typed programmatic API, which can help compose dynamic predicates but may be verbose. Native SQL gives direct control over database-specific syntax and can be useful for advanced features, reports, or carefully optimized operations.

The trade-off is portability and abstraction versus control. JPQL is not simply SQL with different names; it follows persistence query semantics. Native SQL can introduce database-specific coupling and may require explicit result mapping. Choose based on query complexity, provider/database capabilities, maintainability, and measured performance.

### Q16. What are projections, and when are they better than loading entities?

A projection retrieves only the fields required by a use case, often into an interface projection, DTO, or constructor result. This can reduce transferred columns, object creation, persistence-context overhead, and accidental traversal of large entity graphs.

Entities are useful when the use case needs identity, relationships, and managed updates. Read-only endpoints and reporting often benefit from DTOs. Do not assume a projection always produces efficient SQL; inspect the actual query and joins. Also distinguish a projection from an entity: a DTO does not automatically participate in dirty checking.

---

## 4. Transactions and concurrency

### Q17. How does Spring @Transactional interact with JPA?

Spring typically starts or joins a transaction through a transaction manager, while the JPA provider manages the persistence context and flushes changes according to the transaction lifecycle. A method can modify managed entities without issuing SQL at each assignment; changes are synchronized later during flush.

The transaction annotation does not make every operation atomic across databases, HTTP calls, or message brokers. Proxy interception, propagation, rollback rules, and the correct transaction manager all matter. The persistence context's lifetime and the transaction's lifetime are related but conceptually distinct.

### Q18. What is the difference between optimistic and pessimistic locking?

Optimistic locking assumes conflicts are relatively uncommon and detects competing updates, commonly with a version field annotated `@Version`. An update includes the expected version; if another transaction has already changed the row, the update affects no matching version and the provider reports a concurrency conflict.

Pessimistic locking asks the database to acquire a lock to restrict conflicting access for a period. It can be appropriate when conflicts are frequent and waiting is acceptable, but it can reduce concurrency and cause lock waits or deadlocks. Choose based on contention, transaction duration, database semantics, and the business response to conflicts; neither strategy eliminates the need for retry or conflict handling.

### Q19. How does @Version prevent lost updates?

A version property lets the provider include the expected version in an update or delete. If another transaction modifies the same row first, the version no longer matches, so the second update fails rather than silently overwriting the first update. The application can then report a conflict, reload and re-evaluate, or retry if the operation is safe.

Optimistic locking detects a conflict; it does not automatically merge two users' business changes. The transaction boundary and retry policy matter. A retry should not blindly reapply a non-idempotent operation or ignore the user's intent.

### Q20. What are the risks of bulk JPQL update and delete operations?

Bulk update/delete queries operate directly on database rows rather than loading and updating each entity through the normal managed-entity lifecycle. They can be efficient for large sets, but they typically bypass per-entity dirty checking and may bypass lifecycle callbacks and cascade semantics. Managed entities already in the persistence context can become stale after the bulk operation.

Flush pending changes before a bulk query when needed, then clear or refresh the persistence context appropriately to avoid stale in-memory state. Review second-level cache behavior and transaction semantics for the provider/version. Bulk operations are useful, but they are not interchangeable with iterating over managed entities and modifying them one by one.

### Q21. What causes a transaction to roll back, and what should be handled by the service?

Rollback behavior depends on the transaction manager, framework configuration, and exception rules. Spring's common declarative default rolls back on unchecked exceptions and `Error`, while checked exceptions do not trigger rollback by default unless configured. The persistence provider can also mark a transaction rollback-only after certain failures.

The service should define the business operation's consistency boundary, propagate or translate failures deliberately, and avoid catching an exception then continuing as if the transaction were healthy. After a persistence exception, the current transaction may no longer be usable. Retrying should generally occur around a new transaction and only when the operation is safe to repeat.

---

## 5. Queries and Spring Data JPA

### Q22. How does Spring Data JPA decide whether save() should insert or update?

Spring Data JPA uses entity-new detection to decide whether to call persist or merge. The default strategy can consider a non-primitive version property and then the identifier; custom entity information can change the decision. This means the exact behavior depends on how the entity models its ID/version and on repository configuration.

An existing detached entity may lead to merge, which returns a managed instance. A new entity may be persisted. Do not infer the SQL solely from the method name. Inspect entity state, ID/version strategy, provider SQL, and the exact Spring Data JPA version when a save unexpectedly issues a SELECT or UPDATE.

### Q23. Why can save() issue a SELECT before an UPDATE?

When Spring Data JPA uses merge for an entity it considers not new, the provider may need to locate or load the managed instance and reconcile state before updating. The exact SQL sequence depends on identifier generation, versioning, entity state, provider version, and mapping.

A SELECT is not automatically a bug. First determine whether the application is passing a detached entity, whether new-state detection is correct, and whether merge is appropriate. Do not override persistence behavior blindly to remove a query; verify the resulting INSERT/UPDATE semantics, callbacks, optimistic locking, and generated identifiers.

### Q24. What is the difference between Page and Slice?

A Spring Data `Page` provides pagination metadata including total-element/page information, which commonly requires an additional count query. A `Slice` focuses on whether more data is available and can avoid calculating the total count. Exact query behavior depends on repository method and query shape.

For large tables or frequently changing data, total counts can be expensive or not particularly useful. Choose the return type based on what the API needs. If the client only needs “load more,” a Slice or cursor-based approach may be more efficient than returning a total count on every request.

### Q25. What is the difference between derived queries, JPQL, and native queries in Spring Data JPA?

Derived queries express common predicates through repository method names. JPQL provides a declared query over entity types and attributes. Native queries use database SQL and are useful for vendor-specific features or specialized query tuning.

Use derived methods for straightforward operations, but avoid excessively long method names that hide query intent. JPQL is often easier to maintain for domain-oriented queries; native SQL can be necessary for database-specific optimization. For any nontrivial query, inspect the generated SQL and query plan, and verify pagination/count-query behavior.

### Q26. How should pagination and sorting be used safely with Spring Data JPA?

Use a bounded page size and stable sorting with a unique tie-breaker. Without deterministic ordering, records can move between pages even when the query appears unchanged. Offset pagination can become costly at high offsets, while keyset/cursor pagination can be more efficient for sequential traversal if the query has a suitable index and stable key.

Do not expose arbitrary client-supplied property names without validation. Be cautious when combining pagination with collection fetch joins. For high-volume endpoints, consider projections, cursor pagination, and an index aligned with filtering and ordering.

---

## 6. Caching and production troubleshooting

### Q27. What is the difference between first-level and second-level cache?

The first-level cache is the persistence context's identity map and is normally active for managed entities within that context. It avoids repeatedly loading the same persistent identity into multiple managed instances within that context. The second-level cache is an optional shared cache associated with the persistence unit/provider and can retain entity or collection data across persistence contexts.

Second-level caching is provider/configuration-dependent and introduces invalidation and consistency concerns. It is not automatically beneficial for frequently changing data. Query caching, entity caching, and the persistence context are distinct mechanisms; check cache regions, concurrency strategy, invalidation behavior, and actual hit rates before enabling it broadly.

### Q28. How do you investigate a slow Hibernate endpoint?

Start by separating database time from application processing and network time. Capture SQL statements and timings, count queries, inspect execution plans, review rows returned, and correlate the endpoint trace with connection-pool wait time. N+1 queries, missing indexes, excessive joins, over-fetching, large result sets, lock waits, and slow flushes can all contribute.

Then compare the query shape to the use case: does the endpoint need entities or a projection? Is pagination stable? Are associations fetched deliberately? Is the persistence context accumulating too many entities? Optimize based on measurements and validate with realistic data volume rather than simply adding fetch joins or increasing the connection pool.

### Q29. What causes connection-pool exhaustion in a JPA application?

Requests may hold connections too long because transactions are long-running, queries are slow, locks are contended, remote calls happen inside transactions, or concurrency exceeds the pool's capacity. A leak or a blocked transaction can also prevent connections from returning to the pool. Increasing the pool size can overload the database and worsen queueing.

Inspect active/idle/pending connection metrics, acquisition timeouts, transaction duration, slow SQL, database sessions, and thread dumps. Keep transactions short, set sensible timeouts, avoid remote waits while holding database resources, and size the pool with the database's actual connection capacity in mind.

### Q30. What should a production scenario answer include when ORM performance degrades?

A strong investigation has a sequence: establish the symptom and baseline, gather query counts and timings, identify the slow layer, form hypotheses, test the most likely cause, apply a reversible mitigation, and verify the outcome. Include both immediate mitigation and long-term prevention.

For example, if latency increases after a release, compare query count and execution plans before and after, check whether a mapping or serialization change introduced lazy loads, and inspect connection-pool wait time. Avoid conclusions such as “Hibernate is slow” without evidence. The ORM is one part of the path; database plans, network behavior, transactions, object mapping, and response serialization all matter.

---

## 6A. Mapping and write performance

### Q31. What are the common JPA inheritance mapping strategies?

JPA supports several ways to map entity inheritance. With **single-table** mapping, a hierarchy shares one table and a discriminator identifies the concrete type; reads can be simple, but many subtype-specific columns may be nullable. With **joined** mapping, base and subtype data live in separate related tables, which normalizes the schema but can require joins. With **table-per-class** mapping, each concrete class has its own table containing inherited state, which can complicate polymorphic queries and may require unions.

No strategy is best for every model. Consider query patterns, hierarchy size, schema constraints, write/read balance, and generated SQL. Verify provider behavior and database plans for the actual mapping rather than choosing based only on annotation count.

### Q32. How can JDBC batching improve Hibernate write performance?

Without batching, a large number of inserts or updates can cause many database round trips. JDBC batching groups compatible statements so the driver can send multiple operations together, reducing network overhead. Hibernate batching must be configured and the SQL/identifier strategy must allow statements to be batched; statement ordering and flush boundaries can affect results.

Validate batching by inspecting SQL logs, JDBC metrics, and database round trips under realistic volume. Some identifier-generation strategies can inhibit insert batching, and mixing different statement shapes may reduce batching effectiveness. Large batch operations also need a persistence-context strategy—periodic flush/clear may control memory growth—while preserving transaction and error semantics.

### Q33. How do entity identifier-generation strategies affect persistence behavior?

Identifier strategies determine how primary keys are allocated. With identity generation, the database typically generates the key during insertion, which can constrain when the INSERT must occur and may limit insert batching. Sequence-based strategies can allocate identifiers before inserting rows and can support allocation optimizations when the database/provider supports them. Table-based or custom strategies have their own concurrency and performance trade-offs.

The right choice depends on the database, portability requirements, write volume, schema, and provider behavior. Do not assume `GenerationType.AUTO` means the same physical strategy on every database. Check the generated DDL and SQL for the target provider and understand how identifier generation interacts with batching and transaction timing.

---


## 7. Mapping details and persistence operations

### Q34. What is the difference between find(), getReference(), persist(), merge(), remove(), detach(), and refresh()?

These `EntityManager` operations express different intentions. `find()` loads an entity by identity and returns `null` when no row exists. `getReference()` may return a reference whose state is loaded only when needed; it is useful when an association needs an entity reference by ID, but it is not a guarantee that no SQL or exception will occur before later access. `persist()` makes a new instance managed. `merge()` copies state into a managed instance and returns that instance. `remove()` marks a managed entity for deletion. `detach()` stops the context from tracking an entity, while `refresh()` reloads database state into a managed instance and overwrites its in-memory changes.

The key interview point is to reason about entity state and the persistence context, not treat these methods as interchangeable CRUD shortcuts. Refreshing an entity with unsaved edits discards those edits; detaching it means later field changes are not automatically persisted. Exact SQL timing depends on provider behavior, identifier strategy, flush mode, and transaction state.

### Q35. How do identifier generation, embedded values, and composite keys affect mapping?

`@GeneratedValue` delegates identifier generation to a strategy chosen explicitly or through `AUTO`. `IDENTITY` commonly requires the insert to obtain the generated key and can limit insert batching. `SEQUENCE` can allocate identifiers before inserts and may benefit from allocation pooling when supported by the database and provider. `TABLE` uses a table-based generator and can add coordination overhead. The physical behavior of `AUTO` is provider- and database-dependent, so inspect the generated schema and SQL rather than assuming it selects the same strategy everywhere.

Use `@Embeddable` for a value object whose fields are stored as part of an owning entity's table, and `@Embedded` to include it. Use `@EmbeddedId` or `@IdClass` for a composite key when the domain/schema genuinely requires multiple key columns. Composite key classes must satisfy the requirements of the target Jakarta Persistence version and be designed with stable equality and hashing. Avoid choosing a composite key solely to represent a relationship if a simpler surrogate key plus a unique constraint expresses the model more clearly.

### Q36. How do you keep a bidirectional relationship consistent, and when should many-to-many be replaced with an association entity?

The owning side controls the foreign-key or join-table update. In a `@OneToMany`/`@ManyToOne` mapping, the child-side `@ManyToOne` commonly owns the foreign key. Updating only the parent's inverse collection may change the Java object graph without changing the relationship stored in the database. Helper methods should update both sides so in-memory navigation and persistence mapping agree.

`@ManyToMany` is suitable only when the join itself has no meaningful attributes or lifecycle. If the relationship needs fields such as role, quantity, effective date, status, audit data, or its own uniqueness rules, model the join table as an entity with two many-to-one associations. That makes constraints and lifecycle explicit and avoids uncontrolled cascades across shared entities.

```java
public void addLine(OrderLine line) {
    lines.add(line);
    line.setOrder(this); // owning side
}
```

### Q37. What are field access, property access, lifecycle callbacks, and database constraints?

JPA access strategy determines whether the provider reads and writes persistent state through fields or JavaBean properties. It is normally inferred from where mapping annotations such as `@Id` are placed, unless access is explicitly configured. Be consistent within an entity hierarchy and do not put persistence annotations on one access style while assuming the provider will use the other. Property accessors should not contain surprising business logic because the provider may call them during persistence operations.

Lifecycle callbacks such as `@PrePersist`, `@PreUpdate`, and `@PostLoad` can support small entity lifecycle concerns, but they are not a substitute for service-level orchestration or reliable database constraints. Callback timing and which operations trigger callbacks depend on the persistence operation; bulk JPQL updates and deletes bypass normal per-entity lifecycle processing. Likewise, `nullable = false` and `unique = true` contribute mapping/schema metadata, but runtime integrity ultimately depends on the actual database schema and constraints. Validate migrations and constraints in the target database.


## 8. Repository design, dynamic queries, and batching

### Q38. When should you use derived queries, JPQL, Specifications, Criteria, or native SQL?

Use derived query methods for straightforward predicates whose intent remains readable in the method name. Use `@Query` with JPQL when the query is clearer when written explicitly or needs joins, aggregation, or a DTO result. Specifications or Criteria are useful when a search screen combines optional filters dynamically; they avoid creating a separate repository method for every filter combination. Native SQL is justified when a database-specific feature, execution plan, or SQL construct is materially useful and the maintenance cost is understood.

For optional filters, compose predicates only when supplied rather than encoding every combination into an enormous method name. Keep values parameterized; never concatenate user-provided values into JPQL or SQL. Test against realistic data and inspect generated SQL and the execution plan. Dynamic query abstractions improve composition, not the database's underlying cost.

### Q39. Does saveAll() perform one bulk SQL insert, and what is the difference between JpaRepository and other repository interfaces?

No. `saveAll()` iterates over entities through repository save semantics; it does not promise one SQL statement or JDBC batching. Actual round trips depend on Hibernate batching configuration, compatible SQL shapes, identifier generation, flush boundaries, transaction size, and the JDBC driver. For large imports, measure batch size and memory use, consider periodic `flush()` and `clear()`, and evaluate plain JDBC or database bulk-loading tools when ORM lifecycle behavior is unnecessary.

`CrudRepository` provides common CRUD operations, `ListCrudRepository` refines several multi-entity return types to `List` in Spring Data versions that include it, and `JpaRepository` adds JPA-oriented operations such as flushing and batch deletion methods. Available interfaces and signatures are version-dependent. The repository abstraction does not itself guarantee a database bulk operation; inspect the exact Spring Data JPA version and SQL behavior.

### Q40. How should bulk modifying queries be used safely?

A JPQL bulk update changes matching database rows directly and does not synchronize each affected managed entity with the persistence context. That makes it efficient, but already-loaded entities may continue to expose old values. Bulk statements also do not behave like iterating over entities: per-entity callbacks, cascades, and ordinary dirty checking are not applied in the same way, and version handling needs deliberate attention.

In Spring Data JPA, a modifying query is commonly declared with `@Modifying` alongside `@Query`. Decide whether pending changes must be flushed first and whether the context should be cleared afterward; `clearAutomatically = true` can clear it after execution, while `flushAutomatically = true` can flush pending changes before it. These flags are not universal defaults and clearing detaches all managed entities in that context, so use them only when callers understand the consequence.

```java
@Modifying(flushAutomatically = true, clearAutomatically = true)
@Query("update Order o set o.status = :status where o.id = :id")
int updateStatus(@Param("id") long id, @Param("status") OrderStatus status);
```

### Q41. How do auditing, custom repositories, and transaction boundaries fit together?

Spring Data JPA auditing can populate fields such as `@CreatedDate`, `@LastModifiedDate`, `@CreatedBy`, and `@LastModifiedBy` when auditing is enabled and the entity/listener configuration is correct. Auditing records application-level metadata; it does not replace database constraints, immutable audit history, or a separate audit event stream when regulatory requirements demand one.

Use a custom repository implementation when a query or persistence operation cannot be expressed cleanly with derived methods, `@Query`, Specifications, or projections. Keep transaction boundaries around business operations at the service layer when multiple repository calls must succeed or fail together. Repository methods have transactional behavior depending on the method and configuration, but several individually transactional repository calls are not automatically one atomic business transaction unless they participate in the same surrounding transaction.


## 9. Testing and production diagnosis

### Q42. What is the difference between Spring transaction propagation and database isolation?

Propagation describes how Spring handles a method's transaction relative to an existing transaction—for example, joining it with `REQUIRED` or requesting an independent transaction with `REQUIRES_NEW` where the transaction manager supports it. Isolation describes which concurrent database changes a transaction may observe and which anomalies the database prevents. These are separate dimensions: propagation is a Spring transaction-management concern, while isolation semantics are ultimately implemented by the database and transaction manager.

A propagation setting does not guarantee a particular isolation level, and an isolation annotation cannot fix an incorrectly designed business consistency boundary. Verify proxy interception, self-invocation, transaction manager selection, rollback rules, connection behavior, and the target database's isolation semantics. Keep transactions short, and avoid holding a connection while waiting for remote HTTP or Kafka calls.

### Q43. How should JPA repositories be tested without trusting an in-memory database too much?

`@DataJpaTest` is a Spring Boot test slice for JPA components. It usually configures entity scanning, repositories, and a test database, and test transactions are commonly rolled back by default. It is useful for mapping and repository behavior, but an in-memory database can differ from production in SQL dialect, constraints, locking, collation, sequence behavior, JSON types, and query plans.

For behavior that depends on the production engine, use Testcontainers or another representative database environment. Test not only that a repository returns expected objects, but also constraints, pagination, generated SQL where important, optimistic-lock conflicts, and bulk-operation context behavior. A passing repository test does not prove a query will perform well with production cardinality or indexes.

### Q44. How do you diagnose unexpected SELECTs, flushes, and stale entity state?

Start with the exact operation and transaction boundary, then capture SQL, bind values where safe, statement timings, query counts, and Hibernate statistics if available. Identify whether a SELECT comes from entity loading, `merge()` state reconciliation, eager association fetching, lazy initialization, a count query, or application code. Identify whether an UPDATE appears at an explicit flush, an automatic flush before a query, or transaction completion.

For stale state, ask whether a bulk update changed rows behind the persistence context, whether a second-level cache is enabled, and whether the application is reading from the same transaction/consistency view. Use `refresh()` for a specific managed entity only when overwriting its current in-memory state is intended; otherwise reload in a suitable context. Do not disable flushing or caching globally before understanding the mechanism.

### Q45. What is a safe approach to batch processing thousands or millions of entities?

First decide whether the work requires entity lifecycle semantics. If yes, use bounded transaction sizes where the business operation permits, configure JDBC batching, and periodically flush and clear the persistence context so it does not retain every processed entity. `flush()` sends pending changes to the database; `clear()` detaches all managed entities, so references held by the application must not be assumed to remain managed afterward. Choose batch size based on memory, driver behavior, lock duration, rollback cost, and database capacity.

If the operation is a set-based transformation, bulk JPQL or SQL can be much cheaper than loading every row. For very large imports, database-native bulk-loading facilities or plain JDBC may be more appropriate. Measure rows per second, actual round trips, memory, lock waits, and transaction failure/restart behavior. A single enormous transaction can consume connections for too long and make rollback expensive; many tiny transactions can increase overhead and complicate consistency.


## 10. Senior-level production scenarios

### Q46. An endpoint slows down as the number of returned entities grows. How do you prove or rule out N+1?

Capture a representative trace and compare SQL statement count with the number of root records returned. If one query loads the roots and repeated similar queries load an association for each root, N+1 is likely. Confirm which code path triggers association access—mapping, serialization, logging, or business logic—and check whether second-level caching is masking the query count in development.

Then choose a use-case-specific fetch plan: a fetch join for suitable cardinality, an entity graph, batch fetching, or a DTO projection. For pagination, avoid blindly joining multiple collections because row multiplication can break page semantics or inflate memory. Verify statement count and total rows/bytes, execution plans, latency, and memory using realistic data; fewer statements alone does not guarantee less work.

### Q47. A paginated endpoint returns duplicates, misses records, or becomes slow on later pages. What do you investigate?

Check whether sorting is deterministic and includes a unique tie-breaker. Ordering only by a non-unique field can move equal-valued rows between pages, especially while concurrent writes occur. Inspect join cardinality, particularly collection fetch joins, and verify whether pagination is applied to root entities or joined SQL rows. Also inspect count-query correctness and whether the count query duplicates roots.

For large offsets, the database may still scan and discard many rows; keyset/seek pagination can be more efficient when the API's navigation requirements allow it. A two-query strategy—page root IDs, then fetch required details—can avoid collection-fetch pagination problems. Validate results against concurrent updates and the actual database query plan rather than assuming `Pageable` guarantees correct semantics for every query shape.

### Q48. An update fails with optimistic-lock errors under load. Should you automatically retry?

First identify whether the conflict is expected contention or a symptom of stale detached state, long transactions, duplicate processing, or an incorrect version mapping. `@Version` prevents silent lost updates; the exception is a signal that the application's view was no longer current. Capture entity IDs, versions, transaction duration, retry count, and the competing write path without logging sensitive payloads.

If the operation can be safely retried, retry the entire business operation in a new transaction: reload current state, re-evaluate business rules, and apply the intended change. Use bounded retries with backoff and jitter, and consider idempotency keys where requests or messages can be repeated. Do not blindly retry non-idempotent commands such as decrementing inventory or charging a payment. If contention is normal and high, consider a different aggregate boundary or carefully scoped pessimistic locking.

### Q49. A database update succeeds but the service later rolls back, or a database write and Kafka publish can diverge. How do you respond?

SQL execution is not the same as transaction commit. A flush may send statements to the database while the transaction remains uncommitted; a later exception, constraint failure, timeout, or rollback-only marker can cause the transaction to roll back. Check transaction logs, exception paths, propagation, rollback rules, connection/transaction manager, and the commit outcome rather than treating SQL logs as proof of durable success.

For a database-plus-Kafka dual write, a database transaction and a Kafka publish are not automatically atomic together. Publishing before commit can emit an event for a transaction that later rolls back; publishing after commit can fail and leave the database updated without an event. The transactional-outbox pattern writes the business change and an outbox row in the same database transaction, then a separate publisher or CDC pipeline delivers the event with idempotent downstream handling. The outbox improves atomic recording of intent, but delivery, duplicates, ordering, and operational recovery still need design. See the Kafka handbook for messaging delivery semantics.

### Q50. A cascade unexpectedly deletes data, or a relationship change is missing from the database. How do you debug it?

Inspect the mapping for `CascadeType.REMOVE`, `CascadeType.ALL`, `orphanRemoval = true`, and the owning side. Determine whether the related object is truly privately owned or is shared reference data. For a missing relationship, check whether the owning side was changed and whether both sides of a bidirectional association remain consistent. For unexpected deletion, reproduce the exact entity operation and inspect SQL and foreign-key behavior.

Correct the aggregate/lifecycle model rather than merely removing all cascades. Prefer explicit association entities for relationships with independent attributes or lifecycle, and enforce important integrity rules in database constraints. Add integration tests for removal and reassignment paths so future mapping changes cannot silently broaden deletion behavior.

## 11. Last-minute revision

Prioritize these questions first:

1. JPA vs Hibernate vs Spring Data JPA (Q1)
2. Persistence context, lifecycle states, dirty checking, flush vs commit (Q2–Q5)
3. persist vs merge and owning side/mappedBy (Q6–Q7)
4. Cascade vs orphanRemoval and entity equality (Q8–Q9)
5. Lazy loading, LazyInitializationException, and N+1 (Q10–Q13)
6. Collection fetch joins with pagination and DTO projections (Q14–Q16)
7. Transactions, optimistic locking, @Version, and bulk operations (Q17–Q21)
8. save() semantics and unexpected SELECTs (Q22–Q23)
9. Page vs Slice and query selection (Q24–Q26)
10. First/second-level cache, slow query diagnosis, and connection-pool exhaustion (Q27–Q30)

11. EntityManager operations, identifier strategies, embedded values, relationship ownership, and callbacks (Q34–Q37)
12. Derived queries versus JPQL/Specifications/native SQL, repository save semantics, modifying queries, auditing, and custom repositories (Q38–Q41)
13. Propagation versus isolation, realistic JPA integration tests, SQL/flush diagnosis, and batch processing (Q42–Q45)
14. N+1 investigation, stable pagination, optimistic-lock retry design, transaction/outbox failures, and mapping safety (Q46–Q50)

For each, explain provider behavior, the generated SQL or transaction consequence, the trade-off, and how to diagnose it in production.

---

## 12. Official references and version notes

- [Jakarta Persistence specifications](https://jakarta.ee/specifications/persistence/)
- [Hibernate ORM User Guide](https://docs.jboss.org/hibernate/orm/current/userguide/html_single/Hibernate_User_Guide.html)
- [Hibernate ORM documentation and releases](https://hibernate.org/orm/documentation/)
- [Spring Data JPA reference](https://docs.spring.io/spring-data/jpa/reference/)
- [Spring Framework transaction management](https://docs.spring.io/spring-framework/reference/data-access/transaction.html)
- [Spring Boot reference](https://docs.spring.io/spring-boot/reference/)

The Jakarta Persistence specification is the authority for portable persistence behavior. Hibernate documentation covers provider-specific features and implementation behavior. Spring Data documentation defines repository-layer behavior. Do not present draft or milestone specification features as stable contracts.

### Version and compatibility notes

- The namespace changed from `javax.persistence.*` to `jakarta.persistence.*` with Jakarta Persistence 3.0. This is a source and binary compatibility boundary; update imports and compatible dependencies together.
- Hibernate ORM 5.x is associated with the older Java Persistence namespace in common deployments; Hibernate ORM 6.x moved to Jakarta Persistence 3.x and made significant SQL, query, and type-system changes. Hibernate ORM 7.x continues the Jakarta Persistence 3.2 generation with a newer Java baseline. Check the official release compatibility matrix and migration guide for the exact minor version.
- Spring Boot manages a tested dependency set. Prefer its dependency management rather than independently overriding Hibernate, Spring Data JPA, Jakarta Persistence APIs, and the JDBC driver without a compatibility reason.
- Hibernate's supported series and compatibility matrix change over time. Use the official matrix and version-pinned migration guides as the source of truth for supported series, Java baselines, Jakarta Persistence levels, and Spring Boot compatibility. Treat `current` documentation as a moving target.

## 13. Editorial and validation notes

- This guide contains 50 curated questions; regenerate the count from actual question headings after future edits.
- Portable Jakarta Persistence behavior, Hibernate-specific behavior, and Spring Data behavior are distinguished where the difference matters.
- Code is intentionally limited to small illustrative snippets; the focus is interview understanding and production reasoning.
- Exact SQL and performance depend on mappings, provider version, database, driver, and query plan; verify with logs and representative workloads.
- The primary handbook is the consolidated revision path. Legacy files should only be removed after their unique material has been inspected and preserved.
