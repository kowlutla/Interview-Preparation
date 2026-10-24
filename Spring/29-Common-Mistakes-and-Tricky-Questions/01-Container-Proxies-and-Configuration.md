# Common Mistakes and Tricky Questions — Container, Proxies, Configuration

## Q&A

### Q1. Is a bean singleton across all ApplicationContext instances?
No. Singleton scope is per bean definition per container. Separate contexts can create separate instances, which matters in tests, parent-child contexts, and applications that construct multiple contexts.

### Q2. Does @Autowired mean a field is initialized before the constructor runs?
No. Field injection occurs after construction. Constructor injection is preferable for required dependencies because the object cannot be constructed without them.

### Q3. Can Spring inject a List of all implementations?
Yes, when the injection point requests a supported collection of matching beans. This is useful for strategy registries, but define deterministic ordering if execution order matters.

### Q4. Does @Primary resolve every ambiguity?
No. It provides a preferred candidate for certain single-valued resolution cases. Qualifiers, collections, generics, and explicit names can affect resolution.

### Q5. Why can a bean disappear only in one profile?
Registration may be conditioned on @Profile, a property, classpath presence, or another condition. Compare active profiles and the condition report across environments.

### Q6. Does @Lazy make a bean faster?
It defers initialization; it does not reduce total work by itself. It may shift latency and failure into the first request.

### Q7. Are prototype beans automatically destroyed by Spring?
Spring generally does not manage the complete destruction lifecycle of prototype instances after handing them to the caller. The caller must release owned resources.

### Q8. Can @PostConstruct safely call any other bean?
It can create ordering and early-initialization problems, especially when it triggers dependencies or proxies before the context is fully ready. Keep lifecycle callbacks simple.

### Q9. Why might a BeanPostProcessor miss a bean?
If that bean is created too early, before all post-processors are registered, it may not receive normal processing. Avoid eagerly resolving ordinary beans inside infrastructure post-processors.

### Q10. What is the difference between a bean definition and a bean instance?
A bean definition is metadata describing creation/configuration; an instance is the actual object produced from that metadata.

### Q11. Can @Bean methods be called like ordinary factory methods?
In full @Configuration mode, inter-bean calls are intercepted to preserve container semantics. In lite mode, direct Java method calls can create ordinary objects rather than return the managed bean. Avoid relying on implicit behavior without knowing the mode.

### Q12. Why is component-scan package placement important?
The default scan root is based on the configuration class's package. A main class in a narrow package may fail to discover sibling application components.

### Q13. Is @Import equivalent to @ComponentScan?
No. Import registers specific configuration/classes or supported selectors/registrars; component scanning discovers eligible components across packages.

### Q14. Can @ConditionalOnMissingBean be used anywhere without concern?
It is primarily useful in conditional configuration and auto-configuration. Conditions depend on what definitions are known at evaluation time, so ordering and context tests matter.

### Q15. What is the difference between Environment and @ConfigurationProperties?
Environment exposes property sources and resolution; @ConfigurationProperties binds a group of values to a typed object and can support validation.

### Q16. Does @Value support type conversion?
Yes, supported conversion and expression mechanisms apply, but complex grouped configuration is generally easier to maintain with @ConfigurationProperties.

### Q17. Why should secrets not be printed from configuration objects?
Diagnostic output can expose credentials in logs, traces, error reports, or support bundles. Redact secret values and restrict sensitive diagnostic endpoints.

### Q18. How can relaxed binding mask a configuration typo?
Relaxed binding accepts documented naming variants, but it does not make every misspelling valid. A misspelled or unknown property may remain unused unless binding validation or unknown-field policy catches it. Prefer canonical kebab-case keys, validate required values, and test effective binding for critical configuration.

### Q19. Can a configuration property update a singleton automatically?
Not generally. Bound configuration is usually established during startup unless an explicit refresh mechanism is configured and supported.

### Q20. What testing risks do composed annotations introduce?
A composed annotation can combine meta-annotations, attribute aliases, and defaults, so a change may alter component scanning, transaction/security metadata, or request mapping behavior in ways that are not obvious at the use site. Add focused tests for the composed annotation's effective attributes and avoid hiding many unrelated behaviors behind one annotation.

### Q21. Why does self-invocation bypass proxy advice?
A call through this references the target object directly and does not cross the external proxy boundary where advice is applied.

### Q22. What failure can occur when code expects a concrete class but Spring injects a JDK proxy?
The bean may be exposed through its interface proxy, so injection or casting to the concrete implementation can fail with a type mismatch or `ClassCastException`. Prefer injecting the service interface when possible, and choose proxy strategy deliberately only when concrete-type access is genuinely required. Proxy type can also affect framework features and test assumptions.

### Q23. Does @Around advice have to call proceed()?
If the target invocation should happen, advice must proceed appropriately. Omitting it intentionally short-circuits the call; accidental omission suppresses business logic.

### Q24. Why can broad pointcuts be risky?
They may intercept infrastructure or unrelated methods, cause recursion, double logging, unexpected latency, or advice ordering conflicts.

