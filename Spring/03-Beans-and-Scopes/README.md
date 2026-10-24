# Spring Beans and Scopes

## Q&A

### Q1. What does singleton scope mean in Spring?

Singleton means one bean instance per bean definition per Spring IoC container. It is not the same as enforcing the GoF Singleton pattern across the entire JVM. Two application contexts can each create their own instance.

### Q2. Does singleton scope make a bean thread-safe?

No. A singleton service can be called concurrently by many request threads. Prefer stateless services and keep request-specific data in method-local variables or explicit request-scoped collaborators. Mutable shared fields require a sound concurrency design.

### Q3. What is prototype scope?

Prototype scope asks Spring to create a new instance each time that bean is requested from the container. Spring manages construction and initialization, but it does not generally track prototype instances through destruction callbacks after handing them to the caller. The caller must release resources when necessary.

### Q4. What happens when a prototype bean is injected into a singleton?

The prototype is normally resolved when the singleton is created, so the singleton retains that one instance. To obtain a fresh instance on demand, use ObjectProvider, a suitable provider/factory, or a scoped proxy when appropriate.

### Q5. When are request and session scopes available?

They are web-aware scopes tied to an active web request or session and require the relevant web application context. Accessing them outside the required context can fail. They should not be confused with thread-local state that remains valid across asynchronous execution.

### Q6. What are @PostConstruct and @PreDestroy used for?

They mark initialization and destruction callbacks on managed beans, subject to lifecycle and scope semantics. Initialization callbacks should not assume every other bean is fully operational in every ordering scenario, and destruction callbacks are not guaranteed for every abrupt process termination.

### Q7. What is lazy initialization, and what is its trade-off?

Lazy initialization delays creation until a bean is needed. It can reduce startup time and work, but can defer configuration errors and latency to the first request that triggers the bean. Use it based on startup measurements and readiness semantics rather than as a default cure.

### Q8. What is a scoped proxy?

A scoped proxy provides a stable proxy object that resolves the underlying target according to its scope, allowing a bean with a shorter scope to be injected into a longer-lived bean. It is useful for request-scoped dependencies injected into singleton services, but callers must understand proxy behavior and thread/request boundaries.

### Q9. What are common bean lifecycle pitfalls?

Common issues include performing slow network calls during initialization, triggering premature bean creation from post-processors, relying on destruction callbacks during forced termination, and assuming proxies are present inside initialization callbacks. Keep startup work bounded and make lifecycle assumptions explicit.

### Q10. Can Spring override a bean definition with another bean of the same name?

Behavior depends on configuration and Boot settings; bean-definition overriding is not a safe way to resolve accidental conflicts. Prefer unique, meaningful names and explicit conditions or qualifiers. Verify the selected Boot version's defaults when debugging duplicate-bean registration.

## Official references

- [Bean scopes](https://docs.spring.io/spring-framework/reference/core/beans/factory-scopes.html)
- [Bean lifecycle](https://docs.spring.io/spring-framework/reference/core/beans/factory-nature.html)
