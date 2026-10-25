# Spring Data JPA

## Q1. What does Spring Data JPA add on top of JPA?
**Must Know**

JPA gives me the standard persistence APIs, while Spring Data JPA reduces the repository code I need to write. It can derive queries from method names, support explicit queries, provide pagination and sorting, and return projections. It also integrates repositories with Spring's transaction and dependency-injection infrastructure.

It doesn't replace Hibernate or the database. When an application has an N+1 issue, stale managed state, or unexpected SQL, I still need to understand the underlying JPA provider behavior.

## Q2. What does save() actually do?
**Must Know**

Spring Data JPA checks whether it considers the entity new. It generally calls `EntityManager.persist()` for a new entity and `merge()` for an existing one. The new-entity decision can depend on a nullable version property, identifier state, or a custom `Persistable` implementation.

The important detail is that merge returns the managed instance; it does not make the passed detached object managed. So if I need to use the merged entity afterward, I use the return value. I also don't assume save always means an immediate SQL UPDATE.

## Q3. Why might save() trigger a SELECT?
**High Value**

That isn't automatically a bug. If Spring Data decides an entity is not new, it may use merge semantics, and merging can require Hibernate to load the managed state before copying values. This can be surprising when an entity uses an application-assigned identifier.

I inspect the identifier and version fields and how Spring Data determines newness. If needed, I configure the newness strategy appropriately, but only after verifying the entity's lifecycle semantics.

## Q4. Are repository methods transactional by default?
**Must Know**

Spring Data repository methods have default transaction behavior: read operations are generally configured as read-only transactions, while write operations are transactional. Declared query methods and custom repository implementations can have details that need checking in the relevant reference.

For a business operation that updates several entities or calls multiple repositories, I normally put the transaction boundary at the service layer. That makes the intended unit of work explicit instead of relying on separate repository calls to define it.

## Q5. What does @Transactional(readOnly = true) do?
**High Value**

I use it to communicate that a transaction is intended for reading and to allow applicable framework/provider optimizations. However, I don't treat it as a universal guarantee that a write is impossible or as a security control. The exact effect depends on the transaction manager, provider, and database configuration.

If read-only behavior matters for correctness or access control, I enforce it at the appropriate database or application layer and measure any performance benefit.

## Q6. What does @Modifying do?
**Must Know**

I put `@Modifying` on a Spring Data `@Query` method that performs an update or delete rather than returning selected entities. It normally needs a transaction. Because the query operates directly on database rows, entities already managed by the persistence context may not reflect the new values.

I consider whether the context needs to be flushed before the query and cleared or refreshed afterward. Those choices depend on what the caller does next.

## Q7. How does Spring Data parse derived query method names?
**High Value**

Spring Data interprets the method name as a property path and predicate. That works well for straightforward queries, but a method name becomes hard to read when it encodes many nested properties, joins, and OR conditions.

When the method stops communicating intent clearly, I switch to `@Query` or Specifications. I also rely on application startup and tests to catch invalid property paths rather than assuming the method name is correct.

## Q8. What are repository projections?
**High Value**

A projection lets the repository return the data shape the caller needs instead of always returning a full entity. Interface projections can be concise, and class-based projections can define an explicit DTO shape. They are useful for read APIs and summary views.

I verify the SQL rather than assuming projections always select only the expected columns. Nested or open projections can have different loading implications.

## Q9. How does Spring Data auditing work?
**High Value**

Auditing can populate fields such as created time, last-modified time, and the user responsible for a change. I need to enable the auditing infrastructure and configure the current auditor if user identity is tracked.

I distinguish this convenience from a complete compliance audit trail. If I need immutable historical records of every change, I may need a dedicated audit model, event history, or database-level auditing.

## Q10. When would you write a custom repository implementation?
**High Value**

I use a custom repository fragment when a query or persistence operation is too specialized for derived methods or a simple `@Query`. Examples include dynamic native queries or a persistence path that needs careful control over query construction.

I keep the custom code focused and test its transaction and SQL behavior. I don't recreate the whole repository abstraction just to support one specialized method.

## Q11. How do you design pagination for a large table?
**Must Know**

I start by asking whether the caller needs a total count. If not, Slice may avoid an expensive count query. I ensure ordering is deterministic and that the relevant filter and sort columns have appropriate indexes. Deep offset pagination can become expensive, so keyset pagination is worth considering when the access pattern supports it.

Pageable is a convenient API, but it cannot make an inefficient database query efficient by itself.

## Q12. Does saveAndFlush() commit the transaction?
**Must Know**

No. `saveAndFlush()` applies save semantics and forces pending changes to be synchronized with the database. The surrounding transaction can still roll back, so flush is not the same as commit.

I use it when I specifically need SQL to execute at that point—for example, to surface a constraint failure earlier—not as a general way to make repository writes more reliable.

## Official references
- [Spring Data JPA reference](https://docs.spring.io/spring-data/jpa/reference/)
- [Spring Data Commons reference](https://docs.spring.io/spring-data/commons/reference/)
