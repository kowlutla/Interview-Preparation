# Java Interview Handbook — Comprehensive Question Coverage Plan

**Scope:** Java only. Do not edit Spring, Hibernate/JPA, SQL, or any other guide during this work.

**Goal:** Create one complete, dependable Java interview handbook. Question count is not a target or limit. Preserve every distinct, important concept; merge only true duplicates and remove only low-value repetition—not entire topic areas.

## Editorial principles

> **Current implementation status (2026-10-25):** The primary handbook has been expanded to 355 sequentially numbered Q&As. It now has dedicated sections for stream coding exercises, Java 8–25 version evolution with LTS emphasis, core language/API contracts, collections, concurrency, JVM diagnostics, I/O/JDBC/security/testing, and production scenarios. Three Mermaid diagrams provide visual navigation. Structural checks confirmed sequential numbering and no duplicate question titles. A complete line-by-line technical audit and a full legacy-bank content-parity review remain separate tasks; do not treat structural checks as proof that every technical statement has been independently verified.

1. Topic completeness comes before compression. The primary handbook is organized as a 355-question guide; question count is evidence of scope, not by itself proof of completeness.
2. Every major category must have its own section and sufficient questions. Exception handling, wrappers, strings, generics, collections, and concurrency must not be reduced to a single overview question.
3. Similar wording does not make questions duplicates if they test different mechanisms, edge cases, or design decisions.
4. Answers should sound natural when spoken in an interview: direct explanation, how/why it works, a small example when useful, caveats, and practical implications. Do not use a redundant Answer label.
5. Separate language/JVM/API guarantees from OpenJDK/HotSpot implementation details and workload-dependent performance observations.
6. Cover Java 8 fundamentals plus relevant evolution through Java 11, 17, 21, and 25. Verify feature introduction and preview/final status using release-specific official references.
7. Use short snippets and output questions when they teach a concept; do not turn every answer into a coding exercise.
8. Use interview websites to discover recurring themes, not as technical authorities or proof of universal/company-specific frequency.
9. Do not delete legacy Java material until unique topics have been mapped and preserved.

## Comprehensive category map

This is a topic checklist, not a fixed question quota. A category is complete only when its important concepts, common traps, and experienced-level follow-ups have been reviewed.

### 1. Java foundations, types, operators, and execution
- JDK/JVM/JRE terminology; compilation, bytecode, class loading, linking, initialization, main method
- Primitive/reference types, ranges, defaults, literals, conversions, numeric promotion, overflow, floating-point behavior
- Pass-by-value, references, aliasing, null, scope, definite assignment, var
- Operators, precedence, short-circuiting, casts, equality and expression evaluation
- static, final, access modifiers, packages, classpath/module path
- Arrays, covariance, ArrayStoreException, varargs and array equality
- Object creation, constructors, initialization order, nested/inner/local/anonymous classes
- Compile-time versus runtime errors and carefully selected output questions

### 2. OOP, inheritance, polymorphism, and design
- Four OOP principles, composition versus inheritance, SOLID and substitutability
- Overloading versus overriding; static hiding, field hiding, covariant returns, overload resolution
- Interface versus abstract class, default methods and conflicts
- this/super, constructor chaining, final classes/methods/variables
- Immutability, defensive copies, invariants and value objects
- Object methods, equality/hashCode contracts and equality across inheritance
- Builder, Factory, Strategy, Decorator, Adapter, Template Method, Observer, Proxy, Singleton; when patterns are useful or over-engineered

### 3. Strings and text handling
- Immutability, string pool, literals versus new String, intern and reference identity
- == versus equals; String versus StringBuilder versus StringBuffer
- Concatenation, compiler/runtime optimization caveats, common string methods and regex
- split/replace/replaceAll, trim/strip, formatting and text blocks
- Unicode, UTF-16 code units, code points, grapheme clusters, normalization and explicit charsets
- String as a map key, safe text processing and output puzzles

