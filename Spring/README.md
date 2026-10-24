# Spring Framework + Spring Boot Interview Q&A Handbook

A version-aware handbook for Java backend engineers preparing for practical and senior-level interviews. It focuses on how Spring behaves, why its abstractions exist, how to diagnose failures, and how to make production trade-offs—not memorizing annotations.

> **Status:** Initial handbook foundation is being authored. Checked items indicate files exist, not that every topic has completed independent technical review. Question and exercise totals will be updated only after content is present and counted.

## Audience and prerequisites

Designed for engineers with roughly 2–6 years of Java backend experience. You should be comfortable with Java classes/interfaces, exceptions, collections, generics, HTTP, SQL, dependency management, and basic testing. Review the separate [Java handbook](../Java/README.md) for general Java language and JVM fundamentals.

## Recommended study path

1. [Roadmap and version compatibility](00-Roadmap/learning-roadmap.md)
2. [Spring fundamentals](01-Spring-Fundamentals/README.md)
3. [IoC and dependency injection](02-IoC-and-Dependency-Injection/README.md)
4. Spring Boot startup and configuration
5. MVC/REST, validation, and error design
6. Transactions and Spring Data integration
7. Security and testing
8. HTTP clients, reactive programming, caching, async, events, and messaging
9. Observability, Spring Cloud, deployment, troubleshooting, and internals
10. Coding exercises, production scenarios, and mock interviews

## Difficulty guidance

- **Beginner:** Explain purpose and basic behavior in your own words.
- **Intermediate:** Implement a feature and explain the configuration.
- **Advanced:** Diagnose proxy, lifecycle, transaction, concurrency, or version edge cases.
- **Senior:** Compare alternatives, state assumptions, explain failure modes, and defend operational trade-offs.

## Category navigation

| Area | Content |
|---|---|
| Roadmap | [Learning roadmap](00-Roadmap/learning-roadmap.md), [version compatibility](00-Roadmap/version-compatibility.md), [interview strategy](00-Roadmap/interview-preparation-strategy.md) |
| Fundamentals | [Spring fundamentals](01-Spring-Fundamentals/README.md) |
| IoC and DI | [Container and dependency injection](02-IoC-and-Dependency-Injection/README.md) |
| Exercises | [Coding exercises](26-Coding-Exercises/README.md) |
| Scenarios | [Production scenarios](27-Real-World-Scenarios/README.md) |
| Mock interviews | [Mock interviews](28-Mock-Interviews/README.md) |
| References | [Official references](30-References/README.md) |

Other category directories will be added as chapters are authored; navigation must be expanded alongside them.

## Version policy

Examples target modern Spring programming models and identify version-specific behavior explicitly. Do not assume that a Spring Framework major version and Spring Boot major version have identical support windows. Check official compatibility and migration documents before selecting dependencies. Avoid copying legacy WebSecurityConfigurerAdapter examples into new projects; modern servlet security uses SecurityFilterChain. The javax.* versus jakarta.* distinction is a major migration boundary, not a cosmetic rename.

## Progress tracker

- [x] Handbook structure and study path
- [ ] All requested topic categories authored
- [ ] At least 500 distinct Q&As counted
- [ ] At least 75 coding exercises completed
- [ ] At least 75 scenario questions completed
- [ ] Five complete mock interviews completed
- [ ] Full link, version, and technical review completed

## Official documentation

- [Spring Framework Reference](https://docs.spring.io/spring-framework/reference/)
- [Spring Boot Reference](https://docs.spring.io/spring-boot/reference/)
- [Spring Security Reference](https://docs.spring.io/spring-security/reference/)
- [Spring Data JPA Reference](https://docs.spring.io/spring-data/jpa/reference/)
- [Spring Cloud Reference](https://docs.spring.io/spring-cloud/docs/current/reference/html/)
- [Spring for Apache Kafka Reference](https://docs.spring.io/spring-kafka/reference/)
- [Spring Batch Reference](https://docs.spring.io/spring-batch/reference/)
- [Spring guides](https://spring.io/guides)
