# Official Java Documentation and Resources

This index prioritizes specifications, official API documentation, JEPs, and official tools. For version-sensitive interview answers, use the documentation matching the target release. A JEP is useful for feature history/status; the JLS/JVMS/API specification defines the actual contract.

## Core language and JVM

- [Java Language Specification, Java SE 21](https://docs.oracle.com/javase/specs/jls/se21/html/index.html)
- [Java Virtual Machine Specification, Java SE 21](https://docs.oracle.com/javase/specs/jvms/se21/html/index.html)
- [Java SE 21 API documentation](https://docs.oracle.com/en/java/javase/21/docs/api/)
- [Java SE 21 JDK tools reference](https://docs.oracle.com/en/java/javase/21/docs/specs/man/index.html)
- [JDK 21 documentation index](https://docs.oracle.com/en/java/javase/21/)

## Collections, functional programming, and concurrency

- [Collections Framework overview](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/util/doc-files/coll-overview.html)
- [java.util.function package](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/util/function/package-summary.html)
- [java.util.stream package](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/util/stream/package-summary.html)
- [java.util.concurrent package](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/util/concurrent/package-summary.html)
- [JLS Chapter 17 — Threads and Locks](https://docs.oracle.com/javase/specs/jls/se21/html/jls-17.html)

## JVM, GC, and diagnostics

- [Java SE 21 GC Tuning Guide](https://docs.oracle.com/en/java/javase/21/gctuning/)
- [Java SE 21 Flight Recorder documentation](https://docs.oracle.com/en/java/javase/21/jfapi/index.html)
- [Native Memory Tracking](https://docs.oracle.com/en/java/javase/21/vm/native-memory-tracking.html)
- [HotSpot performance enhancements](https://docs.oracle.com/en/java/javase/21/vm/java-hotspot-virtual-machine-performance-enhancements.html)
- [JMH — Java Microbenchmark Harness](https://openjdk.org/projects/code-tools/jmh/)

## Feature history and status (selected)

- [JEP 126 — Lambda Expressions](https://openjdk.org/jeps/126)
- [JEP 286 — Local-Variable Type Inference](https://openjdk.org/jeps/286)
- [JEP 321 — HTTP Client](https://openjdk.org/jeps/321)
- [JEP 361 — Switch Expressions](https://openjdk.org/jeps/361)
- [JEP 378 — Text Blocks](https://openjdk.org/jeps/378)
- [JEP 394 — Pattern Matching for instanceof](https://openjdk.org/jeps/394)
- [JEP 395 — Records](https://openjdk.org/jeps/395)
- [JEP 403 — Strongly Encapsulate JDK Internals](https://openjdk.org/jeps/403)
- [JEP 406 — Pattern Matching for switch (Java 17 preview)](https://openjdk.org/jeps/406)
- [JEP 409 — Sealed Classes](https://openjdk.org/jeps/409)
- [JEP 421 — Deprecate Finalization for Removal](https://openjdk.org/jeps/421)
- [JEP 431 — Sequenced Collections](https://openjdk.org/jeps/431)
- [JEP 439 — Generational ZGC](https://openjdk.org/jeps/439)
- [JEP 440 — Record Patterns](https://openjdk.org/jeps/440)
- [JEP 441 — Pattern Matching for switch](https://openjdk.org/jeps/441)
- [JEP 444 — Virtual Threads](https://openjdk.org/jeps/444)
- [JEP 446 — Scoped Values (Java 21 preview)](https://openjdk.org/jeps/446)
- [JEP 453 — Structured Concurrency (Java 21 preview)](https://openjdk.org/jeps/453)

## Security and I/O

- [Java SE 21 Security Developer's Guide](https://docs.oracle.com/en/java/javase/21/security/)
- [java.io package](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/io/package-summary.html)
- [java.nio.file package](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/nio/file/package-summary.html)
- [JEP 290 — Filter Incoming Serialization Data](https://openjdk.org/jeps/290)

## Version policy for this handbook

1. Prefer JLS/JVMS/API docs for normative behavior.
2. Use JEPs to explain motivation and feature history.
3. Label preview/incubator status by release; do not infer final status from a feature's first appearance.
4. When API behavior differs across versions, cite the API page for the intended baseline.
5. Recheck URLs and support status before a major handbook release.