### 4. Primitive wrappers, boxing, and numeric APIs
- Why wrappers exist; autoboxing/unboxing and conversion rules
- Wrapper caching and why identity comparison is unsafe
- Wrapper equality, compareTo, null unboxing and primitive-versus-wrapper API design
- Integer overflow, floating-point precision, NaN, infinities and signed zero
- BigInteger/BigDecimal, scale, rounding modes and money calculations
- parseXxx/valueOf, conversion failures, Character and Unicode

### 5. Exception handling and error design
- Throwable hierarchy; Error, Exception, RuntimeException; checked versus unchecked
- throw versus throws, propagation, stack unwinding and catch ordering
- try/catch/finally return/throw behavior; multi-catch and precise rethrow
- try-with-resources, AutoCloseable, close order and suppressed exceptions
- Custom exceptions, cause chaining, wrapping and API-boundary decisions
- Exceptions from constructors, static initialization, lambdas, streams and asynchronous tasks
- InterruptedException, restoring interrupt status and cancellation
- Cleanup failures masking the original exception; swallowed/broad catches and duplicate logging
- OutOfMemoryError, StackOverflowError, ExceptionInInitializerError and NoClassDefFoundError
- Production error mapping, diagnostics and safe retry decisions

### 6. Collections and data structures
- Collection hierarchy and why Map is separate; List, Set, Queue, Deque and Map contracts
- ArrayList versus LinkedList, resizing, complexity, iteration and memory locality
- HashMap internals, hashing, collisions, tree bins, resizing and implementation-versus-contract boundaries
- equals/hashCode, mutable keys, nulls, ordering, LinkedHashMap and LRU patterns
- HashSet/LinkedHashSet/TreeSet; HashMap/LinkedHashMap/TreeMap
- Comparable versus Comparator, comparator contracts, sorting and binary search
- PriorityQueue, Deque, iterator/ListIterator, fail-fast limitations and iterator removal
- Arrays.asList, unmodifiable views, immutable factory collections, defensive copies and mutability surprises
- ConcurrentHashMap, CopyOnWriteArrayList, blocking queues, weak/identity/enum collections
- Collection choice by workload, concurrency guarantees and complexity caveats

### 7. Generics and the type system
- Generic classes, interfaces, methods, constructors, bounds and type inference
- extends versus super, PECS, invariance and arrays versus generics
- Type erasure, raw types, reifiable types, generic arrays, class literals and instanceof restrictions
- Bridge methods, wildcard capture, intersection/recursive bounds and overload restrictions after erasure
- Heap pollution, unchecked casts, @SafeVarargs and generic varargs
- Compile-time examples and practical API design

### 8. Core java.lang and object contracts
- Object methods: equals, hashCode, toString, clone, getClass, wait/notify/notifyAll
- Equality laws, hash-based collection consequences and mutable equality state
- Shallow versus deep copy; alternatives to clone
- Comparable, Comparator, Class, System, Runtime, Math/StrictMath and identity versus logical equality
- Monitor ownership and appropriate use of wait/notify

### 9. Functional interfaces, lambdas, streams, and Optional
- Functional interfaces, target typing, lambdas, capture, method/constructor references
- Predicate, Function, Consumer, Supplier, operators and primitive specializations
- Stream laziness, intermediate/terminal operations, single-use behavior and short-circuiting
- map/flatMap, filter, peek, distinct, sorted, limit/skip
- reduce versus collect, identity/associativity/combiner, grouping/partitioning/joining/toMap duplicate keys
- Encounter order, side effects, elided operations, parallel streams, spliterators and common pool
- Optional map/flatMap, orElse/orElseGet and misuse patterns
- When a loop is clearer than a stream

### 10. Date/time, locale, and utility APIs
- java.time versus legacy Date/Calendar; Instant, LocalDateTime, OffsetDateTime, ZonedDateTime
- UTC/time zones, daylight-saving transitions, Duration versus Period, formatting and parsing
- Locale-sensitive formatting, explicit zones, BigDecimal and rounding
- Objects, Arrays, Collections, UUID, Random versus SecureRandom

