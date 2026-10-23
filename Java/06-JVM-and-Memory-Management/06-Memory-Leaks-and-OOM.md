# Memory Leaks and OutOfMemoryError

**Target level:** Advanced → Senior  
**Scope:** Java 8–21

Java memory failures can originate in heap retention, excessive allocation, Metaspace, direct buffers, thread stacks, native libraries, or container limits. Diagnose the specific failure category before changing heap size or garbage collector settings.

## 1. Leak patterns

### Q1. What is the difference between a memory leak and high allocation? *(Intermediate)*

High allocation means objects are created rapidly; many may be short-lived and reclaimed normally. A leak means objects remain reachable longer than intended, so retained memory grows or stays unnecessarily high.

Use allocation profiling to find creation hotspots and heap dumps to find retained objects. The two problems can coexist, but they require different fixes.

### Q2. How can a static collection cause a leak? *(Intermediate)*

A static collection remains reachable through its class for as long as the class/loader remains reachable. If entries are added and never removed, their keys, values, and everything reachable from them can accumulate indefinitely.

Use explicit lifecycle management, bounded caches, or weak-reference patterns only when their semantics fit. Weak references are not a substitute for a clear eviction policy.

### Q3. How can listeners cause memory leaks? *(Intermediate)*

A long-lived event source can retain listeners through strong references. If a listener captures a short-lived controller or request object and is never unregistered, the entire captured object graph can remain reachable.

Define listener registration/unregistration lifecycle and use scoped registration patterns. Heap dumps can reveal the retaining path from the source to the listener.

### Q4. How can ThreadLocal cause a leak? *(Advanced)*

ThreadLocal values are stored in thread-associated maps. In thread pools, workers may outlive requests, so per-request data can remain attached to a long-lived thread if cleanup is missed. Weak keys do not automatically mean values disappear immediately in every lifecycle scenario.

Use try/finally to call remove when the thread-local context should end. Test exceptional paths as well as normal completion.

### Q5. How can a class-loader leak appear after redeployment? *(Advanced)*

Old class loaders may remain reachable through static registries, threads, ThreadLocals, JDBC drivers, logging frameworks, caches, or callbacks. If the loader remains reachable, its class metadata may not be unloaded.

Inspect heap dumps for old loader instances and paths from GC roots. Correct cleanup of thread and global registrations is often more important than increasing Metaspace.

## 2. OutOfMemoryError categories

### Q6. What does Java heap space mean? *(Intermediate)*

It indicates that an allocation could not be satisfied from the Java heap under the current conditions. Possible causes include a heap too small for the live set, a leak, a burst of allocation, or an unexpectedly large object graph.

Collect heap dumps and GC logs when safe, inspect retained size, and verify the configured heap and container limit. Increasing heap may be appropriate only after confirming the live set and total process budget.

### Q7. What does Metaspace OOM indicate? *(Advanced)*

It indicates the JVM could not allocate required class metadata under the current native-memory availability or configured Metaspace limit. High class cardinality, dynamic class generation, or class-loader retention can be involved.

Inspect class-loading/unloading metrics, loader count, and Metaspace usage. Raising the limit can buy headroom but will not fix a loader leak.

### Q8. What does Direct buffer memory mean? *(Advanced)*

It commonly indicates failure to allocate native memory for direct buffers under the relevant limit or process constraints. The Java heap may still have available space because direct buffers use memory outside the ordinary heap.

Inspect direct-buffer metrics, allocation lifecycle, native memory, and container limits. Ensure buffers are not retained unnecessarily and account for I/O framework behavior.

### Q9. What does unable to create native thread mean? *(Advanced)*

The process cannot create another platform thread because of OS limits, native-memory pressure, stack reservation, container restrictions, or excessive thread creation. It is not necessarily a heap OOM.

Check thread count, executor configuration, thread dumps, and native memory. Bound concurrency and fix thread leaks rather than simply increasing heap.

### Q10. What does GC overhead limit exceeded mean? *(Advanced)*

It indicates the JVM is spending an excessive proportion of time collecting garbage while recovering little heap under the relevant runtime policy. It often suggests a leak or a heap too small for the live set, but logs and dumps are needed to confirm.

