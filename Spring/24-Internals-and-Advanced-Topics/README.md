# Spring Internals and Advanced Topics

## Q&A

### Q1. What is a BeanDefinition?

A BeanDefinition is container metadata describing how a bean should be created and configured, including its class or factory method, scope, dependencies, and other settings. Component scanning and configuration processing register definitions before regular singleton instantiation. Inspecting definitions helps diagnose missing or unexpectedly configured beans.

### Q2. Why does Spring use bean post-processors?

Bean post-processors provide lifecycle extension points for inspecting or wrapping bean instances. Auto-proxy creators use this infrastructure to apply aspects such as transactions or caching. Infrastructure ordering matters; a bean created too early may miss processing that would normally apply.

### Q3. Why might an MVC interceptor not run even though a servlet filter logs the request?

A filter can run before MVC selects a handler, while an interceptor runs only after a handler has been mapped. The request may be short-circuited by a security filter, fail handler mapping, target a static/resource path, or use a different dispatcher path. Inspect filter-chain order, the selected HandlerMapping, and response status before assuming the interceptor registration is broken.

### Q4. Why can final methods complicate proxying?

Subclass-based proxies cannot override final methods, and final classes cannot be subclass-proxied. Interface-based proxies have different constraints but only intercept calls made through the proxy contract. Verify the proxy type and method visibility when advice unexpectedly does not execute.

### Q5. What is an ApplicationContext hierarchy?

A context can have a parent context, allowing some bean lookups to fall back to parent definitions while child contexts define their own beans. This is useful in certain web or modular arrangements but can complicate visibility and duplicate definitions. Use it only when the isolation and inheritance model is understood.

### Q6. How should custom composed annotations be designed?

Composed annotations combine existing annotations and may provide aliases or defaults to express a domain convention. Keep their semantics narrow, document meta-annotation behavior, and test that component scanning or interception recognizes them. Avoid building opaque annotation frameworks that obscure the actual configuration.

### Q7. How do classpath problems surface in Spring applications?

Missing or incompatible dependencies can cause class-not-found, method-not-found, linkage, or auto-configuration condition failures. Inspect the resolved dependency graph, managed BOM versions, and the first meaningful cause in the stack trace. Avoid fixing a binary incompatibility by randomly upgrading one transitive library.

## Official references

- [Spring Core container](https://docs.spring.io/spring-framework/reference/core/beans.html)
- [Spring AOP proxying](https://docs.spring.io/spring-framework/reference/core/aop/proxying.html)