### 11. Concurrency fundamentals and Java Memory Model
- Process/thread, platform/virtual threads, thread states, start versus run, join/sleep/yield/daemon
- Runnable/Callable, Future/FutureTask/CompletableFuture
- Race conditions versus data races, atomicity, visibility and ordering
- synchronized instance/static/block behavior, volatile guarantees and limitations
- Happens-before, safe publication, final-field semantics and immutable shared state
- Deadlock, livelock, starvation, contention and thread-dump diagnosis
- Interruption, cancellation, timeouts and cleanup

### 12. Concurrency utilities and advanced concurrency
- Executor, ExecutorService, ScheduledExecutorService and ThreadPoolExecutor
- Pool sizing, queue capacity, rejection, saturation, backpressure, shutdown and graceful termination
- Future limitations; CompletableFuture composition, failures and executor selection
- ReentrantLock, ReadWriteLock, StampedLock, Condition, atomics, CAS, ABA and LongAdder
- CountDownLatch, CyclicBarrier, Phaser, Semaphore, Exchanger and BlockingQueue
- ConcurrentHashMap compound operations, ForkJoinPool/work stealing and blocking pitfalls
- ThreadLocal leaks and context cleanup; virtual-thread trade-offs and scarce-resource limits
- Structured concurrency/scoped values with exact version/status labels
- Thread-starvation deadlocks and deterministic concurrency testing

### 13. JVM, class loading, bytecode, and modules
- Runtime data areas, heap/stacks/metaspace terminology and spec-versus-implementation distinction
- Loading, delegation, linking, verification, preparation, resolution and initialization
- Class identity, class-loader conflicts, linkage errors, classpath/module path
- Bytecode/JARs, javap, class-file versions and binary compatibility
- JIT, profiling, inlining, deoptimization and warm-up
- Reflection, proxies, method handles, invokedynamic, agents and JNI at appropriate depth
- Modules, exports/opens, strong encapsulation and multi-release JARs

### 14. Memory management, garbage collection, and diagnostics
- Reachability, GC roots, strong/soft/weak/phantom references and ReferenceQueue
- Memory leak versus high usage; retained paths, heap dumps, histograms and dominators
- Generational GC, G1/ZGC/Shenandoah and release-specific behavior
- Pause/throughput trade-offs, allocation rate, live set, promotion and GC logs
- Metaspace, direct buffers, thread stacks, code cache and container limits
- JFR/JMC, jcmd/jstack/jmap/jstat and Native Memory Tracking
- Finalization status, Cleaner, try-with-resources and evidence-based tuning

### 15. I/O, NIO, files, serialization, and networking
- Byte versus character streams, Reader/Writer, buffering, charsets and resource ownership
- Path/Files, directory traversal, walk resource closure, channels, buffers, selectors and async I/O
- Object serialization, serialVersionUID, compatibility, filters and security risks
- Sockets, DNS, TLS/certificate/hostname validation, timeouts and redirects
- Standard HttpClient, reuse, async calls, cancellation, path traversal and SSRF

### 16. JDBC and database-facing Java
- Connection, Statement, PreparedStatement, CallableStatement, ResultSet and DataSource
- SQL injection prevention, resource management, autocommit, commit/rollback/savepoints
- Batching, generated keys, fetch size, pool ownership, timeouts/cancellation and unknown outcomes
- Keep Java JDBC coverage distinct from the separate SQL guide

### 17. Enums, annotations, reflection, and annotation processing
- Enum constructors, constant-specific bodies, enum singleton, EnumSet/EnumMap
- Annotation targets, retention, repeatable/type-use annotations
- Reflection access and module boundaries, dynamic proxies
- Annotation processor rounds, generated sources, service registration
- javac --release, javap, jdeps and compiler diagnostics

### 18. Java version evolution and compatibility
- Java 8 lambdas, streams, Optional, java.time, default methods and CompletableFuture
- Java 9–11 modules, collection factories, var, HttpClient and API/runtime changes
- Java 14–17 switch expressions, text blocks, records, pattern matching and sealed types
- Java 21 virtual threads, record patterns, pattern matching for switch, sequenced collections and generational ZGC
- Java 25 and later: relevant finalized/preview features and migration concerns
- LTS, toolchains, --release, source/target, dependency compatibility and feature status

