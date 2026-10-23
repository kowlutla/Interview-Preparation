# Enums and Annotations

**Target level:** Intermediate → Senior  
**Scope:** Java 8–21

Enums model finite alternatives with type safety; annotations attach metadata that compilers, tools, frameworks, or runtimes can interpret. Both are common in Java frameworks, serialization, validation, testing, dependency injection, and API design.

## 1. Enums

### Q1. Why use an enum instead of String constants? *(Beginner)*

An enum defines a finite set of named instances under one type, allowing the compiler to reject unrelated values. String constants can contain typos, unknown values, and inconsistent casing unless validated manually.

Enums can also carry methods and fields. For external protocols or persistence, however, do not rely on ordinal position; use an explicit stable code or a deliberate serialized name.

### Q2. Can enum constants have different behavior? *(Intermediate)*

Yes. An enum may implement an interface, and individual constants can override methods using constant-specific class bodies. This can express a finite strategy set without a large switch.

    enum Operation {
        ADD { int apply(int a, int b) { return a + b; } },
        MULTIPLY { int apply(int a, int b) { return a * b; } };
        abstract int apply(int a, int b);
    }

Use this when the behavior truly belongs to each constant. If behavior needs runtime configuration, external plugins, or independent lifecycle, injected strategy objects may be more suitable.

### Q3. What is the difference between name() and toString() on an enum? *(Intermediate)*

name() returns the exact identifier declared in source and is final. toString() normally returns the name but can be overridden for a friendlier representation. Persisting toString is fragile if the display format changes.

For durable data, define an explicit stable code and conversion policy. Enum.name() is more stable than a presentation-oriented toString but renaming a constant can still break stored values or wire compatibility.

### Q4. Why should ordinal() rarely be persisted? *(Intermediate)*

ordinal() is the declaration position starting at zero. Reordering constants or inserting one in the middle changes ordinals, so stored values may map to a different meaning after deployment.

Use an explicit code such as a string or integer that is assigned intentionally and remains stable. Define behavior for unknown future codes, particularly when services deploy at different times.

### Q5. Are enum instances thread-safe singletons? *(Advanced)*

Each enum constant is created as part of the enum class initialization, and the constant references are safely published under Java class-initialization guarantees. However, an enum singleton is only as thread-safe as its mutable fields and methods. Shared mutable state inside an enum still requires synchronization or immutability.

An enum is useful for a singleton with no complex initialization requirements. Do not treat the enum keyword as a general concurrency mechanism.

## 2. Annotation fundamentals

### Q6. What is an annotation? *(Beginner)*

An annotation is metadata attached to declarations, types, parameters, or other supported program elements. The Java compiler, annotation processors, reflection-based frameworks, and build tools may interpret that metadata.

An annotation does not automatically execute behavior. Its effect depends on a tool or runtime component that reads it. For example, @Override is checked by the compiler, while a framework annotation may be processed at runtime or during compilation.

### Q7. What are @Override, @Deprecated, and @SuppressWarnings used for? *(Beginner)*

@Override asks the compiler to verify that a method actually overrides an inherited method. @Deprecated communicates that an API should no longer be used and can carry since/forRemoval metadata in modern Java. @SuppressWarnings suppresses specified compiler warnings in a scope.

Suppress warnings narrowly and document the safety invariant. A suppression does not make unsafe code safe; it merely changes diagnostics.

### Q8. What are annotation retention policies? *(Intermediate)*

RetentionPolicy.SOURCE means the annotation is discarded by the compiler; CLASS means it is recorded in the class file but not necessarily retained for runtime reflection; RUNTIME makes it available to reflection. If no @Retention is specified, the default is CLASS.

Choose the least retention needed. Runtime reflection can have startup and metadata costs, while source-only annotations suit compile-time tools.

### Q9. What is @Target? *(Intermediate)*

@Target restricts the program elements where an annotation may be used, such as types, methods, fields, parameters, constructors, or type uses. It makes annotation intent explicit and lets the compiler reject misuse.

@Target({ElementType.TYPE, ElementType.METHOD}) is appropriate for an annotation designed for classes and methods. Type-use annotations can annotate a type occurrence, not just the declaration that contains it.

### Q10. How do you define a custom annotation? *(Intermediate)*

Declare an annotation interface using @interface and optionally specify retention, targets, and elements.

    @Retention(RetentionPolicy.RUNTIME)
    @Target(ElementType.METHOD)
    public @interface Audited {
        String action();
    }

