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

These APIs are version-sensitive. Boot's @MockBean has been widely used to replace beans in a test context; newer Spring Framework lines provide @MockitoBean. Check the precise Boot/Framework version and migration documentation before changing imports or relying on a given annotation. Do not mix APIs blindly across dependency lines.

### Q7. Why can Spring tests become flaky or slow?

Common causes include excessive full-context startup, shared mutable test state, nondeterministic asynchronous work, external dependencies, timing assumptions, and misuse of context caching. Prefer deterministic fixtures, bounded waits, isolated test data, and focused contexts. Do not add arbitrary sleeps to conceal race conditions.

### Q8. How should security tests be designed?

Test anonymous access, authenticated-but-forbidden access, authorized access, malformed credentials, and relevant claim/authority combinations. Verify the actual filter chain and endpoint rules, not only service-level authorization annotations. Include negative tests to prevent accidental exposure.

## Official references

- [Spring Boot testing](https://docs.spring.io/spring-boot/reference/testing/index.html)
- [Spring Framework testing](https://docs.spring.io/spring-framework/reference/testing.html)
- [Testcontainers](https://java.testcontainers.org/)
