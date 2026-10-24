# Spring Boot Fundamentals

## Q&A

### Q1. What is a Spring Boot starter?

A starter is a convenient dependency descriptor that brings in a curated set of dependencies for a use case. It reduces dependency-selection effort but does not itself implement the feature or guarantee correct production configuration. The managed versions should normally come from the Boot dependency-management model.

### Q2. What does @SpringBootApplication combine?

It combines the common roles of @SpringBootConfiguration, @EnableAutoConfiguration, and @ComponentScan. The package containing the application class becomes the default component-scan root, so placing it too deep can leave controllers, services, repositories, or configuration classes undiscovered. Keep the main class in a sensible root package, and use explicit scan/import configuration only when the application structure requires it. Auto-configuration is separate from component scanning and depends on classpath and configuration conditions.

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

### Q9. What is the difference between a starter and auto-configuration?

A starter declares a curated dependency set. Auto-configuration is conditional configuration that registers infrastructure when prerequisites match. Adding a starter often brings libraries that allow auto-configuration to activate, but the two concepts are distinct.

### Q10. What is the condition evaluation report used for?

It lists auto-configuration conditions and explains why each configuration matched or backed off. Use it to investigate a missing bean, an unexpected default, or behavior that changed after adding a dependency. Correlate the report with the resolved classpath, active profiles, effective properties, and the first meaningful startup exception; do not assume every negative condition is an error.

### Q11. What is the purpose of SpringApplication events?

They expose lifecycle milestones such as environment preparation, context initialization, and readiness. Use the appropriate phase for a listener; an event's existence does not imply every dependency is healthy or every startup task has completed.

### Q12. Why should application startup avoid remote calls in bean constructors?

Constructors run during object creation, so remote calls there can block context initialization, create hidden network dependencies, or fail startup because of a transient outage. Keep constructors focused on establishing valid object state. If startup genuinely depends on a remote service, use an explicit lifecycle component with a bounded timeout, observable failure policy, and documented readiness strategy; do not let an unbounded call stall deployment indefinitely.

### Q13. What is graceful shutdown in Boot?

Graceful shutdown lets supported web servers stop accepting new requests and gives in-flight requests a configured grace period to finish. Configure the shutdown mode and timeout for the selected Boot version, and ensure the deployment platform sends an appropriate termination signal and allows enough termination grace time. This does not guarantee every background task or message handler completes: those components need their own stop, acknowledgement, checkpoint, and retry/recovery policies.

### Q14. How do you package configuration for an executable JAR?

Build one immutable artifact and provide environment-specific settings through supported external configuration sources such as environment variables, mounted configuration, or command-line/property sources. Keep precedence explicit and validate required values at startup. Never package production secrets in the JAR or log the resolved secret-bearing configuration. External configuration should be versioned and deployed with an auditable change process.

### Q15. What is AOT processing in Spring Boot?

Ahead-of-time processing prepares application metadata/code paths for scenarios such as native images and can improve startup/runtime characteristics in supported cases. It introduces constraints around reflection/dynamic behavior and must be tested with the chosen dependencies and build configuration.

### Q16. Does a successful Boot startup guarantee production correctness?

No. Startup proves only that the initialization path completed under the conditions present at that moment. It does not prove authorization rules, downstream timeouts, database pool capacity, message redelivery, schema compatibility, graceful shutdown, or behavior under load. Validate those with integration and failure-path tests, production-like load, and runtime signals such as readiness, latency, error rate, and saturation.

## Official references

- [Spring Boot reference](https://docs.spring.io/spring-boot/reference/)
- [Auto-configuration](https://docs.spring.io/spring-boot/reference/using/auto-configuration.html)
- [Creating an executable JAR](https://docs.spring.io/spring-boot/maven-plugin/packaging.html)
