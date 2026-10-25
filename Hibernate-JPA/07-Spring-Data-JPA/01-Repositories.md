# Spring Data JPA

## Q1. What does Spring Data JPA add to JPA?
**Must Know**

It provides repository abstractions, derived query parsing, pagination/sorting, projections, auditing, and Spring integration. It delegates persistence semantics to the JPA provider and database. Repository convenience does not remove the need to understand entity state, flush, locking, and generated SQL.

## Q2. What does save() actually do?
**Must Know**

Spring Data chooses persist or merge based on its new-entity detection strategy, which can consider a nullable version property, identifier state, or Persistable customization. With merge, the returned object is the managed instance and may differ from the argument. Save is not always an SQL UPDATE and is not required for an already managed entity.

## Q3. Why can save() issue a SELECT?
**High Value**

New-entity detection may classify an object as existing, leading to merge semantics; merge may load managed state before copying values. Inspect ID/version configuration and newness strategy. For assigned IDs, Persistable or a suitable version strategy may be appropriate.

## Q4. Are repository methods transactional by default?
**Must Know**

Spring Data repository methods have default transaction behavior; read operations are generally read-only transactional and write operations transactional. Declared query methods and custom implementations have details that should be checked in the reference. Service-level transactions are often clearer for business operations spanning multiple repositories.

## Q5. When is @Transactional(readOnly = true) useful?
**High Value**

It communicates intent and can enable optimizations in the transaction manager/provider/database stack. It is not a universal guarantee that writes are impossible, nor a security boundary. Measure effects with the actual configuration.

## Q6. What does @Modifying do?
**Must Know**

It marks an @Query method as an update/delete operation. It generally needs a transaction and may bypass managed entity state. Clear or refresh affected entities when later reads in the same context must see the database changes.

## Q7. How does query method parsing work?
**High Value**

Spring Data parses method names into property paths and predicates. It works well for simple queries but becomes hard to read with many joins or OR conditions. Use @Query or Specifications for complex logic and verify property names.

## Q8. What are repository projections?
**High Value**

Projections define the returned data shape. Interface and class-based projections can reduce loading, but exact SQL depends on query type and projection structure. Inspect generated SQL, especially for nested or open projections.

## Q9. How does Spring Data auditing work?
**High Value**

Auditing can populate created/modified timestamps and principals using annotations and configured infrastructure. Enable auditing and provide the current auditor where needed. It is not automatically a complete compliance audit log.

## Q10. When is a custom repository implementation appropriate?
**High Value**

Use a custom repository fragment for specialized query behavior or persistence operations that do not fit derived methods or @Query. Keep it focused and test SQL and transaction semantics rather than rebuilding the entire abstraction.

## Q11. How should pagination work for large tables?
**Must Know**

Use stable ordering, avoid total counts when unnecessary, and consider keyset pagination for deep pages. Index filter/order columns appropriately. Pageable cannot make an inefficient offset/count query cheap on its own.

## Q12. Does saveAndFlush() commit?
**Must Know**

No. It invokes repository save semantics and forces pending SQL to flush. The surrounding transaction can still roll back. Use it for deliberate early synchronization, not as a substitute for transaction design.

## Official references
- [Spring Data JPA reference](https://docs.spring.io/spring-data/jpa/reference/)
- [Spring Data Commons reference](https://docs.spring.io/spring-data/commons/reference/)
