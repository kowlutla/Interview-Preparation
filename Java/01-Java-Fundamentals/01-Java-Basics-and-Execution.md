# Java Basics and Execution

This chapter establishes the vocabulary used throughout the handbook: Java as a language and platform, source compilation, bytecode, class loading, linking, initialization, execution, and the roles of the JDK and JVM.

**Target level:** Beginner → Expert  
**Scope:** Java 8–21, with notes where behavior is release- or implementation-specific.

## 1. Java as a language and platform

### Q1. What is Java, and what does “platform-independent” mean? *(Beginner)*

Java is a statically typed, class-based programming language and a platform comprising standardized APIs, a class-file format, and a virtual-machine specification. Ordinary Java source is compiled into class-file bytecode; a compatible JVM can execute that bytecode on a different operating system or processor.

Platform independence does not mean every Java program runs everywhere without conditions. A program can depend on native libraries, operating-system behavior, file paths, locale, time-zone data, filesystem semantics, permissions, or APIs unavailable on a target runtime. Native Image binaries and JNI dependencies also have platform-specific constraints.

The precise claim is that Java's defined language and bytecode abstractions allow portability when the target runtime and environment satisfy the program's requirements.

### Q2. What are the JDK, JRE, and JVM? *(Beginner)*

- **JVM (Java Virtual Machine):** The abstract machine specified to load and execute class files, including bytecode verification and runtime execution semantics.
- **JRE (Java Runtime Environment):** The runtime components needed to run Java applications, traditionally the JVM plus standard libraries and supporting files.
- **JDK (Java Development Kit):** The development distribution, including the runtime and tools such as `javac`, `java`, `javadoc`, `jar`, `jdb`, and diagnostic utilities.

Since Java 9, the modular runtime-image model changed how distributions are assembled; a separately installed JRE is not universally provided as a distinct download. Do not treat “JRE is always a separate installation” as a timeless rule.

### Q3. Does the JVM execute Java source code directly? *(Beginner)*

Ordinary Java execution does not send `.java` source directly to the JVM. The compiler typically translates source files into class files. The class loader locates those class files or equivalent class definitions, the JVM links and initializes classes as needed, and execution begins at an appropriate entry point.

Tools such as JShell and source-file launch mode can make source execution appear direct, but they compile or otherwise process the source as part of the tool workflow. The JVM's specified execution unit is the class-file format, not Java source syntax.

### Q4. What does “write once, run anywhere” actually promise? *(Intermediate)*

It describes the portability goal enabled by compiling Java source to standardized bytecode and running it on a conforming JVM. The slogan is not a guarantee against environmental dependencies, implementation bugs, incompatible native code, unavailable modules, or differences in external systems.

For portable services, test on the actual target JDK distribution and operating system, avoid unnecessary native dependencies, use standard APIs, and make locale/time-zone assumptions explicit.

## 2. Source compilation and class files

### Q5. What happens when `javac` compiles a Java source file? *(Intermediate)*

At a high level, the compiler parses source, performs name resolution and type checking, applies language rules such as overload resolution, and emits class files containing bytecode and metadata. Depending on the source and build, compilation may also process annotations and generate additional source or class files.

Example:

```java
// Hello.java
public class Hello {
    public static void main(String[] args) {
        System.out.println("Hello, Java");
    }
}
```

Compile and inspect:

```bash
javac Hello.java
javap -c -v Hello
java Hello
```

`javap -c` displays bytecode instructions; `-v` includes more class-file metadata. Exact bytecode is a compiler output detail and may differ between compiler versions even when the program has the same observable behavior.

### Q6. What is Java bytecode? *(Beginner)*

Bytecode is the instruction set encoded in a Java class file. Instructions operate on the JVM's abstract execution model and include operations for constants, local variables, operand-stack manipulation, arithmetic, branches, method invocation, object creation, and exception handling.

Bytecode is not native machine code. The JVM may interpret it, compile hot methods or code regions to native machine code, or apply a combination of execution strategies. A compatible class file is not guaranteed to use identical native instructions across CPUs or JVM implementations.

### Q7. What is the difference between source code, bytecode and machine code? *(Beginner)*

- **Source code:** Human-authored language text such as `.java`.
- **Bytecode:** JVM class-file instructions and metadata produced by compilation.
- **Machine code:** Instructions executed by a physical processor.

