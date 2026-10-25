# Spring Framework, Spring Boot, and Ecosystem Interview Handbook

**Audience:** Java backend engineers with 5–6 years of experience preparing for senior backend interviews.

**Purpose:** A topic-organized, interview-first revision handbook covering Spring Framework, Spring Boot, and the related Spring ecosystem. Answers explain the mechanism, practical implications, common failure modes, and trade-offs rather than giving one-line definitions. Use short snippets only when they clarify behavior.

**Version note:** Spring APIs and defaults evolve. The stable documentation available at review time lists Spring Framework 7.0.x and 6.2.x; Spring Boot and ecosystem projects have their own compatibility lines. For version-specific answers, verify against the exact Framework, Boot, Java, and Spring Cloud versions used by the target application. [Official Spring Framework reference](https://docs.spring.io/spring-framework/reference/) · [Official Spring Boot reference](https://docs.spring.io/spring-boot/reference/).

## How to use this handbook

- For a first pass, study the mechanism and the reason a feature exists.
- For senior interviews, add a production example, a failure mode, and a trade-off to each answer.
- For troubleshooting questions, explain what you would inspect first and why.
- Treat repeated concepts across topics as deliberate reinforcement where they affect different layers.
- This is an interview-preparation reference, not a claim that every sample is a runnable application.

## Contents

1. [Spring Fundamentals](#spring-fundamentals)
2. [IoC and Dependency Injection](#ioc-and-dependency-injection)
3. [Beans and Scopes](#beans-and-scopes)
4. [Configuration](#configuration)
5. [Spring AOP](#spring-aop)


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

No. A singleton service can be called concurrently by many request threads. Prefer stateless services and keep request-specific data in method-local variables or explicit request-scoped collaborators. Mutable shared fields require a sound concurrency design.

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

A starter declares a curated dependency set. Auto-configuration is conditional configuration that registers infrastructure when prerequisites match. Adding a starter often brings libraries that allow auto-configuration to activate, but the two concepts are distinct.

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

Both express external configuration. YAML can represent nested structures compactly, while properties files use dotted keys. Choose one consistent style per configuration area, avoid duplicate conflicting definitions, and validate effective values in each deployment environment.

### Q78. How are profile-specific files selected?

Boot supports profile-specific configuration files and profile activation through supported configuration sources. Active profiles influence both property loading and beans annotated with `@Profile`. Avoid embedding production secrets in profile files committed to source control.

### Q79. What are configuration imports used for?

Boot configuration imports allow additional configuration sources to be loaded through its external configuration model. Optionality, location semantics, and integration-specific behavior depend on the source type and Boot version. Use the official reference for the exact syntax rather than assuming arbitrary remote URLs are automatically supported.

### Q80. Why use immutable or validated configuration properties?

A dedicated configuration-properties type documents a coherent group of settings and supports type conversion and validation. Validate required URLs, durations, limits, and pool sizes at startup. Keep secrets out of object string representations and diagnostic logs.

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

DispatcherServlet is Spring MVC's front controller. It coordinates handler mapping, handler execution, argument resolution, return-value handling, message conversion, and exception resolution. It does not itself contain application business logic; it orchestrates the web request pipeline.

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

Request validation checks shape and basic constraints at the boundary. Domain validation protects invariants regardless of whether the operation came from HTTP, messaging, or a job. Do not rely exclusively on controller annotations for business correctness.

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

It centralizes cross-controller concerns such as exception-to-response mapping and binding behavior. `@RestControllerAdvice` combines advice semantics with response-body handling. Keep handlers specific enough to preserve meaningful status codes and avoid turning unexpected defects into misleading client errors.

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

@Transactional declares transaction semantics for a method or class when transaction management is configured. In common Spring proxy mode, calls passing through the proxy are intercepted and a transaction manager coordinates the relevant resource transaction. The annotation does not make arbitrary remote calls atomic with a database write.

### Q110. What is transaction propagation?

Propagation determines how a method participates when a transaction already exists. REQUIRED joins an existing transaction or starts one; REQUIRES_NEW suspends an existing transaction and starts a separate one where supported; SUPPORTS can run with or without a transaction. Choose based on business consistency, locking, and failure semantics rather than treating propagation as a performance toggle.

### Q111. What is transaction isolation?

Isolation controls which effects of concurrent transactions are visible, subject to the database and transaction manager. Standard levels describe anomalies such as dirty reads, non-repeatable reads, and phantom reads, but exact behavior depends on the database implementation. Validate assumptions against the production database, not only an in-memory test database.

### Q112. Does Spring roll back for every exception?

Traditional declarative defaults generally roll back for unchecked exceptions and Errors, not all checked exceptions. Explicit rollback rules can change this. Since Spring Framework 6.2, transaction management can also be configured with a global `rollbackOn = ALL_EXCEPTIONS` policy; version and transaction-specific rollback rules still matter. A caught exception that never escapes the transactional boundary may not trigger rollback automatically, and rollback-only status can lead to an unexpected rollback at commit time.

### Q113. Why might @Transactional appear not to work?

Common causes include self-invocation bypassing the proxy, a method not being intercepted, transaction management not being enabled/configured, an unexpected transaction manager, or exception handling that prevents rollback rules from applying. Inspect transaction logs, proxy boundaries, propagation, and the actual exception path.

### Q114. What is the difference between JdbcTemplate and NamedParameterJdbcTemplate?

JdbcTemplate provides JDBC operations with resource handling and exception translation, using positional placeholders. NamedParameterJdbcTemplate supports named parameters that are expanded into JDBC placeholders. Both reduce repetitive JDBC plumbing but do not eliminate the need for correct SQL, transaction boundaries, or query performance analysis.

### Q115. What does readOnly=true guarantee?

It communicates that a transaction is intended to be read-only and may allow transaction-manager or persistence-provider optimizations. It is not a universal guarantee that the database will reject writes; behavior depends on the transaction manager, driver, and database. Do not use it as a security control.

### Q116. Why is a database transaction not enough for reliable Kafka publishing?

A local database transaction and a broker publish usually involve separate resource managers. One can succeed while the other fails, creating inconsistent state. Depending on the use case, use an outbox pattern, idempotent consumer, retry strategy, or a deliberately designed transaction integration; do not assume @Transactional alone creates distributed atomicity.

### Q117. Which coding patterns hold JDBC connections longer than intended?

Long transactions that include remote HTTP calls, slow processing, user interaction, or large result iteration can retain a connection while doing work that does not require the database. Keep transaction boundaries around the required database operation, avoid remote I/O inside them, and use bounded streaming/chunking where appropriate. Confirm the actual connection-acquisition/release lifecycle for the chosen transaction manager and data-access API.

### Q118. What resource-pool risk does REQUIRES_NEW introduce?

When an outer transaction holds a connection and a proxied inner call starts a REQUIRES_NEW transaction, the outer resources may remain bound while the inner transaction needs another connection. Under high concurrency, a small pool can become exhausted or cause threads to wait for connections held by their own suspended outer transactions.

Use REQUIRES_NEW only when independent commit semantics are required. Size and test the connection pool against nested transactional call patterns, keep transaction scopes short, and remember that self-invocation may prevent the intended propagation from being applied.

### Q119. When is NESTED propagation useful?

Where the transaction manager and resource support it, NESTED can use a savepoint inside an existing transaction. It is not universally supported and should not be assumed equivalent to REQUIRES_NEW, which creates a separate transaction.

### Q120. Why can a caught exception still lead to rollback failure at commit?

An inner operation may mark the transaction rollback-only before the exception is caught. The outer method then returns normally, but commit cannot succeed and may result in an unexpected rollback exception. Trace where rollback-only was set rather than focusing only on the final return path.

### Q121. What is the difference between a local transaction and a distributed workflow?

A local transaction coordinates supported resources through one transaction manager. A workflow spanning HTTP services, databases, and a broker needs explicit reliability semantics such as outbox, idempotency, saga compensation, or a supported distributed transaction protocol where appropriate.

### Q122. How can transaction boundaries affect connection pool usage?

A transaction may hold a connection until completion. Long computation or remote calls inside the boundary increase hold time and pool pressure. Measure transaction duration and connection acquisition latency, and keep the critical section focused on required database work.

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

It provides repository abstractions, derived query parsing, pagination and sorting integration, projections, auditing support, and extension points. JPA still defines persistence semantics and the provider implements them; Spring Data does not eliminate the need to understand entity state, query plans, or transaction boundaries.

### Q128. How do repository interfaces work without a handwritten implementation?

Spring Data creates repository proxy implementations based on repository metadata and configured infrastructure. Method names, annotations, and fragments describe query behavior. When a derived query becomes hard to understand, prefer an explicit `@Query` or a custom repository fragment rather than an opaque method name.

### Q129. What is the purpose of Page, Slice, and List return types?

A Page commonly includes total-count metadata and may require an additional count query. A Slice indicates whether more data exists without necessarily calculating the full total. A List returns results without pagination metadata. Choose based on product needs and query cost.

### Q130. What is a projection?

A projection retrieves a subset or alternate view of data instead of exposing the full entity model. Interface and DTO projections can reduce unnecessary data transfer, but actual SQL and optimization depend on query form and provider behavior. Inspect generated SQL and query plans for critical paths.

### Q131. How should dynamic filtering be implemented?

Use a supported approach such as Specifications, query-by-example where suitable, or a custom repository query. Validate allowed filter and sort fields, combine predicates deliberately, and avoid string-concatenating untrusted SQL. Query design should remain understandable and testable.

### Q132. How do N+1 queries arise?

An initial query loads a collection of entities, then accessing relationships triggers additional queries per entity. Detect it with SQL logs, tracing, query-count assertions, and database metrics. Fix with an appropriate fetch strategy, projection, entity graph, or query redesign; indiscriminately eager-loading everything can create oversized joins.

### Q133. How do optimistic and pessimistic locking differ?

Optimistic locking detects conflicting updates, commonly using a version attribute, and fails one competing update rather than holding a long-lived lock throughout work. Pessimistic locking requests database locks for selected records. Choose based on contention, transaction duration, retry policy, and database capabilities.

### Q134. What repository abstractions should you use?

The available interfaces vary by Spring Data version. Common options include `CrudRepository`, `ListCrudRepository`, `PagingAndSortingRepository`, and `JpaRepository`; do not assume inheritance relationships from one version apply unchanged to another. Select the smallest useful contract and consult the reference for the chosen release.

### Q135. Why can repository pagination return inconsistent results during concurrent writes?

Offset pages are positional, so inserts/deletes between requests can shift records across page boundaries. A deterministic sort reduces ambiguity but does not create a consistent snapshot across independent requests. For large changing datasets, consider cursor/keyset pagination and document its consistency model.

### Q136. How do Specifications help with dynamic queries?

Specifications compose predicates in a type-aware way for supported Spring Data JPA setups. They are useful when filters are optional, but complex joins and fetches can become difficult to reason about. Keep sorting, filtering, and fetch strategy deliberate and inspect generated SQL.

### Q137. Why can @Modifying queries leave managed entities stale?

Bulk JPQL/native updates can bypass ordinary entity-by-entity state tracking. The persistence context may still hold old values. Clear or refresh state when appropriate and understand flush/clear options; test subsequent reads in the same transaction.

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

Authentication establishes the identity or principal associated with a request. Authorization decides whether that principal may perform an action on a resource. A valid login or JWT does not automatically imply permission to access every endpoint or tenant's data.

### Q142. What is SecurityFilterChain?

SecurityFilterChain defines servlet filters and request-matching security rules applied to matching requests. Modern configuration commonly declares one or more SecurityFilterChain beans instead of extending the removed WebSecurityConfigurerAdapter pattern. Multiple chains require deliberate ordering and matchers because the first matching chain controls processing.

### Q143. What is the difference between 401 and 403?

401 indicates that the request lacks acceptable authentication credentials or authentication failed. 403 indicates the request was understood but access is denied for the current principal, although exact responses can be customized. A configured entry point and access-denied handler influence the response behavior.

### Q144. How should a REST API validate JWTs?

Prefer Spring Security's OAuth 2.0 Resource Server support and configure trusted issuer or JWK settings rather than hand-writing token parsing. Validate signature and relevant claims such as issuer, expiry, and audience when required by the trust model. A decoded JWT is not trustworthy until cryptographic and claim validation succeeds.

### Q145. What is SecurityContextHolder?

It provides access to the current SecurityContext, which holds authentication information. The default servlet strategy is typically thread-bound; asynchronous execution and reactive pipelines have different context-propagation considerations. Do not assume security context automatically follows arbitrary executor tasks.

### Q146. What is the difference between roles and authorities?

Authorities are granted permission strings. Role-based checks are a convention that often adds a ROLE_ prefix when using role-oriented APIs. Be consistent about prefixes and use authorities that express the actual permission needed rather than overloading broad roles for every decision.

### Q147. Why does CSRF protection matter, and when should it remain enabled?

CSRF protection mitigates cross-site request forgery when browsers automatically attach credentials such as session cookies. Stateless APIs using non-ambient bearer tokens have a different threat profile, but disabling CSRF should follow a deliberate authentication and browser-client analysis, not a blanket rule for every endpoint called REST.

### Q148. What are CORS and CSRF, and why are they not interchangeable?

CORS is a browser-enforced cross-origin access policy; CSRF is an attack in which a victim's browser is induced to submit an authenticated request. CORS is not a general server-side authorization mechanism and does not replace CSRF defenses.

### Q149. How should method-level security be enabled and used?

Enable method security using the supported configuration for the selected version and apply annotations such as @PreAuthorize at appropriate service boundaries. Because interception is proxy-based in common setups, self-invocation and proxy visibility matter. Authorization rules should be tested for both allowed and denied principals.

### Q150. How should multiple SecurityFilterChain beans be ordered?

Each chain has a request matcher and an order. The first matching chain is used, so a broad high-priority matcher can shadow a more specific chain. Keep matchers explicit and test both intended and unintended paths.

### Q151. What is the difference between UserDetailsService and AuthenticationProvider?

UserDetailsService loads user-related data for authentication flows that use it; AuthenticationProvider performs an authentication strategy and returns an authenticated result or rejects the attempt. They are extension points with different responsibilities, not interchangeable service interfaces.

### Q152. How should passwords be stored?

Store password hashes using a supported adaptive password encoder, such as the framework's delegating encoder configuration, and never store plaintext or reversible encrypted passwords as a substitute. Choose parameters appropriate to the deployment and plan migration when algorithms or parameters change.

### Q153. What is the purpose of a PasswordEncoder?

It defines how passwords are encoded and how a submitted password is checked against a stored encoded value. Use an adaptive password-hashing algorithm with cost parameters appropriate to the deployment; never decrypt a password or store plaintext. A delegating encoder can identify the algorithm for each stored hash and support gradual upgrades when the preferred algorithm or work factor changes.

### Q154. What is method security's proxy limitation?

Method security commonly uses Spring AOP interception. A direct self-call may bypass the proxy, and non-proxy-compatible methods may not be intercepted as expected. Put authorization at a clear service boundary and test the invocation path.

### Q155. Should authorization exist only at the controller layer?

No. Controller rules protect the HTTP entry point, but the same business operation may be invoked by a scheduled job, message listener, batch process, or another service. Enforce authorization at a boundary that covers every relevant caller, using method security or domain/application checks where appropriate. Keep request-shape validation separate from permission checks, and test allowed and denied paths through each meaningful entry point.

### Q156. How should multi-tenant authorization be enforced?

Derive the caller's tenant from trusted authentication claims or a verified server-side mapping, then verify that the requested resource belongs to that tenant at the service or data-access boundary. Apply the same tenant context to asynchronous work, scheduled jobs, exports, caches, and message consumers. Never trust a client-supplied tenant ID by itself; test cross-tenant object IDs and ensure query filters, cache keys, and administrative paths cannot bypass isolation.

### Q157. What is session fixation protection?

Session fixation protection changes or otherwise protects the session identifier when the security context changes, preventing an attacker from reusing a session ID known before authentication. Spring Security provides session-management protections, but the exact behavior depends on the authentication mechanism and configuration. Verify that authentication rotates the session as expected, logout invalidates it, and concurrent-session or remember-me behavior matches the application's threat model.

### Q158. How do authentication entry points differ from access-denied handlers?

An AuthenticationEntryPoint handles a request that requires authentication when no acceptable authentication is present, commonly producing a 401 response or initiating a login flow. An AccessDeniedHandler handles an authenticated principal that lacks permission, commonly producing 403. API applications should configure these paths deliberately so clients receive consistent machine-readable errors without leaking internal details; test anonymous, invalid/expired credentials, and authenticated-but-forbidden requests.

### Q159. What should security tests cover beyond a happy path?

Cover anonymous access, authenticated-but-forbidden access, authorized access, malformed/expired tokens, wrong issuer or audience, authority mapping, CSRF where relevant, and tenant isolation. A successful test alone does not establish that forbidden requests are rejected.

### Official references
- [Spring Security reference](https://docs.spring.io/spring-security/reference/)
- [Servlet architecture](https://docs.spring.io/spring-security/reference/servlet/architecture.html)
- [OAuth 2.0 Resource Server JWT](https://docs.spring.io/spring-security/reference/servlet/oauth2/resource-server/jwt.html)

---

## Spring Boot Testing


### Q160. What is the difference between a unit test and an integration test?

A unit test isolates a small unit and usually replaces collaborators with test doubles. An integration test verifies that components work together across real framework configuration or infrastructure boundaries. Use the smallest test scope that proves the behavior, but retain integration coverage for wiring, serialization, transactions, and database-specific behavior.

### Q161. What does @SpringBootTest do?

It creates an application context using Boot's test support and can load a broad slice of the application. It is useful for integration behavior but may be slower and more coupled to configuration than a focused test. Avoid using it for every simple service unit test.

### Q162. What is a test slice?

A slice annotation loads a focused subset of application infrastructure for a testing purpose, such as MVC or JPA. It improves feedback speed and helps isolate concerns, but components outside the slice may need explicit mocks or test configuration. A slice test does not prove that the entire production context starts.

### Q163. When should you use MockMvc?

MockMvc exercises Spring MVC request handling through the test framework without requiring a live HTTP server. It can verify routing, serialization, validation, status codes, and security filters when configured. Use a real-server test when network/server behavior itself matters.

### Q164. What is Testcontainers useful for?

Testcontainers starts disposable infrastructure such as a database or broker for tests, allowing verification against a real engine rather than relying solely on an in-memory substitute. It improves fidelity but adds runtime and environment requirements; keep test data deterministic and ensure containers are cleaned up.

### Q165. What is the difference between @MockBean and @MockitoBean?

`@MockBean` was the established Spring Boot test annotation for replacing beans in an application context. It was deprecated for removal in Spring Boot 3.4 in favor of Spring Framework's `@MockitoBean`, and Boot 4 removes `@MockBean`/`@SpyBean` support. Use `@MockitoBean` (and `@MockitoSpyBean` for spies) on modern Framework/Boot lines, but retain the older annotation only when maintaining a compatible legacy Boot line.

The migration is not always a mechanical import swap: on Framework 6.2/7.0, use the supported test-class field or type-level declarations rather than declaring mocks inside configuration classes. Framework 7.1 prerelease documentation adds constructor-parameter support for these annotations; do not assume that API is available on the stable 7.0 line. Follow the exact Boot/Framework migration guide and verify test-context replacement semantics.

### Q166. Why can Spring tests become flaky or slow?

Common causes include excessive full-context startup, shared mutable test state, nondeterministic asynchronous work, external dependencies, timing assumptions, and misuse of context caching. Prefer deterministic fixtures, bounded waits, isolated test data, and focused contexts. Do not add arbitrary sleeps to conceal race conditions.

### Q167. How should security tests be designed?

Test anonymous access, authenticated-but-forbidden access, authorized access, malformed credentials, and relevant claim/authority combinations. Verify the actual filter chain and endpoint rules, not only service-level authorization annotations. Include negative tests to prevent accidental exposure.

### Q168. When should a test use a real HTTP server instead of MockMvc?

Use a real-server test when transport behavior, actual server configuration, socket handling, compression, or deployed HTTP integration is part of the risk. MockMvc is faster for MVC handler behavior but does not exercise the full network stack.

### Q169. How should asynchronous tests avoid timing races?

Use bounded futures, latches, Awaitility-style conditions, or controlled executors rather than arbitrary sleeps. Assert a clear completion condition and include diagnostics on timeout so the test reveals whether work was never scheduled, rejected, or stalled.

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


### Q173. How do RestTemplate, RestClient, and WebClient differ?

RestTemplate is the established synchronous template-style client; it is deprecated in Spring Framework 7.0 in favor of RestClient. RestClient is a fluent synchronous client introduced in Framework 6.1. WebClient is designed for non-blocking reactive I/O and streaming. Verify the framework baseline before using RestClient; do not assume it exists in older Framework versions.

For new synchronous integrations on Framework 6.1+, evaluate RestClient. For non-blocking reactive pipelines, consider WebClient. Existing RestTemplate applications should follow the official migration guide and can migrate incrementally.

### Q174. How should outbound timeouts be configured?

Configure connection establishment and response/read timeouts using the chosen client and request factory/connector. A timeout budget should align with the caller's deadline and leave time for response handling. An absent timeout can tie up threads and connections during a downstream incident.

### Q175. When are retries appropriate?

Retries can help with transient failures when the operation is safe to repeat and the total deadline allows it. Use bounded attempts, backoff, jitter, and clear retryable-exception rules. Retrying non-idempotent writes without deduplication can create duplicate side effects and amplify an outage.

### Q176. Why is connection pooling important?

Pooling reuses connections and reduces repeated connection setup, while pool limits provide a resource boundary. Size the pool together with request concurrency, connection-acquisition timeout, downstream capacity, and expected request duration; remember that the total across all application replicas matters. A pool that is too small creates waiting, while an oversized pool can overload the dependency. Monitor active, idle, pending, and acquisition-latency metrics and load-test realistic traffic.

### Q177. How should authentication be propagated between services?

Choose an explicit trust model: separate service credentials, token exchange, or narrowly scoped delegated user tokens depending on the downstream authorization requirement. Do not forward an inbound bearer token to every destination simply because the request originated in your service. Allow-list trusted destinations, review redirect behavior, avoid putting credentials in URLs, redact authorization headers from logs, and define token audience/scope so a leaked token cannot be reused against unrelated services.

### Q178. How should client errors be observed?

Capture latency, outcome, timeout/retry counts, and dependency identity with low-cardinality metric labels. Add distributed tracing when supported and avoid logging sensitive request bodies. Separate caller-side timeouts from server-side error responses during diagnosis.

### Q179. What are HTTP Service Clients?

Modern Spring Framework versions support declarative HTTP service interfaces backed by a configured client adapter. They can reduce boilerplate for stable service contracts, but still need explicit timeout, authentication, error mapping, and observability configuration. Verify availability and configuration details for the target Framework release.

### Official references
- [Spring Framework REST clients](https://docs.spring.io/spring-framework/reference/integration/rest-clients.html)
- [Spring Framework 6.1 RestClient announcement](https://spring.io/blog/2023/07/13/new-in-spring-6-1-restclient/)
- [Spring Framework HTTP Service Clients](https://docs.spring.io/spring-framework/reference/integration/rest-clients.html)

---

## Reactive Programming and WebFlux


### Q180. What are Mono and Flux?

In Project Reactor, `Mono<T>` represents an asynchronous sequence with zero or one item, while `Flux<T>` represents a sequence of zero to many items. They describe a publisher pipeline, not necessarily a separate thread or automatic parallel execution.

### Q181. What is backpressure?

Backpressure is a mechanism for coordinating demand between a subscriber and a publisher so the consumer can signal how many items it is ready to process. It helps manage flow but does not magically bound every external resource or make a blocking call safe.

### Q182. How does WebFlux differ from Spring MVC?

Spring MVC is commonly used with a thread-per-request, blocking programming model. WebFlux is designed for non-blocking reactive processing and can use a small event-loop worker set. WebFlux is most beneficial when the whole I/O path is non-blocking and concurrency demands justify the model's complexity.

### Q183. Why are blocking calls dangerous on event-loop threads?

Blocking work prevents an event-loop worker from servicing other tasks, causing latency and throughput degradation. Use reactive clients and reactive data access where appropriate, or isolate unavoidable blocking work on a suitable bounded scheduler. Moving blocking work to another scheduler still consumes threads and must be capacity-planned.

### Q184. How should errors and retries be handled in a reactive pipeline?

Use Reactor operators to map, recover, or retry at a deliberate boundary. Retry only transient and safe-to-repeat operations, bound attempts, and consider the total deadline. An error recovery operator can change the signal into a fallback value, so avoid accidentally turning genuine failures into apparent success.

### Q185. What is cancellation, and why does it matter?

A subscriber can cancel its subscription when it no longer needs data. Well-designed pipelines propagate cancellation to upstream work where supported, releasing resources and avoiding unnecessary processing. External calls and custom publishers must be examined to ensure cancellation really stops or limits work.

### Q186. How is context propagated in reactive applications?

Reactive execution can hop threads, so ordinary ThreadLocal assumptions do not necessarily hold across the pipeline. Reactor Context is the common mechanism for contextual data, and supported integrations can bridge tracing or security context. Verify context propagation behavior for the exact versions and integrations in use.

### Q187. How does Reactor Context differ from a method parameter?

A method parameter explicitly carries data through a call graph; Reactor Context carries subscriber-associated metadata across a reactive chain. Use Context for cross-cutting metadata such as tracing where supported, but use explicit parameters for business inputs that should remain visible in method contracts.

### Q188. What is the danger of onErrorResume used too broadly?

`onErrorResume` replaces an error signal with another publisher; used too broadly, it can hide a database outage or programming defect behind an apparently successful response. Recover only failures with a domain-valid fallback, preserve the original cause in logs and metrics, and distinguish expected absence (such as an optional record) from infrastructure failure. Avoid returning empty data for critical dependencies unless the API contract explicitly permits it.

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

Actuator exposes operational endpoints and integrates health, metrics, and application information with the running application. Common uses include readiness/liveness probes, metrics scraping, and inspecting selected runtime state. Endpoint availability is different from web exposure: configure which endpoints are enabled and exposed, secure sensitive endpoints, and avoid publishing environment/configuration details to untrusted callers. Adding Actuator does not mean every endpoint should be public.

### Q193. What is the difference between liveness and readiness?

Liveness asks whether the process should be restarted because it is unable to make progress. Readiness asks whether the instance should receive traffic. A transient dependency failure should not automatically trigger restart loops unless the operational policy explicitly requires that behavior.

### Q194. What is Micrometer?

Micrometer provides an instrumentation facade for metrics that can be exported to supported monitoring systems. Prefer counters, timers, gauges, and histograms that answer operational questions. Avoid high-cardinality labels such as user IDs, raw URLs, or request IDs.

### Q195. How should health indicators be designed?

A health check should reflect a meaningful operational decision and be fast, bounded, and safe. Decide whether each dependency belongs in liveness or readiness, set timeouts, and avoid making a health endpoint itself a source of load during an incident. Health status is not a replacement for metrics and alerts.

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


### Q202. What does @Cacheable do?

It checks the configured cache for a key and, on a miss, invokes the method and stores the result according to the cache configuration. The annotation relies on caching infrastructure and commonly proxy interception, so self-invocation may bypass it. Cache keys and value semantics must be designed explicitly.

### Q203. How do @CachePut and @CacheEvict differ?

`@CachePut` always invokes the method and updates the cache with the result; `@CacheEvict` removes entries. Use them when the data-change semantics are clear. Combining cache annotations without understanding evaluation and ordering can produce stale or contradictory results.

### Q204. Why is cache invalidation difficult?

The source of truth can change through multiple paths, and distributed caches introduce propagation delay and concurrent updates. Define which writes invalidate which keys, consider TTL as a safety bound rather than perfect consistency, and test concurrent read/write scenarios.

### Q205. What is a cache stampede?

A large number of callers miss or expire the same key and simultaneously recompute the value, potentially overwhelming the backing service. Depending on the cache, use request coalescing, controlled refresh, jittered TTLs, or stale-while-revalidate patterns. Measure whether complexity is justified.

### Q206. When should Redis be used instead of an in-memory cache?

A local cache is fast and simple but is per-process and can diverge across replicas. Redis can share entries across instances but adds network latency, operational dependency, and its own consistency/failure modes. Select based on freshness, sharing, availability, and capacity requirements.

### Q207. What happens to caching when the backing cache is unavailable?

Behavior depends on the cache provider and application configuration. Decide whether to fail open by invoking the underlying method or fail closed, and ensure the fallback cannot overwhelm the source of truth. Monitor cache errors and hit ratio rather than assuming the cache is always available.

### Q208. How do cache keys affect correctness and security?

Keys must include every input that changes the result, including tenant or authorization scope where relevant. A missing dimension can return one user's data to another. Test key separation across tenants, locale, filters, and permission-sensitive responses.

### Q209. How should cache invalidation interact with transaction commit?

Evicting before a database transaction commits can allow a concurrent reader to reload old data; evicting only after commit has its own failure window. Understand provider transaction awareness and consistency requirements, and consider versioned keys or event-driven invalidation for distributed writers.

### Q210. What is cache penetration?

Repeated requests for absent keys can bypass a cache and repeatedly hit the source. Depending on semantics, cache negative results briefly, validate identifiers, or rate-limit abuse. Do not cache authorization failures in a way that leaks across identities.

### Q211. What is stale-while-revalidate?

It serves a previously cached value while refreshing it asynchronously, trading strict freshness for lower latency and reduced stampedes. Define maximum staleness and do not use it for data where stale reads violate critical invariants.

### Q212. How do you observe cache usefulness?

Measure hit/miss rate, load duration, evictions, failures, entry count or estimated memory, and the load placed on the underlying data source. A high hit ratio alone does not prove the cache is useful: stale values, expensive serialization, large entries, or invalidation bugs can outweigh the benefit. Segment metrics by a bounded cache name or operation, not by arbitrary keys or user IDs.

### Official references
- [Spring cache abstraction](https://docs.spring.io/spring-framework/reference/integration/cache.html)

---

## Asynchronous Processing and Scheduling


### Q213. What does @Async do?

When async execution is enabled and the invocation passes through the relevant proxy, Spring submits the method execution to a configured executor. It does not guarantee more capacity or faster completion; executor sizing, queue bounds, rejection policy, and shutdown behavior determine operational safety.

### Q214. Why should thread pools have bounded queues?

An unbounded queue can accumulate work faster than workers process it, increasing memory use and latency until the service becomes unhealthy. Choose worker count and queue capacity based on workload, downstream limits, and acceptable queueing delay. Define what happens when the system is saturated.

### Q215. How are exceptions from asynchronous methods observed?

Exceptions from fire-and-forget void methods require an appropriate async exception handler because the caller cannot receive them. Methods returning Future or CompletableFuture can surface failures when the result is observed. Ensure failures are logged and measured without duplicating logs at every layer.

### Q216. What is the difference between fixed rate and fixed delay scheduling?

Fixed rate schedules executions according to a target cadence; fixed delay measures the delay after an execution completes before the next run. Actual behavior depends on scheduler configuration and task duration. Long tasks, process pauses, and multiple application instances require deliberate concurrency planning.

### Q217. Why can a scheduled job run more than once in a cluster?

Each application replica may register and execute the same schedule. If only one execution is allowed globally, use a distributed coordination/locking mechanism or a scheduler designed for clustered execution, and make the task idempotent. A local synchronized block only coordinates threads in one process.

### Q218. How should transactions interact with async work?

A transaction bound to the calling thread generally does not automatically propagate into a separately executed async task. Define a transaction boundary inside the asynchronous operation if required, and pass immutable data rather than sharing mutable persistence state across threads.

### Q219. How should cron time zones be handled?

Specify the intended time zone explicitly through the scheduling annotation/configuration where supported, rather than relying on the container default. Test daylight-saving transitions, missed execution after downtime, and behavior when multiple replicas run the same schedule. A cron expression only defines trigger timing; it does not provide distributed locking, durable job state, or exactly-once execution. Use a cluster coordination or durable job mechanism when duplicate runs would be harmful.

### Official references
- [Spring task execution and scheduling](https://docs.spring.io/spring-framework/reference/integration/scheduling.html)

---

## Spring Events and Messaging


### Q220. Are Spring application events synchronous by default?

The standard event multicaster invokes listeners synchronously unless configured otherwise. A slow listener can therefore increase the publisher's latency. Asynchronous event delivery changes error handling and transaction/context assumptions and is not a durable message queue.

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

It does not make a slow or failing service healthy, guarantee session consistency, or remove the need for timeouts and retries. Health checks, balancing algorithm, connection reuse, request distribution, and overload behavior all affect outcomes.

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

Estimate concurrent database demand and compare it with database connection limits and query duration. More connections can increase contention and overwhelm the database. Monitor active, idle, pending, acquisition latency, and query timings, then load-test realistic concurrency.

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

Major upgrades may change Java baselines, managed dependency versions, configuration properties, removed APIs, servlet/Jakarta integration, security defaults, and observability behavior. Read migration guides and release notes, resolve compatibility first, and run tests against production-like infrastructure.

### Q262. How should Spring Cloud compatibility be managed?

Use the official compatibility guidance and corresponding BOM rather than independently pinning arbitrary Cloud components. The current official matrix maps Spring Cloud 2025.1.x to Boot 4.0.x and, starting with 2025.1.2, Boot 4.1.x; Spring Cloud 2025.0.x maps to Boot 3.5.x. Verify the matrix again before upgrading.

### Q263. How do you plan a safe migration?

Inventory runtime and dependencies, identify deprecated APIs, update one major boundary at a time where practical, compile and run focused tests, validate configuration and startup conditions, and exercise security, persistence, and downstream calls. Roll out progressively with metrics and a rollback plan.

### Q264. How should deprecated APIs be handled?

Determine the replacement, migration constraints, and reason for deprecation from the official documentation for the project's exact version. Check whether the API is deprecated for removal, whether behavior changes accompany the replacement, and whether third-party integrations depend on it. Avoid copying legacy examples into new code without labeling the version context. Track deprecations during routine upgrades so a future major-version migration is planned rather than rushed.

### Q265. What is the role of a BOM?

A Bill of Materials centralizes compatible dependency versions so related libraries are aligned. Boot's dependency management reduces manual version selection. Overriding managed dependencies can be necessary, but should be evidence-based and tested for binary compatibility.

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
