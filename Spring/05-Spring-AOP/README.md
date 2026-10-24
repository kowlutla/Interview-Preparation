# Spring AOP and Proxies

## Q&A

### Q1. What is Aspect-Oriented Programming?

AOP modularizes cross-cutting behavior such as transaction boundaries, authorization checks, metrics, and auditing that would otherwise be repeated across many methods. Spring AOP is primarily proxy-based and intercepts method calls that pass through the proxy; it is not a general bytecode-weaving system by default.

### Q2. What are advice, pointcut, join point, and aspect?

An aspect groups cross-cutting behavior. Advice is the action executed at a selected point. A pointcut selects join points, which in Spring AOP are method-execution join points on Spring-managed objects. Understanding the selection expression is essential because an overly broad pointcut can affect unrelated code.

### Q3. How do JDK dynamic proxies and CGLIB proxies differ?

JDK proxies implement interfaces; CGLIB-based proxies create a subclass of the target class. Proxy selection depends on configuration and framework context. Class-based proxies cannot override final methods, and final classes cannot be subclass-proxied. Design for interception at public service boundaries rather than assuming every method can be intercepted.

### Q4. What is the self-invocation problem?

When a method on a bean calls another method on the same instance with this.method(), the call does not pass through the Spring proxy. Advice such as transactions, caching, or async execution may therefore not run for the internal call. Refactor the intercepted operation to another bean or redesign the boundary; avoid brittle self-proxy workarounds where possible.

### Q5. When should @Around advice be used?

Use around advice when behavior must run before and after the invocation and may need to control proceeding to the target. Always call ProceedingJoinPoint.proceed() appropriately, preserve return values, and avoid swallowing exceptions accidentally. Use narrower advice types when they express the need more clearly.

### Q6. How do you order multiple aspects?

Define ordering explicitly when the relative sequence matters, using supported ordering mechanisms such as @Order. Ordering affects nested interception and exception flow, so test the observable behavior rather than assuming the order from class names or registration order.

### Q7. Is Spring AOP appropriate for every cross-cutting concern?

No. Use AOP for coherent cross-cutting policies at stable method boundaries. Prefer explicit code for business-critical flow that benefits from local readability, and use servlet filters or MVC interceptors when the concern belongs to the HTTP pipeline rather than service method execution.

## Official references

- [Spring AOP reference](https://docs.spring.io/spring-framework/reference/core/aop.html)
- [Proxying mechanisms](https://docs.spring.io/spring-framework/reference/core/aop/proxying.html)
