# Hibernate + JPA + Spring Data JPA Interview Q&A Handbook

A high-signal, category-wise handbook for Java backend engineers with roughly 4–5 years of experience preparing for senior and product-company interviews. It prioritizes persistence-context mechanics, generated SQL, fetching and performance, transactions, concurrency, repository behavior, and production diagnosis over question volume.

> **Scope:** 107 numbered questions across the core chapters, plus three focused mock interviews (12 questions each). Answers are written in a spoken interview style: direct explanation, practical context, mechanism, and relevant caveats. The material uses small snippets only where they help rather than full applications. The conversational rewrite is complete; an independent line-by-line technical audit and external-link crawl remain separate quality tasks.


## Answer-writing standard

The answers are intended to sound like a developer explaining a concept to an interviewer, not a glossary or documentation excerpt. A strong answer should:

1. Start with a direct explanation in natural language.
2. Explain how the behavior works, not just name it.
3. Use a small, realistic example when it improves understanding.
4. Mention a meaningful caveat, trade-off, or common production failure.
5. Avoid repeated question text, an “Answer:” label, unnecessary formal filler, and large tutorial-style code samples.

Not every question needs all five elements. Straightforward questions should stay concise; high-value questions should go deeper enough that the candidate can defend the answer and handle follow-ups.

## Prerequisites
- Java classes, collections, exceptions, and basic generics.
- SQL, relational modeling, transactions, indexes, and basic query plans.
- Spring dependency injection and proxy-based transaction basics.
- Basic REST/service-layer architecture.

## Top 25 Questions to Master First

