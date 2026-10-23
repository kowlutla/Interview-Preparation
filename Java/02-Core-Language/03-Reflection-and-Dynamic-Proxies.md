# Reflection and Dynamic Proxies

**Target level:** Advanced → Senior  
**Scope:** Java 8–21

Reflection inspects classes, members, constructors, annotations, and runtime types. Dynamic proxies can implement interfaces and route method calls through an invocation handler. These capabilities underpin frameworks and tooling, but introduce access, performance, security, and maintainability trade-offs.

## 1. Reflection fundamentals

### Q1. What is reflection in Java? *(Beginner)*

Reflection is the runtime API for inspecting and, where permitted, interacting with classes, fields, methods, constructors, annotations, and other metadata. Class, Method, Field, Constructor, and related types live in java.lang.reflect.

Reflection is useful for frameworks, testing tools, serializers, dependency injection, and diagnostics. It can make code harder to understand and refactor because dependencies are not always visible to the compiler. Prefer ordinary typed calls when runtime discovery is unnecessary.

### Q2. How do you obtain a Class object? *(Beginner)*

Common options include TypeName.class, object.getClass(), and Class.forName with a binary class name. These differ: .class does not initialize the class, getClass requires an instance, and Class.forName(String) initializes the class by default in its traditional form.

When class initialization matters, use the overload that accepts initialize and a ClassLoader. Class loading, linking, and initialization are distinct operations, and reflection APIs can trigger them under defined circumstances.

### Q3. What is the difference between getMethods and getDeclaredMethods? *(Intermediate)*

getMethods returns public methods, including inherited public methods. getDeclaredMethods returns methods declared directly by the represented class or interface, including non-public methods, but not inherited methods. Similar distinctions exist for fields and constructors.

Reflection order is not a stable declaration-order contract. If building metadata or generated output, sort members explicitly when deterministic order matters. Filter bridge and synthetic methods where appropriate.

### Q4. What are bridge and synthetic methods in reflection? *(Advanced)*

The compiler can generate methods that do not appear directly in source, such as bridge methods to preserve overriding after type erasure and synthetic helpers for language features. Method.isBridge() and isSynthetic() expose relevant metadata.

Frameworks that scan methods must decide whether to include or filter them. Blindly treating every reflected method as a source-declared business method can create duplicate handlers or confusing API descriptions.

### Q5. What is the difference between getAnnotation and getDeclaredAnnotation? *(Intermediate)*

getDeclaredAnnotation examines annotations directly present on the element. getAnnotation may also return inherited class annotations when the annotation type uses @Inherited. The exact behavior depends on the element and annotation type.

For method and parameter annotations, do not assume class-style inheritance. Frameworks often need to merge annotations from interfaces, superclass methods, and implementation methods using an explicit precedence policy.

### Q6. How do you invoke a method reflectively? *(Intermediate)*

Obtain a Method from a Class, then invoke it on a compatible receiver for an instance method, passing arguments in the required order. Static methods do not need an instance receiver.

InvocationTargetException wraps an exception thrown by the target method; inspect getCause() to see the underlying failure. Reflection also introduces access and argument-conversion rules, so validate method signatures before invocation when building a reusable framework.

### Q7. Why can reflective invocation throw InvocationTargetException? *(Intermediate)*

Method.invoke wraps exceptions thrown by the invoked target in InvocationTargetException so the caller can distinguish target failure from reflection setup or access failure. The target cause is available through getCause().

When translating the failure, preserve that cause. Do not log only the wrapper message, because it may omit the actual domain exception and useful stack context.

### Q8. What is the difference between Class.forName and a class literal? *(Advanced)*

A class literal such as Service.class obtains the Class object for a known type without initializing it merely because the literal is evaluated. Class.forName(String) traditionally loads and initializes the named class using the caller's defining loader, while overloads can control initialization and loader selection.

Dynamic class names should be validated and constrained; loading arbitrary names from untrusted input can expand attack surface and trigger static initialization side effects. Prefer explicit registries when possible.

## 2. Access control and modules

### Q9. What does setAccessible(true) do, and why can it fail? *(Advanced)*

It requests suppression of Java language access checks for reflective operations, subject to runtime and module-system rules. In modular applications, a package may need to be opened to the caller's module; otherwise access can fail with InaccessibleObjectException or related access errors.

Do not treat reflective access suppression as a universal bypass. Design module boundaries intentionally and avoid depending on private implementation details of third-party libraries, because upgrades can break such code.

### Q10. What is the difference between exports and opens in JPMS? *(Advanced)*

A module exports a package to make its public types accessible to other modules under module access rules. An open package permits deep reflection on its members by specified modules or more broadly, depending on the declaration. Exports and opens solve different problems.

A framework may require opens for reflective access to non-public members. Prefer targeted opens to broad openness, and document why the framework needs that access. Strong encapsulation helps make internal APIs less fragile.

### Q11. What is MethodHandles, and how does it differ from reflection? *(Advanced)*

MethodHandles provide typed, composable references to methods, constructors, and fields with access governed by Lookup capabilities. They are designed as a core dynamic invocation mechanism and can support optimizations and call-site composition.

