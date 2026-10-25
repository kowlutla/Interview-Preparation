# Java Interview Handbook — Core Concepts to Senior-Level Discussion

A focused, version-aware guide for Java backend engineers. It prioritizes questions that repeatedly appear across interview-preparation sources and covers the concepts most likely to distinguish a prepared engineer: language contracts, collections, functional programming, concurrency, JVM behavior, modern Java, and production diagnosis.

**Scope:** Java 8 fundamentals through modern Java, with selected release notes for Java 11, 17, 21, and 25.  
**Audience:** Developers preparing for experienced and senior Java/backend roles.  
**How to use this guide:** Learn the explanation, then practise saying it in your own words. Do not memorize every sentence. For a short revision window, start with the revision list near the end.

> Interview-question frequency varies by company, interviewer, and role. The selection below is curated from recurring themes in multiple interview resources and cross-checked against official Java documentation; it is not a claim that every company asks these exact questions.

## Contents

1. [Java foundations and language behavior](#1-java-foundations-and-language-behavior)
2. [Object-oriented design and contracts](#2-object-oriented-design-and-contracts)
3. [Collections and data structures](#3-collections-and-data-structures)
4. [Generics, exceptions, and functional programming](#4-generics-exceptions-and-functional-programming)
5. [Concurrency and the Java Memory Model](#5-concurrency-and-the-java-memory-model)
6. [JVM, memory, and performance](#6-jvm-memory-and-performance)
7. [Modern Java](#7-modern-java)
8. [Production and senior-level scenarios](#8-production-and-senior-level-scenarios)
9. [Additional specialist topics](#9-additional-specialist-topics)
10. [Expanded coverage: frequently tested Java topics](#10-expanded-coverage-frequently-tested-java-topics)
14. [Java Streams — coding interview questions](#14-java-streams--coding-interview-questions)
15. [Java version-wise features: Java 8 to Java 25](#15-java-version-wise-features-java-8-to-java-25)
16. [Core Java language, object model, and API contracts](#16-core-java-language-object-model-and-api-contracts)
19. [Last-minute revision](#19-last-minute-revision)
20. [Official references](#20-official-references)
21. [Editorial and validation notes](#21-editorial-and-validation-notes)

---

## 1. Java foundations and language behavior

### Q1. What are the JDK, JVM, and JRE, and how does a Java program run?

The **JDK** is the development kit: it includes tools such as `javac`, `java`, `jar`, `javadoc`, and diagnostic utilities, together with the runtime components. The **JVM** is the abstract machine that loads and executes class files and defines runtime behavior. The **JRE** traditionally refers to the JVM plus the libraries and supporting files needed to run applications; since Java 9's modular runtime-image model, a separate JRE download is not universally provided.

In the common workflow, `javac` compiles source code into class files containing bytecode and metadata. The class loader locates definitions, the JVM links them (including verification and preparation), initializes classes when required, and executes bytecode using interpretation and/or just-in-time compilation. The exact execution strategy is an implementation detail.

“Platform independent” means compatible bytecode can run on a compatible JVM across platforms. It does not remove dependencies on native libraries, OS behavior, permissions, locale, time zones, or environment configuration.

### Q2. Is Java pass-by-value or pass-by-reference?

Java is **always pass-by-value**. For a primitive, the method receives a copy of the primitive value. For an object argument, the method receives a copy of the reference value, so both references can point to the same object.

That allows a method to mutate the referenced object's state, but reassigning its parameter does not change the caller's variable. This distinction explains why a method can modify a passed-in list but cannot make the caller's list variable refer to a newly created list simply by assigning to its parameter.

### Q3. What is the difference between `==` and `equals()`?

For primitives, `==` compares values. For reference types, `==` checks whether both references identify the same object. `equals()` expresses logical equality according to the class's contract; the default implementation inherited from `Object` behaves like identity equality unless overridden.

For example, two distinct `String` objects can be equal according to `equals()` while `==` is false. Use logical equality for values and domain objects when their class defines it. Be careful with arrays: their inherited `equals()` does not compare contents; use `Arrays.equals` or `Arrays.deepEquals` as appropriate.

### Q4. Why is String immutable, and when should StringBuilder or StringBuffer be used?

A `String` cannot have its character content changed after construction. Immutability makes strings safe to share, supports string-pool reuse, simplifies their use as map keys, and reduces synchronization needs. Operations such as concatenation create a resulting string rather than modifying the original.

Use `StringBuilder` for repeated modifications within one thread, such as constructing output in a loop. `StringBuffer` synchronizes its individual methods, but that does not automatically make a multi-step workflow atomic. For simple concatenation expressions, the compiler may optimize construction; choose based on the actual operation rather than assuming every `+` is expensive.

### Q5. What is the contract between equals() and hashCode()?

If two objects are equal according to `equals()`, they must return the same `hashCode()`. Unequal objects may share a hash code, although a good implementation aims to distribute values well. The hash code must remain consistent while the information used by equality remains unchanged.

Hash-based collections use the hash to find a candidate bucket and equality to identify the key. If equality is overridden without a compatible `hashCode()`, lookups can fail even for logically equal keys. Mutable fields used in equality or hashing are particularly dangerous: changing them after insertion can make an entry difficult to find or remove.

### Q6. What is the difference between checked and unchecked exceptions?

Checked exceptions must be caught or declared by the method signature, except for the special treatment of unchecked exceptions. They often represent recoverable conditions that callers are expected to consider, such as certain I/O failures. Unchecked exceptions are subclasses of `RuntimeException` or `Error`; runtime exceptions often indicate invalid arguments, illegal state, or programming errors, while `Error` generally signals serious conditions an application should not casually try to recover from.

The choice is an API-design decision, not simply “checked is good, unchecked is bad.” Use meaningful exception types, preserve the cause when wrapping, and avoid catching an exception unless the code can recover, translate it usefully, add context, or handle it at an appropriate boundary. Do not swallow failures or use exceptions as ordinary control flow.

### Q7. What is the difference between an interface and an abstract class?

An interface defines a contract that unrelated classes can implement, and a class can implement multiple interfaces. Interfaces may include abstract methods, default methods, static methods, and private helper methods; they do not provide per-instance state in the way ordinary classes do.

An abstract class can hold instance state, constructors, and shared protected or public implementation, but a class can extend only one class. Prefer interfaces for capabilities and replaceable contracts. Use an abstract base class when closely related types genuinely share state or implementation that belongs in a common inheritance hierarchy. Neither is automatically better; avoid inheritance solely to reuse a few methods.

---

## 2. Object-oriented design and contracts

### Q8. Explain encapsulation, abstraction, inheritance, and polymorphism with practical meaning.

**Encapsulation** protects an object's invariants by controlling access to state and behavior. Private fields alone are not enough if methods expose mutable internals or allow invalid state transitions. **Abstraction** exposes a useful contract while hiding details callers do not need. **Inheritance** models a valid subtype relationship and reuses or specializes behavior. **Polymorphism** lets callers use a common type while the runtime dispatches an overridden instance method to the implementation appropriate for the actual object.

In backend code, an interface such as `PaymentProvider` can hide whether the implementation calls one external processor or another. That is useful abstraction and polymorphism. The design is only sound if implementations honor the same behavioral contract; a subtype that violates caller expectations is not a good substitute merely because it compiles.

### Q9. What is the difference between overloading and overriding?

**Overloading** means declaring methods with the same name but different parameter lists. The compiler selects the overload using the compile-time types and overload-resolution rules. Return type alone cannot distinguish overloads.

**Overriding** means a subtype provides an implementation of an inherited instance method with a compatible signature and return type. For ordinary virtual instance methods, the implementation is selected at runtime based on the actual object's class. Static methods are hidden rather than overridden, and fields are hidden rather than dynamically dispatched. This is why a reference's declared type can determine which field or static method is accessed while the object's class determines the overridden instance method.

### Q10. What are SOLID principles, and how should they influence design?

- **Single Responsibility:** A module should have one coherent reason to change.
- **Open/Closed:** Extend behavior without repeatedly modifying stable, well-tested core logic where practical.
- **Liskov Substitution:** A subtype must honor the behavioral expectations of its base type.
- **Interface Segregation:** Clients should not depend on methods they do not need.
- **Dependency Inversion:** High-level policy should depend on abstractions rather than directly on low-level implementation details.

SOLID is a set of design heuristics, not a requirement to create an interface for every class. Applying it well reduces change coupling and improves testability. Applying it mechanically can produce excessive indirection, fragmented interfaces, and abstractions that have only one trivial implementation. Use the principles to explain concrete design decisions and trade-offs.

### Q11. Why are immutability and defensive copying useful?

An immutable object cannot be changed after construction, so its invariants are easier to reason about and its state can be shared without coordinating later mutations. Immutability is especially useful for value objects, configuration, map keys, and data crossing concurrency boundaries.

A class must protect its internal state, not merely declare fields `final`. A final reference can still point to a mutable list. Copy mutable inputs on the way in and avoid returning mutable internal objects directly; expose immutable views or copies where appropriate. Deep immutability may require copying nested mutable values too, so understand ownership rather than relying on a shallow copy.

---

## 3. Collections and data structures

### Q12. How does HashMap work internally?

A `HashMap` uses a key's hash information to locate a bucket and then uses equality to find the matching key among candidates. Multiple keys can map to the same bucket, so a hash collision is not the same as key equality. The implementation resizes the table as it grows, redistributing entries to maintain useful lookup performance.

Average lookup and insertion are commonly described as O(1) under reasonable hash distribution; this is not an unconditional worst-case guarantee. Modern OpenJDK implementations can use tree bins for heavily populated buckets under certain capacity and threshold conditions. Treat those internal thresholds as implementation details rather than portable API guarantees. Pre-sizing may reduce resizing when the expected size is known, but excessive capacity wastes memory.

### Q13. Why must HashMap keys have stable equals() and hashCode() values?

A map uses the key's hash to find where its mapping belongs. If fields used by `hashCode()` or `equals()` change after insertion, a later lookup can search a different bucket or fail to match the stored key. The mapping may still physically exist but become practically inaccessible through the mutated key.

Prefer immutable keys such as strings, enums, records whose components are themselves suitably immutable, or dedicated value objects. If mutable objects must be used, ensure the fields participating in equality and hashing do not change while the object is a key.

### Q14. Compare ArrayList and LinkedList.

`ArrayList` stores elements in a resizable array. Indexed reads are O(1), and appending is amortized O(1); inserting or removing near the beginning or middle usually shifts elements and costs O(n). It has good memory locality and is generally the default choice for list workloads.

`LinkedList` stores nodes linked together. Inserting or removing a node is O(1) once the position/node is known, but finding an index or position is O(n), and each node adds memory and pointer-chasing overhead. As a result, LinkedList is not automatically faster for frequent insertions. Choose from the actual access pattern; for most ordinary list use, ArrayList is the more practical default.

### Q15. Compare HashSet, LinkedHashSet, and TreeSet.

`HashSet` provides uniqueness according to `equals()` and `hashCode()`, with no iteration-order guarantee. `LinkedHashSet` preserves a defined encounter order, typically insertion order, at additional memory cost. `TreeSet` keeps elements sorted using natural ordering or a comparator, typically with O(log n) basic operations.

For a sorted set, ordering must be consistent with the intended notion of uniqueness. If a comparator returns zero for two values, TreeSet treats them as the same set element even if their `equals()` methods say otherwise. Pick the implementation based on uniqueness, ordering, and performance needs.

### Q16. What is the difference between HashMap and ConcurrentHashMap?

`HashMap` is not thread-safe for concurrent structural modification. Concurrent access that includes writes requires external coordination or a concurrent collection. `ConcurrentHashMap` is designed for concurrent access and provides thread-safe operations with implementation techniques that allow more concurrency than locking the entire map for every operation.

Use atomic methods such as `computeIfAbsent`, `compute`, `merge`, or `putIfAbsent` when the operation must combine lookup and update safely. A sequence like `if (!map.containsKey(k)) map.put(k, v)` is not atomic just because the map itself is concurrent. Also, thread safety of the map does not make mutable objects stored as values thread-safe.

### Q17. What are fail-fast iterators, and can they guarantee thread safety?

Some collection iterators detect structural modification outside the iterator's supported mutation methods and may throw `ConcurrentModificationException`. This is a best-effort bug-detection mechanism, not a synchronization guarantee. Code must not depend on the exception always occurring.

For single-threaded removal during iteration, use the iterator's `remove()` when supported or use appropriate collection methods. For concurrent workloads, choose a concurrent collection or coordinate access explicitly. For example, `CopyOnWriteArrayList` provides snapshot-style iteration that is useful for read-heavy, write-light workloads but can be expensive when writes are frequent.

### Q18. When should a Queue, Deque, PriorityQueue, or BlockingQueue be used?

A `Queue` represents elements waiting to be processed; a `Deque` supports operations at both ends and can act as a queue or stack. A `PriorityQueue` retrieves elements according to priority/order rather than insertion order, so it is useful for scheduling or selecting the next minimum/maximum. A `BlockingQueue` supports producer-consumer coordination by waiting when appropriate conditions are not met.

Choose based on the ordering and coordination contract. A priority queue is not a sorted list for iteration, and a thread-safe queue alone does not bound the total work accepted by a service. In production, capacity, rejection/backpressure, and shutdown behavior matter as much as the data structure.

---

## 4. Generics, exceptions, and functional programming

### Q19. What are generics, and what is type erasure?

Generics let APIs express relationships between types at compile time, reducing casts and making invalid combinations harder to represent. For example, `List<String>` communicates that the list is intended to contain strings.

Java implements most generic type information through **type erasure**: type parameters are erased to their bounds or `Object` in the runtime representation, with compiler-generated casts and bridge methods where needed. This is why `new T()` is not generally allowed, `List<String>` and `List<Integer>` cannot be distinguished using `instanceof` at runtime in the ordinary way, and generic types are invariant. Wildcards such as `? extends T` and `? super T` express safe producer/consumer relationships.

### Q20. What is the difference between List<? extends T> and List<? super T>?

`? extends T` means the list's element type is some unknown subtype of T. It is useful when reading T values from a producer, but you generally cannot add a non-null T because the actual list could be a list of a narrower subtype.

`? super T` means the element type is T or one of its supertypes. It is useful when writing T values to a consumer, though values read back are only safely known as `Object`. This is the basis of PECS: **Producer Extends, Consumer Super**. It is a useful guideline rather than a substitute for understanding the exact type relationship.

### Q21. What is the difference between map() and flatMap() in streams?

`map()` transforms each input into one output, while `flatMap()` transforms each input into a stream-like result and flattens the nested results into one stream. For example, mapping a list of orders to each order's line items naturally creates nested collections; flatMap turns them into one stream of line items.

Use `map` for one-to-one transformations and `flatMap` for one-to-many or nested-result transformations. If a simple loop is clearer or avoids awkward state, it is perfectly reasonable to use a loop instead of forcing a stream pipeline.

### Q22. How do intermediate and terminal Stream operations differ, and why does laziness matter?

Intermediate operations such as `filter`, `map`, and `sorted` build a pipeline and are generally lazy. Terminal operations such as `collect`, `reduce`, `count`, and `findFirst` trigger evaluation. Short-circuiting terminals can stop once the result is known, so not every element necessarily passes through every stage.

Do not rely on side effects inside intermediate operations for correctness. The Stream contract allows implementations to elide behavioral-parameter execution when it cannot affect the terminal result. Prefer transformations that are stateless and side-effect-free, and use explicit terminal operations or loops for necessary effects.

### Q23. When should parallel streams be used, and what can go wrong?

Parallel streams split work across tasks, often using the common ForkJoinPool. They can help when the dataset is large, the work per element is substantial, operations are independent, and the source splits efficiently. They can hurt when tasks are small, the source is difficult to split, operations block on I/O, shared mutable state creates contention, or preserving encounter order is expensive.

Parallelism is not a free speedup. Measure with representative data and realistic contention. Avoid parallel streams for blocking network/database calls without careful resource and executor design; a shared common pool can interfere with unrelated application work.

### Q24. What are Optional's intended uses and limitations?

`Optional<T>` models a value that may be present or absent and can make return-value absence explicit. Methods such as `map`, `flatMap`, `orElse`, `orElseGet`, and `orElseThrow` support composition without repeated null checks.

A common trap is that `orElse(expression)` evaluates its argument eagerly, while `orElseGet(supplier)` evaluates the supplier only when the Optional is empty. Optional is generally most useful as a return type; using it indiscriminately for fields, method parameters, or collections of optional values can complicate APIs and serialization. It does not replace validation or exception handling for operational failures.

### Q25. What makes a Stream reduce operation correct?

A reduction combines elements using an identity and an accumulator, and parallel reduction may partition the input and combine partial results. For correctness, the reduction operation must obey the required associativity/identity rules; otherwise sequential and parallel execution can produce different or invalid results.

Summing numbers is a natural reduction. Building a mutable collection with an incorrect `reduce` is not; use `collect` and a proper collector for mutable reduction. When results differ under parallel execution, inspect side effects, non-associative operations, encounter-order assumptions, and shared mutable state.

---

## 5. Concurrency and the Java Memory Model

### Q26. What is the difference between a process and a thread?

A process is an operating-system execution environment with its own address space and resources. Threads within a process share heap memory and many process resources but have their own execution state and stacks. Sharing memory makes communication efficient, but also creates the possibility of data races and visibility bugs.

Java's `Thread` API abstracts thread behavior, while the runtime maps platform threads to operating-system threads and schedules virtual threads using runtime-managed mechanisms. Avoid assuming every Java thread corresponds one-to-one with an OS thread.

### Q27. What is the difference between synchronized and volatile?

`synchronized` provides mutual exclusion for a monitor and establishes visibility/order guarantees when a monitor is unlocked and later locked. It can protect compound invariants by ensuring only one thread executes a critical section at a time.

`volatile` provides visibility and ordering for reads/writes to that variable but does not make compound read-modify-write operations atomic. A volatile flag can publish state safely under a well-defined protocol; a shared counter increment usually needs an atomic class or a lock. Choose based on whether the requirement is visibility alone or an indivisible multi-step state transition.

### Q28. What does happens-before mean?

Happens-before is a memory-model relationship that guarantees visibility and ordering between actions. Program order within one thread contributes to it; synchronization constructs create cross-thread relationships. For example, actions before a thread starts happen-before actions in that thread, and a thread's actions happen-before another thread successfully returns from joining it.

A volatile write happens-before a subsequent read of that same variable under the specified rules, and unlocking a monitor happens-before a later lock of that monitor. These are contractual guarantees, not guesses about CPU caches. Reason about concurrency by identifying the synchronizing edge that makes one thread's writes visible to another.

### Q29. Why is count++ not atomic, even if count is volatile?

Increment is a read-modify-write sequence: read the current value, calculate the next value, then write it. Two threads can read the same value and both write the same incremented result, losing one update. Volatile guarantees visibility and ordering of the individual accesses, not atomicity of the whole sequence.

Use `AtomicInteger.incrementAndGet()` for a simple atomic counter, `LongAdder` for high-contention statistical counters where sum consistency requirements allow it, or a lock when the update belongs to a larger invariant. The choice depends on semantics, not just speed.

### Q30. What is the difference between Runnable, Callable, Future, and CompletableFuture?

`Runnable` represents work that does not return a result through its method signature; `Callable<V>` can return a value and throw checked exceptions. `Future<V>` represents a result that may become available later and supports waiting and cancellation, but its basic API can be awkward for composing multiple asynchronous stages.

`CompletableFuture` supports composition, combining results, and explicit completion. Methods ending in `Async` may use a default or supplied executor, depending on the overload. Always handle failures and understand which executor runs each stage; otherwise, blocking work can occupy the common pool or exceptions can remain hidden until a result is observed.

### Q31. Why use ExecutorService instead of creating threads directly?

An `ExecutorService` separates task submission from execution policy and provides lifecycle management, result tracking, and controlled concurrency. A bounded platform-thread pool can cap active work and queue tasks, but its queue and rejection policy must be chosen deliberately. An unbounded queue can turn overload into growing latency and memory use.

Use a suitable executor for the workload, propagate cancellation and interruption correctly, and shut it down gracefully. A pool is not just a performance setting: it is part of the service's capacity and backpressure design. Avoid creating a new platform thread for every task under high load without understanding the cost and concurrency limits.

### Q32. What causes deadlocks, and how can they be prevented or diagnosed?

A deadlock occurs when threads wait indefinitely for resources held by one another. A common pattern is inconsistent lock ordering: one thread acquires A then waits for B, while another acquires B then waits for A. Deadlocks can also involve locks combined with callbacks, futures, database locks, or external resources.

Prevent them by enforcing a global lock order, keeping critical sections small, avoiding blocking calls while holding locks, and using timeouts where the semantics permit. Diagnose using thread dumps and identify the wait-for cycle rather than blindly restarting or changing pool sizes. A timeout may reduce indefinite waiting but does not automatically make a partially completed operation safe to retry.

### Q33. What are race conditions, and how do concurrent collections help?

A race condition occurs when a result depends on the timing or interleaving of operations. A data race is a more specific Java Memory Model concept involving conflicting unsynchronized accesses. Not every race is a data race, and both can lead to incorrect application behavior.

Concurrent collections provide thread-safe operations and specified visibility guarantees for their own data structures. They do not automatically make multi-operation workflows atomic or protect mutable objects stored inside them. Use atomic compound methods such as `compute` where applicable, external locks for larger invariants, or immutable values and clear ownership boundaries.

### Q34. What are virtual threads, and when should they be used?

Virtual threads became a final feature in Java 21. They are lightweight Java threads scheduled by the runtime over a smaller set of carrier platform threads. They make thread-per-task code practical for applications with many concurrent tasks that spend substantial time waiting on blocking I/O.

They improve scalability/throughput for suitable workloads, not the speed of CPU-bound computation and not the capacity of downstream services. Use a virtual thread per task rather than pooling virtual threads as if they were scarce OS threads, but still limit access to scarce resources such as database connections, API quotas, and memory. Check the target JDK's guidance for pinning and observability details.

### Q35. How should interruption and cancellation be handled?

Interruption is a cooperative cancellation signal, not a mechanism that forcibly kills arbitrary code. Blocking methods may throw `InterruptedException`, typically clearing the interrupt status when they do. Code that cannot propagate the exception should usually restore the status with `Thread.currentThread().interrupt()` after performing any necessary cleanup.

Cancellation must be designed through the call chain: tasks need to notice the signal, release resources, and avoid committing partial results incorrectly. Cancelling a Future does not guarantee that underlying I/O or a remote operation has stopped. Treat timeout, cancellation, and unknown remote outcome as distinct states when deciding whether retry is safe.

### Q36. What are CountDownLatch, Semaphore, and BlockingQueue used for?

A `CountDownLatch` lets one or more threads wait until a count reaches zero; it is one-shot. A `Semaphore` controls access to a limited number of permits and is useful for bounding concurrent use of a resource. A `BlockingQueue` coordinates producers and consumers and can apply backpressure when bounded.

Choose the primitive that expresses the invariant. A semaphore does not itself manage a resource lifecycle, and a queue does not guarantee that downstream processing succeeds. Always consider interruption, shutdown, queue capacity, permit release in `finally`, and failure handling.

---

## 6. JVM, memory, and performance

### Q37. What are heap and stack memory, and what causes StackOverflowError versus OutOfMemoryError?

The heap holds objects and arrays managed by the JVM, while each thread has execution-stack frames containing local variables, operand-stack state, and call information. The JVM specification defines abstract runtime areas; exact object layout and memory allocation strategies are implementation-dependent, and a JIT may optimize allocations when observable behavior is preserved.

`StackOverflowError` commonly results from excessive call depth, such as unbounded recursion. `OutOfMemoryError` indicates that the JVM could not satisfy an allocation or another memory-related requirement; causes include heap exhaustion, native-memory pressure, too many threads, or limits imposed by the environment. Diagnose the specific error and its evidence instead of assuming every memory incident is a heap leak.

### Q38. How does garbage collection determine which objects can be reclaimed?

Tracing garbage collectors begin from GC roots—runtime references such as those associated with live threads, stack frames, static references, and JVM internals—and follow reachable references. Objects not reachable from the roots are eligible for reclamation, subject to reference processing and collector behavior.

A Java memory leak usually means an object remains reachable through an unintended reference, not that code forgot to call free. Common causes include unbounded caches, static collections, listener registrations that are never removed, ThreadLocal values, and retaining request data. A heap dump and dominator/retained-size analysis can help locate the first unwanted retaining path.

### Q39. What is the difference between throughput and low-latency garbage collectors?

Collectors trade off application throughput, pause behavior, CPU use, memory overhead, and operational complexity. Some prioritize overall throughput; others are designed to reduce pause times and perform more work concurrently. Collector names and behavior evolve across JDK versions, and terms such as “minor” and “major” collection do not mean the same thing for every collector.

Choose based on measured workload and service-level objectives. Compare pause percentiles, allocation rate, live-set size, CPU, memory footprint, and tail latency under representative load. A low average pause can hide a rare long pause, while a low-pause collector can still reduce throughput or consume additional resources.

### Q40. What is the difference between memory leak and high memory usage?

High memory usage is an observation, not a diagnosis. It may reflect a legitimate live set, allocation bursts, cache warming, heap sizing, native memory, thread stacks, direct buffers, or a leak. A leak is suggested when retained memory grows over time beyond expected workload behavior and objects remain reachable without a valid business reason.

Start with trends and evidence: heap usage after collection, allocation rate, GC logs, heap dumps, class histograms, native-memory tracking where enabled, and thread counts. Identify whether memory is retained in the Java heap or outside it before changing heap settings. Increasing the heap may delay symptoms without fixing an unbounded retention path.

### Q41. How do you investigate a slow Java service?

First establish the symptom: increased latency, reduced throughput, CPU saturation, blocked requests, allocation pressure, GC pauses, lock contention, or downstream delay. Compare the incident against a known baseline and inspect metrics and traces to narrow the scope before changing runtime flags.

Use thread dumps for blocked or stuck threads, Java Flight Recorder (JFR) for runtime events and allocation/CPU evidence, GC logs for collection behavior, and a profiler for hotspots. Check database and remote-call latency too; a Java stack may be waiting on a downstream dependency rather than consuming CPU. Form a hypothesis, gather evidence, apply a reversible mitigation, and verify the effect.

### Q42. What is JIT compilation, and why can a Java application behave differently after warm-up?

The JVM may initially interpret bytecode and later compile frequently executed code into native machine code. The JIT uses runtime information to optimize hot paths, inline calls, eliminate redundant work, and apply other transformations while preserving Java's specified behavior. It can also deoptimize code when assumptions stop holding.

As a result, startup performance can differ from steady-state performance. Microbenchmarks must account for warm-up, dead-code elimination, constant folding, allocation, and measurement noise. Use JMH for microbenchmarks rather than ad hoc loops, and validate performance at application level because a faster micro-operation may not improve end-to-end latency.

### Q43. What is the difference between an OutOfMemoryError and a memory leak in production diagnosis?

An `OutOfMemoryError` is a failure symptom; a leak is one possible cause. The error can occur because of a legitimately oversized live set, a constrained container, native-memory exhaustion, direct-buffer usage, thread creation, or inability to allocate a particular region. Different error messages and runtime evidence point to different areas.

Capture the relevant heap dump or diagnostic evidence when safe and configured, inspect GC logs and container limits, and distinguish Java heap from native memory. Do not automatically increase `-Xmx`: in a container, the total budget also includes metaspace, code cache, direct buffers, thread stacks, native libraries, and other process memory.

---

## 7. Modern Java

### Q44. Which Java 8 features are most important for backend interviews?

The most important topics are lambda expressions, functional interfaces, method references, the Stream API, `Optional`, the `java.time` date/time API, default methods in interfaces, and CompletableFuture. These features changed how Java code expresses data transformations, optional results, dates/times, and asynchronous composition.

The interview value lies in trade-offs, not just syntax. Streams can improve pipeline readability but become harder to debug when overloaded with side effects. Optional can make absence explicit but should not be applied mechanically to every field. CompletableFuture supports composition but requires deliberate executor, timeout, and exception handling.

### Q45. What changed in Java 9–11 that a backend engineer should know?

Java 9 introduced the module system, collection factory methods such as `List.of`, and other API/runtime changes. Java 10 added local-variable type inference with `var` for local variables. Java 11, an LTS release, added useful APIs including standard HTTP Client (introduced earlier as incubating/preview and finalized in Java 11), and it removed several Java EE and CORBA modules from the JDK distribution.

Version details matter when migrating. `var` does not make Java dynamically typed; the compiler infers a static type. Factory-created collections are unmodifiable, not deeply immutable if they contain mutable elements. Check API availability and behavior against the target JDK rather than assuming code that compiles on a recent JDK will run on Java 8.

### Q46. What are records and sealed classes, and when are they useful?

Records, finalized in Java 16, provide a concise way to model transparent data carriers with component fields, accessors, and generated methods such as `equals`, `hashCode`, and `toString`. They are implicitly final and are not a general replacement for every mutable domain entity. Their component references are final, but referenced objects can still be mutable.

Sealed classes and interfaces, finalized in Java 17, restrict which types may extend or implement them. They are useful when a domain has a known set of variants, such as a result type with success and failure cases. Together with pattern matching, they can improve exhaustive handling, but only if the type hierarchy and null behavior are designed deliberately.

### Q47. What are the most important Java 21 features?

Java 21 finalized virtual threads, record patterns, pattern matching for switch, and sequenced collections, among other features. It also delivered generational ZGC. Structured concurrency and scoped values were preview features in Java 21, not finalized APIs in that release.

For backend work, virtual threads are especially relevant to high-concurrency blocking-I/O services. Pattern matching can simplify branching over structured data. Sequenced collections standardize operations for collections with a defined encounter order. Always label preview features accurately and check whether the exact target JDK finalizes or changes an API.

### Q48. What should an experienced developer know about Java 25?

Java 25 is an LTS release. For interviews, know the main language and runtime evolution since Java 17/21, but avoid presenting every release feature as equally important to backend development. Review the official Java 25 release notes and JEPs for finalized versus preview features, APIs that changed, and migration implications.

A strong answer should explain the impact on code, deployment, observability, and compatibility rather than reciting a feature list. In production, confirm the exact vendor distribution, runtime flags, dependency support, container base image, and test matrix before upgrading. Do not use a newer language feature in code that must still compile on an older source/target release.

---

## 8. Production and senior-level scenarios

### Q49. A HashMap lookup unexpectedly returns null even though the key was inserted. What would you investigate?

First confirm that the same map instance is being queried and that the insertion succeeded. Then inspect the key's `equals()` and `hashCode()` implementations, whether fields used by them changed after insertion, whether a different but logically equivalent key is being used, and whether null is being confused with an absent mapping. `containsKey` can distinguish a present mapping to null from no mapping, though the larger issue may be key design.

Also check concurrency: unsynchronized writes to a shared HashMap are not safe. Reproduce with a minimal example and inspect the key's hash and equality behavior before replacing the data structure. The long-term fix is usually stable key semantics and correct synchronization, not retries.

### Q50. An API becomes slow under load, but CPU usage is low. How would you investigate?

Low CPU with high latency often suggests waiting rather than computation: thread pool exhaustion, blocked I/O, database connection-pool saturation, slow downstream services, lock contention, or requests queued behind a constrained resource. Compare request traces, active/queued threads, pool metrics, downstream latency, timeout counts, and thread dumps.

Check whether the service accepts work faster than it can complete it. Increasing the thread pool can worsen the problem if all tasks wait for the same limited database connections. Identify the bottleneck, apply backpressure or capacity limits, and tune the constrained resource based on measurements.

### Q51. A service experiences repeated garbage-collection pressure. What steps would you take?

Determine whether the issue is high allocation rate, a growing retained live set, an undersized heap, or pauses caused by the selected collector and workload. Review GC logs, allocation profiling, post-collection occupancy, request throughput, object lifetime, and container memory limits. JFR and heap dumps can provide complementary evidence.

Look for avoidable temporary allocations, unbounded caches, large buffers, and retained request data. Change one variable at a time and compare the same workload before and after. Collector or heap tuning can help, but should not conceal a leak or a capacity problem.

### Q52. How do you choose between synchronization, atomics, and concurrent collections?

Start with the invariant being protected. Use `synchronized` or a lock when multiple fields or operations must change as one consistent unit. Use atomic variables for small state transitions such as counters or compare-and-set updates. Use concurrent collections when the shared data structure's concurrency contract matches the workload.

Avoid choosing solely by benchmark claims. Atomics can suffer contention and are not a replacement for multi-field invariants; concurrent maps do not make their values thread-safe; locks can be simpler and more correct for compound updates. Make the correctness requirement explicit, then measure if performance is a concern.

### Q53. A task submitted to an executor never appears to finish. What would you check?

Inspect thread dumps and executor metrics to determine whether the task is running, blocked, queued, deadlocked, or waiting on a future that cannot complete. Check pool size, queue growth, rejection policy, nested task submission, and whether all pool threads are waiting for work submitted to the same saturated pool—a form of thread-starvation deadlock.

Observe task exceptions and completion paths; a task may have failed while the caller waits elsewhere. Set realistic timeouts, avoid blocking a pool's workers on tasks that require the same exhausted pool, and ensure cancellation and shutdown are handled. Increasing pool size without identifying the wait cycle can hide the symptom or make resource pressure worse.

### Q54. How should retries be designed when a remote call times out?

A timeout means the caller did not receive a result within the allowed time; it does not prove the remote operation failed. The remote side may have completed the operation while the response was delayed or lost. Blind retries can duplicate payments, orders, or other side effects.

Use bounded retries with backoff and jitter only when the operation is safe to retry or protected by an idempotency mechanism. Set an overall deadline, distinguish transient from permanent failures, and consider circuit breaking and load shedding. For writes, define how the client can query or deduplicate the outcome instead of assuming timeout equals failure.

---

## 9. Additional specialist topics

### Q55. How should Java I/O, NIO, and resource management be handled?

Classic `java.io` APIs provide stream- and reader/writer-oriented access, while `java.nio` adds buffers, channels, paths, and facilities for more flexible file and network I/O. NIO does not mean every operation is non-blocking: channels and APIs have different blocking and asynchronous contracts, so the chosen abstraction must match the workload.

Use try-with-resources for resources implementing `AutoCloseable`, such as streams, readers, channels, and JDBC objects. This ensures cleanup on normal and exceptional paths and preserves suppressed exceptions when close itself fails. Choose explicit charsets such as UTF-8 instead of relying on the platform default, and distinguish bytes, UTF-16 code units, Unicode code points, and user-perceived characters when processing text.

### Q56. What are the risks of Java native serialization?

Java's built-in object serialization can reconstruct object graphs from a byte stream, but deserializing untrusted input is dangerous because the stream can cause classes to be instantiated and methods such as deserialization hooks to execute in surprising ways. It also couples the data format to class structure and versioning details.

Avoid native Java deserialization for untrusted data where possible. Prefer an explicit, schema-oriented format such as JSON, Protocol Buffers, or another format appropriate to the system, and apply authentication, size limits, validation, and versioning. If native serialization must be used, apply strict deserialization filters and treat the input as a security boundary. Serialization is not encryption and does not establish trust.

### Q57. What should a Java backend engineer understand about JDBC transactions and connection pools?

JDBC provides lower-level database access through connections, prepared statements, and result sets. A connection's transaction behavior depends on autocommit and explicit commit/rollback; resource cleanup is essential, and prepared statements should be used for parameter binding rather than concatenating untrusted values into SQL.

In a pooled application, the pool manages physical connections while application code borrows and returns them. Do not close a shared pool itself from request code, hold connections while waiting on remote services, or assume a timed-out client request means the database statement stopped. Align connection, query, and transaction timeouts, and ensure rollback/cleanup occurs when failures happen.

### Q58. What are reflection, annotations, and the Java module system used for?

Annotations attach metadata to program elements; reflection can inspect classes, methods, fields, and annotations at runtime. Frameworks use these mechanisms for dependency injection, mapping, serialization, and test discovery. Annotation processing can also inspect source-level metadata at compile time and generate code before runtime.

The Java Platform Module System (JPMS), introduced in Java 9, adds explicit module dependencies and package exports/opens. Strong encapsulation means reflective access to non-open packages can be restricted, even when code previously relied on deep reflection. Use reflection deliberately, account for module boundaries, and prefer public APIs or compile-time processing when that is simpler and safer.

### Q59. Which design patterns are especially useful in Java backend development?

Patterns are reusable ways to structure recurring design problems, not a checklist to apply everywhere. **Strategy** makes an algorithm replaceable behind a common contract; **Factory** centralizes object creation when construction choices vary; **Decorator** adds behavior around an object while preserving its interface; **Builder** simplifies construction when an object has many optional parameters; and **Adapter** translates one interface into another.

Spring itself uses familiar patterns such as proxies, factories, templates, and dependency injection. The interview answer should connect a pattern to the problem it solves and its trade-off. For example, Strategy can remove a large conditional by selecting an implementation, but it can also create unnecessary classes if the variation is trivial and unlikely to grow.

### Q60. How should Java code be benchmarked and performance-tuned?

Start by defining the observed problem and a measurable target, then profile a representative workload to find the dominant cost. Use JMH for JVM microbenchmarks because warm-up, JIT compilation, dead-code elimination, constant folding, allocation, and measurement noise can invalidate naive timing loops. For application-level performance, use production-like load tests and observability rather than extrapolating from one microbenchmark.

Change one variable at a time and compare results against a baseline. Do not tune GC flags, object allocation, synchronization, or data structures based on intuition alone. Include latency percentiles, throughput, CPU, allocation rate, memory, and contention in the evaluation; an optimization that improves average throughput may still worsen tail latency or resource use.

### Q61. What should Java testing and build-tool compatibility cover?

Tests should cover behavior and contracts at appropriate levels: unit tests for isolated logic, integration tests for interactions, and concurrency tests designed around synchronization rather than arbitrary sleeps. JUnit and Mockito are common tools, but mocks should not replace tests of important framework wiring, database constraints, or real serialization behavior. Use deterministic coordination primitives and bounded timeouts when testing concurrent code.

Build configuration is also part of compatibility. Understand Maven/Gradle toolchains, the compiler's `--release` option, test plugins, dependency versions, and the difference between the JDK used to run the build and the bytecode/API level targeted by compilation. A project that compiles on a developer's newest JDK may still fail on the production runtime if its source/target/API compatibility is not configured and tested correctly.

---

### Q62. What security mistakes should Java backend developers avoid?

Security depends on the application's trust boundaries, not just on using a particular Java API. Validate and constrain untrusted input at the appropriate boundary, use parameterized SQL rather than string concatenation, avoid unsafe deserialization of untrusted data, prevent path traversal by resolving and validating paths against an intended root, and do not disable TLS certificate/hostname verification to “fix” a connection problem.

Use established cryptographic APIs and libraries instead of inventing algorithms, store secrets outside source control, apply least privilege, and avoid logging credentials, tokens, or sensitive payloads. For outbound requests, consider SSRF risks including redirects and DNS resolution behavior. A strong answer explains the threat model and layered controls; no single normalization or validation step guarantees safety in every environment.

### Q63. What should be configured when making HTTP calls from Java?

A production HTTP client needs explicit connection and request/response timeouts, sensible connection reuse/pooling where supported, TLS verification, redirect policy, resource cleanup, and observability. The standard `java.net.http.HttpClient` became a standard API in Java 11; older code may use other client libraries with their own configuration and lifecycle rules.

Timeouts should align with the overall request deadline, and retries should be bounded and restricted to operations that are safe to repeat. A timeout only means the caller did not receive a response in time; the remote service may still have completed the operation. Record useful metrics and correlation context without leaking secrets, and avoid creating a new client for every request when the client is intended to be reused.

### Q64. How does class-loader delegation work, and why can class-loading problems occur?

Class loaders locate and define classes at runtime. The traditional parent-delegation model asks a parent loader to load a class before the child defines it, helping avoid duplicate definitions of core classes and supporting a consistent runtime type system. Application servers, plugin systems, module layers, and custom class loaders can use more complex arrangements.

Two classes with the same fully qualified name are not necessarily the same runtime type if they were defined by different class loaders. This can cause confusing `ClassCastException`, `NoClassDefFoundError`, or linkage errors in application servers and plugin architectures. Diagnose the actual class loader, classpath/module path, dependency conflicts, and loading origin rather than assuming the source code type name uniquely identifies the runtime class.

---

## 10. Expanded coverage: frequently tested Java topics

The earlier sections establish the core concepts. This section fills in additional language, API, runtime, and production topics that commonly appear as follow-up questions. Use it alongside the earlier sections rather than treating it as a separate syllabus.

### Strings, wrappers, and numeric behavior

### Q65. What is the difference between String, StringBuilder, and StringBuffer?

`String` is immutable: every apparent modification produces a new string value. That makes strings safe to share and useful as map keys, but repeated concatenation inside a loop can create many temporary objects. `StringBuilder` is mutable and usually the right choice for building text in one thread. `StringBuffer` offers synchronized methods for legacy thread-safe mutation, but synchronization does not automatically make a multi-step sequence of operations atomic.

Use `+` for simple readable concatenation; the compiler can optimize many such expressions. Prefer a builder when accumulating text repeatedly in a loop or when construction logic is complex. Do not choose `StringBuffer` by default merely because a service is multithreaded: a builder should generally be local to the operation rather than shared between threads.

### Q66. How does the String pool work, and what does intern() do?

The runtime maintains a pool of canonical strings. String literals and compile-time constant string expressions are interned, so equal literals can refer to the same pooled object. Calling `intern()` returns the canonical pooled representation for that string value, either an existing one or a pooled entry.

This is different from value equality: use `equals()` to compare string contents. Do not rely on reference identity as an application-level string comparison strategy. Interning arbitrary, high-cardinality input can also add complexity and memory pressure without a useful benefit; use it only when the workload and lifecycle make canonicalization worthwhile.

### Q67. Why is String a good key for HashMap, and what happens if a key is mutable?

A hash-based map expects a key's equality and hash code to remain stable while it is stored. String is immutable, so its content cannot change after insertion and invalidate the bucket chosen by the map. A mutable key can be inserted under one hash code and later searched for under another, making the entry effectively unreachable through normal lookup even though it remains in the map.

If a custom key represents several fields, base equality and hashing on the same logical identity fields and make those fields immutable where practical. If mutability is unavoidable, remove the key before changing its equality-relevant state and reinsert it afterward.

### Q68. Explain autoboxing, unboxing, and a common Integer comparison trap.

Autoboxing converts a primitive to its wrapper type, such as `int` to `Integer`; unboxing extracts the primitive value. Java guarantees caching for certain small integral wrapper values, but code must not depend on two boxed references being identical for arbitrary values. Therefore, `a == b` can compare references when both operands are wrappers, while `a.equals(b)` compares wrapper values.

Unboxing a null wrapper throws `NullPointerException`. Be explicit about nullability at API boundaries, and use primitive types when null has no meaning. For value comparisons, unbox safely or use the wrapper's value equality; don't use reference identity.

### Q69. What should you know about floating-point equality and BigDecimal?

Binary floating-point types such as `float` and `double` cannot represent many decimal fractions exactly. This can produce results such as a sum that is close to, but not exactly, the expected decimal value. For approximate scientific calculations, use a documented tolerance appropriate to the scale and domain rather than direct equality.

For decimal quantities requiring predictable decimal arithmetic, such as many financial calculations, `BigDecimal` is often appropriate. Constructing it from a decimal string avoids importing the binary approximation of a double. Also note that `BigDecimal.equals()` considers both value and scale, whereas `compareTo()` compares numerical value; this distinction matters for equality and sorted collections.

### Exceptions and resource safety

### Q70. When should you create a custom exception?

Create a custom exception when it communicates a meaningful domain or boundary failure that callers can handle differently, such as an invalid business operation or a missing domain resource. Include useful context, preserve the original cause when wrapping, and choose checked versus unchecked based on the contract rather than habit.

Avoid creating a new exception class for every low-level failure if callers cannot act on the distinction. Do not expose secrets, raw credentials, or sensitive payloads in exception messages. At service boundaries, translate implementation-specific exceptions into stable domain or API errors while retaining enough internal diagnostic context.

### Q71. What is exception chaining, and why should the cause be preserved?

Exception chaining wraps a lower-level exception in a higher-level exception while retaining the original cause. For example, a repository layer might translate a driver exception into a persistence-specific exception. The higher layer can then reason about the abstraction it owns, while logs and diagnostics retain the original failure and stack trace.

When wrapping, pass the original exception as the cause. Avoid logging the same exception at every layer and then rethrowing it, because that creates noisy duplicate logs. Log where the failure can be meaningfully handled or where ownership of the error is clear.

### Q72. How do try-with-resources and suppressed exceptions work?

Try-with-resources automatically closes resources that implement `AutoCloseable`, including when the body throws. Resources are closed in reverse order of initialization. If the body throws and closing also throws, the body exception remains primary and the close failure is recorded as a suppressed exception.

This is usually safer than manual `finally` cleanup, where a close exception can accidentally replace the original failure. Use it for files, streams, JDBC resources, and other closeable objects. When diagnosing failures, inspect suppressed exceptions too; the primary stack trace may not tell the whole story.

### Q73. Can a finally block change a method's return value or hide an exception?

Yes, but it should not. A `return` or a thrown exception from `finally` can override a return or exception from the `try` or `catch` block, which makes control flow surprising and can hide the original failure. The safe rule is to use `finally` for cleanup only and avoid returning or throwing from it unless there is a very specific, justified reason.

Try-with-resources is generally preferable for closeable resources because it preserves the primary exception and records close failures as suppressed exceptions.

### Collections and generics: deeper questions

### Q74. How do HashMap collisions work, and why does capacity matter?

A hash map uses a key's hash to select a bucket and then uses equality to distinguish keys that land in the same bucket. Collisions are expected; a correct map must still distinguish unequal keys with the same hash. In modern OpenJDK implementations, heavily populated buckets can be treeified under certain capacity and collision conditions, but the exact implementation details are not all part of the general Map contract.

Resizing occurs as the map grows beyond its threshold, redistributing entries across a larger table. If an approximate entry count is known, sizing the map sensibly can reduce resize work, but excessive preallocation wastes memory. Correct `equals()` and `hashCode()` implementations matter more than trying to avoid every collision.

### Q75. How do HashMap, LinkedHashMap, and TreeMap differ?

`HashMap` provides hash-based lookup and does not promise iteration order. `LinkedHashMap` maintains a predictable encounter order, usually insertion order, and can also maintain access order for patterns such as a simple LRU cache. `TreeMap` keeps keys ordered according to their natural ordering or a supplied comparator, with logarithmic-time basic operations.

Choose based on the contract you need: no order, predictable encounter order, or sorted keys. A comparator used by a sorted map should be consistent with equality when practical; otherwise the map's notion of duplicate keys can differ from `equals()`, which can surprise callers.

### Q76. What is the difference between Comparable and Comparator?

`Comparable` defines a type's natural ordering through `compareTo()`. `Comparator` defines an external ordering and allows multiple orderings for the same type, such as sorting employees by name, salary, or start date. Comparators can be composed using methods such as `comparing()` and `thenComparing()`.

Ordering must be transitive and consistent. In sorted sets and maps, comparison returning zero means the elements are treated as the same ordering key, even if their `equals()` methods say they differ. Document intentional differences and test boundary values, null handling, and tie-breakers.

### Q77. What are the trade-offs of CopyOnWriteArrayList and BlockingQueue?

`CopyOnWriteArrayList` copies its backing array on mutation, so reads and iteration are simple and iterators observe a snapshot. It works well for small collections with many reads and rare writes, but frequent updates can be expensive in allocation and copying. It is not a general replacement for `ArrayList`.

A `BlockingQueue` supports coordination between producers and consumers, including blocking when a bounded queue is full or empty. Bounded queues can provide backpressure and prevent unbounded buffering, but they require an explicit policy for saturation: block, reject, drop, or apply another domain-specific response.

### Q78. What is the difference between an unmodifiable collection and an immutable collection?

An unmodifiable view prevents mutation through that view, but the underlying collection may still be changed elsewhere. A truly immutable collection cannot have its contents changed after creation, although objects stored inside it may themselves be mutable. An unmodifiable wrapper around a mutable list is therefore not automatically a snapshot.

When exposing collection state from an object, consider defensive copying or immutable collection factories so callers cannot mutate internal state through an alias. Also consider whether element mutability can violate the class's invariants.

### Q79. Why can't you create arrays of parameterized types, and what is reifiable?

Java arrays are covariant and enforce their component type at runtime, while generic type parameters are mostly erased at runtime. Creating a `new List<String>[10]` would imply runtime checks that the JVM cannot reliably perform for the erased element type, so generic array creation is prohibited in the usual form.

A reifiable type is one whose runtime type information is fully available, such as a non-generic class or an array of reifiable component type. Generic code sometimes needs a collection instead of an array or a carefully encapsulated varargs operation with appropriate safety guarantees. Do not suppress unchecked warnings without understanding the invariant that makes the operation safe.

### Q80. What are bounded type parameters and generic methods useful for?

A bounded type parameter restricts the types that can be supplied and makes a useful contract available inside the implementation, such as requiring a type to implement `Comparable`. A generic method declares its own type parameter, allowing the same method to work safely across multiple related types without relying on raw types or casts.

Prefer a type signature that expresses the actual relationship between inputs and outputs. Avoid raw collections: they bypass compile-time checks and can cause failures far from the point where an unsafe value entered the system.

### Functional programming and everyday APIs

### Q81. What is the difference between a Predicate, Function, Consumer, and Supplier?

A `Predicate<T>` accepts a value and returns a boolean, commonly for filtering. A `Function<T,R>` transforms a value into another value. A `Consumer<T>` accepts a value and returns no result, often for an action or side effect. A `Supplier<T>` produces a value without taking an input.

These interfaces make behavior composable and are used by streams, collection APIs, and asynchronous APIs. Keep functions passed to stream pipelines focused and avoid hidden side effects, because lazy evaluation and possible parallel execution can make side-effect timing and ordering harder to reason about.

### Q82. Why are side effects in stream pipelines risky?

Streams describe a computation over a source; intermediate operations are lazy, and a terminal operation triggers evaluation. Side effects inside intermediate operations can be skipped when they do not affect the result, run in an unexpected order, or race when a pipeline is parallel. For example, mutating a shared list inside a parallel `forEach` is not automatically safe.

Prefer transformations and reductions that express the result directly. If the operation is inherently side-effecting, make the boundary explicit and use a suitable concurrency or transaction strategy rather than assuming the stream framework will serialize it.

### Q83. When should Optional be returned, and when should it not be used?

`Optional<T>` is useful as a return type when absence is a normal, expected result and callers should handle it explicitly. It is not a universal replacement for null: using it for every field, method parameter, or collection element can add unnecessary complexity and may not match framework serialization or persistence conventions.

Avoid calling `get()` without checking presence. Methods such as `map`, `flatMap`, `orElseGet`, and `orElseThrow` express intent clearly. Remember that `orElse` evaluates its argument eagerly, while `orElseGet` evaluates the supplier only when needed.

### Q84. What is the difference between orElse() and orElseGet()?

`orElse(value)` evaluates the argument before the call, even when the Optional contains a value. `orElseGet(supplier)` invokes the supplier only when the Optional is empty. If the fallback is cheap and already available, `orElse` is fine; if it performs expensive work or has side effects, `orElseGet` avoids unnecessary evaluation.

The key interview point is evaluation behavior, not merely syntax. A fallback that performs a database call should not be placed in `orElse` if it must only run when the value is absent.

### Q85. Why is the Java Date and Time API preferred over Date and Calendar?

The `java.time` API introduced in Java 8 provides clearer types for different concepts: `LocalDate` for a date without a time zone, `Instant` for a point on the UTC timeline, `LocalDateTime` for a local wall-clock date and time, and `ZonedDateTime` for a date-time associated with a time zone. Most of these types are immutable and thread-safe.

Choose the type based on the domain. A birthday is usually a date, an audit event is usually an instant, and a scheduled meeting often needs a local time plus a zone. A `LocalDateTime` alone does not identify a unique instant because it has no time-zone offset or daylight-saving rules.

### Concurrency and asynchronous programming

### Q86. What is the difference between synchronized, Lock, and atomic classes?

`synchronized` provides mutual exclusion and memory-visibility guarantees tied to monitor acquisition and release. Explicit `Lock` implementations provide capabilities such as interruptible acquisition, timed attempts, and multiple condition variables, but require disciplined unlocking, normally in a `finally` block. Atomic classes support specific thread-safe updates such as compare-and-set without requiring a monitor for that operation.

Choose the simplest mechanism that matches the invariant. Atomics are ideal for independent counters or state transitions, but they do not automatically make a multi-field invariant atomic. A lock or synchronized critical section may be clearer when several values must change together.

### Q87. What are atomicity, visibility, and ordering in concurrent code?

Atomicity means an operation appears indivisible; visibility means one thread's writes become observable to another; ordering concerns which operations may be observed before others. These are related but distinct. A volatile write/read pair establishes visibility and ordering guarantees, but does not turn a compound read-modify-write operation into an atomic one.

Use synchronization, volatile, atomics, or higher-level concurrency utilities according to the actual guarantee required. Avoid reasoning from how code appears to execute on one machine: the Java Memory Model defines which behaviors are permitted across threads.

### Q88. What is safe publication, and why does it matter?

Safe publication means making an object visible to other threads in a way that ensures they see its correctly initialized state. This can be achieved through mechanisms such as static initialization, a volatile reference, synchronization, or thread-safe concurrent utilities. Publishing an object before construction is complete can allow another thread to observe inconsistent state.

Immutability simplifies safe sharing, especially when fields are final and the object does not leak its reference during construction. But final fields do not excuse leaking `this` from a constructor or using unsafe publication for mutable surrounding state.

### Q89. How do wait(), notify(), and notifyAll() work, and why should wait be in a loop?

These methods coordinate threads using an object's monitor. A thread must own that monitor before calling `wait()`, `notify()`, or `notifyAll()`. Calling `wait()` releases the monitor while waiting and reacquires it before returning. A thread should check its condition in a loop because wakeups can be spurious and another thread may consume or change the condition before the awakened thread reacquires the lock.

For new code, higher-level utilities such as blocking queues, latches, semaphores, or condition variables are often easier to reason about. Avoid using notification as a substitute for defining a shared state predicate and its synchronization discipline.

### Q90. What is thread starvation, and how is it different from deadlock and livelock?

Deadlock means threads are stuck waiting on resources held by one another. Starvation means a thread repeatedly fails to get the CPU, lock, or resource it needs because others keep winning. Livelock means threads remain active but repeatedly react to one another without making useful progress.

The remedies differ: deadlock prevention often focuses on lock ordering and reducing nested locks; starvation may require fairer scheduling or bounded resource access; livelock may require backoff or a change in coordination strategy. Thread dumps and runtime metrics help distinguish symptoms rather than treating every hang as a deadlock.

### Q91. How should CompletableFuture exception handling and composition work?

Use `thenApply` to transform a completed result, `thenCompose` to chain an asynchronous operation that itself returns a future, and `thenCombine` when combining independent futures. Exception paths can be handled with methods such as `exceptionally`, `handle`, or `whenComplete`, but these have different semantics: some recover with a value, some transform the outcome, and some observe completion.

Do not block carelessly with `get()` or `join()` inside constrained executor threads; this can cause thread starvation or deadlock-like behavior. Consider the executor, timeout policy, cancellation behavior, and whether failures are propagated or converted into fallback results.

### Q92. How do you choose an ExecutorService and prevent thread-pool exhaustion?

Choose pool sizing and queue behavior based on the workload, resource limits, and latency target. CPU-bound tasks often benefit from a pool near the number of available processors; blocking I/O workloads need a different approach because workers spend time waiting. These are starting points, not universal formulas—measure the actual workload.

Unbounded queues can hide overload while latency and memory grow. Unbounded thread creation can exhaust memory or operating-system resources. Define rejection or backpressure behavior, track active threads and queue depth, bound task duration where possible, and shut down executors deliberately. For blocking task-per-request workloads on Java 21+, virtual threads may simplify concurrency, but they do not remove database connection limits or downstream capacity constraints.

### Q93. What are structured concurrency and scoped values, and what should you know about their status?

Structured concurrency aims to make related concurrent tasks share a clear lifetime and failure boundary, so child tasks are not left running unintentionally after the parent operation ends. Scoped values provide a controlled way to share context across call chains without passing it through every method parameter or relying on mutable thread-local state.

Their availability and final status have changed across JDK releases, and some versions expose these capabilities as preview APIs. In an interview, explain the design goal and verify the target JDK's status before claiming a particular API is final or production-ready. Do not treat preview features as stable language contracts.

### JVM, class loading, and memory

### Q94. What happens during class loading, linking, and initialization?

At a high level, a class is loaded, linked, and initialized. Linking includes verification, preparation, and resolution; initialization executes class initializers and static field initializers according to Java's initialization rules. These stages are related but not identical, and some resolution can be lazy.

Initialization order can explain surprising static initialization bugs and class initialization failures. A failed class initializer can cause an initialization error, and later uses may report that the class could not be initialized. Keep static initialization simple and avoid complex dependency work in static blocks.

### Q95. What is the difference between NoClassDefFoundError, ClassNotFoundException, and LinkageError?

`ClassNotFoundException` is a checked exception commonly encountered when code explicitly requests class loading by name and the loader cannot find the class. `NoClassDefFoundError` is a linkage error that can occur when the runtime cannot find a class definition needed by already-compiled code, including certain failed initialization scenarios. `LinkageError` is a broader family covering incompatibilities encountered when linking classes, including some version or duplicate-definition problems.

Diagnose the underlying classpath, module path, class loader, dependency versions, and original startup logs. The visible exception is often a symptom; the first failure in the logs may be more informative than the final error reported by the framework.

### Q96. What are strong, soft, weak, and phantom references?

A strong reference keeps an object reachable and therefore normally prevents garbage collection. Soft, weak, and phantom references provide weaker reachability semantics for specialized use cases, with different collection and notification behavior. Weak references are used in designs such as certain caches or canonicalization maps, but they should not be treated as a general-purpose cache policy.

Reference queues can help a program learn when referenced objects become eligible for cleanup or associated bookkeeping. Do not depend on a specific garbage-collection time, and do not use finalization for resource management. Prefer explicit lifecycle management and try-with-resources for external resources.

### Q97. What is a class-loader memory leak?

A class loader can become collectible only when it and the classes it defined are no longer reachable through relevant runtime references. In application servers or plugin systems, static registries, thread locals, long-lived threads, callbacks, or cached objects can retain classes from a redeployed application and therefore retain its class loader.

Symptoms can include growing metaspace usage across redeployments and increasing numbers of loaded classes. Investigate class-loader statistics and heap dumps, then identify the retaining references. Closing resources and clearing registries or thread-local values at the correct lifecycle boundary can prevent the leak.

### Q98. What are heap dumps, thread dumps, and Java Flight Recorder used for?

A heap dump captures object and reference information at a point in time and helps investigate retained memory, object populations, and possible leaks. A thread dump captures thread states and stack traces, useful for deadlocks, blocked threads, and executor problems. Java Flight Recorder (JFR) records runtime events such as allocations, garbage collection, locks, and thread activity with configurable overhead.

Use the diagnostic artifact that matches the symptom and correlate it with metrics and timestamps. A heap dump alone may show what retains memory but not why the workload allocated it; a thread dump alone may not explain a latency spike that has already passed. Collect data safely in production because dumps can be large and may contain sensitive values.

### Q99. What is the difference between metaspace, heap, and direct memory?

The heap stores Java objects managed by the garbage collector. Metaspace stores class metadata and is managed separately from the Java object heap. Direct buffers can use memory outside the ordinary heap, which can reduce copying for some I/O paths but still consumes finite native memory.

A process can run out of native memory even when heap usage appears acceptable. When diagnosing memory pressure, inspect heap occupancy, metaspace, direct-buffer usage, thread stacks, native allocations, and the container or operating-system limit rather than looking only at `-Xmx`.

### I/O, JDBC, reflection, and modules

### Q100. Why is the platform default charset a production risk?

The default charset can differ between environments, so code that relies on it may read or write text differently across machines. Specify a charset such as UTF-8 when encoding or decoding text unless the protocol explicitly requires something else. Also distinguish bytes from characters: network protocols and file formats define byte encodings, not an abstract Java string.

Use APIs that accept an explicit charset and test non-ASCII content. This avoids data corruption that may appear only when deployment locale, container image, or operating-system defaults change.

### Q101. What is the difference between byte streams and character streams?

Byte streams such as `InputStream` and `OutputStream` handle raw bytes and are appropriate for binary content. Character streams such as `Reader` and `Writer` handle text through character encoding and decoding. A text file is still stored as bytes, so the encoding must be known at the boundary.

Do not use a character reader for arbitrary binary content or convert bytes to strings without knowing the encoding. For large files, stream the content and manage buffers rather than loading everything into memory unnecessarily.

### Q102. What are Path and Files in NIO.2 useful for?

The `Path` API represents a filesystem path, and `Files` provides operations for reading, writing, copying, moving, and inspecting files. These APIs are generally more expressive than older `File` methods and support attributes, directory traversal, and explicit options. For large files, prefer streaming or buffered operations over convenience methods that read the entire file into memory.

When paths come from users, normalize and resolve them against an intended root and verify that the resolved target remains within that root. Symbolic links and filesystem races can complicate security, so path string checks alone are not a complete defense for sensitive file operations.

### Q103. How should JDBC resources and transactions be managed?

Use try-with-resources for `Connection`, `PreparedStatement`, and `ResultSet` where the surrounding transaction framework does not own their lifecycle. Use prepared statements for parameter values rather than concatenating user input into SQL. A transaction should group changes that must succeed or fail together, and failures should trigger the appropriate rollback behavior.

In frameworks such as Spring, understand whether the transaction is managed declaratively or manually and avoid closing a connection that the framework owns. Keep transactions reasonably short, define isolation requirements, and be careful about remote calls inside a database transaction because they extend lock and connection occupancy.

### Q104. What are transaction isolation levels, and why can anomalies occur?

Isolation defines which concurrent transaction effects are visible to a transaction. Common anomalies include dirty reads, non-repeatable reads, and phantom reads; the exact behavior and implementation details depend on the database. Higher isolation can reduce some anomalies but may increase contention, retries, or locking costs.

Choose isolation based on correctness requirements, not simply by selecting the strongest level everywhere. Understand the database's actual semantics and transaction mechanism, and test concurrency-sensitive workflows. Application-level checks can still race unless they are protected by suitable constraints, locking, or transactional design.

### Q105. What is reflection, and what are its trade-offs?

Reflection lets code inspect classes, methods, fields, constructors, and annotations at runtime, and in some cases invoke or access them dynamically. Frameworks use it for dependency injection, serialization, mapping, and test tooling. The trade-offs include weaker compile-time guarantees, more complex error handling, possible performance costs, and access restrictions under the module system.

Use reflection when runtime discovery is a real requirement, not as a default replacement for ordinary interfaces and polymorphism. Cache reflective metadata when justified by profiling, handle missing members deliberately, and test behavior against the target JDK and module configuration.

### Q106. What problem does the Java Platform Module System solve?

The module system, introduced in Java 9, allows applications and libraries to declare module dependencies and which packages they export. A module descriptor uses declarations such as `requires` and `exports`; qualified exports and `opens` support more controlled access, including certain reflective use cases.

Modules can improve encapsulation and make dependencies more explicit, but migration can reveal libraries that relied on unsupported access to JDK internals. Do not solve every access error by opening all packages broadly; understand which module needs access and whether a supported public API exists.

### Q107. What are annotations and annotation processing used for?

Annotations attach metadata to declarations and can be consumed by the compiler, build tools, runtime reflection, or frameworks. Retention policy determines whether an annotation is available only in source, in class files, or at runtime. Annotation processors can inspect source-level elements and generate additional source or resources during compilation.

Annotations do not automatically cause behavior; some tool or runtime component must interpret them. When debugging framework behavior, check the annotation's retention, target, processor configuration, and whether the relevant framework actually scans that class or package.

### Java evolution and compatibility

### Q108. What is the difference between source, bytecode, and runtime compatibility?

Source compatibility concerns whether code continues to compile under a language version. Bytecode compatibility concerns whether compiled class files can be read by a target JVM. Runtime or API compatibility concerns whether the required classes and methods exist and behave as expected on the deployed platform. These are separate dimensions.

For builds, configure the intended language/API target explicitly and test on the actual supported runtime. A project may compile using a newer JDK while accidentally referencing APIs unavailable on the production JDK if the build is not configured appropriately.

### Q109. Which Java 9–17 changes are worth revising beyond records and sealed classes?

Be familiar with the module system and collection factory methods from Java 9, local-variable type inference with `var` from Java 10, the standard HTTP Client and single-file source execution from Java 11, switch expressions and text blocks from Java 14/15, and records, pattern matching for `instanceof`, and sealed classes from Java 16/17. Exact availability matters: features often progressed through preview stages before becoming final.

Explain what a feature improves and where it does not apply. For example, `var` is local-variable type inference, not dynamic typing; records are concise data carriers with specific generated members, not a universal replacement for mutable domain entities.

### Q110. What should you consider when upgrading a service to a newer JDK?

Check the supported runtime and build tool versions, framework and library compatibility, removed or encapsulated JDK internals, TLS and security defaults, garbage collector behavior, container awareness, and startup/runtime flags. Compile and test with the intended release, then run integration and performance tests on a production-like environment.

An upgrade is not complete merely because compilation succeeds. Monitor startup, memory, latency, GC, dependency warnings, and behavior under load. Roll out gradually where possible and maintain a rollback plan.

### Security, testing, and engineering practice

### Q111. Why are prepared statements safer than constructing SQL strings?

Prepared statements separate SQL structure from parameter values, allowing the database driver to treat supplied values as data rather than executable SQL syntax. This is a key defense against SQL injection for values. Dynamic identifiers such as table names or sort directions usually cannot be handled as ordinary bind parameters, so they should be selected from an allowlist of known identifiers.

Validation remains useful for business rules, but escaping or manual string replacement is not a reliable substitute for parameterized queries. Use the same principle for other interpreters: keep untrusted data separate from executable commands whenever the API permits it.

### Q112. What is deserialization risk in Java, and how can it be reduced?

Deserializing untrusted native Java object streams can trigger complex object construction and gadget behavior before application code has validated the result. The safest approach is generally to avoid native serialization for untrusted input and prefer a data format with an explicit schema and constrained types. If native serialization must be used, apply strict filtering and allowlisting appropriate to the application and JDK.

Do not assume that a signature alone makes every deserialization path safe, and do not deserialize arbitrary classes from external clients. Review dependency chains and keep serialization boundaries narrow.

### Q113. How do you test concurrent code without flaky sleeps?

Test synchronization conditions, not elapsed time guesses. Use latches, barriers, bounded waits, controlled executors, and deterministic setup to coordinate the interleavings relevant to the behavior. A timeout can prevent a hung test from running forever, but a fixed sleep does not prove that a particular thread reached a state.

For complex concurrency algorithms, consider stress testing and specialized tools in addition to unit tests. Ensure tests fail with useful diagnostics and clean up executors so a failed test does not leave background threads running.

### Q114. What is the difference between unit, integration, and contract testing in a Java service?

Unit tests isolate a small unit of behavior and run quickly. Integration tests verify that components work together, such as persistence mappings, serialization, configuration, and framework wiring. Contract tests verify agreed behavior at a boundary between independently developed components, such as a REST API or message schema.

Use the level that gives confidence in the relevant risk. Mocking every dependency can make tests fast but may miss configuration, SQL, transaction, and serialization problems. Conversely, using a full integration environment for every simple branch can make feedback slow and debugging difficult.

### Q115. What does good logging look like in a Java backend service?

Logs should explain significant events and failures with useful context such as correlation identifiers, operation names, durations, and safe identifiers. Use appropriate severity levels and avoid logging secrets, access tokens, personal data, or large unredacted payloads. Include the exception cause when it helps diagnosis, but avoid logging and rethrowing the same failure repeatedly at every layer.

Logging is not a replacement for metrics or traces. Metrics reveal rates and trends; traces connect work across service boundaries; logs provide event-level detail. Together they help narrow down production failures without relying on one noisy log line.

### Q116. How should a Java service handle configuration and secrets?

Separate environment-specific configuration from application code, validate required configuration at startup, and keep secrets in an appropriate secret-management system rather than source control or plain logs. Avoid silent fallbacks for critical settings because they can make a service start with unsafe or incorrect behavior.

Configuration should have clear ownership, safe defaults where possible, and documented units and timeouts. Treat configuration changes as operational changes: test them, audit sensitive changes, and expose enough sanitized effective configuration to diagnose behavior.

### Q117. What is the difference between equals(), hashCode(), and compareTo() contracts?

`equals()` defines logical equality, and equal objects must have equal hash codes. `compareTo()` defines ordering and returns zero when values are considered equal under that ordering. The `Comparable` contract recommends consistency with `equals()`, though it does not strictly require it in all cases.

Hash-based collections use equality and hashing; sorted collections use comparison. Inconsistent contracts can cause a value to appear present in one collection but behave differently in another. For domain objects, choose stable identity fields and test the contracts together.

### Q118. How should you reason about Java API design for nullability?

Decide whether absence is valid for each parameter, field, and return value, then document and enforce that contract. Use validation at boundaries and return meaningful errors instead of allowing a null to travel through multiple layers until an unrelated dereference fails. Optional can communicate an optional return value, but it is not automatically the right type for every field or parameter.

Avoid mixing conventions without explanation. In large codebases, annotations and static analysis can make nullability contracts more visible, but they do not replace clear API design and runtime validation where untrusted input is involved.

### Code-reading and output questions

### Q119. What is printed when a method is overloaded with primitive, boxed, and varargs parameters?

Overload selection is performed at compile time. Java generally prefers a more specific applicable fixed-arity overload before considering variable-arity invocation, and primitive widening, boxing, and reference widening can affect which overload is selected. The exact choice depends on the available signatures and the compile-time types of the arguments.

In an interview, work through the overload-resolution phases rather than guessing from runtime values. If the argument is a reference typed as a superclass or interface, overload resolution uses that declared compile-time type; overriding, by contrast, is dynamically dispatched at runtime.

### Q120. Why can string concatenation and numeric promotion produce surprising output?

The `+` operator can mean numeric addition or string concatenation depending on operand types and evaluation order. Numeric operands may undergo binary numeric promotion, and concatenation converts non-string operands to string representations. Parentheses and the type of the left operand can therefore change the result.

For output questions, annotate each intermediate expression with its type and evaluate left to right according to operator precedence and associativity. Do not rely on visual intuition, especially when strings, chars, boxed values, and arithmetic are mixed.

### Q121. What is the difference between compile-time errors and runtime exceptions in code-output questions?

Compile-time errors occur when source code violates language or type rules, so the program does not compile. Runtime exceptions occur after compilation, when execution reaches an operation that fails. Some examples depend on whether a value is a compile-time constant, the declared type, overload resolution, or generic type erasure.

When asked what a snippet does, first determine whether it compiles. If it does, then evaluate initialization, overload selection, dispatch, and runtime control flow. This sequence prevents wasting time tracing code that is invalid before execution begins.

### Production scenario questions

### Q122. A production service has rising latency but normal CPU utilization. What do you check first?

Look for waiting rather than computation: thread-pool queue depth, blocked or parked threads, connection-pool saturation, slow database queries, downstream latency, lock contention, rate limits, and timeout/retry amplification. Compare request latency percentiles with dependency timings and resource metrics. A thread dump taken during the incident can reveal where request threads are waiting.

Avoid increasing thread counts blindly. If the database pool is exhausted, adding application threads can increase queueing and contention without increasing throughput. Find the constrained resource, bound waiting, and align deadlines across service and dependency calls.

### Q123. A service has high allocation rates but no obvious memory leak. How do you investigate?

High allocation and a memory leak are different problems. Frequent short-lived allocation can increase GC work even if objects are eventually reclaimed; a leak involves objects remaining reachable longer than intended. Use allocation profiling or JFR to identify allocation sites, correlate allocation rate with request volume and latency, and inspect GC behavior.

Optimize only after identifying the dominant allocation source. Reuse can help in selected hot paths, but unnecessary object pools often add complexity and contention. First consider algorithmic improvements, avoid repeated parsing or copying, and verify any change under a representative workload.

### Q124. A request times out, but the client retries and duplicate work occurs. How do you prevent it?

A timeout does not prove that the original operation failed; the server may have completed it while the response was lost. For operations that must not be duplicated, design idempotency into the boundary—for example, use an idempotency key stored with the operation result, unique constraints, or a durable state transition that makes repeated requests safe.

Retries should be bounded, use backoff and jitter, respect an overall deadline, and be applied only when repeating the operation is safe. Exactly-once effects across distributed systems are not achieved merely by retrying or using a message broker; they require careful coordination of durable state and side effects.

### Q125. A CompletableFuture chain hangs under load. What would you investigate?

Inspect which executor each stage uses, whether blocking calls occupy a limited pool, whether a stage waits on work scheduled to the same saturated executor, and whether a future is never completed on an error path. Look at thread dumps, executor metrics, queue depth, timeout behavior, and the chain's exception-handling stages.

Avoid calling blocking joins inside a constrained worker pool when dependent work needs that same pool to make progress. Make timeout and cancellation behavior explicit, and ensure exceptional completion cannot silently leave the request waiting forever.

### Q126. How would you choose between a concurrent collection and external synchronization?

Start with the shared invariant. If threads only need independent thread-safe map or queue operations, a concurrent collection may be sufficient. If correctness depends on a sequence of operations being atomic—for example, check-then-act across multiple map calls—use an atomic collection method, a suitable lock, or redesign the state transition so it is indivisible.

Thread-safe individual methods do not automatically make a multi-step workflow thread-safe. Document the invariant, choose the smallest synchronization boundary that protects it, and test concurrent interleavings that could violate it.

---

### Q127. Why can String.length() differ from the number of visible characters?

Java String stores UTF-16 code units, so `length()` reports the number of UTF-16 code units, not necessarily Unicode code points or user-perceived characters. A character outside the Basic Multilingual Plane uses a surrogate pair and therefore occupies two code units. Some visible symbols are also made from multiple code points, such as a base character plus a combining mark or a joined emoji sequence.

For simple ASCII-oriented identifiers, `length()` often matches expectations. For Unicode-aware processing, use code-point APIs such as `codePointCount` and be clear whether the requirement concerns code units, code points, or grapheme clusters.

### Q128. What are common pitfalls with String.split(), trim(), strip(), and isBlank()?

`split()` accepts a regular expression, not a literal delimiter, and its default behavior discards trailing empty fields. Use a quoted delimiter or an appropriate regex when the delimiter has special meaning, and use a negative limit when trailing empty fields must be preserved. `trim()` removes a narrower range of leading and trailing characters than Unicode-aware `strip()`; `isEmpty()` checks whether length is zero, while `isBlank()` also recognizes strings containing only whitespace.

These methods have different semantics, so choose based on the input format and Java version. In data processing, losing trailing empty columns can silently corrupt records; write a test for empty input, repeated delimiters, trailing delimiters, and non-ASCII whitespace.

### Q129. What does computeIfAbsent() solve, and what should you avoid inside its mapping function?

`computeIfAbsent(key, mappingFunction)` atomically associates a computed value when the key is absent under the map's contract, which is useful for patterns such as building a list per key or caching a result. It is safer than a separate contains/get followed by put when the map is concurrent and the operation needs atomic coordination.

Keep the mapping function short and avoid recursively modifying the same map or performing slow, blocking work inside it. Concurrent map implementations can impose specific restrictions on the mapping function; consult the relevant API contract rather than assuming all Map implementations behave identically.

### Q130. What is IdentityHashMap, and when would you use it?

`IdentityHashMap` compares keys using reference identity (`==`) rather than logical equality (`equals()`). This makes it useful for specialized tasks such as tracking object identity during graph traversal, serialization-like processing, or cycle detection where two equal-but-distinct objects must remain separate.

It is not a general replacement for `HashMap`. Most application maps should use logical equality, and callers should be told explicitly if identity semantics are required because the behavior can be surprising for strings, records, and value objects.

### Q131. What is the ABA problem in compare-and-set algorithms?

With compare-and-set (CAS), a thread checks that a value still equals an expected value before replacing it. The ABA problem occurs when another thread changes the value from A to B and then back to A; a CAS operation sees A again and cannot tell that an intermediate change occurred. Whether this matters depends on the state invariant and what the value represents.

Where change history matters, use a versioned or stamped value, such as an appropriate stamped atomic reference, or redesign the state transition. Do not assume that a successful CAS proves nothing changed in between.

### Q132. Why is double-checked locking unsafe without volatile?

Double-checked locking tries to avoid acquiring a lock on every access by checking a shared reference before and after entering a synchronized block. Without the correct memory semantics, another thread could observe a published reference before construction is safely visible, because object initialization and reference publication are not automatically one indivisible action.

In modern Java, a correctly implemented pattern uses a volatile reference and synchronized initialization, but the initialization-on-demand holder idiom or an enum singleton is often simpler. Explain both the visibility guarantee and why avoiding unnecessary custom singleton machinery is usually preferable.

### Q133. Why is Thread.sleep() not a synchronization mechanism?

`Thread.sleep()` pauses the current thread for a period, subject to scheduling, but it does not establish a happens-before relationship and does not guarantee that another thread's writes become visible. A test or program that “works” after sleeping may still contain a data race and can fail under different timing or hardware.

Coordinate threads through synchronized state, volatile/atomic variables, latches, queues, futures, or other documented synchronization mechanisms. Sleeps can be useful for pacing or simulation, but not for proving that another thread has completed an operation.

### Q134. What are p95 and p99 latency, and why are averages insufficient?

p95 latency is the value at or below which 95% of observed requests complete; p99 similarly describes the 99th percentile. Averages can hide a small fraction of extremely slow requests, even though those tail requests often dominate user experience and timeouts in distributed call chains.

Track percentiles alongside throughput, error rates, saturation, and dependency timings. Compare them over a meaningful window and sufficient request volume, and avoid treating one percentile as the whole story. Tail latency often points to queueing, contention, retries, garbage collection, or a slow dependency.

### Q135. How do you investigate high CPU usage in a Java process?

First establish whether the process is truly CPU-saturated and whether the usage is in application threads, GC, JIT compilation, or native code. Correlate operating-system process and thread metrics with JVM telemetry, then capture thread dumps or a Java Flight Recorder recording while the issue is active. Repeated hot stack traces can help identify loops, excessive serialization, expensive regular expressions, contention-related spinning, or unexpectedly high request volume.

Do not tune the garbage collector or rewrite code before identifying the dominant CPU consumer. Compare the evidence with request rate and deployment changes, then validate a targeted fix under representative load.

### Q136. What tools help diagnose a running JVM, and what is each useful for?

Common tools include `jcmd` for diagnostic commands and recordings, `jstack` or equivalent thread-dump mechanisms for thread states, `jmap` or supported heap-dump commands for heap investigation, `jstat` for selected JVM statistics, and Java Flight Recorder with Java Mission Control for runtime-event analysis. Exact availability and command options depend on the JDK distribution, version, permissions, and environment.

Use the least disruptive tool that answers the question, and check the tool's documentation for the target runtime. Heap dumps and recordings can be large and may contain sensitive data, so plan collection, storage, access control, and cleanup.

### Q137. How do TCP, message framing, and blocking I/O affect Java network code?

TCP provides a reliable byte stream, not message boundaries. A single read may return part of a message, one message, or multiple messages, so the application protocol must define framing—for example, a length prefix, delimiter with escaping, or fixed-size records. Assuming one read equals one request is a common networking bug.

Blocking I/O is straightforward but ties up a thread while waiting. Non-blocking I/O and selectors can multiplex many channels but require more complex state management. Virtual threads can make blocking code easier to scale for many I/O-bound tasks, but connection limits, downstream capacity, timeouts, and backpressure still need explicit handling.

### Q138. How should you evaluate a newer Java API or preview feature before adopting it?

Check the exact JDK release, whether the feature is final, preview, or incubating, the required compiler/runtime flags, and whether dependent libraries and production tooling support it. Preview APIs can change or disappear, so adoption needs a conscious compatibility and upgrade plan rather than assuming source code will remain valid.

Evaluate the feature against a real maintenance or performance need, and test migration, observability, packaging, and production behavior. In interviews, distinguish the feature's design goal from its status in the specific release being discussed.


## 14. Java Streams — coding interview questions

These are practical stream exercises, not just API definitions. Code is intentionally limited to the core expression. In an interview, explain the edge cases, complexity, ordering, and why the chosen operation fits the requirement.

### Q139. How do you filter even numbers and square them?

    List<Integer> squares = numbers.stream()
        .filter(n -> n % 2 == 0)
        .map(n -> n * n)
        .toList();

Filter selects values and map transforms them. Stream.toList() is available from Java 16 and returns an unmodifiable list; for Java 8–15 use Collectors.toList(), whose mutability is not guaranteed by the general contract either.

### Q140. How do you count the frequency of each word?

    Map<String, Long> counts = words.stream()
        .collect(Collectors.groupingBy(
            Function.identity(), Collectors.counting()));

groupingBy creates a group for each distinct word and counting reduces each group to a count. Normalize case and define punctuation handling before counting if the input is natural language.

### Q141. How do you group employees by department?

    Map<String, List<Employee>> byDepartment = employees.stream()
        .collect(Collectors.groupingBy(Employee::getDepartment));

The classifier determines each map key. If a department can be null, decide how to represent missing values explicitly because the default groupingBy collector does not accept a null classifier result.

### Q142. How do you count employees in each department?

    Map<String, Long> countByDepartment = employees.stream()
        .collect(Collectors.groupingBy(
            Employee::getDepartment, Collectors.counting()));

This is a downstream collector. It retains counts rather than lists of employees, which is more efficient when the grouped objects are not needed later.

### Q143. How do you find the second-highest distinct salary?

    Optional<Integer> second = employees.stream()
        .map(Employee::getSalary)
        .distinct()
        .sorted(Comparator.reverseOrder())
        .skip(1)
        .findFirst();

The result is optional because there may be fewer than two distinct salaries. Sorting costs O(n log n); for a very large input where only two values are needed, a one-pass approach can be more efficient.

### Q144. How do you find duplicate values in a list?

    Map<String, Long> counts = values.stream()
        .collect(Collectors.groupingBy(
            Function.identity(), Collectors.counting()));

    Set<String> duplicates = counts.entrySet().stream()
        .filter(e -> e.getValue() > 1)
        .map(Map.Entry::getKey)
        .collect(Collectors.toSet());

Counting first avoids shared mutable state inside the stream. A side-effect-based solution using a shared HashSet is not safe for parallel execution.

### Q145. How do you remove duplicates while preserving encounter order?

    List<String> unique = values.stream()
        .distinct()
        .toList();

For an ordered stream, distinct preserves the first encountered occurrence. Equality relies on equals, so custom objects need correct equality semantics.

### Q146. How do you partition numbers into even and odd groups?

    Map<Boolean, List<Integer>> result = numbers.stream()
        .collect(Collectors.partitioningBy(n -> n % 2 == 0));

partitioningBy is appropriate when there are exactly two logical groups. groupingBy is better when the number of categories is variable.

### Q147. How do you flatten a list of lists?

    List<String> flattened = nested.stream()
        .flatMap(List::stream)
        .toList();

map would produce a stream of lists; flatMap combines the inner streams into one stream. Decide how null inner lists should be handled rather than allowing accidental failures.

### Q148. How do you split sentences into distinct words?

    Set<String> words = sentences.stream()
        .flatMap(s -> Arrays.stream(s.split("\\s+")))
        .map(s -> s.toLowerCase(Locale.ROOT))
        .collect(Collectors.toSet());

This is a simple whitespace tokenizer, not a full natural-language tokenizer. Punctuation, Unicode rules, and locale-specific normalization may require additional handling.

### Q149. How do you convert a list into a map, and what if keys repeat?

    Map<Long, Employee> byId = employees.stream()
        .collect(Collectors.toMap(Employee::getId, Function.identity()));

toMap throws IllegalStateException when duplicate keys occur. If duplicates are valid, provide a merge function and document whether the first value, last value, or a combined value should win.

### Q150. How do you sum values when multiple objects share the same key?

    Map<String, Integer> totals = items.stream()
        .collect(Collectors.toMap(
            Item::getType, Item::getAmount, Integer::sum));

The merge function combines values with the same key. Consider overflow and make sure summing matches the business rule.

### Q151. How do you sort strings by length and then alphabetically?

    List<String> sorted = words.stream()
        .sorted(Comparator.comparingInt(String::length)
            .thenComparing(Comparator.naturalOrder()))
        .toList();

Comparator composition expresses the primary and tie-breaker order. A deterministic tie-breaker makes results and tests more predictable.

### Q152. How do you calculate sum, average, minimum, and maximum in one pass?

    IntSummaryStatistics stats = numbers.stream()
        .mapToInt(Integer::intValue)
        .summaryStatistics();

Primitive streams can reduce boxing. Be mindful of numeric overflow and precision; floating-point statistics are not a substitute for decimal arithmetic in financial domains.

### Q153. How do you join strings with a delimiter?

    String joined = names.stream()
        .filter(Objects::nonNull)
        .collect(Collectors.joining(", "));

joining supports delimiters, prefixes, and suffixes. Decide explicitly whether null values should be rejected, filtered, or represented.

### Q154. How do you test whether any, all, or none of the elements match?

    boolean anyHigh = salaries.stream().anyMatch(s -> s > limit);
    boolean allValid = salaries.stream().allMatch(s -> s >= 0);
    boolean noneNegative = salaries.stream().noneMatch(s -> s < 0);

These are short-circuiting terminal operations. On an empty stream, anyMatch is false while allMatch and noneMatch are true, which follows their logical definitions.

### Q155. What is the difference between findFirst() and findAny()?

findFirst returns the first element according to encounter order when the stream has one. findAny permits any matching element and can offer more flexibility in parallel pipelines. Choose findFirst when order is part of the requirement; do not use it as a substitute for sorting to find a minimum or maximum.

### Q156. What is the difference between reduce() and collect()?

reduce combines elements into a result using an operation such as sum or maximum. collect supports mutable reduction using a supplier, accumulator, and combiner, such as building a list or map. Avoid using reduce with a shared mutable list; use a collector designed for the reduction.

### Q157. Why must a parallel reduce operation be associative?

Parallel execution can split input into chunks and combine partial results in a different grouping. If the operation is not associative, changing the grouping can change the result. The identity must also be a valid identity for the operation. Subtraction is a classic example of an operation that is not associative.

### Q158. How do you get a set of employee names per department?

    Map<String, Set<String>> namesByDepartment = employees.stream()
        .collect(Collectors.groupingBy(
            Employee::getDepartment,
            Collectors.mapping(Employee::getName, Collectors.toSet())));

The downstream mapping collector transforms each employee to a name before collecting. This avoids collecting full employee lists and then traversing them again.

### Q159. How do you calculate average salary per department?

    Map<String, Double> averageByDepartment = employees.stream()
        .collect(Collectors.groupingBy(
            Employee::getDepartment,
            Collectors.averagingDouble(Employee::getSalary)));

This produces one average per group. For financial values, choose a decimal representation and explicit rounding policy when binary floating-point is not appropriate.

### Q160. How do you collect entries into a sorted map?

    Map<String, Long> counts = words.stream()
        .collect(Collectors.groupingBy(
            Function.identity(), TreeMap::new, Collectors.counting()));

The map factory selects TreeMap, which sorts keys. Use it when sorted iteration is a requirement, not simply because sorted output looks convenient.

### Q161. Why should peek() not be used for required business side effects?

peek is primarily an inspection/debugging operation. Stream optimizations may skip processing that does not affect the terminal result, so required side effects inside peek are not reliable. Use an explicit terminal operation or place the side effect at a clear boundary.

### Q162. Why can a Stream only be consumed once?

A Stream is a pipeline, not a reusable collection. After a terminal operation, it is consumed and a second terminal operation normally fails. Create a new stream from a reusable source when multiple traversals are needed.

### Q163. When is parallelStream() a poor choice?

It can be slower for small inputs, cheap operations, blocking I/O, pipelines with shared mutable state, or strict ordering requirements. Parallel streams may also contend for shared fork/join infrastructure. Benchmark representative workloads instead of assuming parallel is faster.

### Q164. How do you find the top three most frequent words?

    List<Map.Entry<String, Long>> top = counts.entrySet().stream()
        .sorted(Map.Entry.<String, Long>comparingByValue().reversed()
            .thenComparing(Map.Entry.comparingByKey()))
        .limit(3)
        .toList();

Sorting all k distinct entries costs O(k log k). For huge k and very small top-N, a bounded heap may be more efficient; this solution is straightforward and readable.

### Q165. How do you find common elements between two lists efficiently?

    Set<String> lookup = new HashSet<>(second);
    List<String> common = first.stream()
        .filter(lookup::contains)
        .distinct()
        .toList();

The set gives efficient average membership checks and the output preserves the first list's encounter order. Clarify whether duplicates should be preserved, because set intersection and multiset intersection are different requirements.

### Q166. How do you safely handle an empty numeric stream?

    OptionalInt maximum = numbers.stream()
        .mapToInt(Integer::intValue)
        .max();

The optional result represents the absence of a maximum. Avoid retrieving a value without checking presence or defining a fallback.

### Q167. How do you compose methods that return Optional values?

    Optional<Country> country = person
        .flatMap(Person::getAddress)
        .flatMap(Address::getCountry);

flatMap avoids nested Optional values when each function already returns Optional. The example assumes the methods use Optional consistently; normalize nullable boundaries deliberately.

### Q168. When is a loop better than a stream?

Loops can be clearer for early exits, complex state changes, coordinated mutation, or detailed error handling. Streams are often concise for filter-map-reduce and collector-based transformations. Explain readability and complexity rather than claiming one style is always faster.

### Q169. How do you implement a collector for a custom aggregation?

A Collector defines how to create an accumulation container, add one element, combine partial containers, and optionally finish the result. Its combiner and characteristics must satisfy the collector contract, particularly if parallel execution is allowed. Prefer standard collectors unless the custom aggregation is genuinely clearer and tested in both sequential and parallel use.

### Q170. What should you check when a stream pipeline gives an unexpected result?

Check the source ordering, intermediate operation order, terminal operation, equals/hashCode behavior used by distinct or map keys, null handling, and whether side effects were incorrectly placed in intermediate operations. For parallel streams, also check ordering requirements and whether the collector supports parallel reduction. Reduce the problem to a small input and test edge cases such as empty input, duplicates, nulls, and ties.

## 15. Java version-wise features: Java 8 to Java 25

Use this release-by-release section to revise when a feature appeared and when it became final. LTS status and vendor support periods are related but not identical; verify the JDK vendor's support policy for production decisions.

| Release | Status | Interview features to know |
|---|---|---|
| Java 8 | LTS | Lambdas, method references, functional interfaces, Streams, Optional, default interface methods, java.time |
| Java 9 | Feature | JPMS modules, collection factories, private interface methods, takeWhile/dropWhile, Flow API |
| Java 10 | Feature | Local-variable type inference with var |
| Java 11 | LTS | Standard HttpClient, String.isBlank/strip/lines/repeat, Files.readString/writeString |
| Java 12–13 | Feature | Switch expressions preview iterations, teeing collector, text blocks preview |
| Java 14 | Feature | Switch expressions final, helpful NullPointerExceptions; records preview |
| Java 15 | Feature | Text blocks final, hidden classes |
| Java 16 | Feature | Records final, pattern matching for instanceof final |
| Java 17 | LTS | Sealed classes final, stronger encapsulation of JDK internals; switch pattern matching preview |
| Java 18–20 | Feature | UTF-8 default charset, virtual threads preview iterations, evolving pattern/concurrency features |
| Java 21 | LTS | Virtual threads, record patterns, pattern matching for switch, sequenced collections, generational ZGC |
| Java 22–24 | Feature | Foreign Function & Memory API final in 22, unnamed variables/patterns, evolving stream and language APIs |
| Java 25 | LTS | Review the finalized Java 25 release features, runtime improvements, and compatibility changes against official release notes |

### Q171. Which Java 8 features are most important for backend interviews?

Prioritize lambdas, method references, functional interfaces, streams, Optional, default interface methods, and java.time. Be able to explain laziness, map versus flatMap, collector behavior, effectively final captured variables, and when a loop is clearer than a stream. These concepts remain common even in codebases running newer JDKs.

### Q172. What is the difference between Java 9 modules and the classpath?

The module system makes dependencies and exported packages explicit through module descriptors. The classpath traditionally exposes classes without the same module-level encapsulation. Modules can improve maintainability and encapsulation, but migration may expose libraries that rely on internal JDK APIs or broad reflective access.

### Q173. What does var mean in Java 10?

var asks the compiler to infer the static type from the initializer of a local variable. It is not dynamic typing, cannot be used without an initializer, and does not mean the variable can later hold a different type. Use it when the initializer makes the type obvious and avoid it when it obscures important information.

### Q174. What Java 11 features should a backend engineer remember?

Java 11 is an LTS release with the standard HTTP Client and useful String and Files APIs. It also removed Java EE and CORBA modules from the JDK distribution. When upgrading from Java 8, check library dependencies, removed modules, TLS behavior, build plugins, and runtime assumptions in addition to source changes.

### Q175. What is the difference between a preview feature and a final feature?

A preview feature is exposed for evaluation and may require explicit compiler/runtime enablement; it can change or be removed before finalization. A final feature is part of the standard platform contract for that release. Always identify both the first preview release and the release where the feature became final.

### Q176. What should you know about records in Java 16 and later?

Records are concise data carriers with generated component accessors and value-oriented equals, hashCode, and toString implementations. Their components are final, but immutability is shallow: a component can reference a mutable object. Use defensive copying if the record must protect its internal state.

### Q177. Why are sealed classes useful in Java 17?

Sealed classes and interfaces restrict which types may extend or implement them. They help model a closed set of domain alternatives and can make exhaustive handling clearer when combined with pattern matching. Use them only when the hierarchy is intentionally controlled rather than open to arbitrary extensions.

### Q178. Why is Java 21 a major LTS milestone?

Java 21 finalized virtual threads, record patterns, pattern matching for switch, and sequenced collections. It also included important runtime improvements. Structured concurrency and scoped values were preview APIs in Java 21, so they should not be described as final in that release.

### Q179. How do you start a virtual thread?

    Thread.startVirtualThread(() -> processRequest());

Or create a named thread:

    Thread worker = Thread.ofVirtual()
        .name("request-worker")
        .start(this::processRequest);

Virtual threads make blocking task-per-thread code more scalable for I/O-bound work. They do not increase database capacity, remove downstream bottlenecks, or make CPU-heavy tasks infinitely faster.

### Q180. How do you run tasks with a virtual-thread-per-task executor?

    try (ExecutorService executor =
             Executors.newVirtualThreadPerTaskExecutor()) {
        Future<Result> result = executor.submit(this::loadResult);
        Result value = result.get();
    }

Use deadlines, cancellation, and explicit error handling in production. A virtual thread per task is not the same as an unbounded amount of downstream capacity; use bounded resource pools and backpressure where needed.

### Q181. What should you know about Java 25?

Java 25 is an LTS release. Prepare its finalized language and API features, runtime and garbage-collection improvements, and migration implications from Java 17 or 21. Verify the exact release notes rather than relying on a remembered list, because feature status can change across preview iterations.

### Q182. How should you plan a Java 17-to-21 or Java 25 migration?

Verify framework, library, build-tool, and runtime compatibility; compile and test against the intended release; review reflective access, removed APIs, security defaults, and JVM flags; and run integration and performance tests under representative load. Roll out gradually where possible and compare latency, memory, GC, and error metrics with a baseline.

### Q183. What is the difference between LTS and a feature release?

LTS means a vendor offers an extended support commitment for that release; the exact duration depends on the vendor and distribution. Feature releases deliver newer platform changes on a shorter cadence. Choose a runtime based on security support, compatibility, operational readiness, and upgrade cost, not the label alone.

### Q184. How should a version matrix distinguish introduced, preview, and finalized features?

Track at least three states: first introduced, preview/incubator status if applicable, and final release. For example, virtual threads appeared as preview before becoming final in Java 21. This avoids saying that a feature was stable merely because it appeared in an earlier release.


### Q185. How do you find the longest string in a list with streams?

    Optional<String> longest = words.stream()
        .max(Comparator.comparingInt(String::length));

The result is optional because the source may be empty. If several strings have the same maximum length, the contract may not express which one should win in every stream configuration; define a tie-breaker if deterministic selection matters.

### Q186. How do you safely handle null values before sorting?

    List<String> sorted = values.stream()
        .filter(Objects::nonNull)
        .sorted()
        .toList();

Filtering nulls is one possible policy, but it is not always the correct business behavior. Another option is a null-aware comparator such as Comparator.nullsLast. State the policy explicitly rather than letting a NullPointerException decide it.

### Q187. How do you use mapMulti, and when might it be useful?

    Stream<String> tokens = lines.stream()
        .<String>mapMulti((line, out) -> {
            for (String token : line.split("\\s+")) {
                out.accept(token);
            }
        });

mapMulti can emit zero or more results for each input element and can be useful for simple one-to-many transformations without creating an inner stream for every element. It became a standard Stream API method in Java 16. Prefer flatMap when it communicates the transformation more clearly.

### Q188. How do you use takeWhile and dropWhile?

    List<Integer> prefix = values.stream()
        .takeWhile(n -> n < 10)
        .toList();

    List<Integer> remainder = values.stream()
        .dropWhile(n -> n < 10)
        .toList();

For ordered streams, takeWhile keeps the longest matching prefix and dropWhile skips that prefix. They are not equivalent to filter: filter can retain later matching values after a non-matching value appears.

### Q189. What does Collectors.teeing do?

    Summary summary = values.stream().collect(
        Collectors.teeing(
            Collectors.minBy(Integer::compareTo),
            Collectors.maxBy(Integer::compareTo),
            Summary::new));

teeing sends each input to two downstream collectors and combines their results with a merger. It can calculate related summaries in one traversal, but the result type and empty-input behavior still need a clear design.

### Q190. What are the common pitfalls of Collectors.toMap?

Duplicate keys cause an exception unless a merge function is supplied. A null key or value may also be rejected depending on the collector and map implementation. Decide whether duplicates are invalid, should overwrite, or should be aggregated; do not add a merge function just to silence an exception without defining the rule.

### Q191. How do you use a method reference, and when is a lambda clearer?

    names.stream().map(String::trim).toList();

A method reference is concise when an existing method exactly matches the functional interface's shape. A lambda can be clearer when it performs additional logic, handles edge cases, or makes the transformation more explicit. Choose readability rather than using method references everywhere.

### Q192. What is the effectively final rule for captured local variables?

A lambda can capture a local variable only when it is final or effectively final, meaning its value is assigned once and not reassigned. This rule avoids the appearance that a lambda is sharing a mutable local stack variable across execution contexts. It does not make the referenced object immutable: a captured reference can still point to a mutable object.

### Q193. How should you use Comparator.comparing with nullable fields?

    Comparator<Employee> byName = Comparator.comparing(
        Employee::getName,
        Comparator.nullsLast(Comparator.naturalOrder()));

The comparator defines how null names are ordered. Make null handling explicit in the comparator or validate the data before sorting. Comparator composition should also include tie-breakers when deterministic ordering is important.

### Q194. How do you avoid accidental integer overflow in stream sums?

    long total = values.stream()
        .mapToLong(Integer::longValue)
        .sum();

Summing into a wider primitive reduces the risk of 32-bit overflow, but long can overflow too. For exact large integer totals, use BigInteger; for decimal financial values, choose BigDecimal with an explicit rounding policy.

### Q195. What is the difference between mapToInt and map?

map transforms each input into another reference value and produces a Stream of objects. mapToInt produces an IntStream, which provides primitive-specialized operations such as sum and summaryStatistics and avoids boxing each value. Use primitive streams when the result is numeric and their operations make the code clearer.

### Q196. How do you combine two independent asynchronous tasks with CompletableFuture?

    CompletableFuture<Result> a = loadA();
    CompletableFuture<Result> b = loadB();

    CompletableFuture<Combined> combined =
        a.thenCombine(b, Combined::new);

thenCombine waits for both stages and combines their results. The tasks are independent only if their creation starts the work independently. Consider failure propagation, deadlines, executor choice, and whether one failed result should cancel or merely affect the combined outcome.

### Q197. What is the difference between thenApply and thenCompose?

thenApply transforms a result synchronously into another value. thenCompose is used when the transformation itself returns a CompletableFuture, flattening the nested asynchronous result. Use thenCompose to chain dependent asynchronous operations without producing CompletableFuture<CompletableFuture<T>>.

### Q198. How do you handle an exceptional CompletableFuture result?

    future.exceptionally(error -> fallbackValue);

exceptionally can recover from a failure by producing a fallback result. handle can inspect either success or failure and transform both; whenComplete is intended for observation while preserving the original outcome. Do not silently convert every error into a default value if doing so hides a business failure.

### Q199. How do you make a Java object immutable in practice?

Make the class's state private, initialize it completely during construction, avoid setters, and prevent mutable internal objects from escaping. Use defensive copies for mutable input and output collections. Final fields help express initialization guarantees but do not make a referenced list or date-like mutable object immutable by themselves.

### Q200. What is the difference between a record and a regular class?

A record is a concise declaration for a data carrier with generated component accessors and value-oriented equality, hashing, and string representation. A regular class gives more freedom over inheritance, mutable lifecycle state, and representation. Records are implicitly final and cannot extend another class, so choose them based on domain semantics rather than code-length reduction alone.

### Q201. How does pattern matching for switch improve code?

    String description = switch (value) {
        case String s -> "text: " + s;
        case Integer i -> "number: " + i;
        case null -> "missing";
        default -> "other";
    };

Pattern matching can combine type testing and binding, reducing repeated casts and making alternatives easier to read. The exact supported syntax depends on the Java release; pattern matching for switch became final in Java 21, while primitive patterns remained preview in Java 25.

### Q202. What are module import declarations in Java 25?

Java 25 finalized module import declarations, which allow importing the exported packages of a module with a module-oriented import form. They can reduce import boilerplate in some applications, but they do not replace understanding module dependencies, exported packages, or classpath/module-path behavior. Verify the syntax against the Java 25 language documentation before using it in a code sample.

### Q203. What are compact source files and instance main methods in Java 25?

Java 25 finalized compact source files and instance main methods, making small teaching programs and simple utilities possible with less ceremony. They reduce the amount of scaffolding required for beginner examples, but traditional classes and static main methods remain important for libraries and conventional applications. These features are useful to recognize in newer examples without assuming all production code should adopt the compact form.

### Q204. What are flexible constructor bodies in Java 25?

Java 25 finalized flexible constructor bodies, which permit certain statements before an explicit superclass constructor invocation while preserving restrictions needed for safe construction. This can make constructor validation and initialization logic clearer. It does not mean that a constructor may freely use the partially initialized object before the superclass constructor has run.

### Q205. What is ScopedValue, and how is it different from ThreadLocal?

Scoped values provide a way to share immutable or effectively immutable contextual data across a bounded dynamic scope, including supported task structures. Java 25 finalized scoped values. ThreadLocal associates data with a thread and can be easy to misuse with pooled threads if values are not cleared; scoped values make lifetime and propagation intent more explicit. They are not a universal replacement for every ThreadLocal use case.

### Q206. What is the status of structured concurrency in Java 25?

Structured concurrency remained a preview API in Java 25, while scoped values became final. Structured concurrency treats related concurrent subtasks as a unit with a common lifetime and coordinated failure handling. Since preview APIs can change and require enablement, verify the target JDK release before using them in production code or describing them as finalized.

### Q207. What performance-oriented Java 25 changes should you recognize?

Java 25 includes performance and runtime improvements such as compact object headers, ahead-of-time cache and method-profiling work, generational Shenandoah, and JFR improvements. Some behavior depends on flags, platform, or collector choice. The interview-level answer should explain the problem each feature targets and emphasize measuring a representative workload rather than enabling tuning flags by intuition.

### Q208. Which Java 25 features are finalized versus preview or incubating?

Among the language-facing finalized features are module import declarations, compact source files and instance main methods, and flexible constructor bodies. Scoped values and the Key Derivation Function API are also final. Primitive types in patterns/instanceof/switch remained a preview feature in Java 25; structured concurrency was still preview and the Vector API remained incubating. Check the official release documentation for the full feature and status list.


## 16. Core Java language, object model, and API contracts

### Q209. What is the difference between a primitive and a reference type?

Primitive variables hold values from primitive types such as int, boolean, and double. Reference variables hold references to objects or null. Primitives do not have object identity or instance methods, while reference values participate in object-oriented behavior and can be null. Wrapper types bridge primitive values and APIs that require objects, such as generic collections.

### Q210. What is numeric promotion in Java?

In arithmetic expressions, smaller integral types are commonly promoted to int, and mixed numeric operands are promoted according to Java's binary numeric promotion rules. This is why adding two byte values produces an int expression and assigning it directly to a byte requires a cast. Be careful with overflow: integer overflow does not automatically throw an exception.

### Q211. What is the difference between widening and narrowing conversions?

Widening conversions generally move to a type with a broader range or precision, such as int to long, and are often implicit. Narrowing conversions can lose information, such as long to int or double to int, and usually require an explicit cast. An explicit cast does not guarantee the value fits; validate ranges when data loss is unacceptable.

### Q212. What is the difference between final, finally, and finalize?

final restricts reassignment, overriding, or inheritance depending on where it is applied. finally is a control-flow block commonly used for cleanup. finalize was an object cleanup mechanism that is deprecated and should not be relied on; it is unpredictable and unsuitable for resource management. Use try-with-resources for closeable resources.

### Q213. What does static mean for fields, methods, and nested classes?

A static field belongs to the class rather than each instance, and a static method is invoked with class-level context rather than an implicit this reference. A static nested class does not require an enclosing instance. Static members can be useful for constants and stateless utilities, but mutable global static state complicates testing, concurrency, and lifecycle management.

### Q214. Can a static method be overridden?

No. Static methods are hidden rather than overridden. The method selected depends on the compile-time type used to refer to the class, whereas overridden instance methods use dynamic dispatch based on the runtime object. This distinction matters in output questions and is a reason not to model polymorphic behavior with static methods.

### Q215. What is the difference between this and super?

this refers to the current instance and can invoke another constructor in the same class. super accesses superclass members or invokes a superclass constructor. A constructor delegation call using this(...) or super(...) has placement rules; modern Java has relaxed some constructor-body restrictions, but code must still follow the target release's rules.

### Q216. Why are constructors not inherited or overridden?

A constructor initializes an instance of the class in which it is declared; it is not an ordinary inherited method. A subclass constructor must arrange for superclass initialization, directly or through another constructor. Constructors can be overloaded, but not overridden, because constructor selection occurs as part of object creation rather than virtual method dispatch.

### Q217. What is covariant return typing?

An overriding method may return a subtype of the return type declared by the overridden method. This allows an override to return a more specific result without forcing callers to cast. Parameter types do not work the same way: changing parameter types creates an overload rather than a valid override.

### Q218. What is the difference between an abstract class and an interface in modern Java?

An abstract class can hold instance state, constructors, and a mix of abstract and concrete methods. An interface defines a type contract and can include abstract, default, static, and private methods, but it does not provide ordinary per-instance fields. Use an interface for capabilities and interchangeable implementations; use an abstract class when shared state or protected implementation is genuinely part of the design.

### Q219. What are default-method conflicts in interfaces?

If unrelated interfaces provide conflicting default implementations, a class may need to override the method and resolve the conflict explicitly. A class implementation takes precedence over an interface default, and a more specific interface can take precedence over a less specific one. Default methods support interface evolution but should not be used to hide ambiguous domain design.

### Q220. What are the rules for equals() symmetry and transitivity?

Equality must be reflexive, symmetric, transitive, consistent, and false for null. A common mistake is implementing equality in a superclass and then extending it with extra equality fields in a subclass, which can violate symmetry. Prefer well-defined value types and avoid inheritance structures that make equality semantics ambiguous.

### Q221. Why should hashCode use the same logical fields as equals?

If two objects are equal according to equals, they must return the same hash code. Hash collisions are allowed, so unequal objects can share a hash code. When equality-relevant fields change while an object is a map key, lookup can fail. Base both methods on stable identity fields and test them together.

### Q222. What is the difference between shallow copy and deep copy?

A shallow copy duplicates the outer object but may retain references to the same nested mutable objects. A deep copy duplicates the relevant nested state as well. Deep copying can be expensive and is domain-dependent; defensive copying of exposed mutable state is often a better targeted solution than indiscriminately copying an entire object graph.

### Q223. What is defensive copying, and where is it needed?

Defensive copying prevents callers from mutating internal state through references they supplied or received. Copy mutable constructor arguments before storing them and return a copy or unmodifiable representation when exposing mutable internal collections. An unmodifiable wrapper alone is not a snapshot if another reference can still mutate the backing collection.

### Q224. Why are enums often better than string constants?

Enums provide a closed set of typed values, can hold methods and fields, and work naturally with switch statements and collections. They reduce invalid values caused by typos and make the domain explicit. Do not persist enum ordinal values as a durable external contract because declaration order can change; use stable names or explicit codes when needed.

### Q225. What is the difference between == and equals for arrays?

Array equality with == compares whether both references point to the same array object. Arrays.equals compares elements for one-dimensional arrays, and Arrays.deepEquals supports nested arrays. Arrays also inherit Object's equals implementation, so two arrays with identical contents are not equal by default under ordinary equals calls.

### Q226. What are common pitfalls with BigDecimal?

Constructing BigDecimal from a double captures the double's binary approximation; use a string or an appropriate exact constructor when decimal intent matters. equals considers both numerical value and scale, while compareTo compares numerical magnitude. Choose a consistent representation and rounding policy, particularly for monetary calculations.

### Q227. What is the difference between checked exceptions and unchecked exceptions in API design?

Checked exceptions require callers to catch or declare them, making recoverable failure possibilities visible in a method signature. Unchecked exceptions are often used for programming errors or failures that callers cannot reasonably recover from locally. Choose based on the abstraction's contract, preserve causes when wrapping, and avoid catching exceptions only to log and rethrow unchanged.

### Q228. Can a catch block catch an exception thrown by another catch block?

No. A catch block handles exceptions from the associated try block; an exception thrown while executing a catch block is not handled by a sibling catch clause. It propagates outward unless another enclosing handler catches it. This is useful to remember when writing exception output questions and nested error-handling logic.

### Q229. What is multi-catch, and what restrictions apply?

Multi-catch allows one catch block to handle several unrelated exception types using a pipe-separated declaration. The alternatives cannot be related by subclassing, because one would be redundant. The multi-catch parameter is effectively final, which prevents reassignment and keeps the declared union type safe.

### Q230. What is the difference between throw and throws?

throw actually raises an exception object at a point in execution. throws declares exception types that a method may propagate as part of its signature. A declaration does not itself throw anything; it describes the method contract and affects compile-time checking for checked exceptions.

### Q231. What is type erasure, and what limitations does it create?

Java generics are largely implemented by erasing type parameters during compilation and inserting casts or bridge methods where required. This supports compatibility with older bytecode, but means some runtime operations cannot distinguish parameterized types such as List<String> and List<Integer>. It also explains restrictions such as generic array creation and many parameterized instanceof checks.

### Q232. What are bridge methods?

The compiler can generate synthetic bridge methods to preserve polymorphism after type erasure or covariant return changes. They help ensure that overriding behavior remains correct when generic source-level signatures erase to compatible JVM-level signatures. They are generally invisible in normal source code but may appear in reflection or bytecode inspection.

### Q233. What is the difference between extends and super wildcards?

Use ? extends T when a generic structure produces values of type T or a subtype and you primarily need to read them. Use ? super T when it consumes values of type T and you need to write them. The PECS mnemonic—Producer Extends, Consumer Super—is a useful starting point, but the actual API contract determines the correct type.

### Q234. Why are generics invariant when arrays are covariant?

A List<Integer> is not a subtype of List<Number>, because otherwise code could insert a Double into a list intended to contain Integer values. Arrays allow covariance but enforce component types at runtime, which can produce ArrayStoreException. Generic invariance catches unsafe assignments at compile time; wildcards express safe producer or consumer relationships.

### Q235. What are raw types, and why should they be avoided?

A raw type uses a generic class without its type argument, such as List rather than List<String>. It bypasses generic type checking and can introduce values that fail later with ClassCastException. Use parameterized types throughout modern code and isolate any unavoidable raw-type interaction behind a narrow, documented boundary.

### Q236. What is the difference between a generic method and a generic class?

A generic class declares type parameters that apply to its instance or class-level API. A generic method declares its own type parameters for one method invocation, even if its enclosing class is not generic. Use the narrowest type parameter scope that expresses the relationship between the method's inputs and outputs.

### Q237. What is an annotation's retention policy?

SOURCE retention means the annotation is discarded by the compiler, CLASS retention keeps it in the class file but does not require runtime reflection visibility, and RUNTIME retention makes it available to runtime reflection. Frameworks that inspect annotations at runtime need appropriate retention. Annotation metadata alone does not trigger behavior; a compiler, processor, or runtime component must interpret it.

### Q238. What is the difference between reflection and annotation processing?

Reflection inspects runtime classes and members while the program is running. Annotation processing operates during compilation and can inspect source-level declarations and generate source or resources. Compile-time processing can catch or generate things earlier, while reflection supports runtime discovery but adds runtime complexity and access constraints.

### Q239. What is a dynamic proxy, and where is it used?

A dynamic proxy can create an object that implements specified interfaces and forwards method calls to an invocation handler. Frameworks use proxies for cross-cutting behavior such as transactions, security, and interception. Proxy-based mechanisms have limitations around concrete classes, final methods, self-invocation, and method visibility depending on the proxy technology.

### Q240. What is the difference between a checked exception and an Error?

Exceptions generally represent conditions application code may handle or propagate. Error represents serious problems associated with the runtime or environment, such as linkage failures or resource exhaustion, and application code usually should not attempt to recover from arbitrary Errors. Cleanup in finally or try-with-resources still matters during abrupt termination, but do not swallow fatal conditions indiscriminately.

## 19. Last-minute revision

If only a short time remains, prioritize these questions and make sure the explanations are clear enough to say aloud:

1. Pass-by-value and object references (Q2)
2. `==` vs `equals()`, and the equals/hashCode contract (Q3, Q5)
3. String immutability and StringBuilder (Q4)
4. Checked vs unchecked exceptions (Q6)
5. Interface vs abstract class; overloading vs overriding (Q7, Q9)
6. SOLID and immutability (Q10, Q11)
7. HashMap internals and mutable keys (Q12, Q13)
8. ArrayList vs LinkedList; HashMap vs ConcurrentHashMap (Q14, Q16)
9. Generics, type erasure, PECS (Q19, Q20)
10. Streams: map/flatMap, laziness, parallel streams, reduction (Q21–Q25)
11. synchronized vs volatile; happens-before; atomicity (Q27–Q29)
12. ExecutorService, CompletableFuture, deadlocks, interruption (Q30–Q35)
13. Heap vs stack, GC, memory leaks, JIT (Q37–Q42)
14. Java 8, records/sealed classes, Java 21 virtual threads, Java 25 migration awareness (Q44–Q48)
15. Diagnose latency, GC pressure, executor starvation, and safe retries (Q49–Q54)
16. Revise I/O/resource management, serialization risks, JDBC transactions, reflection/JPMS, design patterns, benchmarking, testing/build compatibility, secure coding, HTTP client behavior, and class loading (Q55–Q64)

**Revision method:** For each question, first give a concise explanation, then explain the mechanism, one important limitation or trade-off, and a practical example. If an answer takes several minutes, identify the core answer and keep the deeper detail ready for follow-up questions.

---

## 20. Official references

Use official specifications and API documentation as the final authority for language and runtime contracts. Secondary interview guides are useful for discovering recurring themes, not for proving technical behavior or guaranteeing company-specific frequency.

- [Java SE 25 specifications](https://docs.oracle.com/en/java/javase/25/docs/specs/)
- [Java SE 25 API documentation](https://docs.oracle.com/en/java/javase/25/docs/api/)
- [Java Language Specification, Java SE 21](https://docs.oracle.com/javase/specs/jls/se21/html/index.html)
- [Java Virtual Machine Specification, Java SE 21](https://docs.oracle.com/javase/specs/jvms/se21/html/index.html)
- [Java SE 21 API documentation](https://docs.oracle.com/en/java/javase/21/docs/api/)
- [Virtual Threads adoption guide](https://docs.oracle.com/en/java/javase/21/core/virtual-threads.html)
- [Java Microbenchmark Harness (JMH)](https://openjdk.org/projects/code-tools/jmh/)
- [JEP 444 — Virtual Threads](https://openjdk.org/jeps/444)
- [JEP 440 — Record Patterns](https://openjdk.org/jeps/440)
- [JEP 441 — Pattern Matching for switch](https://openjdk.org/jeps/441)
- [JEP 431 — Sequenced Collections](https://openjdk.org/jeps/431)
- [Java SE specifications index](https://docs.oracle.com/javase/specs/)
- [Java interview question themes (secondary source)](https://www.crackr.dev/interview-guides/java-interview-questions)
- [Java interview question set (secondary source)](https://coddy.tech/interview-questions/java)

---

## 21. Editorial and validation notes

- This is the **curated primary interview guide**, not a claim that every advanced Java API can be covered in 54 questions.
- It deliberately favors common concepts and senior-level reasoning over obscure trivia and repetitive question variants.
- Java API contracts, language guarantees, and implementation-specific details are distinguished where that distinction matters.
- Preview/incubator status is release-specific; verify the target JDK before relying on a feature.
- Code execution and a full line-by-line audit of every legacy chapter are separate validation activities. Do not interpret the existence of this guide as evidence that every old snippet has been compiled or executed.
