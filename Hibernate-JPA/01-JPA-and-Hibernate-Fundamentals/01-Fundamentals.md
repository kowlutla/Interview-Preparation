# JPA and Hibernate Fundamentals

## Q1. What is the difference between JPA and Hibernate?
**Must Know**

Jakarta Persistence (JPA) is a specification for object-relational mapping and persistence APIs. Hibernate ORM implements that specification and adds provider-specific features. Standard APIs improve portability, but SQL generation and performance can still vary by provider, database, dialect, and version.

**Key distinction:** JPA defines the contract; Hibernate implements it.

## Q2. How do JDBC, JPA, Hibernate, and Spring Data JPA differ?
**Must Know**

JDBC is the low-level API for executing SQL and processing result sets. JPA defines entity mapping and persistence-context semantics. Hibernate implements JPA and translates entity operations and queries into SQL. Spring Data JPA adds repositories, query derivation, projections, and pagination on top of JPA; it does not replace the provider or database.

## Q3. What are EntityManager and Hibernate Session?
**Must Know**

EntityManager is the standard JPA interface for managing entities and queries. Hibernate Session is its native API and exposes Hibernate-specific features. An EntityManager/Session is generally not thread-safe and should not be shared across concurrent request threads.

## Q4. What are EntityManagerFactory and SessionFactory?
**High Value**

They create EntityManager and Session instances and are relatively expensive to initialize. Applications typically keep one factory per persistence unit for the application lifetime. In Spring Boot, let framework configuration manage them rather than creating one per request.

## Q5. What is a persistence context?
**Must Know**

It is the unit-of-work identity map that tracks managed entity instances. Within one context, a persistent identity normally corresponds to one managed object. It supports dirty checking and synchronizes changes during flush. It is related to, but not identical to, a database transaction.

## Q6. Explain transient, managed, detached, and removed states.
**Must Know**

Transient/new objects are not managed. Managed objects are tracked and their changes can be detected. Detached objects have persistent identity but are no longer tracked by that context. Removed objects are managed and scheduled for deletion. Detaching is not deleting.

## Q7. How does Hibernate translate entity changes into SQL?
**High Value**

Hibernate uses mapping metadata and persistence-context state to determine required inserts, updates, and deletes. SQL is commonly deferred until flush, then executed through JDBC. Exact SQL, action ordering, and batching depend on mappings, identifier strategy, dialect, and provider version.

## Q8. What is dirty checking?
**Must Know**

Dirty checking detects changes to managed entities and schedules updates without an explicit update call. Hibernate can use snapshot comparison or enhanced tracking strategies. It does not automatically persist changes made only to detached objects.

## Q9. What is the difference between flush and commit?
**Must Know**

Flush synchronizes pending changes by issuing SQL. Commit completes the database transaction. Flushed statements can still be rolled back. Flush can happen before commit, before a query under AUTO flush mode, or when requested explicitly, so errors may appear before commit.

## Q10. Is every EntityManager operation executed immediately?
**High Value**

No. Persist, remove, and dirty changes may be queued until flush. Find can return an entity already present in the persistence context. SQL timing also depends on identifier generation and provider behavior. Use SQL logs to confirm timing rather than assuming one API call equals one statement.

## Q11. What does JPA portability mean in practice?
**High Value**

Standard APIs and annotations reduce provider coupling but do not guarantee identical SQL, performance, or database semantics. Keep provider-specific features isolated and test with the target database. Dialect, schema, and isolation behavior still matter.

## Q12. How do Spring transactions and the persistence context relate?
**Must Know**

Spring's JPA integration typically binds an EntityManager to the current transaction/thread. Entities become managed when loaded or persisted in that context; @Transactional does not make arbitrary objects managed. Proxy-based transaction advice can also be bypassed by self-invocation.

## Official references
- [Jakarta Persistence](https://jakarta.ee/specifications/persistence/)
- [Hibernate ORM documentation](https://hibernate.org/orm/documentation/)
- [Hibernate User Guide](https://docs.jboss.org/hibernate/orm/current/userguide/html_single/Hibernate_User_Guide.html)
- [Spring Data JPA](https://docs.spring.io/spring-data/jpa/reference/)
