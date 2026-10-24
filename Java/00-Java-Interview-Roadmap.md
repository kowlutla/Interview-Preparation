# Java Interview Roadmap (JDK 8–21)

## Objective

Build durable understanding of Java language semantics, APIs, concurrency, the JVM, production diagnosis, and version evolution. The aim is not to memorize isolated answers: it is to explain behavior precisely, reason from contracts, identify trade-offs, and debug systems under pressure.

## Target outcomes

By the end of this track, you should be able to:

- Explain compilation, bytecode execution, runtime memory, class loading, JIT compilation, and garbage collection.
- Apply object-oriented design and language semantics without relying on folklore.
- Predict behavior involving equality, generics, overload resolution, exceptions, collections, streams, and concurrency.
- Design thread-safe components and diagnose data races, deadlocks, executor starvation, and asynchronous error handling.
- Investigate latency, CPU, memory growth, GC pauses, classpath/module issues, and production incidents using JVM tools.
- Explain Java 8–21 feature history, including the difference between a standard feature, preview feature, incubator API, deprecated API, and implementation detail.
- Solve coding tasks with testable solutions, state complexity, discuss edge cases, and articulate alternative designs.

## Suggested study sequence

| Phase | Chapters | Focus | Exit criterion |
|---|---|---|---|
| 1. Foundations | `01-Java-Fundamentals` | Execution model, types, OOP, strings, object contracts | Explain behavior and write small examples without notes |
| 2. Language depth | `02-Core-Language` | Exceptions, generics, reflection, modules, I/O | Explain contracts, limitations, and failure modes |
| 3. Data structures | `03-Collections-Framework` | Contracts, complexity, implementation internals | Choose a collection from requirements and justify it |
| 4. Functional Java | `04-Java-8-and-Functional-Programming` | Lambdas, streams, collectors, Optional, time | Write readable pipelines and recognize when loops are better |
| 5. Concurrency | `05-Concurrency-and-Multithreading` | JMM, locks, executors, futures, virtual threads | Identify safety, liveness, cancellation, and backpressure concerns |
| 6. Runtime | `06-JVM-and-Memory-Management` | Class loading, JIT, GC, diagnostics | Build a hypothesis-led production investigation |
| 7. Version evolution | `07-Java-Version-Features` | JDK 8–21, finalized/preview/incubator distinctions | Correctly explain when a feature became available and its status |
| 8. Applied design | `08-Advanced-Java` | JDBC, network I/O, design, security, performance | Justify implementation and operational trade-offs |
| 9. Practice | `09-Coding-and-Problem-Solving` | Coding/output prediction | Solve, test, explain complexity and corner cases |
| 10. Incidents | `10-Real-World-Scenarios` | Root-cause analysis under constraints | Present evidence-based investigation steps and mitigations |
| 11. Interview performance | `11-Senior-Interview-Preparation` | Cross-questioning, architecture, mocks, revision | Give structured answers with assumptions, trade-offs, and follow-ups |

## Difficulty model

- **Beginner:** Define the concept and use the ordinary API correctly.
- **Intermediate:** Explain mechanics, contracts, edge cases, and typical trade-offs.
- **Advanced:** Reason about interactions between language features, APIs, and runtime behavior.
- **Senior:** Diagnose realistic failures, make design choices under constraints, and discuss operations.
- **Expert:** Explain specification boundaries, implementation trade-offs, performance evidence, and less common failure modes.

Difficulty is a guide, not a measure of professional worth. A question may require knowledge across several chapters.

## How to answer senior-level questions

Use this repeatable structure:

1. **Definition and guarantee:** State what the language/API/specification actually guarantees.
2. **Mechanism:** Explain enough of the internal model to justify the behavior.
3. **Example:** Provide a small example or a concrete execution flow.
4. **Limitations:** Name the edge cases, operational constraints, and common misconceptions.
5. **Decision:** Explain when you would use it, avoid it, or choose an alternative.
6. **Evidence:** For performance claims, propose a benchmark or diagnostic measurement rather than assuming the result.

For production debugging, state the symptom and impact, establish scope, gather evidence, form competing hypotheses, run the least invasive discriminating checks, mitigate safely, verify recovery, and then document a root cause and prevention plan.

