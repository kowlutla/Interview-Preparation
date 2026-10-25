# Hibernate + JPA + Spring Data JPA Interview Preparation

A focused guide for Java backend engineers preparing for experienced and senior interviews.

## Start here

**[Hibernate, JPA, and Spring Data JPA Interview Handbook](Hibernate-JPA-Interview-QA.md)**

This is the primary reading path. It contains 50 curated questions covering persistence contexts, entity lifecycle, mappings and relationships, fetching and query performance, transactions and locking, Spring Data repository behavior, batching, testing, and production troubleshooting.

## Recommended study path

1. Read the foundations and lifecycle sections first; they explain why ORM behavior occurs.
2. Study fetching and query performance before memorizing annotations.
3. Review transaction and concurrency behavior with the exact provider/database version in mind.
4. Practice the senior-level production scenarios, then finish with the last-minute revision list and version-specific official references.

## Accuracy and scope

- Jakarta Persistence defines portable behavior; Hibernate adds provider-specific behavior; Spring Data JPA adds repository-layer conventions.
- Generated SQL depends on mapping, provider version, JDBC driver, database, and query plan.
- The guide focuses on interview understanding and production diagnosis, not long code samples.
- Question selection is curated for revision efficiency; it is not a claim of universal company-specific frequency.

## Extended legacy material

Legacy chapter material has not been deleted as part of this update. Before any cleanup, inventory the actual files and compare their unique questions and scenarios against the consolidated guide; preserve anything valuable before removal.

- [Spring Data JPA reference](https://docs.spring.io/spring-data/jpa/reference/)
- [Hibernate ORM User Guide](https://docs.jboss.org/hibernate/orm/current/userguide/html_single/Hibernate_User_Guide.html)
- [Jakarta Persistence specifications](https://jakarta.ee/specifications/persistence/)

**Goal:** A technically dependable guide that readers can understand, explain, and revise efficiently before an interview.