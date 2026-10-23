# Java 18–20 Features

**Target level:** Advanced → Senior  
**Releases:** Java 18, 19, 20  
**Feature status:** Several headline features in this period were preview or incubator, not permanent.

Java 18–20 continued language and runtime evolution on the path to Java 21. The important interview skill is distinguishing finalized APIs from features that required preview/incubator flags and understanding how teams should manage that compatibility risk.

## 1. Java 18

### Q1. What are notable Java 18 changes? *(Intermediate)*

Java 18 finalized UTF-8 as the default charset for standard Java APIs in most cases under JEP 400, added a simple web server tool, and included other runtime/tooling changes. Existing code that relied on the operating system's default charset may behave differently.

Still specify the charset explicitly for protocols and files whose encoding is part of the contract. A standardized default reduces variability but should not replace explicit format requirements.

### Q2. What did JEP 400 change? *(Advanced)*

JEP 400 made UTF-8 the default charset for standard Java APIs in Java 18, with specified exceptions and compatibility considerations. This improves cross-platform consistency for many text operations.

Audit code that reads/writes text without an explicit charset. Correctness should be tied to the data format, not to the host's default configuration.

## 2. Java 19 and 20 preview/incubator topics

### Q3. What was the status of virtual threads in Java 19 and 20? *(Advanced)*

Virtual threads were preview features in Java 19 and Java 20, then became permanent in Java 21 under JEP 444. Early preview APIs and behavior should not be assumed identical to the final Java 21 API.

For Java 21 production code, use the finalized API and document the runtime baseline. For older previews, the exact flags and API shape depend on the release.

### Q4. What was the status of Structured Concurrency in Java 19 and 20? *(Advanced)*

Structured Concurrency was an incubator API in Java 19 and Java 20, then became a preview feature in Java 21 under JEP 453. It was not a permanent Java 19/20 feature. The status category matters: incubator and preview are distinct mechanisms with different release and API expectations.

Preview APIs may change between releases, so do not build a stable public library contract around one preview iteration without an explicit migration plan.

### Q5. What was the status of record patterns in Java 19 and 20? *(Advanced)*

Record patterns were preview features in Java 19 and 20 and became final in Java 21. They support destructuring record components in pattern matching.

Always tie syntax and feature status to a release. Code written against a preview version may require changes when moving to the finalized form.

### Q6. What was the status of pattern matching for switch in Java 19 and 20? *(Advanced)*

Pattern matching for switch remained a preview feature across releases before becoming final in Java 21. Its syntax and semantics evolved through multiple previews.

For compatibility discussions, state the exact JDK release and preview flags. Do not describe Java 19 or Java 20 switch pattern matching as a permanent feature.

### Q7. What was the Vector API status in Java 19 and 20? *(Advanced)*

The Vector API remained an incubator API across these releases. Incubator APIs are experimental platform modules that can evolve and are not part of the permanent Java SE API contract.

Use only when the target runtime and build support the incubator module and the performance benefit is demonstrated. Ordinary Java code remains the baseline for portability.

## 3. Migration practices

### Q8. Why should preview features be isolated? *(Senior)*

Preview features require release-specific flags and can change or disappear. Isolating them behind a small module or internal boundary limits migration cost and prevents a preview dependency from spreading through public APIs.

Document compiler/runtime flags and test the next intended JDK early. Avoid exporting preview types from a widely consumed library unless consumers explicitly accept that dependency.

### Q9. How do you maintain a Java version feature matrix? *(Senior)*

For each feature, record the first preview/incubator release, finalization release if any, API/package, required flags, runtime baseline, and compatibility concerns. Verify each entry against JEPs and release documentation.

This prevents statements such as “available since Java 19” from obscuring that a feature was only previewed there and became permanent later.

### Q10. How should a team evaluate a JDK upgrade across these releases? *(Senior)*

Update the build toolchain, verify dependencies and agents, compile/test at the target release, and run compatibility and load tests. Pay particular attention to charset assumptions, module encapsulation, GC/runtime behavior, and preview APIs.

Roll out incrementally with observability and rollback criteria. Record the vendor/build and JVM options to make the result reproducible.

## Official references

1. [JEP 400 — UTF-8 by Default](https://openjdk.org/jeps/400)
2. [JDK 18 Documentation](https://docs.oracle.com/en/java/javase/18/)
3. [JDK 19 Documentation](https://docs.oracle.com/en/java/javase/19/)
4. [JDK 20 Documentation](https://docs.oracle.com/en/java/javase/20/)
5. [JEP 428 — Structured Concurrency (Java 19 incubator)](https://openjdk.org/jeps/428)
6. [JEP 437 — Structured Concurrency (Java 20 second incubator)](https://openjdk.org/jeps/437)
7. [JEP 453 — Structured Concurrency (Java 21 preview)](https://openjdk.org/jeps/453)
8. [JEP 444 — Virtual Threads (Final in Java 21)](https://openjdk.org/jeps/444)
9. [JEP 440 — Record Patterns (Final in Java 21)](https://openjdk.org/jeps/440)
10. [JEP 441 — Pattern Matching for switch (Final in Java 21)](https://openjdk.org/jeps/441)

## Chapter checklist

- [ ] Explain UTF-8 default charset change and explicit-encoding best practices.
- [ ] Track virtual threads, record patterns, and switch patterns by status and release.
- [ ] Distinguish preview from incubator APIs.
- [ ] Use a version matrix and migration tests for upgrades.
