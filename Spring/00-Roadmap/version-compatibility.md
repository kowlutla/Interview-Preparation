# Spring Version Compatibility

Compatibility and support status change over time. This guide was checked on 2026-10-24; re-check exact patch release requirements before an upgrade.

## Major compatibility landmarks

| Line | Runtime and platform baseline | Practical guidance |
|---|---|---|
| Spring Framework 5.3 / Spring Boot 2.x | Legacy generation; Framework 5.3 uses Java EE 8 / javax namespace for relevant specifications | Framework 5.3 open-source support ended in August 2024; commercial support may differ. |
| Spring Framework 6.x / Spring Boot 3.x | Java 17 baseline; Jakarta namespace migration for specifications that moved from javax to jakarta | Framework 6.2 open-source support ended in June 2026; check current commercial support options. |
| Spring Framework 7.x / Spring Boot 4.x | Java 17 minimum; Framework 7 adopts Jakarta EE 11, including Servlet 6.1 | Framework 7 recommends Java 25 for production while retaining a Java 17 baseline. Boot 4 requires Framework 7.x. |

The official documentation snapshot checked on 2026-10-24 lists Spring Framework 7.0.10 and 6.2.19 as stable, with Framework 7.1.0-RC1 as preview; it lists Spring Boot 4.1.2, 4.0.9, 3.5.16, 3.4.13, and 3.3.13 as stable, with Boot 4.2.0-RC1 as preview. The Boot 4.1.2 release line is aligned with Spring Framework 7.0.10 or above; note that the generic system-requirements page can lag patch-level releases, so verify the exact Boot BOM and resolved Framework version rather than relying only on the landing page. These are point-in-time listings, not a permanent compatibility guarantee. Verify the exact target patch's system-requirements page before upgrading; do not infer the minimum Framework patch from the top-level reference landing page alone.

## Selected compatibility details

The generic Spring Boot system-requirements page observed during this audit still renders Boot 4.1.0 rather than the latest 4.1.2 patch. It lists Java 17 as the minimum, Java 26 as the maximum supported version for that displayed release, Spring Framework 7.0.8 or later, Maven 3.6.3 or later, Gradle 8.14+ or 9.x, Tomcat 11 / Jetty 12.1 (Servlet 6.1), and GraalVM 25+ for native images. Treat those as the page's displayed 4.1.0 requirements, not automatically as the complete requirements for 4.1.2.

The 4.1.2 release line is aligned with Spring Framework 7.0.10 or above according to the current release/BOM snapshot. The exact versioned 4.1.2 system-requirements page could not be independently confirmed through this review's documentation fetch, so verify the target patch's managed dependency versions and published requirements before setting Java/build-tool bounds. Do not infer maximum JDK compatibility from a generic or lagging landing page.

Framework 7.0 retains a JDK 17 baseline and adopts Jakarta EE 11 APIs such as Servlet 6.1, JPA 3.2, and Bean Validation 3.1. The Framework 7.1 release-notes page is a prerelease scheduled ahead of the November 2026 line; do not treat its upcoming API removals or baselines as already released stable behavior.

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
- [Spring Framework 7.1 prerelease notes](https://github.com/spring-projects/spring-framework/wiki/Spring-Framework-7.1-Release-Notes)
- [Spring Framework release list](https://github.com/spring-projects/spring-framework/releases)
- [Spring Boot 4.0 system requirements](https://docs.spring.io/spring-boot/4.0/system-requirements.html)
- [Spring Boot current system requirements](https://docs.spring.io/spring-boot/system-requirements.html)
- [Spring Boot 4.0 migration guide](https://github.com/spring-projects/spring-boot/wiki/Spring-Boot-4.0-Migration-Guide)
- [Spring Cloud compatibility guidance](https://spring.io/projects/spring-cloud)
- [Spring REST clients](https://docs.spring.io/spring-framework/reference/integration/rest-clients.html)
