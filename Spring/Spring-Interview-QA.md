# Spring Framework + Spring Boot Interview Handbook

A focused interview guide for Java backend engineers. It emphasizes the Spring concepts that matter in real services: dependency management, bean lifecycle, proxies, Boot configuration, HTTP APIs, transactions, security, testing, and production behavior.

**Audience:** Engineers preparing for experienced and senior backend roles.  
**Version scope:** Spring Framework 6/7 concepts and Spring Boot 3/4 generation, with version-sensitive behavior called out. Always check the exact Spring, Spring Boot, Java, and dependency versions used by the target application.  
**Study approach:** Understand the mechanism and trade-offs. These answers are models for learning, not scripts to memorize.

> Question selection is curated from recurring interview themes and official documentation; it is not a guarantee that every company asks these exact questions.

## Contents

1. [Core Spring and dependency injection](#1-core-spring-and-dependency-injection)
2. [Bean lifecycle, configuration, and AOP](#2-bean-lifecycle-configuration-and-aop)
3. [Spring Boot and application startup](#3-spring-boot-and-application-startup)
4. [REST APIs and web request handling](#4-rest-apis-and-web-request-handling)
5. [Transactions and data access](#5-transactions-and-data-access)
6. [Security](#6-security)
7. [Testing, observability, and production readiness](#7-testing-observability-and-production-readiness)
8. [Caching, asynchronous work, and messaging](#8-caching-asynchronous-work-and-messaging)
9. [Reactive programming and version evolution](#9-reactive-programming-and-version-evolution)
10. [Resilience, microservices, and migration](#10-resilience-microservices-and-migration)
11. [Senior-level production troubleshooting](#11-senior-level-production-troubleshooting)
12. [Last-minute revision](#12-last-minute-revision)
13. [Official references](#13-official-references)

---

## 1. Core Spring and dependency injection

### Q1. What problem does the Spring Framework solve?

Spring provides infrastructure for building applications with managed objects and well-defined integration points. Its core container creates and wires objects, while other modules support web applications, transactions, data access, security, testing, messaging, and more.

The important value is not annotation usage by itself. Spring helps separate application policy from object construction and cross-cutting infrastructure. For example, a service can depend on a repository interface without manually constructing the repository, managing its lifecycle, or wiring every dependency. The framework does not eliminate design decisions; it makes dependency boundaries and lifecycle behavior explicit enough to manage consistently.

### Q2. What are IoC and dependency injection, and how are they related?

Inversion of Control means application code delegates some control—such as object creation and dependency wiring—to a framework or container. Dependency Injection is one way to implement that inversion: dependencies are supplied to an object instead of the object locating or constructing them itself.

In Spring, the IoC container creates bean instances, resolves dependencies, applies relevant lifecycle callbacks and post-processors, and makes beans available to the application. Constructor injection is generally the clearest default because it makes required dependencies explicit, supports immutability, and makes unit testing straightforward. DI improves decoupling only when the dependency design itself is sensible.

### Q3. Why is constructor injection usually preferred over field injection?

Constructor injection makes the object's required dependencies visible at construction time. It allows fields to be final, prevents creation of an incompletely configured object through ordinary application code, and makes unit tests easy without reflection or a Spring context. It also exposes overly large constructors as a useful design signal that a class may have too many responsibilities.

Field injection can look concise, but it hides requirements and makes plain unit testing less convenient. Setter injection remains reasonable for genuinely optional or reconfigurable dependencies. The important distinction is between required and optional dependencies, not a blanket claim that every injection style has identical behavior.

### Q4. What is the difference between BeanFactory and ApplicationContext?

`BeanFactory` is the core container abstraction for bean creation and dependency resolution. `ApplicationContext` builds on it and adds common application features such as event publication, message resolution, resource loading, and integration with annotation-based infrastructure.

Most Spring applications use an ApplicationContext directly or through Spring Boot. The distinction matters when discussing the container's responsibilities, but application code should usually depend on application-level services rather than manually look up arbitrary beans. Excessive service-locator usage hides dependencies and makes code harder to test.

### Q5. What is a Spring bean, and how does bean scope affect its lifecycle?

A Spring bean is an object managed by the container. The default singleton scope means one bean instance per bean definition per container—not one instance for the entire JVM and not automatically one instance across every application context. Prototype scope creates a new instance when requested from the container, while web-aware scopes such as request and session are tied to web context.

Singleton scope does not make a bean thread-safe. A singleton service handling concurrent requests should normally keep request-specific state in method-local variables and use thread-safe collaborators. Prototype dependencies injected into a singleton are typically resolved when the singleton is created; a fresh prototype per method call requires a provider, lookup method, or another deliberate mechanism.

### Q6. How does Spring resolve circular dependencies, and should they be used?

A circular dependency exists when bean A requires bean B and bean B requires bean A. Constructor-injection cycles cannot be resolved by simply constructing either object first, so Spring fails to create the graph. Some setter/field cycles have historically been resolvable in particular configurations through early singleton references, but behavior depends on the bean setup and Spring version; Spring Boot applications commonly disallow circular references by default.

The better solution is usually to redesign the dependency graph: extract a third collaborator, move orchestration to a separate service, or replace direct callbacks with an event or interface boundary when that fits the domain. Making a cycle technically resolvable can conceal a responsibility problem.

---

## 2. Bean lifecycle, configuration, and AOP

### Q7. What happens during the Spring bean lifecycle?

At a high level, Spring instantiates the bean, populates its dependencies, invokes aware callbacks where applicable, runs relevant bean post-processors, invokes initialization callbacks, and then exposes the bean for use. When the container shuts down, destruction callbacks can be invoked for beans whose lifecycle is managed by the context.

The exact order has framework-level details, especially around post-processors and proxies. Common initialization mechanisms include `@PostConstruct`, `InitializingBean`, and configured init methods; `@PreDestroy`, `DisposableBean`, and destroy methods provide cleanup hooks. Avoid doing slow or failure-prone remote work casually in constructors or initialization callbacks, because it can make application startup fragile and complicate lifecycle control.

### Q8. What are BeanFactoryPostProcessor and BeanPostProcessor?

A `BeanFactoryPostProcessor` works with bean-definition metadata before ordinary bean instantiation, allowing definitions or configuration metadata to be adjusted. A `BeanPostProcessor` works with bean instances around initialization and is often used by Spring infrastructure to wrap beans in proxies or apply other behavior.

This distinction matters because many framework features are implemented through container extension points. For example, annotation-driven transactions and aspects typically rely on proxying infrastructure rather than magical behavior in the method body. Custom post-processors must be registered carefully: their ordering and dependencies can affect which beans they see and whether they are themselves eligible for processing.

### Q9. What is AOP, and how does Spring implement it?

Aspect-oriented programming separates cross-cutting behavior—such as transaction management, authorization, metrics, or auditing—from business logic. Spring AOP commonly implements this through proxies around Spring-managed objects. A proxy intercepts eligible method calls and applies advice before, after, or around the target invocation.

Spring AOP is not identical to full AspectJ weaving. Proxy-based AOP generally intercepts calls that pass through the proxy, so direct self-invocation within the target object does not normally trigger the proxy advice. This is central to understanding why annotations such as `@Transactional` or `@Cacheable` can appear to be ignored in some call paths.

### Q10. What is the difference between JDK dynamic proxies and CGLIB proxies?

JDK dynamic proxies implement one or more interfaces and route calls through an invocation handler. CGLIB-style class proxies create a subclass and override eligible methods to intercept calls. Spring may choose a proxy strategy based on configuration and the bean's type, and the details can differ by framework generation.

Class-based proxies cannot override final methods, and final classes cannot be subclassed. Private methods are not normally interceptable through subclass overriding either. With interface proxies, callers generally see the exposed interface type. Do not assume every method on an object is interceptable; understand the proxy boundary and the method's visibility/modifiers.

### Q11. How do profiles and externalized configuration work in Spring Boot?

Externalized configuration lets application settings come from property files, environment variables, command-line arguments, config trees, and other supported property sources, with a defined precedence model. Profiles let an application activate environment-specific bean definitions or configuration, for example `dev`, `test`, or `prod`.

Profiles should not become a substitute for secure secret management or create completely different application behavior that is difficult to test. Keep defaults safe, validate configuration at startup, and avoid hardcoding credentials in source control. When a property has the wrong value, inspect the effective property sources and precedence rather than only the YAML file you expect to be active.

### Q12. What is the difference between @Value and @ConfigurationProperties?

`@Value` injects an individual property or expression and is useful for a small number of isolated settings. `@ConfigurationProperties` binds a group of related properties to a typed object and supports structured configuration, metadata, and validation patterns.

For larger application configuration, `@ConfigurationProperties` usually provides a clearer contract and reduces scattered string-based property names. Validate required values and ranges so invalid configuration fails early. Be cautious with secrets in logs or diagnostic output, and remember that exact binding and validation setup depends on the Spring Boot version and configuration style.

---

## 3. Spring Boot and application startup

### Q13. What does Spring Boot add on top of Spring Framework?

Spring Boot reduces the setup needed to build and run Spring applications. It provides opinionated dependency management, auto-configuration, externalized configuration conventions, starter dependencies, embedded-server integration, production-oriented features, and executable packaging support.

Boot does not replace Spring's core container or remove the need to understand it. When an application behaves unexpectedly, the right debugging question is often which auto-configuration or conditional bean definition was applied and why—not merely which annotation to add.

### Q14. How does Spring Boot auto-configuration work?

Auto-configuration contributes bean definitions based on conditions such as available classes, existing beans, properties, and application type. It is designed to provide sensible defaults while backing off when the application supplies its own configuration. The conditions and configuration classes are part of Boot's documented extension model.

If an expected bean is missing or an unexpected one appears, inspect the condition evaluation report, startup logs, active properties, classpath, and user-defined beans. Do not assume auto-configuration always runs or always creates the same beans regardless of dependencies. A starter influences the classpath; it does not by itself guarantee that every related feature is enabled.

### Q15. What does @SpringBootApplication do?

`@SpringBootApplication` is a convenience annotation that combines configuration, component scanning, and auto-configuration enablement. Component scanning finds eligible components within the configured package scope, while auto-configuration contributes conditional bean definitions based on the application environment.

Place the main application class in a package above the application's component packages unless you intentionally configure scanning. If a component is not detected, check package boundaries, filters, bean conditions, and whether the class is actually annotated or registered. Broadly scanning unrelated packages can also create unexpected bean conflicts and startup cost.

### Q16. How would you investigate a slow Spring Boot startup?

Separate time spent before the application context starts from time spent creating beans, initializing the embedded server, validating configuration, running migrations, and performing application-specific startup tasks. Use startup metrics or application startup tracing, logs, and targeted timing around expensive initialization rather than guessing from total process startup time.

Look for blocking network calls in bean initialization, expensive classpath scanning, large component graphs, database migration duration, repeated client construction, and eager work that could be deferred safely. Do not blindly make beans lazy: that can move failures from startup into the first request and create unpredictable latency. Optimize the actual critical path and retain reliable readiness checks.

---

## 4. REST APIs and web request handling

### Q17. What is the difference between @Controller and @RestController?

`@Controller` is commonly used for MVC controllers that return view names or use model attributes. `@RestController` combines controller registration with response-body semantics, so return values are written or converted into HTTP response bodies through message converters.

For a JSON API, `@RestController` is often the natural choice. The annotation does not itself define a good API contract: status codes, validation, error formats, serialization, versioning, and idempotency still need deliberate design.

### Q18. How does a Spring MVC request reach a controller method?

In a typical servlet-based Spring MVC application, the request enters the servlet container and is dispatched through `DispatcherServlet`. Handler mappings identify a suitable controller method; argument resolvers bind path variables, query parameters, headers, and request bodies; validation and message conversion may occur; then the handler executes and the result is converted to a response or resolved as a view.

Understanding this path helps diagnose whether a failure is caused by routing, binding, validation, authorization, controller logic, exception handling, or serialization. A controller method may never be called if the request fails earlier in the filter chain or handler mapping.

### Q19. How should request validation and exception handling be designed?

Use Bean Validation constraints for input invariants that can be checked at the request boundary, and translate failures into a stable error response with an appropriate HTTP status. A global exception handler such as `@RestControllerAdvice` can centralize error mapping and keep controllers focused on the use case.

Do not expose stack traces, database details, tokens, or internal class names to clients. Keep errors useful and consistent, include a correlation/request identifier where appropriate, and distinguish invalid input from missing resources, conflicts, authentication failures, authorization failures, and unexpected server errors. Validation at the API boundary does not replace domain-level invariants or database constraints.

### Q20. How do you design an idempotent REST endpoint?

An operation is idempotent when repeating the same request has the same intended effect as performing it once, even if the response may differ. HTTP defines method semantics, but application-level behavior still matters: a retry of a create or payment request must not accidentally create duplicate effects.

For operations that may be retried after timeouts, use a documented idempotency key or business identifier, persist the key/result with appropriate uniqueness guarantees, and define how repeated requests are handled. Make the operation's state transition atomic with its deduplication record where possible. A timeout does not prove the server failed before committing the operation.

### Q21. How should pagination and filtering be implemented in a REST API?

Define stable ordering, validated page/size or cursor parameters, reasonable maximum limits, and explicit response metadata. Offset pagination is straightforward but can become expensive at large offsets and may shift under concurrent inserts/deletes. Cursor/keyset pagination can perform better for deep traversal when a stable indexed ordering key is available.

Do not expose arbitrary database sort fields without validation. Apply filtering and sorting at the database where appropriate, return only needed columns for large datasets, and ensure the ordering has a tie-breaker so pages are deterministic. API pagination is a contract: changing default page size, ordering, or cursor semantics can affect clients.

---

## 5. Transactions and data access

### Q22. How does @Transactional work in Spring?

In the common proxy-based configuration, Spring creates an interceptor around eligible method calls. The interceptor starts or joins a transaction according to the transaction manager and annotation settings, invokes the target method, and commits or rolls back based on the outcome and rollback rules.

The annotation alone does not guarantee a transaction in every call path. Self-invocation usually bypasses the proxy; private/final method constraints depend on the proxy strategy; the correct transaction manager must be configured; and the method must be invoked through the Spring-managed proxy. A transaction also does not automatically span arbitrary remote HTTP calls or messaging systems.

### Q23. Why might @Transactional appear not to work?

Common causes include calling the annotated method from another method on the same object, creating the object with `new` instead of letting Spring manage it, using a method or class that the chosen proxy cannot intercept, missing or incorrect transaction-manager configuration, or misunderstanding the rollback rules.

Also check whether the exception is caught and suppressed inside the transactional method, whether it is a checked exception with default rollback behavior that differs from the expectation, and whether the operation is actually executed in the same transaction. Verify the call path and transaction logs rather than adding annotations at random.

### Q24. What are transaction propagation and isolation?

Propagation determines how a method participates in an existing transaction. For example, `REQUIRED` joins an existing transaction or creates one; `REQUIRES_NEW` suspends the current transaction and creates a separate one when supported. Other propagation modes express different rules for requiring or avoiding a transaction.

Isolation governs what concurrent transactions may observe and is ultimately constrained by the transaction manager and database. Higher isolation can prevent more anomalies but may reduce concurrency or introduce locking costs. Do not assume the annotation's isolation setting has identical semantics across all databases, and remember that self-invocation can prevent the annotated propagation behavior from being applied.

### Q25. When should a transaction roll back, and what are the default rollback rules?

Spring's traditional declarative transaction defaults roll back for unchecked exceptions (`RuntimeException`) and `Error`, not for every checked exception. Rollback rules can be configured explicitly. Exact behavior should be checked against the Spring version and transaction configuration in use.

A caught exception that never escapes the transactional boundary may not trigger rollback automatically. If code marks the transaction rollback-only, a later commit attempt can fail with an unexpected rollback. Define rollback behavior around the business operation's failure semantics; avoid broad catch-and-continue logic that leaves partially updated state.

### Q26. Why should long remote calls usually be kept outside database transactions?

A database transaction holds resources and may retain locks or connections while the application waits on an external service. Slow or unavailable downstream calls can therefore increase lock duration, exhaust the connection pool, and amplify an outage. Most local database transactions cannot atomically commit with an unrelated remote HTTP service.

Prefer short transactions around the database state transition, with explicit workflow design for remote effects. Depending on the use case, an outbox, idempotent consumer, saga, or compensating action may be appropriate. Do not assume annotating a method with `@Transactional` makes a multi-service operation atomic.

### Q27. What is the difference between Spring Data JPA and JDBC access?

Spring Data JPA provides repository abstractions on top of JPA, including derived queries, projections, pagination, and integration with entity persistence. JDBC provides more direct SQL and row-mapping control. JPA is productive for entity-oriented operations, but it can hide query count, fetch behavior, flush timing, and object-graph costs.

Use the abstraction that matches the operation. For complex reporting, bulk updates, database-specific features, or performance-sensitive queries, JPQL/native SQL/JDBC may be more appropriate. Inspect generated SQL and query plans rather than assuming a repository method is efficient because it is concise.

---

## 6. Security

### Q28. How does Spring Security process a request?

In a servlet application, Spring Security commonly applies a chain of servlet filters before the controller. Filters handle concerns such as security context setup, authentication, authorization, exception translation, and exploit protection depending on configuration. Authentication establishes who the caller is; authorization decides whether that caller may perform an action.

A request that never reaches the controller may have been rejected by the security filter chain. Diagnose the configured filter chain, authentication mechanism, authority mapping, request matchers, CSRF policy, and security context propagation. Avoid treating authentication and authorization as interchangeable concepts.

### Q29. How should JWT authentication be explained, and what are its limitations?

A JWT is a token format that can carry claims and is commonly signed so a recipient can verify integrity and issuer-related trust. A signed JWT is not automatically encrypted; its payload may be readable by anyone who obtains it. The application must validate signature, accepted algorithm, issuer, audience where applicable, expiry, and other required claims.

JWT does not automatically solve logout, revocation, key rotation, or permission changes. Token lifetimes, refresh strategy, storage, replay risk, and secret/key handling all matter. Prefer established framework support and well-reviewed libraries over hand-written token parsing or accepting whatever algorithm appears in the token header.

### Q30. What is CSRF, and when is CSRF protection needed?

Cross-Site Request Forgery abuses a browser's tendency to attach ambient credentials, such as session cookies, to a request initiated by another site. If an application relies on cookies for authentication, state-changing requests may need CSRF protection because the browser can attach the session cookie without the user intending the action.

Stateless APIs using credentials explicitly attached by clients, such as a bearer token in an Authorization header not automatically attached by the browser, have a different threat profile. Do not disable CSRF simply because an endpoint returns JSON. Evaluate how credentials are stored and sent, whether browsers attach them automatically, and the application's actual threat model.

### Q31. How should method-level authorization be used?

Method-level authorization applies policy near the operation that requires it, which can protect business actions even when they are invoked through different routes. It is useful for expressing rules such as requiring a particular authority or verifying that a caller may access a resource.

Authorization must be enabled and the method must pass through the relevant security infrastructure. A URL rule alone may not protect internal invocation paths, while method-level annotations do not replace input validation or resource ownership checks. Test both allowed and denied cases, including object-level authorization and attempts to access another user's data.

---

## 7. Testing, observability, and production readiness

### Q32. What is the difference between a unit test and a Spring integration test?

A unit test checks a class or small unit in isolation, often using plain construction and mocks for collaborators. It is fast and helps verify business logic without starting the container. A Spring integration test loads some or all of the application context and verifies framework wiring, configuration, persistence integration, or request behavior.

Use the smallest test scope that gives confidence. Loading the full context for every service method makes tests slower and can hide unclear boundaries. Conversely, a pure unit test cannot prove that component scanning, transaction proxies, security filters, or serialization configuration are correct. Test the behavior at the layer where the risk exists.

### Q33. What are Spring Boot test slices, and when are they useful?

Test slices load a focused subset of the application context for a particular layer, such as MVC or JPA, instead of bootstrapping every component. They provide a useful middle ground between plain unit tests and full application integration tests.

A slice test still needs deliberate collaborators and configuration; it is not a substitute for testing critical end-to-end wiring. Keep a small number of full integration tests for important workflows and use unit/slice tests for fast feedback. The available annotations and mock-bean APIs can vary across Spring Boot generations, so check the matching reference documentation.

### Q34. What are Actuator and Micrometer used for?

Spring Boot Actuator exposes operational endpoints and integrations for application health, metrics, environment information, and other management capabilities. Micrometer provides a vendor-neutral instrumentation facade that can publish metrics to supported monitoring systems.

These endpoints are operational interfaces and must be secured appropriately. Do not expose sensitive environment/configuration details publicly. Health checks should distinguish whether the process is alive from whether it is ready to serve traffic, and a health indicator should not cause cascading failures by performing an unbounded expensive dependency check on every probe.

### Q35. How do you diagnose high latency in a Spring Boot application?

Start from user-visible symptoms and correlate request latency with thread usage, connection pools, database queries, remote calls, GC behavior, and queueing. Distributed traces can show where time is spent across service boundaries; metrics show trends and saturation; logs provide event context; thread dumps and JFR help investigate JVM-side behavior.

Check whether a pool is exhausted, requests are waiting for a downstream dependency, a transaction holds locks too long, or synchronous work blocks a request thread. Apply a mitigation that matches the bottleneck—such as limiting concurrency, tightening timeouts, fixing a query, or restoring a dependency—and confirm that it improves the service-level objective rather than just moving the queue elsewhere.

---

## 8. Caching, asynchronous work, and messaging

### Q36. How does Spring caching work, and what can go wrong?

Spring's cache abstraction allows methods to use annotations such as `@Cacheable`, `@CachePut`, and `@CacheEvict` with a configured cache provider. The abstraction separates application code from the cache implementation, but actual eviction, expiry, serialization, consistency, and distributed behavior depend on the provider and configuration.

Proxy-based caching has a self-invocation limitation similar to transactions. Stale data, incorrect keys, missing tenant/user dimensions, cache stampedes, and unbounded cache growth are common risks. Define invalidation and expiry behavior explicitly, and never treat a cache as the source of truth unless the architecture truly makes it authoritative.

### Q37. What do @Async and @Scheduled do, and what should be considered in production?

`@Async` delegates eligible method calls to an executor, while `@Scheduled` triggers methods according to configured scheduling rules. Both depend on Spring configuration and proxy/lifecycle behavior; self-invocation can bypass asynchronous interception. The executor and scheduler need deliberate thread, queue, error, and shutdown policies.

Do not assume asynchronous execution means reliable background processing. In-memory scheduled or async work may be lost on process restart and may run more than once across multiple application instances. For durable jobs, consider a persistent job system or messaging. Make tasks idempotent where retries are possible, and propagate tracing/security context only through supported, deliberate mechanisms.

### Q38. How should application events and message-driven work be used?

Spring application events are useful for decoupling components within one application process. By default, publishing an event does not create a durable distributed message, and an in-process listener is not automatically an independent transaction or retryable job. Async event listeners add execution-policy and error-handling concerns.

For cross-service communication or durable processing, use a message broker and define delivery semantics, retries, idempotency, ordering, dead-letter handling, and observability. If a database update and event publication must stay consistent, consider a transactional outbox or another explicit coordination design; a simple “save then publish” sequence can lose events if the process fails between the two operations.

---

## 9. Reactive programming and version evolution

### Q39. What is the difference between Spring MVC and WebFlux?

Spring MVC is based on the servlet request-processing model and is a natural fit for many traditional synchronous web applications. WebFlux provides a reactive programming model based on non-blocking I/O and Reactive Streams, using types such as `Mono` and `Flux` in Reactor-based applications.

Reactive code can improve resource efficiency for workloads with many concurrent I/O operations when the entire path is designed to avoid blocking. It does not automatically make every application faster. Blocking JDBC calls inside reactive pipelines can consume event-loop resources and undermine the model. Choose based on workload, dependencies, team familiarity, and measured needs rather than assuming reactive is universally superior.

### Q40. What should be considered when upgrading Spring Boot or Spring Framework?

First identify the exact current and target versions, Java baseline, managed dependency versions, removed/deprecated APIs, security defaults, configuration changes, and any changes in Spring Data, Hibernate, servlet containers, test libraries, or observability integrations. Spring Boot's dependency management is valuable, so avoid overriding versions without understanding compatibility constraints.

Use the official migration guides and release notes, upgrade in controlled steps where practical, run focused integration and security tests, and validate startup, health checks, database migrations, metrics, and deployment behavior. A successful compilation does not prove behavioral compatibility. Test the real packaging/runtime combination, including container images and environment configuration.

---

## 10. Resilience, microservices, and migration

### Q41. How should resilience be designed for calls to downstream services?

A resilient client uses explicit connection/read/response deadlines, bounded retries for transient failures, exponential backoff with jitter, and concurrency limits appropriate to the dependency. A circuit breaker can stop repeatedly calling a dependency that is failing, while bulkheads isolate resource usage so one slow integration does not consume every request thread or connection.

These patterns solve different problems and should not be stacked without a clear failure model. Retries can amplify load during an outage and can duplicate side effects unless the operation is idempotent. A timeout does not prove the remote operation failed. Define an overall deadline, observe retry/circuit-breaker metrics, and return a deliberate fallback or failure response when continuing is unsafe.

### Q42. What does Spring Cloud add to a microservice architecture?

Spring Cloud provides integrations and abstractions for common distributed-system concerns, depending on the selected projects: centralized/external configuration, service discovery, client-side load balancing, gateway routing, resilience integrations, and messaging-related patterns. It is an ecosystem of modules rather than one mandatory runtime layer; not every application needs every component.

These tools do not remove distributed-systems failure modes. Service discovery can return stale endpoints, configuration changes need rollout and validation, gateways become critical infrastructure, and retries can multiply load. Choose components based on deployment architecture and operational needs, and make timeouts, authentication, tracing, version compatibility, and failure handling explicit.

### Q43. What should be checked when upgrading from Spring Boot 2 to 3 or newer generations?

Spring Boot 3 moved to the Jakarta EE namespace for APIs such as Servlet and Persistence, requiring imports and dependencies to migrate from `javax.*` to the relevant `jakarta.*` packages. It also raised the Java baseline and changed dependency generations. Newer Boot generations introduce further requirements and compatibility changes, so the exact target release's migration guide is essential.

Review the Java version, dependency management, Spring Security configuration, Hibernate/Jakarta Persistence generation, servlet container, validation APIs, test libraries, observability, and build plugins. Search for removed/deprecated APIs, run integration tests against the actual database and deployment environment, and verify startup/readiness/metrics after packaging. Compilation alone is not enough to prove runtime compatibility.

---

## 11. Senior-level production troubleshooting

### Q44. When should you use `@Bean` instead of `@Component`, and what does `@Configuration` change?

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

### Q45. How do you resolve ambiguous bean injection, and when are conditional beans useful?

If multiple beans match a required type, Spring needs an unambiguous candidate. Use `@Qualifier` when the injection point specifically needs a named role or implementation; use `@Primary` when one candidate should be the default for ordinary type-based injection. Qualifiers are not merely a fallback name lookup—they express which candidate fits that dependency. If the implementations represent different business behaviors, consider whether a strategy map or explicit selection is clearer than making one globally primary.

Conditional registration is useful when a bean should exist only if a class is on the classpath, a property is enabled, or the application has not supplied its own implementation. Spring Boot auto-configuration relies heavily on these conditions. When a bean is missing or unexpectedly duplicated, inspect the condition evaluation report, active profiles, properties, classpath, and user configuration before adding another bean. Avoid conditions that silently disable a critical capability without a clear startup failure or diagnostic message.

### Q46. What is the difference between a servlet filter, a Spring MVC interceptor, and controller advice?

A servlet filter runs in the servlet chain before or after the servlet, so it can act before Spring MVC chooses a controller. Filters are suitable for cross-cutting servlet concerns such as request wrapping, low-level logging, and integrations that must apply beyond controller methods. Spring Security itself integrates through a filter chain.

A Spring MVC `HandlerInterceptor` operates around handler execution after Spring MVC has mapped a request to a handler. It can inspect handler-related information, but it is not a replacement for security controls in the filter chain. `@ControllerAdvice` is different again: it centralizes controller exception handling, binding behavior, or model-related concerns. Choose based on the lifecycle stage you need. If a controller is never reached, inspect filters and handler mapping first; controller advice cannot handle every failure that occurs before MVC invokes a controller.

### Q47. How would you diagnose a request that is unexpectedly allowed or rejected by Spring Security?

Start by confirming which `SecurityFilterChain` matches the request. With multiple chains, ordering and request matchers determine which configuration applies; rules in a chain that never matches cannot protect the request. Then inspect authentication, the resulting principal and authorities, URL authorization rules, method-level security, and any custom filters or authority converters. Enable appropriate security debug logging temporarily in a safe environment, and reproduce with the exact HTTP method, path, headers, cookies, and token.

Also distinguish CORS from authorization. CORS is a browser-enforced cross-origin policy, not an authentication mechanism; a failed browser preflight can prevent the browser from sending the actual request, while a non-browser client may behave differently. CSRF is a separate protection against forged requests that exploit automatically attached browser credentials. Do not disable CSRF or broadly allow origins just to make a failing request pass. Spring Security's [servlet configuration](https://docs.spring.io/spring-security/reference/servlet/configuration/java.html), [CORS guidance](https://docs.spring.io/spring-security/reference/servlet/integrations/cors.html), and [CSRF reference](https://docs.spring.io/spring-security/reference/servlet/exploits/csrf.html) should be checked for the version in use.

### Q48. A `@Cacheable` method always executes, or an `@Async` method runs on the caller thread. How would you investigate?

First verify that the method is called on a Spring-managed bean through the proxy. Self-invocation, manual construction with `new`, or a call path that bypasses the proxy can prevent both cache and async advice from running. Then check whether the relevant annotation processing is enabled, whether the method is eligible for interception, and whether the selected cache/executor configuration is actually active.

For caching, inspect the cache name, generated key, arguments, condition/unless expressions, provider statistics, and eviction behavior. A key that includes a changing argument—or a condition that skips caching—can make every call a miss. For async work, inspect the configured executor, thread names, task rejection and queue metrics, and exception handling. If a custom executor has a direct or caller-runs policy, work may execute on the caller under some conditions. Confirm behavior with a focused test that asserts the cache interaction or records the executing thread; do not infer success from the annotation being present.

### Q49. A connection pool is exhausted under load. What evidence would you collect before changing pool size?

Treat pool exhaustion as a symptom, not proof that the pool is simply too small. Compare active, idle, and pending connection metrics with request concurrency and latency. Check connection-acquisition timeouts, slow SQL, lock waits, transaction duration, leaked connections, and whether code performs remote calls while holding a database transaction open. A thread dump can reveal many request threads waiting for a connection; database-side session and query diagnostics help distinguish slow queries from application-side connection retention.

Increasing the pool can help if the database has capacity and the workload legitimately needs more concurrent connections, but it can also overload the database and worsen latency. Bound request concurrency, set appropriate acquisition/query/transaction timeouts, shorten transaction scope, and fix slow queries or leaks first. Validate the change under representative load and observe both application pool metrics and database saturation. Pool sizing is a coordinated application-and-database decision, not an isolated tuning knob.

### Q50. A configuration property has the wrong value only after deployment. How do you trace it?

Do not assume the YAML file is the effective source. Identify the exact property key and the value expected by the running version, then inspect active profiles and the ordered property sources that can override it—such as command-line arguments, environment variables, external configuration, and packaged files. Check spelling, relaxed binding, profile activation, deployment manifests, and whether the application is reading a similarly named property from a different configuration namespace.

For grouped application settings, prefer validated `@ConfigurationProperties` over scattered string-based injections. Add startup validation for required values and invalid ranges, but never dump secrets or the full environment into logs. A safe diagnostic should reveal the source and effective non-sensitive value, not expose credentials. Reproduce with the same packaged artifact and deployment configuration; “works locally” often means the local property sources differ, not that Spring resolves the same configuration differently.

### Q51. Why might a Spring-managed bean work while a manually instantiated object does not?

Spring-managed beans can receive dependency injection, lifecycle callbacks, post-processing, and proxy wrappers for features such as transactions, caching, method security, and asynchronous execution. An object created with `new` receives none of that container behavior unless your code supplies it explicitly. Its annotations are metadata; they do not independently activate framework infrastructure on an arbitrary Java object.

If a behavior such as rollback, caching, or authorization disappears, check how the object was created and whether calls pass through the expected proxy. Prefer injecting the collaborator into the class that needs it. If manual construction is intentional—for example, a simple value object or a unit-tested domain class—keep it free of infrastructure behavior that requires container management. This distinction is often the quickest explanation for “the annotation is present but nothing happens.”

---

## 12. Last-minute revision

Prioritize these questions when time is limited:

1. IoC, dependency injection, and constructor injection (Q2–Q3)
2. Bean scopes and singleton thread safety (Q5)
3. Bean lifecycle and post-processors (Q7–Q8)
4. AOP proxies, JDK vs CGLIB, and self-invocation (Q9–Q10)
5. Boot auto-configuration and startup diagnosis (Q13–Q16)
6. MVC request flow, validation, and error handling (Q17–Q19)
7. `@Transactional`, propagation, rollback rules, and self-invocation (Q22–Q26)
8. Authentication vs authorization, JWT, and CSRF (Q28–Q31)
9. Unit tests vs integration tests and test slices (Q32–Q33)
10. Actuator, metrics, traces, and production latency investigation (Q34–Q35)
11. Caching proxy behavior, async execution, and durable messaging (Q36–Q38)
12. MVC vs WebFlux and upgrade discipline (Q39–Q40)

For each answer, be ready to explain the mechanism, one common failure mode, and how you would investigate it in a real application.

---

## 13. Official references

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

## Editorial and validation notes

- The guide prioritizes a focused set of recurring questions over hundreds of repetitive variants. It contains 51 numbered questions, including targeted senior-level production troubleshooting scenarios. It contains 51 numbered questions, including targeted senior-level production troubleshooting scenarios.
- Answers explain mechanisms and practical consequences rather than giving only definitions.
- Small examples are used only where they clarify behavior; full sample applications are intentionally excluded.
- Proxy semantics, transaction behavior, and security configuration must be verified against the exact framework generation.
- This guide is a curated primary study path; it is not a claim that every file in the previous chapter collection has already been migrated or removed.
