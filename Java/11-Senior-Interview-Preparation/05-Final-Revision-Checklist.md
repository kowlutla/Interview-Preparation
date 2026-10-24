# Final Java Interview Revision Checklist

Use this checklist in the final 2–3 weeks before interviews. Mark a topic complete only when you can explain it aloud, solve a representative problem, and discuss at least one failure mode or trade-off.

## 1. Core language

- [ ] Explain JDK, JVM, class files, class loading, linking, initialization, and JIT execution.
- [ ] Predict numeric promotion, narrowing, overflow, floating-point behavior, and short-circuit evaluation.
- [ ] Explain access modifiers, nested classes, constructors, static initialization, and dynamic dispatch.
- [ ] Apply equals/hashCode contracts and explain mutable-key failures.
- [ ] Explain generics, erasure, invariance, PECS, wildcard capture, and heap pollution.
- [ ] Design exception hierarchies and preserve causes/suppressed exceptions.
- [ ] Explain String immutability, pooling, Unicode code units/code points, and secure text handling.
- [ ] Distinguish enums, records, sealed classes, annotations, reflection, and dynamic proxies.

## 2. Collections

- [ ] Choose List, Set, Map, Queue, Deque, and sorted collections from semantics.
- [ ] Explain ArrayList size/capacity, amortized append, and subList views.
- [ ] Explain HashMap hashing, collisions, resizing, null policy, and iteration order.
- [ ] Explain comparator consistency for TreeMap/TreeSet.
- [ ] Distinguish fail-fast, weakly consistent, and snapshot iterators.
- [ ] Understand unmodifiable views versus snapshots and shallow immutability.
- [ ] Select concurrent collections and atomic methods for the required invariant.

## 3. Functional programming

- [ ] Explain target typing, lambda capture, functional interfaces, and method references.
- [ ] Distinguish map, flatMap, reduce, collect, groupingBy, and partitioningBy.
- [ ] Explain laziness, short-circuiting, non-interference, and stream single-use.
- [ ] Explain why parallel reduction requires associativity and why side effects are dangerous.
- [ ] Use Optional for legitimate absence, not as a universal wrapper.
- [ ] Choose LocalDate, Instant, OffsetDateTime, ZonedDateTime, Duration, and Period correctly.

## 4. Concurrency

- [ ] Explain thread states, start versus run, interruption, join, and cooperative cancellation.
- [ ] Explain intrinsic monitors, ReentrantLock, conditions, and lock ordering.
- [ ] Explain JMM happens-before, volatile visibility, atomicity, safe publication, and final fields.
- [ ] Explain CAS, ABA, AtomicLong versus LongAdder, and contention.
- [ ] Configure executors with bounded queues, rejection policy, shutdown, and task error handling.
- [ ] Compose CompletableFuture without blocking constrained executors.
- [ ] Diagnose deadlock, livelock, starvation, race conditions, and thread-pool starvation.
- [ ] Explain virtual threads in Java 21 and why downstream resource limits still apply.
- [ ] Treat structured concurrency and scoped values as preview in Java 21, not final.

## 5. JVM and performance

- [ ] Distinguish heap, stacks, Metaspace, direct memory, code cache, and native memory.
- [ ] Explain GC reachability, roots, collector trade-offs, and why cleanup is nondeterministic.
- [ ] Diagnose leaks with heap dumps, retained size, dominator trees, and reference paths.
- [ ] Use JFR, GC logs, thread dumps, jcmd, jstack, and jdeps appropriately.
- [ ] Separate CPU, allocation, contention, I/O, and downstream bottlenecks.
- [ ] Benchmark with JMH and realistic load tests.
- [ ] Make tuning changes one at a time and validate against SLOs.

## 6. Java 8–21 version awareness

- [ ] Java 8: lambdas, streams, Optional, java.time, default methods.
- [ ] Java 9: modules, collection factory methods, private interface methods.
- [ ] Java 10: local-variable type inference with var.
- [ ] Java 11: String additions, HTTP Client, LTS baseline.
- [ ] Java 14: switch expressions final.
- [ ] Java 15: text blocks final.
- [ ] Java 16: records and pattern matching for instanceof final.
- [ ] Java 17: sealed classes final; switch pattern matching still preview.
- [ ] Java 18: UTF-8 default charset policy.
- [ ] Java 21: virtual threads, record patterns, switch pattern matching, sequenced collections final.
- [ ] Java 21: structured concurrency and scoped values were preview features.

## 7. Production engineering

- [ ] Use parameterized SQL, explicit transactions, and deterministic resource cleanup.
- [ ] Design timeouts, retries, idempotency, and reconciliation for uncertain outcomes.
- [ ] Protect APIs against injection, path traversal, SSRF, secret leakage, and unsafe deserialization.
- [ ] Define queue bounds, cache size/TTL, backpressure, and overload policy.
- [ ] Diagnose p95/p99 regressions with traces, metrics, logs, and runtime evidence.
- [ ] Explain root cause, mitigation, correction, regression test, and prevention in incidents.

## 8. Testing and build tooling

- [ ] Distinguish unit, integration, and end-to-end test boundaries.
- [ ] Use JUnit Jupiter lifecycle, parameterized tests, and assertions deliberately.
- [ ] Use Mockito for meaningful collaborator boundaries without over-mocking.
- [ ] Test asynchronous and concurrent behavior with explicit coordination, not sleeps.
- [ ] Use an injected Clock for deterministic business-time tests.
- [ ] Understand when a real database/Testcontainers integration test is necessary.
- [ ] Verify Maven/Gradle test lifecycle, test discovery, JDK toolchains, and CI runtime matrix.
- [ ] Treat coverage as a diagnostic metric rather than proof of correctness.

## 9. Interview practice

- [ ] Complete all 50 coding/output problems under time limits.
- [ ] Solve at least 10 problems without looking at reference approaches.
- [ ] Explain time/space complexity and edge cases before coding.
- [ ] Walk through all 50 production scenarios aloud.
- [ ] Complete all five mock interviews and score each answer.
- [ ] Practice concise definitions followed by one trade-off and one example.
- [ ] Verify every version-sensitive answer against official documentation.
