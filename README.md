# Interview Preparation

A collection of focused interview-preparation handbooks for Java backend engineers targeting mid-level and senior roles. The guides emphasize clear explanations, practical trade-offs, production scenarios, and useful references.

## Start here

Choose a handbook by topic:

- [Java Interview Handbook](Java/Java-Interview-QA.md)
- [Git Interview Revision Handbook](Git/Git-Interview-QA.md) — 55 focused questions with detailed answers across Git internals, branching, collaboration, recovery, releases, and security.
- [Maven & Gradle Interview Handbook](Maven-Gradle/Maven-Gradle-Interview-QA.md) — 45 focused Q&As on Java builds, dependency resolution, multi-module configuration, testing, packaging, troubleshooting, performance, and tool-selection trade-offs.\n- [CI/CD Interview Handbook](CI-CD/CI-CD-Interview-QA.md) — developer-focused delivery for Java/Spring Boot: Jenkins, GitHub Actions, testing, quality gates, immutable artifacts, release safety, database migrations, troubleshooting, and senior-level scenarios.
- [Design Patterns & LLD Interview Handbook](Design-Patterns-LLD/Design-Patterns-LLD-Interview-QA.md) — object-oriented design, SOLID, GoF patterns, UML, concurrency-aware object models, and practical LLD case studies.
- [System Design HLD Interview Handbook](System-Design-HLD/System-Design-HLD-Interview-QA.md) — 300 numbered Q&As, 80 senior-level scenarios, capacity estimation, consistency, scaling, resilience, and multi-stage mock interviews; make this a priority for dedicated HLD rounds.
- [Spring Framework + Spring Boot Interview Handbook](Spring/Spring-Interview-QA.md)
- [Microservices & REST APIs Interview Handbook](Microservices-REST-APIs/Microservices-REST-APIs-Interview-QA.md)
- [Distributed Systems & Event-Driven Architecture Interview Handbook](Distributed-Systems/Distributed-Systems-Interview-QA.md) — consistency, failure handling, consensus, distributed transactions, messaging semantics, sagas, and production scenarios.
- [Hibernate + JPA + Spring Data JPA Interview Handbook](Hibernate-JPA/Hibernate-JPA-Interview-QA.md)
- [SQL & Database Engineering Interview Handbook](SQL-Database/SQL-Database-Interview-QA.md)
- [Elasticsearch & Search Systems Interview Handbook](Elasticsearch/Elasticsearch-Interview-QA.md) — 40 detailed high-value Q&As, 12 production scenarios, 18 Query DSL patterns, relevance, mappings, pagination, production operations, Java integration, and SQL/Kafka synchronization.
- [Redis & Caching Interview Handbook](Redis-Caching/Redis-Caching-Interview-QA.md) — Redis data structures, cache design, distributed coordination, Spring integration, and Redis–database consistency.
- [Apache Kafka Interview Handbook](Kafka/KAFKA_INTERVIEW_GUIDE.md)
- [Apache Spark & Structured Streaming Interview Handbook](Apache-Spark/Apache-Spark-Interview-QA.md) — 120 core Q&As and 40 production scenarios covering Spark SQL/DataFrames, partitions and shuffles, joins/AQE, Java APIs, event-time windows and watermarks, Kafka offsets and delivery guarantees, performance diagnosis, and failure recovery.
- [Apache Iceberg, Avro & Schema Evolution Interview Handbook](Data-Engineering/Iceberg-Avro-Interview-QA.md) — focused senior-level Q&A on Iceberg metadata and snapshots, Avro serialization and Schema Registry, compatibility, and reliable Kafka–Spark data pipelines.
- [Docker, Kubernetes & Helm Interview Handbook](Docker-Kubernetes-Helm/Docker-Kubernetes-Helm-Interview-QA.md) — Java application containerization, runtime behavior, Kubernetes workloads and networking, Helm releases, security basics, and developer-focused production troubleshooting.

## Repository structure

