# Hibernate + JPA + Spring Data JPA Learning Roadmap

## Recommended order
1. [JPA and Hibernate fundamentals](../01-JPA-and-Hibernate-Fundamentals/01-Fundamentals.md)
2. [Entity mapping and relationships](../02-Entity-Mapping-and-Relationships/01-Mapping.md)
3. [Entity lifecycle and persistence context](../03-Entity-Lifecycle-and-Persistence-Context/01-Lifecycle.md)
4. [Fetching and performance](../04-Fetching-and-Performance/01-Fetching.md)
5. [Transactions and concurrency](../05-Transactions-and-Concurrency/01-Transactions.md)
6. [JPQL, Criteria, and queries](../06-JPQL-Criteria-and-Queries/01-Queries.md)
7. [Spring Data JPA](../07-Spring-Data-JPA/01-Repositories.md)
8. [Caching and optimization](../08-Caching-and-Optimization/01-Caching.md)
9. [Production troubleshooting](../09-Production-Scenarios/01-Troubleshooting.md)
10. [Mock interviews](../10-Mock-Interviews/01-Core-Mock.md)

## Study method
- First explain each concept without notes.
- For each mechanism, state what is portable JPA behavior and what is Hibernate/Spring Data behavior.
- Draw the sequence from repository/service call to persistence context, flush, SQL, and commit.
- For performance topics, predict the SQL, then verify with logs and query statistics.
- For concurrency, reason about two overlapping transactions and the database invariant.
- Revisit questions marked Must Know until you can explain the mechanism, failure mode, and trade-off in two minutes.

## Version checklist
Before relying on an answer in a real project, record the Spring Boot, Spring Data JPA, Hibernate ORM, Jakarta Persistence API, JDBC driver, and database versions. Use the matching reference documentation and migration guide. Boot 3+ uses Jakarta namespaces; Boot 4 compatibility should be checked against the exact release's managed dependency set.
