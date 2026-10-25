# JPA and Hibernate Fundamentals

## Q1. What is the difference between JPA and Hibernate?
**Must Know**

I would explain it this way: JPA, now called Jakarta Persistence, defines the standard APIs and rules we use to persist Java objects in a relational database. Hibernate is an ORM framework that implements that specification. So when I use annotations such as `@Entity` and APIs such as `EntityManager`, I'm generally working with the JPA contract, while Hibernate does the actual ORM work underneath.

The distinction matters when we want portability. Standard JPA code is easier to move between providers, but that doesn't guarantee identical SQL or performance. Hibernate also offers extra features that are not part of the JPA specification.

**Key distinction:** JPA defines the contract; Hibernate implements it.

## Q2. How do JDBC, JPA, Hibernate, and Spring Data JPA differ?
**Must Know**

I think of these as different abstraction levels. With JDBC, I write SQL, manage parameters, and process result sets myself. JPA gives me entity mapping and a persistence context, so I can work with domain objects instead of manually mapping every row. Hibernate is a common JPA provider that implements those features and generates SQL. Spring Data JPA sits above JPA and reduces repository boilerplate with derived queries, pagination, projections, and repository interfaces.

For a normal business service, Spring Data JPA is convenient. If I need precise control over a specialized query or bulk operation, I may use JPQL, native SQL, or JDBC. The abstraction should simplify the code without hiding a performance or correctness problem.

## Q3. What are EntityManager and Hibernate Session?
**Must Know**

In application code, I usually use JPA's `EntityManager` because it gives me a standard way to find, persist, remove, and query entities. Hibernate's `Session` is its native counterpart and exposes additional Hibernate-specific capabilities. When I need one of those features, I can unwrap the Hibernate Session from the EntityManager.

One important point is that these objects are not general-purpose thread-safe objects. I would not keep one in a singleton service and share it across concurrent requests. In a Spring application, I normally let Spring manage the persistence context and transaction integration.

## Q4. What are EntityManagerFactory and SessionFactory?
**High Value**

The factory creates EntityManager or Session instances. Initializing it involves loading mappings and setting up provider infrastructure, so I would normally create it once for a persistence unit and reuse it rather than create one per request. Hibernate's native equivalent is SessionFactory.

In Spring Boot, the framework usually configures and manages the factory for me. If startup fails because of a mapping or dialect issue, factory initialization is one place where those problems can surface.

## Q5. What is a persistence context?
**Must Know**

I describe the persistence context as Hibernate's managed workspace for entity instances. It keeps track of the entities I'm working with, their persistent identities, and changes that need to be synchronized with the database. If I load an entity and change one of its fields, Hibernate can detect that change because the entity is managed by the context.

It also provides identity consistency: within the same persistence context, loading the same persistent identity normally gives me the same managed Java instance. The context is not itself the database transaction, although Spring applications commonly associate it with a transaction.

## Q6. Can you explain the four entity lifecycle states?
**Must Know**

I usually explain the states by asking whether the persistence context is tracking the object.

- **Transient/new:** I have created the Java object, but it is not managed and has not been made persistent.
- **Managed:** the persistence context tracks it, so Hibernate can detect changes.
- **Detached:** it may represent a database row, but the current context no longer tracks it.
- **Removed:** it is managed but scheduled for deletion when changes are flushed.

For example, an entity can become detached when I clear the EntityManager or close the persistence context. Changing that detached object is only an in-memory change; it does not automatically update the database.

## Q7. How does Hibernate turn entity operations into SQL?
**High Value**

When I work with entities, Hibernate uses the mappings and persistence-context state to determine which database operations are needed. It may queue inserts, updates, and deletes rather than execute SQL at the exact moment I call a Java method. At flush, it synchronizes those changes by generating SQL and executing it through JDBC.

The exact statements depend on the mapping, identifier-generation strategy, dialect, and Hibernate version. When investigating an issue, I check the generated SQL rather than assuming that a repository method maps to one specific statement.

## Q8. What is dirty checking, and why is it useful?
**Must Know**

Dirty checking is how Hibernate detects changes to managed entities and schedules the corresponding database updates without requiring me to call an update method explicitly.

For example, suppose I'm developing an order service. I load an order inside a transaction, change its status from `PENDING` to `CONFIRMED`, and let the transaction complete. Because the order is managed, Hibernate detects the changed state and generates the necessary UPDATE during flush.

The important nuance is that the SQL may not execute immediately when I call the setter. It is generally executed during flush, and the transaction can still roll back afterward. Dirty checking also applies to managed entities; changing a detached object alone will not update the database.

## Q9. What is the difference between flush and commit?
**Must Know**

I separate these concepts because they solve different problems. Flush synchronizes pending persistence-context changes with the database by executing SQL. Commit completes the database transaction. A flush does not make the changes durable by itself, because the transaction can still roll back.

For example, a uniqueness violation may be reported during flush rather than when I set a field. Hibernate may also flush automatically before certain queries or before transaction completion. When debugging, I check both the flush point and the transaction outcome.

## Q10. Does every EntityManager operation execute SQL immediately?
**High Value**

No. If I call `persist()`, Hibernate may make the entity managed and delay the INSERT until flush. A call to `remove()` may schedule a DELETE. A change to a managed entity can also wait until flush. On the other hand, identifier generation and provider details can make some operations execute SQL earlier.

I would not infer SQL timing from the method name alone. I enable SQL logging in a safe environment and observe the actual statements and their timing.

## Q11. Does using JPA make an application database- and provider-independent?
**High Value**

It improves portability, but I wouldn't promise complete independence. JPA standardizes APIs and persistence behavior, while SQL generation, query plans, locking details, and some mapping behavior can vary. Native SQL and provider-specific extensions introduce additional coupling.

If portability is a requirement, I keep provider-specific code isolated and test against the actual target database. I also measure performance on that database, because portable code is not necessarily equally efficient everywhere.

## Q12. How do Spring transactions and the persistence context work together?
**Must Know**

In a typical Spring application, Spring's transaction infrastructure starts or joins a transaction and its JPA integration provides an EntityManager associated with the current transactional execution. Entities loaded or persisted in that context are managed, so their changes can be synchronized when the transaction flushes.

I also watch for Spring's proxy behavior: calling a transactional method from another method on the same object may bypass the proxy and therefore not apply the expected transaction advice. If an update unexpectedly doesn't happen, I check transaction boundaries, managed state, and the generated SQL.

## Official references
- [Jakarta Persistence](https://jakarta.ee/specifications/persistence/)
- [Hibernate ORM documentation](https://hibernate.org/orm/documentation/)
- [Hibernate User Guide](https://docs.jboss.org/hibernate/orm/current/userguide/html_single/Hibernate_User_Guide.html)
- [Spring Data JPA](https://docs.spring.io/spring-data/jpa/reference/)