Reflection exposes metadata and invokes through APIs such as Method.invoke; method handles have their own type signatures and adaptation rules. Neither should be assumed universally faster in every context. Benchmark the actual use case and JDK.

## 3. Dynamic proxies

### Q12. What is a dynamic proxy? *(Intermediate)*

Proxy.newProxyInstance creates a runtime object that implements one or more interfaces and routes interface method invocations to an InvocationHandler. The handler receives the proxy, Method metadata, and arguments.

Dynamic proxies are useful for logging, metrics, access checks, remote stubs, and simple decorators. The standard JDK proxy mechanism is interface-based; class-based proxying requires a different technology or framework strategy.

### Q13. What are the limitations of JDK dynamic proxies? *(Advanced)*

JDK dynamic proxies implement interfaces; they do not create a subclass of an arbitrary concrete class. Calls must be made through proxy interfaces to reach the invocation handler. Object methods such as equals, hashCode, and toString also need a deliberate handler policy.

Default methods and duplicate method signatures across interfaces can require careful dispatch. If the design depends on proxying concrete classes or intercepting calls that bypass the proxy, JDK dynamic proxies may not be sufficient.

### Q14. How should an InvocationHandler handle Object methods? *(Advanced)*

Define explicit behavior for equals, hashCode, and toString rather than assuming they automatically behave like the wrapped target. The proxy handler receives those calls as method invocations under the proxy contract.

For a decorator, decide whether equality is proxy identity or delegate equality. Delegate-based equality can violate symmetry or hash contracts when compared with non-proxy objects. A simple, consistent proxy-identity policy is often safer.

### Q15. How do proxies enable cross-cutting concerns? *(Advanced)*

A proxy can intercept calls to apply logging, metrics, authorization, caching, transaction handling, or retry policies without embedding the concern in every implementation. The concern is centralized at a call boundary.

Interception only applies to calls that pass through the proxy. Self-invocation within the target object often bypasses proxy-based advice, and final/private methods may not be interceptable by class-based proxies. Understand the exact framework mechanism before relying on it for correctness or security.

## 4. Production and performance

### Q16. Is reflection always slow? *(Advanced)*

No universal performance statement is reliable across all JDK versions and usage patterns. Reflection has costs for discovery, access checks, boxing, argument arrays, and exception wrapping, but the JVM can optimize some reflective paths, and method handles have different trade-offs.

Avoid reflection in hot paths without evidence, but do not prematurely reject it for startup-time metadata scanning. Benchmark realistic workloads with the target JDK and measure allocations as well as throughput.

### Q17. What are common reflection security risks? *(Senior)*

Risks include exposing private state, invoking methods that callers should not control, loading attacker-selected classes, bypassing intended validation, and making internal implementation details part of a hidden contract. Module boundaries and runtime configuration also affect access.

Restrict class-name registries, validate permitted operations, avoid reflective access to secrets, and do not treat a UI-hidden method as authorization. Reflection should run behind an explicit policy boundary.

### Q18. How can reflection break refactoring? *(Senior)*

A method or field accessed only by string name may not be found by compile-time reference search, so renaming can silently break runtime behavior. Minification, packaging, module boundaries, and framework scanning can also change what is discovered.

Centralize reflective metadata, use constants or generated metadata where possible, add integration tests for discovery, and prefer annotation processors or typed registries when they meet the requirement.

### Q19. How would you design a plugin loading mechanism? *(Senior)*

Define a stable service interface, discover implementations through an explicit registry or ServiceLoader, validate provider metadata, and isolate failures so one faulty plugin does not necessarily crash the entire process. Establish trust and class-loader boundaries before executing plugin code.

Reflection can instantiate a selected implementation, but arbitrary class loading is not a sandbox. Untrusted plugins require process or container isolation, not merely a class loader. Version the plugin contract and define lifecycle, error handling, and resource ownership.

### Q20. How do you troubleshoot a framework method that is not being intercepted? *(Senior)*

Verify that the object being called is actually the proxy, the call goes through the proxied interface or supported class method, and the relevant advice/annotation is registered. Check self-invocation, final/private methods, visibility, module access, and configuration.

Inspect the runtime class and framework diagnostics, then create a minimal integration test that invokes the method through the same boundary as production. An annotation on source code is not proof that interception occurred.

## Official references

1. [Java SE 21 API — Class](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/lang/Class.html)
2. [Java SE 21 API — java.lang.reflect](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/lang/reflect/package-summary.html)
3. [Java SE 21 API — Proxy](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/lang/reflect/Proxy.html)
4. [Java SE 21 API — InvocationHandler](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/lang/reflect/InvocationHandler.html)
5. [Java SE 21 API — MethodHandles](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/lang/invoke/MethodHandles.html)
6. [JEP 403 — Strongly Encapsulate JDK Internals](https://openjdk.org/jeps/403)

## Chapter checklist

- [ ] Choose the right reflection lookup API and account for inherited/synthetic members.
- [ ] Understand invocation exceptions and module access rules.
- [ ] Explain interface-based dynamic proxy limitations.
- [ ] Avoid assuming proxies intercept self-invocation or every method.
- [ ] Treat reflection as a powerful mechanism, not an authorization boundary.