The annotation only carries metadata. A runtime interceptor or processor must interpret it and implement the auditing behavior. Keep annotation elements simple and avoid turning annotations into a complicated configuration language.

### Q11. What types are allowed as annotation elements? *(Advanced)*

Annotation elements can use primitive types, String, Class and certain parameterized Class forms, enums, annotation types, and single-dimensional arrays of permitted element types. They cannot use arbitrary objects such as List or Map as annotation element values.

This constraint keeps annotation metadata representable in class files. If configuration is large or dynamic, use an external configuration object or file rather than encoding everything into annotation members.

### Q12. What is a repeatable annotation? *(Intermediate)*

A repeatable annotation can be applied more than once to the same declaration when declared with @Repeatable and a container annotation. Reflection APIs distinguish direct annotations from inherited or container-associated annotations, so choose the appropriate lookup method.

Repeatable annotations are useful for repeated declarative metadata, such as several aliases or rules. Ensure that repeated values have a clear order and conflict policy.

### Q13. What is an annotation processor? *(Advanced)*

An annotation processor runs during compilation and can inspect annotated elements, validate declarations, and generate source files or other outputs. It is commonly used for metamodels, dependency injection code, serializers, and compile-time checks.

Compile-time processing can catch errors before runtime, but processors need to be compatible with the build tool and JDK. Generated code should be deterministic, debuggable, and included in the normal build lifecycle rather than manually edited.

### Q14. What is the difference between runtime reflection and compile-time annotation processing? *(Advanced)*

Runtime reflection inspects metadata and types while the application runs; annotation processing examines source-level model elements during compilation and may generate code. Runtime processing can support dynamic environments but adds runtime coupling and can be affected by module access rules. Compile-time processing can fail the build early and avoid some reflection overhead.

Frameworks may combine both approaches. Understand when the framework reads the annotation and what is generated, proxied, or invoked as a result.

## 3. Framework and production concerns

### Q15. Why can annotation-driven frameworks behave differently from plain Java calls? *(Senior)*

Frameworks may inspect annotations, generate proxies, register components, or apply interceptors. A method annotated for transactions, caching, or authorization may only receive that behavior when invoked through the framework's proxy or managed lifecycle.

Read the framework's semantics rather than assuming an annotation changes Java language behavior. Self-invocation, private methods, final methods, proxy type, and configuration can affect interception.

### Q16. How can annotations create compatibility issues? *(Senior)*

Removing or changing annotation targets, retention, elements, defaults, or interpretation can break source consumers or framework behavior. A runtime framework may depend on an annotation being present in bytecode or accessible through reflection.

Treat public annotations as API. Define defaults carefully, document versioning, and test both compile-time consumers and runtime reflection where relevant.

### Q17. What is annotation inheritance? *(Advanced)*

@Inherited applies only to class annotations and affects lookup through class inheritance when a subclass lacks a directly declared annotation. It does not automatically apply to method, field, parameter, or interface annotations.

Reflection methods also differ: getAnnotation may return an inherited annotation, while getDeclaredAnnotation examines the element itself. Use the API that matches the intended semantics rather than assuming all annotations are inherited.

### Q18. How should annotations be used in secure system design? *(Senior)*

An annotation is metadata, not an enforcement mechanism by itself. A security annotation such as @RequiresRole is effective only if every relevant invocation passes through the component that enforces it. Direct calls, misconfigured proxies, alternate endpoints, or reflection can bypass assumptions.

Test negative authorization cases at actual boundaries. Use defense in depth and verify enforcement in the framework configuration rather than treating the presence of an annotation as proof that access is controlled.

## Official references

1. [Java Language Specification — Enums](https://docs.oracle.com/javase/specs/jls/se21/html/jls-8.html)
2. [Java Language Specification — Annotations](https://docs.oracle.com/javase/specs/jls/se21/html/jls-9.html)
3. [Java SE 21 API — RetentionPolicy](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/lang/annotation/RetentionPolicy.html)
4. [Java SE 21 API — ElementType](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/lang/annotation/ElementType.html)
5. [Java SE 21 API — AnnotatedElement](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/lang/reflect/AnnotatedElement.html)

## Chapter checklist

- [ ] Model finite values with enums and stable external codes.
- [ ] Explain annotation targets, retention, repeatability, and inheritance.
- [ ] Distinguish compile-time annotation processing from runtime reflection.
- [ ] Understand framework proxy/interceptor semantics and annotation limitations.
- [ ] Treat public annotations as versioned API contracts.