A typical HotSpot execution path begins with class-file bytecode, often executes some code in an interpreter, then compiles sufficiently hot code to machine code using JIT compilation. This is a useful model, not a promise that every method is compiled in exactly the same way.

### Q8. What is contained in a `.class` file? *(Advanced)*

A class file contains a header (including a magic value and class-file version), a constant pool, access flags, class and superclass references, interfaces, fields, methods, attributes, and other metadata required by the JVM. Method bodies for ordinary concrete methods may contain a `Code` attribute with bytecode, exception-table entries, and supporting metadata.

The file does not simply contain a serialized copy of the original Java source. Debugging attributes such as line-number tables and local-variable tables may be present, but their inclusion depends on compiler options and tooling. Use `javap -v` to examine an actual class file.

### Q9. What is the class-file version, and why does it matter? *(Intermediate)*

The class-file version identifies the format and feature level a runtime must understand. A newer compiler may emit class files that an older runtime cannot load, often resulting in `UnsupportedClassVersionError`.

Select the release target deliberately. For example, compiling with a suitable JDK using `javac --release 17 ...` asks the compiler to target Java 17 language/API compatibility constraints (subject to the JDK's supported releases). Merely setting `-source` and `-target` does not by itself enforce use only of APIs present in that target release; `--release` is generally safer for cross-release builds.

### Q10. What is the difference between `-source`, `-target`, and `--release`? *(Advanced)*

- `-source` selects the language source level accepted by the compiler.
- `-target` selects the class-file target level.
- `--release N` coordinates the source level, target class-file version, and the documented Java SE/JDK API surface for release `N`.

Using only `-source` and `-target` can accidentally compile against newer APIs when the build JDK is newer than the deployment JDK. The result may compile successfully and then fail at runtime with a missing class or method. Prefer `--release` for normal cross-release compilation where supported.

## 3. The runtime pipeline

### Q11. What are the major stages between launching a Java application and executing its `main` method? *(Advanced)*

A practical simplified flow is:

1. The launcher starts the selected Java runtime and processes runtime options.
2. The initial class is located and its class definition is obtained by a class loader.
3. The JVM performs the applicable loading, verification, preparation, resolution, and initialization steps.
4. The runtime invokes the entry point using the launcher's rules.
5. Bytecode executes, often with runtime profiling and possible JIT compilation.
6. Other classes may be loaded and initialized lazily as execution requires them.

This is deliberately simplified: resolution can be lazy, class initialization occurs at specified active-use points, and the exact order of implementation activities should not be confused with an observable language guarantee.

### Q12. What is class loading? *(Intermediate)*

Class loading is the process of obtaining a binary representation of a class or interface and creating its runtime representation. A defining class loader establishes the class's runtime identity together with its binary name.

Two classes with the same binary name but different defining class loaders are distinct runtime types. This matters in application servers, plugin systems, test frameworks, hot-reload tools, and environments containing multiple versions of a dependency.

### Q13. What are loading, linking and initialization? *(Advanced)*

The JVM Specification describes these as distinct parts of class lifecycle processing:

- **Loading:** Find a binary representation and create the class or interface.
- **Linking:** Verify the class, prepare static storage with default values, and resolve symbolic references as required.
- **Initialization:** Execute class initialization code, including static field initializers and static initializer blocks in the specified order.

The precise rules and triggers are specified in the [JVM Specification, Chapter 5](https://docs.oracle.com/javase/specs/jvms/se21/html/jvms-5.html). It is inaccurate to claim that every symbolic reference must be resolved before any method can run; resolution may occur lazily.

### Q14. What does bytecode verification do? *(Advanced)*

Verification checks that a class file satisfies structural and type-safety constraints required by the JVM. Checks include valid instruction forms, control-flow consistency, operand-stack type consistency, and rules that help prevent invalid bytecode from performing forbidden operations.

Verification is not a substitute for application security review. A verified program can still contain authorization bugs, unsafe deserialization, injection vulnerabilities, denial-of-service flaws, or incorrect business logic.

### Q15. What are preparation and resolution? *(Advanced)*

During **preparation**, the JVM creates static storage and initializes static fields to their default values according to the specification. A non-constant static field initializer such as `static int count = 42;` is executed during initialization, not by assigning `42` during preparation.

**Resolution** turns symbolic references in the constant pool into concrete runtime references where and when resolution is performed. It can happen eagerly or lazily within the JVM Specification's constraints. Interview answers should distinguish these phases rather than describing all of them as “the class gets loaded.”

### Q16. When is a class initialized? *(Expert)*

Initialization is triggered by specified active uses, including creating an instance of a class, invoking a static method declared by it, assigning a static field declared by it, or reading a non-constant static field declared by it, subject to the detailed rules in the JLS. Initialization may also be triggered by certain reflective operations.

Reading a compile-time constant variable does not necessarily trigger initialization of the class that declares it, because the value may be embedded into the consuming class at compile time. Interface initialization rules have distinctions from class initialization rules. Refer to [JLS §12.4](https://docs.oracle.com/javase/specs/jls/se21/html/jls-12.html#jls-12.4) for normative rules.

### Q17. What happens if a static initializer throws an exception? *(Advanced)*

If class initialization completes abruptly because an exception escapes a static initializer, the class is marked erroneous for that class loader. The first active use typically reports an `ExceptionInInitializerError` when the thrown value is not already an `Error`; subsequent initialization attempts can fail with `NoClassDefFoundError` because the class could not be initialized.

This differs from “the class file cannot be found.” For diagnosis, inspect the first exception in the logs: later linkage errors may be consequences of the original initialization failure.

## 4. JIT and execution strategy

### Q18. What is the interpreter in a JVM? *(Intermediate)*

An interpreter executes bytecode instructions without requiring each method to have first been compiled to native machine code. Interpretation can reduce initial compilation cost and allows code to execute before sufficient profiling data exists for optimization.

Interpreter strategies differ across JVMs and releases. In HotSpot, interpreter execution also contributes profiling information used by optimizing compilers.

### Q19. What is Just-In-Time (JIT) compilation? *(Intermediate)*

JIT compilation translates bytecode or an intermediate representation into native machine code at runtime. HotSpot uses profiling and compilation tiers to optimize code that appears frequently executed, potentially applying inlining, devirtualization, escape analysis, loop optimizations and other transformations.

JIT is not guaranteed to make every Java method faster. Startup time, warm-up, code-cache pressure, workload variability, and optimization/deoptimization costs can affect results. Measure representative workloads rather than assuming compilation eliminates all overhead.

### Q20. What is method inlining, and why can it help? *(Advanced)*

Inlining substitutes a callee's body into the caller's compiled code when the compiler judges it profitable and valid. It can remove call overhead and expose more optimization opportunities, such as constant propagation and elimination of redundant checks.

Inlining is constrained by code size, call-site behavior, compiler heuristics, and runtime assumptions. It is an optimization decision rather than a language-level promise. Excessive inlining can also increase compiled-code size and instruction-cache pressure.

### Q21. What is deoptimization? *(Expert)*

A JVM may optimize compiled code using assumptions supported by profiling—for example, a call site may usually see one implementation. If an assumption later becomes invalid, the runtime can invalidate or deoptimize compiled code and resume execution in a less optimized form while preserving Java semantics.

Deoptimization explains why a hot service's latency may change after a new class is loaded or the observed workload changes. The key guarantee is correct program behavior, not stable optimization decisions.

## 5. Main method and launch behavior

### Q22. Why is `public static void main(String[] args)` used as a Java entry point? *(Beginner)*

It is the conventional launcher entry point for a class-based Java application:

- `public` makes it accessible to the launcher under the traditional entry-point convention.
- `static` allows invocation without first constructing an instance of the class.
- `void` indicates that the method returns no Java result value.
- `String[] args` receives command-line arguments as strings.

The Java language and Java launchers evolve; newer releases support additional launch forms, including source-file launch and compact source arrangements in specific releases. For conventional Java 8–21 interviews, explain the traditional signature first, then note release-specific launch features precisely.

### Q23. Is `String... args` valid for `main`? *(Intermediate)*

A varargs parameter is compiled as an array parameter, so `public static void main(String... args)` has the same parameter type as `String[]` at the bytecode level. Whether a particular launcher's entry-point rules accept a signature depends on the applicable launcher specification/version; verify against the exact target JDK rather than extrapolating from source-level equivalence alone.

For maximum compatibility across traditional launchers, use `public static void main(String[] args)`.

### Q24. Can a Java program have multiple `main` methods? *(Intermediate)*

A class can overload `main`, and different classes can each declare a `main` method. The traditional launcher chooses the configured launch class and a matching supported entry point; it does not use ordinary Java overload resolution based on command-line argument contents.

```java
public class EntryPointDemo {
    public static void main(String[] args) {
        System.out.println("launcher entry");
        main(7);
    }

    public static void main(int value) {
        System.out.println("overload: " + value);
    }
}
```

Traditional output:

```text
launcher entry
overload: 7
```

The overloaded `main(int)` is an ordinary static method unless explicitly called.

## 6. Compilation and diagnostics commands

### Q25. Which commands are useful for inspecting the Java toolchain and a running process? *(Intermediate)*

Common tools include:

```bash
java -version
javac -version
java --help
javac --help
javap -c -v SomeClass
jcmd <pid> VM.version
jcmd <pid> VM.command_line
jcmd <pid> Thread.print
jcmd <pid> GC.heap_info
jstack <pid>
jmap -histo <pid>
```

Availability, permissions, output formats, and supported diagnostic commands vary by JDK distribution, operating system, and target process. Some operations can impose meaningful overhead. Consult the tools shipped with the exact runtime and test diagnostic procedures before relying on them during an incident.

### Q26. Why can `java -version` and `javac -version` report different versions? *(Intermediate)*

The executable resolved through `PATH` for `java` may come from a different installation than the `javac` executable. `JAVA_HOME`, shell configuration, IDE settings, build-tool toolchains, container images, and CI configuration can each select different JDKs.

Troubleshoot the actual executable paths (for example, `which java`, `which javac`, or platform-equivalent commands), inspect environment variables, and check Maven/Gradle toolchain settings. A build that succeeds locally can still emit bytecode or use APIs inconsistent with the production runtime if toolchains are not controlled.

### Q27. Why does a program compile but fail with `NoSuchMethodError` at runtime? *(Advanced)*

`NoSuchMethodError` is a linkage error: compiled code refers to a method that is not available with the expected name and descriptor in the runtime class actually loaded. Common causes are dependency-version conflicts, a runtime classpath different from the compile classpath, binary-incompatible library upgrades, shaded dependencies, and class-loader isolation.

Investigate the loaded class's origin and version, dependency resolution output, classpath/module path, and the exact method descriptor. Fix the dependency alignment or compatibility issue; recompiling blindly may only hide the mismatch in one environment.

### Q28. What is the difference between `ClassNotFoundException` and `NoClassDefFoundError`? *(Advanced)*

`ClassNotFoundException` is a checked exception commonly thrown when code explicitly requests a class by name through APIs such as `Class.forName` or a class loader and the requested definition cannot be found.

`NoClassDefFoundError` is a `LinkageError`, often reported when ordinary execution needs a class definition that is unavailable, or when a class previously failed initialization. The exact cause matters: inspect the nested/root exception and the class loader involved before treating the two as interchangeable.

## 7. Output prediction and debugging fundamentals

### Q29. What is the output of this example, and why is it not a reliable way to infer source-level execution order? *(Advanced)*

```java
class InitializationOrder {
    static {
        System.out.println("static block");
    }

    public static void main(String[] args) {
        System.out.println("main");
    }
}
```

For a normal launch of this class, the output is:

```text
static block
main
```

The class must be initialized before its static `main` invocation proceeds. This example demonstrates a specified lifecycle relationship. It does not imply that all linking or all possible resolution work for every referenced class has been completed before the first printed line.

### Q30. Why can a `static final` field behave differently depending on how it is initialized? *(Advanced)*

A `static final` primitive or `String` initialized with a compile-time constant expression is a *constant variable* under JLS rules. Client bytecode can embed that constant, so reading it may not initialize the declaring class and may preserve the old value in a client recompiled only partially after the constant changes.

```java
class Limits {
    static final int MAX = 100;             // constant variable
    static final int COMPUTED = loadLimit(); // not a constant expression
    static int loadLimit() { return 200; }
}
```

`MAX` and `COMPUTED` are both `static final`, but only `MAX` is a constant variable. See [JLS §4.12.4](https://docs.oracle.com/javase/specs/jls/se21/html/jls-4.html#jls-4.12.4) and [JLS §12.4.1](https://docs.oracle.com/javase/specs/jls/se21/html/jls-12.html#jls-12.4.1).

### Q31. If two classes have the same fully qualified name, are they necessarily the same Java type? *(Expert)*

No. A runtime class is identified by its binary name and defining class loader. Two separate defining class loaders may define different types with the same binary name. This can produce confusing `ClassCastException` messages that appear to say a class cannot be cast to itself.

When diagnosing this, compare both the class name and the defining class loader, and find where each class was loaded from. Avoid diagnosing purely by package/class name.

### Q32. Why is “Java is interpreted” or “Java is compiled” an incomplete answer? *(Intermediate)*

Java uses compilation at more than one level. `javac` normally compiles source to bytecode before execution, while the runtime may interpret bytecode and JIT-compile hot code to native instructions. Alternative JVMs and runtime configurations can use different strategies.

A robust answer distinguishes **source compilation**, **bytecode execution**, and **runtime compilation/optimization** instead of forcing the platform into a single label.

### Q33. What is the difference between a specification guarantee and a HotSpot implementation detail? *(Expert)*

A specification guarantee is part of the defined contract—such as language conversion rules or class initialization semantics. An implementation detail is a choice made by a particular runtime, such as specific object-header layouts, exact garbage-collector heuristics, compilation thresholds, or a particular `HashMap` node representation.

Implementation details can still matter for performance investigations, but label the runtime and release when explaining them. Do not turn a measurement from one build into a universal Java guarantee.

## 8. Version and build compatibility

### Q34. If code uses a Java 21 language feature, can it run on Java 17? *(Intermediate)*

Not as compiled Java 21 code on a Java 17 runtime if its class-file version or required APIs/features exceed Java 17's capabilities. Java 17 does not understand language syntax that was introduced later, and it cannot execute unsupported class-file versions.

If the application must run on Java 17, compile and test against Java 17 constraints, avoid newer APIs, and avoid syntax not accepted at that source level. Runtime compatibility should be validated in CI on the actual target JDK.

### Q35. What should a build pipeline record to make Java behavior reproducible? *(Senior)*

At minimum, record and control:

- JDK vendor and exact version/update used for compilation and tests.
- Target release (`--release` or build-tool equivalent).
- Dependency graph and lock/version policy where supported.
- Runtime image/container base, JVM options and enabled modules.
- Test operating systems and relevant environment assumptions.
- Benchmark workload/configuration when performance is compared.

Reproducibility rarely comes from pinning only a major version. Security fixes and vendor builds matter, so use a managed update policy that permits patched releases while preserving traceability.

## Key takeaways

- Java source is generally compiled to class files; a JVM executes the class-file model and may use interpretation plus JIT compilation.
- The JDK is the development kit; the JVM is the virtual machine; a separate JRE package is not guaranteed for every modern distribution.
- Loading, linking and initialization are distinct lifecycle concepts with specified triggers and failure modes.
- Bytecode compatibility and API compatibility are related but separate. Use `--release` for normal cross-release compilation.
- The runtime's implementation choices must not be confused with language/JVM specification guarantees.
- Diagnose the runtime actually launched, not merely the JDK selected by an IDE.

## Official references

1. [JDK 21 Documentation](https://docs.oracle.com/en/java/javase/21/)
2. [Java Language Specification, Java SE 21](https://docs.oracle.com/javase/specs/jls/se21/html/index.html)
3. [JLS §12 — Execution](https://docs.oracle.com/javase/specs/jls/se21/html/jls-12.html)
4. [Java Virtual Machine Specification, Java SE 21](https://docs.oracle.com/javase/specs/jvms/se21/html/index.html)
5. [JVMS §4 — The class-file format](https://docs.oracle.com/javase/specs/jvms/se21/html/jvms-4.html)
6. [JVMS §5 — Loading, Linking, and Initializing](https://docs.oracle.com/javase/specs/jvms/se21/html/jvms-5.html)
7. [JDK 21 Java language changes](https://docs.oracle.com/en/java/javase/21/language/java-language-changes-release.html)
8. [JEP index — OpenJDK](https://openjdk.org/jeps/0)

## Chapter checklist

- [ ] Explain JDK/JRE/JVM without relying on obsolete packaging assumptions.
- [ ] Compile a source file, inspect the class file, and explain the output.
- [ ] Explain loading, linking, initialization, verification, and initialization failure.
- [ ] Diagnose target-version mismatch and runtime linkage errors.
- [ ] Distinguish specification-level guarantees from HotSpot-specific behavior.
