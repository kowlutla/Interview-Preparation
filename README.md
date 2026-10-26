# Interview Preparation

A collection of focused interview-preparation handbooks for Java backend engineers targeting mid-level and senior roles. The guides emphasize clear explanations, practical trade-offs, production scenarios, and useful references.

## Start here

Choose a handbook by topic:

- [Java Interview Handbook](Java/Java-Interview-QA.md)
- [Design Patterns & LLD Interview Handbook](Design-Patterns-LLD/Design-Patterns-LLD-Interview-QA.md) — object-oriented design, SOLID, GoF patterns, UML, concurrency-aware object models, and practical LLD case studies.
- [Spring Framework + Spring Boot Interview Handbook](Spring/Spring-Interview-QA.md)
- [Microservices & REST APIs Interview Handbook](Microservices-REST-APIs/Microservices-REST-APIs-Interview-QA.md)
- [Distributed Systems & Event-Driven Architecture Interview Handbook](Distributed-Systems/Distributed-Systems-Interview-QA.md) — consistency, failure handling, consensus, distributed transactions, messaging semantics, sagas, and production scenarios.
- [Hibernate + JPA + Spring Data JPA Interview Handbook](Hibernate-JPA/Hibernate-JPA-Interview-QA.md)
- [SQL & Database Engineering Interview Handbook](SQL-Database/SQL-Database-Interview-QA.md)
- [Redis & MySQL Interview Handbook](Redis-MySQL/Redis-MySQL-Interview-QA.md) — Redis data structures and caching, MySQL/InnoDB internals, Spring integration, consistency, and production troubleshooting.
- [Apache Kafka Interview Handbook](Kafka/KAFKA_INTERVIEW_GUIDE.md)
- [Docker, Kubernetes & Helm Interview Handbook](Docker-Kubernetes-Helm/Docker-Kubernetes-Helm-Interview-QA.md) — Java application containerization, runtime behavior, Kubernetes workloads and networking, Helm releases, security basics, and developer-focused production troubleshooting.

## Repository structure

Each topic has one primary Markdown handbook. Open the linked file above to read that topic; there are no separate chapter folders or topic landing pages to navigate.

```text
Interview-Preparation/
├── README.md
├── Java/Java-Interview-QA.md
├── Design-Patterns-LLD/Design-Patterns-LLD-Interview-QA.md
├── Spring/Spring-Interview-QA.md
├── Microservices-REST-APIs/Microservices-REST-APIs-Interview-QA.md
├── Distributed-Systems/Distributed-Systems-Interview-QA.md
├── Hibernate-JPA/Hibernate-JPA-Interview-QA.md
├── SQL-Database/SQL-Database-Interview-QA.md
├── Redis-MySQL/Redis-MySQL-Interview-QA.md
├── Kafka/KAFKA_INTERVIEW_GUIDE.md
└── Docker-Kubernetes-Helm/Docker-Kubernetes-Helm-Interview-QA.md
```

## Suggested study order

1. **Java** — language fundamentals, collections, concurrency, JVM, and modern Java.
2. **Design Patterns & LLD** — object-oriented design, SOLID, design-pattern selection, UML, and realistic low-level design problems.
3. **Spring** — dependency injection, Spring Boot, framework behavior, transactions, security, testing, and production behavior.
4. **Microservices & REST APIs** — service boundaries, HTTP semantics, API contracts, resilience, security, and service-level communication.
5. **Distributed Systems & Event-Driven Architecture** — cross-service correctness, consistency, failure handling, coordination, sagas, messaging semantics, and production reliability.
6. **Hibernate + JPA** — persistence context, mappings, fetching, transactions, query performance, and concurrency.
7. **SQL & databases** — query logic, joins, indexes, transactions, isolation, and database-specific behavior.
8. **Redis & MySQL** — caching patterns, data structures, SQL performance, InnoDB transactions, and consistency across the cache/database boundary.
9. **Kafka** — broker architecture, topics and partitions, producer/consumer behavior, delivery semantics, and production scenarios.
10. **Docker, Kubernetes & Helm** — containerize and run the service, understand its runtime constraints, configure workload health and resources, debug connectivity, and deliver releases safely.

The order moves from language and application fundamentals toward distributed behavior and then application delivery. You can study Docker/Kubernetes earlier if your current project requires it.

Use the handbooks as revision material, and verify version-sensitive details against the official documentation for the JDK, framework, ORM provider, database, messaging technology, or deployment platform relevant to the role.