## Version-awareness rules

- The handbook targets language and platform knowledge from Java 8 through Java 21. A feature introduced in one release may have become permanent in a later release.
- Always name the release when discussing a preview feature. Preview features generally require explicit enablement at compilation and runtime for the corresponding release and are not stable compatibility promises.
- Incubator modules are distinct from finalized Java SE APIs and may require explicit module resolution.
- Distinguish the Java SE specification/API contract from a specific HotSpot implementation detail. Do not assume all JVM implementations use identical internal layouts or optimization decisions.
- For implementation questions about collections, garbage collectors, or JIT behavior, pin the discussion to the JDK release and implementation when it matters.
- Update-release details (security fixes, time-zone data, vendor packaging and licensing) are different from feature-release language/API changes.

Release quick reference: [Java 8–21 feature and status matrix](07-Java-Version-Features/00-Java-8-to-21-Feature-Status-Matrix.md). This matrix is a summary; use the release-specific JEP and API documentation for exact details.

Official version starting points:
- [JDK 21 documentation](https://docs.oracle.com/en/java/javase/21/)
- [Java language changes for JDK 21](https://docs.oracle.com/en/java/javase/21/language/java-language-changes-release.html)
- [OpenJDK JEP index](https://openjdk.org/jeps/0)
- [Java Language Specification, Java SE 21](https://docs.oracle.com/javase/specs/jls/se21/html/index.html)
- [Java Virtual Machine Specification, Java SE 21](https://docs.oracle.com/javase/specs/jvms/se21/html/index.html)

## Practice and review loop

For each chapter:

1. Read the chapter once without trying to memorize it.
2. Re-derive mechanisms using diagrams or small programs.
3. Run code snippets in the documented target JDK; keep output and assumptions with the example.
4. Answer the questions aloud, then answer the follow-ups without looking.
5. Add missed concepts to a personal error log and revisit them after 2–3 days.
6. Complete associated coding problems and test boundary conditions.
7. Revisit the chapter after finishing concurrency/JVM topics, since these often change how earlier behavior is understood.

## Mastery tracker

- [ ] Explain the JDK/JRE/JVM relationship and source-to-runtime pipeline.
- [ ] Explain primitive/reference values, conversions, initialization and control flow.
- [ ] Apply OOP, class/interface rules, records, sealed types and object contracts.
- [ ] Explain exceptions, generics, annotations, reflection, JPMS, I/O, serialization, JUnit 5, Mockito and build/test lifecycles.
- [ ] Know collection contracts, complexity, resizing, hashing and concurrent alternatives.
- [ ] Use lambdas, method references, streams, collectors, Optional and java.time thoughtfully.
- [ ] Explain the JMM, happens-before, synchronization, atomics, executors and async error handling.
- [ ] Explain platform versus virtual threads and recognize the limitations of each.
- [ ] Explain class loading, bytecode, runtime memory areas, JIT and GC at interview depth.
- [ ] Investigate CPU, memory, GC, thread and startup problems using appropriate tools.
- [ ] Compare version features from Java 8 to 21 and identify preview/incubator status accurately.
- [ ] Complete 50+ coding/output problems, 50+ production scenarios and five mock interviews.
- [ ] Reach 400+ unique handbook questions with answers and no known duplicate questions.

## Repository implementation status

The handbook content has been authored directly on the repository's `main` branch. See the [README and live completion tracker](README.md) for the current file inventory and status.

Current verified content scope:
- 57 Q&A chapters with **1,078 numbered Q&As** across fundamentals, core language, collections, functional programming, concurrency, JVM internals, Java 8–21 release features, and advanced Java testing/build tooling.
- 50 coding/output practice problems.
- 85+ production troubleshooting scenarios across collections/concurrency, JVM/memory, performance, and root-cause analysis.
- Five mock interviews, advanced cross-questioning, architecture trade-offs, rapid-fire revision, and a final revision checklist.
- Official documentation index with JLS, JVMS, Java API, JEP, GC, JFR, and diagnostic-tool references.

This is a content-completion snapshot, not a claim that every code example has been compiled or every URL has been exhaustively verified. Continue validating snippets against their stated Java release and expand any topic where interview or production depth requires it.