1. [JPA versus Hibernate](01-JPA-and-Hibernate-Fundamentals/01-Fundamentals.md#q1-what-is-the-difference-between-jpa-and-hibernate)
2. [JDBC vs JPA vs Hibernate vs Spring Data JPA](01-JPA-and-Hibernate-Fundamentals/01-Fundamentals.md#q2-how-do-jdbc-jpa-hibernate-and-spring-data-jpa-differ)
3. [Persistence context](01-JPA-and-Hibernate-Fundamentals/01-Fundamentals.md#q5-what-is-a-persistence-context)
4. [Entity lifecycle states](01-JPA-and-Hibernate-Fundamentals/01-Fundamentals.md#q6-explain-transient-managed-detached-and-removed-states)
5. [Dirty checking](01-JPA-and-Hibernate-Fundamentals/01-Fundamentals.md#q8-what-is-dirty-checking)
6. [Flush vs commit](01-JPA-and-Hibernate-Fundamentals/01-Fundamentals.md#q9-what-is-the-difference-between-flush-and-commit)
7. [persist() vs merge()](03-Entity-Lifecycle-and-Persistence-Context/01-Lifecycle.md#q1-what-is-the-difference-between-persist-and-merge)
8. [Owning side and mappedBy](02-Entity-Mapping-and-Relationships/01-Mapping.md#q3-what-is-the-owning-side-of-a-relationship)
9. [Cascade vs orphanRemoval](02-Entity-Mapping-and-Relationships/01-Mapping.md#q5-how-do-cascade-and-orphanremoval-differ)
10. [Entity equals/hashCode](02-Entity-Mapping-and-Relationships/01-Mapping.md#q8-what-makes-entity-equals-and-hashcode-difficult)
11. [N+1 query problem](04-Fetching-and-Performance/01-Fetching.md#q1-what-is-the-n1-query-problem)
12. [Fixing N+1 queries](04-Fetching-and-Performance/01-Fetching.md#q2-how-do-you-fix-n1-queries)
13. [Fetch join limitations](04-Fetching-and-Performance/01-Fetching.md#q3-what-is-a-fetch-join-and-what-are-its-limitations)
14. [LazyInitializationException](04-Fetching-and-Performance/01-Fetching.md#q6-what-causes-lazyinitializationexception)
15. [Collection fetch join and pagination](04-Fetching-and-Performance/01-Fetching.md#q8-why-can-collection-fetch-joins-break-pagination)
16. [Spring @Transactional with JPA](05-Transactions-and-Concurrency/01-Transactions.md#q1-how-does-transactional-interact-with-jpa)
17. [Optimistic vs pessimistic locking](05-Transactions-and-Concurrency/01-Transactions.md#q2-optimistic-versus-pessimistic-locking)
18. [@Version and lost updates](05-Transactions-and-Concurrency/01-Transactions.md#q3-how-does-version-prevent-lost-updates)
19. [Bulk update/delete implications](05-Transactions-and-Concurrency/01-Transactions.md#q9-what-are-risks-of-bulk-jpql-updatesdeletes)
20. [JPQL vs SQL](06-JPQL-Criteria-and-Queries/01-Queries.md#q1-jpql-versus-sql)
21. [Page vs Slice](06-JPQL-Criteria-and-Queries/01-Queries.md#q5-page-versus-slice)
22. [Spring Data JPA save() semantics](07-Spring-Data-JPA/01-Repositories.md#q2-what-does-save-actually-do)
23. [save() causing SELECT](07-Spring-Data-JPA/01-Repositories.md#q3-why-can-save-issue-a-select)
24. [First-level vs second-level cache](08-Caching-and-Optimization/01-Caching.md#q1-first-level-versus-second-level-cache)
25. [Systematic Hibernate performance incident response](09-Production-Scenarios/01-Troubleshooting.md#q15-how-do-you-approach-a-hibernate-performance-incident)

## Category navigation

- [00 — Learning roadmap](00-Roadmap/learning-roadmap.md)
- [01 — JPA and Hibernate fundamentals](01-JPA-and-Hibernate-Fundamentals/01-Fundamentals.md)
- [02 — Entity mapping and relationships](02-Entity-Mapping-and-Relationships/01-Mapping.md)
- [03 — Entity lifecycle and persistence context](03-Entity-Lifecycle-and-Persistence-Context/01-Lifecycle.md)
- [04 — Fetching and performance](04-Fetching-and-Performance/01-Fetching.md)
- [05 — Transactions and concurrency](05-Transactions-and-Concurrency/01-Transactions.md)
- [06 — JPQL, Criteria, and queries](06-JPQL-Criteria-and-Queries/01-Queries.md)
- [07 — Spring Data JPA](07-Spring-Data-JPA/01-Repositories.md)
- [08 — Caching and optimization](08-Caching-and-Optimization/01-Caching.md)
- [09 — Production troubleshooting scenarios](09-Production-Scenarios/01-Troubleshooting.md)
- [10 — Mock interviews](10-Mock-Interviews/01-Core-Mock.md), [senior performance](10-Mock-Interviews/02-Senior-Performance-Mock.md), [production troubleshooting](10-Mock-Interviews/03-Production-Mock.md)
- [11 — Official references and version notes](11-References/official-documentation.md)

## Recommended study order
Follow the roadmap: fundamentals → mappings → lifecycle → fetching/performance → transactions/concurrency → queries → Spring Data JPA → caching → production scenarios → mocks. Master the Top 25 first, then use the production scenarios to practice diagnosis rather than memorizing fixes.

## Importance labels
- **Must Know** — core mechanics that commonly appear in Java backend interviews.
- **High Value** — practical engineering and senior-level trade-offs.
- **Advanced** — less frequent or provider-specific depth worth understanding.

## Version policy
Use the dependency versions actually managed by the application's Spring Boot release. As of 25 October 2026, Hibernate's official release matrix lists Hibernate ORM 7.4 as the latest stable series and maps it to Jakarta Persistence 3.2; it lists 7.2 with Spring Boot 4.0 and 7.4 with Spring Boot 4.1, while Hibernate 6.6 is listed for Spring Boot 3.4–3.5. Hibernate ORM 8.0 is still in development and targets Jakarta Persistence 4.0. These are release-matrix signals, not permission to override Spring Boot's dependency management: verify the exact Boot patch release and support policy before changing dependencies. [Hibernate release/compatibility matrix](https://hibernate.org/orm/releases/)

Boot 3+ uses the jakarta.persistence namespace. Do not mix javax.persistence and jakarta.persistence APIs. Provider behavior, generated SQL, locking, and execution plans can be version- and database-specific.

## Official references
- [Jakarta Persistence specifications](https://jakarta.ee/specifications/persistence/)
- [Hibernate ORM documentation and release matrix](https://hibernate.org/orm/documentation/)
- [Hibernate migration guides](https://hibernate.org/orm/documentation/migrate/)
- [Spring Data JPA reference](https://docs.spring.io/spring-data/jpa/reference/)
- [Spring Framework transaction management](https://docs.spring.io/spring-framework/reference/data-access/transaction.html)
- [Spring Boot reference](https://docs.spring.io/spring-boot/reference/)

## Progress checklist
- [x] Dedicated handbook directory and category navigation.
- [x] Approximately 100 high-signal questions authored across the core categories.
- [x] Top 25 linked to actual question locations.
- [x] Three mock interviews with concise model answers and follow-ups.
- [x] Official documentation and version-awareness notes included.
- [ ] Independent line-by-line technical review of every answer.
- [ ] Automated local Markdown/link audit and external URL crawl.
- [ ] Validate version-sensitive examples against the exact dependency set used in a target application.
