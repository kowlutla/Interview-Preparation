# Modules and the Java Platform Module System (JPMS)

**Target level:** Advanced → Senior  
**Scope:** Java 9–21

JPMS introduced explicit module boundaries in Java 9. Modules declare dependencies and which packages they export, improving encapsulation, configuration, and platform image construction. They add a layer beyond packages and classpath-based visibility; they do not replace every existing classpath application automatically.

## 1. Module basics

### Q1. What is a Java module? *(Intermediate)*

A module is a named collection of packages and resources with a declared dependency and access boundary. A named module is described by module-info.java and can declare requires, exports, opens, uses, and provides directives.

Modules help make dependencies explicit and can strongly encapsulate implementation packages. They are especially useful for modular applications and libraries, but migration must account for automatic modules, unnamed modules, reflection, and third-party libraries.

### Q2. What is module-info.java? *(Intermediate)*

module-info.java declares a module's name, dependencies, exported packages, reflective openings, service usage, and service providers. It is compiled into module-info.class.

A module descriptor should express the intended public API and dependency graph. Exporting an entire implementation package or using broad opens directives weakens encapsulation and can create accidental compatibility commitments.

### Q3. What is the difference between requires and exports? *(Intermediate)*

requires declares a dependency on another module. exports makes a package's public types accessible to other modules under JPMS access rules. Requiring a module does not automatically make all its packages accessible.

Use qualified exports when a package should be accessible only to specific modules. Keep internal implementation packages unexported unless they are deliberately part of the public contract.

### Q4. What is the difference between exports and opens? *(Advanced)*

exports allows ordinary access to public types in a package from other modules. opens permits deep reflection on members in the package, subject to the directive's target. A package can be open to all modules or opened to specific modules.

Frameworks that reflect on private fields or constructors may require opens. Prefer targeted openings and avoid broad openness when a framework can use public constructors or generated accessors instead.

### Q5. What is requires transitive? *(Advanced)*

requires transitive makes a dependency readable to modules that depend on the declaring module, supporting APIs that expose types from that dependency. It can be appropriate when the dependency is part of the module's public API surface.

Use it carefully because it expands downstream readability and can increase coupling. A library should distinguish implementation dependencies from types consumers must access to use its API.

### Q6. What is requires static? *(Advanced)*

requires static declares a dependency needed at compile time but optional at runtime in the relevant module resolution scenarios. It is useful for optional annotations or tooling dependencies.

Do not use it to hide a dependency that is actually required for normal runtime behavior. Test runtime startup in the intended deployment configuration.

### Q7. What is the unnamed module? *(Intermediate)*

Code on the traditional classpath belongs to an unnamed module associated with its class loader. Unnamed modules read named modules under JPMS compatibility rules, while named modules do not gain unrestricted readability to all unnamed-module code.

This supports migration but is not a substitute for a clear module graph. Classpath and module-path behavior can differ, especially around split packages, reflection, and service loading.

### Q8. What is an automatic module? *(Advanced)*

A non-modular JAR placed on the module path can become an automatic module with a derived module name and broad exports/readability behavior intended to aid migration. Its name may be derived from the JAR filename or manifest metadata.

Automatic modules are a transitional bridge, not the same as an intentionally designed explicit module. Avoid depending on unstable derived names in long-lived module descriptors.

## 2. Services and packaging

### Q9. How do uses and provides work? *(Advanced)*

uses declares that a module consumes a service interface. provides ... with ... declares an implementation provider. ServiceLoader can discover providers under the supported service-loading mechanism.

This separates a stable service contract from implementations and supports plugin-style architectures. Provider lifecycle, error handling, version compatibility, and trust still need explicit design.

### Q10. What is the difference between classpath and module path? *(Advanced)*

Classpath loading uses the traditional class-loader model and unnamed modules. Module path resolution treats modular JARs as named modules and checks the module graph. Packaging on the module path can expose split-package, dependency, and access issues that were not obvious on the classpath.

Test the exact deployment command and packaging. A project that compiles in an IDE may behave differently when launched as a named module.

### Q11. What is jlink? *(Intermediate)*

jlink assembles a custom runtime image from a set of modules and their dependencies. It can reduce deployment size and make the runtime contents explicit.

A custom image must include all modules required by reflection, services, and runtime configuration. Dynamic class loading or optional features may require additional modules; test the packaged image, not just the development JDK.

### Q12. What is jdeps? *(Intermediate)*

jdeps analyzes class or module dependencies and can help identify packages, modules, and internal APIs used by an application. It is useful during migration and dependency review.

Static analysis cannot always infer dynamically loaded classes or reflection-only dependencies. Combine jdeps output with integration tests, startup diagnostics, and knowledge of framework behavior.

## 3. Migration and production

### Q13. Why can reflection fail after modularization? *(Advanced)*

JPMS strengthens encapsulation. Reflective access to non-public members in a package that is not open to the caller's module may fail, even if the code previously worked on the classpath or with older JDK flags.

Prefer public APIs or targeted opens directives. Avoid treating command-line flags that open many JDK internals as a permanent solution; they can disappear or be restricted in later releases.

### Q14. What is a split package, and why is it problematic? *(Advanced)*

A split package occurs when the same package is present in multiple modules in a way that conflicts with module-system rules. This can make module resolution fail or create ambiguous ownership.

Avoid distributing one package across multiple named modules. Refactor package ownership or keep legacy components on the classpath during a controlled migration if appropriate.

### Q15. How would you migrate a large classpath application to JPMS? *(Senior)*

Inventory dependencies with jdeps, identify internal API usage, determine stable module boundaries, and add descriptors incrementally. Start with leaf libraries or well-isolated components, then move toward application modules. Test reflection-heavy frameworks, service loading, plugins, and production packaging.

Use automatic modules as a transition only when necessary. Maintain a tested deployment path, document qualified exports/opens, and avoid turning every package into public API merely to make migration pass.

### Q16. Does JPMS replace package access modifiers? *(Intermediate)*

No. Java access modifiers still apply within and across types and packages. JPMS adds module readability and package export/open rules on top. Access succeeds only when the relevant language and module rules permit it.

Think of access as layered: member/type visibility, package/module relationships, and runtime reflection access. A public class in a non-exported package is not automatically accessible to every other named module.

## Official references

1. [JEP 261 — Module System](https://openjdk.org/jeps/261)
2. [Java SE 21 API — Module](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/lang/Module.html)
3. [Java Language Specification — Modules](https://docs.oracle.com/javase/specs/jls/se21/html/jls-7.html)
4. [Java Platform Module System guide](https://dev.java/learn/modules/)
5. [Java SE 21 tool documentation](https://docs.oracle.com/en/java/javase/21/docs/specs/man/index.html)

## Chapter checklist

- [ ] Explain requires, exports, opens, transitive, and static.
- [ ] Distinguish named, automatic, and unnamed modules.
- [ ] Understand ServiceLoader directives and custom runtime images.
- [ ] Diagnose reflective access and split-package migration failures.
