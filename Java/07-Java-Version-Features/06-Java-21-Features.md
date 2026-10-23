# Java 21 Features

**Target level:** Advanced → Senior  
**Release:** Java 21 LTS  
**Feature status:** Virtual threads, record patterns, and pattern matching for switch are final in Java 21. Structured concurrency and scoped values were preview features in Java 21.

Java 21 is a major modern Java baseline. Interview answers should distinguish permanent features from preview/incubator APIs and explain what each feature changes in application design.

## 1. Final language and runtime features

### Q1. What are the most important Java 21 features? *(Intermediate)*

Major finalized features include virtual threads (JEP 444), record patterns (JEP 440), pattern matching for switch (JEP 441), sequenced collections (JEP 431), and generational ZGC (JEP 439). Structured concurrency and scoped values were preview features in Java 21.

Version status matters: preview features require explicit flags and are not equivalent to permanent APIs. Confirm the target runtime and feature status before adopting them.

### Q2. What are virtual threads? *(Intermediate)*

Virtual threads are lightweight Java threads scheduled by the runtime, designed to make thread-per-task concurrency practical for high-concurrency workloads that spend substantial time waiting. They became final in Java 21.

They do not make CPU-bound work faster or remove downstream limits. Bound database calls, external requests, queues, and other scarce resources even when virtual threads are cheap.

### Q3. What is pattern matching for switch? *(Intermediate)*

Pattern matching for switch became final in Java 21. It allows switch cases to match types and patterns and supports guarded patterns and null handling under defined rules. It can reduce long chains of instanceof checks and casts.

Combine it with sealed hierarchies for controlled variant handling. Still verify exhaustiveness, null semantics, and compatibility with the exact source release.

### Q4. What are record patterns? *(Intermediate)*

Record patterns became final in Java 21. They allow a record's components to be matched and destructured in a pattern, often together with instanceof or switch pattern matching.

They improve readability for nested data models but do not eliminate validation or null handling. The compiler's type and flow rules determine which variables are available in each branch.

### Q5. What are sequenced collections? *(Intermediate)*

Sequenced collections were added in Java 21 through JEP 431. They provide a common API for collections with a defined encounter order, including operations such as first/last element access and reversed views where supported.

They unify APIs for ordered collections but do not make unordered collections ordered by magic. Understand the contract of the specific implementation and whether a reversed result is a view or independent copy.

### Q6. What is generational ZGC? *(Advanced)*

Generational ZGC became available in Java 21 through JEP 439, adding generational collection to ZGC's low-latency design. Generational collection exploits differences between young and old object lifetimes.

Collector performance is workload-dependent. Benchmark allocation rate, live set, CPU, memory overhead, and pause distributions on the exact JDK build.

## 2. Preview features in Java 21

### Q7. What was the status of Structured Concurrency in Java 21? *(Advanced)*

Structured Concurrency was a preview API in Java 21 under JEP 453. It aimed to treat related concurrent tasks as one unit of work, improving cancellation, error handling, and observability. It was not a final Java 21 API.

Preview APIs evolve across releases, so code should not assume the Java 21 preview shape matches a later release. Use only with explicit versioning and a clear evaluation policy.

### Q8. What was the status of Scoped Values in Java 21? *(Advanced)*

Scoped Values were a preview feature in Java 21 under JEP 446. They were designed to share immutable context through a bounded dynamic scope, often as an alternative to some ThreadLocal use cases.

They were not final in Java 21. Review the JDK version and later JEPs before using them, especially in long-lived frameworks where API evolution and context propagation matter.

### Q9. What does preview mean operationally? *(Advanced)*

Preview features generally require enabling preview at compile time and runtime for the corresponding JDK release. They may change or be removed in later releases, so build configuration and deployment must use compatible JDKs.

Document preview flags and isolate preview-dependent code. Avoid silently relying on preview features in libraries whose consumers may not run the same JDK.

## 3. Migration and practical impact

### Q10. How can virtual threads simplify backend code? *(Senior)*

They can support a straightforward thread-per-request or thread-per-task model for blocking I/O, reducing the need to express every operation as callbacks solely to conserve platform threads. This may simplify debugging and stack traces.

The surrounding resource limits remain. A virtual-thread-per-task executor should not be mistaken for a bounded concurrency policy; use semaphores, connection pools, rate limits, and deadlines.

### Q11. How do you choose Java 21 features for a production service? *(Senior)*

Start from a supported JDK baseline, then identify which finalized features solve a real problem. Evaluate virtual threads against blocking I/O patterns, records/pattern matching against domain modeling, and sequenced collections against ordering needs. Treat preview APIs as a separate adoption decision.

Run compatibility, load, security, memory, and observability tests. A new language feature is valuable when it improves clarity or system behavior, not merely because it is new.

### Q12. What is a sensible Java 17-to-21 migration plan? *(Senior)*

