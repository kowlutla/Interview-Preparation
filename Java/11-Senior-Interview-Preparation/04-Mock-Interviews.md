# Five Mock Java Interviews

These mock interviews are designed for a Java backend engineer targeting Senior Software Engineer roles. Use each as a timed session: answer aloud first, then compare your answer with the evaluation points. Interviewers should probe assumptions, trade-offs, failure handling, and operational consequences.

## Mock Interview 1 — Core Java and OOP (45–60 minutes)

### Question 1. Explain the path from Java source code to execution.

**Strong answer should cover:** javac, class-file bytecode, class loading/linking/initialization, interpretation/JIT, runtime profiling, and target JVM compatibility.

**Follow-ups:** What does --release do? Why can NoSuchMethodError occur? What does the class loader contribute to type identity?

**Red flags:** Claiming bytecode is native machine code or that every object is always physically allocated in the heap.

### Question 2. Explain equals and hashCode contracts.

**Strong answer should cover:** reflexivity, symmetry, transitivity, consistency, null behavior, equal values having equal hashes, collisions being permitted, and mutable-key hazards.

**Follow-ups:** Why does HashMap lookup fail after a key mutation? How should arrays be compared? How does inheritance complicate equals?

### Question 3. When would you use a record versus an ordinary class?

**Strong answer should cover:** fixed transparent components, generated value semantics, shallow immutability, validation, finality, lack of arbitrary superclass inheritance, and whether the domain models a value or an entity.

**Follow-ups:** What if a component is a mutable list? What if toString contains a secret? Can a record implement an interface?

### Question 4. Explain generics and type erasure.

**Strong answer should cover:** compile-time type safety, erasure, raw types, invariance, wildcards, PECS, bridge methods, heap pollution, and generic-array restrictions.

**Follow-ups:** Why is List<Integer> not a List<Number>? Why is List<String> not valid in an instanceof test? When is wildcard capture useful?

### Question 5. Design exception handling for a service boundary.

**Strong answer should cover:** domain errors versus infrastructure failures, checked/unchecked trade-offs, cause preservation, safe public error mapping, transaction rollback, and retryability.

**Follow-ups:** How do you preserve InterruptedException? How do suppressed exceptions work? Why is catching Throwable dangerous?

### Evaluation rubric

Score each question 0–4: 0 incorrect, 1 definitions only, 2 correct mechanics, 3 trade-offs and edge cases, 4 production-quality reasoning with examples. Target 16/20 or better.

## Mock Interview 2 — Collections and Algorithms (45–60 minutes)

### Question 1. Choose data structures for a recent-events service.

**Prompt:** Keep events by ID, retrieve the latest event before a timestamp, and preserve a display order for recent entries.

**Strong answer should cover:** HashMap for ID lookup, NavigableMap for ordered time navigation where appropriate, LinkedHashMap or explicit sorting for display order, and a retention policy.

**Follow-ups:** How do duplicate timestamps work? What if events exceed memory? How do concurrent updates affect snapshots?

### Question 2. Explain HashMap collision handling and resizing.

**Strong answer should cover:** hash/equality, bucket lookup, collision nodes, OpenJDK tree bins as implementation detail, capacity/load factor, resizing, and stable key state.

**Follow-ups:** Does HashMap guarantee iteration order? Why can poor hash distribution be a security/performance concern? How would you diagnose a slow map?

### Question 3. Implement top K frequent values.

**Expected approach:** Frequency map plus a bounded min-heap, O(n + u log k) time for n inputs and u distinct values. Discuss tie-breaking, k bounds, memory, and alternatives such as bucket sorting.

### Question 4. Explain fail-fast, weakly consistent, and snapshot iterators.

**Strong answer should cover:** best-effort ConcurrentModificationException, concurrent collection contracts, CopyOnWriteArrayList snapshot semantics, and why none should be mistaken for a universal transactional snapshot.

### Question 5. When is a stream worse than a loop?

**Strong answer should cover:** complex control flow, checked exceptions, allocation/boxing, parallel coordination, side effects, and readability. Mention JMH and realistic profiling rather than assumptions.

### Evaluation rubric

Score algorithm correctness, complexity, edge cases, data-structure selection, and clarity from 0–4 each. Require the candidate to discuss trade-offs rather than just write code.

## Mock Interview 3 — Concurrency and JVM (60 minutes)

### Question 1. Explain happens-before and volatile.

**Strong answer should cover:** visibility/ordering, volatile write/read, monitor unlock/lock, thread start/join, and why volatile increment is not atomic.

**Follow-ups:** How would you safely publish an object? What is double-checked locking's requirement? Why does sleep not fix visibility?

### Question 2. Diagnose a deadlocked service.

**Prompt:** Request latency is rising; thread dumps show two groups waiting on monitors.

