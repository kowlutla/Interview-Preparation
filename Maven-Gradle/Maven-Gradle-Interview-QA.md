# Maven & Gradle Interview Handbook — Senior Java Developers

**Audience:** Senior Java backend engineers (5–10 years).  
**Perspective:** Java developer only — build configuration, dependencies, tests, packaging, troubleshooting, and practical trade-offs.  
**Goal:** A focused revision guide with 49 high-value Q&As. Commands and snippets are illustrative; verify plugin and tool versions against the project you maintain.

## Contents

1. [Build-tool fundamentals — Q1–Q4](#1-build-tool-fundamentals)
2. [Maven essentials — Q5–Q11](#2-maven-essentials)
3. [Gradle essentials — Q12–Q18](#3-gradle-essentials)
4. [Dependency management and conflicts — Q19–Q25](#4-dependency-management-and-conflicts)
5. [Multi-module builds — Q26–Q29](#5-multi-module-builds)
6. [Plugins, testing, and packaging — Q30–Q33](#6-plugins-testing-and-packaging)
7. [Troubleshooting and performance — Q34–Q38](#7-troubleshooting-and-performance)
8. [Choosing Maven or Gradle — Q39–Q41](#8-choosing-maven-or-gradle)
9. [Scenario-based questions — Q42–Q46](#9-scenario-based-questions)
10. [Senior-level mock interview and revision — Q47–Q49](#10-senior-level-mock-interview-and-revision)
11. [Revision checklist and official references](#11-revision-checklist-and-official-references)

---

## 1. Build-tool fundamentals

### Q1. Why does a Java project need Maven or Gradle?

A build tool automates the repeatable steps between source code and a usable artifact: resolving libraries, compiling code, processing resources, running tests, packaging a JAR/WAR, and publishing artifacts when required. It also records build rules so developers and automation use the same process rather than relying on manually assembled classpaths and commands.

The JDK supplies the compiler and runtime; Maven or Gradle orchestrates them. Dependencies are not bundled automatically just because the code compiles. The packaging plugin and application framework determine the resulting artifact layout—for example, a Spring Boot executable JAR includes application dependencies in its supported nested layout.

### Q2. What are the main differences between Maven and Gradle, and when would you choose each?

Maven uses a convention-oriented XML POM and named lifecycle phases. It is often straightforward to understand across teams because most projects follow similar conventions. Gradle uses a task graph and supports Groovy or Kotlin DSLs; its build logic is more programmable and can offer powerful incremental and caching features.

Neither is universally faster or better. Choose Maven when conventional Java builds and low configuration complexity are priorities. Choose Gradle when its task model, build composition, or performance features solve a real need. Team familiarity, plugin quality, ecosystem compatibility, reproducibility, and maintenance cost matter more than popularity alone.

### Q3. What do declarative configuration and build logic mean in practice?

Maven configuration is primarily declarative: a POM describes project coordinates, dependencies, plugins, and lifecycle bindings. Complex custom behavior is usually delegated to plugins. Gradle scripts can declare configuration and also express build logic in a programming language, which is flexible but can become difficult to maintain if business logic is embedded directly in scripts.

For either tool, prefer standard plugins and shared conventions over bespoke logic. Keep application behavior in Java code, not in build scripts. A build should be understandable by another developer and produce the same intended artifact from the same inputs.

### Q4. What happens when a Java developer builds a Spring Boot application?

The build tool loads project/build configuration, resolves dependencies and plugins, compiles production code, processes resources, compiles and runs tests, and packages the output. Maven executes goals bound to requested lifecycle phases; Gradle selects tasks and their dependencies for the requested task graph. A Spring Boot packaging plugin can then create an executable JAR with the required launcher and nested dependencies.

A successful compile proves only that compilation succeeded against the selected compile classpath. It does not prove that runtime dependency versions are compatible, integration tests passed, or the packaged application starts correctly. Use an appropriate verification task/phase and test the artifact when those guarantees matter.

## 2. Maven essentials

### Q5. What is the Maven lifecycle, and how is a phase different from a goal?

Maven has built-in `clean`, `default`, and `site` lifecycles. A **phase** is a standard point in a lifecycle, such as `test` or `package`; a **goal** is a specific action supplied by a plugin, such as `compiler:compile` or `surefire:test`. Packaging and plugin configuration bind goals to phases. Running a phase executes the preceding phases in that lifecycle, including goals bound to them.

This distinction matters when diagnosing a build: adding a plugin does not automatically mean every goal in it runs. Check its executions and phase bindings. The goals bound by default depend on packaging and may be changed by project configuration. [Maven lifecycle reference](https://maven.apache.org/guides/introduction/introduction-to-the-lifecycle.html).

### Q6. What do the common Maven phases and commands do?

- `mvn validate`: checks that project configuration is valid and required information is available; it does not necessarily compile code.
- `mvn compile`: runs earlier default-lifecycle phases and compiles production source.
- `mvn test`: compiles production and test source and runs unit tests bound to the test phase, commonly through Surefire.
- `mvn package`: runs prior phases and packages the artifact, usually as a JAR/WAR.
- `mvn verify`: runs through verification checks; integration-test setup and execution depend on configured plugins.
- `mvn install`: runs through verification and installs the artifact into the local Maven repository.
- `mvn deploy`: runs the lifecycle through deploy and publishes the artifact to a configured remote repository.
- `mvn clean verify`: removes generated output, then runs the default lifecycle through verification.

The exact goals invoked depend on packaging and plugin bindings. `verify` is commonly a good local/CI validation endpoint; `install` is useful when another local project needs the artifact.

### Q7. What is in a `pom.xml`, and what are Maven coordinates?

The Project Object Model (POM) describes a project's identity, packaging, dependencies, properties, build plugins, and optional parent/module configuration. The usual coordinates are `groupId` (organization or namespace), `artifactId` (project name), and `version`; `packaging` describes the artifact type, commonly `jar`, `war`, or `pom`. Together with repository layout, coordinates identify an artifact.

A `pom`-packaged project often acts as a parent or aggregator rather than producing an application JAR. Properties such as a library version can reduce repetition, but do not replace deliberate dependency management. Inspect the effective POM when inheritance or profiles make the final configuration unclear.

### Q8. Explain Maven dependency scopes and transitive dependencies.

A dependency scope controls where a dependency is available and whether it is propagated to consumers. Common scopes are `compile` (default, available across compile/runtime and generally transitive), `provided` (needed to compile but expected from the runtime/container), `runtime` (not needed to compile the project but needed at runtime), `test` (test code only), and `system` (discouraged because it relies on a local path).

A transitive dependency is brought in by another dependency. For example, library A may bring library B even if the application never declares B directly. This is convenient but can introduce unexpected versions. Use `mvn dependency:tree` to understand the path and scope before excluding or overriding a library. [Maven dependency mechanism](https://maven.apache.org/guides/introduction/introduction-to-dependency-mechanism.html).

### Q9. How do `dependencies` and `dependencyManagement` differ?

`dependencies` adds dependencies to the current project. `dependencyManagement` centralizes versions, scopes, and related defaults for dependencies that child POMs may declare; it does not add those dependencies to a module by itself.

For example, a parent can manage a Jackson version, while only modules that actually use Jackson declare it under `dependencies`. This keeps versions aligned without forcing every module to include every library. Managed versions can still be affected by explicit declarations and Maven's resolution rules, so inspect the effective POM and dependency tree if the selected result is unexpected.

### Q10. What is the difference between `plugins` and `pluginManagement`?

The `build/plugins` section configures plugins that participate in the current project's build. `build/pluginManagement` supplies shared plugin versions and configuration for plugins that are activated in this POM or inherited child builds; it does not, by itself, make every managed plugin execute.

A parent POM can manage compiler, Surefire, and packaging-plugin versions to prevent drift. Child modules then declare or inherit the relevant plugin usage according to the parent configuration. Pin plugin versions for predictable builds instead of relying on whatever version Maven happens to resolve by default.

### Q11. How do parent POMs, BOMs, and Maven profiles fit together?

A parent POM provides inheritance: children can inherit properties, dependency management, plugin management, and build configuration. A BOM (Bill of Materials) is a POM imported with `<type>pom</type>` and `<scope>import</scope>` inside `dependencyManagement`; it centralizes a tested set of dependency versions without requiring inheritance from that BOM as the parent.

A profile conditionally changes configuration based on activation rules, such as a property or environment. Profiles can be useful, but environment-dependent activation can make builds surprising; keep important release behavior explicit and documented. Spring Boot's dependency BOM is commonly used to align compatible Spring ecosystem versions.

## 3. Gradle essentials

### Q12. What are Gradle's initialization, configuration, and execution phases?

**Initialization** determines which projects belong to the build, primarily through `settings.gradle` or `settings.gradle.kts`. **Configuration** evaluates settings and project build scripts, applies plugins, configures projects/tasks, and determines the task graph. **Execution** runs the selected tasks and their dependencies.

Do not assume all build-script statements run only when a task executes: ordinary configuration code runs during configuration. Task actions run during execution. This distinction matters for build speed and correctness; avoid doing expensive I/O or resolving dependencies eagerly in configuration code. Gradle's configuration cache can reuse configured state when its inputs remain valid. [Gradle configuration cache](https://docs.gradle.org/current/userguide/configuration_cache.html).

### Q13. What is the role of Gradle settings and build files?

`settings.gradle` / `settings.gradle.kts` defines the build structure, including project names, included subprojects, and plugin/dependency repository settings where appropriate. A root or module `build.gradle` / `build.gradle.kts` applies plugins and configures dependencies, tasks, toolchains, and project behavior.

The Kotlin DSL uses Kotlin syntax and can benefit from IDE type assistance; the Groovy DSL is dynamic and remains common in existing builds. Both use the same Gradle model. Prefer the DSL already used by the project unless migration offers a clear benefit, and avoid mixing styles unnecessarily.

### Q14. What are Gradle tasks and task dependencies?

A task represents a unit of work, such as `compileJava`, `test`, or `jar`. Tasks can declare dependencies on other tasks, inputs, and outputs. When a task is requested, Gradle builds a directed task graph and executes the required tasks in dependency order; independent tasks may run in parallel when Gradle can safely do so.

Task dependencies describe ordering and prerequisite work, not simply the textual order of declarations in the build file. Use `./gradlew tasks` to discover tasks and `./gradlew build --dry-run` to inspect the planned task execution without running task actions. Avoid manually invoking internal tasks when a lifecycle task such as `check` or `build` expresses the intent.

### Q15. Explain common Gradle dependency configurations.

- `implementation`: dependency required by the module but not exposed as part of its public compile API to consumers.
- `api`: dependency types exposed through a library's public API; generally used with the `java-library` plugin.
- `compileOnly`: needed for compilation but not included on the runtime classpath by that configuration.
- `runtimeOnly`: needed at runtime but not for compiling the module's source.
- `testImplementation`: dependency used to compile and run test code.

Choosing `implementation` over `api` where possible reduces unnecessary compile-classpath exposure and can improve incremental compilation in multi-module builds. Configuration behavior depends on applied plugins, so use the Java/Java Library plugin documentation when interpreting a specific project.

### Q16. What does the Gradle Wrapper do, and why should a team commit it?

The Wrapper scripts and wrapper JAR/properties let the repository select and download a specific Gradle distribution. Developers and CI can run `./gradlew` rather than relying on a globally installed Gradle version. This removes a major source of “works on my machine” differences.

Commit the Wrapper files and review distribution URL changes like code. The Wrapper pins Gradle, not the JDK, dependency versions, operating system, or every external tool. Combine it with an explicit Java toolchain and stable dependency policy for a more consistent build. Use the repository's wrapper rather than silently upgrading Gradle locally.

### Q17. How do Gradle plugins and the task graph work together?

A plugin adds conventions, extensions, configurations, and tasks. For example, the Java plugin creates conventional source sets and tasks such as `compileJava`, `test`, and `jar`. Plugins should be applied and configured through documented extension points rather than by reaching into internal implementation details.

A plugin's presence does not mean every task it creates will execute. The requested task graph determines what runs. Use `./gradlew tasks`, `./gradlew help --task test`, and `./gradlew build --dry-run` to explore available tasks and execution dependencies.

### Q18. Which Gradle commands are useful in everyday Java development?

- `./gradlew clean test`: remove generated build output and run the test task.
- `./gradlew build`: run the project's standard build lifecycle, typically including checks and packaging.
- `./gradlew projects`: list projects in a multi-project build.
- `./gradlew dependencies`: display dependency reports for configurations in the selected project.
- `./gradlew dependencyInsight --dependency jackson-databind`: explain why a matching dependency/version was selected in a relevant configuration; specify a configuration if the report is ambiguous.
- `./gradlew :service-module:test`: run the named subproject's test task.
- `./gradlew --offline test`: use cached dependencies only; it fails if required artifacts are unavailable locally.
- `./gradlew build --scan`: publish a Build Scan only when the configured Gradle tooling and organization policy permit it; review any data-sharing implications first.

The wrapper version and available tasks are project-specific, so check `./gradlew --version` and the project's plugin configuration when commands behave differently.

## 4. Dependency management and conflicts

### Q19. How does Maven resolve dependency-version conflicts?

Maven generally uses **nearest definition** mediation: when multiple versions of the same artifact are encountered, the version at the shortest path from the project wins; if paths are equally deep, the first declaration encountered can win. Dependency management can influence versions used in the project, but it is not the same mechanism as Gradle's conflict resolution.

Do not assume Maven always selects the newest version. Inspect the path and selected version with `mvn dependency:tree -Dverbose`; then centralize a compatible version in dependency management or make a deliberate direct declaration if appropriate. Confirm binary compatibility and test the application after changing versions.

### Q20. How does Gradle resolve version conflicts, and how can versions be constrained?

Gradle resolves a dependency graph per configuration and, by default, conflict resolution generally selects the highest version among conflicting requests for a module. That outcome can be changed by constraints, platforms/BOMs, strict versions, resolution rules, capabilities, or other metadata. Therefore, “Gradle always chooses the newest” is also too broad.

Use constraints or a platform to express intended versions, and prefer dependency insight over guessing. Dynamic versions such as `1.+` and changing modules can make resolution vary over time; avoid them for reproducible production builds unless there is a deliberate refresh policy. [Gradle dependency management](https://docs.gradle.org/current/userguide/dependency_management.html).

### Q21. How do you identify and exclude an unwanted transitive dependency?

First identify the path that brings the dependency in. For Maven, run `mvn dependency:tree -Dverbose`; for Gradle, run `./gradlew dependencyInsight --dependency some-library --configuration runtimeClasspath` (use the relevant configuration). Then decide whether the library is unnecessary, should be replaced with a compatible version, or is required by the dependency that brings it in.

Maven example:

```xml
<dependency>
  <groupId>com.example</groupId>
  <artifactId>client-library</artifactId>
  <version>2.4.0</version>
  <exclusions>
    <exclusion>
      <groupId>org.example</groupId>
      <artifactId>legacy-parser</artifactId>
    </exclusion>
  </exclusions>
</dependency>
```

Gradle example:

```kotlin
dependencies {
    implementation("com.example:client-library:2.4.0") {
        exclude(group = "org.example", module = "legacy-parser")
    }
}
```

An exclusion removes that dependency path; another dependency may still bring the same module in. Re-run the report and tests. Excluding a library that is genuinely required can turn a resolution problem into a runtime failure.

### Q22. What is a BOM, and how does Spring Boot dependency management help?

A BOM records a coordinated set of dependency versions tested or intended to work together. It reduces version drift across related libraries. Spring Boot's dependency management provides a curated set of versions; normally, use the versions managed by the selected Boot release rather than overriding individual Spring/Jackson/logging dependencies without a reason.

Maven can import a BOM in `dependencyManagement`. Gradle can use a platform, for example `implementation(platform("org.springframework.boot:spring-boot-dependencies:<boot-version>"))`, or the Spring dependency-management plugin when the project chooses that model. These approaches are not identical in every detail. Follow the build setup recommended for the Spring Boot version in use, and override versions only after compatibility review.

### Q23. How do you align dependency versions across multiple modules?

For Maven, put shared versions in a parent POM's `dependencyManagement` or import a BOM; modules still declare the dependencies they use. For Gradle, centralize versions through a platform/BOM, version catalog, or shared convention/plugin where suitable. The key is one reviewed source of version intent rather than repeated literals in every module.

Centralization prevents accidental drift but does not automatically guarantee compatibility. A module may have explicit overrides, a plugin may add dependencies, or a different configuration may resolve a different graph. Inspect the effective POM or the exact Gradle configuration's dependency report when the result is unexpected.

### Q24. What is the difference between SNAPSHOT, release, and dynamic dependency versions?

A Maven SNAPSHOT (such as `1.2.0-SNAPSHOT`) represents development work that may be updated before a final release; repository metadata can map it to timestamped snapshot artifacts. A release version is intended to be immutable. Gradle also supports changing modules and dynamic selectors such as version ranges or `1.+`, whose selected results can change as repository contents change.

Mutable versions make builds harder to reproduce and can cause two builds with the same source revision to resolve different bytes. Prefer released, immutable versions for production builds; if snapshots or dynamic versions are necessary, understand refresh and cache policies and record what was actually resolved.

### Q25. What should you check when a dependency exists in a repository but the build cannot resolve it?

Verify the exact coordinates, version, repository URL, credentials, network/proxy access, and whether the artifact is published to that repository. Check whether the dependency is declared in the right scope/configuration and whether a parent POM or BOM is available. A cached failure or stale metadata may also affect subsequent attempts.

For Maven, inspect the error and use `mvn -U ...` to force checks for updated snapshots/releases metadata where applicable; `-U` is not a universal fix for wrong coordinates or authentication. For Gradle, `./gradlew --refresh-dependencies ...` refreshes dependency resolution metadata. Avoid deleting all caches first: capture the error and identify whether the failure is repository access, metadata, authentication, or an invalid coordinate.

## 5. Multi-module builds

### Q26. What is the difference between Maven aggregation and inheritance?

Aggregation lets a POM list modules so Maven can build them together. Inheritance lets a child POM inherit configuration from a parent. One POM can do both, but they are separate concepts: a parent can be inherited without aggregating modules, and an aggregator can list modules without being their parent.

Example aggregator:

```xml
<packaging>pom</packaging>
<modules>
  <module>domain</module>
  <module>service</module>
  <module>application</module>
</modules>
```

A typical parent may also define shared properties and dependency/plugin management. Do not assume a module inherits settings just because it appears in an aggregator's `<modules>`; its `<parent>` declaration controls inheritance.

### Q27. How do you build a specific Maven module and its required reactor dependencies?

Use `mvn -pl service-module -am clean verify`. `-pl` selects the project(s) by reactor selector, and `-am` also builds projects the selected project depends on within the reactor. This is useful when changing a shared module and validating a service without rebuilding every unrelated module.

It does not magically build arbitrary dependencies that are not part of the reactor; those still need to be available from a repository. Check module paths and dependency direction if Maven reports that a required artifact cannot be found.

### Q28. How do Gradle multi-project builds work?

The settings file includes subprojects, commonly with `include("domain", "service", "application")`. Each project can have its own build file and dependencies. Project dependencies express relationships between modules, and Gradle uses them to construct task dependencies and classpaths.

For example, the application can depend on service, which depends on domain. Keep module boundaries meaningful: a shared domain module should not depend back on the application layer. Use root conventions or convention plugins to centralize common build policy, but avoid one giant root script with module-specific conditional logic.

### Q29. When should a Java backend use a multi-module build rather than separate repositories?

Use a multi-module build when modules are developed, versioned, and released together, need frequent local changes across boundaries, or benefit from a shared dependency and build policy. A domain/service/application split can enforce compile-time boundaries and allow modules to be tested independently.

Separate repositories may suit components with independent ownership, release cadence, access controls, or technology choices. A multi-module build is not a substitute for good architecture: excessive cross-module dependencies create coupling, while tiny modules add build and navigation overhead. Make module boundaries follow stable responsibilities, not just package names.

## 6. Plugins, testing, and packaging

### Q30. What do the Maven Compiler Plugin, Surefire, and Failsafe do?

The Maven Compiler Plugin compiles production and test source using the configured compiler settings. Surefire runs unit tests in the `test` phase. Failsafe is designed for integration tests and conventionally binds its execution across `integration-test` and `verify`, allowing post-integration cleanup to run before the final verification result is reported.

A common pattern is `mvn verify` with Failsafe configured for integration tests. Merely naming tests “integration” does not guarantee Failsafe will execute them; plugin executions and include patterns must be configured. Pin plugin versions and check their documentation for version-specific behavior.

### Q31. How do you configure Java version compatibility correctly?

The JDK that runs Maven/Gradle and the Java release targeted by compilation are related but distinct. A newer JDK can compile code targeting an older release if configured correctly; conversely, changing `JAVA_HOME` alone may not change source/API compatibility. Prefer Maven Compiler Plugin's `release` setting where supported, because it constrains language features, bytecode target, and available Java platform APIs for that release.

Gradle example:

```kotlin
java {
    toolchain {
        languageVersion = JavaLanguageVersion.of(17)
    }
}
```

A toolchain selects the JDK used by relevant Java tasks. You may also need an explicit release target for the intended API/bytecode level. Verify with `mvn -version` or `./gradlew --version`, and inspect compiler/toolchain configuration rather than assuming the IDE's JDK controls every build.

### Q32. How should unit and integration tests fit into a Java build?

Unit tests should be fast and isolated; integration tests validate boundaries such as database access, messaging, or HTTP interactions. Maven commonly runs unit tests with Surefire and integration tests with Failsafe. Gradle's `test` task runs the configured test source set; projects can define separate integration-test source sets/tasks where needed.

Keep test selection explicit and make the standard verification task include the checks required for confidence. Skipping tests can be useful for a narrowly scoped local diagnostic, but should not become a way to declare a build healthy. A green compile with tests skipped is not equivalent to a verified change.

### Q33. How is an executable Spring Boot JAR packaged?

The Spring Boot Maven or Gradle plugin can repackage/build the application into an executable archive with a Boot launcher and dependencies in the layout expected by Spring Boot. This differs from a plain library JAR, which typically contains project classes/resources but not all runtime dependencies in an executable arrangement.

Use the plugin configuration appropriate to the selected Spring Boot version and build tool. Check the produced artifact, run its tests, and if needed smoke-test it with `java -jar app.jar`. Avoid manually assembling dependency JARs unless the project has a deliberate packaging requirement; the framework plugin handles details that are easy to get wrong.

## 7. Troubleshooting and performance

### Q34. How do you investigate a Java build that fails with “cannot find symbol” or incompatible class versions?

First read the first meaningful compiler error and confirm which JDK/build tool actually ran: `mvn -version` or `./gradlew --version`. Check the module's compiler release/toolchain, generated-source tasks, imports, and compile classpath. If the symbol comes from a dependency, inspect the resolved version rather than only the version written in a build file.

A “class file has wrong version” error often indicates that a class was compiled for a newer Java release than the compiler/runtime supports. Fix the actual mismatch—align JDK/toolchain and release targets, or use a compatible dependency. Do not randomly upgrade every plugin or dependency.

### Q35. What is the difference between a clean build, incremental build, and build cache?

A clean build removes generated outputs and rebuilds from source, which can expose stale-output problems but costs more time. Incremental/up-to-date checks skip work when a task's declared inputs and outputs indicate that the existing output is still valid. Gradle's build cache can reuse outputs from previous task executions when the cache key matches; the dependency cache stores downloaded artifacts and metadata, which is a different purpose.

Gradle's configuration cache is different again: it reuses configured build state/task graph rather than compiled task outputs. Maven also has a local repository for artifacts, but that is not the same as a general task-output cache. Correctly declared task inputs/outputs and stable build logic are prerequisites for safe caching.

### Q36. What diagnostic steps help when a build is slow?

Measure before changing settings. For Maven, use `mvn clean verify` with timing/logging appropriate to the environment and inspect which plugin/test stage consumes time; `mvn -X` can provide verbose diagnostics but is noisy and may reveal sensitive configuration, so handle logs carefully. For Gradle, use `./gradlew build --profile` or an approved Build Scan to identify time spent in configuration, dependency resolution, compilation, and tests.

Then target the measured bottleneck: remove unnecessary work, avoid repeated dependency versions, improve test isolation/parallelism where safe, enable supported caching, or reduce expensive build-script configuration. Parallel execution can expose shared-resource conflicts or memory pressure. Do not disable tests, checks, or clean steps blindly just to make timings look better.

### Q37. How do offline mode and dependency-cache refresh work?

Maven's `-o` option and Gradle's `--offline` option restrict resolution to locally available artifacts; they are useful on a disconnected network but fail when a required artifact is missing. Gradle's `--refresh-dependencies` asks Gradle to refresh dependency metadata and check artifacts, while Maven's `-U` forces checks for updated snapshots and missing releases.

These flags address cache/metadata questions, not invalid coordinates, incompatible versions, or bad credentials. If a build only works online, identify which dependency is missing from cache and whether the repository configuration is reliable. Avoid treating cache deletion as the first troubleshooting step.

### Q38. Why can a build pass locally but fail on another machine or CI?

Common causes include different JDKs or build-tool versions, uncommitted local files, case-sensitive filesystem differences, environment-dependent profiles, mutable dependency versions, missing repository credentials, stale caches, and tests that depend on execution order or external state.

Use the Gradle Wrapper or a documented Maven version, explicit Java toolchains/release settings, immutable dependency versions, and reproducible test setup. Compare the exact command, tool versions, active profiles/properties, and resolved dependency graph. A local success is evidence only for that environment; verify the same source revision and intended build command in a clean environment.

## 8. Choosing Maven or Gradle

### Q39. When is Maven the better choice?

Maven is a strong choice for conventional Java services where lifecycle conventions, a readable POM, broad plugin support, and low build-script complexity meet the need. It can be highly maintainable for large multi-module applications when shared parent/BOM/plugin policies are disciplined.

Do not choose Maven solely because it is “older” or assume it cannot support sophisticated builds. Before adopting Gradle, identify a concrete limitation that matters to the team. Migrating for style preference alone may create cost without improving developer feedback or artifact reliability.

### Q40. When can Gradle be a better fit, and is it always faster?

Gradle can be attractive when a project benefits from a flexible task graph, custom build conventions, composite/multi-project builds, incremental execution, build caching, or configuration caching. Its performance depends on build design, plugins, task inputs, cacheability, test cost, dependency resolution, and machine/network characteristics. A poorly structured Gradle build can be slower or harder to maintain than a conventional Maven build.

Evaluate measured build time, developer workflow, plugin support, and complexity. Caching only helps when tasks are configured correctly and cache keys are stable; it is not a magic switch.

### Q41. What should a team evaluate before migrating between Maven and Gradle?

Inventory the current lifecycle behavior, dependency graph, plugins, generated code, test phases, packaging, publication, Java versions, and multi-module relationships. Map each behavior to a supported plugin/task in the target tool, and verify that exclusions, BOM/platform handling, scopes/configurations, and artifact metadata have equivalent intended semantics.

Build a small proof of concept, compare the resolved dependency graph and packaged artifacts, and run unit/integration tests. Plan for developer onboarding and maintaining two build definitions during transition. Migrate only when the benefits justify the conversion and long-term ownership cost; do not assume a mechanical translation is behaviorally identical.

## 9. Scenario-based questions

### Q42. A Spring Boot service uses an older library version than the version declared directly. How do you diagnose and fix it?

Start by inspecting the resolved graph, not just the POM/build file.

For Maven, run:

```bash
mvn dependency:tree -Dverbose
mvn help:effective-pom
```

Find the library and identify which path requests each version. Maven's nearest-definition mediation and dependency management can explain why the selected version differs from a transitive request. Check whether a parent POM or imported BOM manages the version and whether a profile changes it. Align the version centrally when possible, then re-run the tree and tests.

For Gradle, run `./gradlew dependencyInsight --dependency library-name --configuration runtimeClasspath` (or the configuration that actually fails). It reports selection reasons and dependency paths. Check platforms, constraints, explicit versions, and resolution rules. Fix the source of the conflict rather than adding arbitrary exclusions. Afterward, test the packaged application because version alignment can expose binary incompatibility.

### Q43. Your Java application compiles but fails at runtime with `NoSuchMethodError`. What is your approach?

`NoSuchMethodError` usually means code was compiled expecting a method that is absent from the class definition loaded at runtime. A frequent cause is incompatible versions of the same library on compile and runtime classpaths, although classloader behavior and packaging can also matter.

Identify the class and method in the stack trace. Compare the compile and runtime graphs: use Maven's dependency tree with scopes and Gradle's dependency reports/insight for `runtimeClasspath`. Inspect the actual packaged Spring Boot JAR if the graph looks correct, since the artifact may contain a different dependency set than expected. Align compatible library versions through the BOM or constraints, remove unintended duplicates, and run a test that exercises the failing path. Compilation alone cannot prove runtime binary compatibility.

### Q44. A multi-module build is getting slower, and local builds differ across developers. How would you improve it?

Separate diagnosis into configuration, dependency resolution, compilation, tests, and packaging. For Gradle, compare `./gradlew build --profile` results and inspect task outcomes; for Maven, inspect phase/plugin timings and test reports. Avoid starting with broad cache deletion or disabling tests.

Then address the measured issue: centralize dependency versions, use immutable releases, commit the Gradle Wrapper or document/pin the Maven version, and declare the intended Java toolchain/release. In Gradle, use incremental/up-to-date behavior and build/configuration caches only where supported; confirm custom tasks declare correct inputs/outputs. In either tool, parallelize independent tests only after checking shared database, port, filesystem, and memory assumptions. Verify improvements with repeated comparable builds and a clean checkout.

### Q45. A dependency works when declared directly but fails when brought in transitively. What would you check?

First establish whether the issue is dependency resolution, missing classes, or runtime incompatibility. Compare the dependency's requested and selected versions and scopes. Use `mvn dependency:tree -Dverbose` for Maven or `./gradlew dependencyInsight --dependency artifact-name --configuration runtimeClasspath` for Gradle. Also inspect the compile classpath if the failure occurs during compilation.

A direct declaration may change mediation/selection, scope, or ordering; it does not necessarily mean the library itself is defective. Check whether the transitive parent expects a different version, whether a BOM or constraint manages the artifact, and whether an exclusion removes a required module. Prefer a compatible, centrally managed version over an unexplained direct override. Re-run dependency reports and tests, including the packaged runtime path.

### Q46. A developer upgraded the local JDK, but the project still compiles for an older Java release. How do you fix it?

Check the JDK running the build first with `mvn -version` or `./gradlew --version`. Then inspect the project's compiler configuration: Maven Compiler Plugin's `release` setting, Gradle Java toolchain, and any explicit source/target compatibility settings. The JDK that runs the build and the release level of generated bytecode are different controls; a newer JDK can intentionally compile for an older Java release.

Agree on the project's intended Java baseline before changing settings. Configure the toolchain/compiler release in version-controlled build files, confirm that required APIs are available at that release, and run tests with the supported runtime. Do not rely only on a developer's `JAVA_HOME` or IDE configuration, because those may differ from command-line or CI builds.

## 10. Senior-level mock interview and revision

### Q47. Design a maintainable build for a Spring Boot backend with domain, service, persistence, and application modules. What decisions would you make?

I would start with module boundaries and dependency direction, not with the build tool. The domain module should contain core domain types and rules without depending on Spring or persistence details. The service module can depend on domain and coordinate use cases; persistence implements data access and depends on the abstractions it needs; the application module wires the service and infrastructure into the Spring Boot application. I would avoid cycles and keep module APIs intentionally small.

For Maven, use an aggregator POM to build the modules together and a parent POM or imported BOM to centralize dependency/plugin versions. Remember that aggregation does not itself create inheritance: each child needs the correct parent relationship if it should inherit shared policy. Use `mvn -pl application -am verify` for focused reactor builds with required in-reactor dependencies.

For Gradle, include modules in settings, declare project dependencies explicitly, and use a platform/version catalog or convention plugin for shared policy. Use `implementation` by default and `api` only where types are intentionally exposed to consumers. Commit the Wrapper and configure a Java toolchain. In either tool, pin plugin versions, separate unit and integration testing clearly, package the executable JAR with the appropriate Spring Boot plugin, and keep dependency reports available for troubleshooting.

I would validate the design with a clean build, tests, dependency-graph inspection, and a check that the produced artifact starts and contains the expected runtime libraries. The best build is not the one with the most abstractions; it is the simplest one that is repeatable, understandable, and fast enough for the team.

### Q48. Your team is debating Maven versus Gradle for a new service. How would you make the decision?

I would compare the actual build requirements rather than choosing by reputation. If the service follows a conventional Spring Boot structure, uses standard testing and packaging, and the team is comfortable with Maven, a well-managed POM and BOM may be the lowest-maintenance choice. If the team needs shared build conventions across many modules, custom task composition, or has measured opportunities for Gradle's incremental and caching model, Gradle may be justified.

I would prototype one representative module and compare resolved dependencies, test execution, artifact contents, build duration, IDE support, and the effort required to maintain the build. I would also review plugin maturity and how the team will pin the Gradle Wrapper/JDK or Maven/plugin versions. The decision should account for long-term ownership and onboarding, not just a one-time benchmark.

### Q49. A build is unreliable because developers see different dependency versions. What policy would you propose?

I would centralize dependency intent with a Maven parent/BOM or Gradle platform/version catalog, prefer immutable release versions, and review overrides as deliberate compatibility decisions. I would pin build plugins and the build tool version, and document the supported Java toolchain. I would also make dependency-graph inspection part of troubleshooting: Maven's dependency tree and Gradle's dependency insight show why a version was selected.

Where the project needs stronger repeatability, I would consider dependency locking and artifact verification supported by the chosen tool and repository setup. Those mechanisms should be introduced with a clear update workflow; stale locks or checksums can otherwise obstruct legitimate upgrades. Finally, validate both compile-time and runtime configurations and test the packaged application, because a consistent graph can still contain incompatible libraries.

## 11. Revision checklist and official references

### Before an interview, make sure you can explain

- [ ] Lifecycle phases versus plugin goals; Maven `verify`, `install`, and `deploy` (Q5–Q6).
- [ ] `dependencies` vs `dependencyManagement`; `plugins` vs `pluginManagement`; parent vs BOM (Q9–Q11).
- [ ] Gradle configuration versus execution, tasks, Wrapper, and dependency configurations (Q12–Q16).
- [ ] Maven nearest-definition mediation versus Gradle's version conflict resolution (Q19–Q20).
- [ ] Dependency tree/insight, exclusions, BOMs, and reproducibility (Q21–Q25).
- [ ] Aggregation vs inheritance and selecting modules with their prerequisites (Q26–Q29).
- [ ] Compiler/toolchain settings, Surefire/Failsafe, and executable Spring Boot JARs (Q30–Q33).
- [ ] Build/cache distinctions and evidence-led troubleshooting (Q34–Q38).
- [ ] Trade-offs and migration risks rather than declaring one tool universally superior (Q39–Q49).

### Official documentation

- [Maven Guides](https://maven.apache.org/guides/)
- [Maven POM Reference](https://maven.apache.org/pom.html)
- [Maven Build Lifecycle](https://maven.apache.org/guides/introduction/introduction-to-the-lifecycle.html)
- [Maven Dependency Mechanism](https://maven.apache.org/guides/introduction/introduction-to-dependency-mechanism.html)
- [Maven Compiler Plugin](https://maven.apache.org/plugins/maven-compiler-plugin/)
- [Maven Surefire Plugin](https://maven.apache.org/surefire/maven-surefire-plugin/)
- [Maven Failsafe Plugin](https://maven.apache.org/surefire/maven-failsafe-plugin/)
- [Gradle User Manual](https://docs.gradle.org/current/userguide/userguide.html)
- [Gradle Dependency Management](https://docs.gradle.org/current/userguide/dependency_management.html)
- [Gradle Multi-Project Builds](https://docs.gradle.org/current/userguide/multi_project_builds.html)
- [Gradle Java Toolchains](https://docs.gradle.org/current/userguide/toolchains.html)
- [Gradle Build Cache](https://docs.gradle.org/current/userguide/build_cache.html)
- [Gradle Dependency Cache](https://docs.gradle.org/current/userguide/dependency_caching.html)
- [Spring Boot Build Systems](https://docs.spring.io/spring-boot/)

> **Version note:** Maven and Gradle plugin behavior evolves. Use the documentation matching the project's actual tool/plugin versions, especially for toolchains, dependency platforms, test execution, and caching.
