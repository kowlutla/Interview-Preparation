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
