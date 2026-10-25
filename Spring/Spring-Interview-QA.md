# Spring Framework, Spring Boot, and Ecosystem Interview Handbook

**Audience:** Java backend engineers with 5–6 years of experience preparing for senior backend interviews.

**Purpose:** A topic-organized, interview-first revision handbook covering Spring Framework, Spring Boot, and the related Spring ecosystem. Answers explain the mechanism, practical implications, common failure modes, and trade-offs rather than giving one-line definitions. Use short snippets only when they clarify behavior.

**Version note (reviewed 25 October 2026):** The official documentation currently lists Spring Framework 7.0.10 and 6.2.19 as stable lines; Spring Boot 4.1.x is also documented, with its own Java and Framework requirements. Do not mix version numbers independently: Boot manages a tested dependency set, and Spring Cloud release trains have their own compatibility matrix. In an interview, state the version family you have used and check migration guides before making version-specific claims. [Spring Framework versions](https://docs.spring.io/spring-framework/reference/) · [Spring Boot system requirements](https://docs.spring.io/spring-boot/system-requirements.html) · [Spring Boot reference](https://docs.spring.io/spring-boot/reference/).

## How to use this handbook

- For a first pass, study the mechanism and the reason a feature exists.
- For senior interviews, add a production example, a failure mode, and a trade-off to each answer.
- For troubleshooting questions, explain what you would inspect first and why.
- Treat repeated concepts across topics as deliberate reinforcement where they affect different layers.
- This is an interview-preparation reference, not a claim that every sample is a runnable application.

## Contents

1. [Spring Fundamentals](#spring-fundamentals)
2. [IoC Container and Dependency Injection](#ioc-container-and-dependency-injection)
3. [Spring Beans and Scopes](#spring-beans-and-scopes)
4. [Spring Configuration](#spring-configuration)
5. [Spring AOP and Proxies](#spring-aop-and-proxies)
6. [Spring Boot Fundamentals](#spring-boot-fundamentals)
7. [Auto-Configuration and Application Startup](#auto-configuration-and-application-startup)
8. [Spring Boot Configuration and Profiles](#spring-boot-configuration-and-profiles)
9. [Spring MVC and REST APIs](#spring-mvc-and-rest-apis)
10. [Exception Handling and API Error Design](#exception-handling-and-api-error-design)
11. [Spring Data Access and Transactions](#spring-data-access-and-transactions)
12. [Spring Data JPA Integration](#spring-data-jpa-integration)
13. [Spring Security](#spring-security)
14. [Spring Boot Testing](#spring-boot-testing)
15. [REST Clients and Inter-Service Communication](#rest-clients-and-inter-service-communication)
16. [Reactive Programming and WebFlux](#reactive-programming-and-webflux)
17. [Actuator and Observability](#actuator-and-observability)
18. [Spring Caching](#spring-caching)
19. [Asynchronous Processing and Scheduling](#asynchronous-processing-and-scheduling)
20. [Spring Events and Messaging](#spring-events-and-messaging)
21. [Spring Cloud and Microservices Integration](#spring-cloud-and-microservices-integration)
22. [Deployment and Production Readiness](#deployment-and-production-readiness)
23. [Performance and Troubleshooting](#performance-and-troubleshooting)
24. [Spring Internals and Advanced Topics](#spring-internals-and-advanced-topics)
25. [Spring Version Evolution and Migration](#spring-version-evolution-and-migration)
26. [Advanced Spring Ecosystem and Specialized Modules](#advanced-spring-ecosystem-and-specialized-modules)
27. [Senior-level production troubleshooting](#senior-level-production-troubleshooting)
28. [Last-minute revision checklist](#last-minute-revision-checklist)


## Spring Fundamentals


### Q1. What is the Spring Framework, and what problems does it solve?

Spring Framework is a modular Java framework that provides infrastructure for dependency management, declarative transaction support, web request handling, data access integration, and cross-cutting concerns. It reduces application-level coupling by letting code depend on contracts and collaborators supplied by a container. It does not remove the need to understand Java, HTTP, databases, or deployment; it provides consistent abstractions around them.

### Q2. What is the difference between Spring Framework and Spring Boot?

Spring Framework supplies core infrastructure such as the IoC container, AOP, transaction abstractions, and Spring MVC/WebFlux. Spring Boot builds on Spring Framework and streamlines setup with dependency management, starter dependencies, conditional auto-configuration, executable packaging, and operational features such as Actuator. Boot does not replace Spring; it configures and packages Spring applications.

### Q3. What is Inversion of Control (IoC)?

IoC means a component does not control every detail of creating and locating its collaborators. Responsibility is delegated to an external mechanism, such as Spring's container. Dependency Injection is a common way to implement IoC: the container supplies dependencies to an object.

### Q4. What is Dependency Injection (DI), and why is constructor injection usually preferred?

DI supplies an object's dependencies from outside rather than having the object construct or globally locate them. Constructor injection makes required dependencies explicit, supports immutable fields, and makes plain unit tests straightforward.

```java
@Service
public class OrderService {
    private final OrderRepository repository;

    public OrderService(OrderRepository repository) {
        this.repository = repository;
    }
}
```

This works when OrderRepository is registered as a bean and there is an unambiguous candidate. Field injection hides required dependencies and makes standalone construction harder.

### Q5. What is the difference between BeanFactory and ApplicationContext?

BeanFactory is the basic container abstraction for managing bean definitions and creating beans. ApplicationContext builds on it with application-oriented features such as event publication, message resolution, resource loading, and integration with framework infrastructure. Most application code should use the context indirectly rather than manually calling container lookups.

### Q6. What is a Spring bean?

A bean is an object instantiated, assembled, and otherwise managed by a Spring IoC container. Beans can be registered by component scanning, @Bean factory methods, imports, or programmatic registration. Not every object in a Spring application is a bean; objects created with ordinary new are not automatically managed.

### Q7. What is a POJO, and must every POJO be a Spring bean?

A POJO is an ordinary Java object not forced to inherit from a framework-specific base class. Spring can manage POJOs, but it does not automatically manage every POJO in the process. An object must be registered or created through a managed factory path to participate in container lifecycle and injection.

### Q8. What is the Spring ecosystem?

The ecosystem includes Spring Framework, Spring Boot, Spring Security, Spring Data, Spring Cloud, Spring Batch, and Spring for Apache Kafka. These projects have distinct responsibilities and release compatibility rules; choosing one does not imply every other project is automatically configured or compatible.

### Q9. What is tight coupling, and how does Spring help reduce it?

Tight coupling occurs when a class directly constructs or depends on a concrete collaborator, making replacement and isolated testing difficult. Spring can inject an abstraction or selected implementation, allowing configuration to choose collaborators. DI helps manage coupling but does not automatically make a poorly designed interface or architecture modular.

### Q10. Does using Spring automatically make an application loosely coupled or testable?

No. Spring provides tools that can support loose coupling, but design quality still depends on boundaries, interfaces, dependency direction, and side effects. A service with dozens of unrelated injected dependencies can remain highly coupled even though every dependency comes from the container.

### Q11. What happens at a high level when a Spring application starts?

The application creates an environment and context, registers bean definitions, processes configuration and factory post-processors, creates eligible beans, applies post-processors and proxies, and completes context refresh. Exact steps vary by application type, framework version, lazy initialization, and custom infrastructure. Startup logs and Boot's condition evaluation report help investigate failures.

### Q12. How should you decide whether to use Spring for a small Java program?

Use Spring when dependency management, configuration, integration, or operational infrastructure meaningfully helps. For a tiny utility with few collaborators, a plain Java design may be simpler. Framework use has a learning and startup cost; the goal is not to make every program container-managed.

### Official references
- [Spring Framework overview](https://docs.spring.io/spring-framework/reference/overview.html)
- [Core technologies and IoC container](https://docs.spring.io/spring-framework/reference/core/beans.html)
- [Spring Boot reference](https://docs.spring.io/spring-boot/reference/)

---

## IoC Container and Dependency Injection


### Q13. How does Spring resolve a constructor dependency?

Spring inspects registered bean definitions and the target constructor's parameter types, then resolves candidates using type, qualifiers, primary designation, and other metadata. If a required dependency has no candidate or multiple unresolved candidates, context creation normally fails with a dependency-resolution exception.

### Q14. What is the difference between @Component, @Service, @Repository, and @Controller?

All are stereotype annotations used to identify components for scanning. @Service communicates service-layer intent; @Repository communicates persistence-layer intent and participates in persistence exception translation when the relevant infrastructure applies; @Controller identifies a web controller. @Component is the generic stereotype. Their semantic roles improve readability and framework integration, not business logic by themselves.

### Q15. What is the difference between @Bean and @Component?

@Component is placed on a class discovered by component scanning. @Bean is placed on a method in a configuration class, and the method's return value is registered as a bean. @Bean is particularly useful for configuring third-party classes that cannot be annotated directly or for explicit factory logic.

### Q16. What do @Configuration and @ComponentScan do?

@Configuration marks a class as a source of bean definitions, often declared with @Bean methods. @ComponentScan instructs Spring to discover eligible stereotype-annotated classes in selected packages. In modern Boot applications, @SpringBootApplication includes component scanning rooted at the application's package, so package placement matters.

### Q17. How do @Primary and @Qualifier differ?

@Primary gives one candidate preference when multiple beans match a single-valued injection point. @Qualifier narrows the candidate set using qualifier metadata. A qualifier can express a semantic role and is generally clearer when a consumer specifically needs a named kind of implementation.

### Q18. Is field injection always invalid?

No, but constructor injection is usually preferable for mandatory dependencies because it makes requirements visible and enables final fields. Field injection can be convenient in some framework-managed test fixtures, but it complicates plain unit tests and can hide dependency growth.

### Q19. What is a circular dependency?

A circular dependency exists when bean A requires bean B and bean B requires bean A, directly or through a chain. Constructor-injection cycles cannot be resolved by constructing either object first and normally fail during context creation. Setter/field cycles may be resolvable in some singleton scenarios, but relying on that is fragile; redesigning responsibilities is usually better.

### Q20. What is the difference between BeanFactoryPostProcessor and BeanPostProcessor?

A BeanFactoryPostProcessor works with bean definitions and container metadata before ordinary bean instantiation is complete. A BeanPostProcessor can inspect or wrap bean instances before and after initialization; infrastructure such as proxy creation uses post-processing. Confusing these phases can lead to early-instantiation bugs.

### Q21. When would you use ObjectProvider<T>?

Use ObjectProvider<T> when a dependency is intentionally optional, lazily resolved, or needs candidate iteration at runtime. It exposes operations such as getIfAvailable() and stream access without requiring immediate resolution at the injection point. Do not use it merely to hide an unclear dependency design.

### Q22. What does @Lazy do?

@Lazy can defer bean initialization or dependency resolution, depending on where it is applied. It can reduce startup work when a bean is not always needed, but it moves failures to first use and can make startup health look better than actual request readiness. It is not a general fix for circular dependencies.

### Q23. How can you register a bean conditionally?

Use an appropriate condition such as Boot's @ConditionalOnClass, @ConditionalOnProperty, or @ConditionalOnMissingBean for Boot auto-configuration, or profile/conditional configuration suited to the application. Conditions should be explicit and testable. The exact condition annotations available depend on the framework and Boot modules in use.

### Q24. Why should dependencies generally point toward abstractions?

Depending on an abstraction reduces the consumer's knowledge of a concrete implementation and can make replacement or testing easier. The abstraction must still represent a stable, coherent contract; adding interfaces for every class without a meaningful boundary can create ceremony rather than decoupling.

### Q25. How do you debug NoSuchBeanDefinitionException?

Check that the implementation is on the runtime classpath, component scanning covers its package, conditions and profiles permit registration, and the required bean type is assignable. Inspect the condition evaluation report in Boot applications and verify that configuration is loaded in the failing test or runtime context.

### Q26. How do you debug NoUniqueBeanDefinitionException?

Identify all candidates of the requested type, then decide whether the consumer needs a specific semantic implementation, a default, or all implementations. Use a qualifier for a specific choice, @Primary for a legitimate default, or inject a collection when multiple implementations are intended. Avoid resolving ambiguity by accidental bean names.

### Q27. What is @Import useful for?

@Import brings additional configuration classes or supported import selectors/registrars into a context. It is useful for modular configuration and libraries that provide explicit opt-in setup. It does not mean all classes in an arbitrary package are scanned.

### Official references
- [Container overview](https://docs.spring.io/spring-framework/reference/core/beans/basics.html)
- [Dependency injection](https://docs.spring.io/spring-framework/reference/core/beans/dependencies/factory-collaborators.html)
- [Annotation-based container configuration](https://docs.spring.io/spring-framework/reference/core/beans/annotation-config.html)
- [Spring Boot auto-configuration](https://docs.spring.io/spring-boot/reference/using/auto-configuration.html)

---

## Spring Beans and Scopes


### Q28. What does singleton scope mean in Spring?

Singleton means one bean instance per bean definition per Spring IoC container. It is not the same as enforcing the GoF Singleton pattern across the entire JVM. Two application contexts can each create their own instance.

### Q29. Does singleton scope make a bean thread-safe?

No. Singleton scope controls how many instances the container creates; it does not synchronize method calls or protect mutable state. A singleton service may process many requests concurrently, so prefer stateless services and keep per-request values in method-local variables or explicit request-scoped collaborators. If shared mutable state is necessary, protect it with an appropriate concurrency mechanism and consider whether it belongs in the service at all. Also verify that collaborators such as formatters, clients, and caches are safe for concurrent use.

### Q30. What is prototype scope?

Prototype scope asks Spring to create a new instance each time that bean is requested from the container. Spring manages construction and initialization, but it does not generally track prototype instances through destruction callbacks after handing them to the caller. The caller must release resources when necessary.

### Q31. What happens when a prototype bean is injected into a singleton?

The prototype is normally resolved when the singleton is created, so the singleton retains that one instance. To obtain a fresh instance on demand, use ObjectProvider, a suitable provider/factory, or a scoped proxy when appropriate.

### Q32. When are request and session scopes available?

They are web-aware scopes tied to an active web request or session and require the relevant web application context. Accessing them outside the required context can fail. They should not be confused with thread-local state that remains valid across asynchronous execution.

### Q33. What are @PostConstruct and @PreDestroy used for?

They mark initialization and destruction callbacks on managed beans, subject to lifecycle and scope semantics. Initialization callbacks should not assume every other bean is fully operational in every ordering scenario, and destruction callbacks are not guaranteed for every abrupt process termination.

### Q34. What is lazy initialization, and what is its trade-off?

Lazy initialization delays creation until a bean is needed. It can reduce startup time and work, but can defer configuration errors and latency to the first request that triggers the bean. Use it based on startup measurements and readiness semantics rather than as a default cure.

### Q35. What is a scoped proxy?

A scoped proxy provides a stable proxy object that resolves the underlying target according to its scope, allowing a bean with a shorter scope to be injected into a longer-lived bean. It is useful for request-scoped dependencies injected into singleton services, but callers must understand proxy behavior and thread/request boundaries.

### Q36. What are common bean lifecycle pitfalls?

Common issues include performing slow network calls during initialization, triggering premature bean creation from post-processors, relying on destruction callbacks during forced termination, and assuming proxies are present inside initialization callbacks. Keep startup work bounded and make lifecycle assumptions explicit.

### Q37. Can Spring override a bean definition with another bean of the same name?

Behavior depends on configuration and Boot settings; bean-definition overriding is not a safe way to resolve accidental conflicts. Prefer unique, meaningful names and explicit conditions or qualifiers. Verify the selected Boot version's defaults when debugging duplicate-bean registration.

### Official references
- [Bean scopes](https://docs.spring.io/spring-framework/reference/core/beans/factory-scopes.html)
- [Bean lifecycle](https://docs.spring.io/spring-framework/reference/core/beans/factory-nature.html)

---

## Spring Configuration


### Q38. What is the difference between @Configuration and @Bean?

@Configuration marks a class that declares bean definitions; @Bean marks a factory method whose return value is managed by the container. In full configuration mode, inter-bean method calls are intercepted so container semantics are preserved. Lite configuration modes have different direct method-call behavior, so avoid relying on it without understanding the mode.

### Q39. When should you use @ConfigurationProperties instead of @Value?

Use @ConfigurationProperties for grouped, type-safe configuration with relaxed binding and optional validation. @Value is suitable for a small number of individual values or expressions. For complex application settings, a dedicated configuration properties type is easier to document, test, and evolve.

### Q40. What is relaxed binding in Spring Boot?

Boot's configuration binder can map equivalent property naming styles to Java property names, subject to its binding rules. For example, external kebab-case configuration can bind to a camelCase Java property. Follow the canonical property names in documentation rather than inventing names based on guesswork.

### Q41. What does @Profile do?

@Profile conditions bean registration on active profiles, allowing environment-specific beans or configuration classes to be selected. For example, a local implementation can be enabled for development while a production implementation is selected in production. Profiles can be combined, so inspect the effective active profiles when a bean is unexpectedly missing. Profiles select configuration; they are not a secret-management mechanism or a replacement for validating required properties.

### Q42. How should configuration properties be validated?

Bind values to a dedicated properties class and apply Jakarta Bean Validation constraints when validation support is configured. Fail fast with a clear startup error for invalid required configuration instead of allowing a malformed URL, missing credential, or impossible pool size to fail during live traffic.

### Q43. How do you reason about property precedence?

Spring Boot aggregates property sources and applies a defined precedence order, where higher-priority sources override lower-priority ones. The exact source list and behavior are versioned; consult the Boot externalized-configuration reference. When debugging, inspect the effective value and its source without exposing secrets in logs.

### Q44. Is @PropertySource the main way to configure a Boot application?

Usually not. Boot's externalized configuration supports application.properties/application.yaml, profile-specific files, environment variables, command-line arguments, and imported configuration sources. @PropertySource remains useful for specific cases, but it is not a replacement for understanding Boot's configuration loading model.

### Q45. How should secrets be supplied to production applications?

Use an approved secret-management system or platform-provided secret mechanism, restrict access, rotate credentials, and avoid committing secrets or logging resolved values. Environment variables can be convenient but are not inherently a complete secrets-management strategy.

### Official references
- [Spring Boot externalized configuration](https://docs.spring.io/spring-boot/reference/features/external-config.html)
- [Configuration properties](https://docs.spring.io/spring-boot/reference/features/external-config.html#features.external-config.typesafe-configuration-properties)

---

## Spring AOP and Proxies


### Q46. What is Aspect-Oriented Programming?

AOP modularizes cross-cutting behavior such as transaction boundaries, authorization checks, metrics, and auditing that would otherwise be repeated across many methods. Spring AOP is primarily proxy-based and intercepts method calls that pass through the proxy; it is not a general bytecode-weaving system by default.

### Q47. What are advice, pointcut, join point, and aspect?

An aspect groups cross-cutting behavior. Advice is the action executed at a selected point. A pointcut selects join points, which in Spring AOP are method-execution join points on Spring-managed objects. Understanding the selection expression is essential because an overly broad pointcut can affect unrelated code.

### Q48. How do JDK dynamic proxies and CGLIB proxies differ?

JDK proxies implement interfaces; CGLIB-based proxies create a subclass of the target class. Proxy selection depends on configuration and framework context. Class-based proxies cannot override final methods, and final classes cannot be subclass-proxied. Design for interception at public service boundaries rather than assuming every method can be intercepted.

### Q49. What is the self-invocation problem?

When a method on a bean calls another method on the same instance with this.method(), the call does not pass through the Spring proxy. Advice such as transactions, caching, or async execution may therefore not run for the internal call. Refactor the intercepted operation to another bean or redesign the boundary; avoid brittle self-proxy workarounds where possible.

### Q50. When should @Around advice be used?

Use around advice when behavior must run before and after the invocation and may need to control proceeding to the target. Always call ProceedingJoinPoint.proceed() appropriately, preserve return values, and avoid swallowing exceptions accidentally. Use narrower advice types when they express the need more clearly.

### Q51. How do you order multiple aspects?

Define ordering explicitly when the relative sequence matters, using supported ordering mechanisms such as @Order. Ordering affects nested interception and exception flow, so test the observable behavior rather than assuming the order from class names or registration order.

### Q52. Is Spring AOP appropriate for every cross-cutting concern?

No. Use AOP for coherent cross-cutting policies at stable method boundaries. Prefer explicit code for business-critical flow that benefits from local readability, and use servlet filters or MVC interceptors when the concern belongs to the HTTP pipeline rather than service method execution.

### Official references
- [Spring AOP reference](https://docs.spring.io/spring-framework/reference/core/aop.html)
- [Proxying mechanisms](https://docs.spring.io/spring-framework/reference/core/aop/proxying.html)

---

## Spring Boot Fundamentals


### Q53. What is a Spring Boot starter?

A starter is a convenient dependency descriptor that brings in a curated set of dependencies for a use case. It reduces dependency-selection effort but does not itself implement the feature or guarantee correct production configuration. The managed versions should normally come from the Boot dependency-management model.

### Q54. What does @SpringBootApplication combine?

It combines the common roles of @SpringBootConfiguration, @EnableAutoConfiguration, and @ComponentScan. The package containing the application class becomes the default component-scan root, so placing it too deep can leave controllers, services, repositories, or configuration classes undiscovered. Keep the main class in a sensible root package, and use explicit scan/import configuration only when the application structure requires it. Auto-configuration is separate from component scanning and depends on classpath and configuration conditions.

### Q55. How does auto-configuration work?

Boot evaluates candidate configuration classes and conditional annotations against the classpath, existing beans, properties, and application environment. Auto-configuration provides sensible defaults and often backs off when the application defines its own bean. It is conditional configuration, not hidden magic that overrides every user choice.

### Q56. What does @ConditionalOnMissingBean mean?

It allows a configuration contribution when no matching bean is present at the point the condition is evaluated. It is commonly used to make auto-configuration replaceable by application configuration. Ordering and bean-definition registration matter, so custom auto-configuration should be tested in realistic contexts.

### Q57. What is the purpose of an executable Boot JAR?

It packages an application with the runtime dependencies and a launcher structure so it can be started with the Java launcher without separately assembling a classpath. It is not the same as a native image, and the executable packaging format does not remove the need for runtime configuration and operational controls.

### Q58. What is the difference between CommandLineRunner and ApplicationRunner?

Both provide callbacks after the application context has started. CommandLineRunner receives raw string arguments; ApplicationRunner receives parsed application arguments through ApplicationArguments. Keep startup work bounded, observable, and idempotent because a slow runner can delay the application from becoming ready.

### Q59. How do you diagnose why an auto-configuration did not apply?

Use Boot's condition evaluation report, enable the relevant debug diagnostics, and inspect the classpath, property values, bean definitions, and active profiles. The report explains why conditions matched or did not match; it should be paired with the actual exception and environment rather than treated as a fix by itself.

### Q60. Does Boot require an embedded web server?

No. Boot supports non-web applications as well as servlet and reactive web applications. The application type and dependencies influence which infrastructure is configured. Avoid adding web starters to background jobs that do not need HTTP endpoints.

### Q61. What is the difference between a starter and auto-configuration?

A starter is a dependency descriptor: it brings together libraries commonly needed for a capability and usually relies on Boot's dependency management for compatible versions. Auto-configuration is runtime configuration logic that conditionally registers beans based on the classpath, properties, application type, and existing bean definitions. A starter can make the classes needed for an auto-configuration available, but it does not guarantee that the conditions match or that the feature is production-ready. When debugging, inspect both the resolved dependency tree and the condition evaluation report.

### Q62. What is the condition evaluation report used for?

It lists auto-configuration conditions and explains why each configuration matched or backed off. Use it to investigate a missing bean, an unexpected default, or behavior that changed after adding a dependency. Correlate the report with the resolved classpath, active profiles, effective properties, and the first meaningful startup exception; do not assume every negative condition is an error.

### Q63. What is the purpose of SpringApplication events?

They expose lifecycle milestones such as environment preparation, context initialization, and readiness. Use the appropriate phase for a listener; an event's existence does not imply every dependency is healthy or every startup task has completed.

### Q64. Why should application startup avoid remote calls in bean constructors?

Constructors run during object creation, so remote calls there can block context initialization, create hidden network dependencies, or fail startup because of a transient outage. Keep constructors focused on establishing valid object state. If startup genuinely depends on a remote service, use an explicit lifecycle component with a bounded timeout, observable failure policy, and documented readiness strategy; do not let an unbounded call stall deployment indefinitely.

### Q65. What is graceful shutdown in Boot?

Graceful shutdown lets supported web servers stop accepting new requests and gives in-flight requests a configured grace period to finish. Configure the shutdown mode and timeout for the selected Boot version, and ensure the deployment platform sends an appropriate termination signal and allows enough termination grace time. This does not guarantee every background task or message handler completes: those components need their own stop, acknowledgement, checkpoint, and retry/recovery policies.

### Q66. How do you package configuration for an executable JAR?

Build one immutable artifact and provide environment-specific settings through supported external configuration sources such as environment variables, mounted configuration, or command-line/property sources. Keep precedence explicit and validate required values at startup. Never package production secrets in the JAR or log the resolved secret-bearing configuration. External configuration should be versioned and deployed with an auditable change process.

### Q67. What is AOT processing in Spring Boot?

Ahead-of-time processing prepares application metadata/code paths for scenarios such as native images and can improve startup/runtime characteristics in supported cases. It introduces constraints around reflection/dynamic behavior and must be tested with the chosen dependencies and build configuration.

### Q68. Does a successful Boot startup guarantee production correctness?

No. Startup proves only that the initialization path completed under the conditions present at that moment. It does not prove authorization rules, downstream timeouts, database pool capacity, message redelivery, schema compatibility, graceful shutdown, or behavior under load. Validate those with integration and failure-path tests, production-like load, and runtime signals such as readiness, latency, error rate, and saturation.

### Official references
- [Spring Boot reference](https://docs.spring.io/spring-boot/reference/)
- [Auto-configuration](https://docs.spring.io/spring-boot/reference/using/auto-configuration.html)
- [Creating an executable JAR](https://docs.spring.io/spring-boot/maven-plugin/packaging.html)

---

## Auto-Configuration and Application Startup


### Q69. How does Spring Boot auto-configuration decide what to configure?

Boot imports candidate auto-configuration classes and evaluates conditions against classpath contents, bean definitions, properties, and application type. Conditions can enable defaults only when their prerequisites hold. Inspect the condition evaluation report rather than guessing which starter or property caused a configuration to activate.

### Q70. Why does auto-configuration often back off when a user bean exists?

Many auto-configurations use conditions such as `@ConditionalOnMissingBean` so an application can replace a default. The condition is evaluated during configuration processing, so registration order and the type/name of the user bean can matter. This is a customization mechanism, not a guarantee that any similarly named bean replaces all related infrastructure.

### Q71. How do you exclude an auto-configuration?

Use the supported exclusion mechanisms, such as the exclude attribute on `@SpringBootApplication` or the corresponding configuration property. First verify why the configuration matched; excluding it may remove supporting beans that other components rely on. Prefer correcting the classpath or property condition when that is the actual cause.

### Q72. How do auto-configuration registration mechanisms differ across Boot generations?

The registration metadata evolved. Older Boot generations commonly used `spring.factories` for auto-configuration candidates; modern Boot uses the dedicated `AutoConfiguration.imports` metadata for auto-configuration discovery. When building a starter, follow the documentation for the exact Boot baseline instead of copying old registration examples.

### Q73. What can go wrong if a BeanFactoryPostProcessor triggers bean creation too early?

A BeanFactoryPostProcessor runs before normal singleton instantiation, so eagerly requesting application beans during that phase can create them before all BeanPostProcessors are registered. Those beans may miss proxying or other instance-level processing and can produce lifecycle warnings. Keep factory post-processors focused on bean definitions, use static @Bean methods for infrastructure where required, and inspect startup logs when a bean unexpectedly lacks advice.

### Q74. What does ApplicationContext refresh mean?

Refresh initializes the context's infrastructure, applies factory post-processors, registers bean post-processors, instantiates eligible non-lazy singletons, and completes lifecycle coordination. Exact details depend on context type and version. A failure during refresh commonly prevents the application from reaching its ready state.

### Q75. How do you investigate slow startup?

Measure startup phases before optimizing. Inspect startup logs and Boot startup metrics or the available startup endpoint/tooling, identify expensive bean constructors and initialization callbacks, and check classpath scanning, remote calls, migrations, and connection initialization. Lazy initialization can shift work rather than remove it, so validate first-request latency and readiness afterward.

### Q76. What is a custom auto-configuration module?

It is a library module that conditionally contributes beans when relevant classes or properties are present and backs off when the application supplies replacements. Keep conditions precise, avoid scanning an application's packages from a library, document properties, and test positive, negative, and override cases with Boot's auto-configuration test support.

### Official references
- [Boot auto-configuration](https://docs.spring.io/spring-boot/reference/using/auto-configuration.html)
- [Boot startup and application lifecycle](https://docs.spring.io/spring-boot/reference/features/spring-application.html)

---

## Spring Boot Configuration and Profiles


### Q77. How does application.properties differ from application.yaml?

Both formats feed Boot's externalized-configuration system; the main difference is syntax. Properties files express nested configuration with dotted keys, while YAML represents hierarchy through indentation and is often easier to scan for grouped settings. Neither format is inherently more powerful, and YAML indentation mistakes can change the structure. Avoid defining the same key in several files unless the override is intentional, because property-source precedence determines the effective value. Choose one style consistently, then verify the resolved value and source for each active profile in the deployed environment.

### Q78. How are profile-specific files selected?

Spring Boot can load profile-specific configuration such as `application-dev.yaml` when the corresponding profile is active, and profile activation can come from supported configuration sources such as environment variables or command-line arguments. Profiles influence property resolution and the registration of beans guarded by `@Profile`; they are not simply file names. Confirm the effective active-profile list and property-source precedence when an environment behaves unexpectedly. Keep production secrets in an appropriate secret-management mechanism rather than committing them to profile files.

### Q79. What are configuration imports used for?

Boot configuration imports allow additional configuration sources to be loaded through its external configuration model. Optionality, location semantics, and integration-specific behavior depend on the source type and Boot version. Use the official reference for the exact syntax rather than assuming arbitrary remote URLs are automatically supported.

### Q80. Why use immutable or validated configuration properties?

Immutable or validated configuration properties turn deployment assumptions into startup checks. Group related values in a dedicated type, bind them with @ConfigurationProperties, and validate required URLs, durations, pool sizes, and limits before the service accepts traffic. This catches a missing endpoint or impossible timeout during startup rather than during a customer request. Prefer constructor-bound or otherwise immutable properties where supported by the target version. Keep secrets out of generated toString output and diagnostics, and test both valid configuration and expected startup failure for invalid values.

### Q81. How should Kubernetes configuration be organized?

Keep environment-specific values external to the image, inject non-secret settings through the platform's configuration mechanism, and use a managed secret mechanism for credentials. Define resource requests/limits, probes, and shutdown behavior separately from application properties. Configuration changes do not automatically refresh arbitrary singleton beans.

### Q82. Does changing an environment variable update a running Spring bean?

Usually not. Environment variables are read by the process environment, and property binding generally occurs during startup. Dynamic refresh requires an explicitly supported refresh mechanism and compatible consumers; a process restart or rollout is often the safer operational model.

### Q83. What is the danger of logging the Environment or all configuration properties?

The effective Environment can contain passwords, tokens, connection strings, internal hostnames, or personal data contributed by several property sources. Dumping it wholesale can leak secrets into logs and log aggregation systems with broad retention/access. Restrict configuration-inspection endpoints, use secret-aware redaction, and log only explicitly allow-listed non-sensitive keys needed for diagnosis. When debugging precedence, report the winning source/key where safe rather than printing the full value.

### Official references
- [Externalized configuration](https://docs.spring.io/spring-boot/reference/features/external-config.html)
- [Profiles](https://docs.spring.io/spring-boot/reference/features/profiles.html)

---

## Spring MVC and REST APIs


### Q84. What role does DispatcherServlet play?

DispatcherServlet is Spring MVC's front controller: it receives the servlet request and coordinates handler mapping, handler execution, argument resolution, message conversion, return-value handling, and exception resolution. It does not implement business logic itself. A request can be rejected before reaching a controller by a servlet filter or Spring Security; after the servlet begins processing, failures may arise from handler mapping, argument binding, validation, controller logic, or response serialization. When debugging, trace the request through the filter chain and inspect which handler was selected rather than assuming every HTTP error originates in the controller.

### Q85. What is the difference between @Controller and @RestController?

@Controller marks a Spring MVC controller, where a return value may be interpreted as a view name. @RestController combines controller semantics with response-body handling, so returned objects are generally serialized into the HTTP response. Individual methods can still customize status and headers using ResponseEntity.

### Q86. What is the difference between @PathVariable and @RequestParam?

@PathVariable binds a value from a URI path segment, commonly identifying a resource such as /orders/{id}. @RequestParam binds a query parameter or form parameter, commonly used for filtering, pagination, or optional modifiers. Choose based on resource semantics and the API contract, not personal preference alone.

### Q87. Why use DTOs rather than returning persistence entities directly?

DTOs define the API contract independently of database mapping, reduce accidental exposure of internal fields, and allow validation and serialization shape to evolve deliberately. Returning entities can also trigger lazy-loading queries or recursion through bidirectional relationships. Map at a clear application boundary and avoid leaking persistence details.

### Q88. How does request-body validation work?

Use Jakarta Bean Validation constraints on request DTOs and trigger validation with @Valid or the supported validation mechanism at the controller boundary. Handle validation exceptions consistently and return actionable field errors without echoing sensitive input. Validation is not a replacement for business-rule validation or database constraints.

### Q89. How should pagination be designed?

Accept bounded page/size or cursor parameters, validate limits, and return predictable metadata. Large offset pagination can become expensive for deep pages; cursor/keyset approaches may be preferable for large, frequently changing datasets. Ensure sorting is deterministic, ideally with a unique tie-breaker.

### Q90. What is content negotiation?

Content negotiation selects a representation based on request headers and supported media types. Message converters handle serialization and deserialization for supported types. Explicitly define supported content types and test malformed payloads, unsupported media types, and incompatible Accept headers.

### Q91. What makes an API operation idempotent?

An operation is idempotent if repeating the same request has the same intended effect as performing it once. HTTP method semantics help communicate this, but implementation must honor them. For retryable create operations, an idempotency key and durable deduplication record may be needed to avoid duplicate side effects.

### Q92. How should file uploads be protected?

Set request-size limits at the application and edge layers, validate content and authorization, avoid trusting client-supplied filenames or MIME types, and stream large files when appropriate. Store files outside executable application paths and apply malware scanning or content inspection when required by the threat model.

### Q93. What is the difference between a servlet filter and HandlerInterceptor?

A filter participates in the servlet request/response chain and can run before DispatcherServlet. A HandlerInterceptor participates in Spring MVC's handler lifecycle after handler mapping. Use filters for servlet-wide concerns such as request wrapping and security integration, and interceptors for handler-oriented concerns that need MVC metadata.

### Q94. How does Spring select an HTTP message converter?

Spring MVC considers the target method parameter/return type, request Content-Type, response Accept negotiation, and converters registered in the application. Unsupported media types can produce 415 for request content or 406 for an unacceptable response representation. Customize converters only when the API needs behavior beyond standard support.

### Q95. What is the difference between 400 and 422?

Both can represent client input problems depending on the API contract. 400 is broadly used for malformed or invalid requests; 422 can express syntactically valid content whose instructions cannot be processed semantically. Choose a consistent contract rather than assuming the framework will choose your domain's preferred status automatically.

### Q96. How should API sorting parameters be secured?

Map public sort keys to a fixed set of entity properties or query expressions, validate direction, cap page size, and pass the resulting `Sort`/`Pageable` through supported repository APIs. Never concatenate client input into SQL or JPQL identifiers. Add a unique tie-breaker (for example, ID) after the requested sort so equal values do not cause unstable page boundaries.

### Q97. What is cursor pagination's main trade-off?

It can avoid expensive deep offsets and behave better under some concurrent insert patterns, but it complicates arbitrary page jumps and requires stable cursor fields/order. Define whether cursors are opaque, signed, and bound to the original filter/sort criteria.

### Q98. How should request validation differ from domain validation?

Request validation checks whether the incoming representation is structurally acceptable—for example, a required field is present and a quantity is positive. Domain validation protects business invariants regardless of entry point: HTTP, a Kafka listener, a scheduled job, or a direct service call must not be able to create an invalid state. Use Bean Validation annotations at the API boundary for simple constraints, but keep rules such as “an order can only be cancelled before shipment” in the application/domain layer where they can inspect current state. Avoid duplicating complex rules in controllers and never rely on request validation alone to protect data integrity.

### Q99. How should an API handle unsupported media types and oversized payloads?

Return a clear 415 response when the request media type is unsupported and a documented 413 response when a configured payload limit is exceeded, while recognizing that proxies or servlet containers may reject the request before controller code runs. Set limits at ingress and application layers, keep error bodies stable, and test which layer owns each response. Do not rely on JSON validation alone to constrain payload size.

### Q100. When should an API use streaming responses?

Use streaming for large exports, generated reports, or incremental feeds when avoiding whole-response buffering materially reduces memory or improves time-to-first-byte. Define framing/content type, cancellation, timeout, backpressure, and what happens if an error occurs after headers or partial content have been sent. Clients and intermediaries must support the chosen protocol; for small bounded responses, ordinary serialization is usually simpler.

### Q101. How should large file uploads be streamed and stored efficiently?

Avoid buffering the entire upload in heap memory. Stream to controlled temporary or object storage, enforce byte limits while streaming, apply backpressure/timeouts, and clean up incomplete uploads after cancellation or failure. Promote a file to its final state only after validation succeeds, and make retries or duplicate submissions safe. Keep the security checks from the earlier upload-design question: streaming is a resource strategy, not a substitute for authorization or content validation.

### Official references
- [Spring MVC reference](https://docs.spring.io/spring-framework/reference/web/webmvc.html)
- [Validation](https://docs.spring.io/spring-framework/reference/core/validation/beanvalidation.html)

---

## Exception Handling and API Error Design


### Q102. What is the role of @ControllerAdvice?

@ControllerAdvice centralizes MVC concerns across controllers, most commonly mapping exceptions to consistent HTTP responses and handling selected binding/validation behavior. @RestControllerAdvice adds response-body semantics, so returned objects are serialized rather than treated as view names. Keep handlers specific enough to preserve the distinction between invalid input, missing resources, conflicts, and unexpected server failures. Do not return stack traces or database messages to clients; log safe diagnostics with a correlation ID at an appropriate boundary. For APIs using Problem Details, keep the public error contract stable even if the underlying framework exception changes.

```java
@RestControllerAdvice
class ApiExceptionHandler {
    @ExceptionHandler(OrderNotFoundException.class)
    ProblemDetail handleNotFound(OrderNotFoundException ex) {
        return ProblemDetail.forStatusAndDetail(
            HttpStatus.NOT_FOUND, "Order not found");
    }
}
```

This is a focused fragment; imports and the application's exception type are omitted.

### Q103. When should an exception become HTTP 400, 404, 409, or 500?

Use 400 for invalid request syntax or input according to the API contract, 404 when the requested resource is not found, 409 for a conflict with current resource state, and 500 for unexpected server failures. The correct status depends on the semantics and must remain consistent across endpoints.

### Q104. What is ProblemDetail?

In supported Spring Framework versions, `ProblemDetail` represents an RFC 7807-style problem response. It can provide a standard type/title/status/detail/instance shape with extensions. Verify version support and serialization behavior, and do not expose stack traces or internal exception messages to clients.

### Q105. Should controllers catch every exception?

No. Catch exceptions when the layer can meaningfully recover, translate, or add context. A global boundary can map known failures consistently, while unexpected failures should be logged with correlation context and returned as safe generic errors. Swallowing exceptions or returning HTTP 200 for failed operations breaks client behavior and observability.

### Q106. How should validation errors be represented?

Return a stable structure with a general error code and field-level messages that are safe to disclose. Avoid returning rejected secrets or full request bodies. Keep validation messages suitable for API clients and do not make clients parse framework exception class names.

### Q107. How do downstream service failures affect API error mapping?

Distinguish invalid client input from temporary dependency failures, timeouts, and unavailable downstream systems. Map them according to the API contract, preserve retryability information where appropriate, and avoid exposing internal hostnames. Pair error responses with bounded timeouts, metrics, and trace correlation.

### Q108. How do you preserve diagnostic context without leaking sensitive details?

Log the exception and relevant safe identifiers at the boundary where it can be understood; include a trace or correlation ID in logs and responses. Redact tokens and personal data, avoid logging the same exception at every layer, and keep client-facing details separate from internal diagnostics.

### Official references
- [Spring MVC exception handling](https://docs.spring.io/spring-framework/reference/web/webmvc/mvc-controller/ann-exceptionhandler.html)
- [Problem Details](https://docs.spring.io/spring-framework/reference/web/webmvc/mvc-ann-rest-exceptions.html)

---

## Spring Data Access and Transactions


### Q109. What does @Transactional do?

`@Transactional` declares the transaction boundary and attributes—such as propagation, isolation, timeout, read-only intent, and rollback rules—for a method or class. In the common proxy-based model, a call entering through the Spring proxy starts, joins, or otherwise participates in a transaction; the transaction manager then coordinates the configured resource, such as JDBC or JPA. The method's database work normally commits when it completes successfully and rolls back when the configured rollback rules match. It does **not** make an HTTP call, a Kafka publish, and a database write one atomic operation. Put the boundary around a coherent business use case, commonly at the service layer, rather than adding the annotation indiscriminately to every repository and controller.

### Q110. What is transaction propagation?

Propagation answers: **what should this method do if the caller already has a transaction?** `REQUIRED` is the usual default: join the current transaction or create one. `REQUIRES_NEW` suspends the current transaction and starts an independent one, so the inner work may commit even if the outer transaction later rolls back; it may also need another database connection. `SUPPORTS` runs with a transaction if one exists, while `MANDATORY` requires one and `NEVER` rejects one. `NOT_SUPPORTED` suspends an existing transaction. `NESTED`, where supported, usually uses a savepoint in the same physical transaction. Choose based on required commit/rollback semantics—not as a generic way to “make a method transactional.”

### Q111. What is transaction isolation?

Isolation defines how concurrent transactions observe each other's changes. The familiar levels—READ_UNCOMMITTED, READ_COMMITTED, REPEATABLE_READ, and SERIALIZABLE—describe progressively stronger guarantees, but their exact behavior depends on the database, MVCC implementation, and locking strategy. For example, READ_COMMITTED typically prevents dirty reads but can allow the same row to return different committed values on two reads. Stronger isolation can reduce anomalies while increasing blocking, retries, or contention. Spring's `Isolation.DEFAULT` delegates to the database default; do not assume the same behavior across PostgreSQL, Oracle, MySQL, and an in-memory test database. Set a level only when the business invariant requires it, then test concurrent operations against the actual database engine.

### Q112. Does Spring roll back for every exception?

By default, Spring's declarative transaction rules roll back for `RuntimeException` and `Error`, but not for checked exceptions. This surprises developers who catch or declare a checked business exception and assume the database work will be undone. Specify `rollbackFor` when a checked exception must trigger rollback, or `noRollbackFor` when a matching exception should not. Spring Framework 6.2 also supports changing the global default to roll back on all exceptions; confirm the exact version and configuration before relying on it. A rollback rule is evaluated against an exception that reaches the transaction interceptor—catching an exception inside the method and returning normally usually prevents the default rule from seeing it, unless the transaction was already marked rollback-only.

```java
@Transactional(rollbackFor = PaymentRejectedException.class)
public void placeOrder(Order order) throws PaymentRejectedException {
    orderRepository.save(order);
    paymentService.authorize(order); // checked failure should roll back
}
```

Keep exception policy aligned with business semantics, and test rollback behavior rather than only asserting that an exception was thrown.

### Q113. Why might @Transactional appear not to work?

Start by verifying that transaction management is configured and that the object is a Spring-managed bean. In default proxy mode, the call must enter through the proxy; `this.update()` inside the same class bypasses the transaction interceptor. Also check method visibility and proxy type, whether the annotation is on the concrete implementation, which transaction manager is selected, and whether the code actually accesses a transactional resource. Finally, inspect rollback rules: checked exceptions do not trigger rollback by default, and catching an exception may let the method return normally. Turn on transaction-related debug logging and write a test that verifies the database state after success and failure. Refactoring the transactional operation into a separate injected bean is usually clearer than relying on self-proxy tricks.

### Q114. What is the difference between JdbcTemplate and NamedParameterJdbcTemplate?

JdbcTemplate provides JDBC operations with resource handling and exception translation, using positional placeholders. NamedParameterJdbcTemplate supports named parameters that are expanded into JDBC placeholders. Both reduce repetitive JDBC plumbing but do not eliminate the need for correct SQL, transaction boundaries, or query performance analysis.

### Q115. What does `readOnly = true` guarantee?

It declares intent and can enable optimizations—for example, a JPA provider may avoid some dirty-checking work, and a transaction manager or database driver may apply read-only hints. It is not a portable promise that writes will be rejected, and it is not an authorization boundary. The effect depends on the transaction manager, provider, driver, and database. Use it for genuinely read-only service operations where appropriate, but do not annotate a method read-only if it must update entities, and verify the actual behavior on the production stack.

### Q116. Why is a database transaction not enough for reliable Kafka publishing?

A local database transaction normally protects the database resource, not an independent Kafka broker operation. If the database commits and publishing fails, downstream systems never learn about the change; if the message is published first and the database rolls back, consumers may observe an event for state that does not exist. Spring's transaction synchronization or Kafka transaction support can coordinate specific workflows, but do not assume that simply putting `@Transactional` on a method makes a database and broker one atomic resource. For robust database-to-event delivery, the transactional outbox pattern stores the business change and an outbox record in one database transaction, then publishes the record asynchronously. Consumers should still be idempotent because delivery and retries can produce duplicates.

### Q117. Which coding patterns hold JDBC connections longer than intended?

Long transactions that include remote HTTP calls, slow processing, user interaction, or large result iteration can retain a connection while doing work that does not require the database. Keep transaction boundaries around the required database operation, avoid remote I/O inside them, and use bounded streaming/chunking where appropriate. Confirm the actual connection-acquisition/release lifecycle for the chosen transaction manager and data-access API.

### Q118. What resource-pool risk does `REQUIRES_NEW` introduce?

The outer transaction is suspended, but its connection may remain bound and unavailable for reuse while the inner transaction acquires another connection. If many request threads all hold an outer connection and wait for a second connection, the pool can become saturated; increasing the pool blindly can then overload the database. This risk is especially important when a loop or layered service invokes several `REQUIRES_NEW` operations.

Use it only when the inner operation truly must commit or roll back independently. Keep both scopes short, verify that the call crosses a Spring proxy, and load-test the maximum nesting/concurrency pattern while watching active connections, pending borrowers, acquisition latency, and transaction duration.

### Q119. When is `NESTED` propagation useful?

`NESTED` is useful when a sub-operation should be able to roll back to a savepoint without necessarily rolling back all work performed earlier in the outer transaction. It normally remains part of the same physical transaction; a successful outer commit commits the work that remains. Support depends on the transaction manager and resource, and JPA-specific behavior should not be assumed to match JDBC savepoint behavior. Unlike `REQUIRES_NEW`, it does not generally create an independently committing transaction. Use it only when partial rollback is a real business requirement and test the exact database/transaction-manager combination.

### Q120. Why can a caught exception still lead to rollback failure at commit?

An inner transactional method can mark the shared transaction rollback-only when it encounters a failure. If the outer method catches that exception and continues, its own code may appear successful, but the transaction cannot be committed. At the end of the outer boundary, Spring may throw `UnexpectedRollbackException` rather than silently pretending that the work committed. Investigate the earliest exception and the propagation boundary, not just the final exception. Decide whether the inner failure should abort the whole use case, be isolated with `REQUIRES_NEW`, or be handled as a normal business outcome; each choice changes consistency semantics.

### Q121. What is the difference between a local transaction and a distributed workflow?

A local transaction coordinates supported resources through one transaction manager. A workflow spanning HTTP services, databases, and a broker needs explicit reliability semantics such as outbox, idempotency, saga compensation, or a supported distributed transaction protocol where appropriate.

### Q122. How can transaction boundaries affect connection pool usage?

Transactions often bind a database connection until the transaction completes. A service that reads a row, makes a slow HTTP call, and only then updates the row can hold a scarce connection during unrelated network waiting. Under load, the pool may run out even when SQL execution itself is fast. Keep the transaction around the database consistency boundary, move remote calls outside it where the business sequence allows, and measure connection-acquisition wait, transaction duration, and query/lock time separately. Do not split a transaction without checking whether doing so changes atomicity or exposes partially completed business state.

### Q123. What is transaction-aware caching?

It coordinates cache operations with transaction completion so changes are not made visible at an inappropriate phase. Exact behavior depends on the cache abstraction/provider configuration. Verify rollback and after-commit behavior rather than assuming cache eviction is automatically atomic with the database.

### Q124. How should a service handle an optimistic-lock conflict?

Treat an optimistic-lock failure as evidence that another transaction changed the row after it was read, not as a generic transient database outage. Reload the current state and either re-evaluate the business command, return a conflict response, or ask the caller to reconcile. Retry only if the operation is safe and its rules can be recomputed against fresh data; do not blindly replay external side effects or overwrite the competing update.

### Q125. Why can an in-memory database hide transaction defects?

An in-memory database may differ from production in SQL dialect, implicit casts, constraint timing, lock acquisition, isolation semantics, sequence behavior, and query planning. A test can pass while a production query fails or concurrent updates violate assumptions. Use fast in-memory tests for focused logic, but run migrations and critical transaction/concurrency tests against the production database engine and a compatible version.

### Q126. How do you investigate a deadlock?

Collect database deadlock diagnostics and transaction/query logs, map lock acquisition order, and identify cycles. Reduce transaction scope, access rows in consistent order where possible, and implement safe retry for database-detected transient deadlocks only when the business operation is repeatable.

### Official references
- [Spring transaction management](https://docs.spring.io/spring-framework/reference/data-access/transaction.html)
- [JDBC abstraction](https://docs.spring.io/spring-framework/reference/data-access/jdbc.html)

---

## Spring Data JPA Integration


### Q127. What does Spring Data JPA add on top of JPA?

JPA defines the persistence API and entity model; a provider such as Hibernate implements it. Spring Data JPA builds repository infrastructure on top: it creates repository proxies, derives queries from method names, supports declared queries, paging and sorting, projections, specifications, auditing, and custom repository fragments. It reduces repetitive data-access code but does not remove JPA concepts such as managed versus detached entities, dirty checking, flush timing, lazy loading, or the persistence context. A senior answer should distinguish the repository abstraction from the ORM behavior underneath it, and use SQL logs and database execution plans when performance is the problem.

### Q128. How do repository interfaces work without a handwritten implementation?

Spring Data creates repository proxy implementations based on repository metadata and configured infrastructure. Method names, annotations, and fragments describe query behavior. When a derived query becomes hard to understand, prefer an explicit `@Query` or a custom repository fragment rather than an opaque method name.

### Q129. What is the purpose of Page, Slice, and List return types?

A Page commonly includes total-count metadata and may require an additional count query. A Slice indicates whether more data exists without necessarily calculating the full total. A List returns results without pagination metadata. Choose based on product needs and query cost.

### Q130. What is a projection?

A projection retrieves a subset or alternate view of data instead of exposing the full entity model. Interface and DTO projections can reduce unnecessary data transfer, but actual SQL and optimization depend on query form and provider behavior. Inspect generated SQL and query plans for critical paths.

### Q131. How should dynamic filtering be implemented?

Use a supported approach such as Specifications, query-by-example where suitable, or a custom repository query. Validate allowed filter and sort fields, combine predicates deliberately, and avoid string-concatenating untrusted SQL. Query design should remain understandable and testable.

### Q132. How do N+1 queries arise, and how would you fix them?

The application loads a list of parent entities with one query and then triggers a separate query for a lazy relationship for each parent—for example, one query for 100 orders followed by 100 customer lookups. The code can look harmless while latency and database load grow with result size. Confirm the pattern with SQL logging, tracing, query-count tests, or ORM statistics. Choose a targeted fix: a DTO projection for a read use case, a fetch join for a suitable query, or an `@EntityGraph` when it expresses the required graph clearly. Avoid making every relationship eager: multiple collections can cause row multiplication, duplicate results, pagination problems, and excessive memory use. Validate the generated SQL and execution plan.

### Q133. How do optimistic and pessimistic locking differ?

Optimistic locking assumes concurrent updates are relatively uncommon and detects conflicts, commonly through a @Version field; the losing update fails rather than silently overwriting the newer state. Pessimistic locking asks the database to lock selected rows and can be useful under predictable contention, but increases waiting and deadlock risk. Prefer optimistic locking for many read-heavy workflows and use pessimistic locking only when its blocking semantics are justified. On conflict, reload current state and re-evaluate the business command; blindly retrying a stale entity can repeat invalid decisions or side effects.

```java
@Version
private long version;
```

The provider updates and checks this field as part of entity persistence; do not increment it manually.

### Q134. What repository abstractions should you use?

The available interfaces vary by Spring Data version. Common options include `CrudRepository`, `ListCrudRepository`, `PagingAndSortingRepository`, and `JpaRepository`; do not assume inheritance relationships from one version apply unchanged to another. Select the smallest useful contract and consult the reference for the chosen release.

### Q135. Why can repository pagination return inconsistent results during concurrent writes?

Offset pages are positional, so inserts/deletes between requests can shift records across page boundaries. A deterministic sort reduces ambiguity but does not create a consistent snapshot across independent requests. For large changing datasets, consider cursor/keyset pagination and document its consistency model.

### Q136. How do Specifications help with dynamic queries?

Specifications compose predicates in a type-aware way for supported Spring Data JPA setups. They are useful when filters are optional, but complex joins and fetches can become difficult to reason about. Keep sorting, filtering, and fetch strategy deliberate and inspect generated SQL.

### Q137. Why can @Modifying queries leave managed entities stale?

Bulk JPQL updates and native modifying queries operate directly against database rows rather than updating each already-managed entity through normal dirty checking. As a result, an entity loaded earlier in the same persistence context may still expose stale field values even though the database has changed. Depending on the use case, flush pending changes before the bulk operation and clear or refresh affected state afterward; Spring Data JPA exposes options such as `flushAutomatically` and `clearAutomatically` for relevant modifying queries. These options have side effects, so test the whole transaction and avoid clearing state that still contains unflushed work.

### Q138. What is the risk of using native queries everywhere?

Native SQL is useful for database-specific features, vendor functions, complex reporting, or a measured performance need, but it couples the query to the schema and often to a database dialect. It can bypass assumptions made by entity mappings or complicate pagination/count queries. Keep native queries focused, document why JPQL or derived queries are insufficient, bind parameters safely, and test against the production database engine and representative data.

### Q139. How should auditing timestamps be handled?

Choose a consistent time model: UTC instants are usually appropriate for event timestamps, while business concepts such as a store's opening time may require a local date/time and explicit zone. Configure Spring Data auditing and its date-time provider deliberately, and avoid relying on the host's default time zone. Test persistence, JSON serialization, daylight-saving transitions where relevant, and user-facing display separately; formatting a timestamp is not the same as defining its meaning.

### Q140. How do you diagnose a query that is fast locally but slow in production?

Compare data volume and distribution, index definitions, current database statistics, execution plans, parameter selectivity, lock waits, connection acquisition time, and network latency. Local fixtures often omit skew and scale, so an index that helps a tiny table may not help the production workload. Capture slow SQL and timings first, inspect the plan using representative parameters and safe production-like data, and validate the change with measured latency and resource use.

### Official references
- [Spring Data JPA reference](https://docs.spring.io/spring-data/jpa/reference/)
- [Query methods](https://docs.spring.io/spring-data/jpa/reference/jpa/query-methods.html)

---

## Spring Security


### Q141. What is the difference between authentication and authorization?

Authentication answers **who is making the request?** and establishes an `Authentication` representing the principal and its credentials/authorities. Authorization answers **is this principal allowed to perform this action on this resource now?** A successfully validated JWT proves only what the trusted issuer and validated claims establish; it does not mean the caller may read every customer, order, or tenant. Enforce authorization at the correct boundary, including object-level and tenant-level checks, and test forbidden cases. A common security defect is to authenticate a user correctly but trust a client-supplied resource ID without verifying ownership or permission.

### Q142. What is `SecurityFilterChain`?

In servlet applications, Spring Security uses `FilterChainProxy` to select a `SecurityFilterChain` for each request. Each chain defines its own filters and security configuration. If several chains match, **only the first matching chain is used**, so chain order and request matchers are security-critical: a broad `/api/**` or `any request` matcher placed too early can shadow a more specific policy. Modern applications normally declare `SecurityFilterChain` beans instead of extending the removed `WebSecurityConfigurerAdapter` approach. When debugging, identify the selected chain first, then inspect authentication, authorization rules, custom filters, and the final entry-point/denied-handler behavior.

```java
@Bean
SecurityFilterChain apiSecurity(HttpSecurity http) throws Exception {
    http.securityMatcher("/api/**")
        .authorizeHttpRequests(auth -> auth
            .requestMatchers("/api/health").permitAll()
            .anyRequest().authenticated());
    return http.build();
}
```

This is an illustrative fragment, not a complete security policy; authentication mechanism, CSRF/CORS policy, and error handling must be configured for the application.

### Q143. What is the difference between 401 and 403?

401 indicates that the request lacks acceptable authentication credentials or authentication failed. 403 indicates the request was understood but access is denied for the current principal, although exact responses can be customized. A configured entry point and access-denied handler influence the response behavior.

### Q144. How should a REST API validate JWTs?

Prefer Spring Security OAuth 2.0 Resource Server support instead of writing a custom parser. Configure the trusted issuer and signing-key/JWK source so the framework can validate the signature and standard claims such as expiration and not-before; validate audience when the API's trust model requires it. Also verify algorithm and key-rotation expectations, authority mapping, and failure behavior. Decoding the Base64URL payload is not validation: anyone can construct a token-shaped string, and a correctly signed token from the wrong issuer or intended for a different audience must still be rejected. Keep secrets and raw tokens out of logs, and test expired, malformed, wrong-issuer, wrong-audience, and insufficient-authority cases.

### Q145. What is SecurityContextHolder?

It provides access to the current SecurityContext, which holds authentication information. The default servlet strategy is typically thread-bound; asynchronous execution and reactive pipelines have different context-propagation considerations. Do not assume security context automatically follows arbitrary executor tasks.

### Q146. What is the difference between roles and authorities?

Authorities are granted permission strings. Role-based checks are a convention that often adds a ROLE_ prefix when using role-oriented APIs. Be consistent about prefixes and use authorities that express the actual permission needed rather than overloading broad roles for every decision.

### Q147. Why does CSRF protection matter, and when should it remain enabled?

Cross-site request forgery abuses the browser's automatic attachment of credentials—most commonly session cookies—to make an authenticated user submit an unwanted request. CSRF protection should normally remain enabled for browser applications using cookie/session authentication. A service that accepts bearer tokens explicitly supplied in an `Authorization` header and does not rely on ambient browser credentials may have a different CSRF risk, but that conclusion depends on the full authentication and browser design. Calling an endpoint “REST” or “stateless” is not by itself sufficient justification to disable CSRF. Document the threat model and test the intended browser and non-browser clients.

### Q148. What are CORS and CSRF, and why are they not interchangeable?

CORS is a browser-enforced cross-origin access policy; CSRF is an attack in which a victim's browser is induced to submit an authenticated request. CORS is not a general server-side authorization mechanism and does not replace CSRF defenses.

### Q149. How should method-level security be enabled and used?

Enable method security with `@EnableMethodSecurity` in a configuration class, then place expressions such as `@PreAuthorize` on application/service methods where they protect the business operation rather than only one transport entry point. For example, `@PreAuthorize("hasAuthority('order:read')")` expresses a permission check, while a tenant/resource ownership check may need a domain-aware authorization service. Common proxy limitations apply: a direct self-invocation may bypass interception, and the method must be eligible for the configured proxy mechanism. Keep expressions readable, avoid placing sensitive authorization logic only in controllers, and write tests for permitted, denied, anonymous, and cross-tenant cases.

### Q150. How should multiple `SecurityFilterChain` beans be ordered?

Give each chain a clear, non-overlapping purpose where possible—for example, a high-priority chain for `/api/**` and a fallback chain for browser routes. Spring Security evaluates chains in order and uses the first one whose matcher succeeds; it does not merge all matching chains. Put narrow matchers before broad catch-all matchers and make the fallback policy explicit. Test a request that should match each chain, plus paths that nearly overlap, and inspect debug logs to confirm which chain actually ran. A route that matches no intended chain can be exposed or governed by an unexpected default, so treat chain coverage as a security test.

### Q151. What is the difference between UserDetailsService and AuthenticationProvider?

UserDetailsService loads user-related data for authentication flows that use it, often by username; it does not itself validate a password or complete every authentication mechanism. AuthenticationProvider implements a specific authentication strategy: it checks the supplied Authentication object using the relevant credential/identity logic and returns an authenticated result or rejects the attempt. A common username/password flow asks UserDetailsService for the stored user and uses a PasswordEncoder to compare the submitted password. Custom providers are useful for genuinely different authentication mechanisms, but avoid reimplementing standard OAuth2/JWT validation when Spring Security already provides it.

### Q152. How should passwords be stored?

Store password hashes using a supported adaptive password encoder, such as the framework's delegating encoder configuration, and never store plaintext or reversible encrypted passwords as a substitute. Choose parameters appropriate to the deployment and plan migration when algorithms or parameters change.

### Q153. What is the purpose of a PasswordEncoder?

It defines how passwords are encoded and how a submitted password is checked against a stored encoded value. Use an adaptive password-hashing algorithm with cost parameters appropriate to the deployment; never decrypt a password or store plaintext. A delegating encoder can identify the algorithm for each stored hash and support gradual upgrades when the preferred algorithm or work factor changes.

### Q154. What is method security's proxy limitation?

Method security commonly relies on Spring AOP interception. If one method calls another method on the same object through this, the call bypasses the proxy, so the annotation on the inner method may not be evaluated. Proxy type and method visibility also affect interception. Put security checks at a clear application-service boundary that external callers actually cross, or refactor the protected operation into a separate injected bean. Do not depend on a controller-only check if the same operation is reachable through messaging, jobs, or another internal path. Test both permitted and denied invocations through the real wiring.

### Q155. Should authorization exist only at the controller layer?

Usually not. Controller authorization is useful for protecting HTTP routes, but the same use case may later be invoked by a scheduled task, message listener, batch job, or another application component. Put the invariant at a service/domain boundary that all relevant callers pass through, and use controller rules as an additional edge policy where appropriate. Separate authentication identity, permission checks, and business validation. Also remember that system-initiated work may have no end-user principal, so define its service identity and allowed actions deliberately rather than bypassing checks implicitly. Test the operation through each entry point that matters.

### Q156. How should multi-tenant authorization be enforced?

Treat tenant isolation as a data-access and authorization invariant, not just a URL convention. Resolve the tenant from a trusted claim or server-side identity mapping, then verify that each requested resource belongs to that tenant before reading or mutating it. Apply the same context to repository queries, cache keys, asynchronous tasks, scheduled jobs, exports, and message consumers; background processing must carry an explicit trusted tenant context. A client-supplied `tenantId` may be a filter but must not be treated as proof of access. Test attempts to substitute another tenant's object ID, omit tenant criteria, reuse cached results across tenants, or enter through an administrative path.

### Q157. What is session fixation protection?

Session fixation protection changes or otherwise protects the session identifier when the security context changes, preventing an attacker from reusing a session ID known before authentication. Spring Security provides session-management protections, but the exact behavior depends on the authentication mechanism and configuration. Verify that authentication rotates the session as expected, logout invalidates it, and concurrent-session or remember-me behavior matches the application's threat model.

### Q158. How do authentication entry points differ from access-denied handlers?

An AuthenticationEntryPoint handles a request that requires authentication when no acceptable authentication is present, commonly producing a 401 response or initiating a login flow. An AccessDeniedHandler handles an authenticated principal that lacks permission, commonly producing 403. API applications should configure these paths deliberately so clients receive consistent machine-readable errors without leaking internal details; test anonymous, invalid/expired credentials, and authenticated-but-forbidden requests.

### Q159. What should security tests cover beyond a happy path?

Test the authorization matrix, not just successful login: anonymous requests, authenticated users lacking the required authority, permitted users, wrong roles/authorities, expired or malformed tokens, wrong issuer/audience, CSRF behavior for cookie-authenticated flows, and cross-tenant resource access. Verify that failures return the intended 401/403 behavior and do not leak internal details. Include method-security tests because endpoint tests may not prove every service entry point is protected. Use framework test support to establish principals where appropriate, but retain integration tests that validate the real authentication configuration and token decoder.

### Official references
- [Spring Security reference](https://docs.spring.io/spring-security/reference/)
- [Servlet architecture](https://docs.spring.io/spring-security/reference/servlet/architecture.html)
- [OAuth 2.0 Resource Server JWT](https://docs.spring.io/spring-security/reference/servlet/oauth2/resource-server/jwt.html)

---

## Spring Boot Testing


### Q160. What is the difference between a unit test and an integration test?

A unit test checks a small unit of behavior in isolation—typically with plain Java objects and mocked collaborators—so failures are fast and easy to diagnose. An integration test checks whether multiple real components cooperate correctly, such as Spring wiring, JSON conversion, transaction boundaries, repository queries, or communication with a database/broker. The important question is what boundary the test actually exercises. A test annotated with Spring test support is not automatically a meaningful integration test if every dependency is mocked. Use unit tests for business rules, focused slice tests for framework behavior, and a smaller set of high-value integration tests against real infrastructure where provider-specific behavior matters.

### Q161. What does `@SpringBootTest` do?

It creates a test `ApplicationContext` using Spring Boot's application setup, so the test can exercise real configuration, bean wiring, and selected infrastructure. Depending on `webEnvironment`, it can use a mock web environment or start a server. That breadth makes it useful for a few end-to-end integration checks, but it is slower and failures can be harder to localize than unit or slice tests. Do not use it as the default for every service test. If the context fails, inspect the root cause and test configuration rather than adding mocks until the test happens to start; the aim is to validate the intended application wiring.

### Q162. What is a test slice?

A test slice loads only the infrastructure relevant to a layer, reducing setup and making failures easier to interpret. Examples include `@WebMvcTest` for MVC behavior and `@DataJpaTest` for JPA/repository behavior. Because the slice intentionally excludes unrelated beans, provide mocks or imports for collaborators the test needs; do not interpret a missing bean as proof that production wiring is broken. Conversely, a slice test cannot prove that the complete application context starts. Use slice tests for focused contracts and keep a smaller number of full-context tests for integration risks.

### Q163. When should you use MockMvc?

`MockMvc` runs requests through Spring MVC's request-handling infrastructure without opening a real network socket. It is effective for testing controller mapping, validation, message conversion, exception handling, status codes, and configured servlet security integration. It is faster and more deterministic than starting a server, but it does not verify actual socket behavior, reverse-proxy settings, TLS termination, connection handling, or every container-specific feature. Use a real-server test when transport/deployment behavior is part of the risk, and keep most controller contract tests at the MockMvc level.

### Q164. What is Testcontainers useful for?

Testcontainers lets integration tests start disposable services—such as the database engine, Kafka, or Redis—in containers, so the test exercises real protocol and engine behavior instead of a simplified substitute. It is particularly valuable for SQL dialect, isolation, locking, migrations, broker acknowledgements, and serialization issues. The trade-off is slower startup and a dependency on a container runtime or configured CI environment. Keep fixtures deterministic, isolate state between tests, pin image versions, and make cleanup reliable. Use it selectively for behaviors that an in-memory fake cannot faithfully represent.

### Q165. What is the difference between `@MockBean` and `@MockitoBean`?

`@MockBean` was Spring Boot's established annotation for adding or replacing a mock in the test `ApplicationContext`. It was deprecated in Boot 3.4 in favor of Spring Framework's `@MockitoBean`; modern Boot/Framework lines should use the replacement supported by their version. `@MockitoSpyBean` is the corresponding spy annotation. This is version-sensitive, so check the migration guide before changing imports in a legacy codebase.

The purpose is to replace a Spring bean dependency while keeping the rest of the context real. It is different from a plain Mockito `@Mock`, which creates an object but does not by itself replace a bean in Spring's context. Prefer plain Mockito for isolated unit tests and context-level bean overrides only when the test genuinely needs Spring wiring. Verify the annotation's supported declaration locations and replacement semantics for the exact Framework/Boot version.

### Q166. Why can Spring tests become flaky or slow?

Common causes include excessive full-context startup, shared mutable test state, nondeterministic asynchronous work, external dependencies, timing assumptions, and misuse of context caching. Prefer deterministic fixtures, bounded waits, isolated test data, and focused contexts. Do not add arbitrary sleeps to conceal race conditions.

### Q167. How should security tests be designed?

Security tests should prove both that legitimate requests succeed and that unauthorized requests fail for the right reason. Cover anonymous callers, authenticated users without the required authority, authorized users, malformed or expired credentials, and relevant JWT claims or tenant boundaries. Test through the actual filter chain with MockMvc or the appropriate integration setup; a service-level test alone cannot prove that request matchers, CSRF, CORS, or authentication entry points are configured correctly. Include negative tests for sensitive endpoints because accidental access is often more damaging than a missing happy-path assertion.

### Q168. When should a test use a real HTTP server instead of MockMvc?

Use a real-server test when transport behavior, actual server configuration, socket handling, compression, or deployed HTTP integration is part of the risk. MockMvc is faster for MVC handler behavior but does not exercise the full network stack.

### Q169. How should asynchronous tests avoid timing races?

Avoid arbitrary Thread.sleep calls because they depend on machine speed and can make a test flaky or unnecessarily slow. Use a bounded future, latch, controlled executor, or condition-based wait with an explicit timeout. Assert the observable completion condition, and on timeout report useful state such as task submission, queue depth, or captured exception. Ensure the test can distinguish “task was never scheduled,” “task failed,” and “task is still running.” A deterministic executor or fake clock is often preferable when the behavior being tested is scheduling logic rather than real concurrency.

### Q170. What is the risk of @DirtiesContext?

@DirtiesContext marks a cached test ApplicationContext for removal, causing Spring TestContext to close and rebuild it rather than reuse it. This can substantially increase suite time and may hide state-isolation problems if used routinely. Use it when a test genuinely changes context-level state that cannot be safely reset; otherwise reset the specific database, mock, cache, or singleton state and keep tests independent.

### Q171. How do you test a migration against an existing schema?

Start from a known baseline schema and run the real migration sequence against the production database engine, including representative data where migration behavior depends on volume or constraints. Verify both final schema and transformed data, not just that the migration command exits successfully.

### Q172. How should test secrets and environment configuration be handled?

Use dedicated test credentials and ephemeral or isolated infrastructure, and load secrets through the test environment rather than committing real credentials. Keep configuration deterministic so tests do not accidentally connect to production systems. Be careful with failed assertions, HTTP wire logs, container output, and test reports that may print request payloads or effective properties; redact tokens and personal data before storing CI artifacts, and rotate any credential that is exposed.

### Official references
- [Spring Boot testing](https://docs.spring.io/spring-boot/reference/testing/index.html)
- [Spring Framework testing](https://docs.spring.io/spring-framework/reference/testing.html)
- [Testcontainers](https://java.testcontainers.org/)

---

## REST Clients and Inter-Service Communication


### Q173. How do `RestTemplate`, `RestClient`, and `WebClient` differ?

`RestTemplate` is the older synchronous template-style client and is deprecated in Spring Framework 7 in favor of `RestClient`. `RestClient`, introduced in Framework 6.1, provides a fluent synchronous API and is a natural choice for new blocking integrations on compatible versions. `WebClient` supports non-blocking reactive I/O and streaming, and is useful when the surrounding application is built around reactive pipelines. These clients are not interchangeable just because they all send HTTP requests: the execution model, connector, timeout configuration, error handling, and integration with reactive backpressure differ. Choose the client that matches the application’s concurrency model and the exact Framework version; do not move a blocking call onto a reactive event loop and assume it has become non-blocking.

### Q174. How should outbound timeouts be configured?

Configure the time spent establishing a connection, waiting for a pooled connection, and waiting for the remote response using the options supported by the selected client/connector. These are different failure points and may have different settings. The total downstream budget must fit inside the caller's overall deadline, leaving time for fallback or response serialization. A timeout should be short enough to release resources during an incident but realistic enough for normal latency. Test timeout behavior and cancellation under load; otherwise, requests can accumulate while each waits for a slow dependency.

### Q175. When are retries appropriate?

Retry only when the failure is plausibly transient, the operation is safe to repeat, and the remaining deadline allows another attempt. Use a small bounded attempt count, exponential backoff with jitter, and an explicit allow-list of retryable failures. Do not retry validation errors, authorization failures, or permanent 4xx responses indiscriminately. A timeout is ambiguous: the server may have completed the write even though the response never reached the caller. Use idempotency keys, deduplication, or reconciliation before retrying side-effecting operations. Coordinate retries across layers because client, gateway, and service retries can multiply traffic during an outage.

### Q176. Why is connection pooling important?

Pooling reuses connections and reduces repeated connection setup, while pool limits provide a resource boundary. Size the pool together with request concurrency, connection-acquisition timeout, downstream capacity, and expected request duration; remember that the total across all application replicas matters. A pool that is too small creates waiting, while an oversized pool can overload the dependency. Monitor active, idle, pending, and acquisition-latency metrics and load-test realistic traffic.

### Q177. How should authentication be propagated between services?

Choose an explicit trust model: separate service credentials, token exchange, or narrowly scoped delegated user tokens depending on the downstream authorization requirement. Do not forward an inbound bearer token to every destination simply because the request originated in your service. Allow-list trusted destinations, review redirect behavior, avoid putting credentials in URLs, redact authorization headers from logs, and define token audience/scope so a leaked token cannot be reused against unrelated services.

### Q178. How should client errors be observed?

Observe outbound clients using latency distributions, success/failure counts, timeout and retry metrics, connection-pool saturation, and a bounded set of dependency labels. Use distributed traces to connect the caller's span with the downstream request when tracing is available, and log a correlation identifier rather than full sensitive payloads. During an incident, distinguish a caller-side timeout from an HTTP error returned by the dependency: the remote service may have completed the operation even when the caller stopped waiting. Keep metric labels low-cardinality so observability does not itself overload the metrics system.

### Q179. What are HTTP Service Clients?

Modern Spring Framework versions support declarative HTTP service interfaces backed by a configured client adapter. They can reduce boilerplate for stable service contracts, but still need explicit timeout, authentication, error mapping, and observability configuration. Verify availability and configuration details for the target Framework release.

### Official references
- [Spring Framework REST clients](https://docs.spring.io/spring-framework/reference/integration/rest-clients.html)
- [Spring Framework 6.1 RestClient announcement](https://spring.io/blog/2023/07/13/new-in-spring-6-1-restclient/)
- [Spring Framework HTTP Service Clients](https://docs.spring.io/spring-framework/reference/integration/rest-clients.html)

---

## Reactive Programming and WebFlux


### Q180. What are `Mono` and `Flux`?

In Project Reactor, `Mono<T>` represents a publisher that can emit zero or one item and then complete or fail; `Flux<T>` can emit zero to many items. They are publisher types that describe a computation or sequence, not containers that necessarily hold an already-computed value. Most pipelines are lazy until subscribed, and operators compose the sequence of work. Neither type automatically creates a new thread or makes blocking code non-blocking. In a web application, the framework subscribes to the returned publisher as part of request processing; avoid calling `block()` on event-loop threads.

### Q181. What is backpressure?

Backpressure is a mechanism for coordinating demand between a subscriber and a publisher so the consumer can signal how many items it is ready to process. It helps manage flow but does not magically bound every external resource or make a blocking call safe.

### Q182. How does WebFlux differ from Spring MVC?

Spring MVC is commonly used with imperative request handling and blocking I/O, often using a thread-per-request model. WebFlux is designed for asynchronous, non-blocking I/O and reactive composition, typically using event-loop workers to serve many concurrent requests. WebFlux is not automatically faster: if every request blocks on JDBC or a blocking HTTP client, the event-loop advantage disappears and throughput can degrade sharply. Choose it when the I/O stack supports non-blocking work and the team understands Reactor, cancellation, context propagation, and backpressure. MVC remains a strong choice for conventional CRUD services with blocking persistence.

### Q183. Why are blocking calls dangerous on event-loop threads?

An event-loop worker is expected to execute short, non-blocking tasks for many connections. If it waits synchronously for JDBC, filesystem I/O, `Thread.sleep`, or a blocking HTTP client, every other task assigned to that worker waits too, so a small number of blocked workers can create a broad latency spike. Prefer a non-blocking client and reactive data access when the full path supports it. If blocking work is unavoidable, isolate it with a suitable bounded scheduler and explicit concurrency limits; this protects the event loop but does not make the blocking work free or infinitely scalable. Measure queueing and thread utilization rather than treating `subscribeOn` as a universal fix.

### Q184. How should errors and retries be handled in a reactive pipeline?

Use Reactor operators to map, recover, or retry at a deliberate boundary. Retry only transient and safe-to-repeat operations, bound attempts, and consider the total deadline. An error recovery operator can change the signal into a fallback value, so avoid accidentally turning genuine failures into apparent success.

### Q185. What is cancellation, and why does it matter?

A subscriber can cancel its subscription when it no longer needs data. Well-designed pipelines propagate cancellation to upstream work where supported, releasing resources and avoiding unnecessary processing. External calls and custom publishers must be examined to ensure cancellation really stops or limits work.

### Q186. How is context propagated in reactive applications?

Reactive execution can hop threads, so ordinary ThreadLocal assumptions do not necessarily hold across the pipeline. Reactor Context is the common mechanism for contextual data, and supported integrations can bridge tracing or security context. Verify context propagation behavior for the exact versions and integrations in use.

### Q187. How does Reactor Context differ from a method parameter?

A method parameter explicitly carries data through a call graph; Reactor Context carries subscriber-associated metadata across a reactive chain. Use Context for cross-cutting metadata such as tracing where supported, but use explicit parameters for business inputs that should remain visible in method contracts.

### Q188. What is the danger of `onErrorResume` used too broadly?

`onErrorResume` replaces an error signal with a fallback publisher. That is appropriate when the fallback is valid for the business contract, but dangerous when it converts a database outage, timeout, or programming defect into an empty result that looks successful. Match the recovery to the exception type, preserve diagnostic context in logs/metrics, and keep expected absence separate from dependency failure. For example, “customer not found” may map to an empty/404 result, while “customer database unavailable” should remain an operational failure. Broadly catching `Throwable` or returning `Mono.empty()` for every failure hides incidents and can cause downstream systems to make incorrect decisions.

### Q189. Why can flatMap increase concurrency unexpectedly?

`flatMap` can subscribe to multiple inner publishers concurrently, and its concurrency/prefetch choices affect how much work is in flight. A high concurrency level can saturate a database pool or downstream API even when the event loop itself remains responsive. Set explicit limits, align them with downstream capacity and request deadlines, and use an ordering-preserving operator when output order is part of the contract.

### Q190. How should retries interact with cancellation?

A retry operator can resubscribe after a failure, but cancellation and deadlines should stop unnecessary work. Ensure retries are bounded and safe to repeat, and test that cancellation does not leave unmanaged side effects running.

### Q191. When should a reactive application use a boundedElastic scheduler?

It can isolate unavoidable blocking work from event-loop threads, but it is not a substitute for a non-blocking dependency and still has finite resource capacity. Bound the work, observe queueing, and prefer reactive I/O when feasible.

### Official references
- [Spring WebFlux reference](https://docs.spring.io/spring-framework/reference/web/webflux.html)
- [Project Reactor reference](https://projectreactor.io/docs/core/release/reference/)

---

## Actuator and Observability


### Q192. What does Spring Boot Actuator provide?

Actuator adds production-oriented management capabilities such as health indicators, metrics integration, application information, and selected diagnostic endpoints. It can connect the application to readiness/liveness probes and monitoring systems, but it is not a full observability platform by itself. Distinguish an endpoint being enabled from it being exposed over HTTP or JMX, and then distinguish exposure from authorization. Expose only what operators and orchestration require, secure sensitive endpoints, and review the actual endpoint set after configuration or version changes. Environment, configuration, heap/thread dump, and shutdown-related data or actions can reveal sensitive information or affect availability.

### Q193. What is the difference between liveness and readiness?

Liveness asks whether the process is alive enough that restarting it may help; readiness asks whether it should receive new traffic. A database outage may make an instance temporarily unready, but making database health part of liveness can restart every replica repeatedly without fixing the dependency. Configure probe groups around the orchestration decision they support, use bounded checks, and avoid expensive dependency fan-out on every probe. Also distinguish startup probes where supported: a slow but healthy application may need time to initialize before liveness evaluation begins.

### Q194. What is Micrometer?

Micrometer is an instrumentation facade that provides a common API for recording counters, timers, gauges, distribution summaries, and related measurements across supported monitoring backends. Spring Boot integrates Micrometer with Actuator and can auto-configure meters for common infrastructure. Instrument business-relevant operations as well as technical resources, but choose bounded dimensions: user IDs, request IDs, raw URLs, and arbitrary exception messages can create unbounded metric cardinality. A timer or histogram is often more useful for latency percentiles than an average alone. Define the question each metric should answer and pair important signals with dashboards and alerts.

### Q195. How should health indicators be designed?

Design a health indicator to answer a specific operator decision, not to prove that every subsystem is perfect. Keep it cheap and bounded; a check that performs an expensive query or fans out to dozens of downstream services can amplify an outage. Decide whether a dependency affects liveness, readiness, or only a separate diagnostic indicator. Optional dependencies should not necessarily remove the whole instance from traffic. Use metrics, traces, and alerts to diagnose degradation; a single UP/DOWN result rarely explains root cause. Test timeout and partial-failure behavior so the health endpoint remains useful when dependencies are unhealthy.

### Q196. What is distributed tracing useful for?

Tracing connects spans across service boundaries to show where time is spent and where failures occur. Propagate context using supported instrumentation, sample according to operational needs, and avoid placing secrets or personal data in span attributes. Trace data complements, rather than replaces, logs and metrics.

### Q197. How should production endpoints be secured?

Expose only the endpoints required by the deployment platform and operators, restrict sensitive environment/configuration details, and apply authentication plus network-level controls. Prefer a separate management port/network where appropriate, while remembering that a separate port is not authentication. After upgrades, inspect the effective endpoint exposure and authorization rules in the deployed environment; do not assume prior defaults or security-chain matching still apply.

### Q198. How should custom health indicators avoid cascading failures?

Use short, bounded timeouts and cheap checks; a health indicator should not execute an expensive business workflow or fan out across every downstream dependency. Decide which dependencies are truly required for readiness and report optional degradation separately where possible. During a shared outage, probes must not create a retry storm or turn a partial dependency failure into every instance becoming unready at once.

### Q199. What is the difference between an application metric and a business metric?

Application metrics measure technical behavior such as latency, errors, pool usage, and saturation. Business metrics measure outcomes such as completed orders or failed checkouts. Both are needed because a service can be technically healthy while business success degrades.

### Q200. How do you design alerts from SLOs?

Start with user-visible service-level indicators such as successful requests and latency within a threshold, then choose alert windows that provide actionable response time without excessive noise. Consider multi-window burn-rate alerts for error-budget consumption, and separate symptoms from diagnostic signals. Pair each alert with a dashboard, runbook, and owner; CPU or memory thresholds alone do not always reveal whether users are affected.

### Q201. Why must observability endpoints be reviewed after an upgrade?

Endpoint availability, default exposure, configuration properties, and security integration can change across release lines. After an upgrade, inspect the actual enabled/exposed endpoint set, test anonymous and authorized access, and verify management-port/network policy. Pay particular attention to environment, configuration, heap/thread dumps, and shutdown endpoints because their data or actions can be sensitive. Treat the previous deployment policy as something to re-verify, not as proof of continued safety.

### Official references
- [Spring Boot Actuator](https://docs.spring.io/spring-boot/reference/actuator/)
- [Micrometer documentation](https://docs.micrometer.io/)

---

## Spring Caching


### Q202. What does `@Cacheable` do?

When caching is enabled and the invocation passes through the cache interceptor, Spring evaluates the cache name and key. If a matching entry exists, the method body is skipped and the cached result is returned; on a miss, the method runs and its result is stored according to the cache configuration. This is proxy-based in the common setup, so self-invocation or manual object construction can bypass caching. Define keys from every input that affects the result, choose TTL/eviction semantics deliberately, and avoid caching values whose freshness or tenant/authorization scope is not represented in the key.

```java
@Cacheable(cacheNames = "productById", key = "#productId")
public Product findProduct(String productId) { ... }
```

The example assumes `productId` uniquely identifies the result within the cache's intended security and tenant scope.

### Q203. How do `@CachePut` and `@CacheEvict` differ?

@CachePut always executes the target method and stores its returned value under the selected key, so it can be useful when an update returns the new representation. @CacheEvict removes entries, commonly after a mutation so later reads fetch the source of truth. Neither annotation makes cache changes automatically atomic with a database transaction in every provider/configuration. If eviction occurs before commit, another reader can repopulate stale data; if eviction happens after commit, a crash can leave stale entries until recovery or TTL. Define the consistency requirement, verify transaction-aware cache behavior, and test commit and rollback paths.

### Q204. Why is cache invalidation difficult?

The source of truth can change through multiple paths, and distributed caches introduce propagation delay and concurrent updates. Define which writes invalidate which keys, consider TTL as a safety bound rather than perfect consistency, and test concurrent read/write scenarios.

### Q205. What is a cache stampede?

A large number of callers miss or expire the same key and simultaneously recompute the value, potentially overwhelming the backing service. Depending on the cache, use request coalescing, controlled refresh, jittered TTLs, or stale-while-revalidate patterns. Measure whether complexity is justified.

### Q206. When should Redis be used instead of an in-memory cache?

A local cache is fast and simple but is per-process and can diverge across replicas. Redis can share entries across instances but adds network latency, operational dependency, and its own consistency/failure modes. Select based on freshness, sharing, availability, and capacity requirements.

### Q207. What happens to caching when the backing cache is unavailable?

Behavior depends on the cache provider and application configuration. Decide whether to fail open by invoking the underlying method or fail closed, and ensure the fallback cannot overwhelm the source of truth. Monitor cache errors and hit ratio rather than assuming the cache is always available.

### Q208. How do cache keys affect correctness and security?

A cache key must include every input dimension that can change the returned result. If a result depends on tenant, locale, permission scope, filters, or feature flags, omitting one can return incorrect data—and in a multi-tenant system, may leak another tenant's result. Use stable, bounded key structures and make invalidation use the same key model. Avoid including secrets or high-cardinality values without a clear need. Test two callers with different authorization/tenant contexts and verify they cannot share a cache entry accidentally.

### Q209. How should cache invalidation interact with transaction commit?

Evicting before a database transaction commits can allow a concurrent reader to reload old data; evicting only after commit has its own failure window. Understand provider transaction awareness and consistency requirements, and consider versioned keys or event-driven invalidation for distributed writers.

### Q210. What is cache penetration?

Cache penetration occurs when repeated requests for keys that do not exist bypass the cache and hit the backing store every time. It can arise from normal lookups for absent IDs or deliberate probing. Depending on the domain, validate identifiers, apply rate limits, or briefly cache a negative result. Negative caching must have a short and understood lifetime so newly created records do not remain invisible. Do not cache authorization failures in a way that lets one caller's denial affect another caller, and distinguish “not found” from a database or network failure.

### Q211. What is stale-while-revalidate?

Stale-while-revalidate serves a cached value while refreshing it asynchronously, improving latency and reducing simultaneous recomputation at the cost of temporarily stale reads. It works when the business contract permits bounded staleness, such as some catalog or reference data. Define the maximum stale interval, refresh failure behavior, and whether callers may trigger duplicate refreshes. It is inappropriate for balances, permissions, or state transitions where stale data can violate correctness or security. Treat stale serving as an explicit consistency decision rather than a generic cache optimization.

### Q212. How do you observe cache usefulness?

Measure hit/miss rate, load duration, evictions, failures, entry count or estimated memory, and the load placed on the underlying data source. A high hit ratio alone does not prove the cache is useful: stale values, expensive serialization, large entries, or invalidation bugs can outweigh the benefit. Segment metrics by a bounded cache name or operation, not by arbitrary keys or user IDs.

### Official references
- [Spring cache abstraction](https://docs.spring.io/spring-framework/reference/integration/cache.html)

---

## Asynchronous Processing and Scheduling


### Q213. What does `@Async` do?

When asynchronous execution is enabled and the call crosses the Spring proxy, `@Async` submits the method body to a configured `TaskExecutor` rather than running it on the caller's thread. It changes the execution boundary, not the business guarantee: the caller may return before the work finishes, failures may be observed later, and the task may be rejected when the executor is saturated. Self-invocation and manually constructed objects can bypass the interceptor. Configure thread count, queue capacity, rejection behavior, shutdown policy, and observability deliberately; never assume an unbounded number of tasks can be queued safely.

```java
@Async("notificationExecutor")
public CompletableFuture<Void> sendNotification(Event event) {
    notificationClient.send(event);
    return CompletableFuture.completedFuture(null);
}
```

The executor named `notificationExecutor` must be configured, and callers should observe the returned future if they need to detect completion or failure.

### Q214. Why should thread pools have bounded queues?

A pool with an unbounded queue can keep accepting tasks even when arrival rate exceeds processing capacity. Queue delay and memory use grow, so requests may time out long before the queued tasks execute. A bounded queue makes overload visible and forces an explicit decision—reject work, apply backpressure, shed low-priority tasks, or persist work in a durable queue. Size the pool and queue using measured service time, arrival rate, downstream limits, and acceptable end-to-end latency; adding threads can worsen contention or exhaust database connections. Monitor active threads, queue depth, task wait time, rejection count, and task duration.

### Q215. How are exceptions from asynchronous methods observed?

Exceptions from fire-and-forget void methods require an appropriate async exception handler because the caller cannot receive them. Methods returning Future or CompletableFuture can surface failures when the result is observed. Ensure failures are logged and measured without duplicating logs at every layer.

### Q216. What is the difference between fixed rate and fixed delay scheduling?

Fixed rate schedules executions according to a target cadence; fixed delay measures the delay after an execution completes before the next run. Actual behavior depends on scheduler configuration and task duration. Long tasks, process pauses, and multiple application instances require deliberate concurrency planning.

### Q217. Why can a scheduled job run more than once in a cluster?

Each replica runs its own Spring scheduler, so a three-replica deployment can execute the same `@Scheduled` method three times at roughly the same time. A Java `synchronized` block or local lock coordinates only threads in one JVM. If the business rule requires one global execution, use a distributed lock or a scheduler with cluster coordination, define lock expiry/recovery, and make the job idempotent in case a process crashes after performing work but before releasing the lock. Also decide what happens when one run takes longer than the schedule interval. For critical jobs, store durable progress and use a uniqueness constraint or idempotency key rather than relying solely on timing.

### Q218. How should transactions interact with async work?

Imperative Spring transactions are generally bound to the current thread, so an `@Async` task does not automatically participate in the caller's transaction. The caller can commit while the async task is still running, or the async task can fail after the caller has already returned. If the background operation needs a transaction, define its own transactional service boundary inside the task. Pass stable identifiers or immutable command data rather than sharing managed JPA entities or a persistence context across threads. If the work must reliably happen after a database commit, use transaction-aware event handling or an outbox/durable messaging pattern rather than launching an untracked fire-and-forget task.

### Q219. How should cron time zones be handled?

Specify the intended time zone explicitly through the scheduling annotation/configuration where supported, rather than relying on the container default. Test daylight-saving transitions, missed execution after downtime, and behavior when multiple replicas run the same schedule. A cron expression only defines trigger timing; it does not provide distributed locking, durable job state, or exactly-once execution. Use a cluster coordination or durable job mechanism when duplicate runs would be harmful.

### Official references
- [Spring task execution and scheduling](https://docs.spring.io/spring-framework/reference/integration/scheduling.html)

---

## Spring Events and Messaging


### Q220. Are Spring application events synchronous by default?

Spring's standard application event multicaster invokes listeners synchronously by default, so the publisher's call does not finish until synchronous listeners finish. A slow listener can therefore increase request latency, and an uncaught listener exception may affect the publisher's flow. Configuring asynchronous event delivery changes the failure and context model; it does not make the event durable, and listeners may not run if the process terminates. Use local events for in-process decoupling, transaction-bound listeners when a specific transaction phase matters, and a durable broker/outbox when delivery must survive process failure or cross service boundaries.

### Q221. What does @TransactionalEventListener provide?

It binds listener execution to a transaction phase, commonly after commit. If no transaction is active, behavior depends on the configured fallback option. It helps coordinate in-process reactions with transaction outcomes but does not guarantee durable delivery if the process crashes after commit.

### Q222. Why is an outbox pattern useful?

The application writes business state and an event record in the same local database transaction. A separate publisher relays the outbox record to the broker and marks progress. This avoids the dual-write gap, but requires handling retries, duplicate publication, cleanup, ordering, and operational monitoring.

### Q223. Why must Kafka consumers be idempotent?

At-least-once processing and retries can deliver the same logical event more than once. Use a durable deduplication key or design side effects to be idempotent. Acknowledgment timing and consumer transactions affect delivery behavior, but they do not remove the need to reason about external side effects.

### Q224. How should listener retries and dead-letter handling be designed?

Classify retryable and non-retryable failures, bound attempts, use backoff, and send poison messages to a monitored dead-letter destination when appropriate. Define replay procedures and preserve enough metadata to diagnose failures without exposing secrets. Ensure retry behavior does not block partitions indefinitely.

### Q225. What is the difference between an application event and a broker message?

An application event is an in-process communication mechanism within an application context. A broker message crosses a process boundary and can provide durable retention and independent consumers, depending on configuration. Use local events for local decoupling, not as a substitute for cross-service reliability.

### Official references
- [Spring application events](https://docs.spring.io/spring-framework/reference/core/beans/context-introduction.html)
- [Spring for Apache Kafka](https://docs.spring.io/spring-kafka/reference/)

---

## Spring Cloud and Microservices Integration


### Q226. What is Spring Cloud responsible for?

Spring Cloud provides integrations and abstractions for common distributed-system concerns such as configuration, service discovery, client-side load balancing, gateways, and resilience. Spring Framework provides core application infrastructure and Boot simplifies configuration and packaging; Cloud components have their own compatibility and maintenance considerations.

### Q227. Why must Spring Cloud and Boot versions be compatible?

Cloud release trains are tested against specific Boot generations and managed dependency combinations. Arbitrarily mixing versions can cause linkage errors, missing classes, or subtle runtime failures. Use the official compatibility guidance and its dependency-management BOM.

### Q228. What is the role of an API gateway?

A gateway provides a controlled edge for routing and cross-cutting policies such as authentication integration, rate limiting, and request transformation. It should not become the only place business authorization exists, and its failure characteristics and bypass paths must be considered.

### Q229. How should timeouts, retries, and circuit breakers work together?

Timeouts bound individual waits, retries attempt recovery from selected transient failures, and circuit breakers limit repeated calls to an unhealthy dependency. Poorly aligned policies can multiply latency and load. Budget the full call chain, avoid retry storms, and define fallback behavior that remains correct.

### Q230. What is service discovery?

Service discovery resolves logical service identities to available instances through a registry or platform mechanism. It introduces freshness, health, and propagation considerations. In orchestrated environments, platform-native DNS and service abstractions may be sufficient; do not add a registry without a need.

### Q231. How should distributed tracing be propagated?

Use supported Micrometer Tracing/OpenTelemetry instrumentation and propagate standard trace context across HTTP and messaging boundaries. Avoid manually inventing incompatible headers or logging full sensitive baggage. Keep trace/request identifiers out of high-cardinality metric labels; put them in traces or structured logs instead. Validate an end-to-end trace through asynchronous processing and verify context propagation across executor, broker, and consumer boundaries.

### Q232. What does a load balancer not solve?

A load balancer distributes requests among selected instances; it does not make an unhealthy dependency healthy or guarantee that each instance has capacity. Stale discovery, slow health checks, sticky sessions, uneven request duration, connection reuse, and overload can still cause poor distribution. Clients still need bounded timeouts, retry discipline, and clear behavior when no instance is eligible. A load balancer can also amplify an outage if it sends traffic to instances that are technically up but saturated. Use readiness signals that reflect traffic eligibility and observe per-instance latency, error rate, and saturation.

### Q233. What is the difference between a retry and a circuit breaker?

A retry repeats a selected operation after a failure; a circuit breaker suppresses calls when recent outcomes indicate a dependency is unhealthy. Retries can amplify load, while a breaker can reject calls that might have succeeded. Configure both around an explicit deadline and operation semantics.

### Q234. Why can timeout values not be selected independently per service?

A request may traverse a gateway and several nested services, so each hop's processing, connection wait, and retry budget must fit within the caller's remaining deadline. Independently generous timeouts can make end-to-end latency unpredictable and keep upstream threads occupied after the caller has given up. Propagate deadlines where supported, reserve time for response handling, and avoid retries that can outlive the original request's useful lifetime.

### Q235. What is a service mesh's relationship to Spring Cloud?

A service mesh can provide infrastructure-level traffic, identity, and telemetry policies, while Spring Cloud provides application-library integrations. They may overlap; avoid implementing the same retry/security policy at multiple layers without understanding precedence and compounded behavior.

### Q236. How should service discovery and readiness interact?

Discovery should ideally expose instances that are eligible to receive traffic, but propagation and caching mean stale instances can still be observed. Clients need bounded timeouts and failure handling even when health registration is configured correctly.

### Q237. How do you avoid duplicate resilience policies across gateway and service clients?

Assign clear ownership for connection/response timeouts, retries, and rate limits at each hop. A gateway may protect ingress while a service client handles a narrowly defined transient failure, but their combined budgets must be documented. Nested retries multiply attempts and can amplify an outage; use bounded retries with backoff/jitter and test the full call path under latency, throttling, and dependency failure.

### Official references
- [Spring Cloud project and compatibility](https://spring.io/projects/spring-cloud)
- [Spring Cloud release-train reference documentation](https://spring.io/projects/spring-cloud#overview)

---

## Deployment and Production Readiness


### Q238. What should a production-ready Spring Boot service include?

At minimum, it needs secure external configuration, bounded resource pools, meaningful readiness/liveness semantics, timeouts for remote dependencies, observable errors and latency, graceful shutdown, tested data migrations, and a rollback strategy. Exact requirements depend on the service's risk and workload.

### Q239. How should a deployment drain requests and background work before termination?

Coordinate readiness removal, server graceful shutdown, consumer pause/stop, in-flight request deadlines, and the platform termination grace period. Consumers should acknowledge only after their work is safely committed, and durable jobs need checkpoints or retries if the process is terminated. Test the full sequence under load; a server's graceful-shutdown setting alone does not protect every background task.

### Q240. How should database pool size be selected?

Estimate the database's sustainable connection capacity, query duration, concurrency, and number of application replicas before choosing a pool size. A large pool can increase database contention and memory usage rather than improve throughput; the total connection count across all pods matters. Monitor active/idle connections, pending borrowers, acquisition latency, transaction duration, and slow queries. Load-test realistic request mixes and failure conditions, and investigate leaks or long-held transactions before increasing the limit. Pool size is a system-level capacity decision shared with database operators, not merely a local application property.

### Q241. Why should container memory limits include more than heap?

The process uses native memory for thread stacks, metaspace, direct buffers, libraries, and other runtime structures in addition to Java heap. Set JVM/container sizing based on measurements and leave headroom for non-heap usage; a heap below the container limit does not guarantee the process will avoid an OOM kill.

### Q242. How should a service handle a dependency outage?

Bound waits with timeouts, limit retries, use circuit breaking when appropriate, shed or queue work within explicit bounds, and return a meaningful failure or degraded response. Decide which functions can operate without the dependency and how recovery is detected.

### Q243. What should be checked before a rolling deployment?

Verify readiness semantics, backward-compatible API and schema changes, graceful termination, consumer rebalance behavior, startup time, and capacity while old and new replicas coexist. Use expand-and-contract database migrations when multiple versions may run simultaneously.

### Official references
- [Spring Boot production-ready features](https://docs.spring.io/spring-boot/reference/actuator/)
- [Spring Boot container images](https://docs.spring.io/spring-boot/reference/packaging/container-images/)

---

## Performance and Troubleshooting


### Q244. How do you investigate a slow Spring REST endpoint?

Break latency into server queueing, application processing, database calls, serialization, and downstream HTTP calls. Use request metrics, traces, SQL timings, thread dumps, and load tests. Avoid attributing the issue to Spring before identifying where time is spent.

### Q245. What does a thread dump reveal?

It shows thread states and stack traces at a point in time, helping identify blocked threads, lock contention, deadlocks, saturated workers, and blocking calls on reactive event loops. Capture multiple dumps when useful to distinguish persistent blocking from transient activity.

### Q246. How do you diagnose connection pool exhaustion?

Correlate active/pending connection metrics with transaction duration, slow queries, request concurrency, and thread stacks. Check for connections held while doing remote calls or waiting on locks. Increasing the pool without checking database capacity may make the incident worse.

### Q247. How should memory growth be investigated?

Compare heap usage after garbage collection, allocation rate, object histograms, heap dumps, and container/native memory metrics. Look for retained collections, unbounded caches or queues, listener registrations, and thread-local/context leaks. A high heap before GC alone does not prove a leak.

### Q248. What causes excessive Spring application startup time?

Expensive bean initialization, broad classpath scanning, remote calls during startup, migrations, large dependency graphs, and eager singleton creation can contribute. Measure before optimizing, and verify that lazy initialization does not simply move failures and latency into production requests.

### Q249. How do you distinguish JVM, database, framework, and downstream bottlenecks?

Combine evidence: CPU and GC metrics, thread states, pool wait times, query plans, trace spans, and application timings. A framework stack frame in a trace does not prove the framework is the root cause; it may be waiting on the database or a remote service.

### Q250. What is a safe performance optimization process?

Establish a representative baseline, define a measurable target, change one major factor at a time, and compare throughput, latency percentiles, errors, resource use, and correctness. Keep a rollback path and avoid tuning based on a single local microbenchmark when production behavior is dominated by I/O or contention.

### Official references
- [Spring Boot Actuator metrics](https://docs.spring.io/spring-boot/reference/actuator/metrics.html)
- [Java Flight Recorder command-line tool (JDK 21)](https://docs.oracle.com/en/java/javase/21/docs/specs/man/jfr.html)

---

## Spring Internals and Advanced Topics


### Q251. What is a BeanDefinition?

A BeanDefinition is container metadata describing how a bean should be created and configured, including its class or factory method, scope, dependencies, and other settings. Component scanning and configuration processing register definitions before regular singleton instantiation. Inspecting definitions helps diagnose missing or unexpectedly configured beans.

### Q252. Why does Spring use bean post-processors?

Bean post-processors provide lifecycle extension points for inspecting or wrapping bean instances. Auto-proxy creators use this infrastructure to apply aspects such as transactions or caching. Infrastructure ordering matters; a bean created too early may miss processing that would normally apply.

### Q253. Why might an MVC interceptor not run even though a servlet filter logs the request?

A filter can run before MVC selects a handler, while an interceptor runs only after a handler has been mapped. The request may be short-circuited by a security filter, fail handler mapping, target a static/resource path, or use a different dispatcher path. Inspect filter-chain order, the selected HandlerMapping, and response status before assuming the interceptor registration is broken.

### Q254. Why can final methods complicate proxying?

Subclass-based proxies cannot override final methods, and final classes cannot be subclass-proxied. Interface-based proxies have different constraints but only intercept calls made through the proxy contract. Verify the proxy type and method visibility when advice unexpectedly does not execute.

### Q255. What is an ApplicationContext hierarchy?

A context can have a parent context, allowing some bean lookups to fall back to parent definitions while child contexts define their own beans. This is useful in certain web or modular arrangements but can complicate visibility and duplicate definitions. Use it only when the isolation and inheritance model is understood.

### Q256. How should custom composed annotations be designed?

Composed annotations combine existing annotations and may provide aliases or defaults to express a domain convention. Keep their semantics narrow, document meta-annotation behavior, and test that component scanning or interception recognizes them. Avoid building opaque annotation frameworks that obscure the actual configuration.

### Q257. How do classpath problems surface in Spring applications?

Missing or incompatible dependencies can cause class-not-found, method-not-found, linkage, or auto-configuration condition failures. Inspect the resolved dependency graph, managed BOM versions, and the first meaningful cause in the stack trace. Avoid fixing a binary incompatibility by randomly upgrading one transitive library.

### Official references
- [Spring Core container](https://docs.spring.io/spring-framework/reference/core/beans.html)
- [Spring AOP proxying](https://docs.spring.io/spring-framework/reference/core/aop/proxying.html)

---

## Spring Version Evolution and Migration


### Q258. What is the major Jakarta migration concern in Spring 6 and Boot 3?

The ecosystem moved from Java EE-era javax.* APIs to Jakarta EE jakarta.* APIs for specifications that changed namespace. Application source, servlet containers, validation providers, and third-party libraries must align. Not every javax.* package moved; Java SE packages such as javax.sql remain.

### Q259. What are the important Spring Framework 7 and Boot 4 baselines?

Framework 7 retains a Java 17 baseline and adopts Jakarta EE 11 APIs, including Servlet 6.1, JPA 3.2, and Bean Validation 3.1. Boot 4 requires Java 17 or later and Framework 7.x. The exact Boot patch may require a specific Framework patch, so consult its system-requirements page.

### Q260. What changed for HTTP clients in Framework 7?

Framework 7 deprecates RestTemplate in favor of RestClient. RestClient was introduced in Framework 6.1. For reactive non-blocking and streaming use cases, WebClient remains relevant. Follow the official migration guidance and check custom request factories, interceptors, converters, and error handling.

### Q261. Why is a major Boot upgrade more than a dependency version change?

A major Boot upgrade can change the Java baseline, managed dependency graph, configuration properties, auto-configuration conditions, removed APIs, servlet/Jakarta integrations, security behavior, test utilities, and observability defaults. Updating the version string alone can compile while still changing runtime behavior. Read the official migration guide and release notes, inspect deprecations and the resolved dependency tree, then test startup, authentication/authorization, persistence/migrations, messaging, outbound clients, graceful shutdown, and actuator exposure. Upgrade through a supported dependency set and keep a rollback plan; avoid overriding individual transitive versions to silence the first compatibility error.

### Q262. How should Spring Cloud compatibility be managed?

Use the official compatibility guidance and corresponding BOM rather than independently pinning arbitrary Cloud components. The current official matrix maps Spring Cloud 2025.1.x to Boot 4.0.x and, starting with 2025.1.2, Boot 4.1.x; Spring Cloud 2025.0.x maps to Boot 3.5.x. Verify the matrix again before upgrading.

### Q263. How do you plan a safe migration?

Inventory runtime and dependencies, identify deprecated APIs, update one major boundary at a time where practical, compile and run focused tests, validate configuration and startup conditions, and exercise security, persistence, and downstream calls. Roll out progressively with metrics and a rollback plan.

### Q264. How should deprecated APIs be handled?

Determine the replacement, migration constraints, and reason for deprecation from the official documentation for the project's exact version. Check whether the API is deprecated for removal, whether behavior changes accompany the replacement, and whether third-party integrations depend on it. Avoid copying legacy examples into new code without labeling the version context. Track deprecations during routine upgrades so a future major-version migration is planned rather than rushed.

### Q265. What is the role of a BOM?

A Bill of Materials (BOM) centralizes a tested set of dependency versions so related libraries are aligned. Spring Boot's dependency management reduces the need to specify each transitive version manually, while Spring Cloud and other ecosystem projects may provide their own BOMs with compatibility constraints. Avoid mixing unrelated release trains or overriding a managed version without a documented reason. When diagnosing NoSuchMethodError or ClassNotFoundException, inspect the resolved dependency graph and compare it with the supported matrix instead of upgrading arbitrary jars one at a time.

### Q266. What are notable Spring Framework 7 additions worth checking during migration?

Framework 7 introduces new capabilities including programmatic bean registration, core resilience features, API versioning support, HTTP Service Client configuration improvements, and RestTestClient. Availability and exact APIs should be verified against the 7.0 reference and the Boot line managing it; do not backport assumptions to Framework 6.x.

### Q267. Which dependency compatibility checks are most important during the Jakarta namespace migration?

Do not perform a blind global replacement of `javax.` with `jakarta.`. Identify which imports belong to specifications that actually changed namespace, then align the servlet container, persistence provider, Bean Validation implementation, and third-party libraries to the target Jakarta EE generation. Keep Java SE packages such as `javax.sql` unchanged, inspect transitive dependencies for old APIs, and use compile errors plus targeted integration tests to find remaining incompatibilities.

### Q268. Why should API versioning support be treated as a contract design choice?

Framework-level versioning mechanisms can help route or negotiate API versions, but they do not decide compatibility policy, deprecation windows, or client migration. Define the contract and test how versions are selected before adopting the mechanism.

### Q269. What should be checked when moving tests to newer Spring test APIs?

Framework and Boot lines may introduce or deprecate test utilities, mock-bean annotations, and client test support. Verify the exact API and dependency baseline, then test context wiring and the actual request/response path rather than only updating imports.

### Q270. How should you treat Spring Framework 7.1 prerelease documentation?

As of 2026-10-24, the official release list identifies Framework 7.0.10 as the latest stable 7.0 release and Framework 7.1.0-RC1 as a prerelease. Prerelease notes can help plan compatibility work, but changes in prerelease documentation must not be presented as released stable behavior. For example, the 7.1 prerelease notes flag changes to forwarded-header filter/transformer constructors; proxy deployments should explicitly review which forwarded-header family they trust. Re-check the official release list and the exact target version's migration guide before upgrading.

### Official references
- [Spring Framework versions and support](https://github.com/spring-projects/spring-framework/wiki/Spring-Framework-Versions)
- [Spring Framework 7.0 release notes](https://github.com/spring-projects/spring-framework/wiki/Spring-Framework-7.0-Release-Notes)
- [Spring Framework 7.1 prerelease notes](https://github.com/spring-projects/spring-framework/wiki/Spring-Framework-7.1-Release-Notes)
- [Spring Framework releases](https://github.com/spring-projects/spring-framework/releases)
- [Spring Boot 4.0 system requirements](https://docs.spring.io/spring-boot/4.0/system-requirements.html)
- [Spring Boot current system requirements](https://docs.spring.io/spring-boot/system-requirements.html)
- [Spring Boot 4.0 migration guide](https://github.com/spring-projects/spring-boot/wiki/Spring-Boot-4.0-Migration-Guide)
- [Spring Cloud compatibility](https://spring.io/projects/spring-cloud)
- [Spring REST clients](https://docs.spring.io/spring-framework/reference/integration/rest-clients.html)

---

## Advanced Spring Ecosystem and Specialized Modules

This chapter supplements the core Spring Framework + Spring Boot handbook with ecosystem areas that commonly appear in senior backend roles. It is intentionally separate from the core chapters because these modules are opt-in and have their own operational models.

### Q271. When should you choose Spring Batch instead of a scheduled service that loops over records?

Spring Batch is designed for restartable, stateful batch workloads with explicit job/step boundaries, chunk processing, checkpoints, skip/retry policies, and execution metadata. A simple scheduled service can be appropriate for a small bounded task, but it must otherwise recreate those concerns itself. Choose based on recovery, scale, auditability, and operational requirements rather than on record count alone.

A typical chunk-oriented step reads a bounded chunk, processes each item, and writes the chunk within a transaction boundary supported by the configured transaction manager. If a process fails, persisted execution state can support restart from a checkpoint when the reader/writer and job parameters are designed for restartability.

### Q272. What is the difference between a Spring Batch job, step, tasklet, and chunk-oriented step?

A job is the overall batch workflow; a step is a unit within that workflow. A tasklet step runs a tasklet repeatedly until it signals completion, which suits operations such as cleanup or a controlled one-off task. A chunk-oriented step reads items, optionally processes them, and writes a chunk, which suits record pipelines. Chunk size affects transaction frequency, memory, throughput, and recovery granularity.

### Q273. How do you make a batch job restartable and safe to rerun?

Use stable identifying job parameters, durable job-repository metadata, restart-aware readers/writers, and explicit policies for what happens when a job is launched again. Avoid treating a random run identifier as a business idempotency key. Make external side effects idempotent or record them durably; a database checkpoint alone cannot atomically commit an email, HTTP call, or message publish with the database transaction.

### Q274. What problem does Spring Modulith solve in a modular monolith?

Spring Modulith helps structure and verify module boundaries inside one deployable application. It supports expressing application modules, checking dependencies, documenting module structure, and testing module behavior. It does not automatically turn modules into separately deployable services; the runtime, database, and deployment may remain shared.

A modular monolith can reduce distributed-systems overhead while preserving explicit boundaries. Module boundaries should reflect domain capabilities, not merely package names or technical layers.

### Q275. How should modules communicate in a modular monolith?

Prefer an explicit module API over direct access to another module's internal implementation. Application events can decouple the initiating module from interested listeners, but event semantics must be clear: synchronous in-process events are not a durable message broker and can still participate in the caller's failure path. If an event must survive process failure or coordinate with an external broker, design durable publication, often using an outbox or another persistence-backed mechanism.

### Q276. When is Spring for GraphQL a better fit than REST?

GraphQL can fit client-driven data requirements where clients need different combinations of related fields and a typed schema is valuable. REST can be simpler for resource-oriented APIs, caching through standard HTTP semantics, file transfer, and straightforward operational boundaries. GraphQL is not automatically faster: resolver fan-out, N+1 database access, expensive queries, and weak query-cost controls can make it more expensive.

Use DataLoader-style batching or equivalent resolver batching where appropriate, bound query depth/complexity, authorize at the correct data boundary, and monitor resolver-level latency.

### Q277. How do you prevent the N+1 problem in GraphQL resolvers?

Identify resolver paths that trigger one query per parent item. Batch related loads for a request using a DataLoader pattern, fetch related data efficiently, and avoid solving every case with an enormous eager join that duplicates rows or loads unused data. Add tests and query-count/trace evidence for representative nested queries. Authorization must still be enforced for each requested object or through a demonstrably equivalent secured query.

### Q278. When should a Spring application use WebSocket or STOMP instead of ordinary HTTP polling?

WebSocket supports a persistent, bidirectional connection and is useful for interactive updates such as collaboration, dashboards, or live status. Polling or server-sent events can be simpler when communication is one-way or update frequency is low. STOMP provides a messaging protocol over WebSocket in supported Spring configurations; it does not by itself solve authentication, authorization, multi-node fan-out, backpressure, or delivery guarantees.

Design connection limits, heartbeat/idle policies, reconnect behavior, authorization of destinations, and a strategy for distributing messages across application instances.

### Q279. How should authentication and authorization work for WebSocket messaging?

Authenticate the connection using a supported mechanism and validate the principal before accepting sensitive subscriptions or messages. Apply authorization to destinations and message operations, not only to the HTTP handshake. Treat client-provided user identifiers and destination names as untrusted input. Consider token expiry, reconnect, cross-origin policy, message size limits, rate limiting, and revocation behavior. Verify the exact integration and defaults for the Spring Security version in use.

### Q280. Which runtime assumptions commonly break when building a Spring application as a native image?

Native-image closed-world analysis may not discover reflective access, dynamically loaded classes, resources, serialization types, or generated proxies. Prefer framework/library-provided runtime hints and supported registration mechanisms, and test the produced executable rather than relying only on JVM tests. Diagnose missing metadata from the native build/runtime error, register only the required access, and measure startup, memory, throughput, build time, and compatibility before choosing native deployment.

Measure startup, memory, throughput, and operational behavior under realistic load before choosing native deployment.

### Q281. What are runtime hints, and why might a native build need them?

Native-image compilation performs closed-world analysis and may not discover every class, constructor, resource, proxy, or reflective access used dynamically. Runtime hints describe required reflection, resources, serialization, or proxy behavior for supported AOT/native processing. Prefer framework/library-provided hints and supported registration APIs; avoid blanket reflection registration because it can increase image size and obscure missing dependencies. Test the produced executable, not only the JVM test suite.

### Q282. How do virtual threads affect Spring Boot application design?

Virtual threads reduce the cost of many blocking tasks by allowing the JVM to schedule large numbers of threads efficiently. They do not make database connections, downstream services, CPU, or file descriptors unlimited. Connection pools, bulkheads, rate limits, timeouts, and bounded admission remain important. Pinning and thread-local usage can also affect behavior depending on the JDK and workload.

Compare virtual-thread and platform-thread deployments with production-like I/O patterns; do not assume a virtual-thread switch alone increases throughput.

### Q283. How is Spring Authorization Server different from an OAuth2 resource server?

An authorization server issues tokens and implements authorization protocol responsibilities such as client registration, consent, and token issuance according to its configuration. A resource server accepts requests bearing access tokens and validates them before granting access to protected resources. A single product may host both roles, but they are distinct trust boundaries. Most business APIs need resource-server behavior, not a custom token-issuing implementation.

Prefer a well-supported identity provider when it meets the requirements. If operating an authorization server, plan for key rotation, client lifecycle, protocol security, consent, revocation, incident response, and ongoing upgrades.

### Q284. Why should authorization be enforced at both request and domain boundaries where needed?

Request-level rules can reject unauthorized operations early, while method- or domain-level checks protect operations invoked through alternate entry points such as scheduled jobs, messaging listeners, or internal service calls. Avoid assuming HTTP filter rules automatically protect non-HTTP invocation paths. Keep authorization rules close enough to the protected operation to remain correct, but avoid scattering inconsistent checks across the codebase.

Test allowed and denied cases, including tenant/object-level access. Authentication alone does not establish that a caller may access every object in the system.

### Q285. What does Spring Session provide, and when is it useful?

Spring Session externalizes and manages HTTP session state using supported backends, allowing sessions to be shared across application instances and reducing dependence on a single node's in-memory session. It is useful for clustered applications that still use server-side sessions. It does not eliminate session fixation, expiration, cookie security, CSRF, serialization compatibility, or backend availability concerns.

Choose a backend and TTL strategy deliberately, secure session cookies, and plan behavior when the session store is unavailable.

### Q286. When is Spring Integration appropriate?

Spring Integration provides enterprise integration patterns such as message channels, routers, filters, transformers, service activators, and adapters for external systems. It is useful when an application has meaningful message-driven integration flows that would otherwise become ad hoc glue code. For a simple direct method call or a small isolated adapter, it may add unnecessary abstraction. Evaluate observability, error channels, retries, ordering, idempotency, and operational complexity.

### Q287. How should retry behavior be designed for Spring integrations?

Retry only failures that are plausibly transient, and bound attempts and total elapsed time. Use backoff and jitter to reduce synchronized retry storms. Do not retry validation errors, authorization failures, or non-idempotent operations blindly. A retry around a remote side effect may repeat an operation whose first response was lost; use an idempotency key or reconciliation when duplicate execution is harmful.

Coordinate retry budgets with client timeouts, circuit breakers, queue redelivery, and upstream deadlines so layered retries do not multiply load.

### Q288. What is the role of Spring Cloud Contract in consumer/provider compatibility?

Contract testing captures expectations between a provider and its consumers and can generate or support tests that validate both sides against those expectations. It helps catch API drift earlier than a late end-to-end test alone. Contracts should describe stable behavior rather than private implementation details, and teams still need compatibility policy for optional fields, enum expansion, error formats, authentication, and version transitions.

A passing contract suite does not prove capacity, resilience, deployment configuration, or full production correctness.

### Q289. How should you test an application that combines batch processing, messaging, and a relational database?

Use focused unit tests for transformation logic, framework-level tests for job/step wiring, and integration tests against the actual database engine where SQL and transaction behavior matter. Test restart after failure, duplicate delivery, idempotent writes, poison messages, and external side-effect boundaries. Use deterministic fixtures and clean up state between tests. Avoid relying only on in-memory substitutes when production behavior depends on locking, isolation, dialect, or database-specific features.

### Q290. How do you decide whether to adopt an additional Spring project?

Start from a concrete requirement and compare the project's capabilities against simpler alternatives. Evaluate compatibility with the selected Boot/Framework release train, maintenance and support status, operational footprint, learning cost, testing support, security posture, and migration path. Avoid adding a project because it is part of the Spring ecosystem; every dependency adds upgrade and runtime considerations. Record the decision and revisit it when requirements or support status change.



### Q291. What does Spring for Apache Kafka add beyond the Kafka client?

Spring for Apache Kafka integrates Kafka producers and consumers with Spring configuration, dependency injection, listener containers, message conversion, transactions, error handlers, retry topics, and testing support. It reduces consumer-loop and producer setup code, but the underlying Kafka model still matters: partitions determine parallelism and ordering, consumer groups determine assignment, and offsets determine progress. Configure serializers/deserializers, group identity, acknowledgement mode, concurrency, error handling, and observability explicitly rather than assuming the framework makes delivery exactly once.

```java
@KafkaListener(topics = "order-created", groupId = "billing")
public void handle(OrderCreated event) {
    billingService.process(event);
}
```

This fragment assumes the listener container, deserializer, topic, and error policy are configured. The handler should be idempotent because redelivery can occur.

### Q292. How should Spring Kafka listener errors, retries, and dead-letter topics be designed?

Classify failures before retrying. A temporary database timeout may justify bounded retry with backoff; invalid payloads or permanent business-rule failures often should not be retried indefinitely. Configure the listener error handler and recovery/dead-letter strategy supported by the selected Spring Kafka version, preserve the original topic/partition/offset and exception context, and monitor retry volume and dead-letter growth. Define who investigates and how a message is replayed. A dead-letter topic is not a fix by itself: replay must be controlled, idempotent, and safe with respect to message ordering and already-applied side effects.

### Q293. When would you choose Spring AMQP/RabbitMQ instead of Kafka?

Choose based on delivery and consumption semantics, not simply familiarity. RabbitMQ and Spring AMQP are often a good fit for queue-based work distribution, routing patterns, acknowledgements, and request/task messaging. Kafka is commonly chosen for durable partitioned logs, replay, consumer groups, and high-throughput event streams where retention and independent consumers matter. Either system can support multiple patterns, so compare ordering, retention, routing, throughput, replay, delivery guarantees, operational expertise, and failure recovery. Spring AMQP does not remove the need to design acknowledgement, retry, dead-lettering, idempotency, and publisher-confirm behavior.

### Q294. How does Spring Data JDBC differ from Spring Data JPA?

Spring Data JDBC targets a simpler aggregate-oriented persistence model and avoids many ORM behaviors such as a first-level persistence context, lazy relationship proxies, and automatic dirty checking. Spring Data JPA builds on JPA and an ORM provider, which is powerful for rich object graphs but introduces entity lifecycle, flush timing, fetch plans, and N+1 risks. JDBC can be easier to reason about when explicit SQL-like persistence and aggregate boundaries fit the domain; JPA can be productive when its mapping and unit-of-work model are understood. Choose from domain complexity, query requirements, transaction semantics, and team expertise rather than assuming one is universally faster.

### Q295. What is Spring Data R2DBC, and when should it be used?

Spring Data R2DBC provides reactive relational database access for drivers that support the R2DBC SPI. It can fit a reactive WebFlux application when database I/O must remain non-blocking through the request path. It is not simply JPA but reactive: it does not use the JPA persistence context or Hibernate's ORM model, and transaction/context handling follows reactive semantics. Do not call blocking JDBC/JPA code from a reactive event loop and describe the application as non-blocking. Use R2DBC when the database/driver, query model, and team are a good fit, and compare operational maturity and required ORM capabilities before migrating.

### Q296. What responsibilities belong in Spring Cloud Gateway?

Spring Cloud Gateway routes requests and can apply edge concerns such as filters, authentication integration, rate limiting, header transformation, and traffic shaping. It is a useful policy boundary for ingress, but it should not be the only authorization layer: direct service calls, internal callers, or alternate paths can bypass the gateway. Keep route predicates and filter order understandable, avoid embedding domain workflows in gateway filters, and define timeouts, retry budgets, request-size limits, and observability. Test that sensitive routes cannot be reached through a broader fallback route and that gateway retries do not duplicate non-idempotent operations.

### Q297. When would you use Spring Cloud OpenFeign instead of RestClient or HTTP Service Clients?

OpenFeign provides a declarative interface style for calling remote services and integrates with Spring Cloud features in compatible release trains. RestClient and Spring Framework HTTP Service Clients provide alternatives within the Framework's own client ecosystem. The best choice depends on existing platform conventions, compatibility, load-balancing/resilience integrations, and team familiarity—not on syntax alone. All of them still require explicit timeouts, authentication, error mapping, observability, and retry/idempotency decisions. Check the current Spring Cloud support status and compatibility matrix before adopting OpenFeign in a new application.

### Q298. How do Spring Cloud Config and Spring Vault differ?

Spring Cloud Config centralizes application configuration and supports serving configuration from a managed source such as a Git repository. Spring Vault integrates with HashiCorp Vault for secret and credential access, including supported authentication and secret lifecycle patterns. They solve related but different problems: not every configuration value is a secret, and storing a password in a configuration repository does not make it safely managed. Restrict access, audit changes, plan rotation and renewal, avoid logging resolved secrets, and define startup behavior when the configuration or secret backend is unavailable. Prefer platform/identity-based access over long-lived credentials where supported, and verify compatibility for the selected Boot/Cloud/Vault versions.

### Official references
- [Spring for Apache Kafka](https://docs.spring.io/spring-kafka/reference/)
- [Spring AMQP](https://docs.spring.io/spring-amqp/reference/)
- [Spring Data JDBC](https://docs.spring.io/spring-data/relational/reference/jdbc.html)
- [Spring Data R2DBC](https://docs.spring.io/spring-data/relational/reference/r2dbc.html)
- [Spring Cloud Gateway](https://docs.spring.io/spring-cloud-gateway/reference/)
- [Spring Cloud OpenFeign](https://docs.spring.io/spring-cloud-openfeign/reference/)
- [Spring Cloud Config](https://docs.spring.io/spring-cloud-config/reference/)
- [Spring Vault](https://docs.spring.io/spring-vault/reference/)

## Senior-level production troubleshooting

### Q299. When should you use `@Bean` instead of `@Component`, and what does `@Configuration` change?

Use `@Component` when you own a class that should be discovered through component scanning. Use `@Bean` when you want to declare explicitly how an object is created, especially for a third-party class you cannot annotate or when construction requires configuration. A `@Bean` method's return value becomes a container-managed bean, so it can participate in dependency injection and lifecycle processing.

A full `@Configuration` class is enhanced by Spring so calls between its `@Bean` methods can preserve container semantics: a call to another bean method resolves the managed bean rather than simply constructing a second object. In “lite” configuration—such as a regular component with `@Bean` methods or certain imports—inter-method calls may behave like ordinary Java calls. Prefer method-parameter injection when one bean method depends on another because it makes the dependency explicit and avoids relying on inter-method calls.

```java
@Configuration
class ClientConfig {
    @Bean
    HttpClient httpClient() {
        return HttpClient.newHttpClient();
    }

    @Bean
    CatalogClient catalogClient(HttpClient httpClient) {
        return new CatalogClient(httpClient);
    }
}
```

### Q300. How do you resolve ambiguous bean injection, and when are conditional beans useful?

If multiple beans match a required type, Spring needs an unambiguous candidate. Use `@Qualifier` when the injection point specifically needs a named role or implementation; use `@Primary` when one candidate should be the default for ordinary type-based injection. Qualifiers are not merely a fallback name lookup—they express which candidate fits that dependency. If the implementations represent different business behaviors, consider whether a strategy map or explicit selection is clearer than making one globally primary.

Conditional registration is useful when a bean should exist only if a class is on the classpath, a property is enabled, or the application has not supplied its own implementation. Spring Boot auto-configuration relies heavily on these conditions. When a bean is missing or unexpectedly duplicated, inspect the condition evaluation report, active profiles, properties, classpath, and user configuration before adding another bean. Avoid conditions that silently disable a critical capability without a clear startup failure or diagnostic message. A related extension point is `FactoryBean`: it lets a factory object create the bean exposed to consumers, which is useful for complex or framework-specific object construction. Normally, looking up the bean name returns the product; prefixing the name with `&` retrieves the `FactoryBean` itself. Use it when the factory abstraction adds real value, not as a replacement for an ordinary `@Bean` method.

### Q301. What is the difference between a servlet filter, a Spring MVC interceptor, and controller advice?

A servlet filter runs in the servlet chain before or after the servlet, so it can act before Spring MVC chooses a controller. Filters are suitable for cross-cutting servlet concerns such as request wrapping, low-level logging, and integrations that must apply beyond controller methods. Spring Security itself integrates through a filter chain.

A Spring MVC `HandlerInterceptor` operates around handler execution after Spring MVC has mapped a request to a handler. It can inspect handler-related information, but it is not a replacement for security controls in the filter chain. `@ControllerAdvice` is different again: it centralizes controller exception handling, binding behavior, or model-related concerns. Choose based on the lifecycle stage you need. If a controller is never reached, inspect filters and handler mapping first; controller advice cannot handle every failure that occurs before MVC invokes a controller.

### Q302. How would you diagnose a request that is unexpectedly allowed or rejected by Spring Security?

Start by confirming which `SecurityFilterChain` matches the request. With multiple chains, ordering and request matchers determine which configuration applies; rules in a chain that never matches cannot protect the request. Then inspect authentication, the resulting principal and authorities, URL authorization rules, method-level security, and any custom filters or authority converters. Enable appropriate security debug logging temporarily in a safe environment, and reproduce with the exact HTTP method, path, headers, cookies, and token.

Also distinguish CORS from authorization. CORS is a browser-enforced cross-origin policy, not an authentication mechanism; a failed browser preflight can prevent the browser from sending the actual request, while a non-browser client may behave differently. CSRF is a separate protection against forged requests that exploit automatically attached browser credentials. Do not disable CSRF or broadly allow origins just to make a failing request pass. Spring Security's [servlet configuration](https://docs.spring.io/spring-security/reference/servlet/configuration/java.html), [CORS guidance](https://docs.spring.io/spring-security/reference/servlet/integrations/cors.html), and [CSRF reference](https://docs.spring.io/spring-security/reference/servlet/exploits/csrf.html) should be checked for the version in use.

### Q303. A `@Cacheable` method always executes, or an `@Async` method runs on the caller thread. How would you investigate?

First verify that the method is called on a Spring-managed bean through the proxy. Self-invocation, manual construction with `new`, or a call path that bypasses the proxy can prevent both cache and async advice from running. Then check whether the relevant annotation processing is enabled, whether the method is eligible for interception, and whether the selected cache/executor configuration is actually active.

For caching, inspect the cache name, generated key, arguments, condition/unless expressions, provider statistics, and eviction behavior. A key that includes a changing argument—or a condition that skips caching—can make every call a miss. For async work, inspect the configured executor, thread names, task rejection and queue metrics, and exception handling. If a custom executor has a direct or caller-runs policy, work may execute on the caller under some conditions. Confirm behavior with a focused test that asserts the cache interaction or records the executing thread; do not infer success from the annotation being present.

### Q304. A connection pool is exhausted under load. What evidence would you collect before changing pool size?

Treat pool exhaustion as a symptom, not proof that the pool is simply too small. Compare active, idle, and pending connection metrics with request concurrency and latency. Check connection-acquisition timeouts, slow SQL, lock waits, transaction duration, leaked connections, and whether code performs remote calls while holding a database transaction open. A thread dump can reveal many request threads waiting for a connection; database-side session and query diagnostics help distinguish slow queries from application-side connection retention.

Increasing the pool can help if the database has capacity and the workload legitimately needs more concurrent connections, but it can also overload the database and worsen latency. Bound request concurrency, set appropriate acquisition/query/transaction timeouts, shorten transaction scope, and fix slow queries or leaks first. Validate the change under representative load and observe both application pool metrics and database saturation. Pool sizing is a coordinated application-and-database decision, not an isolated tuning knob.

### Q305. A configuration property has the wrong value only after deployment. How do you trace it?

Do not assume the YAML file is the effective source. Identify the exact property key and the value expected by the running version, then inspect active profiles and the ordered property sources that can override it—such as command-line arguments, environment variables, external configuration, and packaged files. Check spelling, relaxed binding, profile activation, deployment manifests, and whether the application is reading a similarly named property from a different configuration namespace.

For grouped application settings, prefer validated `@ConfigurationProperties` over scattered string-based injections. Add startup validation for required values and invalid ranges, but never dump secrets or the full environment into logs. A safe diagnostic should reveal the source and effective non-sensitive value, not expose credentials. Reproduce with the same packaged artifact and deployment configuration; “works locally” often means the local property sources differ, not that Spring resolves the same configuration differently.

### Q306. Why might a Spring-managed bean work while a manually instantiated object does not?

Spring-managed beans can receive dependency injection, lifecycle callbacks, post-processing, and proxy wrappers for features such as transactions, caching, method security, and asynchronous execution. An object created with `new` receives none of that container behavior unless your code supplies it explicitly. Its annotations are metadata; they do not independently activate framework infrastructure on an arbitrary Java object.

If a behavior such as rollback, caching, or authorization disappears, check how the object was created and whether calls pass through the expected proxy. Prefer injecting the collaborator into the class that needs it. If manual construction is intentional—for example, a simple value object or a unit-tested domain class—keep it free of infrastructure behavior that requires container management. This distinction is often the quickest explanation for “the annotation is present but nothing happens.”

---


### Rapid last-minute revision path

When interview time is limited, revise these areas first. Use the detailed answers earlier in this same handbook rather than treating this checklist as a separate guide.

1. **Core container:** IoC/DI, constructor injection, bean scopes, lifecycle callbacks, post-processors and circular dependencies.
2. **AOP and proxies:** JDK versus class proxies, self-invocation, final/private method limitations, advisor ordering, and transaction/cache/security annotations.
3. **Spring Boot:** starters versus auto-configuration, condition evaluation, configuration/profile precedence, startup lifecycle, readiness and graceful shutdown.
4. **Web APIs:** MVC request flow, validation, exception mapping, status codes, idempotency, pagination and stable API contracts.
5. **Transactions and JPA:** propagation, isolation, rollback rules, transaction boundaries, persistence context, dirty checking, lazy loading, N+1 queries and locking.
6. **Security:** authentication versus authorization, JWT validation, filter-chain matching, CORS versus CSRF, method security and tenant/object-level access.
7. **Testing:** unit tests versus MVC slices versus full integration tests, Testcontainers, security negative-path tests, context caching and asynchronous test determinism.
8. **Production operations:** Actuator, health/readiness/liveness, metrics/logs/traces, connection-pool saturation, slow endpoints, startup delays and safe configuration diagnostics.
9. **Clients and resilience:** timeout budgets, retries/backoff/jitter, circuit breakers, bulkheads, idempotency and remote side effects after a timeout.
10. **Caching, async and messaging:** cache keys/invalidation, proxy boundaries, executor saturation, durable delivery, retries, dead-letter handling, idempotent consumers and outbox consistency.
11. **Reactive and ecosystem:** MVC versus WebFlux, blocking work in reactive pipelines, Spring Cloud compatibility and the trade-offs of adding ecosystem modules.
12. **Upgrades:** Java/framework baselines, dependency alignment, removed APIs, Jakarta namespace changes, security defaults, integration tests and rollback plans.

For each topic, practise a concise answer in this order: **direct answer → mechanism → realistic example → failure mode/trade-off → how to verify it**.

### Official references retained from the concise guide

- [Spring Framework Reference](https://docs.spring.io/spring-framework/reference/)
- [Spring Boot Reference](https://docs.spring.io/spring-boot/reference/)
- [Spring Boot auto-configuration](https://docs.spring.io/spring-boot/reference/using/auto-configuration.html)
- [Spring Boot testing](https://docs.spring.io/spring-boot/reference/testing/)
- [Spring Framework transaction management](https://docs.spring.io/spring-framework/reference/data-access/transaction.html)
- [Spring Security Reference](https://docs.spring.io/spring-security/reference/)
- [Spring Data JPA Reference](https://docs.spring.io/spring-data/jpa/reference/)
- [Spring Boot Actuator](https://docs.spring.io/spring-boot/reference/actuator/)
- [Spring Framework container extension points](https://docs.spring.io/spring-framework/reference/core/beans/factory-extension.html)
- [Spring Security servlet configuration](https://docs.spring.io/spring-security/reference/servlet/configuration/java.html)
- [Spring Security CORS](https://docs.spring.io/spring-security/reference/servlet/integrations/cors.html)
- [Spring Security CSRF](https://docs.spring.io/spring-security/reference/servlet/exploits/csrf.html)
- [Spring Framework bean overrides for tests](https://docs.spring.io/spring-framework/reference/testing/annotations/integration-spring/annotation-mockitobean.html)
- [Spring WebFlux Reference](https://docs.spring.io/spring-framework/reference/web/webflux.html)

Official documentation is authoritative for behavior and version compatibility. Interview-question sites can suggest themes but cannot establish a universal question frequency.


## Last-minute revision checklist

Use this as the final pass before an interview. If you cannot explain a topic without reciting an annotation name, revisit the mechanism and a real failure scenario.

### Core Spring and container

- Explain IoC versus DI, constructor injection, bean definitions versus instances, singleton scope versus thread safety, and how profiles/conditions affect bean registration.
- Walk through bean lifecycle callbacks, BeanFactoryPostProcessor versus BeanPostProcessor, configuration classes, component scanning, and parent-child contexts.
- Explain proxy-based AOP, JDK versus class proxies, self-invocation, final/private method limitations, advisor ordering, and how to debug a proxy that does not intercept a call.

### Spring Boot

- Distinguish starters, auto-configuration, component scanning, conditional beans, and externalized configuration.
- Diagnose a missing or unexpected bean using the condition evaluation report, classpath, active profiles, property-source precedence, and the first causal exception.
- Explain startup/readiness, runners, graceful shutdown, configuration validation, Actuator exposure, and the difference between a successful startup and a production-ready service.

### Web, APIs, and clients

- Trace a request through the servlet container, filters, Spring Security, DispatcherServlet, handler mapping, argument resolution, validation, controller execution, exception mapping, and message conversion.
- Explain status codes, stable error contracts, idempotency keys, pagination trade-offs, CORS versus CSRF, and controller authorization versus service/domain authorization.
- For outbound clients, discuss connection pools, connect/read/response timeouts, retries with backoff and jitter, circuit breakers, bulkheads, tracing, and duplicate side effects after timeouts.

### Transactions and persistence

- Explain transaction propagation, isolation, rollback rules, checked versus unchecked exceptions, rollback-only outcomes, and the limits of proxy interception.
- Distinguish a database transaction from a distributed transaction; describe why remote calls inside a database transaction can be risky and where the outbox/saga patterns fit.
- Cover persistence-context identity, dirty checking, flush versus commit, lazy loading, N+1 queries, bulk updates, optimistic/pessimistic locking, pagination, and database-specific integration testing.

### Security

- Explain authentication versus authorization, SecurityFilterChain matching and ordering, JWT signature/issuer/audience/expiry validation, password hashing, CSRF, CORS, session fixation, and method-security proxy boundaries.
- Include object-level and tenant-level authorization, secret handling, least privilege, denied-path tests, and safe error responses.

### Testing and operations

- Know when to use unit tests, MVC/WebFlux slices, SpringBootTest, Testcontainers, contract tests, and end-to-end tests—and what each layer cannot prove.
- Cover mocking Spring beans with version-appropriate test annotations, context caching, test isolation, migration testing, and deterministic asynchronous tests.
- Explain health versus readiness versus liveness, metrics versus logs versus traces, low-cardinality labels, pool saturation, thread/heap diagnostics, graceful termination, and production-safe configuration.

### Async, messaging, caching, and reactive systems

- Discuss executor sizing and queue saturation, Async proxy boundaries, scheduler time zones, duplicate scheduled execution across replicas, and graceful cancellation.
- Explain event publication versus durable messaging, retries and dead-letter handling, idempotent consumers, outbox consistency, cache key design, invalidation, TTLs, and stampede protection.
- For WebFlux, explain the reactive execution model, backpressure, context propagation, cancellation, and why blocking JDBC/JPA calls can undermine the benefit of a reactive stack.

### Versioning and senior-level judgment

- Verify Framework, Boot, Java, Spring Cloud, and ecosystem compatibility instead of mixing examples from unrelated release trains.
- For migrations, identify removed/deprecated APIs, dependency alignment, configuration changes, security behavior, test updates, observability, rollback strategy, and a staged smoke-test plan.
- Structure senior answers as: **direct answer → mechanism → concrete example → failure mode/trade-off → how to verify or operate it**. Be precise about defaults and version boundaries; say what you would measure or test rather than guessing.


## Official documentation

Use the documentation matching the versions selected by the project; these ecosystem modules do not all share the same release cadence.

- [Spring Batch Reference](https://docs.spring.io/spring-batch/reference/)
- [Spring Modulith Reference](https://docs.spring.io/spring-modulith/reference/)
- [Spring for GraphQL Reference](https://docs.spring.io/spring-graphql/reference/)
- [Spring Framework WebSocket Reference](https://docs.spring.io/spring-framework/reference/web/websocket.html)
- [Spring Boot AOT and native image documentation](https://docs.spring.io/spring-boot/reference/packaging/aot.html)
- [Spring Boot task execution and virtual threads](https://docs.spring.io/spring-boot/reference/features/task-execution-and-scheduling.html)
- [Spring Authorization Server Reference](https://docs.spring.io/spring-authorization-server/reference/)
- [Spring Session Reference](https://docs.spring.io/spring-session/reference/)
- [Spring Integration Reference](https://docs.spring.io/spring-integration/reference/)
- [Spring Cloud Contract Reference](https://docs.spring.io/spring-cloud-contract/reference/)
- [Spring Boot system requirements](https://docs.spring.io/spring-boot/system-requirements.html)

---