### Q25. Does @Order define every possible proxy interaction order?
It influences supported ordering mechanisms, but actual nesting also depends on proxy creation and advisor composition. Test the behavior that matters.

### Q26. Can final methods be intercepted by a class-based proxy?
Subclass proxies cannot override final methods. Use a compatible design and verify the proxy mechanism.

### Q27. Why can a transaction annotation be ignored on a private method?
Proxy-based interception commonly applies to externally invoked eligible methods; private/internal calls do not cross the proxy boundary. Check exact proxy mode and version semantics.

### Q28. Does Spring AOP intercept constructors by default?
Spring AOP's proxy model targets method executions on Spring-managed proxies, not arbitrary constructor execution. Constructor instrumentation needs a different approach.

### Q29. Is @Transactional inherited from every interface automatically in all contexts?
Annotation lookup and proxy behavior have nuances; place transaction metadata on the concrete service method/class where appropriate and verify the selected framework's rules.

### Q30. Can two @Transactional methods use different transaction managers?
Yes, explicit transaction-manager selection is possible, but the transaction scope and resources must be understood. One local transaction manager does not atomically coordinate unrelated resources.

### Q31. Why can bean overriding hide bugs?
An unintended same-name definition can replace another definition depending on configuration. Explicit conditions and qualifiers are easier to reason about than accidental replacement.

### Q32. What lookup surprises can a parent-child ApplicationContext hierarchy introduce?
A child context can generally resolve beans from its parent, but the parent cannot see child beans; child definitions may shadow parent candidates and make behavior differ across contexts. Diagnose which context owns each bean and where the injection occurs before changing component scanning or bean names.

### Q33. Why does startup ordering matter for custom infrastructure?
Post-processors and configuration registrars operate at specific lifecycle phases. Early bean creation can prevent later infrastructure from processing objects normally.

### Q34. What is the role of an ImportSelector?
It programmatically selects configuration classes to import based on metadata or environment. Keep selection deterministic and document the conditions.

### Q35. What is the role of an ImportBeanDefinitionRegistrar?
It programmatically registers bean definitions, useful for advanced framework/library integrations. Prefer ordinary configuration unless dynamic registration is necessary.

### Q36. Does a successful context refresh guarantee the service is ready?
Not necessarily. Lazy dependencies, external connectivity, warmup, or application-specific readiness may still be incomplete. Readiness must represent the service's actual ability to accept traffic.

### Q37. Why can startup runners delay readiness?
Runners execute after context startup but before the application is considered fully ready in the normal lifecycle. Slow runner work can extend startup and deployment time.

### Q38. What should a custom starter avoid?
It should avoid broad component scanning, unconditional beans that cannot be replaced, secret defaults, undocumented properties, and unnecessary transitive dependencies.

### Q39. How should a custom starter be tested?
Test conditions with and without optional classes/properties, default beans, user overrides, and missing required dependencies using focused application contexts.

### Q40. Why should library auto-configuration be version-specific?
Registration metadata and supported APIs can change across Boot generations. A starter should target and test a declared compatibility baseline.

### Q41. How do you distinguish an auto-configuration condition mismatch from a component-scan problem?
Use the condition evaluation report to inspect conditional auto-configuration, then verify whether the expected application component is in the configured scan root or explicitly imported. A condition report explains auto-configuration decisions; it does not prove that an ordinary `@Component` was discovered. Check bean definitions, active profiles, package boundaries, and effective properties before adding dependencies or broadening scans.

### Q42. Why can adding a dependency change application behavior?
It may add classes that activate conditional auto-configuration, change managed versions, or introduce duplicate components. Review resolved dependencies and condition outcomes.

### Q43. Is XML configuration unsupported?
No. XML remains available for legacy and specific integration scenarios, though Java/annotation configuration is common in modern applications. Choose based on maintainability and migration constraints.

### Q44. How do you safely migrate XML bean definitions?
Inventory namespaces and bean dependencies, migrate incrementally, test wiring and lifecycle behavior, and avoid registering the same bean in both XML and component scanning.

### Q45. Why should application configuration be modular?
Clear configuration boundaries improve ownership, testability, and diagnosis. Over-fragmentation, however, makes property origins and bean conditions harder to follow.

### Q46. What does a missing class error suggest?
A dependency may be absent at runtime, excluded transitively, or incompatible with the selected version. Inspect the resolved graph and the first causal stack-trace frame.

### Q47. What does NoSuchMethodError usually indicate?
It often indicates binary incompatibility: code was compiled against one API version but runs with another. Align dependency versions instead of randomly adding jars.

### Q48. Why is the first stack-trace cause important?
Higher-level Spring exceptions often wrap the original cause. The deepest relevant cause often identifies the real configuration or linkage failure.

### Q49. How do you distinguish a Spring bug from misconfiguration?
Build a minimal reproduction, record exact versions, compare behavior with documented contracts, inspect proxy/condition assumptions, and eliminate application-specific factors before reporting a framework defect.

### Q50. What is the best way to answer a tricky Spring question?
State the default behavior, qualify it by version/configuration, explain the mechanism, identify the failure mode, and describe how you would verify it in a test or production diagnostic.
