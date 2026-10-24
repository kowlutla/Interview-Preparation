# Spring Version Compatibility

Version compatibility changes over time. Treat official release and migration documentation as the source of truth when selecting a runtime or upgrading an application.

## Major compatibility landmarks

| Line | Important compatibility concern | Practical guidance |
|---|---|---|
| Spring Framework 5.x / Spring Boot 2.x | Predominantly javax.* APIs for Java EE-era integrations; older security and HTTP-client patterns are common in tutorials | Treat examples as legacy unless verified against the exact minor release. |
| Spring Framework 6.x / Spring Boot 3.x | Java 17 baseline; Jakarta EE 9+ namespace transition from javax.* to jakarta.*; observability and AOT support evolved in this line | Audit third-party libraries and servlet/container compatibility before migrating. |
| Spring Framework 7.x / Spring Boot 4.x | New major generation with updated runtime and dependency requirements; exact minimum Java and ecosystem compatibility must be checked in the selected release's official docs | Do not infer requirements from a previous major version or an unversioned blog. |

This table is a migration map, not a guarantee that every minor release remains supported. Consult [Spring Framework versions and support](https://spring.io/projects/spring-framework#support) and [Spring Boot support](https://spring.io/projects/spring-boot#support).

## Rules for version-aware examples

- State the Spring Boot version and Java runtime when a feature or API depends on them.
- Let the Boot dependency-management BOM control compatible Spring Framework and managed-library versions unless there is a documented reason to override them.
- Check the Spring Cloud release-train compatibility table before combining Boot and Cloud.
- For Boot 3 migration, search application code and dependencies for Java EE javax.* imports that moved to Jakarta namespaces. Not every javax.* package moved; javax.sql remains part of Java SE.
- Modern servlet security uses SecurityFilterChain beans. WebSecurityConfigurerAdapter is a legacy pattern removed from modern Spring Security.
- RestClient is a modern synchronous client in newer Spring Framework versions; verify the framework baseline before using it. RestTemplate remains present in many applications, but new-client selection should follow current documentation.
- Test annotations and mock-bean APIs evolve. Verify whether @MockitoBean is available in the selected framework line before replacing older @MockBean usage.
- Distinguish a framework feature from Boot auto-configuration for that feature. A class may exist in Spring Framework while Boot's automatic setup differs by release.

## Upgrade checklist

1. Identify target Java, Boot, Framework, Security, Data, and Cloud versions.
2. Read official release notes and migration guides for every major version crossed.
3. Remove deprecated APIs and resolve Jakarta namespace incompatibilities.
4. Inspect dependency convergence and remove unnecessary version overrides.
5. Run unit, slice, integration, security, and end-to-end tests.
6. Validate configuration-property changes and auto-configuration condition reports.
7. Exercise readiness, graceful shutdown, database access, and downstream timeout paths in a production-like environment.
8. Roll out incrementally with metrics and a tested rollback plan.

## Official sources

- [Spring Framework reference](https://docs.spring.io/spring-framework/reference/)
- [Spring Framework releases and support](https://spring.io/projects/spring-framework#support)
- [Spring Boot reference](https://docs.spring.io/spring-boot/reference/)
- [Spring Boot migration guides](https://github.com/spring-projects/spring-boot/wiki)
- [Spring Cloud compatibility guidance](https://spring.io/projects/spring-cloud#overview)
