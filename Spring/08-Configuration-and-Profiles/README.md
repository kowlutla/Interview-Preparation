# Spring Boot Configuration and Profiles

## Q&A

### Q1. How does application.properties differ from application.yaml?

Both express external configuration. YAML can represent nested structures compactly, while properties files use dotted keys. Choose one consistent style per configuration area, avoid duplicate conflicting definitions, and validate effective values in each deployment environment.

### Q2. How are profile-specific files selected?

Boot supports profile-specific configuration files and profile activation through supported configuration sources. Active profiles influence both property loading and beans annotated with `@Profile`. Avoid embedding production secrets in profile files committed to source control.

### Q3. What are configuration imports used for?

Boot configuration imports allow additional configuration sources to be loaded through its external configuration model. Optionality, location semantics, and integration-specific behavior depend on the source type and Boot version. Use the official reference for the exact syntax rather than assuming arbitrary remote URLs are automatically supported.

### Q4. Why use immutable or validated configuration properties?

A dedicated configuration-properties type documents a coherent group of settings and supports type conversion and validation. Validate required URLs, durations, limits, and pool sizes at startup. Keep secrets out of object string representations and diagnostic logs.

### Q5. How should Kubernetes configuration be organized?

Keep environment-specific values external to the image, inject non-secret settings through the platform's configuration mechanism, and use a managed secret mechanism for credentials. Define resource requests/limits, probes, and shutdown behavior separately from application properties. Configuration changes do not automatically refresh arbitrary singleton beans.

### Q6. Does changing an environment variable update a running Spring bean?

Usually not. Environment variables are read by the process environment, and property binding generally occurs during startup. Dynamic refresh requires an explicitly supported refresh mechanism and compatible consumers; a process restart or rollout is often the safer operational model.

### Q7. What is the danger of logging the Environment or all configuration properties?

The effective Environment can contain passwords, tokens, connection strings, internal hostnames, or personal data contributed by several property sources. Dumping it wholesale can leak secrets into logs and log aggregation systems with broad retention/access. Restrict configuration-inspection endpoints, use secret-aware redaction, and log only explicitly allow-listed non-sensitive keys needed for diagnosis. When debugging precedence, report the winning source/key where safe rather than printing the full value.

## Official references

- [Externalized configuration](https://docs.spring.io/spring-boot/reference/features/external-config.html)
- [Profiles](https://docs.spring.io/spring-boot/reference/features/profiles.html)
