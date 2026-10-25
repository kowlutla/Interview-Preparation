# Hibernate + JPA + Spring Data JPA Interview Q&A Handbook

A high-signal interview handbook for Java backend engineers with 4–5 years of experience. Answers are written to sound like a developer explaining concepts to an interviewer: direct explanation, mechanism, practical context, and relevant caveats. The goal is interview readiness, not memorization or maximum question count.

## How to use this handbook

- Master the Top 25 first, then use the chapters as a focused revision guide.
- Practise saying answers aloud in your own words. Treat the text as a model explanation, not a script to memorize.
- Use the mock interviews after the main chapters. Answer before reading the model response, then try the follow-up.
- Check the exact Spring Boot, Spring Data JPA, Hibernate ORM, Jakarta Persistence, JDBC driver, and database versions in your project before relying on version-sensitive behavior.

## Top 25 Questions to Master First

1. [JPA versus Hibernate](#top-01)
2. [JDBC vs JPA vs Hibernate vs Spring Data JPA](#top-02)
3. [Persistence context](#top-03)
4. [Entity lifecycle states](#top-04)
5. [Dirty checking](#top-05)
6. [Flush vs commit](#top-06)
7. [persist() vs merge()](#top-07)
8. [Owning side and mappedBy](#top-08)
9. [Cascade vs orphanRemoval](#top-09)
10. [Entity equals/hashCode](#top-10)
11. [N+1 query problem](#top-11)
12. [Fixing N+1 queries](#top-12)
13. [Fetch join limitations](#top-13)
14. [LazyInitializationException](#top-14)
15. [Collection fetch join and pagination](#top-15)
16. [Spring @Transactional with JPA](#top-16)
17. [Optimistic vs pessimistic locking](#top-17)
18. [@Version and lost updates](#top-18)
19. [Bulk update/delete implications](#top-19)
20. [JPQL vs SQL](#top-20)
21. [Page vs Slice](#top-21)
22. [Spring Data JPA save() semantics](#top-22)
23. [save() causing SELECT](#top-23)
24. [First-level vs second-level cache](#top-24)
25. [Systematic Hibernate performance incident response](#top-25)

## Table of Contents

- [Part 1. JPA and Hibernate Fundamentals](#part-1-jpa-and-hibernate-fundamentals)
- [Part 2. Entity Mapping and Relationships](#part-2-entity-mapping-and-relationships)
- [Part 3. Entity Lifecycle and Persistence Context](#part-3-entity-lifecycle-and-persistence-context)
- [Part 4. Fetching and Performance](#part-4-fetching-and-performance)
- [Part 5. Transactions and Concurrency](#part-5-transactions-and-concurrency)
- [Part 6. JPQL, Criteria, and Querying](#part-6-jpql-criteria-and-querying)
- [Part 7. Spring Data JPA](#part-7-spring-data-jpa)
- [Part 8. Caching and Optimization](#part-8-caching-and-optimization)
- [Part 9. Production Troubleshooting Scenarios](#part-9-production-troubleshooting-scenarios)
- [Part 10. Mock Interviews](#part-10-mock-interviews)
- [Part 11. Roadmap and Study Plan](#part-11-roadmap-and-study-plan)
- [Part 12. Official References and Version Notes](#part-12-official-references-and-version-notes)

## Answer-writing standard

Answers start with a direct, natural explanation and then explain the mechanism, practical impact, and important caveats. They should sound like something a backend engineer can confidently say in an interview. Straightforward topics stay concise; commonly tested and practical topics receive more depth. Avoid treating these answers as scripts—understand the ideas and explain them in your own words.


## Part 1. JPA and Hibernate Fundamentals

<a id="top-01"></a>
### Q1. What is the difference between JPA and Hibernate?

I would explain it this way: JPA, now called Jakarta Persistence, defines the standard APIs and rules we use to persist Java objects in a relational database. Hibernate is an ORM framework that implements that specification. So when I use annotations such as `@Entity` and APIs such as `EntityManager`, I'm generally working with the JPA contract, while Hibernate does the actual ORM work underneath.

The distinction matters when we want portability. Standard JPA code is easier to move between providers, but that doesn't guarantee identical SQL or performance. Hibernate also offers extra features that are not part of the JPA specification.

**Key distinction:** JPA defines the contract; Hibernate implements it.

<a id="top-02"></a>
### Q2. How do JDBC, JPA, Hibernate, and Spring Data JPA differ?

I think of these as different abstraction levels. With JDBC, I write SQL, manage parameters, and process result sets myself. JPA gives me entity mapping and a persistence context, so I can work with domain objects instead of manually mapping every row. Hibernate is a common JPA provider that implements those features and generates SQL. Spring Data JPA sits above JPA and reduces repository boilerplate with derived queries, pagination, projections, and repository interfaces.

For a normal business service, Spring Data JPA is convenient. If I need precise control over a specialized query or bulk operation, I may use JPQL, native SQL, or JDBC. The abstraction should simplify the code without hiding a performance or correctness problem.

### Q3. What are EntityManager and Hibernate Session?

In application code, I usually use JPA's `EntityManager` because it gives me a standard way to find, persist, remove, and query entities. Hibernate's `Session` is its native counterpart and exposes additional Hibernate-specific capabilities. When I need one of those features, I can unwrap the Hibernate Session from the EntityManager.

One important point is that these objects are not general-purpose thread-safe objects. I would not keep one in a singleton service and share it across concurrent requests. In a Spring application, I normally let Spring manage the persistence context and transaction integration.

### Q4. What are EntityManagerFactory and SessionFactory?

The factory creates EntityManager or Session instances. Initializing it involves loading mappings and setting up provider infrastructure, so I would normally create it once for a persistence unit and reuse it rather than create one per request. Hibernate's native equivalent is SessionFactory.

In Spring Boot, the framework usually configures and manages the factory for me. If startup fails because of a mapping or dialect issue, factory initialization is one place where those problems can surface.

<a id="top-03"></a>
### Q5. What is a persistence context?

I describe the persistence context as Hibernate's managed workspace for entity instances. It keeps track of the entities I'm working with, their persistent identities, and changes that need to be synchronized with the database. If I load an entity and change one of its fields, Hibernate can detect that change because the entity is managed by the context.

It also provides identity consistency: within the same persistence context, loading the same persistent identity normally gives me the same managed Java instance. The context is not itself the database transaction, although Spring applications commonly associate it with a transaction.

<a id="top-04"></a>
### Q6. Can you explain the four entity lifecycle states?

I usually explain the states by asking whether the persistence context is tracking the object.

- **Transient/new:** I have created the Java object, but it is not managed and has not been made persistent.
- **Managed:** the persistence context tracks it, so Hibernate can detect changes.
- **Detached:** it may represent a database row, but the current context no longer tracks it.
- **Removed:** it is managed but scheduled for deletion when changes are flushed.

For example, an entity can become detached when I clear the EntityManager or close the persistence context. Changing that detached object is only an in-memory change; it does not automatically update the database.

### Q7. How does Hibernate turn entity operations into SQL?

When I work with entities, Hibernate uses the mappings and persistence-context state to determine which database operations are needed. It may queue inserts, updates, and deletes rather than execute SQL at the exact moment I call a Java method. At flush, it synchronizes those changes by generating SQL and executing it through JDBC.

The exact statements depend on the mapping, identifier-generation strategy, dialect, and Hibernate version. When investigating an issue, I check the generated SQL rather than assuming that a repository method maps to one specific statement.

<a id="top-05"></a>
### Q8. What is dirty checking, and why is it useful?

Dirty checking is how Hibernate detects changes to managed entities and schedules the corresponding database updates without requiring me to call an update method explicitly.

For example, suppose I'm developing an order service. I load an order inside a transaction, change its status from `PENDING` to `CONFIRMED`, and let the transaction complete. Because the order is managed, Hibernate detects the changed state and generates the necessary UPDATE during flush.

The important nuance is that the SQL may not execute immediately when I call the setter. It is generally executed during flush, and the transaction can still roll back afterward. Dirty checking also applies to managed entities; changing a detached object alone will not update the database.

<a id="top-06"></a>
### Q9. What is the difference between flush and commit?

I separate these concepts because they solve different problems. Flush synchronizes pending persistence-context changes with the database by executing SQL. Commit completes the database transaction. A flush does not make the changes durable by itself, because the transaction can still roll back.

For example, a uniqueness violation may be reported during flush rather than when I set a field. Hibernate may also flush automatically before certain queries or before transaction completion. When debugging, I check both the flush point and the transaction outcome.

### Q10. Does every EntityManager operation execute SQL immediately?

No. If I call `persist()`, Hibernate may make the entity managed and delay the INSERT until flush. A call to `remove()` may schedule a DELETE. A change to a managed entity can also wait until flush. On the other hand, identifier generation and provider details can make some operations execute SQL earlier.

I would not infer SQL timing from the method name alone. I enable SQL logging in a safe environment and observe the actual statements and their timing.

### Q11. Does using JPA make an application database- and provider-independent?

It improves portability, but I wouldn't promise complete independence. JPA standardizes APIs and persistence behavior, while SQL generation, query plans, locking details, and some mapping behavior can vary. Native SQL and provider-specific extensions introduce additional coupling.

If portability is a requirement, I keep provider-specific code isolated and test against the actual target database. I also measure performance on that database, because portable code is not necessarily equally efficient everywhere.

### Q12. How do Spring transactions and the persistence context work together?

In a typical Spring application, Spring's transaction infrastructure starts or joins a transaction and its JPA integration provides an EntityManager associated with the current transactional execution. Entities loaded or persisted in that context are managed, so their changes can be synchronized when the transaction flushes.

I also watch for Spring's proxy behavior: calling a transactional method from another method on the same object may bypass the proxy and therefore not apply the expected transaction advice. If an update unexpectedly doesn't happen, I check transaction boundaries, managed state, and the generated SQL.


## Part 2. Entity Mapping and Relationships

### Q13. What makes a class a JPA entity?

I use `@Entity` to tell the persistence provider that a class represents persistent data. The entity must satisfy the requirements of the Jakarta Persistence version in use, including having an identifier, a public or protected no-argument constructor under the standard rules, and valid field/property access. The specification also places restrictions on entity classes and persistent members, so I check the version-specific requirements rather than assuming any ordinary Java class can be mapped unchanged. I keep entities focused on domain persistence instead of using them as REST request and response objects.

If the database schema doesn't match the default naming strategy, I make the table and column mappings explicit so the mapping remains understandable.

### Q14. How do you choose an identifier-generation strategy?

I choose the strategy based on the database, schema, insert volume, and batching requirements—not just because one annotation is familiar. With sequence-based generation, supported databases can allocate identifiers efficiently and still allow batching. With identity generation, the provider may need to execute an insert earlier to obtain the generated key, which can limit insert batching. `AUTO` delegates the choice to the provider and dialect, so it may not behave identically across databases.

In a high-volume insert path, I verify the generated SQL and actual batching behavior before deciding which strategy is best.

<a id="top-08"></a>
### Q15. What is the owning side of a relationship?

The owning side is the side Hibernate uses to determine how the relationship is updated in the database. In a typical bidirectional one-to-many/many-to-one mapping, the `@ManyToOne` field owns the foreign key, while the parent's `@OneToMany` collection uses `mappedBy`.

For example, adding an order to a customer's collection is not enough if the owning `order.customer` reference remains null. I set the owning side and keep both Java references consistent, usually through a helper method.

### Q16. What does mappedBy mean?

I use `mappedBy` on the inverse side of a bidirectional relationship to point to the association field on the owning entity. It names a Java attribute, not the database column. For example, `mappedBy = "customer"` points to the field named `customer` in the child entity.

If the value doesn't match the mapped attribute, application startup may fail. If I update only the inverse side at runtime, the in-memory graph may look right while the foreign key remains unchanged.

<a id="top-09"></a>
### Q17. How are cascade and orphanRemoval different?

Cascade means that selected entity operations on a parent propagate to related entities. For example, cascading persist can make a new child persistent when I persist the parent. Orphan removal is different: when a privately owned child is removed from the relationship, the provider schedules that child for deletion.

I use orphan removal only when the child truly belongs to that parent and has no independent lifecycle. I am careful with cascade remove on shared relationships, because deleting one parent should not unexpectedly delete an object still used elsewhere.

### Q18. When would you use OneToOne, OneToMany, ManyToOne, or ManyToMany?

I choose the mapping that reflects the relational model and lifecycle of the domain. ManyToOne is common for a foreign key from a child to a parent; OneToMany represents the parent's collection. OneToOne represents a unique association, often enforced by a unique foreign key. ManyToMany can work when the join has no meaningful attributes and both sides share the relationship.

If the join itself has fields such as role, status, or creation time, I usually model it as a separate association entity. That makes its lifecycle and constraints explicit and is easier to extend later.

### Q19. Why is LAZY often preferable to EAGER?

I generally prefer LAZY for associations because different use cases need different amounts of data. EAGER means the association must be available, but it does not guarantee that Hibernate will fetch it in one join. Depending on the query and mapping, the provider may issue secondary selects, which can cause N+1 behavior.

Lazy loading still needs planning: if I access an uninitialized association after the persistence context closes, Hibernate may throw a LazyInitializationException. I define the fetch plan for the use case instead of making everything EAGER.

<a id="top-10"></a>
### Q20. Why are equals() and hashCode() tricky for entities?

The main difficulty is that an entity's identity can change during its lifecycle. A generated ID may be null before persistence and assigned afterward. If I use that ID naïvely to calculate a hash code, the hash code can change after the entity has been inserted into a HashSet, making lookups unreliable. Mutable business fields create similar problems.

I choose an identity strategy deliberately and test transient, managed, detached, and proxy cases. A stable, genuinely unique natural key can work in some domains; generated-ID strategies need care. I would not apply one generic equals/hashCode implementation to every entity without considering the model.

### Q21. How do you map a composite primary key?

When the relational key is genuinely composite, I can use `@EmbeddedId` with an embeddable key class or `@IdClass`. The key type needs equality and hash-code behavior consistent with the database key. Composite IDs can complicate associations, repository signatures, and query code, so I prefer a surrogate key when it fits the domain and schema rather than introducing a composite key without a reason.

### Q22. What is the difference between JoinColumn and JoinTable?

A `@JoinColumn` typically maps a foreign-key column in one of the related tables. A `@JoinTable` maps an intermediate table that contains join columns pointing to the participating tables. I choose based on the actual relational design and cardinality, not on a blanket rule that a particular annotation belongs to one relationship type.

When debugging, I compare the annotations with the actual schema and inspect the generated SQL to confirm which table and columns Hibernate is using.

### Q23. Why can a bidirectional relationship look correct in Java but not persist correctly?

The usual cause is that the application updated only the inverse side. In a common parent-child mapping, I need to set the child's owning-side reference, not just add the child to the parent's collection. I normally create helper methods that update both sides so the in-memory object graph stays consistent.

I also check whether a new child needs a cascade operation and then inspect the SQL and foreign-key value. Cascade alone does not automatically fix an incorrectly maintained owning side.

### Q24. Should I return JPA entities directly from REST controllers?

I generally avoid doing that. Serializing an entity can trigger lazy loads, expose fields that should not be part of the API, create recursion through bidirectional relationships, and couple the public response format to the database model.

Instead, I fetch the data needed by the use case and map it to a DTO inside the service boundary. For read-heavy endpoints, a DTO projection can be even more efficient because it selects only the required columns.


## Part 3. Entity Lifecycle and Persistence Context

<a id="top-07"></a>
### Q25. What is the difference between persist() and merge()?

I use `persist()` when I want a new entity instance to become managed. I use `merge()` when I need the state of an entity object to be copied into a managed instance. The subtle but important point is that merge returns the managed instance; it does not make the original detached object managed again.

For example, if I merge a detached Product, I continue working with the object returned by merge. This distinction matters when changes are made after the merge call or when the returned instance contains state loaded by the provider.

### Q26. What is the difference between find() and getReference()?

With `find()`, I ask the EntityManager to return the entity for an identifier, or null if it does not exist. The persistence context can satisfy the request without a database query if the entity is already managed. With `getReference()`, I can obtain a reference without immediately loading all state, which is useful when I only need to assign an association by ID.

I don't treat getReference as a promise that no SQL will occur. Accessing the proxy's state can trigger a load, and behavior around a missing row depends on provider and timing.

### Q27. What do detach(), clear(), and refresh() do?

I use `detach(entity)` to stop tracking one entity, and `clear()` to detach everything in the persistence context. `refresh(entity)` reloads the entity's state from the database, overwriting local changes to that entity.

A common batch-processing mistake is to clear the context before flushing pending changes. If those changes must be saved, I flush first, then clear at a deliberate boundary. I also use refresh carefully because it can discard unsaved edits.

### Q28. Why can a managed entity update without calling save()?

Once I load an entity into the current persistence context, Hibernate tracks its state. If I modify a field inside a transaction, dirty checking detects the change and schedules an update during flush. For that already-managed entity, calling Spring Data's save method again is usually unnecessary.

That doesn't mean the update is guaranteed regardless of context. The entity must be managed, the transaction must complete successfully, and the flush must succeed.

### Q29. What is flush mode?

Flush mode controls when the persistence provider synchronizes pending changes with the database. With JPA's AUTO mode, the provider must ensure queries see relevant pending changes, which can mean flushing before a query. COMMIT generally defers synchronization toward transaction completion, but it should not be interpreted as an absolute guarantee that no earlier flush occurs.

I would tune flush behavior only after understanding the query semantics and measuring the effect. Provider-specific modes need to be checked against the Hibernate version in use.

### Q30. Why doesn't changing a detached entity update the database?

Dirty checking only tracks entities managed by a persistence context. If I load an entity, close the context, and then change a field on that Java object, I've changed memory—not the database.

I can load the current managed entity and apply the intended changes, or deliberately merge detached state. I am cautious about merging partially populated objects because missing fields or relationships may unintentionally overwrite existing state.

### Q31. When would you use refresh()?

I use refresh when I need to discard the current in-memory state and reload an entity from the database—for example, after a database-side operation makes my managed instance stale. Refresh can also discard my unsaved changes, so I call it only when that is intended.

If the returned state still looks old, I investigate transaction isolation, context lifetime, and cache behavior rather than assuming refresh always bypasses every layer.

### Q32. What are entity lifecycle callbacks used for?

Callbacks such as `@PrePersist`, `@PreUpdate`, and `@PreRemove` let me run small pieces of logic at defined lifecycle points. They can be useful for simple entity-local rules, but I avoid putting service workflows or complicated database operations inside callbacks because the behavior becomes harder to reason about and test.

I also distinguish callback timing from transaction commit: a callback can run before the transaction has successfully completed.

### Q33. Why can flush fail even when my setters work?

A setter normally changes an in-memory value; it does not prove that the value satisfies every database constraint. When Hibernate flushes, it may execute SQL and encounter a uniqueness, foreign-key, not-null, or optimistic-locking failure.

When that happens, I inspect the deepest database exception and roll back the failed transaction. I don't catch the error and assume the same persistence context can safely continue.

### Q34. How can the persistence context cause memory pressure?

A persistence context retains managed entities and their tracking state. If a batch job loads or inserts tens of thousands of entities in one context, memory usage can grow and flush processing can become expensive.

For large jobs, I process bounded chunks and flush/clear at safe points. I first decide whether the business operation must be fully atomic, because committing each chunk changes the rollback behavior.


## Part 4. Fetching and Performance

<a id="top-11"></a>
### Q35. What is the N+1 query problem?

I call it an N+1 problem when the application runs one query to load a set of root entities and then runs another query for each root to load related data. For example, I might load 100 orders in one query and then trigger 100 separate customer or item queries while building the response.

The endpoint may still look correct in a small test, but the number of database round trips grows with the result size. I confirm the problem using SQL logs, query counts, traces, and realistic data volumes.

<a id="top-12"></a>
### Q36. How would you fix an N+1 query?

First, I identify which association is triggering repeated queries and what data the use case actually needs. Then I compare a fetch join, an entity graph, batch fetching, or a DTO projection. A fetch join may be ideal for a required to-one relationship; a projection is often better when the endpoint only needs a few fields.

I don't automatically join every collection, because one large join can multiply rows and consume more memory than a few bounded queries. After changing the fetch plan, I measure both statement count and total rows transferred.

<a id="top-13"></a>
### Q37. What is a fetch join, and what are its limitations?

A fetch join lets me request an association as part of a JPQL query, so Hibernate can load the relationship along with the root entities. It is useful when I know the response needs that relationship immediately.

The main trap is collection fetching. If a root has multiple child rows, the SQL result contains repeated root columns. Fetching multiple collections can multiply rows, and combining a collection fetch join with pagination can make the database limit rows rather than logical root entities. I check Hibernate warnings and the exact version's behavior instead of assuming the query is safe.

### Q38. When would you use an entity graph?

I use an entity graph when I want to define which attributes a particular operation should fetch without embedding every fetch decision into the JPQL itself. A named graph can be reused, while a dynamic graph can be assembled for a specific use case.

An entity graph describes the fetch plan, not a guarantee that Hibernate will use exactly one SQL join. I still inspect generated SQL and verify that the graph doesn't load more data than the endpoint needs.

### Q39. Why not configure every association as EAGER?

EAGER means the association must be available, not that Hibernate must fetch everything in one join. Depending on the query, Hibernate can issue secondary selects, and loading associations that a caller never uses wastes work.

I generally keep associations LAZY and explicitly fetch what each use case needs. This gives me more predictable query behavior, but it requires a clear service boundary so required data is loaded before the persistence context closes.

<a id="top-14"></a>
### Q40. What causes LazyInitializationException?

With Hibernate, this exception commonly occurs when the code accesses an uninitialized proxy or collection after the Session/persistence context is no longer available to load it. A typical example is returning an entity from a service and then having JSON serialization access a lazy collection after the service transaction has ended.

I fix the fetch plan: load the required data inside the transaction and map it to a DTO before returning. Making everything EAGER can create a different performance problem, so I don't treat it as the default solution.

### Q41. What are the trade-offs of Open Session in View?

Open Session in View keeps the persistence context open through web request processing, which can allow lazy associations to load while a response is being serialized. That may hide missing fetch plans, but it can also trigger unexpected database calls outside the service transaction and make latency difficult to predict.

If I disable OSIV, I make sure service methods explicitly load the data the response needs. Whether OSIV is enabled by default depends on framework configuration and version, so I check the application rather than assume.

<a id="top-15"></a>
### Q42. Why can a collection fetch join break pagination?

A collection join can produce several SQL rows for one root entity. Database pagination applies its limit and offset to those rows, which may cut through a root's children or return an unexpected set of roots. Some provider configurations may instead fetch a large result and paginate in memory, which can be very expensive.

A safer pattern is to page the root IDs first and then fetch the associations for those IDs in a second query. A DTO projection designed for pagination can be another good option.

### Q43. When would you choose a DTO projection over an entity?

If an endpoint only needs a few fields for display or reporting, I usually prefer a DTO projection. It avoids materializing a full managed entity and can avoid loading associations the endpoint never uses. Entities make more sense when I need domain behavior, relationship navigation, or managed updates.

I verify the generated SQL because the projection type alone doesn't guarantee the most efficient query, especially with nested projections or complex joins.

### Q44. What is batch fetching?

Batch fetching lets Hibernate load several eligible lazy associations or entities together, often using an IN-style query, instead of issuing one query for every object. It can reduce round trips without joining all collections into a single enormous result.

It is a useful option when access patterns naturally touch several related entities. I tune the batch size based on measurements and database constraints, and I verify the actual statement count.

### Q45. How do you diagnose a slow Hibernate query?

I start by separating database execution time from ORM and application overhead. I inspect generated SQL, query counts, bind types where safe, tracing data, connection-pool wait time, and the database execution plan. I look for N+1 patterns, unnecessary columns, missing indexes, poor join cardinality, lock waits, and long transactions.

Then I change one likely cause at a time and compare the results under realistic data volume. I don't assume that adding a cache or changing a fetch setting will help without measuring it.

### Q46. Why can an endpoint be slow when each SQL statement is fast?

The endpoint may be executing hundreds of individually fast statements, waiting for a connection, transferring too many rows, serializing a large object graph, or doing non-database work while holding a transaction open. Looking only at the slowest SQL statement misses those costs.

I measure end-to-end latency, total query count, rows returned, pool wait time, and serialization time. That gives me a better basis for deciding whether to reduce round trips, change the fetch plan, or shorten the transaction.


## Part 5. Transactions and Concurrency

<a id="top-16"></a>
### Q47. How does @Transactional interact with JPA?

In a typical Spring application, `@Transactional` tells Spring's transaction infrastructure to start or join a transaction around the method. Spring's JPA integration coordinates the EntityManager so persistence operations participate in that transaction. Managed changes are synchronized when the context flushes, and the transaction manager commits or rolls back the database transaction.

I check how the method is invoked, because proxy-based transaction advice can be bypassed by self-invocation. I also remember that Spring's default rollback rules generally apply to unchecked exceptions and Errors, not every checked exception.

<a id="top-17"></a>
### Q48. What is the difference between optimistic and pessimistic locking?

I choose optimistic locking when conflicts are expected to be relatively rare. The application reads a row and later checks whether it has changed before writing. Pessimistic locking asks the database to lock the row or rows so conflicting work may have to wait.

Optimistic locking avoids holding database locks for the entire business operation, but callers need a conflict strategy. Pessimistic locking can be useful under high contention, but it can increase blocking and deadlock risk. The right choice depends on the invariant, contention, transaction length, and database behavior.

<a id="top-18"></a>
### Q49. How does @Version prevent lost updates?

With a versioned entity, Hibernate includes the version value it read in the update condition and increments the version when the update succeeds. If another transaction has already changed that row, the stored version no longer matches. Hibernate detects that the update did not affect the expected row and raises an optimistic-lock exception.

For example, two users may open the same account settings page. If one saves first, the second user's stale version should not silently overwrite the first user's changes. I then decide whether the application should retry after reloading, ask the user to reconcile, or use a different update model.

### Q50. What is a lost update?

A lost update happens when two transactions read the same starting value, calculate new values independently, and then overwrite one another. The later write can erase the earlier change even though both requests appeared successful.

I prevent it using version-based optimistic locking, suitable pessimistic locking, an atomic database update, or a domain model that avoids the read-modify-write race. I choose based on the business invariant and test with genuinely concurrent transactions.

### Q51. Does flush commit a transaction?

No. Flush sends pending SQL to the database within the current transaction. Commit completes the transaction, while rollback can undo the flushed changes. Flush is useful because it can reveal constraint violations or locking problems earlier, but it does not make the operation durable on its own.

### Q52. How do isolation levels affect JPA applications?

JPA applications inherit most isolation behavior from the database transaction. READ COMMITTED, REPEATABLE READ, and SERIALIZABLE can prevent different anomalies, and their precise behavior varies between database engines.

I don't assume that selecting an isolation name makes every database behave identically. I identify the business invariant, understand the actual database semantics, and use constraints or locking when correctness depends on preventing a specific race.

### Q53. What does @Lock in Spring Data JPA do?

I can annotate a repository query with `@Lock` to request a JPA lock mode, such as a pessimistic write lock where supported. The annotation expresses the request; it doesn't guarantee identical blocking behavior on every database. The transaction boundary, provider, dialect, and database all matter.

For a concurrency issue, I verify the SQL and lock behavior on the actual database rather than relying only on the annotation in the code.

### Q54. Why can a transaction be rollback-only after an exception was caught?

Catching an exception doesn't necessarily make a transaction healthy again. A persistence failure or participating transactional method may already have marked the transaction rollback-only. If the outer method continues and tries to commit, Spring can report an unexpected rollback.

I don't continue writing in a transaction after a serious persistence failure. I let it roll back and, if the work is genuinely independent, run that work in a separately designed transaction.

<a id="top-19"></a>
### Q55. What are the risks of bulk JPQL updates and deletes?

Bulk JPQL operations update database rows directly; they don't update every already-managed entity through normal dirty checking or lifecycle callbacks. That means the persistence context can contain stale values after the query.

I flush pending changes first when required, execute the bulk operation, and then clear or refresh affected entities when subsequent code needs current state. I also consider cache invalidation and transaction semantics. Bulk operations are efficient, but their behavior is different from changing entities one by one.

### Q56. How can long transactions hurt performance?

Long transactions can hold connections and locks, increase contention, keep many entities managed, and extend the period in which concurrent updates can conflict. They become especially problematic when the code performs remote calls or lengthy calculations while the transaction is open.

I keep the transaction around the database work that needs to be atomic. External side effects may need an outbox or another explicit coordination pattern rather than simply extending the database transaction across network calls.

### Q57. Should every optimistic-lock exception be retried?

No. A retry is only useful if repeating the operation is safe and the business rules still hold after reloading current state. A retry that blindly repeats a payment or another non-idempotent side effect can create a more serious bug.

For some operations, a bounded retry is reasonable. For user-edited records, returning a conflict may be the correct behavior. For simple increments, an atomic database update may be more appropriate than loading and rewriting the whole entity.

### Q58. How does transaction propagation affect persistence?

Propagation determines whether a method joins an existing transaction or uses a different one. REQUIRED commonly joins the current transaction or creates one if needed. REQUIRES_NEW suspends the outer transaction and creates an independent transaction, so its changes can commit even if the outer operation later rolls back.

I use REQUIRES_NEW only when that independence is intentional. Otherwise, it can leave partial business outcomes that callers did not expect. Proxy interception and the configured transaction manager also affect the actual behavior.


## Part 6. JPQL, Criteria, and Querying

<a id="top-20"></a>
### Q59. What is the difference between JPQL and SQL?

I use JPQL when I want to query the entity model rather than the physical schema. The query refers to entity names, Java attributes, and mapped relationships. Native SQL refers directly to tables and columns and can use database-specific features.

JPQL is often easier to keep portable, while native SQL can be the right choice for specialized reporting or database features. I choose based on the query's needs and inspect the generated SQL either way.

### Q60. Why should query parameters be bound instead of concatenated?

I bind user-supplied values because parameters keep the data separate from the query structure and help prevent SQL injection. For example, I would use a named parameter for a product SKU instead of building the query string by concatenating the SKU value.

One subtle point is that parameters generally bind values, not arbitrary identifiers such as a column name or sort direction. If users can choose sorting fields, I validate those choices against an allowlist rather than concatenating unchecked input.

### Q61. When would you use native SQL?

I use native SQL when the query needs a database-specific feature, a specialized report, or a form of tuning that is awkward or impossible in JPQL. It can also be useful for carefully designed bulk operations.

The trade-off is tighter coupling to the schema and database dialect, plus potentially more involved result mapping. I first establish a real requirement with measurements rather than switching to native SQL just because it looks more familiar.

### Q62. How would you implement pagination safely?

For a typical Spring Data query, I can pass a Pageable and receive a Page or Slice. I make the sort deterministic, ideally including a unique tie-breaker, so rows don't move unpredictably between pages when sort values are equal.

I also consider the cost of deep offsets and count queries. For very large or frequently changing datasets, keyset pagination may be more efficient. If the query fetches a collection, I avoid combining a collection fetch join with ordinary pagination without checking its behavior.

<a id="top-21"></a>
### Q63. What is the difference between Page and Slice?

I use Page when the caller needs total-element or total-page information. That commonly means Spring Data needs a count query in addition to fetching the requested rows. If the UI only needs to know whether another page exists, Slice can avoid the full count and may be cheaper.

I choose the return type based on what the caller actually needs rather than calculating a total count by default.

### Q64. When are DTO projections useful?

If an endpoint needs only a few fields—for example, an order number, status, and creation date—I usually don't need to load a full entity graph. A DTO projection can select just those values, reduce data transfer, and avoid unnecessary persistence-context tracking.

Spring Data JPA supports interface and class-based projections. I still inspect the generated SQL, because query shape and nested projections can affect which columns and joins are used.

### Q65. What is the Criteria API, and when would you use it?

The Criteria API lets me construct queries programmatically using a typed object model. It can be useful when a search screen has many optional filters and I need to compose predicates dynamically. The trade-off is verbosity; a simple static query is often clearer as JPQL or a repository method.

I use Criteria when dynamic composition makes the code easier to maintain, not as the default for every query.

### Q66. What are Spring Data JPA Specifications?

A Specification wraps a Criteria predicate so I can compose reusable filters. For example, a product search could optionally filter by category, status, and price range without creating a separate repository method for every combination.

I watch for joins that duplicate root rows and for expensive count queries during pagination. If the use case becomes reporting-heavy or structurally complex, a dedicated query implementation may be easier to understand.

### Q67. When should I use a derived query method versus @Query?

Derived methods are convenient when the predicate is simple and the method name remains readable. When a query has several joins, a projection, a fetch plan, or complicated conditions, I prefer an explicit `@Query` because the intent is easier to review.

I avoid extremely long method names that encode an entire query in a method signature. The goal is maintainable query logic, not maximizing the amount of query text hidden in a name.

### Q68. How should I handle a modifying repository query?

For an update or delete query declared with `@Query`, I use `@Modifying` and execute it in an appropriate transaction. These bulk operations bypass normal entity-by-entity dirty checking, so an entity already loaded in the persistence context may still show its old values.

I decide whether to flush pending changes first and whether to clear or refresh managed state afterward. I don't turn on automatic clearing without considering that it detaches all managed entities in that context.

### Q69. Why can a query return duplicate root entities?

When I join a root entity to a collection, the SQL result can contain multiple rows for the same root—one per matching child. How that appears in the Java result depends on query semantics and provider behavior. DISTINCT can help in some cases, but it doesn't automatically make collection-fetch pagination safe.

I inspect the SQL and result shape, then decide whether a projection, a different join, or a two-step fetch is more appropriate.

### Q70. What do you investigate when a query is slow?

I start with the actual SQL and parameters, then inspect the execution plan, indexes, join cardinality, sorting, row counts, and lock waits. I also check whether the ORM is issuing too many statements or retrieving columns and relationships the endpoint doesn't need. For paged results, I look at the count query separately.

I test with representative data volume because a query that looks fine on a tiny development database can behave very differently in production.


## Part 7. Spring Data JPA

### Q71. What does Spring Data JPA add on top of JPA?

JPA gives me the standard persistence APIs, while Spring Data JPA reduces the repository code I need to write. It can derive queries from method names, support explicit queries, provide pagination and sorting, and return projections. It also integrates repositories with Spring's transaction and dependency-injection infrastructure.

It doesn't replace Hibernate or the database. When an application has an N+1 issue, stale managed state, or unexpected SQL, I still need to understand the underlying JPA provider behavior.

<a id="top-22"></a>
### Q72. What does save() actually do?

Spring Data JPA checks whether it considers the entity new. It generally calls `EntityManager.persist()` for a new entity and `merge()` for an existing one. The new-entity decision can depend on a nullable version property, identifier state, or a custom `Persistable` implementation.

The important detail is that merge returns the managed instance; it does not make the passed detached object managed. So if I need to use the merged entity afterward, I use the return value. I also don't assume save always means an immediate SQL UPDATE.

<a id="top-23"></a>
### Q73. Why might save() trigger a SELECT?

That isn't automatically a bug. If Spring Data decides an entity is not new, it may use merge semantics, and merging can require Hibernate to load the managed state before copying values. This can be surprising when an entity uses an application-assigned identifier.

I inspect the identifier and version fields and how Spring Data determines newness. If needed, I configure the newness strategy appropriately, but only after verifying the entity's lifecycle semantics.

### Q74. Are repository methods transactional by default?

Inherited CRUD repository methods have default transaction settings: read operations are generally read-only, while write operations are transactional. But declared query methods do not automatically receive transaction configuration just because they are repository methods; I configure a transaction where needed or let them participate in a service-level transaction. Custom repository implementations also need their transaction behavior checked explicitly.

For a business operation that updates several entities or calls multiple repositories, I normally put the transaction boundary at the service layer. That makes the intended unit of work explicit instead of relying on separate repository calls to define it.

### Q75. What does @Transactional(readOnly = true) do?

I use it to communicate that a transaction is intended for reading and to allow applicable framework/provider optimizations. However, I don't treat it as a universal guarantee that a write is impossible or as a security control. The exact effect depends on the transaction manager, provider, and database configuration.

If read-only behavior matters for correctness or access control, I enforce it at the appropriate database or application layer and measure any performance benefit.

### Q76. What does @Modifying do?

I put `@Modifying` on a Spring Data `@Query` method that performs an update or delete rather than returning selected entities. It normally needs a transaction. Because the query operates directly on database rows, entities already managed by the persistence context may not reflect the new values.

I consider whether the context needs to be flushed before the query and cleared or refreshed afterward. Those choices depend on what the caller does next.

### Q77. How does Spring Data parse derived query method names?

Spring Data interprets the method name as a property path and predicate. That works well for straightforward queries, but a method name becomes hard to read when it encodes many nested properties, joins, and OR conditions.

When the method stops communicating intent clearly, I switch to `@Query` or Specifications. I also rely on application startup and tests to catch invalid property paths rather than assuming the method name is correct.

### Q78. What are repository projections?

A projection lets the repository return the data shape the caller needs instead of always returning a full entity. Interface projections can be concise, and class-based projections can define an explicit DTO shape. They are useful for read APIs and summary views.

I verify the SQL rather than assuming projections always select only the expected columns. Nested or open projections can have different loading implications.

### Q79. How does Spring Data auditing work?

Auditing can populate fields such as created time, last-modified time, and the user responsible for a change. I need to enable the auditing infrastructure and configure the current auditor if user identity is tracked.

I distinguish this convenience from a complete compliance audit trail. If I need immutable historical records of every change, I may need a dedicated audit model, event history, or database-level auditing.

### Q80. When would you write a custom repository implementation?

I use a custom repository fragment when a query or persistence operation is too specialized for derived methods or a simple `@Query`. Examples include dynamic native queries or a persistence path that needs careful control over query construction.

I keep the custom code focused and test its transaction and SQL behavior. I don't recreate the whole repository abstraction just to support one specialized method.

### Q81. How do you design pagination for a large table?

I start by asking whether the caller needs a total count. If not, Slice may avoid an expensive count query. I ensure ordering is deterministic and that the relevant filter and sort columns have appropriate indexes. Deep offset pagination can become expensive, so keyset pagination is worth considering when the access pattern supports it.

Pageable is a convenient API, but it cannot make an inefficient database query efficient by itself.

### Q82. Does saveAndFlush() commit the transaction?

No. `saveAndFlush()` applies save semantics and forces pending changes to be synchronized with the database. The surrounding transaction can still roll back, so flush is not the same as commit.

I use it when I specifically need SQL to execute at that point—for example, to surface a constraint failure earlier—not as a general way to make repository writes more reliable.


## Part 8. Caching and Optimization

<a id="top-24"></a>
### Q83. What is the difference between first-level and second-level cache?

The first-level cache is the persistence context that Hibernate uses to track managed entities and preserve identity within that context. It is part of normal entity management. The second-level cache is optional and can share cached entity data across persistence contexts, depending on configuration.

So if I load the same entity twice in one context, Hibernate can reuse the managed instance. A second-level cache can reduce database reads across transactions, but only if the data's update frequency and consistency requirements make caching worthwhile.

### Q84. What is Hibernate's query cache?

Hibernate's query cache can cache query result information according to its configured strategy, but it isn't a universal cache for every SQL result. Its usefulness depends on the query pattern, cache configuration, and how the underlying data changes; it is often considered alongside second-level caching.

I don't enable it across the application just because it exists. I measure hit rates and database load, and I verify invalidation behavior for the actual access pattern.

### Q85. When does second-level caching help?

I consider it for data that is read frequently, changes relatively infrequently, and has a clear acceptable-consistency model—for example, reference data shared across many requests. It is less attractive for highly mutable records or data that must always be fresh.

I choose a suitable cache provider and concurrency strategy, and I remember that the database remains the source of truth. A cache is useful only if the reduction in database work outweighs invalidation and operational complexity.

### Q86. What can go wrong with cache invalidation?

Stale data can appear when updates happen through direct SQL, another application, or a code path that doesn't coordinate with the same cache. The outcome depends on the cache strategy, transaction boundaries, and whether all writers participate in invalidation.

I test those out-of-band update paths explicitly. If the application cannot guarantee the required freshness, I either change the caching design or avoid caching that data.

### Q87. What is JDBC batching in Hibernate?

JDBC batching lets Hibernate submit multiple compatible parameter sets through a JDBC batch, reducing per-statement overhead and potentially reducing network round trips. It does not guarantee a single server call or a fixed number of round trips; the outcome depends on the driver, database, configuration, statement shape, and identifier generation. Identity-generated IDs can constrain insert batching because the generated key may be needed immediately.

For a batch job, I also consider flushing and clearing in chunks so the persistence context doesn't grow without bound. Then I verify actual batching rather than assuming a configuration property is enough.

### Q88. Why might configured batching not improve performance?

The statements may not actually be batched, the identifier strategy may prevent it, or the SQL statements may not be compatible for batching. Even when batching works, the main cost may be indexes, constraints, lock contention, or database execution rather than network round trips.

I check the executed batch behavior and compare throughput under realistic load. Configuration is only a hypothesis; measured improvement is the evidence.

### Q89. What is Hibernate StatelessSession?

StatelessSession is a Hibernate-specific API intended for certain workloads that need reduced state management. It does not provide the usual persistence-context behavior and automatic dirty checking that I rely on with a normal Session.

It can be useful for specialized bulk processing, but I wouldn't substitute it casually into ordinary domain logic. I first confirm that the application doesn't depend on the lifecycle and identity semantics that a stateful session provides.

### Q90. How do you approach optimizing entity loading?

I start by removing unnecessary work: reduce repeated queries, fetch only the associations the use case needs, use projections when I don't need full entities, and paginate appropriately. Then I measure whether batch fetching, indexes, JDBC batching, or caching would address the remaining bottleneck.

Before tuning, I record a baseline such as latency, SQL count, rows transferred, pool wait, and memory. Afterward, I compare the same metrics under realistic conditions and verify correctness.

### Q91. Do read-only hints guarantee that writes cannot happen?

No. An ORM read-only hint can influence tracking or optimization, but its enforcement depends on the provider and configuration. A database-level read-only transaction may have stronger behavior, but I don't assume every layer interprets the hint identically.

If preventing writes is a correctness or security requirement, I enforce it through appropriate permissions and application design, not just an ORM annotation.

### Q92. How would you optimize a bulk import?

I first determine the required atomicity: must the whole import succeed or fail as one unit, or are chunk-level commits acceptable? Then I validate input in bounded chunks, use JDBC batching when supported, avoid loading large graphs, and flush/clear at safe points if I use a stateful persistence context.

For very large imports, JDBC or database-native bulk loading may be more appropriate. I measure throughput and memory, and I account for the fact that committing each chunk changes rollback semantics.


## Part 9. Production Troubleshooting Scenarios

### Q93. An endpoint triggers hundreds of SQL statements. How do you investigate?

I would start with a representative request and capture its trace and SQL statement count. Then I'd group statements by shape and look for repeated queries that load associations inside a loop. ORM statistics and application traces help connect the SQL to the code path; I avoid logging sensitive bind values in production.

Once I identify the access pattern, I'd compare a projection, fetch join, entity graph, or batch fetching. I'd re-test with realistic result sizes to make sure I reduced round trips without creating an enormous join result.

### Q94. A request fails with LazyInitializationException. What is your approach?

First, I'd locate which lazy association is accessed and where the persistence context ends. A common cause is returning an entity from a service and having JSON serialization touch an uninitialized collection afterward.

I'd define the data the endpoint actually needs, fetch it inside the intended transaction, and map it to a DTO before returning. I wouldn't make every association EAGER as a quick fix, because that can introduce N+1 queries and over-fetching elsewhere.

### Q95. A field changes in Java, but no UPDATE appears. What do you check?

I'd check whether the entity is managed, whether the new value differs from the old value, and whether the operation runs inside the expected transaction. I'd also verify that the transactional method is actually intercepted, that the entity wasn't detached, and that SQL logging is enabled for the correct datasource.

If the field didn't really change, no UPDATE may be correct. If it did, I'd follow the operation through flush and commit rather than adding a save call without understanding the cause.

### Q96. A bidirectional relationship is missing after save. Why?

I'd inspect which side owns the relationship and whether that side was set. In a common parent-child mapping, adding the child to the parent's collection is not enough if the child's foreign-key reference remains null. I'd also check cascade settings if the child is new.

Then I'd flush and inspect the generated SQL and foreign-key value. Helper methods that update both sides can prevent this class of bug from recurring.

### Q97. A paginated endpoint becomes slow after a collection fetch join. Why?

The join can produce several SQL rows per root entity, so the database's limit and offset no longer correspond cleanly to logical root entities. Depending on Hibernate version and configuration, the provider may warn or paginate in memory, which can load far more data than expected.

I'd consider paging root IDs first and fetching their associations in a second query, or using a DTO projection. I'd verify both the result correctness and the execution plan after the change.

### Q98. A bulk update runs, but later reads show old values. Why?

Bulk JPQL updates bypass the normal synchronization of each managed entity. If the persistence context already contains an entity affected by the update, that Java object can still hold the old value.

I'd flush pending changes if needed, run the bulk query, and then clear or refresh the affected state when later code needs current values. If stale data persists across contexts, I'd also check transaction isolation and second-level cache behavior.

### Q99. Optimistic locking fails intermittently under load. What would you do?

I'd treat the exception as evidence that another transaction changed the row, then identify which business operations overlap and inspect the version values. The exception isn't automatically a Hibernate defect; tests with sequential requests may never reproduce it.

Next I'd decide whether the operation can safely retry after reloading, should return a conflict to the caller, or needs a different update/locking strategy. Any retry must be bounded and safe to repeat.

### Q100. save() executes an unexpected SELECT. Is that a bug?

Not necessarily. Spring Data may decide the entity is not new and call merge; merge can load existing state before copying the supplied state into the managed instance. This can happen with application-assigned IDs or a new-entity detection strategy that doesn't match the entity's lifecycle.

I'd inspect the ID/version fields and the returned instance before changing the implementation. The goal is to correct the lifecycle semantics, not to eliminate a SELECT at any cost.

### Q101. Database connections remain occupied for too long. What do you investigate?

I'd inspect connection-pool wait time, transaction duration, slow queries, thread dumps, and whether the code holds a transaction open while making remote calls or doing expensive work. Streaming queries and resources that aren't closed can also keep connections occupied.

I'd fix the actual cause before increasing pool size, because a larger pool can increase database contention without reducing the time each connection is held.

### Q102. SQL statements are fast, but the service is slow. Why?

The service may be executing too many individually fast statements, waiting for a connection, transferring too many rows, serializing a large graph, or doing application work inside a transaction. The slowest single SQL statement doesn't necessarily explain total request latency.

I'd compare end-to-end traces, total statement count, rows transferred, connection wait, and serialization time. Then I'd optimize the dominant measured cost.

### Q103. An entity appears stale after another transaction updates it. Why?

A managed entity isn't automatically refreshed every time another transaction changes the database. The current persistence context may still hold the previous state, and isolation or second-level caching can also affect what is visible.

I'd check the context and transaction lifetime first, then reload or refresh as appropriate. If another service updates the same table directly, I'd also review how that update interacts with the cache strategy.

### Q104. A constraint violation appears at commit rather than save. Why?

The save operation may only have changed managed state, with SQL deferred until flush. Some databases also support constraints that are checked at transaction end. So the point where the exception appears doesn't necessarily mean the invalid value was introduced at commit.

I'd inspect the deepest database exception and reproduce with a controlled flush to determine when the constraint is checked. After a persistence failure, I'd roll back instead of continuing in the same failed transaction.

### Q105. A bulk import causes memory growth. How would you address it?

I'd check whether one persistence context is retaining every entity and its tracking state. For large imports, I'd process bounded chunks and flush/clear at safe intervals, or consider JDBC/database-native loading if ORM lifecycle behavior isn't needed.

Before introducing chunk commits, I'd confirm whether the import must be atomic. Smaller transactions can reduce memory and lock duration, but they change what happens if a later chunk fails.

### Q106. EAGER associations still generate many queries. Why?

EAGER means the association must be available; it doesn't require Hibernate to fetch everything through one join. Depending on the query and mapping, secondary selects can still occur and create an N+1 pattern.

I'd inspect the actual SQL and replace the global expectation with a use-case-specific fetch plan, such as a fetch join, entity graph, or projection.

<a id="top-25"></a>
### Q107. How would you handle a Hibernate performance incident end to end?

I'd first establish impact and capture a representative trace. Then I'd measure total latency, query count, rows transferred, connection-pool waits, lock waits, and memory, and identify the dominant query or access pattern. I'd inspect the database execution plan and form a specific hypothesis.

I'd change one variable at a time, compare before and after under realistic load, and verify that data correctness hasn't regressed. Finally, I'd add a regression test or monitoring signal so the same problem is easier to detect next time.


## Part 10. Mock Interviews

Use these prompts to practise speaking naturally. Answer first, then compare your response with the model answer and handle the follow-up.

### Mock Interview 1 — Core JPA and Hibernate

Aim for a two-minute initial answer, then handle the follow-up without turning the response into a memorized definition.

#### Mock Q1. Explain JPA versus Hibernate.
I'd say JPA is the standard contract for persistence in Java, while Hibernate is an ORM provider that implements that contract and offers extra features. Using JPA APIs improves portability, although generated SQL and performance can still vary.

**Follow-up:** What would you isolate to make a provider migration easier?

#### Mock Q2. Explain the persistence context.
I think of it as Hibernate's managed workspace. It tracks entities, preserves identity within the context, and detects changes so they can be synchronized at flush. It is related to a transaction but isn't the same thing as the database transaction.

**Follow-up:** What happens to dirty checking when an entity becomes detached?

#### Mock Q3. Explain entity lifecycle states.
I distinguish the states by whether the persistence context tracks the object: transient objects are new and unmanaged; managed objects are tracked; detached objects are no longer tracked; removed objects are scheduled for deletion.

**Follow-up:** Does detaching an entity delete its database row?

#### Mock Q4. Compare persist and merge.
I use persist to make a new entity managed. Merge copies state into a managed instance and returns that instance; the object passed to merge doesn't itself become managed just because I called it.

**Follow-up:** Which instance would you continue working with after merge?

#### Mock Q5. Flush versus commit?
Flush sends pending SQL to the database within the transaction. Commit completes the transaction. If the transaction later rolls back, flushed changes can still be undone.

**Follow-up:** When might Hibernate flush automatically?

#### Mock Q6. Explain owning side and mappedBy.
The owning side controls the relationship update in the database. mappedBy points from the inverse side to the owning association attribute. In a common parent-child mapping, setting only the parent's collection may not update the foreign key.

**Follow-up:** How would you prevent inconsistent references in memory?

#### Mock Q7. Cascade versus orphan removal?
Cascade propagates selected entity operations from one entity to related entities. Orphan removal schedules deletion when a privately owned child is removed from the relationship. I only use orphan removal when the child really has no independent lifecycle.

**Follow-up:** Why can cascading REMOVE be dangerous for shared objects?

#### Mock Q8. Why is dirty checking useful?
It lets Hibernate detect changes to managed entities at flush, so I don't need to call update explicitly after every setter. The key requirement is that the entity remains managed and the transaction completes successfully.

**Follow-up:** How could dirty checking affect a large batch job?

#### Mock Q9. Explain lazy loading.
A lazy association is loaded when it is accessed, provided the persistence context can initialize it. If the context has closed, Hibernate may throw LazyInitializationException. I plan the fetch strategy around each use case rather than making everything EAGER.

**Follow-up:** How would you load the required data before returning a REST response?

#### Mock Q10. Explain N+1.
I get N+1 when one query loads the roots and additional queries load related data for individual roots. I confirm it by inspecting statement counts, then choose a fetch join, entity graph, batch fetching, or DTO projection depending on the relationship and response shape.

**Follow-up:** What makes collection fetch joins risky with pagination?

#### Mock Q11. What does Spring Data JPA add?
It reduces repository boilerplate with query derivation, projections, pagination, sorting, and Spring integration. It still relies on the JPA provider for persistence behavior, so I need to understand what Hibernate is doing underneath.

**Follow-up:** What does save() decide internally?

#### Mock Q12. Why are entity equals() and hashCode() tricky?
Generated IDs can change from null to assigned, and mutable fields can destabilize hash collections. I choose an identity strategy based on the domain and test transient, managed, detached, and proxy cases rather than copying a generic implementation.

**Follow-up:** How might Hibernate proxies affect equality?

### Mock Interview 2 — Senior Backend: Fetching, Transactions, Concurrency

#### Mock Q1. An endpoint has an N+1 issue. How do you fix it?
I'd capture a representative trace and query count, identify the association that triggers repeated loads, and compare a projection, entity graph, fetch join, or batch fetching. I'd verify statement count and row volume after the change.

**Follow-up:** What changes when the association is a collection and pagination is required?

#### Mock Q2. Why can collection fetch join pagination be unsafe?
A collection join produces multiple SQL rows per root, so the database limit applies to rows rather than logical root entities. I'd consider paging root IDs first and then fetching associations, or use a DTO projection.

**Follow-up:** How would you detect in-memory pagination?

#### Mock Q3. When is a DTO projection preferable to entities?
If the endpoint needs a read-only shape and doesn't need entity lifecycle tracking or navigation, a DTO projection can retrieve fewer columns and avoid loading a large graph. I'd inspect the generated SQL rather than assuming every projection is optimal.

**Follow-up:** What would you check in the query plan?

#### Mock Q4. Explain OSIV trade-offs.
OSIV can allow lazy loading during response serialization, but it may hide missing fetch plans and trigger database calls outside the service transaction. I'd define the response data explicitly and map it to a DTO within the intended service boundary.

**Follow-up:** What refactoring helps when disabling OSIV?

#### Mock Q5. Explain optimistic locking with @Version.
Hibernate includes the version it read in the update condition and increments it after a successful write. If another transaction changed the row, the update no longer matches and Hibernate reports a conflict.

**Follow-up:** When would you retry versus return a conflict to the caller?

#### Mock Q6. How do you prevent lost updates?
I'd use versioned optimistic locking, a suitable pessimistic lock, an atomic database update, or redesign the invariant. The choice depends on contention and business semantics; pessimistic locking also brings blocking and deadlock risk.

**Follow-up:** How would you test the race?

#### Mock Q7. What happens when a bulk update affects a managed entity?
Bulk updates bypass normal entity dirty checking, so an entity already in the persistence context may still show old values. I'd flush pending changes if necessary, run the bulk query, and clear or refresh the affected state.

**Follow-up:** What about second-level cache invalidation?

#### Mock Q8. Why can a transaction be rollback-only after an exception was caught?
The persistence provider or a participating operation may already have marked it rollback-only. Catching the exception doesn't restore the transaction's ability to commit. I'd let it roll back and isolate independent work in a new transaction only if that is the intended business behavior.

**Follow-up:** Why can REQUIRES_NEW create surprising outcomes?

#### Mock Q9. The connection pool has high wait time. What do you inspect?
I'd look at transaction duration, slow queries, remote calls inside transactions, leaked resources, streaming query lifetimes, pool metrics, and thread dumps. Increasing pool size without fixing long-held connections can just increase database contention.

**Follow-up:** Which metric distinguishes pool wait from SQL execution time?

#### Mock Q10. How would you optimize high-volume inserts?
I'd measure whether JDBC batching is actually happening, check whether the identifier strategy permits it, and process entities in bounded chunks with flush/clear where appropriate. I'd preserve the import's required atomicity.

**Follow-up:** When might JDBC or database-native loading be a better fit?

#### Mock Q11. When is second-level caching worthwhile?
I'd consider it for frequently reused, relatively stable data with an acceptable consistency model. I'd measure cache hit rates and test invalidation, especially if data can be changed outside Hibernate.

**Follow-up:** Why might query cache underperform?

#### Mock Q12. Walk through a persistence performance incident.
I'd establish impact, capture traces and query counts, inspect SQL plans and connection-pool waits, and form a measurable hypothesis. Then I'd change one variable at a time and compare performance and correctness under realistic load.

**Follow-up:** What metric or regression test would help prevent recurrence?

### Mock Interview 3 — Production Troubleshooting and Spring Data JPA

#### Mock Q1. save() triggers a SELECT. Explain.
I'd check Spring Data's new-entity detection first. It may choose merge rather than persist, and merge can load the existing managed state before copying values. I'd inspect identifier/version configuration and use the managed instance returned by merge.

**Follow-up:** How can a nullable @Version property affect new-entity detection?

#### Mock Q2. saveAndFlush() returns, but the row is not durable. Why?
Flush executes pending SQL, but it doesn't commit the surrounding transaction. If the transaction later rolls back, the database changes can be undone.

**Follow-up:** How would you demonstrate this in a test?

#### Mock Q3. No UPDATE appears after a setter. What do you inspect?
I'd check managed state, whether the value actually changed, the transaction boundary, proxy interception, detached state, read-only configuration, and SQL logging. I wouldn't add save() until I understood why dirty checking wasn't producing the expected result.

**Follow-up:** Why can self-invocation affect @Transactional?

#### Mock Q4. LazyInitializationException occurs during JSON serialization. Fix it.
I'd identify the association being accessed and define the response's data requirements. Then I'd load the required data inside the service transaction and map it to a DTO before returning, rather than relying on serialization to trigger database access.

**Follow-up:** What risks arise from OSIV?

#### Mock Q5. A child row is not linked to its parent. Why?
I'd check the owning side and foreign-key mapping. In a common one-to-many/many-to-one mapping, setting only the parent's collection may not set the child's owning reference. I'd also check cascade if the child is new and inspect the SQL.

**Follow-up:** Does cascade automatically set the owning side?

#### Mock Q6. A paginated query returns duplicate roots. Investigate.
I'd inspect collection joins and SQL row multiplication, then check DISTINCT behavior, ordering, and provider version. For pagination, I'd consider a two-step ID-first query or a projection instead of assuming DISTINCT fixes everything.

**Follow-up:** Why doesn't DISTINCT always solve pagination?

#### Mock Q7. Bulk delete succeeds but a later read returns the deleted entity.
I'd check whether the entity is still managed in the current persistence context or cached elsewhere. Bulk operations bypass normal entity synchronization, so I may need to clear or refresh state and verify cache and transaction behavior.

**Follow-up:** What are the trade-offs of clearing the whole context?

#### Mock Q8. An optimistic-lock exception occurs only in production.
I'd look for overlapping updates, inspect version values and the business operation, and try to reproduce with concurrent transactions. Then I'd decide whether bounded retry, conflict response, atomic SQL, or a different locking strategy fits the invariant.

**Follow-up:** What makes a retry safe?

#### Mock Q9. Database time is low but request latency is high.
I'd compare end-to-end traces, total query count, connection acquisition wait, rows transferred, serialization time, and application work inside the transaction. Fast individual statements don't rule out too many round trips or a long-held connection.

**Follow-up:** Which metrics distinguish pool wait from SQL time?

#### Mock Q10. Repository Page queries are slow.
I'd inspect the count query, indexes, offset depth, and deterministic sorting. If the caller doesn't need a total count, Slice may help; for deep pages, I'd evaluate keyset pagination.

**Follow-up:** When is total count unnecessary?

#### Mock Q11. How do you troubleshoot SQL safely in production?
I'd start with tracing, sampling, and statement metrics, avoiding credentials or sensitive bind values in logs. Where safe, I'd capture representative SQL and examine its execution plan. I also account for the overhead that verbose SQL logging can add.

**Follow-up:** Why can SQL logging itself affect latency?

#### Mock Q12. Design a regression test for an ORM bug.
I'd use the actual JPA provider and, where practical, the production database engine or a compatible test database. I'd assert both result correctness and important query behavior, and add concurrent tests for locking issues. An in-memory database may not reproduce the production engine's SQL or locking semantics.

**Follow-up:** What should be covered in a concurrency regression test?


## Part 11. Roadmap and Study Plan

## Recommended order
1. [JPA and Hibernate fundamentals](#part-1-jpa-and-hibernate-fundamentals)
2. [Entity mapping and relationships](#part-2-entity-mapping-and-relationships)
3. [Entity lifecycle and persistence context](#part-3-entity-lifecycle-and-persistence-context)
4. [Fetching and performance](#part-4-fetching-and-performance)
5. [Transactions and concurrency](#part-5-transactions-and-concurrency)
6. [JPQL, Criteria, and queries](#part-6-jpql-criteria-and-querying)
7. [Spring Data JPA](#part-7-spring-data-jpa)
8. [Caching and optimization](#part-8-caching-and-optimization)
9. [Production troubleshooting](#part-9-production-troubleshooting-scenarios)
10. [Mock interviews](#part-10-mock-interviews)

## Study method
- First explain each concept without notes.
- For each mechanism, state what is portable JPA behavior and what is Hibernate/Spring Data behavior.
- Draw the sequence from repository/service call to persistence context, flush, SQL, and commit.
- For performance topics, predict the SQL, then verify with logs and query statistics.
- For concurrency, reason about two overlapping transactions and the database invariant.
- Revisit questions marked Must Know until you can explain the mechanism, failure mode, and trade-off in two minutes.

## Version checklist
Before relying on an answer in a real project, record the Spring Boot, Spring Data JPA, Hibernate ORM, Jakarta Persistence API, JDBC driver, and database versions. Use the matching reference documentation and migration guide. Boot 3+ uses Jakarta namespaces; Boot 4 compatibility should be checked against the exact release's managed dependency set.

## Part 12. Official References and Version Notes

Use documentation matching the exact dependency versions in the application. The current documentation URL may describe a newer release than the deployed one.

### Jakarta Persistence
- [Jakarta Persistence specifications](https://jakarta.ee/specifications/persistence/)

### Hibernate ORM
- [Hibernate ORM documentation index](https://hibernate.org/orm/documentation/)
- [Hibernate ORM User Guide](https://docs.jboss.org/hibernate/orm/current/userguide/html_single/Hibernate_User_Guide.html)
- [Hibernate ORM releases](https://hibernate.org/orm/releases/)
- [Hibernate ORM migration guides](https://github.com/hibernate/hibernate-orm/tree/main/migration-guide)

### Spring Data JPA and Spring
- [Spring Data JPA reference](https://docs.spring.io/spring-data/jpa/reference/)
- [Spring Data JPA API](https://docs.spring.io/spring-data/jpa/docs/current/api/)
- [Spring Framework transaction management](https://docs.spring.io/spring-framework/reference/data-access/transaction.html)
- [Spring Boot reference](https://docs.spring.io/spring-boot/reference/)

### Version and compatibility discipline
- Spring Boot 3 uses the jakarta.persistence namespace; do not copy javax.persistence examples into Boot 3+ applications.
- Verify Spring Boot 4 / Spring Framework 7 requirements against the exact Boot release's system requirements and managed dependency set. Do not select arbitrary Hibernate and Jakarta Persistence versions independently.
- Hibernate behavior, defaults, dialect support, and APIs change across major releases. Read the matching documentation and migration guide.
- Separate portable Jakarta Persistence guarantees from Hibernate-specific batch settings, statistics, Session APIs, and cache behavior.
- SQL generation, isolation, locking, indexes, and execution plans remain database-specific.

### Research checklist
- [ ] Confirm Spring Boot, Spring Data JPA, Hibernate ORM, Jakarta Persistence API, JDBC driver, and database versions.
- [ ] Verify version-sensitive claims against matching release documentation.
- [ ] Check migration guides before major upgrades.
- [ ] Reproduce performance/concurrency behavior on the production database engine where practical.
