# Java 9–11 Features

**Target level:** Intermediate → Senior  
**Releases:** Java 9, 10, 11 LTS

Java 9–11 brought module system changes, collection factory methods, private interface methods, local-variable type inference, new String APIs, HTTP Client, and other library/runtime improvements. This chapter focuses on compatibility and practical impact.

## 1. Java 9

### Q1. What is JPMS in Java 9? *(Intermediate)*

The Java Platform Module System introduced named modules with explicit dependencies and package export/open boundaries. It improved encapsulation and enabled tools such as jlink to build custom runtime images.

JPMS adds a layer beyond packages and classpath visibility. Migrating legacy systems requires checking reflection, service loading, split packages, and third-party dependency behavior.

### Q2. What are List.of, Set.of, and Map.of? *(Intermediate)*

Java 9 introduced factory methods for concise unmodifiable collections. They reject nulls, and Set.of/Map.of reject duplicate set elements or keys according to their contracts. Their iteration order should not be treated as a stable API guarantee unless documented.

They return unmodifiable collections, not deeply immutable object graphs. Use mutable copies when modification is required.

### Q3. What are private interface methods? *(Advanced)*

Java 9 added private methods to interfaces so default and static methods can share implementation without exposing helper methods as public API. They do not become abstract methods that implementing classes must implement.

Use them to reduce duplication inside an interface while keeping the public contract focused.

### Q4. What is the Process API improvement in Java 9? *(Advanced)*

Java 9 expanded Process with methods for process IDs, process information, descendants, and more controlled process handling. It improved observability and interaction with operating-system processes.

Process management still requires timeouts, output-stream handling, exit-code checks, and safe command construction. Do not build shell commands from untrusted input.

### Q5. What is the Java 9 REPL? *(Beginner)*

JShell provides an interactive environment for evaluating Java declarations and expressions without creating a full project. It is useful for experimenting with APIs and language behavior.

JShell is a learning and exploration tool, not a replacement for unit tests, builds, or production diagnostics.

## 2. Java 10

### Q6. What is var in Java 10? *(Intermediate)*

Local-variable type inference allows var in local variable declarations where the compiler can infer a concrete type from the initializer. It does not make Java dynamically typed and is not allowed for fields or method parameters in the same way.

Use var when the initializer makes the type clear and it improves readability. Avoid it when it obscures an important abstraction or numeric type.

### Q7. Can var be initialized with null? *(Intermediate)*

No. A declaration such as var value = null has no type from which to infer the variable's type and is rejected. var requires an initializer whose type can be inferred.

Use an explicit reference type when the initial value is null, or initialize it with a value that communicates the intended type.

### Q8. What does copyOf add to collections in Java 10? *(Intermediate)*

Collection factory APIs such as List.copyOf create unmodifiable snapshots of the input's elements and reject null elements. The collection structure is copied or reused according to implementation optimization, but the API contract is an unmodifiable list.

The copy is shallow: mutable element objects can still change. Use deep copying if the domain requires independent nested state.

## 3. Java 11

### Q9. Why is Java 11 important? *(Beginner)*

Java 11 is an LTS release that added APIs such as String.isBlank, strip, lines, repeat, Files.readString/writeString, and the standardized HTTP Client. It also continued platform and runtime evolution after Java 9 and 10.

LTS support depends on the vendor and distribution. Verify the organization's patch policy rather than assuming every Java 11 build has identical support.

### Q10. What did String gain in Java 11? *(Intermediate)*

String gained isBlank, strip/stripLeading/stripTrailing, lines, and repeat. These methods improve common text operations and use defined whitespace/line semantics.

They are not available on Java 8. When maintaining cross-version libraries, compile against the intended --release baseline or provide compatible alternatives.

### Q11. What is the standardized HTTP Client? *(Intermediate)*

Java 11 standardized java.net.http.HttpClient, which supports synchronous and asynchronous requests, HTTP/2, and WebSocket APIs. It provides a modern alternative to older URLConnection-based code for many use cases.

Production clients still need connection/time limits, cancellation, TLS configuration, retry policy, and observability. Async does not eliminate downstream limits or the need to handle failures.

### Q12. What are Files.readString and writeString? *(Intermediate)*

Java 11 added convenient methods for reading and writing whole text files. They are appropriate for small or bounded files where holding the entire content in memory is acceptable.

For large or untrusted inputs, stream data and enforce limits. Specify the charset explicitly when the file format requires a particular encoding.

### Q13. How do you migrate a Java 8 application to Java 11 safely? *(Senior)*

Compile and test with the intended runtime baseline, update dependencies, inspect removed or encapsulated internals, validate TLS and HTTP behavior, and check build-tool compatibility. Run performance tests and monitor GC, startup, and memory changes.

Do not assume that compilation on Java 11 proves the application will work correctly under production traffic. Include packaging, container, monitoring-agent, and integration tests.

## Official references

1. [JDK 9 Documentation](https://docs.oracle.com/en/java/javase/9/)
2. [JDK 10 Documentation](https://docs.oracle.com/en/java/javase/10/)
3. [JDK 11 Documentation](https://docs.oracle.com/en/java/javase/11/)
4. [JEP 286 — Local-Variable Type Inference](https://openjdk.org/jeps/286)
5. [JEP 321 — HTTP Client](https://openjdk.org/jeps/321)
6. [Java SE 11 API — String](https://docs.oracle.com/en/java/javase/11/docs/api/java.base/java/lang/String.html)

## Chapter checklist

- [ ] Explain modules and collection factory methods.
- [ ] Use var only when type inference improves readability.
- [ ] Know Java 11 String, Files, and HTTP Client additions.
- [ ] Validate Java 8-to-11 migration beyond compilation.
