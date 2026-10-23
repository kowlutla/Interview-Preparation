# Garbage Collection

**Target level:** Advanced → Senior  
**Scope:** Java 8–21

Garbage collection reclaims heap memory for objects that are no longer reachable from GC roots. It is not deterministic resource management and does not guarantee immediate reclamation. GC tuning should begin with the application's latency, throughput, allocation, and memory objectives rather than a favorite collector flag.

## 1. Reachability and collection

### Q1. How does Java garbage collection determine whether an object is live? *(Intermediate)*

A tracing collector starts from GC roots such as active thread references, static references, and other VM-defined roots, then follows reachable references. Objects not reachable from the roots can be reclaimed, subject to collector and reference-processing rules.

An object can be logically unused but still reachable through a cache, listener, ThreadLocal, static collection, or accidental reference chain. Memory leaks in Java often mean unintended reachability rather than manually forgotten free calls.

### Q2. What are GC roots? *(Advanced)*

GC roots are runtime references from which reachability tracing begins, including references associated with live threads, stack frames, static fields, and JVM internals. The exact root set depends on the runtime state and implementation.

When investigating retention, inspect the path from a GC root to the object. The first unintended retaining reference is often more useful than the object's class alone.

### Q3. What is the difference between minor and major/full GC? *(Advanced)*

These terms have historically described collections of different generations or broader heap regions, but exact meanings depend on the collector and JDK. Modern collectors may not map neatly to one universal “minor/major/full” model.

Read the logs for the selected collector and JDK. Prefer the specific event names and regions reported by that collector instead of assuming every JVM uses the same generational collection terminology.

### Q4. What is generational garbage collection? *(Intermediate)*

Generational collectors exploit the empirical observation that many objects die young. They separate allocation/collection strategies for younger and older objects, reducing the work needed for many short-lived allocations.

Generational behavior is collector-dependent and can evolve across JDK versions. It is a performance strategy, not a Java language guarantee.

### Q5. What are stop-the-world pauses? *(Intermediate)*

A stop-the-world pause is a period when application threads are paused for a JVM operation. Many collectors use pauses for some phases, while others perform substantial work concurrently. The duration and frequency depend on collector, heap, allocation rate, live-set size, and workload.

Latency-sensitive systems should monitor pause percentiles and total time, not only average pause duration. A low average can hide rare long pauses that violate an SLO.

## 2. Collectors and trade-offs

### Q6. What are the main garbage collectors in HotSpot? *(Advanced)*

Common HotSpot collectors include Serial, Parallel, G1, ZGC, and Shenandoah in supported distributions/releases. Availability and support can vary by JDK build and vendor. Each collector makes different trade-offs among throughput, pause targets, memory overhead, and concurrent CPU use.

Do not select a collector based only on a slogan. Verify availability on the target JDK and benchmark representative workloads with the required heap size and latency target.

### Q7. What is G1 GC? *(Advanced)*

G1 is a region-based collector designed to balance throughput and predictable pause-time goals. It divides the heap into regions and selects collection work based partly on expected benefit and pause target.

A pause target is not a hard real-time guarantee. If the live set, allocation pressure, or workload exceeds the collector's ability to meet the target, observed pauses can be longer.

### Q8. What is ZGC intended for? *(Advanced)*

ZGC is designed for low-latency garbage collection with substantial concurrent work, targeting applications where pause-time sensitivity matters. Its exact capabilities and platform support evolve across JDK releases.

Low pause time does not mean zero GC cost. Concurrent collection consumes CPU and memory bandwidth, and heap sizing still matters. Test the specific JDK build and workload.

### Q9. What is the difference between throughput and latency-oriented GC tuning? *(Senior)*

Throughput tuning aims to maximize useful application work over time, while latency-oriented tuning focuses on pause durations and tail latency. These goals can conflict because concurrent collection uses CPU and memory bandwidth that could otherwise serve application threads.

Define an SLO and measure the cost of GC, allocation, and dependency latency. A collector with shorter pauses may have lower peak throughput or higher memory overhead.

## 3. References and leaks

### Q10. What are strong, soft, weak, and phantom references? *(Advanced)*

Strong references keep objects reachable normally. Soft references may be cleared under memory pressure according to runtime policy. Weak references do not keep referents strongly reachable, and phantom references support post-mortem cleanup coordination through a ReferenceQueue without exposing the referent for ordinary access.

