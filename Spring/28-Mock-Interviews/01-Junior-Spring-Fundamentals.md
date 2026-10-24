# Mock Interview 1 — Junior Java Developer

## 1. What is Spring?

Spring is a Java framework that provides dependency injection and infrastructure for common application needs. It helps organize collaborators and integrations without forcing every application class to inherit from framework base classes.

**Follow-up:** Does every Java object become a Spring bean? No; it must be registered or created through a managed path.

## 2. What is dependency injection?

A class receives its dependencies from outside instead of constructing them itself. Constructor injection makes required collaborators explicit and simplifies unit testing.

**Follow-up:** Why prefer constructor injection for required dependencies? It makes invalid construction harder and supports final fields.

## 3. What is a bean?

A bean is an object managed by a Spring container, including creation, dependency wiring, and lifecycle callbacks as configured.

**Follow-up:** What is the difference between a bean and a POJO? A POJO is an ordinary Java object; it is a bean only when managed by Spring.

## 4. What does @Service mean?

It is a stereotype annotation for a service-layer component and supports component scanning. It communicates intent but does not implement business logic by itself.

**Follow-up:** How does it differ from @Component? @Component is generic; @Service describes a service role.

## 5. Spring Framework versus Spring Boot?

Spring Framework provides core capabilities; Boot adds opinionated setup, starters, conditional auto-configuration, packaging, and operational integration.

**Follow-up:** Does Boot remove the need to configure security? No. Defaults and auto-configuration do not replace a deliberate security policy.

## 6. What is a singleton bean?

One instance per bean definition per container. It does not mean one instance globally across the JVM, and it does not guarantee thread safety.

**Follow-up:** How do you keep a singleton service safe? Prefer stateless behavior and avoid unsynchronized shared mutable fields.

## 7. What is component scanning?

Spring discovers eligible annotated components in configured packages. The main Boot application package is commonly the default scan root.

**Follow-up:** What happens if a component is outside the scan root? It may not be registered unless imported or explicitly scanned.

## 8. How would you test a service?

For business logic, construct the service directly with fake dependencies and test outputs and failure cases. Use Spring context tests when wiring or framework integration is the behavior under test.

**Follow-up:** Why not use @SpringBootTest for every unit test? It is slower and tests more infrastructure than a pure unit test needs.
