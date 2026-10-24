# IoC Container and Dependency Injection

## Q&A

### Q1. How does Spring resolve a constructor dependency?

Spring inspects registered bean definitions and the target constructor's parameter types, then resolves candidates using type, qualifiers, primary designation, and other metadata. If a required dependency has no candidate or multiple unresolved candidates, context creation normally fails with a dependency-resolution exception.

### Q2. What is the difference between @Component, @Service, @Repository, and @Controller?

All are stereotype annotations used to identify components for scanning. @Service communicates service-layer intent; @Repository communicates persistence-layer intent and participates in persistence exception translation when the relevant infrastructure applies; @Controller identifies a web controller. @Component is the generic stereotype. Their semantic roles improve readability and framework integration, not business logic by themselves.

### Q3. What is the difference between @Bean and @Component?

@Component is placed on a class discovered by component scanning. @Bean is placed on a method in a configuration class, and the method's return value is registered as a bean. @Bean is particularly useful for configuring third-party classes that cannot be annotated directly or for explicit factory logic.

### Q4. What do @Configuration and @ComponentScan do?

@Configuration marks a class as a source of bean definitions, often declared with @Bean methods. @ComponentScan instructs Spring to discover eligible stereotype-annotated classes in selected packages. In modern Boot applications, @SpringBootApplication includes component scanning rooted at the application's package, so package placement matters.

### Q5. How do @Primary and @Qualifier differ?

@Primary gives one candidate preference when multiple beans match a single-valued injection point. @Qualifier narrows the candidate set using qualifier metadata. A qualifier can express a semantic role and is generally clearer when a consumer specifically needs a named kind of implementation.

### Q6. Is field injection always invalid?

No, but constructor injection is usually preferable for mandatory dependencies because it makes requirements visible and enables final fields. Field injection can be convenient in some framework-managed test fixtures, but it complicates plain unit tests and can hide dependency growth.

### Q7. What is a circular dependency?

A circular dependency exists when bean A requires bean B and bean B requires bean A, directly or through a chain. Constructor-injection cycles cannot be resolved by constructing either object first and normally fail during context creation. Setter/field cycles may be resolvable in some singleton scenarios, but relying on that is fragile; redesigning responsibilities is usually better.

### Q8. What is the difference between BeanFactoryPostProcessor and BeanPostProcessor?

A BeanFactoryPostProcessor works with bean definitions and container metadata before ordinary bean instantiation is complete. A BeanPostProcessor can inspect or wrap bean instances before and after initialization; infrastructure such as proxy creation uses post-processing. Confusing these phases can lead to early-instantiation bugs.

### Q9. When would you use ObjectProvider<T>?

Use ObjectProvider<T> when a dependency is intentionally optional, lazily resolved, or needs candidate iteration at runtime. It exposes operations such as getIfAvailable() and stream access without requiring immediate resolution at the injection point. Do not use it merely to hide an unclear dependency design.

### Q10. What does @Lazy do?

@Lazy can defer bean initialization or dependency resolution, depending on where it is applied. It can reduce startup work when a bean is not always needed, but it moves failures to first use and can make startup health look better than actual request readiness. It is not a general fix for circular dependencies.

### Q11. How can you register a bean conditionally?

Use an appropriate condition such as Boot's @ConditionalOnClass, @ConditionalOnProperty, or @ConditionalOnMissingBean for Boot auto-configuration, or profile/conditional configuration suited to the application. Conditions should be explicit and testable. The exact condition annotations available depend on the framework and Boot modules in use.

### Q12. Why should dependencies generally point toward abstractions?

Depending on an abstraction reduces the consumer's knowledge of a concrete implementation and can make replacement or testing easier. The abstraction must still represent a stable, coherent contract; adding interfaces for every class without a meaningful boundary can create ceremony rather than decoupling.

### Q13. How do you debug NoSuchBeanDefinitionException?

Check that the implementation is on the runtime classpath, component scanning covers its package, conditions and profiles permit registration, and the required bean type is assignable. Inspect the condition evaluation report in Boot applications and verify that configuration is loaded in the failing test or runtime context.

### Q14. How do you debug NoUniqueBeanDefinitionException?

Identify all candidates of the requested type, then decide whether the consumer needs a specific semantic implementation, a default, or all implementations. Use a qualifier for a specific choice, @Primary for a legitimate default, or inject a collection when multiple implementations are intended. Avoid resolving ambiguity by accidental bean names.

### Q15. What is @Import useful for?

@Import brings additional configuration classes or supported import selectors/registrars into a context. It is useful for modular configuration and libraries that provide explicit opt-in setup. It does not mean all classes in an arbitrary package are scanned.

## Official references

- [Container overview](https://docs.spring.io/spring-framework/reference/core/beans/basics.html)
- [Dependency injection](https://docs.spring.io/spring-framework/reference/core/beans/dependencies/factory-collaborators.html)
- [Annotation-based container configuration](https://docs.spring.io/spring-framework/reference/core/beans/annotation-config.html)
- [Spring Boot auto-configuration](https://docs.spring.io/spring-boot/reference/using/auto-configuration.html)
