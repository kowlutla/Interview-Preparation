# Spring Fundamentals

## Q&A

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

## Official references

- [Spring Framework overview](https://docs.spring.io/spring-framework/reference/overview.html)
- [Core technologies and IoC container](https://docs.spring.io/spring-framework/reference/core/beans.html)
- [Spring Boot reference](https://docs.spring.io/spring-boot/reference/)
