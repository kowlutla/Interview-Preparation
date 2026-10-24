# Spring Boot Fundamentals

## Q&A

### Q1. What is a Spring Boot starter?

A starter is a convenient dependency descriptor that brings in a curated set of dependencies for a use case. It reduces dependency-selection effort but does not itself implement the feature or guarantee correct production configuration. The managed versions should normally come from the Boot dependency-management model.

### Q2. What does @SpringBootApplication combine?

It combines the common roles of @SpringBootConfiguration, @EnableAutoConfiguration, and @ComponentScan. Its package location defines the default component-scan root, so placing the main class in an overly narrow package can leave application components undiscovered.

### Q3. How does auto-configuration work?

Boot evaluates candidate configuration classes and conditional annotations against the classpath, existing beans, properties, and application environment. Auto-configuration provides sensible defaults and often backs off when the application defines its own bean. It is conditional configuration, not hidden magic that overrides every user choice.

### Q4. What does @ConditionalOnMissingBean mean?

It allows a configuration contribution when no matching bean is present at the point the condition is evaluated. It is commonly used to make auto-configuration replaceable by application configuration. Ordering and bean-definition registration matter, so custom auto-configuration should be tested in realistic contexts.

### Q5. What is the purpose of an executable Boot JAR?

It packages an application with the runtime dependencies and a launcher structure so it can be started with the Java launcher without separately assembling a classpath. It is not the same as a native image, and the executable packaging format does not remove the need for runtime configuration and operational controls.

### Q6. What is the difference between CommandLineRunner and ApplicationRunner?

Both provide callbacks after the application context has started. CommandLineRunner receives raw string arguments; ApplicationRunner receives parsed application arguments through ApplicationArguments. Keep startup work bounded, observable, and idempotent because a slow runner can delay the application from becoming ready.

### Q7. How do you diagnose why an auto-configuration did not apply?

Use Boot's condition evaluation report, enable the relevant debug diagnostics, and inspect the classpath, property values, bean definitions, and active profiles. The report explains why conditions matched or did not match; it should be paired with the actual exception and environment rather than treated as a fix by itself.

### Q8. Does Boot require an embedded web server?

No. Boot supports non-web applications as well as servlet and reactive web applications. The application type and dependencies influence which infrastructure is configured. Avoid adding web starters to background jobs that do not need HTTP endpoints.

## Official references

- [Spring Boot reference](https://docs.spring.io/spring-boot/reference/)
- [Auto-configuration](https://docs.spring.io/spring-boot/reference/using/auto-configuration.html)
- [Creating an executable JAR](https://docs.spring.io/spring-boot/maven-plugin/packaging.html)

### Q9. What is the difference between a starter and auto-configuration?

A starter declares a curated dependency set. Auto-configuration is conditional configuration that registers infrastructure when prerequisites match. Adding a starter often brings libraries that allow auto-configuration to activate, but the two concepts are distinct.

### Q10. What is the condition evaluation report used for?

It explains which auto-configuration conditions matched or failed. Use it alongside startup logs, effective properties, and the root exception to understand missing or unexpected beans.

### Q11. What is the purpose of SpringApplication events?

They expose lifecycle milestones such as environment preparation, context initialization, and readiness. Use the appropriate phase for a listener; an event's existence does not imply every dependency is healthy or every startup task has completed.

### Q12. Why should application startup avoid remote calls in bean constructors?

Constructors run during object creation and can block context initialization or fail the whole startup due to transient dependency issues. Keep construction lightweight and place required startup checks in a bounded, observable lifecycle phase.

### Q13. What is graceful shutdown in Boot?

It allows supported servers to stop accepting new requests and gives in-flight work a configured opportunity to finish. Background executors, message consumers, and scheduled tasks need their own lifecycle and recovery policies.

### Q14. How do you package configuration for an executable JAR?

Package the application artifact separately from environment-specific settings and inject configuration through supported external sources. Avoid rebuilding the artifact for each environment and do not bundle production secrets.

### Q15. What is AOT processing in Spring Boot?

Ahead-of-time processing prepares application metadata/code paths for scenarios such as native images and can improve startup/runtime characteristics in supported cases. It introduces constraints around reflection/dynamic behavior and must be tested with the chosen dependencies and build configuration.

### Q16. Does a successful Boot startup guarantee production correctness?

No. It verifies only the initialization path exercised. Runtime authorization, dependency timeouts, pool saturation, message redelivery, shutdown, and load behavior require tests and operational validation.