Do not use soft references as a predictable cache eviction policy; behavior is runtime-dependent. Use an explicit cache with size/expiration controls for production caching.

### Q11. What is a memory leak in a garbage-collected language? *(Intermediate)*

A Java memory leak usually means objects remain reachable even though the application no longer needs them. Common causes include unbounded caches, static collections, listeners that are never removed, ThreadLocal values retained by pooled threads, and references from long-lived objects.

Heap dumps, dominator trees, retained-size analysis, and reference paths help find the retaining chain. Increasing heap size may postpone the failure without fixing the leak.

### Q12. What is a reference queue? *(Advanced)*

ReferenceQueue allows code to learn when reference objects have been enqueued after their referents meet the relevant reachability conditions. It is used with weak/soft/phantom references for cleanup or bookkeeping.

Enqueue timing is not deterministic, so reference queues should not be treated as a real-time cleanup scheduler. Explicit resource management remains the primary mechanism for closing files, sockets, and native handles.

### Q13. Why is finalize not a valid cleanup strategy? *(Advanced)*

Finalization is deprecated for removal, nondeterministic, and can delay reclamation or introduce security and performance hazards. It is not guaranteed to run promptly and should not be relied on for releasing scarce resources.

Use AutoCloseable and try-with-resources. Cleaner can provide a fallback mechanism for certain native resources, but explicit close should remain the normal path.

## 4. Production diagnosis

### Q14. How do you diagnose high allocation rate? *(Senior)*

Use JFR, allocation profiling, and GC logs to identify allocation hotspots, object types, and rates. Examine temporary collections, boxing, string concatenation, serialization, request object graphs, and excessive buffering.

High allocation is not always a leak; short-lived allocation can be normal. Determine whether allocation causes GC overhead or latency, then optimize the dominant source with measurements.

### Q15. How do you investigate OutOfMemoryError? *(Senior)*

Identify the exact error: Java heap space, GC overhead limit, Metaspace, direct buffer memory, unable to create native thread, or another native allocation failure. Collect heap dumps when safe, GC logs, process memory, thread counts, and container limits.

Do not assume every OOM means the heap is too small. Native memory, thread stacks, direct buffers, class-loader retention, and container limits can be the actual constraint.

### Q16. What does GC overhead limit exceeded mean? *(Advanced)*

It is a JVM condition that can occur when the runtime spends an excessive proportion of time collecting garbage while recovering very little heap. It often indicates insufficient effective heap for the live set or a memory leak, but diagnosis requires logs and heap evidence.

Disabling the check may postpone the error while leaving the system unusable. Find the retained objects and workload cause before changing the flag.

### Q17. How should heap size be chosen in a container? *(Senior)*

Account for container memory limits, heap, Metaspace, thread stacks, direct buffers, code cache, native libraries, and other process overhead. Setting heap maximum equal to the container limit can cause a container OOM even if the Java heap itself stays within its limit.

Measure peak resident memory under representative load and leave a deliberate native-memory margin. Verify container awareness and JVM ergonomics for the target JDK.

### Q18. What GC metrics matter for a production service? *(Senior)*

Track allocation rate, live-set size, heap occupancy before/after collection, pause duration and frequency, concurrent GC CPU, promotion/old-generation behavior where applicable, and OOM events. Correlate these with latency percentiles, throughput, and container memory.

Metrics should answer whether GC is the cause or merely coincident with latency. Use logs and JFR to connect events to workload behavior.

## Official references

1. [Java SE 21 GC Tuning Guide](https://docs.oracle.com/en/java/javase/21/gctuning/)
2. [Java SE 21 API — Cleaner](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/lang/ref/Cleaner.html)
3. [JEP 421 — Deprecate Finalization for Removal](https://openjdk.org/jeps/421)
4. [Java SE 21 Flight Recorder documentation](https://docs.oracle.com/en/java/javase/21/jfapi/index.html)

## Chapter checklist

- [ ] Explain reachability, GC roots, and reference types.
- [ ] Distinguish collector-specific behavior from universal guarantees.
- [ ] Diagnose leaks through retaining paths.
- [ ] Investigate heap and native OOM separately.
- [ ] Tune against explicit latency/throughput goals and measured evidence.