Update the build toolchain and dependencies, remove deprecated/internal API usage, test language/API compatibility, validate containers and observability agents, and run production-like benchmarks. Adopt finalized features incrementally after the runtime migration is stable.

Keep preview-feature evaluation separate from the baseline migration. Record JDK vendor/build, flags, and rollback criteria.

## Official references

1. [JDK 21 Documentation](https://docs.oracle.com/en/java/javase/21/)
2. [JEP 444 — Virtual Threads](https://openjdk.org/jeps/444)
3. [JEP 440 — Record Patterns](https://openjdk.org/jeps/440)
4. [JEP 441 — Pattern Matching for switch](https://openjdk.org/jeps/441)
5. [JEP 431 — Sequenced Collections](https://openjdk.org/jeps/431)
6. [JEP 439 — Generational ZGC](https://openjdk.org/jeps/439)
7. [JEP 453 — Structured Concurrency (Preview in Java 21)](https://openjdk.org/jeps/453)
8. [JEP 446 — Scoped Values (Preview in Java 21)](https://openjdk.org/jeps/446)

## Chapter checklist

- [ ] Identify finalized Java 21 features and their use cases.
- [ ] Distinguish preview APIs from permanent features.
- [ ] Understand virtual threads' resource and cancellation limits.
- [ ] Evaluate migration with compatibility, performance, and operations tests.


## 4. Other Java 21 JEPs and API status

### Q13. What was the status of String Templates in Java 21? *(Advanced)*

String Templates were a preview feature in Java 21 under JEP 430, not a finalized Java SE feature. They aimed to combine literal text with embedded expressions and a template processor. Do not assume Java 21 preview syntax remains unchanged or is available without preview flags.

Because template processors can affect how embedded values are interpreted, security depends on the processor and target language. A template is not automatically safe SQL, HTML, shell, or URL construction.

### Q14. What was the status of the Foreign Function and Memory API in Java 21? *(Advanced)*

The Foreign Function and Memory API was in its third preview in Java 21 under JEP 442. It provides APIs for calling native functions and accessing native memory with more structured lifetime management than ad hoc JNI code. It was not yet final in Java 21.

Preview APIs can change across releases. Native interoperation still requires careful lifetime, bounds, layout, and ABI reasoning; a safer API does not make incorrect native-memory access harmless.

### Q15. What were unnamed patterns and variables in Java 21? *(Intermediate)*

Unnamed patterns and variables were preview features in Java 21 under JEP 443. They allow a program to indicate that a matched component or local variable is intentionally unused, improving readability when only part of a value matters.

They were not final Java 21 syntax. State the target release and preview status whenever discussing them, and do not confuse a Java 21 preview with a permanent language guarantee.

### Q16. What were unnamed classes and instance main methods in Java 21? *(Intermediate)*

JEP 445 previewed unnamed classes and instance main methods in Java 21 to make simple programs easier to begin writing. This was a preview language feature, not the only or universal Java entry-point form.

For production code and interviews, distinguish the traditional declared class with a public static main method from preview source-file/instance-main forms. Use the syntax accepted by the target JDK and its preview configuration.

### Q17. What was the status of the Vector API in Java 21? *(Advanced)*

The Vector API was an incubator API in Java 21 under JEP 448. It aims to express vector computations that can map to supported hardware vector instructions. Incubator APIs are distinct from finalized Java SE APIs and may change or be removed.

Vectorization depends on hardware, workload, and JDK implementation. Measure against scalar alternatives and keep incubator module requirements explicit.

### Q18. What is the Key Encapsulation Mechanism API added in Java 21? *(Advanced)*

JEP 452 delivered the Key Encapsulation Mechanism API in Java 21. A KEM allows a sender and recipient to establish shared secret material using a public-key mechanism; that material is then typically used with symmetric cryptography. It is not itself a complete secure communication protocol.

Use established protocols and vetted providers rather than designing a custom cryptographic exchange. Understand the threat model, key validation, algorithm selection, and lifecycle.

### Q19. What changed around dynamic loading of Java agents in Java 21? *(Advanced)*

JEP 451 prepared to disallow dynamic loading of agents by issuing warnings in the relevant dynamic-attach scenarios, while preserving startup-time agent loading. This is a migration and operational concern for profilers, monitoring tools, test frameworks, and instrumentation agents.

Check the exact JDK release behavior and the tool vendor's compatibility guidance. Do not suppress warnings blindly; determine whether the agent is required and whether startup-time configuration is supported.

### Q20. Why should a Java release summary distinguish feature JEPs from deprecation and platform changes? *(Senior)*

A release contains more than new language syntax. It can include API additions, runtime behavior changes, incubator/preview work, deprecations, and platform-port changes. These have different compatibility and operational implications.

For Java 21, the Windows 32-bit x86 port was deprecated for removal under JEP 449, and JEP 451 prepared for restrictions around dynamic agent loading. A complete migration review checks release notes and all relevant JEPs, not only headline language features.

