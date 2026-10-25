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
7. [Last-minute revision](#7-last-minute-revision)
8. [Official references](#8-official-references)

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

## 7. Last-minute revision

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

For each, explain the provider behavior, the generated SQL or transaction consequence, the trade-off, and the way to diagnose it in production.

---

## 8. Official references

- [Jakarta Persistence specifications](https://jakarta.ee/specifications/persistence/)
- [Hibernate ORM User Guide](https://docs.jboss.org/hibernate/orm/current/userguide/html_single/Hibernate_User_Guide.html)
- [Hibernate ORM documentation and releases](https://hibernate.org/orm/documentation/)
- [Spring Data JPA reference](https://docs.spring.io/spring-data/jpa/reference/)
- [Spring Framework transaction management](https://docs.spring.io/spring-framework/reference/data-access/transaction.html)
- [Spring Boot reference](https://docs.spring.io/spring-boot/reference/)

The Jakarta Persistence specification is the authority for portable persistence behavior. Hibernate documentation covers provider-specific features and implementation behavior. Spring Data documentation defines repository-layer behavior. Do not present draft or milestone specification features as stable contracts.

## Editorial and validation notes

- This guide prioritizes a manageable set of questions with complete explanations rather than maximizing count.
- Portable Jakarta Persistence behavior, Hibernate-specific behavior, and Spring Data behavior are distinguished where the difference matters.
- Code is intentionally limited to small illustrative snippets; the focus is interview understanding and production reasoning.
- Exact SQL and performance depend on mappings, provider version, database, driver, and query plan; verify with logs and representative workloads.
- This curated guide is the primary revision path. Existing chapter files remain until unique material is checked and consolidation is safe.
