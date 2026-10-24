# Auto-Configuration and Application Startup

## Q&A

### Q1. How does Spring Boot auto-configuration decide what to configure?

Boot imports candidate auto-configuration classes and evaluates conditions against classpath contents, bean definitions, properties, and application type. Conditions can enable defaults only when their prerequisites hold. Inspect the condition evaluation report rather than guessing which starter or property caused a configuration to activate.

### Q2. Why does auto-configuration often back off when a user bean exists?

Many auto-configurations use conditions such as `@ConditionalOnMissingBean` so an application can replace a default. The condition is evaluated during configuration processing, so registration order and the type/name of the user bean can matter. This is a customization mechanism, not a guarantee that any similarly named bean replaces all related infrastructure.

### Q3. How do you exclude an auto-configuration?

Use the supported exclusion mechanisms, such as the exclude attribute on `@SpringBootApplication` or the corresponding configuration property. First verify why the configuration matched; excluding it may remove supporting beans that other components rely on. Prefer correcting the classpath or property condition when that is the actual cause.

### Q4. How do auto-configuration registration mechanisms differ across Boot generations?

The registration metadata evolved. Older Boot generations commonly used `spring.factories` for auto-configuration candidates; modern Boot uses the dedicated `AutoConfiguration.imports` metadata for auto-configuration discovery. When building a starter, follow the documentation for the exact Boot baseline instead of copying old registration examples.

### Q5. What is the difference between a BeanFactoryPostProcessor and a BeanPostProcessor during startup?

The former modifies bean-definition metadata before ordinary bean instantiation; the latter can process bean instances before and after initialization. Infrastructure such as proxy creation uses instance post-processing. Accidentally triggering bean creation too early from infrastructure code can cause incomplete post-processing or ordering problems.

### Q6. What does ApplicationContext refresh mean?

Refresh initializes the context's infrastructure, applies factory post-processors, registers bean post-processors, instantiates eligible non-lazy singletons, and completes lifecycle coordination. Exact details depend on context type and version. A failure during refresh commonly prevents the application from reaching its ready state.

### Q7. How do you investigate slow startup?

Measure startup phases before optimizing. Inspect startup logs and Boot startup metrics or the available startup endpoint/tooling, identify expensive bean constructors and initialization callbacks, and check classpath scanning, remote calls, migrations, and connection initialization. Lazy initialization can shift work rather than remove it, so validate first-request latency and readiness afterward.

### Q8. What is a custom auto-configuration module?

It is a library module that conditionally contributes beans when relevant classes or properties are present and backs off when the application supplies replacements. Keep conditions precise, avoid scanning an application's packages from a library, document properties, and test positive, negative, and override cases with Boot's auto-configuration test support.

## Official references

- [Boot auto-configuration](https://docs.spring.io/spring-boot/reference/using/auto-configuration.html)
- [Boot startup and application lifecycle](https://docs.spring.io/spring-boot/reference/features/spring-application.html)