Disabling the limit can leave the process unresponsive while delaying failure. Diagnose retention and workload first.

## 3. Diagnostic workflow

### Q11. What is a heap dump? *(Intermediate)*

A heap dump captures objects and references at a point in time for analysis. It can reveal class histograms, dominator trees, retained sizes, and paths from GC roots.

Heap dumps may contain sensitive application data and can be large. Store them securely and capture them in a way that considers production impact.

### Q12. What is a dominator tree? *(Advanced)*

A dominator tree organizes objects by reference dominance: an object dominates another when every path from the roots to that object passes through the dominator. It helps identify objects whose retention keeps large subgraphs alive.

A high retained-size object is a useful lead, not automatically the root cause. Inspect the retaining path and determine whether the reference is intentional.

### Q13. How do you use GC logs? *(Advanced)*

GC logs show collection events, pause times, heap occupancy, allocation/evacuation behavior, and collector-specific details. Unified logging in modern JDKs uses options such as -Xlog:gc*; exact flags should be verified for the target release.

Correlate GC events with latency and allocation. A GC event near a slow request is not proof that GC caused the request's latency.

### Q14. How do JFR and heap dumps complement each other? *(Senior)*

JFR can show allocation hotspots, GC events, thread activity, locks, and timing over an interval. A heap dump shows the retained object graph at a specific moment. JFR is useful for understanding how pressure is created; heap analysis is useful for understanding what remains.

Use both when the problem includes growth over time. Capture evidence before restarting if safe and preserve enough metadata to compare the failing and healthy states.

### Q15. How do you investigate a container killed without Java heap OOM? *(Senior)*

Check container and kernel OOM events, process RSS, cgroup limits, native memory, direct buffers, thread stacks, Metaspace, and other processes sharing the limit. The container may be killed before the JVM can throw a Java OutOfMemoryError.

Budget heap below the container limit with measured native overhead. Verify the JDK's container-awareness behavior and use Native Memory Tracking where appropriate.

## 4. Prevention

### Q16. How can memory usage be bounded in a backend service? *(Senior)*

Bound caches, queues, request bodies, batch sizes, concurrent tasks, and retained histories. Stream large data rather than buffering it all, and release listeners/thread-local context/resources at lifecycle boundaries.

Set explicit limits based on SLOs and load tests. Unbounded data structures are often more dangerous than a slightly inefficient object layout.

### Q17. How should an OOM response be designed? *(Senior)*

Treat OOM as a serious process-health event. Preserve diagnostics when safe, avoid attempting complex recovery in a heap-exhausted process, and allow orchestration to restart the instance according to a controlled policy. Some OOM conditions are not safely recoverable.

Configure heap-dump-on-OOM and logs thoughtfully, considering disk capacity and sensitive data. Ensure crash-loop behavior does not repeatedly overload dependencies during recovery.

### Q18. What is a memory-budget review checklist? *(Senior)*

Account for heap, Metaspace, thread stacks, direct buffers, code cache, native libraries, queued tasks, caches, per-request objects, and peak concurrent operations. Include deployment/container limits and temporary peaks during resizing or garbage collection.

Validate the budget with representative load and failure scenarios. Monitor both absolute usage and growth rates so a leak can be detected before the hard limit is reached.

## Official references

1. [Java SE 21 GC Tuning Guide](https://docs.oracle.com/en/java/javase/21/gctuning/)
2. [Java SE 21 Native Memory Tracking](https://docs.oracle.com/en/java/javase/21/vm/native-memory-tracking.html)
3. [Java SE 21 tool documentation](https://docs.oracle.com/en/java/javase/21/docs/specs/man/index.html)
4. [Java SE 21 API — ThreadLocal](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/lang/ThreadLocal.html)

## Chapter checklist

- [ ] Separate allocation churn from retained-object leaks.
- [ ] Diagnose the specific OOM category.
- [ ] Use heap dumps, JFR, GC logs, and native-memory evidence together.
- [ ] Bound caches, queues, batches, and concurrency.
- [ ] Include container memory and crash-recovery behavior in the design.
