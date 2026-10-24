# Spring Version Evolution and Migration

## Q&A

### Q1. What is the major Jakarta migration concern in Spring 6 and Boot 3?

The ecosystem moved from Java EE-era `javax.*` APIs to Jakarta EE `jakarta.*` APIs for specifications that changed namespace. Application source, servlet containers, validation providers, and third-party libraries must align. Not every package beginning with javax moved; Java SE packages such as javax.sql remain.

### Q2. Why is a major Boot upgrade more than a dependency version change?

Major upgrades may change Java baselines, managed dependency versions, configuration properties, removed APIs, servlet/Jakarta integration, security defaults, and observability behavior. Read migration guides and release notes, resolve compatibility first, and run tests against production-like infrastructure.

### Q3. How should Spring Cloud compatibility be managed?

Use the official compatibility guidance and corresponding BOM rather than independently pinning arbitrary Cloud components. Validate the selected Boot release train, then test discovery, gateway routing, load balancing, and resilience integrations. Compatibility is a release-specific matrix, not a permanent one-to-one mapping.

### Q4. How do you plan a safe migration?

Inventory runtime and dependencies, identify deprecated APIs, update one major boundary at a time where practical, compile and run focused tests, validate configuration and startup conditions, and exercise security, persistence, and downstream calls. Roll out progressively with metrics and a rollback plan.

### Q5. How should deprecated APIs be handled?

Determine the replacement and the reason for deprecation from official documentation. Avoid mixing legacy examples into new code without a clear migration context. A deprecation warning may not require an immediate emergency change, but it should be tracked before the API is removed in a later major release.

### Q6. What is the role of a BOM?

A Bill of Materials centralizes compatible dependency versions so related libraries are aligned. Boot's dependency management reduces manual version selection. Overriding managed dependencies can be necessary, but should be evidence-based and tested for binary compatibility.

## Official references

- [Spring Framework support](https://spring.io/projects/spring-framework#support)
- [Spring Boot reference and migration guidance](https://docs.spring.io/spring-boot/reference/)
- [Spring Cloud compatibility](https://spring.io/projects/spring-cloud)