### 19. Security and secure coding
- SecureRandom, secrets, cryptographic APIs and TLS verification
- Unsafe deserialization, path traversal, SSRF, redirects and untrusted inputs
- Least privilege, safe logs, reflection/deserialization boundaries and dependency risks
- Why ad hoc cryptography and disabled certificate verification are unsafe

### 20. Performance, testing, and engineering practice
- Complexity versus runtime behavior; JMH pitfalls, warm-up and dead-code elimination
- JFR/profiling, allocation, contention, latency percentiles and evidence-based tuning
- JUnit 5, Mockito, test doubles, deterministic async/concurrency tests and timeouts
- Property-based testing, Testcontainers, Maven/Gradle toolchains and test lifecycle

### 21. Output, compilation, and code-reading questions
- Strings, wrapper caching, initialization order, overload resolution, varargs and boxing
- Exception control flow, finally, try-with-resources and suppressed exceptions
- Generics/erasure, collection order/mutation, streams/collectors and concurrency outcomes
- Explicitly label compile-time error, runtime exception, deterministic output or nondeterministic behavior

### 22. Senior-level production scenarios
- Mutable map key, equality bugs, cache behavior, collection contention
- API latency with low CPU, pool exhaustion, executor starvation and deadlocks
- Lost updates, cancellation, safe shutdown, GC pressure, leaks and native-memory growth
- Slow I/O/database/network calls, timeout alignment, retries and idempotency
- JDK upgrade regression, classpath/module conflict and dependency incompatibility
- Answers must cover symptom, evidence, competing hypotheses, safe mitigation, root cause and prevention

## Review workflow

For each category: inventory the existing Java chapters and examples; extract distinct questions; identify duplicates, weak answers and missing concepts; compare recurring interview themes across multiple sources; verify technical claims against the JLS/JVMS/API/JEP and release documentation; then keep, merge, rewrite, expand, add or remove individual questions with a reason. Mark a category reviewed only after the subtopic checklist and question inventory are checked.

## Acceptance criteria

- Every category above has meaningful question coverage; no major topic is represented by one shallow overview question.
- The current 64 questions are audited, not blindly retained or discarded.
- The legacy bank of more than 1,000 Q&As is mined for unique value before cleanup.
- Answers are complete, natural, technically accurate, and useful to explain aloud.
- Question duplication is judged by knowledge tested, not wording alone.
- Version-sensitive claims cite authoritative references.
- The single canonical Markdown file has a clear contents section, logical categories, revision paths, and references.
- Structural checks, technical review, and code execution are reported separately; never claim checks that were not performed.
- Legacy files remain until topic coverage parity is verified.

## Research starting points

### Official references
- [Java SE 25 specifications](https://docs.oracle.com/en/java/javase/25/docs/specs/)
- [Java SE 25 API documentation](https://docs.oracle.com/en/java/javase/25/docs/api/)
- [Java Language Specification, Java SE 25](https://docs.oracle.com/javase/specs/jls/se25/html/index.html)
- [Java Virtual Machine Specification, Java SE 25](https://docs.oracle.com/javase/specs/jvms/se25/html/index.html)
- [Java concurrency utilities](https://docs.oracle.com/en/java/javase/25/docs/api/java.base/java/util/concurrent/package-summary.html)
- [java.lang package and wrapper classes](https://docs.oracle.com/en/java/javase/25/docs/api/java.base/java/lang/package-summary.html)

### Secondary sources for question discovery
- [Java interview themes from public interview transcripts](https://www.crackr.dev/interview-guides/java-interview-questions)
- [Java interview questions by topic](https://coddy.tech/interview-questions/java)
- [Core Java interview topic guide](https://www.lets-code.co.in/articles/corejavainterviewquestions/)

Secondary sources are discovery signals, not technical authorities. Frequency estimates depend on the sample and role; do not present them as universal.
