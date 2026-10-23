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