**Strong answer should cover:** repeated thread dumps, monitor ownership, wait-for graph, lock ordering, external calls inside critical sections, and prevention tests.

### Question 3. Design a bounded executor for blocking I/O.

**Strong answer should cover:** queue capacity, worker limits, rejection/backpressure, downstream connection limits, deadlines, interruption, task failure observation, and metrics.

**Follow-ups:** Why can newFixedThreadPool hide overload? What happens when submit's Future is ignored? How do you shut down gracefully?

### Question 4. Explain virtual threads and their limitations.

**Strong answer should cover:** Java 21 finalization, carrier threads, blocking workload suitability, pinning concerns in Java 21, downstream capacity, cancellation, and why virtual threads are not a CPU speedup.

### Question 5. Diagnose a memory leak and a native-memory OOM.

**Strong answer should cover:** heap dump, dominator tree, retained size, GC logs, JFR, Metaspace, direct buffers, thread stacks, NMT, and container RSS.

**Follow-ups:** What is a class-loader leak? How can ThreadLocal retain request data? Why can a container kill a process with heap headroom?

### Evaluation rubric

Require correct JMM reasoning, an actionable diagnostic plan, and awareness of resource limits. Penalize fixes based only on sleeps, arbitrary JVM flags, or increasing pool sizes.

## Mock Interview 4 — Streams, APIs, and Java 8–21 (45–60 minutes)

### Question 1. Explain map, flatMap, reduce, and collect.

**Strong answer should cover:** one-to-one mapping, flattening, associative reductions, mutable reduction via collectors, downstream collectors, and parallel correctness.

### Question 2. Design an Optional-based repository API.

**Prompt:** A user may not exist; the database may also be unavailable.

**Strong answer should cover:** Optional.empty for legitimate absence, exceptions/result types for operational failures, and clear HTTP mapping. Do not collapse timeout into “not found.”

### Question 3. Compare LocalDateTime, Instant, OffsetDateTime, and ZonedDateTime.

**Strong answer should cover:** local civil values versus timeline points, offset versus regional time-zone rules, DST gaps/overlaps, and persistence semantics.

### Question 4. Walk through Java 8, 11, 17, and 21 feature status.

**Strong answer should cover:** lambdas/streams/java.time (8), var (10), String APIs/HTTP Client (11), records final (16), sealed classes final (17), virtual threads and pattern matching for switch final (21), plus preview status for structured concurrency/scoped values in Java 21.

### Question 5. Explain source, binary, and runtime compatibility.

**Strong answer should cover:** language level, platform API, class-file version, linkage, dependency convergence, preview flags, and testing with the intended runtime.

### Evaluation rubric

A strong candidate distinguishes feature motivation from syntax and names the release where a feature became final. Check that they do not describe preview APIs as permanent.

## Mock Interview 5 — Senior Production Design and Debugging (60 minutes)

### Question 1. A service's p99 latency doubled after deployment. How do you investigate?

**Strong answer should cover:** deployment diff, traffic mix, p50/p95/p99, CPU, allocation/GC, thread dumps, pool/queue metrics, database queries, downstream traces, and a rollback criterion.

**Follow-ups:** How do you distinguish GC correlation from causation? What if CPU is low? What if only one endpoint is affected?

### Question 2. A payment request times out and the caller retries. How do you prevent duplicate charges?

**Strong answer should cover:** unknown remote outcome, idempotency key, durable operation state, provider support, bounded retry/backoff, and reconciliation.

### Question 3. Design a safe in-memory cache.

**Strong answer should cover:** maximum size, TTL/eviction, concurrency, per-key load coalescing, error caching, invalidation, metrics, and whether stale reads are acceptable.

### Question 4. A container is OOM-killed while Java heap usage is moderate. Explain the investigation.

**Strong answer should cover:** cgroup limit, RSS, native memory, direct buffers, Metaspace, thread stacks, code cache, NMT, and container-aware JVM settings.

### Question 5. How would you make a Java library safe to evolve across multiple services?

**Strong answer should cover:** API surface, source/binary/behavioral/wire compatibility, default methods, semantic versioning, deprecation, contract tests, and version rollout.

### Evaluation rubric

Look for an ordered incident response: impact, mitigation, evidence, hypotheses, root cause, corrective change, regression test, and prevention. Strong senior answers discuss trade-offs and operational ownership.

## Final self-assessment

- Can I explain the mechanism, not just define the term?
- Can I identify assumptions and edge cases before coding?
- Can I describe complexity and resource costs?
- Can I distinguish language guarantees from implementation details?
- Can I build a diagnostic plan using evidence?
- Can I explain a safe failure/retry/cancellation policy?
- Can I state the Java version and feature status precisely?
