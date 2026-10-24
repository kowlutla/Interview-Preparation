# Spring Version Compatibility

Compatibility and support status change over time. This guide was checked on 2026-10-24; re-check exact patch release requirements before an upgrade.

## Major compatibility landmarks

| Line | Runtime and platform baseline | Practical guidance |
|---|---|---|
| Spring Framework 5.3 / Spring Boot 2.x | Legacy generation; Framework 5.3 uses Java EE 8 / javax namespace for relevant specifications | Framework 5.3 open-source support ended in August 2024; commercial support may differ. |
| Spring Framework 6.x / Spring Boot 3.x | Java 17 baseline; Jakarta namespace migration for specifications that moved from javax to jakarta | Framework 6.2 open-source support ended in June 2026; check current commercial support options. |
| Spring Framework 7.x / Spring Boot 4.x | Java 17 minimum; Framework 7 adopts Jakarta EE 11, including Servlet 6.1 | Framework 7 recommends Java 25 for production while retaining a Java 17 baseline. Boot 4 requires Framework 7.x. |

The official documentation currently lists Spring Framework 7.0.10 and Spring Boot 4.1.2 as stable releases (checked 2026-10-24); Boot 4.0.9 and Boot 3.5.16 are also listed as stable lines. Framework 7.1.0-RC1 is a preview release scheduled for November 2026, not a production baseline. The cited Boot 4.0/4.1 patch requirements specify Java 17 or later and Spring Framework 7.0.10 or later. Treat these as a dated snapshot, not a permanent compatibility guarantee; verify the exact patch's system-requirements page before upgrading.

## Selected compatibility details

- Spring Framework 7.0 retains a JDK 17 baseline and adopts Jakarta EE 11 APIs such as Servlet 6.1, JPA 3.2, and Bean Validation 3.1. The Framework 7.1 release-notes page is a preview; do not treat its upcoming API removals or baselines as already released stable behavior.
- Spring Boot 4.x requires Java 17 or later and Spring Framework 7.x. Verify the exact Boot patch's system-requirements page and build-tool versions.
- Spring Cloud compatibility is release-train-specific. The official matrix maps 2025.1.x to Boot 4.0.x and, starting with 2025.1.2, Boot 4.1.x; 2025.0.x maps to Boot 3.5.x. Re-check before upgrading.
- RestClient was introduced in Spring Framework 6.1. Spring Framework 7.0 deprecates RestTemplate in favor of RestClient; existing systems may still use RestTemplate, but new code should follow the target framework's migration guidance.
- Not every javax.* package moved. For example, javax.sql remains part of Java SE. Migrate only APIs whose specifications moved to Jakarta namespaces.
- Modern servlet security uses SecurityFilterChain beans. WebSecurityConfigurerAdapter is a legacy pattern removed from modern Spring Security.
- Let the Boot dependency-management BOM control managed dependency versions unless there is a documented reason to override them.
- Distinguish a Spring Framework feature from Boot auto-configuration for that feature. Framework API availability does not imply identical Boot setup across release lines.

## Upgrade checklist

1. Record target Java, Boot, Framework, Security, Data, Cloud, servlet container, and build-tool versions.
2. Read official release notes and migration guides for every major version crossed.
3. Audit deprecated/removed APIs, Jakarta namespace changes, and third-party compatibility.
4. Inspect dependency convergence and remove unnecessary version overrides.
5. Run unit, slice, integration, security, and end-to-end tests.
6. Validate configuration-property changes and auto-configuration condition reports.
7. Exercise readiness, graceful shutdown, database access, messaging, and downstream timeout paths in a production-like environment.
8. Roll out incrementally with version-correlated metrics and a tested rollback/recovery plan.

## Official sources

- [Spring Framework versions and support](https://github.com/spring-projects/spring-framework/wiki/Spring-Framework-Versions)
- [Spring Framework 7.0 release notes](https://github.com/spring-projects/spring-framework/wiki/Spring-Framework-7.0-Release-Notes)
- [Spring Framework 7.1 preview release notes](https://github.com/spring-projects/spring-framework/wiki/Spring-Framework-7.1-Release-Notes)
- [Spring Framework release list](https://github.com/spring-projects/spring-framework/releases)
- [Spring Boot 4.0 system requirements](https://docs.spring.io/spring-boot/4.0/system-requirements.html)
- [Spring Boot current system requirements](https://docs.spring.io/spring-boot/system-requirements.html)
- [Spring Boot 4.0 migration guide](https://github.com/spring-projects/spring-boot/wiki/Spring-Boot-4.0-Migration-Guide)
- [Spring Cloud compatibility guidance](https://spring.io/projects/spring-cloud)
- [Spring REST clients](https://docs.spring.io/spring-framework/reference/integration/rest-clients.html)
