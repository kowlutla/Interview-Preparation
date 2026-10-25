# Hibernate + JPA + Spring Data JPA Interview Preparation

A focused guide for Java backend engineers preparing for experienced and senior interviews.

## Start here

**[Hibernate, JPA, and Spring Data JPA Interview Handbook](Hibernate-JPA-Interview-QA.md)**

This is the primary reading path. It contains 75 curated questions with small, focused Java, JPQL, and configuration snippets where they clarify behavior. Topics include entity lifecycle, relationship mappings, fetching and query performance, transactions and locking, repository semantics, caching, batching, testing, and production diagnosis.

## Recommended study path

1. Read the foundations and lifecycle sections first; they explain why ORM behavior occurs.
2. Study fetching and query performance before memorizing annotations.
3. Review transaction and concurrency behavior with the exact provider/database version in mind.
4. Master the Top 15 list, practise the senior-level production scenarios, then use the last-minute revision checklist and official version-specific references.

## Accuracy and scope

- Jakarta Persistence defines portable behavior; Hibernate adds provider-specific behavior; Spring Data JPA adds repository-layer conventions.
- Generated SQL depends on mapping, provider version, JDBC driver, database, and query plan.
- The guide focuses on interview understanding and production diagnosis, not long code samples.
- Question selection is curated for revision efficiency; it is not a claim of universal company-specific frequency.

## Repository structure

The `Hibernate-JPA` directory is intentionally kept simple: one primary interview Markdown handbook plus this README. The directory was inspected and contains no separate chapter-wise legacy files to consolidate or delete.

- [Spring Data JPA reference](https://docs.spring.io/spring-data/jpa/reference/)
- [Hibernate ORM User Guide](https://docs.jboss.org/hibernate/orm/current/userguide/html_single/Hibernate_User_Guide.html)
- [Jakarta Persistence specifications](https://jakarta.ee/specifications/persistence/)

**Goal:** A technically dependable guide that readers can understand, explain, and revise efficiently before an interview.