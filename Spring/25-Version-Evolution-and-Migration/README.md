# Spring Version Evolution and Migration

## Q&A

### Q1. What is the major Jakarta migration concern in Spring 6 and Boot 3?

The ecosystem moved from Java EE-era javax.* APIs to Jakarta EE jakarta.* APIs for specifications that changed namespace. Application source, servlet containers, validation providers, and third-party libraries must align. Not every javax.* package moved; Java SE packages such as javax.sql remain.

### Q2. What are the important Spring Framework 7 and Boot 4 baselines?

Framework 7 retains a Java 17 baseline and adopts Jakarta EE 11 APIs, including Servlet 6.1, JPA 3.2, and Bean Validation 3.1. Boot 4 requires Java 17 or later and Framework 7.x. The exact Boot patch may require a specific Framework patch, so consult its system-requirements page.

### Q3. What changed for HTTP clients in Framework 7?

Framework 7 deprecates RestTemplate in favor of RestClient. RestClient was introduced in Framework 6.1. For reactive non-blocking and streaming use cases, WebClient remains relevant. Follow the official migration guidance and check custom request factories, interceptors, converters, and error handling.

### Q4. Why is a major Boot upgrade more than a dependency version change?

Major upgrades may change Java baselines, managed dependency versions, configuration properties, removed APIs, servlet/Jakarta integration, security defaults, and observability behavior. Read migration guides and release notes, resolve compatibility first, and run tests against production-like infrastructure.

### Q5. How should Spring Cloud compatibility be managed?

Use the official compatibility guidance and corresponding BOM rather than independently pinning arbitrary Cloud components. The current official matrix maps Spring Cloud 2025.1.x to Boot 4.0.x and, starting with 2025.1.2, Boot 4.1.x; Spring Cloud 2025.0.x maps to Boot 3.5.x. Verify the matrix again before upgrading.

### Q6. How do you plan a safe migration?

Inventory runtime and dependencies, identify deprecated APIs, update one major boundary at a time where practical, compile and run focused tests, validate configuration and startup conditions, and exercise security, persistence, and downstream calls. Roll out progressively with metrics and a rollback plan.

### Q7. How should deprecated APIs be handled?

Determine the replacement and reason for deprecation from official documentation. Avoid mixing legacy examples into new code without a clear migration context. Track deprecations before removal in a later major release.

### Q8. What is the role of a BOM?

A Bill of Materials centralizes compatible dependency versions so related libraries are aligned. Boot's dependency management reduces manual version selection. Overriding managed dependencies can be necessary, but should be evidence-based and tested for binary compatibility.

## Official references

- [Spring Framework versions and support](https://github.com/spring-projects/spring-framework/wiki/Spring-Framework-Versions)
- [Spring Framework 7.0 release notes](https://github.com/spring-projects/spring-framework/wiki/Spring-Framework-7.0-Release-Notes)
- [Spring Boot 4.0 system requirements](https://docs.spring.io/spring-boot/4.0/system-requirements.html)
- [Spring Boot current system requirements](https://docs.spring.io/spring-boot/system-requirements.html)
- [Spring Boot 4.0 migration guide](https://github.com/spring-projects/spring-boot/wiki/Spring-Boot-4.0-Migration-Guide)
- [Spring Cloud compatibility](https://spring.io/projects/spring-cloud)
- [Spring REST clients](https://docs.spring.io/spring-framework/reference/integration/rest-clients.html)

### Q9. What are notable Spring Framework 7 additions worth checking during migration?

Framework 7 introduces new capabilities including programmatic bean registration, core resilience features, API versioning support, HTTP Service Client configuration improvements, and RestTestClient. Availability and exact APIs should be verified against the 7.0 reference and the Boot line managing it; do not backport assumptions to Framework 6.x.

### Q10. What changed in the Jakarta EE baseline for Framework 7?

Framework 7 adopts Jakarta EE 11 APIs, including Servlet 6.1, JPA 3.2, and Bean Validation 3.1. Review servlet containers, persistence providers, validation implementations, and third-party libraries for compatibility before upgrading.

### Q11. Why should API versioning support be treated as a contract design choice?

Framework-level versioning mechanisms can help route or negotiate API versions, but they do not decide compatibility policy, deprecation windows, or client migration. Define the contract and test how versions are selected before adopting the mechanism.

### Q12. What should be checked when moving tests to newer Spring test APIs?

Framework and Boot lines may introduce or deprecate test utilities, mock-bean annotations, and client test support. Verify the exact API and dependency baseline, then test context wiring and the actual request/response path rather than only updating imports.
