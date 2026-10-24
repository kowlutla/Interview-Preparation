# Spring Boot Testing

## Q&A

### Q1. What is the difference between a unit test and an integration test?

A unit test isolates a small unit and usually replaces collaborators with test doubles. An integration test verifies that components work together across real framework configuration or infrastructure boundaries. Use the smallest test scope that proves the behavior, but retain integration coverage for wiring, serialization, transactions, and database-specific behavior.

### Q2. What does @SpringBootTest do?

It creates an application context using Boot's test support and can load a broad slice of the application. It is useful for integration behavior but may be slower and more coupled to configuration than a focused test. Avoid using it for every simple service unit test.

### Q3. What is a test slice?

A slice annotation loads a focused subset of application infrastructure for a testing purpose, such as MVC or JPA. It improves feedback speed and helps isolate concerns, but components outside the slice may need explicit mocks or test configuration. A slice test does not prove that the entire production context starts.

### Q4. When should you use MockMvc?

MockMvc exercises Spring MVC request handling through the test framework without requiring a live HTTP server. It can verify routing, serialization, validation, status codes, and security filters when configured. Use a real-server test when network/server behavior itself matters.

### Q5. What is Testcontainers useful for?

Testcontainers starts disposable infrastructure such as a database or broker for tests, allowing verification against a real engine rather than relying solely on an in-memory substitute. It improves fidelity but adds runtime and environment requirements; keep test data deterministic and ensure containers are cleaned up.

### Q6. What is the difference between @MockBean and @MockitoBean?

`@MockBean` was the established Spring Boot test annotation for replacing beans in an application context. It was deprecated for removal in Spring Boot 3.4 in favor of Spring Framework's `@MockitoBean`, and Boot 4 removes `@MockBean`/`@SpyBean` support. Use `@MockitoBean` (and `@MockitoSpyBean` for spies) on modern Framework/Boot lines, but retain the older annotation only when maintaining a compatible legacy Boot line.

The migration is not always a mechanical import swap: on Framework 6.2/7.0, use the supported test-class field or type-level declarations rather than declaring mocks inside configuration classes. Framework 7.1 prerelease documentation adds constructor-parameter support for these annotations; do not assume that API is available on the stable 7.0 line. Follow the exact Boot/Framework migration guide and verify test-context replacement semantics.

### Q7. Why can Spring tests become flaky or slow?

Common causes include excessive full-context startup, shared mutable test state, nondeterministic asynchronous work, external dependencies, timing assumptions, and misuse of context caching. Prefer deterministic fixtures, bounded waits, isolated test data, and focused contexts. Do not add arbitrary sleeps to conceal race conditions.

### Q8. How should security tests be designed?

Test anonymous access, authenticated-but-forbidden access, authorized access, malformed credentials, and relevant claim/authority combinations. Verify the actual filter chain and endpoint rules, not only service-level authorization annotations. Include negative tests to prevent accidental exposure.

### Q9. When should a test use a real HTTP server instead of MockMvc?

Use a real-server test when transport behavior, actual server configuration, socket handling, compression, or deployed HTTP integration is part of the risk. MockMvc is faster for MVC handler behavior but does not exercise the full network stack.

### Q10. How should asynchronous tests avoid timing races?

Use bounded futures, latches, Awaitility-style conditions, or controlled executors rather than arbitrary sleeps. Assert a clear completion condition and include diagnostics on timeout so the test reveals whether work was never scheduled, rejected, or stalled.

### Q11. What is the risk of @DirtiesContext?

It evicts a context from the test cache and can substantially increase suite runtime. Use it when a test truly corrupts shared context state; prefer isolating mutable state and resetting the specific resource.

### Q12. How do you test a migration against an existing schema?

Start from a known baseline schema and run the real migration sequence against the production database engine, including representative data where migration behavior depends on volume or constraints. Verify both final schema and transformed data, not just that the migration command exits successfully.

### Q13. How should test secrets and environment configuration be handled?

Use test-only credentials and ephemeral infrastructure, avoid committing real secrets, and make configuration deterministic. Ensure logs and test reports do not include credentials or personal data when failures print configuration/request payloads.

## Official references

- [Spring Boot testing](https://docs.spring.io/spring-boot/reference/testing/index.html)
- [Spring Framework testing](https://docs.spring.io/spring-framework/reference/testing.html)
- [Testcontainers](https://java.testcontainers.org/)
