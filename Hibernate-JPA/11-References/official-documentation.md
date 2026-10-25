# Official Documentation and References

Use documentation matching the exact dependency versions in the application. The current documentation URL may describe a newer release than the deployed one.

## Jakarta Persistence
- [Jakarta Persistence specifications](https://jakarta.ee/specifications/persistence/)

## Hibernate ORM
- [Hibernate ORM documentation index](https://hibernate.org/orm/documentation/)
- [Hibernate ORM User Guide](https://docs.jboss.org/hibernate/orm/current/userguide/html_single/Hibernate_User_Guide.html)
- [Hibernate ORM releases](https://hibernate.org/orm/releases/)
- [Hibernate ORM migration guides](https://github.com/hibernate/hibernate-orm/tree/main/migration-guide)

## Spring Data JPA and Spring
- [Spring Data JPA reference](https://docs.spring.io/spring-data/jpa/reference/)
- [Spring Data JPA API](https://docs.spring.io/spring-data/jpa/docs/current/api/)
- [Spring Framework transaction management](https://docs.spring.io/spring-framework/reference/data-access/transaction.html)
- [Spring Boot reference](https://docs.spring.io/spring-boot/reference/)

## Version and compatibility discipline
- Spring Boot 3 uses the jakarta.persistence namespace; do not copy javax.persistence examples into Boot 3+ applications.
- Verify Spring Boot 4 / Spring Framework 7 requirements against the exact Boot release's system requirements and managed dependency set. Do not select arbitrary Hibernate and Jakarta Persistence versions independently.
- Hibernate behavior, defaults, dialect support, and APIs change across major releases. Read the matching documentation and migration guide.
- Separate portable Jakarta Persistence guarantees from Hibernate-specific batch settings, statistics, Session APIs, and cache behavior.
- SQL generation, isolation, locking, indexes, and execution plans remain database-specific.

## Research checklist
- [ ] Confirm Spring Boot, Spring Data JPA, Hibernate ORM, Jakarta Persistence API, JDBC driver, and database versions.
- [ ] Verify version-sensitive claims against matching release documentation.
- [ ] Check migration guides before major upgrades.
- [ ] Reproduce performance/concurrency behavior on the production database engine where practical.