Each topic has one primary Markdown handbook. Open the linked file above to read that topic; there are no separate chapter folders or topic landing pages to navigate.

```text
Interview-Preparation/
├── README.md
├── Git/Git-Interview-QA.md
├── CI-CD/CI-CD-Interview-QA.md
├── System-Design-HLD/System-Design-HLD-Interview-QA.md
├── Java/Java-Interview-QA.md
├── Design-Patterns-LLD/Design-Patterns-LLD-Interview-QA.md
├── Spring/Spring-Interview-QA.md
├── Microservices-REST-APIs/Microservices-REST-APIs-Interview-QA.md
├── Distributed-Systems/Distributed-Systems-Interview-QA.md
├── Hibernate-JPA/Hibernate-JPA-Interview-QA.md
├── SQL-Database/SQL-Database-Interview-QA.md
├── Elasticsearch/Elasticsearch-Interview-QA.md
├── Redis-Caching/Redis-Caching-Interview-QA.md
├── Kafka/KAFKA_INTERVIEW_GUIDE.md
├── Apache-Spark/Apache-Spark-Interview-QA.md
├── Data-Engineering/Iceberg-Avro-Interview-QA.md
└── Docker-Kubernetes-Helm/Docker-Kubernetes-Helm-Interview-QA.md
```

## Suggested study order

1. **Java** — language fundamentals, collections, concurrency, JVM, and modern Java.
2. **Git** — repository internals, collaboration, history safety, releases, and recovery; use it alongside daily development.
3. **Maven & Gradle** — Java build lifecycles, dependency resolution, multi-module builds, testing, packaging, and troubleshooting.\n4. **CI/CD** — build and test automation, artifact traceability, pipeline security, and safe application releases.
5. **Design Patterns & LLD** — object-oriented design, SOLID, design-pattern selection, UML, and realistic low-level design problems.
6. **System Design HLD (highest priority)** — study this in parallel with the remaining guides. Start with the interview method and quality attributes, then capacity estimation, architecture fundamentals, and the senior scenario bank. Revisit it repeatedly; a dedicated HLD round requires deliberate practice, not just reading.
6. **Spring** — dependency injection, Spring Boot, framework behavior, transactions, security, testing, and production behavior.
7. **Microservices & REST APIs** — service boundaries, HTTP semantics, API contracts, resilience, security, and service-level communication.
8. **Distributed Systems & Event-Driven Architecture** — cross-service correctness, consistency, failure handling, coordination, sagas, messaging semantics, and production reliability.
9. **Hibernate + JPA** — persistence context, mappings, fetching, transactions, query performance, and concurrency.
10. **SQL & databases** — query logic, joins, indexes, transactions, isolation, and database-specific behavior.
11. **Elasticsearch & Search Systems** — focused interview Q&A, practical Query DSL examples, relevance, mappings, pagination, production troubleshooting, Java integration, and SQL/Kafka indexing pipelines.
12. **Redis & Caching** — Redis data structures, cache consistency, coordination, Spring integration, and resilience across the cache/database boundary.
13. **Kafka** — broker architecture, topics and partitions, producer/consumer behavior, delivery semantics, and production scenarios.
14. **Apache Spark & Structured Streaming** — study after Kafka and distributed systems: execution, Spark SQL, shuffles and joins, performance, event-time processing, checkpointing, and reliable Kafka pipelines.
15. **Apache Iceberg, Avro & Schema Evolution** — connect event contracts and compatibility with table snapshots, streaming ingestion, compaction, and replay.
16. **Docker, Kubernetes & Helm** — containerize and run the service, understand its runtime constraints, configure workload health and resources, debug connectivity, and deliver releases safely.

The order moves from language and application fundamentals toward distributed behavior and then application delivery. You can study Docker/Kubernetes earlier if your current project requires it. For data-engineering interviews, study Kafka and distributed systems before Spark, then revisit system design and operational reliability.

Use the handbooks as revision material, and verify version-sensitive details against the official documentation for the JDK, framework, ORM provider, database, messaging technology, or deployment platform relevant to the role.
