# Spring Configuration

## Q&A

### Q1. What is the difference between @Configuration and @Bean?

@Configuration marks a class that declares bean definitions; @Bean marks a factory method whose return value is managed by the container. In full configuration mode, inter-bean method calls are intercepted so container semantics are preserved. Lite configuration modes have different direct method-call behavior, so avoid relying on it without understanding the mode.

### Q2. When should you use @ConfigurationProperties instead of @Value?

Use @ConfigurationProperties for grouped, type-safe configuration with relaxed binding and optional validation. @Value is suitable for a small number of individual values or expressions. For complex application settings, a dedicated configuration properties type is easier to document, test, and evolve.

### Q3. What is relaxed binding in Spring Boot?

Boot's configuration binder can map equivalent property naming styles to Java property names, subject to its binding rules. For example, external kebab-case configuration can bind to a camelCase Java property. Follow the canonical property names in documentation rather than inventing names based on guesswork.

### Q4. What does @Profile do?

@Profile conditions bean registration on active profiles. Profiles are useful for selecting environment-specific implementations, but they should not become a substitute for explicit configuration management or for keeping secrets out of source control.

### Q5. How should configuration properties be validated?

Bind values to a dedicated properties class and apply Jakarta Bean Validation constraints when validation support is configured. Fail fast with a clear startup error for invalid required configuration instead of allowing a malformed URL, missing credential, or impossible pool size to fail during live traffic.

### Q6. How do you reason about property precedence?

Spring Boot aggregates property sources and applies a defined precedence order, where higher-priority sources override lower-priority ones. The exact source list and behavior are versioned; consult the Boot externalized-configuration reference. When debugging, inspect the effective value and its source without exposing secrets in logs.

### Q7. Is @PropertySource the main way to configure a Boot application?

Usually not. Boot's externalized configuration supports application.properties/application.yaml, profile-specific files, environment variables, command-line arguments, and imported configuration sources. @PropertySource remains useful for specific cases, but it is not a replacement for understanding Boot's configuration loading model.

### Q8. How should secrets be supplied to production applications?

Use an approved secret-management system or platform-provided secret mechanism, restrict access, rotate credentials, and avoid committing secrets or logging resolved values. Environment variables can be convenient but are not inherently a complete secrets-management strategy.

## Official references

- [Spring Boot externalized configuration](https://docs.spring.io/spring-boot/reference/features/external-config.html)
- [Configuration properties](https://docs.spring.io/spring-boot/reference/features/external-config.html#features.external-config.typesafe-configuration-properties)
